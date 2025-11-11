```
> [truncated]

---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/multi_agent/assets/diagram.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/multi_agent/assets/multi-output.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/multi_agent/assets/output.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/self-discover/self-discover.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "a38e5d2d-7587-4192-90f2-b58e6c62f08c",
   "metadata": {},
   "source": [
    "# Self-Discover Agent\n",
    "\n",
    "An implementation of the [Self-Discover paper](https://arxiv.org/pdf/2402.03620.pdf).\n",
    "\n",
    "Based on [this implementation from @catid](https://github.com/catid/self-discover/tree/main?tab=readme-ov-file)\n",
    "\n",
    "\n",
    "## Setup\n",
    "\n",
    "First, let's install our required packages and set our API keys"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "2811c3da",
   "metadata": {},
   "outputs": [],
   "source": [
    "%%capture --no-stderr\n",
    "%pip install -U --quiet langchain langgraph langchain_openai"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "5e66899a",
   "metadata": {},
   "outputs": [],
   "source": [
    "import getpass\n",
    "import os\n",
    "\n",
    "\n",
    "def _set_if_undefined(var: str) -> None:\n",
    "    if os.environ.get(var):\n",
    "        return\n",
    "    os.environ[var] = getpass.getpass(var)\n",
    "\n",
    "\n",
    "_set_if_undefined(\"OPENAI_API_KEY\")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "35dce921",
   "metadata": {},
   "source": [
    "<div class=\"admonition tip\">\n",
    "    <p class=\"admonition-title\">Set up <a href=\"https://smith.langchain.com\">LangSmith</a> for LangGraph development</p>\n",
    "    <p style=\"padding-top: 5px;\">\n",
    "        Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. LangSmith lets you use trace data to debug, test, and monitor your LLM apps built with LangGraph — read more about how to get started <a href=\"https://docs.smith.langchain.com\">here</a>. \n",
    "    </p>\n",
    "</div>   "
   ]
  },
  {
   "cell_type": "markdown",
   "id": "35b1729e",
   "metadata": {},
   "source": [
    "## Define the prompts"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 1,
   "id": "a18d8f24-5d9a-45c5-9739-6f3c4ed6c9c9",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Self-Discovery Select Prompt:\n",
      "Select several reasoning modules that are crucial to utilize in order to solve the given task:\n",
      "\n",
      "All reasoning module descriptions:\n",
      "\u001b[33;1m\u001b[1;3m{reasoning_modules}\u001b[0m\n",
      "\n",
      "Task: \u001b[33;1m\u001b[1;3m{task_description}\u001b[0m\n",
      "\n",
      "Select several modules are crucial for solving the task above:\n",
      "\n",
      "Self-Discovery Select Response:\n",
      "Rephrase and specify each reasoning module so that it better helps solving the task:\n",
      "\n",
      "SELECTED module descriptions:\n",
      "\u001b[33;1m\u001b[1;3m{selected_modules}\u001b[0m\n",
      "\n",
      "Task: \u001b[33;1m\u001b[1;3m{task_description}\u001b[0m\n",
      "\n",
      "Adapt each reasoning module description to better solve the task:\n",
      "\n",
      "Self-Discovery Structured Prompt:\n",
      "Operationalize the reasoning modules into a step-by-step reasoning plan in JSON format:\n",
      "\n",
      "Here's an example:\n",
      "\n",
      "Example task:\n",
      "\n",
      "If you follow these instructions, do you return to the starting point? Always face forward. Take 1 step backward. Take 9 steps left. Take 2 steps backward. Take 6 steps forward. Take 4 steps forward. Take 4 steps backward. Take 3 steps right.\n",
      "\n",
      "Example reasoning structure:\n",
      "\n",
      "{\n",
      "    \"Position after instruction 1\":\n",
      "    \"Position after instruction 2\":\n",
      "    \"Position after instruction n\":\n",
      "    \"Is final position the same as starting position\":\n",
      "}\n",
      "\n",
      "Adapted module description:\n",
      "\u001b[33;1m\u001b[1;3m{adapted_modules}\u001b[0m\n",
      "\n",
      "Task: \u001b[33;1m\u001b[1;3m{task_description}\u001b[0m\n",
      "\n",
      "Implement a reasoning structure for solvers to follow step-by-step and arrive at correct answer.\n",
      "\n",
      "Note: do NOT actually arrive at a conclusion in this pass. Your job is to generate a PLAN so that in the future you can fill it out and arrive at the correct conclusion for tasks like this\n",
      "Self-Discovery Structured Response:\n",
      "Follow the step-by-step reasoning plan in JSON to correctly solve the task. Fill in the values following the keys by reasoning specifically about the task given. Do not simply rephrase the keys.\n",
      "    \n",
      "Reasoning Structure:\n",
      "\u001b[33;1m\u001b[1;3m{reasoning_structure}\u001b[0m\n",
      "\n",
      "Task: \u001b[33;1m\u001b[1;3m{task_description}\u001b[0m\n"
     ]
    }
   ],
   "source": [
    "from langchain import hub\n",
    "\n",
    "select_prompt = hub.pull(\"hwchase17/self-discovery-select\")\n",
    "print(\"Self-Discovery Select Prompt:\")\n",
    "select_prompt.pretty_print()\n",
    "print(\"Self-Discovery Select Response:\")\n",
    "adapt_prompt = hub.pull(\"hwchase17/self-discovery-adapt\")\n",
    "adapt_prompt.pretty_print()\n",
    "structured_prompt = hub.pull(\"hwchase17/self-discovery-structure\")\n",
    "print(\"Self-Discovery Structured Prompt:\")\n",
    "structured_prompt.pretty_print()\n",
    "reasoning_prompt = hub.pull(\"hwchase17/self-discovery-reasoning\")\n",
    "print(\"Self-Discovery Structured Response:\")\n",
    "reasoning_prompt.pretty_print()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "bce1135e",
   "metadata": {},
   "source": [
    "## Define the graph"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 2,
   "id": "9f554045-6e79-42d3-be4b-835bbbd0b78c",
   "metadata": {},
   "outputs": [],
   "source": [
    "from typing import Optional\n",
    "from typing_extensions import TypedDict\n",
    "\n",
    "from langchain_core.output_parsers import StrOutputParser\n",
    "from langchain_openai import ChatOpenAI\n",
    "\n",
    "from langgraph.graph import END, START, StateGraph\n",
    "\n",
    "\n",
    "class SelfDiscoverState(TypedDict):\n",
    "    reasoning_modules: str\n",
    "    task_description: str\n",
    "    selected_modules: Optional[str]\n",
    "    adapted_modules: Optional[str]\n",
    "    reasoning_structure: Optional[str]\n",
    "    answer: Optional[str]\n",
    "\n",
    "\n",
    "model = ChatOpenAI(temperature=0, model=\"gpt-4-turbo-preview\")\n",
    "\n",
    "\n",
    "def select(inputs):\n",
    "    select_chain = select_prompt | model | StrOutputParser()\n",
    "    return {\"selected_modules\": select_chain.invoke(inputs)}\n",
    "\n",
    "\n",
    "def adapt(inputs):\n",
    "    adapt_chain = adapt_prompt | model | StrOutputParser()\n",
    "    return {\"adapted_modules\": adapt_chain.invoke(inputs)}\n",
    "\n",
    "\n",
    "def structure(inputs):\n",
    "    structure_chain = structured_prompt | model | StrOutputParser()\n",
    "    return {\"reasoning_structure\": structure_chain.invoke(inputs)}\n",
    "\n",
    "\n",
    "def reason(inputs):\n",
    "    reasoning_chain = reasoning_prompt | model | StrOutputParser()\n",
    "    return {\"answer\": reasoning_chain.invoke(inputs)}\n",
    "\n",
    "\n",
    "graph = StateGraph(SelfDiscoverState)\n",
    "graph.add_node(select)\n",
    "graph.add_node(adapt)\n",
    "graph.add_node(structure)\n",
    "graph.add_node(reason)\n",
    "graph.add_edge(START, \"select\")\n",
    "graph.add_edge(\"select\", \"adapt\")\n",
    "graph.add_edge(\"adapt\", \"structure\")\n",
    "graph.add_edge(\"structure\", \"reason\")\n",
    "graph.add_edge(\"reason\", END)\n",
    "app = graph.compile()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "29fe385b-cf5d-4581-80e7-55462f5628bb",
   "metadata": {},
   "source": [
    "## Invoke the graph"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 3,
   "id": "6cbfbe81-f751-42da-843a-f9003ace663d",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "{'select': {'selected_modules': 'To solve the task of identifying the shape drawn by the SVG path element, the following reasoning modules are crucial:\\n\\n1. **Critical Thinking (10):** This involves analyzing the provided SVG path commands to understand how they contribute to forming a shape. It requires questioning assumptions (e.g., not assuming the shape is simple or common) and evaluating the information given in the path data.\\n\\n2. **Creative Thinking (11):** While the task seems straightforward, creative thinking can help in visualizing the shape described by the path commands without immediately drawing it. This involves imagining the transitions and connections between the points defined in the path.\\n\\n3. **Systems Thinking (13):** Understanding the SVG path as a system of coordinates and lines that connect to form a shape. This includes recognizing the interconnectedness of the start and end points of each line segment and how they contribute to the overall shape.\\n\\n4. **Analytical Problem Solving (29):** This task requires data analysis skills to interpret the SVG path commands and deduce the shape they form. Analyzing the coordinates and the movements (lines and moves) can reveal the structure of the shape.\\n\\n5. **Design Challenge (30):** Interpreting and visualizing SVG paths can be seen as a design challenge, requiring an understanding of how individual parts (line segments) come together to create a whole (shape).\\n\\n6. **Step-by-Step Planning and Implementation (39):** Formulating a plan to sequentially interpret each segment of the SVG path and understanding how each segment contributes to the overall shape. This could involve sketching the path based on the commands to better visualize the shape.\\n\\nThese modules collectively enable a comprehensive approach to solving the task, from understanding and analyzing the SVG path data to creatively and systematically deducing the shape it represents.'}}\n",
      "{'adapt': {'adapted_modules': \"To enhance the process of identifying the shape drawn by the SVG path element, the reasoning modules can be adapted and specified as follows:\\n\\n1. **Enhanced Critical Analysis (10):** This module focuses on a detailed examination of the SVG path commands, challenging initial perceptions and critically assessing each command's role in shaping the figure. It involves a deep dive into the syntax and semantics of the path data, ensuring no detail is overlooked, especially in recognizing less obvious or complex shapes.\\n\\n2. **Visual Creative Thinking (11):** Leveraging imagination to mentally construct the shape from the path commands, this module emphasizes the ability to visualize the sequential flow and connection of points without physical drawing. It encourages innovative approaches to mentally piecing together the described shape, enhancing the ability to predict the outcome based on abstract data.\\n\\n3. **Integrated Systems Analysis (13):** This module treats the SVG path as a complex system where each command and coordinate plays a critical role in the final shape. It focuses on understanding the relationship between individual path segments and their collective contribution to forming a coherent structure, emphasizing the holistic view of the path's construction.\\n\\n4. **Targeted Analytical Problem Solving (29):** Specializing in dissecting the SVG path's commands to systematically uncover the represented shape, this module applies precise analytical techniques to decode the sequence of movements and coordinates. It involves a methodical breakdown of the path data to reveal the underlying geometric figure.\\n\\n5. **Design Synthesis Challenge (30):** Approaching the task as a problem of synthesizing a coherent design from segmented inputs, this module requires an adept understanding of how discrete line segments interconnect to form a unified shape. It challenges one to think like a designer, piecing together the puzzle of path commands into a complete and recognizable form.\\n\\n6. **Sequential Interpretation and Visualization (39):** This module involves developing a step-by-step strategy for interpreting and visualizing the SVG path, focusing on the incremental construction of the shape from the path commands. It advocates for a systematic approach to translating the abstract commands into a tangible visual representation, potentially through sketching or mentally mapping the path's progression.\\n\\nBy refining these modules, the approach to solving the task becomes more targeted, enhancing the ability to accurately identify the shape described by the SVG path element.\"}}\n"
     ]
    }
   ],
   "source": [
    "reasoning_modules = [\n",
    "    \"1. How could I devise an experiment to help solve that problem?\",\n",
    "    \"2. Make a list of ideas for solving this problem, and apply them one by one to the problem to see if any progress can be made.\",\n",
    "    # \"3. How could I measure progress on this problem?\",\n",
    "    \"4. How can I simplify the problem so that it is easier to solve?\",\n",
    "    \"5. What are the key assumptions underlying this problem?\",\n",
    "    \"6. What are the potential risks and drawbacks of each solution?\",\n",
    "    \"7. What are the alternative perspectives or viewpoints on this problem?\",\n",
    "    \"8. What are the long-term implications of this problem and its solutions?\",\n",
    "    \"9. How can I break down this problem into smaller, more manageable parts?\",\n",
    "    \"10. Critical Thinking: This style involves analyzing the problem from different perspectives, questioning assumptions, and evaluating the evidence or information available. It focuses on logical reasoning, evidence-based decision-making, and identifying potential biases or flaws in thinking.\",\n",
    "    \"11. Try creative thinking, generate innovative and out-of-the-box ideas to solve the problem. Explore unconventional solutions, thinking beyond traditional boundaries, and encouraging imagination and originality.\",\n",
    "    # \"12. Seek input and collaboration from others to solve the problem. Emphasize teamwork, open communication, and leveraging the diverse perspectives and expertise of a group to come up with effective solutions.\",\n",
    "    \"13. Use systems thinking: Consider the problem as part of a larger system and understanding the interconnectedness of various elements. Focuses on identifying the underlying causes, feedback loops, and interdependencies that influence the problem, and developing holistic solutions that address the system as a whole.\",\n",
    "    \"14. Use Risk Analysis: Evaluate potential risks, uncertainties, and tradeoffs associated with different solutions or approaches to a problem. Emphasize assessing the potential consequences and likelihood of success or failure, and making informed decisions based on a balanced analysis of risks and benefits.\",\n",
    "    # \"15. Use Reflective Thinking: Step back from the problem, take the time for introspection and self-reflection. Examine personal biases, assumptions, and mental models that may influence problem-solving, and being open to learning from past experiences to improve future approaches.\",\n",
    "    \"16. What is the core issue or problem that needs to be addressed?\",\n",
    "    \"17. What are the underlying causes or factors contributing to the problem?\",\n",
    "    \"18. Are there any potential solutions or strategies that have been tried before? If yes, what were the outcomes and lessons learned?\",\n",
    "    \"19. What are the potential obstacles or challenges that might arise in solving this problem?\",\n",
    "    \"20. Are there any relevant data or information that can provide insights into the problem? If yes, what data sources are available, and how can they be analyzed?\",\n",
    "    \"21. Are there any stakeholders or individuals who are directly affected by the problem? What are their perspectives and needs?\",\n",
    "    \"22. What resources (financial, human, technological, etc.) are needed to tackle the problem effectively?\",\n",
    "    \"23. How can progress or success in solving the problem be measured or evaluated?\",\n",
    "    \"24. What indicators or metrics can be used?\",\n",
    "    \"25. Is the problem a technical or practical one that requires a specific expertise or skill set? Or is it more of a conceptual or theoretical problem?\",\n",
    "    \"26. Does the problem involve a physical constraint, such as limited resources, infrastructure, or space?\",\n",
    "    \"27. Is the problem related to human behavior, such as a social, cultural, or psychological issue?\",\n",
    "    \"28. Does the problem involve decision-making or planning, where choices need to be made under uncertainty or with competing objectives?\",\n",
    "    \"29. Is the problem an analytical one that requires data analysis, modeling, or optimization techniques?\",\n",
    "    \"30. Is the problem a design challenge that requires creative solutions and innovation?\",\n",
    "    \"31. Does the problem require addressing systemic or structural issues rather than just individual instances?\",\n",
    "    \"32. Is the problem time-sensitive or urgent, requiring immediate attention and action?\",\n",
    "    \"33. What kinds of solution typically are produced for this kind of problem specification?\",\n",
    "    \"34. Given the problem specification and the current best solution, have a guess about other possible solutions.\"\n",
    "    \"35. Let’s imagine the current best solution is totally wrong, what other ways are there to think about the problem specification?\"\n",
    "    \"36. What is the best way to modify this current best solution, given what you know about these kinds of problem specification?\"\n",
    "    \"37. Ignoring the current best solution, create an entirely new solution to the problem.\"\n",
    "    # \"38. Let’s think step by step.\"\n",
    "    \"39. Let’s make a step by step plan and implement it with good notation and explanation.\",\n",
    "]\n",
    "\n",
    "\n",
    "task_example = \"Lisa has 10 apples. She gives 3 apples to her friend and then buys 5 more apples from the store. How many apples does Lisa have now?\"\n",
    "\n",
    "task_example = \"\"\"This SVG path element <path d=\"M 55.57,80.69 L 57.38,65.80 M 57.38,65.80 L 48.90,57.46 M 48.90,57.46 L\n",
    "45.58,47.78 M 45.58,47.78 L 53.25,36.07 L 66.29,48.90 L 78.69,61.09 L 55.57,80.69\"/> draws a:\n",
    "(A) circle (B) heptagon (C) hexagon (D) kite (E) line (F) octagon (G) pentagon(H) rectangle (I) sector (J) triangle\"\"\"\n",
    "\n",
    "reasoning_modules_str = \"\\n\".join(reasoning_modules)\n",
    "\n",
    "for s in app.stream(\n",
    "    {\"task_description\": task_example, \"reasoning_modules\": reasoning_modules_str}\n",
    "):\n",
    "    print(s)"
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
   "version": "3.11.9"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 5
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/get-started/1-build-basic-chatbot.md
```md
# Build a basic chatbot

