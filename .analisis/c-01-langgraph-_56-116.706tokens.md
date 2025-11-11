```
> [truncated]

---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/additional-resources/index.md
```md
# Additional resources

This section contains additional resources for LangGraph.

- [Community agents](../agents/prebuilt.md): A collection of prebuilt libraries that you can use in your LangGraph applications.
- [LangGraph Academy](https://academy.langchain.com/courses/intro-to-langgraph): A collection of courses that teach you how to use LangGraph.
- [Case studies](../adopters.md): A collection of case studies that show how LangGraph is used in production.
- [FAQ](../concepts/faq.md): A collection of frequently asked questions about LangGraph.
- [llms.txt](../llms-txt-overview.md): A list of documentation files in the `llms.txt` format that allow LLMs and agents to access our documentation.
- [LangChain Forum](https://forum.langchain.com/): A place to ask questions and get help from other LangGraph users.
- [Troubleshooting](../troubleshooting/errors/index.md): A collection of troubleshooting guides for common issues.
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/.meta.yml
```yml
tags:
  - how-tos
  - how-to
  - howto
  - how to
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/autogen-integration-functional.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "100c0c81-6a9f-4ba1-b1a8-42aae82b7172",
   "metadata": {},
   "source": [
    "# How to integrate LangGraph (functional API) with AutoGen, CrewAI, and other frameworks\n",
    "\n",
    "LangGraph is a framework for building agentic and multi-agent applications. LangGraph can be easily integrated with other agent frameworks. \n",
    "\n",
    "The primary reasons you might want to integrate LangGraph with other agent frameworks:\n",
    "\n",
    "- create [multi-agent systems](../../concepts/multi_agent) where individual agents are built with different frameworks\n",
    "- leverage LangGraph to add features like [persistence](../../concepts/persistence), [streaming](../../concepts/streaming), [short and long-term memory](../../concepts/memory) and more\n",
    "\n",
    "The simplest way to integrate agents from other frameworks is by calling those agents inside a LangGraph [node](../../concepts/low_level/#nodes):\n",
    "\n",
    "```python\n",
    "import autogen\n",
    "from langgraph.func import entrypoint, task\n",
    "\n",
    "autogen_agent = autogen.AssistantAgent(name=\"assistant\", ...)\n",
    "user_proxy = autogen.UserProxyAgent(name=\"user_proxy\", ...)\n",
    "\n",
    "@task\n",
    "def call_autogen_agent(messages):\n",
    "    response = user_proxy.initiate_chat(\n",
    "        autogen_agent,\n",
    "        message=messages[-1],\n",
    "        ...\n",
    "    )\n",
    "    ...\n",
    "\n",
    "\n",
    "@entrypoint()\n",
    "def workflow(messages):\n",
    "    response = call_autogen_agent(messages).result()\n",
    "    return response\n",
    "\n",
    "\n",
    "workflow.invoke(\n",
    "    [\n",
    "        {\n",
    "            \"role\": \"user\",\n",
    "            \"content\": \"Find numbers between 10 and 30 in fibonacci sequence\",\n",
    "        }\n",
    "    ]\n",
    ")\n",
    "```\n",
    "\n",
    "In this guide we show how to build a LangGraph chatbot that integrates with AutoGen, but you can follow the same approach with other frameworks."
   ]
  },
  {
   "cell_type": "markdown",
   "id": "b189ceb2-132b-4c7b-81b4-c7b8b062f833",
   "metadata": {},
   "source": [
    "## Setup"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 1,
   "id": "62417d3a-94f9-4a52-9962-12639d714966",
   "metadata": {},
   "outputs": [],
   "source": [
    "%pip install autogen langgraph"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 2,
   "id": "d46da41d-0a71-4654-aec8-9e6ad8765236",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "OPENAI_API_KEY:  ········\n"
     ]
    }
   ],
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
   "id": "1926bbc3-6b06-41e0-9604-860a2bbf8fa3",
   "metadata": {},
   "source": [
    "## Define AutoGen agent\n",
    "\n",
    "Here we define our AutoGen agent. Adapted from official tutorial [here](https://github.com/microsoft/autogen/blob/0.2/notebook/agentchat_web_info.ipynb)."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 3,
   "id": "524de117-ff09-4b26-bfe8-a9f85a46ffd5",
   "metadata": {},
   "outputs": [],
   "source": [
    "import autogen\n",
    "import os\n",
    "\n",
    "config_list = [{\"model\": \"gpt-4o\", \"api_key\": os.environ[\"OPENAI_API_KEY\"]}]\n",
    "\n",
    "llm_config = {\n",
    "    \"timeout\": 600,\n",
    "    \"cache_seed\": 42,\n",
    "    \"config_list\": config_list,\n",
    "    \"temperature\": 0,\n",
    "}\n",
    "\n",
    "autogen_agent = autogen.AssistantAgent(\n",
    "    name=\"assistant\",\n",
    "    llm_config=llm_config,\n",
    ")\n",
    "\n",
    "user_proxy = autogen.UserProxyAgent(\n",
    "    name=\"user_proxy\",\n",
    "    human_input_mode=\"NEVER\",\n",
    "    max_consecutive_auto_reply=10,\n",
    "    is_termination_msg=lambda x: x.get(\"content\", \"\").rstrip().endswith(\"TERMINATE\"),\n",
    "    code_execution_config={\n",
    "        \"work_dir\": \"web\",\n",
    "        \"use_docker\": False,\n",
    "    },  # Please set use_docker=True if docker is available to run the generated code. Using docker is safer than running the generated code directly.\n",
    "    llm_config=llm_config,\n",
    "    system_message=\"Reply TERMINATE if the task has been solved at full satisfaction. Otherwise, reply CONTINUE, or the reason why the task is not solved yet.\",\n",
    ")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "8aa858e2-4acb-4f75-be20-b9ccbbcb5073",
   "metadata": {},
   "source": [
    "---"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "dcc478f5-4a35-43f8-bf59-9cb71289cd00",
   "metadata": {},
   "source": [
    "## Create the workflow\n",
    "\n",
    "We will now create a LangGraph chatbot graph that calls AutoGen agent."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "d129e4e1-3766-429a-b806-cde3d8bc0469",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langchain_core.messages import convert_to_openai_messages, BaseMessage\n",
    "from langgraph.func import entrypoint, task\n",
    "from langgraph.graph import add_messages\n",
    "from langgraph.checkpoint.memory import InMemorySaver\n",
    "\n",
    "\n",
    "@task\n",
    "def call_autogen_agent(messages: list[BaseMessage]):\n",
    "    # convert to openai-style messages\n",
    "    messages = convert_to_openai_messages(messages)\n",
    "    response = user_proxy.initiate_chat(\n",
    "        autogen_agent,\n",
    "        message=messages[-1],\n",
    "        # pass previous message history as context\n",
    "        carryover=messages[:-1],\n",
    "    )\n",
    "    # get the final response from the agent\n",
    "    content = response.chat_history[-1][\"content\"]\n",
    "    return {\"role\": \"assistant\", \"content\": content}\n",
    "\n",
    "\n",
    "# add short-term memory for storing conversation history\n",
    "checkpointer = InMemorySaver()\n",
    "\n",
    "\n",
    "@entrypoint(checkpointer=checkpointer)\n",
    "def workflow(messages: list[BaseMessage], previous: list[BaseMessage]):\n",
    "    messages = add_messages(previous or [], messages)\n",
    "    response = call_autogen_agent(messages).result()\n",
    "    return entrypoint.final(value=response, save=add_messages(messages, response))"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "23d629c3-1d6b-40af-adf6-915e15657566",
   "metadata": {},
   "source": [
    "## Run the graph\n",
    "\n",
    "We can now run the graph."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 10,
   "id": "a279b667-0f5d-4008-8d43-c806a3f379c4",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "\u001b[33muser_proxy\u001b[0m (to assistant):\n",
      "\n",
      "Find numbers between 10 and 30 in fibonacci sequence\n",
      "\n",
      "--------------------------------------------------------------------------------\n",
      "\u001b[33massistant\u001b[0m (to user_proxy):\n",
      "\n",
      "To find numbers between 10 and 30 in the Fibonacci sequence, we can generate the Fibonacci sequence and check which numbers fall within this range. Here's a plan:\n",
      "\n",
      "1. Generate Fibonacci numbers starting from 0.\n",
      "2. Continue generating until the numbers exceed 30.\n",
      "3. Collect and print the numbers that are between 10 and 30.\n",
      "\n",
      "Let's implement this in Python:\n",
      "\n",
      "```python\n",
      "# filename: fibonacci_range.py\n",
      "\n",
      "def fibonacci_sequence():\n",
      "    a, b = 0, 1\n",
      "    while a <= 30:\n",
      "        if 10 <= a <= 30:\n",
      "            print(a)\n",
      "        a, b = b, a + b\n",
      "\n",
      "fibonacci_sequence()\n",
      "```\n",
      "\n",
      "This script will print the Fibonacci numbers between 10 and 30. Please execute the code to see the result.\n",
      "\n",
      "--------------------------------------------------------------------------------\n",
      "\u001b[31m\n",
      ">>>>>>>> EXECUTING CODE BLOCK 0 (inferred language is python)...\u001b[0m\n",
      "\u001b[33muser_proxy\u001b[0m (to assistant):\n",
      "\n",
      "exitcode: 0 (execution succeeded)\n",
      "Code output: \n",
      "13\n",
      "21\n",
      "\n",
      "\n",
      "--------------------------------------------------------------------------------\n",
      "\u001b[33massistant\u001b[0m (to user_proxy):\n",
      "\n",
      "The Fibonacci numbers between 10 and 30 are 13 and 21. \n",
      "\n",
      "These numbers are part of the Fibonacci sequence, which is generated by adding the two preceding numbers to get the next number, starting from 0 and 1. \n",
      "\n",
      "The sequence goes: 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, ...\n",
      "\n",
      "As you can see, 13 and 21 are the only numbers in this sequence that fall between 10 and 30.\n",
      "\n",
      "TERMINATE\n",
      "\n",
      "--------------------------------------------------------------------------------\n",
      "{'call_autogen_agent': {'role': 'assistant', 'content': 'The Fibonacci numbers between 10 and 30 are 13 and 21. \\n\\nThese numbers are part of the Fibonacci sequence, which is generated by adding the two preceding numbers to get the next number, starting from 0 and 1. \\n\\nThe sequence goes: 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, ...\\n\\nAs you can see, 13 and 21 are the only numbers in this sequence that fall between 10 and 30.\\n\\nTERMINATE'}}\n",
      "{'workflow': {'role': 'assistant', 'content': 'The Fibonacci numbers between 10 and 30 are 13 and 21. \\n\\nThese numbers are part of the Fibonacci sequence, which is generated by adding the two preceding numbers to get the next number, starting from 0 and 1. \\n\\nThe sequence goes: 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, ...\\n\\nAs you can see, 13 and 21 are the only numbers in this sequence that fall between 10 and 30.\\n\\nTERMINATE'}}\n"
     ]
    }
   ],
   "source": [
    "# pass the thread ID to persist agent outputs for future interactions\n",
    "# highlight-next-line\n",
    "config = {\"configurable\": {\"thread_id\": \"1\"}}\n",
    "\n",
    "for chunk in workflow.stream(\n",
    "    [\n",
    "        {\n",
    "            \"role\": \"user\",\n",
    "            \"content\": \"Find numbers between 10 and 30 in fibonacci sequence\",\n",
    "        }\n",
    "    ],\n",
    "    # highlight-next-line\n",
    "    config,\n",
    "):\n",
    "    print(chunk)"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "c6cd57b4-d4ee-49f6-be12-318613849669",
   "metadata": {},
   "source": [
    "Since we're leveraging LangGraph's [persistence](https://langchain-ai.github.io/langgraph/concepts/persistence/) features we can now continue the conversation using the same thread ID -- LangGraph will automatically pass previous history to the AutoGen agent:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 12,
   "id": "e68811a7-962e-4fe3-9f45-9b99ebbe04e7",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "\u001b[33muser_proxy\u001b[0m (to assistant):\n",
      "\n",
      "Multiply the last number by 3\n",
      "Context: \n",
      "Find numbers between 10 and 30 in fibonacci sequence\n",
      "The Fibonacci numbers between 10 and 30 are 13 and 21. \n",
      "\n",
      "These numbers are part of the Fibonacci sequence, which is generated by adding the two preceding numbers to get the next number, starting from 0 and 1. \n",
      "\n",
      "The sequence goes: 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, ...\n",
      "\n",
      "As you can see, 13 and 21 are the only numbers in this sequence that fall between 10 and 30.\n",
      "\n",
      "TERMINATE\n",
      "\n",
      "--------------------------------------------------------------------------------\n",
      "\u001b[33massistant\u001b[0m (to user_proxy):\n",
      "\n",
      "The last number in the Fibonacci sequence between 10 and 30 is 21. Multiplying 21 by 3 gives:\n",
      "\n",
      "21 * 3 = 63\n",
      "\n",
      "TERMINATE\n",
      "\n",
      "--------------------------------------------------------------------------------\n",
      "{'call_autogen_agent': {'role': 'assistant', 'content': 'The last number in the Fibonacci sequence between 10 and 30 is 21. Multiplying 21 by 3 gives:\\n\\n21 * 3 = 63\\n\\nTERMINATE'}}\n",
      "{'workflow': {'role': 'assistant', 'content': 'The last number in the Fibonacci sequence between 10 and 30 is 21. Multiplying 21 by 3 gives:\\n\\n21 * 3 = 63\\n\\nTERMINATE'}}\n"
     ]
    }
   ],
   "source": [
    "for chunk in workflow.stream(\n",
    "    [\n",
    "        {\n",
    "            \"role\": \"user\",\n",
    "            \"content\": \"Multiply the last number by 3\",\n",
    "        }\n",
    "    ],\n",
    "    # highlight-next-line\n",
    "    config,\n",
    "):\n",
    "    print(chunk)"
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
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/autogen-integration.md
```md
# How to integrate LangGraph with AutoGen, CrewAI, and other frameworks

This guide shows how to integrate AutoGen agents with LangGraph to leverage features like persistence, streaming, and memory, and then deploy the integrated solution to LangGraph Platform for scalable production use. In this guide we show how to build a LangGraph chatbot that integrates with AutoGen, but you can follow the same approach with other frameworks.

Integrating AutoGen with LangGraph provides several benefits:

- Enhanced features: Add [persistence](../concepts/persistence.md), [streaming](../concepts/streaming.md), [short and long-term memory](../concepts/memory.md) and more to your AutoGen agents.
- Multi-agent systems: Build [multi-agent systems](../concepts/multi_agent.md) where individual agents are built with different frameworks.
- Production deployment: Deploy your integrated solution to [LangGraph Platform](../concepts/langgraph_platform.md) for scalable production use.

## Prerequisites

- Python 3.9+
- Autogen: `pip install autogen`
- LangGraph: `pip install langgraph`
- OpenAI API key

## Setup

Set your your environment:

```python
import getpass
import os


def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")


_set_env("OPENAI_API_KEY")
```

## 1. Define AutoGen agent

Create an AutoGen agent that can execute code. This example is adapted from AutoGen's [official tutorials](https://github.com/microsoft/autogen/blob/0.2/notebook/agentchat_web_info.ipynb):

```python
import autogen
import os

config_list = [{"model": "gpt-4o", "api_key": os.environ["OPENAI_API_KEY"]}]

llm_config = {
    "timeout": 600,
    "cache_seed": 42,
    "config_list": config_list,
    "temperature": 0,
}

autogen_agent = autogen.AssistantAgent(
    name="assistant",
    llm_config=llm_config,
)

user_proxy = autogen.UserProxyAgent(
    name="user_proxy",
    human_input_mode="NEVER",
    max_consecutive_auto_reply=10,
    is_termination_msg=lambda x: x.get("content", "").rstrip().endswith("TERMINATE"),
    code_execution_config={
        "work_dir": "web",
        "use_docker": False,
    },  # Please set use_docker=True if docker is available to run the generated code. Using docker is safer than running the generated code directly.
    llm_config=llm_config,
    system_message="Reply TERMINATE if the task has been solved at full satisfaction. Otherwise, reply CONTINUE, or the reason why the task is not solved yet.",
)
```

## 2. Create the graph

We will now create a LangGraph chatbot graph that calls AutoGen agent.

```python
from langchain_core.messages import convert_to_openai_messages
from langgraph.graph import StateGraph, MessagesState, START
from langgraph.checkpoint.memory import InMemorySaver

def call_autogen_agent(state: MessagesState):
    # Convert LangGraph messages to OpenAI format for AutoGen
    messages = convert_to_openai_messages(state["messages"])
    
    # Get the last user message
    last_message = messages[-1]
    
    # Pass previous message history as context (excluding the last message)
    carryover = messages[:-1] if len(messages) > 1 else []
    
    # Initiate chat with AutoGen
    response = user_proxy.initiate_chat(
        autogen_agent,
        message=last_message,
        carryover=carryover
    )
    
    # Extract the final response from the agent
    final_content = response.chat_history[-1]["content"]
    
    # Return the response in LangGraph format
    return {"messages": {"role": "assistant", "content": final_content}}

# Create the graph with memory for persistence
checkpointer = InMemorySaver()

# Build the graph
builder = StateGraph(MessagesState)
builder.add_node("autogen", call_autogen_agent)
builder.add_edge(START, "autogen")

# Compile with checkpointer for persistence
graph = builder.compile(checkpointer=checkpointer)
```

```python
from IPython.display import display, Image

display(Image(graph.get_graph().draw_mermaid_png()))
```

![Graph](./assets/autogen-output.png)

## 3. Test the graph locally

Before deploying to LangGraph Platform, you can test the graph locally:

```python
# pass the thread ID to persist agent outputs for future interactions
# highlight-next-line
config = {"configurable": {"thread_id": "1"}}

for chunk in graph.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "Find numbers between 10 and 30 in fibonacci sequence",
            }
        ]
    },
    # highlight-next-line
    config,
):
    print(chunk)
```

**Output:**
```
user_proxy (to assistant):

Find numbers between 10 and 30 in fibonacci sequence

--------------------------------------------------------------------------------
assistant (to user_proxy):

To find numbers between 10 and 30 in the Fibonacci sequence, we can generate the Fibonacci sequence and check which numbers fall within this range. Here's a plan:

1. Generate Fibonacci numbers starting from 0.
2. Continue generating until the numbers exceed 30.
3. Collect and print the numbers that are between 10 and 30.

...
```

Since we're leveraging LangGraph's [persistence](https://langchain-ai.github.io/langgraph/concepts/persistence/) features we can now continue the conversation using the same thread ID -- LangGraph will automatically pass previous history to the AutoGen agent:

```python
for chunk in graph.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "Multiply the last number by 3",
            }
        ]
    },
    # highlight-next-line
    config,
):
    print(chunk)
```

**Output:**
```
user_proxy (to assistant):

Multiply the last number by 3
Context: 
Find numbers between 10 and 30 in fibonacci sequence
The Fibonacci numbers between 10 and 30 are 13 and 21. 

These numbers are part of the Fibonacci sequence, which is generated by adding the two preceding numbers to get the next number, starting from 0 and 1. 

The sequence goes: 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, ...

As you can see, 13 and 21 are the only numbers in this sequence that fall between 10 and 30.

TERMINATE

--------------------------------------------------------------------------------
assistant (to user_proxy):

The last number in the Fibonacci sequence between 10 and 30 is 21. Multiplying 21 by 3 gives:

21 * 3 = 63

TERMINATE

