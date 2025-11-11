    builder = StateGraph(State)
    builder.add_node("double", double)
    builder.set_entry_point("double")
    graph = builder.compile()

    # Define the functional API workflow
    checkpointer = InMemorySaver()

    @entrypoint(checkpointer=checkpointer)
    def workflow(x: int) -> dict:
        result = graph.invoke({"foo": x})
        return {"bar": result["foo"]}

    # Execute the workflow
    config = {"configurable": {"thread_id": str(uuid.uuid4())}}
    print(workflow.invoke(5, config=config))  # Output: {'bar': 10}
    ```
    :::

    :::js
    ```typescript
    import { v4 as uuidv4 } from "uuid";
    import { entrypoint, MemorySaver } from "@langchain/langgraph";
    import { StateGraph } from "@langchain/langgraph";
    import { z } from "zod";

    // Define the shared state type
    const State = z.object({
      foo: z.number(),
    });

    // Build the graph using the Graph API
    const builder = new StateGraph(State)
      .addNode("double", (state) => {
        return { foo: state.foo * 2 };
      })
      .addEdge("__start__", "double");
    const graph = builder.compile();

    // Define the functional API workflow
    const checkpointer = new MemorySaver();

    const workflow = entrypoint(
      { checkpointer, name: "workflow" },
      async (x: number) => {
        const result = await graph.invoke({ foo: x });
        return { bar: result.foo };
      }
    );

    // Execute the workflow
    const config = { configurable: { thread_id: uuidv4() } };
    console.log(await workflow.invoke(5, config)); // Output: { bar: 10 }
    ```
    :::

## Call other entrypoints

You can call other **entrypoints** from within an **entrypoint** or a **task**.

:::python

```python
@entrypoint() # Will automatically use the checkpointer from the parent entrypoint
def some_other_workflow(inputs: dict) -> int:
    return inputs["value"]

@entrypoint(checkpointer=checkpointer)
def my_workflow(inputs: dict) -> int:
    value = some_other_workflow.invoke({"value": 1})
    return value
```

:::

:::js

```typescript
// Will automatically use the checkpointer from the parent entrypoint
const someOtherWorkflow = entrypoint(
  { name: "someOtherWorkflow" },
  async (inputs: { value: number }) => {
    return inputs.value;
  }
);

const myWorkflow = entrypoint(
  { checkpointer, name: "myWorkflow" },
  async (inputs: { value: number }) => {
    const value = await someOtherWorkflow.invoke({ value: 1 });
    return value;
  }
);
```

:::

??? example "Extended example: calling another entrypoint"

    :::python
    ```python
    import uuid
    from langgraph.func import entrypoint
    from langgraph.checkpoint.memory import InMemorySaver

    # Initialize a checkpointer
    checkpointer = InMemorySaver()

    # A reusable sub-workflow that multiplies a number
    @entrypoint()
    def multiply(inputs: dict) -> int:
        return inputs["a"] * inputs["b"]

    # Main workflow that invokes the sub-workflow
    @entrypoint(checkpointer=checkpointer)
    def main(inputs: dict) -> dict:
        result = multiply.invoke({"a": inputs["x"], "b": inputs["y"]})
        return {"product": result}

    # Execute the main workflow
    config = {"configurable": {"thread_id": str(uuid.uuid4())}}
    print(main.invoke({"x": 6, "y": 7}, config=config))  # Output: {'product': 42}
    ```
    :::

    :::js
    ```typescript
    import { v4 as uuidv4 } from "uuid";
    import { entrypoint, MemorySaver } from "@langchain/langgraph";

    // Initialize a checkpointer
    const checkpointer = new MemorySaver();

    // A reusable sub-workflow that multiplies a number
    const multiply = entrypoint(
      { name: "multiply" },
      async (inputs: { a: number; b: number }) => {
        return inputs.a * inputs.b;
      }
    );

    // Main workflow that invokes the sub-workflow
    const main = entrypoint(
      { checkpointer, name: "main" },
      async (inputs: { x: number; y: number }) => {
        const result = await multiply.invoke({ a: inputs.x, b: inputs.y });
        return { product: result };
      }
    );

    // Execute the main workflow
    const config = { configurable: { thread_id: uuidv4() } };
    console.log(await main.invoke({ x: 6, y: 7 }, config)); // Output: { product: 42 }
    ```
    :::

## Streaming

The **Functional API** uses the same streaming mechanism as the **Graph API**. Please
read the [**streaming guide**](../concepts/streaming.md) section for more details.

Example of using the streaming API to stream both updates and custom data.

:::python

```python
from langgraph.func import entrypoint
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.config import get_stream_writer # (1)!

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def main(inputs: dict) -> int:
    writer = get_stream_writer() # (2)!
    writer("Started processing") # (3)!
    result = inputs["x"] * 2
    writer(f"Result is {result}") # (4)!
    return result

config = {"configurable": {"thread_id": "abc"}}

# highlight-next-line
for mode, chunk in main.stream( # (5)!
    {"x": 5},
    stream_mode=["custom", "updates"], # (6)!
    config=config
):
    print(f"{mode}: {chunk}")
```

1. Import `get_stream_writer` from `langgraph.config`.
2. Obtain a stream writer instance within the entrypoint.
3. Emit custom data before computation begins.
4. Emit another custom message after computing the result.
5. Use `.stream()` to process streamed output.
6. Specify which streaming modes to use.

```pycon
('updates', {'add_one': 2})
('updates', {'add_two': 3})
('custom', 'hello')
('custom', 'world')
('updates', {'main': 5})
```

!!! important "Async with Python < 3.11"

    If using Python < 3.11 and writing async code, using `get_stream_writer()` will not work. Instead please
    use the `StreamWriter` class directly. See [Async with Python < 3.11](../how-tos/streaming.md#async) for more details.

    ```python
    from langgraph.types import StreamWriter

    @entrypoint(checkpointer=checkpointer)
    # highlight-next-line
    async def main(inputs: dict, writer: StreamWriter) -> int:
        ...
    ```

:::

:::js

```typescript
import {
  entrypoint,
  MemorySaver,
  LangGraphRunnableConfig,
} from "@langchain/langgraph";

const checkpointer = new MemorySaver();

const main = entrypoint(
  { checkpointer, name: "main" },
  async (
    inputs: { x: number },
    config: LangGraphRunnableConfig
  ): Promise<number> => {
    config.writer?.("Started processing"); // (1)!
    const result = inputs.x * 2;
    config.writer?.(`Result is ${result}`); // (2)!
    return result;
  }
);

const config = { configurable: { thread_id: "abc" } };

// (3)!
for await (const [mode, chunk] of await main.stream(
  { x: 5 },
  { streamMode: ["custom", "updates"], ...config } // (4)!
)) {
  console.log(`${mode}: ${JSON.stringify(chunk)}`);
}
```

1. Emit custom data before computation begins.
2. Emit another custom message after computing the result.
3. Use `.stream()` to process streamed output.
4. Specify which streaming modes to use.

```
updates: {"addOne": 2}
updates: {"addTwo": 3}
custom: "hello"
custom: "world"
updates: {"main": 5}
```

:::

## Retry policy

:::python

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import RetryPolicy

# This variable is just used for demonstration purposes to simulate a network failure.
# It's not something you will have in your actual code.
attempts = 0

# Let's configure the RetryPolicy to retry on ValueError.
# The default RetryPolicy is optimized for retrying specific network errors.
retry_policy = RetryPolicy(retry_on=ValueError)

@task(retry_policy=retry_policy)
def get_info():
    global attempts
    attempts += 1

    if attempts < 2:
        raise ValueError('Failure')
    return "OK"

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def main(inputs, writer):
    return get_info().result()

config = {
    "configurable": {
        "thread_id": "1"
    }
}

main.invoke({'any_input': 'foobar'}, config=config)
```

```pycon
'OK'
```

:::

:::js

```typescript
import {
  MemorySaver,
  entrypoint,
  task,
  RetryPolicy,
} from "@langchain/langgraph";

// This variable is just used for demonstration purposes to simulate a network failure.
// It's not something you will have in your actual code.
let attempts = 0;

// Let's configure the RetryPolicy to retry on ValueError.
// The default RetryPolicy is optimized for retrying specific network errors.
const retryPolicy: RetryPolicy = { retryOn: (error) => error instanceof Error };

const getInfo = task(
  {
    name: "getInfo",
    retry: retryPolicy,
  },
  () => {
    attempts += 1;

    if (attempts < 2) {
      throw new Error("Failure");
    }
    return "OK";
  }
);

const checkpointer = new MemorySaver();

const main = entrypoint(
  { checkpointer, name: "main" },
  async (inputs: Record<string, any>) => {
    return await getInfo();
  }
);

const config = {
  configurable: {
    thread_id: "1",
  },
};

await main.invoke({ any_input: "foobar" }, config);
```

```
'OK'
```

:::

## Caching Tasks

:::python

```python
import time
from langgraph.cache.memory import InMemoryCache
from langgraph.func import entrypoint, task
from langgraph.types import CachePolicy


@task(cache_policy=CachePolicy(ttl=120))  # (1)!
def slow_add(x: int) -> int:
    time.sleep(1)
    return x * 2


@entrypoint(cache=InMemoryCache())
def main(inputs: dict) -> dict[str, int]:
    result1 = slow_add(inputs["x"]).result()
    result2 = slow_add(inputs["x"]).result()
    return {"result1": result1, "result2": result2}


for chunk in main.stream({"x": 5}, stream_mode="updates"):
    print(chunk)

#> {'slow_add': 10}
#> {'slow_add': 10, '__metadata__': {'cached': True}}
#> {'main': {'result1': 10, 'result2': 10}}
```

1. `ttl` is specified in seconds. The cache will be invalidated after this time.
   :::

:::js

```typescript
import {
  InMemoryCache,
  entrypoint,
  task,
  CachePolicy,
} from "@langchain/langgraph";

const slowAdd = task(
  {
    name: "slowAdd",
    cache: { ttl: 120 }, // (1)!
  },
  async (x: number) => {
    await new Promise((resolve) => setTimeout(resolve, 1000));
    return x * 2;
  }
);

const main = entrypoint(
  { cache: new InMemoryCache(), name: "main" },
  async (inputs: { x: number }) => {
    const result1 = await slowAdd(inputs.x);
    const result2 = await slowAdd(inputs.x);
    return { result1, result2 };
  }
);

for await (const chunk of await main.stream(
  { x: 5 },
  { streamMode: "updates" }
)) {
  console.log(chunk);
}

//> { slowAdd: 10 }
//> { slowAdd: 10, '__metadata__': { cached: true } }
//> { main: { result1: 10, result2: 10 } }
```

1. `ttl` is specified in seconds. The cache will be invalidated after this time.
   :::

## Resuming after an error

:::python

```python
import time
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import StreamWriter

# This variable is just used for demonstration purposes to simulate a network failure.
# It's not something you will have in your actual code.
attempts = 0

@task()
def get_info():
    """
    Simulates a task that fails once before succeeding.
    Raises an exception on the first attempt, then returns "OK" on subsequent tries.
    """
    global attempts
    attempts += 1

    if attempts < 2:
        raise ValueError("Failure")  # Simulate a failure on the first attempt
    return "OK"

# Initialize an in-memory checkpointer for persistence
checkpointer = InMemorySaver()

@task
def slow_task():
    """
    Simulates a slow-running task by introducing a 1-second delay.
    """
    time.sleep(1)
    return "Ran slow task."

@entrypoint(checkpointer=checkpointer)
def main(inputs, writer: StreamWriter):
    """
    Main workflow function that runs the slow_task and get_info tasks sequentially.

    Parameters:
    - inputs: Dictionary containing workflow input values.
    - writer: StreamWriter for streaming custom data.

    The workflow first executes `slow_task` and then attempts to execute `get_info`,
    which will fail on the first invocation.
    """
    slow_task_result = slow_task().result()  # Blocking call to slow_task
    get_info().result()  # Exception will be raised here on the first attempt
    return slow_task_result

# Workflow execution configuration with a unique thread identifier
config = {
    "configurable": {
        "thread_id": "1"  # Unique identifier to track workflow execution
    }
}

# This invocation will take ~1 second due to the slow_task execution
try:
    # First invocation will raise an exception due to the `get_info` task failing
    main.invoke({'any_input': 'foobar'}, config=config)
except ValueError:
    pass  # Handle the failure gracefully
```

When we resume execution, we won't need to re-run the `slow_task` as its result is already saved in the checkpoint.

```python
main.invoke(None, config=config)
```

```pycon
'Ran slow task.'
```

:::

:::js

```typescript
import { entrypoint, task, MemorySaver } from "@langchain/langgraph";

// This variable is just used for demonstration purposes to simulate a network failure.
// It's not something you will have in your actual code.
let attempts = 0;

const getInfo = task("getInfo", async () => {
  /**
   * Simulates a task that fails once before succeeding.
   * Throws an exception on the first attempt, then returns "OK" on subsequent tries.
   */
  attempts += 1;

  if (attempts < 2) {
    throw new Error("Failure"); // Simulate a failure on the first attempt
  }
  return "OK";
});

// Initialize an in-memory checkpointer for persistence
const checkpointer = new MemorySaver();

const slowTask = task("slowTask", async () => {
  /**
   * Simulates a slow-running task by introducing a 1-second delay.
   */
  await new Promise((resolve) => setTimeout(resolve, 1000));
  return "Ran slow task.";
});

const main = entrypoint(
  { checkpointer, name: "main" },
  async (inputs: Record<string, any>) => {
    /**
     * Main workflow function that runs the slowTask and getInfo tasks sequentially.
     *
     * Parameters:
     * - inputs: Record<string, any> containing workflow input values.
     *
     * The workflow first executes `slowTask` and then attempts to execute `getInfo`,
     * which will fail on the first invocation.
     */
    const slowTaskResult = await slowTask(); // Blocking call to slowTask
    await getInfo(); // Exception will be raised here on the first attempt
    return slowTaskResult;
  }
);

// Workflow execution configuration with a unique thread identifier
const config = {
  configurable: {
    thread_id: "1", // Unique identifier to track workflow execution
  },
};

// This invocation will take ~1 second due to the slowTask execution
try {
  // First invocation will raise an exception due to the `getInfo` task failing
  await main.invoke({ any_input: "foobar" }, config);
} catch (err) {
  // Handle the failure gracefully
}
```

When we resume execution, we won't need to re-run the `slowTask` as its result is already saved in the checkpoint.

```typescript
await main.invoke(null, config);
```

```
'Ran slow task.'
```

:::

## Human-in-the-loop

The functional API supports [human-in-the-loop](../concepts/human_in_the_loop.md) workflows using the `interrupt` function and the `Command` primitive.

### Basic human-in-the-loop workflow

