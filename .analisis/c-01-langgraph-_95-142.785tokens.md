
    class Handler:
        def handle(self, e: ValueError) -> str:
            return ""

    handle4 = Handler().handle

    def handle5(e: TypeError | ValueError | ToolException):
        return ""

    expected: tuple = (Exception,)
    actual = _infer_handled_types(handle)
    assert expected == actual

    expected = (Exception,)
    actual = _infer_handled_types(handle2)
    assert expected == actual

    expected = (ValueError, ToolException)
    actual = _infer_handled_types(handle3)
    assert expected == actual

    expected = (ValueError,)
    actual = _infer_handled_types(handle4)
    assert expected == actual

    expected = (TypeError, ValueError, ToolException)
    actual = _infer_handled_types(handle5)
    assert expected == actual

    with pytest.raises(ValueError):

        def handler(e: str):
            return ""

        _infer_handled_types(handler)

    with pytest.raises(ValueError):

        def handler(e: list[Exception]):
            return ""

        _infer_handled_types(handler)

    with pytest.raises(ValueError):

        def handler(e: str | int):
            return ""

        _infer_handled_types(handler)


@pytest.mark.parametrize("version", REACT_TOOL_CALL_VERSIONS)
def test_react_agent_with_structured_response(version: str) -> None:
    class WeatherResponse(BaseModel):
        temperature: float = Field(description="The temperature in fahrenheit")

    tool_calls = [[{"args": {}, "id": "1", "name": "get_weather"}], []]

    def get_weather():
        """Get the weather"""
        return "The weather is sunny and 75°F."

    expected_structured_response = WeatherResponse(temperature=75)
    model = FakeToolCallingModel(
        tool_calls=tool_calls, structured_response=expected_structured_response
    )
    for response_format in (WeatherResponse, ("Meow", WeatherResponse)):
        agent = create_react_agent(
            model,
            [get_weather],
            response_format=response_format,
            version=version,
        )
        response = agent.invoke({"messages": [HumanMessage("What's the weather?")]})
        assert response["structured_response"] == expected_structured_response
        assert len(response["messages"]) == 4
        assert response["messages"][-2].content == "The weather is sunny and 75°F."


class CustomState(AgentState):
    user_name: str


class CustomStatePydantic(AgentStatePydantic):
    user_name: str | None = None


@pytest.mark.parametrize("version", REACT_TOOL_CALL_VERSIONS)
@pytest.mark.parametrize("state_schema", [CustomState, CustomStatePydantic])
def test_react_agent_update_state(
    sync_checkpointer: BaseCheckpointSaver,
    version: Literal["v1", "v2"],
    state_schema: StateSchemaType,
) -> None:
    @dec_tool
    def get_user_name(tool_call_id: Annotated[str, InjectedToolCallId]):
        """Retrieve user name"""
        user_name = interrupt("Please provider user name:")
        return Command(
            update={
                "user_name": user_name,
                "messages": [
                    ToolMessage(
                        "Successfully retrieved user name", tool_call_id=tool_call_id
                    )
                ],
            }
        )

    if issubclass(state_schema, AgentStatePydantic):

        def prompt(state: CustomStatePydantic):
            user_name = state.user_name
            if user_name is None:
                return state.messages

            system_msg = f"User name is {user_name}"
            return [{"role": "system", "content": system_msg}] + state.messages
    else:

        def prompt(state: CustomState):
            user_name = state.get("user_name")
            if user_name is None:
                return state["messages"]

            system_msg = f"User name is {user_name}"
            return [{"role": "system", "content": system_msg}] + state["messages"]

    tool_calls = [[{"args": {}, "id": "1", "name": "get_user_name"}]]
    model = FakeToolCallingModel(tool_calls=tool_calls)
    agent = create_react_agent(
        model,
        [get_user_name],
        state_schema=state_schema,
        prompt=prompt,
        checkpointer=sync_checkpointer,
        version=version,
    )
    config = {"configurable": {"thread_id": "1"}}
    # Run until interrupted
    agent.invoke({"messages": [("user", "what's my name")]}, config)
    # supply the value for the interrupt
    response = agent.invoke(Command(resume="Archibald"), config)
    # confirm that the state was updated
    assert response["user_name"] == "Archibald"
    assert len(response["messages"]) == 4
    tool_message: ToolMessage = response["messages"][-2]
    assert tool_message.content == "Successfully retrieved user name"
    assert tool_message.tool_call_id == "1"
    assert tool_message.name == "get_user_name"


@pytest.mark.parametrize("version", REACT_TOOL_CALL_VERSIONS)
def test_react_agent_parallel_tool_calls(
    sync_checkpointer: BaseCheckpointSaver, version: str
) -> None:
    human_assistance_execution_count = 0

    @dec_tool
    def human_assistance(query: str) -> str:
        """Request assistance from a human."""
        nonlocal human_assistance_execution_count
        human_response = interrupt({"query": query})
        human_assistance_execution_count += 1
        return human_response["data"]

    get_weather_execution_count = 0

    @dec_tool
    def get_weather(location: str) -> str:
        """Use this tool to get the weather."""
        nonlocal get_weather_execution_count
        get_weather_execution_count += 1
        return "It's sunny!"

    tool_calls = [
        [
            {"args": {"location": "sf"}, "id": "1", "name": "get_weather"},
            {"args": {"query": "request help"}, "id": "2", "name": "human_assistance"},
        ],
        [],
    ]
    model = FakeToolCallingModel(tool_calls=tool_calls)
    agent = create_react_agent(
        model,
        [human_assistance, get_weather],
        checkpointer=sync_checkpointer,
        version=version,
    )
    config = {"configurable": {"thread_id": "1"}}
    query = "Get user assistance and also check the weather"
    message_types = []
    for event in agent.stream(
        {"messages": [("user", query)]}, config, stream_mode="values"
    ):
        if messages := event.get("messages"):
            message_types.append([m.type for m in messages])

    if version == "v1":
        assert message_types == [
            ["human"],
            ["human", "ai"],
        ]
    elif version == "v2":
        assert message_types == [
            ["human"],
            ["human", "ai"],
            ["human", "ai", "tool"],
        ]

    # Resume
    message_types = []
    for event in agent.stream(
        Command(resume={"data": "Hello"}), config, stream_mode="values"
    ):
        if messages := event.get("messages"):
            message_types.append([m.type for m in messages])

    assert message_types == [
        ["human", "ai"],
        ["human", "ai", "tool", "tool"],
        ["human", "ai", "tool", "tool", "ai"],
    ]

    if version == "v1":
        assert human_assistance_execution_count == 1
        assert get_weather_execution_count == 2
    elif version == "v2":
        assert human_assistance_execution_count == 1
        assert get_weather_execution_count == 1


class _InjectStateSchema(TypedDict):
    messages: list
    foo: str


class _InjectedStatePydanticSchema(BaseModelV1):
    messages: list
    foo: str


class _InjectedStatePydanticV2Schema(BaseModel):
    messages: list
    foo: str


@dataclasses.dataclass
class _InjectedStateDataclassSchema:
    messages: list
    foo: str


T = TypeVar("T")


@pytest.mark.parametrize(
    "schema_",
    [
        _InjectStateSchema,
        pytest.param(
            _InjectedStatePydanticSchema,
            marks=pytest.mark.skipif(
                sys.version_info >= (3, 14),
                reason="Pydantic v1 not supported in Python 3.14+",
            ),
        ),
        _InjectedStatePydanticV2Schema,
        _InjectedStateDataclassSchema,
    ],
)
def test_tool_node_inject_state(schema_: type[T]) -> None:
    def tool1(some_val: int, state: Annotated[T, InjectedState]) -> str:
        """Tool 1 docstring."""
        if isinstance(state, dict):
            return state["foo"]
        else:
            return getattr(state, "foo")

    def tool2(some_val: int, state: Annotated[T, InjectedState()]) -> str:
        """Tool 2 docstring."""
        if isinstance(state, dict):
            return state["foo"]
        else:
            return getattr(state, "foo")

    def tool3(
        some_val: int,
        foo: Annotated[str, InjectedState("foo")],
        msgs: Annotated[list[AnyMessage], InjectedState("messages")],
    ) -> str:
        """Tool 1 docstring."""
        return foo

    def tool4(
        some_val: int, msgs: Annotated[list[AnyMessage], InjectedState("messages")]
    ) -> str:
        """Tool 1 docstring."""
        return msgs[0].content

    node = ToolNode([tool1, tool2, tool3, tool4])
    for tool_name in ("tool1", "tool2", "tool3"):
        tool_call = {
            "name": tool_name,
            "args": {"some_val": 1},
            "id": "some 0",
            "type": "tool_call",
        }
        msg = AIMessage("hi?", tool_calls=[tool_call])
        result = node.invoke(
            schema_(**{"messages": [msg], "foo": "bar"}),
            config=_create_config_with_runtime(),
        )
        tool_message = result["messages"][-1]
        assert tool_message.content == "bar", f"Failed for tool={tool_name}"

    tool_call = {
        "name": "tool4",
        "args": {"some_val": 1},
        "id": "some 0",
        "type": "tool_call",
    }
    msg = AIMessage("hi?", tool_calls=[tool_call])
    result = node.invoke(
        schema_(**{"messages": [msg], "foo": ""}), config=_create_config_with_runtime()
    )
    tool_message = result["messages"][-1]
    assert tool_message.content == "hi?"

    result = node.invoke([msg], config=_create_config_with_runtime())
    tool_message = result[-1]
    assert tool_message.content == "hi?"


class AgentStateExtraKey(AgentState):
    foo: int


class AgentStateExtraKeyPydantic(AgentStatePydantic):
    foo: int


@pytest.mark.parametrize("version", REACT_TOOL_CALL_VERSIONS)
@pytest.mark.parametrize(
    "state_schema", [AgentStateExtraKey, AgentStateExtraKeyPydantic]
)
def test_create_react_agent_inject_vars(
    version: Literal["v1", "v2"], state_schema: StateSchemaType
) -> None:
    """Test that the agent can inject state and store into tool functions."""
    store = InMemoryStore()
    namespace = ("test",)
    store.put(namespace, "test_key", {"bar": 3})

    if issubclass(state_schema, AgentStatePydantic):

        def tool1(
            some_val: int,
            state: Annotated[AgentStateExtraKeyPydantic, InjectedState],
            store: Annotated[BaseStore, InjectedStore()],
        ) -> str:
            """Tool 1 docstring."""
            store_val = store.get(namespace, "test_key").value["bar"]
            return some_val + state.foo + store_val
    else:

        def tool1(
            some_val: int,
            state: Annotated[dict, InjectedState],
            store: Annotated[BaseStore, InjectedStore()],
        ) -> str:
            """Tool 1 docstring."""
            store_val = store.get(namespace, "test_key").value["bar"]
            return some_val + state["foo"] + store_val

    tool_call = {
        "name": "tool1",
        "args": {"some_val": 1},
        "id": "some 0",
        "type": "tool_call",
    }
    model = FakeToolCallingModel(tool_calls=[[tool_call], []])
    agent = create_react_agent(
        model,
        ToolNode([tool1], handle_tool_errors=False),
        state_schema=state_schema,
        store=store,
        version=version,
    )
    result = agent.invoke({"messages": [{"role": "user", "content": "hi"}], "foo": 2})
    assert result["messages"] == [
        _AnyIdHumanMessage(content="hi"),
        AIMessage(content="hi", tool_calls=[tool_call], id="0"),
        _AnyIdToolMessage(content="6", name="tool1", tool_call_id="some 0"),
        AIMessage("hi-hi-6", id="1"),
    ]
    assert result["foo"] == 2


def test_tool_node_inject_store() -> None:
    store = InMemoryStore()
    namespace = ("test",)

    def tool1(some_val: int, store: Annotated[BaseStore, InjectedStore()]) -> str:
        """Tool 1 docstring."""
        store_val = store.get(namespace, "test_key").value["foo"]
        return f"Some val: {some_val}, store val: {store_val}"

    def tool2(some_val: int, store: Annotated[BaseStore, InjectedStore()]) -> str:
        """Tool 2 docstring."""
        store_val = store.get(namespace, "test_key").value["foo"]
        return f"Some val: {some_val}, store val: {store_val}"

    def tool3(
        some_val: int,
        bar: Annotated[str, InjectedState("bar")],
        store: Annotated[BaseStore, InjectedStore()],
    ) -> str:
        """Tool 3 docstring."""
        store_val = store.get(namespace, "test_key").value["foo"]
        return f"Some val: {some_val}, store val: {store_val}, state val: {bar}"

    node = ToolNode([tool1, tool2, tool3], handle_tool_errors=True)
    store.put(namespace, "test_key", {"foo": "bar"})

    class State(MessagesState):
        bar: str

    builder = StateGraph(State)
    builder.add_node("tools", node)
    builder.add_edge(START, "tools")
    graph = builder.compile(store=store)

    for tool_name in ("tool1", "tool2"):
        tool_call = {
            "name": tool_name,
            "args": {"some_val": 1},
            "id": "some 0",
            "type": "tool_call",
        }
        msg = AIMessage("hi?", tool_calls=[tool_call])
        node_result = node.invoke(
            {"messages": [msg]}, config=_create_config_with_runtime(store=store)
        )
        graph_result = graph.invoke({"messages": [msg]})
        for result in (node_result, graph_result):
            result["messages"][-1]
            tool_message = result["messages"][-1]
            assert tool_message.content == "Some val: 1, store val: bar", (
                f"Failed for tool={tool_name}"
            )

    tool_call = {
        "name": "tool3",
        "args": {"some_val": 1},
        "id": "some 0",
        "type": "tool_call",
    }
    msg = AIMessage("hi?", tool_calls=[tool_call])
    node_result = node.invoke(
        {"messages": [msg], "bar": "baz"},
        config=_create_config_with_runtime(store=store),
    )
    graph_result = graph.invoke({"messages": [msg], "bar": "baz"})
    for result in (node_result, graph_result):
        result["messages"][-1]
        tool_message = result["messages"][-1]
        assert tool_message.content == "Some val: 1, store val: bar, state val: baz", (
            f"Failed for tool={tool_name}"
        )

    # test injected store without passing store to compiled graph
    failing_graph = builder.compile()
    with pytest.raises(ValueError):
        failing_graph.invoke({"messages": [msg], "bar": "baz"})


def test_tool_node_ensure_utf8() -> None:
    @dec_tool
    def get_day_list(days: list[str]) -> list[str]:
        """choose days"""
        return days

    data = ["星期一", "水曜日", "목요일", "Friday"]
    tools = [get_day_list]
    tool_calls = [ToolCall(name=get_day_list.name, args={"days": data}, id="test_id")]
    outputs: list[ToolMessage] = ToolNode(tools).invoke(
        [AIMessage(content="", tool_calls=tool_calls)],
        config=_create_config_with_runtime(),
    )
    assert outputs[0].content == json.dumps(data, ensure_ascii=False)


def test_tool_node_messages_key() -> None:
    @dec_tool
    def add(a: int, b: int):
        """Adds a and b."""
        return a + b

    model = FakeToolCallingModel(
        tool_calls=[[ToolCall(name=add.name, args={"a": 1, "b": 2}, id="test_id")]]
    )

    class State(TypedDict):
        subgraph_messages: Annotated[list[AnyMessage], add_messages]

    def call_model(state: State):
        response = model.invoke(state["subgraph_messages"])
        model.tool_calls = []
        return {"subgraph_messages": response}

    builder = StateGraph(State)
    builder.add_node("agent", call_model)
    builder.add_node("tools", ToolNode([add], messages_key="subgraph_messages"))
    builder.add_conditional_edges(
        "agent", partial(tools_condition, messages_key="subgraph_messages")
    )
    builder.add_edge(START, "agent")
    builder.add_edge("tools", "agent")

    graph = builder.compile()
    result = graph.invoke({"subgraph_messages": [HumanMessage(content="hi")]})
    assert result["subgraph_messages"] == [
        _AnyIdHumanMessage(content="hi"),
        AIMessage(
            content="hi",
            id="0",
            tool_calls=[ToolCall(name=add.name, args={"a": 1, "b": 2}, id="test_id")],
        ),
        _AnyIdToolMessage(content="3", name=add.name, tool_call_id="test_id"),
        AIMessage(content="hi-hi-3", id="1"),
    ]


@pytest.mark.parametrize("version", REACT_TOOL_CALL_VERSIONS)
async def test_return_direct(version: str) -> None:
    @dec_tool(return_direct=True)
    def tool_return_direct(input: str) -> str:
        """A tool that returns directly."""
        return f"Direct result: {input}"

    @dec_tool
    def tool_normal(input: str) -> str:
        """A normal tool."""
        return f"Normal result: {input}"

    first_tool_call = [
        ToolCall(
            name="tool_return_direct",
            args={"input": "Test direct"},
            id="1",
        ),
    ]
    expected_ai = AIMessage(
        content="Test direct",
        id="0",
        tool_calls=first_tool_call,
    )
    model = FakeToolCallingModel(tool_calls=[first_tool_call, []])
    agent = create_react_agent(
        model,
        [tool_return_direct, tool_normal],
        version=version,
    )

    # Test direct return for tool_return_direct
    result = agent.invoke(
        {"messages": [HumanMessage(content="Test direct", id="hum0")]}
    )
    assert result["messages"] == [
        HumanMessage(content="Test direct", id="hum0"),
        expected_ai,
        ToolMessage(
            content="Direct result: Test direct",
            name="tool_return_direct",
            tool_call_id="1",
            id=result["messages"][2].id,
        ),
    ]
    second_tool_call = [
        ToolCall(
            name="tool_normal",
            args={"input": "Test normal"},
            id="2",
        ),
    ]
    model = FakeToolCallingModel(tool_calls=[second_tool_call, []])
    agent = create_react_agent(
        model, [tool_return_direct, tool_normal], version=version
    )
    result = agent.invoke(
        {"messages": [HumanMessage(content="Test normal", id="hum1")]}
    )
    assert result["messages"] == [
        HumanMessage(content="Test normal", id="hum1"),
        AIMessage(content="Test normal", id="0", tool_calls=second_tool_call),
        ToolMessage(
            content="Normal result: Test normal",
            name="tool_normal",
            tool_call_id="2",
            id=result["messages"][2].id,
        ),
        AIMessage(content="Test normal-Test normal-Normal result: Test normal", id="1"),
    ]

    both_tool_calls = [
        ToolCall(
            name="tool_return_direct",
            args={"input": "Test both direct"},
            id="3",
        ),
        ToolCall(
            name="tool_normal",
            args={"input": "Test both normal"},
            id="4",
        ),
    ]
    model = FakeToolCallingModel(tool_calls=[both_tool_calls, []])
    agent = create_react_agent(
        model, [tool_return_direct, tool_normal], version=version
    )
    result = agent.invoke({"messages": [HumanMessage(content="Test both", id="hum2")]})
    assert result["messages"] == [
        HumanMessage(content="Test both", id="hum2"),
        AIMessage(content="Test both", id="0", tool_calls=both_tool_calls),
        ToolMessage(
            content="Direct result: Test both direct",
            name="tool_return_direct",
            tool_call_id="3",
            id=result["messages"][2].id,
        ),
        ToolMessage(
            content="Normal result: Test both normal",
            name="tool_normal",
            tool_call_id="4",
            id=result["messages"][3].id,
        ),
    ]


def test__get_state_args() -> None:
    class Schema1(BaseModel):
        a: Annotated[str, InjectedState]

    class Schema2(Schema1):
        b: Annotated[int, InjectedState("bar")]

    @dec_tool(args_schema=Schema2)
    def foo(a: str, b: int) -> float:
        """return"""
        return 0.0

    assert _get_state_args(foo) == {"a": None, "b": "bar"}


def test_inspect_react() -> None:
    model = FakeToolCallingModel(tool_calls=[])
    agent = create_react_agent(model, [])
    inspect.getclosurevars(agent.nodes["agent"].bound.func)


@pytest.mark.parametrize("version", REACT_TOOL_CALL_VERSIONS)
def test_react_with_subgraph_tools(
    sync_checkpointer: BaseCheckpointSaver, version: Literal["v1", "v2"]
) -> None:
    class State(TypedDict):
        a: int
        b: int

    class Output(TypedDict):
        result: int

    # Define the subgraphs
    def add(state):
        return {"result": state["a"] + state["b"]}

    add_subgraph = (
        StateGraph(State, output_schema=Output)
        .add_node(add)
        .add_edge(START, "add")
        .compile()
    )

    def multiply(state):
        return {"result": state["a"] * state["b"]}

    multiply_subgraph = (
        StateGraph(State, output_schema=Output)
        .add_node(multiply)
        .add_edge(START, "multiply")
        .compile()
    )

    multiply_subgraph.invoke({"a": 2, "b": 3})

    # Add subgraphs as tools

    def addition(a: int, b: int):
        """Add two numbers"""
        return add_subgraph.invoke({"a": a, "b": b})["result"]

    def multiplication(a: int, b: int):
        """Multiply two numbers"""
        return multiply_subgraph.invoke({"a": a, "b": b})["result"]

    model = FakeToolCallingModel(
        tool_calls=[
            [
                {"args": {"a": 2, "b": 3}, "id": "1", "name": "addition"},
                {"args": {"a": 2, "b": 3}, "id": "2", "name": "multiplication"},
            ],
            [],
        ]
    )
    tool_node = ToolNode([addition, multiplication], handle_tool_errors=False)
    agent = create_react_agent(
        model,
        tool_node,
        checkpointer=sync_checkpointer,
        version=version,
    )
    result = agent.invoke(
        {"messages": [HumanMessage(content="What's 2 + 3 and 2 * 3?")]},
        config={"configurable": {"thread_id": "1"}},
    )
    assert result["messages"] == [
        _AnyIdHumanMessage(content="What's 2 + 3 and 2 * 3?"),
        AIMessage(
            content="What's 2 + 3 and 2 * 3?",
            id="0",
            tool_calls=[
                ToolCall(name="addition", args={"a": 2, "b": 3}, id="1"),
                ToolCall(name="multiplication", args={"a": 2, "b": 3}, id="2"),
            ],
        ),
        ToolMessage(
            content="5", name="addition", tool_call_id="1", id=result["messages"][2].id
        ),
        ToolMessage(
            content="6",
            name="multiplication",
            tool_call_id="2",
            id=result["messages"][3].id,
        ),
        AIMessage(
            content="What's 2 + 3 and 2 * 3?-What's 2 + 3 and 2 * 3?-5-6", id="1"
        ),
    ]


def test_tool_node_stream_writer() -> None:
    @dec_tool
    def streaming_tool(x: int) -> str:
        """Do something with writer."""
        my_writer = get_stream_writer()
        for value in ["foo", "bar", "baz"]:
            my_writer({"custom_tool_value": value})

        return x

    tool_node = ToolNode([streaming_tool])
    graph = (
        StateGraph(MessagesState)
        .add_node("tools", tool_node)
        .add_edge(START, "tools")
        .compile()
    )

    tool_call = {
        "name": "streaming_tool",
        "args": {"x": 1},
        "id": "1",
        "type": "tool_call",
    }
    inputs = {
        "messages": [AIMessage("", tool_calls=[tool_call])],
    }

    assert list(graph.stream(inputs, stream_mode="custom")) == [
        {"custom_tool_value": "foo"},
        {"custom_tool_value": "bar"},
        {"custom_tool_value": "baz"},
    ]
    assert list(graph.stream(inputs, stream_mode=["custom", "updates"])) == [
        ("custom", {"custom_tool_value": "foo"}),
        ("custom", {"custom_tool_value": "bar"}),
        ("custom", {"custom_tool_value": "baz"}),
        (
            "updates",
            {
                "tools": {
                    "messages": [
                        _AnyIdToolMessage(
                            content="1",
                            name="streaming_tool",
                            tool_call_id="1",
                        ),
                    ],
                },
            },
        ),
    ]


@pytest.mark.parametrize("version", REACT_TOOL_CALL_VERSIONS)
def test_react_agent_subgraph_streaming_sync(version: Literal["v1", "v2"]) -> None:
    """Test React agent streaming when used as a subgraph node sync version"""

    @dec_tool
    def get_weather(city: str) -> str:
        """Get the weather of a city."""
        return f"The weather of {city} is sunny."

    # Create a React agent
    model = FakeToolCallingModel(
        tool_calls=[
            [{"args": {"city": "Tokyo"}, "id": "1", "name": "get_weather"}],
            [],
        ]
    )

    agent = create_react_agent(
        model,
        tools=[get_weather],
        prompt="You are a helpful travel assistant.",
        version=version,
    )

    # Create a subgraph that uses the React agent as a node
    def react_agent_node(state: MessagesState, config: RunnableConfig) -> MessagesState:
        """Node that runs the React agent and collects streaming output."""
        collected_content = ""

        # Stream the agent output and collect content
        for msg_chunk, msg_metadata in agent.stream(
            {"messages": [("user", state["messages"][-1].content)]},
            config,
            stream_mode="messages",
        ):
            if hasattr(msg_chunk, "content") and msg_chunk.content:
                collected_content += msg_chunk.content

        return {"messages": [("assistant", collected_content)]}

    # Create the main workflow with the React agent as a subgraph node
    workflow = StateGraph(MessagesState)
    workflow.add_node("react_agent", react_agent_node)
    workflow.add_edge(START, "react_agent")
    workflow.add_edge("react_agent", "__end__")
    compiled_workflow = workflow.compile()

    # Test the streaming functionality
    result = compiled_workflow.invoke(
        {"messages": [("user", "What is the weather in Tokyo?")]}
    )

    # Verify the result contains expected structure
    assert len(result["messages"]) == 2
    assert result["messages"][0].content == "What is the weather in Tokyo?"
    assert "assistant" in str(result["messages"][1])

    # Test streaming with subgraphs = True
    result = compiled_workflow.invoke(
        {"messages": [("user", "What is the weather in Tokyo?")]},
        subgraphs=True,
    )
    assert len(result["messages"]) == 2

    events = []
    for event in compiled_workflow.stream(
        {"messages": [("user", "What is the weather in Tokyo?")]},
        stream_mode="messages",
        subgraphs=False,
    ):
        events.append(event)

    assert len(events) == 0

    events = []
    for event in compiled_workflow.stream(
        {"messages": [("user", "What is the weather in Tokyo?")]},
        stream_mode="messages",
        subgraphs=True,
    ):
        events.append(event)

    assert len(events) == 3
    namespace, (msg, metadata) = events[0]
    # FakeToolCallingModel returns a single AIMessage with tool calls
    # The content of the AIMessage reflects the input message
    assert msg.content.startswith("You are a helpful travel assistant")
    namespace, (msg, metadata) = events[1]  # ToolMessage
    assert msg.content.startswith("The weather of Tokyo is sunny.")


@pytest.mark.parametrize("version", REACT_TOOL_CALL_VERSIONS)
async def test_react_agent_subgraph_streaming(version: Literal["v1", "v2"]) -> None:
    """Test React agent streaming when used as a subgraph node."""

    @dec_tool
    def get_weather(city: str) -> str:
        """Get the weather of a city."""
        return f"The weather of {city} is sunny."

    # Create a React agent
    model = FakeToolCallingModel(
        tool_calls=[
            [{"args": {"city": "Tokyo"}, "id": "1", "name": "get_weather"}],
            [],
        ]
    )

    agent = create_react_agent(
        model,
        tools=[get_weather],
        prompt="You are a helpful travel assistant.",
        version=version,
    )

    # Create a subgraph that uses the React agent as a node
    async def react_agent_node(
        state: MessagesState, config: RunnableConfig
    ) -> MessagesState:
        """Node that runs the React agent and collects streaming output."""
        collected_content = ""

        # Stream the agent output and collect content
        async for msg_chunk, msg_metadata in agent.astream(
            {"messages": [("user", state["messages"][-1].content)]},
            config,
            stream_mode="messages",
        ):
            if hasattr(msg_chunk, "content") and msg_chunk.content:
                collected_content += msg_chunk.content

        return {"messages": [("assistant", collected_content)]}

    # Create the main workflow with the React agent as a subgraph node
    workflow = StateGraph(MessagesState)
    workflow.add_node("react_agent", react_agent_node)
    workflow.add_edge(START, "react_agent")
    workflow.add_edge("react_agent", "__end__")
    compiled_workflow = workflow.compile()

    # Test the streaming functionality
    result = await compiled_workflow.ainvoke(
        {"messages": [("user", "What is the weather in Tokyo?")]}
    )

    # Verify the result contains expected structure
    assert len(result["messages"]) == 2
    assert result["messages"][0].content == "What is the weather in Tokyo?"
    assert "assistant" in str(result["messages"][1])

    # Test streaming with subgraphs = True
    result = await compiled_workflow.ainvoke(
        {"messages": [("user", "What is the weather in Tokyo?")]},
        subgraphs=True,
    )
    assert len(result["messages"]) == 2

    events = []
    async for event in compiled_workflow.astream(
        {"messages": [("user", "What is the weather in Tokyo?")]},
        stream_mode="messages",
        subgraphs=False,
    ):
        events.append(event)

    assert len(events) == 0

    events = []
    async for event in compiled_workflow.astream(
        {"messages": [("user", "What is the weather in Tokyo?")]},
        stream_mode="messages",
        subgraphs=True,
    ):
        events.append(event)

    assert len(events) == 3
    namespace, (msg, metadata) = events[0]
    # FakeToolCallingModel returns a single AIMessage with tool calls
    # The content of the AIMessage reflects the input message
    assert msg.content.startswith("You are a helpful travel assistant")
    namespace, (msg, metadata) = events[1]  # ToolMessage
    assert msg.content.startswith("The weather of Tokyo is sunny.")


@pytest.mark.parametrize("version", REACT_TOOL_CALL_VERSIONS)
def test_tool_node_node_interrupt(
    sync_checkpointer: BaseCheckpointSaver, version: str
) -> None:
    def tool_normal(some_val: int) -> str:
        """Tool docstring."""
        return "normal"

    def tool_interrupt(some_val: int) -> str:
        """Tool docstring."""
        foo = interrupt("provide value for foo")
        return foo

    # test inside react agent
    model = FakeToolCallingModel(
        tool_calls=[
            [
                ToolCall(name="tool_interrupt", args={"some_val": 0}, id="1"),
                ToolCall(name="tool_normal", args={"some_val": 1}, id="2"),
            ],
            [],
        ]
    )
    config = {"configurable": {"thread_id": "1"}}
    agent = create_react_agent(
        model,
        [tool_interrupt, tool_normal],
        checkpointer=sync_checkpointer,
        version=version,
    )
    result = agent.invoke({"messages": [HumanMessage("hi?")]}, config)
    expected_messages = [
        _AnyIdHumanMessage(content="hi?"),
        AIMessage(
            content="hi?",
            id="0",
            tool_calls=[
                {
                    "name": "tool_interrupt",
                    "args": {"some_val": 0},
                    "id": "1",
                    "type": "tool_call",
                },
                {
                    "name": "tool_normal",
                    "args": {"some_val": 1},
                    "id": "2",
                    "type": "tool_call",
                },
            ],
        ),
        _AnyIdToolMessage(content="normal", name="tool_normal", tool_call_id="2"),
    ]
    if version == "v1":
        # Interrupt blocks second tool result
        assert result["messages"] == expected_messages[:-1]
    elif version == "v2":
        assert result["messages"] == expected_messages

    state = agent.get_state(config)
    assert state.next == ("tools",)
    task = state.tasks[0]
    assert task.name == "tools"
    assert task.interrupts == (
        Interrupt(
            value="provide value for foo",
            id=AnyStr(),
        ),
    )


@pytest.mark.parametrize("tool_style", ["openai", "anthropic"])
def test_should_bind_tools(tool_style: str) -> None:
    @dec_tool
    def some_tool(some_val: int) -> str:
        """Tool docstring."""
        return "meow"

    @dec_tool
    def some_other_tool(some_val: int) -> str:
        """Tool docstring."""
        return "meow"

    model = FakeToolCallingModel(tool_style=tool_style)
    # should bind when a regular model
    assert _should_bind_tools(model, [])
    assert _should_bind_tools(model, [some_tool])

    # should bind when a seq
    seq = model | RunnableLambda(lambda message: message)
    assert _should_bind_tools(seq, [])
    assert _should_bind_tools(seq, [some_tool])

    # should not bind when a model with tools
    assert not _should_bind_tools(model.bind_tools([some_tool]), [some_tool])
    # should not bind when a seq with tools
    seq_with_tools = model.bind_tools([some_tool]) | RunnableLambda(
        lambda message: message
    )
    assert not _should_bind_tools(seq_with_tools, [some_tool])

    # should raise on invalid inputs
    with pytest.raises(ValueError):
        _should_bind_tools(model.bind_tools([some_tool]), [])
    with pytest.raises(ValueError):
        _should_bind_tools(model.bind_tools([some_tool]), [some_other_tool])
    with pytest.raises(ValueError):
        _should_bind_tools(model.bind_tools([some_tool]), [some_tool, some_other_tool])


def test_get_model() -> None:
    model = FakeToolCallingModel(tool_calls=[])
    assert _get_model(model) == model

    @dec_tool
    def some_tool(some_val: int) -> str:
        """Tool docstring."""
        return "meow"

    model_with_tools = model.bind_tools([some_tool])
    assert _get_model(model_with_tools) == model

    seq = model | RunnableLambda(lambda message: message)
    assert _get_model(seq) == model

    seq_with_tools = model.bind_tools([some_tool]) | RunnableLambda(
        lambda message: message
    )
    assert _get_model(seq_with_tools) == model

    with pytest.raises(TypeError):
        _get_model(RunnableLambda(lambda message: message))


@pytest.mark.parametrize("version", REACT_TOOL_CALL_VERSIONS)
def test_dynamic_model_basic(version: str) -> None:
    """Test basic dynamic model functionality."""

    def dynamic_model(state, runtime: Runtime):
        # Return different models based on state
        if "urgent" in state["messages"][-1].content:
            return FakeToolCallingModel(tool_calls=[])
        else:
            return FakeToolCallingModel(tool_calls=[])

    agent = create_react_agent(dynamic_model, [], version=version)

    result = agent.invoke({"messages": [HumanMessage("hello")]})
    assert len(result["messages"]) == 2
    assert result["messages"][-1].content == "hello"

    result = agent.invoke({"messages": [HumanMessage("urgent help")]})
    assert len(result["messages"]) == 2
    assert result["messages"][-1].content == "urgent help"


@pytest.mark.parametrize("version", REACT_TOOL_CALL_VERSIONS)
def test_dynamic_model_with_tools(version: Literal["v1", "v2"]) -> None:
    """Test dynamic model with tool calling."""

    @dec_tool
    def basic_tool(x: int) -> str:
        """Basic tool."""
        return f"basic: {x}"

    @dec_tool
    def advanced_tool(x: int) -> str:
        """Advanced tool."""
        return f"advanced: {x}"

    def dynamic_model(state: dict, runtime: Runtime) -> BaseChatModel:
        # Return model with different behaviors based on message content
        if "advanced" in state["messages"][-1].content:
            return FakeToolCallingModel(
                tool_calls=[
                    [{"args": {"x": 1}, "id": "1", "name": "advanced_tool"}],
                    [],
                ]
            )
        else:
            return FakeToolCallingModel(
                tool_calls=[[{"args": {"x": 1}, "id": "1", "name": "basic_tool"}], []]
            )

    agent = create_react_agent(
        dynamic_model, [basic_tool, advanced_tool], version=version
    )

    # Test basic tool usage
    result = agent.invoke({"messages": [HumanMessage("basic request")]})
    assert len(result["messages"]) == 3
    tool_message = result["messages"][-1]
    assert tool_message.content == "basic: 1"
    assert tool_message.name == "basic_tool"

    # Test advanced tool usage
    result = agent.invoke({"messages": [HumanMessage("advanced request")]})
    assert len(result["messages"]) == 3
    tool_message = result["messages"][-1]
    assert tool_message.content == "advanced: 1"
    assert tool_message.name == "advanced_tool"


@dataclasses.dataclass
class Context:
    user_id: str


@pytest.mark.parametrize("version", REACT_TOOL_CALL_VERSIONS)
def test_dynamic_model_with_context(version: str) -> None:
    """Test dynamic model using config parameters."""

    def dynamic_model(state, runtime: Runtime[Context]):
        # Use context to determine model behavior
        user_id = runtime.context.user_id
        if user_id == "user_premium":
            return FakeToolCallingModel(tool_calls=[])
        else:
            return FakeToolCallingModel(tool_calls=[])

    agent = create_react_agent(
        dynamic_model, [], context_schema=Context, version=version
    )

    # Test with basic user
    result = agent.invoke(
        {"messages": [HumanMessage("hello")]},
        context=Context(user_id="user_basic"),
    )
    assert len(result["messages"]) == 2

    # Test with premium user
    result = agent.invoke(
        {"messages": [HumanMessage("hello")]},
        context=Context(user_id="user_premium"),
    )
    assert len(result["messages"]) == 2


@pytest.mark.parametrize("version", REACT_TOOL_CALL_VERSIONS)
def test_dynamic_model_with_state_schema(version: Literal["v1", "v2"]) -> None:
    """Test dynamic model with custom state schema."""

    class CustomDynamicState(AgentState):
        model_preference: str = "default"

    def dynamic_model(state: CustomDynamicState, runtime: Runtime) -> BaseChatModel:
        # Use custom state field to determine model
        if state.get("model_preference") == "advanced":
            return FakeToolCallingModel(tool_calls=[])
        else:
            return FakeToolCallingModel(tool_calls=[])

    agent = create_react_agent(
        dynamic_model, [], state_schema=CustomDynamicState, version=version
    )

    result = agent.invoke(
        {"messages": [HumanMessage("hello")], "model_preference": "advanced"}
    )
    assert len(result["messages"]) == 2
    assert result["model_preference"] == "advanced"


@pytest.mark.parametrize("version", REACT_TOOL_CALL_VERSIONS)
def test_dynamic_model_with_prompt(version: Literal["v1", "v2"]) -> None:
    """Test dynamic model with different prompt types."""

    def dynamic_model(state: AgentState, runtime: Runtime) -> BaseChatModel:
        return FakeToolCallingModel(tool_calls=[])

    # Test with string prompt
    agent = create_react_agent(dynamic_model, [], prompt="system_msg", version=version)
    result = agent.invoke({"messages": [HumanMessage("human_msg")]})
    assert result["messages"][-1].content == "system_msg-human_msg"

    # Test with callable prompt
    def dynamic_prompt(state: AgentState) -> list[MessageLikeRepresentation]:
        """Generate a dynamic system message based on state."""
        return [{"role": "system", "content": "system_msg"}] + list(state["messages"])

    agent = create_react_agent(
        dynamic_model, [], prompt=dynamic_prompt, version=version
    )
    result = agent.invoke({"messages": [HumanMessage("human_msg")]})
    assert result["messages"][-1].content == "system_msg-human_msg"


async def test_dynamic_model_async() -> None:
    """Test dynamic model with async operations."""

    def dynamic_model(state: AgentState, runtime: Runtime) -> BaseChatModel:
        return FakeToolCallingModel(tool_calls=[])

    agent = create_react_agent(dynamic_model, [])

    result = await agent.ainvoke({"messages": [HumanMessage("hello async")]})
    assert len(result["messages"]) == 2
    assert result["messages"][-1].content == "hello async"


