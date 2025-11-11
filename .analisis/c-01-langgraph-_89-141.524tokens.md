                        CONFIG_KEY_RUNNER_SUBMIT, weakref.WeakMethod(loop.submit)
                    ),
                    put_writes=weakref.WeakMethod(loop.put_writes),
                    node_finished=config[CONF].get(CONFIG_KEY_NODE_FINISHED),
                )
                # enable subgraph streaming
                if subgraphs:
                    loop.config[CONF][CONFIG_KEY_STREAM] = loop.stream
                # enable concurrent streaming
                get_waiter: Callable[[], concurrent.futures.Future[None]] | None = None
                if (
                    self.stream_eager
                    or subgraphs
                    or "messages" in stream_modes
                    or "custom" in stream_modes
                ):
                    # we are careful to have a single waiter live at any one time
                    # because on exit we increment semaphore count by exactly 1
                    waiter: concurrent.futures.Future | None = None
                    # because sync futures cannot be cancelled, we instead
                    # release the stream semaphore on exit, which will cause
                    # a pending waiter to return immediately
                    loop.stack.callback(stream._count.release)

                    def get_waiter() -> concurrent.futures.Future[None]:
                        nonlocal waiter
                        if waiter is None or waiter.done():
                            waiter = loop.submit(stream.wait)
                            return waiter
                        else:
                            return waiter

                # Similarly to Bulk Synchronous Parallel / Pregel model
                # computation proceeds in steps, while there are channel updates.
                # Channel updates from step N are only visible in step N+1
                # channels are guaranteed to be immutable for the duration of the step,
                # with channel updates applied only at the transition between steps.
                while loop.tick():
                    for task in loop.match_cached_writes():
                        loop.output_writes(task.id, task.writes, cached=True)
                    for _ in runner.tick(
                        [t for t in loop.tasks.values() if not t.writes],
                        timeout=self.step_timeout,
                        get_waiter=get_waiter,
                        schedule_task=loop.accept_push,
                    ):
                        # emit output
                        yield from _output(
                            stream_mode, print_mode, subgraphs, stream.get, queue.Empty
                        )
                    loop.after_tick()
                    # wait for checkpoint
                    if durability_ == "sync":
                        loop._put_checkpoint_fut.result()
            # emit output
            yield from _output(
                stream_mode, print_mode, subgraphs, stream.get, queue.Empty
            )
            # handle exit
            if loop.status == "out_of_steps":
                msg = create_error_message(
                    message=(
                        f"Recursion limit of {config['recursion_limit']} reached "
                        "without hitting a stop condition. You can increase the "
                        "limit by setting the `recursion_limit` config key."
                    ),
                    error_code=ErrorCode.GRAPH_RECURSION_LIMIT,
                )
                raise GraphRecursionError(msg)
            # set final channel values as run output
            run_manager.on_chain_end(loop.output)
        except BaseException as e:
            run_manager.on_chain_error(e)
            raise

    async def astream(
        self,
        input: InputT | Command | None,
        config: RunnableConfig | None = None,
        *,
        context: ContextT | None = None,
        stream_mode: StreamMode | Sequence[StreamMode] | None = None,
        print_mode: StreamMode | Sequence[StreamMode] = (),
        output_keys: str | Sequence[str] | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        durability: Durability | None = None,
        subgraphs: bool = False,
        debug: bool | None = None,
        **kwargs: Unpack[DeprecatedKwargs],
    ) -> AsyncIterator[dict[str, Any] | Any]:
        """Asynchronously stream graph steps for a single input.

        Args:
            input: The input to the graph.
            config: The configuration to use for the run.
            context: The static context to use for the run.
                !!! version-added "Added in version 0.6.0"
            stream_mode: The mode to stream output, defaults to `self.stream_mode`.
                Options are:

                - `"values"`: Emit all values in the state after each step, including interrupts.
                    When used with functional API, values are emitted once at the end of the workflow.
                - `"updates"`: Emit only the node or task names and updates returned by the nodes or tasks after each step.
                    If multiple updates are made in the same step (e.g. multiple nodes are run) then those updates are emitted separately.
                - `"custom"`: Emit custom data from inside nodes or tasks using `StreamWriter`.
                - `"messages"`: Emit LLM messages token-by-token together with metadata for any LLM invocations inside nodes or tasks.
                    Will be emitted as 2-tuples `(LLM token, metadata)`.
                - `"checkpoints"`: Emit an event when a checkpoint is created, in the same format as returned by `get_state()`.
                - `"tasks"`: Emit events when tasks start and finish, including their results and errors.
                - `"debug"`: Emit debug events with as much information as possible for each step.

                You can pass a list as the `stream_mode` parameter to stream multiple modes at once.
                The streamed outputs will be tuples of `(mode, data)`.

                See [LangGraph streaming guide](https://docs.langchain.com/oss/python/langgraph/streaming) for more details.
            print_mode: Accepts the same values as `stream_mode`, but only prints the output to the console, for debugging purposes. Does not affect the output of the graph in any way.
            output_keys: The keys to stream, defaults to all non-context channels.
            interrupt_before: Nodes to interrupt before, defaults to all nodes in the graph.
            interrupt_after: Nodes to interrupt after, defaults to all nodes in the graph.
            durability: The durability mode for the graph execution, defaults to `"async"`.
                Options are:

                - `"sync"`: Changes are persisted synchronously before the next step starts.
                - `"async"`: Changes are persisted asynchronously while the next step executes.
                - `"exit"`: Changes are persisted only when the graph exits.
            subgraphs: Whether to stream events from inside subgraphs, defaults to False.
                If `True`, the events will be emitted as tuples `(namespace, data)`,
                or `(namespace, mode, data)` if `stream_mode` is a list,
                where `namespace` is a tuple with the path to the node where a subgraph is invoked,
                e.g. `("parent_node:<task_id>", "child_node:<task_id>")`.

                See [LangGraph streaming guide](https://docs.langchain.com/oss/python/langgraph/streaming) for more details.

        Yields:
            The output of each step in the graph. The output shape depends on the `stream_mode`.
        """
        if (checkpoint_during := kwargs.get("checkpoint_during")) is not None:
            warnings.warn(
                "`checkpoint_during` is deprecated and will be removed. Please use `durability` instead.",
                category=LangGraphDeprecatedSinceV10,
                stacklevel=2,
            )
            if durability is not None:
                raise ValueError(
                    "Cannot use both `checkpoint_during` and `durability` parameters. Please use `durability` instead."
                )
            durability = "async" if checkpoint_during else "exit"

        if stream_mode is None:
            # if being called as a node in another graph, default to values mode
            # but don't overwrite stream_mode arg if provided
            stream_mode = (
                "values"
                if config is not None and CONFIG_KEY_TASK_ID in config.get(CONF, {})
                else self.stream_mode
            )
        if debug or self.debug:
            print_mode = ["updates", "values"]

        stream = AsyncQueue()
        aioloop = asyncio.get_running_loop()
        stream_put = cast(
            Callable[[StreamChunk], None],
            partial(aioloop.call_soon_threadsafe, stream.put_nowait),
        )

        config = ensure_config(self.config, config)
        callback_manager = get_async_callback_manager_for_config(config)
        run_manager = await callback_manager.on_chain_start(
            None,
            input,
            name=config.get("run_name", self.get_name()),
            run_id=config.get("run_id"),
        )
        # if running from astream_log() run each proc with streaming
        do_stream = (
            next(
                (
                    True
                    for h in run_manager.handlers
                    if isinstance(h, _StreamingCallbackHandler)
                    and not isinstance(h, StreamMessagesHandler)
                ),
                False,
            )
            if _StreamingCallbackHandler is not None
            else False
        )
        try:
            # assign defaults
            (
                stream_modes,
                output_keys,
                interrupt_before_,
                interrupt_after_,
                checkpointer,
                store,
                cache,
                durability_,
            ) = self._defaults(
                config,
                stream_mode=stream_mode,
                print_mode=print_mode,
                output_keys=output_keys,
                interrupt_before=interrupt_before,
                interrupt_after=interrupt_after,
                durability=durability,
            )
            if checkpointer is None and durability is not None:
                warnings.warn(
                    "`durability` has no effect when no checkpointer is present.",
                )
            # set up subgraph checkpointing
            if self.checkpointer is True:
                ns = cast(str, config[CONF][CONFIG_KEY_CHECKPOINT_NS])
                config[CONF][CONFIG_KEY_CHECKPOINT_NS] = recast_checkpoint_ns(ns)
            # set up messages stream mode
            if "messages" in stream_modes:
                # namespace can be None in a root level graph?
                ns_ = cast(str | None, config[CONF].get(CONFIG_KEY_CHECKPOINT_NS))
                run_manager.inheritable_handlers.append(
                    StreamMessagesHandler(
                        stream_put,
                        subgraphs,
                        parent_ns=tuple(ns_.split(NS_SEP)) if ns_ else None,
                    )
                )

            # set up custom stream mode
            def stream_writer(c: Any) -> None:
                aioloop.call_soon_threadsafe(
                    stream.put_nowait,
                    (
                        tuple(
                            get_config()[CONF][CONFIG_KEY_CHECKPOINT_NS].split(NS_SEP)[
                                :-1
                            ]
                        ),
                        "custom",
                        c,
                    ),
                )

            if "custom" in stream_modes:

                def stream_writer(c: Any) -> None:
                    aioloop.call_soon_threadsafe(
                        stream.put_nowait,
                        (
                            tuple(
                                get_config()[CONF][CONFIG_KEY_CHECKPOINT_NS].split(
                                    NS_SEP
                                )[:-1]
                            ),
                            "custom",
                            c,
                        ),
                    )
            elif CONFIG_KEY_STREAM in config[CONF]:
                stream_writer = config[CONF][CONFIG_KEY_RUNTIME].stream_writer
            else:

                def stream_writer(c: Any) -> None:
                    pass

            # set durability mode for subgraphs
            if durability is not None:
                config[CONF][CONFIG_KEY_DURABILITY] = durability_

            runtime = Runtime(
                context=_coerce_context(self.context_schema, context),
                store=store,
                stream_writer=stream_writer,
                previous=None,
            )
            parent_runtime = config[CONF].get(CONFIG_KEY_RUNTIME, DEFAULT_RUNTIME)
            runtime = parent_runtime.merge(runtime)
            config[CONF][CONFIG_KEY_RUNTIME] = runtime

            async with AsyncPregelLoop(
                input,
                stream=StreamProtocol(stream.put_nowait, stream_modes),
                config=config,
                store=store,
                cache=cache,
                checkpointer=checkpointer,
                nodes=self.nodes,
                specs=self.channels,
                output_keys=output_keys,
                input_keys=self.input_channels,
                stream_keys=self.stream_channels_asis,
                interrupt_before=interrupt_before_,
                interrupt_after=interrupt_after_,
                manager=run_manager,
                durability=durability_,
                trigger_to_nodes=self.trigger_to_nodes,
                migrate_checkpoint=self._migrate_checkpoint,
                retry_policy=self.retry_policy,
                cache_policy=self.cache_policy,
            ) as loop:
                # create runner
                runner = PregelRunner(
                    submit=config[CONF].get(
                        CONFIG_KEY_RUNNER_SUBMIT, weakref.WeakMethod(loop.submit)
                    ),
                    put_writes=weakref.WeakMethod(loop.put_writes),
                    use_astream=do_stream,
                    node_finished=config[CONF].get(CONFIG_KEY_NODE_FINISHED),
                )
                # enable subgraph streaming
                if subgraphs:
                    loop.config[CONF][CONFIG_KEY_STREAM] = StreamProtocol(
                        stream_put, stream_modes
                    )
                # enable concurrent streaming
                get_waiter: Callable[[], asyncio.Task[None]] | None = None
                _cleanup_waiter: Callable[[], Awaitable[None]] | None = None
                if (
                    self.stream_eager
                    or subgraphs
                    or "messages" in stream_modes
                    or "custom" in stream_modes
                ):
                    # Keep a single waiter task alive; ensure cleanup on exit.
                    waiter: asyncio.Task[None] | None = None

                    def get_waiter() -> asyncio.Task[None]:
                        nonlocal waiter
                        if waiter is None or waiter.done():
                            waiter = aioloop.create_task(stream.wait())

                            def _clear(t: asyncio.Task[None]) -> None:
                                nonlocal waiter
                                if waiter is t:
                                    waiter = None

                            waiter.add_done_callback(_clear)
                        return waiter

                    async def _cleanup_waiter() -> None:
                        """Wake pending waiter and/or cancel+await to avoid pending tasks."""
                        nonlocal waiter
                        # Try to wake via semaphore like SyncPregelLoop
                        with contextlib.suppress(Exception):
                            if hasattr(stream, "_count"):
                                stream._count.release()
                        t = waiter
                        waiter = None
                        if t is not None and not t.done():
                            t.cancel()
                            with contextlib.suppress(asyncio.CancelledError):
                                await t

                # Similarly to Bulk Synchronous Parallel / Pregel model
                # computation proceeds in steps, while there are channel updates
                # channel updates from step N are only visible in step N+1
                # channels are guaranteed to be immutable for the duration of the step,
                # with channel updates applied only at the transition between steps
                try:
                    while loop.tick():
                        for task in await loop.amatch_cached_writes():
                            loop.output_writes(task.id, task.writes, cached=True)
                        async for _ in runner.atick(
                            [t for t in loop.tasks.values() if not t.writes],
                            timeout=self.step_timeout,
                            get_waiter=get_waiter,
                            schedule_task=loop.aaccept_push,
                        ):
                            # emit output
                            for o in _output(
                                stream_mode,
                                print_mode,
                                subgraphs,
                                stream.get_nowait,
                                asyncio.QueueEmpty,
                            ):
                                yield o
                        loop.after_tick()
                        # wait for checkpoint
                        if durability_ == "sync":
                            await cast(asyncio.Future, loop._put_checkpoint_fut)
                finally:
                    # ensure waiter doesn't remain pending on cancel/shutdown
                    if _cleanup_waiter is not None:
                        await _cleanup_waiter()

            # emit output
            for o in _output(
                stream_mode,
                print_mode,
                subgraphs,
                stream.get_nowait,
                asyncio.QueueEmpty,
            ):
                yield o
            # handle exit
            if loop.status == "out_of_steps":
                msg = create_error_message(
                    message=(
                        f"Recursion limit of {config['recursion_limit']} reached "
                        "without hitting a stop condition. You can increase the "
                        "limit by setting the `recursion_limit` config key."
                    ),
                    error_code=ErrorCode.GRAPH_RECURSION_LIMIT,
                )
                raise GraphRecursionError(msg)
            # set final channel values as run output
            await run_manager.on_chain_end(loop.output)
        except BaseException as e:
            await asyncio.shield(run_manager.on_chain_error(e))
            raise

    def invoke(
        self,
        input: InputT | Command | None,
        config: RunnableConfig | None = None,
        *,
        context: ContextT | None = None,
        stream_mode: StreamMode = "values",
        print_mode: StreamMode | Sequence[StreamMode] = (),
        output_keys: str | Sequence[str] | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        durability: Durability | None = None,
        **kwargs: Any,
    ) -> dict[str, Any] | Any:
        """Run the graph with a single input and config.

        Args:
            input: The input data for the graph. It can be a dictionary or any other type.
            config: The configuration for the graph run.
            context: The static context to use for the run.
                !!! version-added "Added in version 0.6.0"
            stream_mode: The stream mode for the graph run.
            print_mode: Accepts the same values as `stream_mode`, but only prints the output to the console, for debugging purposes. Does not affect the output of the graph in any way.
            output_keys: The output keys to retrieve from the graph run.
            interrupt_before: The nodes to interrupt the graph run before.
            interrupt_after: The nodes to interrupt the graph run after.
            durability: The durability mode for the graph execution, defaults to `"async"`.
                Options are:

                - `"sync"`: Changes are persisted synchronously before the next step starts.
                - `"async"`: Changes are persisted asynchronously while the next step executes.
                - `"exit"`: Changes are persisted only when the graph exits.
            **kwargs: Additional keyword arguments to pass to the graph run.

        Returns:
            The output of the graph run. If `stream_mode` is `"values"`, it returns the latest output.
            If `stream_mode` is not `"values"`, it returns a list of output chunks.
        """
        output_keys = output_keys if output_keys is not None else self.output_channels

        latest: dict[str, Any] | Any = None
        chunks: list[dict[str, Any] | Any] = []
        interrupts: list[Interrupt] = []

        for chunk in self.stream(
            input,
            config,
            context=context,
            stream_mode=["updates", "values"]
            if stream_mode == "values"
            else stream_mode,
            print_mode=print_mode,
            output_keys=output_keys,
            interrupt_before=interrupt_before,
            interrupt_after=interrupt_after,
            durability=durability,
            **kwargs,
        ):
            if stream_mode == "values":
                if len(chunk) == 2:
                    mode, payload = cast(tuple[StreamMode, Any], chunk)
                else:
                    _, mode, payload = cast(
                        tuple[tuple[str, ...], StreamMode, Any], chunk
                    )
                if (
                    mode == "updates"
                    and isinstance(payload, dict)
                    and (ints := payload.get(INTERRUPT)) is not None
                ):
                    interrupts.extend(ints)
                elif mode == "values":
                    latest = payload
            else:
                chunks.append(chunk)

        if stream_mode == "values":
            if interrupts:
                return (
                    {**latest, INTERRUPT: interrupts}
                    if isinstance(latest, dict)
                    else {INTERRUPT: interrupts}
                )
            return latest
        else:
            return chunks

    async def ainvoke(
        self,
        input: InputT | Command | None,
        config: RunnableConfig | None = None,
        *,
        context: ContextT | None = None,
        stream_mode: StreamMode = "values",
        print_mode: StreamMode | Sequence[StreamMode] = (),
        output_keys: str | Sequence[str] | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        durability: Durability | None = None,
        **kwargs: Any,
    ) -> dict[str, Any] | Any:
        """Asynchronously run the graph with a single input and config.

        Args:
            input: The input data for the graph. It can be a dictionary or any other type.
            config: The configuration for the graph run.
            context: The static context to use for the run.
                !!! version-added "Added in version 0.6.0"
            stream_mode: The stream mode for the graph run.
            print_mode: Accepts the same values as `stream_mode`, but only prints the output to the console, for debugging purposes. Does not affect the output of the graph in any way.
            output_keys: The output keys to retrieve from the graph run.
            interrupt_before: The nodes to interrupt the graph run before.
            interrupt_after: The nodes to interrupt the graph run after.
            durability: The durability mode for the graph execution, defaults to `"async"`.
                Options are:

                - `"sync"`: Changes are persisted synchronously before the next step starts.
                - `"async"`: Changes are persisted asynchronously while the next step executes.
                - `"exit"`: Changes are persisted only when the graph exits.
            **kwargs: Additional keyword arguments to pass to the graph run.

        Returns:
            The output of the graph run. If `stream_mode` is `"values"`, it returns the latest output.
            If `stream_mode` is not `"values"`, it returns a list of output chunks.
        """
        output_keys = output_keys if output_keys is not None else self.output_channels

        latest: dict[str, Any] | Any = None
        chunks: list[dict[str, Any] | Any] = []
        interrupts: list[Interrupt] = []

        async for chunk in self.astream(
            input,
            config,
            context=context,
            stream_mode=["updates", "values"]
            if stream_mode == "values"
            else stream_mode,
            print_mode=print_mode,
            output_keys=output_keys,
            interrupt_before=interrupt_before,
            interrupt_after=interrupt_after,
            durability=durability,
            **kwargs,
        ):
            if stream_mode == "values":
                if len(chunk) == 2:
                    mode, payload = cast(tuple[StreamMode, Any], chunk)
                else:
                    _, mode, payload = cast(
                        tuple[tuple[str, ...], StreamMode, Any], chunk
                    )
                if (
                    mode == "updates"
                    and isinstance(payload, dict)
                    and (ints := payload.get(INTERRUPT)) is not None
                ):
                    interrupts.extend(ints)
                elif mode == "values":
                    latest = payload
            else:
                chunks.append(chunk)

        if stream_mode == "values":
            if interrupts:
                return (
                    {**latest, INTERRUPT: interrupts}
                    if isinstance(latest, dict)
                    else {INTERRUPT: interrupts}
                )
            return latest
        else:
            return chunks

    def clear_cache(self, nodes: Sequence[str] | None = None) -> None:
        """Clear the cache for the given nodes."""
        if not self.cache:
            raise ValueError("No cache is set for this graph. Cannot clear cache.")
        nodes = nodes or self.nodes.keys()
        # collect namespaces to clear
        namespaces: list[tuple[str, ...]] = []
        for node in nodes:
            if node in self.nodes:
                namespaces.append(
                    (
                        CACHE_NS_WRITES,
                        (identifier(self.nodes[node]) or "__dynamic__"),
                        node,
                    ),
                )
        # clear cache
        self.cache.clear(namespaces)

    async def aclear_cache(self, nodes: Sequence[str] | None = None) -> None:
        """Asynchronously clear the cache for the given nodes."""
        if not self.cache:
            raise ValueError("No cache is set for this graph. Cannot clear cache.")
        nodes = nodes or self.nodes.keys()
        # collect namespaces to clear
        namespaces: list[tuple[str, ...]] = []
        for node in nodes:
            if node in self.nodes:
                namespaces.append(
                    (
                        CACHE_NS_WRITES,
                        (identifier(self.nodes[node]) or "__dynamic__"),
                        node,
                    ),
                )
        # clear cache
        await self.cache.aclear(namespaces)


def _trigger_to_nodes(nodes: dict[str, PregelNode]) -> Mapping[str, Sequence[str]]:
    """Index from a trigger to nodes that depend on it."""
    trigger_to_nodes: defaultdict[str, list[str]] = defaultdict(list)
    for name, node in nodes.items():
        for trigger in node.triggers:
            trigger_to_nodes[trigger].append(name)
    return dict(trigger_to_nodes)


def _output(
    stream_mode: StreamMode | Sequence[StreamMode],
    print_mode: StreamMode | Sequence[StreamMode],
    stream_subgraphs: bool,
    getter: Callable[[], tuple[tuple[str, ...], str, Any]],
    empty_exc: type[Exception],
) -> Iterator:
    while True:
        try:
            ns, mode, payload = getter()
        except empty_exc:
            break
        if mode in print_mode:
            if stream_subgraphs and ns:
                print(
                    " ".join(
                        (
                            get_bolded_text(f"[{mode}]"),
                            get_colored_text(f"[graph={ns}]", color="yellow"),
                            repr(payload),
                        )
                    )
                )
            else:
                print(
                    " ".join(
                        (
                            get_bolded_text(f"[{mode}]"),
                            repr(payload),
                        )
                    )
                )
        if mode in stream_mode:
            if stream_subgraphs and isinstance(stream_mode, list):
                yield (ns, mode, payload)
            elif isinstance(stream_mode, list):
                yield (mode, payload)
            elif stream_subgraphs:
                yield (ns, payload)
            else:
                yield payload


def _coerce_context(
    context_schema: type[ContextT] | None, context: Any
) -> ContextT | None:
    """Coerce context input to the appropriate schema type.

    If context is a dict and context_schema is a dataclass or pydantic model, we coerce.
    Else, we return the context as-is.

    Args:
        context_schema: The schema type to coerce to (BaseModel, dataclass, or TypedDict)
        context: The context value to coerce

    Returns:
        The coerced context value or None if context is None
    """
    if context is None:
        return None

    if context_schema is None:
        return context

    schema_is_class = issubclass(context_schema, BaseModel) or is_dataclass(
        context_schema
    )
    if isinstance(context, dict) and schema_is_class:
        return context_schema(**context)  # type: ignore[misc]

    return cast(ContextT, context)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/protocol.py
```py
from __future__ import annotations

from abc import abstractmethod
from collections.abc import AsyncIterator, Callable, Iterator, Sequence
from typing import Any, Generic, cast

from langchain_core.runnables import Runnable, RunnableConfig
from langchain_core.runnables.graph import Graph as DrawableGraph
from typing_extensions import Self

from langgraph.types import All, Command, StateSnapshot, StateUpdate, StreamMode
from langgraph.typing import ContextT, InputT, OutputT, StateT

__all__ = ("PregelProtocol", "StreamProtocol")


class PregelProtocol(Runnable[InputT, Any], Generic[StateT, ContextT, InputT, OutputT]):
    @abstractmethod
    def with_config(
        self, config: RunnableConfig | None = None, **kwargs: Any
    ) -> Self: ...

    @abstractmethod
    def get_graph(
        self,
        config: RunnableConfig | None = None,
        *,
        xray: int | bool = False,
    ) -> DrawableGraph: ...

    @abstractmethod
    async def aget_graph(
        self,
        config: RunnableConfig | None = None,
        *,
        xray: int | bool = False,
    ) -> DrawableGraph: ...

    @abstractmethod
    def get_state(
        self, config: RunnableConfig, *, subgraphs: bool = False
    ) -> StateSnapshot: ...

    @abstractmethod
    async def aget_state(
        self, config: RunnableConfig, *, subgraphs: bool = False
    ) -> StateSnapshot: ...

    @abstractmethod
    def get_state_history(
        self,
        config: RunnableConfig,
        *,
        filter: dict[str, Any] | None = None,
        before: RunnableConfig | None = None,
        limit: int | None = None,
    ) -> Iterator[StateSnapshot]: ...

    @abstractmethod
    def aget_state_history(
        self,
        config: RunnableConfig,
        *,
        filter: dict[str, Any] | None = None,
        before: RunnableConfig | None = None,
        limit: int | None = None,
    ) -> AsyncIterator[StateSnapshot]: ...

    @abstractmethod
    def bulk_update_state(
        self,
        config: RunnableConfig,
        updates: Sequence[Sequence[StateUpdate]],
    ) -> RunnableConfig: ...

    @abstractmethod
    async def abulk_update_state(
        self,
        config: RunnableConfig,
        updates: Sequence[Sequence[StateUpdate]],
    ) -> RunnableConfig: ...

    @abstractmethod
    def update_state(
        self,
        config: RunnableConfig,
        values: dict[str, Any] | Any | None,
        as_node: str | None = None,
    ) -> RunnableConfig: ...

    @abstractmethod
    async def aupdate_state(
        self,
        config: RunnableConfig,
        values: dict[str, Any] | Any | None,
        as_node: str | None = None,
    ) -> RunnableConfig: ...

    @abstractmethod
    def stream(
        self,
        input: InputT | Command | None,
        config: RunnableConfig | None = None,
        *,
        context: ContextT | None = None,
        stream_mode: StreamMode | list[StreamMode] | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        subgraphs: bool = False,
    ) -> Iterator[dict[str, Any] | Any]: ...

    @abstractmethod
    def astream(
        self,
        input: InputT | Command | None,
        config: RunnableConfig | None = None,
        *,
        context: ContextT | None = None,
        stream_mode: StreamMode | list[StreamMode] | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        subgraphs: bool = False,
    ) -> AsyncIterator[dict[str, Any] | Any]: ...

    @abstractmethod
    def invoke(
        self,
        input: InputT | Command | None,
        config: RunnableConfig | None = None,
        *,
        context: ContextT | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
    ) -> dict[str, Any] | Any: ...

    @abstractmethod
    async def ainvoke(
        self,
        input: InputT | Command | None,
        config: RunnableConfig | None = None,
        *,
        context: ContextT | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
    ) -> dict[str, Any] | Any: ...


StreamChunk = tuple[tuple[str, ...], str, Any]


class StreamProtocol:
    __slots__ = ("modes", "__call__")

    modes: set[StreamMode]

    __call__: Callable[[Self, StreamChunk], None]

    def __init__(
        self,
        __call__: Callable[[StreamChunk], None],
        modes: set[StreamMode],
    ) -> None:
        self.__call__ = cast(Callable[[Self, StreamChunk], None], __call__)
        self.modes = modes

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/remote.py
```py
from __future__ import annotations

import logging
from collections.abc import AsyncIterator, Iterator, Sequence
from dataclasses import asdict
from typing import (
    Any,
    Literal,
    cast,
)
from uuid import UUID

import langsmith as ls
from langchain_core.runnables import RunnableConfig
from langchain_core.runnables.graph import (
    Edge as DrawableEdge,
)
from langchain_core.runnables.graph import (
    Graph as DrawableGraph,
)
from langchain_core.runnables.graph import (
    Node as DrawableNode,
)
from langgraph.checkpoint.base import CheckpointMetadata
from langgraph_sdk.client import (
    LangGraphClient,
    SyncLangGraphClient,
    get_client,
    get_sync_client,
)
from langgraph_sdk.schema import (
    Checkpoint,
    QueryParamTypes,
    ThreadState,
)
from langgraph_sdk.schema import (
    Command as CommandSDK,
)
from langgraph_sdk.schema import (
    StreamMode as StreamModeSDK,
)
from typing_extensions import Self

from langgraph._internal._config import merge_configs
from langgraph._internal._constants import (
    CONF,
    CONFIG_KEY_CHECKPOINT_ID,
    CONFIG_KEY_CHECKPOINT_MAP,
    CONFIG_KEY_CHECKPOINT_NS,
    CONFIG_KEY_STREAM,
    CONFIG_KEY_TASK_ID,
    INTERRUPT,
    NS_SEP,
)
from langgraph.errors import GraphInterrupt, ParentCommand
from langgraph.pregel.protocol import PregelProtocol, StreamProtocol
from langgraph.types import (
    All,
    Command,
    Interrupt,
    PregelTask,
    StateSnapshot,
    StreamMode,
)

logger = logging.getLogger(__name__)

__all__ = ("RemoteGraph", "RemoteException")

_CONF_DROPLIST = frozenset(
    (
        CONFIG_KEY_CHECKPOINT_MAP,
        CONFIG_KEY_CHECKPOINT_ID,
        CONFIG_KEY_CHECKPOINT_NS,
        CONFIG_KEY_TASK_ID,
    ),
)


def _sanitize_config_value(v: Any) -> Any:
    """Recursively sanitize a config value to ensure it contains only primitives."""
    if isinstance(v, (str, int, float, bool, UUID)):
        return v
    elif isinstance(v, dict):
        sanitized_dict = {}
        for k, val in v.items():
            if isinstance(k, str):
                sanitized_value = _sanitize_config_value(val)
                if sanitized_value is not None:
                    sanitized_dict[k] = sanitized_value
        return sanitized_dict
    elif isinstance(v, (list, tuple)):
        sanitized_list = []
        for item in v:
            sanitized_item = _sanitize_config_value(item)
            if sanitized_item is not None:
                sanitized_list.append(sanitized_item)
        return sanitized_list
    return None


class RemoteException(Exception):
    """Exception raised when an error occurs in the remote graph."""

    pass


class RemoteGraph(PregelProtocol):
    """The `RemoteGraph` class is a client implementation for calling remote
    APIs that implement the LangGraph Server API specification.

    For example, the `RemoteGraph` class can be used to call APIs from deployments
    on LangSmith Deployment.

    `RemoteGraph` behaves the same way as a `Graph` and can be used directly as
    a node in another `Graph`.
    """

    assistant_id: str
    name: str | None

    def __init__(
        self,
        assistant_id: str,  # graph_id
        /,
        *,
        url: str | None = None,
        api_key: str | None = None,
        headers: dict[str, str] | None = None,
        client: LangGraphClient | None = None,
        sync_client: SyncLangGraphClient | None = None,
        config: RunnableConfig | None = None,
        name: str | None = None,
        distributed_tracing: bool = False,
    ):
        """Specify `url`, `api_key`, and/or `headers` to create default sync and async clients.

        If `client` or `sync_client` are provided, they will be used instead of the default clients.
        See `LangGraphClient` and `SyncLangGraphClient` for details on the default clients. At least
        one of `url`, `client`, or `sync_client` must be provided.

        Args:
            assistant_id: The assistant ID or graph name of the remote graph to use.
            url: The URL of the remote API.
            api_key: The API key to use for authentication. If not provided, it will be read from the environment (`LANGGRAPH_API_KEY`, `LANGSMITH_API_KEY`, or `LANGCHAIN_API_KEY`).
            headers: Additional headers to include in the requests.
            client: A `LangGraphClient` instance to use instead of creating a default client.
            sync_client: A `SyncLangGraphClient` instance to use instead of creating a default client.
            config: An optional `RunnableConfig` instance with additional configuration.
            name: Human-readable name to attach to the RemoteGraph instance.
                This is useful for adding `RemoteGraph` as a subgraph via `graph.add_node(remote_graph)`.
                If not provided, defaults to the assistant ID.
            distributed_tracing: Whether to enable sending LangSmith distributed tracing headers.
        """
        self.assistant_id = assistant_id
        if name is None:
            self.name = assistant_id
        else:
            self.name = name
        self.config = config
        self.distributed_tracing = distributed_tracing

        if client is None and url is not None:
            client = get_client(url=url, api_key=api_key, headers=headers)
        self.client = client

        if sync_client is None and url is not None:
            sync_client = get_sync_client(url=url, api_key=api_key, headers=headers)
        self.sync_client = sync_client

    def _validate_client(self) -> LangGraphClient:
        if self.client is None:
            raise ValueError(
                "Async client is not initialized: please provide `url` or `client` when initializing `RemoteGraph`."
            )
        return self.client

    def _validate_sync_client(self) -> SyncLangGraphClient:
        if self.sync_client is None:
            raise ValueError(
                "Sync client is not initialized: please provide `url` or `sync_client` when initializing `RemoteGraph`."
            )
        return self.sync_client

    def copy(self, update: dict[str, Any]) -> Self:
        attrs = {**self.__dict__, **update}
        return self.__class__(attrs.pop("assistant_id"), **attrs)

    def with_config(self, config: RunnableConfig | None = None, **kwargs: Any) -> Self:
        return self.copy(
            {"config": merge_configs(self.config, config, cast(RunnableConfig, kwargs))}
        )

    def _get_drawable_nodes(
        self, graph: dict[str, list[dict[str, Any]]]
    ) -> dict[str, DrawableNode]:
        nodes = {}
        for node in graph["nodes"]:
            node_id = str(node["id"])
            node_data = node.get("data", {})

            # Get node name from node_data if available. If not, use node_id.
            node_name = node.get("name")
            if node_name is None:
                if isinstance(node_data, dict):
                    node_name = node_data.get("name", node_id)
                else:
                    node_name = node_id

            nodes[node_id] = DrawableNode(
                id=node_id,
                name=node_name,
                data=node_data,
                metadata=node.get("metadata"),
            )
        return nodes

    def get_graph(
        self,
        config: RunnableConfig | None = None,
        *,
        xray: int | bool = False,
        headers: dict[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> DrawableGraph:
        """Get graph by graph name.

        This method calls `GET /assistants/{assistant_id}/graph`.

        Args:
            config: This parameter is not used.
            xray: Include graph representation of subgraphs. If an integer
                value is provided, only subgraphs with a depth less than or
                equal to the value will be included.

        Returns:
            The graph information for the assistant in JSON format.
        """
        sync_client = self._validate_sync_client()
        graph = sync_client.assistants.get_graph(
            assistant_id=self.assistant_id,
            xray=xray,
            headers=headers,
            params=params,
        )
        return DrawableGraph(
            nodes=self._get_drawable_nodes(graph),
            edges=[DrawableEdge(**edge) for edge in graph["edges"]],
        )

    async def aget_graph(
        self,
        config: RunnableConfig | None = None,
        *,
        xray: int | bool = False,
        headers: dict[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> DrawableGraph:
        """Get graph by graph name.

        This method calls `GET /assistants/{assistant_id}/graph`.

        Args:
            config: This parameter is not used.
            xray: Include graph representation of subgraphs. If an integer
                value is provided, only subgraphs with a depth less than or
                equal to the value will be included.

        Returns:
            The graph information for the assistant in JSON format.
        """
        client = self._validate_client()
        graph = await client.assistants.get_graph(
            assistant_id=self.assistant_id,
            xray=xray,
            headers=headers,
            params=params,
        )
        return DrawableGraph(
            nodes=self._get_drawable_nodes(graph),
            edges=[DrawableEdge(**edge) for edge in graph["edges"]],
        )

    def _create_state_snapshot(self, state: ThreadState) -> StateSnapshot:
        tasks: list[PregelTask] = []
        for task in state["tasks"]:
            interrupts = tuple(
                Interrupt(**interrupt) for interrupt in task["interrupts"]
            )

            tasks.append(
                PregelTask(
                    id=task["id"],
                    name=task["name"],
                    path=tuple(),
                    error=Exception(task["error"]) if task["error"] else None,
                    interrupts=interrupts,
                    state=(
                        self._create_state_snapshot(task["state"])
                        if task["state"]
                        else (
                            cast(RunnableConfig, {"configurable": task["checkpoint"]})
                            if task["checkpoint"]
                            else None
                        )
                    ),
                    result=task.get("result"),
                )
            )

        return StateSnapshot(
            values=state["values"],
            next=tuple(state["next"]) if state["next"] else tuple(),
            config={
                "configurable": {
                    "thread_id": state["checkpoint"]["thread_id"],
                    "checkpoint_ns": state["checkpoint"]["checkpoint_ns"],
                    "checkpoint_id": state["checkpoint"]["checkpoint_id"],
                    "checkpoint_map": state["checkpoint"].get("checkpoint_map", {}),
                }
            },
            metadata=CheckpointMetadata(**state["metadata"]),
            created_at=state["created_at"],
            parent_config=(
                {
                    "configurable": {
                        "thread_id": state["parent_checkpoint"]["thread_id"],
                        "checkpoint_ns": state["parent_checkpoint"]["checkpoint_ns"],
                        "checkpoint_id": state["parent_checkpoint"]["checkpoint_id"],
                        "checkpoint_map": state["parent_checkpoint"].get(
                            "checkpoint_map", {}
                        ),
                    }
                }
                if state["parent_checkpoint"]
                else None
            ),
            tasks=tuple(tasks),
            interrupts=tuple([i for task in tasks for i in task.interrupts]),
        )

    def _get_checkpoint(self, config: RunnableConfig | None) -> Checkpoint | None:
        if config is None:
            return None

        checkpoint = {}

        if "thread_id" in config["configurable"]:
            checkpoint["thread_id"] = config["configurable"]["thread_id"]
        if "checkpoint_ns" in config["configurable"]:
            checkpoint["checkpoint_ns"] = config["configurable"]["checkpoint_ns"]
        if "checkpoint_id" in config["configurable"]:
            checkpoint["checkpoint_id"] = config["configurable"]["checkpoint_id"]
        if "checkpoint_map" in config["configurable"]:
            checkpoint["checkpoint_map"] = config["configurable"]["checkpoint_map"]

        return checkpoint if checkpoint else None

    def _get_config(self, checkpoint: Checkpoint) -> RunnableConfig:
        return {
            "configurable": {
                "thread_id": checkpoint["thread_id"],
                "checkpoint_ns": checkpoint["checkpoint_ns"],
                "checkpoint_id": checkpoint["checkpoint_id"],
                "checkpoint_map": checkpoint.get("checkpoint_map", {}),
            }
        }

    def _sanitize_config(self, config: RunnableConfig) -> RunnableConfig:
        """Sanitize the config to remove non-serializable fields."""
        sanitized: RunnableConfig = {}
        if "recursion_limit" in config:
            sanitized["recursion_limit"] = config["recursion_limit"]
        if "tags" in config:
            sanitized["tags"] = [tag for tag in config["tags"] if isinstance(tag, str)]

        if "metadata" in config:
            sanitized["metadata"] = {}
            for k, v in config["metadata"].items():
                if (
                    isinstance(k, str)
                    and (sanitized_value := _sanitize_config_value(v)) is not None
                ):
                    sanitized["metadata"][k] = sanitized_value

        if "configurable" in config:
            sanitized["configurable"] = {}
            for k, v in config["configurable"].items():
                if (
                    isinstance(k, str)
                    and k not in _CONF_DROPLIST
                    and (sanitized_value := _sanitize_config_value(v)) is not None
                ):
                    sanitized["configurable"][k] = sanitized_value

        return sanitized

    def get_state(
        self,
        config: RunnableConfig,
        *,
        subgraphs: bool = False,
        headers: dict[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> StateSnapshot:
        """Get the state of a thread.

        This method calls `POST /threads/{thread_id}/state/checkpoint` if a
        checkpoint is specified in the config or `GET /threads/{thread_id}/state`
        if no checkpoint is specified.

        Args:
            config: A `RunnableConfig` that includes `thread_id` in the
                `configurable` field.
            subgraphs: Include subgraphs in the state.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            The latest state of the thread.
        """
        sync_client = self._validate_sync_client()
        merged_config = merge_configs(self.config, config)

        state = sync_client.threads.get_state(
            thread_id=merged_config["configurable"]["thread_id"],
            checkpoint=self._get_checkpoint(merged_config),
            subgraphs=subgraphs,
            headers=headers,
            params=params,
        )
        return self._create_state_snapshot(state)

    async def aget_state(
        self,
        config: RunnableConfig,
        *,
        subgraphs: bool = False,
        headers: dict[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> StateSnapshot:
        """Get the state of a thread.

        This method calls `POST /threads/{thread_id}/state/checkpoint` if a
        checkpoint is specified in the config or `GET /threads/{thread_id}/state`
        if no checkpoint is specified.

        Args:
            config: A `RunnableConfig` that includes `thread_id` in the
                `configurable` field.
            subgraphs: Include subgraphs in the state.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            The latest state of the thread.
        """
        client = self._validate_client()
        merged_config = merge_configs(self.config, config)

        state = await client.threads.get_state(
            thread_id=merged_config["configurable"]["thread_id"],
            checkpoint=self._get_checkpoint(merged_config),
            subgraphs=subgraphs,
            headers=headers,
            params=params,
        )
        return self._create_state_snapshot(state)

    def get_state_history(
        self,
        config: RunnableConfig,
        *,
        filter: dict[str, Any] | None = None,
        before: RunnableConfig | None = None,
        limit: int | None = None,
        headers: dict[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> Iterator[StateSnapshot]:
        """Get the state history of a thread.

        This method calls `POST /threads/{thread_id}/history`.

        Args:
            config: A `RunnableConfig` that includes `thread_id` in the
                `configurable` field.
            filter: Metadata to filter on.
            before: A `RunnableConfig` that includes checkpoint metadata.
            limit: Max number of states to return.

        Returns:
            States of the thread.
        """
        sync_client = self._validate_sync_client()
        merged_config = merge_configs(self.config, config)

        states = sync_client.threads.get_history(
            thread_id=merged_config["configurable"]["thread_id"],
            limit=limit if limit else 10,
            before=self._get_checkpoint(before),
            metadata=filter,
            checkpoint=self._get_checkpoint(merged_config),
            headers=headers,
            params=params,
        )
        for state in states:
            yield self._create_state_snapshot(state)

    async def aget_state_history(
        self,
        config: RunnableConfig,
        *,
        filter: dict[str, Any] | None = None,
        before: RunnableConfig | None = None,
        limit: int | None = None,
        headers: dict[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> AsyncIterator[StateSnapshot]:
        """Get the state history of a thread.

        This method calls `POST /threads/{thread_id}/history`.

        Args:
            config: A `RunnableConfig` that includes `thread_id` in the
                `configurable` field.
            filter: Metadata to filter on.
            before: A `RunnableConfig` that includes checkpoint metadata.
            limit: Max number of states to return.
            headers: Optional custom headers to include with the request.
            params: Optional query parameters to include with the request.

        Returns:
            States of the thread.
        """
        client = self._validate_client()
        merged_config = merge_configs(self.config, config)

        states = await client.threads.get_history(
            thread_id=merged_config["configurable"]["thread_id"],
            limit=limit if limit else 10,
            before=self._get_checkpoint(before),
            metadata=filter,
            checkpoint=self._get_checkpoint(merged_config),
            headers=headers,
            params=params,
        )
        for state in states:
            yield self._create_state_snapshot(state)

    def bulk_update_state(
        self,
        config: RunnableConfig,
        updates: list[tuple[dict[str, Any] | None, str | None]],
    ) -> RunnableConfig:
        raise NotImplementedError

    async def abulk_update_state(
        self,
        config: RunnableConfig,
        updates: list[tuple[dict[str, Any] | None, str | None]],
    ) -> RunnableConfig:
        raise NotImplementedError

    def update_state(
        self,
        config: RunnableConfig,
        values: dict[str, Any] | Any | None,
        as_node: str | None = None,
        *,
        headers: dict[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> RunnableConfig:
        """Update the state of a thread.

        This method calls `POST /threads/{thread_id}/state`.

        Args:
            config: A `RunnableConfig` that includes `thread_id` in the
                `configurable` field.
            values: Values to update to the state.
            as_node: Update the state as if this node had just executed.

        Returns:
            `RunnableConfig` for the updated thread.
        """
        sync_client = self._validate_sync_client()
        merged_config = merge_configs(self.config, config)

        response: dict = sync_client.threads.update_state(  # type: ignore
            thread_id=merged_config["configurable"]["thread_id"],
            values=values,
            as_node=as_node,
            checkpoint=self._get_checkpoint(merged_config),
            headers=headers,
            params=params,
        )
        return self._get_config(response["checkpoint"])

    async def aupdate_state(
        self,
        config: RunnableConfig,
        values: dict[str, Any] | Any | None,
        as_node: str | None = None,
        *,
        headers: dict[str, str] | None = None,
        params: QueryParamTypes | None = None,
    ) -> RunnableConfig:
        """Update the state of a thread.

        This method calls `POST /threads/{thread_id}/state`.

        Args:
            config: A `RunnableConfig` that includes `thread_id` in the
                `configurable` field.
            values: Values to update to the state.
            as_node: Update the state as if this node had just executed.

        Returns:
            `RunnableConfig` for the updated thread.
        """
        client = self._validate_client()
        merged_config = merge_configs(self.config, config)

        response: dict = await client.threads.update_state(  # type: ignore
            thread_id=merged_config["configurable"]["thread_id"],
            values=values,
            as_node=as_node,
            checkpoint=self._get_checkpoint(merged_config),
            headers=headers,
            params=params,
        )
        return self._get_config(response["checkpoint"])

    def _get_stream_modes(
        self,
        stream_mode: StreamMode | list[StreamMode] | None,
        config: RunnableConfig | None,
        default: StreamMode = "updates",
    ) -> tuple[list[StreamModeSDK], list[StreamModeSDK], bool, StreamProtocol | None]:
        """Return a tuple of the final list of stream modes sent to the
        remote graph and a boolean flag indicating if stream mode 'updates'
        was present in the original list of stream modes.

        'updates' mode is added to the list of stream modes so that interrupts
        can be detected in the remote graph.
        """
        updated_stream_modes: list[StreamModeSDK] = []
        req_single = True
        # coerce to list, or add default stream mode
        if stream_mode:
            if isinstance(stream_mode, str):
                updated_stream_modes.append(stream_mode)
            else:
                req_single = False
                updated_stream_modes.extend(stream_mode)
        else:
            updated_stream_modes.append(default)
        requested_stream_modes = updated_stream_modes.copy()
        # add any from parent graph
        stream: StreamProtocol | None = (
            (config or {}).get(CONF, {}).get(CONFIG_KEY_STREAM)
        )
        if stream:
            updated_stream_modes.extend(stream.modes)
        # map "messages" to "messages-tuple"
        if "messages" in updated_stream_modes:
            updated_stream_modes.remove("messages")
            updated_stream_modes.append("messages-tuple")

        # if requested "messages-tuple",
        # map to "messages" in requested_stream_modes
        if "messages-tuple" in requested_stream_modes:
            requested_stream_modes.remove("messages-tuple")
            requested_stream_modes.append("messages")

        # add 'updates' mode if not present
        if "updates" not in updated_stream_modes:
            updated_stream_modes.append("updates")

        # remove 'events', as it's not supported in Pregel
        if "events" in updated_stream_modes:
            updated_stream_modes.remove("events")
        return (updated_stream_modes, requested_stream_modes, req_single, stream)

    def stream(
        self,
        input: dict[str, Any] | Any,
        config: RunnableConfig | None = None,
        *,
        stream_mode: StreamMode | list[StreamMode] | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        subgraphs: bool = False,
        headers: dict[str, str] | None = None,
        params: QueryParamTypes | None = None,
        **kwargs: Any,
    ) -> Iterator[dict[str, Any] | Any]:
        """Create a run and stream the results.

        This method calls `POST /threads/{thread_id}/runs/stream` if a `thread_id`
        is speciffed in the `configurable` field of the config or
        `POST /runs/stream` otherwise.

        Args:
            input: Input to the graph.
            config: A `RunnableConfig` for graph invocation.
            stream_mode: Stream mode(s) to use.
            interrupt_before: Interrupt the graph before these nodes.
            interrupt_after: Interrupt the graph after these nodes.
            subgraphs: Stream from subgraphs.
            headers: Additional headers to pass to the request.
            **kwargs: Additional params to pass to client.runs.stream.

        Yields:
            The output of the graph.
        """
        sync_client = self._validate_sync_client()
        merged_config = merge_configs(self.config, config)
        sanitized_config = self._sanitize_config(merged_config)
        stream_modes, requested, req_single, stream = self._get_stream_modes(
            stream_mode, config
        )
        if isinstance(input, Command):
            command: CommandSDK | None = cast(CommandSDK, asdict(input))
            input = None
        else:
            command = None

        for chunk in sync_client.runs.stream(
            thread_id=sanitized_config["configurable"].get("thread_id"),
            assistant_id=self.assistant_id,
            input=input,
            command=command,
            config=sanitized_config,
            stream_mode=stream_modes,
            interrupt_before=interrupt_before,
            interrupt_after=interrupt_after,
            stream_subgraphs=subgraphs or stream is not None,
            if_not_exists="create",
            headers=(
                _merge_tracing_headers(headers) if self.distributed_tracing else headers
            ),
            params=params,
            **kwargs,
        ):
            # split mode and ns
            if NS_SEP in chunk.event:
                mode, ns_ = chunk.event.split(NS_SEP, 1)
                ns = tuple(ns_.split(NS_SEP))
            else:
                mode, ns = chunk.event, ()
            # raise ParentCommand exception for command events
            if mode == "command" and chunk.data.get("graph") == Command.PARENT:
                raise ParentCommand(Command(**chunk.data))
            # prepend caller ns (as it is not passed to remote graph)
            if caller_ns := (config or {}).get(CONF, {}).get(CONFIG_KEY_CHECKPOINT_NS):
                caller_ns = tuple(caller_ns.split(NS_SEP))
                ns = caller_ns + ns
            # stream to parent stream
            if stream is not None and mode in stream.modes:
                stream((ns, mode, chunk.data))
            # raise interrupt or errors
            if chunk.event.startswith("updates"):
                if isinstance(chunk.data, dict) and INTERRUPT in chunk.data:
                    if caller_ns:
                        raise GraphInterrupt(
                            [Interrupt(**i) for i in chunk.data[INTERRUPT]]
                        )
            elif chunk.event.startswith("error"):
                raise RemoteException(chunk.data)
            # filter for what was actually requested
            if mode not in requested:
                continue

            if chunk.event.startswith("messages"):
                chunk = chunk._replace(data=tuple(chunk.data))  # type: ignore

            # emit chunk
            if subgraphs:
                if NS_SEP in chunk.event:
                    mode, ns_ = chunk.event.split(NS_SEP, 1)
                    ns = tuple(ns_.split(NS_SEP))
                else:
                    mode, ns = chunk.event, ()
                if req_single:
                    yield ns, chunk.data
                else:
                    yield ns, mode, chunk.data
            elif req_single:
                yield chunk.data
            else:
                yield chunk

    async def astream(
        self,
        input: dict[str, Any] | Any,
        config: RunnableConfig | None = None,
        *,
        stream_mode: StreamMode | list[StreamMode] | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        subgraphs: bool = False,
        headers: dict[str, str] | None = None,
        params: QueryParamTypes | None = None,
        **kwargs: Any,
    ) -> AsyncIterator[dict[str, Any] | Any]:
        """Create a run and stream the results.

        This method calls `POST /threads/{thread_id}/runs/stream` if a `thread_id`
        is speciffed in the `configurable` field of the config or
        `POST /runs/stream` otherwise.

        Args:
            input: Input to the graph.
            config: A `RunnableConfig` for graph invocation.
            stream_mode: Stream mode(s) to use.
            interrupt_before: Interrupt the graph before these nodes.
            interrupt_after: Interrupt the graph after these nodes.
            subgraphs: Stream from subgraphs.
            headers: Additional headers to pass to the request.
            **kwargs: Additional params to pass to client.runs.stream.

        Yields:
            The output of the graph.
        """
        client = self._validate_client()
        merged_config = merge_configs(self.config, config)
        sanitized_config = self._sanitize_config(merged_config)
        stream_modes, requested, req_single, stream = self._get_stream_modes(
            stream_mode, config
        )
        if isinstance(input, Command):
            command: CommandSDK | None = cast(CommandSDK, asdict(input))
            input = None
        else:
            command = None

        async for chunk in client.runs.stream(
            thread_id=sanitized_config["configurable"].get("thread_id"),
            assistant_id=self.assistant_id,
            input=input,
            command=command,
            config=sanitized_config,
            stream_mode=stream_modes,
            interrupt_before=interrupt_before,
            interrupt_after=interrupt_after,
            stream_subgraphs=subgraphs or stream is not None,
            if_not_exists="create",
            headers=(
                _merge_tracing_headers(headers) if self.distributed_tracing else headers
            ),
            params=params,
            **kwargs,
        ):
            # split mode and ns
            if NS_SEP in chunk.event:
                mode, ns_ = chunk.event.split(NS_SEP, 1)
                ns = tuple(ns_.split(NS_SEP))
            else:
                mode, ns = chunk.event, ()
            # raise ParentCommand exception for command events
            if mode == "command" and chunk.data.get("graph") == Command.PARENT:
                raise ParentCommand(Command(**chunk.data))
            # prepend caller ns (as it is not passed to remote graph)
            if caller_ns := (config or {}).get(CONF, {}).get(CONFIG_KEY_CHECKPOINT_NS):
                caller_ns = tuple(caller_ns.split(NS_SEP))
                ns = caller_ns + ns
            # stream to parent stream
            if stream is not None and mode in stream.modes:
                stream((ns, mode, chunk.data))
            # raise interrupt or errors
            if chunk.event.startswith("updates"):
                if isinstance(chunk.data, dict) and INTERRUPT in chunk.data:
                    if caller_ns:
                        raise GraphInterrupt(
                            [Interrupt(**i) for i in chunk.data[INTERRUPT]]
                        )
            elif chunk.event.startswith("error"):
                raise RemoteException(chunk.data)
            # filter for what was actually requested
            if mode not in requested:
                continue

            if chunk.event.startswith("messages"):
                chunk = chunk._replace(data=tuple(chunk.data))  # type: ignore

            # emit chunk
            if subgraphs:
                if NS_SEP in chunk.event:
                    mode, ns_ = chunk.event.split(NS_SEP, 1)
                    ns = tuple(ns_.split(NS_SEP))
                else:
                    mode, ns = chunk.event, ()
                if req_single:
                    yield ns, chunk.data
                else:
                    yield ns, mode, chunk.data
            elif req_single:
                yield chunk.data
            else:
                yield chunk

    async def astream_events(
        self,
        input: Any,
        config: RunnableConfig | None = None,
        *,
        version: Literal["v1", "v2"],
        include_names: Sequence[All] | None = None,
        include_types: Sequence[All] | None = None,
        include_tags: Sequence[All] | None = None,
        exclude_names: Sequence[All] | None = None,
        exclude_types: Sequence[All] | None = None,
        exclude_tags: Sequence[All] | None = None,
        **kwargs: Any,
    ) -> AsyncIterator[dict[str, Any]]:
        raise NotImplementedError

    def invoke(
        self,
        input: dict[str, Any] | Any,
        config: RunnableConfig | None = None,
        *,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        headers: dict[str, str] | None = None,
        params: QueryParamTypes | None = None,
        **kwargs: Any,
    ) -> dict[str, Any] | Any:
        """Create a run, wait until it finishes and return the final state.

        Args:
            input: Input to the graph.
            config: A `RunnableConfig` for graph invocation.
            interrupt_before: Interrupt the graph before these nodes.
            interrupt_after: Interrupt the graph after these nodes.
            headers: Additional headers to pass to the request.
            **kwargs: Additional params to pass to RemoteGraph.stream.

        Returns:
            The output of the graph.
        """
        for chunk in self.stream(
            input,
            config=config,
            interrupt_before=interrupt_before,
            interrupt_after=interrupt_after,
            headers=headers,
            stream_mode="values",
            params=params,
            **kwargs,
        ):
            pass
        try:
            return chunk
        except UnboundLocalError:
            logger.warning("No events received from remote graph")
            return None

    async def ainvoke(
        self,
        input: dict[str, Any] | Any,
        config: RunnableConfig | None = None,
        *,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        headers: dict[str, str] | None = None,
        params: QueryParamTypes | None = None,
        **kwargs: Any,
    ) -> dict[str, Any] | Any:
        """Create a run, wait until it finishes and return the final state.

        Args:
            input: Input to the graph.
            config: A `RunnableConfig` for graph invocation.
            interrupt_before: Interrupt the graph before these nodes.
            interrupt_after: Interrupt the graph after these nodes.
            headers: Additional headers to pass to the request.
            **kwargs: Additional params to pass to RemoteGraph.astream.

        Returns:
            The output of the graph.
        """
        async for chunk in self.astream(
            input,
            config=config,
            interrupt_before=interrupt_before,
            interrupt_after=interrupt_after,
            headers=headers,
            stream_mode="values",
            params=params,
            **kwargs,
        ):
            pass
        try:
            return chunk
        except UnboundLocalError:
            logger.warning("No events received from remote graph")
            return None


def _merge_tracing_headers(headers: dict[str, str] | None) -> dict[str, str] | None:
    if rt := ls.get_current_run_tree():
        tracing_headers = rt.to_headers()
        if headers:
            if "baggage" in headers:
                tracing_headers["baggage"] = (
                    f"{headers['baggage']},{tracing_headers['baggage']}"
                )
            headers.update(tracing_headers)
        else:
            headers = tracing_headers
    return headers

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/types.py
```py
"""Re-export types moved to langgraph.types"""

from langgraph.types import (
    All,
    CachePolicy,
    PregelExecutableTask,
    PregelTask,
    RetryPolicy,
    StateSnapshot,
    StateUpdate,
    StreamMode,
    StreamWriter,
    default_retry_on,
)

__all__ = [
    "All",
    "StateUpdate",
    "CachePolicy",
    "PregelExecutableTask",
    "PregelTask",
    "RetryPolicy",
    "StateSnapshot",
    "StreamMode",
    "StreamWriter",
    "default_retry_on",
]

from warnings import warn

from langgraph.warnings import LangGraphDeprecatedSinceV10

warn(
    "Importing from langgraph.pregel.types is deprecated. "
    "Please use 'from langgraph.types import ...' instead.",
    LangGraphDeprecatedSinceV10,
    stacklevel=2,
)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/_internal/__init__.py
```py
"""Internal modules for LangGraph.

This module is not part of the public API, and thus stability is not guaranteed.
"""

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/_internal/_cache.py
```py
from __future__ import annotations

from collections.abc import Hashable, Mapping, Sequence
from typing import Any


def _freeze(obj: Any, depth: int = 10) -> Hashable:
    if isinstance(obj, Hashable) or depth <= 0:
        # already hashable, no need to freeze
        return obj
    elif isinstance(obj, Mapping):
        # sort keys so {"a":1,"b":2} == {"b":2,"a":1}
        return tuple(sorted((k, _freeze(v, depth - 1)) for k, v in obj.items()))
    elif isinstance(obj, Sequence):
        return tuple(_freeze(x, depth - 1) for x in obj)
    # numpy / pandas etc. can provide their own .tobytes()
    elif hasattr(obj, "tobytes"):
        return (
            type(obj).__name__,
            obj.tobytes(),
            obj.shape if hasattr(obj, "shape") else None,
        )
    return obj  # strings, ints, dataclasses with frozen=True, etc.


def default_cache_key(*args: Any, **kwargs: Any) -> str | bytes:
    """Default cache key function that uses the arguments and keyword arguments to generate a hashable key."""
    import pickle

    # protocol 5 strikes a good balance between speed and size
    return pickle.dumps((_freeze(args), _freeze(kwargs)), protocol=5, fix_imports=False)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/_internal/_config.py
```py
from __future__ import annotations

from collections import ChainMap
from collections.abc import Mapping, Sequence
from os import getenv
from typing import Any, cast

from langchain_core.callbacks import (
    AsyncCallbackManager,
    BaseCallbackManager,
    CallbackManager,
    Callbacks,
)
from langchain_core.runnables import RunnableConfig
from langchain_core.runnables.config import (
    CONFIG_KEYS,
    COPIABLE_KEYS,
    var_child_runnable_config,
)
from langgraph.checkpoint.base import CheckpointMetadata

from langgraph._internal._constants import (
    CONF,
    CONFIG_KEY_CHECKPOINT_ID,
    CONFIG_KEY_CHECKPOINT_MAP,
    CONFIG_KEY_CHECKPOINT_NS,
    NS_END,
    NS_SEP,
)

DEFAULT_RECURSION_LIMIT = int(getenv("LANGGRAPH_DEFAULT_RECURSION_LIMIT", "25"))


def recast_checkpoint_ns(ns: str) -> str:
    """Remove task IDs from checkpoint namespace.

    Args:
        ns: The checkpoint namespace with task IDs.

    Returns:
        str: The checkpoint namespace without task IDs.
    """
    return NS_SEP.join(
        part.split(NS_END)[0] for part in ns.split(NS_SEP) if not part.isdigit()
    )


def patch_configurable(
    config: RunnableConfig | None, patch: dict[str, Any]
) -> RunnableConfig:
    if config is None:
        return {CONF: patch}
    elif CONF not in config:
        return {**config, CONF: patch}
    else:
        return {**config, CONF: {**config[CONF], **patch}}


def patch_checkpoint_map(
    config: RunnableConfig | None, metadata: CheckpointMetadata | None
) -> RunnableConfig:
    if config is None:
        return config
    elif parents := (metadata.get("parents") if metadata else None):
        conf = config[CONF]
        return patch_configurable(
            config,
            {
                CONFIG_KEY_CHECKPOINT_MAP: {
                    **parents,
                    conf[CONFIG_KEY_CHECKPOINT_NS]: conf[CONFIG_KEY_CHECKPOINT_ID],
                },
            },
        )
    else:
        return config


def merge_configs(*configs: RunnableConfig | None) -> RunnableConfig:
    """Merge multiple configs into one.

    Args:
        *configs: The configs to merge.

    Returns:
        RunnableConfig: The merged config.
    """
    base: RunnableConfig = {}
    # Even though the keys aren't literals, this is correct
    # because both dicts are the same type
    for config in configs:
        if config is None:
            continue
        for key, value in config.items():
            if not value:
                continue
            if key == "metadata":
                if base_value := base.get(key):
                    base[key] = {**base_value, **value}  # type: ignore
                else:
                    base[key] = value  # type: ignore[literal-required]
            elif key == "tags":
                if base_value := base.get(key):
                    base[key] = [*base_value, *value]  # type: ignore
                else:
                    base[key] = value  # type: ignore[literal-required]
            elif key == CONF:
                if base_value := base.get(key):
                    base[key] = {**base_value, **value}  # type: ignore[dict-item]
                else:
                    base[key] = value
            elif key == "callbacks":
                base_callbacks = base.get("callbacks")
                # callbacks can be either None, list[handler] or manager
                # so merging two callbacks values has 6 cases
                if isinstance(value, list):
                    if base_callbacks is None:
                        base["callbacks"] = value.copy()
                    elif isinstance(base_callbacks, list):
                        base["callbacks"] = base_callbacks + value
                    else:
                        # base_callbacks is a manager
                        mngr = base_callbacks.copy()
                        for callback in value:
                            mngr.add_handler(callback, inherit=True)
                        base["callbacks"] = mngr
                elif isinstance(value, BaseCallbackManager):
                    # value is a manager
                    if base_callbacks is None:
                        base["callbacks"] = value.copy()
                    elif isinstance(base_callbacks, list):
                        mngr = value.copy()
                        for callback in base_callbacks:
                            mngr.add_handler(callback, inherit=True)
                        base["callbacks"] = mngr
                    else:
                        # base_callbacks is also a manager
                        base["callbacks"] = base_callbacks.merge(value)
                else:
                    raise NotImplementedError
            elif key == "recursion_limit":
                if config["recursion_limit"] != DEFAULT_RECURSION_LIMIT:
                    base["recursion_limit"] = config["recursion_limit"]
            else:
                base[key] = config[key]  # type: ignore[literal-required]
    if CONF not in base:
        base[CONF] = {}
    return base


def patch_config(
    config: RunnableConfig | None,
    *,
    callbacks: Callbacks = None,
    recursion_limit: int | None = None,
    max_concurrency: int | None = None,
    run_name: str | None = None,
    configurable: dict[str, Any] | None = None,
) -> RunnableConfig:
    """Patch a config with new values.

    Args:
        config: The config to patch.
        callbacks: The callbacks to set.
        recursion_limit: The recursion limit to set.
        max_concurrency: The max number of concurrent steps to run, which also applies to parallelized steps.
        run_name: The run name to set.
        configurable: The configurable to set.

    Returns:
        RunnableConfig: The patched config.
    """
    config = config.copy() if config is not None else {}
    if callbacks is not None:
        # If we're replacing callbacks, we need to unset run_name
        # As that should apply only to the same run as the original callbacks
        config["callbacks"] = callbacks
        if "run_name" in config:
            del config["run_name"]
        if "run_id" in config:
            del config["run_id"]
    if recursion_limit is not None:
        config["recursion_limit"] = recursion_limit
    if max_concurrency is not None:
        config["max_concurrency"] = max_concurrency
    if run_name is not None:
        config["run_name"] = run_name
    if configurable is not None:
        config[CONF] = {**config.get(CONF, {}), **configurable}
    return config


def get_callback_manager_for_config(
    config: RunnableConfig, tags: Sequence[str] | None = None
) -> CallbackManager:
    """Get a callback manager for a config.

    Args:
        config: The config.

    Returns:
        CallbackManager: The callback manager.
    """
    from langchain_core.callbacks.manager import CallbackManager

    # merge tags
    all_tags = config.get("tags")
    if all_tags is not None and tags is not None:
        all_tags = [*all_tags, *tags]
    elif tags is not None:
        all_tags = list(tags)
    # use existing callbacks if they exist
    if (callbacks := config.get("callbacks")) and isinstance(
        callbacks, CallbackManager
    ):
        if all_tags:
            callbacks.add_tags(all_tags)
        if metadata := config.get("metadata"):
            callbacks.add_metadata(metadata)
        return callbacks
    else:
        # otherwise create a new manager
        return CallbackManager.configure(
            inheritable_callbacks=config.get("callbacks"),
            inheritable_tags=all_tags,
            inheritable_metadata=config.get("metadata"),
        )


def get_async_callback_manager_for_config(
    config: RunnableConfig,
    tags: Sequence[str] | None = None,
) -> AsyncCallbackManager:
    """Get an async callback manager for a config.

    Args:
        config: The config.

    Returns:
        AsyncCallbackManager: The async callback manager.
    """
    from langchain_core.callbacks.manager import AsyncCallbackManager

    # merge tags
    all_tags = config.get("tags")
    if all_tags is not None and tags is not None:
        all_tags = [*all_tags, *tags]
    elif tags is not None:
        all_tags = list(tags)
    # use existing callbacks if they exist
    if (callbacks := config.get("callbacks")) and isinstance(
        callbacks, AsyncCallbackManager
    ):
        if all_tags:
            callbacks.add_tags(all_tags)
        if metadata := config.get("metadata"):
            callbacks.add_metadata(metadata)
        return callbacks
    else:
        # otherwise create a new manager
        return AsyncCallbackManager.configure(
            inheritable_callbacks=config.get("callbacks"),
            inheritable_tags=all_tags,
            inheritable_metadata=config.get("metadata"),
        )


def _is_not_empty(value: Any) -> bool:
    if isinstance(value, (list, tuple, dict)):
        return len(value) > 0
    else:
        return value is not None


def ensure_config(*configs: RunnableConfig | None) -> RunnableConfig:
    """Return a config with all keys, merging any provided configs.

    Args:
        *configs: Configs to merge before ensuring defaults.

    Returns:
        RunnableConfig: The merged and ensured config.
    """
    empty = RunnableConfig(
        tags=[],
        metadata=ChainMap(),
        callbacks=None,
        recursion_limit=DEFAULT_RECURSION_LIMIT,
        configurable={},
    )
    if var_config := var_child_runnable_config.get():
        empty.update(
            {
                k: v.copy() if k in COPIABLE_KEYS else v  # type: ignore[attr-defined]
                for k, v in var_config.items()
                if _is_not_empty(v)
            },
        )
    for config in configs:
        if config is None:
            continue
        for k, v in config.items():
            if _is_not_empty(v) and k in CONFIG_KEYS:
                if k == CONF:
                    empty[k] = cast(dict, v).copy()
                else:
                    empty[k] = v  # type: ignore[literal-required]
        for k, v in config.items():
            if _is_not_empty(v) and k not in CONFIG_KEYS:
                empty[CONF][k] = v
    _empty_metadata = empty["metadata"]
    for key, value in empty[CONF].items():
        if _exclude_as_metadata(key, value, _empty_metadata):
            continue
        _empty_metadata[key] = value
    return empty


_OMIT = ("key", "token", "secret", "password", "auth")


def _exclude_as_metadata(key: str, value: Any, metadata: Mapping[str, Any]) -> bool:
    key_lower = key.casefold()
    return (
        key.startswith("__")
        or not isinstance(value, (str, int, float, bool))
        or key in metadata
        or any(substr in key_lower for substr in _OMIT)
    )

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/_internal/_constants.py
```py
"""Constants used for Pregel operations."""

import sys
from typing import Literal, cast

# --- Reserved write keys ---
INPUT = sys.intern("__input__")
# for values passed as input to the graph
INTERRUPT = sys.intern("__interrupt__")
# for dynamic interrupts raised by nodes
RESUME = sys.intern("__resume__")
# for values passed to resume a node after an interrupt
ERROR = sys.intern("__error__")
# for errors raised by nodes
NO_WRITES = sys.intern("__no_writes__")
# marker to signal node didn't write anything
TASKS = sys.intern("__pregel_tasks")
# for Send objects returned by nodes/edges, corresponds to PUSH below
RETURN = sys.intern("__return__")
# for writes of a task where we simply record the return value
PREVIOUS = sys.intern("__previous__")
# the implicit branch that handles each node's Control values


# --- Reserved cache namespaces ---
CACHE_NS_WRITES = sys.intern("__pregel_ns_writes")
# cache namespace for node writes

# --- Reserved config.configurable keys ---
CONFIG_KEY_SEND = sys.intern("__pregel_send")
# holds the `write` function that accepts writes to state/edges/reserved keys
CONFIG_KEY_READ = sys.intern("__pregel_read")
# holds the `read` function that returns a copy of the current state
CONFIG_KEY_CALL = sys.intern("__pregel_call")
# holds the `call` function that accepts a node/func, args and returns a future
CONFIG_KEY_CHECKPOINTER = sys.intern("__pregel_checkpointer")
# holds a `BaseCheckpointSaver` passed from parent graph to child graphs
CONFIG_KEY_STREAM = sys.intern("__pregel_stream")
# holds a `StreamProtocol` passed from parent graph to child graphs
CONFIG_KEY_CACHE = sys.intern("__pregel_cache")
# holds a `BaseCache` made available to subgraphs
CONFIG_KEY_RESUMING = sys.intern("__pregel_resuming")
# holds a boolean indicating if subgraphs should resume from a previous checkpoint
CONFIG_KEY_TASK_ID = sys.intern("__pregel_task_id")
# holds the task ID for the current task
CONFIG_KEY_THREAD_ID = sys.intern("thread_id")
# holds the thread ID for the current invocation
CONFIG_KEY_CHECKPOINT_MAP = sys.intern("checkpoint_map")
# holds a mapping of checkpoint_ns -> checkpoint_id for parent graphs
CONFIG_KEY_CHECKPOINT_ID = sys.intern("checkpoint_id")
# holds the current checkpoint_id, if any
CONFIG_KEY_CHECKPOINT_NS = sys.intern("checkpoint_ns")
# holds the current checkpoint_ns, "" for root graph
CONFIG_KEY_NODE_FINISHED = sys.intern("__pregel_node_finished")
# holds a callback to be called when a node is finished
CONFIG_KEY_SCRATCHPAD = sys.intern("__pregel_scratchpad")
# holds a mutable dict for temporary storage scoped to the current task
CONFIG_KEY_RUNNER_SUBMIT = sys.intern("__pregel_runner_submit")
# holds a function that receives tasks from runner, executes them and returns results
CONFIG_KEY_DURABILITY = sys.intern("__pregel_durability")
# holds the durability mode, one of "sync", "async", or "exit"
CONFIG_KEY_RUNTIME = sys.intern("__pregel_runtime")
# holds a `Runtime` instance with context, store, stream writer, etc.
CONFIG_KEY_RESUME_MAP = sys.intern("__pregel_resume_map")
# holds a mapping of task ns -> resume value for resuming tasks

# --- Other constants ---
PUSH = sys.intern("__pregel_push")
# denotes push-style tasks, ie. those created by Send objects
PULL = sys.intern("__pregel_pull")
# denotes pull-style tasks, ie. those triggered by edges
NS_SEP = sys.intern("|")
# for checkpoint_ns, separates each level (ie. graph|subgraph|subsubgraph)
NS_END = sys.intern(":")
# for checkpoint_ns, for each level, separates the namespace from the task_id
CONF = cast(Literal["configurable"], sys.intern("configurable"))
# key for the configurable dict in RunnableConfig
NULL_TASK_ID = sys.intern("00000000-0000-0000-0000-000000000000")
# the task_id to use for writes that are not associated with a task
OVERWRITE = sys.intern("__overwrite__")
# dict key for the overwrite value, used as `{'__overwrite__': value}`

# redefined to avoid circular import with langgraph.constants
_TAG_HIDDEN = sys.intern("langsmith:hidden")

RESERVED = {
    _TAG_HIDDEN,
    # reserved write keys
    INPUT,
    INTERRUPT,
    RESUME,
    ERROR,
    NO_WRITES,
    # reserved config.configurable keys
    CONFIG_KEY_SEND,
    CONFIG_KEY_READ,
    CONFIG_KEY_CHECKPOINTER,
    CONFIG_KEY_STREAM,
    CONFIG_KEY_CHECKPOINT_MAP,
    CONFIG_KEY_RESUMING,
    CONFIG_KEY_TASK_ID,
    CONFIG_KEY_CHECKPOINT_MAP,
    CONFIG_KEY_CHECKPOINT_ID,
    CONFIG_KEY_CHECKPOINT_NS,
    CONFIG_KEY_RESUME_MAP,
    # other constants
    PUSH,
    PULL,
    NS_SEP,
    NS_END,
    CONF,
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/_internal/_fields.py
```py
from __future__ import annotations

import dataclasses
import types
import weakref
from collections.abc import Generator, Sequence
from typing import Annotated, Any, Optional, Union, get_origin, get_type_hints

from pydantic import BaseModel
from typing_extensions import NotRequired, ReadOnly, Required

from langgraph._internal._typing import MISSING


def _is_optional_type(type_: Any) -> bool:
    """Check if a type is Optional."""

    # Handle new union syntax (PEP 604): str | None
    if isinstance(type_, types.UnionType):
        return any(
            arg is type(None) or _is_optional_type(arg) for arg in type_.__args__
        )

    if hasattr(type_, "__origin__") and hasattr(type_, "__args__"):
        origin = get_origin(type_)
        if origin is Optional:
            return True
        if origin is Union:
            return any(
                arg is type(None) or _is_optional_type(arg) for arg in type_.__args__
            )
        if origin is Annotated:
            return _is_optional_type(type_.__args__[0])
        return origin is None
    if hasattr(type_, "__bound__") and type_.__bound__ is not None:
        return _is_optional_type(type_.__bound__)
    return type_ is None


def _is_required_type(type_: Any) -> bool | None:
    """Check if an annotation is marked as Required/NotRequired.

    Returns:
        - True if required
        - False if not required
        - None if not annotated with either
    """
    origin = get_origin(type_)
    if origin is Required:
        return True
    if origin is NotRequired:
        return False
    if origin is Annotated or getattr(origin, "__args__", None):
        # See https://typing.readthedocs.io/en/latest/spec/typeddict.html#interaction-with-annotated
        return _is_required_type(type_.__args__[0])
    return None


def _is_readonly_type(type_: Any) -> bool:
    """Check if an annotation is marked as ReadOnly.

    Returns:
        - True if is read only
        - False if not read only
    """

    # See: https://typing.readthedocs.io/en/latest/spec/typeddict.html#typing-readonly-type-qualifier
    origin = get_origin(type_)
    if origin is Annotated:
        return _is_readonly_type(type_.__args__[0])
    if origin is ReadOnly:
        return True
    return False


_DEFAULT_KEYS: frozenset[str] = frozenset()


def get_field_default(name: str, type_: Any, schema: type[Any]) -> Any:
    """Determine the default value for a field in a state schema.

    This is based on:
        If TypedDict:
            - Required/NotRequired
            - total=False -> everything optional
        - Type annotation (Optional/Union[None])
    """
    optional_keys = getattr(schema, "__optional_keys__", _DEFAULT_KEYS)
    irq = _is_required_type(type_)
    if name in optional_keys:
        # Either total=False or explicit NotRequired.
        # No type annotation trumps this.
        if irq:
            # Unless it's earlier versions of python & explicit Required
            return ...
        return None
    if irq is not None:
        if irq:
            # Handle Required[<type>]
            # (we already handled NotRequired and total=False)
            return ...
        # Handle NotRequired[<type>] for earlier versions of python
        return None
    if dataclasses.is_dataclass(schema):
        field_info = next(
            (f for f in dataclasses.fields(schema) if f.name == name), None
        )
        if field_info:
            if (
                field_info.default is not dataclasses.MISSING
                and field_info.default is not ...
            ):
                return field_info.default
            elif field_info.default_factory is not dataclasses.MISSING:
                return field_info.default_factory()
    # Note, we ignore ReadOnly attributes,
    # as they don't make much sense. (we don't care if you mutate the state in your node)
    # and mutating state in your node has no effect on our graph state.
    # Base case is the annotation
    if _is_optional_type(type_):
        return None
    return ...


def get_enhanced_type_hints(
    type: type[Any],
) -> Generator[tuple[str, Any, Any, str | None], None, None]:
    """Attempt to extract default values and descriptions from provided type, used for config schema."""
    for name, typ in get_type_hints(type).items():
        default = None
        description = None

        # Pydantic models
        try:
            if hasattr(type, "model_fields") and name in type.model_fields:
                field = type.model_fields[name]

                if hasattr(field, "description") and field.description is not None:
                    description = field.description

                if hasattr(field, "default") and field.default is not None:
                    default = field.default
                    if (
                        hasattr(default, "__class__")
                        and getattr(default.__class__, "__name__", "")
                        == "PydanticUndefinedType"
                    ):
                        default = None

        except (AttributeError, KeyError, TypeError):
            pass

        # TypedDict, dataclass
        try:
            if hasattr(type, "__dict__"):
                type_dict = getattr(type, "__dict__")

                if name in type_dict:
                    default = type_dict[name]
        except (AttributeError, KeyError, TypeError):
            pass

        yield name, typ, default, description


def get_update_as_tuples(input: Any, keys: Sequence[str]) -> list[tuple[str, Any]]:
    """Get Pydantic state update as a list of (key, value) tuples."""
    if isinstance(input, BaseModel):
        keep = input.model_fields_set
        defaults = {k: v.default for k, v in type(input).model_fields.items()}
    else:
        keep = None
        defaults = {}

    # NOTE: This behavior for Pydantic is somewhat inelegant,
    # but we keep around for backwards compatibility
    # if input is a Pydantic model, only update values
    # that are different from the default values or in the keep set
    return [
        (k, value)
        for k in keys
        if (value := getattr(input, k, MISSING)) is not MISSING
        and (
            value is not None
            or defaults.get(k, MISSING) is not None
            or (keep is not None and k in keep)
        )
    ]


ANNOTATED_KEYS_CACHE: weakref.WeakKeyDictionary[type[Any], tuple[str, ...]] = (
    weakref.WeakKeyDictionary()
)


def get_cached_annotated_keys(obj: type[Any]) -> tuple[str, ...]:
    """Return cached annotated keys for a Python class."""
    if obj in ANNOTATED_KEYS_CACHE:
        return ANNOTATED_KEYS_CACHE[obj]
    if isinstance(obj, type):
        keys: list[str] = []
        for base in reversed(obj.__mro__):
            ann = base.__dict__.get("__annotations__")
            # In Python 3.14+, Pydantic models use descriptors for __annotations__
            # so we need to fall back to getattr if __dict__.get returns None
            if ann is None:
                ann = getattr(base, "__annotations__", None)
            if ann is None or isinstance(ann, types.GetSetDescriptorType):
                continue
            keys.extend(ann.keys())
        return ANNOTATED_KEYS_CACHE.setdefault(obj, tuple(keys))
    else:
        raise TypeError(f"Expected a type, got {type(obj)}. ")

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/_internal/_future.py
```py
from __future__ import annotations

import asyncio
import concurrent.futures
import contextvars
import inspect
import sys
import types
from collections.abc import Awaitable, Coroutine, Generator
from typing import TypeVar, cast

T = TypeVar("T")
AnyFuture = asyncio.Future | concurrent.futures.Future

CONTEXT_NOT_SUPPORTED = sys.version_info < (3, 11)
EAGER_NOT_SUPPORTED = sys.version_info < (3, 12)


def _get_loop(fut: asyncio.Future) -> asyncio.AbstractEventLoop:
    # Tries to call Future.get_loop() if it's available.
    # Otherwise fallbacks to using the old '_loop' property.
    try:
        get_loop = fut.get_loop
    except AttributeError:
        pass
    else:
        return get_loop()
    return fut._loop


def _convert_future_exc(exc: BaseException) -> BaseException:
    exc_class = type(exc)
    if exc_class is concurrent.futures.CancelledError:
        return asyncio.CancelledError(*exc.args)
    elif exc_class is concurrent.futures.TimeoutError:
        return asyncio.TimeoutError(*exc.args)
    elif exc_class is concurrent.futures.InvalidStateError:
        return asyncio.InvalidStateError(*exc.args)
    else:
        return exc


def _set_concurrent_future_state(
    concurrent: concurrent.futures.Future,
    source: AnyFuture,
) -> None:
    """Copy state from a future to a concurrent.futures.Future."""
    assert source.done()
    if source.cancelled():
        concurrent.cancel()
    if not concurrent.set_running_or_notify_cancel():
        return
    exception = source.exception()
    if exception is not None:
        concurrent.set_exception(_convert_future_exc(exception))
    else:
        result = source.result()
        concurrent.set_result(result)


def _copy_future_state(source: AnyFuture, dest: asyncio.Future) -> None:
    """Internal helper to copy state from another Future.

    The other Future may be a concurrent.futures.Future.
    """
    if dest.done():
        return
    assert source.done()
    if dest.cancelled():
        return
    if source.cancelled():
        dest.cancel()
    else:
        exception = source.exception()
        if exception is not None:
            dest.set_exception(_convert_future_exc(exception))
        else:
            result = source.result()
            dest.set_result(result)


def _chain_future(source: AnyFuture, destination: AnyFuture) -> None:
    """Chain two futures so that when one completes, so does the other.

    The result (or exception) of source will be copied to destination.
    If destination is cancelled, source gets cancelled too.
    Compatible with both asyncio.Future and concurrent.futures.Future.
    """
    if not asyncio.isfuture(source) and not isinstance(
        source, concurrent.futures.Future
    ):
        raise TypeError("A future is required for source argument")
    if not asyncio.isfuture(destination) and not isinstance(
        destination, concurrent.futures.Future
    ):
        raise TypeError("A future is required for destination argument")
    source_loop = _get_loop(source) if asyncio.isfuture(source) else None
    dest_loop = _get_loop(destination) if asyncio.isfuture(destination) else None

    def _set_state(future: AnyFuture, other: AnyFuture) -> None:
        if asyncio.isfuture(future):
            _copy_future_state(other, future)
        else:
            _set_concurrent_future_state(future, other)

    def _call_check_cancel(destination: AnyFuture) -> None:
        if destination.cancelled():
            if source_loop is None or source_loop is dest_loop:
                source.cancel()
            else:
                source_loop.call_soon_threadsafe(source.cancel)

    def _call_set_state(source: AnyFuture) -> None:
        if destination.cancelled() and dest_loop is not None and dest_loop.is_closed():
            return
        if dest_loop is None or dest_loop is source_loop:
            _set_state(destination, source)
        else:
            if dest_loop.is_closed():
                return
            dest_loop.call_soon_threadsafe(_set_state, destination, source)

    destination.add_done_callback(_call_check_cancel)
    source.add_done_callback(_call_set_state)


def chain_future(source: AnyFuture, destination: AnyFuture) -> AnyFuture:
    # adapted from asyncio.run_coroutine_threadsafe
    try:
        _chain_future(source, destination)
        return destination
    except (SystemExit, KeyboardInterrupt):
        raise
    except BaseException as exc:
        if isinstance(destination, concurrent.futures.Future):
            if destination.set_running_or_notify_cancel():
                destination.set_exception(exc)
        else:
            destination.set_exception(exc)
        raise


def _ensure_future(
    coro_or_future: Coroutine[None, None, T] | Awaitable[T],
    *,
    loop: asyncio.AbstractEventLoop,
    name: str | None = None,
    context: contextvars.Context | None = None,
    lazy: bool = True,
) -> asyncio.Task[T]:
    called_wrap_awaitable = False
    if not asyncio.iscoroutine(coro_or_future):
        if inspect.isawaitable(coro_or_future):
            coro_or_future = cast(
                Coroutine[None, None, T], _wrap_awaitable(coro_or_future)
            )
            called_wrap_awaitable = True
        else:
            raise TypeError(
                "An asyncio.Future, a coroutine or an awaitable is required."
                f" Got {type(coro_or_future).__name__} instead."
            )

    try:
        if CONTEXT_NOT_SUPPORTED:
            return loop.create_task(coro_or_future, name=name)
        elif EAGER_NOT_SUPPORTED or lazy:
            return loop.create_task(coro_or_future, name=name, context=context)
        else:
            return asyncio.eager_task_factory(
                loop, coro_or_future, name=name, context=context
            )
    except RuntimeError:
        if not called_wrap_awaitable:
            coro_or_future.close()
        raise


@types.coroutine
def _wrap_awaitable(awaitable: Awaitable[T]) -> Generator[None, None, T]:
    """Helper for asyncio.ensure_future().

    Wraps awaitable (an object with __await__) into a coroutine
    that will later be wrapped in a Task by ensure_future().
    """
    return (yield from awaitable.__await__())


def run_coroutine_threadsafe(
    coro: Coroutine[None, None, T],
    loop: asyncio.AbstractEventLoop,
    *,
    lazy: bool,
    name: str | None = None,
    context: contextvars.Context | None = None,
) -> asyncio.Future[T]:
    """Submit a coroutine object to a given event loop.

    Return an asyncio.Future to access the result.
    """

    if asyncio._get_running_loop() is loop:
        return _ensure_future(coro, loop=loop, name=name, context=context, lazy=lazy)
    else:
        future: asyncio.Future[T] = asyncio.Future(loop=loop)

        def callback() -> None:
            try:
                chain_future(
                    _ensure_future(coro, loop=loop, name=name, context=context),
                    future,
                )
            except (SystemExit, KeyboardInterrupt):
                raise
            except BaseException as exc:
                future.set_exception(exc)
                raise

        loop.call_soon_threadsafe(callback, context=context)
        return future

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/_internal/_pydantic.py
```py
from __future__ import annotations

import sys
import typing
import warnings
from contextlib import nullcontext
from dataclasses import is_dataclass
from functools import lru_cache
from typing import (
    Any,
    cast,
    overload,
)

from pydantic import (
    BaseModel,
    ConfigDict,
    Field,
    RootModel,
)
from pydantic import (
    create_model as _create_model_base,
)
from pydantic.fields import FieldInfo
from pydantic.json_schema import (
    DEFAULT_REF_TEMPLATE,
    GenerateJsonSchema,
    JsonSchemaMode,
)
from typing_extensions import TypedDict


@overload
def get_fields(model: type[BaseModel]) -> dict[str, FieldInfo]: ...


@overload
def get_fields(model: BaseModel) -> dict[str, FieldInfo]: ...


def get_fields(
    model: type[BaseModel] | BaseModel,
) -> dict[str, FieldInfo]:
    """Get the field names of a Pydantic model."""
    if hasattr(model, "model_fields"):
        return model.model_fields

    if hasattr(model, "__fields__"):
        return model.__fields__
    msg = f"Expected a Pydantic model. Got {type(model)}"
    raise TypeError(msg)


_SchemaConfig = ConfigDict(
    arbitrary_types_allowed=True, frozen=True, protected_namespaces=()
)

NO_DEFAULT = object()


def _create_root_model(
    name: str,
    type_: Any,
    module_name: str | None = None,
    default_: object = NO_DEFAULT,
) -> type[BaseModel]:
    """Create a base class."""

    def schema(
        cls: type[BaseModel],
        by_alias: bool = True,  # noqa: FBT001,FBT002
        ref_template: str = DEFAULT_REF_TEMPLATE,
    ) -> dict[str, Any]:
        # Complains about schema not being defined in superclass
        schema_ = super(cls, cls).schema(  # type: ignore[misc]
            by_alias=by_alias, ref_template=ref_template
        )
        schema_["title"] = name
        return schema_

    def model_json_schema(
        cls: type[BaseModel],
        by_alias: bool = True,  # noqa: FBT001,FBT002
        ref_template: str = DEFAULT_REF_TEMPLATE,
        schema_generator: type[GenerateJsonSchema] = GenerateJsonSchema,
        mode: JsonSchemaMode = "validation",
    ) -> dict[str, Any]:
        # Complains about model_json_schema not being defined in superclass
        schema_ = super(cls, cls).model_json_schema(  # type: ignore[misc]
            by_alias=by_alias,
            ref_template=ref_template,
            schema_generator=schema_generator,
            mode=mode,
        )
        schema_["title"] = name
        return schema_

    base_class_attributes = {
        "__annotations__": {"root": type_},
        "model_config": ConfigDict(arbitrary_types_allowed=True),
        "schema": classmethod(schema),
        "model_json_schema": classmethod(model_json_schema),
        "__module__": module_name or "langchain_core.runnables.utils",
    }

    if default_ is not NO_DEFAULT:
        base_class_attributes["root"] = default_
    with warnings.catch_warnings():
        custom_root_type = type(name, (RootModel,), base_class_attributes)
    return cast("type[BaseModel]", custom_root_type)


@lru_cache(maxsize=256)
def _create_root_model_cached(
    model_name: str,
    type_: Any,
    *,
    module_name: str | None = None,
    default_: object = NO_DEFAULT,
) -> type[BaseModel]:
    return _create_root_model(
        model_name, type_, default_=default_, module_name=module_name
    )


@lru_cache(maxsize=256)
def _create_model_cached(
    model_name: str,
    /,
    **field_definitions: Any,
) -> type[BaseModel]:
    return _create_model_base(
        model_name,
        __config__=_SchemaConfig,
        **_remap_field_definitions(field_definitions),
    )


# Reserved names should capture all the `public` names / methods that are
# used by BaseModel internally. This will keep the reserved names up-to-date.
# For reference, the reserved names are:
# "construct", "copy", "dict", "from_orm", "json", "parse_file", "parse_obj",
# "parse_raw", "schema", "schema_json", "update_forward_refs", "validate",
# "model_computed_fields", "model_config", "model_construct", "model_copy",
# "model_dump", "model_dump_json", "model_extra", "model_fields",
# "model_fields_set", "model_json_schema", "model_parametrized_name",
# "model_post_init", "model_rebuild", "model_validate", "model_validate_json",
# "model_validate_strings"
_RESERVED_NAMES = {key for key in dir(BaseModel) if not key.startswith("_")}


def _remap_field_definitions(field_definitions: dict[str, Any]) -> dict[str, Any]:
    """This remaps fields to avoid colliding with internal pydantic fields."""

    remapped = {}
    for key, value in field_definitions.items():
        if key.startswith("_") or key in _RESERVED_NAMES:
            # Let's add a prefix to avoid colliding with internal pydantic fields
            if isinstance(value, FieldInfo):
                msg = (
                    f"Remapping for fields starting with '_' or fields with a name "
                    f"matching a reserved name {_RESERVED_NAMES} is not supported if "
                    f" the field is a pydantic Field instance. Got {key}."
                )
                raise NotImplementedError(msg)
            type_, default_ = value
            remapped[f"private_{key}"] = (
                type_,
                Field(
                    default=default_,
                    alias=key,
                    serialization_alias=key,
                    title=key.lstrip("_").replace("_", " ").title(),
                ),
            )
        else:
            remapped[key] = value
    return remapped


def create_model(
    model_name: str,
    *,
    field_definitions: dict[str, Any] | None = None,
    root: Any | None = None,
) -> type[BaseModel]:
    """Create a pydantic model with the given field definitions.

    Attention:
        Please do not use outside of langchain packages. This API
        is subject to change at any time.

    Args:
        model_name: The name of the model.
        module_name: The name of the module where the model is defined.
            This is used by Pydantic to resolve any forward references.
        field_definitions: The field definitions for the model.
        root: Type for a root model (RootModel)

    Returns:
        Type[BaseModel]: The created model.
    """
    field_definitions = field_definitions or {}

    if root:
        if field_definitions:
            msg = (
                "When specifying __root__ no other "
                f"fields should be provided. Got {field_definitions}"
            )
            raise NotImplementedError(msg)

        if isinstance(root, tuple):
            kwargs = {"type_": root[0], "default_": root[1]}
        else:
            kwargs = {"type_": root}

        try:
            named_root_model = _create_root_model_cached(model_name, **kwargs)
        except TypeError:
            # something in the arguments into _create_root_model_cached is not hashable
            named_root_model = _create_root_model(
                model_name,
                **kwargs,
            )
        return named_root_model

    # No root, just field definitions
    names = set(field_definitions.keys())

    capture_warnings = False

    for name in names:
        # Also if any non-reserved name is used (e.g., model_id or model_name)
        if name.startswith("model"):
            capture_warnings = True

    with warnings.catch_warnings() if capture_warnings else nullcontext():
        if capture_warnings:
            warnings.filterwarnings(action="ignore")
        try:
            return _create_model_cached(model_name, **field_definitions)
        except TypeError:
            # something in field definitions is not hashable
            return _create_model_base(
                model_name,
                __config__=_SchemaConfig,
                **_remap_field_definitions(field_definitions),
            )


def is_supported_by_pydantic(type_: Any) -> bool:
    """Check if a given "complex" type is supported by pydantic.

    This will return False for primitive types like int, str, etc.

    The check is meant for container types like dataclasses, TypedDicts, etc.
    """
    if is_dataclass(type_):
        return True

    if isinstance(type_, type) and issubclass(type_, BaseModel):
        return True

    if hasattr(type_, "__orig_bases__"):
        for base in type_.__orig_bases__:
            if base is TypedDict:
                return True
            elif base is typing.TypedDict:  # noqa: TID251
                # ignoring TID251 since it's OK to use typing.TypedDict in this case.
                # Pydantic supports typing.TypedDict from Python 3.12
                # For older versions, only typing_extensions.TypedDict is supported.
                if sys.version_info >= (3, 12):
                    return True
    return False

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/_internal/_queue.py
```py
# type: ignore
from __future__ import annotations

import asyncio
import queue
import threading
import types
from collections import deque
from time import monotonic


class AsyncQueue(asyncio.Queue):
    """Async unbounded FIFO queue with a wait() method.

    Subclassed from asyncio.Queue, adding a wait() method."""

    async def wait(self) -> None:
        """If queue is empty, wait until an item is available.

        Copied from Queue.get(), removing the call to .get_nowait(),
        ie. this doesn't consume the item, just waits for it.
        """
        while self.empty():
            getter = self._get_loop().create_future()
            self._getters.append(getter)
            try:
                await getter
            except:
                getter.cancel()  # Just in case getter is not done yet.
                try:
                    # Clean self._getters from canceled getters.
                    self._getters.remove(getter)
                except ValueError:
                    # The getter could be removed from self._getters by a
                    # previous put_nowait call.
                    pass
                if not self.empty() and not getter.cancelled():
                    # We were woken up by put_nowait(), but can't take
                    # the call.  Wake up the next in line.
                    self._wakeup_next(self._getters)
                raise


class Semaphore(threading.Semaphore):
    """Semaphore subclass with a wait() method."""

    def wait(self, blocking: bool = True, timeout: float | None = None):
        """Block until the semaphore can be acquired, but don't acquire it."""
        if not blocking and timeout is not None:
            raise ValueError("can't specify timeout for non-blocking acquire")
        rc = False
        endtime = None
        with self._cond:
            while self._value == 0:
                if not blocking:
                    break
                if timeout is not None:
                    if endtime is None:
                        endtime = monotonic() + timeout
                    else:
                        timeout = endtime - monotonic()
                        if timeout <= 0:
                            break
                self._cond.wait(timeout)
            else:
                rc = True
        return rc


class SyncQueue:
    """Unbounded FIFO queue with a wait() method.
    Adapted from pure Python implementation of queue.SimpleQueue.
    """

    def __init__(self):
        self._queue = deque()
        self._count = Semaphore(0)

    def put(self, item, block=True, timeout=None):
        """Put the item on the queue.

        The optional 'block' and 'timeout' arguments are ignored, as this method
        never blocks.  They are provided for compatibility with the Queue class.
        """
        self._queue.append(item)
        self._count.release()

    def get(self, block=False, timeout=None):
        """Remove and return an item from the queue.

        If optional args 'block' is true and 'timeout' is None (the default),
        block if necessary until an item is available. If 'timeout' is
        a non-negative number, it blocks at most 'timeout' seconds and raises
        the Empty exception if no item was available within that time.
        Otherwise ('block' is false), return an item if one is immediately
        available, else raise the Empty exception ('timeout' is ignored
        in that case).
        """
        if timeout is not None and timeout < 0:
            raise ValueError("'timeout' must be a non-negative number")
        if not self._count.acquire(block, timeout):
            raise queue.Empty
        try:
            return self._queue.popleft()
        except IndexError:
            raise queue.Empty

    def wait(self, block=True, timeout=None):
        """If queue is empty, wait until an item maybe is available,
        but don't consume it.
        """
        if timeout is not None and timeout < 0:
            raise ValueError("'timeout' must be a non-negative number")
        self._count.wait(block, timeout)

    def empty(self):
        """Return True if the queue is empty, False otherwise (not reliable!)."""
        return len(self._queue) == 0

    def qsize(self):
        """Return the approximate size of the queue (not reliable!)."""
        return len(self._queue)

    __class_getitem__ = classmethod(types.GenericAlias)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/_internal/_retry.py
```py
def default_retry_on(exc: Exception) -> bool:
    import httpx
    import requests

    if isinstance(exc, ConnectionError):
        return True
    if isinstance(exc, httpx.HTTPStatusError):
        return 500 <= exc.response.status_code < 600
    if isinstance(exc, requests.HTTPError):
        return 500 <= exc.response.status_code < 600 if exc.response else True
    if isinstance(
        exc,
        (
            ValueError,
            TypeError,
            ArithmeticError,
            ImportError,
            LookupError,
            NameError,
            SyntaxError,
            RuntimeError,
            ReferenceError,
            StopIteration,
            StopAsyncIteration,
            OSError,
        ),
    ):
        return False
    return True

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/_internal/_runnable.py
```py
from __future__ import annotations

import asyncio
import enum
import inspect
import sys
import warnings
from collections.abc import (
    AsyncIterator,
    Awaitable,
    Callable,
    Coroutine,
    Generator,
    Iterator,
    Sequence,
)
from contextlib import AsyncExitStack, contextmanager
from contextvars import Context, Token, copy_context
from functools import partial, wraps
from typing import (
    Any,
    Optional,
    Protocol,
    TypeGuard,
    cast,
)

from langchain_core.runnables.base import (
    Runnable,
    RunnableConfig,
    RunnableLambda,
    RunnableParallel,
    RunnableSequence,
)
from langchain_core.runnables.base import (
    RunnableLike as LCRunnableLike,
)
from langchain_core.runnables.config import (
    run_in_executor,
    var_child_runnable_config,
)
from langchain_core.runnables.utils import Input, Output
from langchain_core.tracers.langchain import LangChainTracer
from langgraph.store.base import BaseStore

from langgraph._internal._config import (
    ensure_config,
    get_async_callback_manager_for_config,
    get_callback_manager_for_config,
    patch_config,
)
from langgraph._internal._constants import (
    CONF,
    CONFIG_KEY_RUNTIME,
)
from langgraph._internal._typing import MISSING
from langgraph.types import StreamWriter

try:
    from langchain_core.tracers._streaming import _StreamingCallbackHandler
except ImportError:
    _StreamingCallbackHandler = None  # type: ignore


def _set_config_context(
    config: RunnableConfig, run: Any = None
) -> Token[RunnableConfig | None]:
    """Set the child Runnable config + tracing context.

    Args:
        config: The config to set.
    """
    config_token = var_child_runnable_config.set(config)
    if run is not None:
        from langsmith.run_helpers import _set_tracing_context

        _set_tracing_context({"parent": run})
    return config_token


def _unset_config_context(token: Token[RunnableConfig | None], run: Any = None) -> None:
    """Set the child Runnable config + tracing context.

    Args:
        token: The config token to reset.
    """
    var_child_runnable_config.reset(token)
    if run is not None:
        from langsmith.run_helpers import _set_tracing_context

        _set_tracing_context(
            {
                "parent": None,
                "project_name": None,
                "tags": None,
                "metadata": None,
                "enabled": None,
                "client": None,
            }
        )


@contextmanager
def set_config_context(
    config: RunnableConfig, run: Any = None
) -> Generator[Context, None, None]:
    """Set the child Runnable config + tracing context.

    Args:
        config: The config to set.
    """
    ctx = copy_context()
    config_token = ctx.run(_set_config_context, config, run)
    try:
        yield ctx
    finally:
        ctx.run(_unset_config_context, config_token, run)


# Before Python 3.11 native StrEnum is not available
class StrEnum(str, enum.Enum):
    """A string enum."""


# Special type to denote any type is accepted
ANY_TYPE = object()

ASYNCIO_ACCEPTS_CONTEXT = sys.version_info >= (3, 11)

# List of keyword arguments that can be injected into nodes / tasks / tools at runtime.
# A named argument may appear multiple times if it appears with distinct types.
KWARGS_CONFIG_KEYS: tuple[tuple[str, tuple[Any, ...], str, Any], ...] = (
    (
        "config",
        (
            RunnableConfig,
            "RunnableConfig",
            Optional[RunnableConfig],  # noqa: UP045
            "Optional[RunnableConfig]",
            inspect.Parameter.empty,
        ),
        # for now, use config directly, eventually, will pop off of Runtime
        "N/A",
        inspect.Parameter.empty,
    ),
    (
        "writer",
        (StreamWriter, "StreamWriter", inspect.Parameter.empty),
        "stream_writer",
        lambda _: None,
    ),
    (
        "store",
        (
            BaseStore,
            "BaseStore",
            inspect.Parameter.empty,
        ),
        "store",
        inspect.Parameter.empty,
    ),
    (
        "store",
        (
            Optional[BaseStore],  # noqa: UP045
            "Optional[BaseStore]",
        ),
        "store",
        None,
    ),
    (
        "previous",
        (ANY_TYPE,),
        "previous",
        inspect.Parameter.empty,
    ),
    (
        "runtime",
        (ANY_TYPE,),
        # we never hit this block, we just inject runtime directly
        "N/A",
        inspect.Parameter.empty,
    ),
)
"""List of kwargs that can be passed to functions, and their corresponding
config keys, default values and type annotations.

Used to configure keyword arguments that can be injected at runtime
from the `Runtime` object as kwargs to `invoke`, `ainvoke`, `stream` and `astream`.

For a keyword to be injected from the config object, the function signature
must contain a kwarg with the same name and a matching type annotation.

Each tuple contains:
- the name of the kwarg in the function signature
- the type annotation(s) for the kwarg
- the `Runtime` attribute for fetching the value (N/A if not applicable)

This is fully internal and should be further refactored to use `get_type_hints`
to resolve forward references and optional types formatted like BaseStore | None.
"""

VALID_KINDS = (inspect.Parameter.POSITIONAL_OR_KEYWORD, inspect.Parameter.KEYWORD_ONLY)


class _RunnableWithWriter(Protocol[Input, Output]):
    def __call__(self, state: Input, *, writer: StreamWriter) -> Output: ...


class _RunnableWithStore(Protocol[Input, Output]):
    def __call__(self, state: Input, *, store: BaseStore) -> Output: ...


class _RunnableWithWriterStore(Protocol[Input, Output]):
    def __call__(
        self, state: Input, *, writer: StreamWriter, store: BaseStore
    ) -> Output: ...


class _RunnableWithConfigWriter(Protocol[Input, Output]):
    def __call__(
        self, state: Input, *, config: RunnableConfig, writer: StreamWriter
    ) -> Output: ...


class _RunnableWithConfigStore(Protocol[Input, Output]):
    def __call__(
        self, state: Input, *, config: RunnableConfig, store: BaseStore
    ) -> Output: ...


class _RunnableWithConfigWriterStore(Protocol[Input, Output]):
    def __call__(
        self,
        state: Input,
        *,
        config: RunnableConfig,
        writer: StreamWriter,
        store: BaseStore,
    ) -> Output: ...


RunnableLike = (
    LCRunnableLike
    | _RunnableWithWriter[Input, Output]
    | _RunnableWithStore[Input, Output]
    | _RunnableWithWriterStore[Input, Output]
    | _RunnableWithConfigWriter[Input, Output]
    | _RunnableWithConfigStore[Input, Output]
    | _RunnableWithConfigWriterStore[Input, Output]
)


class RunnableCallable(Runnable):
    """A much simpler version of RunnableLambda that requires sync and async functions."""

    def __init__(
        self,
        func: Callable[..., Any | Runnable] | None,
        afunc: Callable[..., Awaitable[Any | Runnable]] | None = None,
        *,
        name: str | None = None,
        tags: Sequence[str] | None = None,
        trace: bool = True,
        recurse: bool = True,
        explode_args: bool = False,
        **kwargs: Any,
    ) -> None:
        self.name = name
        if self.name is None:
            if func:
                try:
                    if func.__name__ != "<lambda>":
                        self.name = func.__name__
                except AttributeError:
                    pass
            elif afunc:
                try:
                    self.name = afunc.__name__
                except AttributeError:
                    pass
        self.func = func
        self.afunc = afunc
        self.tags = tags
        self.kwargs = kwargs
        self.trace = trace
        self.recurse = recurse
        self.explode_args = explode_args
        # check signature
        if func is None and afunc is None:
            raise ValueError("At least one of func or afunc must be provided.")

        self.func_accepts: dict[str, tuple[str, Any]] = {}
        params = inspect.signature(cast(Callable, func or afunc)).parameters

        for kw, typ, runtime_key, default in KWARGS_CONFIG_KEYS:
            p = params.get(kw)

            if p is None or p.kind not in VALID_KINDS:
                # If parameter is not found or is not a valid kind, skip
                continue

            if typ != (ANY_TYPE,) and p.annotation not in typ:
                # A specific type is required, but the function annotation does
                # not match the expected type.

                # If this is a config parameter with incorrect typing, emit a warning
                # because we used to support any type but are moving towards more correct typing
                if kw == "config" and p.annotation != inspect.Parameter.empty:
                    warnings.warn(
                        f"The 'config' parameter should be typed as 'RunnableConfig' or "
                        f"'RunnableConfig | None', not '{p.annotation}'. ",
                        UserWarning,
                        stacklevel=4,
                    )
                continue

            # If the kwarg is accepted by the function, store the key / runtime attribute to inject
            self.func_accepts[kw] = (runtime_key, default)

    def __repr__(self) -> str:
        repr_args = {
            k: v
            for k, v in self.__dict__.items()
            if k not in {"name", "func", "afunc", "config", "kwargs", "trace"}
        }
        return f"{self.get_name()}({', '.join(f'{k}={v!r}' for k, v in repr_args.items())})"

    def invoke(
        self, input: Any, config: RunnableConfig | None = None, **kwargs: Any
    ) -> Any:
        if self.func is None:
            raise TypeError(
                f'No synchronous function provided to "{self.name}".'
                "\nEither initialize with a synchronous function or invoke"
                " via the async API (ainvoke, astream, etc.)"
            )
        if config is None:
            config = ensure_config()
        if self.explode_args:
            args, _kwargs = input
            kwargs = {**self.kwargs, **_kwargs, **kwargs}
        else:
            args = (input,)
            kwargs = {**self.kwargs, **kwargs}

        runtime = config.get(CONF, {}).get(CONFIG_KEY_RUNTIME)

        for kw, (runtime_key, default) in self.func_accepts.items():
            # If the kwarg is already set, use the set value
            if kw in kwargs:
                continue

            kw_value: Any = MISSING
            if kw == "config":
                kw_value = config
            elif runtime:
                if kw == "runtime":
                    kw_value = runtime
                else:
                    try:
                        kw_value = getattr(runtime, runtime_key)
                    except AttributeError:
                        pass

            if kw_value is MISSING:
                if default is inspect.Parameter.empty:
                    raise ValueError(
                        f"Missing required config key '{runtime_key}' for '{self.name}'."
                    )
                kw_value = default
            kwargs[kw] = kw_value

        if self.trace:
            callback_manager = get_callback_manager_for_config(config, self.tags)
            run_manager = callback_manager.on_chain_start(
                None,
                input,
                name=config.get("run_name") or self.get_name(),
                run_id=config.pop("run_id", None),
            )
            try:
                child_config = patch_config(config, callbacks=run_manager.get_child())
                # get the run
                for h in run_manager.handlers:
                    if isinstance(h, LangChainTracer):
                        run = h.run_map.get(str(run_manager.run_id))
                        break
                else:
                    run = None
                # run in context
                with set_config_context(child_config, run) as context:
                    ret = context.run(self.func, *args, **kwargs)
            except BaseException as e:
                run_manager.on_chain_error(e)
                raise
            else:
                run_manager.on_chain_end(ret)
        else:
            ret = self.func(*args, **kwargs)
        if self.recurse and isinstance(ret, Runnable):
            return ret.invoke(input, config)
        return ret

    async def ainvoke(
        self, input: Any, config: RunnableConfig | None = None, **kwargs: Any
    ) -> Any:
        if not self.afunc:
            return self.invoke(input, config)
        if config is None:
            config = ensure_config()
        if self.explode_args:
            args, _kwargs = input
            kwargs = {**self.kwargs, **_kwargs, **kwargs}
        else:
            args = (input,)
            kwargs = {**self.kwargs, **kwargs}

        runtime = config.get(CONF, {}).get(CONFIG_KEY_RUNTIME)

        for kw, (runtime_key, default) in self.func_accepts.items():
            # If the kwarg has already been set, use the set value
            if kw in kwargs:
                continue

            kw_value: Any = MISSING
            if kw == "config":
                kw_value = config
            elif runtime:
                if kw == "runtime":
                    kw_value = runtime
                else:
                    try:
                        kw_value = getattr(runtime, runtime_key)
                    except AttributeError:
                        pass
            if kw_value is MISSING:
                if default is inspect.Parameter.empty:
                    raise ValueError(
                        f"Missing required config key '{runtime_key}' for '{self.name}'."
                    )
                kw_value = default
            kwargs[kw] = kw_value

        if self.trace:
            callback_manager = get_async_callback_manager_for_config(config, self.tags)
            run_manager = await callback_manager.on_chain_start(
                None,
                input,
                name=config.get("run_name") or self.name,
                run_id=config.pop("run_id", None),
            )
            try:
                child_config = patch_config(config, callbacks=run_manager.get_child())
                coro = cast(Coroutine[None, None, Any], self.afunc(*args, **kwargs))
                if ASYNCIO_ACCEPTS_CONTEXT:
                    for h in run_manager.handlers:
                        if isinstance(h, LangChainTracer):
                            run = h.run_map.get(str(run_manager.run_id))
                            break
                    else:
                        run = None
                    with set_config_context(child_config, run) as context:
                        ret = await asyncio.create_task(coro, context=context)
                else:
                    ret = await coro
            except BaseException as e:
                await run_manager.on_chain_error(e)
                raise
            else:
                await run_manager.on_chain_end(ret)
        else:
            ret = await self.afunc(*args, **kwargs)
        if self.recurse and isinstance(ret, Runnable):
            return await ret.ainvoke(input, config)
        return ret


def is_async_callable(
    func: Any,
) -> TypeGuard[Callable[..., Awaitable]]:
    """Check if a function is async."""
    return (
        inspect.iscoroutinefunction(func)
        or hasattr(func, "__call__")
        and inspect.iscoroutinefunction(func.__call__)
    )


def is_async_generator(
    func: Any,
) -> TypeGuard[Callable[..., AsyncIterator]]:
    """Check if a function is an async generator."""
    return (
        inspect.isasyncgenfunction(func)
        or hasattr(func, "__call__")
        and inspect.isasyncgenfunction(func.__call__)
    )


def coerce_to_runnable(
    thing: RunnableLike, *, name: str | None, trace: bool
) -> Runnable:
    """Coerce a runnable-like object into a Runnable.

    Args:
        thing: A runnable-like object.

    Returns:
        A Runnable.
    """
    if isinstance(thing, Runnable):
        return thing
    elif is_async_generator(thing) or inspect.isgeneratorfunction(thing):
        return RunnableLambda(thing, name=name)
    elif callable(thing):
        if is_async_callable(thing):
            return RunnableCallable(None, thing, name=name, trace=trace)
        else:
            return RunnableCallable(
                thing,
                wraps(thing)(partial(run_in_executor, None, thing)),  # type: ignore[arg-type]
                name=name,
                trace=trace,
            )
    elif isinstance(thing, dict):
        return RunnableParallel(thing)
    else:
        raise TypeError(
            f"Expected a Runnable, callable or dict."
            f"Instead got an unsupported type: {type(thing)}"
        )


class RunnableSeq(Runnable):
    """Sequence of `Runnable`, where the output of each is the input of the next.

    `RunnableSeq` is a simpler version of `RunnableSequence` that is internal to
    LangGraph.
    """

    def __init__(
        self,
        *steps: RunnableLike,
        name: str | None = None,
        trace_inputs: Callable[[Any], Any] | None = None,
    ) -> None:
        """Create a new RunnableSeq.

        Args:
            steps: The steps to include in the sequence.
            name: The name of the `Runnable`.

        Raises:
            ValueError: If the sequence has less than 2 steps.
        """
        steps_flat: list[Runnable] = []
        for step in steps:
            if isinstance(step, RunnableSequence):
                steps_flat.extend(step.steps)
            elif isinstance(step, RunnableSeq):
                steps_flat.extend(step.steps)
            else:
                steps_flat.append(coerce_to_runnable(step, name=None, trace=True))
        if len(steps_flat) < 2:
            raise ValueError(
                f"RunnableSeq must have at least 2 steps, got {len(steps_flat)}"
            )
        self.steps = steps_flat
        self.name = name
        self.trace_inputs = trace_inputs

    def __or__(
        self,
        other: Any,
    ) -> Runnable:
        if isinstance(other, RunnableSequence):
            return RunnableSeq(
                *self.steps,
                other.first,
                *other.middle,
                other.last,
                name=self.name or other.name,
            )
        elif isinstance(other, RunnableSeq):
            return RunnableSeq(
                *self.steps,
                *other.steps,
                name=self.name or other.name,
            )
        else:
            return RunnableSeq(
                *self.steps,
                coerce_to_runnable(other, name=None, trace=True),
                name=self.name,
            )

    def __ror__(
        self,
        other: Any,
    ) -> Runnable:
        if isinstance(other, RunnableSequence):
            return RunnableSequence(
                other.first,
                *other.middle,
                other.last,
                *self.steps,
                name=other.name or self.name,
            )
        elif isinstance(other, RunnableSeq):
            return RunnableSeq(
                *other.steps,
                *self.steps,
                name=other.name or self.name,
            )
        else:
            return RunnableSequence(
                coerce_to_runnable(other, name=None, trace=True),
                *self.steps,
                name=self.name,
            )

    def invoke(
        self, input: Input, config: RunnableConfig | None = None, **kwargs: Any
    ) -> Any:
        if config is None:
            config = ensure_config()
        # setup callbacks and context
        callback_manager = get_callback_manager_for_config(config)
        # start the root run
        run_manager = callback_manager.on_chain_start(
            None,
            self.trace_inputs(input) if self.trace_inputs is not None else input,
            name=config.get("run_name") or self.get_name(),
            run_id=config.pop("run_id", None),
        )
        # invoke all steps in sequence
        try:
            for i, step in enumerate(self.steps):
                # mark each step as a child run
                config = patch_config(
                    config, callbacks=run_manager.get_child(f"seq:step:{i + 1}")
                )
                # 1st step is the actual node,
                # others are writers which don't need to be run in context
                if i == 0:
                    # get the run object
                    for h in run_manager.handlers:
                        if isinstance(h, LangChainTracer):
                            run = h.run_map.get(str(run_manager.run_id))
                            break
                    else:
                        run = None
                    # run in context
                    with set_config_context(config, run) as context:
                        input = context.run(step.invoke, input, config, **kwargs)
                else:
                    input = step.invoke(input, config)
        # finish the root run
        except BaseException as e:
            run_manager.on_chain_error(e)
            raise
        else:
            run_manager.on_chain_end(input)
            return input

    async def ainvoke(
        self,
        input: Input,
        config: RunnableConfig | None = None,
        **kwargs: Any | None,
    ) -> Any:
        if config is None:
            config = ensure_config()
        # setup callbacks
        callback_manager = get_async_callback_manager_for_config(config)
        # start the root run
        run_manager = await callback_manager.on_chain_start(
            None,
            self.trace_inputs(input) if self.trace_inputs is not None else input,
            name=config.get("run_name") or self.get_name(),
            run_id=config.pop("run_id", None),
        )

        # invoke all steps in sequence
        try:
            for i, step in enumerate(self.steps):
                # mark each step as a child run
                config = patch_config(
                    config, callbacks=run_manager.get_child(f"seq:step:{i + 1}")
                )
                # 1st step is the actual node,
                # others are writers which don't need to be run in context
                if i == 0:
                    if ASYNCIO_ACCEPTS_CONTEXT:
                        # get the run object
                        for h in run_manager.handlers:
                            if isinstance(h, LangChainTracer):
                                run = h.run_map.get(str(run_manager.run_id))
                                break
                        else:
                            run = None
                        # run in context
                        with set_config_context(config, run) as context:
                            input = await asyncio.create_task(
                                step.ainvoke(input, config, **kwargs), context=context
                            )
                    else:
                        input = await step.ainvoke(input, config, **kwargs)
                else:
                    input = await step.ainvoke(input, config)
        # finish the root run
        except BaseException as e:
            await run_manager.on_chain_error(e)
            raise
        else:
            await run_manager.on_chain_end(input)
            return input

    def stream(
        self,
        input: Input,
        config: RunnableConfig | None = None,
        **kwargs: Any | None,
    ) -> Iterator[Any]:
        if config is None:
            config = ensure_config()
        # setup callbacks
        callback_manager = get_callback_manager_for_config(config)
        # start the root run
        run_manager = callback_manager.on_chain_start(
            None,
            self.trace_inputs(input) if self.trace_inputs is not None else input,
            name=config.get("run_name") or self.get_name(),
            run_id=config.pop("run_id", None),
        )
        # get the run object
        for h in run_manager.handlers:
            if isinstance(h, LangChainTracer):
                run = h.run_map.get(str(run_manager.run_id))
                break
        else:
            run = None
        # create first step config
        config = patch_config(
            config,
            callbacks=run_manager.get_child(f"seq:step:{1}"),
        )
        # run all in context
        with set_config_context(config, run) as context:
            try:
                # stream the last steps
                # transform the input stream of each step with the next
                # steps that don't natively support transforming an input stream will
                # buffer input in memory until all available, and then start emitting output
                for idx, step in enumerate(self.steps):
                    if idx == 0:
                        iterator = step.stream(input, config, **kwargs)
                    else:
                        config = patch_config(
                            config,
                            callbacks=run_manager.get_child(f"seq:step:{idx + 1}"),
                        )
                        iterator = step.transform(iterator, config)
                # populates streamed_output in astream_log() output if needed
                if _StreamingCallbackHandler is not None:
                    for h in run_manager.handlers:
                        if isinstance(h, _StreamingCallbackHandler):
                            iterator = h.tap_output_iter(run_manager.run_id, iterator)
                # consume into final output
                output = context.run(_consume_iter, iterator)
                # sequence doesn't emit output, yield to mark as generator
                yield
            except BaseException as e:
                run_manager.on_chain_error(e)
                raise
            else:
                run_manager.on_chain_end(output)

    async def astream(
        self,
        input: Input,
        config: RunnableConfig | None = None,
        **kwargs: Any | None,
    ) -> AsyncIterator[Any]:
        if config is None:
            config = ensure_config()
        # setup callbacks
        callback_manager = get_async_callback_manager_for_config(config)
        # start the root run
        run_manager = await callback_manager.on_chain_start(
            None,
            self.trace_inputs(input) if self.trace_inputs is not None else input,
            name=config.get("run_name") or self.get_name(),
            run_id=config.pop("run_id", None),
        )
        # stream the last steps
        # transform the input stream of each step with the next
        # steps that don't natively support transforming an input stream will
        # buffer input in memory until all available, and then start emitting output
        if ASYNCIO_ACCEPTS_CONTEXT:
            # get the run object
            for h in run_manager.handlers:
                if isinstance(h, LangChainTracer):
                    run = h.run_map.get(str(run_manager.run_id))
                    break
            else:
                run = None
            # create first step config
            config = patch_config(
                config,
                callbacks=run_manager.get_child(f"seq:step:{1}"),
            )
            # run all in context
            with set_config_context(config, run) as context:
                try:
                    async with AsyncExitStack() as stack:
                        for idx, step in enumerate(self.steps):
                            if idx == 0:
                                aiterator = step.astream(input, config, **kwargs)
                            else:
                                config = patch_config(
                                    config,
                                    callbacks=run_manager.get_child(
                                        f"seq:step:{idx + 1}"
                                    ),
                                )
                                aiterator = step.atransform(aiterator, config)
                            if hasattr(aiterator, "aclose"):
                                stack.push_async_callback(aiterator.aclose)
                        # populates streamed_output in astream_log() output if needed
                        if _StreamingCallbackHandler is not None:
                            for h in run_manager.handlers:
                                if isinstance(h, _StreamingCallbackHandler):
                                    aiterator = h.tap_output_aiter(
                                        run_manager.run_id, aiterator
                                    )
                        # consume into final output
                        output = await asyncio.create_task(
                            _consume_aiter(aiterator), context=context
                        )
                        # sequence doesn't emit output, yield to mark as generator
                        yield
                except BaseException as e:
                    await run_manager.on_chain_error(e)
                    raise
                else:
                    await run_manager.on_chain_end(output)
        else:
            try:
                async with AsyncExitStack() as stack:
                    for idx, step in enumerate(self.steps):
                        config = patch_config(
                            config,
                            callbacks=run_manager.get_child(f"seq:step:{idx + 1}"),
                        )
                        if idx == 0:
                            aiterator = step.astream(input, config, **kwargs)
                        else:
                            aiterator = step.atransform(aiterator, config)
                        if hasattr(aiterator, "aclose"):
                            stack.push_async_callback(aiterator.aclose)
                    # populates streamed_output in astream_log() output if needed
                    if _StreamingCallbackHandler is not None:
                        for h in run_manager.handlers:
                            if isinstance(h, _StreamingCallbackHandler):
                                aiterator = h.tap_output_aiter(
                                    run_manager.run_id, aiterator
                                )
                    # consume into final output
                    output = await _consume_aiter(aiterator)
                    # sequence doesn't emit output, yield to mark as generator
                    yield
            except BaseException as e:
                await run_manager.on_chain_error(e)
                raise
            else:
                await run_manager.on_chain_end(output)


def _consume_iter(it: Iterator[Any]) -> Any:
    """Consume an iterator."""
    output: Any = None
    add_supported = False
    for chunk in it:
        # collect final output
        if output is None:
            output = chunk
        elif add_supported:
            try:
                output = output + chunk
            except TypeError:
                output = chunk
                add_supported = False
        else:
            output = chunk
    return output


async def _consume_aiter(it: AsyncIterator[Any]) -> Any:
    """Consume an async iterator."""
    output: Any = None
    add_supported = False
    async for chunk in it:
        # collect final output
        if add_supported:
            try:
                output = output + chunk
            except TypeError:
                output = chunk
                add_supported = False
        else:
            output = chunk
    return output

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/_internal/_scratchpad.py
```py
import dataclasses
from collections.abc import Callable
from typing import Any

from langgraph.types import _DC_KWARGS


@dataclasses.dataclass(**_DC_KWARGS)
class PregelScratchpad:
    step: int
    stop: int
    # call
    call_counter: Callable[[], int]
    # interrupt
    interrupt_counter: Callable[[], int]
    get_null_resume: Callable[[bool], Any]
    resume: list[Any]
    # subgraph
    subgraph_counter: Callable[[], int]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/_internal/_typing.py
```py
"""Private typing utilities for LangGraph."""

from __future__ import annotations

from dataclasses import Field
from typing import Any, ClassVar, Protocol, TypeAlias

from pydantic import BaseModel
from typing_extensions import TypedDict


class TypedDictLikeV1(Protocol):
    """Protocol to represent types that behave like TypedDicts

    Version 1: using `ClassVar` for keys."""

    __required_keys__: ClassVar[frozenset[str]]
    __optional_keys__: ClassVar[frozenset[str]]


class TypedDictLikeV2(Protocol):
    """Protocol to represent types that behave like TypedDicts

    Version 2: not using `ClassVar` for keys."""

    __required_keys__: frozenset[str]
    __optional_keys__: frozenset[str]


class DataclassLike(Protocol):
    """Protocol to represent types that behave like dataclasses.

    Inspired by the private _DataclassT from dataclasses that uses a similar protocol as a bound."""

    __dataclass_fields__: ClassVar[dict[str, Field[Any]]]


StateLike: TypeAlias = TypedDictLikeV1 | TypedDictLikeV2 | DataclassLike | BaseModel
"""Type alias for state-like types.

It can either be a `TypedDict`, `dataclass`, or Pydantic `BaseModel`.
Note: we cannot use either `TypedDict` or `dataclass` directly due to limitations in type checking.
"""

MISSING = object()
"""Unset sentinel value."""


class DeprecatedKwargs(TypedDict):
    """TypedDict to use for extra keyword arguments, enabling type checking warnings for deprecated arguments."""


EMPTY_SEQ: tuple[str, ...] = tuple()
"""An empty sequence of strings."""

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/.claude/settings.local.json
```json
{
  "permissions": {
    "allow": [
      "Bash(rg:*)",
      "Bash(python:*)",
      "Bash(grep:*)",
      "Bash(sed:*)",
      "Bash(awk:*)",
      "Bash(uv run mypy:*)",
      "Bash(uv run:*)",
      "Bash(make test:*)",
      "Bash(make test_parallel:*)"
    ],
    "deny": []
  }
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/bench/__init__.py
```py

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/bench/__main__.py
```py
import random
from uuid import uuid4

from langchain_core.messages import HumanMessage
from langgraph.checkpoint.memory import InMemorySaver
from pyperf._runner import Runner
from uvloop import new_event_loop

from bench.fanout_to_subgraph import fanout_to_subgraph, fanout_to_subgraph_sync
from bench.pydantic_state import pydantic_state
from bench.react_agent import react_agent
from bench.sequential import create_sequential
from bench.wide_dict import wide_dict
from bench.wide_state import wide_state
from langgraph.graph import StateGraph
from langgraph.pregel import Pregel


async def arun(graph: Pregel, input: dict):
    len(
        [
            c
            async for c in graph.astream(
                input,
                {
                    "configurable": {"thread_id": str(uuid4())},
                    "recursion_limit": 1000000000,
                },
                durability="exit",
            )
        ]
    )


async def arun_first_event_latency(graph: Pregel, input: dict) -> None:
    """Latency for the first event.

    Run the graph until the first event is processed and then stop.
    """
    stream = graph.astream(
        input,
        {
            "configurable": {"thread_id": str(uuid4())},
            "recursion_limit": 1000000000,
        },
        durability="exit",
    )

    try:
        async for _ in stream:
            break
    finally:
        await stream.aclose()


def run(graph: Pregel, input: dict):
    len(
        [
            c
            for c in graph.stream(
                input,
                {
                    "configurable": {"thread_id": str(uuid4())},
                    "recursion_limit": 1000000000,
                },
                durability="exit",
            )
        ]
    )


def run_first_event_latency(graph: Pregel, input: dict) -> None:
    """Latency for the first event.

    Run the graph until the first event is processed and then stop.
    """
    stream = graph.stream(
        input,
        {
            "configurable": {"thread_id": str(uuid4())},
            "recursion_limit": 1000000000,
        },
        durability="exit",
    )

    try:
        for _ in stream:
            break
    finally:
        stream.close()


def compile_graph(graph: StateGraph) -> None:
    """Compile the graph."""
    graph.compile()


benchmarks = (
    (
        "fanout_to_subgraph_10x",
        fanout_to_subgraph().compile(checkpointer=None),
        fanout_to_subgraph_sync().compile(checkpointer=None),
        {
            "subjects": [
                random.choices("abcdefghijklmnopqrstuvwxyz", k=1000) for _ in range(10)
            ]
        },
    ),
    (
        "fanout_to_subgraph_10x_checkpoint",
        fanout_to_subgraph().compile(checkpointer=InMemorySaver()),
        fanout_to_subgraph_sync().compile(checkpointer=InMemorySaver()),
        {
            "subjects": [
                random.choices("abcdefghijklmnopqrstuvwxyz", k=1000) for _ in range(10)
            ]
        },
    ),
    (
        "fanout_to_subgraph_100x",
        fanout_to_subgraph().compile(checkpointer=None),
        fanout_to_subgraph_sync().compile(checkpointer=None),
        {
            "subjects": [
                random.choices("abcdefghijklmnopqrstuvwxyz", k=1000) for _ in range(100)
            ]
        },
    ),
    (
        "fanout_to_subgraph_100x_checkpoint",
        fanout_to_subgraph().compile(checkpointer=InMemorySaver()),
        fanout_to_subgraph_sync().compile(checkpointer=InMemorySaver()),
        {
            "subjects": [
                random.choices("abcdefghijklmnopqrstuvwxyz", k=1000) for _ in range(100)
            ]
        },
    ),
    (
        "react_agent_10x",
        react_agent(10, checkpointer=None),
        react_agent(10, checkpointer=None),
        {"messages": [HumanMessage("hi?")]},
    ),
    (
        "react_agent_10x_checkpoint",
        react_agent(10, checkpointer=InMemorySaver()),
        react_agent(10, checkpointer=InMemorySaver()),
        {"messages": [HumanMessage("hi?")]},
    ),
    (
        "react_agent_100x",
        react_agent(100, checkpointer=None),
        react_agent(100, checkpointer=None),
        {"messages": [HumanMessage("hi?")]},
    ),
    (
        "react_agent_100x_checkpoint",
        react_agent(100, checkpointer=InMemorySaver()),
        react_agent(100, checkpointer=InMemorySaver()),
        {"messages": [HumanMessage("hi?")]},
    ),
    (
        "wide_state_25x300",
        wide_state(300).compile(checkpointer=None),
        wide_state(300).compile(checkpointer=None),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(5)
                    }
                    for i in range(5)
                }
            ]
        },
    ),
    (
        "wide_state_25x300_checkpoint",
        wide_state(300).compile(checkpointer=InMemorySaver()),
        wide_state(300).compile(checkpointer=InMemorySaver()),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(5)
                    }
                    for i in range(5)
                }
            ]
        },
    ),
    (
        "wide_state_15x600",
        wide_state(600).compile(checkpointer=None),
        wide_state(600).compile(checkpointer=None),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(5)
                    }
                    for i in range(3)
                }
            ]
        },
    ),
    (
        "wide_state_15x600_checkpoint",
        wide_state(600).compile(checkpointer=InMemorySaver()),
        wide_state(600).compile(checkpointer=InMemorySaver()),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(5)
                    }
                    for i in range(3)
                }
            ]
        },
    ),
    (
        "wide_state_9x1200",
        wide_state(1200).compile(checkpointer=None),
        wide_state(1200).compile(checkpointer=None),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(3)
                    }
                    for i in range(3)
                }
            ]
        },
    ),
    (
        "wide_state_9x1200_checkpoint",
        wide_state(1200).compile(checkpointer=InMemorySaver()),
        wide_state(1200).compile(checkpointer=InMemorySaver()),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(3)
                    }
                    for i in range(3)
                }
            ]
        },
    ),
    (
        "wide_dict_25x300",
        wide_dict(300).compile(checkpointer=None),
        wide_dict(300).compile(checkpointer=None),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(5)
                    }
                    for i in range(5)
                }
            ]
        },
    ),
    (
        "wide_dict_25x300_checkpoint",
        wide_dict(300).compile(checkpointer=InMemorySaver()),
        wide_dict(300).compile(checkpointer=InMemorySaver()),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(5)
                    }
                    for i in range(5)
                }
            ]
        },
    ),
    (
        "wide_dict_15x600",
        wide_dict(600).compile(checkpointer=None),
        wide_dict(600).compile(checkpointer=None),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(5)
                    }
                    for i in range(3)
                }
            ]
        },
    ),
    (
        "wide_dict_15x600_checkpoint",
        wide_dict(600).compile(checkpointer=InMemorySaver()),
        wide_dict(600).compile(checkpointer=InMemorySaver()),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(5)
                    }
                    for i in range(3)
                }
            ]
        },
    ),
    (
        "wide_dict_9x1200",
        wide_dict(1200).compile(checkpointer=None),
        wide_dict(1200).compile(checkpointer=None),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(3)
                    }
                    for i in range(3)
                }
            ]
        },
    ),
    (
        "wide_dict_9x1200_checkpoint",
        wide_dict(1200).compile(checkpointer=InMemorySaver()),
        wide_dict(1200).compile(checkpointer=InMemorySaver()),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(3)
                    }
                    for i in range(3)
                }
            ]
        },
    ),
    (
        "sequential_10",
        create_sequential(10).compile(),
        create_sequential(10).compile(),
        {"messages": []},  # Empty list of messages
    ),
    (
        "sequential_1000",
        create_sequential(1000).compile(),
        create_sequential(1000).compile(),
        {"messages": []},  # Empty list of messages
    ),
    (
        "pydantic_state_25x300",
        pydantic_state(300).compile(checkpointer=None),
        pydantic_state(300).compile(checkpointer=None),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(5)
                    }
                    for i in range(5)
                }
            ]
        },
    ),
    (
        "pydantic_state_25x300_checkpoint",
        pydantic_state(300).compile(checkpointer=InMemorySaver()),
        pydantic_state(300).compile(checkpointer=InMemorySaver()),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(5)
                    }
                    for i in range(5)
                }
            ]
        },
    ),
    (
        "pydantic_state_15x600",
        pydantic_state(600).compile(checkpointer=None),
        pydantic_state(600).compile(checkpointer=None),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(5)
                    }
                    for i in range(3)
                }
            ]
        },
    ),
    (
        "pydantic_state_15x600_checkpoint",
        pydantic_state(600).compile(checkpointer=InMemorySaver()),
        pydantic_state(600).compile(checkpointer=InMemorySaver()),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(5)
                    }
                    for i in range(3)
                }
            ]
        },
    ),
    (
        "pydantic_state_9x1200",
        pydantic_state(1200).compile(checkpointer=None),
        pydantic_state(1200).compile(checkpointer=None),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(3)
                    }
                    for i in range(3)
                }
            ]
        },
    ),
    (
        "pydantic_state_9x1200_checkpoint",
        pydantic_state(1200).compile(checkpointer=InMemorySaver()),
        pydantic_state(1200).compile(checkpointer=InMemorySaver()),
        {
            "messages": [
                {
                    str(i) * 10: {
                        str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                        for j in range(3)
                    }
                    for i in range(3)
                }
            ]
        },
    ),
)


r = Runner()

# Full graph run time
for name, agraph, graph, input in benchmarks:
    r.bench_async_func(name, arun, agraph, input, loop_factory=new_event_loop)
    if graph is not None:
        r.bench_func(name + "_sync", run, graph, input)


# Pick a handful of graphs to measure the first event latency.
# At the moment, limiting just due to the size of the annotation on github.
GRAPHS_FOR_1st_EVENT_LATENCY = (
    "sequential_1000",
    "pydantic_state_25x300",
)

# First event latency
for name, agraph, graph, input in benchmarks:
    if graph not in GRAPHS_FOR_1st_EVENT_LATENCY:
        continue
    r.bench_async_func(
        name + "_first_event_latency",
        arun_first_event_latency,
        agraph,
        input,
        loop_factory=new_event_loop,
    )
    if graph is not None:
        r.bench_func(
            name + "_first_event_latency_sync", run_first_event_latency, graph, input
        )

# Graph compilation times
compilation_benchmarks = (
    (
        "sequential_1000",
        create_sequential(1_000),
    ),
    (
        "pydantic_state_25x300",
        pydantic_state(300),
    ),
    (
        "wide_state_15x600",
        wide_state(600),
    ),
)

for name, graph in compilation_benchmarks:
    r.bench_func(name + "_compilation", compile_graph, graph)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/bench/fanout_to_subgraph.py
```py
import operator
from typing import Annotated

from typing_extensions import TypedDict

from langgraph.constants import END, START
from langgraph.graph.state import StateGraph
from langgraph.types import Send


def fanout_to_subgraph() -> StateGraph:
    class OverallState(TypedDict):
        subjects: list[str]
        jokes: Annotated[list[str], operator.add]

    async def continue_to_jokes(state: OverallState):
        return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]

    class JokeInput(TypedDict):
        subject: str

    class JokeOutput(TypedDict):
        jokes: list[str]

    class JokeState(JokeInput, JokeOutput): ...

    async def bump(state: JokeOutput):
        return {"jokes": [state["jokes"][0] + " a"]}

    async def generate(state: JokeInput):
        return {"jokes": [f"Joke about {state['subject']}"]}

    async def edit(state: JokeInput):
        subject = state["subject"]
        return {"subject": f"{subject} - hohoho"}

    async def bump_loop(state: JokeOutput):
        return END if state["jokes"][0].endswith(" a" * 10) else "bump"

    # subgraph
    subgraph = StateGraph(JokeState, input_schema=JokeInput, output_schema=JokeOutput)
    subgraph.add_node("edit", edit)
    subgraph.add_node("generate", generate)
    subgraph.add_node("bump", bump)
    subgraph.set_entry_point("edit")
    subgraph.add_edge("edit", "generate")
    subgraph.add_edge("generate", "bump")
    subgraph.add_conditional_edges("bump", bump_loop)
    subgraph.set_finish_point("generate")
    subgraphc = subgraph.compile()

    # parent graph
    builder = StateGraph(OverallState)
    builder.add_node("generate_joke", subgraphc)
    builder.add_conditional_edges(START, continue_to_jokes)
    builder.add_edge("generate_joke", END)

    return builder


def fanout_to_subgraph_sync() -> StateGraph:
    class OverallState(TypedDict):
        subjects: list[str]
        jokes: Annotated[list[str], operator.add]

    def continue_to_jokes(state: OverallState):
        return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]

    class JokeInput(TypedDict):
        subject: str

    class JokeOutput(TypedDict):
        jokes: list[str]

    class JokeState(JokeInput, JokeOutput): ...

    def bump(state: JokeOutput):
        return {"jokes": [state["jokes"][0] + " a"]}

    def generate(state: JokeInput):
        return {"jokes": [f"Joke about {state['subject']}"]}

    def edit(state: JokeInput):
        subject = state["subject"]
        return {"subject": f"{subject} - hohoho"}

    def bump_loop(state: JokeOutput):
        return END if state["jokes"][0].endswith(" a" * 10) else "bump"

    # subgraph
    subgraph = StateGraph(JokeState, input_schema=JokeInput, output_schema=JokeOutput)
    subgraph.add_node("edit", edit)
    subgraph.add_node("generate", generate)
    subgraph.add_node("bump", bump)
    subgraph.set_entry_point("edit")
    subgraph.add_edge("edit", "generate")
    subgraph.add_edge("generate", "bump")
    subgraph.add_conditional_edges("bump", bump_loop)
    subgraph.set_finish_point("generate")
    subgraphc = subgraph.compile()

    # parent graph
    builder = StateGraph(OverallState)
    builder.add_node("generate_joke", subgraphc)
    builder.add_conditional_edges(START, continue_to_jokes)
    builder.add_edge("generate_joke", END)

    return builder


if __name__ == "__main__":
    import asyncio
    import random
    import time

    import uvloop
    from langgraph.checkpoint.memory import InMemorySaver

    graph = fanout_to_subgraph().compile(checkpointer=InMemorySaver())
    input = {
        "subjects": [
            random.choices("abcdefghijklmnopqrstuvwxyz", k=1000) for _ in range(1000)
        ]
    }
    config = {"configurable": {"thread_id": "1"}}

    async def run():
        len([c async for c in graph.astream(input, config=config)])

    uvloop.install()
    start = time.time()
    asyncio.run(run())
    end = time.time()
    print(f"Time taken: {end - start:.4f} seconds")

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/bench/pydantic_state.py
```py
import operator
from collections.abc import Sequence
from functools import partial
from random import choice
from typing import Annotated

from pydantic import BaseModel, Field, field_validator

from langgraph.constants import END, START
from langgraph.graph.state import StateGraph


def pydantic_state(n: int) -> StateGraph:
    class State(BaseModel):
        messages: Annotated[list, operator.add] = Field(default_factory=list)

        @field_validator("messages", mode="after")
        @classmethod
        def validate_messages(cls, v):
            if not isinstance(v, list):
                raise TypeError("messages must be a list")
            for msg in v:
                if not isinstance(msg, dict):
                    raise TypeError("messages must be a list of dicts")
                if not all(isinstance(k, str) for k in msg.keys()):
                    raise TypeError("messages must be a list of dicts with str keys")
            return v

        trigger_events: Annotated[list, operator.add] = Field(default_factory=list)
        """The external events that are converted by the graph."""

        @field_validator("trigger_events", mode="after")
        @classmethod
        def validate_trigger_events(cls, v):
            if not isinstance(v, list):
                raise TypeError("trigger_events must be a list")
            for event in v:
                if not isinstance(event, dict):
                    raise TypeError("trigger_events must be a list of dicts")
                if not all(isinstance(k, str) for k in event.keys()):
                    raise TypeError(
                        "trigger_events must be a list of dicts with str keys"
                    )
            return v

        primary_issue_medium: Annotated[str, lambda x, y: y or x] = Field(
            default="email"
        )
        """The primary issue medium for the current conversation."""

        @field_validator("primary_issue_medium", mode="after")
        @classmethod
        def validate_primary_issue_medium(cls, v):
            if not isinstance(v, str):
                raise TypeError("primary_issue_medium must be a string")
            return v

        autoresponse: Annotated[dict | None, lambda _, y: y] = Field(
            default=None
        )  # Always overwrite

        @field_validator("autoresponse", mode="after")
        @classmethod
        def validate_autoresponse(cls, v):
            if v is not None and not isinstance(v, dict):
                raise TypeError("autoresponse must be a dict or None")
            return v

        issue: Annotated[dict | None, lambda x, y: y if y else x] = Field(default=None)

        @field_validator("issue", mode="after")
        @classmethod
        def validate_issue(cls, v):
            if v is not None and not isinstance(v, dict):
                raise TypeError("issue must be a dict or None")
            return v

        relevant_rules: list[dict] | None = Field(default=None)
        """SOPs fetched from the rulebook that are relevant to the current conversation."""

        @field_validator("relevant_rules", mode="after")
        @classmethod
        def validate_relevant_rules(cls, v):
            if v is None:
                return v
            if not isinstance(v, list):
                raise TypeError("relevant_rules must be a list or None")
            for rule in v:
                if not isinstance(rule, dict):
                    raise TypeError("relevant_rules must be a list of dicts")
                if not all(isinstance(k, str) for k in rule.keys()):
                    raise TypeError(
                        "relevant_rules must be a list of dicts with str keys"
                    )
            return v

        memory_docs: list[dict] | None = Field(default=None)
        """Memory docs fetched from the memory service that are relevant to the current conversation."""

        @field_validator("memory_docs", mode="after")
        @classmethod
        def validate_memory_docs(cls, v):
            if v is None:
                return v
            if not isinstance(v, list):
                raise TypeError("memory_docs must be a list or None")
            for doc in v:
                if not isinstance(doc, dict):
                    raise TypeError("memory_docs must be a list of dicts")
                if not all(isinstance(k, str) for k in doc.keys()):
                    raise TypeError("memory_docs must be a list of dicts with str keys")
            return v

        categorizations: Annotated[list[dict], operator.add] = Field(
            default_factory=list
        )
        """The issue categorizations auto-generated by the AI."""

        @field_validator("categorizations", mode="after")
        @classmethod
        def validate_categorizations(cls, v):
            if not isinstance(v, list):
                raise TypeError("categorizations must be a list")
            for categorization in v:
                if not isinstance(categorization, dict):
                    raise TypeError("categorizations must be a list of dicts")
                if not all(isinstance(k, str) for k in categorization.keys()):
                    raise TypeError(
                        "categorizations must be a list of dicts with str keys"
                    )
            return v

        responses: Annotated[list[dict], operator.add] = Field(default_factory=list)
        """The draft responses recommended by the AI."""

        @field_validator("responses", mode="after")
        @classmethod
        def validate_responses(cls, v):
            if not isinstance(v, list):
                raise TypeError("responses must be a list")
            for response in v:
                if not isinstance(response, dict):
                    raise TypeError("responses must be a list of dicts")
                if not all(isinstance(k, str) for k in response.keys()):
                    raise TypeError("responses must be a list of dicts with str keys")
            return v

        user_info: Annotated[dict | None, lambda x, y: y if y is not None else x] = (
            Field(default=None)
        )
        """The current user state (by email)."""

        @field_validator("user_info", mode="after")
        @classmethod
        def validate_user_info(cls, v):
            if v is not None and not isinstance(v, dict):
                raise TypeError("user_info must be a dict or None")
            return v

        crm_info: Annotated[dict | None, lambda x, y: y if y is not None else x] = (
            Field(default=None)
        )
        """The CRM information for organization the current user is from."""

        @field_validator("crm_info", mode="after")
        @classmethod
        def validate_crm_info(cls, v):
            if v is not None and not isinstance(v, dict):
                raise TypeError("crm_info must be a dict or None")
            return v

        email_thread_id: Annotated[
            str | None, lambda x, y: y if y is not None else x
        ] = Field(default=None)
        """The current email thread ID."""

        @field_validator("email_thread_id", mode="after")
        @classmethod
        def validate_email_thread_id(cls, v):
            if v is not None and not isinstance(v, str):
                raise TypeError("email_thread_id must be a string or None")
            return v

        slack_participants: Annotated[dict, operator.or_] = Field(default_factory=dict)
        """The growing list of current slack participants."""

        @field_validator("slack_participants", mode="after")
        @classmethod
        def validate_slack_participants(cls, v):
            if not isinstance(v, dict):
                raise TypeError("slack_participants must be a dict")
            for participant in v:
                if not isinstance(participant, str):
                    raise TypeError("slack_participants must be a dict with str keys")
            return v

        bot_id: str | None = Field(default=None)
        """The ID of the bot user in the slack channel."""

        @field_validator("bot_id", mode="after")
        @classmethod
        def validate_bot_id(cls, v):
            if v is not None and not isinstance(v, str):
                raise TypeError("bot_id must be a string or None")
            return v

        notified_assignees: Annotated[dict, operator.or_] = Field(default_factory=dict)

        @field_validator("notified_assignees", mode="after")
        def validate_notified_assignees(cls, v):
            if not isinstance(v, dict):
                raise TypeError("notified_assignees must be a dict")
            for assignee in v:
                if not isinstance(assignee, str):
                    raise TypeError("notified_assignees must be a dict with str keys")
            return v

    list_fields = {
        "messages",
        "trigger_events",
        "categorizations",
        "responses",
        "memory_docs",
        "relevant_rules",
    }
    dict_fields = {
        "user_info",
        "crm_info",
        "slack_participants",
        "notified_assignees",
        "autoresponse",
        "issue",
    }

    def read_write(read: str, write: Sequence[str], input: State) -> dict:
        val = getattr(input, read)
        val = {val: val} if isinstance(val, str) else val
        val_single = val[-1] if isinstance(val, list) else val
        val_list = val if isinstance(val, list) else [val]
        return {
            k: val_list
            if k in list_fields
            else val_single
            if k in dict_fields
            else "".join(choice("abcdefghijklmnopqrstuvwxyz") for _ in range(n))
            for k in write
        }

    builder = StateGraph(State)
    builder.add_edge(START, "one")
    builder.add_node(
        "one",
        partial(read_write, "messages", ["trigger_events", "primary_issue_medium"]),
    )
    builder.add_edge("one", "two")
    builder.add_node(
        "two",
        partial(read_write, "trigger_events", ["autoresponse", "issue"]),
    )
    builder.add_edge("two", "three")
    builder.add_edge("two", "four")
    builder.add_node(
        "three",
        partial(read_write, "autoresponse", ["relevant_rules"]),
    )
    builder.add_node(
        "four",
        partial(
            read_write,
            "trigger_events",
            ["categorizations", "responses", "memory_docs"],
        ),
    )
    builder.add_node(
        "five",
        partial(
            read_write,
            "categorizations",
            [
                "user_info",
                "crm_info",
                "email_thread_id",
                "slack_participants",
                "bot_id",
                "notified_assignees",
            ],
        ),
    )
    builder.add_edge(["three", "four"], "five")
    builder.add_edge("five", "six")
    builder.add_node(
        "six",
        partial(read_write, "responses", ["messages"]),
    )
    builder.add_conditional_edges(
        "six", lambda state: END if len(state.messages) > n else "one"
    )

    return builder


if __name__ == "__main__":
    import asyncio

    import uvloop
    from langgraph.checkpoint.memory import InMemorySaver

    graph = pydantic_state(1000).compile(checkpointer=InMemorySaver())
    input = {
        "messages": [
            {
                str(i) * 10: {
                    str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                    for j in range(5)
                }
                for i in range(5)
            }
        ]
    }
    config = {"configurable": {"thread_id": "1"}, "recursion_limit": 20000000000}

    async def run():
        async for c in graph.astream(input, config=config):
            print(c.keys())

    uvloop.install()
    asyncio.run(run())

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/bench/react_agent.py
```py
from typing import Any
from uuid import uuid4

from langchain_core.callbacks import CallbackManagerForLLMRun
from langchain_core.language_models.fake_chat_models import (
    FakeMessagesListChatModel,
)
from langchain_core.messages import AIMessage, BaseMessage, HumanMessage
from langchain_core.outputs import ChatGeneration, ChatResult
from langchain_core.tools import StructuredTool
from langgraph.checkpoint.base import BaseCheckpointSaver
from langgraph.prebuilt.chat_agent_executor import create_react_agent

from langgraph.pregel import Pregel


def react_agent(n_tools: int, checkpointer: BaseCheckpointSaver | None) -> Pregel:
    class FakeFunctionChatModel(FakeMessagesListChatModel):
        def bind_tools(self, functions: list):
            return self

        def _generate(
            self,
            messages: list[BaseMessage],
            stop: list[str] | None = None,
            run_manager: CallbackManagerForLLMRun | None = None,
            **kwargs: Any,
        ) -> ChatResult:
            response = self.responses[self.i].copy()
            if self.i < len(self.responses) - 1:
                self.i += 1
            else:
                self.i = 0
            generation = ChatGeneration(message=response)
            return ChatResult(generations=[generation])

    tool = StructuredTool.from_function(
        lambda query: f"result for query: {query}" * 10,
        name=str(uuid4()),
        description="",
    )

    model = FakeFunctionChatModel(
        responses=[
            AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": str(uuid4()),
                        "name": tool.name,
                        "args": {"query": str(uuid4()) * 100},
                    }
                ],
                id=str(uuid4()),
            )
            for _ in range(n_tools)
        ]
        + [
            AIMessage(content="answer" * 100, id=str(uuid4())),
        ]
    )

    return create_react_agent(model, [tool], checkpointer=checkpointer)


if __name__ == "__main__":
    import asyncio

    import uvloop
    from langgraph.checkpoint.memory import InMemorySaver

    graph = react_agent(100, checkpointer=InMemorySaver())
    input = {"messages": [HumanMessage("hi?")]}
    config = {"configurable": {"thread_id": "1"}, "recursion_limit": 20000000000}

    async def run():
        len([c async for c in graph.astream(input, config=config)])

    uvloop.install()
    asyncio.run(run())

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/bench/sequential.py
```py
"""Create a sequential no-op graph consisting of a few hundred nodes."""

from langgraph._internal._runnable import RunnableCallable
from langgraph.graph import MessagesState, StateGraph


def create_sequential(number_nodes: int) -> StateGraph:
    """Create a sequential no-op graph consisting of a few hundred nodes."""
    builder = StateGraph(MessagesState)

    def noop(state: MessagesState) -> None:
        """No-op function."""
        pass

    async def anoop(state: MessagesState) -> None:
        """No-op function."""
        pass

    prev_node = "__start__"

    for i in range(number_nodes):
        name = f"node_{i}"
        builder.add_node(name, RunnableCallable(noop, anoop))
        builder.add_edge(prev_node, name)
        prev_node = name

    builder.add_edge(prev_node, "__end__")
    return builder


if __name__ == "__main__":
    import asyncio
    import time

    import uvloop

    graph = create_sequential(3000).compile()
    input = {"messages": []}  # Empty list of messages
    config = {"recursion_limit": 20000000000}

    async def run():
        len([c async for c in graph.astream(input, config=config)])

    uvloop.install()
    start = time.time()
    asyncio.run(run())
    end = time.time()
    print(f"Time taken: {end - start:.4f} seconds")

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/bench/wide_dict.py
```py
import operator
from collections.abc import Sequence
from functools import partial
from random import choice
from typing import Annotated

from typing_extensions import TypedDict

from langgraph.constants import END, START
from langgraph.graph.state import StateGraph


def wide_dict(n: int) -> StateGraph:
    class State(TypedDict):
        messages: Annotated[list, operator.add]
        trigger_events: Annotated[list, operator.add]
        """The external events that are converted by the graph."""
        primary_issue_medium: Annotated[str, lambda x, y: y or x]
        autoresponse: Annotated[dict | None, lambda _, y: y]  # Always overwrite
        issue: Annotated[dict | None, lambda x, y: y if y else x]
        relevant_rules: list[dict] | None
        """SOPs fetched from the rulebook that are relevant to the current conversation."""
        memory_docs: list[dict] | None
        """Memory docs fetched from the memory service that are relevant to the current conversation."""
        categorizations: Annotated[list[dict], operator.add]
        """The issue categorizations auto-generated by the AI."""
        responses: Annotated[list[dict], operator.add]
        """The draft responses recommended by the AI."""

        user_info: Annotated[dict | None, lambda x, y: y if y is not None else x]
        """The current user state (by email)."""
        crm_info: Annotated[dict | None, lambda x, y: y if y is not None else x]
        """The CRM information for organization the current user is from."""
        email_thread_id: Annotated[str | None, lambda x, y: y if y is not None else x]
        """The current email thread ID."""
        slack_participants: Annotated[dict, operator.or_]
        """The growing list of current slack participants."""
        bot_id: str | None
        """The ID of the bot user in the slack channel."""
        notified_assignees: Annotated[dict, operator.or_]

    list_fields = {
        "messages",
        "trigger_events",
        "categorizations",
        "responses",
        "memory_docs",
        "relevant_rules",
    }
    dict_fields = {
        "user_info",
        "crm_info",
        "slack_participants",
        "notified_assignees",
        "autoresponse",
        "issue",
    }

    def read_write(read: str, write: Sequence[str], input: State) -> dict:
        val = input.get(read)
        val = {val: val} if isinstance(val, str) else val
        val_single = val[-1] if isinstance(val, list) else val
        val_list = val if isinstance(val, list) else [val]
        return {
            k: val_list
            if k in list_fields
            else val_single
            if k in dict_fields
            else "".join(choice("abcdefghijklmnopqrstuvwxyz") for _ in range(n))
            for k in write
        }

    builder = StateGraph(State)
    builder.add_edge(START, "one")
    builder.add_node(
        "one",
        partial(read_write, "messages", ["trigger_events", "primary_issue_medium"]),
    )
    builder.add_edge("one", "two")
    builder.add_node(
        "two",
        partial(read_write, "trigger_events", ["autoresponse", "issue"]),
    )
    builder.add_edge("two", "three")
    builder.add_edge("two", "four")
    builder.add_node(
        "three",
        partial(read_write, "autoresponse", ["relevant_rules"]),
    )
    builder.add_node(
        "four",
        partial(
            read_write,
            "trigger_events",
            ["categorizations", "responses", "memory_docs"],
        ),
    )
    builder.add_node(
        "five",
        partial(
            read_write,
            "categorizations",
            [
                "user_info",
                "crm_info",
                "email_thread_id",
                "slack_participants",
                "bot_id",
                "notified_assignees",
            ],
        ),
    )
    builder.add_edge(["three", "four"], "five")
    builder.add_edge("five", "six")
    builder.add_node(
        "six",
        partial(read_write, "responses", ["messages"]),
    )
    builder.add_conditional_edges(
        "six", lambda state: END if len(state["messages"]) > n else "one"
    )

    return builder


if __name__ == "__main__":
    import asyncio

    import uvloop
    from langgraph.checkpoint.memory import InMemorySaver

    graph = wide_dict(1000).compile(checkpointer=InMemorySaver())
    input = {
        "messages": [
            {
                str(i) * 10: {
                    str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                    for j in range(50)
                }
                for i in range(50)
            }
        ]
    }
    config = {"configurable": {"thread_id": "1"}, "recursion_limit": 20000000000}

    async def run():
        async for c in graph.astream(input, config=config):
            print(c.keys())

    uvloop.install()
    asyncio.run(run())

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/bench/wide_state.py
```py
import operator
from collections.abc import Sequence
from dataclasses import dataclass, field
from functools import partial
from random import choice
from typing import Annotated

from langgraph.constants import END, START
from langgraph.graph.state import StateGraph


def wide_state(n: int) -> StateGraph:
    @dataclass(kw_only=True)
    class State:
        messages: Annotated[list, operator.add] = field(default_factory=list)
        trigger_events: Annotated[list, operator.add] = field(default_factory=list)
        """The external events that are converted by the graph."""
        primary_issue_medium: Annotated[str, lambda x, y: y or x] = field(
            default="email"
        )
        autoresponse: Annotated[dict | None, lambda _, y: y] = field(
            default=None
        )  # Always overwrite
        issue: Annotated[dict | None, lambda x, y: y if y else x] = field(default=None)
        relevant_rules: list[dict] | None = field(default=None)
        """SOPs fetched from the rulebook that are relevant to the current conversation."""
        memory_docs: list[dict] | None = field(default=None)
        """Memory docs fetched from the memory service that are relevant to the current conversation."""
        categorizations: Annotated[list[dict], operator.add] = field(
            default_factory=list
        )
        """The issue categorizations auto-generated by the AI."""
        responses: Annotated[list[dict], operator.add] = field(default_factory=list)
        """The draft responses recommended by the AI."""

        user_info: Annotated[dict | None, lambda x, y: y if y is not None else x] = (
            field(default=None)
        )
        """The current user state (by email)."""
        crm_info: Annotated[dict | None, lambda x, y: y if y is not None else x] = (
            field(default=None)
        )
        """The CRM information for organization the current user is from."""
        email_thread_id: Annotated[
            str | None, lambda x, y: y if y is not None else x
        ] = field(default=None)
        """The current email thread ID."""
        slack_participants: Annotated[dict, operator.or_] = field(default_factory=dict)
        """The growing list of current slack participants."""
        bot_id: str | None = field(default=None)
        """The ID of the bot user in the slack channel."""
        notified_assignees: Annotated[dict, operator.or_] = field(default_factory=dict)

    list_fields = {
        "messages",
        "trigger_events",
        "categorizations",
        "responses",
        "memory_docs",
        "relevant_rules",
    }
    dict_fields = {
        "user_info",
        "crm_info",
        "slack_participants",
        "notified_assignees",
        "autoresponse",
        "issue",
    }

    def read_write(read: str, write: Sequence[str], input: State) -> dict:
        val = getattr(input, read)
        val = {val: val} if isinstance(val, str) else val
        val_single = val[-1] if isinstance(val, list) else val
        val_list = val if isinstance(val, list) else [val]
        return {
            k: val_list
            if k in list_fields
            else val_single
            if k in dict_fields
            else "".join(choice("abcdefghijklmnopqrstuvwxyz") for _ in range(n))
            for k in write
        }

    builder = StateGraph(State)
    builder.add_edge(START, "one")
    builder.add_node(
        "one",
        partial(read_write, "messages", ["trigger_events", "primary_issue_medium"]),
    )
    builder.add_edge("one", "two")
    builder.add_node(
        "two",
        partial(read_write, "trigger_events", ["autoresponse", "issue"]),
    )
    builder.add_edge("two", "three")
    builder.add_edge("two", "four")
    builder.add_node(
        "three",
        partial(read_write, "autoresponse", ["relevant_rules"]),
    )
    builder.add_node(
        "four",
        partial(
            read_write,
            "trigger_events",
            ["categorizations", "responses", "memory_docs"],
        ),
    )
    builder.add_node(
        "five",
        partial(
            read_write,
            "categorizations",
            [
                "user_info",
                "crm_info",
                "email_thread_id",
                "slack_participants",
                "bot_id",
                "notified_assignees",
            ],
        ),
    )
    builder.add_edge(["three", "four"], "five")
    builder.add_edge("five", "six")
    builder.add_node(
        "six",
        partial(read_write, "responses", ["messages"]),
    )
    builder.add_conditional_edges(
        "six", lambda state: END if len(state.messages) > n else "one"
    )

    return builder


if __name__ == "__main__":
    import asyncio

    import uvloop
    from langgraph.checkpoint.memory import InMemorySaver

    graph = wide_state(1000).compile(checkpointer=InMemorySaver())
    input = {
        "messages": [
            {
                str(i) * 10: {
                    str(j) * 10: ["hi?" * 10, True, 1, 6327816386138, None] * 5
                    for j in range(50)
                }
                for i in range(50)
            }
        ]
    }
    config = {"configurable": {"thread_id": "1"}, "recursion_limit": 20000000000}

    async def run():
        async for c in graph.astream(input, config=config):
            print(c.keys())

    uvloop.install()
    asyncio.run(run())

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/__init__.py
```py

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/agents.py
```py
from typing import Literal

from pydantic import BaseModel


# define these objects to avoid importing langchain_core.agents
# and therefore avoid relying on core Pydantic version
class AgentAction(BaseModel):
    """
    Represents a request to execute an action by an agent.

    The action consists of the name of the tool to execute and the input to pass
    to the tool. The log is used to pass along extra information about the action.
    """

    tool: str
    tool_input: str | dict
    log: str
    type: Literal["AgentAction"] = "AgentAction"


class AgentFinish(BaseModel):
    """Final return value of an ActionAgent.

    Agents return an AgentFinish when they have reached a stopping condition.
    """

    return_values: dict
    log: str
    type: Literal["AgentFinish"] = "AgentFinish"

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/any_int.py
```py
class AnyInt(int):
    def __init__(self) -> None:
        super().__init__()

    def __eq__(self, other: object) -> bool:
        return isinstance(other, int)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/any_str.py
```py
import re
from collections.abc import Sequence
from typing import Any

from typing_extensions import Self


class AnyObject:
    def __eq__(self, value):
        return True


class FloatBetween(float):
    def __new__(cls, min_value: float, max_value: float) -> Self:
        return super().__new__(cls, min_value)

    def __init__(self, min_value: float, max_value: float) -> None:
        super().__init__()
        self.min_value = min_value
        self.max_value = max_value

    def __eq__(self, other: object) -> bool:
        return (
            isinstance(other, float)
            and other >= self.min_value
            and other <= self.max_value
        )

    def __hash__(self) -> int:
        return hash((float(self), self.min_value, self.max_value))


class AnyStr(str):
    def __init__(self, prefix: str | re.Pattern = "") -> None:
        super().__init__()
        self.prefix = prefix

    def __eq__(self, other: object) -> bool:
        return isinstance(other, str) and (
            other.startswith(self.prefix)
            if isinstance(self.prefix, str)
            else self.prefix.match(other)
        )

    def __hash__(self) -> int:
        return hash((str(self), self.prefix))


class AnyDict(dict):
    def __init__(self, *args, **kwargs) -> None:
        super().__init__(*args, **kwargs)

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, dict) or len(self) != len(other):
            return False
        for k, v in self.items():
            if kk := next((kk for kk in other if kk == k), None):
                if v == other[kk]:
                    continue
                else:
                    return False
        else:
            return True


class AnyVersion:
    def __init__(self) -> None:
        super().__init__()

    def __eq__(self, other: object) -> bool:
        return isinstance(other, (str, int, float))

    def __hash__(self) -> int:
        return hash(str(self))


class UnsortedSequence:
    def __init__(self, *values: Any) -> None:
        self.seq = values

    def __eq__(self, value: object) -> bool:
        return (
            isinstance(value, Sequence)
            and len(self.seq) == len(value)
            and all(a in value for a in self.seq)
        )

    def __hash__(self) -> int:
        return hash(frozenset(self.seq))

    def __repr__(self) -> str:
        return repr(self.seq)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/compose-postgres.yml
```yml
name: langgraph-tests
services:
  postgres-test:
    image: postgres:16
    ports:
      - "5442:5432"
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    healthcheck:
      test: pg_isready -U postgres
      start_period: 10s
      timeout: 1s
      retries: 5
      interval: 60s
      start_interval: 1s

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/compose-redis.yml
```yml
name: langgraph-tests
services:
  redis-test:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    healthcheck:
      test: redis-cli ping
      start_period: 10s
      timeout: 1s
      retries: 5
      interval: 5s
      start_interval: 1s
    tmpfs:
      - /data  # Use tmpfs for faster testing

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/conftest.py
```py
import os
from collections.abc import AsyncIterator, Iterator
from uuid import UUID

import pytest
import redis
from langgraph.cache.base import BaseCache
from langgraph.cache.memory import InMemoryCache
from langgraph.cache.redis import RedisCache
from langgraph.cache.sqlite import SqliteCache
from langgraph.checkpoint.base import BaseCheckpointSaver
from langgraph.store.base import BaseStore
from pytest_mock import MockerFixture

from langgraph.types import Durability
from tests.conftest_checkpointer import (
    _checkpointer_memory,
    _checkpointer_memory_migrate_sends,
    _checkpointer_postgres,
    _checkpointer_postgres_aio,
    _checkpointer_postgres_aio_pipe,
    _checkpointer_postgres_aio_pool,
    _checkpointer_postgres_pipe,
    _checkpointer_postgres_pool,
    _checkpointer_sqlite,
    _checkpointer_sqlite_aes,
    _checkpointer_sqlite_aio,
)
from tests.conftest_store import (
    _store_memory,
    _store_postgres,
    _store_postgres_aio,
    _store_postgres_aio_pipe,
    _store_postgres_aio_pool,
    _store_postgres_pipe,
    _store_postgres_pool,
)

NO_DOCKER = os.getenv("NO_DOCKER", "false") == "true"


@pytest.fixture
def anyio_backend():
    return "asyncio"


@pytest.fixture()
def deterministic_uuids(mocker: MockerFixture) -> MockerFixture:
    side_effect = (
        UUID(f"00000000-0000-4000-8000-{i:012}", version=4) for i in range(10000)
    )
    return mocker.patch("uuid.uuid4", side_effect=side_effect)


@pytest.fixture(params=["sync", "async", "exit"])
def durability(request: pytest.FixtureRequest) -> Durability:
    return request.param


@pytest.fixture(
    scope="function",
    params=["sqlite", "memory"] if NO_DOCKER else ["sqlite", "memory", "redis"],
)
def cache(request: pytest.FixtureRequest) -> Iterator[BaseCache]:
    if request.param == "sqlite":
        yield SqliteCache(path=":memory:")
    elif request.param == "memory":
        yield InMemoryCache()
    elif request.param == "redis":
        # Get worker ID for parallel test isolation
        worker_id = getattr(request.config, "workerinput", {}).get("workerid", "master")

        redis_client = redis.Redis(
            host="localhost", port=6379, db=0, decode_responses=False
        )
        # Use worker-specific prefix to avoid cache pollution between parallel tests
        cache = RedisCache(redis_client, prefix=f"test:cache:{worker_id}:")
        yield cache

        try:
            # Only clear keys with our specific prefix
            pattern = f"test:cache:{worker_id}:*"
            keys = redis_client.keys(pattern)
            if keys:
                redis_client.delete(*keys)
        except Exception:
            pass
    else:
        raise ValueError(f"Unknown cache type: {request.param}")


@pytest.fixture(
    scope="function",
    params=["in_memory"]
    if NO_DOCKER
    else ["in_memory", "postgres", "postgres_pipe", "postgres_pool"],
)
def sync_store(request: pytest.FixtureRequest) -> Iterator[BaseStore]:
    store_name = request.param
    if store_name is None:
        yield None
    elif store_name == "in_memory":
        with _store_memory() as store:
            yield store
    elif store_name == "postgres":
        with _store_postgres() as store:
            yield store
    elif store_name == "postgres_pipe":
        with _store_postgres_pipe() as store:
            yield store
    elif store_name == "postgres_pool":
        with _store_postgres_pool() as store:
            yield store
    else:
        raise NotImplementedError(f"Unknown store {store_name}")


@pytest.fixture(
    scope="function",
    params=["in_memory"]
    if NO_DOCKER
    else ["in_memory", "postgres_aio", "postgres_aio_pipe", "postgres_aio_pool"],
)
async def async_store(request: pytest.FixtureRequest) -> AsyncIterator[BaseStore]:
    store_name = request.param
    if store_name is None:
        yield None
    elif store_name == "in_memory":
        with _store_memory() as store:
            yield store
    elif store_name == "postgres_aio":
        async with _store_postgres_aio() as store:
            yield store
    elif store_name == "postgres_aio_pipe":
        async with _store_postgres_aio_pipe() as store:
            yield store
    elif store_name == "postgres_aio_pool":
        async with _store_postgres_aio_pool() as store:
            yield store
    else:
        raise NotImplementedError(f"Unknown store {store_name}")


@pytest.fixture(
    scope="function",
    params=[
        "memory",
        "sqlite",
        "sqlite_aes",
    ]
    if NO_DOCKER
    else [
        "memory",
        "memory_migrate_sends",
        "sqlite",
        "sqlite_aes",
        "postgres",
        "postgres_pipe",
        "postgres_pool",
    ],
)
def sync_checkpointer(
    request: pytest.FixtureRequest,
) -> Iterator[BaseCheckpointSaver]:
    checkpointer_name = request.param
    if checkpointer_name == "memory":
        with _checkpointer_memory() as checkpointer:
            yield checkpointer
    elif checkpointer_name == "memory_migrate_sends":
        with _checkpointer_memory_migrate_sends() as checkpointer:
            yield checkpointer
    elif checkpointer_name == "sqlite":
        with _checkpointer_sqlite() as checkpointer:
            yield checkpointer
    elif checkpointer_name == "sqlite_aes":
        with _checkpointer_sqlite_aes() as checkpointer:
            yield checkpointer
    elif checkpointer_name == "postgres":
        with _checkpointer_postgres() as checkpointer:
            yield checkpointer
    elif checkpointer_name == "postgres_pipe":
        with _checkpointer_postgres_pipe() as checkpointer:
            yield checkpointer
    elif checkpointer_name == "postgres_pool":
        with _checkpointer_postgres_pool() as checkpointer:
            yield checkpointer
    else:
        raise NotImplementedError(f"Unknown checkpointer: {checkpointer_name}")


@pytest.fixture(
    scope="function",
    params=[
        "memory",
        "sqlite_aio",
    ]
    if NO_DOCKER
    else [
        "memory",
        "sqlite_aio",
        "postgres_aio",
        "postgres_aio_pipe",
        "postgres_aio_pool",
    ],
)
async def async_checkpointer(
    request: pytest.FixtureRequest,
) -> AsyncIterator[BaseCheckpointSaver]:
    checkpointer_name = request.param
    if checkpointer_name == "memory":
        with _checkpointer_memory() as checkpointer:
            yield checkpointer
    elif checkpointer_name == "sqlite_aio":
        async with _checkpointer_sqlite_aio() as checkpointer:
            yield checkpointer
    elif checkpointer_name == "postgres_aio":
        async with _checkpointer_postgres_aio() as checkpointer:
            yield checkpointer
    elif checkpointer_name == "postgres_aio_pipe":
        async with _checkpointer_postgres_aio_pipe() as checkpointer:
            yield checkpointer
    elif checkpointer_name == "postgres_aio_pool":
        async with _checkpointer_postgres_aio_pool() as checkpointer:
            yield checkpointer
    else:
        raise NotImplementedError(f"Unknown checkpointer: {checkpointer_name}")

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/conftest_checkpointer.py
```py
from contextlib import asynccontextmanager, contextmanager
from uuid import uuid4

import pytest
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.checkpoint.sqlite.aio import AsyncSqliteSaver
from psycopg import AsyncConnection, Connection
from psycopg_pool import AsyncConnectionPool, ConnectionPool

pytest.register_assert_rewrite("tests.memory_assert")

from tests.memory_assert import (  # noqa: E402
    MemorySaverAssertImmutable,
    MemorySaverNeedsPendingSendsMigration,
)

DEFAULT_POSTGRES_URI = "postgres://postgres:postgres@localhost:5442/"


@contextmanager
def _checkpointer_memory():
    yield MemorySaverAssertImmutable()


@contextmanager
def _checkpointer_memory_migrate_sends():
    yield MemorySaverNeedsPendingSendsMigration()


@contextmanager
def _checkpointer_sqlite():
    with SqliteSaver.from_conn_string(":memory:") as checkpointer:
        yield checkpointer


@contextmanager
def _checkpointer_sqlite_aes():
    with SqliteSaver.from_conn_string(":memory:") as checkpointer:
        checkpointer.serde = EncryptedSerializer.from_pycryptodome_aes(
            key=b"1234567890123456"
        )
        yield checkpointer


@contextmanager
def _checkpointer_postgres():
    database = f"test_{uuid4().hex[:16]}"
    # create unique db
    with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
        conn.execute(f"CREATE DATABASE {database}")
    try:
        # yield checkpointer
        with PostgresSaver.from_conn_string(
            DEFAULT_POSTGRES_URI + database
        ) as checkpointer:
            checkpointer.setup()
            yield checkpointer
    finally:
        # drop unique db
        with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
            conn.execute(f"DROP DATABASE {database}")


@contextmanager
def _checkpointer_postgres_pipe():
    database = f"test_{uuid4().hex[:16]}"
    # create unique db
    with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
        conn.execute(f"CREATE DATABASE {database}")
    try:
        # yield checkpointer
        with PostgresSaver.from_conn_string(
            DEFAULT_POSTGRES_URI + database
        ) as checkpointer:
            checkpointer.setup()
            # setup can't run inside pipeline because of implicit transaction
            with checkpointer.conn.pipeline() as pipe:
                checkpointer.pipe = pipe
                yield checkpointer
    finally:
        # drop unique db
        with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
            conn.execute(f"DROP DATABASE {database}")


@contextmanager
def _checkpointer_postgres_pool():
    database = f"test_{uuid4().hex[:16]}"
    # create unique db
    with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
        conn.execute(f"CREATE DATABASE {database}")
    try:
        # yield checkpointer
        with ConnectionPool(
            DEFAULT_POSTGRES_URI + database, max_size=10, kwargs={"autocommit": True}
        ) as pool:
            checkpointer = PostgresSaver(pool)
            checkpointer.setup()
            yield checkpointer
    finally:
        # drop unique db
        with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
            conn.execute(f"DROP DATABASE {database}")


@asynccontextmanager
async def _checkpointer_sqlite_aio():
    async with AsyncSqliteSaver.from_conn_string(":memory:") as checkpointer:
        yield checkpointer


@asynccontextmanager
async def _checkpointer_postgres_aio():
    database = f"test_{uuid4().hex[:16]}"
    # create unique db
    async with await AsyncConnection.connect(
        DEFAULT_POSTGRES_URI, autocommit=True
    ) as conn:
        await conn.execute(f"CREATE DATABASE {database}")
    try:
        # yield checkpointer
        async with AsyncPostgresSaver.from_conn_string(
            DEFAULT_POSTGRES_URI + database
        ) as checkpointer:
            await checkpointer.setup()
            yield checkpointer
    finally:
        # drop unique db
        async with await AsyncConnection.connect(
            DEFAULT_POSTGRES_URI, autocommit=True
        ) as conn:
            await conn.execute(f"DROP DATABASE {database}")


@asynccontextmanager
async def _checkpointer_postgres_aio_pipe():
    database = f"test_{uuid4().hex[:16]}"
    # create unique db
    async with await AsyncConnection.connect(
        DEFAULT_POSTGRES_URI, autocommit=True
    ) as conn:
        await conn.execute(f"CREATE DATABASE {database}")
    try:
        # yield checkpointer
        async with AsyncPostgresSaver.from_conn_string(
            DEFAULT_POSTGRES_URI + database
        ) as checkpointer:
            await checkpointer.setup()
            # setup can't run inside pipeline because of implicit transaction
            async with checkpointer.conn.pipeline() as pipe:
                checkpointer.pipe = pipe
                yield checkpointer
    finally:
        # drop unique db
        async with await AsyncConnection.connect(
            DEFAULT_POSTGRES_URI, autocommit=True
        ) as conn:
            await conn.execute(f"DROP DATABASE {database}")


@asynccontextmanager
async def _checkpointer_postgres_aio_pool():
    database = f"test_{uuid4().hex[:16]}"
    # create unique db
    async with await AsyncConnection.connect(
        DEFAULT_POSTGRES_URI, autocommit=True
    ) as conn:
        await conn.execute(f"CREATE DATABASE {database}")
    try:
        # yield checkpointer
        async with AsyncConnectionPool(
            DEFAULT_POSTGRES_URI + database, max_size=10, kwargs={"autocommit": True}
        ) as pool:
            checkpointer = AsyncPostgresSaver(pool)
            await checkpointer.setup()
            yield checkpointer
    finally:
        # drop unique db
        async with await AsyncConnection.connect(
            DEFAULT_POSTGRES_URI, autocommit=True
        ) as conn:
            await conn.execute(f"DROP DATABASE {database}")


__all__ = [
    "_checkpointer_memory",
    "_checkpointer_memory_migrate_sends",
    "_checkpointer_sqlite",
    "_checkpointer_sqlite_aes",
    "_checkpointer_postgres",
    "_checkpointer_postgres_pipe",
    "_checkpointer_postgres_pool",
    "_checkpointer_sqlite_aio",
    "_checkpointer_postgres_aio",
    "_checkpointer_postgres_aio_pipe",
    "_checkpointer_postgres_aio_pool",
]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/conftest_store.py
```py
from contextlib import asynccontextmanager, contextmanager
from uuid import uuid4

from langgraph.store.memory import InMemoryStore
from langgraph.store.postgres import AsyncPostgresStore, PostgresStore
from psycopg import AsyncConnection, Connection

DEFAULT_POSTGRES_URI = "postgres://postgres:postgres@localhost:5442/"


@contextmanager
def _store_memory():
    store = InMemoryStore()
    yield store


@contextmanager
def _store_postgres():
    database = f"test_{uuid4().hex[:16]}"
    # create unique db
    with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
        conn.execute(f"CREATE DATABASE {database}")
    try:
        # yield store
        with PostgresStore.from_conn_string(DEFAULT_POSTGRES_URI + database) as store:
            store.setup()
            yield store
    finally:
        # drop unique db
        with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
            conn.execute(f"DROP DATABASE {database}")


@contextmanager
def _store_postgres_pipe():
    database = f"test_{uuid4().hex[:16]}"
    # create unique db
    with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
        conn.execute(f"CREATE DATABASE {database}")
    try:
        # yield store
        with PostgresStore.from_conn_string(DEFAULT_POSTGRES_URI + database) as store:
            store.setup()  # Run in its own transaction
        with PostgresStore.from_conn_string(
            DEFAULT_POSTGRES_URI + database, pipeline=True
        ) as store:
            yield store
    finally:
        # drop unique db
        with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
            conn.execute(f"DROP DATABASE {database}")


@contextmanager
def _store_postgres_pool():
    database = f"test_{uuid4().hex[:16]}"
    # create unique db
    with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
        conn.execute(f"CREATE DATABASE {database}")
    try:
        # yield store
        with PostgresStore.from_conn_string(
            DEFAULT_POSTGRES_URI + database, pool_config={"max_size": 10}
        ) as store:
            store.setup()
            yield store
    finally:
        # drop unique db
        with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
            conn.execute(f"DROP DATABASE {database}")


@asynccontextmanager
async def _store_postgres_aio():
    database = f"test_{uuid4().hex[:16]}"
    async with await AsyncConnection.connect(
        DEFAULT_POSTGRES_URI, autocommit=True
    ) as conn:
        await conn.execute(f"CREATE DATABASE {database}")
    try:
        async with AsyncPostgresStore.from_conn_string(
            DEFAULT_POSTGRES_URI + database
        ) as store:
            await store.setup()
            yield store
    finally:
        async with await AsyncConnection.connect(
            DEFAULT_POSTGRES_URI, autocommit=True
        ) as conn:
            await conn.execute(f"DROP DATABASE {database}")


@asynccontextmanager
async def _store_postgres_aio_pipe():
    database = f"test_{uuid4().hex[:16]}"
    async with await AsyncConnection.connect(
        DEFAULT_POSTGRES_URI, autocommit=True
    ) as conn:
        await conn.execute(f"CREATE DATABASE {database}")
    try:
        async with AsyncPostgresStore.from_conn_string(
            DEFAULT_POSTGRES_URI + database
        ) as store:
            await store.setup()  # Run in its own transaction
        async with AsyncPostgresStore.from_conn_string(
            DEFAULT_POSTGRES_URI + database, pipeline=True
        ) as store:
            yield store
    finally:
        async with await AsyncConnection.connect(
            DEFAULT_POSTGRES_URI, autocommit=True
        ) as conn:
            await conn.execute(f"DROP DATABASE {database}")


@asynccontextmanager
async def _store_postgres_aio_pool():
    database = f"test_{uuid4().hex[:16]}"
    async with await AsyncConnection.connect(
        DEFAULT_POSTGRES_URI, autocommit=True
    ) as conn:
        await conn.execute(f"CREATE DATABASE {database}")
    try:
        async with AsyncPostgresStore.from_conn_string(
            DEFAULT_POSTGRES_URI + database,
            pool_config={"max_size": 10},
        ) as store:
            await store.setup()
            yield store
    finally:
        async with await AsyncConnection.connect(
            DEFAULT_POSTGRES_URI, autocommit=True
        ) as conn:
            await conn.execute(f"DROP DATABASE {database}")


__all__ = [
    "_store_memory",
    "_store_postgres",
    "_store_postgres_pipe",
    "_store_postgres_pool",
    "_store_postgres_aio",
    "_store_postgres_aio_pipe",
    "_store_postgres_aio_pool",
]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/fake_chat.py
```py
import re
from collections.abc import AsyncIterator, Iterator
from typing import Any, cast

from langchain_core.callbacks import (
    AsyncCallbackManagerForLLMRun,
    CallbackManagerForLLMRun,
)
from langchain_core.language_models.fake_chat_models import GenericFakeChatModel
from langchain_core.messages import AIMessage, AIMessageChunk, BaseMessage
from langchain_core.outputs import ChatGeneration, ChatGenerationChunk, ChatResult


class FakeChatModel(GenericFakeChatModel):
    messages: list[BaseMessage]

    i: int = 0

    def bind_tools(self, functions: list):
        return self

    def _generate(
        self,
        messages: list[BaseMessage],
        stop: list[str] | None = None,
        run_manager: CallbackManagerForLLMRun | None = None,
        **kwargs: Any,
    ) -> ChatResult:
        """Top Level call"""
        if self.i >= len(self.messages):
            self.i = 0
        message = self.messages[self.i]
        self.i += 1
        if isinstance(message, str):
            message_ = AIMessage(content=message)
        else:
            if hasattr(message, "model_copy"):
                message_ = message.model_copy()
            else:
                message_ = message.copy()
        generation = ChatGeneration(message=message_)
        return ChatResult(generations=[generation])

    def _stream(
        self,
        messages: list[BaseMessage],
        stop: list[str] | None = None,
        run_manager: CallbackManagerForLLMRun | None = None,
        **kwargs: Any,
    ) -> Iterator[ChatGenerationChunk]:
        """Stream the output of the model."""
        chat_result = self._generate(
            messages, stop=stop, run_manager=run_manager, **kwargs
        )
        if not isinstance(chat_result, ChatResult):
            raise ValueError(
                f"Expected generate to return a ChatResult, "
                f"but got {type(chat_result)} instead."
            )

        message = chat_result.generations[0].message

        if not isinstance(message, AIMessage):
            raise ValueError(
                f"Expected invoke to return an AIMessage, "
                f"but got {type(message)} instead."
            )

        content = message.content

        if content:
            # Use a regular expression to split on whitespace with a capture group
            # so that we can preserve the whitespace in the output.
            assert isinstance(content, str)
            content_chunks = cast(list[str], re.split(r"(\s)", content))

            for i, token in enumerate(content_chunks):
                if i == len(content_chunks) - 1:
                    chunk = ChatGenerationChunk(
                        message=AIMessageChunk(
                            content=token, id=message.id, chunk_position="last"
                        )
                    )
                else:
                    chunk = ChatGenerationChunk(
                        message=AIMessageChunk(content=token, id=message.id)
                    )
                if run_manager:
                    run_manager.on_llm_new_token(token, chunk=chunk)
                yield chunk
        else:
            args = message.__dict__
            args.pop("type")
            chunk = ChatGenerationChunk(
                message=AIMessageChunk(**args, chunk_position="last")
            )
            if run_manager:
                run_manager.on_llm_new_token("", chunk=chunk)
            yield chunk

    async def _astream(
        self,
        messages: list[BaseMessage],
        stop: list[str] | None = None,
        run_manager: AsyncCallbackManagerForLLMRun | None = None,
        **kwargs: Any,
    ) -> AsyncIterator[ChatGenerationChunk]:
        """Stream the output of the model."""
        chat_result = self._generate(
            messages, stop=stop, run_manager=run_manager, **kwargs
        )
        if not isinstance(chat_result, ChatResult):
            raise ValueError(
                f"Expected generate to return a ChatResult, "
                f"but got {type(chat_result)} instead."
            )

        message = chat_result.generations[0].message

        if not isinstance(message, AIMessage):
            raise ValueError(
                f"Expected invoke to return an AIMessage, "
                f"but got {type(message)} instead."
            )

        content = message.content

        if content:
            # Use a regular expression to split on whitespace with a capture group
            # so that we can preserve the whitespace in the output.
            assert isinstance(content, str)
            content_chunks = cast(list[str], re.split(r"(\s)", content))

            for i, token in enumerate(content_chunks):
                if i == len(content_chunks) - 1:
                    chunk = ChatGenerationChunk(
                        message=AIMessageChunk(
                            content=token, id=message.id, chunk_position="last"
                        )
                    )
                else:
                    chunk = ChatGenerationChunk(
                        message=AIMessageChunk(content=token, id=message.id)
                    )

                if run_manager:
                    run_manager.on_llm_new_token(token, chunk=chunk)
                yield chunk
        else:
            args = message.__dict__
            args.pop("type")
            chunk = ChatGenerationChunk(
                message=AIMessageChunk(**args, chunk_position="last")
            )
            if run_manager:
                await run_manager.on_llm_new_token("", chunk=chunk)
            yield chunk

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/fake_tracer.py
```py
from typing import Any
from uuid import UUID

from langchain_core.messages.base import BaseMessage
from langchain_core.outputs.chat_generation import ChatGeneration
from langchain_core.outputs.llm_result import LLMResult
from langchain_core.tracers import BaseTracer, Run


class FakeTracer(BaseTracer):
    """Fake tracer that records LangChain execution.
    It replaces run ids with deterministic UUIDs for snapshotting."""

    def __init__(self) -> None:
        """Initialize the tracer."""
        super().__init__()
        self.runs: list[Run] = []
        self.uuids_map: dict[UUID, UUID] = {}
        self.uuids_generator = (
            UUID(f"00000000-0000-4000-8000-{i:012}", version=4) for i in range(10000)
        )

    def _replace_uuid(self, uuid: UUID) -> UUID:
        if uuid not in self.uuids_map:
            self.uuids_map[uuid] = next(self.uuids_generator)
        return self.uuids_map[uuid]

    def _replace_message_id(self, maybe_message: Any) -> Any:
        if isinstance(maybe_message, BaseMessage):
            maybe_message.id = str(next(self.uuids_generator))
        if isinstance(maybe_message, ChatGeneration):
            maybe_message.message.id = str(next(self.uuids_generator))
        if isinstance(maybe_message, LLMResult):
            for i, gen_list in enumerate(maybe_message.generations):
                for j, gen in enumerate(gen_list):
                    maybe_message.generations[i][j] = self._replace_message_id(gen)
        if isinstance(maybe_message, dict):
            for k, v in maybe_message.items():
                maybe_message[k] = self._replace_message_id(v)
        if isinstance(maybe_message, list):
            for i, v in enumerate(maybe_message):
                maybe_message[i] = self._replace_message_id(v)

        return maybe_message

    def _copy_run(self, run: Run) -> Run:
        if run.dotted_order:
            levels = run.dotted_order.split(".")
            processed_levels = []
            for level in levels:
                timestamp, run_id = level.split("Z")
                new_run_id = self._replace_uuid(UUID(run_id))
                processed_level = f"{timestamp}Z{new_run_id}"
                processed_levels.append(processed_level)
            new_dotted_order = ".".join(processed_levels)
        else:
            new_dotted_order = None
        return run.copy(
            update={
                "id": self._replace_uuid(run.id),
                "parent_run_id": (
                    self.uuids_map[run.parent_run_id] if run.parent_run_id else None
                ),
                "child_runs": [self._copy_run(child) for child in run.child_runs],
                "trace_id": self._replace_uuid(run.trace_id) if run.trace_id else None,
                "dotted_order": new_dotted_order,
                "inputs": self._replace_message_id(run.inputs),
                "outputs": self._replace_message_id(run.outputs),
            }
        )

    def _persist_run(self, run: Run) -> None:
        """Persist a run."""

        self.runs.append(self._copy_run(run))

    def flattened_runs(self) -> list[Run]:
        q = [] + self.runs
        result = []
        while q:
            parent = q.pop()
            result.append(parent)
            if parent.child_runs:
                q.extend(parent.child_runs)
        return result

    @property
    def run_ids(self) -> list[UUID | None]:
        runs = self.flattened_runs()
        uuids_map = {v: k for k, v in self.uuids_map.items()}
        return [uuids_map.get(r.id) for r in runs]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/memory_assert.py
```py
import os
import tempfile
from collections import defaultdict
from functools import partial
from typing import Any

from langchain_core.runnables import RunnableConfig
from langgraph.checkpoint.base import (
    BaseCheckpointSaver,
    ChannelVersions,
    Checkpoint,
    CheckpointMetadata,
    CheckpointTuple,
    SerializerProtocol,
)
from langgraph.checkpoint.memory import InMemorySaver, PersistentDict

from langgraph.constants import TASKS


class NoopSerializer(SerializerProtocol):
    def loads_typed(self, data: tuple[str, bytes]) -> Any:
        return data[1]

    def dumps_typed(self, obj: Any) -> tuple[str, bytes]:
        return "type", obj


class MemorySaverNeedsPendingSendsMigration(BaseCheckpointSaver):
    def __init__(self) -> None:
        self.saver = InMemorySaver()

    def __getattribute__(self, name):
        if name in ("saver", "__class__", "get_tuple"):
            return object.__getattribute__(self, name)
        return getattr(self.saver, name)

    def get_tuple(self, config):
        if tup := self.saver.get_tuple(config):
            if tup.checkpoint["v"] == 4 and tup.checkpoint["channel_values"].get(TASKS):
                tup.checkpoint["v"] = 3
                tup.checkpoint["pending_sends"] = tup.checkpoint["channel_values"].pop(
                    TASKS
                )
                tup.checkpoint["channel_versions"].pop(TASKS)
                for seen in tup.checkpoint["versions_seen"].values():
                    seen.pop(TASKS, None)
        return tup


class MemorySaverAssertImmutable(InMemorySaver):
    storage_for_copies: defaultdict[str, dict[str, dict[str, Checkpoint]]]

    def __init__(
        self,
        *,
        serde: SerializerProtocol | None = None,
        put_sleep: float | None = None,
    ) -> None:
        _, filename = tempfile.mkstemp()
        super().__init__(
            serde=serde, factory=partial(PersistentDict, filename=filename)
        )
        self.storage_for_copies = defaultdict(lambda: defaultdict(dict))
        self.put_sleep = put_sleep
        self.stack.callback(os.remove, filename)

    def put(
        self,
        config: dict,
        checkpoint: Checkpoint,
        metadata: CheckpointMetadata,
        new_versions: ChannelVersions,
    ) -> None:
        if self.put_sleep:
            import time

            time.sleep(self.put_sleep)
        # assert checkpoint hasn't been modified since last written
        thread_id = config["configurable"]["thread_id"]
        checkpoint_ns = config["configurable"]["checkpoint_ns"]
        if saved := super().get(config):
            assert (
                self.serde.loads_typed(
                    self.storage_for_copies[thread_id][checkpoint_ns][saved["id"]]
                )
                == saved
            ), config["configurable"]["checkpoint_ns"]
        self.storage_for_copies[thread_id][checkpoint_ns][checkpoint["id"]] = (
            self.serde.dumps_typed(checkpoint)
        )
        # call super to write checkpoint
        return super().put(config, checkpoint, metadata, new_versions)


class MemorySaverNoPending(InMemorySaver):
    def get_tuple(self, config: RunnableConfig) -> CheckpointTuple | None:
        result = super().get_tuple(config)
        if result:
            return CheckpointTuple(result.config, result.checkpoint, result.metadata)
        return result

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/messages.py
```py
"""Redefined messages as a work-around for pydantic issue with AnyStr.

The code below creates version of pydantic models
that will work in unit tests with AnyStr as id field
Please note that the `id` field is assigned AFTER the model is created
to workaround an issue with pydantic ignoring the __eq__ method on
subclassed strings.
"""

from typing import Any

from langchain_core.documents import Document
from langchain_core.messages import AIMessage, AIMessageChunk, HumanMessage, ToolMessage

from tests.any_str import AnyStr


def _AnyIdDocument(**kwargs: Any) -> Document:
    """Create a document with an id field."""
    message = Document(**kwargs)
    message.id = AnyStr()
    return message


def _AnyIdAIMessage(**kwargs: Any) -> AIMessage:
    """Create ai message with an any id field."""
    message = AIMessage(**kwargs)
    message.id = AnyStr()
    return message


def _AnyIdAIMessageChunk(**kwargs: Any) -> AIMessageChunk:
    """Create ai message with an any id field."""
    message = AIMessageChunk(**kwargs)
    message.id = AnyStr()
    return message


def _AnyIdHumanMessage(**kwargs: Any) -> HumanMessage:
    """Create a human message with an any id field."""
    message = HumanMessage(**kwargs)
    message.id = AnyStr()
    return message


def _AnyIdToolMessage(**kwargs: Any) -> ToolMessage:
    """Create a tool message with an any id field."""
    message = ToolMessage(**kwargs)
    message.id = AnyStr()
    return message

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/test_algo.py
```py
from langgraph._internal._constants import PULL, PUSH
from langgraph.pregel._algo import prepare_next_tasks, task_path_str
from langgraph.pregel._checkpoint import channels_from_checkpoint, empty_checkpoint


def test_prepare_next_tasks() -> None:
    config = {}
    processes = {}
    checkpoint = empty_checkpoint()
    channels, managed = channels_from_checkpoint({}, checkpoint)

    assert (
        prepare_next_tasks(
            checkpoint,
            {},
            processes,
            channels,
            managed,
            config,
            0,
            -1,
            for_execution=False,
        )
        == {}
    )
    assert (
        prepare_next_tasks(
            checkpoint,
            {},
            processes,
            channels,
            managed,
            config,
            0,
            -1,
            for_execution=True,
            checkpointer=None,
            store=None,
            manager=None,
        )
        == {}
    )

    # TODO: add more tests


def test_tuple_str() -> None:
    push_path_a = (PUSH, 2)
    pull_path_a = (PULL, "abc")
    push_path_b = (PUSH, push_path_a, 1)
    push_path_c = (PUSH, push_path_b, 3)

    assert task_path_str(push_path_a) == f"~{PUSH}, 0000000002"
    assert task_path_str(push_path_b) == f"~{PUSH}, ~{PUSH}, 0000000002, 0000000001"
    assert (
        task_path_str(push_path_c)
        == f"~{PUSH}, ~{PUSH}, ~{PUSH}, 0000000002, 0000000001, 0000000003"
    )
    assert task_path_str(pull_path_a) == f"~{PULL}, abc"

    path_list = [push_path_b, push_path_a, pull_path_a, push_path_c]
    assert sorted(map(task_path_str, path_list)) == [
        f"~{PULL}, abc",
        f"~{PUSH}, 0000000002",
        f"~{PUSH}, ~{PUSH}, 0000000002, 0000000001",
        f"~{PUSH}, ~{PUSH}, ~{PUSH}, 0000000002, 0000000001, 0000000003",
    ]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/test_channels.py
```py
import operator
from collections.abc import Sequence

import pytest

from langgraph._internal._typing import MISSING
from langgraph.channels.binop import BinaryOperatorAggregate
from langgraph.channels.last_value import LastValue
from langgraph.channels.topic import Topic
from langgraph.channels.untracked_value import UntrackedValue
from langgraph.errors import EmptyChannelError, InvalidUpdateError

pytestmark = pytest.mark.anyio


def test_last_value() -> None:
    channel = LastValue(int).from_checkpoint(MISSING)
    assert channel.ValueType is int
    assert channel.UpdateType is int

    with pytest.raises(EmptyChannelError):
        channel.get()
    with pytest.raises(InvalidUpdateError):
        channel.update([5, 6])

    channel.update([3])
    assert channel.get() == 3
    channel.update([4])
    assert channel.get() == 4
    checkpoint = channel.checkpoint()
    channel = LastValue(int).from_checkpoint(checkpoint)
    assert channel.get() == 4


def test_topic() -> None:
    channel = Topic(str).from_checkpoint(MISSING)
    assert channel.ValueType == Sequence[str]
    assert channel.UpdateType == str | list[str]

    assert channel.update(["a", "b"])
    assert channel.get() == ["a", "b"]
    assert channel.update([["c", "d"], "d"])
    assert channel.get() == ["c", "d", "d"]
    assert channel.update([])
    with pytest.raises(EmptyChannelError):
        channel.get()
    assert not channel.update([]), "channel already empty"
    assert channel.update(["e"])
    assert channel.get() == ["e"]
    checkpoint = channel.checkpoint()
    channel = Topic(str).from_checkpoint(checkpoint)
    assert channel.get() == ["e"]
    channel_copy = Topic(str).from_checkpoint(checkpoint)
    channel_copy.update(["f"])
    assert channel_copy.get() == ["f"]
    assert channel.get() == ["e"]


def test_topic_accumulate() -> None:
    channel = Topic(str, accumulate=True).from_checkpoint(MISSING)
    assert channel.ValueType == Sequence[str]
    assert channel.UpdateType == str | list[str]

    assert channel.update(["a", "b"])
    assert channel.get() == ["a", "b"]
    assert channel.update(["b", ["c", "d"], "d"])
    assert channel.get() == ["a", "b", "b", "c", "d", "d"]
    assert not channel.update([])
    assert channel.get() == ["a", "b", "b", "c", "d", "d"]
    checkpoint = channel.checkpoint()
    channel = Topic(str, accumulate=True).from_checkpoint(checkpoint)
    assert channel.get() == ["a", "b", "b", "c", "d", "d"]
    assert channel.update(["e"])
    assert channel.get() == ["a", "b", "b", "c", "d", "d", "e"]


def test_binop() -> None:
    channel = BinaryOperatorAggregate(int, operator.add).from_checkpoint(MISSING)
    assert channel.ValueType is int
    assert channel.UpdateType is int

    assert channel.get() == 0

    channel.update([1, 2, 3])
    assert channel.get() == 6
    channel.update([4])
    assert channel.get() == 10
    checkpoint = channel.checkpoint()
    channel = BinaryOperatorAggregate(int, operator.add).from_checkpoint(checkpoint)
    assert channel.get() == 10


def test_untracked_value() -> None:
    channel = UntrackedValue(dict).from_checkpoint(MISSING)
    assert channel.ValueType is dict
    assert channel.UpdateType is dict

    # UntrackedValue should start empty
    with pytest.raises(EmptyChannelError):
        channel.get()

    # Should be able to update with a value
    test_data = {"session": "test", "temp": "dir"}
    channel.update([test_data])
    assert channel.get() == test_data

    # Update with new value
    new_data = {"session": "updated", "temp": "newdir"}
    channel.update([new_data])
    assert channel.get() == new_data

    # On checkpoint, UntrackedValue should return MISSING
    checkpoint = channel.checkpoint()
    assert checkpoint is MISSING

    # Creating from checkpoint with MISSING should start empty
    new_channel = UntrackedValue(dict).from_checkpoint(checkpoint)
    with pytest.raises(EmptyChannelError):
        new_channel.get()

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/test_checkpoint_migration.py
```py
import operator
import sys
import time
from collections import defaultdict
from typing import Annotated, Literal

import pytest
from langgraph.checkpoint.base import BaseCheckpointSaver, CheckpointTuple
from typing_extensions import TypedDict

from langgraph._internal._config import patch_configurable
from langgraph.graph.state import StateGraph
from langgraph.pregel._checkpoint import copy_checkpoint
from langgraph.types import Command, Interrupt, PregelTask, StateSnapshot, interrupt
from tests.any_int import AnyInt
from tests.any_str import AnyDict, AnyObject, AnyStr

pytestmark = pytest.mark.anyio

NEEDS_CONTEXTVARS = pytest.mark.skipif(
    sys.version_info < (3, 11),
    reason="Python 3.11+ is required for async contextvars support",
)


def get_expected_history(*, exc_task_results: int = 0) -> list[StateSnapshot]:
    return [
        StateSnapshot(
            values={
                "query": "analyzed: query: what is weather in sf",
                "answer": "doc1,doc2,doc3,doc4",
                "docs": ["doc1", "doc2", "doc3", "doc4"],
            },
            next=(),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "source": "loop",
                "step": 4,
                "parents": {},
            },
            created_at=AnyStr(),
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            tasks=(),
            interrupts=(),
        ),
        StateSnapshot(
            values={
                "query": "analyzed: query: what is weather in sf",
                "docs": ["doc1", "doc2", "doc3", "doc4"],
            },
            next=("qa",),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "source": "loop",
                "step": 3,
                "parents": {},
            },
            created_at=AnyStr(),
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            tasks=(
                PregelTask(
                    id=AnyStr(),
                    name="qa",
                    path=("__pregel_pull", "qa"),
                    error=None,
                    interrupts=()
                    if exc_task_results
                    else (
                        Interrupt(
                            value="",
                            id=AnyStr(),
                        ),
                    ),
                    state=None,
                    result=None
                    if exc_task_results
                    else {"answer": "doc1,doc2,doc3,doc4"},
                ),
            ),
            interrupts=()
            if exc_task_results
            else (
                Interrupt(
                    value="",
                    id=AnyStr(),
                ),
            ),
        ),
        StateSnapshot(
            values={
                "query": "analyzed: query: what is weather in sf",
                "docs": ["doc3", "doc4"],
            },
            next=("retriever_one",),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "source": "loop",
                "step": 2,
                "parents": {},
            },
            created_at=AnyStr(),
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            tasks=(
                PregelTask(
                    id=AnyStr(),
                    name="retriever_one",
                    path=("__pregel_pull", "retriever_one"),
                    error=None,
                    interrupts=(),
                    state=None,
                    result=None if exc_task_results else {"docs": ["doc1", "doc2"]},
                ),
            ),
            interrupts=(),
        ),
        StateSnapshot(
            values={"query": "query: what is weather in sf", "docs": []},
            next=("analyzer_one", "retriever_two"),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "source": "loop",
                "step": 1,
                "parents": {},
            },
            created_at=AnyStr(),
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            tasks=(
                PregelTask(
                    id=AnyStr(),
                    name="analyzer_one",
                    path=("__pregel_pull", "analyzer_one"),
                    error=None,
                    interrupts=(),
                    state=None,
                    result=None
                    if exc_task_results
                    else {"query": "analyzed: query: what is weather in sf"},
                ),
                PregelTask(
                    id=AnyStr(),
                    name="retriever_two",
                    path=("__pregel_pull", "retriever_two"),
                    error=None,
                    interrupts=(),
                    state=None,
                    result=None
                    if exc_task_results >= 2
                    else {"docs": ["doc3", "doc4"]},
                ),
            ),
            interrupts=(),
        ),
        StateSnapshot(
            values={"query": "what is weather in sf", "docs": []},
            next=("rewrite_query",),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "source": "loop",
                "step": 0,
                "parents": {},
            },
            created_at=AnyStr(),
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            tasks=(
                PregelTask(
                    id=AnyStr(),
                    name="rewrite_query",
                    path=("__pregel_pull", "rewrite_query"),
                    error=None,
                    interrupts=(),
                    state=None,
                    result=None
                    if exc_task_results
                    else {"query": "query: what is weather in sf"},
                ),
            ),
            interrupts=(),
        ),
        StateSnapshot(
            values={"docs": []},
            next=("__start__",),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "source": "input",
                "step": -1,
                "parents": {},
            },
            created_at=AnyStr(),
            parent_config=None,
            tasks=(
                PregelTask(
                    id=AnyStr(),
                    name="__start__",
                    path=("__pregel_pull", "__start__"),
                    error=None,
                    interrupts=(),
                    state=None,
                    result={"query": "what is weather in sf"},
                ),
            ),
            interrupts=(),
        ),
    ]


SAVED_CHECKPOINTS = {
    "3": [
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fd5f-2149-6faa-8004-9d848038f10a",
                }
            },
            checkpoint={
                "v": 2,
                "ts": "2025-04-02T15:20:01.237381+00:00",
                "id": "1f00fd5f-2149-6faa-8004-9d848038f10a",
                "channel_versions": {
                    "__start__": "00000000000000000000000000000002.0.6697367414225304",
                    "query": "00000000000000000000000000000004.0.18727156933289513",
                    "branch:to:rewrite_query": "00000000000000000000000000000003.0.14126716107927562",
                    "branch:to:analyzer_one": "00000000000000000000000000000004.0.15766851053750708",
                    "branch:to:retriever_two": "00000000000000000000000000000004.0.04821745244115927",
                    "branch:to:retriever_one": "00000000000000000000000000000005.0.7710812646219019",
                    "docs": "00000000000000000000000000000005.0.7916507770116351",
                    "branch:to:qa": "00000000000000000000000000000006.0.6375257096095945",
                    "answer": "00000000000000000000000000000006.0.9100669543952636",
                },
                "versions_seen": {
                    "__input__": {},
                    "__start__": {
                        "__start__": "00000000000000000000000000000001.0.7234984738744598"
                    },
                    "rewrite_query": {
                        "branch:to:rewrite_query": "00000000000000000000000000000002.0.05597832024496252"
                    },
                    "analyzer_one": {
                        "branch:to:analyzer_one": "00000000000000000000000000000003.0.7165779439892241"
                    },
                    "retriever_two": {
                        "branch:to:retriever_two": "00000000000000000000000000000003.0.7762711252277583"
                    },
                    "retriever_one": {
                        "branch:to:retriever_one": "00000000000000000000000000000004.0.5907938097782264"
                    },
                    "__interrupt__": {
                        "query": "00000000000000000000000000000004.0.18727156933289513",
                        "docs": "00000000000000000000000000000005.0.7916507770116351",
                        "__start__": "00000000000000000000000000000002.0.6697367414225304",
                        "branch:to:rewrite_query": "00000000000000000000000000000003.0.14126716107927562",
                        "branch:to:analyzer_one": "00000000000000000000000000000004.0.15766851053750708",
                        "branch:to:retriever_one": "00000000000000000000000000000005.0.7710812646219019",
                        "branch:to:retriever_two": "00000000000000000000000000000004.0.04821745244115927",
                        "branch:to:qa": "00000000000000000000000000000005.0.5602643794940962",
                    },
                    "qa": {
                        "branch:to:qa": "00000000000000000000000000000005.0.5602643794940962"
                    },
                },
                "channel_values": {
                    "query": "analyzed: query: what is weather in sf",
                    "docs": ["doc1", "doc2", "doc3", "doc4"],
                    "answer": "doc1,doc2,doc3,doc4",
                },
                "updated_channels": None,
            },
            metadata={
                "source": "loop",
                "step": 4,
                "parents": {},
            },
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fd5f-2140-6fd6-8003-2051ce36b79c",
                }
            },
            pending_writes=[],
        ),
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fd5f-2140-6fd6-8003-2051ce36b79c",
                }
            },
            checkpoint={
                "v": 2,
                "ts": "2025-04-02T15:20:01.233695+00:00",
                "id": "1f00fd5f-2140-6fd6-8003-2051ce36b79c",
                "channel_versions": {
                    "__start__": "00000000000000000000000000000002.0.6697367414225304",
                    "query": "00000000000000000000000000000004.0.18727156933289513",
                    "branch:to:rewrite_query": "00000000000000000000000000000003.0.14126716107927562",
                    "branch:to:analyzer_one": "00000000000000000000000000000004.0.15766851053750708",
                    "branch:to:retriever_two": "00000000000000000000000000000004.0.04821745244115927",
                    "branch:to:retriever_one": "00000000000000000000000000000005.0.7710812646219019",
                    "docs": "00000000000000000000000000000005.0.7916507770116351",
                    "branch:to:qa": "00000000000000000000000000000005.0.5602643794940962",
                },
                "versions_seen": {
                    "__input__": {},
                    "__start__": {
                        "__start__": "00000000000000000000000000000001.0.7234984738744598"
                    },
                    "rewrite_query": {
                        "branch:to:rewrite_query": "00000000000000000000000000000002.0.05597832024496252"
                    },
                    "analyzer_one": {
                        "branch:to:analyzer_one": "00000000000000000000000000000003.0.7165779439892241"
                    },
                    "retriever_two": {
                        "branch:to:retriever_two": "00000000000000000000000000000003.0.7762711252277583"
                    },
                    "retriever_one": {
                        "branch:to:retriever_one": "00000000000000000000000000000004.0.5907938097782264"
                    },
                },
                "channel_values": {
                    "query": "analyzed: query: what is weather in sf",
                    "docs": ["doc1", "doc2", "doc3", "doc4"],
                    "branch:to:qa": None,
                },
                "updated_channels": None,
            },
            metadata={
                "source": "loop",
                "step": 3,
                "parents": {},
            },
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fd5f-213c-6940-8002-a28f475a6478",
                }
            },
            pending_writes=[
                (
                    "2430f303-da9f-2e3e-738c-2e8ea28e8973",
                    "__interrupt__",
                    [
                        Interrupt(
                            value="",
                            resumable=True,  # type: ignore[arg-type]
                            ns=["qa:2430f303-da9f-2e3e-738c-2e8ea28e8973"],  # type: ignore[arg-type]
                        )
                    ],
                ),
                ("00000000-0000-0000-0000-000000000000", "__resume__", ""),
                ("2430f303-da9f-2e3e-738c-2e8ea28e8973", "__resume__", [""]),
                (
                    "2430f303-da9f-2e3e-738c-2e8ea28e8973",
                    "answer",
                    "doc1,doc2,doc3,doc4",
                ),
            ],
        ),
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fd5f-213c-6940-8002-a28f475a6478",
                }
            },
            checkpoint={
                "v": 2,
                "ts": "2025-04-02T15:20:01.231890+00:00",
                "id": "1f00fd5f-213c-6940-8002-a28f475a6478",
                "channel_versions": {
                    "__start__": "00000000000000000000000000000002.0.6697367414225304",
                    "query": "00000000000000000000000000000004.0.18727156933289513",
                    "branch:to:rewrite_query": "00000000000000000000000000000003.0.14126716107927562",
                    "branch:to:analyzer_one": "00000000000000000000000000000004.0.15766851053750708",
                    "branch:to:retriever_two": "00000000000000000000000000000004.0.04821745244115927",
                    "branch:to:retriever_one": "00000000000000000000000000000004.0.5907938097782264",
                    "docs": "00000000000000000000000000000004.0.972701399851098",
                },
                "versions_seen": {
                    "__input__": {},
                    "__start__": {
                        "__start__": "00000000000000000000000000000001.0.7234984738744598"
                    },
                    "rewrite_query": {
                        "branch:to:rewrite_query": "00000000000000000000000000000002.0.05597832024496252"
                    },
                    "analyzer_one": {
                        "branch:to:analyzer_one": "00000000000000000000000000000003.0.7165779439892241"
                    },
                    "retriever_two": {
                        "branch:to:retriever_two": "00000000000000000000000000000003.0.7762711252277583"
                    },
                },
                "channel_values": {
                    "query": "analyzed: query: what is weather in sf",
                    "branch:to:retriever_one": None,
                    "docs": ["doc3", "doc4"],
                },
                "updated_channels": None,
            },
            metadata={
                "source": "loop",
                "step": 2,
                "parents": {},
            },
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fd5f-2039-6354-8001-2c508c8dffd9",
                }
            },
            pending_writes=[
                ("a5602426-85f2-1fe4-c9e4-bd0127e8e53e", "docs", ["doc1", "doc2"]),
                ("a5602426-85f2-1fe4-c9e4-bd0127e8e53e", "branch:to:qa", None),
            ],
        ),
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fd5f-2039-6354-8001-2c508c8dffd9",
                }
            },
            checkpoint={
                "v": 2,
                "ts": "2025-04-02T15:20:01.125661+00:00",
                "id": "1f00fd5f-2039-6354-8001-2c508c8dffd9",
                "channel_versions": {
                    "__start__": "00000000000000000000000000000002.0.6697367414225304",
                    "query": "00000000000000000000000000000003.0.04057405566428263",
                    "branch:to:rewrite_query": "00000000000000000000000000000003.0.14126716107927562",
                    "branch:to:analyzer_one": "00000000000000000000000000000003.0.7165779439892241",
                    "branch:to:retriever_two": "00000000000000000000000000000003.0.7762711252277583",
                },
                "versions_seen": {
                    "__input__": {},
                    "__start__": {
                        "__start__": "00000000000000000000000000000001.0.7234984738744598"
                    },
                    "rewrite_query": {
                        "branch:to:rewrite_query": "00000000000000000000000000000002.0.05597832024496252"
                    },
                },
                "channel_values": {
                    "query": "query: what is weather in sf",
                    "branch:to:analyzer_one": None,
                    "branch:to:retriever_two": None,
                },
                "updated_channels": None,
            },
            metadata={
                "source": "loop",
                "step": 1,
                "parents": {},
            },
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fd5f-2038-613e-8000-ce5ebe65eb97",
                }
            },
            pending_writes=[
                (
                    "4e7cb70b-7e0f-52d0-d8aa-5439bd3f84de",
                    "query",
                    "analyzed: query: what is weather in sf",
                ),
                (
                    "4e7cb70b-7e0f-52d0-d8aa-5439bd3f84de",
                    "branch:to:retriever_one",
                    None,
                ),
                ("abcbc448-cfba-ac2b-2e39-346808f20add", "docs", ["doc3", "doc4"]),
            ],
        ),
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fd5f-2038-613e-8000-ce5ebe65eb97",
                }
            },
            checkpoint={
                "v": 2,
                "ts": "2025-04-02T15:20:01.125200+00:00",
                "id": "1f00fd5f-2038-613e-8000-ce5ebe65eb97",
                "channel_versions": {
                    "__start__": "00000000000000000000000000000002.0.6697367414225304",
                    "query": "00000000000000000000000000000002.0.3399249312096154",
                    "branch:to:rewrite_query": "00000000000000000000000000000002.0.05597832024496252",
                },
                "versions_seen": {
                    "__input__": {},
                    "__start__": {
                        "__start__": "00000000000000000000000000000001.0.7234984738744598"
                    },
                },
                "channel_values": {
                    "query": "what is weather in sf",
                    "branch:to:rewrite_query": None,
                },
                "updated_channels": None,
            },
            metadata={
                "source": "loop",
                "step": 0,
                "parents": {},
            },
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fd5f-2036-6ce4-bfff-ac42e9890362",
                }
            },
            pending_writes=[
                (
                    "d1c3a2d6-5ca2-d4c5-5217-35d86cce48a4",
                    "query",
                    "query: what is weather in sf",
                ),
                (
                    "d1c3a2d6-5ca2-d4c5-5217-35d86cce48a4",
                    "branch:to:analyzer_one",
                    None,
                ),
                (
                    "d1c3a2d6-5ca2-d4c5-5217-35d86cce48a4",
                    "branch:to:retriever_two",
                    None,
                ),
            ],
        ),
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fd5f-2036-6ce4-bfff-ac42e9890362",
                }
            },
            checkpoint={
                "v": 2,
                "ts": "2025-04-02T15:20:01.124678+00:00",
                "id": "1f00fd5f-2036-6ce4-bfff-ac42e9890362",
                "channel_versions": {
                    "__start__": "00000000000000000000000000000001.0.7234984738744598"
                },
                "versions_seen": {"__input__": {}},
                "channel_values": {"__start__": {"query": "what is weather in sf"}},
                "updated_channels": None,
            },
            metadata={
                "source": "input",
                "step": -1,
                "parents": {},
            },
            parent_config=None,
            pending_writes=[
                (
                    "a9e2a749-9870-1952-0a6c-b23b6729ffda",
                    "query",
                    "what is weather in sf",
                ),
                (
                    "a9e2a749-9870-1952-0a6c-b23b6729ffda",
                    "branch:to:rewrite_query",
                    None,
                ),
            ],
        ),
    ],
    "2-start:*": [
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fe48-515f-6b88-8004-6fffb69dd465",
                }
            },
            checkpoint={
                "v": 2,
                "ts": "2025-04-02T17:04:20.825576+00:00",
                "id": "1f00fe48-515f-6b88-8004-6fffb69dd465",
                "channel_versions": {
                    "__start__": "00000000000000000000000000000002.0.23383372151016169",
                    "query": "00000000000000000000000000000004.0.05732679770452498",
                    "start:rewrite_query": "00000000000000000000000000000003.0.2916637829964738",
                    "rewrite_query": "00000000000000000000000000000004.0.2372002638794427",
                    "branch:to:retriever_two": "00000000000000000000000000000004.0.8860781568140047",
                    "analyzer_one": "00000000000000000000000000000005.0.648286705356163",
                    "docs": "00000000000000000000000000000005.0.19918575623485935",
                    "retriever_two": "00000000000000000000000000000005.0.46629341414062697",
                    "retriever_one": "00000000000000000000000000000006.0.9577453764095437",
                    "answer": "00000000000000000000000000000006.0.27361287406148327",
                    "qa": "00000000000000000000000000000006.0.24260043089701677",
                },
                "versions_seen": {
                    "__input__": {},
                    "__start__": {
                        "__start__": "00000000000000000000000000000001.0.9575279209966122"
                    },
                    "rewrite_query": {
                        "start:rewrite_query": "00000000000000000000000000000002.0.3082066433110763"
                    },
                    "analyzer_one": {
                        "rewrite_query": "00000000000000000000000000000003.0.9534854313752955"
                    },
                    "retriever_two": {
                        "branch:to:retriever_two": "00000000000000000000000000000003.0.29217346538810884"
                    },
                    "retriever_one": {
                        "analyzer_one": "00000000000000000000000000000004.0.9322215406936268"
                    },
                    "__interrupt__": {
                        "query": "00000000000000000000000000000004.0.05732679770452498",
                        "docs": "00000000000000000000000000000005.0.19918575623485935",
                        "__start__": "00000000000000000000000000000002.0.23383372151016169",
                        "rewrite_query": "00000000000000000000000000000004.0.2372002638794427",
                        "analyzer_one": "00000000000000000000000000000005.0.648286705356163",
                        "retriever_one": "00000000000000000000000000000005.0.0523757506060204",
                        "retriever_two": "00000000000000000000000000000005.0.46629341414062697",
                        "branch:to:retriever_two": "00000000000000000000000000000004.0.8860781568140047",
                        "start:rewrite_query": "00000000000000000000000000000003.0.2916637829964738",
                    },
                    "qa": {
                        "retriever_one": "00000000000000000000000000000005.0.0523757506060204"
                    },
                },
                "channel_values": {
                    "query": "analyzed: query: what is weather in sf",
                    "docs": ["doc1", "doc2", "doc3", "doc4"],
                    "answer": "doc1,doc2,doc3,doc4",
                    "qa": "qa",
                },
            },
            metadata={
                "source": "loop",
                "step": 4,
                "parents": {},
            },
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fe48-515c-679e-8003-5f85a56d5dba",
                }
            },
            pending_writes=[],
        ),
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fe48-515c-679e-8003-5f85a56d5dba",
                }
            },
            checkpoint={
                "v": 2,
                "ts": "2025-04-02T17:04:20.824251+00:00",
                "id": "1f00fe48-515c-679e-8003-5f85a56d5dba",
                "channel_versions": {
                    "__start__": "00000000000000000000000000000002.0.23383372151016169",
                    "query": "00000000000000000000000000000004.0.05732679770452498",
                    "start:rewrite_query": "00000000000000000000000000000003.0.2916637829964738",
                    "rewrite_query": "00000000000000000000000000000004.0.2372002638794427",
                    "branch:to:retriever_two": "00000000000000000000000000000004.0.8860781568140047",
                    "analyzer_one": "00000000000000000000000000000005.0.648286705356163",
                    "docs": "00000000000000000000000000000005.0.19918575623485935",
                    "retriever_two": "00000000000000000000000000000005.0.46629341414062697",
                    "retriever_one": "00000000000000000000000000000005.0.0523757506060204",
                },
                "versions_seen": {
                    "__input__": {},
                    "__start__": {
                        "__start__": "00000000000000000000000000000001.0.9575279209966122"
                    },
                    "rewrite_query": {
                        "start:rewrite_query": "00000000000000000000000000000002.0.3082066433110763"
                    },
                    "analyzer_one": {
                        "rewrite_query": "00000000000000000000000000000003.0.9534854313752955"
                    },
                    "retriever_two": {
                        "branch:to:retriever_two": "00000000000000000000000000000003.0.29217346538810884"
                    },
                    "retriever_one": {
                        "analyzer_one": "00000000000000000000000000000004.0.9322215406936268"
                    },
                },
                "channel_values": {
                    "query": "analyzed: query: what is weather in sf",
                    "docs": ["doc1", "doc2", "doc3", "doc4"],
                    "retriever_one": "retriever_one",
                },
            },
            metadata={
                "source": "loop",
                "step": 3,
                "parents": {},
            },
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fe48-515b-6d12-8002-82a8f9213eae",
                }
            },
            pending_writes=[
                (
                    "4ee8637e-0a95-285e-75bc-4da721c0beab",
                    "__interrupt__",
                    [
                        Interrupt(
                            value="",
                            resumable=True,  # type: ignore[arg-type]
                            ns=["qa:4ee8637e-0a95-285e-75bc-4da721c0beab"],  # type: ignore[arg-type]
                        )
                    ],
                ),
                ("00000000-0000-0000-0000-000000000000", "__resume__", ""),
                ("4ee8637e-0a95-285e-75bc-4da721c0beab", "__resume__", [""]),
                (
                    "4ee8637e-0a95-285e-75bc-4da721c0beab",
                    "answer",
                    "doc1,doc2,doc3,doc4",
                ),
                ("4ee8637e-0a95-285e-75bc-4da721c0beab", "qa", "qa"),
            ],
        ),
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fe48-515b-6d12-8002-82a8f9213eae",
                }
            },
            checkpoint={
                "v": 2,
                "ts": "2025-04-02T17:04:20.823978+00:00",
                "id": "1f00fe48-515b-6d12-8002-82a8f9213eae",
                "channel_versions": {
                    "__start__": "00000000000000000000000000000002.0.23383372151016169",
                    "query": "00000000000000000000000000000004.0.05732679770452498",
                    "start:rewrite_query": "00000000000000000000000000000003.0.2916637829964738",
                    "rewrite_query": "00000000000000000000000000000004.0.2372002638794427",
                    "branch:to:retriever_two": "00000000000000000000000000000004.0.8860781568140047",
                    "analyzer_one": "00000000000000000000000000000004.0.9322215406936268",
                    "docs": "00000000000000000000000000000004.0.49012772235571145",
                    "retriever_two": "00000000000000000000000000000004.0.9223450775254257",
                },
                "versions_seen": {
                    "__input__": {},
                    "__start__": {
                        "__start__": "00000000000000000000000000000001.0.9575279209966122"
                    },
                    "rewrite_query": {
                        "start:rewrite_query": "00000000000000000000000000000002.0.3082066433110763"
                    },
                    "analyzer_one": {
                        "rewrite_query": "00000000000000000000000000000003.0.9534854313752955"
                    },
                    "retriever_two": {
                        "branch:to:retriever_two": "00000000000000000000000000000003.0.29217346538810884"
                    },
                },
                "channel_values": {
                    "query": "analyzed: query: what is weather in sf",
                    "analyzer_one": "analyzer_one",
                    "docs": ["doc3", "doc4"],
                    "retriever_two": "retriever_two",
                },
            },
            metadata={
                "source": "loop",
                "step": 2,
                "parents": {},
            },
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fe48-5059-6b30-8001-2a9ab4ca7d82",
                }
            },
            pending_writes=[
                ("16295c56-f44e-31fa-8fad-fff3f9022629", "docs", ["doc1", "doc2"]),
                (
                    "16295c56-f44e-31fa-8fad-fff3f9022629",
                    "retriever_one",
                    "retriever_one",
                ),
            ],
        ),
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fe48-5059-6b30-8001-2a9ab4ca7d82",
                }
            },
            checkpoint={
                "v": 2,
                "ts": "2025-04-02T17:04:20.718258+00:00",
                "id": "1f00fe48-5059-6b30-8001-2a9ab4ca7d82",
                "channel_versions": {
                    "__start__": "00000000000000000000000000000002.0.23383372151016169",
                    "query": "00000000000000000000000000000003.0.10748450241039154",
                    "start:rewrite_query": "00000000000000000000000000000003.0.2916637829964738",
                    "rewrite_query": "00000000000000000000000000000003.0.9534854313752955",
                    "branch:to:retriever_two": "00000000000000000000000000000003.0.29217346538810884",
                },
                "versions_seen": {
                    "__input__": {},
                    "__start__": {
                        "__start__": "00000000000000000000000000000001.0.9575279209966122"
                    },
                    "rewrite_query": {
                        "start:rewrite_query": "00000000000000000000000000000002.0.3082066433110763"
                    },
                },
                "channel_values": {
                    "query": "query: what is weather in sf",
                    "rewrite_query": "rewrite_query",
                    "branch:to:retriever_two": "rewrite_query",
                },
            },
            metadata={
                "source": "loop",
                "step": 1,
                "parents": {},
            },
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fe48-5058-6a46-8000-086bffc73797",
                }
            },
            pending_writes=[
                (
                    "baecc0e3-ea00-0e00-9436-e33cd2527faf",
                    "query",
                    "analyzed: query: what is weather in sf",
                ),
                (
                    "baecc0e3-ea00-0e00-9436-e33cd2527faf",
                    "analyzer_one",
                    "analyzer_one",
                ),
                ("96b7bfe4-269f-092c-e685-14dba6a27271", "docs", ["doc3", "doc4"]),
                (
                    "96b7bfe4-269f-092c-e685-14dba6a27271",
                    "retriever_two",
                    "retriever_two",
                ),
            ],
        ),
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fe48-5058-6a46-8000-086bffc73797",
                }
            },
            checkpoint={
                "v": 2,
                "ts": "2025-04-02T17:04:20.717827+00:00",
                "id": "1f00fe48-5058-6a46-8000-086bffc73797",
                "channel_versions": {
                    "__start__": "00000000000000000000000000000002.0.23383372151016169",
                    "query": "00000000000000000000000000000002.0.706632616485588",
                    "start:rewrite_query": "00000000000000000000000000000002.0.3082066433110763",
                },
                "versions_seen": {
                    "__input__": {},
                    "__start__": {
                        "__start__": "00000000000000000000000000000001.0.9575279209966122"
                    },
                },
                "channel_values": {
                    "query": "what is weather in sf",
                    "start:rewrite_query": "__start__",
                },
            },
            metadata={
                "source": "loop",
                "step": 0,
                "parents": {},
            },
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fe48-5057-62a4-bfff-1883a92a3e41",
                }
            },
            pending_writes=[
                (
                    "058cf6d6-a83c-6509-b398-5dde0b6c5773",
                    "query",
                    "query: what is weather in sf",
                ),
                (
                    "058cf6d6-a83c-6509-b398-5dde0b6c5773",
                    "rewrite_query",
                    "rewrite_query",
                ),
                (
                    "058cf6d6-a83c-6509-b398-5dde0b6c5773",
                    "branch:to:retriever_two",
                    "rewrite_query",
                ),
            ],
        ),
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00fe48-5057-62a4-bfff-1883a92a3e41",
                }
            },
            checkpoint={
                "v": 2,
                "ts": "2025-04-02T17:04:20.717221+00:00",
                "id": "1f00fe48-5057-62a4-bfff-1883a92a3e41",
                "channel_versions": {
                    "__start__": "00000000000000000000000000000001.0.9575279209966122"
                },
                "versions_seen": {"__input__": {}},
                "channel_values": {"__start__": {"query": "what is weather in sf"}},
            },
            metadata={
                "source": "input",
                "step": -1,
                "parents": {},
            },
            parent_config=None,
            pending_writes=[
                (
                    "891e8564-d78f-7fb2-f15d-bce2a0ddf1c6",
                    "query",
                    "what is weather in sf",
                ),
                (
                    "891e8564-d78f-7fb2-f15d-bce2a0ddf1c6",
                    "start:rewrite_query",
                    "__start__",
                ),
            ],
        ),
    ],
    "2-quadratic": [
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00ffbe-546a-64f0-8004-d2d4e06b6fb6",
                }
            },
            checkpoint={
                "v": 1,
                "ts": "2025-04-02T19:51:40.630535+00:00",
                "id": "1f00ffbe-546a-64f0-8004-d2d4e06b6fb6",
                "channel_values": {
                    "query": "analyzed: query: what is weather in sf",
                    "answer": "doc1,doc2,doc3,doc4",
                    "docs": ["doc1", "doc2", "doc3", "doc4"],
                    "qa": "qa",
                },
                "channel_versions": {
                    "__start__": "00000000000000000000000000000002.0.141080617837112",
                    "query": "00000000000000000000000000000004.0.9004905790383284",
                    "start:rewrite_query": "00000000000000000000000000000003.0.013109117892399547",
                    "rewrite_query": "00000000000000000000000000000004.0.1679336326485974",
                    "branch:rewrite_query:rewrite_query_then:retriever_two": "00000000000000000000000000000004.0.7474512867042074",
                    "analyzer_one": "00000000000000000000000000000005.0.5817293698381076",
                    "docs": "00000000000000000000000000000005.0.9650795030435029",
                    "retriever_two": "00000000000000000000000000000005.0.77101858493518",
                    "retriever_one": "00000000000000000000000000000006.0.4984661612084784",
                    "answer": "00000000000000000000000000000006.0.6244466008661432",
                    "qa": "00000000000000000000000000000006.0.06630110662217248",
                },
                "versions_seen": {
                    "__input__": {},
                    "__start__": {
                        "__start__": "00000000000000000000000000000001.0.6759219622820284"
                    },
                    "rewrite_query": {
                        "start:rewrite_query": "00000000000000000000000000000002.0.32002588286540445"
                    },
                    "analyzer_one": {
                        "rewrite_query": "00000000000000000000000000000003.0.32578323811902354"
                    },
                    "retriever_two": {
                        "branch:rewrite_query:rewrite_query_then:retriever_two": "00000000000000000000000000000003.0.8992241767805405"
                    },
                    "retriever_one": {
                        "analyzer_one": "00000000000000000000000000000004.0.2684613370070208"
                    },
                    "__interrupt__": {
                        "query": "00000000000000000000000000000004.0.9004905790383284",
                        "docs": "00000000000000000000000000000005.0.9650795030435029",
                        "__start__": "00000000000000000000000000000002.0.141080617837112",
                        "rewrite_query": "00000000000000000000000000000004.0.1679336326485974",
                        "analyzer_one": "00000000000000000000000000000005.0.5817293698381076",
                        "retriever_one": "00000000000000000000000000000005.0.222301724202566",
                        "retriever_two": "00000000000000000000000000000005.0.77101858493518",
                        "start:rewrite_query": "00000000000000000000000000000003.0.013109117892399547",
                        "branch:rewrite_query:rewrite_query_then:retriever_two": "00000000000000000000000000000004.0.7474512867042074",
                    },
                    "qa": {
                        "retriever_one": "00000000000000000000000000000005.0.222301724202566"
                    },
                },
            },
            metadata={
                "source": "loop",
                "step": 4,
                "parents": {},
            },
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00ffbe-5466-61ac-8003-7ec684cd12cc",
                }
            },
            pending_writes=[],
        ),
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00ffbe-5466-61ac-8003-7ec684cd12cc",
                }
            },
            checkpoint={
                "v": 1,
                "ts": "2025-04-02T19:51:40.628817+00:00",
                "id": "1f00ffbe-5466-61ac-8003-7ec684cd12cc",
                "channel_values": {
                    "query": "analyzed: query: what is weather in sf",
                    "docs": ["doc1", "doc2", "doc3", "doc4"],
                    "retriever_one": "retriever_one",
                },
                "channel_versions": {
                    "__start__": "00000000000000000000000000000002.0.141080617837112",
                    "query": "00000000000000000000000000000004.0.9004905790383284",
                    "start:rewrite_query": "00000000000000000000000000000003.0.013109117892399547",
                    "rewrite_query": "00000000000000000000000000000004.0.1679336326485974",
                    "branch:rewrite_query:rewrite_query_then:retriever_two": "00000000000000000000000000000004.0.7474512867042074",
                    "analyzer_one": "00000000000000000000000000000005.0.5817293698381076",
                    "docs": "00000000000000000000000000000005.0.9650795030435029",
                    "retriever_two": "00000000000000000000000000000005.0.77101858493518",
                    "retriever_one": "00000000000000000000000000000005.0.222301724202566",
                },
                "versions_seen": {
                    "__input__": {},
                    "__start__": {
                        "__start__": "00000000000000000000000000000001.0.6759219622820284"
                    },
                    "rewrite_query": {
                        "start:rewrite_query": "00000000000000000000000000000002.0.32002588286540445"
                    },
                    "analyzer_one": {
                        "rewrite_query": "00000000000000000000000000000003.0.32578323811902354"
                    },
                    "retriever_two": {
                        "branch:rewrite_query:rewrite_query_then:retriever_two": "00000000000000000000000000000003.0.8992241767805405"
                    },
                    "retriever_one": {
                        "analyzer_one": "00000000000000000000000000000004.0.2684613370070208"
                    },
                },
            },
            metadata={
                "source": "loop",
                "step": 3,
                "parents": {},
            },
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00ffbe-5465-6248-8002-ee1d8bdbbee5",
                }
            },
            pending_writes=[
                (
                    "369e94b1-77d1-d67a-ab59-23d1ba20ee73",
                    "__interrupt__",
                    [
                        Interrupt(
                            value="",
                            resumable=True,
                            ns=["qa:369e94b1-77d1-d67a-ab59-23d1ba20ee73"],  # type: ignore[arg-type]
                        )
                    ],
                ),
                ("00000000-0000-0000-0000-000000000000", "__resume__", ""),
                ("369e94b1-77d1-d67a-ab59-23d1ba20ee73", "__resume__", [""]),
                (
                    "369e94b1-77d1-d67a-ab59-23d1ba20ee73",
                    "answer",
                    "doc1,doc2,doc3,doc4",
                ),
                ("369e94b1-77d1-d67a-ab59-23d1ba20ee73", "qa", "qa"),
            ],
        ),
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00ffbe-5465-6248-8002-ee1d8bdbbee5",
                }
            },
            checkpoint={
                "v": 1,
                "ts": "2025-04-02T19:51:40.628408+00:00",
                "id": "1f00ffbe-5465-6248-8002-ee1d8bdbbee5",
                "channel_values": {
                    "query": "analyzed: query: what is weather in sf",
                    "docs": ["doc3", "doc4"],
                    "analyzer_one": "analyzer_one",
                    "retriever_two": "retriever_two",
                },
                "channel_versions": {
                    "__start__": "00000000000000000000000000000002.0.141080617837112",
                    "query": "00000000000000000000000000000004.0.9004905790383284",
                    "start:rewrite_query": "00000000000000000000000000000003.0.013109117892399547",
                    "rewrite_query": "00000000000000000000000000000004.0.1679336326485974",
                    "branch:rewrite_query:rewrite_query_then:retriever_two": "00000000000000000000000000000004.0.7474512867042074",
                    "analyzer_one": "00000000000000000000000000000004.0.2684613370070208",
                    "docs": "00000000000000000000000000000004.0.37458911821520957",
                    "retriever_two": "00000000000000000000000000000004.0.7340649978617967",
                },
                "versions_seen": {
                    "__input__": {},
                    "__start__": {
                        "__start__": "00000000000000000000000000000001.0.6759219622820284"
                    },
                    "rewrite_query": {
                        "start:rewrite_query": "00000000000000000000000000000002.0.32002588286540445"
                    },
                    "analyzer_one": {
                        "rewrite_query": "00000000000000000000000000000003.0.32578323811902354"
                    },
                    "retriever_two": {
                        "branch:rewrite_query:rewrite_query_then:retriever_two": "00000000000000000000000000000003.0.8992241767805405"
                    },
                },
            },
            metadata={
                "source": "loop",
                "step": 2,
                "parents": {},
            },
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00ffbe-536a-6dca-8001-c7e021a73244",
                }
            },
            pending_writes=[
                ("601f2099-cb23-41f9-ae64-9b4e4a6b675e", "docs", ["doc1", "doc2"]),
                (
                    "601f2099-cb23-41f9-ae64-9b4e4a6b675e",
                    "retriever_one",
                    "retriever_one",
                ),
            ],
        ),
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00ffbe-536a-6dca-8001-c7e021a73244",
                }
            },
            checkpoint={
                "v": 1,
                "ts": "2025-04-02T19:51:40.525915+00:00",
                "id": "1f00ffbe-536a-6dca-8001-c7e021a73244",
                "channel_values": {
                    "query": "query: what is weather in sf",
                    "rewrite_query": "rewrite_query",
                    "branch:rewrite_query:rewrite_query_then:retriever_two": "rewrite_query",
                },
                "channel_versions": {
                    "__start__": "00000000000000000000000000000002.0.141080617837112",
                    "query": "00000000000000000000000000000003.0.8982471206042032",
                    "start:rewrite_query": "00000000000000000000000000000003.0.013109117892399547",
                    "rewrite_query": "00000000000000000000000000000003.0.32578323811902354",
                    "branch:rewrite_query:rewrite_query_then:retriever_two": "00000000000000000000000000000003.0.8992241767805405",
                },
                "versions_seen": {
                    "__input__": {},
                    "__start__": {
                        "__start__": "00000000000000000000000000000001.0.6759219622820284"
                    },
                    "rewrite_query": {
                        "start:rewrite_query": "00000000000000000000000000000002.0.32002588286540445"
                    },
                },
            },
            metadata={
                "source": "loop",
                "step": 1,
                "parents": {},
            },
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00ffbe-5369-6de4-8000-70bd3810f0ca",
                }
            },
            pending_writes=[
                (
                    "88757475-bc8c-934c-7d18-921cb2d94864",
                    "query",
                    "analyzed: query: what is weather in sf",
                ),
                (
                    "88757475-bc8c-934c-7d18-921cb2d94864",
                    "analyzer_one",
                    "analyzer_one",
                ),
                ("53c60600-588b-e49e-a9d2-bbcbf30a7497", "docs", ["doc3", "doc4"]),
                (
                    "53c60600-588b-e49e-a9d2-bbcbf30a7497",
                    "retriever_two",
                    "retriever_two",
                ),
            ],
        ),
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00ffbe-5369-6de4-8000-70bd3810f0ca",
                }
            },
            checkpoint={
                "v": 1,
                "ts": "2025-04-02T19:51:40.525508+00:00",
                "id": "1f00ffbe-5369-6de4-8000-70bd3810f0ca",
                "channel_values": {
                    "query": "what is weather in sf",
                    "start:rewrite_query": "__start__",
                },
                "channel_versions": {
                    "__start__": "00000000000000000000000000000002.0.141080617837112",
                    "query": "00000000000000000000000000000002.0.06948551802156189",
                    "start:rewrite_query": "00000000000000000000000000000002.0.32002588286540445",
                },
                "versions_seen": {
                    "__input__": {},
                    "__start__": {
                        "__start__": "00000000000000000000000000000001.0.6759219622820284"
                    },
                },
            },
            metadata={
                "source": "loop",
                "step": 0,
                "parents": {},
            },
            parent_config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00ffbe-5368-6c0a-bfff-ac22bea4c512",
                }
            },
            pending_writes=[
                (
                    "3bb470dd-9bf8-6216-b5fb-e50162991da1",
                    "query",
                    "query: what is weather in sf",
                ),
                (
                    "3bb470dd-9bf8-6216-b5fb-e50162991da1",
                    "rewrite_query",
                    "rewrite_query",
                ),
                (
                    "3bb470dd-9bf8-6216-b5fb-e50162991da1",
                    "branch:rewrite_query:rewrite_query_then:retriever_two",
                    "rewrite_query",
                ),
            ],
        ),
        CheckpointTuple(
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": "1f00ffbe-5368-6c0a-bfff-ac22bea4c512",
                }
            },
            checkpoint={
                "v": 1,
                "ts": "2025-04-02T19:51:40.525051+00:00",
                "id": "1f00ffbe-5368-6c0a-bfff-ac22bea4c512",
                "channel_values": {"__start__": {"query": "what is weather in sf"}},
                "channel_versions": {
                    "__start__": "00000000000000000000000000000001.0.6759219622820284"
                },
                "versions_seen": {"__input__": {}},
            },
            metadata={
                "source": "input",
                "step": -1,
                "parents": {},
            },
            parent_config=None,
            pending_writes=[
                (
                    "aa8c5e8a-da6f-ccb1-f8a9-3b145cdfe7a4",
                    "query",
                    "what is weather in sf",
                ),
                (
                    "aa8c5e8a-da6f-ccb1-f8a9-3b145cdfe7a4",
                    "start:rewrite_query",
                    "__start__",
                ),
            ],
        ),
    ],
}


def make_state_graph() -> StateGraph:
    def sorted_add(x: list[str], y: list[str] | list[tuple[str, str]]) -> list[str]:
        if isinstance(y[0], tuple):
            for rem, _ in y:
                x.remove(rem)
            y = [t[1] for t in y]
        return sorted(operator.add(x, y))

    class State(TypedDict, total=False):
        query: str
        answer: str
        docs: Annotated[list[str], sorted_add]

    def rewrite_query(data: State) -> State:
        return {"query": f"query: {data['query']}"}

    def analyzer_one(data: State) -> State:
        return {"query": f"analyzed: {data['query']}"}

    def retriever_one(data: State) -> State:
        return {"docs": ["doc1", "doc2"]}

    def retriever_two(data: State) -> State:
        time.sleep(0.1)
        return {"docs": ["doc3", "doc4"]}

    def qa(data: State) -> State:
        interrupt("")
        return {"answer": ",".join(data["docs"])}

    def rewrite_query_then(data: State) -> Literal["retriever_two"]:
        return "retriever_two"

    workflow = StateGraph(State)

    workflow.add_node("rewrite_query", rewrite_query)
    workflow.add_node("analyzer_one", analyzer_one)
    workflow.add_node("retriever_one", retriever_one)
    workflow.add_node("retriever_two", retriever_two)
    workflow.add_node("qa", qa)

    workflow.set_entry_point("rewrite_query")
    workflow.add_edge("rewrite_query", "analyzer_one")
    workflow.add_edge("analyzer_one", "retriever_one")
    workflow.add_conditional_edges("rewrite_query", rewrite_query_then)
    workflow.add_edge("retriever_one", "qa")
    workflow.set_finish_point("qa")
    return workflow


@NEEDS_CONTEXTVARS
@pytest.mark.parametrize("source,target", [("2-start:*", "3"), ("2-quadratic", "3")])
def test_migrate_checkpoints(source: str, target: str) -> None:
    # Check that the migration function works as expected
    builder = make_state_graph()
    graph = builder.compile()

    source_checkpoints = list(reversed(SAVED_CHECKPOINTS[source]))
    target_checkpoints = list(reversed(SAVED_CHECKPOINTS[target]))
    assert len(source_checkpoints) == len(target_checkpoints)
    for idx, (source_checkpoint, target_checkpoint) in enumerate(
        zip(source_checkpoints, target_checkpoints)
    ):
        # copy the checkpoint to avoid modifying the original
        migrated = copy_checkpoint(source_checkpoint.checkpoint)
        # migrate the checkpoint
        graph._migrate_checkpoint(migrated)
        # replace values that don't need to match exactly
        migrated["id"] = AnyStr()
        migrated["ts"] = AnyStr()
        migrated["v"] = AnyInt()
        for k in migrated["channel_values"]:
            migrated["channel_values"][k] = AnyObject()
        for v in migrated["channel_versions"]:
            migrated["channel_versions"][v] = AnyStr(
                migrated["channel_versions"][v].split(".")[0]
            )
        for c in migrated["versions_seen"]:
            for v in migrated["versions_seen"][c]:
                migrated["versions_seen"][c][v] = AnyStr(
                    migrated["versions_seen"][c][v].split(".")[0]
                )
        # check that the migrated checkpoint matches the target checkpoint
        assert migrated == target_checkpoint.checkpoint, (
            f"Checkpoint mismatch at index {idx}"
        )


@NEEDS_CONTEXTVARS
def test_latest_checkpoint_state_graph(
    sync_checkpointer: BaseCheckpointSaver,
) -> None:
    builder = make_state_graph()
    app = builder.compile(checkpointer=sync_checkpointer)
    config = {"configurable": {"thread_id": "1"}}

    assert [
        *app.stream({"query": "what is weather in sf"}, config, durability="async")
    ] == [
        {"rewrite_query": {"query": "query: what is weather in sf"}},
        {"analyzer_one": {"query": "analyzed: query: what is weather in sf"}},
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {
            "__interrupt__": (
                Interrupt(
                    value="",
                    id=AnyStr(),
                ),
            )
        },
    ]

    assert [*app.stream(Command(resume=""), config, durability="async")] == [
        {"qa": {"answer": "doc1,doc2,doc3,doc4"}},
    ]

    # check history with current checkpoints matches expected history
    history = [*app.get_state_history(config)]
    expected_history = get_expected_history()
    assert len(history) == len(expected_history)
    assert history[0] == expected_history[0]
    assert history[1] == expected_history[1]
    assert history[2] == expected_history[2]
    assert history[3] == expected_history[3]
    assert history[4] == expected_history[4]
    assert history[5] == expected_history[5]


@NEEDS_CONTEXTVARS
async def test_latest_checkpoint_state_graph_async(
    async_checkpointer: BaseCheckpointSaver,
) -> None:
    builder = make_state_graph()
    app = builder.compile(checkpointer=async_checkpointer)
    config = {"configurable": {"thread_id": "1"}}

    assert [
        c
        async for c in app.astream(
            {"query": "what is weather in sf"}, config, durability="async"
        )
    ] == [
        {"rewrite_query": {"query": "query: what is weather in sf"}},
        {"analyzer_one": {"query": "analyzed: query: what is weather in sf"}},
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {
            "__interrupt__": (
                Interrupt(
                    value="",
                    id=AnyStr(),
                ),
            )
        },
    ]

    assert [
        c async for c in app.astream(Command(resume=""), config, durability="async")
    ] == [
        {"qa": {"answer": "doc1,doc2,doc3,doc4"}},
    ]

    # check history with current checkpoints matches expected history
    history = [c async for c in app.aget_state_history(config)]
    expected_history = get_expected_history()
    assert len(history) == len(expected_history)
    assert history[0] == expected_history[0]
    assert history[1] == expected_history[1]
    assert history[2] == expected_history[2]
    assert history[3] == expected_history[3]
    assert history[4] == expected_history[4]
    assert history[5] == expected_history[5]


@NEEDS_CONTEXTVARS
@pytest.mark.parametrize("checkpoint_version", ["3", "2-start:*", "2-quadratic"])
def test_saved_checkpoint_state_graph(
    sync_checkpointer: BaseCheckpointSaver,
    checkpoint_version: str,
) -> None:
    builder = make_state_graph()
    app = builder.compile(checkpointer=sync_checkpointer)

    thread1 = "1"
    config = {"configurable": {"thread_id": thread1, "checkpoint_ns": ""}}

    # save checkpoints
    parent_id: str | None = None
    for checkpoint in reversed(SAVED_CHECKPOINTS[checkpoint_version]):
        grouped_writes = defaultdict(list)
        for write in checkpoint.pending_writes:
            grouped_writes[write[0]].append(write[1:])
        for tid, group in grouped_writes.items():
            sync_checkpointer.put_writes(checkpoint.config, group, tid)
        sync_checkpointer.put(
            patch_configurable(config, {"checkpoint_id": parent_id}),
            checkpoint.checkpoint,
            checkpoint.metadata,
            checkpoint.checkpoint["channel_versions"],
        )
        parent_id = checkpoint.checkpoint["id"]

    # load history
    history = [*app.get_state_history(config)]
    # check history with saved checkpoints matches expected history
    exc_task_results: int = 0
    if checkpoint_version == "2-start:*":
        exc_task_results = 1
    elif checkpoint_version == "2-quadratic":
        exc_task_results = 2
    expected_history = get_expected_history(exc_task_results=exc_task_results)
    assert len(history) == len(expected_history)
    assert history[0] == expected_history[0]
    assert history[1] == expected_history[1]
    assert history[2] == expected_history[2]
    assert history[3] == expected_history[3]
    assert history[4] == expected_history[4]
    assert history[5] == expected_history[5]

    # resume from 2nd to latest checkpoint
    assert [*app.stream(Command(resume=""), history[1].config)] == [
        {"qa": {"answer": "doc1,doc2,doc3,doc4"}},
    ]
    # new checkpoint should match the latest checkpoint in history
    latest_state = app.get_state(config)
    assert (
        StateSnapshot(
            values=latest_state.values,
            next=latest_state.next,
            config=patch_configurable(latest_state.config, {"checkpoint_id": AnyStr()}),
            metadata=AnyDict(latest_state.metadata),
            created_at=AnyStr(),
            parent_config=latest_state.parent_config,
            tasks=latest_state.tasks,
            interrupts=latest_state.interrupts,
        )
        == history[0]
    )


@NEEDS_CONTEXTVARS
@pytest.mark.parametrize("checkpoint_version", ["3", "2-start:*", "2-quadratic"])
async def test_saved_checkpoint_state_graph_async(
    async_checkpointer: BaseCheckpointSaver,
    checkpoint_version: str,
) -> None:
    builder = make_state_graph()
    app = builder.compile(checkpointer=async_checkpointer)

    thread1 = "1"
    config = {"configurable": {"thread_id": thread1, "checkpoint_ns": ""}}

    # save checkpoints
    parent_id: str | None = None
    for checkpoint in reversed(SAVED_CHECKPOINTS[checkpoint_version]):
        grouped_writes = defaultdict(list)
        for write in checkpoint.pending_writes:
            grouped_writes[write[0]].append(write[1:])
        for tid, group in grouped_writes.items():
            await async_checkpointer.aput_writes(checkpoint.config, group, tid)
        await async_checkpointer.aput(
            patch_configurable(config, {"checkpoint_id": parent_id}),
            checkpoint.checkpoint,
            checkpoint.metadata,
            checkpoint.checkpoint["channel_versions"],
        )
        parent_id = checkpoint.checkpoint["id"]

    # load history
    history = [c async for c in app.aget_state_history(config)]
    # check history with saved checkpoints matches expected history
    exc_task_results: int = 0
    if checkpoint_version == "2-start:*":
        exc_task_results = 1
    elif checkpoint_version == "2-quadratic":
        exc_task_results = 2
    expected_history = get_expected_history(exc_task_results=exc_task_results)
    assert len(history) == len(expected_history)
    assert history[0] == expected_history[0]
    assert history[1] == expected_history[1]
    assert history[2] == expected_history[2]
    assert history[3] == expected_history[3]
    assert history[4] == expected_history[4]
    assert history[5] == expected_history[5]

    # resume from 2nd to latest checkpoint
    assert [c async for c in app.astream(Command(resume=""), history[1].config)] == [
        {"qa": {"answer": "doc1,doc2,doc3,doc4"}},
    ]
    # new checkpoint should match the latest checkpoint in history
    latest_state = await app.aget_state(config)
    assert (
        StateSnapshot(
            values=latest_state.values,
            next=latest_state.next,
            config=patch_configurable(latest_state.config, {"checkpoint_id": AnyStr()}),
            metadata=AnyDict(latest_state.metadata),
            created_at=AnyStr(),
            parent_config=latest_state.parent_config,
            tasks=latest_state.tasks,
            interrupts=latest_state.interrupts,
        )
        == history[0]
    )

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/test_config_async.py
```py
import pytest
from langchain_core.callbacks import AsyncCallbackManager

from langgraph._internal._config import get_async_callback_manager_for_config

pytestmark = pytest.mark.anyio


def test_new_async_manager_includes_tags() -> None:
    config = {"callbacks": None}
    manager = get_async_callback_manager_for_config(config, tags=["x", "y"])
    assert isinstance(manager, AsyncCallbackManager)
    assert manager.inheritable_tags == ["x", "y"]


def test_new_async_manager_merges_tags_with_config() -> None:
    config = {"callbacks": None, "tags": ["a"]}
    manager = get_async_callback_manager_for_config(config, tags=["b"])
    assert manager.inheritable_tags == ["a", "b"]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/test_deprecation.py
```py
from __future__ import annotations

import warnings
from typing import Any, Optional

import pytest
from langchain_core.runnables import RunnableConfig
from pytest_mock import MockerFixture
from typing_extensions import NotRequired, TypedDict

from langgraph.channels.last_value import LastValue
from langgraph.errors import NodeInterrupt
from langgraph.func import entrypoint, task
from langgraph.graph import StateGraph
from langgraph.graph.message import MessageGraph
from langgraph.pregel import NodeBuilder, Pregel
from langgraph.types import Interrupt, RetryPolicy
from langgraph.warnings import LangGraphDeprecatedSinceV05, LangGraphDeprecatedSinceV10


class PlainState(TypedDict): ...


def test_add_node_retry_arg() -> None:
    builder = StateGraph(PlainState)

    with pytest.warns(
        LangGraphDeprecatedSinceV05,
        match="`retry` is deprecated and will be removed. Please use `retry_policy` instead.",
    ):
        builder.add_node("test_node", lambda state: state, retry=RetryPolicy())  # type: ignore[arg-type]


def test_task_retry_arg() -> None:
    with pytest.warns(
        LangGraphDeprecatedSinceV05,
        match="`retry` is deprecated and will be removed. Please use `retry_policy` instead.",
    ):

        @task(retry=RetryPolicy())  # type: ignore[arg-type]
        def my_task(state: PlainState) -> PlainState:
            return state


def test_entrypoint_retry_arg() -> None:
    with pytest.warns(
        LangGraphDeprecatedSinceV05,
        match="`retry` is deprecated and will be removed. Please use `retry_policy` instead.",
    ):

        @entrypoint(retry=RetryPolicy())  # type: ignore[arg-type]
        def my_entrypoint(state: PlainState) -> PlainState:
            return state


def test_state_graph_input_schema() -> None:
    with pytest.warns(
        LangGraphDeprecatedSinceV05,
        match="`input` is deprecated and will be removed. Please use `input_schema` instead.",
    ):
        StateGraph(PlainState, input=PlainState)  # type: ignore[arg-type]


def test_state_graph_output_schema() -> None:
    with pytest.warns(
        LangGraphDeprecatedSinceV05,
        match="`output` is deprecated and will be removed. Please use `output_schema` instead.",
    ):
        StateGraph(PlainState, output=PlainState)  # type: ignore[arg-type]


def test_add_node_input_schema() -> None:
    builder = StateGraph(PlainState)

    with pytest.warns(
        LangGraphDeprecatedSinceV05,
        match="`input` is deprecated and will be removed. Please use `input_schema` instead.",
    ):
        builder.add_node("test_node", lambda state: state, input=PlainState)  # type: ignore[arg-type]


def test_constants_deprecation() -> None:
    with pytest.warns(
        LangGraphDeprecatedSinceV10,
        match="Importing Send from langgraph.constants is deprecated. Please use 'from langgraph.types import Send' instead.",
    ):
        from langgraph.constants import Send  # noqa: F401

    with pytest.warns(
        LangGraphDeprecatedSinceV10,
        match="Importing Interrupt from langgraph.constants is deprecated. Please use 'from langgraph.types import Interrupt' instead.",
    ):
        from langgraph.constants import Interrupt  # noqa: F401


def test_pregel_types_deprecation() -> None:
    with pytest.warns(
        LangGraphDeprecatedSinceV10,
        match="Importing from langgraph.pregel.types is deprecated. Please use 'from langgraph.types import ...' instead.",
    ):
        from langgraph.pregel.types import StateSnapshot  # noqa: F401


def test_config_schema_deprecation() -> None:
    with pytest.warns(
        LangGraphDeprecatedSinceV10,
        match="`config_schema` is deprecated and will be removed. Please use `context_schema` instead.",
    ):
        builder = StateGraph(PlainState, config_schema=PlainState)
        assert builder.context_schema == PlainState

    builder.add_node("test_node", lambda state: state)
    builder.set_entry_point("test_node")
    graph = builder.compile()

    with pytest.warns(
        LangGraphDeprecatedSinceV10,
        match="`config_schema` is deprecated. Use `get_context_jsonschema` for the relevant schema instead.",
    ):
        assert graph.config_schema() is not None

    with pytest.warns(
        LangGraphDeprecatedSinceV10,
        match="`get_config_jsonschema` is deprecated. Use `get_context_jsonschema` instead.",
    ):
        graph.get_config_jsonschema()


def test_config_schema_deprecation_on_entrypoint() -> None:
    with pytest.warns(
        LangGraphDeprecatedSinceV10,
        match="`config_schema` is deprecated and will be removed. Please use `context_schema` instead.",
    ):

        @entrypoint(config_schema=PlainState)  # type: ignore[arg-type]
        def my_entrypoint(state: PlainState) -> PlainState:
            return state

    with pytest.warns(
        LangGraphDeprecatedSinceV10,
        match="`config_schema` is deprecated. Use `get_context_jsonschema` for the relevant schema instead.",
    ):
        assert my_entrypoint.context_schema == PlainState
        assert my_entrypoint.config_schema() is not None


@pytest.mark.filterwarnings("ignore:`config_type` is deprecated")
def test_config_type_deprecation_pregel(mocker: MockerFixture) -> None:
    add_one = mocker.Mock(side_effect=lambda x: x + 1)
    chain = NodeBuilder().subscribe_only("input").do(add_one).write_to("output")

    with pytest.warns(
        LangGraphDeprecatedSinceV10,
        match="`config_type` is deprecated and will be removed. Please use `context_schema` instead.",
    ):
        instance = Pregel(
            nodes={
                "one": chain,
            },
            channels={
                "input": LastValue(int),
                "output": LastValue(int),
            },
            input_channels="input",
            output_channels="output",
            config_type=PlainState,
        )
        assert instance.context_schema == PlainState


def test_interrupt_attributes_deprecation() -> None:
    interrupt = Interrupt(value="question", id="abc")

    with pytest.warns(
        LangGraphDeprecatedSinceV10,
        match="`interrupt_id` is deprecated. Use `id` instead.",
    ):
        interrupt.interrupt_id


def test_node_interrupt_deprecation() -> None:
    with pytest.warns(
        LangGraphDeprecatedSinceV10,
        match="NodeInterrupt is deprecated. Please use `langgraph.types.interrupt` instead.",
    ):
        NodeInterrupt(value="test")


def test_deprecated_import() -> None:
    with pytest.warns(
        LangGraphDeprecatedSinceV10,
        match="Importing PREVIOUS from langgraph.constants is deprecated. This constant is now private and should not be used directly.",
    ):
        from langgraph.constants import PREVIOUS  # noqa: F401


@pytest.mark.filterwarnings(
    "ignore:`durability` has no effect when no checkpointer is present"
)
def test_checkpoint_during_deprecation_state_graph() -> None:
    class CheckDurability(TypedDict):
        durability: NotRequired[str]

    def plain_node(state: CheckDurability, config: RunnableConfig) -> CheckDurability:
        return {"durability": config["configurable"]["__pregel_durability"]}

    builder = StateGraph(CheckDurability)
    builder.add_node("plain_node", plain_node)
    builder.set_entry_point("plain_node")
    graph = builder.compile()

    with pytest.warns(
        LangGraphDeprecatedSinceV10,
        match="`checkpoint_during` is deprecated and will be removed. Please use `durability` instead.",
    ):
        result = graph.invoke({}, checkpoint_during=True)
        assert result["durability"] == "async"

    with pytest.warns(
        LangGraphDeprecatedSinceV10,
        match="`checkpoint_during` is deprecated and will be removed. Please use `durability` instead.",
    ):
        result = graph.invoke({}, checkpoint_during=False)
        assert result["durability"] == "exit"

    with pytest.warns(
        LangGraphDeprecatedSinceV10,
        match="`checkpoint_during` is deprecated and will be removed. Please use `durability` instead.",
    ):
        for chunk in graph.stream({}, checkpoint_during=True):  # type: ignore[arg-type]
            assert chunk["plain_node"]["durability"] == "async"

    with pytest.warns(
        LangGraphDeprecatedSinceV10,
        match="`checkpoint_during` is deprecated and will be removed. Please use `durability` instead.",
    ):
        for chunk in graph.stream({}, checkpoint_during=False):  # type: ignore[arg-type]
            assert chunk["plain_node"]["durability"] == "exit"


def test_config_parameter_incorrect_typing() -> None:
    """Test that a warning is raised when config parameter is typed incorrectly."""
    builder = StateGraph(PlainState)

    # Test sync function with config: dict
    with pytest.warns(
        UserWarning,
        match="The 'config' parameter should be typed as 'RunnableConfig' or 'RunnableConfig | None', not '.*dict.*'. ",
    ):

        def sync_node_with_dict_config(state: PlainState, config: dict) -> PlainState:
            return state

        builder.add_node(sync_node_with_dict_config)

    # Test async function with config: dict
    with pytest.warns(
        UserWarning,
        match="The 'config' parameter should be typed as 'RunnableConfig' or 'RunnableConfig | None', not '.*dict.*'. ",
    ):

        async def async_node_with_dict_config(
            state: PlainState, config: dict
        ) -> PlainState:
            return state

        builder.add_node(async_node_with_dict_config)

    # Test with other incorrect types
    with pytest.warns(
        UserWarning,
        match="The 'config' parameter should be typed as 'RunnableConfig' or 'RunnableConfig | None', not '.*Any.*'. ",
    ):

        def sync_node_with_any_config(state: PlainState, config: Any) -> PlainState:
            return state

        builder.add_node(sync_node_with_any_config)

    with pytest.warns(
        UserWarning,
        match="The 'config' parameter should be typed as 'RunnableConfig' or 'RunnableConfig | None', not '.*Any.*'. ",
    ):

        async def async_node_with_any_config(
            state: PlainState, config: Any
        ) -> PlainState:
            return state

        builder.add_node(async_node_with_any_config)

    with warnings.catch_warnings(record=True) as w:

        def node_with_correct_config(
            state: PlainState, config: RunnableConfig
        ) -> PlainState:
            return state

        builder.add_node(node_with_correct_config)

        def node_with_optional_config(
            state: PlainState,
            config: Optional[RunnableConfig],  # noqa: UP045
        ) -> PlainState:
            return state

        builder.add_node(node_with_optional_config)

        def node_with_untyped_config(state: PlainState, config) -> PlainState:
            return state

        builder.add_node(node_with_untyped_config)

        async def async_node_with_correct_config(
            state: PlainState, config: RunnableConfig
        ) -> PlainState:
            return state

        builder.add_node(async_node_with_correct_config)

        async def async_node_with_optional_config(
            state: PlainState,
            config: Optional[RunnableConfig],  # noqa: UP045
        ) -> PlainState:
            return state

        builder.add_node(async_node_with_optional_config)

        async def async_node_with_untyped_config(
            state: PlainState, config
        ) -> PlainState:
            return state

        builder.add_node(async_node_with_untyped_config)
        assert len(w) == 0


def test_message_graph_deprecation() -> None:
    with pytest.warns(
        LangGraphDeprecatedSinceV10,
        match="MessageGraph is deprecated in LangGraph v1.0.0, to be removed in v2.0.0. Please use StateGraph with a `messages` key instead.",
    ):
        MessageGraph()

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/test_interrupt_migration.py
```py
import warnings

import pytest
from langgraph.checkpoint.serde.jsonplus import JsonPlusSerializer

from langgraph.types import Interrupt
from langgraph.warnings import LangGraphDeprecatedSinceV10


@pytest.mark.filterwarnings("ignore:LangGraphDeprecatedSinceV10")
def test_interrupt_legacy_ns() -> None:
    with warnings.catch_warnings():
        warnings.filterwarnings("ignore", category=LangGraphDeprecatedSinceV10)

        old_interrupt = Interrupt(
            value="abc", resumable=True, when="during", ns=["a:b", "c:d"]
        )

        new_interrupt = Interrupt.from_ns(value="abc", ns="a:b|c:d")
        assert new_interrupt.value == old_interrupt.value
        assert new_interrupt.id == old_interrupt.id


serializer = JsonPlusSerializer(allowed_json_modules=True)


def test_serialization_roundtrip() -> None:
    """Test that the legacy interrupt (pre v1) can be reserialized as the modern interrupt without id corruption."""

    # generated with:
    # JsonPlusSerializer().dumps_typed(Interrupt(value="legacy_test", ns=["legacy_test"], resumable=True, when="during"))
    legacy_interrupt_bytes = b'{"lc": 2, "type": "constructor", "id": ["langgraph", "types", "Interrupt"], "kwargs": {"value": "legacy_test", "resumable": true, "ns": ["legacy_test"], "when": "during"}}'
    legacy_interrupt_id = "f1fa625689ec006a5b32b76863e22a6c"

    interrupt = serializer.loads_typed(("json", legacy_interrupt_bytes))
    assert interrupt.id == legacy_interrupt_id
    assert interrupt.value == "legacy_test"


def test_serialization_roundtrip_complex_ns() -> None:
    """Test that the legacy interrupt (pre v1), with a more complex ns can be reserialized as the modern interrupt without id corruption."""

    # generated with:
    # JsonPlusSerializer().dumps_typed(Interrupt(value="legacy_test", ns=["legacy:test", "with:complex", "name:space"], resumable=True, when="during"))
    legacy_interrupt_bytes = b'{"lc": 2, "type": "constructor", "id": ["langgraph", "types", "Interrupt"], "kwargs": {"value": "legacy_test", "resumable": true, "ns": ["legacy:test", "with:complex", "name:space"], "when": "during"}}'
    legacy_interrupt_id = "e69356a9ee3630ee7f4f597f2693000c"

    interrupt = serializer.loads_typed(("json", legacy_interrupt_bytes))
    assert interrupt.id == legacy_interrupt_id
    assert interrupt.value == "legacy_test"

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/test_interruption.py
```py
import pytest
from langgraph.checkpoint.base import BaseCheckpointSaver
from typing_extensions import TypedDict

from langgraph.graph import END, START, StateGraph
from langgraph.types import Durability

pytestmark = pytest.mark.anyio


def test_interruption_without_state_updates(
    sync_checkpointer: BaseCheckpointSaver, durability: Durability
) -> None:
    """Test interruption without state updates. This test confirms that
    interrupting doesn't require a state key having been updated in the prev step"""

    class State(TypedDict):
        input: str

    def noop(_state):
        pass

    builder = StateGraph(State)
    builder.add_node("step_1", noop)
    builder.add_node("step_2", noop)
    builder.add_node("step_3", noop)
    builder.add_edge(START, "step_1")
    builder.add_edge("step_1", "step_2")
    builder.add_edge("step_2", "step_3")
    builder.add_edge("step_3", END)

    graph = builder.compile(checkpointer=sync_checkpointer, interrupt_after="*")

    initial_input = {"input": "hello world"}
    thread = {"configurable": {"thread_id": "1"}}

    graph.invoke(initial_input, thread, durability=durability)
    assert graph.get_state(thread).next == ("step_2",)
    n_checkpoints = len([c for c in graph.get_state_history(thread)])
    assert n_checkpoints == (3 if durability != "exit" else 1)

    graph.invoke(None, thread, durability=durability)
    assert graph.get_state(thread).next == ("step_3",)
    n_checkpoints = len([c for c in graph.get_state_history(thread)])
    assert n_checkpoints == (4 if durability != "exit" else 2)

    graph.invoke(None, thread, durability=durability)
    assert graph.get_state(thread).next == ()
    n_checkpoints = len([c for c in graph.get_state_history(thread)])
    assert n_checkpoints == (5 if durability != "exit" else 3)


async def test_interruption_without_state_updates_async(
    async_checkpointer: BaseCheckpointSaver, durability: Durability
) -> None:
    """Test interruption without state updates. This test confirms that
    interrupting doesn't require a state key having been updated in the prev step"""

    class State(TypedDict):
        input: str

    async def noop(_state):
        pass

    builder = StateGraph(State)
    builder.add_node("step_1", noop)
    builder.add_node("step_2", noop)
    builder.add_node("step_3", noop)
    builder.add_edge(START, "step_1")
    builder.add_edge("step_1", "step_2")
    builder.add_edge("step_2", "step_3")
    builder.add_edge("step_3", END)

    graph = builder.compile(checkpointer=async_checkpointer, interrupt_after="*")

    initial_input = {"input": "hello world"}
    thread = {"configurable": {"thread_id": "1"}}

    await graph.ainvoke(initial_input, thread, durability=durability)
    assert (await graph.aget_state(thread)).next == ("step_2",)
    n_checkpoints = len([c async for c in graph.aget_state_history(thread)])
    assert n_checkpoints == (3 if durability != "exit" else 1)

    await graph.ainvoke(None, thread, durability=durability)
    assert (await graph.aget_state(thread)).next == ("step_3",)
    n_checkpoints = len([c async for c in graph.aget_state_history(thread)])
    assert n_checkpoints == (4 if durability != "exit" else 2)

    await graph.ainvoke(None, thread, durability=durability)
    assert (await graph.aget_state(thread)).next == ()
    n_checkpoints = len([c async for c in graph.aget_state_history(thread)])
    assert n_checkpoints == (5 if durability != "exit" else 3)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/test_large_cases.py
```py
import json
import operator
import re
import time
from dataclasses import replace
from typing import Annotated, Any, Literal, cast

import pytest
from langchain_core.messages import AIMessage, AnyMessage, ToolCall
from langchain_core.runnables import RunnableConfig, RunnableMap, RunnablePick
from langchain_core.tools import tool
from langgraph.checkpoint.base import BaseCheckpointSaver
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.prebuilt.chat_agent_executor import create_react_agent
from langgraph.prebuilt.tool_node import ToolNode
from pytest_mock import MockerFixture
from syrupy import SnapshotAssertion
from typing_extensions import TypedDict

from langgraph._internal._constants import PULL, PUSH
from langgraph.channels.last_value import LastValue
from langgraph.channels.untracked_value import UntrackedValue
from langgraph.constants import END, START
from langgraph.graph import StateGraph
from langgraph.graph.message import MessagesState, add_messages
from langgraph.pregel import NodeBuilder, Pregel
from langgraph.types import (
    Command,
    Durability,
    Interrupt,
    PregelTask,
    RetryPolicy,
    Send,
    StateSnapshot,
    StreamWriter,
    interrupt,
)
from tests.agents import AgentAction, AgentFinish
from tests.any_int import AnyInt
from tests.any_str import AnyDict, AnyStr, UnsortedSequence
from tests.fake_chat import FakeChatModel
from tests.fake_tracer import FakeTracer
from tests.messages import (
    _AnyIdAIMessage,
    _AnyIdAIMessageChunk,
    _AnyIdHumanMessage,
    _AnyIdToolMessage,
)


def test_invoke_two_processes_in_out_interrupt(
    sync_checkpointer: BaseCheckpointSaver,
    mocker: MockerFixture,
) -> None:
    add_one = mocker.Mock(side_effect=lambda x: x + 1)
    one = NodeBuilder().subscribe_only("input").do(add_one).write_to("inbox")
    two = NodeBuilder().subscribe_only("inbox").do(add_one).write_to("output")

    app = Pregel(
        nodes={"one": one, "two": two},
        channels={
            "inbox": LastValue(int),
            "output": LastValue(int),
            "input": LastValue(int),
        },
        input_channels="input",
        output_channels="output",
        checkpointer=sync_checkpointer,
        interrupt_after_nodes=["one"],
    )
    thread1 = {"configurable": {"thread_id": "1"}}
    thread2 = {"configurable": {"thread_id": "2"}}

    # start execution, stop at inbox
    assert app.invoke(2, thread1, durability="async") is None

    # inbox == 3
    checkpoint = sync_checkpointer.get(thread1)
    assert checkpoint is not None
    assert checkpoint["channel_values"]["inbox"] == 3

    # resume execution, finish
    assert app.invoke(None, thread1, durability="async") == 4

    # start execution again, stop at inbox
    assert app.invoke(20, thread1, durability="async") is None

    # inbox == 21
    checkpoint = sync_checkpointer.get(thread1)
    assert checkpoint is not None
    assert checkpoint["channel_values"]["inbox"] == 21

    # send a new value in, interrupting the previous execution
    assert app.invoke(3, thread1, durability="async") is None
    assert app.invoke(None, thread1, durability="async") == 5

    # start execution again, stopping at inbox
    assert app.invoke(20, thread2, durability="async") is None

    # inbox == 21
    snapshot = app.get_state(thread2)
    assert snapshot.values["inbox"] == 21
    assert snapshot.next == ("two",)

    # update the state, resume
    app.update_state(thread2, 25, as_node="one")
    assert app.invoke(None, thread2) == 26

    # no pending tasks
    snapshot = app.get_state(thread2)
    assert snapshot.next == ()

    # list history
    history = [c for c in app.get_state_history(thread1)]
    assert len(history) == 8
    assert history == [
        StateSnapshot(
            values={"inbox": 4, "output": 5, "input": 3},
            tasks=(),
            next=(),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "parents": {},
                "source": "loop",
                "step": 6,
            },
            created_at=AnyStr(),
            parent_config=history[1].config,
            interrupts=(),
        ),
        StateSnapshot(
            values={"inbox": 4, "output": 4, "input": 3},
            tasks=(PregelTask(AnyStr(), "two", (PULL, "two"), result={"output": 5}),),
            next=("two",),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "parents": {},
                "source": "loop",
                "step": 5,
            },
            created_at=AnyStr(),
            parent_config=history[2].config,
            interrupts=(),
        ),
        StateSnapshot(
            values={"inbox": 21, "output": 4, "input": 3},
            tasks=(PregelTask(AnyStr(), "one", (PULL, "one"), result={"inbox": 4}),),
            next=("one",),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "parents": {},
                "source": "input",
                "step": 4,
            },
            created_at=AnyStr(),
            parent_config=history[3].config,
            interrupts=(),
        ),
        StateSnapshot(
            values={"inbox": 21, "output": 4, "input": 20},
            tasks=(PregelTask(AnyStr(), "two", (PULL, "two")),),
            next=("two",),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "parents": {},
                "source": "loop",
                "step": 3,
            },
            created_at=AnyStr(),
            parent_config=history[4].config,
            interrupts=(),
        ),
        StateSnapshot(
            values={"inbox": 3, "output": 4, "input": 20},
            tasks=(PregelTask(AnyStr(), "one", (PULL, "one"), result={"inbox": 21}),),
            next=("one",),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "parents": {},
                "source": "input",
                "step": 2,
            },
            created_at=AnyStr(),
            parent_config=history[5].config,
            interrupts=(),
        ),
        StateSnapshot(
            values={"inbox": 3, "output": 4, "input": 2},
            tasks=(),
            next=(),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "parents": {},
                "source": "loop",
                "step": 1,
            },
            created_at=AnyStr(),
            parent_config=history[6].config,
            interrupts=(),
        ),
        StateSnapshot(
            values={"inbox": 3, "input": 2},
            tasks=(PregelTask(AnyStr(), "two", (PULL, "two"), result={"output": 4}),),
            next=("two",),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "parents": {},
                "source": "loop",
                "step": 0,
            },
            created_at=AnyStr(),
            parent_config=history[7].config,
            interrupts=(),
        ),
        StateSnapshot(
            values={"input": 2},
            tasks=(PregelTask(AnyStr(), "one", (PULL, "one"), result={"inbox": 3}),),
            next=("one",),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "parents": {},
                "source": "input",
                "step": -1,
            },
            created_at=AnyStr(),
            parent_config=None,
            interrupts=(),
        ),
    ]

    # re-running from any previous checkpoint should re-run nodes
    assert [c for c in app.stream(None, history[0].config, stream_mode="updates")] == []
    assert [c for c in app.stream(None, history[1].config, stream_mode="updates")] == [
        {"two": {"output": 5}},
    ]
    assert [c for c in app.stream(None, history[2].config, stream_mode="updates")] == [
        {"one": {"inbox": 4}},
        {"__interrupt__": ()},
    ]


def test_fork_always_re_runs_nodes(
    sync_checkpointer: BaseCheckpointSaver, mocker: MockerFixture
) -> None:
    add_one = mocker.Mock(side_effect=lambda _: 1)

    builder = StateGraph(Annotated[int, operator.add])
    builder.add_node("add_one", add_one)
    builder.add_edge(START, "add_one")
    builder.add_conditional_edges("add_one", lambda cnt: "add_one" if cnt < 6 else END)
    graph = builder.compile(checkpointer=sync_checkpointer)

    thread1 = {"configurable": {"thread_id": "1"}}

    # start execution, stop at inbox
    assert [
        *graph.stream(1, thread1, stream_mode=["values", "updates"], durability="async")
    ] == [
        ("values", 1),
        ("updates", {"add_one": 1}),
        ("values", 2),
        ("updates", {"add_one": 1}),
        ("values", 3),
        ("updates", {"add_one": 1}),
        ("values", 4),
        ("updates", {"add_one": 1}),
        ("values", 5),
        ("updates", {"add_one": 1}),
        ("values", 6),
    ]

    # list history
    history = [c for c in graph.get_state_history(thread1)]
    assert history == [
        StateSnapshot(
            values=6,
            tasks=(),
            next=(),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "parents": {},
                "source": "loop",
                "step": 5,
            },
            created_at=AnyStr(),
            parent_config=history[1].config,
            interrupts=(),
        ),
        StateSnapshot(
            values=5,
            tasks=(PregelTask(AnyStr(), "add_one", (PULL, "add_one"), result=1),),
            next=("add_one",),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "parents": {},
                "source": "loop",
                "step": 4,
            },
            created_at=AnyStr(),
            parent_config=history[2].config,
            interrupts=(),
        ),
        StateSnapshot(
            values=4,
            tasks=(PregelTask(AnyStr(), "add_one", (PULL, "add_one"), result=1),),
            next=("add_one",),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "parents": {},
                "source": "loop",
                "step": 3,
            },
            created_at=AnyStr(),
            parent_config=history[3].config,
            interrupts=(),
        ),
        StateSnapshot(
            values=3,
            tasks=(PregelTask(AnyStr(), "add_one", (PULL, "add_one"), result=1),),
            next=("add_one",),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "parents": {},
                "source": "loop",
                "step": 2,
            },
            created_at=AnyStr(),
            parent_config=history[4].config,
            interrupts=(),
        ),
        StateSnapshot(
            values=2,
            tasks=(PregelTask(AnyStr(), "add_one", (PULL, "add_one"), result=1),),
            next=("add_one",),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "parents": {},
                "source": "loop",
                "step": 1,
            },
            created_at=AnyStr(),
            parent_config=history[5].config,
            interrupts=(),
        ),
        StateSnapshot(
            values=1,
            tasks=(PregelTask(AnyStr(), "add_one", (PULL, "add_one"), result=1),),
            next=("add_one",),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "parents": {},
                "source": "loop",
                "step": 0,
            },
            created_at=AnyStr(),
            parent_config=history[6].config,
            interrupts=(),
        ),
        StateSnapshot(
            values=0,
            tasks=(PregelTask(AnyStr(), "__start__", (PULL, "__start__"), result=1),),
            next=("__start__",),
            config={
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            },
            metadata={
                "parents": {},
                "source": "input",
                "step": -1,
            },
            created_at=AnyStr(),
            parent_config=None,
            interrupts=(),
        ),
    ]

    # forking from any previous checkpoint should re-run nodes
    assert [
        c for c in graph.stream(None, history[0].config, stream_mode="updates")
    ] == []
    assert [
        c for c in graph.stream(None, history[1].config, stream_mode="updates")
    ] == [
        {"add_one": 1},
    ]
    assert [
        c for c in graph.stream(None, history[2].config, stream_mode="updates")
    ] == [
        {"add_one": 1},
        {"add_one": 1},
    ]


def test_conditional_state_graph(
    snapshot: SnapshotAssertion,
    sync_checkpointer: BaseCheckpointSaver,
) -> None:
    from langchain_core.language_models.fake import FakeStreamingListLLM
    from langchain_core.prompts import PromptTemplate
    from langchain_core.tools import tool

    class AgentState(TypedDict, total=False):
        input: Annotated[str, UntrackedValue]
        agent_outcome: AgentAction | AgentFinish | None
        intermediate_steps: Annotated[list[tuple[AgentAction, str]], operator.add]

    class ToolState(TypedDict, total=False):
        agent_outcome: AgentAction | AgentFinish

    # Assemble the tools
    @tool()
    def search_api(query: str) -> str:
        """Searches the API for the query."""
        return f"result for {query}"

    tools = [search_api]

    # Construct the agent
    prompt = PromptTemplate.from_template("Hello!")

    llm = FakeStreamingListLLM(
        responses=[
            "tool:search_api:query",
            "tool:search_api:another",
            "finish:answer",
        ]
    )

    def agent_parser(input: str) -> dict[str, AgentAction | AgentFinish]:
        if input.startswith("finish"):
            _, answer = input.split(":")
            return {
                "agent_outcome": AgentFinish(
                    return_values={"answer": answer}, log=input
                )
            }
        else:
            _, tool_name, tool_input = input.split(":")
            return {
                "agent_outcome": AgentAction(
                    tool=tool_name, tool_input=tool_input, log=input
                )
            }

    agent = prompt | llm | agent_parser

    # Define tool execution logic
    def execute_tools(data: ToolState) -> dict:
        # check session in data
        assert "input" not in data
        assert "intermediate_steps" not in data
        # execute the tool
        agent_action: AgentAction = data.pop("agent_outcome")
        observation = {t.name: t for t in tools}[agent_action.tool].invoke(
            agent_action.tool_input
        )
        return {"intermediate_steps": [[agent_action, observation]]}

    # Define decision-making logic
    def should_continue(data: AgentState) -> str:
        # check session in data
        # Logic to decide whether to continue in the loop or exit
        if isinstance(data["agent_outcome"], AgentFinish):
            return "exit"
        else:
            return "continue"

    # Define a new graph
    workflow = StateGraph(AgentState)

    workflow.add_node("agent", agent)
    workflow.add_node("tools", execute_tools, input_schema=ToolState)

    workflow.set_entry_point("agent")

    workflow.add_conditional_edges(
        "agent", should_continue, {"continue": "tools", "exit": END}
    )

    workflow.add_edge("tools", "agent")

    app = workflow.compile()

    if isinstance(sync_checkpointer, InMemorySaver):
        assert json.dumps(app.get_input_jsonschema()) == snapshot
        assert json.dumps(app.get_output_jsonschema()) == snapshot
        assert json.dumps(app.get_graph().to_json(), indent=2) == snapshot
        assert app.get_graph().draw_mermaid(with_styles=False) == snapshot

    assert app.invoke({"input": "what is weather in sf"}) == {
        "input": "what is weather in sf",
        "intermediate_steps": [
            [
                AgentAction(
                    tool="search_api",
                    tool_input="query",
                    log="tool:search_api:query",
                ),
                "result for query",
            ],
            [
                AgentAction(
                    tool="search_api",
                    tool_input="another",
                    log="tool:search_api:another",
                ),
                "result for another",
            ],
        ],
        "agent_outcome": AgentFinish(
            return_values={"answer": "answer"}, log="finish:answer"
        ),
    }

    assert [*app.stream({"input": "what is weather in sf"})] == [
        {
            "agent": {
                "agent_outcome": AgentAction(
                    tool="search_api",
                    tool_input="query",
                    log="tool:search_api:query",
                ),
            }
        },
        {
            "tools": {
                "intermediate_steps": [
                    [
                        AgentAction(
                            tool="search_api",
                            tool_input="query",
                            log="tool:search_api:query",
                        ),
                        "result for query",
                    ]
                ],
            }
        },
        {
            "agent": {
                "agent_outcome": AgentAction(
                    tool="search_api",
                    tool_input="another",
                    log="tool:search_api:another",
                ),
            }
        },
        {
            "tools": {
                "intermediate_steps": [
                    [
                        AgentAction(
                            tool="search_api",
                            tool_input="another",
                            log="tool:search_api:another",
                        ),
                        "result for another",
                    ],
                ],
            }
        },
        {
            "agent": {
                "agent_outcome": AgentFinish(
                    return_values={"answer": "answer"}, log="finish:answer"
                ),
            }
        },
    ]

    # test state get/update methods with interrupt_after

    app_w_interrupt = workflow.compile(
        checkpointer=sync_checkpointer,
        interrupt_after=["agent"],
    )
    config = {"configurable": {"thread_id": "1"}}

    assert [
        c
        for c in app_w_interrupt.stream(
            {"input": "what is weather in sf"}, config, durability="exit"
        )
    ] == [
        {
            "agent": {
                "agent_outcome": AgentAction(
                    tool="search_api",
                    tool_input="query",
                    log="tool:search_api:query",
                ),
            }
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "agent_outcome": AgentAction(
                tool="search_api", tool_input="query", log="tool:search_api:query"
            ),
            "intermediate_steps": [],
        },
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 1,
        },
        parent_config=None,
        interrupts=(),
    )

    app_w_interrupt.update_state(
        config,
        {
            "agent_outcome": AgentAction(
                tool="search_api",
                tool_input="query",
                log="tool:search_api:a different query",
            )
        },
    )

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "agent_outcome": AgentAction(
                tool="search_api",
                tool_input="query",
                log="tool:search_api:a different query",
            ),
            "intermediate_steps": [],
        },
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 2,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    assert [c for c in app_w_interrupt.stream(None, config)] == [
        {
            "tools": {
                "intermediate_steps": [
                    [
                        AgentAction(
                            tool="search_api",
                            tool_input="query",
                            log="tool:search_api:a different query",
                        ),
                        "result for query",
                    ]
                ],
            }
        },
        {
            "agent": {
                "agent_outcome": AgentAction(
                    tool="search_api",
                    tool_input="another",
                    log="tool:search_api:another",
                ),
            }
        },
        {"__interrupt__": ()},
    ]

    app_w_interrupt.update_state(
        config,
        {
            "agent_outcome": AgentFinish(
                return_values={"answer": "a really nice answer"},
                log="finish:a really nice answer",
            )
        },
    )

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "agent_outcome": AgentFinish(
                return_values={"answer": "a really nice answer"},
                log="finish:a really nice answer",
            ),
            "intermediate_steps": [
                [
                    AgentAction(
                        tool="search_api",
                        tool_input="query",
                        log="tool:search_api:a different query",
                    ),
                    "result for query",
                ]
            ],
        },
        tasks=(),
        next=(),
        config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 5,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    # test state get/update methods with interrupt_before

    app_w_interrupt = workflow.compile(
        checkpointer=sync_checkpointer,
        interrupt_before=["tools"],
        debug=True,
    )
    config = {"configurable": {"thread_id": "2"}}
    llm.i = 0  # reset the llm

    assert [
        c
        for c in app_w_interrupt.stream(
            {"input": "what is weather in sf"}, config, durability="exit"
        )
    ] == [
        {
            "agent": {
                "agent_outcome": AgentAction(
                    tool="search_api", tool_input="query", log="tool:search_api:query"
                ),
            }
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "agent_outcome": AgentAction(
                tool="search_api", tool_input="query", log="tool:search_api:query"
            ),
            "intermediate_steps": [],
        },
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 1,
        },
        parent_config=None,
        interrupts=(),
    )

    app_w_interrupt.update_state(
        config,
        {
            "agent_outcome": AgentAction(
                tool="search_api",
                tool_input="query",
                log="tool:search_api:a different query",
            )
        },
    )

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "agent_outcome": AgentAction(
                tool="search_api",
                tool_input="query",
                log="tool:search_api:a different query",
            ),
            "intermediate_steps": [],
        },
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 2,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    assert [c for c in app_w_interrupt.stream(None, config)] == [
        {
            "tools": {
                "intermediate_steps": [
                    [
                        AgentAction(
                            tool="search_api",
                            tool_input="query",
                            log="tool:search_api:a different query",
                        ),
                        "result for query",
                    ]
                ],
            }
        },
        {
            "agent": {
                "agent_outcome": AgentAction(
                    tool="search_api",
                    tool_input="another",
                    log="tool:search_api:another",
                ),
            }
        },
        {"__interrupt__": ()},
    ]

    app_w_interrupt.update_state(
        config,
        {
            "agent_outcome": AgentFinish(
                return_values={"answer": "a really nice answer"},
                log="finish:a really nice answer",
            )
        },
    )

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "agent_outcome": AgentFinish(
                return_values={"answer": "a really nice answer"},
                log="finish:a really nice answer",
            ),
            "intermediate_steps": [
                [
                    AgentAction(
                        tool="search_api",
                        tool_input="query",
                        log="tool:search_api:a different query",
                    ),
                    "result for query",
                ]
            ],
        },
        tasks=(),
        next=(),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 5,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    # test w interrupt before all
    app_w_interrupt = workflow.compile(
        checkpointer=sync_checkpointer,
        interrupt_before="*",
        debug=True,
    )
    config = {"configurable": {"thread_id": "3"}}
    llm.i = 0  # reset the llm

    assert [
        c
        for c in app_w_interrupt.stream(
            {"input": "what is weather in sf"}, config, durability="exit"
        )
    ] == [
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "intermediate_steps": [],
        },
        tasks=(PregelTask(AnyStr(), "agent", (PULL, "agent")),),
        next=("agent",),
        config={
            "configurable": {
                "thread_id": "3",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 0,
        },
        parent_config=None,
        interrupts=(),
    )

    assert [c for c in app_w_interrupt.stream(None, config)] == [
        {
            "agent": {
                "agent_outcome": AgentAction(
                    tool="search_api", tool_input="query", log="tool:search_api:query"
                ),
            }
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "agent_outcome": AgentAction(
                tool="search_api", tool_input="query", log="tool:search_api:query"
            ),
            "intermediate_steps": [],
        },
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "3",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 1,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    assert [c for c in app_w_interrupt.stream(None, config)] == [
        {
            "tools": {
                "intermediate_steps": [
                    [
                        AgentAction(
                            tool="search_api",
                            tool_input="query",
                            log="tool:search_api:query",
                        ),
                        "result for query",
                    ]
                ],
            }
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "agent_outcome": AgentAction(
                tool="search_api", tool_input="query", log="tool:search_api:query"
            ),
            "intermediate_steps": [
                [
                    AgentAction(
                        tool="search_api",
                        tool_input="query",
                        log="tool:search_api:query",
                    ),
                    "result for query",
                ]
            ],
        },
        tasks=(PregelTask(AnyStr(), "agent", (PULL, "agent")),),
        next=("agent",),
        config={
            "configurable": {
                "thread_id": "3",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 2,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    assert [c for c in app_w_interrupt.stream(None, config)] == [
        {
            "agent": {
                "agent_outcome": AgentAction(
                    tool="search_api",
                    tool_input="another",
                    log="tool:search_api:another",
                ),
            }
        },
        {"__interrupt__": ()},
    ]

    # test w interrupt after all
    app_w_interrupt = workflow.compile(
        checkpointer=sync_checkpointer,
        interrupt_after="*",
    )
    config = {"configurable": {"thread_id": "4"}}
    llm.i = 0  # reset the llm

    assert [
        c
        for c in app_w_interrupt.stream(
            {"input": "what is weather in sf"}, config, durability="exit"
        )
    ] == [
        {
            "agent": {
                "agent_outcome": AgentAction(
                    tool="search_api", tool_input="query", log="tool:search_api:query"
                ),
            }
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "agent_outcome": AgentAction(
                tool="search_api", tool_input="query", log="tool:search_api:query"
            ),
            "intermediate_steps": [],
        },
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "4",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 1,
        },
        parent_config=None,
        interrupts=(),
    )

    assert [c for c in app_w_interrupt.stream(None, config)] == [
        {
            "tools": {
                "intermediate_steps": [
                    [
                        AgentAction(
                            tool="search_api",
                            tool_input="query",
                            log="tool:search_api:query",
                        ),
                        "result for query",
                    ]
                ],
            }
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "agent_outcome": AgentAction(
                tool="search_api", tool_input="query", log="tool:search_api:query"
            ),
            "intermediate_steps": [
                [
                    AgentAction(
                        tool="search_api",
                        tool_input="query",
                        log="tool:search_api:query",
                    ),
                    "result for query",
                ]
            ],
        },
        tasks=(PregelTask(AnyStr(), "agent", (PULL, "agent")),),
        next=("agent",),
        config={
            "configurable": {
                "thread_id": "4",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 2,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    assert [c for c in app_w_interrupt.stream(None, config)] == [
        {
            "agent": {
                "agent_outcome": AgentAction(
                    tool="search_api",
                    tool_input="another",
                    log="tool:search_api:another",
                ),
            }
        },
        {"__interrupt__": ()},
    ]


def test_prebuilt_tool_chat(snapshot: SnapshotAssertion) -> None:
    from langchain_core.messages import AIMessage, HumanMessage
    from langchain_core.tools import tool

    @tool()
    def search_api(query: str) -> str:
        """Searches the API for the query."""
        return f"result for {query}"

    tools = [search_api]

    model = FakeChatModel(
        messages=[
            AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "query"},
                    },
                ],
            ),
            AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call234",
                        "name": "search_api",
                        "args": {"query": "another"},
                    },
                    {
                        "id": "tool_call567",
                        "name": "search_api",
                        "args": {"query": "a third one"},
                    },
                ],
            ),
            AIMessage(content="answer"),
        ]
    )

    app = create_react_agent(model, tools)

    assert json.dumps(app.get_input_jsonschema()) == snapshot
    assert json.dumps(app.get_output_jsonschema()) == snapshot
    assert json.dumps(app.get_graph().to_json(), indent=2) == snapshot
    assert app.get_graph().draw_mermaid(with_styles=False) == snapshot

    assert app.invoke(
        {"messages": [HumanMessage(content="what is weather in sf")]}
    ) == {
        "messages": [
            _AnyIdHumanMessage(content="what is weather in sf"),
            _AnyIdAIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "query"},
                    },
                ],
            ),
            _AnyIdToolMessage(
                content="result for query",
                name="search_api",
                tool_call_id="tool_call123",
            ),
            _AnyIdAIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call234",
                        "name": "search_api",
                        "args": {"query": "another"},
                    },
                    {
                        "id": "tool_call567",
                        "name": "search_api",
                        "args": {"query": "a third one"},
                    },
                ],
            ),
            _AnyIdToolMessage(
                content="result for another",
                name="search_api",
                tool_call_id="tool_call234",
            ),
            _AnyIdToolMessage(
                content="result for a third one",
                name="search_api",
                tool_call_id="tool_call567",
                id=AnyStr(),
            ),
            _AnyIdAIMessage(content="answer"),
        ]
    }

    events = [
        c
        for c in app.stream(
            {"messages": [HumanMessage(content="what is weather in sf")]},
            stream_mode="messages",
        )
    ]

    assert events[:3] == [
        (
            _AnyIdAIMessageChunk(
                content="",
                tool_calls=[
                    {
                        "name": "search_api",
                        "args": {"query": "query"},
                        "id": "tool_call123",
                        "type": "tool_call",
                    }
                ],
                tool_call_chunks=[
                    {
                        "name": "search_api",
                        "args": '{"query": "query"}',
                        "id": "tool_call123",
                        "index": None,
                        "type": "tool_call_chunk",
                    }
                ],
                chunk_position="last",
            ),
            {
                "langgraph_step": 1,
                "langgraph_node": "agent",
                "langgraph_triggers": ("branch:to:agent",),
                "langgraph_path": (PULL, "agent"),
                "langgraph_checkpoint_ns": AnyStr("agent:"),
                "checkpoint_ns": AnyStr("agent:"),
                "ls_provider": "fakechatmodel",
                "ls_model_type": "chat",
            },
        ),
        (
            _AnyIdToolMessage(
                content="result for query",
                name="search_api",
                tool_call_id="tool_call123",
            ),
            {
                "langgraph_step": 2,
                "langgraph_node": "tools",
                "langgraph_triggers": (PUSH,),
                "langgraph_path": (PUSH, AnyInt(), False),
                "langgraph_checkpoint_ns": AnyStr("tools:"),
            },
        ),
        (
            _AnyIdAIMessageChunk(
                content="",
                tool_calls=[
                    {
                        "name": "search_api",
                        "args": {"query": "another"},
                        "id": "tool_call234",
                        "type": "tool_call",
                    },
                    {
                        "name": "search_api",
                        "args": {"query": "a third one"},
                        "id": "tool_call567",
                        "type": "tool_call",
                    },
                ],
                tool_call_chunks=[
                    {
                        "name": "search_api",
                        "args": '{"query": "another"}',
                        "id": "tool_call234",
                        "index": None,
                        "type": "tool_call_chunk",
                    },
                    {
                        "name": "search_api",
                        "args": '{"query": "a third one"}',
                        "id": "tool_call567",
                        "index": None,
                        "type": "tool_call_chunk",
                    },
                ],
                chunk_position="last",
            ),
            {
                "langgraph_step": 3,
                "langgraph_node": "agent",
                "langgraph_triggers": ("branch:to:agent",),
                "langgraph_path": (PULL, "agent"),
                "langgraph_checkpoint_ns": AnyStr("agent:"),
                "checkpoint_ns": AnyStr("agent:"),
                "ls_provider": "fakechatmodel",
                "ls_model_type": "chat",
            },
        ),
    ]

    assert events[3:5] == UnsortedSequence(
        (
            _AnyIdToolMessage(
                content="result for another",
                name="search_api",
                tool_call_id="tool_call234",
            ),
            {
                "langgraph_step": 4,
                "langgraph_node": "tools",
                "langgraph_triggers": (PUSH,),
                "langgraph_path": (PUSH, AnyInt(), False),
                "langgraph_checkpoint_ns": AnyStr("tools:"),
            },
        ),
        (
            _AnyIdToolMessage(
                content="result for a third one",
                name="search_api",
                tool_call_id="tool_call567",
            ),
            {
                "langgraph_step": 4,
                "langgraph_node": "tools",
                "langgraph_triggers": (PUSH,),
                "langgraph_path": (PUSH, AnyInt(), False),
                "langgraph_checkpoint_ns": AnyStr("tools:"),
            },
        ),
    )
    assert events[5:] == [
        (
            _AnyIdAIMessageChunk(
                content="answer",
                chunk_position="last",
            ),
            {
                "langgraph_step": 5,
                "langgraph_node": "agent",
                "langgraph_triggers": ("branch:to:agent",),
                "langgraph_path": (PULL, "agent"),
                "langgraph_checkpoint_ns": AnyStr("agent:"),
                "checkpoint_ns": AnyStr("agent:"),
                "ls_provider": "fakechatmodel",
                "ls_model_type": "chat",
            },
        ),
    ]

    assert app.invoke(
        {"messages": [HumanMessage(content="what is weather in sf")]},
        {"recursion_limit": 2},
        debug=True,
    ) == {
        "messages": [
            _AnyIdHumanMessage(content="what is weather in sf"),
            _AnyIdAIMessage(content="Sorry, need more steps to process this request."),
        ]
    }

    model.i = 0  # reset the model

    invoke_updates_events = app.invoke(
        {"messages": [HumanMessage(content="what is weather in sf")]},
        stream_mode="updates",
    )

    stream_updates_events = [
        *app.stream({"messages": [HumanMessage(content="what is weather in sf")]})
    ]

    for output in (invoke_updates_events, stream_updates_events):
        assert output[:3] == [
            {
                "agent": {
                    "messages": [
                        _AnyIdAIMessage(
                            content="",
                            tool_calls=[
                                {
                                    "id": "tool_call123",
                                    "name": "search_api",
                                    "args": {"query": "query"},
                                },
                            ],
                        )
                    ]
                }
            },
            {
                "tools": {
                    "messages": [
                        _AnyIdToolMessage(
                            content="result for query",
                            name="search_api",
                            tool_call_id="tool_call123",
                        )
                    ]
                }
            },
            {
                "agent": {
                    "messages": [
                        _AnyIdAIMessage(
                            content="",
                            tool_calls=[
                                {
                                    "id": "tool_call234",
                                    "name": "search_api",
                                    "args": {"query": "another"},
                                },
                                {
                                    "id": "tool_call567",
                                    "name": "search_api",
                                    "args": {"query": "a third one"},
                                },
                            ],
                        )
                    ]
                }
            },
        ]
        assert output[3:5] == UnsortedSequence(
            {
                "tools": {
                    "messages": [
                        _AnyIdToolMessage(
                            content="result for another",
                            name="search_api",
                            tool_call_id="tool_call234",
                        ),
                    ]
                }
            },
            {
                "tools": {
                    "messages": [
                        _AnyIdToolMessage(
                            content="result for a third one",
                            name="search_api",
                            tool_call_id="tool_call567",
                        ),
                    ]
                }
            },
        )
        assert output[5:] == [
            {"agent": {"messages": [_AnyIdAIMessage(content="answer")]}}
        ]


def test_state_graph_packets(
    sync_checkpointer: BaseCheckpointSaver, mocker: MockerFixture
) -> None:
    from langchain_core.language_models.fake_chat_models import (
        FakeMessagesListChatModel,
    )
    from langchain_core.messages import (
        AIMessage,
        BaseMessage,
        HumanMessage,
        ToolCall,
        ToolMessage,
    )
    from langchain_core.tools import tool

    class AgentState(TypedDict):
        messages: Annotated[list[BaseMessage], add_messages]

    @tool()
    def search_api(query: str) -> str:
        """Searches the API for the query."""
        time.sleep(0.1)
        return f"result for {query}"

    tools = [search_api]
    tools_by_name = {t.name: t for t in tools}

    model = FakeMessagesListChatModel(
        responses=[
            AIMessage(
                id="ai1",
                content="",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "query"},
                    },
                ],
            ),
            AIMessage(
                id="ai2",
                content="",
                tool_calls=[
                    {
                        "id": "tool_call234",
                        "name": "search_api",
                        "args": {"query": "another", "idx": 0},
                    },
                    {
                        "id": "tool_call567",
                        "name": "search_api",
                        "args": {"query": "a third one", "idx": 1},
                    },
                ],
            ),
            AIMessage(id="ai3", content="answer"),
        ]
    )

    def agent(data: AgentState) -> AgentState:
        return {
            "messages": model.invoke(data["messages"]),
            "something_extra": "hi there",
        }

    # Define decision-making logic
    def should_continue(data: dict) -> str:
        assert data["something_extra"] == "hi there", (
            "nodes can pass extra data to their cond edges, which isn't saved in state"
        )
        # Logic to decide whether to continue in the loop or exit
        if tool_calls := data["messages"][-1].tool_calls:
            return [Send("tools", tool_call) for tool_call in tool_calls]
        else:
            return END

    def tools_node(input: ToolCall, config: RunnableConfig) -> AgentState:
        time.sleep(input["args"].get("idx", 0) / 10)
        output = tools_by_name[input["name"]].invoke(input["args"], config)
        return {
            "messages": ToolMessage(
                content=output, name=input["name"], tool_call_id=input["id"]
            )
        }

    # Define a new graph
    workflow = StateGraph(AgentState)

    # Define the two nodes we will cycle between
    workflow.add_node("agent", agent)
    workflow.add_node("tools", tools_node)

    # Set the entrypoint as `agent`
    # This means that this node is the first one called
    workflow.set_entry_point("agent")

    # We now add a conditional edge
    workflow.add_conditional_edges("agent", should_continue)

    # We now add a normal edge from `tools` to `agent`.
    # This means that after `tools` is called, `agent` node is called next.
    workflow.add_edge("tools", "agent")

    # Finally, we compile it!
    # This compiles it into a LangChain Runnable,
    # meaning you can use it as you would any other runnable
    app = workflow.compile()

    assert app.invoke({"messages": HumanMessage(content="what is weather in sf")}) == {
        "messages": [
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                id="ai1",
                content="",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "query"},
                    },
                ],
            ),
            _AnyIdToolMessage(
                content="result for query",
                name="search_api",
                tool_call_id="tool_call123",
            ),
            AIMessage(
                id="ai2",
                content="",
                tool_calls=[
                    {
                        "id": "tool_call234",
                        "name": "search_api",
                        "args": {"query": "another", "idx": 0},
                    },
                    {
                        "id": "tool_call567",
                        "name": "search_api",
                        "args": {"query": "a third one", "idx": 1},
                    },
                ],
            ),
            _AnyIdToolMessage(
                content="result for another",
                name="search_api",
                tool_call_id="tool_call234",
            ),
            _AnyIdToolMessage(
                content="result for a third one",
                name="search_api",
                tool_call_id="tool_call567",
            ),
            AIMessage(content="answer", id="ai3"),
        ]
    }

    assert [
        c
        for c in app.stream(
            {"messages": [HumanMessage(content="what is weather in sf")]}
        )
    ] == [
        {
            "agent": {
                "messages": AIMessage(
                    id="ai1",
                    content="",
                    tool_calls=[
                        {
                            "id": "tool_call123",
                            "name": "search_api",
                            "args": {"query": "query"},
                        },
                    ],
                )
            },
        },
        {
            "tools": {
                "messages": _AnyIdToolMessage(
                    content="result for query",
                    name="search_api",
                    tool_call_id="tool_call123",
                )
            }
        },
        {
            "agent": {
                "messages": AIMessage(
                    id="ai2",
                    content="",
                    tool_calls=[
                        {
                            "id": "tool_call234",
                            "name": "search_api",
                            "args": {"query": "another", "idx": 0},
                        },
                        {
                            "id": "tool_call567",
                            "name": "search_api",
                            "args": {"query": "a third one", "idx": 1},
                        },
                    ],
                )
            }
        },
        {
            "tools": {
                "messages": _AnyIdToolMessage(
                    content="result for another",
                    name="search_api",
                    tool_call_id="tool_call234",
                )
            },
        },
        {
            "tools": {
                "messages": _AnyIdToolMessage(
                    content="result for a third one",
                    name="search_api",
                    tool_call_id="tool_call567",
                ),
            },
        },
        {"agent": {"messages": AIMessage(content="answer", id="ai3")}},
    ]

    # interrupt after agent

    app_w_interrupt = workflow.compile(
        checkpointer=sync_checkpointer,
        interrupt_after=["agent"],
    )
    config = {"configurable": {"thread_id": "1"}}

    assert [
        c
        for c in app_w_interrupt.stream(
            {"messages": HumanMessage(content="what is weather in sf")},
            config,
            durability="exit",
        )
    ] == [
        {
            "agent": {
                "messages": AIMessage(
                    id="ai1",
                    content="",
                    tool_calls=[
                        {
                            "id": "tool_call123",
                            "name": "search_api",
                            "args": {"query": "query"},
                        },
                    ],
                )
            }
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "messages": [
                _AnyIdHumanMessage(content="what is weather in sf"),
                AIMessage(
                    id="ai1",
                    content="",
                    tool_calls=[
                        {
                            "id": "tool_call123",
                            "name": "search_api",
                            "args": {"query": "query"},
                        },
                    ],
                ),
            ]
        },
        tasks=(PregelTask(AnyStr(), "tools", (PUSH, 0, False)),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 1,
        },
        parent_config=None,
        interrupts=(),
    )

    # modify ai message
    last_message = (app_w_interrupt.get_state(config)).values["messages"][-1]
    last_message.tool_calls[0]["args"]["query"] = "a different query"
    app_w_interrupt.update_state(
        config, {"messages": last_message, "something_extra": "hi there"}
    )

    # message was replaced instead of appended
    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "messages": [
                _AnyIdHumanMessage(content="what is weather in sf"),
                AIMessage(
                    id="ai1",
                    content="",
                    tool_calls=[
                        {
                            "id": "tool_call123",
                            "name": "search_api",
                            "args": {"query": "a different query"},
                        },
                    ],
                ),
            ]
        },
        tasks=(PregelTask(AnyStr(), "tools", (PUSH, 0, False)),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 2,
        },
        parent_config=(
            [*app_w_interrupt.checkpointer.list(config, limit=2)][-1].config
        ),
        interrupts=(),
    )

    assert [c for c in app_w_interrupt.stream(None, config)] == [
        {
            "tools": {
                "messages": _AnyIdToolMessage(
                    content="result for a different query",
                    name="search_api",
                    tool_call_id="tool_call123",
                )
            }
        },
        {
            "agent": {
                "messages": AIMessage(
                    id="ai2",
                    content="",
                    tool_calls=[
                        {
                            "id": "tool_call234",
                            "name": "search_api",
                            "args": {"query": "another", "idx": 0},
                        },
                        {
                            "id": "tool_call567",
                            "name": "search_api",
                            "args": {"query": "a third one", "idx": 1},
                        },
                    ],
                )
            },
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "messages": [
                _AnyIdHumanMessage(content="what is weather in sf"),
                AIMessage(
                    id="ai1",
                    content="",
                    tool_calls=[
                        {
                            "id": "tool_call123",
                            "name": "search_api",
                            "args": {"query": "a different query"},
                        },
                    ],
                ),
                _AnyIdToolMessage(
                    content="result for a different query",
                    name="search_api",
                    tool_call_id="tool_call123",
                ),
                AIMessage(
                    id="ai2",
                    content="",
                    tool_calls=[
                        {
                            "id": "tool_call234",
                            "name": "search_api",
                            "args": {"query": "another", "idx": 0},
                        },
                        {
                            "id": "tool_call567",
                            "name": "search_api",
                            "args": {"query": "a third one", "idx": 1},
                        },
                    ],
                ),
            ]
        },
        tasks=(
            PregelTask(AnyStr(), "tools", (PUSH, 0, False)),
            PregelTask(AnyStr(), "tools", (PUSH, 1, False)),
        ),
        next=("tools", "tools"),
        config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 4,
        },
        parent_config=(
            [*app_w_interrupt.checkpointer.list(config, limit=2)][-1].config
        ),
        interrupts=(),
    )

    app_w_interrupt.update_state(
        config,
        {
            "messages": AIMessage(content="answer", id="ai2"),
            "something_extra": "hi there",
        },
    )

    # replaces message even if object identity is different, as long as id is the same
    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "messages": [
                _AnyIdHumanMessage(content="what is weather in sf"),
                AIMessage(
                    id="ai1",
                    content="",
                    tool_calls=[
                        {
                            "id": "tool_call123",
                            "name": "search_api",
                            "args": {"query": "a different query"},
                        },
                    ],
                ),
                _AnyIdToolMessage(
                    content="result for a different query",
                    name="search_api",
                    tool_call_id="tool_call123",
                ),
                AIMessage(content="answer", id="ai2"),
            ]
        },
        tasks=(),
        next=(),
        config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 5,
        },
        parent_config=(
            [*app_w_interrupt.checkpointer.list(config, limit=2)][-1].config
        ),
        interrupts=(),
    )

    # interrupt before tools

    app_w_interrupt = workflow.compile(
        checkpointer=sync_checkpointer,
        interrupt_before=["tools"],
    )
    config = {"configurable": {"thread_id": "2"}}
    model.i = 0

    assert [
        c
        for c in app_w_interrupt.stream(
            {"messages": HumanMessage(content="what is weather in sf")},
            config,
            durability="exit",
        )
    ] == [
        {
            "agent": {
                "messages": AIMessage(
                    id="ai1",
                    content="",
                    tool_calls=[
                        {
                            "id": "tool_call123",
                            "name": "search_api",
                            "args": {"query": "query"},
                        },
                    ],
                )
            }
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "messages": [
                _AnyIdHumanMessage(content="what is weather in sf"),
                AIMessage(
                    id="ai1",
                    content="",
                    tool_calls=[
                        {
                            "id": "tool_call123",
                            "name": "search_api",
                            "args": {"query": "query"},
                        },
                    ],
                ),
            ]
        },
        tasks=(PregelTask(AnyStr(), "tools", (PUSH, 0, False)),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 1,
        },
        parent_config=None,
        interrupts=(),
    )

    # modify ai message
    last_message = (app_w_interrupt.get_state(config)).values["messages"][-1]
    last_message.tool_calls[0]["args"]["query"] = "a different query"
    app_w_interrupt.update_state(
        config, {"messages": last_message, "something_extra": "hi there"}
    )

    # message was replaced instead of appended
    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "messages": [
                _AnyIdHumanMessage(content="what is weather in sf"),
                AIMessage(
                    id="ai1",
                    content="",
                    tool_calls=[
                        {
                            "id": "tool_call123",
                            "name": "search_api",
                            "args": {"query": "a different query"},
                        },
                    ],
                ),
            ]
        },
        tasks=(PregelTask(AnyStr(), "tools", (PUSH, 0, False)),),
        next=("tools",),
        config=app_w_interrupt.checkpointer.get_tuple(config).config,
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 2,
        },
        parent_config=(
            [*app_w_interrupt.checkpointer.list(config, limit=2)][-1].config
        ),
        interrupts=(),
    )

    assert [c for c in app_w_interrupt.stream(None, config)] == [
        {
            "tools": {
                "messages": _AnyIdToolMessage(
                    content="result for a different query",
                    name="search_api",
                    tool_call_id="tool_call123",
                )
            }
        },
        {
            "agent": {
                "messages": AIMessage(
                    id="ai2",
                    content="",
                    tool_calls=[
                        {
                            "id": "tool_call234",
                            "name": "search_api",
                            "args": {"query": "another", "idx": 0},
                        },
                        {
                            "id": "tool_call567",
                            "name": "search_api",
                            "args": {"query": "a third one", "idx": 1},
                        },
                    ],
                )
            },
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "messages": [
                _AnyIdHumanMessage(content="what is weather in sf"),
                AIMessage(
                    id="ai1",
                    content="",
                    tool_calls=[
                        {
                            "id": "tool_call123",
                            "name": "search_api",
                            "args": {"query": "a different query"},
                        },
                    ],
                ),
                _AnyIdToolMessage(
                    content="result for a different query",
                    name="search_api",
                    tool_call_id="tool_call123",
                ),
                AIMessage(
                    id="ai2",
                    content="",
                    tool_calls=[
                        {
                            "id": "tool_call234",
                            "name": "search_api",
                            "args": {"query": "another", "idx": 0},
                        },
                        {
                            "id": "tool_call567",
                            "name": "search_api",
                            "args": {"query": "a third one", "idx": 1},
                        },
                    ],
                ),
            ]
        },
        tasks=(
            PregelTask(AnyStr(), "tools", (PUSH, 0, False)),
            PregelTask(AnyStr(), "tools", (PUSH, 1, False)),
        ),
        next=("tools", "tools"),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 4,
        },
        parent_config=(
            [*app_w_interrupt.checkpointer.list(config, limit=2)][-1].config
        ),
        interrupts=(),
    )

    app_w_interrupt.update_state(
        config,
        {
            "messages": AIMessage(content="answer", id="ai2"),
            "something_extra": "hi there",
        },
    )

    # replaces message even if object identity is different, as long as id is the same
    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values={
            "messages": [
                _AnyIdHumanMessage(content="what is weather in sf"),
                AIMessage(
                    id="ai1",
                    content="",
                    tool_calls=[
                        {
                            "id": "tool_call123",
                            "name": "search_api",
                            "args": {"query": "a different query"},
                        },
                    ],
                ),
                _AnyIdToolMessage(
                    content="result for a different query",
                    name="search_api",
                    tool_call_id="tool_call123",
                ),
                AIMessage(content="answer", id="ai2"),
            ]
        },
        tasks=(),
        next=(),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 5,
        },
        parent_config=(
            [*app_w_interrupt.checkpointer.list(config, limit=2)][-1].config
        ),
        interrupts=(),
    )


def test_message_graph(
    snapshot: SnapshotAssertion,
    deterministic_uuids: MockerFixture,
    sync_checkpointer: BaseCheckpointSaver,
) -> None:
    from copy import deepcopy

    from langchain_core.callbacks import CallbackManagerForLLMRun
    from langchain_core.language_models.fake_chat_models import (
        FakeMessagesListChatModel,
    )
    from langchain_core.messages import AIMessage, BaseMessage, HumanMessage
    from langchain_core.outputs import ChatGeneration, ChatResult
    from langchain_core.tools import tool

    class FakeFunctionChatModel(FakeMessagesListChatModel):
        def bind_functions(self, functions: list):
            return self

        def _generate(
            self,
            messages: list[BaseMessage],
            stop: list[str] | None = None,
            run_manager: CallbackManagerForLLMRun | None = None,
            **kwargs: Any,
        ) -> ChatResult:
            response = deepcopy(self.responses[self.i])
            if self.i < len(self.responses) - 1:
                self.i += 1
            else:
                self.i = 0
            generation = ChatGeneration(message=response)
            return ChatResult(generations=[generation])

    @tool()
    def search_api(query: str) -> str:
        """Searches the API for the query."""
        return f"result for {query}"

    tools = [search_api]

    model = FakeFunctionChatModel(
        responses=[
            AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "query"},
                    }
                ],
                id="ai1",
            ),
            AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call456",
                        "name": "search_api",
                        "args": {"query": "another"},
                    }
                ],
                id="ai2",
            ),
            AIMessage(content="answer", id="ai3"),
        ]
    )

    # Define the function that determines whether to continue or not
    def should_continue(messages):
        last_message = messages[-1]
        # If there is no function call, then we finish
        if not last_message.tool_calls:
            return "end"
        # Otherwise if there is, we continue
        else:
            return "continue"

    # Define a new graph
    workflow = StateGraph(state_schema=Annotated[list[AnyMessage], add_messages])  # type: ignore[arg-type]

    # Define the two nodes we will cycle between
    workflow.add_node("agent", model)
    workflow.add_node("tools", ToolNode(tools))

    # Set the entrypoint as `agent`
    # This means that this node is the first one called
    workflow.set_entry_point("agent")

    # We now add a conditional edge
    workflow.add_conditional_edges(
        # First, we define the start node. We use `agent`.
        # This means these are the edges taken after the `agent` node is called.
        "agent",
        # Next, we pass in the function that will determine which node is called next.
        should_continue,
        # Finally we pass in a mapping.
        # The keys are strings, and the values are other nodes.
        # END is a special node marking that the graph should finish.
        # What will happen is we will call `should_continue`, and then the output of that
        # will be matched against the keys in this mapping.
        # Based on which one it matches, that node will then be called.
        {
            # If `tools`, then we call the tool node.
            "continue": "tools",
            # Otherwise we finish.
            "end": END,
        },
    )

    # We now add a normal edge from `tools` to `agent`.
    # This means that after `tools` is called, `agent` node is called next.
    workflow.add_edge("tools", "agent")

    # Finally, we compile it!
    # This compiles it into a LangChain Runnable,
    # meaning you can use it as you would any other runnable
    app = workflow.compile()

    if isinstance(sync_checkpointer, InMemorySaver):
        assert json.dumps(app.get_input_jsonschema()) == snapshot
        assert json.dumps(app.get_output_jsonschema()) == snapshot
        assert json.dumps(app.get_graph().to_json(), indent=2) == snapshot
        assert app.get_graph().draw_mermaid(with_styles=False) == snapshot

    assert app.invoke([HumanMessage(content="what is weather in sf")]) == [
        _AnyIdHumanMessage(
            content="what is weather in sf",
        ),
        AIMessage(
            content="",
            tool_calls=[
                {
                    "id": "tool_call123",
                    "name": "search_api",
                    "args": {"query": "query"},
                }
            ],
            id="ai1",  # respects ids passed in
        ),
        _AnyIdToolMessage(
            content="result for query",
            name="search_api",
            tool_call_id="tool_call123",
        ),
        AIMessage(
            content="",
            tool_calls=[
                {
                    "id": "tool_call456",
                    "name": "search_api",
                    "args": {"query": "another"},
                }
            ],
            id="ai2",
        ),
        _AnyIdToolMessage(
            content="result for another",
            name="search_api",
            tool_call_id="tool_call456",
        ),
        AIMessage(content="answer", id="ai3"),
    ]

    assert [*app.stream([HumanMessage(content="what is weather in sf")])] == [
        {
            "agent": AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "query"},
                    }
                ],
                id="ai1",
            )
        },
        {
            "tools": [
                _AnyIdToolMessage(
                    content="result for query",
                    name="search_api",
                    tool_call_id="tool_call123",
                )
            ]
        },
        {
            "agent": AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call456",
                        "name": "search_api",
                        "args": {"query": "another"},
                    }
                ],
                id="ai2",
            )
        },
        {
            "tools": [
                _AnyIdToolMessage(
                    content="result for another",
                    name="search_api",
                    tool_call_id="tool_call456",
                )
            ]
        },
        {"agent": AIMessage(content="answer", id="ai3")},
    ]

    app_w_interrupt = workflow.compile(
        checkpointer=sync_checkpointer,
        interrupt_after=["agent"],
    )
    config = {"configurable": {"thread_id": "1"}}

    assert [
        c
        for c in app_w_interrupt.stream(
            ("human", "what is weather in sf"), config, durability="exit"
        )
    ] == [
        {
            "agent": AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "query"},
                    }
                ],
                id="ai1",
            )
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "query"},
                    }
                ],
                id="ai1",
            ),
        ],
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 1,
        },
        parent_config=None,
        interrupts=(),
    )

    # modify ai message
    last_message = app_w_interrupt.get_state(config).values[-1]
    last_message.tool_calls[0]["args"] = {"query": "a different query"}
    next_config = app_w_interrupt.update_state(config, last_message)

    # message was replaced instead of appended
    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                id="ai1",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "a different query"},
                    }
                ],
            ),
        ],
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config=next_config,
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 2,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    assert [c for c in app_w_interrupt.stream(None, config)] == [
        {
            "tools": [
                _AnyIdToolMessage(
                    content="result for a different query",
                    name="search_api",
                    tool_call_id="tool_call123",
                )
            ]
        },
        {
            "agent": AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call456",
                        "name": "search_api",
                        "args": {"query": "another"},
                    }
                ],
                id="ai2",
            )
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                id="ai1",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "a different query"},
                    }
                ],
            ),
            _AnyIdToolMessage(
                content="result for a different query",
                name="search_api",
                tool_call_id="tool_call123",
            ),
            AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call456",
                        "name": "search_api",
                        "args": {"query": "another"},
                    }
                ],
                id="ai2",
            ),
        ],
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 4,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    app_w_interrupt.update_state(
        config,
        AIMessage(content="answer", id="ai2"),  # replace existing message
    )

    # replaces message even if object identity is different, as long as id is the same
    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                id="ai1",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "a different query"},
                    }
                ],
            ),
            _AnyIdToolMessage(
                content="result for a different query",
                name="search_api",
                tool_call_id="tool_call123",
            ),
            AIMessage(content="answer", id="ai2"),
        ],
        tasks=(),
        next=(),
        config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 5,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    app_w_interrupt = workflow.compile(
        checkpointer=sync_checkpointer,
        interrupt_before=["tools"],
    )
    config = {"configurable": {"thread_id": "2"}}
    model.i = 0  # reset the llm

    assert [
        c
        for c in app_w_interrupt.stream(
            "what is weather in sf", config, durability="exit"
        )
    ] == [
        {
            "agent": AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "query"},
                    }
                ],
                id="ai1",
            )
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "query"},
                    }
                ],
                id="ai1",
            ),
        ],
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 1,
        },
        parent_config=None,
        interrupts=(),
    )

    # modify ai message
    last_message = app_w_interrupt.get_state(config).values[-1]
    last_message.tool_calls[0]["args"] = {"query": "a different query"}
    app_w_interrupt.update_state(config, last_message)

    # message was replaced instead of appended
    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                id="ai1",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "a different query"},
                    }
                ],
            ),
        ],
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 2,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    assert [c for c in app_w_interrupt.stream(None, config)] == [
        {
            "tools": [
                _AnyIdToolMessage(
                    content="result for a different query",
                    name="search_api",
                    tool_call_id="tool_call123",
                )
            ]
        },
        {
            "agent": AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call456",
                        "name": "search_api",
                        "args": {"query": "another"},
                    }
                ],
                id="ai2",
            )
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                id="ai1",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "a different query"},
                    }
                ],
            ),
            _AnyIdToolMessage(
                content="result for a different query",
                name="search_api",
                tool_call_id="tool_call123",
            ),
            AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call456",
                        "name": "search_api",
                        "args": {"query": "another"},
                    }
                ],
                id="ai2",
            ),
        ],
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 4,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    app_w_interrupt.update_state(
        config,
        AIMessage(content="answer", id="ai2"),
    )

    # replaces message even if object identity is different, as long as id is the same
    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                id="ai1",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "a different query"},
                    }
                ],
            ),
            _AnyIdToolMessage(
                content="result for a different query",
                name="search_api",
                tool_call_id="tool_call123",
                id=AnyStr(),
            ),
            AIMessage(content="answer", id="ai2"),
        ],
        tasks=(),
        next=(),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 5,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    # add an extra message as if it came from "tools" node
    app_w_interrupt.update_state(config, ("ai", "an extra message"), as_node="tools")

    # extra message is coerced BaseMessge and appended
    # now the next node is "agent" per the graph edges
    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                id="ai1",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "a different query"},
                    }
                ],
            ),
            _AnyIdToolMessage(
                content="result for a different query",
                name="search_api",
                tool_call_id="tool_call123",
                id=AnyStr(),
            ),
            AIMessage(content="answer", id="ai2"),
            _AnyIdAIMessage(content="an extra message"),
        ],
        tasks=(PregelTask(AnyStr(), "agent", (PULL, "agent")),),
        next=("agent",),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 6,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )


def test_root_graph(
    deterministic_uuids: MockerFixture,
    sync_checkpointer: BaseCheckpointSaver,
) -> None:
    from copy import deepcopy

    from langchain_core.callbacks import CallbackManagerForLLMRun
    from langchain_core.language_models.fake_chat_models import (
        FakeMessagesListChatModel,
    )
    from langchain_core.messages import (
        AIMessage,
        BaseMessage,
        HumanMessage,
        ToolMessage,
    )
    from langchain_core.outputs import ChatGeneration, ChatResult
    from langchain_core.tools import tool

    class FakeFunctionChatModel(FakeMessagesListChatModel):
        def bind_functions(self, functions: list):
            return self

        def _generate(
            self,
            messages: list[BaseMessage],
            stop: list[str] | None = None,
            run_manager: CallbackManagerForLLMRun | None = None,
            **kwargs: Any,
        ) -> ChatResult:
            response = deepcopy(self.responses[self.i])
            if self.i < len(self.responses) - 1:
                self.i += 1
            else:
                self.i = 0
            generation = ChatGeneration(message=response)
            return ChatResult(generations=[generation])

    @tool()
    def search_api(query: str) -> str:
        """Searches the API for the query."""
        return f"result for {query}"

    tools = [search_api]

    model = FakeFunctionChatModel(
        responses=[
            AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "query"},
                    }
                ],
                id="ai1",
            ),
            AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call456",
                        "name": "search_api",
                        "args": {"query": "another"},
                    }
                ],
                id="ai2",
            ),
            AIMessage(content="answer", id="ai3"),
        ]
    )

    # Define the function that determines whether to continue or not
    def should_continue(messages):
        last_message = messages[-1]
        # If there is no function call, then we finish
        if not last_message.tool_calls:
            return "end"
        # Otherwise if there is, we continue
        else:
            return "continue"

    class State(TypedDict):
        __root__: Annotated[list[BaseMessage], add_messages]

    # Define a new graph
    workflow = StateGraph(State)

    # Define the two nodes we will cycle between
    workflow.add_node("agent", model)
    workflow.add_node("tools", ToolNode(tools))

    # Set the entrypoint as `agent`
    # This means that this node is the first one called
    workflow.set_entry_point("agent")

    # We now add a conditional edge
    workflow.add_conditional_edges(
        # First, we define the start node. We use `agent`.
        # This means these are the edges taken after the `agent` node is called.
        "agent",
        # Next, we pass in the function that will determine which node is called next.
        should_continue,
        # Finally we pass in a mapping.
        # The keys are strings, and the values are other nodes.
        # END is a special node marking that the graph should finish.
        # What will happen is we will call `should_continue`, and then the output of that
        # will be matched against the keys in this mapping.
        # Based on which one it matches, that node will then be called.
        {
            # If `tools`, then we call the tool node.
            "continue": "tools",
            # Otherwise we finish.
            "end": END,
        },
    )

    # We now add a normal edge from `tools` to `agent`.
    # This means that after `tools` is called, `agent` node is called next.
    workflow.add_edge("tools", "agent")

    # Finally, we compile it!
    # This compiles it into a LangChain Runnable,
    # meaning you can use it as you would any other runnable
    app = workflow.compile()

    assert app.invoke(HumanMessage(content="what is weather in sf")) == [
        _AnyIdHumanMessage(
            content="what is weather in sf",
        ),
        AIMessage(
            content="",
            tool_calls=[
                {
                    "id": "tool_call123",
                    "name": "search_api",
                    "args": {"query": "query"},
                }
            ],
            id="ai1",  # respects ids passed in
        ),
        _AnyIdToolMessage(
            content="result for query",
            name="search_api",
            tool_call_id="tool_call123",
        ),
        AIMessage(
            content="",
            tool_calls=[
                {
                    "id": "tool_call456",
                    "name": "search_api",
                    "args": {"query": "another"},
                }
            ],
            id="ai2",
        ),
        _AnyIdToolMessage(
            content="result for another",
            name="search_api",
            tool_call_id="tool_call456",
        ),
        AIMessage(content="answer", id="ai3"),
    ]

    assert [*app.stream([HumanMessage(content="what is weather in sf")])] == [
        {
            "agent": AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "query"},
                    }
                ],
                id="ai1",
            )
        },
        {
            "tools": [
                ToolMessage(
                    content="result for query",
                    name="search_api",
                    tool_call_id="tool_call123",
                    id="00000000-0000-4000-8000-000000000024",
                )
            ]
        },
        {
            "agent": AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call456",
                        "name": "search_api",
                        "args": {"query": "another"},
                    }
                ],
                id="ai2",
            )
        },
        {
            "tools": [
                ToolMessage(
                    content="result for another",
                    name="search_api",
                    tool_call_id="tool_call456",
                    id="00000000-0000-4000-8000-000000000030",
                )
            ]
        },
        {"agent": AIMessage(content="answer", id="ai3")},
    ]

    app_w_interrupt = workflow.compile(
        checkpointer=sync_checkpointer,
        interrupt_after=["agent"],
    )
    config = {"configurable": {"thread_id": "1"}}

    assert [
        c
        for c in app_w_interrupt.stream(
            ("human", "what is weather in sf"), config, durability="exit"
        )
    ] == [
        {
            "agent": AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "query"},
                    }
                ],
                id="ai1",
            )
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "query"},
                    }
                ],
                id="ai1",
            ),
        ],
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 1,
        },
        parent_config=None,
        interrupts=(),
    )

    # modify ai message
    last_message = app_w_interrupt.get_state(config).values[-1]
    last_message.tool_calls[0]["args"] = {"query": "a different query"}
    next_config = app_w_interrupt.update_state(config, last_message)

    # message was replaced instead of appended
    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                id="ai1",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "a different query"},
                    }
                ],
            ),
        ],
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config=next_config,
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 2,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    assert [c for c in app_w_interrupt.stream(None, config)] == [
        {
            "tools": [
                _AnyIdToolMessage(
                    content="result for a different query",
                    name="search_api",
                    tool_call_id="tool_call123",
                )
            ]
        },
        {
            "agent": AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call456",
                        "name": "search_api",
                        "args": {"query": "another"},
                    }
                ],
                id="ai2",
            )
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                id="ai1",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "a different query"},
                    }
                ],
            ),
            _AnyIdToolMessage(
                content="result for a different query",
                name="search_api",
                tool_call_id="tool_call123",
                id=AnyStr(),
            ),
            AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call456",
                        "name": "search_api",
                        "args": {"query": "another"},
                    }
                ],
                id="ai2",
            ),
        ],
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 4,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    app_w_interrupt.update_state(
        config,
        AIMessage(content="answer", id="ai2"),  # replace existing message
    )

    # replaces message even if object identity is different, as long as id is the same
    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                id="ai1",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "a different query"},
                    }
                ],
            ),
            _AnyIdToolMessage(
                content="result for a different query",
                name="search_api",
                tool_call_id="tool_call123",
                id=AnyStr(),
            ),
            AIMessage(content="answer", id="ai2"),
        ],
        tasks=(),
        next=(),
        config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 5,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    app_w_interrupt = workflow.compile(
        checkpointer=sync_checkpointer,
        interrupt_before=["tools"],
    )
    config = {"configurable": {"thread_id": "2"}}
    model.i = 0  # reset the llm

    assert [
        c
        for c in app_w_interrupt.stream(
            "what is weather in sf", config, durability="exit"
        )
    ] == [
        {
            "agent": AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "query"},
                    }
                ],
                id="ai1",
            )
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "query"},
                    }
                ],
                id="ai1",
            ),
        ],
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 1,
        },
        parent_config=None,
        interrupts=(),
    )

    # modify ai message
    last_message = app_w_interrupt.get_state(config).values[-1]
    last_message.tool_calls[0]["args"] = {"query": "a different query"}
    app_w_interrupt.update_state(config, last_message)

    # message was replaced instead of appended
    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                id="ai1",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "a different query"},
                    }
                ],
            ),
        ],
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 2,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    assert [c for c in app_w_interrupt.stream(None, config)] == [
        {
            "tools": [
                _AnyIdToolMessage(
                    content="result for a different query",
                    name="search_api",
                    tool_call_id="tool_call123",
                )
            ]
        },
        {
            "agent": AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call456",
                        "name": "search_api",
                        "args": {"query": "another"},
                    }
                ],
                id="ai2",
            )
        },
        {"__interrupt__": ()},
    ]

    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                id="ai1",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "a different query"},
                    }
                ],
            ),
            _AnyIdToolMessage(
                content="result for a different query",
                name="search_api",
                tool_call_id="tool_call123",
                id=AnyStr(),
            ),
            AIMessage(
                content="",
                tool_calls=[
                    {
                        "id": "tool_call456",
                        "name": "search_api",
                        "args": {"query": "another"},
                    }
                ],
                id="ai2",
            ),
        ],
        tasks=(PregelTask(AnyStr(), "tools", (PULL, "tools")),),
        next=("tools",),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "loop",
            "step": 4,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    app_w_interrupt.update_state(
        config,
        AIMessage(content="answer", id="ai2"),
    )

    # replaces message even if object identity is different, as long as id is the same
    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                id="ai1",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "a different query"},
                    }
                ],
            ),
            _AnyIdToolMessage(
                content="result for a different query",
                name="search_api",
                tool_call_id="tool_call123",
            ),
            AIMessage(content="answer", id="ai2"),
        ],
        tasks=(),
        next=(),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 5,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    # add an extra message as if it came from "tools" node
    app_w_interrupt.update_state(config, ("ai", "an extra message"), as_node="tools")

    # extra message is coerced BaseMessge and appended
    # now the next node is "agent" per the graph edges
    assert app_w_interrupt.get_state(config) == StateSnapshot(
        values=[
            _AnyIdHumanMessage(content="what is weather in sf"),
            AIMessage(
                content="",
                id="ai1",
                tool_calls=[
                    {
                        "id": "tool_call123",
                        "name": "search_api",
                        "args": {"query": "a different query"},
                    }
                ],
            ),
            _AnyIdToolMessage(
                content="result for a different query",
                name="search_api",
                tool_call_id="tool_call123",
                id=AnyStr(),
            ),
            AIMessage(content="answer", id="ai2"),
            _AnyIdAIMessage(content="an extra message"),
        ],
        tasks=(PregelTask(AnyStr(), "agent", (PULL, "agent")),),
        next=("agent",),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 6,
        },
        parent_config=(
            list(app_w_interrupt.checkpointer.list(config, limit=2))[-1].config
        ),
        interrupts=(),
    )

    # create new graph with one more state key, reuse previous thread history

    def simple_add(left, right):
        if not isinstance(right, list):
            right = [right]
        return left + right

    class MoreState(TypedDict):
        __root__: Annotated[list[BaseMessage], simple_add]
        something_else: str

    # Define a new graph
    new_workflow = StateGraph(MoreState)
    new_workflow.add_node(
        "agent", RunnableMap(__root__=RunnablePick("__root__") | model)
    )
    new_workflow.add_node(
        "tools", RunnableMap(__root__=RunnablePick("__root__") | ToolNode(tools))
    )
    new_workflow.set_entry_point("agent")
    new_workflow.add_conditional_edges(
        "agent",
        RunnablePick("__root__") | should_continue,
        {
            # If `tools`, then we call the tool node.
            "continue": "tools",
            # Otherwise we finish.
            "end": END,
        },
    )
    new_workflow.add_edge("tools", "agent")
    new_app = new_workflow.compile(checkpointer=sync_checkpointer)
    model.i = 0  # reset the llm

    # previous state is converted to new schema
    assert new_app.get_state(config) == StateSnapshot(
        values={
            "__root__": [
                _AnyIdHumanMessage(content="what is weather in sf"),
                AIMessage(
                    content="",
                    id="ai1",
                    tool_calls=[
                        {
                            "id": "tool_call123",
                            "name": "search_api",
                            "args": {"query": "a different query"},
                        }
                    ],
                ),
                _AnyIdToolMessage(
                    content="result for a different query",
                    name="search_api",
                    tool_call_id="tool_call123",
                ),
                AIMessage(content="answer", id="ai2"),
                _AnyIdAIMessage(content="an extra message"),
            ]
        },
        tasks=(PregelTask(AnyStr(), "agent", (PULL, "agent")),),
        next=("agent",),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        created_at=AnyStr(),
        metadata={
            "parents": {},
            "source": "update",
            "step": 6,
        },
        parent_config=(list(new_app.checkpointer.list(config, limit=2))[-1].config),
        interrupts=(),
    )

    # new input is merged to old state
    assert new_app.invoke(
        {
            "__root__": [HumanMessage(content="what is weather in la")],
            "something_else": "value",
        },
        config,
        interrupt_before=["agent"],
    ) == {
        "__root__": [
            HumanMessage(
                content="what is weather in sf",
                id="00000000-0000-4000-8000-000000000051",
            ),
            AIMessage(
                content="",
                id="ai1",
                tool_calls=[
                    {
                        "name": "search_api",
                        "args": {"query": "a different query"},
                        "id": "tool_call123",
                    }
                ],
            ),
            _AnyIdToolMessage(
                content="result for a different query",
                name="search_api",
                tool_call_id="tool_call123",
            ),
            AIMessage(content="answer", id="ai2"),
            AIMessage(
                content="an extra message", id="00000000-0000-4000-8000-000000000066"
            ),
            HumanMessage(content="what is weather in la"),
        ],
        "something_else": "value",
    }


def test_in_one_fan_out_out_one_graph_state() -> None:
    def sorted_add(x: list[str], y: list[str]) -> list[str]:
        return sorted(operator.add(x, y))

    class State(TypedDict, total=False):
        query: str
        answer: str
        docs: Annotated[list[str], sorted_add]

    def rewrite_query(data: State) -> State:
        return {"query": f"query: {data['query']}"}

    def retriever_one(data: State) -> State:
        # timer ensures stream output order is stable
        # also, it confirms that the update order is not dependent on finishing order
        # instead being defined by the order of the nodes/edges in the graph definition
        # ie. stable between invocations
        time.sleep(0.1)
        return {"docs": ["doc1", "doc2"]}

    def retriever_two(data: State) -> State:
        return {"docs": ["doc3", "doc4"]}

    def qa(data: State) -> State:
        return {"answer": ",".join(data["docs"])}

    workflow = StateGraph(State)

    workflow.add_node("rewrite_query", rewrite_query)
    workflow.add_node("retriever_one", retriever_one)
    workflow.add_node("retriever_two", retriever_two)
    workflow.add_node("qa", qa)

    workflow.set_entry_point("rewrite_query")
    workflow.add_edge("rewrite_query", "retriever_one")
    workflow.add_edge("rewrite_query", "retriever_two")
    workflow.add_edge("retriever_one", "qa")
    workflow.add_edge("retriever_two", "qa")
    workflow.set_finish_point("qa")

    app = workflow.compile()

    assert app.invoke({"query": "what is weather in sf"}) == {
        "query": "query: what is weather in sf",
        "docs": ["doc1", "doc2", "doc3", "doc4"],
        "answer": "doc1,doc2,doc3,doc4",
    }

    assert [*app.stream({"query": "what is weather in sf"})] == [
        {"rewrite_query": {"query": "query: what is weather in sf"}},
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {"qa": {"answer": "doc1,doc2,doc3,doc4"}},
    ]

    assert [*app.stream({"query": "what is weather in sf"}, stream_mode="values")] == [
        {"query": "what is weather in sf", "docs": []},
        {"query": "query: what is weather in sf", "docs": []},
        {
            "query": "query: what is weather in sf",
            "docs": ["doc1", "doc2", "doc3", "doc4"],
        },
        {
            "query": "query: what is weather in sf",
            "docs": ["doc1", "doc2", "doc3", "doc4"],
            "answer": "doc1,doc2,doc3,doc4",
        },
    ]

    assert [
        *app.stream(
            {"query": "what is weather in sf"},
            stream_mode=["values", "updates", "debug"],
        )
    ] == [
        ("values", {"query": "what is weather in sf", "docs": []}),
        (
            "debug",
            {
                "type": "task",
                "timestamp": AnyStr(),
                "step": 1,
                "payload": {
                    "id": AnyStr(),
                    "name": "rewrite_query",
                    "input": {"query": "what is weather in sf", "docs": []},
                    "triggers": ("branch:to:rewrite_query",),
                },
            },
        ),
        ("updates", {"rewrite_query": {"query": "query: what is weather in sf"}}),
        (
            "debug",
            {
                "type": "task_result",
                "timestamp": AnyStr(),
                "step": 1,
                "payload": {
                    "id": AnyStr(),
                    "name": "rewrite_query",
                    "result": {
                        "query": "query: what is weather in sf",
                    },
                    "error": None,
                    "interrupts": [],
                },
            },
        ),
        ("values", {"query": "query: what is weather in sf", "docs": []}),
        (
            "debug",
            {
                "type": "task",
                "timestamp": AnyStr(),
                "step": 2,
                "payload": {
                    "id": AnyStr(),
                    "name": "retriever_one",
                    "input": {"query": "query: what is weather in sf", "docs": []},
                    "triggers": ("branch:to:retriever_one",),
                },
            },
        ),
        (
            "debug",
            {
                "type": "task",
