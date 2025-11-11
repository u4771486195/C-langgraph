    }
   },
   "cell_type": "markdown",
   "id": "5afcaed0-3d55-4e1f-95d3-c32c751c29d8",
   "metadata": {},
   "source": [
    "# Adaptive RAG\n",
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
    "![Screenshot 2024-03-26 at 1.36.03 PM.png](attachment:36fa621a-9d3d-4860-a17c-5d20e6987481.png)"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "a85501ca-eb89-4795-aeab-cdab050ead6b",
   "metadata": {},
   "source": [
    "## Setup\n",
    "\n",
    "First, let's install our required packages and set our API keys"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 1,
   "id": "53d1a740-9fea-4a6e-8f95-fb9dbf1c80a1",
   "metadata": {},
   "outputs": [],
   "source": [
    "%%capture --no-stderr\n",
    "%pip install -U langchain_community tiktoken langchain-openai langchain-cohere langchainhub chromadb langchain langgraph  tavily-python"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "222f204d-956f-4128-b597-2c698120edda",
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
    "# _set_env(\"COHERE_API_KEY\")\n",
    "_set_env(\"TAVILY_API_KEY\")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "47e04b18",
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
   "id": "9ac1c2cd-81fb-40eb-8ba1-e9197800cba6",
   "metadata": {},
   "source": [
    "## Create Index\n",
    "\n",
    "Set up a vector database using OpenAI Embeddings and the Chroma vector database.  \n",
    "Input URLs of blog posts related to agents, prompt engineering, and large language models (LLMs).  \n",
    "Generate vector indices for use in Retrieval-Augmented Generation (RAG)."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "b224e5ba-50ca-495a-a7fa-0f75a080e03c",
   "metadata": {},
   "outputs": [],
   "source": [
    "### Build Index\n",
    "\n",
    "from langchain.text_splitter import RecursiveCharacterTextSplitter\n",
    "from langchain_community.document_loaders import WebBaseLoader\n",
    "from langchain_community.vectorstores import Chroma\n",
    "from langchain_openai import OpenAIEmbeddings\n",
    "\n",
    "### from langchain_cohere import CohereEmbeddings\n",
    "\n",
    "# Set embeddings\n",
    "embd = OpenAIEmbeddings()\n",
    "\n",
    "# Docs to index\n",
    "urls = [\n",
    "    \"https://lilianweng.github.io/posts/2023-06-23-agent/\",\n",
    "    \"https://lilianweng.github.io/posts/2023-03-15-prompt-engineering/\",\n",
    "    \"https://lilianweng.github.io/posts/2023-10-25-adv-attack-llm/\",\n",
    "]\n",
    "\n",
    "# Load\n",
    "docs = [WebBaseLoader(url).load() for url in urls]\n",
    "docs_list = [item for sublist in docs for item in sublist]\n",
    "\n",
    "# Split\n",
    "text_splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(\n",
    "    chunk_size=500, chunk_overlap=0\n",
    ")\n",
    "doc_splits = text_splitter.split_documents(docs_list)\n",
    "\n",
    "# Add to vectorstore\n",
    "vectorstore = Chroma.from_documents(\n",
    "    documents=doc_splits,\n",
    "    collection_name=\"rag-chroma\",\n",
    "    embedding=embd,\n",
    ")\n",
    "retriever = vectorstore.as_retriever()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "0f52b427-750c-40f8-8893-e9caab3afd8d",
   "metadata": {},
   "source": [
    "## LLMs"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "d28baefd-a961-49b0-8394-c5478dadda1c",
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
   "cell_type": "markdown",
   "id": "6cdd5ac0-fa18-4ee9-8051-062a0c56268f",
   "metadata": {},
   "source": [
    "### Router for Query Analysis\n",
    "\n",
    "Let’s start with Routing. First, assign the query analysis to the LLM.\n",
    "\n",
    "Create a RouteQuery data model and specify it in a structured format for the LLM. The decision for routing should be embedded in the prompt. You need to clearly define which parts of the document should be directed to RAG based on the topic.\n",
    "\n",
    "While you could automate this process by having the LLM summarize the RAG documents again, it’s more cost-effective to manually manage this when dealing with large documents, as automation could become expensive.\n",
    "\n"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 4,
   "id": "4dec9d98-f3dc-4b7f-abc0-9d01c754f2be",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "datasource='web_search'\n",
      "datasource='vectorstore'\n"
     ]
    }
   ],
   "source": [
    "### Router\n",
    "\n",
    "from typing import Literal\n",
    "\n",
    "from langchain_core.prompts import ChatPromptTemplate\n",
    "from langchain_openai import ChatOpenAI\n",
    "\n",
    "from pydantic import BaseModel, Field\n",
    "\n",
    "\n",
    "# Data model\n",
    "class RouteQuery(BaseModel):\n",
    "    \"\"\"Route a user query to the most relevant datasource.\"\"\"\n",
    "\n",
    "    datasource: Literal[\"vectorstore\", \"web_search\"] = Field(\n",
    "        ...,\n",
    "        description=\"Given a user question choose to route it to web search or a vectorstore.\",\n",
    "    )\n",
    "\n",
    "\n",
    "# LLM with function call\n",
    "llm = ChatOpenAI(model=\"gpt-4o-mini\", temperature=0)\n",
    "structured_llm_router = llm.with_structured_output(RouteQuery)\n",
    "\n",
    "# Prompt\n",
    "system = \"\"\"You are an expert at routing a user question to a vectorstore or web search.\n",
    "The vectorstore contains documents related to agents, prompt engineering, and adversarial attacks.\n",
    "Use the vectorstore for questions on these topics. Otherwise, use web-search.\"\"\"\n",
    "route_prompt = ChatPromptTemplate.from_messages(\n",
    "    [\n",
    "        (\"system\", system),\n",
    "        (\"human\", \"{question}\"),\n",
    "    ]\n",
    ")\n",
    "\n",
    "question_router = route_prompt | structured_llm_router\n",
    "print(\n",
    "    question_router.invoke(\n",
    "        {\"question\": \"Who will the Bears draft first in the NFL draft?\"}\n",
    "    )\n",
    ")\n",
    "print(question_router.invoke({\"question\": \"What are the types of agent memory?\"}))"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "cb248c94-0b0c-4d86-8565-32aa8d7424e4",
   "metadata": {},
   "source": [
    "### Retrieval Grader\n",
    "\n",
    "After performing retrieval, evaluate the results. Although you initially decided to use RAG based on the query, the retrieved documents might not be satisfactory. Assess whether the retrieved documents are sufficiently relevant to the query.\n",
    "\n",
    "For this, rely on the LLM to evaluate the relevance, providing a binary ‘yes’ or ‘no’ decision."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 5,
   "id": "856801cb-f42a-44e7-956f-47845e3664ca",
   "metadata": {},
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
    "### Retrieval Grader\n",
    "\n",
    "\n",
    "# Data model\n",
    "class GradeDocuments(BaseModel):\n",
    "    \"\"\"Binary score for relevance check on retrieved documents.\"\"\"\n",
    "\n",
    "    binary_score: str = Field(\n",
    "        description=\"Documents are relevant to the question, 'yes' or 'no'\"\n",
    "    )\n",
    "\n",
    "\n",
    "# LLM with function call\n",
    "llm = ChatOpenAI(model=\"gpt-4o-mini\", temperature=0)\n",
    "structured_llm_grader = llm.with_structured_output(GradeDocuments)\n",
    "\n",
    "# Prompt\n",
    "system = \"\"\"You are a grader assessing relevance of a retrieved document to a user question. \\n \n",
    "    If the document contains keyword(s) or semantic meaning related to the user question, grade it as relevant. \\n\n",
    "    It does not need to be a stringent test. The goal is to filter out erroneous retrievals. \\n\n",
    "    Give a binary score 'yes' or 'no' score to indicate whether the document is relevant to the question.\"\"\"\n",
    "grade_prompt = ChatPromptTemplate.from_messages(\n",
    "    [\n",
    "        (\"system\", system),\n",
    "        (\"human\", \"Retrieved document: \\n\\n {document} \\n\\n User question: {question}\"),\n",
    "    ]\n",
    ")\n",
    "\n",
    "retrieval_grader = grade_prompt | structured_llm_grader\n",
    "question = \"agent memory\"\n",
    "docs = retriever.invoke(question)\n",
    "doc_txt = docs[1].page_content\n",
    "print(retrieval_grader.invoke({\"question\": question, \"document\": doc_txt}))"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 6,
   "id": "2272333e-50b2-42ab-b472-e1055a3b94a8",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Agent memory in LLM-powered autonomous systems consists of short-term and long-term memory. Short-term memory utilizes in-context learning for immediate tasks, while long-term memory allows agents to retain and recall information over extended periods, often using external storage for efficient retrieval. This memory structure supports the agent's ability to reflect on past actions and improve future performance.\n"
     ]
    }
   ],
   "source": [
    "### Generate\n",
    "\n",
    "from langchain import hub\n",
    "from langchain_core.output_parsers import StrOutputParser\n",
    "\n",
    "# Prompt\n",
    "prompt = hub.pull(\"rlm/rag-prompt\")\n",
    "\n",
    "# LLM\n",
    "llm = ChatOpenAI(model_name=\"gpt-4o-mini\", temperature=0)\n",
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
    "docs_txt = format_docs(docs)\n",
    "generation = rag_chain.invoke({\"context\": docs_txt, \"question\": question})\n",
    "print(generation)"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "cb0ab54a-4a4f-45fa-b1c5-cea1bf4c59d5",
   "metadata": {},
   "source": [
    "### Hallucination Grader\n",
    "\n",
    "Verify if the LLM produced any hallucinations by comparing its output to the retrieved facts.  \n",
    "Provide the LLM’s evaluation in a binary ‘yes’ or ‘no’ format.\n"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 7,
   "id": "f0c08d14-77a0-4eed-b882-2d636abb22a3",
   "metadata": {},
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
    "### Hallucination Grader\n",
    "\n",
    "\n",
    "# Data model\n",
    "class GradeHallucinations(BaseModel):\n",
    "    \"\"\"Binary score for hallucination present in generation answer.\"\"\"\n",
    "\n",
    "    binary_score: str = Field(\n",
    "        description=\"Answer is grounded in the facts, 'yes' or 'no'\"\n",
    "    )\n",
    "\n",
    "\n",
    "# LLM with function call\n",
    "llm = ChatOpenAI(model=\"gpt-4o-mini\", temperature=0)\n",
    "structured_llm_grader = llm.with_structured_output(GradeHallucinations)\n",
    "\n",
    "# Prompt\n",
    "system = \"\"\"You are a grader assessing whether an LLM generation is grounded in / supported by a set of retrieved facts. \\n \n",
    "     Give a binary score 'yes' or 'no'. 'Yes' means that the answer is grounded in / supported by the set of facts.\"\"\"\n",
    "hallucination_prompt = ChatPromptTemplate.from_messages(\n",
    "    [\n",
    "        (\"system\", system),\n",
    "        (\"human\", \"Set of facts: \\n\\n {documents} \\n\\n LLM generation: {generation}\"),\n",
    "    ]\n",
    ")\n",
    "\n",
    "hallucination_grader = hallucination_prompt | structured_llm_grader\n",
    "hallucination_grader.invoke({\"documents\": docs, \"generation\
```
> [truncated]

---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/rag/langgraph_adaptive_rag_local.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "code",
   "execution_count": 11,
   "id": "2631596b-8a66-4d23-a00d-d2c745c611d4",
   "metadata": {},
   "outputs": [],
   "source": [
    "%%capture --no-stderr\n",
    "%pip install --quiet -U langchain langchain_community tiktoken langchain-nomic \"nomic[local]\" langchain-ollama scikit-learn langgraph tavily-python bs4"
   ]
  },
  {
   "attachments": {
    "6cd777a6-a0b3-4feb-bd07-8e9e8a4b32a0.png": {