@pytest.mark.parametrize("version", REACT_TOOL_CALL_VERSIONS)
def test_dynamic_model_with_structured_response(version: str) -> None:
    """Test dynamic model with structured response format."""

    class TestResponse(BaseModel):
        message: str
        confidence: float

    def dynamic_model(state, runtime: Runtime):
        expected_response = TestResponse(message="dynamic response", confidence=0.9)
        return FakeToolCallingModel(
            tool_calls=[], structured_response=expected_response
        )

    agent = create_react_agent(
        dynamic_model, [], response_format=TestResponse, version=version
    )

    result = agent.invoke({"messages": [HumanMessage("hello")]})
    assert "structured_response" in result
    assert result["structured_response"].message == "dynamic response"
    assert result["structured_response"].confidence == 0.9


def test_dynamic_model_with_checkpointer(sync_checkpointer):
    """Test dynamic model with checkpointer."""
    call_count = 0

    def dynamic_model(state: AgentState, runtime: Runtime) -> BaseChatModel:
        nonlocal call_count
        call_count += 1
        return FakeToolCallingModel(
            tool_calls=[],
            # Incrementing the call count as it is used to assign an id
            # to the AIMessage.
            # The default reducer semantics are to overwrite an existing message
            # with the new one if the id matches.
            index=call_count,
        )

    agent = create_react_agent(dynamic_model, [], checkpointer=sync_checkpointer)
    config = {"configurable": {"thread_id": "test_dynamic"}}

    # First call
    result1 = agent.invoke({"messages": [HumanMessage("hello")]}, config)
    assert len(result1["messages"]) == 2  # Human + AI message

    # Second call - should load from checkpoint
    result2 = agent.invoke({"messages": [HumanMessage("world")]}, config)
    assert len(result2["messages"]) == 4

    # Dynamic model should be called each time
    assert call_count >= 2


@pytest.mark.parametrize("version", REACT_TOOL_CALL_VERSIONS)
def test_dynamic_model_state_dependent_tools(version: Literal["v1", "v2"]) -> None:
    """Test dynamic model that changes available tools based on state."""

    @dec_tool
    def tool_a(x: int) -> str:
        """Tool A."""
        return f"A: {x}"

    @dec_tool
    def tool_b(x: int) -> str:
        """Tool B."""
        return f"B: {x}"

    def dynamic_model(state, runtime: Runtime):
        # Switch tools based on message history
        if any("use_b" in msg.content for msg in state["messages"]):
            return FakeToolCallingModel(
                tool_calls=[[{"args": {"x": 2}, "id": "1", "name": "tool_b"}], []]
            )
        else:
            return FakeToolCallingModel(
                tool_calls=[[{"args": {"x": 1}, "id": "1", "name": "tool_a"}], []]
            )

    agent = create_react_agent(dynamic_model, [tool_a, tool_b], version=version)

    # Ask to use tool B
    result = agent.invoke({"messages": [HumanMessage("use_b please")]})
    last_message = result["messages"][-1]
    assert isinstance(last_message, ToolMessage)
    assert last_message.content == "B: 2"

    # Ask to use tool A
    result = agent.invoke({"messages": [HumanMessage("hello")]})
    last_message = result["messages"][-1]
    assert isinstance(last_message, ToolMessage)
    assert last_message.content == "A: 1"


@pytest.mark.parametrize("version", REACT_TOOL_CALL_VERSIONS)
def test_dynamic_model_error_handling(version: Literal["v1", "v2"]) -> None:
    """Test error handling in dynamic model."""

    def failing_dynamic_model(state, runtime: Runtime):
        if "fail" in state["messages"][-1].content:
            raise ValueError("Dynamic model failed")
        return FakeToolCallingModel(tool_calls=[])

    agent = create_react_agent(failing_dynamic_model, [], version=version)

    # Normal operation should work
    result = agent.invoke({"messages": [HumanMessage("hello")]})
    assert len(result["messages"]) == 2

    # Should propagate the error
    with pytest.raises(ValueError, match="Dynamic model failed"):
        agent.invoke({"messages": [HumanMessage("fail now")]})


def test_dynamic_model_vs_static_model_behavior():
    """Test that dynamic and static models produce equivalent results when configured the same."""
    # Static model
    static_model = FakeToolCallingModel(tool_calls=[])
    static_agent = create_react_agent(static_model, [])

    # Dynamic model returning the same model
    def dynamic_model(state, runtime: Runtime):
        return FakeToolCallingModel(tool_calls=[])

    dynamic_agent = create_react_agent(dynamic_model, [])

    input_msg = {"messages": [HumanMessage("test message")]}

    static_result = static_agent.invoke(input_msg)
    dynamic_result = dynamic_agent.invoke(input_msg)

    # Results should be equivalent (content-wise, IDs may differ)
    assert len(static_result["messages"]) == len(dynamic_result["messages"])
    assert static_result["messages"][0].content == dynamic_result["messages"][0].content
    assert static_result["messages"][1].content == dynamic_result["messages"][1].content


def test_dynamic_model_receives_correct_state():
    """Test that the dynamic model function receives the correct state, not the model input."""
    received_states = []

    class CustomAgentState(AgentState):
        custom_field: str

    def dynamic_model(state, runtime: Runtime) -> BaseChatModel:
        # Capture the state that's passed to the dynamic model function
        received_states.append(state)
        return FakeToolCallingModel(tool_calls=[])

    agent = create_react_agent(dynamic_model, [], state_schema=CustomAgentState)

    # Test with initial state
    input_state = {"messages": [HumanMessage("hello")], "custom_field": "test_value"}
    agent.invoke(input_state)

    # The dynamic model function should receive the original state, not the processed model input
    assert len(received_states) == 1
    received_state = received_states[0]

    # Should have the custom field from original state
    assert "custom_field" in received_state
    assert received_state["custom_field"] == "test_value"

    # Should have the original messages
    assert len(received_state["messages"]) == 1
    assert received_state["messages"][0].content == "hello"


async def test_dynamic_model_receives_correct_state_async():
    """Test that the async dynamic model function receives the correct state, not the model input."""
    received_states = []

    class CustomAgentStateAsync(AgentState):
        custom_field: str

    def dynamic_model(state, runtime: Runtime):
        # Capture the state that's passed to the dynamic model function
        received_states.append(state)
        return FakeToolCallingModel(tool_calls=[])

    agent = create_react_agent(dynamic_model, [], state_schema=CustomAgentStateAsync)

    # Test with initial state
    input_state = {
        "messages": [HumanMessage("hello async")],
        "custom_field": "test_value_async",
    }
    await agent.ainvoke(input_state)

    # The dynamic model function should receive the original state, not the processed model input
    assert len(received_states) == 1
    received_state = received_states[0]

    # Should have the custom field from original state
    assert "custom_field" in received_state
    assert received_state["custom_field"] == "test_value_async"

    # Should have the original messages
    assert len(received_state["messages"]) == 1
    assert received_state["messages"][0].content == "hello async"


def test_pre_model_hook() -> None:
    model = FakeToolCallingModel(tool_calls=[])

    # Test `llm_input_messages`
    def pre_model_hook(state: AgentState):
        return {"llm_input_messages": [HumanMessage("Hello!")]}

    agent = create_react_agent(model, [], pre_model_hook=pre_model_hook)
    assert "pre_model_hook" in agent.nodes
    result = agent.invoke({"messages": [HumanMessage("hi?")]})
    assert result == {
        "messages": [
            _AnyIdHumanMessage(content="hi?"),
            AIMessage(content="Hello!", id="0"),
        ]
    }

    # Test `messages`
    def pre_model_hook(state: AgentState):
        return {
            "messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES), HumanMessage("Hello!")]
        }

    agent = create_react_agent(model, [], pre_model_hook=pre_model_hook)
    result = agent.invoke({"messages": [HumanMessage("hi?")]})
    assert result == {
        "messages": [
            _AnyIdHumanMessage(content="Hello!"),
            AIMessage(content="Hello!", id="1"),
        ]
    }


def test_post_model_hook() -> None:
    class FlagState(AgentState):
        flag: bool

    model = FakeToolCallingModel(tool_calls=[])

    def post_model_hook(state: FlagState) -> dict[str, bool]:
        return {"flag": True}

    pmh_agent = create_react_agent(
        model, [], post_model_hook=post_model_hook, state_schema=FlagState
    )

    assert "post_model_hook" in pmh_agent.nodes

    result = pmh_agent.invoke({"messages": [HumanMessage("hi?")], "flag": False})
    assert result["flag"] is True

    events = list(pmh_agent.stream({"messages": [HumanMessage("hi?")], "flag": False}))
    assert events == [
        {
            "agent": {
                "messages": [
                    AIMessage(
                        content="hi?",
                        additional_kwargs={},
                        response_metadata={},
                        id="1",
                    )
                ]
            }
        },
        {"post_model_hook": {"flag": True}},
    ]


def test_post_model_hook_with_structured_output() -> None:
    class WeatherResponse(BaseModel):
        temperature: float = Field(description="The temperature in fahrenheit")

    tool_calls = [[{"args": {}, "id": "1", "name": "get_weather"}]]

    def get_weather():
        """Get the weather"""
        return "The weather is sunny and 75°F."

    expected_structured_response = WeatherResponse(temperature=75)
    model = FakeToolCallingModel(
        tool_calls=tool_calls, structured_response=expected_structured_response
    )

    class State(AgentState):
        flag: bool
        structured_response: WeatherResponse

    def post_model_hook(state: State) -> dict[str, bool] | Command:
        return {"flag": True}

    agent = create_react_agent(
        model,
        [get_weather],
        response_format=WeatherResponse,
        post_model_hook=post_model_hook,
        state_schema=State,
    )

    assert "post_model_hook" in agent.nodes
    assert "generate_structured_response" in agent.nodes

    response = agent.invoke(
        {"messages": [HumanMessage("What's the weather?")], "flag": False}
    )
    assert response["flag"] is True
    assert response["structured_response"] == expected_structured_response

    events = list(
        agent.stream({"messages": [HumanMessage("What's the weather?")], "flag": False})
    )
    assert "generate_structured_response" in events[-1]
    assert events == [
        {
            "agent": {
                "messages": [
                    AIMessage(
                        content="What's the weather?",
                        additional_kwargs={},
                        response_metadata={},
                        id="2",
                        tool_calls=[
                            {
                                "name": "get_weather",
                                "args": {},
                                "id": "1",
                                "type": "tool_call",
                            }
                        ],
                    )
                ]
            }
        },
        {"post_model_hook": {"flag": True}},
        {
            "tools": {
                "messages": [
                    _AnyIdToolMessage(
                        content="The weather is sunny and 75°F.",
                        name="get_weather",
                        tool_call_id="1",
                    ),
                ]
            }
        },
        {
            "agent": {
                "messages": [
                    AIMessage(
                        content="What's the weather?-What's the weather?-The weather is sunny and 75°F.",
                        additional_kwargs={},
                        response_metadata={},
                        id="3",
                        tool_calls=[
                            {
                                "name": "get_weather",
                                "args": {},
                                "id": "1",
                                "type": "tool_call",
                            }
                        ],
                    )
                ]
            }
        },
        {"post_model_hook": {"flag": True}},
        {
            "generate_structured_response": {
                "structured_response": WeatherResponse(temperature=75.0)
            }
        },
    ]


@pytest.mark.parametrize(
    "state_schema", [AgentStateExtraKey, AgentStateExtraKeyPydantic]
)
def test_create_react_agent_inject_vars_with_post_model_hook(
    state_schema: StateSchemaType,
) -> None:
    store = InMemoryStore()
    namespace = ("test",)
    store.put(namespace, "test_key", {"bar": 3})

    if issubclass(state_schema, AgentStatePydantic):

        def tool1(
            some_val: int,
            state: Annotated[AgentStateExtraKeyPydantic, InjectedState],
            store: Annotated[BaseStore, InjectedStore()],
        ) -> str:
            """Tool 1 docstring."""
            store_val = store.get(namespace, "test_key").value["bar"]
            return some_val + state.foo + store_val
    else:

        def tool1(
            some_val: int,
            state: Annotated[dict, InjectedState],
            store: Annotated[BaseStore, InjectedStore()],
        ) -> str:
            """Tool 1 docstring."""
            store_val = store.get(namespace, "test_key").value["bar"]
            return some_val + state["foo"] + store_val

    tool_call = {
        "name": "tool1",
        "args": {"some_val": 1},
        "id": "some 0",
        "type": "tool_call",
    }

    def post_model_hook(state: dict) -> dict:
        """Post model hook is injecting a new foo key."""
        return {"foo": 2}

    model = FakeToolCallingModel(tool_calls=[[tool_call], []])
    agent = create_react_agent(
        model,
        ToolNode([tool1], handle_tool_errors=False),
        state_schema=state_schema,
        store=store,
        post_model_hook=post_model_hook,
    )
    input_message = HumanMessage("hi")
    result = agent.invoke({"messages": [input_message], "foo": 2})
    assert result["messages"] == [
        input_message,
        AIMessage(content="hi", tool_calls=[tool_call], id="0"),
        _AnyIdToolMessage(content="6", name="tool1", tool_call_id="some 0"),
        AIMessage("hi-hi-6", id="1"),
    ]
    assert result["foo"] == 2

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/prebuilt/tests/test_react_agent_graph.py
```py
from collections.abc import Callable

import pytest
from pydantic import BaseModel
from syrupy import SnapshotAssertion

from langgraph.prebuilt import create_react_agent
from tests.model import FakeToolCallingModel

model = FakeToolCallingModel()


def tool() -> None:
    """Testing tool."""
    ...


def pre_model_hook() -> None:
    """Pre-model hook."""
    ...


def post_model_hook() -> None:
    """Post-model hook."""
    ...


class ResponseFormat(BaseModel):
    """Response format for the agent."""

    result: str


@pytest.mark.parametrize("tools", [[], [tool]])
@pytest.mark.parametrize("pre_model_hook", [None, pre_model_hook])
@pytest.mark.parametrize("post_model_hook", [None, post_model_hook])
@pytest.mark.parametrize("response_format", [None, ResponseFormat])
def test_react_agent_graph_structure(
    snapshot: SnapshotAssertion,
    tools: list[Callable],
    pre_model_hook: Callable | None,
    post_model_hook: Callable | None,
    response_format: type[BaseModel] | None,
) -> None:
    agent = create_react_agent(
        model,
        tools=tools,
        pre_model_hook=pre_model_hook,
        post_model_hook=post_model_hook,
        response_format=response_format,
    )
    assert agent.get_graph().draw_mermaid(with_styles=False) == snapshot

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/prebuilt/tests/test_tool_node.py
```py
import contextlib
import dataclasses
import json
import sys
from functools import partial
from typing import (
    Annotated,
    Any,
    NoReturn,
    TypeVar,
)
from unittest.mock import Mock

import pytest
from langchain_core.messages import (
    AIMessage,
    AnyMessage,
    HumanMessage,
    RemoveMessage,
    ToolCall,
    ToolMessage,
)
from langchain_core.runnables.config import RunnableConfig
from langchain_core.tools import BaseTool, ToolException
from langchain_core.tools import tool as dec_tool
from langgraph.config import get_stream_writer
from langgraph.errors import GraphBubbleUp, GraphInterrupt
from langgraph.graph import START, MessagesState, StateGraph
from langgraph.graph.message import REMOVE_ALL_MESSAGES, add_messages
from langgraph.store.base import BaseStore
from langgraph.store.memory import InMemoryStore
from langgraph.types import Command, Send
from pydantic import BaseModel
from pydantic.v1 import BaseModel as BaseModelV1
from typing_extensions import TypedDict

from langgraph.prebuilt import (
    InjectedState,
    InjectedStore,
    ToolNode,
)
from langgraph.prebuilt.tool_node import (
    TOOL_CALL_ERROR_TEMPLATE,
    ToolInvocationError,
    tools_condition,
)

from .messages import _AnyIdHumanMessage, _AnyIdToolMessage
from .model import FakeToolCallingModel

pytestmark = pytest.mark.anyio


def _create_mock_runtime(store: BaseStore | None = None) -> Mock:
    """Create a mock Runtime object for testing ToolNode outside of graph context.

    This helper is needed because ToolNode._func expects a Runtime parameter
    which is injected by RunnableCallable from config["configurable"]["__pregel_runtime"].
    When testing ToolNode directly (outside a graph), we need to provide this manually.
    """
    mock_runtime = Mock()
    mock_runtime.store = store
    mock_runtime.context = None
    mock_runtime.stream_writer = lambda *args, **kwargs: None
    return mock_runtime


def _create_config_with_runtime(store: BaseStore | None = None) -> RunnableConfig:
    """Create a RunnableConfig with mock Runtime for testing ToolNode.

    Returns:
        RunnableConfig with __pregel_runtime in configurable dict.
    """
    return {"configurable": {"__pregel_runtime": _create_mock_runtime(store)}}


def tool1(some_val: int, some_other_val: str) -> str:
    """Tool 1 docstring."""
    if some_val == 0:
        msg = "Test error"
        raise ValueError(msg)
    return f"{some_val} - {some_other_val}"


async def tool2(some_val: int, some_other_val: str) -> str:
    """Tool 2 docstring."""
    if some_val == 0:
        msg = "Test error"
        raise ToolException(msg)
    return f"tool2: {some_val} - {some_other_val}"


async def tool3(some_val: int, some_other_val: str) -> str:
    """Tool 3 docstring."""
    return [
        {"key_1": some_val, "key_2": "foo"},
        {"key_1": some_other_val, "key_2": "baz"},
    ]


async def tool4(some_val: int, some_other_val: str) -> str:
    """Tool 4 docstring."""
    return [
        {"type": "image_url", "image_url": {"url": "abdc"}},
    ]


@dec_tool
def tool5(some_val: int) -> NoReturn:
    """Tool 5 docstring."""
    msg = "Test error"
    raise ToolException(msg)


tool5.handle_tool_error = "foo"


async def test_tool_node() -> None:
    """Test tool node."""
    result = ToolNode([tool1]).invoke(
        {
            "messages": [
                AIMessage(
                    "hi?",
                    tool_calls=[
                        {
                            "name": "tool1",
                            "args": {"some_val": 1, "some_other_val": "foo"},
                            "id": "some 0",
                        }
                    ],
                )
            ]
        },
        config=_create_config_with_runtime(),
    )

    tool_message: ToolMessage = result["messages"][-1]
    assert tool_message.type == "tool"
    assert tool_message.content == "1 - foo"
    assert tool_message.tool_call_id == "some 0"

    result2 = await ToolNode([tool2]).ainvoke(
        {
            "messages": [
                AIMessage(
                    "hi?",
                    tool_calls=[
                        {
                            "name": "tool2",
                            "args": {"some_val": 2, "some_other_val": "bar"},
                            "id": "some 1",
                        }
                    ],
                )
            ]
        },
        config=_create_config_with_runtime(),
    )

    tool_message: ToolMessage = result2["messages"][-1]
    assert tool_message.type == "tool"
    assert tool_message.content == "tool2: 2 - bar"

    # list of dicts tool content
    result3 = await ToolNode([tool3]).ainvoke(
        {
            "messages": [
                AIMessage(
                    "hi?",
                    tool_calls=[
                        {
                            "name": "tool3",
                            "args": {"some_val": 2, "some_other_val": "bar"},
                            "id": "some 2",
                        }
                    ],
                )
            ]
        },
        config=_create_config_with_runtime(),
    )
    tool_message: ToolMessage = result3["messages"][-1]
    assert tool_message.type == "tool"
    assert (
        tool_message.content
        == '[{"key_1": 2, "key_2": "foo"}, {"key_1": "bar", "key_2": "baz"}]'
    )
    assert tool_message.tool_call_id == "some 2"

    # list of content blocks tool content
    result4 = await ToolNode([tool4]).ainvoke(
        {
            "messages": [
                AIMessage(
                    "hi?",
                    tool_calls=[
                        {
                            "name": "tool4",
                            "args": {"some_val": 2, "some_other_val": "bar"},
                            "id": "some 3",
                        }
                    ],
                )
            ]
        },
        config=_create_config_with_runtime(),
    )
    tool_message: ToolMessage = result4["messages"][-1]
    assert tool_message.type == "tool"
    assert tool_message.content == [{"type": "image_url", "image_url": {"url": "abdc"}}]
    assert tool_message.tool_call_id == "some 3"


async def test_tool_node_tool_call_input() -> None:
    # Single tool call
    tool_call_1 = {
        "name": "tool1",
        "args": {"some_val": 1, "some_other_val": "foo"},
        "id": "some 0",
        "type": "tool_call",
    }
    result = ToolNode([tool1]).invoke(
        [tool_call_1], config=_create_config_with_runtime()
    )
    assert result["messages"] == [
        ToolMessage(content="1 - foo", tool_call_id="some 0", name="tool1"),
    ]

    # Multiple tool calls
    tool_call_2 = {
        "name": "tool1",
        "args": {"some_val": 2, "some_other_val": "bar"},
        "id": "some 1",
        "type": "tool_call",
    }
    result = ToolNode([tool1]).invoke(
        [tool_call_1, tool_call_2], config=_create_config_with_runtime()
    )
    assert result["messages"] == [
        ToolMessage(content="1 - foo", tool_call_id="some 0", name="tool1"),
        ToolMessage(content="2 - bar", tool_call_id="some 1", name="tool1"),
    ]

    # Test with unknown tool
    tool_call_3 = tool_call_1.copy()
    tool_call_3["name"] = "tool2"
    result = ToolNode([tool1]).invoke(
        [tool_call_1, tool_call_3], config=_create_config_with_runtime()
    )
    assert result["messages"] == [
        ToolMessage(content="1 - foo", tool_call_id="some 0", name="tool1"),
        ToolMessage(
            content="Error: tool2 is not a valid tool, try one of [tool1].",
            name="tool2",
            tool_call_id="some 0",
            status="error",
        ),
    ]


def test_tool_node_error_handling_default_invocation() -> None:
    tn = ToolNode([tool1])
    result = tn.invoke(
        {
            "messages": [
                AIMessage(
                    "hi?",
                    tool_calls=[
                        {
                            "name": "tool1",
                            "args": {"invalid": 0, "args": "foo"},
                            "id": "some id",
                        },
                    ],
                )
            ]
        },
        config=_create_config_with_runtime(),
    )

    assert all(m.type == "tool" for m in result["messages"])
    assert all(m.status == "error" for m in result["messages"])
    assert (
        "Error invoking tool 'tool1' with kwargs {'invalid': 0, 'args': 'foo'} with error:\n"
        in result["messages"][0].content
    )


def test_tool_node_error_handling_default_exception() -> None:
    tn = ToolNode([tool1])
    with pytest.raises(ValueError):
        tn.invoke(
            {
                "messages": [
                    AIMessage(
                        "hi?",
                        tool_calls=[
                            {
                                "name": "tool1",
                                "args": {"some_val": 0, "some_other_val": "foo"},
                                "id": "some id",
                            },
                        ],
                    )
                ]
            },
            config=_create_config_with_runtime(),
        )


async def test_tool_node_error_handling() -> None:
    def handle_all(e: ValueError | ToolException | ToolInvocationError):
        return TOOL_CALL_ERROR_TEMPLATE.format(error=repr(e))

    # test catching all exceptions, via:
    # - handle_tool_errors = True
    # - passing a tuple of all exceptions
    # - passing a callable with all exceptions in the signature
    for handle_tool_errors in (
        True,
        (ValueError, ToolException, ToolInvocationError),
        handle_all,
    ):
        result_error = await ToolNode(
            [tool1, tool2, tool3], handle_tool_errors=handle_tool_errors
        ).ainvoke(
            {
                "messages": [
                    AIMessage(
                        "hi?",
                        tool_calls=[
                            {
                                "name": "tool1",
                                "args": {"some_val": 0, "some_other_val": "foo"},
                                "id": "some id",
                            },
                            {
                                "name": "tool2",
                                "args": {"some_val": 0, "some_other_val": "bar"},
                                "id": "some other id",
                            },
                            {
                                "name": "tool3",
                                "args": {"some_val": 0},
                                "id": "another id",
                            },
                        ],
                    )
                ]
            },
            config=_create_config_with_runtime(),
        )

        assert all(m.type == "tool" for m in result_error["messages"])
        assert all(m.status == "error" for m in result_error["messages"])
        assert (
            result_error["messages"][0].content
            == f"Error: {ValueError('Test error')!r}\n Please fix your mistakes."
        )
        assert (
            result_error["messages"][1].content
            == f"Error: {ToolException('Test error')!r}\n Please fix your mistakes."
        )
        # Check that the validation error contains the field name
        assert "some_other_val" in result_error["messages"][2].content

        assert result_error["messages"][0].tool_call_id == "some id"
        assert result_error["messages"][1].tool_call_id == "some other id"
        assert result_error["messages"][2].tool_call_id == "another id"


async def test_tool_node_error_handling_callable() -> None:
    def handle_value_error(e: ValueError) -> str:
        return "Value error"

    def handle_tool_exception(e: ToolException) -> str:
        return "Tool exception"

    for handle_tool_errors in ("Value error", handle_value_error):
        result_error = await ToolNode(
            [tool1], handle_tool_errors=handle_tool_errors
        ).ainvoke(
            {
                "messages": [
                    AIMessage(
                        "hi?",
                        tool_calls=[
                            {
                                "name": "tool1",
                                "args": {"some_val": 0, "some_other_val": "foo"},
                                "id": "some id",
                            },
                        ],
                    )
                ]
            },
            config=_create_config_with_runtime(),
        )
        tool_message: ToolMessage = result_error["messages"][-1]
        assert tool_message.type == "tool"
        assert tool_message.status == "error"
        assert tool_message.content == "Value error"

    # test raising for an unhandled exception, via:
    # - passing a tuple of all exceptions
    # - passing a callable with all exceptions in the signature
    for handle_tool_errors in ((ValueError,), handle_value_error):
        with pytest.raises(ToolException) as exc_info:
            await ToolNode(
                [tool1, tool2], handle_tool_errors=handle_tool_errors
            ).ainvoke(
                {
                    "messages": [
                        AIMessage(
                            "hi?",
                            tool_calls=[
                                {
                                    "name": "tool1",
                                    "args": {"some_val": 0, "some_other_val": "foo"},
                                    "id": "some id",
                                },
                                {
                                    "name": "tool2",
                                    "args": {"some_val": 0, "some_other_val": "bar"},
                                    "id": "some other id",
                                },
                            ],
                        )
                    ]
                },
                config=_create_config_with_runtime(),
            )
        assert str(exc_info.value) == "Test error"

    for handle_tool_errors in ((ToolException,), handle_tool_exception):
        with pytest.raises(ValueError) as exc_info:
            await ToolNode(
                [tool1, tool2], handle_tool_errors=handle_tool_errors
            ).ainvoke(
                {
                    "messages": [
                        AIMessage(
                            "hi?",
                            tool_calls=[
                                {
                                    "name": "tool1",
                                    "args": {"some_val": 0, "some_other_val": "foo"},
                                    "id": "some id",
                                },
                                {
                                    "name": "tool2",
                                    "args": {"some_val": 0, "some_other_val": "bar"},
                                    "id": "some other id",
                                },
                            ],
                        )
                    ]
                },
                config=_create_config_with_runtime(),
            )
        assert str(exc_info.value) == "Test error"


async def test_tool_node_handle_tool_errors_false() -> None:
    with pytest.raises(ValueError) as exc_info:
        ToolNode([tool1], handle_tool_errors=False).invoke(
            {
                "messages": [
                    AIMessage(
                        "hi?",
                        tool_calls=[
                            {
                                "name": "tool1",
                                "args": {"some_val": 0, "some_other_val": "foo"},
                                "id": "some id",
                            }
                        ],
                    )
                ]
            },
            config=_create_config_with_runtime(),
        )

    assert str(exc_info.value) == "Test error"

    with pytest.raises(ToolException):
        await ToolNode([tool2], handle_tool_errors=False).ainvoke(
            {
                "messages": [
                    AIMessage(
                        "hi?",
                        tool_calls=[
                            {
                                "name": "tool2",
                                "args": {"some_val": 0, "some_other_val": "bar"},
                                "id": "some id",
                            }
                        ],
                    )
                ]
            },
            config=_create_config_with_runtime(),
        )

    assert str(exc_info.value) == "Test error"

    # test validation errors get raised if handle_tool_errors is False
    with pytest.raises(ToolInvocationError):
        ToolNode([tool1], handle_tool_errors=False).invoke(
            {
                "messages": [
                    AIMessage(
                        "hi?",
                        tool_calls=[
                            {
                                "name": "tool1",
                                "args": {"some_val": 0},
                                "id": "some id",
                            }
                        ],
                    )
                ]
            },
            config=_create_config_with_runtime(),
        )


def test_tool_node_individual_tool_error_handling() -> None:
    # test error handling on individual tools (and that it overrides overall error handling!)
    result_individual_tool_error_handler = ToolNode(
        [tool5], handle_tool_errors="bar"
    ).invoke(
        {
            "messages": [
                AIMessage(
                    "hi?",
                    tool_calls=[
                        {
                            "name": "tool5",
                            "args": {"some_val": 0},
                            "id": "some 0",
                        }
                    ],
                )
            ]
        },
        config=_create_config_with_runtime(),
    )

    tool_message: ToolMessage = result_individual_tool_error_handler["messages"][-1]
    assert tool_message.type == "tool"
    assert tool_message.status == "error"
    assert tool_message.content == "foo"
    assert tool_message.tool_call_id == "some 0"


def test_tool_node_incorrect_tool_name() -> None:
    result_incorrect_name = ToolNode([tool1, tool2]).invoke(
        {
            "messages": [
                AIMessage(
                    "hi?",
                    tool_calls=[
                        {
                            "name": "tool3",
                            "args": {"some_val": 1, "some_other_val": "foo"},
                            "id": "some 0",
                        }
                    ],
                )
            ]
        },
        config=_create_config_with_runtime(),
    )

    tool_message: ToolMessage = result_incorrect_name["messages"][-1]
    assert tool_message.type == "tool"
    assert tool_message.status == "error"
    assert (
        tool_message.content
        == "Error: tool3 is not a valid tool, try one of [tool1, tool2]."
    )
    assert tool_message.tool_call_id == "some 0"


def test_tool_node_node_interrupt() -> None:
    def tool_interrupt(some_val: int) -> None:
        """Tool docstring."""
        msg = "foo"
        raise GraphBubbleUp(msg)

    def handle(e: GraphInterrupt) -> str:
        return "handled"

    for handle_tool_errors in (True, (GraphBubbleUp,), "handled", handle, False):
        node = ToolNode([tool_interrupt], handle_tool_errors=handle_tool_errors)
        with pytest.raises(GraphBubbleUp) as exc_info:
            node.invoke(
                {
                    "messages": [
                        AIMessage(
                            "hi?",
                            tool_calls=[
                                {
                                    "name": "tool_interrupt",
                                    "args": {"some_val": 0},
                                    "id": "some 0",
                                }
                            ],
                        )
                    ]
                },
                config=_create_config_with_runtime(),
            )
            assert exc_info.value == "foo"


@pytest.mark.parametrize("input_type", ["dict", "tool_calls"])
async def test_tool_node_command(input_type: str) -> None:
    from langchain_core.tools.base import InjectedToolCallId

    @dec_tool
    def transfer_to_bob(tool_call_id: Annotated[str, InjectedToolCallId]):
        """Transfer to Bob"""
        return Command(
            update={
                "messages": [
                    ToolMessage(content="Transferred to Bob", tool_call_id=tool_call_id)
                ]
            },
            goto="bob",
            graph=Command.PARENT,
        )

    @dec_tool
    async def async_transfer_to_bob(tool_call_id: Annotated[str, InjectedToolCallId]):
        """Transfer to Bob"""
        return Command(
            update={
                "messages": [
                    ToolMessage(content="Transferred to Bob", tool_call_id=tool_call_id)
                ]
            },
            goto="bob",
            graph=Command.PARENT,
        )

    class CustomToolSchema(BaseModel):
        tool_call_id: Annotated[str, InjectedToolCallId]

    class MyCustomTool(BaseTool):
        def _run(*args: Any, **kwargs: Any):
            return Command(
                update={
                    "messages": [
                        ToolMessage(
                            content="Transferred to Bob",
                            tool_call_id=kwargs["tool_call_id"],
                        )
                    ]
                },
                goto="bob",
                graph=Command.PARENT,
            )

        async def _arun(*args: Any, **kwargs: Any):
            return Command(
                update={
                    "messages": [
                        ToolMessage(
                            content="Transferred to Bob",
                            tool_call_id=kwargs["tool_call_id"],
                        )
                    ]
                },
                goto="bob",
                graph=Command.PARENT,
            )

    custom_tool = MyCustomTool(
        name="custom_transfer_to_bob",
        description="Transfer to bob",
        args_schema=CustomToolSchema,
    )
    async_custom_tool = MyCustomTool(
        name="async_custom_transfer_to_bob",
        description="Transfer to bob",
        args_schema=CustomToolSchema,
    )

    # test mixing regular tools and tools returning commands
    def add(a: int, b: int) -> int:
        """Add two numbers"""
        return a + b

    tool_calls = [
        {"args": {"a": 1, "b": 2}, "id": "1", "name": "add", "type": "tool_call"},
        {"args": {}, "id": "2", "name": "transfer_to_bob", "type": "tool_call"},
    ]
    if input_type == "dict":
        input_ = {"messages": [AIMessage("", tool_calls=tool_calls)]}
    elif input_type == "tool_calls":
        input_ = tool_calls
    result = ToolNode([add, transfer_to_bob]).invoke(
        input_, config=_create_config_with_runtime()
    )

    assert result == [
        {
            "messages": [
                ToolMessage(
                    content="3",
                    tool_call_id="1",
                    name="add",
                )
            ]
        },
        Command(
            update={
                "messages": [
                    ToolMessage(
                        content="Transferred to Bob",
                        tool_call_id="2",
                        name="transfer_to_bob",
                    )
                ]
            },
            goto="bob",
            graph=Command.PARENT,
        ),
    ]

    # test tools returning commands

    # test sync tools
    for tool in [transfer_to_bob, custom_tool]:
        result = ToolNode([tool]).invoke(
            {
                "messages": [
                    AIMessage(
                        "", tool_calls=[{"args": {}, "id": "1", "name": tool.name}]
                    )
                ]
            },
            config=_create_config_with_runtime(),
        )
        assert result == [
            Command(
                update={
                    "messages": [
                        ToolMessage(
                            content="Transferred to Bob",
                            tool_call_id="1",
                            name=tool.name,
                        )
                    ]
                },
                goto="bob",
                graph=Command.PARENT,
            )
        ]

    # test async tools
    for tool in [async_transfer_to_bob, async_custom_tool]:
        result = await ToolNode([tool]).ainvoke(
            {
                "messages": [
                    AIMessage(
                        "", tool_calls=[{"args": {}, "id": "1", "name": tool.name}]
                    )
                ]
            },
            config=_create_config_with_runtime(),
        )
        assert result == [
            Command(
                update={
                    "messages": [
                        ToolMessage(
                            content="Transferred to Bob",
                            tool_call_id="1",
                            name=tool.name,
                        )
                    ]
                },
                goto="bob",
                graph=Command.PARENT,
            )
        ]

    # test multiple commands
    result = ToolNode([transfer_to_bob, custom_tool]).invoke(
        {
            "messages": [
                AIMessage(
                    "",
                    tool_calls=[
                        {"args": {}, "id": "1", "name": "transfer_to_bob"},
                        {"args": {}, "id": "2", "name": "custom_transfer_to_bob"},
                    ],
                )
            ]
        },
        config=_create_config_with_runtime(),
    )
    assert result == [
        Command(
            update={
                "messages": [
                    ToolMessage(
                        content="Transferred to Bob",
                        tool_call_id="1",
                        name="transfer_to_bob",
                    )
                ]
            },
            goto="bob",
            graph=Command.PARENT,
        ),
        Command(
            update={
                "messages": [
                    ToolMessage(
                        content="Transferred to Bob",
                        tool_call_id="2",
                        name="custom_transfer_to_bob",
                    )
                ]
            },
            goto="bob",
            graph=Command.PARENT,
        ),
    ]

    # test validation (mismatch between input type and command.update type)
    with pytest.raises(ValueError):

        @dec_tool
        def list_update_tool(tool_call_id: Annotated[str, InjectedToolCallId]):
            """My tool"""
            return Command(
                update=[ToolMessage(content="foo", tool_call_id=tool_call_id)]
            )

        ToolNode([list_update_tool]).invoke(
            {
                "messages": [
                    AIMessage(
                        "",
                        tool_calls=[
                            {"args": {}, "id": "1", "name": "list_update_tool"}
                        ],
                    )
                ]
            },
            config=_create_config_with_runtime(),
        )

    # test validation (missing tool message in the update for current graph)
    with pytest.raises(ValueError):

        @dec_tool
        def no_update_tool():
            """My tool"""
            return Command(update={"messages": []})

        ToolNode([no_update_tool]).invoke(
            {
                "messages": [
                    AIMessage(
                        "",
                        tool_calls=[{"args": {}, "id": "1", "name": "no_update_tool"}],
                    )
                ]
            },
            config=_create_config_with_runtime(),
        )

    # test validation (tool message with a wrong tool call ID)
    with pytest.raises(ValueError):

        @dec_tool
        def mismatching_tool_call_id_tool():
            """My tool"""
            return Command(
                update={"messages": [ToolMessage(content="foo", tool_call_id="2")]}
            )

        ToolNode([mismatching_tool_call_id_tool]).invoke(
            {
                "messages": [
                    AIMessage(
                        "",
                        tool_calls=[
                            {
                                "args": {},
                                "id": "1",
                                "name": "mismatching_tool_call_id_tool",
                            }
                        ],
                    )
                ]
            },
            config=_create_config_with_runtime(),
        )

    # test validation (missing tool message in the update for parent graph is OK)
    @dec_tool
    def node_update_parent_tool():
        """No update"""
        return Command(update={"messages": []}, graph=Command.PARENT)

    assert ToolNode([node_update_parent_tool]).invoke(
        {
            "messages": [
                AIMessage(
                    "",
                    tool_calls=[
                        {"args": {}, "id": "1", "name": "node_update_parent_tool"}
                    ],
                )
            ]
        },
        config=_create_config_with_runtime(),
    ) == [Command(update={"messages": []}, graph=Command.PARENT)]


