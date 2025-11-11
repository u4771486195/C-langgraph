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
    "### Answer Grader\n",
    "\n",
    "\n",
    "# Data model\n",
    "class GradeAnswer(BaseModel):\n",
    "    \"\"\"Binary score to assess answer addresses question.\"\"\"\n",
    "\n",
    "    binary_score: str = Field(\n",
    "        description=\"Answer addresses the question, 'yes' or 'no'\"\n",
    "    )\n",
    "\n",
    "\n",
    "# LLM with function call\n",
    "llm = ChatOpenAI(model=\"gpt-4o-mini\", temperature=0)\n",
    "structured_llm_grader = llm.with_structured_output(GradeAnswer)\n",
    "\n",
    "# Prompt\n",
    "answer_prompt = hub.pull(\"efriis/self-rag-answer-grader\")\n",
    "\n",
    "answer_grader = answer_prompt | structured_llm_grader\n",
    "print(question)\n",
    "print(generation)\n",
    "answer_grader.invoke({\"question\": question, \"generation\": generation})"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 9,
   "id": "c6f4c70e-1660-4149-82c0-837f19fc9fb5",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "movies starring jason momoa\n"
     ]
    },
    {
     "data": {
      "text/plain": [
       "'Which movies feature Jason Momoa in a leading role?'"
      ]
     },
     "execution_count": 9,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "### Question Re-writer\n\n# LLM\nllm = ChatOpenAI(model=\"gpt-3.5-turbo-0125\", temperature=0)\n\n# Prompt\nre_write_prompt = hub.pull(\"efriis/self-rag-question-rewriter\")\n\nquestion_rewriter = re_write_prompt | llm | StrOutputParser()\nprint(question)\nquestion_rewriter.invoke({\"question\": question})"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "276001c5-c079-4e5b-9f42-81a06704d200",
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
   "execution_count": 10,
   "id": "f1617e9e-66a8-4c1a-a1fe-cc936284c085",
   "metadata": {},
   "outputs": [],
   "source": [
    "from typing import List\n\nfrom typing_extensions import TypedDict\n\n\nclass GraphState(TypedDict):\n    \"\"\"\n    Represents the state of our graph.\n\n    Attributes:\n        question: question\n        generation: LLM generation\n        documents: list of documents\n    \"\"\"\n\n    question: str\n    generation: str\n    documents: List[str]"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 11,
   "id": "add509d8-6682-4127-8d95-13dd37d79702",
   "metadata": {},
   "outputs": [],
   "source": [
    "### Nodes\n\n\ndef retrieve(state):\n    \"\"\"\n    Retrieve documents\n\n    Args:\n        state (dict): The current graph state\n\n    Returns:\n        state (dict): New key added to state, documents, that contains retrieved documents\n    \"\"\"\n    print(\"---RETRIEVE---\")\n    question = state[\"question\"]\n\n    # Retrieval\n    documents = retriever.invoke(question)\n    return {\"documents\": documents, \"question\": question}\n\n\ndef generate(state):\n    \"\"\"\n    Generate answer\n\n    Args:\n        state (dict): The current graph state\n\n    Returns:\n        state (dict): New key added to state, generation, that contains LLM generation\n    \"\"\"\n    print(\"---GENERATE---\")\n    question = state[\"question\"]\n    documents = state[\"documents\"]\n\n    # RAG generation\n    generation = rag_chain.invoke({\"context\": documents, \"question\": question})\n    return {\"documents\": documents, \"question\": question, \"generation\": generation}\n\n\ndef grade_documents(state):\n    \"\"\"\n    Determines whether the retrieved documents are relevant to the question.\n\n    Args:\n        state (dict): The current graph state\n\n    Returns:\n        state (dict): Updates documents key with only filtered relevant documents\n    \"\"\"\n\n    print(\"---CHECK DOCUMENT RELEVANCE TO QUESTION---\")\n    question = state[\"question\"]\n    documents = state[\"documents\"]\n\n    # Score each doc\n    filtered_docs = []\n    for d in documents:\n        score = retrieval_grader.invoke(\n            {\"question\": question, \"document\": d.page_content}\n        )\n        grade = score.binary_score\n        if grade == \"yes\":\n            print(\"---GRADE: DOCUMENT RELEVANT---\")\n            filtered_docs.append(d)\n        else:\n            print(\"---GRADE: DOCUMENT NOT RELEVANT---\")\n            continue\n    return {\"documents\": filtered_docs, \"question\": question}\n\n\ndef transform_query(state):\n    \"\"\"\n    Transform the query to produce a better question.\n\n    Args:\n        state (dict): The current graph state\n\n    Returns:\n        state (dict): Updates question key with a re-phrased question\n    \"\"\"\n\n    print(\"---TRANSFORM QUERY---\")\n    question = state[\"question\"]\n    documents = state[\"documents\"]\n\n    # Re-write question\n    better_question = question_rewriter.invoke({\"question\": question})\n    return {\"documents\": documents, \"question\": better_question}"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 12,
   "id": "09fc91b4",
   "metadata": {},
   "outputs": [],
   "source": [
    "### Edges\n\n\ndef decide_to_generate(state):\n    \"\"\"\n    Determines whether to generate an answer, or re-generate a question.\n\n    Args:\n        state (dict): The current graph state\n\n    Returns:\n        str: Binary decision for next node to call\n    \"\"\"\n\n    print(\"---ASSESS GRADED DOCUMENTS---\")\n    state[\"question\"]\n    filtered_documents = state[\"documents\"]\n\n    if not filtered_documents:\n        # All documents have been filtered check_relevance\n        # We will re-generate a new query\n        print(\n            \"---DECISION: ALL DOCUMENTS ARE NOT RELEVANT TO QUESTION, TRANSFORM QUERY---\"\n        )\n        return \"transform_query\"\n    else:\n        # We have relevant documents, so generate answer\n        print(\"---DECISION: GENERATE---\")\n        return \"generate\"\n\n\ndef grade_generation_v_documents_and_question(state):\n    \"\"\"\n    Determines whether the generation is grounded in the document and answers question.\n\n    Args:\n        state (dict): The current graph state\n\n    Returns:\n        str: Decision for next node to call\n    \"\"\"\n\n    print(\"---CHECK HALLUCINATIONS---\")\n    question = state[\"question\"]\n    documents = state[\"documents\"]\n    generation = state[\"generation\"]\n\n    score = hallucination_grader.invoke(\n        {\"documents\": documents, \"generation\": generation}\n    )\n    grade = score.binary_score\n\n    # Check hallucination\n    if grade == \"yes\":\n        print(\"---DECISION: GENERATION IS GROUNDED IN DOCUMENTS---\")\n        # Check question-answering\n        print(\"---GRADE GENERATION vs QUESTION---\")\n        score = answer_grader.invoke({\"question\": question, \"generation\": generation})\n        grade = score.binary_score\n        if grade == \"yes\":\n            print(\"---DECISION: GENERATION ADDRESSES QUESTION---\")\n            return \"useful\"\n        else:\n            print(\"---DECISION: GENERATION DOES NOT ADDRESS QUESTION---\")\n            return \"not useful\"\n    else:\n        pprint(\"---DECISION: GENERATION IS NOT GROUNDED IN DOCUMENTS, RE-TRY---\")\n        return \"not supported\""
   ]
  },
  {
   "cell_type": "markdown",
   "id": "61cd5797-1782-4d78-a277-8196d13f3e1b",
   "metadata": {},
   "source": [
    "## Build Graph\n",
    "\n",
    "The just follows the flow we outlined in the figure above."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 13,
   "id": "0e09ca9f-e36d-4ef4-a0d5-79fdbada9fe0",
   "metadata": {},
   "outputs": [],
   "source": ["from langgraph.graph import END, StateGraph, START\n\nworkflow = StateGraph(GraphState)\n\n# Define the nodes\nworkflow.add_node(\"retrieve\", retrieve)  # retrieve\nworkflow.add_node(\"grade_documents\", grade_documents)  # grade documents\nworkflow.add_node(\"generate\", generate)  # generate\nworkflow.add_node(\"transform_query\", transform_query)  # transform_query\n\n# Build graph\nworkflow.add_edge(START, \"retrieve\")\nworkflow.add_edge(\"retrieve\", \"grade_documents\")\nworkflow.add_conditional_edges(\n    \"grade_documents\",\n    decide_to_generate,\n    {\n        \"transform_query\": \"transform_query\",\n        \"generate\": \"generate\",\n    },\n)\nworkflow.add_edge(\"transform_query\", \"retrieve\")\nworkflow.add_conditional_edges(\n    \"generate\",\n    grade_generation_v_documents_and_question,\n    {\n        \"not supported\": \"generate\",\n        \"useful\": END,\n        \"not useful\": \"transform_query\",\n    },\n)\n\n# Compile\napp = workflow.compile()"]
  },
  {
   "cell_type": "code",
   "execution_count": 14,
   "id": "fb69dbb9-91ee-4868-8c3c-93af3cd885be",
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "---RETRIEVE---\n",
      "\"Node 'retrieve':\"\n",
      "'\\n---\\n'\n",
      "---CHECK DOCUMENT RELEVANCE TO QUESTION---\n",
      "---GRADE: DOCUMENT RELEVANT---\n",
      "---GRADE: DOCUMENT RELEVANT---\n",
      "---GRADE: DOCUMENT NOT RELEVANT---\n",
      "---GRADE: DOCUMENT NOT RELEVANT---\n",
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
      "'Daniel Craig stars as 007 in \"Skyfall\" (2012) and \"Spectre\" (2015).'\n"
     ]
    }
   ],
   "source": [
    "from pprint import pprint\n\n# Run\ninputs = {\"question\": \"Movies that star Daniel Craig\"}\nfor output in app.stream(inputs):\n    for key, value in output.items():\n        # Node\n        pprint(f\"Node '{key}':\")\n    pprint(\"\\n---\\n\")\n\n# Final generation\npprint(value[\"generation\"])"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "4138bc51-8c84-4b8a-8d24-f7f470721f6f",
   "metadata": {},
   "outputs": [],
   "source": [
    "inputs = {\"question\": \"Which movies are about aliens?\"}\nfor output in app.stream(inputs):\n    for key, value in output.items():\n        # Node\n        pprint(f\"Node '{key}':\")\n    pprint(\"\\n---\\n\")\n\n# Final generation\npprint(value[\"generation\"])"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "42369ab8-322d-434a-b5dd-2266e4cb2903",
   "metadata": {},
   "outputs": [],
   "source": [
    ""
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
https://github.com/langchain-ai/langgraph/blob/main/examples/tutorials/sql-agent.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "83c2223f",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/sql-agent.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/tutorials/tnt-llm/tnt-llm.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "11140167",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/tnt-llm/tnt-llm.ipynb"
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
   "version": "3.11.2"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 5
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/memory/add-summary-conversation-history.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "298784f6",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/memory/add-summary-conversation-history.ipynb"
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
   "version": "3.11.1"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 5
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/memory/delete-messages.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "3f4370fd",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/memory/delete-messages.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/memory/manage-conversation-history.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "6ec7cb13",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/memory/manage-conversation-history.ipynb"
   ]
  }
 ],
 "metadata": {
  "kernelspec": {
   "display_name": "Python 3",
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
https://github.com/langchain-ai/langgraph/blob/main/examples/cloud_examples/langgraph_to_langgraph_cloud.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "2b789e16",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/langgraph_to_langgraph_cloud.ipynb"
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
   "version": "3.11.1"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 5
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/reflexion/reflexion.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "caf07859",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/reflexion/reflexion.ipynb"
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
   "version": "3.12.2"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 5
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/.gitignore
```text
site/

.vercel

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/Makefile
```text
.PHONY: lint-docs format-docs build-docs serve-docs serve-clean-docs clean-docs codespell llms-text build-prebuilt tests

build-prebuilt:
	# Use to create an update to date prebuilt page.
	# Looks up download stats for each of the prebuilt packages and
	# generates the final prebuilt page.
	@if [ "$(DOWNLOAD_STATS)" = "true" ]; then \
		set -x; \
		uv run python -m _scripts.third_party_page.get_download_stats stats.yml; \
		set +x; \
	else \
		set -x; \
		uv run python -m _scripts.third_party_page.get_download_stats --fake stats.yml; \
		set +x; \
	fi
	uv run python -m _scripts.third_party_page.create_third_party_page stats.yml docs/agents/prebuilt.md

build-docs: build-prebuilt
	TARGET_LANGUAGE=python uv run python -m mkdocs build --clean -f mkdocs.yml --strict

build-docs-js: build-prebuilt
	TARGET_LANGUAGE=js uv run python -m mkdocs build --clean -f mkdocs.yml --strict

llms-text:
	uv run python -m _scripts.generate_llms_text docs/llms-full.txt

install-vercel-deps:
	curl -sL "https://astral.sh/uv/install.sh" | bash -s
	export PATH="${HOME}/.cargo/bin:${PATH}"
	uv venv --python 3.11
	uv sync --all-groups

tests:
	# Run unit tests
	uv run pytest tests/unit_tests


vercel-build-docs: install-vercel-deps
	make build-docs


serve-clean-docs: clean-docs
	uv run python -m mkdocs serve -c -f mkdocs.yml --strict -w ../libs/langgraph

serve-docs:
	uv run python -m mkdocs serve -f mkdocs.yml -w ../libs/langgraph  -w ../libs/checkpoint -w ../libs/sdk-py --dirty

clean-docs:
	find ./docs -name "*.ipynb" -type f -delete
	rm -rf site

## Run format against the project documentation.
format-docs:
	uv run ruff format docs
	uv run ruff check --fix docs

# Check the docs for linting violations
lint-docs:
	uv run ruff format --check docs
	uv run ruff check docs

codespell:
	./codespell_notebooks.sh .

start-services:
	docker compose -f test-compose.yml up -V --force-recreate --wait --remove-orphans

stop-services:
	docker compose -f test-compose.yml down

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/README.md
```md
# LangGraph Documentation

For more information on contributing to our documentation, see the [Contributing Guide](../CONTRIBUTING.md).

## Structure

The primary documentation is located in the `docs/` directory. This directory contains both the source files for the main documentation as well as the API reference doc build process.

### Main Documentation

Main documentation files are located in `docs/docs/` and are written in Markdown format. The site uses [**MkDocs**](https://www.mkdocs.org/) with the [Material theme](https://squidfunk.github.io/mkdocs-material/) and includes:

- **Concepts**: Core LangGraph concepts and explanations
- **Tutorials**: Step-by-step learning guides
- **How-tos**: Task-focused guides for specific use cases
- **Examples**: Real-world applications and use cases
- **Jupyter Notebooks**: Interactive tutorials that are automatically converted to markdown

### API Reference

API reference documentation is defined in `docs/docs/reference/`. Each `.md` file outlines the "template" that each page is built from. Reference content is automatically generated from docstrings in the codebase using the **mkdocstrings** plugin. Once generated, the content is plugged into the corresponding markdown file where it is referenced by using manual directives to specify which classes and/or functions are documented:

```markdown
::: langgraph.graph.state.StateGraph
    options:
      show_if_no_docstring: true
      show_root_heading: true
      show_root_full_path: false
      members:
        - add_node
        - add_edge
        - add_conditional_edges
        - add_sequence
        - compile
```

## Build Process

Docs are built following these steps:

1. **Content Processing:**
   - `_scripts/notebook_hooks.py` - Main processing pipeline that:
     - Converts how-tos/tutorial Jupyter notebooks to markdown using `notebook_convert.py`
     - Adds automatic API reference links to code blocks using `generate_api_reference_links.py`
     - Handles conditional rendering for Python/JS versions
     - Processes highlight comments and custom syntax

2. **API Reference Generation:**
   - **mkdocstrings** plugin extracts docstrings from Python source code
   - Manual `::: module.Class` directives in reference pages (`/docs/docs/*`) specify what to document
   - Cross-references are automatically generated between docs and API

3. **Site Generation:**
   - **MkDocs** processes all markdown files and generates static HTML
   - Custom hooks handle redirects and inject additional functionality

4. **Deployment:**
   - Site is deployed with Vercel
   - `make build-docs` generates production build (also usable for local testing)
   - Automatic redirects handle URL changes between versions

### Local Development

For local development, use the Makefile targets:

```bash
# Serve docs locally with hot reloading
make serve-docs

# Clean build for production testing
make build-docs

# Serve with clean build
make serve-clean-docs
```

The `serve-docs` command:

- Watches source files for changes
- Includes dirty builds for faster iteration
- Serves on [http://127.0.0.1:8000/langgraph/](http://127.0.0.1:8000/langgraph/)

## Standards

**Docstring Format:**
The API reference uses **Google-style docstrings** with Markdown markup. The `mkdocstrings` plugin processes these to generate documentation.

**Required format:**

```python
def example_function(param1: str, param2: int = 5) -> bool:
    """Brief description of the function.

    Longer description can go here. Use Markdown syntax for
    rich formatting like **bold** and *italic*.

    Args:
        param1: Description of the first parameter.
        param2: Description of the second parameter with default value.

    Returns:
        Description of the return value.

    Raises:
        ValueError: When param1 is empty.
        TypeError: When param2 is not an integer.

    !!! warning
        This function is experimental and may change.

    !!! version-added "Added in version 0.2.0"
    """
```

**Special Markers:**

- **MkDocs admonitions**: `!!! warning`, `!!! note`, `!!! version-added`
- **Code blocks**: Standard markdown ``` syntax
- **Cross-references**: Automatic linking via `generate_api_reference_links.py`

## Execute notebooks

If you would like to automatically execute all of the notebooks, to mimic the "Run notebooks" GitHub action, you can run:

```bash
python _scripts/prepare_notebooks_for_ci.py
./_scripts/execute_notebooks.sh
```

**Note**: if you want to run the notebooks without `%pip install` cells, you can run:

```bash
python _scripts/prepare_notebooks_for_ci.py --comment-install-cells
./_scripts/execute_notebooks.sh
```

`prepare_notebooks_for_ci.py` script will add VCR cassette context manager for each cell in the notebook, so that:

- when the notebook is run for the first time, cells with network requests will be recorded to a VCR cassette file
- when the notebook is run subsequently, the cells with network requests will be replayed from the cassettes

## Adding new notebooks

If you are adding a notebook with API requests, it's **recommended** to record network requests so that they can be subsequently replayed. If this is not done, the notebook runner will make API requests every time the notebook is run, which can be costly and slow.

To record network requests, please make sure to first run `prepare_notebooks_for_ci.py` script.

Then, run

```bash
jupyter execute <path_to_notebook>
```

Once the notebook is executed, you should see the new VCR cassettes recorded in `cassettes` directory and discard the updated notebook.

## Updating existing notebooks

If you are updating an existing notebook, please make sure to remove any existing cassettes for the notebook in `cassettes` directory (each cassette is prefixed with the notebook name), and then run the steps from the "Adding new notebooks" section above.

To delete cassettes for a notebook, you can run:

```bash
rm cassettes/<notebook_name>*
```

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/codespell_notebooks.sh
```sh
ERROR_FOUND=0
for file in $(find $1 -name "*.ipynb" | grep -v ".ipynb_checkpoints"); do
    # Adding regexp to ignore base64 strings
    OUTPUT=$(cat "$file" | jupytext --from ipynb --to py:percent | codespell --ignore-regex='[A-Za-z0-9+/=]{25,}' -)
    if [ -n "$OUTPUT" ]; then
        echo "Errors found in $file"
        echo "$OUTPUT"
        ERROR_FOUND=1
    fi
done

for file in $(find $1 -name "*.md"); do
    # Adding regexp to ignore base64 strings
    OUTPUT=$(cat "$file" | codespell --ignore-regex='[A-Za-z0-9+/=]{25,}' -)
    if [ -n "$OUTPUT" ]; then
        echo "Errors found in $file"
        echo "$OUTPUT"
        ERROR_FOUND=1
    fi
done

if [ "$ERROR_FOUND" -ne 0 ]; then
    exit 1
fi

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/mkdocs.yml
```yml
site_name: "LangGraph"
site_description: Build reliable, stateful AI systems, without giving up control
site_url: https://langchain-ai.github.io/langgraph/
repo_url: https://github.com/langchain-ai/langgraph
edit_uri: edit/main/docs/docs/
theme:
  name: material
  custom_dir: overrides
  logo_dark_mode: static/wordmark_light.svg
  logo_light_mode: static/wordmark_dark.svg
  favicon: static/favicon.png
  icon:
    repo: fontawesome/brands/git-alt
  features:
    - announce.dismiss
    - content.code.annotate
    - content.code.copy
    - content.code.select
    - content.tabs.link
    - content.action.edit
    - content.tooltips
    - navigation.indexes
    - navigation.footer
    - navigation.instant
    - navigation.sections
    - navigation.instant.prefetch
    - navigation.instant.progress
    - navigation.path
    - navigation.tabs
    - navigation.top
    - navigation.prune
    - navigation.tracking
    - search.highlight
    - search.share
    - search.suggest
  palette:
    - scheme: default
      primary: white
      accent: gray
      toggle:
        icon: material/brightness-7
        name: Switch to dark mode
    - scheme: slate
      primary: grey
      accent: white
      toggle:
        icon: material/brightness-4
        name: Switch to light mode
  font:
    text: "Public Sans"
    code: "Roboto Mono"
plugins:
  - search:
      separator: '[\s\u200b\-,:!=\[\]()"`/]+|\.(?!\d)|&[lg]t;'
  - exclude-search:
      exclude:
        - additional-resources/index.md
        - agents/prebuilt.md
        - cloud/concepts/cron_jobs.md
        - cloud/concepts/data_storage_and_privacy.md
        - cloud/concepts/webhooks.md
        - cloud/deployment/cloud.md
        - cloud/deployment/custom_docker.md
        - cloud/deployment/egress.md
        - cloud/deployment/graph_rebuild.md
        - cloud/deployment/self_hosted_control_plane.md
        - cloud/deployment/self_hosted_data_plane.md
        - cloud/deployment/semantic_search.md
        - cloud/deployment/setup_javascript.md
        - cloud/deployment/setup_pyproject.md
        - cloud/deployment/setup.md
        - cloud/deployment/standalone_container.md
        - cloud/how-tos/add-human-in-the-loop.md
        - cloud/how-tos/background_run.md
        - cloud/how-tos/clone_traces_studio.md
        - cloud/how-tos/configurable_headers.md
        - cloud/how-tos/configuration_cloud.md
        - cloud/how-tos/cron_jobs.md
        - cloud/how-tos/datasets_studio.md
        - cloud/how-tos/enqueue_concurrent.md
        - cloud/how-tos/generative_ui_react.md
        - cloud/how-tos/human_in_the_loop_time_travel.md
        - cloud/how-tos/interrupt_concurrent.md
        - cloud/how-tos/invoke_studio.md
        - cloud/how-tos/iterate_graph_studio.md
        - cloud/how-tos/reject_concurrent.md
        - cloud/how-tos/rollback_concurrent.md
        - cloud/how-tos/same-thread.md
        - cloud/how-tos/stateless_runs.md
        - cloud/how-tos/streaming.md
        - cloud/how-tos/studio/manage_assistants.md
        - cloud/how-tos/studio/quick_start.md
        - cloud/how-tos/studio/run_evals.md
        - cloud/how-tos/threads_studio.md
        - cloud/how-tos/use_stream_react.md
        - cloud/how-tos/use_threads.md
        - cloud/how-tos/webhooks.md
        - cloud/quick_start.md
        - cloud/reference/api/api_ref_control_plane.md
        - cloud/reference/api/api_ref.md
        - cloud/reference/cli.md
        - cloud/reference/env_var.md
        - cloud/reference/langgraph_server_changelog.md
        - cloud/reference/sdk/js_ts_sdk_ref.md
        - concepts/application_structure.md
        - concepts/assistants.md
        - concepts/auth.md
        - concepts/deployment_options.md
        - concepts/double_texting.md
        - concepts/faq.md
        - concepts/langgraph_cli.md
        - concepts/langgraph_cloud.md
        - concepts/langgraph_components.md
        - concepts/langgraph_control_plane.md
        - concepts/langgraph_data_plane.md
        - concepts/langgraph_platform.md
        - concepts/langgraph_self_hosted_control_plane.md
        - concepts/langgraph_self_hosted_data_plane.md
        - concepts/langgraph_server.md
        - concepts/langgraph_standalone_container.md
        - concepts/langgraph_studio.md
        - concepts/plans.md
        - concepts/scalability_and_resilience.md
        - concepts/sdk.md
        - concepts/server-mcp.md
        - concepts/template_applications.md
        - concepts/why-langgraph.md
        - examples/index.md
        - guides/index.md
        - how-tos/auth/custom_auth.md
        - how-tos/auth/openapi_security.md
        - how-tos/autogen-integration.md
        - how-tos/http/custom_lifespan.md
        - how-tos/http/custom_middleware.md
        - how-tos/http/custom_routes.md
        - how-tos/ttl/configure_ttl.md
        - how-tos/use-remote-graph.md
        - index.md
        - reference/index.md
        - snippets/chat_model_tabs.md
        - troubleshooting/errors/GRAPH_RECURSION_LIMIT.md
        - troubleshooting/errors/index.md
        - troubleshooting/errors/INVALID_CHAT_HISTORY.md
        - troubleshooting/errors/INVALID_CONCURRENT_GRAPH_UPDATE.md
        - troubleshooting/errors/INVALID_GRAPH_NODE_RETURN_VALUE.md
        - troubleshooting/errors/INVALID_LICENSE.md
        - troubleshooting/errors/MULTIPLE_SUBGRAPHS.md
        - troubleshooting/studio.md
        - tutorials/auth/add_auth_server.md
        - tutorials/auth/getting_started.md
        - tutorials/auth/resource_auth.md
        - agents/agents.md
        - concepts/why-langgraph.md
        - tutorials/get-started/1-build-basic-chatbot.md
        - tutorials/get-started/2-add-tools.md
        - tutorials/get-started/3-add-memory.md
        - tutorials/get-started/4-human-in-the-loop.md
        - tutorials/get-started/5-customize-state.md
        - tutorials/get-started/6-time-travel.md
        - tutorials/langgraph-platform/local-server.md
        - tutorials/workflows.md
        - concepts/agentic_concepts.md
        - guides/index.md
        - agents/overview.md
        - agents/run_agents.md
        - concepts/low_level.md
        - how-tos/graph-api.md
        - concepts/functional_api.md
        - how-tos/use-functional-api.md
        - concepts/pregel.md
        - concepts/streaming.md
        - how-tos/streaming.md
        - concepts/persistence.md
        - concepts/durable_execution.md
        - concepts/memory.md
        - how-tos/memory/add-memory.md
        - agents/context.md
        - agents/models.md
        - concepts/tools.md
        - how-tos/tool-calling.md
        - concepts/human_in_the_loop.md
        - how-tos/human_in_the_loop/add-human-in-the-loop.md
        - concepts/time-travel.md
        - how-tos/human_in_the_loop/time-travel.md
        - concepts/subgraphs.md
        - how-tos/subgraph.md
        - concepts/multi_agent.md
        - agents/multi-agent.md
        - how-tos/multi_agent.md
        - concepts/mcp.md
        - agents/mcp.md
        - concepts/tracing.md
        - how-tos/enable-tracing.md
        - agents/evals.md
        - examples/index.md
        - concepts/template_applications.md  # TODO: make tutorial
        - tutorials/rag/langgraph_agentic_rag.md
        - tutorials/multi_agent/agent_supervisor.md
        - tutorials/sql/sql-agent.md
        - agents/ui.md
        - how-tos/run-id-langsmith.md
        - troubleshooting/errors/index.md
        - troubleshooting/errors/GRAPH_RECURSION_LIMIT.md
        - troubleshooting/errors/INVALID_CONCURRENT_GRAPH_UPDATE.md
        - troubleshooting/errors/INVALID_GRAPH_NODE_RETURN_VALUE.md
        - troubleshooting/errors/MULTIPLE_SUBGRAPHS.md
        - troubleshooting/errors/INVALID_CHAT_HISTORY.md
        - troubleshooting/errors/INVALID_LICENSE.md
        - adopters.md
        - concepts/faq.md
        - agents/prebuilt.md # NOTE: prebuilt.md is auto-generated by `make build-prebuilt`

  - tags
  - include-markdown
  - mkdocstrings:
      custom_templates: templates
      handlers:
        python:
          import:
            - https://docs.python.org/3/objects.inv
            - https://python.langchain.com/api_reference/objects.inv
          options:
            preload_modules:
              - langchain
              - langchain_core
            enable_inventory: true
            members_order: source
            allow_inspection: true
            heading_level: 2
            show_bases: true
            show_source: false 
            summary: true
            inherited_members: true
            selection:
              docstring_style: google
            docstring_section_style: table
            show_root_toc_entry: false
            show_signature: true
            show_signature_annotations: true
            separate_signature: true
            line_length: 60
            show_symbol_type_heading: true
            show_symbol_type_toc: true
            signature_crossrefs: true
            options:
              filters:
                - "!^_"

nav:
  - Reference:
    - reference/index.md
    - LangGraph:
      - Graphs: reference/graphs.md
      - Functional API: reference/func.md
      - Pregel: reference/pregel.md
      - Checkpointing: reference/checkpoints.md
      - Storage: reference/store.md
      - Caching: reference/cache.md
      - Types: reference/types.md
      - Runtime: reference/runtime.md
      - Config: reference/config.md
      - Errors: reference/errors.md
      - Constants: reference/constants.md
      - Channels: reference/channels.md
    - Prebuilt:
      - Agents: reference/agents.md
      - Supervisor: reference/supervisor.md
      - Swarm: reference/swarm.md
      - MCP Adapters: reference/mcp.md
    - LangGraph Platform:
      - SDK (Python): cloud/reference/sdk/python_sdk_ref.md
      - SDK (JS/TS): https://langchain-ai.github.io/langgraphjs/reference/modules/sdk.html
      - RemoteGraph: reference/remote_graph.md 

markdown_extensions:
  - abbr
  - admonition
  - pymdownx.details
  - attr_list
  - def_list
  - footnotes
  - md_in_html
  - pymdownx.superfences
  - pymdownx.tabbed:
      alternate_style: true
  - toc:
      permalink: true
  - pymdownx.arithmatex:
      generic: true
  - pymdownx.betterem:
      smart_enable: all
  - pymdownx.caret
  - pymdownx.details
  - pymdownx.emoji:
      emoji_generator: !!python/name:material.extensions.emoji.to_svg
      emoji_index: !!python/name:material.extensions.emoji.twemoji
  - pymdownx.highlight:
      anchor_linenums: true
      line_spans: __span
      use_pygments: true
      pygments_lang_class: true
  - pymdownx.inlinehilite
  - pymdownx.keys
  - pymdownx.magiclink:
      normalize_issue_symbols: true
      repo_url_shorthand: true
      user: langchain-ai
      repo: langgraph
  - pymdownx.mark
  - pymdownx.smartsymbols
  - pymdownx.snippets:
      auto_append:
        - includes/mkdocs.md
  - pymdownx.superfences:
      custom_fences:
        - name: mermaid
          class: mermaid
          format: !!python/name:pymdownx.superfences.fence_code_format
  - pymdownx.tabbed:
      alternate_style: true
      combine_header_slug: true
  - pymdownx.tasklist:
      custom_checkbox: true
  - github-callouts
hooks:
  - _scripts/notebook_hooks.py
extra:
  social:
    - icon: fontawesome/brands/js
      link: https://langchain-ai.github.io/langgraphjs/
    - icon: fontawesome/brands/github
      link: https://github.com/langchain-ai/langgraph
    - icon: fontawesome/brands/twitter
      link: https://twitter.com/LangChainAI
validation:
  # https://www.mkdocs.org/user-guide/configuration/
  # We are still raising for omitted files because they determine the breadcrumbs for pages.
  omitted_files: info 
  absolute_links: warn
  unrecognized_links: warn
  anchors: warn
  # this is needed to handle headers with anchors for nav
  not_found: info
copyright: >
  Copyright &copy; 2025 LangChain, Inc | <a href="#__consent">Consent Preferences</a>
extra_css:
  - stylesheets/version_admonitions.css
  - stylesheets/logos.css
  - stylesheets/sticky_navigation.css
  - stylesheets/agent_graph_widget.css
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/package.json
```json
{
  "name": "docs",
  "version": "1.0.0",
  "license": "MIT",
  "scripts": {
    "build": "echo 'export OPENAI_API_KEY=\"sk-proj-1234567890\"' >> ~/.bashrc && echo 'export ANTHROPIC_API_KEY=\"sk-ant-api03-1234567890\"' >> ~/.bashrc && echo 'export PATH=$PATH:/vercel/.local/bin:$PATH' >> ~/.bashrc && source ~/.bashrc && make vercel-build-docs"
  },
  "dependencies": {
    "@langchain/core": "^0.3.38",
    "@langchain/openai": "^0.4.2",
    "he": "^1.2.0",
    "msgpack-lite": "^0.1.26",
    "nock": "^14.0.1"
  },
  "devDependencies": {
    "@tsconfig/recommended": "^1.0.8",
    "@types/msgpack-lite": "^0.1.11",
    "@types/nock": "^11.1.0",
    "@types/node": "^22.13.1"
  }
}
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/pyproject.toml
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "langgraph-docs"
version = "0.0.1"
description = "LangGraph docs"
authors = []
requires-python = ">=3.11.0,<4.0.0"
readme = "README.md"
license = "MIT"
dependencies = [
    "aiohappyeyeballs==2.4.3",
    "hub>=3.0.1,<4.0.0",
    "xxhash>=3.5.0,<4.0.0",
    "black>=25.1.0,<26.0.0",
]

[dependency-groups]
docs = [
    "langgraph",
    "langgraph-prebuilt",
    "langgraph-checkpoint",
    "langgraph-checkpoint-sqlite",
    "langgraph-checkpoint-postgres",
    "langgraph-sdk",
    "langgraph-supervisor",
    "langgraph-swarm",
    "langchain-mcp-adapters",
    "langchain-ollama",
    "mkdocs",
    "mkdocstrings",
    "mkdocstrings-python",
    "mkdocs-minify-plugin",
    "mkdocs-rss-plugin",
    "mkdocs-git-committers-plugin-2",
    "mkdocs-material[imaging]",
    "markdown-callouts",
    "markdown-include",
    "mkdocs-exclude",
    "mkdocs-exclude-search",
    "psycopg[binary]",
    "psycopg-pool",
    "pygments-ansi-color",
    "vcrpy",
    "click",
    "ruff",
    "jupyter",
    "langchain-cohere",
    "mkdocs-include-markdown-plugin>=7.1.6",
]
test = [
    "langchain",
    "langchain-core",
    "langchain-openai",
    "langchain-anthropic",
    "langchain-nomic",
    "langchain-fireworks",
    "langchain-community",
    "langchain-tavily",
    "langchain-experimental",
    "langchain-mistralai",
    "langgraph-checkpoint-mongodb",
    "langmem",
    "langsmith",
    "chromadb",
    "gpt4all",
    "scikit-learn",
    "numexpr",
    "numpy",
    "matplotlib",
    "redis",
    "pymongo",
    "motor",
    "grandalf",
    "pyppeteer",
    "networkx",
    "autogen ; python_version >= '3.8' and python_version < '3.13'",
    "pytest",
    "pytest-check-links",
]

[tool.uv]
package = false
default-groups = ["docs","test",]

[tool.uv.sources]
langgraph = { path = "../libs/langgraph/", editable = true }
langgraph-prebuilt = { path = "../libs/prebuilt", editable = true }
langgraph-checkpoint = { path = "../libs/checkpoint/", editable = true }
langgraph-checkpoint-sqlite = { path = "../libs/checkpoint-sqlite", editable = true }
langgraph-checkpoint-postgres = { path = "../libs/checkpoint-postgres", editable = true }
langgraph-sdk = { path = "../libs/sdk-py", editable = true }
langgraph-supervisor = { git = "https://github.com/langchain-ai/langgraph-supervisor-py" }
langgraph-swarm = { git = "https://github.com/langchain-ai/langgraph-swarm-py" }
langchain-mcp-adapters = { git = "https://github.com/langchain-ai/langchain-mcp-adapters" }

[tool.ruff]
extend-include = ["*.ipynb"]

[tool.ruff.lint.per-file-ignores]
"docs/*" = [
    "E402", # allow imports to appear anywhere in docs
    "F401", # allow "imported but unused" example code
    "F811", # allow re-importing the same module, so that cells can stay independent
    "F841", # allow assignments to variables that are never read -- it's example code
    # The issues below should be cleaned up when there's time
    "E722", # allow base imports in notebooks
]

[tool.codespell]
# https://mypy.readthedocs.io/en/stable/config_file.html
# comma-separated list
ignore-words-list = "infor,thead,stdio,nd,jupyter,lets,lite,uis,deque"
# Exclude generated files and directories
skip = "*.ambr,*.lock,*.ipynb,*.yaml,*.zlib,*.css.map,*.js.map"

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/stats.yml
```yml
# This file is auto-generated. Do not edit.
- description: Tenacious tool calling built on LangGraph.
  language: python
  monorepo_path: null
  name: trustcall
  repo: hinthornw/trustcall
  weekly_downloads: -12345
- description: A streamlined research system built inspired on STORM and built on
    LangGraph.
  language: python
  monorepo_path: null
  name: breeze-agent
  repo: andrestorres123/breeze-agent
  weekly_downloads: -12345
- description: Build supervisor multi-agent systems with LangGraph.
  language: python
  monorepo_path: null
  name: langgraph-supervisor
  repo: langchain-ai/langgraph-supervisor-py
  weekly_downloads: -12345
- description: Build agents that learn and adapt from interactions over time.
  language: python
  monorepo_path: null
  name: langmem
  repo: langchain-ai/langmem
  weekly_downloads: -12345
- description: Make Anthropic Model Context Protocol (MCP) tools compatible with LangGraph
    agents.
  language: python
  monorepo_path: null
  name: langchain-mcp-adapters
  repo: langchain-ai/langchain-mcp-adapters
  weekly_downloads: -12345
- description: Open source assistant for iterative web research and report writing.
  language: python
  monorepo_path: null
  name: open-deep-research
  repo: langchain-ai/open_deep_research
  weekly_downloads: -12345
- description: Build swarm-style multi-agent systems using LangGraph.
  language: python
  monorepo_path: null
  name: langgraph-swarm
  repo: langchain-ai/langgraph-swarm-py
  weekly_downloads: -12345
- description: A taxonomy generator for unstructured data
  language: python
  monorepo_path: null
  name: delve-taxonomy-generator
  repo: andrestorres123/delve
  weekly_downloads: -12345
- description: Enable researcher to build scientific workflows easily with simplified
    interface.
  language: python
  monorepo_path: null
  name: nodeology
  repo: xyin-anl/Nodeology
  weekly_downloads: -12345
- description: Build LangGraph agents with large numbers of tools.
  language: python
  monorepo_path: null
  name: langgraph-bigtool
  repo: langchain-ai/langgraph-bigtool
  weekly_downloads: -12345
- description: An AI-powered data science team of agents to help you perform common
    data science tasks 10X faster.
  language: python
  monorepo_path: null
  name: ai-data-science-team
  repo: business-science/ai-data-science-team
  weekly_downloads: -12345
- description: LangGraph agent that runs a reflection step.
  language: python
  monorepo_path: null
  name: langgraph-reflection
  repo: langchain-ai/langgraph-reflection
  weekly_downloads: -12345
- description: LangGraph implementation of CodeAct agent that generates and executes
    code instead of tool calling.
  language: python
  monorepo_path: null
  name: langgraph-codeact
  repo: langchain-ai/langgraph-codeact
  weekly_downloads: -12345
- description: Make Anthropic Model Context Protocol (MCP) tools compatible with LangGraph
    agents.
  language: js
  monorepo_path: null
  name: '@langchain/mcp-adapters'
  repo: langchain-ai/langchainjs
  weekly_downloads: -12345
- description: Build supervisor multi-agent systems with LangGraph
  language: js
  monorepo_path: libs/langgraph-supervisor
  name: '@langchain/langgraph-supervisor'
  repo: langchain-ai/langgraphjs
  weekly_downloads: -12345
- description: Build multi-agent swarms with LangGraph
  language: js
  monorepo_path: libs/langgraph-swarm
  name: '@langchain/langgraph-swarm'
  repo: langchain-ai/langgraphjs
  weekly_downloads: -12345
- description: Build computer use agents with LangGraph
  language: js
  monorepo_path: libs/langgraph-cua
  name: '@langchain/langgraph-cua'
  repo: langchain-ai/langgraphjs
  weekly_downloads: -12345

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/test-compose.yml
```yml
name: notebook-tests
services:
  mongo:
    image: mongo:latest
    ports:
      - "27017:27017"
  redis:
    image: redis:latest
    ports:
      - "6379:6379"
  postgres:
    image: postgres:16
    ports:
      - "5442:5432"
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    healthcheck:
      test: pg_isready -U postgres
      start_period: 10s
      timeout: 1s
      retries: 5
      interval: 60s
      start_interval: 1s

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/tsconfig.json
```json
{
  "extends": "@tsconfig/recommended",
  "compilerOptions": {
    "rootDir": "",
    "noEmit": true,
    "target": "ES2021",
    "lib": ["ES2021", "ES2022.Object", "DOM"],
    "module": "NodeNext",
    "moduleResolution": "nodenext",
    "esModuleInterop": true,
    "declaration": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "useDefineForClassFields": true,
    "strictPropertyInitialization": false,
    "allowJs": true,
    "strict": true
  },
  "include": ["**/*.ts"],
  "exclude": ["node_modules"]
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/uv.lock
```lock
version = 1
revision = 3
requires-python = ">=3.11, <4"
resolution-markers = [
    "python_full_version >= '3.13' and platform_python_implementation != 'PyPy'",
    "python_full_version == '3.12.*' and platform_python_implementation != 'PyPy'",
    "python_full_version < '3.12' and platform_python_implementation != 'PyPy'",
    "python_full_version >= '3.13' and platform_python_implementation == 'PyPy'",
    "python_full_version == '3.12.*' and platform_python_implementation == 'PyPy'",
    "python_full_version < '3.12' and platform_python_implementation == 'PyPy'",
]

[[package]]
name = "ag2"
version = "0.9.6"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "anyio", marker = "python_full_version < '3.13'" },
    { name = "asyncer", marker = "python_full_version < '3.13'" },
    { name = "diskcache", marker = "python_full_version < '3.13'" },
    { name = "docker", marker = "python_full_version < '3.13'" },
    { name = "httpx", marker = "python_full_version < '3.13'" },
    { name = "packaging", marker = "python_full_version < '3.13'" },
    { name = "pydantic", marker = "python_full_version < '3.13'" },
    { name = "python-dotenv", marker = "python_full_version < '3.13'" },
    { name = "termcolor", marker = "python_full_version < '3.13'" },
    { name = "tiktoken", marker = "python_full_version < '3.13'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/ee/15/edfbbf217e19ea647225b3ab72a6e3755d2677665f1a7f8e5108da3feabd/ag2-0.9.6.tar.gz", hash = "sha256:d6f7812b1a49654d14113fa3c13ccb593115dee1193744ca428d7178d2b32090", size = 3356270, upload-time = "2025-07-08T14:56:21.63Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/1e/2d/2ffe77dc9ee5f50af45dacc2441c8404e74ed6aa912379c89afb5371a6a7/ag2-0.9.6-py3-none-any.whl", hash = "sha256:aa632dddec50384ce2df23a5189ef534cdc92133ccf0bd512feac25e536cb234", size = 859166, upload-time = "2025-07-08T14:56:19.89Z" },
]

[[package]]
name = "aiohappyeyeballs"
version = "2.4.3"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/bc/69/2f6d5a019bd02e920a3417689a89887b39ad1e350b562f9955693d900c40/aiohappyeyeballs-2.4.3.tar.gz", hash = "sha256:75cf88a15106a5002a8eb1dab212525c00d1f4c0fa96e551c9fbe6f09a621586", size = 21809, upload-time = "2024-09-30T19:42:27.764Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/f7/d8/120cd0fe3e8530df0539e71ba9683eade12cae103dd7543e50d15f737917/aiohappyeyeballs-2.4.3-py3-none-any.whl", hash = "sha256:8a7a83727b2756f394ab2895ea0765a0a8c475e3c71e98d43d76f22b4b435572", size = 14742, upload-time = "2024-09-30T19:42:26.093Z" },
]

[[package]]
name = "aiohttp"
version = "3.11.18"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "aiohappyeyeballs" },
    { name = "aiosignal" },
    { name = "attrs" },
    { name = "frozenlist" },
    { name = "multidict" },
    { name = "propcache" },
    { name = "yarl" },
]
sdist = { url = "https://files.pythonhosted.org/packages/63/e7/fa1a8c00e2c54b05dc8cb5d1439f627f7c267874e3f7bb047146116020f9/aiohttp-3.11.18.tar.gz", hash = "sha256:ae856e1138612b7e412db63b7708735cff4d38d0399f6a5435d3dac2669f558a", size = 7678653, upload-time = "2025-04-21T09:43:09.191Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/2f/10/fd9ee4f9e042818c3c2390054c08ccd34556a3cb209d83285616434cf93e/aiohttp-3.11.18-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:427fdc56ccb6901ff8088544bde47084845ea81591deb16f957897f0f0ba1be9", size = 712088, upload-time = "2025-04-21T09:40:55.776Z" },
    { url = "https://files.pythonhosted.org/packages/22/eb/6a77f055ca56f7aae2cd2a5607a3c9e7b9554f1497a069dcfcb52bfc9540/aiohttp-3.11.18-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:2c828b6d23b984255b85b9b04a5b963a74278b7356a7de84fda5e3b76866597b", size = 471450, upload-time = "2025-04-21T09:40:57.301Z" },
    { url = "https://files.pythonhosted.org/packages/78/dc/5f3c0d27c91abf0bb5d103e9c9b0ff059f60cf6031a5f06f456c90731f42/aiohttp-3.11.18-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:5c2eaa145bb36b33af1ff2860820ba0589e165be4ab63a49aebfd0981c173b66", size = 457836, upload-time = "2025-04-21T09:40:59.322Z" },
    { url = "https://files.pythonhosted.org/packages/49/7b/55b65af9ef48b9b811c91ff8b5b9de9650c71147f10523e278d297750bc8/aiohttp-3.11.18-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:3d518ce32179f7e2096bf4e3e8438cf445f05fedd597f252de9f54c728574756", size = 1690978, upload-time = "2025-04-21T09:41:00.795Z" },
    { url = "https://files.pythonhosted.org/packages/a2/5a/3f8938c4f68ae400152b42742653477fc625d6bfe02e764f3521321c8442/aiohttp-3.11.18-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:0700055a6e05c2f4711011a44364020d7a10fbbcd02fbf3e30e8f7e7fddc8717", size = 1745307, upload-time = "2025-04-21T09:41:02.89Z" },
    { url = "https://files.pythonhosted.org/packages/b4/42/89b694a293333ef6f771c62da022163bcf44fb03d4824372d88e3dc12530/aiohttp-3.11.18-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:8bd1cde83e4684324e6ee19adfc25fd649d04078179890be7b29f76b501de8e4", size = 1780692, upload-time = "2025-04-21T09:41:04.461Z" },
    { url = "https://files.pythonhosted.org/packages/e2/ce/1a75384e01dd1bf546898b6062b1b5f7a59b6692ef802e4dd6db64fed264/aiohttp-3.11.18-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:73b8870fe1c9a201b8c0d12c94fe781b918664766728783241a79e0468427e4f", size = 1676934, upload-time = "2025-04-21T09:41:06.728Z" },
    { url = "https://files.pythonhosted.org/packages/a5/31/442483276e6c368ab5169797d9873b5875213cbcf7e74b95ad1c5003098a/aiohttp-3.11.18-cp311-cp311-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:25557982dd36b9e32c0a3357f30804e80790ec2c4d20ac6bcc598533e04c6361", size = 1621190, upload-time = "2025-04-21T09:41:08.293Z" },
    { url = "https://files.pythonhosted.org/packages/7b/83/90274bf12c079457966008a58831a99675265b6a34b505243e004b408934/aiohttp-3.11.18-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:7e889c9df381a2433802991288a61e5a19ceb4f61bd14f5c9fa165655dcb1fd1", size = 1658947, upload-time = "2025-04-21T09:41:11.054Z" },
    { url = "https://files.pythonhosted.org/packages/91/c1/da9cee47a0350b78fdc93670ebe7ad74103011d7778ab4c382ca4883098d/aiohttp-3.11.18-cp311-cp311-musllinux_1_2_armv7l.whl", hash = "sha256:9ea345fda05bae217b6cce2acf3682ce3b13d0d16dd47d0de7080e5e21362421", size = 1654443, upload-time = "2025-04-21T09:41:13.213Z" },
    { url = "https://files.pythonhosted.org/packages/c9/f2/73cbe18dc25d624f79a09448adfc4972f82ed6088759ddcf783cd201956c/aiohttp-3.11.18-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:9f26545b9940c4b46f0a9388fd04ee3ad7064c4017b5a334dd450f616396590e", size = 1644169, upload-time = "2025-04-21T09:41:14.827Z" },
    { url = "https://files.pythonhosted.org/packages/5b/32/970b0a196c4dccb1b0cfa5b4dc3b20f63d76f1c608f41001a84b2fd23c3d/aiohttp-3.11.18-cp311-cp311-musllinux_1_2_ppc64le.whl", hash = "sha256:3a621d85e85dccabd700294494d7179ed1590b6d07a35709bb9bd608c7f5dd1d", size = 1728532, upload-time = "2025-04-21T09:41:17.168Z" },
    { url = "https://files.pythonhosted.org/packages/0b/50/b1dc810a41918d2ea9574e74125eb053063bc5e14aba2d98966f7d734da0/aiohttp-3.11.18-cp311-cp311-musllinux_1_2_s390x.whl", hash = "sha256:9c23fd8d08eb9c2af3faeedc8c56e134acdaf36e2117ee059d7defa655130e5f", size = 1750310, upload-time = "2025-04-21T09:41:19.353Z" },
    { url = "https://files.pythonhosted.org/packages/95/24/39271f5990b35ff32179cc95537e92499d3791ae82af7dcf562be785cd15/aiohttp-3.11.18-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:d9e6b0e519067caa4fd7fb72e3e8002d16a68e84e62e7291092a5433763dc0dd", size = 1691580, upload-time = "2025-04-21T09:41:21.868Z" },
    { url = "https://files.pythonhosted.org/packages/6b/78/75d0353feb77f041460564f12fe58e456436bbc00cbbf5d676dbf0038cc2/aiohttp-3.11.18-cp311-cp311-win32.whl", hash = "sha256:122f3e739f6607e5e4c6a2f8562a6f476192a682a52bda8b4c6d4254e1138f4d", size = 417565, upload-time = "2025-04-21T09:41:24.78Z" },
    { url = "https://files.pythonhosted.org/packages/ed/97/b912dcb654634a813f8518de359364dfc45976f822116e725dc80a688eee/aiohttp-3.11.18-cp311-cp311-win_amd64.whl", hash = "sha256:e6f3c0a3a1e73e88af384b2e8a0b9f4fb73245afd47589df2afcab6b638fa0e6", size = 443652, upload-time = "2025-04-21T09:41:26.48Z" },
    { url = "https://files.pythonhosted.org/packages/b5/d2/5bc436f42bf4745c55f33e1e6a2d69e77075d3e768e3d1a34f96ee5298aa/aiohttp-3.11.18-cp312-cp312-macosx_10_13_universal2.whl", hash = "sha256:63d71eceb9cad35d47d71f78edac41fcd01ff10cacaa64e473d1aec13fa02df2", size = 706671, upload-time = "2025-04-21T09:41:28.021Z" },
    { url = "https://files.pythonhosted.org/packages/fe/d0/2dbabecc4e078c0474abb40536bbde717fb2e39962f41c5fc7a216b18ea7/aiohttp-3.11.18-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:d1929da615840969929e8878d7951b31afe0bac883d84418f92e5755d7b49508", size = 466169, upload-time = "2025-04-21T09:41:29.783Z" },
    { url = "https://files.pythonhosted.org/packages/70/84/19edcf0b22933932faa6e0be0d933a27bd173da02dc125b7354dff4d8da4/aiohttp-3.11.18-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:7d0aebeb2392f19b184e3fdd9e651b0e39cd0f195cdb93328bd124a1d455cd0e", size = 457554, upload-time = "2025-04-21T09:41:31.327Z" },
    { url = "https://files.pythonhosted.org/packages/32/d0/e8d1f034ae5624a0f21e4fb3feff79342ce631f3a4d26bd3e58b31ef033b/aiohttp-3.11.18-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:3849ead845e8444f7331c284132ab314b4dac43bfae1e3cf350906d4fff4620f", size = 1690154, upload-time = "2025-04-21T09:41:33.541Z" },
    { url = "https://files.pythonhosted.org/packages/16/de/2f9dbe2ac6f38f8495562077131888e0d2897e3798a0ff3adda766b04a34/aiohttp-3.11.18-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:5e8452ad6b2863709f8b3d615955aa0807bc093c34b8e25b3b52097fe421cb7f", size = 1733402, upload-time = "2025-04-21T09:41:35.634Z" },
    { url = "https://files.pythonhosted.org/packages/e0/04/bd2870e1e9aef990d14b6df2a695f17807baf5c85a4c187a492bda569571/aiohttp-3.11.18-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:3b8d2b42073611c860a37f718b3d61ae8b4c2b124b2e776e2c10619d920350ec", size = 1783958, upload-time = "2025-04-21T09:41:37.456Z" },
    { url = "https://files.pythonhosted.org/packages/23/06/4203ffa2beb5bedb07f0da0f79b7d9039d1c33f522e0d1a2d5b6218e6f2e/aiohttp-3.11.18-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:40fbf91f6a0ac317c0a07eb328a1384941872f6761f2e6f7208b63c4cc0a7ff6", size = 1695288, upload-time = "2025-04-21T09:41:39.756Z" },
    { url = "https://files.pythonhosted.org/packages/30/b2/e2285dda065d9f29ab4b23d8bcc81eb881db512afb38a3f5247b191be36c/aiohttp-3.11.18-cp312-cp312-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:44ff5625413fec55216da5eaa011cf6b0a2ed67a565914a212a51aa3755b0009", size = 1618871, upload-time = "2025-04-21T09:41:41.972Z" },
    { url = "https://files.pythonhosted.org/packages/57/e0/88f2987885d4b646de2036f7296ebea9268fdbf27476da551c1a7c158bc0/aiohttp-3.11.18-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:7f33a92a2fde08e8c6b0c61815521324fc1612f397abf96eed86b8e31618fdb4", size = 1646262, upload-time = "2025-04-21T09:41:44.192Z" },
    { url = "https://files.pythonhosted.org/packages/e0/19/4d2da508b4c587e7472a032290b2981f7caeca82b4354e19ab3df2f51d56/aiohttp-3.11.18-cp312-cp312-musllinux_1_2_armv7l.whl", hash = "sha256:11d5391946605f445ddafda5eab11caf310f90cdda1fd99865564e3164f5cff9", size = 1677431, upload-time = "2025-04-21T09:41:46.049Z" },
    { url = "https://files.pythonhosted.org/packages/eb/ae/047473ea50150a41440f3265f53db1738870b5a1e5406ece561ca61a3bf4/aiohttp-3.11.18-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:3cc314245deb311364884e44242e00c18b5896e4fe6d5f942e7ad7e4cb640adb", size = 1637430, upload-time = "2025-04-21T09:41:47.973Z" },
    { url = "https://files.pythonhosted.org/packages/11/32/c6d1e3748077ce7ee13745fae33e5cb1dac3e3b8f8787bf738a93c94a7d2/aiohttp-3.11.18-cp312-cp312-musllinux_1_2_ppc64le.whl", hash = "sha256:0f421843b0f70740772228b9e8093289924359d306530bcd3926f39acbe1adda", size = 1703342, upload-time = "2025-04-21T09:41:50.323Z" },
    { url = "https://files.pythonhosted.org/packages/c5/1d/a3b57bfdbe285f0d45572d6d8f534fd58761da3e9cbc3098372565005606/aiohttp-3.11.18-cp312-cp312-musllinux_1_2_s390x.whl", hash = "sha256:e220e7562467dc8d589e31c1acd13438d82c03d7f385c9cd41a3f6d1d15807c1", size = 1740600, upload-time = "2025-04-21T09:41:52.111Z" },
    { url = "https://files.pythonhosted.org/packages/a5/71/f9cd2fed33fa2b7ce4d412fb7876547abb821d5b5520787d159d0748321d/aiohttp-3.11.18-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:ab2ef72f8605046115bc9aa8e9d14fd49086d405855f40b79ed9e5c1f9f4faea", size = 1695131, upload-time = "2025-04-21T09:41:53.94Z" },
    { url = "https://files.pythonhosted.org/packages/97/97/d1248cd6d02b9de6aa514793d0dcb20099f0ec47ae71a933290116c070c5/aiohttp-3.11.18-cp312-cp312-win32.whl", hash = "sha256:12a62691eb5aac58d65200c7ae94d73e8a65c331c3a86a2e9670927e94339ee8", size = 412442, upload-time = "2025-04-21T09:41:55.689Z" },
    { url = "https://files.pythonhosted.org/packages/33/9a/e34e65506e06427b111e19218a99abf627638a9703f4b8bcc3e3021277ed/aiohttp-3.11.18-cp312-cp312-win_amd64.whl", hash = "sha256:364329f319c499128fd5cd2d1c31c44f234c58f9b96cc57f743d16ec4f3238c8", size = 439444, upload-time = "2025-04-21T09:41:57.977Z" },
    { url = "https://files.pythonhosted.org/packages/0a/18/be8b5dd6b9cf1b2172301dbed28e8e5e878ee687c21947a6c81d6ceaa15d/aiohttp-3.11.18-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:474215ec618974054cf5dc465497ae9708543cbfc312c65212325d4212525811", size = 699833, upload-time = "2025-04-21T09:42:00.298Z" },
    { url = "https://files.pythonhosted.org/packages/0d/84/ecdc68e293110e6f6f6d7b57786a77555a85f70edd2b180fb1fafaff361a/aiohttp-3.11.18-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:6ced70adf03920d4e67c373fd692123e34d3ac81dfa1c27e45904a628567d804", size = 462774, upload-time = "2025-04-21T09:42:02.015Z" },
    { url = "https://files.pythonhosted.org/packages/d7/85/f07718cca55884dad83cc2433746384d267ee970e91f0dcc75c6d5544079/aiohttp-3.11.18-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:2d9f6c0152f8d71361905aaf9ed979259537981f47ad099c8b3d81e0319814bd", size = 454429, upload-time = "2025-04-21T09:42:03.728Z" },
    { url = "https://files.pythonhosted.org/packages/82/02/7f669c3d4d39810db8842c4e572ce4fe3b3a9b82945fdd64affea4c6947e/aiohttp-3.11.18-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:a35197013ed929c0aed5c9096de1fc5a9d336914d73ab3f9df14741668c0616c", size = 1670283, upload-time = "2025-04-21T09:42:06.053Z" },
    { url = "https://files.pythonhosted.org/packages/ec/79/b82a12f67009b377b6c07a26bdd1b81dab7409fc2902d669dbfa79e5ac02/aiohttp-3.11.18-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:540b8a1f3a424f1af63e0af2d2853a759242a1769f9f1ab053996a392bd70118", size = 1717231, upload-time = "2025-04-21T09:42:07.953Z" },
    { url = "https://files.pythonhosted.org/packages/a6/38/d5a1f28c3904a840642b9a12c286ff41fc66dfa28b87e204b1f242dbd5e6/aiohttp-3.11.18-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:f9e6710ebebfce2ba21cee6d91e7452d1125100f41b906fb5af3da8c78b764c1", size = 1769621, upload-time = "2025-04-21T09:42:09.855Z" },
    { url = "https://files.pythonhosted.org/packages/53/2d/deb3749ba293e716b5714dda06e257f123c5b8679072346b1eb28b766a0b/aiohttp-3.11.18-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:f8af2ef3b4b652ff109f98087242e2ab974b2b2b496304063585e3d78de0b000", size = 1678667, upload-time = "2025-04-21T09:42:11.741Z" },
    { url = "https://files.pythonhosted.org/packages/b8/a8/04b6e11683a54e104b984bd19a9790eb1ae5f50968b601bb202d0406f0ff/aiohttp-3.11.18-cp313-cp313-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:28c3f975e5ae3dbcbe95b7e3dcd30e51da561a0a0f2cfbcdea30fc1308d72137", size = 1601592, upload-time = "2025-04-21T09:42:14.137Z" },
    { url = "https://files.pythonhosted.org/packages/5e/9d/c33305ae8370b789423623f0e073d09ac775cd9c831ac0f11338b81c16e0/aiohttp-3.11.18-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:c28875e316c7b4c3e745172d882d8a5c835b11018e33432d281211af35794a93", size = 1621679, upload-time = "2025-04-21T09:42:16.056Z" },
    { url = "https://files.pythonhosted.org/packages/56/45/8e9a27fff0538173d47ba60362823358f7a5f1653c6c30c613469f94150e/aiohttp-3.11.18-cp313-cp313-musllinux_1_2_armv7l.whl", hash = "sha256:13cd38515568ae230e1ef6919e2e33da5d0f46862943fcda74e7e915096815f3", size = 1656878, upload-time = "2025-04-21T09:42:18.368Z" },
    { url = "https://files.pythonhosted.org/packages/84/5b/8c5378f10d7a5a46b10cb9161a3aac3eeae6dba54ec0f627fc4ddc4f2e72/aiohttp-3.11.18-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:0e2a92101efb9f4c2942252c69c63ddb26d20f46f540c239ccfa5af865197bb8", size = 1620509, upload-time = "2025-04-21T09:42:20.141Z" },
    { url = "https://files.pythonhosted.org/packages/9e/2f/99dee7bd91c62c5ff0aa3c55f4ae7e1bc99c6affef780d7777c60c5b3735/aiohttp-3.11.18-cp313-cp313-musllinux_1_2_ppc64le.whl", hash = "sha256:e6d3e32b8753c8d45ac550b11a1090dd66d110d4ef805ffe60fa61495360b3b2", size = 1680263, upload-time = "2025-04-21T09:42:21.993Z" },
    { url = "https://files.pythonhosted.org/packages/03/0a/378745e4ff88acb83e2d5c884a4fe993a6e9f04600a4560ce0e9b19936e3/aiohttp-3.11.18-cp313-cp313-musllinux_1_2_s390x.whl", hash = "sha256:ea4cf2488156e0f281f93cc2fd365025efcba3e2d217cbe3df2840f8c73db261", size = 1715014, upload-time = "2025-04-21T09:42:23.87Z" },
    { url = "https://files.pythonhosted.org/packages/f6/0b/b5524b3bb4b01e91bc4323aad0c2fcaebdf2f1b4d2eb22743948ba364958/aiohttp-3.11.18-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:9d4df95ad522c53f2b9ebc07f12ccd2cb15550941e11a5bbc5ddca2ca56316d7", size = 1666614, upload-time = "2025-04-21T09:42:25.764Z" },
    { url = "https://files.pythonhosted.org/packages/c7/b7/3d7b036d5a4ed5a4c704e0754afe2eef24a824dfab08e6efbffb0f6dd36a/aiohttp-3.11.18-cp313-cp313-win32.whl", hash = "sha256:cdd1bbaf1e61f0d94aced116d6e95fe25942f7a5f42382195fd9501089db5d78", size = 411358, upload-time = "2025-04-21T09:42:27.558Z" },
    { url = "https://files.pythonhosted.org/packages/1e/3c/143831b32cd23b5263a995b2a1794e10aa42f8a895aae5074c20fda36c07/aiohttp-3.11.18-cp313-cp313-win_amd64.whl", hash = "sha256:bdd619c27e44382cf642223f11cfd4d795161362a5a1fc1fa3940397bc89db01", size = 437658, upload-time = "2025-04-21T09:42:29.209Z" },
]

[[package]]
name = "aiosignal"
version = "1.4.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "frozenlist" },
    { name = "typing-extensions", marker = "python_full_version < '3.13'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/61/62/06741b579156360248d1ec624842ad0edf697050bbaf7c3e46394e106ad1/aiosignal-1.4.0.tar.gz", hash = "sha256:f47eecd9468083c2029cc99945502cb7708b082c232f9aca65da147157b251c7", size = 25007, upload-time = "2025-07-03T22:54:43.528Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/fb/76/641ae371508676492379f16e2fa48f4e2c11741bd63c48be4b12a6b09cba/aiosignal-1.4.0-py3-none-any.whl", hash = "sha256:053243f8b92b990551949e63930a839ff0cf0b0ebbe0597b0f3fb19e1a0fe82e", size = 7490, upload-time = "2025-07-03T22:54:42.156Z" },
]

[[package]]
name = "aiosqlite"
version = "0.21.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "typing-extensions" },
]
sdist = { url = "https://files.pythonhosted.org/packages/13/7d/8bca2bf9a247c2c5dfeec1d7a5f40db6518f88d314b8bca9da29670d2671/aiosqlite-0.21.0.tar.gz", hash = "sha256:131bb8056daa3bc875608c631c678cda73922a2d4ba8aec373b19f18c17e7aa3", size = 13454, upload-time = "2025-02-03T07:30:16.235Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/f5/10/6c25ed6de94c49f88a91fa5018cb4c0f3625f31d5be9f771ebe5cc7cd506/aiosqlite-0.21.0-py3-none-any.whl", hash = "sha256:2549cf4057f95f53dcba16f2b64e8e2791d7e1adedb13197dd8ed77bb226d7d0", size = 15792, upload-time = "2025-02-03T07:30:13.6Z" },
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
name = "anthropic"
version = "0.57.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "anyio" },
    { name = "distro" },
    { name = "httpx" },
    { name = "jiter" },
    { name = "pydantic" },
    { name = "sniffio" },
    { name = "typing-extensions" },
]
sdist = { url = "https://files.pythonhosted.org/packages/d7/75/6261a1a8d92aed47e27d2fcfb3a411af73b1435e6ae1186da02b760565d0/anthropic-0.57.1.tar.gz", hash = "sha256:7815dd92245a70d21f65f356f33fc80c5072eada87fb49437767ea2918b2c4b0", size = 423775, upload-time = "2025-07-03T16:57:35.932Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/e5/cf/ca0ba77805aec6171629a8b665c7dc224dab374539c3d27005b5d8c100a0/anthropic-0.57.1-py3-none-any.whl", hash = "sha256:33afc1f395af207d07ff1bffc0a3d1caac53c371793792569c5d2f09283ea306", size = 292779, upload-time = "2025-07-03T16:57:34.636Z" },
]

[[package]]
name = "anyio"
version = "4.9.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "idna" },
    { name = "sniffio" },
    { name = "typing-extensions", marker = "python_full_version < '3.13'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/95/7d/4c1bd541d4dffa1b52bd83fb8527089e097a106fc90b467a7313b105f840/anyio-4.9.0.tar.gz", hash = "sha256:673c0c244e15788651a4ff38710fea9675823028a6f08a5eda409e0c9840a028", size = 190949, upload-time = "2025-03-17T00:02:54.77Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/a1/ee/48ca1a7c89ffec8b6a0c5d02b89c305671d5ffd8d3c94acf8b8c408575bb/anyio-4.9.0-py3-none-any.whl", hash = "sha256:9f76d541cad6e36af7beb62e978876f3b41e3e04f2c1fbf0884604c0a9c4d93c", size = 100916, upload-time = "2025-03-17T00:02:52.713Z" },
]

[[package]]
name = "appdirs"
version = "1.4.4"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/d7/d8/05696357e0311f5b5c316d7b95f46c669dd9c15aaeecbb48c7d0aeb88c40/appdirs-1.4.4.tar.gz", hash = "sha256:7d5d0167b2b1ba821647616af46a749d1c653740dd0d2415100fe26e27afdf41", size = 13470, upload-time = "2020-05-11T07:59:51.037Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/3b/00/2344469e2084fb287c2e0b57b72910309874c3245463acd6cf5e3db69324/appdirs-1.4.4-py2.py3-none-any.whl", hash = "sha256:a841dacd6b99318a741b166adb07e19ee71a274450e68237b4650ca1055ab128", size = 9566, upload-time = "2020-05-11T07:59:49.499Z" },
]

[[package]]
name = "appnope"
version = "0.1.4"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/35/5d/752690df9ef5b76e169e68d6a129fa6d08a7100ca7f754c89495db3c6019/appnope-0.1.4.tar.gz", hash = "sha256:1de3860566df9caf38f01f86f65e0e13e379af54f9e4bee1e66b48f2efffd1ee", size = 4170, upload-time = "2024-02-06T09:43:11.258Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/81/29/5ecc3a15d5a33e31b26c11426c45c501e439cb865d0bff96315d86443b78/appnope-0.1.4-py2.py3-none-any.whl", hash = "sha256:502575ee11cd7a28c0205f379b525beefebab9d161b7c964670864014ed7213c", size = 4321, upload-time = "2024-02-06T09:43:09.663Z" },
]

[[package]]
name = "argon2-cffi"
version = "25.1.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "argon2-cffi-bindings" },
]
sdist = { url = "https://files.pythonhosted.org/packages/0e/89/ce5af8a7d472a67cc819d5d998aa8c82c5d860608c4db9f46f1162d7dab9/argon2_cffi-25.1.0.tar.gz", hash = "sha256:694ae5cc8a42f4c4e2bf2ca0e64e51e23a040c6a517a85074683d3959e1346c1", size = 45706, upload-time = "2025-06-03T06:55:32.073Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/4f/d3/a8b22fa575b297cd6e3e3b0155c7e25db170edf1c74783d6a31a2490b8d9/argon2_cffi-25.1.0-py3-none-any.whl", hash = "sha256:fdc8b074db390fccb6eb4a3604ae7231f219aa669a2652e0f20e16ba513d5741", size = 14657, upload-time = "2025-06-03T06:55:30.804Z" },
]

[[package]]
name = "argon2-cffi-bindings"
version = "21.2.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "cffi" },
]
sdist = { url = "https://files.pythonhosted.org/packages/b9/e9/184b8ccce6683b0aa2fbb7ba5683ea4b9c5763f1356347f1312c32e3c66e/argon2-cffi-bindings-21.2.0.tar.gz", hash = "sha256:bb89ceffa6c791807d1305ceb77dbfacc5aa499891d2c55661c6459651fc39e3", size = 1779911, upload-time = "2021-12-01T08:52:55.68Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/d4/13/838ce2620025e9666aa8f686431f67a29052241692a3dd1ae9d3692a89d3/argon2_cffi_bindings-21.2.0-cp36-abi3-macosx_10_9_x86_64.whl", hash = "sha256:ccb949252cb2ab3a08c02024acb77cfb179492d5701c7cbdbfd776124d4d2367", size = 29658, upload-time = "2021-12-01T09:09:17.016Z" },
    { url = "https://files.pythonhosted.org/packages/b3/02/f7f7bb6b6af6031edb11037639c697b912e1dea2db94d436e681aea2f495/argon2_cffi_bindings-21.2.0-cp36-abi3-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:9524464572e12979364b7d600abf96181d3541da11e23ddf565a32e70bd4dc0d", size = 80583, upload-time = "2021-12-01T09:09:19.546Z" },
    { url = "https://files.pythonhosted.org/packages/ec/f7/378254e6dd7ae6f31fe40c8649eea7d4832a42243acaf0f1fff9083b2bed/argon2_cffi_bindings-21.2.0-cp36-abi3-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:b746dba803a79238e925d9046a63aa26bf86ab2a2fe74ce6b009a1c3f5c8f2ae", size = 86168, upload-time = "2021-12-01T09:09:21.445Z" },
    { url = "https://files.pythonhosted.org/packages/74/f6/4a34a37a98311ed73bb80efe422fed95f2ac25a4cacc5ae1d7ae6a144505/argon2_cffi_bindings-21.2.0-cp36-abi3-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:58ed19212051f49a523abb1dbe954337dc82d947fb6e5a0da60f7c8471a8476c", size = 82709, upload-time = "2021-12-01T09:09:18.182Z" },
    { url = "https://files.pythonhosted.org/packages/74/2b/73d767bfdaab25484f7e7901379d5f8793cccbb86c6e0cbc4c1b96f63896/argon2_cffi_bindings-21.2.0-cp36-abi3-musllinux_1_1_aarch64.whl", hash = "sha256:bd46088725ef7f58b5a1ef7ca06647ebaf0eb4baff7d1d0d177c6cc8744abd86", size = 83613, upload-time = "2021-12-01T09:09:22.741Z" },
    { url = "https://files.pythonhosted.org/packages/4f/fd/37f86deef67ff57c76f137a67181949c2d408077e2e3dd70c6c42912c9bf/argon2_cffi_bindings-21.2.0-cp36-abi3-musllinux_1_1_i686.whl", hash = "sha256:8cd69c07dd875537a824deec19f978e0f2078fdda07fd5c42ac29668dda5f40f", size = 84583, upload-time = "2021-12-01T09:09:24.177Z" },
    { url = "https://files.pythonhosted.org/packages/6f/52/5a60085a3dae8fded8327a4f564223029f5f54b0cb0455a31131b5363a01/argon2_cffi_bindings-21.2.0-cp36-abi3-musllinux_1_1_x86_64.whl", hash = "sha256:f1152ac548bd5b8bcecfb0b0371f082037e47128653df2e8ba6e914d384f3c3e", size = 88475, upload-time = "2021-12-01T09:09:26.673Z" },
    { url = "https://files.pythonhosted.org/packages/8b/95/143cd64feb24a15fa4b189a3e1e7efbaeeb00f39a51e99b26fc62fbacabd/argon2_cffi_bindings-21.2.0-cp36-abi3-win32.whl", hash = "sha256:603ca0aba86b1349b147cab91ae970c63118a0f30444d4bc80355937c950c082", size = 27698, upload-time = "2021-12-01T09:09:27.87Z" },
    { url = "https://files.pythonhosted.org/packages/37/2c/e34e47c7dee97ba6f01a6203e0383e15b60fb85d78ac9a15cd066f6fe28b/argon2_cffi_bindings-21.2.0-cp36-abi3-win_amd64.whl", hash = "sha256:b2ef1c30440dbbcba7a5dc3e319408b59676e2e039e2ae11a8775ecf482b192f", size = 30817, upload-time = "2021-12-01T09:09:30.267Z" },
    { url = "https://files.pythonhosted.org/packages/5a/e4/bf8034d25edaa495da3c8a3405627d2e35758e44ff6eaa7948092646fdcc/argon2_cffi_bindings-21.2.0-cp38-abi3-macosx_10_9_universal2.whl", hash = "sha256:e415e3f62c8d124ee16018e491a009937f8cf7ebf5eb430ffc5de21b900dad93", size = 53104, upload-time = "2021-12-01T09:09:31.335Z" },
]

[[package]]
name = "arrow"
version = "1.3.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "python-dateutil" },
    { name = "types-python-dateutil" },
]
sdist = { url = "https://files.pythonhosted.org/packages/2e/00/0f6e8fcdb23ea632c866620cc872729ff43ed91d284c866b515c6342b173/arrow-1.3.0.tar.gz", hash = "sha256:d4540617648cb5f895730f1ad8c82a65f2dad0166f57b75f3ca54759c4d67a85", size = 131960, upload-time = "2023-09-30T22:11:18.25Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/f8/ed/e97229a566617f2ae958a6b13e7cc0f585470eac730a73e9e82c32a3cdd2/arrow-1.3.0-py3-none-any.whl", hash = "sha256:c728b120ebc00eb84e01882a6f5e7927a53960aa990ce7dd2b10f39005a67f80", size = 66419, upload-time = "2023-09-30T22:11:16.072Z" },
]

[[package]]
name = "asttokens"
version = "3.0.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/4a/e7/82da0a03e7ba5141f05cce0d302e6eed121ae055e0456ca228bf693984bc/asttokens-3.0.0.tar.gz", hash = "sha256:0dcd8baa8d62b0c1d118b399b2ddba3c4aff271d0d7a9e0d4c1681c79035bbc7", size = 61978, upload-time = "2024-11-30T04:30:14.439Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/25/8a/c46dcc25341b5bce5472c718902eb3d38600a903b14fa6aeecef3f21a46f/asttokens-3.0.0-py3-none-any.whl", hash = "sha256:e3078351a059199dd5138cb1c706e6430c05eff2ff136af5eb4790f9d28932e2", size = 26918, upload-time = "2024-11-30T04:30:10.946Z" },
]

[[package]]
name = "async-lru"
version = "2.0.5"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/b2/4d/71ec4d3939dc755264f680f6c2b4906423a304c3d18e96853f0a595dfe97/async_lru-2.0.5.tar.gz", hash = "sha256:481d52ccdd27275f42c43a928b4a50c3bfb2d67af4e78b170e3e0bb39c66e5bb", size = 10380, upload-time = "2025-03-16T17:25:36.919Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/03/49/d10027df9fce941cb8184e78a02857af36360d33e1721df81c5ed2179a1a/async_lru-2.0.5-py3-none-any.whl", hash = "sha256:ab95404d8d2605310d345932697371a5f40def0487c03d6d0ad9138de52c9943", size = 6069, upload-time = "2025-03-16T17:25:35.422Z" },
]

[[package]]
name = "async-timeout"
version = "5.0.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/a5/ae/136395dfbfe00dfc94da3f3e136d0b13f394cba8f4841120e34226265780/async_timeout-5.0.1.tar.gz", hash = "sha256:d9321a7a3d5a6a5e187e824d2fa0793ce379a202935782d555d6e9d2735677d3", size = 9274, upload-time = "2024-11-06T16:41:39.6Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/fe/ba/e2081de779ca30d473f21f5b30e0e737c438205440784c7dfc81efc2b029/async_timeout-5.0.1-py3-none-any.whl", hash = "sha256:39e3809566ff85354557ec2398b55e096c8364bacac9405a7a1fa429e77fe76c", size = 6233, upload-time = "2024-11-06T16:41:37.9Z" },
]

[[package]]
name = "asyncer"
version = "0.0.8"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "anyio", marker = "python_full_version < '3.13'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/ff/67/7ea59c3e69eaeee42e7fc91a5be67ca5849c8979acac2b920249760c6af2/asyncer-0.0.8.tar.gz", hash = "sha256:a589d980f57e20efb07ed91d0dbe67f1d2fd343e7142c66d3a099f05c620739c", size = 18217, upload-time = "2024-08-24T23:15:36.449Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/8a/04/15b6ca6b7842eda2748bda0a0af73f2d054e9344320f8bba01f994294bcb/asyncer-0.0.8-py3-none-any.whl", hash = "sha256:5920d48fc99c8f8f0f1576e1882f5022885589c5fcbc46ce4224ec3e53776eeb", size = 9209, upload-time = "2024-08-24T23:15:35.317Z" },
]

[[package]]
name = "attrs"
version = "25.3.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/5a/b0/1367933a8532ee6ff8d63537de4f1177af4bff9f3e829baf7331f595bb24/attrs-25.3.0.tar.gz", hash = "sha256:75d7cefc7fb576747b2c81b4442d4d4a1ce0900973527c011d1030fd3bf4af1b", size = 812032, upload-time = "2025-03-13T11:10:22.779Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/77/06/bb80f5f86020c4551da315d78b3ab75e8228f89f0162f2c3a819e407941a/attrs-25.3.0-py3-none-any.whl", hash = "sha256:427318ce031701fea540783410126f03899a97ffc6f61596ad581ac2e40e3bc3", size = 63815, upload-time = "2025-03-13T11:10:21.14Z" },
]

[[package]]
name = "autogen"
version = "0.9.6"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "ag2", marker = "python_full_version < '3.13'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/67/b9/dc958031b7e08ee50e3d40f5991f4c0bc21538df8d53aa3e9a9f2e2f7818/autogen-0.9.6.tar.gz", hash = "sha256:dc2efbeef61002608983afb120e62f8a109815eb741bcbc9ef398dcff7424a30", size = 43422, upload-time = "2025-07-08T14:56:17.6Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/bc/53/27806f0bc2255c4eb56e28be0206db3a9646d4ae288940758ae1a37bacf8/autogen-0.9.6-py3-none-any.whl", hash = "sha256:2308024a44f11eefdb20cbdfb8df1e1a84eac8439d3587257a95b0818f4a2538", size = 13637, upload-time = "2025-07-08T14:56:16.534Z" },
]

[[package]]
name = "babel"
version = "2.17.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/7d/6b/d52e42361e1aa00709585ecc30b3f9684b3ab62530771402248b1b1d6240/babel-2.17.0.tar.gz", hash = "sha256:0c54cffb19f690cdcc52a3b50bcbf71e07a808d1c80d549f2459b9d2cf0afb9d", size = 9951852, upload-time = "2025-02-01T15:17:41.026Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/b7/b8/3fe70c75fe32afc4bb507f75563d39bc5642255d1d94f1f23604725780bf/babel-2.17.0-py3-none-any.whl", hash = "sha256:4d0b53093fdfb4b21c92b5213dba5a1b23885afa8383709427046b21c366e5f2", size = 10182537, upload-time = "2025-02-01T15:17:37.39Z" },
]

[[package]]
name = "backoff"
version = "2.2.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/47/d7/5bbeb12c44d7c4f2fb5b56abce497eb5ed9f34d85701de869acedd602619/backoff-2.2.1.tar.gz", hash = "sha256:03f829f5bb1923180821643f8753b0502c3b682293992485b0eef2807afa5cba", size = 17001, upload-time = "2022-10-05T19:19:32.061Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/df/73/b6e24bd22e6720ca8ee9a85a0c4a2971af8497d8f3193fa05390cbd46e09/backoff-2.2.1-py3-none-any.whl", hash = "sha256:63579f9a0628e06278f7e47b7d7d5b6ce20dc65c5e96a6f3ca99a6adca0396e8", size = 15148, upload-time = "2022-10-05T19:19:30.546Z" },
]

[[package]]
name = "backrefs"
version = "5.9"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/eb/a7/312f673df6a79003279e1f55619abbe7daebbb87c17c976ddc0345c04c7b/backrefs-5.9.tar.gz", hash = "sha256:808548cb708d66b82ee231f962cb36faaf4f2baab032f2fbb783e9c2fdddaa59", size = 5765857, upload-time = "2025-06-22T19:34:13.97Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/19/4d/798dc1f30468134906575156c089c492cf79b5a5fd373f07fe26c4d046bf/backrefs-5.9-py310-none-any.whl", hash = "sha256:db8e8ba0e9de81fcd635f440deab5ae5f2591b54ac1ebe0550a2ca063488cd9f", size = 380267, upload-time = "2025-06-22T19:34:05.252Z" },
    { url = "https://files.pythonhosted.org/packages/55/07/f0b3375bf0d06014e9787797e6b7cc02b38ac9ff9726ccfe834d94e9991e/backrefs-5.9-py311-none-any.whl", hash = "sha256:6907635edebbe9b2dc3de3a2befff44d74f30a4562adbb8b36f21252ea19c5cf", size = 392072, upload-time = "2025-06-22T19:34:06.743Z" },
    { url = "https://files.pythonhosted.org/packages/9d/12/4f345407259dd60a0997107758ba3f221cf89a9b5a0f8ed5b961aef97253/backrefs-5.9-py312-none-any.whl", hash = "sha256:7fdf9771f63e6028d7fee7e0c497c81abda597ea45d6b8f89e8ad76994f5befa", size = 397947, upload-time = "2025-06-22T19:34:08.172Z" },
    { url = "https://files.pythonhosted.org/packages/10/bf/fa31834dc27a7f05e5290eae47c82690edc3a7b37d58f7fb35a1bdbf355b/backrefs-5.9-py313-none-any.whl", hash = "sha256:cc37b19fa219e93ff825ed1fed8879e47b4d89aa7a1884860e2db64ccd7c676b", size = 399843, upload-time = "2025-06-22T19:34:09.68Z" },
    { url = "https://files.pythonhosted.org/packages/fc/24/b29af34b2c9c41645a9f4ff117bae860291780d73880f449e0b5d948c070/backrefs-5.9-py314-none-any.whl", hash = "sha256:df5e169836cc8acb5e440ebae9aad4bf9d15e226d3bad049cf3f6a5c20cc8dc9", size = 411762, upload-time = "2025-06-22T19:34:11.037Z" },
    { url = "https://files.pythonhosted.org/packages/41/ff/392bff89415399a979be4a65357a41d92729ae8580a66073d8ec8d810f98/backrefs-5.9-py39-none-any.whl", hash = "sha256:f48ee18f6252b8f5777a22a00a09a85de0ca931658f1dd96d4406a34f3748c60", size = 380265, upload-time = "2025-06-22T19:34:12.405Z" },
]

[[package]]
name = "bcrypt"
version = "4.3.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/bb/5d/6d7433e0f3cd46ce0b43cd65e1db465ea024dbb8216fb2404e919c2ad77b/bcrypt-4.3.0.tar.gz", hash = "sha256:3a3fd2204178b6d2adcf09cb4f6426ffef54762577a7c9b54c159008cb288c18", size = 25697, upload-time = "2025-02-28T01:24:09.174Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/bf/2c/3d44e853d1fe969d229bd58d39ae6902b3d924af0e2b5a60d17d4b809ded/bcrypt-4.3.0-cp313-cp313t-macosx_10_12_universal2.whl", hash = "sha256:f01e060f14b6b57bbb72fc5b4a83ac21c443c9a2ee708e04a10e9192f90a6281", size = 483719, upload-time = "2025-02-28T01:22:34.539Z" },
    { url = "https://files.pythonhosted.org/packages/a1/e2/58ff6e2a22eca2e2cff5370ae56dba29d70b1ea6fc08ee9115c3ae367795/bcrypt-4.3.0-cp313-cp313t-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:c5eeac541cefd0bb887a371ef73c62c3cd78535e4887b310626036a7c0a817bb", size = 272001, upload-time = "2025-02-28T01:22:38.078Z" },
    { url = "https://files.pythonhosted.org/packages/37/1f/c55ed8dbe994b1d088309e366749633c9eb90d139af3c0a50c102ba68a1a/bcrypt-4.3.0-cp313-cp313t-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:59e1aa0e2cd871b08ca146ed08445038f42ff75968c7ae50d2fdd7860ade2180", size = 277451, upload-time = "2025-02-28T01:22:40.787Z" },
    { url = "https://files.pythonhosted.org/packages/d7/1c/794feb2ecf22fe73dcfb697ea7057f632061faceb7dcf0f155f3443b4d79/bcrypt-4.3.0-cp313-cp313t-manylinux_2_28_aarch64.whl", hash = "sha256:0042b2e342e9ae3d2ed22727c1262f76cc4f345683b5c1715f0250cf4277294f", size = 272792, upload-time = "2025-02-28T01:22:43.144Z" },
    { url = "https://files.pythonhosted.org/packages/13/b7/0b289506a3f3598c2ae2bdfa0ea66969812ed200264e3f61df77753eee6d/bcrypt-4.3.0-cp313-cp313t-manylinux_2_28_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:74a8d21a09f5e025a9a23e7c0fd2c7fe8e7503e4d356c0a2c1486ba010619f09", size = 289752, upload-time = "2025-02-28T01:22:45.56Z" },
    { url = "https://files.pythonhosted.org/packages/dc/24/d0fb023788afe9e83cc118895a9f6c57e1044e7e1672f045e46733421fe6/bcrypt-4.3.0-cp313-cp313t-manylinux_2_28_x86_64.whl", hash = "sha256:0142b2cb84a009f8452c8c5a33ace5e3dfec4159e7735f5afe9a4d50a8ea722d", size = 277762, upload-time = "2025-02-28T01:22:47.023Z" },
    { url = "https://files.pythonhosted.org/packages/e4/38/cde58089492e55ac4ef6c49fea7027600c84fd23f7520c62118c03b4625e/bcrypt-4.3.0-cp313-cp313t-manylinux_2_34_aarch64.whl", hash = "sha256:12fa6ce40cde3f0b899729dbd7d5e8811cb892d31b6f7d0334a1f37748b789fd", size = 272384, upload-time = "2025-02-28T01:22:49.221Z" },
    { url = "https://files.pythonhosted.org/packages/de/6a/d5026520843490cfc8135d03012a413e4532a400e471e6188b01b2de853f/bcrypt-4.3.0-cp313-cp313t-manylinux_2_34_x86_64.whl", hash = "sha256:5bd3cca1f2aa5dbcf39e2aa13dd094ea181f48959e1071265de49cc2b82525af", size = 277329, upload-time = "2025-02-28T01:22:51.603Z" },
    { url = "https://files.pythonhosted.org/packages/b3/a3/4fc5255e60486466c389e28c12579d2829b28a527360e9430b4041df4cf9/bcrypt-4.3.0-cp313-cp313t-musllinux_1_1_aarch64.whl", hash = "sha256:335a420cfd63fc5bc27308e929bee231c15c85cc4c496610ffb17923abf7f231", size = 305241, upload-time = "2025-02-28T01:22:53.283Z" },
    { url = "https://files.pythonhosted.org/packages/c7/15/2b37bc07d6ce27cc94e5b10fd5058900eb8fb11642300e932c8c82e25c4a/bcrypt-4.3.0-cp313-cp313t-musllinux_1_1_x86_64.whl", hash = "sha256:0e30e5e67aed0187a1764911af023043b4542e70a7461ad20e837e94d23e1d6c", size = 309617, upload-time = "2025-02-28T01:22:55.461Z" },
    { url = "https://files.pythonhosted.org/packages/5f/1f/99f65edb09e6c935232ba0430c8c13bb98cb3194b6d636e61d93fe60ac59/bcrypt-4.3.0-cp313-cp313t-musllinux_1_2_aarch64.whl", hash = "sha256:3b8d62290ebefd49ee0b3ce7500f5dbdcf13b81402c05f6dafab9a1e1b27212f", size = 335751, upload-time = "2025-02-28T01:22:57.81Z" },
    { url = "https://files.pythonhosted.org/packages/00/1b/b324030c706711c99769988fcb694b3cb23f247ad39a7823a78e361bdbb8/bcrypt-4.3.0-cp313-cp313t-musllinux_1_2_x86_64.whl", hash = "sha256:2ef6630e0ec01376f59a006dc72918b1bf436c3b571b80fa1968d775fa02fe7d", size = 355965, upload-time = "2025-02-28T01:22:59.181Z" },
    { url = "https://files.pythonhosted.org/packages/aa/dd/20372a0579dd915dfc3b1cd4943b3bca431866fcb1dfdfd7518c3caddea6/bcrypt-4.3.0-cp313-cp313t-win32.whl", hash = "sha256:7a4be4cbf241afee43f1c3969b9103a41b40bcb3a3f467ab19f891d9bc4642e4", size = 155316, upload-time = "2025-02-28T01:23:00.763Z" },
    { url = "https://files.pythonhosted.org/packages/6d/52/45d969fcff6b5577c2bf17098dc36269b4c02197d551371c023130c0f890/bcrypt-4.3.0-cp313-cp313t-win_amd64.whl", hash = "sha256:5c1949bf259a388863ced887c7861da1df681cb2388645766c89fdfd9004c669", size = 147752, upload-time = "2025-02-28T01:23:02.908Z" },
    { url = "https://files.pythonhosted.org/packages/11/22/5ada0b9af72b60cbc4c9a399fdde4af0feaa609d27eb0adc61607997a3fa/bcrypt-4.3.0-cp38-abi3-macosx_10_12_universal2.whl", hash = "sha256:f81b0ed2639568bf14749112298f9e4e2b28853dab50a8b357e31798686a036d", size = 498019, upload-time = "2025-02-28T01:23:05.838Z" },
    { url = "https://files.pythonhosted.org/packages/b8/8c/252a1edc598dc1ce57905be173328eda073083826955ee3c97c7ff5ba584/bcrypt-4.3.0-cp38-abi3-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:864f8f19adbe13b7de11ba15d85d4a428c7e2f344bac110f667676a0ff84924b", size = 279174, upload-time = "2025-02-28T01:23:07.274Z" },
    { url = "https://files.pythonhosted.org/packages/29/5b/4547d5c49b85f0337c13929f2ccbe08b7283069eea3550a457914fc078aa/bcrypt-4.3.0-cp38-abi3-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:3e36506d001e93bffe59754397572f21bb5dc7c83f54454c990c74a468cd589e", size = 283870, upload-time = "2025-02-28T01:23:09.151Z" },
    { url = "https://files.pythonhosted.org/packages/be/21/7dbaf3fa1745cb63f776bb046e481fbababd7d344c5324eab47f5ca92dd2/bcrypt-4.3.0-cp38-abi3-manylinux_2_28_aarch64.whl", hash = "sha256:842d08d75d9fe9fb94b18b071090220697f9f184d4547179b60734846461ed59", size = 279601, upload-time = "2025-02-28T01:23:11.461Z" },
    { url = "https://files.pythonhosted.org/packages/6d/64/e042fc8262e971347d9230d9abbe70d68b0a549acd8611c83cebd3eaec67/bcrypt-4.3.0-cp38-abi3-manylinux_2_28_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:7c03296b85cb87db865d91da79bf63d5609284fc0cab9472fdd8367bbd830753", size = 297660, upload-time = "2025-02-28T01:23:12.989Z" },
    { url = "https://files.pythonhosted.org/packages/50/b8/6294eb84a3fef3b67c69b4470fcdd5326676806bf2519cda79331ab3c3a9/bcrypt-4.3.0-cp38-abi3-manylinux_2_28_x86_64.whl", hash = "sha256:62f26585e8b219cdc909b6a0069efc5e4267e25d4a3770a364ac58024f62a761", size = 284083, upload-time = "2025-02-28T01:23:14.5Z" },
    { url = "https://files.pythonhosted.org/packages/62/e6/baff635a4f2c42e8788fe1b1633911c38551ecca9a749d1052d296329da6/bcrypt-4.3.0-cp38-abi3-manylinux_2_34_aarch64.whl", hash = "sha256:beeefe437218a65322fbd0069eb437e7c98137e08f22c4660ac2dc795c31f8bb", size = 279237, upload-time = "2025-02-28T01:23:16.686Z" },
    { url = "https://files.pythonhosted.org/packages/39/48/46f623f1b0c7dc2e5de0b8af5e6f5ac4cc26408ac33f3d424e5ad8da4a90/bcrypt-4.3.0-cp38-abi3-manylinux_2_34_x86_64.whl", hash = "sha256:97eea7408db3a5bcce4a55d13245ab3fa566e23b4c67cd227062bb49e26c585d", size = 283737, upload-time = "2025-02-28T01:23:18.897Z" },
    { url = "https://files.pythonhosted.org/packages/49/8b/70671c3ce9c0fca4a6cc3cc6ccbaa7e948875a2e62cbd146e04a4011899c/bcrypt-4.3.0-cp38-abi3-musllinux_1_1_aarch64.whl", hash = "sha256:191354ebfe305e84f344c5964c7cd5f924a3bfc5d405c75ad07f232b6dffb49f", size = 312741, upload-time = "2025-02-28T01:23:21.041Z" },
    { url = "https://files.pythonhosted.org/packages/27/fb/910d3a1caa2d249b6040a5caf9f9866c52114d51523ac2fb47578a27faee/bcrypt-4.3.0-cp38-abi3-musllinux_1_1_x86_64.whl", hash = "sha256:41261d64150858eeb5ff43c753c4b216991e0ae16614a308a15d909503617732", size = 316472, upload-time = "2025-02-28T01:23:23.183Z" },
    { url = "https://files.pythonhosted.org/packages/dc/cf/7cf3a05b66ce466cfb575dbbda39718d45a609daa78500f57fa9f36fa3c0/bcrypt-4.3.0-cp38-abi3-musllinux_1_2_aarch64.whl", hash = "sha256:33752b1ba962ee793fa2b6321404bf20011fe45b9afd2a842139de3011898fef", size = 343606, upload-time = "2025-02-28T01:23:25.361Z" },
    { url = "https://files.pythonhosted.org/packages/e3/b8/e970ecc6d7e355c0d892b7f733480f4aa8509f99b33e71550242cf0b7e63/bcrypt-4.3.0-cp38-abi3-musllinux_1_2_x86_64.whl", hash = "sha256:50e6e80a4bfd23a25f5c05b90167c19030cf9f87930f7cb2eacb99f45d1c3304", size = 362867, upload-time = "2025-02-28T01:23:26.875Z" },
    { url = "https://files.pythonhosted.org/packages/a9/97/8d3118efd8354c555a3422d544163f40d9f236be5b96c714086463f11699/bcrypt-4.3.0-cp38-abi3-win32.whl", hash = "sha256:67a561c4d9fb9465ec866177e7aebcad08fe23aaf6fbd692a6fab69088abfc51", size = 160589, upload-time = "2025-02-28T01:23:28.381Z" },
    { url = "https://files.pythonhosted.org/packages/29/07/416f0b99f7f3997c69815365babbc2e8754181a4b1899d921b3c7d5b6f12/bcrypt-4.3.0-cp38-abi3-win_amd64.whl", hash = "sha256:584027857bc2843772114717a7490a37f68da563b3620f78a849bcb54dc11e62", size = 152794, upload-time = "2025-02-28T01:23:30.187Z" },
    { url = "https://files.pythonhosted.org/packages/6e/c1/3fa0e9e4e0bfd3fd77eb8b52ec198fd6e1fd7e9402052e43f23483f956dd/bcrypt-4.3.0-cp39-abi3-macosx_10_12_universal2.whl", hash = "sha256:0d3efb1157edebfd9128e4e46e2ac1a64e0c1fe46fb023158a407c7892b0f8c3", size = 498969, upload-time = "2025-02-28T01:23:31.945Z" },
    { url = "https://files.pythonhosted.org/packages/ce/d4/755ce19b6743394787fbd7dff6bf271b27ee9b5912a97242e3caf125885b/bcrypt-4.3.0-cp39-abi3-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:08bacc884fd302b611226c01014eca277d48f0a05187666bca23aac0dad6fe24", size = 279158, upload-time = "2025-02-28T01:23:34.161Z" },
    { url = "https://files.pythonhosted.org/packages/9b/5d/805ef1a749c965c46b28285dfb5cd272a7ed9fa971f970435a5133250182/bcrypt-4.3.0-cp39-abi3-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:f6746e6fec103fcd509b96bacdfdaa2fbde9a553245dbada284435173a6f1aef", size = 284285, upload-time = "2025-02-28T01:23:35.765Z" },
    { url = "https://files.pythonhosted.org/packages/ab/2b/698580547a4a4988e415721b71eb45e80c879f0fb04a62da131f45987b96/bcrypt-4.3.0-cp39-abi3-manylinux_2_28_aarch64.whl", hash = "sha256:afe327968aaf13fc143a56a3360cb27d4ad0345e34da12c7290f1b00b8fe9a8b", size = 279583, upload-time = "2025-02-28T01:23:38.021Z" },
    { url = "https://files.pythonhosted.org/packages/f2/87/62e1e426418204db520f955ffd06f1efd389feca893dad7095bf35612eec/bcrypt-4.3.0-cp39-abi3-manylinux_2_28_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:d9af79d322e735b1fc33404b5765108ae0ff232d4b54666d46730f8ac1a43676", size = 297896, upload-time = "2025-02-28T01:23:39.575Z" },
    { url = "https://files.pythonhosted.org/packages/cb/c6/8fedca4c2ada1b6e889c52d2943b2f968d3427e5d65f595620ec4c06fa2f/bcrypt-4.3.0-cp39-abi3-manylinux_2_28_x86_64.whl", hash = "sha256:f1e3ffa1365e8702dc48c8b360fef8d7afeca482809c5e45e653af82ccd088c1", size = 284492, upload-time = "2025-02-28T01:23:40.901Z" },
    { url = "https://files.pythonhosted.org/packages/4d/4d/c43332dcaaddb7710a8ff5269fcccba97ed3c85987ddaa808db084267b9a/bcrypt-4.3.0-cp39-abi3-manylinux_2_34_aarch64.whl", hash = "sha256:3004df1b323d10021fda07a813fd33e0fd57bef0e9a480bb143877f6cba996fe", size = 279213, upload-time = "2025-02-28T01:23:42.653Z" },
    { url = "https://files.pythonhosted.org/packages/dc/7f/1e36379e169a7df3a14a1c160a49b7b918600a6008de43ff20d479e6f4b5/bcrypt-4.3.0-cp39-abi3-manylinux_2_34_x86_64.whl", hash = "sha256:531457e5c839d8caea9b589a1bcfe3756b0547d7814e9ce3d437f17da75c32b0", size = 284162, upload-time = "2025-02-28T01:23:43.964Z" },
    { url = "https://files.pythonhosted.org/packages/1c/0a/644b2731194b0d7646f3210dc4d80c7fee3ecb3a1f791a6e0ae6bb8684e3/bcrypt-4.3.0-cp39-abi3-musllinux_1_1_aarch64.whl", hash = "sha256:17a854d9a7a476a89dcef6c8bd119ad23e0f82557afbd2c442777a16408e614f", size = 312856, upload-time = "2025-02-28T01:23:46.011Z" },
    { url = "https://files.pythonhosted.org/packages/dc/62/2a871837c0bb6ab0c9a88bf54de0fc021a6a08832d4ea313ed92a669d437/bcrypt-4.3.0-cp39-abi3-musllinux_1_1_x86_64.whl", hash = "sha256:6fb1fd3ab08c0cbc6826a2e0447610c6f09e983a281b919ed721ad32236b8b23", size = 316726, upload-time = "2025-02-28T01:23:47.575Z" },
    { url = "https://files.pythonhosted.org/packages/0c/a1/9898ea3faac0b156d457fd73a3cb9c2855c6fd063e44b8522925cdd8ce46/bcrypt-4.3.0-cp39-abi3-musllinux_1_2_aarch64.whl", hash = "sha256:e965a9c1e9a393b8005031ff52583cedc15b7884fce7deb8b0346388837d6cfe", size = 343664, upload-time = "2025-02-28T01:23:49.059Z" },
    { url = "https://files.pythonhosted.org/packages/40/f2/71b4ed65ce38982ecdda0ff20c3ad1b15e71949c78b2c053df53629ce940/bcrypt-4.3.0-cp39-abi3-musllinux_1_2_x86_64.whl", hash = "sha256:79e70b8342a33b52b55d93b3a59223a844962bef479f6a0ea318ebbcadf71505", size = 363128, upload-time = "2025-02-28T01:23:50.399Z" },
    { url = "https://files.pythonhosted.org/packages/11/99/12f6a58eca6dea4be992d6c681b7ec9410a1d9f5cf368c61437e31daa879/bcrypt-4.3.0-cp39-abi3-win32.whl", hash = "sha256:b4d4e57f0a63fd0b358eb765063ff661328f69a04494427265950c71b992a39a", size = 160598, upload-time = "2025-02-28T01:23:51.775Z" },
    { url = "https://files.pythonhosted.org/packages/a9/cf/45fb5261ece3e6b9817d3d82b2f343a505fd58674a92577923bc500bd1aa/bcrypt-4.3.0-cp39-abi3-win_amd64.whl", hash = "sha256:e53e074b120f2877a35cc6c736b8eb161377caae8925c17688bd46ba56daaa5b", size = 152799, upload-time = "2025-02-28T01:23:53.139Z" },
    { url = "https://files.pythonhosted.org/packages/4c/b1/1289e21d710496b88340369137cc4c5f6ee036401190ea116a7b4ae6d32a/bcrypt-4.3.0-pp311-pypy311_pp73-manylinux_2_28_aarch64.whl", hash = "sha256:a839320bf27d474e52ef8cb16449bb2ce0ba03ca9f44daba6d93fa1d8828e48a", size = 275103, upload-time = "2025-02-28T01:24:00.764Z" },
    { url = "https://files.pythonhosted.org/packages/94/41/19be9fe17e4ffc5d10b7b67f10e459fc4eee6ffe9056a88de511920cfd8d/bcrypt-4.3.0-pp311-pypy311_pp73-manylinux_2_28_x86_64.whl", hash = "sha256:bdc6a24e754a555d7316fa4774e64c6c3997d27ed2d1964d55920c7c227bc4ce", size = 280513, upload-time = "2025-02-28T01:24:02.243Z" },
    { url = "https://files.pythonhosted.org/packages/aa/73/05687a9ef89edebdd8ad7474c16d8af685eb4591c3c38300bb6aad4f0076/bcrypt-4.3.0-pp311-pypy311_pp73-manylinux_2_34_aarch64.whl", hash = "sha256:55a935b8e9a1d2def0626c4269db3fcd26728cbff1e84f0341465c31c4ee56d8", size = 274685, upload-time = "2025-02-28T01:24:04.512Z" },
    { url = "https://files.pythonhosted.org/packages/63/13/47bba97924ebe86a62ef83dc75b7c8a881d53c535f83e2c54c4bd701e05c/bcrypt-4.3.0-pp311-pypy311_pp73-manylinux_2_34_x86_64.whl", hash = "sha256:57967b7a28d855313a963aaea51bf6df89f833db4320da458e5b3c5ab6d4c938", size = 280110, upload-time = "2025-02-28T01:24:05.896Z" },
]

[[package]]
name = "beautifulsoup4"
version = "4.13.4"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "soupsieve" },
    { name = "typing-extensions" },
]
sdist = { url = "https://files.pythonhosted.org/packages/d8/e4/0c4c39e18fd76d6a628d4dd8da40543d136ce2d1752bd6eeeab0791f4d6b/beautifulsoup4-4.13.4.tar.gz", hash = "sha256:dbb3c4e1ceae6aefebdaf2423247260cd062430a410e38c66f2baa50a8437195", size = 621067, upload-time = "2025-04-15T17:05:13.836Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/50/cd/30110dc0ffcf3b131156077b90e9f60ed75711223f306da4db08eff8403b/beautifulsoup4-4.13.4-py3-none-any.whl", hash = "sha256:9bbbb14bfde9d79f38b8cd5f8c7c85f4b8f2523190ebed90e950a8dea4cb1c4b", size = 187285, upload-time = "2025-04-15T17:05:12.221Z" },
]

[[package]]
name = "black"
version = "25.1.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "click" },
    { name = "mypy-extensions" },
    { name = "packaging" },
    { name = "pathspec" },
    { name = "platformdirs" },
]
sdist = { url = "https://files.pythonhosted.org/packages/94/49/26a7b0f3f35da4b5a65f081943b7bcd22d7002f5f0fb8098ec1ff21cb6ef/black-25.1.0.tar.gz", hash = "sha256:33496d5cd1222ad73391352b4ae8da15253c5de89b93a80b3e2c8d9a19ec2666", size = 649449, upload-time = "2025-01-29T04:15:40.373Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/7e/4f/87f596aca05c3ce5b94b8663dbfe242a12843caaa82dd3f85f1ffdc3f177/black-25.1.0-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:a39337598244de4bae26475f77dda852ea00a93bd4c728e09eacd827ec929df0", size = 1614372, upload-time = "2025-01-29T05:37:11.71Z" },
    { url = "https://files.pythonhosted.org/packages/e7/d0/2c34c36190b741c59c901e56ab7f6e54dad8df05a6272a9747ecef7c6036/black-25.1.0-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:96c1c7cd856bba8e20094e36e0f948718dc688dba4a9d78c3adde52b9e6c2299", size = 1442865, upload-time = "2025-01-29T05:37:14.309Z" },
    { url = "https://files.pythonhosted.org/packages/21/d4/7518c72262468430ead45cf22bd86c883a6448b9eb43672765d69a8f1248/black-25.1.0-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:bce2e264d59c91e52d8000d507eb20a9aca4a778731a08cfff7e5ac4a4bb7096", size = 1749699, upload-time = "2025-01-29T04:18:17.688Z" },
    { url = "https://files.pythonhosted.org/packages/58/db/4f5beb989b547f79096e035c4981ceb36ac2b552d0ac5f2620e941501c99/black-25.1.0-cp311-cp311-win_amd64.whl", hash = "sha256:172b1dbff09f86ce6f4eb8edf9dede08b1fce58ba194c87d7a4f1a5aa2f5b3c2", size = 1428028, upload-time = "2025-01-29T04:18:51.711Z" },
    { url = "https://files.pythonhosted.org/packages/83/71/3fe4741df7adf015ad8dfa082dd36c94ca86bb21f25608eb247b4afb15b2/black-25.1.0-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:4b60580e829091e6f9238c848ea6750efed72140b91b048770b64e74fe04908b", size = 1650988, upload-time = "2025-01-29T05:37:16.707Z" },
    { url = "https://files.pythonhosted.org/packages/13/f3/89aac8a83d73937ccd39bbe8fc6ac8860c11cfa0af5b1c96d081facac844/black-25.1.0-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:1e2978f6df243b155ef5fa7e558a43037c3079093ed5d10fd84c43900f2d8ecc", size = 1453985, upload-time = "2025-01-29T05:37:18.273Z" },
    { url = "https://files.pythonhosted.org/packages/6f/22/b99efca33f1f3a1d2552c714b1e1b5ae92efac6c43e790ad539a163d1754/black-25.1.0-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:3b48735872ec535027d979e8dcb20bf4f70b5ac75a8ea99f127c106a7d7aba9f", size = 1783816, upload-time = "2025-01-29T04:18:33.823Z" },
    { url = "https://files.pythonhosted.org/packages/18/7e/a27c3ad3822b6f2e0e00d63d58ff6299a99a5b3aee69fa77cd4b0076b261/black-25.1.0-cp312-cp312-win_amd64.whl", hash = "sha256:ea0213189960bda9cf99be5b8c8ce66bb054af5e9e861249cd23471bd7b0b3ba", size = 1440860, upload-time = "2025-01-29T04:19:12.944Z" },
    { url = "https://files.pythonhosted.org/packages/98/87/0edf98916640efa5d0696e1abb0a8357b52e69e82322628f25bf14d263d1/black-25.1.0-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:8f0b18a02996a836cc9c9c78e5babec10930862827b1b724ddfe98ccf2f2fe4f", size = 1650673, upload-time = "2025-01-29T05:37:20.574Z" },
    { url = "https://files.pythonhosted.org/packages/52/e5/f7bf17207cf87fa6e9b676576749c6b6ed0d70f179a3d812c997870291c3/black-25.1.0-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:afebb7098bfbc70037a053b91ae8437c3857482d3a690fefc03e9ff7aa9a5fd3", size = 1453190, upload-time = "2025-01-29T05:37:22.106Z" },
    { url = "https://files.pythonhosted.org/packages/e3/ee/adda3d46d4a9120772fae6de454c8495603c37c4c3b9c60f25b1ab6401fe/black-25.1.0-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:030b9759066a4ee5e5aca28c3c77f9c64789cdd4de8ac1df642c40b708be6171", size = 1782926, upload-time = "2025-01-29T04:18:58.564Z" },
    { url = "https://files.pythonhosted.org/packages/cc/64/94eb5f45dcb997d2082f097a3944cfc7fe87e071907f677e80788a2d7b7a/black-25.1.0-cp313-cp313-win_amd64.whl", hash = "sha256:a22f402b410566e2d1c950708c77ebf5ebd5d0d88a6a2e87c86d9fb48afa0d18", size = 1442613, upload-time = "2025-01-29T04:19:27.63Z" },
    { url = "https://files.pythonhosted.org/packages/09/71/54e999902aed72baf26bca0d50781b01838251a462612966e9fc4891eadd/black-25.1.0-py3-none-any.whl", hash = "sha256:95e8176dae143ba9097f351d174fdaf0ccd29efb414b362ae3fd72bf0f710717", size = 207646, upload-time = "2025-01-29T04:15:38.082Z" },
]

[[package]]
name = "bleach"
version = "6.2.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "webencodings" },
]
sdist = { url = "https://files.pythonhosted.org/packages/76/9a/0e33f5054c54d349ea62c277191c020c2d6ef1d65ab2cb1993f91ec846d1/bleach-6.2.0.tar.gz", hash = "sha256:123e894118b8a599fd80d3ec1a6d4cc7ce4e5882b1317a7e1ba69b56e95f991f", size = 203083, upload-time = "2024-10-29T18:30:40.477Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/fc/55/96142937f66150805c25c4d0f31ee4132fd33497753400734f9dfdcbdc66/bleach-6.2.0-py3-none-any.whl", hash = "sha256:117d9c6097a7c3d22fd578fcd8d35ff1e125df6736f554da4e432fdd63f31e5e", size = 163406, upload-time = "2024-10-29T18:30:38.186Z" },
]

[package.optional-dependencies]
css = [
    { name = "tinycss2" },
]

[[package]]
name = "bracex"
version = "2.6"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/63/9a/fec38644694abfaaeca2798b58e276a8e61de49e2e37494ace423395febc/bracex-2.6.tar.gz", hash = "sha256:98f1347cd77e22ee8d967a30ad4e310b233f7754dbf31ff3fceb76145ba47dc7", size = 26642, upload-time = "2025-06-22T19:12:31.254Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/9d/2a/9186535ce58db529927f6cf5990a849aa9e052eea3e2cfefe20b9e1802da/bracex-2.6-py3-none-any.whl", hash = "sha256:0b0049264e7340b3ec782b5cb99beb325f36c3782a32e36e876452fd49a09952", size = 11508, upload-time = "2025-06-22T19:12:29.781Z" },
]

[[package]]
name = "build"
version = "1.2.2.post1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "colorama", marker = "os_name == 'nt'" },
    { name = "packaging" },
    { name = "pyproject-hooks" },
]
sdist = { url = "https://files.pythonhosted.org/packages/7d/46/aeab111f8e06793e4f0e421fcad593d547fb8313b50990f31681ee2fb1ad/build-1.2.2.post1.tar.gz", hash = "sha256:b36993e92ca9375a219c99e606a122ff365a760a2d4bba0caa09bd5278b608b7", size = 46701, upload-time = "2024-10-06T17:22:25.251Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/84/c2/80633736cd183ee4a62107413def345f7e6e3c01563dbca1417363cf957e/build-1.2.2.post1-py3-none-any.whl", hash = "sha256:1d61c0887fa860c01971625baae8bdd338e517b836a2f70dd1f7aa3a6b2fc5b5", size = 22950, upload-time = "2024-10-06T17:22:23.299Z" },
]

[[package]]
name = "cachecontrol"
version = "0.14.3"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "msgpack" },
    { name = "requests" },
]
sdist = { url = "https://files.pythonhosted.org/packages/58/3a/0cbeb04ea57d2493f3ec5a069a117ab467f85e4a10017c6d854ddcbff104/cachecontrol-0.14.3.tar.gz", hash = "sha256:73e7efec4b06b20d9267b441c1f733664f989fb8688391b670ca812d70795d11", size = 28985, upload-time = "2025-04-30T16:45:06.135Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/81/4c/800b0607b00b3fd20f1087f80ab53d6b4d005515b0f773e4831e37cfa83f/cachecontrol-0.14.3-py3-none-any.whl", hash = "sha256:b35e44a3113f17d2a31c1e6b27b9de6d4405f84ae51baa8c1d3cc5b633010cae", size = 21802, upload-time = "2025-04-30T16:45:03.863Z" },
]

[package.optional-dependencies]
filecache = [
    { name = "filelock" },
]

[[package]]
name = "cachetools"
version = "5.5.2"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/6c/81/3747dad6b14fa2cf53fcf10548cf5aea6913e96fab41a3c198676f8948a5/cachetools-5.5.2.tar.gz", hash = "sha256:1a661caa9175d26759571b2e19580f9d6393969e5dfca11fdb1f947a23e640d4", size = 28380, upload-time = "2025-02-20T21:01:19.524Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/72/76/20fa66124dbe6be5cafeb312ece67de6b61dd91a0247d1ea13db4ebb33c2/cachetools-5.5.2-py3-none-any.whl", hash = "sha256:d26a22bcc62eb95c3beabd9f1ee5e820d3d2704fe2967cbe350e20c8ffcd3f0a", size = 10080, upload-time = "2025-02-20T21:01:16.647Z" },
]

[[package]]
name = "cairocffi"
version = "1.7.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "cffi" },
]
sdist = { url = "https://files.pythonhosted.org/packages/70/c5/1a4dc131459e68a173cbdab5fad6b524f53f9c1ef7861b7698e998b837cc/cairocffi-1.7.1.tar.gz", hash = "sha256:2e48ee864884ec4a3a34bfa8c9ab9999f688286eb714a15a43ec9d068c36557b", size = 88096, upload-time = "2024-06-18T10:56:06.741Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/93/d8/ba13451aa6b745c49536e87b6bf8f629b950e84bd0e8308f7dc6883b67e2/cairocffi-1.7.1-py3-none-any.whl", hash = "sha256:9803a0e11f6c962f3b0ae2ec8ba6ae45e957a146a004697a1ac1bbf16b073b3f", size = 75611, upload-time = "2024-06-18T10:55:59.489Z" },
]

[[package]]
name = "cairosvg"
version = "2.8.2"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "cairocffi" },
    { name = "cssselect2" },
    { name = "defusedxml" },
    { name = "pillow" },
    { name = "tinycss2" },
]
sdist = { url = "https://files.pythonhosted.org/packages/ab/b9/5106168bd43d7cd8b7cc2a2ee465b385f14b63f4c092bb89eee2d48c8e67/cairosvg-2.8.2.tar.gz", hash = "sha256:07cbf4e86317b27a92318a4cac2a4bb37a5e9c1b8a27355d06874b22f85bef9f", size = 8398590, upload-time = "2025-05-15T06:56:32.653Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/67/48/816bd4aaae93dbf9e408c58598bc32f4a8c65f4b86ab560864cb3ee60adb/cairosvg-2.8.2-py3-none-any.whl", hash = "sha256:eab46dad4674f33267a671dce39b64be245911c901c70d65d2b7b0821e852bf5", size = 45773, upload-time = "2025-05-15T06:56:28.552Z" },
]

[[package]]
name = "certifi"
version = "2025.7.9"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/de/8a/c729b6b60c66a38f590c4e774decc4b2ec7b0576be8f1aa984a53ffa812a/certifi-2025.7.9.tar.gz", hash = "sha256:c1d2ec05395148ee10cf672ffc28cd37ea0ab0d99f9cc74c43e588cbd111b079", size = 160386, upload-time = "2025-07-09T02:13:58.874Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/66/f3/80a3f974c8b535d394ff960a11ac20368e06b736da395b551a49ce950cce/certifi-2025.7.9-py3-none-any.whl", hash = "sha256:d842783a14f8fdd646895ac26f719a061408834473cfc10203f6a575beb15d39", size = 159230, upload-time = "2025-07-09T02:13:57.007Z" },
]

[[package]]
name = "cffi"
version = "1.17.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "pycparser" },
]
sdist = { url = "https://files.pythonhosted.org/packages/fc/97/c783634659c2920c3fc70419e3af40972dbaf758daa229a7d6ea6135c90d/cffi-1.17.1.tar.gz", hash = "sha256:1c39c6016c32bc48dd54561950ebd6836e1670f2ae46128f67cf49e789c52824", size = 516621, upload-time = "2024-09-04T20:45:21.852Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/6b/f4/927e3a8899e52a27fa57a48607ff7dc91a9ebe97399b357b85a0c7892e00/cffi-1.17.1-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:a45e3c6913c5b87b3ff120dcdc03f6131fa0065027d0ed7ee6190736a74cd401", size = 182264, upload-time = "2024-09-04T20:43:51.124Z" },
    { url = "https://files.pythonhosted.org/packages/6c/f5/6c3a8efe5f503175aaddcbea6ad0d2c96dad6f5abb205750d1b3df44ef29/cffi-1.17.1-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:30c5e0cb5ae493c04c8b42916e52ca38079f1b235c2f8ae5f4527b963c401caf", size = 178651, upload-time = "2024-09-04T20:43:52.872Z" },
    { url = "https://files.pythonhosted.org/packages/94/dd/a3f0118e688d1b1a57553da23b16bdade96d2f9bcda4d32e7d2838047ff7/cffi-1.17.1-cp311-cp311-manylinux_2_12_i686.manylinux2010_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:f75c7ab1f9e4aca5414ed4d8e5c0e303a34f4421f8a0d47a4d019ceff0ab6af4", size = 445259, upload-time = "2024-09-04T20:43:56.123Z" },
    { url = "https://files.pythonhosted.org/packages/2e/ea/70ce63780f096e16ce8588efe039d3c4f91deb1dc01e9c73a287939c79a6/cffi-1.17.1-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:a1ed2dd2972641495a3ec98445e09766f077aee98a1c896dcb4ad0d303628e41", size = 469200, upload-time = "2024-09-04T20:43:57.891Z" },
    { url = "https://files.pythonhosted.org/packages/1c/a0/a4fa9f4f781bda074c3ddd57a572b060fa0df7655d2a4247bbe277200146/cffi-1.17.1-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:46bf43160c1a35f7ec506d254e5c890f3c03648a4dbac12d624e4490a7046cd1", size = 477235, upload-time = "2024-09-04T20:44:00.18Z" },
    { url = "https://files.pythonhosted.org/packages/62/12/ce8710b5b8affbcdd5c6e367217c242524ad17a02fe5beec3ee339f69f85/cffi-1.17.1-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:a24ed04c8ffd54b0729c07cee15a81d964e6fee0e3d4d342a27b020d22959dc6", size = 459721, upload-time = "2024-09-04T20:44:01.585Z" },
    { url = "https://files.pythonhosted.org/packages/ff/6b/d45873c5e0242196f042d555526f92aa9e0c32355a1be1ff8c27f077fd37/cffi-1.17.1-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:610faea79c43e44c71e1ec53a554553fa22321b65fae24889706c0a84d4ad86d", size = 467242, upload-time = "2024-09-04T20:44:03.467Z" },
    { url = "https://files.pythonhosted.org/packages/1a/52/d9a0e523a572fbccf2955f5abe883cfa8bcc570d7faeee06336fbd50c9fc/cffi-1.17.1-cp311-cp311-musllinux_1_1_aarch64.whl", hash = "sha256:a9b15d491f3ad5d692e11f6b71f7857e7835eb677955c00cc0aefcd0669adaf6", size = 477999, upload-time = "2024-09-04T20:44:05.023Z" },
    { url = "https://files.pythonhosted.org/packages/44/74/f2a2460684a1a2d00ca799ad880d54652841a780c4c97b87754f660c7603/cffi-1.17.1-cp311-cp311-musllinux_1_1_i686.whl", hash = "sha256:de2ea4b5833625383e464549fec1bc395c1bdeeb5f25c4a3a82b5a8c756ec22f", size = 454242, upload-time = "2024-09-04T20:44:06.444Z" },
    { url = "https://files.pythonhosted.org/packages/f8/4a/34599cac7dfcd888ff54e801afe06a19c17787dfd94495ab0c8d35fe99fb/cffi-1.17.1-cp311-cp311-musllinux_1_1_x86_64.whl", hash = "sha256:fc48c783f9c87e60831201f2cce7f3b2e4846bf4d8728eabe54d60700b318a0b", size = 478604, upload-time = "2024-09-04T20:44:08.206Z" },
    { url = "https://files.pythonhosted.org/packages/34/33/e1b8a1ba29025adbdcda5fb3a36f94c03d771c1b7b12f726ff7fef2ebe36/cffi-1.17.1-cp311-cp311-win32.whl", hash = "sha256:85a950a4ac9c359340d5963966e3e0a94a676bd6245a4b55bc43949eee26a655", size = 171727, upload-time = "2024-09-04T20:44:09.481Z" },
    { url = "https://files.pythonhosted.org/packages/3d/97/50228be003bb2802627d28ec0627837ac0bf35c90cf769812056f235b2d1/cffi-1.17.1-cp311-cp311-win_amd64.whl", hash = "sha256:caaf0640ef5f5517f49bc275eca1406b0ffa6aa184892812030f04c2abf589a0", size = 181400, upload-time = "2024-09-04T20:44:10.873Z" },
    { url = "https://files.pythonhosted.org/packages/5a/84/e94227139ee5fb4d600a7a4927f322e1d4aea6fdc50bd3fca8493caba23f/cffi-1.17.1-cp312-cp312-macosx_10_9_x86_64.whl", hash = "sha256:805b4371bf7197c329fcb3ead37e710d1bca9da5d583f5073b799d5c5bd1eee4", size = 183178, upload-time = "2024-09-04T20:44:12.232Z" },
    { url = "https://files.pythonhosted.org/packages/da/ee/fb72c2b48656111c4ef27f0f91da355e130a923473bf5ee75c5643d00cca/cffi-1.17.1-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:733e99bc2df47476e3848417c5a4540522f234dfd4ef3ab7fafdf555b082ec0c", size = 178840, upload-time = "2024-09-04T20:44:13.739Z" },
    { url = "https://files.pythonhosted.org/packages/cc/b6/db007700f67d151abadf508cbfd6a1884f57eab90b1bb985c4c8c02b0f28/cffi-1.17.1-cp312-cp312-manylinux_2_12_i686.manylinux2010_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:1257bdabf294dceb59f5e70c64a3e2f462c30c7ad68092d01bbbfb1c16b1ba36", size = 454803, upload-time = "2024-09-04T20:44:15.231Z" },
    { url = "https://files.pythonhosted.org/packages/1a/df/f8d151540d8c200eb1c6fba8cd0dfd40904f1b0682ea705c36e6c2e97ab3/cffi-1.17.1-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:da95af8214998d77a98cc14e3a3bd00aa191526343078b530ceb0bd710fb48a5", size = 478850, upload-time = "2024-09-04T20:44:17.188Z" },
    { url = "https://files.pythonhosted.org/packages/28/c0/b31116332a547fd2677ae5b78a2ef662dfc8023d67f41b2a83f7c2aa78b1/cffi-1.17.1-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:d63afe322132c194cf832bfec0dc69a99fb9bb6bbd550f161a49e9e855cc78ff", size = 485729, upload-time = "2024-09-04T20:44:18.688Z" },
    { url = "https://files.pythonhosted.org/packages/91/2b/9a1ddfa5c7f13cab007a2c9cc295b70fbbda7cb10a286aa6810338e60ea1/cffi-1.17.1-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:f79fc4fc25f1c8698ff97788206bb3c2598949bfe0fef03d299eb1b5356ada99", size = 471256, upload-time = "2024-09-04T20:44:20.248Z" },
    { url = "https://files.pythonhosted.org/packages/b2/d5/da47df7004cb17e4955df6a43d14b3b4ae77737dff8bf7f8f333196717bf/cffi-1.17.1-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:b62ce867176a75d03a665bad002af8e6d54644fad99a3c70905c543130e39d93", size = 479424, upload-time = "2024-09-04T20:44:21.673Z" },
    { url = "https://files.pythonhosted.org/packages/0b/ac/2a28bcf513e93a219c8a4e8e125534f4f6db03e3179ba1c45e949b76212c/cffi-1.17.1-cp312-cp312-musllinux_1_1_aarch64.whl", hash = "sha256:386c8bf53c502fff58903061338ce4f4950cbdcb23e2902d86c0f722b786bbe3", size = 484568, upload-time = "2024-09-04T20:44:23.245Z" },
    { url = "https://files.pythonhosted.org/packages/d4/38/ca8a4f639065f14ae0f1d9751e70447a261f1a30fa7547a828ae08142465/cffi-1.17.1-cp312-cp312-musllinux_1_1_x86_64.whl", hash = "sha256:4ceb10419a9adf4460ea14cfd6bc43d08701f0835e979bf821052f1805850fe8", size = 488736, upload-time = "2024-09-04T20:44:24.757Z" },
    { url = "https://files.pythonhosted.org/packages/86/c5/28b2d6f799ec0bdecf44dced2ec5ed43e0eb63097b0f58c293583b406582/cffi-1.17.1-cp312-cp312-win32.whl", hash = "sha256:a08d7e755f8ed21095a310a693525137cfe756ce62d066e53f502a83dc550f65", size = 172448, upload-time = "2024-09-04T20:44:26.208Z" },
    { url = "https://files.pythonhosted.org/packages/50/b9/db34c4755a7bd1cb2d1603ac3863f22bcecbd1ba29e5ee841a4bc510b294/cffi-1.17.1-cp312-cp312-win_amd64.whl", hash = "sha256:51392eae71afec0d0c8fb1a53b204dbb3bcabcb3c9b807eedf3e1e6ccf2de903", size = 181976, upload-time = "2024-09-04T20:44:27.578Z" },
    { url = "https://files.pythonhosted.org/packages/8d/f8/dd6c246b148639254dad4d6803eb6a54e8c85c6e11ec9df2cffa87571dbe/cffi-1.17.1-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:f3a2b4222ce6b60e2e8b337bb9596923045681d71e5a082783484d845390938e", size = 182989, upload-time = "2024-09-04T20:44:28.956Z" },
    { url = "https://files.pythonhosted.org/packages/8b/f1/672d303ddf17c24fc83afd712316fda78dc6fce1cd53011b839483e1ecc8/cffi-1.17.1-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:0984a4925a435b1da406122d4d7968dd861c1385afe3b45ba82b750f229811e2", size = 178802, upload-time = "2024-09-04T20:44:30.289Z" },
    { url = "https://files.pythonhosted.org/packages/0e/2d/eab2e858a91fdff70533cab61dcff4a1f55ec60425832ddfdc9cd36bc8af/cffi-1.17.1-cp313-cp313-manylinux_2_12_i686.manylinux2010_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:d01b12eeeb4427d3110de311e1774046ad344f5b1a7403101878976ecd7a10f3", size = 454792, upload-time = "2024-09-04T20:44:32.01Z" },
    { url = "https://files.pythonhosted.org/packages/75/b2/fbaec7c4455c604e29388d55599b99ebcc250a60050610fadde58932b7ee/cffi-1.17.1-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:706510fe141c86a69c8ddc029c7910003a17353970cff3b904ff0686a5927683", size = 478893, upload-time = "2024-09-04T20:44:33.606Z" },
    { url = "https://files.pythonhosted.org/packages/4f/b7/6e4a2162178bf1935c336d4da8a9352cccab4d3a5d7914065490f08c0690/cffi-1.17.1-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:de55b766c7aa2e2a3092c51e0483d700341182f08e67c63630d5b6f200bb28e5", size = 485810, upload-time = "2024-09-04T20:44:35.191Z" },
    { url = "https://files.pythonhosted.org/packages/c7/8a/1d0e4a9c26e54746dc08c2c6c037889124d4f59dffd853a659fa545f1b40/cffi-1.17.1-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:c59d6e989d07460165cc5ad3c61f9fd8f1b4796eacbd81cee78957842b834af4", size = 471200, upload-time = "2024-09-04T20:44:36.743Z" },
    { url = "https://files.pythonhosted.org/packages/26/9f/1aab65a6c0db35f43c4d1b4f580e8df53914310afc10ae0397d29d697af4/cffi-1.17.1-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:dd398dbc6773384a17fe0d3e7eeb8d1a21c2200473ee6806bb5e6a8e62bb73dd", size = 479447, upload-time = "2024-09-04T20:44:38.492Z" },
    { url = "https://files.pythonhosted.org/packages/5f/e4/fb8b3dd8dc0e98edf1135ff067ae070bb32ef9d509d6cb0f538cd6f7483f/cffi-1.17.1-cp313-cp313-musllinux_1_1_aarch64.whl", hash = "sha256:3edc8d958eb099c634dace3c7e16560ae474aa3803a5df240542b305d14e14ed", size = 484358, upload-time = "2024-09-04T20:44:40.046Z" },
    { url = "https://files.pythonhosted.org/packages/f1/47/d7145bf2dc04684935d57d67dff9d6d795b2ba2796806bb109864be3a151/cffi-1.17.1-cp313-cp313-musllinux_1_1_x86_64.whl", hash = "sha256:72e72408cad3d5419375fc87d289076ee319835bdfa2caad331e377589aebba9", size = 488469, upload-time = "2024-09-04T20:44:41.616Z" },
    { url = "https://files.pythonhosted.org/packages/bf/ee/f94057fa6426481d663b88637a9a10e859e492c73d0384514a17d78ee205/cffi-1.17.1-cp313-cp313-win32.whl", hash = "sha256:e03eab0a8677fa80d646b5ddece1cbeaf556c313dcfac435ba11f107ba117b5d", size = 172475, upload-time = "2024-09-04T20:44:43.733Z" },
    { url = "https://files.pythonhosted.org/packages/7c/fc/6a8cb64e5f0324877d503c854da15d76c1e50eb722e320b15345c4d0c6de/cffi-1.17.1-cp313-cp313-win_amd64.whl", hash = "sha256:f6a16c31041f09ead72d69f583767292f750d24913dadacf5756b966aacb3f1a", size = 182009, upload-time = "2024-09-04T20:44:45.309Z" },
]

[[package]]
name = "charset-normalizer"
version = "3.4.2"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/e4/33/89c2ced2b67d1c2a61c19c6751aa8902d46ce3dacb23600a283619f5a12d/charset_normalizer-3.4.2.tar.gz", hash = "sha256:5baececa9ecba31eff645232d59845c07aa030f0c81ee70184a90d35099a0e63", size = 126367, upload-time = "2025-05-02T08:34:42.01Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/05/85/4c40d00dcc6284a1c1ad5de5e0996b06f39d8232f1031cd23c2f5c07ee86/charset_normalizer-3.4.2-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:be1e352acbe3c78727a16a455126d9ff83ea2dfdcbc83148d2982305a04714c2", size = 198794, upload-time = "2025-05-02T08:32:11.945Z" },
    { url = "https://files.pythonhosted.org/packages/41/d9/7a6c0b9db952598e97e93cbdfcb91bacd89b9b88c7c983250a77c008703c/charset_normalizer-3.4.2-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:aa88ca0b1932e93f2d961bf3addbb2db902198dca337d88c89e1559e066e7645", size = 142846, upload-time = "2025-05-02T08:32:13.946Z" },
    { url = "https://files.pythonhosted.org/packages/66/82/a37989cda2ace7e37f36c1a8ed16c58cf48965a79c2142713244bf945c89/charset_normalizer-3.4.2-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:d524ba3f1581b35c03cb42beebab4a13e6cdad7b36246bd22541fa585a56cccd", size = 153350, upload-time = "2025-05-02T08:32:15.873Z" },
    { url = "https://files.pythonhosted.org/packages/df/68/a576b31b694d07b53807269d05ec3f6f1093e9545e8607121995ba7a8313/charset_normalizer-3.4.2-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:28a1005facc94196e1fb3e82a3d442a9d9110b8434fc1ded7a24a2983c9888d8", size = 145657, upload-time = "2025-05-02T08:32:17.283Z" },
    { url = "https://files.pythonhosted.org/packages/92/9b/ad67f03d74554bed3aefd56fe836e1623a50780f7c998d00ca128924a499/charset_normalizer-3.4.2-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:fdb20a30fe1175ecabed17cbf7812f7b804b8a315a25f24678bcdf120a90077f", size = 147260, upload-time = "2025-05-02T08:32:18.807Z" },
    { url = "https://files.pythonhosted.org/packages/a6/e6/8aebae25e328160b20e31a7e9929b1578bbdc7f42e66f46595a432f8539e/charset_normalizer-3.4.2-cp311-cp311-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:0f5d9ed7f254402c9e7d35d2f5972c9bbea9040e99cd2861bd77dc68263277c7", size = 149164, upload-time = "2025-05-02T08:32:20.333Z" },
    { url = "https://files.pythonhosted.org/packages/8b/f2/b3c2f07dbcc248805f10e67a0262c93308cfa149a4cd3d1fe01f593e5fd2/charset_normalizer-3.4.2-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:efd387a49825780ff861998cd959767800d54f8308936b21025326de4b5a42b9", size = 144571, upload-time = "2025-05-02T08:32:21.86Z" },
    { url = "https://files.pythonhosted.org/packages/60/5b/c3f3a94bc345bc211622ea59b4bed9ae63c00920e2e8f11824aa5708e8b7/charset_normalizer-3.4.2-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:f0aa37f3c979cf2546b73e8222bbfa3dc07a641585340179d768068e3455e544", size = 151952, upload-time = "2025-05-02T08:32:23.434Z" },
    { url = "https://files.pythonhosted.org/packages/e2/4d/ff460c8b474122334c2fa394a3f99a04cf11c646da895f81402ae54f5c42/charset_normalizer-3.4.2-cp311-cp311-musllinux_1_2_ppc64le.whl", hash = "sha256:e70e990b2137b29dc5564715de1e12701815dacc1d056308e2b17e9095372a82", size = 155959, upload-time = "2025-05-02T08:32:24.993Z" },
    { url = "https://files.pythonhosted.org/packages/a2/2b/b964c6a2fda88611a1fe3d4c400d39c66a42d6c169c924818c848f922415/charset_normalizer-3.4.2-cp311-cp311-musllinux_1_2_s390x.whl", hash = "sha256:0c8c57f84ccfc871a48a47321cfa49ae1df56cd1d965a09abe84066f6853b9c0", size = 153030, upload-time = "2025-05-02T08:32:26.435Z" },
    { url = "https://files.pythonhosted.org/packages/59/2e/d3b9811db26a5ebf444bc0fa4f4be5aa6d76fc6e1c0fd537b16c14e849b6/charset_normalizer-3.4.2-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:6b66f92b17849b85cad91259efc341dce9c1af48e2173bf38a85c6329f1033e5", size = 148015, upload-time = "2025-05-02T08:32:28.376Z" },
    { url = "https://files.pythonhosted.org/packages/90/07/c5fd7c11eafd561bb51220d600a788f1c8d77c5eef37ee49454cc5c35575/charset_normalizer-3.4.2-cp311-cp311-win32.whl", hash = "sha256:daac4765328a919a805fa5e2720f3e94767abd632ae410a9062dff5412bae65a", size = 98106, upload-time = "2025-05-02T08:32:30.281Z" },
    { url = "https://files.pythonhosted.org/packages/a8/05/5e33dbef7e2f773d672b6d79f10ec633d4a71cd96db6673625838a4fd532/charset_normalizer-3.4.2-cp311-cp311-win_amd64.whl", hash = "sha256:e53efc7c7cee4c1e70661e2e112ca46a575f90ed9ae3fef200f2a25e954f4b28", size = 105402, upload-time = "2025-05-02T08:32:32.191Z" },
    { url = "https://files.pythonhosted.org/packages/d7/a4/37f4d6035c89cac7930395a35cc0f1b872e652eaafb76a6075943754f095/charset_normalizer-3.4.2-cp312-cp312-macosx_10_13_universal2.whl", hash = "sha256:0c29de6a1a95f24b9a1aa7aefd27d2487263f00dfd55a77719b530788f75cff7", size = 199936, upload-time = "2025-05-02T08:32:33.712Z" },
    { url = "https://files.pythonhosted.org/packages/ee/8a/1a5e33b73e0d9287274f899d967907cd0bf9c343e651755d9307e0dbf2b3/charset_normalizer-3.4.2-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:cddf7bd982eaa998934a91f69d182aec997c6c468898efe6679af88283b498d3", size = 143790, upload-time = "2025-05-02T08:32:35.768Z" },
    { url = "https://files.pythonhosted.org/packages/66/52/59521f1d8e6ab1482164fa21409c5ef44da3e9f653c13ba71becdd98dec3/charset_normalizer-3.4.2-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:fcbe676a55d7445b22c10967bceaaf0ee69407fbe0ece4d032b6eb8d4565982a", size = 153924, upload-time = "2025-05-02T08:32:37.284Z" },
    { url = "https://files.pythonhosted.org/packages/86/2d/fb55fdf41964ec782febbf33cb64be480a6b8f16ded2dbe8db27a405c09f/charset_normalizer-3.4.2-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:d41c4d287cfc69060fa91cae9683eacffad989f1a10811995fa309df656ec214", size = 146626, upload-time = "2025-05-02T08:32:38.803Z" },
    { url = "https://files.pythonhosted.org/packages/8c/73/6ede2ec59bce19b3edf4209d70004253ec5f4e319f9a2e3f2f15601ed5f7/charset_normalizer-3.4.2-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:4e594135de17ab3866138f496755f302b72157d115086d100c3f19370839dd3a", size = 148567, upload-time = "2025-05-02T08:32:40.251Z" },
    { url = "https://files.pythonhosted.org/packages/09/14/957d03c6dc343c04904530b6bef4e5efae5ec7d7990a7cbb868e4595ee30/charset_normalizer-3.4.2-cp312-cp312-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:cf713fe9a71ef6fd5adf7a79670135081cd4431c2943864757f0fa3a65b1fafd", size = 150957, upload-time = "2025-05-02T08:32:41.705Z" },
    { url = "https://files.pythonhosted.org/packages/0d/c8/8174d0e5c10ccebdcb1b53cc959591c4c722a3ad92461a273e86b9f5a302/charset_normalizer-3.4.2-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:a370b3e078e418187da8c3674eddb9d983ec09445c99a3a263c2011993522981", size = 145408, upload-time = "2025-05-02T08:32:43.709Z" },
    { url = "https://files.pythonhosted.org/packages/58/aa/8904b84bc8084ac19dc52feb4f5952c6df03ffb460a887b42615ee1382e8/charset_normalizer-3.4.2-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:a955b438e62efdf7e0b7b52a64dc5c3396e2634baa62471768a64bc2adb73d5c", size = 153399, upload-time = "2025-05-02T08:32:46.197Z" },
    { url = "https://files.pythonhosted.org/packages/c2/26/89ee1f0e264d201cb65cf054aca6038c03b1a0c6b4ae998070392a3ce605/charset_normalizer-3.4.2-cp312-cp312-musllinux_1_2_ppc64le.whl", hash = "sha256:7222ffd5e4de8e57e03ce2cef95a4c43c98fcb72ad86909abdfc2c17d227fc1b", size = 156815, upload-time = "2025-05-02T08:32:48.105Z" },
    { url = "https://files.pythonhosted.org/packages/fd/07/68e95b4b345bad3dbbd3a8681737b4338ff2c9df29856a6d6d23ac4c73cb/charset_normalizer-3.4.2-cp312-cp312-musllinux_1_2_s390x.whl", hash = "sha256:bee093bf902e1d8fc0ac143c88902c3dfc8941f7ea1d6a8dd2bcb786d33db03d", size = 154537, upload-time = "2025-05-02T08:32:49.719Z" },
    { url = "https://files.pythonhosted.org/packages/77/1a/5eefc0ce04affb98af07bc05f3bac9094513c0e23b0562d64af46a06aae4/charset_normalizer-3.4.2-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:dedb8adb91d11846ee08bec4c8236c8549ac721c245678282dcb06b221aab59f", size = 149565, upload-time = "2025-05-02T08:32:51.404Z" },
    { url = "https://files.pythonhosted.org/packages/37/a0/2410e5e6032a174c95e0806b1a6585eb21e12f445ebe239fac441995226a/charset_normalizer-3.4.2-cp312-cp312-win32.whl", hash = "sha256:db4c7bf0e07fc3b7d89ac2a5880a6a8062056801b83ff56d8464b70f65482b6c", size = 98357, upload-time = "2025-05-02T08:32:53.079Z" },
    { url = "https://files.pythonhosted.org/packages/6c/4f/c02d5c493967af3eda9c771ad4d2bbc8df6f99ddbeb37ceea6e8716a32bc/charset_normalizer-3.4.2-cp312-cp312-win_amd64.whl", hash = "sha256:5a9979887252a82fefd3d3ed2a8e3b937a7a809f65dcb1e068b090e165bbe99e", size = 105776, upload-time = "2025-05-02T08:32:54.573Z" },
    { url = "https://files.pythonhosted.org/packages/ea/12/a93df3366ed32db1d907d7593a94f1fe6293903e3e92967bebd6950ed12c/charset_normalizer-3.4.2-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:926ca93accd5d36ccdabd803392ddc3e03e6d4cd1cf17deff3b989ab8e9dbcf0", size = 199622, upload-time = "2025-05-02T08:32:56.363Z" },
    { url = "https://files.pythonhosted.org/packages/04/93/bf204e6f344c39d9937d3c13c8cd5bbfc266472e51fc8c07cb7f64fcd2de/charset_normalizer-3.4.2-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:eba9904b0f38a143592d9fc0e19e2df0fa2e41c3c3745554761c5f6447eedabf", size = 143435, upload-time = "2025-05-02T08:32:58.551Z" },
    { url = "https://files.pythonhosted.org/packages/22/2a/ea8a2095b0bafa6c5b5a55ffdc2f924455233ee7b91c69b7edfcc9e02284/charset_normalizer-3.4.2-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:3fddb7e2c84ac87ac3a947cb4e66d143ca5863ef48e4a5ecb83bd48619e4634e", size = 153653, upload-time = "2025-05-02T08:33:00.342Z" },
    { url = "https://files.pythonhosted.org/packages/b6/57/1b090ff183d13cef485dfbe272e2fe57622a76694061353c59da52c9a659/charset_normalizer-3.4.2-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:98f862da73774290f251b9df8d11161b6cf25b599a66baf087c1ffe340e9bfd1", size = 146231, upload-time = "2025-05-02T08:33:02.081Z" },
    { url = "https://files.pythonhosted.org/packages/e2/28/ffc026b26f441fc67bd21ab7f03b313ab3fe46714a14b516f931abe1a2d8/charset_normalizer-3.4.2-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:6c9379d65defcab82d07b2a9dfbfc2e95bc8fe0ebb1b176a3190230a3ef0e07c", size = 148243, upload-time = "2025-05-02T08:33:04.063Z" },
    { url = "https://files.pythonhosted.org/packages/c0/0f/9abe9bd191629c33e69e47c6ef45ef99773320e9ad8e9cb08b8ab4a8d4cb/charset_normalizer-3.4.2-cp313-cp313-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:e635b87f01ebc977342e2697d05b56632f5f879a4f15955dfe8cef2448b51691", size = 150442, upload-time = "2025-05-02T08:33:06.418Z" },
    { url = "https://files.pythonhosted.org/packages/67/7c/a123bbcedca91d5916c056407f89a7f5e8fdfce12ba825d7d6b9954a1a3c/charset_normalizer-3.4.2-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:1c95a1e2902a8b722868587c0e1184ad5c55631de5afc0eb96bc4b0d738092c0", size = 145147, upload-time = "2025-05-02T08:33:08.183Z" },
    { url = "https://files.pythonhosted.org/packages/ec/fe/1ac556fa4899d967b83e9893788e86b6af4d83e4726511eaaad035e36595/charset_normalizer-3.4.2-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:ef8de666d6179b009dce7bcb2ad4c4a779f113f12caf8dc77f0162c29d20490b", size = 153057, upload-time = "2025-05-02T08:33:09.986Z" },
    { url = "https://files.pythonhosted.org/packages/2b/ff/acfc0b0a70b19e3e54febdd5301a98b72fa07635e56f24f60502e954c461/charset_normalizer-3.4.2-cp313-cp313-musllinux_1_2_ppc64le.whl", hash = "sha256:32fc0341d72e0f73f80acb0a2c94216bd704f4f0bce10aedea38f30502b271ff", size = 156454, upload-time = "2025-05-02T08:33:11.814Z" },
    { url = "https://files.pythonhosted.org/packages/92/08/95b458ce9c740d0645feb0e96cea1f5ec946ea9c580a94adfe0b617f3573/charset_normalizer-3.4.2-cp313-cp313-musllinux_1_2_s390x.whl", hash = "sha256:289200a18fa698949d2b39c671c2cc7a24d44096784e76614899a7ccf2574b7b", size = 154174, upload-time = "2025-05-02T08:33:13.707Z" },
    { url = "https://files.pythonhosted.org/packages/78/be/8392efc43487ac051eee6c36d5fbd63032d78f7728cb37aebcc98191f1ff/charset_normalizer-3.4.2-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:4a476b06fbcf359ad25d34a057b7219281286ae2477cc5ff5e3f70a246971148", size = 149166, upload-time = "2025-05-02T08:33:15.458Z" },
    { url = "https://files.pythonhosted.org/packages/44/96/392abd49b094d30b91d9fbda6a69519e95802250b777841cf3bda8fe136c/charset_normalizer-3.4.2-cp313-cp313-win32.whl", hash = "sha256:aaeeb6a479c7667fbe1099af9617c83aaca22182d6cf8c53966491a0f1b7ffb7", size = 98064, upload-time = "2025-05-02T08:33:17.06Z" },
    { url = "https://files.pythonhosted.org/packages/e9/b0/0200da600134e001d91851ddc797809e2fe0ea72de90e09bec5a2fbdaccb/charset_normalizer-3.4.2-cp313-cp313-win_amd64.whl", hash = "sha256:aa6af9e7d59f9c12b33ae4e9450619cf2488e2bbe9b44030905877f0b2324980", size = 105641, upload-time = "2025-05-02T08:33:18.753Z" },
    { url = "https://files.pythonhosted.org/packages/20/94/c5790835a017658cbfabd07f3bfb549140c3ac458cfc196323996b10095a/charset_normalizer-3.4.2-py3-none-any.whl", hash = "sha256:7f56930ab0abd1c45cd15be65cc741c28b1c9a34876ce8c17a2fa107810c0af0", size = 52626, upload-time = "2025-05-02T08:34:40.053Z" },
]

[[package]]
name = "chromadb"
version = "1.0.15"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "bcrypt" },
    { name = "build" },
    { name = "grpcio" },
    { name = "httpx" },
    { name = "importlib-resources" },
    { name = "jsonschema" },
    { name = "kubernetes" },
    { name = "mmh3" },
    { name = "numpy" },
    { name = "onnxruntime" },
    { name = "opentelemetry-api" },
    { name = "opentelemetry-exporter-otlp-proto-grpc" },
    { name = "opentelemetry-sdk" },
    { name = "orjson" },
    { name = "overrides" },
    { name = "posthog" },
    { name = "pybase64" },
    { name = "pydantic" },
    { name = "pypika" },
    { name = "pyyaml" },
    { name = "rich" },
    { name = "tenacity" },
    { name = "tokenizers" },
    { name = "tqdm" },
    { name = "typer" },
    { name = "typing-extensions" },
    { name = "uvicorn", extra = ["standard"] },
]
sdist = { url = "https://files.pythonhosted.org/packages/ad/e2/0653b2e539db5512d2200c759f1bc7f9ef5609fe47f3c7d24b82f62dc00f/chromadb-1.0.15.tar.gz", hash = "sha256:3e910da3f5414e2204f89c7beca1650847f2bf3bd71f11a2e40aad1eb31050aa", size = 1218840, upload-time = "2025-07-02T17:07:09.875Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/85/5a/866c6f0c2160cbc8dca0cf77b2fb391dcf435b32a58743da1bc1a08dc442/chromadb-1.0.15-cp39-abi3-macosx_10_12_x86_64.whl", hash = "sha256:51791553014297798b53df4e043e9c30f4e8bd157647971a6bb02b04bfa65f82", size = 18838820, upload-time = "2025-07-02T17:07:07.632Z" },
    { url = "https://files.pythonhosted.org/packages/e1/18/ff9b58ab5d334f5ecff7fdbacd6761bac467176708fa4d2500ae7c048af0/chromadb-1.0.15-cp39-abi3-macosx_11_0_arm64.whl", hash = "sha256:48015803c0631c3a817befc276436dc084bb628c37fd4214047212afb2056291", size = 18057131, upload-time = "2025-07-02T17:07:05.15Z" },
    { url = "https://files.pythonhosted.org/packages/31/49/74e34cc5aeeb25aff2c0ede6790b3671e14c1b91574dd8f98d266a4c5aad/chromadb-1.0.15-cp39-abi3-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:3b73cd6fb32fcdd91c577cca16ea6112b691d72b441bb3f2140426d1e79e453a", size = 18595284, upload-time = "2025-07-02T17:06:59.102Z" },
    { url = "https://files.pythonhosted.org/packages/cb/33/190df917a057067e37f8b48d082d769bed8b3c0c507edefc7b6c6bb577d0/chromadb-1.0.15-cp39-abi3-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:479f1b401af9e7c20f50642ffb3376abbfd78e2b5b170429f7c79eff52e367db", size = 19526626, upload-time = "2025-07-02T17:07:02.163Z" },
    { url = "https://files.pythonhosted.org/packages/a1/30/6890da607358993f87a01e80bcce916b4d91515ce865f07dc06845cb472f/chromadb-1.0.15-cp39-abi3-win_amd64.whl", hash = "sha256:e0cb3b93fdc42b1786f151d413ef36299f30f783a30ce08bf0bfb12e552b4190", size = 19520490, upload-time = "2025-07-02T17:07:11.559Z" },
]

[[package]]
name = "click"
version = "8.2.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "colorama", marker = "sys_platform == 'win32'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/60/6c/8ca2efa64cf75a977a0d7fac081354553ebe483345c734fb6b6515d96bbc/click-8.2.1.tar.gz", hash = "sha256:27c491cc05d968d271d5a1db13e3b5a184636d9d930f148c50b038f0d0646202", size = 286342, upload-time = "2025-05-20T23:19:49.832Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/85/32/10bb5764d90a8eee674e9dc6f4db6a0ab47c8c4d0d83c27f7c39ac415a4d/click-8.2.1-py3-none-any.whl", hash = "sha256:61a3265b914e850b85317d0b3109c7f8cd35a670f963866005d6ef1d5175a12b", size = 102215, upload-time = "2025-05-20T23:19:47.796Z" },
]

[[package]]
name = "cohere"
version = "5.16.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "fastavro" },
    { name = "httpx" },
    { name = "httpx-sse" },
    { name = "pydantic" },
    { name = "pydantic-core" },
    { name = "requests" },
    { name = "tokenizers" },
    { name = "types-requests" },
    { name = "typing-extensions" },
]
sdist = { url = "https://files.pythonhosted.org/packages/ed/c7/fd1e4c61cf3f0aac9d9d73fce63a766c9778e1270f7a26812eb289b4851d/cohere-5.16.1.tar.gz", hash = "sha256:02aa87668689ad0fbac2cda979c190310afdb99fb132552e8848fdd0aff7cd40", size = 162300, upload-time = "2025-07-09T20:47:36.348Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/82/c6/72309ac75f3567425ca31a601ad394bfee8d0f4a1569dfbc80cbb2890d07/cohere-5.16.1-py3-none-any.whl", hash = "sha256:37e2c1d69b1804071b5e5f5cb44f8b74127e318376e234572d021a1a729c6baa", size = 291894, upload-time = "2025-07-09T20:47:34.919Z" },
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
name = "coloredlogs"
version = "15.0.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "humanfriendly" },
]
sdist = { url = "https://files.pythonhosted.org/packages/cc/c7/eed8f27100517e8c0e6b923d5f0845d0cb99763da6fdee00478f91db7325/coloredlogs-15.0.1.tar.gz", hash = "sha256:7c991aa71a4577af2f82600d8f8f3a89f936baeaf9b50a9c197da014e5bf16b0", size = 278520, upload-time = "2021-06-11T10:22:45.202Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/a7/06/3d6badcf13db419e25b07041d9c7b4a2c331d3f4e7134445ec5df57714cd/coloredlogs-15.0.1-py2.py3-none-any.whl", hash = "sha256:612ee75c546f53e92e70049c9dbfcc18c935a2b9a53b66085ce9ef6a6e5c0934", size = 46018, upload-time = "2021-06-11T10:22:42.561Z" },
]

[[package]]
name = "comm"
version = "0.2.2"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "traitlets" },
]
sdist = { url = "https://files.pythonhosted.org/packages/e9/a8/fb783cb0abe2b5fded9f55e5703015cdf1c9c85b3669087c538dd15a6a86/comm-0.2.2.tar.gz", hash = "sha256:3fd7a84065306e07bea1773df6eb8282de51ba82f77c72f9c85716ab11fe980e", size = 6210, upload-time = "2024-03-12T16:53:41.133Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/e6/75/49e5bfe642f71f272236b5b2d2691cf915a7283cc0ceda56357b61daa538/comm-0.2.2-py3-none-any.whl", hash = "sha256:e6fb86cb70ff661ee8c9c14e7d36d6de3b4066f1441be4063df9c5009f0a64d3", size = 7180, upload-time = "2024-03-12T16:53:39.226Z" },
]

[[package]]
name = "contourpy"
version = "1.3.2"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "numpy" },
]
sdist = { url = "https://files.pythonhosted.org/packages/66/54/eb9bfc647b19f2009dd5c7f5ec51c4e6ca831725f1aea7a993034f483147/contourpy-1.3.2.tar.gz", hash = "sha256:b6945942715a034c671b7fc54f9588126b0b8bf23db2696e3ca8328f3ff0ab54", size = 13466130, upload-time = "2025-04-15T17:47:53.79Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/b3/b9/ede788a0b56fc5b071639d06c33cb893f68b1178938f3425debebe2dab78/contourpy-1.3.2-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:6a37a2fb93d4df3fc4c0e363ea4d16f83195fc09c891bc8ce072b9d084853445", size = 269636, upload-time = "2025-04-15T17:35:54.473Z" },
    { url = "https://files.pythonhosted.org/packages/e6/75/3469f011d64b8bbfa04f709bfc23e1dd71be54d05b1b083be9f5b22750d1/contourpy-1.3.2-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:b7cd50c38f500bbcc9b6a46643a40e0913673f869315d8e70de0438817cb7773", size = 254636, upload-time = "2025-04-15T17:35:58.283Z" },
    { url = "https://files.pythonhosted.org/packages/8d/2f/95adb8dae08ce0ebca4fd8e7ad653159565d9739128b2d5977806656fcd2/contourpy-1.3.2-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:d6658ccc7251a4433eebd89ed2672c2ed96fba367fd25ca9512aa92a4b46c4f1", size = 313053, upload-time = "2025-04-15T17:36:03.235Z" },
    { url = "https://files.pythonhosted.org/packages/c3/a6/8ccf97a50f31adfa36917707fe39c9a0cbc24b3bbb58185577f119736cc9/contourpy-1.3.2-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:70771a461aaeb335df14deb6c97439973d253ae70660ca085eec25241137ef43", size = 352985, upload-time = "2025-04-15T17:36:08.275Z" },
    { url = "https://files.pythonhosted.org/packages/1d/b6/7925ab9b77386143f39d9c3243fdd101621b4532eb126743201160ffa7e6/contourpy-1.3.2-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:65a887a6e8c4cd0897507d814b14c54a8c2e2aa4ac9f7686292f9769fcf9a6ab", size = 323750, upload-time = "2025-04-15T17:36:13.29Z" },
    { url = "https://files.pythonhosted.org/packages/c2/f3/20c5d1ef4f4748e52d60771b8560cf00b69d5c6368b5c2e9311bcfa2a08b/contourpy-1.3.2-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:3859783aefa2b8355697f16642695a5b9792e7a46ab86da1118a4a23a51a33d7", size = 326246, upload-time = "2025-04-15T17:36:18.329Z" },
    { url = "https://files.pythonhosted.org/packages/8c/e5/9dae809e7e0b2d9d70c52b3d24cba134dd3dad979eb3e5e71f5df22ed1f5/contourpy-1.3.2-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:eab0f6db315fa4d70f1d8ab514e527f0366ec021ff853d7ed6a2d33605cf4b83", size = 1308728, upload-time = "2025-04-15T17:36:33.878Z" },
    { url = "https://files.pythonhosted.org/packages/e2/4a/0058ba34aeea35c0b442ae61a4f4d4ca84d6df8f91309bc2d43bb8dd248f/contourpy-1.3.2-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:d91a3ccc7fea94ca0acab82ceb77f396d50a1f67412efe4c526f5d20264e6ecd", size = 1375762, upload-time = "2025-04-15T17:36:51.295Z" },
    { url = "https://files.pythonhosted.org/packages/09/33/7174bdfc8b7767ef2c08ed81244762d93d5c579336fc0b51ca57b33d1b80/contourpy-1.3.2-cp311-cp311-win32.whl", hash = "sha256:1c48188778d4d2f3d48e4643fb15d8608b1d01e4b4d6b0548d9b336c28fc9b6f", size = 178196, upload-time = "2025-04-15T17:36:55.002Z" },
    { url = "https://files.pythonhosted.org/packages/5e/fe/4029038b4e1c4485cef18e480b0e2cd2d755448bb071eb9977caac80b77b/contourpy-1.3.2-cp311-cp311-win_amd64.whl", hash = "sha256:5ebac872ba09cb8f2131c46b8739a7ff71de28a24c869bcad554477eb089a878", size = 222017, upload-time = "2025-04-15T17:36:58.576Z" },
    { url = "https://files.pythonhosted.org/packages/34/f7/44785876384eff370c251d58fd65f6ad7f39adce4a093c934d4a67a7c6b6/contourpy-1.3.2-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:4caf2bcd2969402bf77edc4cb6034c7dd7c0803213b3523f111eb7460a51b8d2", size = 271580, upload-time = "2025-04-15T17:37:03.105Z" },
    { url = "https://files.pythonhosted.org/packages/93/3b/0004767622a9826ea3d95f0e9d98cd8729015768075d61f9fea8eeca42a8/contourpy-1.3.2-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:82199cb78276249796419fe36b7386bd8d2cc3f28b3bc19fe2454fe2e26c4c15", size = 255530, upload-time = "2025-04-15T17:37:07.026Z" },
    { url = "https://files.pythonhosted.org/packages/e7/bb/7bd49e1f4fa805772d9fd130e0d375554ebc771ed7172f48dfcd4ca61549/contourpy-1.3.2-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:106fab697af11456fcba3e352ad50effe493a90f893fca6c2ca5c033820cea92", size = 307688, upload-time = "2025-04-15T17:37:11.481Z" },
    { url = "https://files.pythonhosted.org/packages/fc/97/e1d5dbbfa170725ef78357a9a0edc996b09ae4af170927ba8ce977e60a5f/contourpy-1.3.2-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:d14f12932a8d620e307f715857107b1d1845cc44fdb5da2bc8e850f5ceba9f87", size = 347331, upload-time = "2025-04-15T17:37:18.212Z" },
    { url = "https://files.pythonhosted.org/packages/6f/66/e69e6e904f5ecf6901be3dd16e7e54d41b6ec6ae3405a535286d4418ffb4/contourpy-1.3.2-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:532fd26e715560721bb0d5fc7610fce279b3699b018600ab999d1be895b09415", size = 318963, upload-time = "2025-04-15T17:37:22.76Z" },
    { url = "https://files.pythonhosted.org/packages/a8/32/b8a1c8965e4f72482ff2d1ac2cd670ce0b542f203c8e1d34e7c3e6925da7/contourpy-1.3.2-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:f26b383144cf2d2c29f01a1e8170f50dacf0eac02d64139dcd709a8ac4eb3cfe", size = 323681, upload-time = "2025-04-15T17:37:33.001Z" },
    { url = "https://files.pythonhosted.org/packages/30/c6/12a7e6811d08757c7162a541ca4c5c6a34c0f4e98ef2b338791093518e40/contourpy-1.3.2-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:c49f73e61f1f774650a55d221803b101d966ca0c5a2d6d5e4320ec3997489441", size = 1308674, upload-time = "2025-04-15T17:37:48.64Z" },
    { url = "https://files.pythonhosted.org/packages/2a/8a/bebe5a3f68b484d3a2b8ffaf84704b3e343ef1addea528132ef148e22b3b/contourpy-1.3.2-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:3d80b2c0300583228ac98d0a927a1ba6a2ba6b8a742463c564f1d419ee5b211e", size = 1380480, upload-time = "2025-04-15T17:38:06.7Z" },
    { url = "https://files.pythonhosted.org/packages/34/db/fcd325f19b5978fb509a7d55e06d99f5f856294c1991097534360b307cf1/contourpy-1.3.2-cp312-cp312-win32.whl", hash = "sha256:90df94c89a91b7362e1142cbee7568f86514412ab8a2c0d0fca72d7e91b62912", size = 178489, upload-time = "2025-04-15T17:38:10.338Z" },
    { url = "https://files.pythonhosted.org/packages/01/c8/fadd0b92ffa7b5eb5949bf340a63a4a496a6930a6c37a7ba0f12acb076d6/contourpy-1.3.2-cp312-cp312-win_amd64.whl", hash = "sha256:8c942a01d9163e2e5cfb05cb66110121b8d07ad438a17f9e766317bcb62abf73", size = 223042, upload-time = "2025-04-15T17:38:14.239Z" },
    { url = "https://files.pythonhosted.org/packages/2e/61/5673f7e364b31e4e7ef6f61a4b5121c5f170f941895912f773d95270f3a2/contourpy-1.3.2-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:de39db2604ae755316cb5967728f4bea92685884b1e767b7c24e983ef5f771cb", size = 271630, upload-time = "2025-04-15T17:38:19.142Z" },
    { url = "https://files.pythonhosted.org/packages/ff/66/a40badddd1223822c95798c55292844b7e871e50f6bfd9f158cb25e0bd39/contourpy-1.3.2-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:3f9e896f447c5c8618f1edb2bafa9a4030f22a575ec418ad70611450720b5b08", size = 255670, upload-time = "2025-04-15T17:38:23.688Z" },
    { url = "https://files.pythonhosted.org/packages/1e/c7/cf9fdee8200805c9bc3b148f49cb9482a4e3ea2719e772602a425c9b09f8/contourpy-1.3.2-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:71e2bd4a1c4188f5c2b8d274da78faab884b59df20df63c34f74aa1813c4427c", size = 306694, upload-time = "2025-04-15T17:38:28.238Z" },
    { url = "https://files.pythonhosted.org/packages/dd/e7/ccb9bec80e1ba121efbffad7f38021021cda5be87532ec16fd96533bb2e0/contourpy-1.3.2-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:de425af81b6cea33101ae95ece1f696af39446db9682a0b56daaa48cfc29f38f", size = 345986, upload-time = "2025-04-15T17:38:33.502Z" },
    { url = "https://files.pythonhosted.org/packages/dc/49/ca13bb2da90391fa4219fdb23b078d6065ada886658ac7818e5441448b78/contourpy-1.3.2-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:977e98a0e0480d3fe292246417239d2d45435904afd6d7332d8455981c408b85", size = 318060, upload-time = "2025-04-15T17:38:38.672Z" },
    { url = "https://files.pythonhosted.org/packages/c8/65/5245ce8c548a8422236c13ffcdcdada6a2a812c361e9e0c70548bb40b661/contourpy-1.3.2-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:434f0adf84911c924519d2b08fc10491dd282b20bdd3fa8f60fd816ea0b48841", size = 322747, upload-time = "2025-04-15T17:38:43.712Z" },
    { url = "https://files.pythonhosted.org/packages/72/30/669b8eb48e0a01c660ead3752a25b44fdb2e5ebc13a55782f639170772f9/contourpy-1.3.2-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:c66c4906cdbc50e9cba65978823e6e00b45682eb09adbb78c9775b74eb222422", size = 1308895, upload-time = "2025-04-15T17:39:00.224Z" },
    { url = "https://files.pythonhosted.org/packages/05/5a/b569f4250decee6e8d54498be7bdf29021a4c256e77fe8138c8319ef8eb3/contourpy-1.3.2-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:8b7fc0cd78ba2f4695fd0a6ad81a19e7e3ab825c31b577f384aa9d7817dc3bef", size = 1379098, upload-time = "2025-04-15T17:43:29.649Z" },
    { url = "https://files.pythonhosted.org/packages/19/ba/b227c3886d120e60e41b28740ac3617b2f2b971b9f601c835661194579f1/contourpy-1.3.2-cp313-cp313-win32.whl", hash = "sha256:15ce6ab60957ca74cff444fe66d9045c1fd3e92c8936894ebd1f3eef2fff075f", size = 178535, upload-time = "2025-04-15T17:44:44.532Z" },
    { url = "https://files.pythonhosted.org/packages/12/6e/2fed56cd47ca739b43e892707ae9a13790a486a3173be063681ca67d2262/contourpy-1.3.2-cp313-cp313-win_amd64.whl", hash = "sha256:e1578f7eafce927b168752ed7e22646dad6cd9bca673c60bff55889fa236ebf9", size = 223096, upload-time = "2025-04-15T17:44:48.194Z" },
    { url = "https://files.pythonhosted.org/packages/54/4c/e76fe2a03014a7c767d79ea35c86a747e9325537a8b7627e0e5b3ba266b4/contourpy-1.3.2-cp313-cp313t-macosx_10_13_x86_64.whl", hash = "sha256:0475b1f6604896bc7c53bb070e355e9321e1bc0d381735421a2d2068ec56531f", size = 285090, upload-time = "2025-04-15T17:43:34.084Z" },
    { url = "https://files.pythonhosted.org/packages/7b/e2/5aba47debd55d668e00baf9651b721e7733975dc9fc27264a62b0dd26eb8/contourpy-1.3.2-cp313-cp313t-macosx_11_0_arm64.whl", hash = "sha256:c85bb486e9be652314bb5b9e2e3b0d1b2e643d5eec4992c0fbe8ac71775da739", size = 268643, upload-time = "2025-04-15T17:43:38.626Z" },
    { url = "https://files.pythonhosted.org/packages/a1/37/cd45f1f051fe6230f751cc5cdd2728bb3a203f5619510ef11e732109593c/contourpy-1.3.2-cp313-cp313t-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:745b57db7758f3ffc05a10254edd3182a2a83402a89c00957a8e8a22f5582823", size = 310443, upload-time = "2025-04-15T17:43:44.522Z" },
    { url = "https://files.pythonhosted.org/packages/8b/a2/36ea6140c306c9ff6dd38e3bcec80b3b018474ef4d17eb68ceecd26675f4/contourpy-1.3.2-cp313-cp313t-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:970e9173dbd7eba9b4e01aab19215a48ee5dd3f43cef736eebde064a171f89a5", size = 349865, upload-time = "2025-04-15T17:43:49.545Z" },
    { url = "https://files.pythonhosted.org/packages/95/b7/2fc76bc539693180488f7b6cc518da7acbbb9e3b931fd9280504128bf956/contourpy-1.3.2-cp313-cp313t-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:c6c4639a9c22230276b7bffb6a850dfc8258a2521305e1faefe804d006b2e532", size = 321162, upload-time = "2025-04-15T17:43:54.203Z" },
    { url = "https://files.pythonhosted.org/packages/f4/10/76d4f778458b0aa83f96e59d65ece72a060bacb20cfbee46cf6cd5ceba41/contourpy-1.3.2-cp313-cp313t-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:cc829960f34ba36aad4302e78eabf3ef16a3a100863f0d4eeddf30e8a485a03b", size = 327355, upload-time = "2025-04-15T17:44:01.025Z" },
    { url = "https://files.pythonhosted.org/packages/43/a3/10cf483ea683f9f8ab096c24bad3cce20e0d1dd9a4baa0e2093c1c962d9d/contourpy-1.3.2-cp313-cp313t-musllinux_1_2_aarch64.whl", hash = "sha256:d32530b534e986374fc19eaa77fcb87e8a99e5431499949b828312bdcd20ac52", size = 1307935, upload-time = "2025-04-15T17:44:17.322Z" },
    { url = "https://files.pythonhosted.org/packages/78/73/69dd9a024444489e22d86108e7b913f3528f56cfc312b5c5727a44188471/contourpy-1.3.2-cp313-cp313t-musllinux_1_2_x86_64.whl", hash = "sha256:e298e7e70cf4eb179cc1077be1c725b5fd131ebc81181bf0c03525c8abc297fd", size = 1372168, upload-time = "2025-04-15T17:44:33.43Z" },
    { url = "https://files.pythonhosted.org/packages/0f/1b/96d586ccf1b1a9d2004dd519b25fbf104a11589abfd05484ff12199cca21/contourpy-1.3.2-cp313-cp313t-win32.whl", hash = "sha256:d0e589ae0d55204991450bb5c23f571c64fe43adaa53f93fc902a84c96f52fe1", size = 189550, upload-time = "2025-04-15T17:44:37.092Z" },
    { url = "https://files.pythonhosted.org/packages/b0/e6/6000d0094e8a5e32ad62591c8609e269febb6e4db83a1c75ff8868b42731/contourpy-1.3.2-cp313-cp313t-win_amd64.whl", hash = "sha256:78e9253c3de756b3f6a5174d024c4835acd59eb3f8e2ca13e775dbffe1558f69", size = 238214, upload-time = "2025-04-15T17:44:40.827Z" },
    { url = "https://files.pythonhosted.org/packages/ff/c0/91f1215d0d9f9f343e4773ba6c9b89e8c0cc7a64a6263f21139da639d848/contourpy-1.3.2-pp311-pypy311_pp73-macosx_10_15_x86_64.whl", hash = "sha256:5f5964cdad279256c084b69c3f412b7801e15356b16efa9d78aa974041903da0", size = 266807, upload-time = "2025-04-15T17:45:15.535Z" },
    { url = "https://files.pythonhosted.org/packages/d4/79/6be7e90c955c0487e7712660d6cead01fa17bff98e0ea275737cc2bc8e71/contourpy-1.3.2-pp311-pypy311_pp73-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:49b65a95d642d4efa8f64ba12558fcb83407e58a2dfba9d796d77b63ccfcaff5", size = 318729, upload-time = "2025-04-15T17:45:20.166Z" },
    { url = "https://files.pythonhosted.org/packages/87/68/7f46fb537958e87427d98a4074bcde4b67a70b04900cfc5ce29bc2f556c1/contourpy-1.3.2-pp311-pypy311_pp73-win_amd64.whl", hash = "sha256:8c5acb8dddb0752bf252e01a3035b21443158910ac16a3b0d20e7fed7d534ce5", size = 221791, upload-time = "2025-04-15T17:45:24.794Z" },
]

[[package]]
name = "csscompressor"
version = "0.9.5"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/f1/2a/8c3ac3d8bc94e6de8d7ae270bb5bc437b210bb9d6d9e46630c98f4abd20c/csscompressor-0.9.5.tar.gz", hash = "sha256:afa22badbcf3120a4f392e4d22f9fff485c044a1feda4a950ecc5eba9dd31a05", size = 237808, upload-time = "2017-11-26T21:13:08.238Z" }

[[package]]
name = "cssselect2"
version = "0.8.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "tinycss2" },
    { name = "webencodings" },
]
sdist = { url = "https://files.pythonhosted.org/packages/9f/86/fd7f58fc498b3166f3a7e8e0cddb6e620fe1da35b02248b1bd59e95dbaaa/cssselect2-0.8.0.tar.gz", hash = "sha256:7674ffb954a3b46162392aee2a3a0aedb2e14ecf99fcc28644900f4e6e3e9d3a", size = 35716, upload-time = "2025-03-05T14:46:07.988Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/0f/e7/aa315e6a749d9b96c2504a1ba0ba031ba2d0517e972ce22682e3fccecb09/cssselect2-0.8.0-py3-none-any.whl", hash = "sha256:46fc70ebc41ced7a32cd42d58b1884d72ade23d21e5a4eaaf022401c13f0e76e", size = 15454, upload-time = "2025-03-05T14:46:06.463Z" },
]

[[package]]
name = "cycler"
version = "0.12.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/a9/95/a3dbbb5028f35eafb79008e7522a75244477d2838f38cbb722248dabc2a8/cycler-0.12.1.tar.gz", hash = "sha256:88bb128f02ba341da8ef447245a9e138fae777f6a23943da4540077d3601eb1c", size = 7615, upload-time = "2023-10-07T05:32:18.335Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/e7/05/c19819d5e3d95294a6f5947fb9b9629efb316b96de511b418c53d245aae6/cycler-0.12.1-py3-none-any.whl", hash = "sha256:85cef7cff222d8644161529808465972e51340599459b8ac3ccbac5a854e0d30", size = 8321, upload-time = "2023-10-07T05:32:16.783Z" },
]

[[package]]
name = "dataclasses-json"
version = "0.6.7"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "marshmallow" },
    { name = "typing-inspect" },
]
sdist = { url = "https://files.pythonhosted.org/packages/64/a4/f71d9cf3a5ac257c993b5ca3f93df5f7fb395c725e7f1e6479d2514173c3/dataclasses_json-0.6.7.tar.gz", hash = "sha256:b6b3e528266ea45b9535223bc53ca645f5208833c29229e847b3f26a1cc55fc0", size = 32227, upload-time = "2024-06-09T16:20:19.103Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/c3/be/d0d44e092656fe7a06b55e6103cbce807cdbdee17884a5367c68c9860853/dataclasses_json-0.6.7-py3-none-any.whl", hash = "sha256:0dbf33f26c8d5305befd61b39d2b3414e8a407bedc2834dea9b8d642666fb40a", size = 28686, upload-time = "2024-06-09T16:20:16.715Z" },
]

[[package]]
name = "debugpy"
version = "1.8.14"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/bd/75/087fe07d40f490a78782ff3b0a30e3968936854105487decdb33446d4b0e/debugpy-1.8.14.tar.gz", hash = "sha256:7cd287184318416850aa8b60ac90105837bb1e59531898c07569d197d2ed5322", size = 1641444, upload-time = "2025-04-10T19:46:10.981Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/67/e8/57fe0c86915671fd6a3d2d8746e40485fd55e8d9e682388fbb3a3d42b86f/debugpy-1.8.14-cp311-cp311-macosx_14_0_universal2.whl", hash = "sha256:1b2ac8c13b2645e0b1eaf30e816404990fbdb168e193322be8f545e8c01644a9", size = 2175064, upload-time = "2025-04-10T19:46:19.486Z" },
    { url = "https://files.pythonhosted.org/packages/3b/97/2b2fd1b1c9569c6764ccdb650a6f752e4ac31be465049563c9eb127a8487/debugpy-1.8.14-cp311-cp311-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:cf431c343a99384ac7eab2f763980724834f933a271e90496944195318c619e2", size = 3132359, upload-time = "2025-04-10T19:46:21.192Z" },
    { url = "https://files.pythonhosted.org/packages/c0/ee/b825c87ed06256ee2a7ed8bab8fb3bb5851293bf9465409fdffc6261c426/debugpy-1.8.14-cp311-cp311-win32.whl", hash = "sha256:c99295c76161ad8d507b413cd33422d7c542889fbb73035889420ac1fad354f2", size = 5133269, upload-time = "2025-04-10T19:46:23.047Z" },
    { url = "https://files.pythonhosted.org/packages/d5/a6/6c70cd15afa43d37839d60f324213843174c1d1e6bb616bd89f7c1341bac/debugpy-1.8.14-cp311-cp311-win_amd64.whl", hash = "sha256:7816acea4a46d7e4e50ad8d09d963a680ecc814ae31cdef3622eb05ccacf7b01", size = 5158156, upload-time = "2025-04-10T19:46:24.521Z" },
    { url = "https://files.pythonhosted.org/packages/d9/2a/ac2df0eda4898f29c46eb6713a5148e6f8b2b389c8ec9e425a4a1d67bf07/debugpy-1.8.14-cp312-cp312-macosx_14_0_universal2.whl", hash = "sha256:8899c17920d089cfa23e6005ad9f22582fd86f144b23acb9feeda59e84405b84", size = 2501268, upload-time = "2025-04-10T19:46:26.044Z" },
    { url = "https://files.pythonhosted.org/packages/10/53/0a0cb5d79dd9f7039169f8bf94a144ad3efa52cc519940b3b7dde23bcb89/debugpy-1.8.14-cp312-cp312-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:f6bb5c0dcf80ad5dbc7b7d6eac484e2af34bdacdf81df09b6a3e62792b722826", size = 4221077, upload-time = "2025-04-10T19:46:27.464Z" },
    { url = "https://files.pythonhosted.org/packages/f8/d5/84e01821f362327bf4828728aa31e907a2eca7c78cd7c6ec062780d249f8/debugpy-1.8.14-cp312-cp312-win32.whl", hash = "sha256:281d44d248a0e1791ad0eafdbbd2912ff0de9eec48022a5bfbc332957487ed3f", size = 5255127, upload-time = "2025-04-10T19:46:29.467Z" },
    { url = "https://files.pythonhosted.org/packages/33/16/1ed929d812c758295cac7f9cf3dab5c73439c83d9091f2d91871e648093e/debugpy-1.8.14-cp312-cp312-win_amd64.whl", hash = "sha256:5aa56ef8538893e4502a7d79047fe39b1dae08d9ae257074c6464a7b290b806f", size = 5297249, upload-time = "2025-04-10T19:46:31.538Z" },
    { url = "https://files.pythonhosted.org/packages/4d/e4/395c792b243f2367d84202dc33689aa3d910fb9826a7491ba20fc9e261f5/debugpy-1.8.14-cp313-cp313-macosx_14_0_universal2.whl", hash = "sha256:329a15d0660ee09fec6786acdb6e0443d595f64f5d096fc3e3ccf09a4259033f", size = 2485676, upload-time = "2025-04-10T19:46:32.96Z" },
    { url = "https://files.pythonhosted.org/packages/ba/f1/6f2ee3f991327ad9e4c2f8b82611a467052a0fb0e247390192580e89f7ff/debugpy-1.8.14-cp313-cp313-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:0f920c7f9af409d90f5fd26e313e119d908b0dd2952c2393cd3247a462331f15", size = 4217514, upload-time = "2025-04-10T19:46:34.336Z" },
    { url = "https://files.pythonhosted.org/packages/79/28/b9d146f8f2dc535c236ee09ad3e5ac899adb39d7a19b49f03ac95d216beb/debugpy-1.8.14-cp313-cp313-win32.whl", hash = "sha256:3784ec6e8600c66cbdd4ca2726c72d8ca781e94bce2f396cc606d458146f8f4e", size = 5254756, upload-time = "2025-04-10T19:46:36.199Z" },
    { url = "https://files.pythonhosted.org/packages/e0/62/a7b4a57013eac4ccaef6977966e6bec5c63906dd25a86e35f155952e29a1/debugpy-1.8.14-cp313-cp313-win_amd64.whl", hash = "sha256:684eaf43c95a3ec39a96f1f5195a7ff3d4144e4a18d69bb66beeb1a6de605d6e", size = 5297119, upload-time = "2025-04-10T19:46:38.141Z" },
    { url = "https://files.pythonhosted.org/packages/97/1a/481f33c37ee3ac8040d3d51fc4c4e4e7e61cb08b8bc8971d6032acc2279f/debugpy-1.8.14-py2.py3-none-any.whl", hash = "sha256:5cd9a579d553b6cb9759a7908a41988ee6280b961f24f63336835d9418216a20", size = 5256230, upload-time = "2025-04-10T19:46:54.077Z" },
]

[[package]]
name = "decorator"
version = "5.2.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/43/fa/6d96a0978d19e17b68d634497769987b16c8f4cd0a7a05048bec693caa6b/decorator-5.2.1.tar.gz", hash = "sha256:65f266143752f734b0a7cc83c46f4618af75b8c5911b00ccb61d0ac9b6da0360", size = 56711, upload-time = "2025-02-24T04:41:34.073Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/4e/8c/f3147f5c4b73e7550fe5f9352eaa956ae838d5c51eb58e7a25b9f3e2643b/decorator-5.2.1-py3-none-any.whl", hash = "sha256:d316bb415a2d9e2d2b3abcc4084c6502fc09240e292cd76a76afc106a1c8e04a", size = 9190, upload-time = "2025-02-24T04:41:32.565Z" },
]

[[package]]
name = "deeplake"
version = "4.2.14"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "numpy" },
]
wheels = [
    { url = "https://files.pythonhosted.org/packages/c0/03/72f7fa3c56065d173b1c7eafa5d89a6e6b66f032881a7282e48c940c8d12/deeplake-4.2.14-cp311-cp311-macosx_10_12_x86_64.whl", hash = "sha256:6d10bc2431ef91e10791c55aa9dc913cc6d1cdc5ce91781c3b40a0de62c2b044", size = 29631779, upload-time = "2025-07-08T22:32:16.878Z" },
    { url = "https://files.pythonhosted.org/packages/69/2a/a9adecfe0b72498e2a129f6e9586fd80b52eb13365225645d534c8153679/deeplake-4.2.14-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:6b4af747c9ca59d1122445c32d4fdca1939d6c0ddcd164ae5c33704ff8464255", size = 29005223, upload-time = "2025-07-08T22:32:22.148Z" },
    { url = "https://files.pythonhosted.org/packages/45/7f/4a17eb863b4aa0522bb031c6b15bbdfb164da1fabe2aa49da9ad972cd3fe/deeplake-4.2.14-cp311-cp311-manylinux2014_aarch64.whl", hash = "sha256:d6867b6e21d3130d396b5bcc164dbd16f9dae657bee9935a07115696b6f9ff3a", size = 31431418, upload-time = "2025-07-08T23:16:13.182Z" },
    { url = "https://files.pythonhosted.org/packages/41/3f/d7117de15b640a35e912277583b2d142d8b85a1d721f31d627772698ec29/deeplake-4.2.14-cp311-cp311-manylinux2014_x86_64.whl", hash = "sha256:f3d153684cda7d406de1257fa6af2e30241b6ee94c08e7a8cd0b4714ce76ec9a", size = 32373803, upload-time = "2025-07-08T22:32:27.356Z" },
    { url = "https://files.pythonhosted.org/packages/9e/c0/a127ef8d87650c3d275d8aa9340a4eceb5d0c6e64994b538d30d4b649897/deeplake-4.2.14-cp312-cp312-macosx_10_12_x86_64.whl", hash = "sha256:d450c297f5d5e40956dfd0c6d3ff613c715f3c7c4a830daaeda939899b5a01cf", size = 29655346, upload-time = "2025-07-08T22:32:33.441Z" },
    { url = "https://files.pythonhosted.org/packages/00/18/3afbb70d33152a13baeda31ed00d51be4c48fc34d115b33ae757a6957445/deeplake-4.2.14-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:13fd1cb8dd203386b59c1bde13611733daca4fe38a014811c070767df22e1f7b", size = 29014575, upload-time = "2025-07-08T22:32:39.715Z" },
    { url = "https://files.pythonhosted.org/packages/96/7e/3beeb92cfd27d310a6e9b8c7704a82348d29f831cf7405cea46a28afcb05/deeplake-4.2.14-cp312-cp312-manylinux2014_aarch64.whl", hash = "sha256:bea882ffdfd51b96b0cce13f9e74b1c5bc2fe7a207221f5d45b26604f2788aed", size = 31422188, upload-time = "2025-07-08T22:32:44.524Z" },
    { url = "https://files.pythonhosted.org/packages/cf/11/b309ab40265673188ff7bf64660ba63376d26e618ee93836ec92da0235fb/deeplake-4.2.14-cp312-cp312-manylinux2014_x86_64.whl", hash = "sha256:91ec4772ed000a0f76a11fae782dedde1908911b4d5317af8d37c9ac8cfa9b5f", size = 32375346, upload-time = "2025-07-08T22:32:51.32Z" },
    { url = "https://files.pythonhosted.org/packages/4e/a3/04d7c6efc7f8017cbc742ded1eb8a1ad409d0f113810a4feb51784482d22/deeplake-4.2.14-cp313-cp313-macosx_10_12_x86_64.whl", hash = "sha256:f3b125c43511d14a3cf8476455ef20fdc7cd7678ccf5630638c2e253d69c43b8", size = 29653093, upload-time = "2025-07-08T22:32:57.92Z" },
    { url = "https://files.pythonhosted.org/packages/e9/0f/a7556de99ba18b1d9bbb4e39e93d18f963af53f2b99b6e724327b1105a70/deeplake-4.2.14-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:8001f78f4d43fd2d4f2501d992aac5a60910341f5750c372d67d1e633288f4e4", size = 29014303, upload-time = "2025-07-08T22:33:03.454Z" },
    { url = "https://files.pythonhosted.org/packages/f4/3f/dec601558b8325e239fad7341e209bb5d6e4e7c9eb70d918f1e9fb8b8507/deeplake-4.2.14-cp313-cp313-manylinux2014_aarch64.whl", hash = "sha256:ba2ae603dd5ea6a0690bd31fe093a1ca44c9045dfb7a9ba2bbaa01c737249b41", size = 31423253, upload-time = "2025-07-08T22:33:08.552Z" },
    { url = "https://files.pythonhosted.org/packages/b6/6b/60e591585ad86350a6c32d1e94ecdd2203c65fe6601f190c87344146ea42/deeplake-4.2.14-cp313-cp313-manylinux2014_x86_64.whl", hash = "sha256:3684b7ed62bf072bcfae3f28f7ec2f7bccbb92762674a0415270c75077cdb8f7", size = 32375603, upload-time = "2025-07-08T22:33:13.385Z" },
]

[[package]]
name = "defusedxml"
version = "0.7.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/0f/d5/c66da9b79e5bdb124974bfe172b4daf3c984ebd9c2a06e2b8a4dc7331c72/defusedxml-0.7.1.tar.gz", hash = "sha256:1bb3032db185915b62d7c6209c5a8792be6a32ab2fedacc84e01b52c51aa3e69", size = 75520, upload-time = "2021-03-08T10:59:26.269Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/07/6c/aa3f2f849e01cb6a001cd8554a88d4c77c5c1a31c95bdf1cf9301e6d9ef4/defusedxml-0.7.1-py2.py3-none-any.whl", hash = "sha256:a352e7e428770286cc899e2542b6cdaedb2b4953ff269a210103ec58f6198a61", size = 25604, upload-time = "2021-03-08T10:59:24.45Z" },
]

[[package]]
name = "diskcache"
version = "5.6.3"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/3f/21/1c1ffc1a039ddcc459db43cc108658f32c57d271d7289a2794e401d0fdb6/diskcache-5.6.3.tar.gz", hash = "sha256:2c3a3fa2743d8535d832ec61c2054a1641f41775aa7c556758a109941e33e4fc", size = 67916, upload-time = "2023-08-31T06:12:00.316Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/3f/27/4570e78fc0bf5ea0ca45eb1de3818a23787af9b390c0b0a0033a1b8236f9/diskcache-5.6.3-py3-none-any.whl", hash = "sha256:5e31b2d5fbad117cc363ebaf6b689474db18a1f6438bc82358b024abd4c2ca19", size = 45550, upload-time = "2023-08-31T06:11:58.822Z" },
]

[[package]]
name = "distro"
version = "1.9.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/fc/f8/98eea607f65de6527f8a2e8885fc8015d3e6f5775df186e443e0964a11c3/distro-1.9.0.tar.gz", hash = "sha256:2fa77c6fd8940f116ee1d6b94a2f90b13b5ea8d019b98bc8bafdcabcdd9bdbed", size = 60722, upload-time = "2023-12-24T09:54:32.31Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/12/b3/231ffd4ab1fc9d679809f356cebee130ac7daa00d6d6f3206dd4fd137e9e/distro-1.9.0-py3-none-any.whl", hash = "sha256:7bffd925d65168f85027d8da9af6bddab658135b840670a223589bc0c8ef02b2", size = 20277, upload-time = "2023-12-24T09:54:30.421Z" },
]

[[package]]
name = "dnspython"
version = "2.7.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/b5/4a/263763cb2ba3816dd94b08ad3a33d5fdae34ecb856678773cc40a3605829/dnspython-2.7.0.tar.gz", hash = "sha256:ce9c432eda0dc91cf618a5cedf1a4e142651196bbcd2c80e89ed5a907e5cfaf1", size = 345197, upload-time = "2024-10-05T20:14:59.362Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/68/1b/e0a87d256e40e8c888847551b20a017a6b98139178505dc7ffb96f04e954/dnspython-2.7.0-py3-none-any.whl", hash = "sha256:b4c34b7d10b51bcc3a5071e7b8dee77939f1e878477eeecc965e9835f63c6c86", size = 313632, upload-time = "2024-10-05T20:14:57.687Z" },
]

[[package]]
name = "docker"
version = "7.1.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "pywin32", marker = "python_full_version < '3.13' and sys_platform == 'win32'" },
    { name = "requests", marker = "python_full_version < '3.13'" },
    { name = "urllib3", marker = "python_full_version < '3.13'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/91/9b/4a2ea29aeba62471211598dac5d96825bb49348fa07e906ea930394a83ce/docker-7.1.0.tar.gz", hash = "sha256:ad8c70e6e3f8926cb8a92619b832b4ea5299e2831c14284663184e200546fa6c", size = 117834, upload-time = "2024-05-23T11:13:57.216Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/e3/26/57c6fb270950d476074c087527a558ccb6f4436657314bfb6cdf484114c4/docker-7.1.0-py3-none-any.whl", hash = "sha256:c96b93b7f0a746f9e77d325bcfb87422a3d8bd4f03136ae8a85b37f1898d5fc0", size = 147774, upload-time = "2024-05-23T11:13:55.01Z" },
]

[[package]]
name = "docutils"
version = "0.21.2"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/ae/ed/aefcc8cd0ba62a0560c3c18c33925362d46c6075480bfa4df87b28e169a9/docutils-0.21.2.tar.gz", hash = "sha256:3a6b18732edf182daa3cd12775bbb338cf5691468f91eeeb109deff6ebfa986f", size = 2204444, upload-time = "2024-04-23T18:57:18.24Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/8f/d7/9322c609343d929e75e7e5e6255e614fcc67572cfd083959cdef3b7aad79/docutils-0.21.2-py3-none-any.whl", hash = "sha256:dafca5b9e384f0e419294eb4d2ff9fa826435bf15f15b7bd45723e8ad76811b2", size = 587408, upload-time = "2024-04-23T18:57:14.835Z" },
]

[[package]]
name = "durationpy"
version = "0.10"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/9d/a4/e44218c2b394e31a6dd0d6b095c4e1f32d0be54c2a4b250032d717647bab/durationpy-0.10.tar.gz", hash = "sha256:1fa6893409a6e739c9c72334fc65cca1f355dbdd93405d30f726deb5bde42fba", size = 3335, upload-time = "2025-05-17T13:52:37.26Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/b0/0d/9feae160378a3553fa9a339b0e9c1a048e147a4127210e286ef18b730f03/durationpy-0.10-py3-none-any.whl", hash = "sha256:3b41e1b601234296b4fb368338fdcd3e13e0b4fb5b67345948f4f2bf9868b286", size = 3922, upload-time = "2025-05-17T13:52:36.463Z" },
]

[[package]]
name = "dydantic"
version = "0.0.8"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "pydantic" },
]
sdist = { url = "https://files.pythonhosted.org/packages/08/c5/2d097e5a4816b15186c1ae06c5cfe3c332e69a0f3556dc6cee2d370acf2a/dydantic-0.0.8.tar.gz", hash = "sha256:14a31d4cdfce314ce3e69e8f8c7c46cbc26ce3ce4485de0832260386c612942f", size = 8115, upload-time = "2025-01-29T20:36:13.771Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/7a/7c/a1b120141a300853d82291faf0ba1a95133fa390e4b7d773647b69c8c0f4/dydantic-0.0.8-py3-none-any.whl", hash = "sha256:cd0a991f523bd8632699872f1c0c4278415dd04783e36adec5428defa0afb721", size = 8637, upload-time = "2025-01-29T20:36:12.217Z" },
]

[[package]]
name = "executing"
version = "2.2.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/91/50/a9d80c47ff289c611ff12e63f7c5d13942c65d68125160cefd768c73e6e4/executing-2.2.0.tar.gz", hash = "sha256:5d108c028108fe2551d1a7b2e8b713341e2cb4fc0aa7dcf966fa4327a5226755", size = 978693, upload-time = "2025-01-22T15:41:29.403Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/7b/8f/c4d9bafc34ad7ad5d8dc16dd1347ee0e507a52c3adb6bfa8887e1c6a26ba/executing-2.2.0-py2.py3-none-any.whl", hash = "sha256:11387150cad388d62750327a53d3339fad4888b39a6fe233c3afbb54ecffd3aa", size = 26702, upload-time = "2025-01-22T15:41:25.929Z" },
]

[[package]]
name = "fastavro"
version = "1.11.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/48/8f/32664a3245247b13702d13d2657ea534daf64e58a3f72a3a2d10598d6916/fastavro-1.11.1.tar.gz", hash = "sha256:bf6acde5ee633a29fb8dfd6dfea13b164722bc3adc05a0e055df080549c1c2f8", size = 1016250, upload-time = "2025-05-18T04:54:31.413Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/8e/63/f33d6fd50d8711f305f07ad8c7b4a25f2092288f376f484c979dcf277b07/fastavro-1.11.1-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:3573340e4564e8962e22f814ac937ffe0d4be5eabbd2250f77738dc47e3c8fe9", size = 957526, upload-time = "2025-05-18T04:54:47.701Z" },
    { url = "https://files.pythonhosted.org/packages/f4/09/a57ad9d8cb9b8affb2e43c29d8fb8cbdc0f1156f8496067a0712c944bacc/fastavro-1.11.1-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:7291cf47735b8bd6ff5d9b33120e6e0974f52fd5dff90cd24151b22018e7fd29", size = 3322808, upload-time = "2025-05-18T04:54:50.419Z" },
    { url = "https://files.pythonhosted.org/packages/86/70/d6df59309d3754d6d4b0c7beca45b9b1a957d6725aed8da3aca247db3475/fastavro-1.11.1-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:bf3bb065d657d5bac8b2cb39945194aa086a9b3354f2da7f89c30e4dc20e08e2", size = 3330870, upload-time = "2025-05-18T04:54:52.406Z" },
    { url = "https://files.pythonhosted.org/packages/ad/ea/122315154d2a799a2787058435ef0d4d289c0e8e575245419436e9b702ca/fastavro-1.11.1-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:8758317c85296b848698132efb13bc44a4fbd6017431cc0f26eaeb0d6fa13d35", size = 3343369, upload-time = "2025-05-18T04:54:54.652Z" },
    { url = "https://files.pythonhosted.org/packages/62/12/7800de5fec36d55a818adf3db3b085b1a033c4edd60323cf6ca0754cf8cb/fastavro-1.11.1-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:ad99d57228f83bf3e2214d183fbf6e2fda97fd649b2bdaf8e9110c36cbb02624", size = 3430629, upload-time = "2025-05-18T04:54:56.513Z" },
    { url = "https://files.pythonhosted.org/packages/48/65/2b74ccfeba9dcc3f7dbe64907307386b4a0af3f71d2846f63254df0f1e1d/fastavro-1.11.1-cp311-cp311-win_amd64.whl", hash = "sha256:9134090178bdbf9eefd467717ced3dc151e27a7e7bfc728260ce512697efe5a4", size = 451621, upload-time = "2025-05-18T04:54:58.156Z" },
    { url = "https://files.pythonhosted.org/packages/99/58/8e789b0a2f532b22e2d090c20d27c88f26a5faadcba4c445c6958ae566cf/fastavro-1.11.1-cp312-cp312-macosx_10_13_universal2.whl", hash = "sha256:e8bc238f2637cd5d15238adbe8fb8c58d2e6f1870e0fb28d89508584670bae4b", size = 939583, upload-time = "2025-05-18T04:54:59.853Z" },
    { url = "https://files.pythonhosted.org/packages/34/3f/02ed44742b1224fe23c9fc9b9b037fc61769df716c083cf80b59a02b9785/fastavro-1.11.1-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:b403933081c83fc4d8a012ee64b86e560a024b1280e3711ee74f2abc904886e8", size = 3257734, upload-time = "2025-05-18T04:55:02.366Z" },
    { url = "https://files.pythonhosted.org/packages/cc/bc/9cc8b19eeee9039dd49719f8b4020771e805def262435f823fa8f27ddeea/fastavro-1.11.1-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:3f6ecb4b5f77aa756d973b7dd1c2fb4e4c95b4832a3c98b059aa96c61870c709", size = 3318218, upload-time = "2025-05-18T04:55:04.352Z" },
    { url = "https://files.pythonhosted.org/packages/39/77/3b73a986606494596b6d3032eadf813a05b59d1623f54384a23de4217d5f/fastavro-1.11.1-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:059893df63ef823b0231b485c9d43016c7e32850cae7bf69f4e9d46dd41c28f2", size = 3297296, upload-time = "2025-05-18T04:55:06.175Z" },
    { url = "https://files.pythonhosted.org/packages/8e/1c/b69ceef6494bd0df14752b5d8648b159ad52566127bfd575e9f5ecc0c092/fastavro-1.11.1-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:5120ffc9a200699218e01777e695a2f08afb3547ba818184198c757dc39417bd", size = 3438056, upload-time = "2025-05-18T04:55:08.276Z" },
    { url = "https://files.pythonhosted.org/packages/ef/11/5c2d0db3bd0e6407546fabae9e267bb0824eacfeba79e7dd81ad88afa27d/fastavro-1.11.1-cp312-cp312-win_amd64.whl", hash = "sha256:7bb9d0d2233f33a52908b6ea9b376fe0baf1144bdfdfb3c6ad326e200a8b56b0", size = 442824, upload-time = "2025-05-18T04:55:10.385Z" },
    { url = "https://files.pythonhosted.org/packages/ec/08/8e25b9e87a98f8c96b25e64565fa1a1208c0095bb6a84a5c8a4b925688a5/fastavro-1.11.1-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:f963b8ddaf179660e814ab420850c1b4ea33e2ad2de8011549d958b21f77f20a", size = 931520, upload-time = "2025-05-18T04:55:11.614Z" },
    { url = "https://files.pythonhosted.org/packages/02/ee/7cf5561ef94781ed6942cee6b394a5e698080f4247f00f158ee396ec244d/fastavro-1.11.1-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:0253e5b6a3c9b62fae9fc3abd8184c5b64a833322b6af7d666d3db266ad879b5", size = 3195989, upload-time = "2025-05-18T04:55:13.732Z" },
    { url = "https://files.pythonhosted.org/packages/b3/31/f02f097d79f090e5c5aca8a743010c4e833a257c0efdeb289c68294f7928/fastavro-1.11.1-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:ca637b150e1f4c0e8e564fad40a16bd922bcb7ffd1a6e4836e6084f2c4f4e8db", size = 3239755, upload-time = "2025-05-18T04:55:16.463Z" },
    { url = "https://files.pythonhosted.org/packages/09/4c/46626b4ee4eb8eb5aa7835973c6ba8890cf082ef2daface6071e788d2992/fastavro-1.11.1-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:76af1709031621828ca6ce7f027f7711fa33ac23e8269e7a5733996ff8d318da", size = 3243788, upload-time = "2025-05-18T04:55:18.544Z" },
    { url = "https://files.pythonhosted.org/packages/a7/6f/8ed42524e9e8dc0554f0f211dd1c6c7a9dde83b95388ddcf7c137e70796f/fastavro-1.11.1-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:8224e6d8d9864d4e55dafbe88920d6a1b8c19cc3006acfac6aa4f494a6af3450", size = 3378330, upload-time = "2025-05-18T04:55:20.887Z" },
    { url = "https://files.pythonhosted.org/packages/b8/51/38cbe243d5facccab40fc43a4c17db264c261be955ce003803d25f0da2c3/fastavro-1.11.1-cp313-cp313-win_amd64.whl", hash = "sha256:cde7ed91b52ff21f0f9f157329760ba7251508ca3e9618af3ffdac986d9faaa2", size = 443115, upload-time = "2025-05-18T04:55:22.107Z" },
    { url = "https://files.pythonhosted.org/packages/d0/57/0d31ed1a49c65ad9f0f0128d9a928972878017781f9d4336f5f60982334c/fastavro-1.11.1-cp313-cp313t-macosx_10_13_universal2.whl", hash = "sha256:e5ed1325c1c414dd954e7a2c5074daefe1eceb672b8c727aa030ba327aa00693", size = 1021401, upload-time = "2025-05-18T04:55:23.431Z" },
    { url = "https://files.pythonhosted.org/packages/56/7a/a3f1a75fbfc16b3eff65dc0efcdb92364967923194312b3f8c8fc2cb95be/fastavro-1.11.1-cp313-cp313t-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:8cd3c95baeec37188899824faf44a5ee94dfc4d8667b05b2f867070c7eb174c4", size = 3384349, upload-time = "2025-05-18T04:55:25.575Z" },
    { url = "https://files.pythonhosted.org/packages/be/84/02bceb7518867df84027232a75225db758b9b45f12017c9743f45b73101e/fastavro-1.11.1-cp313-cp313t-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:2e0babcd81acceb4c60110af9efa25d890dbb68f7de880f806dadeb1e70fe413", size = 3240658, upload-time = "2025-05-18T04:55:27.633Z" },
    { url = "https://files.pythonhosted.org/packages/f2/17/508c846c644d39bc432b027112068b8e96e7560468304d4c0757539dd73a/fastavro-1.11.1-cp313-cp313t-musllinux_1_2_aarch64.whl", hash = "sha256:b2c0cb8063c7208b53b6867983dc6ae7cc80b91116b51d435d2610a5db2fc52f", size = 3372809, upload-time = "2025-05-18T04:55:30.063Z" },
    { url = "https://files.pythonhosted.org/packages/fe/84/9c2917a70ed570ddbfd1d32ac23200c1d011e36c332e59950d2f6d204941/fastavro-1.11.1-cp313-cp313t-musllinux_1_2_x86_64.whl", hash = "sha256:1bc2824e9969c04ab6263d269a1e0e5d40b9bd16ade6b70c29d6ffbc4f3cc102", size = 3387171, upload-time = "2025-05-18T04:55:32.531Z" },
]

[[package]]
name = "fastjsonschema"
version = "2.21.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/8b/50/4b769ce1ac4071a1ef6d86b1a3fb56cdc3a37615e8c5519e1af96cdac366/fastjsonschema-2.21.1.tar.gz", hash = "sha256:794d4f0a58f848961ba16af7b9c85a3e88cd360df008c59aac6fc5ae9323b5d4", size = 373939, upload-time = "2024-12-02T10:55:15.133Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/90/2b/0817a2b257fe88725c25589d89aec060581aabf668707a8d03b2e9e0cb2a/fastjsonschema-2.21.1-py3-none-any.whl", hash = "sha256:c9e5b7e908310918cf494a434eeb31384dd84a98b57a30bcb1f535015b554667", size = 23924, upload-time = "2024-12-02T10:55:07.599Z" },
]

[[package]]
name = "filelock"
version = "3.18.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/0a/10/c23352565a6544bdc5353e0b15fc1c563352101f30e24bf500207a54df9a/filelock-3.18.0.tar.gz", hash = "sha256:adbc88eabb99d2fec8c9c1b229b171f18afa655400173ddc653d5d01501fb9f2", size = 18075, upload-time = "2025-03-14T07:11:40.47Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/4d/36/2a115987e2d8c300a974597416d9de88f2444426de9571f4b59b2cca3acc/filelock-3.18.0-py3-none-any.whl", hash = "sha256:c401f4f8377c4464e6db25fff06205fd89bdd83b65eb0488ed1b160f780e21de", size = 16215, upload-time = "2025-03-14T07:11:39.145Z" },
]

[[package]]
name = "fireworks-ai"
version = "0.15.15"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "httpx" },
    { name = "httpx-sse" },
    { name = "httpx-ws" },
    { name = "pillow" },
    { name = "pydantic" },
]
sdist = { url = "https://files.pythonhosted.org/packages/20/5b/7ed59473cd8420a44d11d15594b00c3395e9af726f7c5a632171b02ffdc8/fireworks_ai-0.15.15.tar.gz", hash = "sha256:d558e02df06844cb33344d33ecfb1c619a5e82d2ec4d8f51a0a45b7de5d3f4a0", size = 91475, upload-time = "2025-06-20T21:11:27.957Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/5f/e7/319a4ce37bed682741bc8ebbb84b7983da3d8cd7ac069d86b52a37d79f2e/fireworks_ai-0.15.15-py3-none-any.whl", hash = "sha256:1047b8e575a536898a827b089b0022c1fab207940f9773b90fa357ebf942f5c9", size = 112831, upload-time = "2025-06-20T21:11:26.701Z" },
]

[[package]]
name = "flatbuffers"
version = "25.2.10"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/e4/30/eb5dce7994fc71a2f685d98ec33cc660c0a5887db5610137e60d8cbc4489/flatbuffers-25.2.10.tar.gz", hash = "sha256:97e451377a41262f8d9bd4295cc836133415cc03d8cb966410a4af92eb00d26e", size = 22170, upload-time = "2025-02-11T04:26:46.257Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/b8/25/155f9f080d5e4bc0082edfda032ea2bc2b8fab3f4d25d46c1e9dd22a1a89/flatbuffers-25.2.10-py2.py3-none-any.whl", hash = "sha256:ebba5f4d5ea615af3f7fd70fc310636fbb2bbd1f566ac0a23d98dd412de50051", size = 30953, upload-time = "2025-02-11T04:26:44.484Z" },
]

[[package]]
name = "fonttools"
version = "4.58.5"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/52/97/5735503e58d3816b0989955ef9b2df07e4c99b246469bd8b3823a14095da/fonttools-4.58.5.tar.gz", hash = "sha256:b2a35b0a19f1837284b3a23dd64fd7761b8911d50911ecd2bdbaf5b2d1b5df9c", size = 3526243, upload-time = "2025-07-03T14:04:47.736Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/14/50/26c683bf6f30dcbde6955c8e07ec6af23764aab86ff06b36383654ab6739/fonttools-4.58.5-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:cda226253bf14c559bc5a17c570d46abd70315c9a687d91c0e01147f87736182", size = 2769557, upload-time = "2025-07-03T14:03:35.383Z" },
    { url = "https://files.pythonhosted.org/packages/b1/00/c3c75fb6196b9ff9988e6a82319ae23f4ae7098e1c01e2408e58d2e7d9c7/fonttools-4.58.5-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:83a96e4a4e65efd6c098da549ec34f328f08963acd2d7bc910ceba01d2dc73e6", size = 2329367, upload-time = "2025-07-03T14:03:37.322Z" },
    { url = "https://files.pythonhosted.org/packages/59/e9/6946366c8e88650c199da9b284559de5d47a6e66ed6d175a166953347959/fonttools-4.58.5-cp311-cp311-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:2d172b92dff59ef8929b4452d5a7b19b8e92081aa87bfb2d82b03b1ff14fc667", size = 5019491, upload-time = "2025-07-03T14:03:39.759Z" },
    { url = "https://files.pythonhosted.org/packages/76/12/2f3f7d09bba7a93bd48dcb54b170fba665f0b7e80e959ac831b907d40785/fonttools-4.58.5-cp311-cp311-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:0bfddfd09aafbbfb3bd98ae67415fbe51eccd614c17db0c8844fe724fbc5d43d", size = 4961579, upload-time = "2025-07-03T14:03:41.611Z" },
    { url = "https://files.pythonhosted.org/packages/2c/95/87e84071189e51c714074646dfac8275b2e9c6b2b118600529cc74f7451e/fonttools-4.58.5-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:cfde5045f1bc92ad11b4b7551807564045a1b38cb037eb3c2bc4e737cd3a8d0f", size = 4997792, upload-time = "2025-07-03T14:03:44.529Z" },
    { url = "https://files.pythonhosted.org/packages/73/47/5c4df7473ecbeb8aa4e01373e4f614ca33f53227fe13ae673c6d5ca99be7/fonttools-4.58.5-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:3515ac47a9a5ac025d2899d195198314023d89492340ba86e4ba79451f7518a8", size = 5109361, upload-time = "2025-07-03T14:03:46.693Z" },
    { url = "https://files.pythonhosted.org/packages/06/00/31406853c570210232b845e08e5a566e15495910790381566ffdbdc7f9a2/fonttools-4.58.5-cp311-cp311-win32.whl", hash = "sha256:9f7e2ab9c10b6811b4f12a0768661325a48e664ec0a0530232c1605896a598db", size = 2201369, upload-time = "2025-07-03T14:03:48.885Z" },
    { url = "https://files.pythonhosted.org/packages/c5/90/ac0facb57962cef53a5734d0be5d2f2936e55aa5c62647c38ca3497263d8/fonttools-4.58.5-cp311-cp311-win_amd64.whl", hash = "sha256:126c16ec4a672c9cb5c1c255dc438d15436b470afc8e9cac25a2d39dd2dc26eb", size = 2249021, upload-time = "2025-07-03T14:03:51.232Z" },
    { url = "https://files.pythonhosted.org/packages/d6/68/66b498ee66f3e7e92fd68476c2509508082b7f57d68c0cdb4b8573f44331/fonttools-4.58.5-cp312-cp312-macosx_10_13_universal2.whl", hash = "sha256:c3af3fefaafb570a03051a0d6899b8374dcf8e6a4560e42575843aef33bdbad6", size = 2754751, upload-time = "2025-07-03T14:03:52.976Z" },
    { url = "https://files.pythonhosted.org/packages/f1/1e/edbc14b79290980c3944a1f43098624bc8965f534964aa03d52041f24cb4/fonttools-4.58.5-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:688137789dbd44e8757ad77b49a771539d8069195ffa9a8bcf18176e90bbd86d", size = 2322342, upload-time = "2025-07-03T14:03:54.957Z" },
    { url = "https://files.pythonhosted.org/packages/c1/d7/3c87cf147185d91c2e946460a5cf68c236427b4a23ab96793ccb7d8017c9/fonttools-4.58.5-cp312-cp312-manylinux1_x86_64.manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_5_x86_64.whl", hash = "sha256:2af65836cf84cd7cb882d0b353bdc73643a497ce23b7414c26499bb8128ca1af", size = 4897011, upload-time = "2025-07-03T14:03:56.829Z" },
    { url = "https://files.pythonhosted.org/packages/a0/d6/fbb44cc85d4195fe54356658bd9f934328b4f74ae14addd90b4b5558b5c9/fonttools-4.58.5-cp312-cp312-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:d2d79cfeb456bf438cb9fb87437634d4d6f228f27572ca5c5355e58472d5519d", size = 4942291, upload-time = "2025-07-03T14:03:59.204Z" },
    { url = "https://files.pythonhosted.org/packages/4d/c8/453f82e21aedf25cdc2ae619c03a73512398cec9bd8b6c3b1c571e0b6632/fonttools-4.58.5-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:0feac9dda9a48a7a342a593f35d50a5cee2dbd27a03a4c4a5192834a4853b204", size = 4886824, upload-time = "2025-07-03T14:04:01.517Z" },
    { url = "https://files.pythonhosted.org/packages/40/54/e9190001b8e22d123f78925b2f508c866d9d18531694b979277ad45d59b0/fonttools-4.58.5-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:36555230e168511e83ad8637232268649634b8dfff6ef58f46e1ebc057a041ad", size = 5038510, upload-time = "2025-07-03T14:04:03.917Z" },
    { url = "https://files.pythonhosted.org/packages/cf/9c/07cdad4774841a6304aabae939f8cbb9538cb1d8e97f5016b334da98e73a/fonttools-4.58.5-cp312-cp312-win32.whl", hash = "sha256:26ec05319353842d127bd02516eacb25b97ca83966e40e9ad6fab85cab0576f4", size = 2188459, upload-time = "2025-07-03T14:04:06.103Z" },
    { url = "https://files.pythonhosted.org/packages/0e/4d/1eaaad22781d55f49d1b184563842172aeb6a4fe53c029e503be81114314/fonttools-4.58.5-cp312-cp312-win_amd64.whl", hash = "sha256:778a632e538f82c1920579c0c01566a8f83dc24470c96efbf2fbac698907f569", size = 2236565, upload-time = "2025-07-03T14:04:08.27Z" },
    { url = "https://files.pythonhosted.org/packages/3a/ee/764dd8b99891f815241f449345863cfed9e546923d9cef463f37fd1d7168/fonttools-4.58.5-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:f4b6f1360da13cecc88c0d60716145b31e1015fbe6a59e32f73a4404e2ea92cf", size = 2745867, upload-time = "2025-07-03T14:04:10.586Z" },
    { url = "https://files.pythonhosted.org/packages/e2/23/8fef484c02fef55e226dfeac4339a015c5480b6a496064058491759ac71e/fonttools-4.58.5-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:4a036822e915692aa2c03e2decc60f49a8190f8111b639c947a4f4e5774d0d7a", size = 2317933, upload-time = "2025-07-03T14:04:12.335Z" },
    { url = "https://files.pythonhosted.org/packages/ab/47/f92b135864fa777e11ad68420bf89446c91a572fe2782745586f8e6aac0c/fonttools-4.58.5-cp313-cp313-manylinux1_x86_64.manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_5_x86_64.whl", hash = "sha256:a6d7709fcf4577b0f294ee6327088884ca95046e1eccde87c53bbba4d5008541", size = 4877844, upload-time = "2025-07-03T14:04:14.58Z" },
    { url = "https://files.pythonhosted.org/packages/3e/65/6c1a83511d8ac32411930495645edb3f8dfabebcb78f08cf6009ba2585ec/fonttools-4.58.5-cp313-cp313-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:b9b5099ca99b79d6d67162778b1b1616fc0e1de02c1a178248a0da8d78a33852", size = 4940106, upload-time = "2025-07-03T14:04:16.563Z" },
    { url = "https://files.pythonhosted.org/packages/fa/90/df8eb77d6cf266cbbba01866a1349a3e9121e0a63002cf8d6754e994f755/fonttools-4.58.5-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:3f2c05a8d82a4d15aebfdb3506e90793aea16e0302cec385134dd960647a36c0", size = 4879458, upload-time = "2025-07-03T14:04:19.584Z" },
    { url = "https://files.pythonhosted.org/packages/26/b1/e32f8de51b7afcfea6ad62780da2fa73212c43a32cd8cafcc852189d7949/fonttools-4.58.5-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:79f0c4b1cc63839b61deeac646d8dba46f8ed40332c2ac1b9997281462c2e4ba", size = 5021917, upload-time = "2025-07-03T14:04:21.736Z" },
    { url = "https://files.pythonhosted.org/packages/89/72/578aa7fe32918dd763c62f447aaed672d665ee10e3eeb1725f4d6493fe96/fonttools-4.58.5-cp313-cp313-win32.whl", hash = "sha256:a1a9a2c462760976882131cbab7d63407813413a2d32cd699e86a1ff22bf7aa5", size = 2186827, upload-time = "2025-07-03T14:04:24.237Z" },
    { url = "https://files.pythonhosted.org/packages/71/a3/21e921b16cb9c029d3308e0cb79c9a937e9ff1fc1ee28c2419f0957b9e7c/fonttools-4.58.5-cp313-cp313-win_amd64.whl", hash = "sha256:bca61b14031a4b7dc87e14bf6ca34c275f8e4b9f7a37bc2fe746b532a924cf30", size = 2235706, upload-time = "2025-07-03T14:04:26.082Z" },
    { url = "https://files.pythonhosted.org/packages/d7/d4/1d85a1996b6188cd2713230e002d79a6f3a289bb17cef600cba385848b72/fonttools-4.58.5-py3-none-any.whl", hash = "sha256:e48a487ed24d9b611c5c4b25db1e50e69e9854ca2670e39a3486ffcd98863ec4", size = 1115318, upload-time = "2025-07-03T14:04:45.378Z" },
]

[[package]]
name = "fqdn"
version = "1.5.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/30/3e/a80a8c077fd798951169626cde3e239adeba7dab75deb3555716415bd9b0/fqdn-1.5.1.tar.gz", hash = "sha256:105ed3677e767fb5ca086a0c1f4bb66ebc3c100be518f0e0d755d9eae164d89f", size = 6015, upload-time = "2021-03-11T07:16:29.08Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/cf/58/8acf1b3e91c58313ce5cb67df61001fc9dcd21be4fadb76c1a2d540e09ed/fqdn-1.5.1-py3-none-any.whl", hash = "sha256:3a179af3761e4df6eb2e026ff9e1a3033d3587bf980a0b1b2e1e5d08d7358014", size = 9121, upload-time = "2021-03-11T07:16:28.351Z" },
]

[[package]]
name = "frozenlist"
version = "1.7.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/79/b1/b64018016eeb087db503b038296fd782586432b9c077fc5c7839e9cb6ef6/frozenlist-1.7.0.tar.gz", hash = "sha256:2e310d81923c2437ea8670467121cc3e9b0f76d3043cc1d2331d56c7fb7a3a8f", size = 45078, upload-time = "2025-06-09T23:02:35.538Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/34/7e/803dde33760128acd393a27eb002f2020ddb8d99d30a44bfbaab31c5f08a/frozenlist-1.7.0-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:aa51e147a66b2d74de1e6e2cf5921890de6b0f4820b257465101d7f37b49fb5a", size = 82251, upload-time = "2025-06-09T23:00:16.279Z" },
    { url = "https://files.pythonhosted.org/packages/75/a9/9c2c5760b6ba45eae11334db454c189d43d34a4c0b489feb2175e5e64277/frozenlist-1.7.0-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:9b35db7ce1cd71d36ba24f80f0c9e7cff73a28d7a74e91fe83e23d27c7828750", size = 48183, upload-time = "2025-06-09T23:00:17.698Z" },
    { url = "https://files.pythonhosted.org/packages/47/be/4038e2d869f8a2da165f35a6befb9158c259819be22eeaf9c9a8f6a87771/frozenlist-1.7.0-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:34a69a85e34ff37791e94542065c8416c1afbf820b68f720452f636d5fb990cd", size = 47107, upload-time = "2025-06-09T23:00:18.952Z" },
    { url = "https://files.pythonhosted.org/packages/79/26/85314b8a83187c76a37183ceed886381a5f992975786f883472fcb6dc5f2/frozenlist-1.7.0-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:4a646531fa8d82c87fe4bb2e596f23173caec9185bfbca5d583b4ccfb95183e2", size = 237333, upload-time = "2025-06-09T23:00:20.275Z" },
    { url = "https://files.pythonhosted.org/packages/1f/fd/e5b64f7d2c92a41639ffb2ad44a6a82f347787abc0c7df5f49057cf11770/frozenlist-1.7.0-cp311-cp311-manylinux_2_17_armv7l.manylinux2014_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:79b2ffbba483f4ed36a0f236ccb85fbb16e670c9238313709638167670ba235f", size = 231724, upload-time = "2025-06-09T23:00:21.705Z" },
    { url = "https://files.pythonhosted.org/packages/20/fb/03395c0a43a5976af4bf7534759d214405fbbb4c114683f434dfdd3128ef/frozenlist-1.7.0-cp311-cp311-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:a26f205c9ca5829cbf82bb2a84b5c36f7184c4316617d7ef1b271a56720d6b30", size = 245842, upload-time = "2025-06-09T23:00:23.148Z" },
    { url = "https://files.pythonhosted.org/packages/d0/15/c01c8e1dffdac5d9803507d824f27aed2ba76b6ed0026fab4d9866e82f1f/frozenlist-1.7.0-cp311-cp311-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:bcacfad3185a623fa11ea0e0634aac7b691aa925d50a440f39b458e41c561d98", size = 239767, upload-time = "2025-06-09T23:00:25.103Z" },
    { url = "https://files.pythonhosted.org/packages/14/99/3f4c6fe882c1f5514b6848aa0a69b20cb5e5d8e8f51a339d48c0e9305ed0/frozenlist-1.7.0-cp311-cp311-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:72c1b0fe8fe451b34f12dce46445ddf14bd2a5bcad7e324987194dc8e3a74c86", size = 224130, upload-time = "2025-06-09T23:00:27.061Z" },
    { url = "https://files.pythonhosted.org/packages/4d/83/220a374bd7b2aeba9d0725130665afe11de347d95c3620b9b82cc2fcab97/frozenlist-1.7.0-cp311-cp311-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:61d1a5baeaac6c0798ff6edfaeaa00e0e412d49946c53fae8d4b8e8b3566c4ae", size = 235301, upload-time = "2025-06-09T23:00:29.02Z" },
    { url = "https://files.pythonhosted.org/packages/03/3c/3e3390d75334a063181625343e8daab61b77e1b8214802cc4e8a1bb678fc/frozenlist-1.7.0-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:7edf5c043c062462f09b6820de9854bf28cc6cc5b6714b383149745e287181a8", size = 234606, upload-time = "2025-06-09T23:00:30.514Z" },
    { url = "https://files.pythonhosted.org/packages/23/1e/58232c19608b7a549d72d9903005e2d82488f12554a32de2d5fb59b9b1ba/frozenlist-1.7.0-cp311-cp311-musllinux_1_2_armv7l.whl", hash = "sha256:d50ac7627b3a1bd2dcef6f9da89a772694ec04d9a61b66cf87f7d9446b4a0c31", size = 248372, upload-time = "2025-06-09T23:00:31.966Z" },
    { url = "https://files.pythonhosted.org/packages/c0/a4/e4a567e01702a88a74ce8a324691e62a629bf47d4f8607f24bf1c7216e7f/frozenlist-1.7.0-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:ce48b2fece5aeb45265bb7a58259f45027db0abff478e3077e12b05b17fb9da7", size = 229860, upload-time = "2025-06-09T23:00:33.375Z" },
    { url = "https://files.pythonhosted.org/packages/73/a6/63b3374f7d22268b41a9db73d68a8233afa30ed164c46107b33c4d18ecdd/frozenlist-1.7.0-cp311-cp311-musllinux_1_2_ppc64le.whl", hash = "sha256:fe2365ae915a1fafd982c146754e1de6ab3478def8a59c86e1f7242d794f97d5", size = 245893, upload-time = "2025-06-09T23:00:35.002Z" },
    { url = "https://files.pythonhosted.org/packages/6d/eb/d18b3f6e64799a79673c4ba0b45e4cfbe49c240edfd03a68be20002eaeaa/frozenlist-1.7.0-cp311-cp311-musllinux_1_2_s390x.whl", hash = "sha256:45a6f2fdbd10e074e8814eb98b05292f27bad7d1883afbe009d96abdcf3bc898", size = 246323, upload-time = "2025-06-09T23:00:36.468Z" },
    { url = "https://files.pythonhosted.org/packages/5a/f5/720f3812e3d06cd89a1d5db9ff6450088b8f5c449dae8ffb2971a44da506/frozenlist-1.7.0-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:21884e23cffabb157a9dd7e353779077bf5b8f9a58e9b262c6caad2ef5f80a56", size = 233149, upload-time = "2025-06-09T23:00:37.963Z" },
    { url = "https://files.pythonhosted.org/packages/69/68/03efbf545e217d5db8446acfd4c447c15b7c8cf4dbd4a58403111df9322d/frozenlist-1.7.0-cp311-cp311-win32.whl", hash = "sha256:284d233a8953d7b24f9159b8a3496fc1ddc00f4db99c324bd5fb5f22d8698ea7", size = 39565, upload-time = "2025-06-09T23:00:39.753Z" },
    { url = "https://files.pythonhosted.org/packages/58/17/fe61124c5c333ae87f09bb67186d65038834a47d974fc10a5fadb4cc5ae1/frozenlist-1.7.0-cp311-cp311-win_amd64.whl", hash = "sha256:387cbfdcde2f2353f19c2f66bbb52406d06ed77519ac7ee21be0232147c2592d", size = 44019, upload-time = "2025-06-09T23:00:40.988Z" },
    { url = "https://files.pythonhosted.org/packages/ef/a2/c8131383f1e66adad5f6ecfcce383d584ca94055a34d683bbb24ac5f2f1c/frozenlist-1.7.0-cp312-cp312-macosx_10_13_universal2.whl", hash = "sha256:3dbf9952c4bb0e90e98aec1bd992b3318685005702656bc6f67c1a32b76787f2", size = 81424, upload-time = "2025-06-09T23:00:42.24Z" },
    { url = "https://files.pythonhosted.org/packages/4c/9d/02754159955088cb52567337d1113f945b9e444c4960771ea90eb73de8db/frozenlist-1.7.0-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:1f5906d3359300b8a9bb194239491122e6cf1444c2efb88865426f170c262cdb", size = 47952, upload-time = "2025-06-09T23:00:43.481Z" },
    { url = "https://files.pythonhosted.org/packages/01/7a/0046ef1bd6699b40acd2067ed6d6670b4db2f425c56980fa21c982c2a9db/frozenlist-1.7.0-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:3dabd5a8f84573c8d10d8859a50ea2dec01eea372031929871368c09fa103478", size = 46688, upload-time = "2025-06-09T23:00:44.793Z" },
    { url = "https://files.pythonhosted.org/packages/d6/a2/a910bafe29c86997363fb4c02069df4ff0b5bc39d33c5198b4e9dd42d8f8/frozenlist-1.7.0-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:aa57daa5917f1738064f302bf2626281a1cb01920c32f711fbc7bc36111058a8", size = 243084, upload-time = "2025-06-09T23:00:46.125Z" },
    { url = "https://files.pythonhosted.org/packages/64/3e/5036af9d5031374c64c387469bfcc3af537fc0f5b1187d83a1cf6fab1639/frozenlist-1.7.0-cp312-cp312-manylinux_2_17_armv7l.manylinux2014_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:c193dda2b6d49f4c4398962810fa7d7c78f032bf45572b3e04dd5249dff27e08", size = 233524, upload-time = "2025-06-09T23:00:47.73Z" },
    { url = "https://files.pythonhosted.org/packages/06/39/6a17b7c107a2887e781a48ecf20ad20f1c39d94b2a548c83615b5b879f28/frozenlist-1.7.0-cp312-cp312-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:bfe2b675cf0aaa6d61bf8fbffd3c274b3c9b7b1623beb3809df8a81399a4a9c4", size = 248493, upload-time = "2025-06-09T23:00:49.742Z" },
    { url = "https://files.pythonhosted.org/packages/be/00/711d1337c7327d88c44d91dd0f556a1c47fb99afc060ae0ef66b4d24793d/frozenlist-1.7.0-cp312-cp312-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:8fc5d5cda37f62b262405cf9652cf0856839c4be8ee41be0afe8858f17f4c94b", size = 244116, upload-time = "2025-06-09T23:00:51.352Z" },
    { url = "https://files.pythonhosted.org/packages/24/fe/74e6ec0639c115df13d5850e75722750adabdc7de24e37e05a40527ca539/frozenlist-1.7.0-cp312-cp312-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:b0d5ce521d1dd7d620198829b87ea002956e4319002ef0bc8d3e6d045cb4646e", size = 224557, upload-time = "2025-06-09T23:00:52.855Z" },
    { url = "https://files.pythonhosted.org/packages/8d/db/48421f62a6f77c553575201e89048e97198046b793f4a089c79a6e3268bd/frozenlist-1.7.0-cp312-cp312-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:488d0a7d6a0008ca0db273c542098a0fa9e7dfaa7e57f70acef43f32b3f69dca", size = 241820, upload-time = "2025-06-09T23:00:54.43Z" },
    { url = "https://files.pythonhosted.org/packages/1d/fa/cb4a76bea23047c8462976ea7b7a2bf53997a0ca171302deae9d6dd12096/frozenlist-1.7.0-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:15a7eaba63983d22c54d255b854e8108e7e5f3e89f647fc854bd77a237e767df", size = 236542, upload-time = "2025-06-09T23:00:56.409Z" },
    { url = "https://files.pythonhosted.org/packages/5d/32/476a4b5cfaa0ec94d3f808f193301debff2ea42288a099afe60757ef6282/frozenlist-1.7.0-cp312-cp312-musllinux_1_2_armv7l.whl", hash = "sha256:1eaa7e9c6d15df825bf255649e05bd8a74b04a4d2baa1ae46d9c2d00b2ca2cb5", size = 249350, upload-time = "2025-06-09T23:00:58.468Z" },
    { url = "https://files.pythonhosted.org/packages/8d/ba/9a28042f84a6bf8ea5dbc81cfff8eaef18d78b2a1ad9d51c7bc5b029ad16/frozenlist-1.7.0-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:e4389e06714cfa9d47ab87f784a7c5be91d3934cd6e9a7b85beef808297cc025", size = 225093, upload-time = "2025-06-09T23:01:00.015Z" },
    { url = "https://files.pythonhosted.org/packages/bc/29/3a32959e68f9cf000b04e79ba574527c17e8842e38c91d68214a37455786/frozenlist-1.7.0-cp312-cp312-musllinux_1_2_ppc64le.whl", hash = "sha256:73bd45e1488c40b63fe5a7df892baf9e2a4d4bb6409a2b3b78ac1c6236178e01", size = 245482, upload-time = "2025-06-09T23:01:01.474Z" },
    { url = "https://files.pythonhosted.org/packages/80/e8/edf2f9e00da553f07f5fa165325cfc302dead715cab6ac8336a5f3d0adc2/frozenlist-1.7.0-cp312-cp312-musllinux_1_2_s390x.whl", hash = "sha256:99886d98e1643269760e5fe0df31e5ae7050788dd288947f7f007209b8c33f08", size = 249590, upload-time = "2025-06-09T23:01:02.961Z" },
    { url = "https://files.pythonhosted.org/packages/1c/80/9a0eb48b944050f94cc51ee1c413eb14a39543cc4f760ed12657a5a3c45a/frozenlist-1.7.0-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:290a172aae5a4c278c6da8a96222e6337744cd9c77313efe33d5670b9f65fc43", size = 237785, upload-time = "2025-06-09T23:01:05.095Z" },
    { url = "https://files.pythonhosted.org/packages/f3/74/87601e0fb0369b7a2baf404ea921769c53b7ae00dee7dcfe5162c8c6dbf0/frozenlist-1.7.0-cp312-cp312-win32.whl", hash = "sha256:426c7bc70e07cfebc178bc4c2bf2d861d720c4fff172181eeb4a4c41d4ca2ad3", size = 39487, upload-time = "2025-06-09T23:01:06.54Z" },
    { url = "https://files.pythonhosted.org/packages/0b/15/c026e9a9fc17585a9d461f65d8593d281fedf55fbf7eb53f16c6df2392f9/frozenlist-1.7.0-cp312-cp312-win_amd64.whl", hash = "sha256:563b72efe5da92e02eb68c59cb37205457c977aa7a449ed1b37e6939e5c47c6a", size = 43874, upload-time = "2025-06-09T23:01:07.752Z" },
    { url = "https://files.pythonhosted.org/packages/24/90/6b2cebdabdbd50367273c20ff6b57a3dfa89bd0762de02c3a1eb42cb6462/frozenlist-1.7.0-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:ee80eeda5e2a4e660651370ebffd1286542b67e268aa1ac8d6dbe973120ef7ee", size = 79791, upload-time = "2025-06-09T23:01:09.368Z" },
    { url = "https://files.pythonhosted.org/packages/83/2e/5b70b6a3325363293fe5fc3ae74cdcbc3e996c2a11dde2fd9f1fb0776d19/frozenlist-1.7.0-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:d1a81c85417b914139e3a9b995d4a1c84559afc839a93cf2cb7f15e6e5f6ed2d", size = 47165, upload-time = "2025-06-09T23:01:10.653Z" },
    { url = "https://files.pythonhosted.org/packages/f4/25/a0895c99270ca6966110f4ad98e87e5662eab416a17e7fd53c364bf8b954/frozenlist-1.7.0-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:cbb65198a9132ebc334f237d7b0df163e4de83fb4f2bdfe46c1e654bdb0c5d43", size = 45881, upload-time = "2025-06-09T23:01:12.296Z" },
    { url = "https://files.pythonhosted.org/packages/19/7c/71bb0bbe0832793c601fff68cd0cf6143753d0c667f9aec93d3c323f4b55/frozenlist-1.7.0-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:dab46c723eeb2c255a64f9dc05b8dd601fde66d6b19cdb82b2e09cc6ff8d8b5d", size = 232409, upload-time = "2025-06-09T23:01:13.641Z" },
    { url = "https://files.pythonhosted.org/packages/c0/45/ed2798718910fe6eb3ba574082aaceff4528e6323f9a8570be0f7028d8e9/frozenlist-1.7.0-cp313-cp313-manylinux_2_17_armv7l.manylinux2014_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:6aeac207a759d0dedd2e40745575ae32ab30926ff4fa49b1635def65806fddee", size = 225132, upload-time = "2025-06-09T23:01:15.264Z" },
    { url = "https://files.pythonhosted.org/packages/ba/e2/8417ae0f8eacb1d071d4950f32f229aa6bf68ab69aab797b72a07ea68d4f/frozenlist-1.7.0-cp313-cp313-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:bd8c4e58ad14b4fa7802b8be49d47993182fdd4023393899632c88fd8cd994eb", size = 237638, upload-time = "2025-06-09T23:01:16.752Z" },
    { url = "https://files.pythonhosted.org/packages/f8/b7/2ace5450ce85f2af05a871b8c8719b341294775a0a6c5585d5e6170f2ce7/frozenlist-1.7.0-cp313-cp313-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:04fb24d104f425da3540ed83cbfc31388a586a7696142004c577fa61c6298c3f", size = 233539, upload-time = "2025-06-09T23:01:18.202Z" },
    { url = "https://files.pythonhosted.org/packages/46/b9/6989292c5539553dba63f3c83dc4598186ab2888f67c0dc1d917e6887db6/frozenlist-1.7.0-cp313-cp313-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:6a5c505156368e4ea6b53b5ac23c92d7edc864537ff911d2fb24c140bb175e60", size = 215646, upload-time = "2025-06-09T23:01:19.649Z" },
    { url = "https://files.pythonhosted.org/packages/72/31/bc8c5c99c7818293458fe745dab4fd5730ff49697ccc82b554eb69f16a24/frozenlist-1.7.0-cp313-cp313-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:8bd7eb96a675f18aa5c553eb7ddc24a43c8c18f22e1f9925528128c052cdbe00", size = 232233, upload-time = "2025-06-09T23:01:21.175Z" },
    { url = "https://files.pythonhosted.org/packages/59/52/460db4d7ba0811b9ccb85af996019f5d70831f2f5f255f7cc61f86199795/frozenlist-1.7.0-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:05579bf020096fe05a764f1f84cd104a12f78eaab68842d036772dc6d4870b4b", size = 227996, upload-time = "2025-06-09T23:01:23.098Z" },
    { url = "https://files.pythonhosted.org/packages/ba/c9/f4b39e904c03927b7ecf891804fd3b4df3db29b9e487c6418e37988d6e9d/frozenlist-1.7.0-cp313-cp313-musllinux_1_2_armv7l.whl", hash = "sha256:376b6222d114e97eeec13d46c486facd41d4f43bab626b7c3f6a8b4e81a5192c", size = 242280, upload-time = "2025-06-09T23:01:24.808Z" },
    { url = "https://files.pythonhosted.org/packages/b8/33/3f8d6ced42f162d743e3517781566b8481322be321b486d9d262adf70bfb/frozenlist-1.7.0-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:0aa7e176ebe115379b5b1c95b4096fb1c17cce0847402e227e712c27bdb5a949", size = 217717, upload-time = "2025-06-09T23:01:26.28Z" },
    { url = "https://files.pythonhosted.org/packages/3e/e8/ad683e75da6ccef50d0ab0c2b2324b32f84fc88ceee778ed79b8e2d2fe2e/frozenlist-1.7.0-cp313-cp313-musllinux_1_2_ppc64le.whl", hash = "sha256:3fbba20e662b9c2130dc771e332a99eff5da078b2b2648153a40669a6d0e36ca", size = 236644, upload-time = "2025-06-09T23:01:27.887Z" },
    { url = "https://files.pythonhosted.org/packages/b2/14/8d19ccdd3799310722195a72ac94ddc677541fb4bef4091d8e7775752360/frozenlist-1.7.0-cp313-cp313-musllinux_1_2_s390x.whl", hash = "sha256:f3f4410a0a601d349dd406b5713fec59b4cee7e71678d5b17edda7f4655a940b", size = 238879, upload-time = "2025-06-09T23:01:29.524Z" },
    { url = "https://files.pythonhosted.org/packages/ce/13/c12bf657494c2fd1079a48b2db49fa4196325909249a52d8f09bc9123fd7/frozenlist-1.7.0-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:e2cdfaaec6a2f9327bf43c933c0319a7c429058e8537c508964a133dffee412e", size = 232502, upload-time = "2025-06-09T23:01:31.287Z" },
    { url = "https://files.pythonhosted.org/packages/d7/8b/e7f9dfde869825489382bc0d512c15e96d3964180c9499efcec72e85db7e/frozenlist-1.7.0-cp313-cp313-win32.whl", hash = "sha256:5fc4df05a6591c7768459caba1b342d9ec23fa16195e744939ba5914596ae3e1", size = 39169, upload-time = "2025-06-09T23:01:35.503Z" },
    { url = "https://files.pythonhosted.org/packages/35/89/a487a98d94205d85745080a37860ff5744b9820a2c9acbcdd9440bfddf98/frozenlist-1.7.0-cp313-cp313-win_amd64.whl", hash = "sha256:52109052b9791a3e6b5d1b65f4b909703984b770694d3eb64fad124c835d7cba", size = 43219, upload-time = "2025-06-09T23:01:36.784Z" },
    { url = "https://files.pythonhosted.org/packages/56/d5/5c4cf2319a49eddd9dd7145e66c4866bdc6f3dbc67ca3d59685149c11e0d/frozenlist-1.7.0-cp313-cp313t-macosx_10_13_universal2.whl", hash = "sha256:a6f86e4193bb0e235ef6ce3dde5cbabed887e0b11f516ce8a0f4d3b33078ec2d", size = 84345, upload-time = "2025-06-09T23:01:38.295Z" },
    { url = "https://files.pythonhosted.org/packages/a4/7d/ec2c1e1dc16b85bc9d526009961953df9cec8481b6886debb36ec9107799/frozenlist-1.7.0-cp313-cp313t-macosx_10_13_x86_64.whl", hash = "sha256:82d664628865abeb32d90ae497fb93df398a69bb3434463d172b80fc25b0dd7d", size = 48880, upload-time = "2025-06-09T23:01:39.887Z" },
    { url = "https://files.pythonhosted.org/packages/69/86/f9596807b03de126e11e7d42ac91e3d0b19a6599c714a1989a4e85eeefc4/frozenlist-1.7.0-cp313-cp313t-macosx_11_0_arm64.whl", hash = "sha256:912a7e8375a1c9a68325a902f3953191b7b292aa3c3fb0d71a216221deca460b", size = 48498, upload-time = "2025-06-09T23:01:41.318Z" },
    { url = "https://files.pythonhosted.org/packages/5e/cb/df6de220f5036001005f2d726b789b2c0b65f2363b104bbc16f5be8084f8/frozenlist-1.7.0-cp313-cp313t-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:9537c2777167488d539bc5de2ad262efc44388230e5118868e172dd4a552b146", size = 292296, upload-time = "2025-06-09T23:01:42.685Z" },
    { url = "https://files.pythonhosted.org/packages/83/1f/de84c642f17c8f851a2905cee2dae401e5e0daca9b5ef121e120e19aa825/frozenlist-1.7.0-cp313-cp313t-manylinux_2_17_armv7l.manylinux2014_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:f34560fb1b4c3e30ba35fa9a13894ba39e5acfc5f60f57d8accde65f46cc5e74", size = 273103, upload-time = "2025-06-09T23:01:44.166Z" },
    { url = "https://files.pythonhosted.org/packages/88/3c/c840bfa474ba3fa13c772b93070893c6e9d5c0350885760376cbe3b6c1b3/frozenlist-1.7.0-cp313-cp313t-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:acd03d224b0175f5a850edc104ac19040d35419eddad04e7cf2d5986d98427f1", size = 292869, upload-time = "2025-06-09T23:01:45.681Z" },
    { url = "https://files.pythonhosted.org/packages/a6/1c/3efa6e7d5a39a1d5ef0abeb51c48fb657765794a46cf124e5aca2c7a592c/frozenlist-1.7.0-cp313-cp313t-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:f2038310bc582f3d6a09b3816ab01737d60bf7b1ec70f5356b09e84fb7408ab1", size = 291467, upload-time = "2025-06-09T23:01:47.234Z" },
    { url = "https://files.pythonhosted.org/packages/4f/00/d5c5e09d4922c395e2f2f6b79b9a20dab4b67daaf78ab92e7729341f61f6/frozenlist-1.7.0-cp313-cp313t-manylinux_2_5_i686.manylinux1_i686.manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:b8c05e4c8e5f36e5e088caa1bf78a687528f83c043706640a92cb76cd6999384", size = 266028, upload-time = "2025-06-09T23:01:48.819Z" },
    { url = "https://files.pythonhosted.org/packages/4e/27/72765be905619dfde25a7f33813ac0341eb6b076abede17a2e3fbfade0cb/frozenlist-1.7.0-cp313-cp313t-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:765bb588c86e47d0b68f23c1bee323d4b703218037765dcf3f25c838c6fecceb", size = 284294, upload-time = "2025-06-09T23:01:50.394Z" },
    { url = "https://files.pythonhosted.org/packages/88/67/c94103a23001b17808eb7dd1200c156bb69fb68e63fcf0693dde4cd6228c/frozenlist-1.7.0-cp313-cp313t-musllinux_1_2_aarch64.whl", hash = "sha256:32dc2e08c67d86d0969714dd484fd60ff08ff81d1a1e40a77dd34a387e6ebc0c", size = 281898, upload-time = "2025-06-09T23:01:52.234Z" },
    { url = "https://files.pythonhosted.org/packages/42/34/a3e2c00c00f9e2a9db5653bca3fec306349e71aff14ae45ecc6d0951dd24/frozenlist-1.7.0-cp313-cp313t-musllinux_1_2_armv7l.whl", hash = "sha256:c0303e597eb5a5321b4de9c68e9845ac8f290d2ab3f3e2c864437d3c5a30cd65", size = 290465, upload-time = "2025-06-09T23:01:53.788Z" },
    { url = "https://files.pythonhosted.org/packages/bb/73/f89b7fbce8b0b0c095d82b008afd0590f71ccb3dee6eee41791cf8cd25fd/frozenlist-1.7.0-cp313-cp313t-musllinux_1_2_i686.whl", hash = "sha256:a47f2abb4e29b3a8d0b530f7c3598badc6b134562b1a5caee867f7c62fee51e3", size = 266385, upload-time = "2025-06-09T23:01:55.769Z" },
    { url = "https://files.pythonhosted.org/packages/cd/45/e365fdb554159462ca12df54bc59bfa7a9a273ecc21e99e72e597564d1ae/frozenlist-1.7.0-cp313-cp313t-musllinux_1_2_ppc64le.whl", hash = "sha256:3d688126c242a6fabbd92e02633414d40f50bb6002fa4cf995a1d18051525657", size = 288771, upload-time = "2025-06-09T23:01:57.4Z" },
    { url = "https://files.pythonhosted.org/packages/00/11/47b6117002a0e904f004d70ec5194fe9144f117c33c851e3d51c765962d0/frozenlist-1.7.0-cp313-cp313t-musllinux_1_2_s390x.whl", hash = "sha256:4e7e9652b3d367c7bd449a727dc79d5043f48b88d0cbfd4f9f1060cf2b414104", size = 288206, upload-time = "2025-06-09T23:01:58.936Z" },
    { url = "https://files.pythonhosted.org/packages/40/37/5f9f3c3fd7f7746082ec67bcdc204db72dad081f4f83a503d33220a92973/frozenlist-1.7.0-cp313-cp313t-musllinux_1_2_x86_64.whl", hash = "sha256:1a85e345b4c43db8b842cab1feb41be5cc0b10a1830e6295b69d7310f99becaf", size = 282620, upload-time = "2025-06-09T23:02:00.493Z" },
    { url = "https://files.pythonhosted.org/packages/0b/31/8fbc5af2d183bff20f21aa743b4088eac4445d2bb1cdece449ae80e4e2d1/frozenlist-1.7.0-cp313-cp313t-win32.whl", hash = "sha256:3a14027124ddb70dfcee5148979998066897e79f89f64b13328595c4bdf77c81", size = 43059, upload-time = "2025-06-09T23:02:02.072Z" },
    { url = "https://files.pythonhosted.org/packages/bb/ed/41956f52105b8dbc26e457c5705340c67c8cc2b79f394b79bffc09d0e938/frozenlist-1.7.0-cp313-cp313t-win_amd64.whl", hash = "sha256:3bf8010d71d4507775f658e9823210b7427be36625b387221642725b515dcf3e", size = 47516, upload-time = "2025-06-09T23:02:03.779Z" },
    { url = "https://files.pythonhosted.org/packages/ee/45/b82e3c16be2182bff01179db177fe144d58b5dc787a7d4492c6ed8b9317f/frozenlist-1.7.0-py3-none-any.whl", hash = "sha256:9a5af342e34f7e97caf8c995864c7a396418ae2859cc6fdf1b1073020d516a7e", size = 13106, upload-time = "2025-06-09T23:02:34.204Z" },
]

[[package]]
name = "fsspec"
version = "2025.5.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/00/f7/27f15d41f0ed38e8fcc488584b57e902b331da7f7c6dcda53721b15838fc/fsspec-2025.5.1.tar.gz", hash = "sha256:2e55e47a540b91843b755e83ded97c6e897fa0942b11490113f09e9c443c2475", size = 303033, upload-time = "2025-05-24T12:03:23.792Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/bb/61/78c7b3851add1481b048b5fdc29067397a1784e2910592bc81bb3f608635/fsspec-2025.5.1-py3-none-any.whl", hash = "sha256:24d3a2e663d5fc735ab256263c4075f374a174c3410c0b25e5bd1970bceaa462", size = 199052, upload-time = "2025-05-24T12:03:21.66Z" },
]

[[package]]
name = "ghp-import"
version = "2.1.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "python-dateutil" },
]
sdist = { url = "https://files.pythonhosted.org/packages/d9/29/d40217cbe2f6b1359e00c6c307bb3fc876ba74068cbab3dde77f03ca0dc4/ghp-import-2.1.0.tar.gz", hash = "sha256:9c535c4c61193c2df8871222567d7fd7e5014d835f97dc7b7439069e2413d343", size = 10943, upload-time = "2022-05-02T15:47:16.11Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/f7/ec/67fbef5d497f86283db54c22eec6f6140243aae73265799baaaa19cd17fb/ghp_import-2.1.0-py3-none-any.whl", hash = "sha256:8337dd7b50877f163d4c0289bc1f1c7f127550241988d568c1db512c4324a619", size = 11034, upload-time = "2022-05-02T15:47:14.552Z" },
]

[[package]]
name = "gitdb"
version = "4.0.12"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "smmap" },
]
sdist = { url = "https://files.pythonhosted.org/packages/72/94/63b0fc47eb32792c7ba1fe1b694daec9a63620db1e313033d18140c2320a/gitdb-4.0.12.tar.gz", hash = "sha256:5ef71f855d191a3326fcfbc0d5da835f26b13fbcba60c32c21091c349ffdb571", size = 394684, upload-time = "2025-01-02T07:20:46.413Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/a0/61/5c78b91c3143ed5c14207f463aecfc8f9dbb5092fb2869baf37c273b2705/gitdb-4.0.12-py3-none-any.whl", hash = "sha256:67073e15955400952c6565cc3e707c554a4eea2e428946f7a4c162fab9bd9bcf", size = 62794, upload-time = "2025-01-02T07:20:43.624Z" },
]

[[package]]
name = "gitpython"
version = "3.1.44"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "gitdb" },
]
sdist = { url = "https://files.pythonhosted.org/packages/c0/89/37df0b71473153574a5cdef8f242de422a0f5d26d7a9e231e6f169b4ad14/gitpython-3.1.44.tar.gz", hash = "sha256:c87e30b26253bf5418b01b0660f818967f3c503193838337fe5e573331249269", size = 214196, upload-time = "2025-01-02T07:32:43.59Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/1d/9a/4114a9057db2f1462d5c8f8390ab7383925fe1ac012eaa42402ad65c2963/GitPython-3.1.44-py3-none-any.whl", hash = "sha256:9e0e10cda9bed1ee64bc9a6de50e7e38a9c9943241cd7f585f6df3ed28011110", size = 207599, upload-time = "2025-01-02T07:32:40.731Z" },
]

[[package]]
name = "google-auth"
version = "2.40.3"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "cachetools" },
    { name = "pyasn1-modules" },
    { name = "rsa" },
]
sdist = { url = "https://files.pythonhosted.org/packages/9e/9b/e92ef23b84fa10a64ce4831390b7a4c2e53c0132568d99d4ae61d04c8855/google_auth-2.40.3.tar.gz", hash = "sha256:500c3a29adedeb36ea9cf24b8d10858e152f2412e3ca37829b3fa18e33d63b77", size = 281029, upload-time = "2025-06-04T18:04:57.577Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/17/63/b19553b658a1692443c62bd07e5868adaa0ad746a0751ba62c59568cd45b/google_auth-2.40.3-py2.py3-none-any.whl", hash = "sha256:1370d4593e86213563547f97a92752fc658456fe4514c809544f330fed45a7ca", size = 216137, upload-time = "2025-06-04T18:04:55.573Z" },
]

[[package]]
name = "googleapis-common-protos"
version = "1.70.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "protobuf" },
]
sdist = { url = "https://files.pythonhosted.org/packages/39/24/33db22342cf4a2ea27c9955e6713140fedd51e8b141b5ce5260897020f1a/googleapis_common_protos-1.70.0.tar.gz", hash = "sha256:0e1b44e0ea153e6594f9f394fef15193a68aaaea2d843f83e2742717ca753257", size = 145903, upload-time = "2025-04-14T10:17:02.924Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/86/f1/62a193f0227cf15a920390abe675f386dec35f7ae3ffe6da582d3ade42c7/googleapis_common_protos-1.70.0-py3-none-any.whl", hash = "sha256:b8bfcca8c25a2bb253e0e0b0adaf8c00773e5e6af6fd92397576680b807e0fd8", size = 294530, upload-time = "2025-04-14T10:17:01.271Z" },
]

[[package]]
name = "gpt4all"
version = "2.8.2"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "requests" },
    { name = "tqdm" },
]
wheels = [
    { url = "https://files.pythonhosted.org/packages/19/92/bad46a9bc5b993727045d09ce35e2dcc3939be02f47acfc99b2be0d33b33/gpt4all-2.8.2-py3-none-macosx_10_15_universal2.whl", hash = "sha256:41d6de60fe1f9c9a29bcbb585c5eb39083893cf00db0b836de1f2d88e2fbbb17", size = 6622150, upload-time = "2024-08-14T18:21:13.648Z" },
    { url = "https://files.pythonhosted.org/packages/93/60/ef1efd83c5757a69e8ed842b8a12b091e94a9bb2ceacc8b6e1c2e6487a77/gpt4all-2.8.2-py3-none-manylinux1_x86_64.whl", hash = "sha256:c61e977afa72475c076211fd9b59a88334fc0062615be8c2f7716363a21fbafa", size = 121614072, upload-time = "2024-08-14T18:21:21.736Z" },
    { url = "https://files.pythonhosted.org/packages/04/92/dd9dd077f0fc61218e4116a7154958eff36f06893506ad3290802218b1c9/gpt4all-2.8.2-py3-none-win_amd64.whl", hash = "sha256:a164674943df732808266e5bf63332fadef95eac802c201b47c7b378e5bd9f45", size = 119593619, upload-time = "2024-08-14T18:21:31.375Z" },
]

[[package]]
name = "grandalf"
version = "0.8"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "pyparsing" },
]
sdist = { url = "https://files.pythonhosted.org/packages/95/0e/4ac934b416857969f9135dec17ac80660634327e003a870835dd1f382659/grandalf-0.8.tar.gz", hash = "sha256:2813f7aab87f0d20f334a3162ccfbcbf085977134a17a5b516940a93a77ea974", size = 38128, upload-time = "2023-01-26T07:37:06.668Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/61/30/44c7eb0a952478dbb5f2f67df806686d6a7e4b19f6204e091c4f49dc7c69/grandalf-0.8-py3-none-any.whl", hash = "sha256:793ca254442f4a79252ea9ff1ab998e852c1e071b863593e5383afee906b4185", size = 41802, upload-time = "2023-01-10T15:16:19.753Z" },
]

[[package]]
name = "greenlet"
version = "3.2.3"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/c9/92/bb85bd6e80148a4d2e0c59f7c0c2891029f8fd510183afc7d8d2feeed9b6/greenlet-3.2.3.tar.gz", hash = "sha256:8b0dd8ae4c0d6f5e54ee55ba935eeb3d735a9b58a8a1e5b5cbab64e01a39f365", size = 185752, upload-time = "2025-06-05T16:16:09.955Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/fc/2e/d4fcb2978f826358b673f779f78fa8a32ee37df11920dc2bb5589cbeecef/greenlet-3.2.3-cp311-cp311-macosx_11_0_universal2.whl", hash = "sha256:784ae58bba89fa1fa5733d170d42486580cab9decda3484779f4759345b29822", size = 270219, upload-time = "2025-06-05T16:10:10.414Z" },
    { url = "https://files.pythonhosted.org/packages/16/24/929f853e0202130e4fe163bc1d05a671ce8dcd604f790e14896adac43a52/greenlet-3.2.3-cp311-cp311-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:0921ac4ea42a5315d3446120ad48f90c3a6b9bb93dd9b3cf4e4d84a66e42de83", size = 630383, upload-time = "2025-06-05T16:38:51.785Z" },
    { url = "https://files.pythonhosted.org/packages/d1/b2/0320715eb61ae70c25ceca2f1d5ae620477d246692d9cc284c13242ec31c/greenlet-3.2.3-cp311-cp311-manylinux2014_ppc64le.manylinux_2_17_ppc64le.whl", hash = "sha256:d2971d93bb99e05f8c2c0c2f4aa9484a18d98c4c3bd3c62b65b7e6ae33dfcfaf", size = 642422, upload-time = "2025-06-05T16:41:35.259Z" },
    { url = "https://files.pythonhosted.org/packages/bd/49/445fd1a210f4747fedf77615d941444349c6a3a4a1135bba9701337cd966/greenlet-3.2.3-cp311-cp311-manylinux2014_s390x.manylinux_2_17_s390x.whl", hash = "sha256:c667c0bf9d406b77a15c924ef3285e1e05250948001220368e039b6aa5b5034b", size = 638375, upload-time = "2025-06-05T16:48:18.235Z" },
    { url = "https://files.pythonhosted.org/packages/7e/c8/ca19760cf6eae75fa8dc32b487e963d863b3ee04a7637da77b616703bc37/greenlet-3.2.3-cp311-cp311-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:592c12fb1165be74592f5de0d70f82bc5ba552ac44800d632214b76089945147", size = 637627, upload-time = "2025-06-05T16:13:02.858Z" },
    { url = "https://files.pythonhosted.org/packages/65/89/77acf9e3da38e9bcfca881e43b02ed467c1dedc387021fc4d9bd9928afb8/greenlet-3.2.3-cp311-cp311-manylinux_2_24_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:29e184536ba333003540790ba29829ac14bb645514fbd7e32af331e8202a62a5", size = 585502, upload-time = "2025-06-05T16:12:49.642Z" },
    { url = "https://files.pythonhosted.org/packages/97/c6/ae244d7c95b23b7130136e07a9cc5aadd60d59b5951180dc7dc7e8edaba7/greenlet-3.2.3-cp311-cp311-musllinux_1_1_aarch64.whl", hash = "sha256:93c0bb79844a367782ec4f429d07589417052e621aa39a5ac1fb99c5aa308edc", size = 1114498, upload-time = "2025-06-05T16:36:46.598Z" },
    { url = "https://files.pythonhosted.org/packages/89/5f/b16dec0cbfd3070658e0d744487919740c6d45eb90946f6787689a7efbce/greenlet-3.2.3-cp311-cp311-musllinux_1_1_x86_64.whl", hash = "sha256:751261fc5ad7b6705f5f76726567375bb2104a059454e0226e1eef6c756748ba", size = 1139977, upload-time = "2025-06-05T16:12:38.262Z" },
    { url = "https://files.pythonhosted.org/packages/66/77/d48fb441b5a71125bcac042fc5b1494c806ccb9a1432ecaa421e72157f77/greenlet-3.2.3-cp311-cp311-win_amd64.whl", hash = "sha256:83a8761c75312361aa2b5b903b79da97f13f556164a7dd2d5448655425bd4c34", size = 297017, upload-time = "2025-06-05T16:25:05.225Z" },
    { url = "https://files.pythonhosted.org/packages/f3/94/ad0d435f7c48debe960c53b8f60fb41c2026b1d0fa4a99a1cb17c3461e09/greenlet-3.2.3-cp312-cp312-macosx_11_0_universal2.whl", hash = "sha256:25ad29caed5783d4bd7a85c9251c651696164622494c00802a139c00d639242d", size = 271992, upload-time = "2025-06-05T16:11:23.467Z" },
    { url = "https://files.pythonhosted.org/packages/93/5d/7c27cf4d003d6e77749d299c7c8f5fd50b4f251647b5c2e97e1f20da0ab5/greenlet-3.2.3-cp312-cp312-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:88cd97bf37fe24a6710ec6a3a7799f3f81d9cd33317dcf565ff9950c83f55e0b", size = 638820, upload-time = "2025-06-05T16:38:52.882Z" },
    { url = "https://files.pythonhosted.org/packages/c6/7e/807e1e9be07a125bb4c169144937910bf59b9d2f6d931578e57f0bce0ae2/greenlet-3.2.3-cp312-cp312-manylinux2014_ppc64le.manylinux_2_17_ppc64le.whl", hash = "sha256:baeedccca94880d2f5666b4fa16fc20ef50ba1ee353ee2d7092b383a243b0b0d", size = 653046, upload-time = "2025-06-05T16:41:36.343Z" },
    { url = "https://files.pythonhosted.org/packages/9d/ab/158c1a4ea1068bdbc78dba5a3de57e4c7aeb4e7fa034320ea94c688bfb61/greenlet-3.2.3-cp312-cp312-manylinux2014_s390x.manylinux_2_17_s390x.whl", hash = "sha256:be52af4b6292baecfa0f397f3edb3c6092ce071b499dd6fe292c9ac9f2c8f264", size = 647701, upload-time = "2025-06-05T16:48:19.604Z" },
    { url = "https://files.pythonhosted.org/packages/cc/0d/93729068259b550d6a0288da4ff72b86ed05626eaf1eb7c0d3466a2571de/greenlet-3.2.3-cp312-cp312-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:0cc73378150b8b78b0c9fe2ce56e166695e67478550769536a6742dca3651688", size = 649747, upload-time = "2025-06-05T16:13:04.628Z" },
    { url = "https://files.pythonhosted.org/packages/f6/f6/c82ac1851c60851302d8581680573245c8fc300253fc1ff741ae74a6c24d/greenlet-3.2.3-cp312-cp312-manylinux_2_24_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:706d016a03e78df129f68c4c9b4c4f963f7d73534e48a24f5f5a7101ed13dbbb", size = 605461, upload-time = "2025-06-05T16:12:50.792Z" },
    { url = "https://files.pythonhosted.org/packages/98/82/d022cf25ca39cf1200650fc58c52af32c90f80479c25d1cbf57980ec3065/greenlet-3.2.3-cp312-cp312-musllinux_1_1_aarch64.whl", hash = "sha256:419e60f80709510c343c57b4bb5a339d8767bf9aef9b8ce43f4f143240f88b7c", size = 1121190, upload-time = "2025-06-05T16:36:48.59Z" },
    { url = "https://files.pythonhosted.org/packages/f5/e1/25297f70717abe8104c20ecf7af0a5b82d2f5a980eb1ac79f65654799f9f/greenlet-3.2.3-cp312-cp312-musllinux_1_1_x86_64.whl", hash = "sha256:93d48533fade144203816783373f27a97e4193177ebaaf0fc396db19e5d61163", size = 1149055, upload-time = "2025-06-05T16:12:40.457Z" },
    { url = "https://files.pythonhosted.org/packages/1f/8f/8f9e56c5e82eb2c26e8cde787962e66494312dc8cb261c460e1f3a9c88bc/greenlet-3.2.3-cp312-cp312-win_amd64.whl", hash = "sha256:7454d37c740bb27bdeddfc3f358f26956a07d5220818ceb467a483197d84f849", size = 297817, upload-time = "2025-06-05T16:29:49.244Z" },
    { url = "https://files.pythonhosted.org/packages/b1/cf/f5c0b23309070ae93de75c90d29300751a5aacefc0a3ed1b1d8edb28f08b/greenlet-3.2.3-cp313-cp313-macosx_11_0_universal2.whl", hash = "sha256:500b8689aa9dd1ab26872a34084503aeddefcb438e2e7317b89b11eaea1901ad", size = 270732, upload-time = "2025-06-05T16:10:08.26Z" },
    { url = "https://files.pythonhosted.org/packages/48/ae/91a957ba60482d3fecf9be49bc3948f341d706b52ddb9d83a70d42abd498/greenlet-3.2.3-cp313-cp313-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:a07d3472c2a93117af3b0136f246b2833fdc0b542d4a9799ae5f41c28323faef", size = 639033, upload-time = "2025-06-05T16:38:53.983Z" },
    { url = "https://files.pythonhosted.org/packages/6f/df/20ffa66dd5a7a7beffa6451bdb7400d66251374ab40b99981478c69a67a8/greenlet-3.2.3-cp313-cp313-manylinux2014_ppc64le.manylinux_2_17_ppc64le.whl", hash = "sha256:8704b3768d2f51150626962f4b9a9e4a17d2e37c8a8d9867bbd9fa4eb938d3b3", size = 652999, upload-time = "2025-06-05T16:41:37.89Z" },
    { url = "https://files.pythonhosted.org/packages/51/b4/ebb2c8cb41e521f1d72bf0465f2f9a2fd803f674a88db228887e6847077e/greenlet-3.2.3-cp313-cp313-manylinux2014_s390x.manylinux_2_17_s390x.whl", hash = "sha256:5035d77a27b7c62db6cf41cf786cfe2242644a7a337a0e155c80960598baab95", size = 647368, upload-time = "2025-06-05T16:48:21.467Z" },
    { url = "https://files.pythonhosted.org/packages/8e/6a/1e1b5aa10dced4ae876a322155705257748108b7fd2e4fae3f2a091fe81a/greenlet-3.2.3-cp313-cp313-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:2d8aa5423cd4a396792f6d4580f88bdc6efcb9205891c9d40d20f6e670992efb", size = 650037, upload-time = "2025-06-05T16:13:06.402Z" },
    { url = "https://files.pythonhosted.org/packages/26/f2/ad51331a157c7015c675702e2d5230c243695c788f8f75feba1af32b3617/greenlet-3.2.3-cp313-cp313-manylinux_2_24_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:2c724620a101f8170065d7dded3f962a2aea7a7dae133a009cada42847e04a7b", size = 608402, upload-time = "2025-06-05T16:12:51.91Z" },
    { url = "https://files.pythonhosted.org/packages/26/bc/862bd2083e6b3aff23300900a956f4ea9a4059de337f5c8734346b9b34fc/greenlet-3.2.3-cp313-cp313-musllinux_1_1_aarch64.whl", hash = "sha256:873abe55f134c48e1f2a6f53f7d1419192a3d1a4e873bace00499a4e45ea6af0", size = 1119577, upload-time = "2025-06-05T16:36:49.787Z" },
    { url = "https://files.pythonhosted.org/packages/86/94/1fc0cc068cfde885170e01de40a619b00eaa8f2916bf3541744730ffb4c3/greenlet-3.2.3-cp313-cp313-musllinux_1_1_x86_64.whl", hash = "sha256:024571bbce5f2c1cfff08bf3fbaa43bbc7444f580ae13b0099e95d0e6e67ed36", size = 1147121, upload-time = "2025-06-05T16:12:42.527Z" },