In this tutorial, you will build a basic chatbot. This chatbot is the basis for the following series of tutorials where you will progressively add more sophisticated capabilities, and be introduced to key LangGraph concepts along the way. Let's dive in! 🌟

## Prerequisites

Before you start this tutorial, ensure you have access to a LLM that supports
tool-calling features, such as [OpenAI](https://platform.openai.com/api-keys),
[Anthropic](https://console.anthropic.com/settings/keys), or
[Google Gemini](https://ai.google.dev/gemini-api/docs/api-key).

## 1. Install packages

Install the required packages:

:::python

```bash
pip install -U langgraph langsmith
```

:::

:::js
=== "npm"

    ```bash
    npm install @langchain/langgraph @langchain/core zod
    ```

=== "yarn"

    ```bash
    yarn add @langchain/langgraph @langchain/core zod
    ```

=== "pnpm"

    ```bash
    pnpm add @langchain/langgraph @langchain/core zod
    ```

=== "bun"

    ```bash
    bun add @langchain/langgraph @langchain/core zod
    ```

:::

!!! tip

    Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. LangSmith lets you use trace data to debug, test, and monitor your LLM apps built with LangGraph. For more information on how to get started, see [LangSmith docs](https://docs.smith.langchain.com).

## 2. Create a `StateGraph`

Now you can create a basic chatbot using LangGraph. This chatbot will respond directly to user messages.

Start by creating a `StateGraph`. A `StateGraph` object defines the structure of our chatbot as a "state machine". We'll add `nodes` to represent the llm and functions our chatbot can call and `edges` to specify how the bot should transition between these functions.

:::python

```python
from typing import Annotated

from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages


class State(TypedDict):
    # Messages have the type "list". The `add_messages` function
    # in the annotation defines how this state key should be updated
    # (in this case, it appends messages to the list, rather than overwriting them)
    messages: Annotated[list, add_messages]


graph_builder = StateGraph(State)
```

:::

:::js

```typescript
import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({ messages: MessagesZodState.shape.messages });

const graph = new StateGraph(State).compile();
```

:::

Our graph can now handle two key tasks:

1. Each `node` can receive the current `State` as input and output an update to the state.
2. Updates to `messages` will be appended to the existing list rather than overwriting it, thanks to the prebuilt reducer function.

!!! tip "Concept"

    When defining a graph, the first step is to define its `State`. The `State` includes the graph's schema and [reducer functions](https://langchain-ai.github.io/langgraph/concepts/low_level/#reducers) that handle state updates. In our example, `State` is a schema with one key: `messages`. The reducer function is used to append new messages to the list instead of overwriting it. Keys without a reducer annotation will overwrite previous values.

    To learn more about state, reducers, and related concepts, see [LangGraph reference docs](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.message.add_messages).

## 3. Add a node

Next, add a "`chatbot`" node. **Nodes** represent units of work and are typically regular functions.

Let's first select a chat model:

:::python

{% include-markdown "../../../snippets/chat_model_tabs.md" %}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

:::

:::js

```typescript
import { ChatOpenAI } from "@langchain/openai";
// or import { ChatAnthropic } from "@langchain/anthropic";

const llm = new ChatOpenAI({
  model: "gpt-4o",
  temperature: 0,
});
```

:::

We can now incorporate the chat model into a simple node:

:::python

```python

def chatbot(state: State):
    return {"messages": [llm.invoke(state["messages"])]}


# The first argument is the unique node name
# The second argument is the function or object that will be called whenever
# the node is used.
graph_builder.add_node("chatbot", chatbot)
```

:::

:::js

```typescript hl_lines="7-9"
import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({ messages: MessagesZodState.shape.messages });

const graph = new StateGraph(State)
  .addNode("chatbot", async (state: z.infer<typeof State>) => {
    return { messages: [await llm.invoke(state.messages)] };
  })
  .compile();
```

:::

**Notice** how the `chatbot` node function takes the current `State` as input and returns a dictionary containing an updated `messages` list under the key "messages". This is the basic pattern for all LangGraph node functions.

:::python
The `add_messages` function in our `State` will append the LLM's response messages to whatever messages are already in the state.
:::

:::js
The `addMessages` function used within `MessagesZodState` will append the LLM's response messages to whatever messages are already in the state.
:::

## 4. Add an `entry` point

Add an `entry` point to tell the graph **where to start its work** each time it is run:

:::python

```python
graph_builder.add_edge(START, "chatbot")
```

:::

:::js

```typescript hl_lines="10"
import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({ messages: MessagesZodState.shape.messages });

const graph = new StateGraph(State)
  .addNode("chatbot", async (state: z.infer<typeof State>) => {
    return { messages: [await llm.invoke(state.messages)] };
  })
  .addEdge(START, "chatbot")
  .compile();
```

:::

## 5. Add an `exit` point

Add an `exit` point to indicate **where the graph should finish execution**. This is helpful for more complex flows, but even in a simple graph like this, adding an end node improves clarity.

:::python

```python
graph_builder.add_edge("chatbot", END)
```

:::

:::js

```typescript hl_lines="11"
import { StateGraph, MessagesZodState, START, END } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({ messages: MessagesZodState.shape.messages });

const graph = new StateGraph(State)
  .addNode("chatbot", async (state: z.infer<typeof State>) => {
    return { messages: [await llm.invoke(state.messages)] };
  })
  .addEdge(START, "chatbot")
  .addEdge("chatbot", END)
  .compile();
```

:::

This tells the graph to terminate after running the chatbot node.

## 6. Compile the graph

Before running the graph, we'll need to compile it. We can do so by calling `compile()`
on the graph builder. This creates a `CompiledGraph` we can invoke on our state.

:::python

```python
graph = graph_builder.compile()
```

:::

:::js

```typescript hl_lines="12"
import { StateGraph, MessagesZodState, START, END } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({ messages: MessagesZodState.shape.messages });

const graph = new StateGraph(State)
  .addNode("chatbot", async (state: z.infer<typeof State>) => {
    return { messages: [await llm.invoke(state.messages)] };
  })
  .addEdge(START, "chatbot")
  .addEdge("chatbot", END)
  .compile();
```

:::

## 7. Visualize the graph (optional)

:::python
You can visualize the graph using the `get_graph` method and one of the "draw" methods, like `draw_ascii` or `draw_png`. The `draw` methods each require additional dependencies.

```python
from IPython.display import Image, display

try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    # This requires some extra dependencies and is optional
    pass
```

:::

:::js
You can visualize the graph using the `getGraph` method and render the graph with the `drawMermaidPng` method.

```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("basic-chatbot.png", imageBuffer);
```

:::

![basic chatbot diagram](basic-chatbot.png)

## 8. Run the chatbot

Now run the chatbot!

!!! tip

    You can exit the chat loop at any time by typing `quit`, `exit`, or `q`.

:::python

```python
def stream_graph_updates(user_input: str):
    for event in graph.stream({"messages": [{"role": "user", "content": user_input}]}):
        for value in event.values():
            print("Assistant:", value["messages"][-1].content)


while True:
    try:
        user_input = input("User: ")
        if user_input.lower() in ["quit", "exit", "q"]:
            print("Goodbye!")
            break
        stream_graph_updates(user_input)
    except:
        # fallback if input() is not available
        user_input = "What do you know about LangGraph?"
        print("User: " + user_input)
        stream_graph_updates(user_input)
        break
```

:::

:::js

```typescript
import { HumanMessage } from "@langchain/core/messages";

async function streamGraphUpdates(userInput: string) {
  const stream = await graph.stream({
    messages: [new HumanMessage(userInput)],
  });

import * as readline from "node:readline/promises";
import { StateGraph, MessagesZodState, START, END } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { z } from "zod";

const llm = new ChatOpenAI({ model: "gpt-4o-mini" });

const State = z.object({ messages: MessagesZodState.shape.messages });

const graph = new StateGraph(State)
  .addNode("chatbot", async (state: z.infer<typeof State>) => {
    return { messages: [await llm.invoke(state.messages)] };
  })
  .addEdge(START, "chatbot")
  .addEdge("chatbot", END)
  .compile();

async function generateText(content: string) {
  const stream = await graph.stream(
    { messages: [{ type: "human", content }] },
    { streamMode: "values" }
  );

  for await (const event of stream) {
    for (const value of Object.values(event)) {
      console.log(
        "Assistant:",
        value.messages[value.messages.length - 1].content
      );
    const lastMessage = event.messages.at(-1);
    if (lastMessage?.getType() === "ai") {
      console.log(`Assistant: ${lastMessage.text}`);
    }
  }
}

const prompt = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

while (true) {
  const human = await prompt.question("User: ");
  if (["quit", "exit", "q"].includes(human.trim())) break;
  await generateText(human || "What do you know about LangGraph?");
}

prompt.close();
```

:::

```
Assistant: LangGraph is a library designed to help build stateful multi-agent applications using language models. It provides tools for creating workflows and state machines to coordinate multiple AI agents or language model interactions. LangGraph is built on top of LangChain, leveraging its components while adding graph-based coordination capabilities. It's particularly useful for developing more complex, stateful AI applications that go beyond simple query-response interactions.
```

:::python

```
Goodbye!
```

:::

**Congratulations!** You've built your first chatbot using LangGraph. This bot can engage in basic conversation by taking user input and generating responses using an LLM. You can inspect a [LangSmith Trace](https://smith.langchain.com/public/7527e308-9502-4894-b347-f34385740d5a/r) for the call above.

:::python

Below is the full code for this tutorial:

```python
from typing import Annotated

from langchain.chat_models import init_chat_model
from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages


class State(TypedDict):
    messages: Annotated[list, add_messages]


graph_builder = StateGraph(State)


llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")


def chatbot(state: State):
    return {"messages": [llm.invoke(state["messages"])]}


# The first argument is the unique node name
# The second argument is the function or object that will be called whenever
# the node is used.
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_edge(START, "chatbot")
graph_builder.add_edge("chatbot", END)
graph = graph_builder.compile()
```

:::

:::js

```typescript
import { StateGraph, START, END, MessagesZodState } from "@langchain/langgraph";
import { z } from "zod";
import { ChatOpenAI } from "@langchain/openai";

const llm = new ChatOpenAI({
  model: "gpt-4o",
  temperature: 0,
});

const State = z.object({ messages: MessagesZodState.shape.messages });

const graph = new StateGraph(State);
  // The first argument is the unique node name
  // The second argument is the function or object that will be called whenever
  // the node is used.
  .addNode("chatbot", async (state) => {
    return { messages: [await llm.invoke(state.messages)] };
  });
  .addEdge(START, "chatbot");
  .addEdge("chatbot", END)
  .compile();
```

:::

## Next steps

You may have noticed that the bot's knowledge is limited to what's in its training data. In the next part, we'll [add a web search tool](./2-add-tools.md) to expand the bot's knowledge and make it more capable.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/get-started/2-add-tools.md
```md
# Add tools

To handle queries that your chatbot can't answer "from memory", integrate a web search tool. The chatbot can use this tool to find relevant information and provide better responses.

!!! note

    This tutorial builds on [Build a basic chatbot](./1-build-basic-chatbot.md).

## Prerequisites

Before you start this tutorial, ensure you have the following:

:::python

- An API key for the [Tavily Search Engine](https://python.langchain.com/docs/integrations/tools/tavily_search/).

:::

:::js

- An API key for the [Tavily Search Engine](https://js.langchain.com/docs/integrations/tools/tavily_search/).

:::

## 1. Install the search engine

:::python
Install the requirements to use the [Tavily Search Engine](https://python.langchain.com/docs/integrations/tools/tavily_search/):

```bash
pip install -U langchain-tavily
```

:::

:::js
Install the requirements to use the [Tavily Search Engine](https://docs.tavily.com/):

=== "npm"

    ```bash
    npm install @langchain/tavily
    ```

=== "yarn"

    ```bash
    yarn add @langchain/tavily
    ```

=== "pnpm"

    ```bash
    pnpm add @langchain/tavily
    ```

=== "bun"

    ```bash
    bun add @langchain/tavily
    ```

:::

## 2. Configure your environment

Configure your environment with your search engine API key:

:::python
```python
import os

os.environ["TAVILY_API_KEY"] = "tvly-..."
```
:::

:::js

```typescript
process.env.TAVILY_API_KEY = "tvly-...";
```

:::

## 3. Define the tool

Define the web search tool:

:::python

```python
from langchain_tavily import TavilySearch

tool = TavilySearch(max_results=2)
tools = [tool]
tool.invoke("What's a 'node' in LangGraph?")
```

:::

:::js

```typescript
import { TavilySearch } from "@langchain/tavily";

const tool = new TavilySearch({ maxResults: 2 });
const tools = [tool];

await tool.invoke({ query: "What's a 'node' in LangGraph?" });
```

:::

The results are page summaries our chat bot can use to answer questions:

:::python

```
{'query': "What's a 'node' in LangGraph?",
'follow_up_questions': None,
'answer': None,
'images': [],
'results': [{'title': "Introduction to LangGraph: A Beginner's Guide - Medium",
'url': 'https://medium.com/@cplog/introduction-to-langgraph-a-beginners-guide-14f9be027141',
'content': 'Stateful Graph: LangGraph revolves around the concept of a stateful graph, where each node in the graph represents a step in your computation, and the graph maintains a state that is passed around and updated as the computation progresses. LangGraph supports conditional edges, allowing you to dynamically determine the next node to execute based on the current state of the graph. We define nodes for classifying the input, handling greetings, and handling search queries. def classify_input_node(state): LangGraph is a versatile tool for building complex, stateful applications with LLMs. By understanding its core concepts and working through simple examples, beginners can start to leverage its power for their projects. Remember to pay attention to state management, conditional edges, and ensuring there are no dead-end nodes in your graph.',
'score': 0.7065353,
'raw_content': None},
{'title': 'LangGraph Tutorial: What Is LangGraph and How to Use It?',
'url': 'https://www.datacamp.com/tutorial/langgraph-tutorial',
'content': 'LangGraph is a library within the LangChain ecosystem that provides a framework for defining, coordinating, and executing multiple LLM agents (or chains) in a structured and efficient manner. By managing the flow of data and the sequence of operations, LangGraph allows developers to focus on the high-level logic of their applications rather than the intricacies of agent coordination. Whether you need a chatbot that can handle various types of user requests or a multi-agent system that performs complex tasks, LangGraph provides the tools to build exactly what you need. LangGraph significantly simplifies the development of complex LLM applications by providing a structured framework for managing state and coordinating agent interactions.',
'score': 0.5008063,
'raw_content': None}],
'response_time': 1.38}
```

:::

:::js

```json
{
  "query": "What's a 'node' in LangGraph?",
  "follow_up_questions": null,
  "answer": null,
  "images": [],
  "results": [
    {
      "url": "https://blog.langchain.dev/langgraph/",
      "title": "LangGraph - LangChain Blog",
      "content": "TL;DR: LangGraph is module built on top of LangChain to better enable creation of cyclical graphs, often needed for agent runtimes. This state is updated by nodes in the graph, which return operations to attributes of this state (in the form of a key-value store). After adding nodes, you can then add edges to create the graph. An example of this may be in the basic agent runtime, where we always want the model to be called after we call a tool. The state of this graph by default contains concepts that should be familiar to you if you've used LangChain agents: `input`, `chat_history`, `intermediate_steps` (and `agent_outcome` to represent the most recent agent outcome)",
      "score": 0.7407191,
      "raw_content": null
    },
    {
      "url": "https://medium.com/@cplog/introduction-to-langgraph-a-beginners-guide-14f9be027141",
      "title": "Introduction to LangGraph: A Beginner's Guide - Medium",
      "content": "*   **Stateful Graph:** LangGraph revolves around the concept of a stateful graph, where each node in the graph represents a step in your computation, and the graph maintains a state that is passed around and updated as the computation progresses. LangGraph supports conditional edges, allowing you to dynamically determine the next node to execute based on the current state of the graph. Image 10: Introduction to AI Agent with LangChain and LangGraph: A Beginner’s Guide Image 18: How to build LLM Agent with LangGraph — StateGraph and Reducer Image 20: Simplest Graphs using LangGraph Framework Image 24: Building a ReAct Agent with Langgraph: A Step-by-Step Guide Image 28: Building an Agentic RAG with LangGraph: A Step-by-Step Guide",
      "score": 0.65279555,
      "raw_content": null
    }
  ],
  "response_time": 1.34
}
```

:::

## 4. Define the graph

:::python
For the `StateGraph` you created in the [first tutorial](./1-build-basic-chatbot.md), add `bind_tools` on the LLM. This lets the LLM know the correct JSON format to use if it wants to use the search engine.
:::

:::js
For the `StateGraph` you created in the [first tutorial](./1-build-basic-chatbot.md), add `bindTools` on the LLM. This lets the LLM know the correct JSON format to use if it wants to use the search engine.
:::

Let's first select our LLM:

:::python
{% include-markdown "../../../snippets/chat_model_tabs.md" %}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

:::

:::js

```typescript
import { ChatAnthropic } from "@langchain/anthropic";

const llm = new ChatAnthropic({ model: "claude-3-5-sonnet-latest" });
```

:::

We can now incorporate it into a `StateGraph`:

:::python

```python
from typing import Annotated

from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

# Modification: tell the LLM which tools it can call
# highlight-next-line
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

graph_builder.add_node("chatbot", chatbot)
```

:::

:::js

```typescript hl_lines="7-8"
import { StateGraph, MessagesZodState } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({ messages: MessagesZodState.shape.messages });

const chatbot = async (state: z.infer<typeof State>) => {
  // Modification: tell the LLM which tools it can call
  const llmWithTools = llm.bindTools(tools);

  return { messages: [await llmWithTools.invoke(state.messages)] };
};
```

:::

## 5. Create a function to run the tools

:::python

Now, create a function to run the tools if they are called. Do this by adding the tools to a new node called `BasicToolNode` that checks the most recent message in the state and calls tools if the message contains `tool_calls`. It relies on the LLM's `tool_calling` support, which is available in Anthropic, OpenAI, Google Gemini, and a number of other LLM providers.

```python
import json

from langchain_core.messages import ToolMessage


class BasicToolNode:
    """A node that runs the tools requested in the last AIMessage."""

    def __init__(self, tools: list) -> None:
        self.tools_by_name = {tool.name: tool for tool in tools}

    def __call__(self, inputs: dict):
        if messages := inputs.get("messages", []):
            message = messages[-1]
        else:
            raise ValueError("No message found in input")
        outputs = []
        for tool_call in message.tool_calls:
            tool_result = self.tools_by_name[tool_call["name"]].invoke(
                tool_call["args"]
            )
            outputs.append(
                ToolMessage(
                    content=json.dumps(tool_result),
                    name=tool_call["name"],
                    tool_call_id=tool_call["id"],
                )
            )
        return {"messages": outputs}


tool_node = BasicToolNode(tools=[tool])
graph_builder.add_node("tools", tool_node)
```

!!! note

    If you do not want to build this yourself in the future, you can use LangGraph's prebuilt [ToolNode](https://langchain-ai.github.io/langgraph/reference/agents/#langgraph.prebuilt.tool_node.ToolNode).

:::

:::js

Now, create a function to run the tools if they are called. Do this by adding the tools to a new node called `"tools"` that checks the most recent message in the state and calls tools if the message contains `tool_calls`. It relies on the LLM's tool calling support, which is available in Anthropic, OpenAI, Google Gemini, and a number of other LLM providers.

```typescript
import type { StructuredToolInterface } from "@langchain/core/tools";
import { isAIMessage, ToolMessage } from "@langchain/core/messages";

function createToolNode(tools: StructuredToolInterface[]) {
  const toolByName: Record<string, StructuredToolInterface> = {};
  for (const tool of tools) {
    toolByName[tool.name] = tool;
  }

  return async (inputs: z.infer<typeof State>) => {
    const { messages } = inputs;
    if (!messages || messages.length === 0) {
      throw new Error("No message found in input");
    }

    const message = messages.at(-1);
    if (!message || !isAIMessage(message) || !message.tool_calls) {
      throw new Error("Last message is not an AI message with tool calls");
    }

    const outputs: ToolMessage[] = [];
    for (const toolCall of message.tool_calls) {
      if (!toolCall.id) throw new Error("Tool call ID is required");

      const tool = toolByName[toolCall.name];
      if (!tool) throw new Error(`Tool ${toolCall.name} not found`);

      const result = await tool.invoke(toolCall.args);

      outputs.push(
        new ToolMessage({
          content: JSON.stringify(result),
          name: toolCall.name,
          tool_call_id: toolCall.id,
        })
      );
    }

    return { messages: outputs };
  };
}
```

!!! note

    If you do not want to build this yourself in the future, you can use LangGraph's prebuilt [ToolNode](https://langchain-ai.github.io/langgraphjs/reference/classes/langgraph_prebuilt.ToolNode.html).

:::

## 6. Define the `conditional_edges`

With the tool node added, now you can define the `conditional_edges`.

**Edges** route the control flow from one node to the next. **Conditional edges** start from a single node and usually contain "if" statements to route to different nodes depending on the current graph state. These functions receive the current graph `state` and return a string or list of strings indicating which node(s) to call next.

:::python
Next, define a router function called `route_tools` that checks for `tool_calls` in the chatbot's output. Provide this function to the graph by calling `add_conditional_edges`, which tells the graph that whenever the `chatbot` node completes to check this function to see where to go next.
:::

:::js
Next, define a router function called `routeTools` that checks for `tool_calls` in the chatbot's output. Provide this function to the graph by calling `addConditionalEdges`, which tells the graph that whenever the `chatbot` node completes to check this function to see where to go next.
:::

The condition will route to `tools` if tool calls are present and `END` if not. Because the condition can return `END`, you do not need to explicitly set a `finish_point` this time.

:::python

```python
def route_tools(
    state: State,
):
    """
    Use in the conditional_edge to route to the ToolNode if the last message
    has tool calls. Otherwise, route to the end.
    """
    if isinstance(state, list):
        ai_message = state[-1]
    elif messages := state.get("messages", []):
        ai_message = messages[-1]
    else:
        raise ValueError(f"No messages found in input state to tool_edge: {state}")
    if hasattr(ai_message, "tool_calls") and len(ai_message.tool_calls) > 0:
        return "tools"
    return END


# The `tools_condition` function returns "tools" if the chatbot asks to use a tool, and "END" if
# it is fine directly responding. This conditional routing defines the main agent loop.
graph_builder.add_conditional_edges(
    "chatbot",
    route_tools,
    # The following dictionary lets you tell the graph to interpret the condition's outputs as a specific node
    # It defaults to the identity function, but if you
    # want to use a node named something else apart from "tools",
    # You can update the value of the dictionary to something else
    # e.g., "tools": "my_tools"
    {"tools": "tools", END: END},
)
# Any time a tool is called, we return to the chatbot to decide the next step
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")
graph = graph_builder.compile()
```

!!! note

    You can replace this with the prebuilt [tools_condition](https://langchain-ai.github.io/langgraph/reference/prebuilt/#tools_condition) to be more concise.

:::

:::js

```typescript
import { END, START } from "@langchain/langgraph";

const routeTools = (state: z.infer<typeof State>) => {
  /**
   * Use as conditional edge to route to the ToolNode if the last message
   * has tool calls.
   */
  const lastMessage = state.messages.at(-1);
  if (
    lastMessage &&
    isAIMessage(lastMessage) &&
    lastMessage.tool_calls?.length
  ) {
    return "tools";
  }

  /** Otherwise, route to the end. */
  return END;
};

const graph = new StateGraph(State)
  .addNode("chatbot", chatbot)

  // The `routeTools` function returns "tools" if the chatbot asks to use a tool, and "END" if
  // it is fine directly responding. This conditional routing defines the main agent loop.
  .addNode("tools", createToolNode(tools))

  // Start the graph with the chatbot
  .addEdge(START, "chatbot")

  // The `routeTools` function returns "tools" if the chatbot asks to use a tool, and "END" if
  // it is fine directly responding.
  .addConditionalEdges("chatbot", routeTools, ["tools", END])

  // Any time a tool is called, we need to return to the chatbot
  .addEdge("tools", "chatbot")
  .compile();
```

!!! note

    You can replace this with the prebuilt [toolsCondition](https://langchain-ai.github.io/langgraphjs/reference/functions/langgraph_prebuilt.toolsCondition.html) to be more concise.

:::

## 7. Visualize the graph (optional)

:::python
You can visualize the graph using the `get_graph` method and one of the "draw" methods, like `draw_ascii` or `draw_png`. The `draw` methods each require additional dependencies.

```python
from IPython.display import Image, display

try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    # This requires some extra dependencies and is optional
    pass
```

:::

:::js
You can visualize the graph using the `getGraph` method and render the graph with the `drawMermaidPng` method.

```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("chatbot-with-tools.png", imageBuffer);
```

:::

![chatbot-with-tools-diagram](chatbot-with-tools.png)

## 8. Ask the bot questions

Now you can ask the chatbot questions outside its training data:

:::python

```python
def stream_graph_updates(user_input: str):
    for event in graph.stream({"messages": [{"role": "user", "content": user_input}]}):
        for value in event.values():
            print("Assistant:", value["messages"][-1].content)

while True:
    try:
        user_input = input("User: ")
        if user_input.lower() in ["quit", "exit", "q"]:
            print("Goodbye!")
            break

        stream_graph_updates(user_input)
    except:
        # fallback if input() is not available
        user_input = "What do you know about LangGraph?"
        print("User: " + user_input)
        stream_graph_updates(user_input)
        break
```

```
Assistant: [{'text': "To provide you with accurate and up-to-date information about LangGraph, I'll need to search for the latest details. Let me do that for you.", 'type': 'text'}, {'id': 'toolu_01Q588CszHaSvvP2MxRq9zRD', 'input': {'query': 'LangGraph AI tool information'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
Assistant: [{"url": "https://www.langchain.com/langgraph", "content": "LangGraph sets the foundation for how we can build and scale AI workloads \u2014 from conversational agents, complex task automation, to custom LLM-backed experiences that 'just work'. The next chapter in building complex production-ready features with LLMs is agentic, and with LangGraph and LangSmith, LangChain delivers an out-of-the-box solution ..."}, {"url": "https://github.com/langchain-ai/langgraph", "content": "Overview. LangGraph is a library for building stateful, multi-actor applications with LLMs, used to create agent and multi-agent workflows. Compared to other LLM frameworks, it offers these core benefits: cycles, controllability, and persistence. LangGraph allows you to define flows that involve cycles, essential for most agentic architectures ..."}]
Assistant: Based on the search results, I can provide you with information about LangGraph:

1. Purpose:
   LangGraph is a library designed for building stateful, multi-actor applications with Large Language Models (LLMs). It's particularly useful for creating agent and multi-agent workflows.

2. Developer:
   LangGraph is developed by LangChain, a company known for its tools and frameworks in the AI and LLM space.

3. Key Features:
   - Cycles: LangGraph allows the definition of flows that involve cycles, which is essential for most agentic architectures.
   - Controllability: It offers enhanced control over the application flow.
   - Persistence: The library provides ways to maintain state and persistence in LLM-based applications.

4. Use Cases:
   LangGraph can be used for various applications, including:
   - Conversational agents
   - Complex task automation
   - Custom LLM-backed experiences

5. Integration:
   LangGraph works in conjunction with LangSmith, another tool by LangChain, to provide an out-of-the-box solution for building complex, production-ready features with LLMs.

6. Significance:
...
   LangGraph is noted to offer unique benefits compared to other LLM frameworks, particularly in its ability to handle cycles, provide controllability, and maintain persistence.

LangGraph appears to be a significant tool in the evolving landscape of LLM-based application development, offering developers new ways to create more complex, stateful, and interactive AI systems.
Goodbye!
```

:::

:::js

```typescript
import readline from "node:readline/promises";

const prompt = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

async function generateText(content: string) {
  const stream = await graph.stream(
    { messages: [{ type: "human", content }] },
    { streamMode: "values" }
  );

  for await (const event of stream) {
    const lastMessage = event.messages.at(-1);

    if (lastMessage?.getType() === "ai" || lastMessage?.getType() === "tool") {
      console.log(`Assistant: ${lastMessage?.text}`);
    }
  }
}

while (true) {
  const human = await prompt.question("User: ");
  if (["quit", "exit", "q"].includes(human.trim())) break;
  await generateText(human || "What do you know about LangGraph?");
}

prompt.close();
```

```
User: What do you know about LangGraph?
Assistant: I'll search for the latest information about LangGraph for you.
Assistant: [{"title":"Introduction to LangGraph: A Beginner's Guide - Medium","url":"https://medium.com/@cplog/introduction-to-langgraph-a-beginners-guide-14f9be027141","content":"..."}]
Assistant: Based on the search results, I can provide you with information about LangGraph:

LangGraph is a library within the LangChain ecosystem designed for building stateful, multi-actor applications with Large Language Models (LLMs). Here are the key aspects:

**Core Purpose:**
- LangGraph is specifically designed for creating agent and multi-agent workflows
- It provides a framework for defining, coordinating, and executing multiple LLM agents in a structured manner

**Key Features:**
1. **Stateful Graph Architecture**: LangGraph revolves around a stateful graph where each node represents a step in computation, and the graph maintains state that is passed around and updated as the computation progresses

2. **Conditional Edges**: It supports conditional edges, allowing you to dynamically determine the next node to execute based on the current state of the graph

3. **Cycles**: Unlike other LLM frameworks, LangGraph allows you to define flows that involve cycles, which is essential for most agentic architectures

4. **Controllability**: It offers enhanced control over the application flow

5. **Persistence**: The library provides ways to maintain state and persistence in LLM-based applications

**Use Cases:**
- Conversational agents
- Complex task automation
- Custom LLM-backed experiences
- Multi-agent systems that perform complex tasks

**Benefits:**
LangGraph allows developers to focus on the high-level logic of their applications rather than the intricacies of agent coordination, making it easier to build complex, production-ready features with LLMs.

This makes LangGraph a significant tool in the evolving landscape of LLM-based application development.
```

:::

## 9. Use prebuilts

For ease of use, adjust your code to replace the following with LangGraph prebuilt components. These have built in functionality like parallel API execution.

:::python

- `BasicToolNode` is replaced with the prebuilt [ToolNode](https://langchain-ai.github.io/langgraph/reference/prebuilt/#toolnode)
- `route_tools` is replaced with the prebuilt [tools_condition](https://langchain-ai.github.io/langgraph/reference/prebuilt/#tools_condition)

{% include-markdown "../../../snippets/chat_model_tabs.md" %}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

```python hl_lines="25 30"
from typing import Annotated

from langchain_tavily import TavilySearch
from langchain_core.messages import BaseMessage
from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

tool = TavilySearch(max_results=2)
tools = [tool]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=[tool])
graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)
# Any time a tool is called, we return to the chatbot to decide the next step
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")
graph = graph_builder.compile()
```

:::

:::js

- `createToolNode` is replaced with the prebuilt [ToolNode](https://langchain-ai.github.io/langgraphjs/reference/classes/langgraph_prebuilt.ToolNode.html)
- `routeTools` is replaced with the prebuilt [toolsCondition](https://langchain-ai.github.io/langgraphjs/reference/functions/langgraph_prebuilt.toolsCondition.html)

```typescript
import { TavilySearch } from "@langchain/tavily";
import { ChatOpenAI } from "@langchain/openai";
import { StateGraph, START, MessagesZodState, END } from "@langchain/langgraph";
import { ToolNode, toolsCondition } from "@langchain/langgraph/prebuilt";
import { z } from "zod";

const State = z.object({ messages: MessagesZodState.shape.messages });

const tools = [new TavilySearch({ maxResults: 2 })];

const llm = new ChatOpenAI({ model: "gpt-4o-mini" }).bindTools(tools);

const graph = new StateGraph(State)
  .addNode("chatbot", async (state) => ({
    messages: [await llm.invoke(state.messages)],
  }))
  .addNode("tools", new ToolNode(tools))
  .addConditionalEdges("chatbot", toolsCondition, ["tools", END])
  .addEdge("tools", "chatbot")
  .addEdge(START, "chatbot")
  .compile();
```

:::

**Congratulations!** You've created a conversational agent in LangGraph that can use a search engine to retrieve updated information when needed. Now it can handle a wider range of user queries.

:::python

To inspect all the steps your agent just took, check out this [LangSmith trace](https://smith.langchain.com/public/4fbd7636-25af-4638-9587-5a02fdbb0172/r).

:::

## Next steps

The chatbot cannot remember past interactions on its own, which limits its ability to have coherent, multi-turn conversations. In the next part, you will [add **memory**](./3-add-memory.md) to address this.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/get-started/3-add-memory.md
```md
# Add memory

The chatbot can now [use tools](./2-add-tools.md) to answer user questions, but it does not remember the context of previous interactions. This limits its ability to have coherent, multi-turn conversations.

LangGraph solves this problem through **persistent checkpointing**. If you provide a `checkpointer` when compiling the graph and a `thread_id` when calling your graph, LangGraph automatically saves the state after each step. When you invoke the graph again using the same `thread_id`, the graph loads its saved state, allowing the chatbot to pick up where it left off.

We will see later that **checkpointing** is _much_ more powerful than simple chat memory - it lets you save and resume complex state at any time for error recovery, human-in-the-loop workflows, time travel interactions, and more. But first, let's add checkpointing to enable multi-turn conversations.

!!! note

    This tutorial builds on [Add tools](./2-add-tools.md).

## 1. Create a `MemorySaver` checkpointer

Create a `MemorySaver` checkpointer:

:::python

```python
from langgraph.checkpoint.memory import InMemorySaver

memory = InMemorySaver()
```

:::

:::js

```typescript
import { MemorySaver } from "@langchain/langgraph";

const memory = new MemorySaver();
```

:::

This is in-memory checkpointer, which is convenient for the tutorial. However, in a production application, you would likely change this to use `SqliteSaver` or `PostgresSaver` and connect a database.

## 2. Compile the graph

Compile the graph with the provided checkpointer, which will checkpoint the `State` as the graph works through each node:

:::python

```python
graph = graph_builder.compile(checkpointer=memory)
```

:::

:::js

```typescript hl_lines="7"
const graph = new StateGraph(State)
  .addNode("chatbot", chatbot)
  .addNode("tools", new ToolNode(tools))
  .addConditionalEdges("chatbot", toolsCondition, ["tools", END])
  .addEdge("tools", "chatbot")
  .addEdge(START, "chatbot")
  .compile({ checkpointer: memory });
```

:::

## 3. Interact with your chatbot

Now you can interact with your bot!

1.  Pick a thread to use as the key for this conversation.

    :::python

    ```python
    config = {"configurable": {"thread_id": "1"}}
    ```

    :::

    :::js

    ```typescript
    const config = { configurable: { thread_id: "1" } };
    ```

    :::

2.  Call your chatbot:

    :::python

    ```python
    user_input = "Hi there! My name is Will."

    # The config is the **second positional argument** to stream() or invoke()!
    events = graph.stream(
        {"messages": [{"role": "user", "content": user_input}]},
        config,
        stream_mode="values",
    )
    for event in events:
        event["messages"][-1].pretty_print()
    ```

    ```
    ================================ Human Message =================================

    Hi there! My name is Will.
    ================================== Ai Message ==================================

    Hello Will! It's nice to meet you. How can I assist you today? Is there anything specific you'd like to know or discuss?
    ```

    !!! note

        The config was provided as the **second positional argument** when calling our graph. It importantly is _not_ nested within the graph inputs (`{'messages': []}`).

    :::

    :::js

    ```typescript
    const userInput = "Hi there! My name is Will.";

    const events = await graph.stream(
      { messages: [{ type: "human", content: userInput }] },
      { configurable: { thread_id: "1" }, streamMode: "values" }
    );

    for await (const event of events) {
      const lastMessage = event.messages.at(-1);
      console.log(`${lastMessage?.getType()}: ${lastMessage?.text}`);
    }
    ```

    ```
    human: Hi there! My name is Will.
    ai: Hello Will! It's nice to meet you. How can I assist you today? Is there anything specific you'd like to know or discuss?
    ```

    !!! note
    !!! note

        The config was provided as the **second parameter** when calling our graph. It importantly is _not_ nested within the graph inputs (`{"messages": []}`).

    :::

## 4. Ask a follow up question

Ask a follow up question:

:::python

```python
user_input = "Remember my name?"

# The config is the **second positional argument** to stream() or invoke()!
events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    config,
    stream_mode="values",
)
for event in events:
    event["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

Remember my name?
================================== Ai Message ==================================

Of course, I remember your name, Will. I always try to pay attention to important details that users share with me. Is there anything else you'd like to talk about or any questions you have? I'm here to help with a wide range of topics or tasks.
```

:::

:::js

```typescript
const userInput2 = "Remember my name?";

const events2 = await graph.stream(
  { messages: [{ type: "human", content: userInput2 }] },
  { configurable: { thread_id: "1" }, streamMode: "values" }
);

for await (const event of events2) {
  const lastMessage = event.messages.at(-1);
  console.log(`${lastMessage?.getType()}: ${lastMessage?.text}`);
}
```

```
human: Remember my name?
ai: Yes, your name is Will. How can I help you today?
```

:::

**Notice** that we aren't using an external list for memory: it's all handled by the checkpointer! You can inspect the full execution in this [LangSmith trace](https://smith.langchain.com/public/29ba22b5-6d40-4fbe-8d27-b369e3329c84/r) to see what's going on.

Don't believe me? Try this using a different config.

:::python

```python
# The only difference is we change the `thread_id` here to "2" instead of "1"
events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    # highlight-next-line
    {"configurable": {"thread_id": "2"}},
    stream_mode="values",
)
for event in events:
    event["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

Remember my name?
================================== Ai Message ==================================

I apologize, but I don't have any previous context or memory of your name. As an AI assistant, I don't retain information from past conversations. Each interaction starts fresh. Could you please tell me your name so I can address you properly in this conversation?
```

:::

:::js

```typescript hl_lines="3-4"
const events3 = await graph.stream(
  { messages: [{ type: "human", content: userInput2 }] },
  // The only difference is we change the `thread_id` here to "2" instead of "1"
  { configurable: { thread_id: "2" }, streamMode: "values" }
);

for await (const event of events3) {
  const lastMessage = event.messages.at(-1);
  console.log(`${lastMessage?.getType()}: ${lastMessage?.text}`);
}
```

```
human: Remember my name?
ai: I don't have the ability to remember personal information about users between interactions. However, I'm here to help you with any questions or topics you want to discuss!
```

:::

**Notice** that the **only** change we've made is to modify the `thread_id` in the config. See this call's [LangSmith trace](https://smith.langchain.com/public/51a62351-2f0a-4058-91cc-9996c5561428/r) for comparison.

## 5. Inspect the state

:::python

By now, we have made a few checkpoints across two different threads. But what goes into a checkpoint? To inspect a graph's `state` for a given config at any time, call `get_state(config)`.

```python
snapshot = graph.get_state(config)
snapshot
```

```
StateSnapshot(values={'messages': [HumanMessage(content='Hi there! My name is Will.', additional_kwargs={}, response_metadata={}, id='8c1ca919-c553-4ebf-95d4-b59a2d61e078'), AIMessage(content="Hello Will! It's nice to meet you. How can I assist you today? Is there anything specific you'd like to know or discuss?", additional_kwargs={}, response_metadata={'id': 'msg_01WTQebPhNwmMrmmWojJ9KXJ', 'model': 'claude-3-5-sonnet-20240620', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 405, 'output_tokens': 32}}, id='run-58587b77-8c82-41e6-8a90-d62c444a261d-0', usage_metadata={'input_tokens': 405, 'output_tokens': 32, 'total_tokens': 437}), HumanMessage(content='Remember my name?', additional_kwargs={}, response_metadata={}, id='daba7df6-ad75-4d6b-8057-745881cea1ca'), AIMessage(content="Of course, I remember your name, Will. I always try to pay attention to important details that users share with me. Is there anything else you'd like to talk about or any questions you have? I'm here to help with a wide range of topics or tasks.", additional_kwargs={}, response_metadata={'id': 'msg_01E41KitY74HpENRgXx94vag', 'model': 'claude-3-5-sonnet-20240620', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 444, 'output_tokens': 58}}, id='run-ffeaae5c-4d2d-4ddb-bd59-5d5cbf2a5af8-0', usage_metadata={'input_tokens': 444, 'output_tokens': 58, 'total_tokens': 502})]}, next=(), config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef7d06e-93e0-6acc-8004-f2ac846575d2'}}, metadata={'source': 'loop', 'writes': {'chatbot': {'messages': [AIMessage(content="Of course, I remember your name, Will. I always try to pay attention to important details that users share with me. Is there anything else you'd like to talk about or any questions you have? I'm here to help with a wide range of topics or tasks.", additional_kwargs={}, response_metadata={'id': 'msg_01E41KitY74HpENRgXx94vag', 'model': 'claude-3-5-sonnet-20240620', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 444, 'output_tokens': 58}}, id='run-ffeaae5c-4d2d-4ddb-bd59-5d5cbf2a5af8-0', usage_metadata={'input_tokens': 444, 'output_tokens': 58, 'total_tokens': 502})]}}, 'step': 4, 'parents': {}}, created_at='2024-09-27T19:30:10.820758+00:00', parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef7d06e-859f-6206-8003-e1bd3c264b8f'}}, tasks=())
```

```
snapshot.next  # (since the graph ended this turn, `next` is empty. If you fetch a state from within a graph invocation, next tells which node will execute next)
```

:::

:::js

By now, we have made a few checkpoints across two different threads. But what goes into a checkpoint? To inspect a graph's `state` for a given config at any time, call `getState(config)`.

```typescript
await graph.getState({ configurable: { thread_id: "1" } });
```

```typescript
{
  values: {
    messages: [
      HumanMessage {
        "id": "32fabcef-b3b8-481f-8bcb-fd83399a5f8d",
        "content": "Hi there! My name is Will.",
        "additional_kwargs": {},
        "response_metadata": {}
      },
      AIMessage {
        "id": "chatcmpl-BrPbTsCJbVqBvXWySlYoTJvM75Kv8",
        "content": "Hello Will! How can I assist you today?",
        "additional_kwargs": {},
        "response_metadata": {},
        "tool_calls": [],
        "invalid_tool_calls": []
      },
      HumanMessage {
        "id": "561c3aad-f8fc-4fac-94a6-54269a220856",
        "content": "Remember my name?",
        "additional_kwargs": {},
        "response_metadata": {}
      },
      AIMessage {
        "id": "chatcmpl-BrPbU4BhhsUikGbW37hYuF5vvnnE2",
        "content": "Yes, I remember your name, Will! How can I help you today?",
        "additional_kwargs": {},
        "response_metadata": {},
        "tool_calls": [],
        "invalid_tool_calls": []
      }
    ]
  },
  next: [],
  tasks: [],
  metadata: {
    source: 'loop',
    step: 4,
    parents: {},
    thread_id: '1'
  },
  config: {
    configurable: {
      thread_id: '1',
      checkpoint_id: '1f05cccc-9bb6-6270-8004-1d2108bcec77',
      checkpoint_ns: ''
    }
  },
  createdAt: '2025-07-09T13:58:27.607Z',
  parentConfig: {
    configurable: {
      thread_id: '1',
      checkpoint_ns: '',
      checkpoint_id: '1f05cccc-78fa-68d0-8003-ffb01a76b599'
    }
  }
}
```

```typescript
import * as assert from "node:assert";

// Since the graph ended this turn, `next` is empty.
// If you fetch a state from within a graph invocation, next tells which node will execute next)
assert.deepEqual(snapshot.next, []);
```

:::

The snapshot above contains the current state values, corresponding config, and the `next` node to process. In our case, the graph has reached an `END` state, so `next` is empty.

**Congratulations!** Your chatbot can now maintain conversation state across sessions thanks to LangGraph's checkpointing system. This opens up exciting possibilities for more natural, contextual interactions. LangGraph's checkpointing even handles **arbitrarily complex graph states**, which is much more expressive and powerful than simple chat memory.

Check out the code snippet below to review the graph from this tutorial:

:::python

{% include-markdown "../../../snippets/chat_model_tabs.md" %}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

```python hl_lines="36 37"
from typing import Annotated

from langchain.chat_models import init_chat_model
from langchain_tavily import TavilySearch
from langchain_core.messages import BaseMessage
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

tool = TavilySearch(max_results=2)
tools = [tool]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=[tool])
graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)
graph_builder.add_edge("tools", "chatbot")
graph_builder.set_entry_point("chatbot")
memory = InMemorySaver()
graph = graph_builder.compile(checkpointer=memory)
```

:::

:::js

```typescript hl_lines="16 26"
import { END, MessagesZodState, START } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { TavilySearch } from "@langchain/tavily";

import { MemorySaver } from "@langchain/langgraph";
import { StateGraph } from "@langchain/langgraph";
import { ToolNode, toolsCondition } from "@langchain/langgraph/prebuilt";
import { z } from "zod";

const State = z.object({
  messages: MessagesZodState.shape.messages,
});

const tools = [new TavilySearch({ maxResults: 2 })];
const llm = new ChatOpenAI({ model: "gpt-4o-mini" }).bindTools(tools);
const memory = new MemorySaver();

async function generateText(content: string) {

const graph = new StateGraph(State)
  .addNode("chatbot", async (state) => ({
    messages: [await llm.invoke(state.messages)],
  }))
  .addNode("tools", new ToolNode(tools))
  .addConditionalEdges("chatbot", toolsCondition, ["tools", END])
  .addEdge("tools", "chatbot")
  .addEdge(START, "chatbot")
  .compile({ checkpointer: memory });
```

:::

## Next steps

In the next tutorial, you will [add human-in-the-loop to the chatbot](./4-human-in-the-loop.md) to handle situations where it may need guidance or verification before proceeding.


```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/get-started/4-human-in-the-loop.md
```md
# Add human-in-the-loop controls

Agents can be unreliable and may need human input to successfully accomplish tasks. Similarly, for some actions, you may want to require human approval before running to ensure that everything is running as intended.

LangGraph's [persistence](../../concepts/persistence.md) layer supports **human-in-the-loop** workflows, allowing execution to pause and resume based on user feedback. The primary interface to this functionality is the [`interrupt`](../../how-tos/human_in_the_loop/add-human-in-the-loop.md) function. Calling `interrupt` inside a node will pause execution. Execution can be resumed, together with new input from a human, by passing in a [Command](../../concepts/low_level.md#command).

:::python
`interrupt` is ergonomically similar to Python's built-in `input()`, [with some caveats](../../how-tos/human_in_the_loop/add-human-in-the-loop.md).
:::

:::js
`interrupt` is ergonomically similar to Node.js's built-in `readline.question()` function, [with some caveats](../../how-tos/human_in_the_loop/add-human-in-the-loop.md).
`interrupt` is ergonomically similar to Node.js's built-in `readline.question()` function, [with some caveats](../../how-tos/human_in_the_loop/add-human-in-the-loop.md).
:::

!!! note

    This tutorial builds on [Add memory](./3-add-memory.md).

## 1. Add the `human_assistance` tool

Starting with the existing code from the [Add memory to the chatbot](./3-add-memory.md) tutorial, add the `human_assistance` tool to the chatbot. This tool uses `interrupt` to receive information from a human.

Let's first select a chat model:

:::python
{% include-markdown "../../../snippets/chat_model_tabs.md" %}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

:::

:::js

```typescript
// Add your API key here
process.env.ANTHROPIC_API_KEY = "YOUR_API_KEY";
```

:::

We can now incorporate it into our `StateGraph` with an additional tool:

:::python

```python hl_lines="12 19 20 21 22 23"
from typing import Annotated

from langchain_tavily import TavilySearch
from langchain_core.tools import tool
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

from langgraph.types import Command, interrupt

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

@tool
def human_assistance(query: str) -> str:
    """Request assistance from a human."""
    human_response = interrupt({"query": query})
    return human_response["data"]

tool = TavilySearch(max_results=2)
tools = [tool, human_assistance]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    message = llm_with_tools.invoke(state["messages"])
    # Because we will be interrupting during tool execution,
    # we disable parallel tool calling to avoid repeating any
    # tool invocations when we resume.
    assert len(message.tool_calls) <= 1
    return {"messages": [message]}

graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=tools)
graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")
```

:::

:::js

```typescript hl_lines="1 7-19"
import { interrupt, MessagesZodState } from "@langchain/langgraph";
import { ChatAnthropic } from "@langchain/anthropic";
import { TavilySearch } from "@langchain/tavily";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const humanAssistance = tool(
  async ({ query }) => {
    const humanResponse = interrupt({ query });
    return humanResponse.data;
  },
  {
    name: "humanAssistance",
    description: "Request assistance from a human.",
    schema: z.object({
      query: z.string().describe("Human readable question for the human"),
    }),
  }
);

const searchTool = new TavilySearch({ maxResults: 2 });
const searchTool = new TavilySearch({ maxResults: 2 });
const tools = [searchTool, humanAssistance];

const llmWithTools = new ChatAnthropic({
  model: "claude-3-5-sonnet-latest",
}).bindTools(tools);
const llmWithTools = new ChatAnthropic({
  model: "claude-3-5-sonnet-latest",
}).bindTools(tools);

async function chatbot(state: z.infer<typeof MessagesZodState>) {
async function chatbot(state: z.infer<typeof MessagesZodState>) {
  const message = await llmWithTools.invoke(state.messages);


  // Because we will be interrupting during tool execution,
  // we disable parallel tool calling to avoid repeating any
  // tool invocations when we resume.
  if (message.tool_calls && message.tool_calls.length > 1) {
    throw new Error("Multiple tool calls not supported with interrupts");
  }

  return { messages: message };
}
```

:::

!!! tip

    For more information and examples of human-in-the-loop workflows, see [Human-in-the-loop](../../concepts/human_in_the_loop.md).

## 2. Compile the graph

We compile the graph with a checkpointer, as before:

:::python

```python
memory = InMemorySaver()

graph = graph_builder.compile(checkpointer=memory)
```

:::

:::js

```typescript hl_lines="3 11"
import { StateGraph, MemorySaver, START, END } from "@langchain/langgraph";

const memory = new MemorySaver();

const graph = new StateGraph(MessagesZodState)
  .addNode("chatbot", chatbot)
  .addNode("tools", new ToolNode(tools))
  .addConditionalEdges("chatbot", toolsCondition, ["tools", END])
  .addEdge("tools", "chatbot")
  .addEdge(START, "chatbot")
  .compile({ checkpointer: memory });
const graph = new StateGraph(MessagesZodState)
  .addNode("chatbot", chatbot)
  .addNode("tools", new ToolNode(tools))
  .addConditionalEdges("chatbot", toolsCondition, ["tools", END])
  .addEdge("tools", "chatbot")
  .addEdge(START, "chatbot")
  .compile({ checkpointer: memory });
```

:::

## 3. Visualize the graph (optional)

Visualizing the graph, you get the same layout as before – just with the added tool!

:::python

```python
from IPython.display import Image, display

try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    # This requires some extra dependencies and is optional
    pass
```

:::

:::js

```typescript
import * as fs from "node:fs/promises";
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("chatbot-with-tools.png", imageBuffer);
await fs.writeFile("chatbot-with-tools.png", imageBuffer);
```

:::

![chatbot-with-tools-diagram](chatbot-with-tools.png)

## 4. Prompt the chatbot

Now, prompt the chatbot with a question that will engage the new `human_assistance` tool:

:::python

```python
user_input = "I need some expert guidance for building an AI agent. Could you request assistance for me?"
config = {"configurable": {"thread_id": "1"}}

events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    config,
    stream_mode="values",
)
for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

I need some expert guidance for building an AI agent. Could you request assistance for me?
================================== Ai Message ==================================

[{'text': "Certainly! I'd be happy to request expert assistance for you regarding building an AI agent. To do this, I'll use the human_assistance function to relay your request. Let me do that for you now.", 'type': 'text'}, {'id': 'toolu_01ABUqneqnuHNuo1vhfDFQCW', 'input': {'query': 'A user is requesting expert guidance for building an AI agent. Could you please provide some expert advice or resources on this topic?'}, 'name': 'human_assistance', 'type': 'tool_use'}]
Tool Calls:
  human_assistance (toolu_01ABUqneqnuHNuo1vhfDFQCW)
 Call ID: toolu_01ABUqneqnuHNuo1vhfDFQCW
  Args:
    query: A user is requesting expert guidance for building an AI agent. Could you please provide some expert advice or resources on this topic?
```

:::

:::js

```typescript
import { isAIMessage } from "@langchain/core/messages";

const userInput =
  "I need some expert guidance for building an AI agent. Could you request assistance for me?";

const events = await graph.stream(
  { messages: [{ role: "user", content: userInput }] },
  { configurable: { thread_id: "1" }, streamMode: "values" }
  { configurable: { thread_id: "1" }, streamMode: "values" }
);

for await (const event of events) {
  if ("messages" in event) {
    const lastMessage = event.messages.at(-1);
    console.log(`[${lastMessage?.getType()}]: ${lastMessage?.text}`);

    if (
      lastMessage &&
      isAIMessage(lastMessage) &&
      lastMessage.tool_calls?.length
    ) {
    const lastMessage = event.messages.at(-1);
    console.log(`[${lastMessage?.getType()}]: ${lastMessage?.text}`);

    if (
      lastMessage &&
      isAIMessage(lastMessage) &&
      lastMessage.tool_calls?.length
    ) {
      console.log("Tool calls:", lastMessage.tool_calls);
    }
  }
}
```

```
[human]: I need some expert guidance for building an AI agent. Could you request assistance for me?
[ai]: I'll help you request human assistance for guidance on building an AI agent.
[ai]: I'll help you request human assistance for guidance on building an AI agent.
Tool calls: [
  {
    name: 'humanAssistance',
    args: {
      query: 'I would like expert guidance on building an AI agent. Could you please provide assistance with this topic?'
      query: 'I would like expert guidance on building an AI agent. Could you please provide assistance with this topic?'
    },
    id: 'toolu_01Bpxc8rFVMhSaRosS6b85Ts',
    type: 'tool_call'
    id: 'toolu_01Bpxc8rFVMhSaRosS6b85Ts',
    type: 'tool_call'
  }
]
```

:::

The chatbot generated a tool call, but then execution has been interrupted. If you inspect the graph state, you see that it stopped at the tools node:

:::python

```python
snapshot = graph.get_state(config)
snapshot.next
```

```
('tools',)
```

:::

:::js

```typescript
const snapshot = await graph.getState({ configurable: { thread_id: "1" } });
snapshot.next;
const snapshot = await graph.getState({ configurable: { thread_id: "1" } });
snapshot.next;
```

```json
["tools"]
```

:::

!!! info Additional information

    :::python

    Take a closer look at the `human_assistance` tool:

    ```python
    @tool
    def human_assistance(query: str) -> str:
        """Request assistance from a human."""
        human_response = interrupt({"query": query})
        return human_response["data"]
    ```

    Similar to Python's built-in `input()` function, calling `interrupt` inside the tool will pause execution. Progress is persisted based on the [checkpointer](../../concepts/persistence.md#checkpointer-libraries); so if it is persisting with Postgres, it can resume at any time as long as the database is alive. In this example, it is persisting with the in-memory checkpointer and can resume any time if the Python kernel is running.
    :::

    :::js

    Take a closer look at the `humanAssistance` tool:

    ```typescript hl_lines="3"
    const humanAssistance = tool(
      async ({ query }) => {
        const humanResponse = interrupt({ query });
        return humanResponse.data;
      },
      {
        name: "humanAssistance",
        description: "Request assistance from a human.",
        schema: z.object({
          query: z.string().describe("Human readable question for the human"),
        }),
      },
    );

    Take a closer look at the `humanAssistance` tool:

    ```typescript hl_lines="3"
    const humanAssistance = tool(
      async ({ query }) => {
        const humanResponse = interrupt({ query });
        return humanResponse.data;
      },
      {
        name: "humanAssistance",
        description: "Request assistance from a human.",
        schema: z.object({
          query: z.string().describe("Human readable question for the human"),
        }),
      },
    );
    ```

    Calling `interrupt` inside the tool will pause execution. Progress is persisted based on the [checkpointer](../../concepts/persistence.md#checkpointer-libraries); so if it is persisting with Postgres, it can resume at any time as long as the database is alive. In this example, it is persisting with the in-memory checkpointer and can resume any time if the JavaScript runtime is running.
    :::

## 5. Resume execution

To resume execution, pass a [`Command`](../../concepts/low_level.md#command) object containing data expected by the tool. The format of this data can be customized based on needs.

:::python

For this example, use a dict with a key `"data"`:

```python
human_response = (
    "We, the experts are here to help! We'd recommend you check out LangGraph to build your agent."
    " It's much more reliable and extensible than simple autonomous agents."
)

human_command = Command(resume={"data": human_response})

events = graph.stream(human_command, config, stream_mode="values")
for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```
================================== Ai Message ==================================

[{'text': "Certainly! I'd be happy to request expert assistance for you regarding building an AI agent. To do this, I'll use the human_assistance function to relay your request. Let me do that for you now.", 'type': 'text'}, {'id': 'toolu_01ABUqneqnuHNuo1vhfDFQCW', 'input': {'query': 'A user is requesting expert guidance for building an AI agent. Could you please provide some expert advice or resources on this topic?'}, 'name': 'human_assistance', 'type': 'tool_use'}]
Tool Calls:
  human_assistance (toolu_01ABUqneqnuHNuo1vhfDFQCW)
 Call ID: toolu_01ABUqneqnuHNuo1vhfDFQCW
  Args:
    query: A user is requesting expert guidance for building an AI agent. Could you please provide some expert advice or resources on this topic?
================================= Tool Message =================================
Name: human_assistance

We, the experts are here to help! We'd recommend you check out LangGraph to build your agent. It's much more reliable and extensible than simple autonomous agents.
================================== Ai Message ==================================

Thank you for your patience. I've received some expert advice regarding your request for guidance on building an AI agent. Here's what the experts have suggested:

The experts recommend that you look into LangGraph for building your AI agent. They mention that LangGraph is a more reliable and extensible option compared to simple autonomous agents.

LangGraph is likely a framework or library designed specifically for creating AI agents with advanced capabilities. Here are a few points to consider based on this recommendation:

1. Reliability: The experts emphasize that LangGraph is more reliable than simpler autonomous agent approaches. This could mean it has better stability, error handling, or consistent performance.

2. Extensibility: LangGraph is described as more extensible, which suggests that it probably offers a flexible architecture that allows you to easily add new features or modify existing ones as your agent's requirements evolve.

3. Advanced capabilities: Given that it's recommended over "simple autonomous agents," LangGraph likely provides more sophisticated tools and techniques for building complex AI agents.
...
2. Look for tutorials or guides specifically focused on building AI agents with LangGraph.
3. Check if there are any community forums or discussion groups where you can ask questions and get support from other developers using LangGraph.

If you'd like more specific information about LangGraph or have any questions about this recommendation, please feel free to ask, and I can request further assistance from the experts.
Output is truncated. View as a scrollable element or open in a text editor. Adjust cell output settings...
```

:::

:::js
For this example, use an object with a key `"data"`:

```typescript
import { Command } from "@langchain/langgraph";

const humanResponse =
  "We, the experts are here to help! We'd recommend you check out LangGraph to build your agent." +
  " It's much more reliable and extensible than simple autonomous agents.";
(" It's much more reliable and extensible than simple autonomous agents.");

const humanCommand = new Command({ resume: { data: humanResponse } });

const resumeEvents = await graph.stream(humanCommand, {
  configurable: { thread_id: "1" },
  streamMode: "values",
});
const resumeEvents = await graph.stream(humanCommand, {
  configurable: { thread_id: "1" },
  streamMode: "values",
});

for await (const event of resumeEvents) {
  if ("messages" in event) {
    const lastMessage = event.messages.at(-1);
    console.log(`[${lastMessage?.getType()}]: ${lastMessage?.text}`);
    const lastMessage = event.messages.at(-1);
    console.log(`[${lastMessage?.getType()}]: ${lastMessage?.text}`);
  }
}
```

```
[tool]: We, the experts are here to help! We'd recommend you check out LangGraph to build your agent. It's much more reliable and extensible than simple autonomous agents.
[ai]: Thank you for your patience. I've received some expert advice regarding your request for guidance on building an AI agent. Here's what the experts have suggested:

The experts recommend that you look into LangGraph for building your AI agent. They mention that LangGraph is a more reliable and extensible option compared to simple autonomous agents.

LangGraph is likely a framework or library designed specifically for creating AI agents with advanced capabilities. Here are a few points to consider based on this recommendation:

1. Reliability: The experts emphasize that LangGraph is more reliable than simpler autonomous agent approaches. This could mean it has better stability, error handling, or consistent performance.

2. Extensibility: LangGraph is described as more extensible, which suggests that it probably offers a flexible architecture that allows you to easily add new features or modify existing ones as your agent's requirements evolve.

3. Advanced capabilities: Given that it's recommended over "simple autonomous agents," LangGraph likely provides more sophisticated tools and techniques for building complex AI agents.

...
```

:::

The input has been received and processed as a tool message. Review this call's [LangSmith trace](https://smith.langchain.com/public/9f0f87e3-56a7-4dde-9c76-b71675624e91/r) to see the exact work that was done in the above call. Notice that the state is loaded in the first step so that our chatbot can continue where it left off.

**Congratulations!** You've used an `interrupt` to add human-in-the-loop execution to your chatbot, allowing for human oversight and intervention when needed. This opens up the potential UIs you can create with your AI systems. Since you have already added a **checkpointer**, as long as the underlying persistence layer is running, the graph can be paused **indefinitely** and resumed at any time as if nothing had happened.

Check out the code snippet below to review the graph from this tutorial:

:::python

{% include-markdown "../../../snippets/chat_model_tabs.md" %}

```python
from typing import Annotated

from langchain_tavily import TavilySearch
from langchain_core.tools import tool
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.types import Command, interrupt

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

@tool
def human_assistance(query: str) -> str:
    """Request assistance from a human."""
    human_response = interrupt({"query": query})
    return human_response["data"]

tool = TavilySearch(max_results=2)
tools = [tool, human_assistance]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    message = llm_with_tools.invoke(state["messages"])
    assert(len(message.tool_calls) <= 1)
    return {"messages": [message]}

graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=tools)
graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")

memory = InMemorySaver()
graph = graph_builder.compile(checkpointer=memory)
```

:::

:::js

```typescript
import {
  interrupt,
  MessagesZodState,
  StateGraph,
  MemorySaver,
  START,
  END,
} from "@langchain/langgraph";
import { ToolNode, toolsCondition } from "@langchain/langgraph/prebuilt";
import { isAIMessage } from "@langchain/core/messages";
import { ChatAnthropic } from "@langchain/anthropic";
import { TavilySearch } from "@langchain/tavily";
import {
  interrupt,
  MessagesZodState,
  StateGraph,
  MemorySaver,
  START,
  END,
} from "@langchain/langgraph";
import { ToolNode, toolsCondition } from "@langchain/langgraph/prebuilt";
import { isAIMessage } from "@langchain/core/messages";
import { ChatAnthropic } from "@langchain/anthropic";
import { TavilySearch } from "@langchain/tavily";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const humanAssistance = tool(
  async ({ query }) => {
    const humanResponse = interrupt({ query });
    return humanResponse.data;
  },
  {
    name: "humanAssistance",
    description: "Request assistance from a human.",
    schema: z.object({
      query: z.string().describe("Human readable question for the human"),
    }),
  }
);
const humanAssistance = tool(
  async ({ query }) => {
    const humanResponse = interrupt({ query });
    return humanResponse.data;
  },
  {
    name: "humanAssistance",
    description: "Request assistance from a human.",
    schema: z.object({
      query: z.string().describe("Human readable question for the human"),
    }),
  }
);

const searchTool = new TavilySearch({ maxResults: 2 });
const searchTool = new TavilySearch({ maxResults: 2 });
const tools = [searchTool, humanAssistance];

const llmWithTools = new ChatAnthropic({
  model: "claude-3-5-sonnet-latest",
}).bindTools(tools);
const llmWithTools = new ChatAnthropic({
  model: "claude-3-5-sonnet-latest",
}).bindTools(tools);

const chatbot = async (state: z.infer<typeof MessagesZodState>) => {
const chatbot = async (state: z.infer<typeof MessagesZodState>) => {
  const message = await llmWithTools.invoke(state.messages);

  // Because we will be interrupting during tool execution,
  // we disable parallel tool calling to avoid repeating any
  // tool invocations when we resume.

  // Because we will be interrupting during tool execution,
  // we disable parallel tool calling to avoid repeating any
  // tool invocations when we resume.
  if (message.tool_calls && message.tool_calls.length > 1) {
    throw new Error("Multiple tool calls not supported with interrupts");
  }

  return { messages: message };

  return { messages: message };
};

const memory = new MemorySaver();

const graph = new StateGraph(MessagesZodState)
  .addNode("chatbot", chatbot)
  .addNode("tools", new ToolNode(tools))
  .addConditionalEdges("chatbot", toolsCondition, ["tools", END])
  .addEdge("tools", "chatbot")
  .addEdge(START, "chatbot")
  .compile({ checkpointer: memory });

const graph = new StateGraph(MessagesZodState)
  .addNode("chatbot", chatbot)
  .addNode("tools", new ToolNode(tools))
  .addConditionalEdges("chatbot", toolsCondition, ["tools", END])
  .addEdge("tools", "chatbot")
  .addEdge(START, "chatbot")
  .compile({ checkpointer: memory });
```

:::

## Next steps

So far, the tutorial examples have relied on a simple state with one entry: a list of messages. You can go far with this simple state, but if you want to define complex behavior without relying on the message list, you can [add additional fields to the state](./5-customize-state.md).

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/get-started/5-customize-state.md
```md
# Customize state

In this tutorial, you will add additional fields to the state to define complex behavior without relying on the message list. The chatbot will use its search tool to find specific information and forward them to a human for review.

!!! note

    This tutorial builds on [Add human-in-the-loop controls](./4-human-in-the-loop.md).

## 1. Add keys to the state

Update the chatbot to research the birthday of an entity by adding `name` and `birthday` keys to the state:

:::python

```python
from typing import Annotated

from typing_extensions import TypedDict

from langgraph.graph.message import add_messages


class State(TypedDict):
    messages: Annotated[list, add_messages]
    # highlight-next-line
    name: str
    # highlight-next-line
    birthday: str
```

:::

:::js

```typescript
import { MessagesZodState } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  messages: MessagesZodState.shape.messages,
  // highlight-next-line
  name: z.string(),
  // highlight-next-line
  birthday: z.string(),
});
```

:::

Adding this information to the state makes it easily accessible by other graph nodes (like a downstream node that stores or processes the information), as well as the graph's persistence layer.

## 2. Update the state inside the tool

:::python

Now, populate the state keys inside of the `human_assistance` tool. This allows a human to review the information before it is stored in the state. Use [`Command`](../../concepts/low_level.md#using-inside-tools) to issue a state update from inside the tool.

```python
from langchain_core.messages import ToolMessage
from langchain_core.tools import InjectedToolCallId, tool

from langgraph.types import Command, interrupt

@tool
# Note that because we are generating a ToolMessage for a state update, we
# generally require the ID of the corresponding tool call. We can use
# LangChain's InjectedToolCallId to signal that this argument should not
# be revealed to the model in the tool's schema.
def human_assistance(
    name: str, birthday: str, tool_call_id: Annotated[str, InjectedToolCallId]
) -> str:
    """Request assistance from a human."""
    human_response = interrupt(
        {
            "question": "Is this correct?",
            "name": name,
            "birthday": birthday,
        },
    )
    # If the information is correct, update the state as-is.
    if human_response.get("correct", "").lower().startswith("y"):
        verified_name = name
        verified_birthday = birthday
        response = "Correct"
    # Otherwise, receive information from the human reviewer.
    else:
        verified_name = human_response.get("name", name)
        verified_birthday = human_response.get("birthday", birthday)
        response = f"Made a correction: {human_response}"

    # This time we explicitly update the state with a ToolMessage inside
    # the tool.
    state_update = {
        "name": verified_name,
        "birthday": verified_birthday,
        "messages": [ToolMessage(response, tool_call_id=tool_call_id)],
    }
    # We return a Command object in the tool to update our state.
    return Command(update=state_update)
```

:::

:::js

Now, populate the state keys inside of the `humanAssistance` tool. This allows a human to review the information before it is stored in the state. Use [`Command`](../../concepts/low_level.md#using-inside-tools) to issue a state update from inside the tool.

```typescript
import { tool } from "@langchain/core/tools";
import { ToolMessage } from "@langchain/core/messages";
import { Command, interrupt } from "@langchain/langgraph";

const humanAssistance = tool(
  async (input, config) => {
    // Note that because we are generating a ToolMessage for a state update,
    // we generally require the ID of the corresponding tool call.
    // This is available in the tool's config.
    const toolCallId = config?.toolCall?.id as string | undefined;
    if (!toolCallId) throw new Error("Tool call ID is required");

    const humanResponse = await interrupt({
      question: "Is this correct?",
      name: input.name,
      birthday: input.birthday,
    });

    // We explicitly update the state with a ToolMessage inside the tool.
    const stateUpdate = (() => {
      // If the information is correct, update the state as-is.
      if (humanResponse.correct?.toLowerCase().startsWith("y")) {
        return {
          name: input.name,
          birthday: input.birthday,
          messages: [
            new ToolMessage({ content: "Correct", tool_call_id: toolCallId }),
          ],
        };
      }

      // Otherwise, receive information from the human reviewer.
      return {
        name: humanResponse.name || input.name,
        birthday: humanResponse.birthday || input.birthday,
        messages: [
          new ToolMessage({
            content: `Made a correction: ${JSON.stringify(humanResponse)}`,
            tool_call_id: toolCallId,
          }),
        ],
      };
    })();

    // We return a Command object in the tool to update our state.
    return new Command({ update: stateUpdate });
  },
  {
    name: "humanAssistance",
    description: "Request assistance from a human.",
    schema: z.object({
      name: z.string().describe("The name of the entity"),
      birthday: z.string().describe("The birthday/release date of the entity"),
    }),
  }
);
```

:::

The rest of the graph stays the same.

## 3. Prompt the chatbot

:::python
Prompt the chatbot to look up the "birthday" of the LangGraph library and direct the chatbot to reach out to the `human_assistance` tool once it has the required information. By setting `name` and `birthday` in the arguments for the tool, you force the chatbot to generate proposals for these fields.

```python
user_input = (
    "Can you look up when LangGraph was released? "
    "When you have the answer, use the human_assistance tool for review."
)
config = {"configurable": {"thread_id": "1"}}

events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    config,
    stream_mode="values",
)
for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

:::

:::js
Prompt the chatbot to look up the "birthday" of the LangGraph library and direct the chatbot to reach out to the `humanAssistance` tool once it has the required information. By setting `name` and `birthday` in the arguments for the tool, you force the chatbot to generate proposals for these fields.

```typescript
import { isAIMessage } from "@langchain/core/messages";

const userInput =
  "Can you look up when LangGraph was released? " +
  "When you have the answer, use the humanAssistance tool for review.";

const events = await graph.stream(
  { messages: [{ role: "user", content: userInput }] },
  { configurable: { thread_id: "1" }, streamMode: "values" }
);

for await (const event of events) {
  if ("messages" in event) {
    const lastMessage = event.messages.at(-1);

    console.log(
      "=".repeat(32),
      `${lastMessage?.getType()} Message`,
      "=".repeat(32)
    );
    console.log(lastMessage?.text);

    if (
      lastMessage &&
      isAIMessage(lastMessage) &&
      lastMessage.tool_calls?.length
    ) {
      console.log("Tool Calls:");
      for (const call of lastMessage.tool_calls) {
        console.log(`  ${call.name} (${call.id})`);
        console.log(`  Args: ${JSON.stringify(call.args)}`);
      }
    }
  }
}
```

:::

```
================================ Human Message =================================

Can you look up when LangGraph was released? When you have the answer, use the human_assistance tool for review.
================================== Ai Message ==================================

[{'text': "Certainly! I'll start by searching for information about LangGraph's release date using the Tavily search function. Then, I'll use the human_assistance tool for review.", 'type': 'text'}, {'id': 'toolu_01JoXQPgTVJXiuma8xMVwqAi', 'input': {'query': 'LangGraph release date'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
Tool Calls:
  tavily_search_results_json (toolu_01JoXQPgTVJXiuma8xMVwqAi)
 Call ID: toolu_01JoXQPgTVJXiuma8xMVwqAi
  Args:
    query: LangGraph release date
================================= Tool Message =================================
Name: tavily_search_results_json

[{"url": "https://blog.langchain.dev/langgraph-cloud/", "content": "We also have a new stable release of LangGraph. By LangChain 6 min read Jun 27, 2024 (Oct '24) Edit: Since the launch of LangGraph Platform, we now have multiple deployment options alongside LangGraph Studio - which now fall under LangGraph Platform. LangGraph Platform is synonymous with our Cloud SaaS deployment option."}, {"url": "https://changelog.langchain.com/announcements/langgraph-cloud-deploy-at-scale-monitor-carefully-iterate-boldly", "content": "LangChain - Changelog | ☁ 🚀 LangGraph Platform: Deploy at scale, monitor LangChain LangSmith LangGraph LangChain LangSmith LangGraph LangChain LangSmith LangGraph LangChain Changelog Sign up for our newsletter to stay up to date DATE: The LangChain Team LangGraph LangGraph Platform ☁ 🚀 LangGraph Platform: Deploy at scale, monitor carefully, iterate boldly DATE: June 27, 2024 AUTHOR: The LangChain Team LangGraph Platform is now in closed beta, offering scalable, fault-tolerant deployment for LangGraph agents. LangGraph Platform also includes a new playground-like studio for debugging agent failure modes and quick iteration: Join the waitlist today for LangGraph Platform. And to learn more, read our blog post announcement or check out our docs. Subscribe By clicking subscribe, you accept our privacy policy and terms and conditions."}]
================================== Ai Message ==================================

[{'text': "Based on the search results, it appears that LangGraph was already in existence before June 27, 2024, when LangGraph Platform was announced. However, the search results don't provide a specific release date for the original LangGraph. \n\nGiven this information, I'll use the human_assistance tool to review and potentially provide more accurate information about LangGraph's initial release date.", 'type': 'text'}, {'id': 'toolu_01JDQAV7nPqMkHHhNs3j3XoN', 'input': {'name': 'Assistant', 'birthday': '2023-01-01'}, 'name': 'human_assistance', 'type': 'tool_use'}]
Tool Calls:
  human_assistance (toolu_01JDQAV7nPqMkHHhNs3j3XoN)
 Call ID: toolu_01JDQAV7nPqMkHHhNs3j3XoN
  Args:
    name: Assistant
    birthday: 2023-01-01
```

:::python
We've hit the `interrupt` in the `human_assistance` tool again.
:::

:::js
We've hit the `interrupt` in the `humanAssistance` tool again.
:::

## 4. Add human assistance

The chatbot failed to identify the correct date, so supply it with information:

:::python

```python
human_command = Command(
    resume={
        "name": "LangGraph",
        "birthday": "Jan 17, 2024",
    },
)

events = graph.stream(human_command, config, stream_mode="values")
for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

:::

:::js

```typescript
import { Command } from "@langchain/langgraph";

const humanCommand = new Command({
  resume: {
    name: "LangGraph",
    birthday: "Jan 17, 2024",
  },
});

const resumeEvents = await graph.stream(humanCommand, {
  configurable: { thread_id: "1" },
  streamMode: "values",
});

for await (const event of resumeEvents) {
  if ("messages" in event) {
    const lastMessage = event.messages.at(-1);

    console.log(
      "=".repeat(32),
      `${lastMessage?.getType()} Message`,
      "=".repeat(32)
    );
    console.log(lastMessage?.text);

    if (
      lastMessage &&
      isAIMessage(lastMessage) &&
      lastMessage.tool_calls?.length
    ) {
      console.log("Tool Calls:");
      for (const call of lastMessage.tool_calls) {
        console.log(`  ${call.name} (${call.id})`);
        console.log(`  Args: ${JSON.stringify(call.args)}`);
      }
    }
  }
}
```

:::

```
================================== Ai Message ==================================

[{'text': "Based on the search results, it appears that LangGraph was already in existence before June 27, 2024, when LangGraph Platform was announced. However, the search results don't provide a specific release date for the original LangGraph. \n\nGiven this information, I'll use the human_assistance tool to review and potentially provide more accurate information about LangGraph's initial release date.", 'type': 'text'}, {'id': 'toolu_01JDQAV7nPqMkHHhNs3j3XoN', 'input': {'name': 'Assistant', 'birthday': '2023-01-01'}, 'name': 'human_assistance', 'type': 'tool_use'}]
Tool Calls:
  human_assistance (toolu_01JDQAV7nPqMkHHhNs3j3XoN)
 Call ID: toolu_01JDQAV7nPqMkHHhNs3j3XoN
  Args:
    name: Assistant
    birthday: 2023-01-01
================================= Tool Message =================================
Name: human_assistance

Made a correction: {'name': 'LangGraph', 'birthday': 'Jan 17, 2024'}
================================== Ai Message ==================================

Thank you for the human assistance. I can now provide you with the correct information about LangGraph's release date.

LangGraph was initially released on January 17, 2024. This information comes from the human assistance correction, which is more accurate than the search results I initially found.

To summarize:
1. LangGraph's original release date: January 17, 2024
2. LangGraph Platform announcement: June 27, 2024

It's worth noting that LangGraph had been in development and use for some time before the LangGraph Platform announcement, but the official initial release of LangGraph itself was on January 17, 2024.
```

Note that these fields are now reflected in the state:

:::python

```python
snapshot = graph.get_state(config)

{k: v for k, v in snapshot.values.items() if k in ("name", "birthday")}
```

```
{'name': 'LangGraph', 'birthday': 'Jan 17, 2024'}
```

:::

:::js

```typescript
const snapshot = await graph.getState(config);

const relevantState = Object.fromEntries(
  Object.entries(snapshot.values).filter(([k]) =>
    ["name", "birthday"].includes(k)
  )
);
```

```
{ name: 'LangGraph', birthday: 'Jan 17, 2024' }
```

:::

This makes them easily accessible to downstream nodes (e.g., a node that further processes or stores the information).

## 5. Manually update the state

:::python
LangGraph gives a high degree of control over the application state. For instance, at any point (including when interrupted), you can manually override a key using `graph.update_state`:

```python
graph.update_state(config, {"name": "LangGraph (library)"})
```

```
{'configurable': {'thread_id': '1',
  'checkpoint_ns': '',
  'checkpoint_id': '1efd4ec5-cf69-6352-8006-9278f1730162'}}
```

:::

:::js
LangGraph gives a high degree of control over the application state. For instance, at any point (including when interrupted), you can manually override a key using `graph.updateState`:

```typescript
await graph.updateState(
  { configurable: { thread_id: "1" } },
  { name: "LangGraph (library)" }
);
```

```typescript
{
  configurable: {
    thread_id: '1',
    checkpoint_ns: '',
    checkpoint_id: '1efd4ec5-cf69-6352-8006-9278f1730162'
  }
}
```

:::

## 6. View the new value

:::python
If you call `graph.get_state`, you can see the new value is reflected:

```python
snapshot = graph.get_state(config)

{k: v for k, v in snapshot.values.items() if k in ("name", "birthday")}
```

```
{'name': 'LangGraph (library)', 'birthday': 'Jan 17, 2024'}
```

:::

:::js
If you call `graph.getState`, you can see the new value is reflected:

```typescript
const updatedSnapshot = await graph.getState(config);

const updatedRelevantState = Object.fromEntries(
  Object.entries(updatedSnapshot.values).filter(([k]) =>
    ["name", "birthday"].includes(k)
  )
);
```

```typescript
{ name: 'LangGraph (library)', birthday: 'Jan 17, 2024' }
```

:::

Manual state updates will [generate a trace](https://smith.langchain.com/public/7ebb7827-378d-49fe-9f6c-5df0e90086c8/r) in LangSmith. If desired, they can also be used to [control human-in-the-loop workflows](../../how-tos/human_in_the_loop/add-human-in-the-loop.md). Use of the `interrupt` function is generally recommended instead, as it allows data to be transmitted in a human-in-the-loop interaction independently of state updates.

**Congratulations!** You've added custom keys to the state to facilitate a more complex workflow, and learned how to generate state updates from inside tools.

Check out the code snippet below to review the graph from this tutorial:

:::python

{% include-markdown "../../../snippets/chat_model_tabs.md" %}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

```python
from typing import Annotated

from langchain_tavily import TavilySearch
from langchain_core.messages import ToolMessage
from langchain_core.tools import InjectedToolCallId, tool
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.types import Command, interrupt

class State(TypedDict):
    messages: Annotated[list, add_messages]
    name: str
    birthday: str

@tool
def human_assistance(
    name: str, birthday: str, tool_call_id: Annotated[str, InjectedToolCallId]
) -> str:
    """Request assistance from a human."""
    human_response = interrupt(
        {
            "question": "Is this correct?",
            "name": name,
            "birthday": birthday,
        },
    )
    if human_response.get("correct", "").lower().startswith("y"):
        verified_name = name
        verified_birthday = birthday
        response = "Correct"
    else:
        verified_name = human_response.get("name", name)
        verified_birthday = human_response.get("birthday", birthday)
        response = f"Made a correction: {human_response}"

    state_update = {
        "name": verified_name,
        "birthday": verified_birthday,
        "messages": [ToolMessage(response, tool_call_id=tool_call_id)],
    }
    return Command(update=state_update)


tool = TavilySearch(max_results=2)
tools = [tool, human_assistance]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    message = llm_with_tools.invoke(state["messages"])
    assert(len(message.tool_calls) <= 1)
    return {"messages": [message]}

graph_builder = StateGraph(State)
graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=tools)
graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")

memory = InMemorySaver()
graph = graph_builder.compile(checkpointer=memory)
```

:::

:::js

```typescript
import {
  Command,
  interrupt,
  MessagesZodState,
  MemorySaver,
  StateGraph,
  END,
  START,
} from "@langchain/langgraph";
import { ToolNode, toolsCondition } from "@langchain/langgraph/prebuilt";
import { ChatAnthropic } from "@langchain/anthropic";
import { TavilySearch } from "@langchain/tavily";
import { ToolMessage } from "@langchain/core/messages";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const State = z.object({
  messages: MessagesZodState.shape.messages,
  name: z.string(),
  birthday: z.string(),
});

const humanAssistance = tool(
  async (input, config) => {
    // Note that because we are generating a ToolMessage for a state update, we
    // generally require the ID of the corresponding tool call. This is available
    // in the tool's config.
    const toolCallId = config?.toolCall?.id as string | undefined;
    if (!toolCallId) throw new Error("Tool call ID is required");

    const humanResponse = await interrupt({
      question: "Is this correct?",
      name: input.name,
      birthday: input.birthday,
    });

    // We explicitly update the state with a ToolMessage inside the tool.
    const stateUpdate = (() => {
      // If the information is correct, update the state as-is.
      if (humanResponse.correct?.toLowerCase().startsWith("y")) {
        return {
          name: input.name,
          birthday: input.birthday,
          messages: [
            new ToolMessage({ content: "Correct", tool_call_id: toolCallId }),
          ],
        };
      }

      // Otherwise, receive information from the human reviewer.
      return {
        name: humanResponse.name || input.name,
        birthday: humanResponse.birthday || input.birthday,
        messages: [
          new ToolMessage({
            content: `Made a correction: ${JSON.stringify(humanResponse)}`,
            tool_call_id: toolCallId,
          }),
        ],
      };
    })();

    // We return a Command object in the tool to update our state.
    return new Command({ update: stateUpdate });
  },
  {
    name: "humanAssistance",
    description: "Request assistance from a human.",
    schema: z.object({
      name: z.string().describe("The name of the entity"),
      birthday: z.string().describe("The birthday/release date of the entity"),
    }),
  }
);

const searchTool = new TavilySearch({ maxResults: 2 });

const tools = [searchTool, humanAssistance];
const llmWithTools = new ChatAnthropic({
  model: "claude-3-5-sonnet-latest",
}).bindTools(tools);

const memory = new MemorySaver();

const chatbot = async (state: z.infer<typeof State>) => {
  const message = await llmWithTools.invoke(state.messages);
  return { messages: message };
};

const graph = new StateGraph(State)
  .addNode("chatbot", chatbot)
  .addNode("tools", new ToolNode(tools))
  .addConditionalEdges("chatbot", toolsCondition, ["tools", END])
  .addEdge("tools", "chatbot")
  .addEdge(START, "chatbot")
  .compile({ checkpointer: memory });
```

:::

## Next steps

There's one more concept to review before finishing the LangGraph basics tutorials: connecting `checkpointing` and `state updates` to [time travel](./6-time-travel.md).

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/get-started/6-time-travel.md
```md
# Time travel

In a typical chatbot workflow, the user interacts with the bot one or more times to accomplish a task. [Memory](./3-add-memory.md) and a [human-in-the-loop](./4-human-in-the-loop.md) enable checkpoints in the graph state and control future responses.

What if you want a user to be able to start from a previous response and explore a different outcome? Or what if you want users to be able to rewind your chatbot's work to fix mistakes or try a different strategy, something that is common in applications like autonomous software engineers?

You can create these types of experiences using LangGraph's built-in **time travel** functionality.

!!! note

    This tutorial builds on [Customize state](./5-customize-state.md).

## 1. Rewind your graph

:::python
Rewind your graph by fetching a checkpoint using the graph's `get_state_history` method. You can then resume execution at this previous point in time.
:::

:::js
Rewind your graph by fetching a checkpoint using the graph's `getStateHistory` method. You can then resume execution at this previous point in time.
:::

:::python

{% include-markdown "../../../snippets/chat_model_tabs.md" %}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

```python
from typing import Annotated

from langchain_tavily import TavilySearch
from langchain_core.messages import BaseMessage
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

tool = TavilySearch(max_results=2)
tools = [tool]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=[tool])
graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")

memory = InMemorySaver()
graph = graph_builder.compile(checkpointer=memory)
```

:::

:::js

```typescript
import {
  StateGraph,
  START,
  END,
  MessagesZodState,
  MemorySaver,
} from "@langchain/langgraph";
import { ToolNode, toolsCondition } from "@langchain/langgraph/prebuilt";
import { TavilySearch } from "@langchain/tavily";
import { ChatOpenAI } from "@langchain/openai";
import { z } from "zod";

const State = z.object({ messages: MessagesZodState.shape.messages });

const tools = [new TavilySearch({ maxResults: 2 })];
const llmWithTools = new ChatOpenAI({ model: "gpt-4o-mini" }).bindTools(tools);
const memory = new MemorySaver();

const graph = new StateGraph(State)
  .addNode("chatbot", async (state) => ({
    messages: [await llmWithTools.invoke(state.messages)],
  }))
  .addNode("tools", new ToolNode(tools))
  .addConditionalEdges("chatbot", toolsCondition, ["tools", END])
  .addEdge("tools", "chatbot")
  .addEdge(START, "chatbot")
  .compile({ checkpointer: memory });
```

:::

## 2. Add steps

Add steps to your graph. Every step will be checkpointed in its state history:

:::python

```python
config = {"configurable": {"thread_id": "1"}}
events = graph.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": (
                    "I'm learning LangGraph. "
                    "Could you do some research on it for me?"
                ),
            },
        ],
    },
    config,
    stream_mode="values",
)
for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

I'm learning LangGraph. Could you do some research on it for me?
================================== Ai Message ==================================

[{'text': "Certainly! I'd be happy to research LangGraph for you. To get the most up-to-date and accurate information, I'll use the Tavily search engine to look this up. Let me do that for you now.", 'type': 'text'}, {'id': 'toolu_01BscbfJJB9EWJFqGrN6E54e', 'input': {'query': 'LangGraph latest information and features'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
Tool Calls:
  tavily_search_results_json (toolu_01BscbfJJB9EWJFqGrN6E54e)
 Call ID: toolu_01BscbfJJB9EWJFqGrN6E54e
  Args:
    query: LangGraph latest information and features
================================= Tool Message =================================
Name: tavily_search_results_json

[{"url": "https://blockchain.news/news/langchain-new-features-upcoming-events-update", "content": "LangChain, a leading platform in the AI development space, has released its latest updates, showcasing new use cases and enhancements across its ecosystem. According to the LangChain Blog, the updates cover advancements in LangGraph Platform, LangSmith's self-improving evaluators, and revamped documentation for LangGraph."}, {"url": "https://blog.langchain.dev/langgraph-platform-announce/", "content": "With these learnings under our belt, we decided to couple some of our latest offerings under LangGraph Platform. LangGraph Platform today includes LangGraph Server, LangGraph Studio, plus the CLI and SDK. ... we added features in LangGraph Server to deliver on a few key value areas. Below, we'll focus on these aspects of LangGraph Platform."}]
================================== Ai Message ==================================

Thank you for your patience. I've found some recent information about LangGraph for you. Let me summarize the key points:

1. LangGraph is part of the LangChain ecosystem, which is a leading platform in AI development.

2. Recent updates and features of LangGraph include:

   a. LangGraph Platform: This seems to be a cloud-based version of LangGraph, though specific details weren't provided in the search results.
...
3. Keep an eye on LangGraph Platform developments, as cloud-based solutions often provide an easier starting point for learners.
4. Consider how LangGraph fits into the broader LangChain ecosystem, especially its interaction with tools like LangSmith.

Is there any specific aspect of LangGraph you'd like to know more about? I'd be happy to do a more focused search on particular features or use cases.
Output is truncated. View as a scrollable element or open in a text editor. Adjust cell output settings...
```

```python
events = graph.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": (
                    "Ya that's helpful. Maybe I'll "
                    "build an autonomous agent with it!"
                ),
            },
        ],
    },
    config,
    stream_mode="values",
)
for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

Ya that's helpful. Maybe I'll build an autonomous agent with it!
================================== Ai Message ==================================

[{'text': "That's an exciting idea! Building an autonomous agent with LangGraph is indeed a great application of this technology. LangGraph is particularly well-suited for creating complex, multi-step AI workflows, which is perfect for autonomous agents. Let me gather some more specific information about using LangGraph for building autonomous agents.", 'type': 'text'}, {'id': 'toolu_01QWNHhUaeeWcGXvA4eHT7Zo', 'input': {'query': 'Building autonomous agents with LangGraph examples and tutorials'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
Tool Calls:
  tavily_search_results_json (toolu_01QWNHhUaeeWcGXvA4eHT7Zo)
 Call ID: toolu_01QWNHhUaeeWcGXvA4eHT7Zo
  Args:
    query: Building autonomous agents with LangGraph examples and tutorials
================================= Tool Message =================================
Name: tavily_search_results_json

[{"url": "https://towardsdatascience.com/building-autonomous-multi-tool-agents-with-gemini-2-0-and-langgraph-ad3d7bd5e79d", "content": "Building Autonomous Multi-Tool Agents with Gemini 2.0 and LangGraph | by Youness Mansar | Jan, 2025 | Towards Data Science Building Autonomous Multi-Tool Agents with Gemini 2.0 and LangGraph A practical tutorial with full code examples for building and running multi-tool agents Towards Data Science LLMs are remarkable — they can memorize vast amounts of information, answer general knowledge questions, write code, generate stories, and even fix your grammar. In this tutorial, we are going to build a simple LLM agent that is equipped with four tools that it can use to answer a user's question. This Agent will have the following specifications: Follow Published in Towards Data Science --------------------------------- Your home for data science and AI. Follow Follow Follow"}, {"url": "https://github.com/anmolaman20/Tools_and_Agents", "content": "GitHub - anmolaman20/Tools_and_Agents: This repository provides resources for building AI agents using Langchain and Langgraph. This repository provides resources for building AI agents using Langchain and Langgraph. This repository provides resources for building AI agents using Langchain and Langgraph. This repository serves as a comprehensive guide for building AI-powered agents using Langchain and Langgraph. It provides hands-on examples, practical tutorials, and resources for developers and AI enthusiasts to master building intelligent systems and workflows. AI Agent Development: Gain insights into creating intelligent systems that think, reason, and adapt in real time. This repository is ideal for AI practitioners, developers exploring language models, or anyone interested in building intelligent systems. This repository provides resources for building AI agents using Langchain and Langgraph."}]
================================== Ai Message ==================================

Great idea! Building an autonomous agent with LangGraph is definitely an exciting project. Based on the latest information I've found, here are some insights and tips for building autonomous agents with LangGraph:

1. Multi-Tool Agents: LangGraph is particularly well-suited for creating autonomous agents that can use multiple tools. This allows your agent to have a diverse set of capabilities and choose the right tool for each task.

2. Integration with Large Language Models (LLMs): You can combine LangGraph with powerful LLMs like Gemini 2.0 to create more intelligent and capable agents. The LLM can serve as the "brain" of your agent, making decisions and generating responses.

3. Workflow Management: LangGraph excels at managing complex, multi-step AI workflows. This is crucial for autonomous agents that need to break down tasks into smaller steps and execute them in the right order.
...
6. Pay attention to how you structure the agent's decision-making process and workflow.
7. Don't forget to implement proper error handling and safety measures, especially if your agent will be interacting with external systems or making important decisions.

Building an autonomous agent is an iterative process, so be prepared to refine and improve your agent over time. Good luck with your project! If you need any more specific information as you progress, feel free to ask.
Output is truncated. View as a scrollable element or open in a text editor. Adjust cell output settings...
```

:::

:::js

```typescript
import { randomUUID } from "node:crypto";
const threadId = randomUUID();

let iter = 0;

for (const userInput of [
  "I'm learning LangGraph. Could you do some research on it for me?",
  "Ya that's helpful. Maybe I'll build an autonomous agent with it!",
]) {
  iter += 1;

  console.log(`\n--- Conversation Turn ${iter} ---\n`);
  const events = await graph.stream(
    { messages: [{ role: "user", content: userInput }] },
    { configurable: { thread_id: threadId }, streamMode: "values" }
  );

  for await (const event of events) {
    if ("messages" in event) {
      const lastMessage = event.messages.at(-1);

      console.log(
        "=".repeat(32),
        `${lastMessage?.getType()} Message`,
        "=".repeat(32)
      );
      console.log(lastMessage?.text);
    }
  }
}
```

```
--- Conversation Turn 1 ---

================================ human Message ================================
I'm learning LangGraph.js. Could you do some research on it for me?
================================ ai Message ================================
I'll search for information about LangGraph.js for you.
================================ tool Message ================================
{
  "query": "LangGraph.js framework TypeScript langchain what is it tutorial guide",
  "follow_up_questions": null,
  "answer": null,
  "images": [],
  "results": [
    {
      "url": "https://techcommunity.microsoft.com/blog/educatordeveloperblog/an-absolute-beginners-guide-to-langgraph-js/4212496",
      "title": "An Absolute Beginner's Guide to LangGraph.js",
      "content": "(...)",
      "score": 0.79369855,
      "raw_content": null
    },
    {
      "url": "https://langchain-ai.github.io/langgraphjs/",
      "title": "LangGraph.js",
      "content": "(...)",
      "score": 0.78154784,
      "raw_content": null
    }
  ],
  "response_time": 2.37
}
================================ ai Message ================================
Let me provide you with an overview of LangGraph.js based on the search results:

LangGraph.js is a JavaScript/TypeScript library that's part of the LangChain ecosystem, specifically designed for creating and managing complex LLM (Large Language Model) based workflows. Here are the key points about LangGraph.js:

1. Purpose:
- It's a low-level orchestration framework for building controllable agents
- Particularly useful for creating agentic workflows where LLMs decide the course of action based on current state
- Helps model workflows as graphs with nodes and edges

(...)

--- Conversation Turn 2 ---

================================ human Message ================================
Ya that's helpful. Maybe I'll build an autonomous agent with it!
================================ ai Message ================================
Let me search for specific information about building autonomous agents with LangGraph.js.
================================ tool Message ================================
{
  "query": "how to build autonomous agents with LangGraph.js examples tutorial react agent",
  "follow_up_questions": null,
  "answer": null,
  "images": [],
  "results": [
    {
      "url": "https://ai.google.dev/gemini-api/docs/langgraph-example",
      "title": "ReAct agent from scratch with Gemini 2.5 and LangGraph",
      "content": "(...)",
      "score": 0.7602419,
      "raw_content": null
    },
    {
      "url": "https://www.youtube.com/watch?v=ZfjaIshGkmk",
      "title": "Build Autonomous AI Agents with ReAct and LangGraph Tools",
      "content": "(...)",
      "score": 0.7471924,
      "raw_content": null
    }
  ],
  "response_time": 1.98
}
================================ ai Message ================================
Based on the search results, I can provide you with a practical overview of how to build an autonomous agent with LangGraph.js. Here's what you need to know:

1. Basic Structure for Building an Agent:
- LangGraph.js provides a ReAct (Reason + Act) pattern implementation
- The basic components include:
  - State management for conversation history
  - Nodes for different actions
  - Edges for decision-making flow
  - Tools for specific functionalities

(...)

```

:::

## 3. Replay the full state history

Now that you have added steps to the chatbot, you can `replay` the full state history to see everything that occurred.

:::python

```python
to_replay = None
for state in graph.get_state_history(config):
    print("Num Messages: ", len(state.values["messages"]), "Next: ", state.next)
    print("-" * 80)
    if len(state.values["messages"]) == 6:
        # We are somewhat arbitrarily selecting a specific state based on the number of chat messages in the state.
        to_replay = state
```

```
Num Messages:  8 Next:  ()
--------------------------------------------------------------------------------
Num Messages:  7 Next:  ('chatbot',)
--------------------------------------------------------------------------------
Num Messages:  6 Next:  ('tools',)
--------------------------------------------------------------------------------
Num Messages:  5 Next:  ('chatbot',)
--------------------------------------------------------------------------------
Num Messages:  4 Next:  ('__start__',)
--------------------------------------------------------------------------------
Num Messages:  4 Next:  ()
--------------------------------------------------------------------------------
Num Messages:  3 Next:  ('chatbot',)
--------------------------------------------------------------------------------
Num Messages:  2 Next:  ('tools',)
--------------------------------------------------------------------------------
Num Messages:  1 Next:  ('chatbot',)
--------------------------------------------------------------------------------
Num Messages:  0 Next:  ('__start__',)
--------------------------------------------------------------------------------
```

:::

:::js

```typescript
import type { StateSnapshot } from "@langchain/langgraph";

let toReplay: StateSnapshot | undefined;
for await (const state of graph.getStateHistory({
  configurable: { thread_id: threadId },
})) {
  console.log(
    `Num Messages: ${state.values.messages.length}, Next: ${JSON.stringify(
      state.next
    )}`
  );
  console.log("-".repeat(80));
  if (state.values.messages.length === 6) {
    // We are somewhat arbitrarily selecting a specific state based on the number of chat messages in the state.
    toReplay = state;
  }
}
```

```
Num Messages: 8 Next:  []
--------------------------------------------------------------------------------
Num Messages: 7 Next:  ["chatbot"]
--------------------------------------------------------------------------------
Num Messages: 6 Next:  ["tools"]
--------------------------------------------------------------------------------
Num Messages: 7, Next: ["chatbot"]
--------------------------------------------------------------------------------
Num Messages: 6, Next: ["tools"]
--------------------------------------------------------------------------------
Num Messages: 5, Next: ["chatbot"]
--------------------------------------------------------------------------------
Num Messages: 4, Next: ["__start__"]
--------------------------------------------------------------------------------
Num Messages: 4, Next: []
--------------------------------------------------------------------------------
Num Messages: 3, Next: ["chatbot"]
--------------------------------------------------------------------------------
Num Messages: 2, Next: ["tools"]
--------------------------------------------------------------------------------
Num Messages: 1, Next: ["chatbot"]
--------------------------------------------------------------------------------
Num Messages: 0, Next: ["__start__"]
--------------------------------------------------------------------------------
```

:::

Checkpoints are saved for every step of the graph. This **spans invocations** so you can rewind across a full thread's history.

## Resume from a checkpoint

:::python

Resume from the `to_replay` state, which is after the `chatbot` node in the second graph invocation. Resuming from this point will call the **action** node next.
:::

:::js
Resume from the `toReplay` state, which is after a specific node in one of the graph invocations. Resuming from this point will call the next scheduled node.
:::

:::python

```python
print(to_replay.next)
print(to_replay.config)
```

```
('tools',)
{'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1efd43e3-0c1f-6c4e-8006-891877d65740'}}
```

:::

:::js

Resume from the `toReplay` state, which is after the `chatbot` node in one of the graph invocations. Resuming from this point will call the next scheduled node.

```typescript
console.log(toReplay.next);
console.log(toReplay.config);
```

```
["tools"]
{
  configurable: {
    thread_id: "007708b8-ea9b-4ff7-a7ad-3843364dbf75",
    checkpoint_ns: "",
    checkpoint_id: "1efd43e3-0c1f-6c4e-8006-891877d65740"
  }
}
```

:::

## 4. Load a state from a moment-in-time

:::python

The checkpoint's `to_replay.config` contains a `checkpoint_id` timestamp. Providing this `checkpoint_id` value tells LangGraph's checkpointer to **load** the state from that moment in time.

```python
# The `checkpoint_id` in the `to_replay.config` corresponds to a state we've persisted to our checkpointer.
for event in graph.stream(None, to_replay.config, stream_mode="values"):
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```
================================== Ai Message ==================================

[{'text': "That's an exciting idea! Building an autonomous agent with LangGraph is indeed a great application of this technology. LangGraph is particularly well-suited for creating complex, multi-step AI workflows, which is perfect for autonomous agents. Let me gather some more specific information about using LangGraph for building autonomous agents.", 'type': 'text'}, {'id': 'toolu_01QWNHhUaeeWcGXvA4eHT7Zo', 'input': {'query': 'Building autonomous agents with LangGraph examples and tutorials'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
Tool Calls:
  tavily_search_results_json (toolu_01QWNHhUaeeWcGXvA4eHT7Zo)
 Call ID: toolu_01QWNHhUaeeWcGXvA4eHT7Zo
  Args:
    query: Building autonomous agents with LangGraph examples and tutorials
================================= Tool Message =================================
Name: tavily_search_results_json

[{"url": "https://towardsdatascience.com/building-autonomous-multi-tool-agents-with-gemini-2-0-and-langgraph-ad3d7bd5e79d", "content": "Building Autonomous Multi-Tool Agents with Gemini 2.0 and LangGraph | by Youness Mansar | Jan, 2025 | Towards Data Science Building Autonomous Multi-Tool Agents with Gemini 2.0 and LangGraph A practical tutorial with full code examples for building and running multi-tool agents Towards Data Science LLMs are remarkable — they can memorize vast amounts of information, answer general knowledge questions, write code, generate stories, and even fix your grammar. In this tutorial, we are going to build a simple LLM agent that is equipped with four tools that it can use to answer a user's question. This Agent will have the following specifications: Follow Published in Towards Data Science --------------------------------- Your home for data science and AI. Follow Follow Follow"}, {"url": "https://github.com/anmolaman20/Tools_and_Agents", "content": "GitHub - anmolaman20/Tools_and_Agents: This repository provides resources for building AI agents using Langchain and Langgraph. This repository provides resources for building AI agents using Langchain and Langgraph. This repository provides resources for building AI agents using Langchain and Langgraph. This repository serves as a comprehensive guide for building AI-powered agents using Langchain and Langgraph. It provides hands-on examples, practical tutorials, and resources for developers and AI enthusiasts to master building intelligent systems and workflows. AI Agent Development: Gain insights into creating intelligent systems that think, reason, and adapt in real time. This repository is ideal for AI practitioners, developers exploring language models, or anyone interested in building intelligent systems. This repository provides resources for building AI agents using Langchain and Langgraph."}]
================================== Ai Message ==================================

Great idea! Building an autonomous agent with LangGraph is definitely an exciting project. Based on the latest information I've found, here are some insights and tips for building autonomous agents with LangGraph:

1. Multi-Tool Agents: LangGraph is particularly well-suited for creating autonomous agents that can use multiple tools. This allows your agent to have a diverse set of capabilities and choose the right tool for each task.

2. Integration with Large Language Models (LLMs): You can combine LangGraph with powerful LLMs like Gemini 2.0 to create more intelligent and capable agents. The LLM can serve as the "brain" of your agent, making decisions and generating responses.

3. Workflow Management: LangGraph excels at managing complex, multi-step AI workflows. This is crucial for autonomous agents that need to break down tasks into smaller steps and execute them in the right order.
...

Remember, building an autonomous agent is an iterative process. Start simple and gradually increase complexity as you become more comfortable with LangGraph and its capabilities.

Would you like more information on any specific aspect of building your autonomous agent with LangGraph?
Output is truncated. View as a scrollable element or open in a text editor. Adjust cell output settings...
```

The graph resumed execution from the `tools` node. You can tell this is the case since the first value printed above is the response from our search engine tool.
:::

:::js

The checkpoint's `toReplay.config` contains a `checkpoint_id` timestamp. Providing this `checkpoint_id` value tells LangGraph's checkpointer to **load** the state from that moment in time.

```typescript
// The `checkpoint_id` in the `toReplay.config` corresponds to a state we've persisted to our checkpointer.
for await (const event of await graph.stream(null, {
  ...toReplay?.config,
  streamMode: "values",
})) {
  if ("messages" in event) {
    const lastMessage = event.messages.at(-1);

    console.log(
      "=".repeat(32),
      `${lastMessage?.getType()} Message`,
      "=".repeat(32)
    );
    console.log(lastMessage?.text);
  }
}
```

```
================================ ai Message ================================
Let me search for specific information about building autonomous agents with LangGraph.js.
================================ tool Message ================================
{
  "query": "how to build autonomous agents with LangGraph.js examples tutorial",
  "follow_up_questions": null,
  "answer": null,
  "images": [],
  "results": [
    {
      "url": "https://www.mongodb.com/developer/languages/typescript/build-javascript-ai-agent-langgraphjs-mongodb/",
      "title": "Build a JavaScript AI Agent With LangGraph.js and MongoDB",
      "content": "(...)",
      "score": 0.7672197,
      "raw_content": null
    },
    {
      "url": "https://medium.com/@lorevanoudenhove/how-to-build-ai-agents-with-langgraph-a-step-by-step-guide-5d84d9c7e832",
      "title": "How to Build AI Agents with LangGraph: A Step-by-Step Guide",
      "content": "(...)",
      "score": 0.7407191,
      "raw_content": null
    }
  ],
  "response_time": 0.82
}
================================ ai Message ================================
Based on the search results, I can share some practical information about building autonomous agents with LangGraph.js. Here are some concrete examples and approaches:

1. Example HR Assistant Agent:
- Can handle HR-related queries using employee information
- Features include:
  - Starting and continuing conversations
  - Looking up information using vector search
  - Persisting conversation state using checkpoints
  - Managing threaded conversations

2. Energy Savings Calculator Agent:
- Functions as a lead generation tool for solar panel sales
- Capabilities include:
  - Calculating potential energy savings
  - Handling multi-step conversations
  - Processing user inputs for personalized estimates
  - Managing conversation state

(...)
```

The graph resumed execution from the `tools` node. You can tell this is the case since the first value printed above is the response from our search engine tool.
:::

**Congratulations!** You've now used time-travel checkpoint traversal in LangGraph. Being able to rewind and explore alternative paths opens up a world of possibilities for debugging, experimentation, and interactive applications.

## Learn more

Take your LangGraph journey further by exploring deployment and advanced features:

- **[LangGraph Server quickstart](../../tutorials/langgraph-platform/local-server.md)**: Launch a LangGraph server locally and interact with it using the REST API and LangGraph Studio Web UI.
- **[LangGraph Platform quickstart](../../cloud/quick_start.md)**: Deploy your LangGraph app using LangGraph Platform.
- **[LangGraph Platform concepts](../../concepts/langgraph_platform.md)**: Understand the foundational concepts of the LangGraph Platform.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/get-started/basic-chatbot.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/get-started/chatbot-with-tools.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/workflows/img/agent.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/workflows/img/augmented_llm.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/workflows/img/evaluator_optimizer.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/workflows/img/parallelization.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/workflows/img/prompt_chain.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/workflows/img/routing.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/workflows/img/worker.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/usaco/usaco.ipynb
```ipynb
{
 "cells": [
  {
   "attachments": {
    "a53accbd-8074-456e-8268-547edff14571.png": {