async def test_tool_node_command_list_input() -> None:
    from langchain_core.tools.base import InjectedToolCallId

    @dec_tool
    def transfer_to_bob(tool_call_id: Annotated[str, InjectedToolCallId]):
        """Transfer to Bob"""
        return Command(
            update=[
                ToolMessage(content="Transferred to Bob", tool_call_id=tool_call_id)
            ],
            goto="bob",
            graph=Command.PARENT,
        )

    @dec_tool
    async def async_transfer_to_bob(tool_call_id: Annotated[str, InjectedToolCallId]):
        """Transfer to Bob"""
        return Command(
            update=[
                ToolMessage(content="Transferred to Bob", tool_call_id=tool_call_id)
            ],
            goto="bob",
            graph=Command.PARENT,
        )

    class CustomToolSchema(BaseModel):
        tool_call_id: Annotated[str, InjectedToolCallId]

    class MyCustomTool(BaseTool):
        def _run(*args: Any, **kwargs: Any):
            return Command(
                update=[
                    ToolMessage(
                        content="Transferred to Bob",
                        tool_call_id=kwargs["tool_call_id"],
                    )
                ],
                goto="bob",
                graph=Command.PARENT,
            )

        async def _arun(*args: Any, **kwargs: Any):
            return Command(
                update=[
                    ToolMessage(
                        content="Transferred to Bob",
                        tool_call_id=kwargs["tool_call_id"],
                    )
                ],
                goto="bob",
                graph=Command.PARENT,
            )

    custom_tool = MyCustomTool(
        name="custom_transfer_to_bob",
        description="Transfer to bob",
        args_schema=CustomToolSchema,
    )
    async_custom_tool = MyCustomTool(
        name="async_custom_transfer_to_bob",
        description="Transfer to bob",
        args_schema=CustomToolSchema,
    )

    # test mixing regular tools and tools returning commands
    def add(a: int, b: int) -> int:
        """Add two numbers"""
        return a + b

    result = ToolNode([add, transfer_to_bob]).invoke(
        [
            AIMessage(
                "",
                tool_calls=[
                    {"args": {"a": 1, "b": 2}, "id": "1", "name": "add"},
                    {"args": {}, "id": "2", "name": "transfer_to_bob"},
                ],
            )
        ],
        config=_create_config_with_runtime(),
    )

    assert result == [
        [
            ToolMessage(
                content="3",
                tool_call_id="1",
                name="add",
            )
        ],
        Command(
            update=[
                ToolMessage(
                    content="Transferred to Bob",
                    tool_call_id="2",
                    name="transfer_to_bob",
                )
            ],
            goto="bob",
            graph=Command.PARENT,
        ),
    ]

    # test tools returning commands

    # test sync tools
    for tool in [transfer_to_bob, custom_tool]:
        result = ToolNode([tool]).invoke(
            [AIMessage("", tool_calls=[{"args": {}, "id": "1", "name": tool.name}])],
            config=_create_config_with_runtime(),
        )
        assert result == [
            Command(
                update=[
                    ToolMessage(
                        content="Transferred to Bob",
                        tool_call_id="1",
                        name=tool.name,
                    )
                ],
                goto="bob",
                graph=Command.PARENT,
            )
        ]

    # test async tools
    for tool in [async_transfer_to_bob, async_custom_tool]:
        result = await ToolNode([tool]).ainvoke(
            [AIMessage("", tool_calls=[{"args": {}, "id": "1", "name": tool.name}])],
            config=_create_config_with_runtime(),
        )
        assert result == [
            Command(
                update=[
                    ToolMessage(
                        content="Transferred to Bob",
                        tool_call_id="1",
                        name=tool.name,
                    )
                ],
                goto="bob",
                graph=Command.PARENT,
            )
        ]

    # test multiple commands
    result = ToolNode([transfer_to_bob, custom_tool]).invoke(
        [
            AIMessage(
                "",
                tool_calls=[
                    {"args": {}, "id": "1", "name": "transfer_to_bob"},
                    {"args": {}, "id": "2", "name": "custom_transfer_to_bob"},
                ],
            )
        ],
        config=_create_config_with_runtime(),
    )
    assert result == [
        Command(
            update=[
                ToolMessage(
                    content="Transferred to Bob",
                    tool_call_id="1",
                    name="transfer_to_bob",
                )
            ],
            goto="bob",
            graph=Command.PARENT,
        ),
        Command(
            update=[
                ToolMessage(
                    content="Transferred to Bob",
                    tool_call_id="2",
                    name="custom_transfer_to_bob",
                )
            ],
            goto="bob",
            graph=Command.PARENT,
        ),
    ]

    # test validation (mismatch between input type and command.update type)
    with pytest.raises(ValueError):

        @dec_tool
        def list_update_tool(tool_call_id: Annotated[str, InjectedToolCallId]):
            """My tool"""
            return Command(
                update={
                    "messages": [ToolMessage(content="foo", tool_call_id=tool_call_id)]
                }
            )

        ToolNode([list_update_tool]).invoke(
            [
                AIMessage(
                    "",
                    tool_calls=[{"args": {}, "id": "1", "name": "list_update_tool"}],
                )
            ],
            config=_create_config_with_runtime(),
        )

    # test validation (missing tool message in the update for current graph)
    with pytest.raises(ValueError):

        @dec_tool
        def no_update_tool():
            """My tool"""
            return Command(update=[])

        ToolNode([no_update_tool]).invoke(
            [
                AIMessage(
                    "",
                    tool_calls=[{"args": {}, "id": "1", "name": "no_update_tool"}],
                )
            ],
            config=_create_config_with_runtime(),
        )

    # test validation (tool message with a wrong tool call ID)
    with pytest.raises(ValueError):

        @dec_tool
        def mismatching_tool_call_id_tool():
            """My tool"""
            return Command(update=[ToolMessage(content="foo", tool_call_id="2")])

        ToolNode([mismatching_tool_call_id_tool]).invoke(
            [
                AIMessage(
                    "",
                    tool_calls=[
                        {"args": {}, "id": "1", "name": "mismatching_tool_call_id_tool"}
                    ],
                )
            ],
            config=_create_config_with_runtime(),
        )

    # test validation (missing tool message in the update for parent graph is OK)
    @dec_tool
    def node_update_parent_tool():
        """No update"""
        return Command(update=[], graph=Command.PARENT)

    assert ToolNode([node_update_parent_tool]).invoke(
        [
            AIMessage(
                "",
                tool_calls=[{"args": {}, "id": "1", "name": "node_update_parent_tool"}],
            )
        ],
        config=_create_config_with_runtime(),
    ) == [Command(update=[], graph=Command.PARENT)]


def test_tool_node_parent_command_with_send() -> None:
    from langchain_core.tools.base import InjectedToolCallId

    @dec_tool
    def transfer_to_alice(tool_call_id: Annotated[str, InjectedToolCallId]):
        """Transfer to Alice"""
        return Command(
            goto=[
                Send(
                    "alice",
                    {
                        "messages": [
                            ToolMessage(
                                content="Transferred to Alice",
                                name="transfer_to_alice",
                                tool_call_id=tool_call_id,
                            )
                        ]
                    },
                )
            ],
            graph=Command.PARENT,
        )

    @dec_tool
    def transfer_to_bob(tool_call_id: Annotated[str, InjectedToolCallId]):
        """Transfer to Bob"""
        return Command(
            goto=[
                Send(
                    "bob",
                    {
                        "messages": [
                            ToolMessage(
                                content="Transferred to Bob",
                                name="transfer_to_bob",
                                tool_call_id=tool_call_id,
                            )
                        ]
                    },
                )
            ],
            graph=Command.PARENT,
        )

    tool_calls = [
        {"args": {}, "id": "1", "name": "transfer_to_alice", "type": "tool_call"},
        {"args": {}, "id": "2", "name": "transfer_to_bob", "type": "tool_call"},
    ]

    result = ToolNode([transfer_to_alice, transfer_to_bob]).invoke(
        [AIMessage("", tool_calls=tool_calls)],
        config=_create_config_with_runtime(),
    )

    assert result == [
        Command(
            goto=[
                Send(
                    "alice",
                    {
                        "messages": [
                            ToolMessage(
                                content="Transferred to Alice",
                                name="transfer_to_alice",
                                tool_call_id="1",
                            )
                        ]
                    },
                ),
                Send(
                    "bob",
                    {
                        "messages": [
                            ToolMessage(
                                content="Transferred to Bob",
                                name="transfer_to_bob",
                                tool_call_id="2",
                            )
                        ]
                    },
                ),
            ],
            graph=Command.PARENT,
        )
    ]


async def test_tool_node_command_remove_all_messages() -> None:
    from langchain_core.tools.base import InjectedToolCallId

    @dec_tool
    def remove_all_messages_tool(tool_call_id: Annotated[str, InjectedToolCallId]):
        """A tool that removes all messages."""
        return Command(update={"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES)]})

    tool_node = ToolNode([remove_all_messages_tool])
    tool_call = {
        "name": "remove_all_messages_tool",
        "args": {},
        "id": "tool_call_123",
    }
    result = await tool_node.ainvoke(
        {"messages": [AIMessage(content="", tool_calls=[tool_call])]},
        config=_create_config_with_runtime(),
    )

    assert isinstance(result, list)
    assert len(result) == 1
    command = result[0]
    assert isinstance(command, Command)
    assert command.update == {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES)]}


class _InjectStateSchema(TypedDict):
    messages: list
    foo: str


class _InjectedStatePydanticV2Schema(BaseModel):
    messages: list
    foo: str


@dataclasses.dataclass
class _InjectedStateDataclassSchema:
    messages: list
    foo: str


_INJECTED_STATE_SCHEMAS = [
    _InjectStateSchema,
    _InjectedStatePydanticV2Schema,
    _InjectedStateDataclassSchema,
]

if sys.version_info < (3, 14):

    class _InjectedStatePydanticSchema(BaseModelV1):
        messages: list
        foo: str

    _INJECTED_STATE_SCHEMAS.append(_InjectedStatePydanticSchema)

T = TypeVar("T")


@pytest.mark.parametrize("schema_", _INJECTED_STATE_SCHEMAS)
def test_tool_node_inject_state(schema_: type[T]) -> None:
    def tool1(some_val: int, state: Annotated[T, InjectedState]) -> str:
        """Tool 1 docstring."""
        if isinstance(state, dict):
            return state["foo"]
        return state.foo

    def tool2(some_val: int, state: Annotated[T, InjectedState()]) -> str:
        """Tool 2 docstring."""
        if isinstance(state, dict):
            return state["foo"]
        return state.foo

    def tool3(
        some_val: int,
        foo: Annotated[str, InjectedState("foo")],
        msgs: Annotated[list[AnyMessage], InjectedState("messages")],
    ) -> str:
        """Tool 1 docstring."""
        return foo

    def tool4(
        some_val: int, msgs: Annotated[list[AnyMessage], InjectedState("messages")]
    ) -> str:
        """Tool 1 docstring."""
        return msgs[0].content

    node = ToolNode([tool1, tool2, tool3, tool4], handle_tool_errors=True)
    for tool_name in ("tool1", "tool2", "tool3"):
        tool_call = {
            "name": tool_name,
            "args": {"some_val": 1},
            "id": "some 0",
            "type": "tool_call",
        }
        msg = AIMessage("hi?", tool_calls=[tool_call])
        result = node.invoke(
            schema_(messages=[msg], foo="bar"), config=_create_config_with_runtime()
        )
        tool_message = result["messages"][-1]
        assert tool_message.content == "bar", f"Failed for tool={tool_name}"

        if tool_name == "tool3":
            failure_input = None
            with contextlib.suppress(Exception):
                failure_input = schema_(messages=[msg], notfoo="bar")
            if failure_input is not None:
                with pytest.raises(KeyError):
                    node.invoke(failure_input, config=_create_config_with_runtime())

                with pytest.raises(ValueError):
                    node.invoke([msg], config=_create_config_with_runtime())
        else:
            failure_input = None
            try:
                failure_input = schema_(messages=[msg], notfoo="bar")
            except Exception:
                # We'd get a validation error from pydantic state and wouldn't make it to the node
                # anyway
                pass
            if failure_input is not None:
                messages_ = node.invoke(
                    failure_input, config=_create_config_with_runtime()
                )
                tool_message = messages_["messages"][-1]
                assert "KeyError" in tool_message.content
                tool_message = node.invoke([msg], config=_create_config_with_runtime())[
                    -1
                ]
                assert "KeyError" in tool_message.content

    tool_call = {
        "name": "tool4",
        "args": {"some_val": 1},
        "id": "some 0",
        "type": "tool_call",
    }
    msg = AIMessage("hi?", tool_calls=[tool_call])
    result = node.invoke(
        schema_(messages=[msg], foo=""), config=_create_config_with_runtime()
    )
    tool_message = result["messages"][-1]
    assert tool_message.content == "hi?"

    result = node.invoke([msg], config=_create_config_with_runtime())
    tool_message = result[-1]
    assert tool_message.content == "hi?"


def test_tool_node_inject_store() -> None:
    store = InMemoryStore()
    namespace = ("test",)

    def tool1(some_val: int, store: Annotated[BaseStore, InjectedStore()]) -> str:
        """Tool 1 docstring."""
        store_val = store.get(namespace, "test_key").value["foo"]
        return f"Some val: {some_val}, store val: {store_val}"

    def tool2(some_val: int, store: Annotated[BaseStore, InjectedStore()]) -> str:
        """Tool 2 docstring."""
        store_val = store.get(namespace, "test_key").value["foo"]
        return f"Some val: {some_val}, store val: {store_val}"

    def tool3(
        some_val: int,
        bar: Annotated[str, InjectedState("bar")],
        store: Annotated[BaseStore, InjectedStore()],
    ) -> str:
        """Tool 3 docstring."""
        store_val = store.get(namespace, "test_key").value["foo"]
        return f"Some val: {some_val}, store val: {store_val}, state val: {bar}"

    node = ToolNode([tool1, tool2, tool3], handle_tool_errors=True)
    store.put(namespace, "test_key", {"foo": "bar"})

    class State(MessagesState):
        bar: str

    builder = StateGraph(State)
    builder.add_node("tools", node)
    builder.add_edge(START, "tools")
    graph = builder.compile(store=store)

    for tool_name in ("tool1", "tool2"):
        tool_call = {
            "name": tool_name,
            "args": {"some_val": 1},
            "id": "some 0",
            "type": "tool_call",
        }
        msg = AIMessage("hi?", tool_calls=[tool_call])
        node_result = node.invoke(
            {"messages": [msg]}, config=_create_config_with_runtime(store=store)
        )
        graph_result = graph.invoke({"messages": [msg]})
        for result in (node_result, graph_result):
            result["messages"][-1]
            tool_message = result["messages"][-1]
            assert tool_message.content == "Some val: 1, store val: bar", (
                f"Failed for tool={tool_name}"
            )

    tool_call = {
        "name": "tool3",
        "args": {"some_val": 1},
        "id": "some 0",
        "type": "tool_call",
    }
    msg = AIMessage("hi?", tool_calls=[tool_call])
    node_result = node.invoke(
        {"messages": [msg], "bar": "baz"},
        config=_create_config_with_runtime(store=store),
    )
    graph_result = graph.invoke({"messages": [msg], "bar": "baz"})
    for result in (node_result, graph_result):
        result["messages"][-1]
        tool_message = result["messages"][-1]
        assert tool_message.content == "Some val: 1, store val: bar, state val: baz", (
            f"Failed for tool={tool_name}"
        )

    # test injected store without passing store to compiled graph
    failing_graph = builder.compile()
    with pytest.raises(ValueError):
        failing_graph.invoke({"messages": [msg], "bar": "baz"})


def test_tool_node_ensure_utf8() -> None:
    @dec_tool
    def get_day_list(days: list[str]) -> list[str]:
        """choose days"""
        return days

    data = ["星期一", "水曜日", "목요일", "Friday"]
    tools = [get_day_list]
    tool_calls = [ToolCall(name=get_day_list.name, args={"days": data}, id="test_id")]
    outputs: list[ToolMessage] = ToolNode(tools).invoke(
        [AIMessage(content="", tool_calls=tool_calls)],
        config=_create_config_with_runtime(),
    )
    assert outputs[0].content == json.dumps(data, ensure_ascii=False)


def test_tool_node_messages_key() -> None:
    @dec_tool
    def add(a: int, b: int) -> int:
        """Adds a and b."""
        return a + b

    model = FakeToolCallingModel(
        tool_calls=[[ToolCall(name=add.name, args={"a": 1, "b": 2}, id="test_id")]]
    )

    class State(TypedDict):
        subgraph_messages: Annotated[list[AnyMessage], add_messages]

    def call_model(state: State) -> dict[str, Any]:
        response = model.invoke(state["subgraph_messages"])
        model.tool_calls = []
        return {"subgraph_messages": response}

    builder = StateGraph(State)
    builder.add_node("agent", call_model)
    builder.add_node("tools", ToolNode([add], messages_key="subgraph_messages"))
    builder.add_conditional_edges(
        "agent", partial(tools_condition, messages_key="subgraph_messages")
    )
    builder.add_edge(START, "agent")
    builder.add_edge("tools", "agent")

    graph = builder.compile()
    result = graph.invoke({"subgraph_messages": [HumanMessage(content="hi")]})
    assert result["subgraph_messages"] == [
        _AnyIdHumanMessage(content="hi"),
        AIMessage(
            content="hi",
            id="0",
            tool_calls=[ToolCall(name=add.name, args={"a": 1, "b": 2}, id="test_id")],
        ),
        _AnyIdToolMessage(content="3", name=add.name, tool_call_id="test_id"),
        AIMessage(content="hi-hi-3", id="1"),
    ]


def test_tool_node_stream_writer() -> None:
    @dec_tool
    def streaming_tool(x: int) -> str:
        """Do something with writer."""
        my_writer = get_stream_writer()
        for value in ["foo", "bar", "baz"]:
            my_writer({"custom_tool_value": value})

        return x

    tool_node = ToolNode([streaming_tool])
    graph = (
        StateGraph(MessagesState)
        .add_node("tools", tool_node)
        .add_edge(START, "tools")
        .compile()
    )

    tool_call = {
        "name": "streaming_tool",
        "args": {"x": 1},
        "id": "1",
        "type": "tool_call",
    }
    inputs = {
        "messages": [AIMessage("", tool_calls=[tool_call])],
    }

    assert list(graph.stream(inputs, stream_mode="custom")) == [
        {"custom_tool_value": "foo"},
        {"custom_tool_value": "bar"},
        {"custom_tool_value": "baz"},
    ]
    assert list(graph.stream(inputs, stream_mode=["custom", "updates"])) == [
        ("custom", {"custom_tool_value": "foo"}),
        ("custom", {"custom_tool_value": "bar"}),
        ("custom", {"custom_tool_value": "baz"}),
        (
            "updates",
            {
                "tools": {
                    "messages": [
                        _AnyIdToolMessage(
                            content="1",
                            name="streaming_tool",
                            tool_call_id="1",
                        ),
                    ],
                },
            },
        ),
    ]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/prebuilt/tests/test_tool_node_interceptor_unregistered.py
```py
"""Test tool node interceptor handling of unregistered tools."""

from collections.abc import Awaitable, Callable
from unittest.mock import Mock

import pytest
from langchain_core.messages import AIMessage, ToolMessage
from langchain_core.runnables.config import RunnableConfig
from langchain_core.tools import tool as dec_tool
from langgraph.store.base import BaseStore
from langgraph.types import Command

from langgraph.prebuilt import ToolNode
from langgraph.prebuilt.tool_node import ToolCallRequest

pytestmark = pytest.mark.anyio


def _create_mock_runtime(store: BaseStore | None = None) -> Mock:
    """Create a mock Runtime object for testing ToolNode outside of graph context.

    This helper is needed because ToolNode._func expects a Runtime parameter
    which is injected by RunnableCallable from config["configurable"]["__pregel_runtime"].
    When testing ToolNode directly (outside a graph), we need to provide this manually.
    """
    mock_runtime = Mock()
    mock_runtime.store = store
    mock_runtime.context = None
    mock_runtime.stream_writer = lambda *args, **kwargs: None
    return mock_runtime


def _create_config_with_runtime(store: BaseStore | None = None) -> RunnableConfig:
    """Create a RunnableConfig with mock Runtime for testing ToolNode.

    Returns:
        RunnableConfig with __pregel_runtime in configurable dict.
    """
    return {"configurable": {"__pregel_runtime": _create_mock_runtime(store)}}


@dec_tool
def registered_tool(x: int) -> str:
    """A registered tool."""
    return f"Result: {x}"


def test_interceptor_can_handle_unregistered_tool_sync() -> None:
    """Test that interceptor can handle requests for unregistered tools (sync)."""

    def interceptor(
        request: ToolCallRequest,
        execute: Callable[[ToolCallRequest], ToolMessage | Command],
    ) -> ToolMessage | Command:
        """Intercept and handle unregistered tools."""
        if request.tool_call["name"] == "unregistered_tool":
            # Short-circuit without calling execute for unregistered tool
            return ToolMessage(
                content="Handled by interceptor",
                tool_call_id=request.tool_call["id"],
                name="unregistered_tool",
            )
        # Pass through for registered tools
        return execute(request)

    node = ToolNode([registered_tool], wrap_tool_call=interceptor)

    # Test registered tool works normally
    result = node.invoke(
        [
            AIMessage(
                "",
                tool_calls=[
                    {
                        "name": "registered_tool",
                        "args": {"x": 42},
                        "id": "1",
                        "type": "tool_call",
                    }
                ],
            )
        ],
        config=_create_config_with_runtime(),
    )
    assert result[0].content == "Result: 42"
    assert result[0].tool_call_id == "1"

    # Test unregistered tool is intercepted and handled
    result = node.invoke(
        [
            AIMessage(
                "",
                tool_calls=[
                    {
                        "name": "unregistered_tool",
                        "args": {"x": 99},
                        "id": "2",
                        "type": "tool_call",
                    }
                ],
            )
        ],
        config=_create_config_with_runtime(),
    )
    assert result[0].content == "Handled by interceptor"
    assert result[0].tool_call_id == "2"
    assert result[0].name == "unregistered_tool"


async def test_interceptor_can_handle_unregistered_tool_async() -> None:
    """Test that interceptor can handle requests for unregistered tools (async)."""

    async def async_interceptor(
        request: ToolCallRequest,
        execute: Callable[[ToolCallRequest], Awaitable[ToolMessage | Command]],
    ) -> ToolMessage | Command:
        """Intercept and handle unregistered tools."""
        if request.tool_call["name"] == "unregistered_tool":
            # Short-circuit without calling execute for unregistered tool
            return ToolMessage(
                content="Handled by async interceptor",
                tool_call_id=request.tool_call["id"],
                name="unregistered_tool",
            )
        # Pass through for registered tools
        return await execute(request)

    node = ToolNode([registered_tool], awrap_tool_call=async_interceptor)

    # Test registered tool works normally
    result = await node.ainvoke(
        [
            AIMessage(
                "",
                tool_calls=[
                    {
                        "name": "registered_tool",
                        "args": {"x": 42},
                        "id": "1",
                        "type": "tool_call",
                    }
                ],
            )
        ],
        config=_create_config_with_runtime(),
    )
    assert result[0].content == "Result: 42"
    assert result[0].tool_call_id == "1"

    # Test unregistered tool is intercepted and handled
    result = await node.ainvoke(
        [
            AIMessage(
                "",
                tool_calls=[
                    {
                        "name": "unregistered_tool",
                        "args": {"x": 99},
                        "id": "2",
                        "type": "tool_call",
                    }
                ],
            )
        ],
        config=_create_config_with_runtime(),
    )
    assert result[0].content == "Handled by async interceptor"
    assert result[0].tool_call_id == "2"
    assert result[0].name == "unregistered_tool"


def test_unregistered_tool_error_when_interceptor_calls_execute() -> None:
    """Test that unregistered tools error if interceptor tries to execute them."""

    def bad_interceptor(
        request: ToolCallRequest,
        execute: Callable[[ToolCallRequest], ToolMessage | Command],
    ) -> ToolMessage | Command:
        """Interceptor that tries to execute unregistered tool."""
        # This should fail validation when execute is called
        return execute(request)

    node = ToolNode([registered_tool], wrap_tool_call=bad_interceptor)

    # Registered tool should still work
    result = node.invoke(
        [
            AIMessage(
                "",
                tool_calls=[
                    {
                        "name": "registered_tool",
                        "args": {"x": 42},
                        "id": "1",
                        "type": "tool_call",
                    }
                ],
            )
        ],
        config=_create_config_with_runtime(),
    )
    assert result[0].content == "Result: 42"

    # Unregistered tool should error when interceptor calls execute
    result = node.invoke(
        [
            AIMessage(
                "",
                tool_calls=[
                    {
                        "name": "unregistered_tool",
                        "args": {"x": 99},
                        "id": "2",
                        "type": "tool_call",
                    }
                ],
            )
        ],
        config=_create_config_with_runtime(),
    )
    # Should get validation error message
    assert result[0].status == "error"
    assert "is not a valid tool" in result[0].content
    assert result[0].tool_call_id == "2"


def test_interceptor_handles_mix_of_registered_and_unregistered() -> None:
    """Test interceptor handling mix of registered and unregistered tools."""

    def selective_interceptor(
        request: ToolCallRequest,
        execute: Callable[[ToolCallRequest], ToolMessage | Command],
    ) -> ToolMessage | Command:
        """Handle unregistered tools, pass through registered ones."""
        if request.tool_call["name"] == "magic_tool":
            return ToolMessage(
                content=f"Magic result: {request.tool_call['args'].get('value', 0) * 2}",
                tool_call_id=request.tool_call["id"],
                name="magic_tool",
            )
        return execute(request)

    node = ToolNode([registered_tool], wrap_tool_call=selective_interceptor)

    # Test multiple tool calls - mix of registered and unregistered
    result = node.invoke(
        [
            AIMessage(
                "",
                tool_calls=[
                    {
                        "name": "registered_tool",
                        "args": {"x": 10},
                        "id": "1",
                        "type": "tool_call",
                    },
                    {
                        "name": "magic_tool",
                        "args": {"value": 5},
                        "id": "2",
                        "type": "tool_call",
                    },
                    {
                        "name": "registered_tool",
                        "args": {"x": 20},
                        "id": "3",
                        "type": "tool_call",
                    },
                ],
            )
        ],
        config=_create_config_with_runtime(),
    )

    # All tools should execute successfully
    assert len(result) == 3
    assert result[0].content == "Result: 10"
    assert result[0].tool_call_id == "1"
    assert result[1].content == "Magic result: 10"
    assert result[1].tool_call_id == "2"
    assert result[2].content == "Result: 20"
    assert result[2].tool_call_id == "3"


def test_interceptor_command_for_unregistered_tool() -> None:
    """Test interceptor returning Command for unregistered tool."""

    def command_interceptor(
        request: ToolCallRequest,
        execute: Callable[[ToolCallRequest], ToolMessage | Command],
    ) -> ToolMessage | Command:
        """Return Command for unregistered tools."""
        if request.tool_call["name"] == "routing_tool":
            return Command(
                update=[
                    ToolMessage(
                        content="Routing to special handler",
                        tool_call_id=request.tool_call["id"],
                        name="routing_tool",
                    )
                ],
                goto="special_node",
            )
        return execute(request)

    node = ToolNode([registered_tool], wrap_tool_call=command_interceptor)

    result = node.invoke(
        [
            AIMessage(
                "",
                tool_calls=[
                    {
                        "name": "routing_tool",
                        "args": {},
                        "id": "1",
                        "type": "tool_call",
                    }
                ],
            )
        ],
        config=_create_config_with_runtime(),
    )

    # Should get Command back
    assert len(result) == 1
    assert isinstance(result[0], Command)
    assert result[0].goto == "special_node"
    assert result[0].update is not None
    assert len(result[0].update) == 1
    assert result[0].update[0].content == "Routing to special handler"


def test_interceptor_exception_with_unregistered_tool() -> None:
    """Test that interceptor exceptions are caught by error handling."""

    def failing_interceptor(
        request: ToolCallRequest,
        execute: Callable[[ToolCallRequest], ToolMessage | Command],
    ) -> ToolMessage | Command:
        """Interceptor that throws exception for unregistered tools."""
        if request.tool_call["name"] == "bad_tool":
            msg = "Interceptor failed"
            raise ValueError(msg)
        return execute(request)

    node = ToolNode(
        [registered_tool], wrap_tool_call=failing_interceptor, handle_tool_errors=True
    )

    # Interceptor exception should be caught and converted to error message
    result = node.invoke(
        [
            AIMessage(
                "",
                tool_calls=[
                    {
                        "name": "bad_tool",
                        "args": {},
                        "id": "1",
                        "type": "tool_call",
                    }
                ],
            )
        ],
        config=_create_config_with_runtime(),
    )

    assert len(result) == 1
    assert result[0].status == "error"
    assert "Interceptor failed" in result[0].content
    assert result[0].tool_call_id == "1"

    # Test that exception is raised when handle_tool_errors is False
    node_no_handling = ToolNode(
        [registered_tool], wrap_tool_call=failing_interceptor, handle_tool_errors=False
    )

    with pytest.raises(ValueError, match="Interceptor failed"):
        node_no_handling.invoke(
            [
                AIMessage(
                    "",
                    tool_calls=[
                        {
                            "name": "bad_tool",
                            "args": {},
                            "id": "2",
                            "type": "tool_call",
                        }
                    ],
                )
            ],
            config=_create_config_with_runtime(),
        )


async def test_async_interceptor_exception_with_unregistered_tool() -> None:
    """Test that async interceptor exceptions are caught by error handling."""

    async def failing_async_interceptor(
        request: ToolCallRequest,
        execute: Callable[[ToolCallRequest], Awaitable[ToolMessage | Command]],
    ) -> ToolMessage | Command:
        """Async interceptor that throws exception for unregistered tools."""
        if request.tool_call["name"] == "bad_async_tool":
            msg = "Async interceptor failed"
            raise RuntimeError(msg)
        return await execute(request)

    node = ToolNode(
        [registered_tool],
        awrap_tool_call=failing_async_interceptor,
        handle_tool_errors=True,
    )

    # Interceptor exception should be caught and converted to error message
    result = await node.ainvoke(
        [
            AIMessage(
                "",
                tool_calls=[
                    {
                        "name": "bad_async_tool",
                        "args": {},
                        "id": "1",
                        "type": "tool_call",
                    }
                ],
            )
        ],
        config=_create_config_with_runtime(),
    )

    assert len(result) == 1
    assert result[0].status == "error"
    assert "Async interceptor failed" in result[0].content
    assert result[0].tool_call_id == "1"

    # Test that exception is raised when handle_tool_errors is False
    node_no_handling = ToolNode(
        [registered_tool],
        awrap_tool_call=failing_async_interceptor,
        handle_tool_errors=False,
    )

    with pytest.raises(RuntimeError, match="Async interceptor failed"):
        await node_no_handling.ainvoke(
            [
                AIMessage(
                    "",
                    tool_calls=[
                        {
                            "name": "bad_async_tool",
                            "args": {},
                            "id": "2",
                            "type": "tool_call",
                        }
                    ],
                )
            ],
            config=_create_config_with_runtime(),
        )


def test_interceptor_with_dict_input_format() -> None:
    """Test that interceptor works with dict input format."""

    def interceptor(
        request: ToolCallRequest,
        execute: Callable[[ToolCallRequest], ToolMessage | Command],
    ) -> ToolMessage | Command:
        """Intercept unregistered tools with dict input."""
        if request.tool_call["name"] == "dict_tool":
            return ToolMessage(
                content="Handled dict input",
                tool_call_id=request.tool_call["id"],
                name="dict_tool",
            )
        return execute(request)

    node = ToolNode([registered_tool], wrap_tool_call=interceptor)

    # Test with dict input format
    result = node.invoke(
        {
            "messages": [
                AIMessage(
                    "",
                    tool_calls=[
                        {
                            "name": "dict_tool",
                            "args": {"value": 5},
                            "id": "1",
                            "type": "tool_call",
                        }
                    ],
                )
            ]
        },
        config=_create_config_with_runtime(),
    )

    # Should return dict format output
    assert isinstance(result, dict)
    assert "messages" in result
    assert len(result["messages"]) == 1
    assert result["messages"][0].content == "Handled dict input"
    assert result["messages"][0].tool_call_id == "1"


def test_interceptor_verifies_tool_is_none_for_unregistered() -> None:
    """Test that request.tool is None for unregistered tools."""

    captured_requests: list[ToolCallRequest] = []

    def capturing_interceptor(
        request: ToolCallRequest,
        execute: Callable[[ToolCallRequest], ToolMessage | Command],
    ) -> ToolMessage | Command:
        """Capture request to verify tool field."""
        captured_requests.append(request)
        if request.tool is None:
            # Tool is unregistered
            return ToolMessage(
                content=f"Unregistered: {request.tool_call['name']}",
                tool_call_id=request.tool_call["id"],
                name=request.tool_call["name"],
            )
        # Tool is registered
        return execute(request)

    node = ToolNode([registered_tool], wrap_tool_call=capturing_interceptor)

    # Test unregistered tool
    node.invoke(
        [
            AIMessage(
                "",
                tool_calls=[
                    {
                        "name": "unknown_tool",
                        "args": {},
                        "id": "1",
                        "type": "tool_call",
                    }
                ],
            )
        ],
        config=_create_config_with_runtime(),
    )

    assert len(captured_requests) == 1
    assert captured_requests[0].tool is None
    assert captured_requests[0].tool_call["name"] == "unknown_tool"

    # Clear and test registered tool
    captured_requests.clear()
    node.invoke(
        [
            AIMessage(
                "",
                tool_calls=[
                    {
                        "name": "registered_tool",
                        "args": {"x": 10},
                        "id": "2",
                        "type": "tool_call",
                    }
                ],
            )
        ],
        config=_create_config_with_runtime(),
    )

    assert len(captured_requests) == 1
    assert captured_requests[0].tool is not None
    assert captured_requests[0].tool.name == "registered_tool"

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/prebuilt/tests/test_tool_node_validation_error_filtering.py
```py
"""Unit tests for ValidationError filtering in ToolNode.

This module tests that validation errors are filtered to only include arguments
that the LLM controls. Injected arguments (InjectedState, InjectedStore,
ToolRuntime) are automatically provided by the system and should not appear in
validation error messages. This ensures the LLM receives focused, actionable
feedback about the parameters it can actually control, improving error correction
and reducing confusion from irrelevant system implementation details.
"""

from typing import Annotated
from unittest.mock import Mock

import pytest
from langchain_core.messages import AIMessage
from langchain_core.runnables.config import RunnableConfig
from langchain_core.tools import tool as dec_tool
from langgraph.store.base import BaseStore
from langgraph.store.memory import InMemoryStore

from langgraph.prebuilt import InjectedState, InjectedStore, ToolNode, ToolRuntime
from langgraph.prebuilt.tool_node import ToolInvocationError

pytestmark = pytest.mark.anyio


def _create_mock_runtime(store: BaseStore | None = None) -> Mock:
    """Create a mock Runtime object for testing ToolNode outside of graph context."""
    mock_runtime = Mock()
    mock_runtime.store = store
    mock_runtime.context = None
    mock_runtime.stream_writer = lambda *args, **kwargs: None
    return mock_runtime


def _create_config_with_runtime(store: BaseStore | None = None) -> RunnableConfig:
    """Create a RunnableConfig with mock Runtime for testing ToolNode."""
    return {"configurable": {"__pregel_runtime": _create_mock_runtime(store)}}


async def test_filter_injected_state_validation_errors() -> None:
    """Test that validation errors for InjectedState arguments are filtered out.

    InjectedState parameters are not controlled by the LLM, so any validation
    errors related to them should not appear in error messages. This ensures
    the LLM receives only actionable feedback about its own tool call arguments.
    """

    @dec_tool
    def my_tool(
        value: int,
        state: Annotated[dict, InjectedState],
    ) -> str:
        """Tool that uses injected state.

        Args:
            value: An integer value.
            state: The graph state (injected).
        """
        return f"value={value}, messages={len(state.get('messages', []))}"

    tool_node = ToolNode([my_tool])

    # Call with invalid 'value' argument (should be int, not str)
    result = await tool_node.ainvoke(
        {
            "messages": [
                AIMessage(
                    "hi?",
                    tool_calls=[
                        {
                            "name": "my_tool",
                            "args": {"value": "not_an_int"},  # Invalid type
                            "id": "call_1",
                            "type": "tool_call",
                        }
                    ],
                )
            ]
        },
        config=_create_config_with_runtime(),
    )

    # Should get a ToolMessage with error
    assert len(result["messages"]) == 1
    tool_message = result["messages"][0]
    assert tool_message.status == "error"
    assert tool_message.tool_call_id == "call_1"

    # Error should mention 'value' but NOT 'state' (which is injected)
    assert "value" in tool_message.content
    assert "state" not in tool_message.content.lower()


async def test_filter_injected_store_validation_errors() -> None:
    """Test that validation errors for InjectedStore arguments are filtered out.

    InjectedStore parameters are not controlled by the LLM, so any validation
    errors related to them should not appear in error messages. This keeps
    error feedback focused on LLM-controllable parameters.
    """

    @dec_tool
    def my_tool(
        key: str,
        store: Annotated[BaseStore, InjectedStore()],
    ) -> str:
        """Tool that uses injected store.

        Args:
            key: A key to look up.
            store: The persistent store (injected).
        """
        return f"key={key}"

    tool_node = ToolNode([my_tool])

    # Call with invalid 'key' argument (missing required argument)
    result = await tool_node.ainvoke(
        {
            "messages": [
                AIMessage(
                    "hi?",
                    tool_calls=[
                        {
                            "name": "my_tool",
                            "args": {},  # Missing 'key'
                            "id": "call_1",
                            "type": "tool_call",
                        }
                    ],
                )
            ]
        },
        config=_create_config_with_runtime(store=InMemoryStore()),
    )

    # Should get a ToolMessage with error
    assert len(result["messages"]) == 1
    tool_message = result["messages"][0]
    assert tool_message.status == "error"

    # Error should mention 'key' is required
    assert "key" in tool_message.content.lower()
    # The error should be about 'key' field specifically (not about store field)
    # Note: 'store' might appear in input_value representation, but the validation
    # error itself should only be for 'key'
    assert (
        "field required" in tool_message.content.lower()
        or "missing" in tool_message.content.lower()
    )


async def test_filter_tool_runtime_validation_errors() -> None:
    """Test that validation errors for ToolRuntime arguments are filtered out.

    ToolRuntime parameters are not controlled by the LLM, so any validation
    errors related to them should not appear in error messages. This ensures
    the LLM only sees errors for parameters it can fix.
    """

    @dec_tool
    def my_tool(
        query: str,
        runtime: ToolRuntime,
    ) -> str:
        """Tool that uses ToolRuntime.

        Args:
            query: A query string.
            runtime: The tool runtime context (injected).
        """
        return f"query={query}"

    tool_node = ToolNode([my_tool])

    # Call with invalid 'query' argument (wrong type)
    result = await tool_node.ainvoke(
        {
            "messages": [
                AIMessage(
                    "hi?",
                    tool_calls=[
                        {
                            "name": "my_tool",
                            "args": {"query": 123},  # Should be str, not int
                            "id": "call_1",
                            "type": "tool_call",
                        }
                    ],
                )
            ]
        },
        config=_create_config_with_runtime(),
    )

    # Should get a ToolMessage with error
    assert len(result["messages"]) == 1
    tool_message = result["messages"][0]
    assert tool_message.status == "error"

    # Error should mention 'query' but NOT 'runtime' (which is injected)
    assert "query" in tool_message.content.lower()
    assert "runtime" not in tool_message.content.lower()


