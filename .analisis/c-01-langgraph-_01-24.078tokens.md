# Consolidated Repository Snapshot



## langgraph @ 0d4ac836e3a943a6a807910001f84a1d11b52776
Source: https://github.com/langchain-ai/langgraph

https://github.com/langchain-ai/langgraph/blob/main/.gitignore
```text
.vs/
.vscode/
.idea/
# Byte-compiled / optimized / DLL files
__pycache__/
*.py[cod]
*$py.class

# C extensions
*.so

# Distribution / packaging
.Python
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
pip-wheel-metadata/
share/python-wheels/
*.egg-info/
.installed.cfg
*.egg
MANIFEST

# PyInstaller
#  Usually these files are written by a python script from a template
#  before PyInstaller builds the exe, so as to inject date/other infos into it.
*.manifest
*.spec

# Installer logs
pip-log.txt
pip-delete-this-directory.txt

# Unit test / coverage reports
htmlcov/
.tox/
.nox/
.coverage
.coverage.*
.cache
nosetests.xml
coverage.xml
*.cover
*.py,cover
.hypothesis/
.pytest_cache/

# Translations
*.mo
*.pot

# Django stuff:
*.log
local_settings.py
db.sqlite3
db.sqlite3-journal

# Flask stuff:
instance/
.webassets-cache

# Scrapy stuff:
.scrapy

# Sphinx documentation
docs/_build/
docs/docs/_build/

# PyBuilder
target/

# Jupyter Notebook
.ipynb_checkpoints
notebooks/

# IPython
profile_default/
ipython_config.py

# pyenv
.python-version

# pipenv
#   According to pypa/pipenv#598, it is recommended to include Pipfile.lock in version control.
#   However, in case of collaboration, if having platform-specific dependencies or dependencies
#   having no cross-platform support, pipenv may install dependencies that don't work, or not
#   install all needed dependencies.
#Pipfile.lock

# PEP 582; used by e.g. github.com/David-OConnor/pyflow
__pypackages__/

# Celery stuff
celerybeat-schedule
celerybeat.pid

# SageMath parsed files
*.sage.py

# Environments
.env
.envrc
.venv
.venvs
env/
venv/
ENV/
env.bak/
venv.bak/

# Spyder project settings
.spyderproject
.spyproject

# Rope project settings
.ropeproject

# mkdocs documentation
/site

# mypy
.mypy_cache/
.dmypy.json
dmypy.json

# Pyre type checker
.pyre/

# macOS display setting files
.DS_Store

# Wandb directory
wandb/

# asdf tool versions
.tool-versions
/.ruff_cache/

*.pkl
*.bin

# integration test artifacts
data_map*
\[('_type', 'fake'), ('stop', None)]

# Replit files
*replit*

node_modules
docs/.yarn/
docs/node_modules/
docs/.docusaurus/
docs/.cache-loader/
docs/_dist
docs/api_reference/api_reference.rst
docs/api_reference/experimental_api_reference.rst
docs/api_reference/_build
docs/api_reference/*/
!docs/api_reference/_static/
!docs/api_reference/templates/
!docs/api_reference/themes/
docs/docs_skeleton/build
docs/docs_skeleton/node_modules
docs/docs_skeleton/yarn.lock

# Any new jupyter notebooks
# not intended for the repo
Untitled*.ipynb

Chinook.db

.vercel
.turbo
.editorconfig
.scratch

```
---
https://github.com/langchain-ai/langgraph/blob/main/.markdown-link-check.config.json
```json
{
  "aliveStatusCodes": [200, 206, 402],
  "ignorePatterns": ["*dcbadge.vercel.app*"]
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/AGENTS.md
```md
# AGENTS Instructions

This repository is a monorepo. Each library lives in a subdirectory under `libs/`.

When you modify code in any library, run the following commands in that library's directory before creating a pull request:

- `make format` – run code formatters
- `make lint` – run the linter
- `make test` – execute the test suite

To run a particular test file or to pass additional pytest options you can specify the `TEST` variable:

```
TEST=path/to/test.py make test
```

Other pytest arguments can also be supplied inside the `TEST` variable.

## Libraries

The repository contains several Python and JavaScript/TypeScript libraries.
Below is a high-level overview:

- **checkpoint** – base interfaces for LangGraph checkpointers.
- **checkpoint-postgres** – Postgres implementation of the checkpoint saver.
- **checkpoint-sqlite** – SQLite implementation of the checkpoint saver.
- **cli** – official command-line interface for LangGraph.
- **langgraph** – core framework for building stateful, multi-actor agents.
- **prebuilt** – high-level APIs for creating and running agents and tools.
- **sdk-js** – JS/TS SDK for interacting with the LangGraph REST API.
- **sdk-py** – Python SDK for the LangGraph Server API.

### Dependency map

The diagram below lists downstream libraries for each production dependency as
declared in that library's `pyproject.toml` (or `package.json`).

```text
checkpoint
├── checkpoint-postgres
├── checkpoint-sqlite
├── prebuilt
└── langgraph

prebuilt
└── langgraph

sdk-py
├── langgraph
└── cli

sdk-js (standalone)
```

Changes to a library may impact all of its dependents shown above.

```
---
https://github.com/langchain-ai/langgraph/blob/main/CONTRIBUTING.md
```md
# Contributing to LangGraph

Thank you for being interested in contributing to LangGraph!

## General guidelines

Here are some things to keep in mind for all types of contributions:

