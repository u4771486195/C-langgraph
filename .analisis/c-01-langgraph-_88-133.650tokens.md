        """Replace the runtime with a new runtime with the given overrides."""
        return replace(self, **overrides)


DEFAULT_RUNTIME = Runtime(
    context=None,
    store=None,
    stream_writer=_no_op_stream_writer,
    previous=None,
)


def get_runtime(context_schema: type[ContextT] | None = None) -> Runtime[ContextT]:
    """Get the runtime for the current graph run.

    Args:
        context_schema: Optional schema used for type hinting the return type of the runtime.

    Returns:
        The runtime for the current graph run.
    """

    # TODO: in an ideal world, we would have a context manager for
    # the runtime that's independent of the config. this will follow
    # from the removal of the configurable packing
    runtime = cast(Runtime[ContextT], get_config()[CONF].get(CONFIG_KEY_RUNTIME))
    return runtime

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/types.py
```py
from __future__ import annotations

import sys
from collections import deque
from collections.abc import Callable, Hashable, Sequence
from dataclasses import asdict, dataclass
from typing import (
    TYPE_CHECKING,
    Any,
    ClassVar,
    Generic,
    Literal,
    NamedTuple,
    TypeVar,
    final,
)
from warnings import warn

from langchain_core.runnables import Runnable, RunnableConfig
from langgraph.checkpoint.base import BaseCheckpointSaver, CheckpointMetadata
from typing_extensions import Unpack, deprecated
from xxhash import xxh3_128_hexdigest

from langgraph._internal._cache import default_cache_key
from langgraph._internal._fields import get_cached_annotated_keys, get_update_as_tuples
from langgraph._internal._retry import default_retry_on
from langgraph._internal._typing import MISSING, DeprecatedKwargs
from langgraph.warnings import LangGraphDeprecatedSinceV10

if TYPE_CHECKING:
    from langgraph.pregel.protocol import PregelProtocol


try:
    from langchain_core.messages.tool import ToolOutputMixin
except ImportError:

    class ToolOutputMixin:  # type: ignore[no-redef]
        pass


__all__ = (
    "All",
    "Checkpointer",
    "StreamMode",
    "StreamWriter",
    "RetryPolicy",
    "CachePolicy",
    "Interrupt",
    "StateUpdate",
    "PregelTask",
    "PregelExecutableTask",
    "StateSnapshot",
    "Send",
    "Command",
    "Durability",
    "interrupt",
    "Overwrite",
)

Durability = Literal["sync", "async", "exit"]
"""Durability mode for the graph execution.
- `"sync"`: Changes are persisted synchronously before the next step starts.
- `"async"`: Changes are persisted asynchronously while the next step executes.
- `"exit"`: Changes are persisted only when the graph exits."""

All = Literal["*"]
"""Special value to indicate that graph should interrupt on all nodes."""

Checkpointer = None | bool | BaseCheckpointSaver
"""Type of the checkpointer to use for a subgraph.
- True enables persistent checkpointing for this subgraph.
- False disables checkpointing, even if the parent graph has a checkpointer.
- None inherits checkpointer from the parent graph."""

StreamMode = Literal[
    "values", "updates", "checkpoints", "tasks", "debug", "messages", "custom"
]
"""How the stream method should emit outputs.

- `"values"`: Emit all values in the state after each step, including interrupts.
    When used with functional API, values are emitted once at the end of the workflow.
- `"updates"`: Emit only the node or task names and updates returned by the nodes or tasks after each step.
    If multiple updates are made in the same step (e.g. multiple nodes are run) then those updates are emitted separately.
- `"custom"`: Emit custom data using from inside nodes or tasks using `StreamWriter`.
- `"messages"`: Emit LLM messages token-by-token together with metadata for any LLM invocations inside nodes or tasks.
- `"checkpoints"`: Emit an event when a checkpoint is created, in the same format as returned by `get_state()`.
- `"tasks"`: Emit events when tasks start and finish, including their results and errors.
- `"debug"`: Emit `"checkpoints"` and `"tasks"` events for debugging purposes.
"""

StreamWriter = Callable[[Any], None]
"""`Callable` that accepts a single argument and writes it to the output stream.
Always injected into nodes if requested as a keyword argument, but it's a no-op
when not using `stream_mode="custom"`."""

_DC_KWARGS = {"kw_only": True, "slots": True, "frozen": True}


class RetryPolicy(NamedTuple):
    """Configuration for retrying nodes.

    !!! version-added "Added in version 0.2.24"
    """

    initial_interval: float = 0.5
    """Amount of time that must elapse before the first retry occurs. In seconds."""
    backoff_factor: float = 2.0
    """Multiplier by which the interval increases after each retry."""
    max_interval: float = 128.0
    """Maximum amount of time that may elapse between retries. In seconds."""
    max_attempts: int = 3
    """Maximum number of attempts to make before giving up, including the first."""
    jitter: bool = True
    """Whether to add random jitter to the interval between retries."""
    retry_on: (
        type[Exception] | Sequence[type[Exception]] | Callable[[Exception], bool]
    ) = default_retry_on
    """List of exception classes that should trigger a retry, or a callable that returns `True` for exceptions that should trigger a retry."""


KeyFuncT = TypeVar("KeyFuncT", bound=Callable[..., str | bytes])


@dataclass(**_DC_KWARGS)
class CachePolicy(Generic[KeyFuncT]):
    """Configuration for caching nodes."""

    key_func: KeyFuncT = default_cache_key  # type: ignore[assignment]
    """Function to generate a cache key from the node's input.
    Defaults to hashing the input with pickle."""

    ttl: int | None = None
    """Time to live for the cache entry in seconds. If `None`, the entry never expires."""


_DEFAULT_INTERRUPT_ID = "placeholder-id"


@final
@dataclass(init=False, slots=True)
class Interrupt:
    """Information about an interrupt that occurred in a node.

    !!! version-added "Added in version 0.2.24"

    !!! version-changed "Changed in version v0.4.0"
        * `interrupt_id` was introduced as a property

    !!! version-changed "Changed in version v0.6.0"

        The following attributes have been removed:

        * `ns`
        * `when`
        * `resumable`
        * `interrupt_id`, deprecated in favor of `id`
    """

    value: Any
    """The value associated with the interrupt."""

    id: str
    """The ID of the interrupt. Can be used to resume the interrupt directly."""

    def __init__(
        self,
        value: Any,
        id: str = _DEFAULT_INTERRUPT_ID,
        **deprecated_kwargs: Unpack[DeprecatedKwargs],
    ) -> None:
        self.value = value

        if (
            (ns := deprecated_kwargs.get("ns", MISSING)) is not MISSING
            and (id == _DEFAULT_INTERRUPT_ID)
            and (isinstance(ns, Sequence))
        ):
            self.id = xxh3_128_hexdigest("|".join(ns).encode())
        else:
            self.id = id

    @classmethod
    def from_ns(cls, value: Any, ns: str) -> Interrupt:
        return cls(value=value, id=xxh3_128_hexdigest(ns.encode()))

    @property
    @deprecated("`interrupt_id` is deprecated. Use `id` instead.", category=None)
    def interrupt_id(self) -> str:
        warn(
            "`interrupt_id` is deprecated. Use `id` instead.",
            LangGraphDeprecatedSinceV10,
            stacklevel=2,
        )
        return self.id


class StateUpdate(NamedTuple):
    values: dict[str, Any] | None
    as_node: str | None = None
    task_id: str | None = None


class PregelTask(NamedTuple):
    """A Pregel task."""

    id: str
    name: str
    path: tuple[str | int | tuple, ...]
    error: Exception | None = None
    interrupts: tuple[Interrupt, ...] = ()
    state: None | RunnableConfig | StateSnapshot = None
    result: Any | None = None


if sys.version_info > (3, 11):
    _T_DC_KWARGS = {"weakref_slot": True, "slots": True, "frozen": True}
else:
    _T_DC_KWARGS = {"frozen": True}


class CacheKey(NamedTuple):
    """Cache key for a task."""

    ns: tuple[str, ...]
    """Namespace for the cache entry."""
    key: str
    """Key for the cache entry."""
    ttl: int | None
    """Time to live for the cache entry in seconds."""


@dataclass(**_T_DC_KWARGS)
class PregelExecutableTask:
    name: str
    input: Any
    proc: Runnable
    writes: deque[tuple[str, Any]]
    config: RunnableConfig
    triggers: Sequence[str]
    retry_policy: Sequence[RetryPolicy]
    cache_key: CacheKey | None
    id: str
    path: tuple[str | int | tuple, ...]
    writers: Sequence[Runnable] = ()
    subgraphs: Sequence[PregelProtocol] = ()


class StateSnapshot(NamedTuple):
    """Snapshot of the state of the graph at the beginning of a step."""

    values: dict[str, Any] | Any
    """Current values of channels."""
    next: tuple[str, ...]
    """The name of the node to execute in each task for this step."""
    config: RunnableConfig
    """Config used to fetch this snapshot."""
    metadata: CheckpointMetadata | None
    """Metadata associated with this snapshot."""
    created_at: str | None
    """Timestamp of snapshot creation."""
    parent_config: RunnableConfig | None
    """Config used to fetch the parent snapshot, if any."""
    tasks: tuple[PregelTask, ...]
    """Tasks to execute in this step. If already attempted, may contain an error."""
    interrupts: tuple[Interrupt, ...]
    """Interrupts that occurred in this step that are pending resolution."""


class Send:
    """A message or packet to send to a specific node in the graph.

    The `Send` class is used within a `StateGraph`'s conditional edges to
    dynamically invoke a node with a custom state at the next step.

    Importantly, the sent state can differ from the core graph's state,
    allowing for flexible and dynamic workflow management.

    One such example is a "map-reduce" workflow where your graph invokes
    the same node multiple times in parallel with different states,
    before aggregating the results back into the main graph's state.

    Attributes:
        node (str): The name of the target node to send the message to.
        arg (Any): The state or message to send to the target node.

    !!! example

        ```python
        from typing import Annotated
        from langgraph.types import Send
        from langgraph.graph import END, START
        from langgraph.graph import StateGraph
        import operator

        class OverallState(TypedDict):
            subjects: list[str]
            jokes: Annotated[list[str], operator.add]

        def continue_to_jokes(state: OverallState):
            return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]

        builder = StateGraph(OverallState)
        builder.add_node("generate_joke", lambda state: {"jokes": [f"Joke about {state['subject']}"]})
        builder.add_conditional_edges(START, continue_to_jokes)
        builder.add_edge("generate_joke", END)
        graph = builder.compile()

        # Invoking with two subjects results in a generated joke for each
        graph.invoke({"subjects": ["cats", "dogs"]})
        # {'subjects': ['cats', 'dogs'], 'jokes': ['Joke about cats', 'Joke about dogs']}
        ```
    """

    __slots__ = ("node", "arg")

    node: str
    arg: Any

    def __init__(self, /, node: str, arg: Any) -> None:
        """
        Initialize a new instance of the `Send` class.

        Args:
            node: The name of the target node to send the message to.
            arg: The state or message to send to the target node.
        """
        self.node = node
        self.arg = arg

    def __hash__(self) -> int:
        return hash((self.node, self.arg))

    def __repr__(self) -> str:
        return f"Send(node={self.node!r}, arg={self.arg!r})"

    def __eq__(self, value: object) -> bool:
        return (
            isinstance(value, Send)
            and self.node == value.node
            and self.arg == value.arg
        )


N = TypeVar("N", bound=Hashable)


@dataclass(**_DC_KWARGS)
class Command(Generic[N], ToolOutputMixin):
    """One or more commands to update the graph's state and send messages to nodes.

    Args:
        graph: Graph to send the command to. Supported values are:

            - `None`: the current graph
            - `Command.PARENT`: closest parent graph
        update: Update to apply to the graph's state.
        resume: Value to resume execution with. To be used together with [`interrupt()`][langgraph.types.interrupt].
            Can be one of the following:

            - Mapping of interrupt ids to resume values
            - A single value with which to resume the next interrupt
        goto: Can be one of the following:

            - Name of the node to navigate to next (any node that belongs to the specified `graph`)
            - Sequence of node names to navigate to next
            - `Send` object (to execute a node with the input provided)
            - Sequence of `Send` objects
    """

    graph: str | None = None
    update: Any | None = None
    resume: dict[str, Any] | Any | None = None
    goto: Send | Sequence[Send | N] | N = ()

    def __repr__(self) -> str:
        # get all non-None values
        contents = ", ".join(
            f"{key}={value!r}" for key, value in asdict(self).items() if value
        )
        return f"Command({contents})"

    def _update_as_tuples(self) -> Sequence[tuple[str, Any]]:
        if isinstance(self.update, dict):
            return list(self.update.items())
        elif isinstance(self.update, (list, tuple)) and all(
            isinstance(t, tuple) and len(t) == 2 and isinstance(t[0], str)
            for t in self.update
        ):
            return self.update
        elif keys := get_cached_annotated_keys(type(self.update)):
            return get_update_as_tuples(self.update, keys)
        elif self.update is not None:
            return [("__root__", self.update)]
        else:
            return []

    PARENT: ClassVar[Literal["__parent__"]] = "__parent__"


def interrupt(value: Any) -> Any:
    """Interrupt the graph with a resumable exception from within a node.

    The `interrupt` function enables human-in-the-loop workflows by pausing graph
    execution and surfacing a value to the client. This value can communicate context
    or request input required to resume execution.

    In a given node, the first invocation of this function raises a `GraphInterrupt`
    exception, halting execution. The provided `value` is included with the exception
    and sent to the client executing the graph.

    A client resuming the graph must use the [`Command`][langgraph.types.Command]
    primitive to specify a value for the interrupt and continue execution.
    The graph resumes from the start of the node, **re-executing** all logic.

    If a node contains multiple `interrupt` calls, LangGraph matches resume values
    to interrupts based on their order in the node. This list of resume values
    is scoped to the specific task executing the node and is not shared across tasks.

    To use an `interrupt`, you must enable a checkpointer, as the feature relies
    on persisting the graph state.

    !!! example

        ```python
        import uuid
        from typing import Optional
        from typing_extensions import TypedDict

        from langgraph.checkpoint.memory import InMemorySaver
        from langgraph.constants import START
        from langgraph.graph import StateGraph
        from langgraph.types import interrupt, Command


        class State(TypedDict):
            \"\"\"The graph state.\"\"\"

            foo: str
            human_value: Optional[str]
            \"\"\"Human value will be updated using an interrupt.\"\"\"


        def node(state: State):
            answer = interrupt(
                # This value will be sent to the client
                # as part of the interrupt information.
                \"what is your age?\"
            )
            print(f\"> Received an input from the interrupt: {answer}\")
            return {\"human_value\": answer}


        builder = StateGraph(State)
        builder.add_node(\"node\", node)
        builder.add_edge(START, \"node\")

        # A checkpointer must be enabled for interrupts to work!
        checkpointer = InMemorySaver()
        graph = builder.compile(checkpointer=checkpointer)

        config = {
            \"configurable\": {
                \"thread_id\": uuid.uuid4(),
            }
        }

        for chunk in graph.stream({\"foo\": \"abc\"}, config):
            print(chunk)

        # > {'__interrupt__': (Interrupt(value='what is your age?', id='45fda8478b2ef754419799e10992af06'),)}

        command = Command(resume=\"some input from a human!!!\")

        for chunk in graph.stream(Command(resume=\"some input from a human!!!\"), config):
            print(chunk)

        # > Received an input from the interrupt: some input from a human!!!
        # > {'node': {'human_value': 'some input from a human!!!'}}
        ```

    Args:
        value: The value to surface to the client when the graph is interrupted.

    Returns:
        Any: On subsequent invocations within the same node (same task to be precise), returns the value provided during the first invocation

    Raises:
        GraphInterrupt: On the first invocation within the node, halts execution and surfaces the provided value to the client.
    """
    from langgraph._internal._constants import (
        CONFIG_KEY_CHECKPOINT_NS,
        CONFIG_KEY_SCRATCHPAD,
        CONFIG_KEY_SEND,
        RESUME,
    )
    from langgraph.config import get_config
    from langgraph.errors import GraphInterrupt

    conf = get_config()["configurable"]
    # track interrupt index
    scratchpad = conf[CONFIG_KEY_SCRATCHPAD]
    idx = scratchpad.interrupt_counter()
    # find previous resume values
    if scratchpad.resume:
        if idx < len(scratchpad.resume):
            conf[CONFIG_KEY_SEND]([(RESUME, scratchpad.resume)])
            return scratchpad.resume[idx]
    # find current resume value
    v = scratchpad.get_null_resume(True)
    if v is not None:
        assert len(scratchpad.resume) == idx, (scratchpad.resume, idx)
        scratchpad.resume.append(v)
        conf[CONFIG_KEY_SEND]([(RESUME, scratchpad.resume)])
        return v
    # no resume value found
    raise GraphInterrupt(
        (
            Interrupt.from_ns(
                value=value,
                ns=conf[CONFIG_KEY_CHECKPOINT_NS],
            ),
        )
    )


@dataclass(slots=True)
class Overwrite:
    """Bypass a reducer and write the wrapped value directly to a `BinaryOperatorAggregate` channel.

    Receiving multiple `Overwrite` values for the same channel in a single super-step
    will raise an `InvalidUpdateError`.

    !!! example

        ```python
        from typing import Annotated
        import operator
        from langgraph.graph import StateGraph
        from langgraph.types import Overwrite

        class State(TypedDict):
            messages: Annotated[list, operator.add]

        def node_a(state: TypedDict):
            # Normal update: uses the reducer (operator.add)
            return {"messages": ["a"]}

        def node_b(state: State):
            # Overwrite: bypasses the reducer and replaces the entire value
            return {"messages": Overwrite(value=["b"])}

        builder = StateGraph(State)
        builder.add_node("node_a", node_a)
        builder.add_node("node_b", node_b)
        builder.set_entry_point("node_a")
        builder.add_edge("node_a", "node_b")
        graph = builder.compile()

        # Without Overwrite in node_b, messages would be ["START", "a", "b"]
        # With Overwrite, messages is just ["b"]
        result = graph.invoke({"messages": ["START"]})
        assert result == {"messages": ["b"]}
        ```
    """

    value: Any
    """The value to write directly to the channel, bypassing any reducer."""

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/typing.py
```py
from __future__ import annotations

from typing_extensions import TypeVar

from langgraph._internal._typing import StateLike

__all__ = (
    "StateT",
    "StateT_co",
    "StateT_contra",
    "InputT",
    "OutputT",
    "ContextT",
)

StateT = TypeVar("StateT", bound=StateLike)
"""Type variable used to represent the state in a graph."""

StateT_co = TypeVar("StateT_co", bound=StateLike, covariant=True)

StateT_contra = TypeVar("StateT_contra", bound=StateLike, contravariant=True)

ContextT = TypeVar("ContextT", bound=StateLike | None, default=None)
"""Type variable used to represent graph run scoped context.

Defaults to `None`.
"""

ContextT_contra = TypeVar(
    "ContextT_contra", bound=StateLike | None, contravariant=True, default=None
)

InputT = TypeVar("InputT", bound=StateLike, default=StateT)
"""Type variable used to represent the input to a `StateGraph`.

Defaults to `StateT`.
"""

OutputT = TypeVar("OutputT", bound=StateLike, default=StateT)
"""Type variable used to represent the output of a `StateGraph`.

Defaults to `StateT`.
"""

NodeInputT = TypeVar("NodeInputT", bound=StateLike)
"""Type variable used to represent the input to a node."""

NodeInputT_contra = TypeVar("NodeInputT_contra", bound=StateLike, contravariant=True)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/version.py
```py
"""Exports package version."""

from importlib import metadata

__all__ = ("__version__",)

try:
    __version__ = metadata.version(__package__)
except metadata.PackageNotFoundError:
    # Case where package metadata is not available.
    __version__ = ""
del metadata  # optional, avoids polluting the results of dir(__package__)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/warnings.py
```py
"""LangGraph specific warnings."""

from __future__ import annotations

__all__ = (
    "LangGraphDeprecationWarning",
    "LangGraphDeprecatedSinceV05",
    "LangGraphDeprecatedSinceV10",
)


class LangGraphDeprecationWarning(DeprecationWarning):
    """A LangGraph specific deprecation warning.

    Attributes:
        message: Description of the warning.
        since: LangGraph version in which the deprecation was introduced.
        expected_removal: LangGraph version in what the corresponding functionality expected to be removed.

    Inspired by the Pydantic `PydanticDeprecationWarning` class, which sets a great standard
    for deprecation warnings with clear versioning information.
    """

    message: str
    since: tuple[int, int]
    expected_removal: tuple[int, int]

    def __init__(
        self,
        message: str,
        *args: object,
        since: tuple[int, int],
        expected_removal: tuple[int, int] | None = None,
    ) -> None:
        super().__init__(message, *args)
        self.message = message.rstrip(".")
        self.since = since
        self.expected_removal = (
            expected_removal if expected_removal is not None else (since[0] + 1, 0)
        )

    def __str__(self) -> str:
        message = (
            f"{self.message}. Deprecated in LangGraph V{self.since[0]}.{self.since[1]}"
            f" to be removed in V{self.expected_removal[0]}.{self.expected_removal[1]}."
        )
        return message


class LangGraphDeprecatedSinceV05(LangGraphDeprecationWarning):
    """A specific `LangGraphDeprecationWarning` subclass defining functionality deprecated since LangGraph v0.5.0"""

    def __init__(self, message: str, *args: object) -> None:
        super().__init__(message, *args, since=(0, 5), expected_removal=(2, 0))


class LangGraphDeprecatedSinceV10(LangGraphDeprecationWarning):
    """A specific `LangGraphDeprecationWarning` subclass defining functionality deprecated since LangGraph v1.0.0"""

    def __init__(self, message: str, *args: object) -> None:
        super().__init__(message, *args, since=(1, 0), expected_removal=(2, 0))

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/utils/__init__.py
```py
"""Legacy utilities module, to be removed in v1."""

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/utils/config.py
```py
"""Backwards compat imports for config utilities, to be removed in v1."""

from langgraph._internal._config import ensure_config, patch_configurable  # noqa: F401
from langgraph.config import get_config, get_store  # noqa: F401

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/utils/runnable.py
```py
"""Backwards compat imports for runnable utilities, to be removed in v1."""

from langgraph._internal._runnable import RunnableCallable, RunnableLike  # noqa: F401

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/func/__init__.py
```py
from __future__ import annotations

import functools
import inspect
import warnings
from collections.abc import Awaitable, Callable, Sequence
from dataclasses import dataclass
from typing import (
    Any,
    Generic,
    TypeVar,
    cast,
    get_args,
    get_origin,
    overload,
)

from langgraph.cache.base import BaseCache
from langgraph.checkpoint.base import BaseCheckpointSaver
from langgraph.store.base import BaseStore
from typing_extensions import Unpack

from langgraph._internal._constants import CACHE_NS_WRITES, PREVIOUS
from langgraph._internal._typing import MISSING, DeprecatedKwargs
from langgraph.channels.ephemeral_value import EphemeralValue
from langgraph.channels.last_value import LastValue
from langgraph.constants import END, START
from langgraph.pregel import Pregel
from langgraph.pregel._call import (
    P,
    SyncAsyncFuture,
    T,
    call,
    get_runnable_for_entrypoint,
    identifier,
)
from langgraph.pregel._read import PregelNode
from langgraph.pregel._write import ChannelWrite, ChannelWriteEntry
from langgraph.types import _DC_KWARGS, CachePolicy, RetryPolicy, StreamMode
from langgraph.typing import ContextT
from langgraph.warnings import LangGraphDeprecatedSinceV05, LangGraphDeprecatedSinceV10

__all__ = ("task", "entrypoint")


class _TaskFunction(Generic[P, T]):
    def __init__(
        self,
        func: Callable[P, Awaitable[T]] | Callable[P, T],
        *,
        retry_policy: Sequence[RetryPolicy],
        cache_policy: CachePolicy[Callable[P, str | bytes]] | None = None,
        name: str | None = None,
    ) -> None:
        if name is not None:
            if hasattr(func, "__func__"):
                # handle class methods
                # NOTE: we're modifying the instance method to avoid modifying
                # the original class method in case it's shared across multiple tasks
                instance_method = functools.partial(func.__func__, func.__self__)  # type: ignore [union-attr]
                instance_method.__name__ = name  # type: ignore [attr-defined]
                func = instance_method
            else:
                # handle regular functions / partials / callable classes, etc.
                func.__name__ = name
        self.func = func
        self.retry_policy = retry_policy
        self.cache_policy = cache_policy
        functools.update_wrapper(self, func)

    def __call__(self, *args: P.args, **kwargs: P.kwargs) -> SyncAsyncFuture[T]:
        return call(
            self.func,
            retry_policy=self.retry_policy,
            cache_policy=self.cache_policy,
            *args,
            **kwargs,
        )

    def clear_cache(self, cache: BaseCache) -> None:
        """Clear the cache for this task."""
        if self.cache_policy is not None:
            cache.clear(((CACHE_NS_WRITES, identifier(self.func) or "__dynamic__"),))

    async def aclear_cache(self, cache: BaseCache) -> None:
        """Clear the cache for this task."""
        if self.cache_policy is not None:
            await cache.aclear(
                ((CACHE_NS_WRITES, identifier(self.func) or "__dynamic__"),)
            )


@overload
def task(
    __func_or_none__: None = None,
    *,
    name: str | None = None,
    retry_policy: RetryPolicy | Sequence[RetryPolicy] | None = None,
    cache_policy: CachePolicy[Callable[P, str | bytes]] | None = None,
    **kwargs: Unpack[DeprecatedKwargs],
) -> Callable[
    [Callable[P, Awaitable[T]] | Callable[P, T]],
    _TaskFunction[P, T],
]: ...


@overload
def task(__func_or_none__: Callable[P, Awaitable[T]]) -> _TaskFunction[P, T]: ...


@overload
def task(__func_or_none__: Callable[P, T]) -> _TaskFunction[P, T]: ...


def task(
    __func_or_none__: Callable[P, Awaitable[T]] | Callable[P, T] | None = None,
    *,
    name: str | None = None,
    retry_policy: RetryPolicy | Sequence[RetryPolicy] | None = None,
    cache_policy: CachePolicy[Callable[P, str | bytes]] | None = None,
    **kwargs: Unpack[DeprecatedKwargs],
) -> (
    Callable[[Callable[P, Awaitable[T]] | Callable[P, T]], _TaskFunction[P, T]]
    | _TaskFunction[P, T]
):
    """Define a LangGraph task using the `task` decorator.

    !!! important "Requires python 3.11 or higher for async functions"
        The `task` decorator supports both sync and async functions. To use async
        functions, ensure that you are using Python 3.11 or higher.

    Tasks can only be called from within an [`entrypoint`][langgraph.func.entrypoint] or
    from within a `StateGraph`. A task can be called like a regular function with the
    following differences:

    - When a checkpointer is enabled, the function inputs and outputs must be serializable.
    - The decorated function can only be called from within an entrypoint or `StateGraph`.
    - Calling the function produces a future. This makes it easy to parallelize tasks.

    Args:
        name: An optional name for the task. If not provided, the function name will be used.
        retry_policy: An optional retry policy (or list of policies) to use for the task in case of a failure.
        cache_policy: An optional cache policy to use for the task. This allows caching of the task results.

    Returns:
        A callable function when used as a decorator.

    Example: Sync Task
        ```python
        from langgraph.func import entrypoint, task


        @task
        def add_one_task(a: int) -> int:
            return a + 1


        @entrypoint()
        def add_one(numbers: list[int]) -> list[int]:
            futures = [add_one_task(n) for n in numbers]
            results = [f.result() for f in futures]
            return results


        # Call the entrypoint
        add_one.invoke([1, 2, 3])  # Returns [2, 3, 4]
        ```

    Example: Async Task
        ```python
        import asyncio
        from langgraph.func import entrypoint, task


        @task
        async def add_one_task(a: int) -> int:
            return a + 1


        @entrypoint()
        async def add_one(numbers: list[int]) -> list[int]:
            futures = [add_one_task(n) for n in numbers]
            return asyncio.gather(*futures)


        # Call the entrypoint
        await add_one.ainvoke([1, 2, 3])  # Returns [2, 3, 4]
        ```
    """
    if (retry := kwargs.get("retry", MISSING)) is not MISSING:
        warnings.warn(
            "`retry` is deprecated and will be removed. Please use `retry_policy` instead.",
            category=LangGraphDeprecatedSinceV05,
            stacklevel=2,
        )
        if retry_policy is None:
            retry_policy = retry  # type: ignore[assignment]

    retry_policies: Sequence[RetryPolicy] = (
        ()
        if retry_policy is None
        else (retry_policy,)
        if isinstance(retry_policy, RetryPolicy)
        else retry_policy
    )

    def decorator(
        func: Callable[P, Awaitable[T]] | Callable[P, T],
    ) -> Callable[P, SyncAsyncFuture[T]]:
        return _TaskFunction(
            func, retry_policy=retry_policies, cache_policy=cache_policy, name=name
        )

    if __func_or_none__ is not None:
        return decorator(__func_or_none__)

    return decorator


R = TypeVar("R")
S = TypeVar("S")


# The decorator was wrapped in a class to support the `final` attribute.
# In this form, the `final` attribute should play nicely with IDE autocompletion,
# and type checking tools.
# In addition, we'll be able to surface this information in the API Reference.
class entrypoint(Generic[ContextT]):
    """Define a LangGraph workflow using the `entrypoint` decorator.

    ### Function signature

    The decorated function must accept a **single parameter**, which serves as the input
    to the function. This input parameter can be of any type. Use a dictionary
    to pass **multiple parameters** to the function.

    ### Injectable parameters

    The decorated function can request access to additional parameters
    that will be injected automatically at run time. These parameters include:

    | Parameter        | Description                                                                                          |
    |------------------|------------------------------------------------------------------------------------------------------|
    | **`config`**     | A configuration object (aka `RunnableConfig`) that holds run-time configuration values.              |
    | **`previous`**   | The previous return value for the given thread (available only when a checkpointer is provided).     |
    | **`runtime`**    | A `Runtime` object that contains information about the current run, including context, store, writer |

    The entrypoint decorator can be applied to sync functions or async functions.

    ### State management

    The **`previous`** parameter can be used to access the return value of the previous
    invocation of the entrypoint on the same thread id. This value is only available
    when a checkpointer is provided.

    If you want **`previous`** to be different from the return value, you can use the
    `entrypoint.final` object to return a value while saving a different value to the
    checkpoint.

    Args:
        checkpointer: Specify a checkpointer to create a workflow that can persist
            its state across runs.
        store: A generalized key-value store. Some implementations may support
            semantic search capabilities through an optional `index` configuration.
        cache: A cache to use for caching the results of the workflow.
        context_schema: Specifies the schema for the context object that will be
            passed to the workflow.
        cache_policy: A cache policy to use for caching the results of the workflow.
        retry_policy: A retry policy (or list of policies) to use for the workflow in case of a failure.

    !!! warning "`config_schema` Deprecated"
        The `config_schema` parameter is deprecated in v0.6.0 and support will be removed in v2.0.0.
        Please use `context_schema` instead to specify the schema for run-scoped context.


    Example: Using entrypoint and tasks
        ```python
        import time

        from langgraph.func import entrypoint, task
        from langgraph.types import interrupt, Command
        from langgraph.checkpoint.memory import InMemorySaver

        @task
        def compose_essay(topic: str) -> str:
            time.sleep(1.0)  # Simulate slow operation
            return f"An essay about {topic}"

        @entrypoint(checkpointer=InMemorySaver())
        def review_workflow(topic: str) -> dict:
            \"\"\"Manages the workflow for generating and reviewing an essay.

            The workflow includes:
            1. Generating an essay about the given topic.
            2. Interrupting the workflow for human review of the generated essay.

            Upon resuming the workflow, compose_essay task will not be re-executed
            as its result is cached by the checkpointer.

            Args:
                topic: The subject of the essay.

            Returns:
                dict: A dictionary containing the generated essay and the human review.
            \"\"\"
            essay_future = compose_essay(topic)
            essay = essay_future.result()
            human_review = interrupt({
                \"question\": \"Please provide a review\",
                \"essay\": essay
            })
            return {
                \"essay\": essay,
                \"review\": human_review,
            }

        # Example configuration for the workflow
        config = {
            \"configurable\": {
                \"thread_id\": \"some_thread\"
            }
        }

        # Topic for the essay
        topic = \"cats\"

        # Stream the workflow to generate the essay and await human review
        for result in review_workflow.stream(topic, config):
            print(result)

        # Example human review provided after the interrupt
        human_review = \"This essay is great.\"

        # Resume the workflow with the provided human review
        for result in review_workflow.stream(Command(resume=human_review), config):
            print(result)
        ```

    Example: Accessing the previous return value
        When a checkpointer is enabled the function can access the previous return value
        of the previous invocation on the same thread id.

        ```python
        from typing import Optional

        from langgraph.checkpoint.memory import MemorySaver

        from langgraph.func import entrypoint


        @entrypoint(checkpointer=InMemorySaver())
        def my_workflow(input_data: str, previous: Optional[str] = None) -> str:
            return "world"


        config = {"configurable": {"thread_id": "some_thread"}}
        my_workflow.invoke("hello", config)
        ```

    Example: Using entrypoint.final to save a value
        The `entrypoint.final` object allows you to return a value while saving
        a different value to the checkpoint. This value will be accessible
        in the next invocation of the entrypoint via the `previous` parameter, as
        long as the same thread id is used.

        ```python
        from typing import Any

        from langgraph.checkpoint.memory import MemorySaver

        from langgraph.func import entrypoint


        @entrypoint(checkpointer=InMemorySaver())
        def my_workflow(
            number: int,
            *,
            previous: Any = None,
        ) -> entrypoint.final[int, int]:
            previous = previous or 0
            # This will return the previous value to the caller, saving
            # 2 * number to the checkpoint, which will be used in the next invocation
            # for the `previous` parameter.
            return entrypoint.final(value=previous, save=2 * number)


        config = {"configurable": {"thread_id": "some_thread"}}

        my_workflow.invoke(3, config)  # 0 (previous was None)
        my_workflow.invoke(1, config)  # 6 (previous was 3 * 2 from the previous invocation)
        ```
    """

    def __init__(
        self,
        checkpointer: BaseCheckpointSaver | None = None,
        store: BaseStore | None = None,
        cache: BaseCache | None = None,
        context_schema: type[ContextT] | None = None,
        cache_policy: CachePolicy | None = None,
        retry_policy: RetryPolicy | Sequence[RetryPolicy] | None = None,
        **kwargs: Unpack[DeprecatedKwargs],
    ) -> None:
        """Initialize the entrypoint decorator."""
        if (config_schema := kwargs.get("config_schema", MISSING)) is not MISSING:
            warnings.warn(
                "`config_schema` is deprecated and will be removed. Please use `context_schema` instead.",
                category=LangGraphDeprecatedSinceV10,
                stacklevel=2,
            )
            if context_schema is None:
                context_schema = cast(type[ContextT], config_schema)

        if (retry := kwargs.get("retry", MISSING)) is not MISSING:
            warnings.warn(
                "`retry` is deprecated and will be removed. Please use `retry_policy` instead.",
                category=LangGraphDeprecatedSinceV05,
                stacklevel=2,
            )
            if retry_policy is None:
                retry_policy = cast("RetryPolicy | Sequence[RetryPolicy]", retry)

        self.checkpointer = checkpointer
        self.store = store
        self.cache = cache
        self.cache_policy = cache_policy
        self.retry_policy = retry_policy
        self.context_schema = context_schema

    @dataclass(**_DC_KWARGS)
    class final(Generic[R, S]):
        """A primitive that can be returned from an entrypoint.

        This primitive allows to save a value to the checkpointer distinct from the
        return value from the entrypoint.

        Example: Decoupling the return value and the save value
            ```python
            from langgraph.checkpoint.memory import InMemorySaver
            from langgraph.func import entrypoint


            @entrypoint(checkpointer=InMemorySaver())
            def my_workflow(
                number: int,
                *,
                previous: Any = None,
            ) -> entrypoint.final[int, int]:
                previous = previous or 0
                # This will return the previous value to the caller, saving
                # 2 * number to the checkpoint, which will be used in the next invocation
                # for the `previous` parameter.
                return entrypoint.final(value=previous, save=2 * number)


            config = {"configurable": {"thread_id": "1"}}

            my_workflow.invoke(3, config)  # 0 (previous was None)
            my_workflow.invoke(1, config)  # 6 (previous was 3 * 2 from the previous invocation)
            ```
        """

        value: R
        """Value to return. A value will always be returned even if it is `None`."""
        save: S
        """The value for the state for the next checkpoint.

        A value will always be saved even if it is `None`.
        """

    def __call__(self, func: Callable[..., Any]) -> Pregel:
        """Convert a function into a Pregel graph.

        Args:
            func: The function to convert. Support both sync and async functions.

        Returns:
            A Pregel graph.
        """
        # wrap generators in a function that writes to StreamWriter
        if inspect.isgeneratorfunction(func) or inspect.isasyncgenfunction(func):
            raise NotImplementedError(
                "Generators are not supported in the Functional API."
            )

        bound = get_runnable_for_entrypoint(func)
        stream_mode: StreamMode = "updates"

        # get input and output types
        sig = inspect.signature(func)
        first_parameter_name = next(iter(sig.parameters.keys()), None)
        if not first_parameter_name:
            raise ValueError("Entrypoint function must have at least one parameter")
        input_type = (
            sig.parameters[first_parameter_name].annotation
            if sig.parameters[first_parameter_name].annotation
            is not inspect.Signature.empty
            else Any
        )

        def _pluck_return_value(value: Any) -> Any:
            """Extract the return_ value the entrypoint.final object or passthrough."""
            return value.value if isinstance(value, entrypoint.final) else value

        def _pluck_save_value(value: Any) -> Any:
            """Get save value from the entrypoint.final object or passthrough."""
            return value.save if isinstance(value, entrypoint.final) else value

        output_type, save_type = Any, Any
        if sig.return_annotation is not inspect.Signature.empty:
            # User does not parameterize entrypoint.final properly
            if (
                sig.return_annotation is entrypoint.final
            ):  # Un-parameterized entrypoint.final
                output_type = save_type = Any
            else:
                origin = get_origin(sig.return_annotation)
                if origin is entrypoint.final:
                    type_annotations = get_args(sig.return_annotation)
                    if len(type_annotations) != 2:
                        raise TypeError(
                            "Please an annotation for both the return_ and "
                            "the save values."
                            "For example, `-> entrypoint.final[int, str]` would assign a "
                            "return_ a type of `int` and save the type `str`."
                        )
                    output_type, save_type = get_args(sig.return_annotation)
                else:
                    output_type = save_type = sig.return_annotation

        return Pregel(
            nodes={
                func.__name__: PregelNode(
                    bound=bound,
                    triggers=[START],
                    channels=START,
                    writers=[
                        ChannelWrite(
                            [
                                ChannelWriteEntry(END, mapper=_pluck_return_value),
                                ChannelWriteEntry(PREVIOUS, mapper=_pluck_save_value),
                            ]
                        )
                    ],
                )
            },
            channels={
                START: EphemeralValue(input_type),
                END: LastValue(output_type, END),
                PREVIOUS: LastValue(save_type, PREVIOUS),
            },
            input_channels=START,
            output_channels=END,
            stream_channels=END,
            stream_mode=stream_mode,
            stream_eager=True,
            checkpointer=self.checkpointer,
            store=self.store,
            cache=self.cache,
            cache_policy=self.cache_policy,
            retry_policy=self.retry_policy or (),
            context_schema=self.context_schema,  # type: ignore[arg-type]
        )

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/graph/__init__.py
```py
from langgraph.constants import END, START
from langgraph.graph.message import MessageGraph, MessagesState, add_messages
from langgraph.graph.state import StateGraph

__all__ = (
    "END",
    "START",
    "StateGraph",
    "add_messages",
    "MessagesState",
    "MessageGraph",
)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/graph/_branch.py
```py
from __future__ import annotations

from collections.abc import Awaitable, Callable, Hashable, Sequence
from inspect import (
    isfunction,
    ismethod,
    signature,
)
from itertools import zip_longest
from types import FunctionType
from typing import (
    Any,
    Literal,
    NamedTuple,
    cast,
    get_args,
    get_origin,
    get_type_hints,
)

from langchain_core.runnables import (
    Runnable,
    RunnableConfig,
    RunnableLambda,
)

from langgraph._internal._runnable import (
    RunnableCallable,
)
from langgraph.constants import END, START
from langgraph.errors import InvalidUpdateError
from langgraph.pregel._write import PASSTHROUGH, ChannelWrite, ChannelWriteEntry
from langgraph.types import Send

_Writer = Callable[
    [Sequence[str | Send], bool],
    Sequence[ChannelWriteEntry | Send],
]


def _get_branch_path_input_schema(
    path: Callable[..., Hashable | Sequence[Hashable]]
    | Callable[..., Awaitable[Hashable | Sequence[Hashable]]]
    | Runnable[Any, Hashable | Sequence[Hashable]],
) -> type[Any] | None:
    input = None
    # detect input schema annotation in the branch callable
    try:
        callable_: (
            Callable[..., Hashable | Sequence[Hashable]]
            | Callable[..., Awaitable[Hashable | Sequence[Hashable]]]
            | None
        ) = None
        if isinstance(path, (RunnableCallable, RunnableLambda)):
            if isfunction(path.func) or ismethod(path.func):
                callable_ = path.func
            elif (callable_method := getattr(path.func, "__call__", None)) and ismethod(
                callable_method
            ):
                callable_ = callable_method
            elif isfunction(path.afunc) or ismethod(path.afunc):
                callable_ = path.afunc
            elif (
                callable_method := getattr(path.afunc, "__call__", None)
            ) and ismethod(callable_method):
                callable_ = callable_method
        elif callable(path):
            callable_ = path

        if callable_ is not None and (hints := get_type_hints(callable_)):
            first_parameter_name = next(
                iter(signature(cast(FunctionType, callable_)).parameters.keys())
            )
            if input_hint := hints.get(first_parameter_name):
                if isinstance(input_hint, type) and get_type_hints(input_hint):
                    input = input_hint
    except (TypeError, StopIteration):
        pass

    return input


class BranchSpec(NamedTuple):
    path: Runnable[Any, Hashable | list[Hashable]]
    ends: dict[Hashable, str] | None
    input_schema: type[Any] | None = None

    @classmethod
    def from_path(
        cls,
        path: Runnable[Any, Hashable | list[Hashable]],
        path_map: dict[Hashable, str] | list[str] | None,
        infer_schema: bool = False,
    ) -> BranchSpec:
        # coerce path_map to a dictionary
        path_map_: dict[Hashable, str] | None = None
        try:
            if isinstance(path_map, dict):
                path_map_ = path_map.copy()
            elif isinstance(path_map, list):
                path_map_ = {name: name for name in path_map}
            else:
                # find func
                func: Callable | None = None
                if isinstance(path, (RunnableCallable, RunnableLambda)):
                    func = path.func or path.afunc
                if func is not None:
                    # find callable method
                    if (cal := getattr(path, "__call__", None)) and ismethod(cal):
                        func = cal
                    # get the return type
                    if rtn_type := get_type_hints(func).get("return"):
                        if get_origin(rtn_type) is Literal:
                            path_map_ = {name: name for name in get_args(rtn_type)}
        except Exception:
            pass
        # infer input schema
        input_schema = _get_branch_path_input_schema(path) if infer_schema else None
        # create branch
        return cls(path=path, ends=path_map_, input_schema=input_schema)

    def run(
        self,
        writer: _Writer,
        reader: Callable[[RunnableConfig], Any] | None = None,
    ) -> RunnableCallable:
        return ChannelWrite.register_writer(
            RunnableCallable(
                func=self._route,
                afunc=self._aroute,
                writer=writer,
                reader=reader,
                name=None,
                trace=False,
            ),
            list(
                zip_longest(
                    writer([e for e in self.ends.values()], True),
                    [str(la) for la, e in self.ends.items()],
                )
            )
            if self.ends
            else None,
        )

    def _route(
        self,
        input: Any,
        config: RunnableConfig,
        *,
        reader: Callable[[RunnableConfig], Any] | None,
        writer: _Writer,
    ) -> Runnable:
        if reader:
            value = reader(config)
            # passthrough additional keys from node to branch
            # only doable when using dict states
            if (
                isinstance(value, dict)
                and isinstance(input, dict)
                and self.input_schema is None
            ):
                value = {**input, **value}
        else:
            value = input
        result = self.path.invoke(value, config)
        return self._finish(writer, input, result, config)

    async def _aroute(
        self,
        input: Any,
        config: RunnableConfig,
        *,
        reader: Callable[[RunnableConfig], Any] | None,
        writer: _Writer,
    ) -> Runnable:
        if reader:
            value = reader(config)
            # passthrough additional keys from node to branch
            # only doable when using dict states
            if (
                isinstance(value, dict)
                and isinstance(input, dict)
                and self.input_schema is None
            ):
                value = {**input, **value}
        else:
            value = input
        result = await self.path.ainvoke(value, config)
        return self._finish(writer, input, result, config)

    def _finish(
        self,
        writer: _Writer,
        input: Any,
        result: Any,
        config: RunnableConfig,
    ) -> Runnable | Any:
        if not isinstance(result, (list, tuple)):
            result = [result]
        if self.ends:
            destinations: Sequence[Send | str] = [
                r if isinstance(r, Send) else self.ends[r] for r in result
            ]
        else:
            destinations = cast(Sequence[Send | str], result)
        if any(dest is None or dest == START for dest in destinations):
            raise ValueError("Branch did not return a valid destination")
        if any(p.node == END for p in destinations if isinstance(p, Send)):
            raise InvalidUpdateError("Cannot send a packet to the END node")
        entries = writer(destinations, False)
        if not entries:
            return input
        else:
            need_passthrough = False
            for e in entries:
                if isinstance(e, ChannelWriteEntry):
                    if e.value is PASSTHROUGH:
                        need_passthrough = True
                        break
            if need_passthrough:
                return ChannelWrite(entries)
            else:
                ChannelWrite.do_write(config, entries)
                return input

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/graph/_node.py
```py
from __future__ import annotations

from collections.abc import Sequence
from dataclasses import dataclass
from typing import Any, Generic, Protocol, TypeAlias

from langchain_core.runnables import Runnable, RunnableConfig
from langgraph.store.base import BaseStore

from langgraph._internal._typing import EMPTY_SEQ
from langgraph.runtime import Runtime
from langgraph.types import CachePolicy, RetryPolicy, StreamWriter
from langgraph.typing import ContextT, NodeInputT, NodeInputT_contra


class _Node(Protocol[NodeInputT_contra]):
    def __call__(self, state: NodeInputT_contra) -> Any: ...


class _NodeWithConfig(Protocol[NodeInputT_contra]):
    def __call__(self, state: NodeInputT_contra, config: RunnableConfig) -> Any: ...


class _NodeWithWriter(Protocol[NodeInputT_contra]):
    def __call__(self, state: NodeInputT_contra, *, writer: StreamWriter) -> Any: ...


class _NodeWithStore(Protocol[NodeInputT_contra]):
    def __call__(self, state: NodeInputT_contra, *, store: BaseStore) -> Any: ...


class _NodeWithWriterStore(Protocol[NodeInputT_contra]):
    def __call__(
        self, state: NodeInputT_contra, *, writer: StreamWriter, store: BaseStore
    ) -> Any: ...


class _NodeWithConfigWriter(Protocol[NodeInputT_contra]):
    def __call__(
        self, state: NodeInputT_contra, *, config: RunnableConfig, writer: StreamWriter
    ) -> Any: ...


class _NodeWithConfigStore(Protocol[NodeInputT_contra]):
    def __call__(
        self, state: NodeInputT_contra, *, config: RunnableConfig, store: BaseStore
    ) -> Any: ...


class _NodeWithConfigWriterStore(Protocol[NodeInputT_contra]):
    def __call__(
        self,
        state: NodeInputT_contra,
        *,
        config: RunnableConfig,
        writer: StreamWriter,
        store: BaseStore,
    ) -> Any: ...


class _NodeWithRuntime(Protocol[NodeInputT_contra, ContextT]):
    def __call__(
        self, state: NodeInputT_contra, *, runtime: Runtime[ContextT]
    ) -> Any: ...


# TODO: we probably don't want to explicitly support the config / store signatures once
# we move to adding a context arg. Maybe what we do is we add support for kwargs with param spec
# this is purely for typing purposes though, so can easily change in the coming weeks.
StateNode: TypeAlias = (
    _Node[NodeInputT]
    | _NodeWithConfig[NodeInputT]
    | _NodeWithWriter[NodeInputT]
    | _NodeWithStore[NodeInputT]
    | _NodeWithWriterStore[NodeInputT]
    | _NodeWithConfigWriter[NodeInputT]
    | _NodeWithConfigStore[NodeInputT]
    | _NodeWithConfigWriterStore[NodeInputT]
    | _NodeWithRuntime[NodeInputT, ContextT]
    | Runnable[NodeInputT, Any]
)


@dataclass(slots=True)
class StateNodeSpec(Generic[NodeInputT, ContextT]):
    runnable: StateNode[NodeInputT, ContextT]
    metadata: dict[str, Any] | None
    input_schema: type[NodeInputT]
    retry_policy: RetryPolicy | Sequence[RetryPolicy] | None
    cache_policy: CachePolicy | None
    ends: tuple[str, ...] | dict[str, str] | None = EMPTY_SEQ
    defer: bool = False

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/graph/message.py
```py
from __future__ import annotations

import uuid
import warnings
from collections.abc import Callable, Sequence
from functools import partial
from typing import (
    Annotated,
    Any,
    Literal,
    cast,
)

from langchain_core.messages import (
    AnyMessage,
    BaseMessage,
    BaseMessageChunk,
    MessageLikeRepresentation,
    RemoveMessage,
    convert_to_messages,
    message_chunk_to_message,
)
from typing_extensions import TypedDict, deprecated

from langgraph._internal._constants import CONF, CONFIG_KEY_SEND, NS_SEP
from langgraph.graph.state import StateGraph
from langgraph.warnings import LangGraphDeprecatedSinceV10

__all__ = (
    "add_messages",
    "MessagesState",
    "MessageGraph",
    "REMOVE_ALL_MESSAGES",
)

Messages = list[MessageLikeRepresentation] | MessageLikeRepresentation

REMOVE_ALL_MESSAGES = "__remove_all__"


def _add_messages_wrapper(func: Callable) -> Callable[[Messages, Messages], Messages]:
    def _add_messages(
        left: Messages | None = None, right: Messages | None = None, **kwargs: Any
    ) -> Messages | Callable[[Messages, Messages], Messages]:
        if left is not None and right is not None:
            return func(left, right, **kwargs)
        elif left is not None or right is not None:
            msg = (
                f"Must specify non-null arguments for both 'left' and 'right'. Only "
                f"received: '{'left' if left else 'right'}'."
            )
            raise ValueError(msg)
        else:
            return partial(func, **kwargs)

    _add_messages.__doc__ = func.__doc__
    return cast(Callable[[Messages, Messages], Messages], _add_messages)


@_add_messages_wrapper
def add_messages(
    left: Messages,
    right: Messages,
    *,
    format: Literal["langchain-openai"] | None = None,
) -> Messages:
    """Merges two lists of messages, updating existing messages by ID.

    By default, this ensures the state is "append-only", unless the
    new message has the same ID as an existing message.

    Args:
        left: The base list of `Messages`.
        right: The list of `Messages` (or single `Message`) to merge
            into the base list.
        format: The format to return messages in. If `None` then `Messages` will be
            returned as is. If `langchain-openai` then `Messages` will be returned as
            `BaseMessage` objects with their contents formatted to match OpenAI message
            format, meaning contents can be string, `'text'` blocks, or `'image_url'` blocks
            and tool responses are returned as their own `ToolMessage` objects.

            !!! important "Requirement"

                Must have `langchain-core>=0.3.11` installed to use this feature.

    Returns:
        A new list of messages with the messages from `right` merged into `left`.
        If a message in `right` has the same ID as a message in `left`, the
            message from `right` will replace the message from `left`.

    Example:
        ```python title="Basic usage"
        from langchain_core.messages import AIMessage, HumanMessage

        msgs1 = [HumanMessage(content="Hello", id="1")]
        msgs2 = [AIMessage(content="Hi there!", id="2")]
        add_messages(msgs1, msgs2)
        # [HumanMessage(content='Hello', id='1'), AIMessage(content='Hi there!', id='2')]
        ```

        ```python title="Overwrite existing message"
        msgs1 = [HumanMessage(content="Hello", id="1")]
        msgs2 = [HumanMessage(content="Hello again", id="1")]
        add_messages(msgs1, msgs2)
        # [HumanMessage(content='Hello again', id='1')]
        ```

        ```python title="Use in a StateGraph"
        from typing import Annotated
        from typing_extensions import TypedDict
        from langgraph.graph import StateGraph


        class State(TypedDict):
            messages: Annotated[list, add_messages]


        builder = StateGraph(State)
        builder.add_node("chatbot", lambda state: {"messages": [("assistant", "Hello")]})
        builder.set_entry_point("chatbot")
        builder.set_finish_point("chatbot")
        graph = builder.compile()
        graph.invoke({})
        # {'messages': [AIMessage(content='Hello', id=...)]}
        ```

        ```python title="Use OpenAI message format"
        from typing import Annotated
        from typing_extensions import TypedDict
        from langgraph.graph import StateGraph, add_messages


        class State(TypedDict):
            messages: Annotated[list, add_messages(format="langchain-openai")]


        def chatbot_node(state: State) -> list:
            return {
                "messages": [
                    {
                        "role": "user",
                        "content": [
                            {
                                "type": "text",
                                "text": "Here's an image:",
                                "cache_control": {"type": "ephemeral"},
                            },
                            {
                                "type": "image",
                                "source": {
                                    "type": "base64",
                                    "media_type": "image/jpeg",
                                    "data": "1234",
                                },
                            },
                        ],
                    },
                ]
            }


        builder = StateGraph(State)
        builder.add_node("chatbot", chatbot_node)
        builder.set_entry_point("chatbot")
        builder.set_finish_point("chatbot")
        graph = builder.compile()
        graph.invoke({"messages": []})
        # {
        #     'messages': [
        #         HumanMessage(
        #             content=[
        #                 {"type": "text", "text": "Here's an image:"},
        #                 {
        #                     "type": "image_url",
        #                     "image_url": {"url": "data:image/jpeg;base64,1234"},
        #                 },
        #             ],
        #         ),
        #     ]
        # }
        ```

    """
    remove_all_idx = None
    # coerce to list
    if not isinstance(left, list):
        left = [left]  # type: ignore[assignment]
    if not isinstance(right, list):
        right = [right]  # type: ignore[assignment]
    # coerce to message
    left = [
        message_chunk_to_message(cast(BaseMessageChunk, m))
        for m in convert_to_messages(left)
    ]
    right = [
        message_chunk_to_message(cast(BaseMessageChunk, m))
        for m in convert_to_messages(right)
    ]
    # assign missing ids
    for m in left:
        if m.id is None:
            m.id = str(uuid.uuid4())
    for idx, m in enumerate(right):
        if m.id is None:
            m.id = str(uuid.uuid4())
        if isinstance(m, RemoveMessage) and m.id == REMOVE_ALL_MESSAGES:
            remove_all_idx = idx

    if remove_all_idx is not None:
        return right[remove_all_idx + 1 :]

    # merge
    merged = left.copy()
    merged_by_id = {m.id: i for i, m in enumerate(merged)}
    ids_to_remove = set()
    for m in right:
        if (existing_idx := merged_by_id.get(m.id)) is not None:
            if isinstance(m, RemoveMessage):
                ids_to_remove.add(m.id)
            else:
                ids_to_remove.discard(m.id)
                merged[existing_idx] = m
        else:
            if isinstance(m, RemoveMessage):
                raise ValueError(
                    f"Attempting to delete a message with an ID that doesn't exist ('{m.id}')"
                )

            merged_by_id[m.id] = len(merged)
            merged.append(m)
    merged = [m for m in merged if m.id not in ids_to_remove]

    if format == "langchain-openai":
        merged = _format_messages(merged)
    elif format:
        msg = f"Unrecognized {format=}. Expected one of 'langchain-openai', None."
        raise ValueError(msg)
    else:
        pass

    return merged


@deprecated(
    "MessageGraph is deprecated in langgraph 1.0.0, to be removed in 2.0.0. Please use StateGraph with a `messages` key instead.",
    category=None,
)
class MessageGraph(StateGraph):
    """A StateGraph where every node receives a list of messages as input and returns one or more messages as output.

    MessageGraph is a subclass of StateGraph whose entire state is a single, append-only* list of messages.
    Each node in a MessageGraph takes a list of messages as input and returns zero or more
    messages as output. The `add_messages` function is used to merge the output messages from each node
    into the existing list of messages in the graph's state.

    Examples:
        ```pycon
        >>> from langgraph.graph.message import MessageGraph
        ...
        >>> builder = MessageGraph()
        >>> builder.add_node("chatbot", lambda state: [("assistant", "Hello!")])
        >>> builder.set_entry_point("chatbot")
        >>> builder.set_finish_point("chatbot")
        >>> builder.compile().invoke([("user", "Hi there.")])
        [HumanMessage(content="Hi there.", id='...'), AIMessage(content="Hello!", id='...')]
        ```

        ```pycon
        >>> from langchain_core.messages import AIMessage, HumanMessage, ToolMessage
        >>> from langgraph.graph.message import MessageGraph
        ...
        >>> builder = MessageGraph()
        >>> builder.add_node(
        ...     "chatbot",
        ...     lambda state: [
        ...         AIMessage(
        ...             content="Hello!",
        ...             tool_calls=[{"name": "search", "id": "123", "args": {"query": "X"}}],
        ...         )
        ...     ],
        ... )
        >>> builder.add_node(
        ...     "search", lambda state: [ToolMessage(content="Searching...", tool_call_id="123")]
        ... )
        >>> builder.set_entry_point("chatbot")
        >>> builder.add_edge("chatbot", "search")
        >>> builder.set_finish_point("search")
        >>> builder.compile().invoke([HumanMessage(content="Hi there. Can you search for X?")])
        {'messages': [HumanMessage(content="Hi there. Can you search for X?", id='b8b7d8f4-7f4d-4f4d-9c1d-f8b8d8f4d9c1'),
                     AIMessage(content="Hello!", id='f4d9c1d8-8d8f-4d9c-b8b7-d8f4f4d9c1d8'),
                     ToolMessage(content="Searching...", id='d8f4f4d9-c1d8-4f4d-b8b7-d8f4f4d9c1d8', tool_call_id="123")]}
        ```
    """

    def __init__(self) -> None:
        warnings.warn(
            "MessageGraph is deprecated in LangGraph v1.0.0, to be removed in v2.0.0. Please use StateGraph with a `messages` key instead.",
            category=LangGraphDeprecatedSinceV10,
            stacklevel=2,
        )
        super().__init__(Annotated[list[AnyMessage], add_messages])  # type: ignore[arg-type]


class MessagesState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]


def _format_messages(messages: Sequence[BaseMessage]) -> list[BaseMessage]:
    try:
        from langchain_core.messages import convert_to_openai_messages
    except ImportError:
        msg = (
            "Must have langchain-core>=0.3.11 installed to use automatic message "
            "formatting (format='langchain-openai'). Please update your langchain-core "
            "version or remove the 'format' flag. Returning un-formatted "
            "messages."
        )
        warnings.warn(msg)
        return list(messages)
    else:
        return convert_to_messages(convert_to_openai_messages(messages))


def push_message(
    message: MessageLikeRepresentation | BaseMessageChunk,
    *,
    state_key: str | None = "messages",
) -> AnyMessage:
    """Write a message manually to the `messages` / `messages-tuple` stream mode.

    Will automatically write to the channel specified in the `state_key` unless `state_key` is `None`.
    """

    from langchain_core.callbacks.base import (
        BaseCallbackHandler,
        BaseCallbackManager,
    )

    from langgraph.config import get_config
    from langgraph.pregel._messages import StreamMessagesHandler

    config = get_config()
    message = next(x for x in convert_to_messages([message]))

    if message.id is None:
        raise ValueError("Message ID is required")

    if isinstance(config["callbacks"], BaseCallbackManager):
        manager = config["callbacks"]
        handlers = manager.handlers
    elif isinstance(config["callbacks"], list) and all(
        isinstance(x, BaseCallbackHandler) for x in config["callbacks"]
    ):
        handlers = config["callbacks"]

    if stream_handler := next(
        (x for x in handlers if isinstance(x, StreamMessagesHandler)), None
    ):
        metadata = config["metadata"]
        message_meta = (
            tuple(cast(str, metadata["langgraph_checkpoint_ns"]).split(NS_SEP)),
            metadata,
        )
        stream_handler._emit(message_meta, message, dedupe=False)

    if state_key:
        config[CONF][CONFIG_KEY_SEND]([(state_key, message)])

    return message

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/graph/state.py
```py
from __future__ import annotations

import inspect
import logging
import typing
import warnings
from collections import defaultdict
from collections.abc import Awaitable, Callable, Hashable, Sequence
from functools import partial
from inspect import isclass, isfunction, ismethod, signature
from types import FunctionType
from types import NoneType as NoneType
from typing import (
    Any,
    Generic,
    Literal,
    Union,
    cast,
    get_args,
    get_origin,
    get_type_hints,
    overload,
)

from langchain_core.runnables import Runnable, RunnableConfig
from langgraph.cache.base import BaseCache
from langgraph.checkpoint.base import Checkpoint
from langgraph.store.base import BaseStore
from pydantic import BaseModel, TypeAdapter
from typing_extensions import NotRequired, Required, Self, Unpack, is_typeddict

from langgraph._internal._constants import (
    INTERRUPT,
    NS_END,
    NS_SEP,
    TASKS,
)
from langgraph._internal._fields import (
    get_cached_annotated_keys,
    get_field_default,
    get_update_as_tuples,
)
from langgraph._internal._pydantic import create_model
from langgraph._internal._runnable import coerce_to_runnable
from langgraph._internal._typing import EMPTY_SEQ, MISSING, DeprecatedKwargs
from langgraph.channels.base import BaseChannel
from langgraph.channels.binop import BinaryOperatorAggregate
from langgraph.channels.ephemeral_value import EphemeralValue
from langgraph.channels.last_value import LastValue, LastValueAfterFinish
from langgraph.channels.named_barrier_value import (
    NamedBarrierValue,
    NamedBarrierValueAfterFinish,
)
from langgraph.constants import END, START, TAG_HIDDEN
from langgraph.errors import (
    ErrorCode,
    InvalidUpdateError,
    ParentCommand,
    create_error_message,
)
from langgraph.graph._branch import BranchSpec
from langgraph.graph._node import StateNode, StateNodeSpec
from langgraph.managed.base import (
    ManagedValueSpec,
    is_managed_value,
)
from langgraph.pregel import Pregel
from langgraph.pregel._read import ChannelRead, PregelNode
from langgraph.pregel._write import (
    ChannelWrite,
    ChannelWriteEntry,
    ChannelWriteTupleEntry,
)
from langgraph.types import (
    All,
    CachePolicy,
    Checkpointer,
    Command,
    RetryPolicy,
    Send,
)
from langgraph.typing import ContextT, InputT, NodeInputT, OutputT, StateT
from langgraph.warnings import LangGraphDeprecatedSinceV05, LangGraphDeprecatedSinceV10

__all__ = ("StateGraph", "CompiledStateGraph")

logger = logging.getLogger(__name__)

_CHANNEL_BRANCH_TO = "branch:to:{}"


def _warn_invalid_state_schema(schema: type[Any] | Any) -> None:
    if isinstance(schema, type):
        return
    if typing.get_args(schema):
        return
    warnings.warn(
        f"Invalid state_schema: {schema}. Expected a type or Annotated[type, reducer]. "
        "Please provide a valid schema to ensure correct updates.\n"
        " See: https://langchain-ai.github.io/langgraph/reference/graphs/#stategraph"
    )


def _get_node_name(node: StateNode[Any, ContextT]) -> str:
    try:
        return getattr(node, "__name__", node.__class__.__name__)
    except AttributeError:
        raise TypeError(f"Unsupported node type: {type(node)}")


class StateGraph(Generic[StateT, ContextT, InputT, OutputT]):
    """A graph whose nodes communicate by reading and writing to a shared state.

    The signature of each node is `State -> Partial<State>`.

    Each state key can optionally be annotated with a reducer function that
    will be used to aggregate the values of that key received from multiple nodes.
    The signature of a reducer function is `(Value, Value) -> Value`.

    !!! warning

        `StateGraph` is a builder class and cannot be used directly for execution.
        You must first call `.compile()` to create an executable graph that supports
        methods like `invoke()`, `stream()`, `astream()`, and `ainvoke()`. See the
        `CompiledStateGraph` documentation for more details.

    Args:
        state_schema: The schema class that defines the state.
        context_schema: The schema class that defines the runtime context.
            Use this to expose immutable context data to your nodes, like `user_id`, `db_conn`, etc.
        input_schema: The schema class that defines the input to the graph.
        output_schema: The schema class that defines the output from the graph.

    !!! warning "`config_schema` Deprecated"
        The `config_schema` parameter is deprecated in v0.6.0 and support will be removed in v2.0.0.
        Please use `context_schema` instead to specify the schema for run-scoped context.

    Example:
        ```python
        from langchain_core.runnables import RunnableConfig
        from typing_extensions import Annotated, TypedDict
        from langgraph.checkpoint.memory import InMemorySaver
        from langgraph.graph import StateGraph
        from langgraph.runtime import Runtime


        def reducer(a: list, b: int | None) -> list:
            if b is not None:
                return a + [b]
            return a


        class State(TypedDict):
            x: Annotated[list, reducer]


        class Context(TypedDict):
            r: float


        graph = StateGraph(state_schema=State, context_schema=Context)


        def node(state: State, runtime: Runtime[Context]) -> dict:
            r = runtime.context.get("r", 1.0)
            x = state["x"][-1]
            next_value = x * r * (1 - x)
            return {"x": next_value}


        graph.add_node("A", node)
        graph.set_entry_point("A")
        graph.set_finish_point("A")
        compiled = graph.compile()

        step1 = compiled.invoke({"x": 0.5}, context={"r": 3.0})
        # {'x': [0.5, 0.75]}
        ```
    """

    edges: set[tuple[str, str]]
    nodes: dict[str, StateNodeSpec[Any, ContextT]]
    branches: defaultdict[str, dict[str, BranchSpec]]
    channels: dict[str, BaseChannel]
    managed: dict[str, ManagedValueSpec]
    schemas: dict[type[Any], dict[str, BaseChannel | ManagedValueSpec]]
    waiting_edges: set[tuple[tuple[str, ...], str]]

    compiled: bool
    state_schema: type[StateT]
    context_schema: type[ContextT] | None
    input_schema: type[InputT]
    output_schema: type[OutputT]

    def __init__(
        self,
        state_schema: type[StateT],
        context_schema: type[ContextT] | None = None,
        *,
        input_schema: type[InputT] | None = None,
        output_schema: type[OutputT] | None = None,
        **kwargs: Unpack[DeprecatedKwargs],
    ) -> None:
        if (config_schema := kwargs.get("config_schema", MISSING)) is not MISSING:
            warnings.warn(
                "`config_schema` is deprecated and will be removed. Please use `context_schema` instead.",
                category=LangGraphDeprecatedSinceV10,
                stacklevel=2,
            )
            if context_schema is None:
                context_schema = cast(type[ContextT], config_schema)

        if (input_ := kwargs.get("input", MISSING)) is not MISSING:
            warnings.warn(
                "`input` is deprecated and will be removed. Please use `input_schema` instead.",
                category=LangGraphDeprecatedSinceV05,
                stacklevel=2,
            )
            if input_schema is None:
                input_schema = cast(type[InputT], input_)

        if (output := kwargs.get("output", MISSING)) is not MISSING:
            warnings.warn(
                "`output` is deprecated and will be removed. Please use `output_schema` instead.",
                category=LangGraphDeprecatedSinceV05,
                stacklevel=2,
            )
            if output_schema is None:
                output_schema = cast(type[OutputT], output)

        self.nodes = {}
        self.edges = set()
        self.branches = defaultdict(dict)
        self.schemas = {}
        self.channels = {}
        self.managed = {}
        self.compiled = False
        self.waiting_edges = set()

        self.state_schema = state_schema
        self.input_schema = cast(type[InputT], input_schema or state_schema)
        self.output_schema = cast(type[OutputT], output_schema or state_schema)
        self.context_schema = context_schema

        self._add_schema(self.state_schema)
        self._add_schema(self.input_schema, allow_managed=False)
        self._add_schema(self.output_schema, allow_managed=False)

    @property
    def _all_edges(self) -> set[tuple[str, str]]:
        return self.edges | {
            (start, end) for starts, end in self.waiting_edges for start in starts
        }

    def _add_schema(self, schema: type[Any], /, allow_managed: bool = True) -> None:
        if schema not in self.schemas:
            _warn_invalid_state_schema(schema)
            channels, managed, type_hints = _get_channels(schema)
            if managed and not allow_managed:
                names = ", ".join(managed)
                schema_name = getattr(schema, "__name__", "")
                raise ValueError(
                    f"Invalid managed channels detected in {schema_name}: {names}."
                    " Managed channels are not permitted in Input/Output schema."
                )
            self.schemas[schema] = {**channels, **managed}
            for key, channel in channels.items():
                if key in self.channels:
                    if self.channels[key] != channel:
                        if isinstance(channel, LastValue):
                            pass
                        else:
                            raise ValueError(
                                f"Channel '{key}' already exists with a different type"
                            )
                else:
                    self.channels[key] = channel
            for key, managed in managed.items():
                if key in self.managed:
                    if self.managed[key] != managed:
                        raise ValueError(
                            f"Managed value '{key}' already exists with a different type"
                        )
                else:
                    self.managed[key] = managed

    @overload
    def add_node(
        self,
        node: StateNode[NodeInputT, ContextT],
        *,
        defer: bool = False,
        metadata: dict[str, Any] | None = None,
        input_schema: None = None,
        retry_policy: RetryPolicy | Sequence[RetryPolicy] | None = None,
        cache_policy: CachePolicy | None = None,
        destinations: dict[str, str] | tuple[str, ...] | None = None,
        **kwargs: Unpack[DeprecatedKwargs],
    ) -> Self:
        """Add a new node to the `StateGraph`, input schema is inferred as the state schema.
        Will take the name of the function/runnable as the node name.
        """
        ...

    @overload
    def add_node(
        self,
        node: StateNode[NodeInputT, ContextT],
        *,
        defer: bool = False,
        metadata: dict[str, Any] | None = None,
        input_schema: type[NodeInputT],
        retry_policy: RetryPolicy | Sequence[RetryPolicy] | None = None,
        cache_policy: CachePolicy | None = None,
        destinations: dict[str, str] | tuple[str, ...] | None = None,
        **kwargs: Unpack[DeprecatedKwargs],
    ) -> Self:
        """Add a new node to the `StateGraph`, input schema is specified.
        Will take the name of the function/runnable as the node name.
        """
        ...

    @overload
    def add_node(
        self,
        node: str,
        action: StateNode[NodeInputT, ContextT],
        *,
        defer: bool = False,
        metadata: dict[str, Any] | None = None,
        input_schema: None = None,
        retry_policy: RetryPolicy | Sequence[RetryPolicy] | None = None,
        cache_policy: CachePolicy | None = None,
        destinations: dict[str, str] | tuple[str, ...] | None = None,
        **kwargs: Unpack[DeprecatedKwargs],
    ) -> Self:
        """Add a new node to the `StateGraph`, input schema is inferred as the state schema."""
        ...

    @overload
    def add_node(
        self,
        node: str | StateNode[NodeInputT, ContextT],
        action: StateNode[NodeInputT, ContextT] | None = None,
        *,
        defer: bool = False,
        metadata: dict[str, Any] | None = None,
        input_schema: type[NodeInputT],
        retry_policy: RetryPolicy | Sequence[RetryPolicy] | None = None,
        cache_policy: CachePolicy | None = None,
        destinations: dict[str, str] | tuple[str, ...] | None = None,
        **kwargs: Unpack[DeprecatedKwargs],
    ) -> Self:
        """Add a new node to the `StateGraph`, input schema is specified."""
        ...

    def add_node(
        self,
        node: str | StateNode[NodeInputT, ContextT],
        action: StateNode[NodeInputT, ContextT] | None = None,
        *,
        defer: bool = False,
        metadata: dict[str, Any] | None = None,
        input_schema: type[NodeInputT] | None = None,
        retry_policy: RetryPolicy | Sequence[RetryPolicy] | None = None,
        cache_policy: CachePolicy | None = None,
        destinations: dict[str, str] | tuple[str, ...] | None = None,
        **kwargs: Unpack[DeprecatedKwargs],
    ) -> Self:
        """Add a new node to the `StateGraph`.

        Args:
            node: The function or runnable this node will run.
                If a string is provided, it will be used as the node name, and action will be used as the function or runnable.
            action: The action associated with the node.
                Will be used as the node function or runnable if `node` is a string (node name).
            defer: Whether to defer the execution of the node until the run is about to end.
            metadata: The metadata associated with the node.
            input_schema: The input schema for the node. (default: the graph's state schema)
            retry_policy: The retry policy for the node.
                If a sequence is provided, the first matching policy will be applied.
            cache_policy: The cache policy for the node.
            destinations: Destinations that indicate where a node can route to.
                This is useful for edgeless graphs with nodes that return `Command` objects.
                If a `dict` is provided, the keys will be used as the target node names and the values will be used as the labels for the edges.
                If a `tuple` is provided, the values will be used as the target node names.

                !!! note

                    This is only used for graph rendering and doesn't have any effect on the graph execution.

        Example:
            ```python
            from typing_extensions import TypedDict

            from langchain_core.runnables import RunnableConfig
            from langgraph.graph import START, StateGraph


            class State(TypedDict):
                x: int


            def my_node(state: State, config: RunnableConfig) -> State:
                return {"x": state["x"] + 1}


            builder = StateGraph(State)
            builder.add_node(my_node)  # node name will be 'my_node'
            builder.add_edge(START, "my_node")
            graph = builder.compile()
            graph.invoke({"x": 1})
            # {'x': 2}
            ```

        Example: Customize the name:
            ```python
            builder = StateGraph(State)
            builder.add_node("my_fair_node", my_node)
            builder.add_edge(START, "my_fair_node")
            graph = builder.compile()
            graph.invoke({"x": 1})
            # {'x': 2}
            ```

        Returns:
            Self: The instance of the `StateGraph`, allowing for method chaining.
        """
        if (retry := kwargs.get("retry", MISSING)) is not MISSING:
            warnings.warn(
                "`retry` is deprecated and will be removed. Please use `retry_policy` instead.",
                category=LangGraphDeprecatedSinceV05,
            )
            if retry_policy is None:
                retry_policy = retry  # type: ignore[assignment]

        if (input_ := kwargs.get("input", MISSING)) is not MISSING:
            warnings.warn(
                "`input` is deprecated and will be removed. Please use `input_schema` instead.",
                category=LangGraphDeprecatedSinceV05,
            )
            if input_schema is None:
                input_schema = cast(type[NodeInputT] | None, input_)

        if not isinstance(node, str):
            action = node
            if isinstance(action, Runnable):
                node = action.get_name()
            else:
                node = getattr(action, "__name__", action.__class__.__name__)
            if node is None:
                raise ValueError(
                    "Node name must be provided if action is not a function"
                )
        if self.compiled:
            logger.warning(
                "Adding a node to a graph that has already been compiled. This will "
                "not be reflected in the compiled graph."
            )
        if not isinstance(node, str):
            action = node
            node = cast(str, getattr(action, "name", getattr(action, "__name__", None)))
            if node is None:
                raise ValueError(
                    "Node name must be provided if action is not a function"
                )
        if action is None:
            raise RuntimeError
        if node in self.nodes:
            raise ValueError(f"Node `{node}` already present.")
        if node == END or node == START:
            raise ValueError(f"Node `{node}` is reserved.")

        for character in (NS_SEP, NS_END):
            if character in node:
                raise ValueError(
                    f"'{character}' is a reserved character and is not allowed in the node names."
                )

        inferred_input_schema = None

        ends: tuple[str, ...] | dict[str, str] = EMPTY_SEQ
        try:
            if (
                isfunction(action)
                or ismethod(action)
                or ismethod(getattr(action, "__call__", None))
            ) and (
                hints := get_type_hints(getattr(action, "__call__"))
                or get_type_hints(action)
            ):
                if input_schema is None:
                    first_parameter_name = next(
                        iter(
                            inspect.signature(
                                cast(FunctionType, action)
                            ).parameters.keys()
                        )
                    )
                    if input_hint := hints.get(first_parameter_name):
                        if isinstance(input_hint, type) and get_type_hints(input_hint):
                            inferred_input_schema = input_hint
                if rtn := hints.get("return"):
                    # Handle Union types
                    rtn_origin = get_origin(rtn)
                    if rtn_origin is Union:
                        rtn_args = get_args(rtn)
                        # Look for Command in the union
                        for arg in rtn_args:
                            arg_origin = get_origin(arg)
                            if arg_origin is Command:
                                rtn = arg
                                rtn_origin = arg_origin
                                break

                    # Check if it's a Command type
                    if (
                        rtn_origin is Command
                        and (rargs := get_args(rtn))
                        and get_origin(rargs[0]) is Literal
                        and (vals := get_args(rargs[0]))
                    ):
                        ends = vals
        except (NameError, TypeError, StopIteration):
            pass

        if destinations is not None:
            ends = destinations

        if input_schema is not None:
            self.nodes[node] = StateNodeSpec[NodeInputT, ContextT](
                coerce_to_runnable(action, name=node, trace=False),  # type: ignore[arg-type]
                metadata,
                input_schema=input_schema,
                retry_policy=retry_policy,
                cache_policy=cache_policy,
                ends=ends,
                defer=defer,
            )
        elif inferred_input_schema is not None:
            self.nodes[node] = StateNodeSpec(
                coerce_to_runnable(action, name=node, trace=False),  # type: ignore[arg-type]
                metadata,
                input_schema=inferred_input_schema,
                retry_policy=retry_policy,
                cache_policy=cache_policy,
                ends=ends,
                defer=defer,
            )
        else:
            self.nodes[node] = StateNodeSpec[StateT, ContextT](
                coerce_to_runnable(action, name=node, trace=False),  # type: ignore[arg-type]
                metadata,
                input_schema=self.state_schema,
                retry_policy=retry_policy,
                cache_policy=cache_policy,
                ends=ends,
                defer=defer,
            )

        input_schema = input_schema or inferred_input_schema
        if input_schema is not None:
            self._add_schema(input_schema)

        return self

    def add_edge(self, start_key: str | list[str], end_key: str) -> Self:
        """Add a directed edge from the start node (or list of start nodes) to the end node.

        When a single start node is provided, the graph will wait for that node to complete
        before executing the end node. When multiple start nodes are provided,
        the graph will wait for ALL of the start nodes to complete before executing the end node.

        Args:
            start_key: The key(s) of the start node(s) of the edge.
            end_key: The key of the end node of the edge.

        Raises:
            ValueError: If the start key is `'END'` or if the start key or end key is not present in the graph.

        Returns:
            Self: The instance of the `StateGraph`, allowing for method chaining.
        """
        if self.compiled:
            logger.warning(
                "Adding an edge to a graph that has already been compiled. This will "
                "not be reflected in the compiled graph."
            )

        if isinstance(start_key, str):
            if start_key == END:
                raise ValueError("END cannot be a start node")
            if end_key == START:
                raise ValueError("START cannot be an end node")

            # run this validation only for non-StateGraph graphs
            if not hasattr(self, "channels") and start_key in set(
                start for start, _ in self.edges
            ):
                raise ValueError(
                    f"Already found path for node '{start_key}'.\n"
                    "For multiple edges, use StateGraph with an Annotated state key."
                )

            self.edges.add((start_key, end_key))
            return self

        for start in start_key:
            if start == END:
                raise ValueError("END cannot be a start node")
            if start not in self.nodes:
                raise ValueError(f"Need to add_node `{start}` first")
        if end_key == START:
            raise ValueError("START cannot be an end node")
        if end_key != END and end_key not in self.nodes:
            raise ValueError(f"Need to add_node `{end_key}` first")

        self.waiting_edges.add((tuple(start_key), end_key))
        return self

    def add_conditional_edges(
        self,
        source: str,
        path: Callable[..., Hashable | Sequence[Hashable]]
        | Callable[..., Awaitable[Hashable | Sequence[Hashable]]]
        | Runnable[Any, Hashable | Sequence[Hashable]],
        path_map: dict[Hashable, str] | list[str] | None = None,
    ) -> Self:
        """Add a conditional edge from the starting node to any number of destination nodes.

        Args:
            source: The starting node. This conditional edge will run when
                exiting this node.
            path: The callable that determines the next
                node or nodes. If not specifying `path_map` it should return one or
                more nodes. If it returns `'END'`, the graph will stop execution.
            path_map: Optional mapping of paths to node
                names. If omitted the paths returned by `path` should be node names.

        Returns:
            Self: The instance of the graph, allowing for method chaining.

        !!! warning
            Without type hints on the `path` function's return value (e.g., `-> Literal["foo", "__end__"]:`)
            or a path_map, the graph visualization assumes the edge could transition to any node in the graph.

        """  # noqa: E501
        if self.compiled:
            logger.warning(
                "Adding an edge to a graph that has already been compiled. This will "
                "not be reflected in the compiled graph."
            )

        # find a name for the condition
        path = coerce_to_runnable(path, name=None, trace=True)
        name = path.name or "condition"
        # validate the condition
        if name in self.branches[source]:
            raise ValueError(
                f"Branch with name `{path.name}` already exists for node `{source}`"
            )
        # save it
        self.branches[source][name] = BranchSpec.from_path(path, path_map, True)
        if schema := self.branches[source][name].input_schema:
            self._add_schema(schema)
        return self

    def add_sequence(
        self,
        nodes: Sequence[
            StateNode[NodeInputT, ContextT]
            | tuple[str, StateNode[NodeInputT, ContextT]]
        ],
    ) -> Self:
        """Add a sequence of nodes that will be executed in the provided order.

        Args:
            nodes: A sequence of `StateNode` (callables that accept a `state` arg) or `(name, StateNode)` tuples.
                If no names are provided, the name will be inferred from the node object (e.g. a `Runnable` or a `Callable` name).
                Each node will be executed in the order provided.

        Raises:
            ValueError: If the sequence is empty.
            ValueError: If the sequence contains duplicate node names.

        Returns:
            Self: The instance of the `StateGraph`, allowing for method chaining.
        """
        if len(nodes) < 1:
            raise ValueError("Sequence requires at least one node.")

        previous_name: str | None = None
        for node in nodes:
            if isinstance(node, tuple) and len(node) == 2:
                name, node = node
            else:
                name = _get_node_name(node)

            if name in self.nodes:
                raise ValueError(
                    f"Node names must be unique: node with the name '{name}' already exists. "
                    "If you need to use two different runnables/callables with the same name (for example, using `lambda`), please provide them as tuples (name, runnable/callable)."
                )

            self.add_node(name, node)
            if previous_name is not None:
                self.add_edge(previous_name, name)

            previous_name = name

        return self

    def set_entry_point(self, key: str) -> Self:
        """Specifies the first node to be called in the graph.

        Equivalent to calling `add_edge(START, key)`.

        Parameters:
            key (str): The key of the node to set as the entry point.

        Returns:
            Self: The instance of the graph, allowing for method chaining.
        """
        return self.add_edge(START, key)

    def set_conditional_entry_point(
        self,
        path: Callable[..., Hashable | Sequence[Hashable]]
        | Callable[..., Awaitable[Hashable | Sequence[Hashable]]]
        | Runnable[Any, Hashable | Sequence[Hashable]],
        path_map: dict[Hashable, str] | list[str] | None = None,
    ) -> Self:
        """Sets a conditional entry point in the graph.

        Args:
            path: The callable that determines the next
                node or nodes. If not specifying `path_map` it should return one or
                more nodes. If it returns END, the graph will stop execution.
            path_map: Optional mapping of paths to node
                names. If omitted the paths returned by `path` should be node names.

        Returns:
            Self: The instance of the graph, allowing for method chaining.
        """
        return self.add_conditional_edges(START, path, path_map)

    def set_finish_point(self, key: str) -> Self:
        """Marks a node as a finish point of the graph.

        If the graph reaches this node, it will cease execution.

        Parameters:
            key (str): The key of the node to set as the finish point.

        Returns:
            Self: The instance of the graph, allowing for method chaining.
        """
        return self.add_edge(key, END)

    def validate(self, interrupt: Sequence[str] | None = None) -> Self:
        # assemble sources
        all_sources = {src for src, _ in self._all_edges}
        for start, branches in self.branches.items():
            all_sources.add(start)
        for name, spec in self.nodes.items():
            if spec.ends:
                all_sources.add(name)
        # validate sources
        for source in all_sources:
            if source not in self.nodes and source != START:
                raise ValueError(f"Found edge starting at unknown node '{source}'")

        if START not in all_sources:
            raise ValueError(
                "Graph must have an entrypoint: add at least one edge from START to another node"
            )

        # assemble targets
        all_targets = {end for _, end in self._all_edges}
        for start, branches in self.branches.items():
            for cond, branch in branches.items():
                if branch.ends is not None:
                    for end in branch.ends.values():
                        if end not in self.nodes and end != END:
                            raise ValueError(
                                f"At '{start}' node, '{cond}' branch found unknown target '{end}'"
                            )
                        all_targets.add(end)
                else:
                    all_targets.add(END)
                    for node in self.nodes:
                        if node != start:
                            all_targets.add(node)
        for name, spec in self.nodes.items():
            if spec.ends:
                all_targets.update(spec.ends)
        for target in all_targets:
            if target not in self.nodes and target != END:
                raise ValueError(f"Found edge ending at unknown node `{target}`")
        # validate interrupts
        if interrupt:
            for node in interrupt:
                if node not in self.nodes:
                    raise ValueError(f"Interrupt node `{node}` not found")

        self.compiled = True
        return self

    def compile(
        self,
        checkpointer: Checkpointer = None,
        *,
        cache: BaseCache | None = None,
        store: BaseStore | None = None,
        interrupt_before: All | list[str] | None = None,
        interrupt_after: All | list[str] | None = None,
        debug: bool = False,
        name: str | None = None,
    ) -> CompiledStateGraph[StateT, ContextT, InputT, OutputT]:
        """Compiles the `StateGraph` into a `CompiledStateGraph` object.

        The compiled graph implements the `Runnable` interface and can be invoked,
        streamed, batched, and run asynchronously.

        Args:
            checkpointer: A checkpoint saver object or flag.
                If provided, this `Checkpointer` serves as a fully versioned "short-term memory" for the graph,
                allowing it to be paused, resumed, and replayed from any point.
                If `None`, it may inherit the parent graph's checkpointer when used as a subgraph.
                If `False`, it will not use or inherit any checkpointer.
            interrupt_before: An optional list of node names to interrupt before.
            interrupt_after: An optional list of node names to interrupt after.
            debug: A flag indicating whether to enable debug mode.
            name: The name to use for the compiled graph.

        Returns:
            CompiledStateGraph: The compiled `StateGraph`.
        """
        # assign default values
        interrupt_before = interrupt_before or []
        interrupt_after = interrupt_after or []

        # validate the graph
        self.validate(
            interrupt=(
                (interrupt_before if interrupt_before != "*" else []) + interrupt_after
                if interrupt_after != "*"
                else []
            )
        )

        # prepare output channels
        output_channels = (
            "__root__"
            if len(self.schemas[self.output_schema]) == 1
            and "__root__" in self.schemas[self.output_schema]
            else [
                key
                for key, val in self.schemas[self.output_schema].items()
                if not is_managed_value(val)
            ]
        )
        stream_channels = (
            "__root__"
            if len(self.channels) == 1 and "__root__" in self.channels
            else [
                key for key, val in self.channels.items() if not is_managed_value(val)
            ]
        )

        compiled = CompiledStateGraph[StateT, ContextT, InputT, OutputT](
            builder=self,
            schema_to_mapper={},
            context_schema=self.context_schema,
            nodes={},
            channels={
                **self.channels,
                **self.managed,
                START: EphemeralValue(self.input_schema),
            },
            input_channels=START,
            stream_mode="updates",
            output_channels=output_channels,
            stream_channels=stream_channels,
            checkpointer=checkpointer,
            interrupt_before_nodes=interrupt_before,
            interrupt_after_nodes=interrupt_after,
            auto_validate=False,
            debug=debug,
            store=store,
            cache=cache,
            name=name or "LangGraph",
        )

        compiled.attach_node(START, None)
        for key, node in self.nodes.items():
            compiled.attach_node(key, node)

        for start, end in self.edges:
            compiled.attach_edge(start, end)

        for starts, end in self.waiting_edges:
            compiled.attach_edge(starts, end)

        for start, branches in self.branches.items():
            for name, branch in branches.items():
                compiled.attach_branch(start, name, branch)

        return compiled.validate()


class CompiledStateGraph(
    Pregel[StateT, ContextT, InputT, OutputT],
    Generic[StateT, ContextT, InputT, OutputT],
):
    builder: StateGraph[StateT, ContextT, InputT, OutputT]
    schema_to_mapper: dict[type[Any], Callable[[Any], Any] | None]

    def __init__(
        self,
        *,
        builder: StateGraph[StateT, ContextT, InputT, OutputT],
        schema_to_mapper: dict[type[Any], Callable[[Any], Any] | None],
        **kwargs: Any,
    ) -> None:
        super().__init__(**kwargs)
        self.builder = builder
        self.schema_to_mapper = schema_to_mapper

    def get_input_jsonschema(
        self, config: RunnableConfig | None = None
    ) -> dict[str, Any]:
        return _get_json_schema(
            typ=self.builder.input_schema,
            schemas=self.builder.schemas,
            channels=self.builder.channels,
            name=self.get_name("Input"),
        )

    def get_output_jsonschema(
        self, config: RunnableConfig | None = None
    ) -> dict[str, Any]:
        return _get_json_schema(
            typ=self.builder.output_schema,
            schemas=self.builder.schemas,
            channels=self.builder.channels,
            name=self.get_name("Output"),
        )

    def attach_node(self, key: str, node: StateNodeSpec[Any, ContextT] | None) -> None:
        if key == START:
            output_keys = [
                k
                for k, v in self.builder.schemas[self.builder.input_schema].items()
                if not is_managed_value(v)
            ]
        else:
            output_keys = list(self.builder.channels) + [
                k for k, v in self.builder.managed.items()
            ]

        def _get_updates(
            input: None | dict | Any,
        ) -> Sequence[tuple[str, Any]] | None:
            if input is None:
                return None
            elif isinstance(input, dict):
                return [(k, v) for k, v in input.items() if k in output_keys]
            elif isinstance(input, Command):
                if input.graph == Command.PARENT:
                    return None
                return [
                    (k, v) for k, v in input._update_as_tuples() if k in output_keys
                ]
            elif (
                isinstance(input, (list, tuple))
                and input
                and any(isinstance(i, Command) for i in input)
            ):
                updates: list[tuple[str, Any]] = []
                for i in input:
                    if isinstance(i, Command):
                        if i.graph == Command.PARENT:
                            continue
                        updates.extend(
                            (k, v) for k, v in i._update_as_tuples() if k in output_keys
                        )
                    else:
                        updates.extend(_get_updates(i) or ())
                return updates
            elif (t := type(input)) and get_cached_annotated_keys(t):
                return get_update_as_tuples(input, output_keys)
            else:
                msg = create_error_message(
                    message=f"Expected dict, got {input}",
                    error_code=ErrorCode.INVALID_GRAPH_NODE_RETURN_VALUE,
                )
                raise InvalidUpdateError(msg)

        # state updaters
        write_entries: tuple[ChannelWriteEntry | ChannelWriteTupleEntry, ...] = (
            ChannelWriteTupleEntry(
                mapper=_get_root if output_keys == ["__root__"] else _get_updates
            ),
            ChannelWriteTupleEntry(
                mapper=_control_branch,
                static=_control_static(node.ends)
                if node is not None and node.ends is not None
                else None,
            ),
        )

        # add node and output channel
        if key == START:
            self.nodes[key] = PregelNode(
                tags=[TAG_HIDDEN],
                triggers=[START],
                channels=START,
                writers=[ChannelWrite(write_entries)],
            )
        elif node is not None:
            input_schema = node.input_schema if node else self.builder.state_schema
            input_channels = list(self.builder.schemas[input_schema])
            is_single_input = len(input_channels) == 1 and "__root__" in input_channels
            if input_schema in self.schema_to_mapper:
                mapper = self.schema_to_mapper[input_schema]
            else:
                mapper = _pick_mapper(input_channels, input_schema)
                self.schema_to_mapper[input_schema] = mapper

            branch_channel = _CHANNEL_BRANCH_TO.format(key)
            self.channels[branch_channel] = (
                LastValueAfterFinish(Any)
                if node.defer
                else EphemeralValue(Any, guard=False)
            )
            self.nodes[key] = PregelNode(
                triggers=[branch_channel],
                # read state keys and managed values
                channels=("__root__" if is_single_input else input_channels),
                # coerce state dict to schema class (eg. pydantic model)
                mapper=mapper,
                # publish to state keys
                writers=[ChannelWrite(write_entries)],
                metadata=node.metadata,
                retry_policy=node.retry_policy,
                cache_policy=node.cache_policy,
                bound=node.runnable,  # type: ignore[arg-type]
            )
        else:
            raise RuntimeError

    def attach_edge(self, starts: str | Sequence[str], end: str) -> None:
        if isinstance(starts, str):
            # subscribe to start channel
            if end != END:
                self.nodes[starts].writers.append(
                    ChannelWrite(
                        (ChannelWriteEntry(_CHANNEL_BRANCH_TO.format(end), None),)
                    )
                )
        elif end != END:
            channel_name = f"join:{'+'.join(starts)}:{end}"
            # register channel
            if self.builder.nodes[end].defer:
                self.channels[channel_name] = NamedBarrierValueAfterFinish(
                    str, set(starts)
                )
            else:
                self.channels[channel_name] = NamedBarrierValue(str, set(starts))
            # subscribe to channel
            self.nodes[end].triggers.append(channel_name)
            # publish to channel
            for start in starts:
                self.nodes[start].writers.append(
                    ChannelWrite((ChannelWriteEntry(channel_name, start),))
                )

    def attach_branch(
        self, start: str, name: str, branch: BranchSpec, *, with_reader: bool = True
    ) -> None:
        def get_writes(
            packets: Sequence[str | Send], static: bool = False
        ) -> Sequence[ChannelWriteEntry | Send]:
            writes = [
                (
                    ChannelWriteEntry(
                        p if p == END else _CHANNEL_BRANCH_TO.format(p), None
                    )
                    if not isinstance(p, Send)
                    else p
                )
                for p in packets
                if (True if static else p != END)
            ]
            if not writes:
                return []
            return writes

        if with_reader:
            # get schema
            schema = branch.input_schema or (
                self.builder.nodes[start].input_schema
                if start in self.builder.nodes
                else self.builder.state_schema
            )
            channels = list(self.builder.schemas[schema])
            # get mapper
            if schema in self.schema_to_mapper:
                mapper = self.schema_to_mapper[schema]
            else:
                mapper = _pick_mapper(channels, schema)
                self.schema_to_mapper[schema] = mapper
            # create reader
            reader: Callable[[RunnableConfig], Any] | None = partial(
                ChannelRead.do_read,
                select=channels[0] if channels == ["__root__"] else channels,
                fresh=True,
                # coerce state dict to schema class (eg. pydantic model)
                mapper=mapper,
            )
        else:
            reader = None

        # attach branch publisher
        self.nodes[start].writers.append(branch.run(get_writes, reader))

    def _migrate_checkpoint(self, checkpoint: Checkpoint) -> None:
        """Migrate a checkpoint to new channel layout."""
        super()._migrate_checkpoint(checkpoint)

        values = checkpoint["channel_values"]
        versions = checkpoint["channel_versions"]
        seen = checkpoint["versions_seen"]

        # empty checkpoints do not need migration
        if not versions:
            return

        # current version
        if checkpoint["v"] >= 3:
            return

        # Migrate from start:node to branch:to:node
        for k in list(versions):
            if k.startswith("start:"):
                # confirm node is present
                node = k.split(":")[1]
                if node not in self.nodes:
                    continue
                # get next version
                new_k = f"branch:to:{node}"
                new_v = (
                    max(versions[new_k], versions.pop(k))
                    if new_k in versions
                    else versions.pop(k)
                )
                # update seen
                for ss in (seen.get(node, {}), seen.get(INTERRUPT, {})):
                    if k in ss:
                        s = ss.pop(k)
                        if new_k in ss:
                            ss[new_k] = max(s, ss[new_k])
                        else:
                            ss[new_k] = s
                # update value
                if new_k not in values and k in values:
                    values[new_k] = values.pop(k)
                # update version
                versions[new_k] = new_v

        # Migrate from branch:source:condition:node to branch:to:node
        for k in list(versions):
            if k.startswith("branch:") and k.count(":") == 3:
                # confirm node is present
                node = k.split(":")[-1]
                if node not in self.nodes:
                    continue
                # get next version
                new_k = f"branch:to:{node}"
                new_v = (
                    max(versions[new_k], versions.pop(k))
                    if new_k in versions
                    else versions.pop(k)
                )
                # update seen
                for ss in (seen.get(node, {}), seen.get(INTERRUPT, {})):
                    if k in ss:
                        s = ss.pop(k)
                        if new_k in ss:
                            ss[new_k] = max(s, ss[new_k])
                        else:
                            ss[new_k] = s
                # update value
                if new_k not in values and k in values:
                    values[new_k] = values.pop(k)
                # update version
                versions[new_k] = new_v

        if not set(self.nodes).isdisjoint(versions):
            # Migrate from "node" to "branch:to:node"
            source_to_target = defaultdict(list)
            for start, end in self.builder.edges:
                if start != START and end != END:
                    source_to_target[start].append(end)
            for k in list(versions):
                if k == START:
                    continue
                if k in self.nodes:
                    v = versions.pop(k)
                    c = values.pop(k, MISSING)
                    for end in source_to_target[k]:
                        # get next version
                        new_k = f"branch:to:{end}"
                        new_v = max(versions[new_k], v) if new_k in versions else v
                        # update seen
                        for ss in (seen.get(end, {}), seen.get(INTERRUPT, {})):
                            if k in ss:
                                s = ss.pop(k)
                                if new_k in ss:
                                    ss[new_k] = max(s, ss[new_k])
                                else:
                                    ss[new_k] = s
                        # update value
                        if new_k not in values and c is not MISSING:
                            values[new_k] = c
                        # update version
                        versions[new_k] = new_v
                    # pop interrupt seen
                    if INTERRUPT in seen:
                        seen[INTERRUPT].pop(k, MISSING)


def _pick_mapper(
    state_keys: Sequence[str], schema: type[Any]
) -> Callable[[Any], Any] | None:
    if state_keys == ["__root__"]:
        return None
    if isclass(schema) and issubclass(schema, dict):
        return None
    return partial(_coerce_state, schema)


def _coerce_state(schema: type[Any], input: dict[str, Any]) -> dict[str, Any]:
    return schema(**input)


def _control_branch(value: Any) -> Sequence[tuple[str, Any]]:
    if isinstance(value, Send):
        return ((TASKS, value),)
    commands: list[Command] = []
    if isinstance(value, Command):
        commands.append(value)
    elif isinstance(value, (list, tuple)):
        for cmd in value:
            if isinstance(cmd, Command):
                commands.append(cmd)
    rtn: list[tuple[str, Any]] = []
    for command in commands:
        if command.graph == Command.PARENT:
            raise ParentCommand(command)

        goto_targets = (
            [command.goto] if isinstance(command.goto, (Send, str)) else command.goto
        )

        for go in goto_targets:
            if isinstance(go, Send):
                rtn.append((TASKS, go))
            elif isinstance(go, str) and go != END:
                # END is a special case, it's not actually a node in a practical sense
                # but rather a special terminal node that we don't need to branch to
                rtn.append((_CHANNEL_BRANCH_TO.format(go), None))
    return rtn


def _control_static(
    ends: tuple[str, ...] | dict[str, str],
) -> Sequence[tuple[str, Any, str | None]]:
    if isinstance(ends, dict):
        return [
            (k if k == END else _CHANNEL_BRANCH_TO.format(k), None, label)
            for k, label in ends.items()
        ]
    else:
        return [
            (e if e == END else _CHANNEL_BRANCH_TO.format(e), None, None) for e in ends
        ]


def _get_root(input: Any) -> Sequence[tuple[str, Any]] | None:
    if isinstance(input, Command):
        if input.graph == Command.PARENT:
            return ()
        return input._update_as_tuples()
    elif (
        isinstance(input, (list, tuple))
        and input
        and any(isinstance(i, Command) for i in input)
    ):
        updates: list[tuple[str, Any]] = []
        for i in input:
            if isinstance(i, Command):
                if i.graph == Command.PARENT:
                    continue
                updates.extend(i._update_as_tuples())
            else:
                updates.append(("__root__", i))
        return updates
    elif input is not None:
        return [("__root__", input)]


def _get_channels(
    schema: type[dict],
) -> tuple[dict[str, BaseChannel], dict[str, ManagedValueSpec], dict[str, Any]]:
    if not hasattr(schema, "__annotations__"):
        return (
            {"__root__": _get_channel("__root__", schema, allow_managed=False)},
            {},
            {},
        )

    type_hints = get_type_hints(schema, include_extras=True)
    all_keys = {
        name: _get_channel(name, typ)
        for name, typ in type_hints.items()
        if name != "__slots__"
    }
    return (
        {k: v for k, v in all_keys.items() if isinstance(v, BaseChannel)},
        {k: v for k, v in all_keys.items() if is_managed_value(v)},
        type_hints,
    )


@overload
def _get_channel(
    name: str, annotation: Any, *, allow_managed: Literal[False]
) -> BaseChannel: ...


@overload
def _get_channel(
    name: str, annotation: Any, *, allow_managed: Literal[True] = True
) -> BaseChannel | ManagedValueSpec: ...


def _get_channel(
    name: str, annotation: Any, *, allow_managed: bool = True
) -> BaseChannel | ManagedValueSpec:
    # Strip out Required and NotRequired wrappers
    if hasattr(annotation, "__origin__") and annotation.__origin__ in (
        Required,
        NotRequired,
    ):
        annotation = annotation.__args__[0]
    if manager := _is_field_managed_value(name, annotation):
        if allow_managed:
            return manager
        else:
            raise ValueError(f"This {annotation} not allowed in this position")
    elif channel := _is_field_channel(annotation):
        channel.key = name
        return channel
    elif channel := _is_field_binop(annotation):
        channel.key = name
        return channel

    fallback: LastValue = LastValue(annotation)
    fallback.key = name
    return fallback


def _is_field_channel(typ: type[Any]) -> BaseChannel | None:
    if hasattr(typ, "__metadata__"):
        meta = typ.__metadata__
        # Search through all annotated medata to find channel annotations
        for item in meta:
            if isinstance(item, BaseChannel):
                return item
            elif isclass(item) and issubclass(item, BaseChannel):
                # ex, Annotated[int, EphemeralValue, SomeOtherAnnotation]
                # would return EphemeralValue(int)
                return item(typ.__origin__ if hasattr(typ, "__origin__") else typ)
    return None


def _is_field_binop(typ: type[Any]) -> BinaryOperatorAggregate | None:
    if hasattr(typ, "__metadata__"):
        meta = typ.__metadata__
        if len(meta) >= 1 and callable(meta[-1]):
            sig = signature(meta[-1])
            params = list(sig.parameters.values())
            if (
                sum(
                    p.kind in (p.POSITIONAL_ONLY, p.POSITIONAL_OR_KEYWORD)
                    for p in params
                )
                == 2
            ):
                return BinaryOperatorAggregate(typ, meta[-1])
            else:
                raise ValueError(
                    f"Invalid reducer signature. Expected (a, b) -> c. Got {sig}"
                )
    return None


def _is_field_managed_value(name: str, typ: type[Any]) -> ManagedValueSpec | None:
    if hasattr(typ, "__metadata__"):
        meta = typ.__metadata__
        if len(meta) >= 1:
            decoration = get_origin(meta[-1]) or meta[-1]
            if is_managed_value(decoration):
                return decoration

    # Handle Required, NotRequired, etc wrapped types by extracting the inner type
    if (
        get_origin(typ) is not None
        and (args := get_args(typ))
        and (inner_type := args[0])
    ):
        return _is_field_managed_value(name, inner_type)

    return None


def _get_json_schema(
    typ: type,
    schemas: dict,
    channels: dict,
    name: str,
) -> dict[str, Any]:
    if isclass(typ) and issubclass(typ, BaseModel):
        return typ.model_json_schema()
    elif is_typeddict(typ):
        return TypeAdapter(typ).json_schema()
    else:
        keys = list(schemas[typ].keys())
        if len(keys) == 1 and keys[0] == "__root__":
            return create_model(
                name,
                root=(channels[keys[0]].UpdateType, None),
            ).model_json_schema()
        else:
            return create_model(
                name,
                field_definitions={
                    k: (
                        channels[k].UpdateType,
                        (
                            get_field_default(
                                k,
                                channels[k].UpdateType,
                                typ,
                            )
                        ),
                    )
                    for k in schemas[typ]
                    if k in channels and isinstance(channels[k], BaseChannel)
                },
            ).model_json_schema()

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/graph/ui.py
```py
from __future__ import annotations

from typing import Any, Literal, cast
from uuid import uuid4

from langchain_core.messages import AnyMessage
from typing_extensions import TypedDict

from langgraph.config import get_config, get_stream_writer
from langgraph.constants import CONF

__all__ = (
    "UIMessage",
    "RemoveUIMessage",
    "AnyUIMessage",
    "push_ui_message",
    "delete_ui_message",
    "ui_message_reducer",
)


class UIMessage(TypedDict):
    """A message type for UI updates in LangGraph.

    This TypedDict represents a UI message that can be sent to update the UI state.
    It contains information about the UI component to render and its properties.

    Attributes:
        type: Literal type indicating this is a UI message.
        id: Unique identifier for the UI message.
        name: Name of the UI component to render.
        props: Properties to pass to the UI component.
        metadata: Additional metadata about the UI message.
    """

    type: Literal["ui"]
    id: str
    name: str
    props: dict[str, Any]
    metadata: dict[str, Any]


class RemoveUIMessage(TypedDict):
    """A message type for removing UI components in LangGraph.

    This TypedDict represents a message that can be sent to remove a UI component
    from the current state.

    Attributes:
        type: Literal type indicating this is a remove-ui message.
        id: Unique identifier of the UI message to remove.
    """

    type: Literal["remove-ui"]
    id: str


AnyUIMessage = UIMessage | RemoveUIMessage


def push_ui_message(
    name: str,
    props: dict[str, Any],
    *,
    id: str | None = None,
    metadata: dict[str, Any] | None = None,
    message: AnyMessage | None = None,
    state_key: str | None = "ui",
    merge: bool = False,
) -> UIMessage:
    """Push a new UI message to update the UI state.

    This function creates and sends a UI message that will be rendered in the UI.
    It also updates the graph state with the new UI message.

    Args:
        name: Name of the UI component to render.
        props: Properties to pass to the UI component.
        id: Optional unique identifier for the UI message.
            If not provided, a random UUID will be generated.
        metadata: Optional additional metadata about the UI message.
        message: Optional message object to associate with the UI message.
        state_key: Key in the graph state where the UI messages are stored.
        merge: Whether to merge props with existing UI message (True) or replace
            them (False).

    Returns:
        The created UI message.

    Example:
        ```python
        push_ui_message(
            name="component-name",
            props={"content": "Hello world"},
        )
        ```

    """
    from langgraph._internal._constants import CONFIG_KEY_SEND

    writer = get_stream_writer()
    config = get_config()

    message_id = None
    if message:
        if isinstance(message, dict) and "id" in message:
            message_id = message.get("id")
        elif hasattr(message, "id"):
            message_id = message.id

    evt: UIMessage = {
        "type": "ui",
        "id": id or str(uuid4()),
        "name": name,
        "props": props,
        "metadata": {
            "merge": merge,
            "run_id": config.get("run_id", None),
            "tags": config.get("tags", None),
            "name": config.get("run_name", None),
            **(metadata or {}),
            **({"message_id": message_id} if message_id else {}),
        },
    }

    writer(evt)
    if state_key:
        config[CONF][CONFIG_KEY_SEND]([(state_key, evt)])

    return evt


def delete_ui_message(id: str, *, state_key: str = "ui") -> RemoveUIMessage:
    """Delete a UI message by ID from the UI state.

    This function creates and sends a message to remove a UI component from the current state.
    It also updates the graph state to remove the UI message.

    Args:
        id: Unique identifier of the UI component to remove.
        state_key: Key in the graph state where the UI messages are stored. Defaults to "ui".

    Returns:
        The remove UI message.

    Example:
        ```python
        delete_ui_message("message-123")
        ```

    """
    from langgraph._internal._constants import CONFIG_KEY_SEND

    writer = get_stream_writer()
    config = get_config()

    evt: RemoveUIMessage = {"type": "remove-ui", "id": id}

    writer(evt)
    config[CONF][CONFIG_KEY_SEND]([(state_key, evt)])

    return evt


def ui_message_reducer(
    left: list[AnyUIMessage] | AnyUIMessage,
    right: list[AnyUIMessage] | AnyUIMessage,
) -> list[AnyUIMessage]:
    """Merge two lists of UI messages, supporting removing UI messages.

    This function combines two lists of UI messages, handling both regular UI messages
    and `remove-ui` messages. When a `remove-ui` message is encountered, it removes any
    UI message with the matching ID from the current state.

    Args:
        left: First list of UI messages or single UI message.
        right: Second list of UI messages or single UI message.

    Returns:
        Combined list of UI messages with removals applied.

    Example:
        ```python
        messages = ui_message_reducer(
            [{"type": "ui", "id": "1", "name": "Chat", "props": {}}],
            {"type": "remove-ui", "id": "1"},
        )
        ```

    """
    if not isinstance(left, list):
        left = [left]

    if not isinstance(right, list):
        right = [right]

    # merge messages
    merged = left.copy()
    merged_by_id = {m.get("id"): i for i, m in enumerate(merged)}
    ids_to_remove = set()

    for msg in right:
        msg_id = msg.get("id")

        if (existing_idx := merged_by_id.get(msg_id)) is not None:
            if msg.get("type") == "remove-ui":
                ids_to_remove.add(msg_id)
            else:
                ids_to_remove.discard(msg_id)

                if cast(UIMessage, msg).get("metadata", {}).get("merge", False):
                    prev_msg = merged[existing_idx]
                    msg = msg.copy()
                    msg["props"] = {**prev_msg["props"], **msg["props"]}

                merged[existing_idx] = msg
        else:
            if msg.get("type") == "remove-ui":
                raise ValueError(
                    f"Attempting to delete an UI message with an ID that doesn't exist ('{msg_id}')"
                )

            merged_by_id[msg_id] = len(merged)
            merged.append(msg)

    merged = [m for m in merged if m.get("id") not in ids_to_remove]
    return merged

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/managed/__init__.py
```py
from langgraph.managed.is_last_step import IsLastStep, RemainingSteps

__all__ = ("IsLastStep", "RemainingSteps")

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/managed/base.py
```py
from abc import ABC, abstractmethod
from inspect import isclass
from typing import (
    Any,
    Generic,
    TypeGuard,
    TypeVar,
)

from langgraph._internal._scratchpad import PregelScratchpad

V = TypeVar("V")
U = TypeVar("U")

__all__ = ("ManagedValueSpec", "ManagedValueMapping")


class ManagedValue(ABC, Generic[V]):
    @staticmethod
    @abstractmethod
    def get(scratchpad: PregelScratchpad) -> V: ...


ManagedValueSpec = type[ManagedValue]


def is_managed_value(value: Any) -> TypeGuard[ManagedValueSpec]:
    return isclass(value) and issubclass(value, ManagedValue)


ManagedValueMapping = dict[str, ManagedValueSpec]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/managed/is_last_step.py
```py
from typing import Annotated

from langgraph._internal._scratchpad import PregelScratchpad
from langgraph.managed.base import ManagedValue

__all__ = ("IsLastStep", "RemainingStepsManager")


class IsLastStepManager(ManagedValue[bool]):
    @staticmethod
    def get(scratchpad: PregelScratchpad) -> bool:
        return scratchpad.step == scratchpad.stop - 1


IsLastStep = Annotated[bool, IsLastStepManager]


class RemainingStepsManager(ManagedValue[int]):
    @staticmethod
    def get(scratchpad: PregelScratchpad) -> int:
        return scratchpad.stop - scratchpad.step


RemainingSteps = Annotated[int, RemainingStepsManager]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/channels/__init__.py
```py
from langgraph.channels.any_value import AnyValue
from langgraph.channels.base import BaseChannel
from langgraph.channels.binop import BinaryOperatorAggregate
from langgraph.channels.ephemeral_value import EphemeralValue
from langgraph.channels.last_value import LastValue, LastValueAfterFinish
from langgraph.channels.named_barrier_value import (
    NamedBarrierValue,
    NamedBarrierValueAfterFinish,
)
from langgraph.channels.topic import Topic
from langgraph.channels.untracked_value import UntrackedValue

__all__ = (
    # base
    "BaseChannel",
    # value types
    "AnyValue",
    "LastValue",
    "LastValueAfterFinish",
    "UntrackedValue",
    "EphemeralValue",
    "BinaryOperatorAggregate",
    "NamedBarrierValue",
    "NamedBarrierValueAfterFinish",
    # topics
    "Topic",
)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/channels/any_value.py
```py
from __future__ import annotations

from collections.abc import Sequence
from typing import Any, Generic

from typing_extensions import Self

from langgraph._internal._typing import MISSING
from langgraph.channels.base import BaseChannel, Value
from langgraph.errors import EmptyChannelError

__all__ = ("AnyValue",)


class AnyValue(Generic[Value], BaseChannel[Value, Value, Value]):
    """Stores the last value received, assumes that if multiple values are
    received, they are all equal."""

    __slots__ = ("typ", "value")

    value: Value | Any

    def __init__(self, typ: Any, key: str = "") -> None:
        super().__init__(typ, key)
        self.value = MISSING

    def __eq__(self, value: object) -> bool:
        return isinstance(value, AnyValue)

    @property
    def ValueType(self) -> type[Value]:
        """The type of the value stored in the channel."""
        return self.typ

    @property
    def UpdateType(self) -> type[Value]:
        """The type of the update received by the channel."""
        return self.typ

    def copy(self) -> Self:
        """Return a copy of the channel."""
        empty = self.__class__(self.typ, self.key)
        empty.value = self.value
        return empty

    def from_checkpoint(self, checkpoint: Value) -> Self:
        empty = self.__class__(self.typ, self.key)
        if checkpoint is not MISSING:
            empty.value = checkpoint
        return empty

    def update(self, values: Sequence[Value]) -> bool:
        if len(values) == 0:
            if self.value is MISSING:
                return False
            else:
                self.value = MISSING
                return True

        self.value = values[-1]
        return True

    def get(self) -> Value:
        if self.value is MISSING:
            raise EmptyChannelError()
        return self.value

    def is_available(self) -> bool:
        return self.value is not MISSING

    def checkpoint(self) -> Value:
        return self.value

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/channels/base.py
```py
from __future__ import annotations

from abc import ABC, abstractmethod
from collections.abc import Sequence
from typing import Any, Generic, TypeVar

from typing_extensions import Self

from langgraph._internal._typing import MISSING
from langgraph.errors import EmptyChannelError

Value = TypeVar("Value")
Update = TypeVar("Update")
Checkpoint = TypeVar("Checkpoint")

__all__ = ("BaseChannel",)


class BaseChannel(Generic[Value, Update, Checkpoint], ABC):
    """Base class for all channels."""

    __slots__ = ("key", "typ")

    def __init__(self, typ: Any, key: str = "") -> None:
        self.typ = typ
        self.key = key

    @property
    @abstractmethod
    def ValueType(self) -> Any:
        """The type of the value stored in the channel."""

    @property
    @abstractmethod
    def UpdateType(self) -> Any:
        """The type of the update received by the channel."""

    # serialize/deserialize methods

    def copy(self) -> Self:
        """Return a copy of the channel.
        By default, delegates to `checkpoint()` and `from_checkpoint()`.
        Subclasses can override this method with a more efficient implementation."""
        return self.from_checkpoint(self.checkpoint())

    def checkpoint(self) -> Checkpoint | Any:
        """Return a serializable representation of the channel's current state.
        Raises `EmptyChannelError` if the channel is empty (never updated yet),
        or doesn't support checkpoints."""
        try:
            return self.get()
        except EmptyChannelError:
            return MISSING

    @abstractmethod
    def from_checkpoint(self, checkpoint: Checkpoint | Any) -> Self:
        """Return a new identical channel, optionally initialized from a checkpoint.
        If the checkpoint contains complex data structures, they should be copied."""

    # read methods

    @abstractmethod
    def get(self) -> Value:
        """Return the current value of the channel.

        Raises `EmptyChannelError` if the channel is empty (never updated yet)."""

    def is_available(self) -> bool:
        """Return `True` if the channel is available (not empty), `False` otherwise.
        Subclasses should override this method to provide a more efficient
        implementation than calling `get()` and catching `EmptyChannelError`.
        """
        try:
            self.get()
            return True
        except EmptyChannelError:
            return False

    # write methods

    @abstractmethod
    def update(self, values: Sequence[Update]) -> bool:
        """Update the channel's value with the given sequence of updates.
        The order of the updates in the sequence is arbitrary.
        This method is called by Pregel for all channels at the end of each step.
        If there are no updates, it is called with an empty sequence.
        Raises `InvalidUpdateError` if the sequence of updates is invalid.
        Returns `True` if the channel was updated, `False` otherwise."""

    def consume(self) -> bool:
        """Notify the channel that a subscribed task ran. By default, no-op.
        A channel can use this method to modify its state, preventing the value
        from being consumed again.

        Returns `True` if the channel was updated, `False` otherwise.
        """
        return False

    def finish(self) -> bool:
        """Notify the channel that the Pregel run is finishing. By default, no-op.
        A channel can use this method to modify its state, preventing finish.

        Returns `True` if the channel was updated, `False` otherwise.
        """
        return False

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/channels/binop.py
```py
import collections.abc
from collections.abc import Callable, Sequence
from typing import Any, Generic

from typing_extensions import NotRequired, Required, Self

from langgraph._internal._constants import OVERWRITE
from langgraph._internal._typing import MISSING
from langgraph.channels.base import BaseChannel, Value
from langgraph.errors import (
    EmptyChannelError,
    ErrorCode,
    InvalidUpdateError,
    create_error_message,
)
from langgraph.types import Overwrite

__all__ = ("BinaryOperatorAggregate",)


# Adapted from typing_extensions
def _strip_extras(t):  # type: ignore[no-untyped-def]
    """Strips Annotated, Required and NotRequired from a given type."""
    if hasattr(t, "__origin__"):
        return _strip_extras(t.__origin__)
    if hasattr(t, "__origin__") and t.__origin__ in (Required, NotRequired):
        return _strip_extras(t.__args__[0])

    return t


def _get_overwrite(value: Any) -> tuple[bool, Any]:
    """Inspects the given value and returns (is_overwrite, overwrite_value)."""
    if isinstance(value, Overwrite):
        return True, value.value
    if isinstance(value, dict) and set(value.keys()) == {OVERWRITE}:
        return True, value[OVERWRITE]
    return False, None


class BinaryOperatorAggregate(Generic[Value], BaseChannel[Value, Value, Value]):
    """Stores the result of applying a binary operator to the current value and each new value.

    ```python
    import operator

    total = Channels.BinaryOperatorAggregate(int, operator.add)
    ```
    """

    __slots__ = ("value", "operator")

    def __init__(self, typ: type[Value], operator: Callable[[Value, Value], Value]):
        super().__init__(typ)
        self.operator = operator
        # special forms from typing or collections.abc are not instantiable
        # so we need to replace them with their concrete counterparts
        typ = _strip_extras(typ)
        if typ in (collections.abc.Sequence, collections.abc.MutableSequence):
            typ = list
        if typ in (collections.abc.Set, collections.abc.MutableSet):
            typ = set
        if typ in (collections.abc.Mapping, collections.abc.MutableMapping):
            typ = dict
        try:
            self.value = typ()
        except Exception:
            self.value = MISSING

    def __eq__(self, value: object) -> bool:
        return isinstance(value, BinaryOperatorAggregate) and (
            value.operator is self.operator
            if value.operator.__name__ != "<lambda>"
            and self.operator.__name__ != "<lambda>"
            else True
        )

    @property
    def ValueType(self) -> type[Value]:
        """The type of the value stored in the channel."""
        return self.typ

    @property
    def UpdateType(self) -> type[Value]:
        """The type of the update received by the channel."""
        return self.typ

    def copy(self) -> Self:
        """Return a copy of the channel."""
        empty = self.__class__(self.typ, self.operator)
        empty.key = self.key
        empty.value = self.value
        return empty

    def from_checkpoint(self, checkpoint: Value) -> Self:
        empty = self.__class__(self.typ, self.operator)
        empty.key = self.key
        if checkpoint is not MISSING:
            empty.value = checkpoint
        return empty

    def update(self, values: Sequence[Value]) -> bool:
        if not values:
            return False
        if self.value is MISSING:
            self.value = values[0]
            values = values[1:]
        seen_overwrite: bool = False
        for value in values:
            is_overwrite, overwrite_value = _get_overwrite(value)
            if is_overwrite:
                if seen_overwrite:
                    msg = create_error_message(
                        message="Can receive only one Overwrite value per super-step.",
                        error_code=ErrorCode.INVALID_CONCURRENT_GRAPH_UPDATE,
                    )
                    raise InvalidUpdateError(msg)
                self.value = overwrite_value
                seen_overwrite = True
                continue
            if not seen_overwrite:
                self.value = self.operator(self.value, value)
        return True

    def get(self) -> Value:
        if self.value is MISSING:
            raise EmptyChannelError()
        return self.value

    def is_available(self) -> bool:
        return self.value is not MISSING

    def checkpoint(self) -> Value:
        return self.value

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/channels/ephemeral_value.py
```py
from __future__ import annotations

from collections.abc import Sequence
from typing import Any, Generic

from typing_extensions import Self

from langgraph._internal._typing import MISSING
from langgraph.channels.base import BaseChannel, Value
from langgraph.errors import EmptyChannelError, InvalidUpdateError

__all__ = ("EphemeralValue",)


class EphemeralValue(Generic[Value], BaseChannel[Value, Value, Value]):
    """Stores the value received in the step immediately preceding, clears after."""

    __slots__ = ("value", "guard")

    value: Value | Any
    guard: bool

    def __init__(self, typ: Any, guard: bool = True) -> None:
        super().__init__(typ)
        self.guard = guard
        self.value = MISSING

    def __eq__(self, value: object) -> bool:
        return isinstance(value, EphemeralValue) and value.guard == self.guard

    @property
    def ValueType(self) -> type[Value]:
        """The type of the value stored in the channel."""
        return self.typ

    @property
    def UpdateType(self) -> type[Value]:
        """The type of the update received by the channel."""
        return self.typ

    def copy(self) -> Self:
        """Return a copy of the channel."""
        empty = self.__class__(self.typ, self.guard)
        empty.key = self.key
        empty.value = self.value
        return empty

    def from_checkpoint(self, checkpoint: Value) -> Self:
        empty = self.__class__(self.typ, self.guard)
        empty.key = self.key
        if checkpoint is not MISSING:
            empty.value = checkpoint
        return empty

    def update(self, values: Sequence[Value]) -> bool:
        if len(values) == 0:
            if self.value is not MISSING:
                self.value = MISSING
                return True
            else:
                return False
        if len(values) != 1 and self.guard:
            raise InvalidUpdateError(
                f"At key '{self.key}': EphemeralValue(guard=True) can receive only one value per step. Use guard=False if you want to store any one of multiple values."
            )

        self.value = values[-1]
        return True

    def get(self) -> Value:
        if self.value is MISSING:
            raise EmptyChannelError()
        return self.value

    def is_available(self) -> bool:
        return self.value is not MISSING

    def checkpoint(self) -> Value:
        return self.value

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/channels/last_value.py
```py
from __future__ import annotations

from collections.abc import Sequence
from typing import Any, Generic

from typing_extensions import Self

from langgraph._internal._typing import MISSING
from langgraph.channels.base import BaseChannel, Value
from langgraph.errors import (
    EmptyChannelError,
    ErrorCode,
    InvalidUpdateError,
    create_error_message,
)

__all__ = ("LastValue", "LastValueAfterFinish")


class LastValue(Generic[Value], BaseChannel[Value, Value, Value]):
    """Stores the last value received, can receive at most one value per step."""

    __slots__ = ("value",)

    value: Value | Any

    def __init__(self, typ: Any, key: str = "") -> None:
        super().__init__(typ, key)
        self.value = MISSING

    def __eq__(self, value: object) -> bool:
        return isinstance(value, LastValue)

    @property
    def ValueType(self) -> type[Value]:
        """The type of the value stored in the channel."""
        return self.typ

    @property
    def UpdateType(self) -> type[Value]:
        """The type of the update received by the channel."""
        return self.typ

    def copy(self) -> Self:
        """Return a copy of the channel."""
        empty = self.__class__(self.typ, self.key)
        empty.value = self.value
        return empty

    def from_checkpoint(self, checkpoint: Value) -> Self:
        empty = self.__class__(self.typ, self.key)
        if checkpoint is not MISSING:
            empty.value = checkpoint
        return empty

    def update(self, values: Sequence[Value]) -> bool:
        if len(values) == 0:
            return False
        if len(values) != 1:
            msg = create_error_message(
                message=f"At key '{self.key}': Can receive only one value per step. Use an Annotated key to handle multiple values.",
                error_code=ErrorCode.INVALID_CONCURRENT_GRAPH_UPDATE,
            )
            raise InvalidUpdateError(msg)

        self.value = values[-1]
        return True

    def get(self) -> Value:
        if self.value is MISSING:
            raise EmptyChannelError()
        return self.value

    def is_available(self) -> bool:
        return self.value is not MISSING

    def checkpoint(self) -> Value:
        return self.value


class LastValueAfterFinish(
    Generic[Value], BaseChannel[Value, Value, tuple[Value, bool]]
):
    """Stores the last value received, but only made available after finish().
    Once made available, clears the value."""

    __slots__ = ("value", "finished")

    value: Value | Any
    finished: bool

    def __init__(self, typ: Any, key: str = "") -> None:
        super().__init__(typ, key)
        self.value = MISSING
        self.finished = False

    def __eq__(self, value: object) -> bool:
        return isinstance(value, LastValueAfterFinish)

    @property
    def ValueType(self) -> type[Value]:
        """The type of the value stored in the channel."""
        return self.typ

    @property
    def UpdateType(self) -> type[Value]:
        """The type of the update received by the channel."""
        return self.typ

    def checkpoint(self) -> tuple[Value | Any, bool] | Any:
        if self.value is MISSING:
            return MISSING
        return (self.value, self.finished)

    def from_checkpoint(self, checkpoint: tuple[Value | Any, bool] | Any) -> Self:
        empty = self.__class__(self.typ)
        empty.key = self.key
        if checkpoint is not MISSING:
            empty.value, empty.finished = checkpoint
        return empty

    def update(self, values: Sequence[Value | Any]) -> bool:
        if len(values) == 0:
            return False

        self.finished = False
        self.value = values[-1]
        return True

    def consume(self) -> bool:
        if self.finished:
            self.finished = False
            self.value = MISSING
            return True

        return False

    def finish(self) -> bool:
        if not self.finished and self.value is not MISSING:
            self.finished = True
            return True
        else:
            return False

    def get(self) -> Value:
        if self.value is MISSING or not self.finished:
            raise EmptyChannelError()
        return self.value

    def is_available(self) -> bool:
        return self.value is not MISSING and self.finished

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/channels/named_barrier_value.py
```py
from collections.abc import Sequence
from typing import Generic

from typing_extensions import Self

from langgraph._internal._typing import MISSING
from langgraph.channels.base import BaseChannel, Value
from langgraph.errors import EmptyChannelError, InvalidUpdateError

__all__ = ("NamedBarrierValue", "NamedBarrierValueAfterFinish")


class NamedBarrierValue(Generic[Value], BaseChannel[Value, Value, set[Value]]):
    """A channel that waits until all named values are received before making the value available."""

    __slots__ = ("names", "seen")

    names: set[Value]
    seen: set[Value]

    def __init__(self, typ: type[Value], names: set[Value]) -> None:
        super().__init__(typ)
        self.names = names
        self.seen: set[str] = set()

    def __eq__(self, value: object) -> bool:
        return isinstance(value, NamedBarrierValue) and value.names == self.names

    @property
    def ValueType(self) -> type[Value]:
        """The type of the value stored in the channel."""
        return self.typ

    @property
    def UpdateType(self) -> type[Value]:
        """The type of the update received by the channel."""
        return self.typ

    def copy(self) -> Self:
        """Return a copy of the channel."""
        empty = self.__class__(self.typ, self.names)
        empty.key = self.key
        empty.seen = self.seen.copy()
        return empty

    def checkpoint(self) -> set[Value]:
        return self.seen

    def from_checkpoint(self, checkpoint: set[Value]) -> Self:
        empty = self.__class__(self.typ, self.names)
        empty.key = self.key
        if checkpoint is not MISSING:
            empty.seen = checkpoint
        return empty

    def update(self, values: Sequence[Value]) -> bool:
        updated = False
        for value in values:
            if value in self.names:
                if value not in self.seen:
                    self.seen.add(value)
                    updated = True
            else:
                raise InvalidUpdateError(
                    f"At key '{self.key}': Value {value} not in {self.names}"
                )
        return updated

    def get(self) -> Value:
        if self.seen != self.names:
            raise EmptyChannelError()
        return None

    def is_available(self) -> bool:
        return self.seen == self.names

    def consume(self) -> bool:
        if self.seen == self.names:
            self.seen = set()
            return True
        return False


class NamedBarrierValueAfterFinish(
    Generic[Value], BaseChannel[Value, Value, set[Value]]
):
    """A channel that waits until all named values are received before making the value ready to be made available. It is only made available after finish() is called."""

    __slots__ = ("names", "seen", "finished")

    names: set[Value]
    seen: set[Value]

    def __init__(self, typ: type[Value], names: set[Value]) -> None:
        super().__init__(typ)
        self.names = names
        self.seen: set[str] = set()
        self.finished = False

    def __eq__(self, value: object) -> bool:
        return (
            isinstance(value, NamedBarrierValueAfterFinish)
            and value.names == self.names
        )

    @property
    def ValueType(self) -> type[Value]:
        """The type of the value stored in the channel."""
        return self.typ

    @property
    def UpdateType(self) -> type[Value]:
        """The type of the update received by the channel."""
        return self.typ

    def copy(self) -> Self:
        """Return a copy of the channel."""
        empty = self.__class__(self.typ, self.names)
        empty.key = self.key
        empty.seen = self.seen.copy()
        empty.finished = self.finished
        return empty

    def checkpoint(self) -> tuple[set[Value], bool]:
        return (self.seen, self.finished)

    def from_checkpoint(self, checkpoint: tuple[set[Value], bool]) -> Self:
        empty = self.__class__(self.typ, self.names)
        empty.key = self.key
        if checkpoint is not MISSING:
            empty.seen, empty.finished = checkpoint
        return empty

    def update(self, values: Sequence[Value]) -> bool:
        updated = False
        for value in values:
            if value in self.names:
                if value not in self.seen:
                    self.seen.add(value)
                    updated = True
            else:
                raise InvalidUpdateError(
                    f"At key '{self.key}': Value {value} not in {self.names}"
                )
        return updated

    def get(self) -> Value:
        if not self.finished or self.seen != self.names:
            raise EmptyChannelError()
        return None

    def is_available(self) -> bool:
        return self.finished and self.seen == self.names

    def consume(self) -> bool:
        if self.finished and self.seen == self.names:
            self.finished = False
            self.seen = set()
            return True
        return False

    def finish(self) -> bool:
        if not self.finished and self.seen == self.names:
            self.finished = True
            return True
        else:
            return False

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/channels/topic.py
```py
from __future__ import annotations

from collections.abc import Iterator, Sequence
from typing import Any, Generic

from typing_extensions import Self

from langgraph._internal._typing import MISSING
from langgraph.channels.base import BaseChannel, Value
from langgraph.errors import EmptyChannelError

__all__ = ("Topic",)


def _flatten(values: Sequence[Value | list[Value]]) -> Iterator[Value]:
    for value in values:
        if isinstance(value, list):
            yield from value
        else:
            yield value


class Topic(
    Generic[Value],
    BaseChannel[Sequence[Value], Value | list[Value], list[Value]],
):
    """A configurable PubSub Topic.

    Args:
        typ: The type of the value stored in the channel.
        accumulate: Whether to accumulate values across steps. If `False`, the channel will be emptied after each step.
    """

    __slots__ = ("values", "accumulate")

    def __init__(self, typ: type[Value], accumulate: bool = False) -> None:
        super().__init__(typ)
        # attrs
        self.accumulate = accumulate
        # state
        self.values = list[Value]()

    def __eq__(self, value: object) -> bool:
        return isinstance(value, Topic) and value.accumulate == self.accumulate

    @property
    def ValueType(self) -> Any:
        """The type of the value stored in the channel."""
        return Sequence[self.typ]  # type: ignore[name-defined]

    @property
    def UpdateType(self) -> Any:
        """The type of the update received by the channel."""
        return self.typ | list[self.typ]  # type: ignore[name-defined]

    def copy(self) -> Self:
        """Return a copy of the channel."""
        empty = self.__class__(self.typ, self.accumulate)
        empty.key = self.key
        empty.values = self.values.copy()
        return empty

    def checkpoint(self) -> list[Value]:
        return self.values

    def from_checkpoint(self, checkpoint: list[Value]) -> Self:
        empty = self.__class__(self.typ, self.accumulate)
        empty.key = self.key
        if checkpoint is not MISSING:
            if isinstance(checkpoint, tuple):
                # backwards compatibility
                empty.values = checkpoint[1]
            else:
                empty.values = checkpoint
        return empty

    def update(self, values: Sequence[Value | list[Value]]) -> bool:
        updated = False
        if not self.accumulate:
            updated = bool(self.values)
            self.values = list[Value]()
        if flat_values := tuple(_flatten(values)):
            updated = True
            self.values.extend(flat_values)
        return updated

    def get(self) -> Sequence[Value]:
        if self.values:
            return list(self.values)
        else:
            raise EmptyChannelError

    def is_available(self) -> bool:
        return bool(self.values)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/channels/untracked_value.py
```py
from __future__ import annotations

from collections.abc import Sequence
from typing import Any, Generic

from typing_extensions import Self

from langgraph._internal._typing import MISSING
from langgraph.channels.base import BaseChannel, Value
from langgraph.errors import EmptyChannelError, InvalidUpdateError

__all__ = ("UntrackedValue",)


class UntrackedValue(Generic[Value], BaseChannel[Value, Value, Value]):
    """Stores the last value received, never checkpointed."""

    __slots__ = ("value", "guard")

    guard: bool
    value: Value | Any

    def __init__(self, typ: type[Value], guard: bool = True) -> None:
        super().__init__(typ)
        self.guard = guard
        self.value = MISSING

    def __eq__(self, value: object) -> bool:
        return isinstance(value, UntrackedValue) and value.guard == self.guard

    @property
    def ValueType(self) -> type[Value]:
        """The type of the value stored in the channel."""
        return self.typ

    @property
    def UpdateType(self) -> type[Value]:
        """The type of the update received by the channel."""
        return self.typ

    def copy(self) -> Self:
        """Return a copy of the channel."""
        empty = self.__class__(self.typ, self.guard)
        empty.key = self.key
        empty.value = self.value
        return empty

    def checkpoint(self) -> Value | Any:
        return MISSING

    def from_checkpoint(self, checkpoint: Value) -> Self:
        empty = self.__class__(self.typ, self.guard)
        empty.key = self.key
        return empty

    def update(self, values: Sequence[Value]) -> bool:
        if len(values) == 0:
            return False
        if len(values) != 1 and self.guard:
            raise InvalidUpdateError(
                f"At key '{self.key}': UntrackedValue(guard=True) can receive only one value per step. Use guard=False if you want to store any one of multiple values."
            )

        self.value = values[-1]
        return True

    def get(self) -> Value:
        if self.value is MISSING:
            raise EmptyChannelError()
        return self.value

    def is_available(self) -> bool:
        return self.value is not MISSING

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/__init__.py
```py
from langgraph.pregel.main import NodeBuilder, Pregel

__all__ = ("Pregel", "NodeBuilder")

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/_algo.py
```py
from __future__ import annotations

import binascii
import itertools
import sys
import threading
from collections import defaultdict, deque
from collections.abc import Callable, Iterable, Mapping, Sequence
from copy import copy
from functools import partial
from hashlib import sha1
from typing import (
    Any,
    Literal,
    NamedTuple,
    Protocol,
    cast,
    overload,
)

from langchain_core.callbacks import Callbacks
from langchain_core.callbacks.manager import AsyncParentRunManager, ParentRunManager
from langchain_core.runnables.config import RunnableConfig
from langgraph.checkpoint.base import (
    BaseCheckpointSaver,
    ChannelVersions,
    Checkpoint,
    PendingWrite,
    V,
)
from langgraph.store.base import BaseStore
from xxhash import xxh3_128_hexdigest

from langgraph._internal._config import merge_configs, patch_config
from langgraph._internal._constants import (
    CACHE_NS_WRITES,
    CONF,
    CONFIG_KEY_CHECKPOINT_ID,
    CONFIG_KEY_CHECKPOINT_MAP,
    CONFIG_KEY_CHECKPOINT_NS,
    CONFIG_KEY_CHECKPOINTER,
    CONFIG_KEY_READ,
    CONFIG_KEY_RESUME_MAP,
    CONFIG_KEY_RUNTIME,
    CONFIG_KEY_SCRATCHPAD,
    CONFIG_KEY_SEND,
    CONFIG_KEY_TASK_ID,
    ERROR,
    INTERRUPT,
    NO_WRITES,
    NS_END,
    NS_SEP,
    NULL_TASK_ID,
    PREVIOUS,
    PULL,
    PUSH,
    RESERVED,
    RESUME,
    RETURN,
    TASKS,
)
from langgraph._internal._scratchpad import PregelScratchpad
from langgraph._internal._typing import EMPTY_SEQ, MISSING
from langgraph.channels.base import BaseChannel
from langgraph.channels.topic import Topic
from langgraph.channels.untracked_value import UntrackedValue
from langgraph.constants import TAG_HIDDEN
from langgraph.managed.base import ManagedValueMapping
from langgraph.pregel._call import get_runnable_for_task, identifier
from langgraph.pregel._io import read_channels
from langgraph.pregel._log import logger
from langgraph.pregel._read import INPUT_CACHE_KEY_TYPE, PregelNode
from langgraph.runtime import DEFAULT_RUNTIME, Runtime
from langgraph.types import (
    All,
    CacheKey,
    CachePolicy,
    PregelExecutableTask,
    PregelTask,
    RetryPolicy,
    Send,
)

GetNextVersion = Callable[[V | None, None], V]
SUPPORTS_EXC_NOTES = sys.version_info >= (3, 11)


class WritesProtocol(Protocol):
    """Protocol for objects containing writes to be applied to checkpoint.
    Implemented by PregelTaskWrites and PregelExecutableTask."""

    @property
    def path(self) -> tuple[str | int | tuple, ...]: ...

    @property
    def name(self) -> str: ...

    @property
    def writes(self) -> Sequence[tuple[str, Any]]: ...

    @property
    def triggers(self) -> Sequence[str]: ...


class PregelTaskWrites(NamedTuple):
    """Simplest implementation of WritesProtocol, for usage with writes that
    don't originate from a runnable task, eg. graph input, update_state, etc."""

    path: tuple[str | int | tuple, ...]
    name: str
    writes: Sequence[tuple[str, Any]]
    triggers: Sequence[str]


class Call:
    __slots__ = ("func", "input", "retry_policy", "cache_policy", "callbacks")

    func: Callable
    input: tuple[tuple[Any, ...], dict[str, Any]]
    retry_policy: Sequence[RetryPolicy] | None
    cache_policy: CachePolicy | None
    callbacks: Callbacks

    def __init__(
        self,
        func: Callable,
        input: tuple[tuple[Any, ...], dict[str, Any]],
        *,
        retry_policy: Sequence[RetryPolicy] | None,
        cache_policy: CachePolicy | None,
        callbacks: Callbacks,
    ) -> None:
        self.func = func
        self.input = input
        self.retry_policy = retry_policy
        self.cache_policy = cache_policy
        self.callbacks = callbacks


def should_interrupt(
    checkpoint: Checkpoint,
    interrupt_nodes: All | Sequence[str],
    tasks: Iterable[PregelExecutableTask],
) -> list[PregelExecutableTask]:
    """Check if the graph should be interrupted based on current state."""
    version_type = type(next(iter(checkpoint["channel_versions"].values()), None))
    null_version = version_type()  # type: ignore[misc]
    seen = checkpoint["versions_seen"].get(INTERRUPT, {})
    # interrupt if any channel has been updated since last interrupt
    any_updates_since_prev_interrupt = any(
        version > seen.get(chan, null_version)  # type: ignore[operator]
        for chan, version in checkpoint["channel_versions"].items()
    )
    # and any triggered node is in interrupt_nodes list
    return (
        [
            task
            for task in tasks
            if (
                (
                    not task.config
                    or TAG_HIDDEN not in task.config.get("tags", EMPTY_SEQ)
                )
                if interrupt_nodes == "*"
                else task.name in interrupt_nodes
            )
        ]
        if any_updates_since_prev_interrupt
        else []
    )


def local_read(
    scratchpad: PregelScratchpad,
    channels: Mapping[str, BaseChannel],
    managed: ManagedValueMapping,
    task: WritesProtocol,
    select: list[str] | str,
    fresh: bool = False,
) -> dict[str, Any] | Any:
    """Function injected under CONFIG_KEY_READ in task config, to read current state.
    Used by conditional edges to read a copy of the state with reflecting the writes
    from that node only."""
    updated: dict[str, list[Any]] = defaultdict(list)
    if isinstance(select, str):
        managed_keys = []
        for c, v in task.writes:
            if c == select:
                updated[c].append(v)
    else:
        managed_keys = [k for k in select if k in managed]
        select = [k for k in select if k not in managed]
        for c, v in task.writes:
            if c in select:
                updated[c].append(v)
    if fresh:
        # apply writes
        local_channels: dict[str, BaseChannel] = {}
        for k in channels:
            cc = channels[k].copy()
            cc.update(updated[k])
            local_channels[k] = cc
        # read fresh values
        values = read_channels(local_channels, select)
    else:
        values = read_channels(channels, select)
    if managed_keys:
        values.update({k: managed[k].get(scratchpad) for k in managed_keys})
    return values


def increment(current: int | None, channel: None) -> int:
    """Default channel versioning function, increments the current int version."""
    return current + 1 if current is not None else 1


def apply_writes(
    checkpoint: Checkpoint,
    channels: Mapping[str, BaseChannel],
    tasks: Iterable[WritesProtocol],
    get_next_version: GetNextVersion | None,
    trigger_to_nodes: Mapping[str, Sequence[str]],
) -> set[str]:
    """Apply writes from a set of tasks (usually the tasks from a Pregel step)
    to the checkpoint and channels, and return managed values writes to be applied
    externally.

    Args:
        checkpoint: The checkpoint to update.
        channels: The channels to update.
        tasks: The tasks to apply writes from.
        get_next_version: Optional function to determine the next version of a channel.
        trigger_to_nodes: Mapping of channel names to the set of nodes that can be triggered by updates to that channel.

    Returns:
        Set of channels that were updated in this step.
    """
    # sort tasks on path, to ensure deterministic order for update application
    # any path parts after the 3rd are ignored for sorting
    # (we use them for eg. task ids which aren't good for sorting)
    tasks = sorted(tasks, key=lambda t: task_path_str(t.path[:3]))
    # if no task has triggers this is applying writes from the null task only
    # so we don't do anything other than update the channels written to
    bump_step = any(t.triggers for t in tasks)

    # update seen versions
    for task in tasks:
        checkpoint["versions_seen"].setdefault(task.name, {}).update(
            {
                chan: checkpoint["channel_versions"][chan]
                for chan in task.triggers
                if chan in checkpoint["channel_versions"]
            }
        )

    # Find the highest version of all channels
    if get_next_version is None:
        next_version = None
    else:
        next_version = get_next_version(
            max(checkpoint["channel_versions"].values())
            if checkpoint["channel_versions"]
            else None,
            None,
        )

    # Consume all channels that were read
    for chan in {
        chan
        for task in tasks
        for chan in task.triggers
        if chan not in RESERVED and chan in channels
    }:
        if channels[chan].consume() and next_version is not None:
            checkpoint["channel_versions"][chan] = next_version

    # Group writes by channel
    pending_writes_by_channel: dict[str, list[Any]] = defaultdict(list)
    for task in tasks:
        for chan, val in task.writes:
            if chan in (NO_WRITES, PUSH, RESUME, INTERRUPT, RETURN, ERROR):
                pass
            elif chan in channels:
                pending_writes_by_channel[chan].append(val)
            else:
                logger.warning(
                    f"Task {task.name} with path {task.path} wrote to unknown channel {chan}, ignoring it."
                )

    # Apply writes to channels
    updated_channels: set[str] = set()
    for chan, vals in pending_writes_by_channel.items():
        if chan in channels:
            if channels[chan].update(vals) and next_version is not None:
                checkpoint["channel_versions"][chan] = next_version
                # unavailable channels can't trigger tasks, so don't add them
                if channels[chan].is_available():
                    updated_channels.add(chan)

    # Channels that weren't updated in this step are notified of a new step
    if bump_step:
        for chan in channels:
            if channels[chan].is_available() and chan not in updated_channels:
                if channels[chan].update(EMPTY_SEQ) and next_version is not None:
                    checkpoint["channel_versions"][chan] = next_version
                    # unavailable channels can't trigger tasks, so don't add them
                    if channels[chan].is_available():
                        updated_channels.add(chan)

    # If this is (tentatively) the last superstep, notify all channels of finish
    if bump_step and updated_channels.isdisjoint(trigger_to_nodes):
        for chan in channels:
            if channels[chan].finish() and next_version is not None:
                checkpoint["channel_versions"][chan] = next_version
                # unavailable channels can't trigger tasks, so don't add them
                if channels[chan].is_available():
                    updated_channels.add(chan)

    # Return managed values writes to be applied externally
    return updated_channels


@overload
def prepare_next_tasks(
    checkpoint: Checkpoint,
    pending_writes: list[PendingWrite],
    processes: Mapping[str, PregelNode],
    channels: Mapping[str, BaseChannel],
    managed: ManagedValueMapping,
    config: RunnableConfig,
    step: int,
    stop: int,
    *,
    for_execution: Literal[False],
    store: Literal[None] = None,
    checkpointer: Literal[None] = None,
    manager: Literal[None] = None,
    trigger_to_nodes: Mapping[str, Sequence[str]] | None = None,
    updated_channels: set[str] | None = None,
    retry_policy: Sequence[RetryPolicy] = (),
    cache_policy: Literal[None] = None,
) -> dict[str, PregelTask]: ...


@overload
def prepare_next_tasks(
    checkpoint: Checkpoint,
    pending_writes: list[PendingWrite],
    processes: Mapping[str, PregelNode],
    channels: Mapping[str, BaseChannel],
    managed: ManagedValueMapping,
    config: RunnableConfig,
    step: int,
    stop: int,
    *,
    for_execution: Literal[True],
    store: BaseStore | None,
    checkpointer: BaseCheckpointSaver | None,
    manager: None | ParentRunManager | AsyncParentRunManager,
    trigger_to_nodes: Mapping[str, Sequence[str]] | None = None,
    updated_channels: set[str] | None = None,
    retry_policy: Sequence[RetryPolicy] = (),
    cache_policy: CachePolicy | None = None,
) -> dict[str, PregelExecutableTask]: ...


def prepare_next_tasks(
    checkpoint: Checkpoint,
    pending_writes: list[PendingWrite],
    processes: Mapping[str, PregelNode],
    channels: Mapping[str, BaseChannel],
    managed: ManagedValueMapping,
    config: RunnableConfig,
    step: int,
    stop: int,
    *,
    for_execution: bool,
    store: BaseStore | None = None,
    checkpointer: BaseCheckpointSaver | None = None,
    manager: None | ParentRunManager | AsyncParentRunManager = None,
    trigger_to_nodes: Mapping[str, Sequence[str]] | None = None,
    updated_channels: set[str] | None = None,
    retry_policy: Sequence[RetryPolicy] = (),
    cache_policy: CachePolicy | None = None,
) -> dict[str, PregelTask] | dict[str, PregelExecutableTask]:
    """Prepare the set of tasks that will make up the next Pregel step.

    Args:
        checkpoint: The current checkpoint.
        pending_writes: The list of pending writes.
        processes: The mapping of process names to PregelNode instances.
        channels: The mapping of channel names to BaseChannel instances.
        managed: The mapping of managed value names to functions.
        config: The `Runnable` configuration.
        step: The current step.
        for_execution: Whether the tasks are being prepared for execution.
        store: An instance of BaseStore to make it available for usage within tasks.
        checkpointer: `Checkpointer` instance used for saving checkpoints.
        manager: The parent run manager to use for the tasks.
        trigger_to_nodes: Optional: Mapping of channel names to the set of nodes
            that are can be triggered by that channel.
        updated_channels: Optional. Set of channel names that have been updated during
            the previous step. Using in conjunction with trigger_to_nodes to speed
            up the process of determining which nodes should be triggered in the next
            step.

    Returns:
        A dictionary of tasks to be executed. The keys are the task ids and the values
        are the tasks themselves. This is the union of all PUSH tasks (Sends)
        and PULL tasks (nodes triggered by edges).
    """
    input_cache: dict[INPUT_CACHE_KEY_TYPE, Any] = {}
    checkpoint_id_bytes = binascii.unhexlify(checkpoint["id"].replace("-", ""))
    null_version = checkpoint_null_version(checkpoint)
    tasks: list[PregelTask | PregelExecutableTask] = []
    # Consume pending tasks
    tasks_channel = cast(Topic[Send] | None, channels.get(TASKS))
    if tasks_channel and tasks_channel.is_available():
        for idx, _ in enumerate(tasks_channel.get()):
            if task := prepare_single_task(
                (PUSH, idx),
                None,
                checkpoint=checkpoint,
                checkpoint_id_bytes=checkpoint_id_bytes,
                checkpoint_null_version=null_version,
                pending_writes=pending_writes,
                processes=processes,
                channels=channels,
                managed=managed,
                config=config,
                step=step,
                stop=stop,
                for_execution=for_execution,
                store=store,
                checkpointer=checkpointer,
                manager=manager,
                input_cache=input_cache,
                cache_policy=cache_policy,
                retry_policy=retry_policy,
            ):
                tasks.append(task)

    # This section is an optimization that allows which nodes will be active
    # during the next step.
    # When there's information about:
    # 1. Which channels were updated in the previous step
    # 2. Which nodes are triggered by which channels
    # Then we can determine which nodes should be triggered in the next step
    # without having to cycle through all nodes.
    if updated_channels and trigger_to_nodes:
        triggered_nodes: set[str] = set()
        # Get all nodes that have triggers associated with an updated channel
        for channel in updated_channels:
            if node_ids := trigger_to_nodes.get(channel):
                triggered_nodes.update(node_ids)
        # Sort the nodes to ensure deterministic order
        candidate_nodes: Iterable[str] = sorted(triggered_nodes)
    elif not checkpoint["channel_versions"]:
        candidate_nodes = ()
    else:
        candidate_nodes = processes.keys()

    # Check if any processes should be run in next step
    # If so, prepare the values to be passed to them
    for name in candidate_nodes:
        if task := prepare_single_task(
            (PULL, name),
            None,
            checkpoint=checkpoint,
            checkpoint_id_bytes=checkpoint_id_bytes,
            checkpoint_null_version=null_version,
            pending_writes=pending_writes,
            processes=processes,
            channels=channels,
            managed=managed,
            config=config,
            step=step,
            stop=stop,
            for_execution=for_execution,
            store=store,
            checkpointer=checkpointer,
            manager=manager,
            input_cache=input_cache,
            cache_policy=cache_policy,
            retry_policy=retry_policy,
        ):
            tasks.append(task)
    return {t.id: t for t in tasks}


PUSH_TRIGGER = (PUSH,)


def prepare_single_task(
    task_path: tuple[Any, ...],
    task_id_checksum: str | None,
    *,
    checkpoint: Checkpoint,
    checkpoint_id_bytes: bytes,
    checkpoint_null_version: V | None,
    pending_writes: list[PendingWrite],
    processes: Mapping[str, PregelNode],
    channels: Mapping[str, BaseChannel],
    managed: ManagedValueMapping,
    config: RunnableConfig,
    step: int,
    stop: int,
    for_execution: bool,
    store: BaseStore | None = None,
    checkpointer: BaseCheckpointSaver | None = None,
    manager: None | ParentRunManager | AsyncParentRunManager = None,
    input_cache: dict[INPUT_CACHE_KEY_TYPE, Any] | None = None,
    cache_policy: CachePolicy | None = None,
    retry_policy: Sequence[RetryPolicy] = (),
) -> None | PregelTask | PregelExecutableTask:
    """Prepares a single task for the next Pregel step, given a task path, which
    uniquely identifies a PUSH or PULL task within the graph."""
    configurable = config.get(CONF, {})
    parent_ns = configurable.get(CONFIG_KEY_CHECKPOINT_NS, "")
    task_id_func = _xxhash_str if checkpoint["v"] > 1 else _uuid5_str

    if task_path[0] == PUSH and isinstance(task_path[-1], Call):
        # (PUSH, parent task path, idx of PUSH write, id of parent task, Call)
        task_path_t = cast(tuple[str, tuple, int, str, Call], task_path)
        call = task_path_t[-1]
        proc_ = get_runnable_for_task(call.func)
        name = proc_.name
        if name is None:
            raise ValueError("`call` functions must have a `__name__` attribute")
        # create task id
        triggers: Sequence[str] = PUSH_TRIGGER
        checkpoint_ns = f"{parent_ns}{NS_SEP}{name}" if parent_ns else name
        task_id = task_id_func(
            checkpoint_id_bytes,
            checkpoint_ns,
            str(step),
            name,
            PUSH,
            task_path_str(task_path[1]),
            str(task_path[2]),
        )
        task_checkpoint_ns = f"{checkpoint_ns}:{task_id}"
        # we append True to the task path to indicate that a call is being
        # made, so we should not return interrupts from this task (responsibility lies with the parent)
        task_path = (*task_path[:3], True)
        metadata = {
            "langgraph_step": step,
            "langgraph_node": name,
            "langgraph_triggers": triggers,
            "langgraph_path": task_path,
            "langgraph_checkpoint_ns": task_checkpoint_ns,
        }
        if task_id_checksum is not None:
            assert task_id == task_id_checksum, f"{task_id} != {task_id_checksum}"
        if for_execution:
            writes: deque[tuple[str, Any]] = deque()
            cache_policy = call.cache_policy or cache_policy
            if cache_policy:
                args_key = cache_policy.key_func(*call.input[0], **call.input[1])
                cache_key: CacheKey | None = CacheKey(
                    (
                        CACHE_NS_WRITES,
                        (identifier(call.func) or "__dynamic__"),
                    ),
                    xxh3_128_hexdigest(
                        args_key.encode() if isinstance(args_key, str) else args_key,
                    ),
                    cache_policy.ttl,
                )
            else:
                cache_key = None
            scratchpad = _scratchpad(
                config[CONF].get(CONFIG_KEY_SCRATCHPAD),
                pending_writes,
                task_id,
                xxh3_128_hexdigest(task_checkpoint_ns.encode()),
                config[CONF].get(CONFIG_KEY_RESUME_MAP),
                step,
                stop,
            )
            runtime = cast(
                Runtime, configurable.get(CONFIG_KEY_RUNTIME, DEFAULT_RUNTIME)
            )
            runtime = runtime.override(store=store)
            return PregelExecutableTask(
                name,
                call.input,
                proc_,
                writes,
                patch_config(
                    merge_configs(config, {"metadata": metadata}),
                    run_name=name,
                    callbacks=call.callbacks
                    or (manager.get_child(f"graph:step:{step}") if manager else None),
                    configurable={
                        CONFIG_KEY_TASK_ID: task_id,
                        # deque.extend is thread-safe
                        CONFIG_KEY_SEND: writes.extend,
                        CONFIG_KEY_READ: partial(
                            local_read,
                            scratchpad,
                            channels,
                            managed,
                            PregelTaskWrites(task_path, name, writes, triggers),
                        ),
                        CONFIG_KEY_CHECKPOINTER: (
                            checkpointer or configurable.get(CONFIG_KEY_CHECKPOINTER)
                        ),
                        CONFIG_KEY_CHECKPOINT_MAP: {
                            **configurable.get(CONFIG_KEY_CHECKPOINT_MAP, {}),
                            parent_ns: checkpoint["id"],
                        },
                        CONFIG_KEY_CHECKPOINT_ID: None,
                        CONFIG_KEY_CHECKPOINT_NS: task_checkpoint_ns,
                        CONFIG_KEY_SCRATCHPAD: scratchpad,
                        CONFIG_KEY_RUNTIME: runtime,
                    },
                ),
                triggers,
                call.retry_policy or retry_policy,
                cache_key,
                task_id,
                task_path,
            )
        else:
            return PregelTask(task_id, name, task_path)
    elif task_path[0] == PUSH:
        if len(task_path) == 2:
            # SEND tasks, executed in superstep n+1
            # (PUSH, idx of pending send)
            idx = cast(int, task_path[1])
            if not channels[TASKS].is_available():
                return
            sends: Sequence[Send] = channels[TASKS].get()
            if idx < 0 or idx >= len(sends):
                return
            packet = sends[idx]
            if not isinstance(packet, Send):
                logger.warning(
                    f"Ignoring invalid packet type {type(packet)} in pending sends"
                )
                return

            if packet.node not in processes:
                logger.warning(
                    f"Ignoring unknown node name {packet.node} in pending sends"
                )
                return
            # find process
            proc = processes[packet.node]
            proc_node = proc.node
            if proc_node is None:
                return
            # create task id
            triggers = PUSH_TRIGGER
            checkpoint_ns = (
                f"{parent_ns}{NS_SEP}{packet.node}" if parent_ns else packet.node
            )
            task_id = task_id_func(
                checkpoint_id_bytes,
                checkpoint_ns,
                str(step),
                packet.node,
                PUSH,
                str(idx),
            )
        else:
            logger.warning(f"Ignoring invalid PUSH task path {task_path}")
            return
        task_checkpoint_ns = f"{checkpoint_ns}:{task_id}"
        # we append False to the task path to indicate that a call is not being made
        # so we should return interrupts from this task
        task_path = (*task_path[:3], False)
        metadata = {
            "langgraph_step": step,
            "langgraph_node": packet.node,
            "langgraph_triggers": triggers,
            "langgraph_path": task_path,
            "langgraph_checkpoint_ns": task_checkpoint_ns,
        }
        if task_id_checksum is not None:
            assert task_id == task_id_checksum, f"{task_id} != {task_id_checksum}"
        if for_execution:
            if proc.metadata:
                metadata.update(proc.metadata)
            writes = deque()
            cache_policy = proc.cache_policy or cache_policy
            if cache_policy:
                args_key = cache_policy.key_func(packet.arg)
                cache_key = CacheKey(
                    (
                        CACHE_NS_WRITES,
                        (identifier(proc) or "__dynamic__"),
                        packet.node,
                    ),
                    xxh3_128_hexdigest(
                        args_key.encode() if isinstance(args_key, str) else args_key,
                    ),
                    cache_policy.ttl,
                )
            else:
                cache_key = None
            scratchpad = _scratchpad(
                config[CONF].get(CONFIG_KEY_SCRATCHPAD),
                pending_writes,
                task_id,
                xxh3_128_hexdigest(task_checkpoint_ns.encode()),
                config[CONF].get(CONFIG_KEY_RESUME_MAP),
                step,
                stop,
            )
            runtime = cast(
                Runtime, configurable.get(CONFIG_KEY_RUNTIME, DEFAULT_RUNTIME)
            )
            runtime = runtime.override(
                store=store, previous=checkpoint["channel_values"].get(PREVIOUS, None)
            )
            additional_config: RunnableConfig = {
                "metadata": metadata,
                "tags": proc.tags,
            }
            return PregelExecutableTask(
                packet.node,
                packet.arg,
                proc_node,
                writes,
                patch_config(
                    merge_configs(config, additional_config),
                    run_name=packet.node,
                    callbacks=(
                        manager.get_child(f"graph:step:{step}") if manager else None
                    ),
                    configurable={
                        CONFIG_KEY_TASK_ID: task_id,
                        # deque.extend is thread-safe
                        CONFIG_KEY_SEND: writes.extend,
                        CONFIG_KEY_READ: partial(
                            local_read,
                            scratchpad,
                            channels,
                            managed,
                            PregelTaskWrites(task_path, packet.node, writes, triggers),
                        ),
                        CONFIG_KEY_CHECKPOINTER: (
                            checkpointer or configurable.get(CONFIG_KEY_CHECKPOINTER)
                        ),
                        CONFIG_KEY_CHECKPOINT_MAP: {
                            **configurable.get(CONFIG_KEY_CHECKPOINT_MAP, {}),
                            parent_ns: checkpoint["id"],
                        },
                        CONFIG_KEY_CHECKPOINT_ID: None,
                        CONFIG_KEY_CHECKPOINT_NS: task_checkpoint_ns,
                        CONFIG_KEY_SCRATCHPAD: scratchpad,
                        CONFIG_KEY_RUNTIME: runtime,
                    },
                ),
                triggers,
                proc.retry_policy or retry_policy,
                cache_key,
                task_id,
                task_path,
                writers=proc.flat_writers,
                subgraphs=proc.subgraphs,
            )
        else:
            return PregelTask(task_id, packet.node, task_path)
    elif task_path[0] == PULL:
        # (PULL, node name)
        name = cast(str, task_path[1])
        if name not in processes:
            return
        proc = processes[name]
        if checkpoint_null_version is None:
            return
        # If any of the channels read by this process were updated
        if _triggers(
            channels,
            checkpoint["channel_versions"],
            checkpoint["versions_seen"].get(name),
            checkpoint_null_version,
            proc,
        ):
            triggers = tuple(sorted(proc.triggers))
            # create task id
            checkpoint_ns = f"{parent_ns}{NS_SEP}{name}" if parent_ns else name
            task_id = task_id_func(
                checkpoint_id_bytes,
                checkpoint_ns,
                str(step),
                name,
                PULL,
                *triggers,
            )
            task_checkpoint_ns = f"{checkpoint_ns}{NS_END}{task_id}"
            # create scratchpad
            scratchpad = _scratchpad(
                config[CONF].get(CONFIG_KEY_SCRATCHPAD),
                pending_writes,
                task_id,
                xxh3_128_hexdigest(task_checkpoint_ns.encode()),
                config[CONF].get(CONFIG_KEY_RESUME_MAP),
                step,
                stop,
            )
            # create task input
            try:
                val = _proc_input(
                    proc,
                    managed,
                    channels,
                    for_execution=for_execution,
                    input_cache=input_cache,
                    scratchpad=scratchpad,
                )
                if val is MISSING:
                    return
            except Exception as exc:
                if SUPPORTS_EXC_NOTES:
                    exc.add_note(
                        f"Before task with name '{name}' and path '{task_path[:3]}'"
                    )
                raise

            metadata = {
                "langgraph_step": step,
                "langgraph_node": name,
                "langgraph_triggers": triggers,
                "langgraph_path": task_path[:3],
                "langgraph_checkpoint_ns": task_checkpoint_ns,
            }
            if task_id_checksum is not None:
                assert task_id == task_id_checksum, f"{task_id} != {task_id_checksum}"
            if for_execution:
                if node := proc.node:
                    if proc.metadata:
                        metadata.update(proc.metadata)
                    writes = deque()
                    cache_policy = proc.cache_policy or cache_policy
                    if cache_policy:
                        args_key = cache_policy.key_func(val)
                        cache_key = CacheKey(
                            (
                                CACHE_NS_WRITES,
                                (identifier(proc) or "__dynamic__"),
                                name,
                            ),
                            xxh3_128_hexdigest(
                                args_key.encode()
                                if isinstance(args_key, str)
                                else args_key,
                            ),
                            cache_policy.ttl,
                        )
                    else:
                        cache_key = None
                    runtime = cast(
                        Runtime, configurable.get(CONFIG_KEY_RUNTIME, DEFAULT_RUNTIME)
                    )
                    runtime = runtime.override(
                        previous=checkpoint["channel_values"].get(PREVIOUS, None),
                        store=store,
                    )
                    additional_config = {
                        "metadata": metadata,
                        "tags": proc.tags,
                    }
                    return PregelExecutableTask(
                        name,
                        val,
                        node,
                        writes,
                        patch_config(
                            merge_configs(config, additional_config),
                            run_name=name,
                            callbacks=(
                                manager.get_child(f"graph:step:{step}")
                                if manager
                                else None
                            ),
                            configurable={
                                CONFIG_KEY_TASK_ID: task_id,
                                # deque.extend is thread-safe
                                CONFIG_KEY_SEND: writes.extend,
                                CONFIG_KEY_READ: partial(
                                    local_read,
                                    scratchpad,
                                    channels,
                                    managed,
                                    PregelTaskWrites(
                                        task_path[:3],
                                        name,
                                        writes,
                                        triggers,
                                    ),
                                ),
                                CONFIG_KEY_CHECKPOINTER: (
                                    checkpointer
                                    or configurable.get(CONFIG_KEY_CHECKPOINTER)
                                ),
                                CONFIG_KEY_CHECKPOINT_MAP: {
                                    **configurable.get(CONFIG_KEY_CHECKPOINT_MAP, {}),
                                    parent_ns: checkpoint["id"],
                                },
                                CONFIG_KEY_CHECKPOINT_ID: None,
                                CONFIG_KEY_CHECKPOINT_NS: task_checkpoint_ns,
                                CONFIG_KEY_SCRATCHPAD: scratchpad,
                                CONFIG_KEY_RUNTIME: runtime,
                            },
                        ),
                        triggers,
                        proc.retry_policy or retry_policy,
                        cache_key,
                        task_id,
                        task_path[:3],
                        writers=proc.flat_writers,
                        subgraphs=proc.subgraphs,
                    )
            else:
                return PregelTask(task_id, name, task_path[:3])


def checkpoint_null_version(
    checkpoint: Checkpoint,
) -> V | None:
    """Get the null version for the checkpoint, if available."""
    for version in checkpoint["channel_versions"].values():
        return type(version)()
    return None


def _triggers(
    channels: Mapping[str, BaseChannel],
    versions: ChannelVersions,
    seen: ChannelVersions | None,
    null_version: V,
    proc: PregelNode,
) -> bool:
    if seen is None:
        for chan in proc.triggers:
            if channels[chan].is_available():
                return True
    else:
        for chan in proc.triggers:
            if channels[chan].is_available() and versions.get(  # type: ignore[operator]
                chan, null_version
            ) > seen.get(chan, null_version):
                return True
    return False


def _scratchpad(
    parent_scratchpad: PregelScratchpad | None,
    pending_writes: list[PendingWrite],
    task_id: str,
    namespace_hash: str,
    resume_map: dict[str, Any] | None,
    step: int,
    stop: int,
) -> PregelScratchpad:
    if len(pending_writes) > 0:
        # find global resume value
        for w in pending_writes:
            if w[0] == NULL_TASK_ID and w[1] == RESUME:
                null_resume_write = w
                break
        else:
            # None cannot be used as a resume value, because it would be difficult to
            # distinguish from missing when used over http
            null_resume_write = None

        # find task-specific resume value
        for w in pending_writes:
            if w[0] == task_id and w[1] == RESUME:
                task_resume_write = w[2]
                if not isinstance(task_resume_write, list):
                    task_resume_write = [task_resume_write]
                break
        else:
            task_resume_write = []
        del w

        # find namespace and task-specific resume value
        if resume_map and namespace_hash in resume_map:
            mapped_resume_write = resume_map[namespace_hash]
            task_resume_write.append(mapped_resume_write)

    else:
        null_resume_write = None
        task_resume_write = []

    def get_null_resume(consume: bool = False) -> Any:
        if null_resume_write is None:
            if parent_scratchpad is not None:
                return parent_scratchpad.get_null_resume(consume)
            return None
        if consume:
            try:
                pending_writes.remove(null_resume_write)
                return null_resume_write[2]
            except ValueError:
                return None
        return null_resume_write[2]

    # using itertools.count as an atomic counter (+= 1 is not thread-safe)
    return PregelScratchpad(
        step=step,
        stop=stop,
        # call
        call_counter=LazyAtomicCounter(),
        # interrupt
        interrupt_counter=LazyAtomicCounter(),
        resume=task_resume_write,
        get_null_resume=get_null_resume,
        # subgraph
        subgraph_counter=LazyAtomicCounter(),
    )


def _proc_input(
    proc: PregelNode,
    managed: ManagedValueMapping,
    channels: Mapping[str, BaseChannel],
    *,
    for_execution: bool,
    scratchpad: PregelScratchpad,
    input_cache: dict[INPUT_CACHE_KEY_TYPE, Any] | None,
) -> Any:
    """Prepare input for a PULL task, based on the process's channels and triggers."""
    # if in cache return shallow copy
    if input_cache is not None and proc.input_cache_key in input_cache:
        return copy(input_cache[proc.input_cache_key])
    # If all trigger channels subscribed by this process are not empty
    # then invoke the process with the values of all non-empty channels
    if isinstance(proc.channels, list):
        val: dict[str, Any] = {}
        for chan in proc.channels:
            if chan in channels:
                if channels[chan].is_available():
                    val[chan] = channels[chan].get()
            else:
                val[chan] = managed[chan].get(scratchpad)
    elif isinstance(proc.channels, str):
        if proc.channels in channels:
            if channels[proc.channels].is_available():
                val = channels[proc.channels].get()
            else:
                return MISSING
        else:
            return MISSING
    else:
        raise RuntimeError(
            f"Invalid channels type, expected list or dict, got {proc.channels}"
        )

    # If the process has a mapper, apply it to the value
    if for_execution and proc.mapper is not None:
        val = proc.mapper(val)

    # Cache the input value
    if input_cache is not None:
        input_cache[proc.input_cache_key] = val

    return val


def _uuid5_str(namespace: bytes, *parts: str | bytes) -> str:
    """Generate a UUID from the SHA-1 hash of a namespace and str parts."""

    sha = sha1(namespace, usedforsecurity=False)
    sha.update(b"".join(p.encode() if isinstance(p, str) else p for p in parts))
    hex = sha.hexdigest()
    return f"{hex[:8]}-{hex[8:12]}-{hex[12:16]}-{hex[16:20]}-{hex[20:32]}"


def _xxhash_str(namespace: bytes, *parts: str | bytes) -> str:
    """Generate a UUID from the XXH3 hash of a namespace and str parts."""
    hex = xxh3_128_hexdigest(
        namespace + b"".join(p.encode() if isinstance(p, str) else p for p in parts)
    )
    return f"{hex[:8]}-{hex[8:12]}-{hex[12:16]}-{hex[16:20]}-{hex[20:32]}"


def task_path_str(tup: str | int | tuple) -> str:
    """Generate a string representation of the task path."""
    return (
        f"~{', '.join(task_path_str(x) for x in tup)}"
        if isinstance(tup, (tuple, list))
        else f"{tup:010d}"
        if isinstance(tup, int)
        else str(tup)
    )


LAZY_ATOMIC_COUNTER_LOCK = threading.Lock()


class LazyAtomicCounter:
    __slots__ = ("_counter",)

    _counter: Callable[[], int] | None

    def __init__(self) -> None:
        self._counter = None

    def __call__(self) -> int:
        if self._counter is None:
            with LAZY_ATOMIC_COUNTER_LOCK:
                if self._counter is None:
                    self._counter = itertools.count(0).__next__
        return self._counter()


def sanitize_untracked_values_in_send(
    packet: Send, channels: Mapping[str, BaseChannel]
) -> Send:
    """Pop any values belonging to UntrackedValue channels in Send.arg for safe checkpointing.

    Send is often called with state to be passed to the dest node, which may contain
    UntrackedValues at the top level. Send is not typed and arg may be a nested dict."""

    if not isinstance(packet.arg, dict):
        # Command
        return packet

    # top level keys should be the channel names
    sanitized_arg = {
        k: v
        for k, v in packet.arg.items()
        if not isinstance(channels.get(k), UntrackedValue)
    }
    return Send(node=packet.node, arg=sanitized_arg)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/_call.py
```py
"""Utility to convert a user provided function into a Runnable with a ChannelWrite."""

from __future__ import annotations

import concurrent.futures
import functools
import inspect
import sys
import types
from collections.abc import Awaitable, Callable, Generator, Sequence
from typing import Any, Generic, TypeVar, cast

from langchain_core.runnables import Runnable
from typing_extensions import ParamSpec

from langgraph._internal._constants import CONF, CONFIG_KEY_CALL, RETURN
from langgraph._internal._runnable import (
    RunnableCallable,
    RunnableSeq,
    is_async_callable,
    run_in_executor,
)
from langgraph.config import get_config
from langgraph.pregel._write import ChannelWrite, ChannelWriteEntry
from langgraph.types import CachePolicy, RetryPolicy

##
# Utilities borrowed from cloudpickle.
# https://github.com/cloudpipe/cloudpickle/blob/6220b0ce83ffee5e47e06770a1ee38ca9e47c850/cloudpickle/cloudpickle.py#L265


def _getattribute(obj: Any, name: str) -> Any:
    parent = None
    for subpath in name.split("."):
        if subpath == "<locals>":
            raise AttributeError(f"Can't get local attribute {name!r} on {obj!r}")
        try:
            parent = obj
            obj = getattr(obj, subpath)
        except AttributeError:
            raise AttributeError(f"Can't get attribute {name!r} on {obj!r}") from None
    return obj, parent


def _whichmodule(obj: Any, name: str) -> str | None:
    """Find the module an object belongs to.

    This function differs from ``pickle.whichmodule`` in two ways:
    - it does not mangle the cases where obj's module is __main__ and obj was
      not found in any module.
    - Errors arising during module introspection are ignored, as those errors
      are considered unwanted side effects.
    """
    module_name = getattr(obj, "__module__", None)

    if module_name is not None:
        return module_name
    # Protect the iteration by using a copy of sys.modules against dynamic
    # modules that trigger imports of other modules upon calls to getattr or
    # other threads importing at the same time.
    for module_name, module in sys.modules.copy().items():
        # Some modules such as coverage can inject non-module objects inside
        # sys.modules
        if (
            module_name == "__main__"
            or module_name == "__mp_main__"
            or module is None
            or not isinstance(module, types.ModuleType)
        ):
            continue
        try:
            if _getattribute(module, name)[0] is obj:
                return module_name
        except Exception:
            pass
    return None


def identifier(obj: Any, name: str | None = None) -> str | None:
    """Return the module and name of an object."""
    from langgraph._internal._runnable import RunnableCallable, RunnableSeq
    from langgraph.pregel._read import PregelNode

    if isinstance(obj, PregelNode):
        obj = obj.bound
    if isinstance(obj, RunnableSeq):
        obj = obj.steps[0]
    if isinstance(obj, RunnableCallable):
        obj = obj.func
    if name is None:
        name = getattr(obj, "__qualname__", None)
    if name is None:  # pragma: no cover
        # This used to be needed for Python 2.7 support but is probably not
        # needed anymore. However we keep the __name__ introspection in case
        # users of cloudpickle rely on this old behavior for unknown reasons.
        name = getattr(obj, "__name__", None)
    if name is None:
        return None

    module_name = getattr(obj, "__module__", None)
    if module_name is None:
        # In this case, obj.__module__ is None. obj is thus treated as dynamic.
        return None

    return f"{module_name}.{name}"


def _lookup_module_and_qualname(
    obj: Any, name: str | None = None
) -> tuple[types.ModuleType, str] | None:
    if name is None:
        name = getattr(obj, "__qualname__", None)
    if name is None:  # pragma: no cover
        # This used to be needed for Python 2.7 support but is probably not
        # needed anymore. However we keep the __name__ introspection in case
        # users of cloudpickle rely on this old behavior for unknown reasons.
        name = getattr(obj, "__name__", None)
    if name is None:
        return None

    module_name = _whichmodule(obj, name)

    if module_name is None:
        # In this case, obj.__module__ is None AND obj was not found in any
        # imported module. obj is thus treated as dynamic.
        return None

    if module_name == "__main__":
        return None

    # Note: if module_name is in sys.modules, the corresponding module is
    # assumed importable at unpickling time. See #357
    module = sys.modules.get(module_name, None)
    if module is None:
        # The main reason why obj's module would not be imported is that this
        # module has been dynamically created, using for example
        # types.ModuleType. The other possibility is that module was removed
        # from sys.modules after obj was created/imported. But this case is not
        # supported, as the standard pickle does not support it either.
        return None

    try:
        obj2, parent = _getattribute(module, name)
    except AttributeError:
        # obj was not found inside the module it points to
        return None
    if obj2 is not obj:
        return None
    return module, name


def _explode_args_trace_inputs(
    sig: inspect.Signature, input: tuple[tuple[Any, ...], dict[str, Any]]
) -> dict[str, Any]:
    args, kwargs = input
    bound = sig.bind_partial(*args, **kwargs)
    bound.apply_defaults()
    arguments = dict(bound.arguments)
    arguments.pop("self", None)
    arguments.pop("cls", None)
    for param_name, param in sig.parameters.items():
        if param.kind == inspect.Parameter.VAR_KEYWORD:
            # Update with the **kwargs, and remove the original entry
            # This is to help flatten out keyword arguments
            if param_name in arguments:
                arguments.update(arguments.pop(param_name))
    return arguments


def get_runnable_for_entrypoint(func: Callable[..., Any]) -> Runnable:
    key = (func, False)
    if key in CACHE:
        return CACHE[key]
    else:
        if is_async_callable(func):
            run = RunnableCallable(
                None, func, name=func.__name__, trace=False, recurse=False
            )
        else:
            afunc = functools.update_wrapper(
                functools.partial(run_in_executor, None, func), func
            )
            run = RunnableCallable(
                func,
                afunc,
                name=func.__name__,
                trace=False,
                recurse=False,
            )
        if not _lookup_module_and_qualname(func):
            return run
        return CACHE.setdefault(key, run)


def get_runnable_for_task(func: Callable[..., Any]) -> Runnable:
    key = (func, True)
    if key in CACHE:
        return CACHE[key]
    else:
        if hasattr(func, "__name__"):
            name = func.__name__
        elif hasattr(func, "func"):
            name = func.func.__name__
        elif hasattr(func, "__class__"):
            name = func.__class__.__name__
        else:
            name = str(func)

        if is_async_callable(func):
            run = RunnableCallable(
                None,
                func,
                explode_args=True,
                name=name,
                trace=False,
                recurse=False,
            )
        else:
            run = RunnableCallable(
                func,
                functools.wraps(func)(functools.partial(run_in_executor, None, func)),
                explode_args=True,
                name=name,
                trace=False,
                recurse=False,
            )
        seq = RunnableSeq(
            run,
            ChannelWrite([ChannelWriteEntry(RETURN)]),
            name=name,
            trace_inputs=functools.partial(
                _explode_args_trace_inputs, inspect.signature(func)
            ),
        )
        if not _lookup_module_and_qualname(func):
            return seq
        return CACHE.setdefault(key, seq)


CACHE: dict[tuple[Callable[..., Any], bool], Runnable] = {}


P = ParamSpec("P")
P1 = TypeVar("P1")
T = TypeVar("T")


class SyncAsyncFuture(Generic[T], concurrent.futures.Future[T]):
    def __await__(self) -> Generator[T, None, T]:
        yield cast(T, ...)


def call(
    func: Callable[P, Awaitable[T]] | Callable[P, T],
    *args: Any,
    retry_policy: Sequence[RetryPolicy] | None = None,
    cache_policy: CachePolicy | None = None,
    **kwargs: Any,
) -> SyncAsyncFuture[T]:
    config = get_config()
    impl = config[CONF][CONFIG_KEY_CALL]
    fut = impl(
        func,
        (args, kwargs),
        retry_policy=retry_policy,
        cache_policy=cache_policy,
        callbacks=config["callbacks"],
    )
    return fut

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/_checkpoint.py
```py
from __future__ import annotations

from collections.abc import Mapping
from datetime import datetime, timezone

from langgraph.checkpoint.base import Checkpoint
from langgraph.checkpoint.base.id import uuid6

from langgraph._internal._typing import MISSING
from langgraph.channels.base import BaseChannel
from langgraph.managed.base import ManagedValueMapping, ManagedValueSpec

LATEST_VERSION = 4


def empty_checkpoint() -> Checkpoint:
    return Checkpoint(
        v=LATEST_VERSION,
        id=str(uuid6(clock_seq=-2)),
        ts=datetime.now(timezone.utc).isoformat(),
        channel_values={},
        channel_versions={},
        versions_seen={},
    )


def create_checkpoint(
    checkpoint: Checkpoint,
    channels: Mapping[str, BaseChannel] | None,
    step: int,
    *,
    id: str | None = None,
    updated_channels: set[str] | None = None,
) -> Checkpoint:
    """Create a checkpoint for the given channels."""
    ts = datetime.now(timezone.utc).isoformat()
    if channels is None:
        values = checkpoint["channel_values"]
    else:
        values = {}
        for k in channels:
            if k not in checkpoint["channel_versions"]:
                continue
            v = channels[k].checkpoint()
            if v is not MISSING:
                values[k] = v
    return Checkpoint(
        v=LATEST_VERSION,
        ts=ts,
        id=id or str(uuid6(clock_seq=step)),
        channel_values=values,
        channel_versions=checkpoint["channel_versions"],
        versions_seen=checkpoint["versions_seen"],
        updated_channels=None if updated_channels is None else sorted(updated_channels),
    )


def channels_from_checkpoint(
    specs: Mapping[str, BaseChannel | ManagedValueSpec],
    checkpoint: Checkpoint,
) -> tuple[Mapping[str, BaseChannel], ManagedValueMapping]:
    """Get channels from a checkpoint."""
    channel_specs: dict[str, BaseChannel] = {}
    managed_specs: dict[str, ManagedValueSpec] = {}
    for k, v in specs.items():
        if isinstance(v, BaseChannel):
            channel_specs[k] = v
        else:
            managed_specs[k] = v
    return (
        {
            k: v.from_checkpoint(checkpoint["channel_values"].get(k, MISSING))
            for k, v in channel_specs.items()
        },
        managed_specs,
    )


def copy_checkpoint(checkpoint: Checkpoint) -> Checkpoint:
    return Checkpoint(
        v=checkpoint["v"],
        ts=checkpoint["ts"],
        id=checkpoint["id"],
        channel_values=checkpoint["channel_values"].copy(),
        channel_versions=checkpoint["channel_versions"].copy(),
        versions_seen={k: v.copy() for k, v in checkpoint["versions_seen"].items()},
        updated_channels=checkpoint.get("updated_channels", None),
    )

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/_config.py
```py

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/_draw.py
```py
from __future__ import annotations

from collections import defaultdict
from collections.abc import Mapping, Sequence
from typing import Any, NamedTuple, cast

from langchain_core.runnables.config import RunnableConfig
from langchain_core.runnables.graph import Graph, Node
from langgraph.checkpoint.base import BaseCheckpointSaver

from langgraph._internal._constants import CONF, CONFIG_KEY_SEND, INPUT
from langgraph.channels.base import BaseChannel
from langgraph.channels.last_value import LastValueAfterFinish
from langgraph.constants import END, START
from langgraph.managed.base import ManagedValueSpec
from langgraph.pregel._algo import (
    PregelTaskWrites,
    apply_writes,
    increment,
    prepare_next_tasks,
)
from langgraph.pregel._checkpoint import channels_from_checkpoint, empty_checkpoint
from langgraph.pregel._io import map_input
from langgraph.pregel._read import PregelNode
from langgraph.pregel._write import ChannelWrite
from langgraph.types import All, Checkpointer


class Edge(NamedTuple):
    source: str
    target: str
    conditional: bool
    data: str | None


class TriggerEdge(NamedTuple):
    source: str
    conditional: bool
    data: str | None


def draw_graph(
    config: RunnableConfig,
    *,
    nodes: dict[str, PregelNode],
    specs: dict[str, BaseChannel | ManagedValueSpec],
    input_channels: str | Sequence[str],
    interrupt_after_nodes: All | Sequence[str],
    interrupt_before_nodes: All | Sequence[str],
    trigger_to_nodes: Mapping[str, Sequence[str]],
    checkpointer: Checkpointer,
    subgraphs: dict[str, Graph],
    limit: int = 250,
) -> Graph:
    """Get the graph for this Pregel instance.

    Args:
        config: The configuration to use for the graph.
        subgraphs: The subgraphs to include in the graph.
        checkpointer: The checkpointer to use for the graph.

    Returns:
        The graph for this Pregel instance.
    """
    # (src, dest, is_conditional, label)
    edges: set[Edge] = set()

    step = -1
    checkpoint = empty_checkpoint()
    get_next_version = (
        checkpointer.get_next_version
        if isinstance(checkpointer, BaseCheckpointSaver)
        else increment
    )
    channels, managed = channels_from_checkpoint(
        specs,
        checkpoint,
    )
    static_seen: set[Any] = set()
    sources: dict[str, set[TriggerEdge]] = {}
    step_sources: dict[str, set[TriggerEdge]] = {}
    static_declared_writes: dict[str, set[TriggerEdge]] = defaultdict(set)
    # remove node mappers
    nodes = {
        k: v.copy(update={"mapper": None}) if v.mapper is not None else v
        for k, v in nodes.items()
    }
    # apply input writes
    input_writes = list(map_input(input_channels, {}))
    updated_channels = apply_writes(
        checkpoint,
        channels,
        [
            PregelTaskWrites((), INPUT, input_writes, []),
        ],
        get_next_version,
        trigger_to_nodes,
    )
    # prepare first tasks
    tasks = prepare_next_tasks(
        checkpoint,
        [],
        nodes,
        channels,
        managed,
        config,
        step,
        -1,
        for_execution=True,
        store=None,
        checkpointer=None,
        manager=None,
        trigger_to_nodes=trigger_to_nodes,
        updated_channels=updated_channels,
    )
    start_tasks = tasks
    # run the pregel loop
    for step in range(step, limit):
        if not tasks:
            break
        conditionals: dict[tuple[str, str, Any], str | None] = {}
        # run task writers
        for task in tasks.values():
            for w in task.writers:
                # apply regular writes
                if isinstance(w, ChannelWrite):
                    empty_input = (
                        cast(BaseChannel, specs["__root__"]).ValueType()
                        if "__root__" in specs
                        else None
                    )
                    w.invoke(empty_input, task.config)
                # apply conditional writes declared for static analysis, only once
                if w not in static_seen:
                    static_seen.add(w)
                    # apply static writes
                    if writes := ChannelWrite.get_static_writes(w):
                        # END writes are not written, but become edges directly
                        for t in writes:
                            if t[0] == END:
                                edges.add(Edge(task.name, t[0], True, t[2]))
                        writes = [t for t in writes if t[0] != END]
                        conditionals.update(
                            {(task.name, t[0], t[1] or None): t[2] for t in writes}
                        )
                        # record static writes for edge creation
                        for t in writes:
                            static_declared_writes[task.name].add(
                                TriggerEdge(t[0], True, t[2])
                            )
                        task.config[CONF][CONFIG_KEY_SEND]([t[:2] for t in writes])
        # collect sources
        step_sources = {}
        for task in tasks.values():
            task_edges = {
                TriggerEdge(
                    w[0],
                    (task.name, w[0], w[1] or None) in conditionals,
                    conditionals.get((task.name, w[0], w[1] or None)),
                )
                for w in task.writes
            }
            task_edges |= static_declared_writes.get(task.name, set())
            step_sources[task.name] = task_edges
        sources.update(step_sources)
        # invert triggers
        trigger_to_sources: dict[str, set[TriggerEdge]] = defaultdict(set)
        for src, triggers in sources.items():
            for trigger, cond, label in triggers:
                trigger_to_sources[trigger].add(TriggerEdge(src, cond, label))
        # apply writes
        updated_channels = apply_writes(
            checkpoint, channels, tasks.values(), get_next_version, trigger_to_nodes
        )
        # prepare next tasks
        tasks = prepare_next_tasks(
            checkpoint,
            [],
            nodes,
            channels,
            managed,
            config,
            step,
            limit,
            for_execution=True,
            store=None,
            checkpointer=None,
            manager=None,
            trigger_to_nodes=trigger_to_nodes,
            updated_channels=updated_channels,
        )
        # collect deferred nodes
        deferred_nodes: set[str] = set()
        edges_to_deferred_nodes: set[Edge] = set()
        for channel, item in channels.items():
            if isinstance(item, LastValueAfterFinish):
                deferred_node = channel.split(":", 2)[-1]
                deferred_nodes.add(deferred_node)
        # collect edges
        for task in tasks.values():
            added = False
            for trigger in task.triggers:
                for src, cond, label in sorted(trigger_to_sources[trigger]):
                    # record edge to be reviewed later
                    if task.name in deferred_nodes:
                        edges_to_deferred_nodes.add(Edge(src, task.name, cond, label))
                    edges.add(Edge(src, task.name, cond, label))
                    # if the edge is from this step, skip adding the implicit edges
                    if (trigger, cond, label) in step_sources.get(src, set()):
                        added = True
                    else:
                        sources[src].discard(TriggerEdge(trigger, cond, label))
            # if no edges from this step, add implicit edges from all previous tasks
            if not added:
                for src in step_sources:
                    edges.add(Edge(src, task.name, True, None))

    # assemble the graph
    graph = Graph()
    # add nodes
    for name, node in nodes.items():
        metadata = dict(node.metadata or {})
        if name in deferred_nodes:
            metadata["defer"] = True
        if name in interrupt_before_nodes and name in interrupt_after_nodes:
            metadata["__interrupt"] = "before,after"
        elif name in interrupt_before_nodes:
            metadata["__interrupt"] = "before"
        elif name in interrupt_after_nodes:
            metadata["__interrupt"] = "after"
        graph.add_node(node.bound, name, metadata=metadata or None)
    # add start node
    if START not in nodes:
        graph.add_node(None, START)
        for task in start_tasks.values():
            add_edge(graph, START, task.name)
    # add discovered edges
    for src, dest, is_conditional, label in sorted(edges):
        add_edge(
            graph,
            src,
            dest,
            data=label if label != dest else None,
            conditional=is_conditional,
        )
    # add end edges
    termini = {d for _, d, _, _ in edges if d != END}.difference(
        s for s, _, _, _ in edges
    )
    end_edge_exists = any(d == END for _, d, _, _ in edges)
    if termini:
        for src in sorted(termini):
            add_edge(graph, src, END)
    elif len(step_sources) == 1 and not end_edge_exists:
        for src in sorted(step_sources):
            add_edge(graph, src, END, conditional=True)
    # replace subgraphs
    for name, subgraph in subgraphs.items():
        if (
            len(subgraph.nodes) > 1
            and name in graph.nodes
            and subgraph.first_node()
            and subgraph.last_node()
        ):
            subgraph.trim_first_node()
            subgraph.trim_last_node()
            # replace the node with the subgraph
            graph.nodes.pop(name)
            first, last = graph.extend(subgraph, prefix=name)
            for idx, edge in enumerate(graph.edges):
                if edge.source == name:
                    edge = edge.copy(source=cast(Node, last).id)
                if edge.target == name:
                    edge = edge.copy(target=cast(Node, first).id)
                graph.edges[idx] = edge

    return graph


def add_edge(
    graph: Graph,
    source: str,
    target: str,
    *,
    data: Any | None = None,
    conditional: bool = False,
) -> None:
    """Add an edge to the graph."""
    for edge in graph.edges:
        if edge.source == source and edge.target == target:
            return
    if target not in graph.nodes and target == END:
        graph.add_node(None, END)
    graph.add_edge(graph.nodes[source], graph.nodes[target], data, conditional)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/_executor.py
```py
from __future__ import annotations

import asyncio
import concurrent.futures
import time
from collections.abc import Awaitable, Callable, Coroutine
from contextlib import AbstractAsyncContextManager, AbstractContextManager, ExitStack
from contextvars import copy_context
from types import TracebackType
from typing import (
    Protocol,
    TypeVar,
    cast,
)

from langchain_core.runnables import RunnableConfig
from langchain_core.runnables.config import get_executor_for_config
from typing_extensions import ParamSpec

from langgraph._internal._future import CONTEXT_NOT_SUPPORTED, run_coroutine_threadsafe
from langgraph.errors import GraphBubbleUp

P = ParamSpec("P")
T = TypeVar("T")


class Submit(Protocol[P, T]):
    def __call__(  # type: ignore[valid-type]
        self,
        fn: Callable[P, T],
        *args: P.args,
        __name__: str | None = None,
        __cancel_on_exit__: bool = False,
        __reraise_on_exit__: bool = True,
        __next_tick__: bool = False,
        **kwargs: P.kwargs,
    ) -> concurrent.futures.Future[T]: ...


class BackgroundExecutor(AbstractContextManager):
    """A context manager that runs sync tasks in the background.
    Uses a thread pool executor to delegate tasks to separate threads.
    On exit,
    - cancels any (not yet started) tasks with `__cancel_on_exit__=True`
    - waits for all tasks to finish
    - re-raises the first exception from tasks with `__reraise_on_exit__=True`"""

    def __init__(self, config: RunnableConfig) -> None:
        self.stack = ExitStack()
        self.executor = self.stack.enter_context(get_executor_for_config(config))
        # mapping of Future to (__cancel_on_exit__, __reraise_on_exit__) flags
        self.tasks: dict[concurrent.futures.Future, tuple[bool, bool]] = {}

    def submit(  # type: ignore[valid-type]
        self,
        fn: Callable[P, T],
        *args: P.args,
        __name__: str | None = None,  # currently not used in sync version
        __cancel_on_exit__: bool = False,  # for sync, can cancel only if not started
        __reraise_on_exit__: bool = True,
        __next_tick__: bool = False,
        **kwargs: P.kwargs,
    ) -> concurrent.futures.Future[T]:
        ctx = copy_context()
        if __next_tick__:
            task = cast(
                concurrent.futures.Future[T],
                self.executor.submit(next_tick, ctx.run, fn, *args, **kwargs),  # type: ignore[arg-type]
            )
        else:
            task = self.executor.submit(ctx.run, fn, *args, **kwargs)
        self.tasks[task] = (__cancel_on_exit__, __reraise_on_exit__)
        # add a callback to remove the task from the tasks dict when it's done
        task.add_done_callback(self.done)
        return task

    def done(self, task: concurrent.futures.Future) -> None:
        """Remove the task from the tasks dict when it's done."""
        try:
            task.result()
        except GraphBubbleUp:
            # This exception is an interruption signal, not an error
            # so we don't want to re-raise it on exit
            self.tasks.pop(task)
        except BaseException:
            pass
        else:
            self.tasks.pop(task)

    def __enter__(self) -> Submit:
        return self.submit

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc_value: BaseException | None,
        traceback: TracebackType | None,
    ) -> bool | None:
        # copy the tasks as done() callback may modify the dict
        tasks = self.tasks.copy()
        # cancel all tasks that should be cancelled
        for task, (cancel, _) in tasks.items():
            if cancel:
                task.cancel()
        # wait for all tasks to finish
        if pending := {t for t in tasks if not t.done()}:
            concurrent.futures.wait(pending)
        # shutdown the executor
        self.stack.__exit__(exc_type, exc_value, traceback)
        # if there's already an exception being raised, don't raise another one
        if exc_type is None:
            # re-raise the first exception that occurred in a task
            for task, (_, reraise) in tasks.items():
                if not reraise:
                    continue
                try:
                    task.result()
                except concurrent.futures.CancelledError:
                    pass


class AsyncBackgroundExecutor(AbstractAsyncContextManager):
    """A context manager that runs async tasks in the background.
    Uses the current event loop to delegate tasks to asyncio tasks.
    On exit,
    - cancels any tasks with `__cancel_on_exit__=True`
    - waits for all tasks to finish
    - re-raises the first exception from tasks with `__reraise_on_exit__=True`
      ignoring CancelledError"""

    def __init__(self, config: RunnableConfig) -> None:
        self.tasks: dict[asyncio.Future, tuple[bool, bool]] = {}
        self.sentinel = object()
        self.loop = asyncio.get_running_loop()
        if max_concurrency := config.get("max_concurrency"):
            self.semaphore: asyncio.Semaphore | None = asyncio.Semaphore(
                max_concurrency
            )
        else:
            self.semaphore = None

    def submit(  # type: ignore[valid-type]
        self,
        fn: Callable[P, Awaitable[T]],
        *args: P.args,
        __name__: str | None = None,
        __cancel_on_exit__: bool = False,
        __reraise_on_exit__: bool = True,
        __next_tick__: bool = False,  # noop in async (always True)
        **kwargs: P.kwargs,
    ) -> asyncio.Future[T]:
        coro = cast(Coroutine[None, None, T], fn(*args, **kwargs))
        if self.semaphore:
            coro = gated(self.semaphore, coro)
        if CONTEXT_NOT_SUPPORTED:
            task = run_coroutine_threadsafe(
                coro, self.loop, name=__name__, lazy=__next_tick__
            )
        else:
            task = run_coroutine_threadsafe(
                coro,
                self.loop,
                name=__name__,
                context=copy_context(),
                lazy=__next_tick__,
            )
        self.tasks[task] = (__cancel_on_exit__, __reraise_on_exit__)
        task.add_done_callback(self.done)
        return task

    def done(self, task: asyncio.Future) -> None:
        try:
            if exc := task.exception():
                # This exception is an interruption signal, not an error
                # so we don't want to re-raise it on exit
                if isinstance(exc, GraphBubbleUp):
                    self.tasks.pop(task)
            else:
                self.tasks.pop(task)
        except asyncio.CancelledError:
            self.tasks.pop(task)

    async def __aenter__(self) -> Submit:
        return self.submit

    async def __aexit__(
        self,
        exc_type: type[BaseException] | None,
        exc_value: BaseException | None,
        traceback: TracebackType | None,
    ) -> None:
        # copy the tasks as done() callback may modify the dict
        tasks = self.tasks.copy()
        # cancel all tasks that should be cancelled
        for task, (cancel, _) in tasks.items():
            if cancel:
                task.cancel(self.sentinel)
        # wait for all tasks to finish
        if tasks:
            await asyncio.wait(tasks)
        # if there's already an exception being raised, don't raise another one
        if exc_type is None:
            # re-raise the first exception that occurred in a task
            for task, (_, reraise) in tasks.items():
                if not reraise:
                    continue
                try:
                    if exc := task.exception():
                        raise exc
                except asyncio.CancelledError:
                    pass


async def gated(semaphore: asyncio.Semaphore, coro: Coroutine[None, None, T]) -> T:
    """A coroutine that waits for a semaphore before running another coroutine."""
    async with semaphore:
        return await coro


def next_tick(fn: Callable[P, T], *args: P.args, **kwargs: P.kwargs) -> T:
    """A function that yields control to other threads before running another function."""
    time.sleep(0)
    return fn(*args, **kwargs)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/_io.py
```py
from __future__ import annotations

from collections import Counter
from collections.abc import Iterator, Mapping, Sequence
from typing import Any, Literal

from langgraph._internal._constants import (
    ERROR,
    INTERRUPT,
    NULL_TASK_ID,
    RESUME,
    RETURN,
    TASKS,
)
from langgraph._internal._typing import EMPTY_SEQ, MISSING
from langgraph.channels.base import BaseChannel, EmptyChannelError
from langgraph.constants import START, TAG_HIDDEN
from langgraph.errors import InvalidUpdateError
from langgraph.pregel._log import logger
from langgraph.types import Command, PregelExecutableTask, Send


def read_channel(
    channels: Mapping[str, BaseChannel],
    chan: str,
    *,
    catch: bool = True,
) -> Any:
    try:
        return channels[chan].get()
    except EmptyChannelError:
        if catch:
            return None
        else:
            raise


def read_channels(
    channels: Mapping[str, BaseChannel],
    select: Sequence[str] | str,
    *,
    skip_empty: bool = True,
) -> dict[str, Any] | Any:
    if isinstance(select, str):
        return read_channel(channels, select)
    else:
        values: dict[str, Any] = {}
        for k in select:
            try:
                values[k] = read_channel(channels, k, catch=not skip_empty)
            except EmptyChannelError:
                pass
        return values


def map_command(cmd: Command) -> Iterator[tuple[str, str, Any]]:
    """Map input chunk to a sequence of pending writes in the form (channel, value)."""
    if cmd.graph == Command.PARENT:
        raise InvalidUpdateError("There is no parent graph")
    if cmd.goto:
        if isinstance(cmd.goto, (tuple, list)):
            sends = cmd.goto
        else:
            sends = [cmd.goto]
        for send in sends:
            if isinstance(send, Send):
                yield (NULL_TASK_ID, TASKS, send)
            elif isinstance(send, str):
                yield (NULL_TASK_ID, f"branch:to:{send}", START)
            else:
                raise TypeError(
                    f"In Command.goto, expected Send/str, got {type(send).__name__}"
                )
    if cmd.resume is not None:
        yield (NULL_TASK_ID, RESUME, cmd.resume)
    if cmd.update:
        for k, v in cmd._update_as_tuples():
            yield (NULL_TASK_ID, k, v)


def map_input(
    input_channels: str | Sequence[str],
    chunk: dict[str, Any] | Any | None,
) -> Iterator[tuple[str, Any]]:
    """Map input chunk to a sequence of pending writes in the form (channel, value)."""
    if chunk is None:
        return
    elif isinstance(input_channels, str):
        yield (input_channels, chunk)
    else:
        if not isinstance(chunk, dict):
            raise TypeError(f"Expected chunk to be a dict, got {type(chunk).__name__}")
        for k in chunk:
            if k in input_channels:
                yield (k, chunk[k])
            else:
                logger.warning(f"Input channel {k} not found in {input_channels}")


def map_output_values(
    output_channels: str | Sequence[str],
    pending_writes: Literal[True] | Sequence[tuple[str, Any]],
    channels: Mapping[str, BaseChannel],
) -> Iterator[dict[str, Any] | Any]:
    """Map pending writes (a sequence of tuples (channel, value)) to output chunk."""
    if isinstance(output_channels, str):
        if pending_writes is True or any(
            chan == output_channels for chan, _ in pending_writes
        ):
            yield read_channel(channels, output_channels)
    else:
        if pending_writes is True or {
            c for c, _ in pending_writes if c in output_channels
        }:
            yield read_channels(channels, output_channels)


def map_output_updates(
    output_channels: str | Sequence[str],
    tasks: list[tuple[PregelExecutableTask, Sequence[tuple[str, Any]]]],
    cached: bool = False,
) -> Iterator[dict[str, Any | dict[str, Any]]]:
    """Map pending writes (a sequence of tuples (channel, value)) to output chunk."""
    output_tasks = [
        (t, ww)
        for t, ww in tasks
        if (not t.config or TAG_HIDDEN not in t.config.get("tags", EMPTY_SEQ))
        and ww[0][0] != ERROR
        and ww[0][0] != INTERRUPT
    ]
    if not output_tasks:
        return
    updated: list[tuple[str, Any]] = []
    for task, writes in output_tasks:
        rtn = next((value for chan, value in writes if chan == RETURN), MISSING)
        if rtn is not MISSING:
            updated.append((task.name, rtn))
        elif isinstance(output_channels, str):
            updated.extend(
                (task.name, value) for chan, value in writes if chan == output_channels
            )
        elif any(chan in output_channels for chan, _ in writes):
            counts = Counter(chan for chan, _ in writes)
            if any(counts[chan] > 1 for chan in output_channels):
                updated.extend(
                    (
                        task.name,
                        {chan: value},
                    )
                    for chan, value in writes
                    if chan in output_channels
                )
            else:
                updated.append(
                    (
                        task.name,
                        {
                            chan: value
                            for chan, value in writes
                            if chan in output_channels
                        },
                    )
                )
    grouped: dict[str, Any] = {t.name: [] for t, _ in output_tasks}
    for node, value in updated:
        grouped[node].append(value)
    for node, value in grouped.items():
        if len(value) == 0:
            grouped[node] = None
        if len(value) == 1:
            grouped[node] = value[0]
    if cached:
        grouped["__metadata__"] = {"cached": cached}
    yield grouped

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/_log.py
```py
import logging

logger = logging.getLogger("langgraph")

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/_loop.py
```py
from __future__ import annotations

import asyncio
import binascii
import concurrent.futures
from collections import defaultdict, deque
from collections.abc import Callable, Iterator, Mapping, Sequence
from contextlib import (
    AbstractAsyncContextManager,
    AbstractContextManager,
    AsyncExitStack,
    ExitStack,
)
from datetime import datetime, timezone
from inspect import signature
from types import TracebackType
from typing import (
    Any,
    Literal,
    TypeVar,
    cast,
)

from langchain_core.callbacks import AsyncParentRunManager, ParentRunManager
from langchain_core.runnables import RunnableConfig
from langgraph.cache.base import BaseCache
from langgraph.checkpoint.base import (
    WRITES_IDX_MAP,
    BaseCheckpointSaver,
    ChannelVersions,
    Checkpoint,
    CheckpointMetadata,
    CheckpointTuple,
    PendingWrite,
)
from langgraph.store.base import BaseStore
from typing_extensions import ParamSpec, Self

from langgraph._internal._config import patch_configurable
from langgraph._internal._constants import (
    CONF,
    CONFIG_KEY_CHECKPOINT_ID,
    CONFIG_KEY_CHECKPOINT_MAP,
    CONFIG_KEY_CHECKPOINT_NS,
    CONFIG_KEY_RESUME_MAP,
    CONFIG_KEY_RESUMING,
    CONFIG_KEY_SCRATCHPAD,
    CONFIG_KEY_STREAM,
    CONFIG_KEY_TASK_ID,
    CONFIG_KEY_THREAD_ID,
    ERROR,
    INPUT,
    INTERRUPT,
    NS_END,
    NS_SEP,
    NULL_TASK_ID,
    PUSH,
    RESUME,
    TASKS,
)
from langgraph._internal._scratchpad import PregelScratchpad
from langgraph._internal._typing import EMPTY_SEQ, MISSING
from langgraph.channels.base import BaseChannel
from langgraph.channels.untracked_value import UntrackedValue
from langgraph.constants import TAG_HIDDEN
from langgraph.errors import (
    EmptyInputError,
    GraphInterrupt,
)
from langgraph.managed.base import (
    ManagedValueMapping,
    ManagedValueSpec,
)
from langgraph.pregel._algo import (
    Call,
    GetNextVersion,
    PregelTaskWrites,
    apply_writes,
    checkpoint_null_version,
    increment,
    prepare_next_tasks,
    prepare_single_task,
    sanitize_untracked_values_in_send,
    should_interrupt,
    task_path_str,
)
from langgraph.pregel._checkpoint import (
    channels_from_checkpoint,
    copy_checkpoint,
    create_checkpoint,
    empty_checkpoint,
)
from langgraph.pregel._executor import (
    AsyncBackgroundExecutor,
    BackgroundExecutor,
    Submit,
)
from langgraph.pregel._io import (
    map_command,
    map_input,
    map_output_updates,
    map_output_values,
    read_channels,
)
from langgraph.pregel._read import PregelNode
from langgraph.pregel._utils import get_new_channel_versions, is_xxh3_128_hexdigest
from langgraph.pregel.debug import (
    map_debug_checkpoint,
    map_debug_task_results,
    map_debug_tasks,
)
from langgraph.pregel.protocol import StreamChunk, StreamProtocol
from langgraph.types import (
    All,
    CachePolicy,
    Command,
    Durability,
    PregelExecutableTask,
    RetryPolicy,
    Send,
    StreamMode,
)

V = TypeVar("V")
P = ParamSpec("P")


WritesT = Sequence[tuple[str, Any]]


def DuplexStream(*streams: StreamProtocol) -> StreamProtocol:
    def __call__(value: StreamChunk) -> None:
        for stream in streams:
            if value[1] in stream.modes:
                stream(value)

    return StreamProtocol(__call__, {mode for s in streams for mode in s.modes})


class PregelLoop:
    config: RunnableConfig
    store: BaseStore | None
    stream: StreamProtocol | None
    step: int
    stop: int

    input: Any | None
    cache: BaseCache[WritesT] | None
    checkpointer: BaseCheckpointSaver | None
    nodes: Mapping[str, PregelNode]
    specs: Mapping[str, BaseChannel | ManagedValueSpec]
    input_keys: str | Sequence[str]
    output_keys: str | Sequence[str]
    stream_keys: str | Sequence[str]
    skip_done_tasks: bool
    is_nested: bool
    manager: None | AsyncParentRunManager | ParentRunManager
    interrupt_after: All | Sequence[str]
    interrupt_before: All | Sequence[str]
    durability: Durability
    retry_policy: Sequence[RetryPolicy]
    cache_policy: CachePolicy | None

    checkpointer_get_next_version: GetNextVersion
    checkpointer_put_writes: Callable[[RunnableConfig, WritesT, str], Any] | None
    checkpointer_put_writes_accepts_task_path: bool
    _checkpointer_put_after_previous: (
        Callable[
            [
                concurrent.futures.Future | None,
                RunnableConfig,
                Checkpoint,
                str,
                ChannelVersions,
            ],
            Any,
        ]
        | None
    )
    _migrate_checkpoint: Callable[[Checkpoint], None] | None
    submit: Submit
    channels: Mapping[str, BaseChannel]
    managed: ManagedValueMapping
    checkpoint: Checkpoint
    checkpoint_id_saved: str
    checkpoint_ns: tuple[str, ...]
    checkpoint_config: RunnableConfig
    checkpoint_metadata: CheckpointMetadata
    checkpoint_pending_writes: list[PendingWrite]
    checkpoint_previous_versions: dict[str, str | float | int]
    prev_checkpoint_config: RunnableConfig | None

    status: Literal[
        "input",
        "pending",
        "done",
        "interrupt_before",
        "interrupt_after",
        "out_of_steps",
    ]
    tasks: dict[str, PregelExecutableTask]
    output: None | dict[str, Any] | Any = None
    updated_channels: set[str] | None = None

    # public

    def __init__(
        self,
        input: Any | None,
        *,
        stream: StreamProtocol | None,
        config: RunnableConfig,
        store: BaseStore | None,
        cache: BaseCache | None,
        checkpointer: BaseCheckpointSaver | None,
        nodes: Mapping[str, PregelNode],
        specs: Mapping[str, BaseChannel | ManagedValueSpec],
        input_keys: str | Sequence[str],
        output_keys: str | Sequence[str],
        stream_keys: str | Sequence[str],
        trigger_to_nodes: Mapping[str, Sequence[str]],
        durability: Durability,
        interrupt_after: All | Sequence[str] = EMPTY_SEQ,
        interrupt_before: All | Sequence[str] = EMPTY_SEQ,
        manager: None | AsyncParentRunManager | ParentRunManager = None,
        migrate_checkpoint: Callable[[Checkpoint], None] | None = None,
        retry_policy: Sequence[RetryPolicy] = (),
        cache_policy: CachePolicy | None = None,
    ) -> None:
        self.stream = stream
        self.config = config
        self.store = store
        self.step = 0
        self.stop = 0
        self.input = input
        self.checkpointer = checkpointer
        self.cache = cache
        self.nodes = nodes
        self.specs = specs
        self.input_keys = input_keys
        self.output_keys = output_keys
        self.stream_keys = stream_keys
        self.interrupt_after = interrupt_after
        self.interrupt_before = interrupt_before
        self.manager = manager
        self.is_nested = CONFIG_KEY_TASK_ID in self.config.get(CONF, {})
        self.skip_done_tasks = CONFIG_KEY_CHECKPOINT_ID not in config[CONF]
        self._migrate_checkpoint = migrate_checkpoint
        self.trigger_to_nodes = trigger_to_nodes
        self.retry_policy = retry_policy
        self.cache_policy = cache_policy
        self.durability = durability
        if self.stream is not None and CONFIG_KEY_STREAM in config[CONF]:
            self.stream = DuplexStream(self.stream, config[CONF][CONFIG_KEY_STREAM])
        scratchpad: PregelScratchpad | None = config[CONF].get(CONFIG_KEY_SCRATCHPAD)
        if isinstance(scratchpad, PregelScratchpad):
            # if count is > 0, append to checkpoint_ns
            # if count is 0, leave as is
            if cnt := scratchpad.subgraph_counter():
                self.config = patch_configurable(
                    self.config,
                    {
                        CONFIG_KEY_CHECKPOINT_NS: NS_SEP.join(
                            (
                                config[CONF][CONFIG_KEY_CHECKPOINT_NS],
                                str(cnt),
                            )
                        )
                    },
                )
        if not self.is_nested and config[CONF].get(CONFIG_KEY_CHECKPOINT_NS):
            self.config = patch_configurable(
                self.config,
                {CONFIG_KEY_CHECKPOINT_NS: "", CONFIG_KEY_CHECKPOINT_ID: None},
            )
        if (
            CONFIG_KEY_CHECKPOINT_MAP in self.config[CONF]
            and self.config[CONF].get(CONFIG_KEY_CHECKPOINT_NS)
            in self.config[CONF][CONFIG_KEY_CHECKPOINT_MAP]
        ):
            self.checkpoint_config = patch_configurable(
                self.config,
                {
                    CONFIG_KEY_CHECKPOINT_ID: self.config[CONF][
                        CONFIG_KEY_CHECKPOINT_MAP
                    ][self.config[CONF][CONFIG_KEY_CHECKPOINT_NS]]
                },
            )
        else:
            self.checkpoint_config = self.config
        if thread_id := self.checkpoint_config[CONF].get(CONFIG_KEY_THREAD_ID):
            if not isinstance(thread_id, str):
                self.checkpoint_config = patch_configurable(
                    self.checkpoint_config,
                    {CONFIG_KEY_THREAD_ID: str(thread_id)},
                )
        self.checkpoint_ns = (
            tuple(cast(str, self.config[CONF][CONFIG_KEY_CHECKPOINT_NS]).split(NS_SEP))
            if self.config[CONF].get(CONFIG_KEY_CHECKPOINT_NS)
            else ()
        )
        self.prev_checkpoint_config = None

    def put_writes(self, task_id: str, writes: WritesT) -> None:
        """Put writes for a task, to be read by the next tick."""
        if not writes:
            return
        # deduplicate writes to special channels, last write wins
        if all(w[0] in WRITES_IDX_MAP for w in writes):
            writes = list({w[0]: w for w in writes}.values())
        if task_id == NULL_TASK_ID:
            # writes for the null task are accumulated
            self.checkpoint_pending_writes = [
                w
                for w in self.checkpoint_pending_writes
                if w[0] != task_id or w[1] not in WRITES_IDX_MAP
            ]
            writes_to_save: WritesT = [
                w[1:] for w in self.checkpoint_pending_writes if w[0] == task_id
            ] + list(writes)
        else:
            # remove existing writes for this task
            self.checkpoint_pending_writes = [
                w for w in self.checkpoint_pending_writes if w[0] != task_id
            ]
            writes_to_save = writes

        # check if any writes are to an UntrackedValue channel
        if any(
            isinstance(channel, UntrackedValue) for channel in self.channels.values()
        ):
            # we do not persist untracked values in checkpoints
            writes_to_save = [
                # sanitize UntrackedValues that are nested within Send packets
                (
                    (c, sanitize_untracked_values_in_send(v, self.channels))
                    if c == TASKS and isinstance(v, Send)
                    else (c, v)
                )
                for c, v in writes_to_save
                # dont persist UntrackedValue channel writes
                if not isinstance(self.specs.get(c), UntrackedValue)
            ]

        # save writes
        self.checkpoint_pending_writes.extend((task_id, c, v) for c, v in writes)
        if self.durability != "exit" and self.checkpointer_put_writes is not None:
            config = patch_configurable(
                self.checkpoint_config,
                {
                    CONFIG_KEY_CHECKPOINT_NS: self.config[CONF].get(
                        CONFIG_KEY_CHECKPOINT_NS, ""
                    ),
                    CONFIG_KEY_CHECKPOINT_ID: self.checkpoint["id"],
                },
            )
            if self.checkpointer_put_writes_accepts_task_path:
                if hasattr(self, "tasks"):
                    task = self.tasks.get(task_id)
                else:
                    task = None
                self.submit(
                    self.checkpointer_put_writes,
                    config,
                    writes_to_save,
                    task_id,
                    task_path_str(task.path) if task else "",
                )
            else:
                self.submit(
                    self.checkpointer_put_writes,
                    config,
                    writes_to_save,
                    task_id,
                )
        # output writes
        if hasattr(self, "tasks"):
            self.output_writes(task_id, writes)

    def _put_pending_writes(self) -> None:
        if self.checkpointer_put_writes is None:
            return
        if not self.checkpoint_pending_writes:
            return
        # patch config
        config = patch_configurable(
            self.checkpoint_config,
            {
                CONFIG_KEY_CHECKPOINT_NS: self.config[CONF].get(
                    CONFIG_KEY_CHECKPOINT_NS, ""
                ),
                CONFIG_KEY_CHECKPOINT_ID: self.checkpoint["id"],
            },
        )
        # group by task id
        by_task = defaultdict(list)
        for task_id, channel, value in self.checkpoint_pending_writes:
            by_task[task_id].append((channel, value))
        # submit writes to checkpointer
        for task_id, writes in by_task.items():
            if self.checkpointer_put_writes_accepts_task_path and hasattr(
                self, "tasks"
            ):
                task = self.tasks.get(task_id)
                self.submit(
                    self.checkpointer_put_writes,
                    config,
                    writes,
                    task_id,
                    task_path_str(task.path) if task else "",
                )
            else:
                self.submit(
                    self.checkpointer_put_writes,
                    config,
                    writes,
                    task_id,
                )

    def accept_push(
        self, task: PregelExecutableTask, write_idx: int, call: Call | None = None
    ) -> PregelExecutableTask | None:
        """Accept a PUSH from a task, potentially returning a new task to start."""
        checkpoint_id_bytes = binascii.unhexlify(self.checkpoint["id"].replace("-", ""))
        null_version = checkpoint_null_version(self.checkpoint)
        if pushed := cast(
            PregelExecutableTask | None,
            prepare_single_task(
                (PUSH, task.path, write_idx, task.id, call),
                None,
                checkpoint=self.checkpoint,
                checkpoint_id_bytes=checkpoint_id_bytes,
                checkpoint_null_version=null_version,
                pending_writes=self.checkpoint_pending_writes,
                processes=self.nodes,
                channels=self.channels,
                managed=self.managed,
                config=task.config,
                step=self.step,
                stop=self.stop,
                for_execution=True,
                store=self.store,
                checkpointer=self.checkpointer,
                manager=self.manager,
                retry_policy=self.retry_policy,
                cache_policy=self.cache_policy,
            ),
        ):
            # produce debug output
            self._emit("tasks", map_debug_tasks, [pushed])
            # save the new task
            self.tasks[pushed.id] = pushed
            # match any pending writes to the new task
            if self.skip_done_tasks:
                self._match_writes({pushed.id: pushed})
            # return the new task, to be started if not run before
            return pushed

    def tick(self) -> bool:
        """Execute a single iteration of the Pregel loop.

        Returns:
            True if more iterations are needed.
        """

        # check if iteration limit is reached
        if self.step > self.stop:
            self.status = "out_of_steps"
            return False

        # prepare next tasks
        self.tasks = prepare_next_tasks(
            self.checkpoint,
            self.checkpoint_pending_writes,
            self.nodes,
            self.channels,
            self.managed,
            self.config,
            self.step,
            self.stop,
            for_execution=True,
            manager=self.manager,
            store=self.store,
            checkpointer=self.checkpointer,
            trigger_to_nodes=self.trigger_to_nodes,
            updated_channels=self.updated_channels,
            retry_policy=self.retry_policy,
            cache_policy=self.cache_policy,
        )

        # produce debug output
        if self._checkpointer_put_after_previous is not None:
            self._emit(
                "checkpoints",
                map_debug_checkpoint,
                {
                    **self.checkpoint_config,
                    CONF: {
                        **self.checkpoint_config[CONF],
                        CONFIG_KEY_CHECKPOINT_ID: self.checkpoint["id"],
                    },
                },
                self.channels,
                self.stream_keys,
                self.checkpoint_metadata,
                self.tasks.values(),
                self.checkpoint_pending_writes,
                self.prev_checkpoint_config,
                self.output_keys,
            )

        # if no more tasks, we're done
        if not self.tasks:
            self.status = "done"
            return False

        # if there are pending writes from a previous loop, apply them
        if self.skip_done_tasks and self.checkpoint_pending_writes:
            self._match_writes(self.tasks)

        # before execution, check if we should interrupt
        if self.interrupt_before and should_interrupt(
            self.checkpoint, self.interrupt_before, self.tasks.values()
        ):
            self.status = "interrupt_before"
            raise GraphInterrupt()

        # produce debug output
        self._emit("tasks", map_debug_tasks, self.tasks.values())

        # print output for any tasks we applied previous writes to
        for task in self.tasks.values():
            if task.writes:
                self.output_writes(task.id, task.writes, cached=True)

        return True

    def after_tick(self) -> None:
        # finish superstep
        writes = [w for t in self.tasks.values() for w in t.writes]
        # all tasks have finished
        self.updated_channels = apply_writes(
            self.checkpoint,
            self.channels,
            self.tasks.values(),
            self.checkpointer_get_next_version,
            self.trigger_to_nodes,
        )
        # produce values output
        if not self.updated_channels.isdisjoint(
            (self.output_keys,)
            if isinstance(self.output_keys, str)
            else self.output_keys
        ):
            self._emit(
                "values", map_output_values, self.output_keys, writes, self.channels
            )
        # clear pending writes
        self.checkpoint_pending_writes.clear()
        # "not skip_done_tasks" only applies to first tick after resuming
        self.skip_done_tasks = True
        # save checkpoint
        self._put_checkpoint({"source": "loop"})
        # after execution, check if we should interrupt
        if self.interrupt_after and should_interrupt(
            self.checkpoint, self.interrupt_after, self.tasks.values()
        ):
            self.status = "interrupt_after"
            raise GraphInterrupt()
        # unset resuming flag
        self.config[CONF].pop(CONFIG_KEY_RESUMING, None)

    def match_cached_writes(self) -> Sequence[PregelExecutableTask]:
        raise NotImplementedError

    async def amatch_cached_writes(self) -> Sequence[PregelExecutableTask]:
        raise NotImplementedError

    # private

    def _match_writes(self, tasks: Mapping[str, PregelExecutableTask]) -> None:
        for tid, k, v in self.checkpoint_pending_writes:
            if k in (ERROR, INTERRUPT, RESUME):
                continue
            if task := tasks.get(tid):
                task.writes.append((k, v))

    def _pending_interrupts(self) -> set[str]:
        """Return the set of interrupt ids that are pending without corresponding resume values."""
        # mapping of task ids to interrupt ids
        pending_interrupts: dict[str, str] = {}

        # set of resume task ids
        pending_resumes: set[str] = set()

        for task_id, write_type, value in self.checkpoint_pending_writes:
            if write_type == INTERRUPT:
                # interrupts is always a list, but there should only be one element
                pending_interrupts[task_id] = value[0].id
            elif write_type == RESUME:
                pending_resumes.add(task_id)

        resumed_interrupt_ids = {
            pending_interrupts[task_id]
            for task_id in pending_resumes
            if task_id in pending_interrupts
        }

        # Keep only interrupts whose interrupt_id is not resumed
        hanging_interrupts: set[str] = {
            interrupt_id
            for interrupt_id in pending_interrupts.values()
            if interrupt_id not in resumed_interrupt_ids
        }

        return hanging_interrupts

    def _first(
        self, *, input_keys: str | Sequence[str], updated_channels: set[str] | None
    ) -> set[str] | None:
        # resuming from previous checkpoint requires
        # - finding a previous checkpoint
        # - receiving None input (outer graph) or RESUMING flag (subgraph)
        configurable = self.config.get(CONF, {})
        is_resuming = bool(self.checkpoint["channel_versions"]) and bool(
            configurable.get(
                CONFIG_KEY_RESUMING,
                self.input is None
                or isinstance(self.input, Command)
                or (
                    not self.is_nested
                    and self.config.get("metadata", {}).get("run_id")
                    == self.checkpoint_metadata.get("run_id", MISSING)
                ),
            )
        )

        # map command to writes
        if isinstance(self.input, Command):
            if (resume := self.input.resume) is not None:
                if not self.checkpointer:
                    raise RuntimeError(
                        "Cannot use Command(resume=...) without checkpointer"
                    )

                if resume_is_map := (
                    isinstance(resume, dict)
                    and all(is_xxh3_128_hexdigest(k) for k in resume)
                ):
                    self.config[CONF][CONFIG_KEY_RESUME_MAP] = resume
                else:
                    if len(self._pending_interrupts()) > 1:
                        raise RuntimeError(
                            "When there are multiple pending interrupts, you must specify the interrupt id when resuming. "
                            "Docs: https://docs.langchain.com/oss/python/langgraph/add-human-in-the-loop#resume-multiple-interrupts-with-one-invocation."
                        )

            writes: defaultdict[str, list[tuple[str, Any]]] = defaultdict(list)
            # group writes by task ID
            for tid, c, v in map_command(cmd=self.input):
                if not (c == RESUME and resume_is_map):
                    writes[tid].append((c, v))
            if not writes and not resume_is_map:
                raise EmptyInputError("Received empty Command input")
            # save writes
            for tid, ws in writes.items():
                self.put_writes(tid, ws)
        # apply NULL writes
        if null_writes := [
            w[1:] for w in self.checkpoint_pending_writes if w[0] == NULL_TASK_ID
        ]:
            null_updated_channels = apply_writes(
                self.checkpoint,
                self.channels,
                [PregelTaskWrites((), INPUT, null_writes, [])],
                self.checkpointer_get_next_version,
                self.trigger_to_nodes,
            )
            if updated_channels is not None:
                updated_channels.update(null_updated_channels)
        # proceed past previous checkpoint
        if is_resuming:
            self.checkpoint["versions_seen"].setdefault(INTERRUPT, {})
            for k in self.channels:
                if k in self.checkpoint["channel_versions"]:
                    version = self.checkpoint["channel_versions"][k]
                    self.checkpoint["versions_seen"][INTERRUPT][k] = version
            # produce values output
            self._emit(
                "values", map_output_values, self.output_keys, True, self.channels
            )
        # map inputs to channel updates
        elif input_writes := deque(map_input(input_keys, self.input)):
            # discard any unfinished tasks from previous checkpoint
            discard_tasks = prepare_next_tasks(
                self.checkpoint,
                self.checkpoint_pending_writes,
                self.nodes,
                self.channels,
                self.managed,
                self.config,
                self.step,
                self.stop,
                for_execution=True,
                store=None,
                checkpointer=None,
                manager=None,
                updated_channels=updated_channels,
            )
            # apply input writes
            updated_channels = apply_writes(
                self.checkpoint,
                self.channels,
                [
                    *discard_tasks.values(),
                    PregelTaskWrites((), INPUT, input_writes, []),
                ],
                self.checkpointer_get_next_version,
                self.trigger_to_nodes,
            )
            # save input checkpoint
            self.updated_channels = updated_channels
            self._put_checkpoint({"source": "input"})
        elif CONFIG_KEY_RESUMING not in configurable:
            raise EmptyInputError(f"Received no input for {input_keys}")
        # update config
        if not self.is_nested:
            self.config = patch_configurable(
                self.config, {CONFIG_KEY_RESUMING: is_resuming}
            )
        # set flag
        self.status = "pending"
        return updated_channels

    def _put_checkpoint(self, metadata: CheckpointMetadata) -> None:
        # assign step and parents
        exiting = metadata is self.checkpoint_metadata
        if exiting and self.checkpoint["id"] == self.checkpoint_id_saved:
            # checkpoint already saved
            return
        if not exiting:
            metadata["step"] = self.step
            metadata["parents"] = self.config[CONF].get(CONFIG_KEY_CHECKPOINT_MAP, {})
            self.checkpoint_metadata = metadata
        # do checkpoint?
        do_checkpoint = self._checkpointer_put_after_previous is not None and (
            exiting or self.durability != "exit"
        )
        # create new checkpoint
        self.checkpoint = create_checkpoint(
            self.checkpoint,
            self.channels if do_checkpoint else None,
            self.step,
            id=self.checkpoint["id"] if exiting else None,
            updated_channels=self.updated_channels,
        )
        # sanitize TASK channel in the checkpoint before saving (durability=="exit")
        if TASKS in self.checkpoint["channel_values"] and any(
            isinstance(channel, UntrackedValue) for channel in self.channels.values()
        ):
            sanitized_tasks = [
                sanitize_untracked_values_in_send(value, self.channels)
                if isinstance(value, Send)
                else value
                for value in self.checkpoint["channel_values"][TASKS]
            ]
            self.checkpoint["channel_values"][TASKS] = sanitized_tasks
        # bail if no checkpointer
        if do_checkpoint and self._checkpointer_put_after_previous is not None:
            self.prev_checkpoint_config = (
                self.checkpoint_config
                if CONFIG_KEY_CHECKPOINT_ID in self.checkpoint_config[CONF]
                and self.checkpoint_config[CONF][CONFIG_KEY_CHECKPOINT_ID]
                else None
            )
            self.checkpoint_config = {
                **self.checkpoint_config,
                CONF: {
                    **self.checkpoint_config[CONF],
                    CONFIG_KEY_CHECKPOINT_NS: self.config[CONF].get(
                        CONFIG_KEY_CHECKPOINT_NS, ""
                    ),
                },
            }

            channel_versions = self.checkpoint["channel_versions"].copy()
            new_versions = get_new_channel_versions(
                self.checkpoint_previous_versions, channel_versions
            )
            self.checkpoint_previous_versions = channel_versions

            # save it, without blocking
            # if there's a previous checkpoint save in progress, wait for it
            # ensuring checkpointers receive checkpoints in order
            self._put_checkpoint_fut = self.submit(
                self._checkpointer_put_after_previous,
                getattr(self, "_put_checkpoint_fut", None),
                self.checkpoint_config,
                copy_checkpoint(self.checkpoint),
                self.checkpoint_metadata,
                new_versions,
            )
            self.checkpoint_config = {
                **self.checkpoint_config,
                CONF: {
                    **self.checkpoint_config[CONF],
                    CONFIG_KEY_CHECKPOINT_ID: self.checkpoint["id"],
                },
            }
        if not exiting:
            # increment step
            self.step += 1

    def _suppress_interrupt(
        self,
        exc_type: type[BaseException] | None,
        exc_value: BaseException | None,
        traceback: TracebackType | None,
    ) -> bool | None:
        # persist current checkpoint and writes
        if self.durability == "exit" and (
            # if it's a top graph
            not self.is_nested
            # or a nested graph with error or interrupt
            or exc_value is not None
            # or a nested graph with checkpointer=True
            or all(NS_END not in part for part in self.checkpoint_ns)
        ):
            self._put_checkpoint(self.checkpoint_metadata)
            self._put_pending_writes()
        # suppress interrupt
        suppress = isinstance(exc_value, GraphInterrupt) and not self.is_nested
        if suppress:
            # emit one last "values" event, with pending writes applied
            if (
                hasattr(self, "tasks")
                and self.checkpoint_pending_writes
                and any(task.writes for task in self.tasks.values())
            ):
                updated_channels = apply_writes(
                    self.checkpoint,
                    self.channels,
                    self.tasks.values(),
                    self.checkpointer_get_next_version,
                    self.trigger_to_nodes,
                )
                if not updated_channels.isdisjoint(
                    (self.output_keys,)
                    if isinstance(self.output_keys, str)
                    else self.output_keys
                ):
                    self._emit(
                        "values",
                        map_output_values,
                        self.output_keys,
                        [w for t in self.tasks.values() for w in t.writes],
                        self.channels,
                    )
            # emit INTERRUPT if exception is empty (otherwise emitted by put_writes)
            if exc_value is not None and (not exc_value.args or not exc_value.args[0]):
                self._emit(
                    "updates",
                    lambda: iter(
                        [{INTERRUPT: cast(GraphInterrupt, exc_value).args[0]}]
                    ),
                )
            # save final output
            self.output = read_channels(self.channels, self.output_keys)
            # suppress interrupt
            return True
        elif exc_type is None:
            # save final output
            self.output = read_channels(self.channels, self.output_keys)

    def _emit(
        self,
        mode: StreamMode,
        values: Callable[P, Iterator[Any]],
        *args: P.args,
        **kwargs: P.kwargs,
    ) -> None:
        if self.stream is None:
            return
        debug_remap = mode in ("checkpoints", "tasks") and "debug" in self.stream.modes
        if mode not in self.stream.modes and not debug_remap:
            return
        for v in values(*args, **kwargs):
            if mode in self.stream.modes:
                self.stream((self.checkpoint_ns, mode, v))
            # "debug" mode is "checkpoints" or "tasks" with a wrapper dict
            if debug_remap:
                self.stream(
                    (
                        self.checkpoint_ns,
                        "debug",
                        {
                            "step": self.step - 1
                            if mode == "checkpoints"
                            else self.step,
                            "timestamp": datetime.now(timezone.utc).isoformat(),
                            "type": "checkpoint"
                            if mode == "checkpoints"
                            else "task_result"
                            if "result" in v
                            else "task",
                            "payload": v,
                        },
                    )
                )

    def output_writes(
        self, task_id: str, writes: WritesT, *, cached: bool = False
    ) -> None:
        if task := self.tasks.get(task_id):
            if task.config is not None and TAG_HIDDEN in task.config.get(
                "tags", EMPTY_SEQ
            ):
                return
            if writes[0][0] == INTERRUPT:
                # in loop.py we append a bool to the PUSH task paths to indicate
                # whether or not a call was present. If so,
                # we don't emit the interrupt as it'll be emitted by the parent
                if task.path[0] == PUSH and task.path[-1] is True:
                    return
                interrupts = [
                    {
                        INTERRUPT: tuple(
                            v
                            for w in writes
                            if w[0] == INTERRUPT
                            for v in (w[1] if isinstance(w[1], Sequence) else (w[1],))
                        )
                    }
                ]
                stream_modes = self.stream.modes if self.stream else []
                if "updates" in stream_modes:
                    self._emit("updates", lambda: iter(interrupts))
                elif "values" in stream_modes:
                    self._emit("values", lambda: iter(interrupts))
            elif writes[0][0] != ERROR:
                self._emit(
                    "updates",
                    map_output_updates,
                    self.output_keys,
                    [(task, writes)],
                    cached,
                )
            if not cached:
                self._emit(
                    "tasks",
                    map_debug_task_results,
                    (task, writes),
                    self.stream_keys,
                )


class SyncPregelLoop(PregelLoop, AbstractContextManager):
    def __init__(
        self,
        input: Any | None,
        *,
        stream: StreamProtocol | None,
        config: RunnableConfig,
        store: BaseStore | None,
        cache: BaseCache | None,
        checkpointer: BaseCheckpointSaver | None,
        nodes: Mapping[str, PregelNode],
        specs: Mapping[str, BaseChannel | ManagedValueSpec],
        trigger_to_nodes: Mapping[str, Sequence[str]],
        durability: Durability,
        manager: None | AsyncParentRunManager | ParentRunManager = None,
        interrupt_after: All | Sequence[str] = EMPTY_SEQ,
        interrupt_before: All | Sequence[str] = EMPTY_SEQ,
        input_keys: str | Sequence[str] = EMPTY_SEQ,
        output_keys: str | Sequence[str] = EMPTY_SEQ,
        stream_keys: str | Sequence[str] = EMPTY_SEQ,
        migrate_checkpoint: Callable[[Checkpoint], None] | None = None,
        retry_policy: Sequence[RetryPolicy] = (),
        cache_policy: CachePolicy | None = None,
    ) -> None:
        super().__init__(
            input,
            stream=stream,
            config=config,
            checkpointer=checkpointer,
            cache=cache,
            store=store,
            nodes=nodes,
            specs=specs,
            input_keys=input_keys,
            output_keys=output_keys,
            stream_keys=stream_keys,
            interrupt_after=interrupt_after,
            interrupt_before=interrupt_before,
            manager=manager,
            migrate_checkpoint=migrate_checkpoint,
            trigger_to_nodes=trigger_to_nodes,
            retry_policy=retry_policy,
            cache_policy=cache_policy,
            durability=durability,
        )
        self.stack = ExitStack()
        if checkpointer:
            self.checkpointer_get_next_version = checkpointer.get_next_version
            self.checkpointer_put_writes = checkpointer.put_writes
            self.checkpointer_put_writes_accepts_task_path = (
                signature(checkpointer.put_writes).parameters.get("task_path")
                is not None
            )
        else:
            self.checkpointer_get_next_version = increment
            self._checkpointer_put_after_previous = None  # type: ignore[assignment]
            self.checkpointer_put_writes = None
            self.checkpointer_put_writes_accepts_task_path = False

    def _checkpointer_put_after_previous(
        self,
        prev: concurrent.futures.Future | None,
        config: RunnableConfig,
        checkpoint: Checkpoint,
        metadata: CheckpointMetadata,
        new_versions: ChannelVersions,
    ) -> RunnableConfig:
        try:
            if prev is not None:
                prev.result()
        finally:
            cast(BaseCheckpointSaver, self.checkpointer).put(
                config, checkpoint, metadata, new_versions
            )

    def match_cached_writes(self) -> Sequence[PregelExecutableTask]:
        if self.cache is None:
            return ()
        matched: list[PregelExecutableTask] = []
        if cached := {
            (t.cache_key.ns, t.cache_key.key): t
            for t in self.tasks.values()
            if t.cache_key and not t.writes
        }:
            for key, values in self.cache.get(tuple(cached)).items():
                task = cached[key]
                task.writes.extend(values)
                matched.append(task)
        return matched

    def accept_push(
        self, task: PregelExecutableTask, write_idx: int, call: Call | None = None
    ) -> PregelExecutableTask | None:
        if pushed := super().accept_push(task, write_idx, call):
            for task in self.match_cached_writes():
                self.output_writes(task.id, task.writes, cached=True)
        return pushed

    def put_writes(self, task_id: str, writes: WritesT) -> None:
        """Put writes for a task, to be read by the next tick."""
        super().put_writes(task_id, writes)
        if not writes or self.cache is None or not hasattr(self, "tasks"):
            return
        task = self.tasks.get(task_id)
        if task is None or task.cache_key is None:
            return
        self.submit(
            self.cache.set,
            {
                (task.cache_key.ns, task.cache_key.key): (
                    task.writes,
                    task.cache_key.ttl,
                )
            },
        )

    # context manager

    def __enter__(self) -> Self:
        if self.checkpointer:
            saved = self.checkpointer.get_tuple(self.checkpoint_config)
        else:
            saved = None
        if saved is None:
            saved = CheckpointTuple(
                self.checkpoint_config, empty_checkpoint(), {"step": -2}, None, []
            )
        elif self._migrate_checkpoint is not None:
            self._migrate_checkpoint(saved.checkpoint)
        self.checkpoint_config = {
            **self.checkpoint_config,
            **saved.config,
            CONF: {
                CONFIG_KEY_CHECKPOINT_NS: "",
                **self.checkpoint_config.get(CONF, {}),
                **saved.config.get(CONF, {}),
            },
        }
        self.prev_checkpoint_config = saved.parent_config
        self.checkpoint_id_saved = saved.checkpoint["id"]
        self.checkpoint = saved.checkpoint
        self.checkpoint_metadata = saved.metadata
        self.checkpoint_pending_writes = (
            [(str(tid), k, v) for tid, k, v in saved.pending_writes]
            if saved.pending_writes is not None
            else []
        )

        self.submit = self.stack.enter_context(BackgroundExecutor(self.config))
        self.channels, self.managed = channels_from_checkpoint(
            self.specs, self.checkpoint
        )
        self.stack.push(self._suppress_interrupt)
        self.status = "input"
        self.step = self.checkpoint_metadata["step"] + 1
        self.stop = self.step + self.config["recursion_limit"] + 1
        self.checkpoint_previous_versions = self.checkpoint["channel_versions"].copy()
        self.updated_channels = self._first(
            input_keys=self.input_keys,
            updated_channels=set(self.checkpoint.get("updated_channels"))  # type: ignore[arg-type]
            if self.checkpoint.get("updated_channels")
            else None,
        )

        return self

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc_value: BaseException | None,
        traceback: TracebackType | None,
    ) -> bool | None:
        # unwind stack
        return self.stack.__exit__(exc_type, exc_value, traceback)


class AsyncPregelLoop(PregelLoop, AbstractAsyncContextManager):
    def __init__(
        self,
        input: Any | None,
        *,
        stream: StreamProtocol | None,
        config: RunnableConfig,
        store: BaseStore | None,
        cache: BaseCache | None,
        checkpointer: BaseCheckpointSaver | None,
        nodes: Mapping[str, PregelNode],
        specs: Mapping[str, BaseChannel | ManagedValueSpec],
        trigger_to_nodes: Mapping[str, Sequence[str]],
        durability: Durability,
        interrupt_after: All | Sequence[str] = EMPTY_SEQ,
        interrupt_before: All | Sequence[str] = EMPTY_SEQ,
        manager: None | AsyncParentRunManager | ParentRunManager = None,
        input_keys: str | Sequence[str] = EMPTY_SEQ,
        output_keys: str | Sequence[str] = EMPTY_SEQ,
        stream_keys: str | Sequence[str] = EMPTY_SEQ,
        migrate_checkpoint: Callable[[Checkpoint], None] | None = None,
        retry_policy: Sequence[RetryPolicy] = (),
        cache_policy: CachePolicy | None = None,
    ) -> None:
        super().__init__(
            input,
            stream=stream,
            config=config,
            checkpointer=checkpointer,
            cache=cache,
            store=store,
            nodes=nodes,
            specs=specs,
            input_keys=input_keys,
            output_keys=output_keys,
            stream_keys=stream_keys,
            interrupt_after=interrupt_after,
            interrupt_before=interrupt_before,
            manager=manager,
            migrate_checkpoint=migrate_checkpoint,
            trigger_to_nodes=trigger_to_nodes,
            retry_policy=retry_policy,
            cache_policy=cache_policy,
            durability=durability,
        )
        self.stack = AsyncExitStack()
        if checkpointer:
            self.checkpointer_get_next_version = checkpointer.get_next_version
            self.checkpointer_put_writes = checkpointer.aput_writes
            self.checkpointer_put_writes_accepts_task_path = (
                signature(checkpointer.aput_writes).parameters.get("task_path")
                is not None
            )
        else:
            self.checkpointer_get_next_version = increment
            self._checkpointer_put_after_previous = None  # type: ignore[assignment]
            self.checkpointer_put_writes = None
            self.checkpointer_put_writes_accepts_task_path = False

    async def _checkpointer_put_after_previous(
        self,
        prev: asyncio.Task | None,
        config: RunnableConfig,
        checkpoint: Checkpoint,
        metadata: CheckpointMetadata,
        new_versions: ChannelVersions,
    ) -> RunnableConfig:
        try:
            if prev is not None:
                await prev
        finally:
            await cast(BaseCheckpointSaver, self.checkpointer).aput(
                config, checkpoint, metadata, new_versions
            )

    async def amatch_cached_writes(self) -> Sequence[PregelExecutableTask]:
        if self.cache is None:
            return []
        matched: list[PregelExecutableTask] = []
        if cached := {
            (t.cache_key.ns, t.cache_key.key): t
            for t in self.tasks.values()
            if t.cache_key and not t.writes
        }:
            for key, values in (await self.cache.aget(tuple(cached))).items():
                task = cached[key]
                task.writes.extend(values)
                matched.append(task)
        return matched

    async def aaccept_push(
        self, task: PregelExecutableTask, write_idx: int, call: Call | None = None
    ) -> PregelExecutableTask | None:
        if pushed := super().accept_push(task, write_idx, call):
            for task in await self.amatch_cached_writes():
                self.output_writes(task.id, task.writes, cached=True)
        return pushed

    def put_writes(self, task_id: str, writes: WritesT) -> None:
        """Put writes for a task, to be read by the next tick."""
        super().put_writes(task_id, writes)
        if not writes or self.cache is None or not hasattr(self, "tasks"):
            return
        task = self.tasks.get(task_id)
        if task is None or task.cache_key is None:
            return
        if writes[0][0] in (INTERRUPT, ERROR):
            # only cache successful tasks
            return
        self.submit(
            self.cache.aset,
            {
                (task.cache_key.ns, task.cache_key.key): (
                    task.writes,
                    task.cache_key.ttl,
                )
            },
        )

    # context manager

    async def __aenter__(self) -> Self:
        if self.checkpointer:
            saved = await self.checkpointer.aget_tuple(self.checkpoint_config)
        else:
            saved = None
        if saved is None:
            saved = CheckpointTuple(
                self.checkpoint_config, empty_checkpoint(), {"step": -2}, None, []
            )
        elif self._migrate_checkpoint is not None:
            self._migrate_checkpoint(saved.checkpoint)
        self.checkpoint_config = {
            **self.checkpoint_config,
            **saved.config,
            CONF: {
                CONFIG_KEY_CHECKPOINT_NS: "",
                **self.checkpoint_config.get(CONF, {}),
                **saved.config.get(CONF, {}),
            },
        }
        self.prev_checkpoint_config = saved.parent_config
        self.checkpoint_id_saved = saved.checkpoint["id"]
        self.checkpoint = saved.checkpoint
        self.checkpoint_metadata = saved.metadata
        self.checkpoint_pending_writes = (
            [(str(tid), k, v) for tid, k, v in saved.pending_writes]
            if saved.pending_writes is not None
            else []
        )

        self.submit = await self.stack.enter_async_context(
            AsyncBackgroundExecutor(self.config)
        )
        self.channels, self.managed = channels_from_checkpoint(
            self.specs, self.checkpoint
        )
        self.stack.push(self._suppress_interrupt)
        self.status = "input"
        self.step = self.checkpoint_metadata["step"] + 1
        self.stop = self.step + self.config["recursion_limit"] + 1
        self.checkpoint_previous_versions = self.checkpoint["channel_versions"].copy()
        self.updated_channels = self._first(
            input_keys=self.input_keys,
            updated_channels=set(self.checkpoint.get("updated_channels"))  # type: ignore[arg-type]
            if self.checkpoint.get("updated_channels")
            else None,
        )

        return self

    async def __aexit__(
        self,
        exc_type: type[BaseException] | None,
        exc_value: BaseException | None,
        traceback: TracebackType | None,
    ) -> bool | None:
        # unwind stack
        exit_task = asyncio.create_task(
            self.stack.__aexit__(exc_type, exc_value, traceback)
        )
        try:
            return await exit_task
        except asyncio.CancelledError as e:
            # Bubble up the exit task upon cancellation to permit the API
            # consumer to await it before e.g., reusing the DB connection.
            e.args = (*e.args, exit_task)
            raise

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/_messages.py
```py
from __future__ import annotations

from collections.abc import AsyncIterator, Callable, Iterator, Sequence
from typing import (
    Any,
    TypeVar,
    cast,
)
from uuid import UUID, uuid4

from langchain_core.callbacks import BaseCallbackHandler
from langchain_core.messages import BaseMessage
from langchain_core.outputs import ChatGeneration, ChatGenerationChunk, LLMResult

from langgraph._internal._constants import NS_SEP
from langgraph.constants import TAG_HIDDEN, TAG_NOSTREAM
from langgraph.pregel.protocol import StreamChunk
from langgraph.types import Command

try:
    from langchain_core.tracers._streaming import _StreamingCallbackHandler
except ImportError:
    _StreamingCallbackHandler = object  # type: ignore

T = TypeVar("T")
Meta = tuple[tuple[str, ...], dict[str, Any]]


class StreamMessagesHandler(BaseCallbackHandler, _StreamingCallbackHandler):
    """A callback handler that implements stream_mode=messages.

    Collects messages from:
    (1) chat model stream events; and
    (2) node outputs.
    """

    run_inline = True
    """We want this callback to run in the main thread to avoid order/locking issues."""

    def __init__(
        self,
        stream: Callable[[StreamChunk], None],
        subgraphs: bool,
        *,
        parent_ns: tuple[str, ...] | None = None,
    ) -> None:
        """Configure the handler to stream messages from LLMs and nodes.

        Args:
            stream: A callable that takes a StreamChunk and emits it.
            subgraphs: Whether to emit messages from subgraphs.
            parent_ns: The namespace where the handler was created.
                We keep track of this namespace to allow calls to subgraphs that
                were explicitly requested as a stream with `messages` mode
                configured.

        Example:
            parent_ns is used to handle scenarios where the subgraph is explicitly
            streamed with `stream_mode="messages"`.

            ```python
            def parent_graph_node():
                # This node is in the parent graph.
                async for event in some_subgraph(..., stream_mode="messages"):
                    do something with event # <-- these events will be emitted
                return ...

            parent_graph.invoke(subgraphs=False)
            ```
        """
        self.stream = stream
        self.subgraphs = subgraphs
        self.metadata: dict[UUID, Meta] = {}
        self.seen: set[int | str] = set()
        self.parent_ns = parent_ns

    def _emit(self, meta: Meta, message: BaseMessage, *, dedupe: bool = False) -> None:
        if dedupe and message.id in self.seen:
            return
        else:
            if message.id is None:
                message.id = str(uuid4())
            self.seen.add(message.id)
            self.stream((meta[0], "messages", (message, meta[1])))

    def _find_and_emit_messages(self, meta: Meta, response: Any) -> None:
        if isinstance(response, BaseMessage):
            self._emit(meta, response, dedupe=True)
        elif isinstance(response, Sequence):
            for value in response:
                if isinstance(value, BaseMessage):
                    self._emit(meta, value, dedupe=True)
        elif isinstance(response, dict):
            for value in response.values():
                if isinstance(value, BaseMessage):
                    self._emit(meta, value, dedupe=True)
                elif isinstance(value, Sequence):
                    for item in value:
                        if isinstance(item, BaseMessage):
                            self._emit(meta, item, dedupe=True)
        elif hasattr(response, "__dir__") and callable(response.__dir__):
            for key in dir(response):
                try:
                    value = getattr(response, key)
                    if isinstance(value, BaseMessage):
                        self._emit(meta, value, dedupe=True)
                    elif isinstance(value, Sequence):
                        for item in value:
                            if isinstance(item, BaseMessage):
                                self._emit(meta, item, dedupe=True)
                except AttributeError:
                    pass

    def tap_output_aiter(
        self, run_id: UUID, output: AsyncIterator[T]
    ) -> AsyncIterator[T]:
        return output

    def tap_output_iter(self, run_id: UUID, output: Iterator[T]) -> Iterator[T]:
        return output

    def on_chat_model_start(
        self,
        serialized: dict[str, Any],
        messages: list[list[BaseMessage]],
        *,
        run_id: UUID,
        parent_run_id: UUID | None = None,
        tags: list[str] | None = None,
        metadata: dict[str, Any] | None = None,
        **kwargs: Any,
    ) -> Any:
        if metadata and (not tags or (TAG_NOSTREAM not in tags)):
            ns = tuple(cast(str, metadata["langgraph_checkpoint_ns"]).split(NS_SEP))[
                :-1
            ]
            if not self.subgraphs and len(ns) > 0 and ns != self.parent_ns:
                return
            if tags:
                if filtered_tags := [t for t in tags if not t.startswith("seq:step")]:
                    metadata["tags"] = filtered_tags
            self.metadata[run_id] = (ns, metadata)

    def on_llm_new_token(
        self,
        token: str,
        *,
        chunk: ChatGenerationChunk | None = None,
        run_id: UUID,
        parent_run_id: UUID | None = None,
        tags: list[str] | None = None,
        **kwargs: Any,
    ) -> Any:
        if not isinstance(chunk, ChatGenerationChunk):
            return
        if meta := self.metadata.get(run_id):
            self._emit(meta, chunk.message)

    def on_llm_end(
        self,
        response: LLMResult,
        *,
        run_id: UUID,
        parent_run_id: UUID | None = None,
        **kwargs: Any,
    ) -> Any:
        if meta := self.metadata.get(run_id):
            if response.generations and response.generations[0]:
                gen = response.generations[0][0]
                if isinstance(gen, ChatGeneration):
                    self._emit(meta, gen.message, dedupe=True)
        self.metadata.pop(run_id, None)

    def on_llm_error(
        self,
        error: BaseException,
        *,
        run_id: UUID,
        parent_run_id: UUID | None = None,
        **kwargs: Any,
    ) -> Any:
        self.metadata.pop(run_id, None)

    def on_chain_start(
        self,
        serialized: dict[str, Any],
        inputs: dict[str, Any],
        *,
        run_id: UUID,
        parent_run_id: UUID | None = None,
        tags: list[str] | None = None,
        metadata: dict[str, Any] | None = None,
        **kwargs: Any,
    ) -> Any:
        if (
            metadata
            and kwargs.get("name") == metadata.get("langgraph_node")
            and (not tags or TAG_HIDDEN not in tags)
        ):
            ns = tuple(cast(str, metadata["langgraph_checkpoint_ns"]).split(NS_SEP))[
                :-1
            ]
            if not self.subgraphs and len(ns) > 0:
                return
            self.metadata[run_id] = (ns, metadata)
            if isinstance(inputs, dict):
                for key, value in inputs.items():
                    if isinstance(value, BaseMessage):
                        if value.id is not None:
                            self.seen.add(value.id)
                    elif isinstance(value, Sequence) and not isinstance(value, str):
                        for item in value:
                            if isinstance(item, BaseMessage):
                                if item.id is not None:
                                    self.seen.add(item.id)

    def on_chain_end(
        self,
        response: Any,
        *,
        run_id: UUID,
        parent_run_id: UUID | None = None,
        **kwargs: Any,
    ) -> Any:
        if meta := self.metadata.pop(run_id, None):
            # Handle Command node updates
            if isinstance(response, Command):
                self._find_and_emit_messages(meta, response.update)
            # Handle list of Command updates
            elif isinstance(response, Sequence) and any(
                isinstance(value, Command) for value in response
            ):
                for value in response:
                    if isinstance(value, Command):
                        self._find_and_emit_messages(meta, value.update)
                    else:
                        self._find_and_emit_messages(meta, value)
            # Handle basic updates / streaming
            else:
                self._find_and_emit_messages(meta, response)

    def on_chain_error(
        self,
        error: BaseException,
        *,
        run_id: UUID,
        parent_run_id: UUID | None = None,
        **kwargs: Any,
    ) -> Any:
        self.metadata.pop(run_id, None)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/_read.py
```py
from __future__ import annotations

from collections.abc import AsyncIterator, Callable, Iterator, Mapping, Sequence
from functools import cached_property
from typing import (
    Any,
)

from langchain_core.runnables import Runnable, RunnableConfig

from langgraph._internal._config import merge_configs
from langgraph._internal._constants import CONF, CONFIG_KEY_READ
from langgraph._internal._runnable import RunnableCallable, RunnableSeq
from langgraph.pregel._utils import find_subgraph_pregel
from langgraph.pregel._write import ChannelWrite
from langgraph.pregel.protocol import PregelProtocol
from langgraph.types import CachePolicy, RetryPolicy

READ_TYPE = Callable[[str | Sequence[str], bool], Any | dict[str, Any]]
INPUT_CACHE_KEY_TYPE = tuple[Callable[..., Any], tuple[str, ...]]


class ChannelRead(RunnableCallable):
    """Implements the logic for reading state from CONFIG_KEY_READ.
    Usable both as a runnable as well as a static method to call imperatively."""

    channel: str | list[str]

    fresh: bool = False

    mapper: Callable[[Any], Any] | None = None

    def __init__(
        self,
        channel: str | list[str],
        *,
        fresh: bool = False,
        mapper: Callable[[Any], Any] | None = None,
        tags: list[str] | None = None,
    ) -> None:
        super().__init__(
            func=self._read,
            afunc=self._aread,
            tags=tags,
            name=None,
            trace=False,
        )
        self.fresh = fresh
        self.mapper = mapper
        self.channel = channel

    def get_name(self, suffix: str | None = None, *, name: str | None = None) -> str:
        if name:
            pass
        elif isinstance(self.channel, str):
            name = f"ChannelRead<{self.channel}>"
        else:
            name = f"ChannelRead<{','.join(self.channel)}>"
        return super().get_name(suffix, name=name)

    def _read(self, _: Any, config: RunnableConfig) -> Any:
        return self.do_read(
            config, select=self.channel, fresh=self.fresh, mapper=self.mapper
        )

    async def _aread(self, _: Any, config: RunnableConfig) -> Any:
        return self.do_read(
            config, select=self.channel, fresh=self.fresh, mapper=self.mapper
        )

    @staticmethod
    def do_read(
        config: RunnableConfig,
        *,
        select: str | list[str],
        fresh: bool = False,
        mapper: Callable[[Any], Any] | None = None,
    ) -> Any:
        try:
            read: READ_TYPE = config[CONF][CONFIG_KEY_READ]
        except KeyError:
            raise RuntimeError(
                "Not configured with a read function"
                "Make sure to call in the context of a Pregel process"
            )
        if mapper:
            return mapper(read(select, fresh))
        else:
            return read(select, fresh)


DEFAULT_BOUND = RunnableCallable(lambda input: input)


class PregelNode:
    """A node in a Pregel graph. This won't be invoked as a runnable by the graph
    itself, but instead acts as a container for the components necessary to make
    a PregelExecutableTask for a node."""

    channels: str | list[str]
    """The channels that will be passed as input to `bound`.
    If a str, the node will be invoked with its value if it isn't empty.
    If a list, the node will be invoked with a dict of those channels' values."""

    triggers: list[str]
    """If any of these channels is written to, this node will be triggered in
    the next step."""

    mapper: Callable[[Any], Any] | None
    """A function to transform the input before passing it to `bound`."""

    writers: list[Runnable]
    """A list of writers that will be executed after `bound`, responsible for
    taking the output of `bound` and writing it to the appropriate channels."""

    bound: Runnable[Any, Any]
    """The main logic of the node. This will be invoked with the input from 
    `channels`."""

    retry_policy: Sequence[RetryPolicy] | None
    """The retry policies to use when invoking the node."""

    cache_policy: CachePolicy | None
    """The cache policy to use when invoking the node."""

    tags: Sequence[str] | None
    """Tags to attach to the node for tracing."""

    metadata: Mapping[str, Any] | None
    """Metadata to attach to the node for tracing."""

    subgraphs: Sequence[PregelProtocol]
    """Subgraphs used by the node."""

    def __init__(
        self,
        *,
        channels: str | list[str],
        triggers: Sequence[str],
        mapper: Callable[[Any], Any] | None = None,
        writers: list[Runnable] | None = None,
        tags: list[str] | None = None,
        metadata: Mapping[str, Any] | None = None,
        bound: Runnable[Any, Any] | None = None,
        retry_policy: RetryPolicy | Sequence[RetryPolicy] | None = None,
        cache_policy: CachePolicy | None = None,
        subgraphs: Sequence[PregelProtocol] | None = None,
    ) -> None:
        self.channels = channels
        self.triggers = list(triggers)
        self.mapper = mapper
        self.writers = writers or []
        self.bound = bound if bound is not None else DEFAULT_BOUND
        self.cache_policy = cache_policy
        if isinstance(retry_policy, RetryPolicy):
            self.retry_policy = (retry_policy,)
        else:
            self.retry_policy = retry_policy
        self.tags = tags
        self.metadata = metadata
        if subgraphs is not None:
            self.subgraphs = subgraphs
        elif self.bound is not DEFAULT_BOUND:
            try:
                subgraph = find_subgraph_pregel(self.bound)
            except Exception:
                subgraph = None
            if subgraph:
                self.subgraphs = [subgraph]
            else:
                self.subgraphs = []
        else:
            self.subgraphs = []

    def copy(self, update: dict[str, Any]) -> PregelNode:
        attrs = {**self.__dict__, **update}
        # Drop the cached properties
        attrs.pop("flat_writers", None)
        attrs.pop("node", None)
        attrs.pop("input_cache_key", None)
        return PregelNode(**attrs)

    @cached_property
    def flat_writers(self) -> list[Runnable]:
        """Get writers with optimizations applied. Dedupes consecutive ChannelWrites."""
        writers = self.writers.copy()
        while (
            len(writers) > 1
            and isinstance(writers[-1], ChannelWrite)
            and isinstance(writers[-2], ChannelWrite)
        ):
            # we can combine writes if they are consecutive
            # careful to not modify the original writers list or ChannelWrite
            writers[-2] = ChannelWrite(
                writes=writers[-2].writes + writers[-1].writes,
            )
            writers.pop()
        return writers

    @cached_property
    def node(self) -> Runnable[Any, Any] | None:
        """Get a runnable that combines `bound` and `writers`."""
        writers = self.flat_writers
        if self.bound is DEFAULT_BOUND and not writers:
            return None
        elif self.bound is DEFAULT_BOUND and len(writers) == 1:
            return writers[0]
        elif self.bound is DEFAULT_BOUND:
            return RunnableSeq(*writers)
        elif writers:
            return RunnableSeq(self.bound, *writers)
        else:
            return self.bound

    @cached_property
    def input_cache_key(self) -> INPUT_CACHE_KEY_TYPE:
        """Get a cache key for the input to the node.
        This is used to avoid calculating the same input multiple times."""
        return (
            self.mapper,
            tuple(self.channels)
            if isinstance(self.channels, list)
            else (self.channels,),
        )

    def invoke(
        self,
        input: Any,
        config: RunnableConfig | None = None,
        **kwargs: Any | None,
    ) -> Any:
        self_config: RunnableConfig = {"metadata": self.metadata, "tags": self.tags}
        return self.bound.invoke(
            input,
            merge_configs(self_config, config),
            **kwargs,
        )

    async def ainvoke(
        self,
        input: Any,
        config: RunnableConfig | None = None,
        **kwargs: Any | None,
    ) -> Any:
        self_config: RunnableConfig = {"metadata": self.metadata, "tags": self.tags}
        return await self.bound.ainvoke(
            input,
            merge_configs(self_config, config),
            **kwargs,
        )

    def stream(
        self,
        input: Any,
        config: RunnableConfig | None = None,
        **kwargs: Any | None,
    ) -> Iterator[Any]:
        self_config: RunnableConfig = {"metadata": self.metadata, "tags": self.tags}
        yield from self.bound.stream(
            input,
            merge_configs(self_config, config),
            **kwargs,
        )

    async def astream(
        self,
        input: Any,
        config: RunnableConfig | None = None,
        **kwargs: Any | None,
    ) -> AsyncIterator[Any]:
        self_config: RunnableConfig = {"metadata": self.metadata, "tags": self.tags}
        async for item in self.bound.astream(
            input,
            merge_configs(self_config, config),
            **kwargs,
        ):
            yield item

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/_retry.py
```py
from __future__ import annotations

import asyncio
import logging
import random
import sys
import time
from collections.abc import Awaitable, Callable, Sequence
from dataclasses import replace
from typing import Any

from langgraph._internal._config import patch_configurable
from langgraph._internal._constants import (
    CONF,
    CONFIG_KEY_CHECKPOINT_NS,
    CONFIG_KEY_RESUMING,
    NS_SEP,
)
from langgraph.errors import GraphBubbleUp, ParentCommand
from langgraph.types import Command, PregelExecutableTask, RetryPolicy

logger = logging.getLogger(__name__)
SUPPORTS_EXC_NOTES = sys.version_info >= (3, 11)


def run_with_retry(
    task: PregelExecutableTask,
    retry_policy: Sequence[RetryPolicy] | None,
    configurable: dict[str, Any] | None = None,
) -> None:
    """Run a task with retries."""
    retry_policy = task.retry_policy or retry_policy
    attempts = 0
    config = task.config
    if configurable is not None:
        config = patch_configurable(config, configurable)
    while True:
        try:
            # clear any writes from previous attempts
            task.writes.clear()
            # run the task
            return task.proc.invoke(task.input, config)
        except ParentCommand as exc:
            ns: str = config[CONF][CONFIG_KEY_CHECKPOINT_NS]
            cmd = exc.args[0]
            if cmd.graph in (ns, task.name):
                # this command is for the current graph, handle it
                for w in task.writers:
                    w.invoke(cmd, config)
                break
            elif cmd.graph == Command.PARENT:
                # this command is for the parent graph, assign it to the parent
                parts = ns.split(NS_SEP)
                if parts[-1].isdigit():
                    parts.pop()
                parent_ns = NS_SEP.join(parts[:-1])
                exc.args = (replace(cmd, graph=parent_ns),)
            # bubble up
            raise
        except GraphBubbleUp:
            # if interrupted, end
            raise
        except Exception as exc:
            if SUPPORTS_EXC_NOTES:
                exc.add_note(f"During task with name '{task.name}' and id '{task.id}'")
            if not retry_policy:
                raise

            # Check which retry policy applies to this exception
            matching_policy = None
            for policy in retry_policy:
                if _should_retry_on(policy, exc):
                    matching_policy = policy
                    break

            if not matching_policy:
                raise

            # increment attempts
            attempts += 1
            # check if we should give up
            if attempts >= matching_policy.max_attempts:
                raise
            # sleep before retrying
            interval = matching_policy.initial_interval
            # Apply backoff factor based on attempt count
            interval = min(
                matching_policy.max_interval,
                interval * (matching_policy.backoff_factor ** (attempts - 1)),
            )

            # Apply jitter if configured
            sleep_time = (
                interval + random.uniform(0, 1) if matching_policy.jitter else interval
            )
            time.sleep(sleep_time)

            # log the retry
            logger.info(
                f"Retrying task {task.name} after {sleep_time:.2f} seconds (attempt {attempts}) after {exc.__class__.__name__} {exc}",
                exc_info=exc,
            )
            # signal subgraphs to resume (if available)
            config = patch_configurable(config, {CONFIG_KEY_RESUMING: True})


async def arun_with_retry(
    task: PregelExecutableTask,
    retry_policy: Sequence[RetryPolicy] | None,
    stream: bool = False,
    match_cached_writes: Callable[[], Awaitable[Sequence[PregelExecutableTask]]]
    | None = None,
    configurable: dict[str, Any] | None = None,
) -> None:
    """Run a task asynchronously with retries."""
    retry_policy = task.retry_policy or retry_policy
    attempts = 0
    config = task.config
    if configurable is not None:
        config = patch_configurable(config, configurable)
    if match_cached_writes is not None and task.cache_key is not None:
        for t in await match_cached_writes():
            if t is task:
                # if the task is already cached, return
                return
    while True:
        try:
            # clear any writes from previous attempts
            task.writes.clear()
            # run the task
            if stream:
                async for _ in task.proc.astream(task.input, config):
                    pass
                # if successful, end
                break
            else:
                return await task.proc.ainvoke(task.input, config)
        except ParentCommand as exc:
            ns: str = config[CONF][CONFIG_KEY_CHECKPOINT_NS]
            cmd = exc.args[0]
            if cmd.graph in (ns, task.name):
                # this command is for the current graph, handle it
                for w in task.writers:
                    w.invoke(cmd, config)
                break
            elif cmd.graph == Command.PARENT:
                # this command is for the parent graph, assign it to the parent
                parts = ns.split(NS_SEP)
                if parts[-1].isdigit():
                    parts.pop()
                parent_ns = NS_SEP.join(parts[:-1])
                exc.args = (replace(cmd, graph=parent_ns),)
            # bubble up
            raise
        except GraphBubbleUp:
            # if interrupted, end
            raise
        except Exception as exc:
            if SUPPORTS_EXC_NOTES:
                exc.add_note(f"During task with name '{task.name}' and id '{task.id}'")
            if not retry_policy:
                raise

            # Check which retry policy applies to this exception
            matching_policy = None
            for policy in retry_policy:
                if _should_retry_on(policy, exc):
                    matching_policy = policy
                    break

            if not matching_policy:
                raise

            # increment attempts
            attempts += 1
            # check if we should give up
            if attempts >= matching_policy.max_attempts:
                raise
            # sleep before retrying
            interval = matching_policy.initial_interval
            # Apply backoff factor based on attempt count
            interval = min(
                matching_policy.max_interval,
                interval * (matching_policy.backoff_factor ** (attempts - 1)),
            )

            # Apply jitter if configured
            sleep_time = (
                interval + random.uniform(0, 1) if matching_policy.jitter else interval
            )
            await asyncio.sleep(sleep_time)

            # log the retry
            logger.info(
                f"Retrying task {task.name} after {sleep_time:.2f} seconds (attempt {attempts}) after {exc.__class__.__name__} {exc}",
                exc_info=exc,
            )
            # signal subgraphs to resume (if available)
            config = patch_configurable(config, {CONFIG_KEY_RESUMING: True})


def _should_retry_on(retry_policy: RetryPolicy, exc: Exception) -> bool:
    """Check if the given exception should be retried based on the retry policy."""
    if isinstance(retry_policy.retry_on, Sequence):
        return isinstance(exc, tuple(retry_policy.retry_on))
    elif isinstance(retry_policy.retry_on, type) and issubclass(
        retry_policy.retry_on, Exception
    ):
        return isinstance(exc, retry_policy.retry_on)
    elif callable(retry_policy.retry_on):
        return retry_policy.retry_on(exc)  # type: ignore[call-arg]
    else:
        raise TypeError(
            "retry_on must be an Exception class, a list or tuple of Exception classes, or a callable"
        )

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/_runner.py
```py
from __future__ import annotations

import asyncio
import concurrent.futures
import inspect
import threading
import time
import weakref
from collections.abc import (
    AsyncIterator,
    Awaitable,
    Callable,
    Iterable,
    Iterator,
    Sequence,
)
from functools import partial
from typing import (
    Any,
    Generic,
    TypeVar,
    cast,
)

from langchain_core.callbacks import Callbacks

from langgraph._internal._constants import (
    CONF,
    CONFIG_KEY_CALL,
    CONFIG_KEY_SCRATCHPAD,
    ERROR,
    INTERRUPT,
    NO_WRITES,
    RESUME,
    RETURN,
)
from langgraph._internal._future import chain_future, run_coroutine_threadsafe
from langgraph._internal._scratchpad import PregelScratchpad
from langgraph._internal._typing import MISSING
from langgraph.constants import TAG_HIDDEN
from langgraph.errors import GraphBubbleUp, GraphInterrupt
from langgraph.pregel._algo import Call
from langgraph.pregel._executor import Submit
from langgraph.pregel._retry import arun_with_retry, run_with_retry
from langgraph.types import (
    CachePolicy,
    PregelExecutableTask,
    RetryPolicy,
)

F = TypeVar("F", concurrent.futures.Future, asyncio.Future)
E = TypeVar("E", threading.Event, asyncio.Event)

# List of filenames to exclude from exception traceback
# Note: Frames will be removed if they are the last frame in traceback, recursively
EXCLUDED_FRAME_FNAMES = (
    "langgraph/pregel/retry.py",
    "langgraph/pregel/runner.py",
    "langgraph/pregel/executor.py",
    "langgraph/utils/runnable.py",
    "langchain_core/runnables/config.py",
    "concurrent/futures/thread.py",
    "concurrent/futures/_base.py",
)

SKIP_RERAISE_SET: weakref.WeakSet[concurrent.futures.Future | asyncio.Future] = (
    weakref.WeakSet()
)


class FuturesDict(Generic[F, E], dict[F, PregelExecutableTask | None]):
    event: E
    callback: weakref.ref[Callable[[PregelExecutableTask, BaseException | None], None]]
    counter: int
    done: set[F]
    lock: threading.Lock

    def __init__(
        self,
        event: E,
        callback: weakref.ref[
            Callable[[PregelExecutableTask, BaseException | None], None]
        ],
        future_type: type[F],
        # used for generic typing, newer py supports FutureDict[...](...)
    ) -> None:
        super().__init__()
        self.lock = threading.Lock()
        self.event = event
        self.callback = callback
        self.counter = 0
        self.done: set[F] = set()

    def __setitem__(
        self,
        key: F,
        value: PregelExecutableTask | None,
    ) -> None:
        super().__setitem__(key, value)  # type: ignore[index]
        if value is not None:
            with self.lock:
                self.event.clear()
                self.counter += 1
            key.add_done_callback(partial(self.on_done, value))

    def on_done(
        self,
        task: PregelExecutableTask,
        fut: F,
    ) -> None:
        try:
            if cb := self.callback():
                cb(task, _exception(fut))
        finally:
            with self.lock:
                self.done.add(fut)
                self.counter -= 1
                if self.counter == 0 or _should_stop_others(self.done):
                    self.event.set()


class PregelRunner:
    """Responsible for executing a set of Pregel tasks concurrently, committing
    their writes, yielding control to caller when there is output to emit, and
    interrupting other tasks if appropriate."""

    def __init__(
        self,
        *,
        submit: weakref.ref[Submit],
        put_writes: weakref.ref[Callable[[str, Sequence[tuple[str, Any]]], None]],
        use_astream: bool = False,
        node_finished: Callable[[str], None] | None = None,
    ) -> None:
        self.submit = submit
        self.put_writes = put_writes
        self.use_astream = use_astream
        self.node_finished = node_finished

    def tick(
        self,
        tasks: Iterable[PregelExecutableTask],
        *,
        reraise: bool = True,
        timeout: float | None = None,
        retry_policy: Sequence[RetryPolicy] | None = None,
        get_waiter: Callable[[], concurrent.futures.Future[None]] | None = None,
        schedule_task: Callable[
            [PregelExecutableTask, int, Call | None],
            PregelExecutableTask | None,
        ],
    ) -> Iterator[None]:
        tasks = tuple(tasks)
        futures = FuturesDict(
            callback=weakref.WeakMethod(self.commit),
            event=threading.Event(),
            future_type=concurrent.futures.Future,
        )
        # give control back to the caller
        yield
        # fast path if single task with no timeout and no waiter
        if len(tasks) == 0:
            return
        elif len(tasks) == 1 and timeout is None and get_waiter is None:
            t = tasks[0]
            try:
                run_with_retry(
                    t,
                    retry_policy,
                    configurable={
                        CONFIG_KEY_CALL: partial(
                            _call,
                            weakref.ref(t),
                            retry_policy=retry_policy,
                            futures=weakref.ref(futures),
                            schedule_task=schedule_task,
                            submit=self.submit,
                        ),
                    },
                )
                self.commit(t, None)
            except Exception as exc:
                self.commit(t, exc)
                if reraise and futures:
                    # will be re-raised after futures are done
                    fut: concurrent.futures.Future = concurrent.futures.Future()
                    fut.set_exception(exc)
                    futures.done.add(fut)
                elif reraise:
                    if tb := exc.__traceback__:
                        while tb.tb_next is not None and any(
                            tb.tb_frame.f_code.co_filename.endswith(name)
                            for name in EXCLUDED_FRAME_FNAMES
                        ):
                            tb = tb.tb_next
                        exc.__traceback__ = tb
                    raise
            if not futures:  # maybe `t` scheduled another task
                return
            else:
                tasks = ()  # don't reschedule this task
        # add waiter task if requested
        if get_waiter is not None:
            futures[get_waiter()] = None
        # schedule tasks
        for t in tasks:
            fut = self.submit()(  # type: ignore[misc]
                run_with_retry,
                t,
                retry_policy,
                configurable={
                    CONFIG_KEY_CALL: partial(
                        _call,
                        weakref.ref(t),
                        retry_policy=retry_policy,
                        futures=weakref.ref(futures),
                        schedule_task=schedule_task,
                        submit=self.submit,
                    ),
                },
                __reraise_on_exit__=reraise,
            )
            futures[fut] = t
        # execute tasks, and wait for one to fail or all to finish.
        # each task is independent from all other concurrent tasks
        # yield updates/debug output as each task finishes
        end_time = timeout + time.monotonic() if timeout else None
        while len(futures) > (1 if get_waiter is not None else 0):
            done, inflight = concurrent.futures.wait(
                futures,
                return_when=concurrent.futures.FIRST_COMPLETED,
                timeout=(max(0, end_time - time.monotonic()) if end_time else None),
            )
            if not done:
                break  # timed out
            for fut in done:
                task = futures.pop(fut)
                if task is None:
                    # waiter task finished, schedule another
                    if inflight and get_waiter is not None:
                        futures[get_waiter()] = None
            else:
                # remove references to loop vars
                del fut, task
            # maybe stop other tasks
            if _should_stop_others(done):
                break
            # give control back to the caller
            yield
        # wait for done callbacks
        futures.event.wait(
            timeout=(max(0, end_time - time.monotonic()) if end_time else None)
        )
        # give control back to the caller
        yield
        # panic on failure or timeout
        try:
            _panic_or_proceed(
                futures.done.union(f for f, t in futures.items() if t is not None),
                panic=reraise,
            )
        except Exception as exc:
            if tb := exc.__traceback__:
                while tb.tb_next is not None and any(
                    tb.tb_frame.f_code.co_filename.endswith(name)
                    for name in EXCLUDED_FRAME_FNAMES
                ):
                    tb = tb.tb_next
                exc.__traceback__ = tb
            raise

    async def atick(
        self,
        tasks: Iterable[PregelExecutableTask],
        *,
        reraise: bool = True,
        timeout: float | None = None,
        retry_policy: Sequence[RetryPolicy] | None = None,
        get_waiter: Callable[[], asyncio.Future[None]] | None = None,
        schedule_task: Callable[
            [PregelExecutableTask, int, Call | None],
            Awaitable[PregelExecutableTask | None],
        ],
    ) -> AsyncIterator[None]:
        try:
            loop = asyncio.get_event_loop()
        except RuntimeError:
            loop = asyncio.new_event_loop()
        asyncio.set_event_loop(loop)
        tasks = tuple(tasks)
        futures = FuturesDict(
            callback=weakref.WeakMethod(self.commit),
            event=asyncio.Event(),
            future_type=asyncio.Future,
        )
        # give control back to the caller
        yield
        # fast path if single task with no waiter and no timeout
        if len(tasks) == 0:
            return
        elif len(tasks) == 1 and get_waiter is None and timeout is None:
            t = tasks[0]
            try:
                await arun_with_retry(
                    t,
                    retry_policy,
                    stream=self.use_astream,
                    configurable={
                        CONFIG_KEY_CALL: partial(
                            _acall,
                            weakref.ref(t),
                            stream=self.use_astream,
                            retry_policy=retry_policy,
                            futures=weakref.ref(futures),
                            schedule_task=schedule_task,
                            submit=self.submit,
                            loop=loop,
                        ),
                    },
                )
                self.commit(t, None)
            except Exception as exc:
                self.commit(t, exc)
                if reraise and futures:
                    # will be re-raised after futures are done
                    fut: asyncio.Future = loop.create_future()
                    fut.set_exception(exc)
                    futures.done.add(fut)
                elif reraise:
                    if tb := exc.__traceback__:
                        while tb.tb_next is not None and any(
                            tb.tb_frame.f_code.co_filename.endswith(name)
                            for name in EXCLUDED_FRAME_FNAMES
                        ):
                            tb = tb.tb_next
                        exc.__traceback__ = tb
                    raise
            if not futures:  # maybe `t` scheduled another task
                return
            else:
                tasks = ()  # don't reschedule this task
        # add waiter task if requested
        if get_waiter is not None:
            futures[get_waiter()] = None
        # schedule tasks
        for t in tasks:
            fut = cast(
                asyncio.Future,
                self.submit()(  # type: ignore[misc]
                    arun_with_retry,
                    t,
                    retry_policy,
                    stream=self.use_astream,
                    configurable={
                        CONFIG_KEY_CALL: partial(
                            _acall,
                            weakref.ref(t),
                            retry_policy=retry_policy,
                            stream=self.use_astream,
                            futures=weakref.ref(futures),
                            schedule_task=schedule_task,
                            submit=self.submit,
                            loop=loop,
                        ),
                    },
                    __name__=t.name,
                    __cancel_on_exit__=True,
                    __reraise_on_exit__=reraise,
                ),
            )
            futures[fut] = t
        # execute tasks, and wait for one to fail or all to finish.
        # each task is independent from all other concurrent tasks
        # yield updates/debug output as each task finishes
        end_time = timeout + loop.time() if timeout else None
        while len(futures) > (1 if get_waiter is not None else 0):
            done, inflight = await asyncio.wait(
                futures,
                return_when=asyncio.FIRST_COMPLETED,
                timeout=(max(0, end_time - loop.time()) if end_time else None),
            )
            if not done:
                break  # timed out
            for fut in done:
                task = futures.pop(fut)
                if task is None:
                    # waiter task finished, schedule another
                    if inflight and get_waiter is not None:
                        futures[get_waiter()] = None
            else:
                # remove references to loop vars
                del fut, task
            # maybe stop other tasks
            if _should_stop_others(done):
                break
            # give control back to the caller
            yield
        # wait for done callbacks
        await asyncio.wait_for(
            futures.event.wait(),
            timeout=(max(0, end_time - loop.time()) if end_time else None),
        )
        # give control back to the caller
        yield
        # cancel waiter task
        for fut in futures:
            fut.cancel()
        # panic on failure or timeout
        try:
            _panic_or_proceed(
                futures.done.union(f for f, t in futures.items() if t is not None),
                timeout_exc_cls=asyncio.TimeoutError,
                panic=reraise,
            )
        except Exception as exc:
            if tb := exc.__traceback__:
                while tb.tb_next is not None and any(
                    tb.tb_frame.f_code.co_filename.endswith(name)
                    for name in EXCLUDED_FRAME_FNAMES
                ):
                    tb = tb.tb_next
                exc.__traceback__ = tb
            raise

    def commit(
        self,
        task: PregelExecutableTask,
        exception: BaseException | None,
    ) -> None:
        if isinstance(exception, asyncio.CancelledError):
            # for cancelled tasks, also save error in task,
            # so loop can finish super-step
            task.writes.append((ERROR, exception))
            self.put_writes()(task.id, task.writes)  # type: ignore[misc]
        elif exception:
            if isinstance(exception, GraphInterrupt):
                # save interrupt to checkpointer
                if exception.args[0]:
                    writes = [(INTERRUPT, exception.args[0])]
                    if resumes := [w for w in task.writes if w[0] == RESUME]:
                        writes.extend(resumes)
                    self.put_writes()(task.id, writes)  # type: ignore[misc]
            elif isinstance(exception, GraphBubbleUp):
                # exception will be raised in _panic_or_proceed
                pass
            else:
                # save error to checkpointer
                task.writes.append((ERROR, exception))
                self.put_writes()(task.id, task.writes)  # type: ignore[misc]
        else:
            if self.node_finished and (
                task.config is None or TAG_HIDDEN not in task.config.get("tags", [])
            ):
                self.node_finished(task.name)
            if not task.writes:
                # add no writes marker
                task.writes.append((NO_WRITES, None))
            # save task writes to checkpointer
            self.put_writes()(task.id, task.writes)  # type: ignore[misc]


def _should_stop_others(
    done: set[F],
) -> bool:
    """Check if any task failed, if so, cancel all other tasks.
    GraphInterrupts are not considered failures."""
    for fut in done:
        if fut.cancelled():
            continue
        elif exc := fut.exception():
            if not isinstance(exc, GraphBubbleUp) and fut not in SKIP_RERAISE_SET:
                return True

    return False


def _exception(
    fut: concurrent.futures.Future[Any] | asyncio.Future[Any],
) -> BaseException | None:
    """Return the exception from a future, without raising CancelledError."""
    if fut.cancelled():
        if isinstance(fut, asyncio.Future):
            return asyncio.CancelledError()
        else:
            return concurrent.futures.CancelledError()
    else:
        return fut.exception()


def _panic_or_proceed(
    futs: set[concurrent.futures.Future] | set[asyncio.Future],
    *,
    timeout_exc_cls: type[Exception] = TimeoutError,
    panic: bool = True,
) -> None:
    """Cancel remaining tasks if any failed, re-raise exception if panic is True."""
    done: set[concurrent.futures.Future[Any] | asyncio.Future[Any]] = set()
    inflight: set[concurrent.futures.Future[Any] | asyncio.Future[Any]] = set()
    for fut in futs:
        if fut.cancelled():
            continue
        elif fut.done():
            done.add(fut)
        else:
            inflight.add(fut)
    interrupts: list[GraphInterrupt] = []
    while done:
        # if any task failed
        fut = done.pop()
        if exc := _exception(fut):
            # cancel all pending tasks
            while inflight:
                inflight.pop().cancel()
            # raise the exception
            if panic:
                if isinstance(exc, GraphInterrupt):
                    # collect interrupts
                    interrupts.append(exc)
                elif fut not in SKIP_RERAISE_SET:
                    raise exc
    # raise combined interrupts
    if interrupts:
        raise GraphInterrupt(tuple(i for exc in interrupts for i in exc.args[0]))
    if inflight:
        # if we got here means we timed out
        while inflight:
            # cancel all pending tasks
            inflight.pop().cancel()
        # raise timeout error
        raise timeout_exc_cls("Timed out")


def _call(
    task: weakref.ref[PregelExecutableTask],
    func: Callable[[Any], Awaitable[Any] | Any],
    input: Any,
    *,
    retry_policy: Sequence[RetryPolicy] | None = None,
    cache_policy: CachePolicy | None = None,
    callbacks: Callbacks = None,
    futures: weakref.ref[FuturesDict],
    schedule_task: Callable[
        [PregelExecutableTask, int, Call | None], PregelExecutableTask | None
    ],
    submit: weakref.ref[Submit],
) -> concurrent.futures.Future[Any]:
    if inspect.iscoroutinefunction(func):
        raise RuntimeError("In an sync context async tasks cannot be called")

    fut: concurrent.futures.Future | None = None
    # schedule PUSH tasks, collect futures
    scratchpad: PregelScratchpad = task().config[CONF][CONFIG_KEY_SCRATCHPAD]  # type: ignore[union-attr]
    # schedule the next task, if the callback returns one
    if next_task := schedule_task(
        task(),  # type: ignore[arg-type]
        scratchpad.call_counter(),
        Call(
            func,
            input,
            retry_policy=retry_policy,
            cache_policy=cache_policy,
            callbacks=callbacks,
        ),
    ):
        if fut := next(
            (
                f
                for f, t in futures().items()  # type: ignore[union-attr]
                if t is not None and t == next_task.id
            ),
            None,
        ):
            # if the parent task was retried,
            # the next task might already be running
            pass
        elif next_task.writes:
            # if it already ran, return the result
            fut = concurrent.futures.Future()
            ret = next((v for c, v in next_task.writes if c == RETURN), MISSING)
            if ret is not MISSING:
                fut.set_result(ret)
            elif exc := next((v for c, v in next_task.writes if c == ERROR), None):
                fut.set_exception(
                    exc if isinstance(exc, BaseException) else Exception(exc)
                )
            else:
                fut.set_result(None)
        else:
            # schedule the next task
            fut = submit()(  # type: ignore[misc]
                run_with_retry,
                next_task,
                retry_policy,
                configurable={
                    CONFIG_KEY_CALL: partial(
                        _call,
                        weakref.ref(next_task),
                        futures=futures,
                        retry_policy=retry_policy,
                        callbacks=callbacks,
                        schedule_task=schedule_task,
                        submit=submit,
                    ),
                },
                __reraise_on_exit__=False,
                # starting a new task in the next tick ensures
                # updates from this tick are committed/streamed first
                __next_tick__=True,
            )
            # exceptions for call() tasks are raised into the parent task
            # so we should not re-raise at the end of the tick
            SKIP_RERAISE_SET.add(fut)
            futures()[fut] = next_task  # type: ignore[index]
    fut = cast(asyncio.Future | concurrent.futures.Future, fut)
    # return a chained future to ensure commit() callback is called
    # before the returned future is resolved, to ensure stream order etc
    return chain_future(fut, concurrent.futures.Future())


def _acall(
    task: weakref.ref[PregelExecutableTask],
    func: Callable[[Any], Awaitable[Any] | Any],
    input: Any,
    *,
    retry_policy: Sequence[RetryPolicy] | None = None,
    cache_policy: CachePolicy | None = None,
    callbacks: Callbacks = None,
    # injected dependencies
    futures: weakref.ref[FuturesDict],
    schedule_task: Callable[
        [PregelExecutableTask, int, Call | None],
        Awaitable[PregelExecutableTask | None],
    ],
    submit: weakref.ref[Submit],
    loop: asyncio.AbstractEventLoop,
    stream: bool = False,
) -> asyncio.Future[Any] | concurrent.futures.Future[Any]:
    # return a chained future to ensure commit() callback is called
    # before the returned future is resolved, to ensure stream order etc
    try:
        in_async = asyncio.current_task() is not None
    except RuntimeError:
        in_async = False
    # if in async context return an async future, otherwise return a sync future
    if in_async:
        fut: asyncio.Future[Any] | concurrent.futures.Future[Any] = asyncio.Future(
            loop=loop
        )
    else:
        fut = concurrent.futures.Future()
    # schedule the next task
    run_coroutine_threadsafe(
        _acall_impl(
            fut,
            task,
            func,
            input,
            retry_policy=retry_policy,
            cache_policy=cache_policy,
            callbacks=callbacks,
            futures=futures,
            schedule_task=schedule_task,
            submit=submit,
            loop=loop,
            stream=stream,
        ),
        loop,
        lazy=False,
    )
    return fut


async def _acall_impl(
    destination: asyncio.Future[Any] | concurrent.futures.Future[Any],
    task: weakref.ref[PregelExecutableTask],
    func: Callable[[Any], Awaitable[Any] | Any],
    input: Any,
    *,
    retry_policy: Sequence[RetryPolicy] | None = None,
    cache_policy: CachePolicy | None = None,
    callbacks: Callbacks = None,
    # injected dependencies
    futures: weakref.ref[FuturesDict[asyncio.Future, asyncio.Event]],
    schedule_task: Callable[
        [PregelExecutableTask, int, Call | None],
        Awaitable[PregelExecutableTask | None],
    ],
    submit: weakref.ref[Submit],
    loop: asyncio.AbstractEventLoop,
    stream: bool = False,
) -> None:
    try:
        fut: asyncio.Future | None = None
        # schedule PUSH tasks, collect futures
        scratchpad: PregelScratchpad = task().config[CONF][CONFIG_KEY_SCRATCHPAD]  # type: ignore[union-attr]
        # schedule the next task, if the callback returns one
        if next_task := await schedule_task(
            task(),  # type: ignore[arg-type]
            scratchpad.call_counter(),
            Call(
                func,
                input,
                retry_policy=retry_policy,
                cache_policy=cache_policy,
                callbacks=callbacks,
            ),
        ):
            if fut := next(
                (
                    f
                    for f, t in futures().items()  # type: ignore[union-attr]
                    if t is not None and t == next_task.id
                ),
                None,
            ):
                # if the parent task was retried,
                # the next task might already be running
                pass
            elif next_task.writes:
                # if it already ran, return the result
                fut = asyncio.Future(loop=loop)
                ret = next((v for c, v in next_task.writes if c == RETURN), MISSING)
                if ret is not MISSING:
                    fut.set_result(ret)
                elif exc := next((v for c, v in next_task.writes if c == ERROR), None):
                    fut.set_exception(
                        exc if isinstance(exc, BaseException) else Exception(exc)
                    )
                else:
                    fut.set_result(None)
                futures()[fut] = next_task  # type: ignore[index]
            else:
                # schedule the next task
                fut = cast(
                    asyncio.Future,
                    submit()(  # type: ignore[misc]
                        arun_with_retry,
                        next_task,
                        retry_policy,
                        stream=stream,
                        configurable={
                            CONFIG_KEY_CALL: partial(
                                _acall,
                                weakref.ref(next_task),
                                stream=stream,
                                futures=futures,
                                schedule_task=schedule_task,
                                submit=submit,
                                loop=loop,
                            ),
                        },
                        __name__=next_task.name,
                        __cancel_on_exit__=True,
                        __reraise_on_exit__=False,
                        # starting a new task in the next tick ensures
                        # updates from this tick are committed/streamed first
                        __next_tick__=True,
                    ),
                )
                # exceptions for call() tasks are raised into the parent task
                # so we should not re-raise at the end of the tick
                SKIP_RERAISE_SET.add(fut)
                futures()[fut] = next_task  # type: ignore[index]
        if fut is not None:
            chain_future(fut, destination)
        else:
            destination.set_exception(RuntimeError("Task not scheduled"))
    except Exception as exc:
        destination.set_exception(exc)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/_utils.py
```py
from __future__ import annotations

import ast
import inspect
import re
import textwrap
from collections.abc import Callable
from typing import Any

from langchain_core.runnables import Runnable, RunnableLambda, RunnableSequence
from langgraph.checkpoint.base import ChannelVersions
from typing_extensions import override

from langgraph._internal._runnable import RunnableCallable, RunnableSeq
from langgraph.pregel.protocol import PregelProtocol


def get_new_channel_versions(
    previous_versions: ChannelVersions, current_versions: ChannelVersions
) -> ChannelVersions:
    """Get subset of current_versions that are newer than previous_versions."""
    if previous_versions:
        version_type = type(next(iter(current_versions.values()), None))
        null_version = version_type()  # type: ignore[misc]
        new_versions = {
            k: v
            for k, v in current_versions.items()
            if v > previous_versions.get(k, null_version)  # type: ignore[operator]
        }
    else:
        new_versions = current_versions

    return new_versions


def find_subgraph_pregel(candidate: Runnable) -> PregelProtocol | None:
    from langgraph.pregel import Pregel

    candidates: list[Runnable] = [candidate]

    for c in candidates:
        if (
            isinstance(c, PregelProtocol)
            # subgraphs that disabled checkpointing are not considered
            and (not isinstance(c, Pregel) or c.checkpointer is not False)
        ):
            return c
        elif isinstance(c, RunnableSequence) or isinstance(c, RunnableSeq):
            candidates.extend(c.steps)
        elif isinstance(c, RunnableLambda):
            candidates.extend(c.deps)
        elif isinstance(c, RunnableCallable):
            if c.func is not None:
                candidates.extend(
                    nl.__self__ if hasattr(nl, "__self__") else nl
                    for nl in get_function_nonlocals(c.func)
                )
            elif c.afunc is not None:
                candidates.extend(
                    nl.__self__ if hasattr(nl, "__self__") else nl
                    for nl in get_function_nonlocals(c.afunc)
                )

    return None


def get_function_nonlocals(func: Callable) -> list[Any]:
    """Get the nonlocal variables accessed by a function.

    Args:
        func: The function to check.

    Returns:
        List[Any]: The nonlocal variables accessed by the function.
    """
    try:
        code = inspect.getsource(func)
        tree = ast.parse(textwrap.dedent(code))
        visitor = FunctionNonLocals()
        visitor.visit(tree)
        values: list[Any] = []
        closure = (
            inspect.getclosurevars(func.__wrapped__)
            if hasattr(func, "__wrapped__") and callable(func.__wrapped__)
            else inspect.getclosurevars(func)
        )
        candidates = {**closure.globals, **closure.nonlocals}
        for k, v in candidates.items():
            if k in visitor.nonlocals:
                values.append(v)
            for kk in visitor.nonlocals:
                if "." in kk and kk.startswith(k):
                    vv = v
                    for part in kk.split(".")[1:]:
                        if vv is None:
                            break
                        else:
                            try:
                                vv = getattr(vv, part)
                            except AttributeError:
                                break
                    else:
                        values.append(vv)
    except (SyntaxError, TypeError, OSError, SystemError):
        return []

    return values


class FunctionNonLocals(ast.NodeVisitor):
    """Get the nonlocal variables accessed of a function."""

    def __init__(self) -> None:
        self.nonlocals: set[str] = set()

    @override
    def visit_FunctionDef(self, node: ast.FunctionDef) -> Any:
        """Visit a function definition.

        Args:
            node: The node to visit.

        Returns:
            Any: The result of the visit.
        """
        visitor = NonLocals()
        visitor.visit(node)
        self.nonlocals.update(visitor.loads - visitor.stores)

    @override
    def visit_AsyncFunctionDef(self, node: ast.AsyncFunctionDef) -> Any:
        """Visit an async function definition.

        Args:
            node: The node to visit.

        Returns:
            Any: The result of the visit.
        """
        visitor = NonLocals()
        visitor.visit(node)
        self.nonlocals.update(visitor.loads - visitor.stores)

    @override
    def visit_Lambda(self, node: ast.Lambda) -> Any:
        """Visit a lambda function.

        Args:
            node: The node to visit.

        Returns:
            Any: The result of the visit.
        """
        visitor = NonLocals()
        visitor.visit(node)
        self.nonlocals.update(visitor.loads - visitor.stores)


class NonLocals(ast.NodeVisitor):
    """Get nonlocal variables accessed."""

    def __init__(self) -> None:
        self.loads: set[str] = set()
        self.stores: set[str] = set()

    @override
    def visit_Name(self, node: ast.Name) -> Any:
        """Visit a name node.

        Args:
            node: The node to visit.

        Returns:
            Any: The result of the visit.
        """
        if isinstance(node.ctx, ast.Load):
            self.loads.add(node.id)
        elif isinstance(node.ctx, ast.Store):
            self.stores.add(node.id)

    @override
    def visit_Attribute(self, node: ast.Attribute) -> Any:
        """Visit an attribute node.

        Args:
            node: The node to visit.

        Returns:
            Any: The result of the visit.
        """
        if isinstance(node.ctx, ast.Load):
            parent = node.value
            attr_expr = node.attr
            while isinstance(parent, ast.Attribute):
                attr_expr = parent.attr + "." + attr_expr
                parent = parent.value
            if isinstance(parent, ast.Name):
                self.loads.add(parent.id + "." + attr_expr)
                self.loads.discard(parent.id)
            elif isinstance(parent, ast.Call):
                if isinstance(parent.func, ast.Name):
                    self.loads.add(parent.func.id)
                else:
                    parent = parent.func
                    attr_expr = ""
                    while isinstance(parent, ast.Attribute):
                        if attr_expr:
                            attr_expr = parent.attr + "." + attr_expr
                        else:
                            attr_expr = parent.attr
                        parent = parent.value
                    if isinstance(parent, ast.Name):
                        self.loads.add(parent.id + "." + attr_expr)


def is_xxh3_128_hexdigest(value: str) -> bool:
    """Check if the given string matches the format of xxh3_128_hexdigest."""
    return bool(re.fullmatch(r"[0-9a-f]{32}", value))

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/_validate.py
```py
from __future__ import annotations

from collections.abc import Mapping, Sequence
from typing import Any

from langgraph._internal._constants import RESERVED
from langgraph.channels.base import BaseChannel
from langgraph.managed.base import ManagedValueMapping
from langgraph.pregel._read import PregelNode
from langgraph.types import All


def validate_graph(
    nodes: Mapping[str, PregelNode],
    channels: dict[str, BaseChannel],
    managed: ManagedValueMapping,
    input_channels: str | Sequence[str],
    output_channels: str | Sequence[str],
    stream_channels: str | Sequence[str] | None,
    interrupt_after_nodes: All | Sequence[str],
    interrupt_before_nodes: All | Sequence[str],
) -> None:
    for chan in channels:
        if chan in RESERVED:
            raise ValueError(f"Channel name '{chan}' is reserved")
    for name in managed:
        if name in RESERVED:
            raise ValueError(f"Managed name '{name}' is reserved")

    subscribed_channels = set[str]()
    for name, node in nodes.items():
        if name in RESERVED:
            raise ValueError(f"Node name '{name}' is reserved")
        if isinstance(node, PregelNode):
            subscribed_channels.update(node.triggers)
            if isinstance(node.channels, str):
                if node.channels not in channels:
                    raise ValueError(
                        f"Node {name} reads channel '{node.channels}' "
                        f"not in known channels: '{repr(sorted(channels))[:100]}'"
                    )
            else:
                for chan in node.channels:
                    if chan not in channels and chan not in managed:
                        raise ValueError(
                            f"Node {name} reads channel '{chan}' "
                            f"not in known channels: '{repr(sorted(channels))[:100]}'"
                        )
        else:
            raise TypeError(
                f"Invalid node type {type(node)}, expected PregelNode or NodeBuilder"
            )

    for chan in subscribed_channels:
        if chan not in channels:
            raise ValueError(
                f"Subscribed channel '{chan}' not "
                f"in known channels: '{repr(sorted(channels))[:100]}'"
            )

    if isinstance(input_channels, str):
        if input_channels not in channels:
            raise ValueError(
                f"Input channel '{input_channels}' not "
                f"in known channels: '{repr(sorted(channels))[:100]}'"
            )
        if input_channels not in subscribed_channels:
            raise ValueError(
                f"Input channel {input_channels} is not subscribed to by any node"
            )
    else:
        for chan in input_channels:
            if chan not in channels:
                raise ValueError(
                    f"Input channel '{chan}' not in '{repr(sorted(channels))[:100]}'"
                )
        if all(chan not in subscribed_channels for chan in input_channels):
            raise ValueError(
                f"None of the input channels {input_channels} "
                f"are subscribed to by any node"
            )

    all_output_channels = set[str]()
    if isinstance(output_channels, str):
        all_output_channels.add(output_channels)
    else:
        all_output_channels.update(output_channels)
    if isinstance(stream_channels, str):
        all_output_channels.add(stream_channels)
    elif stream_channels is not None:
        all_output_channels.update(stream_channels)

    for chan in all_output_channels:
        if chan not in channels:
            raise ValueError(
                f"Output channel '{chan}' not "
                f"in known channels: '{repr(sorted(channels))[:100]}'"
            )

    if interrupt_after_nodes != "*":
        for n in interrupt_after_nodes:
            if n not in nodes:
                raise ValueError(f"Node {n} not in nodes")
    if interrupt_before_nodes != "*":
        for n in interrupt_before_nodes:
            if n not in nodes:
                raise ValueError(f"Node {n} not in nodes")


def validate_keys(
    keys: str | Sequence[str] | None,
    channels: Mapping[str, Any],
) -> None:
    if isinstance(keys, str):
        if keys not in channels:
            raise ValueError(f"Key {keys} not in channels")
    elif keys is not None:
        for chan in keys:
            if chan not in channels:
                raise ValueError(f"Key {chan} not in channels")

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/_write.py
```py
from __future__ import annotations

from collections.abc import Callable, Sequence
from typing import (
    Any,
    NamedTuple,
    TypeVar,
    cast,
)

from langchain_core.runnables import Runnable, RunnableConfig

from langgraph._internal._constants import CONF, CONFIG_KEY_SEND, TASKS
from langgraph._internal._runnable import RunnableCallable
from langgraph._internal._typing import MISSING
from langgraph.errors import InvalidUpdateError
from langgraph.types import Send

TYPE_SEND = Callable[[Sequence[tuple[str, Any]]], None]
R = TypeVar("R", bound=Runnable)

SKIP_WRITE = object()
PASSTHROUGH = object()


class ChannelWriteEntry(NamedTuple):
    channel: str
    """Channel name to write to."""
    value: Any = PASSTHROUGH
    """Value to write, or PASSTHROUGH to use the input."""
    skip_none: bool = False
    """Whether to skip writing if the value is None."""
    mapper: Callable | None = None
    """Function to transform the value before writing."""


class ChannelWriteTupleEntry(NamedTuple):
    mapper: Callable[[Any], Sequence[tuple[str, Any]] | None]
    """Function to extract tuples from value."""
    value: Any = PASSTHROUGH
    """Value to write, or PASSTHROUGH to use the input."""
    static: Sequence[tuple[str, Any, str | None]] | None = None
    """Optional, declared writes for static analysis."""


class ChannelWrite(RunnableCallable):
    """Implements the logic for sending writes to CONFIG_KEY_SEND.
    Can be used as a runnable or as a static method to call imperatively."""

    writes: list[ChannelWriteEntry | ChannelWriteTupleEntry | Send]
    """Sequence of write entries or Send objects to write."""

    def __init__(
        self,
        writes: Sequence[ChannelWriteEntry | ChannelWriteTupleEntry | Send],
        *,
        tags: Sequence[str] | None = None,
    ):
        super().__init__(
            func=self._write,
            afunc=self._awrite,
            name=None,
            tags=tags,
            trace=False,
        )
        self.writes = cast(
            list[ChannelWriteEntry | ChannelWriteTupleEntry | Send], writes
        )

    def get_name(self, suffix: str | None = None, *, name: str | None = None) -> str:
        if not name:
            name = f"ChannelWrite<{','.join(w.channel if isinstance(w, ChannelWriteEntry) else '...' if isinstance(w, ChannelWriteTupleEntry) else w.node for w in self.writes)}>"
        return super().get_name(suffix, name=name)

    def _write(self, input: Any, config: RunnableConfig) -> None:
        writes = [
            ChannelWriteEntry(write.channel, input, write.skip_none, write.mapper)
            if isinstance(write, ChannelWriteEntry) and write.value is PASSTHROUGH
            else ChannelWriteTupleEntry(write.mapper, input)
            if isinstance(write, ChannelWriteTupleEntry) and write.value is PASSTHROUGH
            else write
            for write in self.writes
        ]
        self.do_write(
            config,
            writes,
        )
        return input

    async def _awrite(self, input: Any, config: RunnableConfig) -> None:
        writes = [
            ChannelWriteEntry(write.channel, input, write.skip_none, write.mapper)
            if isinstance(write, ChannelWriteEntry) and write.value is PASSTHROUGH
            else ChannelWriteTupleEntry(write.mapper, input)
            if isinstance(write, ChannelWriteTupleEntry) and write.value is PASSTHROUGH
            else write
            for write in self.writes
        ]
        self.do_write(
            config,
            writes,
        )
        return input

    @staticmethod
    def do_write(
        config: RunnableConfig,
        writes: Sequence[ChannelWriteEntry | ChannelWriteTupleEntry | Send],
        allow_passthrough: bool = True,
    ) -> None:
        # validate
        for w in writes:
            if isinstance(w, ChannelWriteEntry):
                if w.channel == TASKS:
                    raise InvalidUpdateError(
                        "Cannot write to the reserved channel TASKS"
                    )
                if w.value is PASSTHROUGH and not allow_passthrough:
                    raise InvalidUpdateError("PASSTHROUGH value must be replaced")
            if isinstance(w, ChannelWriteTupleEntry):
                if w.value is PASSTHROUGH and not allow_passthrough:
                    raise InvalidUpdateError("PASSTHROUGH value must be replaced")
        # if we want to persist writes found before hitting a ParentCommand
        # can move this to a finally block
        write: TYPE_SEND = config[CONF][CONFIG_KEY_SEND]
        write(_assemble_writes(writes))

    @staticmethod
    def is_writer(runnable: Runnable) -> bool:
        """Used by PregelNode to distinguish between writers and other runnables."""
        return (
            isinstance(runnable, ChannelWrite)
            or getattr(runnable, "_is_channel_writer", MISSING) is not MISSING
        )

    @staticmethod
    def get_static_writes(
        runnable: Runnable,
    ) -> Sequence[tuple[str, Any, str | None]] | None:
        """Used to get conditional writes a writer declares for static analysis."""
        if isinstance(runnable, ChannelWrite):
            return [
                w
                for entry in runnable.writes
                if isinstance(entry, ChannelWriteTupleEntry) and entry.static
                for w in entry.static
            ] or None
        elif writes := getattr(runnable, "_is_channel_writer", MISSING):
            if writes is not MISSING:
                writes = cast(
                    Sequence[tuple[ChannelWriteEntry | Send, str | None]],
                    writes,
                )
                entries = [e for e, _ in writes]
                labels = [la for _, la in writes]
                return [(*t, la) for t, la in zip(_assemble_writes(entries), labels)]

    @staticmethod
    def register_writer(
        runnable: R,
        static: Sequence[tuple[ChannelWriteEntry | Send, str | None]] | None = None,
    ) -> R:
        """Used to mark a runnable as a writer, so that it can be detected by is_writer.
        Instances of ChannelWrite are automatically marked as writers.
        Optionally, a list of declared writes can be passed for static analysis."""
        # using object.__setattr__ to work around objects that override __setattr__
        # eg. pydantic models and dataclasses
        object.__setattr__(runnable, "_is_channel_writer", static)
        return runnable


def _assemble_writes(
    writes: Sequence[ChannelWriteEntry | ChannelWriteTupleEntry | Send],
) -> list[tuple[str, Any]]:
    """Assembles the writes into a list of tuples."""
    tuples: list[tuple[str, Any]] = []
    for w in writes:
        if isinstance(w, Send):
            tuples.append((TASKS, w))
        elif isinstance(w, ChannelWriteTupleEntry):
            if ww := w.mapper(w.value):
                tuples.extend(ww)
        elif isinstance(w, ChannelWriteEntry):
            value = w.mapper(w.value) if w.mapper is not None else w.value
            if value is SKIP_WRITE:
                continue
            if w.skip_none and value is None:
                continue
            tuples.append((w.channel, value))
        else:
            raise ValueError(f"Invalid write entry: {w}")
    return tuples

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/debug.py
```py
from __future__ import annotations

from collections.abc import Iterable, Iterator, Mapping, Sequence
from dataclasses import asdict
from typing import Any
from uuid import UUID

from langchain_core.runnables import RunnableConfig
from langgraph.checkpoint.base import CheckpointMetadata, PendingWrite
from typing_extensions import TypedDict

from langgraph._internal._config import patch_checkpoint_map
from langgraph._internal._constants import (
    CONF,
    CONFIG_KEY_CHECKPOINT_NS,
    ERROR,
    INTERRUPT,
    NS_END,
    NS_SEP,
    RETURN,
)
from langgraph._internal._typing import MISSING
from langgraph.channels.base import BaseChannel
from langgraph.constants import TAG_HIDDEN
from langgraph.pregel._io import read_channels
from langgraph.types import PregelExecutableTask, PregelTask, StateSnapshot

__all__ = ("TaskPayload", "TaskResultPayload", "CheckpointTask", "CheckpointPayload")


class TaskPayload(TypedDict):
    id: str
    name: str
    input: Any
    triggers: list[str]


class TaskResultPayload(TypedDict):
    id: str
    name: str
    error: str | None
    interrupts: list[dict]
    result: dict[str, Any]


class CheckpointTask(TypedDict):
    id: str
    name: str
    error: str | None
    interrupts: list[dict]
    state: StateSnapshot | RunnableConfig | None


class CheckpointPayload(TypedDict):
    config: RunnableConfig | None
    metadata: CheckpointMetadata
    values: dict[str, Any]
    next: list[str]
    parent_config: RunnableConfig | None
    tasks: list[CheckpointTask]


TASK_NAMESPACE = UUID("6ba7b831-9dad-11d1-80b4-00c04fd430c8")


def map_debug_tasks(tasks: Iterable[PregelExecutableTask]) -> Iterator[TaskPayload]:
    """Produce "task" events for stream_mode=debug."""
    for task in tasks:
        if task.config is not None and TAG_HIDDEN in task.config.get("tags", []):
            continue

        yield {
            "id": task.id,
            "name": task.name,
            "input": task.input,
            "triggers": task.triggers,
        }


def is_multiple_channel_write(value: Any) -> bool:
    """Return True if the payload already wraps multiple writes from the same channel."""
    return (
        isinstance(value, dict)
        and "$writes" in value
        and isinstance(value["$writes"], list)
    )


def map_task_result_writes(writes: Sequence[tuple[str, Any]]) -> dict[str, Any]:
    """Folds task writes into a result dict and aggregates multiple writes to the same channel.

    If the channel contains a single write, we record the write in the result dict as `{channel: write}`
    If the channel contains multiple writes, we record the writes in the result dict as `{channel: {'$writes': [write1, write2, ...]}}`"""

    result: dict[str, Any] = {}
    for channel, value in writes:
        existing = result.get(channel)

        if existing is not None:
            channel_writes = (
                existing["$writes"]
                if is_multiple_channel_write(existing)
                else [existing]
            )
            channel_writes.append(value)
            result[channel] = {"$writes": channel_writes}
        else:
            result[channel] = value
    return result


def map_debug_task_results(
    task_tup: tuple[PregelExecutableTask, Sequence[tuple[str, Any]]],
    stream_keys: str | Sequence[str],
) -> Iterator[TaskResultPayload]:
    """Produce "task_result" events for stream_mode=debug."""
    stream_channels_list = (
        [stream_keys] if isinstance(stream_keys, str) else stream_keys
    )
    task, writes = task_tup
    yield {
        "id": task.id,
        "name": task.name,
        "error": next((w[1] for w in writes if w[0] == ERROR), None),
        "result": map_task_result_writes(
            [w for w in writes if w[0] in stream_channels_list or w[0] == RETURN]
        ),
        "interrupts": [
            asdict(v)
            for w in writes
            if w[0] == INTERRUPT
            for v in (w[1] if isinstance(w[1], Sequence) else [w[1]])
        ],
    }


def rm_pregel_keys(config: RunnableConfig | None) -> RunnableConfig | None:
    """Remove pregel-specific keys from the config."""
    if config is None:
        return config
    return {
        "configurable": {
            k: v
            for k, v in config.get("configurable", {}).items()
            if not k.startswith("__pregel_")
        }
    }


def map_debug_checkpoint(
    config: RunnableConfig,
    channels: Mapping[str, BaseChannel],
    stream_channels: str | Sequence[str],
    metadata: CheckpointMetadata,
    tasks: Iterable[PregelExecutableTask],
    pending_writes: list[PendingWrite],
    parent_config: RunnableConfig | None,
    output_keys: str | Sequence[str],
) -> Iterator[CheckpointPayload]:
    """Produce "checkpoint" events for stream_mode=debug."""

    parent_ns = config[CONF].get(CONFIG_KEY_CHECKPOINT_NS, "")
    task_states: dict[str, RunnableConfig | StateSnapshot] = {}

    for task in tasks:
        if not task.subgraphs:
            continue

        # assemble checkpoint_ns for this task
        task_ns = f"{task.name}{NS_END}{task.id}"
        if parent_ns:
            task_ns = f"{parent_ns}{NS_SEP}{task_ns}"

        # set config as signal that subgraph checkpoints exist
        task_states[task.id] = {
            CONF: {
                "thread_id": config[CONF]["thread_id"],
                CONFIG_KEY_CHECKPOINT_NS: task_ns,
            }
        }

    yield {
        "config": rm_pregel_keys(patch_checkpoint_map(config, metadata)),
        "parent_config": rm_pregel_keys(patch_checkpoint_map(parent_config, metadata)),
        "values": read_channels(channels, stream_channels),
        "metadata": metadata,
        "next": [t.name for t in tasks],
        "tasks": [
            {
                "id": t.id,
                "name": t.name,
                "error": t.error,
                "state": t.state,
            }
            if t.error
            else {
                "id": t.id,
                "name": t.name,
                "result": t.result,
                "interrupts": tuple(asdict(i) for i in t.interrupts),
                "state": t.state,
            }
            if t.result
            else {
                "id": t.id,
                "name": t.name,
                "interrupts": tuple(asdict(i) for i in t.interrupts),
                "state": t.state,
            }
            for t in tasks_w_writes(tasks, pending_writes, task_states, output_keys)
        ],
    }


def tasks_w_writes(
    tasks: Iterable[PregelTask | PregelExecutableTask],
    pending_writes: list[PendingWrite] | None,
    states: dict[str, RunnableConfig | StateSnapshot] | None,
    output_keys: str | Sequence[str],
) -> tuple[PregelTask, ...]:
    """Apply writes / subgraph states to tasks to be returned in a StateSnapshot."""
    pending_writes = pending_writes or []
    out: list[PregelTask] = []
    for task in tasks:
        rtn = next(
            (
                val
                for tid, chan, val in pending_writes
                if tid == task.id and chan == RETURN
            ),
            MISSING,
        )
        task_error = next(
            (exc for tid, n, exc in pending_writes if tid == task.id and n == ERROR),
            None,
        )
        task_interrupts = tuple(
            v
            for tid, n, vv in pending_writes
            if tid == task.id and n == INTERRUPT
            for v in (vv if isinstance(vv, Sequence) else [vv])
        )

        task_writes = [
            (chan, val)
            for tid, chan, val in pending_writes
            if tid == task.id and chan not in (ERROR, INTERRUPT, RETURN)
        ]

        if rtn is not MISSING:
            task_result = rtn
        elif isinstance(output_keys, str):
            # unwrap single channel writes to just the write value
            filtered_writes = [
                (chan, val) for chan, val in task_writes if chan == output_keys
            ]
            mapped_writes = map_task_result_writes(filtered_writes)
            task_result = mapped_writes.get(str(output_keys)) if mapped_writes else None
        else:
            if isinstance(output_keys, str):
                output_keys = [output_keys]
            # map task result writes to the desired output channels
            # repeateed writes to the same channel are aggregated into: {'$writes': [write1, write2, ...]}
            filtered_writes = [
                (chan, val) for chan, val in task_writes if chan in output_keys
            ]
            mapped_writes = map_task_result_writes(filtered_writes)
            task_result = mapped_writes if filtered_writes else {}

        has_writes = rtn is not MISSING or any(
            w[0] == task.id and w[1] not in (ERROR, INTERRUPT) for w in pending_writes
        )

        out.append(
            PregelTask(
                task.id,
                task.name,
                task.path,
                task_error,
                task_interrupts,
                states.get(task.id) if states else None,
                task_result if has_writes else None,
            )
        )
    return tuple(out)


COLOR_MAPPING = {
    "black": "0;30",
    "red": "0;31",
    "green": "0;32",
    "yellow": "0;33",
    "blue": "0;34",
    "magenta": "0;35",
    "cyan": "0;36",
    "white": "0;37",
    "gray": "1;30",
}


def get_colored_text(text: str, color: str) -> str:
    """Get colored text."""
    return f"\033[1;3{COLOR_MAPPING[color]}m{text}\033[0m"


def get_bolded_text(text: str) -> str:
    """Get bolded text."""
    return f"\033[1m{text}\033[0m"

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/pregel/main.py
```py
from __future__ import annotations

import asyncio
import concurrent
import concurrent.futures
import contextlib
import queue
import warnings
import weakref
from collections import defaultdict, deque
from collections.abc import (
    AsyncIterator,
    Awaitable,
    Callable,
    Iterator,
    Mapping,
    Sequence,
)
from dataclasses import is_dataclass
from functools import partial
from inspect import isclass
from typing import (
    Any,
    Generic,
    cast,
    get_type_hints,
)
from uuid import UUID, uuid5

from langchain_core.globals import get_debug
from langchain_core.runnables import (
    RunnableSequence,
)
from langchain_core.runnables.base import Input, Output
from langchain_core.runnables.config import (
    RunnableConfig,
    get_async_callback_manager_for_config,
    get_callback_manager_for_config,
)
from langchain_core.runnables.graph import Graph
from langgraph.cache.base import BaseCache
from langgraph.checkpoint.base import (
    BaseCheckpointSaver,
    Checkpoint,
    CheckpointTuple,
)
from langgraph.store.base import BaseStore
from pydantic import BaseModel, TypeAdapter
from typing_extensions import Self, Unpack, deprecated, is_typeddict

from langgraph._internal._config import (
    ensure_config,
    merge_configs,
    patch_checkpoint_map,
    patch_config,
    patch_configurable,
    recast_checkpoint_ns,
)
from langgraph._internal._constants import (
    CACHE_NS_WRITES,
    CONF,
    CONFIG_KEY_CACHE,
    CONFIG_KEY_CHECKPOINT_ID,
    CONFIG_KEY_CHECKPOINT_NS,
    CONFIG_KEY_CHECKPOINTER,
    CONFIG_KEY_DURABILITY,
    CONFIG_KEY_NODE_FINISHED,
    CONFIG_KEY_READ,
    CONFIG_KEY_RUNNER_SUBMIT,
    CONFIG_KEY_RUNTIME,
    CONFIG_KEY_SEND,
    CONFIG_KEY_STREAM,
    CONFIG_KEY_TASK_ID,
    CONFIG_KEY_THREAD_ID,
    ERROR,
    INPUT,
    INTERRUPT,
    NS_END,
    NS_SEP,
    NULL_TASK_ID,
    PUSH,
    TASKS,
)
from langgraph._internal._pydantic import create_model
from langgraph._internal._queue import (  # type: ignore[attr-defined]
    AsyncQueue,
    SyncQueue,
)
from langgraph._internal._runnable import (
    Runnable,
    RunnableLike,
    RunnableSeq,
    coerce_to_runnable,
)
from langgraph._internal._typing import MISSING, DeprecatedKwargs
from langgraph.channels.base import BaseChannel
from langgraph.channels.topic import Topic
from langgraph.config import get_config
from langgraph.constants import END
from langgraph.errors import (
    ErrorCode,
    GraphRecursionError,
    InvalidUpdateError,
    create_error_message,
)
from langgraph.managed.base import ManagedValueSpec
from langgraph.pregel._algo import (
    PregelTaskWrites,
    _scratchpad,
    apply_writes,
    local_read,
    prepare_next_tasks,
)
from langgraph.pregel._call import identifier
from langgraph.pregel._checkpoint import (
    channels_from_checkpoint,
    copy_checkpoint,
    create_checkpoint,
    empty_checkpoint,
)
from langgraph.pregel._draw import draw_graph
from langgraph.pregel._io import map_input, read_channels
from langgraph.pregel._loop import AsyncPregelLoop, SyncPregelLoop
from langgraph.pregel._messages import StreamMessagesHandler
from langgraph.pregel._read import DEFAULT_BOUND, PregelNode
from langgraph.pregel._retry import RetryPolicy
from langgraph.pregel._runner import PregelRunner
from langgraph.pregel._utils import get_new_channel_versions
from langgraph.pregel._validate import validate_graph, validate_keys
from langgraph.pregel._write import ChannelWrite, ChannelWriteEntry
from langgraph.pregel.debug import get_bolded_text, get_colored_text, tasks_w_writes
from langgraph.pregel.protocol import PregelProtocol, StreamChunk, StreamProtocol
from langgraph.runtime import DEFAULT_RUNTIME, Runtime
from langgraph.types import (
    All,
    CachePolicy,
    Checkpointer,
    Command,
    Durability,
    Interrupt,
    Send,
    StateSnapshot,
    StateUpdate,
    StreamMode,
)
from langgraph.typing import ContextT, InputT, OutputT, StateT
from langgraph.warnings import LangGraphDeprecatedSinceV10

try:
    from langchain_core.tracers._streaming import _StreamingCallbackHandler
except ImportError:
    _StreamingCallbackHandler = None  # type: ignore

__all__ = ("NodeBuilder", "Pregel")

_WriteValue = Callable[[Input], Output] | Any


class NodeBuilder:
    __slots__ = (
        "_channels",
        "_triggers",
        "_tags",
        "_metadata",
        "_writes",
        "_bound",
        "_retry_policy",
        "_cache_policy",
    )

    _channels: str | list[str]
    _triggers: list[str]
    _tags: list[str]
    _metadata: dict[str, Any]
    _writes: list[ChannelWriteEntry]
    _bound: Runnable
    _retry_policy: list[RetryPolicy]
    _cache_policy: CachePolicy | None

    def __init__(
        self,
    ) -> None:
        self._channels = []
        self._triggers = []
        self._tags = []
        self._metadata = {}
        self._writes = []
        self._bound = DEFAULT_BOUND
        self._retry_policy = []
        self._cache_policy = None

    def subscribe_only(
        self,
        channel: str,
    ) -> Self:
        """Subscribe to a single channel."""
        if not self._channels:
            self._channels = channel
        else:
            raise ValueError(
                "Cannot subscribe to single channels when other channels are already subscribed to"
            )

        self._triggers.append(channel)

        return self

    def subscribe_to(
        self,
        *channels: str,
        read: bool = True,
    ) -> Self:
        """Add channels to subscribe to. Node will be invoked when any of these
        channels are updated, with a dict of the channel values as input.

        Args:
            channels: Channel name(s) to subscribe to
            read: If `True`, the channels will be included in the input to the node.
                Otherwise, they will trigger the node without being sent in input.

        Returns:
            Self for chaining
        """
        if isinstance(self._channels, str):
            raise ValueError(
                "Cannot subscribe to channels when subscribed to a single channel"
            )
        if read:
            if not self._channels:
                self._channels = list(channels)
            else:
                self._channels.extend(channels)

        if isinstance(channels, str):
            self._triggers.append(channels)
        else:
            self._triggers.extend(channels)

        return self

    def read_from(
        self,
        *channels: str,
    ) -> Self:
        """Adds the specified channels to read from, without subscribing to them."""
        assert isinstance(self._channels, list), (
            "Cannot read additional channels when subscribed to single channels"
        )
        self._channels.extend(channels)
        return self

    def do(
        self,
        node: RunnableLike,
    ) -> Self:
        """Adds the specified node."""
        if self._bound is not DEFAULT_BOUND:
            self._bound = RunnableSeq(
                self._bound, coerce_to_runnable(node, name=None, trace=True)
            )
        else:
            self._bound = coerce_to_runnable(node, name=None, trace=True)
        return self

    def write_to(
        self,
        *channels: str | ChannelWriteEntry,
        **kwargs: _WriteValue,
    ) -> Self:
        """Add channel writes.

        Args:
            *channels: Channel names to write to
            **kwargs: Channel name and value mappings

        Returns:
            Self for chaining
        """
        self._writes.extend(
            ChannelWriteEntry(c) if isinstance(c, str) else c for c in channels
        )
        self._writes.extend(
            ChannelWriteEntry(k, mapper=v)
            if callable(v)
            else ChannelWriteEntry(k, value=v)
            for k, v in kwargs.items()
        )

        return self

    def meta(self, *tags: str, **metadata: Any) -> Self:
        """Add tags or metadata to the node."""
        self._tags.extend(tags)
        self._metadata.update(metadata)
        return self

    def add_retry_policies(self, *policies: RetryPolicy) -> Self:
        """Adds retry policies to the node."""
        self._retry_policy.extend(policies)
        return self

    def add_cache_policy(self, policy: CachePolicy) -> Self:
        """Adds cache policies to the node."""
        self._cache_policy = policy
        return self

    def build(self) -> PregelNode:
        """Builds the node."""
        return PregelNode(
            channels=self._channels,
            triggers=self._triggers,
            tags=self._tags,
            metadata=self._metadata,
            writers=[ChannelWrite(self._writes)],
            bound=self._bound,
            retry_policy=self._retry_policy,
            cache_policy=self._cache_policy,
        )


class Pregel(
    PregelProtocol[StateT, ContextT, InputT, OutputT],
    Generic[StateT, ContextT, InputT, OutputT],
):
    """Pregel manages the runtime behavior for LangGraph applications.

    ## Overview

    Pregel combines [**actors**](https://en.wikipedia.org/wiki/Actor_model)
    and **channels** into a single application.
    **Actors** read data from channels and write data to channels.
    Pregel organizes the execution of the application into multiple steps,
    following the **Pregel Algorithm**/**Bulk Synchronous Parallel** model.

    Each step consists of three phases:

    - **Plan**: Determine which **actors** to execute in this step. For example,
        in the first step, select the **actors** that subscribe to the special
        **input** channels; in subsequent steps,
        select the **actors** that subscribe to channels updated in the previous step.
    - **Execution**: Execute all selected **actors** in parallel,
        until all complete, or one fails, or a timeout is reached. During this
        phase, channel updates are invisible to actors until the next step.
    - **Update**: Update the channels with the values written by the **actors**
        in this step.

    Repeat until no **actors** are selected for execution, or a maximum number of
    steps is reached.

    ## Actors

    An **actor** is a `PregelNode`.
    It subscribes to channels, reads data from them, and writes data to them.
    It can be thought of as an **actor** in the Pregel algorithm.
    `PregelNodes` implement LangChain's
    Runnable interface.

    ## Channels

    Channels are used to communicate between actors (`PregelNodes`).
    Each channel has a value type, an update type, and an update function – which
    takes a sequence of updates and
    modifies the stored value. Channels can be used to send data from one chain to
    another, or to send data from a chain to itself in a future step. LangGraph
    provides a number of built-in channels:

    ### Basic channels: LastValue and Topic

    - `LastValue`: The default channel, stores the last value sent to the channel,
       useful for input and output values, or for sending data from one step to the next
    - `Topic`: A configurable PubSub Topic, useful for sending multiple values
       between *actors*, or for accumulating output. Can be configured to deduplicate
       values, and/or to accumulate values over the course of multiple steps.

    ### Advanced channels: Context and BinaryOperatorAggregate

    - `Context`: exposes the value of a context manager, managing its lifecycle.
      Useful for accessing external resources that require setup and/or teardown. eg.
      `client = Context(httpx.Client)`
    - `BinaryOperatorAggregate`: stores a persistent value, updated by applying
       a binary operator to the current value and each update
       sent to the channel, useful for computing aggregates over multiple steps. eg.
      `total = BinaryOperatorAggregate(int, operator.add)`

    ## Examples

    Most users will interact with Pregel via a
    [StateGraph (Graph API)][langgraph.graph.StateGraph] or via an
    [entrypoint (Functional API)][langgraph.func.entrypoint].

    However, for **advanced** use cases, Pregel can be used directly. If you're
    not sure whether you need to use Pregel directly, then the answer is probably no
    - you should use the Graph API or Functional API instead. These are higher-level
    interfaces that will compile down to Pregel under the hood.

    Here are some examples to give you a sense of how it works:

    Example: Single node application
        ```python
        from langgraph.channels import EphemeralValue
        from langgraph.pregel import Pregel, NodeBuilder

        node1 = (
            NodeBuilder().subscribe_only("a")
            .do(lambda x: x + x)
            .write_to("b")
        )

        app = Pregel(
            nodes={"node1": node1},
            channels={
                "a": EphemeralValue(str),
                "b": EphemeralValue(str),
            },
            input_channels=["a"],
            output_channels=["b"],
        )

        app.invoke({"a": "foo"})
        ```

        ```con
        {'b': 'foofoo'}
        ```

    Example: Using multiple nodes and multiple output channels
        ```python
        from langgraph.channels import LastValue, EphemeralValue
        from langgraph.pregel import Pregel, NodeBuilder

        node1 = (
            NodeBuilder().subscribe_only("a")
            .do(lambda x: x + x)
            .write_to("b")
        )

        node2 = (
            NodeBuilder().subscribe_to("b")
            .do(lambda x: x["b"] + x["b"])
            .write_to("c")
        )


        app = Pregel(
            nodes={"node1": node1, "node2": node2},
            channels={
                "a": EphemeralValue(str),
                "b": LastValue(str),
                "c": EphemeralValue(str),
            },
            input_channels=["a"],
            output_channels=["b", "c"],
        )

        app.invoke({"a": "foo"})
        ```

        ```con
        {'b': 'foofoo', 'c': 'foofoofoofoo'}
        ```

    Example: Using a Topic channel
        ```python
        from langgraph.channels import LastValue, EphemeralValue, Topic
        from langgraph.pregel import Pregel, NodeBuilder

        node1 = (
            NodeBuilder().subscribe_only("a")
            .do(lambda x: x + x)
            .write_to("b", "c")
        )

        node2 = (
            NodeBuilder().subscribe_only("b")
            .do(lambda x: x + x)
            .write_to("c")
        )


        app = Pregel(
            nodes={"node1": node1, "node2": node2},
            channels={
                "a": EphemeralValue(str),
                "b": EphemeralValue(str),
                "c": Topic(str, accumulate=True),
            },
            input_channels=["a"],
            output_channels=["c"],
        )

        app.invoke({"a": "foo"})
        ```

        ```pycon
        {"c": ["foofoo", "foofoofoofoo"]}
        ```

    Example: Using a BinaryOperatorAggregate channel
        ```python
        from langgraph.channels import EphemeralValue, BinaryOperatorAggregate
        from langgraph.pregel import Pregel, NodeBuilder


        node1 = (
            NodeBuilder().subscribe_only("a")
            .do(lambda x: x + x)
            .write_to("b", "c")
        )

        node2 = (
            NodeBuilder().subscribe_only("b")
            .do(lambda x: x + x)
            .write_to("c")
        )


        def reducer(current, update):
            if current:
                return current + " | " + update
            else:
                return update


        app = Pregel(
            nodes={"node1": node1, "node2": node2},
            channels={
                "a": EphemeralValue(str),
                "b": EphemeralValue(str),
                "c": BinaryOperatorAggregate(str, operator=reducer),
            },
            input_channels=["a"],
            output_channels=["c"],
        )

        app.invoke({"a": "foo"})
        ```

        ```con
        {'c': 'foofoo | foofoofoofoo'}
        ```

    Example: Introducing a cycle
        This example demonstrates how to introduce a cycle in the graph, by having
        a chain write to a channel it subscribes to. Execution will continue
        until a None value is written to the channel.

        ```python
        from langgraph.channels import EphemeralValue
        from langgraph.pregel import Pregel, NodeBuilder, ChannelWriteEntry

        example_node = (
            NodeBuilder()
            .subscribe_only("value")
            .do(lambda x: x + x if len(x) < 10 else None)
            .write_to(ChannelWriteEntry(channel="value", skip_none=True))
        )

        app = Pregel(
            nodes={"example_node": example_node},
            channels={
                "value": EphemeralValue(str),
            },
            input_channels=["value"],
            output_channels=["value"],
        )

        app.invoke({"value": "a"})
        ```

        ```con
        {'value': 'aaaaaaaaaaaaaaaa'}
        ```
    """

    nodes: dict[str, PregelNode]

    channels: dict[str, BaseChannel | ManagedValueSpec]

    stream_mode: StreamMode = "values"
    """Mode to stream output, defaults to 'values'."""

    stream_eager: bool = False
    """Whether to force emitting stream events eagerly, automatically turned on
    for stream_mode "messages" and "custom"."""

    output_channels: str | Sequence[str]

    stream_channels: str | Sequence[str] | None = None
    """Channels to stream, defaults to all channels not in reserved channels"""

    interrupt_after_nodes: All | Sequence[str]

    interrupt_before_nodes: All | Sequence[str]

    input_channels: str | Sequence[str]

    step_timeout: float | None = None
    """Maximum time to wait for a step to complete, in seconds."""

    debug: bool
    """Whether to print debug information during execution."""

    checkpointer: Checkpointer = None
    """`Checkpointer` used to save and load graph state."""

    store: BaseStore | None = None
    """Memory store to use for SharedValues."""

    cache: BaseCache | None = None
    """Cache to use for storing node results."""

    retry_policy: Sequence[RetryPolicy] = ()
    """Retry policies to use when running tasks. Empty set disables retries."""

    cache_policy: CachePolicy | None = None
    """Cache policy to use for all nodes. Can be overridden by individual nodes."""

    context_schema: type[ContextT] | None = None
    """Specifies the schema for the context object that will be passed to the workflow."""

    config: RunnableConfig | None = None

    name: str = "LangGraph"

    trigger_to_nodes: Mapping[str, Sequence[str]]

    def __init__(
        self,
        *,
        nodes: dict[str, PregelNode | NodeBuilder],
        channels: dict[str, BaseChannel | ManagedValueSpec] | None,
        auto_validate: bool = True,
        stream_mode: StreamMode = "values",
        stream_eager: bool = False,
        output_channels: str | Sequence[str],
        stream_channels: str | Sequence[str] | None = None,
        interrupt_after_nodes: All | Sequence[str] = (),
        interrupt_before_nodes: All | Sequence[str] = (),
        input_channels: str | Sequence[str],
        step_timeout: float | None = None,
        debug: bool | None = None,
        checkpointer: BaseCheckpointSaver | None = None,
        store: BaseStore | None = None,
        cache: BaseCache | None = None,
        retry_policy: RetryPolicy | Sequence[RetryPolicy] = (),
        cache_policy: CachePolicy | None = None,
        context_schema: type[ContextT] | None = None,
        config: RunnableConfig | None = None,
        trigger_to_nodes: Mapping[str, Sequence[str]] | None = None,
        name: str = "LangGraph",
        **deprecated_kwargs: Unpack[DeprecatedKwargs],
    ) -> None:
        if (
            config_type := deprecated_kwargs.get("config_type", MISSING)
        ) is not MISSING:
            warnings.warn(
                "`config_type` is deprecated and will be removed. Please use `context_schema` instead.",
                category=LangGraphDeprecatedSinceV10,
                stacklevel=2,
            )

            if context_schema is None:
                context_schema = cast(type[ContextT], config_type)

        self.nodes = {
            k: v.build() if isinstance(v, NodeBuilder) else v for k, v in nodes.items()
        }
        self.channels = channels or {}
        if TASKS in self.channels and not isinstance(self.channels[TASKS], Topic):
            raise ValueError(
                f"Channel '{TASKS}' is reserved and cannot be used in the graph."
            )
        else:
            self.channels[TASKS] = Topic(Send, accumulate=False)
        self.stream_mode = stream_mode
        self.stream_eager = stream_eager
        self.output_channels = output_channels
        self.stream_channels = stream_channels
        self.interrupt_after_nodes = interrupt_after_nodes
        self.interrupt_before_nodes = interrupt_before_nodes
        self.input_channels = input_channels
        self.step_timeout = step_timeout
        self.debug = debug if debug is not None else get_debug()
        self.checkpointer = checkpointer
        self.store = store
        self.cache = cache
        self.retry_policy = (
            (retry_policy,) if isinstance(retry_policy, RetryPolicy) else retry_policy
        )
        self.cache_policy = cache_policy
        self.context_schema = context_schema
        self.config = config
        self.trigger_to_nodes = trigger_to_nodes or {}
        self.name = name
        if auto_validate:
            self.validate()

    def get_graph(
        self, config: RunnableConfig | None = None, *, xray: int | bool = False
    ) -> Graph:
        """Return a drawable representation of the computation graph."""
        # gather subgraphs
        if xray:
            subgraphs = {
                k: v.get_graph(
                    config,
                    xray=xray if isinstance(xray, bool) or xray <= 0 else xray - 1,
                )
                for k, v in self.get_subgraphs()
            }
        else:
            subgraphs = {}

        return draw_graph(
            merge_configs(self.config, config),
            nodes=self.nodes,
            specs=self.channels,
            input_channels=self.input_channels,
            interrupt_after_nodes=self.interrupt_after_nodes,
            interrupt_before_nodes=self.interrupt_before_nodes,
            trigger_to_nodes=self.trigger_to_nodes,
            checkpointer=self.checkpointer,
            subgraphs=subgraphs,
        )

    async def aget_graph(
        self, config: RunnableConfig | None = None, *, xray: int | bool = False
    ) -> Graph:
        """Return a drawable representation of the computation graph."""

        # gather subgraphs
        if xray:
            subpregels: dict[str, PregelProtocol] = {
                k: v async for k, v in self.aget_subgraphs()
            }
            subgraphs = {
                k: v
                for k, v in zip(
                    subpregels,
                    await asyncio.gather(
                        *(
                            p.aget_graph(
                                config,
                                xray=xray
                                if isinstance(xray, bool) or xray <= 0
                                else xray - 1,
                            )
                            for p in subpregels.values()
                        )
                    ),
                )
            }
        else:
            subgraphs = {}

        return draw_graph(
            merge_configs(self.config, config),
            nodes=self.nodes,
            specs=self.channels,
            input_channels=self.input_channels,
            interrupt_after_nodes=self.interrupt_after_nodes,
            interrupt_before_nodes=self.interrupt_before_nodes,
            trigger_to_nodes=self.trigger_to_nodes,
            checkpointer=self.checkpointer,
            subgraphs=subgraphs,
        )

    def _repr_mimebundle_(self, **kwargs: Any) -> dict[str, Any]:
        """Mime bundle used by Jupyter to display the graph"""
        return {
            "text/plain": repr(self),
            "image/png": self.get_graph().draw_mermaid_png(),
        }

    def copy(self, update: dict[str, Any] | None = None) -> Self:
        attrs = {k: v for k, v in self.__dict__.items() if k != "__orig_class__"}
        attrs.update(update or {})
        return self.__class__(**attrs)

    def with_config(self, config: RunnableConfig | None = None, **kwargs: Any) -> Self:
        """Create a copy of the Pregel object with an updated config."""
        return self.copy(
            {"config": merge_configs(self.config, config, cast(RunnableConfig, kwargs))}
        )

    def validate(self) -> Self:
        validate_graph(
            self.nodes,
            {k: v for k, v in self.channels.items() if isinstance(v, BaseChannel)},
            {k: v for k, v in self.channels.items() if not isinstance(v, BaseChannel)},
            self.input_channels,
            self.output_channels,
            self.stream_channels,
            self.interrupt_after_nodes,
            self.interrupt_before_nodes,
        )
        self.trigger_to_nodes = _trigger_to_nodes(self.nodes)
        return self

    @deprecated(
        "`config_schema` is deprecated. Use `get_context_jsonschema` for the relevant schema instead.",
        category=None,
    )
    def config_schema(self, *, include: Sequence[str] | None = None) -> type[BaseModel]:
        warnings.warn(
            "`config_schema` is deprecated. Use `get_context_jsonschema` for the relevant schema instead.",
            category=LangGraphDeprecatedSinceV10,
            stacklevel=2,
        )

        include = include or []
        fields = {
            **(
                {"configurable": (self.context_schema, None)}
                if self.context_schema
                else {}
            ),
            **{
                field_name: (field_type, None)
                for field_name, field_type in get_type_hints(RunnableConfig).items()
                if field_name in [i for i in include if i != "configurable"]
            },
        }
        return create_model(self.get_name("Config"), field_definitions=fields)

    @deprecated(
        "`get_config_jsonschema` is deprecated. Use `get_context_jsonschema` instead.",
        category=None,
    )
    def get_config_jsonschema(
        self, *, include: Sequence[str] | None = None
    ) -> dict[str, Any]:
        warnings.warn(
            "`get_config_jsonschema` is deprecated. Use `get_context_jsonschema` instead.",
            category=LangGraphDeprecatedSinceV10,
            stacklevel=2,
        )

        with warnings.catch_warnings():
            warnings.filterwarnings("ignore", category=LangGraphDeprecatedSinceV10)
            schema = self.config_schema(include=include)
        return schema.model_json_schema()

    def get_context_jsonschema(self) -> dict[str, Any] | None:
        if (context_schema := self.context_schema) is None:
            return None

        if isclass(context_schema) and issubclass(context_schema, BaseModel):
            return context_schema.model_json_schema()
        elif is_typeddict(context_schema) or is_dataclass(context_schema):
            return TypeAdapter(context_schema).json_schema()
        else:
            raise ValueError(
                f"Invalid context schema type: {context_schema}. Must be a BaseModel, TypedDict or dataclass."
            )

    @property
    def InputType(self) -> Any:
        if isinstance(self.input_channels, str):
            channel = self.channels[self.input_channels]
            if isinstance(channel, BaseChannel):
                return channel.UpdateType

    def get_input_schema(self, config: RunnableConfig | None = None) -> type[BaseModel]:
        config = merge_configs(self.config, config)
        if isinstance(self.input_channels, str):
            return super().get_input_schema(config)
        else:
            return create_model(
                self.get_name("Input"),
                field_definitions={
                    k: (c.UpdateType, None)
                    for k in self.input_channels or self.channels.keys()
                    if (c := self.channels[k]) and isinstance(c, BaseChannel)
                },
            )

    def get_input_jsonschema(
        self, config: RunnableConfig | None = None
    ) -> dict[str, Any]:
        schema = self.get_input_schema(config)
        return schema.model_json_schema()

    @property
    def OutputType(self) -> Any:
        if isinstance(self.output_channels, str):
            channel = self.channels[self.output_channels]
            if isinstance(channel, BaseChannel):
                return channel.ValueType

    def get_output_schema(
        self, config: RunnableConfig | None = None
    ) -> type[BaseModel]:
        config = merge_configs(self.config, config)
        if isinstance(self.output_channels, str):
            return super().get_output_schema(config)
        else:
            return create_model(
                self.get_name("Output"),
                field_definitions={
                    k: (c.ValueType, None)
                    for k in self.output_channels
                    if (c := self.channels[k]) and isinstance(c, BaseChannel)
                },
            )

    def get_output_jsonschema(
        self, config: RunnableConfig | None = None
    ) -> dict[str, Any]:
        schema = self.get_output_schema(config)
        return schema.model_json_schema()

    @property
    def stream_channels_list(self) -> Sequence[str]:
        stream_channels = self.stream_channels_asis
        return (
            [stream_channels] if isinstance(stream_channels, str) else stream_channels
        )

    @property
    def stream_channels_asis(self) -> str | Sequence[str]:
        return self.stream_channels or [
            k for k in self.channels if isinstance(self.channels[k], BaseChannel)
        ]

    def get_subgraphs(
        self, *, namespace: str | None = None, recurse: bool = False
    ) -> Iterator[tuple[str, PregelProtocol]]:
        """Get the subgraphs of the graph.

        Args:
            namespace: The namespace to filter the subgraphs by.
            recurse: Whether to recurse into the subgraphs.
                If `False`, only the immediate subgraphs will be returned.

        Returns:
            An iterator of the `(namespace, subgraph)` pairs.
        """
        for name, node in self.nodes.items():
            # filter by prefix
            if namespace is not None:
                if not namespace.startswith(name):
                    continue

            # find the subgraph, if any
            graph = node.subgraphs[0] if node.subgraphs else None

            # if found, yield recursively
            if graph:
                if name == namespace:
                    yield name, graph
                    return  # we found it, stop searching
                if namespace is None:
                    yield name, graph
                if recurse and isinstance(graph, Pregel):
                    if namespace is not None:
                        namespace = namespace[len(name) + 1 :]
                    yield from (
                        (f"{name}{NS_SEP}{n}", s)
                        for n, s in graph.get_subgraphs(
                            namespace=namespace, recurse=recurse
                        )
                    )

    async def aget_subgraphs(
        self, *, namespace: str | None = None, recurse: bool = False
    ) -> AsyncIterator[tuple[str, PregelProtocol]]:
        """Get the subgraphs of the graph.

        Args:
            namespace: The namespace to filter the subgraphs by.
            recurse: Whether to recurse into the subgraphs.
                If `False`, only the immediate subgraphs will be returned.

        Returns:
            An iterator of the `(namespace, subgraph)` pairs.
        """
        for name, node in self.get_subgraphs(namespace=namespace, recurse=recurse):
            yield name, node

    def _migrate_checkpoint(self, checkpoint: Checkpoint) -> None:
        """Migrate a saved checkpoint to new channel layout."""
        if checkpoint["v"] < 4 and checkpoint.get("pending_sends"):
            pending_sends: list[Send] = checkpoint.pop("pending_sends")
            checkpoint["channel_values"][TASKS] = pending_sends
            checkpoint["channel_versions"][TASKS] = max(
                checkpoint["channel_versions"].values()
            )

    def _prepare_state_snapshot(
        self,
        config: RunnableConfig,
        saved: CheckpointTuple | None,
        recurse: BaseCheckpointSaver | None = None,
        apply_pending_writes: bool = False,
    ) -> StateSnapshot:
        if not saved:
            return StateSnapshot(
                values={},
                next=(),
                config=config,
                metadata=None,
                created_at=None,
                parent_config=None,
                tasks=(),
                interrupts=(),
            )

        # migrate checkpoint if needed
        self._migrate_checkpoint(saved.checkpoint)

        step = saved.metadata.get("step", -1) + 1
        stop = step + 2
        channels, managed = channels_from_checkpoint(
            self.channels,
            saved.checkpoint,
        )
        # tasks for this checkpoint
        next_tasks = prepare_next_tasks(
            saved.checkpoint,
            saved.pending_writes or [],
            self.nodes,
            channels,
            managed,
            saved.config,
            step,
            stop,
            for_execution=True,
            store=self.store,
            checkpointer=(
                self.checkpointer
                if isinstance(self.checkpointer, BaseCheckpointSaver)
                else None
            ),
            manager=None,
        )
        # get the subgraphs
        subgraphs = dict(self.get_subgraphs())
        parent_ns = saved.config[CONF].get(CONFIG_KEY_CHECKPOINT_NS, "")
        task_states: dict[str, RunnableConfig | StateSnapshot] = {}
        for task in next_tasks.values():
            if task.name not in subgraphs:
                continue
            # assemble checkpoint_ns for this task
            task_ns = f"{task.name}{NS_END}{task.id}"
            if parent_ns:
                task_ns = f"{parent_ns}{NS_SEP}{task_ns}"
            if not recurse:
                # set config as signal that subgraph checkpoints exist
                config = {
                    CONF: {
                        "thread_id": saved.config[CONF]["thread_id"],
                        CONFIG_KEY_CHECKPOINT_NS: task_ns,
                    }
                }
                task_states[task.id] = config
            else:
                # get the state of the subgraph
                config = {
                    CONF: {
                        CONFIG_KEY_CHECKPOINTER: recurse,
                        "thread_id": saved.config[CONF]["thread_id"],
                        CONFIG_KEY_CHECKPOINT_NS: task_ns,
                    }
                }
                task_states[task.id] = subgraphs[task.name].get_state(
                    config, subgraphs=True
                )
        # apply pending writes
        if null_writes := [
            w[1:] for w in saved.pending_writes or [] if w[0] == NULL_TASK_ID
        ]:
            apply_writes(
                saved.checkpoint,
                channels,
                [PregelTaskWrites((), INPUT, null_writes, [])],
                None,
                self.trigger_to_nodes,
            )
        if apply_pending_writes and saved.pending_writes:
            for tid, k, v in saved.pending_writes:
                if k in (ERROR, INTERRUPT):
                    continue
                if tid not in next_tasks:
                    continue
                next_tasks[tid].writes.append((k, v))
            if tasks := [t for t in next_tasks.values() if t.writes]:
                apply_writes(
                    saved.checkpoint, channels, tasks, None, self.trigger_to_nodes
                )
        tasks_with_writes = tasks_w_writes(
            next_tasks.values(),
            saved.pending_writes,
            task_states,
            self.stream_channels_asis,
        )
        # assemble the state snapshot
        return StateSnapshot(
            read_channels(channels, self.stream_channels_asis),
            tuple(t.name for t in next_tasks.values() if not t.writes),
            patch_checkpoint_map(saved.config, saved.metadata),
            saved.metadata,
            saved.checkpoint["ts"],
            patch_checkpoint_map(saved.parent_config, saved.metadata),
            tasks_with_writes,
            tuple([i for task in tasks_with_writes for i in task.interrupts]),
        )

    async def _aprepare_state_snapshot(
        self,
        config: RunnableConfig,
        saved: CheckpointTuple | None,
        recurse: BaseCheckpointSaver | None = None,
        apply_pending_writes: bool = False,
    ) -> StateSnapshot:
        if not saved:
            return StateSnapshot(
                values={},
                next=(),
                config=config,
                metadata=None,
                created_at=None,
                parent_config=None,
                tasks=(),
                interrupts=(),
            )

        # migrate checkpoint if needed
        self._migrate_checkpoint(saved.checkpoint)

        step = saved.metadata.get("step", -1) + 1
        stop = step + 2
        channels, managed = channels_from_checkpoint(
            self.channels,
            saved.checkpoint,
        )
        # tasks for this checkpoint
        next_tasks = prepare_next_tasks(
            saved.checkpoint,
            saved.pending_writes or [],
            self.nodes,
            channels,
            managed,
            saved.config,
            step,
            stop,
            for_execution=True,
            store=self.store,
            checkpointer=(
                self.checkpointer
                if isinstance(self.checkpointer, BaseCheckpointSaver)
                else None
            ),
            manager=None,
        )
        # get the subgraphs
        subgraphs = {n: g async for n, g in self.aget_subgraphs()}
        parent_ns = saved.config[CONF].get(CONFIG_KEY_CHECKPOINT_NS, "")
        task_states: dict[str, RunnableConfig | StateSnapshot] = {}
        for task in next_tasks.values():
            if task.name not in subgraphs:
                continue
            # assemble checkpoint_ns for this task
            task_ns = f"{task.name}{NS_END}{task.id}"
            if parent_ns:
                task_ns = f"{parent_ns}{NS_SEP}{task_ns}"
            if not recurse:
                # set config as signal that subgraph checkpoints exist
                config = {
                    CONF: {
                        "thread_id": saved.config[CONF]["thread_id"],
                        CONFIG_KEY_CHECKPOINT_NS: task_ns,
                    }
                }
                task_states[task.id] = config
            else:
                # get the state of the subgraph
                config = {
                    CONF: {
                        CONFIG_KEY_CHECKPOINTER: recurse,
                        "thread_id": saved.config[CONF]["thread_id"],
                        CONFIG_KEY_CHECKPOINT_NS: task_ns,
                    }
                }
                task_states[task.id] = await subgraphs[task.name].aget_state(
                    config, subgraphs=True
                )
        # apply pending writes
        if null_writes := [
            w[1:] for w in saved.pending_writes or [] if w[0] == NULL_TASK_ID
        ]:
            apply_writes(
                saved.checkpoint,
                channels,
                [PregelTaskWrites((), INPUT, null_writes, [])],
                None,
                self.trigger_to_nodes,
            )
        if apply_pending_writes and saved.pending_writes:
            for tid, k, v in saved.pending_writes:
                if k in (ERROR, INTERRUPT):
                    continue
                if tid not in next_tasks:
                    continue
                next_tasks[tid].writes.append((k, v))
            if tasks := [t for t in next_tasks.values() if t.writes]:
                apply_writes(
                    saved.checkpoint, channels, tasks, None, self.trigger_to_nodes
                )

        tasks_with_writes = tasks_w_writes(
            next_tasks.values(),
            saved.pending_writes,
            task_states,
            self.stream_channels_asis,
        )
        # assemble the state snapshot
        return StateSnapshot(
            read_channels(channels, self.stream_channels_asis),
            tuple(t.name for t in next_tasks.values() if not t.writes),
            patch_checkpoint_map(saved.config, saved.metadata),
            saved.metadata,
            saved.checkpoint["ts"],
            patch_checkpoint_map(saved.parent_config, saved.metadata),
            tasks_with_writes,
            tuple([i for task in tasks_with_writes for i in task.interrupts]),
        )

    def get_state(
        self, config: RunnableConfig, *, subgraphs: bool = False
    ) -> StateSnapshot:
        """Get the current state of the graph."""
        checkpointer: BaseCheckpointSaver | None = ensure_config(config)[CONF].get(
            CONFIG_KEY_CHECKPOINTER, self.checkpointer
        )
        if not checkpointer:
            raise ValueError("No checkpointer set")

        if (
            checkpoint_ns := config[CONF].get(CONFIG_KEY_CHECKPOINT_NS, "")
        ) and CONFIG_KEY_CHECKPOINTER not in config[CONF]:
            # remove task_ids from checkpoint_ns
            recast = recast_checkpoint_ns(checkpoint_ns)
            # find the subgraph with the matching name
            for _, pregel in self.get_subgraphs(namespace=recast, recurse=True):
                return pregel.get_state(
                    patch_configurable(config, {CONFIG_KEY_CHECKPOINTER: checkpointer}),
                    subgraphs=subgraphs,
                )
            else:
                raise ValueError(f"Subgraph {recast} not found")

        config = merge_configs(self.config, config) if self.config else config
        if self.checkpointer is True:
            ns = cast(str, config[CONF][CONFIG_KEY_CHECKPOINT_NS])
            config = merge_configs(
                config, {CONF: {CONFIG_KEY_CHECKPOINT_NS: recast_checkpoint_ns(ns)}}
            )
        thread_id = config[CONF][CONFIG_KEY_THREAD_ID]
        if not isinstance(thread_id, str):
            config[CONF][CONFIG_KEY_THREAD_ID] = str(thread_id)

        saved = checkpointer.get_tuple(config)
        return self._prepare_state_snapshot(
            config,
            saved,
            recurse=checkpointer if subgraphs else None,
            apply_pending_writes=CONFIG_KEY_CHECKPOINT_ID not in config[CONF],
        )

    async def aget_state(
        self, config: RunnableConfig, *, subgraphs: bool = False
    ) -> StateSnapshot:
        """Get the current state of the graph."""
        checkpointer: BaseCheckpointSaver | None = ensure_config(config)[CONF].get(
            CONFIG_KEY_CHECKPOINTER, self.checkpointer
        )
        if not checkpointer:
            raise ValueError("No checkpointer set")

        if (
            checkpoint_ns := config[CONF].get(CONFIG_KEY_CHECKPOINT_NS, "")
        ) and CONFIG_KEY_CHECKPOINTER not in config[CONF]:
            # remove task_ids from checkpoint_ns
            recast = recast_checkpoint_ns(checkpoint_ns)
            # find the subgraph with the matching name
            async for _, pregel in self.aget_subgraphs(namespace=recast, recurse=True):
                return await pregel.aget_state(
                    patch_configurable(config, {CONFIG_KEY_CHECKPOINTER: checkpointer}),
                    subgraphs=subgraphs,
                )
            else:
                raise ValueError(f"Subgraph {recast} not found")

        config = merge_configs(self.config, config) if self.config else config
        if self.checkpointer is True:
            ns = cast(str, config[CONF][CONFIG_KEY_CHECKPOINT_NS])
            config = merge_configs(
                config, {CONF: {CONFIG_KEY_CHECKPOINT_NS: recast_checkpoint_ns(ns)}}
            )
        thread_id = config[CONF][CONFIG_KEY_THREAD_ID]
        if not isinstance(thread_id, str):
            config[CONF][CONFIG_KEY_THREAD_ID] = str(thread_id)

        saved = await checkpointer.aget_tuple(config)
        return await self._aprepare_state_snapshot(
            config,
            saved,
            recurse=checkpointer if subgraphs else None,
            apply_pending_writes=CONFIG_KEY_CHECKPOINT_ID not in config[CONF],
        )

    def get_state_history(
        self,
        config: RunnableConfig,
        *,
        filter: dict[str, Any] | None = None,
        before: RunnableConfig | None = None,
        limit: int | None = None,
    ) -> Iterator[StateSnapshot]:
        """Get the history of the state of the graph."""
        config = ensure_config(config)
        checkpointer: BaseCheckpointSaver | None = config[CONF].get(
            CONFIG_KEY_CHECKPOINTER, self.checkpointer
        )
        if not checkpointer:
            raise ValueError("No checkpointer set")

        if (
            checkpoint_ns := config[CONF].get(CONFIG_KEY_CHECKPOINT_NS, "")
        ) and CONFIG_KEY_CHECKPOINTER not in config[CONF]:
            # remove task_ids from checkpoint_ns
            recast = recast_checkpoint_ns(checkpoint_ns)
            # find the subgraph with the matching name
            for _, pregel in self.get_subgraphs(namespace=recast, recurse=True):
                yield from pregel.get_state_history(
                    patch_configurable(config, {CONFIG_KEY_CHECKPOINTER: checkpointer}),
                    filter=filter,
                    before=before,
                    limit=limit,
                )
                return
            else:
                raise ValueError(f"Subgraph {recast} not found")

        config = merge_configs(
            self.config,
            config,
            {
                CONF: {
                    CONFIG_KEY_CHECKPOINT_NS: checkpoint_ns,
                    CONFIG_KEY_THREAD_ID: str(config[CONF][CONFIG_KEY_THREAD_ID]),
                }
            },
        )
        # eagerly consume list() to avoid holding up the db cursor
        for checkpoint_tuple in list(
            checkpointer.list(config, before=before, limit=limit, filter=filter)
        ):
            yield self._prepare_state_snapshot(
                checkpoint_tuple.config, checkpoint_tuple
            )

    async def aget_state_history(
        self,
        config: RunnableConfig,
        *,
        filter: dict[str, Any] | None = None,
        before: RunnableConfig | None = None,
        limit: int | None = None,
    ) -> AsyncIterator[StateSnapshot]:
        """Asynchronously get the history of the state of the graph."""
        config = ensure_config(config)
        checkpointer: BaseCheckpointSaver | None = ensure_config(config)[CONF].get(
            CONFIG_KEY_CHECKPOINTER, self.checkpointer
        )
        if not checkpointer:
            raise ValueError("No checkpointer set")

        if (
            checkpoint_ns := config[CONF].get(CONFIG_KEY_CHECKPOINT_NS, "")
        ) and CONFIG_KEY_CHECKPOINTER not in config[CONF]:
            # remove task_ids from checkpoint_ns
            recast = recast_checkpoint_ns(checkpoint_ns)
            # find the subgraph with the matching name
            async for _, pregel in self.aget_subgraphs(namespace=recast, recurse=True):
                async for state in pregel.aget_state_history(
                    patch_configurable(config, {CONFIG_KEY_CHECKPOINTER: checkpointer}),
                    filter=filter,
                    before=before,
                    limit=limit,
                ):
                    yield state
                return
            else:
                raise ValueError(f"Subgraph {recast} not found")

        config = merge_configs(
            self.config,
            config,
            {
                CONF: {
                    CONFIG_KEY_CHECKPOINT_NS: checkpoint_ns,
                    CONFIG_KEY_THREAD_ID: str(config[CONF][CONFIG_KEY_THREAD_ID]),
                }
            },
        )
        # eagerly consume list() to avoid holding up the db cursor
        for checkpoint_tuple in [
            c
            async for c in checkpointer.alist(
                config, before=before, limit=limit, filter=filter
            )
        ]:
            yield await self._aprepare_state_snapshot(
                checkpoint_tuple.config, checkpoint_tuple
            )

    def bulk_update_state(
        self,
        config: RunnableConfig,
        supersteps: Sequence[Sequence[StateUpdate]],
    ) -> RunnableConfig:
        """Apply updates to the graph state in bulk. Requires a checkpointer to be set.

        Args:
            config: The config to apply the updates to.
            supersteps: A list of supersteps, each including a list of updates to apply sequentially to a graph state.
                        Each update is a tuple of the form `(values, as_node, task_id)` where `task_id` is optional.

        Raises:
            ValueError: If no checkpointer is set or no updates are provided.
            InvalidUpdateError: If an invalid update is provided.

        Returns:
            RunnableConfig: The updated config.
        """

        checkpointer: BaseCheckpointSaver | None = ensure_config(config)[CONF].get(
            CONFIG_KEY_CHECKPOINTER, self.checkpointer
        )
        if not checkpointer:
            raise ValueError("No checkpointer set")

        if len(supersteps) == 0:
            raise ValueError("No supersteps provided")

        if any(len(u) == 0 for u in supersteps):
            raise ValueError("No updates provided")

        # delegate to subgraph
        if (
            checkpoint_ns := config[CONF].get(CONFIG_KEY_CHECKPOINT_NS, "")
        ) and CONFIG_KEY_CHECKPOINTER not in config[CONF]:
            # remove task_ids from checkpoint_ns
            recast = recast_checkpoint_ns(checkpoint_ns)
            # find the subgraph with the matching name
            for _, pregel in self.get_subgraphs(namespace=recast, recurse=True):
                return pregel.bulk_update_state(
                    patch_configurable(config, {CONFIG_KEY_CHECKPOINTER: checkpointer}),
                    supersteps,
                )
            else:
                raise ValueError(f"Subgraph {recast} not found")

        def perform_superstep(
            input_config: RunnableConfig, updates: Sequence[StateUpdate]
        ) -> RunnableConfig:
            # get last checkpoint
            config = ensure_config(self.config, input_config)
            saved = checkpointer.get_tuple(config)
            if saved is not None:
                self._migrate_checkpoint(saved.checkpoint)
            checkpoint = (
                copy_checkpoint(saved.checkpoint) if saved else empty_checkpoint()
            )
            checkpoint_previous_versions = (
                saved.checkpoint["channel_versions"].copy() if saved else {}
            )
            step = saved.metadata.get("step", -1) if saved else -1
            # merge configurable fields with previous checkpoint config
            checkpoint_config = patch_configurable(
                config,
                {
                    CONFIG_KEY_CHECKPOINT_NS: config[CONF].get(
                        CONFIG_KEY_CHECKPOINT_NS, ""
                    )
                },
            )
            if saved:
                checkpoint_config = patch_configurable(config, saved.config[CONF])
            channels, managed = channels_from_checkpoint(
                self.channels,
                checkpoint,
            )
            values, as_node = updates[0][:2]

            # no values as END, just clear all tasks
            if values is None and as_node == END:
                if len(updates) > 1:
                    raise InvalidUpdateError(
                        "Cannot apply multiple updates when clearing state"
                    )

                if saved is not None:
                    # tasks for this checkpoint
                    next_tasks = prepare_next_tasks(
                        checkpoint,
                        saved.pending_writes or [],
                        self.nodes,
                        channels,
                        managed,
                        saved.config,
                        step + 1,
                        step + 3,
                        for_execution=True,
                        store=self.store,
                        checkpointer=checkpointer,
                        manager=None,
                    )
                    # apply null writes
                    if null_writes := [
                        w[1:]
                        for w in saved.pending_writes or []
                        if w[0] == NULL_TASK_ID
                    ]:
                        apply_writes(
                            checkpoint,
                            channels,
                            [PregelTaskWrites((), INPUT, null_writes, [])],
                            checkpointer.get_next_version,
                            self.trigger_to_nodes,
                        )
                    # apply writes from tasks that already ran
                    for tid, k, v in saved.pending_writes or []:
                        if k in (ERROR, INTERRUPT):
                            continue
                        if tid not in next_tasks:
                            continue
                        next_tasks[tid].writes.append((k, v))
                    # clear all current tasks
                    apply_writes(
                        checkpoint,
                        channels,
                        next_tasks.values(),
                        checkpointer.get_next_version,
                        self.trigger_to_nodes,
                    )
                # save checkpoint
                next_config = checkpointer.put(
                    checkpoint_config,
                    create_checkpoint(checkpoint, channels, step),
                    {
                        "source": "update",
                        "step": step + 1,
                        "parents": saved.metadata.get("parents", {}) if saved else {},
                    },
                    get_new_channel_versions(
                        checkpoint_previous_versions,
                        checkpoint["channel_versions"],
                    ),
                )
                return patch_checkpoint_map(
                    next_config, saved.metadata if saved else None
                )

            # act as an input
            if as_node == INPUT:
                if len(updates) > 1:
                    raise InvalidUpdateError(
                        "Cannot apply multiple updates when updating as input"
                    )

                if input_writes := deque(map_input(self.input_channels, values)):
                    apply_writes(
                        checkpoint,
                        channels,
                        [PregelTaskWrites((), INPUT, input_writes, [])],
                        checkpointer.get_next_version,
                        self.trigger_to_nodes,
                    )

                    # apply input write to channels
                    next_step = (
                        step + 1
                        if saved and saved.metadata.get("step") is not None
                        else -1
                    )
                    next_config = checkpointer.put(
                        checkpoint_config,
                        create_checkpoint(checkpoint, channels, next_step),
                        {
                            "source": "input",
                            "step": next_step,
                            "parents": saved.metadata.get("parents", {})
                            if saved
                            else {},
                        },
                        get_new_channel_versions(
                            checkpoint_previous_versions,
                            checkpoint["channel_versions"],
                        ),
                    )

                    # store the writes
                    checkpointer.put_writes(
                        next_config,
                        input_writes,
                        str(uuid5(UUID(checkpoint["id"]), INPUT)),
                    )

                    return patch_checkpoint_map(
                        next_config, saved.metadata if saved else None
                    )
                else:
                    raise InvalidUpdateError(
                        f"Received no input writes for {self.input_channels}"
                    )

            # copy checkpoint
            if as_node == "__copy__":
                if len(updates) > 1:
                    raise InvalidUpdateError(
                        "Cannot copy checkpoint with multiple updates"
                    )

                if saved is None:
                    raise InvalidUpdateError("Cannot copy a non-existent checkpoint")

                next_checkpoint = create_checkpoint(checkpoint, None, step)

                # copy checkpoint
                next_config = checkpointer.put(
                    saved.parent_config
                    or patch_configurable(
                        saved.config, {CONFIG_KEY_CHECKPOINT_ID: None}
                    ),
                    next_checkpoint,
                    {
                        "source": "fork",
                        "step": step + 1,
                        "parents": saved.metadata.get("parents", {}),
                    },
                    {},
                )

                # we want to both clone a checkpoint and update state in one go.
                # reuse the same task ID if possible.
                if isinstance(values, list) and len(values) > 0:
                    # figure out the task IDs for the next update checkpoint
                    next_tasks = prepare_next_tasks(
                        next_checkpoint,
                        saved.pending_writes or [],
                        self.nodes,
                        channels,
                        managed,
                        next_config,
                        step + 2,
                        step + 4,
                        for_execution=True,
                        store=self.store,
                        checkpointer=checkpointer,
                        manager=None,
                    )

                    tasks_group_by = defaultdict(list)
                    user_group_by: dict[str, list[StateUpdate]] = defaultdict(list)

                    for task in next_tasks.values():
                        tasks_group_by[task.name].append(task.id)

                    for item in values:
                        if not isinstance(item, Sequence):
                            raise InvalidUpdateError(
                                f"Invalid update item: {item} when copying checkpoint"
                            )

                        values, as_node = item[:2]

                        user_group = user_group_by[as_node]
                        tasks_group = tasks_group_by[as_node]

                        target_idx = len(user_group)
                        task_id = (
                            tasks_group[target_idx]
                            if target_idx < len(tasks_group)
                            else None
                        )

                        user_group_by[as_node].append(
                            StateUpdate(values=values, as_node=as_node, task_id=task_id)
                        )

                    return perform_superstep(
                        patch_checkpoint_map(next_config, saved.metadata),
                        [item for lst in user_group_by.values() for item in lst],
                    )

                return patch_checkpoint_map(next_config, saved.metadata)

            # task ids can be provided in the StateUpdate, but if not,
            # we use the task id generated by prepare_next_tasks
            node_to_task_ids: dict[str, deque[str]] = defaultdict(deque)
            if saved is not None and saved.pending_writes is not None:
                # we call prepare_next_tasks to discover the task IDs that
                # would have been generated, so we can reuse them and
                # properly populate task.result in state history
                next_tasks = prepare_next_tasks(
                    checkpoint,
                    saved.pending_writes,
                    self.nodes,
                    channels,
                    managed,
                    saved.config,
                    step + 1,
                    step + 3,
                    for_execution=True,
                    store=self.store,
                    checkpointer=checkpointer,
                    manager=None,
                )
                # collect task ids to reuse so we can properly attach task results
                for t in next_tasks.values():
                    node_to_task_ids[t.name].append(t.id)

            valid_updates: list[tuple[str, dict[str, Any] | None, str | None]] = []
            if len(updates) == 1:
                values, as_node, task_id = updates[0]
                # find last node that updated the state, if not provided
                if as_node is None and len(self.nodes) == 1:
                    as_node = tuple(self.nodes)[0]
                elif as_node is None and not any(
                    v
                    for vv in checkpoint["versions_seen"].values()
                    for v in vv.values()
                ):
                    if (
                        isinstance(self.input_channels, str)
                        and self.input_channels in self.nodes
                    ):
                        as_node = self.input_channels
                elif as_node is None:
                    last_seen_by_node = sorted(
                        (v, n)
                        for n, seen in checkpoint["versions_seen"].items()
                        if n in self.nodes
                        for v in seen.values()
                    )
                    # if two nodes updated the state at the same time, it's ambiguous
                    if last_seen_by_node:
                        if len(last_seen_by_node) == 1:
                            as_node = last_seen_by_node[0][1]
                        elif last_seen_by_node[-1][0] != last_seen_by_node[-2][0]:
                            as_node = last_seen_by_node[-1][1]
                if as_node is None:
                    raise InvalidUpdateError("Ambiguous update, specify as_node")
                if as_node not in self.nodes:
                    raise InvalidUpdateError(f"Node {as_node} does not exist")
                valid_updates.append((as_node, values, task_id))
            else:
                for values, as_node, task_id in updates:
                    if as_node is None:
                        raise InvalidUpdateError(
                            "as_node is required when applying multiple updates"
                        )
                    if as_node not in self.nodes:
                        raise InvalidUpdateError(f"Node {as_node} does not exist")

                    valid_updates.append((as_node, values, task_id))

            run_tasks: list[PregelTaskWrites] = []
            run_task_ids: list[str] = []

            for as_node, values, provided_task_id in valid_updates:
                # create task to run all writers of the chosen node
                writers = self.nodes[as_node].flat_writers
                if not writers:
                    raise InvalidUpdateError(f"Node {as_node} has no writers")
                writes: deque[tuple[str, Any]] = deque()
                task = PregelTaskWrites((), as_node, writes, [INTERRUPT])
                # get the task ids that were prepared for this node
                # if a task id was provided in the StateUpdate, we use it
                # otherwise, we use the next available task id
                prepared_task_ids = node_to_task_ids.get(as_node, deque())
                task_id = provided_task_id or (
                    prepared_task_ids.popleft()
                    if prepared_task_ids
                    else str(uuid5(UUID(checkpoint["id"]), INTERRUPT))
                )
                run_tasks.append(task)
                run_task_ids.append(task_id)
                run = RunnableSequence(*writers) if len(writers) > 1 else writers[0]
                # execute task
                run.invoke(
                    values,
                    patch_config(
                        config,
                        run_name=self.name + "UpdateState",
                        configurable={
                            # deque.extend is thread-safe
                            CONFIG_KEY_SEND: writes.extend,
                            CONFIG_KEY_TASK_ID: task_id,
                            CONFIG_KEY_READ: partial(
                                local_read,
                                _scratchpad(
                                    None,
                                    [],
                                    task_id,
                                    "",
                                    None,
                                    step,
                                    step + 2,
                                ),
                                channels,
                                managed,
                                task,
                            ),
                        },
                    ),
                )
            # save task writes
            for task_id, task in zip(run_task_ids, run_tasks):
                # channel writes are saved to current checkpoint
                channel_writes = [w for w in task.writes if w[0] != PUSH]
                if saved and channel_writes:
                    checkpointer.put_writes(checkpoint_config, channel_writes, task_id)
            # apply to checkpoint and save
            apply_writes(
                checkpoint,
                channels,
                run_tasks,
                checkpointer.get_next_version,
                self.trigger_to_nodes,
            )
            checkpoint = create_checkpoint(checkpoint, channels, step + 1)
            next_config = checkpointer.put(
                checkpoint_config,
                checkpoint,
                {
                    "source": "update",
                    "step": step + 1,
                    "parents": saved.metadata.get("parents", {}) if saved else {},
                },
                get_new_channel_versions(
                    checkpoint_previous_versions, checkpoint["channel_versions"]
                ),
            )
            for task_id, task in zip(run_task_ids, run_tasks):
                # save push writes
                if push_writes := [w for w in task.writes if w[0] == PUSH]:
                    checkpointer.put_writes(next_config, push_writes, task_id)

            return patch_checkpoint_map(next_config, saved.metadata if saved else None)

        current_config = patch_configurable(
            config, {CONFIG_KEY_THREAD_ID: str(config[CONF][CONFIG_KEY_THREAD_ID])}
        )
        for superstep in supersteps:
            current_config = perform_superstep(current_config, superstep)
        return current_config

    async def abulk_update_state(
        self,
        config: RunnableConfig,
        supersteps: Sequence[Sequence[StateUpdate]],
    ) -> RunnableConfig:
        """Asynchronously apply updates to the graph state in bulk. Requires a checkpointer to be set.

        Args:
            config: The config to apply the updates to.
            supersteps: A list of supersteps, each including a list of updates to apply sequentially to a graph state.
                        Each update is a tuple of the form `(values, as_node, task_id)` where `task_id` is optional.

        Raises:
            ValueError: If no checkpointer is set or no updates are provided.
            InvalidUpdateError: If an invalid update is provided.

        Returns:
            RunnableConfig: The updated config.
        """

        checkpointer: BaseCheckpointSaver | None = ensure_config(config)[CONF].get(
            CONFIG_KEY_CHECKPOINTER, self.checkpointer
        )
        if not checkpointer:
            raise ValueError("No checkpointer set")

        if len(supersteps) == 0:
            raise ValueError("No supersteps provided")

        if any(len(u) == 0 for u in supersteps):
            raise ValueError("No updates provided")

        # delegate to subgraph
        if (
            checkpoint_ns := config[CONF].get(CONFIG_KEY_CHECKPOINT_NS, "")
        ) and CONFIG_KEY_CHECKPOINTER not in config[CONF]:
            # remove task_ids from checkpoint_ns
            recast = recast_checkpoint_ns(checkpoint_ns)
            # find the subgraph with the matching name
            async for _, pregel in self.aget_subgraphs(namespace=recast, recurse=True):
                return await pregel.abulk_update_state(
                    patch_configurable(config, {CONFIG_KEY_CHECKPOINTER: checkpointer}),
                    supersteps,
                )
            else:
                raise ValueError(f"Subgraph {recast} not found")

        async def aperform_superstep(
            input_config: RunnableConfig, updates: Sequence[StateUpdate]
        ) -> RunnableConfig:
            # get last checkpoint
            config = ensure_config(self.config, input_config)
            saved = await checkpointer.aget_tuple(config)
            if saved is not None:
                self._migrate_checkpoint(saved.checkpoint)
            checkpoint = (
                copy_checkpoint(saved.checkpoint) if saved else empty_checkpoint()
            )
            checkpoint_previous_versions = (
                saved.checkpoint["channel_versions"].copy() if saved else {}
            )
            step = saved.metadata.get("step", -1) if saved else -1
            # merge configurable fields with previous checkpoint config
            checkpoint_config = patch_configurable(
                config,
                {
                    CONFIG_KEY_CHECKPOINT_NS: config[CONF].get(
                        CONFIG_KEY_CHECKPOINT_NS, ""
                    )
                },
            )
            if saved:
                checkpoint_config = patch_configurable(config, saved.config[CONF])
            channels, managed = channels_from_checkpoint(
                self.channels,
                checkpoint,
            )
            values, as_node = updates[0][:2]
            # no values, just clear all tasks
            if values is None and as_node == END:
                if len(updates) > 1:
                    raise InvalidUpdateError(
                        "Cannot apply multiple updates when clearing state"
                    )
                if saved is not None:
                    # tasks for this checkpoint
                    next_tasks = prepare_next_tasks(
                        checkpoint,
                        saved.pending_writes or [],
                        self.nodes,
                        channels,
                        managed,
                        saved.config,
                        step + 1,
                        step + 3,
                        for_execution=True,
                        store=self.store,
                        checkpointer=checkpointer,
                        manager=None,
                    )
                    # apply null writes
                    if null_writes := [
                        w[1:]
                        for w in saved.pending_writes or []
                        if w[0] == NULL_TASK_ID
                    ]:
                        apply_writes(
                            checkpoint,
                            channels,
                            [PregelTaskWrites((), INPUT, null_writes, [])],
                            checkpointer.get_next_version,
                            self.trigger_to_nodes,
                        )
                    # apply writes from tasks that already ran
                    for tid, k, v in saved.pending_writes or []:
                        if k in (ERROR, INTERRUPT):
                            continue
                        if tid not in next_tasks:
                            continue
                        next_tasks[tid].writes.append((k, v))
                    # clear all current tasks
                    apply_writes(
                        checkpoint,
                        channels,
                        next_tasks.values(),
                        checkpointer.get_next_version,
                        self.trigger_to_nodes,
                    )
                # save checkpoint
                next_config = await checkpointer.aput(
                    checkpoint_config,
                    create_checkpoint(checkpoint, channels, step),
                    {
                        "source": "update",
                        "step": step + 1,
                        "parents": saved.metadata.get("parents", {}) if saved else {},
                    },
                    get_new_channel_versions(
                        checkpoint_previous_versions, checkpoint["channel_versions"]
                    ),
                )
                return patch_checkpoint_map(
                    next_config, saved.metadata if saved else None
                )

            # act as an input
            if as_node == INPUT:
                if len(updates) > 1:
                    raise InvalidUpdateError(
                        "Cannot apply multiple updates when updating as input"
                    )

                if input_writes := deque(map_input(self.input_channels, values)):
                    apply_writes(
                        checkpoint,
                        channels,
                        [PregelTaskWrites((), INPUT, input_writes, [])],
                        checkpointer.get_next_version,
                        self.trigger_to_nodes,
                    )

                    # apply input write to channels
                    next_step = (
                        step + 1
                        if saved and saved.metadata.get("step") is not None
                        else -1
                    )
                    next_config = await checkpointer.aput(
                        checkpoint_config,
                        create_checkpoint(checkpoint, channels, next_step),
                        {
                            "source": "input",
                            "step": next_step,
                            "parents": saved.metadata.get("parents", {})
                            if saved
                            else {},
                        },
                        get_new_channel_versions(
                            checkpoint_previous_versions,
                            checkpoint["channel_versions"],
                        ),
                    )

                    # store the writes
                    await checkpointer.aput_writes(
                        next_config,
                        input_writes,
                        str(uuid5(UUID(checkpoint["id"]), INPUT)),
                    )

                    return patch_checkpoint_map(
                        next_config, saved.metadata if saved else None
                    )
                else:
                    raise InvalidUpdateError(
                        f"Received no input writes for {self.input_channels}"
                    )

            # no values, copy checkpoint
            if as_node == "__copy__":
                if len(updates) > 1:
                    raise InvalidUpdateError(
                        "Cannot copy checkpoint with multiple updates"
                    )

                if saved is None:
                    raise InvalidUpdateError("Cannot copy a non-existent checkpoint")

                next_checkpoint = create_checkpoint(checkpoint, None, step)

                # copy checkpoint
                next_config = await checkpointer.aput(
                    saved.parent_config
                    or patch_configurable(
                        saved.config, {CONFIG_KEY_CHECKPOINT_ID: None}
                    ),
                    next_checkpoint,
                    {
                        "source": "fork",
                        "step": step + 1,
                        "parents": saved.metadata.get("parents", {}),
                    },
                    {},
                )

                # we want to both clone a checkpoint and update state in one go.
                # reuse the same task ID if possible.
                if isinstance(values, list) and len(values) > 0:
                    # figure out the task IDs for the next update checkpoint
                    next_tasks = prepare_next_tasks(
                        next_checkpoint,
                        saved.pending_writes or [],
                        self.nodes,
                        channels,
                        managed,
                        next_config,
                        step + 2,
                        step + 4,
                        for_execution=True,
                        store=self.store,
                        checkpointer=checkpointer,
                        manager=None,
                    )

                    tasks_group_by = defaultdict(list)
                    user_group_by: dict[str, list[StateUpdate]] = defaultdict(list)

                    for task in next_tasks.values():
                        tasks_group_by[task.name].append(task.id)

                    for item in values:
                        if not isinstance(item, Sequence):
                            raise InvalidUpdateError(
                                f"Invalid update item: {item} when copying checkpoint"
                            )

                        values, as_node = item[:2]
                        user_group = user_group_by[as_node]
                        tasks_group = tasks_group_by[as_node]

                        target_idx = len(user_group)
                        task_id = (
                            tasks_group[target_idx]
                            if target_idx < len(tasks_group)
                            else None
                        )

                        user_group_by[as_node].append(
                            StateUpdate(values=values, as_node=as_node, task_id=task_id)
                        )

                    return await aperform_superstep(
                        patch_checkpoint_map(next_config, saved.metadata),
                        [item for lst in user_group_by.values() for item in lst],
                    )

                return patch_checkpoint_map(
                    next_config, saved.metadata if saved else None
                )

            # task ids can be provided in the StateUpdate, but if not,
            # we use the task id generated by prepare_next_tasks
            node_to_task_ids: dict[str, deque[str]] = defaultdict(deque)
            if saved is not None and saved.pending_writes is not None:
                # we call prepare_next_tasks to discover the task IDs that
                # would have been generated, so we can reuse them and
                # properly populate task.result in state history
                next_tasks = prepare_next_tasks(
                    checkpoint,
                    saved.pending_writes,
                    self.nodes,
                    channels,
                    managed,
                    saved.config,
                    step + 1,
                    step + 3,
                    for_execution=True,
                    store=self.store,
                    checkpointer=checkpointer,
                    manager=None,
                )
                # collect task ids to reuse so we can properly attach task results
                for t in next_tasks.values():
                    node_to_task_ids[t.name].append(t.id)

            valid_updates: list[tuple[str, dict[str, Any] | None, str | None]] = []
            if len(updates) == 1:
                values, as_node, task_id = updates[0]
                # find last node that updated the state, if not provided
                if as_node is None and len(self.nodes) == 1:
                    as_node = tuple(self.nodes)[0]
                elif as_node is None and not saved:
                    if (
                        isinstance(self.input_channels, str)
                        and self.input_channels in self.nodes
                    ):
                        as_node = self.input_channels
                elif as_node is None:
                    last_seen_by_node = sorted(
                        (v, n)
                        for n, seen in checkpoint["versions_seen"].items()
                        if n in self.nodes
                        for v in seen.values()
                    )
                    # if two nodes updated the state at the same time, it's ambiguous
                    if last_seen_by_node:
                        if len(last_seen_by_node) == 1:
                            as_node = last_seen_by_node[0][1]
                        elif last_seen_by_node[-1][0] != last_seen_by_node[-2][0]:
                            as_node = last_seen_by_node[-1][1]
                if as_node is None:
                    raise InvalidUpdateError("Ambiguous update, specify as_node")
                if as_node not in self.nodes:
                    raise InvalidUpdateError(f"Node {as_node} does not exist")
                valid_updates.append((as_node, values, task_id))
            else:
                for values, as_node, task_id in updates:
                    if as_node is None:
                        raise InvalidUpdateError(
                            "as_node is required when applying multiple updates"
                        )
                    if as_node not in self.nodes:
                        raise InvalidUpdateError(f"Node {as_node} does not exist")

                    valid_updates.append((as_node, values, task_id))

            run_tasks: list[PregelTaskWrites] = []
            run_task_ids: list[str] = []

            for as_node, values, provided_task_id in valid_updates:
                # create task to run all writers of the chosen node
                writers = self.nodes[as_node].flat_writers
                if not writers:
                    raise InvalidUpdateError(f"Node {as_node} has no writers")
                writes: deque[tuple[str, Any]] = deque()
                task = PregelTaskWrites((), as_node, writes, [INTERRUPT])
                # get the task ids that were prepared for this node
                # if a task id was provided in the StateUpdate, we use it
                # otherwise, we use the next available task id
                prepared_task_ids = node_to_task_ids.get(as_node, deque())
                task_id = provided_task_id or (
                    prepared_task_ids.popleft()
                    if prepared_task_ids
                    else str(uuid5(UUID(checkpoint["id"]), INTERRUPT))
                )
                run_tasks.append(task)
                run_task_ids.append(task_id)
                run = RunnableSequence(*writers) if len(writers) > 1 else writers[0]
                # execute task
                await run.ainvoke(
                    values,
                    patch_config(
                        config,
                        run_name=self.name + "UpdateState",
                        configurable={
                            # deque.extend is thread-safe
                            CONFIG_KEY_SEND: writes.extend,
                            CONFIG_KEY_TASK_ID: task_id,
                            CONFIG_KEY_READ: partial(
                                local_read,
                                _scratchpad(
                                    None,
                                    [],
                                    task_id,
                                    "",
                                    None,
                                    step,
                                    step + 2,
                                ),
                                channels,
                                managed,
                                task,
                            ),
                        },
                    ),
                )
            # save task writes
            for task_id, task in zip(run_task_ids, run_tasks):
                # channel writes are saved to current checkpoint
                channel_writes = [w for w in task.writes if w[0] != PUSH]
                if saved and channel_writes:
                    await checkpointer.aput_writes(
                        checkpoint_config, channel_writes, task_id
                    )
            # apply to checkpoint and save
            apply_writes(
                checkpoint,
                channels,
                run_tasks,
                checkpointer.get_next_version,
                self.trigger_to_nodes,
            )
            checkpoint = create_checkpoint(checkpoint, channels, step + 1)
            # save checkpoint, after applying writes
            next_config = await checkpointer.aput(
                checkpoint_config,
                checkpoint,
                {
                    "source": "update",
                    "step": step + 1,
                    "parents": saved.metadata.get("parents", {}) if saved else {},
                },
                get_new_channel_versions(
                    checkpoint_previous_versions, checkpoint["channel_versions"]
                ),
            )
            for task_id, task in zip(run_task_ids, run_tasks):
                # save push writes
                if push_writes := [w for w in task.writes if w[0] == PUSH]:
                    await checkpointer.aput_writes(next_config, push_writes, task_id)
            return patch_checkpoint_map(next_config, saved.metadata if saved else None)

        current_config = patch_configurable(
            config, {CONFIG_KEY_THREAD_ID: str(config[CONF][CONFIG_KEY_THREAD_ID])}
        )
        for superstep in supersteps:
            current_config = await aperform_superstep(current_config, superstep)
        return current_config

    def update_state(
        self,
        config: RunnableConfig,
        values: dict[str, Any] | Any | None,
        as_node: str | None = None,
        task_id: str | None = None,
    ) -> RunnableConfig:
        """Update the state of the graph with the given values, as if they came from
        node `as_node`. If `as_node` is not provided, it will be set to the last node
        that updated the state, if not ambiguous.
        """
        return self.bulk_update_state(config, [[StateUpdate(values, as_node, task_id)]])

    async def aupdate_state(
        self,
        config: RunnableConfig,
        values: dict[str, Any] | Any,
        as_node: str | None = None,
        task_id: str | None = None,
    ) -> RunnableConfig:
        """Asynchronously update the state of the graph with the given values, as if they came from
        node `as_node`. If `as_node` is not provided, it will be set to the last node
        that updated the state, if not ambiguous.
        """
        return await self.abulk_update_state(
            config, [[StateUpdate(values, as_node, task_id)]]
        )

    def _defaults(
        self,
        config: RunnableConfig,
        *,
        stream_mode: StreamMode | Sequence[StreamMode],
        print_mode: StreamMode | Sequence[StreamMode],
        output_keys: str | Sequence[str] | None,
        interrupt_before: All | Sequence[str] | None,
        interrupt_after: All | Sequence[str] | None,
        durability: Durability | None = None,
    ) -> tuple[
        set[StreamMode],
        str | Sequence[str],
        All | Sequence[str],
        All | Sequence[str],
        BaseCheckpointSaver | None,
        BaseStore | None,
        BaseCache | None,
        Durability,
    ]:
        if config["recursion_limit"] < 1:
            raise ValueError("recursion_limit must be at least 1")
        if output_keys is None:
            output_keys = self.stream_channels_asis
        else:
            validate_keys(output_keys, self.channels)
        interrupt_before = interrupt_before or self.interrupt_before_nodes
        interrupt_after = interrupt_after or self.interrupt_after_nodes
        if isinstance(stream_mode, str):
            stream_modes = {stream_mode}
        else:
            stream_modes = set(stream_mode)
        if isinstance(print_mode, str):
            stream_modes.add(print_mode)
        else:
            stream_modes.update(print_mode)
        if self.checkpointer is False:
            checkpointer: BaseCheckpointSaver | None = None
        elif CONFIG_KEY_CHECKPOINTER in config.get(CONF, {}):
            checkpointer = config[CONF][CONFIG_KEY_CHECKPOINTER]
        elif self.checkpointer is True:
            raise RuntimeError("checkpointer=True cannot be used for root graphs.")
        else:
            checkpointer = self.checkpointer
        if checkpointer and not config.get(CONF):
            raise ValueError(
                "Checkpointer requires one or more of the following 'configurable' "
                "keys: thread_id, checkpoint_ns, checkpoint_id"
            )
        if CONFIG_KEY_RUNTIME in config.get(CONF, {}):
            store: BaseStore | None = config[CONF][CONFIG_KEY_RUNTIME].store
        else:
            store = self.store
        if CONFIG_KEY_CACHE in config.get(CONF, {}):
            cache: BaseCache | None = config[CONF][CONFIG_KEY_CACHE]
        else:
            cache = self.cache
        if durability is None:
            durability = config.get(CONF, {}).get(CONFIG_KEY_DURABILITY, "async")
        return (
            stream_modes,
            output_keys,
            interrupt_before,
            interrupt_after,
            checkpointer,
            store,
            cache,
            durability,
        )

    def stream(
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
    ) -> Iterator[dict[str, Any] | Any]:
        """Stream graph steps for a single input.

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

        stream = SyncQueue()

        config = ensure_config(self.config, config)
        callback_manager = get_callback_manager_for_config(config)
        run_manager = callback_manager.on_chain_start(
            None,
            input,
            name=config.get("run_name", self.get_name()),
            run_id=config.get("run_id"),
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
                ns_ = cast(str | None, config[CONF].get(CONFIG_KEY_CHECKPOINT_NS))
                run_manager.inheritable_handlers.append(
                    StreamMessagesHandler(
                        stream.put,
                        subgraphs,
                        parent_ns=tuple(ns_.split(NS_SEP)) if ns_ else None,
                    )
                )

            # set up custom stream mode
            if "custom" in stream_modes:

                def stream_writer(c: Any) -> None:
                    stream.put(
                        (
                            tuple(
                                get_config()[CONF][CONFIG_KEY_CHECKPOINT_NS].split(
                                    NS_SEP
                                )[:-1]
                            ),
                            "custom",
                            c,
                        )
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

            with SyncPregelLoop(
                input,
                stream=StreamProtocol(stream.put, stream_modes),
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