async def test_filter_multiple_injected_args() -> None:
    """Test filtering when a tool has multiple injected arguments.

    When a tool uses multiple injected parameters (state, store, runtime), none of
    them should appear in validation error messages since they're all system-provided
    and not controlled by the LLM. Only LLM-controllable parameter errors should appear.
    """

    @dec_tool
    def my_tool(
        value: int,
        state: Annotated[dict, InjectedState],
        store: Annotated[BaseStore, InjectedStore()],
        runtime: ToolRuntime,
    ) -> str:
        """Tool with multiple injected arguments.

        Args:
            value: An integer value.
            state: The graph state (injected).
            store: The persistent store (injected).
            runtime: The tool runtime context (injected).
        """
        return f"value={value}"

    tool_node = ToolNode([my_tool])

    # Call with invalid 'value' - injected args should be filtered from error
    result = await tool_node.ainvoke(
        {
            "messages": [
                AIMessage(
                    "hi?",
                    tool_calls=[
                        {
                            "name": "my_tool",
                            "args": {"value": "not_an_int"},
                            "id": "call_1",
                            "type": "tool_call",
                        }
                    ],
                )
            ]
        },
        config=_create_config_with_runtime(store=InMemoryStore()),
    )

    tool_message = result["messages"][0]
    assert tool_message.status == "error"

    # Only 'value' error should be reported
    assert "value" in tool_message.content
    # None of the injected args should appear in error
    assert "state" not in tool_message.content.lower()
    assert "store" not in tool_message.content.lower()
    assert "runtime" not in tool_message.content.lower()


async def test_no_filtering_when_all_errors_are_model_args() -> None:
    """Test that validation errors for LLM-controlled arguments are preserved.

    When validation fails for arguments the LLM controls, those errors should
    be fully reported to help the LLM correct its tool calls. This ensures
    the LLM receives complete feedback about all issues it can fix.
    """

    @dec_tool
    def my_tool(
        value1: int,
        value2: str,
        state: Annotated[dict, InjectedState],
    ) -> str:
        """Tool with both regular and injected arguments.

        Args:
            value1: First value.
            value2: Second value.
            state: The graph state (injected).
        """
        return f"value1={value1}, value2={value2}"

    tool_node = ToolNode([my_tool])

    # Call with invalid arguments for BOTH non-injected parameters
    result = await tool_node.ainvoke(
        {
            "messages": [
                AIMessage(
                    "hi?",
                    tool_calls=[
                        {
                            "name": "my_tool",
                            "args": {
                                "value1": "not_an_int",  # Invalid
                                "value2": 456,  # Invalid (should be str)
                            },
                            "id": "call_1",
                            "type": "tool_call",
                        }
                    ],
                )
            ]
        },
        config=_create_config_with_runtime(),
    )

    tool_message = result["messages"][0]
    assert tool_message.status == "error"

    # Both errors should be present
    assert "value1" in tool_message.content
    assert "value2" in tool_message.content
    # Injected state should not appear
    assert "state" not in tool_message.content.lower()


async def test_validation_error_with_no_injected_args() -> None:
    """Test that tools without injected arguments show all validation errors.

    For tools that only have LLM-controlled parameters, all validation errors
    should be reported since everything is under the LLM's control and can be
    corrected by the LLM in subsequent tool calls.
    """

    @dec_tool
    def my_tool(value1: int, value2: str) -> str:
        """Regular tool without injected arguments.

        Args:
            value1: First value.
            value2: Second value.
        """
        return f"{value1} {value2}"

    tool_node = ToolNode([my_tool])

    result = await tool_node.ainvoke(
        {
            "messages": [
                AIMessage(
                    "hi?",
                    tool_calls=[
                        {
                            "name": "my_tool",
                            "args": {"value1": "invalid", "value2": 123},
                            "id": "call_1",
                            "type": "tool_call",
                        }
                    ],
                )
            ]
        },
        config=_create_config_with_runtime(),
    )

    tool_message = result["messages"][0]
    assert tool_message.status == "error"

    # Both errors should be present since there are no injected args to filter
    assert "value1" in tool_message.content
    assert "value2" in tool_message.content


async def test_tool_invocation_error_without_handle_errors() -> None:
    """Test that ToolInvocationError contains only LLM-controlled parameter errors.

    When handle_tool_errors is False, the raised ToolInvocationError should still
    filter out system-injected arguments from the error details, ensuring that
    error messages focus on what the LLM can control.
    """

    @dec_tool
    def my_tool(
        value: int,
        state: Annotated[dict, InjectedState],
    ) -> str:
        """Tool with injected state.

        Args:
            value: An integer value.
            state: The graph state (injected).
        """
        return f"value={value}"

    tool_node = ToolNode([my_tool], handle_tool_errors=False)

    # Should raise ToolInvocationError with filtered errors
    with pytest.raises(ToolInvocationError) as exc_info:
        await tool_node.ainvoke(
            {
                "messages": [
                    AIMessage(
                        "hi?",
                        tool_calls=[
                            {
                                "name": "my_tool",
                                "args": {"value": "not_an_int"},
                                "id": "call_1",
                                "type": "tool_call",
                            }
                        ],
                    )
                ]
            },
            config=_create_config_with_runtime(),
        )

    error = exc_info.value
    assert error.tool_name == "my_tool"
    assert error.filtered_errors is not None
    assert len(error.filtered_errors) > 0

    # Filtered errors should only contain 'value' error, not 'state'
    error_locs = [err["loc"] for err in error.filtered_errors]
    assert any("value" in str(loc) for loc in error_locs)
    assert not any("state" in str(loc) for loc in error_locs)


async def test_sync_tool_validation_error_filtering() -> None:
    """Test that error filtering works for sync tools.

    Error filtering should work identically for both sync and async tool execution,
    excluding injected arguments from validation error messages.
    """

    @dec_tool
    def my_tool(
        value: int,
        state: Annotated[dict, InjectedState],
    ) -> str:
        """Sync tool with injected state.

        Args:
            value: An integer value.
            state: The graph state (injected).
        """
        return f"value={value}"

    tool_node = ToolNode([my_tool])

    # Test sync invocation
    result = tool_node.invoke(
        {
            "messages": [
                AIMessage(
                    "hi?",
                    tool_calls=[
                        {
                            "name": "my_tool",
                            "args": {"value": "not_an_int"},
                            "id": "call_1",
                            "type": "tool_call",
                        }
                    ],
                )
            ]
        },
        config=_create_config_with_runtime(),
    )

    tool_message = result["messages"][0]
    assert tool_message.status == "error"
    assert "value" in tool_message.content
    assert "state" not in tool_message.content.lower()

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/prebuilt/tests/test_validation_node.py
```py
import sys
from typing import Any

import pytest
from langchain_core.messages import AIMessage
from langchain_core.tools import tool as dec_tool
from pydantic import BaseModel
from pydantic.v1 import BaseModel as BaseModelV1

from langgraph.prebuilt import ValidationNode

pytestmark = pytest.mark.anyio


def my_function(some_val: int, some_other_val: str) -> str:
    return f"{some_val} - {some_other_val}"


class MyModel(BaseModel):
    some_val: int
    some_other_val: str


class MyModelV1(BaseModelV1):
    some_val: int
    some_other_val: str


@dec_tool
def my_tool(some_val: int, some_other_val: str) -> str:
    """Cool."""
    return f"{some_val} - {some_other_val}"


@pytest.mark.parametrize(
    "tool_schema",
    [
        my_function,
        MyModel,
        pytest.param(
            MyModelV1,
            marks=pytest.mark.skipif(
                sys.version_info >= (3, 14),
                reason="Pydantic v1 not supported in Python 3.14+",
            ),
        ),
        my_tool,
    ],
)
@pytest.mark.parametrize("use_message_key", [True, False])
async def test_validation_node(tool_schema: Any, use_message_key: bool):
    validation_node = ValidationNode([tool_schema])
    tool_name = getattr(tool_schema, "name", getattr(tool_schema, "__name__", None))
    inputs = [
        AIMessage(
            "hi?",
            tool_calls=[
                {
                    "name": tool_name,
                    "args": {"some_val": 1, "some_other_val": "foo"},
                    "id": "some 0",
                },
                {
                    "name": tool_name,
                    # Wrong type for some_val
                    "args": {"some_val": "bar", "some_other_val": "foo"},
                    "id": "some 1",
                },
            ],
        ),
    ]
    if use_message_key:
        inputs = {"messages": inputs}
    result = await validation_node.ainvoke(inputs)
    if use_message_key:
        result = result["messages"]

    def check_results(messages: list):
        assert len(messages) == 2
        assert all(m.type == "tool" for m in messages)
        assert not messages[0].additional_kwargs.get("is_error")
        assert messages[1].additional_kwargs.get("is_error")

    check_results(result)
    result_sync = validation_node.invoke(inputs)
    if use_message_key:
        result_sync = result_sync["messages"]
    check_results(result_sync)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/prebuilt/tests/__snapshots__/test_react_agent_graph.ambr
```ambr
# serializer version: 1
# name: test_react_agent_graph_structure[None-None-None-tools0]
  '''
  graph TD;
  	__start__ --> agent;
  	agent --> __end__;
  
  '''
# ---
# name: test_react_agent_graph_structure[None-None-None-tools1]
  '''
  graph TD;
  	__start__ --> agent;
  	agent -.-> __end__;
  	agent -.-> tools;
  	tools --> agent;
  
  '''
# ---
# name: test_react_agent_graph_structure[None-None-pre_model_hook-tools0]
  '''
  graph TD;
  	__start__ --> pre_model_hook;
  	pre_model_hook --> agent;
  	agent --> __end__;
  
  '''
# ---
# name: test_react_agent_graph_structure[None-None-pre_model_hook-tools1]
  '''
  graph TD;
  	__start__ --> pre_model_hook;
  	agent -.-> __end__;
  	agent -.-> tools;
  	pre_model_hook --> agent;
  	tools --> pre_model_hook;
  
  '''
# ---
# name: test_react_agent_graph_structure[None-post_model_hook-None-tools0]
  '''
  graph TD;
  	__start__ --> agent;
  	agent --> post_model_hook;
  	post_model_hook --> __end__;
  
  '''
# ---
# name: test_react_agent_graph_structure[None-post_model_hook-None-tools1]
  '''
  graph TD;
  	__start__ --> agent;
  	agent --> post_model_hook;
  	post_model_hook -.-> __end__;
  	post_model_hook -.-> agent;
  	post_model_hook -.-> tools;
  	tools --> agent;
  
  '''
# ---
# name: test_react_agent_graph_structure[None-post_model_hook-pre_model_hook-tools0]
  '''
  graph TD;
  	__start__ --> pre_model_hook;
  	agent --> post_model_hook;
  	pre_model_hook --> agent;
  	post_model_hook --> __end__;
  
  '''
# ---
# name: test_react_agent_graph_structure[None-post_model_hook-pre_model_hook-tools1]
  '''
  graph TD;
  	__start__ --> pre_model_hook;
  	agent --> post_model_hook;
  	post_model_hook -.-> __end__;
  	post_model_hook -.-> pre_model_hook;
  	post_model_hook -.-> tools;
  	pre_model_hook --> agent;
  	tools --> pre_model_hook;
  
  '''
# ---
# name: test_react_agent_graph_structure[ResponseFormat-None-None-tools0]
  '''
  graph TD;
  	__start__ --> agent;
  	agent --> generate_structured_response;
  	generate_structured_response --> __end__;
  
  '''
# ---
# name: test_react_agent_graph_structure[ResponseFormat-None-None-tools1]
  '''
  graph TD;
  	__start__ --> agent;
  	agent -.-> generate_structured_response;
  	agent -.-> tools;
  	tools --> agent;
  	generate_structured_response --> __end__;
  
  '''
# ---
# name: test_react_agent_graph_structure[ResponseFormat-None-pre_model_hook-tools0]
  '''
  graph TD;
  	__start__ --> pre_model_hook;
  	agent --> generate_structured_response;
  	pre_model_hook --> agent;
  	generate_structured_response --> __end__;
  
  '''
# ---
# name: test_react_agent_graph_structure[ResponseFormat-None-pre_model_hook-tools1]
  '''
  graph TD;
  	__start__ --> pre_model_hook;
  	agent -.-> generate_structured_response;
  	agent -.-> tools;
  	pre_model_hook --> agent;
  	tools --> pre_model_hook;
  	generate_structured_response --> __end__;
  
  '''
# ---
# name: test_react_agent_graph_structure[ResponseFormat-post_model_hook-None-tools0]
  '''
  graph TD;
  	__start__ --> agent;
  	agent --> post_model_hook;
  	post_model_hook --> generate_structured_response;
  	generate_structured_response --> __end__;
  
  '''
# ---
# name: test_react_agent_graph_structure[ResponseFormat-post_model_hook-None-tools1]
  '''
  graph TD;
  	__start__ --> agent;
  	agent --> post_model_hook;
  	post_model_hook -.-> agent;
  	post_model_hook -.-> generate_structured_response;
  	post_model_hook -.-> tools;
  	tools --> agent;
  	generate_structured_response --> __end__;
  
  '''
# ---
# name: test_react_agent_graph_structure[ResponseFormat-post_model_hook-pre_model_hook-tools0]
  '''
  graph TD;
  	__start__ --> pre_model_hook;
  	agent --> post_model_hook;
  	post_model_hook --> generate_structured_response;
  	pre_model_hook --> agent;
  	generate_structured_response --> __end__;
  
  '''
# ---
# name: test_react_agent_graph_structure[ResponseFormat-post_model_hook-pre_model_hook-tools1]
  '''
  graph TD;
  	__start__ --> pre_model_hook;
  	agent --> post_model_hook;
  	post_model_hook -.-> generate_structured_response;
  	post_model_hook -.-> pre_model_hook;
  	post_model_hook -.-> tools;
  	pre_model_hook --> agent;
  	tools --> pre_model_hook;
  	generate_structured_response --> __end__;
  
  '''
# ---

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/LICENSE
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
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/Makefile
```text
.PHONY: test lint format test-integration update-schema

######################
# TESTING AND COVERAGE
######################

TEST?= "tests/unit_tests"
test:
	uv run pytest $(TEST)
test-integration:
	uv run pytest tests/integration_tests

######################
# LINTING AND FORMATTING
######################

# Define a variable for Python and notebook files.
PYTHON_FILES=.
MYPY_CACHE=.mypy_cache
lint format: PYTHON_FILES=.
lint_diff format_diff: PYTHON_FILES=$(shell git diff --name-only --relative --diff-filter=d main . | grep -E '\.py$$|\.ipynb$$')
lint_package: PYTHON_FILES=langgraph_cli
lint_tests: PYTHON_FILES=tests
lint_tests: MYPY_CACHE=.mypy_cache_test

lint lint_diff lint_package lint_tests:
	uv run ruff check .
	[ "$(PYTHON_FILES)" = "" ] || uv run ruff format $(PYTHON_FILES) --diff
	[ "$(PYTHON_FILES)" = "" ] || uv run ruff check --select I $(PYTHON_FILES)
	[ "$(PYTHON_FILES)" = "" ] || mkdir -p $(MYPY_CACHE) || uv run mypy $(PYTHON_FILES) --cache-dir $(MYPY_CACHE)

format format_diff:
	uv run ruff format $(PYTHON_FILES)
	uv run ruff check --select I --fix $(PYTHON_FILES)

update-schema:
	uv run python generate_schema.py

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/README.md
```md
# LangGraph CLI

The official command-line interface for LangGraph, providing tools to create, develop, and deploy LangGraph applications.

## Installation

Install via pip:
```bash
pip install langgraph-cli
```

For development mode with hot reloading:
```bash
pip install "langgraph-cli[inmem]"
```

## Commands

### `langgraph new` 🌱
Create a new LangGraph project from a template
```bash
langgraph new [PATH] --template TEMPLATE_NAME
```

### `langgraph dev` 🏃‍♀️
Run LangGraph API server in development mode with hot reloading
```bash
langgraph dev [OPTIONS]
  --host TEXT                 Host to bind to (default: 127.0.0.1)
  --port INTEGER             Port to bind to (default: 2024)
  --no-reload               Disable auto-reload
  --debug-port INTEGER      Enable remote debugging
  --no-browser             Skip opening browser window
  -c, --config FILE        Config file path (default: langgraph.json)
```

### `langgraph up` 🚀
Launch LangGraph API server in Docker
```bash
langgraph up [OPTIONS]
  -p, --port INTEGER        Port to expose (default: 8123)
  --wait                   Wait for services to start
  --watch                  Restart on file changes
  --verbose               Show detailed logs
  -c, --config FILE       Config file path
  -d, --docker-compose    Additional services file
```

### `langgraph build`
Build a Docker image for your LangGraph application
```bash
langgraph build -t IMAGE_TAG [OPTIONS]
  --platform TEXT          Target platforms (e.g., linux/amd64,linux/arm64)
  --pull / --no-pull      Use latest/local base image
  -c, --config FILE       Config file path
```

### `langgraph dockerfile`
Generate a Dockerfile for custom deployments
```bash
langgraph dockerfile SAVE_PATH [OPTIONS]
  -c, --config FILE       Config file path
```

## Configuration

The CLI uses a `langgraph.json` configuration file with these key settings:

```json
{
  "dependencies": ["langchain_openai", "./your_package"],  // Required: Package dependencies
  "graphs": {
    "my_graph": "./your_package/file.py:graph"            // Required: Graph definitions
  },
  "env": "./.env",                                        // Optional: Environment variables
  "python_version": "3.11",                               // Optional: Python version (3.11/3.12)
  "pip_config_file": "./pip.conf",                        // Optional: pip configuration
  "dockerfile_lines": []                                  // Optional: Additional Dockerfile commands
}
```

See the [full documentation](https://langchain-ai.github.io/langgraph/cloud/reference/cli/) for detailed configuration options.

## Development

To develop the CLI itself:

1. Clone the repository
2. Navigate to the CLI directory: `cd libs/cli`
3. Install development dependencies: `uv pip install`
4. Make your changes to the CLI code
5. Test your changes:
   ```bash
   # Run CLI commands directly
   uv run langgraph --help
   
   # Or use the examples
   cd examples
   uv pip install
   uv run langgraph dev  # or other commands
   ```

## License

This project is licensed under the terms specified in the repository's LICENSE file.

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/generate_schema.py
```py
#!/usr/bin/env python3
"""
Script to generate a JSON schema for the langgraph-cli Config class.

This script creates a schema.json file that can be referenced in langgraph.json files
to provide IDE autocompletion and validation.
"""

import inspect
import json
import textwrap
from pathlib import Path

import msgspec

from langgraph_cli.schemas import (
    AuthConfig,
    CheckpointerConfig,
    Config,
    ConfigurableHeaderConfig,
    CorsConfig,
    HttpConfig,
    IndexConfig,
    SecurityConfig,
    SerdeConfig,
    StoreConfig,
    ThreadTTLConfig,
    TTLConfig,
)


def add_descriptions_to_schema(schema, cls):
    """Add docstring descriptions to the schema properties."""
    if schema.get("description"):
        schema["description"] = inspect.cleandoc(schema["description"])
    elif class_doc := inspect.getdoc(cls):
        schema["description"] = inspect.cleandoc(class_doc)
    # Get attribute docstrings from the class
    attr_docs = {}

    # Also check class annotations for docstrings
    source_lines = inspect.getsourcelines(cls)[0]
    current_attr = None
    docstring_lines = []

    for line in source_lines:
        line = line.strip()

        # Check for attribute definition (TypedDict style)
        if ":" in line and not line.startswith("#") and not line.startswith('"""'):
            parts = line.split(":", 1)
            if len(parts) == 2 and parts[0].strip().isidentifier():
                # If we were collecting a docstring, save it for the previous attribute
                if current_attr and docstring_lines:
                    attr_docs[current_attr] = "\n".join(docstring_lines).strip('"')
                    docstring_lines = []

                current_attr = parts[0].strip()

        # Check for docstring after attribute
        elif line.startswith('"""') and current_attr:
            # Start or end of a docstring
            if len(line) > 3 and line.endswith('"""'):
                # Single line docstring
                attr_docs[current_attr] = line.strip('"')
                current_attr = None
            elif docstring_lines:
                # End of multi-line docstring
                docstring_lines.append(line.rstrip('"'))
                attr_docs[current_attr] = "\n".join(docstring_lines).strip('"')
                docstring_lines = []
                current_attr = None
            else:
                # Start of multi-line docstring
                docstring_lines.append(line.lstrip('"'))

        # Continue multi-line docstring
        elif docstring_lines and current_attr:
            docstring_lines.append(line.strip('"'))

    # Add the last docstring if there is one
    if current_attr and docstring_lines:
        attr_docs[current_attr] = "\n".join(docstring_lines).strip('"')

    # Add descriptions to properties
    if "properties" in schema:
        for prop_name, prop_schema in schema["properties"].items():
            # First try to get from attribute docstrings
            if prop_name in attr_docs and "description" not in prop_schema:
                prop_schema["description"] = textwrap.dedent(attr_docs[prop_name])
            # Fall back to class docstring parsing
            elif class_doc:
                for line in class_doc.split("\n"):
                    if line.strip().startswith(
                        f"{prop_name}:"
                    ) or line.strip().startswith(f'"{prop_name}"'):
                        description = line.split(":", 1)[1].strip()
                        if description and "description" not in prop_schema:
                            prop_schema["description"] = description
                            break

    # Recursively process nested definitions
    if "$defs" in schema:
        for def_name, def_schema in schema["$defs"].items():
            # Find the class that corresponds to this definition
            for potential_cls in [
                Config,
                StoreConfig,
                IndexConfig,
                AuthConfig,
                SecurityConfig,
                HttpConfig,
                CorsConfig,
                ThreadTTLConfig,
                CheckpointerConfig,
                SerdeConfig,
                TTLConfig,
                ConfigurableHeaderConfig,
            ]:
                if potential_cls.__name__ == def_name:
                    add_descriptions_to_schema(def_schema, potential_cls)
                    break

    return schema


def generate_schema():
    """Generate a JSON schema for the Config class using msgspec."""
    # Generate the basic schema
    schema = msgspec.json.schema(Config)

    # Add title and description
    schema["title"] = "LangGraph CLI Configuration"
    schema["description"] = "Configuration schema for langgraph-cli"

    # Add docstring descriptions
    schema = add_descriptions_to_schema(schema, Config)

    # Add constraint that only one of python_version or node_version should be specified
    config_schema = schema["$defs"]["Config"]

    # Create two subschemas: one with python_version and one with node_version
    # Define properties specific to Python projects
    python_specific_props = ["python_version", "pip_config_file"]
    # Define properties specific to Node.js projects
    node_specific_props = ["node_version"]
    # Define properties common to both project types
    common_props = [
        k
        for k in config_schema["properties"]
        if k not in python_specific_props and k not in node_specific_props
    ]

    # Create Python schema with python_version and pip_config_file
    python_schema = {
        "type": "object",
        "properties": {
            # Include Python-specific properties
            **{k: config_schema["properties"][k].copy() for k in python_specific_props},
            # Include common properties
            **{k: config_schema["properties"][k].copy() for k in common_props},
        },
        "required": ["dependencies", "graphs"],
    }

    # Add enum constraint for python_version
    if "python_version" in python_schema["properties"]:
        python_schema["properties"]["python_version"]["enum"] = ["3.11", "3.12", "3.13"]

    # Create Node.js schema with node_version
    node_schema = {
        "type": "object",
        "properties": {
            # Include Node-specific properties
            **{k: config_schema["properties"][k].copy() for k in node_specific_props},
            # Include common properties
            **{k: config_schema["properties"][k].copy() for k in common_props},
        },
        "required": ["node_version", "graphs"],
    }

    # Add enum constraint for node_version
    if "node_version" in node_schema["properties"]:
        node_schema["properties"]["node_version"]["anyOf"] = [
            {"type": "string", "enum": ["20"]},
            {"type": "null"},
        ]

    # Add enum constraint for image_distro
    if "image_distro" in node_schema["properties"]:
        node_schema["properties"]["image_distro"]["anyOf"] = [
            {"type": "string", "enum": ["debian", "wolfi"]},
            {"type": "null"},
        ]

    # Replace the Config schema with a oneOf constraint
    config_schema["oneOf"] = [python_schema, node_schema]

    # Remove the properties field as it's now defined in the oneOf subschemas
    if "properties" in config_schema:
        del config_schema["properties"]

    return schema


def main():
    """Generate the schema and write it to a file."""
    schema = generate_schema()

    # Add versioning to the schema
    import importlib.metadata

    try:
        version = importlib.metadata.version("langgraph_cli").split(".")
        schema_version = f"v{version[0]}"
    except importlib.metadata.PackageNotFoundError:
        schema_version = "v1"

    # Add version to schema
    schema["version"] = schema_version

    config_dir = Path(__file__).parent / "schemas"

    # Create versioned schema file
    versioned_path = config_dir / f"schema.{schema_version}.json"
    with open(versioned_path, "w") as f:
        json.dump(schema, f, indent=2)

    # Also create a latest version
    latest_path = config_dir / "schema.json"
    with open(latest_path, "w") as f:
        json.dump(schema, f, indent=2)

    print(f"Schema written to {versioned_path} and {latest_path}")
    print(
        f"You can now add '$schema: https://raw.githubusercontent.com/langchain-ai/langgraph/refs/heads/main/libs/cli/schemas/schema.json'"
        f" or '$schema: https://raw.githubusercontent.com/langchain-ai/langgraph/refs/heads/main/libs/cli/schemas/schema.{schema_version}.json'"
        " to your langgraph.json files"
    )


if __name__ == "__main__":
    main()

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/pyproject.toml
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "langgraph-cli"
dynamic = ["version"]
description = "CLI for interacting with LangGraph API"
authors = []
requires-python = ">=3.10"
readme = "README.md"
license = "MIT"
license-files = ['LICENSE']
dependencies = [
    "click>=8.1.7",
    "langgraph-sdk>=0.1.0 ; python_version >= '3.11'",
]
[tool.hatch.version]
path = "langgraph_cli/__init__.py"
[project.optional-dependencies]
inmem = [
    "langgraph-api>=0.4,<0.6.0 ; python_version >= '3.11'",
    "langgraph-runtime-inmem>=0.7 ; python_version >= '3.11'",
    "python-dotenv>=0.8.0",
]

[project.urls]
Source = "https://github.com/langchain-ai/langgraph/tree/main/libs/cli"
Twitter = "https://x.com/LangChainAI"
Slack = "https://www.langchain.com/join-community"
Reddit = "https://www.reddit.com/r/LangChain/"

[project.scripts]
langgraph = "langgraph_cli.cli:cli"

[dependency-groups]
test = [
    "pytest",
    "pytest-asyncio",
    "pytest-mock",
    "pytest-watch",
    "msgspec",
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

[tool.uv]
default-groups = ['dev']

[tool.hatch.build.targets.wheel]
include = ["langgraph_cli"]

[tool.pytest.ini_options]
addopts = "--strict-markers --strict-config --durations=5 -vv"
asyncio_mode = "auto"

[tool.ruff]
lint.select = [
  "E",  # pycodestyle
  "F",  # Pyflakes
  "UP", # pyupgrade
  "B",  # flake8-bugbear
  "I",  # isort
  "UP", # pyupgrade
]
lint.ignore = ["E501", "B008"]
target-version = "py310"

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/cli/uv.lock
```lock
version = 1
revision = 3
requires-python = ">=3.10"
resolution-markers = [
    "python_full_version >= '3.11'",
    "python_full_version < '3.11'",
]

[[package]]
name = "annotated-types"
version = "0.7.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/ee/67/531ea369ba64dcff5ec9c3402f9f51bf748cec26dde048a2f973a4eea7f5/annotated_types-0.7.0.tar.gz", hash = "sha256:aff07c09a53a08bc8cfccb9c85b05f1aa9a2a6f23728d790723543408344ce89", size = 16081, upload-time = "2024-05-20T21:33:25.928Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/78/b6/6307fbef88d9b5ee7421e68d78a9f162e0da4900bc5f5793f6d3d0e34fb8/annotated_types-0.7.0-py3-none-any.whl", hash = "sha256:1f02e8b43a8fbbc3f3e0d4f0f4bfc8131bcb4eebe8849b8e5c773f3a1c582a53", size = 13643, upload-time = "2024-05-20T21:33:24.1Z" },
]

[[package]]
name = "anyio"
version = "4.11.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "idna", marker = "python_full_version >= '3.11'" },
    { name = "sniffio", marker = "python_full_version >= '3.11'" },
    { name = "typing-extensions", marker = "python_full_version >= '3.11' and python_full_version < '3.13'" },
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
name = "blockbuster"
version = "1.5.25"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "forbiddenfruit", marker = "python_full_version >= '3.11' and implementation_name == 'cpython'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/7f/bc/57c49465decaeeedd58ce2d970b4cdfd93a74ba9993abff2dc498a31c283/blockbuster-1.5.25.tar.gz", hash = "sha256:b72f1d2aefdeecd2a820ddf1e1c8593bf00b96e9fdc4cd2199ebafd06f7cb8f0", size = 36058, upload-time = "2025-07-14T16:00:20.766Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/0b/01/dccc277c014f171f61a6047bb22c684e16c7f2db6bb5c8cce1feaf41ec55/blockbuster-1.5.25-py3-none-any.whl", hash = "sha256:cb06229762273e0f5f3accdaed3d2c5a3b61b055e38843de202311ede21bb0f5", size = 13196, upload-time = "2025-07-14T16:00:19.396Z" },
]

[[package]]
name = "certifi"
version = "2025.10.5"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/4c/5b/b6ce21586237c77ce67d01dc5507039d444b630dd76611bbca2d8e5dcd91/certifi-2025.10.5.tar.gz", hash = "sha256:47c09d31ccf2acf0be3f701ea53595ee7e0b8fa08801c6624be771df09ae7b43", size = 164519, upload-time = "2025-10-05T04:12:15.808Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/e4/37/af0d2ef3967ac0d6113837b44a4f0bfe1328c2b9763bd5b1744520e5cfed/certifi-2025.10.5-py3-none-any.whl", hash = "sha256:0f212c2744a9bb6de0c56639a6f68afe01ecd92d91f14ae897c4fe7bbeeef0de", size = 163286, upload-time = "2025-10-05T04:12:14.03Z" },
]