We will create three [tasks](../concepts/functional_api.md#task):

1. Append `"bar"`.
2. Pause for human input. When resuming, append human input.
3. Append `"qux"`.

:::python

```python
from langgraph.func import entrypoint, task
from langgraph.types import Command, interrupt


@task
def step_1(input_query):
    """Append bar."""
    return f"{input_query} bar"


@task
def human_feedback(input_query):
    """Append user input."""
    feedback = interrupt(f"Please provide feedback: {input_query}")
    return f"{input_query} {feedback}"


@task
def step_3(input_query):
    """Append qux."""
    return f"{input_query} qux"
```

:::

:::js

```typescript
import { entrypoint, task, interrupt, Command } from "@langchain/langgraph";

const step1 = task("step1", async (inputQuery: string) => {
  // Append bar
  return `${inputQuery} bar`;
});

const humanFeedback = task("humanFeedback", async (inputQuery: string) => {
  // Append user input
  const feedback = interrupt(`Please provide feedback: ${inputQuery}`);
  return `${inputQuery} ${feedback}`;
});

const step3 = task("step3", async (inputQuery: string) => {
  // Append qux
  return `${inputQuery} qux`;
});
```

:::

We can now compose these tasks in an [entrypoint](../concepts/functional_api.md#entrypoint):

:::python

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()


@entrypoint(checkpointer=checkpointer)
def graph(input_query):
    result_1 = step_1(input_query).result()
    result_2 = human_feedback(result_1).result()
    result_3 = step_3(result_2).result()

    return result_3
```

:::

:::js

```typescript
import { MemorySaver } from "@langchain/langgraph";

const checkpointer = new MemorySaver();

const graph = entrypoint(
  { checkpointer, name: "graph" },
  async (inputQuery: string) => {
    const result1 = await step1(inputQuery);
    const result2 = await humanFeedback(result1);
    const result3 = await step3(result2);

    return result3;
  }
);
```

:::

[interrupt()](../how-tos/human_in_the_loop/add-human-in-the-loop.md#pause-using-interrupt) is called inside a task, enabling a human to review and edit the output of the previous task. The results of prior tasks-- in this case `step_1`-- are persisted, so that they are not run again following the `interrupt`.

Let's send in a query string:

:::python

```python
config = {"configurable": {"thread_id": "1"}}

for event in graph.stream("foo", config):
    print(event)
    print("\n")
```

:::

:::js

```typescript
const config = { configurable: { thread_id: "1" } };

for await (const event of await graph.stream("foo", config)) {
  console.log(event);
  console.log("\n");
}
```

:::

Note that we've paused with an `interrupt` after `step_1`. The interrupt provides instructions to resume the run. To resume, we issue a [Command](../how-tos/human_in_the_loop/add-human-in-the-loop.md#resume-using-the-command-primitive) containing the data expected by the `human_feedback` task.

:::python

```python
# Continue execution
for event in graph.stream(Command(resume="baz"), config):
    print(event)
    print("\n")
```

:::

:::js

```typescript
// Continue execution
for await (const event of await graph.stream(
  new Command({ resume: "baz" }),
  config
)) {
  console.log(event);
  console.log("\n");
}
```

:::

After resuming, the run proceeds through the remaining step and terminates as expected.

### Review tool calls

To review tool calls before execution, we add a `review_tool_call` function that calls [`interrupt`](../how-tos/human_in_the_loop/add-human-in-the-loop.md#pause-using-interrupt). When this function is called, execution will be paused until we issue a command to resume it.

Given a tool call, our function will `interrupt` for human review. At that point we can either:

- Accept the tool call
- Revise the tool call and continue
- Generate a custom tool message (e.g., instructing the model to re-format its tool call)

:::python

```python
from typing import Union

def review_tool_call(tool_call: ToolCall) -> Union[ToolCall, ToolMessage]:
    """Review a tool call, returning a validated version."""
    human_review = interrupt(
        {
            "question": "Is this correct?",
            "tool_call": tool_call,
        }
    )
    review_action = human_review["action"]
    review_data = human_review.get("data")
    if review_action == "continue":
        return tool_call
    elif review_action == "update":
        updated_tool_call = {**tool_call, **{"args": review_data}}
        return updated_tool_call
    elif review_action == "feedback":
        return ToolMessage(
            content=review_data, name=tool_call["name"], tool_call_id=tool_call["id"]
        )
```

:::

:::js

```typescript
import { ToolCall } from "@langchain/core/messages/tool";
import { ToolMessage } from "@langchain/core/messages";

function reviewToolCall(toolCall: ToolCall): ToolCall | ToolMessage {
  // Review a tool call, returning a validated version
  const humanReview = interrupt({
    question: "Is this correct?",
    tool_call: toolCall,
  });

  const reviewAction = humanReview.action;
  const reviewData = humanReview.data;

  if (reviewAction === "continue") {
    return toolCall;
  } else if (reviewAction === "update") {
    const updatedToolCall = { ...toolCall, args: reviewData };
    return updatedToolCall;
  } else if (reviewAction === "feedback") {
    return new ToolMessage({
      content: reviewData,
      name: toolCall.name,
      tool_call_id: toolCall.id,
    });
  }

  throw new Error(`Unknown review action: ${reviewAction}`);
}
```

:::

We can now update our [entrypoint](../concepts/functional_api.md#entrypoint) to review the generated tool calls. If a tool call is accepted or revised, we execute in the same way as before. Otherwise, we just append the `ToolMessage` supplied by the human. The results of prior tasks — in this case the initial model call — are persisted, so that they are not run again following the `interrupt`.

:::python

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph.message import add_messages
from langgraph.types import Command, interrupt


checkpointer = InMemorySaver()


@entrypoint(checkpointer=checkpointer)
def agent(messages, previous):
    if previous is not None:
        messages = add_messages(previous, messages)

    llm_response = call_model(messages).result()
    while True:
        if not llm_response.tool_calls:
            break

        # Review tool calls
        tool_results = []
        tool_calls = []
        for i, tool_call in enumerate(llm_response.tool_calls):
            review = review_tool_call(tool_call)
            if isinstance(review, ToolMessage):
                tool_results.append(review)
            else:  # is a validated tool call
                tool_calls.append(review)
                if review != tool_call:
                    llm_response.tool_calls[i] = review  # update message

        # Execute remaining tool calls
        tool_result_futures = [call_tool(tool_call) for tool_call in tool_calls]
        remaining_tool_results = [fut.result() for fut in tool_result_futures]

        # Append to message list
        messages = add_messages(
            messages,
            [llm_response, *tool_results, *remaining_tool_results],
        )

        # Call model again
        llm_response = call_model(messages).result()

    # Generate final response
    messages = add_messages(messages, llm_response)
    return entrypoint.final(value=llm_response, save=messages)
```

:::

:::js

```typescript
import {
  MemorySaver,
  entrypoint,
  interrupt,
  Command,
  addMessages,
} from "@langchain/langgraph";
import { ToolMessage, AIMessage, BaseMessage } from "@langchain/core/messages";

const checkpointer = new MemorySaver();

const agent = entrypoint(
  { checkpointer, name: "agent" },
  async (
    messages: BaseMessage[],
    previous?: BaseMessage[]
  ): Promise<BaseMessage> => {
    if (previous !== undefined) {
      messages = addMessages(previous, messages);
    }

    let llmResponse = await callModel(messages);
    while (true) {
      if (!llmResponse.tool_calls?.length) {
        break;
      }

      // Review tool calls
      const toolResults: ToolMessage[] = [];
      const toolCalls: ToolCall[] = [];

      for (let i = 0; i < llmResponse.tool_calls.length; i++) {
        const review = reviewToolCall(llmResponse.tool_calls[i]);
        if (review instanceof ToolMessage) {
          toolResults.push(review);
        } else {
          // is a validated tool call
          toolCalls.push(review);
          if (review !== llmResponse.tool_calls[i]) {
            llmResponse.tool_calls[i] = review; // update message
          }
        }
      }

      // Execute remaining tool calls
      const remainingToolResults = await Promise.all(
        toolCalls.map((toolCall) => callTool(toolCall))
      );

      // Append to message list
      messages = addMessages(messages, [
        llmResponse,
        ...toolResults,
        ...remainingToolResults,
      ]);

      // Call model again
      llmResponse = await callModel(messages);
    }

    // Generate final response
    messages = addMessages(messages, llmResponse);
    return entrypoint.final({ value: llmResponse, save: messages });
  }
);
```

:::

## Short-term memory

Short-term memory allows storing information across different **invocations** of the same **thread id**. See [short-term memory](../concepts/functional_api.md#short-term-memory) for more details.

### Manage checkpoints

You can view and delete the information stored by the checkpointer.

#### View thread state (checkpoint)

:::python

```python
config = {
    "configurable": {
        # highlight-next-line
        "thread_id": "1",
        # optionally provide an ID for a specific checkpoint,
        # otherwise the latest checkpoint is shown
        # highlight-next-line
        # "checkpoint_id": "1f029ca3-1f5b-6704-8004-820c16b69a5a"

    }
}
# highlight-next-line
graph.get_state(config)
```

```
StateSnapshot(
    values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today?), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]}, next=(),
    config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
    metadata={
        'source': 'loop',
        'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}},
        'step': 4,
        'parents': {},
        'thread_id': '1'
    },
    created_at='2025-05-05T16:01:24.680462+00:00',
    parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
    tasks=(),
    interrupts=()
)
```

:::

:::js

```typescript
const config = {
  configurable: {
    // highlight-next-line
    thread_id: "1",
    // optionally provide an ID for a specific checkpoint,
    // otherwise the latest checkpoint is shown
    // highlight-next-line
    // checkpoint_id: "1f029ca3-1f5b-6704-8004-820c16b69a5a"
  },
};
// highlight-next-line
await graph.getState(config);
```

```
StateSnapshot {
  values: {
    messages: [
      HumanMessage { content: "hi! I'm bob" },
      AIMessage { content: "Hi Bob! How are you doing today?" },
      HumanMessage { content: "what's my name?" },
      AIMessage { content: "Your name is Bob." }
    ]
  },
  next: [],
  config: { configurable: { thread_id: '1', checkpoint_ns: '', checkpoint_id: '1f029ca3-1f5b-6704-8004-820c16b69a5a' } },
  metadata: {
    source: 'loop',
    writes: { call_model: { messages: AIMessage { content: "Your name is Bob." } } },
    step: 4,
    parents: {},
    thread_id: '1'
  },
  createdAt: '2025-05-05T16:01:24.680462+00:00',
  parentConfig: { configurable: { thread_id: '1', checkpoint_ns: '', checkpoint_id: '1f029ca3-1790-6b0a-8003-baf965b6a38f' } },
  tasks: [],
  interrupts: []
}
```

:::

#### View the history of the thread (checkpoints)

:::python

```python
config = {
    "configurable": {
        # highlight-next-line
        "thread_id": "1"
    }
}
# highlight-next-line
list(graph.get_state_history(config))
```

```
[
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
        next=(),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
        metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}}, 'step': 4, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:24.680462+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
        tasks=(),
        interrupts=()
    ),
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?")]},
        next=('call_model',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
        metadata={'source': 'loop', 'writes': None, 'step': 3, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:23.863421+00:00',
        parent_config={...}
        tasks=(PregelTask(id='8ab4155e-6b15-b885-9ce5-bed69a2c305c', name='call_model', path=('__pregel_pull', 'call_model'), error=None, interrupts=(), state=None, result={'messages': AIMessage(content='Your name is Bob.')}),),
        interrupts=()
    ),
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]},
        next=('__start__',),
        config={...},
        metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "what's my name?"}]}}, 'step': 2, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:23.863173+00:00',
        parent_config={...}
        tasks=(PregelTask(id='24ba39d6-6db1-4c9b-f4c5-682aeaf38dcd', name='__start__', path=('__pregel_pull', '__start__'), error=None, interrupts=(), state=None, result={'messages': [{'role': 'user', 'content': "what's my name?"}]}),),
        interrupts=()
    ),
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]},
        next=(),
        config={...},
        metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}}, 'step': 1, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:23.862295+00:00',
        parent_config={...}
        tasks=(),
        interrupts=()
    ),
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob")]},
        next=('call_model',),
        config={...},
        metadata={'source': 'loop', 'writes': None, 'step': 0, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:22.278960+00:00',
        parent_config={...}
        tasks=(PregelTask(id='8cbd75e0-3720-b056-04f7-71ac805140a0', name='call_model', path=('__pregel_pull', 'call_model'), error=None, interrupts=(), state=None, result={'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}),),
        interrupts=()
    ),
    StateSnapshot(
        values={'messages': []},
        next=('__start__',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-0870-6ce2-bfff-1f3f14c3e565'}},
        metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}}, 'step': -1, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:22.277497+00:00',
        parent_config=None,
        tasks=(PregelTask(id='d458367b-8265-812c-18e2-33001d199ce6', name='__start__', path=('__pregel_pull', '__start__'), error=None, interrupts=(), state=None, result={'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}),),
        interrupts=()
    )
]
```

:::

:::js

```typescript
const config = {
  configurable: {
    // highlight-next-line
    thread_id: "1",
  },
};
// highlight-next-line
const history = [];
for await (const state of graph.getStateHistory(config)) {
  history.push(state);
}
```

```
[
  StateSnapshot {
    values: {
      messages: [
        HumanMessage { content: "hi! I'm bob" },
        AIMessage { content: "Hi Bob! How are you doing today? Is there anything I can help you with?" },
        HumanMessage { content: "what's my name?" },
        AIMessage { content: "Your name is Bob." }
      ]
    },
    next: [],
    config: { configurable: { thread_id: '1', checkpoint_ns: '', checkpoint_id: '1f029ca3-1f5b-6704-8004-820c16b69a5a' } },
    metadata: { source: 'loop', writes: { call_model: { messages: AIMessage { content: "Your name is Bob." } } }, step: 4, parents: {}, thread_id: '1' },
    createdAt: '2025-05-05T16:01:24.680462+00:00',
    parentConfig: { configurable: { thread_id: '1', checkpoint_ns: '', checkpoint_id: '1f029ca3-1790-6b0a-8003-baf965b6a38f' } },
    tasks: [],
    interrupts: []
  },
  // ... more state snapshots
]
```

:::

### Decouple return value from saved value

Use `entrypoint.final` to decouple what is returned to the caller from what is persisted in the checkpoint. This is useful when:

- You want to return a computed result (e.g., a summary or status), but save a different internal value for use on the next invocation.
- You need to control what gets passed to the previous parameter on the next run.

:::python

```python
from typing import Optional
from langgraph.func import entrypoint
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def accumulate(n: int, *, previous: Optional[int]) -> entrypoint.final[int, int]:
    previous = previous or 0
    total = previous + n
    # Return the *previous* value to the caller but save the *new* total to the checkpoint.
    return entrypoint.final(value=previous, save=total)

config = {"configurable": {"thread_id": "my-thread"}}

print(accumulate.invoke(1, config=config))  # 0
print(accumulate.invoke(2, config=config))  # 1
print(accumulate.invoke(3, config=config))  # 3
```

:::

:::js

```typescript
import { entrypoint, MemorySaver } from "@langchain/langgraph";

const checkpointer = new MemorySaver();

const accumulate = entrypoint(
  { checkpointer, name: "accumulate" },
  async (n: number, previous?: number) => {
    const prev = previous || 0;
    const total = prev + n;
    // Return the *previous* value to the caller but save the *new* total to the checkpoint.
    return entrypoint.final({ value: prev, save: total });
  }
);

const config = { configurable: { thread_id: "my-thread" } };

console.log(await accumulate.invoke(1, config)); // 0
console.log(await accumulate.invoke(2, config)); // 1
console.log(await accumulate.invoke(3, config)); // 3
```

:::

### Chatbot example

An example of a simple chatbot using the functional API and the `InMemorySaver` checkpointer.
The bot is able to remember the previous conversation and continue from where it left off.

:::python

```python
from langchain_core.messages import BaseMessage
from langgraph.graph import add_messages
from langgraph.func import entrypoint, task
from langgraph.checkpoint.memory import InMemorySaver
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(model="claude-3-5-sonnet-latest")

@task
def call_model(messages: list[BaseMessage]):
    response = model.invoke(messages)
    return response

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def workflow(inputs: list[BaseMessage], *, previous: list[BaseMessage]):
    if previous:
        inputs = add_messages(previous, inputs)

    response = call_model(inputs).result()
    return entrypoint.final(value=response, save=add_messages(inputs, response))

config = {"configurable": {"thread_id": "1"}}
input_message = {"role": "user", "content": "hi! I'm bob"}
for chunk in workflow.stream([input_message], config, stream_mode="values"):
    chunk.pretty_print()

input_message = {"role": "user", "content": "what's my name?"}
for chunk in workflow.stream([input_message], config, stream_mode="values"):
    chunk.pretty_print()
```

:::

:::js

```typescript
import { BaseMessage } from "@langchain/core/messages";
import {
  addMessages,
  entrypoint,
  task,
  MemorySaver,
} from "@langchain/langgraph";
import { ChatAnthropic } from "@langchain/anthropic";

const model = new ChatAnthropic({ model: "claude-3-5-sonnet-latest" });

const callModel = task(
  "callModel",
  async (messages: BaseMessage[]): Promise<BaseMessage> => {
    const response = await model.invoke(messages);
    return response;
  }
);

const checkpointer = new MemorySaver();

const workflow = entrypoint(
  { checkpointer, name: "workflow" },
  async (
    inputs: BaseMessage[],
    previous?: BaseMessage[]
  ): Promise<BaseMessage> => {
    let messages = inputs;
    if (previous) {
      messages = addMessages(previous, inputs);
    }

    const response = await callModel(messages);
    return entrypoint.final({
      value: response,
      save: addMessages(messages, response),
    });
  }
);

const config = { configurable: { thread_id: "1" } };
const inputMessage = { role: "user", content: "hi! I'm bob" };

for await (const chunk of await workflow.stream([inputMessage], {
  ...config,
  streamMode: "values",
})) {
  console.log(chunk.content);
}

const inputMessage2 = { role: "user", content: "what's my name?" };
for await (const chunk of await workflow.stream([inputMessage2], {
  ...config,
  streamMode: "values",
})) {
  console.log(chunk.content);
}
```

:::

??? example "Extended example: build a simple chatbot"

     [How to add thread-level persistence (functional API)](./persistence-functional.ipynb): Shows how to add thread-level persistence to a functional API workflow and implements a simple chatbot.

## Long-term memory

[long-term memory](../concepts/memory.md#long-term-memory) allows storing information across different **thread ids**. This could be useful for learning information about a given user in one conversation and using it in another.

??? example "Extended example: add long-term memory"

    [How to add cross-thread persistence (functional API)](./cross-thread-persistence-functional.ipynb): Shows how to add cross-thread persistence to a functional API workflow and implements a simple chatbot.

## Workflows

- [Workflows and agent](../tutorials/workflows.md) guide for more examples of how to build workflows using the Functional API.

## Agents

- [How to create an agent from scratch (Functional API)](./react-agent-from-scratch-functional.ipynb): Shows how to create a simple agent from scratch using the functional API.
- [How to build a multi-agent network](./multi-agent-network-functional.ipynb): Shows how to build a multi-agent network using the functional API.
- [How to add multi-turn conversation in a multi-agent application (functional API)](./multi-agent-multi-turn-convo-functional.ipynb): allow an end-user to engage in a multi-turn conversation with one or more agents.

## Integrate with other libraries

- [Add LangGraph's features to other frameworks using the functional API](./autogen-integration-functional.ipynb): Add LangGraph features like persistence, memory and streaming to other agent frameworks that do not provide them out of the box.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/use-remote-graph.md
```md
# How to interact with the deployment using RemoteGraph

!!! info "Prerequisites"

    - [LangGraph Platform](../concepts/langgraph_platform.md)
    - [LangGraph Server](../concepts/langgraph_server.md)

`RemoteGraph` is an interface that allows you to interact with your LangGraph Platform deployment as if it were a regular, locally-defined LangGraph graph (e.g. a `CompiledGraph`). This guide shows you how you can initialize a `RemoteGraph` and interact with it.

## Initializing the graph

:::python

When initializing a `RemoteGraph`, you must always specify:

- `name`: the name of the graph you want to interact with. This is the same graph name you use in `langgraph.json` configuration file for your deployment.
- `api_key`: a valid LangSmith API key. Can be set as an environment variable (`LANGSMITH_API_KEY`) or passed directly via the `api_key` argument. The API key could also be provided via the `client` / `sync_client` arguments, if `LangGraphClient` / `SyncLangGraphClient` were initialized with `api_key` argument.

Additionally, you have to provide one of the following:

- `url`: URL of the deployment you want to interact with. If you pass `url` argument, both sync and async clients will be created using the provided URL, headers (if provided) and default configuration values (e.g. timeout, etc).
- `client`: a `LangGraphClient` instance for interacting with the deployment asynchronously (e.g. using `.astream()`, `.ainvoke()`, `.aget_state()`, `.aupdate_state()`, etc.)
- `sync_client`: a `SyncLangGraphClient` instance for interacting with the deployment synchronously (e.g. using `.stream()`, `.invoke()`, `.get_state()`, `.update_state()`, etc.)

!!! Note

    If you pass both `client` or `sync_client` as well as `url` argument, they will take precedence over the `url` argument. If none of the `client` / `sync_client` / `url` arguments are provided, `RemoteGraph` will raise a `ValueError` at runtime.

:::

:::js

When initializing a `RemoteGraph`, you must always specify:

- `name`: the name of the graph you want to interact with. This is the same graph name you use in `langgraph.json` configuration file for your deployment.
- `apiKey`: a valid LangSmith API key. Can be set as an environment variable (`LANGSMITH_API_KEY`) or passed directly via the `apiKey` argument. The API key could also be provided via the `client`if `LangGraphClient` were initialized with `apiKey` argument.

Additionally, you have to provide one of the following:

- `url`: URL of the deployment you want to interact with. If you pass `url` argument, both sync and async clients will be created using the provided URL, headers (if provided) and default configuration values (e.g. timeout, etc).
- `client`: a `LangGraphClient` instance for interacting with the deployment asynchronously

:::

### Using URL

:::python

```python
from langgraph.pregel.remote import RemoteGraph

url = <DEPLOYMENT_URL>
graph_name = "agent"
remote_graph = RemoteGraph(graph_name, url=url)
```

:::

:::js

```ts
import { RemoteGraph } from "@langchain/langgraph/remote";

const url = `<DEPLOYMENT_URL>`;
const graphName = "agent";
const remoteGraph = new RemoteGraph({ graphId: graphName, url });
```

:::

### Using clients

:::python

```python
from langgraph_sdk import get_client, get_sync_client
from langgraph.pregel.remote import RemoteGraph

url = <DEPLOYMENT_URL>
graph_name = "agent"
client = get_client(url=url)
sync_client = get_sync_client(url=url)
remote_graph = RemoteGraph(graph_name, client=client, sync_client=sync_client)
```

:::

:::js

```ts
import { Client } from "@langchain/langgraph-sdk";
import { RemoteGraph } from "@langchain/langgraph/remote";

const client = new Client({ apiUrl: `<DEPLOYMENT_URL>` });
const graphName = "agent";
const remoteGraph = new RemoteGraph({ graphId: graphName, client });
```

:::

## Invoking the graph

:::python
Since `RemoteGraph` is a `Runnable` that implements the same methods as `CompiledGraph`, you can interact with it the same way you normally would with a compiled graph, i.e. by calling `.invoke()`, `.stream()`, `.get_state()`, `.update_state()`, etc (as well as their async counterparts).

### Asynchronously

!!! Note

    To use the graph asynchronously, you must provide either the `url` or `client` when initializing the `RemoteGraph`.

```python
# invoke the graph
result = await remote_graph.ainvoke({
    "messages": [{"role": "user", "content": "what's the weather in sf"}]
})

# stream outputs from the graph
async for chunk in remote_graph.astream({
    "messages": [{"role": "user", "content": "what's the weather in la"}]
}):
    print(chunk)
```

### Synchronously

!!! Note

    To use the graph synchronously, you must provide either the `url` or `sync_client` when initializing the `RemoteGraph`.

```python
# invoke the graph
result = remote_graph.invoke({
    "messages": [{"role": "user", "content": "what's the weather in sf"}]
})

# stream outputs from the graph
for chunk in remote_graph.stream({
    "messages": [{"role": "user", "content": "what's the weather in la"}]
}):
    print(chunk)
```

:::

:::js
Since `RemoteGraph` is a `Runnable` that implements the same methods as `CompiledGraph`, you can interact with it the same way you normally would with a compiled graph, i.e. by calling `.invoke()`, `.stream()`, `.getState()`, `.updateState()`, etc.

```ts
// invoke the graph
const result = await remoteGraph.invoke({
    messages: [{role: "user", content: "what's the weather in sf"}]
})

// stream outputs from the graph
for await (const chunk of await remoteGraph.stream({
    messages: [{role: "user", content: "what's the weather in la"}]
})):
    console.log(chunk)
```

:::

## Thread-level persistence

By default, the graph runs (i.e. `.invoke()` or `.stream()` invocations) are stateless - the checkpoints and the final state of the graph are not persisted. If you would like to persist the outputs of the graph run (for example, to enable human-in-the-loop features), you can create a thread and provide the thread ID via the `config` argument, same as you would with a regular compiled graph:

:::python

```python
from langgraph_sdk import get_sync_client
url = <DEPLOYMENT_URL>
graph_name = "agent"
sync_client = get_sync_client(url=url)
remote_graph = RemoteGraph(graph_name, url=url)

# create a thread (or use an existing thread instead)
thread = sync_client.threads.create()

# invoke the graph with the thread config
config = {"configurable": {"thread_id": thread["thread_id"]}}
result = remote_graph.invoke({
    "messages": [{"role": "user", "content": "what's the weather in sf"}]
}, config=config)

# verify that the state was persisted to the thread
thread_state = remote_graph.get_state(config)
print(thread_state)
```

:::

:::js

```ts
import { Client } from "@langchain/langgraph-sdk";
import { RemoteGraph } from "@langchain/langgraph/remote";

const url = `<DEPLOYMENT_URL>`;
const graphName = "agent";
const client = new Client({ apiUrl: url });
const remoteGraph = new RemoteGraph({ graphId: graphName, url });

// create a thread (or use an existing thread instead)
const thread = await client.threads.create();

// invoke the graph with the thread config
const config = { configurable: { thread_id: thread.thread_id } };
const result = await remoteGraph.invoke(
  {
    messages: [{ role: "user", content: "what's the weather in sf" }],
  },
  config
);

// verify that the state was persisted to the thread
const threadState = await remoteGraph.getState(config);
console.log(threadState);
```

:::

## Using as a subgraph

!!! Note

    If you need to use a `checkpointer` with a graph that has a `RemoteGraph` subgraph node, make sure to use UUIDs as thread IDs.

Since the `RemoteGraph` behaves the same way as a regular `CompiledGraph`, it can be also used as a subgraph in another graph. For example:

:::python

```python
from langgraph_sdk import get_sync_client
from langgraph.graph import StateGraph, MessagesState, START
from typing import TypedDict

url = <DEPLOYMENT_URL>
graph_name = "agent"
remote_graph = RemoteGraph(graph_name, url=url)

# define parent graph
builder = StateGraph(MessagesState)
# add remote graph directly as a node
builder.add_node("child", remote_graph)
builder.add_edge(START, "child")
graph = builder.compile()

# invoke the parent graph
result = graph.invoke({
    "messages": [{"role": "user", "content": "what's the weather in sf"}]
})
print(result)

# stream outputs from both the parent graph and subgraph
for chunk in graph.stream({
    "messages": [{"role": "user", "content": "what's the weather in sf"}]
}, subgraphs=True):
    print(chunk)
```

:::

:::js

```ts
import { MessagesAnnotation, StateGraph, START } from "@langchain/langgraph";
import { RemoteGraph } from "@langchain/langgraph/remote";

const url = `<DEPLOYMENT_URL>`;
const graphName = "agent";
const remoteGraph = new RemoteGraph({ graphId: graphName, url });

// define parent graph and add remote graph directly as a node
const graph = new StateGraph(MessagesAnnotation)
  .addNode("child", remoteGraph)
  .addEdge(START, "child")
  .compile();

// invoke the parent graph
const result = await graph.invoke({
  messages: [{ role: "user", content: "what's the weather in sf" }],
});
console.log(result);

// stream outputs from both the parent graph and subgraph
for await (const chunk of await graph.stream(
  {
    messages: [{ role: "user", content: "what's the weather in la" }],
  },
  { subgraphs: true }
)) {
  console.log(chunk);
}
```

:::

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/human_in_the_loop/add-human-in-the-loop.md
```md
---
search:
  boost: 2
tags:
  - human-in-the-loop
  - hil
  - interrupt
hide:
  - tags
---

# Enable human intervention

To review, edit, and approve tool calls in an agent or workflow, use interrupts to pause a graph and wait for human input. Interrupts use LangGraph's [persistence](../../concepts/persistence.md) layer, which saves the graph state, to indefinitely pause graph execution until you resume.

!!! info

    For more information about human-in-the-loop workflows, see the [Human-in-the-Loop](../../concepts/human_in_the_loop.md) conceptual guide.

## Pause using `interrupt`

:::python
[Dynamic interrupts](../../concepts/human_in_the_loop.md#key-capabilities) (also known as dynamic breakpoints) are triggered based on the current state of the graph. You can set dynamic interrupts by calling @[`interrupt` function][interrupt] in the appropriate place. The graph will pause, which allows for human intervention, and then resumes the graph with their input. It's useful for tasks like approvals, edits, or gathering additional context.

!!! note

    As of v1.0, `interrupt` is the recommended way to pause a graph. `NodeInterrupt` is deprecated and will be removed in v2.0.

:::

:::js
[Dynamic interrupts](../../concepts/human_in_the_loop.md#key-capabilities) (also known as dynamic breakpoints) are triggered based on the current state of the graph. You can set dynamic interrupts by calling @[`interrupt` function][interrupt] in the appropriate place. The graph will pause, which allows for human intervention, and then resumes the graph with their input. It's useful for tasks like approvals, edits, or gathering additional context.
:::

To use `interrupt` in your graph, you need to:

1. [**Specify a checkpointer**](../../concepts/persistence.md#checkpoints) to save the graph state after each step.
2. **Call `interrupt()`** in the appropriate place. See the [Common Patterns](#common-patterns) section for examples.
3. **Run the graph** with a [**thread ID**](../../concepts/persistence.md#threads) until the `interrupt` is hit.
4. **Resume execution** using `invoke`/`stream` (see [**The `Command` primitive**](#resume-using-the-command-primitive)).

:::python

```python
# highlight-next-line
from langgraph.types import interrupt, Command

def human_node(state: State):
    # highlight-next-line
    value = interrupt( # (1)!
        {
            "text_to_revise": state["some_text"] # (2)!
        }
    )
    return {
        "some_text": value # (3)!
    }


graph = graph_builder.compile(checkpointer=checkpointer) # (4)!

# Run the graph until the interrupt is hit.
config = {"configurable": {"thread_id": "some_id"}}
result = graph.invoke({"some_text": "original text"}, config=config) # (5)!
print(result['__interrupt__']) # (6)!
# > [
# >    Interrupt(
# >       value={'text_to_revise': 'original text'},
# >       resumable=True,
# >       ns=['human_node:6ce9e64f-edef-fe5d-f7dc-511fa9526960']
# >    )
# > ]

# highlight-next-line
print(graph.invoke(Command(resume="Edited text"), config=config)) # (7)!
# > {'some_text': 'Edited text'}
```

1. `interrupt(...)` pauses execution at `human_node`, surfacing the given payload to a human.
2. Any JSON serializable value can be passed to the `interrupt` function. Here, a dict containing the text to revise.
3. Once resumed, the return value of `interrupt(...)` is the human-provided input, which is used to update the state.
4. A checkpointer is required to persist graph state. In production, this should be durable (e.g., backed by a database).
5. The graph is invoked with some initial state.
6. When the graph hits the interrupt, it returns an `Interrupt` object with the payload and metadata.
7. The graph is resumed with a `Command(resume=...)`, injecting the human's input and continuing execution.
   :::

:::js

```typescript
// highlight-next-line
import { interrupt, Command } from "@langchain/langgraph";

const graph = graphBuilder
  .addNode("humanNode", (state) => {
    // highlight-next-line
    const value = interrupt(
      // (1)!
      {
        textToRevise: state.someText, // (2)!
      }
    );
    return {
      someText: value, // (3)!
    };
  })
  .addEdge(START, "humanNode")
  .compile({ checkpointer }); // (4)!

// Run the graph until the interrupt is hit.
const config = { configurable: { thread_id: "some_id" } };
const result = await graph.invoke({ someText: "original text" }, config); // (5)!
console.log(result.__interrupt__); // (6)!
// > [
// >   {
// >     value: { textToRevise: 'original text' },
// >     resumable: true,
// >     ns: ['humanNode:6ce9e64f-edef-fe5d-f7dc-511fa9526960'],
// >     when: 'during'
// >   }
// > ]

// highlight-next-line
console.log(await graph.invoke(new Command({ resume: "Edited text" }), config)); // (7)!
// > { someText: 'Edited text' }
```

1. `interrupt(...)` pauses execution at `humanNode`, surfacing the given payload to a human.
2. Any JSON serializable value can be passed to the `interrupt` function. Here, an object containing the text to revise.
3. Once resumed, the return value of `interrupt(...)` is the human-provided input, which is used to update the state.
4. A checkpointer is required to persist graph state. In production, this should be durable (e.g., backed by a database).
5. The graph is invoked with some initial state.
6. When the graph hits the interrupt, it returns an object with `__interrupt__` containing the payload and metadata.
7. The graph is resumed with a `Command({ resume: ... })`, injecting the human's input and continuing execution.
   :::

??? example "Extended example: using `interrupt`"

    :::python
    ```python
    from typing import TypedDict
    import uuid
    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.constants import START
    from langgraph.graph import StateGraph

    # highlight-next-line
    from langgraph.types import interrupt, Command


    class State(TypedDict):
        some_text: str


    def human_node(state: State):
        # highlight-next-line
        value = interrupt(  # (1)!
            {
                "text_to_revise": state["some_text"]  # (2)!
            }
        )
        return {
            "some_text": value  # (3)!
        }


    # Build the graph
    graph_builder = StateGraph(State)
    graph_builder.add_node("human_node", human_node)
    graph_builder.add_edge(START, "human_node")
    checkpointer = InMemorySaver()  # (4)!
    graph = graph_builder.compile(checkpointer=checkpointer)
    # Pass a thread ID to the graph to run it.
    config = {"configurable": {"thread_id": uuid.uuid4()}}
    # Run the graph until the interrupt is hit.
    result = graph.invoke({"some_text": "original text"}, config=config)  # (5)!

    print(result['__interrupt__']) # (6)!
    # > [
    # >    Interrupt(
    # >       value={'text_to_revise': 'original text'},
    # >       resumable=True,
    # >       ns=['human_node:6ce9e64f-edef-fe5d-f7dc-511fa9526960']
    # >    )
    # > ]
    print(result["__interrupt__"])  # (6)!
    # > [Interrupt(value={'text_to_revise': 'original text'}, id='6d7c4048049254c83195429a3659661d')]

    # highlight-next-line
    print(graph.invoke(Command(resume="Edited text"), config=config)) # (7)!
    # > {'some_text': 'Edited text'}
    ```

    1. `interrupt(...)` pauses execution at `human_node`, surfacing the given payload to a human.
    2. Any JSON serializable value can be passed to the `interrupt` function. Here, a dict containing the text to revise.
    3. Once resumed, the return value of `interrupt(...)` is the human-provided input, which is used to update the state.
    4. A checkpointer is required to persist graph state. In production, this should be durable (e.g., backed by a database).
    5. The graph is invoked with some initial state.
    6. When the graph hits the interrupt, it returns an `Interrupt` object with the payload and metadata.
    7. The graph is resumed with a `Command(resume=...)`, injecting the human's input and continuing execution.
    :::

    :::js
    ```typescript
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";
    import { MemorySaver, StateGraph, START, interrupt, Command } from "@langchain/langgraph";

    const StateAnnotation = z.object({
      someText: z.string(),
    });

    // Build the graph
    const graphBuilder = new StateGraph(StateAnnotation)
      .addNode("humanNode", (state) => {
        // highlight-next-line
        const value = interrupt( // (1)!
          {
            textToRevise: state.someText // (2)!
          }
        );
        return {
          someText: value // (3)!
        };
      })
      .addEdge(START, "humanNode");

    const checkpointer = new MemorySaver(); // (4)!

    const graph = graphBuilder.compile({ checkpointer });

    // Pass a thread ID to the graph to run it.
    const config = { configurable: { thread_id: uuidv4() } };

    // Run the graph until the interrupt is hit.
    const result = await graph.invoke({ someText: "original text" }, config); // (5)!

    console.log(result.__interrupt__); // (6)!
    // > [
    // >   {
    // >     value: { textToRevise: 'original text' },
    // >     resumable: true,
    // >     ns: ['humanNode:6ce9e64f-edef-fe5d-f7dc-511fa9526960'],
    // >     when: 'during'
    // >   }
    // > ]

    // highlight-next-line
    console.log(await graph.invoke(new Command({ resume: "Edited text" }), config)); // (7)!
    // > { someText: 'Edited text' }
    ```

    1. `interrupt(...)` pauses execution at `humanNode`, surfacing the given payload to a human.
    2. Any JSON serializable value can be passed to the `interrupt` function. Here, an object containing the text to revise.
    3. Once resumed, the return value of `interrupt(...)` is the human-provided input, which is used to update the state.
    4. A checkpointer is required to persist graph state. In production, this should be durable (e.g., backed by a database).
    5. The graph is invoked with some initial state.
    6. When the graph hits the interrupt, it returns an object with `__interrupt__` containing the payload and metadata.
    7. The graph is resumed with a `Command({ resume: ... })`, injecting the human's input and continuing execution.
    :::

!!! tip "New in 0.4.0"

    :::python
    `__interrupt__` is a special key that will be returned when running the graph if the graph is interrupted. Support for `__interrupt__` in `invoke` and `ainvoke` has been added in version 0.4.0. If you're on an older version, you will only see `__interrupt__` in the result if you use `stream` or `astream`. You can also use `graph.get_state(thread_id)` to get the interrupt value(s).
    :::

    :::js
    `__interrupt__` is a special key that will be returned when running the graph if the graph is interrupted. Support for `__interrupt__` in `invoke` has been added in version 0.4.0. If you're on an older version, you will only see `__interrupt__` in the result if you use `stream`. You can also use `graph.getState(config)` to get the interrupt value(s).
    :::

!!! warning

    :::python
    Interrupts resemble Python's input() function in terms of developer experience, but they do not automatically resume execution from the interruption point. Instead, they rerun the entire node where the interrupt was used. For this reason, interrupts are typically best placed at the start of a node or in a dedicated node.
    :::

    :::js
    Interrupts are both powerful and ergonomic, but it's important to note that they do not automatically resume execution from the interrupt point. Instead, they rerun the entire where the interrupt was used. For this reason, interrupts are typically best placed at the state of a node or in a dedicated node.
    :::

## Resume using the `Command` primitive

:::python
!!! warning

    Resuming from an `interrupt` is different from Python's `input()` function, where execution resumes from the exact point where the `input()` function was called.

:::

When the `interrupt` function is used within a graph, execution pauses at that point and awaits user input.

:::python
To resume execution, use the @[`Command`][Command] primitive, which can be supplied via the `invoke` or `stream` methods. The graph resumes execution from the beginning of the node where `interrupt(...)` was initially called. This time, the `interrupt` function will return the value provided in `Command(resume=value)` rather than pausing again. All code from the beginning of the node to the `interrupt` will be re-executed.

```python
# Resume graph execution by providing the user's input.
graph.invoke(Command(resume={"age": "25"}), thread_config)
```

:::

:::js
To resume execution, use the @[`Command`][Command] primitive, which can be supplied via the `invoke` or `stream` methods. The graph resumes execution from the beginning of the node where `interrupt(...)` was initially called. This time, the `interrupt` function will return the value provided in `Command(resume=value)` rather than pausing again. All code from the beginning of the node to the `interrupt` will be re-executed.

```typescript
// Resume graph execution by providing the user's input.
await graph.invoke(new Command({ resume: { age: "25" } }), threadConfig);
```

:::

### Resume multiple interrupts with one invocation

When nodes with interrupt conditions are run in parallel, it's possible to have multiple interrupts in the task queue.
For example, the following graph has two nodes run in parallel that require human input:

<figure markdown="1">
![image](../assets/human_in_loop_parallel.png){: style="max-height:400px"}
</figure>

:::python
Once your graph has been interrupted and is stalled, you can resume all the interrupts at once with `Command.resume`, passing a dictionary mapping of interrupt ids to resume values.

```python
from typing import TypedDict
import uuid
from langchain_core.runnables import RunnableConfig
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.constants import START
from langgraph.graph import StateGraph
from langgraph.types import interrupt, Command


class State(TypedDict):
    text_1: str
    text_2: str


def human_node_1(state: State):
    value = interrupt({"text_to_revise": state["text_1"]})
    return {"text_1": value}


def human_node_2(state: State):
    value = interrupt({"text_to_revise": state["text_2"]})
    return {"text_2": value}


graph_builder = StateGraph(State)
graph_builder.add_node("human_node_1", human_node_1)
graph_builder.add_node("human_node_2", human_node_2)

# Add both nodes in parallel from START
graph_builder.add_edge(START, "human_node_1")
graph_builder.add_edge(START, "human_node_2")

checkpointer = InMemorySaver()
graph = graph_builder.compile(checkpointer=checkpointer)

thread_id = str(uuid.uuid4())
config: RunnableConfig = {"configurable": {"thread_id": thread_id}}
result = graph.invoke(
    {"text_1": "original text 1", "text_2": "original text 2"}, config=config
)

# Resume with mapping of interrupt IDs to values
resume_map = {
    i.id: f"edited text for {i.value['text_to_revise']}"
    for i in graph.get_state(config).interrupts
}
print(graph.invoke(Command(resume=resume_map), config=config))
# > {'text_1': 'edited text for original text 1', 'text_2': 'edited text for original text 2'}
```

:::

:::js

```typescript
const state = await parentGraph.getState(threadConfig);
const resumeMap = Object.fromEntries(
  state.interrupts.map((i) => [
    i.interruptId,
    `human input for prompt ${i.value}`,
  ])
);

await parentGraph.invoke(new Command({ resume: resumeMap }), threadConfig);
```

:::

## Common patterns

Below we show different design patterns that can be implemented using `interrupt` and `Command`.

### Approve or reject

<figure markdown="1">
![image](../../concepts/img/human_in_the_loop/approve-or-reject.png){: style="max-height:400px"}
<figcaption>Depending on the human's approval or rejection, the graph can proceed with the action or take an alternative path.</figcaption>
</figure>

Pause the graph before a critical step, such as an API call, to review and approve the action. If the action is rejected, you can prevent the graph from executing the step, and potentially take an alternative action.

:::python

```python
from typing import Literal
from langgraph.types import interrupt, Command

def human_approval(state: State) -> Command[Literal["some_node", "another_node"]]:
    is_approved = interrupt(
        {
            "question": "Is this correct?",
            # Surface the output that should be
            # reviewed and approved by the human.
            "llm_output": state["llm_output"]
        }
    )

    if is_approved:
        return Command(goto="some_node")
    else:
        return Command(goto="another_node")

# Add the node to the graph in an appropriate location
# and connect it to the relevant nodes.
graph_builder.add_node("human_approval", human_approval)
graph = graph_builder.compile(checkpointer=checkpointer)

# After running the graph and hitting the interrupt, the graph will pause.
# Resume it with either an approval or rejection.
thread_config = {"configurable": {"thread_id": "some_id"}}
graph.invoke(Command(resume=True), config=thread_config)
```

:::

:::js

```typescript
import { interrupt, Command } from "@langchain/langgraph";

// Add the node to the graph in an appropriate location
// and connect it to the relevant nodes.
graphBuilder.addNode("humanApproval", (state) => {
  const isApproved = interrupt({
    question: "Is this correct?",
    // Surface the output that should be
    // reviewed and approved by the human.
    llmOutput: state.llmOutput,
  });

  if (isApproved) {
    return new Command({ goto: "someNode" });
  } else {
    return new Command({ goto: "anotherNode" });
  }
});
const graph = graphBuilder.compile({ checkpointer });

// After running the graph and hitting the interrupt, the graph will pause.
// Resume it with either an approval or rejection.
const threadConfig = { configurable: { thread_id: "some_id" } };
await graph.invoke(new Command({ resume: true }), threadConfig);
```

:::

??? example "Extended example: approve or reject with interrupt"

    :::python
    ```python
    from typing import Literal, TypedDict
    import uuid

    from langgraph.constants import START, END
    from langgraph.graph import StateGraph
    from langgraph.types import interrupt, Command
    from langgraph.checkpoint.memory import InMemorySaver

    # Define the shared graph state
    class State(TypedDict):
        llm_output: str
        decision: str

    # Simulate an LLM output node
    def generate_llm_output(state: State) -> State:
        return {"llm_output": "This is the generated output."}

    # Human approval node
    def human_approval(state: State) -> Command[Literal["approved_path", "rejected_path"]]:
        decision = interrupt({
            "question": "Do you approve the following output?",
            "llm_output": state["llm_output"]
        })

        if decision == "approve":
            return Command(goto="approved_path", update={"decision": "approved"})
        else:
            return Command(goto="rejected_path", update={"decision": "rejected"})

    # Next steps after approval
    def approved_node(state: State) -> State:
        print("✅ Approved path taken.")
        return state

    # Alternative path after rejection
    def rejected_node(state: State) -> State:
        print("❌ Rejected path taken.")
        return state

    # Build the graph
    builder = StateGraph(State)
    builder.add_node("generate_llm_output", generate_llm_output)
    builder.add_node("human_approval", human_approval)
    builder.add_node("approved_path", approved_node)
    builder.add_node("rejected_path", rejected_node)

    builder.set_entry_point("generate_llm_output")
    builder.add_edge("generate_llm_output", "human_approval")
    builder.add_edge("approved_path", END)
    builder.add_edge("rejected_path", END)

    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    # Run until interrupt
    config = {"configurable": {"thread_id": uuid.uuid4()}}
    result = graph.invoke({}, config=config)
    print(result["__interrupt__"])
    # Output:
    # Interrupt(value={'question': 'Do you approve the following output?', 'llm_output': 'This is the generated output.'}, ...)

    # Simulate resuming with human input
    # To test rejection, replace resume="approve" with resume="reject"
    final_result = graph.invoke(Command(resume="approve"), config=config)
    print(final_result)
    ```
    :::

    :::js
    ```typescript
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";
    import {
      StateGraph,
      START,
      END,
      interrupt,
      Command,
      MemorySaver
    } from "@langchain/langgraph";

    // Define the shared graph state
    const StateAnnotation = z.object({
      llmOutput: z.string(),
      decision: z.string(),
    });

    // Simulate an LLM output node
    function generateLlmOutput(state: z.infer<typeof StateAnnotation>) {
      return { llmOutput: "This is the generated output." };
    }

    // Human approval node
    function humanApproval(state: z.infer<typeof StateAnnotation>): Command {
      const decision = interrupt({
        question: "Do you approve the following output?",
        llmOutput: state.llmOutput
      });

      if (decision === "approve") {
        return new Command({
          goto: "approvedPath",
          update: { decision: "approved" }
        });
      } else {
        return new Command({
          goto: "rejectedPath",
          update: { decision: "rejected" }
        });
      }
    }

    // Next steps after approval
    function approvedNode(state: z.infer<typeof StateAnnotation>) {
      console.log("✅ Approved path taken.");
      return state;
    }

    // Alternative path after rejection
    function rejectedNode(state: z.infer<typeof StateAnnotation>) {
      console.log("❌ Rejected path taken.");
      return state;
    }

    // Build the graph
    const builder = new StateGraph(StateAnnotation)
      .addNode("generateLlmOutput", generateLlmOutput)
      .addNode("humanApproval", humanApproval, {
        ends: ["approvedPath", "rejectedPath"]
      })
      .addNode("approvedPath", approvedNode)
      .addNode("rejectedPath", rejectedNode)
      .addEdge(START, "generateLlmOutput")
      .addEdge("generateLlmOutput", "humanApproval")
      .addEdge("approvedPath", END)
      .addEdge("rejectedPath", END);

    const checkpointer = new MemorySaver();
    const graph = builder.compile({ checkpointer });

    // Run until interrupt
    const config = { configurable: { thread_id: uuidv4() } };
    const result = await graph.invoke({}, config);
    console.log(result.__interrupt__);
    // Output:
    // [{
    //   value: {
    //     question: 'Do you approve the following output?',
    //     llmOutput: 'This is the generated output.'
    //   },
    //   ...
    // }]

    // Simulate resuming with human input
    // To test rejection, replace resume: "approve" with resume: "reject"
    const finalResult = await graph.invoke(
      new Command({ resume: "approve" }),
      config
    );
    console.log(finalResult);
    ```
    :::

### Review and edit state

<figure markdown="1">
![image](../../concepts/img/human_in_the_loop/edit-graph-state-simple.png){: style="max-height:400px"}
<figcaption>A human can review and edit the state of the graph. This is useful for correcting mistakes or updating the state with additional information.
</figcaption>
</figure>

:::python

```python
from langgraph.types import interrupt

def human_editing(state: State):
    ...
    result = interrupt(
        # Interrupt information to surface to the client.
        # Can be any JSON serializable value.
        {
            "task": "Review the output from the LLM and make any necessary edits.",
            "llm_generated_summary": state["llm_generated_summary"]
        }
    )

    # Update the state with the edited text
    return {
        "llm_generated_summary": result["edited_text"]
    }

# Add the node to the graph in an appropriate location
# and connect it to the relevant nodes.
graph_builder.add_node("human_editing", human_editing)
graph = graph_builder.compile(checkpointer=checkpointer)

...

# After running the graph and hitting the interrupt, the graph will pause.
# Resume it with the edited text.
thread_config = {"configurable": {"thread_id": "some_id"}}
graph.invoke(
    Command(resume={"edited_text": "The edited text"}),
    config=thread_config
)
```

:::

:::js

```typescript
import { interrupt } from "@langchain/langgraph";

function humanEditing(state: z.infer<typeof StateAnnotation>) {
  const result = interrupt({
    // Interrupt information to surface to the client.
    // Can be any JSON serializable value.
    task: "Review the output from the LLM and make any necessary edits.",
    llmGeneratedSummary: state.llmGeneratedSummary,
  });

  // Update the state with the edited text
  return {
    llmGeneratedSummary: result.editedText,
  };
}

// Add the node to the graph in an appropriate location
// and connect it to the relevant nodes.
graphBuilder.addNode("humanEditing", humanEditing);
const graph = graphBuilder.compile({ checkpointer });

// After running the graph and hitting the interrupt, the graph will pause.
// Resume it with the edited text.
const threadConfig = { configurable: { thread_id: "some_id" } };
await graph.invoke(
  new Command({ resume: { editedText: "The edited text" } }),
  threadConfig
);
```

:::

??? example "Extended example: edit state with interrupt"

    :::python
    ```python
    from typing import TypedDict
    import uuid

    from langgraph.constants import START, END
    from langgraph.graph import StateGraph
    from langgraph.types import interrupt, Command
    from langgraph.checkpoint.memory import InMemorySaver

    # Define the graph state
    class State(TypedDict):
        summary: str

    # Simulate an LLM summary generation
    def generate_summary(state: State) -> State:
        return {
            "summary": "The cat sat on the mat and looked at the stars."
        }

    # Human editing node
    def human_review_edit(state: State) -> State:
        result = interrupt({
            "task": "Please review and edit the generated summary if necessary.",
            "generated_summary": state["summary"]
        })
        return {
            "summary": result["edited_summary"]
        }

    # Simulate downstream use of the edited summary
    def downstream_use(state: State) -> State:
        print(f"✅ Using edited summary: {state['summary']}")
        return state

    # Build the graph
    builder = StateGraph(State)
    builder.add_node("generate_summary", generate_summary)
    builder.add_node("human_review_edit", human_review_edit)
    builder.add_node("downstream_use", downstream_use)

    builder.set_entry_point("generate_summary")
    builder.add_edge("generate_summary", "human_review_edit")
    builder.add_edge("human_review_edit", "downstream_use")
    builder.add_edge("downstream_use", END)

    # Set up in-memory checkpointing for interrupt support
    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    # Invoke the graph until it hits the interrupt
    config = {"configurable": {"thread_id": uuid.uuid4()}}
    result = graph.invoke({}, config=config)

    # Output interrupt payload
    print(result["__interrupt__"])
    # Example output:
    # > [
    # >     Interrupt(
    # >         value={
    # >             'task': 'Please review and edit the generated summary if necessary.',
    # >             'generated_summary': 'The cat sat on the mat and looked at the stars.'
    # >         },
    # >         id='...'
    # >     )
    # > ]

    # Resume the graph with human-edited input
    edited_summary = "The cat lay on the rug, gazing peacefully at the night sky."
    resumed_result = graph.invoke(
        Command(resume={"edited_summary": edited_summary}),
        config=config
    )
    print(resumed_result)
    ```
    :::

    :::js
    ```typescript
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";
    import {
      StateGraph,
      START,
      END,
      interrupt,
      Command,
      MemorySaver
    } from "@langchain/langgraph";

    // Define the graph state
    const StateAnnotation = z.object({
      summary: z.string(),
    });

    // Simulate an LLM summary generation
    function generateSummary(state: z.infer<typeof StateAnnotation>) {
      return {
        summary: "The cat sat on the mat and looked at the stars."
      };
    }

    // Human editing node
    function humanReviewEdit(state: z.infer<typeof StateAnnotation>) {
      const result = interrupt({
        task: "Please review and edit the generated summary if necessary.",
        generatedSummary: state.summary
      });
      return {
        summary: result.editedSummary
      };
    }

    // Simulate downstream use of the edited summary
    function downstreamUse(state: z.infer<typeof StateAnnotation>) {
      console.log(`✅ Using edited summary: ${state.summary}`);
      return state;
    }

    // Build the graph
    const builder = new StateGraph(StateAnnotation)
      .addNode("generateSummary", generateSummary)
      .addNode("humanReviewEdit", humanReviewEdit)
      .addNode("downstreamUse", downstreamUse)
      .addEdge(START, "generateSummary")
      .addEdge("generateSummary", "humanReviewEdit")
      .addEdge("humanReviewEdit", "downstreamUse")
      .addEdge("downstreamUse", END);

    // Set up in-memory checkpointing for interrupt support
    const checkpointer = new MemorySaver();
    const graph = builder.compile({ checkpointer });

    // Invoke the graph until it hits the interrupt
    const config = { configurable: { thread_id: uuidv4() } };
    const result = await graph.invoke({}, config);

    // Output interrupt payload
    console.log(result.__interrupt__);
    // Example output:
    // [{
    //   value: {
    //     task: 'Please review and edit the generated summary if necessary.',
    //     generatedSummary: 'The cat sat on the mat and looked at the stars.'
    //   },
    //   resumable: true,
    //   ...
    // }]

    // Resume the graph with human-edited input
    const editedSummary = "The cat lay on the rug, gazing peacefully at the night sky.";
    const resumedResult = await graph.invoke(
      new Command({ resume: { editedSummary } }),
      config
    );
    console.log(resumedResult);
    ```
    :::

### Review tool calls

<figure markdown="1">
![image](../../concepts/img/human_in_the_loop/tool-call-review.png){: style="max-height:400px"}
<figcaption>A human can review and edit the output from the LLM before proceeding. This is particularly
critical in applications where the tool calls requested by the LLM may be sensitive or require human oversight.
</figcaption>
</figure>

To add a human approval step to a tool:

1. Use `interrupt()` in the tool to pause execution.
2. Resume with a `Command` to continue based on human input.

:::python

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt
from langgraph.prebuilt import create_react_agent

# An example of a sensitive tool that requires human review / approval
def book_hotel(hotel_name: str):
    """Book a hotel"""
    # highlight-next-line
    response = interrupt(  # (1)!
        f"Trying to call `book_hotel` with args {{'hotel_name': {hotel_name}}}. "
        "Please approve or suggest edits."
    )
    if response["type"] == "accept":
        pass
    elif response["type"] == "edit":
        hotel_name = response["args"]["hotel_name"]
    else:
        raise ValueError(f"Unknown response type: {response['type']}")
    return f"Successfully booked a stay at {hotel_name}."

# highlight-next-line
checkpointer = InMemorySaver() # (2)!

agent = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    tools=[book_hotel],
    # highlight-next-line
    checkpointer=checkpointer, # (3)!
)
```

1. The @[`interrupt` function][interrupt] pauses the agent graph at a specific node. In this case, we call `interrupt()` at the beginning of the tool function, which pauses the graph at the node that executes the tool. The information inside `interrupt()` (e.g., tool calls) can be presented to a human, and the graph can be resumed with the user input (tool call approval, edit or feedback).
2. The `InMemorySaver` is used to store the agent state at every step in the tool calling loop. This enables [short-term memory](../memory/add-memory.md#add-short-term-memory) and [human-in-the-loop](../../concepts/human_in_the_loop.md) capabilities. In this example, we use `InMemorySaver` to store the agent state in memory. In a production application, the agent state will be stored in a database.
3. Initialize the agent with the `checkpointer`.
   :::

:::js

```typescript
import { MemorySaver } from "@langchain/langgraph";
import { interrupt } from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

// An example of a sensitive tool that requires human review / approval
const bookHotel = tool(
  async ({ hotelName }) => {
    // highlight-next-line
    const response = interrupt(
      // (1)!
      `Trying to call \`bookHotel\` with args {"hotelName": "${hotelName}"}. ` +
        "Please approve or suggest edits."
    );
    if (response.type === "accept") {
      // Continue with original args
    } else if (response.type === "edit") {
      hotelName = response.args.hotelName;
    } else {
      throw new Error(`Unknown response type: ${response.type}`);
    }
    return `Successfully booked a stay at ${hotelName}.`;
  },
  {
    name: "bookHotel",
    description: "Book a hotel",
    schema: z.object({
      hotelName: z.string(),
    }),
  }
);

// highlight-next-line
const checkpointer = new MemorySaver(); // (2)!

const agent = createReactAgent({
  llm: model,
  tools: [bookHotel],
  // highlight-next-line
  checkpointSaver: checkpointer, // (3)!
});
```

1. The @[`interrupt` function][interrupt] pauses the agent graph at a specific node. In this case, we call `interrupt()` at the beginning of the tool function, which pauses the graph at the node that executes the tool. The information inside `interrupt()` (e.g., tool calls) can be presented to a human, and the graph can be resumed with the user input (tool call approval, edit or feedback).
2. The `MemorySaver` is used to store the agent state at every step in the tool calling loop. This enables [short-term memory](../memory/add-memory.md#add-short-term-memory) and [human-in-the-loop](../../concepts/human_in_the_loop.md) capabilities. In this example, we use `MemorySaver` to store the agent state in memory. In a production application, the agent state will be stored in a database.
3. Initialize the agent with the `checkpointSaver`.
   :::

Run the agent with the `stream()` method, passing the `config` object to specify the thread ID. This allows the agent to resume the same conversation on future invocations.

:::python

```python
config = {
   "configurable": {
      # highlight-next-line
      "thread_id": "1"
   }
}

for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "book a stay at McKittrick hotel"}]},
    # highlight-next-line
    config
):
    print(chunk)
    print("\n")
```

:::

:::js

```typescript
const config = {
  configurable: {
    // highlight-next-line
    thread_id: "1",
  },
};

const stream = await agent.stream(
  { messages: [{ role: "user", content: "book a stay at McKittrick hotel" }] },
  // highlight-next-line
  config
);

for await (const chunk of stream) {
  console.log(chunk);
  console.log("\n");
}
```

:::

> You should see that the agent runs until it reaches the `interrupt()` call, at which point it pauses and waits for human input.

Resume the agent with a `Command` to continue based on human input.

:::python

```python
from langgraph.types import Command

for chunk in agent.stream(
    # highlight-next-line
    Command(resume={"type": "accept"}),  # (1)!
    # Command(resume={"type": "edit", "args": {"hotel_name": "McKittrick Hotel"}}),
    config
):
    print(chunk)
    print("\n")
```

1. The @[`interrupt` function][interrupt] is used in conjunction with the @[`Command`][Command] object to resume the graph with a value provided by the human.
   :::

:::js

```typescript
import { Command } from "@langchain/langgraph";

const resumeStream = await agent.stream(
  // highlight-next-line
  new Command({ resume: { type: "accept" } }), // (1)!
  // new Command({ resume: { type: "edit", args: { hotelName: "McKittrick Hotel" } } }),
  config
);

for await (const chunk of resumeStream) {
  console.log(chunk);
  console.log("\n");
}
```

1. The @[`interrupt` function][interrupt] is used in conjunction with the @[`Command`][Command] object to resume the graph with a value provided by the human.
   :::

### Add interrupts to any tool

You can create a wrapper to add interrupts to _any_ tool. The example below provides a reference implementation compatible with [Agent Inbox UI](https://github.com/langchain-ai/agent-inbox) and [Agent Chat UI](https://github.com/langchain-ai/agent-chat-ui).

:::python

```python title="Wrapper that adds human-in-the-loop to any tool"
from typing import Callable
from langchain_core.tools import BaseTool, tool as create_tool
from langchain_core.runnables import RunnableConfig
from langgraph.types import interrupt
from langgraph.prebuilt.interrupt import HumanInterruptConfig, HumanInterrupt

def add_human_in_the_loop(
    tool: Callable | BaseTool,
    *,
    interrupt_config: HumanInterruptConfig = None,
) -> BaseTool:
    """Wrap a tool to support human-in-the-loop review."""
    if not isinstance(tool, BaseTool):
        tool = create_tool(tool)

    if interrupt_config is None:
        interrupt_config = {
            "allow_accept": True,
            "allow_edit": True,
            "allow_respond": True,
        }

    @create_tool(  # (1)!
        tool.name,
        description=tool.description,
        args_schema=tool.args_schema
    )
    def call_tool_with_interrupt(config: RunnableConfig, **tool_input):
        request: HumanInterrupt = {
            "action_request": {
                "action": tool.name,
                "args": tool_input
            },
            "config": interrupt_config,
            "description": "Please review the tool call"
        }
        # highlight-next-line
        response = interrupt([request])[0]  # (2)!
        # approve the tool call
        if response["type"] == "accept":
            tool_response = tool.invoke(tool_input, config)
        # update tool call args
        elif response["type"] == "edit":
            tool_input = response["args"]["args"]
            tool_response = tool.invoke(tool_input, config)
        # respond to the LLM with user feedback
        elif response["type"] == "response":
            user_feedback = response["args"]
            tool_response = user_feedback
        else:
            raise ValueError(f"Unsupported interrupt response type: {response['type']}")

        return tool_response

    return call_tool_with_interrupt
```

1. This wrapper creates a new tool that calls `interrupt()` **before** executing the wrapped tool.
2. `interrupt()` is using special input and output format that's expected by [Agent Inbox UI](https://github.com/langchain-ai/agent-inbox): - a list of @[`HumanInterrupt`][HumanInterrupt] objects is sent to `AgentInbox` render interrupt information to the end user - resume value is provided by `AgentInbox` as a list (i.e., `Command(resume=[...])`)
   :::

:::js

```typescript title="Wrapper that adds human-in-the-loop to any tool"
import { StructuredTool, tool } from "@langchain/core/tools";
import { RunnableConfig } from "@langchain/core/runnables";
import { interrupt } from "@langchain/langgraph";

interface HumanInterruptConfig {
  allowAccept?: boolean;
  allowEdit?: boolean;
  allowRespond?: boolean;
}

interface HumanInterrupt {
  actionRequest: {
    action: string;
    args: Record<string, any>;
  };
  config: HumanInterruptConfig;
  description: string;
}

function addHumanInTheLoop(
  originalTool: StructuredTool,
  interruptConfig: HumanInterruptConfig = {
    allowAccept: true,
    allowEdit: true,
    allowRespond: true,
  }
): StructuredTool {
  // Wrap the original tool to support human-in-the-loop review
  return tool(
    // (1)!
    async (toolInput: Record<string, any>, config?: RunnableConfig) => {
      const request: HumanInterrupt = {
        actionRequest: {
          action: originalTool.name,
          args: toolInput,
        },
        config: interruptConfig,
        description: "Please review the tool call",
      };

      // highlight-next-line
      const response = interrupt([request])[0]; // (2)!

      // approve the tool call
      if (response.type === "accept") {
        return await originalTool.invoke(toolInput, config);
      }
      // update tool call args
      else if (response.type === "edit") {
        const updatedArgs = response.args.args;
        return await originalTool.invoke(updatedArgs, config);
      }
      // respond to the LLM with user feedback
      else if (response.type === "response") {
        return response.args;
      } else {
        throw new Error(
          `Unsupported interrupt response type: ${response.type}`
        );
      }
    },
    {
      name: originalTool.name,
      description: originalTool.description,
      schema: originalTool.schema,
    }
  );
}
```

1. This wrapper creates a new tool that calls `interrupt()` **before** executing the wrapped tool.
2. `interrupt()` is using special input and output format that's expected by [Agent Inbox UI](https://github.com/langchain-ai/agent-inbox): - a list of [`HumanInterrupt`] objects is sent to `AgentInbox` render interrupt information to the end user - resume value is provided by `AgentInbox` as a list (i.e., `Command({ resume: [...] })`)
   :::

You can use the wrapper to add `interrupt()` to any tool without having to add it _inside_ the tool:

:::python

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.prebuilt import create_react_agent

# highlight-next-line
checkpointer = InMemorySaver()

def book_hotel(hotel_name: str):
   """Book a hotel"""
   return f"Successfully booked a stay at {hotel_name}."


agent = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    tools=[
        # highlight-next-line
        add_human_in_the_loop(book_hotel), # (1)!
    ],
    # highlight-next-line
    checkpointer=checkpointer,
)

config = {"configurable": {"thread_id": "1"}}

# Run the agent
for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "book a stay at McKittrick hotel"}]},
    # highlight-next-line
    config
):
    print(chunk)
    print("\n")
```

1. The `add_human_in_the_loop` wrapper is used to add `interrupt()` to the tool. This allows the agent to pause execution and wait for human input before proceeding with the tool call.
   :::

:::js

```typescript
import { MemorySaver } from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

// highlight-next-line
const checkpointer = new MemorySaver();

const bookHotel = tool(
  async ({ hotelName }) => {
    return `Successfully booked a stay at ${hotelName}.`;
  },
  {
    name: "bookHotel",
    description: "Book a hotel",
    schema: z.object({
      hotelName: z.string(),
    }),
  }
);

const agent = createReactAgent({
  llm: model,
  tools: [
    // highlight-next-line
    addHumanInTheLoop(bookHotel), // (1)!
  ],
  // highlight-next-line
  checkpointSaver: checkpointer,
});

const config = { configurable: { thread_id: "1" } };

// Run the agent
const stream = await agent.stream(
  { messages: [{ role: "user", content: "book a stay at McKittrick hotel" }] },
  // highlight-next-line
  config
);

for await (const chunk of stream) {
  console.log(chunk);
  console.log("\n");
}
```

1. The `addHumanInTheLoop` wrapper is used to add `interrupt()` to the tool. This allows the agent to pause execution and wait for human input before proceeding with the tool call.
   :::

> You should see that the agent runs until it reaches the `interrupt()` call,
> at which point it pauses and waits for human input.

Resume the agent with a `Command` to continue based on human input.

:::python

```python
from langgraph.types import Command

for chunk in agent.stream(
    # highlight-next-line
    Command(resume=[{"type": "accept"}]),
    # Command(resume=[{"type": "edit", "args": {"args": {"hotel_name": "McKittrick Hotel"}}}]),
    config
):
    print(chunk)
    print("\n")
```

:::

:::js

```typescript
import { Command } from "@langchain/langgraph";

const resumeStream = await agent.stream(
  // highlight-next-line
  new Command({ resume: [{ type: "accept" }] }),
  // new Command({ resume: [{ type: "edit", args: { args: { hotelName: "McKittrick Hotel" } } }] }),
  config
);

for await (const chunk of resumeStream) {
  console.log(chunk);
  console.log("\n");
}
```

:::

### Validate human input

If you need to validate the input provided by the human within the graph itself (rather than on the client side), you can achieve this by using multiple interrupt calls within a single node.

:::python

```python
from langgraph.types import interrupt

def human_node(state: State):
    """Human node with validation."""
    question = "What is your age?"

    while True:
        answer = interrupt(question)

        # Validate answer, if the answer isn't valid ask for input again.
        if not isinstance(answer, int) or answer < 0:
            question = f"'{answer} is not a valid age. What is your age?"
            answer = None
            continue
        else:
            # If the answer is valid, we can proceed.
            break

    print(f"The human in the loop is {answer} years old.")
    return {
        "age": answer
    }
```

:::

:::js

```typescript
import { interrupt } from "@langchain/langgraph";

graphBuilder.addNode("humanNode", (state) => {
  // Human node with validation.
  let question = "What is your age?";

  while (true) {
    const answer = interrupt(question);

    // Validate answer, if the answer isn't valid ask for input again.
    if (typeof answer !== "number" || answer < 0) {
      question = `'${answer}' is not a valid age. What is your age?`;
      continue;
    } else {
      // If the answer is valid, we can proceed.
      break;
    }
  }

  console.log(`The human in the loop is ${answer} years old.`);
  return {
    age: answer,
  };
});
```

:::

??? example "Extended example: validating user input"

    :::python
    ```python
    from typing import TypedDict
    import uuid

    from langgraph.constants import START, END
    from langgraph.graph import StateGraph
    from langgraph.types import interrupt, Command
    from langgraph.checkpoint.memory import InMemorySaver

    # Define graph state
    class State(TypedDict):
        age: int

    # Node that asks for human input and validates it
    def get_valid_age(state: State) -> State:
        prompt = "Please enter your age (must be a non-negative integer)."

        while True:
            user_input = interrupt(prompt)

            # Validate the input
            try:
                age = int(user_input)
                if age < 0:
                    raise ValueError("Age must be non-negative.")
                break  # Valid input received
            except (ValueError, TypeError):
                prompt = f"'{user_input}' is not valid. Please enter a non-negative integer for age."

        return {"age": age}

    # Node that uses the valid input
    def report_age(state: State) -> State:
        print(f"✅ Human is {state['age']} years old.")
        return state

    # Build the graph
    builder = StateGraph(State)
    builder.add_node("get_valid_age", get_valid_age)
    builder.add_node("report_age", report_age)

    builder.set_entry_point("get_valid_age")
    builder.add_edge("get_valid_age", "report_age")
    builder.add_edge("report_age", END)

    # Create the graph with a memory checkpointer
    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    # Run the graph until the first interrupt
    config = {"configurable": {"thread_id": uuid.uuid4()}}
    result = graph.invoke({}, config=config)
    print(result["__interrupt__"])  # First prompt: "Please enter your age..."

    # Simulate an invalid input (e.g., string instead of integer)
    result = graph.invoke(Command(resume="not a number"), config=config)
    print(result["__interrupt__"])  # Follow-up prompt with validation message

    # Simulate a second invalid input (e.g., negative number)
    result = graph.invoke(Command(resume="-10"), config=config)
    print(result["__interrupt__"])  # Another retry

    # Provide valid input
    final_result = graph.invoke(Command(resume="25"), config=config)
    print(final_result)  # Should include the valid age
    ```
    :::

    :::js
    ```typescript
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";
    import {
      StateGraph,
      START,
      END,
      interrupt,
      Command,
      MemorySaver
    } from "@langchain/langgraph";

    // Define graph state
    const StateAnnotation = z.object({
      age: z.number(),
    });

    // Node that asks for human input and validates it
    function getValidAge(state: z.infer<typeof StateAnnotation>) {
      let prompt = "Please enter your age (must be a non-negative integer).";

      while (true) {
        const userInput = interrupt(prompt);

        // Validate the input
        try {
          const age = parseInt(userInput as string);
          if (isNaN(age) || age < 0) {
            throw new Error("Age must be non-negative.");
          }
          return { age };
        } catch (error) {
          prompt = `'${userInput}' is not valid. Please enter a non-negative integer for age.`;
        }
      }
    }

    // Node that uses the valid input
    function reportAge(state: z.infer<typeof StateAnnotation>) {
      console.log(`✅ Human is ${state.age} years old.`);
      return state;
    }

    // Build the graph
    const builder = new StateGraph(StateAnnotation)
      .addNode("getValidAge", getValidAge)
      .addNode("reportAge", reportAge)
      .addEdge(START, "getValidAge")
      .addEdge("getValidAge", "reportAge")
      .addEdge("reportAge", END);

    // Create the graph with a memory checkpointer
    const checkpointer = new MemorySaver();
    const graph = builder.compile({ checkpointer });

    // Run the graph until the first interrupt
    const config = { configurable: { thread_id: uuidv4() } };
    let result = await graph.invoke({}, config);
    console.log(result.__interrupt__);  // First prompt: "Please enter your age..."

    // Simulate an invalid input (e.g., string instead of integer)
    result = await graph.invoke(new Command({ resume: "not a number" }), config);
    console.log(result.__interrupt__);  // Follow-up prompt with validation message

    // Simulate a second invalid input (e.g., negative number)
    result = await graph.invoke(new Command({ resume: "-10" }), config);
    console.log(result.__interrupt__);  // Another retry

    // Provide valid input
    const finalResult = await graph.invoke(new Command({ resume: "25" }), config);
    console.log(finalResult);  // Should include the valid age
    ```
    :::

:::python

## Debug with interrupts

To debug and test a graph, use [static interrupts](../../concepts/human_in_the_loop.md#key-capabilities) (also known as static breakpoints) to step through the graph execution one node at a time or to pause the graph execution at specific nodes. Static interrupts are triggered at defined points either before or after a node executes. You can set static interrupts by specifying `interrupt_before` and `interrupt_after` at compile time or run time.

!!! warning

    Static interrupts are **not** recommended for human-in-the-loop workflows. Use [dynamic interrupts](#pause-using-interrupt) instead.

=== "Compile time"

    ```python
    # highlight-next-line
    graph = graph_builder.compile( # (1)!
        # highlight-next-line
        interrupt_before=["node_a"], # (2)!
        # highlight-next-line
        interrupt_after=["node_b", "node_c"], # (3)!
        checkpointer=checkpointer, # (4)!
    )

    config = {
        "configurable": {
            "thread_id": "some_thread"
        }
    }

    # Run the graph until the breakpoint
    graph.invoke(inputs, config=thread_config) # (5)!

    # Resume the graph
    graph.invoke(None, config=thread_config) # (6)!
    ```

    1. The breakpoints are set during `compile` time.
    2. `interrupt_before` specifies the nodes where execution should pause before the node is executed.
    3. `interrupt_after` specifies the nodes where execution should pause after the node is executed.
    4. A checkpointer is required to enable breakpoints.
    5. The graph is run until the first breakpoint is hit.
    6. The graph is resumed by passing in `None` for the input. This will run the graph until the next breakpoint is hit.

=== "Run time"

    ```python
    # highlight-next-line
    graph.invoke( # (1)!
        inputs,
        # highlight-next-line
        interrupt_before=["node_a"], # (2)!
        # highlight-next-line
        interrupt_after=["node_b", "node_c"] # (3)!
        config={
            "configurable": {"thread_id": "some_thread"}
        },
    )

    config = {
        "configurable": {
            "thread_id": "some_thread"
        }
    }

    # Run the graph until the breakpoint
    graph.invoke(inputs, config=config) # (4)!

    # Resume the graph
    graph.invoke(None, config=config) # (5)!
    ```

    1. `graph.invoke` is called with the `interrupt_before` and `interrupt_after` parameters. This is a run-time configuration and can be changed for every invocation.
    2. `interrupt_before` specifies the nodes where execution should pause before the node is executed.
    3. `interrupt_after` specifies the nodes where execution should pause after the node is executed.
    4. The graph is run until the first breakpoint is hit.
    5. The graph is resumed by passing in `None` for the input. This will run the graph until the next breakpoint is hit.

    !!! note

        You cannot set static breakpoints at runtime for **sub-graphs**.
        If you have a sub-graph, you must set the breakpoints at compilation time.

??? example "Setting static breakpoints"

    ```python
    from IPython.display import Image, display
    from typing_extensions import TypedDict

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.graph import StateGraph, START, END


    class State(TypedDict):
        input: str


    def step_1(state):
        print("---Step 1---")
        pass


    def step_2(state):
        print("---Step 2---")
        pass


    def step_3(state):
        print("---Step 3---")
        pass


    builder = StateGraph(State)
    builder.add_node("step_1", step_1)
    builder.add_node("step_2", step_2)
    builder.add_node("step_3", step_3)
    builder.add_edge(START, "step_1")
    builder.add_edge("step_1", "step_2")
    builder.add_edge("step_2", "step_3")
    builder.add_edge("step_3", END)

    # Set up a checkpointer
    checkpointer = InMemorySaver() # (1)!

    graph = builder.compile(
        checkpointer=checkpointer, # (2)!
        interrupt_before=["step_3"] # (3)!
    )

    # View
    display(Image(graph.get_graph().draw_mermaid_png()))


    # Input
    initial_input = {"input": "hello world"}

    # Thread
    thread = {"configurable": {"thread_id": "1"}}

    # Run the graph until the first interruption
    for event in graph.stream(initial_input, thread, stream_mode="values"):
        print(event)

    # This will run until the breakpoint
    # You can get the state of the graph at this point
    print(graph.get_state(config))

    # You can continue the graph execution by passing in `None` for the input
    for event in graph.stream(None, thread, stream_mode="values"):
        print(event)
    ```

### Use static interrupts in LangGraph Studio

You can use [LangGraph Studio](../../concepts/langgraph_studio.md) to debug your graph. You can set static breakpoints in the UI and then run the graph. You can also use the UI to inspect the graph state at any point in the execution.

![image](../../concepts/img/human_in_the_loop/static-interrupt.png){: style="max-height:400px"}

LangGraph Studio is free with [locally deployed applications](../../tutorials/langgraph-platform/local-server.md) using `langgraph dev`.

:::

## Debug with interrupts

To debug and test a graph, use [static interrupts](../../concepts/human_in_the_loop.md#key-capabilities) (also known as static breakpoints) to step through the graph execution one node at a time or to pause the graph execution at specific nodes. Static interrupts are triggered at defined points either before or after a node executes. You can set static interrupts by specifying `interrupt_before` and `interrupt_after` at compile time or run time.

!!! warning

    Static interrupts are **not** recommended for human-in-the-loop workflows. Use [dynamic interrupts](#pause-using-interrupt) instead.

=== "Compile time"

    ```python
    # highlight-next-line
    graph = graph_builder.compile( # (1)!
        # highlight-next-line
        interrupt_before=["node_a"], # (2)!
        # highlight-next-line
        interrupt_after=["node_b", "node_c"], # (3)!
        checkpointer=checkpointer, # (4)!
    )

    config = {
        "configurable": {
            "thread_id": "some_thread"
        }
    }

    # Run the graph until the breakpoint
    graph.invoke(inputs, config=thread_config) # (5)!

    # Resume the graph
    graph.invoke(None, config=thread_config) # (6)!
    ```

    1. The breakpoints are set during `compile` time.
    2. `interrupt_before` specifies the nodes where execution should pause before the node is executed.
    3. `interrupt_after` specifies the nodes where execution should pause after the node is executed.
    4. A checkpointer is required to enable breakpoints.
    5. The graph is run until the first breakpoint is hit.
    6. The graph is resumed by passing in `None` for the input. This will run the graph until the next breakpoint is hit.

=== "Run time"

    ```python
    # highlight-next-line
    graph.invoke( # (1)!
        inputs,
        # highlight-next-line
        interrupt_before=["node_a"], # (2)!
        # highlight-next-line
        interrupt_after=["node_b", "node_c"] # (3)!
        config={
            "configurable": {"thread_id": "some_thread"}
        },
    )

    config = {
        "configurable": {
            "thread_id": "some_thread"
        }
    }

    # Run the graph until the breakpoint
    graph.invoke(inputs, config=config) # (4)!

    # Resume the graph
    graph.invoke(None, config=config) # (5)!
    ```

    1. `graph.invoke` is called with the `interrupt_before` and `interrupt_after` parameters. This is a run-time configuration and can be changed for every invocation.
    2. `interrupt_before` specifies the nodes where execution should pause before the node is executed.
    3. `interrupt_after` specifies the nodes where execution should pause after the node is executed.
    4. The graph is run until the first breakpoint is hit.
    5. The graph is resumed by passing in `None` for the input. This will run the graph until the next breakpoint is hit.

    !!! note

        You cannot set static breakpoints at runtime for **sub-graphs**.
        If you have a sub-graph, you must set the breakpoints at compilation time.

??? example "Setting static breakpoints"

    ```python
    from IPython.display import Image, display
    from typing_extensions import TypedDict

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.graph import StateGraph, START, END


    class State(TypedDict):
        input: str


    def step_1(state):
        print("---Step 1---")
        pass


    def step_2(state):
        print("---Step 2---")
        pass


    def step_3(state):
        print("---Step 3---")
        pass


    builder = StateGraph(State)
    builder.add_node("step_1", step_1)
    builder.add_node("step_2", step_2)
    builder.add_node("step_3", step_3)
    builder.add_edge(START, "step_1")
    builder.add_edge("step_1", "step_2")
    builder.add_edge("step_2", "step_3")
    builder.add_edge("step_3", END)

    # Set up a checkpointer
    checkpointer = InMemorySaver() # (1)!

    graph = builder.compile(
        checkpointer=checkpointer, # (2)!
        interrupt_before=["step_3"] # (3)!
    )

    # View
    display(Image(graph.get_graph().draw_mermaid_png()))


    # Input
    initial_input = {"input": "hello world"}

    # Thread
    thread = {"configurable": {"thread_id": "1"}}

    # Run the graph until the first interruption
    for event in graph.stream(initial_input, thread, stream_mode="values"):
        print(event)

    # This will run until the breakpoint
    # You can get the state of the graph at this point
    print(graph.get_state(config))

    # You can continue the graph execution by passing in `None` for the input
    for event in graph.stream(None, thread, stream_mode="values"):
        print(event)
    ```

### Use static interrupts in LangGraph Studio

You can use [LangGraph Studio](../../concepts/langgraph_studio.md) to debug your graph. You can set static breakpoints in the UI and then run the graph. You can also use the UI to inspect the graph state at any point in the execution.

![image](../../concepts/img/human_in_the_loop/static-interrupt.png){: style="max-height:400px"}

LangGraph Studio is free with [locally deployed applications](../../tutorials/langgraph-platform/local-server.md) using `langgraph dev`.

## Considerations

When using human-in-the-loop, there are some considerations to keep in mind.

### Using with code with side-effects

Place code with side effects, such as API calls, after the `interrupt` or in a separate node to avoid duplication, as these are re-triggered every time the node is resumed.

=== "Side effects after interrupt"

    :::python
    ```python
    from langgraph.types import interrupt

    def human_node(state: State):
        """Human node with validation."""

        answer = interrupt(question)

        api_call(answer) # OK as it's after the interrupt
    ```
    :::

    :::js
    ```typescript
    import { interrupt } from "@langchain/langgraph";

    function humanNode(state: z.infer<typeof StateAnnotation>) {
      // Human node with validation.

      const answer = interrupt(question);

      apiCall(answer); // OK as it's after the interrupt
    }
    ```
    :::

=== "Side effects in a separate node"

    :::python
    ```python
    from langgraph.types import interrupt

    def human_node(state: State):
        """Human node with validation."""

        answer = interrupt(question)

        return {
            "answer": answer
        }

    def api_call_node(state: State):
        api_call(...) # OK as it's in a separate node
    ```
    :::

    :::js
    ```typescript
    import { interrupt } from "@langchain/langgraph";

    function humanNode(state: z.infer<typeof StateAnnotation>) {
      // Human node with validation.

      const answer = interrupt(question);

      return {
        answer
      };
    }

    function apiCallNode(state: z.infer<typeof StateAnnotation>) {
      apiCall(state.answer); // OK as it's in a separate node
    }
    ```
    :::

### Using with subgraphs called as functions

When invoking a subgraph as a function, the parent graph will resume execution from the **beginning of the node** where the subgraph was invoked where the `interrupt` was triggered. Similarly, the **subgraph** will resume from the **beginning of the node** where the `interrupt()` function was called.

:::python

```python
def node_in_parent_graph(state: State):
    some_code()  # <-- This will re-execute when the subgraph is resumed.
    # Invoke a subgraph as a function.
    # The subgraph contains an `interrupt` call.
    subgraph_result = subgraph.invoke(some_input)
    ...
```

:::

:::js

```typescript
async function nodeInParentGraph(state: z.infer<typeof StateAnnotation>) {
  someCode(); // <-- This will re-execute when the subgraph is resumed.
  // Invoke a subgraph as a function.
  // The subgraph contains an `interrupt` call.
  const subgraphResult = await subgraph.invoke(someInput);
  // ...
}
```

:::

??? example "Extended example: parent and subgraph execution flow"

    Say we have a parent graph with 3 nodes:

    **Parent Graph**: `node_1` → `node_2` (subgraph call) → `node_3`

    And the subgraph has 3 nodes, where the second node contains an `interrupt`:

    **Subgraph**: `sub_node_1` → `sub_node_2` (`interrupt`) → `sub_node_3`

    When resuming the graph, the execution will proceed as follows:

    1. **Skip `node_1`** in the parent graph (already executed, graph state was saved in snapshot).
    2. **Re-execute `node_2`** in the parent graph from the start.
    3. **Skip `sub_node_1`** in the subgraph (already executed, graph state was saved in snapshot).
    4. **Re-execute `sub_node_2`** in the subgraph from the beginning.
    5. Continue with `sub_node_3` and subsequent nodes.

    Here is abbreviated example code that you can use to understand how subgraphs work with interrupts.
    It counts the number of times each node is entered and prints the count.

    :::python
    ```python
    import uuid
    from typing import TypedDict

    from langgraph.graph import StateGraph
    from langgraph.constants import START
    from langgraph.types import interrupt, Command
    from langgraph.checkpoint.memory import InMemorySaver


    class State(TypedDict):
        """The graph state."""
        state_counter: int


    counter_node_in_subgraph = 0

    def node_in_subgraph(state: State):
        """A node in the sub-graph."""
        global counter_node_in_subgraph
        counter_node_in_subgraph += 1  # This code will **NOT** run again!
        print(f"Entered `node_in_subgraph` a total of {counter_node_in_subgraph} times")

    counter_human_node = 0

    def human_node(state: State):
        global counter_human_node
        counter_human_node += 1 # This code will run again!
        print(f"Entered human_node in sub-graph a total of {counter_human_node} times")
        answer = interrupt("what is your name?")
        print(f"Got an answer of {answer}")


    checkpointer = InMemorySaver()

    subgraph_builder = StateGraph(State)
    subgraph_builder.add_node("some_node", node_in_subgraph)
    subgraph_builder.add_node("human_node", human_node)
    subgraph_builder.add_edge(START, "some_node")
    subgraph_builder.add_edge("some_node", "human_node")
    subgraph = subgraph_builder.compile(checkpointer=checkpointer)


    counter_parent_node = 0

    def parent_node(state: State):
        """This parent node will invoke the subgraph."""
        global counter_parent_node

        counter_parent_node += 1 # This code will run again on resuming!
        print(f"Entered `parent_node` a total of {counter_parent_node} times")

        # Please note that we're intentionally incrementing the state counter
        # in the graph state as well to demonstrate that the subgraph update
        # of the same key will not conflict with the parent graph (until
        subgraph_state = subgraph.invoke(state)
        return subgraph_state


    builder = StateGraph(State)
    builder.add_node("parent_node", parent_node)
    builder.add_edge(START, "parent_node")

    # A checkpointer must be enabled for interrupts to work!
    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    config = {
        "configurable": {
          "thread_id": uuid.uuid4(),
        }
    }

    for chunk in graph.stream({"state_counter": 1}, config):
        print(chunk)

    print('--- Resuming ---')

    for chunk in graph.stream(Command(resume="35"), config):
        print(chunk)
    ```

    This will print out

    ```pycon
    Entered `parent_node` a total of 1 times
    Entered `node_in_subgraph` a total of 1 times
    Entered human_node in sub-graph a total of 1 times
    {'__interrupt__': (Interrupt(value='what is your name?', id='...'),)}
    --- Resuming ---
    Entered `parent_node` a total of 2 times
    Entered human_node in sub-graph a total of 2 times
    Got an answer of 35
    {'parent_node': {'state_counter': 1}}
    ```
    :::

    :::js
    ```typescript
    import { v4 as uuidv4 } from "uuid";
    import {
      StateGraph,
      START,
      interrupt,
      Command,
      MemorySaver
    } from "@langchain/langgraph";
    import { z } from "zod";

    const StateAnnotation = z.object({
      stateCounter: z.number(),
    });

    // Global variable to track the number of attempts
    let counterNodeInSubgraph = 0;

    function nodeInSubgraph(state: z.infer<typeof StateAnnotation>) {
      // A node in the sub-graph.
      counterNodeInSubgraph += 1; // This code will **NOT** run again!
      console.log(`Entered 'nodeInSubgraph' a total of ${counterNodeInSubgraph} times`);
      return {};
    }

    let counterHumanNode = 0;

    function humanNode(state: z.infer<typeof StateAnnotation>) {
      counterHumanNode += 1; // This code will run again!
      console.log(`Entered humanNode in sub-graph a total of ${counterHumanNode} times`);
      const answer = interrupt("what is your name?");
      console.log(`Got an answer of ${answer}`);
      return {};
    }

    const checkpointer = new MemorySaver();

    const subgraphBuilder = new StateGraph(StateAnnotation)
      .addNode("someNode", nodeInSubgraph)
      .addNode("humanNode", humanNode)
      .addEdge(START, "someNode")
      .addEdge("someNode", "humanNode");
    const subgraph = subgraphBuilder.compile({ checkpointer });

    let counterParentNode = 0;

    async function parentNode(state: z.infer<typeof StateAnnotation>) {
      // This parent node will invoke the subgraph.
      counterParentNode += 1; // This code will run again on resuming!
      console.log(`Entered 'parentNode' a total of ${counterParentNode} times`);

      // Please note that we're intentionally incrementing the state counter
      // in the graph state as well to demonstrate that the subgraph update
      // of the same key will not conflict with the parent graph (until
      const subgraphState = await subgraph.invoke(state);
      return subgraphState;
    }

    const builder = new StateGraph(StateAnnotation)
      .addNode("parentNode", parentNode)
      .addEdge(START, "parentNode");

    // A checkpointer must be enabled for interrupts to work!
    const graph = builder.compile({ checkpointer });

    const config = {
      configurable: {
        thread_id: uuidv4(),
      }
    };

    const stream = await graph.stream({ stateCounter: 1 }, config);
    for await (const chunk of stream) {
      console.log(chunk);
    }

    console.log('--- Resuming ---');

    const resumeStream = await graph.stream(new Command({ resume: "35" }), config);
    for await (const chunk of resumeStream) {
      console.log(chunk);
    }
    ```

    This will print out

    ```
    Entered 'parentNode' a total of 1 times
    Entered 'nodeInSubgraph' a total of 1 times
    Entered humanNode in sub-graph a total of 1 times
    { __interrupt__: [{ value: 'what is your name?', resumable: true, ns: ['parentNode:4c3a0248-21f0-1287-eacf-3002bc304db4', 'humanNode:2fe86d52-6f70-2a3f-6b2f-b1eededd6348'], when: 'during' }] }
    --- Resuming ---
    Entered 'parentNode' a total of 2 times
    Entered humanNode in sub-graph a total of 2 times
    Got an answer of 35
    { parentNode: null }
    ```
    :::

### Using multiple interrupts in a single node

Using multiple interrupts within a **single** node can be helpful for patterns like [validating human input](#validate-human-input). However, using multiple interrupts in the same node can lead to unexpected behavior if not handled carefully.

When a node contains multiple interrupt calls, LangGraph keeps a list of resume values specific to the task executing the node. Whenever execution resumes, it starts at the beginning of the node. For each interrupt encountered, LangGraph checks if a matching value exists in the task's resume list. Matching is **strictly index-based**, so the order of interrupt calls within the node is critical.

To avoid issues, refrain from dynamically changing the node's structure between executions. This includes adding, removing, or reordering interrupt calls, as such changes can result in mismatched indices. These problems often arise from unconventional patterns, such as mutating state via `Command(resume=..., update=SOME_STATE_MUTATION)` or relying on global variables to modify the node's structure dynamically.

:::python
??? example "Extended example: incorrect code that introduces non-determinism"

    ```python
    import uuid
    from typing import TypedDict, Optional

    from langgraph.graph import StateGraph
    from langgraph.constants import START
    from langgraph.types import interrupt, Command
    from langgraph.checkpoint.memory import InMemorySaver


    class State(TypedDict):
        """The graph state."""

        age: Optional[str]
        name: Optional[str]


    def human_node(state: State):
        if not state.get('name'):
            name = interrupt("what is your name?")
        else:
            name = "N/A"

        if not state.get('age'):
            age = interrupt("what is your age?")
        else:
            age = "N/A"

        print(f"Name: {name}. Age: {age}")

        return {
            "age": age,
            "name": name,
        }


    builder = StateGraph(State)
    builder.add_node("human_node", human_node)
    builder.add_edge(START, "human_node")

    # A checkpointer must be enabled for interrupts to work!
    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    config = {
        "configurable": {
            "thread_id": uuid.uuid4(),
        }
    }

    for chunk in graph.stream({"age": None, "name": None}, config):
        print(chunk)

    for chunk in graph.stream(Command(resume="John", update={"name": "foo"}), config):
        print(chunk)
    ```

    ```pycon
    {'__interrupt__': (Interrupt(value='what is your name?', id='...'),)}
    Name: N/A. Age: John
    {'human_node': {'age': 'John', 'name': 'N/A'}}
    ```

:::

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/human_in_the_loop/time-travel.md
```md
# Use time-travel

To use [time-travel](../../concepts/time-travel.md) in LangGraph:

:::python

1. [Run the graph](#1-run-the-graph) with initial inputs using @[`invoke`][CompiledStateGraph.invoke] or @[`stream`][CompiledStateGraph.stream] methods.
2. [Identify a checkpoint in an existing thread](#2-identify-a-checkpoint): Use the @[`get_state_history()`][get_state_history] method to retrieve the execution history for a specific `thread_id` and locate the desired `checkpoint_id`.  
   Alternatively, set an [interrupt](../../how-tos/human_in_the_loop/add-human-in-the-loop.md) before the node(s) where you want execution to pause. You can then find the most recent checkpoint recorded up to that interrupt.
3. [Update the graph state (optional)](#3-update-the-state-optional): Use the @[`update_state`][update_state] method to modify the graph's state at the checkpoint and resume execution from alternative state.
4. [Resume execution from the checkpoint](#4-resume-execution-from-the-checkpoint): Use the `invoke` or `stream` methods with an input of `None` and a configuration containing the appropriate `thread_id` and `checkpoint_id`.
   :::

:::js

1. [Run the graph](#1-run-the-graph) with initial inputs using @[`invoke`][CompiledStateGraph.invoke] or @[`stream`][CompiledStateGraph.stream] methods.
2. [Identify a checkpoint in an existing thread](#2-identify-a-checkpoint): Use the @[`getStateHistory()`][get_state_history] method to retrieve the execution history for a specific `thread_id` and locate the desired `checkpoint_id`.  
   Alternatively, set a [breakpoint](../../concepts/breakpoints.md) before the node(s) where you want execution to pause. You can then find the most recent checkpoint recorded up to that breakpoint.
3. [Update the graph state (optional)](#3-update-the-state-optional): Use the @[`updateState`][update_state] method to modify the graph's state at the checkpoint and resume execution from alternative state.
4. [Resume execution from the checkpoint](#4-resume-execution-from-the-checkpoint): Use the `invoke` or `stream` methods with an input of `null` and a configuration containing the appropriate `thread_id` and `checkpoint_id`.
   :::

!!! tip

    For a conceptual overview of time-travel, see [Time travel](../../concepts/time-travel.md).

## In a workflow

This example builds a simple LangGraph workflow that generates a joke topic and writes a joke using an LLM. It demonstrates how to run the graph, retrieve past execution checkpoints, optionally modify the state, and resume execution from a chosen checkpoint to explore alternate outcomes.

### Setup

First we need to install the packages required

:::python

```python
%%capture --no-stderr
%pip install --quiet -U langgraph langchain_anthropic
```

:::

:::js

```bash
npm install @langchain/langgraph @langchain/anthropic
```

:::

Next, we need to set API keys for Anthropic (the LLM we will use)

:::python

```python
import getpass
import os


def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")


_set_env("ANTHROPIC_API_KEY")
```

:::

:::js

```typescript
process.env.ANTHROPIC_API_KEY = "YOUR_API_KEY";
```

:::

<div class="admonition tip">
    <p class="admonition-title">Set up <a href="https://smith.langchain.com">LangSmith</a> for LangGraph development</p>
    <p style="padding-top: 5px;">
        Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. LangSmith lets you use trace data to debug, test, and monitor your LLM apps built with LangGraph — read more about how to get started <a href="https://docs.smith.langchain.com">here</a>. 
    </p>
</div>

:::python

```python
import uuid

from typing_extensions import TypedDict, NotRequired
from langgraph.graph import StateGraph, START, END
from langchain.chat_models import init_chat_model
from langgraph.checkpoint.memory import InMemorySaver


class State(TypedDict):
    topic: NotRequired[str]
    joke: NotRequired[str]


llm = init_chat_model(
    "anthropic:claude-3-7-sonnet-latest",
    temperature=0,
)


def generate_topic(state: State):
    """LLM call to generate a topic for the joke"""
    msg = llm.invoke("Give me a funny topic for a joke")
    return {"topic": msg.content}


def write_joke(state: State):
    """LLM call to write a joke based on the topic"""
    msg = llm.invoke(f"Write a short joke about {state['topic']}")
    return {"joke": msg.content}


# Build workflow
workflow = StateGraph(State)

# Add nodes
workflow.add_node("generate_topic", generate_topic)
workflow.add_node("write_joke", write_joke)

# Add edges to connect nodes
workflow.add_edge(START, "generate_topic")
workflow.add_edge("generate_topic", "write_joke")
workflow.add_edge("write_joke", END)

# Compile
checkpointer = InMemorySaver()
graph = workflow.compile(checkpointer=checkpointer)
graph
```

:::

:::js

```typescript
import { v4 as uuidv4 } from "uuid";
import { z } from "zod";
import { StateGraph, START, END } from "@langchain/langgraph";
import { ChatAnthropic } from "@langchain/anthropic";
import { MemorySaver } from "@langchain/langgraph";

const State = z.object({
  topic: z.string().optional(),
  joke: z.string().optional(),
});

const llm = new ChatAnthropic({
  model: "claude-3-5-sonnet-latest",
  temperature: 0,
});

// Build workflow
const workflow = new StateGraph(State)
  // Add nodes
  .addNode("generateTopic", async (state) => {
    // LLM call to generate a topic for the joke
    const msg = await llm.invoke("Give me a funny topic for a joke");
    return { topic: msg.content };
  })
  .addNode("writeJoke", async (state) => {
    // LLM call to write a joke based on the topic
    const msg = await llm.invoke(`Write a short joke about ${state.topic}`);
    return { joke: msg.content };
  })
  // Add edges to connect nodes
  .addEdge(START, "generateTopic")
  .addEdge("generateTopic", "writeJoke")
  .addEdge("writeJoke", END);

// Compile
const checkpointer = new MemorySaver();
const graph = workflow.compile({ checkpointer });
```

:::

### 1. Run the graph

:::python

```python
config = {
    "configurable": {
        "thread_id": uuid.uuid4(),
    }
}
state = graph.invoke({}, config)

print(state["topic"])
print()
print(state["joke"])
```

:::

:::js

```typescript
const config = {
  configurable: {
    thread_id: uuidv4(),
  },
};

const state = await graph.invoke({}, config);

console.log(state.topic);
console.log();
console.log(state.joke);
```

:::

**Output:**

```
How about "The Secret Life of Socks in the Dryer"? You know, exploring the mysterious phenomenon of how socks go into the laundry as pairs but come out as singles. Where do they go? Are they starting new lives elsewhere? Is there a sock paradise we don't know about? There's a lot of comedic potential in the everyday mystery that unites us all!

# The Secret Life of Socks in the Dryer

I finally discovered where all my missing socks go after the dryer. Turns out they're not missing at all—they've just eloped with someone else's socks from the laundromat to start new lives together.

My blue argyle is now living in Bermuda with a red polka dot, posting vacation photos on Sockstagram and sending me lint as alimony.
```

### 2. Identify a checkpoint

:::python

```python
# The states are returned in reverse chronological order.
states = list(graph.get_state_history(config))

for state in states:
    print(state.next)
    print(state.config["configurable"]["checkpoint_id"])
    print()
```

**Output:**

```
()
1f02ac4a-ec9f-6524-8002-8f7b0bbeed0e

('write_joke',)
1f02ac4a-ce2a-6494-8001-cb2e2d651227

('generate_topic',)
1f02ac4a-a4e0-630d-8000-b73c254ba748

('__start__',)
1f02ac4a-a4dd-665e-bfff-e6c8c44315d9
```

:::

:::js

```typescript
// The states are returned in reverse chronological order.
const states = [];
for await (const state of graph.getStateHistory(config)) {
  states.push(state);
}

for (const state of states) {
  console.log(state.next);
  console.log(state.config.configurable?.checkpoint_id);
  console.log();
}
```

**Output:**

```
[]
1f02ac4a-ec9f-6524-8002-8f7b0bbeed0e

['writeJoke']
1f02ac4a-ce2a-6494-8001-cb2e2d651227

['generateTopic']
1f02ac4a-a4e0-630d-8000-b73c254ba748

['__start__']
1f02ac4a-a4dd-665e-bfff-e6c8c44315d9
```

:::

:::python

```python
# This is the state before last (states are listed in chronological order)
selected_state = states[1]
print(selected_state.next)
print(selected_state.values)
```

**Output:**

```
('write_joke',)
{'topic': 'How about "The Secret Life of Socks in the Dryer"? You know, exploring the mysterious phenomenon of how socks go into the laundry as pairs but come out as singles. Where do they go? Are they starting new lives elsewhere? Is there a sock paradise we don\\'t know about? There\\'s a lot of comedic potential in the everyday mystery that unites us all!'}
```

:::

:::js

```typescript
// This is the state before last (states are listed in chronological order)
const selectedState = states[1];
console.log(selectedState.next);
console.log(selectedState.values);
```

**Output:**

```
['writeJoke']
{'topic': 'How about "The Secret Life of Socks in the Dryer"? You know, exploring the mysterious phenomenon of how socks go into the laundry as pairs but come out as singles. Where do they go? Are they starting new lives elsewhere? Is there a sock paradise we don\\'t know about? There\\'s a lot of comedic potential in the everyday mystery that unites us all!'}
```

:::

### 3. Update the state (optional)

:::python
`update_state` will create a new checkpoint. The new checkpoint will be associated with the same thread, but a new checkpoint ID.

```python
new_config = graph.update_state(selected_state.config, values={"topic": "chickens"})
print(new_config)
```

**Output:**

```
{'configurable': {'thread_id': 'c62e2e03-c27b-4cb6-8cea-ea9bfedae006', 'checkpoint_ns': '', 'checkpoint_id': '1f02ac4a-ecee-600b-8002-a1d21df32e4c'}}
```

:::

:::js
`updateState` will create a new checkpoint. The new checkpoint will be associated with the same thread, but a new checkpoint ID.

```typescript
const newConfig = await graph.updateState(selectedState.config, {
  topic: "chickens",
});
console.log(newConfig);
```

**Output:**

```
{'configurable': {'thread_id': 'c62e2e03-c27b-4cb6-8cea-ea9bfedae006', 'checkpoint_ns': '', 'checkpoint_id': '1f02ac4a-ecee-600b-8002-a1d21df32e4c'}}
```

:::

### 4. Resume execution from the checkpoint

:::python

```python
graph.invoke(None, new_config)
```

**Output:**

```python
{'topic': 'chickens',
 'joke': 'Why did the chicken join a band?\n\nBecause it had excellent drumsticks!'}
```

:::

:::js

```typescript
await graph.invoke(null, newConfig);
```

**Output:**

```typescript
{
  'topic': 'chickens',
  'joke': 'Why did the chicken join a band?\n\nBecause it had excellent drumsticks!'
}
```

:::

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/human_in_the_loop/wait-user-input.ipynb
```ipynb
{
 "cells": [
  {
   "attachments": {},
   "cell_type": "markdown",
   "id": "51466c8d-8ce4-4b3d-be4e-18fdbeda5f53",
   "metadata": {},
   "source": [
    "# How to wait for user input using `interrupt`\n",
    "\n",
    "!!! tip \"Prerequisites\"\n",
    "\n",
    "    This guide assumes familiarity with the following concepts:\n",
    "\n",
    "    * [Human-in-the-loop](../../../concepts/human_in_the_loop)\n",
    "    * [LangGraph Glossary](../../../concepts/low_level)\n",
    "    \n",
    "\n",
    "**Human-in-the-loop (HIL)** interactions are crucial for [agentic systems](https://langchain-ai.github.io/langgraph/concepts/agentic_concepts/#human-in-the-loop). Waiting for human input is a common HIL interaction pattern, allowing the agent to ask the user clarifying questions and await input before proceeding. \n",
    "\n",
    "We can implement this in LangGraph using the [`interrupt()`][langgraph.types.interrupt] function. `interrupt` allows us to stop graph execution to collect input from a user and continue execution with collected input."
   ]
  },
  {
   "cell_type": "markdown",
   "id": "7cbd446a-808f-4394-be92-d45ab818953c",
   "metadata": {},
   "source": [
    "## Setup\n",
    "\n",
    "First we need to install the packages required"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 1,
   "id": "af4ce0ba-7596-4e5f-8bf8-0b0bd6e62833",
   "metadata": {},
   "outputs": [],
   "source": [
    "%%capture --no-stderr\n",
    "%pip install --quiet -U langgraph langchain_anthropic"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "0abe11f4-62ed-4dc4-8875-3db21e260d1d",
   "metadata": {},
   "source": [
    "Next, we need to set API keys for Anthropic and / or OpenAI (the LLM(s) we will use)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "c903a1cf-2977-4e2d-ad7d-8b3946821d89",
   "metadata": {},
   "outputs": [],
   "source": [
    "import getpass\n",
    "import os\n",
    "\n",
    "\n",
    "def _set_env(var: str):\n",
    "    if not os.environ.get(var):\n",
    "        os.environ[var] = getpass.getpass(f\"{var}: \")\n",
    "\n",
    "\n",
    "_set_env(\"ANTHROPIC_API_KEY\")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "f0ed46a8-effe-4596-b0e1-a6a29ee16f5c",
   "metadata": {},
   "source": [
    "<div class=\"admonition tip\">\n",
    "    <p class=\"admonition-title\">Set up <a href=\"https://smith.langchain.com\">LangSmith</a> for LangGraph development</p>\n",
    "    <p style=\"padding-top: 5px;\">\n",
    "        Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. LangSmith lets you use trace data to debug, test, and monitor your LLM apps built with LangGraph — read more about how to get started <a href=\"https://docs.smith.langchain.com\">here</a>. \n",
    "    </p>\n",
    "</div>"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "e6cf1fad-5ab6-49c5-b0c8-15a1b6e8cf21",
   "metadata": {},
   "source": [
    "## Simple Usage\n",
    "\n",
    "Let's explore a basic example of using human feedback. A straightforward approach is to create a node, **`human_feedback`**, designed specifically to collect user input. This allows us to gather feedback at a specific, chosen point in our graph.\n",
    "\n",
    "Steps:\n",
    "\n",
    "1. **Call `interrupt()`** inside the **`human_feedback`** node.  \n",
    "2. **Set up a [checkpointer](https://langchain-ai.github.io/langgraph/concepts/low_level/#checkpointer)** to save the graph's state up to this node.  \n",
    "3. **Use `Command(resume=...)`** to provide the requested value to the **`human_feedback`** node and resume execution."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 3,
   "id": "58eae42d-be32-48da-8d0a-ab64471657d9",
   "metadata": {},
   "outputs": [
    {
     "data": {
      "image/png": "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",
      "text/plain": [
       "<IPython.core.display.Image object>"
      ]
     },
     "metadata": {},
     "output_type": "display_data"
    }
   ],
   "source": [
    "from typing_extensions import TypedDict\n",
    "from langgraph.graph import StateGraph, START, END\n",
    "\n",
    "# highlight-next-line\n",
    "from langgraph.types import Command, interrupt\n",
    "from langgraph.checkpoint.memory import InMemorySaver\n",
    "from IPython.display import Image, display\n",
    "\n",
    "\n",
    "class State(TypedDict):\n",
    "    input: str\n",
    "    user_feedback: str\n",
    "\n",
    "\n",
    "def step_1(state):\n",
    "    print(\"---Step 1---\")\n",
    "    pass\n",
    "\n",
    "\n",
    "def human_feedback(state):\n",
    "    print(\"---human_feedback---\")\n",
    "    # highlight-next-line\n",
    "    feedback = interrupt(\"Please provide feedback:\")\n",
    "    return {\"user_feedback\": feedback}\n",
    "\n",
    "\n",
    "def step_3(state):\n",
    "    print(\"---Step 3---\")\n",
    "    pass\n",
    "\n",
    "\n",
    "builder = StateGraph(State)\n",
    "builder.add_node(\"step_1\", step_1)\n",
    "builder.add_node(\"human_feedback\", human_feedback)\n",
    "builder.add_node(\"step_3\", step_3)\n",
    "builder.add_edge(START, \"step_1\")\n",
    "builder.add_edge(\"step_1\", \"human_feedback\")\n",
    "builder.add_edge(\"human_feedback\", \"step_3\")\n",
    "builder.add_edge(\"step_3\", END)\n",
    "\n",
    "# Set up memory\n",
    "memory = InMemorySaver()\n",
    "\n",
    "# Add\n",
    "graph = builder.compile(checkpointer=memory)\n",
    "\n",
    "# View\n",
    "display(Image(graph.get_graph().draw_mermaid_png()))"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "ce0fe2bc-86fc-465f-956c-729805d50404",
   "metadata": {},
   "source": [
    "Run until our `interrupt()` at `human_feedback`:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 4,
   "id": "eb8e7d47-e7c9-4217-b72c-08394a2c4d3e",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "---Step 1---\n",
      "{'step_1': None}\n",
      "\n",
      "\n",
      "---human_feedback---\n",
      "{'__interrupt__': (Interrupt(value='Please provide feedback:', resumable=True, ns=['human_feedback:c723d73a-d2cb-32cf-452f-e147367868bd']),)}\n",
      "\n",
      "\n"
     ]
    }
   ],
   "source": [
    "# Input\n",
    "initial_input = {\"input\": \"hello world\"}\n",
    "\n",
    "# Thread\n",
    "thread = {\"configurable\": {\"thread_id\": \"1\"}}\n",
    "\n",
    "# Run the graph until the first interruption\n",
    "for event in graph.stream(initial_input, thread, stream_mode=\"updates\"):\n",
    "    print(event)\n",
    "    print(\"\\n\")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "28a7d545-ab19-4800-985b-62837d060809",
   "metadata": {},
   "source": [
    "Now, we can manually update our graph state with the user input:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 5,
   "id": "3cca588f-e8d8-416b-aba7-0f3ae5e51598",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "---human_feedback---\n",
      "{'human_feedback': {'user_feedback': 'go to step 3!'}}\n",
      "\n",
      "\n",
      "---Step 3---\n",
      "{'step_3': None}\n",
      "\n",
      "\n"
     ]
    }
   ],
   "source": [
    "# Continue the graph execution\n",
    "for event in graph.stream(\n",
    "    # highlight-next-line\n",
    "    Command(resume=\"go to step 3!\"),\n",
    "    thread,\n",
    "    stream_mode=\"updates\",\n",
    "):\n",
    "    print(event)\n",
    "    print(\"\\n\")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "a75a1060-47aa-4cc6-8c41-e6ba2e9d7923",
   "metadata": {},
   "source": [
    "We can see our feedback was added to state - "
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 6,
   "id": "2b83e5ca-8497-43ca-bff7-7203e654c4d3",
   "metadata": {},
   "outputs": [
    {
     "data": {
      "text/plain": [
       "{'input': 'hello world', 'user_feedback': 'go to step 3!'}"
      ]
     },
     "execution_count": 6,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "graph.get_state(thread).values"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "b22b9598-7ce4-4d16-b932-bba2bc2803ec",
   "metadata": {},
   "source": [
    "## Agent\n",
    "\n",
    "In the context of [agents](../../../concepts/agentic_concepts), waiting for user feedback is especially useful for asking clarifying questions. To illustrate this, we’ll create a simple [ReAct-style agent](../../../concepts/agentic_concepts#react-implementation) capable of [tool calling](https://python.langchain.com/docs/concepts/tool_calling/). \n",
    "\n",
    "For this example, we’ll use Anthropic's chat model along with a **mock tool** (purely for demonstration purposes)."
   ]
  },
  {
   "cell_type": "markdown",
   "id": "01789855-b769-426d-a329-3cdb29684df8",
   "metadata": {},
   "source": [
    "<div class=\"admonition note\">\n",
    "    <p class=\"admonition-title\">Using Pydantic with LangChain</p>\n",
    "    <p>\n",
    "        This notebook uses Pydantic v2 <code>BaseModel</code>, which requires <code>langchain-core >= 0.3</code>. Using <code>langchain-core < 0.3</code> will result in errors due to mixing of Pydantic v1 and v2 <code>BaseModels</code>.\n",
    "    </p>\n",
    "</div>  "
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 7,
   "id": "f5319e01",
   "metadata": {},
   "outputs": [
    {
     "data": {
      "image/png": "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",
      "text/plain": [
       "<IPython.core.display.Image object>"
      ]
     },
     "metadata": {},
     "output_type": "display_data"
    }
   ],
   "source": [
    "# Set up the state\n",
    "from langgraph.graph import MessagesState, START\n",
    "\n",
    "# Set up the tool\n",
    "# We will have one real tool - a search tool\n",
    "# We'll also have one \"fake\" tool - a \"ask_human\" tool\n",
    "# Here we define any ACTUAL tools\n",
    "from langchain_core.tools import tool\n",
    "from langgraph.prebuilt import ToolNode\n",
    "\n",
    "\n",
    "@tool\n",
    "def search(query: str):\n",
    "    \"\"\"Call to surf the web.\"\"\"\n",
    "    # This is a placeholder for the actual implementation\n",
    "    # Don't let the LLM know this though 😊\n",
    "    return f\"I looked up: {query}. Result: It's sunny in San Francisco, but you better look out if you're a Gemini 😈.\"\n",
    "\n",
    "\n",
    "tools = [search]\n",
    "tool_node = ToolNode(tools)\n",
    "\n",
    "# Set up the model\n",
    "from langchain_anthropic import ChatAnthropic\n",
    "\n",
    "model = ChatAnthropic(model=\"claude-3-5-sonnet-latest\")\n",
    "\n",
    "from pydantic import BaseModel\n",
    "\n",
    "\n",
    "# We are going \"bind\" all tools to the model\n",
    "# We have the ACTUAL tools from above, but we also need a mock tool to ask a human\n",
    "# Since `bind_tools` takes in tools but also just tool definitions,\n",
    "# We can define a tool definition for `ask_human`\n",
    "class AskHuman(BaseModel):\n",
    "    \"\"\"Ask the human a question\"\"\"\n",
    "\n",
    "    question: str\n",
    "\n",
    "\n",
    "model = model.bind_tools(tools + [AskHuman])\n",
    "\n",
    "# Define nodes and conditional edges\n",
    "\n",
    "\n",
    "# Define the function that determines whether to continue or not\n",
    "def should_continue(state):\n",
    "    messages = state[\"messages\"]\n",
    "    last_message = messages[-1]\n",
    "    # If there is no function call, then we finish\n",
    "    if not last_message.tool_calls:\n",
    "        return END\n",
    "    # If tool call is asking Human, we return that node\n",
    "    # You could also add logic here to let some system know that there's something that requires Human input\n",
    "    # For example, send a slack message, etc\n",
    "    elif last_message.tool_calls[0][\"name\"] == \"AskHuman\":\n",
    "        return \"ask_human\"\n",
    "    # Otherwise if there is, we continue\n",
    "    else:\n",
    "        return \"action\"\n",
    "\n",
    "\n",
    "# Define the function that calls the model\n",
    "def call_model(state):\n",
    "    messages = state[\"messages\"]\n",
    "    response = model.invoke(messages)\n",
    "    # We return a list, because this will get added to the existing list\n",
    "    return {\"messages\": [response]}\n",
    "\n",
    "\n",
    "# We define a fake node to ask the human\n",
    "def ask_human(state):\n",
    "    tool_call_id = state[\"messages\"][-1].tool_calls[0][\"id\"]\n",
    "    ask = AskHuman.model_validate(state[\"messages\"][-1].tool_calls[0][\"args\"])\n",
    "    # highlight-next-line\n",
    "    location = interrupt(ask.question)\n",
    "    tool_message = [{\"tool_call_id\": tool_call_id, \"type\": \"tool\", \"content\": location}]\n",
    "    return {\"messages\": tool_message}\n",
    "\n",
    "\n",
    "# Build the graph\n",
    "\n",
    "from langgraph.graph import END, StateGraph\n",
    "\n",
    "# Define a new graph\n",
    "workflow = StateGraph(MessagesState)\n",
    "\n",
    "# Define the three nodes we will cycle between\n",
    "workflow.add_node(\"agent\", call_model)\n",
    "workflow.add_node(\"action\", tool_node)\n",
    "workflow.add_node(\"ask_human\", ask_human)\n",
    "\n",
    "# Set the entrypoint as `agent`\n",
    "# This means that this node is the first one called\n",
    "workflow.add_edge(START, \"agent\")\n",
    "\n",
    "# We now add a conditional edge\n",
    "workflow.add_conditional_edges(\n",
    "    # First, we define the start node. We use `agent`.\n",
    "    # This means these are the edges taken after the `agent` node is called.\n",
    "    \"agent\",\n",
    "    # Next, we pass in the function that will determine which node is called next.\n",
    "    should_continue,\n",
    "    path_map=[\"ask_human\", \"action\", END],\n",
    ")\n",
    "\n",
    "# We now add a normal edge from `tools` to `agent`.\n",
    "# This means that after `tools` is called, `agent` node is called next.\n",
    "workflow.add_edge(\"action\", \"agent\")\n",
    "\n",
    "# After we get back the human response, we go back to the agent\n",
    "workflow.add_edge(\"ask_human\", \"agent\")\n",
    "\n",
    "# Set up memory\n",
    "from langgraph.checkpoint.memory import InMemorySaver\n",
    "\n",
    "memory = InMemorySaver()\n",
    "\n",
    "# Finally, we compile it!\n",
    "# This compiles it into a LangChain Runnable,\n",
    "# meaning you can use it as you would any other runnable\n",
    "app = workflow.compile(checkpointer=memory)\n",
    "\n",
    "display(Image(app.get_graph().draw_mermaid_png()))"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "2a1b56c5-bd61-4192-8bdb-458a1e9f0159",
   "metadata": {},
   "source": [
    "## Interacting with the Agent\n",
    "\n",
    "We can now interact with the agent. Let's ask it to ask the user where they are, then tell them the weather. \n",
    "\n",
    "This should make it use the `ask_human` tool first, then use the normal tool."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 8,
   "id": "cfd140f0-a5a6-4697-8115-322242f197b5",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "================================\u001b[1m Human Message \u001b[0m=================================\n",
      "\n",
      "Ask the user where they are, then look up the weather there\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "[{'text': \"I'll help you with that. Let me first ask the user about their location.\", 'type': 'text'}, {'id': 'toolu_012Z9yyZjvH8xKgMShgwpQZ9', 'input': {'question': 'Where are you located?'}, 'name': 'AskHuman', 'type': 'tool_use'}]\n",
      "Tool Calls:\n",
      "  AskHuman (toolu_012Z9yyZjvH8xKgMShgwpQZ9)\n",
      " Call ID: toolu_012Z9yyZjvH8xKgMShgwpQZ9\n",
      "  Args:\n",
      "    question: Where are you located?\n"
     ]
    }
   ],
   "source": [
    "config = {\"configurable\": {\"thread_id\": \"2\"}}\n",
    "for event in app.stream(\n",
    "    {\n",
    "        \"messages\": [\n",
    "            (\n",
    "                \"user\",\n",
    "                \"Ask the user where they are, then look up the weather there\",\n",
    "            )\n",
    "        ]\n",
    "    },\n",
    "    config,\n",
    "    stream_mode=\"values\",\n",
    "):\n",
    "    if \"messages\" in event:\n",
    "        event[\"messages\"][-1].pretty_print()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 9,
   "id": "924a30ea-94c0-468e-90fe-47eb9c08584d",
   "metadata": {},
   "outputs": [
    {
     "data": {
      "text/plain": [
       "('ask_human',)"
      ]
     },
     "execution_count": 9,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "app.get_state(config).next"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "6a30c9fb-2a40-45cc-87ba-406c11c9f0cf",
   "metadata": {},
   "source": [
    "You can see that our graph got interrupted inside the `ask_human` node, which is now waiting for a `location` to be provided. We can provide this value by invoking the graph with a `Command(resume=\"<location>\")` input:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 10,
   "id": "a9f599b5-1a55-406b-a76b-f52b3ca06975",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "[{'text': \"I'll help you with that. Let me first ask the user about their location.\", 'type': 'text'}, {'id': 'toolu_012Z9yyZjvH8xKgMShgwpQZ9', 'input': {'question': 'Where are you located?'}, 'name': 'AskHuman', 'type': 'tool_use'}]\n",
      "Tool Calls:\n",
      "  AskHuman (toolu_012Z9yyZjvH8xKgMShgwpQZ9)\n",
      " Call ID: toolu_012Z9yyZjvH8xKgMShgwpQZ9\n",
      "  Args:\n",
      "    question: Where are you located?\n",
      "=================================\u001b[1m Tool Message \u001b[0m=================================\n",
      "\n",
      "san francisco\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "[{'text': \"Now I'll search for the weather in San Francisco.\", 'type': 'text'}, {'id': 'toolu_01QrWBCDouvBuJPZa4veepLw', 'input': {'query': 'current weather in san francisco'}, 'name': 'search', 'type': 'tool_use'}]\n",
      "Tool Calls:\n",
      "  search (toolu_01QrWBCDouvBuJPZa4veepLw)\n",
      " Call ID: toolu_01QrWBCDouvBuJPZa4veepLw\n",
      "  Args:\n",
      "    query: current weather in san francisco\n",
      "=================================\u001b[1m Tool Message \u001b[0m=================================\n",
      "Name: search\n",
      "\n",
      "I looked up: current weather in san francisco. Result: It's sunny in San Francisco, but you better look out if you're a Gemini 😈.\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "Based on the search results, it's currently sunny in San Francisco. Would you like more specific weather details?\n"
     ]
    }
   ],
   "source": [
    "for event in app.stream(\n",
    "    # highlight-next-line\n",
    "    Command(resume=\"san francisco\"),\n",
    "    config,\n",
    "    stream_mode=\"values\",\n",
    "):\n",
    "    if \"messages\" in event:\n",
    "        event[\"messages\"][-1].pretty_print()"
   ]
  }
 ],
 "metadata": {
  "kernelspec": {
   "display_name": "Python 3 (ipykernel)",
   "language": "python",
   "name": "python3"
  },
  "language_info": {
   "codemirror_mode": {
    "name": "ipython",
    "version": 3
   },
   "file_extension": ".py",
   "mimetype": "text/x-python",
   "name": "python",
   "nbconvert_exporter": "python",
   "pygments_lexer": "ipython3",
   "version": "3.11.4"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 5
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/assets/410e0089-2ab8-46bb-a61a-827187fd46b3.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/assets/autogen-output.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/assets/d38d1f2b-0f4c-4707-b531-a3c749de987f.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/assets/graph_api_image_1.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/assets/graph_api_image_10.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/assets/graph_api_image_11.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/assets/graph_api_image_2.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/assets/graph_api_image_3.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/assets/graph_api_image_4.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/assets/graph_api_image_5.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/assets/graph_api_image_6.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/assets/graph_api_image_7.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/assets/graph_api_image_8.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/assets/graph_api_image_9.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/assets/human_in_loop_parallel.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/auth/custom_auth.md
```md
# Add custom authentication

!!! tip "Prerequisites"

    This guide assumes familiarity with the following concepts:

      *  [**Authentication & Access Control**](../../concepts/auth.md)
      *  [**LangGraph Platform**](../../concepts/langgraph_platform.md)

    For a more guided walkthrough, see [**setting up custom authentication**](../../tutorials/auth/getting_started.md) tutorial.

???+ note "Support by deployment type"

    Custom auth is supported for all deployments in the **managed LangGraph Platform**, as well as **Enterprise** self-hosted plans.

This guide shows how to add custom authentication to your LangGraph Platform application. This guide applies to both LangGraph Platform and self-hosted deployments. It does not apply to isolated usage of the LangGraph open source library in your own custom server.

!!! note

    Custom auth is supported for all **managed LangGraph Platform** deployments, as well as **Enterprise** self-hosted plans.

## Add custom authentication to your deployment

To leverage custom authentication and access user-level metadata in your deployments, set up custom authentication to automatically populate the `config["configurable"]["langgraph_auth_user"]` object through a custom authentication handler. You can then access this object in your graph with the `langgraph_auth_user` key to [allow an agent to perform authenticated actions on behalf of the user](#enable-agent-authentication).

:::python

1.  Implement authentication:

    !!! note

        Without a custom `@auth.authenticate` handler, LangGraph sees only the API-key owner (usually the developer), so requests aren’t scoped to individual end-users. To propagate custom tokens, you must implement your own handler.

    ```python
    from langgraph_sdk import Auth
    import requests

    auth = Auth()

    def is_valid_key(api_key: str) -> bool:
        is_valid = # your API key validation logic
        return is_valid

    @auth.authenticate # (1)!
    async def authenticate(headers: dict) -> Auth.types.MinimalUserDict:
        api_key = headers.get("x-api-key")
        if not api_key or not is_valid_key(api_key):
            raise Auth.exceptions.HTTPException(status_code=401, detail="Invalid API key")

        # Fetch user-specific tokens from your secret store
        user_tokens = await fetch_user_tokens(api_key)

        return { # (2)!
            "identity": api_key,  #  fetch user ID from LangSmith
            "github_token" : user_tokens.github_token
            "jira_token" : user_tokens.jira_token
            # ... custom fields/secrets here
        }
    ```

    1. This handler receives the request (headers, etc.), validates the user, and returns a dictionary with at least an identity field.
    2. You can add any custom fields you want (e.g., OAuth tokens, roles, org IDs, etc.).

2.  In your `langgraph.json`, add the path to your auth file:

    ```json hl_lines="7-9"
    {
      "dependencies": ["."],
      "graphs": {
        "agent": "./agent.py:graph"
      },
      "env": ".env",
      "auth": {
        "path": "./auth.py:my_auth"
      }
    }
    ```

3.  Once you've set up authentication in your server, requests must include the required authorization information based on your chosen scheme. Assuming you are using JWT token authentication, you could access your deployments using any of the following methods:

    === "Python Client"

        ```python
        from langgraph_sdk import get_client

        my_token = "your-token" # In practice, you would generate a signed token with your auth provider
        client = get_client(
            url="http://localhost:2024",
            headers={"Authorization": f"Bearer {my_token}"}
        )
        threads = await client.threads.search()
        ```

    === "Python RemoteGraph"

        ```python
        from langgraph.pregel.remote import RemoteGraph

        my_token = "your-token" # In practice, you would generate a signed token with your auth provider
        remote_graph = RemoteGraph(
            "agent",
            url="http://localhost:2024",
            headers={"Authorization": f"Bearer {my_token}"}
        )
        threads = await remote_graph.ainvoke(...)
        ```
        ```python
        from langgraph.pregel.remote import RemoteGraph

        my_token = "your-token" # In practice, you would generate a signed token with your auth provider
        remote_graph = RemoteGraph(
            "agent",
            url="http://localhost:2024",
            headers={"Authorization": f"Bearer {my_token}"}
        )
        threads = await remote_graph.ainvoke(...)
        ```

    === "CURL"

        ```bash
        curl -H "Authorization: Bearer ${your-token}" http://localhost:2024/threads
        ```

## Enable agent authentication

After [authentication](#add-custom-authentication-to-your-deployment), the platform creates a special configuration object (`config`) that is passed to LangGraph Platform deployment. This object contains information about the current user, including any custom fields you return from your `@auth.authenticate` handler.

To allow an agent to perform authenticated actions on behalf of the user, access this object in your graph with the `langgraph_auth_user` key:

```python
def my_node(state, config):
    user_config = config["configurable"].get("langgraph_auth_user")
    # token was resolved during the @auth.authenticate function
    token = user_config.get("github_token","")
    ...
```

!!! note

    Fetch user credentials from a secure secret store. Storing secrets in graph state is not recommended.

### Authorizing a Studio user

By default, if you add custom authorization on your resources, this will also apply to interactions made from the Studio. If you want, you can handle logged-in Studio users differently by checking [is_studio_user()](../../reference/functions/sdk_auth.isStudioUser.html).

!!! note
    `is_studio_user` was added in version 0.1.73 of the langgraph-sdk. If you're on an older version, you can still check whether `isinstance(ctx.user, StudioUser)`.

```python
from langgraph_sdk.auth import is_studio_user, Auth
auth = Auth()

# ... Setup authenticate, etc.

@auth.on
async def add_owner(
    ctx: Auth.types.AuthContext,
    value: dict  # The payload being sent to this access method
) -> dict:  # Returns a filter dict that restricts access to resources
    if is_studio_user(ctx.user):
        return {}

    filters = {"owner": ctx.user.identity}
    metadata = value.setdefault("metadata", {})
    metadata.update(filters)
    return filters
```

Only use this if you want to permit developer access to a graph deployed on the managed LangGraph Platform SaaS.

:::

:::js

1.  Implement authentication:

    !!! note

        Without a custom `authenticate` handler, LangGraph sees only the API-key owner (usually the developer), so requests aren’t scoped to individual end-users. To propagate custom tokens, you must implement your own handler.

    ```typescript
    import { Auth, HTTPException } from "@langchain/langgraph-sdk/auth";

    const auth = new Auth()
      .authenticate(async (request) => {
        const authorization = request.headers.get("Authorization");
        const token = authorization?.split(" ")[1]; // "Bearer <token>"
        if (!token) {
          throw new HTTPException(401, "No token provided");
        }
        try {
          const user = await verifyToken(token);
          return user;
        } catch (error) {
          throw new HTTPException(401, "Invalid token");
        }
      })
      // Add authorization rules to actually control access to resources
      .on("*", async ({ user, value }) => {
        const filters = { owner: user.identity };
        const metadata = value.metadata ?? {};
        metadata.update(filters);
        return filters;
      })
      // Assumes you organize information in store like (user_id, resource_type, resource_id)
      .on("store", async ({ user, value }) => {
        const namespace = value.namespace;
        if (namespace[0] !== user.identity) {
          throw new HTTPException(403, "Not authorized");
        }
      });
    ```

    1. This handler receives the request (headers, etc.), validates the user, and returns an object with at least an identity field.
    2. You can add any custom fields you want (e.g., OAuth tokens, roles, org IDs, etc.).

2.  In your `langgraph.json`, add the path to your auth file:

    ```json hl_lines="7-9"
    {
      "dependencies": ["."],
      "graphs": {
        "agent": "./agent.ts:graph"
      },
      "env": ".env",
      "auth": {
        "path": "./auth.ts:my_auth"
      }
    }
    ```

3.  Once you've set up authentication in your server, requests must include the required authorization information based on your chosen scheme. Assuming you are using JWT token authentication, you could access your deployments using any of the following methods:

    === "SDK Client"

        ```javascript
        import { Client } from "@langchain/langgraph-sdk";

        const my_token = "your-token"; // In practice, you would generate a signed token with your auth provider
        const client = new Client({
          apiUrl: "http://localhost:2024",
          defaultHeaders: { Authorization: `Bearer ${my_token}` },
        });
        const threads = await client.threads.search();
        ```

    === "RemoteGraph"

        ```javascript
        import { RemoteGraph } from "@langchain/langgraph/remote";

        const my_token = "your-token"; // In practice, you would generate a signed token with your auth provider
        const remoteGraph = new RemoteGraph({
        graphId: "agent",
          url: "http://localhost:2024",
          headers: { Authorization: `Bearer ${my_token}` },
        });
        const threads = await remoteGraph.invoke(...);
        ```

    === "CURL"

        ```bash
        curl -H "Authorization: Bearer ${your-token}" http://localhost:2024/threads
        ```

## Enable agent authentication

After [authentication](#add-custom-authentication-to-your-deployment), the platform creates a special configuration object (`config`) that is passed to LangGraph Platform deployment. This object contains information about the current user, including any custom fields you return from your `authenticate` handler.

To allow an agent to perform authenticated actions on behalf of the user, access this object in your graph with the `langgraph_auth_user` key:

```ts
async function myNode(state, config) {
  const userConfig = config["configurable"]["langgraph_auth_user"];
  // token was resolved during the authenticate function
  const token = userConfig["github_token"];
  ...
}
```

!!! note

    Fetch user credentials from a secure secret store. Storing secrets in graph state is not recommended.

:::

## Learn more

- [Authentication & Access Control](../../concepts/auth.md)
- [LangGraph Platform](../../concepts/langgraph_platform.md)
- [Setting up custom authentication tutorial](../../tutorials/auth/getting_started.md)

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/auth/openapi_security.md
```md
# Document API authentication in OpenAPI

This guide shows how to customize the OpenAPI security schema for your LangGraph Platform API documentation. A well-documented security schema helps API consumers understand how to authenticate with your API and even enables automatic client generation. See the [Authentication & Access Control conceptual guide](../../concepts/auth.md) for more details about LangGraph's authentication system.

!!! note "Implementation vs Documentation"

    This guide only covers how to document your security requirements in OpenAPI. To implement the actual authentication logic, see [How to add custom authentication](./custom_auth.md).

This guide applies to all LangGraph Platform deployments (Cloud and self-hosted). It does not apply to usage of the LangGraph open source library if you are not using LangGraph Platform.

## Default Schema

The default security scheme varies by deployment type:

=== "LangGraph Platform"

By default, LangGraph Platform requires a LangSmith API key in the `x-api-key` header:

```yaml
components:
  securitySchemes:
    apiKeyAuth:
      type: apiKey
      in: header
      name: x-api-key
security:
  - apiKeyAuth: []
```

When using one of the LangGraph SDK's, this can be inferred from environment variables.

=== "Self-hosted"

By default, self-hosted deployments have no security scheme. This means they are to be deployed only on a secured network or with authentication. To add custom authentication, see [How to add custom authentication](./custom_auth.md).

## Custom Security Schema

To customize the security schema in your OpenAPI documentation, add an `openapi` field to your `auth` configuration in `langgraph.json`. Remember that this only updates the API documentation - you must also implement the corresponding authentication logic as shown in [How to add custom authentication](./custom_auth.md).

Note that LangGraph Platform does not provide authentication endpoints - you'll need to handle user authentication in your client application and pass the resulting credentials to the LangGraph API.

:::python
=== "OAuth2 with Bearer Token"

    ```json
    {
      "auth": {
        "path": "./auth.py:my_auth",  // Implement auth logic here
        "openapi": {
          "securitySchemes": {
            "OAuth2": {
              "type": "oauth2",
              "flows": {
                "implicit": {
                  "authorizationUrl": "https://your-auth-server.com/oauth/authorize",
                  "scopes": {
                    "me": "Read information about the current user",
                    "threads": "Access to create and manage threads"
                  }
                }
              }
            }
          },
          "security": [
            {"OAuth2": ["me", "threads"]}
          ]
        }
      }
    }
    ```

=== "API Key"

    ```json
    {
      "auth": {
        "path": "./auth.py:my_auth",  // Implement auth logic here
        "openapi": {
          "securitySchemes": {
            "apiKeyAuth": {
              "type": "apiKey",
              "in": "header",
              "name": "X-API-Key"
            }
          },
          "security": [
            {"apiKeyAuth": []}
          ]
        }
      }
    }
    ```

:::

:::js
=== "OAuth2 with Bearer Token"

    ```json
    {
      "auth": {
        "path": "./auth.ts:my_auth",  // Implement auth logic here
        "openapi": {
          "securitySchemes": {
            "OAuth2": {
              "type": "oauth2",
              "flows": {
                "implicit": {
                  "authorizationUrl": "https://your-auth-server.com/oauth/authorize",
                  "scopes": {
                    "me": "Read information about the current user",
                    "threads": "Access to create and manage threads"
                  }
                }
              }
            }
          },
          "security": [
            {"OAuth2": ["me", "threads"]}
          ]
        }
      }
    }
    ```

=== "API Key"

    ```json
    {
      "auth": {
        "path": "./auth.ts:my_auth",  // Implement auth logic here
        "openapi": {
          "securitySchemes": {
            "apiKeyAuth": {
              "type": "apiKey",
              "in": "header",
              "name": "X-API-Key"
            }
          },
          "security": [
            {"apiKeyAuth": []}
          ]
        }
      }
    }
    ```

:::

## Testing

After updating your configuration:

1. Deploy your application
2. Visit `/docs` to see the updated OpenAPI documentation
3. Try out the endpoints using credentials from your authentication server (make sure you've implemented the authentication logic first)

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/memory/add-memory.md
```md
# Add and manage memory

AI applications need [memory](../../concepts/memory.md) to share context across multiple interactions. In LangGraph, you can add two types of memory:

- [Add short-term memory](#add-short-term-memory) as a part of your agent's [state](../../concepts/low_level.md#state) to enable multi-turn conversations.
- [Add long-term memory](#add-long-term-memory) to store user-specific or application-level data across sessions.

## Add short-term memory

**Short-term** memory (thread-level [persistence](../../concepts/persistence.md)) enables agents to track multi-turn conversations. To add short-term memory:

:::python

```python
# highlight-next-line
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph

# highlight-next-line
checkpointer = InMemorySaver()

builder = StateGraph(...)
# highlight-next-line
graph = builder.compile(checkpointer=checkpointer)

graph.invoke(
    {"messages": [{"role": "user", "content": "hi! i am Bob"}]},
    # highlight-next-line
    {"configurable": {"thread_id": "1"}},
)
```

:::

:::js

```typescript
import { MemorySaver, StateGraph } from "@langchain/langgraph";

const checkpointer = new MemorySaver();

const builder = new StateGraph(...);
const graph = builder.compile({ checkpointer });

await graph.invoke(
  { messages: [{ role: "user", content: "hi! i am Bob" }] },
  { configurable: { thread_id: "1" } }
);
```

:::

### Use in production

In production, use a checkpointer backed by a database:

:::python

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
# highlight-next-line
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    builder = StateGraph(...)
    # highlight-next-line
    graph = builder.compile(checkpointer=checkpointer)
```

:::

:::js

```typescript
import { PostgresSaver } from "@langchain/langgraph-checkpoint-postgres";

const DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable";
const checkpointer = PostgresSaver.fromConnString(DB_URI);

const builder = new StateGraph(...);
const graph = builder.compile({ checkpointer });
```

:::

??? example "Example: using Postgres checkpointer"

    :::python
    ```
    pip install -U "psycopg[binary,pool]" langgraph langgraph-checkpoint-postgres
    ```

    !!! Setup
        You need to call `checkpointer.setup()` the first time you're using Postgres checkpointer

    === "Sync"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.postgres import PostgresSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
        # highlight-next-line
        with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
            # checkpointer.setup()

            def call_model(state: MessagesState):
                response = model.invoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

    === "Async"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
        # highlight-next-line
        async with AsyncPostgresSaver.from_conn_string(DB_URI) as checkpointer:
            # await checkpointer.setup()

            async def call_model(state: MessagesState):
                response = await model.ainvoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```
    :::

    :::js
    ```
    npm install @langchain/langgraph-checkpoint-postgres
    ```

    !!! Setup
        You need to call `checkpointer.setup()` the first time you're using Postgres checkpointer

    ```typescript
    import { ChatAnthropic } from "@langchain/anthropic";
    import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";
    import { PostgresSaver } from "@langchain/langgraph-checkpoint-postgres";

    const model = new ChatAnthropic({ model: "claude-3-5-haiku-20241022" });

    const DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable";
    const checkpointer = PostgresSaver.fromConnString(DB_URI);
    // await checkpointer.setup();

    const builder = new StateGraph(MessagesZodState)
      .addNode("call_model", async (state) => {
        const response = await model.invoke(state.messages);
        return { messages: [response] };
      })
      .addEdge(START, "call_model");

    const graph = builder.compile({ checkpointer });

    const config = {
      configurable: {
        thread_id: "1"
      }
    };

    for await (const chunk of await graph.stream(
      { messages: [{ role: "user", content: "hi! I'm bob" }] },
      { ...config, streamMode: "values" }
    )) {
      console.log(chunk.messages.at(-1)?.content);
    }

    for await (const chunk of await graph.stream(
      { messages: [{ role: "user", content: "what's my name?" }] },
      { ...config, streamMode: "values" }
    )) {
      console.log(chunk.messages.at(-1)?.content);
    }
    ```
    :::

:::python
??? example "Example: using [MongoDB](https://pypi.org/project/langgraph-checkpoint-mongodb/) checkpointer"

    ```
    pip install -U pymongo langgraph langgraph-checkpoint-mongodb
    ```

    !!! note "Setup"

        To use the MongoDB checkpointer, you will need a MongoDB cluster. Follow [this guide](https://www.mongodb.com/docs/guides/atlas/cluster/) to create a cluster if you don't already have one.

    === "Sync"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.mongodb import MongoDBSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "localhost:27017"
        # highlight-next-line
        with MongoDBSaver.from_conn_string(DB_URI) as checkpointer:

            def call_model(state: MessagesState):
                response = model.invoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

    === "Async"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.mongodb.aio import AsyncMongoDBSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "localhost:27017"
        # highlight-next-line
        async with AsyncMongoDBSaver.from_conn_string(DB_URI) as checkpointer:

            async def call_model(state: MessagesState):
                response = await model.ainvoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

??? example "Example: using [Redis](https://pypi.org/project/langgraph-checkpoint-redis/) checkpointer"

    ```
    pip install -U langgraph langgraph-checkpoint-redis
    ```

    !!! Setup
        You need to call `checkpointer.setup()` the first time you're using Redis checkpointer


    === "Sync"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.redis import RedisSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "redis://localhost:6379"
        # highlight-next-line
        with RedisSaver.from_conn_string(DB_URI) as checkpointer:
            # checkpointer.setup()

            def call_model(state: MessagesState):
                response = model.invoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

    === "Async"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.redis.aio import AsyncRedisSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "redis://localhost:6379"
        # highlight-next-line
        async with AsyncRedisSaver.from_conn_string(DB_URI) as checkpointer:
            # await checkpointer.asetup()

            async def call_model(state: MessagesState):
                response = await model.ainvoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

:::

### Use in subgraphs

If your graph contains [subgraphs](../../concepts/subgraphs.md), you only need to provide the checkpointer when compiling the parent graph. LangGraph will automatically propagate the checkpointer to the child subgraphs.

:::python

```python
from langgraph.graph import START, StateGraph
from langgraph.checkpoint.memory import InMemorySaver
from typing import TypedDict

class State(TypedDict):
    foo: str

# Subgraph

def subgraph_node_1(state: State):
    return {"foo": state["foo"] + "bar"}

subgraph_builder = StateGraph(State)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
# highlight-next-line
subgraph = subgraph_builder.compile()

# Parent graph

builder = StateGraph(State)
# highlight-next-line
builder.add_node("node_1", subgraph)
builder.add_edge(START, "node_1")

checkpointer = InMemorySaver()
# highlight-next-line
graph = builder.compile(checkpointer=checkpointer)
```

:::

:::js

```typescript
import { StateGraph, START, MemorySaver } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({ foo: z.string() });

const subgraphBuilder = new StateGraph(State)
  .addNode("subgraph_node_1", (state) => {
    return { foo: state.foo + "bar" };
  })
  .addEdge(START, "subgraph_node_1");
const subgraph = subgraphBuilder.compile();

const builder = new StateGraph(State)
  .addNode("node_1", subgraph)
  .addEdge(START, "node_1");

const checkpointer = new MemorySaver();
const graph = builder.compile({ checkpointer });
```

:::

If you want the subgraph to have its own memory, you can compile it with the appropriate checkpointer option. This is useful in [multi-agent](../../concepts/multi_agent.md) systems, if you want agents to keep track of their internal message histories.

:::python

```python
subgraph_builder = StateGraph(...)
# highlight-next-line
subgraph = subgraph_builder.compile(checkpointer=True)
```

:::

:::js

```typescript
const subgraphBuilder = new StateGraph(...);
// highlight-next-line
const subgraph = subgraphBuilder.compile({ checkpointer: true });
```

:::

### Read short-term memory in tools { #read-short-term }

LangGraph allows agents to access their short-term memory (state) inside the tools.

:::python

```python
from typing import Annotated
from langgraph.prebuilt import InjectedState, create_react_agent

class CustomState(AgentState):
    # highlight-next-line
    user_id: str

def get_user_info(
    # highlight-next-line
    state: Annotated[CustomState, InjectedState]
) -> str:
    """Look up user info."""
    # highlight-next-line
    user_id = state["user_id"]
    return "User is John Smith" if user_id == "user_123" else "Unknown user"

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[get_user_info],
    # highlight-next-line
    state_schema=CustomState,
)

agent.invoke({
    "messages": "look up user information",
    # highlight-next-line
    "user_id": "user_123"
})
```

:::

:::js

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";
import {
  MessagesZodState,
  LangGraphRunnableConfig,
} from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";

const CustomState = z.object({
  messages: MessagesZodState.shape.messages,
  userId: z.string(),
});

const getUserInfo = tool(
  async (_, config: LangGraphRunnableConfig) => {
    const userId = config.configurable?.userId;
    return userId === "user_123" ? "User is John Smith" : "Unknown user";
  },
  {
    name: "get_user_info",
    description: "Look up user info.",
    schema: z.object({}),
  }
);

const agent = createReactAgent({
  llm: model,
  tools: [getUserInfo],
  stateSchema: CustomState,
});

await agent.invoke({
  messages: [{ role: "user", content: "look up user information" }],
  userId: "user_123",
});
```

:::

See the [Context](../../agents/context.md) guide for more information.

### Write short-term memory from tools { #write-short-term }

To modify the agent's short-term memory (state) during execution, you can return state updates directly from the tools. This is useful for persisting intermediate results or making information accessible to subsequent tools or prompts.

:::python

```python
from typing import Annotated
from langchain_core.tools import InjectedToolCallId
from langchain_core.runnables import RunnableConfig
from langchain_core.messages import ToolMessage
from langgraph.prebuilt import InjectedState, create_react_agent
from langgraph.prebuilt.chat_agent_executor import AgentState
from langgraph.types import Command

class CustomState(AgentState):
    # highlight-next-line
    user_name: str

def update_user_info(
    tool_call_id: Annotated[str, InjectedToolCallId],
    config: RunnableConfig
) -> Command:
    """Look up and update user info."""
    user_id = config["configurable"].get("user_id")
    name = "John Smith" if user_id == "user_123" else "Unknown user"
    # highlight-next-line
    return Command(update={
        # highlight-next-line
        "user_name": name,
        # update the message history
        "messages": [
            ToolMessage(
                "Successfully looked up user information",
                tool_call_id=tool_call_id
            )
        ]
    })

def greet(
    # highlight-next-line
    state: Annotated[CustomState, InjectedState]
) -> str:
    """Use this to greet the user once you found their info."""
    user_name = state["user_name"]
    return f"Hello {user_name}!"

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[update_user_info, greet],
    # highlight-next-line
    state_schema=CustomState
)

agent.invoke(
    {"messages": [{"role": "user", "content": "greet the user"}]},
    # highlight-next-line
    config={"configurable": {"user_id": "user_123"}}
)
```

:::

:::js

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";
import {
  MessagesZodState,
  LangGraphRunnableConfig,
  Command,
} from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";

const CustomState = z.object({
  messages: MessagesZodState.shape.messages,
  userName: z.string().optional(),
});

const updateUserInfo = tool(
  async (_, config: LangGraphRunnableConfig) => {
    const userId = config.configurable?.userId;
    const name = userId === "user_123" ? "John Smith" : "Unknown user";
    return new Command({
      update: {
        userName: name,
        // update the message history
        messages: [
          {
            role: "tool",
            content: "Successfully looked up user information",
            tool_call_id: config.toolCall?.id,
          },
        ],
      },
    });
  },
  {
    name: "update_user_info",
    description: "Look up and update user info.",
    schema: z.object({}),
  }
);

const greet = tool(
  async (_, config: LangGraphRunnableConfig) => {
    const userName = config.configurable?.userName;
    return `Hello ${userName}!`;
  },
  {
    name: "greet",
    description: "Use this to greet the user once you found their info.",
    schema: z.object({}),
  }
);

const agent = createReactAgent({
  llm: model,
  tools: [updateUserInfo, greet],
  stateSchema: CustomState,
});

await agent.invoke(
  { messages: [{ role: "user", content: "greet the user" }] },
  { configurable: { userId: "user_123" } }
);
```

:::

## Add long-term memory

Use long-term memory to store user-specific or application-specific data across conversations.

:::python

```python
# highlight-next-line
from langgraph.store.memory import InMemoryStore
from langgraph.graph import StateGraph

# highlight-next-line
store = InMemoryStore()

builder = StateGraph(...)
# highlight-next-line
graph = builder.compile(store=store)
```

:::

:::js

```typescript
import { InMemoryStore, StateGraph } from "@langchain/langgraph";

const store = new InMemoryStore();

const builder = new StateGraph(...);
const graph = builder.compile({ store });
```

:::

### Use in production

In production, use a store backed by a database:

:::python

```python
from langgraph.store.postgres import PostgresStore

DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
# highlight-next-line
with PostgresStore.from_conn_string(DB_URI) as store:
    builder = StateGraph(...)
    # highlight-next-line
    graph = builder.compile(store=store)
```

:::

:::js

```typescript
import { PostgresStore } from "@langchain/langgraph-checkpoint-postgres";

const DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable";
const store = PostgresStore.fromConnString(DB_URI);

const builder = new StateGraph(...);
const graph = builder.compile({ store });
```

:::

??? example "Example: using Postgres store"

    :::python
    ```
    pip install -U "psycopg[binary,pool]" langgraph langgraph-checkpoint-postgres
    ```

    !!! Setup
        You need to call `store.setup()` the first time you're using Postgres store

    === "Sync"

        ```python
        from langchain_core.runnables import RunnableConfig
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.postgres import PostgresSaver
        # highlight-next-line
        from langgraph.store.postgres import PostgresStore
        from langgraph.store.base import BaseStore

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"

        with (
            # highlight-next-line
            PostgresStore.from_conn_string(DB_URI) as store,
            PostgresSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            # store.setup()
            # checkpointer.setup()

            def call_model(
                state: MessagesState,
                config: RunnableConfig,
                *,
                # highlight-next-line
                store: BaseStore,
            ):
                user_id = config["configurable"]["user_id"]
                namespace = ("memories", user_id)
                # highlight-next-line
                memories = store.search(namespace, query=str(state["messages"][-1].content))
                info = "\n".join([d.value["data"] for d in memories])
                system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

                # Store new memories if the user asks the model to remember
                last_message = state["messages"][-1]
                if "remember" in last_message.content.lower():
                    memory = "User name is Bob"
                    # highlight-next-line
                    store.put(namespace, str(uuid.uuid4()), {"data": memory})

                response = model.invoke(
                    [{"role": "system", "content": system_msg}] + state["messages"]
                )
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                # highlight-next-line
                store=store,
            )

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1",
                    # highlight-next-line
                    "user_id": "1",
                }
            }
            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "2",
                    "user_id": "1",
                }
            }

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "what is my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()
        ```

    === "Async"

        ```python
        from langchain_core.runnables import RunnableConfig
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
        # highlight-next-line
        from langgraph.store.postgres.aio import AsyncPostgresStore
        from langgraph.store.base import BaseStore

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"

        async with (
            # highlight-next-line
            AsyncPostgresStore.from_conn_string(DB_URI) as store,
            AsyncPostgresSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            # await store.setup()
            # await checkpointer.setup()

            async def call_model(
                state: MessagesState,
                config: RunnableConfig,
                *,
                # highlight-next-line
                store: BaseStore,
            ):
                user_id = config["configurable"]["user_id"]
                namespace = ("memories", user_id)
                # highlight-next-line
                memories = await store.asearch(namespace, query=str(state["messages"][-1].content))
                info = "\n".join([d.value["data"] for d in memories])
                system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

                # Store new memories if the user asks the model to remember
                last_message = state["messages"][-1]
                if "remember" in last_message.content.lower():
                    memory = "User name is Bob"
                    # highlight-next-line
                    await store.aput(namespace, str(uuid.uuid4()), {"data": memory})

                response = await model.ainvoke(
                    [{"role": "system", "content": system_msg}] + state["messages"]
                )
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                # highlight-next-line
                store=store,
            )

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1",
                    # highlight-next-line
                    "user_id": "1",
                }
            }
            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "2",
                    "user_id": "1",
                }
            }

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "what is my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()
        ```
    :::

    :::js
    ```
    npm install @langchain/langgraph-checkpoint-postgres
    ```

    !!! Setup
        You need to call `store.setup()` the first time you're using Postgres store

    ```typescript
    import { ChatAnthropic } from "@langchain/anthropic";
    import { StateGraph, MessagesZodState, START, LangGraphRunnableConfig } from "@langchain/langgraph";
    import { PostgresSaver, PostgresStore } from "@langchain/langgraph-checkpoint-postgres";
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";

    const model = new ChatAnthropic({ model: "claude-3-5-haiku-20241022" });

    const DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable";

    const store = PostgresStore.fromConnString(DB_URI);
    const checkpointer = PostgresSaver.fromConnString(DB_URI);
    // await store.setup();
    // await checkpointer.setup();

    const callModel = async (
      state: z.infer<typeof MessagesZodState>,
      config: LangGraphRunnableConfig,
    ) => {
      const userId = config.configurable?.userId;
      const namespace = ["memories", userId];
      const memories = await config.store?.search(namespace, { query: state.messages.at(-1)?.content });
      const info = memories?.map(d => d.value.data).join("\n") || "";
      const systemMsg = `You are a helpful assistant talking to the user. User info: ${info}`;

      // Store new memories if the user asks the model to remember
      const lastMessage = state.messages.at(-1);
      if (lastMessage?.content?.toLowerCase().includes("remember")) {
        const memory = "User name is Bob";
        await config.store?.put(namespace, uuidv4(), { data: memory });
      }

      const response = await model.invoke([
        { role: "system", content: systemMsg },
        ...state.messages
      ]);
      return { messages: [response] };
    };

    const builder = new StateGraph(MessagesZodState)
      .addNode("call_model", callModel)
      .addEdge(START, "call_model");

    const graph = builder.compile({
      checkpointer,
      store,
    });

    const config = {
      configurable: {
        thread_id: "1",
        userId: "1",
      }
    };

    for await (const chunk of await graph.stream(
      { messages: [{ role: "user", content: "Hi! Remember: my name is Bob" }] },
      { ...config, streamMode: "values" }
    )) {
      console.log(chunk.messages.at(-1)?.content);
    }

    const config2 = {
      configurable: {
        thread_id: "2",
        userId: "1",
      }
    };

    for await (const chunk of await graph.stream(
      { messages: [{ role: "user", content: "what is my name?" }] },
      { ...config2, streamMode: "values" }
    )) {
      console.log(chunk.messages.at(-1)?.content);
    }
    ```
    :::

:::python
??? example "Example: using [Redis](https://pypi.org/project/langgraph-checkpoint-redis/) store"

    ```
    pip install -U langgraph langgraph-checkpoint-redis
    ```

    !!! Setup
        You need to call `store.setup()` the first time you're using Redis store


    === "Sync"

        ```python
        from langchain_core.runnables import RunnableConfig
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.redis import RedisSaver
        # highlight-next-line
        from langgraph.store.redis import RedisStore
        from langgraph.store.base import BaseStore

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "redis://localhost:6379"

        with (
            # highlight-next-line
            RedisStore.from_conn_string(DB_URI) as store,
            RedisSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            store.setup()
            checkpointer.setup()

            def call_model(
                state: MessagesState,
                config: RunnableConfig,
                *,
                # highlight-next-line
                store: BaseStore,
            ):
                user_id = config["configurable"]["user_id"]
                namespace = ("memories", user_id)
                # highlight-next-line
                memories = store.search(namespace, query=str(state["messages"][-1].content))
                info = "\n".join([d.value["data"] for d in memories])
                system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

                # Store new memories if the user asks the model to remember
                last_message = state["messages"][-1]
                if "remember" in last_message.content.lower():
                    memory = "User name is Bob"
                    # highlight-next-line
                    store.put(namespace, str(uuid.uuid4()), {"data": memory})

                response = model.invoke(
                    [{"role": "system", "content": system_msg}] + state["messages"]
                )
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                # highlight-next-line
                store=store,
            )

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1",
                    # highlight-next-line
                    "user_id": "1",
                }
            }
            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "2",
                    "user_id": "1",
                }
            }

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "what is my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()
        ```

    === "Async"

        ```python
        from langchain_core.runnables import RunnableConfig
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.redis.aio import AsyncRedisSaver
        # highlight-next-line
        from langgraph.store.redis.aio import AsyncRedisStore
        from langgraph.store.base import BaseStore

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "redis://localhost:6379"

        async with (
            # highlight-next-line
            AsyncRedisStore.from_conn_string(DB_URI) as store,
            AsyncRedisSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            # await store.setup()
            # await checkpointer.asetup()

            async def call_model(
                state: MessagesState,
                config: RunnableConfig,
                *,
                # highlight-next-line
                store: BaseStore,
            ):
                user_id = config["configurable"]["user_id"]
                namespace = ("memories", user_id)
                # highlight-next-line
                memories = await store.asearch(namespace, query=str(state["messages"][-1].content))
                info = "\n".join([d.value["data"] for d in memories])
                system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

                # Store new memories if the user asks the model to remember
                last_message = state["messages"][-1]
                if "remember" in last_message.content.lower():
                    memory = "User name is Bob"
                    # highlight-next-line
                    await store.aput(namespace, str(uuid.uuid4()), {"data": memory})

                response = await model.ainvoke(
                    [{"role": "system", "content": system_msg}] + state["messages"]
                )
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                # highlight-next-line
                store=store,
            )

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1",
                    # highlight-next-line
                    "user_id": "1",
                }
            }
            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "2",
                    "user_id": "1",
                }
            }

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "what is my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()
        ```

:::

### Read long-term memory in tools { #read-long-term }

:::python

```python title="A tool the agent can use to look up user information"
from langchain_core.runnables import RunnableConfig
from langgraph.config import get_store
from langgraph.prebuilt import create_react_agent
from langgraph.store.memory import InMemoryStore

# highlight-next-line
store = InMemoryStore() # (1)!

# highlight-next-line
store.put(  # (2)!
    ("users",),  # (3)!
    "user_123",  # (4)!
    {
        "name": "John Smith",
        "language": "English",
    } # (5)!
)

def get_user_info(config: RunnableConfig) -> str:
    """Look up user info."""
    # Same as that provided to `create_react_agent`
    # highlight-next-line
    store = get_store() # (6)!
    user_id = config["configurable"].get("user_id")
    # highlight-next-line
    user_info = store.get(("users",), user_id) # (7)!
    return str(user_info.value) if user_info else "Unknown user"

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[get_user_info],
    # highlight-next-line
    store=store # (8)!
)

# Run the agent
agent.invoke(
    {"messages": [{"role": "user", "content": "look up user information"}]},
    # highlight-next-line
    config={"configurable": {"user_id": "user_123"}}
)
```

1. The `InMemoryStore` is a store that stores data in memory. In a production setting, you would typically use a database or other persistent storage. Please review the [store documentation](../../reference/store.md) for more options. If you're deploying with **LangGraph Platform**, the platform will provide a production-ready store for you.
2. For this example, we write some sample data to the store using the `put` method. Please see the @[BaseStore.put] API reference for more details.
3. The first argument is the namespace. This is used to group related data together. In this case, we are using the `users` namespace to group user data.
4. A key within the namespace. This example uses a user ID for the key.
5. The data that we want to store for the given user.
6. The `get_store` function is used to access the store. You can call it from anywhere in your code, including tools and prompts. This function returns the store that was passed to the agent when it was created.
7. The `get` method is used to retrieve data from the store. The first argument is the namespace, and the second argument is the key. This will return a `StoreValue` object, which contains the value and metadata about the value.
8. The `store` is passed to the agent. This enables the agent to access the store when running tools. You can also use the `get_store` function to access the store from anywhere in your code.
   :::

:::js

```typescript title="A tool the agent can use to look up user information"
import { tool } from "@langchain/core/tools";
import { z } from "zod";
import { LangGraphRunnableConfig, InMemoryStore } from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";

const store = new InMemoryStore(); // (1)!

await store.put(
  // (2)!
  ["users"], // (3)!
  "user_123", // (4)!
  {
    name: "John Smith",
    language: "English",
  } // (5)!
);

const getUserInfo = tool(
  async (_, config: LangGraphRunnableConfig) => {
    /**Look up user info.*/
    // Same as that provided to `createReactAgent`
    const store = config.store; // (6)!
    const userId = config.configurable?.userId;
    const userInfo = await store?.get(["users"], userId); // (7)!
    return userInfo?.value ? JSON.stringify(userInfo.value) : "Unknown user";
  },
  {
    name: "get_user_info",
    description: "Look up user info.",
    schema: z.object({}),
  }
);

const agent = createReactAgent({
  llm: model,
  tools: [getUserInfo],
  store, // (8)!
});

// Run the agent
await agent.invoke(
  { messages: [{ role: "user", content: "look up user information" }] },
  { configurable: { userId: "user_123" } }
);
```

1. The `InMemoryStore` is a store that stores data in memory. In a production setting, you would typically use a database or other persistent storage. Please review the [store documentation](../../reference/store.md) for more options. If you're deploying with **LangGraph Platform**, the platform will provide a production-ready store for you.
2. For this example, we write some sample data to the store using the `put` method. Please see the @[BaseStore.put] API reference for more details.
3. The first argument is the namespace. This is used to group related data together. In this case, we are using the `users` namespace to group user data.
4. A key within the namespace. This example uses a user ID for the key.
5. The data that we want to store for the given user.
6. The store is accessible through the config. You can call it from anywhere in your code, including tools and prompts. This function returns the store that was passed to the agent when it was created.
7. The `get` method is used to retrieve data from the store. The first argument is the namespace, and the second argument is the key. This will return a `StoreValue` object, which contains the value and metadata about the value.
8. The `store` is passed to the agent. This enables the agent to access the store when running tools. You can also use the store from the config to access it from anywhere in your code.
   :::

### Write long-term memory from tools { #write-long-term }

:::python

```python title="Example of a tool that updates user information"
from typing_extensions import TypedDict

from langgraph.config import get_store
from langchain_core.runnables import RunnableConfig
from langgraph.prebuilt import create_react_agent
from langgraph.store.memory import InMemoryStore

store = InMemoryStore() # (1)!

class UserInfo(TypedDict): # (2)!
    name: str

def save_user_info(user_info: UserInfo, config: RunnableConfig) -> str: # (3)!
    """Save user info."""
    # Same as that provided to `create_react_agent`
    # highlight-next-line
    store = get_store() # (4)!
    user_id = config["configurable"].get("user_id")
    # highlight-next-line
    store.put(("users",), user_id, user_info) # (5)!
    return "Successfully saved user info."

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[save_user_info],
    # highlight-next-line
    store=store
)

# Run the agent
agent.invoke(
    {"messages": [{"role": "user", "content": "My name is John Smith"}]},
    # highlight-next-line
    config={"configurable": {"user_id": "user_123"}} # (6)!
)

# You can access the store directly to get the value
store.get(("users",), "user_123").value
```

1. The `InMemoryStore` is a store that stores data in memory. In a production setting, you would typically use a database or other persistent storage. Please review the [store documentation](../../reference/store.md) for more options. If you're deploying with **LangGraph Platform**, the platform will provide a production-ready store for you.
2. The `UserInfo` class is a `TypedDict` that defines the structure of the user information. The LLM will use this to format the response according to the schema.
3. The `save_user_info` function is a tool that allows an agent to update user information. This could be useful for a chat application where the user wants to update their profile information.
4. The `get_store` function is used to access the store. You can call it from anywhere in your code, including tools and prompts. This function returns the store that was passed to the agent when it was created.
5. The `put` method is used to store data in the store. The first argument is the namespace, and the second argument is the key. This will store the user information in the store.
6. The `user_id` is passed in the config. This is used to identify the user whose information is being updated.
   :::

:::js

```typescript title="Example of a tool that updates user information"
import { tool } from "@langchain/core/tools";
import { z } from "zod";
import { LangGraphRunnableConfig, InMemoryStore } from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";

const store = new InMemoryStore(); // (1)!

const UserInfo = z.object({
  // (2)!
  name: z.string(),
});

const saveUserInfo = tool(
  async (
    userInfo: z.infer<typeof UserInfo>,
    config: LangGraphRunnableConfig
  ) => {
    // (3)!
    /**Save user info.*/
    // Same as that provided to `createReactAgent`
    const store = config.store; // (4)!
    const userId = config.configurable?.userId;
    await store?.put(["users"], userId, userInfo); // (5)!
    return "Successfully saved user info.";
  },
  {
    name: "save_user_info",
    description: "Save user info.",
    schema: UserInfo,
  }
);

const agent = createReactAgent({
  llm: model,
  tools: [saveUserInfo],
  store,
});

// Run the agent
await agent.invoke(
  { messages: [{ role: "user", content: "My name is John Smith" }] },
  { configurable: { userId: "user_123" } } // (6)!
);

// You can access the store directly to get the value
const result = await store.get(["users"], "user_123");
console.log(result?.value);
```

1. The `InMemoryStore` is a store that stores data in memory. In a production setting, you would typically use a database or other persistent storage. Please review the [store documentation](../../reference/store.md) for more options. If you're deploying with **LangGraph Platform**, the platform will provide a production-ready store for you.
2. The `UserInfo` schema defines the structure of the user information. The LLM will use this to format the response according to the schema.
3. The `saveUserInfo` function is a tool that allows an agent to update user information. This could be useful for a chat application where the user wants to update their profile information.
4. The store is accessible through the config. You can call it from anywhere in your code, including tools and prompts. This function returns the store that was passed to the agent when it was created.
5. The `put` method is used to store data in the store. The first argument is the namespace, and the second argument is the key. This will store the user information in the store.
6. The `userId` is passed in the config. This is used to identify the user whose information is being updated.
   :::

### Use semantic search

Enable semantic search in your graph's memory store to let graph agents search for items in the store by semantic similarity.

:::python

```python
from langchain.embeddings import init_embeddings
from langgraph.store.memory import InMemoryStore

# Create store with semantic search enabled
embeddings = init_embeddings("openai:text-embedding-3-small")
store = InMemoryStore(
    index={
        "embed": embeddings,
        "dims": 1536,
    }
)

store.put(("user_123", "memories"), "1", {"text": "I love pizza"})
store.put(("user_123", "memories"), "2", {"text": "I am a plumber"})

items = store.search(
    ("user_123", "memories"), query="I'm hungry", limit=1
)
```

:::

:::js

```typescript
import { OpenAIEmbeddings } from "@langchain/openai";
import { InMemoryStore } from "@langchain/langgraph";

// Create store with semantic search enabled
const embeddings = new OpenAIEmbeddings({ model: "text-embedding-3-small" });
const store = new InMemoryStore({
  index: {
    embeddings,
    dims: 1536,
  },
});

await store.put(["user_123", "memories"], "1", { text: "I love pizza" });
await store.put(["user_123", "memories"], "2", { text: "I am a plumber" });

const items = await store.search(["user_123", "memories"], {
  query: "I'm hungry",
  limit: 1,
});
```

:::

??? example "Long-term memory with semantic search"

    :::python
    ```python
    from typing import Optional

    from langchain.embeddings import init_embeddings
    from langchain.chat_models import init_chat_model
    from langgraph.store.base import BaseStore
    from langgraph.store.memory import InMemoryStore
    from langgraph.graph import START, MessagesState, StateGraph

    llm = init_chat_model("openai:gpt-4o-mini")

    # Create store with semantic search enabled
    embeddings = init_embeddings("openai:text-embedding-3-small")
    store = InMemoryStore(
        index={
            "embed": embeddings,
            "dims": 1536,
        }
    )

    store.put(("user_123", "memories"), "1", {"text": "I love pizza"})
    store.put(("user_123", "memories"), "2", {"text": "I am a plumber"})

    def chat(state, *, store: BaseStore):
        # Search based on user's last message
        items = store.search(
            ("user_123", "memories"), query=state["messages"][-1].content, limit=2
        )
        memories = "\n".join(item.value["text"] for item in items)
        memories = f"## Memories of user\n{memories}" if memories else ""
        response = llm.invoke(
            [
                {"role": "system", "content": f"You are a helpful assistant.\n{memories}"},
                *state["messages"],
            ]
        )
        return {"messages": [response]}


    builder = StateGraph(MessagesState)
    builder.add_node(chat)
    builder.add_edge(START, "chat")
    graph = builder.compile(store=store)

    for message, metadata in graph.stream(
        input={"messages": [{"role": "user", "content": "I'm hungry"}]},
        stream_mode="messages",
    ):
        print(message.content, end="")
    ```
    :::

    :::js
    ```typescript
    import { OpenAIEmbeddings, ChatOpenAI } from "@langchain/openai";
    import { StateGraph, START, MessagesZodState, InMemoryStore } from "@langchain/langgraph";
    import { z } from "zod";

    const llm = new ChatOpenAI({ model: "gpt-4o-mini" });

    // Create store with semantic search enabled
    const embeddings = new OpenAIEmbeddings({ model: "text-embedding-3-small" });
    const store = new InMemoryStore({
      index: {
        embeddings,
        dims: 1536,
      }
    });

    await store.put(["user_123", "memories"], "1", { text: "I love pizza" });
    await store.put(["user_123", "memories"], "2", { text: "I am a plumber" });

    const chat = async (state: z.infer<typeof MessagesZodState>, config) => {
      // Search based on user's last message
      const items = await config.store.search(
        ["user_123", "memories"],
        { query: state.messages.at(-1)?.content, limit: 2 }
      );
      const memories = items.map(item => item.value.text).join("\n");
      const memoriesText = memories ? `## Memories of user\n${memories}` : "";

      const response = await llm.invoke([
        { role: "system", content: `You are a helpful assistant.\n${memoriesText}` },
        ...state.messages,
      ]);

      return { messages: [response] };
    };

    const builder = new StateGraph(MessagesZodState)
      .addNode("chat", chat)
      .addEdge(START, "chat");
    const graph = builder.compile({ store });

    for await (const [message, metadata] of await graph.stream(
      { messages: [{ role: "user", content: "I'm hungry" }] },
      { streamMode: "messages" }
    )) {
      if (message.content) {
        console.log(message.content);
      }
    }
    ```
    :::

See [this guide](../../cloud/deployment/semantic_search.md) for more information on how to use semantic search with LangGraph memory store.

## Manage short-term memory

With [short-term memory](#add-short-term-memory) enabled, long conversations can exceed the LLM's context window. Common solutions are:

- [Trim messages](#trim-messages): Remove first or last N messages (before calling LLM)
- [Delete messages](#delete-messages) from LangGraph state permanently
- [Summarize messages](#summarize-messages): Summarize earlier messages in the history and replace them with a summary
- [Manage checkpoints](#manage-checkpoints) to store and retrieve message history
- Custom strategies (e.g., message filtering, etc.)

This allows the agent to keep track of the conversation without exceeding the LLM's context window.

### Trim messages

Most LLMs have a maximum supported context window (denominated in tokens). One way to decide when to truncate messages is to count the tokens in the message history and truncate whenever it approaches that limit. If you're using LangChain, you can use the trim messages utility and specify the number of tokens to keep from the list, as well as the `strategy` (e.g., keep the last `maxTokens`) to use for handling the boundary.

=== "In an agent"

    :::python
    To trim message history in an agent, use @[`pre_model_hook`][create_react_agent] with the [`trim_messages`](https://python.langchain.com/api_reference/core/messages/langchain_core.messages.utils.trim_messages.html) function:

    ```python
    # highlight-next-line
    from langchain_core.messages.utils import (
        # highlight-next-line
        trim_messages,
        # highlight-next-line
        count_tokens_approximately
    # highlight-next-line
    )
    from langgraph.prebuilt import create_react_agent

    # This function will be called every time before the node that calls LLM
    def pre_model_hook(state):
        trimmed_messages = trim_messages(
            state["messages"],
            strategy="last",
            token_counter=count_tokens_approximately,
            max_tokens=384,
            start_on="human",
            end_on=("human", "tool"),
        )
        # highlight-next-line
        return {"llm_input_messages": trimmed_messages}

    checkpointer = InMemorySaver()
    agent = create_react_agent(
        model,
        tools,
        # highlight-next-line
        pre_model_hook=pre_model_hook,
        checkpointer=checkpointer,
    )
    ```
    :::

    :::js
    To trim message history in an agent, use `stateModifier` with the [`trimMessages`](https://js.langchain.com/docs/how_to/trim_messages/) function:

    ```typescript
    import { trimMessages } from "@langchain/core/messages";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";

    // This function will be called every time before the node that calls LLM
    const stateModifier = async (state) => {
      return trimMessages(state.messages, {
        strategy: "last",
        maxTokens: 384,
        startOn: "human",
        endOn: ["human", "tool"],
      });
    };

    const checkpointer = new MemorySaver();
    const agent = createReactAgent({
      llm: model,
      tools,
      stateModifier,
      checkpointer,
    });
    ```
    :::

=== "In a workflow"

    :::python
    To trim message history, use the [`trim_messages`](https://python.langchain.com/api_reference/core/messages/langchain_core.messages.utils.trim_messages.html) function:

    ```python
    # highlight-next-line
    from langchain_core.messages.utils import (
        # highlight-next-line
        trim_messages,
        # highlight-next-line
        count_tokens_approximately
    # highlight-next-line
    )

    def call_model(state: MessagesState):
        # highlight-next-line
        messages = trim_messages(
            state["messages"],
            strategy="last",
            token_counter=count_tokens_approximately,
            max_tokens=128,
            start_on="human",
            end_on=("human", "tool"),
        )
        response = model.invoke(messages)
        return {"messages": [response]}

    builder = StateGraph(MessagesState)
    builder.add_node(call_model)
    ...
    ```
    :::

    :::js
    To trim message history, use the [`trimMessages`](https://js.langchain.com/docs/how_to/trim_messages/) function:

    ```typescript
    import { trimMessages } from "@langchain/core/messages";

    const callModel = async (state: z.infer<typeof MessagesZodState>) => {
      const messages = trimMessages(state.messages, {
        strategy: "last",
        maxTokens: 128,
        startOn: "human",
        endOn: ["human", "tool"],
      });
      const response = await model.invoke(messages);
      return { messages: [response] };
    };

    const builder = new StateGraph(MessagesZodState)
      .addNode("call_model", callModel);
    // ...
    ```
    :::

??? example "Full example: trim messages"

    :::python
    ```python
    # highlight-next-line
    from langchain_core.messages.utils import (
        # highlight-next-line
        trim_messages,
        # highlight-next-line
        count_tokens_approximately
    # highlight-next-line
    )
    from langchain.chat_models import init_chat_model
    from langgraph.graph import StateGraph, START, MessagesState

    model = init_chat_model("anthropic:claude-3-7-sonnet-latest")
    summarization_model = model.bind(max_tokens=128)

    def call_model(state: MessagesState):
        # highlight-next-line
        messages = trim_messages(
            state["messages"],
            strategy="last",
            token_counter=count_tokens_approximately,
            max_tokens=128,
            start_on="human",
            end_on=("human", "tool"),
        )
        response = model.invoke(messages)
        return {"messages": [response]}

    checkpointer = InMemorySaver()
    builder = StateGraph(MessagesState)
    builder.add_node(call_model)
    builder.add_edge(START, "call_model")
    graph = builder.compile(checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "1"}}
    graph.invoke({"messages": "hi, my name is bob"}, config)
    graph.invoke({"messages": "write a short poem about cats"}, config)
    graph.invoke({"messages": "now do the same but for dogs"}, config)
    final_response = graph.invoke({"messages": "what's my name?"}, config)

    final_response["messages"][-1].pretty_print()
    ```

    ```
    ================================== Ai Message ==================================

    Your name is Bob, as you mentioned when you first introduced yourself.
    ```
    :::

    :::js
    ```typescript
    import { trimMessages } from "@langchain/core/messages";
    import { ChatAnthropic } from "@langchain/anthropic";
    import { StateGraph, START, MessagesZodState, MemorySaver } from "@langchain/langgraph";
    import { z } from "zod";

    const model = new ChatAnthropic({ model: "claude-3-5-sonnet-20241022" });

    const callModel = async (state: z.infer<typeof MessagesZodState>) => {
      const messages = trimMessages(state.messages, {
        strategy: "last",
        maxTokens: 128,
        startOn: "human",
        endOn: ["human", "tool"],
      });
      const response = await model.invoke(messages);
      return { messages: [response] };
    };

    const checkpointer = new MemorySaver();
    const builder = new StateGraph(MessagesZodState)
      .addNode("call_model", callModel)
      .addEdge(START, "call_model");
    const graph = builder.compile({ checkpointer });

    const config = { configurable: { thread_id: "1" } };
    await graph.invoke({ messages: [{ role: "user", content: "hi, my name is bob" }] }, config);
    await graph.invoke({ messages: [{ role: "user", content: "write a short poem about cats" }] }, config);
    await graph.invoke({ messages: [{ role: "user", content: "now do the same but for dogs" }] }, config);
    const finalResponse = await graph.invoke({ messages: [{ role: "user", content: "what's my name?" }] }, config);

    console.log(finalResponse.messages.at(-1)?.content);
    ```

    ```
    Your name is Bob, as you mentioned when you first introduced yourself.
    ```
    :::

### Delete messages

You can delete messages from the graph state to manage the message history. This is useful when you want to remove specific messages or clear the entire message history.

:::python
To delete messages from the graph state, you can use the `RemoveMessage`. For `RemoveMessage` to work, you need to use a state key with @[`add_messages`][add_messages] [reducer](../../concepts/low_level.md#reducers), like [`MessagesState`](../../concepts/low_level.md#messagesstate).

To remove specific messages:

```python
# highlight-next-line
from langchain_core.messages import RemoveMessage

def delete_messages(state):
    messages = state["messages"]
    if len(messages) > 2:
        # remove the earliest two messages
        # highlight-next-line
        return {"messages": [RemoveMessage(id=m.id) for m in messages[:2]]}
```

To remove **all** messages:

```python
# highlight-next-line
from langgraph.graph.message import REMOVE_ALL_MESSAGES

def delete_messages(state):
    # highlight-next-line
    return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES)]}
```

:::

:::js
To delete messages from the graph state, you can use the `RemoveMessage`. For `RemoveMessage` to work, you need to use a state key with @[`messagesStateReducer`][messagesStateReducer] [reducer](../../concepts/low_level.md#reducers), like `MessagesZodState`.

To remove specific messages:

```typescript
import { RemoveMessage } from "@langchain/core/messages";

const deleteMessages = (state) => {
  const messages = state.messages;
  if (messages.length > 2) {
    // remove the earliest two messages
    return {
      messages: messages
        .slice(0, 2)
        .map((m) => new RemoveMessage({ id: m.id })),
    };
  }
};
```

:::

!!! warning

    When deleting messages, **make sure** that the resulting message history is valid. Check the limitations of the LLM provider you're using. For example:

    * some providers expect message history to start with a `user` message
    * most providers require `assistant` messages with tool calls to be followed by corresponding `tool` result messages.

??? example "Full example: delete messages"

    :::python
    ```python
    # highlight-next-line
    from langchain_core.messages import RemoveMessage

    def delete_messages(state):
        messages = state["messages"]
        if len(messages) > 2:
            # remove the earliest two messages
            # highlight-next-line
            return {"messages": [RemoveMessage(id=m.id) for m in messages[:2]]}

    def call_model(state: MessagesState):
        response = model.invoke(state["messages"])
        return {"messages": response}

    builder = StateGraph(MessagesState)
    builder.add_sequence([call_model, delete_messages])
    builder.add_edge(START, "call_model")

    checkpointer = InMemorySaver()
    app = builder.compile(checkpointer=checkpointer)

    for event in app.stream(
        {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
        config,
        stream_mode="values"
    ):
        print([(message.type, message.content) for message in event["messages"]])

    for event in app.stream(
        {"messages": [{"role": "user", "content": "what's my name?"}]},
        config,
        stream_mode="values"
    ):
        print([(message.type, message.content) for message in event["messages"]])
    ```

    ```
    [('human', "hi! I'm bob")]
    [('human', "hi! I'm bob"), ('ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?')]
    [('human', "hi! I'm bob"), ('ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?'), ('human', "what's my name?")]
    [('human', "hi! I'm bob"), ('ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?'), ('human', "what's my name?"), ('ai', 'Your name is Bob.')]
    [('human', "what's my name?"), ('ai', 'Your name is Bob.')]
    ```
    :::

    :::js
    ```typescript
    import { RemoveMessage } from "@langchain/core/messages";
    import { ChatAnthropic } from "@langchain/anthropic";
    import { StateGraph, START, MessagesZodState, MemorySaver } from "@langchain/langgraph";
    import { z } from "zod";

    const model = new ChatAnthropic({ model: "claude-3-5-sonnet-20241022" });

    const deleteMessages = (state: z.infer<typeof MessagesZodState>) => {
      const messages = state.messages;
      if (messages.length > 2) {
        // remove the earliest two messages
        return { messages: messages.slice(0, 2).map(m => new RemoveMessage({ id: m.id })) };
      }
      return {};
    };

    const callModel = async (state: z.infer<typeof MessagesZodState>) => {
      const response = await model.invoke(state.messages);
      return { messages: [response] };
    };

    const builder = new StateGraph(MessagesZodState)
      .addNode("call_model", callModel)
      .addNode("delete_messages", deleteMessages)
      .addEdge(START, "call_model")
      .addEdge("call_model", "delete_messages");

    const checkpointer = new MemorySaver();
    const app = builder.compile({ checkpointer });

    const config = { configurable: { thread_id: "1" } };

    for await (const event of await app.stream(
      { messages: [{ role: "user", content: "hi! I'm bob" }] },
      { ...config, streamMode: "values" }
    )) {
      console.log(event.messages.map(message => [message.getType(), message.content]));
    }

    for await (const event of await app.stream(
      { messages: [{ role: "user", content: "what's my name?" }] },
      { ...config, streamMode: "values" }
    )) {
      console.log(event.messages.map(message => [message.getType(), message.content]));
    }
    ```

    ```
    [['human', "hi! I'm bob"]]
    [['human', "hi! I'm bob"], ['ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?']]
    [['human', "hi! I'm bob"], ['ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?'], ['human', "what's my name?"]]
    [['human', "hi! I'm bob"], ['ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?'], ['human', "what's my name?"], ['ai', 'Your name is Bob.']]
    [['human', "what's my name?"], ['ai', 'Your name is Bob.']]
    ```
    :::

### Summarize messages

The problem with trimming or removing messages, as shown above, is that you may lose information from culling of the message queue. Because of this, some applications benefit from a more sophisticated approach of summarizing the message history using a chat model.

![](../../concepts/img/memory/summary.png)

=== "In an agent"

    :::python
    To summarize message history in an agent, use @[`pre_model_hook`][create_react_agent] with a prebuilt [`SummarizationNode`](https://langchain-ai.github.io/langmem/reference/short_term/#langmem.short_term.SummarizationNode) abstraction:

    ```python
    from langchain_anthropic import ChatAnthropic
    from langmem.short_term import SummarizationNode, RunningSummary
    from langchain_core.messages.utils import count_tokens_approximately
    from langgraph.prebuilt import create_react_agent
    from langgraph.prebuilt.chat_agent_executor import AgentState
    from langgraph.checkpoint.memory import InMemorySaver
    from typing import Any

    model = ChatAnthropic(model="claude-3-7-sonnet-latest")

    summarization_node = SummarizationNode( # (1)!
        token_counter=count_tokens_approximately,
        model=model,
        max_tokens=384,
        max_summary_tokens=128,
        output_messages_key="llm_input_messages",
    )

    class State(AgentState):
        # NOTE: we're adding this key to keep track of previous summary information
        # to make sure we're not summarizing on every LLM call
        # highlight-next-line
        context: dict[str, RunningSummary]  # (2)!


    checkpointer = InMemorySaver() # (3)!

    agent = create_react_agent(
        model=model,
        tools=tools,
        # highlight-next-line
        pre_model_hook=summarization_node, # (4)!
        # highlight-next-line
        state_schema=State, # (5)!
        checkpointer=checkpointer,
    )
    ```

    1. The `InMemorySaver` is a checkpointer that stores the agent's state in memory. In a production setting, you would typically use a database or other persistent storage. Please review the [checkpointer documentation](../../reference/checkpoints.md) for more options. If you're deploying with **LangGraph Platform**, the platform will provide a production-ready checkpointer for you.
    2. The `context` key is added to the agent's state. The key contains book-keeping information for the summarization node. It is used to keep track of the last summary information and ensure that the agent doesn't summarize on every LLM call, which can be inefficient.
    3. The `checkpointer` is passed to the agent. This enables the agent to persist its state across invocations.
    4. The `pre_model_hook` is set to the `SummarizationNode`. This node will summarize the message history before sending it to the LLM. The summarization node will automatically handle the summarization process and update the agent's state with the new summary. You can replace this with a custom implementation if you prefer. Please see the @[create_react_agent][create_react_agent] API reference for more details.
    5. The `state_schema` is set to the `State` class, which is the custom state that contains an extra `context` key.
    :::

=== "In a workflow"

    :::python
    Prompting and orchestration logic can be used to summarize the message history. For example, in LangGraph you can extend the [`MessagesState`](../../concepts/low_level.md#working-with-messages-in-graph-state) to include a `summary` key:

    ```python
    from langgraph.graph import MessagesState
    class State(MessagesState):
        summary: str
    ```

    Then, you can generate a summary of the chat history, using any existing summary as context for the next summary. This `summarize_conversation` node can be called after some number of messages have accumulated in the `messages` state key.

    ```python
    def summarize_conversation(state: State):

        # First, we get any existing summary
        summary = state.get("summary", "")

        # Create our summarization prompt
        if summary:

            # A summary already exists
            summary_message = (
                f"This is a summary of the conversation to date: {summary}\n\n"
                "Extend the summary by taking into account the new messages above:"
            )

        else:
            summary_message = "Create a summary of the conversation above:"

        # Add prompt to our history
        messages = state["messages"] + [HumanMessage(content=summary_message)]
        response = model.invoke(messages)

        # Delete all but the 2 most recent messages
        delete_messages = [RemoveMessage(id=m.id) for m in state["messages"][:-2]]
        return {"summary": response.content, "messages": delete_messages}
    ```
    :::

    :::js
    Prompting and orchestration logic can be used to summarize the message history. For example, in LangGraph you can extend the [`MessagesZodState`](../../concepts/low_level.md#working-with-messages-in-graph-state) to include a `summary` key:

    ```typescript
    import { MessagesZodState } from "@langchain/langgraph";
    import { z } from "zod";

    const State = MessagesZodState.merge(z.object({
      summary: z.string().optional(),
    }));
    ```

    Then, you can generate a summary of the chat history, using any existing summary as context for the next summary. This `summarizeConversation` node can be called after some number of messages have accumulated in the `messages` state key.

    ```typescript
    import { RemoveMessage, HumanMessage } from "@langchain/core/messages";

    const summarizeConversation = async (state: z.infer<typeof State>) => {
      // First, we get any existing summary
      const summary = state.summary || "";

      // Create our summarization prompt
      let summaryMessage: string;
      if (summary) {
        // A summary already exists
        summaryMessage =
          `This is a summary of the conversation to date: ${summary}\n\n` +
          "Extend the summary by taking into account the new messages above:";
      } else {
        summaryMessage = "Create a summary of the conversation above:";
      }

      // Add prompt to our history
      const messages = [
        ...state.messages,
        new HumanMessage({ content: summaryMessage })
      ];
      const response = await model.invoke(messages);

      // Delete all but the 2 most recent messages
      const deleteMessages = state.messages
        .slice(0, -2)
        .map(m => new RemoveMessage({ id: m.id }));

      return {
        summary: response.content,
        messages: deleteMessages
      };
    };
    ```
    :::

??? example "Full example: summarize messages"

    :::python
    ```python
    from typing import Any, TypedDict

    from langchain.chat_models import init_chat_model
    from langchain_core.messages import AnyMessage
    from langchain_core.messages.utils import count_tokens_approximately
    from langgraph.graph import StateGraph, START, MessagesState
    from langgraph.checkpoint.memory import InMemorySaver
    # highlight-next-line
    from langmem.short_term import SummarizationNode, RunningSummary

    model = init_chat_model("anthropic:claude-3-7-sonnet-latest")
    summarization_model = model.bind(max_tokens=128)

    class State(MessagesState):
        # highlight-next-line
        context: dict[str, RunningSummary]  # (1)!

    class LLMInputState(TypedDict):  # (2)!
        summarized_messages: list[AnyMessage]
        context: dict[str, RunningSummary]

    # highlight-next-line
    summarization_node = SummarizationNode(
        token_counter=count_tokens_approximately,
        model=summarization_model,
        max_tokens=256,
        max_tokens_before_summary=256,
        max_summary_tokens=128,
    )

    # highlight-next-line
    def call_model(state: LLMInputState):  # (3)!
        response = model.invoke(state["summarized_messages"])
        return {"messages": [response]}

    checkpointer = InMemorySaver()
    builder = StateGraph(State)
    builder.add_node(call_model)
    # highlight-next-line
    builder.add_node("summarize", summarization_node)
    builder.add_edge(START, "summarize")
    builder.add_edge("summarize", "call_model")
    graph = builder.compile(checkpointer=checkpointer)

    # Invoke the graph
    config = {"configurable": {"thread_id": "1"}}
    graph.invoke({"messages": "hi, my name is bob"}, config)
    graph.invoke({"messages": "write a short poem about cats"}, config)
    graph.invoke({"messages": "now do the same but for dogs"}, config)
    final_response = graph.invoke({"messages": "what's my name?"}, config)

    final_response["messages"][-1].pretty_print()
    print("\nSummary:", final_response["context"]["running_summary"].summary)
    ```

    1. We will keep track of our running summary in the `context` field
    (expected by the `SummarizationNode`).
    2. Define private state that will be used only for filtering
    the inputs to `call_model` node.
    3. We're passing a private input state here to isolate the messages returned by the summarization node

    ```
    ================================== Ai Message ==================================

    From our conversation, I can see that you introduced yourself as Bob. That's the name you shared with me when we began talking.

    Summary: In this conversation, I was introduced to Bob, who then asked me to write a poem about cats. I composed a poem titled "The Mystery of Cats" that captured cats' graceful movements, independent nature, and their special relationship with humans. Bob then requested a similar poem about dogs, so I wrote "The Joy of Dogs," which highlighted dogs' loyalty, enthusiasm, and loving companionship. Both poems were written in a similar style but emphasized the distinct characteristics that make each pet special.
    ```
    :::

    :::js
    ```typescript
    import { ChatAnthropic } from "@langchain/anthropic";
    import {
      SystemMessage,
      HumanMessage,
      RemoveMessage,
      type BaseMessage
    } from "@langchain/core/messages";
    import {
      MessagesZodState,
      StateGraph,
      START,
      END,
      MemorySaver,
    } from "@langchain/langgraph";
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";

    const memory = new MemorySaver();

    // We will add a `summary` attribute (in addition to `messages` key,
    // which MessagesZodState already has)
    const GraphState = z.object({
      messages: MessagesZodState.shape.messages,
      summary: z.string().default(""),
    });

    // We will use this model for both the conversation and the summarization
    const model = new ChatAnthropic({ model: "claude-3-haiku-20240307" });

    // Define the logic to call the model
    const callModel = async (state: z.infer<typeof GraphState>) => {
      // If a summary exists, we add this in as a system message
      const { summary } = state;
      let { messages } = state;
      if (summary) {
        const systemMessage = new SystemMessage({
          id: uuidv4(),
          content: `Summary of conversation earlier: ${summary}`,
        });
        messages = [systemMessage, ...messages];
      }
      const response = await model.invoke(messages);
      // We return an object, because this will get added to the existing state
      return { messages: [response] };
    };

    // We now define the logic for determining whether to end or summarize the conversation
    const shouldContinue = (state: z.infer<typeof GraphState>) => {
      const messages = state.messages;
      // If there are more than six messages, then we summarize the conversation
      if (messages.length > 6) {
        return "summarize_conversation";
      }
      // Otherwise we can just end
      return END;
    };

    const summarizeConversation = async (state: z.infer<typeof GraphState>) => {
      // First, we summarize the conversation
      const { summary, messages } = state;
      let summaryMessage: string;
      if (summary) {
        // If a summary already exists, we use a different system prompt
        // to summarize it than if one didn't
        summaryMessage =
          `This is summary of the conversation to date: ${summary}\n\n` +
          "Extend the summary by taking into account the new messages above:";
      } else {
        summaryMessage = "Create a summary of the conversation above:";
      }

      const allMessages = [
        ...messages,
        new HumanMessage({ id: uuidv4(), content: summaryMessage }),
      ];

      const response = await model.invoke(allMessages);

      // We now need to delete messages that we no longer want to show up
      // I will delete all but the last two messages, but you can change this
      const deleteMessages = messages
        .slice(0, -2)
        .map((m) => new RemoveMessage({ id: m.id! }));

      if (typeof response.content !== "string") {
        throw new Error("Expected a string response from the model");
      }

      return { summary: response.content, messages: deleteMessages };
    };

    // Define a new graph
    const workflow = new StateGraph(GraphState)
      // Define the conversation node and the summarize node
      .addNode("conversation", callModel)
      .addNode("summarize_conversation", summarizeConversation)
      // Set the entrypoint as conversation
      .addEdge(START, "conversation")
      // We now add a conditional edge
      .addConditionalEdges(
        // First, we define the start node. We use `conversation`.
        // This means these are the edges taken after the `conversation` node is called.
        "conversation",
        // Next, we pass in the function that will determine which node is called next.
        shouldContinue,
      )
      // We now add a normal edge from `summarize_conversation` to END.
      // This means that after `summarize_conversation` is called, we end.
      .addEdge("summarize_conversation", END);

    // Finally, we compile it!
    const app = workflow.compile({ checkpointer: memory });
    ```
    :::

### Manage checkpoints

You can view and delete the information stored by the checkpointer.

#### View thread state (checkpoint)

:::python
=== "Graph/Functional API"

    ```python
    config = {
        "configurable": {
            # highlight-next-line
            "thread_id": "1",
            # optionally provide an ID for a specific checkpoint,
            # otherwise the latest checkpoint is shown
            # highlight-next-line
            # "checkpoint_id": "1f029ca3-1f5b-6704-8004-820c16b69a5a"

        }
    }
    # highlight-next-line
    graph.get_state(config)
    ```

    ```
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today?), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]}, next=(),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
        metadata={
            'source': 'loop',
            'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}},
            'step': 4,
            'parents': {},
            'thread_id': '1'
        },
        created_at='2025-05-05T16:01:24.680462+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
        tasks=(),
        interrupts=()
    )
    ```

=== "Checkpointer API"

    ```python
    config = {
        "configurable": {
            # highlight-next-line
            "thread_id": "1",
            # optionally provide an ID for a specific checkpoint,
            # otherwise the latest checkpoint is shown
            # highlight-next-line
            # "checkpoint_id": "1f029ca3-1f5b-6704-8004-820c16b69a5a"

        }
    }
    # highlight-next-line
    checkpointer.get_tuple(config)
    ```

    ```
    CheckpointTuple(
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
        checkpoint={
            'v': 3,
            'ts': '2025-05-05T16:01:24.680462+00:00',
            'id': '1f029ca3-1f5b-6704-8004-820c16b69a5a',
            'channel_versions': {'__start__': '00000000000000000000000000000005.0.5290678567601859', 'messages': '00000000000000000000000000000006.0.3205149138784782', 'branch:to:call_model': '00000000000000000000000000000006.0.14611156755133758'}, 'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000004.0.5736472536395331'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000005.0.1410174088651449'}},
            'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today?), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
        },
        metadata={
            'source': 'loop',
            'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}},
            'step': 4,
            'parents': {},
            'thread_id': '1'
        },
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
        pending_writes=[]
    )
    ```

:::

:::js

```typescript
const config = {
  configurable: {
    thread_id: "1",
    // optionally provide an ID for a specific checkpoint,
    // otherwise the latest checkpoint is shown
    // checkpoint_id: "1f029ca3-1f5b-6704-8004-820c16b69a5a"
  },
};
await graph.getState(config);
```

```
{
  values: { messages: [HumanMessage(...), AIMessage(...), HumanMessage(...), AIMessage(...)] },
  next: [],
  config: { configurable: { thread_id: '1', checkpoint_ns: '', checkpoint_id: '1f029ca3-1f5b-6704-8004-820c16b69a5a' } },
  metadata: {
    source: 'loop',
    writes: { call_model: { messages: AIMessage(...) } },
    step: 4,
    parents: {},
    thread_id: '1'
  },
  createdAt: '2025-05-05T16:01:24.680462+00:00',
  parentConfig: { configurable: { thread_id: '1', checkpoint_ns: '', checkpoint_id: '1f029ca3-1790-6b0a-8003-baf965b6a38f' } },
  tasks: [],
  interrupts: []
}
```

:::

#### View the history of the thread (checkpoints)

:::python
=== "Graph/Functional API"

    ```python
    config = {
        "configurable": {
            # highlight-next-line
            "thread_id": "1"
        }
    }
    # highlight-next-line
    list(graph.get_state_history(config))
    ```

    ```
    [
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
            next=(),
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}}, 'step': 4, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:24.680462+00:00',
            parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            tasks=(),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?")]},
            next=('call_model',),
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            metadata={'source': 'loop', 'writes': None, 'step': 3, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:23.863421+00:00',
            parent_config={...}
            tasks=(PregelTask(id='8ab4155e-6b15-b885-9ce5-bed69a2c305c', name='call_model', path=('__pregel_pull', 'call_model'), error=None, interrupts=(), state=None, result={'messages': AIMessage(content='Your name is Bob.')}),),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]},
            next=('__start__',),
            config={...},
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "what's my name?"}]}}, 'step': 2, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:23.863173+00:00',
            parent_config={...}
            tasks=(PregelTask(id='24ba39d6-6db1-4c9b-f4c5-682aeaf38dcd', name='__start__', path=('__pregel_pull', '__start__'), error=None, interrupts=(), state=None, result={'messages': [{'role': 'user', 'content': "what's my name?"}]}),),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]},
            next=(),
            config={...},
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}}, 'step': 1, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:23.862295+00:00',
            parent_config={...}
            tasks=(),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob")]},
            next=('call_model',),
            config={...},
            metadata={'source': 'loop', 'writes': None, 'step': 0, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:22.278960+00:00',
            parent_config={...}
            tasks=(PregelTask(id='8cbd75e0-3720-b056-04f7-71ac805140a0', name='call_model', path=('__pregel_pull', 'call_model'), error=None, interrupts=(), state=None, result={'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}),),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': []},
            next=('__start__',),
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-0870-6ce2-bfff-1f3f14c3e565'}},
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}}, 'step': -1, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:22.277497+00:00',
            parent_config=None,
            tasks=(PregelTask(id='d458367b-8265-812c-18e2-33001d199ce6', name='__start__', path=('__pregel_pull', '__start__'), error=None, interrupts=(), state=None, result={'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}),),
            interrupts=()
        )
    ]
    ```

=== "Checkpointer API"

    ```python
    config = {
        "configurable": {
            # highlight-next-line
            "thread_id": "1"
        }
    }
    # highlight-next-line
    list(checkpointer.list(config))
    ```

    ```
    [
        CheckpointTuple(
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:24.680462+00:00',
                'id': '1f029ca3-1f5b-6704-8004-820c16b69a5a',
                'channel_versions': {'__start__': '00000000000000000000000000000005.0.5290678567601859', 'messages': '00000000000000000000000000000006.0.3205149138784782', 'branch:to:call_model': '00000000000000000000000000000006.0.14611156755133758'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000004.0.5736472536395331'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000005.0.1410174088651449'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
            },
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}}, 'step': 4, 'parents': {}, 'thread_id': '1'},
            parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            pending_writes=[]
        ),
        CheckpointTuple(
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:23.863421+00:00',
                'id': '1f029ca3-1790-6b0a-8003-baf965b6a38f',
                'channel_versions': {'__start__': '00000000000000000000000000000005.0.5290678567601859', 'messages': '00000000000000000000000000000006.0.3205149138784782', 'branch:to:call_model': '00000000000000000000000000000006.0.14611156755133758'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000004.0.5736472536395331'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000005.0.1410174088651449'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?")], 'branch:to:call_model': None}
            },
            metadata={'source': 'loop', 'writes': None, 'step': 3, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[('8ab4155e-6b15-b885-9ce5-bed69a2c305c', 'messages', AIMessage(content='Your name is Bob.'))]
        ),
        CheckpointTuple(
            config={...},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:23.863173+00:00',
                'id': '1f029ca3-1790-616e-8002-9e021694a0cd',
                'channel_versions': {'__start__': '00000000000000000000000000000004.0.5736472536395331', 'messages': '00000000000000000000000000000003.0.7056767754077798', 'branch:to:call_model': '00000000000000000000000000000003.0.22059023329132854'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000001.0.7040775356287469'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000002.0.9300422176788571'}},
                'channel_values': {'__start__': {'messages': [{'role': 'user', 'content': "what's my name?"}]}, 'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]}
            },
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "what's my name?"}]}}, 'step': 2, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[('24ba39d6-6db1-4c9b-f4c5-682aeaf38dcd', 'messages', [{'role': 'user', 'content': "what's my name?"}]), ('24ba39d6-6db1-4c9b-f4c5-682aeaf38dcd', 'branch:to:call_model', None)]
        ),
        CheckpointTuple(
            config={...},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:23.862295+00:00',
                'id': '1f029ca3-178d-6f54-8001-d7b180db0c89',
                'channel_versions': {'__start__': '00000000000000000000000000000002.0.18673090920108737', 'messages': '00000000000000000000000000000003.0.7056767754077798', 'branch:to:call_model': '00000000000000000000000000000003.0.22059023329132854'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000001.0.7040775356287469'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000002.0.9300422176788571'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]}
            },
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}}, 'step': 1, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[]
        ),
        CheckpointTuple(
            config={...},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:22.278960+00:00',
                'id': '1f029ca3-0874-6612-8000-339f2abc83b1',
                'channel_versions': {'__start__': '00000000000000000000000000000002.0.18673090920108737', 'messages': '00000000000000000000000000000002.0.30296526818059655', 'branch:to:call_model': '00000000000000000000000000000002.0.9300422176788571'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000001.0.7040775356287469'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob")], 'branch:to:call_model': None}
            },
            metadata={'source': 'loop', 'writes': None, 'step': 0, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[('8cbd75e0-3720-b056-04f7-71ac805140a0', 'messages', AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'))]
        ),
        CheckpointTuple(
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-0870-6ce2-bfff-1f3f14c3e565'}},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:22.277497+00:00',
                'id': '1f029ca3-0870-6ce2-bfff-1f3f14c3e565',
                'channel_versions': {'__start__': '00000000000000000000000000000001.0.7040775356287469'},
                'versions_seen': {'__input__': {}},
                'channel_values': {'__start__': {'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}}
            },
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}}, 'step': -1, 'parents': {}, 'thread_id': '1'},
            parent_config=None,
            pending_writes=[('d458367b-8265-812c-18e2-33001d199ce6', 'messages', [{'role': 'user', 'content': "hi! I'm bob"}]), ('d458367b-8265-812c-18e2-33001d199ce6', 'branch:to:call_model', None)]
        )
    ]
    ```

:::

:::js

```typescript
const config = {
  configurable: {
    thread_id: "1",
  },
};

