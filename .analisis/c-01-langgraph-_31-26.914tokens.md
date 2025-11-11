    }
   },
   "cell_type": "markdown",
   "id": "16dc0e41-80bd-4453-b421-dcf315741bf4",
   "metadata": {},
   "source": [
    "# Code generation with RAG and self-correction\n",
    "\n",
    "AlphaCodium presented an approach for code generation that uses control flow.\n",
    "\n",
    "Main idea: [construct an answer to a coding question iteratively.](https://x.com/karpathy/status/1748043513156272416?s=20). \n",
    "\n",
    "[AlphaCodium](https://github.com/Codium-ai/AlphaCodium) iteravely tests and improves an answer on public and AI-generated tests for a particular question. \n",
    "\n",
    "We will implement some of these ideas from scratch using [LangGraph](https://langchain-ai.github.io/langgraph/):\n",
    "\n",
    "1. We start with a set of documentation specified by a user\n",
    "2. We use a long context LLM to ingest it and perform RAG to answer a question based upon it\n",
    "3. We will invoke a tool to produce a structured output\n",
    "4. We will perform two unit tests (check imports and code execution) prior returning the solution to the user \n",
    "\n",
    "![Screenshot 2024-05-23 at 2.17.42 PM.png](attachment:67b615fe-0c25-4410-9d58-835982547001.png)"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "95a34aa2",
   "metadata": {},
   "source": [
    "## Setup\n",
    "\n",
    "First, let's install our required packages and set the API keys we will need"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "e3900420",
   "metadata": {},
   "outputs": [],
   "source": [
    "! pip install -U langchain_community langchain-openai langchain-anthropic langchain langgraph bs4"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "602be48f",
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
    "_set_env(\"OPENAI_API_KEY\")\n",
    "_set_env(\"ANTHROPIC_API_KEY\")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "0963fd21",
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
   "id": "38330223-d8c8-4156-82b6-93e63343bc01",
   "metadata": {},
   "source": [
    "## Docs\n",
    "\n",
    "Load [LangChain Expression Language](https://python.langchain.com/docs/concepts/#langchain-expression-language-lcel) (LCEL) docs as an example."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 1,
   "id": "c2eb35d1-4990-47dc-a5c4-208bae588a82",
   "metadata": {},
   "outputs": [],
   "source": [
    "from bs4 import BeautifulSoup as Soup\n",
    "from langchain_community.document_loaders.recursive_url_loader import RecursiveUrlLoader\n",
    "\n",
    "# LCEL docs\n",
    "url = \"https://python.langchain.com/docs/concepts/lcel/\"\n",
    "loader = RecursiveUrlLoader(\n",
    "    url=url, max_depth=20, extractor=lambda x: Soup(x, \"html.parser\").text\n",
    ")\n",
    "docs = loader.load()\n",
    "\n",
    "# Sort the list based on the URLs and get the text\n",
    "d_sorted = sorted(docs, key=lambda x: x.metadata[\"source\"])\n",
    "d_reversed = list(reversed(d_sorted))\n",
    "concatenated_content = \"\\n\\n\\n --- \\n\\n\\n\".join(\n",
    "    [doc.page_content for doc in d_reversed]\n",
    ")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "662d4ff4-1709-412f-bfed-5eb2b8d3d3dc",
   "metadata": {},
   "source": [
    "## LLMs\n",
    "\n",
    "### Code solution\n",
    "\n",
    "First, we will try OpenAI and [Claude3](https://python.langchain.com/docs/integrations/providers/anthropic/) with function calling.\n",
    "\n",
    "We will create a `code_gen_chain` w/ either OpenAI or Claude and test them here."
   ]
  },
  {
   "cell_type": "markdown",
   "id": "95944645-35bc-4798-8a27-c262b245a74c",
   "metadata": {},
   "source": [
    "<div class=\"admonition note\">\n",
    "    <p class=\"admonition-title\">Using Pydantic with LangChain</p>\n",
    "    <p>\n",
    "        This notebook uses Pydantic v2 <code>BaseModel</code>, which requires <code>langchain-core >= 0.3</code>. Using <code>langchain-core < 0.3</code> will result in errors due to mixing of Pydantic v1 and v2 <code>BaseModels</code>.\n",
    "    </p>\n",
    "</div>"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 5,
   "id": "3ba3df70-f6b4-4ea5-a210-e10944960bc6",
   "metadata": {},
   "outputs": [
    {
     "data": {
      "text/plain": [
       "code(prefix='To build a Retrieval-Augmented Generation (RAG) chain in LCEL, you will need to set up a chain that combines a retriever and a language model (LLM). The retriever will fetch relevant documents based on a query, and the LLM will generate a response using the retrieved documents as context. Here’s how you can do it:', imports='from langchain_core.prompts import ChatPromptTemplate\\nfrom langchain_openai import ChatOpenAI\\nfrom langchain_core.output_parsers import StrOutputParser\\nfrom langchain_core.retrievers import MyRetriever', code='# Define the retriever\\nretriever = MyRetriever()  # Replace with your specific retriever implementation\\n\\n# Define the LLM model\\nmodel = ChatOpenAI(model=\"gpt-4\")\\n\\n# Create a prompt template for the LLM\\nprompt_template = ChatPromptTemplate.from_template(\"Given the following documents, answer the question: {question}\\nDocuments: {documents}\")\\n\\n# Create the RAG chain\\nrag_chain = prompt_template | retriever | model | StrOutputParser()\\n\\n# Example usage\\nquery = \"What are the benefits of using RAG?\"\\nresponse = rag_chain.invoke({\"question\": query})\\nprint(response)')"
      ]
     },
     "execution_count": 5,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "from langchain_core.prompts import ChatPromptTemplate\n",
    "from langchain_openai import ChatOpenAI\n",
    "from pydantic import BaseModel, Field\n",
    "\n",
    "### OpenAI\n",
    "\n",
    "# Grader prompt\n",
    "code_gen_prompt = ChatPromptTemplate.from_messages(\n",
    "    [\n",
    "        (\n",
    "            \"system\",\n",
    "            \"\"\"You are a coding assistant with expertise in LCEL, LangChain expression language. \\n \n",
    "    Here is a full set of LCEL documentation:  \\n ------- \\n  {context} \\n ------- \\n Answer the user \n",
    "    question based on the above provided documentation. Ensure any code you provide can be executed \\n \n",
    "    with all required imports and variables defined. Structure your answer with a description of the code solution. \\n\n",
    "    Then list the imports. And finally list the functioning code block. Here is the user question:\"\"\",\n",
    "        ),\n",
    "        (\"placeholder\", \"{messages}\"),\n",
    "    ]\n",
    ")\n",
    "\n",
    "\n",
    "# Data model\n",
    "class code(BaseModel):\n",
    "    \"\"\"Schema for code solutions to questions about LCEL.\"\"\"\n",
    "\n",
    "    prefix: str = Field(description=\"Description of the problem and approach\")\n",
    "    imports: str = Field(description=\"Code block import statements\")\n",
    "    code: str = Field(description=\"Code block not including import statements\")\n",
    "\n",
    "\n",
    "expt_llm = \"gpt-4o-mini\"\n",
    "llm = ChatOpenAI(temperature=0, model=expt_llm)\n",
    "code_gen_chain_oai = code_gen_prompt | llm.with_structured_output(code)\n",
    "question = \"How do I build a RAG chain in LCEL?\"\n",
    "solution = code_gen_chain_oai.invoke(\n",
    "    {\"context\": concatenated_content, \"messages\": [(\"user\", question)]}\n",
    ")\n",
    "solution"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 6,
   "id": "cd30b67d-96db-4e51-a540-ae23fcc1f878",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langchain_anthropic import ChatAnthropic\n",
    "from langchain_core.prompts import ChatPromptTemplate\n",
    "\n",
    "### Anthropic\n",
    "\n",
    "# Prompt to enforce tool use\n",
    "code_gen_prompt_claude = ChatPromptTemplate.from_messages(\n",
    "    [\n",
    "        (\n",
    "            \"system\",\n",
    "            \"\"\"<instructions> You are a coding assistant with expertise in LCEL, LangChain expression language. \\n \n",
    "    Here is the LCEL documentation:  \\n ------- \\n  {context} \\n ------- \\n Answer the user  question based on the \\n \n",
    "    above provided documentation. Ensure any code you provide can be executed with all required imports and variables \\n\n",
    "    defined. Structure your answer: 1) a prefix describing the code solution, 2) the imports, 3) the functioning code block. \\n\n",
    "    Invoke the code tool to structure the output correctly. </instructions> \\n Here is the user question:\"\"\",\n",
    "        ),\n",
    "        (\"placeholder\", \"{messages}\"),\n",
    "    ]\n",
    ")\n",
    "\n",
    "\n",
    "# LLM\n",
    "expt_llm = \"claude-3-opus-20240229\"\n",
    "llm = ChatAnthropic(\n",
    "    model=expt_llm,\n",
    "    default_headers={\"anthropic-beta\": \"tools-2024-04-04\"},\n",
    ")\n",
    "\n",
    "structured_llm_claude = llm.with_structured_output(code, include_raw=True)\n",
    "\n",
    "\n",
    "# Optional: Check for errors in case tool use is flaky\n",
    "def check_claude_output(tool_output):\n",
    "    \"\"\"Check for parse error or failure to call the tool\"\"\"\n",
    "\n",
    "    # Error with parsing\n",
    "    if tool_output[\"parsing_error\"]:\n",
    "        # Report back output and parsing errors\n",
    "        print(\"Parsing error!\")\n",
    "        raw_output = str(tool_output[\"raw\"].content)\n",
    "        error = tool_output[\"parsing_error\"]\n",
    "        raise ValueError(\n",
    "            f\"Error parsing your output! Be sure to invoke the tool. Output: {raw_output}. \\n Parse error: {error}\"\n",
    "        )\n",
    "\n",
    "    # Tool was not invoked\n",
    "    elif not tool_output[\"parsed\"]:\n",
    "        print(\"Failed to invoke tool!\")\n",
    "        raise ValueError(\n",
    "            \"You did not use the provided tool! Be sure to invoke the tool to structure the output.\"\n",
    "        )\n",
    "    return tool_output\n",
    "\n",
    "\n",
    "# Chain with output check\n",
    "code_chain_claude_raw = (\n",
    "    code_gen_prompt_claude | structured_llm_claude | check_claude_output\n",
    ")\n",
    "\n",
    "\n",
    "def insert_errors(inputs):\n",
    "    \"\"\"Insert errors for tool parsing in the messages\"\"\"\n",
    "\n",
    "    # Get errors\n",
    "    error = inputs[\"error\"]\n",
    "    messages = inputs[\"messages\"]\n",
    "    messages += [\n",
    "        (\n",
    "            \"assistant\",\n",
    "            f\"Retry. You are required to fix the parsing errors: {error} \\n\\n You must invoke the provided tool.\",\n",
    "        )\n",
    "    ]\n",
    "    return {\n",
    "        \"messages\": messages,\n",
    "        \"context\": inputs[\"context\"],\n",
    "    }\n",
    "\n",
    "\n",
    "# This will be run as a fallback chain\n",
    "fallback_chain = insert_errors | code_chain_claude_raw\n",
    "N = 3  # Max re-tries\n",
    "code_gen_chain_re_try = code_chain_claude_raw.with_fallbacks(\n",
    "    fallbacks=[fallback_chain] * N, exception_key=\"error\"\n",
    ")\n",
    "\n",
    "\n",
    "def parse_output(solution):\n",
    "    \"\"\"When we add 'include_raw=True' to structured output,\n",
    "    it will return a dict w 'raw', 'parsed', 'parsing_error'.\"\"\"\n",
    "\n",
    "    return solution[\"parsed\"]\n",
    "\n",
    "\n",
    "# Optional: With re-try to correct for failure to invoke tool\n",
    "code_gen_chain = code_gen_chain_re_try | parse_output\n",
    "\n",
    "# No re-try\n",
    "code_gen_chain = code_gen_prompt_claude | structured_llm_claude | parse_output"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 7,
   "id": "9f14750f-dddc-485b-ba29-5392cdf4ba43",
   "metadata": {
    "scrolled": true
   },
   "outputs": [
    {
     "data": {
      "text/plain": [
       "code(prefix=\"To build a RAG (Retrieval Augmented Generation) chain in LCEL, you can use a retriever to fetch relevant documents and then pass those documents to a chat model to generate a response based on the retrieved context. Here's an example of how to do this:\", imports='from langchain_expressions import retrieve, chat_completion', code='question = \"What is the capital of France?\"\\n\\nrelevant_docs = retrieve(question)\\n\\nresult = chat_completion(\\n    model=\\'openai-gpt35\\', \\n    messages=[\\n        {{{\"role\": \"system\", \"content\": \"Answer the question based on the retrieved context.}}},\\n        {{{\"role\": \"user\", \"content\": \\'\\'\\'\\n            Context: {relevant_docs}\\n            Question: {question}\\n        \\'\\'\\'}}\\n    ]\\n)\\n\\nprint(result)')"
      ]
     },
     "execution_count": 7,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "# Test\n",
    "question = \"How do I build a RAG chain in LCEL?\"\n",
    "solution = code_gen_chain.invoke(\n",
    "    {\"context\": concatenated_content, \"messages\": [(\"user\", question)]}\n",
    ")\n",
    "solution"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "131f2055-2f64-4d19-a3d1-2d3cb8b42894",
   "metadata": {},
   "source": [
    "## State \n",
    "\n",
    "Our state is a dict that will contain keys (errors, question, code generation) relevant to code generation."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 8,
   "id": "c185f1a2-e943-4bed-b833-4243c9c64092",
   "metadata": {},
   "outputs": [],
   "source": [
    "from typing import List\n",
    "from typing_extensions import TypedDict\n",
    "\n",
    "\n",
    "class GraphState(TypedDict):\n",
    "    \"\"\"\n",
    "    Represents the state of our graph.\n",
    "\n",
    "    Attributes:\n",
    "        error : Binary flag for control flow to indicate whether test error was tripped\n",
    "        messages : With user question, error messages, reasoning\n",
    "        generation : Code solution\n",
    "        iterations : Number of tries\n",
    "    \"\"\"\n",
    "\n",
    "    error: str\n",
    "    messages: List\n",
    "    generation: str\n",
    "    iterations: int"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "64454465-26a3-40de-ad85-bcf59a2c3086",
   "metadata": {},
   "source": [
    "## Graph \n",
    "\n",
    "Our graph lays out the logical flow shown in the figure above."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 9,
   "id": "b70e8301-63ae-4f7e-ad8f-c9a052fe3566",
   "metadata": {},
   "outputs": [],
   "source": [
    "### Parameter\n",
    "\n",
    "# Max tries\n",
    "max_iterations = 3\n",
    "# Reflect\n",
    "# flag = 'reflect'\n",
    "flag = \"do not reflect\"\n",
    "\n",
    "### Nodes\n",
    "\n",
    "\n",
    "def generate(state: GraphState):\n",
    "    \"\"\"\n",
    "    Generate a code solution\n",
    "\n",
    "    Args:\n",
    "        state (dict): The current graph state\n",
    "\n",
    "    Returns:\n",
    "        state (dict): New key added to state, generation\n",
    "    \"\"\"\n",
    "\n",
    "    print(\"---GENERATING CODE SOLUTION---\")\n",
    "\n",
    "    # State\n",
    "    messages = state[\"messages\"]\n",
    "    iterations = state[\"iterations\"]\n",
    "    error = state[\"error\"]\n",
    "\n",
    "    # We have been routed back to generation with an error\n",
    "    if error == \"yes\":\n",
    "        messages += [\n",
    "            (\n",
    "                \"user\",\n",
    "                \"Now, try again. Invoke the code tool to structure the output with a prefix, imports, and code block:\",\n",
    "            )\n",
    "        ]\n",
    "\n",
    "    # Solution\n",
    "    code_solution = code_gen_chain.invoke(\n",
    "        {\"context\": concatenated_content, \"messages\": messages}\n",
    "    )\n",
    "    messages += [\n",
    "        (\n",
    "            \"assistant\",\n",
    "            f\"{code_solution.prefix} \\n Imports: {code_solution.imports} \\n Code: {code_solution.code}\",\n",
    "        )\n",
    "    ]\n",
    "\n",
    "    # Increment\n",
    "    iterations = iterations + 1\n",
    "    return {\"generation\": code_solution, \"messages\": messages, \"iterations\": iterations}\n",
    "\n",
    "\n",
    "def code_check(state: GraphState):\n",
    "    \"\"\"\n",
    "    Check code\n",
    "\n",
    "    Args:\n",
    "        state (dict): The current graph state\n",
    "\n",
    "    Returns:\n",
    "        state (dict): New key added to state, error\n",
    "    \"\"\"\n",
    "\n",
    "    print(\"---CHECKING CODE---\")\n",
    "\n",
    "    # State\n",
    "    messages = state[\"messages\"]\n",
    "    code_solution = state[\"generation\"]\n",
    "    iterations = state[\"iterations\"]\n",
    "\n",
    "    # Get solution components\n",
    "    imports = code_solution.imports\n",
    "    code = code_solution.code\n",
    "\n",
    "    # Check imports\n",
    "    try:\n",
    "        exec(imports)\n",
    "    except Exception as e:\n",
    "        print(\"---CODE IMPORT CHECK: FAILED---\")\n",
    "        error_message = [(\"user\", f\"Your solution failed the import test: {e}\")]\n",
    "        messages += error_message\n",
    "        return {\n",
    "            \"generation\": code_solution,\n",
    "            \"messages\": messages,\n",
    "            \"iterations\": iterations,\n",
    "            \"error\": \"yes\",\n",
    "        }\n",
    "\n",
    "    # Check execution\n",
    "    try:\n",
    "        exec(imports + \"\\n\" + code)\n",
    "    except Exception as e:\n",
    "        print(\"---CODE BLOCK CHECK: FAILED---\")\n",
    "        error_message = [(\"user\", f\"Your solution failed the code execution test: {e}\")]\n",
    "        messages += error_message\n",
    "        return {\n",
    "            \"generation\": code_solution,\n",
    "            \"messages\": messages,\n",
    "            \"iterations\": iterations,\n",
    "            \"error\": \"yes\",\n",
    "        }\n",
    "\n",
    "    # No errors\n",
    "    print(\"---NO CODE TEST FAILURES---\")\n",
    "    return {\n",
    "        \"generation\": code_solution,\n",
    "        \"messages\": messages,\n",
    "        \"iterations\": iterations,\n",
    "        \"error\": \"no\",\n",
    "    }\n",
    "\n",
    "\n",
    "def reflect(state: GraphState):\n",
    "    \"\"\"\n",
    "    Reflect on errors\n",
    "\n",
    "    Args:\n",
    "        state (dict): The current graph state\n",
    "\n",
    "    Returns:\n",
    "        state (dict): New key added to state, generation\n",
    "    \"\"\"\n",
    "\n",
    "    print(\"---GENERATING CODE SOLUTION---\")\n",
    "\n",
    "    # State\n",
    "    messages = state[\"messages\"]\n",
    "    iterations = state[\"iterations\"]\n",
    "    code_solution = state[\"generation\"]\n",
    "\n",
    "    # Prompt reflection\n",
    "\n",
    "    # Add reflection\n",
    "    reflections = code_gen_chain.invoke(\n",
    "        {\"context\": concatenated_content, \"messages\": messages}\n",
    "    )\n",
    "    messages += [(\"assistant\", f\"Here are reflections on the error: {reflections}\")]\n",
    "    return {\"generation\": code_solution, \"messages\": messages, \"iterations\": iterations}\n",
    "\n",
    "\n",
    "### Edges\n",
    "\n",
    "\n",
    "def decide_to_finish(state: GraphState):\n",
    "    \"\"\"\n",
    "    Determines whether to finish.\n",
    "\n",
    "    Args:\n",
    "        state (dict): The current graph state\n",
    "\n",
    "    Returns:\n",
    "        str: Next node to call\n",
    "    \"\"\"\n",
    "    error = state[\"error\"]\n",
    "    iterations = state[\"iterations\"]\n",
    "\n",
    "    if error == \"no\" or iterations == max_iterations:\n",
    "        print(\"---DECISION: FINISH---\")\n",
    "        return \"end\"\n",
    "    else:\n",
    "        print(\"---DECISION: RE-TRY SOLUTION---\")\n",
    "        if flag == \"reflect\":\n",
    "            return \"reflect\"\n",
    "        else:\n",
    "            return \"generate\""
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 10,
   "id": "f66b4e00-4731-42c8-bc38-72dd0ff7c92c",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langgraph.graph import END, StateGraph, START\n",
    "\n",
    "workflow = StateGraph(GraphState)\n",
    "\n",
    "# Define the nodes\n",
    "workflow.add_node(\"generate\", generate)  # generation solution\n",
    "workflow.add_node(\"check_code\", code_check)  # check code\n",
    "workflow.add_node(\"reflect\", reflect)  # reflect\n",
    "\n",
    "# Build graph\n",
    "workflow.add_edge(START, \"generate\")\n",
    "workflow.add_edge(\"generate\", \"check_code\")\n",
    "workflow.add_conditional_edges(\n",
    "    \"check_code\",\n",
    "    decide_to_finish,\n",
    "    {\n",
    "        \"end\": END,\n",
    "        \"reflect\": \"reflect\",\n",
    "        \"generate\": \"generate\",\n",
    "    },\n",
    ")\n",
    "workflow.add_edge(\"reflect\", \"generate\")\n",
    "app = workflow.compile()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 13,
   "id": "9bcaafe4-ddcf-4fab-8620-2d9b6c508f98",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "---GENERATING CODE SOLUTION---\n",
      "---CHECKING CODE---\n",
      "---CODE IMPORT CHECK: FAILED---\n",
      "---DECISION: RE-TRY SOLUTION---\n",
      "---GENERATING CODE SOLUTION---\n",
      "---CHECKING CODE---\n",
      "---CODE IMPORT CHECK: FAILED---\n",
      "---DECISION: RE-TRY SOLUTION---\n",
      "---GENERATING CODE SOLUTION---\n",
      "---CHECKING CODE---\n",
      "---CODE BLOCK CHECK: FAILED---\n",
      "---DECISION: FINISH---\n"
     ]
    }
   ],
   "source": [
    "question = \"How can I directly pass a string to a runnable and use it to construct the input needed for my prompt?\"\n",
    "solution = app.invoke({\"messages\": [(\"user\", question)], \"iterations\": 0, \"error\": \"\"})"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 18,
   "id": "9d28692e",
   "metadata": {},
   "outputs": [
    {
     "data": {
      "text/plain": [
       "code(prefix='To directly pass a string to a runnable and use it to construct the input needed for a prompt, you can use the `_from_value` method on a PromptTemplate in LCEL. Create a PromptTemplate with the desired template string, then call `_from_value` on it with a dictionary mapping the input variable names to their values. This will return a PromptValue that you can pass directly to any chain or model that accepts a prompt input.', imports='from langchain_core.prompts import PromptTemplate', code='user_string = \"langchain is awesome\"\\n\\nprompt_template = PromptTemplate.from_template(\"Tell me more about how {user_input}.\")\\n\\nprompt_value = prompt_template._from_value({\"user_input\": user_string})\\n\\n# Pass the PromptValue directly to a model or chain \\nchain.run(prompt_value)')"
      ]
     },
     "execution_count": 18,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "solution[\"generation\"]"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "744f48a5-9ad3-4342-899f-7dd4266a9a15",
   "metadata": {},
   "source": [
    "## Eval"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "89852874-b538-4c8d-a4c3-1d68302db492",
   "metadata": {},
   "source": [
    "[Here](https://smith.langchain.com/public/326674a6-62bd-462d-88ae-eea49d503f9d/d) is a public dataset of LCEL questions. \n",
    "\n",
    "I saved this as `lcel-teacher-eval`.\n",
    "\n",
    "You can also find the csv [here](https://github.com/langchain-ai/lcel-teacher/blob/main/eval/eval.csv)."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 19,
   "id": "678e8954-56b5-4cc6-be26-f7f2a060b242",
   "metadata": {},
   "outputs": [],
   "source": [
    "import langsmith\n",
    "\n",
    "client = langsmith.Client()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 20,
   "id": "ef7cf662-7a6f-4dee-965c-6309d4045feb",
   "metadata": {},
   "outputs": [
    {
     "data": {
      "text/plain": [
       "Dataset(name='lcel-teacher-eval', description='Eval set for LCEL teacher', data_type=<DataType.kv: 'kv'>, id=UUID('8b57696d-14ea-4f00-9997-b3fc74a16846'), created_at=datetime.datetime(2024, 9, 16, 22, 50, 4, 169288, tzinfo=datetime.timezone.utc), modified_at=datetime.datetime(2024, 9, 16, 22, 50, 4, 169288, tzinfo=datetime.timezone.utc), example_count=0, session_count=0, last_session_start_time=None, inputs_schema=None, outputs_schema=None)"
      ]
     },
     "execution_count": 20,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "# Clone the dataset to your tenant to use it\n",
    "try:\n",
    "    public_dataset = (\n",
    "        \"https://smith.langchain.com/public/326674a6-62bd-462d-88ae-eea49d503f9d/d\"\n",
    "    )\n",
    "    client.clone_public_dataset(public_dataset)\n",
    "except:\n",
    "    print(\"Please setup LangSmith\")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "9d171396-022b-47ec-a741-c782aff9fdae",
   "metadata": {},
   "source": [
    "Custom evals."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 21,
   "id": "455a34ea-52cb-4ae5-9f4a-7e4a08cd0c09",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langsmith.schemas import Example, Run\n",
    "\n",
    "\n",
    "def check_import(run: Run, example: Example) -> dict:\n",
    "    imports = run.outputs.get(\"imports\")\n",
    "    try:\n",
    "        exec(imports)\n",
    "        return {\"key\": \"import_check\", \"score\": 1}\n",
    "    except Exception:\n",
    "        return {\"key\": \"import_check\", \"score\": 0}\n",
    "\n",
    "\n",
    "def check_execution(run: Run, example: Example) -> dict:\n",
    "    imports = run.outputs.get(\"imports\")\n",
    "    code = run.outputs.get(\"code\")\n",
    "    try:\n",
    "        exec(imports + \"\\n\" + code)\n",
    "        return {\"key\": \"code_execution_check\", \"score\": 1}\n",
    "    except Exception:\n",
    "        return {\"key\": \"code_execution_check\", \"score\": 0}"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "c90bf261-0d94-4779-bbde-c76adeefe3d7",
   "metadata": {},
   "source": [
    "Compare LangGraph to Context Stuffing."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 33,
   "id": "c8fa6bcb-b245-4422-b79a-582cd8a7d7ea",
   "metadata": {},
   "outputs": [],
   "source": [
    "def predict_base_case(example: dict):\n",
    "    \"\"\"Context stuffing\"\"\"\n",
    "    solution = code_gen_chain.invoke(\n",
    "        {\"context\": concatenated_content, \"messages\": [(\"user\", example[\"question\"])]}\n",
    "    )\n",
    "    return {\"imports\": solution.imports, \"code\": solution.code}\n",
    "\n",
    "\n",
    "def predict_langgraph(example: dict):\n",
    "    \"\"\"LangGraph\"\"\"\n",
    "    graph = app.invoke(\n",
    "        {\"messages\": [(\"user\", example[\"question\"])], \"iterations\": 0, \"error\": \"\"}\n",
    "    )\n",
    "    solution = graph[\"generation\"]\n",
    "    return {\"imports\": solution.imports, \"code\": solution.code}"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 34,
   "id": "d9c57468-97f6-47d6-a5e9-c09b53bfdd83",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langsmith.evaluation import evaluate\n",
    "\n",
    "# Evaluator\n",
    "code_evalulator = [check_import, check_execution]\n",
    "\n",
    "# Dataset\n",
    "dataset_name = \"lcel-teacher-eval\""
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "2dacccf0-d73f-4017-aaf0-9806ffe5bd2c",
   "metadata": {},
   "outputs": [],
   "source": [
    "# Run base case\n",
    "try:\n",
    "    experiment_results_ = evaluate(\n",
    "        predict_base_case,\n",
    "        data=dataset_name,\n",
    "        evaluators=code_evalulator,\n",
    "        experiment_prefix=f\"test-without-langgraph-{expt_llm}\",\n",
    "        max_concurrency=2,\n",
    "        metadata={\n",
    "            \"llm\": expt_llm,\n",
    "        },\n",
    "    )\n",
    "except:\n",
    "    print(\"Please setup LangSmith\")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "71d90f9e-9dad-410c-a709-093d275029ae",
   "metadata": {},
   "outputs": [],
   "source": [
    "# Run with langgraph\n",
    "try:\n",
    "    experiment_results = evaluate(\n",
    "        predict_langgraph,\n",
    "        data=dataset_name,\n",
    "        evaluators=code_evalulator,\n",
    "        experiment_prefix=f\"test-with-langgraph-{expt_llm}-{flag}\",\n",
    "        max_concurrency=2,\n",
    "        metadata={\n",
    "            \"llm\": expt_llm,\n",
    "            \"feedback\": flag,\n",
    "        },\n",
    "    )\n",
    "except:\n",
    "    print(\"Please setup LangSmith\")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "d69da747-b4ea-455d-9314-60c3d9d30549",
   "metadata": {},
   "source": [
    "`Results:`\n",
    "\n",
    "* `LangGraph outperforms base case`: adding re-try loop improve performance\n",
    "* `Reflection did not help`: reflection prior to re-try regression vs just passing errors directly back to the LLM\n",
    "* `GPT-4 outperforms Claude3`: Claude3 had 3 and 1 run fail due to tool-use error for Opus and Haiku, respectively\n",
    "\n",
    "https://smith.langchain.com/public/78a3d858-c811-4e46-91cb-0f10ef56260b/d"
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
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/sql/output.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/sql/sql-agent.md
```md
# Build a SQL agent

In this tutorial, we will walk through how to build an agent that can answer questions about a SQL database.

At a high level, the agent will:

1. Fetch the available tables from the database
2. Decide which tables are relevant to the question
3. Fetch the schemas for the relevant tables
4. Generate a query based on the question and information from the schemas
5. Double-check the query for common mistakes using an LLM
6. Execute the query and return the results
7. Correct mistakes surfaced by the database engine until the query is successful
8. Formulate a response based on the results

!!! warning "Security note"
    Building Q&A systems of SQL databases requires executing model-generated SQL queries. There are inherent risks in doing this. Make sure that your database connection permissions are always scoped as narrowly as possible for your agent's needs. This will mitigate though not eliminate the risks of building a model-driven system.

## 1. Setup

Let's first install some dependencies. This tutorial uses SQL database and tool abstractions from [langchain-community](https://python.langchain.com/docs/concepts/architecture/#langchain-community). We will also require a LangChain [chat model](https://python.langchain.com/docs/concepts/chat_models/).

```python
%%capture --no-stderr
%pip install -U langgraph langchain_community "langchain[openai]"
```

!!! tip
    Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. [LangSmith](https://docs.smith.langchain.com) lets you use trace data to debug, test, and monitor your LLM apps built with LangGraph.

### Select a LLM

First we [initialize our LLM](https://python.langchain.com/docs/how_to/chat_models_universal_init/). Any model supporting [tool-calling](https://python.langchain.com/docs/integrations/chat/#featured-providers) should work. We use OpenAI below.

```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("openai:gpt-4.1")
```

### Configure the database

We will be creating a SQLite database for this tutorial. SQLite is a lightweight database that is easy to set up and use. We will be loading the `chinook` database, which is a sample database that represents a digital media store.
Find more information about the database [here](https://www.sqlitetutorial.net/sqlite-sample-database/).

For convenience, we have hosted the database (`Chinook.db`) on a public GCS bucket.

```python
import requests

url = "https://storage.googleapis.com/benchmarks-artifacts/chinook/Chinook.db"

response = requests.get(url)

if response.status_code == 200:
    # Open a local file in binary write mode
    with open("Chinook.db", "wb") as file:
        # Write the content of the response (the file) to the local file
        file.write(response.content)
    print("File downloaded and saved as Chinook.db")
else:
    print(f"Failed to download the file. Status code: {response.status_code}")
```

We will use a handy SQL database wrapper available in the `langchain_community` package to interact with the database. The wrapper provides a simple interface to execute SQL queries and fetch results:

```python
from langchain_community.utilities import SQLDatabase

db = SQLDatabase.from_uri("sqlite:///Chinook.db")

print(f"Dialect: {db.dialect}")
print(f"Available tables: {db.get_usable_table_names()}")
print(f'Sample output: {db.run("SELECT * FROM Artist LIMIT 5;")}')
```

**Output:**
```
Dialect: sqlite
Available tables: ['Album', 'Artist', 'Customer', 'Employee', 'Genre', 'Invoice', 'InvoiceLine', 'MediaType', 'Playlist', 'PlaylistTrack', 'Track']
Sample output: [(1, 'AC/DC'), (2, 'Accept'), (3, 'Aerosmith'), (4, 'Alanis Morissette'), (5, 'Alice In Chains')]
```

### Tools for database interactions

`langchain-community` implements some built-in tools for interacting with our `SQLDatabase`, including tools for listing tables, reading table schemas, and checking and running queries:

```python
from langchain_community.agent_toolkits import SQLDatabaseToolkit

toolkit = SQLDatabaseToolkit(db=db, llm=llm)

tools = toolkit.get_tools()

for tool in tools:
    print(f"{tool.name}: {tool.description}\n")
```

**Output:**
```
sql_db_query: Input to this tool is a detailed and correct SQL query, output is a result from the database. If the query is not correct, an error message will be returned. If an error is returned, rewrite the query, check the query, and try again. If you encounter an issue with Unknown column 'xxxx' in 'field list', use sql_db_schema to query the correct table fields.

sql_db_schema: Input to this tool is a comma-separated list of tables, output is the schema and sample rows for those tables. Be sure that the tables actually exist by calling sql_db_list_tables first! Example Input: table1, table2, table3

sql_db_list_tables: Input is an empty string, output is a comma-separated list of tables in the database.

sql_db_query_checker: Use this tool to double check if your query is correct before executing it. Always use this tool before executing a query with sql_db_query!

```

## 2. Using a prebuilt agent

Given these tools, we can initialize a pre-built agent in a single line. To customize our agents behavior, we write a descriptive system prompt.

```python
from langgraph.prebuilt import create_react_agent

system_prompt = """
You are an agent designed to interact with a SQL database.
Given an input question, create a syntactically correct {dialect} query to run,
then look at the results of the query and return the answer. Unless the user
specifies a specific number of examples they wish to obtain, always limit your
query to at most {top_k} results.

You can order the results by a relevant column to return the most interesting
examples in the database. Never query for all the columns from a specific table,
only ask for the relevant columns given the question.

You MUST double check your query before executing it. If you get an error while
executing a query, rewrite the query and try again.

DO NOT make any DML statements (INSERT, UPDATE, DELETE, DROP etc.) to the
database.

To start you should ALWAYS look at the tables in the database to see what you
can query. Do NOT skip this step.

Then you should query the schema of the most relevant tables.
""".format(
    dialect=db.dialect,
    top_k=5,
)

agent = create_react_agent(
    llm,
    tools,
    prompt=system_prompt,
)
```

!!! note
    This system prompt includes a number of instructions, such as always running specific tools before or after others. In the [next section](#3-customizing-the-agent), we will enforce these behaviors through the graph's structure, providing us a greater degree of control and allowing us to simplify the prompt.

Let's run this agent on a sample query and observe its behavior:

```python
question = "Which genre on average has the longest tracks?"

for step in agent.stream(
    {"messages": [{"role": "user", "content": question}]},
    stream_mode="values",
):
    step["messages"][-1].pretty_print()
```

**Output:**
```
================================ Human Message =================================

Which genre on average has the longest tracks?
================================== Ai Message ==================================
Tool Calls:
  sql_db_list_tables (call_d8lCgywSroCgpVl558nmXKwA)
 Call ID: call_d8lCgywSroCgpVl558nmXKwA
  Args:
================================= Tool Message =================================
Name: sql_db_list_tables

Album, Artist, Customer, Employee, Genre, Invoice, InvoiceLine, MediaType, Playlist, PlaylistTrack, Track
================================== Ai Message ==================================
Tool Calls:
  sql_db_schema (call_nNf6IIUcwMYLIkE0l6uWkZHe)
 Call ID: call_nNf6IIUcwMYLIkE0l6uWkZHe
  Args:
    table_names: Genre, Track
================================= Tool Message =================================
Name: sql_db_schema


CREATE TABLE "Genre" (
    "GenreId" INTEGER NOT NULL, 
    "Name" NVARCHAR(120), 
    PRIMARY KEY ("GenreId")
)

/*
3 rows from Genre table:
GenreId Name
1   Rock
2   Jazz
3   Metal
*/


CREATE TABLE "Track" (
    "TrackId" INTEGER NOT NULL, 
    "Name" NVARCHAR(200) NOT NULL, 
    "AlbumId" INTEGER, 
    "MediaTypeId" INTEGER NOT NULL, 
    "GenreId" INTEGER, 
    "Composer" NVARCHAR(220), 
    "Milliseconds" INTEGER NOT NULL, 
    "Bytes" INTEGER, 
    "UnitPrice" NUMERIC(10, 2) NOT NULL, 
    PRIMARY KEY ("TrackId"), 
    FOREIGN KEY("MediaTypeId") REFERENCES "MediaType" ("MediaTypeId"), 
    FOREIGN KEY("GenreId") REFERENCES "Genre" ("GenreId"), 
    FOREIGN KEY("AlbumId") REFERENCES "Album" ("AlbumId")
)

/*
3 rows from Track table:
TrackId Name    AlbumId MediaTypeId GenreId Composer    Milliseconds    Bytes   UnitPrice
1   For Those About To Rock (We Salute You) 1   1   1   Angus Young, Malcolm Young, Brian Johnson   343719  11170334    0.99
2   Balls to the Wall   2   2   1   None    342562  5510424 0.99
3   Fast As a Shark 3   2   1   F. Baltes, S. Kaufman, U. Dirkscneider & W. Hoffman 230619  3990994 0.99
*/
================================== Ai Message ==================================
Tool Calls:
  sql_db_query_checker (call_urTRmtiGtTxkwHtscec7Fd2K)
 Call ID: call_urTRmtiGtTxkwHtscec7Fd2K
  Args:
    query: SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgMilliseconds
FROM Track
JOIN Genre ON Track.GenreId = Genre.GenreId
GROUP BY Genre.Name
ORDER BY AvgMilliseconds DESC
LIMIT 1;
================================= Tool Message =================================
Name: sql_db_query_checker

\`\`\`sql
SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgMilliseconds
FROM Track
JOIN Genre ON Track.GenreId = Genre.GenreId
GROUP BY Genre.Name
ORDER BY AvgMilliseconds DESC
LIMIT 1;
\`\`\`
================================== Ai Message ==================================
Tool Calls:
  sql_db_query (call_RNMqyUEMv0rvy0UxSwrXY2AV)
 Call ID: call_RNMqyUEMv0rvy0UxSwrXY2AV
  Args:
    query: SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgMilliseconds
FROM Track
JOIN Genre ON Track.GenreId = Genre.GenreId
GROUP BY Genre.Name
ORDER BY AvgMilliseconds DESC
LIMIT 1;
================================= Tool Message =================================
Name: sql_db_query

[('Sci Fi & Fantasy', 2911783.0384615385)]
================================== Ai Message ==================================

The genre with the longest average track length is "Sci Fi & Fantasy," with an average duration of about 2,911,783 milliseconds (approximately 48.5 minutes) per track.
```

This worked well enough: the agent correctly listed the tables, obtained the schemas, wrote a query, checked the query, and ran it to inform its final response.

!!! tip
    You can inspect all aspects of the above run, including steps taken, tools invoked, what prompts were seen by the LLM, and more in the [LangSmith trace](https://smith.langchain.com/public/bd594960-73e3-474b-b6f2-db039d7c713a/r).

## 3. Customizing the agent

The prebuilt agent lets us get started quickly, but at each step the agent has access to the full set of tools. Above, we relied on the system prompt to constrain its behavior— for example, we instructed the agent to always start with the "list tables" tool, and to always run a query-checker tool before executing the query.

We can enforce a higher degree of control in LangGraph by customizing the agent. Below, we implement a simple ReAct-agent setup, with dedicated nodes for specific tool-calls. We will use the same [state](../../concepts/low_level.md#state) as the pre-built agent.

We construct dedicated nodes for the following steps:

- Listing DB tables
- Calling the "get schema" tool
- Generating a query
- Checking the query

Putting these steps in dedicated nodes lets us (1) force tool-calls when needed, and (2) customize the prompts associated with each step.

```python
from typing import Literal
from langchain_core.messages import AIMessage
from langchain_core.runnables import RunnableConfig
from langgraph.graph import END, START, MessagesState, StateGraph
from langgraph.prebuilt import ToolNode


get_schema_tool = next(tool for tool in tools if tool.name == "sql_db_schema")
get_schema_node = ToolNode([get_schema_tool], name="get_schema")

run_query_tool = next(tool for tool in tools if tool.name == "sql_db_query")
run_query_node = ToolNode([run_query_tool], name="run_query")


# Example: create a predetermined tool call
def list_tables(state: MessagesState):
    tool_call = {
        "name": "sql_db_list_tables",
        "args": {},
        "id": "abc123",
        "type": "tool_call",
    }
    tool_call_message = AIMessage(content="", tool_calls=[tool_call])

    list_tables_tool = next(tool for tool in tools if tool.name == "sql_db_list_tables")
    tool_message = list_tables_tool.invoke(tool_call)
    response = AIMessage(f"Available tables: {tool_message.content}")

    return {"messages": [tool_call_message, tool_message, response]}


# Example: force a model to create a tool call
def call_get_schema(state: MessagesState):
    # Note that LangChain enforces that all models accept `tool_choice="any"`
    # as well as `tool_choice=<string name of tool>`.
    llm_with_tools = llm.bind_tools([get_schema_tool], tool_choice="any")
    response = llm_with_tools.invoke(state["messages"])

    return {"messages": [response]}


generate_query_system_prompt = """
You are an agent designed to interact with a SQL database.
Given an input question, create a syntactically correct {dialect} query to run,
then look at the results of the query and return the answer. Unless the user
specifies a specific number of examples they wish to obtain, always limit your
query to at most {top_k} results.

You can order the results by a relevant column to return the most interesting
examples in the database. Never query for all the columns from a specific table,
only ask for the relevant columns given the question.

DO NOT make any DML statements (INSERT, UPDATE, DELETE, DROP etc.) to the database.
""".format(
    dialect=db.dialect,
    top_k=5,
)


def generate_query(state: MessagesState):
    system_message = {
        "role": "system",
        "content": generate_query_system_prompt,
    }
    # We do not force a tool call here, to allow the model to
    # respond naturally when it obtains the solution.
    llm_with_tools = llm.bind_tools([run_query_tool])
    response = llm_with_tools.invoke([system_message] + state["messages"])

    return {"messages": [response]}


check_query_system_prompt = """
You are a SQL expert with a strong attention to detail.
Double check the {dialect} query for common mistakes, including:
- Using NOT IN with NULL values
- Using UNION when UNION ALL should have been used
- Using BETWEEN for exclusive ranges
- Data type mismatch in predicates
- Properly quoting identifiers
- Using the correct number of arguments for functions
- Casting to the correct data type
- Using the proper columns for joins

If there are any of the above mistakes, rewrite the query. If there are no mistakes,
just reproduce the original query.

You will call the appropriate tool to execute the query after running this check.
""".format(dialect=db.dialect)


def check_query(state: MessagesState):
    system_message = {
        "role": "system",
        "content": check_query_system_prompt,
    }

    # Generate an artificial user message to check
    tool_call = state["messages"][-1].tool_calls[0]
    user_message = {"role": "user", "content": tool_call["args"]["query"]}
    llm_with_tools = llm.bind_tools([run_query_tool], tool_choice="any")
    response = llm_with_tools.invoke([system_message, user_message])
    response.id = state["messages"][-1].id

    return {"messages": [response]}
```

Finally, we assemble these steps into a workflow using the Graph API. We define a [conditional edge](../../concepts/low_level.md#conditional-edges) at the query generation step that will route to the query checker if a query is generated, or end if there are no tool calls present, such that the LLM has delivered a response to the query.

```python
def should_continue(state: MessagesState) -> Literal[END, "check_query"]:
    messages = state["messages"]
    last_message = messages[-1]
    if not last_message.tool_calls:
        return END
    else:
        return "check_query"


builder = StateGraph(MessagesState)
builder.add_node(list_tables)
builder.add_node(call_get_schema)
builder.add_node(get_schema_node, "get_schema")
builder.add_node(generate_query)
builder.add_node(check_query)
builder.add_node(run_query_node, "run_query")

builder.add_edge(START, "list_tables")
builder.add_edge("list_tables", "call_get_schema")
builder.add_edge("call_get_schema", "get_schema")
builder.add_edge("get_schema", "generate_query")
builder.add_conditional_edges(
    "generate_query",
    should_continue,
)
builder.add_edge("check_query", "run_query")
builder.add_edge("run_query", "generate_query")

agent = builder.compile()
```

We visualize the application below:

```python
from IPython.display import Image, display
from langchain_core.runnables.graph import CurveStyle, MermaidDrawMethod, NodeStyles

display(Image(agent.get_graph().draw_mermaid_png()))
```

![Graph](./output.png)

**Note:** When you run this code, it will generate and display a visual representation of the SQL agent graph showing the flow between the different nodes (list_tables → call_get_schema → get_schema → generate_query → check_query → run_query).

We can now invoke the graph exactly as before:

```python
question = "Which genre on average has the longest tracks?"

for step in agent.stream(
    {"messages": [{"role": "user", "content": question}]},
    stream_mode="values",
):
    step["messages"][-1].pretty_print()
```

**Output:**
```
================================ Human Message =================================

Which genre on average has the longest tracks?
================================== Ai Message ==================================

Available tables: Album, Artist, Customer, Employee, Genre, Invoice, InvoiceLine, MediaType, Playlist, PlaylistTrack, Track
================================== Ai Message ==================================
Tool Calls:
  sql_db_schema (call_qxKtYiHgf93AiTDin9ez5wFp)
 Call ID: call_qxKtYiHgf93AiTDin9ez5wFp
  Args:
    table_names: Genre,Track
================================= Tool Message =================================
Name: sql_db_schema


CREATE TABLE "Genre" (
    "GenreId" INTEGER NOT NULL, 
    "Name" NVARCHAR(120), 
    PRIMARY KEY ("GenreId")
)

/*
3 rows from Genre table:
GenreId Name
1   Rock
2   Jazz
3   Metal
*/


CREATE TABLE "Track" (
    "TrackId" INTEGER NOT NULL, 
    "Name" NVARCHAR(200) NOT NULL, 
    "AlbumId" INTEGER, 
    "MediaTypeId" INTEGER NOT NULL, 
    "GenreId" INTEGER, 
    "Composer" NVARCHAR(220), 
    "Milliseconds" INTEGER NOT NULL, 
    "Bytes" INTEGER, 
    "UnitPrice" NUMERIC(10, 2) NOT NULL, 
    PRIMARY KEY ("TrackId"), 
    FOREIGN KEY("MediaTypeId") REFERENCES "MediaType" ("MediaTypeId"), 
    FOREIGN KEY("GenreId") REFERENCES "Genre" ("GenreId"), 
    FOREIGN KEY("AlbumId") REFERENCES "Album" ("AlbumId")
)

/*
3 rows from Track table:
TrackId Name    AlbumId MediaTypeId GenreId Composer    Milliseconds    Bytes   UnitPrice
1   For Those About To Rock (We Salute You) 1   1   1   Angus Young, Malcolm Young, Brian Johnson   343719  11170334    0.99
2   Balls to the Wall   2   2   1   None    342562  5510424 0.99
3   Fast As a Shark 3   2   1   F. Baltes, S. Kaufman, U. Dirkscneider & W. Hoffman 230619  3990994 0.99
*/
================================== Ai Message ==================================
Tool Calls:
  sql_db_query (call_RPN3GABMfb6DTaFTLlwnZxVN)
 Call ID: call_RPN3GABMfb6DTaFTLlwnZxVN
  Args:
    query: SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgTrackLength
FROM Track
JOIN Genre ON Track.GenreId = Genre.GenreId
GROUP BY Genre.GenreId
ORDER BY AvgTrackLength DESC
LIMIT 1;
================================== Ai Message ==================================
Tool Calls:
  sql_db_query (call_PR4s8ymiF3ZQLaoZADXtdqcl)
 Call ID: call_PR4s8ymiF3ZQLaoZADXtdqcl
  Args:
    query: SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgTrackLength
FROM Track
JOIN Genre ON Track.GenreId = Genre.GenreId
GROUP BY Genre.GenreId
ORDER BY AvgTrackLength DESC
LIMIT 1;
================================= Tool Message =================================
Name: sql_db_query

[('Sci Fi & Fantasy', 2911783.0384615385)]
================================== Ai Message ==================================

The genre with the longest tracks on average is "Sci Fi & Fantasy," with an average track length of approximately 2,911,783 milliseconds.
```

!!! tip
    See [LangSmith trace](https://smith.langchain.com/public/94b8c9ac-12f7-4692-8706-836a1f30f1ea/r) for the above run.

## Next steps

Check out [this guide](https://docs.smith.langchain.com/evaluation/how_to_guides/langgraph) for evaluating LangGraph applications, including SQL agents like this one, using LangSmith. 
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/multi_agent/agent_supervisor.md
```md
# Multi-agent supervisor

[**Supervisor**](../../concepts/multi_agent.md#supervisor) is a multi-agent architecture where **specialized** agents are coordinated by a central **supervisor agent**. The supervisor agent controls all communication flow and task delegation, making decisions about which agent to invoke based on the current context and task requirements.

In this tutorial, you will build a supervisor system with two agents — a research and a math expert. By the end of the tutorial you will:

1. Build specialized research and math agents
2. Build a supervisor for orchestrating them with the prebuilt [`langgraph-supervisor`](https://langchain-ai.github.io/langgraph/agents/multi-agent/#supervisor)
3. Build a supervisor from scratch
4. Implement advanced task delegation

![diagram](assets/diagram.png)

## Setup

First, let's install required packages and set our API keys

```python
%%capture --no-stderr
%pip install -U langgraph langgraph-supervisor langchain-tavily "langchain[openai]"
```

```python
import getpass
import os


def _set_if_undefined(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"Please provide your {var}")


_set_if_undefined("OPENAI_API_KEY")
_set_if_undefined("TAVILY_API_KEY")
```

!!! tip
    Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. [LangSmith](https://docs.smith.langchain.com) lets you use trace data to debug, test, and monitor your LLM apps built with LangGraph.

## 1. Create worker agents

First, let's create our specialized worker agents — research agent and math agent:

* Research agent will have access to a web search tool using [Tavily API](https://tavily.com/)
* Math agent will have access to simple math tools (`add`, `multiply`, `divide`)

### Research agent

For web search, we will use `TavilySearch` tool from `langchain-tavily`:

```python
from langchain_tavily import TavilySearch

web_search = TavilySearch(max_results=3)
web_search_results = web_search.invoke("who is the mayor of NYC?")

print(web_search_results["results"][0]["content"])
```

**Output:**
```
Find events, attractions, deals, and more at nyctourism.com Skip Main Navigation Menu The Official Website of the City of New York Text Size Powered by Translate SearchSearch Primary Navigation The official website of NYC Home NYC Resources NYC311 Office of the Mayor Events Connect Jobs Search Office of the Mayor | Mayor's Bio | City of New York Secondary Navigation MayorBiographyNewsOfficials Eric L. Adams 110th Mayor of New York City Mayor Eric Adams has served the people of New York City as an NYPD officer, State Senator, Brooklyn Borough President, and now as the 110th Mayor of the City of New York. Mayor Eric Adams has served the people of New York City as an NYPD officer, State Senator, Brooklyn Borough President, and now as the 110th Mayor of the City of New York. He gave voice to a diverse coalition of working families in all five boroughs and is leading the fight to bring back New York City's economy, reduce inequality, improve public safety, and build a stronger, healthier city that delivers for all New Yorkers. As the representative of one of the nation's largest counties, Eric fought tirelessly to grow the local economy, invest in schools, reduce inequality, improve public safety, and advocate for smart policies and better government that delivers for all New Yorkers.
```

To create individual worker agents, we will use LangGraph's prebuilt [agent](../../agents/agents.md).

```python
from langgraph.prebuilt import create_react_agent

research_agent = create_react_agent(
    model="openai:gpt-4.1",
    tools=[web_search],
    prompt=(
        "You are a research agent.\n\n"
        "INSTRUCTIONS:\n"
        "- Assist ONLY with research-related tasks, DO NOT do any math\n"
        "- After you're done with your tasks, respond to the supervisor directly\n"
        "- Respond ONLY with the results of your work, do NOT include ANY other text."
    ),
    name="research_agent",
)
```

Let's [run the agent](../../agents/run_agents.md) to verify that it behaves as expected. 

!!! note "We'll use `pretty_print_messages` helper to render the streamed agent outputs nicely"

  ```python
  from langchain_core.messages import convert_to_messages


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
  ```

  ```python
  for chunk in research_agent.stream(
      {"messages": [{"role": "user", "content": "who is the mayor of NYC?"}]}
  ):
      pretty_print_messages(chunk)
  ```

  **Output:**
  ```
  Update from node agent:


  ================================== Ai Message ==================================
  Name: research_agent
  Tool Calls:
    tavily_search (call_U748rQhQXT36sjhbkYLSXQtJ)
   Call ID: call_U748rQhQXT36sjhbkYLSXQtJ
    Args:
      query: current mayor of New York City
      search_depth: basic


  Update from node tools:


  ================================= Tool Message ==================================
  Name: tavily_search

  {"query": "current mayor of New York City", "follow_up_questions": null, "answer": null, "images": [], "results": [{"title": "List of mayors of New York City - Wikipedia", "url": "https://en.wikipedia.org/wiki/List_of_mayors_of_New_York_City", "content": "The mayor of New York City is the chief executive of the Government of New York City, as stipulated by New York City's charter.The current officeholder, the 110th in the sequence of regular mayors, is Eric Adams, a member of the Democratic Party.. During the Dutch colonial period from 1624 to 1664, New Amsterdam was governed by the Director of Netherland.", "score": 0.9039154, "raw_content": null}, {"title": "Office of the Mayor | Mayor's Bio | City of New York - NYC.gov", "url": "https://www.nyc.gov/office-of-the-mayor/bio.page", "content": "Mayor Eric Adams has served the people of New York City as an NYPD officer, State Senator, Brooklyn Borough President, and now as the 110th Mayor of the City of New York. He gave voice to a diverse coalition of working families in all five boroughs and is leading the fight to bring back New York City's economy, reduce inequality, improve", "score": 0.8405867, "raw_content": null}, {"title": "Eric Adams - Wikipedia", "url": "https://en.wikipedia.org/wiki/Eric_Adams", "content": "Eric Leroy Adams (born September 1, 1960) is an American politician and former police officer who has served as the 110th mayor of New York City since 2022. Adams was an officer in the New York City Transit Police and then the New York City Police Department (```
  ```

### Math agent

For math agent tools we will use [vanilla Python functions](../../how-tos/tool-calling.md#define-a-tool):

```python
def add(a: float, b: float):
    """Add two numbers."""
    return a + b


def multiply(a: float, b: float):
    """Multiply two numbers."""
    return a * b


def divide(a: float, b: float):
    """Divide two numbers."""
    return a / b


math_agent = create_react_agent(
    model="openai:gpt-4.1",
    tools=[add, multiply, divide],
    prompt=(
        "You are a math agent.\n\n"
        "INSTRUCTIONS:\n"
        "- Assist ONLY with math-related tasks\n"
        "- After you're done with your tasks, respond to the supervisor directly\n"
        "- Respond ONLY with the results of your work, do NOT include ANY other text."
    ),
    name="math_agent",
)
```

Let's run the math agent:

```python
for chunk in math_agent.stream(
    {"messages": [{"role": "user", "content": "what's (3 + 5) x 7"}]}
):
    pretty_print_messages(chunk)
```

**Output:**
```
Update from node agent:


================================== Ai Message ==================================
Name: math_agent
Tool Calls:
  add (call_p6OVLDHB4LyCNCxPOZzWR15v)
 Call ID: call_p6OVLDHB4LyCNCxPOZzWR15v
  Args:
    a: 3
    b: 5


Update from node tools:


================================= Tool Message ==================================
Name: add

8.0


Update from node agent:


================================== Ai Message ==================================
Name: math_agent
Tool Calls:
  multiply (call_EoaWHMLFZAX4AkajQCtZvbli)
 Call ID: call_EoaWHMLFZAX4AkajQCtZvbli
  Args:
    a: 8
    b: 7


Update from node tools:


================================= Tool Message ==================================
Name: multiply

56.0


Update from node agent:


================================== Ai Message ==================================
Name: math_agent

56


```

## 2. Create supervisor with `langgraph-supervisor`

To implement out multi-agent system, we will use @[`create_supervisor`][create_supervisor] from the prebuilt `langgraph-supervisor` library:

```python
from langgraph_supervisor import create_supervisor
from langchain.chat_models import init_chat_model

supervisor = create_supervisor(
    model=init_chat_model("openai:gpt-4.1"),
    agents=[research_agent, math_agent],
    prompt=(
        "You are a supervisor managing two agents:\n"
        "- a research agent. Assign research-related tasks to this agent\n"
        "- a math agent. Assign math-related tasks to this agent\n"
        "Assign work to one agent at a time, do not call agents in parallel.\n"
        "Do not do any work yourself."
    ),
    add_handoff_back_messages=True,
    output_mode="full_history",
).compile()
```

```python
from IPython.display import display, Image

display(Image(supervisor.get_graph().draw_mermaid_png()))
```

![Graph](assets/output.png)

**Note:** When you run this code, it will generate and display a visual representation of the supervisor graph showing the flow between the supervisor and worker agents.

Let's now run it with a query that requires both agents:

* research agent will look up the necessary GDP information
* math agent will perform division to find the percentage of NY state GDP, as requested

```python
for chunk in supervisor.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "find US and New York state GDP in 2024. what % of US GDP was New York state?",
            }
        ]
    },
):
    pretty_print_messages(chunk, last_message=True)

final_message_history = chunk["supervisor"]["messages"]
```

**Output:**
```
Update from node supervisor:


================================= Tool Message ==================================
Name: transfer_to_research_agent

Successfully transferred to research_agent


Update from node research_agent:


================================= Tool Message ==================================
Name: transfer_back_to_supervisor

Successfully transferred back to supervisor


Update from node supervisor:


================================= Tool Message ==================================
Name: transfer_to_math_agent

Successfully transferred to math_agent


Update from node math_agent:


================================= Tool Message ==================================
Name: transfer_back_to_supervisor

Successfully transferred back to supervisor


Update from node supervisor:


================================== Ai Message ==================================
Name: supervisor

In 2024, the US GDP was $29.18 trillion and New York State's GDP was $2.297 trillion. New York State accounted for approximately 7.87% of the total US GDP in 2024.


```

## 3. Create supervisor from scratch

Let's now implement this same multi-agent system from scratch. We will need to:

1. [Set up how the supervisor communicates](#set-up-agent-communication) with individual agents
2. [Create the supervisor agent](#create-supervisor-agent)
3. Combine supervisor and worker agents into a [single multi-agent graph](#create-multi-agent-graph).

### Set up agent communication

We will need to define a way for the supervisor agent to communicate with the worker agents. A common way to implement this in multi-agent architectures is using **handoffs**, where one agent *hands off* control to another. Handoffs allow you to specify:

- **destination**: target agent to transfer to
- **payload**: information to pass to that agent

We will implement handoffs via **handoff tools** and give these tools to the supervisor agent: when the supervisor calls these tools, it will hand off control to a worker agent, passing the full message history to that agent.

```python
from typing import Annotated
from langchain_core.tools import tool, InjectedToolCallId
from langgraph.prebuilt import InjectedState
from langgraph.graph import StateGraph, START, MessagesState
from langgraph.types import Command


def create_handoff_tool(*, agent_name: str, description: str | None = None):
    name = f"transfer_to_{agent_name}"
    description = description or f"Ask {agent_name} for help."

    @tool(name, description=description)
    def handoff_tool(
        state: Annotated[MessagesState, InjectedState],
        tool_call_id: Annotated[str, InjectedToolCallId],
    ) -> Command:
        tool_message = {
            "role": "tool",
            "content": f"Successfully transferred to {agent_name}",
            "name": name,
            "tool_call_id": tool_call_id,
        }
        # highlight-next-line
        return Command(
            # highlight-next-line
            goto=agent_name,  # (1)!
            # highlight-next-line
            update={**state, "messages": state["messages"] + [tool_message]},  # (2)!
            # highlight-next-line
            graph=Command.PARENT,  # (3)!
        )

    return handoff_tool


# Handoffs
assign_to_research_agent = create_handoff_tool(
    agent_name="research_agent",
    description="Assign task to a researcher agent.",
)

assign_to_math_agent = create_handoff_tool(
    agent_name="math_agent",
    description="Assign task to a math agent.",
)
```

1. Name of the agent or node to hand off to.
2. Take the agent's messages and add them to the parent's state as part of the handoff. The next agent will see the parent state.
3. Indicate to LangGraph that we need to navigate to agent node in a **parent** multi-agent graph.

### Create supervisor agent

Then, let's create the supervisor agent with the handoff tools we just defined. We will use the prebuilt @[`create_react_agent`][create_react_agent]:

```python
supervisor_agent = create_react_agent(
    model="openai:gpt-4.1",
    tools=[assign_to_research_agent, assign_to_math_agent],
    prompt=(
        "You are a supervisor managing two agents:\n"
        "- a research agent. Assign research-related tasks to this agent\n"
        "- a math agent. Assign math-related tasks to this agent\n"
        "Assign work to one agent at a time, do not call agents in parallel.\n"
        "Do not do any work yourself."
    ),
    name="supervisor",
)
```

### Create multi-agent graph

Putting this all together, let's create a graph for our overall multi-agent system. We will add the supervisor and the individual agents as subgraph [nodes](../../concepts/low_level.md#nodes).

```python
from langgraph.graph import END

# Define the multi-agent supervisor graph
supervisor = (
    StateGraph(MessagesState)
    # NOTE: `destinations` is only needed for visualization and doesn't affect runtime behavior
    .add_node(supervisor_agent, destinations=("research_agent", "math_agent", END))
    .add_node(research_agent)
    .add_node(math_agent)
    .add_edge(START, "supervisor")
    # always return back to the supervisor
    .add_edge("research_agent", "supervisor")
    .add_edge("math_agent", "supervisor")
    .compile()
)
```

Notice that we've added explicit [edges](../../concepts/low_level.md#edges) from worker agents back to the supervisor — this means that they are guaranteed to return control back to the supervisor. If you want the agents to respond directly to the user (i.e., turn the system into a router, you can remove these edges).

```python
from IPython.display import display, Image

display(Image(supervisor.get_graph().draw_mermaid_png()))
```

![Graph](assets/multi-output.png)

**Note:** When you run this code, it will generate and display a visual representation of the multi-agent supervisor graph showing the flow between the supervisor and worker agents.

With the multi-agent graph created, let's now run it!

```python
for chunk in supervisor.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "find US and New York state GDP in 2024. what % of US GDP was New York state?",
            }
        ]
    },
):
    pretty_print_messages(chunk, last_message=True)

final_message_history = chunk["supervisor"]["messages"]
```

**Output:**
```
Update from node supervisor:


================================= Tool Message ==================================
Name: transfer_to_research_agent

Successfully transferred to research_agent


Update from node research_agent:


================================== Ai Message ==================================
Name: research_agent

- US GDP in 2024 is projected to be about $28.18 trillion USD (Statista; CBO projection).
- New York State's nominal GDP for 2024 is estimated at approximately $2.16 trillion USD (various economic reports).
- New York State's share of US GDP in 2024 is roughly 7.7%.

Sources:
- https://www.statista.com/statistics/216985/forecast-of-us-gross-domestic-product/
- https://nyassembly.gov/Reports/WAM/2025economic_revenue/2025_report.pdf?v=1740533306


Update from node supervisor:


================================= Tool Message ==================================
Name: transfer_to_math_agent

Successfully transferred to math_agent


Update from node math_agent:


================================== Ai Message ==================================
Name: math_agent

US GDP in 2024: $28.18 trillion
New York State GDP in 2024: $2.16 trillion
Percentage of US GDP from New York State: 7.67%


Update from node supervisor:


================================== Ai Message ==================================
Name: supervisor

Here are your results:

- 2024 US GDP (projected): $28.18 trillion USD
- 2024 New York State GDP (estimated): $2.16 trillion USD
- New York State's share of US GDP: approximately 7.7%

If you need the calculation steps or sources, let me know!


```

Let's examine the full resulting message history:

```python
for message in final_message_history:
    message.pretty_print()
```

**Output:**
```
================================ Human Message ==================================

find US and New York state GDP in 2024. what % of US GDP was New York state?
================================== Ai Message ===================================
Name: supervisor
Tool Calls:
  transfer_to_research_agent (call_KlGgvF5ahlAbjX8d2kHFjsC3)
 Call ID: call_KlGgvF5ahlAbjX8d2kHFjsC3
  Args:
================================= Tool Message ==================================
Name: transfer_to_research_agent

Successfully transferred to research_agent
================================== Ai Message ===================================
Name: research_agent
Tool Calls:
  tavily_search (call_ZOaTVUA6DKrOjWQldLhtrsO2)
 Call ID: call_ZOaTVUA6DKrOjWQldLhtrsO2
  Args:
    query: US GDP 2024 estimate or actual
    search_depth: advanced
  tavily_search (call_QsRAasxW9K03lTlqjuhNLFbZ)
 Call ID: call_QsRAasxW9K03lTlqjuhNLFbZ
  Args:
    query: New York state GDP 2024 estimate or actual
    search_depth: advanced
================================= Tool Message ==================================
Name: tavily_search

{"query": "US GDP 2024 estimate or actual", "follow_up_questions": null, "answer": null, "images": [], "results": [{"url": "https://www.advisorperspectives.com/dshort/updates/2025/05/29/gdp-gross-domestic-product-q1-2025-second-estimate", "title": "Q1 GDP Second Estimate: Real GDP at -0.2%, Higher Than Expected", "content": "> Real gross domestic product (GDP) decreased at an annual rate of 0.2 percent in the first quarter of 2025 (January, February, and March), according to the second estimate released by the U.S. Bureau of Economic Analysis. In the fourth quarter of 2024, real GDP increased 2.4 percent. The decrease in real GDP in the first quarter primarily reflected an increase in imports, which are a subtraction in the calculation of GDP, and a decrease in government spending. These movements were partly [...] by [Harry Mamaysky](https://www.advisor```
```

!!! important
    You can see that the supervisor system appends **all** of the individual agent messages (i.e., their internal tool-calling loop) to the full message history. This means that on every supervisor turn, supervisor agent sees this full history. If you want more control over:

    * **how inputs are passed to agents**: you can use LangGraph @[`Send()`][Send] primitive to directly send data to the worker agents during the handoff. See the [task delegation](#4-create-delegation-tasks) example below
    * **how agent outputs are added**: you can control how much of the agent's internal message history is added to the overall supervisor message history by wrapping the agent in a separate node function:

        ```python
        def call_research_agent(state):
            # return agent's final response,
            # excluding inner monologue
            response = research_agent.invoke(state)
            # highlight-next-line
            return {"messages": response["messages"][-1]}
        ```

## 4. Create delegation tasks

So far the individual agents relied on **interpreting full message history** to determine their tasks. An alternative approach is to ask the supervisor to **formulate a task explicitly**. We can do so by adding a `task_description` parameter to the `handoff_tool` function.

```python
from langgraph.types import Send


def create_task_description_handoff_tool(
    *, agent_name: str, description: str | None = None
):
    name = f"transfer_to_{agent_name}"
    description = description or f"Ask {agent_name} for help."

    @tool(name, description=description)
    def handoff_tool(
        # this is populated by the supervisor LLM
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


assign_to_research_agent_with_description = create_task_description_handoff_tool(
    agent_name="research_agent",
    description="Assign task to a researcher agent.",
)

assign_to_math_agent_with_description = create_task_description_handoff_tool(
    agent_name="math_agent",
    description="Assign task to a math agent.",
)

supervisor_agent_with_description = create_react_agent(
    model="openai:gpt-4.1",
    tools=[
        assign_to_research_agent_with_description,
        assign_to_math_agent_with_description,
    ],
    prompt=(
        "You are a supervisor managing two agents:\n"
        "- a research agent. Assign research-related tasks to this assistant\n"
        "- a math agent. Assign math-related tasks to this assistant\n"
        "Assign work to one agent at a time, do not call agents in parallel.\n"
        "Do not do any work yourself."
    ),
    name="supervisor",
)

supervisor_with_description = (
    StateGraph(MessagesState)
    .add_node(
        supervisor_agent_with_description, destinations=("research_agent", "math_agent")
    )
    .add_node(research_agent)
    .add_node(math_agent)
    .add_edge(START, "supervisor")
    .add_edge("research_agent", "supervisor")
    .add_edge("math_agent", "supervisor")
    .compile()
)
```

!!! note
    We're using @[`Send()`][Send] primitive in the `handoff_tool`. This means that instead of receiving the full `supervisor` graph state as input, each worker agent only sees the contents of the `Send` payload. In this example, we're sending the task description as a single "human" message.

Let's now running it with the same input query:

```python
for chunk in supervisor_with_description.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "find US and New York state GDP in 2024. what % of US GDP was New York state?",
            }
        ]
    },
    subgraphs=True,
):
    pretty_print_messages(chunk, last_message=True)
```

**Output:**
```
Update from subgraph supervisor:


	Update from node agent:


	================================== Ai Message ==================================
	Name: supervisor
	Tool Calls:
	  transfer_to_research_agent (call_tk8q8py8qK6MQz6Kj6mijKua)
	 Call ID: call_tk8q8py8qK6MQz6Kj6mijKua
	  Args:
	    task_description: Find the 2024 GDP (Gross Domestic Product) for both the United States and New York state, using the most up-to-date and reputable sources available. Provide both GDP values and cite the data sources.


Update from subgraph research_agent:


	Update from node agent:


	================================== Ai Message ==================================
	Name: research_agent
	Tool Calls:
	  tavily_search (call_KqvhSvOIhAvXNsT6BOwbPlRB)
	 Call ID: call_KqvhSvOIhAvXNsT6BOwbPlRB
	  Args:
	    query: 2024 United States GDP value from a reputable source
	    search_depth: advanced
	  tavily_search (call_kbbAWBc9KwCWKHmM5v04H88t)
	 Call ID: call_kbbAWBc9KwCWKHmM5v04H88t
	  Args:
	    query: 2024 New York state GDP value from a reputable source
	    search_depth: advanced


Update from subgraph research_agent:


	Update from node tools:


	================================= Tool Message ==================================
	Name: tavily_search
	
	{"query": "2024 United States GDP value from a reputable source", "follow_up_questions": null, "answer": null, "images": [], "results": [{"url": "https://www.focus-economics.com/countries/united-states/", "title": "United States Economy Overview - Focus Economics", "content": "The United States' Macroeconomic Analysis:\n------------------------------------------\n\n**Nominal GDP of USD 29,185 billion in 2024.**\n\n**Nominal GDP of USD 29,179 billion in 2024.**\n\n**GDP per capita of USD 86,635 compared to the global average of USD 10,589.**\n\n**GDP per capita of USD 86,652 compared to the global average of USD 10,589.**\n\n**Average real GDP growth of 2.5% over the last decade.**\n\n**Average real GDP growth of ```
``` 

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/multi_agent/hierarchical_agent_teams.ipynb
```ipynb
{
 "cells": [
  {
   "attachments": {
    "d98ed25c-51cb-441f-a6f4-016921d59fc3.png": {