- Follow the ["fork and pull request"](https://docs.github.com/en/get-started/exploring-projects-on-github/contributing-to-a-project) workflow.
- Fill out the checked-in pull request template when opening pull requests. Note related issues and tag relevant maintainers.
- Ensure your PR passes formatting, linting, and testing checks before requesting a review.
  - If you would like comments or feedback, please tag a maintainer.
- Backwards compatibility is key. Your changes must not be breaking, except in case of critical bug and security fixes.
- Look for duplicate PRs or issues that have already been opened before opening a new one.
- Keep scope as isolated as possible. As a general rule, your changes should not affect more than one package at a time.

### Bugfixes

For bug fixes, please open up an issue before proposing a fix to ensure the proposal properly addresses the underlying problem. In general, bug fixes should all have an accompanying unit test that fails before the fix.

### New features

For new features, please start a new [discussion](https://forum.langchain.com/), where the maintainers will help with scoping out the necessary changes.

## Contribute Documentation

Documentation is a vital part of LangGraph. We welcome both new documentation for new features and
community improvements to our current documentation. Please read the resources below before getting started:

- [Documentation style guide](#documentation-style-guide)
- [Documentation setup](#setup)

## Documentation Style Guide

As LangGraph continues to grow, the surface area of documentation required to cover it continues to grow too.
This page provides guidelines for anyone writing documentation for LangGraph, as well as some of our philosophies around organization and structure.

## Philosophy

LangGraph's documentation follows the [Diataxis framework](https://diataxis.fr).
Under this framework, all documentation falls under one of four categories: [Tutorials](#tutorials),
[How-to guides](#how-to-guides),
[References](#references), and [Explanations (aka conceptual guides)](#conceptual-guide).

### Tutorials

Tutorials are lessons that take the reader through a practical activity. Their purpose is to help the user
gain understanding of concepts and how they interact by showing one way to achieve some goal in a hands-on way.

They should **avoid** giving
multiple permutations of ways to achieve that goal in-depth. Choice is burdensome. Instead, they should guide a new user through a recommended path to accomplishing a concrete goal. While the end result of a tutorial does not necessarily need to
be completely production-ready, it should be useful and practically satisfy the goal that you clearly stated in the tutorial's introduction.

To quote the Diataxis website:

> A tutorial serves the user’s *acquisition* of skills and knowledge - their study. Its purpose is not to help the user get something done, but to help them learn.

In LangGraph, these are often higher level guides that show off end-to-end use cases.

Some examples include:

- [Build a Customer Support Bot](https://langchain-ai.github.io/langgraph/tutorials/customer-support/customer-support/)
- [Build a SQL Agent](https://langchain-ai.github.io/langgraph/tutorials/sql/sql-agent/)

Here are some high-level tips on writing a good tutorial:

- Focus on guiding the user to get something done, but keep in mind the end-goal is more to impart principles than to create a perfect production system.
- Be specific, not abstract and follow one path.
  - No need to go deeply into alternative approaches, but it’s ok to reference them, ideally with a link to an appropriate how-to guide.
- Get "a point on the board" as soon as possible - something the user can run that outputs something.
  - You can iterate and expand afterwards.
  - Try to frequently checkpoint at given steps where the user can run code and see progress.
- Focus on results, not technical explanation.
  - Crosslink heavily to appropriate conceptual/reference pages
- The first time you mention a LangGraph concept, use its full name (e.g. "human-in-the-loop"), and link to its conceptual/other documentation page.
  - It's also helpful to add a prerequisite callout that links to any pages with necessary background information.
- End with a recap/next steps section summarizing what the tutorial covered and future reading, such as related how-to guides.
- Use phrases like "Next we can run X & Y. We will expect Z.". Then afterwards, use language like "Notice Z" that recalls our expectations and directs the reader's attention to the topic we are trying to teach.
- Do not shy away from repetition.

### How-to guides

A how-to guide, as the name implies, demonstrates how to do something discrete and specific.
It should assume that the user is already familiar with underlying concepts, and is trying to solve an immediate problem, but
should still give some background or list the scenarios where the information contained within can be relevant.
They can and should discuss alternatives if one approach may be better than another in certain cases.

To quote the Diataxis website:

> A how-to guide serves the work of the already-competent user, whom you can assume to know what they want to do, and to be able to follow your instructions correctly.

Some examples include:

- [How to add persistence to your graph](https://langchain-ai.github.io/langgraph/how-tos/persistence/)
- [How to view and update past graph state](https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/time-travel/)

Here are some high-level tips on writing a good how-to guide:

- Clearly explain what you are guiding the user through at the start
- Assume higher intent than a tutorial and show what the user needs to do to get that task done
- Assume familiarity of concepts, but explain why suggested actions are helpful
  - Crosslink heavily to conceptual/reference pages
- Discuss alternatives and responses to real-world tradeoffs that may arise when solving a problem
- Use lots of example code, ideally within complete code blocks that the reader can copy and run.
- End with a recap/next steps section summarizing what the tutorial covered and future reading, such as other related how-to guides

### Conceptual guides

LangGraph's conceptual guides fall under the **Explanation** quadrant of Diataxis. They should cover LangChain terms and concepts
in a more abstract way than how-to guides or tutorials, and should be geared towards curious users interested in
gaining a deeper understanding of the framework. Try to avoid excessively large code examples. The goal here is to
impart perspective to the user rather than to finish a practical project. These guides should cover **why** things work the way they do.

To quote the Diataxis website:

> The perspective of explanation is higher and wider than that of the other types. It does not take the user’s eye-level view, as in a how-to guide, or a close-up view of the machinery, like reference material. Its scope in each case is a topic - “an area of knowledge”, that somehow has to be bounded in a reasonable, meaningful way.

Some examples include:

- [What does it mean to be agentic?](https://langchain-ai.github.io/langgraph/concepts/high_level/)
- [Tool calling](https://langchain-ai.github.io/langgraph/concepts/agentic_concepts/#tool-calling)

Here are some high-level tips on writing a good conceptual guide:

- Explain design decisions. Why does concept X exist and why was it designed this way?
- Use analogies and reference other concepts and alternatives
- Avoid blending in too much reference content
- You can and should reference content covered in other guides, but make sure to link to them

### References

References contain detailed, low-level information that describes exactly what functionality exists and how to use it.
In LangGraph, this is mainly our API reference pages, which are populated from docstrings within code.
References pages are generally not read end-to-end, but are consulted as necessary when a user needs to know
how to use something specific.

To quote the Diataxis website:

> The only purpose of a reference guide is to describe, as succinctly as possible, and in an orderly way. Whereas the content of tutorials and how-to guides are led by needs of the user, reference material is led by the product it describes.

Many of the reference pages in LangChain are automatically generated from code,
but here are some high-level tips on writing a good docstring:

- Be concise
- Discuss special cases and deviations from a user's expectations
- Go into detail on required inputs and outputs
- Light details on when one might use the feature are fine, but in-depth details belong in other sections.

Each category serves a distinct purpose and requires a specific approach to writing and structuring the content.

## General guidelines

Here are some other guidelines you should think about when writing and organizing documentation.

We generally do not merge new tutorials from outside contributors without an actual need.
We welcome updates as well as new integration docs, how-tos, and references.

### Avoid duplication

Multiple pages that cover the same material in depth are difficult to maintain and cause confusion. There should
be only one (very rarely two), canonical pages for a given concept or feature. Instead, you should link to other guides.

### Link to other sections

Because sections of the docs do not exist in a vacuum, it is important to link to other sections as often as possible
to allow a developer to learn more about an unfamiliar topic inline.

This includes linking to the API references as well as conceptual sections!

### Be concise

In general, take a less-is-more approach. If a section with a good explanation of a concept already exists, you should link to it rather than
re-explain it, unless the concept you are documenting presents some new wrinkle.

Be concise, including in code samples.

### General style

- Use active voice and present tense whenever possible
- Use examples and code snippets to illustrate concepts and usage
- Use appropriate header levels (`#`, `##`, `###`, etc.) to organize the content hierarchically
- Use fewer cells with more code to make copy/paste easier
- Use bullet points and numbered lists to break down information into easily digestible chunks
- Use tables (especially for **Reference** sections) and diagrams often to present information visually
- Include the table of contents for longer documentation pages to help readers navigate the content, but hide it for shorter pages

## Setup

LangGraph documentation consists of two components:

1. Main Documentation: Hosted at [https://langchain-ai.github.io/langgraph/](https://langchain-ai.github.io/langgraph/),
this comprehensive resource serves as the primary user-facing documentation.
It covers a wide array of topics, including tutorials, use cases, integrations,
and more, offering extensive guidance on building with LangGraph.
The content for this documentation lives in the `/docs` directory of the monorepo.
2. In-code Documentation: This is documentation of the codebase itself, which is also
used to generate the externally facing [API Reference](https://langchain-ai.github.io/langgraph/reference/graphs/).
The content for the API reference is autogenerated by scanning the docstrings in the codebase. For this reason we ask that developers document their code well.

We appreciate all contributions to the documentation, whether it be fixing a typo,
adding a new tutorial or example and whether it be in the main documentation or the API Reference.

### 📜 Main Documentation

The content for the main documentation is located in the `/docs` directory of the monorepo.

The documentation is written using a combination of ipython notebooks (`.ipynb` files)
and markdown (`.md` files). The notebooks are converted to markdown
and then built using [MkDocs](https://www.mkdocs.org/).

Feel free to make contributions to the main documentation! 🥰

After modifying the documentation:

1. Run the linting and formatting commands (see below) to ensure that the documentation is well-formatted and free of errors.
2. Optionally build the documentation locally to verify that the changes look good.
3. Make a pull request with the changes.

### ⚒️ Linting and Building Documentation Locally

After writing up the documentation, you may want to lint and build the documentation
locally to ensure that it looks good and is free of errors.

If you're unable to build it locally that's okay as well, as you will be able to
see a preview of the documentation on the pull request page.

From the **monorepo root**, run the following command to install the dependencies:

<!-- TODO -->
```bash
poetry install --with docs --no-root
```

#### Building

The code that builds the documentation is located in the `/docs` directory of the monorepo.

Before building the documentation, it is always a good idea to clean the build directory:

```bash
make clean-docs
```

You can build and preview the documentation as outlined below:

```bash
make serve-docs
```

#### Linting

To spell check the docs, run the following from the `docs` directory:

```bash
codespell --skip="*.ambr,*.lock,*.ipynb,*.yaml,*.zlib,*.css.map,*.js.map" --ignore-words-list="infor,thead,stdio,nd,jupyter,lets,lite,uis,deque" .
```

### ️In-code Documentation

The in-code documentation is autogenerated from docstrings.

For the API reference to be useful, the codebase must be well-documented. This means that all functions, classes, and methods should have a docstring that explains what they do, what the arguments are, and what the return value is. This is a good practice in general, but it is especially important for LangGraph because the API reference is the primary resource for developers to understand how to use the codebase.

We generally follow the [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings) for docstrings.

Here is an example of a well-documented function:

```python

def my_function(arg1: int, arg2: str) -> float:
    """This is a short description of the function. (It should be a single sentence.)

    This is a longer description of the function. It should explain what
    the function does, what the arguments are, and what the return value is.
    It should wrap at 88 characters.

    Examples:
        This is a section for examples of how to use the function.

        ```python
        my_function(1, "hello")
        \```

    Args:
        arg1: This is a description of arg1. We do not need to specify the type since
            it is already specified in the function signature.
        arg2: This is a description of arg2.

    Returns:
        This is a description of the return value.
    """
    return 3.14
```

```
---
https://github.com/langchain-ai/langgraph/blob/main/LICENSE
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
https://github.com/langchain-ai/langgraph/blob/main/Makefile
```text
# Define the directories containing projects
LIBS_DIRS := $(wildcard libs/*)

# Default target
.PHONY: all
all: lint format lock test

# Install dependencies for all projects
.PHONY: install
install:
	@echo "Creating virtual environment..."
	@uv venv
	@for dir in $(LIBS_DIRS); do \
		if [ -f $$dir/pyproject.toml ]; then \
			echo "Installing dependencies for $$dir"; \
			uv pip install -e $$dir; \
		fi; \
	done

# Lint all projects
.PHONY: lint
lint:
	@for dir in $(LIBS_DIRS); do \
		if [ -f $$dir/Makefile ]; then \
			echo "Running lint in $$dir"; \
			$(MAKE) -C $$dir lint; \
		fi; \
	done

# Format all projects
.PHONY: format
format:
	@for dir in $(LIBS_DIRS); do \
		if [ -f $$dir/Makefile ]; then \
			echo "Running format in $$dir"; \
			$(MAKE) -C $$dir format; \
		fi; \
	done

# Lock all projects
.PHONY: lock
lock:
	@for dir in $(LIBS_DIRS); do \
		if [ -f $$dir/Makefile ]; then \
			echo "Running lock in $$dir"; \
			(cd $$dir && uv lock); \
		fi; \
	done

# Lock all projects and upgrade dependencies
.PHONY: lock-upgrade
lock-upgrade:
	@for dir in $(LIBS_DIRS); do \
		if [ -f $$dir/Makefile ]; then \
			echo "Running lock-upgrade in $$dir"; \
			(cd $$dir && uv lock --upgrade); \
		fi; \
	done

# Test all projects
.PHONY: test
test:
	@for dir in $(LIBS_DIRS); do \
		if [ -f $$dir/Makefile ]; then \
			echo "Running test in $$dir"; \
			$(MAKE) -C $$dir test; \
		fi; \
	done
```
---
https://github.com/langchain-ai/langgraph/blob/main/README.md
```md
<picture class="github-only">
  <source media="(prefers-color-scheme: light)" srcset="https://langchain-ai.github.io/langgraph/static/wordmark_dark.svg">
  <source media="(prefers-color-scheme: dark)" srcset="https://langchain-ai.github.io/langgraph/static/wordmark_light.svg">
  <img alt="LangGraph Logo" src="https://langchain-ai.github.io/langgraph/static/wordmark_dark.svg" width="80%">
</picture>

<div>
<br>
</div>

[![Version](https://img.shields.io/pypi/v/langgraph.svg)](https://pypi.org/project/langgraph/)
[![Downloads](https://static.pepy.tech/badge/langgraph/month)](https://pepy.tech/project/langgraph)
[![Open Issues](https://img.shields.io/github/issues-raw/langchain-ai/langgraph)](https://github.com/langchain-ai/langgraph/issues)
[![Docs](https://img.shields.io/badge/docs-latest-blue)](https://langchain-ai.github.io/langgraph/)

Trusted by companies shaping the future of agents – including Klarna, Replit, Elastic, and more – LangGraph is a low-level orchestration framework for building, managing, and deploying long-running, stateful agents.

## Get started

Install LangGraph:

```
pip install -U langgraph
```

Then, create an agent [using prebuilt components](https://langchain-ai.github.io/langgraph/agents/agents/):

```python
# pip install -qU "langchain[anthropic]" to call the model

from langgraph.prebuilt import create_react_agent

def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[get_weather],
    prompt="You are a helpful assistant"
)

# Run the agent
agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)
```

For more information, see the [Quickstart](https://langchain-ai.github.io/langgraph/agents/agents/). Or, to learn how to build an [agent workflow](https://langchain-ai.github.io/langgraph/concepts/low_level/) with a customizable architecture, long-term memory, and other complex task handling, see the [LangGraph basics tutorials](https://langchain-ai.github.io/langgraph/tutorials/get-started/1-build-basic-chatbot/).

## Core benefits

LangGraph provides low-level supporting infrastructure for *any* long-running, stateful workflow or agent. LangGraph does not abstract prompts or architecture, and provides the following central benefits:

- [Durable execution](https://langchain-ai.github.io/langgraph/concepts/durable_execution/): Build agents that persist through failures and can run for extended periods, automatically resuming from exactly where they left off.
- [Human-in-the-loop](https://langchain-ai.github.io/langgraph/concepts/human_in_the_loop/): Seamlessly incorporate human oversight by inspecting and modifying agent state at any point during execution.
- [Comprehensive memory](https://langchain-ai.github.io/langgraph/concepts/memory/): Create truly stateful agents with both short-term working memory for ongoing reasoning and long-term persistent memory across sessions.
- [Debugging with LangSmith](http://www.langchain.com/langsmith): Gain deep visibility into complex agent behavior with visualization tools that trace execution paths, capture state transitions, and provide detailed runtime metrics.
- [Production-ready deployment](https://langchain-ai.github.io/langgraph/concepts/deployment_options/): Deploy sophisticated agent systems confidently with scalable infrastructure designed to handle the unique challenges of stateful, long-running workflows.

## LangGraph’s ecosystem

While LangGraph can be used standalone, it also integrates seamlessly with any LangChain product, giving developers a full suite of tools for building agents. To improve your LLM application development, pair LangGraph with:

- [LangSmith](http://www.langchain.com/langsmith) — Helpful for agent evals and observability. Debug poor-performing LLM app runs, evaluate agent trajectories, gain visibility in production, and improve performance over time.
- [LangSmith Deployment](https://langchain-ai.github.io/langgraph/concepts/langgraph_platform/) — Deploy and scale agents effortlessly with a purpose-built deployment platform for long running, stateful workflows. Discover, reuse, configure, and share agents across teams — and iterate quickly with visual prototyping in [LangGraph Studio](https://langchain-ai.github.io/langgraph/concepts/langgraph_studio/).
- [LangChain](https://docs.langchain.com/oss/python/langchain/overview) – Provides integrations and composable components to streamline LLM application development.

> [!NOTE]
> Looking for the JS version of LangGraph? See the [JS repo](https://github.com/langchain-ai/langgraphjs) and the [JS docs](https://langchain-ai.github.io/langgraphjs/).

## Additional resources

- [Guides](https://langchain-ai.github.io/langgraph/guides/): Quick, actionable code snippets for topics such as streaming, adding memory & persistence, and design patterns (e.g. branching, subgraphs, etc.).
- [Reference](https://langchain-ai.github.io/langgraph/reference/graphs/): Detailed reference on core classes, methods, how to use the graph and checkpointing APIs, and higher-level prebuilt components.
- [Examples](https://langchain-ai.github.io/langgraph/examples/): Guided examples on getting started with LangGraph.
- [LangChain Forum](https://forum.langchain.com/): Connect with the community and share all of your technical questions, ideas, and feedback.
- [LangChain Academy](https://academy.langchain.com/courses/intro-to-langgraph): Learn the basics of LangGraph in our free, structured course.
- [Templates](https://langchain-ai.github.io/langgraph/concepts/template_applications/): Pre-built reference apps for common agentic workflows (e.g. ReAct agent, memory, retrieval etc.) that can be cloned and adapted.
- [Case studies](https://www.langchain.com/built-with-langgraph): Hear how industry leaders use LangGraph to ship AI applications at scale.

## Acknowledgements

LangGraph is inspired by [Pregel](https://research.google/pubs/pub37252/) and [Apache Beam](https://beam.apache.org/). The public interface draws inspiration from [NetworkX](https://networkx.org/documentation/latest/). LangGraph is built by LangChain Inc, the creators of LangChain, but can be used without LangChain.

```
---
https://github.com/langchain-ai/langgraph/blob/main/security.md
```md
# Security Policy

## Reporting OSS Vulnerabilities

LangChain is partnered with [huntr by Protect AI](https://huntr.com/) to provide
a bounty program for our open source projects.

Please report security vulnerabilities associated with the LangChain
open source projects by visiting the following link:

[https://huntr.com/bounties/disclose/](https://huntr.com/bounties/disclose/?target=https%3A%2F%2Fgithub.com%2Flangchain-ai%2Flangchain&validSearch=true)

Before reporting a vulnerability, please review:

1) In-Scope Targets and Out-of-Scope Targets below.
2) The [langchain-ai/langchain](https://github.com/langchain-ai/langchain) monorepo structure.
3) LangChain [security guidelines](https://python.langchain.com/docs/security) to
   understand what we consider to be a security vulnerability vs. developer
   responsibility.

### In-Scope Targets

The following packages and repositories are eligible for bug bounties:

- langchain-core
- langchain (see exceptions)
- langchain-community (see exceptions)
- langgraph
- langserve

### Out of Scope Targets

All out of scope targets defined by huntr as well as:

- **langchain-experimental**: This repository is for experimental code and is not
  eligible for bug bounties, bug reports to it will be marked as interesting or waste of
  time and published with no bounty attached.
- **tools**: Tools in either langchain or langchain-community are not eligible for bug
  bounties. This includes the following directories
  - langchain/tools
  - langchain-community/tools
  - Please review our [security guidelines](https://python.langchain.com/docs/security)
    for more details, but generally tools interact with the real world. Developers are
    expected to understand the security implications of their code and are responsible
    for the security of their tools.
- Code documented with security notices. This will be decided done on a case by
  case basis, but likely will not be eligible for a bounty as the code is already
  documented with guidelines for developers that should be followed for making their
  application secure.
- Any LangSmith related repositories or APIs see below.

## Reporting LangSmith Vulnerabilities

Please report security vulnerabilities associated with LangSmith by email to `security@langchain.dev`.

- LangSmith site: <https://smith.langchain.com>
- SDK client: <https://github.com/langchain-ai/langsmith-sdk>

### Other Security Concerns

For any other security concerns, please contact us at `security@langchain.dev`.

```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/README.md
```md
# LangGraph examples

This directory should NOT be used for documentation. All new documentation must be added to `docs/docs/` directory.
```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/async.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "23544406",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/async.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/branching.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "14f7ca50",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/branching.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/configuration.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "e9a58c69",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/configuration.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/create-react-agent-hitl.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "a1e6efeb",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/create-react-agent-hitl.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/create-react-agent-memory.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "1ef41a89",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/create-react-agent-memory.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/create-react-agent-system-prompt.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "9e2f7902",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/create-react-agent-system-prompt.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/create-react-agent.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "eb07372e",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/create-react-agent.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/input_output_schema.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "fc0793cb",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/input_output_schema.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/map-reduce.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "42abb708",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/map-reduce.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/node-retries.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "017a01f4",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/node-retries.ipynb"
   ]
  }
 ],
 "metadata": {
  "kernelspec": {
   "display_name": "env",
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
 "nbformat_minor": 2
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/pass-config-to-tools.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "05f6ad0a",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/pass-config-to-tools.ipynb"
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
 "nbformat_minor": 4
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/pass-run-time-values-to-tools.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "8f38bec5",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/pass-run-time-values-to-tools.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/pass_private_state.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "4da17088",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/pass_private_state.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/persistence.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "d16e8b9c",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/memory/add-memory.md."
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
https://github.com/langchain-ai/langgraph/blob/main/examples/persistence_mongodb.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "78217098",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/persistence_mongodb.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/persistence_postgres.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "18526f23",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/memory/add-memory.md"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/persistence_redis.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "eee6ecdd",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/persistence_redis.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/react-agent-from-scratch.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "294995c4",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/react-agent-from-scratch.ipynb"
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
 "nbformat_minor": 2
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/react-agent-structured-output.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "40f0d107",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/react-agent-structured-output.ipynb"
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
 "nbformat_minor": 4
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/recursion-limit.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "fa3f7c50",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/recursion-limit.ipynb"
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
 "nbformat_minor": 2
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/run-id-langsmith.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "bbd6e9b8",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/run-id-langsmith.ipynb"
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
 "nbformat_minor": 4
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/state-model.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "4149ffcc",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/state-model.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/stream-multiple.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "e663f597",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/stream-multiple.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/stream-updates.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "e6829c80",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/stream-updates.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/stream-values.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "5ec11895",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/stream-values.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/streaming-content.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "6619387c",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/streaming-content.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/streaming-events-from-within-tools-without-langchain.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "57b7e303",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/streaming-events-from-within-tools-without-langchain.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/streaming-events-from-within-tools.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "8e71a0c8",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/streaming-events-from-within-tools.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/streaming-from-final-node.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "756e4554",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/streaming-from-final-node.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/streaming-subgraphs.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "47164a72",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/streaming-subgraphs.ipynb"
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
 "nbformat_minor": 2
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/streaming-tokens-without-langchain.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "218dfbcb",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/streaming-tokens-without-langchain.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/streaming-tokens.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "99eb887e",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/streaming-tokens.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/subgraph-transform-state.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "0de7689f",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/subgraph-transform-state.ipynb"
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
 "nbformat_minor": 4
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/subgraph.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "f49876e1",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/subgraph.ipynb"
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
 "nbformat_minor": 4
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/subgraphs-manage-state.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "5106959e",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/subgraphs-manage-state.ipynb"
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
 "nbformat_minor": 4
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/tool-calling-errors.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "dc21501d",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/tool-calling-errors.ipynb"
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
 "nbformat_minor": 4
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/tool-calling.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "7fd8bd65",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/tool-calling.ipynb"
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
 "nbformat_minor": 4
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/examples/visualization.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "9c9cb15a",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/visualization.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/human_in_the_loop/wait-user-input.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "3ecab357",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/how-tos/human_in_the_loop/wait-user-input.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/plan-and-execute/plan-and-execute.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "9138f92e",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/plan-and-execute/plan-and-execute.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/reflection/reflection.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "658773a2",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/reflection/reflection.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/rewoo/rewoo.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "961f43ec",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/rewoo/rewoo.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/code_assistant/langgraph_code_assistant.ipynb
```ipynb
{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "1f2f13ca",
   "metadata": {},
   "source": [
    "This file has been moved to https://github.com/langchain-ai/langgraph/blob/main/docs/docs/tutorials/code_assistant/langgraph_code_assistant.ipynb"
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
https://github.com/langchain-ai/langgraph/blob/main/examples/code_assistant/langgraph_code_assistant_mistral.ipynb
```ipynb
{
 "cells": [
  {
   "attachments": {
    "15d3ac32-cdf3-4800-a30c-f26d828d69c8.png": {