const history = [];
for await (const state of graph.getStateHistory(config)) {
  history.push(state);
}
```

:::

#### Delete all checkpoints for a thread

:::python

```python
thread_id = "1"
checkpointer.delete_thread(thread_id)
```

:::

:::js

```typescript
const threadId = "1";
await checkpointer.deleteThread(threadId);
```

:::

:::python

## Prebuilt memory tools

**LangMem** is a LangChain-maintained library that offers tools for managing long-term memories in your agent. See the [LangMem documentation](https://langchain-ai.github.io/langmem/) for usage examples.

:::


```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/memory/semantic-search.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# How to add semantic search to your agent's memory\n",
    "\n",
    "This guide shows how to enable semantic search in your agent's memory store. This lets search for items in the store by semantic similarity.\n",
    "\n",
    "!!! tip Prerequisites\n",
    "    This guide assumes familiarity with the [memory in LangGraph](https://langchain-ai.github.io/langgraph/concepts/memory/).\n",
    "\n",
    "First, install this guide's prerequisites."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "%%capture --no-stderr\n",
    "%pip install -U langgraph langchain-openai langchain"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 1,
   "metadata": {},
   "outputs": [],
   "source": [
    "import getpass\n",
    "import os\n",
    "\n",
    "\n",
    "def _set_env(var: str):\n",
    "    if not os.environ.get(var):\n",
    "        os.environ[var] = getpass.getpass(f\"{var}: \")\n",
    "\n",
    "\n",
    "_set_env(\"OPENAI_API_KEY\")"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "Next, create the store with an [index configuration](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.IndexConfig). By default, stores are configured without semantic/vector search. You can opt in to indexing items when creating the store by providing an [IndexConfig](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.IndexConfig) to the store's constructor. If your store class does not implement this interface, or if you do not pass in an index configuration, semantic search is disabled, and all `index` arguments passed to `put` or `aput` will have no effect. Below is an example."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 2,
   "metadata": {},
   "outputs": [
    {
     "name": "stderr",
     "output_type": "stream",
     "text": [
      "/var/folders/gf/6rnp_mbx5914kx7qmmh7xzmw0000gn/T/ipykernel_83572/2318027494.py:5: LangChainBetaWarning: The function `init_embeddings` is in beta. It is actively being worked on, so the API may change.\n",
      "  embeddings = init_embeddings(\"openai:text-embedding-3-small\")\n"
     ]
    }
   ],
   "source": [
    "from langchain.embeddings import init_embeddings\n",
    "from langgraph.store.memory import InMemoryStore\n",
    "\n",
    "# Create store with semantic search enabled\n",
    "embeddings = init_embeddings(\"openai:text-embedding-3-small\")\n",
    "store = InMemoryStore(\n",
    "    index={\n",
    "        \"embed\": embeddings,\n",
    "        \"dims\": 1536,\n",
    "    }\n",
    ")"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "Now let's store some memories:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 3,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Store some memories\n",
    "store.put((\"user_123\", \"memories\"), \"1\", {\"text\": \"I love pizza\"})\n",
    "store.put((\"user_123\", \"memories\"), \"2\", {\"text\": \"I prefer Italian food\"})\n",
    "store.put((\"user_123\", \"memories\"), \"3\", {\"text\": \"I don't like spicy food\"})\n",
    "store.put((\"user_123\", \"memories\"), \"3\", {\"text\": \"I am studying econometrics\"})\n",
    "store.put((\"user_123\", \"memories\"), \"3\", {\"text\": \"I am a plumber\"})"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "Search memories using natural language:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 4,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Memory: I prefer Italian food (similarity: 0.46482669521168163)\n",
      "Memory: I love pizza (similarity: 0.35514845174380766)\n",
      "Memory: I am a plumber (similarity: 0.155698702336571)\n"
     ]
    }
   ],
   "source": [
    "# Find memories about food preferences\n",
    "memories = store.search((\"user_123\", \"memories\"), query=\"I like food?\", limit=5)\n",
    "\n",
    "for memory in memories:\n",
    "    print(f\"Memory: {memory.value['text']} (similarity: {memory.score})\")"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## Using in your agent\n",
    "\n",
    "Add semantic search to any node by injecting the store."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 5,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "What are you in the mood for? Since you love Italian food and pizza, would you like to order a pizza or try making one at home?"
     ]
    }
   ],
   "source": [
    "from typing import Optional\n",
    "\n",
    "from langchain.chat_models import init_chat_model\n",
    "from langgraph.store.base import BaseStore\n",
    "\n",
    "from langgraph.graph import START, MessagesState, StateGraph\n",
    "\n",
    "llm = init_chat_model(\"openai:gpt-4o-mini\")\n",
    "\n",
    "\n",
    "def chat(state, *, store: BaseStore):\n",
    "    # Search based on user's last message\n",
    "    items = store.search(\n",
    "        (\"user_123\", \"memories\"), query=state[\"messages\"][-1].content, limit=2\n",
    "    )\n",
    "    memories = \"\\n\".join(item.value[\"text\"] for item in items)\n",
    "    memories = f\"## Memories of user\\n{memories}\" if memories else \"\"\n",
    "    response = llm.invoke(\n",
    "        [\n",
    "            {\"role\": \"system\", \"content\": f\"You are a helpful assistant.\\n{memories}\"},\n",
    "            *state[\"messages\"],\n",
    "        ]\n",
    "    )\n",
    "    return {\"messages\": [response]}\n",
    "\n",
    "\n",
    "builder = StateGraph(MessagesState)\n",
    "builder.add_node(chat)\n",
    "builder.add_edge(START, \"chat\")\n",
    "graph = builder.compile(store=store)\n",
    "\n",
    "for message, metadata in graph.stream(\n",
    "    input={\"messages\": [{\"role\": \"user\", \"content\": \"I'm hungry\"}]},\n",
    "    stream_mode=\"messages\",\n",
    "):\n",
    "    print(message.content, end=\"\")"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## Using in `create_react_agent` {#using-in-create-react-agent}\n",
    "\n",
    "Add semantic search to your tool calling agent by injecting the store in the `prompt` function. You can also use the store in a tool to let your agent manually store or search for memories."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 6,
   "metadata": {},
   "outputs": [],
   "source": [
    "import uuid\n",
    "from typing import Optional\n",
    "\n",
    "from langchain.chat_models import init_chat_model\n",
    "from langgraph.prebuilt import InjectedStore\n",
    "from langgraph.store.base import BaseStore\n",
    "from typing_extensions import Annotated\n",
    "\n",
    "from langgraph.prebuilt import create_react_agent\n",
    "\n",
    "\n",
    "def prepare_messages(state, *, store: BaseStore):\n",
    "    # Search based on user's last message\n",
    "    items = store.search(\n",
    "        (\"user_123\", \"memories\"), query=state[\"messages\"][-1].content, limit=2\n",
    "    )\n",
    "    memories = \"\\n\".join(item.value[\"text\"] for item in items)\n",
    "    memories = f\"## Memories of user\\n{memories}\" if memories else \"\"\n",
    "    return [\n",
    "        {\"role\": \"system\", \"content\": f\"You are a helpful assistant.\\n{memories}\"}\n",
    "    ] + state[\"messages\"]\n",
    "\n",
    "\n",
    "# You can also use the store directly within a tool!\n",
    "def upsert_memory(\n",
    "    content: str,\n",
    "    *,\n",
    "    memory_id: Optional[uuid.UUID] = None,\n",
    "    store: Annotated[BaseStore, InjectedStore],\n",
    "):\n",
    "    \"\"\"Upsert a memory in the database.\"\"\"\n",
    "    # The LLM can use this tool to store a new memory\n",
    "    mem_id = memory_id or uuid.uuid4()\n",
    "    store.put(\n",
    "        (\"user_123\", \"memories\"),\n",
    "        key=str(mem_id),\n",
    "        value={\"text\": content},\n",
    "    )\n",
    "    return f\"Stored memory {mem_id}\"\n",
    "\n",
    "\n",
    "agent = create_react_agent(\n",
    "    init_chat_model(\"openai:gpt-4o-mini\"),\n",
    "    tools=[upsert_memory],\n",
    "    # The 'prompt' function is run to prepare the messages for the LLM. It is called\n",
    "    # right before each LLM call\n",
    "    prompt=prepare_messages,\n",
    "    store=store,\n",
    ")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 7,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "What are you in the mood for? Since you love Italian food and pizza, maybe something in that realm would be great! Would you like suggestions for a specific dish or restaurant?"
     ]
    }
   ],
   "source": [
    "for message, metadata in agent.stream(\n",
    "    input={\"messages\": [{\"role\": \"user\", \"content\": \"I'm hungry\"}]},\n",
    "    stream_mode=\"messages\",\n",
    "):\n",
    "    print(message.content, end=\"\")"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## Advanced Usage\n",
    "\n",
    "#### Multi-vector indexing\n",
    "\n",
    "Store and search different aspects of memories separately to improve recall or omit certain fields from being indexed."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 8,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Expect mem 2\n",
      "Item: mem2; Score (0.5895009051396596)\n",
      "Memory: Ate alone at home\n",
      "Emotion: felt a bit lonely\n",
      "\n",
      "Expect mem1\n",
      "Item: mem1; Score (0.6207546534134083)\n",
      "Memory: Had pizza with friends at Mario's\n",
      "Emotion: felt happy and connected\n",
      "\n",
      "Expect random lower score (ravioli not indexed)\n",
      "Item: mem1; Score (0.2686278787315685)\n",
      "Memory: Had pizza with friends at Mario's\n",
      "Emotion: felt happy and connected\n",
      "\n"
     ]
    }
   ],
   "source": [
    "# Configure store to embed both memory content and emotional context\n",
    "store = InMemoryStore(\n",
    "    index={\"embed\": embeddings, \"dims\": 1536, \"fields\": [\"memory\", \"emotional_context\"]}\n",
    ")\n",
    "# Store memories with different content/emotion pairs\n",
    "store.put(\n",
    "    (\"user_123\", \"memories\"),\n",
    "    \"mem1\",\n",
    "    {\n",
    "        \"memory\": \"Had pizza with friends at Mario's\",\n",
    "        \"emotional_context\": \"felt happy and connected\",\n",
    "        \"this_isnt_indexed\": \"I prefer ravioli though\",\n",
    "    },\n",
    ")\n",
    "store.put(\n",
    "    (\"user_123\", \"memories\"),\n",
    "    \"mem2\",\n",
    "    {\n",
    "        \"memory\": \"Ate alone at home\",\n",
    "        \"emotional_context\": \"felt a bit lonely\",\n",
    "        \"this_isnt_indexed\": \"I like pie\",\n",
    "    },\n",
    ")\n",
    "\n",
    "# Search focusing on emotional state - matches mem2\n",
    "results = store.search(\n",
    "    (\"user_123\", \"memories\"), query=\"times they felt isolated\", limit=1\n",
    ")\n",
    "print(\"Expect mem 2\")\n",
    "for r in results:\n",
    "    print(f\"Item: {r.key}; Score ({r.score})\")\n",
    "    print(f\"Memory: {r.value['memory']}\")\n",
    "    print(f\"Emotion: {r.value['emotional_context']}\\n\")\n",
    "\n",
    "# Search focusing on social eating - matches mem1\n",
    "print(\"Expect mem1\")\n",
    "results = store.search((\"user_123\", \"memories\"), query=\"fun pizza\", limit=1)\n",
    "for r in results:\n",
    "    print(f\"Item: {r.key}; Score ({r.score})\")\n",
    "    print(f\"Memory: {r.value['memory']}\")\n",
    "    print(f\"Emotion: {r.value['emotional_context']}\\n\")\n",
    "\n",
    "print(\"Expect random lower score (ravioli not indexed)\")\n",
    "results = store.search((\"user_123\", \"memories\"), query=\"ravioli\", limit=1)\n",
    "for r in results:\n",
    "    print(f\"Item: {r.key}; Score ({r.score})\")\n",
    "    print(f\"Memory: {r.value['memory']}\")\n",
    "    print(f\"Emotion: {r.value['emotional_context']}\\n\")"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "#### Override fields at storage time\n",
    "You can override which fields to embed when storing a specific memory using `put(..., index=[...fields])`, regardless of the store's default configuration."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 9,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Expect mem1\n",
      "Item: mem1; Score (0.3374968677940555)\n",
      "Memory: I love spicy food\n",
      "Context: At a Thai restaurant\n",
      "\n",
      "Expect mem2\n",
      "Item: mem2; Score (0.36784461593247436)\n",
      "Memory: The restaurant was too loud\n",
      "Context: Dinner at an Italian place\n",
      "\n"
     ]
    }
   ],
   "source": [
    "store = InMemoryStore(\n",
    "    index={\n",
    "        \"embed\": embeddings,\n",
    "        \"dims\": 1536,\n",
    "        \"fields\": [\"memory\"],\n",
    "    }  # Default to embed memory field\n",
    ")\n",
    "\n",
    "# Store one memory with default indexing\n",
    "store.put(\n",
    "    (\"user_123\", \"memories\"),\n",
    "    \"mem1\",\n",
    "    {\"memory\": \"I love spicy food\", \"context\": \"At a Thai restaurant\"},\n",
    ")\n",
    "\n",
    "# Store another overriding which fields to embed\n",
    "store.put(\n",
    "    (\"user_123\", \"memories\"),\n",
    "    \"mem2\",\n",
    "    {\"memory\": \"The restaurant was too loud\", \"context\": \"Dinner at an Italian place\"},\n",
    "    index=[\"context\"],  # Override: only embed the context\n",
    ")\n",
    "\n",
    "# Search about food - matches mem1 (using default field)\n",
    "print(\"Expect mem1\")\n",
    "results = store.search(\n",
    "    (\"user_123\", \"memories\"), query=\"what food do they like\", limit=1\n",
    ")\n",
    "for r in results:\n",
    "    print(f\"Item: {r.key}; Score ({r.score})\")\n",
    "    print(f\"Memory: {r.value['memory']}\")\n",
    "    print(f\"Context: {r.value['context']}\\n\")\n",
    "\n",
    "# Search about restaurant atmosphere - matches mem2 (using overridden field)\n",
    "print(\"Expect mem2\")\n",
    "results = store.search(\n",
    "    (\"user_123\", \"memories\"), query=\"restaurant environment\", limit=1\n",
    ")\n",
    "for r in results:\n",
    "    print(f\"Item: {r.key}; Score ({r.score})\")\n",
    "    print(f\"Memory: {r.value['memory']}\")\n",
    "    print(f\"Context: {r.value['context']}\\n\")"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "#### Disable Indexing for Specific Memories\n",
    "\n",
    "Some memories shouldn't be searchable by content. You can disable indexing for these while still storing them using \n",
    "`put(..., index=False)`. Example:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 10,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Expect mem1\n",
      "Item: mem1; Score (0.32269984224327286)\n",
      "Memory: I love chocolate ice cream\n",
      "Type: preference\n",
      "\n",
      "Expect low score (mem2 not indexed)\n",
      "Item: mem1; Score (0.010241633698527089)\n",
      "Memory: I love chocolate ice cream\n",
      "Type: preference\n",
      "\n"
     ]
    }
   ],
   "source": [
    "store = InMemoryStore(index={\"embed\": embeddings, \"dims\": 1536, \"fields\": [\"memory\"]})\n",
    "\n",
    "# Store a normal indexed memory\n",
    "store.put(\n",
    "    (\"user_123\", \"memories\"),\n",
    "    \"mem1\",\n",
    "    {\"memory\": \"I love chocolate ice cream\", \"type\": \"preference\"},\n",
    ")\n",
    "\n",
    "# Store a system memory without indexing\n",
    "store.put(\n",
    "    (\"user_123\", \"memories\"),\n",
    "    \"mem2\",\n",
    "    {\"memory\": \"User completed onboarding\", \"type\": \"system\"},\n",
    "    index=False,  # Disable indexing entirely\n",
    ")\n",
    "\n",
    "# Search about food preferences - finds mem1\n",
    "print(\"Expect mem1\")\n",
    "results = store.search((\"user_123\", \"memories\"), query=\"what food preferences\", limit=1)\n",
    "for r in results:\n",
    "    print(f\"Item: {r.key}; Score ({r.score})\")\n",
    "    print(f\"Memory: {r.value['memory']}\")\n",
    "    print(f\"Type: {r.value['type']}\\n\")\n",
    "\n",
    "# Search about onboarding - won't find mem2 (not indexed)\n",
    "print(\"Expect low score (mem2 not indexed)\")\n",
    "results = store.search((\"user_123\", \"memories\"), query=\"onboarding status\", limit=1)\n",
    "for r in results:\n",
    "    print(f\"Item: {r.key}; Score ({r.score})\")\n",
    "    print(f\"Memory: {r.value['memory']}\")\n",
    "    print(f\"Type: {r.value['type']}\\n\")"
   ]
  }
 ],
 "metadata": {
  "kernelspec": {
   "display_name": "Python 3 (ipykernel)",
   "language": "python",
   "name": "python3"
  },
  "language_info": {
   "codemirror_mode": {
    "name": "ipython",
    "version": 3
   },
   "file_extension": ".py",
   "mimetype": "text/x-python",
   "name": "python",
   "nbconvert_exporter": "python",
   "pygments_lexer": "ipython3",
   "version": "3.12.3"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 4
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/http/custom_lifespan.md
```md
# How to add custom lifespan events

When deploying agents to LangGraph Platform, you often need to initialize resources like database connections when your server starts up, and ensure they're properly closed when it shuts down. Lifespan events let you hook into your server's startup and shutdown sequence to handle these critical setup and teardown tasks.

This works the same way as [adding custom routes](./custom_routes.md). You just need to provide your own [`Starlette`](https://www.starlette.io/applications/) app (including [`FastAPI`](https://fastapi.tiangolo.com/), [`FastHTML`](https://fastht.ml/) and other compatible apps).

Below is an example using FastAPI.

???+ note "Python only"

    We currently only support custom lifespan events in Python deployments with `langgraph-api>=0.0.26`.

## Create app

Starting from an **existing** LangGraph Platform application, add the following lifespan code to your `webapp.py` file. If you are starting from scratch, you can create a new app from a template using the CLI.

```bash
langgraph new --template=new-langgraph-project-python my_new_project
```

Once you have a LangGraph project, add the following app code:

```python
# ./src/agent/webapp.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

@asynccontextmanager
async def lifespan(app: FastAPI):
    # for example...
    engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
    # Create reusable session factory
    async_session = sessionmaker(engine, class_=AsyncSession)
    # Store in app state
    app.state.db_session = async_session
    yield
    # Clean up connections
    await engine.dispose()

# highlight-next-line
app = FastAPI(lifespan=lifespan)

# ... can add custom routes if needed.
```

## Configure `langgraph.json`

Add the following to your `langgraph.json` configuration file. Make sure the path points to the `webapp.py` file you created above.

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent/graph.py:graph"
  },
  "env": ".env",
  "http": {
    "app": "./src/agent/webapp.py:app"
  }
  // Other configuration options like auth, store, etc.
}
```

## Start server

Test the server out locally:

```bash
langgraph dev --no-browser
```

You should see your startup message printed when the server starts, and your cleanup message when you stop it with `Ctrl+C`.

## Deploying

You can deploy your app as-is to LangGraph Platform or to your self-hosted platform.

## Next steps

Now that you've added lifespan events to your deployment, you can use similar techniques to add [custom routes](./custom_routes.md) or [custom middleware](./custom_middleware.md) to further customize your server's behavior.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/http/custom_middleware.md
```md
# How to add custom middleware

When deploying agents to LangGraph Platform, you can add custom middleware to your server to handle concerns like logging request metrics, injecting or checking headers, and enforcing security policies without modifying core server logic. This works the same way as [adding custom routes](./custom_routes.md). You just need to provide your own [`Starlette`](https://www.starlette.io/applications/) app (including [`FastAPI`](https://fastapi.tiangolo.com/), [`FastHTML`](https://fastht.ml/) and other compatible apps).

Adding middleware lets you intercept and modify requests and responses globally across your deployment, whether they're hitting your custom endpoints or the built-in LangGraph Platform APIs.

Below is an example using FastAPI.

???+ note "Python only"

    We currently only support custom middleware in Python deployments with `langgraph-api>=0.0.26`.

## Create app

Starting from an **existing** LangGraph Platform application, add the following middleware code to your webapp file. If you are starting from scratch, you can create a new app from a template using the CLI.

```bash
langgraph new --template=new-langgraph-project-python my_new_project
```

Once you have a LangGraph project, add the following app code:

```python
# ./src/agent/webapp.py
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware

# highlight-next-line
app = FastAPI()

class CustomHeaderMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        response.headers['X-Custom-Header'] = 'Hello from middleware!'
        return response

# Add the middleware to the app
app.add_middleware(CustomHeaderMiddleware)
```

## Configure `langgraph.json`

Add the following to your `langgraph.json` configuration file. Make sure the path points to the `webapp.py` file you created above.

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent/graph.py:graph"
  },
  "env": ".env",
  "http": {
    "app": "./src/agent/webapp.py:app"
  }
  // Other configuration options like auth, store, etc.
}
```

## Start server

Test the server out locally:

```bash
langgraph dev --no-browser
```

Now any request to your server will include the custom header `X-Custom-Header` in its response.

## Deploying

You can deploy this app as-is to LangGraph Platform or to your self-hosted platform.

## Next steps

Now that you've added custom middleware to your deployment, you can use similar techniques to add [custom routes](./custom_routes.md) or define [custom lifespan events](./custom_lifespan.md) to further customize your server's behavior.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/http/custom_routes.md
```md
# How to add custom routes

When deploying agents to LangGraph platform, your server automatically exposes routes for creating runs and threads, interacting with the long-term memory store, managing configurable assistants, and other core functionality ([see all default API endpoints](../../cloud/reference/api/api_ref.md)).

You can add custom routes by providing your own [`Starlette`](https://www.starlette.io/applications/) app (including [`FastAPI`](https://fastapi.tiangolo.com/), [`FastHTML`](https://fastht.ml/) and other compatible apps). You make LangGraph Platform aware of this by providing a path to the app in your `langgraph.json` configuration file.

Defining a custom app object lets you add any routes you'd like, so you can do anything from adding a `/login` endpoint to writing an entire full-stack web-app, all deployed in a single LangGraph Server.

Below is an example using FastAPI.

## Create app

Starting from an **existing** LangGraph Platform application, add the following custom route code to your webapp file. If you are starting from scratch, you can create a new app from a template using the CLI.

```bash
langgraph new --template=new-langgraph-project-python my_new_project
```

Once you have a LangGraph project, add the following app code:

```python
# ./src/agent/webapp.py
from fastapi import FastAPI

# highlight-next-line
app = FastAPI()


@app.get("/hello")
def read_root():
    return {"Hello": "World"}

```

## Configure `langgraph.json`

Add the following to your `langgraph.json` configuration file. Make sure the path points to the FastAPI application instance `app` in the `webapp.py` file you created above.

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent/graph.py:graph"
  },
  "env": ".env",
  "http": {
    "app": "./src/agent/webapp.py:app"
  }
  // Other configuration options like auth, store, etc.
}
```

## Start server

Test the server out locally:

```bash
langgraph dev --no-browser
```

If you navigate to `localhost:2024/hello` in your browser (`2024` is the default development port), you should see the `/hello` endpoint returning `{"Hello": "World"}`.

!!! note "Shadowing default endpoints"

    The routes you create in the app are given priority over the system defaults, meaning you can shadow and redefine the behavior of any default endpoint.

## Deploying

You can deploy this app as-is to LangGraph Platform or to your self-hosted platform.

## Next steps

Now that you've added a custom route to your deployment, you can use this same technique to further customize how your server behaves, such as defining custom [custom middleware](./custom_middleware.md) and [custom lifespan events](./custom_lifespan.md).

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/ttl/configure_ttl.md
```md
# How to add TTLs to your LangGraph application

!!! tip "Prerequisites"

    This guide assumes familiarity with the [LangGraph Platform](../../concepts/langgraph_platform.md), [Persistence](../../concepts/persistence.md), and [Cross-thread persistence](../../concepts/persistence.md#memory-store) concepts.

???+ note "LangGraph platform only"
    
    TTLs are only supported for LangGraph platform deployments. This guide does not apply to LangGraph OSS.

The LangGraph Platform persists both [checkpoints](../../concepts/persistence.md#checkpoints) (thread state) and [cross-thread memories](../../concepts/persistence.md#memory-store) (store items). Configure Time-to-Live (TTL) policies in `langgraph.json` to automatically manage the lifecycle of this data, preventing indefinite accumulation.

## Configuring Checkpoint TTL

Checkpoints capture the state of conversation threads. Setting a TTL ensures old checkpoints and threads are automatically deleted.

Add a `checkpointer.ttl` configuration to your `langgraph.json` file:

:::python
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.py:graph"
  },
  "checkpointer": {
    "ttl": {
      "strategy": "delete",
      "sweep_interval_minutes": 60,
      "default_ttl": 43200 
    }
  }
}
```
:::

:::js
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.ts:graph"
  },
  "checkpointer": {
    "ttl": {
      "strategy": "delete",
      "sweep_interval_minutes": 60,
      "default_ttl": 43200 
    }
  }
}
```
:::

*   `strategy`: Specifies the action taken on expiration. Currently, only `"delete"` is supported, which deletes all checkpoints in the thread upon expiration.
*   `sweep_interval_minutes`: Defines how often, in minutes, the system checks for expired checkpoints.
*   `default_ttl`: Sets the default lifespan of checkpoints in minutes (e.g., 43200 minutes = 30 days).

## Configuring Store Item TTL

Store items allow cross-thread data persistence. Configuring TTL for store items helps manage memory by removing stale data.

Add a `store.ttl` configuration to your `langgraph.json` file:

:::python
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.py:graph"
  },
  "store": {
    "ttl": {
      "refresh_on_read": true,
      "sweep_interval_minutes": 120,
      "default_ttl": 10080
    }
  }
}
```
:::