[[package]]
name = "cffi"
version = "2.0.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "pycparser", marker = "python_full_version >= '3.11' and implementation_name != 'PyPy'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/eb/56/b1ba7935a17738ae8453301356628e8147c79dbb825bcbc73dc7401f9846/cffi-2.0.0.tar.gz", hash = "sha256:44d1b5909021139fe36001ae048dbdde8214afa20200eda0f64c068cac5d5529", size = 523588, upload-time = "2025-09-08T23:24:04.541Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/93/d7/516d984057745a6cd96575eea814fe1edd6646ee6efd552fb7b0921dec83/cffi-2.0.0-cp310-cp310-macosx_10_13_x86_64.whl", hash = "sha256:0cf2d91ecc3fcc0625c2c530fe004f82c110405f101548512cce44322fa8ac44", size = 184283, upload-time = "2025-09-08T23:22:08.01Z" },
    { url = "https://files.pythonhosted.org/packages/9e/84/ad6a0b408daa859246f57c03efd28e5dd1b33c21737c2db84cae8c237aa5/cffi-2.0.0-cp310-cp310-macosx_11_0_arm64.whl", hash = "sha256:f73b96c41e3b2adedc34a7356e64c8eb96e03a3782b535e043a986276ce12a49", size = 180504, upload-time = "2025-09-08T23:22:10.637Z" },
    { url = "https://files.pythonhosted.org/packages/50/bd/b1a6362b80628111e6653c961f987faa55262b4002fcec42308cad1db680/cffi-2.0.0-cp310-cp310-manylinux1_i686.manylinux2014_i686.manylinux_2_17_i686.manylinux_2_5_i686.whl", hash = "sha256:53f77cbe57044e88bbd5ed26ac1d0514d2acf0591dd6bb02a3ae37f76811b80c", size = 208811, upload-time = "2025-09-08T23:22:12.267Z" },
    { url = "https://files.pythonhosted.org/packages/4f/27/6933a8b2562d7bd1fb595074cf99cc81fc3789f6a6c05cdabb46284a3188/cffi-2.0.0-cp310-cp310-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:3e837e369566884707ddaf85fc1744b47575005c0a229de3327f8f9a20f4efeb", size = 216402, upload-time = "2025-09-08T23:22:13.455Z" },
    { url = "https://files.pythonhosted.org/packages/05/eb/b86f2a2645b62adcfff53b0dd97e8dfafb5c8aa864bd0d9a2c2049a0d551/cffi-2.0.0-cp310-cp310-manylinux2014_ppc64le.manylinux_2_17_ppc64le.whl", hash = "sha256:5eda85d6d1879e692d546a078b44251cdd08dd1cfb98dfb77b670c97cee49ea0", size = 203217, upload-time = "2025-09-08T23:22:14.596Z" },
    { url = "https://files.pythonhosted.org/packages/9f/e0/6cbe77a53acf5acc7c08cc186c9928864bd7c005f9efd0d126884858a5fe/cffi-2.0.0-cp310-cp310-manylinux2014_s390x.manylinux_2_17_s390x.whl", hash = "sha256:9332088d75dc3241c702d852d4671613136d90fa6881da7d770a483fd05248b4", size = 203079, upload-time = "2025-09-08T23:22:15.769Z" },
    { url = "https://files.pythonhosted.org/packages/98/29/9b366e70e243eb3d14a5cb488dfd3a0b6b2f1fb001a203f653b93ccfac88/cffi-2.0.0-cp310-cp310-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:fc7de24befaeae77ba923797c7c87834c73648a05a4bde34b3b7e5588973a453", size = 216475, upload-time = "2025-09-08T23:22:17.427Z" },
    { url = "https://files.pythonhosted.org/packages/21/7a/13b24e70d2f90a322f2900c5d8e1f14fa7e2a6b3332b7309ba7b2ba51a5a/cffi-2.0.0-cp310-cp310-musllinux_1_2_aarch64.whl", hash = "sha256:cf364028c016c03078a23b503f02058f1814320a56ad535686f90565636a9495", size = 218829, upload-time = "2025-09-08T23:22:19.069Z" },
    { url = "https://files.pythonhosted.org/packages/60/99/c9dc110974c59cc981b1f5b66e1d8af8af764e00f0293266824d9c4254bc/cffi-2.0.0-cp310-cp310-musllinux_1_2_i686.whl", hash = "sha256:e11e82b744887154b182fd3e7e8512418446501191994dbf9c9fc1f32cc8efd5", size = 211211, upload-time = "2025-09-08T23:22:20.588Z" },
    { url = "https://files.pythonhosted.org/packages/49/72/ff2d12dbf21aca1b32a40ed792ee6b40f6dc3a9cf1644bd7ef6e95e0ac5e/cffi-2.0.0-cp310-cp310-musllinux_1_2_x86_64.whl", hash = "sha256:8ea985900c5c95ce9db1745f7933eeef5d314f0565b27625d9a10ec9881e1bfb", size = 218036, upload-time = "2025-09-08T23:22:22.143Z" },
    { url = "https://files.pythonhosted.org/packages/e2/cc/027d7fb82e58c48ea717149b03bcadcbdc293553edb283af792bd4bcbb3f/cffi-2.0.0-cp310-cp310-win32.whl", hash = "sha256:1f72fb8906754ac8a2cc3f9f5aaa298070652a0ffae577e0ea9bd480dc3c931a", size = 172184, upload-time = "2025-09-08T23:22:23.328Z" },
    { url = "https://files.pythonhosted.org/packages/33/fa/072dd15ae27fbb4e06b437eb6e944e75b068deb09e2a2826039e49ee2045/cffi-2.0.0-cp310-cp310-win_amd64.whl", hash = "sha256:b18a3ed7d5b3bd8d9ef7a8cb226502c6bf8308df1525e1cc676c3680e7176739", size = 182790, upload-time = "2025-09-08T23:22:24.752Z" },
    { url = "https://files.pythonhosted.org/packages/12/4a/3dfd5f7850cbf0d06dc84ba9aa00db766b52ca38d8b86e3a38314d52498c/cffi-2.0.0-cp311-cp311-macosx_10_13_x86_64.whl", hash = "sha256:b4c854ef3adc177950a8dfc81a86f5115d2abd545751a304c5bcf2c2c7283cfe", size = 184344, upload-time = "2025-09-08T23:22:26.456Z" },
    { url = "https://files.pythonhosted.org/packages/4f/8b/f0e4c441227ba756aafbe78f117485b25bb26b1c059d01f137fa6d14896b/cffi-2.0.0-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:2de9a304e27f7596cd03d16f1b7c72219bd944e99cc52b84d0145aefb07cbd3c", size = 180560, upload-time = "2025-09-08T23:22:28.197Z" },
    { url = "https://files.pythonhosted.org/packages/b1/b7/1200d354378ef52ec227395d95c2576330fd22a869f7a70e88e1447eb234/cffi-2.0.0-cp311-cp311-manylinux1_i686.manylinux2014_i686.manylinux_2_17_i686.manylinux_2_5_i686.whl", hash = "sha256:baf5215e0ab74c16e2dd324e8ec067ef59e41125d3eade2b863d294fd5035c92", size = 209613, upload-time = "2025-09-08T23:22:29.475Z" },
    { url = "https://files.pythonhosted.org/packages/b8/56/6033f5e86e8cc9bb629f0077ba71679508bdf54a9a5e112a3c0b91870332/cffi-2.0.0-cp311-cp311-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:730cacb21e1bdff3ce90babf007d0a0917cc3e6492f336c2f0134101e0944f93", size = 216476, upload-time = "2025-09-08T23:22:31.063Z" },
    { url = "https://files.pythonhosted.org/packages/dc/7f/55fecd70f7ece178db2f26128ec41430d8720f2d12ca97bf8f0a628207d5/cffi-2.0.0-cp311-cp311-manylinux2014_ppc64le.manylinux_2_17_ppc64le.whl", hash = "sha256:6824f87845e3396029f3820c206e459ccc91760e8fa24422f8b0c3d1731cbec5", size = 203374, upload-time = "2025-09-08T23:22:32.507Z" },
    { url = "https://files.pythonhosted.org/packages/84/ef/a7b77c8bdc0f77adc3b46888f1ad54be8f3b7821697a7b89126e829e676a/cffi-2.0.0-cp311-cp311-manylinux2014_s390x.manylinux_2_17_s390x.whl", hash = "sha256:9de40a7b0323d889cf8d23d1ef214f565ab154443c42737dfe52ff82cf857664", size = 202597, upload-time = "2025-09-08T23:22:34.132Z" },
    { url = "https://files.pythonhosted.org/packages/d7/91/500d892b2bf36529a75b77958edfcd5ad8e2ce4064ce2ecfeab2125d72d1/cffi-2.0.0-cp311-cp311-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:8941aaadaf67246224cee8c3803777eed332a19d909b47e29c9842ef1e79ac26", size = 215574, upload-time = "2025-09-08T23:22:35.443Z" },
    { url = "https://files.pythonhosted.org/packages/44/64/58f6255b62b101093d5df22dcb752596066c7e89dd725e0afaed242a61be/cffi-2.0.0-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:a05d0c237b3349096d3981b727493e22147f934b20f6f125a3eba8f994bec4a9", size = 218971, upload-time = "2025-09-08T23:22:36.805Z" },
    { url = "https://files.pythonhosted.org/packages/ab/49/fa72cebe2fd8a55fbe14956f9970fe8eb1ac59e5df042f603ef7c8ba0adc/cffi-2.0.0-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:94698a9c5f91f9d138526b48fe26a199609544591f859c870d477351dc7b2414", size = 211972, upload-time = "2025-09-08T23:22:38.436Z" },
    { url = "https://files.pythonhosted.org/packages/0b/28/dd0967a76aab36731b6ebfe64dec4e981aff7e0608f60c2d46b46982607d/cffi-2.0.0-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:5fed36fccc0612a53f1d4d9a816b50a36702c28a2aa880cb8a122b3466638743", size = 217078, upload-time = "2025-09-08T23:22:39.776Z" },
    { url = "https://files.pythonhosted.org/packages/2b/c0/015b25184413d7ab0a410775fdb4a50fca20f5589b5dab1dbbfa3baad8ce/cffi-2.0.0-cp311-cp311-win32.whl", hash = "sha256:c649e3a33450ec82378822b3dad03cc228b8f5963c0c12fc3b1e0ab940f768a5", size = 172076, upload-time = "2025-09-08T23:22:40.95Z" },
    { url = "https://files.pythonhosted.org/packages/ae/8f/dc5531155e7070361eb1b7e4c1a9d896d0cb21c49f807a6c03fd63fc877e/cffi-2.0.0-cp311-cp311-win_amd64.whl", hash = "sha256:66f011380d0e49ed280c789fbd08ff0d40968ee7b665575489afa95c98196ab5", size = 182820, upload-time = "2025-09-08T23:22:42.463Z" },
    { url = "https://files.pythonhosted.org/packages/95/5c/1b493356429f9aecfd56bc171285a4c4ac8697f76e9bbbbb105e537853a1/cffi-2.0.0-cp311-cp311-win_arm64.whl", hash = "sha256:c6638687455baf640e37344fe26d37c404db8b80d037c3d29f58fe8d1c3b194d", size = 177635, upload-time = "2025-09-08T23:22:43.623Z" },
    { url = "https://files.pythonhosted.org/packages/ea/47/4f61023ea636104d4f16ab488e268b93008c3d0bb76893b1b31db1f96802/cffi-2.0.0-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:6d02d6655b0e54f54c4ef0b94eb6be0607b70853c45ce98bd278dc7de718be5d", size = 185271, upload-time = "2025-09-08T23:22:44.795Z" },
    { url = "https://files.pythonhosted.org/packages/df/a2/781b623f57358e360d62cdd7a8c681f074a71d445418a776eef0aadb4ab4/cffi-2.0.0-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:8eca2a813c1cb7ad4fb74d368c2ffbbb4789d377ee5bb8df98373c2cc0dee76c", size = 181048, upload-time = "2025-09-08T23:22:45.938Z" },
    { url = "https://files.pythonhosted.org/packages/ff/df/a4f0fbd47331ceeba3d37c2e51e9dfc9722498becbeec2bd8bc856c9538a/cffi-2.0.0-cp312-cp312-manylinux1_i686.manylinux2014_i686.manylinux_2_17_i686.manylinux_2_5_i686.whl", hash = "sha256:21d1152871b019407d8ac3985f6775c079416c282e431a4da6afe7aefd2bccbe", size = 212529, upload-time = "2025-09-08T23:22:47.349Z" },
    { url = "https://files.pythonhosted.org/packages/d5/72/12b5f8d3865bf0f87cf1404d8c374e7487dcf097a1c91c436e72e6badd83/cffi-2.0.0-cp312-cp312-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:b21e08af67b8a103c71a250401c78d5e0893beff75e28c53c98f4de42f774062", size = 220097, upload-time = "2025-09-08T23:22:48.677Z" },
    { url = "https://files.pythonhosted.org/packages/c2/95/7a135d52a50dfa7c882ab0ac17e8dc11cec9d55d2c18dda414c051c5e69e/cffi-2.0.0-cp312-cp312-manylinux2014_ppc64le.manylinux_2_17_ppc64le.whl", hash = "sha256:1e3a615586f05fc4065a8b22b8152f0c1b00cdbc60596d187c2a74f9e3036e4e", size = 207983, upload-time = "2025-09-08T23:22:50.06Z" },
    { url = "https://files.pythonhosted.org/packages/3a/c8/15cb9ada8895957ea171c62dc78ff3e99159ee7adb13c0123c001a2546c1/cffi-2.0.0-cp312-cp312-manylinux2014_s390x.manylinux_2_17_s390x.whl", hash = "sha256:81afed14892743bbe14dacb9e36d9e0e504cd204e0b165062c488942b9718037", size = 206519, upload-time = "2025-09-08T23:22:51.364Z" },
    { url = "https://files.pythonhosted.org/packages/78/2d/7fa73dfa841b5ac06c7b8855cfc18622132e365f5b81d02230333ff26e9e/cffi-2.0.0-cp312-cp312-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:3e17ed538242334bf70832644a32a7aae3d83b57567f9fd60a26257e992b79ba", size = 219572, upload-time = "2025-09-08T23:22:52.902Z" },
    { url = "https://files.pythonhosted.org/packages/07/e0/267e57e387b4ca276b90f0434ff88b2c2241ad72b16d31836adddfd6031b/cffi-2.0.0-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:3925dd22fa2b7699ed2617149842d2e6adde22b262fcbfada50e3d195e4b3a94", size = 222963, upload-time = "2025-09-08T23:22:54.518Z" },
    { url = "https://files.pythonhosted.org/packages/b6/75/1f2747525e06f53efbd878f4d03bac5b859cbc11c633d0fb81432d98a795/cffi-2.0.0-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:2c8f814d84194c9ea681642fd164267891702542f028a15fc97d4674b6206187", size = 221361, upload-time = "2025-09-08T23:22:55.867Z" },
    { url = "https://files.pythonhosted.org/packages/7b/2b/2b6435f76bfeb6bbf055596976da087377ede68df465419d192acf00c437/cffi-2.0.0-cp312-cp312-win32.whl", hash = "sha256:da902562c3e9c550df360bfa53c035b2f241fed6d9aef119048073680ace4a18", size = 172932, upload-time = "2025-09-08T23:22:57.188Z" },
    { url = "https://files.pythonhosted.org/packages/f8/ed/13bd4418627013bec4ed6e54283b1959cf6db888048c7cf4b4c3b5b36002/cffi-2.0.0-cp312-cp312-win_amd64.whl", hash = "sha256:da68248800ad6320861f129cd9c1bf96ca849a2771a59e0344e88681905916f5", size = 183557, upload-time = "2025-09-08T23:22:58.351Z" },
    { url = "https://files.pythonhosted.org/packages/95/31/9f7f93ad2f8eff1dbc1c3656d7ca5bfd8fb52c9d786b4dcf19b2d02217fa/cffi-2.0.0-cp312-cp312-win_arm64.whl", hash = "sha256:4671d9dd5ec934cb9a73e7ee9676f9362aba54f7f34910956b84d727b0d73fb6", size = 177762, upload-time = "2025-09-08T23:22:59.668Z" },
    { url = "https://files.pythonhosted.org/packages/4b/8d/a0a47a0c9e413a658623d014e91e74a50cdd2c423f7ccfd44086ef767f90/cffi-2.0.0-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:00bdf7acc5f795150faa6957054fbbca2439db2f775ce831222b66f192f03beb", size = 185230, upload-time = "2025-09-08T23:23:00.879Z" },
    { url = "https://files.pythonhosted.org/packages/4a/d2/a6c0296814556c68ee32009d9c2ad4f85f2707cdecfd7727951ec228005d/cffi-2.0.0-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:45d5e886156860dc35862657e1494b9bae8dfa63bf56796f2fb56e1679fc0bca", size = 181043, upload-time = "2025-09-08T23:23:02.231Z" },
    { url = "https://files.pythonhosted.org/packages/b0/1e/d22cc63332bd59b06481ceaac49d6c507598642e2230f201649058a7e704/cffi-2.0.0-cp313-cp313-manylinux1_i686.manylinux2014_i686.manylinux_2_17_i686.manylinux_2_5_i686.whl", hash = "sha256:07b271772c100085dd28b74fa0cd81c8fb1a3ba18b21e03d7c27f3436a10606b", size = 212446, upload-time = "2025-09-08T23:23:03.472Z" },
    { url = "https://files.pythonhosted.org/packages/a9/f5/a2c23eb03b61a0b8747f211eb716446c826ad66818ddc7810cc2cc19b3f2/cffi-2.0.0-cp313-cp313-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:d48a880098c96020b02d5a1f7d9251308510ce8858940e6fa99ece33f610838b", size = 220101, upload-time = "2025-09-08T23:23:04.792Z" },
    { url = "https://files.pythonhosted.org/packages/f2/7f/e6647792fc5850d634695bc0e6ab4111ae88e89981d35ac269956605feba/cffi-2.0.0-cp313-cp313-manylinux2014_ppc64le.manylinux_2_17_ppc64le.whl", hash = "sha256:f93fd8e5c8c0a4aa1f424d6173f14a892044054871c771f8566e4008eaa359d2", size = 207948, upload-time = "2025-09-08T23:23:06.127Z" },
    { url = "https://files.pythonhosted.org/packages/cb/1e/a5a1bd6f1fb30f22573f76533de12a00bf274abcdc55c8edab639078abb6/cffi-2.0.0-cp313-cp313-manylinux2014_s390x.manylinux_2_17_s390x.whl", hash = "sha256:dd4f05f54a52fb558f1ba9f528228066954fee3ebe629fc1660d874d040ae5a3", size = 206422, upload-time = "2025-09-08T23:23:07.753Z" },
    { url = "https://files.pythonhosted.org/packages/98/df/0a1755e750013a2081e863e7cd37e0cdd02664372c754e5560099eb7aa44/cffi-2.0.0-cp313-cp313-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:c8d3b5532fc71b7a77c09192b4a5a200ea992702734a2e9279a37f2478236f26", size = 219499, upload-time = "2025-09-08T23:23:09.648Z" },
    { url = "https://files.pythonhosted.org/packages/50/e1/a969e687fcf9ea58e6e2a928ad5e2dd88cc12f6f0ab477e9971f2309b57c/cffi-2.0.0-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:d9b29c1f0ae438d5ee9acb31cadee00a58c46cc9c0b2f9038c6b0b3470877a8c", size = 222928, upload-time = "2025-09-08T23:23:10.928Z" },
    { url = "https://files.pythonhosted.org/packages/36/54/0362578dd2c9e557a28ac77698ed67323ed5b9775ca9d3fe73fe191bb5d8/cffi-2.0.0-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:6d50360be4546678fc1b79ffe7a66265e28667840010348dd69a314145807a1b", size = 221302, upload-time = "2025-09-08T23:23:12.42Z" },
    { url = "https://files.pythonhosted.org/packages/eb/6d/bf9bda840d5f1dfdbf0feca87fbdb64a918a69bca42cfa0ba7b137c48cb8/cffi-2.0.0-cp313-cp313-win32.whl", hash = "sha256:74a03b9698e198d47562765773b4a8309919089150a0bb17d829ad7b44b60d27", size = 172909, upload-time = "2025-09-08T23:23:14.32Z" },
    { url = "https://files.pythonhosted.org/packages/37/18/6519e1ee6f5a1e579e04b9ddb6f1676c17368a7aba48299c3759bbc3c8b3/cffi-2.0.0-cp313-cp313-win_amd64.whl", hash = "sha256:19f705ada2530c1167abacb171925dd886168931e0a7b78f5bffcae5c6b5be75", size = 183402, upload-time = "2025-09-08T23:23:15.535Z" },
    { url = "https://files.pythonhosted.org/packages/cb/0e/02ceeec9a7d6ee63bb596121c2c8e9b3a9e150936f4fbef6ca1943e6137c/cffi-2.0.0-cp313-cp313-win_arm64.whl", hash = "sha256:256f80b80ca3853f90c21b23ee78cd008713787b1b1e93eae9f3d6a7134abd91", size = 177780, upload-time = "2025-09-08T23:23:16.761Z" },
    { url = "https://files.pythonhosted.org/packages/92/c4/3ce07396253a83250ee98564f8d7e9789fab8e58858f35d07a9a2c78de9f/cffi-2.0.0-cp314-cp314-macosx_10_13_x86_64.whl", hash = "sha256:fc33c5141b55ed366cfaad382df24fe7dcbc686de5be719b207bb248e3053dc5", size = 185320, upload-time = "2025-09-08T23:23:18.087Z" },
    { url = "https://files.pythonhosted.org/packages/59/dd/27e9fa567a23931c838c6b02d0764611c62290062a6d4e8ff7863daf9730/cffi-2.0.0-cp314-cp314-macosx_11_0_arm64.whl", hash = "sha256:c654de545946e0db659b3400168c9ad31b5d29593291482c43e3564effbcee13", size = 181487, upload-time = "2025-09-08T23:23:19.622Z" },
    { url = "https://files.pythonhosted.org/packages/d6/43/0e822876f87ea8a4ef95442c3d766a06a51fc5298823f884ef87aaad168c/cffi-2.0.0-cp314-cp314-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:24b6f81f1983e6df8db3adc38562c83f7d4a0c36162885ec7f7b77c7dcbec97b", size = 220049, upload-time = "2025-09-08T23:23:20.853Z" },
    { url = "https://files.pythonhosted.org/packages/b4/89/76799151d9c2d2d1ead63c2429da9ea9d7aac304603de0c6e8764e6e8e70/cffi-2.0.0-cp314-cp314-manylinux2014_ppc64le.manylinux_2_17_ppc64le.whl", hash = "sha256:12873ca6cb9b0f0d3a0da705d6086fe911591737a59f28b7936bdfed27c0d47c", size = 207793, upload-time = "2025-09-08T23:23:22.08Z" },
    { url = "https://files.pythonhosted.org/packages/bb/dd/3465b14bb9e24ee24cb88c9e3730f6de63111fffe513492bf8c808a3547e/cffi-2.0.0-cp314-cp314-manylinux2014_s390x.manylinux_2_17_s390x.whl", hash = "sha256:d9b97165e8aed9272a6bb17c01e3cc5871a594a446ebedc996e2397a1c1ea8ef", size = 206300, upload-time = "2025-09-08T23:23:23.314Z" },
    { url = "https://files.pythonhosted.org/packages/47/d9/d83e293854571c877a92da46fdec39158f8d7e68da75bf73581225d28e90/cffi-2.0.0-cp314-cp314-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:afb8db5439b81cf9c9d0c80404b60c3cc9c3add93e114dcae767f1477cb53775", size = 219244, upload-time = "2025-09-08T23:23:24.541Z" },
    { url = "https://files.pythonhosted.org/packages/2b/0f/1f177e3683aead2bb00f7679a16451d302c436b5cbf2505f0ea8146ef59e/cffi-2.0.0-cp314-cp314-musllinux_1_2_aarch64.whl", hash = "sha256:737fe7d37e1a1bffe70bd5754ea763a62a066dc5913ca57e957824b72a85e205", size = 222828, upload-time = "2025-09-08T23:23:26.143Z" },
    { url = "https://files.pythonhosted.org/packages/c6/0f/cafacebd4b040e3119dcb32fed8bdef8dfe94da653155f9d0b9dc660166e/cffi-2.0.0-cp314-cp314-musllinux_1_2_x86_64.whl", hash = "sha256:38100abb9d1b1435bc4cc340bb4489635dc2f0da7456590877030c9b3d40b0c1", size = 220926, upload-time = "2025-09-08T23:23:27.873Z" },
    { url = "https://files.pythonhosted.org/packages/3e/aa/df335faa45b395396fcbc03de2dfcab242cd61a9900e914fe682a59170b1/cffi-2.0.0-cp314-cp314-win32.whl", hash = "sha256:087067fa8953339c723661eda6b54bc98c5625757ea62e95eb4898ad5e776e9f", size = 175328, upload-time = "2025-09-08T23:23:44.61Z" },
    { url = "https://files.pythonhosted.org/packages/bb/92/882c2d30831744296ce713f0feb4c1cd30f346ef747b530b5318715cc367/cffi-2.0.0-cp314-cp314-win_amd64.whl", hash = "sha256:203a48d1fb583fc7d78a4c6655692963b860a417c0528492a6bc21f1aaefab25", size = 185650, upload-time = "2025-09-08T23:23:45.848Z" },
    { url = "https://files.pythonhosted.org/packages/9f/2c/98ece204b9d35a7366b5b2c6539c350313ca13932143e79dc133ba757104/cffi-2.0.0-cp314-cp314-win_arm64.whl", hash = "sha256:dbd5c7a25a7cb98f5ca55d258b103a2054f859a46ae11aaf23134f9cc0d356ad", size = 180687, upload-time = "2025-09-08T23:23:47.105Z" },
    { url = "https://files.pythonhosted.org/packages/3e/61/c768e4d548bfa607abcda77423448df8c471f25dbe64fb2ef6d555eae006/cffi-2.0.0-cp314-cp314t-macosx_10_13_x86_64.whl", hash = "sha256:9a67fc9e8eb39039280526379fb3a70023d77caec1852002b4da7e8b270c4dd9", size = 188773, upload-time = "2025-09-08T23:23:29.347Z" },
    { url = "https://files.pythonhosted.org/packages/2c/ea/5f76bce7cf6fcd0ab1a1058b5af899bfbef198bea4d5686da88471ea0336/cffi-2.0.0-cp314-cp314t-macosx_11_0_arm64.whl", hash = "sha256:7a66c7204d8869299919db4d5069a82f1561581af12b11b3c9f48c584eb8743d", size = 185013, upload-time = "2025-09-08T23:23:30.63Z" },
    { url = "https://files.pythonhosted.org/packages/be/b4/c56878d0d1755cf9caa54ba71e5d049479c52f9e4afc230f06822162ab2f/cffi-2.0.0-cp314-cp314t-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:7cc09976e8b56f8cebd752f7113ad07752461f48a58cbba644139015ac24954c", size = 221593, upload-time = "2025-09-08T23:23:31.91Z" },
    { url = "https://files.pythonhosted.org/packages/e0/0d/eb704606dfe8033e7128df5e90fee946bbcb64a04fcdaa97321309004000/cffi-2.0.0-cp314-cp314t-manylinux2014_ppc64le.manylinux_2_17_ppc64le.whl", hash = "sha256:92b68146a71df78564e4ef48af17551a5ddd142e5190cdf2c5624d0c3ff5b2e8", size = 209354, upload-time = "2025-09-08T23:23:33.214Z" },
    { url = "https://files.pythonhosted.org/packages/d8/19/3c435d727b368ca475fb8742ab97c9cb13a0de600ce86f62eab7fa3eea60/cffi-2.0.0-cp314-cp314t-manylinux2014_s390x.manylinux_2_17_s390x.whl", hash = "sha256:b1e74d11748e7e98e2f426ab176d4ed720a64412b6a15054378afdb71e0f37dc", size = 208480, upload-time = "2025-09-08T23:23:34.495Z" },
    { url = "https://files.pythonhosted.org/packages/d0/44/681604464ed9541673e486521497406fadcc15b5217c3e326b061696899a/cffi-2.0.0-cp314-cp314t-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:28a3a209b96630bca57cce802da70c266eb08c6e97e5afd61a75611ee6c64592", size = 221584, upload-time = "2025-09-08T23:23:36.096Z" },
    { url = "https://files.pythonhosted.org/packages/25/8e/342a504ff018a2825d395d44d63a767dd8ebc927ebda557fecdaca3ac33a/cffi-2.0.0-cp314-cp314t-musllinux_1_2_aarch64.whl", hash = "sha256:7553fb2090d71822f02c629afe6042c299edf91ba1bf94951165613553984512", size = 224443, upload-time = "2025-09-08T23:23:37.328Z" },
    { url = "https://files.pythonhosted.org/packages/e1/5e/b666bacbbc60fbf415ba9988324a132c9a7a0448a9a8f125074671c0f2c3/cffi-2.0.0-cp314-cp314t-musllinux_1_2_x86_64.whl", hash = "sha256:6c6c373cfc5c83a975506110d17457138c8c63016b563cc9ed6e056a82f13ce4", size = 223437, upload-time = "2025-09-08T23:23:38.945Z" },
    { url = "https://files.pythonhosted.org/packages/a0/1d/ec1a60bd1a10daa292d3cd6bb0b359a81607154fb8165f3ec95fe003b85c/cffi-2.0.0-cp314-cp314t-win32.whl", hash = "sha256:1fc9ea04857caf665289b7a75923f2c6ed559b8298a1b8c49e59f7dd95c8481e", size = 180487, upload-time = "2025-09-08T23:23:40.423Z" },
    { url = "https://files.pythonhosted.org/packages/bf/41/4c1168c74fac325c0c8156f04b6749c8b6a8f405bbf91413ba088359f60d/cffi-2.0.0-cp314-cp314t-win_amd64.whl", hash = "sha256:d68b6cef7827e8641e8ef16f4494edda8b36104d79773a334beaa1e3521430f6", size = 191726, upload-time = "2025-09-08T23:23:41.742Z" },
    { url = "https://files.pythonhosted.org/packages/ae/3a/dbeec9d1ee0844c679f6bb5d6ad4e9f198b1224f4e7a32825f47f6192b0c/cffi-2.0.0-cp314-cp314t-win_arm64.whl", hash = "sha256:0a1527a803f0a659de1af2e1fd700213caba79377e27e4693648c2923da066f9", size = 184195, upload-time = "2025-09-08T23:23:43.004Z" },
]