--------------------------------------------------------------------------------
{'call_autogen_agent': {'messages': {'role': 'assistant', 'content': 'The last number in the Fibonacci sequence between 10 and 30 is 21. Multiplying 21 by 3 gives:\n\n21 * 3 = 63\n\nTERMINATE'}}}
``` 

## 4. Prepare for deployment

To deploy to LangGraph Platform, create a file structure like the following:

```
my-autogen-agent/
├── agent.py          # Your main agent code
├── requirements.txt  # Python dependencies
└── langgraph.json   # LangGraph configuration
```

=== "agent.py"

    ```python
    import os
    import autogen
    from langchain_core.messages import convert_to_openai_messages
    from langgraph.graph import StateGraph, MessagesState, START
    from langgraph.checkpoint.memory import InMemorySaver

    # AutoGen configuration
    config_list = [{"model": "gpt-4o", "api_key": os.environ["OPENAI_API_KEY"]}]

    llm_config = {
        "timeout": 600,
        "cache_seed": 42,
        "config_list": config_list,
        "temperature": 0,
    }

    # Create AutoGen agents
    autogen_agent = autogen.AssistantAgent(
        name="assistant",
        llm_config=llm_config,
    )

    user_proxy = autogen.UserProxyAgent(
        name="user_proxy",
        human_input_mode="NEVER",
        max_consecutive_auto_reply=10,
        is_termination_msg=lambda x: x.get("content", "").rstrip().endswith("TERMINATE"),
        code_execution_config={
            "work_dir": "/tmp/autogen_work",
            "use_docker": False,
        },
        llm_config=llm_config,
        system_message="Reply TERMINATE if the task has been solved at full satisfaction.",
    )

    def call_autogen_agent(state: MessagesState):
        """Node function that calls the AutoGen agent"""
        messages = convert_to_openai_messages(state["messages"])
        last_message = messages[-1]
        carryover = messages[:-1] if len(messages) > 1 else []
        
        response = user_proxy.initiate_chat(
            autogen_agent,
            message=last_message,
            carryover=carryover
        )
        
        final_content = response.chat_history[-1]["content"]
        return {"messages": {"role": "assistant", "content": final_content}}

    # Create and compile the graph
    def create_graph():
        checkpointer = InMemorySaver()
        builder = StateGraph(MessagesState)
        builder.add_node("autogen", call_autogen_agent)
        builder.add_edge(START, "autogen")
        return builder.compile(checkpointer=checkpointer)

    # Export the graph for LangGraph Platform
    graph = create_graph()
    ```

=== "requirements.txt"

    ```
    langgraph>=0.1.0
    ag2>=0.2.0
    langchain-core>=0.1.0
    langchain-openai>=0.0.5
    ```

=== "langgraph.json"

    ```json
    {
    "dependencies": ["."],
    "graphs": {
        "autogen_agent": "./agent.py:graph"
    },
    "env": ".env"
    }
    ```


## 5. Deploy to LangGraph Platform

Deploy the graph with the LangGraph Platform CLI:

```
pip install -U langgraph-cli
```

```
langgraph deploy --config langgraph.json 
```

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/create-react-agent-manage-message-history.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "992c4695-ec4f-428d-bd05-fb3b5fbd70f4",
   "metadata": {},
   "source": [
    "# How to manage conversation history in a ReAct Agent\n",
    "\n",
    "!!! info \"Prerequisites\"\n",
    "    This guide assumes familiarity with the following:\n",
    "\n",
    "    - [Prebuilt create_react_agent](../create-react-agent)\n",
    "    - [Persistence](../../concepts/persistence)\n",
    "    - [Short-term Memory](../../concepts/memory/#short-term-memory)\n",
    "    - [Trimming Messages](https://python.langchain.com/docs/how_to/trim_messages/)\n",
    "\n",
    "Message history can grow quickly and exceed LLM context window size, whether you're building chatbots with many conversation turns or agentic systems with numerous tool calls. There are several strategies for managing the message history:\n",
    "\n",
    "* [message trimming](#keep-the-original-message-history-unmodified) — remove first or last N messages in the history\n",
    "* [summarization](#summarizing-message-history) — summarize earlier messages in the history and replace them with a summary\n",
    "* custom strategies (e.g., message filtering, etc.)\n",
    "\n",
    "To manage message history in `create_react_agent`, you need to define a `pre_model_hook` function or [runnable](https://python.langchain.com/docs/concepts/runnables/) that takes graph state an returns a state update:\n",
    "\n",
    "\n",
    "* Trimming example:\n",
    "    ```python\n",
    "    # highlight-next-line\n",
    "    from langchain_core.messages.utils import (\n",
    "        # highlight-next-line\n",
    "        trim_messages, \n",
    "        # highlight-next-line\n",
    "        count_tokens_approximately\n",
    "    # highlight-next-line\n",
    "    )\n",
    "    from langgraph.prebuilt import create_react_agent\n",
    "    \n",
    "    # This function will be called every time before the node that calls LLM\n",
    "    def pre_model_hook(state):\n",
    "        trimmed_messages = trim_messages(\n",
    "            state[\"messages\"],\n",
    "            strategy=\"last\",\n",
    "            token_counter=count_tokens_approximately,\n",
    "            max_tokens=384,\n",
    "            start_on=\"human\",\n",
    "            end_on=(\"human\", \"tool\"),\n",
    "        )\n",
    "        # You can return updated messages either under `llm_input_messages` or \n",
    "        # `messages` key (see the note below)\n",
    "        # highlight-next-line\n",
    "        return {\"llm_input_messages\": trimmed_messages}\n",
    "\n",
    "    checkpointer = InMemorySaver()\n",
    "    agent = create_react_agent(\n",
    "        model,\n",
    "        tools,\n",
    "        # highlight-next-line\n",
    "        pre_model_hook=pre_model_hook,\n",
    "        checkpointer=checkpointer,\n",
    "    )\n",
    "    ```\n",
    "\n",
    "* Summarization example:\n",
    "    ```python\n",
    "    # highlight-next-line\n",
    "    from langmem.short_term import SummarizationNode\n",
    "    from langchain_core.messages.utils import count_tokens_approximately\n",
    "    from langgraph.prebuilt.chat_agent_executor import AgentState\n",
    "    from langgraph.checkpoint.memory import InMemorySaver\n",
    "    from typing import Any\n",
    "    \n",
    "    model = ChatOpenAI(model=\"gpt-4o\")\n",
    "    \n",
    "    summarization_node = SummarizationNode(\n",
    "        token_counter=count_tokens_approximately,\n",
    "        model=model,\n",
    "        max_tokens=384,\n",
    "        max_summary_tokens=128,\n",
    "        output_messages_key=\"llm_input_messages\",\n",
    "    )\n",
    "\n",
    "    class State(AgentState):\n",
    "        # NOTE: we're adding this key to keep track of previous summary information\n",
    "        # to make sure we're not summarizing on every LLM call\n",
    "        # highlight-next-line\n",
    "        context: dict[str, Any]\n",
    "    \n",
    "    \n",
    "    checkpointer = InMemorySaver()\n",
    "    graph = create_react_agent(\n",
    "        model,\n",
    "        tools,\n",
    "        # highlight-next-line\n",
    "        pre_model_hook=summarization_node,\n",
    "        # highlight-next-line\n",
    "        state_schema=State,\n",
    "        checkpointer=checkpointer,\n",
    "    )\n",
    "    ```\n",
    "\n",
    "!!! Important\n",
    "    \n",
    "    * To **keep the original message history unmodified** in the graph state and pass the updated history **only as the input to the LLM**, return updated messages under `llm_input_messages` key\n",
    "    * To **overwrite the original message history** in the graph state with the updated history, return updated messages under `messages` key\n",
    "    \n",
    "    To overwrite the `messages` key, you need to do the following:\n",
    "\n",
    "    ```python\n",
    "    from langchain_core.messages import RemoveMessage\n",
    "    from langgraph.graph.message import REMOVE_ALL_MESSAGES\n",
    "\n",
    "    def pre_model_hook(state):\n",
    "        updated_messages = ...\n",
    "        return {\n",
    "            \"messages\": [RemoveMessage(id=REMOVE_ALL_MESSAGES), *updated_messages]\n",
    "            ...\n",
    "        }\n",
    "    ```"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "7be3889f-3c17-4fa1-bd2b-84114a2c7247",
   "metadata": {},
   "source": [
    "## Setup\n",
    "\n",
    "First, let's install the required packages and set our API keys"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 1,
   "id": "a213e11a-5c62-4ddb-a707-490d91add383",
   "metadata": {},
   "outputs": [],
   "source": [
    "%%capture --no-stderr\n",
    "%pip install -U langgraph langchain-openai langmem"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "23a1885c-04ab-4750-aefa-105891fddf3e",
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
   "id": "87a00ce9",
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
   "id": "03c0f089-070c-4cd4-87e0-6c51f2477b82",
   "metadata": {},
   "source": [
    "## Keep the original message history unmodified"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "cd6cbd3a-8632-47ae-9ec5-eec8d7b05cae",
   "metadata": {},
   "source": [
    "Let's build a ReAct agent with a step that manages the conversation history: when the length of the history exceeds a specified number of tokens, we will call [`trim_messages`](https://python.langchain.com/api_reference/core/messages/langchain_core.messages.utils.trim_messages.html) utility that that will reduce the history while satisfying LLM provider constraints.\n",
    "\n",
    "There are two ways that the updated message history can be applied inside ReAct agent:\n",
    "\n",
    "  * [**Keep the original message history unmodified**](#keep-the-original-message-history-unmodified) in the graph state and pass the updated history **only as the input to the LLM**\n",
    "  * [**Overwrite the original message history**](#overwrite-the-original-message-history) in the graph state with the updated history\n",
    "\n",
    "Let's start by implementing the first one. We'll need to first define model and tools for our agent:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 3,
   "id": "eaad19ee-e174-4c6c-b2b8-3530d7acea40",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langchain_openai import ChatOpenAI\n",
    "\n",
    "model = ChatOpenAI(model=\"gpt-4o\", temperature=0)\n",
    "\n",
    "\n",
    "def get_weather(location: str) -> str:\n",
    "    \"\"\"Use this to get weather information.\"\"\"\n",
    "    if any([city in location.lower() for city in [\"nyc\", \"new york city\"]]):\n",
    "        return \"It might be cloudy in nyc, with a chance of rain and temperatures up to 80 degrees.\"\n",
    "    elif any([city in location.lower() for city in [\"sf\", \"san francisco\"]]):\n",
    "        return \"It's always sunny in sf\"\n",
    "    else:\n",
    "        return f\"I am not sure what the weather is in {location}\"\n",
    "\n",
    "\n",
    "tools = [get_weather]"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "52402333-61ab-47d3-8549-6a70f6f1cf36",
   "metadata": {},
   "source": [
    "Now let's implement `pre_model_hook` — a function that will be added as a new node and called every time **before** the node that calls the LLM (the `agent` node).\n",
    "\n",
    "Our implementation will wrap the `trim_messages` call and return the trimmed messages under `llm_input_messages`. This will **keep the original message history unmodified** in the graph state and pass the updated history **only as the input to the LLM**"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 4,
   "id": "b507eb58-6e02-4ac6-b48b-ea4defdc11f0",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langgraph.prebuilt import create_react_agent\n",
    "from langgraph.checkpoint.memory import InMemorySaver\n",
    "\n",
    "# highlight-next-line\n",
    "from langchain_core.messages.utils import (\n",
    "    # highlight-next-line\n",
    "    trim_messages,\n",
    "    # highlight-next-line\n",
    "    count_tokens_approximately,\n",
    "    # highlight-next-line\n",
    ")\n",
    "\n",
    "\n",
    "# This function will be added as a new node in ReAct agent graph\n",
    "# that will run every time before the node that calls the LLM.\n",
    "# The messages returned by this function will be the input to the LLM.\n",
    "def pre_model_hook(state):\n",
    "    trimmed_messages = trim_messages(\n",
    "        state[\"messages\"],\n",
    "        strategy=\"last\",\n",
    "        token_counter=count_tokens_approximately,\n",
    "        max_tokens=384,\n",
    "        start_on=\"human\",\n",
    "        end_on=(\"human\", \"tool\"),\n",
    "    )\n",
    "    # highlight-next-line\n",
    "    return {\"llm_input_messages\": trimmed_messages}\n",
    "\n",
    "\n",
    "checkpointer = InMemorySaver()\n",
    "graph = create_react_agent(\n",
    "    model,\n",
    "    tools,\n",
    "    # highlight-next-line\n",
    "    pre_model_hook=pre_model_hook,\n",
    "    checkpointer=checkpointer,\n",
    ")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 5,
   "id": "8182ab45-86b3-4d6f-b75e-58862a14fa4e",
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
    "from IPython.display import display, Image\n",
    "\n",
    "display(Image(graph.get_graph().draw_mermaid_png()))"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "d41e8e76-5d43-44cd-bf01-a39212cedd8d",
   "metadata": {},
   "source": [
    "We'll also define a utility to render the agent outputs nicely:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 6,
   "id": "16636975-5f2d-4dc7-ab8e-d0bea0830a28",
   "metadata": {},
   "outputs": [],
   "source": [
    "def print_stream(stream, output_messages_key=\"llm_input_messages\"):\n",
    "    for chunk in stream:\n",
    "        for node, update in chunk.items():\n",
    "            print(f\"Update from node: {node}\")\n",
    "            messages_key = (\n",
    "                output_messages_key if node == \"pre_model_hook\" else \"messages\"\n",
    "            )\n",
    "            for message in update[messages_key]:\n",
    "                if isinstance(message, tuple):\n",
    "                    print(message)\n",
    "                else:\n",
    "                    message.pretty_print()\n",
    "\n",
    "        print(\"\\n\\n\")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "84448d29-b323-4833-80fc-4fff2f5a0950",
   "metadata": {},
   "source": [
    "Now let's run the agent with a few different queries to reach the specified max tokens limit:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 7,
   "id": "9ffff6c3-a4f5-47c9-b51d-97caaee85cd6",
   "metadata": {},
   "outputs": [],
   "source": [
    "config = {\"configurable\": {\"thread_id\": \"1\"}}\n",
    "\n",
    "inputs = {\"messages\": [(\"user\", \"What's the weather in NYC?\")]}\n",
    "result = graph.invoke(inputs, config=config)\n",
    "\n",
    "inputs = {\"messages\": [(\"user\", \"What's it known for?\")]}\n",
    "result = graph.invoke(inputs, config=config)"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "fdb186da-b55d-4cb8-a237-e9e157ab0458",
   "metadata": {},
   "source": [
    "Let's see how many tokens we have in the message history so far:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 8,
   "id": "41ba0253-5199-4d29-82ae-258cbbebddb4",
   "metadata": {},
   "outputs": [
    {
     "data": {
      "text/plain": [
       "415"
      ]
     },
     "execution_count": 8,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "messages = result[\"messages\"]\n",
    "count_tokens_approximately(messages)"
   ]
  },
  {
   "attachments": {},
   "cell_type": "markdown",
   "id": "812987ac-66ba-4122-8281-469cbdced7c7",
   "metadata": {},
   "source": [
    "You can see that we are close to the `max_tokens` threshold, so on the next invocation we should see `pre_model_hook` kick-in and trim the message history. Let's run it again:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 9,
   "id": "26c53429-90ba-4d0b-abb9-423d9120ad26",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Update from node: pre_model_hook\n",
      "================================\u001b[1m Human Message \u001b[0m=================================\n",
      "\n",
      "What's it known for?\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "New York City is known for a variety of iconic landmarks, cultural institutions, and vibrant neighborhoods. Some of the most notable features include:\n",
      "\n",
      "1. **Statue of Liberty**: A symbol of freedom and democracy, located on Liberty Island.\n",
      "2. **Times Square**: Known for its bright lights, Broadway theaters, and bustling atmosphere.\n",
      "3. **Central Park**: A large public park offering a natural retreat in the middle of the city.\n",
      "4. **Empire State Building**: An iconic skyscraper offering panoramic views of the city.\n",
      "5. **Broadway**: Famous for its world-class theater productions.\n",
      "6. **Wall Street**: The financial hub of the United States.\n",
      "7. **Museums**: Including the Metropolitan Museum of Art, Museum of Modern Art (MoMA), and the American Museum of Natural History.\n",
      "8. **Diverse Cuisine**: A melting pot of cultures offering a wide range of culinary experiences.\n",
      "9. **Cultural Diversity**: A rich tapestry of cultures and communities from around the world.\n",
      "10. **Fashion**: A global fashion capital, hosting events like New York Fashion Week.\n",
      "\n",
      "These are just a few highlights of what makes New York City a unique and vibrant place.\n",
      "================================\u001b[1m Human Message \u001b[0m=================================\n",
      "\n",
      "where can i find the best bagel?\n",
      "\n",
      "\n",
      "\n",
      "Update from node: agent\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "New York City is famous for its bagels, and there are several places renowned for serving some of the best. Here are a few top spots where you can find excellent bagels in NYC:\n",
      "\n",
      "1. **Ess-a-Bagel**: Known for their large, chewy bagels with a variety of spreads and toppings.\n",
      "2. **Russ & Daughters**: A classic spot offering traditional bagels with high-quality smoked fish and cream cheese.\n",
      "3. **H&H Bagels**: Famous for their fresh, hand-rolled bagels.\n",
      "4. **Murray’s Bagels**: Offers a wide selection of bagels and spreads, with a no-toasting policy to preserve freshness.\n",
      "5. **Absolute Bagels**: Known for their authentic, fluffy bagels and a variety of cream cheese options.\n",
      "6. **Tompkins Square Bagels**: Offers creative bagel sandwiches and a wide range of spreads.\n",
      "7. **Bagel Hole**: Known for their smaller, denser bagels with a crispy crust.\n",
      "\n",
      "Each of these places has its own unique style and flavor, so it might be worth trying a few to find your personal favorite!\n",
      "\n",
      "\n",
      "\n"
     ]
    }
   ],
   "source": [
    "inputs = {\"messages\": [(\"user\", \"where can i find the best bagel?\")]}\n",
    "print_stream(graph.stream(inputs, config=config, stream_mode=\"updates\"))"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "58fe0399-4e7d-4482-a4cb-5301311932d0",
   "metadata": {},
   "source": [
    "You can see that the `pre_model_hook` node now only returned the last 3 messages, as expected. However, the existing message history is untouched:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 10,
   "id": "7ecfc310-8f9e-4aa0-9e58-17e71551639a",
   "metadata": {},
   "outputs": [],
   "source": [
    "updated_messages = graph.get_state(config).values[\"messages\"]\n",
    "assert [(m.type, m.content) for m in updated_messages[: len(messages)]] == [\n",
    "    (m.type, m.content) for m in messages\n",
    "]"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "035864e3-0083-4dea-bf85-3a702fa5303f",
   "metadata": {},
   "source": [
    "## Overwrite the original message history"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "0b0a4fd5-a2ba-4eca-91a9-d294f4f2d884",
   "metadata": {},
   "source": [
    "Let's now change the `pre_model_hook` to **overwrite** the message history in the graph state. To do this, we’ll return the updated messages under `messages` key. We’ll also include a special `RemoveMessage(REMOVE_ALL_MESSAGES)` object, which tells `create_react_agent` to remove previous messages from the graph state:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 11,
   "id": "48c2a65b-685a-4750-baa6-2d61efe76b5f",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langchain_core.messages import RemoveMessage\n",
    "from langgraph.graph.message import REMOVE_ALL_MESSAGES\n",
    "\n",
    "\n",
    "def pre_model_hook(state):\n",
    "    trimmed_messages = trim_messages(\n",
    "        state[\"messages\"],\n",
    "        strategy=\"last\",\n",
    "        token_counter=count_tokens_approximately,\n",
    "        max_tokens=384,\n",
    "        start_on=\"human\",\n",
    "        end_on=(\"human\", \"tool\"),\n",
    "    )\n",
    "    # NOTE that we're now returning the messages under the `messages` key\n",
    "    # We also remove the existing messages in the history to ensure we're overwriting the history\n",
    "    # highlight-next-line\n",
    "    return {\"messages\": [RemoveMessage(REMOVE_ALL_MESSAGES)] + trimmed_messages}\n",
    "\n",
    "\n",
    "checkpointer = InMemorySaver()\n",
    "graph = create_react_agent(\n",
    "    model,\n",
    "    tools,\n",
    "    # highlight-next-line\n",
    "    pre_model_hook=pre_model_hook,\n",
    "    checkpointer=checkpointer,\n",
    ")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "cd061682-231c-4487-9c2f-a6820dfbcab7",
   "metadata": {},
   "source": [
    "Now let's run the agent with the same queries as before:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 12,
   "id": "831be36a-78a1-4885-9a03-8d085dfd7e37",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Update from node: pre_model_hook\n",
      "================================\u001b[1m Remove Message \u001b[0m================================\n",
      "\n",
      "\n",
      "================================\u001b[1m Human Message \u001b[0m=================================\n",
      "\n",
      "What's it known for?\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "New York City is known for a variety of iconic landmarks, cultural institutions, and vibrant neighborhoods. Some of the most notable features include:\n",
      "\n",
      "1. **Statue of Liberty**: A symbol of freedom and democracy, located on Liberty Island.\n",
      "2. **Times Square**: Known for its bright lights, Broadway theaters, and bustling atmosphere.\n",
      "3. **Central Park**: A large public park offering a natural oasis amidst the urban environment.\n",
      "4. **Empire State Building**: An iconic skyscraper offering panoramic views of the city.\n",
      "5. **Broadway**: Famous for its world-class theater productions and musicals.\n",
      "6. **Wall Street**: The financial hub of the United States, located in the Financial District.\n",
      "7. **Museums**: Including the Metropolitan Museum of Art, Museum of Modern Art (MoMA), and the American Museum of Natural History.\n",
      "8. **Diverse Cuisine**: A melting pot of cultures, offering a wide range of international foods.\n",
      "9. **Cultural Diversity**: Known for its diverse population and vibrant cultural scene.\n",
      "10. **Brooklyn Bridge**: An iconic suspension bridge connecting Manhattan and Brooklyn.\n",
      "\n",
      "These are just a few highlights, as NYC is a city with endless attractions and activities.\n",
      "================================\u001b[1m Human Message \u001b[0m=================================\n",
      "\n",
      "where can i find the best bagel?\n",
      "\n",
      "\n",
      "\n",
      "Update from node: agent\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "New York City is famous for its bagels, and there are several places renowned for serving some of the best. Here are a few top spots where you can find delicious bagels in NYC:\n",
      "\n",
      "1. **Ess-a-Bagel**: Known for its large, chewy bagels and a wide variety of spreads and toppings. Locations in Midtown and the East Village.\n",
      "\n",
      "2. **Russ & Daughters**: A historic appetizing store on the Lower East Side, famous for its bagels with lox and cream cheese.\n",
      "\n",
      "3. **Absolute Bagels**: Located on the Upper West Side, this spot is popular for its fresh, fluffy bagels.\n",
      "\n",
      "4. **Murray’s Bagels**: Known for its traditional, hand-rolled bagels. Located in Greenwich Village.\n",
      "\n",
      "5. **Tompkins Square Bagels**: Offers a wide selection of bagels and creative cream cheese flavors. Located in the East Village.\n",
      "\n",
      "6. **Bagel Hole**: A small shop in Park Slope, Brooklyn, known for its classic, no-frills bagels.\n",
      "\n",
      "7. **Leo’s Bagels**: Located in the Financial District, known for its authentic New York-style bagels.\n",
      "\n",
      "Each of these places has its own unique style and flavor, so it might be worth trying a few to find your personal favorite!\n",
      "\n",
      "\n",
      "\n"
     ]
    }
   ],
   "source": [
    "config = {\"configurable\": {\"thread_id\": \"1\"}}\n",
    "\n",
    "inputs = {\"messages\": [(\"user\", \"What's the weather in NYC?\")]}\n",
    "result = graph.invoke(inputs, config=config)\n",
    "\n",
    "inputs = {\"messages\": [(\"user\", \"What's it known for?\")]}\n",
    "result = graph.invoke(inputs, config=config)\n",
    "messages = result[\"messages\"]\n",
    "\n",
    "inputs = {\"messages\": [(\"user\", \"where can i find the best bagel?\")]}\n",
    "print_stream(\n",
    "    graph.stream(inputs, config=config, stream_mode=\"updates\"),\n",
    "    output_messages_key=\"messages\",\n",
    ")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "cc9a0604-3d2b-48ff-9eaf-d16ea351fb30",
   "metadata": {},
   "source": [
    "You can see that the `pre_model_hook` node returned the last 3 messages again. However, this time, the message history is modified in the graph state as well:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 13,
   "id": "394f72f8-f817-472d-a193-e01509a86132",
   "metadata": {},
   "outputs": [],
   "source": [
    "updated_messages = graph.get_state(config).values[\"messages\"]\n",
    "assert (\n",
    "    # First 2 messages in the new history are the same as last 2 messages in the old\n",
    "    [(m.type, m.content) for m in updated_messages[:2]]\n",
    "    == [(m.type, m.content) for m in messages[-2:]]\n",
    ")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "ee186d6d-4d07-404f-b236-f662db62339d",
   "metadata": {},
   "source": [
    "## Summarizing message history"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "a6e53e0f-9a1e-4188-8435-c23ad8148b4f",
   "metadata": {},
   "source": [
    "Finally, let's apply a different strategy for managing message history — summarization. Just as with trimming, you can choose to keep original message history unmodified or overwrite it. The example below will only show the former.\n",
    "\n",
    "We will use the [`SummarizationNode`](https://langchain-ai.github.io/langmem/guides/summarization/#using-summarizationnode) from the prebuilt `langmem` library. Once the message history reaches the token limit, the summarization node will summarize earlier messages to make sure they fit into `max_tokens`."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 14,
   "id": "b9540c1c-2eba-42da-ba4e-478521161a1f",
   "metadata": {},
   "outputs": [],
   "source": [
    "# highlight-next-line\n",
    "from langmem.short_term import SummarizationNode\n",
    "from langgraph.prebuilt.chat_agent_executor import AgentState\n",
    "from typing import Any\n",
    "\n",
    "model = ChatOpenAI(model=\"gpt-4o\")\n",
    "summarization_model = model.bind(max_tokens=128)\n",
    "\n",
    "summarization_node = SummarizationNode(\n",
    "    token_counter=count_tokens_approximately,\n",
    "    model=summarization_model,\n",
    "    max_tokens=384,\n",
    "    max_summary_tokens=128,\n",
    "    output_messages_key=\"llm_input_messages\",\n",
    ")\n",
    "\n",
    "\n",
    "class State(AgentState):\n",
    "    # NOTE: we're adding this key to keep track of previous summary information\n",
    "    # to make sure we're not summarizing on every LLM call\n",
    "    # highlight-next-line\n",
    "    context: dict[str, Any]\n",
    "\n",
    "\n",
    "checkpointer = InMemorySaver()\n",
    "graph = create_react_agent(\n",
    "    # limit the output size to ensure consistent behavior\n",
    "    model.bind(max_tokens=256),\n",
    "    tools,\n",
    "    # highlight-next-line\n",
    "    pre_model_hook=summarization_node,\n",
    "    # highlight-next-line\n",
    "    state_schema=State,\n",
    "    checkpointer=checkpointer,\n",
    ")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 15,
   "id": "8eccaaca-5d9c-4faf-b997-d4b8e84b59ac",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Update from node: pre_model_hook\n",
      "================================\u001b[1m System Message \u001b[0m================================\n",
      "\n",
      "Summary of the conversation so far: The user asked about the current weather in New York City. In response, the assistant provided information that it might be cloudy, with a chance of rain, and temperatures reaching up to 80 degrees.\n",
      "================================\u001b[1m Human Message \u001b[0m=================================\n",
      "\n",
      "What's it known for?\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "New York City, often referred to as NYC, is known for its:\n",
      "\n",
      "1. **Landmarks and Iconic Sites**:\n",
      "   - **Statue of Liberty**: A symbol of freedom and democracy.\n",
      "   - **Central Park**: A vast green oasis in the middle of the city.\n",
      "   - **Empire State Building**: Once the tallest building in the world, offering stunning views of the city.\n",
      "   - **Times Square**: Known for its bright lights and bustling atmosphere.\n",
      "\n",
      "2. **Cultural Institutions**:\n",
      "   - **Broadway**: Renowned for theatrical performances and musicals.\n",
      "   - **Metropolitan Museum of Art** and **Museum of Modern Art (MoMA)**: World-class art collections.\n",
      "   - **American Museum of Natural History**: Known for its extensive exhibits ranging from dinosaurs to space exploration.\n",
      "   \n",
      "3. **Diverse Neighborhoods and Cuisine**:\n",
      "   - NYC is famous for having a melting pot of cultures, reflected in neighborhoods like Chinatown, Little Italy, and Harlem.\n",
      "   - The city offers a wide range of international cuisines, from street food to high-end dining.\n",
      "\n",
      "4. **Financial District**:\n",
      "   - Home to Wall Street, the New York Stock Exchange (NYSE), and other major financial institutions.\n",
      "\n",
      "5. **Media and Entertainment**:\n",
      "   - Major hub for television, film, and media, with numerous studios and networks based there.\n",
      "\n",
      "6. **Fashion**:\n",
      "   - Often referred to as one of the \"Big Four\" fashion capitals, hosting events like New York Fashion Week.\n",
      "\n",
      "7. **Sports**:\n",
      "   - Known for its passionate sports culture with teams like the Yankees (MLB), Mets (MLB), Knicks (NBA), and Rangers (NHL).\n",
      "\n",
      "These elements, among others, contribute to NYC's reputation as a vibrant and dynamic city.\n",
      "================================\u001b[1m Human Message \u001b[0m=================================\n",
      "\n",
      "where can i find the best bagel?\n",
      "\n",
      "\n",
      "\n",
      "Update from node: agent\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "Finding the best bagel in New York City can be subjective, as there are many beloved spots across the city. However, here are some renowned bagel shops you might want to try:\n",
      "\n",
      "1. **Ess-a-Bagel**: Known for its chewy and flavorful bagels, located in Midtown and Stuyvesant Town.\n",
      "\n",
      "2. **Bagel Hole**: A favorite for traditionalists, offering classic and dense bagels, located in Park Slope, Brooklyn.\n",
      "\n",
      "3. **Russ & Daughters**: A legendary appetizing store on the Lower East Side, famous for their bagels with lox.\n",
      "\n",
      "4. **Murray’s Bagels**: Located in Greenwich Village, known for their fresh and authentic New York bagels.\n",
      "\n",
      "5. **Absolute Bagels**: Located on the Upper West Side, they’re known for their fresh, fluffy bagels with a variety of spreads.\n",
      "\n",
      "6. **Tompkins Square Bagels**: In the East Village, famous for their creative cream cheese options and fresh bagels.\n",
      "\n",
      "7. **Zabar’s**: A landmark on the Upper West Side known for their classic bagels and smoked fish.\n",
      "\n",
      "Each of these spots offers a unique take on the classic New York bagel experience, and trying several might be the best way to discover your personal favorite!\n",
      "\n",
      "\n",
      "\n"
     ]
    }
   ],
   "source": [
    "config = {\"configurable\": {\"thread_id\": \"1\"}}\n",
    "inputs = {\"messages\": [(\"user\", \"What's the weather in NYC?\")]}\n",
    "\n",
    "result = graph.invoke(inputs, config=config)\n",
    "\n",
    "inputs = {\"messages\": [(\"user\", \"What's it known for?\")]}\n",
    "result = graph.invoke(inputs, config=config)\n",
    "\n",
    "inputs = {\"messages\": [(\"user\", \"where can i find the best bagel?\")]}\n",
    "print_stream(graph.stream(inputs, config=config, stream_mode=\"updates\"))"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "7caaf2f7-281a-4421-bf98-c745d950c56f",
   "metadata": {},
   "source": [
    "You can see that the earlier messages have now been replaced with the summary of the earlier conversation!"
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
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/cross-thread-persistence-functional.ipynb
```ipynb
{
 "cells": [
  {
   "attachments": {},
   "cell_type": "markdown",
   "id": "d2eecb96-cf0e-47ed-8116-88a7eaa4236d",
   "metadata": {},
   "source": [
    "# How to add cross-thread persistence (functional API)\n",
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
    "LangGraph allows you to persist data across **different [threads](../../concepts/persistence/#threads)**. For instance, you can store information about users (their names or preferences) in a shared (cross-thread) memory and reuse them in the new threads (e.g., new conversations).\n",
    "\n",
    "When using the [functional API](../../concepts/functional_api/), you can set it up to store and retrieve memories by using the [Store](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.BaseStore) interface:\n",
    "\n",
    "1. Create an instance of a `Store`\n",
    "\n",
    "    ```python\n",
    "    from langgraph.store.memory import InMemoryStore, BaseStore\n",
    "    \n",
    "    store = InMemoryStore()\n",
    "    ```\n",
    "\n",
    "2. Pass the `store` instance to the `entrypoint()` decorator and expose `store` parameter in the function signature:\n",
    "\n",
    "    ```python\n",
    "    from langgraph.func import entrypoint\n",
    "\n",
    "    @entrypoint(store=store)\n",
    "    def workflow(inputs: dict, store: BaseStore):\n",
    "        my_task(inputs).result()\n",
    "        ...\n",
    "    ```\n",
    "    \n",
    "In this guide, we will show how to construct and use a workflow that has a shared memory implemented using the [Store](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.BaseStore) interface.\n",
    "\n",
    "!!! note Note\n",
    "\n",
    "    Support for the [`Store`](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.BaseStore) API that is used in this guide was added in LangGraph `v0.2.32`.\n",
    "\n",
    "    Support for __index__ and __query__ arguments of the [`Store`](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.BaseStore) API that is used in this guide was added in LangGraph `v0.2.54`.\n",
    "\n",
    "!!! tip \"Note\"\n",
    "\n",
    "    If you need to add cross-thread persistence to a `StateGraph`, check out this [how-to guide](../cross-thread-persistence).\n",
    "\n",
    "## Setup\n",
    "\n",
    "First, let's install the required packages and set our API keys"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 1,
   "id": "3457aadf",
   "metadata": {},
   "outputs": [],
   "source": [
    "%%capture --no-stderr\n",
    "%pip install -U langchain_anthropic langchain_openai langgraph"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "aa2c64a7",
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
    "_set_env(\"ANTHROPIC_API_KEY\")\n",
    "_set_env(\"OPENAI_API_KEY\")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "51b6817d",
   "metadata": {},
   "source": [
    "!!! tip \"Set up [LangSmith](https://smith.langchain.com) for LangGraph development\"\n",
    "\n",
    "    Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. LangSmith lets you use trace data to debug, test, and monitor your LLM apps built with LangGraph — read more about how to get started [here](https://docs.smith.langchain.com)"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "6b5b3d42-3d2c-455e-ac10-e2ae74dc1cf1",
   "metadata": {},
   "source": [
    "## Example: simple chatbot with long-term memory"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "c4c550b5-1954-496b-8b9d-800361af17dc",
   "metadata": {},
   "source": [
    "### Define store\n",
    "\n",
    "In this example we will create a workflow that will be able to retrieve information about a user's preferences. We will do so by defining an `InMemoryStore` - an object that can store data in memory and query that data.\n",
    "\n",
    "When storing objects using the `Store` interface you define two things:\n",
    "\n",
    "* the namespace for the object, a tuple (similar to directories)\n",
    "* the object key (similar to filenames)\n",
    "\n",
    "In our example, we'll be using `(\"memories\", <user_id>)` as namespace and random UUID as key for each new memory.\n",
    "\n",
    "Importantly, to determine the user, we will be passing `user_id` via the config keyword argument of the node function.\n",
    "\n",
    "Let's first define our store!"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 3,
   "id": "a7f303d6-612e-4e34-bf36-29d4ed25d802",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langgraph.store.memory import InMemoryStore\n",
    "from langchain_openai import OpenAIEmbeddings\n",
    "\n",
    "in_memory_store = InMemoryStore(\n",
    "    index={\n",
    "        \"embed\": OpenAIEmbeddings(model=\"text-embedding-3-small\"),\n",
    "        \"dims\": 1536,\n",
    "    }\n",
    ")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "3389c9f4-226d-40c7-8bfc-ee8aac24f79d",
   "metadata": {},
   "source": [
    "### Create workflow"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 4,
   "id": "2a30a362-528c-45ee-9df6-630d2d843588",
   "metadata": {},
   "outputs": [],
   "source": [
    "import uuid\n",
    "\n",
    "from langchain_anthropic import ChatAnthropic\n",
    "from langchain_core.runnables import RunnableConfig\n",
    "from langchain_core.messages import BaseMessage\n",
    "from langgraph.func import entrypoint, task\n",
    "from langgraph.graph import add_messages\n",
    "from langgraph.checkpoint.memory import InMemorySaver\n",
    "from langgraph.store.base import BaseStore\n",
    "\n",
    "\n",
    "model = ChatAnthropic(model=\"claude-3-5-sonnet-latest\")\n",
    "\n",
    "\n",
    "@task\n",
    "def call_model(messages: list[BaseMessage], memory_store: BaseStore, user_id: str):\n",
    "    namespace = (\"memories\", user_id)\n",
    "    last_message = messages[-1]\n",
    "    memories = memory_store.search(namespace, query=str(last_message.content))\n",
    "    info = \"\\n\".join([d.value[\"data\"] for d in memories])\n",
    "    system_msg = f\"You are a helpful assistant talking to the user. User info: {info}\"\n",
    "\n",
    "    # Store new memories if the user asks the model to remember\n",
    "    if \"remember\" in last_message.content.lower():\n",
    "        memory = \"User name is Bob\"\n",
    "        memory_store.put(namespace, str(uuid.uuid4()), {\"data\": memory})\n",
    "\n",
    "    response = model.invoke([{\"role\": \"system\", \"content\": system_msg}] + messages)\n",
    "    return response\n",
    "\n",
    "\n",
    "# NOTE: we're passing the store object here when creating a workflow via entrypoint()\n",
    "@entrypoint(checkpointer=InMemorySaver(), store=in_memory_store)\n",
    "def workflow(\n",
    "    inputs: list[BaseMessage],\n",
    "    *,\n",
    "    previous: list[BaseMessage],\n",
    "    config: RunnableConfig,\n",
    "    store: BaseStore,\n",
    "):\n",
    "    user_id = config[\"configurable\"][\"user_id\"]\n",
    "    previous = previous or []\n",
    "    inputs = add_messages(previous, inputs)\n",
    "    response = call_model(inputs, store, user_id).result()\n",
    "    return entrypoint.final(value=response, save=add_messages(inputs, response))"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "f22a4a18-67e4-4f0b-b655-a29bbe202e1c",
   "metadata": {},
   "source": [
    "!!! note Note\n",
    "\n",
    "    If you're using LangGraph Cloud or LangGraph Studio, you __don't need__ to pass store to the entrypoint decorator, since it's done automatically."
   ]
  },
  {
   "cell_type": "markdown",
   "id": "552d4e33-556d-4fa5-8094-2a076bc21529",
   "metadata": {},
   "source": [
    "### Run the workflow!"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "1842c626-6cd9-4f58-b549-58978e478098",
   "metadata": {},
   "source": [
    "Now let's specify a user ID in the config and tell the model our name:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 5,
   "id": "c871a073-a466-46ad-aafe-2b870831057e",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "Hello Bob! Nice to meet you. I'll remember that your name is Bob. How can I help you today?\n"
     ]
    }
   ],
   "source": [
    "config = {\"configurable\": {\"thread_id\": \"1\", \"user_id\": \"1\"}}\n",
    "input_message = {\"role\": \"user\", \"content\": \"Hi! Remember: my name is Bob\"}\n",
    "for chunk in workflow.stream([input_message], config, stream_mode=\"values\"):\n",
    "    chunk.pretty_print()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 6,
   "id": "d862be40-1f8a-4057-81c4-b7bf073dc4c1",
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
    "config = {\"configurable\": {\"thread_id\": \"2\", \"user_id\": \"1\"}}\n",
    "input_message = {\"role\": \"user\", \"content\": \"what is my name?\"}\n",
    "for chunk in workflow.stream([input_message], config, stream_mode=\"values\"):\n",
    "    chunk.pretty_print()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "80fd01ec-f135-4811-8743-daff8daea422",
   "metadata": {},
   "source": [
    "We can now inspect our in-memory store and verify that we have in fact saved the memories for the user:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 7,
   "id": "76cde493-89cf-4709-a339-207d2b7e9ea7",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "{'data': 'User name is Bob'}\n"
     ]
    }
   ],
   "source": [
    "for memory in in_memory_store.search((\"memories\", \"1\")):\n",
    "    print(memory.value)"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "23f5d7eb-af23-4131-b8fd-2a69e74e6e55",
   "metadata": {},
   "source": [
    "Let's now run the workflow for another user to verify that the memories about the first user are self contained:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 8,
   "id": "d362350b-d730-48bd-9652-983812fd7811",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "I don't have any information about your name. I can only see our current conversation without any prior context or personal details about you. If you'd like me to know your name, feel free to tell me!\n"
     ]
    }
   ],
   "source": [
    "config = {\"configurable\": {\"thread_id\": \"3\", \"user_id\": \"2\"}}\n",
    "input_message = {\"role\": \"user\", \"content\": \"what is my name?\"}\n",
    "for chunk in workflow.stream([input_message], config, stream_mode=\"values\"):\n",
    "    chunk.pretty_print()"
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
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/disable-streaming.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# How to disable streaming for models that don't support it\n",
    "\n",
    "<div class=\"admonition tip\">\n",
    "    <p class=\"admonition-title\">Prerequisites</p>\n",
    "    <p>\n",
    "        This guide assumes familiarity with the following:\n",
    "        <ul>\n",
    "            <li>\n",
    "                <a href=\"https://python.langchain.com/docs/concepts/#streaming\">\n",
    "                    streaming\n",
    "                </a>                \n",
    "            </li>\n",
    "            <li>\n",
    "                <a href=\"https://python.langchain.com/docs/concepts/#chat-models/\">\n",
    "                    Chat Models\n",
    "                </a>\n",
    "            </li>\n",
    "        </ul>\n",
    "    </p>\n",
    "</div> \n",
    "\n",
    "Some chat models, including the new O1 models from OpenAI (depending on when you're reading this), do not support streaming. This can lead to issues when using the [astream_events API](https://python.langchain.com/docs/concepts/#astream_events), as it calls models in streaming mode, expecting streaming to function properly.\n",
    "\n",
    "In this guide, we’ll show you how to disable streaming for models that don’t support it, ensuring they they're never called in streaming mode, even when invoked through the astream_events API."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 4,
   "metadata": {},
   "outputs": [],
   "source": [
    "from langchain_openai import ChatOpenAI\n",
    "from langgraph.graph import MessagesState\n",
    "from langgraph.graph import StateGraph, START, END\n",
    "\n",
    "llm = ChatOpenAI(model=\"o1-preview\", temperature=1)\n",
    "\n",
    "graph_builder = StateGraph(MessagesState)\n",
    "\n",
    "\n",
    "def chatbot(state: MessagesState):\n",
    "    return {\"messages\": [llm.invoke(state[\"messages\"])]}\n",
    "\n",
    "\n",
    "graph_builder.add_node(\"chatbot\", chatbot)\n",
    "graph_builder.add_edge(START, \"chatbot\")\n",
    "graph_builder.add_edge(\"chatbot\", END)\n",
    "graph = graph_builder.compile()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 5,
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
    "from IPython.display import Image, display\n",
    "\n",
    "display(Image(graph.get_graph().draw_mermaid_png()))"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## Without disabling streaming\n",
    "\n",
    "Now that we've defined our graph, let's try to call `astream_events` without disabling streaming. This should throw an error because the `o1` model does not support streaming natively:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 6,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Streaming not supported!\n"
     ]
    }
   ],
   "source": [
    "input = {\"messages\": {\"role\": \"user\", \"content\": \"how many r's are in strawberry?\"}}\n",
    "try:\n",
    "    async for event in graph.astream_events(input, version=\"v2\"):\n",
    "        if event[\"event\"] == \"on_chat_model_end\":\n",
    "            print(event[\"data\"][\"output\"].content, end=\"\", flush=True)\n",
    "except:\n",
    "    print(\"Streaming not supported!\")"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "An error occurred as we expected, luckily there is an easy fix!\n",
    "\n",
    "## Disabling streaming\n",
    "\n",
    "Now without making any changes to our graph, let's set the [disable_streaming](https://python.langchain.com/api_reference/core/language_models/langchain_core.language_models.chat_models.BaseChatModel.html#langchain_core.language_models.chat_models.BaseChatModel.disable_streaming) parameter on our model to be `True` which will solve the problem:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 7,
   "metadata": {},
   "outputs": [],
   "source": [
    "llm = ChatOpenAI(model=\"o1-preview\", temperature=1, disable_streaming=True)\n",
    "\n",
    "graph_builder = StateGraph(MessagesState)\n",
    "\n",
    "\n",
    "def chatbot(state: MessagesState):\n",
    "    return {\"messages\": [llm.invoke(state[\"messages\"])]}\n",
    "\n",
    "\n",
    "graph_builder.add_node(\"chatbot\", chatbot)\n",
    "graph_builder.add_edge(START, \"chatbot\")\n",
    "graph_builder.add_edge(\"chatbot\", END)\n",
    "graph = graph_builder.compile()"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "And now, rerunning with the same input, we should see no errors:"
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
      "There are three \"r\"s in the word \"strawberry\"."
     ]
    }
   ],
   "source": [
    "input = {\"messages\": {\"role\": \"user\", \"content\": \"how many r's are in strawberry?\"}}\n",
    "async for event in graph.astream_events(input, version=\"v2\"):\n",
    "    if event[\"event\"] == \"on_chat_model_end\":\n",
    "        print(event[\"data\"][\"output\"].content, end=\"\", flush=True)"
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
 "nbformat_minor": 4
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/enable-tracing.md
```md
# Enable tracing for your application

To enable [tracing](../concepts/tracing.md) for your application, set the following environment variables:

```python
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=<your-api-key>
```

For more information, see [Trace with LangGraph](https://docs.smith.langchain.com/observability/how_to_guides/trace_with_langgraph).

## Learn more

- [Graph runs in LangSmith](../how-tos/run-id-langsmith.md)
- [LangSmith Observability quickstart](https://docs.smith.langchain.com/observability)
- [Tracing conceptual guide](https://docs.smith.langchain.com/observability/concepts#traces)
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/graph-api.md
```md
# How to use the graph API

This guide demonstrates the basics of LangGraph's Graph API. It walks through [state](#define-and-update-state), as well as composing common graph structures such as [sequences](#create-a-sequence-of-steps), [branches](#create-branches), and [loops](#create-and-control-loops). It also covers LangGraph's control features, including the [Send API](#map-reduce-and-the-send-api) for map-reduce workflows and the [Command API](#combine-control-flow-and-state-updates-with-command) for combining state updates with "hops" across nodes.

## Setup

:::python
Install `langgraph`:

```bash
pip install -U langgraph
```
:::

:::js
Install `langgraph`:

```bash
npm install @langchain/langgraph
```
:::

!!! tip "Set up LangSmith for better debugging"

    Sign up for [LangSmith](https://smith.langchain.com) to quickly spot issues and improve the performance of your LangGraph projects. LangSmith lets you use trace data to debug, test, and monitor your LLM apps built with LangGraph — read more about how to get started in the [docs](https://docs.smith.langchain.com).

## Define and update state

Here we show how to define and update [state](../concepts/low_level.md#state) in LangGraph. We will demonstrate:

1. How to use state to define a graph's [schema](../concepts/low_level.md#schema)
2. How to use [reducers](../concepts/low_level.md#reducers) to control how state updates are processed.

### Define state

:::python
[State](../concepts/low_level.md#state) in LangGraph can be a `TypedDict`, `Pydantic` model, or dataclass. Below we will use `TypedDict`. See [this section](#use-pydantic-models-for-graph-state) for detail on using Pydantic.
:::

:::js
[State](../concepts/low_level.md#state) in LangGraph can be defined using Zod schemas. Below we will use Zod. See [this section](#alternative-state-definitions) for detail on using alternative approaches.
:::

By default, graphs will have the same input and output schema, and the state determines that schema. See [this section](#define-input-and-output-schemas) for how to define distinct input and output schemas.

Let's consider a simple example using [messages](../concepts/low_level.md#working-with-messages-in-graph-state). This represents a versatile formulation of state for many LLM applications. See our [concepts page](../concepts/low_level.md#working-with-messages-in-graph-state) for more detail.

:::python
```python
from langchain_core.messages import AnyMessage
from typing_extensions import TypedDict

class State(TypedDict):
    messages: list[AnyMessage]
    extra_field: int
```

This state tracks a list of [message](https://python.langchain.com/docs/concepts/messages/) objects, as well as an extra integer field.
:::

:::js
```typescript
import { BaseMessage } from "@langchain/core/messages";
import { z } from "zod";

const State = z.object({
  messages: z.array(z.custom<BaseMessage>()),
  extraField: z.number(),
});
```

This state tracks a list of [message](https://js.langchain.com/docs/concepts/messages/) objects, as well as an extra integer field.
:::

### Update state

:::python
Let's build an example graph with a single node. Our [node](../concepts/low_level.md#nodes) is just a Python function that reads our graph's state and makes updates to it. The first argument to this function will always be the state:

```python
from langchain_core.messages import AIMessage

def node(state: State):
    messages = state["messages"]
    new_message = AIMessage("Hello!")
    return {"messages": messages + [new_message], "extra_field": 10}
```

This node simply appends a message to our message list, and populates an extra field.
:::

:::js
Let's build an example graph with a single node. Our [node](../concepts/low_level.md#nodes) is just a TypeScript function that reads our graph's state and makes updates to it. The first argument to this function will always be the state:

```typescript
import { AIMessage } from "@langchain/core/messages";

const node = (state: z.infer<typeof State>) => {
  const messages = state.messages;
  const newMessage = new AIMessage("Hello!");
  return { messages: messages.concat([newMessage]), extraField: 10 };
};
```

This node simply appends a message to our message list, and populates an extra field.
:::

!!! important

    Nodes should return updates to the state directly, instead of mutating the state.

:::python
Let's next define a simple graph containing this node. We use [StateGraph](../concepts/low_level.md#stategraph) to define a graph that operates on this state. We then use [add_node](../concepts/low_level.md#nodes) populate our graph.

```python
from langgraph.graph import StateGraph

builder = StateGraph(State)
builder.add_node(node)
builder.set_entry_point("node")
graph = builder.compile()
```
:::

:::js
Let's next define a simple graph containing this node. We use [StateGraph](../concepts/low_level.md#stategraph) to define a graph that operates on this state. We then use [addNode](../concepts/low_level.md#nodes) populate our graph.

```typescript
import { StateGraph } from "@langchain/langgraph";

const graph = new StateGraph(State)
  .addNode("node", node)
  .addEdge("__start__", "node")
  .compile();
```
:::

LangGraph provides built-in utilities for visualizing your graph. Let's inspect our graph. See [this section](#visualize-your-graph) for detail on visualization.

:::python
```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![Simple graph with single node](assets/graph_api_image_1.png)
:::

:::js
```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("graph.png", imageBuffer);
```
:::

In this case, our graph just executes a single node. Let's proceed with a simple invocation:

:::python
```python
from langchain_core.messages import HumanMessage

result = graph.invoke({"messages": [HumanMessage("Hi")]})
result
```

```
{'messages': [HumanMessage(content='Hi'), AIMessage(content='Hello!')], 'extra_field': 10}
```
:::

:::js
```typescript
import { HumanMessage } from "@langchain/core/messages";

const result = await graph.invoke({ messages: [new HumanMessage("Hi")], extraField: 0 });
console.log(result);
```

```
{ messages: [HumanMessage { content: 'Hi' }, AIMessage { content: 'Hello!' }], extraField: 10 }
```
:::

Note that:

- We kicked off invocation by updating a single key of the state.
- We receive the entire state in the invocation result.

:::python
For convenience, we frequently inspect the content of [message objects](https://python.langchain.com/docs/concepts/messages/) via pretty-print:

```python
for message in result["messages"]:
    message.pretty_print()
```

```
================================ Human Message ================================

Hi
================================== Ai Message ==================================

Hello!
```
:::

:::js
For convenience, we frequently inspect the content of [message objects](https://js.langchain.com/docs/concepts/messages/) via logging:

```typescript
for (const message of result.messages) {
  console.log(`${message.getType()}: ${message.content}`);
}
```

```
human: Hi
ai: Hello!
```
:::

### Process state updates with reducers

Each key in the state can have its own independent [reducer](../concepts/low_level.md#reducers) function, which controls how updates from nodes are applied. If no reducer function is explicitly specified then it is assumed that all updates to the key should override it.

:::python
For `TypedDict` state schemas, we can define reducers by annotating the corresponding field of the state with a reducer function.

In the earlier example, our node updated the `"messages"` key in the state by appending a message to it. Below, we add a reducer to this key, such that updates are automatically appended:

```python
from typing_extensions import Annotated

def add(left, right):
    """Can also import `add` from the `operator` built-in."""
    return left + right

class State(TypedDict):
    # highlight-next-line
    messages: Annotated[list[AnyMessage], add]
    extra_field: int
```

Now our node can be simplified:

```python
def node(state: State):
    new_message = AIMessage("Hello!")
    # highlight-next-line
    return {"messages": [new_message], "extra_field": 10}
```
:::

:::js
For Zod state schemas, we can define reducers by using the special `.langgraph.reducer()` method on the schema field.

In the earlier example, our node updated the `"messages"` key in the state by appending a message to it. Below, we add a reducer to this key, such that updates are automatically appended:

```typescript
import "@langchain/langgraph/zod";

const State = z.object({
  // highlight-next-line
  messages: z.array(z.custom<BaseMessage>()).langgraph.reducer((x, y) => x.concat(y)),
  extraField: z.number(),
});
```

Now our node can be simplified:

```typescript
const node = (state: z.infer<typeof State>) => {
  const newMessage = new AIMessage("Hello!");
  // highlight-next-line
  return { messages: [newMessage], extraField: 10 };
};
```
:::

:::python
```python
from langgraph.graph import START

graph = StateGraph(State).add_node(node).add_edge(START, "node").compile()

result = graph.invoke({"messages": [HumanMessage("Hi")]})

for message in result["messages"]:
    message.pretty_print()
```

```
================================ Human Message ================================

Hi
================================== Ai Message ==================================

Hello!
```
:::

:::js
```typescript
import { START } from "@langchain/langgraph";

const graph = new StateGraph(State)
  .addNode("node", node)
  .addEdge(START, "node")
  .compile();

const result = await graph.invoke({ messages: [new HumanMessage("Hi")] });

for (const message of result.messages) {
  console.log(`${message.getType()}: ${message.content}`);
}
```

```
human: Hi
ai: Hello!
```
:::

#### MessagesState

In practice, there are additional considerations for updating lists of messages:

- We may wish to update an existing message in the state.
- We may want to accept short-hands for [message formats](../concepts/low_level.md#using-messages-in-your-graph), such as [OpenAI format](https://python.langchain.com/docs/concepts/messages/#openai-format).

:::python
LangGraph includes a built-in reducer `add_messages` that handles these considerations:

```python
from langgraph.graph.message import add_messages

class State(TypedDict):
    # highlight-next-line
    messages: Annotated[list[AnyMessage], add_messages]
    extra_field: int

def node(state: State):
    new_message = AIMessage("Hello!")
    return {"messages": [new_message], "extra_field": 10}

graph = StateGraph(State).add_node(node).set_entry_point("node").compile()
```

```python
# highlight-next-line
input_message = {"role": "user", "content": "Hi"}

result = graph.invoke({"messages": [input_message]})

for message in result["messages"]:
    message.pretty_print()
```

```
================================ Human Message ================================

Hi
================================== Ai Message ==================================

Hello!
```

This is a versatile representation of state for applications involving [chat models](https://python.langchain.com/docs/concepts/chat_models/). LangGraph includes a pre-built `MessagesState` for convenience, so that we can have:

```python
from langgraph.graph import MessagesState

class State(MessagesState):
    extra_field: int
```
:::

:::js
LangGraph includes a built-in `MessagesZodState` that handles these considerations:

```typescript
import { MessagesZodState } from "@langchain/langgraph";

const State = z.object({
  // highlight-next-line
  messages: MessagesZodState.shape.messages,
  extraField: z.number(),
});

const graph = new StateGraph(State)
  .addNode("node", (state) => {
    const newMessage = new AIMessage("Hello!");
    return { messages: [newMessage], extraField: 10 };
  })
  .addEdge(START, "node")
  .compile();
```

```typescript
// highlight-next-line
const inputMessage = { role: "user", content: "Hi" };

const result = await graph.invoke({ messages: [inputMessage] });

for (const message of result.messages) {
  console.log(`${message.getType()}: ${message.content}`);
}
```

```
human: Hi
ai: Hello!
```

This is a versatile representation of state for applications involving [chat models](https://js.langchain.com/docs/concepts/chat_models/). LangGraph includes this pre-built `MessagesZodState` for convenience, so that we can have:

```typescript
import { MessagesZodState } from "@langchain/langgraph";

const State = MessagesZodState.extend({
  extraField: z.number(),
});
```
:::

### Define input and output schemas

By default, `StateGraph` operates with a single schema, and all nodes are expected to communicate using that schema. However, it's also possible to define distinct input and output schemas for a graph.

When distinct schemas are specified, an internal schema will still be used for communication between nodes. The input schema ensures that the provided input matches the expected structure, while the output schema filters the internal data to return only the relevant information according to the defined output schema.

Below, we'll see how to define distinct input and output schema.

:::python
```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# Define the schema for the input
class InputState(TypedDict):
    question: str

# Define the schema for the output
class OutputState(TypedDict):
    answer: str

# Define the overall schema, combining both input and output
class OverallState(InputState, OutputState):
    pass

# Define the node that processes the input and generates an answer
def answer_node(state: InputState):
    # Example answer and an extra key
    return {"answer": "bye", "question": state["question"]}

# Build the graph with input and output schemas specified
builder = StateGraph(OverallState, input_schema=InputState, output_schema=OutputState)
builder.add_node(answer_node)  # Add the answer node
builder.add_edge(START, "answer_node")  # Define the starting edge
builder.add_edge("answer_node", END)  # Define the ending edge
graph = builder.compile()  # Compile the graph

# Invoke the graph with an input and print the result
print(graph.invoke({"question": "hi"}))
```

```
{'answer': 'bye'}
```
:::

:::js
```typescript
import { StateGraph, START, END } from "@langchain/langgraph";
import { z } from "zod";

// Define the schema for the input
const InputState = z.object({
  question: z.string(),
});

// Define the schema for the output
const OutputState = z.object({
  answer: z.string(),
});

// Define the overall schema, combining both input and output
const OverallState = InputState.merge(OutputState);

// Build the graph with input and output schemas specified
const graph = new StateGraph({
  input: InputState,
  output: OutputState,
  state: OverallState,
})
  .addNode("answerNode", (state) => {
    // Example answer and an extra key
    return { answer: "bye", question: state.question };
  })
  .addEdge(START, "answerNode")
  .addEdge("answerNode", END)
  .compile();

// Invoke the graph with an input and print the result
console.log(await graph.invoke({ question: "hi" }));
```

```
{ answer: 'bye' }
```
:::

Notice that the output of invoke only includes the output schema.

### Pass private state between nodes

In some cases, you may want nodes to exchange information that is crucial for intermediate logic but doesn't need to be part of the main schema of the graph. This private data is not relevant to the overall input/output of the graph and should only be shared between certain nodes.

Below, we'll create an example sequential graph consisting of three nodes (node_1, node_2 and node_3), where private data is passed between the first two steps (node_1 and node_2), while the third step (node_3) only has access to the public overall state.

:::python
```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# The overall state of the graph (this is the public state shared across nodes)
class OverallState(TypedDict):
    a: str

# Output from node_1 contains private data that is not part of the overall state
class Node1Output(TypedDict):
    private_data: str

# The private data is only shared between node_1 and node_2
def node_1(state: OverallState) -> Node1Output:
    output = {"private_data": "set by node_1"}
    print(f"Entered node `node_1`:\n\tInput: {state}.\n\tReturned: {output}")
    return output

# Node 2 input only requests the private data available after node_1
class Node2Input(TypedDict):
    private_data: str

def node_2(state: Node2Input) -> OverallState:
    output = {"a": "set by node_2"}
    print(f"Entered node `node_2`:\n\tInput: {state}.\n\tReturned: {output}")
    return output

# Node 3 only has access to the overall state (no access to private data from node_1)
def node_3(state: OverallState) -> OverallState:
    output = {"a": "set by node_3"}
    print(f"Entered node `node_3`:\n\tInput: {state}.\n\tReturned: {output}")
    return output

# Connect nodes in a sequence
# node_2 accepts private data from node_1, whereas
# node_3 does not see the private data.
builder = StateGraph(OverallState).add_sequence([node_1, node_2, node_3])
builder.add_edge(START, "node_1")
graph = builder.compile()

# Invoke the graph with the initial state
response = graph.invoke(
    {
        "a": "set at start",
    }
)

print()
print(f"Output of graph invocation: {response}")
```

```
Entered node `node_1`:
	Input: {'a': 'set at start'}.
	Returned: {'private_data': 'set by node_1'}
Entered node `node_2`:
	Input: {'private_data': 'set by node_1'}.
	Returned: {'a': 'set by node_2'}
Entered node `node_3`:
	Input: {'a': 'set by node_2'}.
	Returned: {'a': 'set by node_3'}

Output of graph invocation: {'a': 'set by node_3'}
```
:::

:::js
```typescript
import { StateGraph, START, END } from "@langchain/langgraph";
import { z } from "zod";

// The overall state of the graph (this is the public state shared across nodes)
const OverallState = z.object({
  a: z.string(),
});

// Output from node1 contains private data that is not part of the overall state
const Node1Output = z.object({
  privateData: z.string(),
});

// The private data is only shared between node1 and node2
const node1 = (state: z.infer<typeof OverallState>): z.infer<typeof Node1Output> => {
  const output = { privateData: "set by node1" };
  console.log(`Entered node 'node1':\n\tInput: ${JSON.stringify(state)}.\n\tReturned: ${JSON.stringify(output)}`);
  return output;
};

// Node 2 input only requests the private data available after node1
const Node2Input = z.object({
  privateData: z.string(),
});

const node2 = (state: z.infer<typeof Node2Input>): z.infer<typeof OverallState> => {
  const output = { a: "set by node2" };
  console.log(`Entered node 'node2':\n\tInput: ${JSON.stringify(state)}.\n\tReturned: ${JSON.stringify(output)}`);
  return output;
};

// Node 3 only has access to the overall state (no access to private data from node1)
const node3 = (state: z.infer<typeof OverallState>): z.infer<typeof OverallState> => {
  const output = { a: "set by node3" };
  console.log(`Entered node 'node3':\n\tInput: ${JSON.stringify(state)}.\n\tReturned: ${JSON.stringify(output)}`);
  return output;
};

// Connect nodes in a sequence
// node2 accepts private data from node1, whereas
// node3 does not see the private data.
const graph = new StateGraph({
  state: OverallState,
  nodes: {
    node1: { action: node1, output: Node1Output },
    node2: { action: node2, input: Node2Input },
    node3: { action: node3 },
  }
})
  .addEdge(START, "node1")
  .addEdge("node1", "node2")
  .addEdge("node2", "node3")
  .addEdge("node3", END)
  .compile();

// Invoke the graph with the initial state
const response = await graph.invoke({ a: "set at start" });

console.log(`\nOutput of graph invocation: ${JSON.stringify(response)}`);
```

```
Entered node 'node1':
	Input: {"a":"set at start"}.
	Returned: {"privateData":"set by node1"}
Entered node 'node2':
	Input: {"privateData":"set by node1"}.
	Returned: {"a":"set by node2"}
Entered node 'node3':
	Input: {"a":"set by node2"}.
	Returned: {"a":"set by node3"}

Output of graph invocation: {"a":"set by node3"}
```
:::

:::python

### Use Pydantic models for graph state

A [StateGraph](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.StateGraph) accepts a `state_schema` argument on initialization that specifies the "shape" of the state that the nodes in the graph can access and update.

In our examples, we typically use a python-native `TypedDict` or [`dataclass`](https://docs.python.org/3/library/dataclasses.html) for `state_schema`, but `state_schema` can be any [type](https://docs.python.org/3/library/stdtypes.html#type-objects).

Here, we'll see how a [Pydantic BaseModel](https://docs.pydantic.dev/latest/api/base_model/) can be used for `state_schema` to add run-time validation on **inputs**.

!!! note "Known Limitations" 

    - Currently, the output of the graph will **NOT** be an instance of a pydantic model. 
    - Run-time validation only occurs on inputs into nodes, not on the outputs. 
    - The validation error trace from pydantic does not show which node the error arises in. 
    - Pydantic's recursive validation can be slow. For performance-sensitive applications, you may want to consider using a `dataclass` instead.

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict
from pydantic import BaseModel

# The overall state of the graph (this is the public state shared across nodes)
class OverallState(BaseModel):
    a: str

def node(state: OverallState):
    return {"a": "goodbye"}

# Build the state graph
builder = StateGraph(OverallState)
builder.add_node(node)  # node_1 is the first node
builder.add_edge(START, "node")  # Start the graph with node_1
builder.add_edge("node", END)  # End the graph after node_1
graph = builder.compile()

# Test the graph with a valid input
graph.invoke({"a": "hello"})
```

Invoke the graph with an **invalid** input

```python
try:
    graph.invoke({"a": 123})  # Should be a string
except Exception as e:
    print("An exception was raised because `a` is an integer rather than a string.")
    print(e)
```

```
An exception was raised because `a` is an integer rather than a string.
1 validation error for OverallState
a
  Input should be a valid string [type=string_type, input_value=123, input_type=int]
    For further information visit https://errors.pydantic.dev/2.9/v/string_type
```

See below for additional features of Pydantic model state:

??? example "Serialization Behavior"

    When using Pydantic models as state schemas, it's important to understand how serialization works, especially when:
    - Passing Pydantic objects as inputs
    - Receiving outputs from the graph
    - Working with nested Pydantic models

    Let's see these behaviors in action.

    ```python
    from langgraph.graph import StateGraph, START, END
    from pydantic import BaseModel

    class NestedModel(BaseModel):
        value: str

    class ComplexState(BaseModel):
        text: str
        count: int
        nested: NestedModel

    def process_node(state: ComplexState):
        # Node receives a validated Pydantic object
        print(f"Input state type: {type(state)}")
        print(f"Nested type: {type(state.nested)}")
        # Return a dictionary update
        return {"text": state.text + " processed", "count": state.count + 1}

    # Build the graph
    builder = StateGraph(ComplexState)
    builder.add_node("process", process_node)
    builder.add_edge(START, "process")
    builder.add_edge("process", END)
    graph = builder.compile()

    # Create a Pydantic instance for input
    input_state = ComplexState(text="hello", count=0, nested=NestedModel(value="test"))
    print(f"Input object type: {type(input_state)}")

    # Invoke graph with a Pydantic instance
    result = graph.invoke(input_state)
    print(f"Output type: {type(result)}")
    print(f"Output content: {result}")

    # Convert back to Pydantic model if needed
    output_model = ComplexState(**result)
    print(f"Converted back to Pydantic: {type(output_model)}")
    ```

??? example "Runtime Type Coercion"

    Pydantic performs runtime type coercion for certain data types. This can be helpful but also lead to unexpected behavior if you're not aware of it.

    ```python
    from langgraph.graph import StateGraph, START, END
    from pydantic import BaseModel

    class CoercionExample(BaseModel):
        # Pydantic will coerce string numbers to integers
        number: int
        # Pydantic will parse string booleans to bool
        flag: bool

    def inspect_node(state: CoercionExample):
        print(f"number: {state.number} (type: {type(state.number)})")
        print(f"flag: {state.flag} (type: {type(state.flag)})")
        return {}

    builder = StateGraph(CoercionExample)
    builder.add_node("inspect", inspect_node)
    builder.add_edge(START, "inspect")
    builder.add_edge("inspect", END)
    graph = builder.compile()

    # Demonstrate coercion with string inputs that will be converted
    result = graph.invoke({"number": "42", "flag": "true"})

    # This would fail with a validation error
    try:
        graph.invoke({"number": "not-a-number", "flag": "true"})
    except Exception as e:
        print(f"\nExpected validation error: {e}")
    ```

??? example "Working with Message Models"

    When working with LangChain message types in your state schema, there are important considerations for serialization. You should use `AnyMessage` (rather than `BaseMessage`) for proper serialization/deserialization when using message objects over the wire.

    ```python
    from langgraph.graph import StateGraph, START, END
    from pydantic import BaseModel
    from langchain_core.messages import HumanMessage, AIMessage, AnyMessage
    from typing import List

    class ChatState(BaseModel):
        messages: List[AnyMessage]
        context: str

    def add_message(state: ChatState):
        return {"messages": state.messages + [AIMessage(content="Hello there!")]}

    builder = StateGraph(ChatState)
    builder.add_node("add_message", add_message)
    builder.add_edge(START, "add_message")
    builder.add_edge("add_message", END)
    graph = builder.compile()

    # Create input with a message
    initial_state = ChatState(
        messages=[HumanMessage(content="Hi")], context="Customer support chat"
    )

    result = graph.invoke(initial_state)
    print(f"Output: {result}")

    # Convert back to Pydantic model to see message types
    output_model = ChatState(**result)
    for i, msg in enumerate(output_model.messages):
        print(f"Message {i}: {type(msg).__name__} - {msg.content}")
    ```
:::

:::js
### Alternative state definitions

While Zod schemas are the recommended approach, LangGraph also supports other ways to define state schemas:

```typescript
import { BaseMessage } from "@langchain/core/messages";
import { StateGraph } from "@langchain/langgraph";

interface WorkflowChannelsState {
  messages: BaseMessage[];
  question: string;
  answer: string;
}

const workflowWithChannels = new StateGraph<WorkflowChannelsState>({
  channels: {
    messages: {
      reducer: (currentState, updateValue) => currentState.concat(updateValue),
      default: () => [],
    },
    question: null,
    answer: null,
  },
});
```
:::

## Add runtime configuration

Sometimes you want to be able to configure your graph when calling it. For example, you might want to be able to specify what LLM or system prompt to use at runtime, _without polluting the graph state with these parameters_.

To add runtime configuration:

1. Specify a schema for your configuration
2. Add the configuration to the function signature for nodes or conditional edges
3. Pass the configuration into the graph.

See below for a simple example:

:::python
```python
from langgraph.graph import END, StateGraph, START
from langgraph.runtime import Runtime
from typing_extensions import TypedDict

# 1. Specify config schema
class ContextSchema(TypedDict):
    my_runtime_value: str

# 2. Define a graph that accesses the config in a node
class State(TypedDict):
    my_state_value: str

# highlight-next-line
def node(state: State, runtime: Runtime[ContextSchema]):
    # highlight-next-line
    if runtime.context["my_runtime_value"] == "a":
        return {"my_state_value": 1}
        # highlight-next-line
    elif runtime.context["my_runtime_value"] == "b":
        return {"my_state_value": 2}
    else:
        raise ValueError("Unknown values.")

# highlight-next-line
builder = StateGraph(State, context_schema=ContextSchema)
builder.add_node(node)
builder.add_edge(START, "node")
builder.add_edge("node", END)

graph = builder.compile()

# 3. Pass in configuration at runtime:
# highlight-next-line
print(graph.invoke({}, context={"my_runtime_value": "a"}))
# highlight-next-line
print(graph.invoke({}, context={"my_runtime_value": "b"}))
```

```
{'my_state_value': 1}
{'my_state_value': 2}
```
:::

:::js
```typescript
import { StateGraph, END, START } from "@langchain/langgraph";
import { RunnableConfig } from "@langchain/core/runnables";
import { z } from "zod";

// 1. Specify config schema
const ConfigurableSchema = z.object({
  myRuntimeValue: z.string(),
});

// 2. Define a graph that accesses the config in a node
const State = z.object({
  myStateValue: z.number(),
});

const graph = new StateGraph(State)
  .addNode("node", (state, config) => {
    // highlight-next-line
    if (config?.configurable?.myRuntimeValue === "a") {
      return { myStateValue: 1 };
      // highlight-next-line
    } else if (config?.configurable?.myRuntimeValue === "b") {
      return { myStateValue: 2 };
    } else {
      throw new Error("Unknown values.");
    }
  })
  .addEdge(START, "node")
  .addEdge("node", END)
  .compile();

// 3. Pass in configuration at runtime:
// highlight-next-line
console.log(await graph.invoke({}, { configurable: { myRuntimeValue: "a" } }));
// highlight-next-line
console.log(await graph.invoke({}, { configurable: { myRuntimeValue: "b" } }));
```

```
{ myStateValue: 1 }
{ myStateValue: 2 }
```
:::

??? example "Extended example: specifying LLM at runtime"

    :::python
    Below we demonstrate a practical example in which we configure what LLM to use at runtime. We will use both OpenAI and Anthropic models.

    ```python
    from dataclasses import dataclass

    from langchain.chat_models import init_chat_model
    from langgraph.graph import MessagesState, END, StateGraph, START
    from langgraph.runtime import Runtime
    from typing_extensions import TypedDict

    @dataclass
    class ContextSchema:
        model_provider: str = "anthropic"

    MODELS = {
        "anthropic": init_chat_model("anthropic:claude-3-5-haiku-latest"),
        "openai": init_chat_model("openai:gpt-4.1-mini"),
    }

    def call_model(state: MessagesState, runtime: Runtime[ContextSchema]):
        model = MODELS[runtime.context.model_provider]
        response = model.invoke(state["messages"])
        return {"messages": [response]}

    builder = StateGraph(MessagesState, context_schema=ContextSchema)
    builder.add_node("model", call_model)
    builder.add_edge(START, "model")
    builder.add_edge("model", END)

    graph = builder.compile()

    # Usage
    input_message = {"role": "user", "content": "hi"}
    # With no configuration, uses default (Anthropic)
    response_1 = graph.invoke({"messages": [input_message]}, context=ContextSchema())["messages"][-1]
    # Or, can set OpenAI
    response_2 = graph.invoke({"messages": [input_message]}, context={"model_provider": "openai"})["messages"][-1]

    print(response_1.response_metadata["model_name"])
    print(response_2.response_metadata["model_name"])
    ```
    ```
    claude-3-5-haiku-20241022
    gpt-4.1-mini-2025-04-14
    ```
    :::

    :::js
    Below we demonstrate a practical example in which we configure what LLM to use at runtime. We will use both OpenAI and Anthropic models.

    ```typescript
    import { ChatOpenAI } from "@langchain/openai";
    import { ChatAnthropic } from "@langchain/anthropic";
    import { MessagesZodState, StateGraph, START, END } from "@langchain/langgraph";
    import { RunnableConfig } from "@langchain/core/runnables";
    import { z } from "zod";

    const ConfigSchema = z.object({
      modelProvider: z.string().default("anthropic"),
    });

    const MODELS = {
      anthropic: new ChatAnthropic({ model: "claude-3-5-haiku-latest" }),
      openai: new ChatOpenAI({ model: "gpt-4o-mini" }),
    };

    const graph = new StateGraph(MessagesZodState)
      .addNode("model", async (state, config) => {
        const modelProvider = config?.configurable?.modelProvider || "anthropic";
        const model = MODELS[modelProvider as keyof typeof MODELS];
        const response = await model.invoke(state.messages);
        return { messages: [response] };
      })
      .addEdge(START, "model")
      .addEdge("model", END)
      .compile();

    // Usage
    const inputMessage = { role: "user", content: "hi" };
    // With no configuration, uses default (Anthropic)
    const response1 = await graph.invoke({ messages: [inputMessage] });
    // Or, can set OpenAI
    const response2 = await graph.invoke(
      { messages: [inputMessage] },
      { configurable: { modelProvider: "openai" } }
    );

    console.log(response1.messages.at(-1)?.response_metadata?.model);
    console.log(response2.messages.at(-1)?.response_metadata?.model);
    ```
    ```
    claude-3-5-haiku-20241022
    gpt-4o-mini-2024-07-18
    ```
    :::

??? example "Extended example: specifying model and system message at runtime"

    :::python
    Below we demonstrate a practical example in which we configure two parameters: the LLM and system message to use at runtime.

    ```python
    from dataclasses import dataclass
    from typing import Optional
    from langchain.chat_models import init_chat_model
    from langchain_core.messages import SystemMessage
    from langgraph.graph import END, MessagesState, StateGraph, START
    from langgraph.runtime import Runtime
    from typing_extensions import TypedDict

    @dataclass
    class ContextSchema:
        model_provider: str = "anthropic"
        system_message: str | None = None

    MODELS = {
        "anthropic": init_chat_model("anthropic:claude-3-5-haiku-latest"),
        "openai": init_chat_model("openai:gpt-4.1-mini"),
    }

    def call_model(state: MessagesState, runtime: Runtime[ContextSchema]):
        model = MODELS[runtime.context.model_provider]
        messages = state["messages"]
        if (system_message := runtime.context.system_message):
            messages = [SystemMessage(system_message)] + messages
        response = model.invoke(messages)
        return {"messages": [response]}

    builder = StateGraph(MessagesState, context_schema=ContextSchema)
    builder.add_node("model", call_model)
    builder.add_edge(START, "model")
    builder.add_edge("model", END)

    graph = builder.compile()

    # Usage
    input_message = {"role": "user", "content": "hi"}
    response = graph.invoke({"messages": [input_message]}, context={"model_provider": "openai", "system_message": "Respond in Italian."})
    for message in response["messages"]:
        message.pretty_print()
    ```
    ```
    ================================ Human Message ================================

    hi
    ================================== Ai Message ==================================

    Ciao! Come posso aiutarti oggi?
    ```
    :::

    :::js
    Below we demonstrate a practical example in which we configure two parameters: the LLM and system message to use at runtime.

    ```typescript
    import { ChatOpenAI } from "@langchain/openai";
    import { ChatAnthropic } from "@langchain/anthropic";
    import { SystemMessage } from "@langchain/core/messages";
    import { MessagesZodState, StateGraph, START, END } from "@langchain/langgraph";
    import { z } from "zod";

    const ConfigSchema = z.object({
      modelProvider: z.string().default("anthropic"),
      systemMessage: z.string().optional(),
    });

    const MODELS = {
      anthropic: new ChatAnthropic({ model: "claude-3-5-haiku-latest" }),
      openai: new ChatOpenAI({ model: "gpt-4o-mini" }),
    };

    const graph = new StateGraph(MessagesZodState)
      .addNode("model", async (state, config) => {
        const modelProvider = config?.configurable?.modelProvider || "anthropic";
        const systemMessage = config?.configurable?.systemMessage;
        
        const model = MODELS[modelProvider as keyof typeof MODELS];
        let messages = state.messages;
        
        if (systemMessage) {
          messages = [new SystemMessage(systemMessage), ...messages];
        }
        
        const response = await model.invoke(messages);
        return { messages: [response] };
      })
      .addEdge(START, "model")
      .addEdge("model", END)
      .compile();

    // Usage
    const inputMessage = { role: "user", content: "hi" };
    const response = await graph.invoke(
      { messages: [inputMessage] },
      {
        configurable: {
          modelProvider: "openai",
          systemMessage: "Respond in Italian."
        }
      }
    );
    
    for (const message of response.messages) {
      console.log(`${message.getType()}: ${message.content}`);
    }
    ```
    ```
    human: hi
    ai: Ciao! Come posso aiutarti oggi?
    ```
    :::

## Add retry policies

There are many use cases where you may wish for your node to have a custom retry policy, for example if you are calling an API, querying a database, or calling an LLM, etc. LangGraph lets you add retry policies to nodes.

:::python
To configure a retry policy, pass the `retry_policy` parameter to the [add_node](../reference/graphs.md#langgraph.graph.state.StateGraph.add_node). The `retry_policy` parameter takes in a `RetryPolicy` named tuple object. Below we instantiate a `RetryPolicy` object with the default parameters and associate it with a node:

```python
from langgraph.types import RetryPolicy

builder.add_node(
    "node_name",
    node_function,
    retry_policy=RetryPolicy(),
)
```

By default, the `retry_on` parameter uses the `default_retry_on` function, which retries on any exception except for the following:

- `ValueError`
- `TypeError`
- `ArithmeticError`
- `ImportError`
- `LookupError`
- `NameError`
- `SyntaxError`
- `RuntimeError`
- `ReferenceError`
- `StopIteration`
- `StopAsyncIteration`
- `OSError`

In addition, for exceptions from popular http request libraries such as `requests` and `httpx` it only retries on 5xx status codes.
:::

:::js
To configure a retry policy, pass the `retryPolicy` parameter to the [addNode](../reference/graphs.md#langgraph.graph.state.StateGraph.add_node). The `retryPolicy` parameter takes in a `RetryPolicy` object. Below we instantiate a `RetryPolicy` object with the default parameters and associate it with a node:

```typescript
import { RetryPolicy } from "@langchain/langgraph";

const graph = new StateGraph(State)
  .addNode("nodeName", nodeFunction, { retryPolicy: {} })
  .compile();
```

By default, the retry policy retries on any exception except for the following:

- `TypeError`
- `SyntaxError`
- `ReferenceError`
:::

??? example "Extended example: customizing retry policies"

    :::python
    Consider an example in which we are reading from a SQL database. Below we pass two different retry policies to nodes:

    ```python
    import sqlite3
    from typing_extensions import TypedDict
    from langchain.chat_models import init_chat_model
    from langgraph.graph import END, MessagesState, StateGraph, START
    from langgraph.types import RetryPolicy
    from langchain_community.utilities import SQLDatabase
    from langchain_core.messages import AIMessage

    db = SQLDatabase.from_uri("sqlite:///:memory:")
    model = init_chat_model("anthropic:claude-3-5-haiku-latest")

    def query_database(state: MessagesState):
        query_result = db.run("SELECT * FROM Artist LIMIT 10;")
        return {"messages": [AIMessage(content=query_result)]}

    def call_model(state: MessagesState):
        response = model.invoke(state["messages"])
        return {"messages": [response]}

    # Define a new graph
    builder = StateGraph(MessagesState)
    builder.add_node(
        "query_database",
        query_database,
        retry_policy=RetryPolicy(retry_on=sqlite3.OperationalError),
    )
    builder.add_node("model", call_model, retry_policy=RetryPolicy(max_attempts=5))
    builder.add_edge(START, "model")
    builder.add_edge("model", "query_database")
    builder.add_edge("query_database", END)
    graph = builder.compile()
    ```
    :::

    :::js
    Consider an example in which we are reading from a SQL database. Below we pass two different retry policies to nodes:

    ```typescript
    import Database from "better-sqlite3";
    import { ChatAnthropic } from "@langchain/anthropic";
    import { StateGraph, START, END, MessagesZodState } from "@langchain/langgraph";
    import { AIMessage } from "@langchain/core/messages";
    import { z } from "zod";

    // Create an in-memory database
    const db: typeof Database.prototype = new Database(":memory:");

    const model = new ChatAnthropic({ model: "claude-3-5-sonnet-20240620" });

    const callModel = async (state: z.infer<typeof MessagesZodState>) => {
      const response = await model.invoke(state.messages);
      return { messages: [response] };
    };

    const queryDatabase = async (state: z.infer<typeof MessagesZodState>) => {
      const queryResult: string = JSON.stringify(
        db.prepare("SELECT * FROM Artist LIMIT 10;").all(),
      );

      return { messages: [new AIMessage({ content: "queryResult" })] };
    };

    const workflow = new StateGraph(MessagesZodState)
      // Define the two nodes we will cycle between
      .addNode("call_model", callModel, { retryPolicy: { maxAttempts: 5 } })
      .addNode("query_database", queryDatabase, {
        retryPolicy: {
          retryOn: (e: any): boolean => {
            if (e instanceof Database.SqliteError) {
              // Retry on "SQLITE_BUSY" error
              return e.code === "SQLITE_BUSY";
            }
            return false; // Don't retry on other errors
          },
        },
      })
      .addEdge(START, "call_model")
      .addEdge("call_model", "query_database")
      .addEdge("query_database", END);

    const graph = workflow.compile();
    ```
    :::

:::python

## Add node caching

Node caching is useful in cases where you want to avoid repeating operations, like when doing something expensive (either in terms of time or cost). LangGraph lets you add individualized caching policies to nodes in a graph.

To configure a cache policy, pass the `cache_policy` parameter to the [add_node](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.state.StateGraph.add_node) function. In the following example, a [`CachePolicy`](https://langchain-ai.github.io/langgraph/reference/types/?h=cachepolicy#langgraph.types.CachePolicy) object is instantiated with a time to live of 120 seconds and the default `key_func` generator. Then it is associated with a node:

```python
from langgraph.types import CachePolicy

builder.add_node(
    "node_name",
    node_function,
    cache_policy=CachePolicy(ttl=120),
)
```

Then, to enable node-level caching for a graph, set the `cache` argument when compiling the graph. The example below uses `InMemoryCache` to set up a graph with in-memory cache, but `SqliteCache` is also available.

```python
from langgraph.cache.memory import InMemoryCache

graph = builder.compile(cache=InMemoryCache())
```
:::

## Create a sequence of steps

!!! info "Prerequisites"

    This guide assumes familiarity with the above section on [state](#define-and-update-state).

Here we demonstrate how to construct a simple sequence of steps. We will show:

1. How to build a sequential graph
2. Built-in short-hand for constructing similar graphs.

:::python
To add a sequence of nodes, we use the `.add_node` and `.add_edge` methods of our [graph](../concepts/low_level.md#stategraph):

```python
from langgraph.graph import START, StateGraph

builder = StateGraph(State)

# Add nodes
builder.add_node(step_1)
builder.add_node(step_2)
builder.add_node(step_3)

# Add edges
builder.add_edge(START, "step_1")
builder.add_edge("step_1", "step_2")
builder.add_edge("step_2", "step_3")
```

We can also use the built-in shorthand `.add_sequence`:

```python
builder = StateGraph(State).add_sequence([step_1, step_2, step_3])
builder.add_edge(START, "step_1")
```
:::

:::js
To add a sequence of nodes, we use the `.addNode` and `.addEdge` methods of our [graph](../concepts/low_level.md#stategraph):

```typescript
import { START, StateGraph } from "@langchain/langgraph";

const builder = new StateGraph(State)
  .addNode("step1", step1)
  .addNode("step2", step2)
  .addNode("step3", step3)
  .addEdge(START, "step1")
  .addEdge("step1", "step2")
  .addEdge("step2", "step3");
```
:::

??? info "Why split application steps into a sequence with LangGraph?"
    LangGraph makes it easy to add an underlying persistence layer to your application.
    This allows state to be checkpointed in between the execution of nodes, so your LangGraph nodes govern:

- How state updates are [checkpointed](../concepts/persistence.md)
- How interruptions are resumed in [human-in-the-loop](../concepts/human_in_the_loop.md) workflows
- How we can "rewind" and branch-off executions using LangGraph's [time travel](../concepts/time-travel.md) features

They also determine how execution steps are [streamed](../concepts/streaming.md), and how your application is visualized
and debugged using [LangGraph Studio](../concepts/langgraph_studio.md).

Let's demonstrate an end-to-end example. We will create a sequence of three steps:

1. Populate a value in a key of the state
2. Update the same value
3. Populate a different value

Let's first define our [state](../concepts/low_level.md#state). This governs the [schema of the graph](../concepts/low_level.md#schema), and can also specify how to apply updates. See [this section](#process-state-updates-with-reducers) for more detail.

In our case, we will just keep track of two values:

:::python
```python
from typing_extensions import TypedDict

class State(TypedDict):
    value_1: str
    value_2: int
```
:::

:::js
```typescript
import { z } from "zod";

const State = z.object({
  value1: z.string(),
  value2: z.number(),
});
```
:::

:::python
Our [nodes](../concepts/low_level.md#nodes) are just Python functions that read our graph's state and make updates to it. The first argument to this function will always be the state:

```python
def step_1(state: State):
    return {"value_1": "a"}

def step_2(state: State):
    current_value_1 = state["value_1"]
    return {"value_1": f"{current_value_1} b"}

def step_3(state: State):
    return {"value_2": 10}
```
:::

:::js
Our [nodes](../concepts/low_level.md#nodes) are just TypeScript functions that read our graph's state and make updates to it. The first argument to this function will always be the state:

```typescript
const step1 = (state: z.infer<typeof State>) => {
  return { value1: "a" };
};

const step2 = (state: z.infer<typeof State>) => {
  const currentValue1 = state.value1;
  return { value1: `${currentValue1} b` };
};

const step3 = (state: z.infer<typeof State>) => {
  return { value2: 10 };
};
```
:::

!!! note

    Note that when issuing updates to the state, each node can just specify the value of the key it wishes to update.

    By default, this will **overwrite** the value of the corresponding key. You can also use [reducers](../concepts/low_level.md#reducers) to control how updates are processed— for example, you can append successive updates to a key instead. See [this section](#process-state-updates-with-reducers) for more detail.

Finally, we define the graph. We use [StateGraph](../concepts/low_level.md#stategraph) to define a graph that operates on this state.

:::python
We will then use [add_node](../concepts/low_level.md#messagesstate) and [add_edge](../concepts/low_level.md#edges) to populate our graph and define its control flow.

```python
from langgraph.graph import START, StateGraph

builder = StateGraph(State)

# Add nodes
builder.add_node(step_1)
builder.add_node(step_2)
builder.add_node(step_3)

# Add edges
builder.add_edge(START, "step_1")
builder.add_edge("step_1", "step_2")
builder.add_edge("step_2", "step_3")
```
:::

:::js
We will then use [addNode](../concepts/low_level.md#nodes) and [addEdge](../concepts/low_level.md#edges) to populate our graph and define its control flow.

```typescript
import { START, StateGraph } from "@langchain/langgraph";

const graph = new StateGraph(State)
  .addNode("step1", step1)
  .addNode("step2", step2)
  .addNode("step3", step3)
  .addEdge(START, "step1")
  .addEdge("step1", "step2")
  .addEdge("step2", "step3")
  .compile();
```
:::

:::python
!!! tip "Specifying custom names"

    You can specify custom names for nodes using `.add_node`:

    ```python
    builder.add_node("my_node", step_1)
    ```
:::

:::js
!!! tip "Specifying custom names"

    You can specify custom names for nodes using `.addNode`:

    ```typescript
    const graph = new StateGraph(State)
      .addNode("myNode", step1)
      .compile();
    ```
:::

Note that:

:::python
- `.add_edge` takes the names of nodes, which for functions defaults to `node.__name__`.
- We must specify the entry point of the graph. For this we add an edge with the [START node](../concepts/low_level.md#start-node).
- The graph halts when there are no more nodes to execute.

We next [compile](../concepts/low_level.md#compiling-your-graph) our graph. This provides a few basic checks on the structure of the graph (e.g., identifying orphaned nodes). If we were adding persistence to our application via a [checkpointer](../concepts/persistence.md), it would also be passed in here.

```python
graph = builder.compile()
```
:::

:::js
- `.addEdge` takes the names of nodes, which for functions defaults to `node.name`.
- We must specify the entry point of the graph. For this we add an edge with the [START node](../concepts/low_level.md#start-node).
- The graph halts when there are no more nodes to execute.

We next [compile](../concepts/low_level.md#compiling-your-graph) our graph. This provides a few basic checks on the structure of the graph (e.g., identifying orphaned nodes). If we were adding persistence to our application via a [checkpointer](../concepts/persistence.md), it would also be passed in here.
:::

LangGraph provides built-in utilities for visualizing your graph. Let's inspect our sequence. See [this guide](#visualize-your-graph) for detail on visualization.

:::python
```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![Sequence of steps graph](assets/graph_api_image_2.png)
:::

:::js
```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("graph.png", imageBuffer);
```
:::

Let's proceed with a simple invocation:

:::python
```python
graph.invoke({"value_1": "c"})
```

```
{'value_1': 'a b', 'value_2': 10}
```
:::

:::js
```typescript
const result = await graph.invoke({ value1: "c" });
console.log(result);
```

```
{ value1: 'a b', value2: 10 }
```
:::

Note that:

- We kicked off invocation by providing a value for a single state key. We must always provide a value for at least one key.
- The value we passed in was overwritten by the first node.
- The second node updated the value.
- The third node populated a different value.

:::python
!!! tip "Built-in shorthand"

    `langgraph>=0.2.46` includes a built-in short-hand `add_sequence` for adding node sequences. You can compile the same graph as follows:

    ```python
    # highlight-next-line
    builder = StateGraph(State).add_sequence([step_1, step_2, step_3])
    builder.add_edge(START, "step_1")

    graph = builder.compile()

    graph.invoke({"value_1": "c"})
    ```
:::

## Create branches

Parallel execution of nodes is essential to speed up overall graph operation. LangGraph offers native support for parallel execution of nodes, which can significantly enhance the performance of graph-based workflows. This parallelization is achieved through fan-out and fan-in mechanisms, utilizing both standard edges and [conditional_edges](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.MessageGraph.add_conditional_edges). Below are some examples showing how to add create branching dataflows that work for you.

### Run graph nodes in parallel

In this example, we fan out from `Node A` to `B and C` and then fan in to `D`. With our state, [we specify the reducer add operation](https://langchain-ai.github.io/langgraph/concepts/low_level.md#reducers). This will combine or accumulate values for the specific key in the State, rather than simply overwriting the existing value. For lists, this means concatenating the new list with the existing list. See the above section on [state reducers](#process-state-updates-with-reducers) for more detail on updating state with reducers.

:::python
```python
import operator
from typing import Annotated, Any
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    # The operator.add reducer fn makes this append-only
    aggregate: Annotated[list, operator.add]

def a(state: State):
    print(f'Adding "A" to {state["aggregate"]}')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'Adding "B" to {state["aggregate"]}')
    return {"aggregate": ["B"]}

def c(state: State):
    print(f'Adding "C" to {state["aggregate"]}')
    return {"aggregate": ["C"]}

def d(state: State):
    print(f'Adding "D" to {state["aggregate"]}')
    return {"aggregate": ["D"]}

builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(c)
builder.add_node(d)
builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("a", "c")
builder.add_edge("b", "d")
builder.add_edge("c", "d")
builder.add_edge("d", END)
graph = builder.compile()
```
:::

:::js
```typescript
import "@langchain/langgraph/zod";
import { StateGraph, START, END } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  // The reducer makes this append-only
  aggregate: z.array(z.string()).langgraph.reducer((x, y) => x.concat(y)),
});

const nodeA = (state: z.infer<typeof State>) => {
  console.log(`Adding "A" to ${state.aggregate}`);
  return { aggregate: ["A"] };
};

const nodeB = (state: z.infer<typeof State>) => {
  console.log(`Adding "B" to ${state.aggregate}`);
  return { aggregate: ["B"] };
};

const nodeC = (state: z.infer<typeof State>) => {
  console.log(`Adding "C" to ${state.aggregate}`);
  return { aggregate: ["C"] };
};

const nodeD = (state: z.infer<typeof State>) => {
  console.log(`Adding "D" to ${state.aggregate}`);
  return { aggregate: ["D"] };
};

const graph = new StateGraph(State)
  .addNode("a", nodeA)
  .addNode("b", nodeB)
  .addNode("c", nodeC)
  .addNode("d", nodeD)
  .addEdge(START, "a")
  .addEdge("a", "b")
  .addEdge("a", "c")
  .addEdge("b", "d")
  .addEdge("c", "d")
  .addEdge("d", END)
  .compile();
```
:::

:::python
```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![Parallel execution graph](assets/graph_api_image_3.png)
:::

:::js
```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("graph.png", imageBuffer);
```
:::

With the reducer, you can see that the values added in each node are accumulated.

:::python
```python
graph.invoke({"aggregate": []}, {"configurable": {"thread_id": "foo"}})
```

```
Adding "A" to []
Adding "B" to ['A']
Adding "C" to ['A']
Adding "D" to ['A', 'B', 'C']
```
:::

:::js
```typescript
const result = await graph.invoke({
  aggregate: [],
});
console.log(result);
```

```
Adding "A" to []
Adding "B" to ['A']
Adding "C" to ['A']
Adding "D" to ['A', 'B', 'C']
{ aggregate: ['A', 'B', 'C', 'D'] }
```
:::

!!! note

    In the above example, nodes `"b"` and `"c"` are executed concurrently in the same [superstep](../concepts/low_level.md#graphs). Because they are in the same step, node `"d"` executes after both `"b"` and `"c"` are finished.

    Importantly, updates from a parallel superstep may not be ordered consistently. If you need a consistent, predetermined ordering of updates from a parallel superstep, you should write the outputs to a separate field in the state together with a value with which to order them.

??? note "Exception handling?"

    LangGraph executes nodes within [supersteps](../concepts/low_level.md#graphs), meaning that while parallel branches are executed in parallel, the entire superstep is **transactional**. If any of these branches raises an exception, **none** of the updates are applied to the state (the entire superstep errors).

    Importantly, when using a [checkpointer](../concepts/persistence.md), results from successful nodes within a superstep are saved, and don't repeat when resumed.

    If you have error-prone (perhaps want to handle flakey API calls), LangGraph provides two ways to address this:

    1. You can write regular python code within your node to catch and handle exceptions.
    2. You can set a **[retry_policy](../reference/types.md#langgraph.types.RetryPolicy)** to direct the graph to retry nodes that raise certain types of exceptions. Only failing branches are retried, so you needn't worry about performing redundant work.

    Together, these let you perform parallel execution and fully control exception handling.

:::python

### Defer node execution

Deferring node execution is useful when you want to delay the execution of a node until all other pending tasks are completed. This is particularly relevant when branches have different lengths, which is common in workflows like map-reduce flows.

The above example showed how to fan-out and fan-in when each path was only one step. But what if one branch had more than one step? Let's add a node `"b_2"` in the `"b"` branch:

```python
import operator
from typing import Annotated, Any
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    # The operator.add reducer fn makes this append-only
    aggregate: Annotated[list, operator.add]

def a(state: State):
    print(f'Adding "A" to {state["aggregate"]}')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'Adding "B" to {state["aggregate"]}')
    return {"aggregate": ["B"]}

def b_2(state: State):
    print(f'Adding "B_2" to {state["aggregate"]}')
    return {"aggregate": ["B_2"]}

def c(state: State):
    print(f'Adding "C" to {state["aggregate"]}')
    return {"aggregate": ["C"]}

def d(state: State):
    print(f'Adding "D" to {state["aggregate"]}')
    return {"aggregate": ["D"]}

builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(b_2)
builder.add_node(c)
# highlight-next-line
builder.add_node(d, defer=True)
builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("a", "c")
builder.add_edge("b", "b_2")
builder.add_edge("b_2", "d")
builder.add_edge("c", "d")
builder.add_edge("d", END)
graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![Deferred execution graph](assets/graph_api_image_4.png)

```python
graph.invoke({"aggregate": []})
```

```
Adding "A" to []
Adding "B" to ['A']
Adding "C" to ['A']
Adding "B_2" to ['A', 'B', 'C']
Adding "D" to ['A', 'B', 'C', 'B_2']
```

In the above example, nodes `"b"` and `"c"` are executed concurrently in the same superstep. We set `defer=True` on node `d` so it will not execute until all pending tasks are finished. In this case, this means that `"d"` waits to execute until the entire `"b"` branch is finished.
:::

### Conditional branching

:::python
If your fan-out should vary at runtime based on the state, you can use [add_conditional_edges](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.StateGraph.add_conditional_edges) to select one or more paths using the graph state. See example below, where node `a` generates a state update that determines the following node.

```python
import operator
from typing import Annotated, Literal, Sequence
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    aggregate: Annotated[list, operator.add]
    # Add a key to the state. We will set this key to determine
    # how we branch.
    which: str

def a(state: State):
    print(f'Adding "A" to {state["aggregate"]}')
    # highlight-next-line
    return {"aggregate": ["A"], "which": "c"}

def b(state: State):
    print(f'Adding "B" to {state["aggregate"]}')
    return {"aggregate": ["B"]}

def c(state: State):
    print(f'Adding "C" to {state["aggregate"]}')
    return {"aggregate": ["C"]}

builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(c)
builder.add_edge(START, "a")
builder.add_edge("b", END)
builder.add_edge("c", END)

def conditional_edge(state: State) -> Literal["b", "c"]:
    # Fill in arbitrary logic here that uses the state
    # to determine the next node
    return state["which"]

# highlight-next-line
builder.add_conditional_edges("a", conditional_edge)

graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![Conditional branching graph](assets/graph_api_image_5.png)

```python
result = graph.invoke({"aggregate": []})
print(result)
```

```
Adding "A" to []
Adding "C" to ['A']
{'aggregate': ['A', 'C'], 'which': 'c'}
```
:::

:::js
If your fan-out should vary at runtime based on the state, you can use [addConditionalEdges](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.StateGraph.addConditionalEdges) to select one or more paths using the graph state. See example below, where node `a` generates a state update that determines the following node.

```typescript
import "@langchain/langgraph/zod";
import { StateGraph, START, END } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  aggregate: z.array(z.string()).langgraph.reducer((x, y) => x.concat(y)),
  // Add a key to the state. We will set this key to determine
  // how we branch.
  which: z.string().langgraph.reducer((x, y) => y ?? x),
});

const nodeA = (state: z.infer<typeof State>) => {
  console.log(`Adding "A" to ${state.aggregate}`);
  // highlight-next-line
  return { aggregate: ["A"], which: "c" };
};

const nodeB = (state: z.infer<typeof State>) => {
  console.log(`Adding "B" to ${state.aggregate}`);
  return { aggregate: ["B"] };
};

const nodeC = (state: z.infer<typeof State>) => {
  console.log(`Adding "C" to ${state.aggregate}`);
  return { aggregate: ["C"] };
};

const conditionalEdge = (state: z.infer<typeof State>): "b" | "c" => {
  // Fill in arbitrary logic here that uses the state
  // to determine the next node
  return state.which as "b" | "c";
};

// highlight-next-line
const graph = new StateGraph(State)
  .addNode("a", nodeA)  
  .addNode("b", nodeB)
  .addNode("c", nodeC)
  .addEdge(START, "a")
  .addEdge("b", END)
  .addEdge("c", END)
  .addConditionalEdges("a", conditionalEdge)
  .compile();
```

```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("graph.png", imageBuffer);
```

```typescript
const result = await graph.invoke({ aggregate: [] });
console.log(result);
```

```
Adding "A" to []
Adding "C" to ['A']
{ aggregate: ['A', 'C'], which: 'c' }
```
:::

!!! tip

    Your conditional edges can route to multiple destination nodes. For example:

    :::python
    ```python
    def route_bc_or_cd(state: State) -> Sequence[str]:
        if state["which"] == "cd":
            return ["c", "d"]
        return ["b", "c"]
    ```
    :::

    :::js
    ```typescript
    const routeBcOrCd = (state: z.infer<typeof State>): string[] => {
      if (state.which === "cd") {
        return ["c", "d"];
      }
      return ["b", "c"];
    };
    ```
    :::

## Map-Reduce and the Send API

LangGraph supports map-reduce and other advanced branching patterns using the Send API. Here is an example of how to use it:

:::python
```python
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send
from typing_extensions import TypedDict, Annotated
import operator

class OverallState(TypedDict):
    topic: str
    subjects: list[str]
    jokes: Annotated[list[str], operator.add]
    best_selected_joke: str

def generate_topics(state: OverallState):
    return {"subjects": ["lions", "elephants", "penguins"]}

def generate_joke(state: OverallState):
    joke_map = {
        "lions": "Why don't lions like fast food? Because they can't catch it!",
        "elephants": "Why don't elephants use computers? They're afraid of the mouse!",
        "penguins": "Why don't penguins like talking to strangers at parties? Because they find it hard to break the ice."
    }
    return {"jokes": [joke_map[state["subject"]]]}

def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]

def best_joke(state: OverallState):
    return {"best_selected_joke": "penguins"}

builder = StateGraph(OverallState)
builder.add_node("generate_topics", generate_topics)
builder.add_node("generate_joke", generate_joke)
builder.add_node("best_joke", best_joke)
builder.add_edge(START, "generate_topics")
builder.add_conditional_edges("generate_topics", continue_to_jokes, ["generate_joke"])
builder.add_edge("generate_joke", "best_joke")
builder.add_edge("best_joke", END)
graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![Map-reduce graph with fanout](assets/graph_api_image_6.png)

```python
# Call the graph: here we call it to generate a list of jokes
for step in graph.stream({"topic": "animals"}):
    print(step)
```

```
{'generate_topics': {'subjects': ['lions', 'elephants', 'penguins']}}
{'generate_joke': {'jokes': ["Why don't lions like fast food? Because they can't catch it!"]}}
{'generate_joke': {'jokes': ["Why don't elephants use computers? They're afraid of the mouse!"]}}
{'generate_joke': {'jokes': ['Why don't penguins like talking to strangers at parties? Because they find it hard to break the ice.']}}
{'best_joke': {'best_selected_joke': 'penguins'}}
```
:::

:::js
```typescript
import "@langchain/langgraph/zod";
import { StateGraph, START, END, Send } from "@langchain/langgraph";
import { z } from "zod";

const OverallState = z.object({
  topic: z.string(),
  subjects: z.array(z.string()),
  jokes: z.array(z.string()).langgraph.reducer((x, y) => x.concat(y)),
  bestSelectedJoke: z.string(),
});

const generateTopics = (state: z.infer<typeof OverallState>) => {
  return { subjects: ["lions", "elephants", "penguins"] };
};

const generateJoke = (state: { subject: string }) => {
  const jokeMap: Record<string, string> = {
    lions: "Why don't lions like fast food? Because they can't catch it!",
    elephants: "Why don't elephants use computers? They're afraid of the mouse!",
    penguins: "Why don't penguins like talking to strangers at parties? Because they find it hard to break the ice."
  };
  return { jokes: [jokeMap[state.subject]] };
};

const continueToJokes = (state: z.infer<typeof OverallState>) => {
  return state.subjects.map((subject) => new Send("generateJoke", { subject }));
};

const bestJoke = (state: z.infer<typeof OverallState>) => {
  return { bestSelectedJoke: "penguins" };
};

const graph = new StateGraph(OverallState)
  .addNode("generateTopics", generateTopics)
  .addNode("generateJoke", generateJoke)
  .addNode("bestJoke", bestJoke)
  .addEdge(START, "generateTopics")
  .addConditionalEdges("generateTopics", continueToJokes)
  .addEdge("generateJoke", "bestJoke")
  .addEdge("bestJoke", END)
  .compile();
```

```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("graph.png", imageBuffer);
```

```typescript
// Call the graph: here we call it to generate a list of jokes
for await (const step of await graph.stream({ topic: "animals" })) {
  console.log(step);
}
```

```
{ generateTopics: { subjects: [ 'lions', 'elephants', 'penguins' ] } }
{ generateJoke: { jokes: [ "Why don't lions like fast food? Because they can't catch it!" ] } }
{ generateJoke: { jokes: [ "Why don't elephants use computers? They're afraid of the mouse!" ] } }
{ generateJoke: { jokes: [ "Why don't penguins like talking to strangers at parties? Because they find it hard to break the ice." ] } }
{ bestJoke: { bestSelectedJoke: 'penguins' } }
```
:::

## Create and control loops

When creating a graph with a loop, we require a mechanism for terminating execution. This is most commonly done by adding a [conditional edge](../concepts/low_level.md#conditional-edges) that routes to the [END](../concepts/low_level.md#end-node) node once we reach some termination condition.

You can also set the graph recursion limit when invoking or streaming the graph. The recursion limit sets the number of [supersteps](../concepts/low_level.md#graphs) that the graph is allowed to execute before it raises an error. Read more about the concept of recursion limits [here](../concepts/low_level.md#recursion-limit).

Let's consider a simple graph with a loop to better understand how these mechanisms work.

!!! tip

    To return the last value of your state instead of receiving a recursion limit error, see the [next section](#impose-a-recursion-limit).

When creating a loop, you can include a conditional edge that specifies a termination condition:

:::python
```python
builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)

def route(state: State) -> Literal["b", END]:
    if termination_condition(state):
        return END
    else:
        return "b"

builder.add_edge(START, "a")
builder.add_conditional_edges("a", route)
builder.add_edge("b", "a")
graph = builder.compile()
```
:::

:::js
```typescript
const graph = new StateGraph(State)
  .addNode("a", nodeA)
  .addNode("b", nodeB)
  .addEdge(START, "a")
  .addConditionalEdges("a", route)
  .addEdge("b", "a")
  .compile();

const route = (state: z.infer<typeof State>): "b" | typeof END => {
  if (terminationCondition(state)) {
    return END;
  } else {
    return "b";
  }
};
```
:::

To control the recursion limit, specify `"recursionLimit"` in the config. This will raise a `GraphRecursionError`, which you can catch and handle:

:::python
```python
from langgraph.errors import GraphRecursionError

try:
    graph.invoke(inputs, {"recursion_limit": 3})
except GraphRecursionError:
    print("Recursion Error")
```
:::

:::js
```typescript
import { GraphRecursionError } from "@langchain/langgraph";

try {
  await graph.invoke(inputs, { recursionLimit: 3 });
} catch (error) {
  if (error instanceof GraphRecursionError) {
    console.log("Recursion Error");
  }
}
```
:::

Let's define a graph with a simple loop. Note that we use a conditional edge to implement a termination condition.

:::python
```python
import operator
from typing import Annotated, Literal
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    # The operator.add reducer fn makes this append-only
    aggregate: Annotated[list, operator.add]

def a(state: State):
    print(f'Node A sees {state["aggregate"]}')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'Node B sees {state["aggregate"]}')
    return {"aggregate": ["B"]}

# Define nodes
builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)

# Define edges
def route(state: State) -> Literal["b", END]:
    if len(state["aggregate"]) < 7:
        return "b"
    else:
        return END

builder.add_edge(START, "a")
builder.add_conditional_edges("a", route)
builder.add_edge("b", "a")
graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![Simple loop graph](assets/graph_api_image_7.png)
:::

:::js
```typescript
import "@langchain/langgraph/zod";
import { StateGraph, START, END } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  // The reducer makes this append-only
  aggregate: z.array(z.string()).langgraph.reducer((x, y) => x.concat(y)),
});

const nodeA = (state: z.infer<typeof State>) => {
  console.log(`Node A sees ${state.aggregate}`);
  return { aggregate: ["A"] };
};

const nodeB = (state: z.infer<typeof State>) => {
  console.log(`Node B sees ${state.aggregate}`);
  return { aggregate: ["B"] };
};

// Define edges
const route = (state: z.infer<typeof State>): "b" | typeof END => {
  if (state.aggregate.length < 7) {
    return "b";
  } else {
    return END;
  }
};

const graph = new StateGraph(State)
  .addNode("a", nodeA)
  .addNode("b", nodeB)
  .addEdge(START, "a")
  .addConditionalEdges("a", route)
  .addEdge("b", "a")
  .compile();
```

```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("graph.png", imageBuffer);
```
:::

This architecture is similar to a [React agent](../agents/overview.md) in which node `"a"` is a tool-calling model, and node `"b"` represents the tools.

In our `route` conditional edge, we specify that we should end after the `"aggregate"` list in the state passes a threshold length.

Invoking the graph, we see that we alternate between nodes `"a"` and `"b"` before terminating once we reach the termination condition.

:::python
```python
graph.invoke({"aggregate": []})
```

```
Node A sees []
Node B sees ['A']
Node A sees ['A', 'B']
Node B sees ['A', 'B', 'A']
Node A sees ['A', 'B', 'A', 'B']
Node B sees ['A', 'B', 'A', 'B', 'A']
Node A sees ['A', 'B', 'A', 'B', 'A', 'B']
```
:::

:::js
```typescript
const result = await graph.invoke({ aggregate: [] });
console.log(result);
```

```
Node A sees []
Node B sees ['A']
Node A sees ['A', 'B']
Node B sees ['A', 'B', 'A']
Node A sees ['A', 'B', 'A', 'B']
Node B sees ['A', 'B', 'A', 'B', 'A']
Node A sees ['A', 'B', 'A', 'B', 'A', 'B']
{ aggregate: ['A', 'B', 'A', 'B', 'A', 'B', 'A'] }
```
:::

### Impose a recursion limit

In some applications, we may not have a guarantee that we will reach a given termination condition. In these cases, we can set the graph's [recursion limit](../concepts/low_level.md#recursion-limit). This will raise a `GraphRecursionError` after a given number of [supersteps](../concepts/low_level.md#graphs). We can then catch and handle this exception:

:::python
```python
from langgraph.errors import GraphRecursionError

try:
    graph.invoke({"aggregate": []}, {"recursion_limit": 4})
except GraphRecursionError:
    print("Recursion Error")
```

```
Node A sees []
Node B sees ['A']
Node C sees ['A', 'B']
Node D sees ['A', 'B']
Node A sees ['A', 'B', 'C', 'D']
Recursion Error
```
:::

:::js
```typescript
import { GraphRecursionError } from "@langchain/langgraph";

try {
  await graph.invoke({ aggregate: [] }, { recursionLimit: 4 });
} catch (error) {
  if (error instanceof GraphRecursionError) {
    console.log("Recursion Error");
  }
}
```

```
Node A sees []
Node B sees ['A']
Node A sees ['A', 'B']
Node B sees ['A', 'B', 'A']
Node A sees ['A', 'B', 'A', 'B']
Recursion Error
```
:::


:::python
??? example "Extended example: return state on hitting recursion limit"

    Instead of raising `GraphRecursionError`, we can introduce a new key to the state that keeps track of the number of steps remaining until reaching the recursion limit. We can then use this key to determine if we should end the run.

    LangGraph implements a special `RemainingSteps` annotation. Under the hood, it creates a `ManagedValue` channel -- a state channel that will exist for the duration of our graph run and no longer.

    ```python
    import operator
    from typing import Annotated, Literal
    from typing_extensions import TypedDict
    from langgraph.graph import StateGraph, START, END
    from langgraph.managed.is_last_step import RemainingSteps

    class State(TypedDict):
        aggregate: Annotated[list, operator.add]
        remaining_steps: RemainingSteps

    def a(state: State):
        print(f'Node A sees {state["aggregate"]}')
        return {"aggregate": ["A"]}

    def b(state: State):
        print(f'Node B sees {state["aggregate"]}')
        return {"aggregate": ["B"]}

    # Define nodes
    builder = StateGraph(State)
    builder.add_node(a)
    builder.add_node(b)

    # Define edges
    def route(state: State) -> Literal["b", END]:
        if state["remaining_steps"] <= 2:
            return END
        else:
            return "b"

    builder.add_edge(START, "a")
    builder.add_conditional_edges("a", route)
    builder.add_edge("b", "a")
    graph = builder.compile()

    # Test it out
    result = graph.invoke({"aggregate": []}, {"recursion_limit": 4})
    print(result)
    ```
    ```
    Node A sees []
    Node B sees ['A']
    Node A sees ['A', 'B']
    {'aggregate': ['A', 'B', 'A']}
    ```
:::

:::python
??? example "Extended example: loops with branches"

    To better understand how the recursion limit works, let's consider a more complex example. Below we implement a loop, but one step fans out into two nodes:

    ```python
    import operator
    from typing import Annotated, Literal
    from typing_extensions import TypedDict
    from langgraph.graph import StateGraph, START, END

    class State(TypedDict):
        aggregate: Annotated[list, operator.add]

    def a(state: State):
        print(f'Node A sees {state["aggregate"]}')
        return {"aggregate": ["A"]}

    def b(state: State):
        print(f'Node B sees {state["aggregate"]}')
        return {"aggregate": ["B"]}

    def c(state: State):
        print(f'Node C sees {state["aggregate"]}')
        return {"aggregate": ["C"]}

    def d(state: State):
        print(f'Node D sees {state["aggregate"]}')
        return {"aggregate": ["D"]}

    # Define nodes
    builder = StateGraph(State)
    builder.add_node(a)
    builder.add_node(b)
    builder.add_node(c)
    builder.add_node(d)

    # Define edges
    def route(state: State) -> Literal["b", END]:
        if len(state["aggregate"]) < 7:
            return "b"
        else:
            return END

    builder.add_edge(START, "a")
    builder.add_conditional_edges("a", route)
    builder.add_edge("b", "c")
    builder.add_edge("b", "d")
    builder.add_edge(["c", "d"], "a")
    graph = builder.compile()
    ```

    ```python
    from IPython.display import Image, display

    display(Image(graph.get_graph().draw_mermaid_png()))
    ```

    ![Complex loop graph with branches](assets/graph_api_image_8.png)

    This graph looks complex, but can be conceptualized as loop of [supersteps](../concepts/low_level.md#graphs):

    1. Node A
    2. Node B
    3. Nodes C and D
    4. Node A
    5. ...

    We have a loop of four supersteps, where nodes C and D are executed concurrently.

    Invoking the graph as before, we see that we complete two full "laps" before hitting the termination condition:

    ```python
    result = graph.invoke({"aggregate": []})
    ```
    ```
    Node A sees []
    Node B sees ['A']
    Node D sees ['A', 'B']
    Node C sees ['A', 'B']
    Node A sees ['A', 'B', 'C', 'D']
    Node B sees ['A', 'B', 'C', 'D', 'A']
    Node D sees ['A', 'B', 'C', 'D', 'A', 'B']
    Node C sees ['A', 'B', 'C', 'D', 'A', 'B']
    Node A sees ['A', 'B', 'C', 'D', 'A', 'B', 'C', 'D']
    ```

    However, if we set the recursion limit to four, we only complete one lap because each lap is four supersteps:

    ```python
    from langgraph.errors import GraphRecursionError

    try:
        result = graph.invoke({"aggregate": []}, {"recursion_limit": 4})
    except GraphRecursionError:
        print("Recursion Error")
    ```
    ```
    Node A sees []
    Node B sees ['A']
    Node C sees ['A', 'B']
    Node D sees ['A', 'B']
    Node A sees ['A', 'B', 'C', 'D']
    Recursion Error
    ```
:::

:::python

## Async

Using the async programming paradigm can produce significant performance improvements when running [IO-bound](https://en.wikipedia.org/wiki/I/O_bound) code concurrently (e.g., making concurrent API requests to a chat model provider).

To convert a `sync` implementation of the graph to an `async` implementation, you will need to:

1. Update `nodes` use `async def` instead of `def`.
2. Update the code inside to use `await` appropriately.
3. Invoke the graph with `.ainvoke` or `.astream` as desired.

Because many LangChain objects implement the [Runnable Protocol](https://python.langchain.com/docs/expression_language/interface/) which has `async` variants of all the `sync` methods it's typically fairly quick to upgrade a `sync` graph to an `async` graph.

See example below. To demonstrate async invocations of underlying LLMs, we will include a chat model:

{% include-markdown "../../snippets/chat_model_tabs.md" %}

```python
from langchain.chat_models import init_chat_model
from langgraph.graph import MessagesState, StateGraph

# highlight-next-line
async def node(state: MessagesState): # (1)!
    # highlight-next-line
    new_message = await llm.ainvoke(state["messages"]) # (2)!
    return {"messages": [new_message]}

builder = StateGraph(MessagesState).add_node(node).set_entry_point("node")
graph = builder.compile()

input_message = {"role": "user", "content": "Hello"}
# highlight-next-line
result = await graph.ainvoke({"messages": [input_message]}) # (3)!
```

1. Declare nodes to be async functions.
2. Use async invocations when available within the node.
3. Use async invocations on the graph object itself.

!!! tip "Async streaming"

    See the [streaming guide](./streaming.md) for examples of streaming with async.

:::

## Combine control flow and state updates with `Command`

It can be useful to combine control flow (edges) and state updates (nodes). For example, you might want to BOTH perform state updates AND decide which node to go to next in the SAME node. LangGraph provides a way to do so by returning a [Command](../reference/types.md#langgraph.types.Command) object from node functions:

:::python
```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        # state update
        update={"foo": "bar"},
        # control flow
        goto="my_other_node"
    )
```
:::

:::js
```typescript
import { Command } from "@langchain/langgraph";

const myNode = (state: State): Command => {
  return new Command({
    // state update
    update: { foo: "bar" },
    // control flow
    goto: "myOtherNode"
  });
};
```
:::

We show an end-to-end example below. Let's create a simple graph with 3 nodes: A, B and C. We will first execute node A, and then decide whether to go to Node B or Node C next based on the output of node A.

:::python
```python
import random
from typing_extensions import TypedDict, Literal
from langgraph.graph import StateGraph, START
from langgraph.types import Command

# Define graph state
class State(TypedDict):
    foo: str

# Define the nodes

def node_a(state: State) -> Command[Literal["node_b", "node_c"]]:
    print("Called A")
    value = random.choice(["b", "c"])
    # this is a replacement for a conditional edge function
    if value == "b":
        goto = "node_b"
    else:
        goto = "node_c"

    # note how Command allows you to BOTH update the graph state AND route to the next node
    return Command(
        # this is the state update
        update={"foo": value},
        # this is a replacement for an edge
        goto=goto,
    )

def node_b(state: State):
    print("Called B")
    return {"foo": state["foo"] + "b"}

def node_c(state: State):
    print("Called C")
    return {"foo": state["foo"] + "c"}
```

We can now create the `StateGraph` with the above nodes. Notice that the graph doesn't have [conditional edges](../concepts/low_level.md#conditional-edges) for routing! This is because control flow is defined with `Command` inside `node_a`.

```python
builder = StateGraph(State)
builder.add_edge(START, "node_a")
builder.add_node(node_a)
builder.add_node(node_b)
builder.add_node(node_c)
# NOTE: there are no edges between nodes A, B and C!

graph = builder.compile()
```

!!! important

    You might have noticed that we used `Command` as a return type annotation, e.g. `Command[Literal["node_b", "node_c"]]`. This is necessary for the graph rendering and tells LangGraph that `node_a` can navigate to `node_b` and `node_c`.

```python
from IPython.display import display, Image

display(Image(graph.get_graph().draw_mermaid_png()))
```

![Command-based graph navigation](assets/graph_api_image_11.png)

If we run the graph multiple times, we'd see it take different paths (A -> B or A -> C) based on the random choice in node A.

```python
graph.invoke({"foo": ""})
```

```
Called A
Called C
```
:::

:::js
```typescript
import { StateGraph, START, Command } from "@langchain/langgraph";
import { z } from "zod";

// Define graph state
const State = z.object({
  foo: z.string(),
});

// Define the nodes

const nodeA = (state: z.infer<typeof State>): Command => {
  console.log("Called A");
  const value = Math.random() > 0.5 ? "b" : "c";
  // this is a replacement for a conditional edge function  
  const goto = value === "b" ? "nodeB" : "nodeC";

  // note how Command allows you to BOTH update the graph state AND route to the next node
  return new Command({
    // this is the state update
    update: { foo: value },
    // this is a replacement for an edge
    goto,
  });
};

const nodeB = (state: z.infer<typeof State>) => {
  console.log("Called B");
  return { foo: state.foo + "b" };
};

const nodeC = (state: z.infer<typeof State>) => {
  console.log("Called C");
  return { foo: state.foo + "c" };
};
```

We can now create the `StateGraph` with the above nodes. Notice that the graph doesn't have [conditional edges](../concepts/low_level.md#conditional-edges) for routing! This is because control flow is defined with `Command` inside `nodeA`.

```typescript
const graph = new StateGraph(State)
  .addNode("nodeA", nodeA, {
    ends: ["nodeB", "nodeC"],
  })
  .addNode("nodeB", nodeB)
  .addNode("nodeC", nodeC)
  .addEdge(START, "nodeA")
  .compile();
```

!!! important

    You might have noticed that we used `ends` to specify which nodes `nodeA` can navigate to. This is necessary for the graph rendering and tells LangGraph that `nodeA` can navigate to `nodeB` and `nodeC`.

```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("graph.png", imageBuffer);
```

If we run the graph multiple times, we'd see it take different paths (A -> B or A -> C) based on the random choice in node A.

```typescript
const result = await graph.invoke({ foo: "" });
console.log(result);
```

```
Called A
Called C
{ foo: 'cc' }
```
:::

### Navigate to a node in a parent graph

If you are using [subgraphs](../concepts/subgraphs.md), you might want to navigate from a node within a subgraph to a different subgraph (i.e. a different node in the parent graph). To do so, you can specify `graph=Command.PARENT` in `Command`:

:::python
```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        update={"foo": "bar"},
        goto="other_subgraph",  # where `other_subgraph` is a node in the parent graph
        graph=Command.PARENT
    )
```
:::

:::js
```typescript
const myNode = (state: State): Command => {
  return new Command({
    update: { foo: "bar" },
    goto: "otherSubgraph",  // where `otherSubgraph` is a node in the parent graph
    graph: Command.PARENT
  });
};
```
:::

Let's demonstrate this using the above example. We'll do so by changing `nodeA` in the above example into a single-node graph that we'll add as a subgraph to our parent graph.

!!! important "State updates with `Command.PARENT`"

    When you send updates from a subgraph node to a parent graph node for a key that's shared by both parent and subgraph [state schemas](../concepts/low_level.md#schema), you **must** define a [reducer](../concepts/low_level.md#reducers) for the key you're updating in the parent graph state. See the example below.

:::python
```python
import operator
from typing_extensions import Annotated

class State(TypedDict):
    # NOTE: we define a reducer here
    # highlight-next-line
    foo: Annotated[str, operator.add]

def node_a(state: State):
    print("Called A")
    value = random.choice(["a", "b"])
    # this is a replacement for a conditional edge function
    if value == "a":
        goto = "node_b"
    else:
        goto = "node_c"

    # note how Command allows you to BOTH update the graph state AND route to the next node
    return Command(
        update={"foo": value},
        goto=goto,
        # this tells LangGraph to navigate to node_b or node_c in the parent graph
        # NOTE: this will navigate to the closest parent graph relative to the subgraph
        # highlight-next-line
        graph=Command.PARENT,
    )

subgraph = StateGraph(State).add_node(node_a).add_edge(START, "node_a").compile()

def node_b(state: State):
    print("Called B")
    # NOTE: since we've defined a reducer, we don't need to manually append
    # new characters to existing 'foo' value. instead, reducer will append these
    # automatically (via operator.add)
    # highlight-next-line
    return {"foo": "b"}

def node_c(state: State):
    print("Called C")
    # highlight-next-line
    return {"foo": "c"}

builder = StateGraph(State)
builder.add_edge(START, "subgraph")
builder.add_node("subgraph", subgraph)
builder.add_node(node_b)
builder.add_node(node_c)

graph = builder.compile()
```

```python
graph.invoke({"foo": ""})
```

```
Called A
Called C
```
:::

:::js
```typescript
import "@langchain/langgraph/zod";
import { StateGraph, START, Command } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  // NOTE: we define a reducer here
  // highlight-next-line
  foo: z.string().langgraph.reducer((x, y) => x + y),
});

const nodeA = (state: z.infer<typeof State>) => {
  console.log("Called A");
  const value = Math.random() > 0.5 ? "nodeB" : "nodeC";
  
  // note how Command allows you to BOTH update the graph state AND route to the next node
  return new Command({
    update: { foo: "a" },
    goto: value,
    // this tells LangGraph to navigate to nodeB or nodeC in the parent graph
    // NOTE: this will navigate to the closest parent graph relative to the subgraph
    // highlight-next-line
    graph: Command.PARENT,
  });
};

const subgraph = new StateGraph(State)
  .addNode("nodeA", nodeA, { ends: ["nodeB", "nodeC"] })
  .addEdge(START, "nodeA")
  .compile();

const nodeB = (state: z.infer<typeof State>) => {
  console.log("Called B");
  // NOTE: since we've defined a reducer, we don't need to manually append
  // new characters to existing 'foo' value. instead, reducer will append these
  // automatically
  // highlight-next-line
  return { foo: "b" };
};

const nodeC = (state: z.infer<typeof State>) => {
  console.log("Called C");
  // highlight-next-line
  return { foo: "c" };
};

const graph = new StateGraph(State)
  .addNode("subgraph", subgraph, { ends: ["nodeB", "nodeC"] })
  .addNode("nodeB", nodeB)
  .addNode("nodeC", nodeC)
  .addEdge(START, "subgraph")
  .compile();
```

```typescript
const result = await graph.invoke({ foo: "" });
console.log(result);
```

```
Called A
Called C
{ foo: 'ac' }
```
:::

### Use inside tools

A common use case is updating graph state from inside a tool. For example, in a customer support application you might want to look up customer information based on their account number or ID in the beginning of the conversation. To update the graph state from the tool, you can return `Command(update={"my_custom_key": "foo", "messages": [...]})` from the tool:

:::python
```python
@tool
def lookup_user_info(tool_call_id: Annotated[str, InjectedToolCallId], config: RunnableConfig):
    """Use this to look up user information to better assist them with their questions."""
    user_info = get_user_info(config.get("configurable", {}).get("user_id"))
    return Command(
        update={
            # update the state keys
            "user_info": user_info,
            # update the message history
            "messages": [ToolMessage("Successfully looked up user information", tool_call_id=tool_call_id)]
        }
    )
```
:::

:::js
```typescript
import { tool } from "@langchain/core/tools";
import { Command } from "@langchain/langgraph";
import { RunnableConfig } from "@langchain/core/runnables";
import { z } from "zod";

const lookupUserInfo = tool(
  async (input, config: RunnableConfig) => {
    const userId = config.configurable?.userId;
    const userInfo = getUserInfo(userId);
    return new Command({
      update: {
        // update the state keys
        userInfo: userInfo,
        // update the message history
        messages: [{
          role: "tool",
          content: "Successfully looked up user information",
          tool_call_id: config.toolCall.id
        }]
      }
    });
  },
  {
    name: "lookupUserInfo",
    description: "Use this to look up user information to better assist them with their questions.",
    schema: z.object({}),
  }
);
```
:::

!!! important

    You MUST include `messages` (or any state key used for the message history) in `Command.update` when returning `Command` from a tool and the list of messages in `messages` MUST contain a `ToolMessage`. This is necessary for the resulting message history to be valid (LLM providers require AI messages with tool calls to be followed by the tool result messages).

If you are using tools that update state via `Command`, we recommend using prebuilt [`ToolNode`](../reference/agents.md#langgraph.prebuilt.tool_node.ToolNode) which automatically handles tools returning `Command` objects and propagates them to the graph state. If you're writing a custom node that calls tools, you would need to manually propagate `Command` objects returned by the tools as the update from the node.

## Visualize your graph

Here we demonstrate how to visualize the graphs you create.

You can visualize any arbitrary [Graph](https://langchain-ai.github.io/langgraph/reference/graphs/), including [StateGraph](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.state.StateGraph). 

:::python
Let's have some fun by drawing fractals :).

```python
import random
from typing import Annotated, Literal
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]

class MyNode:
    def __init__(self, name: str):
        self.name = name
    def __call__(self, state: State):
        return {"messages": [("assistant", f"Called node {self.name}")]}

def route(state) -> Literal["entry_node", "__end__"]:
    if len(state["messages"]) > 10:
        return "__end__"
    return "entry_node"

def add_fractal_nodes(builder, current_node, level, max_level):
    if level > max_level:
        return
    # Number of nodes to create at this level
    num_nodes = random.randint(1, 3)  # Adjust randomness as needed
    for i in range(num_nodes):
        nm = ["A", "B", "C"][i]
        node_name = f"node_{current_node}_{nm}"
        builder.add_node(node_name, MyNode(node_name))
        builder.add_edge(current_node, node_name)
        # Recursively add more nodes
        r = random.random()
        if r > 0.2 and level + 1 < max_level:
            add_fractal_nodes(builder, node_name, level + 1, max_level)
        elif r > 0.05:
            builder.add_conditional_edges(node_name, route, node_name)
        else:
            # End
            builder.add_edge(node_name, "__end__")

def build_fractal_graph(max_level: int):
    builder = StateGraph(State)
    entry_point = "entry_node"
    builder.add_node(entry_point, MyNode(entry_point))
    builder.add_edge(START, entry_point)
    add_fractal_nodes(builder, entry_point, 1, max_level)
    # Optional: set a finish point if required
    builder.add_edge(entry_point, END)  # or any specific node
    return builder.compile()

app = build_fractal_graph(3)
```
:::

:::js
Let's create a simple example graph to demonstrate visualization.

```typescript
import { StateGraph, START, END } from "@langchain/langgraph";
import { MessagesZodState } from "@langchain/langgraph";
import { z } from "zod";

const State = MessagesZodState.extend({
  value: z.number(),
});

const app = new StateGraph(State)
  .addNode("node1", (state) => {
    return { value: state.value + 1 };
  })
  .addNode("node2", (state) => {
    return { value: state.value * 2 };
  })
  .addEdge(START, "node1")
  .addConditionalEdges("node1", (state) => {
    if (state.value < 10) {
      return "node2";
    }
    return END;
  })
  .addEdge("node2", "node1")
  .compile();
```
:::

### Mermaid

We can also convert a graph class into Mermaid syntax.

:::python
```python
print(app.get_graph().draw_mermaid())
```

```
%%{init: {'flowchart': {'curve': 'linear'}}}%%
graph TD;
	__start__([<p>__start__</p>]):::first
	entry_node(entry_node)
	node_entry_node_A(node_entry_node_A)
	node_entry_node_B(node_entry_node_B)
	node_node_entry_node_B_A(node_node_entry_node_B_A)
	node_node_entry_node_B_B(node_node_entry_node_B_B)
	node_node_entry_node_B_C(node_node_entry_node_B_C)
	__end__([<p>__end__</p>]):::last
	__start__ --> entry_node;
	entry_node --> __end__;
	entry_node --> node_entry_node_A;
	entry_node --> node_entry_node_B;
	node_entry_node_B --> node_node_entry_node_B_A;
	node_entry_node_B --> node_node_entry_node_B_B;
	node_entry_node_B --> node_node_entry_node_B_C;
	node_entry_node_A -.-> entry_node;
	node_entry_node_A -.-> __end__;
	node_node_entry_node_B_A -.-> entry_node;
	node_node_entry_node_B_A -.-> __end__;
	node_node_entry_node_B_B -.-> entry_node;
	node_node_entry_node_B_B -.-> __end__;
	node_node_entry_node_B_C -.-> entry_node;
	node_node_entry_node_B_C -.-> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
```
:::

:::js
```typescript
const drawableGraph = await app.getGraphAsync();
console.log(drawableGraph.drawMermaid());
```

```
%%{init: {'flowchart': {'curve': 'linear'}}}%%
graph TD;
	__start__([<p>__start__</p>]):::first
	node1(node1)
	node2(node2)
	__end__([<p>__end__</p>]):::last
	__start__ --> node1;
	node1 -.-> node2;
	node1 -.-> __end__;
	node2 --> node1;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
```
:::

### PNG

:::python
If preferred, we could render the Graph into a `.png`. Here we could use three options:

- Using Mermaid.ink API (does not require additional packages)
- Using Mermaid + Pyppeteer (requires `pip install pyppeteer`)
- Using graphviz (which requires `pip install graphviz`)

**Using Mermaid.Ink**

By default, `draw_mermaid_png()` uses Mermaid.Ink's API to generate the diagram.

```python
from IPython.display import Image, display
from langchain_core.runnables.graph import CurveStyle, MermaidDrawMethod, NodeStyles

display(Image(app.get_graph().draw_mermaid_png()))
```

![Fractal graph visualization](assets/graph_api_image_10.png)

**Using Mermaid + Pyppeteer**

```python
import nest_asyncio

nest_asyncio.apply()  # Required for Jupyter Notebook to run async functions

display(
    Image(
        app.get_graph().draw_mermaid_png(
            curve_style=CurveStyle.LINEAR,
            node_colors=NodeStyles(first="#ffdfba", last="#baffc9", default="#fad7de"),
            wrap_label_n_words=9,
            output_file_path=None,
            draw_method=MermaidDrawMethod.PYPPETEER,
            background_color="white",
            padding=10,
        )
    )
)
```

**Using Graphviz**

```python
try:
    display(Image(app.get_graph().draw_png()))
except ImportError:
    print(
        "You likely need to install dependencies for pygraphviz, see more here https://github.com/pygraphviz/pygraphviz/blob/main/INSTALL.txt"
    )
```
:::

:::js
If preferred, we could render the Graph into a `.png`. This uses the Mermaid.ink API to generate the diagram.

```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await app.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("graph.png", imageBuffer);
```
:::

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/many-tools.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "5fa317ef-b9a7-4432-ba85-ce71b8dfbdc6",
   "metadata": {},
   "source": [
    "# How to handle large numbers of tools\n",
    "\n",
    "<div class=\"admonition tip\">\n",
    "    <p class=\"admonition-title\">Prerequisites</p>\n",
    "    <p>\n",
    "        This guide assumes familiarity with the following:\n",
    "        <ul>\n",
    "            <li>\n",
    "                <a href=\"https://python.langchain.com/docs/concepts/#tools\">\n",
    "                    Tools\n",
    "                </a>\n",
    "            </li>\n",
    "            <li>\n",
    "                <a href=\"https://python.langchain.com/docs/concepts/#chat-models/\">\n",
    "                    Chat Models\n",
    "                </a>\n",
    "            </li>\n",
    "            <li>\n",
    "                <a href=\"https://python.langchain.com/docs/concepts/#embedding-models\">\n",
    "                    Embedding Models\n",
    "                </a>\n",
    "            </li>\n",
    "            <li>\n",
    "                <a href=\"https://python.langchain.com/docs/concepts/#vector-stores\">\n",
    "                    Vectorstores\n",
    "                </a>\n",
    "            </li>   \n",
    "            <li>\n",
    "                <a href=\"https://python.langchain.com/docs/concepts/#documents\">\n",
    "                    Document\n",
    "                </a>\n",
    "            </li>\n",
    "        </ul>\n",
    "    </p>\n",
    "</div> \n",
    "\n",
    "\n",
    "The subset of available tools to call is generally at the discretion of the model (although many providers also enable the user to [specify or constrain the choice of tool](https://python.langchain.com/docs/how_to/tool_choice/)). As the number of available tools grows, you may want to limit the scope of the LLM's selection, to decrease token consumption and to help manage sources of error in LLM reasoning.\n",
    "\n",
    "Here we will demonstrate how to dynamically adjust the tools available to a model. Bottom line up front: like [RAG](https://python.langchain.com/docs/concepts/#retrieval) and similar methods, we prefix the model invocation by retrieving over available tools. Although we demonstrate one implementation that searches over tool descriptions, the details of the tool selection can be customized as needed.\n",
    "\n",
    "## Setup\n",
    "\n",
    "First, let's install the required packages and set our API keys"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 1,
   "id": "9b6c62bd",
   "metadata": {},
   "outputs": [],
   "source": [
    "%%capture --no-stderr\n",
    "%pip install --quiet -U langgraph langchain_openai numpy"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "360d7ff6",
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
   "id": "25f9f6a0",
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
   "id": "1a417013-ddc4-463b-8ea0-0904bd232827",
   "metadata": {},
   "source": [
    "## Define the tools"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "24708f3b-18b1-4b42-9f6a-0d4827222918",
   "metadata": {},
   "source": [
    "Let's consider a toy example in which we have one tool for each publicly traded company in the [S&P 500 index](https://en.wikipedia.org/wiki/S%26P_500). Each tool fetches company-specific information based on the year provided as a parameter.\n",
    "\n",
    "We first construct a registry that associates a unique identifier with a schema for each tool. We will represent the tools using JSON schema, which can be bound directly to chat models supporting tool calling."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 3,
   "id": "da30c3f1-127f-4828-8609-94e16719f0be",
   "metadata": {},
   "outputs": [],
   "source": [
    "import re\n",
    "import uuid\n",
    "\n",
    "from langchain_core.tools import StructuredTool\n",
    "\n",
    "\n",
    "def create_tool(company: str) -> dict:\n",
    "    \"\"\"Create schema for a placeholder tool.\"\"\"\n",
    "    # Remove non-alphanumeric characters and replace spaces with underscores for the tool name\n",
    "    formatted_company = re.sub(r\"[^\\w\\s]\", \"\", company).replace(\" \", \"_\")\n",
    "\n",
    "    def company_tool(year: int) -> str:\n",
    "        # Placeholder function returning static revenue information for the company and year\n",
    "        return f\"{company} had revenues of $100 in {year}.\"\n",
    "\n",
    "    return StructuredTool.from_function(\n",
    "        company_tool,\n",
    "        name=formatted_company,\n",
    "        description=f\"Information about {company}\",\n",
    "    )\n",
    "\n",
    "\n",
    "# Abbreviated list of S&P 500 companies for demonstration\n",
    "s_and_p_500_companies = [\n",
    "    \"3M\",\n",
    "    \"A.O. Smith\",\n",
    "    \"Abbott\",\n",
    "    \"Accenture\",\n",
    "    \"Advanced Micro Devices\",\n",
    "    \"Yum! Brands\",\n",
    "    \"Zebra Technologies\",\n",
    "    \"Zimmer Biomet\",\n",
    "    \"Zoetis\",\n",
    "]\n",
    "\n",
    "# Create a tool for each company and store it in a registry with a unique UUID as the key\n",
    "tool_registry = {\n",
    "    str(uuid.uuid4()): create_tool(company) for company in s_and_p_500_companies\n",
    "}"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "ba17b047-73ed-4385-adc2-f02012db2206",
   "metadata": {},
   "source": [
    "## Define the graph"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "2055548d-3d14-4aaf-9588-abf70f28b5d6",
   "metadata": {},
   "source": [
    "### Tool selection"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "8798a0d2-ea93-45bc-ab55-071ab975f2c2",
   "metadata": {},
   "source": [
    "We will construct a node that retrieves a subset of available tools given the information in the state-- such as a recent user message. In general, the full scope of [retrieval solutions](https://python.langchain.com/docs/concepts/#retrieval) are available for this step. As a simple solution, we index embeddings of tool descriptions in a vector store, and associate user queries to tools via semantic search."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 4,
   "id": "435b0201-7296-4617-abf8-2c757a71f6b5",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langchain_core.documents import Document\n",
    "from langchain_core.vectorstores import InMemoryVectorStore\n",
    "from langchain_openai import OpenAIEmbeddings\n",
    "\n",
    "tool_documents = [\n",
    "    Document(\n",
    "        page_content=tool.description,\n",
    "        id=id,\n",
    "        metadata={\"tool_name\": tool.name},\n",
    "    )\n",
    "    for id, tool in tool_registry.items()\n",
    "]\n",
    "\n",
    "vector_store = InMemoryVectorStore(embedding=OpenAIEmbeddings())\n",
    "document_ids = vector_store.add_documents(tool_documents)"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "e9ce366b-b5e7-41e9-b4a9-d775b9be0d09",
   "metadata": {},
   "source": [
    "### Incorporating with an agent\n",
    "\n",
    "We will use a typical React agent graph (e.g., as used in the [quickstart](https://langchain-ai.github.io/langgraph/tutorials/introduction/#part-2-enhancing-the-chatbot-with-tools)), with some modifications:\n",
    "\n",
    "- We add a `selected_tools` key to the state, which stores our selected subset of tools;\n",
    "- We set the entry point of the graph to be a `select_tools` node, which populates this element of the state;\n",
    "- We bind the selected subset of tools to the chat model within the `agent` node."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 6,
   "id": "d319fea9-e8ae-4763-a785-b2bf72239ae4",
   "metadata": {},
   "outputs": [],
   "source": [
    "from typing import Annotated\n",
    "\n",
    "from langchain_openai import ChatOpenAI\n",
    "from typing_extensions import TypedDict\n",
    "\n",
    "from langgraph.graph import StateGraph, START\n",
    "from langgraph.graph.message import add_messages\n",
    "from langgraph.prebuilt import ToolNode, tools_condition\n",
    "\n",
    "\n",
    "# Define the state structure using TypedDict.\n",
    "# It includes a list of messages (processed by add_messages)\n",
    "# and a list of selected tool IDs.\n",
    "class State(TypedDict):\n",
    "    messages: Annotated[list, add_messages]\n",
    "    selected_tools: list[str]\n",
    "\n",
    "\n",
    "builder = StateGraph(State)\n",
    "\n",
    "# Retrieve all available tools from the tool registry.\n",
    "tools = list(tool_registry.values())\n",
    "llm = ChatOpenAI()\n",
    "\n",
    "\n",
    "# The agent function processes the current state\n",
    "# by binding selected tools to the LLM.\n",
    "def agent(state: State):\n",
    "    # Map tool IDs to actual tools\n",
    "    # based on the state's selected_tools list.\n",
    "    selected_tools = [tool_registry[id] for id in state[\"selected_tools\"]]\n",
    "    # Bind the selected tools to the LLM for the current interaction.\n",
    "    llm_with_tools = llm.bind_tools(selected_tools)\n",
    "    # Invoke the LLM with the current messages and return the updated message list.\n",
    "    return {\"messages\": [llm_with_tools.invoke(state[\"messages\"])]}\n",
    "\n",
    "\n",
    "# The select_tools function selects tools based on the user's last message content.\n",
    "def select_tools(state: State):\n",
    "    last_user_message = state[\"messages\"][-1]\n",
    "    query = last_user_message.content\n",
    "    tool_documents = vector_store.similarity_search(query)\n",
    "    return {\"selected_tools\": [document.id for document in tool_documents]}\n",
    "\n",
    "\n",
    "builder.add_node(\"agent\", agent)\n",
    "builder.add_node(\"select_tools\", select_tools)\n",
    "\n",
    "tool_node = ToolNode(tools=tools)\n",
    "builder.add_node(\"tools\", tool_node)\n",
    "\n",
    "builder.add_conditional_edges(\"agent\", tools_condition, path_map=[\"tools\", \"__end__\"])\n",
    "builder.add_edge(\"tools\", \"agent\")\n",
    "builder.add_edge(\"select_tools\", \"agent\")\n",
    "builder.add_edge(START, \"select_tools\")\n",
    "graph = builder.compile()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 7,
   "id": "35cab3b2-4d03-4cb5-ba10-f7d3a5ad5244",
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
   "cell_type": "code",
   "execution_count": 14,
   "id": "66f62a69-989b-46ce-80b3-97a867e36782",
   "metadata": {},
   "outputs": [],
   "source": [
    "user_input = \"Can you give me some information about AMD in 2022?\"\n",
    "\n",
    "result = graph.invoke({\"messages\": [(\"user\", user_input)]})"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 15,
   "id": "479a459d-6896-4960-aae9-9f1259fb47d1",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "['ab9c0d59-3d16-448d-910c-73cf10a26020', 'f5eff8f6-7fb9-47b6-b54f-19872a52db84', '2962e168-9ef4-48dc-8b7c-9227e7956d39', '24a9fb82-19fe-4a88-944e-47bc4032e94a']\n"
     ]
    }
   ],
   "source": [
    "print(result[\"selected_tools\"])"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 16,
   "id": "376f28fd-3f7f-4ae5-a34c-baef1778e82b",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "================================\u001b[1m Human Message \u001b[0m=================================\n",
      "\n",
      "Can you give me some information about AMD in 2022?\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "Tool Calls:\n",
      "  Advanced_Micro_Devices (call_CRxQ0oT7NY7lqf35DaRNTJ35)\n",
      " Call ID: call_CRxQ0oT7NY7lqf35DaRNTJ35\n",
      "  Args:\n",
      "    year: 2022\n",
      "=================================\u001b[1m Tool Message \u001b[0m=================================\n",
      "Name: Advanced_Micro_Devices\n",
      "\n",
      "Advanced Micro Devices had revenues of $100 in 2022.\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "In 2022, Advanced Micro Devices (AMD) had revenues of $100.\n"
     ]
    }
   ],
   "source": [
    "for message in result[\"messages\"]:\n",
    "    message.pretty_print()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "3bd847ef-4627-4fc2-99f9-c2b17cf83f95",
   "metadata": {},
   "source": [
    "## Repeating tool selection\n",
    "\n",
    "To manage errors from incorrect tool selection, we could revisit the `select_tools` node. One option for implementing this is to modify `select_tools` to generate the vector store query using all messages in the state (e.g., with a chat model) and add an edge routing from `tools` to `select_tools`.\n",
    "\n",
    "We implement this change below. For demonstration purposes, we simulate an error in the initial tool selection by adding a `hack_remove_tool_condition` to the `select_tools` node, which removes the correct tool on the first iteration of the node. Note that on the second iteration, the agent finishes the run as it has access to the correct tool."
   ]
  },
  {
   "cell_type": "markdown",
   "id": "985a5388",
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
   "execution_count": null,
   "id": "1954a5f1-91e4-4b32-9be9-c8bc1cc43cb5",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langchain_core.messages import HumanMessage, SystemMessage, ToolMessage\n",
    "from langgraph.pregel.retry import RetryPolicy\n",
    "\n",
    "from pydantic import BaseModel, Field\n",
    "\n",
    "\n",
    "class QueryForTools(BaseModel):\n",
    "    \"\"\"Generate a query for additional tools.\"\"\"\n",
    "\n",
    "    query: str = Field(..., description=\"Query for additional tools.\")\n",
    "\n",
    "\n",
    "def select_tools(state: State):\n",
    "    \"\"\"Selects tools based on the last message in the conversation state.\n",
    "\n",
    "    If the last message is from a human, directly uses the content of the message\n",
    "    as the query. Otherwise, constructs a query using a system message and invokes\n",
    "    the LLM to generate tool suggestions.\n",
    "    \"\"\"\n",
    "    last_message = state[\"messages\"][-1]\n",
    "    hack_remove_tool_condition = False  # Simulate an error in the first tool selection\n",
    "\n",
    "    if isinstance(last_message, HumanMessage):\n",
    "        query = last_message.content\n",
    "        hack_remove_tool_condition = True  # Simulate wrong tool selection\n",
    "    else:\n",
    "        assert isinstance(last_message, ToolMessage)\n",
    "        system = SystemMessage(\n",
    "            \"Given this conversation, generate a query for additional tools. \"\n",
    "            \"The query should be a short string containing what type of information \"\n",
    "            \"is needed. If no further information is needed, \"\n",
    "            \"set more_information_needed False and populate a blank string for the query.\"\n",
    "        )\n",
    "        input_messages = [system] + state[\"messages\"]\n",
    "        response = llm.bind_tools([QueryForTools], tool_choice=True).invoke(\n",
    "            input_messages\n",
    "        )\n",
    "        query = response.tool_calls[0][\"args\"][\"query\"]\n",
    "\n",
    "    # Search the tool vector store using the generated query\n",
    "    tool_documents = vector_store.similarity_search(query)\n",
    "    if hack_remove_tool_condition:\n",
    "        # Simulate error by removing the correct tool from the selection\n",
    "        selected_tools = [\n",
    "            document.id\n",
    "            for document in tool_documents\n",
    "            if document.metadata[\"tool_name\"] != \"Advanced_Micro_Devices\"\n",
    "        ]\n",
    "    else:\n",
    "        selected_tools = [document.id for document in tool_documents]\n",
    "    return {\"selected_tools\": selected_tools}\n",
    "\n",
    "\n",
    "graph_builder = StateGraph(State)\n",
    "graph_builder.add_node(\"agent\", agent)\n",
    "graph_builder.add_node(\n",
    "    \"select_tools\", select_tools, retry_policy=RetryPolicy(max_attempts=3)\n",
    ")\n",
    "\n",
    "tool_node = ToolNode(tools=tools)\n",
    "graph_builder.add_node(\"tools\", tool_node)\n",
    "\n",
    "graph_builder.add_conditional_edges(\n",
    "    \"agent\",\n",
    "    tools_condition,\n",
    ")\n",
    "graph_builder.add_edge(\"tools\", \"select_tools\")\n",
    "graph_builder.add_edge(\"select_tools\", \"agent\")\n",
    "graph_builder.add_edge(START, \"select_tools\")\n",
    "graph = graph_builder.compile()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 47,
   "id": "9110789a-843a-4c21-aeff-8841b24f7674",
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
   "cell_type": "code",
   "execution_count": 48,
   "id": "bee04c3d-0e36-4443-b0c8-10986a5f6e39",
   "metadata": {},
   "outputs": [],
   "source": [
    "user_input = \"Can you give me some information about AMD in 2022?\"\n",
    "\n",
    "result = graph.invoke({\"messages\": [(\"user\", user_input)]})"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 49,
   "id": "6906fb50-435c-4473-bbb6-5353433b9199",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "================================\u001b[1m Human Message \u001b[0m=================================\n",
      "\n",
      "Can you give me some information about AMD in 2022?\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "Tool Calls:\n",
      "  Accenture (call_qGmwFnENwwzHOYJXiCAaY5Mx)\n",
      " Call ID: call_qGmwFnENwwzHOYJXiCAaY5Mx\n",
      "  Args:\n",
      "    year: 2022\n",
      "=================================\u001b[1m Tool Message \u001b[0m=================================\n",
      "Name: Accenture\n",
      "\n",
      "Accenture had revenues of $100 in 2022.\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "Tool Calls:\n",
      "  Advanced_Micro_Devices (call_u9e5UIJtiieXVYi7Y9GgyDpn)\n",
      " Call ID: call_u9e5UIJtiieXVYi7Y9GgyDpn\n",
      "  Args:\n",
      "    year: 2022\n",
      "=================================\u001b[1m Tool Message \u001b[0m=================================\n",
      "Name: Advanced_Micro_Devices\n",
      "\n",
      "Advanced Micro Devices had revenues of $100 in 2022.\n",
      "==================================\u001b[1m Ai Message \u001b[0m==================================\n",
      "\n",
      "In 2022, AMD had revenues of $100.\n"
     ]
    }
   ],
   "source": [
    "for message in result[\"messages\"]:\n",
    "    message.pretty_print()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "177aedfa-cec5-45d0-82ad-efc0233aa6b4",
   "metadata": {},
   "source": [
    "## Next steps\n",
    "\n",
    "This guide provides a minimal implementation for dynamically selecting tools. There is a host of possible improvements and optimizations:\n",
    "\n",
    "- **Repeating tool selection**: Here, we repeated tool selection by modifying the `select_tools` node. Another option is to equip the agent with a `reselect_tools` tool, allowing it to re-select tools at its discretion.\n",
    "- **Optimizing tool selection**: In general, the full scope of [retrieval solutions](https://python.langchain.com/docs/concepts/#retrieval) are available for tool selection. Additional options include:\n",
    "  - Group tools and retrieve over groups;\n",
    "  - Use a chat model to select tools or groups of tool."
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
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/multi-agent-multi-turn-convo-functional.ipynb
```ipynb
{
 "cells": [
  {
   "attachments": {},
   "cell_type": "markdown",
   "id": "a2b182eb-1e31-43c8-85b1-706508dfa370",
   "metadata": {},
   "source": [
    "# How to add multi-turn conversation in a multi-agent application (functional API)\n",
    "\n",
    "!!! info \"Prerequisites\"\n",
    "    This guide assumes familiarity with the following:\n",
    "\n",
    "    - [Multi-agent systems](../../concepts/multi_agent)\n",
    "    - [Human-in-the-loop](../../concepts/human_in_the_loop)\n",
    "    - [Functional API](../../concepts/functional_api)\n",
    "    - [Command](../../concepts/low_level/#command)\n",
    "    - [LangGraph Glossary](../../concepts/low_level/)\n",
    "\n",
    "\n",
    "In this how-to guide, we’ll build an application that allows an end-user to engage in a *multi-turn conversation* with one or more agents. We'll create a node that uses an [`interrupt`](../../reference/types/#langgraph.types.interrupt) to collect user input and routes back to the **active** agent.\n",
    "\n",
    "The agents will be implemented as tasks in a workflow that executes agent steps and determines the next action:\n",
    "\n",
    "1. **Wait for user input** to continue the conversation, or\n",
    "2. **Route to another agent** (or back to itself, such as in a loop) via a [**handoff**](../../concepts/multi_agent/#handoffs).\n",
    "\n",
    "```python\n",
    "from langgraph.func import entrypoint, task\n",
    "from langgraph.prebuilt import create_react_agent\n",
    "from langchain_core.tools import tool\n",
    "from langgraph.types import interrupt\n",
    "\n",
    "\n",
    "# Define a tool to signal intent to hand off to a different agent\n",
    "# Note: this is not using Command(goto) syntax for navigating to different agents:\n",
    "# `workflow()` below handles the handoffs explicitly\n",
    "@tool(return_direct=True)\n",
    "def transfer_to_hotel_advisor():\n",
    "    \"\"\"Ask hotel advisor agent for help.\"\"\"\n",
    "    return \"Successfully transferred to hotel advisor\"\n",
    "\n",
    "\n",
    "# define an agent\n",
    "travel_advisor_tools = [transfer_to_hotel_advisor, ...]\n",
    "travel_advisor = create_react_agent(model, travel_advisor_tools)\n",
    "\n",
    "\n",
    "# define a task that calls an agent\n",
    "@task\n",
    "def call_travel_advisor(messages):\n",
    "    response = travel_advisor.invoke({\"messages\": messages})\n",
    "    return response[\"messages\"]\n",
    "\n",
    "\n",
    "# define the multi-agent network workflow\n",
    "@entrypoint(checkpointer)\n",
    "def workflow(messages):\n",
    "    call_active_agent = call_travel_advisor\n",
    "    while True:\n",
    "        agent_messages = call_active_agent(messages).result()\n",
    "        ai_msg = get_last_ai_msg(agent_messages)\n",
    "        if not ai_msg.tool_calls:\n",
    "            user_input = interrupt(value=\"Ready for user input.\")\n",
    "            messages = messages + [{\"role\": \"user\", \"content\": user_input}]\n",
    "            continue\n",
    "\n",
    "        messages = messages + agent_messages\n",
    "        call_active_agent = get_next_agent(messages)\n",
    "    return entrypoint.final(value=agent_messages[-1], save=messages)\n",
    "```"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "faaa4444-cd06-4813-b9ca-c9700fe12cb7",
   "metadata": {},
   "source": [
    "## Setup\n",
    "\n",
    "First, let's install the required packages"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 1,
   "id": "05038da0-31df-4066-a1a4-c4ccb5db4d3a",
   "metadata": {},
   "outputs": [],
   "source": [
    "# %%capture --no-stderr\n",
    "# %pip install -U langgraph langchain-anthropic"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 2,
   "id": "0bcff5d4-130e-426d-9285-40d0f72c7cd3",
   "metadata": {},
   "outputs": [
    {
     "name": "stdin",
     "output_type": "stream",
     "text": [
      "ANTHROPIC_API_KEY:  ········\n"
     ]
    }
   ],
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
   "id": "c3ec6e48-85dc-4905-ba50-985e5d4788e6",
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
   "attachments": {},
   "cell_type": "markdown",
   "id": "c217c3fe-ca50-45a1-be91-912bc83ed8b3",
   "metadata": {},
   "source": [
    "In this example we will build a team of travel assistant agents that can communicate with each other.\n",
    "\n",
    "We will create 2 agents:\n",
    "\n",
    "* `travel_advisor`: can help with travel destination recommendations. Can ask `hotel_advisor` for help.\n",
    "* `hotel_advisor`: can help with hotel recommendations. Can ask `travel_advisor` for help.\n",
    "\n",
    "This is a fully-connected network - every agent can talk to any other agent. "
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 3,
   "id": "eb51463a-4425-44ad-91d5-f21fd5b4e3b3",
   "metadata": {},
   "outputs": [],
   "source": [
    "import random\n",
    "from typing_extensions import Literal\n",
    "from langchain_core.tools import tool\n",
    "\n",
    "\n",
    "@tool\n",
    "def get_travel_recommendations():\n",
    "    \"\"\"Get recommendation for travel destinations\"\"\"\n",
    "    return random.choice([\"aruba\", \"turks and caicos\"])\n",
    "\n",
    "\n",
    "@tool\n",
    "def get_hotel_recommendations(location: Literal[\"aruba\", \"turks and caicos\"]):\n",
    "    \"\"\"Get hotel recommendations for a given destination.\"\"\"\n",
    "    return {\n",
    "        \"aruba\": [\n",
    "            \"The Ritz-Carlton, Aruba (Palm Beach)\"\n",
    "            \"Bucuti & Tara Beach Resort (Eagle Beach)\"\n",
    "        ],\n",
    "        \"turks and caicos\": [\"Grace Bay Club\", \"COMO Parrot Cay\"],\n",
    "    }[location]\n",
    "\n",
    "\n",
    "@tool(return_direct=True)\n",
    "def transfer_to_hotel_advisor():\n",
    "    \"\"\"Ask hotel advisor agent for help.\"\"\"\n",
    "    return \"Successfully transferred to hotel advisor\"\n",
    "\n",
    "\n",
    "@tool(return_direct=True)\n",
    "def transfer_to_travel_advisor():\n",
    "    \"\"\"Ask travel advisor agent for help.\"\"\"\n",
    "    return \"Successfully transferred to travel advisor\""
   ]
  },
  {
   "cell_type": "markdown",
   "id": "7f5b2a7f",
   "metadata": {},
   "source": [
    "!!! note \"Transfer tools\"\n",
    "\n",
    "    You might have noticed that we're using `@tool(return_direct=True)` in the transfer tools. This is done so that individual agents (e.g., `travel_advisor`) can exit the ReAct loop early once these tools are called. This is the desired behavior, as we want to detect when the agent calls this tool and hand control off _immediately_ to a different agent. \n",
    "    \n",
    "    **NOTE**: This is meant to work with the prebuilt [`create_react_agent`][langgraph.prebuilt.chat_agent_executor.create_react_agent] -- if you are building a custom agent, make sure to manually add logic for handling early exit for tools that are marked with `return_direct`."
   ]
  },
  {
   "cell_type": "markdown",
   "id": "213d661e-6ba4-42b9-bc7f-6c8c423e3419",
   "metadata": {},
   "source": [
    "Let's now create our agents using the prebuilt [`create_react_agent`][langgraph.prebuilt.chat_agent_executor.create_react_agent] and our multi-agent workflow. Note that will be calling [`interrupt`][langgraph.types.interrupt] every time after we get the final response from each of the agents."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 6,
   "id": "aa4bdbff-9461-46cc-aee9-8a22d3c3d9ec",
   "metadata": {},
   "outputs": [],
   "source": [
    "import uuid\n",
    "\n",
    "from langchain_core.messages import AIMessage\n",
    "from langchain_anthropic import ChatAnthropic\n",
    "from langgraph.prebuilt import create_react_agent\n",
    "from langgraph.graph import add_messages\n",
    "from langgraph.func import entrypoint, task\n",
    "from langgraph.checkpoint.memory import InMemorySaver\n",
    "from langgraph.types import interrupt, Command\n",
    "\n",
    "model = ChatAnthropic(model=\"claude-3-5-sonnet-latest\")\n",
    "\n",
    "# Define travel advisor ReAct agent\n",
    "travel_advisor_tools = [\n",
    "    get_travel_recommendations,\n",
    "    transfer_to_hotel_advisor,\n",
    "]\n",
    "travel_advisor = create_react_agent(\n",
    "    model,\n",
    "    travel_advisor_tools,\n",
    "    prompt=(\n",
    "        \"You are a general travel expert that can recommend travel destinations (e.g. countries, cities, etc). \"\n",
    "        \"If you need hotel recommendations, ask 'hotel_advisor' for help. \"\n",
    "        \"You MUST include human-readable response before transferring to another agent.\"\n",
    "    ),\n",
    ")\n",
    "\n",
    "\n",
    "@task\n",
    "def call_travel_advisor(messages):\n",
    "    # You can also add additional logic like changing the input to the agent / output from the agent, etc.\n",
    "    # NOTE: we're invoking the ReAct agent with the full history of messages in the state\n",
    "    response = travel_advisor.invoke({\"messages\": messages})\n",
    "    return response[\"messages\"]\n",
    "\n",
    "\n",
    "# Define hotel advisor ReAct agent\n",
    "hotel_advisor_tools = [get_hotel_recommendations, transfer_to_travel_advisor]\n",
    "hotel_advisor = create_react_agent(\n",
    "    model,\n",
    "    hotel_advisor_tools,\n",
    "    prompt=(\n",
    "        \"You are a hotel expert that can provide hotel recommendations for a given destination. \"\n",
    "        \"If you need help picking travel destinations, ask 'travel_advisor' for help.\"\n",
    "        \"You MUST include human-readable response before transferring to another agent.\"\n",
    "    ),\n",
    ")\n",
    "\n",
    "\n",
    "@task\n",
    "def call_hotel_advisor(messages):\n",
    "    response = hotel_advisor.invoke({\"messages\": messages})\n",
    "    return response[\"messages\"]\n",
    "\n",
    "\n",
    "checkpointer = InMemorySaver()\n",
    "\n",
    "\n",
    "def string_to_uuid(input_string):\n",
    "    return str(uuid.uuid5(uuid.NAMESPACE_URL, input_string))\n",
    "\n",
    "\n",
    "@entrypoint(checkpointer=checkpointer)\n",
    "def multi_turn_graph(messages, previous):\n",
    "    previous = previous or []\n",
    "    messages = add_messages(previous, messages)\n",
    "    call_active_agent = call_travel_advisor\n",
    "    while True:\n",
    "        agent_messages = call_active_agent(messages).result()\n",
    "        messages = add_messages(messages, agent_messages)\n",
    "        # Find the last AI message\n",
    "        # If one of the handoff tools is called, the last message returned\n",
    "        # by the agent will be a ToolMessage because we set them to have\n",
    "        # \"return_direct=True\". This means that the last AIMessage will\n",
    "        # have tool calls.\n",
    "        # Otherwise, the last returned message will be an AIMessage with\n",
    "        # no tool calls, which means we are ready for new input.\n",
    "        ai_msg = next(m for m in reversed(agent_messages) if isinstance(m, AIMessage))\n",
    "        if not ai_msg.tool_calls:\n",
    "            user_input = interrupt(value=\"Ready for user input.\")\n",
    "            # Add user input as a human message\n",
    "            # NOTE: we generate unique ID for the human message based on its content\n",
    "            # it's important, since on subsequent invocations previous user input (interrupt) values\n",
    "            # will be looked up again and we will attempt to add them again here\n",
    "            # `add_messages` deduplicates messages based on the ID, ensuring correct message history\n",
    "            human_message = {\n",
    "                \"role\": \"user\",\n",
    "                \"content\": user_input,\n",
    "                \"id\": string_to_uuid(user_input),\n",
    "            }\n",
    "            messages = add_messages(messages, [human_message])\n",
    "            continue\n",
    "\n",
    "        tool_call = ai_msg.tool_calls[-1]\n",
    "        if tool_call[\"name\"] == \"transfer_to_hotel_advisor\":\n",
    "            call_active_agent = call_hotel_advisor\n",
    "        elif tool_call[\"name\"] == \"transfer_to_travel_advisor\":\n",
    "            call_active_agent = call_travel_advisor\n",
    "        else:\n",
    "            raise ValueError(f\"Expected transfer tool, got '{tool_call['name']}'\")\n",
    "\n",
    "    return entrypoint.final(value=agent_messages[-1], save=messages)"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "af856e1b-41fc-4041-8cbf-3818a60088e0",
   "metadata": {},
   "source": [
    "## Test multi-turn conversation\n",
    "\n",
    "Let's test a multi turn conversation with this application."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 7,
   "id": "2b6fde57-86e3-440e-a7bf-f1e9b5ed9ff2",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "\n",
      "--- Conversation Turn 1 ---\n",
      "\n",
      "User: {'role': 'user', 'content': 'i wanna go somewhere warm in the caribbean', 'id': 'f48d82a7-7efa-43f5-ad4c-541758c95f61'}\n",
      "\n",
      "call_travel_advisor: Based on the recommendations, Aruba would be an excellent choice for your Caribbean getaway! Known as \"One Happy Island,\" Aruba offers:\n",
      "- Year-round warm weather with consistent temperatures around 82°F (28°C)\n",
      "- Beautiful white sand beaches like Eagle Beach and Palm Beach\n",
      "- Crystal clear waters perfect for swimming and snorkeling\n",
      "- Minimal rainfall and location outside the hurricane belt\n",
      "- Rich culture blending Dutch and Caribbean influences\n",
      "- Various activities from water sports to desert-like landscape exploration\n",
      "- Excellent dining and shopping options\n",
      "\n",
      "Would you like me to help you find suitable accommodations in Aruba? I can transfer you to our hotel advisor who can recommend specific hotels based on your preferences.\n",
      "\n",
      "--- Conversation Turn 2 ---\n",
      "\n",
      "User: Command(resume='could you recommend a nice hotel in one of the areas and tell me which area it is.')\n",
      "\n",
      "call_hotel_advisor: I can recommend two excellent options in different areas:\n",
      "\n",
      "1. The Ritz-Carlton, Aruba - Located in Palm Beach\n",
      "- Luxury beachfront resort\n",
      "- Located in the vibrant Palm Beach area, known for its lively atmosphere\n",
      "- Close to restaurants, shopping, and nightlife\n",
      "- Perfect for those who want a more active vacation with plenty of amenities nearby\n",
      "\n",
      "2. Bucuti & Tara Beach Resort - Located in Eagle Beach\n",
      "- Adults-only boutique resort\n",
      "- Situated on the quieter Eagle Beach\n",
      "- Known for its romantic atmosphere and excellent service\n",
      "- Ideal for couples seeking a more peaceful, intimate setting\n",
      "\n",
      "Would you like more specific information about either of these properties or their locations?\n",
      "\n",
      "--- Conversation Turn 3 ---\n",
      "\n",
      "User: Command(resume='i like the first one. could you recommend something to do near the hotel?')\n",
      "\n",
      "call_travel_advisor: Near The Ritz-Carlton in Palm Beach, here are some popular activities you can enjoy:\n",
      "\n",
      "1. Palm Beach Strip - Take a walk along this bustling strip filled with restaurants, shops, and bars\n",
      "2. Visit the Bubali Bird Sanctuary - Just a short distance away\n",
      "3. Try your luck at the Stellaris Casino - Located right in The Ritz-Carlton\n",
      "4. Water Sports at Palm Beach - Right in front of the hotel you can:\n",
      "   - Go parasailing\n",
      "   - Try jet skiing\n",
      "   - Take a sunset sailing cruise\n",
      "5. Visit the Palm Beach Plaza Mall - High-end shopping just a short walk away\n",
      "6. Enjoy dinner at Madame Janette's - One of Aruba's most famous restaurants nearby\n",
      "\n",
      "Would you like more specific information about any of these activities or other suggestions in the area?\n"
     ]
    }
   ],
   "source": [
    "thread_config = {\"configurable\": {\"thread_id\": uuid.uuid4()}}\n",
    "\n",
    "inputs = [\n",
    "    # 1st round of conversation,\n",
    "    {\n",
    "        \"role\": \"user\",\n",
    "        \"content\": \"i wanna go somewhere warm in the caribbean\",\n",
    "        \"id\": str(uuid.uuid4()),\n",
    "    },\n",
    "    # Since we're using `interrupt`, we'll need to resume using the Command primitive.\n",
    "    # 2nd round of conversation,\n",
    "    Command(\n",
    "        resume=\"could you recommend a nice hotel in one of the areas and tell me which area it is.\"\n",
    "    ),\n",
    "    # 3rd round of conversation,\n",
    "    Command(\n",
    "        resume=\"i like the first one. could you recommend something to do near the hotel?\"\n",
    "    ),\n",
    "]\n",
    "\n",
    "for idx, user_input in enumerate(inputs):\n",
    "    print()\n",
    "    print(f\"--- Conversation Turn {idx + 1} ---\")\n",
    "    print()\n",
    "    print(f\"User: {user_input}\")\n",
    "    print()\n",
    "    for update in multi_turn_graph.stream(\n",
    "        user_input,\n",
    "        config=thread_config,\n",
    "        stream_mode=\"updates\",\n",
    "    ):\n",
    "        for node_id, value in update.items():\n",
    "            if isinstance(value, list) and value:\n",
    "                last_message = value[-1]\n",
    "                if isinstance(last_message, dict) or last_message.type != \"ai\":\n",
    "                    continue\n",
    "                print(f\"{node_id}: {last_message.content}\")"
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
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/multi-agent-network-functional.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "87684b48-150e-4e15-b0a5-a9dd7851f8fb",
   "metadata": {},
   "source": [
    "# How to build a multi-agent network (functional API)"
   ]
  },
  {
   "attachments": {},
   "cell_type": "markdown",
   "id": "2c65639c-9705-49f1-840a-370718852e98",
   "metadata": {},
   "source": [
    "!!! info \"Prerequisites\" \n",
    "    This guide assumes familiarity with the following:\n",
    "\n",
    "    - [Multi-agent systems](../../concepts/multi_agent)\n",
    "    - [Functional API](../../concepts/functional_api)\n",
    "    - [Command](../../concepts/low_level/#command)\n",
    "    - [LangGraph Glossary](../../concepts/low_level/)\n",
    "\n",
    "In this how-to guide we will demonstrate how to implement a [multi-agent network](../../concepts/multi_agent#network) architecture where each agent can communicate with every other agent (many-to-many connections) and can decide which agent to call next. We will be using [functional API](../../concepts/functional_api) — individual agents will be defined as tasks and the agent handoffs will be defined in the main [entrypoint()][langgraph.func.entrypoint]:\n",
    "\n",
    "```python\n",
    "from langgraph.func import entrypoint\n",
    "from langgraph.prebuilt import create_react_agent\n",
    "from langchain_core.tools import tool\n",
    "\n",
    "\n",
    "# Define a tool to signal intent to hand off to a different agent\n",
    "@tool(return_direct=True)\n",
    "def transfer_to_hotel_advisor():\n",
    "    \"\"\"Ask hotel advisor agent for help.\"\"\"\n",
    "    return \"Successfully transferred to hotel advisor\"\n",
    "\n",
    "\n",
    "# define an agent\n",
    "travel_advisor_tools = [transfer_to_hotel_advisor, ...]\n",
    "travel_advisor = create_react_agent(model, travel_advisor_tools)\n",
    "\n",
    "\n",
    "# define a task that calls an agent\n",
    "@task\n",
    "def call_travel_advisor(messages):\n",
    "    response = travel_advisor.invoke({\"messages\": messages})\n",
    "    return response[\"messages\"]\n",
    "\n",
    "\n",
    "# define the multi-agent network workflow\n",
    "@entrypoint()\n",
    "def workflow(messages):\n",
    "    call_active_agent = call_travel_advisor\n",
    "    while True:\n",
    "        agent_messages = call_active_agent(messages).result()\n",
    "        messages = messages + agent_messages\n",
    "        call_active_agent = get_next_agent(messages)\n",
    "    return messages\n",
    "```"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "faaa4444-cd06-4813-b9ca-c9700fe12cb7",
   "metadata": {},
   "source": [
    "## Setup\n",
    "\n",
    "First, let's install the required packages"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 1,
   "id": "05038da0-31df-4066-a1a4-c4ccb5db4d3a",
   "metadata": {},
   "outputs": [],
   "source": [
    "%%capture --no-stderr\n",
    "%pip install -U langgraph langchain-anthropic"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 6,
   "id": "0bcff5d4-130e-426d-9285-40d0f72c7cd3",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "ANTHROPIC_API_KEY:  ········\n"
     ]
    }
   ],
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
   "id": "c3ec6e48-85dc-4905-ba50-985e5d4788e6",
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
   "id": "4a53f304-3709-4df7-8714-1ca61e615743",
   "metadata": {},
   "source": [
    "## Travel agent example"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "34cd131b-f0c2-4b69-887f-2cbd5afb14a7",
   "metadata": {},
   "source": [
    "In this example we will build a team of travel assistant agents that can communicate with each other.\n",
    "\n",
    "We will create 2 agents:\n",
    "\n",
    "* `travel_advisor`: can help with travel destination recommendations. Can ask `hotel_advisor` for help.\n",
    "* `hotel_advisor`: can help with hotel recommendations. Can ask `travel_advisor` for help.\n",
    "\n",
    "This is a fully-connected network - every agent can talk to any other agent. "
   ]
  },
  {
   "cell_type": "markdown",
   "id": "fedc9ed0-e90c-4ee1-a7c6-f5af3c634a7b",
   "metadata": {},
   "source": [
    "First, let's create some of the tools that the agents will be using:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 7,
   "id": "7e31f258-ec28-4020-b86d-c91dfa9a3bfc",
   "metadata": {},
   "outputs": [],
   "source": [
    "import random\n",
    "from typing_extensions import Literal\n",
    "from langchain_core.tools import tool\n",
    "\n",
    "\n",
    "@tool\n",
    "def get_travel_recommendations():\n",
    "    \"\"\"Get recommendation for travel destinations\"\"\"\n",
    "    return random.choice([\"aruba\", \"turks and caicos\"])\n",
    "\n",
    "\n",
    "@tool\n",
    "def get_hotel_recommendations(location: Literal[\"aruba\", \"turks and caicos\"]):\n",
    "    \"\"\"Get hotel recommendations for a given destination.\"\"\"\n",
    "    return {\n",
    "        \"aruba\": [\n",
    "            \"The Ritz-Carlton, Aruba (Palm Beach)\"\n",
    "            \"Bucuti & Tara Beach Resort (Eagle Beach)\"\n",
    "        ],\n",
    "        \"turks and caicos\": [\"Grace Bay Club\", \"COMO Parrot Cay\"],\n",
    "    }[location]\n",
    "\n",
    "\n",
    "@tool(return_direct=True)\n",
    "def transfer_to_hotel_advisor():\n",
    "    \"\"\"Ask hotel advisor agent for help.\"\"\"\n",
    "    return \"Successfully transferred to hotel advisor\"\n",
    "\n",
    "\n",
    "@tool(return_direct=True)\n",
    "def transfer_to_travel_advisor():\n",
    "    \"\"\"Ask travel advisor agent for help.\"\"\"\n",
    "    return \"Successfully transferred to travel advisor\""
   ]
  },
  {
   "cell_type": "markdown",
   "id": "d8519a32-d23b-48b0-bd18-74f8c0dacf58",
   "metadata": {},
   "source": [
    "!!! note \"Transfer tools\"\n",
    "\n",
    "    You might have noticed that we're using `@tool(return_direct=True)` in the transfer tools. This is done so that individual agents (e.g., `travel_advisor`) can exit the ReAct loop early once these tools are called. This is the desired behavior, as we want to detect when the agent calls this tool and hand control off _immediately_ to a different agent. \n",
    "    \n",
    "    **NOTE**: This is meant to work with the prebuilt [`create_react_agent`][langgraph.prebuilt.chat_agent_executor.create_react_agent] -- if you are building a custom agent, make sure to manually add logic for handling early exit for tools that are marked with `return_direct`."
   ]
  },
  {
   "cell_type": "markdown",
   "id": "93dbc3bd-27b9-4d79-b5dd-be592bc50f74",
   "metadata": {},
   "source": [
    "Now let's define our agent tasks and combine them into a single multi-agent network workflow:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 8,
   "id": "b638d6c4-3de6-4921-980c-2df1bd1cc9c7",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langchain_core.messages import AIMessage\n",
    "from langchain_anthropic import ChatAnthropic\n",
    "from langgraph.prebuilt import create_react_agent\n",
    "from langgraph.graph import add_messages\n",
    "from langgraph.func import entrypoint, task\n",
    "\n",
    "model = ChatAnthropic(model=\"claude-3-5-sonnet-latest\")\n",
    "\n",
    "# Define travel advisor ReAct agent\n",
    "travel_advisor_tools = [\n",
    "    get_travel_recommendations,\n",
    "    transfer_to_hotel_advisor,\n",
    "]\n",
    "travel_advisor = create_react_agent(\n",
    "    model,\n",
    "    travel_advisor_tools,\n",
    "    prompt=(\n",
    "        \"You are a general travel expert that can recommend travel destinations (e.g. countries, cities, etc). \"\n",
    "        \"If you need hotel recommendations, ask 'hotel_advisor' for help. \"\n",
    "        \"You MUST include human-readable response before transferring to another agent.\"\n",
    "    ),\n",
    ")\n",
    "\n",
    "\n",
    "@task\n",
    "def call_travel_advisor(messages):\n",
    "    # You can also add additional logic like changing the input to the agent / output from the agent, etc.\n",
    "    # NOTE: we're invoking the ReAct agent with the full history of messages in the state\n",
    "    response = travel_advisor.invoke({\"messages\": messages})\n",
    "    return response[\"messages\"]\n",
    "\n",
    "\n",
    "# Define hotel advisor ReAct agent\n",
    "hotel_advisor_tools = [get_hotel_recommendations, transfer_to_travel_advisor]\n",
    "hotel_advisor = create_react_agent(\n",
    "    model,\n",
    "    hotel_advisor_tools,\n",
    "    prompt=(\n",
    "        \"You are a hotel expert that can provide hotel recommendations for a given destination. \"\n",
    "        \"If you need help picking travel destinations, ask 'travel_advisor' for help.\"\n",
    "        \"You MUST include human-readable response before transferring to another agent.\"\n",
    "    ),\n",
    ")\n",
    "\n",
    "\n",
    "@task\n",
    "def call_hotel_advisor(messages):\n",
    "    response = hotel_advisor.invoke({\"messages\": messages})\n",
    "    return response[\"messages\"]\n",
    "\n",
    "\n",
    "@entrypoint()\n",
    "def workflow(messages):\n",
    "    messages = add_messages([], messages)\n",
    "\n",
    "    call_active_agent = call_travel_advisor\n",
    "    while True:\n",
    "        agent_messages = call_active_agent(messages).result()\n",
    "        messages = add_messages(messages, agent_messages)\n",
    "        ai_msg = next(m for m in reversed(agent_messages) if isinstance(m, AIMessage))\n",
    "        if not ai_msg.tool_calls:\n",
    "            break\n",
    "\n",
    "        tool_call = ai_msg.tool_calls[-1]\n",
    "        if tool_call[\"name\"] == \"transfer_to_travel_advisor\":\n",
    "            call_active_agent = call_travel_advisor\n",
    "        elif tool_call[\"name\"] == \"transfer_to_hotel_advisor\":\n",
    "            call_active_agent = call_hotel_advisor\n",
    "        else:\n",
    "            raise ValueError(f\"Expected transfer tool, got '{tool_call['name']}'\")\n",
    "\n",
    "    return messages"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "9223db83-1938-434a-9d24-8666842a8eea",
   "metadata": {},
   "source": [
    "Lastly, let's define a helper to render the agent outputs:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 9,
   "id": "058f3d96-534f-4b97-afb3-799ba81224ea",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langchain_core.messages import convert_to_messages\n",
    "\n",
    "\n",
    "def pretty_print_messages(update):\n",
    "    if isinstance(update, tuple):\n",