:::js
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.ts:graph"
  },
  "store": {
    "ttl": {
      "refresh_on_read": true,
      "sweep_interval_minutes": 120,
      "default_ttl": 10080
    }
  }
}
```
:::

*   `refresh_on_read`: (Optional, default `true`) If `true`, accessing an item via `get` or `search` resets its expiration timer. If `false`, TTL only refreshes on `put`.
*   `sweep_interval_minutes`: (Optional) Defines how often, in minutes, the system checks for expired items. If omitted, no sweeping occurs.
*   `default_ttl`: (Optional) Sets the default lifespan of store items in minutes (e.g., 10080 minutes = 7 days). If omitted, items do not expire by default.

## Combining TTL Configurations

You can configure TTLs for both checkpoints and store items in the same `langgraph.json` file to set different policies for each data type. Here is an example:

:::python
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.py:graph"
  },
  "checkpointer": {
    "ttl": {
      "strategy": "delete",
      "sweep_interval_minutes": 60,
      "default_ttl": 43200
    }
  },
  "store": {
    "ttl": {
      "refresh_on_read": true,
      "sweep_interval_minutes": 120,
      "default_ttl": 10080
    }
  }
}
```
:::

:::js
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.ts:graph"
  },
  "checkpointer": {
    "ttl": {
      "strategy": "delete",
      "sweep_interval_minutes": 60,
      "default_ttl": 43200
    }
  },
  "store": {
    "ttl": {
      "refresh_on_read": true,
      "sweep_interval_minutes": 120,
      "default_ttl": 10080
    }
  }
}
```
:::

## Runtime Overrides

The default `store.ttl` settings from `langgraph.json` can be overridden at runtime by providing specific TTL values in SDK method calls like `get`, `put`, and `search`.

## Deployment Process

After configuring TTLs in `langgraph.json`, deploy or restart your LangGraph application for the changes to take effect. Use `langgraph dev` for local development or `langgraph up` for Docker deployment.

See the @[langgraph.json CLI reference][langgraph.json] for more details on the other configurable options.
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/cassettes/agent-simulation-evaluation_32848c2e-be82-46f3-81db-b23fea45461c.msgpack.zlib
```zlib
