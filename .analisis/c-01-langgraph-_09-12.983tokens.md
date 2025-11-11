    }
   },
   "cell_type": "markdown",
   "id": "bb89d3f0-7ade-43a8-a527-4bec45971cf6",
   "metadata": {},
   "source": [
    "# Adaptive RAG using local LLMs\n",
    "\n",
    "Adaptive RAG is a strategy for RAG that unites (1) [query analysis](https://blog.langchain.dev/query-construction/) with (2) [active / self-corrective RAG](https://blog.langchain.dev/agentic-rag-with-langgraph/).\n",
    "\n",
    "In the [paper](https://arxiv.org/abs/2403.14403), they report query analysis to route across:\n",
    "\n",
    "* No Retrieval\n",
    "* Single-shot RAG\n",
    "* Iterative RAG\n",
    "\n",
    "Let's build on this using LangGraph. \n",
    "\n",
    "In our implementation, we will route between:\n",
    "\n",
    "* Web search: for questions related to recent events\n",
    "* Self-corrective RAG: for questions related to our index\n",
    "\n",
    "![Screenshot 2024-04-01 at 1.29.15 PM.png](attachment:3755396d-c4a8-45bd-87d4-00cb56339fe5.png)"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "8cece98f-a3ed-417e-8b6a-1754e8f9c42a",
   "metadata": {},
   "source": [
    "## Setup\n",
    "\n",
    "First, let's install our required packages and set our API keys"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "88debf5c-6972-415c-b8fb-f65eab203b7a",
   "metadata": {},
   "outputs": [],
   "source": [
    "%capture --no-stderr\n",
    "%pip install -U langchain-nomic langchain_community tiktoken langchainhub chromadb langchain langgraph tavily-python nomic[local]"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "2369652a",
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
    "_set_env(\"TAVILY_API_KEY\")\n",
    "_set_env(\"NOMIC_API_KEY\")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "aea269f6",
   "metadata": {},
   "source": [
    "<div class=\"admonition tip\">\n",
    "    <p class=\"admonition-title\">Set up <a href=\"https://smith.langchain.com\">LangSmith</a> for LangGraph development</p>\n",
    "    <p style=\"padding-top: 5px;\">\n",
    "        Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. LangSmith lets you use trace data to debug, test, and monitor your LLM apps built with LangGraph — read more about how to get started <a href=\"https://docs.smith.langchain.com\">here</a>. \n",
    "    </p>\n",
    "</div>    "
   ]
  },
  {
   "cell_type": "markdown",
   "id": "6a5d4a26-249b-4551-aa13-6c373429618e",
   "metadata": {},
   "source": [
    "### LLMs\n",
    "\n",
    "#### Local Embeddings\n",
    "\n",
    "You can use `GPT4AllEmbeddings()` from Nomic, which can access use Nomic's recently released [v1](https://blog.nomic.ai/posts/nomic-embed-text-v1) and [v1.5](https://blog.nomic.ai/posts/nomic-embed-matryoshka) embeddings.\n",
    "\n",
    "Follow the documentation [here](https://docs.gpt4all.io/gpt4all_python_embedding.html#supported-embedding-models).\n",
    "\n",
    "#### Local LLM\n",
    "\n",
    "(1) Download [Ollama app](https://ollama.ai/).\n",
    "\n",
    "(2) Download a `Mistral` model from various Mistral versions [here](https://ollama.ai/library/mistral) and Mixtral versions [here](https://ollama.ai/library/mixtral) available. Also, try one of the [quantized command-R models](https://ollama.com/library/command-r).\n",
    "\n",
    "```\n",
    "ollama pull mistral\n",
    "```"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 2,
   "id": "af8379bd-7eae-4ba6-b632-12e89eab9920",
   "metadata": {},
   "outputs": [],
   "source": [
    "# Ollama model name\n",
    "local_llm = \"mistral\""
   ]
  },
  {
   "cell_type": "markdown",
   "id": "04718a0c-7a48-4243-97a2-940a0239cc12",
   "metadata": {},
   "source": [
    "## Create Index"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 3,
   "id": "f9ff6b99-080d-4827-b2cb-f775543d76f5",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langchain.text_splitter import RecursiveCharacterTextSplitter\n",
    "from langchain_community.document_loaders import WebBaseLoader\n",
    "from langchain_community.vectorstores import Chroma\n",
    "from langchain_nomic.embeddings import NomicEmbeddings\n",
    "\n",
    "urls = [\n",
    "    \"https://lilianweng.github.io/posts/2023-06-23-agent/\",\n",
    "    \"https://lilianweng.github.io/posts/2023-03-15-prompt-engineering/\",\n",
    "    \"https://lilianweng.github.io/posts/2023-10-25-adv-attack-llm/\",\n",
    "]\n",
    "\n",
    "docs = [WebBaseLoader(url).load() for url in urls]\n",
    "docs_list = [item for sublist in docs for item in sublist]\n",
    "\n",
    "text_splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(\n",
    "    chunk_size=250, chunk_overlap=0\n",
    ")\n",
    "doc_splits = text_splitter.split_documents(docs_list)\n",
    "\n",
    "# Add to vectorDB\n",
    "vectorstore = Chroma.from_documents(\n",
    "    documents=doc_splits,\n",
    "    collection_name=\"rag-chroma\",\n",
    "    embedding=NomicEmbeddings(model=\"nomic-embed-text-v1.5\", inference_mode=\"local\"),\n",
    ")\n",
    "retriever = vectorstore.as_retriever()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "2f3eb922-27a1-4a72-a727-85fbf5b3daf1",
   "metadata": {},
   "source": [
    "## LLMs\n",
    "\n",
    "Note: tested cmd-R on Mac M2 32GB and [latency is ~52 sec for RAG generation](https://smith.langchain.com/public/3998fe48-efc2-4d18-9069-972643d0982d/r)."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 4,
   "id": "7045e064-e666-4aea-9111-6e9d2007f27e",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "{'datasource': 'vectorstore'}\n"
     ]
    }
   ],
   "source": [
    "### Router\n",
    "\n",
    "from langchain.prompts import PromptTemplate\n",
    "from langchain_community.chat_models import ChatOllama\n",
    "from langchain_core.output_parsers import JsonOutputParser\n",
    "\n",
    "# LLM\n",
    "llm = ChatOllama(model=local_llm, format=\"json\", temperature=0)\n",
    "\n",
    "prompt = PromptTemplate(\n",
    "    template=\"\"\"You are an expert at routing a user question to a vectorstore or web search. \\n\n",
    "    Use the vectorstore for questions on LLM  agents, prompt engineering, and adversarial attacks. \\n\n",
    "    You do not need to be stringent with the keywords in the question related to these topics. \\n\n",
    "    Otherwise, use web-search. Give a binary choice 'web_search' or 'vectorstore' based on the question. \\n\n",
    "    Return the a JSON with a single key 'datasource' and no premable or explanation. \\n\n",
    "    Question to route: {question}\"\"\",\n",
    "    input_variables=[\"question\"],\n",
    ")\n",
    "\n",
    "question_router = prompt | llm | JsonOutputParser()\n",
    "question = \"llm agent memory\"\n",
    "docs = retriever.get_relevant_documents(question)\n",
    "doc_txt = docs[1].page_content\n",
    "print(question_router.invoke({\"question\": question}))"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 7,
   "id": "813cdcef-8b75-4214-a2ed-b89077b3d287",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "{'score': 'yes'}\n"
     ]
    }
   ],
   "source": [
    "### Retrieval Grader\n",
    "\n",
    "from langchain.prompts import PromptTemplate\n",
    "from langchain_community.chat_models import ChatOllama\n",
    "from langchain_core.output_parsers import JsonOutputParser\n",
    "\n",
    "# LLM\n",
    "llm = ChatOllama(model=local_llm, format=\"json\", temperature=0)\n",
    "\n",
    "prompt = PromptTemplate(\n",
    "    template=\"\"\"You are a grader assessing relevance of a retrieved document to a user question. \\n \n",
    "    Here is the retrieved document: \\n\\n {document} \\n\\n\n",
    "    Here is the user question: {question} \\n\n",
    "    If the document contains keywords related to the user question, grade it as relevant. \\n\n",
    "    It does not need to be a stringent test. The goal is to filter out erroneous retrievals. \\n\n",
    "    Give a binary score 'yes' or 'no' score to indicate whether the document is relevant to the question. \\n\n",
    "    Provide the binary score as a JSON with a single key 'score' and no premable or explanation.\"\"\",\n",
    "    input_variables=[\"question\", \"document\"],\n",
    ")\n",
    "\n",
    "retrieval_grader = prompt | llm | JsonOutputParser()\n",
    "question = \"agent memory\"\n",
    "docs = retriever.get_relevant_documents(question)\n",
    "doc_txt = docs[1].page_content\n",
    "print(retrieval_grader.invoke({\"question\": question, \"document\": doc_txt}))"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 8,
   "id": "aeb8b373-0289-4dec-bd4b-8b2701200301",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      " In an LLM-powered autonomous agent system, the Large Language Model (LLM) functions as the agent's brain. The agent has key components including memory, planning, and reflection mechanisms. The memory component is a long-term memory module that records a comprehensive list of agents’ experience in natural language. It includes a memory stream, which is an external database for storing past experiences. The reflection mechanism synthesizes memories into higher-level inferences over time and guides the agent's future behavior.\n"
     ]
    }
   ],
   "source": [
    "### Generate\n",
    "\n",
    "from langchain import hub\n",
    "from langchain_community.chat_models import ChatOllama\n",
    "from langchain_core.output_parsers import StrOutputParser\n",
    "\n",
    "# Prompt\n",
    "prompt = hub.pull(\"rlm/rag-prompt\")\n",
    "\n",
    "# LLM\n",
    "llm = ChatOllama(model=local_llm, temperature=0)\n",
    "\n",
    "\n",
    "# Post-processing\n",
    "def format_docs(docs):\n",
    "    return \"\\n\\n\".join(doc.page_content for doc in docs)\n",
    "\n",
    "\n",
    "# Chain\n",
    "rag_chain = prompt | llm | StrOutputParser()\n",
    "\n",
    "# Run\n",
    "question = \"agent memory\"\n",
    "generation = rag_chain.invoke({\"context\": docs, \"question\": question})\n",
    "print(generation)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 9,
   "id": "38345cff-e2d0-436e-aa09-599522a61eed",
   "metadata": {},
   "outputs": [
    {
     "data": {
      "text/plain": [
       "{'score': 'yes'}"
      ]
     },
     "execution_count": 9,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "### Hallucination Grader\n",
    "\n",
    "# LLM\n",
    "llm = ChatOllama(model=local_llm, format=\"json\", temperature=0)\n",
    "\n",
    "# Prompt\n",
    "prompt = PromptTemplate(\n",
    "    template=\"\"\"You are a grader assessing whether an answer is grounded in / supported by a set of facts. \\n \n",
    "    Here are the facts:\n",
    "    \\n ------- \\n\n",
    "    {documents} \n",
    "    \\n ------- \\n\n",
    "    Here is the answer: {generation}\n",
    "    Give a binary score 'yes' or 'no' score to indicate whether the answer is grounded in / supported by a set of facts. \\n\n",
    "    Provide the binary score as a JSON with a single key 'score' and no preamble or explanation.\"\"\",\n",
    "    input_variables=[\"generation\", \"documents\"],\n",
    ")\n",
    "\n",
    "hallucination_grader = prompt | llm | JsonOutputParser()\n",
    "hallucination_grader.invoke({\"documents\": docs, \"generation\": generation})"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 10,
   "id": "9771caa1-5542-47c3-8354-aeeafcf51964",
   "metadata": {},
   "outputs": [
    {
     "data": {
      "text/plain": [
       "{'score': 'yes'}"
      ]
     },
     "execution_count": 10,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "### Answer Grader\n",
    "\n",
    "# LLM\n",
    "llm = ChatOllama(model=local_llm, format=\"json\", temperature=0)\n",
    "\n",
    "# Prompt\n",
    "prompt = PromptTemplate(\n",
    "    template=\"\"\"You are a grader assessing whether an answer is useful to resolve a question. \\n \n",
    "    Here is the answer:\n",
    "    \\n ------- \\n\n",
    "    {generation} \n",
    "    \\n ------- \\n\n",
    "    Here is the question: {question}\n",
    "    Give a binary score 'yes' or 'no' to indicate whether the answer is useful to resolve a question. \\n\n",
    "    Provide the binary score as a JSON with a single key 'score' and no preamble or explanation.\"\"\",\n",
    "    input_variables=[\"generation\", \"question\"],\n",
    ")\n",
    "\n",
    "answer_grader = prompt | llm | JsonOutputParser()\n",
    "answer_grader.invoke({\"question\": question, \"generation\": generation})"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 11,
   "id": "830ba5f7-9c8d-4c01-83b1-e4d51d40d48f",
   "metadata": {},
   "outputs": [
    {
     "data": {
      "text/plain": [
       "' What is agent memory and how can it be effectively utilized in vector database retrieval?'"
      ]
     },
     "execution_count": 11,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "### Question Re-writer\n",
    "\n",
    "# LLM\n",
    "llm = ChatOllama(model=local_llm, temperature=0)\n",
    "\n",
    "# Prompt\n",
    "re_write_prompt = PromptTemplate(\n",
    "    template=\"\"\"You a question re-writer that converts an input question to a better version that is optimized \\n \n",
    "     for vectorstore retrieval. Look at the initial and formulate an improved question. \\n\n",
    "     Here is the initial question: \\n\\n {question}. Improved question with no preamble: \\n \"\"\",\n",
    "    input_variables=[\"generation\", \"question\"],\n",
    ")\n",
    "\n",
    "question_rewriter = re_write_prompt | llm | StrOutputParser()\n",
    "question_rewriter.invoke({\"question\": question})"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "686c9bb1-5069-45f9-8a7e-cba34fe07dd9",
   "metadata": {},
   "source": [
    "## Web Search Tool"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 12,
   "id": "6c3c1c70-ff84-41e8-bf72-738ed52f2dde",
   "metadata": {},
   "outputs": [],
   "source": [
    "### Search\n",
    "\n",
    "from langchain_community.tools.tavily_search import TavilySearchResults\n",
    "\n",
    "web_search_tool = TavilySearchResults(k=3)"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "630d1751-a20b-4858-b3fd-0312de4f3ad7",
   "metadata": {},
   "source": [
    "# Graph \n",
    "\n",
    "Capture the flow in as a graph.\n",
    "\n",
    "## Graph state"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 13,
   "id": "6e09087e-b2a9-437a-abee-129e426df799",
   "metadata": {},
   "outputs": [],
   "source": [
    "from typing import List\n",
    "\n",
    "from typing_extensions import TypedDict\n",
    "\n",
    "\n",
    "class GraphState(TypedDict):\n",
    "    \"\"\"\n",
    "    Represents the state of our graph.\n",
    "\n",
    "    Attributes:\n",
    "        question: question\n",
    "        generation: LLM generation\n",
    "        documents: list of documents\n",
    "    \"\"\"\n",
    "\n",
    "    question: str\n",
    "    generation: str\n",
    "    documents: List[str]"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 14,
   "id": "7c5fa507-77ae-426a-a65f-f518b9525bd0",
   "metadata": {},
   "outputs": [],
   "source": [
    "### Nodes\n",
    "\n",
    "from langchain.schema import Document\n",
    "\n",
    "\n",
    "def retrieve(state):\n",
    "    \"\"\"\n",
    "    Retrieve documents\n",
    "\n",
    "    Args:\n",
    "        state (dict): The current graph state\n",
    "\n",
    "    Returns:\n",
    "        state (dict): New key added to state, documents, that contains retrieved documents\n",
    "    \"\"\"\n",
    "    print(\"---RETRIEVE---\")\n",
    "    question = state[\"question\"]\n",
    "\n",
    "    # Retrieval\n",
    "    documents = retriever.get_relevant_documents(question)\n",
    "    return {\"documents\": documents, \"question\": question}\n",
    "\n",
    "\n",
    "def generate(state):\n",
    "    \"\"\"\n",
    "    Generate answer\n",
    "\n",
    "    Args:\n",
    "        state (dict): The current graph state\n",
    "\n",
    "    Returns:\n",
    "        state (dict): New key added to state, generation, that contains LLM generation\n",
    "    \"\"\"\n",
    "    print(\"---GENERATE---\")\n",
    "    question = state[\"question\"]\n",
    "    documents = state[\"documents\"]\n",
    "\n",
    "    # RAG generation\n",
    "    generation = rag_chain.invoke({\"context\": documents, \"question\": question})\n",
    "    return {\"documents\": documents, \"question\": question, \"generation\": generation}\n",
    "\n",
    "\n",
    "def grade_documents(state):\n",
    "    \"\"\"\n",
    "    Determines whether the retrieved documents are relevant to the question.\n",
    "\n",
    "    Args:\n",
    "        state (dict): The current graph state\n",
    "\n",
    "    Returns:\n",
    "        state (dict): Updates documents key with only filtered relevant documents\n",
    "    \"\"\"\n",
    "\n",
    "    print(\"---CHECK DOCUMENT RELEVANCE TO QUESTION---\")\n",
    "    question = state[\"question\"]\n",
    "    documents = state[\"documents\"]\n",
    "\n",
    "    # Score each doc\n",
    "    filtered_docs = []\n",
    "    for d in documents:\n",
    "        score = retrieval_grader.invoke(\n",
    "            {\"question\": question, \"document\": d.page_content}\n",
    "        )\n",
    "        grade = score[\"score\"]\n",
    "        if grade == \"yes\":\n",
    "            print(\"---GRADE: DOCUMENT RELEVANT---\")\n",
    "            filtered_docs.append(d)\n",
    "        else:\n",
    "            print(\"---GRADE: DOCUMENT NOT RELEVANT---\")\n",
    "            continue\n",
    "    return {\"documents\": filtered_docs, \"question\": question}\n",
    "\n",
    "\n",
    "def transform_query(state):\n",
    "    \"\"\"\n",
    "    Transform the query to produce a better question.\n",
    "\n",
    "    Args:\n",
    "        state (dict): The current graph state\n",
    "\n",
    "    Returns:\n",
    "        state (dict): Updates question key with a re-phrased question\n",
    "    \"\"\"\n",
    "\n",
    "    print(\"---TRANSFORM QUERY---\")\n",
    "    question = state[\"question\"]\n",
    "    documents = state[\"documents\"]\n",
    "\n",
    "    # Re-write question\n",
    "    better_question = question_rewriter.invoke({\"question\": question})\n",
    "    return {\"documents\": documents, \"question\": better_question}\n",
    "\n",
    "\n",
    "def web_search(state):\n",
    "    \"\"\"\n",
    "    Web search based on the re-phrased question.\n",
    "\n",
    "    Args:\n",
    "        state (dict): The current graph state\n",
    "\n",
    "    Returns:\n",
    "        state (dict): Updates documents key with appended web results\n",
    "    \"\"\"\n",
    "\n",
    "    print(\"---WEB SEARCH---\")\n",
    "    question = state[\"question\"]\n",
    "\n",
    "    # Web search\n",
    "    docs = web_search_tool.invoke({\"query\": question})\n",
    "    web_results = \"\\n\".join([d[\"content\"] for d in docs])\n",
    "    web_results = Document(page_content=web_results)\n",
    "\n",
    "    return {\"documents\": web_results, \"question\": question}\n",
    "\n",
    "\n",
    "### Edges ###\n",
    "\n",
    "\n",
    "def route_question(state):\n",
    "    \"\"\"\n",
    "    Route question to web search or RAG.\n",
    "\n",
    "    Args:\n",
    "        state (dict): The current graph state\n",
    "\n",
    "    Returns:\n",
    "        str: Next node to call\n",
    "    \"\"\"\n",
    "\n",
    "    print(\"---ROUTE QUESTION---\")\n",
    "    question = state[\"question\"]\n",
    "    print(question)\n",
    "    source = question_router.invoke({\"question\": question})\n",
    "    print(source)\n",
    "    print(source[\"datasource\"])\n",
    "    if source[\"datasource\"] == \"web_search\":\n",
    "        print(\"---ROUTE QUESTION TO WEB SEARCH---\")\n",
    "        return \"web_search\"\n",
    "    elif source[\"datasource\"] == \"vectorstore\":\n",
    "        print(\"---ROUTE QUESTION TO RAG---\")\n",
    "        return \"vectorstore\"\n",
    "\n",
    "\n",
    "def decide_to_generate(state):\n",
    "    \"\"\"\n",
    "    Determines whether to generate an answer, or re-generate a question.\n",
    "\n",
    "    Args:\n",
    "        state (dict): The current graph state\n",
    "\n",
    "    Returns:\n",
    "        str: Binary decision for next node to call\n",
    "    \"\"\"\n",
    "\n",
    "    print(\"---ASSESS GRADED DOCUMENTS---\")\n",
    "    state[\"question\"]\n",
    "    filtered_documents = state[\"documents\"]\n",
    "\n",
    "    if not filtered_documents:\n",
    "        # All documents have been filtered check_relevance\n",
    "        # We will re-generate a new query\n",
    "        print(\n",
    "            \"---DECISION: ALL DOCUMENTS ARE NOT RELEVANT TO QUESTION, TRANSFORM QUERY---\"\n",
    "        )\n",
    "        return \"transform_query\"\n",
    "    else:\n",
    "        # We have relevant documents, so generate answer\n",
    "        print(\"---DECISION: GENERATE---\")\n",
    "        return \"generate\"\n",
    "\n",
    "\n",
    "def grade_generation_v_documents_and_question(state):\n",
    "    \"\"\"\n",
    "    Determines whether the generation is grounded in the document and answers question.\n",
    "\n",
    "    Args:\n",
    "        state (dict): The current graph state\n",
    "\n",
    "    Returns:\n",
    "        str: Decision for next node to call\n",
    "    \"\"\"\n",
    "\n",
    "    print(\"---CHECK HALLUCINATIONS---\")\n",
    "    question = state[\"question\"]\n",
    "    documents = state[\"documents\"]\n",
    "    generation = state[\"generation\"]\n",
    "\n",
    "    score = hallucination_grader.invoke(\n",
    "        {\"documents\": documents, \"generation\": generation}\n",
    "    )\n",
    "    grade = score[\"score\"]\n",
    "\n",
    "    # Check hallucination\n",
    "    if grade == \"yes\":\n",
    "        print(\"---DECISION: GENERATION IS GROUNDED IN DOCUMENTS---\")\n",
    "        # Check question-answering\n",
    "        print(\"---GRADE GENERATION vs QUESTION---\")\n",
    "        score = answer_grader.invoke({\"question\": question, \"generation\": generation})\n",
    "        grade = score[\"score\"]\n",
    "        if grade == \"yes\":\n",
    "            print(\"---DECISION: GENERATION ADDRESSES QUESTION---\")\n",
    "            return \"useful\"\n",
    "        else:\n",
    "            print(\"---DECISION: GENERATION DOES NOT ADDRESS QUESTION---\")\n",
    "            return \"not useful\"\n",
    "    else:\n",
    "        pprint(\"---DECISION: GENERATION IS NOT GROUNDED IN DOCUMENTS, RE-TRY---\")\n",
    "        return \"not supported\""
   ]
  },
  {
   "cell_type": "markdown",
   "id": "ed7d8eb6-31d7-4ab5-8a88-7081b64582bb",
   "metadata": {},
   "source": [
    "## Build Graph"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 15,
   "id": "450eb313-ca75-4a43-b57e-7034bd3f40bf",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langgraph.graph import END, StateGraph, START\n",
    "\n",
    "workflow = StateGraph(GraphState)\n",
    "\n",
    "# Define the nodes\n",
    "workflow.add_node(\"web_search\", web_search)  # web search\n",
    "workflow.add_node(\"retrieve\", retrieve)  # retrieve\n",
    "workflow.add_node(\"grade_documents\", grade_documents)  # grade documents\n",
    "workflow.add_node(\"generate\", generate)  # generate\n",
    "workflow.add_node(\"transform_query\", transform_query)  # transform_query\n",
    "\n",
    "# Build graph\n",
    "workflow.add_conditional_edges(\n",
    "    START,\n",
    "    route_question,\n",
    "    {\n",
    "        \"web_search\": \"web_search\",\n",
    "        \"vectorstore\": \"retrieve\",\n",
    "    },\n",
    ")\n",
    "workflow.add_edge(\"web_search\", \"generate\")\n",
    "workflow.add_edge(\"retrieve\", \"grade_documents\")\n",
    "workflow.add_conditional_edges(\n",
    "    \"grade_documents\",\n",
    "    decide_to_generate,\n",
    "    {\n",
    "        \"transform_query\": \"transform_query\",\n",
    "        \"generate\": \"generate\",\n",
    "    },\n",
    ")\n",
    "workflow.add_edge(\"transform_query\", \"retrieve\")\n",
    "workflow.add_conditional_edges(\n",
    "    \"generate\",\n",
    "    grade_generation_v_documents_and_question,\n",
    "    {\n",
    "        \"not supported\": \"generate\",\n",
    "        \"useful\": END,\n",
    "        \"not useful\": \"transform_query\",\n",
    "    },\n",
    ")\n",
    "\n",
    "# Compile\n",
    "app = workflow.compile()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 16,
   "id": "b095c1db-8bd1-4a34-937c-1a9b74ae74ff",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "---ROUTE QUESTION---\n",
      "What is the AlphaCodium paper about?\n",
      "{'datasource': 'web_search'}\n",
      "web_search\n",
      "---ROUTE QUESTION TO WEB SEARCH---\n",
      "---WEB SEARCH---\n",
      "\"Node 'web_search':\"\n",
      "'\\n---\\n'\n",
      "---GENERATE---\n",
      "---CHECK HALLUCINATIONS---\n",
      "---DECISION: GENERATION IS GROUNDED IN DOCUMENTS---\n",
      "---GRADE GENERATION vs QUESTION---\n",
      "---DECISION: GENERATION ADDRESSES QUESTION---\n",
      "\"Node 'generate':\"\n",
      "'\\n---\\n'\n",
      "(' The AlphaCodium paper introduces a new approach for code generation by '\n",
      " 'Large Language Models (LLMs). It presents AlphaCodium, an iterative process '\n",
      " 'that involves generating additional data to aid the flow, and testing it on '\n",
      " 'the CodeContests dataset. The results show that AlphaCodium outperforms '\n",
      " \"DeepMind's AlphaCode and AlphaCode2 without fine-tuning a model. The \"\n",
      " 'approach includes a pre-processing phase for problem reasoning in natural '\n",
      " 'language and an iterative code generation phase with runs and fixes against '\n",
      " 'tests.')\n"
     ]
    }
   ],
   "source": [
    "from pprint import pprint\n",
    "\n",
    "# Run\n",
    "inputs = {\"question\": \"What is the AlphaCodium paper about?\"}\n",
    "for output in app.stream(inputs):\n",
    "    for key, value in output.items():\n",
    "        # Node\n",
    "        pprint(f\"Node '{key}':\")\n",
    "        # Optional: print full state at each node\n",
    "        # pprint.pprint(value[\"keys\"], indent=2, width=80, depth=None)\n",
    "    pprint(\"\\n---\\n\")\n",
    "\n",
    "# Final generation\n",
    "pprint(value[\"generation\"])"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "644c7293-9cb5-4236-ba08-1e63b0309cb7",
   "metadata": {},
   "source": [
    "Trace: \n",
    "\n",
    "https://smith.langchain.com/public/81813813-be53-403c-9877-afcd5786ca2e/r"
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
   "version": "3.11.8"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 5
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/rag/langgraph_agentic_rag.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "425fb020-e864-40ce-a31f-8da40c73d14b",
   "metadata": {},
   "source": [
    "# Agentic RAG\n",
    "\n",
    "[Retrieval Agents](https://python.langchain.com/v0.2/docs/tutorials/qa_chat_history/#agents) are useful when we want to make decisions about whether to retrieve from an index.\n",
    "\n",
    "To implement a retrieval agent, we simple need to give an LLM access to a retriever tool.\n",
    "\n",
    "We can incorporate this into [LangGraph](https://langchain-ai.github.io/langgraph/).\n",
    "\n",
    "## Setup\n",
    "\n",
    "First, let's download the required packages and set our API keys:"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "969fb438",
   "metadata": {},
   "outputs": [],
   "source": [
    "%%capture --no-stderr\n",
    "%pip install -U --quiet langchain-community tiktoken langchain-openai langchainhub chromadb langchain langgraph langchain-text-splitters"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 1,
   "id": "e4958a8c",
   "metadata": {},
   "outputs": [],
   "source": [
    "import getpass\n",
    "import os\n",
    "\n",
    "\n",
    "def _set_env(key: str):\n",
    "    if key not in os.environ:\n",
    "        os.environ[key] = getpass.getpass(f\"{key}:\")\n",
    "\n",
    "\n",
    "_set_env(\"OPENAI_API_KEY\")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "3d07e8d4",
   "metadata": {},
   "source": [
    "<div class=\"admonition tip\">\n",
    "    <p class=\"admonition-title\">Set up <a href=\"https://smith.langchain.com\">LangSmith</a> for LangGraph development</p>\n",
    "    <p style=\"padding-top: 5px;\">\n",
    "        Sign up for LangSmith to quickly spot issues and improve the performance of your LangGraph projects. LangSmith lets you use trace data to debug, test, and monitor your LLM apps built with LangGraph — read more about how to get started <a href=\"https://docs.smith.langchain.com\">here</a>. \n",
    "    </p>\n",
    "</div>    "
   ]
  },
  {
   "cell_type": "markdown",
   "id": "c74e4532",
   "metadata": {},
   "source": [
    "## Retriever\n",
    "\n",
    "First, we index 3 blog posts."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 2,
   "id": "e50c9efe-4abe-42fa-b35a-05eeeede9ec6",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langchain_community.document_loaders import WebBaseLoader\n",
    "from langchain_community.vectorstores import Chroma\n",
    "from langchain_openai import OpenAIEmbeddings\n",
    "from langchain_text_splitters import RecursiveCharacterTextSplitter\n",
    "\n",
    "urls = [\n",
    "    \"https://lilianweng.github.io/posts/2023-06-23-agent/\",\n",
    "    \"https://lilianweng.github.io/posts/2023-03-15-prompt-engineering/\",\n",
    "    \"https://lilianweng.github.io/posts/2023-10-25-adv-attack-llm/\",\n",
    "]\n",
    "\n",
    "docs = [WebBaseLoader(url).load() for url in urls]\n",
    "docs_list = [item for sublist in docs for item in sublist]\n",
    "\n",
    "text_splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(\n",
    "    chunk_size=100, chunk_overlap=50\n",
    ")\n",
    "doc_splits = text_splitter.split_documents(docs_list)\n",
    "\n",
    "# Add to vectorDB\n",
    "vectorstore = Chroma.from_documents(\n",
    "    documents=doc_splits,\n",
    "    collection_name=\"rag-chroma\",\n",
    "    embedding=OpenAIEmbeddings(),\n",
    ")\n",
    "retriever = vectorstore.as_retriever()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "225d2277-45b2-4ae8-a7d6-62b07fb4a002",
   "metadata": {},
   "source": [
    "Then we create a retriever tool."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 3,
   "id": "0b97bdd8-d7e3-444d-ac96-5ef4725f9048",
   "metadata": {},
   "outputs": [],
   "source": [
    "from langchain.tools.retriever import create_retriever_tool\n",
    "\n",
    "retriever_tool = create_retriever_tool(\n",
    "    retriever,\n",
    "    \"retrieve_blog_posts\",\n",
    "    \"Search and return information about Lilian Weng blog posts on LLM agents, prompt engineering, and adversarial attacks on LLMs.\",\n",
    ")\n",
    "\n",
    "tools = [retriever_tool]"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "fe6e8f78-1ef7-42ad-b2bf-835ed5850553",
   "metadata": {},
   "source": [
    "## Agent State\n",
    " \n",
    "We will define a graph.\n",
    "\n",
    "A `state` object that it passes around to each node.\n",
    "\n",
    "Our state will be a list of `messages`.\n",
    "\n",
    "Each node in our graph will append to it."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 4,
   "id": "0e378706-47d5-425a-8ba0-57b9acffbd0c",
   "metadata": {},
   "outputs": [],
   "source": [
    "from typing import Annotated, Sequence, TypedDict\n",
    "\n",
    "from langchain_core.messages import BaseMessage\n",
    "\n",
    "from langgraph.graph.message import add_messages\n",
    "\n",
    "\n",
    "class AgentState(TypedDict):\n",
    "    # The add_messages function defines how an update should be processed\n",
    "    # Default is to replace. add_messages says \"append\"\n",
    "    messages: Annotated[Sequence[BaseMessage], add_messages]"
   ]
  },
  {
   "attachments": {
    "7ad1a116-28d7-473f-8cff-5f2efd0bf118.png": {
