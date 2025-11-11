    }
   },
   "cell_type": "markdown",
   "id": "5afcaed0-3d55-4e1f-95d3-c32c751c29d8",
   "metadata": {
    "id": "5afcaed0-3d55-4e1f-95d3-c32c751c29d8"
   },
   "source": [
    "# Adaptive RAG Cohere Command R\n",
    "\n",
    "Adaptive RAG is a strategy for RAG that unites (1) [query analysis](https://blog.langchain.dev/query-construction/) with (2) [active / self-corrective RAG](https://blog.langchain.dev/agentic-rag-with-langgraph/).\n",
    "\n",
    "In the paper, they report query analysis to route across:\n",
    "\n",
    "* No Retrieval (LLM answers)\n",
    "* Single-shot RAG\n",
    "* Iterative RAG\n",
    "\n",
    "Let's build on this to perform query analysis to route across some more interesting cases:\n",
    "\n",
    "* No Retrieval (LLM answers)\n",
    "* Web-search\n",
    "* Iterative RAG\n",
    "\n",
    "We'll use [Command R](https://cohere.com/blog/command-r), a recent release from Cohere that:\n",
    "\n",
    "* Has strong accuracy on RAG and Tool Use\n",
    "* Has 128k context\n",
    "* Has low latency \n",
    "  \n",
    "![Screenshot 2024-04-02 at 8.11.18 PM.png](attachment:2a4ecdd2-280d-4311-a2cd-cd6138090be9.png)"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "a85501ca-eb89-4795-aeab-cdab050ead6b",
   "metadata": {
    "id": "a85501ca-eb89-4795-aeab-cdab050ead6b"
   },
   "source": [
    "# Environment"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "f6c329ba-cb85-4576-9828-4f2ac648d1a6",
   "metadata": {},
   "outputs": [],
   "source": [
    "! pip install --quiet langchain langchain_cohere langchain-openai tiktoken langchainhub chromadb langgraph"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "222f204d-956f-4128-b597-2c698120edda",
   "metadata": {
    "id": "222f204d-956f-4128-b597-2c698120edda"
   },
   "outputs": [],
   "source": [
    "### LLMs\nimport os\n\nos.environ[\"COHERE_API_KEY\"] = \"<your-api-key>\""
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "08edba00-988a-478b-96fc-ae0199cbef49",
   "metadata": {
    "id": "08edba00-988a-478b-96fc-ae0199cbef49"
   },
   "outputs": [],
   "source": [
    "# ### Tracing (optional)\n# os.environ['LANGCHAIN_TRACING_V2'] = 'true'\n# os.environ['LANGCHAIN_ENDPOINT'] = 'https://api.smith.langchain.com'\n# os.environ['LANGCHAIN_API_KEY'] ='<your-api-key>'"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "9ac1c2cd-81fb-40eb-8ba1-e9197800cba6",
   "metadata": {
    "id": "9ac1c2cd-81fb-40eb-8ba1-e9197800cba6"
   },
   "source": [
    "## Index"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 1,
   "id": "b224e5ba-50ca-495a-a7fa-0f75a080e03c",
   "metadata": {
    "id": "b224e5ba-50ca-495a-a7fa-0f75a080e03c"
   },
   "outputs": [],
   "source": [
    "### Build Index\n\nfrom langchain.text_splitter import RecursiveCharacterTextSplitter\nfrom langchain_cohere import CohereEmbeddings\nfrom langchain_community.document_loaders import WebBaseLoader\nfrom langchain_community.vectorstores import Chroma\n\n# Set embeddings\nembd = CohereEmbeddings()\n\n# Docs to index\nurls = [\n    \"https://lilianweng.github.io/posts/2023-06-23-agent/\",\n    \"https://lilianweng.github.io/posts/2023-03-15-prompt-engineering/\",\n    \"https://lilianweng.github.io/posts/2023-10-25-adv-attack-llm/\",\n]\n\n# Load\ndocs = [WebBaseLoader(url).load() for url in urls]\ndocs_list = [item for sublist in docs for item in sublist]\n\n# Split\ntext_splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(\n    chunk_size=512, chunk_overlap=0\n)\ndoc_splits = text_splitter.split_documents(docs_list)\n\n# Add to vectorstore\nvectorstore = Chroma.from_documents(\n    documents=doc_splits,\n    embedding=embd,\n)\n\nretriever = vectorstore.as_retriever()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "0f52b427-750c-40f8-8893-e9caab3afd8d",
   "metadata": {
    "id": "0f52b427-750c-40f8-8893-e9caab3afd8d"
   },
   "source": [
    "## LLMs"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "H5CztTqsBOTZ",
   "metadata": {
    "id": "H5CztTqsBOTZ"
   },
   "source": [
    "We use a router to pick between tools. \n",
    " \n",
    "Cohere model decides which tool(s) to call, as well as the how to query them."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 2,
   "id": "bYK-e0diGdPf",
   "metadata": {
    "colab": {
     "base_uri": "https://localhost:8080/"
    },
    "id": "bYK-e0diGdPf",
    "outputId": "895a8ea5-57ee-49fe-ef28-277eac8a7bb7"
   },
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "[{'id': 'f811e3b9-052e-49db-a234-5fc3efbcc5ba', 'function': {'name': 'web_search', 'arguments': '{\"query\": \"NFL draft bears first pick\"}'}, 'type': 'function'}]\n",
      "[{'id': '4bc53113-8f32-4d6d-ac9b-c07ef9aae9fd', 'function': {'name': 'vectorstore', 'arguments': '{\"query\": \"types of agent memory\"}'}, 'type': 'function'}]\n",
      "False\n"
     ]
    }
   ],
   "source": [
    "### Router\n\nfrom langchain_cohere import ChatCohere\nfrom langchain_core.prompts import ChatPromptTemplate\nfrom langchain_core.pydantic_v1 import BaseModel, Field\n\n\n# Data model\nclass web_search(BaseModel):\n    \"\"\"\n    The internet. Use web_search for questions that are related to anything else than agents, prompt engineering, and adversarial attacks.\n    \"\"\"\n\n    query: str = Field(description=\"The query to use when searching the internet.\")\n\n\nclass vectorstore(BaseModel):\n    \"\"\"\n    A vectorstore containing documents related to agents, prompt engineering, and adversarial attacks. Use the vectorstore for questions on these topics.\n    \"\"\"\n\n    query: str = Field(description=\"The query to use when searching the vectorstore.\")\n\n\n# Preamble\npreamble = \"\"\"You are an expert at routing a user question to a vectorstore or web search.\nThe vectorstore contains documents related to agents, prompt engineering, and adversarial attacks.\nUse the vectorstore for questions on these topics. Otherwise, use web-search.\"\"\"\n\n# LLM with tool use and preamble\nllm = ChatCohere(model=\"command-r\", temperature=0)\nstructured_llm_router = llm.bind_tools(\n    tools=[web_search, vectorstore], preamble=preamble\n)\n\n# Prompt\nroute_prompt = ChatPromptTemplate.from_messages(\n    [\n        (\"human\", \"{question}\"),\n    ]\n)\n\nquestion_router = route_prompt | structured_llm_router\nresponse = question_router.invoke(\n    {\"question\": \"Who will the Bears draft first in the NFL draft?\"}\n)\nprint(response.response_metadata[\"tool_calls\"])\nresponse = question_router.invoke({\"question\": \"What are the types of agent memory?\"})\nprint(response.response_metadata[\"tool_calls\"])\nresponse = question_router.invoke({\"question\": \"Hi how are you?\"})\nprint(\"tool_calls\" in response.response_metadata)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 4,
   "id": "oaLWNbWxBgjE",
   "metadata": {
    "colab": {
     "base_uri": "https://localhost:8080/"
    },
    "id": "oaLWNbWxBgjE",
    "outputId": "57a5c27b-044b-4df5-f55d-7bf23d3976d1"
   },
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "binary_score='yes'\n"
     ]
    }
   ],
   "source": [
    "### Retrieval Grader\n\n\n# Data model\nclass GradeDocuments(BaseModel):\n    \"\"\"Binary score for relevance check on retrieved documents.\"\"\"\n\n    binary_score: str = Field(\n        description=\"Documents are relevant to the question, 'yes' or 'no'\"\n    )\n\n\n# Prompt\npreamble = \"\"\"You are a grader assessing relevance of a retrieved document to a user question. \\n\nIf the document contains keyword(s) or semantic meaning related to the user question, grade it as relevant. \\n\nGive a binary score 'yes' or 'no' score to indicate whether the document is relevant to the question.\"\"\"\n\n# LLM with function call\nllm = ChatCohere(model=\"command-r\", temperature=0)\nstructured_llm_grader = llm.with_structured_output(GradeDocuments, preamble=preamble)\n\ngrade_prompt = ChatPromptTemplate.from_messages(\n    [\n        (\"human\", \"Retrieved document: \\n\\n {document} \\n\\n User question: {question}\"),\n    ]\n)\n\nretrieval_grader = grade_prompt | structured_llm_grader\nquestion = \"types of agent memory\"\ndocs = retriever.invoke(question)\ndoc_txt = docs[1].page_content\nresponse = retrieval_grader.invoke({\"question\": question, \"document\": doc_txt})\nprint(response)"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "D43a7vM4EElX",
   "metadata": {
    "id": "D43a7vM4EElX"
   },
   "source": [
    "Generate"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 5,
   "id": "BTIUdjRMEq_h",
   "metadata": {
    "colab": {
     "base_uri": "https://localhost:8080/"
    },
    "id": "BTIUdjRMEq_h",
    "outputId": "11a99f62-2449-45db-9281-5bfd60e3c966"
   },
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "There are three types of agent memory: sensory memory, short-term memory, and long-term memory.\n"
     ]
    }
   ],
   "source": [
    "### Generate\n\nfrom langchain_core.messages import HumanMessage\nfrom langchain_core.output_parsers import StrOutputParser\n\n# Preamble\npreamble = \"\"\"You are an assistant for question-answering tasks. Use the following pieces of retrieved context to answer the question. If you don't know the answer, just say that you don't know. Use three sentences maximum and keep the answer concise.\"\"\"\n\n# LLM\nllm = ChatCohere(model_name=\"command-r\", temperature=0).bind(preamble=preamble)\n\n\n# Prompt\ndef prompt(x):\n    return ChatPromptTemplate.from_messages(\n        [\n            HumanMessage(\n                f\"Question: {x['question']} \\nAnswer: \",\n                additional_kwargs={\"documents\": x[\"documents\"]},\n            )\n        ]\n    )\n\n\n# Chain\nrag_chain = prompt | llm | StrOutputParser()\n\n# Run\ngeneration = rag_chain.invoke({\"documents\": docs, \"question\": question})\nprint(generation)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 6,
   "id": "bc000a7d-84b6-4eb2-88ad-65cc62a44431",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "I don't have feelings as an AI chatbot, but I'm here to assist you with any queries or concerns you may have. How can I help you today?\n"
     ]
    }
   ],
   "source": [
    "### LLM fallback\n\nfrom langchain_core.output_parsers import StrOutputParser\n\n# Preamble\npreamble = \"\"\"You are an assistant for question-answering tasks. Answer the question based upon your knowledge. Use three sentences maximum and keep the answer concise.\"\"\"\n\n# LLM\nllm = ChatCohere(model_name=\"command-r\", temperature=0).bind(preamble=preamble)\n\n\n# Prompt\ndef prompt(x):\n    return ChatPromptTemplate.from_messages(\n        [HumanMessage(f\"Question: {x['question']} \\nAnswer: \")]\n    )\n\n\n# Chain\nllm_chain = prompt | llm | StrOutputParser()\n\n# Run\nquestion = \"Hi how are you?\"\ngeneration = llm_chain.invoke({\"question\": question})\nprint(generation)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 7,
   "id": "y0msuR2DHQkY",
   "metadata": {
    "colab": {
     "base_uri": "https://localhost:8080/"
    },
    "id": "y0msuR2DHQkY",
    "outputId": "f0e91c2a-5542-45c0-a7a6-60453e0b2bc4"
   },
   "outputs": [
    {
     "data": {
      "text/plain": [
       "GradeHallucinations(binary_score='yes')"
      ]
     },
     "execution_count": 7,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "### Hallucination Grader\n\n\n# Data model\nclass GradeHallucinations(BaseModel):\n    \"\"\"Binary score for hallucination present in generation answer.\"\"\"\n\n    binary_score: str = Field(\n        description=\"Answer is grounded in the facts, 'yes' or 'no'\"\n    )\n\n\n# Preamble\npreamble = \"\"\"You are a grader assessing whether an LLM generation is grounded in / supported by a set of retrieved facts. \\n\nGive a binary score 'yes' or 'no'. 'Yes' means that the answer is grounded in / supported by the set of facts.\"\"\"\n\n# LLM with function call\nllm = ChatCohere(model=\"command-r\", temperature=0)\nstructured_llm_grader = llm.with_structured_output(\n    GradeHallucinations, preamble=preamble\n)\n\n# Prompt\nhallucination_prompt = ChatPromptTemplate.from_messages(\n    [\n        # (\"system\", system),\n        (\"human\", \"Set of facts: \\n\\n {documents} \\n\\n LLM generation: {generation}\"),\n    ]\n)\n\nhallucination_grader = hallucination_prompt | structured_llm_grader\nhallucination_grader.invoke({\"documents\": docs, \"generation\": generation})"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 8,
   "id": "f0c08d14-77a0-4eed-b882-2d636abb22a3",
   "metadata": {
    "colab": {
     "base_uri": "https://localhost:8080/"
    },
    "id": "f0c08d14-77a0-4eed-b882-2d636abb22a3",
    "outputId": "c4f88c9a-65fd-4dad-e739-3c9bd547a9f5"
   },
   "outputs": [
    {
     "data": {
      "text/plain": [
       "GradeAnswer(binary_score='yes')"
      ]
     },
     "execution_count": 8,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "### Answer Grader\n\n\n# Data model\nclass GradeAnswer(BaseModel):\n    \"\"\"Binary score to assess answer addresses question.\"\"\"\n\n    binary_score: str = Field(\n        description=\"Answer addresses the question, 'yes' or 'no'\"\n    )\n\n\n# Preamble\npreamble = \"\"\"You are a grader assessing whether an answer addresses / resolves a question \\n\nGive a binary score 'yes' or 'no'. Yes' means that the answer resolves the question.\"\"\"\n\n# LLM with function call\nllm = ChatCohere(model=\"command-r\", temperature=0)\nstructured_llm_grader = llm.with_structured_output(GradeAnswer, preamble=preamble)\n\n# Prompt\nanswer_prompt = ChatPromptTemplate.from_messages(\n    [\n        (\"human\", \"User question: \\n\\n {question} \\n\\n LLM generation: {generation}\"),\n    ]\n)\n\nanswer_grader = answer_prompt | structured_llm_grader\nanswer_grader.invoke({\"question\": question, \"generation\": generation})"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "d07c0b31-b919-4498-869f-9673125c2473",
   "metadata": {
    "id": "d07c0b31-b919-4498-869f-9673125c2473"
   },
   "source": [
    "## Web Search Tool"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 9,
   "id": "01d829bb-1074-4976-b650-ead41dcb9788",
   "metadata": {
    "id": "01d829bb-1074-4976-b650-ead41dcb9788"
   },
   "outputs": [],
   "source": [
    "### Search\n# os.environ['TAVILY_API_KEY'] ='<your-api-key>'\n\nfrom langchain_community.tools.tavily_search import TavilySearchResults\n\nweb_search_tool = TavilySearchResults()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "efbbff0e-8843-45bb-b2ff-137bef707ef4",
   "metadata": {
    "id": "efbbff0e-8843-45bb-b2ff-137bef707ef4"
   },
   "source": [
    "# Graph\n",
    "\n",
    "Capture the flow in as a graph.\n",
    "\n",
    "## Graph state"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 10,
   "id": "e723fcdb-06e6-402d-912e-899795b78408",
   "metadata": {
    "id": "e723fcdb-06e6-402d-912e-899795b78408"
   },
   "outputs": [],
   "source": [
    "from typing import List\n\nfrom typing_extensions import TypedDict\n\n\nclass GraphState(TypedDict):\n    \"\"\"|\n    Represents the state of our graph.\n\n    Attributes:\n        question: question\n        generation: LLM generation\n        documents: list of documents\n    \"\"\"\n\n    question: str\n    generation: str\n    documents: List[str]"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "7e2d6c0d-42e8-4399-9751-e315be16607a",
   "metadata": {
    "id": "7e2d6c0d-42e8-4399-9751-e315be16607a"
   },
   "source": [
    "## Graph Flow"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 11,
   "id": "b76b5ec3-0720-443d-85b1-c0e79659ca0a",
   "metadata": {
    "id": "b76b5ec3-0720-443d-85b1-c0e79659ca0a"
   },
   "outputs": [],
   "source": [
    "from langchain.schema import Document\n\n\ndef retrieve(state):\n    \"\"\"\n    Retrieve documents\n\n    Args:\n        state (dict): The current graph state\n\n    Returns:\n        state (dict): New key added to state, documents, that contains retrieved documents\n    \"\"\"\n    print(\"---RETRIEVE---\")\n    question = state[\"question\"]\n\n    # Retrieval\n    documents = retriever.invoke(question)\n    return {\"documents\": documents, \"question\": question}\n\n\ndef llm_fallback(state):\n    \"\"\"\n    Generate answer using the LLM w/o vectorstore\n\n    Args:\n        state (dict): The current graph state\n\n    Returns:\n        state (dict): New key added to state, generation, that contains LLM generation\n    \"\"\"\n    print(\"---LLM Fallback---\")\n    question = state[\"question\"]\n    generation = llm_chain.invoke({\"question\": question})\n    return {\"question\": question, \"generation\": generation}\n\n\ndef generate(state):\n    \"\"\"\n    Generate answer using the vectorstore\n\n    Args:\n        state (dict): The current graph state\n\n    Returns:\n        state (dict): New key added to state, generation, that contains LLM generation\n    \"\"\"\n    print(\"---GENERATE---\")\n    question = state[\"question\"]\n    documents = state[\"documents\"]\n    if not isinstance(documents, list):\n        documents = [documents]\n\n    # RAG generation\n    generation = rag_chain.invoke({\"documents\": documents, \"question\": question})\n    return {\"documents\": documents, \"question\": question, \"generation\": generation}\n\n\ndef grade_documents(state):\n    \"\"\"\n    Determines whether the retrieved documents are relevant to the question.\n\n    Args:\n        state (dict): The current graph state\n\n    Returns:\n        state (dict): Updates documents key with only filtered relevant documents\n    \"\"\"\n\n    print(\"---CHECK DOCUMENT RELEVANCE TO QUESTION---\")\n    question = state[\"question\"]\n    documents = state[\"documents\"]\n\n    # Score each doc\n    filtered_docs = []\n    for d in documents:\n        score = retrieval_grader.invoke(\n            {\"question\": question, \"document\": d.page_content}\n        )\n        grade = score.binary_score\n        if grade == \"yes\":\n            print(\"---GRADE: DOCUMENT RELEVANT---\")\n            filtered_docs.append(d)\n        else:\n            print(\"---GRADE: DOCUMENT NOT RELEVANT---\")\n            continue\n    return {\"documents\": filtered_docs, \"question\": question}\n\n\ndef web_search(state):\n    \"\"\"\n    Web search based on the re-phrased question.\n\n    Args:\n        state (dict): The current graph state\n\n    Returns:\n        state (dict): Updates documents key with appended web results\n    \"\"\"\n\n    print(\"---WEB SEARCH---\")\n    question = state[\"question\"]\n\n    # Web search\n    docs = web_search_tool.invoke({\"query\": question})\n    web_results = \"\\n\".join([d[\"content\"] for d in docs])\n    web_results = Document(page_content=web_results)\n\n    return {\"documents\": web_results, \"question\": question}\n\n\n### Edges ###\n\n\ndef route_question(state):\n    \"\"\"\n    Route question to web search or RAG.\n\n    Args:\n        state (dict): The current graph state\n\n    Returns:\n        str: Next node to call\n    \"\"\"\n\n    print(\"---ROUTE QUESTION---\")\n    question = state[\"question\"]\n    source = question_router.invoke({\"question\": question})\n\n    # Fallback to LLM or raise error if no decision\n    if \"tool_calls\" not in source.additional_kwargs:\n        print(\"---ROUTE QUESTION TO LLM---\")\n        return \"llm_fallback\"\n    if len(source.additional_kwargs[\"tool_calls\"]) == 0:\n        raise \"Router could not decide source\"\n\n    # Choose datasource\n    datasource = source.additional_kwargs[\"tool_calls\"][0][\"function\"][\"name\"]\n    if datasource == \"web_search\":\n        print(\"---ROUTE QUESTION TO WEB SEARCH---\")\n        return \"web_search\"\n    elif datasource == \"vectorstore\":\n        print(\"---ROUTE QUESTION TO RAG---\")\n        return \"vectorstore\"\n    else:\n        print(\"---ROUTE QUESTION TO LLM---\")\n        return \"vectorstore\"\n\n\ndef decide_to_generate(state):\n    \"\"\"\n    Determines whether to generate an answer, or re-generate a question.\n\n    Args:\n        state (dict): The current graph state\n\n    Returns:\n        str: Binary decision for next node to call\n    \"\"\"\n\n    print(\"---ASSESS GRADED DOCUMENTS---\")\n    state[\"question\"]\n    filtered_documents = state[\"documents\"]\n\n    if not filtered_documents:\n        # All documents have been filtered check_relevance\n        # We will re-generate a new query\n        print(\"---DECISION: ALL DOCUMENTS ARE NOT RELEVANT TO QUESTION, WEB SEARCH---\")\n        return \"web_search\"\n    else:\n        # We have relevant documents, so generate answer\n        print(\"---DECISION: GENERATE---\")\n        return \"generate\"\n\n\ndef grade_generation_v_documents_and_question(state):\n    \"\"\"\n    Determines whether the generation is grounded in the document and answers question.\n\n    Args:\n        state (dict): The current graph state\n\n    Returns:\n        str: Decision for next node to call\n    \"\"\"\n\n    print(\"---CHECK HALLUCINATIONS---\")\n    question = state[\"question\"]\n    documents = state[\"documents\"]\n    generation = state[\"generation\"]\n\n    score = hallucination_grader.invoke(\n        {\"documents\": documents, \"generation\": generation}\n    )\n    grade = score.binary_score\n\n    # Check hallucination\n    if grade == \"yes\":\n        print(\"---DECISION: GENERATION IS GROUNDED IN DOCUMENTS---\")\n        # Check question-answering\n        print(\"---GRADE GENERATION vs QUESTION---\")\n        score = answer_grader.invoke({\"question\": question, \"generation\": generation})\n        grade = score.binary_score\n        if grade == \"yes\":\n            print(\"---DECISION: GENERATION ADDRESSES QUESTION---\")\n            return \"useful\"\n        else:\n            print(\"---DECISION: GENERATION DOES NOT ADDRESS QUESTION---\")\n            return \"not useful\"\n    else:\n        pprint(\"---DECISION: GENERATION IS NOT GROUNDED IN DOCUMENTS, RE-TRY---\")\n        return \"not supported\""
   ]
  },
  {
   "cell_type": "markdown",
   "id": "3ab01f36-5628-49ab-bfd3-84bb6f1a1b0f",
   "metadata": {
    "id": "3ab01f36-5628-49ab-bfd3-84bb6f1a1b0f"
   },
   "source": [
    "## Build Graph"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 12,
   "id": "67854e07-9293-4c3c-bf9a-bc9a605570ee",
   "metadata": {
    "id": "67854e07-9293-4c3c-bf9a-bc9a605570ee"
   },
   "outputs": [],
   "source": [
    "import pprint\n",
    "\n",
    "from langgraph.graph import END, StateGraph, START\n",
    "\n",
    "workflow = StateGraph(GraphState)\n",
    "\n",
    "# Define the nodes\n",
    "workflow.add_node(\"web_search\", web_search)  # web search\n",
    "workflow.add_node(\"retrieve\", retrieve)  # retrieve\n",
    "workflow.add_node(\"grade_documents\", grade_documents)  # grade documents\n",
    "workflow.add_node(\"generate\", generate)  # rag\n",
    "workflow.add_node(\"llm_fallback\", llm_fallback)  # llm\n",
    "\n",
    "# Build graph\n",
    "workflow.add_conditional_edges(\n",
    "    START,\n",
    "    route_question,\n",
    "    {\n",
    "        \"web_search\": \"web_search\",\n",
    "        \"vectorstore\": \"retrieve\",\n",
    "        \"llm_fallback\": \"llm_fallback\",\n",
    "    },\n",
    ")\n",
    "workflow.add_edge(\"web_search\", \"generate\")\n",
    "workflow.add_edge(\"retrieve\", \"grade_documents\")\n",
    "workflow.add_conditional_edges(\n",
    "    \"grade_documents\",\n",
    "    decide_to_generate,\n",
    "    {\n",
    "        \"web_search\": \"web_search\",\n",
    "        \"generate\": \"generate\",\n",
    "    },\n",
    ")\n",
    "workflow.add_conditional_edges(\n",
    "    \"generate\",\n",
    "    grade_generation_v_documents_and_question,\n",
    "    {\n",
    "        \"not supported\": \"generate\",  # Hallucinations: re-generate\n",
    "        \"not useful\": \"web_search\",  # Fails to answer question: fall-back to web-search\n",
    "        \"useful\": END,\n",
    "    },\n",
    ")\n",
    "workflow.add_edge(\"llm_fallback\", END)\n",
    "\n",
    "# Compile\n",
    "app = workflow.compile()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 13,
   "id": "29acc541-d726-4b75-84d1-a215845fe88a",
   "metadata": {
    "colab": {
     "base_uri": "https://localhost:8080/"
    },
    "id": "29acc541-d726-4b75-84d1-a215845fe88a",
    "outputId": "47caec8e-54e3-4f89-dfbb-94fc034666f7"
   },
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "---ROUTE QUESTION---\n",
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
      "'The Bears are expected to draft Caleb Williams with their first pick.'\n"
     ]
    }
   ],
   "source": [
    "# Run\ninputs = {\n    \"question\": \"What player are the Bears expected to draft first in the 2024 NFL draft?\"\n}\nfor output in app.stream(inputs):\n    for key, value in output.items():\n        # Node\n        pprint.pprint(f\"Node '{key}':\")\n        # Optional: print full state at each node\n    pprint.pprint(\"\\n---\\n\")\n\n# Final generation\npprint.pprint(value[\"generation\"])"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "11fddd00-58bf-4910-bf36-be9e5bfba778",
   "metadata": {
    "id": "11fddd00-58bf-4910-bf36-be9e5bfba778"
   },
   "source": [
    "Trace:\n",
    "\n",
    "https://smith.langchain.com/public/623da7bb-84a7-4e53-a63e-7ccd77fb9be5/r"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 14,
   "id": "69a985dd-03c6-45af-a67b-b15746a2cb5f",
   "metadata": {
    "colab": {
     "base_uri": "https://localhost:8080/"
    },
    "id": "69a985dd-03c6-45af-a67b-b15746a2cb5f",
    "outputId": "e5f799cc-6f36-494f-c8b2-192de1edb7fc"
   },
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "---ROUTE QUESTION---\n",
      "---ROUTE QUESTION TO RAG---\n",
      "---RETRIEVE---\n",
      "\"Node 'retrieve':\"\n",
      "'\\n---\\n'\n",
      "---CHECK DOCUMENT RELEVANCE TO QUESTION---\n",
      "---GRADE: DOCUMENT RELEVANT---\n",
      "---GRADE: DOCUMENT RELEVANT---\n",
      "---GRADE: DOCUMENT RELEVANT---\n",
      "---GRADE: DOCUMENT RELEVANT---\n",
      "---ASSESS GRADED DOCUMENTS---\n",
      "---DECISION: GENERATE---\n",
      "\"Node 'grade_documents':\"\n",
      "'\\n---\\n'\n",
      "---GENERATE---\n",
      "---CHECK HALLUCINATIONS---\n",
      "---DECISION: GENERATION IS GROUNDED IN DOCUMENTS---\n",
      "---GRADE GENERATION vs QUESTION---\n",
      "---DECISION: GENERATION ADDRESSES QUESTION---\n",
      "\"Node 'generate':\"\n",
      "'\\n---\\n'\n",
      "'Sensory, short-term, and long-term memory.'\n"
     ]
    }
   ],
   "source": [
    "# Run\ninputs = {\"question\": \"What are the types of agent memory?\"}\nfor output in app.stream(inputs):\n    for key, value in output.items():\n        # Node\n        pprint.pprint(f\"Node '{key}':\")\n        # Optional: print full state at each node\n        # pprint.pprint(value[\"keys\"], indent=2, width=80, depth=None)\n    pprint.pprint(\"\\n---\\n\")\n\n# Final generation\npprint.pprint(value[\"generation\"])"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "ebf41097-fc4c-4072-95b3-e7e07731ada1",
   "metadata": {
    "id": "ebf41097-fc4c-4072-95b3-e7e07731ada1"
   },
   "source": [
    "Trace:\n",
    "\n",
    "https://smith.langchain.com/public/57f3973b-6879-4fbe-ae31-9ae524c3a697/r"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 15,
   "id": "qPwP_2PNiOjQ",
   "metadata": {
    "id": "qPwP_2PNiOjQ"
   },
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "---ROUTE QUESTION---\n",
      "---ROUTE QUESTION TO LLM---\n",
      "---LLM Fallback---\n",
      "\"Node 'llm_fallback':\"\n",
      "'\\n---\\n'\n",
      "(\"I don't have feelings as an AI assistant, but I'm here to help you with your \"\n",
      " 'queries. How can I assist you today?')\n"
     ]
    }
   ],
   "source": [
    "# Run\ninputs = {\"question\": \"Hello, how are you today?\"}\nfor output in app.stream(inputs):\n    for key, value in output.items():\n        # Node\n        pprint.pprint(f\"Node '{key}':\")\n        # Optional: print full state at each node\n        # pprint.pprint(value[\"keys\"], indent=2, width=80, depth=None)\n    pprint.pprint(\"\\n---\\n\")\n\n# Final generation\npprint.pprint(value[\"generation\"])"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "4107c8a4-6171-4c1b-840a-77a3d09f84fc",
   "metadata": {},
   "source": [
    "Trace: \n",
    "\n",
    "https://smith.langchain.com/public/1f628ee4-8d2d-451e-aeb1-5d5e0ede2b4f/r"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "ce3cda0a-c4bd-41ea-830b-d992f27fde15",
   "metadata": {},
   "outputs": [],
   "source": [
    ""
   ]
  }
 ],
 "metadata": {
  "colab": {
   "provenance": []
  },
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
https://github.com/langchain-ai/langgraph/blob/main/examples/rag/langgraph_adaptive_rag_local.ipynb
```ipynb
{
 "cells": [
  {
   "attachments": {
    "3755396d-c4a8-45bd-87d4-00cb56339fe5.png": {
