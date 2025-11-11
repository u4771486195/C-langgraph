            {"my_key": "value ⛰️", "market": "DE"}, thread1, durability="exit"
        )
    ] == [
        {
            "__interrupt__": (
                Interrupt(
                    value="Just because...",
                    id=AnyStr(),
                ),
            )
        },
    ]
    assert [c.metadata async for c in tool_two.checkpointer.alist(thread1root)] == [
        {
            "parents": {},
            "source": "loop",
            "step": 0,
        },
    ]
    tup = await tool_two.checkpointer.aget_tuple(thread1)
    assert await tool_two.aget_state(thread1) == StateSnapshot(
        values={"my_key": "value ⛰️", "market": "DE"},
        next=("tool_two",),
        tasks=(
            PregelTask(
                AnyStr(),
                "tool_two",
                (PULL, "tool_two"),
                interrupts=(
                    Interrupt(
                        value="Just because...",
                        id=AnyStr(),
                    ),
                ),
                state={
                    "configurable": {
                        "thread_id": "1",
                        "checkpoint_ns": AnyStr("tool_two:"),
                    }
                },
            ),
        ),
        config=tup.config,
        created_at=tup.checkpoint["ts"],
        metadata={
            "parents": {},
            "source": "loop",
            "step": 0,
        },
        parent_config=None,
        interrupts=(
            Interrupt(
                value="Just because...",
                id=AnyStr(),
            ),
        ),
    )

    # clear the interrupt and next tasks
    await tool_two.aupdate_state(thread1, None, as_node=END)
    # interrupt is cleared, as well as the next tasks
    tup = await tool_two.checkpointer.aget_tuple(thread1)
    assert await tool_two.aget_state(thread1) == StateSnapshot(
        values={"my_key": "value ⛰️", "market": "DE"},
        next=(),
        tasks=(),
        config=tup.config,
        created_at=tup.checkpoint["ts"],
        metadata={
            "parents": {},
            "source": "update",
            "step": 1,
        },
        parent_config=(
            [c async for c in tool_two.checkpointer.alist(thread1root, limit=2)][
                -1
            ].config
        ),
        interrupts=(),
    )


@NEEDS_CONTEXTVARS
async def test_partial_pending_checkpoint(
    async_checkpointer: BaseCheckpointSaver,
) -> None:
    class State(TypedDict):
        my_key: Annotated[str, operator.add]
        market: str

    def tool_one(s: State) -> State:
        return {"my_key": " one"}

    tool_two_node_count = 0

    def tool_two_node(s: State) -> State:
        nonlocal tool_two_node_count
        tool_two_node_count += 1
        if s["market"] == "DE":
            answer = interrupt("Just because...")
        else:
            answer = " all good"
        return {"my_key": answer}

    def start(state: State) -> list[Send | str]:
        return ["tool_two", Send("tool_one", state)]

    tool_two_graph = StateGraph(State)
    tool_two_graph.add_node("tool_two", tool_two_node, retry_policy=RetryPolicy())
    tool_two_graph.add_node("tool_one", tool_one)
    tool_two_graph.set_conditional_entry_point(start)
    tool_two = tool_two_graph.compile()

    tracer = FakeTracer()
    assert await tool_two.ainvoke(
        {"my_key": "value", "market": "DE"}, {"callbacks": [tracer]}, debug=True
    ) == {
        "my_key": "value one",
        "market": "DE",
        "__interrupt__": [Interrupt(value="Just because...", id=AnyStr())],
    }
    assert tool_two_node_count == 1, "interrupts aren't retried"
    assert len(tracer.runs) == 1
    run = tracer.runs[0]
    assert run.end_time is not None
    assert run.error is None
    assert run.outputs == {"market": "DE", "my_key": "value one"}

    assert await tool_two.ainvoke({"my_key": "value", "market": "US"}) == {
        "my_key": "value all good one",
        "market": "US",
    }

    tool_two = tool_two_graph.compile(checkpointer=async_checkpointer)

    # missing thread_id
    with pytest.raises(ValueError, match="thread_id"):
        await tool_two.ainvoke({"my_key": "value", "market": "DE"})

    # flow: interrupt -> resume with answer
    thread2 = {"configurable": {"thread_id": "2"}}
    # stop when about to enter node
    assert [
        c
        async for c in tool_two.astream({"my_key": "value ⛰️", "market": "DE"}, thread2)
    ] == UnsortedSequence(
        {
            "__interrupt__": (
                Interrupt(
                    value="Just because...",
                    id=AnyStr(),
                ),
            )
        },
        {
            "tool_one": {"my_key": " one"},
        },
    )
    # resume with answer
    assert [
        c async for c in tool_two.astream(Command(resume=" my answer"), thread2)
    ] == [
        {
            "__metadata__": {"cached": True},
            "tool_one": {"my_key": " one"},
        },
        {"tool_two": {"my_key": " my answer"}},
    ]

    # flow: interrupt -> clear tasks
    thread1 = {"configurable": {"thread_id": "1"}}
    # stop when about to enter node
    assert await tool_two.ainvoke(
        {"my_key": "value ⛰️", "market": "DE"}, thread1, durability="exit"
    ) == {
        "my_key": "value ⛰️ one",
        "market": "DE",
        "__interrupt__": [
            Interrupt(
                value="Just because...",
                id=AnyStr(),
            )
        ],
    }

    assert [c.metadata async for c in tool_two.checkpointer.alist(thread1)] == [
        {
            "parents": {},
            "source": "loop",
            "step": 0,
        },
    ]

    tup = await tool_two.checkpointer.aget_tuple(thread1)
    assert await tool_two.aget_state(thread1) == StateSnapshot(
        values={"my_key": "value ⛰️ one", "market": "DE"},
        next=("tool_two",),
        tasks=(
            PregelTask(
                AnyStr(),
                name="tool_one",
                path=("__pregel_push", 0, False),
                error=None,
                interrupts=(),
                state=None,
                result={"my_key": " one"},
            ),
            PregelTask(
                AnyStr(),
                "tool_two",
                (PULL, "tool_two"),
                interrupts=(
                    Interrupt(
                        value="Just because...",
                        id=AnyStr(),
                    ),
                ),
            ),
        ),
        config=tup.config,
        created_at=tup.checkpoint["ts"],
        metadata={
            "parents": {},
            "source": "loop",
            "step": 0,
        },
        parent_config=None,
        interrupts=(
            Interrupt(
                value="Just because...",
                id=AnyStr(),
            ),
        ),
    )

    # clear the interrupt and next tasks
    await tool_two.aupdate_state(thread1, None, as_node=END)
    # interrupt and next tasks are cleared, finished tasks are kept
    tup_upd = await tool_two.checkpointer.aget_tuple(thread1)
    assert await tool_two.aget_state(thread1) == StateSnapshot(
        values={"my_key": "value ⛰️ one", "market": "DE"},
        next=(),
        tasks=(),
        config=tup_upd.config,
        created_at=tup_upd.checkpoint["ts"],
        metadata={
            "parents": {},
            "source": "update",
            "step": 1,
        },
        parent_config=tup.config,
        interrupts=(),
    )


@NEEDS_CONTEXTVARS
async def test_node_not_cancelled_on_other_node_interrupted(
    async_checkpointer: BaseCheckpointSaver,
) -> None:
    class State(TypedDict):
        hello: Annotated[str, operator.add]

    awhiles = 0
    inner_task_cancelled = False

    async def awhile(input: State) -> None:
        nonlocal awhiles

        awhiles += 1
        try:
            await asyncio.sleep(1)
            return {"hello": " again"}
        except asyncio.CancelledError:
            nonlocal inner_task_cancelled
            inner_task_cancelled = True
            raise

    async def iambad(input: State) -> None:
        return {"hello": interrupt("I am bad")}

    builder = StateGraph(State)
    builder.add_node("agent", awhile)
    builder.add_node("bad", iambad)
    builder.set_conditional_entry_point(lambda _: ["agent", "bad"])

    graph = builder.compile(checkpointer=async_checkpointer)
    thread = {"configurable": {"thread_id": "1"}}

    # writes from "awhile" are applied to last chunk
    assert await graph.ainvoke({"hello": "world"}, thread) == {
        "hello": "world again",
        "__interrupt__": [
            Interrupt(
                value="I am bad",
                id=AnyStr(),
            )
        ],
    }

    assert not inner_task_cancelled
    assert awhiles == 1

    assert await graph.ainvoke(None, thread, debug=True) == {
        "hello": "world again",
        "__interrupt__": [
            Interrupt(
                value="I am bad",
                id=AnyStr(),
            )
        ],
    }

    assert not inner_task_cancelled
    assert awhiles == 1

    # resume with answer
    assert await graph.ainvoke(Command(resume=" okay"), thread) == {
        "hello": "world again okay"
    }

    assert not inner_task_cancelled
    assert awhiles == 1


@pytest.mark.parametrize("stream_hang_s", [0.3, 0.6])
async def test_step_timeout_on_stream_hang(stream_hang_s: float) -> None:
    inner_task_cancelled = False

    async def awhile(input: Any) -> None:
        try:
            await asyncio.sleep(1.5)
        except asyncio.CancelledError:
            nonlocal inner_task_cancelled
            inner_task_cancelled = True
            raise

    async def alittlewhile(input: Any) -> None:
        await asyncio.sleep(0.6)
        return {"hello": "1"}

    class State(TypedDict):
        hello: str

    builder = StateGraph(State)
    builder.add_node(awhile)
    builder.add_node(alittlewhile)
    builder.set_conditional_entry_point(lambda _: ["awhile", "alittlewhile"])
    graph = builder.compile()
    graph.step_timeout = 1

    with pytest.raises(asyncio.TimeoutError):
        async for chunk in graph.astream({"hello": "world"}, stream_mode="updates"):
            assert chunk == {"alittlewhile": {"hello": "1"}}
            await asyncio.sleep(stream_hang_s)

    assert inner_task_cancelled


async def test_cancel_graph_astream(async_checkpointer: BaseCheckpointSaver) -> None:
    class State(TypedDict):
        value: Annotated[int, operator.add]

    class AwhileMaker:
        def __init__(self) -> None:
            self.reset()

        async def __call__(self, input: State) -> Any:
            self.started = True
            try:
                await asyncio.sleep(1.5)
            except asyncio.CancelledError:
                self.cancelled = True
                raise

        def reset(self):
            self.started = False
            self.cancelled = False

    async def alittlewhile(input: State) -> None:
        await asyncio.sleep(0.6)
        return {"value": 2}

    awhile = AwhileMaker()
    aparallelwhile = AwhileMaker()
    builder = StateGraph(State)
    builder.add_node("awhile", awhile)
    builder.add_node("aparallelwhile", aparallelwhile)
    builder.add_node(alittlewhile)
    builder.add_edge(START, "alittlewhile")
    builder.add_edge(START, "aparallelwhile")
    builder.add_edge("alittlewhile", "awhile")

    graph = builder.compile(checkpointer=async_checkpointer)

    # test interrupting astream
    got_event = False
    thread1: RunnableConfig = {"configurable": {"thread_id": "1"}}
    async with aclosing(graph.astream({"value": 1}, thread1)) as stream:
        async for chunk in stream:
            assert chunk == {"alittlewhile": {"value": 2}}
            got_event = True
            break

    assert got_event

    # node aparallelwhile should start, but be cancelled
    assert aparallelwhile.started is True
    assert aparallelwhile.cancelled is True

    # node "awhile" should never start
    assert awhile.started is False

    # checkpoint with output of "alittlewhile" should not be saved
    # but we should have applied pending writes
    state = await graph.aget_state(thread1)
    assert state is not None
    assert state.values == {"value": 3}  # 1 + 2
    assert state.next == ("aparallelwhile",)
    assert state.metadata == {
        "parents": {},
        "source": "loop",
        "step": 0,
    }


async def test_cancel_graph_astream_events_v2(
    async_checkpointer: BaseCheckpointSaver,
) -> None:
    class State(TypedDict):
        value: int

    class AwhileMaker:
        def __init__(self) -> None:
            self.reset()

        async def __call__(self, input: State) -> Any:
            self.started = True
            try:
                await asyncio.sleep(1.5)
            except asyncio.CancelledError:
                self.cancelled = True
                raise

        def reset(self):
            self.started = False
            self.cancelled = False

    async def alittlewhile(input: State) -> None:
        await asyncio.sleep(0.6)
        return {"value": 2}

    awhile = AwhileMaker()
    anotherwhile = AwhileMaker()
    builder = StateGraph(State)
    builder.add_node(alittlewhile)
    builder.add_node("awhile", awhile)
    builder.add_node("anotherwhile", anotherwhile)
    builder.add_edge(START, "alittlewhile")
    builder.add_edge("alittlewhile", "awhile")
    builder.add_edge("awhile", "anotherwhile")

    graph = builder.compile(checkpointer=async_checkpointer)

    # test interrupting astream_events v2
    got_event = False
    thread2: RunnableConfig = {"configurable": {"thread_id": "2"}}
    async with aclosing(
        graph.astream_events({"value": 1}, thread2, version="v2")
    ) as stream:
        async for chunk in stream:
            if chunk["event"] == "on_chain_stream" and not chunk["parent_ids"]:
                got_event = True
                assert chunk["data"]["chunk"] == {"alittlewhile": {"value": 2}}
                await asyncio.sleep(0.1)
                break

    # did break
    assert got_event

    # node "awhile" maybe starts (impl detail of astream_events)
    # if it does start, it must be cancelled
    if awhile.started:
        assert awhile.cancelled is True

    # node "anotherwhile" should never start
    assert anotherwhile.started is False

    # checkpoint with output of "alittlewhile" should not be saved
    state = await graph.aget_state(thread2)
    assert state is not None
    assert state.values == {"value": 2}
    assert state.next == ("awhile",)
    assert state.metadata == {
        "parents": {},
        "source": "loop",
        "step": 1,
    }


async def test_node_schemas_custom_output() -> None:
    class State(TypedDict):
        hello: str
        bye: str
        messages: Annotated[list[str], add_messages]

    class Output(TypedDict):
        messages: list[str]

    class StateForA(TypedDict):
        hello: str
        messages: Annotated[list[str], add_messages]

    async def node_a(state: StateForA):
        assert state == {
            "hello": "there",
            "messages": [_AnyIdHumanMessage(content="hello")],
        }

    class StateForB(TypedDict):
        bye: str
        now: int

    async def node_b(state: StateForB):
        assert state == {
            "bye": "world",
        }
        return {
            "now": 123,
            "hello": "again",
        }

    class StateForC(TypedDict):
        hello: str
        now: int

    async def node_c(state: StateForC):
        assert state == {
            "hello": "again",
            "now": 123,
        }

    builder = StateGraph(State, output_schema=Output)
    builder.add_node("a", node_a)
    builder.add_node("b", node_b)
    builder.add_node("c", node_c)
    builder.add_edge(START, "a")
    builder.add_edge("a", "b")
    builder.add_edge("b", "c")
    graph = builder.compile()

    assert await graph.ainvoke(
        {"hello": "there", "bye": "world", "messages": "hello"}
    ) == {
        "messages": [_AnyIdHumanMessage(content="hello")],
    }

    builder = StateGraph(State, output_schema=Output)
    builder.add_node("a", node_a)
    builder.add_node("b", node_b)
    builder.add_node("c", node_c)
    builder.add_edge(START, "a")
    builder.add_edge("a", "b")
    builder.add_edge("b", "c")
    graph = builder.compile()

    assert await graph.ainvoke(
        {
            "hello": "there",
            "bye": "world",
            "messages": "hello",
            "now": 345,  # ignored because not in input schema
        }
    ) == {
        "messages": [_AnyIdHumanMessage(content="hello")],
    }

    assert [
        c
        async for c in graph.astream(
            {
                "hello": "there",
                "bye": "world",
                "messages": "hello",
                "now": 345,  # ignored because not in input schema
            }
        )
    ] == [
        {"a": None},
        {"b": {"hello": "again", "now": 123}},
        {"c": None},
    ]


async def test_invoke_single_process_in_out(mocker: MockerFixture) -> None:
    add_one = mocker.Mock(side_effect=lambda x: x + 1)
    chain = NodeBuilder().subscribe_only("input").do(add_one).write_to("output")

    app = Pregel(
        nodes={
            "one": chain,
        },
        channels={
            "input": LastValue(int),
            "output": LastValue(int),
        },
        input_channels="input",
        output_channels="output",
    )

    assert app.input_schema.model_json_schema() == {
        "title": "LangGraphInput",
        "type": "integer",
    }
    assert app.output_schema.model_json_schema() == {
        "title": "LangGraphOutput",
        "type": "integer",
    }
    assert await app.ainvoke(2) == 3
    assert await app.ainvoke(2, output_keys=["output"]) == {"output": 3}


async def test_invoke_single_process_in_write_kwargs(mocker: MockerFixture) -> None:
    add_one = mocker.Mock(side_effect=lambda x: x + 1)
    chain = (
        NodeBuilder()
        .subscribe_only("input")
        .do(add_one)
        .write_to("output", fixed=5, output_plus_one=lambda x: x + 1)
    )

    app = Pregel(
        nodes={"one": chain},
        channels={
            "input": LastValue(int),
            "output": LastValue(int),
            "fixed": LastValue(int),
            "output_plus_one": LastValue(int),
        },
        output_channels=["output", "fixed", "output_plus_one"],
        input_channels="input",
    )

    assert app.input_schema.model_json_schema() == {
        "title": "LangGraphInput",
        "type": "integer",
    }
    assert app.output_schema.model_json_schema() == {
        "title": "LangGraphOutput",
        "type": "object",
        "properties": {
            "output": {"title": "Output", "type": "integer", "default": None},
            "fixed": {"title": "Fixed", "type": "integer", "default": None},
            "output_plus_one": {
                "title": "Output Plus One",
                "type": "integer",
                "default": None,
            },
        },
    }
    assert await app.ainvoke(2) == {"output": 3, "fixed": 5, "output_plus_one": 4}


async def test_invoke_single_process_in_out_dict(mocker: MockerFixture) -> None:
    add_one = mocker.Mock(side_effect=lambda x: x + 1)
    chain = NodeBuilder().subscribe_only("input").do(add_one).write_to("output")

    app = Pregel(
        nodes={"one": chain},
        channels={"input": LastValue(int), "output": LastValue(int)},
        input_channels="input",
        output_channels=["output"],
    )

    assert app.input_schema.model_json_schema() == {
        "title": "LangGraphInput",
        "type": "integer",
    }
    assert app.output_schema.model_json_schema() == {
        "title": "LangGraphOutput",
        "type": "object",
        "properties": {
            "output": {"title": "Output", "type": "integer", "default": None}
        },
    }
    assert await app.ainvoke(2) == {"output": 3}


async def test_invoke_single_process_in_dict_out_dict(mocker: MockerFixture) -> None:
    add_one = mocker.Mock(side_effect=lambda x: x + 1)
    chain = NodeBuilder().subscribe_only("input").do(add_one).write_to("output")

    app = Pregel(
        nodes={"one": chain},
        channels={"input": LastValue(int), "output": LastValue(int)},
        input_channels=["input"],
        output_channels=["output"],
    )

    assert app.input_schema.model_json_schema() == {
        "title": "LangGraphInput",
        "type": "object",
        "properties": {"input": {"title": "Input", "type": "integer", "default": None}},
    }
    assert app.output_schema.model_json_schema() == {
        "title": "LangGraphOutput",
        "type": "object",
        "properties": {
            "output": {"title": "Output", "type": "integer", "default": None}
        },
    }
    assert await app.ainvoke({"input": 2}) == {"output": 3}


async def test_invoke_two_processes_in_out(mocker: MockerFixture) -> None:
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
        stream_channels=["inbox", "output"],
    )

    assert await app.ainvoke(2) == 4

    with pytest.raises(GraphRecursionError):
        await app.ainvoke(2, {"recursion_limit": 1})

    step = 0
    async for values in app.astream(2):
        step += 1
        if step == 1:
            assert values == {
                "inbox": 3,
            }
        elif step == 2:
            assert values == {
                "inbox": 3,
                "output": 4,
            }
    assert step == 2


async def test_batch_two_processes_in_out() -> None:
    async def add_one_with_delay(inp: int) -> int:
        await asyncio.sleep(inp / 10)
        return inp + 1

    one = NodeBuilder().subscribe_only("input").do(add_one_with_delay).write_to("one")
    two = NodeBuilder().subscribe_only("one").do(add_one_with_delay).write_to("output")

    app = Pregel(
        nodes={"one": one, "two": two},
        channels={
            "one": LastValue(int),
            "output": LastValue(int),
            "input": LastValue(int),
        },
        input_channels="input",
        output_channels="output",
    )

    assert await app.abatch([3, 2, 1, 3, 5]) == [5, 4, 3, 5, 7]
    assert await app.abatch([3, 2, 1, 3, 5], output_keys=["output"]) == [
        {"output": 5},
        {"output": 4},
        {"output": 3},
        {"output": 5},
        {"output": 7},
    ]


async def test_invoke_many_processes_in_out(mocker: MockerFixture) -> None:
    test_size = 100
    add_one = mocker.Mock(side_effect=lambda x: x + 1)

    nodes = {"-1": NodeBuilder().subscribe_only("input").do(add_one).write_to("-1")}
    for i in range(test_size - 2):
        nodes[str(i)] = (
            NodeBuilder().subscribe_only(str(i - 1)).do(add_one).write_to(str(i))
        )
    nodes["last"] = NodeBuilder().subscribe_only(str(i)).do(add_one).write_to("output")

    app = Pregel(
        nodes=nodes,
        channels={str(i): LastValue(int) for i in range(-1, test_size - 2)}
        | {"input": LastValue(int), "output": LastValue(int)},
        input_channels="input",
        output_channels="output",
    )

    # No state is left over from previous invocations
    for _ in range(10):
        assert await app.ainvoke(2, {"recursion_limit": test_size}) == 2 + test_size

    # Concurrent invocations do not interfere with each other
    assert await asyncio.gather(
        *(app.ainvoke(2, {"recursion_limit": test_size}) for _ in range(10))
    ) == [2 + test_size for _ in range(10)]


async def test_batch_many_processes_in_out(mocker: MockerFixture) -> None:
    test_size = 100
    add_one = mocker.Mock(side_effect=lambda x: x + 1)

    nodes = {"-1": NodeBuilder().subscribe_only("input").do(add_one).write_to("-1")}
    for i in range(test_size - 2):
        nodes[str(i)] = (
            NodeBuilder().subscribe_only(str(i - 1)).do(add_one).write_to(str(i))
        )
    nodes["last"] = NodeBuilder().subscribe_only(str(i)).do(add_one).write_to("output")

    app = Pregel(
        nodes=nodes,
        channels={str(i): LastValue(int) for i in range(-1, test_size - 2)}
        | {"input": LastValue(int), "output": LastValue(int)},
        input_channels="input",
        output_channels="output",
    )

    # No state is left over from previous invocations
    for _ in range(3):
        # Then invoke pubsub
        assert await app.abatch([2, 1, 3, 4, 5], {"recursion_limit": test_size}) == [
            2 + test_size,
            1 + test_size,
            3 + test_size,
            4 + test_size,
            5 + test_size,
        ]

    # Concurrent invocations do not interfere with each other
    assert await asyncio.gather(
        *(app.abatch([2, 1, 3, 4, 5], {"recursion_limit": test_size}) for _ in range(3))
    ) == [
        [2 + test_size, 1 + test_size, 3 + test_size, 4 + test_size, 5 + test_size]
        for _ in range(3)
    ]


async def test_invoke_two_processes_two_in_two_out_invalid(
    mocker: MockerFixture,
) -> None:
    add_one = mocker.Mock(side_effect=lambda x: x + 1)

    one = NodeBuilder().subscribe_only("input").do(add_one).write_to("output")
    two = NodeBuilder().subscribe_only("input").do(add_one).write_to("output")

    app = Pregel(
        nodes={"one": one, "two": two},
        channels={"output": LastValue(int), "input": LastValue(int)},
        input_channels="input",
        output_channels="output",
    )

    with pytest.raises(InvalidUpdateError):
        # LastValue channels can only be updated once per iteration
        await app.ainvoke(2)


async def test_invoke_two_processes_two_in_two_out_valid(mocker: MockerFixture) -> None:
    add_one = mocker.Mock(side_effect=lambda x: x + 1)

    one = NodeBuilder().subscribe_only("input").do(add_one).write_to("output")
    two = NodeBuilder().subscribe_only("input").do(add_one).write_to("output")

    app = Pregel(
        nodes={"one": one, "two": two},
        channels={
            "input": LastValue(int),
            "output": Topic(int),
        },
        input_channels="input",
        output_channels="output",
    )

    # An Topic channel accumulates updates into a sequence
    assert await app.ainvoke(2) == [3, 3]


async def test_invoke_checkpoint(
    mocker: MockerFixture, async_checkpointer: BaseCheckpointSaver
) -> None:
    add_one = mocker.Mock(side_effect=lambda x: x["total"] + x["input"])
    errored_once = False

    def raise_if_above_10(input: int) -> int:
        nonlocal errored_once
        if input > 4:
            if errored_once:
                pass
            else:
                errored_once = True
                raise ConnectionError("I will be retried")
        if input > 10:
            raise ValueError("Input is too large")
        return input

    one = (
        NodeBuilder()
        .subscribe_to("input")
        .read_from("total")
        .do(add_one)
        .write_to("output", "total")
        .do(raise_if_above_10)
    )

    app = Pregel(
        nodes={"one": one},
        channels={
            "total": BinaryOperatorAggregate(int, operator.add),
            "input": LastValue(int),
            "output": LastValue(int),
        },
        input_channels="input",
        output_channels="output",
        checkpointer=async_checkpointer,
        retry_policy=RetryPolicy(),
    )

    # total starts out as 0, so output is 0+2=2
    assert await app.ainvoke(2, {"configurable": {"thread_id": "1"}}) == 2
    checkpoint = await async_checkpointer.aget({"configurable": {"thread_id": "1"}})
    assert checkpoint is not None
    assert checkpoint["channel_values"].get("total") == 2
    # total is now 2, so output is 2+3=5
    assert await app.ainvoke(3, {"configurable": {"thread_id": "1"}}) == 5
    assert errored_once, "errored and retried"
    checkpoint = await async_checkpointer.aget({"configurable": {"thread_id": "1"}})
    assert checkpoint is not None
    assert checkpoint["channel_values"].get("total") == 7
    # total is now 2+5=7, so output would be 7+4=11, but raises ValueError
    with pytest.raises(ValueError):
        await app.ainvoke(4, {"configurable": {"thread_id": "1"}})
    # checkpoint is not updated
    checkpoint = await async_checkpointer.aget({"configurable": {"thread_id": "1"}})
    assert checkpoint is not None
    assert checkpoint["channel_values"].get("total") == 7
    # on a new thread, total starts out as 0, so output is 0+5=5
    assert await app.ainvoke(5, {"configurable": {"thread_id": "2"}}) == 5
    checkpoint = await async_checkpointer.aget({"configurable": {"thread_id": "1"}})
    assert checkpoint is not None
    assert checkpoint["channel_values"].get("total") == 7
    checkpoint = await async_checkpointer.aget({"configurable": {"thread_id": "2"}})
    assert checkpoint is not None
    assert checkpoint["channel_values"].get("total") == 5


async def test_pending_writes_resume(
    async_checkpointer: BaseCheckpointSaver, durability: Durability
) -> None:
    class State(TypedDict):
        value: Annotated[int, operator.add]

    class AwhileMaker:
        def __init__(self, sleep: float, rtn: dict | Exception) -> None:
            self.sleep = sleep
            self.rtn = rtn
            self.reset()

        async def __call__(self, input: State) -> Any:
            self.calls += 1
            await asyncio.sleep(self.sleep)
            if isinstance(self.rtn, Exception):
                raise self.rtn
            else:
                return self.rtn

        def reset(self):
            self.calls = 0

    one = AwhileMaker(0.1, {"value": 2})
    two = AwhileMaker(0.2, ConnectionError("I'm not good"))
    builder = StateGraph(State)
    builder.add_node("one", one)
    builder.add_node(
        "two",
        two,
        retry_policy=RetryPolicy(max_attempts=2, initial_interval=0, jitter=False),
    )
    builder.add_edge(START, "one")
    builder.add_edge(START, "two")
    graph = builder.compile(checkpointer=async_checkpointer)

    thread1: RunnableConfig = {"configurable": {"thread_id": "1"}}
    with pytest.raises(ConnectionError, match="I'm not good"):
        await graph.ainvoke({"value": 1}, thread1, durability=durability)

    # both nodes should have been called once
    assert one.calls == 1
    assert two.calls == 2

    # latest checkpoint should be before nodes "one", "two"
    # but we should have applied pending writes from "one"
    state = await graph.aget_state(thread1)
    assert state is not None
    assert state.values == {"value": 3}
    assert state.next == ("two",)
    assert state.tasks == (
        PregelTask(AnyStr(), "one", (PULL, "one"), result={"value": 2}),
        PregelTask(
            AnyStr(),
            "two",
            (PULL, "two"),
            'ConnectionError("I\'m not good")',
        ),
    )
    assert state.metadata == {
        "parents": {},
        "source": "loop",
        "step": 0,
    }
    # get_state with checkpoint_id should not apply any pending writes
    state = await graph.aget_state(state.config)
    assert state is not None
    assert state.values == {"value": 1}
    assert state.next == ("one", "two")
    # should contain pending write of "one"
    checkpoint = await async_checkpointer.aget_tuple(thread1)
    assert checkpoint is not None
    # should contain error from "two"
    expected_writes = [
        (AnyStr(), "value", 2),
        (AnyStr(), ERROR, 'ConnectionError("I\'m not good")'),
    ]
    assert len(checkpoint.pending_writes) == 2
    assert all(w in expected_writes for w in checkpoint.pending_writes)
    # both non-error pending writes come from same task
    non_error_writes = [w for w in checkpoint.pending_writes if w[1] != ERROR]
    # error write is from the other task
    error_write = next(w for w in checkpoint.pending_writes if w[1] == ERROR)
    assert error_write[0] != non_error_writes[0][0]

    # resume execution
    with pytest.raises(ConnectionError, match="I'm not good"):
        await graph.ainvoke(None, thread1, durability=durability)

    # node "one" succeeded previously, so shouldn't be called again
    assert one.calls == 1
    # node "two" should have been called once again
    assert two.calls == 4

    # confirm no new checkpoints saved
    state_two = await graph.aget_state(thread1)
    assert state_two.metadata == state.metadata

    # resume execution, without exception
    two.rtn = {"value": 3}
    # both the pending write and the new write were applied, 1 + 2 + 3 = 6
    assert await graph.ainvoke(None, thread1, durability=durability) == {"value": 6}

    # check all final checkpoints
    checkpoints = [c async for c in async_checkpointer.alist(thread1)]
    # we should have 3
    assert len(checkpoints) == (3 if durability != "exit" else 2)
    # the last one not too interesting for this test
    assert checkpoints[0] == CheckpointTuple(
        config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        checkpoint={
            "v": 4,
            "id": AnyStr(),
            "ts": AnyStr(),
            "versions_seen": {
                "one": {
                    "branch:to:one": AnyVersion(),
                },
                "two": {
                    "branch:to:two": AnyVersion(),
                },
                "__input__": {},
                "__start__": {
                    "__start__": AnyVersion(),
                },
                "__interrupt__": {
                    "value": AnyVersion(),
                    "__start__": AnyVersion(),
                    "branch:to:one": AnyVersion(),
                    "branch:to:two": AnyVersion(),
                },
            },
            "channel_versions": {
                "value": AnyVersion(),
                "__start__": AnyVersion(),
                "branch:to:one": AnyVersion(),
                "branch:to:two": AnyVersion(),
            },
            "channel_values": {"value": 6},
            "updated_channels": ["value"],
        },
        metadata={
            "parents": {},
            "step": 1,
            "source": "loop",
        },
        parent_config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": checkpoints[1].config["configurable"]["checkpoint_id"],
            }
        },
        pending_writes=[],
    )
    # the previous one we assert that pending writes contains both
    # - original error
    # - successful writes from resuming after preventing error
    assert checkpoints[1] == CheckpointTuple(
        config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        checkpoint={
            "v": 4,
            "id": AnyStr(),
            "ts": AnyStr(),
            "versions_seen": {
                "__input__": {},
                "__start__": {
                    "__start__": AnyVersion(),
                },
            },
            "channel_versions": {
                "value": AnyVersion(),
                "__start__": AnyVersion(),
                "branch:to:one": AnyVersion(),
                "branch:to:two": AnyVersion(),
            },
            "channel_values": {
                "value": 1,
                "branch:to:one": None,
                "branch:to:two": None,
            },
            "updated_channels": ["branch:to:one", "branch:to:two", "value"],
        },
        metadata={
            "parents": {},
            "step": 0,
            "source": "loop",
        },
        parent_config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": checkpoints[2].config["configurable"]["checkpoint_id"],
            }
        }
        if durability != "exit"
        else None,
        pending_writes=UnsortedSequence(
            (AnyStr(), "value", 2),
            (AnyStr(), "__error__", 'ConnectionError("I\'m not good")'),
            (AnyStr(), "value", 3),
        )
        if durability != "exit"
        else UnsortedSequence(
            (AnyStr(), "value", 2),
            (AnyStr(), "__error__", 'ConnectionError("I\'m not good")'),
            # the write against the previous checkpoint is not saved, as it is
            # produced in a run where only the next checkpoint (the last) is saved
        ),
    )
    if durability == "exit":
        return
    assert checkpoints[2] == CheckpointTuple(
        config={
            "configurable": {
                "thread_id": "1",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        checkpoint={
            "v": 4,
            "id": AnyStr(),
            "ts": AnyStr(),
            "versions_seen": {"__input__": {}},
            "channel_versions": {
                "__start__": AnyVersion(),
            },
            "channel_values": {"__start__": {"value": 1}},
            "updated_channels": ["__start__"],
        },
        metadata={
            "parents": {},
            "step": -1,
            "source": "input",
        },
        parent_config=None,
        pending_writes=UnsortedSequence(
            (AnyStr(), "value", 1),
            (AnyStr(), "branch:to:one", None),
            (AnyStr(), "branch:to:two", None),
        ),
    )


async def test_run_from_checkpoint_id_retains_previous_writes(
    async_checkpointer: BaseCheckpointSaver,
) -> None:
    class MyState(TypedDict):
        myval: Annotated[int, operator.add]
        otherval: bool

    class Anode:
        def __init__(self):
            self.switch = False

        async def __call__(self, state: MyState):
            self.switch = not self.switch
            return {"myval": 2 if self.switch else 1, "otherval": self.switch}

    builder = StateGraph(MyState)
    thenode = Anode()  # Fun.
    builder.add_node("node_one", thenode)
    builder.add_node("node_two", thenode)
    builder.add_edge(START, "node_one")

    def _getedge(src: str):
        swap = "node_one" if src == "node_two" else "node_two"

        def _edge(st: MyState) -> Literal["__end__", "node_one", "node_two"]:
            if st["myval"] > 3:
                return END
            if st["otherval"]:
                return swap
            return src

        return _edge

    builder.add_conditional_edges("node_one", _getedge("node_one"))
    builder.add_conditional_edges("node_two", _getedge("node_two"))
    graph = builder.compile(checkpointer=async_checkpointer)

    thread_id = uuid.uuid4()
    thread1 = {"configurable": {"thread_id": str(thread_id)}}

    result = await graph.ainvoke({"myval": 1}, thread1, durability="async")
    assert result["myval"] == 4
    history = [c async for c in graph.aget_state_history(thread1)]

    assert len(history) == 4
    assert history[0].values == {"myval": 4, "otherval": False}
    assert history[-1].values == {"myval": 0}

    second_run_config = {
        **thread1,
        "configurable": {
            **thread1["configurable"],
            "checkpoint_id": history[1].config["configurable"]["checkpoint_id"],
        },
    }
    second_result = await graph.ainvoke(None, second_run_config)
    assert second_result == {"myval": 5, "otherval": True}

    new_history = [
        c
        async for c in graph.aget_state_history(
            {"configurable": {"thread_id": str(thread_id), "checkpoint_ns": ""}}
        )
    ]

    assert len(new_history) == len(history) + 1
    for original, new in zip(history, new_history[1:]):
        assert original.values == new.values
        assert original.next == new.next
        assert original.metadata["step"] == new.metadata["step"]

    def _get_tasks(hist: list, start: int):
        return [h.tasks for h in hist[start:]]

    assert _get_tasks(new_history, 1) == _get_tasks(history, 0)


async def test_cond_edge_after_send() -> None:
    class Node:
        def __init__(self, name: str):
            self.name = name
            setattr(self, "__name__", name)

        async def __call__(self, state):
            return [self.name]

    async def send_for_fun(state):
        return [Send("2", state), Send("2", state)]

    async def route_to_three(state) -> Literal["3"]:
        return "3"

    builder = StateGraph(Annotated[list, operator.add])
    builder.add_node(Node("1"))
    builder.add_node(Node("2"))
    builder.add_node(Node("3"))
    builder.add_edge(START, "1")
    builder.add_conditional_edges("1", send_for_fun)
    builder.add_conditional_edges("2", route_to_three)
    graph = builder.compile()

    assert await graph.ainvoke(["0"]) == ["0", "1", "2", "2", "3"]


async def test_concurrent_emit_sends() -> None:
    class Node:
        def __init__(self, name: str):
            self.name = name
            setattr(self, "__name__", name)

        async def __call__(self, state):
            return (
                [self.name]
                if isinstance(state, list)
                else ["|".join((self.name, str(state)))]
            )

    async def send_for_fun(state):
        return [Send("2", 1), Send("2", 2), "3.1"]

    async def send_for_profit(state):
        return [Send("2", 3), Send("2", 4)]

    async def route_to_three(state) -> Literal["3"]:
        return "3"

    builder = StateGraph(Annotated[list, operator.add])
    builder.add_node(Node("1"))
    builder.add_node(Node("1.1"))
    builder.add_node(Node("2"))
    builder.add_node(Node("3"))
    builder.add_node(Node("3.1"))
    builder.add_edge(START, "1")
    builder.add_edge(START, "1.1")
    builder.add_conditional_edges("1", send_for_fun)
    builder.add_conditional_edges("1.1", send_for_profit)
    builder.add_conditional_edges("2", route_to_three)
    graph = builder.compile()
    assert await graph.ainvoke(["0"]) == (
        [
            "0",
            "1",
            "1.1",
            "3.1",
            "2|1",
            "2|2",
            "2|3",
            "2|4",
            "3",
        ]
    )


async def test_send_sequences(async_checkpointer: BaseCheckpointSaver) -> None:
    class Node:
        def __init__(self, name: str):
            self.name = name
            setattr(self, "__name__", name)

        async def __call__(self, state):
            update = (
                [self.name]
                if isinstance(state, list)  # or isinstance(state, Control)
                else ["|".join((self.name, str(state)))]
            )
            if isinstance(state, Command):
                return replace(state, update=update)
            else:
                return update

    async def send_for_fun(state):
        return [
            Send("2", Command(goto=Send("2", 3))),
            Send("2", Command(goto=Send("2", 4))),
            "3.1",
        ]

    async def route_to_three(state) -> Literal["3"]:
        return "3"

    builder = StateGraph(Annotated[list, operator.add])
    builder.add_node(Node("1"))
    builder.add_node(Node("2"))
    builder.add_node(Node("3"))
    builder.add_node(Node("3.1"))
    builder.add_edge(START, "1")
    builder.add_conditional_edges("1", send_for_fun)
    builder.add_conditional_edges("2", route_to_three)
    graph = builder.compile()
    assert await graph.ainvoke(["0"]) == [
        "0",
        "1",
        "3.1",
        "2|Command(goto=Send(node='2', arg=3))",
        "2|Command(goto=Send(node='2', arg=4))",
        "3",
        "2|3",
        "2|4",
        "3",
    ]

    graph = builder.compile(checkpointer=async_checkpointer, interrupt_before=["3.1"])
    thread1 = {"configurable": {"thread_id": "1"}}
    assert await graph.ainvoke(["0"], thread1) == [
        "0",
        "1",
    ]
    assert await graph.ainvoke(None, thread1) == [
        "0",
        "1",
        "3.1",
        "2|Command(goto=Send(node='2', arg=3))",
        "2|Command(goto=Send(node='2', arg=4))",
        "3",
        "2|3",
        "2|4",
        "3",
    ]


@NEEDS_CONTEXTVARS
async def test_imp_task(
    async_checkpointer: BaseCheckpointSaver, durability: Durability
) -> None:
    mapper_calls = 0

    @task()
    async def mapper(input: int) -> str:
        nonlocal mapper_calls
        mapper_calls += 1
        await asyncio.sleep(0.1 * input)
        return str(input) * 2

    @entrypoint(checkpointer=async_checkpointer)
    async def graph(input: list[int]) -> list[str]:
        futures = [mapper(i) for i in input]
        mapped = await asyncio.gather(*futures)
        answer = interrupt("question")
        return [m + answer for m in mapped]

    tracer = FakeTracer()
    thread1 = {"configurable": {"thread_id": "1"}, "callbacks": [tracer]}
    assert [c async for c in graph.astream([0, 1], thread1, durability=durability)] == [
        {"mapper": "00"},
        {"mapper": "11"},
        {
            "__interrupt__": (
                Interrupt(
                    value="question",
                    id=AnyStr(),
                ),
            )
        },
    ]
    assert mapper_calls == 2
    assert len(tracer.runs) == 1
    assert len(tracer.runs[0].child_runs) == 1
    entrypoint_run = tracer.runs[0].child_runs[0]
    assert entrypoint_run.name == "graph"
    mapper_runs = [r for r in entrypoint_run.child_runs if r.name == "mapper"]
    assert len(mapper_runs) == 2
    assert any(r.inputs == {"input": 0} for r in mapper_runs)
    assert any(r.inputs == {"input": 1} for r in mapper_runs)

    assert await graph.ainvoke(
        Command(resume="answer"), thread1, durability=durability
    ) == [
        "00answer",
        "11answer",
    ]
    assert mapper_calls == 2


@NEEDS_CONTEXTVARS
async def test_imp_nested(
    async_checkpointer: BaseCheckpointSaver, durability: Durability
) -> None:
    async def mynode(input: list[str]) -> list[str]:
        return [it + "a" for it in input]

    builder = StateGraph(list[str])
    builder.add_node(mynode)
    builder.add_edge(START, "mynode")
    add_a = builder.compile()

    @task
    def submapper(input: int) -> str:
        return str(input)

    @task
    async def mapper(input: int) -> str:
        await asyncio.sleep(input / 100)
        return await submapper(input) * 2

    @entrypoint(checkpointer=async_checkpointer)
    async def graph(input: list[int]) -> list[str]:
        futures = [mapper(i) for i in input]
        mapped = await asyncio.gather(*futures)
        answer = interrupt("question")
        final = [m + answer for m in mapped]
        return await add_a.ainvoke(final)

    assert graph.get_input_jsonschema() == {
        "type": "array",
        "items": {"type": "integer"},
        "title": "LangGraphInput",
    }
    assert graph.get_output_jsonschema() == {
        "type": "array",
        "items": {"type": "string"},
        "title": "LangGraphOutput",
    }

    thread1 = {"configurable": {"thread_id": "1"}}
    assert [c async for c in graph.astream([0, 1], thread1, durability=durability)] == [
        {"submapper": "0"},
        {"mapper": "00"},
        {"submapper": "1"},
        {"mapper": "11"},
        {
            "__interrupt__": (
                Interrupt(
                    value="question",
                    id=AnyStr(),
                ),
            )
        },
    ]

    assert await graph.ainvoke(
        Command(resume="answer"), thread1, durability=durability
    ) == [
        "00answera",
        "11answera",
    ]


@NEEDS_CONTEXTVARS
async def test_imp_task_cancel(
    async_checkpointer: BaseCheckpointSaver, durability: Durability
) -> None:
    mapper_calls = 0
    mapper_cancels = 0

    @task()
    async def mapper(input: int) -> str:
        nonlocal mapper_calls, mapper_cancels
        mapper_calls += 1
        try:
            await asyncio.sleep(1)
        except asyncio.CancelledError:
            mapper_cancels += 1
            raise
        return str(input) * 2

    @entrypoint(checkpointer=async_checkpointer)
    async def graph(input: list[int]) -> list[str]:
        futures = [mapper(i) for i in input]
        await asyncio.sleep(0.1)
        futures.pop().cancel()  # cancel one
        mapped = await asyncio.gather(*futures)
        answer = interrupt("question")
        return [m + answer for m in mapped]

    thread1 = {"configurable": {"thread_id": "1"}}
    assert [c async for c in graph.astream([0, 1], thread1, durability=durability)] == [
        {"mapper": "00"},
        {
            "__interrupt__": (
                Interrupt(
                    value="question",
                    id=AnyStr(),
                ),
            )
        },
    ]
    assert mapper_calls == 2
    assert mapper_cancels == 1

    assert await graph.ainvoke(
        Command(resume="answer"), thread1, durability=durability
    ) == [
        "00answer",
    ]
    assert mapper_calls == 3
    assert mapper_cancels == 2


@NEEDS_CONTEXTVARS
async def test_imp_sync_from_async(
    async_checkpointer: BaseCheckpointSaver, durability: Durability
) -> None:
    @task()
    def foo(state: dict) -> dict:
        return {"a": state["a"] + "foo", "b": "bar"}

    @task
    def bar(a: str, b: str, c: str | None = None) -> dict:
        return {"a": a + b, "c": (c or "") + "bark"}

    @task()
    def baz(state: dict) -> dict:
        return {"a": state["a"] + "baz", "c": "something else"}

    @entrypoint(checkpointer=async_checkpointer)
    def graph(state: dict) -> dict:
        foo_result = foo(state).result()
        fut_bar = bar(foo_result["a"], foo_result["b"])
        fut_baz = baz(fut_bar.result())
        return fut_baz.result()

    thread1 = {"configurable": {"thread_id": "1"}}
    assert [
        c async for c in graph.astream({"a": "0"}, thread1, durability=durability)
    ] == [
        {"foo": {"a": "0foo", "b": "bar"}},
        {"bar": {"a": "0foobar", "c": "bark"}},
        {"baz": {"a": "0foobarbaz", "c": "something else"}},
        {"graph": {"a": "0foobarbaz", "c": "something else"}},
    ]


@NEEDS_CONTEXTVARS
async def test_imp_stream_order(
    async_checkpointer: BaseCheckpointSaver, durability: Durability
) -> None:
    @task()
    async def foo(state: dict) -> dict:
        return {"a": state["a"] + "foo", "b": "bar"}

    @task
    async def bar(a: str, b: str, c: str | None = None) -> dict:
        return {"a": a + b, "c": (c or "") + "bark"}

    @task()
    async def baz(state: dict) -> dict:
        return {"a": state["a"] + "baz", "c": "something else"}

    @entrypoint(checkpointer=async_checkpointer)
    async def graph(state: dict) -> dict:
        foo_res = await foo(state)

        fut_bar = bar(foo_res["a"], foo_res["b"])
        fut_baz = baz(await fut_bar)
        return await fut_baz

    thread1 = {"configurable": {"thread_id": "1"}}
    assert [
        c async for c in graph.astream({"a": "0"}, thread1, durability=durability)
    ] == [
        {"foo": {"a": "0foo", "b": "bar"}},
        {"bar": {"a": "0foobar", "c": "bark"}},
        {"baz": {"a": "0foobarbaz", "c": "something else"}},
        {"graph": {"a": "0foobarbaz", "c": "something else"}},
    ]


@pytest.mark.skipif(
    sys.version_info < (3, 11),
    reason="Requires Python 3.11 or higher for context management",
)
async def test_send_dedupe_on_resume(
    async_checkpointer: BaseCheckpointSaver, durability: Durability
) -> None:
    class InterruptOnce:
        ticks: int = 0

        def __call__(self, state):
            self.ticks += 1
            if self.ticks == 1:
                interrupt("Bahh")
            return ["|".join(("flaky", str(state)))]

    class Node:
        def __init__(self, name: str):
            self.name = name
            self.ticks = 0
            setattr(self, "__name__", name)

        def __call__(self, state):
            self.ticks += 1
            update = (
                [self.name]
                if isinstance(state, list)
                else ["|".join((self.name, str(state)))]
            )
            if isinstance(state, Command):
                return replace(state, update=update)
            else:
                return update

    def send_for_fun(state):
        return [
            Send("2", Command(goto=Send("2", 3))),
            Send("2", Command(goto=Send("flaky", 4))),
            "3.1",
        ]

    def route_to_three(state) -> Literal["3"]:
        return "3"

    builder = StateGraph(Annotated[list, operator.add])
    builder.add_node(Node("1"))
    builder.add_node(Node("2"))
    builder.add_node(Node("3"))
    builder.add_node(Node("3.1"))
    builder.add_node("flaky", InterruptOnce())
    builder.add_edge(START, "1")
    builder.add_conditional_edges("1", send_for_fun)
    builder.add_conditional_edges("2", route_to_three)

    graph = builder.compile(checkpointer=async_checkpointer)
    thread1 = {"configurable": {"thread_id": "1"}}
    assert await graph.ainvoke(["0"], thread1, durability=durability) == {
        "__interrupt__": [
            Interrupt(
                value="Bahh",
                id=AnyStr(),
            ),
        ],
    }
    assert builder.nodes["2"].runnable.func.ticks == 3
    assert builder.nodes["flaky"].runnable.func.ticks == 1
    # resume execution
    assert await graph.ainvoke(None, thread1, durability=durability) == [
        "0",
        "1",
        "3.1",
        "2|Command(goto=Send(node='2', arg=3))",
        "2|Command(goto=Send(node='flaky', arg=4))",
        "3",
        "2|3",
        "flaky|4",
        "3",
    ]
    # node "2" doesn't get called again, as we recover writes saved before
    assert builder.nodes["2"].runnable.func.ticks == 3
    # node "flaky" gets called again, as it was interrupted
    assert builder.nodes["flaky"].runnable.func.ticks == 2
    # check history
    history = [c async for c in graph.aget_state_history(thread1)]
    assert len(history) == (6 if durability != "exit" else 2)
    expected_history = [
        StateSnapshot(
            values=[
                "0",
                "1",
                "3.1",
                "2|Command(goto=Send(node='2', arg=3))",
                "2|Command(goto=Send(node='flaky', arg=4))",
                "3",
                "2|3",
                "flaky|4",
                "3",
            ],
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
            values=[
                "0",
                "1",
                "3.1",
                "2|Command(goto=Send(node='2', arg=3))",
                "2|Command(goto=Send(node='flaky', arg=4))",
                "3",
                "2|3",
                "flaky|4",
            ],
            next=("3",),
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
                    name="3",
                    path=("__pregel_pull", "3"),
                    error=None,
                    interrupts=(),
                    state=None,
                    result=["3"],
                ),
            ),
            interrupts=(),
        ),
        StateSnapshot(
            values=[
                "0",
                "1",
                "3.1",
                "2|Command(goto=Send(node='2', arg=3))",
                "2|Command(goto=Send(node='flaky', arg=4))",
            ],
            next=("2", "flaky", "3"),
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
                    name="2",
                    path=("__pregel_push", 0, False),
                    error=None,
                    interrupts=(),
                    state=None,
                    result=["2|3"],
                ),
                PregelTask(
                    id=AnyStr(),
                    name="flaky",
                    path=("__pregel_push", 1, False),
                    error=None,
                    interrupts=(Interrupt(value="Bahh", id=AnyStr()),),
                    state=None,
                    result=["flaky|4"] if durability != "exit" else None,
                ),
                PregelTask(
                    id=AnyStr(),
                    name="3",
                    path=("__pregel_pull", "3"),
                    error=None,
                    interrupts=(),
                    state=None,
                    result=["3"],
                ),
            ),
            interrupts=(Interrupt(value="Bahh", id=AnyStr()),),
        ),
        StateSnapshot(
            values=["0", "1"],
            next=("2", "2", "3.1"),
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
                    name="2",
                    path=("__pregel_push", 0, False),
                    error=None,
                    interrupts=(),
                    state=None,
                    result=["2|Command(goto=Send(node='2', arg=3))"],
                ),
                PregelTask(
                    id=AnyStr(),
                    name="2",
                    path=("__pregel_push", 1, False),
                    error=None,
                    interrupts=(),
                    state=None,
                    result=["2|Command(goto=Send(node='flaky', arg=4))"],
                ),
                PregelTask(
                    id=AnyStr(),
                    name="3.1",
                    path=("__pregel_pull", "3.1"),
                    error=None,
                    interrupts=(),
                    state=None,
                    result=["3.1"],
                ),
            ),
            interrupts=(),
        ),
        StateSnapshot(
            values=["0"],
            next=("1",),
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
                    name="1",
                    path=("__pregel_pull", "1"),
                    error=None,
                    interrupts=(),
                    state=None,
                    result=["1"],
                ),
            ),
            interrupts=(),
        ),
        StateSnapshot(
            values=[],
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
                    result=["0"],
                ),
            ),
            interrupts=(),
        ),
    ]
    if durability != "exit":
        assert history == expected_history
    else:
        assert history[0] == expected_history[0]._replace(
            parent_config=history[1].config
        )
        assert history[1] == expected_history[2]._replace(parent_config=None)


async def test_send_react_interrupt(async_checkpointer: BaseCheckpointSaver) -> None:
    from langchain_core.messages import AIMessage, HumanMessage, ToolCall, ToolMessage

    ai_message = AIMessage(
        "",
        id="ai1",
        tool_calls=[ToolCall(name="foo", args={"hi": [1, 2, 3]}, id=AnyStr())],
    )

    async def agent(state):
        return {"messages": ai_message}

    def route(state):
        if isinstance(state["messages"][-1], AIMessage):
            return [
                Send(call["name"], call) for call in state["messages"][-1].tool_calls
            ]

    foo_called = 0

    async def foo(call: ToolCall):
        nonlocal foo_called
        foo_called += 1
        return {"messages": ToolMessage(str(call["args"]), tool_call_id=call["id"])}

    builder = StateGraph(MessagesState)
    builder.add_node(agent)
    builder.add_node(foo)
    builder.add_edge(START, "agent")
    builder.add_conditional_edges("agent", route)
    graph = builder.compile()

    assert await graph.ainvoke({"messages": [HumanMessage("hello")]}) == {
        "messages": [
            _AnyIdHumanMessage(content="hello"),
            _AnyIdAIMessage(
                content="",
                tool_calls=[
                    {
                        "name": "foo",
                        "args": {"hi": [1, 2, 3]},
                        "id": "",
                        "type": "tool_call",
                    }
                ],
            ),
            _AnyIdToolMessage(
                content="{'hi': [1, 2, 3]}",
                tool_call_id=AnyStr(),
            ),
        ]
    }
    assert foo_called == 1

    # simple interrupt-resume flow
    foo_called = 0
    graph = builder.compile(checkpointer=async_checkpointer, interrupt_before=["foo"])
    thread1 = {"configurable": {"thread_id": "1"}}
    assert await graph.ainvoke({"messages": [HumanMessage("hello")]}, thread1) == {
        "messages": [
            _AnyIdHumanMessage(content="hello"),
            _AnyIdAIMessage(
                content="",
                tool_calls=[
                    {
                        "name": "foo",
                        "args": {"hi": [1, 2, 3]},
                        "id": "",
                        "type": "tool_call",
                    }
                ],
            ),
        ]
    }
    assert foo_called == 0
    assert await graph.ainvoke(None, thread1) == {
        "messages": [
            _AnyIdHumanMessage(content="hello"),
            _AnyIdAIMessage(
                content="",
                tool_calls=[
                    {
                        "name": "foo",
                        "args": {"hi": [1, 2, 3]},
                        "id": "",
                        "type": "tool_call",
                    }
                ],
            ),
            _AnyIdToolMessage(
                content="{'hi': [1, 2, 3]}",
                tool_call_id=AnyStr(),
            ),
        ]
    }
    assert foo_called == 1

    # interrupt-update-resume flow
    foo_called = 0
    graph = builder.compile(checkpointer=async_checkpointer, interrupt_before=["foo"])
    thread1 = {"configurable": {"thread_id": "2"}}
    assert await graph.ainvoke(
        {"messages": [HumanMessage("hello")]}, thread1, durability="exit"
    ) == {
        "messages": [
            _AnyIdHumanMessage(content="hello"),
            _AnyIdAIMessage(
                content="",
                tool_calls=[
                    {
                        "name": "foo",
                        "args": {"hi": [1, 2, 3]},
                        "id": "",
                        "type": "tool_call",
                    }
                ],
            ),
        ]
    }
    assert foo_called == 0

    # get state should show the pending task
    state = await graph.aget_state(thread1)
    assert state == StateSnapshot(
        values={
            "messages": [
                _AnyIdHumanMessage(content="hello"),
                _AnyIdAIMessage(
                    content="",
                    tool_calls=[
                        {
                            "name": "foo",
                            "args": {"hi": [1, 2, 3]},
                            "id": "",
                            "type": "tool_call",
                        }
                    ],
                ),
            ]
        },
        next=("foo",),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        metadata={
            "step": 1,
            "source": "loop",
            "parents": {},
        },
        created_at=AnyStr(),
        parent_config=None,
        tasks=(
            PregelTask(
                id=AnyStr(),
                name="foo",
                path=("__pregel_push", 0, False),
                error=None,
                interrupts=(),
                state=None,
                result=None,
            ),
        ),
        interrupts=(),
    )

    # remove the tool call, clearing the pending task
    await graph.aupdate_state(
        thread1, {"messages": AIMessage("Bye now", id=ai_message.id, tool_calls=[])}
    )

    # tool call no longer in pending tasks
    assert await graph.aget_state(thread1) == StateSnapshot(
        values={
            "messages": [
                _AnyIdHumanMessage(content="hello"),
                _AnyIdAIMessage(
                    content="Bye now",
                    tool_calls=[],
                ),
            ]
        },
        next=(),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        metadata={
            "step": 2,
            "source": "update",
            "parents": {},
        },
        created_at=AnyStr(),
        parent_config=(
            {
                "configurable": {
                    "thread_id": "2",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            }
        ),
        tasks=(),
        interrupts=(),
    )

    # tool call not executed
    assert await graph.ainvoke(None, thread1) == {
        "messages": [
            _AnyIdHumanMessage(content="hello"),
            _AnyIdAIMessage(content="Bye now"),
        ]
    }
    assert foo_called == 0

    # interrupt-update-resume flow, creating new Send in update call
    foo_called = 0
    graph = builder.compile(checkpointer=async_checkpointer, interrupt_before=["foo"])
    thread1 = {"configurable": {"thread_id": "3"}}
    assert await graph.ainvoke(
        {"messages": [HumanMessage("hello")]}, thread1, durability="exit"
    ) == {
        "messages": [
            _AnyIdHumanMessage(content="hello"),
            _AnyIdAIMessage(
                content="",
                tool_calls=[
                    {
                        "name": "foo",
                        "args": {"hi": [1, 2, 3]},
                        "id": "",
                        "type": "tool_call",
                    }
                ],
            ),
        ]
    }
    assert foo_called == 0

    # get state should show the pending task
    state = await graph.aget_state(thread1)
    assert state == StateSnapshot(
        values={
            "messages": [
                _AnyIdHumanMessage(content="hello"),
                _AnyIdAIMessage(
                    content="",
                    tool_calls=[
                        {
                            "name": "foo",
                            "args": {"hi": [1, 2, 3]},
                            "id": "",
                            "type": "tool_call",
                        }
                    ],
                ),
            ]
        },
        next=("foo",),
        config={
            "configurable": {
                "thread_id": "3",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        metadata={
            "step": 1,
            "source": "loop",
            "parents": {},
        },
        created_at=AnyStr(),
        parent_config=None,
        tasks=(
            PregelTask(
                id=AnyStr(),
                name="foo",
                path=("__pregel_push", 0, False),
                error=None,
                interrupts=(),
                state=None,
                result=None,
            ),
        ),
        interrupts=(),
    )

    # replace the tool call, should clear previous send, create new one
    await graph.aupdate_state(
        thread1,
        {
            "messages": AIMessage(
                "",
                id=ai_message.id,
                tool_calls=[
                    {
                        "name": "foo",
                        "args": {"hi": [4, 5, 6]},
                        "id": "tool1",
                        "type": "tool_call",
                    }
                ],
            )
        },
    )

    # prev tool call no longer in pending tasks, new tool call is
    assert await graph.aget_state(thread1) == StateSnapshot(
        values={
            "messages": [
                _AnyIdHumanMessage(content="hello"),
                _AnyIdAIMessage(
                    content="",
                    tool_calls=[
                        {
                            "name": "foo",
                            "args": {"hi": [4, 5, 6]},
                            "id": "tool1",
                            "type": "tool_call",
                        }
                    ],
                ),
            ]
        },
        next=("foo",),
        config={
            "configurable": {
                "thread_id": "3",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        metadata={
            "step": 2,
            "source": "update",
            "parents": {},
        },
        created_at=AnyStr(),
        parent_config=(
            {
                "configurable": {
                    "thread_id": "3",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            }
        ),
        tasks=(
            PregelTask(
                id=AnyStr(),
                name="foo",
                path=("__pregel_push", 0, False),
                error=None,
                interrupts=(),
                state=None,
                result=None,
            ),
        ),
        interrupts=(),
    )

    # prev tool call not executed, new tool call is
    assert await graph.ainvoke(None, thread1) == {
        "messages": [
            _AnyIdHumanMessage(content="hello"),
            AIMessage(
                "",
                id="ai1",
                tool_calls=[
                    {
                        "name": "foo",
                        "args": {"hi": [4, 5, 6]},
                        "id": "tool1",
                        "type": "tool_call",
                    }
                ],
            ),
            _AnyIdToolMessage(content="{'hi': [4, 5, 6]}", tool_call_id="tool1"),
        ]
    }
    assert foo_called == 1


async def test_send_react_interrupt_control(
    async_checkpointer: BaseCheckpointSaver, snapshot: SnapshotAssertion
) -> None:
    from langchain_core.messages import AIMessage, HumanMessage, ToolCall, ToolMessage

    ai_message = AIMessage(
        "",
        id="ai1",
        tool_calls=[ToolCall(name="foo", args={"hi": [1, 2, 3]}, id=AnyStr())],
    )

    async def agent(state) -> Command[Literal["foo"]]:
        return Command(
            update={"messages": ai_message},
            goto=[Send(call["name"], call) for call in ai_message.tool_calls],
        )

    foo_called = 0

    async def foo(call: ToolCall):
        nonlocal foo_called
        foo_called += 1
        return {"messages": ToolMessage(str(call["args"]), tool_call_id=call["id"])}

    builder = StateGraph(MessagesState)
    builder.add_node(agent)
    builder.add_node(foo)
    builder.add_edge(START, "agent")
    graph = builder.compile()
    if isinstance(async_checkpointer, InMemorySaver):
        assert graph.get_graph().draw_mermaid() == snapshot

    assert await graph.ainvoke({"messages": [HumanMessage("hello")]}) == {
        "messages": [
            _AnyIdHumanMessage(content="hello"),
            _AnyIdAIMessage(
                content="",
                tool_calls=[
                    {
                        "name": "foo",
                        "args": {"hi": [1, 2, 3]},
                        "id": "",
                        "type": "tool_call",
                    }
                ],
            ),
            _AnyIdToolMessage(
                content="{'hi': [1, 2, 3]}",
                tool_call_id=AnyStr(),
            ),
        ]
    }
    assert foo_called == 1

    # simple interrupt-resume flow
    foo_called = 0
    graph = builder.compile(checkpointer=async_checkpointer, interrupt_before=["foo"])
    thread1 = {"configurable": {"thread_id": "1"}}
    assert await graph.ainvoke({"messages": [HumanMessage("hello")]}, thread1) == {
        "messages": [
            _AnyIdHumanMessage(content="hello"),
            _AnyIdAIMessage(
                content="",
                tool_calls=[
                    {
                        "name": "foo",
                        "args": {"hi": [1, 2, 3]},
                        "id": "",
                        "type": "tool_call",
                    }
                ],
            ),
        ]
    }
    assert foo_called == 0
    assert await graph.ainvoke(None, thread1) == {
        "messages": [
            _AnyIdHumanMessage(content="hello"),
            _AnyIdAIMessage(
                content="",
                tool_calls=[
                    {
                        "name": "foo",
                        "args": {"hi": [1, 2, 3]},
                        "id": "",
                        "type": "tool_call",
                    }
                ],
            ),
            _AnyIdToolMessage(
                content="{'hi': [1, 2, 3]}",
                tool_call_id=AnyStr(),
            ),
        ]
    }
    assert foo_called == 1

    # interrupt-update-resume flow
    foo_called = 0
    graph = builder.compile(checkpointer=async_checkpointer, interrupt_before=["foo"])
    thread1 = {"configurable": {"thread_id": "2"}}
    assert await graph.ainvoke(
        {"messages": [HumanMessage("hello")]}, thread1, durability="exit"
    ) == {
        "messages": [
            _AnyIdHumanMessage(content="hello"),
            _AnyIdAIMessage(
                content="",
                tool_calls=[
                    {
                        "name": "foo",
                        "args": {"hi": [1, 2, 3]},
                        "id": "",
                        "type": "tool_call",
                    }
                ],
            ),
        ]
    }
    assert foo_called == 0

    # get state should show the pending task
    state = await graph.aget_state(thread1)
    assert state == StateSnapshot(
        values={
            "messages": [
                _AnyIdHumanMessage(content="hello"),
                _AnyIdAIMessage(
                    content="",
                    tool_calls=[
                        {
                            "name": "foo",
                            "args": {"hi": [1, 2, 3]},
                            "id": "",
                            "type": "tool_call",
                        }
                    ],
                ),
            ]
        },
        next=("foo",),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        metadata={
            "step": 1,
            "source": "loop",
            "parents": {},
        },
        created_at=AnyStr(),
        parent_config=None,
        tasks=(
            PregelTask(
                id=AnyStr(),
                name="foo",
                path=("__pregel_push", 0, False),
                error=None,
                interrupts=(),
                state=None,
                result=None,
            ),
        ),
        interrupts=(),
    )

    # remove the tool call, clearing the pending task
    await graph.aupdate_state(
        thread1, {"messages": AIMessage("Bye now", id=ai_message.id, tool_calls=[])}
    )

    # tool call no longer in pending tasks
    assert await graph.aget_state(thread1) == StateSnapshot(
        values={
            "messages": [
                _AnyIdHumanMessage(content="hello"),
                _AnyIdAIMessage(
                    content="Bye now",
                    tool_calls=[],
                ),
            ]
        },
        next=(),
        config={
            "configurable": {
                "thread_id": "2",
                "checkpoint_ns": "",
                "checkpoint_id": AnyStr(),
            }
        },
        metadata={
            "step": 2,
            "source": "update",
            "parents": {},
        },
        created_at=AnyStr(),
        parent_config=(
            {
                "configurable": {
                    "thread_id": "2",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            }
        ),
        tasks=(),
        interrupts=(),
    )

    # tool call not executed
    assert await graph.ainvoke(None, thread1) == {
        "messages": [
            _AnyIdHumanMessage(content="hello"),
            _AnyIdAIMessage(content="Bye now"),
        ]
    }
    assert foo_called == 0


async def test_max_concurrency(async_checkpointer: BaseCheckpointSaver) -> None:
    class Node:
        def __init__(self, name: str):
            self.name = name
            setattr(self, "__name__", name)
            self.currently = 0
            self.max_currently = 0

        async def __call__(self, state):
            self.currently += 1
            if self.currently > self.max_currently:
                self.max_currently = self.currently
            await asyncio.sleep(random.random() / 10)
            self.currently -= 1
            return [state]

    def one(state):
        return ["1"]

    def three(state):
        return ["3"]

    async def send_to_many(state):
        return [Send("2", idx) for idx in range(100)]

    async def route_to_three(state) -> Literal["3"]:
        return "3"

    node2 = Node("2")
    builder = StateGraph(Annotated[list, operator.add])
    builder.add_node("1", one)
    builder.add_node(node2)
    builder.add_node("3", three)
    builder.add_edge(START, "1")
    builder.add_conditional_edges("1", send_to_many)
    builder.add_conditional_edges("2", route_to_three)
    graph = builder.compile()

    assert await graph.ainvoke(["0"]) == ["0", "1", *range(100), "3"]
    assert node2.max_currently == 100
    assert node2.currently == 0
    node2.max_currently = 0

    assert await graph.ainvoke(["0"], {"max_concurrency": 10}) == [
        "0",
        "1",
        *range(100),
        "3",
    ]
    assert node2.max_currently == 10
    assert node2.currently == 0

    graph = builder.compile(checkpointer=async_checkpointer, interrupt_before=["2"])
    thread1 = {"max_concurrency": 10, "configurable": {"thread_id": "1"}}

    assert await graph.ainvoke(["0"], thread1, debug=True) == ["0", "1"]
    state = await graph.aget_state(thread1)
    assert state.values == ["0", "1"]
    assert await graph.ainvoke(None, thread1) == ["0", "1", *range(100), "3"]


async def test_max_concurrency_control(async_checkpointer: BaseCheckpointSaver) -> None:
    async def node1(state) -> Command[Literal["2"]]:
        return Command(update=["1"], goto=[Send("2", idx) for idx in range(100)])

    node2_currently = 0
    node2_max_currently = 0

    async def node2(state) -> Command[Literal["3"]]:
        nonlocal node2_currently, node2_max_currently
        node2_currently += 1
        if node2_currently > node2_max_currently:
            node2_max_currently = node2_currently
        await asyncio.sleep(0.1)
        node2_currently -= 1

        return Command(update=[state], goto="3")

    async def node3(state) -> Literal["3"]:
        return ["3"]

    builder = StateGraph(Annotated[list, operator.add])
    builder.add_node("1", node1)
    builder.add_node("2", node2)
    builder.add_node("3", node3)
    builder.add_edge(START, "1")
    graph = builder.compile()

    if isinstance(async_checkpointer, InMemorySaver):
        assert (
            graph.get_graph().draw_mermaid()
            == """---
config:
  flowchart:
    curve: linear
---
graph TD;
	__start__([<p>__start__</p>]):::first
	1(1)
	2(2)
	3(3)
	__end__([<p>__end__</p>]):::last
	1 -.-> 2;
	2 -.-> 3;
	__start__ --> 1;
	3 --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
"""
        )

    assert await graph.ainvoke(["0"], debug=True) == ["0", "1", *range(100), "3"]
    assert node2_max_currently == 100
    assert node2_currently == 0
    node2_max_currently = 0

    assert await graph.ainvoke(["0"], {"max_concurrency": 10}) == [
        "0",
        "1",
        *range(100),
        "3",
    ]
    assert node2_max_currently == 10
    assert node2_currently == 0

    graph = builder.compile(checkpointer=async_checkpointer, interrupt_before=["2"])
    thread1 = {"max_concurrency": 10, "configurable": {"thread_id": "1"}}

    assert await graph.ainvoke(["0"], thread1) == ["0", "1"]
    assert await graph.ainvoke(None, thread1) == ["0", "1", *range(100), "3"]


async def test_invoke_checkpoint_three(
    mocker: MockerFixture, async_checkpointer: BaseCheckpointSaver
) -> None:
    add_one = mocker.Mock(side_effect=lambda x: x["total"] + x["input"])

    def raise_if_above_10(input: int) -> int:
        if input > 10:
            raise ValueError("Input is too large")
        return input

    one = (
        NodeBuilder()
        .subscribe_to("input")
        .read_from("total")
        .do(add_one)
        .write_to("output", "total")
        .do(raise_if_above_10)
    )

    app = Pregel(
        nodes={"one": one},
        channels={
            "total": BinaryOperatorAggregate(int, operator.add),
            "input": LastValue(int),
            "output": LastValue(int),
        },
        input_channels="input",
        output_channels="output",
        checkpointer=async_checkpointer,
        debug=True,
    )

    thread_1 = {"configurable": {"thread_id": "1"}}
    # total starts out as 0, so output is 0+2=2
    assert await app.ainvoke(2, thread_1, durability="async") == 2
    state = await app.aget_state(thread_1)
    assert state is not None
    assert state.values.get("total") == 2
    assert (
        state.config["configurable"]["checkpoint_id"]
        == (await async_checkpointer.aget(thread_1))["id"]
    )
    # total is now 2, so output is 2+3=5
    assert await app.ainvoke(3, thread_1, durability="async") == 5
    state = await app.aget_state(thread_1)
    assert state is not None
    assert state.values.get("total") == 7
    assert (
        state.config["configurable"]["checkpoint_id"]
        == (await async_checkpointer.aget(thread_1))["id"]
    )
    # total is now 2+5=7, so output would be 7+4=11, but raises ValueError
    with pytest.raises(ValueError):
        await app.ainvoke(4, thread_1, durability="async")
    # checkpoint is not updated
    state = await app.aget_state(thread_1)
    assert state is not None
    assert state.values.get("total") == 7
    assert state.next == ("one",)
    """we checkpoint inputs and it failed on "one", so the next node is one"""
    # we can recover from error by sending new inputs
    assert await app.ainvoke(2, thread_1, durability="async") == 9
    state = await app.aget_state(thread_1)
    assert state is not None
    assert state.values.get("total") == 16, "total is now 7+9=16"
    assert state.next == ()

    thread_2 = {"configurable": {"thread_id": "2"}}
    # on a new thread, total starts out as 0, so output is 0+5=5
    assert await app.ainvoke(5, thread_2) == 5
    state = await app.aget_state(thread_1)
    assert state is not None
    assert state.values.get("total") == 16
    assert state.next == ()
    state = await app.aget_state(thread_2)
    assert state is not None
    assert state.values.get("total") == 5
    assert state.next == ()

    assert len([c async for c in app.aget_state_history(thread_1, limit=1)]) == 1
    # list all checkpoints for thread 1
    thread_1_history = [c async for c in app.aget_state_history(thread_1)]
    # there are 7 checkpoints
    assert len(thread_1_history) == 7
    assert Counter(c.metadata["source"] for c in thread_1_history) == {
        "input": 4,
        "loop": 3,
    }
    # sorted descending
    assert (
        thread_1_history[0].config["configurable"]["checkpoint_id"]
        > thread_1_history[1].config["configurable"]["checkpoint_id"]
    )
    # cursor pagination
    cursored = [
        c
        async for c in app.aget_state_history(
            thread_1, limit=1, before=thread_1_history[0].config
        )
    ]
    assert len(cursored) == 1
    assert cursored[0].config == thread_1_history[1].config
    # the last checkpoint
    assert thread_1_history[0].values["total"] == 16
    # the first "loop" checkpoint
    assert thread_1_history[-2].values["total"] == 2
    # can get each checkpoint using aget with config
    assert (await async_checkpointer.aget(thread_1_history[0].config))[
        "id"
    ] == thread_1_history[0].config["configurable"]["checkpoint_id"]
    assert (await async_checkpointer.aget(thread_1_history[1].config))[
        "id"
    ] == thread_1_history[1].config["configurable"]["checkpoint_id"]

    thread_1_next_config = await app.aupdate_state(thread_1_history[1].config, 10)
    # update creates a new checkpoint
    assert (
        thread_1_next_config["configurable"]["checkpoint_id"]
        > thread_1_history[0].config["configurable"]["checkpoint_id"]
    )
    # 1 more checkpoint in history
    assert len([c async for c in app.aget_state_history(thread_1)]) == 8
    assert Counter(
        [c.metadata["source"] async for c in app.aget_state_history(thread_1)]
    ) == {
        "update": 1,
        "input": 4,
        "loop": 3,
    }
    # the latest checkpoint is the updated one
    assert await app.aget_state(thread_1) == await app.aget_state(thread_1_next_config)


async def test_invoke_two_processes_two_in_join_two_out(mocker: MockerFixture) -> None:
    add_one = mocker.Mock(side_effect=lambda x: x + 1)
    add_10_each = mocker.Mock(side_effect=lambda x: sorted(y + 10 for y in x))

    one = NodeBuilder().subscribe_only("input").do(add_one).write_to("inbox")
    chain_three = NodeBuilder().subscribe_only("input").do(add_one).write_to("inbox")
    chain_four = (
        NodeBuilder().subscribe_only("inbox").do(add_10_each).write_to("output")
    )

    app = Pregel(
        nodes={
            "one": one,
            "chain_three": chain_three,
            "chain_four": chain_four,
        },
        channels={
            "inbox": Topic(int),
            "output": LastValue(int),
            "input": LastValue(int),
        },
        input_channels="input",
        output_channels="output",
    )

    # Then invoke app
    # We get a single array result as chain_four waits for all publishers to finish
    # before operating on all elements published to topic_two as an array
    for _ in range(100):
        assert await app.ainvoke(2) == [13, 13]

    assert await asyncio.gather(*(app.ainvoke(2) for _ in range(100))) == [
        [13, 13] for _ in range(100)
    ]


async def test_invoke_two_processes_one_in_two_out(mocker: MockerFixture) -> None:
    add_one = mocker.Mock(side_effect=lambda x: x + 1)

    one = (
        NodeBuilder().subscribe_only("input").do(add_one).write_to("output", "between")
    )
    two = NodeBuilder().subscribe_only("between").do(add_one).write_to("output")

    app = Pregel(
        nodes={"one": one, "two": two},
        channels={
            "input": LastValue(int),
            "between": LastValue(int),
            "output": LastValue(int),
        },
        stream_channels=["output", "between"],
        input_channels="input",
        output_channels="output",
    )

    # Then invoke pubsub
    assert [c async for c in app.astream(2)] == [
        {"between": 3, "output": 3},
        {"between": 3, "output": 4},
    ]


async def test_invoke_two_processes_no_out(mocker: MockerFixture) -> None:
    add_one = mocker.Mock(side_effect=lambda x: x + 1)
    one = NodeBuilder().subscribe_only("input").do(add_one).write_to("between")
    two = NodeBuilder().subscribe_only("between").do(add_one)

    app = Pregel(
        nodes={"one": one, "two": two},
        channels={
            "input": LastValue(int),
            "between": LastValue(int),
            "output": LastValue(int),
        },
        input_channels="input",
        output_channels="output",
    )

    # It finishes executing (once no more messages being published)
    # but returns nothing, as nothing was published to "output" topic
    assert await app.ainvoke(2) is None


async def test_conditional_entrypoint_graph_state() -> None:
    class AgentState(TypedDict, total=False):
        input: str
        output: str
        steps: Annotated[list[str], operator.add]

    async def left(data: AgentState) -> AgentState:
        return {"output": data["input"] + "->left"}

    async def right(data: AgentState) -> AgentState:
        return {"output": data["input"] + "->right"}

    def should_start(data: AgentState) -> str:
        assert data["steps"] == [], "Expected input to be read from the state"
        # Logic to decide where to start
        if len(data["input"]) > 10:
            return "go-right"
        else:
            return "go-left"

    # Define a new graph
    workflow = StateGraph(AgentState)

    workflow.add_node("left", left)
    workflow.add_node("right", right)

    workflow.set_conditional_entry_point(
        should_start, {"go-left": "left", "go-right": "right"}
    )

    workflow.add_conditional_edges("left", lambda data: END)
    workflow.add_edge("right", END)

    app = workflow.compile()

    assert await app.ainvoke({"input": "what is weather in sf"}) == {
        "input": "what is weather in sf",
        "output": "what is weather in sf->right",
        "steps": [],
    }

    assert [c async for c in app.astream({"input": "what is weather in sf"})] == [
        {"right": {"output": "what is weather in sf->right"}},
    ]


async def test_in_one_fan_out_state_graph_waiting_edge(
    async_checkpointer: BaseCheckpointSaver,
) -> None:
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

    async def rewrite_query(data: State) -> State:
        return {"query": f"query: {data['query']}"}

    async def analyzer_one(data: State) -> State:
        return {"query": f"analyzed: {data['query']}"}

    async def retriever_one(data: State) -> State:
        return {"docs": ["doc1", "doc2"]}

    async def retriever_two(data: State) -> State:
        await asyncio.sleep(0.1)
        return {"docs": ["doc3", "doc4"]}

    async def qa(data: State) -> State:
        return {"answer": ",".join(data["docs"])}

    workflow = StateGraph(State)

    workflow.add_node("rewrite_query", rewrite_query)
    workflow.add_node("analyzer_one", analyzer_one)
    workflow.add_node("retriever_one", retriever_one)
    workflow.add_node("retriever_two", retriever_two)
    workflow.add_node("qa", qa)

    workflow.set_entry_point("rewrite_query")
    workflow.add_edge("rewrite_query", "analyzer_one")
    workflow.add_edge("analyzer_one", "retriever_one")
    workflow.add_edge("rewrite_query", "retriever_two")
    workflow.add_edge(["retriever_one", "retriever_two"], "qa")
    workflow.set_finish_point("qa")

    app = workflow.compile()

    assert await app.ainvoke({"query": "what is weather in sf"}) == {
        "query": "analyzed: query: what is weather in sf",
        "docs": ["doc1", "doc2", "doc3", "doc4"],
        "answer": "doc1,doc2,doc3,doc4",
    }

    assert [c async for c in app.astream({"query": "what is weather in sf"})] == [
        {"rewrite_query": {"query": "query: what is weather in sf"}},
        {"analyzer_one": {"query": "analyzed: query: what is weather in sf"}},
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {"qa": {"answer": "doc1,doc2,doc3,doc4"}},
    ]

    app_w_interrupt = workflow.compile(
        checkpointer=async_checkpointer,
        interrupt_after=["retriever_one"],
    )
    config = {"configurable": {"thread_id": "1"}}

    assert [
        c
        async for c in app_w_interrupt.astream(
            {"query": "what is weather in sf"}, config
        )
    ] == [
        {"rewrite_query": {"query": "query: what is weather in sf"}},
        {"analyzer_one": {"query": "analyzed: query: what is weather in sf"}},
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {"__interrupt__": ()},
    ]

    assert [c async for c in app_w_interrupt.astream(None, config)] == [
        {"qa": {"answer": "doc1,doc2,doc3,doc4"}},
    ]


@pytest.mark.parametrize("use_waiting_edge", (True, False))
async def test_in_one_fan_out_state_graph_defer_node(
    async_checkpointer: BaseCheckpointSaver, use_waiting_edge: bool
) -> None:
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

    async def rewrite_query(data: State) -> State:
        return {"query": f"query: {data['query']}"}

    async def analyzer_one(data: State) -> State:
        return {"query": f"analyzed: {data['query']}"}

    async def retriever_one(data: State) -> State:
        return {"docs": ["doc1", "doc2"]}

    async def retriever_two(data: State) -> State:
        await asyncio.sleep(0.1)
        return {"docs": ["doc3", "doc4"]}

    async def qa(data: State) -> State:
        return {"answer": ",".join(data["docs"])}

    workflow = StateGraph(State)

    workflow.add_node("rewrite_query", rewrite_query)
    workflow.add_node("analyzer_one", analyzer_one)
    workflow.add_node("retriever_one", retriever_one)
    workflow.add_node("retriever_two", retriever_two)
    workflow.add_node("qa", qa, defer=True)

    workflow.set_entry_point("rewrite_query")
    workflow.add_edge("rewrite_query", "analyzer_one")
    workflow.add_edge("analyzer_one", "retriever_one")
    workflow.add_edge("rewrite_query", "retriever_two")
    if use_waiting_edge:
        workflow.add_edge(["retriever_one", "retriever_two"], "qa")
    else:
        workflow.add_edge("retriever_one", "qa")
        workflow.add_edge("retriever_two", "qa")
    workflow.set_finish_point("qa")

    app = workflow.compile()

    assert await app.ainvoke({"query": "what is weather in sf"}, debug=True) == {
        "query": "analyzed: query: what is weather in sf",
        "docs": ["doc1", "doc2", "doc3", "doc4"],
        "answer": "doc1,doc2,doc3,doc4",
    }

    assert [c async for c in app.astream({"query": "what is weather in sf"})] == [
        {"rewrite_query": {"query": "query: what is weather in sf"}},
        {"analyzer_one": {"query": "analyzed: query: what is weather in sf"}},
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {"qa": {"answer": "doc1,doc2,doc3,doc4"}},
    ]

    app_w_interrupt = workflow.compile(
        checkpointer=async_checkpointer,
        interrupt_after=["retriever_one"],
    )
    config = {"configurable": {"thread_id": "1"}}

    assert [
        c
        async for c in app_w_interrupt.astream(
            {"query": "what is weather in sf"}, config
        )
    ] == [
        {"rewrite_query": {"query": "query: what is weather in sf"}},
        {"analyzer_one": {"query": "analyzed: query: what is weather in sf"}},
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {"__interrupt__": ()},
    ]

    assert [c async for c in app_w_interrupt.astream(None, config)] == [
        {"qa": {"answer": "doc1,doc2,doc3,doc4"}},
    ]


async def test_in_one_fan_out_state_graph_waiting_edge_via_branch(
    async_checkpointer: BaseCheckpointSaver,
) -> None:
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

    async def rewrite_query(data: State) -> State:
        return {"query": f"query: {data['query']}"}

    async def analyzer_one(data: State) -> State:
        return {"query": f"analyzed: {data['query']}"}

    async def retriever_one(data: State) -> State:
        return {"docs": ["doc1", "doc2"]}

    async def retriever_two(data: State) -> State:
        await asyncio.sleep(0.1)
        return {"docs": ["doc3", "doc4"]}

    async def qa(data: State) -> State:
        return {"answer": ",".join(data["docs"])}

    workflow = StateGraph(State)

    workflow.add_node("rewrite_query", rewrite_query)
    workflow.add_node("analyzer_one", analyzer_one)
    workflow.add_node("retriever_one", retriever_one)
    workflow.add_node("retriever_two", retriever_two)
    workflow.add_node("qa", qa)

    workflow.set_entry_point("rewrite_query")
    workflow.add_edge("rewrite_query", "analyzer_one")
    workflow.add_edge("analyzer_one", "retriever_one")
    workflow.add_conditional_edges(
        "rewrite_query", lambda _: "retriever_two", {"retriever_two": "retriever_two"}
    )
    workflow.add_edge(["retriever_one", "retriever_two"], "qa")
    workflow.set_finish_point("qa")

    app = workflow.compile()

    assert await app.ainvoke({"query": "what is weather in sf"}, debug=True) == {
        "query": "analyzed: query: what is weather in sf",
        "docs": ["doc1", "doc2", "doc3", "doc4"],
        "answer": "doc1,doc2,doc3,doc4",
    }

    assert [c async for c in app.astream({"query": "what is weather in sf"})] == [
        {"rewrite_query": {"query": "query: what is weather in sf"}},
        {"analyzer_one": {"query": "analyzed: query: what is weather in sf"}},
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {"qa": {"answer": "doc1,doc2,doc3,doc4"}},
    ]

    app_w_interrupt = workflow.compile(
        checkpointer=async_checkpointer,
        interrupt_after=["retriever_one"],
    )
    config = {"configurable": {"thread_id": "1"}}

    assert [
        c
        async for c in app_w_interrupt.astream(
            {"query": "what is weather in sf"}, config
        )
    ] == [
        {"rewrite_query": {"query": "query: what is weather in sf"}},
        {"analyzer_one": {"query": "analyzed: query: what is weather in sf"}},
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {"__interrupt__": ()},
    ]

    assert [c async for c in app_w_interrupt.astream(None, config)] == [
        {"qa": {"answer": "doc1,doc2,doc3,doc4"}},
    ]


async def test_nested_pydantic_models() -> None:
    """Test that nested Pydantic models are properly constructed from leaf nodes up."""

    class NestedModel(BaseModel):
        value: int
        name: str
        something: str | None = None

    # Forward reference model
    class RecursiveModel(BaseModel):
        value: str
        child: Optional["RecursiveModel"] = None

    # Discriminated union models
    class Cat(BaseModel):
        pet_type: Literal["cat"]
        meow: str

    class Dog(BaseModel):
        pet_type: Literal["dog"]
        bark: str

    # Cyclic reference model
    class Person(BaseModel):
        id: str
        name: str
        friends: list[str] = Field(default_factory=list)  # IDs of friends

    class MyEnum(enum.Enum):
        A = 1
        B = 2

    class MyTypedDict(TypedDict):
        x: int
        my_enum: MyEnum

    class State(BaseModel):
        # Basic nested model tests
        top_level: str
        nested: NestedModel
        optional_nested: NestedModel | None = None
        dict_nested: dict[str, NestedModel]
        my_set: set[int]
        another_set: set
        my_enum: MyEnum
        list_nested: Annotated[
            dict | list[dict[str, NestedModel]], lambda x, y: (x or []) + [y]
        ]
        list_nested_reversed: Annotated[
            list[dict[str, NestedModel]] | NestedModel | dict | list,
            lambda x, y: (x or []) + [y],
        ]
        tuple_nested: tuple[str, NestedModel]
        tuple_list_nested: list[tuple[int, NestedModel]]
        complex_tuple: tuple[str, dict[str, tuple[int, NestedModel]]]
        my_typed_dict: MyTypedDict

        # Forward reference test
        recursive: RecursiveModel

        # Discriminated union test
        pet: Cat | Dog

        # Cyclic reference test
        people: dict[str, Person]  # Map of ID -> Person

    inputs = {
        # Basic nested models
        "top_level": "initial",
        "nested": {"value": 42, "name": "test"},
        "optional_nested": {"value": 10, "name": "optional"},
        "my_set": [1, 2, 7],
        "another_set": ["foo", 3],
        "my_enum": MyEnum.B,
        "my_typed_dict": {"x": 1, "my_enum": MyEnum.A},
        "dict_nested": {"a": {"value": 5, "name": "a"}},
        "list_nested": [{"a": {"value": 6, "name": "b"}}],
        "list_nested_reversed": ["foo", "bar"],
        "tuple_nested": ["tuple-key", {"value": 7, "name": "tuple-value"}],
        "tuple_list_nested": [[1, {"value": 8, "name": "tuple-in-list"}]],
        "complex_tuple": [
            "complex",
            {"nested": [9, {"value": 10, "name": "deep"}]},
        ],
        # Forward reference
        "recursive": {"value": "parent", "child": {"value": "child", "child": None}},
        # Discriminated union (using a cat in this case)
        "pet": {"pet_type": "cat", "meow": "meow!"},
        # Cyclic references
        "people": {
            "1": {
                "id": "1",
                "name": "Alice",
                "friends": ["2", "3"],  # Alice is friends with Bob and Charlie
            },
            "2": {
                "id": "2",
                "name": "Bob",
                "friends": ["1"],  # Bob is friends with Alice
            },
            "3": {
                "id": "3",
                "name": "Charlie",
                "friends": ["1", "2"],  # Charlie is friends with Alice and Bob
            },
        },
    }

    update = {"top_level": "updated", "nested": {"value": 100, "name": "updated"}}

    async def node_fn(state: State) -> dict:
        assert state == State(**inputs)
        return update

    builder = StateGraph(State)
    builder.add_node("process", node_fn)
    builder.set_entry_point("process")
    builder.set_finish_point("process")
    graph = builder.compile()

    result = await graph.ainvoke(inputs.copy())

    assert result == {**inputs, **update}


async def test_in_one_fan_out_state_graph_waiting_edge_custom_state_class(
    async_checkpointer: BaseCheckpointSaver,
) -> None:
    def sorted_add(x: list[str], y: list[str] | list[tuple[str, str]]) -> list[str]:
        if isinstance(y[0], tuple):
            for rem, _ in y:
                x.remove(rem)
            y = [t[1] for t in y]
        return sorted(operator.add(x, y))

    class State(BaseModel):
        model_config = ConfigDict(arbitrary_types_allowed=True)

        query: str
        answer: str | None = None
        docs: Annotated[list[str], sorted_add]

    class Input(BaseModel):
        query: str

    class Output(BaseModel):
        answer: str
        docs: list[str]

    class StateUpdate(BaseModel):
        query: str | None = None
        answer: str | None = None
        docs: list[str] | None = None

    async def rewrite_query(data: State) -> State:
        return {"query": f"query: {data.query}"}

    async def analyzer_one(data: State) -> State:
        return StateUpdate(query=f"analyzed: {data.query}")

    async def retriever_one(data: State) -> State:
        return {"docs": ["doc1", "doc2"]}

    async def retriever_two(data: State) -> State:
        await asyncio.sleep(0.1)
        return {"docs": ["doc3", "doc4"]}

    async def qa(data: State) -> State:
        return {"answer": ",".join(data.docs)}

    async def decider(data: State) -> str:
        assert isinstance(data, State)
        return "retriever_two"

    workflow = StateGraph(State, input_schema=Input, output_schema=Output)

    workflow.add_node("rewrite_query", rewrite_query)
    workflow.add_node("analyzer_one", analyzer_one)
    workflow.add_node("retriever_one", retriever_one)
    workflow.add_node("retriever_two", retriever_two)
    workflow.add_node("qa", qa)

    workflow.set_entry_point("rewrite_query")
    workflow.add_edge("rewrite_query", "analyzer_one")
    workflow.add_edge("analyzer_one", "retriever_one")
    workflow.add_conditional_edges(
        "rewrite_query", decider, {"retriever_two": "retriever_two"}
    )
    workflow.add_edge(["retriever_one", "retriever_two"], "qa")
    workflow.set_finish_point("qa")

    app = workflow.compile()

    with pytest.raises(ValidationError):
        await app.ainvoke({"query": {}})

    assert await app.ainvoke({"query": "what is weather in sf"}) == {
        "docs": ["doc1", "doc2", "doc3", "doc4"],
        "answer": "doc1,doc2,doc3,doc4",
    }

    assert [c async for c in app.astream({"query": "what is weather in sf"})] == [
        {"rewrite_query": {"query": "query: what is weather in sf"}},
        {"analyzer_one": {"query": "analyzed: query: what is weather in sf"}},
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {"qa": {"answer": "doc1,doc2,doc3,doc4"}},
    ]

    app_w_interrupt = workflow.compile(
        checkpointer=async_checkpointer,
        interrupt_after=["retriever_one"],
    )
    config = {"configurable": {"thread_id": "1"}}

    assert [
        c
        async for c in app_w_interrupt.astream(
            {"query": "what is weather in sf"}, config
        )
    ] == [
        {"rewrite_query": {"query": "query: what is weather in sf"}},
        {"analyzer_one": {"query": "analyzed: query: what is weather in sf"}},
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {"__interrupt__": ()},
    ]

    assert [c async for c in app_w_interrupt.astream(None, config)] == [
        {"qa": {"answer": "doc1,doc2,doc3,doc4"}},
    ]

    assert await app_w_interrupt.aget_state(config) == StateSnapshot(
        values={
            "query": "analyzed: query: what is weather in sf",
            "answer": "doc1,doc2,doc3,doc4",
            "docs": ["doc1", "doc2", "doc3", "doc4"],
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
        metadata={
            "parents": {},
            "source": "loop",
            "step": 4,
        },
        created_at=AnyStr(),
        parent_config=(
            {
                "configurable": {
                    "thread_id": "1",
                    "checkpoint_ns": "",
                    "checkpoint_id": AnyStr(),
                }
            }
        ),
        interrupts=(),
    )

    assert await app_w_interrupt.aupdate_state(
        config, {"docs": ["doc5"]}, as_node="rewrite_query"
    ) == {
        "configurable": {
            "thread_id": "1",
            "checkpoint_id": AnyStr(),
            "checkpoint_ns": "",
        }
    }


async def test_in_one_fan_out_state_graph_waiting_edge_custom_state_class_pydantic2(
    snapshot: SnapshotAssertion, async_checkpointer: BaseCheckpointSaver
) -> None:
    def sorted_add(x: list[str], y: list[str] | list[tuple[str, str]]) -> list[str]:
        if isinstance(y[0], tuple):
            for rem, _ in y:
                x.remove(rem)
            y = [t[1] for t in y]
        return sorted(operator.add(x, y))

    class InnerObject(BaseModel):
        yo: int

    class State(BaseModel):
        query: str
        inner: InnerObject
        answer: str | None = None
        docs: Annotated[list[str], sorted_add]

    class StateUpdate(BaseModel):
        query: str | None = None
        answer: str | None = None
        docs: list[str] | None = None

    async def rewrite_query(data: State) -> State:
        return {"query": f"query: {data.query}"}

    async def analyzer_one(data: State) -> State:
        return StateUpdate(query=f"analyzed: {data.query}")

    async def retriever_one(data: State) -> State:
        return {"docs": ["doc1", "doc2"]}

    async def retriever_two(data: State) -> State:
        await asyncio.sleep(0.1)
        return {"docs": ["doc3", "doc4"]}

    async def qa(data: State) -> State:
        return {"answer": ",".join(data.docs)}

    async def decider(data: State) -> str:
        assert isinstance(data, State)
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
    workflow.add_conditional_edges(
        "rewrite_query", decider, {"retriever_two": "retriever_two"}
    )
    workflow.add_edge(["retriever_one", "retriever_two"], "qa")
    workflow.set_finish_point("qa")

    app = workflow.compile()

    if isinstance(async_checkpointer, InMemorySaver):
        assert app.get_graph().draw_mermaid(with_styles=False) == snapshot
        assert app.get_input_jsonschema() == snapshot
        assert app.get_output_jsonschema() == snapshot

    with pytest.raises(ValidationError):
        await app.ainvoke({"query": {}})

    assert await app.ainvoke(
        {"query": "what is weather in sf", "inner": {"yo": 1}}
    ) == {
        "query": "analyzed: query: what is weather in sf",
        "docs": ["doc1", "doc2", "doc3", "doc4"],
        "answer": "doc1,doc2,doc3,doc4",
        "inner": {"yo": 1},
    }

    assert [
        c
        async for c in app.astream(
            {"query": "what is weather in sf", "inner": {"yo": 1}}
        )
    ] == [
        {"rewrite_query": {"query": "query: what is weather in sf"}},
        {"analyzer_one": {"query": "analyzed: query: what is weather in sf"}},
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {"qa": {"answer": "doc1,doc2,doc3,doc4"}},
    ]

    app_w_interrupt = workflow.compile(
        checkpointer=async_checkpointer,
        interrupt_after=["retriever_one"],
    )
    config = {"configurable": {"thread_id": "1"}}

    assert [
        c
        async for c in app_w_interrupt.astream(
            {"query": "what is weather in sf", "inner": {"yo": 1}}, config
        )
    ] == [
        {"rewrite_query": {"query": "query: what is weather in sf"}},
        {"analyzer_one": {"query": "analyzed: query: what is weather in sf"}},
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {"__interrupt__": ()},
    ]

    assert [c async for c in app_w_interrupt.astream(None, config)] == [
        {"qa": {"answer": "doc1,doc2,doc3,doc4"}},
    ]

    assert await app_w_interrupt.aupdate_state(
        config, {"docs": ["doc5"]}, as_node="rewrite_query"
    ) == {
        "configurable": {
            "thread_id": "1",
            "checkpoint_id": AnyStr(),
            "checkpoint_ns": "",
        }
    }


async def test_in_one_fan_out_state_graph_waiting_edge_plus_regular(
    async_checkpointer: BaseCheckpointSaver,
) -> None:
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

    async def rewrite_query(data: State) -> State:
        return {"query": f"query: {data['query']}"}

    async def analyzer_one(data: State) -> State:
        await asyncio.sleep(0.1)
        return {"query": f"analyzed: {data['query']}"}

    async def retriever_one(data: State) -> State:
        return {"docs": ["doc1", "doc2"]}

    async def retriever_two(data: State) -> State:
        await asyncio.sleep(0.2)
        return {"docs": ["doc3", "doc4"]}

    async def qa(data: State) -> State:
        return {"answer": ",".join(data["docs"])}

    workflow = StateGraph(State)

    workflow.add_node("rewrite_query", rewrite_query)
    workflow.add_node("analyzer_one", analyzer_one)
    workflow.add_node("retriever_one", retriever_one)
    workflow.add_node("retriever_two", retriever_two)
    workflow.add_node("qa", qa)

    workflow.set_entry_point("rewrite_query")
    workflow.add_edge("rewrite_query", "analyzer_one")
    workflow.add_edge("analyzer_one", "retriever_one")
    workflow.add_edge("rewrite_query", "retriever_two")
    workflow.add_edge(["retriever_one", "retriever_two"], "qa")
    workflow.set_finish_point("qa")

    # silly edge, to make sure having been triggered before doesn't break
    # semantics of named barrier (== waiting edges)
    workflow.add_edge("rewrite_query", "qa")

    app = workflow.compile()

    assert await app.ainvoke({"query": "what is weather in sf"}) == {
        "query": "analyzed: query: what is weather in sf",
        "docs": ["doc1", "doc2", "doc3", "doc4"],
        "answer": "doc1,doc2,doc3,doc4",
    }

    assert [c async for c in app.astream({"query": "what is weather in sf"})] == [
        {"rewrite_query": {"query": "query: what is weather in sf"}},
        {"qa": {"answer": ""}},
        {"analyzer_one": {"query": "analyzed: query: what is weather in sf"}},
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {"qa": {"answer": "doc1,doc2,doc3,doc4"}},
    ]

    app_w_interrupt = workflow.compile(
        checkpointer=async_checkpointer,
        interrupt_after=["retriever_one"],
    )
    config = {"configurable": {"thread_id": "1"}}

    assert [
        c
        async for c in app_w_interrupt.astream(
            {"query": "what is weather in sf"}, config
        )
    ] == [
        {"rewrite_query": {"query": "query: what is weather in sf"}},
        {"qa": {"answer": ""}},
        {"analyzer_one": {"query": "analyzed: query: what is weather in sf"}},
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {"__interrupt__": ()},
    ]

    assert [c async for c in app_w_interrupt.astream(None, config)] == [
        {"qa": {"answer": "doc1,doc2,doc3,doc4"}},
    ]


@pytest.mark.parametrize("with_cache", [True, False])
async def test_in_one_fan_out_state_graph_waiting_edge_multiple(
    with_cache: bool, cache: BaseCache
) -> None:
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

    rewrite_query_count = 0

    async def rewrite_query(data: State) -> State:
        nonlocal rewrite_query_count
        rewrite_query_count += 1
        return {"query": f"query: {data['query']}"}

    async def analyzer_one(data: State) -> State:
        return {"query": f"analyzed: {data['query']}"}

    async def retriever_one(data: State) -> State:
        return {"docs": ["doc1", "doc2"]}

    async def retriever_two(data: State) -> State:
        await asyncio.sleep(0.1)
        return {"docs": ["doc3", "doc4"]}

    async def qa(data: State) -> State:
        return {"answer": ",".join(data["docs"])}

    async def decider(data: State) -> None:
        return None

    def decider_cond(data: State) -> str:
        if data["query"].count("analyzed") > 1:
            return "qa"
        else:
            return "rewrite_query"

    workflow = StateGraph(State)

    workflow.add_node(
        "rewrite_query",
        rewrite_query,
        cache_policy=CachePolicy() if with_cache else None,
    )
    workflow.add_node("analyzer_one", analyzer_one)
    workflow.add_node("retriever_one", retriever_one)
    workflow.add_node("retriever_two", retriever_two)
    workflow.add_node("decider", decider)
    workflow.add_node("qa", qa)

    workflow.set_entry_point("rewrite_query")
    workflow.add_edge("rewrite_query", "analyzer_one")
    workflow.add_edge("analyzer_one", "retriever_one")
    workflow.add_edge("rewrite_query", "retriever_two")
    workflow.add_edge(["retriever_one", "retriever_two"], "decider")
    workflow.add_conditional_edges("decider", decider_cond)
    workflow.set_finish_point("qa")

    app = workflow.compile(cache=cache)

    assert await app.ainvoke({"query": "what is weather in sf"}) == {
        "query": "analyzed: query: analyzed: query: what is weather in sf",
        "answer": "doc1,doc1,doc2,doc2,doc3,doc3,doc4,doc4",
        "docs": ["doc1", "doc1", "doc2", "doc2", "doc3", "doc3", "doc4", "doc4"],
    }
    assert rewrite_query_count == 2

    assert [c async for c in app.astream({"query": "what is weather in sf"})] == [
        {
            "rewrite_query": {"query": "query: what is weather in sf"},
            "__metadata__": {"cached": True},
        }
        if with_cache
        else {"rewrite_query": {"query": "query: what is weather in sf"}},
        {"analyzer_one": {"query": "analyzed: query: what is weather in sf"}},
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {"decider": None},
        {
            "rewrite_query": {"query": "query: analyzed: query: what is weather in sf"},
            "__metadata__": {"cached": True},
        }
        if with_cache
        else {
            "rewrite_query": {"query": "query: analyzed: query: what is weather in sf"}
        },
        {
            "analyzer_one": {
                "query": "analyzed: query: analyzed: query: what is weather in sf"
            }
        },
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {"decider": None},
        {"qa": {"answer": "doc1,doc1,doc2,doc2,doc3,doc3,doc4,doc4"}},
    ]
    assert rewrite_query_count == 2 if with_cache else 4

    # clear the cache
    if with_cache:
        await app.aclear_cache()

        assert await app.ainvoke({"query": "what is weather in sf"}) == {
            "query": "analyzed: query: analyzed: query: what is weather in sf",
            "answer": "doc1,doc1,doc2,doc2,doc3,doc3,doc4,doc4",
            "docs": ["doc1", "doc1", "doc2", "doc2", "doc3", "doc3", "doc4", "doc4"],
        }
        assert rewrite_query_count == 4


async def test_in_one_fan_out_state_graph_waiting_edge_multiple_cond_edge() -> None:
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

    async def rewrite_query(data: State) -> State:
        return {"query": f"query: {data['query']}"}

    async def retriever_picker(data: State) -> list[str]:
        return ["analyzer_one", "retriever_two"]

    async def analyzer_one(data: State) -> State:
        return {"query": f"analyzed: {data['query']}"}

    async def retriever_one(data: State) -> State:
        return {"docs": ["doc1", "doc2"]}

    async def retriever_two(data: State) -> State:
        await asyncio.sleep(0.1)
        return {"docs": ["doc3", "doc4"]}

    async def qa(data: State) -> State:
        return {"answer": ",".join(data["docs"])}

    async def decider(data: State) -> None:
        return None

    def decider_cond(data: State) -> str:
        if data["query"].count("analyzed") > 1:
            return "qa"
        else:
            return "rewrite_query"

    workflow = StateGraph(State)

    workflow.add_node("rewrite_query", rewrite_query)
    workflow.add_node("analyzer_one", analyzer_one)
    workflow.add_node("retriever_one", retriever_one)
    workflow.add_node("retriever_two", retriever_two)
    workflow.add_node("decider", decider)
    workflow.add_node("qa", qa)

    workflow.set_entry_point("rewrite_query")
    workflow.add_conditional_edges("rewrite_query", retriever_picker)
    workflow.add_edge("analyzer_one", "retriever_one")
    workflow.add_edge(["retriever_one", "retriever_two"], "decider")
    workflow.add_conditional_edges("decider", decider_cond)
    workflow.set_finish_point("qa")

    app = workflow.compile()

    assert await app.ainvoke({"query": "what is weather in sf"}) == {
        "query": "analyzed: query: analyzed: query: what is weather in sf",
        "answer": "doc1,doc1,doc2,doc2,doc3,doc3,doc4,doc4",
        "docs": ["doc1", "doc1", "doc2", "doc2", "doc3", "doc3", "doc4", "doc4"],
    }

    assert [c async for c in app.astream({"query": "what is weather in sf"})] == [
        {"rewrite_query": {"query": "query: what is weather in sf"}},
        {"analyzer_one": {"query": "analyzed: query: what is weather in sf"}},
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {"decider": None},
        {"rewrite_query": {"query": "query: analyzed: query: what is weather in sf"}},
        {
            "analyzer_one": {
                "query": "analyzed: query: analyzed: query: what is weather in sf"
            }
        },
        {"retriever_two": {"docs": ["doc3", "doc4"]}},
        {"retriever_one": {"docs": ["doc1", "doc2"]}},
        {"decider": None},
        {"qa": {"answer": "doc1,doc1,doc2,doc2,doc3,doc3,doc4,doc4"}},
    ]


async def test_nested_graph(snapshot: SnapshotAssertion) -> None:
    def never_called_fn(state: Any):
        assert 0, "This function should never be called"

    never_called = RunnableLambda(never_called_fn)

    class InnerState(TypedDict):
        my_key: str
        my_other_key: str

    def up(state: InnerState):
        return {"my_key": state["my_key"] + " there", "my_other_key": state["my_key"]}

    inner = StateGraph(InnerState)
    inner.add_node("up", up)
    inner.set_entry_point("up")
    inner.set_finish_point("up")

    class State(TypedDict):
        my_key: str
        never_called: Any

    async def side(state: State):
        return {"my_key": state["my_key"] + " and back again"}

    graph = StateGraph(State)
    graph.add_node("inner", inner.compile())
    graph.add_node("side", side)
    graph.set_entry_point("inner")
    graph.add_edge("inner", "side")
    graph.set_finish_point("side")

    app = graph.compile()

    assert await app.ainvoke({"my_key": "my value", "never_called": never_called}) == {
        "my_key": "my value there and back again",
        "never_called": never_called,
    }
    assert [
        chunk
        async for chunk in app.astream(
            {"my_key": "my value", "never_called": never_called}
        )
    ] == [
        {"inner": {"my_key": "my value there"}},
        {"side": {"my_key": "my value there and back again"}},
    ]
    assert [
        chunk
        async for chunk in app.astream(
            {"my_key": "my value", "never_called": never_called}, stream_mode="values"
        )
    ] == [
        {"my_key": "my value", "never_called": never_called},
        {"my_key": "my value there", "never_called": never_called},
        {"my_key": "my value there and back again", "never_called": never_called},
    ]
    times_called = 0
    async for event in app.astream_events(
        {"my_key": "my value", "never_called": never_called},
        version="v2",
        config={"run_id": UUID(int=0)},
        stream_mode="values",
    ):
        if event["event"] == "on_chain_end" and event["run_id"] == str(UUID(int=0)):
            times_called += 1
            assert event["data"] == {
                "output": {
                    "my_key": "my value there and back again",
                    "never_called": never_called,
                }
            }
    assert times_called == 1
    times_called = 0
    async for event in app.astream_events(
        {"my_key": "my value", "never_called": never_called},
        version="v2",
        config={"run_id": UUID(int=0)},
    ):
        if event["event"] == "on_chain_end" and event["run_id"] == str(UUID(int=0)):
            times_called += 1
            assert event["data"] == {
                "output": {
                    "my_key": "my value there and back again",
                    "never_called": never_called,
                }
            }
    assert times_called == 1

    chain = app | RunnablePassthrough()

    assert await chain.ainvoke(
        {"my_key": "my value", "never_called": never_called}
    ) == {
        "my_key": "my value there and back again",
        "never_called": never_called,
    }
    assert [
        chunk
        async for chunk in chain.astream(
            {"my_key": "my value", "never_called": never_called}
        )
    ] == [
        {"inner": {"my_key": "my value there"}},
        {"side": {"my_key": "my value there and back again"}},
    ]
    times_called = 0
    async for event in chain.astream_events(
        {"my_key": "my value", "never_called": never_called},
        version="v2",
        config={"run_id": UUID(int=0)},
    ):
        if event["event"] == "on_chain_end" and event["run_id"] == str(UUID(int=0)):
            times_called += 1
            assert event["data"] == {
                "output": {"side": {"my_key": "my value there and back again"}}
            }
    assert times_called == 1


async def test_subgraph_checkpoint_true(
    async_checkpointer: BaseCheckpointSaver, durability: Durability
) -> None:
    class InnerState(TypedDict):
        my_key: Annotated[str, operator.add]
        my_other_key: str

    def inner_1(state: InnerState):
        return {"my_key": " got here", "my_other_key": state["my_key"]}

    def inner_2(state: InnerState):
        return {"my_key": " and there"}

    inner = StateGraph(InnerState)
    inner.add_node("inner_1", inner_1)
    inner.add_node("inner_2", inner_2)
    inner.add_edge("inner_1", "inner_2")
    inner.set_entry_point("inner_1")
    inner.set_finish_point("inner_2")

    class State(TypedDict):
        my_key: str

    graph = StateGraph(State)
    graph.add_node("inner", inner.compile(checkpointer=True))
    graph.add_edge(START, "inner")
    graph.add_conditional_edges(
        "inner", lambda s: "inner" if s["my_key"].count("there") < 2 else END
    )

    app = graph.compile(checkpointer=async_checkpointer)

    config = {"configurable": {"thread_id": "2"}}
    assert [
        c
        async for c in app.astream(
            {"my_key": ""},
            config,
            subgraphs=True,
            durability=durability,
        )
    ] == [
        (("inner",), {"inner_1": {"my_key": " got here", "my_other_key": ""}}),
        (("inner",), {"inner_2": {"my_key": " and there"}}),
        ((), {"inner": {"my_key": " got here and there"}}),
        (
            ("inner",),
            {
                "inner_1": {
                    "my_key": " got here",
                    "my_other_key": " got here and there got here and there",
                }
            },
        ),
        (("inner",), {"inner_2": {"my_key": " and there"}}),
        (
            (),
            {
                "inner": {
                    "my_key": " got here and there got here and there got here and there"
                }
            },
        ),
    ]


async def test_subgraph_durability_inherited(
    durability: Durability,
) -> None:
    async_checkpointer = InMemorySaver()

    class InnerState(TypedDict):
        my_key: Annotated[str, operator.add]
        my_other_key: str

    def inner_1(state: InnerState):
        return {"my_key": " got here", "my_other_key": state["my_key"]}

    def inner_2(state: InnerState):
        return {"my_key": " and there"}

    inner = StateGraph(InnerState)
    inner.add_node("inner_1", inner_1)
    inner.add_node("inner_2", inner_2)
    inner.add_edge("inner_1", "inner_2")
    inner.set_entry_point("inner_1")
    inner.set_finish_point("inner_2")

    class State(TypedDict):
        my_key: str

    inner_app = inner.compile(checkpointer=async_checkpointer)
    graph = StateGraph(State)
    graph.add_node("inner", inner_app)
    graph.add_edge(START, "inner")
    graph.add_conditional_edges(
        "inner", lambda s: "inner" if s["my_key"].count("there") < 2 else END
    )
    app = graph.compile(checkpointer=async_checkpointer)
    thread_id = str(uuid.uuid4())
    config = {"configurable": {"thread_id": thread_id}}
    await app.ainvoke({"my_key": ""}, config, subgraphs=True, durability=durability)
    if durability != "exit":
        checkpoints = list(async_checkpointer.list(config))
        assert len(checkpoints) == 12
    else:
        checkpoints = list(async_checkpointer.list(config))
        assert len(checkpoints) == 1


@NEEDS_CONTEXTVARS
async def test_subgraph_checkpoint_true_interrupt(
    async_checkpointer: BaseCheckpointSaver, durability: Durability
) -> None:
    # Define subgraph
    class SubgraphState(TypedDict):
        # note that none of these keys are shared with the parent graph state
        bar: str
        baz: str

    def subgraph_node_1(state: SubgraphState):
        baz_value = interrupt("Provide baz value")
        return {"baz": baz_value}

    def subgraph_node_2(state: SubgraphState):
        return {"bar": state["bar"] + state["baz"]}

    subgraph_builder = StateGraph(SubgraphState)
    subgraph_builder.add_node(subgraph_node_1)
    subgraph_builder.add_node(subgraph_node_2)
    subgraph_builder.add_edge(START, "subgraph_node_1")
    subgraph_builder.add_edge("subgraph_node_1", "subgraph_node_2")
    subgraph = subgraph_builder.compile(checkpointer=True)

    class ParentState(TypedDict):
        foo: str

    def node_1(state: ParentState):
        return {"foo": "hi! " + state["foo"]}

    async def node_2(state: ParentState, config: RunnableConfig):
        response = await subgraph.ainvoke({"bar": state["foo"]})
        return {"foo": response["bar"]}

    builder = StateGraph(ParentState)
    builder.add_node("node_1", node_1)
    builder.add_node("node_2", node_2)
    builder.add_edge(START, "node_1")
    builder.add_edge("node_1", "node_2")

    graph = builder.compile(checkpointer=async_checkpointer)
    config = {"configurable": {"thread_id": "1"}}

    assert await graph.ainvoke({"foo": "foo"}, config, durability=durability) == {
        "foo": "hi! foo",
        "__interrupt__": [
            Interrupt(
                value="Provide baz value",
                id=AnyStr(),
            )
        ],
    }
    assert (await graph.aget_state(config, subgraphs=True)).tasks[0].state.values == {
        "bar": "hi! foo"
    }
    assert await graph.ainvoke(
        Command(resume="baz"), config, durability=durability
    ) == {"foo": "hi! foobaz"}


async def test_stream_subgraphs_during_execution(
    async_checkpointer: BaseCheckpointSaver,
) -> None:
    class InnerState(TypedDict):
        my_key: Annotated[str, operator.add]
        my_other_key: str

    async def inner_1(state: InnerState):
        return {"my_key": "got here", "my_other_key": state["my_key"]}

    async def inner_2(state: InnerState):
        await asyncio.sleep(0.5)
        return {
            "my_key": " and there",
            "my_other_key": state["my_key"],
        }

    inner = StateGraph(InnerState)
    inner.add_node("inner_1", inner_1)
    inner.add_node("inner_2", inner_2)
    inner.add_edge("inner_1", "inner_2")
    inner.set_entry_point("inner_1")
    inner.set_finish_point("inner_2")

    class State(TypedDict):
        my_key: Annotated[str, operator.add]

    async def outer_1(state: State):
        await asyncio.sleep(0.2)
        return {"my_key": " and parallel"}

    async def outer_2(state: State):
        return {"my_key": " and back again"}

    graph = StateGraph(State)
    graph.add_node("inner", inner.compile())
    graph.add_node("outer_1", outer_1)
    graph.add_node("outer_2", outer_2)

    graph.add_edge(START, "inner")
    graph.add_edge(START, "outer_1")
    graph.add_edge(["inner", "outer_1"], "outer_2")
    graph.add_edge("outer_2", END)

    app = graph.compile(checkpointer=async_checkpointer)

    start = perf_counter()
    chunks: list[tuple[float, Any]] = []
    config = {"configurable": {"thread_id": "2"}}
    async for c in app.astream({"my_key": ""}, config, subgraphs=True):
        chunks.append((round(perf_counter() - start, 1), c))
    for idx in range(len(chunks)):
        elapsed, c = chunks[idx]
        chunks[idx] = (round(elapsed - chunks[0][0], 1), c)

    assert chunks == [
        # arrives before "inner" finishes
        (
            FloatBetween(0.0, 0.1),
            (
                (AnyStr("inner:"),),
                {"inner_1": {"my_key": "got here", "my_other_key": ""}},
            ),
        ),
        (FloatBetween(0.2, 0.4), ((), {"outer_1": {"my_key": " and parallel"}})),
        (
            FloatBetween(0.5, 0.8),
            (
                (AnyStr("inner:"),),
                {"inner_2": {"my_key": " and there", "my_other_key": "got here"}},
            ),
        ),
        (FloatBetween(0.5, 0.8), ((), {"inner": {"my_key": "got here and there"}})),
        (FloatBetween(0.5, 0.8), ((), {"outer_2": {"my_key": " and back again"}})),
    ]


@NEEDS_CONTEXTVARS
async def test_stream_buffering_single_node(
    async_checkpointer: BaseCheckpointSaver,
) -> None:
    class State(TypedDict):
        my_key: Annotated[str, operator.add]

    async def node(state: State, writer: StreamWriter):
        writer("Before sleep")
        await asyncio.sleep(0.2)
        writer("After sleep")
        return {"my_key": "got here"}

    builder = StateGraph(State)
    builder.add_node("node", node)
    builder.add_edge(START, "node")
    builder.add_edge("node", END)

    graph = builder.compile(checkpointer=async_checkpointer)

    start = perf_counter()
    chunks: list[tuple[float, Any]] = []
    config = {"configurable": {"thread_id": "2"}}
    async for c in graph.astream({"my_key": ""}, config, stream_mode="custom"):
        chunks.append((round(perf_counter() - start, 1), c))

    assert chunks == [
        (FloatBetween(0.0, 0.1), "Before sleep"),
        (FloatBetween(0.2, 0.3), "After sleep"),
    ]


async def test_nested_graph_interrupts_parallel(
    async_checkpointer: BaseCheckpointSaver, durability: Durability
) -> None:
    class InnerState(TypedDict):
        my_key: Annotated[str, operator.add]
        my_other_key: str

    async def inner_1(state: InnerState):
        await asyncio.sleep(0.1)
        return {"my_key": "got here", "my_other_key": state["my_key"]}

    async def inner_2(state: InnerState):
        return {
            "my_key": " and there",
            "my_other_key": state["my_key"],
        }

    inner = StateGraph(InnerState)
    inner.add_node("inner_1", inner_1)
    inner.add_node("inner_2", inner_2)
    inner.add_edge("inner_1", "inner_2")
    inner.set_entry_point("inner_1")
    inner.set_finish_point("inner_2")

    class State(TypedDict):
        my_key: Annotated[str, operator.add]

    async def outer_1(state: State):
        return {"my_key": " and parallel"}

    async def outer_2(state: State):
        return {"my_key": " and back again"}

    graph = StateGraph(State)
    graph.add_node(
        "inner",
        inner.compile(interrupt_before=["inner_2"]),
    )
    graph.add_node("outer_1", outer_1)
    graph.add_node("outer_2", outer_2)

    graph.add_edge(START, "inner")
    graph.add_edge(START, "outer_1")
    graph.add_edge(["inner", "outer_1"], "outer_2")
    graph.set_finish_point("outer_2")

    app = graph.compile(checkpointer=async_checkpointer)

    # test invoke w/ nested interrupt
    config = {"configurable": {"thread_id": "1"}}
    assert await app.ainvoke({"my_key": ""}, config, durability=durability) == {
        "my_key": " and parallel",
    }

    assert await app.ainvoke(None, config, durability=durability) == {
        "my_key": "got here and there and parallel and back again",
    }

    # below combo of assertions is asserting two things
    # - outer_1 finishes before inner interrupts (because we see its output in stream, which only happens after node finishes)
    # - the writes of outer are persisted in 1st call and used in 2nd call, ie outer isn't called again (because we dont see outer_1 output again in 2nd stream)
    # test stream updates w/ nested interrupt
    config = {"configurable": {"thread_id": "2"}}
    assert [
        c
        async for c in app.astream(
            {"my_key": ""},
            config,
            subgraphs=True,
            durability=durability,
        )
    ] == [
        # we got to parallel node first
        ((), {"outer_1": {"my_key": " and parallel"}}),
        (
            (AnyStr("inner:"),),
            {"inner_1": {"my_key": "got here", "my_other_key": ""}},
        ),
        ((), {"__interrupt__": ()}),
    ]
    assert [c async for c in app.astream(None, config, durability=durability)] == [
        {"outer_1": {"my_key": " and parallel"}, "__metadata__": {"cached": True}},
        {"inner": {"my_key": "got here and there"}},
        {"outer_2": {"my_key": " and back again"}},
    ]

    # test stream values w/ nested interrupt
    config = {"configurable": {"thread_id": "3"}}
    assert [
        c
        async for c in app.astream(
            {"my_key": ""},
            config,
            stream_mode="values",
            durability=durability,
        )
    ] == [
        {"my_key": ""},
        {"my_key": " and parallel"},
    ]
    assert [
        c
        async for c in app.astream(
            None, config, stream_mode="values", durability=durability
        )
    ] == [
        {"my_key": ""},
        {"my_key": "got here and there and parallel"},
        {"my_key": "got here and there and parallel and back again"},
    ]

    # # test interrupts BEFORE the parallel node
    app = graph.compile(checkpointer=async_checkpointer, interrupt_before=["outer_1"])
    config = {"configurable": {"thread_id": "4"}}
    assert [
        c
        async for c in app.astream(
            {"my_key": ""},
            config,
            stream_mode="values",
            durability=durability,
        )
    ] == [
        {"my_key": ""},
    ]
    # while we're waiting for the node w/ interrupt inside to finish
    assert [
        c
        async for c in app.astream(
            None, config, stream_mode="values", durability=durability
        )
    ] == [
        {"my_key": ""},
        {"my_key": " and parallel"},
    ]
    assert [
        c
        async for c in app.astream(
            None, config, stream_mode="values", durability=durability
        )
    ] == [
        {"my_key": ""},
        {"my_key": "got here and there and parallel"},
        {"my_key": "got here and there and parallel and back again"},
    ]

    # test interrupts AFTER the parallel node
    app = graph.compile(checkpointer=async_checkpointer, interrupt_after=["outer_1"])
    config = {"configurable": {"thread_id": "5"}}
    assert [
        c
        async for c in app.astream(
            {"my_key": ""},
            config,
            stream_mode="values",
            durability=durability,
        )
    ] == [
        {"my_key": ""},
        {"my_key": " and parallel"},
    ]
    assert [
        c
        async for c in app.astream(
            None, config, stream_mode="values", durability=durability
        )
    ] == [
        {"my_key": ""},
        {"my_key": "got here and there and parallel"},
    ]
    assert [
        c
        async for c in app.astream(
            None, config, stream_mode="values", durability=durability
        )
    ] == [
        {"my_key": "got here and there and parallel"},
        {"my_key": "got here and there and parallel and back again"},
    ]


async def test_doubly_nested_graph_interrupts(
    async_checkpointer: BaseCheckpointSaver, durability: Durability
) -> None:
    class State(TypedDict):
        my_key: str

    class ChildState(TypedDict):
        my_key: str

    class GrandChildState(TypedDict):
        my_key: str

    async def grandchild_1(state: ChildState):
        return {"my_key": state["my_key"] + " here"}

    async def grandchild_2(state: ChildState):
        return {
            "my_key": state["my_key"] + " and there",
        }

    grandchild = StateGraph(GrandChildState)
    grandchild.add_node("grandchild_1", grandchild_1)
    grandchild.add_node("grandchild_2", grandchild_2)
    grandchild.add_edge("grandchild_1", "grandchild_2")
    grandchild.set_entry_point("grandchild_1")
    grandchild.set_finish_point("grandchild_2")

    child = StateGraph(ChildState)
    child.add_node(
        "child_1",
        grandchild.compile(interrupt_before=["grandchild_2"]),
    )
    child.set_entry_point("child_1")
    child.set_finish_point("child_1")

    async def parent_1(state: State):
        return {"my_key": "hi " + state["my_key"]}

    async def parent_2(state: State):
        return {"my_key": state["my_key"] + " and back again"}

    graph = StateGraph(State)
    graph.add_node("parent_1", parent_1)
    graph.add_node("child", child.compile())
    graph.add_node("parent_2", parent_2)
    graph.set_entry_point("parent_1")
    graph.add_edge("parent_1", "child")
    graph.add_edge("child", "parent_2")
    graph.set_finish_point("parent_2")

    app = graph.compile(checkpointer=async_checkpointer)

    # test invoke w/ nested interrupt
    config = {"configurable": {"thread_id": "1"}}
    assert await app.ainvoke({"my_key": "my value"}, config, durability=durability) == {
        "my_key": "hi my value",
    }

    assert await app.ainvoke(None, config, durability=durability) == {
        "my_key": "hi my value here and there and back again",
    }

    # test stream updates w/ nested interrupt
    nodes: list[str] = []
    config = {
        "configurable": {"thread_id": "2", CONFIG_KEY_NODE_FINISHED: nodes.append}
    }
    assert [
        c
        async for c in app.astream(
            {"my_key": "my value"}, config, durability=durability
        )
    ] == [
        {"parent_1": {"my_key": "hi my value"}},
        {"__interrupt__": ()},
    ]
    assert nodes == ["parent_1", "grandchild_1"]
    assert [c async for c in app.astream(None, config, durability=durability)] == [
        {"child": {"my_key": "hi my value here and there"}},
        {"parent_2": {"my_key": "hi my value here and there and back again"}},
    ]
    assert nodes == [
        "parent_1",
        "grandchild_1",
        "grandchild_2",
        "child_1",
        "child",
        "parent_2",
    ]

    # test stream values w/ nested interrupt
    config = {"configurable": {"thread_id": "3"}}
    assert [
        c
        async for c in app.astream(
            {"my_key": "my value"},
            config,
            stream_mode="values",
            durability=durability,
        )
    ] == [
        {"my_key": "my value"},
        {"my_key": "hi my value"},
    ]
    assert [
        c
        async for c in app.astream(
            None, config, stream_mode="values", durability=durability
        )
    ] == [
        {"my_key": "hi my value"},
        {"my_key": "hi my value here and there"},
        {"my_key": "hi my value here and there and back again"},
    ]


async def test_checkpoint_metadata(async_checkpointer: BaseCheckpointSaver) -> None:
    """This test verifies that a run's configurable fields are merged with the
    previous checkpoint config for each step in the run.
    """
    # set up test
    from langchain_core.language_models.fake_chat_models import (
        FakeMessagesListChatModel,
    )
    from langchain_core.messages import AIMessage, AnyMessage
    from langchain_core.prompts import ChatPromptTemplate
    from langchain_core.tools import tool

    # graph state
    class BaseState(TypedDict):
        messages: Annotated[list[AnyMessage], add_messages]

    # initialize graph nodes
    @tool()
    def search_api(query: str) -> str:
        """Searches the API for the query."""
        return f"result for {query}"

    tools = [search_api]

    prompt = ChatPromptTemplate.from_messages(
        [
            ("system", "You are a nice assistant."),
            ("placeholder", "{messages}"),
        ]
    )

    model = FakeMessagesListChatModel(
        responses=[
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
            AIMessage(content="answer"),
        ]
    )

    def agent(state: BaseState, config: RunnableConfig) -> BaseState:
        formatted = prompt.invoke(state)
        response = model.invoke(formatted)
        return {"messages": response}

    def should_continue(data: BaseState) -> str:
        # Logic to decide whether to continue in the loop or exit
        if not data["messages"][-1].tool_calls:
            return "exit"
        else:
            return "continue"

    # define graphs w/ and w/o interrupt
    workflow = StateGraph(BaseState)
    workflow.add_node("agent", agent)
    workflow.add_node("tools", ToolNode(tools))
    workflow.set_entry_point("agent")
    workflow.add_conditional_edges(
        "agent", should_continue, {"continue": "tools", "exit": END}
    )
    workflow.add_edge("tools", "agent")

    # graph w/o interrupt
    app = workflow.compile(checkpointer=async_checkpointer)

    # graph w/ interrupt
    app_w_interrupt = workflow.compile(
        checkpointer=async_checkpointer, interrupt_before=["tools"]
    )

    # assertions

    # invoke graph w/o interrupt
    await app.ainvoke(
        {"messages": ["what is weather in sf"]},
        {
            "configurable": {
                "thread_id": "1",
                "test_config_1": "foo",
                "test_config_2": "bar",
            },
        },
    )

    config = {"configurable": {"thread_id": "1"}}

    # assert that checkpoint metadata contains the run's configurable fields
    chkpnt_metadata_1 = (await async_checkpointer.aget_tuple(config)).metadata
    assert chkpnt_metadata_1["test_config_1"] == "foo"
    assert chkpnt_metadata_1["test_config_2"] == "bar"

    # Verify that all checkpoint metadata have the expected keys. This check
    # is needed because a run may have an arbitrary number of steps depending
    # on how the graph is constructed.
    chkpnt_tuples_1 = async_checkpointer.alist(config)
    async for chkpnt_tuple in chkpnt_tuples_1:
        assert chkpnt_tuple.metadata["test_config_1"] == "foo"
        assert chkpnt_tuple.metadata["test_config_2"] == "bar"

    # invoke graph, but interrupt before tool call
    await app_w_interrupt.ainvoke(
        {"messages": ["what is weather in sf"]},
        {
            "configurable": {
                "thread_id": "2",
                "test_config_3": "foo",
                "test_config_4": "bar",
            },
        },
    )

    config = {"configurable": {"thread_id": "2"}}

    # assert that checkpoint metadata contains the run's configurable fields
    chkpnt_metadata_2 = (await async_checkpointer.aget_tuple(config)).metadata
    assert chkpnt_metadata_2["test_config_3"] == "foo"
    assert chkpnt_metadata_2["test_config_4"] == "bar"

    # resume graph execution
    await app_w_interrupt.ainvoke(
        input=None,
        config={
            "configurable": {
                "thread_id": "2",
                "test_config_3": "foo",
                "test_config_4": "bar",
            }
        },
    )

    # assert that checkpoint metadata contains the run's configurable fields
    chkpnt_metadata_3 = (await async_checkpointer.aget_tuple(config)).metadata
    assert chkpnt_metadata_3["test_config_3"] == "foo"
    assert chkpnt_metadata_3["test_config_4"] == "bar"

    # Verify that all checkpoint metadata have the expected keys. This check
    # is needed because a run may have an arbitrary number of steps depending
    # on how the graph is constructed.
    chkpnt_tuples_2 = async_checkpointer.alist(config)
    async for chkpnt_tuple in chkpnt_tuples_2:
        assert chkpnt_tuple.metadata["test_config_3"] == "foo"
        assert chkpnt_tuple.metadata["test_config_4"] == "bar"


async def test_checkpointer_null_pending_writes() -> None:
    class Node:
        def __init__(self, name: str):
            self.name = name
            setattr(self, "__name__", name)

        def __call__(self, state):
            return [self.name]

    builder = StateGraph(Annotated[list, operator.add])
    builder.add_node(Node("1"))
    builder.add_edge(START, "1")
    graph = builder.compile(checkpointer=MemorySaverNoPending())
    assert graph.invoke([], {"configurable": {"thread_id": "foo"}}) == ["1"]
    assert graph.invoke([], {"configurable": {"thread_id": "foo"}}) == ["1"] * 2
    assert (await graph.ainvoke([], {"configurable": {"thread_id": "foo"}})) == [
        "1"
    ] * 3
    assert (await graph.ainvoke([], {"configurable": {"thread_id": "foo"}})) == [
        "1"
    ] * 4


async def test_store_injected_async(
    async_checkpointer: BaseCheckpointSaver, async_store: BaseStore
) -> None:
    class State(TypedDict):
        count: Annotated[int, operator.add]

    doc_id = str(uuid.uuid4())
    doc = {"some-key": "this-is-a-val"}
    uid = uuid.uuid4().hex
    namespace = (f"foo-{uid}", "bar")
    thread_1 = str(uuid.uuid4())
    thread_2 = str(uuid.uuid4())

    class Node:
        def __init__(self, i: int | None = None):
            self.i = i

        async def __call__(
            self, inputs: State, config: RunnableConfig, store: BaseStore
        ):
            assert isinstance(store, BaseStore)
            await store.aput(
                (
                    namespace
                    if self.i is not None
                    and config["configurable"]["thread_id"] in (thread_1, thread_2)
                    else (f"foo_{self.i}", "bar")
                ),
                doc_id,
                {
                    **doc,
                    "from_thread": config["configurable"]["thread_id"],
                    "some_val": inputs["count"],
                },
            )
            return {"count": 1}

    def other_node(inputs: State, config: RunnableConfig, store: BaseStore):
        assert isinstance(store, BaseStore)
        store.put(("not", "interesting"), "key", {"val": "val"})
        item = store.get(("not", "interesting"), "key")
        assert item is not None
        assert item.value == {"val": "val"}
        return {"count": 0}

    builder = StateGraph(State)
    builder.add_node("node", Node())
    builder.add_node("other_node", other_node)
    builder.add_edge("__start__", "node")
    builder.add_edge("node", "other_node")

    N = 50
    M = 1

    for i in range(N):
        builder.add_node(f"node_{i}", Node(i))
        builder.add_edge("__start__", f"node_{i}")

    graph = builder.compile(store=async_store, checkpointer=async_checkpointer)

    # Test batch operations with multiple threads
    results = await graph.abatch(
        [{"count": 0}] * M,
        ([{"configurable": {"thread_id": str(uuid.uuid4())}}] * (M - 1))
        + [{"configurable": {"thread_id": thread_1}}],
    )
    result = results[-1]
    assert result == {"count": N + 1}
    returned_doc = (await async_store.aget(namespace, doc_id)).value
    assert returned_doc == {**doc, "from_thread": thread_1, "some_val": 0}
    assert len(await async_store.asearch(namespace)) == 1

    # Check results after another turn of the same thread
    result = await graph.ainvoke(
        {"count": 0}, {"configurable": {"thread_id": thread_1}}
    )
    assert result == {"count": (N + 1) * 2}
    returned_doc = (await async_store.aget(namespace, doc_id)).value
    assert returned_doc == {**doc, "from_thread": thread_1, "some_val": N + 1}
    assert len(await async_store.asearch(namespace)) == 1

    # Test with a different thread
    result = await graph.ainvoke(
        {"count": 0}, {"configurable": {"thread_id": thread_2}}
    )
    assert result == {"count": N + 1}
    returned_doc = (await async_store.aget(namespace, doc_id)).value
    assert returned_doc == {
        **doc,
        "from_thread": thread_2,
        "some_val": 0,
    }  # Overwrites the whole doc
    assert (
        len(await async_store.asearch(namespace)) == 1
    )  # still overwriting the same one


async def test_debug_retry(async_checkpointer: BaseCheckpointSaver):
    class State(TypedDict):
        messages: Annotated[list[str], operator.add]

    def node(name):
        async def _node(state: State):
            return {"messages": [f"entered {name} node"]}

        return _node

    builder = StateGraph(State)
    builder.add_node("one", node("one"))
    builder.add_node("two", node("two"))
    builder.add_edge(START, "one")
    builder.add_edge("one", "two")
    builder.add_edge("two", END)

    graph = builder.compile(checkpointer=async_checkpointer)

    config = {"configurable": {"thread_id": "1"}}
    await graph.ainvoke({"messages": []}, config=config, durability="async")

    # re-run step: 1
    async for c in async_checkpointer.alist(config):
        if c.metadata["step"] == 1:
            target_config = c.parent_config
            break
    assert target_config is not None

    update_config = await graph.aupdate_state(target_config, values=None)

    events = [
        c
        async for c in graph.astream(
            None, config=update_config, stream_mode="debug", durability="async"
        )
    ]

    checkpoint_events = list(
        reversed([e["payload"] for e in events if e["type"] == "checkpoint"])
    )

    checkpoint_history = {
        c.config["configurable"]["checkpoint_id"]: c
        async for c in graph.aget_state_history(config)
    }

    def lax_normalize_config(config: dict | None) -> dict | None:
        if config is None:
            return None
        return config["configurable"]

    for stream in checkpoint_events:
        stream_conf = lax_normalize_config(stream["config"])
        stream_parent_conf = lax_normalize_config(stream["parent_config"])
        assert stream_conf != stream_parent_conf

        # ensure the streamed checkpoint == checkpoint from checkpointer.list()
        history = checkpoint_history[stream["config"]["configurable"]["checkpoint_id"]]
        history_conf = lax_normalize_config(history.config)
        assert stream_conf == history_conf

        history_parent_conf = lax_normalize_config(history.parent_config)
        assert stream_parent_conf == history_parent_conf


async def test_debug_subgraphs(
    async_checkpointer: BaseCheckpointSaver, durability: Durability
):
    class State(TypedDict):
        messages: Annotated[list[str], operator.add]

    def node(name):
        async def _node(state: State):
            return {"messages": [f"entered {name} node"]}

        return _node

    parent = StateGraph(State)
    child = StateGraph(State)

    child.add_node("c_one", node("c_one"))
    child.add_node("c_two", node("c_two"))
    child.add_edge(START, "c_one")
    child.add_edge("c_one", "c_two")
    child.add_edge("c_two", END)

    parent.add_node("p_one", node("p_one"))
    parent.add_node("p_two", child.compile())
    parent.add_edge(START, "p_one")
    parent.add_edge("p_one", "p_two")
    parent.add_edge("p_two", END)

    graph = parent.compile(checkpointer=async_checkpointer)

    config = {"configurable": {"thread_id": "1"}}
    events = [
        c
        async for c in graph.astream(
            {"messages": []},
            config=config,
            stream_mode="debug",
            durability=durability,
        )
    ]

    checkpoint_events = list(
        reversed([e["payload"] for e in events if e["type"] == "checkpoint"])
    )
    if durability == "exit":
        checkpoint_events = checkpoint_events[:1]
    checkpoint_history = [c async for c in graph.aget_state_history(config)]

    assert len(checkpoint_events) == len(checkpoint_history)

    def normalize_config(config: dict | None) -> dict | None:
        if config is None:
            return None
        return config["configurable"]

    for stream, history in zip(checkpoint_events, checkpoint_history):
        assert stream["values"] == history.values
        assert stream["next"] == list(history.next)
        assert normalize_config(stream["config"]) == normalize_config(history.config)
        assert normalize_config(stream["parent_config"]) == normalize_config(
            history.parent_config
        )

        assert len(stream["tasks"]) == len(history.tasks)
        for stream_task, history_task in zip(stream["tasks"], history.tasks):
            assert stream_task["id"] == history_task.id
            assert stream_task["name"] == history_task.name
            assert stream_task["interrupts"] == history_task.interrupts
            assert stream_task.get("error") == history_task.error
            assert stream_task.get("state") == history_task.state


async def test_debug_nested_subgraphs(
    async_checkpointer: BaseCheckpointSaver, durability: Durability
) -> None:
    from collections import defaultdict

    class State(TypedDict):
        messages: Annotated[list[str], operator.add]

    def node(name):
        async def _node(state: State):
            return {"messages": [f"entered {name} node"]}

        return _node

    grand_parent = StateGraph(State)
    parent = StateGraph(State)
    child = StateGraph(State)

    child.add_node("c_one", node("c_one"))
    child.add_node("c_two", node("c_two"))
    child.add_edge(START, "c_one")
    child.add_edge("c_one", "c_two")
    child.add_edge("c_two", END)

    parent.add_node("p_one", node("p_one"))
    parent.add_node("p_two", child.compile())
    parent.add_edge(START, "p_one")
    parent.add_edge("p_one", "p_two")
    parent.add_edge("p_two", END)

    grand_parent.add_node("gp_one", node("gp_one"))
    grand_parent.add_node("gp_two", parent.compile())
    grand_parent.add_edge(START, "gp_one")
    grand_parent.add_edge("gp_one", "gp_two")
    grand_parent.add_edge("gp_two", END)

    graph = grand_parent.compile(checkpointer=async_checkpointer)

    config = {"configurable": {"thread_id": "1"}}
    events = [
        c
        async for c in graph.astream(
            {"messages": []},
            config=config,
            stream_mode="debug",
            subgraphs=True,
            durability=durability,
        )
    ]

    stream_ns: dict[tuple, dict] = defaultdict(list)
    for ns, e in events:
        if e["type"] == "checkpoint":
            stream_ns[ns].append(e["payload"])

    assert list(stream_ns.keys()) == [
        (),
        (AnyStr("gp_two:"),),
        (AnyStr("gp_two:"), AnyStr("p_two:")),
    ]

    history_ns = {}
    for ns in stream_ns.keys():

        async def get_history():
            history = [
                c
                async for c in graph.aget_state_history(
                    {"configurable": {"thread_id": "1", "checkpoint_ns": "|".join(ns)}}
                )
            ]
            return history[::-1]

        history_ns[ns] = await get_history()

    def normalize_config(config: dict | None) -> dict | None:
        if config is None:
            return None

        clean_config = {}
        clean_config["thread_id"] = config["configurable"]["thread_id"]
        clean_config["checkpoint_id"] = config["configurable"]["checkpoint_id"]
        clean_config["checkpoint_ns"] = config["configurable"]["checkpoint_ns"]
        if "checkpoint_map" in config["configurable"]:
            clean_config["checkpoint_map"] = config["configurable"]["checkpoint_map"]

        return clean_config

    for checkpoint_events, checkpoint_history, ns in zip(
        stream_ns.values(), history_ns.values(), stream_ns.keys()
    ):
        if durability == "exit":
            checkpoint_events = checkpoint_events[-1:]
            if ns:  # Save no checkpoints for subgraphs when durability="exit"
                assert not checkpoint_history
                continue
        assert len(checkpoint_events) == len(checkpoint_history)
        for stream, history in zip(checkpoint_events, checkpoint_history):
            assert stream["values"] == history.values
            assert stream["next"] == list(history.next)
            assert normalize_config(stream["config"]) == normalize_config(
                history.config
            )
            assert normalize_config(stream["parent_config"]) == normalize_config(
                history.parent_config
            )

            assert len(stream["tasks"]) == len(history.tasks)
            for stream_task, history_task in zip(stream["tasks"], history.tasks):
                assert stream_task["id"] == history_task.id
                assert stream_task["name"] == history_task.name
                assert stream_task["interrupts"] == history_task.interrupts
                assert stream_task.get("error") == history_task.error
                assert stream_task.get("state") == history_task.state


@pytest.mark.parametrize("subgraph_persist", [True, False])
async def test_parent_command(
    async_checkpointer: BaseCheckpointSaver, subgraph_persist: bool
) -> None:
    from langchain_core.messages import BaseMessage
    from langchain_core.tools import tool

    @tool(return_direct=True)
    def get_user_name() -> Command:
        """Retrieve user name"""
        return Command(update={"user_name": "Meow"}, graph=Command.PARENT)

    subgraph_builder = StateGraph(MessagesState)
    subgraph_builder.add_node("tool", get_user_name)
    subgraph_builder.add_edge(START, "tool")
    subgraph = subgraph_builder.compile(checkpointer=subgraph_persist)

    class CustomParentState(TypedDict):
        messages: Annotated[list[BaseMessage], add_messages]
        # this key is not available to the child graph
        user_name: str

    builder = StateGraph(CustomParentState)
    builder.add_node("alice", subgraph)
    builder.add_edge(START, "alice")

    graph = builder.compile(checkpointer=async_checkpointer)

    config = {"configurable": {"thread_id": "1"}}

    assert await graph.ainvoke(
        {"messages": [("user", "get user name")]}, config, durability="exit"
    ) == {
        "messages": [
            _AnyIdHumanMessage(
                content="get user name", additional_kwargs={}, response_metadata={}
            ),
        ],
        "user_name": "Meow",
    }
    assert await graph.aget_state(config) == StateSnapshot(
        values={
            "messages": [
                _AnyIdHumanMessage(
                    content="get user name",
                    additional_kwargs={},
                    response_metadata={},
                ),
            ],
            "user_name": "Meow",
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
            "step": 1,
            "parents": {},
        },
        created_at=AnyStr(),
        parent_config=None,
        tasks=(),
        interrupts=(),
    )


@NEEDS_CONTEXTVARS
async def test_interrupt_subgraph(async_checkpointer: BaseCheckpointSaver) -> None:
    class State(TypedDict):
        baz: str

    def foo(state):
        return {"baz": "foo"}

    def bar(state):
        value = interrupt("Please provide baz value:")
        return {"baz": value}

    child_builder = StateGraph(State)
    child_builder.add_node(bar)
    child_builder.add_edge(START, "bar")

    builder = StateGraph(State)
    builder.add_node(foo)
    builder.add_node("bar", child_builder.compile())
    builder.add_edge(START, "foo")
    builder.add_edge("foo", "bar")

    graph = builder.compile(checkpointer=async_checkpointer)

    thread1 = {"configurable": {"thread_id": "1"}}
    # First run, interrupted at bar
    assert await graph.ainvoke({"baz": ""}, thread1)
    # Resume with answer
    assert await graph.ainvoke(Command(resume="bar"), thread1)


@NEEDS_CONTEXTVARS
async def test_interrupt_multiple(async_checkpointer: BaseCheckpointSaver):
    class State(TypedDict):
        my_key: Annotated[str, operator.add]

    async def node(s: State) -> State:
        answer = interrupt({"value": 1})
        answer2 = interrupt({"value": 2})
        return {"my_key": answer + " " + answer2}

    builder = StateGraph(State)
    builder.add_node("node", node)
    builder.add_edge(START, "node")

    graph = builder.compile(checkpointer=async_checkpointer)
    thread1 = {"configurable": {"thread_id": "1"}}

    assert [
        e async for e in graph.astream({"my_key": "DE", "market": "DE"}, thread1)
    ] == [
        {
            "__interrupt__": (
                Interrupt(
                    value={"value": 1},
                    id=AnyStr(),
                ),
            )
        }
    ]

    assert [
        event
        async for event in graph.astream(
            Command(resume="answer 1", update={"my_key": "foofoo"}),
            thread1,
            stream_mode="updates",
        )
    ] == [
        {
            "__interrupt__": (
                Interrupt(
                    value={"value": 2},
                    id=AnyStr(),
                ),
            )
        }
    ]

    assert [
        event
        async for event in graph.astream(
            Command(resume="answer 2"), thread1, stream_mode="updates"
        )
    ] == [
        {"node": {"my_key": "answer 1 answer 2"}},
    ]


@NEEDS_CONTEXTVARS
async def test_interrupt_loop(async_checkpointer: BaseCheckpointSaver) -> None:
    class State(TypedDict):
        age: int
        other: str

    async def ask_age(s: State):
        """Ask an expert for help."""
        question = "How old are you?"
        value = None
        for _ in range(10):
            value: str = interrupt(question)
            if not value.isdigit() or int(value) < 18:
                question = "invalid response"
                value = None
            else:
                break

        return {"age": int(value)}

    builder = StateGraph(State)
    builder.add_node("node", ask_age)
    builder.add_edge(START, "node")

    graph = builder.compile(checkpointer=async_checkpointer)
    thread1 = {"configurable": {"thread_id": "1"}}

    assert [e async for e in graph.astream({"other": ""}, thread1)] == [
        {
            "__interrupt__": (
                Interrupt(
                    value="How old are you?",
                    id=AnyStr(),
                ),
            )
        }
    ]

    assert [
        event
        async for event in graph.astream(
            Command(resume="13"),
            thread1,
        )
    ] == [
        {
            "__interrupt__": (
                Interrupt(
                    value="invalid response",
                    id=AnyStr(),
                ),
            )
        }
    ]

    assert [
        event
        async for event in graph.astream(
            Command(resume="15"),
            thread1,
        )
    ] == [
        {
            "__interrupt__": (
                Interrupt(
                    value="invalid response",
                    id=AnyStr(),
                ),
            )
        }
    ]

    assert [event async for event in graph.astream(Command(resume="19"), thread1)] == [
        {"node": {"age": 19}},
    ]


@NEEDS_CONTEXTVARS
async def test_interrupt_functional(async_checkpointer: BaseCheckpointSaver) -> None:
    @task
    async def foo(state: dict) -> dict:
        return {"a": state["a"] + "foo"}

    @task
    async def bar(state: dict) -> dict:
        return {"a": state["a"] + "bar", "b": state["b"]}

    @entrypoint(checkpointer=async_checkpointer)
    async def graph(inputs: dict) -> dict:
        foo_result = await foo(inputs)
        value = interrupt("Provide value for bar:")
        bar_input = {**foo_result, "b": value}
        bar_result = await bar(bar_input)
        return bar_result

    config = {"configurable": {"thread_id": "1"}}
    # First run, interrupted at bar
    await graph.ainvoke({"a": ""}, config)
    # Resume with an answer
    res = await graph.ainvoke(Command(resume="bar"), config)
    assert res == {"a": "foobar", "b": "bar"}


@NEEDS_CONTEXTVARS
async def test_interrupt_task_functional(
    async_checkpointer: BaseCheckpointSaver,
) -> None:
    @task
    async def foo(state: dict) -> dict:
        return {"a": state["a"] + "foo"}

    @task
    async def bar(state: dict) -> dict:
        value = interrupt("Provide value for bar:")
        return {"a": state["a"] + value}

    @entrypoint(checkpointer=async_checkpointer)
    async def graph(inputs: dict) -> dict:
        foo_result = await foo(inputs)
        bar_result = await bar(foo_result)
        return bar_result

    config = {"configurable": {"thread_id": "1"}}
    # First run, interrupted at bar
    assert await graph.ainvoke({"a": ""}, config) == {
        "__interrupt__": [
            Interrupt(
                value="Provide value for bar:",
                id=AnyStr(),
            ),
        ]
    }
    # Resume with an answer
    res = await graph.ainv
```
> [truncated]

---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/test_pydantic.py
```py
import datetime
import decimal
import ipaddress
import pathlib
import re
import sys
import uuid
from enum import Enum
from typing import Annotated, Literal, Optional

from pydantic import (
    BaseModel,
    ByteSize,
    Field,
    SecretStr,
    confloat,
    conint,
    conlist,
    constr,
    field_validator,
    model_validator,
)

from langgraph._internal._pydantic import is_supported_by_pydantic
from langgraph.constants import END, START
from langgraph.graph.state import StateGraph


def test_is_supported_by_pydantic() -> None:
    """Test if types are supported by pydantic."""
    import typing

    import pydantic
    import typing_extensions

    class TypedDictExtensions(typing_extensions.TypedDict):
        x: int

    assert is_supported_by_pydantic(TypedDictExtensions) is True

    class VanillaClass:
        x: int

    assert is_supported_by_pydantic(VanillaClass) is False

    class BuiltinTypedDict(typing.TypedDict):  # noqa: TID251
        x: int

    if sys.version_info >= (3, 12):
        assert is_supported_by_pydantic(BuiltinTypedDict) is True
    else:
        assert is_supported_by_pydantic(BuiltinTypedDict) is False

    class PydanticModel(pydantic.BaseModel):
        x: int

    assert is_supported_by_pydantic(PydanticModel) is True


def test_nested_pydantic_models() -> None:
    """Test that nested Pydantic models are properly constructed from leaf nodes up."""

    class NestedModel(BaseModel):
        value: int
        name: str

    # For constrained types
    PositiveInt = Annotated[int, Field(gt=0)]
    NonNegativeFloat = Annotated[float, Field(ge=0)]

    # Enum type
    class UserRole(Enum):
        ADMIN = "admin"
        USER = "user"
        GUEST = "guest"

    # Forward reference model
    class RecursiveModel(BaseModel):
        value: str
        child: Optional["RecursiveModel"] = None

    # Discriminated union models
    class Cat(BaseModel):
        pet_type: Literal["cat"]
        meow: str

    class Dog(BaseModel):
        pet_type: Literal["dog"]
        bark: str

    # Cyclic reference model
    class Person(BaseModel):
        id: str
        name: str
        friends: list[str] = Field(default_factory=list)  # IDs of friends

    conlist_type = conlist(item_type=int, min_length=2, max_length=5)

    class State(BaseModel):
        # Basic nested model tests
        top_level: str
        auuid: uuid.UUID
        nested: NestedModel
        optional_nested: Annotated[NestedModel | None, lambda x, y: y, "Foo"]
        dict_nested: dict[str, NestedModel]
        simple_str_list: list[str]
        list_nested: Annotated[
            dict | list[dict[str, NestedModel]], lambda x, y: (x or []) + [y]
        ]
        tuple_nested: tuple[str, NestedModel]
        tuple_list_nested: list[tuple[int, NestedModel]]
        complex_tuple: tuple[str, dict[str, tuple[int, NestedModel]]]

        # Forward reference test
        recursive: RecursiveModel

        # Discriminated union test
        pet: Cat | Dog

        # Cyclic reference test
        people: dict[str, Person]  # Map of ID -> Person

        # Rich type adapters
        ip_address: ipaddress.IPv4Address
        ip_address_v6: ipaddress.IPv6Address
        amount: decimal.Decimal
        file_path: pathlib.Path
        timestamp: datetime.datetime
        date_only: datetime.date
        time_only: datetime.time
        duration: datetime.timedelta
        immutable_set: frozenset[int]
        binary_data: bytes
        pattern: re.Pattern
        secret: SecretStr
        file_size: ByteSize

        # Constrained types
        positive_value: PositiveInt
        non_negative: NonNegativeFloat
        limited_string: constr(min_length=3, max_length=10)
        bounded_int: conint(ge=10, le=100)
        restricted_float: confloat(gt=0, lt=1)
        required_list: conlist_type

        # Enum & Literal
        role: UserRole
        status: Literal["active", "inactive", "pending"]

        # Annotated & NewType
        validated_age: Annotated[int, Field(gt=0, lt=120)]

        # Generic containers with validators
        decimal_list: list[decimal.Decimal]
        id_tuple: tuple[uuid.UUID, uuid.UUID]

    inputs = {
        # Basic nested models
        "top_level": "initial",
        "auuid": str(uuid.uuid4()),
        "nested": {"value": 42, "name": "test"},
        "optional_nested": {"value": 10, "name": "optional"},
        "dict_nested": {"a": {"value": 5, "name": "a"}},
        "list_nested": [{"a": {"value": 6, "name": "b"}}],
        "tuple_nested": ["tuple-key", {"value": 7, "name": "tuple-value"}],
        "tuple_list_nested": [[1, {"value": 8, "name": "tuple-in-list"}]],
        "simple_str_list": ["siss", "boom", "bah"],
        "complex_tuple": [
            "complex",
            {"nested": [9, {"value": 10, "name": "deep"}]},
        ],
        # Forward reference
        "recursive": {"value": "parent", "child": {"value": "child", "child": None}},
        # Discriminated union (using a cat in this case)
        "pet": {"pet_type": "cat", "meow": "meow!"},
        # Cyclic references
        "people": {
            "1": {
                "id": "1",
                "name": "Alice",
                "friends": ["2", "3"],  # Alice is friends with Bob and Charlie
            },
            "2": {
                "id": "2",
                "name": "Bob",
                "friends": ["1"],  # Bob is friends with Alice
            },
            "3": {
                "id": "3",
                "name": "Charlie",
                "friends": ["1", "2"],  # Charlie is friends with Alice and Bob
            },
        },
        # Rich type adapters
        "ip_address": "192.168.1.1",
        "ip_address_v6": "2001:db8::1",
        "amount": "123.45",
        "file_path": "/tmp/test.txt",
        "timestamp": "2025-04-07T10:58:04",
        "date_only": "2025-04-07",
        "time_only": "10:58:04",
        "duration": 3600,  # seconds
        "immutable_set": [1, 2, 3, 4],
        "binary_data": b"hello world",
        "pattern": "^test$",
        "secret": "password123",
        "file_size": 1024,
        # Constrained types
        "positive_value": 42,
        "non_negative": 0.0,
        "limited_string": "test",
        "bounded_int": 50,
        "restricted_float": 0.5,
        "required_list": [10, 20, 30],
        # Enum & Literal
        "role": "admin",
        "status": "active",
        # Annotated & NewType
        "validated_age": 30,
        # Generic containers with validators
        "decimal_list": ["10.5", "20.75", "30.25"],
        "id_tuple": [str(uuid.uuid4()), str(uuid.uuid4())],
    }

    update = {"top_level": "updated", "nested": {"value": 100, "name": "updated"}}

    expected = State(**inputs)

    def node_fn(state: State) -> dict:
        # Basic assertions
        assert isinstance(state.auuid, uuid.UUID)
        assert state == expected

        # Rich type assertions
        assert isinstance(state.ip_address, ipaddress.IPv4Address)
        assert isinstance(state.ip_address_v6, ipaddress.IPv6Address)
        assert isinstance(state.amount, decimal.Decimal)
        assert isinstance(state.file_path, pathlib.Path)
        assert isinstance(state.timestamp, datetime.datetime)
        assert isinstance(state.date_only, datetime.date)
        assert isinstance(state.time_only, datetime.time)
        assert isinstance(state.duration, datetime.timedelta)
        assert isinstance(state.immutable_set, frozenset)
        assert isinstance(state.binary_data, bytes)
        assert isinstance(state.pattern, re.Pattern)

        # Constrained types
        assert state.positive_value > 0
        assert state.non_negative >= 0
        assert 3 <= len(state.limited_string) <= 10
        assert 10 <= state.bounded_int <= 100
        assert 0 < state.restricted_float < 1
        assert 2 <= len(state.required_list) <= 5

        # Enum & Literal
        assert state.role == UserRole.ADMIN
        assert state.status == "active"

        # Annotated
        assert 0 < state.validated_age < 120

        # Generic containers
        assert len(state.decimal_list) == 3
        assert len(state.id_tuple) == 2

        return update

    builder = StateGraph(State)
    builder.add_node("process", node_fn)
    builder.set_entry_point("process")
    builder.set_finish_point("process")
    graph = builder.compile()

    result = graph.invoke(inputs.copy())

    assert result == {**inputs, **update}

    new_inputs = inputs.copy()
    new_inputs["list_nested"] = {"foo": "bar"}
    expected = State(**new_inputs)
    assert {**new_inputs, **update} == graph.invoke(new_inputs.copy())


def test_pydantic_state_field_validator():
    class State(BaseModel):
        name: str
        text: str = ""
        only_root: int = 13

        @field_validator("name", mode="after")
        @classmethod
        def validate_name(cls, value):
            if value[0].islower():
                raise ValueError("Name must start with a capital letter")
            return "Validated " + value

        @model_validator(mode="before")
        @classmethod
        def validate_amodel(cls, values: "State"):
            return values | {"only_root": 392}

    input_state = {"name": "John"}

    def process_node(state: State):
        assert State.model_validate(input_state) == state
        return {"text": "Hello, " + state.name + "!"}

    builder = StateGraph(state_schema=State)
    builder.add_node("process", process_node)
    builder.add_edge(START, "process")
    builder.add_edge("process", END)
    g = builder.compile()
    res = g.invoke(input_state)
    assert res["text"] == "Hello, Validated John!"

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/test_remote_graph.py
```py
import re
import sys
from typing import Annotated
from unittest.mock import AsyncMock, MagicMock

import langsmith as ls
import pytest
from langchain_core.messages import AnyMessage, BaseMessage
from langchain_core.runnables import RunnableConfig
from langchain_core.runnables.graph import Edge as DrawableEdge
from langchain_core.runnables.graph import Node as DrawableNode
from langgraph_sdk.schema import StreamPart
from typing_extensions import TypedDict

from langgraph.errors import GraphInterrupt
from langgraph.graph import StateGraph, add_messages
from langgraph.pregel import Pregel
from langgraph.pregel.remote import RemoteGraph
from langgraph.types import Interrupt, StateSnapshot
from tests.any_str import AnyStr
from tests.conftest import NO_DOCKER
from tests.example_app.example_graph import app

if NO_DOCKER:
    pytest.skip(
        "Skipping tests that require Docker. Unset NO_DOCKER to run them.",
        allow_module_level=True,
    )

pytestmark = pytest.mark.anyio

NEEDS_CONTEXTVARS = pytest.mark.skipif(
    sys.version_info < (3, 11),
    reason="Python 3.11+ is required for async contextvars support",
)

SKIP_PYTHON_314 = pytest.mark.skipif(
    sys.version_info >= (3, 14),
    reason="Not yet testing Python 3.14 with the server bc of dependency limits on the api side",
)


def test_with_config():
    # set up test
    remote_pregel = RemoteGraph(
        "test_graph_id",
        config={
            "configurable": {
                "foo": "bar",
                "thread_id": "thread_id_1",
            }
        },
    )

    # call method / assertions
    config = {"configurable": {"hello": "world"}}
    remote_pregel_copy = remote_pregel.with_config(config)

    # assert that a copy was returned
    assert remote_pregel_copy != remote_pregel
    # assert that configs were merged
    assert remote_pregel_copy.config == {
        "configurable": {
            "foo": "bar",
            "thread_id": "thread_id_1",
            "hello": "world",
        }
    }


def test_get_graph():
    # set up test
    mock_sync_client = MagicMock()
    mock_sync_client.assistants.get_graph.return_value = {
        "nodes": [
            {"id": "__start__", "type": "schema", "data": "__start__"},
            {"id": "__end__", "type": "schema", "data": "__end__"},
            {
                "id": "agent",
                "type": "runnable",
                "data": {
                    "id": ["langgraph", "utils", "RunnableCallable"],
                    "name": "agent_1",
                },
            },
        ],
        "edges": [
            {"source": "__start__", "target": "agent"},
            {"source": "agent", "target": "__end__"},
        ],
    }

    remote_pregel = RemoteGraph("test_graph_id", sync_client=mock_sync_client)

    # call method / assertions
    drawable_graph = remote_pregel.get_graph()

    assert drawable_graph.nodes == {
        "__start__": DrawableNode(
            id="__start__", name="__start__", data="__start__", metadata=None
        ),
        "__end__": DrawableNode(
            id="__end__", name="__end__", data="__end__", metadata=None
        ),
        "agent": DrawableNode(
            id="agent",
            name="agent_1",
            data={"id": ["langgraph", "utils", "RunnableCallable"], "name": "agent_1"},
            metadata=None,
        ),
    }

    assert drawable_graph.edges == [
        DrawableEdge(source="__start__", target="agent"),
        DrawableEdge(source="agent", target="__end__"),
    ]


@pytest.mark.anyio
async def test_aget_graph():
    # set up test
    mock_async_client = AsyncMock()
    mock_async_client.assistants.get_graph.return_value = {
        "nodes": [
            {"id": "__start__", "type": "schema", "data": "__start__"},
            {"id": "__end__", "type": "schema", "data": "__end__"},
            {
                "id": "agent",
                "type": "runnable",
                "data": {
                    "id": ["langgraph", "utils", "RunnableCallable"],
                    "name": "agent_1",
                },
            },
        ],
        "edges": [
            {"source": "__start__", "target": "agent"},
            {"source": "agent", "target": "__end__"},
        ],
    }

    remote_pregel = RemoteGraph("test_graph_id", client=mock_async_client)

    # call method / assertions
    drawable_graph = await remote_pregel.aget_graph()

    assert drawable_graph.nodes == {
        "__start__": DrawableNode(
            id="__start__", name="__start__", data="__start__", metadata=None
        ),
        "__end__": DrawableNode(
            id="__end__", name="__end__", data="__end__", metadata=None
        ),
        "agent": DrawableNode(
            id="agent",
            name="agent_1",
            data={"id": ["langgraph", "utils", "RunnableCallable"], "name": "agent_1"},
            metadata=None,
        ),
    }

    assert drawable_graph.edges == [
        DrawableEdge(source="__start__", target="agent"),
        DrawableEdge(source="agent", target="__end__"),
    ]


def test_get_state():
    # set up test
    mock_sync_client = MagicMock()
    mock_sync_client.threads.get_state.return_value = {
        "values": {"messages": [{"type": "human", "content": "hello"}]},
        "next": None,
        "checkpoint": {
            "thread_id": "thread_1",
            "checkpoint_ns": "ns",
            "checkpoint_id": "checkpoint_1",
            "checkpoint_map": {},
        },
        "metadata": {},
        "created_at": "timestamp",
        "parent_checkpoint": None,
        "tasks": [],
    }

    # call method / assertions
    remote_pregel = RemoteGraph(
        "test_graph_id",
        sync_client=mock_sync_client,
    )

    config = {"configurable": {"thread_id": "thread1"}}
    state_snapshot = remote_pregel.get_state(config)

    assert state_snapshot == StateSnapshot(
        values={"messages": [{"type": "human", "content": "hello"}]},
        next=(),
        config={
            "configurable": {
                "thread_id": "thread_1",
                "checkpoint_ns": "ns",
                "checkpoint_id": "checkpoint_1",
                "checkpoint_map": {},
            }
        },
        metadata={},
        created_at="timestamp",
        parent_config=None,
        tasks=(),
        interrupts=(),
    )


@pytest.mark.anyio
async def test_aget_state():
    mock_async_client = AsyncMock()
    mock_async_client.threads.get_state.return_value = {
        "values": {"messages": [{"type": "human", "content": "hello"}]},
        "next": None,
        "checkpoint": {
            "thread_id": "thread_1",
            "checkpoint_ns": "ns",
            "checkpoint_id": "checkpoint_2",
            "checkpoint_map": {},
        },
        "metadata": {},
        "created_at": "timestamp",
        "parent_checkpoint": {
            "thread_id": "thread_1",
            "checkpoint_ns": "ns",
            "checkpoint_id": "checkpoint_1",
            "checkpoint_map": {},
        },
        "tasks": [],
    }

    # call method / assertions
    remote_pregel = RemoteGraph(
        "test_graph_id",
        client=mock_async_client,
    )

    config = {"configurable": {"thread_id": "thread1"}}
    state_snapshot = await remote_pregel.aget_state(config)

    assert state_snapshot == StateSnapshot(
        values={"messages": [{"type": "human", "content": "hello"}]},
        next=(),
        config={
            "configurable": {
                "thread_id": "thread_1",
                "checkpoint_ns": "ns",
                "checkpoint_id": "checkpoint_2",
                "checkpoint_map": {},
            }
        },
        metadata={},
        created_at="timestamp",
        parent_config={
            "configurable": {
                "thread_id": "thread_1",
                "checkpoint_ns": "ns",
                "checkpoint_id": "checkpoint_1",
                "checkpoint_map": {},
            }
        },
        tasks=(),
        interrupts=(),
    )


def test_get_state_history():
    # set up test
    mock_sync_client = MagicMock()
    mock_sync_client.threads.get_history.return_value = [
        {
            "values": {"messages": [{"type": "human", "content": "hello"}]},
            "next": None,
            "checkpoint": {
                "thread_id": "thread_1",
                "checkpoint_ns": "ns",
                "checkpoint_id": "checkpoint_1",
                "checkpoint_map": {},
            },
            "metadata": {},
            "created_at": "timestamp",
            "parent_checkpoint": None,
            "tasks": [],
        }
    ]

    # call method / assertions
    remote_pregel = RemoteGraph(
        "test_graph_id",
        sync_client=mock_sync_client,
    )

    config = {"configurable": {"thread_id": "thread1"}}
    state_history_snapshot = list(
        remote_pregel.get_state_history(config, filter=None, before=None, limit=None)
    )

    assert len(state_history_snapshot) == 1
    assert state_history_snapshot[0] == StateSnapshot(
        values={"messages": [{"type": "human", "content": "hello"}]},
        next=(),
        config={
            "configurable": {
                "thread_id": "thread_1",
                "checkpoint_ns": "ns",
                "checkpoint_id": "checkpoint_1",
                "checkpoint_map": {},
            }
        },
        metadata={},
        created_at="timestamp",
        parent_config=None,
        tasks=(),
        interrupts=(),
    )


@pytest.mark.anyio
async def test_aget_state_history():
    # set up test
    mock_async_client = AsyncMock()
    mock_async_client.threads.get_history.return_value = [
        {
            "values": {"messages": [{"type": "human", "content": "hello"}]},
            "next": None,
            "checkpoint": {
                "thread_id": "thread_1",
                "checkpoint_ns": "ns",
                "checkpoint_id": "checkpoint_1",
                "checkpoint_map": {},
            },
            "metadata": {},
            "created_at": "timestamp",
            "parent_checkpoint": None,
            "tasks": [],
        }
    ]

    # call method / assertions
    remote_pregel = RemoteGraph(
        "test_graph_id",
        client=mock_async_client,
    )

    config = {"configurable": {"thread_id": "thread1"}}
    state_history_snapshot = []
    async for state_snapshot in remote_pregel.aget_state_history(
        config, filter=None, before=None, limit=None
    ):
        state_history_snapshot.append(state_snapshot)

    assert len(state_history_snapshot) == 1
    assert state_history_snapshot[0] == StateSnapshot(
        values={"messages": [{"type": "human", "content": "hello"}]},
        next=(),
        config={
            "configurable": {
                "thread_id": "thread_1",
                "checkpoint_ns": "ns",
                "checkpoint_id": "checkpoint_1",
                "checkpoint_map": {},
            }
        },
        metadata={},
        created_at="timestamp",
        parent_config=None,
        tasks=(),
        interrupts=(),
    )


def test_update_state():
    # set up test
    mock_sync_client = MagicMock()
    mock_sync_client.threads.update_state.return_value = {
        "checkpoint": {
            "thread_id": "thread_1",
            "checkpoint_ns": "ns",
            "checkpoint_id": "checkpoint_1",
            "checkpoint_map": {},
        }
    }

    # call method / assertions
    remote_pregel = RemoteGraph(
        "test_graph_id",
        sync_client=mock_sync_client,
    )

    config = {"configurable": {"thread_id": "thread1"}}
    response = remote_pregel.update_state(config, {"key": "value"})

    assert response == {
        "configurable": {
            "thread_id": "thread_1",
            "checkpoint_ns": "ns",
            "checkpoint_id": "checkpoint_1",
            "checkpoint_map": {},
        }
    }


@pytest.mark.anyio
async def test_aupdate_state():
    # set up test
    mock_async_client = AsyncMock()
    mock_async_client.threads.update_state.return_value = {
        "checkpoint": {
            "thread_id": "thread_1",
            "checkpoint_ns": "ns",
            "checkpoint_id": "checkpoint_1",
            "checkpoint_map": {},
        }
    }

    # call method / assertions
    remote_pregel = RemoteGraph(
        "test_graph_id",
        client=mock_async_client,
    )

    config = {"configurable": {"thread_id": "thread1"}}
    response = await remote_pregel.aupdate_state(config, {"key": "value"})

    assert response == {
        "configurable": {
            "thread_id": "thread_1",
            "checkpoint_ns": "ns",
            "checkpoint_id": "checkpoint_1",
            "checkpoint_map": {},
        }
    }


def test_stream():
    # set up test
    mock_sync_client = MagicMock()
    mock_sync_client.runs.stream.return_value = [
        StreamPart(event="values", data={"chunk": "data1"}),
        StreamPart(event="values", data={"chunk": "data2"}),
        StreamPart(event="values", data={"chunk": "data3"}),
        StreamPart(event="updates", data={"chunk": "data4"}),
        StreamPart(
            event="messages",
            data=[
                {
                    "content": [{"text": "Hello", "type": "text", "index": 0}],
                    "type": "AIMessageChunk",
                },
                {
                    "langgraph_step": 1,
                    "langgraph_node": "call_llm",
                    "langgraph_triggers": ["branch:to:call_llm"],
                    "langgraph_path": ["__pregel_pull", "call_llm"],
                },
            ],
        ),
        StreamPart(
            event="updates",
            data={
                "__interrupt__": [
                    {
                        "value": {"question": "Does this look good?"},
                        "id": AnyStr(),
                    }
                ]
            },
        ),
    ]

    # call method / assertions
    remote_pregel = RemoteGraph(
        "test_graph_id",
        sync_client=mock_sync_client,
    )

    # test raising graph interrupt if invoked as a subgraph
    with pytest.raises(GraphInterrupt) as exc:
        for stream_part in remote_pregel.stream(
            {"input": "data"},
            # pretend we invoked this as a subgraph
            config={
                "configurable": {"thread_id": "thread_1", "checkpoint_ns": "some_ns"}
            },
            stream_mode="values",
        ):
            pass

    assert exc.value.args[0] == [
        Interrupt(
            value={"question": "Does this look good?"},
            id=AnyStr(),
        )
    ]

    # stream modes doesn't include 'updates'
    stream_parts = []
    for stream_part in remote_pregel.stream(
        {"input": "data"},
        config={"configurable": {"thread_id": "thread_1"}},
        stream_mode="values",
    ):
        stream_parts.append(stream_part)

    assert stream_parts == [
        {"chunk": "data1"},
        {"chunk": "data2"},
        {"chunk": "data3"},
    ]

    # stream_mode messages
    stream_parts = []
    for stream_part in remote_pregel.stream(
        {"input": "data"},
        config={"configurable": {"thread_id": "thread_1"}},
        stream_mode="messages",
    ):
        stream_parts.append(stream_part)

    assert stream_parts == [
        (
            {
                "content": [{"text": "Hello", "type": "text", "index": 0}],
                "type": "AIMessageChunk",
            },
            {
                "langgraph_step": 1,
                "langgraph_node": "call_llm",
                "langgraph_triggers": ["branch:to:call_llm"],
                "langgraph_path": ["__pregel_pull", "call_llm"],
            },
        ),
    ]

    mock_sync_client.runs.stream.return_value = [
        StreamPart(event="updates", data={"chunk": "data3"}),
        StreamPart(event="updates", data={"chunk": "data4"}),
        StreamPart(event="updates", data={"__interrupt__": ()}),
    ]

    # default stream_mode is updates
    stream_parts = []
    for stream_part in remote_pregel.stream(
        {"input": "data"},
        config={"configurable": {"thread_id": "thread_1"}},
    ):
        stream_parts.append(stream_part)

    assert stream_parts == [
        {"chunk": "data3"},
        {"chunk": "data4"},
        {"__interrupt__": ()},
    ]

    # list stream_mode includes mode names
    stream_parts = []
    for stream_part in remote_pregel.stream(
        {"input": "data"},
        config={"configurable": {"thread_id": "thread_1"}},
        stream_mode=["updates"],
    ):
        stream_parts.append(stream_part)

    assert stream_parts == [
        ("updates", {"chunk": "data3"}),
        ("updates", {"chunk": "data4"}),
        ("updates", {"__interrupt__": ()}),
    ]

    # subgraphs + list modes
    stream_parts = []
    for stream_part in remote_pregel.stream(
        {"input": "data"},
        config={"configurable": {"thread_id": "thread_1"}},
        stream_mode=["updates"],
        subgraphs=True,
    ):
        stream_parts.append(stream_part)

    assert stream_parts == [
        ((), "updates", {"chunk": "data3"}),
        ((), "updates", {"chunk": "data4"}),
        ((), "updates", {"__interrupt__": ()}),
    ]

    # subgraphs + single mode
    stream_parts = []
    for stream_part in remote_pregel.stream(
        {"input": "data"},
        config={"configurable": {"thread_id": "thread_1"}},
        subgraphs=True,
    ):
        stream_parts.append(stream_part)

    assert stream_parts == [
        ((), {"chunk": "data3"}),
        ((), {"chunk": "data4"}),
        ((), {"__interrupt__": ()}),
    ]


@pytest.mark.anyio
async def test_astream():
    # set up test
    mock_async_client = MagicMock()
    async_iter = MagicMock()
    async_iter.__aiter__.return_value = [
        StreamPart(event="values", data={"chunk": "data1"}),
        StreamPart(event="values", data={"chunk": "data2"}),
        StreamPart(event="values", data={"chunk": "data3"}),
        StreamPart(event="updates", data={"chunk": "data4"}),
        StreamPart(
            event="messages",
            data=[
                {
                    "content": [{"text": "Hello", "type": "text", "index": 0}],
                    "type": "AIMessageChunk",
                },
                {
                    "langgraph_step": 1,
                    "langgraph_node": "call_llm",
                    "langgraph_triggers": ["branch:to:call_llm"],
                    "langgraph_path": ["__pregel_pull", "call_llm"],
                },
            ],
        ),
        StreamPart(
            event="updates",
            data={
                "__interrupt__": [
                    {
                        "value": {"question": "Does this look good?"},
                        "id": AnyStr(),
                    }
                ]
            },
        ),
    ]
    mock_async_client.runs.stream.return_value = async_iter

    # call method / assertions
    remote_pregel = RemoteGraph(
        "test_graph_id",
        client=mock_async_client,
    )

    # test raising graph interrupt if invoked as a subgraph
    with pytest.raises(GraphInterrupt) as exc:
        async for stream_part in remote_pregel.astream(
            {"input": "data"},
            # pretend we invoked this as a subgraph
            config={
                "configurable": {"thread_id": "thread_1", "checkpoint_ns": "some_ns"}
            },
            stream_mode="values",
        ):
            pass

    assert exc.value.args[0] == [
        Interrupt(
            value={"question": "Does this look good?"},
            id=AnyStr(),
        )
    ]

    # stream modes doesn't include 'updates'
    stream_parts = []
    async for stream_part in remote_pregel.astream(
        {"input": "data"},
        config={"configurable": {"thread_id": "thread_1"}},
        stream_mode="values",
    ):
        stream_parts.append(stream_part)

    assert stream_parts == [
        {"chunk": "data1"},
        {"chunk": "data2"},
        {"chunk": "data3"},
    ]

    # stream_mode messages
    stream_parts = []
    async for stream_part in remote_pregel.astream(
        {"input": "data"},
        config={"configurable": {"thread_id": "thread_1"}},
        stream_mode="messages",
    ):
        stream_parts.append(stream_part)

    assert stream_parts == [
        (
            {
                "content": [{"text": "Hello", "type": "text", "index": 0}],
                "type": "AIMessageChunk",
            },
            {
                "langgraph_step": 1,
                "langgraph_node": "call_llm",
                "langgraph_triggers": ["branch:to:call_llm"],
                "langgraph_path": ["__pregel_pull", "call_llm"],
            },
        ),
    ]

    async_iter = MagicMock()
    async_iter.__aiter__.return_value = [
        StreamPart(event="updates", data={"chunk": "data3"}),
        StreamPart(event="updates", data={"chunk": "data4"}),
        StreamPart(event="updates", data={"__interrupt__": ()}),
    ]
    mock_async_client.runs.stream.return_value = async_iter

    # default stream_mode is updates
    stream_parts = []
    async for stream_part in remote_pregel.astream(
        {"input": "data"},
        config={"configurable": {"thread_id": "thread_1"}},
    ):
        stream_parts.append(stream_part)

    assert stream_parts == [
        {"chunk": "data3"},
        {"chunk": "data4"},
        {"__interrupt__": ()},
    ]

    # list stream_mode includes mode names
    stream_parts = []
    async for stream_part in remote_pregel.astream(
        {"input": "data"},
        config={"configurable": {"thread_id": "thread_1"}},
        stream_mode=["updates"],
    ):
        stream_parts.append(stream_part)

    assert stream_parts == [
        ("updates", {"chunk": "data3"}),
        ("updates", {"chunk": "data4"}),
        ("updates", {"__interrupt__": ()}),
    ]

    # subgraphs + list modes
    stream_parts = []
    async for stream_part in remote_pregel.astream(
        {"input": "data"},
        config={"configurable": {"thread_id": "thread_1"}},
        stream_mode=["updates"],
        subgraphs=True,
    ):
        stream_parts.append(stream_part)

    assert stream_parts == [
        ((), "updates", {"chunk": "data3"}),
        ((), "updates", {"chunk": "data4"}),
        ((), "updates", {"__interrupt__": ()}),
    ]

    # subgraphs + single mode
    stream_parts = []
    async for stream_part in remote_pregel.astream(
        {"input": "data"},
        config={"configurable": {"thread_id": "thread_1"}},
        subgraphs=True,
    ):
        stream_parts.append(stream_part)

    assert stream_parts == [
        ((), {"chunk": "data3"}),
        ((), {"chunk": "data4"}),
        ((), {"__interrupt__": ()}),
    ]

    async_iter = MagicMock()
    async_iter.__aiter__.return_value = [
        StreamPart(event="updates|my|subgraph", data={"chunk": "data3"}),
        StreamPart(event="updates|hello|subgraph", data={"chunk": "data4"}),
        StreamPart(event="updates|bye|subgraph", data={"__interrupt__": ()}),
    ]
    mock_async_client.runs.stream.return_value = async_iter

    # subgraphs + list modes
    stream_parts = []
    async for stream_part in remote_pregel.astream(
        {"input": "data"},
        config={"configurable": {"thread_id": "thread_1"}},
        stream_mode=["updates"],
        subgraphs=True,
    ):
        stream_parts.append(stream_part)

    assert stream_parts == [
        (("my", "subgraph"), "updates", {"chunk": "data3"}),
        (("hello", "subgraph"), "updates", {"chunk": "data4"}),
        (("bye", "subgraph"), "updates", {"__interrupt__": ()}),
    ]

    # subgraphs + single mode
    stream_parts = []
    async for stream_part in remote_pregel.astream(
        {"input": "data"},
        config={"configurable": {"thread_id": "thread_1"}},
        subgraphs=True,
    ):
        stream_parts.append(stream_part)

    assert stream_parts == [
        (("my", "subgraph"), {"chunk": "data3"}),
        (("hello", "subgraph"), {"chunk": "data4"}),
        (("bye", "subgraph"), {"__interrupt__": ()}),
    ]


def test_invoke():
    # set up test
    mock_sync_client = MagicMock()
    mock_sync_client.runs.stream.return_value = [
        StreamPart(event="values", data={"chunk": "data1"}),
        StreamPart(event="values", data={"chunk": "data2"}),
        StreamPart(
            event="values", data={"messages": [{"type": "human", "content": "world"}]}
        ),
    ]

    # call method / assertions
    remote_pregel = RemoteGraph(
        "test_graph_id",
        sync_client=mock_sync_client,
    )

    config = {"configurable": {"thread_id": "thread_1"}}
    result = remote_pregel.invoke(
        {"input": {"messages": [{"type": "human", "content": "hello"}]}}, config
    )

    assert result == {"messages": [{"type": "human", "content": "world"}]}


@pytest.mark.anyio
async def test_ainvoke():
    # set up test
    mock_async_client = MagicMock()
    async_iter = MagicMock()
    async_iter.__aiter__.return_value = [
        StreamPart(event="values", data={"chunk": "data1"}),
        StreamPart(event="values", data={"chunk": "data2"}),
        StreamPart(
            event="values", data={"messages": [{"type": "human", "content": "world"}]}
        ),
    ]
    mock_async_client.runs.stream.return_value = async_iter

    # call method / assertions
    remote_pregel = RemoteGraph(
        "test_graph_id",
        client=mock_async_client,
    )

    config = {"configurable": {"thread_id": "thread_1"}}
    result = await remote_pregel.ainvoke(
        {"input": {"messages": [{"type": "human", "content": "hello"}]}}, config
    )

    assert result == {"messages": [{"type": "human", "content": "world"}]}


@pytest.mark.skip(
    "Unskip this test to manually test the LangSmith Deployment integration"
)
@pytest.mark.anyio
async def test_langgraph_cloud_integration():
    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph_sdk.client import get_client, get_sync_client

    from langgraph.graph import END, START, MessagesState, StateGraph

    # create RemotePregel instance
    client = get_client()
    sync_client = get_sync_client()
    remote_pregel = RemoteGraph(
        "agent",
        client=client,
        sync_client=sync_client,
    )

    # define graph
    workflow = StateGraph(MessagesState)
    workflow.add_node("agent", remote_pregel)
    workflow.add_edge(START, "agent")
    workflow.add_edge("agent", END)
    app = workflow.compile(checkpointer=InMemorySaver())

    # test invocation
    input = {
        "messages": [
            {
                "role": "human",
                "content": "What's the weather in SF?",
            }
        ]
    }

    # test invoke
    app.invoke(
        input,
        config={"configurable": {"thread_id": "39a6104a-34e7-4f83-929c-d9eb163003c9"}},
        interrupt_before=["agent"],
    )

    # test stream
    async for _ in app.astream(
        input,
        config={"configurable": {"thread_id": "2dc3e3e7-39ac-4597-aa57-4404b944e82a"}},
        subgraphs=True,
        stream_mode=["debug", "messages"],
    ):
        pass

    # test stream events
    async for chunk in remote_pregel.astream_events(
        input,
        config={"configurable": {"thread_id": "2dc3e3e7-39ac-4597-aa57-4404b944e82a"}},
        version="v2",
        subgraphs=True,
        stream_mode=[],
    ):
        pass

    # test get state
    await remote_pregel.aget_state(
        config={"configurable": {"thread_id": "2dc3e3e7-39ac-4597-aa57-4404b944e82a"}},
        subgraphs=True,
    )

    # test update state
    await remote_pregel.aupdate_state(
        config={"configurable": {"thread_id": "6645e002-ed50-4022-92a3-d0d186fdf812"}},
        values={
            "messages": [
                {
                    "role": "ai",
                    "content": "Hello world again!",
                }
            ]
        },
    )

    # test get history
    async for state in remote_pregel.aget_state_history(
        config={"configurable": {"thread_id": "2dc3e3e7-39ac-4597-aa57-4404b944e82a"}},
    ):
        pass

    # test get graph
    remote_pregel.graph_id = "fe096781-5601-53d2-b2f6-0d3403f7e9ca"  # must be UUID
    await remote_pregel.aget_graph(xray=True)


def test_sanitize_config():
    # Create a test instance
    remote = RemoteGraph("test-graph")

    # Test 1: Basic config with primitives
    basic_config: RunnableConfig = {
        "recursion_limit": 10,
        "tags": ["tag1", "tag2"],
        "metadata": {"str_key": "value", "int_key": 42, "bool_key": True},
        "configurable": {"param1": "value1", "param2": 123},
    }
    sanitized = remote._sanitize_config(basic_config)
    assert sanitized["recursion_limit"] == 10
    assert sanitized["tags"] == ["tag1", "tag2"]
    assert sanitized["metadata"] == {
        "str_key": "value",
        "int_key": 42,
        "bool_key": True,
    }
    assert sanitized["configurable"] == {"param1": "value1", "param2": 123}

    # Test 2: Config with non-string tags and complex metadata
    complex_config: RunnableConfig = {
        "tags": ["tag1", 123, {"obj": "tag"}, "tag2"],  # Only string tags should remain
        "metadata": {
            "nested": {
                "key": "value",
                "num": 42,
                "invalid": lambda x: x,
            },  # Last item should be removed
            "list": [1, 2, "three"],
            "invalid": lambda x: x,  # Should be removed
            "tuple": (1, 2, 3),  # Should be converted to list
        },
    }
    sanitized = remote._sanitize_config(complex_config)
    assert sanitized["tags"] == ["tag1", "tag2"]
    assert sanitized["metadata"] == {
        "nested": {"key": "value", "num": 42},
        "list": [1, 2, "three"],
        "tuple": [1, 2, 3],
    }
    assert "invalid" not in sanitized["metadata"]

    # Test 3: Config with configurable fields that should be dropped
    config_with_drops: RunnableConfig = {
        "configurable": {
            "normal_param": "value",
            "checkpoint_map": {"key": "value"},  # Should be dropped
            "checkpoint_id": "123",  # Should be dropped
            "checkpoint_ns": "ns",  # Should be dropped
        }
    }
    sanitized = remote._sanitize_config(config_with_drops)
    assert sanitized["configurable"] == {"normal_param": "value"}
    assert "checkpoint_map" not in sanitized["configurable"]
    assert "checkpoint_id" not in sanitized["configurable"]
    assert "checkpoint_ns" not in sanitized["configurable"]

    # Test 4: Empty config
    empty_config: RunnableConfig = {}
    sanitized = remote._sanitize_config(empty_config)
    assert sanitized == {}

    # Test 5: Config with non-string keys in configurable
    invalid_keys_config: RunnableConfig = {
        "configurable": {
            "valid": "value",
            123: "invalid",  # Should be dropped
            ("tuple", "key"): "invalid",  # Should be dropped
        }
    }
    sanitized = remote._sanitize_config(invalid_keys_config)
    assert sanitized["configurable"] == {"valid": "value"}

    # Test 6: Deeply nested structures
    nested_config: RunnableConfig = {
        "metadata": {
            "level1": {
                "level2": {
                    "level3": {
                        "str": "value",
                        "list": [1, [2, [3]]],
                        "dict": {"a": {"b": {"c": "d"}}},
                    }
                }
            }
        }
    }
    sanitized = remote._sanitize_config(nested_config)
    assert sanitized["metadata"]["level1"]["level2"]["level3"]["str"] == "value"
    assert sanitized["metadata"]["level1"]["level2"]["level3"]["list"] == [1, [2, [3]]]
    assert sanitized["metadata"]["level1"]["level2"]["level3"]["dict"] == {
        "a": {"b": {"c": "d"}}
    }


"""Test RemoteGraph against an actual server."""


@pytest.fixture
def remote_graph() -> RemoteGraph:
    return RemoteGraph("app", url="http://localhost:2024")


@pytest.fixture
def nested_remote_graph(remote_graph: RemoteGraph) -> Pregel:
    class State(TypedDict):
        messages: Annotated[list[AnyMessage], add_messages]

    return (
        StateGraph(State)
        .add_node("nested", remote_graph)
        .add_edge("__start__", "nested")
        .compile(name="nested_remote_graph")
    )


@pytest.fixture
async def nested_graph() -> Pregel:
    class State(TypedDict):
        messages: Annotated[list[AnyMessage], add_messages]

    return (
        StateGraph(State)
        .add_node("nested", app)
        .add_edge("__start__", "nested")
        .compile(name="nested_graph")
    )


def get_message_dict(msg: BaseMessage | dict):
    # just get the core stuff from within the message
    if isinstance(msg, dict):
        return {
            "content": msg.get("content"),
            "type": msg.get("type"),
            "name": msg.get("name"),
            "tool_calls": msg.get("tool_calls"),
            "invalid_tool_calls": msg.get("invalid_tool_calls"),
        }
    return {
        "content": msg.content,
        "type": msg.type,
        "name": msg.name,
        "tool_calls": getattr(msg, "tool_calls", None),
        "invalid_tool_calls": getattr(msg, "invalid_tool_calls", None),
    }


@NEEDS_CONTEXTVARS
@SKIP_PYTHON_314
async def test_remote_graph_basic_invoke(remote_graph: RemoteGraph) -> None:
    # Basic smoke test of the remote graph
    response = await remote_graph.ainvoke(
        {"messages": [{"role": "user", "content": "hello"}]}
    )
    assert response == {
        "content": "answer",
        "additional_kwargs": {},
        "response_metadata": {},
        "type": "ai",
        "name": None,
        "id": "ai3",
        "tool_calls": [],
        "invalid_tool_calls": [],
        "usage_metadata": None,
    }


class monotonic_uid:
    def __init__(self):
        self._uid = 0

    def __call__(self, match=None):
        val = self._uid
        self._uid += 1
        hexval = f"{val:032x}"
        uuid_str = f"{hexval[:8]}-{hexval[8:12]}-{hexval[12:16]}-{hexval[16:20]}-{hexval[20:32]}"
        return uuid_str


uid_pattern = re.compile(
    r"[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}"
)


@NEEDS_CONTEXTVARS
@SKIP_PYTHON_314
async def test_remote_graph_stream_messages_tuple(
    nested_graph: Pregel, nested_remote_graph: Pregel
) -> None:
    events = []
    namespaces = []
    uid_generator = monotonic_uid()
    async for ns, messages in nested_remote_graph.astream(
        {"messages": [{"role": "user", "content": "hello"}]},
        stream_mode="messages",
        subgraphs=True,
    ):
        events.extend(messages)
        namespaces.append(
            tuple(uid_pattern.sub(uid_generator, ns_part) for ns_part in ns)
        )
    inmem_events = []
    inmem_namespaces = []
    uid_generator = monotonic_uid()
    async for ns, messages in nested_graph.astream(
        {"messages": [{"role": "user", "content": "hello"}]},
        stream_mode="messages",
        subgraphs=True,
    ):
        inmem_events.extend(messages)
        inmem_namespaces.append(
            tuple(uid_pattern.sub(uid_generator, ns_part) for ns_part in ns)
        )
    assert len(events) == len(inmem_events)
    assert len(namespaces) == len(inmem_namespaces)

    coerced_events = [get_message_dict(e) for e in events]
    coerced_inmem_events = [get_message_dict(e) for e in inmem_events]
    assert coerced_events == coerced_inmem_events
    # TODO: Fix the namespace matching in the next api release.
    # assert namespaces == inmem_namespaces


@pytest.mark.anyio
@pytest.mark.parametrize("distributed_tracing", [False, True])
@pytest.mark.parametrize("stream", [False, True])
@pytest.mark.parametrize("headers", [None, {"foo": "bar"}])
async def test_include_headers(
    distributed_tracing: bool, stream: bool, headers: dict[str, str] | None
):
    mock_async_client = MagicMock()
    async_iter = MagicMock()
    return_value = [
        StreamPart(event="values", data={"chunk": "data1"}),
    ]
    async_iter.__aiter__.return_value = return_value
    astream_mock = mock_async_client.runs.stream
    astream_mock.return_value = async_iter

    mock_sync_client = MagicMock()
    sync_iter = MagicMock()
    sync_iter.__iter__.return_value = return_value
    stream_mock = mock_sync_client.runs.stream
    stream_mock.return_value = async_iter

    remote_pregel = RemoteGraph(
        "test_graph_id",
        client=mock_async_client,
        sync_client=mock_sync_client,
        distributed_tracing=distributed_tracing,
    )

    config = {"configurable": {"thread_id": "thread_1"}}
    with ls.tracing_context(enabled=True, client=MagicMock()):
        with ls.trace("foo"):
            if stream:
                async for _ in remote_pregel.astream(
                    {"input": {"messages": [{"type": "human", "content": "hello"}]}},
                    config,
                    headers=headers,
                ):
                    pass

            else:
                await remote_pregel.ainvoke(
                    {"input": {"messages": [{"type": "human", "content": "hello"}]}},
                    config,
                    headers=headers,
                )
    expected = headers.copy() if headers else None
    if distributed_tracing:
        if expected is None:
            expected = {}
        expected["langsmith-trace"] = AnyStr()
        expected["baggage"] = AnyStr("langsmith-metadata=")

    assert astream_mock.call_args.kwargs["headers"] == expected
    stream_mock.assert_not_called()

    with ls.tracing_context(enabled=True, client=MagicMock()):
        with ls.trace("foo"):
            if stream:
                for _ in remote_pregel.stream(
                    {"input": {"messages": [{"type": "human", "content": "hello"}]}},
                    config,
                    headers=headers,
                ):
                    pass

            else:
                remote_pregel.invoke(
                    {"input": {"messages": [{"type": "human", "content": "hello"}]}},
                    config,
                    headers=headers,
                )
    assert stream_mock.call_args.kwargs["headers"] == expected

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/test_retry.py
```py
from unittest.mock import Mock, patch

import pytest
from typing_extensions import TypedDict

from langgraph.graph import START, StateGraph
from langgraph.pregel._retry import _should_retry_on
from langgraph.types import RetryPolicy


def test_should_retry_on_single_exception():
    """Test retry with a single exception type."""
    policy = RetryPolicy(retry_on=ValueError)

    # Should retry on ValueError
    assert _should_retry_on(policy, ValueError("test error")) is True

    # Should not retry on other exceptions
    assert _should_retry_on(policy, TypeError("test error")) is False
    assert _should_retry_on(policy, Exception("test error")) is False


def test_should_retry_on_sequence_of_exceptions():
    """Test retry with a sequence of exception types."""
    policy = RetryPolicy(retry_on=(ValueError, KeyError))

    # Should retry on listed exceptions
    assert _should_retry_on(policy, ValueError("test error")) is True
    assert _should_retry_on(policy, KeyError("test error")) is True

    # Should not retry on other exceptions
    assert _should_retry_on(policy, TypeError("test error")) is False
    assert _should_retry_on(policy, Exception("test error")) is False


def test_should_retry_on_subclass_of_exception():
    """Test retry on subclass of specified exception."""

    class CustomError(ValueError):
        pass

    policy = RetryPolicy(retry_on=ValueError)

    # Should retry on subclass of specified exception
    assert _should_retry_on(policy, CustomError("test error")) is True


def test_should_retry_on_callable():
    """Test retry with a callable predicate."""

    # Only retry on ValueError with message containing 'retry'
    def should_retry(exc: Exception) -> bool:
        return isinstance(exc, ValueError) and "retry" in str(exc)

    policy = RetryPolicy(retry_on=should_retry)

    # Should retry when predicate returns True
    assert _should_retry_on(policy, ValueError("please retry this")) is True

    # Should not retry when predicate returns False
    assert _should_retry_on(policy, ValueError("other error")) is False
    assert _should_retry_on(policy, TypeError("please retry this")) is False


def test_should_retry_on_invalid_type():
    """Test retry with an invalid retry_on type."""
    policy = RetryPolicy(retry_on=123)  # type: ignore

    with pytest.raises(TypeError, match="retry_on must be an Exception class"):
        _should_retry_on(policy, ValueError("test error"))


def test_should_retry_on_empty_sequence():
    """Test retry with an empty sequence."""
    policy = RetryPolicy(retry_on=())

    # Should not retry when sequence is empty
    assert _should_retry_on(policy, ValueError("test error")) is False


def test_should_retry_default_retry_on():
    """Test the default retry_on function."""
    import httpx
    import requests

    # Create a RetryPolicy with default_retry_on
    policy = RetryPolicy()

    # Should retry on ConnectionError
    assert _should_retry_on(policy, ConnectionError("connection refused")) is True

    # Should not retry on common programming errors
    assert _should_retry_on(policy, ValueError("invalid value")) is False
    assert _should_retry_on(policy, TypeError("invalid type")) is False
    assert _should_retry_on(policy, ArithmeticError("division by zero")) is False
    assert _should_retry_on(policy, ImportError("module not found")) is False
    assert _should_retry_on(policy, LookupError("key not found")) is False
    assert _should_retry_on(policy, NameError("name not defined")) is False
    assert _should_retry_on(policy, SyntaxError("invalid syntax")) is False
    assert _should_retry_on(policy, RuntimeError("runtime error")) is False
    assert _should_retry_on(policy, ReferenceError("weak reference")) is False
    assert _should_retry_on(policy, StopIteration()) is False
    assert _should_retry_on(policy, StopAsyncIteration()) is False
    assert _should_retry_on(policy, OSError("file not found")) is False

    # Should retry on httpx.HTTPStatusError with 5xx status code
    response_5xx = Mock()
    response_5xx.status_code = 503
    http_error_5xx = httpx.HTTPStatusError(
        "server error", request=Mock(), response=response_5xx
    )
    assert _should_retry_on(policy, http_error_5xx) is True

    # Should not retry on httpx.HTTPStatusError with 4xx status code
    response_4xx = Mock()
    response_4xx.status_code = 404
    http_error_4xx = httpx.HTTPStatusError(
        "not found", request=Mock(), response=response_4xx
    )
    assert _should_retry_on(policy, http_error_4xx) is False

    # Should retry on requests.HTTPError with 5xx status code
    response_req_5xx = Mock()
    response_req_5xx.status_code = 502
    req_error_5xx = requests.HTTPError("bad gateway")
    req_error_5xx.response = response_req_5xx
    assert _should_retry_on(policy, req_error_5xx) is True

    # Should not retry on requests.HTTPError with 4xx status code
    response_req_4xx = Mock()
    response_req_4xx.status_code = 400
    req_error_4xx = requests.HTTPError("bad request")
    req_error_4xx.response = response_req_4xx
    assert _should_retry_on(policy, req_error_4xx) is False

    # Should retry on requests.HTTPError with no response
    req_error_no_resp = requests.HTTPError("connection error")
    req_error_no_resp.response = None
    assert _should_retry_on(policy, req_error_no_resp) is True

    # Should retry on other exceptions by default
    class CustomException(Exception):
        pass

    assert _should_retry_on(policy, CustomException("custom error")) is True


def test_graph_with_single_retry_policy():
    """Test a simple graph with a single RetryPolicy for a node."""

    class State(TypedDict):
        foo: str

    attempt_count = 0

    def failing_node(state: State):
        nonlocal attempt_count
        attempt_count += 1
        if attempt_count < 3:  # Fail the first two attempts
            raise ValueError("Intentional failure")
        return {"foo": "success"}

    def other_node(state: State):
        return {"foo": "other_node"}

    # Create a retry policy with specific parameters
    retry_policy = RetryPolicy(
        max_attempts=3,
        initial_interval=0.01,  # Short interval for tests
        backoff_factor=2.0,
        jitter=False,  # Disable jitter for predictable timing
        retry_on=ValueError,
    )

    # Create and compile the graph
    graph = (
        StateGraph(State)
        .add_node("failing_node", failing_node, retry_policy=retry_policy)
        .add_node("other_node", other_node)
        .add_edge(START, "failing_node")
        .add_edge("failing_node", "other_node")
        .compile()
    )

    with patch("time.sleep") as mock_sleep:
        result = graph.invoke({"foo": ""})

    # Verify retry behavior
    assert attempt_count == 3  # The node should have been tried 3 times
    assert result["foo"] == "other_node"  # Final result should be from other_node

    # Verify the sleep intervals
    call_args_list = [args[0][0] for args in mock_sleep.call_args_list]
    assert call_args_list == [0.01, 0.02]


def test_graph_with_jitter_retry_policy():
    """Test a graph with a RetryPolicy that uses jitter."""

    class State(TypedDict):
        foo: str

    attempt_count = 0

    def failing_node(state):
        nonlocal attempt_count
        attempt_count += 1
        if attempt_count < 2:  # Fail the first attempt
            raise ValueError("Intentional failure")
        return {"foo": "success"}

    # Create a retry policy with jitter enabled
    retry_policy = RetryPolicy(
        max_attempts=3,
        initial_interval=0.01,
        jitter=True,  # Enable jitter for randomized backoff
        retry_on=ValueError,
    )

    # Create and compile the graph
    graph = (
        StateGraph(State)
        .add_node("failing_node", failing_node, retry_policy=retry_policy)
        .add_edge(START, "failing_node")
        .compile()
    )

    # Test graph execution with mocked random and sleep
    with (
        patch("random.uniform", return_value=0.05) as mock_random,
        patch("time.sleep") as mock_sleep,
    ):
        result = graph.invoke({"foo": ""})

    # Verify retry behavior
    assert attempt_count == 2  # The node should have been tried twice
    assert result["foo"] == "success"

    # Verify jitter was applied
    mock_random.assert_called_with(0, 1)  # Jitter should use random.uniform(0, 1)
    mock_sleep.assert_called_with(0.01 + 0.05)  # Sleep should include jitter


def test_graph_with_multiple_retry_policies():
    """Test a graph with multiple retry policies for a node."""

    class State(TypedDict):
        foo: str
        error_type: str

    attempt_counts = {"value_error": 0, "key_error": 0}

    def failing_node(state):
        error_type = state["error_type"]

        if error_type == "value_error":
            attempt_counts["value_error"] += 1
            if attempt_counts["value_error"] < 2:
                raise ValueError("Value error")
        elif error_type == "key_error":
            attempt_counts["key_error"] += 1
            if attempt_counts["key_error"] < 3:
                raise KeyError("Key error")

        return {"foo": f"recovered_from_{error_type}"}

    # Create multiple retry policies
    value_error_policy = RetryPolicy(
        max_attempts=2,
        initial_interval=0.01,
        jitter=False,
        retry_on=ValueError,
    )

    key_error_policy = RetryPolicy(
        max_attempts=3,
        initial_interval=0.02,
        jitter=False,
        retry_on=KeyError,
    )

    # Create and compile the graph with a list of retry policies
    graph = (
        StateGraph(State)
        .add_node(
            "failing_node",
            failing_node,
            retry_policy=(value_error_policy, key_error_policy),
        )
        .add_edge(START, "failing_node")
        .compile()
    )

    # Test ValueError scenario
    with patch("time.sleep"):
        result_value_error = graph.invoke({"foo": "", "error_type": "value_error"})

    assert attempt_counts["value_error"] == 2
    assert result_value_error["foo"] == "recovered_from_value_error"

    # Reset attempt counts
    attempt_counts = {"value_error": 0, "key_error": 0}

    # Test KeyError scenario
    with patch("time.sleep"):
        result_key_error = graph.invoke({"foo": "", "error_type": "key_error"})

    assert attempt_counts["key_error"] == 3
    assert result_key_error["foo"] == "recovered_from_key_error"


def test_graph_with_max_attempts_exceeded():
    """Test a graph where max_attempts is exceeded."""

    class State(TypedDict):
        foo: str

    def always_failing_node(state):
        raise ValueError("Always fails")

    # Create a retry policy with limited attempts
    retry_policy = RetryPolicy(
        max_attempts=2,
        initial_interval=0.01,
        jitter=False,
        retry_on=ValueError,
    )

    # Create and compile the graph
    graph = (
        StateGraph(State)
        .add_node("always_failing", always_failing_node, retry_policy=retry_policy)
        .add_edge(START, "always_failing")
        .compile()
    )

    # Test graph execution
    with (
        patch("time.sleep") as mock_sleep,
        pytest.raises(ValueError, match="Always fails"),
    ):
        graph.invoke({"foo": ""})

    mock_sleep.assert_called_with(0.01)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/test_runnable.py
```py
from __future__ import annotations

from typing import Any, Optional

import pytest
from langchain_core.runnables.config import RunnableConfig
from langgraph.store.base import BaseStore

from langgraph._internal._runnable import RunnableCallable
from langgraph.runtime import Runtime
from langgraph.types import StreamWriter

pytestmark = pytest.mark.anyio


def test_runnable_callable_func_accepts():
    def sync_func(x: Any) -> str:
        return f"{x}"

    async def async_func(x: Any) -> str:
        return f"{x}"

    def func_with_store(x: Any, store: BaseStore) -> str:
        return f"{x}"

    def func_with_writer(x: Any, writer: StreamWriter) -> str:
        return f"{x}"

    async def afunc_with_store(x: Any, store: BaseStore) -> str:
        return f"{x}"

    async def afunc_with_writer(x: Any, writer: StreamWriter) -> str:
        return f"{x}"

    runnables = {
        "sync": RunnableCallable(sync_func),
        "async": RunnableCallable(func=None, afunc=async_func),
        "with_store": RunnableCallable(func_with_store),
        "with_writer": RunnableCallable(func_with_writer),
        "awith_store": RunnableCallable(afunc_with_store),
        "awith_writer": RunnableCallable(afunc_with_writer),
    }

    expected_store = {"with_store": True, "awith_store": True}
    expected_writer = {"with_writer": True, "awith_writer": True}

    for name, runnable in runnables.items():
        if expected_writer.get(name, False):
            assert "writer" in runnable.func_accepts
        else:
            assert "writer" not in runnable.func_accepts

        if expected_store.get(name, False):
            assert "store" in runnable.func_accepts
        else:
            assert "store" not in runnable.func_accepts


async def test_runnable_callable_basic():
    def sync_func(x: Any) -> str:
        return f"{x}"

    async def async_func(x: Any) -> str:
        return f"{x}"

    runnable_sync = RunnableCallable(sync_func)
    runnable_async = RunnableCallable(func=None, afunc=async_func)

    result_sync = runnable_sync.invoke("test")
    assert result_sync == "test"

    # Test asynchronous ainvoke
    result_async = await runnable_async.ainvoke("test")
    assert result_async == "test"


def test_runnable_callable_injectable_arguments() -> None:
    """Test injectable arguments for RunnableCallable.

    This test verifies that injectable arguments like BaseStore work correctly.
    It tests:
    - Optional store injection
    - Required store injection
    - Store injection via config
    - Store injection override behavior
    - Store value injection and validation
    """

    # Test Optional[BaseStore] annotation.
    def func_optional_store(inputs: Any, store: Optional[BaseStore]) -> str:  # noqa: UP045
        """Test function that accepts an optional store parameter."""
        assert store is None
        return "success"

    assert (
        RunnableCallable(func_optional_store).invoke(
            {"x": "1"},
            config={
                "configurable": {
                    "__pregel_runtime": Runtime(
                        store=None,
                        context=None,
                        stream_writer=lambda _: None,
                        previous=None,
                    )
                }
            },
        )
        == "success"
    )

    # Test BaseStore annotation
    def func_required_store(inputs: Any, store: BaseStore) -> str:
        """Test function that requires a store parameter."""
        assert store is None
        return "success"

    with pytest.raises(ValueError):
        # Should fail b/c store is not Optional and config is not populated with store.
        assert RunnableCallable(func_required_store).invoke({}) == "success"

    # Manually provide store
    assert RunnableCallable(func_required_store).invoke({}, store=None) == "success"

    # Specify a value for store in the config
    assert (
        RunnableCallable(func_required_store).invoke(
            {},
            config={
                "configurable": {
                    "__pregel_runtime": Runtime(
                        store=None,
                        context=None,
                        stream_writer=lambda _: None,
                        previous=None,
                    )
                }
            },
        )
        == "success"
    )

    # Specify a value for store in config, but override with None
    assert (
        RunnableCallable(func_optional_store).invoke(
            {"x": "1"},
            store=None,
            config={
                "configurable": {
                    "__pregel_runtime": Runtime(
                        store="foobar",  # type: ignore[assignment]
                        context=None,
                        stream_writer=lambda _: None,
                        previous=None,
                    )
                }
            },
        )
        == "success"
    )

    # Set of tests where we verify that 'foobar' is injected as the store value.
    def func_required_store_v2(inputs: Any, store: BaseStore) -> str:
        """Test function that requires a store parameter and validates its value.

        The store value is expected to be 'foobar' when injected.
        """
        assert store == "foobar"
        return "success"

    assert (
        RunnableCallable(func_required_store_v2).invoke(
            {},
            config={
                "configurable": {
                    "__pregel_runtime": Runtime(
                        store="foobar",  # type: ignore[assignment]
                        context=None,
                        stream_writer=lambda _: None,
                        previous=None,
                    )
                }
            },
        )
        == "success"
    )

    assert RunnableCallable(func_required_store_v2).invoke(
        # And manual override takes precedence.
        {},
        store="foobar",
        config={
            "configurable": {
                "__pregel_runtime": Runtime(
                    store="foobar",  # type: ignore[assignment]
                    context=None,
                    stream_writer=lambda _: None,
                    previous=None,
                )
            }
        },
    )


async def test_runnable_callable_injectable_arguments_async() -> None:
    """Test injectable arguments for async RunnableCallable.

    This test verifies that injectable arguments like BaseStore work correctly
    in the async context. It tests:
    - Optional store injection
    - Required store injection
    - Store injection via config
    - Store injection override behavior
    """

    # Test Optional[BaseStore] annotation.
    def func_optional_store(inputs: Any, store: Optional[BaseStore]) -> str:  # noqa: UP045
        """Test function that accepts an optional store parameter."""
        assert store is None
        return "success"

    async def afunc_optional_store(inputs: Any, store: BaseStore | None) -> str:
        """Async version of func_optional_store."""
        assert store is None
        return "success"

    assert (
        await RunnableCallable(
            func=func_optional_store, afunc=afunc_optional_store
        ).ainvoke({"x": "1"})
        == "success"
    )

    # Test BaseStore annotation
    def func_required_store(inputs: Any, store: BaseStore) -> str:
        """Test function that requires a store parameter."""
        assert store is None
        return "success"

    async def afunc_required_store(inputs: Any, store: BaseStore) -> str:
        """Async version of func_required_store."""
        assert store is None
        return "success"

    with pytest.raises(ValueError):
        # Should fail b/c store is not Optional and config is not populated with store.
        assert (
            await RunnableCallable(
                func=func_required_store, afunc=afunc_required_store
            ).ainvoke(
                {},
            )
            == "success"
        )

    # Manually provide store
    assert (
        await RunnableCallable(
            func=func_required_store, afunc=afunc_required_store
        ).ainvoke(
            {},
            store=None,
            config={
                "configurable": {
                    "__pregel_runtime": Runtime(
                        store=None,
                        context=None,
                        stream_writer=lambda _: None,
                        previous=None,
                    )
                }
            },
        )
        == "success"
    )

    # Specify a value for store in the config
    assert (
        await RunnableCallable(
            func=func_required_store, afunc=afunc_required_store
        ).ainvoke(
            {},
            config={
                "configurable": {
                    "__pregel_runtime": Runtime(
                        store=None,
                        context=None,
                        stream_writer=lambda _: None,
                        previous=None,
                    )
                }
            },
        )
        == "success"
    )

    # Specify a value for store in config, but override with None
    assert (
        await RunnableCallable(
            func=func_optional_store, afunc=afunc_optional_store
        ).ainvoke(
            {"x": "1"},
            store=None,
            config={
                "configurable": {
                    "__pregel_runtime": Runtime(
                        store="foobar",
                        context=None,
                        stream_writer=lambda _: None,
                        previous=None,
                    )
                }
            },
        )
        == "success"
    )

    # Set of tests where we verify that 'foobar' is injected as the store value.
    def func_required_store_v2(inputs: Any, store: BaseStore) -> str:
        """Test function that requires a store parameter with specific value.

        The store parameter is expected to be 'foobar' when injected.
        """
        assert store == "foobar"
        return "success"

    async def afunc_required_store_v2(inputs: Any, store: BaseStore) -> str:
        """Async version of func_required_store_v2.

        The store parameter is expected to be 'foobar' when injected.
        """
        assert store == "foobar"
        return "success"

    assert (
        await RunnableCallable(
            func=func_required_store_v2, afunc=afunc_required_store_v2
        ).ainvoke(
            {},
            config={
                "configurable": {
                    "__pregel_runtime": Runtime(
                        store="foobar",
                        context=None,
                        stream_writer=lambda _: None,
                        previous=None,
                    )
                }
            },
        )
        == "success"
    )

    assert (
        await RunnableCallable(
            func=func_required_store_v2, afunc=afunc_required_store_v2
        ).ainvoke(
            # And manual override takes precedence.
            {},
            store="foobar",
            config={
                "configurable": {
                    "__pregel_runtime": Runtime(
                        store="foobar",
                        context=None,
                        stream_writer=lambda _: None,
                        previous=None,
                    )
                }
            },
        )
        == "success"
    )


def test_config_injection() -> None:
    def func(x: Any, config: RunnableConfig) -> list[str]:
        return config.get("tags", [])

    assert RunnableCallable(func).invoke(
        "test", config={"tags": ["test"], "configurable": {}}
    ) == ["test"]

    def func_optional(x: Any, config: Optional[RunnableConfig]) -> list[str]:  # noqa: UP045
        return config.get("tags", []) if config else []

    assert RunnableCallable(func_optional).invoke(
        "test", config={"tags": ["test"], "configurable": {}}
    ) == ["test"]

    def func_untyped(x: Any, config) -> list[str]:
        return config.get("tags", [])

    assert RunnableCallable(func_untyped).invoke(
        "test", config={"tags": ["test"], "configurable": {}}
    ) == ["test"]


def test_config_ensured() -> None:
    def func(input: str, config: RunnableConfig) -> None:
        assert input == "test"
        assert config is not None
        assert config.get("configurable") is not None

    RunnableCallable(func).invoke("test")


async def test_config_ensured_async() -> None:
    async def func(input: str, config: RunnableConfig) -> None:
        assert input == "test"
        assert config is not None
        assert config.get("configurable") is not None

    await RunnableCallable(func).ainvoke("test")

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/test_runtime.py
```py
from dataclasses import dataclass
from typing import Any

import pytest
from pydantic import BaseModel, ValidationError
from typing_extensions import TypedDict

from langgraph.graph import END, START, StateGraph
from langgraph.runtime import Runtime, get_runtime


def test_injected_runtime() -> None:
    @dataclass
    class Context:
        api_key: str

    class State(TypedDict):
        message: str

    def injected_runtime(state: State, runtime: Runtime[Context]) -> dict[str, Any]:
        return {"message": f"api key: {runtime.context.api_key}"}

    graph = StateGraph(state_schema=State, context_schema=Context)
    graph.add_node("injected_runtime", injected_runtime)
    graph.add_edge(START, "injected_runtime")
    graph.add_edge("injected_runtime", END)
    compiled = graph.compile()
    result = compiled.invoke(
        {"message": "hello world"}, context=Context(api_key="sk_123456")
    )
    assert result == {"message": "api key: sk_123456"}


def test_context_runtime() -> None:
    @dataclass
    class Context:
        api_key: str

    class State(TypedDict):
        message: str

    def context_runtime(state: State) -> dict[str, Any]:
        runtime = get_runtime(Context)
        return {"message": f"api key: {runtime.context.api_key}"}

    graph = StateGraph(state_schema=State, context_schema=Context)
    graph.add_node("context_runtime", context_runtime)
    graph.add_edge(START, "context_runtime")
    graph.add_edge("context_runtime", END)
    compiled = graph.compile()
    result = compiled.invoke(
        {"message": "hello world"}, context=Context(api_key="sk_123456")
    )
    assert result == {"message": "api key: sk_123456"}


def test_override_runtime() -> None:
    @dataclass
    class Context:
        api_key: str

    prev = Runtime(context=Context(api_key="abc"))
    new = prev.override(context=Context(api_key="def"))
    assert new.override(context=Context(api_key="def")).context.api_key == "def"


def test_merge_runtime() -> None:
    @dataclass
    class Context:
        api_key: str

    runtime1 = Runtime(context=Context(api_key="abc"))
    runtime2 = Runtime(context=Context(api_key="def"))
    runtime3 = Runtime(context=None)

    assert runtime1.merge(runtime2).context.api_key == "def"
    # override only applies to non-falsy values
    assert runtime1.merge(runtime3).context.api_key == "abc"  # type: ignore


def test_runtime_propogated_to_subgraph() -> None:
    @dataclass
    class Context:
        username: str

    class State(TypedDict, total=False):
        subgraph: str
        main: str

    def subgraph_node_1(state: State, runtime: Runtime[Context]):
        return {"subgraph": f"{runtime.context.username}!"}

    subgraph_builder = StateGraph(State, context_schema=Context)
    subgraph_builder.add_node(subgraph_node_1)
    subgraph_builder.set_entry_point("subgraph_node_1")
    subgraph = subgraph_builder.compile()

    def main_node(state: State, runtime: Runtime[Context]):
        return {"main": f"{runtime.context.username}!"}

    builder = StateGraph(State, context_schema=Context)
    builder.add_node(main_node)
    builder.add_node("node_1", subgraph)
    builder.set_entry_point("main_node")
    builder.add_edge("main_node", "node_1")
    graph = builder.compile()

    context = Context(username="Alice")
    result = graph.invoke({}, context=context)
    assert result == {"subgraph": "Alice!", "main": "Alice!"}


def test_context_coercion_dataclass() -> None:
    """Test that dict context is coerced to dataclass."""

    @dataclass
    class Context:
        api_key: str
        timeout: int = 30

    class State(TypedDict):
        message: str

    def node_with_context(state: State, runtime: Runtime[Context]) -> dict[str, Any]:
        return {
            "message": f"api_key: {runtime.context.api_key}, timeout: {runtime.context.timeout}"
        }

    graph = StateGraph(state_schema=State, context_schema=Context)
    graph.add_node("node", node_with_context)
    graph.add_edge(START, "node")
    graph.add_edge("node", END)
    compiled = graph.compile()

    # Test dict coercion with all fields
    result = compiled.invoke(
        {"message": "test"}, context={"api_key": "sk_test", "timeout": 60}
    )
    assert result == {"message": "api_key: sk_test, timeout: 60"}

    # Test dict coercion with default field
    result = compiled.invoke({"message": "test"}, context={"api_key": "sk_test2"})
    assert result == {"message": "api_key: sk_test2, timeout: 30"}

    # Test with actual dataclass instance (should still work)
    result = compiled.invoke(
        {"message": "test"}, context=Context(api_key="sk_test3", timeout=90)
    )
    assert result == {"message": "api_key: sk_test3, timeout: 90"}


def test_context_coercion_pydantic() -> None:
    """Test that dict context is coerced to Pydantic model."""

    class Context(BaseModel):
        api_key: str
        timeout: int = 30
        tags: list[str] = []

    class State(TypedDict):
        message: str

    def node_with_context(state: State, runtime: Runtime[Context]) -> dict[str, Any]:
        return {
            "message": f"api_key: {runtime.context.api_key}, timeout: {runtime.context.timeout}, tags: {runtime.context.tags}"
        }

    graph = StateGraph(state_schema=State, context_schema=Context)
    graph.add_node("node", node_with_context)
    graph.add_edge(START, "node")
    graph.add_edge("node", END)
    compiled = graph.compile()

    # Test dict coercion with all fields
    result = compiled.invoke(
        {"message": "test"},
        context={"api_key": "sk_test", "timeout": 60, "tags": ["prod", "v2"]},
    )
    assert result == {"message": "api_key: sk_test, timeout: 60, tags: ['prod', 'v2']"}

    # Test dict coercion with defaults
    result = compiled.invoke({"message": "test"}, context={"api_key": "sk_test2"})
    assert result == {"message": "api_key: sk_test2, timeout: 30, tags: []"}

    # Test with actual Pydantic instance (should still work)
    result = compiled.invoke(
        {"message": "test"},
        context=Context(api_key="sk_test3", timeout=90, tags=["test"]),
    )
    assert result == {"message": "api_key: sk_test3, timeout: 90, tags: ['test']"}


def test_context_coercion_typeddict() -> None:
    """Test that dict context with TypedDict schema passes through as-is."""

    class Context(TypedDict):
        api_key: str
        timeout: int

    class State(TypedDict):
        message: str

    def node_with_context(state: State, runtime: Runtime[Context]) -> dict[str, Any]:
        # TypedDict context is just a dict at runtime
        return {
            "message": f"api_key: {runtime.context['api_key']}, timeout: {runtime.context['timeout']}"
        }

    graph = StateGraph(state_schema=State, context_schema=Context)
    graph.add_node("node", node_with_context)
    graph.add_edge(START, "node")
    graph.add_edge("node", END)
    compiled = graph.compile()

    # Test dict passes through for TypedDict
    result = compiled.invoke(
        {"message": "test"}, context={"api_key": "sk_test", "timeout": 60}
    )
    assert result == {"message": "api_key: sk_test, timeout: 60"}


def test_context_coercion_none() -> None:
    """Test that None context is handled properly."""

    @dataclass
    class Context:
        api_key: str

    class State(TypedDict):
        message: str

    def node_without_context(state: State, runtime: Runtime[Context]) -> dict[str, Any]:
        # Should be None when no context provided
        return {"message": f"context is None: {runtime.context is None}"}

    graph = StateGraph(state_schema=State, context_schema=Context)
    graph.add_node("node", node_without_context)
    graph.add_edge(START, "node")
    graph.add_edge("node", END)
    compiled = graph.compile()

    # Test with None context
    result = compiled.invoke({"message": "test"}, context=None)
    assert result == {"message": "context is None: True"}

    # Test without context parameter (defaults to None)
    result = compiled.invoke({"message": "test"})
    assert result == {"message": "context is None: True"}


def test_context_coercion_errors() -> None:
    """Test error handling for invalid context."""

    @dataclass
    class Context:
        api_key: str  # Required field

    class State(TypedDict):
        message: str

    def node_with_context(state: State, runtime: Runtime[Context]) -> dict[str, Any]:
        return {"message": "should not reach here"}

    graph = StateGraph(state_schema=State, context_schema=Context)
    graph.add_node("node", node_with_context)
    graph.add_edge(START, "node")
    graph.add_edge("node", END)
    compiled = graph.compile()

    # Test missing required field
    with pytest.raises(TypeError):
        compiled.invoke({"message": "test"}, context={"timeout": 60})

    # Test invalid dict keys
    with pytest.raises(TypeError):
        compiled.invoke(
            {"message": "test"}, context={"api_key": "test", "invalid_field": "value"}
        )


@pytest.mark.anyio
async def test_context_coercion_async() -> None:
    """Test context coercion with async methods."""

    @dataclass
    class Context:
        api_key: str
        async_mode: bool = True

    class State(TypedDict):
        message: str

    async def async_node(state: State, runtime: Runtime[Context]) -> dict[str, Any]:
        return {
            "message": f"async api_key: {runtime.context.api_key}, async_mode: {runtime.context.async_mode}"
        }

    graph = StateGraph(state_schema=State, context_schema=Context)
    graph.add_node("node", async_node)
    graph.add_edge(START, "node")
    graph.add_edge("node", END)
    compiled = graph.compile()

    # Test dict coercion with ainvoke
    result = await compiled.ainvoke(
        {"message": "test"}, context={"api_key": "sk_async", "async_mode": False}
    )
    assert result == {"message": "async api_key: sk_async, async_mode: False"}

    # Test dict coercion with astream
    chunks = []
    async for chunk in compiled.astream(
        {"message": "test"}, context={"api_key": "sk_stream"}
    ):
        chunks.append(chunk)

    # Find the chunk with our node output
    node_output = None
    for chunk in chunks:
        if "node" in chunk:
            node_output = chunk["node"]
            break

    assert node_output == {"message": "async api_key: sk_stream, async_mode: True"}


def test_context_coercion_stream() -> None:
    """Test context coercion with sync stream method."""

    @dataclass
    class Context:
        api_key: str
        stream_mode: str = "default"

    class State(TypedDict):
        message: str

    def node_with_context(state: State, runtime: Runtime[Context]) -> dict[str, Any]:
        return {
            "message": f"stream api_key: {runtime.context.api_key}, mode: {runtime.context.stream_mode}"
        }

    graph = StateGraph(state_schema=State, context_schema=Context)
    graph.add_node("node", node_with_context)
    graph.add_edge(START, "node")
    graph.add_edge("node", END)
    compiled = graph.compile()

    # Test dict coercion with stream
    chunks = []
    for chunk in compiled.stream(
        {"message": "test"}, context={"api_key": "sk_stream", "stream_mode": "fast"}
    ):
        chunks.append(chunk)

    # Find the chunk with our node output
    node_output = None
    for chunk in chunks:
        if "node" in chunk:
            node_output = chunk["node"]
            break

    assert node_output == {"message": "stream api_key: sk_stream, mode: fast"}


def test_context_coercion_pydantic_validation_errors() -> None:
    """Test that Pydantic validation errors are raised."""

    class Context(BaseModel):
        api_key: str
        timeout: int

    class State(TypedDict):
        message: str

    def node_with_context(state: State, runtime: Runtime[Context]) -> dict[str, Any]:
        return {
            "message": f"api_key: {runtime.context.api_key}, timeout: {runtime.context.timeout}"
        }

    graph = StateGraph(state_schema=State, context_schema=Context)
    graph.add_node("node", node_with_context)
    graph.add_edge(START, "node")
    graph.add_edge("node", END)

    compiled = graph.compile()

    with pytest.raises(ValidationError):
        compiled.invoke(
            {"message": "test"}, context={"api_key": "sk_test", "timeout": "not_an_int"}
        )

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/test_state.py
```py
import inspect
import operator
import warnings
from dataclasses import dataclass, field
from typing import Annotated, Any, Union
from typing import Annotated as Annotated2

import pytest
from langchain_core.runnables import RunnableConfig
from pydantic import BaseModel
from typing_extensions import NotRequired, Required, TypedDict

from langgraph.channels.binop import BinaryOperatorAggregate
from langgraph.channels.ephemeral_value import EphemeralValue
from langgraph.graph.state import (
    StateGraph,
    _get_node_name,
    _is_field_channel,
    _warn_invalid_state_schema,
)


class State(BaseModel):
    foo: str
    bar: int


class State2(TypedDict):
    foo: str
    bar: int


@pytest.mark.parametrize(
    "schema",
    [
        {"foo": "bar"},
        ["hi", lambda x, y: x + y],
        State(foo="bar", bar=1),
        State2(foo="bar", bar=1),
    ],
)
def test_warns_invalid_schema(schema: Any):
    with pytest.warns(UserWarning):
        _warn_invalid_state_schema(schema)


@pytest.mark.parametrize(
    "schema",
    [
        Annotated[dict, lambda x, y: y],
        Annotated2[list, lambda x, y: y],
        dict,
        State,
        State2,
    ],
)
def test_doesnt_warn_valid_schema(schema: Any):
    # Assert the function does not raise a warning
    with warnings.catch_warnings():
        warnings.simplefilter("error")
        _warn_invalid_state_schema(schema)


def test_state_schema_with_type_hint():
    class InputState(TypedDict):
        question: str

    class OutputState(TypedDict):
        input_state: InputState

    class FooState(InputState):
        foo: str

    def complete_hint(state: InputState) -> OutputState:
        return {"input_state": state}

    def miss_first_hint(state, config: RunnableConfig) -> OutputState:
        return {"input_state": state}

    def only_return_hint(state, config) -> OutputState:
        return {"input_state": state}

    def miss_all_hint(state, config):
        return {"input_state": state}

    def pre_foo(_) -> FooState:
        return {"foo": "bar"}

    def pre_bar(_) -> FooState:
        return {"foo": "bar"}

    class Foo:
        def __call__(self, state: FooState) -> OutputState:
            assert state.pop("foo") == "bar"
            return {"input_state": state}

    class Bar:
        def my_node(self, state: FooState) -> OutputState:
            assert state.pop("foo") == "bar"
            return {"input_state": state}

    graph = StateGraph(InputState, output_schema=OutputState)
    actions = [
        complete_hint,
        miss_first_hint,
        only_return_hint,
        miss_all_hint,
        pre_foo,
        Foo(),
        pre_bar,
        Bar().my_node,
    ]

    for action in actions:
        graph.add_node(action)

    def get_name(action) -> str:
        return getattr(action, "__name__", action.__class__.__name__)

    graph.set_entry_point(get_name(actions[0]))
    for i in range(len(actions) - 1):
        graph.add_edge(get_name(actions[i]), get_name(actions[i + 1]))
    graph.set_finish_point(get_name(actions[-1]))

    graph = graph.compile()

    input_state = InputState(question="Hello World!")
    output_state = OutputState(input_state=input_state)
    foo_state = FooState(foo="bar")
    for i, c in enumerate(graph.stream(input_state, stream_mode="updates")):
        node_name = get_name(actions[i])
        if node_name in {"pre_foo", "pre_bar"}:
            assert c[node_name] == foo_state
        else:
            assert c[node_name] == output_state


@pytest.mark.parametrize("total_", [True, False])
def test_state_schema_optional_values(total_: bool):
    class SomeParentState(TypedDict):
        val0a: str
        val0b: str | None

    class InputState(SomeParentState, total=total_):  # type: ignore
        val1: str
        val2: str | None
        val3: Required[Annotated[dict, operator.or_]]
        val4: NotRequired[dict]
        val5: Annotated[Required[str], "foo"]
        val6: Annotated[NotRequired[str], "bar"]

    class OutputState(SomeParentState, total=total_):  # type: ignore
        out_val1: str
        out_val2: str | None
        out_val3: Required[str]
        out_val4: NotRequired[dict]
        out_val5: Annotated[Required[str], "foo"]
        out_val6: Annotated[NotRequired[str], "bar"]

    class State(InputState):  # this would be ignored
        val4: dict

    builder = StateGraph(State, input_schema=InputState, output_schema=OutputState)
    builder.add_node("n", lambda x: x)
    builder.add_edge("__start__", "n")
    graph = builder.compile()
    json_schema = graph.get_input_jsonschema()

    assert isinstance(graph.channels["val3"], BinaryOperatorAggregate)

    if total_ is False:
        expected_required = set()
        expected_optional = {"val2", "val1"}
    else:
        expected_required = {"val1", "val2"}
        expected_optional = set()

    # The others should always have precedence based on the required annotation
    expected_required |= {"val0a", "val0b", "val3", "val5"}
    expected_optional |= {"val4", "val6"}

    assert set(json_schema.get("required", set())) == expected_required
    assert (
        set(json_schema["properties"].keys()) == expected_required | expected_optional
    )

    # Check output schema. Should be the same process
    output_schema = graph.get_output_jsonschema()
    if total_ is False:
        expected_required = set()
        expected_optional = {"out_val2", "out_val1"}
    else:
        expected_required = {"out_val1", "out_val2"}
        expected_optional = set()

    expected_required |= {"val0a", "val0b", "out_val3", "out_val5"}
    expected_optional |= {"out_val4", "out_val6"}

    assert set(output_schema.get("required", set())) == expected_required
    assert (
        set(output_schema["properties"].keys()) == expected_required | expected_optional
    )


@pytest.mark.parametrize("kw_only_", [False, True])
def test_state_schema_default_values(kw_only_: bool):
    kwargs = {}
    if "kw_only" in inspect.signature(dataclass).parameters:
        kwargs = {"kw_only": kw_only_}

    @dataclass(**kwargs)
    class InputState:
        val1: str
        val2: int | None
        val3: Annotated[float | None, "optional annotated"]
        val4: str | None = None
        val5: list[int] = field(default_factory=lambda: [1, 2, 3])
        val6: dict[str, int] = field(default_factory=lambda: {"a": 1})
        val7: str = field(default=...)
        val8: Annotated[int, "some metadata"] = 42
        val9: Annotated[str, "more metadata"] = field(default="some foo")
        val10: str = "default"
        val11: Annotated[list[str], "annotated list"] = field(
            default_factory=lambda: ["a", "b"]
        )

    builder = StateGraph(InputState)
    builder.add_node("n", lambda x: x)
    builder.add_edge("__start__", "n")
    graph = builder.compile()
    for json_schema in [graph.get_input_jsonschema(), graph.get_output_jsonschema()]:
        expected_required = {"val1", "val7"}
        expected_optional = {
            "val2",
            "val3",
            "val4",
            "val5",
            "val6",
            "val8",
            "val9",
            "val10",
            "val11",
        }

    assert set(json_schema.get("required", set())) == expected_required
    assert (
        set(json_schema["properties"].keys()) == expected_required | expected_optional
    )


def test__get_node_name() -> None:
    # lambda
    assert _get_node_name(lambda x: x) == "<lambda>"

    # regular function
    def func(state):
        return

    assert _get_node_name(func) == "func"

    class MyClass:
        def __call__(self, state):
            return

        def class_method(self, state):
            return

    # callable class
    assert _get_node_name(MyClass()) == "MyClass"

    # class method
    assert _get_node_name(MyClass().class_method) == "class_method"


def test_input_schema_conditional_edge():
    class OverallState(TypedDict):
        foo: Annotated[int, operator.add]
        bar: str

    class PrivateState(TypedDict):
        baz: str

    builder = StateGraph(OverallState)

    def node_1(state: OverallState):
        return {"foo": 1, "baz": "bar"}

    def node_2(state: PrivateState):
        return {"foo": 1, "bar": state["baz"], "something_else": "meow"}

    def node_3(state: OverallState):
        return {"foo": 1}

    def router(state: OverallState):
        assert state == {"foo": 2, "bar": "bar"}
        if state["foo"] == 2:
            return "node_3"
        else:
            return "__end__"

    builder.add_node(node_1)
    builder.add_node(node_2)
    builder.add_node(node_3)
    builder.add_conditional_edges("node_2", router)
    builder.add_edge("__start__", "node_1")
    builder.add_edge("node_1", "node_2")
    graph = builder.compile()
    assert graph.invoke({"foo": 0}) == {"foo": 3, "bar": "bar"}


def test_private_input_schema_conditional_edge():
    class OverallState(TypedDict):
        foo: Annotated[int, operator.add]
        bar: str

    class RouterState(TypedDict):
        baz: str

    class Node2State(TypedDict):
        foo: Annotated[int, operator.add]
        baz: str

    builder = StateGraph(OverallState)

    def node_1(state: OverallState):
        return {"foo": 1, "baz": "meow"}

    def node_2(state: Node2State):
        return {"foo": 1, "bar": state["baz"]}

    def router(state: RouterState):
        assert state == {"baz": "meow"}
        if state["baz"] == "meow":
            return "node_2"
        else:
            return "__end__"

    builder.add_node(node_1)
    builder.add_node(node_2)
    builder.add_conditional_edges("node_1", router)
    builder.add_edge("__start__", "node_1")
    graph = builder.compile()
    assert graph.invoke({"foo": 0}) == {"foo": 2, "bar": "meow"}


def test_is_field_channel() -> None:
    """Test channel detection across all scenarios."""
    # Basic detection
    result = _is_field_channel(Annotated[int, EphemeralValue])
    assert isinstance(result, EphemeralValue) and result.typ is int

    # Main fix: handles extraneous annotations
    result = _is_field_channel(Annotated[str, "metadata", EphemeralValue, "more"])
    assert isinstance(result, EphemeralValue) and result.typ is str

    # Complex types work
    union_type = Union[int, str]  # noqa: UP007
    result = _is_field_channel(Annotated[union_type, EphemeralValue])
    assert isinstance(result, EphemeralValue) and result.typ is union_type

    # Pre-instantiated channels
    instantiated = EphemeralValue(int)
    result = _is_field_channel(Annotated[int, instantiated])
    assert result is instantiated

    # Pre-instantiated channels with multiple annotations
    instantiated = EphemeralValue(int)
    result = _is_field_channel(Annotated[int, "metadata", instantiated, "more"])
    assert result is instantiated

    # No channel cases
    assert _is_field_channel(int) is None
    assert _is_field_channel(Annotated[int, "just_metadata"]) is None

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/test_tracing_interops.py
```py
import json
import sys
import time
from collections.abc import Callable
from typing import Any, TypeVar
from unittest.mock import MagicMock

import langsmith as ls
import pytest
from langchain_core.runnables import RunnableConfig
from langchain_core.tracers import LangChainTracer
from typing_extensions import TypedDict

from langgraph.graph import StateGraph

pytestmark = pytest.mark.anyio


def _get_mock_client(**kwargs: Any) -> ls.Client:
    mock_session = MagicMock()
    return ls.Client(session=mock_session, api_key="test", **kwargs)


def _get_calls(
    mock_client: Any,
    verbs: set[str] = {"POST"},
) -> list:
    return [
        c
        for c in mock_client.session.request.mock_calls
        if c.args and c.args[0] in verbs
    ]


T = TypeVar("T")


def wait_for(
    condition: Callable[[], tuple[T, bool]],
    max_sleep_time: int = 10,
    sleep_time: int = 3,
) -> T:
    """Wait for a condition to be true."""
    start_time = time.time()
    last_e = None
    while time.time() - start_time < max_sleep_time:
        try:
            res, cond = condition()
            if cond:
                return res
        except Exception as e:
            last_e = e
            time.sleep(sleep_time)
    total_time = time.time() - start_time
    if last_e is not None:
        raise last_e
    raise ValueError(f"Callable did not return within {total_time}")


@pytest.mark.skip("This test times out in CI")
async def test_nested_tracing():
    lt_py_311 = sys.version_info < (3, 11)
    mock_client = _get_mock_client()

    class State(TypedDict):
        value: str

    @ls.traceable
    async def some_traceable(content: State):
        return await child_graph.ainvoke(content)

    async def parent_node(state: State, config: RunnableConfig) -> State:
        if lt_py_311:
            result = await some_traceable(state, langsmith_extra={"config": config})
        else:
            result = await some_traceable(state)
        return {"value": f"parent_{result['value']}"}

    async def child_node(state: State) -> State:
        return {"value": f"child_{state['value']}"}

    child_builder = StateGraph(State)
    child_builder.add_node(child_node)
    child_builder.add_edge("__start__", "child_node")
    child_graph = child_builder.compile().with_config(run_name="child_graph")

    parent_builder = StateGraph(State)
    parent_builder.add_node(parent_node)
    parent_builder.add_edge("__start__", "parent_node")
    parent_graph = parent_builder.compile()

    tracer = LangChainTracer(client=mock_client)
    result = await parent_graph.ainvoke({"value": "input"}, {"callbacks": [tracer]})

    assert result == {"value": "parent_child_input"}

    def get_posts():
        post_calls = _get_calls(mock_client, verbs={"POST"})

        posts = [p for c in post_calls for p in json.loads(c.kwargs["data"])["post"]]
        names = [p.get("name") for p in posts]
        if "child_node" in names:
            return posts, True
        return None, False

    posts = wait_for(get_posts)
    # If the callbacks weren't propagated correctly, we'd
    # end up with broken dotted_orders
    parent_run = next(data for data in posts if data["name"] == "parent_node")
    child_run = next(data for data in posts if data["name"] == "child_graph")
    traceable_run = next(data for data in posts if data["name"] == "some_traceable")

    assert child_run["dotted_order"].startswith(traceable_run["dotted_order"])
    assert traceable_run["dotted_order"].startswith(parent_run["dotted_order"])

    assert child_run["parent_run_id"] == traceable_run["id"]
    assert traceable_run["parent_run_id"] == parent_run["id"]
    assert parent_run["trace_id"] == child_run["trace_id"] == traceable_run["trace_id"]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/test_type_checking.py
```py
from dataclasses import dataclass
from operator import add
from typing import Annotated, Any

import pytest
from langchain_core.runnables import RunnableConfig
from pydantic import BaseModel
from typing_extensions import TypedDict

from langgraph.graph import StateGraph
from langgraph.types import Command


def test_typed_dict_state() -> None:
    class TypedDictState(TypedDict):
        info: Annotated[list[str], add]

    graph_builder = StateGraph(TypedDictState)

    def valid(state: TypedDictState) -> Any: ...

    def valid_with_config(state: TypedDictState, config: RunnableConfig) -> Any: ...

    def invalid() -> Any: ...

    def invalid_node() -> Any: ...

    graph_builder.add_node("valid", valid)
    graph_builder.add_node("invalid", valid_with_config)
    graph_builder.add_node("invalid_node", invalid_node)  # type: ignore[call-overload]
    graph_builder.set_entry_point("valid")
    graph = graph_builder.compile()

    graph.invoke({"info": ["hello", "world"]})
    graph.invoke({"invalid": "lalala"})  # type: ignore[arg-type]


def test_dataclass_state() -> None:
    @dataclass
    class DataclassState:
        info: Annotated[list[str], add]

    def valid(state: DataclassState) -> Any: ...

    def valid_with_config(state: DataclassState, config: RunnableConfig) -> Any: ...

    def invalid() -> Any: ...

    graph_builder = StateGraph(DataclassState)
    graph_builder.add_node("valid", valid)
    graph_builder.add_node("invalid", valid_with_config)
    graph_builder.add_node("invalid_node", invalid)  # type: ignore[call-overload]

    graph_builder.set_entry_point("valid")
    graph = graph_builder.compile()

    graph.invoke(DataclassState(info=["hello", "world"]))
    graph.invoke({"invalid": 1})  # type: ignore[arg-type]
    graph.invoke({"info": ["hello", "world"]})  # type: ignore[arg-type]


def test_base_model_state() -> None:
    class PydanticState(BaseModel):
        info: Annotated[list[str], add]

    def valid(state: PydanticState) -> Any: ...

    def valid_with_config(state: PydanticState, config: RunnableConfig) -> Any: ...

    def invalid() -> Any: ...

    graph_builder = StateGraph(PydanticState)
    graph_builder.add_node("valid", valid)
    graph_builder.add_node("invalid", valid_with_config)
    graph_builder.add_node("invalid_node", invalid)  # type: ignore[call-overload]

    graph_builder.set_entry_point("valid")
    graph = graph_builder.compile()

    graph.invoke(PydanticState(info=["hello", "world"]))
    graph.invoke({"invalid": 1})  # type: ignore[arg-type]
    graph.invoke({"info": ["hello", "world"]})  # type: ignore[arg-type]


def test_plain_class_not_allowed() -> None:
    class NotAllowed:
        info: Annotated[list[str], add]

    StateGraph(NotAllowed)  # type: ignore[type-var]


def test_input_state_specified() -> None:
    class InputState(TypedDict):
        something: int

    class State(InputState):
        info: Annotated[list[str], add]

    def valid(state: State) -> Any: ...

    new_builder = StateGraph(State, input_schema=InputState)
    new_builder.add_node("valid", valid)
    new_builder.set_entry_point("valid")
    new_graph = new_builder.compile()

    new_graph.invoke({"something": 1})
    new_graph.invoke({"something": 2, "info": ["hello", "world"]})  # type: ignore[arg-type]


@pytest.mark.skip("Purely for type checking")
def test_invoke_with_all_valid_types() -> None:
    class State(TypedDict):
        a: int

    def a(state: State) -> Any: ...

    graph = StateGraph(State).add_node("a", a).set_entry_point("a").compile()
    graph.invoke({"a": 1})
    graph.invoke(None)
    graph.invoke(Command())


def test_add_node_with_explicit_input_schema() -> None:
    class A(TypedDict):
        a1: int
        a2: str

    class B(TypedDict):
        b1: int
        b2: str

    class ANarrow(TypedDict):
        a1: int

    class BNarrow(TypedDict):
        b1: int

    class State(A, B): ...

    def a(state: A) -> Any: ...

    def b(state: B) -> Any: ...

    workflow = StateGraph(State)
    # input schema matches typed schemas
    workflow.add_node("a", a, input_schema=A)
    workflow.add_node("b", b, input_schema=B)

    # input schema does not match typed schemas
    workflow.add_node("a_wrong", a, input_schema=B)  # type: ignore[arg-type]
    workflow.add_node("b_wrong", b, input_schema=A)  # type: ignore[arg-type]

    # input schema is more broad than the typed schemas, which is allowed
    # by the principles of contravariance
    workflow.add_node("a_inclusive", a, input_schema=State)
    workflow.add_node("b_inclusive", b, input_schema=State)

    # input schema is more narrow than the typed schemas, which is not allowed
    # because it violates the principles of contravariance
    workflow.add_node("a_narrow", a, input_schema=ANarrow)  # type: ignore[arg-type]
    workflow.add_node("b_narrow", b, input_schema=BNarrow)  # type: ignore[arg-type]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/test_utils.py
```py
import functools
import sys
import uuid
from collections.abc import Callable
from typing import (
    Annotated,
    Any,
    ForwardRef,
    Literal,
    Optional,
    TypeVar,
    Union,
)
from unittest.mock import patch

import langsmith
import pytest
from typing_extensions import NotRequired, Required, TypedDict

from langgraph._internal._config import _is_not_empty, ensure_config
from langgraph._internal._fields import (
    _is_optional_type,
    get_enhanced_type_hints,
    get_field_default,
)
from langgraph._internal._runnable import is_async_callable, is_async_generator
from langgraph.constants import END
from langgraph.graph import StateGraph
from langgraph.graph.state import CompiledStateGraph

# ruff: noqa: UP045, UP007

pytestmark = pytest.mark.anyio


def test_is_async() -> None:
    async def func() -> None:
        pass

    assert is_async_callable(func)
    wrapped_func = functools.wraps(func)(func)
    assert is_async_callable(wrapped_func)

    def sync_func() -> None:
        pass

    assert not is_async_callable(sync_func)
    wrapped_sync_func = functools.wraps(sync_func)(sync_func)
    assert not is_async_callable(wrapped_sync_func)

    class AsyncFuncCallable:
        async def __call__(self) -> None:
            pass

    runnable = AsyncFuncCallable()
    assert is_async_callable(runnable)
    wrapped_runnable = functools.wraps(runnable)(runnable)
    assert is_async_callable(wrapped_runnable)

    class SyncFuncCallable:
        def __call__(self) -> None:
            pass

    sync_runnable = SyncFuncCallable()
    assert not is_async_callable(sync_runnable)
    wrapped_sync_runnable = functools.wraps(sync_runnable)(sync_runnable)
    assert not is_async_callable(wrapped_sync_runnable)


def test_is_generator() -> None:
    async def gen():
        yield

    assert is_async_generator(gen)

    wrapped_gen = functools.wraps(gen)(gen)
    assert is_async_generator(wrapped_gen)

    def sync_gen():
        yield

    assert not is_async_generator(sync_gen)
    wrapped_sync_gen = functools.wraps(sync_gen)(sync_gen)
    assert not is_async_generator(wrapped_sync_gen)

    class AsyncGenCallable:
        async def __call__(self):
            yield

    runnable = AsyncGenCallable()
    assert is_async_generator(runnable)
    wrapped_runnable = functools.wraps(runnable)(runnable)
    assert is_async_generator(wrapped_runnable)

    class SyncGenCallable:
        def __call__(self):
            yield

    sync_runnable = SyncGenCallable()
    assert not is_async_generator(sync_runnable)
    wrapped_sync_runnable = functools.wraps(sync_runnable)(sync_runnable)
    assert not is_async_generator(wrapped_sync_runnable)


@pytest.fixture
def rt_graph() -> CompiledStateGraph:
    class State(TypedDict):
        foo: int
        node_run_id: int

    def node(_: State):
        from langsmith import get_current_run_tree  # type: ignore

        return {"node_run_id": get_current_run_tree().id}  # type: ignore

    graph = StateGraph(State)
    graph.add_node(node)
    graph.set_entry_point("node")
    graph.add_edge("node", END)
    return graph.compile()


def test_runnable_callable_tracing_nested(rt_graph: CompiledStateGraph) -> None:
    with patch("langsmith.client.Client", spec=langsmith.Client) as mock_client:
        with patch("langchain_core.tracers.langchain.get_client") as mock_get_client:
            mock_get_client.return_value = mock_client
            with langsmith.tracing_context(enabled=True):
                res = rt_graph.invoke({"foo": 1})
    assert isinstance(res["node_run_id"], uuid.UUID)


@pytest.mark.skipif(
    sys.version_info < (3, 11),
    reason="Python 3.11+ is required for async contextvars support",
)
async def test_runnable_callable_tracing_nested_async(
    rt_graph: CompiledStateGraph,
) -> None:
    with patch("langsmith.client.Client", spec=langsmith.Client) as mock_client:
        with patch("langchain_core.tracers.langchain.get_client") as mock_get_client:
            mock_get_client.return_value = mock_client
            with langsmith.tracing_context(enabled=True):
                res = await rt_graph.ainvoke({"foo": 1})
    assert isinstance(res["node_run_id"], uuid.UUID)


def test_is_optional_type():
    assert _is_optional_type(None)
    assert not _is_optional_type(type(None))
    assert _is_optional_type(Optional[list])
    assert not _is_optional_type(int)
    assert _is_optional_type(Optional[Literal[1, 2, 3]])
    assert not _is_optional_type(Literal[1, 2, 3])
    assert _is_optional_type(Optional[list[int]])
    assert _is_optional_type(Optional[dict[str, int]])
    assert not _is_optional_type(list[int | None])
    assert _is_optional_type(Union[str | None, int | None])
    assert _is_optional_type(Union[str | None | int | None, float | None | dict | None])
    assert not _is_optional_type(Union[str | int, float | dict])

    assert _is_optional_type(Union[int, None])
    assert _is_optional_type(Union[str, None, int])
    assert _is_optional_type(Union[None, str, int])
    assert not _is_optional_type(Union[int, str])

    assert not _is_optional_type(Any)  # Do we actually want this?
    assert _is_optional_type(Optional[Any])

    class MyClass:
        pass

    assert _is_optional_type(Optional[MyClass])
    assert not _is_optional_type(MyClass)
    assert _is_optional_type(Optional[ForwardRef("MyClass")])
    assert not _is_optional_type(ForwardRef("MyClass"))

    assert _is_optional_type(Optional[list[int] | dict[str, int | None]])
    assert not _is_optional_type(Union[list[int], dict[str, int | None]])

    assert _is_optional_type(Optional[Callable[[int], str]])
    assert not _is_optional_type(Callable[[int], str | None])

    T = TypeVar("T")
    assert _is_optional_type(Optional[T])
    assert not _is_optional_type(T)

    U = TypeVar("U", bound=T | None)  # type: ignore
    assert _is_optional_type(U)


def test_is_required():
    class MyBaseTypedDict(TypedDict):
        val_1: Required[str | None]
        val_2: Required[str]
        val_3: NotRequired[str]
        val_4: NotRequired[str | None]
        val_5: Annotated[NotRequired[int], "foo"]
        val_6: NotRequired[Annotated[int, "foo"]]
        val_7: Annotated[Required[int], "foo"]
        val_8: Required[Annotated[int, "foo"]]
        val_9: str | None
        val_10: str

    annos = MyBaseTypedDict.__annotations__
    assert get_field_default("val_1", annos["val_1"], MyBaseTypedDict) == ...
    assert get_field_default("val_2", annos["val_2"], MyBaseTypedDict) == ...
    assert get_field_default("val_3", annos["val_3"], MyBaseTypedDict) is None
    assert get_field_default("val_4", annos["val_4"], MyBaseTypedDict) is None
    # See https://peps.python.org/pep-0655/#interaction-with-annotated
    assert get_field_default("val_5", annos["val_5"], MyBaseTypedDict) is None
    assert get_field_default("val_6", annos["val_6"], MyBaseTypedDict) is None
    assert get_field_default("val_7", annos["val_7"], MyBaseTypedDict) == ...
    assert get_field_default("val_8", annos["val_8"], MyBaseTypedDict) == ...
    assert get_field_default("val_9", annos["val_9"], MyBaseTypedDict) is None
    assert get_field_default("val_10", annos["val_10"], MyBaseTypedDict) == ...

    class MyChildDict(MyBaseTypedDict):
        val_11: int
        val_11b: int | None
        val_11c: int | None | str

    class MyGrandChildDict(MyChildDict, total=False):
        val_12: int
        val_13: Required[str]

    cannos = MyChildDict.__annotations__
    gcannos = MyGrandChildDict.__annotations__
    assert get_field_default("val_11", cannos["val_11"], MyChildDict) == ...
    assert get_field_default("val_11b", cannos["val_11b"], MyChildDict) is None
    assert get_field_default("val_11c", cannos["val_11c"], MyChildDict) is None
    assert get_field_default("val_12", gcannos["val_12"], MyGrandChildDict) is None
    assert get_field_default("val_9", gcannos["val_9"], MyGrandChildDict) is None
    assert get_field_default("val_13", gcannos["val_13"], MyGrandChildDict) == ...


def test_enhanced_type_hints() -> None:
    from dataclasses import dataclass
    from typing import Annotated

    from pydantic import BaseModel, Field

    class MyTypedDict(TypedDict):
        val_1: str
        val_2: int = 42
        val_3: str = "default"

    hints = list(get_enhanced_type_hints(MyTypedDict))
    assert len(hints) == 3
    assert hints[0] == ("val_1", str, None, None)
    assert hints[1] == ("val_2", int, 42, None)
    assert hints[2] == ("val_3", str, "default", None)

    @dataclass
    class MyDataclass:
        val_1: str
        val_2: int = 42
        val_3: str = "default"

    hints = list(get_enhanced_type_hints(MyDataclass))
    assert len(hints) == 3
    assert hints[0] == ("val_1", str, None, None)
    assert hints[1] == ("val_2", int, 42, None)
    assert hints[2] == ("val_3", str, "default", None)

    class MyPydanticModel(BaseModel):
        val_1: str
        val_2: int = 42
        val_3: str = Field(default="default", description="A description")

    hints = list(get_enhanced_type_hints(MyPydanticModel))
    assert len(hints) == 3
    assert hints[0] == ("val_1", str, None, None)
    assert hints[1] == ("val_2", int, 42, None)
    assert hints[2] == ("val_3", str, "default", "A description")

    class MyPydanticModelWithAnnotated(BaseModel):
        val_1: Annotated[str, Field(description="A description")]
        val_2: Annotated[int, Field(default=42)]
        val_3: Annotated[
            str, Field(default="default", description="Another description")
        ]

    hints = list(get_enhanced_type_hints(MyPydanticModelWithAnnotated))
    assert len(hints) == 3
    assert hints[0] == ("val_1", str, None, "A description")
    assert hints[1] == ("val_2", int, 42, None)
    assert hints[2] == ("val_3", str, "default", "Another description")


def test_is_not_empty() -> None:
    assert _is_not_empty("foo")
    assert _is_not_empty("")
    assert _is_not_empty(1)
    assert _is_not_empty(0)
    assert not _is_not_empty(None)
    assert not _is_not_empty([])
    assert not _is_not_empty(())
    assert not _is_not_empty({})


def test_configurable_metadata():
    config = {
        "configurable": {
            "a-key": "foo",
            "somesecretval": "bar",
            "sometoken": "thetoken",
            "__dontinclude": "bar",
            "includeme": "hi",
            "andme": 42,
            "nested": {"foo": "bar"},
            "nooverride": -2,
        },
        "metadata": {"nooverride": 18},
    }
    expected = {"includeme", "andme", "nooverride"}
    merged = ensure_config(config)
    metadata = merged["metadata"]
    assert metadata.keys() == expected
    assert metadata["nooverride"] == 18

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/__snapshots__/test_large_cases.ambr
```ambr
# serializer version: 1
# name: test_conditional_state_graph[memory]
  '{"$defs": {"AgentAction": {"description": "Represents a request to execute an action by an agent.\\n\\nThe action consists of the name of the tool to execute and the input to pass\\nto the tool. The log is used to pass along extra information about the action.", "properties": {"tool": {"title": "Tool", "type": "string"}, "tool_input": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}], "title": "Tool Input"}, "log": {"title": "Log", "type": "string"}, "type": {"const": "AgentAction", "default": "AgentAction", "title": "Type", "type": "string"}}, "required": ["tool", "tool_input", "log"], "title": "AgentAction", "type": "object"}, "AgentFinish": {"description": "Final return value of an ActionAgent.\\n\\nAgents return an AgentFinish when they have reached a stopping condition.", "properties": {"return_values": {"additionalProperties": true, "title": "Return Values", "type": "object"}, "log": {"title": "Log", "type": "string"}, "type": {"const": "AgentFinish", "default": "AgentFinish", "title": "Type", "type": "string"}}, "required": ["return_values", "log"], "title": "AgentFinish", "type": "object"}}, "properties": {"input": {"title": "Input", "type": "string"}, "agent_outcome": {"anyOf": [{"$ref": "#/$defs/AgentAction"}, {"$ref": "#/$defs/AgentFinish"}, {"type": "null"}], "title": "Agent Outcome"}, "intermediate_steps": {"items": {"maxItems": 2, "minItems": 2, "prefixItems": [{"$ref": "#/$defs/AgentAction"}, {"type": "string"}], "type": "array"}, "title": "Intermediate Steps", "type": "array"}}, "title": "AgentState", "type": "object"}'
# ---
# name: test_conditional_state_graph[memory].1
  '{"$defs": {"AgentAction": {"description": "Represents a request to execute an action by an agent.\\n\\nThe action consists of the name of the tool to execute and the input to pass\\nto the tool. The log is used to pass along extra information about the action.", "properties": {"tool": {"title": "Tool", "type": "string"}, "tool_input": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}], "title": "Tool Input"}, "log": {"title": "Log", "type": "string"}, "type": {"const": "AgentAction", "default": "AgentAction", "title": "Type", "type": "string"}}, "required": ["tool", "tool_input", "log"], "title": "AgentAction", "type": "object"}, "AgentFinish": {"description": "Final return value of an ActionAgent.\\n\\nAgents return an AgentFinish when they have reached a stopping condition.", "properties": {"return_values": {"additionalProperties": true, "title": "Return Values", "type": "object"}, "log": {"title": "Log", "type": "string"}, "type": {"const": "AgentFinish", "default": "AgentFinish", "title": "Type", "type": "string"}}, "required": ["return_values", "log"], "title": "AgentFinish", "type": "object"}}, "properties": {"input": {"title": "Input", "type": "string"}, "agent_outcome": {"anyOf": [{"$ref": "#/$defs/AgentAction"}, {"$ref": "#/$defs/AgentFinish"}, {"type": "null"}], "title": "Agent Outcome"}, "intermediate_steps": {"items": {"maxItems": 2, "minItems": 2, "prefixItems": [{"$ref": "#/$defs/AgentAction"}, {"type": "string"}], "type": "array"}, "title": "Intermediate Steps", "type": "array"}}, "title": "AgentState", "type": "object"}'
# ---
# name: test_conditional_state_graph[memory].2
  '''
  {
    "nodes": [
      {
        "id": "__start__",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "__start__"
        }
      },
      {
        "id": "agent",
        "type": "runnable",
        "data": {
          "id": [
            "langchain",
            "schema",
            "runnable",
            "RunnableSequence"
          ],
          "name": "agent"
        }
      },
      {
        "id": "tools",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "tools"
        }
      },
      {
        "id": "__end__"
      }
    ],
    "edges": [
      {
        "source": "__start__",
        "target": "agent"
      },
      {
        "source": "agent",
        "target": "__end__",
        "data": "exit",
        "conditional": true
      },
      {
        "source": "agent",
        "target": "tools",
        "data": "continue",
        "conditional": true
      },
      {
        "source": "tools",
        "target": "agent"
      }
    ]
  }
  '''
# ---
# name: test_conditional_state_graph[memory].3
  '''
  graph TD;
  	__start__ --> agent;
  	agent -. &nbsp;exit&nbsp; .-> __end__;
  	agent -. &nbsp;continue&nbsp; .-> tools;
  	tools --> agent;
  
  '''
# ---
# name: test_message_graph[memory]
  '{"$defs": {"AIMessage": {"additionalProperties": true, "description": "Message from an AI.\\n\\nAn `AIMessage` is returned from a chat model as a response to a prompt.\\n\\nThis message represents the output of the model and consists of both\\nthe raw output as returned by the model and standardized fields\\n(e.g., tool calls, usage metadata) added by the LangChain framework.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "ai", "default": "ai", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}, "tool_calls": {"default": [], "items": {"$ref": "#/$defs/ToolCall"}, "title": "Tool Calls", "type": "array"}, "invalid_tool_calls": {"default": [], "items": {"$ref": "#/$defs/InvalidToolCall"}, "title": "Invalid Tool Calls", "type": "array"}, "usage_metadata": {"anyOf": [{"$ref": "#/$defs/UsageMetadata"}, {"type": "null"}], "default": null}}, "required": ["content"], "title": "AIMessage", "type": "object"}, "AIMessageChunk": {"additionalProperties": true, "description": "Message chunk from an AI (yielded when streaming).", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "AIMessageChunk", "default": "AIMessageChunk", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}, "tool_calls": {"default": [], "items": {"$ref": "#/$defs/ToolCall"}, "title": "Tool Calls", "type": "array"}, "invalid_tool_calls": {"default": [], "items": {"$ref": "#/$defs/InvalidToolCall"}, "title": "Invalid Tool Calls", "type": "array"}, "usage_metadata": {"anyOf": [{"$ref": "#/$defs/UsageMetadata"}, {"type": "null"}], "default": null}, "tool_call_chunks": {"default": [], "items": {"$ref": "#/$defs/ToolCallChunk"}, "title": "Tool Call Chunks", "type": "array"}, "chunk_position": {"anyOf": [{"const": "last", "type": "string"}, {"type": "null"}], "default": null, "title": "Chunk Position"}}, "required": ["content"], "title": "AIMessageChunk", "type": "object"}, "ChatMessage": {"additionalProperties": true, "description": "Message that can be assigned an arbitrary speaker (i.e. role).", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "chat", "default": "chat", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}, "role": {"title": "Role", "type": "string"}}, "required": ["content", "role"], "title": "ChatMessage", "type": "object"}, "ChatMessageChunk": {"additionalProperties": true, "description": "Chat Message chunk.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "ChatMessageChunk", "default": "ChatMessageChunk", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}, "role": {"title": "Role", "type": "string"}}, "required": ["content", "role"], "title": "ChatMessageChunk", "type": "object"}, "FunctionMessage": {"additionalProperties": true, "description": "Message for passing the result of executing a tool back to a model.\\n\\n`FunctionMessage` are an older version of the `ToolMessage` schema, and\\ndo not contain the `tool_call_id` field.\\n\\nThe `tool_call_id` field is used to associate the tool call request with the\\ntool call response. Useful in situations where a chat model is able\\nto request multiple tool calls in parallel.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "function", "default": "function", "title": "Type", "type": "string"}, "name": {"title": "Name", "type": "string"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}}, "required": ["content", "name"], "title": "FunctionMessage", "type": "object"}, "FunctionMessageChunk": {"additionalProperties": true, "description": "Function Message chunk.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "FunctionMessageChunk", "default": "FunctionMessageChunk", "title": "Type", "type": "string"}, "name": {"title": "Name", "type": "string"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}}, "required": ["content", "name"], "title": "FunctionMessageChunk", "type": "object"}, "HumanMessage": {"additionalProperties": true, "description": "Message from the user.\\n\\nA `HumanMessage` is a message that is passed in from a user to the model.\\n\\nExample:\\n    ```python\\n    from langchain_core.messages import HumanMessage, SystemMessage\\n\\n    messages = [\\n        SystemMessage(content=\\"You are a helpful assistant! Your name is Bob.\\"),\\n        HumanMessage(content=\\"What is your name?\\"),\\n    ]\\n\\n    # Instantiate a chat model and invoke it with the messages\\n    model = ...\\n    print(model.invoke(messages))\\n    ```", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "human", "default": "human", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}}, "required": ["content"], "title": "HumanMessage", "type": "object"}, "HumanMessageChunk": {"additionalProperties": true, "description": "Human Message chunk.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "HumanMessageChunk", "default": "HumanMessageChunk", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}}, "required": ["content"], "title": "HumanMessageChunk", "type": "object"}, "InputTokenDetails": {"additionalProperties": true, "description": "Breakdown of input token counts.\\n\\nDoes *not* need to sum to full input token count. Does *not* need to have all keys.\\n\\nExample:\\n    ```python\\n    {\\n        \\"audio\\": 10,\\n        \\"cache_creation\\": 200,\\n        \\"cache_read\\": 100,\\n    }\\n    ```\\n\\n!!! version-added \\"Added in version 0.3.9\\"\\n\\nMay also hold extra provider-specific keys.", "properties": {"audio": {"title": "Audio", "type": "integer"}, "cache_creation": {"title": "Cache Creation", "type": "integer"}, "cache_read": {"title": "Cache Read", "type": "integer"}}, "title": "InputTokenDetails", "type": "object"}, "InvalidToolCall": {"additionalProperties": true, "description": "Allowance for errors made by LLM.\\n\\nHere we add an `error` key to surface errors made during generation\\n(e.g., invalid JSON arguments.)", "properties": {"type": {"const": "invalid_tool_call", "title": "Type", "type": "string"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "title": "Id"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "title": "Name"}, "args": {"anyOf": [{"type": "string"}, {"type": "null"}], "title": "Args"}, "error": {"anyOf": [{"type": "string"}, {"type": "null"}], "title": "Error"}, "index": {"anyOf": [{"type": "integer"}, {"type": "string"}], "title": "Index"}, "extras": {"additionalProperties": true, "title": "Extras", "type": "object"}}, "required": ["type", "id", "name", "args", "error"], "title": "InvalidToolCall", "type": "object"}, "OutputTokenDetails": {"additionalProperties": true, "description": "Breakdown of output token counts.\\n\\nDoes *not* need to sum to full output token count. Does *not* need to have all keys.\\n\\nExample:\\n    ```python\\n    {\\n        \\"audio\\": 10,\\n        \\"reasoning\\": 200,\\n    }\\n    ```\\n\\n!!! version-added \\"Added in version 0.3.9\\"", "properties": {"audio": {"title": "Audio", "type": "integer"}, "reasoning": {"title": "Reasoning", "type": "integer"}}, "title": "OutputTokenDetails", "type": "object"}, "SystemMessage": {"additionalProperties": true, "description": "Message for priming AI behavior.\\n\\nThe system message is usually passed in as the first of a sequence\\nof input messages.\\n\\nExample:\\n    ```python\\n    from langchain_core.messages import HumanMessage, SystemMessage\\n\\n    messages = [\\n        SystemMessage(content=\\"You are a helpful assistant! Your name is Bob.\\"),\\n        HumanMessage(content=\\"What is your name?\\"),\\n    ]\\n\\n    # Define a chat model and invoke it with the messages\\n    print(model.invoke(messages))\\n    ```", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "system", "default": "system", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}}, "required": ["content"], "title": "SystemMessage", "type": "object"}, "SystemMessageChunk": {"additionalProperties": true, "description": "System Message chunk.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "SystemMessageChunk", "default": "SystemMessageChunk", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}}, "required": ["content"], "title": "SystemMessageChunk", "type": "object"}, "ToolCall": {"additionalProperties": true, "description": "Represents an AI\'s request to call a tool.\\n\\nExample:\\n    ```python\\n    {\\"name\\": \\"foo\\", \\"args\\": {\\"a\\": 1}, \\"id\\": \\"123\\"}\\n    ```\\n\\n    This represents a request to call the tool named `\'foo\'` with arguments\\n    `{\\"a\\": 1}` and an identifier of `\'123\'`.", "properties": {"name": {"title": "Name", "type": "string"}, "args": {"additionalProperties": true, "title": "Args", "type": "object"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "title": "Id"}, "type": {"const": "tool_call", "title": "Type", "type": "string"}}, "required": ["name", "args", "id"], "title": "ToolCall", "type": "object"}, "ToolCallChunk": {"additionalProperties": true, "description": "A chunk of a tool call (yielded when streaming).\\n\\nWhen merging `ToolCallChunk`s (e.g., via `AIMessageChunk.__add__`),\\nall string attributes are concatenated. Chunks are only merged if their\\nvalues of `index` are equal and not None.\\n\\nExample:\\n```python\\nleft_chunks = [ToolCallChunk(name=\\"foo\\", args=\'{\\"a\\":\', index=0)]\\nright_chunks = [ToolCallChunk(name=None, args=\\"1}\\", index=0)]\\n\\n(\\n    AIMessageChunk(content=\\"\\", tool_call_chunks=left_chunks)\\n    + AIMessageChunk(content=\\"\\", tool_call_chunks=right_chunks)\\n).tool_call_chunks == [ToolCallChunk(name=\\"foo\\", args=\'{\\"a\\":1}\', index=0)]\\n```", "properties": {"name": {"anyOf": [{"type": "string"}, {"type": "null"}], "title": "Name"}, "args": {"anyOf": [{"type": "string"}, {"type": "null"}], "title": "Args"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "title": "Id"}, "index": {"anyOf": [{"type": "integer"}, {"type": "null"}], "title": "Index"}, "type": {"const": "tool_call_chunk", "title": "Type", "type": "string"}}, "required": ["name", "args", "id", "index"], "title": "ToolCallChunk", "type": "object"}, "ToolMessage": {"additionalProperties": true, "description": "Message for passing the result of executing a tool back to a model.\\n\\n`ToolMessage` objects contain the result of a tool invocation. Typically, the result\\nis encoded inside the `content` field.\\n\\nExample: A `ToolMessage` representing a result of `42` from a tool call with id\\n\\n```python\\nfrom langchain_core.messages import ToolMessage\\n\\nToolMessage(content=\\"42\\", tool_call_id=\\"call_Jja7J89XsjrOLA5r!MEOW!SL\\")\\n```\\n\\nExample: A `ToolMessage` where only part of the tool output is sent to the model\\nand the full output is passed in to artifact.\\n\\n```python\\nfrom langchain_core.messages import ToolMessage\\n\\ntool_output = {\\n    \\"stdout\\": \\"From the graph we can see that the correlation between \\"\\n    \\"x and y is ...\\",\\n    \\"stderr\\": None,\\n    \\"artifacts\\": {\\"type\\": \\"image\\", \\"base64_data\\": \\"/9j/4gIcSU...\\"},\\n}\\n\\nToolMessage(\\n    content=tool_output[\\"stdout\\"],\\n    artifact=tool_output,\\n    tool_call_id=\\"call_Jja7J89XsjrOLA5r!MEOW!SL\\",\\n)\\n```\\n\\nThe `tool_call_id` field is used to associate the tool call request with the\\ntool call response. Useful in situations where a chat model is able\\nto request multiple tool calls in parallel.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "tool", "default": "tool", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}, "tool_call_id": {"title": "Tool Call Id", "type": "string"}, "artifact": {"default": null, "title": "Artifact"}, "status": {"default": "success", "enum": ["success", "error"], "title": "Status", "type": "string"}}, "required": ["content", "tool_call_id"], "title": "ToolMessage", "type": "object"}, "ToolMessageChunk": {"additionalProperties": true, "description": "Tool Message chunk.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "ToolMessageChunk", "default": "ToolMessageChunk", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}, "tool_call_id": {"title": "Tool Call Id", "type": "string"}, "artifact": {"default": null, "title": "Artifact"}, "status": {"default": "success", "enum": ["success", "error"], "title": "Status", "type": "string"}}, "required": ["content", "tool_call_id"], "title": "ToolMessageChunk", "type": "object"}, "UsageMetadata": {"additionalProperties": true, "description": "Usage metadata for a message, such as token counts.\\n\\nThis is a standard representation of token usage that is consistent across models.\\n\\nExample:\\n    ```python\\n    {\\n        \\"input_tokens\\": 350,\\n        \\"output_tokens\\": 240,\\n        \\"total_tokens\\": 590,\\n        \\"input_token_details\\": {\\n            \\"audio\\": 10,\\n            \\"cache_creation\\": 200,\\n            \\"cache_read\\": 100,\\n        },\\n        \\"output_token_details\\": {\\n            \\"audio\\": 10,\\n            \\"reasoning\\": 200,\\n        },\\n    }\\n    ```\\n\\n!!! warning \\"Behavior changed in 0.3.9\\"\\n    Added `input_token_details` and `output_token_details`.", "properties": {"input_tokens": {"title": "Input Tokens", "type": "integer"}, "output_tokens": {"title": "Output Tokens", "type": "integer"}, "total_tokens": {"title": "Total Tokens", "type": "integer"}, "input_token_details": {"$ref": "#/$defs/InputTokenDetails"}, "output_token_details": {"$ref": "#/$defs/OutputTokenDetails"}}, "required": ["input_tokens", "output_tokens", "total_tokens"], "title": "UsageMetadata", "type": "object"}}, "default": null, "items": {"oneOf": [{"$ref": "#/$defs/AIMessage"}, {"$ref": "#/$defs/HumanMessage"}, {"$ref": "#/$defs/ChatMessage"}, {"$ref": "#/$defs/SystemMessage"}, {"$ref": "#/$defs/FunctionMessage"}, {"$ref": "#/$defs/ToolMessage"}, {"$ref": "#/$defs/AIMessageChunk"}, {"$ref": "#/$defs/HumanMessageChunk"}, {"$ref": "#/$defs/ChatMessageChunk"}, {"$ref": "#/$defs/SystemMessageChunk"}, {"$ref": "#/$defs/FunctionMessageChunk"}, {"$ref": "#/$defs/ToolMessageChunk"}]}, "title": "LangGraphInput", "type": "array"}'
# ---
# name: test_message_graph[memory].1
  '{"$defs": {"AIMessage": {"additionalProperties": true, "description": "Message from an AI.\\n\\nAn `AIMessage` is returned from a chat model as a response to a prompt.\\n\\nThis message represents the output of the model and consists of both\\nthe raw output as returned by the model and standardized fields\\n(e.g., tool calls, usage metadata) added by the LangChain framework.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "ai", "default": "ai", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}, "tool_calls": {"default": [], "items": {"$ref": "#/$defs/ToolCall"}, "title": "Tool Calls", "type": "array"}, "invalid_tool_calls": {"default": [], "items": {"$ref": "#/$defs/InvalidToolCall"}, "title": "Invalid Tool Calls", "type": "array"}, "usage_metadata": {"anyOf": [{"$ref": "#/$defs/UsageMetadata"}, {"type": "null"}], "default": null}}, "required": ["content"], "title": "AIMessage", "type": "object"}, "AIMessageChunk": {"additionalProperties": true, "description": "Message chunk from an AI (yielded when streaming).", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "AIMessageChunk", "default": "AIMessageChunk", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}, "tool_calls": {"default": [], "items": {"$ref": "#/$defs/ToolCall"}, "title": "Tool Calls", "type": "array"}, "invalid_tool_calls": {"default": [], "items": {"$ref": "#/$defs/InvalidToolCall"}, "title": "Invalid Tool Calls", "type": "array"}, "usage_metadata": {"anyOf": [{"$ref": "#/$defs/UsageMetadata"}, {"type": "null"}], "default": null}, "tool_call_chunks": {"default": [], "items": {"$ref": "#/$defs/ToolCallChunk"}, "title": "Tool Call Chunks", "type": "array"}, "chunk_position": {"anyOf": [{"const": "last", "type": "string"}, {"type": "null"}], "default": null, "title": "Chunk Position"}}, "required": ["content"], "title": "AIMessageChunk", "type": "object"}, "ChatMessage": {"additionalProperties": true, "description": "Message that can be assigned an arbitrary speaker (i.e. role).", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "chat", "default": "chat", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}, "role": {"title": "Role", "type": "string"}}, "required": ["content", "role"], "title": "ChatMessage", "type": "object"}, "ChatMessageChunk": {"additionalProperties": true, "description": "Chat Message chunk.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "ChatMessageChunk", "default": "ChatMessageChunk", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}, "role": {"title": "Role", "type": "string"}}, "required": ["content", "role"], "title": "ChatMessageChunk", "type": "object"}, "FunctionMessage": {"additionalProperties": true, "description": "Message for passing the result of executing a tool back to a model.\\n\\n`FunctionMessage` are an older version of the `ToolMessage` schema, and\\ndo not contain the `tool_call_id` field.\\n\\nThe `tool_call_id` field is used to associate the tool call request with the\\ntool call response. Useful in situations where a chat model is able\\nto request multiple tool calls in parallel.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "function", "default": "function", "title": "Type", "type": "string"}, "name": {"title": "Name", "type": "string"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}}, "required": ["content", "name"], "title": "FunctionMessage", "type": "object"}, "FunctionMessageChunk": {"additionalProperties": true, "description": "Function Message chunk.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "FunctionMessageChunk", "default": "FunctionMessageChunk", "title": "Type", "type": "string"}, "name": {"title": "Name", "type": "string"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}}, "required": ["content", "name"], "title": "FunctionMessageChunk", "type": "object"}, "HumanMessage": {"additionalProperties": true, "description": "Message from the user.\\n\\nA `HumanMessage` is a message that is passed in from a user to the model.\\n\\nExample:\\n    ```python\\n    from langchain_core.messages import HumanMessage, SystemMessage\\n\\n    messages = [\\n        SystemMessage(content=\\"You are a helpful assistant! Your name is Bob.\\"),\\n        HumanMessage(content=\\"What is your name?\\"),\\n    ]\\n\\n    # Instantiate a chat model and invoke it with the messages\\n    model = ...\\n    print(model.invoke(messages))\\n    ```", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "human", "default": "human", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}}, "required": ["content"], "title": "HumanMessage", "type": "object"}, "HumanMessageChunk": {"additionalProperties": true, "description": "Human Message chunk.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "HumanMessageChunk", "default": "HumanMessageChunk", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}}, "required": ["content"], "title": "HumanMessageChunk", "type": "object"}, "InputTokenDetails": {"additionalProperties": true, "description": "Breakdown of input token counts.\\n\\nDoes *not* need to sum to full input token count. Does *not* need to have all keys.\\n\\nExample:\\n    ```python\\n    {\\n        \\"audio\\": 10,\\n        \\"cache_creation\\": 200,\\n        \\"cache_read\\": 100,\\n    }\\n    ```\\n\\n!!! version-added \\"Added in version 0.3.9\\"\\n\\nMay also hold extra provider-specific keys.", "properties": {"audio": {"title": "Audio", "type": "integer"}, "cache_creation": {"title": "Cache Creation", "type": "integer"}, "cache_read": {"title": "Cache Read", "type": "integer"}}, "title": "InputTokenDetails", "type": "object"}, "InvalidToolCall": {"additionalProperties": true, "description": "Allowance for errors made by LLM.\\n\\nHere we add an `error` key to surface errors made during generation\\n(e.g., invalid JSON arguments.)", "properties": {"type": {"const": "invalid_tool_call", "title": "Type", "type": "string"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "title": "Id"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "title": "Name"}, "args": {"anyOf": [{"type": "string"}, {"type": "null"}], "title": "Args"}, "error": {"anyOf": [{"type": "string"}, {"type": "null"}], "title": "Error"}, "index": {"anyOf": [{"type": "integer"}, {"type": "string"}], "title": "Index"}, "extras": {"additionalProperties": true, "title": "Extras", "type": "object"}}, "required": ["type", "id", "name", "args", "error"], "title": "InvalidToolCall", "type": "object"}, "OutputTokenDetails": {"additionalProperties": true, "description": "Breakdown of output token counts.\\n\\nDoes *not* need to sum to full output token count. Does *not* need to have all keys.\\n\\nExample:\\n    ```python\\n    {\\n        \\"audio\\": 10,\\n        \\"reasoning\\": 200,\\n    }\\n    ```\\n\\n!!! version-added \\"Added in version 0.3.9\\"", "properties": {"audio": {"title": "Audio", "type": "integer"}, "reasoning": {"title": "Reasoning", "type": "integer"}}, "title": "OutputTokenDetails", "type": "object"}, "SystemMessage": {"additionalProperties": true, "description": "Message for priming AI behavior.\\n\\nThe system message is usually passed in as the first of a sequence\\nof input messages.\\n\\nExample:\\n    ```python\\n    from langchain_core.messages import HumanMessage, SystemMessage\\n\\n    messages = [\\n        SystemMessage(content=\\"You are a helpful assistant! Your name is Bob.\\"),\\n        HumanMessage(content=\\"What is your name?\\"),\\n    ]\\n\\n    # Define a chat model and invoke it with the messages\\n    print(model.invoke(messages))\\n    ```", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "system", "default": "system", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}}, "required": ["content"], "title": "SystemMessage", "type": "object"}, "SystemMessageChunk": {"additionalProperties": true, "description": "System Message chunk.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "SystemMessageChunk", "default": "SystemMessageChunk", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}}, "required": ["content"], "title": "SystemMessageChunk", "type": "object"}, "ToolCall": {"additionalProperties": true, "description": "Represents an AI\'s request to call a tool.\\n\\nExample:\\n    ```python\\n    {\\"name\\": \\"foo\\", \\"args\\": {\\"a\\": 1}, \\"id\\": \\"123\\"}\\n    ```\\n\\n    This represents a request to call the tool named `\'foo\'` with arguments\\n    `{\\"a\\": 1}` and an identifier of `\'123\'`.", "properties": {"name": {"title": "Name", "type": "string"}, "args": {"additionalProperties": true, "title": "Args", "type": "object"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "title": "Id"}, "type": {"const": "tool_call", "title": "Type", "type": "string"}}, "required": ["name", "args", "id"], "title": "ToolCall", "type": "object"}, "ToolCallChunk": {"additionalProperties": true, "description": "A chunk of a tool call (yielded when streaming).\\n\\nWhen merging `ToolCallChunk`s (e.g., via `AIMessageChunk.__add__`),\\nall string attributes are concatenated. Chunks are only merged if their\\nvalues of `index` are equal and not None.\\n\\nExample:\\n```python\\nleft_chunks = [ToolCallChunk(name=\\"foo\\", args=\'{\\"a\\":\', index=0)]\\nright_chunks = [ToolCallChunk(name=None, args=\\"1}\\", index=0)]\\n\\n(\\n    AIMessageChunk(content=\\"\\", tool_call_chunks=left_chunks)\\n    + AIMessageChunk(content=\\"\\", tool_call_chunks=right_chunks)\\n).tool_call_chunks == [ToolCallChunk(name=\\"foo\\", args=\'{\\"a\\":1}\', index=0)]\\n```", "properties": {"name": {"anyOf": [{"type": "string"}, {"type": "null"}], "title": "Name"}, "args": {"anyOf": [{"type": "string"}, {"type": "null"}], "title": "Args"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "title": "Id"}, "index": {"anyOf": [{"type": "integer"}, {"type": "null"}], "title": "Index"}, "type": {"const": "tool_call_chunk", "title": "Type", "type": "string"}}, "required": ["name", "args", "id", "index"], "title": "ToolCallChunk", "type": "object"}, "ToolMessage": {"additionalProperties": true, "description": "Message for passing the result of executing a tool back to a model.\\n\\n`ToolMessage` objects contain the result of a tool invocation. Typically, the result\\nis encoded inside the `content` field.\\n\\nExample: A `ToolMessage` representing a result of `42` from a tool call with id\\n\\n```python\\nfrom langchain_core.messages import ToolMessage\\n\\nToolMessage(content=\\"42\\", tool_call_id=\\"call_Jja7J89XsjrOLA5r!MEOW!SL\\")\\n```\\n\\nExample: A `ToolMessage` where only part of the tool output is sent to the model\\nand the full output is passed in to artifact.\\n\\n```python\\nfrom langchain_core.messages import ToolMessage\\n\\ntool_output = {\\n    \\"stdout\\": \\"From the graph we can see that the correlation between \\"\\n    \\"x and y is ...\\",\\n    \\"stderr\\": None,\\n    \\"artifacts\\": {\\"type\\": \\"image\\", \\"base64_data\\": \\"/9j/4gIcSU...\\"},\\n}\\n\\nToolMessage(\\n    content=tool_output[\\"stdout\\"],\\n    artifact=tool_output,\\n    tool_call_id=\\"call_Jja7J89XsjrOLA5r!MEOW!SL\\",\\n)\\n```\\n\\nThe `tool_call_id` field is used to associate the tool call request with the\\ntool call response. Useful in situations where a chat model is able\\nto request multiple tool calls in parallel.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "tool", "default": "tool", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}, "tool_call_id": {"title": "Tool Call Id", "type": "string"}, "artifact": {"default": null, "title": "Artifact"}, "status": {"default": "success", "enum": ["success", "error"], "title": "Status", "type": "string"}}, "required": ["content", "tool_call_id"], "title": "ToolMessage", "type": "object"}, "ToolMessageChunk": {"additionalProperties": true, "description": "Tool Message chunk.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"const": "ToolMessageChunk", "default": "ToolMessageChunk", "title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}, "tool_call_id": {"title": "Tool Call Id", "type": "string"}, "artifact": {"default": null, "title": "Artifact"}, "status": {"default": "success", "enum": ["success", "error"], "title": "Status", "type": "string"}}, "required": ["content", "tool_call_id"], "title": "ToolMessageChunk", "type": "object"}, "UsageMetadata": {"additionalProperties": true, "description": "Usage metadata for a message, such as token counts.\\n\\nThis is a standard representation of token usage that is consistent across models.\\n\\nExample:\\n    ```python\\n    {\\n        \\"input_tokens\\": 350,\\n        \\"output_tokens\\": 240,\\n        \\"total_tokens\\": 590,\\n        \\"input_token_details\\": {\\n            \\"audio\\": 10,\\n            \\"cache_creation\\": 200,\\n            \\"cache_read\\": 100,\\n        },\\n        \\"output_token_details\\": {\\n            \\"audio\\": 10,\\n            \\"reasoning\\": 200,\\n        },\\n    }\\n    ```\\n\\n!!! warning \\"Behavior changed in 0.3.9\\"\\n    Added `input_token_details` and `output_token_details`.", "properties": {"input_tokens": {"title": "Input Tokens", "type": "integer"}, "output_tokens": {"title": "Output Tokens", "type": "integer"}, "total_tokens": {"title": "Total Tokens", "type": "integer"}, "input_token_details": {"$ref": "#/$defs/InputTokenDetails"}, "output_token_details": {"$ref": "#/$defs/OutputTokenDetails"}}, "required": ["input_tokens", "output_tokens", "total_tokens"], "title": "UsageMetadata", "type": "object"}}, "default": null, "items": {"oneOf": [{"$ref": "#/$defs/AIMessage"}, {"$ref": "#/$defs/HumanMessage"}, {"$ref": "#/$defs/ChatMessage"}, {"$ref": "#/$defs/SystemMessage"}, {"$ref": "#/$defs/FunctionMessage"}, {"$ref": "#/$defs/ToolMessage"}, {"$ref": "#/$defs/AIMessageChunk"}, {"$ref": "#/$defs/HumanMessageChunk"}, {"$ref": "#/$defs/ChatMessageChunk"}, {"$ref": "#/$defs/SystemMessageChunk"}, {"$ref": "#/$defs/FunctionMessageChunk"}, {"$ref": "#/$defs/ToolMessageChunk"}]}, "title": "LangGraphOutput", "type": "array"}'
# ---
# name: test_message_graph[memory].2
  '''
  {
    "nodes": [
      {
        "id": "__start__",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "__start__"
        }
      },
      {
        "id": "agent",
        "type": "runnable",
        "data": {
          "id": [
            "tests",
            "test_large_cases",
            "FakeFunctionChatModel"
          ],
          "name": "agent"
        }
      },
      {
        "id": "tools",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "prebuilt",
            "tool_node",
            "ToolNode"
          ],
          "name": "tools"
        }
      },
      {
        "id": "__end__"
      }
    ],
    "edges": [
      {
        "source": "__start__",
        "target": "agent"
      },
      {
        "source": "agent",
        "target": "__end__",
        "data": "end",
        "conditional": true
      },
      {
        "source": "agent",
        "target": "tools",
        "data": "continue",
        "conditional": true
      },
      {
        "source": "tools",
        "target": "agent"
      }
    ]
  }
  '''
# ---
# name: test_message_graph[memory].3
  '''
  graph TD;
  	__start__ --> agent;
  	agent -. &nbsp;end&nbsp; .-> __end__;
  	agent -. &nbsp;continue&nbsp; .-> tools;
  	tools --> agent;
  
  '''
# ---
# name: test_prebuilt_tool_chat
  '{"$defs": {"BaseMessage": {"additionalProperties": true, "description": "Base abstract message class.\\n\\nMessages are the inputs and outputs of a chat model.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}}, "required": ["content", "type"], "title": "BaseMessage", "type": "object"}}, "deprecated": true, "description": "The state of the agent.", "properties": {"messages": {"items": {"$ref": "#/$defs/BaseMessage"}, "title": "Messages", "type": "array"}, "remaining_steps": {"title": "Remaining Steps", "type": "integer"}}, "required": ["messages"], "title": "AgentState", "type": "object"}'
# ---
# name: test_prebuilt_tool_chat.1
  '{"$defs": {"BaseMessage": {"additionalProperties": true, "description": "Base abstract message class.\\n\\nMessages are the inputs and outputs of a chat model.", "properties": {"content": {"anyOf": [{"type": "string"}, {"items": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}]}, "type": "array"}], "title": "Content"}, "additional_kwargs": {"additionalProperties": true, "title": "Additional Kwargs", "type": "object"}, "response_metadata": {"additionalProperties": true, "title": "Response Metadata", "type": "object"}, "type": {"title": "Type", "type": "string"}, "name": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Name"}, "id": {"anyOf": [{"type": "string"}, {"type": "null"}], "default": null, "title": "Id"}}, "required": ["content", "type"], "title": "BaseMessage", "type": "object"}}, "deprecated": true, "description": "The state of the agent.", "properties": {"messages": {"items": {"$ref": "#/$defs/BaseMessage"}, "title": "Messages", "type": "array"}, "remaining_steps": {"title": "Remaining Steps", "type": "integer"}}, "required": ["messages"], "title": "AgentState", "type": "object"}'
# ---
# name: test_prebuilt_tool_chat.2
  '''
  {
    "nodes": [
      {
        "id": "__start__",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "__start__"
        }
      },
      {
        "id": "agent",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "agent"
        }
      },
      {
        "id": "tools",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "prebuilt",
            "tool_node",
            "ToolNode"
          ],
          "name": "tools"
        }
      },
      {
        "id": "__end__"
      }
    ],
    "edges": [
      {
        "source": "__start__",
        "target": "agent"
      },
      {
        "source": "agent",
        "target": "__end__",
        "conditional": true
      },
      {
        "source": "agent",
        "target": "tools",
        "conditional": true
      },
      {
        "source": "tools",
        "target": "agent"
      }
    ]
  }
  '''
# ---
# name: test_prebuilt_tool_chat.3
  '''
  graph TD;
  	__start__ --> agent;
  	agent -.-> __end__;
  	agent -.-> tools;
  	tools --> agent;
  
  '''
# ---
# name: test_send_react_interrupt_control[memory]
  '''
  ---
  config:
    flowchart:
      curve: linear
  ---
  graph TD;
  	__start__([<p>__start__</p>]):::first
  	agent(agent)
  	foo(foo)
  	__end__([<p>__end__</p>]):::last
  	__start__ --> agent;
  	agent -.-> foo;
  	foo --> __end__;
  	classDef default fill:#f2f0ff,line-height:1.2
  	classDef first fill-opacity:0
  	classDef last fill:#bfb6fc
  
  '''
# ---
# name: test_weather_subgraph[memory]
  '''
  ---
  config:
    flowchart:
      curve: linear
  ---
  graph TD;
  	__start__([<p>__start__</p>]):::first
  	router_node(router_node)
  	normal_llm_node(normal_llm_node)
  	__end__([<p>__end__</p>]):::last
  	__start__ --> router_node;
  	router_node -.-> normal_llm_node;
  	router_node -.-> weather_graph\3amodel_node;
  	normal_llm_node --> __end__;
  	weather_graph\3aweather_node --> __end__;
  	subgraph weather_graph
  	weather_graph\3amodel_node(model_node)
  	weather_graph\3aweather_node(weather_node<hr/><small><em>__interrupt = before</em></small>)
  	weather_graph\3amodel_node --> weather_graph\3aweather_node;
  	end
  	classDef default fill:#f2f0ff,line-height:1.2
  	classDef first fill-opacity:0
  	classDef last fill:#bfb6fc
  
  '''
# ---

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/__snapshots__/test_pregel.ambr
```ambr
# serializer version: 1
# name: test_conditional_entrypoint_graph_state
  '{"properties": {"input": {"title": "Input", "type": "string"}, "output": {"title": "Output", "type": "string"}, "steps": {"items": {"type": "string"}, "title": "Steps", "type": "array"}}, "title": "AgentState", "type": "object"}'
# ---
# name: test_conditional_entrypoint_graph_state.1
  '{"properties": {"input": {"title": "Input", "type": "string"}, "output": {"title": "Output", "type": "string"}, "steps": {"items": {"type": "string"}, "title": "Steps", "type": "array"}}, "title": "AgentState", "type": "object"}'
# ---
# name: test_conditional_entrypoint_graph_state.2
  '''
  {
    "nodes": [
      {
        "id": "__start__",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "__start__"
        }
      },
      {
        "id": "left",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "left"
        }
      },
      {
        "id": "right",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "right"
        }
      },
      {
        "id": "__end__"
      }
    ],
    "edges": [
      {
        "source": "__start__",
        "target": "left",
        "data": "go-left",
        "conditional": true
      },
      {
        "source": "__start__",
        "target": "right",
        "data": "go-right",
        "conditional": true
      },
      {
        "source": "left",
        "target": "__end__",
        "conditional": true
      },
      {
        "source": "right",
        "target": "__end__"
      }
    ]
  }
  '''
# ---
# name: test_conditional_entrypoint_graph_state.3
  '''
  graph TD;
  	__start__ -. &nbsp;go-left&nbsp; .-> left;
  	__start__ -. &nbsp;go-right&nbsp; .-> right;
  	left -.-> __end__;
  	right --> __end__;
  
  '''
# ---
# name: test_conditional_entrypoint_to_multiple_state_graph
  '{"properties": {"locations": {"items": {"type": "string"}, "title": "Locations", "type": "array"}, "results": {"items": {"type": "string"}, "title": "Results", "type": "array"}}, "required": ["locations", "results"], "title": "OverallState", "type": "object"}'
# ---
# name: test_conditional_entrypoint_to_multiple_state_graph.1
  '{"properties": {"locations": {"items": {"type": "string"}, "title": "Locations", "type": "array"}, "results": {"items": {"type": "string"}, "title": "Results", "type": "array"}}, "required": ["locations", "results"], "title": "OverallState", "type": "object"}'
# ---
# name: test_conditional_entrypoint_to_multiple_state_graph.2
  '''
  {
    "nodes": [
      {
        "id": "__start__",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "__start__"
        }
      },
      {
        "id": "get_weather",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "get_weather"
        }
      },
      {
        "id": "__end__"
      }
    ],
    "edges": [
      {
        "source": "__start__",
        "target": "get_weather",
        "conditional": true
      },
      {
        "source": "get_weather",
        "target": "__end__"
      }
    ]
  }
  '''
# ---
# name: test_conditional_entrypoint_to_multiple_state_graph.3
  '''
  graph TD;
  	__start__ -.-> get_weather;
  	get_weather --> __end__;
  
  '''
# ---
# name: test_conditional_state_graph_with_list_edge_inputs
  '''
  {
    "nodes": [
      {
        "id": "__start__",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "__start__"
        }
      },
      {
        "id": "A",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "A"
        }
      },
      {
        "id": "B",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "B"
        }
      },
      {
        "id": "__end__"
      }
    ],
    "edges": [
      {
        "source": "__start__",
        "target": "A"
      },
      {
        "source": "__start__",
        "target": "B"
      },
      {
        "source": "A",
        "target": "__end__"
      },
      {
        "source": "B",
        "target": "__end__"
      }
    ]
  }
  '''
# ---
# name: test_conditional_state_graph_with_list_edge_inputs.1
  '''
  graph TD;
  	__start__ --> A;
  	__start__ --> B;
  	A --> __end__;
  	B --> __end__;
  
  '''
# ---
# name: test_get_graph_loop
  '''
  {
    "nodes": [
      {
        "id": "__start__",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "__start__"
        }
      },
      {
        "id": "human",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "human"
        }
      },
      {
        "id": "agent",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "agent"
        }
      },
      {
        "id": "__end__"
      }
    ],
    "edges": [
      {
        "source": "__start__",
        "target": "human"
      },
      {
        "source": "agent",
        "target": "human"
      },
      {
        "source": "human",
        "target": "agent"
      },
      {
        "source": "agent",
        "target": "__end__",
        "conditional": true
      }
    ]
  }
  '''
# ---
# name: test_get_graph_loop.1
  '''
  graph TD;
  	__start__ --> human;
  	agent --> human;
  	human --> agent;
  	agent -.-> __end__;
  
  '''
# ---
# name: test_get_graph_nonterminal_last_step_source
  '''
  {
    "edges": [
      {
        "source": "__start__",
        "target": "human"
      },
      {
        "conditional": true,
        "source": "chatbot",
        "target": "human"
      },
      {
        "conditional": true,
        "source": "chatbot",
        "target": "tools"
      },
      {
        "conditional": true,
        "source": "human",
        "target": "__end__"
      },
      {
        "conditional": true,
        "source": "human",
        "target": "chatbot"
      },
      {
        "source": "tools",
        "target": "chatbot"
      }
    ],
    "nodes": [
      {
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "__start__"
        },
        "id": "__start__",
        "type": "runnable"
      },
      {
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "chatbot"
        },
        "id": "chatbot",
        "type": "runnable"
      },
      {
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "tools"
        },
        "id": "tools",
        "type": "runnable"
      },
      {
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "human"
        },
        "id": "human",
        "type": "runnable"
      },
      {
        "id": "__end__"
      }
    ]
  }
  '''
# ---
# name: test_get_graph_root_channel
  '''
  {
    "nodes": [
      {
        "id": "__start__",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "__start__"
        }
      },
      {
        "id": "child",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "graph",
            "state",
            "CompiledStateGraph"
          ],
          "name": "child"
        }
      },
      {
        "id": "__end__"
      }
    ],
    "edges": [
      {
        "source": "__start__",
        "target": "child"
      },
      {
        "source": "child",
        "target": "__end__"
      }
    ]
  }
  '''
# ---
# name: test_get_graph_root_channel.1
  '''
  graph TD;
  	__start__ --> child;
  	child --> __end__;
  
  '''
# ---
# name: test_get_graph_self_loop
  '''
  {
    "nodes": [
      {
        "id": "__start__",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "__start__"
        }
      },
      {
        "id": "worker_node",
        "type": "runnable",
        "data": {
          "id": [
            "langgraph",
            "_internal",
            "_runnable",
            "RunnableCallable"
          ],
          "name": "worker_node"
        }
      },
      {
        "id": "__end__"
      }
    ],
    "edges": [
      {
        "source": "__start__",
        "target": "worker_node"
      },
      {
        "source": "worker_node",
        "target": "__end__",
        "conditional": true
      },
      {
        "source": "worker_node",
        "target": "worker_node",
        "conditional": true
      }
    ]
  }
  '''
# ---
# name: test_get_graph_self_loop.1
  '''
  graph TD;
  	__start__ --> worker_node;
  	worker_node -.-> __end__;
  	worker_node -.-> worker_node;
  
  '''
# ---
# name: test_in_one_fan_out_state_graph_defer_node[memory-False]
  '''
  graph TD;
  	__start__ --> rewrite_query;
  	analyzer_one -.-> qa;
  	retriever_one --> analyzer_one;
  	retriever_one --> qa;
  	retriever_two --> qa;
  	rewrite_query --> retriever_one;
  	rewrite_query --> retriever_two;
  	qa --> __end__;
  
  '''
# ---
# name: test_in_one_fan_out_state_graph_defer_node[memory-True]
  '''
  graph TD;
  	__start__ --> rewrite_query;
  	analyzer_one -.-> qa;
  	retriever_one --> analyzer_one;
  	retriever_one --> qa;
  	retriever_two --> qa;
  	rewrite_query --> retriever_one;
  	rewrite_query --> retriever_two;
  	qa --> __end__;
  
  '''
# ---
# name: test_in_one_fan_out_state_graph_waiting_edge[memory]
  '''
  graph TD;
  	__start__ --> rewrite_query;
  	analyzer_one --> retriever_one;
  	retriever_one --> qa;
  	retriever_two --> qa;
  	rewrite_query --> analyzer_one;
  	rewrite_query --> retriever_two;
  	qa --> __end__;
  
  '''
# ---
# name: test_in_one_fan_out_state_graph_waiting_edge_custom_state_class_pydantic2[memory]
  '''
  graph TD;
  	__start__ --> rewrite_query;
  	analyzer_one --> retriever_one;
  	retriever_one --> qa;
  	retriever_two --> qa;
  	rewrite_query --> analyzer_one;
  	rewrite_query -.-> retriever_two;
  	qa --> __end__;
  
  '''
# ---
# name: test_in_one_fan_out_state_graph_waiting_edge_custom_state_class_pydantic2[memory].1
  dict({
    '$defs': dict({
      'InnerObject': dict({
        'properties': dict({
          'yo': dict({
            'title': 'Yo',
            'type': 'integer',
          }),
        }),
        'required': list([
          'yo',
        ]),
        'title': 'InnerObject',
        'type': 'object',
      }),
    }),
    'properties': dict({
      'inner': dict({
        '$ref': '#/$defs/InnerObject',
      }),
      'query': dict({
        'title': 'Query',
        'type': 'string',
      }),
    }),
    'required': list([
      'query',
      'inner',
    ]),
    'title': 'Input',
    'type': 'object',
  })
# ---
# name: test_in_one_fan_out_state_graph_waiting_edge_custom_state_class_pydantic2[memory].2
  dict({
    'properties': dict({
      'answer': dict({
        'title': 'Answer',
        'type': 'string',
      }),
      'docs': dict({
        'items': dict({
          'type': 'string',
        }),
        'title': 'Docs',
        'type': 'array',
      }),
    }),
    'required': list([
      'answer',
      'docs',
    ]),
    'title': 'Output',
    'type': 'object',
  })
# ---
# name: test_in_one_fan_out_state_graph_waiting_edge_via_branch[memory]
  '''
  graph TD;
  	__start__ --> rewrite_query;
  	analyzer_one --> retriever_one;
  	retriever_one --> qa;
  	retriever_two --> qa;
  	rewrite_query --> analyzer_one;
  	rewrite_query -.-> retriever_two;
  	qa --> __end__;
  
  '''
# ---
# name: test_migration_graph
  '''
  graph TD;
  	B -. &nbsp;X&nbsp; .-> C;
  	B -. &nbsp;Y&nbsp; .-> D;
  	D --> B;
  	__start__ --> B;
  	C --> __end__;
  
  '''
# ---
# name: test_multiple_sinks_subgraphs
  '''
  ---
  config:
    flowchart:
      curve: linear
  ---
  graph TD;
  	__start__([<p>__start__</p>]):::first
  	uno(uno)
  	dos(dos)
  	__end__([<p>__end__</p>]):::last
  	__start__ --> uno;
  	uno -.-> dos;
  	uno -.-> subgraph\3aone;
  	dos --> __end__;
  	subgraph\3a__end__ --> __end__;
  	subgraph subgraph
  	subgraph\3aone(one)
  	subgraph\3atwo(two)
  	subgraph\3athree(three)
  	subgraph\3a__end__(<p>__end__</p>)
  	subgraph\3aone -.-> subgraph\3athree;
  	subgraph\3aone -.-> subgraph\3atwo;
  	subgraph\3athree --> subgraph\3a__end__;
  	subgraph\3atwo --> subgraph\3a__end__;
  	end
  	classDef default fill:#f2f0ff,line-height:1.2
  	classDef first fill-opacity:0
  	classDef last fill:#bfb6fc
  
  '''
# ---
# name: test_nested_graph
  '''
  graph TD;
  	__start__ --> inner;
  	inner --> side;
  	side --> __end__;
  
  '''
# ---
# name: test_nested_graph.1
  '''
  ---
  config:
    flowchart:
      curve: linear
  ---
  graph TD;
  	__start__([<p>__start__</p>]):::first
  	side(side)
  	__end__([<p>__end__</p>]):::last
  	__start__ --> inner\3aup;
  	inner\3aup --> side;
  	side --> __end__;
  	subgraph inner
  	inner\3aup(up)
  	end
  	classDef default fill:#f2f0ff,line-height:1.2
  	classDef first fill-opacity:0
  	classDef last fill:#bfb6fc
  
  '''
# ---
# name: test_nested_graph_xray
  dict({
    'edges': list([
      dict({
        'conditional': True,
        'source': '__start__',
        'target': 'tool_one',
      }),
      dict({
        'conditional': True,
        'source': '__start__',
        'target': 'tool_three',
      }),
      dict({
        'conditional': True,
        'source': '__start__',
        'target': 'tool_two:__start__',
      }),
      dict({
        'source': 'tool_one',
        'target': '__end__',
      }),
      dict({
        'source': 'tool_three',
        'target': '__end__',
      }),
      dict({
        'source': 'tool_two:__end__',
        'target': '__end__',
      }),
      dict({
        'conditional': True,
        'source': 'tool_two:__start__',
        'target': 'tool_two:tool_two_fast',
      }),
      dict({
        'conditional': True,
        'source': 'tool_two:__start__',
        'target': 'tool_two:tool_two_slow',
      }),
      dict({
        'source': 'tool_two:tool_two_fast',
        'target': 'tool_two:__end__',
      }),
      dict({
        'source': 'tool_two:tool_two_slow',
        'target': 'tool_two:__end__',
      }),
    ]),
    'nodes': list([
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': '__start__',
        }),
        'id': '__start__',
        'type': 'runnable',
      }),
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': 'tool_one',
        }),
        'id': 'tool_one',
        'type': 'runnable',
      }),
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': 'tool_three',
        }),
        'id': 'tool_three',
        'type': 'runnable',
      }),
      dict({
        'id': '__end__',
      }),
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': 'tool_two:__start__',
        }),
        'id': 'tool_two:__start__',
        'type': 'runnable',
      }),
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': 'tool_two:tool_two_slow',
        }),
        'id': 'tool_two:tool_two_slow',
        'type': 'runnable',
      }),
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': 'tool_two:tool_two_fast',
        }),
        'id': 'tool_two:tool_two_fast',
        'type': 'runnable',
      }),
      dict({
        'id': 'tool_two:__end__',
      }),
    ]),
  })
# ---
# name: test_nested_graph_xray.1
  '''
  ---
  config:
    flowchart:
      curve: linear
  ---
  graph TD;
  	__start__([<p>__start__</p>]):::first
  	tool_one(tool_one)
  	tool_three(tool_three)
  	__end__([<p>__end__</p>]):::last
  	__start__ -.-> tool_one;
  	__start__ -.-> tool_three;
  	__start__ -.-> tool_two\3a__start__;
  	tool_one --> __end__;
  	tool_three --> __end__;
  	tool_two\3a__end__ --> __end__;
  	subgraph tool_two
  	tool_two\3a__start__(<p>__start__</p>)
  	tool_two\3atool_two_slow(tool_two_slow)
  	tool_two\3atool_two_fast(tool_two_fast)
  	tool_two\3a__end__(<p>__end__</p>)
  	tool_two\3a__start__ -.-> tool_two\3atool_two_fast;
  	tool_two\3a__start__ -.-> tool_two\3atool_two_slow;
  	tool_two\3atool_two_fast --> tool_two\3a__end__;
  	tool_two\3atool_two_slow --> tool_two\3a__end__;
  	end
  	classDef default fill:#f2f0ff,line-height:1.2
  	classDef first fill-opacity:0
  	classDef last fill:#bfb6fc
  
  '''
# ---
# name: test_repeat_condition
  '''
  graph TD;
  	Call\20Tool -.-> Chart\20Generator;
  	Call\20Tool -.-> Researcher;
  	Chart\20Generator -. &nbsp;call_tool&nbsp; .-> Call\20Tool;
  	Chart\20Generator -. &nbsp;continue&nbsp; .-> Researcher;
  	Chart\20Generator -. &nbsp;end&nbsp; .-> __end__;
  	Researcher -. &nbsp;call_tool&nbsp; .-> Call\20Tool;
  	Researcher -. &nbsp;continue&nbsp; .-> Chart\20Generator;
  	Researcher -. &nbsp;end&nbsp; .-> __end__;
  	__start__ --> Researcher;
  	Researcher -. &nbsp;redo&nbsp; .-> Researcher;
  
  '''
# ---
# name: test_simple_multi_edge
  '''
  graph TD;
  	__start__ --> up;
  	side --> down;
  	up --> down;
  	up --> other;
  	up --> side;
  	down --> __end__;
  	other --> __end__;
  
  '''
# ---
# name: test_state_graph_w_config_inherited_state_keys
  '{"properties": {"tools": {"items": {"type": "string"}, "title": "Tools", "type": "array"}}, "title": "Context", "type": "object"}'
# ---
# name: test_state_graph_w_config_inherited_state_keys.1
  '{"$defs": {"AgentAction": {"description": "Represents a request to execute an action by an agent.\\n\\nThe action consists of the name of the tool to execute and the input to pass\\nto the tool. The log is used to pass along extra information about the action.", "properties": {"tool": {"title": "Tool", "type": "string"}, "tool_input": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}], "title": "Tool Input"}, "log": {"title": "Log", "type": "string"}, "type": {"const": "AgentAction", "default": "AgentAction", "title": "Type", "type": "string"}}, "required": ["tool", "tool_input", "log"], "title": "AgentAction", "type": "object"}, "AgentFinish": {"description": "Final return value of an ActionAgent.\\n\\nAgents return an AgentFinish when they have reached a stopping condition.", "properties": {"return_values": {"additionalProperties": true, "title": "Return Values", "type": "object"}, "log": {"title": "Log", "type": "string"}, "type": {"const": "AgentFinish", "default": "AgentFinish", "title": "Type", "type": "string"}}, "required": ["return_values", "log"], "title": "AgentFinish", "type": "object"}}, "properties": {"input": {"title": "Input", "type": "string"}, "agent_outcome": {"anyOf": [{"$ref": "#/$defs/AgentAction"}, {"$ref": "#/$defs/AgentFinish"}, {"type": "null"}], "title": "Agent Outcome"}, "intermediate_steps": {"items": {"maxItems": 2, "minItems": 2, "prefixItems": [{"$ref": "#/$defs/AgentAction"}, {"type": "string"}], "type": "array"}, "title": "Intermediate Steps", "type": "array"}}, "required": ["input", "agent_outcome"], "title": "AgentState", "type": "object"}'
# ---
# name: test_state_graph_w_config_inherited_state_keys.2
  '{"$defs": {"AgentAction": {"description": "Represents a request to execute an action by an agent.\\n\\nThe action consists of the name of the tool to execute and the input to pass\\nto the tool. The log is used to pass along extra information about the action.", "properties": {"tool": {"title": "Tool", "type": "string"}, "tool_input": {"anyOf": [{"type": "string"}, {"additionalProperties": true, "type": "object"}], "title": "Tool Input"}, "log": {"title": "Log", "type": "string"}, "type": {"const": "AgentAction", "default": "AgentAction", "title": "Type", "type": "string"}}, "required": ["tool", "tool_input", "log"], "title": "AgentAction", "type": "object"}, "AgentFinish": {"description": "Final return value of an ActionAgent.\\n\\nAgents return an AgentFinish when they have reached a stopping condition.", "properties": {"return_values": {"additionalProperties": true, "title": "Return Values", "type": "object"}, "log": {"title": "Log", "type": "string"}, "type": {"const": "AgentFinish", "default": "AgentFinish", "title": "Type", "type": "string"}}, "required": ["return_values", "log"], "title": "AgentFinish", "type": "object"}}, "properties": {"input": {"title": "Input", "type": "string"}, "agent_outcome": {"anyOf": [{"$ref": "#/$defs/AgentAction"}, {"$ref": "#/$defs/AgentFinish"}, {"type": "null"}], "title": "Agent Outcome"}, "intermediate_steps": {"items": {"maxItems": 2, "minItems": 2, "prefixItems": [{"$ref": "#/$defs/AgentAction"}, {"type": "string"}], "type": "array"}, "title": "Intermediate Steps", "type": "array"}}, "required": ["input", "agent_outcome"], "title": "AgentState", "type": "object"}'
# ---
# name: test_xray_bool
  '''
  ---
  config:
    flowchart:
      curve: linear
  ---
  graph TD;
  	__start__([<p>__start__</p>]):::first
  	gp_one(gp_one)
  	__end__([<p>__end__</p>]):::last
  	__start__ --> gp_one;
  	gp_one -. &nbsp;1&nbsp; .-> __end__;
  	gp_one -. &nbsp;0&nbsp; .-> gp_two\3a__start__;
  	gp_two\3a__end__ --> gp_one;
  	subgraph gp_two
  	gp_two\3a__start__(<p>__start__</p>)
  	gp_two\3ap_one(p_one)
  	gp_two\3a__end__(<p>__end__</p>)
  	gp_two\3a__start__ --> gp_two\3ap_one;
  	gp_two\3ap_one -. &nbsp;1&nbsp; .-> gp_two\3a__end__;
  	gp_two\3ap_one -. &nbsp;0&nbsp; .-> gp_two\3ap_two\3a__start__;
  	gp_two\3ap_two\3a__end__ --> gp_two\3ap_one;
  	subgraph p_two
  	gp_two\3ap_two\3a__start__(<p>__start__</p>)
  	gp_two\3ap_two\3ac_one(c_one)
  	gp_two\3ap_two\3ac_two(c_two)
  	gp_two\3ap_two\3a__end__(<p>__end__</p>)
  	gp_two\3ap_two\3a__start__ --> gp_two\3ap_two\3ac_one;
  	gp_two\3ap_two\3ac_one -. &nbsp;1&nbsp; .-> gp_two\3ap_two\3a__end__;
  	gp_two\3ap_two\3ac_one -. &nbsp;0&nbsp; .-> gp_two\3ap_two\3ac_two;
  	gp_two\3ap_two\3ac_two --> gp_two\3ap_two\3ac_one;
  	end
  	end
  	classDef default fill:#f2f0ff,line-height:1.2
  	classDef first fill-opacity:0
  	classDef last fill:#bfb6fc
  
  '''
# ---
# name: test_xray_issue
  '''
  ---
  config:
    flowchart:
      curve: linear
  ---
  graph TD;
  	__start__([<p>__start__</p>]):::first
  	p_one(p_one)
  	__end__([<p>__end__</p>]):::last
  	__start__ --> p_one;
  	p_one -. &nbsp;1&nbsp; .-> __end__;
  	p_one -. &nbsp;0&nbsp; .-> p_two\3a__start__;
  	p_two\3a__end__ --> p_one;
  	subgraph p_two
  	p_two\3a__start__(<p>__start__</p>)
  	p_two\3ac_one(c_one)
  	p_two\3ac_two(c_two)
  	p_two\3a__end__(<p>__end__</p>)
  	p_two\3a__start__ --> p_two\3ac_one;
  	p_two\3ac_one -. &nbsp;1&nbsp; .-> p_two\3a__end__;
  	p_two\3ac_one -. &nbsp;0&nbsp; .-> p_two\3ac_two;
  	p_two\3ac_two --> p_two\3ac_one;
  	end
  	classDef default fill:#f2f0ff,line-height:1.2
  	classDef first fill-opacity:0
  	classDef last fill:#bfb6fc
  
  '''
# ---
# name: test_xray_lance
  dict({
    'edges': list([
      dict({
        'source': '__start__',
        'target': 'ask_question',
      }),
      dict({
        'conditional': True,
        'source': 'answer_question',
        'target': '__end__',
      }),
      dict({
        'conditional': True,
        'source': 'answer_question',
        'target': 'ask_question',
      }),
      dict({
        'source': 'ask_question',
        'target': 'answer_question',
      }),
    ]),
    'nodes': list([
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': '__start__',
        }),
        'id': '__start__',
        'type': 'runnable',
      }),
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': 'ask_question',
        }),
        'id': 'ask_question',
        'type': 'runnable',
      }),
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': 'answer_question',
        }),
        'id': 'answer_question',
        'type': 'runnable',
      }),
      dict({
        'id': '__end__',
      }),
    ]),
  })
# ---
# name: test_xray_lance.1
  dict({
    'edges': list([
      dict({
        'source': '__start__',
        'target': 'generate_analysts',
      }),
      dict({
        'source': 'conduct_interview',
        'target': 'generate_sections',
      }),
      dict({
        'conditional': True,
        'source': 'generate_analysts',
        'target': 'conduct_interview',
      }),
      dict({
        'source': 'generate_sections',
        'target': '__end__',
      }),
    ]),
    'nodes': list([
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': '__start__',
        }),
        'id': '__start__',
        'type': 'runnable',
      }),
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': 'generate_analysts',
        }),
        'id': 'generate_analysts',
        'type': 'runnable',
      }),
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            'graph',
            'state',
            'CompiledStateGraph',
          ]),
          'name': 'conduct_interview',
        }),
        'id': 'conduct_interview',
        'type': 'runnable',
      }),
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': 'generate_sections',
        }),
        'id': 'generate_sections',
        'type': 'runnable',
      }),
      dict({
        'id': '__end__',
      }),
    ]),
  })
# ---
# name: test_xray_lance.2
  dict({
    'edges': list([
      dict({
        'source': '__start__',
        'target': 'generate_analysts',
      }),
      dict({
        'source': 'conduct_interview:__end__',
        'target': 'generate_sections',
      }),
      dict({
        'conditional': True,
        'source': 'generate_analysts',
        'target': 'conduct_interview:__start__',
      }),
      dict({
        'source': 'generate_sections',
        'target': '__end__',
      }),
      dict({
        'source': 'conduct_interview:__start__',
        'target': 'conduct_interview:ask_question',
      }),
      dict({
        'conditional': True,
        'source': 'conduct_interview:answer_question',
        'target': 'conduct_interview:__end__',
      }),
      dict({
        'conditional': True,
        'source': 'conduct_interview:answer_question',
        'target': 'conduct_interview:ask_question',
      }),
      dict({
        'source': 'conduct_interview:ask_question',
        'target': 'conduct_interview:answer_question',
      }),
    ]),
    'nodes': list([
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': '__start__',
        }),
        'id': '__start__',
        'type': 'runnable',
      }),
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': 'generate_analysts',
        }),
        'id': 'generate_analysts',
        'type': 'runnable',
      }),
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': 'generate_sections',
        }),
        'id': 'generate_sections',
        'type': 'runnable',
      }),
      dict({
        'id': '__end__',
      }),
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': 'conduct_interview:__start__',
        }),
        'id': 'conduct_interview:__start__',
        'type': 'runnable',
      }),
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': 'conduct_interview:ask_question',
        }),
        'id': 'conduct_interview:ask_question',
        'type': 'runnable',
      }),
      dict({
        'data': dict({
          'id': list([
            'langgraph',
            '_internal',
            '_runnable',
            'RunnableCallable',
          ]),
          'name': 'conduct_interview:answer_question',
        }),
        'id': 'conduct_interview:answer_question',
        'type': 'runnable',
      }),
      dict({
        'id': 'conduct_interview:__end__',
      }),
    ]),
  })
# ---

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/__snapshots__/test_pregel_async.ambr
```ambr
# serializer version: 1
# name: test_in_one_fan_out_state_graph_waiting_edge_custom_state_class_pydantic2[memory]
  '''
  graph TD;
  	__start__ --> rewrite_query;
  	analyzer_one --> retriever_one;
  	retriever_one --> qa;
  	retriever_two --> qa;
  	rewrite_query --> analyzer_one;
  	rewrite_query -.-> retriever_two;
  	qa --> __end__;
  
  '''
# ---
# name: test_in_one_fan_out_state_graph_waiting_edge_custom_state_class_pydantic2[memory].1
  dict({
    '$defs': dict({
      'InnerObject': dict({
        'properties': dict({
          'yo': dict({
            'title': 'Yo',
            'type': 'integer',
          }),
        }),
        'required': list([
          'yo',
        ]),
        'title': 'InnerObject',
        'type': 'object',
      }),
    }),
    'properties': dict({
      'answer': dict({
        'anyOf': list([
          dict({
            'type': 'string',
          }),
          dict({
            'type': 'null',
          }),
        ]),
        'default': None,
        'title': 'Answer',
      }),
      'docs': dict({
        'items': dict({
          'type': 'string',
        }),
        'title': 'Docs',
        'type': 'array',
      }),
      'inner': dict({
        '$ref': '#/$defs/InnerObject',
      }),
      'query': dict({
        'title': 'Query',
        'type': 'string',
      }),
    }),
    'required': list([
      'query',
      'inner',
      'docs',
    ]),
    'title': 'State',
    'type': 'object',
  })
# ---
# name: test_in_one_fan_out_state_graph_waiting_edge_custom_state_class_pydantic2[memory].2
  dict({
    '$defs': dict({
      'InnerObject': dict({
        'properties': dict({
          'yo': dict({
            'title': 'Yo',
            'type': 'integer',
          }),
        }),
        'required': list([
          'yo',
        ]),
        'title': 'InnerObject',
        'type': 'object',
      }),
    }),
    'properties': dict({
      'answer': dict({
        'anyOf': list([
          dict({
            'type': 'string',
          }),
          dict({
            'type': 'null',
          }),
        ]),
        'default': None,
        'title': 'Answer',
      }),
      'docs': dict({
        'items': dict({
          'type': 'string',
        }),
        'title': 'Docs',
        'type': 'array',
      }),
      'inner': dict({
        '$ref': '#/$defs/InnerObject',
      }),
      'query': dict({
        'title': 'Query',
        'type': 'string',
      }),
    }),
    'required': list([
      'query',
      'inner',
      'docs',
    ]),
    'title': 'State',
    'type': 'object',
  })
# ---
# name: test_send_react_interrupt_control[memory]
  '''
  ---
  config:
    flowchart:
      curve: linear
  ---
  graph TD;
  	__start__([<p>__start__</p>]):::first
  	agent(agent)
  	foo(foo)
  	__end__([<p>__end__</p>]):::last
  	__start__ --> agent;
  	agent -.-> foo;
  	foo --> __end__;
  	classDef default fill:#f2f0ff,line-height:1.2
  	classDef first fill-opacity:0
  	classDef last fill:#bfb6fc
  
  '''
# ---

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/example_app/example_graph.py
```py
from typing import Annotated

from langchain_core.messages import AIMessage, BaseMessage, ToolMessage
from langchain_core.tools import tool
from typing_extensions import TypedDict

from langgraph.func import entrypoint, task
from langgraph.graph.message import add_messages
from tests.fake_chat import FakeChatModel


class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]


@tool
def search_api(query: str) -> str:
    """Searches the API for the query."""
    return f"result for {query}"


tools = [search_api]
tools_by_name = {t.name: t for t in tools}


def get_model():
    model = FakeChatModel(
        messages=[
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
    return model


@task
def foo():
    return "foo"


@entrypoint()
async def app(state: AgentState) -> AgentState:
    model = get_model()
    max_steps = 100
    messages = state["messages"][:]
    await foo()  # Very useful call here ya know.
    for _ in range(max_steps):
        message = await model.ainvoke(messages)
        messages.append(message)
        if not message.tool_calls:
            break
        # Assume it's the search tool
        tool_results = await search_api.abatch(
            [t["args"]["query"] for t in message.tool_calls]
        )
        messages.extend(
            [
                ToolMessage(content=tool_res, tool_call_id=tc["id"])
                for tc, tool_res in zip(message.tool_calls, tool_results)
            ]
        )

    return entrypoint.final(value=messages[-1], save={"messages": messages})

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/example_app/langgraph.json
```json
{
  "$schema": "https://langgra.ph/schema.json",
  "graphs": {
    "app": "tests/example_app/example_graph.py:app"
  },
  "dependencies": ["tests/example_app"]
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/tests/example_app/requirements.txt
```txt
langchain-core
-e .
```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/LICENSE
```text
MIT License

Copyright (c) 2024 LangChain, Inc.

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/Makefile
```text
.PHONY: lint format test

test:
	uv run pytest tests

######################
# LINTING AND FORMATTING
######################

# Define a variable for Python and notebook files.
PYTHON_FILES=.
MYPY_CACHE=.mypy_cache
lint format: PYTHON_FILES=.
lint_diff format_diff: PYTHON_FILES=$(shell git diff --name-only --relative --diff-filter=d main . | grep -E '\.py$$|\.ipynb$$')

lint lint_diff:
	uv run ruff check .
	[ "$(PYTHON_FILES)" = "" ] || uv run ruff format $(PYTHON_FILES) --diff
	[ "$(PYTHON_FILES)" = "" ] || uv run ruff check --select I $(PYTHON_FILES)
	uvx ty check .

format format_diff:
	uv run ruff check --select I --fix $(PYTHON_FILES)
	uv run ruff format $(PYTHON_FILES)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/README.md
```md
# LangGraph Python SDK

This repository contains the Python SDK for interacting with the LangSmith Deployment REST API.

## Quick Start

To get started with the Python SDK, [install the package](https://pypi.org/project/langgraph-sdk/)

```bash
pip install -U langgraph-sdk
```

You will need a running LangGraph API server. If you're running a server locally using `langgraph-cli`, SDK will automatically point at `http://localhost:8123`, otherwise
you would need to specify the server URL when creating a client.

```python
from langgraph_sdk import get_client

# If you're using a remote server, initialize the client with `get_client(url=REMOTE_URL)`
client = get_client()

# List all assistants
assistants = await client.assistants.search()

# We auto-create an assistant for each graph you register in config.
agent = assistants[0]

# Start a new thread
thread = await client.threads.create()

# Start a streaming run
input = {"messages": [{"role": "human", "content": "what's the weather in la"}]}
async for chunk in client.runs.stream(thread['thread_id'], agent['assistant_id'], input=input):
    print(chunk)
```

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/pyproject.toml
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "langgraph-sdk"
dynamic = ["version"]
description = "SDK for interacting with LangGraph API"
authors = []
requires-python = ">=3.10"
readme = "README.md"
license = "MIT"
license-files = ['LICENSE']
dependencies = [
    "httpx>=0.25.2",
    "orjson>=3.10.1",
]

[tool.hatch.version]
path = "langgraph_sdk/__init__.py"

[project.urls]
Source = "https://github.com/langchain-ai/langgraph/tree/main/libs/sdk-py"
Twitter = "https://x.com/LangChainAI"
Slack = "https://www.langchain.com/join-community"
Reddit = "https://www.reddit.com/r/LangChain/"

[dependency-groups]
test = [
  "pytest",
  "pytest-asyncio",
  "pytest-mock",
  "pytest-watch",
]
lint = [
  "ruff",
  "codespell",
  "mypy",
]
dev = [
  {include-group = "test"},
  {include-group = "lint"},
]

[tool.hatch.build.targets.wheel]
include = ["langgraph_sdk"]

[tool.pytest.ini_options]
addopts = "--strict-markers --strict-config --durations=5 -vv"
asyncio_mode = "auto"

[tool.uv]
default-groups = ['dev']

[tool.ruff]
lint.select = [
  "E",  # pycodestyle
  "F",  # Pyflakes
  "UP", # pyupgrade
  "B",  # flake8-bugbear
  "I",  # isort
  "ARG", # flake8-unused-arguments
  "UP", # pyupgrade
]
lint.ignore = ["E501", "B008"]
target-version = "py310"
```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/uv.lock
```lock
version = 1
revision = 2
requires-python = ">=3.10"

[[package]]
name = "anyio"
version = "4.11.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "exceptiongroup", marker = "python_full_version < '3.11'" },
    { name = "idna" },
    { name = "sniffio" },
    { name = "typing-extensions", marker = "python_full_version < '3.13'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/c6/78/7d432127c41b50bccba979505f272c16cbcadcc33645d5fa3a738110ae75/anyio-4.11.0.tar.gz", hash = "sha256:82a8d0b81e318cc5ce71a5f1f8b5c4e63619620b63141ef8c995fa0db95a57c4", size = 219094, upload-time = "2025-09-23T09:19:12.58Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/15/b3/9b1a8074496371342ec1e796a96f99c82c945a339cd81a8e73de28b4cf9e/anyio-4.11.0-py3-none-any.whl", hash = "sha256:0287e96f4d26d4149305414d4e3bc32f0dcd0862365a4bddea19d7a1ec38c4fc", size = 109097, upload-time = "2025-09-23T09:19:10.601Z" },
]

[[package]]
name = "backports-asyncio-runner"
version = "1.2.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/8e/ff/70dca7d7cb1cbc0edb2c6cc0c38b65cba36cccc491eca64cabd5fe7f8670/backports_asyncio_runner-1.2.0.tar.gz", hash = "sha256:a5aa7b2b7d8f8bfcaa2b57313f70792df84e32a2a746f585213373f900b42162", size = 69893, upload-time = "2025-07-02T02:27:15.685Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/a0/59/76ab57e3fe74484f48a53f8e337171b4a2349e506eabe136d7e01d059086/backports_asyncio_runner-1.2.0-py3-none-any.whl", hash = "sha256:0da0a936a8aeb554eccb426dc55af3ba63bcdc69fa1a600b5bb305413a4477b5", size = 12313, upload-time = "2025-07-02T02:27:14.263Z" },
]

[[package]]
name = "certifi"
version = "2025.8.3"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/dc/67/960ebe6bf230a96cda2e0abcf73af550ec4f090005363542f0765df162e0/certifi-2025.8.3.tar.gz", hash = "sha256:e564105f78ded564e3ae7c923924435e1daa7463faeab5bb932bc53ffae63407", size = 162386, upload-time = "2025-08-03T03:07:47.08Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/e5/48/1549795ba7742c948d2ad169c1c8cdbae65bc450d6cd753d124b17c8cd32/certifi-2025.8.3-py3-none-any.whl", hash = "sha256:f6c12493cfb1b06ba2ff328595af9350c65d6644968e5d3a2ffd78699af217a5", size = 161216, upload-time = "2025-08-03T03:07:45.777Z" },
]

[[package]]
name = "codespell"
version = "2.4.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/15/e0/709453393c0ea77d007d907dd436b3ee262e28b30995ea1aa36c6ffbccaf/codespell-2.4.1.tar.gz", hash = "sha256:299fcdcb09d23e81e35a671bbe746d5ad7e8385972e65dbb833a2eaac33c01e5", size = 344740, upload-time = "2025-01-28T18:52:39.411Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/20/01/b394922252051e97aab231d416c86da3d8a6d781eeadcdca1082867de64e/codespell-2.4.1-py3-none-any.whl", hash = "sha256:3dadafa67df7e4a3dbf51e0d7315061b80d265f9552ebd699b3dd6834b47e425", size = 344501, upload-time = "2025-01-28T18:52:37.057Z" },
]

[[package]]
name = "colorama"
version = "0.4.6"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/d8/53/6f443c9a4a8358a93a6792e2acffb9d9d5cb0a5cfd8802644b7b1c9a02e4/colorama-0.4.6.tar.gz", hash = "sha256:08695f5cb7ed6e0531a20572697297273c47b8cae5a63ffc6d6ed5c201be6e44", size = 27697, upload-time = "2022-10-25T02:36:22.414Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/d1/d6/3965ed04c63042e047cb6a3e6ed1a63a35087b6a609aa3a15ed8ac56c221/colorama-0.4.6-py2.py3-none-any.whl", hash = "sha256:4f1d9991f5acc0ca119f9d443620b77f9d6b33703e51011c16baf57afb285fc6", size = 25335, upload-time = "2022-10-25T02:36:20.889Z" },
]

[[package]]
name = "docopt"
version = "0.6.2"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/a2/55/8f8cab2afd404cf578136ef2cc5dfb50baa1761b68c9da1fb1e4eed343c9/docopt-0.6.2.tar.gz", hash = "sha256:49b3a825280bd66b3aa83585ef59c4a8c82f2c8a522dbe754a8bc8d08c85c491", size = 25901, upload-time = "2014-06-16T11:18:57.406Z" }

[[package]]
name = "exceptiongroup"
version = "1.3.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "typing-extensions", marker = "python_full_version < '3.13'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/0b/9f/a65090624ecf468cdca03533906e7c69ed7588582240cfe7cc9e770b50eb/exceptiongroup-1.3.0.tar.gz", hash = "sha256:b241f5885f560bc56a59ee63ca4c6a8bfa46ae4ad651af316d4e81817bb9fd88", size = 29749, upload-time = "2025-05-10T17:42:51.123Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/36/f4/c6e662dade71f56cd2f3735141b265c3c79293c109549c1e6933b0651ffc/exceptiongroup-1.3.0-py3-none-any.whl", hash = "sha256:4d111e6e0c13d0644cad6ddaa7ed0261a0b36971f6d23e7ec9b4b9097da78a10", size = 16674, upload-time = "2025-05-10T17:42:49.33Z" },
]

[[package]]
name = "h11"
version = "0.16.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/01/ee/02a2c011bdab74c6fb3c75474d40b3052059d95df7e73351460c8588d963/h11-0.16.0.tar.gz", hash = "sha256:4e35b956cf45792e4caa5885e69fba00bdbc6ffafbfa020300e549b208ee5ff1", size = 101250, upload-time = "2025-04-24T03:35:25.427Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/04/4b/29cac41a4d98d144bf5f6d33995617b185d14b22401f75ca86f384e87ff1/h11-0.16.0-py3-none-any.whl", hash = "sha256:63cf8bbe7522de3bf65932fda1d9c2772064ffb3dae62d55932da54b31cb6c86", size = 37515, upload-time = "2025-04-24T03:35:24.344Z" },
]

[[package]]
name = "httpcore"