[[package]]
name = "charset-normalizer"
version = "3.4.4"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/13/69/33ddede1939fdd074bce5434295f38fae7136463422fe4fd3e0e89b98062/charset_normalizer-3.4.4.tar.gz", hash = "sha256:94537985111c35f28720e43603b8e7b43a6ecfb2ce1d3058bbe955b73404e21a", size = 129418, upload-time = "2025-10-14T04:42:32.879Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/1f/b8/6d51fc1d52cbd52cd4ccedd5b5b2f0f6a11bbf6765c782298b0f3e808541/charset_normalizer-3.4.4-cp310-cp310-macosx_10_9_universal2.whl", hash = "sha256:e824f1492727fa856dd6eda4f7cee25f8518a12f3c4a56a74e8095695089cf6d", size = 209709, upload-time = "2025-10-14T04:40:11.385Z" },
    { url = "https://files.pythonhosted.org/packages/5c/af/1f9d7f7faafe2ddfb6f72a2e07a548a629c61ad510fe60f9630309908fef/charset_normalizer-3.4.4-cp310-cp310-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:4bd5d4137d500351a30687c2d3971758aac9a19208fc110ccb9d7188fbe709e8", size = 148814, upload-time = "2025-10-14T04:40:13.135Z" },
    { url = "https://files.pythonhosted.org/packages/79/3d/f2e3ac2bbc056ca0c204298ea4e3d9db9b4afe437812638759db2c976b5f/charset_normalizer-3.4.4-cp310-cp310-manylinux2014_armv7l.manylinux_2_17_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:027f6de494925c0ab2a55eab46ae5129951638a49a34d87f4c3eda90f696b4ad", size = 144467, upload-time = "2025-10-14T04:40:14.728Z" },
    { url = "https://files.pythonhosted.org/packages/ec/85/1bf997003815e60d57de7bd972c57dc6950446a3e4ccac43bc3070721856/charset_normalizer-3.4.4-cp310-cp310-manylinux2014_ppc64le.manylinux_2_17_ppc64le.manylinux_2_28_ppc64le.whl", hash = "sha256:f820802628d2694cb7e56db99213f930856014862f3fd943d290ea8438d07ca8", size = 162280, upload-time = "2025-10-14T04:40:16.14Z" },
    { url = "https://files.pythonhosted.org/packages/3e/8e/6aa1952f56b192f54921c436b87f2aaf7c7a7c3d0d1a765547d64fd83c13/charset_normalizer-3.4.4-cp310-cp310-manylinux2014_s390x.manylinux_2_17_s390x.manylinux_2_28_s390x.whl", hash = "sha256:798d75d81754988d2565bff1b97ba5a44411867c0cf32b77a7e8f8d84796b10d", size = 159454, upload-time = "2025-10-14T04:40:17.567Z" },
    { url = "https://files.pythonhosted.org/packages/36/3b/60cbd1f8e93aa25d1c669c649b7a655b0b5fb4c571858910ea9332678558/charset_normalizer-3.4.4-cp310-cp310-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:9d1bb833febdff5c8927f922386db610b49db6e0d4f4ee29601d71e7c2694313", size = 153609, upload-time = "2025-10-14T04:40:19.08Z" },
    { url = "https://files.pythonhosted.org/packages/64/91/6a13396948b8fd3c4b4fd5bc74d045f5637d78c9675585e8e9fbe5636554/charset_normalizer-3.4.4-cp310-cp310-manylinux_2_31_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:9cd98cdc06614a2f768d2b7286d66805f94c48cde050acdbbb7db2600ab3197e", size = 151849, upload-time = "2025-10-14T04:40:20.607Z" },
    { url = "https://files.pythonhosted.org/packages/b7/7a/59482e28b9981d105691e968c544cc0df3b7d6133152fb3dcdc8f135da7a/charset_normalizer-3.4.4-cp310-cp310-musllinux_1_2_aarch64.whl", hash = "sha256:077fbb858e903c73f6c9db43374fd213b0b6a778106bc7032446a8e8b5b38b93", size = 151586, upload-time = "2025-10-14T04:40:21.719Z" },
    { url = "https://files.pythonhosted.org/packages/92/59/f64ef6a1c4bdd2baf892b04cd78792ed8684fbc48d4c2afe467d96b4df57/charset_normalizer-3.4.4-cp310-cp310-musllinux_1_2_armv7l.whl", hash = "sha256:244bfb999c71b35de57821b8ea746b24e863398194a4014e4c76adc2bbdfeff0", size = 145290, upload-time = "2025-10-14T04:40:23.069Z" },
    { url = "https://files.pythonhosted.org/packages/6b/63/3bf9f279ddfa641ffa1962b0db6a57a9c294361cc2f5fcac997049a00e9c/charset_normalizer-3.4.4-cp310-cp310-musllinux_1_2_ppc64le.whl", hash = "sha256:64b55f9dce520635f018f907ff1b0df1fdc31f2795a922fb49dd14fbcdf48c84", size = 163663, upload-time = "2025-10-14T04:40:24.17Z" },
    { url = "https://files.pythonhosted.org/packages/ed/09/c9e38fc8fa9e0849b172b581fd9803bdf6e694041127933934184e19f8c3/charset_normalizer-3.4.4-cp310-cp310-musllinux_1_2_riscv64.whl", hash = "sha256:faa3a41b2b66b6e50f84ae4a68c64fcd0c44355741c6374813a800cd6695db9e", size = 151964, upload-time = "2025-10-14T04:40:25.368Z" },
    { url = "https://files.pythonhosted.org/packages/d2/d1/d28b747e512d0da79d8b6a1ac18b7ab2ecfd81b2944c4c710e166d8dd09c/charset_normalizer-3.4.4-cp310-cp310-musllinux_1_2_s390x.whl", hash = "sha256:6515f3182dbe4ea06ced2d9e8666d97b46ef4c75e326b79bb624110f122551db", size = 161064, upload-time = "2025-10-14T04:40:26.806Z" },
    { url = "https://files.pythonhosted.org/packages/bb/9a/31d62b611d901c3b9e5500c36aab0ff5eb442043fb3a1c254200d3d397d9/charset_normalizer-3.4.4-cp310-cp310-musllinux_1_2_x86_64.whl", hash = "sha256:cc00f04ed596e9dc0da42ed17ac5e596c6ccba999ba6bd92b0e0aef2f170f2d6", size = 155015, upload-time = "2025-10-14T04:40:28.284Z" },
    { url = "https://files.pythonhosted.org/packages/1f/f3/107e008fa2bff0c8b9319584174418e5e5285fef32f79d8ee6a430d0039c/charset_normalizer-3.4.4-cp310-cp310-win32.whl", hash = "sha256:f34be2938726fc13801220747472850852fe6b1ea75869a048d6f896838c896f", size = 99792, upload-time = "2025-10-14T04:40:29.613Z" },
    { url = "https://files.pythonhosted.org/packages/eb/66/e396e8a408843337d7315bab30dbf106c38966f1819f123257f5520f8a96/charset_normalizer-3.4.4-cp310-cp310-win_amd64.whl", hash = "sha256:a61900df84c667873b292c3de315a786dd8dac506704dea57bc957bd31e22c7d", size = 107198, upload-time = "2025-10-14T04:40:30.644Z" },
    { url = "https://files.pythonhosted.org/packages/b5/58/01b4f815bf0312704c267f2ccb6e5d42bcc7752340cd487bc9f8c3710597/charset_normalizer-3.4.4-cp310-cp310-win_arm64.whl", hash = "sha256:cead0978fc57397645f12578bfd2d5ea9138ea0fac82b2f63f7f7c6877986a69", size = 100262, upload-time = "2025-10-14T04:40:32.108Z" },
    { url = "https://files.pythonhosted.org/packages/ed/27/c6491ff4954e58a10f69ad90aca8a1b6fe9c5d3c6f380907af3c37435b59/charset_normalizer-3.4.4-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:6e1fcf0720908f200cd21aa4e6750a48ff6ce4afe7ff5a79a90d5ed8a08296f8", size = 206988, upload-time = "2025-10-14T04:40:33.79Z" },
    { url = "https://files.pythonhosted.org/packages/94/59/2e87300fe67ab820b5428580a53cad894272dbb97f38a7a814a2a1ac1011/charset_normalizer-3.4.4-cp311-cp311-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:5f819d5fe9234f9f82d75bdfa9aef3a3d72c4d24a6e57aeaebba32a704553aa0", size = 147324, upload-time = "2025-10-14T04:40:34.961Z" },
    { url = "https://files.pythonhosted.org/packages/07/fb/0cf61dc84b2b088391830f6274cb57c82e4da8bbc2efeac8c025edb88772/charset_normalizer-3.4.4-cp311-cp311-manylinux2014_armv7l.manylinux_2_17_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:a59cb51917aa591b1c4e6a43c132f0cdc3c76dbad6155df4e28ee626cc77a0a3", size = 142742, upload-time = "2025-10-14T04:40:36.105Z" },
    { url = "https://files.pythonhosted.org/packages/62/8b/171935adf2312cd745d290ed93cf16cf0dfe320863ab7cbeeae1dcd6535f/charset_normalizer-3.4.4-cp311-cp311-manylinux2014_ppc64le.manylinux_2_17_ppc64le.manylinux_2_28_ppc64le.whl", hash = "sha256:8ef3c867360f88ac904fd3f5e1f902f13307af9052646963ee08ff4f131adafc", size = 160863, upload-time = "2025-10-14T04:40:37.188Z" },
    { url = "https://files.pythonhosted.org/packages/09/73/ad875b192bda14f2173bfc1bc9a55e009808484a4b256748d931b6948442/charset_normalizer-3.4.4-cp311-cp311-manylinux2014_s390x.manylinux_2_17_s390x.manylinux_2_28_s390x.whl", hash = "sha256:d9e45d7faa48ee908174d8fe84854479ef838fc6a705c9315372eacbc2f02897", size = 157837, upload-time = "2025-10-14T04:40:38.435Z" },
    { url = "https://files.pythonhosted.org/packages/6d/fc/de9cce525b2c5b94b47c70a4b4fb19f871b24995c728e957ee68ab1671ea/charset_normalizer-3.4.4-cp311-cp311-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:840c25fb618a231545cbab0564a799f101b63b9901f2569faecd6b222ac72381", size = 151550, upload-time = "2025-10-14T04:40:40.053Z" },
    { url = "https://files.pythonhosted.org/packages/55/c2/43edd615fdfba8c6f2dfbd459b25a6b3b551f24ea21981e23fb768503ce1/charset_normalizer-3.4.4-cp311-cp311-manylinux_2_31_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:ca5862d5b3928c4940729dacc329aa9102900382fea192fc5e52eb69d6093815", size = 149162, upload-time = "2025-10-14T04:40:41.163Z" },
    { url = "https://files.pythonhosted.org/packages/03/86/bde4ad8b4d0e9429a4e82c1e8f5c659993a9a863ad62c7df05cf7b678d75/charset_normalizer-3.4.4-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:d9c7f57c3d666a53421049053eaacdd14bbd0a528e2186fcb2e672effd053bb0", size = 150019, upload-time = "2025-10-14T04:40:42.276Z" },
    { url = "https://files.pythonhosted.org/packages/1f/86/a151eb2af293a7e7bac3a739b81072585ce36ccfb4493039f49f1d3cae8c/charset_normalizer-3.4.4-cp311-cp311-musllinux_1_2_armv7l.whl", hash = "sha256:277e970e750505ed74c832b4bf75dac7476262ee2a013f5574dd49075879e161", size = 143310, upload-time = "2025-10-14T04:40:43.439Z" },
    { url = "https://files.pythonhosted.org/packages/b5/fe/43dae6144a7e07b87478fdfc4dbe9efd5defb0e7ec29f5f58a55aeef7bf7/charset_normalizer-3.4.4-cp311-cp311-musllinux_1_2_ppc64le.whl", hash = "sha256:31fd66405eaf47bb62e8cd575dc621c56c668f27d46a61d975a249930dd5e2a4", size = 162022, upload-time = "2025-10-14T04:40:44.547Z" },
    { url = "https://files.pythonhosted.org/packages/80/e6/7aab83774f5d2bca81f42ac58d04caf44f0cc2b65fc6db2b3b2e8a05f3b3/charset_normalizer-3.4.4-cp311-cp311-musllinux_1_2_riscv64.whl", hash = "sha256:0d3d8f15c07f86e9ff82319b3d9ef6f4bf907608f53fe9d92b28ea9ae3d1fd89", size = 149383, upload-time = "2025-10-14T04:40:46.018Z" },
    { url = "https://files.pythonhosted.org/packages/4f/e8/b289173b4edae05c0dde07f69f8db476a0b511eac556dfe0d6bda3c43384/charset_normalizer-3.4.4-cp311-cp311-musllinux_1_2_s390x.whl", hash = "sha256:9f7fcd74d410a36883701fafa2482a6af2ff5ba96b9a620e9e0721e28ead5569", size = 159098, upload-time = "2025-10-14T04:40:47.081Z" },
    { url = "https://files.pythonhosted.org/packages/d8/df/fe699727754cae3f8478493c7f45f777b17c3ef0600e28abfec8619eb49c/charset_normalizer-3.4.4-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:ebf3e58c7ec8a8bed6d66a75d7fb37b55e5015b03ceae72a8e7c74495551e224", size = 152991, upload-time = "2025-10-14T04:40:48.246Z" },
    { url = "https://files.pythonhosted.org/packages/1a/86/584869fe4ddb6ffa3bd9f491b87a01568797fb9bd8933f557dba9771beaf/charset_normalizer-3.4.4-cp311-cp311-win32.whl", hash = "sha256:eecbc200c7fd5ddb9a7f16c7decb07b566c29fa2161a16cf67b8d068bd21690a", size = 99456, upload-time = "2025-10-14T04:40:49.376Z" },
    { url = "https://files.pythonhosted.org/packages/65/f6/62fdd5feb60530f50f7e38b4f6a1d5203f4d16ff4f9f0952962c044e919a/charset_normalizer-3.4.4-cp311-cp311-win_amd64.whl", hash = "sha256:5ae497466c7901d54b639cf42d5b8c1b6a4fead55215500d2f486d34db48d016", size = 106978, upload-time = "2025-10-14T04:40:50.844Z" },
    { url = "https://files.pythonhosted.org/packages/7a/9d/0710916e6c82948b3be62d9d398cb4fcf4e97b56d6a6aeccd66c4b2f2bd5/charset_normalizer-3.4.4-cp311-cp311-win_arm64.whl", hash = "sha256:65e2befcd84bc6f37095f5961e68a6f077bf44946771354a28ad434c2cce0ae1", size = 99969, upload-time = "2025-10-14T04:40:52.272Z" },
    { url = "https://files.pythonhosted.org/packages/f3/85/1637cd4af66fa687396e757dec650f28025f2a2f5a5531a3208dc0ec43f2/charset_normalizer-3.4.4-cp312-cp312-macosx_10_13_universal2.whl", hash = "sha256:0a98e6759f854bd25a58a73fa88833fba3b7c491169f86ce1180c948ab3fd394", size = 208425, upload-time = "2025-10-14T04:40:53.353Z" },
    { url = "https://files.pythonhosted.org/packages/9d/6a/04130023fef2a0d9c62d0bae2649b69f7b7d8d24ea5536feef50551029df/charset_normalizer-3.4.4-cp312-cp312-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:b5b290ccc2a263e8d185130284f8501e3e36c5e02750fc6b6bdeb2e9e96f1e25", size = 148162, upload-time = "2025-10-14T04:40:54.558Z" },
    { url = "https://files.pythonhosted.org/packages/78/29/62328d79aa60da22c9e0b9a66539feae06ca0f5a4171ac4f7dc285b83688/charset_normalizer-3.4.4-cp312-cp312-manylinux2014_armv7l.manylinux_2_17_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:74bb723680f9f7a6234dcf67aea57e708ec1fbdf5699fb91dfd6f511b0a320ef", size = 144558, upload-time = "2025-10-14T04:40:55.677Z" },
    { url = "https://files.pythonhosted.org/packages/86/bb/b32194a4bf15b88403537c2e120b817c61cd4ecffa9b6876e941c3ee38fe/charset_normalizer-3.4.4-cp312-cp312-manylinux2014_ppc64le.manylinux_2_17_ppc64le.manylinux_2_28_ppc64le.whl", hash = "sha256:f1e34719c6ed0b92f418c7c780480b26b5d9c50349e9a9af7d76bf757530350d", size = 161497, upload-time = "2025-10-14T04:40:57.217Z" },
    { url = "https://files.pythonhosted.org/packages/19/89/a54c82b253d5b9b111dc74aca196ba5ccfcca8242d0fb64146d4d3183ff1/charset_normalizer-3.4.4-cp312-cp312-manylinux2014_s390x.manylinux_2_17_s390x.manylinux_2_28_s390x.whl", hash = "sha256:2437418e20515acec67d86e12bf70056a33abdacb5cb1655042f6538d6b085a8", size = 159240, upload-time = "2025-10-14T04:40:58.358Z" },
    { url = "https://files.pythonhosted.org/packages/c0/10/d20b513afe03acc89ec33948320a5544d31f21b05368436d580dec4e234d/charset_normalizer-3.4.4-cp312-cp312-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:11d694519d7f29d6cd09f6ac70028dba10f92f6cdd059096db198c283794ac86", size = 153471, upload-time = "2025-10-14T04:40:59.468Z" },
    { url = "https://files.pythonhosted.org/packages/61/fa/fbf177b55bdd727010f9c0a3c49eefa1d10f960e5f09d1d887bf93c2e698/charset_normalizer-3.4.4-cp312-cp312-manylinux_2_31_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:ac1c4a689edcc530fc9d9aa11f5774b9e2f33f9a0c6a57864e90908f5208d30a", size = 150864, upload-time = "2025-10-14T04:41:00.623Z" },
    { url = "https://files.pythonhosted.org/packages/05/12/9fbc6a4d39c0198adeebbde20b619790e9236557ca59fc40e0e3cebe6f40/charset_normalizer-3.4.4-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:21d142cc6c0ec30d2efee5068ca36c128a30b0f2c53c1c07bd78cb6bc1d3be5f", size = 150647, upload-time = "2025-10-14T04:41:01.754Z" },
    { url = "https://files.pythonhosted.org/packages/ad/1f/6a9a593d52e3e8c5d2b167daf8c6b968808efb57ef4c210acb907c365bc4/charset_normalizer-3.4.4-cp312-cp312-musllinux_1_2_armv7l.whl", hash = "sha256:5dbe56a36425d26d6cfb40ce79c314a2e4dd6211d51d6d2191c00bed34f354cc", size = 145110, upload-time = "2025-10-14T04:41:03.231Z" },
    { url = "https://files.pythonhosted.org/packages/30/42/9a52c609e72471b0fc54386dc63c3781a387bb4fe61c20231a4ebcd58bdd/charset_normalizer-3.4.4-cp312-cp312-musllinux_1_2_ppc64le.whl", hash = "sha256:5bfbb1b9acf3334612667b61bd3002196fe2a1eb4dd74d247e0f2a4d50ec9bbf", size = 162839, upload-time = "2025-10-14T04:41:04.715Z" },
    { url = "https://files.pythonhosted.org/packages/c4/5b/c0682bbf9f11597073052628ddd38344a3d673fda35a36773f7d19344b23/charset_normalizer-3.4.4-cp312-cp312-musllinux_1_2_riscv64.whl", hash = "sha256:d055ec1e26e441f6187acf818b73564e6e6282709e9bcb5b63f5b23068356a15", size = 150667, upload-time = "2025-10-14T04:41:05.827Z" },
    { url = "https://files.pythonhosted.org/packages/e4/24/a41afeab6f990cf2daf6cb8c67419b63b48cf518e4f56022230840c9bfb2/charset_normalizer-3.4.4-cp312-cp312-musllinux_1_2_s390x.whl", hash = "sha256:af2d8c67d8e573d6de5bc30cdb27e9b95e49115cd9baad5ddbd1a6207aaa82a9", size = 160535, upload-time = "2025-10-14T04:41:06.938Z" },
    { url = "https://files.pythonhosted.org/packages/2a/e5/6a4ce77ed243c4a50a1fecca6aaaab419628c818a49434be428fe24c9957/charset_normalizer-3.4.4-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:780236ac706e66881f3b7f2f32dfe90507a09e67d1d454c762cf642e6e1586e0", size = 154816, upload-time = "2025-10-14T04:41:08.101Z" },
    { url = "https://files.pythonhosted.org/packages/a8/ef/89297262b8092b312d29cdb2517cb1237e51db8ecef2e9af5edbe7b683b1/charset_normalizer-3.4.4-cp312-cp312-win32.whl", hash = "sha256:5833d2c39d8896e4e19b689ffc198f08ea58116bee26dea51e362ecc7cd3ed26", size = 99694, upload-time = "2025-10-14T04:41:09.23Z" },
    { url = "https://files.pythonhosted.org/packages/3d/2d/1e5ed9dd3b3803994c155cd9aacb60c82c331bad84daf75bcb9c91b3295e/charset_normalizer-3.4.4-cp312-cp312-win_amd64.whl", hash = "sha256:a79cfe37875f822425b89a82333404539ae63dbdddf97f84dcbc3d339aae9525", size = 107131, upload-time = "2025-10-14T04:41:10.467Z" },
    { url = "https://files.pythonhosted.org/packages/d0/d9/0ed4c7098a861482a7b6a95603edce4c0d9db2311af23da1fb2b75ec26fc/charset_normalizer-3.4.4-cp312-cp312-win_arm64.whl", hash = "sha256:376bec83a63b8021bb5c8ea75e21c4ccb86e7e45ca4eb81146091b56599b80c3", size = 100390, upload-time = "2025-10-14T04:41:11.915Z" },
    { url = "https://files.pythonhosted.org/packages/97/45/4b3a1239bbacd321068ea6e7ac28875b03ab8bc0aa0966452db17cd36714/charset_normalizer-3.4.4-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:e1f185f86a6f3403aa2420e815904c67b2f9ebc443f045edd0de921108345794", size = 208091, upload-time = "2025-10-14T04:41:13.346Z" },
    { url = "https://files.pythonhosted.org/packages/7d/62/73a6d7450829655a35bb88a88fca7d736f9882a27eacdca2c6d505b57e2e/charset_normalizer-3.4.4-cp313-cp313-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:6b39f987ae8ccdf0d2642338faf2abb1862340facc796048b604ef14919e55ed", size = 147936, upload-time = "2025-10-14T04:41:14.461Z" },
    { url = "https://files.pythonhosted.org/packages/89/c5/adb8c8b3d6625bef6d88b251bbb0d95f8205831b987631ab0c8bb5d937c2/charset_normalizer-3.4.4-cp313-cp313-manylinux2014_armv7l.manylinux_2_17_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:3162d5d8ce1bb98dd51af660f2121c55d0fa541b46dff7bb9b9f86ea1d87de72", size = 144180, upload-time = "2025-10-14T04:41:15.588Z" },
    { url = "https://files.pythonhosted.org/packages/91/ed/9706e4070682d1cc219050b6048bfd293ccf67b3d4f5a4f39207453d4b99/charset_normalizer-3.4.4-cp313-cp313-manylinux2014_ppc64le.manylinux_2_17_ppc64le.manylinux_2_28_ppc64le.whl", hash = "sha256:81d5eb2a312700f4ecaa977a8235b634ce853200e828fbadf3a9c50bab278328", size = 161346, upload-time = "2025-10-14T04:41:16.738Z" },
    { url = "https://files.pythonhosted.org/packages/d5/0d/031f0d95e4972901a2f6f09ef055751805ff541511dc1252ba3ca1f80cf5/charset_normalizer-3.4.4-cp313-cp313-manylinux2014_s390x.manylinux_2_17_s390x.manylinux_2_28_s390x.whl", hash = "sha256:5bd2293095d766545ec1a8f612559f6b40abc0eb18bb2f5d1171872d34036ede", size = 158874, upload-time = "2025-10-14T04:41:17.923Z" },
    { url = "https://files.pythonhosted.org/packages/f5/83/6ab5883f57c9c801ce5e5677242328aa45592be8a00644310a008d04f922/charset_normalizer-3.4.4-cp313-cp313-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:a8a8b89589086a25749f471e6a900d3f662d1d3b6e2e59dcecf787b1cc3a1894", size = 153076, upload-time = "2025-10-14T04:41:19.106Z" },
    { url = "https://files.pythonhosted.org/packages/75/1e/5ff781ddf5260e387d6419959ee89ef13878229732732ee73cdae01800f2/charset_normalizer-3.4.4-cp313-cp313-manylinux_2_31_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:bc7637e2f80d8530ee4a78e878bce464f70087ce73cf7c1caf142416923b98f1", size = 150601, upload-time = "2025-10-14T04:41:20.245Z" },
    { url = "https://files.pythonhosted.org/packages/d7/57/71be810965493d3510a6ca79b90c19e48696fb1ff964da319334b12677f0/charset_normalizer-3.4.4-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:f8bf04158c6b607d747e93949aa60618b61312fe647a6369f88ce2ff16043490", size = 150376, upload-time = "2025-10-14T04:41:21.398Z" },
    { url = "https://files.pythonhosted.org/packages/e5/d5/c3d057a78c181d007014feb7e9f2e65905a6c4ef182c0ddf0de2924edd65/charset_normalizer-3.4.4-cp313-cp313-musllinux_1_2_armv7l.whl", hash = "sha256:554af85e960429cf30784dd47447d5125aaa3b99a6f0683589dbd27e2f45da44", size = 144825, upload-time = "2025-10-14T04:41:22.583Z" },
    { url = "https://files.pythonhosted.org/packages/e6/8c/d0406294828d4976f275ffbe66f00266c4b3136b7506941d87c00cab5272/charset_normalizer-3.4.4-cp313-cp313-musllinux_1_2_ppc64le.whl", hash = "sha256:74018750915ee7ad843a774364e13a3db91682f26142baddf775342c3f5b1133", size = 162583, upload-time = "2025-10-14T04:41:23.754Z" },
    { url = "https://files.pythonhosted.org/packages/d7/24/e2aa1f18c8f15c4c0e932d9287b8609dd30ad56dbe41d926bd846e22fb8d/charset_normalizer-3.4.4-cp313-cp313-musllinux_1_2_riscv64.whl", hash = "sha256:c0463276121fdee9c49b98908b3a89c39be45d86d1dbaa22957e38f6321d4ce3", size = 150366, upload-time = "2025-10-14T04:41:25.27Z" },
    { url = "https://files.pythonhosted.org/packages/e4/5b/1e6160c7739aad1e2df054300cc618b06bf784a7a164b0f238360721ab86/charset_normalizer-3.4.4-cp313-cp313-musllinux_1_2_s390x.whl", hash = "sha256:362d61fd13843997c1c446760ef36f240cf81d3ebf74ac62652aebaf7838561e", size = 160300, upload-time = "2025-10-14T04:41:26.725Z" },
    { url = "https://files.pythonhosted.org/packages/7a/10/f882167cd207fbdd743e55534d5d9620e095089d176d55cb22d5322f2afd/charset_normalizer-3.4.4-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:9a26f18905b8dd5d685d6d07b0cdf98a79f3c7a918906af7cc143ea2e164c8bc", size = 154465, upload-time = "2025-10-14T04:41:28.322Z" },
    { url = "https://files.pythonhosted.org/packages/89/66/c7a9e1b7429be72123441bfdbaf2bc13faab3f90b933f664db506dea5915/charset_normalizer-3.4.4-cp313-cp313-win32.whl", hash = "sha256:9b35f4c90079ff2e2edc5b26c0c77925e5d2d255c42c74fdb70fb49b172726ac", size = 99404, upload-time = "2025-10-14T04:41:29.95Z" },
    { url = "https://files.pythonhosted.org/packages/c4/26/b9924fa27db384bdcd97ab83b4f0a8058d96ad9626ead570674d5e737d90/charset_normalizer-3.4.4-cp313-cp313-win_amd64.whl", hash = "sha256:b435cba5f4f750aa6c0a0d92c541fb79f69a387c91e61f1795227e4ed9cece14", size = 107092, upload-time = "2025-10-14T04:41:31.188Z" },
    { url = "https://files.pythonhosted.org/packages/af/8f/3ed4bfa0c0c72a7ca17f0380cd9e4dd842b09f664e780c13cff1dcf2ef1b/charset_normalizer-3.4.4-cp313-cp313-win_arm64.whl", hash = "sha256:542d2cee80be6f80247095cc36c418f7bddd14f4a6de45af91dfad36d817bba2", size = 100408, upload-time = "2025-10-14T04:41:32.624Z" },
    { url = "https://files.pythonhosted.org/packages/2a/35/7051599bd493e62411d6ede36fd5af83a38f37c4767b92884df7301db25d/charset_normalizer-3.4.4-cp314-cp314-macosx_10_13_universal2.whl", hash = "sha256:da3326d9e65ef63a817ecbcc0df6e94463713b754fe293eaa03da99befb9a5bd", size = 207746, upload-time = "2025-10-14T04:41:33.773Z" },
    { url = "https://files.pythonhosted.org/packages/10/9a/97c8d48ef10d6cd4fcead2415523221624bf58bcf68a802721a6bc807c8f/charset_normalizer-3.4.4-cp314-cp314-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:8af65f14dc14a79b924524b1e7fffe304517b2bff5a58bf64f30b98bbc5079eb", size = 147889, upload-time = "2025-10-14T04:41:34.897Z" },
    { url = "https://files.pythonhosted.org/packages/10/bf/979224a919a1b606c82bd2c5fa49b5c6d5727aa47b4312bb27b1734f53cd/charset_normalizer-3.4.4-cp314-cp314-manylinux2014_armv7l.manylinux_2_17_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:74664978bb272435107de04e36db5a9735e78232b85b77d45cfb38f758efd33e", size = 143641, upload-time = "2025-10-14T04:41:36.116Z" },
    { url = "https://files.pythonhosted.org/packages/ba/33/0ad65587441fc730dc7bd90e9716b30b4702dc7b617e6ba4997dc8651495/charset_normalizer-3.4.4-cp314-cp314-manylinux2014_ppc64le.manylinux_2_17_ppc64le.manylinux_2_28_ppc64le.whl", hash = "sha256:752944c7ffbfdd10c074dc58ec2d5a8a4cd9493b314d367c14d24c17684ddd14", size = 160779, upload-time = "2025-10-14T04:41:37.229Z" },
    { url = "https://files.pythonhosted.org/packages/67/ed/331d6b249259ee71ddea93f6f2f0a56cfebd46938bde6fcc6f7b9a3d0e09/charset_normalizer-3.4.4-cp314-cp314-manylinux2014_s390x.manylinux_2_17_s390x.manylinux_2_28_s390x.whl", hash = "sha256:d1f13550535ad8cff21b8d757a3257963e951d96e20ec82ab44bc64aeb62a191", size = 159035, upload-time = "2025-10-14T04:41:38.368Z" },
    { url = "https://files.pythonhosted.org/packages/67/ff/f6b948ca32e4f2a4576aa129d8bed61f2e0543bf9f5f2b7fc3758ed005c9/charset_normalizer-3.4.4-cp314-cp314-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:ecaae4149d99b1c9e7b88bb03e3221956f68fd6d50be2ef061b2381b61d20838", size = 152542, upload-time = "2025-10-14T04:41:39.862Z" },
    { url = "https://files.pythonhosted.org/packages/16/85/276033dcbcc369eb176594de22728541a925b2632f9716428c851b149e83/charset_normalizer-3.4.4-cp314-cp314-manylinux_2_31_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:cb6254dc36b47a990e59e1068afacdcd02958bdcce30bb50cc1700a8b9d624a6", size = 149524, upload-time = "2025-10-14T04:41:41.319Z" },
    { url = "https://files.pythonhosted.org/packages/9e/f2/6a2a1f722b6aba37050e626530a46a68f74e63683947a8acff92569f979a/charset_normalizer-3.4.4-cp314-cp314-musllinux_1_2_aarch64.whl", hash = "sha256:c8ae8a0f02f57a6e61203a31428fa1d677cbe50c93622b4149d5c0f319c1d19e", size = 150395, upload-time = "2025-10-14T04:41:42.539Z" },
    { url = "https://files.pythonhosted.org/packages/60/bb/2186cb2f2bbaea6338cad15ce23a67f9b0672929744381e28b0592676824/charset_normalizer-3.4.4-cp314-cp314-musllinux_1_2_armv7l.whl", hash = "sha256:47cc91b2f4dd2833fddaedd2893006b0106129d4b94fdb6af1f4ce5a9965577c", size = 143680, upload-time = "2025-10-14T04:41:43.661Z" },
    { url = "https://files.pythonhosted.org/packages/7d/a5/bf6f13b772fbb2a90360eb620d52ed8f796f3c5caee8398c3b2eb7b1c60d/charset_normalizer-3.4.4-cp314-cp314-musllinux_1_2_ppc64le.whl", hash = "sha256:82004af6c302b5d3ab2cfc4cc5f29db16123b1a8417f2e25f9066f91d4411090", size = 162045, upload-time = "2025-10-14T04:41:44.821Z" },
    { url = "https://files.pythonhosted.org/packages/df/c5/d1be898bf0dc3ef9030c3825e5d3b83f2c528d207d246cbabe245966808d/charset_normalizer-3.4.4-cp314-cp314-musllinux_1_2_riscv64.whl", hash = "sha256:2b7d8f6c26245217bd2ad053761201e9f9680f8ce52f0fcd8d0755aeae5b2152", size = 149687, upload-time = "2025-10-14T04:41:46.442Z" },
    { url = "https://files.pythonhosted.org/packages/a5/42/90c1f7b9341eef50c8a1cb3f098ac43b0508413f33affd762855f67a410e/charset_normalizer-3.4.4-cp314-cp314-musllinux_1_2_s390x.whl", hash = "sha256:799a7a5e4fb2d5898c60b640fd4981d6a25f1c11790935a44ce38c54e985f828", size = 160014, upload-time = "2025-10-14T04:41:47.631Z" },
    { url = "https://files.pythonhosted.org/packages/76/be/4d3ee471e8145d12795ab655ece37baed0929462a86e72372fd25859047c/charset_normalizer-3.4.4-cp314-cp314-musllinux_1_2_x86_64.whl", hash = "sha256:99ae2cffebb06e6c22bdc25801d7b30f503cc87dbd283479e7b606f70aff57ec", size = 154044, upload-time = "2025-10-14T04:41:48.81Z" },
    { url = "https://files.pythonhosted.org/packages/b0/6f/8f7af07237c34a1defe7defc565a9bc1807762f672c0fde711a4b22bf9c0/charset_normalizer-3.4.4-cp314-cp314-win32.whl", hash = "sha256:f9d332f8c2a2fcbffe1378594431458ddbef721c1769d78e2cbc06280d8155f9", size = 99940, upload-time = "2025-10-14T04:41:49.946Z" },
    { url = "https://files.pythonhosted.org/packages/4b/51/8ade005e5ca5b0d80fb4aff72a3775b325bdc3d27408c8113811a7cbe640/charset_normalizer-3.4.4-cp314-cp314-win_amd64.whl", hash = "sha256:8a6562c3700cce886c5be75ade4a5db4214fda19fede41d9792d100288d8f94c", size = 107104, upload-time = "2025-10-14T04:41:51.051Z" },
    { url = "https://files.pythonhosted.org/packages/da/5f/6b8f83a55bb8278772c5ae54a577f3099025f9ade59d0136ac24a0df4bde/charset_normalizer-3.4.4-cp314-cp314-win_arm64.whl", hash = "sha256:de00632ca48df9daf77a2c65a484531649261ec9f25489917f09e455cb09ddb2", size = 100743, upload-time = "2025-10-14T04:41:52.122Z" },
    { url = "https://files.pythonhosted.org/packages/0a/4c/925909008ed5a988ccbb72dcc897407e5d6d3bd72410d69e051fc0c14647/charset_normalizer-3.4.4-py3-none-any.whl", hash = "sha256:7a32c560861a02ff789ad905a2fe94e3f840803362c84fecf1851cb4cf3dc37f", size = 53402, upload-time = "2025-10-14T04:42:31.76Z" },
]

[[package]]
name = "click"
version = "8.3.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "colorama", marker = "sys_platform == 'win32'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/46/61/de6cd827efad202d7057d93e0fed9294b96952e188f7384832791c7b2254/click-8.3.0.tar.gz", hash = "sha256:e7b8232224eba16f4ebe410c25ced9f7875cb5f3263ffc93cc3e8da705e229c4", size = 276943, upload-time = "2025-09-18T17:32:23.696Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/db/d3/9dcc0f5797f070ec8edf30fbadfb200e71d9db6b84d211e3b2085a7589a0/click-8.3.0-py3-none-any.whl", hash = "sha256:9b9f285302c6e3064f4330c05f05b81945b2a39544279343e6e7c5f27a9baddc", size = 107295, upload-time = "2025-09-18T17:32:22.42Z" },
]

[[package]]
name = "cloudpickle"
version = "3.1.2"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/27/fb/576f067976d320f5f0114a8d9fa1215425441bb35627b1993e5afd8111e5/cloudpickle-3.1.2.tar.gz", hash = "sha256:7fda9eb655c9c230dab534f1983763de5835249750e85fbcef43aaa30a9a2414", size = 22330, upload-time = "2025-11-03T09:25:26.604Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/88/39/799be3f2f0f38cc727ee3b4f1445fe6d5e4133064ec2e4115069418a5bb6/cloudpickle-3.1.2-py3-none-any.whl", hash = "sha256:9acb47f6afd73f60dc1df93bb801b472f05ff42fa6c84167d25cb206be1fbf4a", size = 22228, upload-time = "2025-11-03T09:25:25.534Z" },
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
name = "cryptography"
version = "44.0.3"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "cffi", marker = "python_full_version >= '3.11' and platform_python_implementation != 'PyPy'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/53/d6/1411ab4d6108ab167d06254c5be517681f1e331f90edf1379895bcb87020/cryptography-44.0.3.tar.gz", hash = "sha256:fe19d8bc5536a91a24a8133328880a41831b6c5df54599a8417b62fe015d3053", size = 711096, upload-time = "2025-05-02T19:36:04.667Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/08/53/c776d80e9d26441bb3868457909b4e74dd9ccabd182e10b2b0ae7a07e265/cryptography-44.0.3-cp37-abi3-macosx_10_9_universal2.whl", hash = "sha256:962bc30480a08d133e631e8dfd4783ab71cc9e33d5d7c1e192f0b7c06397bb88", size = 6670281, upload-time = "2025-05-02T19:34:50.665Z" },
    { url = "https://files.pythonhosted.org/packages/6a/06/af2cf8d56ef87c77319e9086601bef621bedf40f6f59069e1b6d1ec498c5/cryptography-44.0.3-cp37-abi3-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:4ffc61e8f3bf5b60346d89cd3d37231019c17a081208dfbbd6e1605ba03fa137", size = 3959305, upload-time = "2025-05-02T19:34:53.042Z" },
    { url = "https://files.pythonhosted.org/packages/ae/01/80de3bec64627207d030f47bf3536889efee8913cd363e78ca9a09b13c8e/cryptography-44.0.3-cp37-abi3-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:58968d331425a6f9eedcee087f77fd3c927c88f55368f43ff7e0a19891f2642c", size = 4171040, upload-time = "2025-05-02T19:34:54.675Z" },
    { url = "https://files.pythonhosted.org/packages/bd/48/bb16b7541d207a19d9ae8b541c70037a05e473ddc72ccb1386524d4f023c/cryptography-44.0.3-cp37-abi3-manylinux_2_28_aarch64.whl", hash = "sha256:e28d62e59a4dbd1d22e747f57d4f00c459af22181f0b2f787ea83f5a876d7c76", size = 3963411, upload-time = "2025-05-02T19:34:56.61Z" },
    { url = "https://files.pythonhosted.org/packages/42/b2/7d31f2af5591d217d71d37d044ef5412945a8a8e98d5a2a8ae4fd9cd4489/cryptography-44.0.3-cp37-abi3-manylinux_2_28_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:af653022a0c25ef2e3ffb2c673a50e5a0d02fecc41608f4954176f1933b12359", size = 3689263, upload-time = "2025-05-02T19:34:58.591Z" },
    { url = "https://files.pythonhosted.org/packages/25/50/c0dfb9d87ae88ccc01aad8eb93e23cfbcea6a6a106a9b63a7b14c1f93c75/cryptography-44.0.3-cp37-abi3-manylinux_2_28_x86_64.whl", hash = "sha256:157f1f3b8d941c2bd8f3ffee0af9b049c9665c39d3da9db2dc338feca5e98a43", size = 4196198, upload-time = "2025-05-02T19:35:00.988Z" },
    { url = "https://files.pythonhosted.org/packages/66/c9/55c6b8794a74da652690c898cb43906310a3e4e4f6ee0b5f8b3b3e70c441/cryptography-44.0.3-cp37-abi3-manylinux_2_34_aarch64.whl", hash = "sha256:c6cd67722619e4d55fdb42ead64ed8843d64638e9c07f4011163e46bc512cf01", size = 3966502, upload-time = "2025-05-02T19:35:03.091Z" },
    { url = "https://files.pythonhosted.org/packages/b6/f7/7cb5488c682ca59a02a32ec5f975074084db4c983f849d47b7b67cc8697a/cryptography-44.0.3-cp37-abi3-manylinux_2_34_x86_64.whl", hash = "sha256:b424563394c369a804ecbee9b06dfb34997f19d00b3518e39f83a5642618397d", size = 4196173, upload-time = "2025-05-02T19:35:05.018Z" },
    { url = "https://files.pythonhosted.org/packages/d2/0b/2f789a8403ae089b0b121f8f54f4a3e5228df756e2146efdf4a09a3d5083/cryptography-44.0.3-cp37-abi3-musllinux_1_2_aarch64.whl", hash = "sha256:c91fc8e8fd78af553f98bc7f2a1d8db977334e4eea302a4bfd75b9461c2d8904", size = 4087713, upload-time = "2025-05-02T19:35:07.187Z" },
    { url = "https://files.pythonhosted.org/packages/1d/aa/330c13655f1af398fc154089295cf259252f0ba5df93b4bc9d9c7d7f843e/cryptography-44.0.3-cp37-abi3-musllinux_1_2_x86_64.whl", hash = "sha256:25cd194c39fa5a0aa4169125ee27d1172097857b27109a45fadc59653ec06f44", size = 4299064, upload-time = "2025-05-02T19:35:08.879Z" },
    { url = "https://files.pythonhosted.org/packages/10/a8/8c540a421b44fd267a7d58a1fd5f072a552d72204a3f08194f98889de76d/cryptography-44.0.3-cp37-abi3-win32.whl", hash = "sha256:3be3f649d91cb182c3a6bd336de8b61a0a71965bd13d1a04a0e15b39c3d5809d", size = 2773887, upload-time = "2025-05-02T19:35:10.41Z" },
    { url = "https://files.pythonhosted.org/packages/b9/0d/c4b1657c39ead18d76bbd122da86bd95bdc4095413460d09544000a17d56/cryptography-44.0.3-cp37-abi3-win_amd64.whl", hash = "sha256:3883076d5c4cc56dbef0b898a74eb6992fdac29a7b9013870b34efe4ddb39a0d", size = 3209737, upload-time = "2025-05-02T19:35:12.12Z" },
    { url = "https://files.pythonhosted.org/packages/34/a3/ad08e0bcc34ad436013458d7528e83ac29910943cea42ad7dd4141a27bbb/cryptography-44.0.3-cp39-abi3-macosx_10_9_universal2.whl", hash = "sha256:5639c2b16764c6f76eedf722dbad9a0914960d3489c0cc38694ddf9464f1bb2f", size = 6673501, upload-time = "2025-05-02T19:35:13.775Z" },
    { url = "https://files.pythonhosted.org/packages/b1/f0/7491d44bba8d28b464a5bc8cc709f25a51e3eac54c0a4444cf2473a57c37/cryptography-44.0.3-cp39-abi3-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:f3ffef566ac88f75967d7abd852ed5f182da252d23fac11b4766da3957766759", size = 3960307, upload-time = "2025-05-02T19:35:15.917Z" },
    { url = "https://files.pythonhosted.org/packages/f7/c8/e5c5d0e1364d3346a5747cdcd7ecbb23ca87e6dea4f942a44e88be349f06/cryptography-44.0.3-cp39-abi3-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:192ed30fac1728f7587c6f4613c29c584abdc565d7417c13904708db10206645", size = 4170876, upload-time = "2025-05-02T19:35:18.138Z" },
    { url = "https://files.pythonhosted.org/packages/73/96/025cb26fc351d8c7d3a1c44e20cf9a01e9f7cf740353c9c7a17072e4b264/cryptography-44.0.3-cp39-abi3-manylinux_2_28_aarch64.whl", hash = "sha256:7d5fe7195c27c32a64955740b949070f21cba664604291c298518d2e255931d2", size = 3964127, upload-time = "2025-05-02T19:35:19.864Z" },
    { url = "https://files.pythonhosted.org/packages/01/44/eb6522db7d9f84e8833ba3bf63313f8e257729cf3a8917379473fcfd6601/cryptography-44.0.3-cp39-abi3-manylinux_2_28_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:3f07943aa4d7dad689e3bb1638ddc4944cc5e0921e3c227486daae0e31a05e54", size = 3689164, upload-time = "2025-05-02T19:35:21.449Z" },
    { url = "https://files.pythonhosted.org/packages/68/fb/d61a4defd0d6cee20b1b8a1ea8f5e25007e26aeb413ca53835f0cae2bcd1/cryptography-44.0.3-cp39-abi3-manylinux_2_28_x86_64.whl", hash = "sha256:cb90f60e03d563ca2445099edf605c16ed1d5b15182d21831f58460c48bffb93", size = 4198081, upload-time = "2025-05-02T19:35:23.187Z" },
    { url = "https://files.pythonhosted.org/packages/1b/50/457f6911d36432a8811c3ab8bd5a6090e8d18ce655c22820994913dd06ea/cryptography-44.0.3-cp39-abi3-manylinux_2_34_aarch64.whl", hash = "sha256:ab0b005721cc0039e885ac3503825661bd9810b15d4f374e473f8c89b7d5460c", size = 3967716, upload-time = "2025-05-02T19:35:25.426Z" },
    { url = "https://files.pythonhosted.org/packages/35/6e/dca39d553075980ccb631955c47b93d87d27f3596da8d48b1ae81463d915/cryptography-44.0.3-cp39-abi3-manylinux_2_34_x86_64.whl", hash = "sha256:3bb0847e6363c037df8f6ede57d88eaf3410ca2267fb12275370a76f85786a6f", size = 4197398, upload-time = "2025-05-02T19:35:27.678Z" },
    { url = "https://files.pythonhosted.org/packages/9b/9d/d1f2fe681eabc682067c66a74addd46c887ebacf39038ba01f8860338d3d/cryptography-44.0.3-cp39-abi3-musllinux_1_2_aarch64.whl", hash = "sha256:b0cc66c74c797e1db750aaa842ad5b8b78e14805a9b5d1348dc603612d3e3ff5", size = 4087900, upload-time = "2025-05-02T19:35:29.312Z" },
    { url = "https://files.pythonhosted.org/packages/c4/f5/3599e48c5464580b73b236aafb20973b953cd2e7b44c7c2533de1d888446/cryptography-44.0.3-cp39-abi3-musllinux_1_2_x86_64.whl", hash = "sha256:6866df152b581f9429020320e5eb9794c8780e90f7ccb021940d7f50ee00ae0b", size = 4301067, upload-time = "2025-05-02T19:35:31.547Z" },
    { url = "https://files.pythonhosted.org/packages/a7/6c/d2c48c8137eb39d0c193274db5c04a75dab20d2f7c3f81a7dcc3a8897701/cryptography-44.0.3-cp39-abi3-win32.whl", hash = "sha256:c138abae3a12a94c75c10499f1cbae81294a6f983b3af066390adee73f433028", size = 2775467, upload-time = "2025-05-02T19:35:33.805Z" },
    { url = "https://files.pythonhosted.org/packages/c9/ad/51f212198681ea7b0deaaf8846ee10af99fba4e894f67b353524eab2bbe5/cryptography-44.0.3-cp39-abi3-win_amd64.whl", hash = "sha256:5d186f32e52e66994dce4f766884bcb9c68b8da62d61d9d215bfe5fb56d21334", size = 3210375, upload-time = "2025-05-02T19:35:35.369Z" },
    { url = "https://files.pythonhosted.org/packages/7f/10/abcf7418536df1eaba70e2cfc5c8a0ab07aa7aa02a5cbc6a78b9d8b4f121/cryptography-44.0.3-pp310-pypy310_pp73-macosx_10_9_x86_64.whl", hash = "sha256:cad399780053fb383dc067475135e41c9fe7d901a97dd5d9c5dfb5611afc0d7d", size = 3393192, upload-time = "2025-05-02T19:35:37.468Z" },
    { url = "https://files.pythonhosted.org/packages/06/59/ecb3ef380f5891978f92a7f9120e2852b1df6f0a849c277b8ea45b865db2/cryptography-44.0.3-pp310-pypy310_pp73-manylinux_2_28_aarch64.whl", hash = "sha256:21a83f6f35b9cc656d71b5de8d519f566df01e660ac2578805ab245ffd8523f8", size = 3898419, upload-time = "2025-05-02T19:35:39.065Z" },
    { url = "https://files.pythonhosted.org/packages/bb/d0/35e2313dbb38cf793aa242182ad5bc5ef5c8fd4e5dbdc380b936c7d51169/cryptography-44.0.3-pp310-pypy310_pp73-manylinux_2_28_x86_64.whl", hash = "sha256:fc3c9babc1e1faefd62704bb46a69f359a9819eb0292e40df3fb6e3574715cd4", size = 4117892, upload-time = "2025-05-02T19:35:40.839Z" },
    { url = "https://files.pythonhosted.org/packages/dc/c8/31fb6e33b56c2c2100d76de3fd820afaa9d4d0b6aea1ccaf9aaf35dc7ce3/cryptography-44.0.3-pp310-pypy310_pp73-manylinux_2_34_aarch64.whl", hash = "sha256:e909df4053064a97f1e6565153ff8bb389af12c5c8d29c343308760890560aff", size = 3900855, upload-time = "2025-05-02T19:35:42.599Z" },
    { url = "https://files.pythonhosted.org/packages/43/2a/08cc2ec19e77f2a3cfa2337b429676406d4bb78ddd130a05c458e7b91d73/cryptography-44.0.3-pp310-pypy310_pp73-manylinux_2_34_x86_64.whl", hash = "sha256:dad80b45c22e05b259e33ddd458e9e2ba099c86ccf4e88db7bbab4b747b18d06", size = 4117619, upload-time = "2025-05-02T19:35:44.774Z" },
    { url = "https://files.pythonhosted.org/packages/02/68/fc3d3f84022a75f2ac4b1a1c0e5d6a0c2ea259e14cd4aae3e0e68e56483c/cryptography-44.0.3-pp310-pypy310_pp73-win_amd64.whl", hash = "sha256:479d92908277bed6e1a1c69b277734a7771c2b78633c224445b5c60a9f4bc1d9", size = 3136570, upload-time = "2025-05-02T19:35:46.94Z" },
    { url = "https://files.pythonhosted.org/packages/8d/4b/c11ad0b6c061902de5223892d680e89c06c7c4d606305eb8de56c5427ae6/cryptography-44.0.3-pp311-pypy311_pp73-macosx_10_9_x86_64.whl", hash = "sha256:896530bc9107b226f265effa7ef3f21270f18a2026bc09fed1ebd7b66ddf6375", size = 3390230, upload-time = "2025-05-02T19:35:49.062Z" },
    { url = "https://files.pythonhosted.org/packages/58/11/0a6bf45d53b9b2290ea3cec30e78b78e6ca29dc101e2e296872a0ffe1335/cryptography-44.0.3-pp311-pypy311_pp73-manylinux_2_28_aarch64.whl", hash = "sha256:9b4d4a5dbee05a2c390bf212e78b99434efec37b17a4bff42f50285c5c8c9647", size = 3895216, upload-time = "2025-05-02T19:35:51.351Z" },
    { url = "https://files.pythonhosted.org/packages/0a/27/b28cdeb7270e957f0077a2c2bfad1b38f72f1f6d699679f97b816ca33642/cryptography-44.0.3-pp311-pypy311_pp73-manylinux_2_28_x86_64.whl", hash = "sha256:02f55fb4f8b79c1221b0961488eaae21015b69b210e18c386b69de182ebb1259", size = 4115044, upload-time = "2025-05-02T19:35:53.044Z" },
    { url = "https://files.pythonhosted.org/packages/35/b0/ec4082d3793f03cb248881fecefc26015813199b88f33e3e990a43f79835/cryptography-44.0.3-pp311-pypy311_pp73-manylinux_2_34_aarch64.whl", hash = "sha256:dd3db61b8fe5be220eee484a17233287d0be6932d056cf5738225b9c05ef4fff", size = 3898034, upload-time = "2025-05-02T19:35:54.72Z" },
    { url = "https://files.pythonhosted.org/packages/0b/7f/adf62e0b8e8d04d50c9a91282a57628c00c54d4ae75e2b02a223bd1f2613/cryptography-44.0.3-pp311-pypy311_pp73-manylinux_2_34_x86_64.whl", hash = "sha256:978631ec51a6bbc0b7e58f23b68a8ce9e5f09721940933e9c217068388789fe5", size = 4114449, upload-time = "2025-05-02T19:35:57.139Z" },
    { url = "https://files.pythonhosted.org/packages/87/62/d69eb4a8ee231f4bf733a92caf9da13f1c81a44e874b1d4080c25ecbb723/cryptography-44.0.3-pp311-pypy311_pp73-win_amd64.whl", hash = "sha256:5d20cc348cca3a8aa7312f42ab953a56e15323800ca3ab0706b8cd452a3a056c", size = 3134369, upload-time = "2025-05-02T19:35:58.907Z" },
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
    { name = "typing-extensions", marker = "python_full_version < '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/0b/9f/a65090624ecf468cdca03533906e7c69ed7588582240cfe7cc9e770b50eb/exceptiongroup-1.3.0.tar.gz", hash = "sha256:b241f5885f560bc56a59ee63ca4c6a8bfa46ae4ad651af316d4e81817bb9fd88", size = 29749, upload-time = "2025-05-10T17:42:51.123Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/36/f4/c6e662dade71f56cd2f3735141b265c3c79293c109549c1e6933b0651ffc/exceptiongroup-1.3.0-py3-none-any.whl", hash = "sha256:4d111e6e0c13d0644cad6ddaa7ed0261a0b36971f6d23e7ec9b4b9097da78a10", size = 16674, upload-time = "2025-05-10T17:42:49.33Z" },
]

[[package]]
name = "forbiddenfruit"
version = "0.1.4"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/e6/79/d4f20e91327c98096d605646bdc6a5ffedae820f38d378d3515c42ec5e60/forbiddenfruit-0.1.4.tar.gz", hash = "sha256:e3f7e66561a29ae129aac139a85d610dbf3dd896128187ed5454b6421f624253", size = 43756, upload-time = "2021-01-16T21:03:35.401Z" }

[[package]]
name = "googleapis-common-protos"
version = "1.71.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "protobuf", marker = "python_full_version >= '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/30/43/b25abe02db2911397819003029bef768f68a974f2ece483e6084d1a5f754/googleapis_common_protos-1.71.0.tar.gz", hash = "sha256:1aec01e574e29da63c80ba9f7bbf1ccfaacf1da877f23609fe236ca7c72a2e2e", size = 146454, upload-time = "2025-10-20T14:58:08.732Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/25/e8/eba9fece11d57a71e3e22ea672742c8f3cf23b35730c9e96db768b295216/googleapis_common_protos-1.71.0-py3-none-any.whl", hash = "sha256:59034a1d849dc4d18971997a72ac56246570afdd17f9369a0ff68218d50ab78c", size = 294576, upload-time = "2025-10-20T14:56:21.295Z" },
]

