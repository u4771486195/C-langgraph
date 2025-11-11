    "        ns, update = update\n",
    "        # skip parent graph updates in the printouts\n",
    "        if len(ns) == 0:\n",
    "            return\n",
    "\n",
    "        graph_id = ns[-1].split(\":\")[0]\n",
    "        print(f\"Update from subgraph {graph_id}:\")\n",
    "        print(\"\\n\")\n",
    "\n",
    "    for node_name, node_update in update.items():\n",
    "        print(f\"Update from node {node_name}:\")\n",
    "        print(\"\\n\")\n",
    "\n",
    "        for m in convert_to_messages(node_update[\"messages\"]):\n",
    "            m.pretty_print()\n",
    "        print(\"\\n\")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "7132e2c0-d937-4325-a30e-e715c5304fe0",
   "metadata": {},
   "source": [
    "Let's test it out using the same input as our original multi-agent system:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 10,
   "id": "29b47c57-ad05-4f10-83bf-c3ff6ff8eb93",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Update from subgraph call_travel_advisor:\n",
      "\n",
      "\n",
      "Update from node agent:\n",
      "\n",
      "\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "[{'text': \"I'll help you find a warm Caribbean destination and then get some hotel recommendations for you.\\n\\nLet me first get some destination recommendations for the Caribbean region.\", 'type': 'text'}, {'id': 'toolu_015vT8PkPq1VXvjrDvSpWUwJ', 'input': {}, 'name': 'get_travel_recommendations', 'type': 'tool_use'}]\n",
      "Tool Calls:\n",
      "  get_travel_recommendations (toolu_015vT8PkPq1VXvjrDvSpWUwJ)\n",
      " Call ID: toolu_015vT8PkPq1VXvjrDvSpWUwJ\n",
      "  Args:\n",
      "\n",
      "\n",
      "Update from subgraph call_travel_advisor:\n",
      "\n",
      "\n",
      "Update from node tools:\n",
      "\n",
      "\n",
      "=================================\u001b[1m Tool Message \u001b[0m=================================\n",
      "Name: get_travel_recommendations\n",
      "\n",
      "turks and caicos\n",
      "\n",
      "\n",
      "Update from subgraph call_travel_advisor:\n",
      "\n",
      "\n",
      "Update from node agent:\n",
      "\n",
      "\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "[{'text': \"Based on the recommendation, I suggest Turks and Caicos! This beautiful British Overseas Territory is known for its stunning white-sand beaches, crystal-clear turquoise waters, and year-round warm weather. Grace Bay Beach in Providenciales is consistently ranked among the world's best beaches. The islands offer excellent snorkeling, diving, and water sports opportunities, plus a relaxed Caribbean atmosphere.\\n\\nNow, let me connect you with our hotel advisor to get some specific hotel recommendations for Turks and Caicos.\", 'type': 'text'}, {'id': 'toolu_01JY7pNNWFuaWoe9ymxFYiPV', 'input': {}, 'name': 'transfer_to_hotel_advisor', 'type': 'tool_use'}]\n",
      "Tool Calls:\n",
      "  transfer_to_hotel_advisor (toolu_01JY7pNNWFuaWoe9ymxFYiPV)\n",
      " Call ID: toolu_01JY7pNNWFuaWoe9ymxFYiPV\n",
      "  Args:\n",
      "\n",
      "\n",
      "Update from subgraph call_travel_advisor:\n",
      "\n",
      "\n",
      "Update from node tools:\n",
      "\n",
      "\n",
      "=================================\u001b[1m Tool Message \u001b[0m=================================\n",
      "Name: transfer_to_hotel_advisor\n",
      "\n",
      "Successfully transferred to hotel advisor\n",
      "\n",
      "\n",
      "Update from subgraph call_hotel_advisor:\n",
      "\n",
      "\n",
      "Update from node agent:\n",
      "\n",
      "\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "[{'text': 'Let me get some hotel recommendations for Turks and Caicos:', 'type': 'text'}, {'id': 'toolu_0129ELa7jFocn16bowaGNapg', 'input': {'location': 'turks and caicos'}, 'name': 'get_hotel_recommendations', 'type': 'tool_use'}]\n",
      "Tool Calls:\n",
      "  get_hotel_recommendations (toolu_0129ELa7jFocn16bowaGNapg)\n",
      " Call ID: toolu_0129ELa7jFocn16bowaGNapg\n",
      "  Args:\n",
      "    location: turks and caicos\n",
      "\n",
      "\n",
      "Update from subgraph call_hotel_advisor:\n",
      "\n",
      "\n",
      "Update from node tools:\n",
      "\n",
      "\n",
      "=================================\u001b[1m Tool Message \u001b[0m=================================\n",
      "Name: get_hotel_recommendations\n",
      "\n",
      "[\"Grace Bay Club\", \"COMO Parrot Cay\"]\n",
      "\n",
      "\n",
      "Update from subgraph call_hotel_advisor:\n",
      "\n",
      "\n",
      "Update from node agent:\n",
      "\n",
      "\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "Here are two excellent hotel options in Turks and Caicos:\n",
      "\n",
      "1. Grace Bay Club: This luxury resort is located on the world-famous Grace Bay Beach. It offers all-oceanfront suites, exceptional dining options, and personalized service. The resort features adult-only and family-friendly sections, making it perfect for any type of traveler.\n",
      "\n",
      "2. COMO Parrot Cay: This exclusive private island resort offers the ultimate luxury escape. It's known for its pristine beach, world-class spa, and holistic wellness programs. The resort provides an intimate, secluded experience with top-notch amenities and service.\n",
      "\n",
      "Would you like more specific information about either of these properties or would you like to explore hotels in another destination?\n",
      "\n",
      "\n"
     ]
    }
   ],
   "source": [
    "for chunk in workflow.stream(\n",
    "    [\n",
    "        {\n",
    "            \"role\": \"user\",\n",
    "            \"content\": \"i wanna go somewhere warm in the caribbean. pick one destination and give me hotel recommendations\",\n",
    "        }\n",
    "    ],\n",
    "    subgraphs=True,\n",
    "):\n",
    "    pretty_print_messages(chunk)"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "d7d89ee0-0229-4718-9b98-bdd3f59c1014",
   "metadata": {},
   "source": [
    "Voila - `travel_advisor` picks a destination and then makes a decision to call `hotel_advisor` for more info!"
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
 "nbformat_minor": 5
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/multi_agent.md
```md
# Build multi-agent systems

A single agent might struggle if it needs to specialize in multiple domains or manage many tools. To tackle this, you can break your agent into smaller, independent agents and composing them into a [multi-agent system](../concepts/multi_agent.md).

In multi-agent systems, agents need to communicate between each other. They do so via [handoffs](#handoffs) — a primitive that describes which agent to hand control to and the payload to send to that agent.

This guide covers the following:

* implementing [handoffs](#handoffs) between agents
* using handoffs and the prebuilt [agent](../agents/agents.md) to [build a custom multi-agent system](#build-a-multi-agent-system)

To get started with building multi-agent systems, check out LangGraph [prebuilt implementations](#prebuilt-implementations) of two of the most popular multi-agent architectures — [supervisor](../agents/multi-agent.md#supervisor) and [swarm](../agents/multi-agent.md#swarm).

## Handoffs

To set up communication between the agents in a multi-agent system you can use [**handoffs**](../concepts/multi_agent.md#handoffs) — a pattern where one agent *hands off* control to another. Handoffs allow you to specify:

- **destination**: target agent to navigate to (e.g., name of the LangGraph node to go to)
- **payload**: information to pass to that agent (e.g., state update)

### Create handoffs

To implement handoffs, you can return `Command` objects from your agent nodes or tools:

:::python
```python
from typing import Annotated
from langchain_core.tools import tool, InjectedToolCallId
from langgraph.prebuilt import create_react_agent, InjectedState
from langgraph.graph import StateGraph, START, MessagesState
from langgraph.types import Command

def create_handoff_tool(*, agent_name: str, description: str | None = None):
    name = f"transfer_to_{agent_name}"
    description = description or f"Transfer to {agent_name}"

    @tool(name, description=description)
    def handoff_tool(
        # highlight-next-line
        state: Annotated[MessagesState, InjectedState], # (1)!
        # highlight-next-line
        tool_call_id: Annotated[str, InjectedToolCallId],
    ) -> Command:
        tool_message = {
            "role": "tool",
            "content": f"Successfully transferred to {agent_name}",
            "name": name,
            "tool_call_id": tool_call_id,
        }
        return Command(  # (2)!
            # highlight-next-line
            goto=agent_name,  # (3)!
            # highlight-next-line
            update={"messages": state["messages"] + [tool_message]},  # (4)!
            # highlight-next-line
            graph=Command.PARENT,  # (5)!
        )
    return handoff_tool
```

1. Access the [state](../concepts/low_level.md#state) of the agent that is calling the handoff tool using the @[InjectedState] annotation. 
2. The `Command` primitive allows specifying a state update and a node transition as a single operation, making it useful for implementing handoffs.
3. Name of the agent or node to hand off to.
4. Take the agent's messages and **add** them to the parent's **state** as part of the handoff. The next agent will see the parent state.
5. Indicate to LangGraph that we need to navigate to agent node in a **parent** multi-agent graph.

!!! tip

    If you want to use tools that return `Command`, you can either use prebuilt @[`create_react_agent`][create_react_agent] / @[`ToolNode`][ToolNode] components, or implement your own tool-executing node that collects `Command` objects returned by the tools and returns a list of them, e.g.:
    
    ```python
    def call_tools(state):
        ...
        commands = [tools_by_name[tool_call["name"]].invoke(tool_call) for tool_call in tool_calls]
        return commands
    ```
:::

:::js
```typescript
import { tool } from "@langchain/core/tools";
import { Command, MessagesZodState } from "@langchain/langgraph";
import { z } from "zod";

function createHandoffTool({
  agentName,
  description,
}: {
  agentName: string;
  description?: string;
}) {
  const name = `transfer_to_${agentName}`;
  const toolDescription = description || `Transfer to ${agentName}`;

  return tool(
    async (_, config) => {
      // (1)!
      const state = config.state;
      const toolCallId = config.toolCall.id;

      const toolMessage = {
        role: "tool" as const,
        content: `Successfully transferred to ${agentName}`,
        name: name,
        tool_call_id: toolCallId,
      };

      return new Command({
        // (3)!
        goto: agentName,
        // (4)!
        update: { messages: [...state.messages, toolMessage] },
        // (5)!
        graph: Command.PARENT,
      });
    },
    {
      name,
      description: toolDescription,
      schema: z.object({}),
    }
  );
}
```

1. Access the [state](../concepts/low_level.md#state) of the agent that is calling the handoff tool through the `config` parameter.
2. The `Command` primitive allows specifying a state update and a node transition as a single operation, making it useful for implementing handoffs.
3. Name of the agent or node to hand off to.
4. Take the agent's messages and **add** them to the parent's **state** as part of the handoff. The next agent will see the parent state.
5. Indicate to LangGraph that we need to navigate to agent node in a **parent** multi-agent graph.

!!! tip

    If you want to use tools that return `Command`, you can either use prebuilt @[`create_react_agent`][create_react_agent] / @[`ToolNode`][ToolNode] components, or implement your own tool-executing node that collects `Command` objects returned by the tools and returns a list of them, e.g.:
    
    ```typescript
    const callTools = async (state) => {
      // ...
      const commands = await Promise.all(
        toolCalls.map(toolCall => toolsByName[toolCall.name].invoke(toolCall))
      );
      return commands;
    };
    ```
:::

!!! Important

    This handoff implementation assumes that:
    
    - each agent receives overall message history (across all agents) in the multi-agent system as its input. If you want more control over agent inputs, see [this section](#control-agent-inputs)
    - each agent outputs its internal messages history to the overall message history of the multi-agent system. If you want more control over **how agent outputs are added**, wrap the agent in a separate node function:

      :::python
      ```python
      def call_hotel_assistant(state):
          # return agent's final response,
          # excluding inner monologue
          response = hotel_assistant.invoke(state)
          # highlight-next-line
          return {"messages": response["messages"][-1]}
      ```
      :::

      :::js
      ```typescript
      const callHotelAssistant = async (state) => {
        // return agent's final response,
        // excluding inner monologue
        const response = await hotelAssistant.invoke(state);
        // highlight-next-line
        return { messages: [response.messages.at(-1)] };
      };
      ```
      :::

### Control agent inputs

:::python
You can use the @[`Send()`][Send] primitive to directly send data to the worker agents during the handoff. For example, you can request that the calling agent populate a task description for the next agent:

```python

from typing import Annotated
from langchain_core.tools import tool, InjectedToolCallId
from langgraph.prebuilt import InjectedState
from langgraph.graph import StateGraph, START, MessagesState
# highlight-next-line
from langgraph.types import Command, Send

def create_task_description_handoff_tool(
    *, agent_name: str, description: str | None = None
):
    name = f"transfer_to_{agent_name}"
    description = description or f"Ask {agent_name} for help."

    @tool(name, description=description)
    def handoff_tool(
        # this is populated by the calling agent
        task_description: Annotated[
            str,
            "Description of what the next agent should do, including all of the relevant context.",
        ],
        # these parameters are ignored by the LLM
        state: Annotated[MessagesState, InjectedState],
    ) -> Command:
        task_description_message = {"role": "user", "content": task_description}
        agent_input = {**state, "messages": [task_description_message]}
        return Command(
            # highlight-next-line
            goto=[Send(agent_name, agent_input)],
            graph=Command.PARENT,
        )

    return handoff_tool
```
:::

:::js
You can use the @[`Send()`][Send] primitive to directly send data to the worker agents during the handoff. For example, you can request that the calling agent populate a task description for the next agent:

```typescript
import { tool } from "@langchain/core/tools";
import { Command, Send, MessagesZodState } from "@langchain/langgraph";
import { z } from "zod";

function createTaskDescriptionHandoffTool({
  agentName,
  description,
}: {
  agentName: string;
  description?: string;
}) {
  const name = `transfer_to_${agentName}`;
  const toolDescription = description || `Ask ${agentName} for help.`;

  return tool(
    async (
      { taskDescription },
      config
    ) => {
      const state = config.state;
      
      const taskDescriptionMessage = {
        role: "user" as const,
        content: taskDescription,
      };
      const agentInput = {
        ...state,
        messages: [taskDescriptionMessage],
      };
      
      return new Command({
        // highlight-next-line
        goto: [new Send(agentName, agentInput)],
        graph: Command.PARENT,
      });
    },
    {
      name,
      description: toolDescription,
      schema: z.object({
        taskDescription: z
          .string()
          .describe(
            "Description of what the next agent should do, including all of the relevant context."
          ),
      }),
    }
  );
}
```
:::

See the multi-agent [supervisor](../tutorials/multi_agent/agent_supervisor.md#4-create-delegation-tasks) example for a full example of using @[`Send()`][Send] in handoffs.

## Build a multi-agent system

You can use handoffs in any agents built with LangGraph. We recommend using the prebuilt [agent](../agents/overview.md) or [`ToolNode`](./tool-calling.md#toolnode), as they natively support handoffs tools returning `Command`. Below is an example of how you can implement a multi-agent system for booking travel using handoffs:

:::python
```python
from langgraph.prebuilt import create_react_agent
from langgraph.graph import StateGraph, START, MessagesState

def create_handoff_tool(*, agent_name: str, description: str | None = None):
    # same implementation as above
    ...
    return Command(...)

# Handoffs
transfer_to_hotel_assistant = create_handoff_tool(agent_name="hotel_assistant")
transfer_to_flight_assistant = create_handoff_tool(agent_name="flight_assistant")

# Define agents
flight_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[..., transfer_to_hotel_assistant],
    # highlight-next-line
    name="flight_assistant"
)
hotel_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[..., transfer_to_flight_assistant],
    # highlight-next-line
    name="hotel_assistant"
)

# Define multi-agent graph
multi_agent_graph = (
    StateGraph(MessagesState)
    # highlight-next-line
    .add_node(flight_assistant)
    # highlight-next-line
    .add_node(hotel_assistant)
    .add_edge(START, "flight_assistant")
    .compile()
)
```
:::

:::js
```typescript
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { StateGraph, START, MessagesZodState } from "@langchain/langgraph";
import { z } from "zod";

function createHandoffTool({
  agentName,
  description,
}: {
  agentName: string;
  description?: string;
}) {
  // same implementation as above
  // ...
  return new Command(/* ... */);
}

// Handoffs
const transferToHotelAssistant = createHandoffTool({
  agentName: "hotel_assistant",
});
const transferToFlightAssistant = createHandoffTool({
  agentName: "flight_assistant",
});

// Define agents
const flightAssistant = createReactAgent({
  llm: model,
  // highlight-next-line
  tools: [/* ... */, transferToHotelAssistant],
  // highlight-next-line
  name: "flight_assistant",
});

const hotelAssistant = createReactAgent({
  llm: model,
  // highlight-next-line
  tools: [/* ... */, transferToFlightAssistant],
  // highlight-next-line
  name: "hotel_assistant",
});

// Define multi-agent graph
const multiAgentGraph = new StateGraph(MessagesZodState)
  // highlight-next-line
  .addNode("flight_assistant", flightAssistant)
  // highlight-next-line
  .addNode("hotel_assistant", hotelAssistant)
  .addEdge(START, "flight_assistant")
  .compile();
```
:::

??? example "Full example: Multi-agent system for booking travel"

    :::python
    ```python
    from typing import Annotated
    from langchain_core.messages import convert_to_messages
    from langchain_core.tools import tool, InjectedToolCallId
    from langgraph.prebuilt import create_react_agent, InjectedState
    from langgraph.graph import StateGraph, START, MessagesState
    from langgraph.types import Command
    
    # We'll use `pretty_print_messages` helper to render the streamed agent outputs nicely
    
    def pretty_print_message(message, indent=False):
        pretty_message = message.pretty_repr(html=True)
        if not indent:
            print(pretty_message)
            return
    
        indented = "\n".join("\t" + c for c in pretty_message.split("\n"))
        print(indented)
    
    
    def pretty_print_messages(update, last_message=False):
        is_subgraph = False
        if isinstance(update, tuple):
            ns, update = update
            # skip parent graph updates in the printouts
            if len(ns) == 0:
                return
    
            graph_id = ns[-1].split(":")[0]
            print(f"Update from subgraph {graph_id}:")
            print("\n")
            is_subgraph = True
    
        for node_name, node_update in update.items():
            update_label = f"Update from node {node_name}:"
            if is_subgraph:
                update_label = "\t" + update_label
    
            print(update_label)
            print("\n")
    
            messages = convert_to_messages(node_update["messages"])
            if last_message:
                messages = messages[-1:]
    
            for m in messages:
                pretty_print_message(m, indent=is_subgraph)
            print("\n")


    def create_handoff_tool(*, agent_name: str, description: str | None = None):
        name = f"transfer_to_{agent_name}"
        description = description or f"Transfer to {agent_name}"
    
        @tool(name, description=description)
        def handoff_tool(
            # highlight-next-line
            state: Annotated[MessagesState, InjectedState], # (1)!
            # highlight-next-line
            tool_call_id: Annotated[str, InjectedToolCallId],
        ) -> Command:
            tool_message = {
                "role": "tool",
                "content": f"Successfully transferred to {agent_name}",
                "name": name,
                "tool_call_id": tool_call_id,
            }
            return Command(  # (2)!
                # highlight-next-line
                goto=agent_name,  # (3)!
                # highlight-next-line
                update={"messages": state["messages"] + [tool_message]},  # (4)!
                # highlight-next-line
                graph=Command.PARENT,  # (5)!
            )
        return handoff_tool
    
    # Handoffs
    transfer_to_hotel_assistant = create_handoff_tool(
        agent_name="hotel_assistant",
        description="Transfer user to the hotel-booking assistant.",
    )
    transfer_to_flight_assistant = create_handoff_tool(
        agent_name="flight_assistant",
        description="Transfer user to the flight-booking assistant.",
    )
    
    # Simple agent tools
    def book_hotel(hotel_name: str):
        """Book a hotel"""
        return f"Successfully booked a stay at {hotel_name}."
    
    def book_flight(from_airport: str, to_airport: str):
        """Book a flight"""
        return f"Successfully booked a flight from {from_airport} to {to_airport}."
    
    # Define agents
    flight_assistant = create_react_agent(
        model="anthropic:claude-3-5-sonnet-latest",
        # highlight-next-line
        tools=[book_flight, transfer_to_hotel_assistant],
        prompt="You are a flight booking assistant",
        # highlight-next-line
        name="flight_assistant"
    )
    hotel_assistant = create_react_agent(
        model="anthropic:claude-3-5-sonnet-latest",
        # highlight-next-line
        tools=[book_hotel, transfer_to_flight_assistant],
        prompt="You are a hotel booking assistant",
        # highlight-next-line
        name="hotel_assistant"
    )
    
    # Define multi-agent graph
    multi_agent_graph = (
        StateGraph(MessagesState)
        .add_node(flight_assistant)
        .add_node(hotel_assistant)
        .add_edge(START, "flight_assistant")
        .compile()
    )
    
    # Run the multi-agent graph
    for chunk in multi_agent_graph.stream(
        {
            "messages": [
                {
                    "role": "user",
                    "content": "book a flight from BOS to JFK and a stay at McKittrick Hotel"
                }
            ]
        },
        # highlight-next-line
        subgraphs=True
    ):
        pretty_print_messages(chunk)
    ```

    1. Access agent's state
    2. The `Command` primitive allows specifying a state update and a node transition as a single operation, making it useful for implementing handoffs.
    3. Name of the agent or node to hand off to.
    4. Take the agent's messages and **add** them to the parent's **state** as part of the handoff. The next agent will see the parent state.
    5. Indicate to LangGraph that we need to navigate to agent node in a **parent** multi-agent graph.
    :::

    :::js
    ```typescript
    import { tool } from "@langchain/core/tools";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";
    import { StateGraph, START, MessagesZodState, Command } from "@langchain/langgraph";
    import { ChatAnthropic } from "@langchain/anthropic";
    import { isBaseMessage } from "@langchain/core/messages";
    import { z } from "zod";

    // We'll use a helper to render the streamed agent outputs nicely
    const prettyPrintMessages = (update: Record<string, any>) => {
      // Handle tuple case with namespace
      if (Array.isArray(update)) {
        const [ns, updateData] = update;
        // Skip parent graph updates in the printouts
        if (ns.length === 0) {
          return;
        }

        const graphId = ns[ns.length - 1].split(":")[0];
        console.log(`Update from subgraph ${graphId}:\n`);
        update = updateData;
      }

      for (const [nodeName, updateValue] of Object.entries(update)) {
        console.log(`Update from node ${nodeName}:\n`);

        const messages = updateValue.messages || [];
        for (const message of messages) {
          if (isBaseMessage(message)) {
            const textContent =
              typeof message.content === "string"
                ? message.content
                : JSON.stringify(message.content);
            console.log(`${message.getType()}: ${textContent}`);
          }
        }
        console.log("\n");
      }
    };

    function createHandoffTool({
      agentName,
      description,
    }: {
      agentName: string;
      description?: string;
    }) {
      const name = `transfer_to_${agentName}`;
      const toolDescription = description || `Transfer to ${agentName}`;

      return tool(
        async (_, config) => {
          // highlight-next-line
          const state = config.state; // (1)!
          const toolCallId = config.toolCall.id;

          const toolMessage = {
            role: "tool" as const,
            content: `Successfully transferred to ${agentName}`,
            name: name,
            tool_call_id: toolCallId,
          };

          return new Command({
            // highlight-next-line
            goto: agentName, // (3)!
            // highlight-next-line
            update: { messages: [...state.messages, toolMessage] }, // (4)!
            // highlight-next-line
            graph: Command.PARENT, // (5)!
          });
        },
        {
          name,
          description: toolDescription,
          schema: z.object({}),
        }
      );
    }

    // Handoffs
    const transferToHotelAssistant = createHandoffTool({
      agentName: "hotel_assistant",
      description: "Transfer user to the hotel-booking assistant.",
    });

    const transferToFlightAssistant = createHandoffTool({
      agentName: "flight_assistant",
      description: "Transfer user to the flight-booking assistant.",
    });

    // Simple agent tools
    const bookHotel = tool(
      async ({ hotelName }) => {
        return `Successfully booked a stay at ${hotelName}.`;
      },
      {
        name: "book_hotel",
        description: "Book a hotel",
        schema: z.object({
          hotelName: z.string(),
        }),
      }
    );

    const bookFlight = tool(
      async ({ fromAirport, toAirport }) => {
        return `Successfully booked a flight from ${fromAirport} to ${toAirport}.`;
      },
      {
        name: "book_flight",
        description: "Book a flight",
        schema: z.object({
          fromAirport: z.string(),
          toAirport: z.string(),
        }),
      }
    );

    const model = new ChatAnthropic({
      model: "claude-3-5-sonnet-latest",
    });

    // Define agents
    const flightAssistant = createReactAgent({
      llm: model,
      // highlight-next-line
      tools: [bookFlight, transferToHotelAssistant],
      prompt: "You are a flight booking assistant",
      // highlight-next-line
      name: "flight_assistant",
    });

    const hotelAssistant = createReactAgent({
      llm: model,
      // highlight-next-line
      tools: [bookHotel, transferToFlightAssistant],
      prompt: "You are a hotel booking assistant",
      // highlight-next-line
      name: "hotel_assistant",
    });

    // Define multi-agent graph
    const multiAgentGraph = new StateGraph(MessagesZodState)
      .addNode("flight_assistant", flightAssistant)
      .addNode("hotel_assistant", hotelAssistant)
      .addEdge(START, "flight_assistant")
      .compile();

    // Run the multi-agent graph
    const stream = await multiAgentGraph.stream(
      {
        messages: [
          {
            role: "user",
            content: "book a flight from BOS to JFK and a stay at McKittrick Hotel",
          },
        ],
      },
      // highlight-next-line
      { subgraphs: true }
    );

    for await (const chunk of stream) {
      prettyPrintMessages(chunk);
    }
    ```

    1. Access agent's state
    2. The `Command` primitive allows specifying a state update and a node transition as a single operation, making it useful for implementing handoffs.
    3. Name of the agent or node to hand off to.
    4. Take the agent's messages and **add** them to the parent's **state** as part of the handoff. The next agent will see the parent state.
    5. Indicate to LangGraph that we need to navigate to agent node in a **parent** multi-agent graph.
    :::

## Multi-turn conversation

Users might want to engage in a *multi-turn conversation* with one or more agents. To build a system that can handle this, you can create a node that uses an @[`interrupt`][interrupt] to collect user input and routes back to the **active** agent.

The agents can then be implemented as nodes in a graph that executes agent steps and determines the next action:

1. **Wait for user input** to continue the conversation, or  
2. **Route to another agent** (or back to itself, such as in a loop) via a [handoff](#handoffs)

:::python
```python
def human(state) -> Command[Literal["agent", "another_agent"]]:
    """A node for collecting user input."""
    user_input = interrupt(value="Ready for user input.")

    # Determine the active agent.
    active_agent = ...

    ...
    return Command(
        update={
            "messages": [{
                "role": "human",
                "content": user_input,
            }]
        },
        goto=active_agent
    )

def agent(state) -> Command[Literal["agent", "another_agent", "human"]]:
    # The condition for routing/halting can be anything, e.g. LLM tool call / structured output, etc.
    goto = get_next_agent(...)  # 'agent' / 'another_agent'
    if goto:
        return Command(goto=goto, update={"my_state_key": "my_state_value"})
    else:
        return Command(goto="human") # Go to human node
```
:::

:::js
```typescript
import { interrupt, Command } from "@langchain/langgraph";

function human(state: MessagesState): Command {
  const userInput: string = interrupt("Ready for user input.");

  // Determine the active agent
  const activeAgent = /* ... */;

  return new Command({
    update: {
      messages: [{
        role: "human",
        content: userInput,
      }]
    },
    goto: activeAgent,
  });
}

function agent(state: MessagesState): Command {
  // The condition for routing/halting can be anything, e.g. LLM tool call / structured output, etc.
  const goto = getNextAgent(/* ... */); // 'agent' / 'anotherAgent'

  if (goto) {
    return new Command({
      goto,
      update: { myStateKey: "myStateValue" }
    });
  }

  return new Command({ goto: "human" });
}
```
:::

??? example "Full example: multi-agent system for travel recommendations"

    In this example, we will build a team of travel assistant agents that can communicate with each other via handoffs.
    
    We will create 2 agents:
    
    * travel_advisor: can help with travel destination recommendations. Can ask hotel_advisor for help.
    * hotel_advisor: can help with hotel recommendations. Can ask travel_advisor for help.

    :::python
    ```python
    from langchain_anthropic import ChatAnthropic
    from langgraph.graph import MessagesState, StateGraph, START
    from langgraph.prebuilt import create_react_agent, InjectedState
    from langgraph.types import Command, interrupt
    from langgraph.checkpoint.memory import InMemorySaver
    
    
    model = ChatAnthropic(model="claude-3-5-sonnet-latest")

    class MultiAgentState(MessagesState):
        last_active_agent: str
    
    
    # Define travel advisor tools and ReAct agent
    travel_advisor_tools = [
        get_travel_recommendations,
        make_handoff_tool(agent_name="hotel_advisor"),
    ]
    travel_advisor = create_react_agent(
        model,
        travel_advisor_tools,
        prompt=(
            "You are a general travel expert that can recommend travel destinations (e.g. countries, cities, etc). "
            "If you need hotel recommendations, ask 'hotel_advisor' for help. "
            "You MUST include human-readable response before transferring to another agent."
        ),
    )
    
    
    def call_travel_advisor(
        state: MultiAgentState,
    ) -> Command[Literal["hotel_advisor", "human"]]:
        # You can also add additional logic like changing the input to the agent / output from the agent, etc.
        # NOTE: we're invoking the ReAct agent with the full history of messages in the state
        response = travel_advisor.invoke(state)
        update = {**response, "last_active_agent": "travel_advisor"}
        return Command(update=update, goto="human")
    
    
    # Define hotel advisor tools and ReAct agent
    hotel_advisor_tools = [
        get_hotel_recommendations,
        make_handoff_tool(agent_name="travel_advisor"),
    ]
    hotel_advisor = create_react_agent(
        model,
        hotel_advisor_tools,
        prompt=(
            "You are a hotel expert that can provide hotel recommendations for a given destination. "
            "If you need help picking travel destinations, ask 'travel_advisor' for help."
            "You MUST include human-readable response before transferring to another agent."
        ),
    )
    
    
    def call_hotel_advisor(
        state: MultiAgentState,
    ) -> Command[Literal["travel_advisor", "human"]]:
        response = hotel_advisor.invoke(state)
        update = {**response, "last_active_agent": "hotel_advisor"}
        return Command(update=update, goto="human")
    
    
    def human_node(
        state: MultiAgentState, config
    ) -> Command[Literal["hotel_advisor", "travel_advisor", "human"]]:
        """A node for collecting user input."""
    
        user_input = interrupt(value="Ready for user input.")
        active_agent = state["last_active_agent"]
    
        return Command(
            update={
                "messages": [
                    {
                        "role": "human",
                        "content": user_input,
                    }
                ]
            },
            goto=active_agent,
        )
    
    
    builder = StateGraph(MultiAgentState)
    builder.add_node("travel_advisor", call_travel_advisor)
    builder.add_node("hotel_advisor", call_hotel_advisor)
    
    # This adds a node to collect human input, which will route
    # back to the active agent.
    builder.add_node("human", human_node)
    
    # We'll always start with a general travel advisor.
    builder.add_edge(START, "travel_advisor")
    
    
    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)
    ```
    
    Let's test a multi turn conversation with this application.

    ```python
    import uuid
    
    thread_config = {"configurable": {"thread_id": str(uuid.uuid4())}}
    
    inputs = [
        # 1st round of conversation,
        {
            "messages": [
                {"role": "user", "content": "i wanna go somewhere warm in the caribbean"}
            ]
        },
        # Since we're using `interrupt`, we'll need to resume using the Command primitive.
        # 2nd round of conversation,
        Command(
            resume="could you recommend a nice hotel in one of the areas and tell me which area it is."
        ),
        # 3rd round of conversation,
        Command(
            resume="i like the first one. could you recommend something to do near the hotel?"
        ),
    ]
    
    for idx, user_input in enumerate(inputs):
        print()
        print(f"--- Conversation Turn {idx + 1} ---")
        print()
        print(f"User: {user_input}")
        print()
        for update in graph.stream(
            user_input,
            config=thread_config,
            stream_mode="updates",
        ):
            for node_id, value in update.items():
                if isinstance(value, dict) and value.get("messages", []):
                    last_message = value["messages"][-1]
                    if isinstance(last_message, dict) or last_message.type != "ai":
                        continue
                    print(f"{node_id}: {last_message.content}")
    ```
    
    ```
    --- Conversation Turn 1 ---
    
    User: {'messages': [{'role': 'user', 'content': 'i wanna go somewhere warm in the caribbean'}]}
    
    travel_advisor: Based on the recommendations, Aruba would be an excellent choice for your Caribbean getaway! Aruba is known as "One Happy Island" and offers:
    - Year-round warm weather with consistent temperatures around 82°F (28°C)
    - Beautiful white sand beaches like Eagle Beach and Palm Beach
    - Clear turquoise waters perfect for swimming and snorkeling
    - Minimal rainfall and location outside the hurricane belt
    - A blend of Caribbean and Dutch culture
    - Great dining options and nightlife
    - Various water sports and activities
    
    Would you like me to get some specific hotel recommendations in Aruba for your stay? I can transfer you to our hotel advisor who can help with accommodations.
    
    --- Conversation Turn 2 ---
    
    User: Command(resume='could you recommend a nice hotel in one of the areas and tell me which area it is.')
    
    hotel_advisor: Based on the recommendations, I can suggest two excellent options:
    
    1. The Ritz-Carlton, Aruba - Located in Palm Beach
    - This luxury resort is situated in the vibrant Palm Beach area
    - Known for its exceptional service and amenities
    - Perfect if you want to be close to dining, shopping, and entertainment
    - Features multiple restaurants, a casino, and a world-class spa
    - Located on a pristine stretch of Palm Beach
    
    2. Bucuti & Tara Beach Resort - Located in Eagle Beach
    - An adults-only boutique resort on Eagle Beach
    - Known for being more intimate and peaceful
    - Award-winning for its sustainability practices
    - Perfect for a romantic getaway or peaceful vacation
    - Located on one of the most beautiful beaches in the Caribbean
    
    Would you like more specific information about either of these properties or their locations?
    
    --- Conversation Turn 3 ---
    
    User: Command(resume='i like the first one. could you recommend something to do near the hotel?')
    
    travel_advisor: Near the Ritz-Carlton in Palm Beach, here are some highly recommended activities:
    
    1. Visit the Palm Beach Plaza Mall - Just a short walk from the hotel, featuring shopping, dining, and entertainment
    2. Try your luck at the Stellaris Casino - It's right in the Ritz-Carlton
    3. Take a sunset sailing cruise - Many depart from the nearby pier
    4. Visit the California Lighthouse - A scenic landmark just north of Palm Beach
    5. Enjoy water sports at Palm Beach:
       - Jet skiing
       - Parasailing
       - Snorkeling
       - Stand-up paddleboarding
    
    Would you like more specific information about any of these activities or would you like to know about other options in the area?
    ```
    :::

    :::js
    ```typescript
    import { ChatAnthropic } from "@langchain/anthropic";
    import { StateGraph, START, MessagesZodState, Command, interrupt, MemorySaver } from "@langchain/langgraph";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";
    import { tool } from "@langchain/core/tools";
    import { z } from "zod";

    const model = new ChatAnthropic({ model: "claude-3-5-sonnet-latest" });

    const MultiAgentState = MessagesZodState.extend({
      lastActiveAgent: z.string().optional(),
    });

    // Define travel advisor tools
    const getTravelRecommendations = tool(
      async () => {
        // Placeholder implementation
        return "Based on current trends, I recommend visiting Japan, Portugal, or New Zealand.";
      },
      {
        name: "get_travel_recommendations",
        description: "Get current travel destination recommendations",
        schema: z.object({}),
      }
    );

    const makeHandoffTool = (agentName: string) => {
      return tool(
        async (_, config) => {
          const state = config.state;
          const toolCallId = config.toolCall.id;

          const toolMessage = {
            role: "tool" as const,
            content: `Successfully transferred to ${agentName}`,
            name: `transfer_to_${agentName}`,
            tool_call_id: toolCallId,
          };

          return new Command({
            goto: agentName,
            update: { messages: [...state.messages, toolMessage] },
            graph: Command.PARENT,
          });
        },
        {
          name: `transfer_to_${agentName}`,
          description: `Transfer to ${agentName}`,
          schema: z.object({}),
        }
      );
    };

    const travelAdvisorTools = [
      getTravelRecommendations,
      makeHandoffTool("hotel_advisor"),
    ];

    const travelAdvisor = createReactAgent({
      llm: model,
      tools: travelAdvisorTools,
      prompt: [
        "You are a general travel expert that can recommend travel destinations (e.g. countries, cities, etc). ",
        "If you need hotel recommendations, ask 'hotel_advisor' for help. ",
        "You MUST include human-readable response before transferring to another agent."
      ].join("")
    });

    const callTravelAdvisor = async (
      state: z.infer<typeof MultiAgentState>
    ): Promise<Command> => {
      const response = await travelAdvisor.invoke(state);
      const update = { ...response, lastActiveAgent: "travel_advisor" };
      return new Command({ update, goto: "human" });
    };

    // Define hotel advisor tools
    const getHotelRecommendations = tool(
      async () => {
        // Placeholder implementation
        return "I recommend the Ritz-Carlton for luxury stays or boutique hotels for unique experiences.";
      },
      {
        name: "get_hotel_recommendations",
        description: "Get hotel recommendations for destinations",
        schema: z.object({}),
      }
    );

    const hotelAdvisorTools = [
      getHotelRecommendations,
      makeHandoffTool("travel_advisor"),
    ];

    const hotelAdvisor = createReactAgent({
      llm: model,
      tools: hotelAdvisorTools,
      prompt: [
        "You are a hotel expert that can provide hotel recommendations for a given destination. ",
        "If you need help picking travel destinations, ask 'travel_advisor' for help.",
        "You MUST include human-readable response before transferring to another agent."
      ].join("")
    });

    const callHotelAdvisor = async (
      state: z.infer<typeof MultiAgentState>
    ): Promise<Command> => {
      const response = await hotelAdvisor.invoke(state);
      const update = { ...response, lastActiveAgent: "hotel_advisor" };
      return new Command({ update, goto: "human" });
    };

    const humanNode = async (
      state: z.infer<typeof MultiAgentState>
    ): Promise<Command> => {
      const userInput: string = interrupt("Ready for user input.");
      const activeAgent = state.lastActiveAgent || "travel_advisor";

      return new Command({
        update: {
          messages: [
            {
              role: "human",
              content: userInput,
            }
          ]
        },
        goto: activeAgent,
      });
    };

    const builder = new StateGraph(MultiAgentState)
      .addNode("travel_advisor", callTravelAdvisor)
      .addNode("hotel_advisor", callHotelAdvisor)
      .addNode("human", humanNode)
      .addEdge(START, "travel_advisor");

    const checkpointer = new MemorySaver();
    const graph = builder.compile({ checkpointer });
    ```
    
    Let's test a multi turn conversation with this application.

    ```typescript
    import { v4 as uuidv4 } from "uuid";
    import { Command } from "@langchain/langgraph";

    const threadConfig = { configurable: { thread_id: uuidv4() } };

    const inputs = [
      // 1st round of conversation
      {
        messages: [
          { role: "user", content: "i wanna go somewhere warm in the caribbean" }
        ]
      },
      // Since we're using `interrupt`, we'll need to resume using the Command primitive.
      // 2nd round of conversation
      new Command({
        resume: "could you recommend a nice hotel in one of the areas and tell me which area it is."
      }),
      // 3rd round of conversation
      new Command({
        resume: "i like the first one. could you recommend something to do near the hotel?"
      }),
    ];

    for (const [idx, userInput] of inputs.entries()) {
      console.log();
      console.log(`--- Conversation Turn ${idx + 1} ---`);
      console.log();
      console.log(`User: ${JSON.stringify(userInput)}`);
      console.log();
      
      for await (const update of await graph.stream(
        userInput,
        { ...threadConfig, streamMode: "updates" }
      )) {
        for (const [nodeId, value] of Object.entries(update)) {
          if (value?.messages?.length) {
            const lastMessage = value.messages.at(-1);
            if (lastMessage?.getType?.() === "ai") {
              console.log(`${nodeId}: ${lastMessage.content}`);
            }
          }
        }
      }
    }
    ```
    
    ```
    --- Conversation Turn 1 ---
    
    User: {"messages":[{"role":"user","content":"i wanna go somewhere warm in the caribbean"}]}
    
    travel_advisor: Based on the recommendations, Aruba would be an excellent choice for your Caribbean getaway! Aruba is known as "One Happy Island" and offers:
    - Year-round warm weather with consistent temperatures around 82°F (28°C)
    - Beautiful white sand beaches like Eagle Beach and Palm Beach
    - Clear turquoise waters perfect for swimming and snorkeling
    - Minimal rainfall and location outside the hurricane belt
    - A blend of Caribbean and Dutch culture
    - Great dining options and nightlife
    - Various water sports and activities
    
    Would you like me to get some specific hotel recommendations in Aruba for your stay? I can transfer you to our hotel advisor who can help with accommodations.
    
    --- Conversation Turn 2 ---
    
    User: Command { resume: 'could you recommend a nice hotel in one of the areas and tell me which area it is.' }
    
    hotel_advisor: Based on the recommendations, I can suggest two excellent options:
    
    1. The Ritz-Carlton, Aruba - Located in Palm Beach
    - This luxury resort is situated in the vibrant Palm Beach area
    - Known for its exceptional service and amenities
    - Perfect if you want to be close to dining, shopping, and entertainment
    - Features multiple restaurants, a casino, and a world-class spa
    - Located on a pristine stretch of Palm Beach
    
    2. Bucuti & Tara Beach Resort - Located in Eagle Beach
    - An adults-only boutique resort on Eagle Beach
    - Known for being more intimate and peaceful
    - Award-winning for its sustainability practices
    - Perfect for a romantic getaway or peaceful vacation
    - Located on one of the most beautiful beaches in the Caribbean
    
    Would you like more specific information about either of these properties or their locations?
    
    --- Conversation Turn 3 ---
    
    User: Command { resume: 'i like the first one. could you recommend something to do near the hotel?' }
    
    travel_advisor: Near the Ritz-Carlton in Palm Beach, here are some highly recommended activities:
    
    1. Visit the Palm Beach Plaza Mall - Just a short walk from the hotel, featuring shopping, dining, and entertainment
    2. Try your luck at the Stellaris Casino - It's right in the Ritz-Carlton
    3. Take a sunset sailing cruise - Many depart from the nearby pier
    4. Visit the California Lighthouse - A scenic landmark just north of Palm Beach
    5. Enjoy water sports at Palm Beach:
       - Jet skiing
       - Parasailing
       - Snorkeling
       - Stand-up paddleboarding
    
    Would you like more specific information about any of these activities or would you like to know about other options in the area?
    ```
    :::

## Prebuilt implementations

LangGraph comes with prebuilt implementations of two of the most popular multi-agent architectures:

:::python
- [supervisor](../agents/multi-agent.md#supervisor) — individual agents are coordinated by a central supervisor agent. The supervisor controls all communication flow and task delegation, making decisions about which agent to invoke based on the current context and task requirements. You can use [`langgraph-supervisor`](https://github.com/langchain-ai/langgraph-supervisor-py) library to create a supervisor multi-agent systems.
- [swarm](../agents/multi-agent.md#supervisor) — agents dynamically hand off control to one another based on their specializations. The system remembers which agent was last active, ensuring that on subsequent interactions, the conversation resumes with that agent. You can use [`langgraph-swarm`](https://github.com/langchain-ai/langgraph-swarm-py) library to create a swarm multi-agent systems.
:::

:::js
- [supervisor](../agents/multi-agent.md#supervisor) — individual agents are coordinated by a central supervisor agent. The supervisor controls all communication flow and task delegation, making decisions about which agent to invoke based on the current context and task requirements. You can use [`langgraph-supervisor`](https://github.com/langchain-ai/langgraph-supervisor-js) library to create a supervisor multi-agent systems.
- [swarm](../agents/multi-agent.md#supervisor) — agents dynamically hand off control to one another based on their specializations. The system remembers which agent was last active, ensuring that on subsequent interactions, the conversation resumes with that agent. You can use [`langgraph-swarm`](https://github.com/langchain-ai/langgraph-swarm-js) library to create a swarm multi-agent systems.
:::
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/persistence-functional.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "51466c8d-8ce4-4b3d-be4e-18fdbeda5f53",
   "metadata": {},
   "source": [
    "# How to add thread-level persistence (functional API)\n",
    "\n",
    "!!! info \"Prerequisites\"\n",
    "\n",
    "    This guide assumes familiarity with the following:\n",
    "    \n",
    "    - [Functional API](../../concepts/functional_api/)\n",
    "    - [Persistence](../../concepts/persistence/)\n",
    "    - [Memory](../../concepts/memory/)\n",
    "    - [Chat Models](https://python.langchain.com/docs/concepts/chat_models/)\n",
    "\n",
    "!!! info \"Not needed for LangGraph API users\"\n",
    "\n",
    "    If you're using the LangGraph API, you needn't manually implement a checkpointer. The API automatically handles checkpointing for you. This guide is relevant when implementing LangGraph in your own custom server.\n",
    "\n",
    "Many AI applications need memory to share context across multiple interactions on the same [thread](../../concepts/persistence#threads) (e.g., multiple turns of a conversation). In LangGraph functional API, this kind of memory can be added to any [entrypoint()][langgraph.func.entrypoint] workflow using [thread-level persistence](https://langchain-ai.github.io/langgraph/concepts/persistence).\n",
    "\n",
    "When creating a LangGraph workflow, you can set it up to persist its results by using a [checkpointer](https://langchain-ai.github.io/langgraph/reference/checkpoints/#basecheckpointsaver):\n",
    "\n",
    "\n",
    "1. Create an instance of a checkpointer:\n",
    "\n",
    "    ```python\n",
    "    from langgraph.checkpoint.memory import InMemorySaver\n",
    "    \n",
    "    checkpointer = InMemorySaver()       \n",
    "    ```\n",
    "\n",
    "2. Pass `checkpointer` instance to the `entrypoint()` decorator:\n",
    "\n",
    "    ```python\n",
    "    from langgraph.func import entrypoint\n",
    "    \n",
    "    @entrypoint(checkpointer=checkpointer)\n",
    "    def workflow(inputs)\n",
    "        ...\n",
    "    ```\n",
    "\n",
    "3. Optionally expose `previous` parameter in the workflow function signature:\n",
    "\n",
    "    ```python\n",
    "    @entrypoint(checkpointer=checkpointer)\n",
    "    def workflow(\n",
    "        inputs,\n",
    "        *,\n",
    "        # you can optionally specify `previous` in the workflow function signature\n",
    "        # to access the return value from the workflow as of the last execution\n",
    "        previous\n",
    "    ):\n",
    "        previous = previous or []\n",
    "        combined_inputs = previous + inputs\n",
    "        result = do_something(combined_inputs)\n",
    "        ...\n",
    "    ```\n",
    "\n",
    "4. Optionally choose which values will be returned from the workflow and which will be saved by the checkpointer as `previous`:\n",
    "\n",
    "    ```python\n",
    "    @entrypoint(checkpointer=checkpointer)\n",
    "    def workflow(inputs, *, previous):\n",
    "        ...\n",
    "        result = do_something(...)\n",
    "        return entrypoint.final(value=result, save=combine(inputs, result))\n",
    "    ```\n",
    "\n",
    "This guide shows how you can add thread-level persistence to your workflow.\n",
    "\n",
    "!!! tip \"Note\"\n",
    "\n",
    "    If you need memory that is __shared__ across multiple conversations or users (cross-thread persistence), check out this [how-to guide](../cross-thread-persistence-functional).\n",
    "\n",
    "!!! tip \"Note\"\n",
    "\n",
    "    If you need to add thread-level persistence to a `StateGraph`, check out this [how-to guide](../persistence)."
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
    "Next, we need to set API key for Anthropic (the LLM we will use)."
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
   "id": "4cf509bc",
   "metadata": {},
   "source": [
    "## Example: simple chatbot with short-term memory\n",
    "\n",
    "We will be using a workflow with a single task that calls a [chat model](https://python.langchain.com/docs/concepts/chat_models/).\n",
    "\n",
    "Let's first define the model we'll be using:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 3,
   "id": "892b54b9-75f0-4804-9ed0-88b5e5532989",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langchain_anthropic import ChatAnthropic\n",
    "\n",
    "model = ChatAnthropic(model=\"claude-3-5-sonnet-latest\")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "7b7a2792-982b-4e47-83eb-0c594725d1c1",
   "metadata": {},
   "source": [
    "Now we can define our task and workflow. To add in persistence, we need to pass in a [Checkpointer](https://langchain-ai.github.io/langgraph/reference/checkpoints/#langgraph.checkpoint.base.BaseCheckpointSaver) to the [entrypoint()][langgraph.func.entrypoint] decorator."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 4,
   "id": "87326ea6-34c5-46da-a41f-dda26ef9bd74",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langchain_core.messages import BaseMessage\n",
    "from langgraph.graph import add_messages\n",
    "from langgraph.func import entrypoint, task\n",
    "from langgraph.checkpoint.memory import InMemorySaver\n",
    "\n",
    "\n",
    "@task\n",
    "def call_model(messages: list[BaseMessage]):\n",
    "    response = model.invoke(messages)\n",
    "    return response\n",
    "\n",
    "\n",
    "checkpointer = InMemorySaver()\n",
    "\n",
    "\n",
    "@entrypoint(checkpointer=checkpointer)\n",
    "def workflow(inputs: list[BaseMessage], *, previous: list[BaseMessage]):\n",
    "    if previous:\n",
    "        inputs = add_messages(previous, inputs)\n",
    "\n",
    "    response = call_model(inputs).result()\n",
    "    return entrypoint.final(value=response, save=add_messages(inputs, response))"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "250d8fd9-2e7a-4892-9adc-19762a1e3cce",
   "metadata": {},
   "source": [
    "If we try to use this workflow, the context of the conversation will be persisted across interactions:"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "7654ebcc-2179-41b4-92d1-6666f6f8634f",
   "metadata": {},
   "source": [
    "!!! note Note\n",
    "\n",
    "    If you're using LangGraph Platform or LangGraph Studio, you __don't need__ to pass checkpointer to the entrypoint decorator, since it's done automatically."
   ]
  },
  {
   "cell_type": "markdown",
   "id": "2a1b56c5-bd61-4192-8bdb-458a1e9f0159",
   "metadata": {},
   "source": [
    "We can now interact with the agent and see that it remembers previous messages!"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 5,
   "id": "cfd140f0-a5a6-4697-8115-322242f197b5",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "Hi Bob! I'm Claude. Nice to meet you! How are you today?\n"
     ]
    }
   ],
   "source": [
    "config = {\"configurable\": {\"thread_id\": \"1\"}}\n",
    "input_message = {\"role\": \"user\", \"content\": \"hi! I'm bob\"}\n",
    "for chunk in workflow.stream([input_message], config, stream_mode=\"values\"):\n",
    "    chunk.pretty_print()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "1bb07bf8-68b7-4049-a0f1-eb67a4879a3a",
   "metadata": {},
   "source": [
    "You can always resume previous threads:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 6,
   "id": "08ae8246-11d5-40e1-8567-361e5bef8917",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "Your name is Bob.\n"
     ]
    }
   ],
   "source": [
    "input_message = {\"role\": \"user\", \"content\": \"what's my name?\"}\n",
    "for chunk in workflow.stream([input_message], config, stream_mode=\"values\"):\n",
    "    chunk.pretty_print()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "3f47bbfc-d9ef-4288-ba4a-ebbc0136fa9d",
   "metadata": {},
   "source": [
    "If we want to start a new conversation, we can pass in a different `thread_id`. Poof! All the memories are gone!"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 7,
   "id": "273d56a8-f40f-4a51-a27f-7c6bb2bda0ba",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "I don't know your name unless you tell me. Each conversation I have starts fresh, so I don't have access to any previous interactions or personal information unless you share it with me.\n"
     ]
    }
   ],
   "source": [
    "input_message = {\"role\": \"user\", \"content\": \"what's my name?\"}\n",
    "for chunk in workflow.stream(\n",
    "    [input_message],\n",
    "    {\"configurable\": {\"thread_id\": \"2\"}},\n",
    "    stream_mode=\"values\",\n",
    "):\n",
    "    chunk.pretty_print()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "ac7926a8-4c88-4b16-973c-53d6da3f4a08",
   "metadata": {},
   "source": [
    "!!! tip \"Streaming tokens\"\n",
    "\n",
    "    If you would like to stream LLM tokens from your chatbot, you can use `stream_mode=\"messages\"`. Check out this [how-to guide](../streaming-tokens) to learn more."
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
 "nbformat_minor": 5
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/react-agent-from-scratch-functional.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# How to create a ReAct agent from scratch (Functional API)\n",
    "\n",
    "!!! info \"Prerequisites\"\n",
    "    This guide assumes familiarity with the following:\n",
    "    \n",
    "    - [Chat Models](https://python.langchain.com/docs/concepts/chat_models)\n",
    "    - [Messages](https://python.langchain.com/docs/concepts/messages)\n",
    "    - [Tool Calling](https://python.langchain.com/docs/concepts/tool_calling/)\n",
    "    - [Entrypoints](../../concepts/functional_api/#entrypoint) and [Tasks](../../concepts/functional_api/#task)\n",
    "\n",
    "This guide demonstrates how to implement a ReAct agent using the LangGraph [Functional API](../../concepts/functional_api).\n",
    "\n",
    "The ReAct agent is a [tool-calling agent](../../concepts/agentic_concepts/#tool-calling-agent) that operates as follows:\n",
    "\n",
    "1. Queries are issued to a chat model;\n",
    "2. If the model generates no [tool calls](../../concepts/agentic_concepts/#tool-calling), we return the model response.\n",
    "3. If the model generates tool calls, we execute the tool calls with available tools, append them as [tool messages](https://python.langchain.com/docs/concepts/messages/) to our message list, and repeat the process.\n",
    "\n",
    "This is a simple and versatile set-up that can be extended with memory, human-in-the-loop capabilities, and other features. See the dedicated [how-to guides](../../how-tos/#prebuilt-react-agent) for examples.\n",
    "\n",
    "## Setup\n",
    "\n",
    "First, let's install the required packages and set our API keys:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 1,
   "metadata": {},
   "outputs": [],
   "source": [
    "%%capture --no-stderr\n",
    "%pip install -U langgraph langchain-openai"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 2,
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
    "<div class=\"admonition tip\">\n",
    "     <p class=\"admonition-title\">Set up <a href=\"https://smith.langchain.com\">LangSmith</a> for better debugging</p>\n",
    "     <p style=\"padding-top: 5px;\">\n",
    "         Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. LangSmith lets you use trace data to debug, test, and monitor your LLM aps built with LangGraph — read more about how to get started in the <a href=\"https://docs.smith.langchain.com\">docs</a>. \n",
    "     </p>\n",
    " </div>"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## Create ReAct agent\n",
    "\n",
    "Now that you have installed the required packages and set your environment variables, we can create our agent.\n",
    "\n",
    "### Define model and tools\n",
    "\n",
    "Let's first define the tools and model we will use for our example. Here we will use a single place-holder tool that gets a description of the weather for a location.\n",
    "\n",
    "We will use an [OpenAI](https://python.langchain.com/docs/integrations/providers/openai/) chat model for this example, but any model [supporting tool-calling](https://python.langchain.com/docs/integrations/chat/) will suffice."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 1,
   "metadata": {},
   "outputs": [],
   "source": [
    "from langchain_openai import ChatOpenAI\n",
    "from langchain_core.tools import tool\n",
    "\n",
    "model = ChatOpenAI(model=\"gpt-4o-mini\")\n",
    "\n",
    "\n",
    "@tool\n",
    "def get_weather(location: str):\n",
    "    \"\"\"Call to get the weather from a specific location.\"\"\"\n",
    "    # This is a placeholder for the actual implementation\n",
    "    if any([city in location.lower() for city in [\"sf\", \"san francisco\"]]):\n",
    "        return \"It's sunny!\"\n",
    "    elif \"boston\" in location.lower():\n",
    "        return \"It's rainy!\"\n",
    "    else:\n",
    "        return f\"I am not sure what the weather is in {location}\"\n",
    "\n",
    "\n",
    "tools = [get_weather]"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### Define tasks\n",
    "\n",
    "We next define the [tasks](../../concepts/functional_api/#task) we will execute. Here there are two different tasks:\n",
    "\n",
    "1. **Call model**: We want to query our chat model with a list of messages.\n",
    "2. **Call tool**: If our model generates tool calls, we want to execute them."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 2,
   "metadata": {},
   "outputs": [],
   "source": [
    "from langchain_core.messages import ToolMessage\n",
    "from langgraph.func import entrypoint, task\n",
    "\n",
    "tools_by_name = {tool.name: tool for tool in tools}\n",
    "\n",
    "\n",
    "@task\n",
    "def call_model(messages):\n",
    "    \"\"\"Call model with a sequence of messages.\"\"\"\n",
    "    response = model.bind_tools(tools).invoke(messages)\n",
    "    return response\n",
    "\n",
    "\n",
    "@task\n",
    "def call_tool(tool_call):\n",
    "    tool = tools_by_name[tool_call[\"name\"]]\n",
    "    observation = tool.invoke(tool_call[\"args\"])\n",
    "    return ToolMessage(content=observation, tool_call_id=tool_call[\"id\"])"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### Define entrypoint\n",
    "\n",
    "Our [entrypoint](../../concepts/functional_api/#entrypoint) will handle the orchestration of these two tasks. As described above, when our `call_model` task generates tool calls, the `call_tool` task will generate responses for each. We append all messages to a single messages list.\n",
    "\n",
    "!!! tip\n",
    "    Note that because tasks return future-like objects, the below implementation executes tools in parallel."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 3,
   "metadata": {},
   "outputs": [],
   "source": [
    "from langgraph.graph.message import add_messages\n",
    "\n",
    "\n",
    "@entrypoint()\n",
    "def agent(messages):\n",
    "    llm_response = call_model(messages).result()\n",
    "    while True:\n",
    "        if not llm_response.tool_calls:\n",
    "            break\n",
    "\n",
    "        # Execute tools\n",
    "        tool_result_futures = [\n",
    "            call_tool(tool_call) for tool_call in llm_response.tool_calls\n",
    "        ]\n",
    "        tool_results = [fut.result() for fut in tool_result_futures]\n",
    "\n",
    "        # Append to message list\n",
    "        messages = add_messages(messages, [llm_response, *tool_results])\n",
    "\n",
    "        # Call model again\n",
    "        llm_response = call_model(messages).result()\n",
    "\n",
    "    return llm_response"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## Usage\n",
    "\n",
    "To use our agent, we invoke it with a messages list. Based on our implementation, these can be LangChain [message](https://python.langchain.com/docs/concepts/messages/) objects or OpenAI-style dicts:"
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
      "{'role': 'user', 'content': \"What's the weather in san francisco?\"}\n",
      "\n",
      "call_model:\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "Tool Calls:\n",
      "  get_weather (call_tNnkrjnoz6MNfCHJpwfuEQ0v)\n",
      " Call ID: call_tNnkrjnoz6MNfCHJpwfuEQ0v\n",
      "  Args:\n",
      "    location: san francisco\n",
      "\n",
      "call_tool:\n",
      "=================================\u001b[1m Tool Message \u001b[0m=================================\n",
      "\n",
      "It's sunny!\n",
      "\n",
      "call_model:\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "The weather in San Francisco is sunny!\n"
     ]
    }
   ],
   "source": [
    "user_message = {\"role\": \"user\", \"content\": \"What's the weather in san francisco?\"}\n",
    "print(user_message)\n",
    "\n",
    "for step in agent.stream([user_message]):\n",
    "    for task_name, message in step.items():\n",
    "        if task_name == \"agent\":\n",
    "            continue  # Just print task updates\n",
    "        print(f\"\\n{task_name}:\")\n",
    "        message.pretty_print()"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "Perfect! The graph correctly calls the `get_weather` tool and responds to the user after receiving the information from the tool. Check out the LangSmith trace [here](https://smith.langchain.com/public/d5a0d5ea-bdaa-4032-911e-7db177c8141b/r)."
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## Add thread-level persistence\n",
    "\n",
    "Adding [thread-level persistence](../../concepts/persistence#threads) lets us support conversational experiences with our agent: subsequent invocations will append to the prior messages list, retaining the full conversational context.\n",
    "\n",
    "To add thread-level persistence to our agent:\n",
    "\n",
    "1. Select a [checkpointer](../../concepts/persistence#checkpointer-libraries): here we will use [InMemorySaver](../../reference/checkpoints/#langgraph.checkpoint.memory.InMemorySaver), a simple in-memory checkpointer.\n",
    "2. Update our entrypoint to accept the previous messages state as a second argument. Here, we simply append the message updates to the previous sequence of messages.\n",
    "3. Choose which values will be returned from the workflow and which will be saved by the checkpointer as `previous` using `entrypoint.final` (optional)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 5,
   "metadata": {},
   "outputs": [],
   "source": [
    "from langgraph.checkpoint.memory import InMemorySaver\n",
    "\n",
    "# highlight-next-line\n",
    "checkpointer = InMemorySaver()\n",
    "\n",
    "\n",
    "# highlight-next-line\n",
    "@entrypoint(checkpointer=checkpointer)\n",
    "# highlight-next-line\n",
    "def agent(messages, previous):\n",
    "    # highlight-next-line\n",
    "    if previous is not None:\n",
    "        # highlight-next-line\n",
    "        messages = add_messages(previous, messages)\n",
    "\n",
    "    llm_response = call_model(messages).result()\n",
    "    while True:\n",
    "        if not llm_response.tool_calls:\n",
    "            break\n",
    "\n",
    "        # Execute tools\n",
    "        tool_result_futures = [\n",
    "            call_tool(tool_call) for tool_call in llm_response.tool_calls\n",
    "        ]\n",
    "        tool_results = [fut.result() for fut in tool_result_futures]\n",
    "\n",
    "        # Append to message list\n",
    "        messages = add_messages(messages, [llm_response, *tool_results])\n",
    "\n",
    "        # Call model again\n",
    "        llm_response = call_model(messages).result()\n",
    "\n",
    "    # Generate final response\n",
    "    messages = add_messages(messages, llm_response)\n",
    "    # highlight-next-line\n",
    "    return entrypoint.final(value=llm_response, save=messages)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "We will now need to pass in a config when running our application. The config will specify an identifier for the conversational thread.\n",
    "\n",
    "!!! tip\n",
    "\n",
    "    Read more about thread-level persistence in our [concepts page](../../concepts/persistence/) and [how-to guides](../../how-tos/#persistence)."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 6,
   "metadata": {},
   "outputs": [],
   "source": [
    "config = {\"configurable\": {\"thread_id\": \"1\"}}"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "We start a thread the same way as before, this time passing in the config:"
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
      "{'role': 'user', 'content': \"What's the weather in san francisco?\"}\n",
      "\n",
      "call_model:\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "Tool Calls:\n",
      "  get_weather (call_lubbUSdDofmOhFunPEZLBz3g)\n",
      " Call ID: call_lubbUSdDofmOhFunPEZLBz3g\n",
      "  Args:\n",
      "    location: San Francisco\n",
      "\n",
      "call_tool:\n",
      "=================================\u001b[1m Tool Message \u001b[0m=================================\n",
      "\n",
      "It's sunny!\n",
      "\n",
      "call_model:\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "The weather in San Francisco is sunny!\n"
     ]
    }
   ],
   "source": [
    "user_message = {\"role\": \"user\", \"content\": \"What's the weather in san francisco?\"}\n",
    "print(user_message)\n",
    "\n",
    "# highlight-next-line\n",
    "for step in agent.stream([user_message], config):\n",
    "    for task_name, message in step.items():\n",
    "        if task_name == \"agent\":\n",
    "            continue  # Just print task updates\n",
    "        print(f\"\\n{task_name}:\")\n",
    "        message.pretty_print()"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "When we ask a follow-up conversation, the model uses the prior context to infer that we are asking about the weather:"
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
      "{'role': 'user', 'content': 'How does it compare to Boston, MA?'}\n",
      "\n",
      "call_model:\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "Tool Calls:\n",
      "  get_weather (call_8sTKYAhSIHOdjLD5d6gaswuV)\n",
      " Call ID: call_8sTKYAhSIHOdjLD5d6gaswuV\n",
      "  Args:\n",
      "    location: Boston, MA\n",
      "\n",
      "call_tool:\n",
      "=================================\u001b[1m Tool Message \u001b[0m=================================\n",
      "\n",
      "It's rainy!\n",
      "\n",
      "call_model:\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "Compared to San Francisco, which is sunny, Boston, MA is experiencing rainy weather.\n"
     ]
    }
   ],
   "source": [
    "user_message = {\"role\": \"user\", \"content\": \"How does it compare to Boston, MA?\"}\n",
    "print(user_message)\n",
    "\n",
    "for step in agent.stream([user_message], config):\n",
    "    for task_name, message in step.items():\n",
    "        if task_name == \"agent\":\n",
    "            continue  # Just print task updates\n",
    "        print(f\"\\n{task_name}:\")\n",
    "        message.pretty_print()"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "In the [LangSmith trace](https://smith.langchain.com/public/20a1116b-bb3b-44c1-8765-7a28663439d9/r), we can see that the full conversational context is retained in each model call."
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
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/react-agent-from-scratch.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# How to create a ReAct agent from scratch\n",
    "\n",
    "!!! info \"Prerequisites\"\n",
    "    This guide assumes familiarity with the following:\n",
    "    \n",
    "    - [Tool calling agent](../../concepts/agentic_concepts/#tool-calling-agent)\n",
    "    - [Chat Models](https://python.langchain.com/docs/concepts/chat_models/)\n",
    "    - [Messages](https://python.langchain.com/docs/concepts/messages/)\n",
    "    - [LangGraph Glossary](../../concepts/low_level/)\n",
    "\n",
    "Using the prebuilt ReAct agent [create_react_agent][langgraph.prebuilt.chat_agent_executor.create_react_agent] is a great way to get started, but sometimes you might want more control and customization. In those cases, you can create a custom ReAct agent. This guide shows how to implement ReAct agent from scratch using LangGraph.\n",
    "\n",
    "## Setup\n",
    "\n",
    "First, let's install the required packages and set our API keys:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 1,
   "metadata": {},
   "outputs": [],
   "source": [
    "%%capture --no-stderr\n",
    "%pip install -U langgraph langchain-openai"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 2,
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
    "<div class=\"admonition tip\">\n",
    "     <p class=\"admonition-title\">Set up <a href=\"https://smith.langchain.com\">LangSmith</a> for better debugging</p>\n",
    "     <p style=\"padding-top: 5px;\">\n",
    "         Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. LangSmith lets you use trace data to debug, test, and monitor your LLM aps built with LangGraph — read more about how to get started in the <a href=\"https://docs.smith.langchain.com\">docs</a>. \n",
    "     </p>\n",
    " </div>"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## Create ReAct agent\n",
    "\n",
    "Now that you have installed the required packages and set your environment variables, we can code our ReAct agent!\n",
    "\n",
    "### Define graph state\n",
    "\n",
    "We are going to define the most basic ReAct state in this example, which will just contain a list of messages.\n",
    "\n",
    "For your specific use case, feel free to add any other state keys that you need."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 4,
   "metadata": {},
   "outputs": [],
   "source": [
    "from typing import (\n",
    "    Annotated,\n",
    "    Sequence,\n",
    "    TypedDict,\n",
    ")\n",
    "from langchain_core.messages import BaseMessage\n",
    "from langgraph.graph.message import add_messages\n",
    "\n",
    "\n",
    "class AgentState(TypedDict):\n",
    "    \"\"\"The state of the agent.\"\"\"\n",
    "\n",
    "    # add_messages is a reducer\n",
    "    # See https://langchain-ai.github.io/langgraph/concepts/low_level/#reducers\n",
    "    messages: Annotated[Sequence[BaseMessage], add_messages]"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### Define model and tools\n",
    "\n",
    "Next, let's define the tools and model we will use for our example."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 5,
   "metadata": {},
   "outputs": [],
   "source": [
    "from langchain_openai import ChatOpenAI\n",
    "from langchain_core.tools import tool\n",
    "\n",
    "model = ChatOpenAI(model=\"gpt-4o-mini\")\n",
    "\n",
    "\n",
    "@tool\n",
    "def get_weather(location: str):\n",
    "    \"\"\"Call to get the weather from a specific location.\"\"\"\n",
    "    # This is a placeholder for the actual implementation\n",
    "    # Don't let the LLM know this though 😊\n",
    "    if any([city in location.lower() for city in [\"sf\", \"san francisco\"]]):\n",
    "        return \"It's sunny in San Francisco, but you better look out if you're a Gemini 😈.\"\n",
    "    else:\n",
    "        return f\"I am not sure what the weather is in {location}\"\n",
    "\n",
    "\n",
    "tools = [get_weather]\n",
    "\n",
    "model = model.bind_tools(tools)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### Define nodes and edges\n",
    "\n",
    "Next let's define our nodes and edges. In our basic ReAct agent there are only two nodes, one for calling the model and one for using tools, however you can modify this basic structure to work better for your use case. The tool node we define here is a simplified version of the prebuilt [`ToolNode`](https://langchain-ai.github.io/langgraph/how-tos/tool-calling/), which has some additional features.\n",
    "\n",
    "Perhaps you want to add a node for [adding structured output](https://langchain-ai.github.io/langgraph/how-tos/react-agent-structured-output/) or a node for executing some external action (sending an email, adding a calendar event, etc.). Maybe you just want to change the way the `call_model` node works and how `should_continue` decides whether to call tools - the possibilities are endless and LangGraph makes it easy to customize this basic structure for your specific use case."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 6,
   "metadata": {},
   "outputs": [],
   "source": [
    "import json\n",
    "from langchain_core.messages import ToolMessage, SystemMessage\n",
    "from langchain_core.runnables import RunnableConfig\n",
    "\n",
    "tools_by_name = {tool.name: tool for tool in tools}\n",
    "\n",
    "\n",
    "# Define our tool node\n",
    "def tool_node(state: AgentState):\n",
    "    outputs = []\n",
    "    for tool_call in state[\"messages\"][-1].tool_calls:\n",
    "        tool_result = tools_by_name[tool_call[\"name\"]].invoke(tool_call[\"args\"])\n",
    "        outputs.append(\n",
    "            ToolMessage(\n",
    "                content=json.dumps(tool_result),\n",
    "                name=tool_call[\"name\"],\n",
    "                tool_call_id=tool_call[\"id\"],\n",
    "            )\n",
    "        )\n",
    "    return {\"messages\": outputs}\n",
    "\n",
    "\n",
    "# Define the node that calls the model\n",
    "def call_model(\n",
    "    state: AgentState,\n",
    "    config: RunnableConfig,\n",
    "):\n",
    "    # this is similar to customizing the create_react_agent with 'prompt' parameter, but is more flexible\n",
    "    system_prompt = SystemMessage(\n",
    "        \"You are a helpful AI assistant, please respond to the users query to the best of your ability!\"\n",
    "    )\n",
    "    response = model.invoke([system_prompt] + state[\"messages\"], config)\n",
    "    # We return a list, because this will get added to the existing list\n",
    "    return {\"messages\": [response]}\n",
    "\n",
    "\n",
    "# Define the conditional edge that determines whether to continue or not\n",
    "def should_continue(state: AgentState):\n",
    "    messages = state[\"messages\"]\n",
    "    last_message = messages[-1]\n",
    "    # If there is no function call, then we finish\n",
    "    if not last_message.tool_calls:\n",
    "        return \"end\"\n",
    "    # Otherwise if there is, we continue\n",
    "    else:\n",
    "        return \"continue\""
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "### Define the graph\n",
    "\n",
    "Now that we have defined all of our nodes and edges, we can define and compile our graph. Depending on if you have added more nodes or different edges, you will need to edit this to fit your specific use case."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 7,
   "metadata": {},
   "outputs": [
    {
     "data": {
      "image/jpeg": "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",
      "text/plain": [
       "<IPython.core.display.Image object>"
      ]
     },
     "metadata": {},
     "output_type": "display_data"
    }
   ],
   "source": [
    "from langgraph.graph import StateGraph, END\n",
    "\n",
    "# Define a new graph\n",
    "workflow = StateGraph(AgentState)\n",
    "\n",
    "# Define the two nodes we will cycle between\n",
    "workflow.add_node(\"agent\", call_model)\n",
    "workflow.add_node(\"tools\", tool_node)\n",
    "\n",
    "# Set the entrypoint as `agent`\n",
    "# This means that this node is the first one called\n",
    "workflow.set_entry_point(\"agent\")\n",
    "\n",
    "# We now add a conditional edge\n",
    "workflow.add_conditional_edges(\n",
    "    # First, we define the start node. We use `agent`.\n",
    "    # This means these are the edges taken after the `agent` node is called.\n",
    "    \"agent\",\n",
    "    # Next, we pass in the function that will determine which node is called next.\n",
    "    should_continue,\n",
    "    # Finally we pass in a mapping.\n",
    "    # The keys are strings, and the values are other nodes.\n",
    "    # END is a special node marking that the graph should finish.\n",
    "    # What will happen is we will call `should_continue`, and then the output of that\n",
    "    # will be matched against the keys in this mapping.\n",
    "    # Based on which one it matches, that node will then be called.\n",
    "    {\n",
    "        # If `tools`, then we call the tool node.\n",
    "        \"continue\": \"tools\",\n",
    "        # Otherwise we finish.\n",
    "        \"end\": END,\n",
    "    },\n",
    ")\n",
    "\n",
    "# We now add a normal edge from `tools` to `agent`.\n",
    "# This means that after `tools` is called, `agent` node is called next.\n",
    "workflow.add_edge(\"tools\", \"agent\")\n",
    "\n",
    "# Now we can compile and visualize our graph\n",
    "graph = workflow.compile()\n",
    "\n",
    "from IPython.display import Image, display\n",
    "\n",
    "try:\n",
    "    display(Image(graph.get_graph().draw_mermaid_png()))\n",
    "except Exception:\n",
    "    # This requires some extra dependencies and is optional\n",
    "    pass"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## Use ReAct agent\n",
    "\n",
    "Now that we have created our react agent, let's actually put it to the test!"
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
      "================================\u001b[1m Human Message \u001b[0m=================================\n",
      "\n",
      "what is the weather in sf\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "Tool Calls:\n",
      "  get_weather (call_azW0cQ4XjWWj0IAkWAxq9nLB)\n",
      " Call ID: call_azW0cQ4XjWWj0IAkWAxq9nLB\n",
      "  Args:\n",
      "    location: San Francisco\n",
      "=================================\u001b[1m Tool Message \u001b[0m=================================\n",
      "Name: get_weather\n",
      "\n",
      "\"It's sunny in San Francisco, but you better look out if you're a Gemini \\ud83d\\ude08.\"\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "The weather in San Francisco is sunny! However, it seems there's a playful warning for Geminis. Enjoy the sunshine!\n"
     ]
    }
   ],
   "source": [
    "# Helper function for formatting the stream nicely\n",
    "def print_stream(stream):\n",
    "    for s in stream:\n",
    "        message = s[\"messages\"][-1]\n",
    "        if isinstance(message, tuple):\n",
    "            print(message)\n",
    "        else:\n",
    "            message.pretty_print()\n",
    "\n",
    "\n",
    "inputs = {\"messages\": [(\"user\", \"what is the weather in sf\")]}\n",
    "print_stream(graph.stream(inputs, stream_mode=\"values\"))"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "Perfect! The graph correctly calls the `get_weather` tool and responds to the user after receiving the information from the tool."
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
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/react-agent-structured-output.ipynb
```ipynb
{
 "cells": [
  {
   "attachments": {
    "59e8ed35-f2b4-421e-8d21-880e7ab31e5f.png": {