[[package]]
name = "grpcio"
version = "1.76.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "typing-extensions", marker = "python_full_version >= '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/b6/e0/318c1ce3ae5a17894d5791e87aea147587c9e702f24122cc7a5c8bbaeeb1/grpcio-1.76.0.tar.gz", hash = "sha256:7be78388d6da1a25c0d5ec506523db58b18be22d9c37d8d3a32c08be4987bd73", size = 12785182, upload-time = "2025-10-21T16:23:12.106Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/88/17/ff4795dc9a34b6aee6ec379f1b66438a3789cd1315aac0cbab60d92f74b3/grpcio-1.76.0-cp310-cp310-linux_armv7l.whl", hash = "sha256:65a20de41e85648e00305c1bb09a3598f840422e522277641145a32d42dcefcc", size = 5840037, upload-time = "2025-10-21T16:20:25.069Z" },
    { url = "https://files.pythonhosted.org/packages/4e/ff/35f9b96e3fa2f12e1dcd58a4513a2e2294a001d64dec81677361b7040c9a/grpcio-1.76.0-cp310-cp310-macosx_11_0_universal2.whl", hash = "sha256:40ad3afe81676fd9ec6d9d406eda00933f218038433980aa19d401490e46ecde", size = 11836482, upload-time = "2025-10-21T16:20:30.113Z" },
    { url = "https://files.pythonhosted.org/packages/3e/1c/8374990f9545e99462caacea5413ed783014b3b66ace49e35c533f07507b/grpcio-1.76.0-cp310-cp310-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:035d90bc79eaa4bed83f524331d55e35820725c9fbb00ffa1904d5550ed7ede3", size = 6407178, upload-time = "2025-10-21T16:20:32.733Z" },
    { url = "https://files.pythonhosted.org/packages/1e/77/36fd7d7c75a6c12542c90a6d647a27935a1ecaad03e0ffdb7c42db6b04d2/grpcio-1.76.0-cp310-cp310-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:4215d3a102bd95e2e11b5395c78562967959824156af11fa93d18fdd18050990", size = 7075684, upload-time = "2025-10-21T16:20:35.435Z" },
    { url = "https://files.pythonhosted.org/packages/38/f7/e3cdb252492278e004722306c5a8935eae91e64ea11f0af3437a7de2e2b7/grpcio-1.76.0-cp310-cp310-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:49ce47231818806067aea3324d4bf13825b658ad662d3b25fada0bdad9b8a6af", size = 6611133, upload-time = "2025-10-21T16:20:37.541Z" },
    { url = "https://files.pythonhosted.org/packages/7e/20/340db7af162ccd20a0893b5f3c4a5d676af7b71105517e62279b5b61d95a/grpcio-1.76.0-cp310-cp310-musllinux_1_2_aarch64.whl", hash = "sha256:8cc3309d8e08fd79089e13ed4819d0af72aa935dd8f435a195fd152796752ff2", size = 7195507, upload-time = "2025-10-21T16:20:39.643Z" },
    { url = "https://files.pythonhosted.org/packages/10/f0/b2160addc1487bd8fa4810857a27132fb4ce35c1b330c2f3ac45d697b106/grpcio-1.76.0-cp310-cp310-musllinux_1_2_i686.whl", hash = "sha256:971fd5a1d6e62e00d945423a567e42eb1fa678ba89072832185ca836a94daaa6", size = 8160651, upload-time = "2025-10-21T16:20:42.492Z" },
    { url = "https://files.pythonhosted.org/packages/2c/2c/ac6f98aa113c6ef111b3f347854e99ebb7fb9d8f7bb3af1491d438f62af4/grpcio-1.76.0-cp310-cp310-musllinux_1_2_x86_64.whl", hash = "sha256:9d9adda641db7207e800a7f089068f6f645959f2df27e870ee81d44701dd9db3", size = 7620568, upload-time = "2025-10-21T16:20:45.995Z" },
    { url = "https://files.pythonhosted.org/packages/90/84/7852f7e087285e3ac17a2703bc4129fafee52d77c6c82af97d905566857e/grpcio-1.76.0-cp310-cp310-win32.whl", hash = "sha256:063065249d9e7e0782d03d2bca50787f53bd0fb89a67de9a7b521c4a01f1989b", size = 3998879, upload-time = "2025-10-21T16:20:48.592Z" },
    { url = "https://files.pythonhosted.org/packages/10/30/d3d2adcbb6dd3ff59d6ac3df6ef830e02b437fb5c90990429fd180e52f30/grpcio-1.76.0-cp310-cp310-win_amd64.whl", hash = "sha256:a6ae758eb08088d36812dd5d9af7a9859c05b1e0f714470ea243694b49278e7b", size = 4706892, upload-time = "2025-10-21T16:20:50.697Z" },
    { url = "https://files.pythonhosted.org/packages/a0/00/8163a1beeb6971f66b4bbe6ac9457b97948beba8dd2fc8e1281dce7f79ec/grpcio-1.76.0-cp311-cp311-linux_armv7l.whl", hash = "sha256:2e1743fbd7f5fa713a1b0a8ac8ebabf0ec980b5d8809ec358d488e273b9cf02a", size = 5843567, upload-time = "2025-10-21T16:20:52.829Z" },
    { url = "https://files.pythonhosted.org/packages/10/c1/934202f5cf335e6d852530ce14ddb0fef21be612ba9ecbbcbd4d748ca32d/grpcio-1.76.0-cp311-cp311-macosx_11_0_universal2.whl", hash = "sha256:a8c2cf1209497cf659a667d7dea88985e834c24b7c3b605e6254cbb5076d985c", size = 11848017, upload-time = "2025-10-21T16:20:56.705Z" },
    { url = "https://files.pythonhosted.org/packages/11/0b/8dec16b1863d74af6eb3543928600ec2195af49ca58b16334972f6775663/grpcio-1.76.0-cp311-cp311-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:08caea849a9d3c71a542827d6df9d5a69067b0a1efbea8a855633ff5d9571465", size = 6412027, upload-time = "2025-10-21T16:20:59.3Z" },
    { url = "https://files.pythonhosted.org/packages/d7/64/7b9e6e7ab910bea9d46f2c090380bab274a0b91fb0a2fe9b0cd399fffa12/grpcio-1.76.0-cp311-cp311-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:f0e34c2079d47ae9f6188211db9e777c619a21d4faba6977774e8fa43b085e48", size = 7075913, upload-time = "2025-10-21T16:21:01.645Z" },
    { url = "https://files.pythonhosted.org/packages/68/86/093c46e9546073cefa789bd76d44c5cb2abc824ca62af0c18be590ff13ba/grpcio-1.76.0-cp311-cp311-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:8843114c0cfce61b40ad48df65abcfc00d4dba82eae8718fab5352390848c5da", size = 6615417, upload-time = "2025-10-21T16:21:03.844Z" },
    { url = "https://files.pythonhosted.org/packages/f7/b6/5709a3a68500a9c03da6fb71740dcdd5ef245e39266461a03f31a57036d8/grpcio-1.76.0-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:8eddfb4d203a237da6f3cc8a540dad0517d274b5a1e9e636fd8d2c79b5c1d397", size = 7199683, upload-time = "2025-10-21T16:21:06.195Z" },
    { url = "https://files.pythonhosted.org/packages/91/d3/4b1f2bf16ed52ce0b508161df3a2d186e4935379a159a834cb4a7d687429/grpcio-1.76.0-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:32483fe2aab2c3794101c2a159070584e5db11d0aa091b2c0ea9c4fc43d0d749", size = 8163109, upload-time = "2025-10-21T16:21:08.498Z" },
    { url = "https://files.pythonhosted.org/packages/5c/61/d9043f95f5f4cf085ac5dd6137b469d41befb04bd80280952ffa2a4c3f12/grpcio-1.76.0-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:dcfe41187da8992c5f40aa8c5ec086fa3672834d2be57a32384c08d5a05b4c00", size = 7626676, upload-time = "2025-10-21T16:21:10.693Z" },
    { url = "https://files.pythonhosted.org/packages/36/95/fd9a5152ca02d8881e4dd419cdd790e11805979f499a2e5b96488b85cf27/grpcio-1.76.0-cp311-cp311-win32.whl", hash = "sha256:2107b0c024d1b35f4083f11245c0e23846ae64d02f40b2b226684840260ed054", size = 3997688, upload-time = "2025-10-21T16:21:12.746Z" },
    { url = "https://files.pythonhosted.org/packages/60/9c/5c359c8d4c9176cfa3c61ecd4efe5affe1f38d9bae81e81ac7186b4c9cc8/grpcio-1.76.0-cp311-cp311-win_amd64.whl", hash = "sha256:522175aba7af9113c48ec10cc471b9b9bd4f6ceb36aeb4544a8e2c80ed9d252d", size = 4709315, upload-time = "2025-10-21T16:21:15.26Z" },
    { url = "https://files.pythonhosted.org/packages/bf/05/8e29121994b8d959ffa0afd28996d452f291b48cfc0875619de0bde2c50c/grpcio-1.76.0-cp312-cp312-linux_armv7l.whl", hash = "sha256:81fd9652b37b36f16138611c7e884eb82e0cec137c40d3ef7c3f9b3ed00f6ed8", size = 5799718, upload-time = "2025-10-21T16:21:17.939Z" },
    { url = "https://files.pythonhosted.org/packages/d9/75/11d0e66b3cdf998c996489581bdad8900db79ebd83513e45c19548f1cba4/grpcio-1.76.0-cp312-cp312-macosx_11_0_universal2.whl", hash = "sha256:04bbe1bfe3a68bbfd4e52402ab7d4eb59d72d02647ae2042204326cf4bbad280", size = 11825627, upload-time = "2025-10-21T16:21:20.466Z" },
    { url = "https://files.pythonhosted.org/packages/28/50/2f0aa0498bc188048f5d9504dcc5c2c24f2eb1a9337cd0fa09a61a2e75f0/grpcio-1.76.0-cp312-cp312-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:d388087771c837cdb6515539f43b9d4bf0b0f23593a24054ac16f7a960be16f4", size = 6359167, upload-time = "2025-10-21T16:21:23.122Z" },
    { url = "https://files.pythonhosted.org/packages/66/e5/bbf0bb97d29ede1d59d6588af40018cfc345b17ce979b7b45424628dc8bb/grpcio-1.76.0-cp312-cp312-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:9f8f757bebaaea112c00dba718fc0d3260052ce714e25804a03f93f5d1c6cc11", size = 7044267, upload-time = "2025-10-21T16:21:25.995Z" },
    { url = "https://files.pythonhosted.org/packages/f5/86/f6ec2164f743d9609691115ae8ece098c76b894ebe4f7c94a655c6b03e98/grpcio-1.76.0-cp312-cp312-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:980a846182ce88c4f2f7e2c22c56aefd515daeb36149d1c897f83cf57999e0b6", size = 6573963, upload-time = "2025-10-21T16:21:28.631Z" },
    { url = "https://files.pythonhosted.org/packages/60/bc/8d9d0d8505feccfdf38a766d262c71e73639c165b311c9457208b56d92ae/grpcio-1.76.0-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:f92f88e6c033db65a5ae3d97905c8fea9c725b63e28d5a75cb73b49bda5024d8", size = 7164484, upload-time = "2025-10-21T16:21:30.837Z" },
    { url = "https://files.pythonhosted.org/packages/67/e6/5d6c2fc10b95edf6df9b8f19cf10a34263b7fd48493936fffd5085521292/grpcio-1.76.0-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:4baf3cbe2f0be3289eb68ac8ae771156971848bb8aaff60bad42005539431980", size = 8127777, upload-time = "2025-10-21T16:21:33.577Z" },
    { url = "https://files.pythonhosted.org/packages/3f/c8/dce8ff21c86abe025efe304d9e31fdb0deaaa3b502b6a78141080f206da0/grpcio-1.76.0-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:615ba64c208aaceb5ec83bfdce7728b80bfeb8be97562944836a7a0a9647d882", size = 7594014, upload-time = "2025-10-21T16:21:41.882Z" },
    { url = "https://files.pythonhosted.org/packages/e0/42/ad28191ebf983a5d0ecef90bab66baa5a6b18f2bfdef9d0a63b1973d9f75/grpcio-1.76.0-cp312-cp312-win32.whl", hash = "sha256:45d59a649a82df5718fd9527ce775fd66d1af35e6d31abdcdc906a49c6822958", size = 3984750, upload-time = "2025-10-21T16:21:44.006Z" },
    { url = "https://files.pythonhosted.org/packages/9e/00/7bd478cbb851c04a48baccaa49b75abaa8e4122f7d86da797500cccdd771/grpcio-1.76.0-cp312-cp312-win_amd64.whl", hash = "sha256:c088e7a90b6017307f423efbb9d1ba97a22aa2170876223f9709e9d1de0b5347", size = 4704003, upload-time = "2025-10-21T16:21:46.244Z" },
    { url = "https://files.pythonhosted.org/packages/fc/ed/71467ab770effc9e8cef5f2e7388beb2be26ed642d567697bb103a790c72/grpcio-1.76.0-cp313-cp313-linux_armv7l.whl", hash = "sha256:26ef06c73eb53267c2b319f43e6634c7556ea37672029241a056629af27c10e2", size = 5807716, upload-time = "2025-10-21T16:21:48.475Z" },
    { url = "https://files.pythonhosted.org/packages/2c/85/c6ed56f9817fab03fa8a111ca91469941fb514e3e3ce6d793cb8f1e1347b/grpcio-1.76.0-cp313-cp313-macosx_11_0_universal2.whl", hash = "sha256:45e0111e73f43f735d70786557dc38141185072d7ff8dc1829d6a77ac1471468", size = 11821522, upload-time = "2025-10-21T16:21:51.142Z" },
    { url = "https://files.pythonhosted.org/packages/ac/31/2b8a235ab40c39cbc141ef647f8a6eb7b0028f023015a4842933bc0d6831/grpcio-1.76.0-cp313-cp313-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:83d57312a58dcfe2a3a0f9d1389b299438909a02db60e2f2ea2ae2d8034909d3", size = 6362558, upload-time = "2025-10-21T16:21:54.213Z" },
    { url = "https://files.pythonhosted.org/packages/bd/64/9784eab483358e08847498ee56faf8ff6ea8e0a4592568d9f68edc97e9e9/grpcio-1.76.0-cp313-cp313-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:3e2a27c89eb9ac3d81ec8835e12414d73536c6e620355d65102503064a4ed6eb", size = 7049990, upload-time = "2025-10-21T16:21:56.476Z" },
    { url = "https://files.pythonhosted.org/packages/2b/94/8c12319a6369434e7a184b987e8e9f3b49a114c489b8315f029e24de4837/grpcio-1.76.0-cp313-cp313-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:61f69297cba3950a524f61c7c8ee12e55c486cb5f7db47ff9dcee33da6f0d3ae", size = 6575387, upload-time = "2025-10-21T16:21:59.051Z" },
    { url = "https://files.pythonhosted.org/packages/15/0f/f12c32b03f731f4a6242f771f63039df182c8b8e2cf8075b245b409259d4/grpcio-1.76.0-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:6a15c17af8839b6801d554263c546c69c4d7718ad4321e3166175b37eaacca77", size = 7166668, upload-time = "2025-10-21T16:22:02.049Z" },
    { url = "https://files.pythonhosted.org/packages/ff/2d/3ec9ce0c2b1d92dd59d1c3264aaec9f0f7c817d6e8ac683b97198a36ed5a/grpcio-1.76.0-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:25a18e9810fbc7e7f03ec2516addc116a957f8cbb8cbc95ccc80faa072743d03", size = 8124928, upload-time = "2025-10-21T16:22:04.984Z" },
    { url = "https://files.pythonhosted.org/packages/1a/74/fd3317be5672f4856bcdd1a9e7b5e17554692d3db9a3b273879dc02d657d/grpcio-1.76.0-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:931091142fd8cc14edccc0845a79248bc155425eee9a98b2db2ea4f00a235a42", size = 7589983, upload-time = "2025-10-21T16:22:07.881Z" },
    { url = "https://files.pythonhosted.org/packages/45/bb/ca038cf420f405971f19821c8c15bcbc875505f6ffadafe9ffd77871dc4c/grpcio-1.76.0-cp313-cp313-win32.whl", hash = "sha256:5e8571632780e08526f118f74170ad8d50fb0a48c23a746bef2a6ebade3abd6f", size = 3984727, upload-time = "2025-10-21T16:22:10.032Z" },
    { url = "https://files.pythonhosted.org/packages/41/80/84087dc56437ced7cdd4b13d7875e7439a52a261e3ab4e06488ba6173b0a/grpcio-1.76.0-cp313-cp313-win_amd64.whl", hash = "sha256:f9f7bd5faab55f47231ad8dba7787866b69f5e93bc306e3915606779bbfb4ba8", size = 4702799, upload-time = "2025-10-21T16:22:12.709Z" },
    { url = "https://files.pythonhosted.org/packages/b4/46/39adac80de49d678e6e073b70204091e76631e03e94928b9ea4ecf0f6e0e/grpcio-1.76.0-cp314-cp314-linux_armv7l.whl", hash = "sha256:ff8a59ea85a1f2191a0ffcc61298c571bc566332f82e5f5be1b83c9d8e668a62", size = 5808417, upload-time = "2025-10-21T16:22:15.02Z" },
    { url = "https://files.pythonhosted.org/packages/9c/f5/a4531f7fb8b4e2a60b94e39d5d924469b7a6988176b3422487be61fe2998/grpcio-1.76.0-cp314-cp314-macosx_11_0_universal2.whl", hash = "sha256:06c3d6b076e7b593905d04fdba6a0525711b3466f43b3400266f04ff735de0cd", size = 11828219, upload-time = "2025-10-21T16:22:17.954Z" },
    { url = "https://files.pythonhosted.org/packages/4b/1c/de55d868ed7a8bd6acc6b1d6ddc4aa36d07a9f31d33c912c804adb1b971b/grpcio-1.76.0-cp314-cp314-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:fd5ef5932f6475c436c4a55e4336ebbe47bd3272be04964a03d316bbf4afbcbc", size = 6367826, upload-time = "2025-10-21T16:22:20.721Z" },
    { url = "https://files.pythonhosted.org/packages/59/64/99e44c02b5adb0ad13ab3adc89cb33cb54bfa90c74770f2607eea629b86f/grpcio-1.76.0-cp314-cp314-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:b331680e46239e090f5b3cead313cc772f6caa7d0fc8de349337563125361a4a", size = 7049550, upload-time = "2025-10-21T16:22:23.637Z" },
    { url = "https://files.pythonhosted.org/packages/43/28/40a5be3f9a86949b83e7d6a2ad6011d993cbe9b6bd27bea881f61c7788b6/grpcio-1.76.0-cp314-cp314-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:2229ae655ec4e8999599469559e97630185fdd53ae1e8997d147b7c9b2b72cba", size = 6575564, upload-time = "2025-10-21T16:22:26.016Z" },
    { url = "https://files.pythonhosted.org/packages/4b/a9/1be18e6055b64467440208a8559afac243c66a8b904213af6f392dc2212f/grpcio-1.76.0-cp314-cp314-musllinux_1_2_aarch64.whl", hash = "sha256:490fa6d203992c47c7b9e4a9d39003a0c2bcc1c9aa3c058730884bbbb0ee9f09", size = 7176236, upload-time = "2025-10-21T16:22:28.362Z" },
    { url = "https://files.pythonhosted.org/packages/0f/55/dba05d3fcc151ce6e81327541d2cc8394f442f6b350fead67401661bf041/grpcio-1.76.0-cp314-cp314-musllinux_1_2_i686.whl", hash = "sha256:479496325ce554792dba6548fae3df31a72cef7bad71ca2e12b0e58f9b336bfc", size = 8125795, upload-time = "2025-10-21T16:22:31.075Z" },
    { url = "https://files.pythonhosted.org/packages/4a/45/122df922d05655f63930cf42c9e3f72ba20aadb26c100ee105cad4ce4257/grpcio-1.76.0-cp314-cp314-musllinux_1_2_x86_64.whl", hash = "sha256:1c9b93f79f48b03ada57ea24725d83a30284a012ec27eab2cf7e50a550cbbbcc", size = 7592214, upload-time = "2025-10-21T16:22:33.831Z" },
    { url = "https://files.pythonhosted.org/packages/4a/6e/0b899b7f6b66e5af39e377055fb4a6675c9ee28431df5708139df2e93233/grpcio-1.76.0-cp314-cp314-win32.whl", hash = "sha256:747fa73efa9b8b1488a95d0ba1039c8e2dca0f741612d80415b1e1c560febf4e", size = 4062961, upload-time = "2025-10-21T16:22:36.468Z" },
    { url = "https://files.pythonhosted.org/packages/19/41/0b430b01a2eb38ee887f88c1f07644a1df8e289353b78e82b37ef988fb64/grpcio-1.76.0-cp314-cp314-win_amd64.whl", hash = "sha256:922fa70ba549fce362d2e2871ab542082d66e2aaf0c19480ea453905b01f384e", size = 4834462, upload-time = "2025-10-21T16:22:39.772Z" },
]

[[package]]
name = "grpcio-tools"
version = "1.75.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "grpcio", marker = "python_full_version >= '3.11'" },
    { name = "protobuf", marker = "python_full_version >= '3.11'" },
    { name = "setuptools", marker = "python_full_version >= '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/7d/76/0cd2a2bb379275c319544a3ab613dc3cea7a167503908c1b4de55f82bd9e/grpcio_tools-1.75.1.tar.gz", hash = "sha256:bb78960cf3d58941e1fec70cbdaccf255918beed13c34112a6915a6d8facebd1", size = 5390470, upload-time = "2025-09-26T09:10:11.948Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/26/b7/7d1b0b7669f993a6a393083a876937f478ca034c283eb23baf6720d8c85a/grpcio_tools-1.75.1-cp310-cp310-linux_armv7l.whl", hash = "sha256:ae0f04d5ec8b8e13476bf516a08fc1de4e58c6bf79f99123a6b964ca7d02c790", size = 2545419, upload-time = "2025-09-26T09:07:44.432Z" },
    { url = "https://files.pythonhosted.org/packages/3e/c0/db5d052d1ba5e859c833d1366960d784c0b44c8330012717aeaa123b6b9f/grpcio_tools-1.75.1-cp310-cp310-macosx_11_0_universal2.whl", hash = "sha256:24a881ad7292e904fc256892b647da17d9137ef2e72faf8b7c8e515314ad1377", size = 5841650, upload-time = "2025-09-26T09:07:50.987Z" },
    { url = "https://files.pythonhosted.org/packages/af/13/ab49230ef106f2b9de156a813bc14049e6fd4fe9c26fa0cde496f0e86a09/grpcio_tools-1.75.1-cp310-cp310-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:1b5810ace274dba12ecfac69ac32c8047c6ee0200a23274cb4885ed4187271f8", size = 2591560, upload-time = "2025-09-26T09:07:52.777Z" },
    { url = "https://files.pythonhosted.org/packages/29/fd/1fd3069fb0559c2f90d85b0fd3a73adc3f63966c6300fee01e4e52740229/grpcio_tools-1.75.1-cp310-cp310-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:ab33993288b97b1180e092fa447a8ce00fbc8c59d67b23553245b88d14fe36bb", size = 2904895, upload-time = "2025-09-26T09:07:55.002Z" },
    { url = "https://files.pythonhosted.org/packages/d6/51/e58fae40132a4589819c388333545a33a89f91b8affcac45623ace9ca659/grpcio_tools-1.75.1-cp310-cp310-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:4cac693621043ef11d3ab2318e811d919779f8cd5011ba8e37f44c178c831d94", size = 2656151, upload-time = "2025-09-26T09:07:56.762Z" },
    { url = "https://files.pythonhosted.org/packages/39/c9/a33736c2a8ceee39991f2c9f67a426ab799c6caf09145120bae6080428d5/grpcio_tools-1.75.1-cp310-cp310-musllinux_1_2_aarch64.whl", hash = "sha256:a09cd5d267b296af67116fe098633ad770bc8c19831a5f3c896f65fad90c1064", size = 3105152, upload-time = "2025-09-26T09:07:58.702Z" },
    { url = "https://files.pythonhosted.org/packages/ac/71/2f09cbe321f057a47a8ae4dacf004bbe8a171fd712b8f7f689ec7b2f1c49/grpcio_tools-1.75.1-cp310-cp310-musllinux_1_2_i686.whl", hash = "sha256:dff4bcb4d16cf9ef745c1984394ed15187e6c23d73d71377377deaf443d11358", size = 3654551, upload-time = "2025-09-26T09:08:00.87Z" },
    { url = "https://files.pythonhosted.org/packages/05/54/91481a5b96cab2a81326ab1041fd7cfc6b6ce0cd82bab14ebcdb6ed78d4e/grpcio_tools-1.75.1-cp310-cp310-musllinux_1_2_x86_64.whl", hash = "sha256:16d5986b37e2a9203f85e456c7ff8705b932718021d408adfe4a79e0f4d95949", size = 3322220, upload-time = "2025-09-26T09:08:02.724Z" },
    { url = "https://files.pythonhosted.org/packages/a2/07/272955f15a35ef0069ebe17a5fc14282c6dc6690edeb4dbe6a81dd2d1efb/grpcio_tools-1.75.1-cp310-cp310-win32.whl", hash = "sha256:3fbac14998bfadc6b9140b6339dbc5f673700ebb4d45ba0c4d4fbe0ffb8559a9", size = 992986, upload-time = "2025-09-26T09:08:04.6Z" },
    { url = "https://files.pythonhosted.org/packages/16/9a/482b05c1277b3385be7e426f000efb921ce3ae76bcb8aa4f9b9f724c58d3/grpcio_tools-1.75.1-cp310-cp310-win_amd64.whl", hash = "sha256:b56e495844eb899de721eb77d9e077192bdeb40842f598481d32a8f6de3db124", size = 1157427, upload-time = "2025-09-26T09:08:06.246Z" },
    { url = "https://files.pythonhosted.org/packages/45/28/71ab934662d41ded4e451d9af0ec6f9aade3525e470fdfd10bd20e588e44/grpcio_tools-1.75.1-cp311-cp311-linux_armv7l.whl", hash = "sha256:f0635231feb70a9d551452829943a1a5fa651283e7a300aadc22df5ea5da696f", size = 2545461, upload-time = "2025-09-26T09:08:08.514Z" },
    { url = "https://files.pythonhosted.org/packages/69/40/d90f6fdb51f51b2a518401207b3920fcfdfa996ed7bca844096f111ed839/grpcio_tools-1.75.1-cp311-cp311-macosx_11_0_universal2.whl", hash = "sha256:626293296ef7e2d87ab1a80b81a55eef91883c65b59a97576099a28b9535100b", size = 5842958, upload-time = "2025-09-26T09:08:11.468Z" },
    { url = "https://files.pythonhosted.org/packages/b4/b7/52e6f32fd0101e3ac9c654a6441b254ba5874f146b543b20afbcb8246947/grpcio_tools-1.75.1-cp311-cp311-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:071339d90f1faab332ce4919c815a10b9c3ed2c09473f550f686bf9cc148579f", size = 2591669, upload-time = "2025-09-26T09:08:13.481Z" },
    { url = "https://files.pythonhosted.org/packages/0a/3c/115c59a5c0c8e9d7d99a40bac8d5e91c05b6735b3bb185265d40e9fc4346/grpcio_tools-1.75.1-cp311-cp311-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:44195f58c052fa935b78c7438c85cbcd4b273dd685028e4f6d4d7b30d47daad1", size = 2904952, upload-time = "2025-09-26T09:08:15.299Z" },
    { url = "https://files.pythonhosted.org/packages/a9/cd/d2a3583a5b1d71da88f7998f20fb5a0b6fe5bb96bb916a610c29269063b6/grpcio_tools-1.75.1-cp311-cp311-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:860fafdb85726029d646c99859ff7bdca5aae61b5ff038c3bd355fc1ec6b2764", size = 2656311, upload-time = "2025-09-26T09:08:17.094Z" },
    { url = "https://files.pythonhosted.org/packages/aa/09/67b9215d39add550e430c9677bd43c9a315da07ab62fa3a5f44f1cf5bb75/grpcio_tools-1.75.1-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:4559547a0cb3d3db1b982eea87d4656036339b400f48127fef932210672fb59e", size = 3105583, upload-time = "2025-09-26T09:08:19.179Z" },
    { url = "https://files.pythonhosted.org/packages/98/d7/d400b90812470f3dc2466964e62fc03592de46b5c824c82ef5303be60167/grpcio_tools-1.75.1-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:9af65a310807d7f36a8f7cddea142fe97d6dffba74444f38870272f2e5a3a06b", size = 3654677, upload-time = "2025-09-26T09:08:21.227Z" },
    { url = "https://files.pythonhosted.org/packages/9c/93/edf6de71b4f936b3f09461a3286db1f902c6366c5de06ef19a8c2523034a/grpcio_tools-1.75.1-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:8c1de31aefc0585d2f915a7cd0994d153547495b8d79c44c58048a3ede0b65be", size = 3322147, upload-time = "2025-09-26T09:08:23.08Z" },
    { url = "https://files.pythonhosted.org/packages/80/00/0f8c6204e34070e7d4f344b27e4b1b0320dfdd94574f79738a43504d182e/grpcio_tools-1.75.1-cp311-cp311-win32.whl", hash = "sha256:efaf95fcaa5d3ac1bcfe44ceed9e2512eb95b5c8c476569bdbbe2bee4b59c8a9", size = 993388, upload-time = "2025-09-26T09:08:24.708Z" },
    { url = "https://files.pythonhosted.org/packages/b0/ae/6f738154980f606293988a64ef4bb0ea2bb12029a4529464aac56fe2ab99/grpcio_tools-1.75.1-cp311-cp311-win_amd64.whl", hash = "sha256:7cefe76fc35c825f0148d60d2294a527053d0f5dd6a60352419214a8c53223c9", size = 1157907, upload-time = "2025-09-26T09:08:26.537Z" },
    { url = "https://files.pythonhosted.org/packages/ef/a7/581bb204d19a347303ed5e25b19f7d8c6365a28c242fca013d1d6d78ad7e/grpcio_tools-1.75.1-cp312-cp312-linux_armv7l.whl", hash = "sha256:49b68936cf212052eeafa50b824e17731b78d15016b235d36e0d32199000b14c", size = 2546099, upload-time = "2025-09-26T09:08:28.794Z" },
    { url = "https://files.pythonhosted.org/packages/9f/59/ab65998eba14ff9d292c880f6a276fe7d0571bba3bb4ddf66aca1f8438b5/grpcio_tools-1.75.1-cp312-cp312-macosx_11_0_universal2.whl", hash = "sha256:08cb6e568e58b76a2178ad3b453845ff057131fff00f634d7e15dcd015cd455b", size = 5839838, upload-time = "2025-09-26T09:08:31.038Z" },
    { url = "https://files.pythonhosted.org/packages/7e/65/7027f71069b4c1e8c7b46de8c46c297c9d28ef6ed4ea0161e8c82c75d1d0/grpcio_tools-1.75.1-cp312-cp312-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:168402ad29a249092673079cf46266936ec2fb18d4f854d96e9c5fa5708efa39", size = 2592916, upload-time = "2025-09-26T09:08:33.216Z" },
    { url = "https://files.pythonhosted.org/packages/0f/84/1abfb3c679b78c7fca7524031cf9de4c4c509c441b48fd26291ac16dd1af/grpcio_tools-1.75.1-cp312-cp312-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:bbae11c29fcf450730f021bfc14b12279f2f985e2e493ccc2f133108728261db", size = 2905276, upload-time = "2025-09-26T09:08:35.691Z" },
    { url = "https://files.pythonhosted.org/packages/99/cd/7f9e05f1eddccb61bc0ead1e49eb2222441957b02ed11acfcd2f795b03a8/grpcio_tools-1.75.1-cp312-cp312-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:38c6c7d5d4800f636ee691cd073db1606d1a6a76424ca75c9b709436c9c20439", size = 2656424, upload-time = "2025-09-26T09:08:38.255Z" },
    { url = "https://files.pythonhosted.org/packages/29/1d/8b7852771c2467728341f7b9c3ca4ebc76e4e23485c6a3e6d97a8323ad2a/grpcio_tools-1.75.1-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:626f6a61a8f141dde9a657775854d1c0d99509f9a2762b82aa401a635f6ec73d", size = 3108985, upload-time = "2025-09-26T09:08:40.291Z" },
    { url = "https://files.pythonhosted.org/packages/c2/6a/069da89cdf2e97e4558bfceef5b60bf0ef200c443b465e7691869006dd32/grpcio_tools-1.75.1-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:f61a8334ae38d4f98c744a732b89527e5af339d17180e25fff0676060f8709b7", size = 3657940, upload-time = "2025-09-26T09:08:42.437Z" },
    { url = "https://files.pythonhosted.org/packages/c3/e4/ca8dae800c084beb89e2720346f70012d36dfb9df02d8eacd518c06cf4a0/grpcio_tools-1.75.1-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:bd0c3fb40d89a1e24a41974e77c7331e80396ab7cde39bc396a13d6b5e2a750b", size = 3324878, upload-time = "2025-09-26T09:08:45.083Z" },
    { url = "https://files.pythonhosted.org/packages/58/06/cbe923679309bf970923f4a11351ea9e485291b504d7243130fdcfdcb03f/grpcio_tools-1.75.1-cp312-cp312-win32.whl", hash = "sha256:004bc5327593eea48abd03be3188e757c3ca0039079587a6aac24275127cac20", size = 993071, upload-time = "2025-09-26T09:08:46.785Z" },
    { url = "https://files.pythonhosted.org/packages/7c/0c/84d6be007262c5d88a590082f3a1fe62d4b0eeefa10c6cdb3548f3663e80/grpcio_tools-1.75.1-cp312-cp312-win_amd64.whl", hash = "sha256:23952692160b5fe7900653dfdc9858dc78c2c42e15c27e19ee780c8917ba6028", size = 1157506, upload-time = "2025-09-26T09:08:48.844Z" },
    { url = "https://files.pythonhosted.org/packages/47/fa/624bbe1b2ccf4f6044bf3cd314fe2c35f78f702fcc2191dc65519baddca4/grpcio_tools-1.75.1-cp313-cp313-linux_armv7l.whl", hash = "sha256:ca9e116aab0ecf4365fc2980f2e8ae1b22273c3847328b9a8e05cbd14345b397", size = 2545752, upload-time = "2025-09-26T09:08:51.433Z" },
    { url = "https://files.pythonhosted.org/packages/b9/4c/6d884e2337feff0a656e395338019adecc3aa1daeae9d7e8eb54340d4207/grpcio_tools-1.75.1-cp313-cp313-macosx_11_0_universal2.whl", hash = "sha256:9fe87a926b65eb7f41f8738b6d03677cc43185ff77a9d9b201bdb2f673f3fa1e", size = 5838163, upload-time = "2025-09-26T09:08:53.858Z" },
    { url = "https://files.pythonhosted.org/packages/d1/2a/2ba7b6911a754719643ed92ae816a7f989af2be2882b9a9e1f90f4b0e882/grpcio_tools-1.75.1-cp313-cp313-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:45503a6094f91b3fd31c3d9adef26ac514f102086e2a37de797e220a6791ee87", size = 2592148, upload-time = "2025-09-26T09:08:55.86Z" },
    { url = "https://files.pythonhosted.org/packages/88/db/fa613a45c3c7b00f905bd5ad3a93c73194724d0a2dd72adae3be32983343/grpcio_tools-1.75.1-cp313-cp313-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:b01b60b3de67be531a39fd869d7613fa8f178aff38c05e4d8bc2fc530fa58cb5", size = 2905215, upload-time = "2025-09-26T09:08:58.27Z" },
    { url = "https://files.pythonhosted.org/packages/d7/0c/ee4786972bb82f60e4f313bb2227c79c2cd20eb13c94c0263067923cfd12/grpcio_tools-1.75.1-cp313-cp313-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:09e2b9b9488735514777d44c1e4eda813122d2c87aad219f98d5d49b359a8eab", size = 2656251, upload-time = "2025-09-26T09:09:00.249Z" },
    { url = "https://files.pythonhosted.org/packages/77/f1/cc5a50658d705d0b71ff8a4fbbfcc6279d3c95731a2ef7285e13dc40e2fe/grpcio_tools-1.75.1-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:55e60300e62b220fabe6f062fe69f143abaeff3335f79b22b56d86254f3c3c80", size = 3108911, upload-time = "2025-09-26T09:09:02.515Z" },
    { url = "https://files.pythonhosted.org/packages/09/d8/43545f77c4918e778e90bc2c02b3462ac71cee14f29d85cdb69b089538eb/grpcio_tools-1.75.1-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:49ce00fcc6facbbf52bf376e55b8e08810cecd03dab0b3a2986d73117c6f6ee4", size = 3657021, upload-time = "2025-09-26T09:09:05.331Z" },
    { url = "https://files.pythonhosted.org/packages/fc/0b/2ae5925374b66bc8df5b828eff1a5f9459349c83dae1773f0aa9858707e6/grpcio_tools-1.75.1-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:71e95479aea868f8c8014d9dc4267f26ee75388a0d8a552e1648cfa0b53d24b4", size = 3324450, upload-time = "2025-09-26T09:09:07.867Z" },
    { url = "https://files.pythonhosted.org/packages/6e/53/9f887bacbecf892ac5b0b282477ca8cfa5b73911b04259f0d88b52e9a055/grpcio_tools-1.75.1-cp313-cp313-win32.whl", hash = "sha256:fff9d2297416eae8861e53154ccf70a19994e5935e6c8f58ebf431f81cbd8d12", size = 992434, upload-time = "2025-09-26T09:09:09.966Z" },
    { url = "https://files.pythonhosted.org/packages/a5/f0/9979d97002edffdc2a88e5f2e0dccea396dd4a6eab34fa2f705fe43eae2f/grpcio_tools-1.75.1-cp313-cp313-win_amd64.whl", hash = "sha256:1849ddd508143eb48791e81d42ddc924c554d1b4900e06775a927573a8d4267f", size = 1157069, upload-time = "2025-09-26T09:09:12.287Z" },
    { url = "https://files.pythonhosted.org/packages/a6/0b/4ff4ead293f2b016668628a240937828444094778c8037d2bbef700e9097/grpcio_tools-1.75.1-cp314-cp314-linux_armv7l.whl", hash = "sha256:f281b594489184b1f9a337cdfed1fc1ddb8428f41c4b4023de81527e90b38e1e", size = 2545868, upload-time = "2025-09-26T09:09:14.716Z" },
    { url = "https://files.pythonhosted.org/packages/0e/78/aa6bf73a18de5357c01ef87eea92150931586b25196fa4df197a37bae11d/grpcio_tools-1.75.1-cp314-cp314-macosx_11_0_universal2.whl", hash = "sha256:becf8332f391abc62bf4eea488b63be063d76a7cf2ef00b2e36c617d9ee9216b", size = 5838010, upload-time = "2025-09-26T09:09:20.415Z" },
    { url = "https://files.pythonhosted.org/packages/99/65/7eaad673bc971af45e079d3b13c20d9ba9842b8788d31953e3234c2e2cee/grpcio_tools-1.75.1-cp314-cp314-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:a08330f24e5cd7b39541882a95a8ba04ffb4df79e2984aa0cd01ed26dcdccf49", size = 2593170, upload-time = "2025-09-26T09:09:22.889Z" },
    { url = "https://files.pythonhosted.org/packages/e4/db/57e1e29e9186c7ed223ce8a9b609d3f861c4db015efb643dfe60b403c137/grpcio_tools-1.75.1-cp314-cp314-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:6bf3742bd8f102630072ed317d1496f31c454cd85ad19d37a68bd85bf9d5f8b9", size = 2905167, upload-time = "2025-09-26T09:09:25.96Z" },
    { url = "https://files.pythonhosted.org/packages/cd/7b/894f891f3cf19812192f8bbf1e0e1c958055676ecf0a5466a350730a006d/grpcio_tools-1.75.1-cp314-cp314-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:f26028949474feb380460ce52d9d090d00023940c65236294a66c42ac5850e8b", size = 2656210, upload-time = "2025-09-26T09:09:28.786Z" },
    { url = "https://files.pythonhosted.org/packages/99/76/8e48427da93ef243c09629969c7b5a2c59dceb674b6b623c1f5fbaa5c8c5/grpcio_tools-1.75.1-cp314-cp314-musllinux_1_2_aarch64.whl", hash = "sha256:1bd68fb98bf08f11b6c3210834a14eefe585bad959bdba38e78b4ae3b04ba5bd", size = 3109226, upload-time = "2025-09-26T09:09:31.307Z" },
    { url = "https://files.pythonhosted.org/packages/b3/7e/ecf71c316c2a88c2478b7c6372d0f82d05f07edbf0f31b6da613df99ec7c/grpcio_tools-1.75.1-cp314-cp314-musllinux_1_2_i686.whl", hash = "sha256:f1496e21586193da62c3a73cd16f9c63c5b3efd68ff06dab96dbdfefa90d40bf", size = 3657139, upload-time = "2025-09-26T09:09:35.043Z" },
    { url = "https://files.pythonhosted.org/packages/6f/f3/b2613e81da2085f40a989c0601ec9efc11e8b32fcb71b1234b64a18af830/grpcio_tools-1.75.1-cp314-cp314-musllinux_1_2_x86_64.whl", hash = "sha256:14a78b1e36310cdb3516cdf9ee2726107875e0b247e2439d62fc8dc38cf793c1", size = 3324513, upload-time = "2025-09-26T09:09:37.44Z" },
    { url = "https://files.pythonhosted.org/packages/9a/1f/2df4fa8634542524bc22442ffe045d41905dae62cc5dd14408b80c5ac1b8/grpcio_tools-1.75.1-cp314-cp314-win32.whl", hash = "sha256:0e6f916daf222002fb98f9a6f22de0751959e7e76a24941985cc8e43cea77b50", size = 1015283, upload-time = "2025-09-26T09:09:39.461Z" },
    { url = "https://files.pythonhosted.org/packages/23/4f/f27c973ff50486a70be53a3978b6b0244398ca170a4e19d91988b5295d92/grpcio_tools-1.75.1-cp314-cp314-win_amd64.whl", hash = "sha256:878c3b362264588c45eba57ce088755f8b2b54893d41cc4a68cdeea62996da5c", size = 1189364, upload-time = "2025-09-26T09:09:42.036Z" },
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
version = "1.0.9"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "certifi", marker = "python_full_version >= '3.11'" },
    { name = "h11", marker = "python_full_version >= '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/06/94/82699a10bca87a5556c9c59b5963f2d039dbd239f25bc2a63907a05a14cb/httpcore-1.0.9.tar.gz", hash = "sha256:6e34463af53fd2ab5d807f399a9b45ea31c3dfa2276f15a2c3f00afff6e176e8", size = 85484, upload-time = "2025-04-24T22:06:22.219Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/7e/f5/f66802a942d491edb555dd61e3a9961140fd64c90bce1eafd741609d334d/httpcore-1.0.9-py3-none-any.whl", hash = "sha256:2d400746a40668fc9dec9810239072b40b4484b640a8c38fd654a024c7a1bf55", size = 78784, upload-time = "2025-04-24T22:06:20.566Z" },
]

[[package]]
name = "httpx"
version = "0.28.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "anyio", marker = "python_full_version >= '3.11'" },
    { name = "certifi", marker = "python_full_version >= '3.11'" },
    { name = "httpcore", marker = "python_full_version >= '3.11'" },
    { name = "idna", marker = "python_full_version >= '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/b1/df/48c586a5fe32a0f01324ee087459e112ebb7224f646c0b5023f5e79e9956/httpx-0.28.1.tar.gz", hash = "sha256:75e98c5f16b0f35b567856f597f06ff2270a374470a5c2392242528e3e3e42fc", size = 141406, upload-time = "2024-12-06T15:37:23.222Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/2a/39/e50c7c3a983047577ee07d2a9e53faf5a69493943ec3f6a384bdc792deb2/httpx-0.28.1-py3-none-any.whl", hash = "sha256:d909fcccc110f8c7faf814ca82a9a4d816bc5a6dbfea25d6591d6985b8ba59ad", size = 73517, upload-time = "2024-12-06T15:37:21.509Z" },
]

[[package]]
name = "idna"
version = "3.11"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/6f/6d/0703ccc57f3a7233505399edb88de3cbd678da106337b9fcde432b65ed60/idna-3.11.tar.gz", hash = "sha256:795dafcc9c04ed0c1fb032c2aa73654d8e8c5023a7df64a53f39190ada629902", size = 194582, upload-time = "2025-10-12T14:55:20.501Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/0e/61/66938bbb5fc52dbdf84594873d5b51fb1f7c7794e9c0f5bd885f30bc507b/idna-3.11-py3-none-any.whl", hash = "sha256:771a87f49d9defaf64091e6e6fe9c18d4833f140bd19464795bc32d966ca37ea", size = 71008, upload-time = "2025-10-12T14:55:18.883Z" },
]

[[package]]
name = "importlib-metadata"
version = "8.7.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "zipp", marker = "python_full_version >= '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/76/66/650a33bd90f786193e4de4b3ad86ea60b53c89b669a5c7be931fac31cdb0/importlib_metadata-8.7.0.tar.gz", hash = "sha256:d13b81ad223b890aa16c5471f2ac3056cf76c5f10f82d6f9292f0b415f389000", size = 56641, upload-time = "2025-04-27T15:29:01.736Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/20/b0/36bd937216ec521246249be3bf9855081de4c5e06a0c9b4219dbeda50373/importlib_metadata-8.7.0-py3-none-any.whl", hash = "sha256:e5dd1551894c77868a30651cef00984d50e1002d06942a7101d34870c5f02afd", size = 27656, upload-time = "2025-04-27T15:29:00.214Z" },
]

[[package]]
name = "iniconfig"
version = "2.3.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/72/34/14ca021ce8e5dfedc35312d08ba8bf51fdd999c576889fc2c24cb97f4f10/iniconfig-2.3.0.tar.gz", hash = "sha256:c76315c77db068650d49c5b56314774a7804df16fee4402c1f19d6d15d8c4730", size = 20503, upload-time = "2025-10-18T21:55:43.219Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/cb/b1/3846dd7f199d53cb17f49cba7e651e9ce294d8497c8c150530ed11865bb8/iniconfig-2.3.0-py3-none-any.whl", hash = "sha256:f631c04d2c48c52b84d0d0549c99ff3859c98df65b3101406327ecc7d53fbf12", size = 7484, upload-time = "2025-10-18T21:55:41.639Z" },
]

[[package]]
name = "jsonpatch"
version = "1.33"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "jsonpointer", marker = "python_full_version >= '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/42/78/18813351fe5d63acad16aec57f94ec2b70a09e53ca98145589e185423873/jsonpatch-1.33.tar.gz", hash = "sha256:9fcd4009c41e6d12348b4a0ff2563ba56a2923a7dfee731d004e212e1ee5030c", size = 21699, upload-time = "2023-06-26T12:07:29.144Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/73/07/02e16ed01e04a374e644b575638ec7987ae846d25ad97bcc9945a3ee4b0e/jsonpatch-1.33-py2.py3-none-any.whl", hash = "sha256:0ae28c0cd062bbd8b8ecc26d7d164fbbea9652a1a3693f3b956c1eae5145dade", size = 12898, upload-time = "2023-06-16T21:01:28.466Z" },
]

[[package]]
name = "jsonpointer"
version = "3.0.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/6a/0a/eebeb1fa92507ea94016a2a790b93c2ae41a7e18778f85471dc54475ed25/jsonpointer-3.0.0.tar.gz", hash = "sha256:2b2d729f2091522d61c3b31f82e11870f60b68f43fbc705cb76bf4b832af59ef", size = 9114, upload-time = "2024-06-10T19:24:42.462Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/71/92/5e77f98553e9e75130c78900d000368476aed74276eb8ae8796f65f00918/jsonpointer-3.0.0-py2.py3-none-any.whl", hash = "sha256:13e088adc14fca8b6aa8177c044e12701e6ad4b28ff10e65f2267a90109c9942", size = 7595, upload-time = "2024-06-10T19:24:40.698Z" },
]

[[package]]
name = "jsonschema-rs"
version = "0.29.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/b0/b4/33a9b25cad41d1e533c1ab7ff30eaec50628dd1bcb92171b99a2e944d61f/jsonschema_rs-0.29.1.tar.gz", hash = "sha256:a9f896a9e4517630374f175364705836c22f09d5bd5bbb06ec0611332b6702fd", size = 1406679, upload-time = "2025-02-08T21:25:12.639Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/4e/2a/cc53cf158d9ea3629b0c818fdbce9ad2cd8f656f11cafd0e21124487887e/jsonschema_rs-0.29.1-cp310-cp310-macosx_10_12_x86_64.macosx_11_0_arm64.macosx_10_12_universal2.whl", hash = "sha256:7d84c4e1483a103694150b5706d638606443c814662738f14d34ca16948349df", size = 3828955, upload-time = "2025-02-08T21:23:47.957Z" },
    { url = "https://files.pythonhosted.org/packages/83/81/6ff994c6bf223eab69109f367fe8c5551a84f9e17869b6d51682776916fa/jsonschema_rs-0.29.1-cp310-cp310-macosx_10_12_x86_64.whl", hash = "sha256:0e7b72365ae0875c0e99f951ad4cec00c6f5e57b25ed5b8495b8f2c810ad21a6", size = 1968807, upload-time = "2025-02-08T21:23:52.069Z" },
    { url = "https://files.pythonhosted.org/packages/e2/11/28243681ff1337bfe988d5b13acc11bd7f962a16eb5721f4fba86c8c7157/jsonschema_rs-0.29.1-cp310-cp310-manylinux_2_12_i686.manylinux2010_i686.whl", hash = "sha256:7078d3cc635b9832ba282ec4de3af4cdaba4af74691e52aa3ea5f7d1baaa3b76", size = 2066165, upload-time = "2025-02-08T21:23:55.573Z" },
    { url = "https://files.pythonhosted.org/packages/b5/17/263f9dccc3d80b7b43ee564b414155a62f6685eb2e657a3d1ebb68f791af/jsonschema_rs-0.29.1-cp310-cp310-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:30ed43c64e0d732edf32a881f0c1db340c9484f3a28c41f7da96667a49eb0f34", size = 2067619, upload-time = "2025-02-08T21:23:58.528Z" },
    { url = "https://files.pythonhosted.org/packages/3c/a7/00f4fec03168c71206bf9da3ff9d943cb2b44ad95f44255c4ae2efcdd4a0/jsonschema_rs-0.29.1-cp310-cp310-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:24a14806090a5ebf1616a0047eb8049f70f3a830c460e71d5f4a78b300644ec9", size = 2085023, upload-time = "2025-02-08T21:24:00.133Z" },
    { url = "https://files.pythonhosted.org/packages/8f/d9/ae4b6de008b45827e534f90cd68f9f34702bae02f9050101e556683d3c55/jsonschema_rs-0.29.1-cp310-cp310-win32.whl", hash = "sha256:3d739419212e87219e0aa5b9b81eee726e755f606ac63f4795e37efeb9635ed9", size = 1704378, upload-time = "2025-02-08T21:24:02.611Z" },
    { url = "https://files.pythonhosted.org/packages/d1/ca/4b8df88f81f560d712144f1dd2526852fc2d183eb5ae044c524146666d68/jsonschema_rs-0.29.1-cp310-cp310-win_amd64.whl", hash = "sha256:a8fa9007e76cea86877165ebb13ed94648246a185d5eabaf9125e97636bc56e4", size = 1872145, upload-time = "2025-02-08T21:24:05.489Z" },
    { url = "https://files.pythonhosted.org/packages/ad/e2/9c3af8c7d56ff1b6bac88137f60bf02f2814c60d1f658ef06b2ddc2a21b1/jsonschema_rs-0.29.1-cp311-cp311-macosx_10_12_x86_64.macosx_11_0_arm64.macosx_10_12_universal2.whl", hash = "sha256:b4458f1a027ab0c64e91edcb23c48220d60a503e741030bcf260fbbe12979ad2", size = 3828925, upload-time = "2025-02-08T21:24:07.289Z" },
    { url = "https://files.pythonhosted.org/packages/3f/29/f9377e55f10ef173c4cf1c2c88bc30e4a1a4ea1c60659c524903cac85a07/jsonschema_rs-0.29.1-cp311-cp311-macosx_10_12_x86_64.whl", hash = "sha256:faf3d90b5473bf654fd6ffb490bd6fdd2e54f4034f652d1749bee963b3104ce3", size = 1968915, upload-time = "2025-02-08T21:24:09.123Z" },
    { url = "https://files.pythonhosted.org/packages/0f/ae/8c514ebab1d312a2422bece0a1ccca45b82a36131d4cb63e01b4469ac99a/jsonschema_rs-0.29.1-cp311-cp311-manylinux_2_12_i686.manylinux2010_i686.whl", hash = "sha256:e96919483960737ea5cd8d36e0752c63b875459f31ae14b3a6e80df925b74947", size = 2066366, upload-time = "2025-02-08T21:24:10.469Z" },
    { url = "https://files.pythonhosted.org/packages/05/3e/04c6b25ae1b53c8c72eaf35cdda4f84558ca4df011d370b5906a6f56ba7f/jsonschema_rs-0.29.1-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:2e70f1ff7281810327b354ecaeba6cdce7fe498483338207fe7edfae1b21c212", size = 2067599, upload-time = "2025-02-08T21:24:12.006Z" },
    { url = "https://files.pythonhosted.org/packages/1f/78/b9b8934e4db4f43f61e65c5f285432c2d07cb1935ad9df88d5080a4a311b/jsonschema_rs-0.29.1-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:07fef0706a5df7ba5f301a6920b28b0a4013ac06623aed96a6180e95c110b82a", size = 2084926, upload-time = "2025-02-08T21:24:14.544Z" },
    { url = "https://files.pythonhosted.org/packages/5c/ae/676d67d2583cdd50b07b5a0989b501aebf003b12232d14f87fc7fb991f2c/jsonschema_rs-0.29.1-cp311-cp311-win32.whl", hash = "sha256:07524370bdce055d4f106b7fed1afdfc86facd7d004cbb71adeaff3e06861bf6", size = 1704339, upload-time = "2025-02-08T21:24:16.145Z" },
    { url = "https://files.pythonhosted.org/packages/4b/3e/4767dce237d8ea2ff5f684699ef1b9dae5017dc41adaa6f3dc3a85b84608/jsonschema_rs-0.29.1-cp311-cp311-win_amd64.whl", hash = "sha256:36fa23c85333baa8ce5bf0564fb19de3d95b7640c0cab9e3205ddc44a62fdbf0", size = 1872253, upload-time = "2025-02-08T21:24:18.43Z" },
    { url = "https://files.pythonhosted.org/packages/7b/4a/67ea15558ab85e67d1438b2e5da63b8e89b273c457106cbc87f8f4959a3d/jsonschema_rs-0.29.1-cp312-cp312-macosx_10_12_x86_64.macosx_11_0_arm64.macosx_10_12_universal2.whl", hash = "sha256:9fe7529faa6a84d23e31b1f45853631e4d4d991c85f3d50e6d1df857bb52b72d", size = 3825206, upload-time = "2025-02-08T21:24:19.985Z" },
    { url = "https://files.pythonhosted.org/packages/b9/2e/bc75ed65d11ba47200ade9795ebd88eb2e64c2852a36d9be640172563430/jsonschema_rs-0.29.1-cp312-cp312-macosx_10_12_x86_64.whl", hash = "sha256:b5d7e385298f250ed5ce4928fd59fabf2b238f8167f2c73b9414af8143dfd12e", size = 1966302, upload-time = "2025-02-08T21:24:21.673Z" },
    { url = "https://files.pythonhosted.org/packages/95/dd/4a90e96811f897de066c69d95bc0983138056b19cb169f2a99c736e21933/jsonschema_rs-0.29.1-cp312-cp312-manylinux_2_12_i686.manylinux2010_i686.whl", hash = "sha256:64a29be0504731a2e3164f66f609b9999aa66a2df3179ecbfc8ead88e0524388", size = 2062846, upload-time = "2025-02-08T21:24:23.171Z" },
    { url = "https://files.pythonhosted.org/packages/21/91/61834396748a741021716751a786312b8a8319715e6c61421447a07c887c/jsonschema_rs-0.29.1-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:7e91defda5dfa87306543ee9b34d97553d9422c134998c0b64855b381f8b531d", size = 2065564, upload-time = "2025-02-08T21:24:24.574Z" },
    { url = "https://files.pythonhosted.org/packages/f0/2c/920d92e88b9bdb6cb14867a55e5572e7b78bfc8554f9c625caa516aa13dd/jsonschema_rs-0.29.1-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:96f87680a6a1c16000c851d3578534ae3c154da894026c2a09a50f727bd623d4", size = 2083055, upload-time = "2025-02-08T21:24:26.834Z" },
    { url = "https://files.pythonhosted.org/packages/6d/0a/f4c1bea3193992fe4ff9ce330c6a594481caece06b1b67d30b15992bbf54/jsonschema_rs-0.29.1-cp312-cp312-win32.whl", hash = "sha256:bcfc0d52ecca6c1b2fbeede65c1ad1545de633045d42ad0c6699039f28b5fb71", size = 1701065, upload-time = "2025-02-08T21:24:28.282Z" },
    { url = "https://files.pythonhosted.org/packages/5e/89/3f89de071920208c0eb64b827a878d2e587f6a3431b58c02f63c3468b76e/jsonschema_rs-0.29.1-cp312-cp312-win_amd64.whl", hash = "sha256:a414c162d687ee19171e2d8aae821f396d2f84a966fd5c5c757bd47df0954452", size = 1871774, upload-time = "2025-02-08T21:24:30.824Z" },
    { url = "https://files.pythonhosted.org/packages/1b/9b/d642024e8b39753b789598363fd5998eb3053b52755a5df6a021d53741d5/jsonschema_rs-0.29.1-cp313-cp313-macosx_10_12_x86_64.macosx_11_0_arm64.macosx_10_12_universal2.whl", hash = "sha256:0afee5f31a940dec350a33549ec03f2d1eda2da3049a15cd951a266a57ef97ee", size = 3824864, upload-time = "2025-02-08T21:24:32.252Z" },
    { url = "https://files.pythonhosted.org/packages/aa/3d/48a7baa2373b941e89a12e720dae123fd0a663c28c4e82213a29c89a4715/jsonschema_rs-0.29.1-cp313-cp313-macosx_10_12_x86_64.whl", hash = "sha256:c38453a5718bcf2ad1b0163d128814c12829c45f958f9407c69009d8b94a1232", size = 1966084, upload-time = "2025-02-08T21:24:33.8Z" },
    { url = "https://files.pythonhosted.org/packages/1e/e4/f260917a17bb28bb1dec6fa5e869223341fac2c92053aa9bd23c1caaefa0/jsonschema_rs-0.29.1-cp313-cp313-manylinux_2_12_i686.manylinux2010_i686.whl", hash = "sha256:5dc8bdb1067bf4f6d2f80001a636202dc2cea027b8579f1658ce8e736b06557f", size = 2062430, upload-time = "2025-02-08T21:24:35.174Z" },
    { url = "https://files.pythonhosted.org/packages/f5/e7/61353403b76768601d802afa5b7b5902d52c33d1dd0f3159aafa47463634/jsonschema_rs-0.29.1-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:4bcfe23992623a540169d0845ea8678209aa2fe7179941dc7c512efc0c2b6b46", size = 2065443, upload-time = "2025-02-08T21:24:36.778Z" },
    { url = "https://files.pythonhosted.org/packages/40/ed/40b971a09f46a22aa956071ea159413046e9d5fcd280a5910da058acdeb2/jsonschema_rs-0.29.1-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:0f2a526c0deacd588864d3400a0997421dffef6fe1df5cfda4513a453c01ad42", size = 2082606, upload-time = "2025-02-08T21:24:38.388Z" },
    { url = "https://files.pythonhosted.org/packages/bc/59/1c142e1bfb87d57c18fb189149f7aa8edf751725d238d787015278b07600/jsonschema_rs-0.29.1-cp313-cp313-win32.whl", hash = "sha256:68acaefb54f921243552d15cfee3734d222125584243ca438de4444c5654a8a3", size = 1700666, upload-time = "2025-02-08T21:24:40.573Z" },
    { url = "https://files.pythonhosted.org/packages/13/e8/f0ad941286cd350b879dd2b3c848deecd27f0b3fbc0ff44f2809ad59718d/jsonschema_rs-0.29.1-cp313-cp313-win_amd64.whl", hash = "sha256:1c4e5a61ac760a2fc3856a129cc84aa6f8fba7b9bc07b19fe4101050a8ecc33c", size = 1871619, upload-time = "2025-02-08T21:24:42.286Z" },
]

[[package]]
name = "langchain-core"
version = "1.0.3"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "jsonpatch", marker = "python_full_version >= '3.11'" },
    { name = "langsmith", marker = "python_full_version >= '3.11'" },
    { name = "packaging", marker = "python_full_version >= '3.11'" },
    { name = "pydantic", marker = "python_full_version >= '3.11'" },
    { name = "pyyaml", marker = "python_full_version >= '3.11'" },
    { name = "tenacity", marker = "python_full_version >= '3.11'" },
    { name = "typing-extensions", marker = "python_full_version >= '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/41/15/dfe0c2af463d63296fe18608a06570ce3a4b245253d4f26c301481380f7d/langchain_core-1.0.3.tar.gz", hash = "sha256:10744945d21168fb40d1162a5f1cf69bf0137ff6ad2b12c87c199a5297410887", size = 770278, upload-time = "2025-11-03T14:32:09.712Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/f2/1b/b0a37674bdcbd2931944e12ea742fd167098de5212ee2391e91dce631162/langchain_core-1.0.3-py3-none-any.whl", hash = "sha256:64f1bd45f04b174bbfd54c135a8adc52f4902b347c15a117d6383b412bf558a5", size = 469927, upload-time = "2025-11-03T14:32:08.322Z" },
]

[[package]]
name = "langgraph"
version = "1.0.2"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "langchain-core", marker = "python_full_version >= '3.11'" },
    { name = "langgraph-checkpoint", marker = "python_full_version >= '3.11'" },
    { name = "langgraph-prebuilt", marker = "python_full_version >= '3.11'" },
    { name = "langgraph-sdk", marker = "python_full_version >= '3.11'" },
    { name = "pydantic", marker = "python_full_version >= '3.11'" },
    { name = "xxhash", marker = "python_full_version >= '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/0e/25/18e6e056ee1a8af64fcab441b4a3f2e158399935b08f148c7718fc42ecdb/langgraph-1.0.2.tar.gz", hash = "sha256:dae1af08d6025cb1fcaed68f502c01af7d634d9044787c853a46c791cfc52f67", size = 482660, upload-time = "2025-10-29T18:38:28.374Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/d7/b1/9f4912e13d4ed691f2685c8a4b764b5a9237a30cca0c5782bc213d9f0a9a/langgraph-1.0.2-py3-none-any.whl", hash = "sha256:b3d56b8c01de857b5fb1da107e8eab6e30512a377685eeedb4f76456724c9729", size = 156751, upload-time = "2025-10-29T18:38:26.577Z" },
]

[[package]]
name = "langgraph-api"
version = "0.5.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "cloudpickle", marker = "python_full_version >= '3.11'" },
    { name = "cryptography", marker = "python_full_version >= '3.11'" },
    { name = "grpcio", marker = "python_full_version >= '3.11'" },
    { name = "grpcio-tools", marker = "python_full_version >= '3.11'" },
    { name = "httpx", marker = "python_full_version >= '3.11'" },
    { name = "jsonschema-rs", marker = "python_full_version >= '3.11'" },
    { name = "langchain-core", marker = "python_full_version >= '3.11'" },
    { name = "langgraph", marker = "python_full_version >= '3.11'" },
    { name = "langgraph-checkpoint", marker = "python_full_version >= '3.11'" },
    { name = "langgraph-runtime-inmem", marker = "python_full_version >= '3.11'" },
    { name = "langgraph-sdk", marker = "python_full_version >= '3.11'" },
    { name = "langsmith", marker = "python_full_version >= '3.11'" },
    { name = "opentelemetry-api", marker = "python_full_version >= '3.11'" },
    { name = "opentelemetry-exporter-otlp-proto-http", marker = "python_full_version >= '3.11'" },
    { name = "opentelemetry-sdk", marker = "python_full_version >= '3.11'" },
    { name = "orjson", marker = "python_full_version >= '3.11'" },
    { name = "protobuf", marker = "python_full_version >= '3.11'" },
    { name = "pyjwt", marker = "python_full_version >= '3.11'" },
    { name = "sse-starlette", marker = "python_full_version >= '3.11'" },
    { name = "starlette", marker = "python_full_version >= '3.11'" },
    { name = "structlog", marker = "python_full_version >= '3.11'" },
    { name = "tenacity", marker = "python_full_version >= '3.11'" },
    { name = "truststore", marker = "python_full_version >= '3.11'" },
    { name = "uvicorn", marker = "python_full_version >= '3.11'" },
    { name = "watchfiles", marker = "python_full_version >= '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/e3/50/ba0561eb7636ebe7c7cbb51e07add9b35e15bd36adbb4731f218cbbcf9a3/langgraph_api-0.5.1.tar.gz", hash = "sha256:0f3461bf79a03d30a81f96db12898e77fedb0cc4da67f08dda6b6d0b7aeb4a3a", size = 350752, upload-time = "2025-11-03T18:39:07.046Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/35/55/f4cadc8bd77669a7aa810403405daef03ee5d54aed6ba7db2cc0bf923518/langgraph_api-0.5.1-py3-none-any.whl", hash = "sha256:81b9de4b3c3ab76ce7b5c86d6adf6329e2dca503f6c23d337fee5ed1d398364f", size = 262101, upload-time = "2025-11-03T18:39:04.929Z" },
]

[[package]]
name = "langgraph-checkpoint"
version = "3.0.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "langchain-core", marker = "python_full_version >= '3.11'" },
    { name = "ormsgpack", marker = "python_full_version >= '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/b7/cb/2a6dad2f0a14317580cc122e2a60e7f0ecabb50aaa6dc5b7a6a2c94cead7/langgraph_checkpoint-3.0.0.tar.gz", hash = "sha256:f738695ad938878d8f4775d907d9629e9fcd345b1950196effb08f088c52369e", size = 132132, upload-time = "2025-10-20T18:35:49.132Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/85/2a/2efe0b5a72c41e3a936c81c5f5d8693987a1b260287ff1bbebaae1b7b888/langgraph_checkpoint-3.0.0-py3-none-any.whl", hash = "sha256:560beb83e629784ab689212a3d60834fb3196b4bbe1d6ac18e5cad5d85d46010", size = 46060, upload-time = "2025-10-20T18:35:48.255Z" },
]

[[package]]
name = "langgraph-cli"
source = { editable = "." }
dependencies = [
    { name = "click" },
    { name = "langgraph-sdk", marker = "python_full_version >= '3.11'" },
]

[package.optional-dependencies]
inmem = [
    { name = "langgraph-api", marker = "python_full_version >= '3.11'" },
    { name = "langgraph-runtime-inmem", marker = "python_full_version >= '3.11'" },
    { name = "python-dotenv" },
]

[package.dev-dependencies]
dev = [
    { name = "codespell" },
    { name = "msgspec" },
    { name = "mypy" },
    { name = "pytest" },
    { name = "pytest-asyncio" },
    { name = "pytest-mock" },
    { name = "pytest-watch" },
    { name = "ruff" },
]
lint = [
    { name = "codespell" },
    { name = "mypy" },
    { name = "ruff" },
]
test = [
    { name = "msgspec" },
    { name = "pytest" },
    { name = "pytest-asyncio" },
    { name = "pytest-mock" },
    { name = "pytest-watch" },
]

[package.metadata]
requires-dist = [
    { name = "click", specifier = ">=8.1.7" },
    { name = "langgraph-api", marker = "python_full_version >= '3.11' and extra == 'inmem'", specifier = ">=0.4,<0.6.0" },
    { name = "langgraph-runtime-inmem", marker = "python_full_version >= '3.11' and extra == 'inmem'", specifier = ">=0.7" },
    { name = "langgraph-sdk", marker = "python_full_version >= '3.11'", specifier = ">=0.1.0" },
    { name = "python-dotenv", marker = "extra == 'inmem'", specifier = ">=0.8.0" },
]
provides-extras = ["inmem"]

[package.metadata.requires-dev]
dev = [
    { name = "codespell" },
    { name = "msgspec" },
    { name = "mypy" },
    { name = "pytest" },
    { name = "pytest-asyncio" },
    { name = "pytest-mock" },
    { name = "pytest-watch" },
    { name = "ruff" },
]
lint = [
    { name = "codespell" },
    { name = "mypy" },
    { name = "ruff" },
]
test = [
    { name = "msgspec" },
    { name = "pytest" },
    { name = "pytest-asyncio" },
    { name = "pytest-mock" },
    { name = "pytest-watch" },
]

[[package]]
name = "langgraph-prebuilt"
version = "1.0.2"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "langchain-core", marker = "python_full_version >= '3.11'" },
    { name = "langgraph-checkpoint", marker = "python_full_version >= '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/33/2f/b940590436e07b3450fe6d791aad5e581363ad536c4f1771e3ba46530268/langgraph_prebuilt-1.0.2.tar.gz", hash = "sha256:9896dbabf04f086eb59df4294f54ab5bdb21cd78e27e0a10e695dffd1cc6097d", size = 142075, upload-time = "2025-10-29T18:29:00.401Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/27/2f/9a7d00d4afa036e65294059c7c912002fb72ba5dbbd5c2a871ca06360278/langgraph_prebuilt-1.0.2-py3-none-any.whl", hash = "sha256:d9499f7c449fb637ee7b87e3f6a3b74095f4202053c74d33894bd839ea4c57c7", size = 34286, upload-time = "2025-10-29T18:28:59.26Z" },
]

[[package]]
name = "langgraph-runtime-inmem"
version = "0.15.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "blockbuster", marker = "python_full_version >= '3.11'" },
    { name = "langgraph", marker = "python_full_version >= '3.11'" },
    { name = "langgraph-checkpoint", marker = "python_full_version >= '3.11'" },
    { name = "sse-starlette", marker = "python_full_version >= '3.11'" },
    { name = "starlette", marker = "python_full_version >= '3.11'" },
    { name = "structlog", marker = "python_full_version >= '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/46/20/7d30e44fa4c237b2e404875329bffac84d445b3be568aa633bb6f65c545d/langgraph_runtime_inmem-0.15.0.tar.gz", hash = "sha256:e5bd37b1942e7d7c816f0591b48d89dcca06197ed3d1733fbb2b9293bbc4cb88", size = 98010, upload-time = "2025-10-31T19:49:48.475Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/6c/27/57bb2e6a9dadf8da92e56aafcc55cc545af7b65f57d4c9d294c41ccf5e60/langgraph_runtime_inmem-0.15.0-py3-none-any.whl", hash = "sha256:f19c4b635552457359fd91db59e48579ef074e8f75db97a7c68003f63cc76a12", size = 34387, upload-time = "2025-10-31T19:49:47.282Z" },
]

[[package]]
name = "langgraph-sdk"
version = "0.2.9"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "httpx", marker = "python_full_version >= '3.11'" },
    { name = "orjson", marker = "python_full_version >= '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/23/d8/40e01190a73c564a4744e29a6c902f78d34d43dad9b652a363a92a67059c/langgraph_sdk-0.2.9.tar.gz", hash = "sha256:b3bd04c6be4fa382996cd2be8fbc1e7cc94857d2bc6b6f4599a7f2a245975303", size = 99802, upload-time = "2025-09-20T18:49:14.734Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/66/05/b2d34e16638241e6f27a6946d28160d4b8b641383787646d41a3727e0896/langgraph_sdk-0.2.9-py3-none-any.whl", hash = "sha256:fbf302edadbf0fb343596f91c597794e936ef68eebc0d3e1d358b6f9f72a1429", size = 56752, upload-time = "2025-09-20T18:49:13.346Z" },
]

[[package]]
name = "langsmith"
version = "0.4.39"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "httpx", marker = "python_full_version >= '3.11'" },
    { name = "orjson", marker = "python_full_version >= '3.11' and platform_python_implementation != 'PyPy'" },
    { name = "packaging", marker = "python_full_version >= '3.11'" },
    { name = "pydantic", marker = "python_full_version >= '3.11'" },
    { name = "requests", marker = "python_full_version >= '3.11'" },
    { name = "requests-toolbelt", marker = "python_full_version >= '3.11'" },
    { name = "zstandard", marker = "python_full_version >= '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/a7/67/cf7c22d2744286f872aacee2ac13928c46e2ba5d486514d60cd4ab59f58d/langsmith-0.4.39.tar.gz", hash = "sha256:8f2e6bae5cba88f86d8df2a4f95b20a319c65e9945be639302876ab6ef2f13e0", size = 943095, upload-time = "2025-11-01T00:06:18.59Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/1f/38/9a97f650b8cdb2ba0356d65aef9239f4a30db69ae44c30daa2cf8dd3f350/langsmith-0.4.39-py3-none-any.whl", hash = "sha256:48872eaaa449fc10781b5251f4fc05bc7d5c2d1d733a734566a96dd9166108b4", size = 397767, upload-time = "2025-11-01T00:06:16.433Z" },
]

[[package]]
name = "msgspec"
version = "0.19.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/cf/9b/95d8ce458462b8b71b8a70fa94563b2498b89933689f3a7b8911edfae3d7/msgspec-0.19.0.tar.gz", hash = "sha256:604037e7cd475345848116e89c553aa9a233259733ab51986ac924ab1b976f8e", size = 216934, upload-time = "2024-12-27T17:40:28.597Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/13/40/817282b42f58399762267b30deb8ac011d8db373f8da0c212c85fbe62b8f/msgspec-0.19.0-cp310-cp310-macosx_10_9_x86_64.whl", hash = "sha256:d8dd848ee7ca7c8153462557655570156c2be94e79acec3561cf379581343259", size = 190019, upload-time = "2024-12-27T17:39:13.803Z" },
    { url = "https://files.pythonhosted.org/packages/92/99/bd7ed738c00f223a8119928661167a89124140792af18af513e6519b0d54/msgspec-0.19.0-cp310-cp310-macosx_11_0_arm64.whl", hash = "sha256:0553bbc77662e5708fe66aa75e7bd3e4b0f209709c48b299afd791d711a93c36", size = 183680, upload-time = "2024-12-27T17:39:17.847Z" },
    { url = "https://files.pythonhosted.org/packages/e5/27/322badde18eb234e36d4a14122b89edd4e2973cdbc3da61ca7edf40a1ccd/msgspec-0.19.0-cp310-cp310-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:fe2c4bf29bf4e89790b3117470dea2c20b59932772483082c468b990d45fb947", size = 209334, upload-time = "2024-12-27T17:39:19.065Z" },
    { url = "https://files.pythonhosted.org/packages/c6/65/080509c5774a1592b2779d902a70b5fe008532759927e011f068145a16cb/msgspec-0.19.0-cp310-cp310-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:00e87ecfa9795ee5214861eab8326b0e75475c2e68a384002aa135ea2a27d909", size = 211551, upload-time = "2024-12-27T17:39:21.767Z" },
    { url = "https://files.pythonhosted.org/packages/6f/2e/1c23c6b4ca6f4285c30a39def1054e2bee281389e4b681b5e3711bd5a8c9/msgspec-0.19.0-cp310-cp310-musllinux_1_2_aarch64.whl", hash = "sha256:3c4ec642689da44618f68c90855a10edbc6ac3ff7c1d94395446c65a776e712a", size = 215099, upload-time = "2024-12-27T17:39:24.71Z" },
    { url = "https://files.pythonhosted.org/packages/83/fe/95f9654518879f3359d1e76bc41189113aa9102452170ab7c9a9a4ee52f6/msgspec-0.19.0-cp310-cp310-musllinux_1_2_x86_64.whl", hash = "sha256:2719647625320b60e2d8af06b35f5b12d4f4d281db30a15a1df22adb2295f633", size = 218211, upload-time = "2024-12-27T17:39:27.396Z" },
    { url = "https://files.pythonhosted.org/packages/79/f6/71ca7e87a1fb34dfe5efea8156c9ef59dd55613aeda2ca562f122cd22012/msgspec-0.19.0-cp310-cp310-win_amd64.whl", hash = "sha256:695b832d0091edd86eeb535cd39e45f3919f48d997685f7ac31acb15e0a2ed90", size = 186174, upload-time = "2024-12-27T17:39:29.647Z" },
    { url = "https://files.pythonhosted.org/packages/24/d4/2ec2567ac30dab072cce3e91fb17803c52f0a37aab6b0c24375d2b20a581/msgspec-0.19.0-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:aa77046904db764b0462036bc63ef71f02b75b8f72e9c9dd4c447d6da1ed8f8e", size = 187939, upload-time = "2024-12-27T17:39:32.347Z" },
    { url = "https://files.pythonhosted.org/packages/2b/c0/18226e4328897f4f19875cb62bb9259fe47e901eade9d9376ab5f251a929/msgspec-0.19.0-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:047cfa8675eb3bad68722cfe95c60e7afabf84d1bd8938979dd2b92e9e4a9551", size = 182202, upload-time = "2024-12-27T17:39:33.633Z" },
    { url = "https://files.pythonhosted.org/packages/81/25/3a4b24d468203d8af90d1d351b77ea3cffb96b29492855cf83078f16bfe4/msgspec-0.19.0-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:e78f46ff39a427e10b4a61614a2777ad69559cc8d603a7c05681f5a595ea98f7", size = 209029, upload-time = "2024-12-27T17:39:35.023Z" },
    { url = "https://files.pythonhosted.org/packages/85/2e/db7e189b57901955239f7689b5dcd6ae9458637a9c66747326726c650523/msgspec-0.19.0-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:6c7adf191e4bd3be0e9231c3b6dc20cf1199ada2af523885efc2ed218eafd011", size = 210682, upload-time = "2024-12-27T17:39:36.384Z" },
    { url = "https://files.pythonhosted.org/packages/03/97/7c8895c9074a97052d7e4a1cc1230b7b6e2ca2486714eb12c3f08bb9d284/msgspec-0.19.0-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:f04cad4385e20be7c7176bb8ae3dca54a08e9756cfc97bcdb4f18560c3042063", size = 214003, upload-time = "2024-12-27T17:39:39.097Z" },
    { url = "https://files.pythonhosted.org/packages/61/61/e892997bcaa289559b4d5869f066a8021b79f4bf8e955f831b095f47a4cd/msgspec-0.19.0-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:45c8fb410670b3b7eb884d44a75589377c341ec1392b778311acdbfa55187716", size = 216833, upload-time = "2024-12-27T17:39:41.203Z" },
