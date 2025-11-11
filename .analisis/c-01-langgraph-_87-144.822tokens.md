        {% if config.separate_signature %}
          {% filter format_signature(function, config.line_length, crossrefs=config.signature_crossrefs) %}
            {{ function.name }}
          {% endfilter %}
        {% endif %}
      {% endblock signature %}

    {% else %}

      {% if config.show_root_toc_entry %}
        {% filter heading(
            heading_level,
            role="function",
            id=html_id,
            toc_label=(('<code class="doc-symbol doc-symbol-toc doc-symbol-' + symbol_type + '"></code>&nbsp;')|safe if config.show_symbol_type_toc else '') + (config.toc_label if config.toc_label and root else function.name),
            hidden=True,
          ) %}
        {% endfilter %}
      {% endif %}
      {% set heading_level = heading_level - 1 %}
    {% endif %}

    <div class="doc doc-contents {% if root %}first{% endif %}">
      {% block contents scoped %}
        {#- Contents block.
        
        This block renders the function’s docstring and source.
        -#}
        {% block docstring scoped %}
          {% with docstring_sections = function.docstring.parsed %}
            {% include "docstring"|get_template with context %}
          {% endwith %}
        {% endblock docstring %}

        {% block source scoped %}
          {% if config.show_source and function.source %}
            <details class="quote">
              <summary>{{ lang.t("Source code in") }} <code>
                {%- if function.relative_filepath.is_absolute() -%}
                  {{ function.relative_package_filepath }}
                {%- else -%}
                  {{ function.relative_filepath }}
                {%- endif -%}
              </code></summary>
              {{ function.source|highlight(language="python", linestart=function.lineno or 0, linenums=True) }}
            </details>
          {% endif %}
        {% endblock source %}
      {% endblock contents %}
    </div>

  {% endwith %}
</div>

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/.gitignore
```text
.langgraph_api/
.devserver.pid

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/LICENSE
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
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/Makefile
```text
.PHONY: all format lint test test_watch integration_tests spell_check spell_fix benchmark profile start-dev-server integration_tests

# Default target executed when no arguments are given to make.
all: help

######################
# TESTING AND COVERAGE
######################

# Benchmarks

OUTPUT ?= out/benchmark.json

install:  ## Install dependencies
	uv sync --frozen --all-extras --all-packages --group dev

benchmark:
	mkdir -p out
	rm -f $(OUTPUT)
	uv run python -m bench -o $(OUTPUT) --rigorous

benchmark-fast:
	mkdir -p out
	rm -f $(OUTPUT)
	uv run python -m bench -o $(OUTPUT) --fast

GRAPH ?= bench/fanout_to_subgraph.py

profile:
	mkdir -p out
	sudo uv run py-spy record -g -o out/profile.svg -- python $(GRAPH)

# Run unit tests and generate a coverage report.
coverage:
	uv run pytest --cov \
		--cov-config=.coveragerc \
		--cov-report xml \
		--cov-report term-missing:skip-covered

start-services:
	docker compose -f tests/compose-postgres.yml -f tests/compose-redis.yml up -V --force-recreate --wait --remove-orphans

stop-services:
	docker compose -f tests/compose-postgres.yml -f tests/compose-redis.yml down -v

start-dev-server:
	LOG_LEVEL=warning uv run langgraph dev --config tests/example_app/langgraph.json --no-browser & echo "$$!" > .devserver.pid
	@echo "Dev server started."

stop-dev-server:
	@if [ -f .devserver.pid ]; then \
		kill `cat .devserver.pid` && rm .devserver.pid; \
		 echo "Dev server stopped."; \
	else \
		echo "No dev server PID file found."; \
	fi

TEST ?= .
NO_DOCKER ?= $(sh command -v docker >/dev/null 2>&1 && echo "false" || echo "true")

test:
	if [ "$(NO_DOCKER)" = "false" ]; then \
		make start-services &&\
		make start-dev-server &&\
		uv run pytest $(TEST); \
		EXIT_CODE=$$?; \
		make stop-services; \
		make stop-dev-server; \
		exit $$EXIT_CODE; \
	else \
		NO_DOCKER=true uv run pytest $(TEST) ; \
		EXIT_CODE=$$?; \
		exit $$EXIT_CODE; \
	fi

test_parallel:
	make start-services &&\
	make start-dev-server &&\
	uv run pytest -n auto --dist worksteal $(TEST) -vv --lf; \
	EXIT_CODE=$$?; \
	make stop-services; \
	make stop-dev-server; \
	exit $$EXIT_CODE

integration_tests:
	uv run pytest integration_tests

WORKERS ?= auto
XDIST_ARGS := $(if $(WORKERS),-n $(WORKERS) --dist worksteal,)
MAXFAIL ?=
MAXFAIL_ARGS := $(if $(MAXFAIL),--maxfail $(MAXFAIL),)
# Add an '-x' if xdist is enabled
XDIST_ARGS := $(if $(WORKERS),-x $(XDIST_ARGS),)

test_watch:
	make start-services &&\
	make start-dev-server &&\
	uv run ptw . -- --ff -vv $(XDIST_ARGS) $(MAXFAIL_ARGS) $(TEST); \
	EXIT_CODE=$$?; \
	make stop-services; \
	make stop-dev-server; \
	exit $$EXIT_CODE

test_watch_all:
	npx concurrently -n langgraph,checkpoint,checkpoint-sqlite,postgres "make test_watch" "make -C ../checkpoint test_watch" "make -C ../checkpoint-sqlite test_watch" "make -C ../checkpoint-postgres test_watch"


######################
# LINTING AND FORMATTING
######################

# Define a variable for Python and notebook files.
PYTHON_FILES=.
MYPY_CACHE=.mypy_cache
lint format: PYTHON_FILES=.
lint_diff format_diff: PYTHON_FILES=$(shell git diff --name-only --relative --diff-filter=d main . | grep -E r'\.py$$|\.ipynb$$')
lint_package: PYTHON_FILES=langgraph
lint_tests: PYTHON_FILES=tests
lint_tests: MYPY_CACHE=.mypy_cache_test

lint lint_diff lint_package lint_tests:
	uv run ruff check .
	[ "$(PYTHON_FILES)" = "" ] || uv run ruff format $(PYTHON_FILES) --diff
	[ "$(PYTHON_FILES)" = "" ] || uv run ruff check --select I $(PYTHON_FILES)
	[ "$(PYTHON_FILES)" = "" ] || mkdir -p $(MYPY_CACHE)
	[ "$(PYTHON_FILES)" = "" ] || uv run mypy langgraph --cache-dir $(MYPY_CACHE)

format format_diff:
	uv run ruff format $(PYTHON_FILES)
	uv run ruff check --select I --fix $(PYTHON_FILES)

spell_check:
	uv run codespell --toml pyproject.toml

spell_fix:
	uv run codespell --toml pyproject.toml -w


######################
# HELP
######################

help:
	@echo '===================='
	@echo '-- DOCUMENTATION --'
	
	@echo '-- LINTING --'
	@echo 'format                       - run code formatters'
	@echo 'lint                         - run linters'
	@echo 'spell_check               	- run codespell on the project'
	@echo 'spell_fix               		- run codespell on the project and fix the errors'
	@echo '-- TESTS --'
	@echo 'coverage                     - run unit tests and generate coverage report'
	@echo 'test                         - run unit tests'
	@echo 'test TEST_FILE=<test_file>   - run all tests in file'
	@echo 'test_watch                   - run unit tests in watch mode'

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/README.md
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
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/pyproject.toml
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "langgraph"
version = "1.0.3"
description = "Building stateful, multi-actor applications with LLMs"
authors = []
requires-python = ">=3.10"
readme = "README.md"
license = "MIT"
license-files = ['LICENSE']
classifiers = [
    'Development Status :: 5 - Production/Stable',
    'Programming Language :: Python',
    'Programming Language :: Python :: Implementation :: CPython',
    'Programming Language :: Python :: Implementation :: PyPy',
    'Programming Language :: Python :: 3',
    'Programming Language :: Python :: 3 :: Only',
    'Programming Language :: Python :: 3.10',
    'Programming Language :: Python :: 3.11',
    'Programming Language :: Python :: 3.12',
    'Programming Language :: Python :: 3.13',
]
dependencies = [
    "langchain-core>=0.1",
    "langgraph-checkpoint>=2.1.0,<4.0.0",
    "langgraph-sdk>=0.2.2,<0.3.0",
    "langgraph-prebuilt>=1.0.2,<1.1.0",
    "xxhash>=3.5.0",
    "pydantic>=2.7.4",
]


[project.urls]
Homepage = "https://docs.langchain.com/oss/python/langgraph/overview"
Documentation = "https://reference.langchain.com/python/langgraph/"
Source = "https://github.com/langchain-ai/langgraph/tree/main/libs/langgraph"
Changelog = "https://github.com/langchain-ai/langgraph/releases"
Twitter = "https://x.com/LangChainAI"
Slack = "https://www.langchain.com/join-community"
Reddit = "https://www.reddit.com/r/LangChain/"

[dependency-groups]
test = [
    "pytest",
    "pytest-cov",
    "pytest-dotenv",
    "pytest-mock",
    "syrupy",
    "httpx",
    "pytest-watcher",
    "pytest-xdist[psutil]",
    "pytest-repeat",
    "langchain-core>=1.0.0",
    "langgraph-prebuilt",
    "langgraph-checkpoint",
    "langgraph-checkpoint-sqlite",
    "langgraph-checkpoint-postgres",
    "langgraph-sdk",
    "psycopg[binary]",
    "uvloop==0.21.0beta1",
    "pyperf",
    "py-spy",
    "pycryptodome",
    "langgraph-cli; python_version < '3.14'",
    "langgraph-cli[inmem]; python_version < '3.14'",
    "redis",
]
lint = [
    "mypy",
    "ruff",
    "types-requests",
]
dev = [
    {include-group = "test"},
    {include-group = "lint"},
    "jupyter",
]


[tool.uv.sources]
langgraph-prebuilt = { path = "../prebuilt", editable = true }
langgraph-checkpoint = { path = "../checkpoint", editable = true }
langgraph-checkpoint-sqlite = { path = "../checkpoint-sqlite", editable = true }
langgraph-checkpoint-postgres = { path = "../checkpoint-postgres", editable = true }
langgraph-sdk = { path = "../sdk-py", editable = true }
langgraph-cli = { path = "../cli", editable = true }

[tool.ruff]
lint.select = [ "E", "F", "I", "TID251", "UP" ]
lint.ignore = [ "E501" ]
line-length = 88
indent-width = 4
extend-include = ["*.ipynb"]
target-version = "py310"

[tool.ruff.lint.flake8-tidy-imports.banned-api]
"typing.TypedDict".msg = "Use typing_extensions.TypedDict instead."

[tool.mypy]
# https://mypy.readthedocs.io/en/stable/config_file.html
disallow_untyped_defs = "True"
explicit_package_bases = "True"
warn_no_return = "False"
warn_unused_ignores = "True"
warn_redundant_casts = "True"
allow_redefinition = "True"
disable_error_code = "typeddict-item, return-value, override, has-type"

[tool.coverage.run]
omit = ["tests/*"]

[tool.pytest-watcher]
now = true
delay = 0.1
patterns = ["*.py"]

[tool.hatch.build.targets.wheel]
packages = ["langgraph"]

[tool.pytest.ini_options]
addopts = "--full-trace --strict-markers --strict-config --durations=5 --snapshot-warn-unused"

[tool.codespell]
# Ignore words specific to the LangGraph library code
ignore-words-list = "infor,thead,stdio,nd,jupyter,lets,lite,uis,deque,langgraph,langchain,pydantic,typing,async,await,coroutine,iterable,iterables,serializable,deserializable,checkpointer,checkpointing,stateful,statefulness,prebuilt,prebuilt,supervisor,supervisory,swarm,swarming,multiactor,multiactors,subgraph,subgraphs,workflow,workflows,streaming,streamable,streamed,streamer,streamers,streaming,streamable,streamed,streamer,streamers"

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/uv.lock
```lock
version = 1
revision = 3
requires-python = ">=3.10"
resolution-markers = [
    "python_full_version >= '3.14'",
    "python_full_version >= '3.11' and python_full_version < '3.14'",
    "python_full_version < '3.11'",
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
name = "anyio"
version = "4.11.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "exceptiongroup", marker = "python_full_version < '3.11'" },
    { name = "idna" },
    { name = "sniffio" },
    { name = "typing-extensions", marker = "python_full_version < '3.13'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/c6/78/7d432127c41b50bccba979505f272c16cbcadcc33645d5fa3a738110ae75/anyio-4.11.0.tar.gz", hash = "sha256:82a8d0b81e318cc5ce71a5f1f8b5c4e63619620b63141ef8c995fa0db95a57c4", size = 219094, upload-time = "2025-09-23T09:19:12.58Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/15/b3/9b1a8074496371342ec1e796a96f99c82c945a339cd81a8e73de28b4cf9e/anyio-4.11.0-py3-none-any.whl", hash = "sha256:0287e96f4d26d4149305414d4e3bc32f0dcd0862365a4bddea19d7a1ec38c4fc", size = 109097, upload-time = "2025-09-23T09:19:10.601Z" },
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
version = "25.1.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "cffi" },
]
sdist = { url = "https://files.pythonhosted.org/packages/5c/2d/db8af0df73c1cf454f71b2bbe5e356b8c1f8041c979f505b3d3186e520a9/argon2_cffi_bindings-25.1.0.tar.gz", hash = "sha256:b957f3e6ea4d55d820e40ff76f450952807013d361a65d7f28acc0acbf29229d", size = 1783441, upload-time = "2025-07-30T10:02:05.147Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/60/97/3c0a35f46e52108d4707c44b95cfe2afcafc50800b5450c197454569b776/argon2_cffi_bindings-25.1.0-cp314-cp314t-macosx_10_13_universal2.whl", hash = "sha256:3d3f05610594151994ca9ccb3c771115bdb4daef161976a266f0dd8aa9996b8f", size = 54393, upload-time = "2025-07-30T10:01:40.97Z" },
    { url = "https://files.pythonhosted.org/packages/9d/f4/98bbd6ee89febd4f212696f13c03ca302b8552e7dbf9c8efa11ea4a388c3/argon2_cffi_bindings-25.1.0-cp314-cp314t-macosx_10_13_x86_64.whl", hash = "sha256:8b8efee945193e667a396cbc7b4fb7d357297d6234d30a489905d96caabde56b", size = 29328, upload-time = "2025-07-30T10:01:41.916Z" },
    { url = "https://files.pythonhosted.org/packages/43/24/90a01c0ef12ac91a6be05969f29944643bc1e5e461155ae6559befa8f00b/argon2_cffi_bindings-25.1.0-cp314-cp314t-macosx_11_0_arm64.whl", hash = "sha256:3c6702abc36bf3ccba3f802b799505def420a1b7039862014a65db3205967f5a", size = 31269, upload-time = "2025-07-30T10:01:42.716Z" },
    { url = "https://files.pythonhosted.org/packages/d4/d3/942aa10782b2697eee7af5e12eeff5ebb325ccfb86dd8abda54174e377e4/argon2_cffi_bindings-25.1.0-cp314-cp314t-manylinux_2_26_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:a1c70058c6ab1e352304ac7e3b52554daadacd8d453c1752e547c76e9c99ac44", size = 86558, upload-time = "2025-07-30T10:01:43.943Z" },
    { url = "https://files.pythonhosted.org/packages/0d/82/b484f702fec5536e71836fc2dbc8c5267b3f6e78d2d539b4eaa6f0db8bf8/argon2_cffi_bindings-25.1.0-cp314-cp314t-manylinux_2_26_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:e2fd3bfbff3c5d74fef31a722f729bf93500910db650c925c2d6ef879a7e51cb", size = 92364, upload-time = "2025-07-30T10:01:44.887Z" },
    { url = "https://files.pythonhosted.org/packages/c9/c1/a606ff83b3f1735f3759ad0f2cd9e038a0ad11a3de3b6c673aa41c24bb7b/argon2_cffi_bindings-25.1.0-cp314-cp314t-musllinux_1_2_aarch64.whl", hash = "sha256:c4f9665de60b1b0e99bcd6be4f17d90339698ce954cfd8d9cf4f91c995165a92", size = 85637, upload-time = "2025-07-30T10:01:46.225Z" },
    { url = "https://files.pythonhosted.org/packages/44/b4/678503f12aceb0262f84fa201f6027ed77d71c5019ae03b399b97caa2f19/argon2_cffi_bindings-25.1.0-cp314-cp314t-musllinux_1_2_x86_64.whl", hash = "sha256:ba92837e4a9aa6a508c8d2d7883ed5a8f6c308c89a4790e1e447a220deb79a85", size = 91934, upload-time = "2025-07-30T10:01:47.203Z" },
    { url = "https://files.pythonhosted.org/packages/f0/c7/f36bd08ef9bd9f0a9cff9428406651f5937ce27b6c5b07b92d41f91ae541/argon2_cffi_bindings-25.1.0-cp314-cp314t-win32.whl", hash = "sha256:84a461d4d84ae1295871329b346a97f68eade8c53b6ed9a7ca2d7467f3c8ff6f", size = 28158, upload-time = "2025-07-30T10:01:48.341Z" },
    { url = "https://files.pythonhosted.org/packages/b3/80/0106a7448abb24a2c467bf7d527fe5413b7fdfa4ad6d6a96a43a62ef3988/argon2_cffi_bindings-25.1.0-cp314-cp314t-win_amd64.whl", hash = "sha256:b55aec3565b65f56455eebc9b9f34130440404f27fe21c3b375bf1ea4d8fbae6", size = 32597, upload-time = "2025-07-30T10:01:49.112Z" },
    { url = "https://files.pythonhosted.org/packages/05/b8/d663c9caea07e9180b2cb662772865230715cbd573ba3b5e81793d580316/argon2_cffi_bindings-25.1.0-cp314-cp314t-win_arm64.whl", hash = "sha256:87c33a52407e4c41f3b70a9c2d3f6056d88b10dad7695be708c5021673f55623", size = 28231, upload-time = "2025-07-30T10:01:49.92Z" },
    { url = "https://files.pythonhosted.org/packages/1d/57/96b8b9f93166147826da5f90376e784a10582dd39a393c99bb62cfcf52f0/argon2_cffi_bindings-25.1.0-cp39-abi3-macosx_10_9_universal2.whl", hash = "sha256:aecba1723ae35330a008418a91ea6cfcedf6d31e5fbaa056a166462ff066d500", size = 54121, upload-time = "2025-07-30T10:01:50.815Z" },
    { url = "https://files.pythonhosted.org/packages/0a/08/a9bebdb2e0e602dde230bdde8021b29f71f7841bd54801bcfd514acb5dcf/argon2_cffi_bindings-25.1.0-cp39-abi3-macosx_10_9_x86_64.whl", hash = "sha256:2630b6240b495dfab90aebe159ff784d08ea999aa4b0d17efa734055a07d2f44", size = 29177, upload-time = "2025-07-30T10:01:51.681Z" },
    { url = "https://files.pythonhosted.org/packages/b6/02/d297943bcacf05e4f2a94ab6f462831dc20158614e5d067c35d4e63b9acb/argon2_cffi_bindings-25.1.0-cp39-abi3-macosx_11_0_arm64.whl", hash = "sha256:7aef0c91e2c0fbca6fc68e7555aa60ef7008a739cbe045541e438373bc54d2b0", size = 31090, upload-time = "2025-07-30T10:01:53.184Z" },
    { url = "https://files.pythonhosted.org/packages/c1/93/44365f3d75053e53893ec6d733e4a5e3147502663554b4d864587c7828a7/argon2_cffi_bindings-25.1.0-cp39-abi3-manylinux_2_26_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:1e021e87faa76ae0d413b619fe2b65ab9a037f24c60a1e6cc43457ae20de6dc6", size = 81246, upload-time = "2025-07-30T10:01:54.145Z" },
    { url = "https://files.pythonhosted.org/packages/09/52/94108adfdd6e2ddf58be64f959a0b9c7d4ef2fa71086c38356d22dc501ea/argon2_cffi_bindings-25.1.0-cp39-abi3-manylinux_2_26_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:d3e924cfc503018a714f94a49a149fdc0b644eaead5d1f089330399134fa028a", size = 87126, upload-time = "2025-07-30T10:01:55.074Z" },
    { url = "https://files.pythonhosted.org/packages/72/70/7a2993a12b0ffa2a9271259b79cc616e2389ed1a4d93842fac5a1f923ffd/argon2_cffi_bindings-25.1.0-cp39-abi3-musllinux_1_2_aarch64.whl", hash = "sha256:c87b72589133f0346a1cb8d5ecca4b933e3c9b64656c9d175270a000e73b288d", size = 80343, upload-time = "2025-07-30T10:01:56.007Z" },
    { url = "https://files.pythonhosted.org/packages/78/9a/4e5157d893ffc712b74dbd868c7f62365618266982b64accab26bab01edc/argon2_cffi_bindings-25.1.0-cp39-abi3-musllinux_1_2_x86_64.whl", hash = "sha256:1db89609c06afa1a214a69a462ea741cf735b29a57530478c06eb81dd403de99", size = 86777, upload-time = "2025-07-30T10:01:56.943Z" },
    { url = "https://files.pythonhosted.org/packages/74/cd/15777dfde1c29d96de7f18edf4cc94c385646852e7c7b0320aa91ccca583/argon2_cffi_bindings-25.1.0-cp39-abi3-win32.whl", hash = "sha256:473bcb5f82924b1becbb637b63303ec8d10e84c8d241119419897a26116515d2", size = 27180, upload-time = "2025-07-30T10:01:57.759Z" },
    { url = "https://files.pythonhosted.org/packages/e2/c6/a759ece8f1829d1f162261226fbfd2c6832b3ff7657384045286d2afa384/argon2_cffi_bindings-25.1.0-cp39-abi3-win_amd64.whl", hash = "sha256:a98cd7d17e9f7ce244c0803cad3c23a7d379c301ba618a5fa76a67d116618b98", size = 31715, upload-time = "2025-07-30T10:01:58.56Z" },
    { url = "https://files.pythonhosted.org/packages/42/b9/f8d6fa329ab25128b7e98fd83a3cb34d9db5b059a9847eddb840a0af45dd/argon2_cffi_bindings-25.1.0-cp39-abi3-win_arm64.whl", hash = "sha256:b0fdbcf513833809c882823f98dc2f931cf659d9a1429616ac3adebb49f5db94", size = 27149, upload-time = "2025-07-30T10:01:59.329Z" },
    { url = "https://files.pythonhosted.org/packages/11/2d/ba4e4ca8d149f8dcc0d952ac0967089e1d759c7e5fcf0865a317eb680fbb/argon2_cffi_bindings-25.1.0-pp310-pypy310_pp73-macosx_10_15_x86_64.whl", hash = "sha256:6dca33a9859abf613e22733131fc9194091c1fa7cb3e131c143056b4856aa47e", size = 24549, upload-time = "2025-07-30T10:02:00.101Z" },
    { url = "https://files.pythonhosted.org/packages/5c/82/9b2386cc75ac0bd3210e12a44bfc7fd1632065ed8b80d573036eecb10442/argon2_cffi_bindings-25.1.0-pp310-pypy310_pp73-macosx_11_0_arm64.whl", hash = "sha256:21378b40e1b8d1655dd5310c84a40fc19a9aa5e6366e835ceb8576bf0fea716d", size = 25539, upload-time = "2025-07-30T10:02:00.929Z" },
    { url = "https://files.pythonhosted.org/packages/31/db/740de99a37aa727623730c90d92c22c9e12585b3c98c54b7960f7810289f/argon2_cffi_bindings-25.1.0-pp310-pypy310_pp73-manylinux_2_26_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:5d588dec224e2a83edbdc785a5e6f3c6cd736f46bfd4b441bbb5aa1f5085e584", size = 28467, upload-time = "2025-07-30T10:02:02.08Z" },
    { url = "https://files.pythonhosted.org/packages/71/7a/47c4509ea18d755f44e2b92b7178914f0c113946d11e16e626df8eaa2b0b/argon2_cffi_bindings-25.1.0-pp310-pypy310_pp73-manylinux_2_26_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:5acb4e41090d53f17ca1110c3427f0a130f944b896fc8c83973219c97f57b690", size = 27355, upload-time = "2025-07-30T10:02:02.867Z" },
    { url = "https://files.pythonhosted.org/packages/ee/82/82745642d3c46e7cea25e1885b014b033f4693346ce46b7f47483cf5d448/argon2_cffi_bindings-25.1.0-pp310-pypy310_pp73-win_amd64.whl", hash = "sha256:da0c79c23a63723aa5d782250fbf51b768abca630285262fb5144ba5ae01e520", size = 29187, upload-time = "2025-07-30T10:02:03.674Z" },
]

[[package]]
name = "arrow"
version = "1.4.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "python-dateutil" },
    { name = "tzdata" },
]
sdist = { url = "https://files.pythonhosted.org/packages/b9/33/032cdc44182491aa708d06a68b62434140d8c50820a087fac7af37703357/arrow-1.4.0.tar.gz", hash = "sha256:ed0cc050e98001b8779e84d461b0098c4ac597e88704a655582b21d116e526d7", size = 152931, upload-time = "2025-10-18T17:46:46.761Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/ed/c9/d7977eaacb9df673210491da99e6a247e93df98c715fc43fd136ce1d3d33/arrow-1.4.0-py3-none-any.whl", hash = "sha256:749f0769958ebdc79c173ff0b0670d59051a535fa26e8eba02953dc19eb43205", size = 68797, upload-time = "2025-10-18T17:46:45.663Z" },
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
dependencies = [
    { name = "typing-extensions", marker = "python_full_version < '3.11'" },
]
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
name = "attrs"
version = "25.4.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/6b/5c/685e6633917e101e5dcb62b9dd76946cbb57c26e133bae9e0cd36033c0a9/attrs-25.4.0.tar.gz", hash = "sha256:16d5969b87f0859ef33a48b35d55ac1be6e42ae49d5e853b597db70c35c57e11", size = 934251, upload-time = "2025-10-06T13:54:44.725Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/3a/2a/7cc015f5b9f5db42b7d48157e23356022889fc354a2813c15934b7cb5c0e/attrs-25.4.0-py3-none-any.whl", hash = "sha256:adcf7e2a1fb3b36ac48d97835bb6d8ade15b8dcce26aba8bf1d14847b57a3373", size = 67615, upload-time = "2025-10-06T13:54:43.17Z" },
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
name = "beautifulsoup4"
version = "4.14.2"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "soupsieve" },
    { name = "typing-extensions" },
]
sdist = { url = "https://files.pythonhosted.org/packages/77/e9/df2358efd7659577435e2177bfa69cba6c33216681af51a707193dec162a/beautifulsoup4-4.14.2.tar.gz", hash = "sha256:2a98ab9f944a11acee9cc848508ec28d9228abfd522ef0fad6a02a72e0ded69e", size = 625822, upload-time = "2025-09-29T10:05:42.613Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/94/fe/3aed5d0be4d404d12d36ab97e2f1791424d9ca39c2f754a6285d59a3b01d/beautifulsoup4-4.14.2-py3-none-any.whl", hash = "sha256:5ef6fa3a8cbece8488d66985560f97ed091e22bbc4e9c2338508a9d5de6d4515", size = 106392, upload-time = "2025-09-29T10:05:43.771Z" },
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
name = "blockbuster"
version = "1.5.25"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "forbiddenfruit", marker = "python_full_version >= '3.11' and python_full_version < '3.14' and implementation_name == 'cpython'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/7f/bc/57c49465decaeeedd58ce2d970b4cdfd93a74ba9993abff2dc498a31c283/blockbuster-1.5.25.tar.gz", hash = "sha256:b72f1d2aefdeecd2a820ddf1e1c8593bf00b96e9fdc4cd2199ebafd06f7cb8f0", size = 36058, upload-time = "2025-07-14T16:00:20.766Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/0b/01/dccc277c014f171f61a6047bb22c684e16c7f2db6bb5c8cce1feaf41ec55/blockbuster-1.5.25-py3-none-any.whl", hash = "sha256:cb06229762273e0f5f3accdaed3d2c5a3b61b055e38843de202311ede21bb0f5", size = 13196, upload-time = "2025-07-14T16:00:19.396Z" },
]

[[package]]
name = "certifi"
version = "2025.10.5"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/4c/5b/b6ce21586237c77ce67d01dc5507039d444b630dd76611bbca2d8e5dcd91/certifi-2025.10.5.tar.gz", hash = "sha256:47c09d31ccf2acf0be3f701ea53595ee7e0b8fa08801c6624be771df09ae7b43", size = 164519, upload-time = "2025-10-05T04:12:15.808Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/e4/37/af0d2ef3967ac0d6113837b44a4f0bfe1328c2b9763bd5b1744520e5cfed/certifi-2025.10.5-py3-none-any.whl", hash = "sha256:0f212c2744a9bb6de0c56639a6f68afe01ecd92d91f14ae897c4fe7bbeeef0de", size = 163286, upload-time = "2025-10-05T04:12:14.03Z" },
]

[[package]]
name = "cffi"
version = "2.0.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "pycparser", marker = "implementation_name != 'PyPy'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/eb/56/b1ba7935a17738ae8453301356628e8147c79dbb825bcbc73dc7401f9846/cffi-2.0.0.tar.gz", hash = "sha256:44d1b5909021139fe36001ae048dbdde8214afa20200eda0f64c068cac5d5529", size = 523588, upload-time = "2025-09-08T23:24:04.541Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/93/d7/516d984057745a6cd96575eea814fe1edd6646ee6efd552fb7b0921dec83/cffi-2.0.0-cp310-cp310-macosx_10_13_x86_64.whl", hash = "sha256:0cf2d91ecc3fcc0625c2c530fe004f82c110405f101548512cce44322fa8ac44", size = 184283, upload-time = "2025-09-08T23:22:08.01Z" },
    { url = "https://files.pythonhosted.org/packages/9e/84/ad6a0b408daa859246f57c03efd28e5dd1b33c21737c2db84cae8c237aa5/cffi-2.0.0-cp310-cp310-macosx_11_0_arm64.whl", hash = "sha256:f73b96c41e3b2adedc34a7356e64c8eb96e03a3782b535e043a986276ce12a49", size = 180504, upload-time = "2025-09-08T23:22:10.637Z" },
    { url = "https://files.pythonhosted.org/packages/50/bd/b1a6362b80628111e6653c961f987faa55262b4002fcec42308cad1db680/cffi-2.0.0-cp310-cp310-manylinux1_i686.manylinux2014_i686.manylinux_2_17_i686.manylinux_2_5_i686.whl", hash = "sha256:53f77cbe57044e88bbd5ed26ac1d0514d2acf0591dd6bb02a3ae37f76811b80c", size = 208811, upload-time = "2025-09-08T23:22:12.267Z" },
    { url = "https://files.pythonhosted.org/packages/4f/27/6933a8b2562d7bd1fb595074cf99cc81fc3789f6a6c05cdabb46284a3188/cffi-2.0.0-cp310-cp310-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:3e837e369566884707ddaf85fc1744b47575005c0a229de3327f8f9a20f4efeb", size = 216402, upload-time = "2025-09-08T23:22:13.455Z" },
    { url = "https://files.pythonhosted.org/packages/05/eb/b86f2a2645b62adcfff53b0dd97e8dfafb5c8aa864bd0d9a2c2049a0d551/cffi-2.0.0-cp310-cp310-manylinux2014_ppc64le.manylinux_2_17_ppc64le.whl", hash = "sha256:5eda85d6d1879e692d546a078b44251cdd08dd1cfb98dfb77b670c97cee49ea0", size = 203217, upload-time = "2025-09-08T23:22:14.596Z" },
    { url = "https://files.pythonhosted.org/packages/9f/e0/6cbe77a53acf5acc7c08cc186c9928864bd7c005f9efd0d126884858a5fe/cffi-2.0.0-cp310-cp310-manylinux2014_s390x.manylinux_2_17_s390x.whl", hash = "sha256:9332088d75dc3241c702d852d4671613136d90fa6881da7d770a483fd05248b4", size = 203079, upload-time = "2025-09-08T23:22:15.769Z" },
    { url = "https://files.pythonhosted.org/packages/98/29/9b366e70e243eb3d14a5cb488dfd3a0b6b2f1fb001a203f653b93ccfac88/cffi-2.0.0-cp310-cp310-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:fc7de24befaeae77ba923797c7c87834c73648a05a4bde34b3b7e5588973a453", size = 216475, upload-time = "2025-09-08T23:22:17.427Z" },
    { url = "https://files.pythonhosted.org/packages/21/7a/13b24e70d2f90a322f2900c5d8e1f14fa7e2a6b3332b7309ba7b2ba51a5a/cffi-2.0.0-cp310-cp310-musllinux_1_2_aarch64.whl", hash = "sha256:cf364028c016c03078a23b503f02058f1814320a56ad535686f90565636a9495", size = 218829, upload-time = "2025-09-08T23:22:19.069Z" },
    { url = "https://files.pythonhosted.org/packages/60/99/c9dc110974c59cc981b1f5b66e1d8af8af764e00f0293266824d9c4254bc/cffi-2.0.0-cp310-cp310-musllinux_1_2_i686.whl", hash = "sha256:e11e82b744887154b182fd3e7e8512418446501191994dbf9c9fc1f32cc8efd5", size = 211211, upload-time = "2025-09-08T23:22:20.588Z" },
    { url = "https://files.pythonhosted.org/packages/49/72/ff2d12dbf21aca1b32a40ed792ee6b40f6dc3a9cf1644bd7ef6e95e0ac5e/cffi-2.0.0-cp310-cp310-musllinux_1_2_x86_64.whl", hash = "sha256:8ea985900c5c95ce9db1745f7933eeef5d314f0565b27625d9a10ec9881e1bfb", size = 218036, upload-time = "2025-09-08T23:22:22.143Z" },
    { url = "https://files.pythonhosted.org/packages/e2/cc/027d7fb82e58c48ea717149b03bcadcbdc293553edb283af792bd4bcbb3f/cffi-2.0.0-cp310-cp310-win32.whl", hash = "sha256:1f72fb8906754ac8a2cc3f9f5aaa298070652a0ffae577e0ea9bd480dc3c931a", size = 172184, upload-time = "2025-09-08T23:22:23.328Z" },
    { url = "https://files.pythonhosted.org/packages/33/fa/072dd15ae27fbb4e06b437eb6e944e75b068deb09e2a2826039e49ee2045/cffi-2.0.0-cp310-cp310-win_amd64.whl", hash = "sha256:b18a3ed7d5b3bd8d9ef7a8cb226502c6bf8308df1525e1cc676c3680e7176739", size = 182790, upload-time = "2025-09-08T23:22:24.752Z" },
    { url = "https://files.pythonhosted.org/packages/12/4a/3dfd5f7850cbf0d06dc84ba9aa00db766b52ca38d8b86e3a38314d52498c/cffi-2.0.0-cp311-cp311-macosx_10_13_x86_64.whl", hash = "sha256:b4c854ef3adc177950a8dfc81a86f5115d2abd545751a304c5bcf2c2c7283cfe", size = 184344, upload-time = "2025-09-08T23:22:26.456Z" },
    { url = "https://files.pythonhosted.org/packages/4f/8b/f0e4c441227ba756aafbe78f117485b25bb26b1c059d01f137fa6d14896b/cffi-2.0.0-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:2de9a304e27f7596cd03d16f1b7c72219bd944e99cc52b84d0145aefb07cbd3c", size = 180560, upload-time = "2025-09-08T23:22:28.197Z" },
    { url = "https://files.pythonhosted.org/packages/b1/b7/1200d354378ef52ec227395d95c2576330fd22a869f7a70e88e1447eb234/cffi-2.0.0-cp311-cp311-manylinux1_i686.manylinux2014_i686.manylinux_2_17_i686.manylinux_2_5_i686.whl", hash = "sha256:baf5215e0ab74c16e2dd324e8ec067ef59e41125d3eade2b863d294fd5035c92", size = 209613, upload-time = "2025-09-08T23:22:29.475Z" },
    { url = "https://files.pythonhosted.org/packages/b8/56/6033f5e86e8cc9bb629f0077ba71679508bdf54a9a5e112a3c0b91870332/cffi-2.0.0-cp311-cp311-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:730cacb21e1bdff3ce90babf007d0a0917cc3e6492f336c2f0134101e0944f93", size = 216476, upload-time = "2025-09-08T23:22:31.063Z" },
    { url = "https://files.pythonhosted.org/packages/dc/7f/55fecd70f7ece178db2f26128ec41430d8720f2d12ca97bf8f0a628207d5/cffi-2.0.0-cp311-cp311-manylinux2014_ppc64le.manylinux_2_17_ppc64le.whl", hash = "sha256:6824f87845e3396029f3820c206e459ccc91760e8fa24422f8b0c3d1731cbec5", size = 203374, upload-time = "2025-09-08T23:22:32.507Z" },
    { url = "https://files.pythonhosted.org/packages/84/ef/a7b77c8bdc0f77adc3b46888f1ad54be8f3b7821697a7b89126e829e676a/cffi-2.0.0-cp311-cp311-manylinux2014_s390x.manylinux_2_17_s390x.whl", hash = "sha256:9de40a7b0323d889cf8d23d1ef214f565ab154443c42737dfe52ff82cf857664", size = 202597, upload-time = "2025-09-08T23:22:34.132Z" },
    { url = "https://files.pythonhosted.org/packages/d7/91/500d892b2bf36529a75b77958edfcd5ad8e2ce4064ce2ecfeab2125d72d1/cffi-2.0.0-cp311-cp311-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:8941aaadaf67246224cee8c3803777eed332a19d909b47e29c9842ef1e79ac26", size = 215574, upload-time = "2025-09-08T23:22:35.443Z" },
    { url = "https://files.pythonhosted.org/packages/44/64/58f6255b62b101093d5df22dcb752596066c7e89dd725e0afaed242a61be/cffi-2.0.0-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:a05d0c237b3349096d3981b727493e22147f934b20f6f125a3eba8f994bec4a9", size = 218971, upload-time = "2025-09-08T23:22:36.805Z" },
    { url = "https://files.pythonhosted.org/packages/ab/49/fa72cebe2fd8a55fbe14956f9970fe8eb1ac59e5df042f603ef7c8ba0adc/cffi-2.0.0-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:94698a9c5f91f9d138526b48fe26a199609544591f859c870d477351dc7b2414", size = 211972, upload-time = "2025-09-08T23:22:38.436Z" },
    { url = "https://files.pythonhosted.org/packages/0b/28/dd0967a76aab36731b6ebfe64dec4e981aff7e0608f60c2d46b46982607d/cffi-2.0.0-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:5fed36fccc0612a53f1d4d9a816b50a36702c28a2aa880cb8a122b3466638743", size = 217078, upload-time = "2025-09-08T23:22:39.776Z" },
    { url = "https://files.pythonhosted.org/packages/2b/c0/015b25184413d7ab0a410775fdb4a50fca20f5589b5dab1dbbfa3baad8ce/cffi-2.0.0-cp311-cp311-win32.whl", hash = "sha256:c649e3a33450ec82378822b3dad03cc228b8f5963c0c12fc3b1e0ab940f768a5", size = 172076, upload-time = "2025-09-08T23:22:40.95Z" },
    { url = "https://files.pythonhosted.org/packages/ae/8f/dc5531155e7070361eb1b7e4c1a9d896d0cb21c49f807a6c03fd63fc877e/cffi-2.0.0-cp311-cp311-win_amd64.whl", hash = "sha256:66f011380d0e49ed280c789fbd08ff0d40968ee7b665575489afa95c98196ab5", size = 182820, upload-time = "2025-09-08T23:22:42.463Z" },
    { url = "https://files.pythonhosted.org/packages/95/5c/1b493356429f9aecfd56bc171285a4c4ac8697f76e9bbbbb105e537853a1/cffi-2.0.0-cp311-cp311-win_arm64.whl", hash = "sha256:c6638687455baf640e37344fe26d37c404db8b80d037c3d29f58fe8d1c3b194d", size = 177635, upload-time = "2025-09-08T23:22:43.623Z" },
    { url = "https://files.pythonhosted.org/packages/ea/47/4f61023ea636104d4f16ab488e268b93008c3d0bb76893b1b31db1f96802/cffi-2.0.0-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:6d02d6655b0e54f54c4ef0b94eb6be0607b70853c45ce98bd278dc7de718be5d", size = 185271, upload-time = "2025-09-08T23:22:44.795Z" },
    { url = "https://files.pythonhosted.org/packages/df/a2/781b623f57358e360d62cdd7a8c681f074a71d445418a776eef0aadb4ab4/cffi-2.0.0-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:8eca2a813c1cb7ad4fb74d368c2ffbbb4789d377ee5bb8df98373c2cc0dee76c", size = 181048, upload-time = "2025-09-08T23:22:45.938Z" },
    { url = "https://files.pythonhosted.org/packages/ff/df/a4f0fbd47331ceeba3d37c2e51e9dfc9722498becbeec2bd8bc856c9538a/cffi-2.0.0-cp312-cp312-manylinux1_i686.manylinux2014_i686.manylinux_2_17_i686.manylinux_2_5_i686.whl", hash = "sha256:21d1152871b019407d8ac3985f6775c079416c282e431a4da6afe7aefd2bccbe", size = 212529, upload-time = "2025-09-08T23:22:47.349Z" },
    { url = "https://files.pythonhosted.org/packages/d5/72/12b5f8d3865bf0f87cf1404d8c374e7487dcf097a1c91c436e72e6badd83/cffi-2.0.0-cp312-cp312-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:b21e08af67b8a103c71a250401c78d5e0893beff75e28c53c98f4de42f774062", size = 220097, upload-time = "2025-09-08T23:22:48.677Z" },
    { url = "https://files.pythonhosted.org/packages/c2/95/7a135d52a50dfa7c882ab0ac17e8dc11cec9d55d2c18dda414c051c5e69e/cffi-2.0.0-cp312-cp312-manylinux2014_ppc64le.manylinux_2_17_ppc64le.whl", hash = "sha256:1e3a615586f05fc4065a8b22b8152f0c1b00cdbc60596d187c2a74f9e3036e4e", size = 207983, upload-time = "2025-09-08T23:22:50.06Z" },
    { url = "https://files.pythonhosted.org/packages/3a/c8/15cb9ada8895957ea171c62dc78ff3e99159ee7adb13c0123c001a2546c1/cffi-2.0.0-cp312-cp312-manylinux2014_s390x.manylinux_2_17_s390x.whl", hash = "sha256:81afed14892743bbe14dacb9e36d9e0e504cd204e0b165062c488942b9718037", size = 206519, upload-time = "2025-09-08T23:22:51.364Z" },
    { url = "https://files.pythonhosted.org/packages/78/2d/7fa73dfa841b5ac06c7b8855cfc18622132e365f5b81d02230333ff26e9e/cffi-2.0.0-cp312-cp312-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:3e17ed538242334bf70832644a32a7aae3d83b57567f9fd60a26257e992b79ba", size = 219572, upload-time = "2025-09-08T23:22:52.902Z" },
    { url = "https://files.pythonhosted.org/packages/07/e0/267e57e387b4ca276b90f0434ff88b2c2241ad72b16d31836adddfd6031b/cffi-2.0.0-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:3925dd22fa2b7699ed2617149842d2e6adde22b262fcbfada50e3d195e4b3a94", size = 222963, upload-time = "2025-09-08T23:22:54.518Z" },
    { url = "https://files.pythonhosted.org/packages/b6/75/1f2747525e06f53efbd878f4d03bac5b859cbc11c633d0fb81432d98a795/cffi-2.0.0-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:2c8f814d84194c9ea681642fd164267891702542f028a15fc97d4674b6206187", size = 221361, upload-time = "2025-09-08T23:22:55.867Z" },
    { url = "https://files.pythonhosted.org/packages/7b/2b/2b6435f76bfeb6bbf055596976da087377ede68df465419d192acf00c437/cffi-2.0.0-cp312-cp312-win32.whl", hash = "sha256:da902562c3e9c550df360bfa53c035b2f241fed6d9aef119048073680ace4a18", size = 172932, upload-time = "2025-09-08T23:22:57.188Z" },
    { url = "https://files.pythonhosted.org/packages/f8/ed/13bd4418627013bec4ed6e54283b1959cf6db888048c7cf4b4c3b5b36002/cffi-2.0.0-cp312-cp312-win_amd64.whl", hash = "sha256:da68248800ad6320861f129cd9c1bf96ca849a2771a59e0344e88681905916f5", size = 183557, upload-time = "2025-09-08T23:22:58.351Z" },
    { url = "https://files.pythonhosted.org/packages/95/31/9f7f93ad2f8eff1dbc1c3656d7ca5bfd8fb52c9d786b4dcf19b2d02217fa/cffi-2.0.0-cp312-cp312-win_arm64.whl", hash = "sha256:4671d9dd5ec934cb9a73e7ee9676f9362aba54f7f34910956b84d727b0d73fb6", size = 177762, upload-time = "2025-09-08T23:22:59.668Z" },
    { url = "https://files.pythonhosted.org/packages/4b/8d/a0a47a0c9e413a658623d014e91e74a50cdd2c423f7ccfd44086ef767f90/cffi-2.0.0-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:00bdf7acc5f795150faa6957054fbbca2439db2f775ce831222b66f192f03beb", size = 185230, upload-time = "2025-09-08T23:23:00.879Z" },
    { url = "https://files.pythonhosted.org/packages/4a/d2/a6c0296814556c68ee32009d9c2ad4f85f2707cdecfd7727951ec228005d/cffi-2.0.0-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:45d5e886156860dc35862657e1494b9bae8dfa63bf56796f2fb56e1679fc0bca", size = 181043, upload-time = "2025-09-08T23:23:02.231Z" },
    { url = "https://files.pythonhosted.org/packages/b0/1e/d22cc63332bd59b06481ceaac49d6c507598642e2230f201649058a7e704/cffi-2.0.0-cp313-cp313-manylinux1_i686.manylinux2014_i686.manylinux_2_17_i686.manylinux_2_5_i686.whl", hash = "sha256:07b271772c100085dd28b74fa0cd81c8fb1a3ba18b21e03d7c27f3436a10606b", size = 212446, upload-time = "2025-09-08T23:23:03.472Z" },
    { url = "https://files.pythonhosted.org/packages/a9/f5/a2c23eb03b61a0b8747f211eb716446c826ad66818ddc7810cc2cc19b3f2/cffi-2.0.0-cp313-cp313-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:d48a880098c96020b02d5a1f7d9251308510ce8858940e6fa99ece33f610838b", size = 220101, upload-time = "2025-09-08T23:23:04.792Z" },
    { url = "https://files.pythonhosted.org/packages/f2/7f/e6647792fc5850d634695bc0e6ab4111ae88e89981d35ac269956605feba/cffi-2.0.0-cp313-cp313-manylinux2014_ppc64le.manylinux_2_17_ppc64le.whl", hash = "sha256:f93fd8e5c8c0a4aa1f424d6173f14a892044054871c771f8566e4008eaa359d2", size = 207948, upload-time = "2025-09-08T23:23:06.127Z" },
    { url = "https://files.pythonhosted.org/packages/cb/1e/a5a1bd6f1fb30f22573f76533de12a00bf274abcdc55c8edab639078abb6/cffi-2.0.0-cp313-cp313-manylinux2014_s390x.manylinux_2_17_s390x.whl", hash = "sha256:dd4f05f54a52fb558f1ba9f528228066954fee3ebe629fc1660d874d040ae5a3", size = 206422, upload-time = "2025-09-08T23:23:07.753Z" },
    { url = "https://files.pythonhosted.org/packages/98/df/0a1755e750013a2081e863e7cd37e0cdd02664372c754e5560099eb7aa44/cffi-2.0.0-cp313-cp313-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:c8d3b5532fc71b7a77c09192b4a5a200ea992702734a2e9279a37f2478236f26", size = 219499, upload-time = "2025-09-08T23:23:09.648Z" },
    { url = "https://files.pythonhosted.org/packages/50/e1/a969e687fcf9ea58e6e2a928ad5e2dd88cc12f6f0ab477e9971f2309b57c/cffi-2.0.0-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:d9b29c1f0ae438d5ee9acb31cadee00a58c46cc9c0b2f9038c6b0b3470877a8c", size = 222928, upload-time = "2025-09-08T23:23:10.928Z" },
    { url = "https://files.pythonhosted.org/packages/36/54/0362578dd2c9e557a28ac77698ed67323ed5b9775ca9d3fe73fe191bb5d8/cffi-2.0.0-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:6d50360be4546678fc1b79ffe7a66265e28667840010348dd69a314145807a1b", size = 221302, upload-time = "2025-09-08T23:23:12.42Z" },
    { url = "https://files.pythonhosted.org/packages/eb/6d/bf9bda840d5f1dfdbf0feca87fbdb64a918a69bca42cfa0ba7b137c48cb8/cffi-2.0.0-cp313-cp313-win32.whl", hash = "sha256:74a03b9698e198d47562765773b4a8309919089150a0bb17d829ad7b44b60d27", size = 172909, upload-time = "2025-09-08T23:23:14.32Z" },
    { url = "https://files.pythonhosted.org/packages/37/18/6519e1ee6f5a1e579e04b9ddb6f1676c17368a7aba48299c3759bbc3c8b3/cffi-2.0.0-cp313-cp313-win_amd64.whl", hash = "sha256:19f705ada2530c1167abacb171925dd886168931e0a7b78f5bffcae5c6b5be75", size = 183402, upload-time = "2025-09-08T23:23:15.535Z" },
    { url = "https://files.pythonhosted.org/packages/cb/0e/02ceeec9a7d6ee63bb596121c2c8e9b3a9e150936f4fbef6ca1943e6137c/cffi-2.0.0-cp313-cp313-win_arm64.whl", hash = "sha256:256f80b80ca3853f90c21b23ee78cd008713787b1b1e93eae9f3d6a7134abd91", size = 177780, upload-time = "2025-09-08T23:23:16.761Z" },
    { url = "https://files.pythonhosted.org/packages/92/c4/3ce07396253a83250ee98564f8d7e9789fab8e58858f35d07a9a2c78de9f/cffi-2.0.0-cp314-cp314-macosx_10_13_x86_64.whl", hash = "sha256:fc33c5141b55ed366cfaad382df24fe7dcbc686de5be719b207bb248e3053dc5", size = 185320, upload-time = "2025-09-08T23:23:18.087Z" },
    { url = "https://files.pythonhosted.org/packages/59/dd/27e9fa567a23931c838c6b02d0764611c62290062a6d4e8ff7863daf9730/cffi-2.0.0-cp314-cp314-macosx_11_0_arm64.whl", hash = "sha256:c654de545946e0db659b3400168c9ad31b5d29593291482c43e3564effbcee13", size = 181487, upload-time = "2025-09-08T23:23:19.622Z" },
    { url = "https://files.pythonhosted.org/packages/d6/43/0e822876f87ea8a4ef95442c3d766a06a51fc5298823f884ef87aaad168c/cffi-2.0.0-cp314-cp314-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:24b6f81f1983e6df8db3adc38562c83f7d4a0c36162885ec7f7b77c7dcbec97b", size = 220049, upload-time = "2025-09-08T23:23:20.853Z" },
    { url = "https://files.pythonhosted.org/packages/b4/89/76799151d9c2d2d1ead63c2429da9ea9d7aac304603de0c6e8764e6e8e70/cffi-2.0.0-cp314-cp314-manylinux2014_ppc64le.manylinux_2_17_ppc64le.whl", hash = "sha256:12873ca6cb9b0f0d3a0da705d6086fe911591737a59f28b7936bdfed27c0d47c", size = 207793, upload-time = "2025-09-08T23:23:22.08Z" },
    { url = "https://files.pythonhosted.org/packages/bb/dd/3465b14bb9e24ee24cb88c9e3730f6de63111fffe513492bf8c808a3547e/cffi-2.0.0-cp314-cp314-manylinux2014_s390x.manylinux_2_17_s390x.whl", hash = "sha256:d9b97165e8aed9272a6bb17c01e3cc5871a594a446ebedc996e2397a1c1ea8ef", size = 206300, upload-time = "2025-09-08T23:23:23.314Z" },
    { url = "https://files.pythonhosted.org/packages/47/d9/d83e293854571c877a92da46fdec39158f8d7e68da75bf73581225d28e90/cffi-2.0.0-cp314-cp314-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:afb8db5439b81cf9c9d0c80404b60c3cc9c3add93e114dcae767f1477cb53775", size = 219244, upload-time = "2025-09-08T23:23:24.541Z" },
    { url = "https://files.pythonhosted.org/packages/2b/0f/1f177e3683aead2bb00f7679a16451d302c436b5cbf2505f0ea8146ef59e/cffi-2.0.0-cp314-cp314-musllinux_1_2_aarch64.whl", hash = "sha256:737fe7d37e1a1bffe70bd5754ea763a62a066dc5913ca57e957824b72a85e205", size = 222828, upload-time = "2025-09-08T23:23:26.143Z" },
    { url = "https://files.pythonhosted.org/packages/c6/0f/cafacebd4b040e3119dcb32fed8bdef8dfe94da653155f9d0b9dc660166e/cffi-2.0.0-cp314-cp314-musllinux_1_2_x86_64.whl", hash = "sha256:38100abb9d1b1435bc4cc340bb4489635dc2f0da7456590877030c9b3d40b0c1", size = 220926, upload-time = "2025-09-08T23:23:27.873Z" },
    { url = "https://files.pythonhosted.org/packages/3e/aa/df335faa45b395396fcbc03de2dfcab242cd61a9900e914fe682a59170b1/cffi-2.0.0-cp314-cp314-win32.whl", hash = "sha256:087067fa8953339c723661eda6b54bc98c5625757ea62e95eb4898ad5e776e9f", size = 175328, upload-time = "2025-09-08T23:23:44.61Z" },
    { url = "https://files.pythonhosted.org/packages/bb/92/882c2d30831744296ce713f0feb4c1cd30f346ef747b530b5318715cc367/cffi-2.0.0-cp314-cp314-win_amd64.whl", hash = "sha256:203a48d1fb583fc7d78a4c6655692963b860a417c0528492a6bc21f1aaefab25", size = 185650, upload-time = "2025-09-08T23:23:45.848Z" },
    { url = "https://files.pythonhosted.org/packages/9f/2c/98ece204b9d35a7366b5b2c6539c350313ca13932143e79dc133ba757104/cffi-2.0.0-cp314-cp314-win_arm64.whl", hash = "sha256:dbd5c7a25a7cb98f5ca55d258b103a2054f859a46ae11aaf23134f9cc0d356ad", size = 180687, upload-time = "2025-09-08T23:23:47.105Z" },
    { url = "https://files.pythonhosted.org/packages/3e/61/c768e4d548bfa607abcda77423448df8c471f25dbe64fb2ef6d555eae006/cffi-2.0.0-cp314-cp314t-macosx_10_13_x86_64.whl", hash = "sha256:9a67fc9e8eb39039280526379fb3a70023d77caec1852002b4da7e8b270c4dd9", size = 188773, upload-time = "2025-09-08T23:23:29.347Z" },
    { url = "https://files.pythonhosted.org/packages/2c/ea/5f76bce7cf6fcd0ab1a1058b5af899bfbef198bea4d5686da88471ea0336/cffi-2.0.0-cp314-cp314t-macosx_11_0_arm64.whl", hash = "sha256:7a66c7204d8869299919db4d5069a82f1561581af12b11b3c9f48c584eb8743d", size = 185013, upload-time = "2025-09-08T23:23:30.63Z" },
    { url = "https://files.pythonhosted.org/packages/be/b4/c56878d0d1755cf9caa54ba71e5d049479c52f9e4afc230f06822162ab2f/cffi-2.0.0-cp314-cp314t-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:7cc09976e8b56f8cebd752f7113ad07752461f48a58cbba644139015ac24954c", size = 221593, upload-time = "2025-09-08T23:23:31.91Z" },
    { url = "https://files.pythonhosted.org/packages/e0/0d/eb704606dfe8033e7128df5e90fee946bbcb64a04fcdaa97321309004000/cffi-2.0.0-cp314-cp314t-manylinux2014_ppc64le.manylinux_2_17_ppc64le.whl", hash = "sha256:92b68146a71df78564e4ef48af17551a5ddd142e5190cdf2c5624d0c3ff5b2e8", size = 209354, upload-time = "2025-09-08T23:23:33.214Z" },
    { url = "https://files.pythonhosted.org/packages/d8/19/3c435d727b368ca475fb8742ab97c9cb13a0de600ce86f62eab7fa3eea60/cffi-2.0.0-cp314-cp314t-manylinux2014_s390x.manylinux_2_17_s390x.whl", hash = "sha256:b1e74d11748e7e98e2f426ab176d4ed720a64412b6a15054378afdb71e0f37dc", size = 208480, upload-time = "2025-09-08T23:23:34.495Z" },
    { url = "https://files.pythonhosted.org/packages/d0/44/681604464ed9541673e486521497406fadcc15b5217c3e326b061696899a/cffi-2.0.0-cp314-cp314t-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:28a3a209b96630bca57cce802da70c266eb08c6e97e5afd61a75611ee6c64592", size = 221584, upload-time = "2025-09-08T23:23:36.096Z" },
    { url = "https://files.pythonhosted.org/packages/25/8e/342a504ff018a2825d395d44d63a767dd8ebc927ebda557fecdaca3ac33a/cffi-2.0.0-cp314-cp314t-musllinux_1_2_aarch64.whl", hash = "sha256:7553fb2090d71822f02c629afe6042c299edf91ba1bf94951165613553984512", size = 224443, upload-time = "2025-09-08T23:23:37.328Z" },
    { url = "https://files.pythonhosted.org/packages/e1/5e/b666bacbbc60fbf415ba9988324a132c9a7a0448a9a8f125074671c0f2c3/cffi-2.0.0-cp314-cp314t-musllinux_1_2_x86_64.whl", hash = "sha256:6c6c373cfc5c83a975506110d17457138c8c63016b563cc9ed6e056a82f13ce4", size = 223437, upload-time = "2025-09-08T23:23:38.945Z" },
    { url = "https://files.pythonhosted.org/packages/a0/1d/ec1a60bd1a10daa292d3cd6bb0b359a81607154fb8165f3ec95fe003b85c/cffi-2.0.0-cp314-cp314t-win32.whl", hash = "sha256:1fc9ea04857caf665289b7a75923f2c6ed559b8298a1b8c49e59f7dd95c8481e", size = 180487, upload-time = "2025-09-08T23:23:40.423Z" },
    { url = "https://files.pythonhosted.org/packages/bf/41/4c1168c74fac325c0c8156f04b6749c8b6a8f405bbf91413ba088359f60d/cffi-2.0.0-cp314-cp314t-win_amd64.whl", hash = "sha256:d68b6cef7827e8641e8ef16f4494edda8b36104d79773a334beaa1e3521430f6", size = 191726, upload-time = "2025-09-08T23:23:41.742Z" },
    { url = "https://files.pythonhosted.org/packages/ae/3a/dbeec9d1ee0844c679f6bb5d6ad4e9f198b1224f4e7a32825f47f6192b0c/cffi-2.0.0-cp314-cp314t-win_arm64.whl", hash = "sha256:0a1527a803f0a659de1af2e1fd700213caba79377e27e4693648c2923da066f9", size = 184195, upload-time = "2025-09-08T23:23:43.004Z" },
]

[[package]]
name = "charset-normalizer"
version = "3.4.4"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/13/69/33ddede1939fdd074bce5434295f38fae7136463422fe4fd3e0e89b98062/charset_normalizer-3.4.4.tar.gz", hash = "sha256:94537985111c35f28720e43603b8e7b43a6ecfb2ce1d3058bbe955b73404e21a", size = 129418, upload-time = "2025-10-14T04:42:32.879Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/1f/b8/6d51fc1d52cbd52cd4ccedd5b5b2f0f6a11bbf6765c782298b0f3e808541/charset_normalizer-3.4.4-cp310-cp310-macosx_10_9_universal2.whl", hash = "sha256:e824f1492727fa856dd6eda4f7cee25f8518a12f3c4a56a74e8095695089cf6d", size = 209709, upload-time = "2025-10-14T04:40:11.385Z" },
    { url = "https://files.pythonhosted.org/packages/5c/af/1f9d7f7faafe2ddfb6f72a2e07a548a629c61ad510fe60f9630309908fef/charset_normalizer-3.4.4-cp310-cp310-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:4bd5d4137d500351a30687c2d3971758aac9a19208fc110ccb9d7188fbe709e8", size = 148814, upload-time = "2025-10-14T04:40:13.135Z" },
    { url = "https://files.pythonhosted.org/packages/79/3d/f2e3ac2bbc056ca0c204298ea4e3d9db9b4afe437812638759db2c976b5f/charset_normalizer-3.4.4-cp310-cp310-manylinux2014_armv7l.manylinux_2_17_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:027f6de494925c0ab2a55eab46ae5129951638a49a34d87f4c3eda90f696b4ad", size = 144467, upload-time = "2025-10-14T04:40:14.728Z" },
    { url = "https://files.pythonhosted.org/packages/ec/85/1bf997003815e60d57de7bd972c57dc6950446a3e4ccac43bc3070721856/charset_normalizer-3.4.4-cp310-cp310-manylinux2014_ppc64le.manylinux_2_17_ppc64le.manylinux_2_28_ppc64le.whl", hash = "sha256:f820802628d2694cb7e56db99213f930856014862f3fd943d290ea8438d07ca8", size = 162280, upload-time = "2025-10-14T04:40:16.14Z" },
    { url = "https://files.pythonhosted.org/packages/3e/8e/6aa1952f56b192f54921c436b87f2aaf7c7a7c3d0d1a765547d64fd83c13/charset_normalizer-3.4.4-cp310-cp310-manylinux2014_s390x.manylinux_2_17_s390x.manylinux_2_28_s390x.whl", hash = "sha256:798d75d81754988d2565bff1b97ba5a44411867c0cf32b77a7e8f8d84796b10d", size = 159454, upload-time = "2025-10-14T04:40:17.567Z" },
    { url = "https://files.pythonhosted.org/packages/36/3b/60cbd1f8e93aa25d1c669c649b7a655b0b5fb4c571858910ea9332678558/charset_normalizer-3.4.4-cp310-cp310-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:9d1bb833febdff5c8927f922386db610b49db6e0d4f4ee29601d71e7c2694313", size = 153609, upload-time = "2025-10-14T04:40:19.08Z" },
    { url = "https://files.pythonhosted.org/packages/64/91/6a13396948b8fd3c4b4fd5bc74d045f5637d78c9675585e8e9fbe5636554/charset_normalizer-3.4.4-cp310-cp310-manylinux_2_31_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:9cd98cdc06614a2f768d2b7286d66805f94c48cde050acdbbb7db2600ab3197e", size = 151849, upload-time = "2025-10-14T04:40:20.607Z" },
    { url = "https://files.pythonhosted.org/packages/b7/7a/59482e28b9981d105691e968c544cc0df3b7d6133152fb3dcdc8f135da7a/charset_normalizer-3.4.4-cp310-cp310-musllinux_1_2_aarch64.whl", hash = "sha256:077fbb858e903c73f6c9db43374fd213b0b6a778106bc7032446a8e8b5b38b93", size = 151586, upload-time = "2025-10-14T04:40:21.719Z" },
    { url = "https://files.pythonhosted.org/packages/92/59/f64ef6a1c4bdd2baf892b04cd78792ed8684fbc48d4c2afe467d96b4df57/charset_normalizer-3.4.4-cp310-cp310-musllinux_1_2_armv7l.whl", hash = "sha256:244bfb999c71b35de57821b8ea746b24e863398194a4014e4c76adc2bbdfeff0", size = 145290, upload-time = "2025-10-14T04:40:23.069Z" },
    { url = "https://files.pythonhosted.org/packages/6b/63/3bf9f279ddfa641ffa1962b0db6a57a9c294361cc2f5fcac997049a00e9c/charset_normalizer-3.4.4-cp310-cp310-musllinux_1_2_ppc64le.whl", hash = "sha256:64b55f9dce520635f018f907ff1b0df1fdc31f2795a922fb49dd14fbcdf48c84", size = 163663, upload-time = "2025-10-14T04:40:24.17Z" },
    { url = "https://files.pythonhosted.org/packages/ed/09/c9e38fc8fa9e0849b172b581fd9803bdf6e694041127933934184e19f8c3/charset_normalizer-3.4.4-cp310-cp310-musllinux_1_2_riscv64.whl", hash = "sha256:faa3a41b2b66b6e50f84ae4a68c64fcd0c44355741c6374813a800cd6695db9e", size = 151964, upload-time = "2025-10-14T04:40:25.368Z" },
    { url = "https://files.pythonhosted.org/packages/d2/d1/d28b747e512d0da79d8b6a1ac18b7ab2ecfd81b2944c4c710e166d8dd09c/charset_normalizer-3.4.4-cp310-cp310-musllinux_1_2_s390x.whl", hash = "sha256:6515f3182dbe4ea06ced2d9e8666d97b46ef4c75e326b79bb624110f122551db", size = 161064, upload-time = "2025-10-14T04:40:26.806Z" },
    { url = "https://files.pythonhosted.org/packages/bb/9a/31d62b611d901c3b9e5500c36aab0ff5eb442043fb3a1c254200d3d397d9/charset_normalizer-3.4.4-cp310-cp310-musllinux_1_2_x86_64.whl", hash = "sha256:cc00f04ed596e9dc0da42ed17ac5e596c6ccba999ba6bd92b0e0aef2f170f2d6", size = 155015, upload-time = "2025-10-14T04:40:28.284Z" },
    { url = "https://files.pythonhosted.org/packages/1f/f3/107e008fa2bff0c8b9319584174418e5e5285fef32f79d8ee6a430d0039c/charset_normalizer-3.4.4-cp310-cp310-win32.whl", hash = "sha256:f34be2938726fc13801220747472850852fe6b1ea75869a048d6f896838c896f", size = 99792, upload-time = "2025-10-14T04:40:29.613Z" },
    { url = "https://files.pythonhosted.org/packages/eb/66/e396e8a408843337d7315bab30dbf106c38966f1819f123257f5520f8a96/charset_normalizer-3.4.4-cp310-cp310-win_amd64.whl", hash = "sha256:a61900df84c667873b292c3de315a786dd8dac506704dea57bc957bd31e22c7d", size = 107198, upload-time = "2025-10-14T04:40:30.644Z" },
    { url = "https://files.pythonhosted.org/packages/b5/58/01b4f815bf0312704c267f2ccb6e5d42bcc7752340cd487bc9f8c3710597/charset_normalizer-3.4.4-cp310-cp310-win_arm64.whl", hash = "sha256:cead0978fc57397645f12578bfd2d5ea9138ea0fac82b2f63f7f7c6877986a69", size = 100262, upload-time = "2025-10-14T04:40:32.108Z" },
    { url = "https://files.pythonhosted.org/packages/ed/27/c6491ff4954e58a10f69ad90aca8a1b6fe9c5d3c6f380907af3c37435b59/charset_normalizer-3.4.4-cp311-cp311-macosx_10_9_universal2.whl", hash = "sha256:6e1fcf0720908f200cd21aa4e6750a48ff6ce4afe7ff5a79a90d5ed8a08296f8", size = 206988, upload-time = "2025-10-14T04:40:33.79Z" },
    { url = "https://files.pythonhosted.org/packages/94/59/2e87300fe67ab820b5428580a53cad894272dbb97f38a7a814a2a1ac1011/charset_normalizer-3.4.4-cp311-cp311-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:5f819d5fe9234f9f82d75bdfa9aef3a3d72c4d24a6e57aeaebba32a704553aa0", size = 147324, upload-time = "2025-10-14T04:40:34.961Z" },
    { url = "https://files.pythonhosted.org/packages/07/fb/0cf61dc84b2b088391830f6274cb57c82e4da8bbc2efeac8c025edb88772/charset_normalizer-3.4.4-cp311-cp311-manylinux2014_armv7l.manylinux_2_17_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:a59cb51917aa591b1c4e6a43c132f0cdc3c76dbad6155df4e28ee626cc77a0a3", size = 142742, upload-time = "2025-10-14T04:40:36.105Z" },
    { url = "https://files.pythonhosted.org/packages/62/8b/171935adf2312cd745d290ed93cf16cf0dfe320863ab7cbeeae1dcd6535f/charset_normalizer-3.4.4-cp311-cp311-manylinux2014_ppc64le.manylinux_2_17_ppc64le.manylinux_2_28_ppc64le.whl", hash = "sha256:8ef3c867360f88ac904fd3f5e1f902f13307af9052646963ee08ff4f131adafc", size = 160863, upload-time = "2025-10-14T04:40:37.188Z" },
    { url = "https://files.pythonhosted.org/packages/09/73/ad875b192bda14f2173bfc1bc9a55e009808484a4b256748d931b6948442/charset_normalizer-3.4.4-cp311-cp311-manylinux2014_s390x.manylinux_2_17_s390x.manylinux_2_28_s390x.whl", hash = "sha256:d9e45d7faa48ee908174d8fe84854479ef838fc6a705c9315372eacbc2f02897", size = 157837, upload-time = "2025-10-14T04:40:38.435Z" },
    { url = "https://files.pythonhosted.org/packages/6d/fc/de9cce525b2c5b94b47c70a4b4fb19f871b24995c728e957ee68ab1671ea/charset_normalizer-3.4.4-cp311-cp311-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:840c25fb618a231545cbab0564a799f101b63b9901f2569faecd6b222ac72381", size = 151550, upload-time = "2025-10-14T04:40:40.053Z" },
    { url = "https://files.pythonhosted.org/packages/55/c2/43edd615fdfba8c6f2dfbd459b25a6b3b551f24ea21981e23fb768503ce1/charset_normalizer-3.4.4-cp311-cp311-manylinux_2_31_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:ca5862d5b3928c4940729dacc329aa9102900382fea192fc5e52eb69d6093815", size = 149162, upload-time = "2025-10-14T04:40:41.163Z" },
    { url = "https://files.pythonhosted.org/packages/03/86/bde4ad8b4d0e9429a4e82c1e8f5c659993a9a863ad62c7df05cf7b678d75/charset_normalizer-3.4.4-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:d9c7f57c3d666a53421049053eaacdd14bbd0a528e2186fcb2e672effd053bb0", size = 150019, upload-time = "2025-10-14T04:40:42.276Z" },
    { url = "https://files.pythonhosted.org/packages/1f/86/a151eb2af293a7e7bac3a739b81072585ce36ccfb4493039f49f1d3cae8c/charset_normalizer-3.4.4-cp311-cp311-musllinux_1_2_armv7l.whl", hash = "sha256:277e970e750505ed74c832b4bf75dac7476262ee2a013f5574dd49075879e161", size = 143310, upload-time = "2025-10-14T04:40:43.439Z" },
    { url = "https://files.pythonhosted.org/packages/b5/fe/43dae6144a7e07b87478fdfc4dbe9efd5defb0e7ec29f5f58a55aeef7bf7/charset_normalizer-3.4.4-cp311-cp311-musllinux_1_2_ppc64le.whl", hash = "sha256:31fd66405eaf47bb62e8cd575dc621c56c668f27d46a61d975a249930dd5e2a4", size = 162022, upload-time = "2025-10-14T04:40:44.547Z" },
    { url = "https://files.pythonhosted.org/packages/80/e6/7aab83774f5d2bca81f42ac58d04caf44f0cc2b65fc6db2b3b2e8a05f3b3/charset_normalizer-3.4.4-cp311-cp311-musllinux_1_2_riscv64.whl", hash = "sha256:0d3d8f15c07f86e9ff82319b3d9ef6f4bf907608f53fe9d92b28ea9ae3d1fd89", size = 149383, upload-time = "2025-10-14T04:40:46.018Z" },
    { url = "https://files.pythonhosted.org/packages/4f/e8/b289173b4edae05c0dde07f69f8db476a0b511eac556dfe0d6bda3c43384/charset_normalizer-3.4.4-cp311-cp311-musllinux_1_2_s390x.whl", hash = "sha256:9f7fcd74d410a36883701fafa2482a6af2ff5ba96b9a620e9e0721e28ead5569", size = 159098, upload-time = "2025-10-14T04:40:47.081Z" },
    { url = "https://files.pythonhosted.org/packages/d8/df/fe699727754cae3f8478493c7f45f777b17c3ef0600e28abfec8619eb49c/charset_normalizer-3.4.4-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:ebf3e58c7ec8a8bed6d66a75d7fb37b55e5015b03ceae72a8e7c74495551e224", size = 152991, upload-time = "2025-10-14T04:40:48.246Z" },
    { url = "https://files.pythonhosted.org/packages/1a/86/584869fe4ddb6ffa3bd9f491b87a01568797fb9bd8933f557dba9771beaf/charset_normalizer-3.4.4-cp311-cp311-win32.whl", hash = "sha256:eecbc200c7fd5ddb9a7f16c7decb07b566c29fa2161a16cf67b8d068bd21690a", size = 99456, upload-time = "2025-10-14T04:40:49.376Z" },
    { url = "https://files.pythonhosted.org/packages/65/f6/62fdd5feb60530f50f7e38b4f6a1d5203f4d16ff4f9f0952962c044e919a/charset_normalizer-3.4.4-cp311-cp311-win_amd64.whl", hash = "sha256:5ae497466c7901d54b639cf42d5b8c1b6a4fead55215500d2f486d34db48d016", size = 106978, upload-time = "2025-10-14T04:40:50.844Z" },
    { url = "https://files.pythonhosted.org/packages/7a/9d/0710916e6c82948b3be62d9d398cb4fcf4e97b56d6a6aeccd66c4b2f2bd5/charset_normalizer-3.4.4-cp311-cp311-win_arm64.whl", hash = "sha256:65e2befcd84bc6f37095f5961e68a6f077bf44946771354a28ad434c2cce0ae1", size = 99969, upload-time = "2025-10-14T04:40:52.272Z" },
    { url = "https://files.pythonhosted.org/packages/f3/85/1637cd4af66fa687396e757dec650f28025f2a2f5a5531a3208dc0ec43f2/charset_normalizer-3.4.4-cp312-cp312-macosx_10_13_universal2.whl", hash = "sha256:0a98e6759f854bd25a58a73fa88833fba3b7c491169f86ce1180c948ab3fd394", size = 208425, upload-time = "2025-10-14T04:40:53.353Z" },
    { url = "https://files.pythonhosted.org/packages/9d/6a/04130023fef2a0d9c62d0bae2649b69f7b7d8d24ea5536feef50551029df/charset_normalizer-3.4.4-cp312-cp312-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:b5b290ccc2a263e8d185130284f8501e3e36c5e02750fc6b6bdeb2e9e96f1e25", size = 148162, upload-time = "2025-10-14T04:40:54.558Z" },
    { url = "https://files.pythonhosted.org/packages/78/29/62328d79aa60da22c9e0b9a66539feae06ca0f5a4171ac4f7dc285b83688/charset_normalizer-3.4.4-cp312-cp312-manylinux2014_armv7l.manylinux_2_17_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:74bb723680f9f7a6234dcf67aea57e708ec1fbdf5699fb91dfd6f511b0a320ef", size = 144558, upload-time = "2025-10-14T04:40:55.677Z" },
    { url = "https://files.pythonhosted.org/packages/86/bb/b32194a4bf15b88403537c2e120b817c61cd4ecffa9b6876e941c3ee38fe/charset_normalizer-3.4.4-cp312-cp312-manylinux2014_ppc64le.manylinux_2_17_ppc64le.manylinux_2_28_ppc64le.whl", hash = "sha256:f1e34719c6ed0b92f418c7c780480b26b5d9c50349e9a9af7d76bf757530350d", size = 161497, upload-time = "2025-10-14T04:40:57.217Z" },
    { url = "https://files.pythonhosted.org/packages/19/89/a54c82b253d5b9b111dc74aca196ba5ccfcca8242d0fb64146d4d3183ff1/charset_normalizer-3.4.4-cp312-cp312-manylinux2014_s390x.manylinux_2_17_s390x.manylinux_2_28_s390x.whl", hash = "sha256:2437418e20515acec67d86e12bf70056a33abdacb5cb1655042f6538d6b085a8", size = 159240, upload-time = "2025-10-14T04:40:58.358Z" },
    { url = "https://files.pythonhosted.org/packages/c0/10/d20b513afe03acc89ec33948320a5544d31f21b05368436d580dec4e234d/charset_normalizer-3.4.4-cp312-cp312-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:11d694519d7f29d6cd09f6ac70028dba10f92f6cdd059096db198c283794ac86", size = 153471, upload-time = "2025-10-14T04:40:59.468Z" },
    { url = "https://files.pythonhosted.org/packages/61/fa/fbf177b55bdd727010f9c0a3c49eefa1d10f960e5f09d1d887bf93c2e698/charset_normalizer-3.4.4-cp312-cp312-manylinux_2_31_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:ac1c4a689edcc530fc9d9aa11f5774b9e2f33f9a0c6a57864e90908f5208d30a", size = 150864, upload-time = "2025-10-14T04:41:00.623Z" },
    { url = "https://files.pythonhosted.org/packages/05/12/9fbc6a4d39c0198adeebbde20b619790e9236557ca59fc40e0e3cebe6f40/charset_normalizer-3.4.4-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:21d142cc6c0ec30d2efee5068ca36c128a30b0f2c53c1c07bd78cb6bc1d3be5f", size = 150647, upload-time = "2025-10-14T04:41:01.754Z" },
    { url = "https://files.pythonhosted.org/packages/ad/1f/6a9a593d52e3e8c5d2b167daf8c6b968808efb57ef4c210acb907c365bc4/charset_normalizer-3.4.4-cp312-cp312-musllinux_1_2_armv7l.whl", hash = "sha256:5dbe56a36425d26d6cfb40ce79c314a2e4dd6211d51d6d2191c00bed34f354cc", size = 145110, upload-time = "2025-10-14T04:41:03.231Z" },
    { url = "https://files.pythonhosted.org/packages/30/42/9a52c609e72471b0fc54386dc63c3781a387bb4fe61c20231a4ebcd58bdd/charset_normalizer-3.4.4-cp312-cp312-musllinux_1_2_ppc64le.whl", hash = "sha256:5bfbb1b9acf3334612667b61bd3002196fe2a1eb4dd74d247e0f2a4d50ec9bbf", size = 162839, upload-time = "2025-10-14T04:41:04.715Z" },
    { url = "https://files.pythonhosted.org/packages/c4/5b/c0682bbf9f11597073052628ddd38344a3d673fda35a36773f7d19344b23/charset_normalizer-3.4.4-cp312-cp312-musllinux_1_2_riscv64.whl", hash = "sha256:d055ec1e26e441f6187acf818b73564e6e6282709e9bcb5b63f5b23068356a15", size = 150667, upload-time = "2025-10-14T04:41:05.827Z" },
    { url = "https://files.pythonhosted.org/packages/e4/24/a41afeab6f990cf2daf6cb8c67419b63b48cf518e4f56022230840c9bfb2/charset_normalizer-3.4.4-cp312-cp312-musllinux_1_2_s390x.whl", hash = "sha256:af2d8c67d8e573d6de5bc30cdb27e9b95e49115cd9baad5ddbd1a6207aaa82a9", size = 160535, upload-time = "2025-10-14T04:41:06.938Z" },
    { url = "https://files.pythonhosted.org/packages/2a/e5/6a4ce77ed243c4a50a1fecca6aaaab419628c818a49434be428fe24c9957/charset_normalizer-3.4.4-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:780236ac706e66881f3b7f2f32dfe90507a09e67d1d454c762cf642e6e1586e0", size = 154816, upload-time = "2025-10-14T04:41:08.101Z" },
    { url = "https://files.pythonhosted.org/packages/a8/ef/89297262b8092b312d29cdb2517cb1237e51db8ecef2e9af5edbe7b683b1/charset_normalizer-3.4.4-cp312-cp312-win32.whl", hash = "sha256:5833d2c39d8896e4e19b689ffc198f08ea58116bee26dea51e362ecc7cd3ed26", size = 99694, upload-time = "2025-10-14T04:41:09.23Z" },
    { url = "https://files.pythonhosted.org/packages/3d/2d/1e5ed9dd3b3803994c155cd9aacb60c82c331bad84daf75bcb9c91b3295e/charset_normalizer-3.4.4-cp312-cp312-win_amd64.whl", hash = "sha256:a79cfe37875f822425b89a82333404539ae63dbdddf97f84dcbc3d339aae9525", size = 107131, upload-time = "2025-10-14T04:41:10.467Z" },
    { url = "https://files.pythonhosted.org/packages/d0/d9/0ed4c7098a861482a7b6a95603edce4c0d9db2311af23da1fb2b75ec26fc/charset_normalizer-3.4.4-cp312-cp312-win_arm64.whl", hash = "sha256:376bec83a63b8021bb5c8ea75e21c4ccb86e7e45ca4eb81146091b56599b80c3", size = 100390, upload-time = "2025-10-14T04:41:11.915Z" },
    { url = "https://files.pythonhosted.org/packages/97/45/4b3a1239bbacd321068ea6e7ac28875b03ab8bc0aa0966452db17cd36714/charset_normalizer-3.4.4-cp313-cp313-macosx_10_13_universal2.whl", hash = "sha256:e1f185f86a6f3403aa2420e815904c67b2f9ebc443f045edd0de921108345794", size = 208091, upload-time = "2025-10-14T04:41:13.346Z" },
    { url = "https://files.pythonhosted.org/packages/7d/62/73a6d7450829655a35bb88a88fca7d736f9882a27eacdca2c6d505b57e2e/charset_normalizer-3.4.4-cp313-cp313-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:6b39f987ae8ccdf0d2642338faf2abb1862340facc796048b604ef14919e55ed", size = 147936, upload-time = "2025-10-14T04:41:14.461Z" },
    { url = "https://files.pythonhosted.org/packages/89/c5/adb8c8b3d6625bef6d88b251bbb0d95f8205831b987631ab0c8bb5d937c2/charset_normalizer-3.4.4-cp313-cp313-manylinux2014_armv7l.manylinux_2_17_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:3162d5d8ce1bb98dd51af660f2121c55d0fa541b46dff7bb9b9f86ea1d87de72", size = 144180, upload-time = "2025-10-14T04:41:15.588Z" },
    { url = "https://files.pythonhosted.org/packages/91/ed/9706e4070682d1cc219050b6048bfd293ccf67b3d4f5a4f39207453d4b99/charset_normalizer-3.4.4-cp313-cp313-manylinux2014_ppc64le.manylinux_2_17_ppc64le.manylinux_2_28_ppc64le.whl", hash = "sha256:81d5eb2a312700f4ecaa977a8235b634ce853200e828fbadf3a9c50bab278328", size = 161346, upload-time = "2025-10-14T04:41:16.738Z" },
    { url = "https://files.pythonhosted.org/packages/d5/0d/031f0d95e4972901a2f6f09ef055751805ff541511dc1252ba3ca1f80cf5/charset_normalizer-3.4.4-cp313-cp313-manylinux2014_s390x.manylinux_2_17_s390x.manylinux_2_28_s390x.whl", hash = "sha256:5bd2293095d766545ec1a8f612559f6b40abc0eb18bb2f5d1171872d34036ede", size = 158874, upload-time = "2025-10-14T04:41:17.923Z" },
    { url = "https://files.pythonhosted.org/packages/f5/83/6ab5883f57c9c801ce5e5677242328aa45592be8a00644310a008d04f922/charset_normalizer-3.4.4-cp313-cp313-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:a8a8b89589086a25749f471e6a900d3f662d1d3b6e2e59dcecf787b1cc3a1894", size = 153076, upload-time = "2025-10-14T04:41:19.106Z" },
    { url = "https://files.pythonhosted.org/packages/75/1e/5ff781ddf5260e387d6419959ee89ef13878229732732ee73cdae01800f2/charset_normalizer-3.4.4-cp313-cp313-manylinux_2_31_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:bc7637e2f80d8530ee4a78e878bce464f70087ce73cf7c1caf142416923b98f1", size = 150601, upload-time = "2025-10-14T04:41:20.245Z" },
    { url = "https://files.pythonhosted.org/packages/d7/57/71be810965493d3510a6ca79b90c19e48696fb1ff964da319334b12677f0/charset_normalizer-3.4.4-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:f8bf04158c6b607d747e93949aa60618b61312fe647a6369f88ce2ff16043490", size = 150376, upload-time = "2025-10-14T04:41:21.398Z" },
    { url = "https://files.pythonhosted.org/packages/e5/d5/c3d057a78c181d007014feb7e9f2e65905a6c4ef182c0ddf0de2924edd65/charset_normalizer-3.4.4-cp313-cp313-musllinux_1_2_armv7l.whl", hash = "sha256:554af85e960429cf30784dd47447d5125aaa3b99a6f0683589dbd27e2f45da44", size = 144825, upload-time = "2025-10-14T04:41:22.583Z" },
    { url = "https://files.pythonhosted.org/packages/e6/8c/d0406294828d4976f275ffbe66f00266c4b3136b7506941d87c00cab5272/charset_normalizer-3.4.4-cp313-cp313-musllinux_1_2_ppc64le.whl", hash = "sha256:74018750915ee7ad843a774364e13a3db91682f26142baddf775342c3f5b1133", size = 162583, upload-time = "2025-10-14T04:41:23.754Z" },
    { url = "https://files.pythonhosted.org/packages/d7/24/e2aa1f18c8f15c4c0e932d9287b8609dd30ad56dbe41d926bd846e22fb8d/charset_normalizer-3.4.4-cp313-cp313-musllinux_1_2_riscv64.whl", hash = "sha256:c0463276121fdee9c49b98908b3a89c39be45d86d1dbaa22957e38f6321d4ce3", size = 150366, upload-time = "2025-10-14T04:41:25.27Z" },
    { url = "https://files.pythonhosted.org/packages/e4/5b/1e6160c7739aad1e2df054300cc618b06bf784a7a164b0f238360721ab86/charset_normalizer-3.4.4-cp313-cp313-musllinux_1_2_s390x.whl", hash = "sha256:362d61fd13843997c1c446760ef36f240cf81d3ebf74ac62652aebaf7838561e", size = 160300, upload-time = "2025-10-14T04:41:26.725Z" },
    { url = "https://files.pythonhosted.org/packages/7a/10/f882167cd207fbdd743e55534d5d9620e095089d176d55cb22d5322f2afd/charset_normalizer-3.4.4-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:9a26f18905b8dd5d685d6d07b0cdf98a79f3c7a918906af7cc143ea2e164c8bc", size = 154465, upload-time = "2025-10-14T04:41:28.322Z" },
    { url = "https://files.pythonhosted.org/packages/89/66/c7a9e1b7429be72123441bfdbaf2bc13faab3f90b933f664db506dea5915/charset_normalizer-3.4.4-cp313-cp313-win32.whl", hash = "sha256:9b35f4c90079ff2e2edc5b26c0c77925e5d2d255c42c74fdb70fb49b172726ac", size = 99404, upload-time = "2025-10-14T04:41:29.95Z" },
    { url = "https://files.pythonhosted.org/packages/c4/26/b9924fa27db384bdcd97ab83b4f0a8058d96ad9626ead570674d5e737d90/charset_normalizer-3.4.4-cp313-cp313-win_amd64.whl", hash = "sha256:b435cba5f4f750aa6c0a0d92c541fb79f69a387c91e61f1795227e4ed9cece14", size = 107092, upload-time = "2025-10-14T04:41:31.188Z" },
    { url = "https://files.pythonhosted.org/packages/af/8f/3ed4bfa0c0c72a7ca17f0380cd9e4dd842b09f664e780c13cff1dcf2ef1b/charset_normalizer-3.4.4-cp313-cp313-win_arm64.whl", hash = "sha256:542d2cee80be6f80247095cc36c418f7bddd14f4a6de45af91dfad36d817bba2", size = 100408, upload-time = "2025-10-14T04:41:32.624Z" },
    { url = "https://files.pythonhosted.org/packages/2a/35/7051599bd493e62411d6ede36fd5af83a38f37c4767b92884df7301db25d/charset_normalizer-3.4.4-cp314-cp314-macosx_10_13_universal2.whl", hash = "sha256:da3326d9e65ef63a817ecbcc0df6e94463713b754fe293eaa03da99befb9a5bd", size = 207746, upload-time = "2025-10-14T04:41:33.773Z" },
    { url = "https://files.pythonhosted.org/packages/10/9a/97c8d48ef10d6cd4fcead2415523221624bf58bcf68a802721a6bc807c8f/charset_normalizer-3.4.4-cp314-cp314-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:8af65f14dc14a79b924524b1e7fffe304517b2bff5a58bf64f30b98bbc5079eb", size = 147889, upload-time = "2025-10-14T04:41:34.897Z" },
    { url = "https://files.pythonhosted.org/packages/10/bf/979224a919a1b606c82bd2c5fa49b5c6d5727aa47b4312bb27b1734f53cd/charset_normalizer-3.4.4-cp314-cp314-manylinux2014_armv7l.manylinux_2_17_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:74664978bb272435107de04e36db5a9735e78232b85b77d45cfb38f758efd33e", size = 143641, upload-time = "2025-10-14T04:41:36.116Z" },
    { url = "https://files.pythonhosted.org/packages/ba/33/0ad65587441fc730dc7bd90e9716b30b4702dc7b617e6ba4997dc8651495/charset_normalizer-3.4.4-cp314-cp314-manylinux2014_ppc64le.manylinux_2_17_ppc64le.manylinux_2_28_ppc64le.whl", hash = "sha256:752944c7ffbfdd10c074dc58ec2d5a8a4cd9493b314d367c14d24c17684ddd14", size = 160779, upload-time = "2025-10-14T04:41:37.229Z" },
    { url = "https://files.pythonhosted.org/packages/67/ed/331d6b249259ee71ddea93f6f2f0a56cfebd46938bde6fcc6f7b9a3d0e09/charset_normalizer-3.4.4-cp314-cp314-manylinux2014_s390x.manylinux_2_17_s390x.manylinux_2_28_s390x.whl", hash = "sha256:d1f13550535ad8cff21b8d757a3257963e951d96e20ec82ab44bc64aeb62a191", size = 159035, upload-time = "2025-10-14T04:41:38.368Z" },
    { url = "https://files.pythonhosted.org/packages/67/ff/f6b948ca32e4f2a4576aa129d8bed61f2e0543bf9f5f2b7fc3758ed005c9/charset_normalizer-3.4.4-cp314-cp314-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:ecaae4149d99b1c9e7b88bb03e3221956f68fd6d50be2ef061b2381b61d20838", size = 152542, upload-time = "2025-10-14T04:41:39.862Z" },
    { url = "https://files.pythonhosted.org/packages/16/85/276033dcbcc369eb176594de22728541a925b2632f9716428c851b149e83/charset_normalizer-3.4.4-cp314-cp314-manylinux_2_31_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:cb6254dc36b47a990e59e1068afacdcd02958bdcce30bb50cc1700a8b9d624a6", size = 149524, upload-time = "2025-10-14T04:41:41.319Z" },
    { url = "https://files.pythonhosted.org/packages/9e/f2/6a2a1f722b6aba37050e626530a46a68f74e63683947a8acff92569f979a/charset_normalizer-3.4.4-cp314-cp314-musllinux_1_2_aarch64.whl", hash = "sha256:c8ae8a0f02f57a6e61203a31428fa1d677cbe50c93622b4149d5c0f319c1d19e", size = 150395, upload-time = "2025-10-14T04:41:42.539Z" },
    { url = "https://files.pythonhosted.org/packages/60/bb/2186cb2f2bbaea6338cad15ce23a67f9b0672929744381e28b0592676824/charset_normalizer-3.4.4-cp314-cp314-musllinux_1_2_armv7l.whl", hash = "sha256:47cc91b2f4dd2833fddaedd2893006b0106129d4b94fdb6af1f4ce5a9965577c", size = 143680, upload-time = "2025-10-14T04:41:43.661Z" },
    { url = "https://files.pythonhosted.org/packages/7d/a5/bf6f13b772fbb2a90360eb620d52ed8f796f3c5caee8398c3b2eb7b1c60d/charset_normalizer-3.4.4-cp314-cp314-musllinux_1_2_ppc64le.whl", hash = "sha256:82004af6c302b5d3ab2cfc4cc5f29db16123b1a8417f2e25f9066f91d4411090", size = 162045, upload-time = "2025-10-14T04:41:44.821Z" },
    { url = "https://files.pythonhosted.org/packages/df/c5/d1be898bf0dc3ef9030c3825e5d3b83f2c528d207d246cbabe245966808d/charset_normalizer-3.4.4-cp314-cp314-musllinux_1_2_riscv64.whl", hash = "sha256:2b7d8f6c26245217bd2ad053761201e9f9680f8ce52f0fcd8d0755aeae5b2152", size = 149687, upload-time = "2025-10-14T04:41:46.442Z" },
    { url = "https://files.pythonhosted.org/packages/a5/42/90c1f7b9341eef50c8a1cb3f098ac43b0508413f33affd762855f67a410e/charset_normalizer-3.4.4-cp314-cp314-musllinux_1_2_s390x.whl", hash = "sha256:799a7a5e4fb2d5898c60b640fd4981d6a25f1c11790935a44ce38c54e985f828", size = 160014, upload-time = "2025-10-14T04:41:47.631Z" },
    { url = "https://files.pythonhosted.org/packages/76/be/4d3ee471e8145d12795ab655ece37baed0929462a86e72372fd25859047c/charset_normalizer-3.4.4-cp314-cp314-musllinux_1_2_x86_64.whl", hash = "sha256:99ae2cffebb06e6c22bdc25801d7b30f503cc87dbd283479e7b606f70aff57ec", size = 154044, upload-time = "2025-10-14T04:41:48.81Z" },
    { url = "https://files.pythonhosted.org/packages/b0/6f/8f7af07237c34a1defe7defc565a9bc1807762f672c0fde711a4b22bf9c0/charset_normalizer-3.4.4-cp314-cp314-win32.whl", hash = "sha256:f9d332f8c2a2fcbffe1378594431458ddbef721c1769d78e2cbc06280d8155f9", size = 99940, upload-time = "2025-10-14T04:41:49.946Z" },
    { url = "https://files.pythonhosted.org/packages/4b/51/8ade005e5ca5b0d80fb4aff72a3775b325bdc3d27408c8113811a7cbe640/charset_normalizer-3.4.4-cp314-cp314-win_amd64.whl", hash = "sha256:8a6562c3700cce886c5be75ade4a5db4214fda19fede41d9792d100288d8f94c", size = 107104, upload-time = "2025-10-14T04:41:51.051Z" },
    { url = "https://files.pythonhosted.org/packages/da/5f/6b8f83a55bb8278772c5ae54a577f3099025f9ade59d0136ac24a0df4bde/charset_normalizer-3.4.4-cp314-cp314-win_arm64.whl", hash = "sha256:de00632ca48df9daf77a2c65a484531649261ec9f25489917f09e455cb09ddb2", size = 100743, upload-time = "2025-10-14T04:41:52.122Z" },
    { url = "https://files.pythonhosted.org/packages/0a/4c/925909008ed5a988ccbb72dcc897407e5d6d3bd72410d69e051fc0c14647/charset_normalizer-3.4.4-py3-none-any.whl", hash = "sha256:7a32c560861a02ff789ad905a2fe94e3f840803362c84fecf1851cb4cf3dc37f", size = 53402, upload-time = "2025-10-14T04:42:31.76Z" },
]

[[package]]
name = "click"
version = "8.3.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "colorama", marker = "python_full_version < '3.14' and sys_platform == 'win32'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/46/61/de6cd827efad202d7057d93e0fed9294b96952e188f7384832791c7b2254/click-8.3.0.tar.gz", hash = "sha256:e7b8232224eba16f4ebe410c25ced9f7875cb5f3263ffc93cc3e8da705e229c4", size = 276943, upload-time = "2025-09-18T17:32:23.696Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/db/d3/9dcc0f5797f070ec8edf30fbadfb200e71d9db6b84d211e3b2085a7589a0/click-8.3.0-py3-none-any.whl", hash = "sha256:9b9f285302c6e3064f4330c05f05b81945b2a39544279343e6e7c5f27a9baddc", size = 107295, upload-time = "2025-09-18T17:32:22.42Z" },
]

[[package]]
name = "cloudpickle"
version = "3.1.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/52/39/069100b84d7418bc358d81669d5748efb14b9cceacd2f9c75f550424132f/cloudpickle-3.1.1.tar.gz", hash = "sha256:b216fa8ae4019d5482a8ac3c95d8f6346115d8835911fd4aefd1a445e4242c64", size = 22113, upload-time = "2025-01-14T17:02:05.085Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/7e/e8/64c37fadfc2816a7701fa8a6ed8d87327c7d54eacfbfb6edab14a2f2be75/cloudpickle-3.1.1-py3-none-any.whl", hash = "sha256:c8c5a44295039331ee9dad40ba100a9c7297b6f988e50e87ccdf3765a668350e", size = 20992, upload-time = "2025-01-14T17:02:02.417Z" },
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
name = "comm"
version = "0.2.3"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/4c/13/7d740c5849255756bc17888787313b61fd38a0a8304fc4f073dfc46122aa/comm-0.2.3.tar.gz", hash = "sha256:2dc8048c10962d55d7ad693be1e7045d891b7ce8d999c97963a5e3e99c055971", size = 6319, upload-time = "2025-07-25T14:02:04.452Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/60/97/891a0971e1e4a8c5d2b20bbe0e524dc04548d2307fee33cdeba148fd4fc7/comm-0.2.3-py3-none-any.whl", hash = "sha256:c615d91d75f7f04f095b30d1c1711babd43bdc6419c1be9886a85f2f4e489417", size = 7294, upload-time = "2025-07-25T14:02:02.896Z" },
]

[[package]]
name = "coverage"
version = "7.11.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/1c/38/ee22495420457259d2f3390309505ea98f98a5eed40901cf62196abad006/coverage-7.11.0.tar.gz", hash = "sha256:167bd504ac1ca2af7ff3b81d245dfea0292c5032ebef9d66cc08a7d28c1b8050", size = 811905, upload-time = "2025-10-15T15:15:08.542Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/12/95/c49df0aceb5507a80b9fe5172d3d39bf23f05be40c23c8d77d556df96cec/coverage-7.11.0-cp310-cp310-macosx_10_9_x86_64.whl", hash = "sha256:eb53f1e8adeeb2e78962bade0c08bfdc461853c7969706ed901821e009b35e31", size = 215800, upload-time = "2025-10-15T15:12:19.824Z" },
    { url = "https://files.pythonhosted.org/packages/dc/c6/7bb46ce01ed634fff1d7bb53a54049f539971862cc388b304ff3c51b4f66/coverage-7.11.0-cp310-cp310-macosx_11_0_arm64.whl", hash = "sha256:d9a03ec6cb9f40a5c360f138b88266fd8f58408d71e89f536b4f91d85721d075", size = 216198, upload-time = "2025-10-15T15:12:22.549Z" },
    { url = "https://files.pythonhosted.org/packages/94/b2/75d9d8fbf2900268aca5de29cd0a0fe671b0f69ef88be16767cc3c828b85/coverage-7.11.0-cp310-cp310-manylinux1_i686.manylinux_2_28_i686.manylinux_2_5_i686.whl", hash = "sha256:0d7f0616c557cbc3d1c2090334eddcbb70e1ae3a40b07222d62b3aa47f608fab", size = 242953, upload-time = "2025-10-15T15:12:24.139Z" },
    { url = "https://files.pythonhosted.org/packages/65/ac/acaa984c18f440170525a8743eb4b6c960ace2dbad80dc22056a437fc3c6/coverage-7.11.0-cp310-cp310-manylinux1_x86_64.manylinux_2_28_x86_64.manylinux_2_5_x86_64.whl", hash = "sha256:e44a86a47bbdf83b0a3ea4d7df5410d6b1a0de984fbd805fa5101f3624b9abe0", size = 244766, upload-time = "2025-10-15T15:12:25.974Z" },
    { url = "https://files.pythonhosted.org/packages/d8/0d/938d0bff76dfa4a6b228c3fc4b3e1c0e2ad4aa6200c141fcda2bd1170227/coverage-7.11.0-cp310-cp310-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:596763d2f9a0ee7eec6e643e29660def2eef297e1de0d334c78c08706f1cb785", size = 246625, upload-time = "2025-10-15T15:12:27.387Z" },
    { url = "https://files.pythonhosted.org/packages/38/54/8f5f5e84bfa268df98f46b2cb396b1009734cfb1e5d6adb663d284893b32/coverage-7.11.0-cp310-cp310-manylinux_2_31_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:ef55537ff511b5e0a43edb4c50a7bf7ba1c3eea20b4f49b1490f1e8e0e42c591", size = 243568, upload-time = "2025-10-15T15:12:28.799Z" },
    { url = "https://files.pythonhosted.org/packages/68/30/8ba337c2877fe3f2e1af0ed7ff4be0c0c4aca44d6f4007040f3ca2255e99/coverage-7.11.0-cp310-cp310-musllinux_1_2_aarch64.whl", hash = "sha256:9cbabd8f4d0d3dc571d77ae5bdbfa6afe5061e679a9d74b6797c48d143307088", size = 244665, upload-time = "2025-10-15T15:12:30.297Z" },
    { url = "https://files.pythonhosted.org/packages/cc/fb/c6f1d6d9a665536b7dde2333346f0cc41dc6a60bd1ffc10cd5c33e7eb000/coverage-7.11.0-cp310-cp310-musllinux_1_2_i686.whl", hash = "sha256:e24045453384e0ae2a587d562df2a04d852672eb63051d16096d3f08aa4c7c2f", size = 242681, upload-time = "2025-10-15T15:12:32.326Z" },
    { url = "https://files.pythonhosted.org/packages/be/38/1b532319af5f991fa153c20373291dc65c2bf532af7dbcffdeef745c8f79/coverage-7.11.0-cp310-cp310-musllinux_1_2_riscv64.whl", hash = "sha256:7161edd3426c8d19bdccde7d49e6f27f748f3c31cc350c5de7c633fea445d866", size = 242912, upload-time = "2025-10-15T15:12:34.079Z" },
    { url = "https://files.pythonhosted.org/packages/67/3d/f39331c60ef6050d2a861dc1b514fa78f85f792820b68e8c04196ad733d6/coverage-7.11.0-cp310-cp310-musllinux_1_2_x86_64.whl", hash = "sha256:3d4ed4de17e692ba6415b0587bc7f12bc80915031fc9db46a23ce70fc88c9841", size = 243559, upload-time = "2025-10-15T15:12:35.809Z" },
    { url = "https://files.pythonhosted.org/packages/4b/55/cb7c9df9d0495036ce582a8a2958d50c23cd73f84a23284bc23bd4711a6f/coverage-7.11.0-cp310-cp310-win32.whl", hash = "sha256:765c0bc8fe46f48e341ef737c91c715bd2a53a12792592296a095f0c237e09cf", size = 218266, upload-time = "2025-10-15T15:12:37.429Z" },
    { url = "https://files.pythonhosted.org/packages/68/a8/b79cb275fa7bd0208767f89d57a1b5f6ba830813875738599741b97c2e04/coverage-7.11.0-cp310-cp310-win_amd64.whl", hash = "sha256:24d6f3128f1b2d20d84b24f4074475457faedc3d4613a7e66b5e769939c7d969", size = 219169, upload-time = "2025-10-15T15:12:39.25Z" },
    { url = "https://files.pythonhosted.org/packages/49/3a/ee1074c15c408ddddddb1db7dd904f6b81bc524e01f5a1c5920e13dbde23/coverage-7.11.0-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:3d58ecaa865c5b9fa56e35efc51d1014d4c0d22838815b9fce57a27dd9576847", size = 215912, upload-time = "2025-10-15T15:12:40.665Z" },
    { url = "https://files.pythonhosted.org/packages/70/c4/9f44bebe5cb15f31608597b037d78799cc5f450044465bcd1ae8cb222fe1/coverage-7.11.0-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:b679e171f1c104a5668550ada700e3c4937110dbdd153b7ef9055c4f1a1ee3cc", size = 216310, upload-time = "2025-10-15T15:12:42.461Z" },
    { url = "https://files.pythonhosted.org/packages/42/01/5e06077cfef92d8af926bdd86b84fb28bf9bc6ad27343d68be9b501d89f2/coverage-7.11.0-cp311-cp311-manylinux1_i686.manylinux_2_28_i686.manylinux_2_5_i686.whl", hash = "sha256:ca61691ba8c5b6797deb221a0d09d7470364733ea9c69425a640f1f01b7c5bf0", size = 246706, upload-time = "2025-10-15T15:12:44.001Z" },
    { url = "https://files.pythonhosted.org/packages/40/b8/7a3f1f33b35cc4a6c37e759137533119560d06c0cc14753d1a803be0cd4a/coverage-7.11.0-cp311-cp311-manylinux1_x86_64.manylinux_2_28_x86_64.manylinux_2_5_x86_64.whl", hash = "sha256:aef1747ede4bd8ca9cfc04cc3011516500c6891f1b33a94add3253f6f876b7b7", size = 248634, upload-time = "2025-10-15T15:12:45.768Z" },
    { url = "https://files.pythonhosted.org/packages/7a/41/7f987eb33de386bc4c665ab0bf98d15fcf203369d6aacae74f5dd8ec489a/coverage-7.11.0-cp311-cp311-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:a1839d08406e4cba2953dcc0ffb312252f14d7c4c96919f70167611f4dee2623", size = 250741, upload-time = "2025-10-15T15:12:47.222Z" },
    { url = "https://files.pythonhosted.org/packages/23/c1/a4e0ca6a4e83069fb8216b49b30a7352061ca0cb38654bd2dc96b7b3b7da/coverage-7.11.0-cp311-cp311-manylinux_2_31_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:e0eb0a2dcc62478eb5b4cbb80b97bdee852d7e280b90e81f11b407d0b81c4287", size = 246837, upload-time = "2025-10-15T15:12:48.904Z" },
    { url = "https://files.pythonhosted.org/packages/5d/03/ced062a17f7c38b4728ff76c3acb40d8465634b20b4833cdb3cc3a74e115/coverage-7.11.0-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:bc1fbea96343b53f65d5351d8fd3b34fd415a2670d7c300b06d3e14a5af4f552", size = 248429, upload-time = "2025-10-15T15:12:50.73Z" },
    { url = "https://files.pythonhosted.org/packages/97/af/a7c6f194bb8c5a2705ae019036b8fe7f49ea818d638eedb15fdb7bed227c/coverage-7.11.0-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:214b622259dd0cf435f10241f1333d32caa64dbc27f8790ab693428a141723de", size = 246490, upload-time = "2025-10-15T15:12:52.646Z" },
    { url = "https://files.pythonhosted.org/packages/ab/c3/aab4df02b04a8fde79068c3c41ad7a622b0ef2b12e1ed154da986a727c3f/coverage-7.11.0-cp311-cp311-musllinux_1_2_riscv64.whl", hash = "sha256:258d9967520cca899695d4eb7ea38be03f06951d6ca2f21fb48b1235f791e601", size = 246208, upload-time = "2025-10-15T15:12:54.586Z" },
    { url = "https://files.pythonhosted.org/packages/30/d8/e282ec19cd658238d60ed404f99ef2e45eed52e81b866ab1518c0d4163cf/coverage-7.11.0-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:cf9e6ff4ca908ca15c157c409d608da77a56a09877b97c889b98fb2c32b6465e", size = 247126, upload-time = "2025-10-15T15:12:56.485Z" },
    { url = "https://files.pythonhosted.org/packages/d1/17/a635fa07fac23adb1a5451ec756216768c2767efaed2e4331710342a3399/coverage-7.11.0-cp311-cp311-win32.whl", hash = "sha256:fcc15fc462707b0680cff6242c48625da7f9a16a28a41bb8fd7a4280920e676c", size = 218314, upload-time = "2025-10-15T15:12:58.365Z" },
    { url = "https://files.pythonhosted.org/packages/2a/29/2ac1dfcdd4ab9a70026edc8d715ece9b4be9a1653075c658ee6f271f394d/coverage-7.11.0-cp311-cp311-win_amd64.whl", hash = "sha256:865965bf955d92790f1facd64fe7ff73551bd2c1e7e6b26443934e9701ba30b9", size = 219203, upload-time = "2025-10-15T15:12:59.902Z" },
    { url = "https://files.pythonhosted.org/packages/03/21/5ce8b3a0133179115af4c041abf2ee652395837cb896614beb8ce8ddcfd9/coverage-7.11.0-cp311-cp311-win_arm64.whl", hash = "sha256:5693e57a065760dcbeb292d60cc4d0231a6d4b6b6f6a3191561e1d5e8820b745", size = 217879, upload-time = "2025-10-15T15:13:01.35Z" },
    { url = "https://files.pythonhosted.org/packages/c4/db/86f6906a7c7edc1a52b2c6682d6dd9be775d73c0dfe2b84f8923dfea5784/coverage-7.11.0-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:9c49e77811cf9d024b95faf86c3f059b11c0c9be0b0d61bc598f453703bd6fd1", size = 216098, upload-time = "2025-10-15T15:13:02.916Z" },
    { url = "https://files.pythonhosted.org/packages/21/54/e7b26157048c7ba555596aad8569ff903d6cd67867d41b75287323678ede/coverage-7.11.0-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:a61e37a403a778e2cda2a6a39abcc895f1d984071942a41074b5c7ee31642007", size = 216331, upload-time = "2025-10-15T15:13:04.403Z" },
    { url = "https://files.pythonhosted.org/packages/b9/19/1ce6bf444f858b83a733171306134a0544eaddf1ca8851ede6540a55b2ad/coverage-7.11.0-cp312-cp312-manylinux1_i686.manylinux_2_28_i686.manylinux_2_5_i686.whl", hash = "sha256:c79cae102bb3b1801e2ef1511fb50e91ec83a1ce466b2c7c25010d884336de46", size = 247825, upload-time = "2025-10-15T15:13:05.92Z" },
    { url = "https://files.pythonhosted.org/packages/71/0b/d3bcbbc259fcced5fb67c5d78f6e7ee965f49760c14afd931e9e663a83b2/coverage-7.11.0-cp312-cp312-manylinux1_x86_64.manylinux_2_28_x86_64.manylinux_2_5_x86_64.whl", hash = "sha256:16ce17ceb5d211f320b62df002fa7016b7442ea0fd260c11cec8ce7730954893", size = 250573, upload-time = "2025-10-15T15:13:07.471Z" },
    { url = "https://files.pythonhosted.org/packages/58/8d/b0ff3641a320abb047258d36ed1c21d16be33beed4152628331a1baf3365/coverage-7.11.0-cp312-cp312-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:80027673e9d0bd6aef86134b0771845e2da85755cf686e7c7c59566cf5a89115", size = 251706, upload-time = "2025-10-15T15:13:09.4Z" },
    { url = "https://files.pythonhosted.org/packages/59/c8/5a586fe8c7b0458053d9c687f5cff515a74b66c85931f7fe17a1c958b4ac/coverage-7.11.0-cp312-cp312-manylinux_2_31_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:4d3ffa07a08657306cd2215b0da53761c4d73cb54d9143b9303a6481ec0cd415", size = 248221, upload-time = "2025-10-15T15:13:10.964Z" },
    { url = "https://files.pythonhosted.org/packages/d0/ff/3a25e3132804ba44cfa9a778cdf2b73dbbe63ef4b0945e39602fc896ba52/coverage-7.11.0-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:a3b6a5f8b2524fd6c1066bc85bfd97e78709bb5e37b5b94911a6506b65f47186", size = 249624, upload-time = "2025-10-15T15:13:12.5Z" },
    { url = "https://files.pythonhosted.org/packages/c5/12/ff10c8ce3895e1b17a73485ea79ebc1896a9e466a9d0f4aef63e0d17b718/coverage-7.11.0-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:fcc0a4aa589de34bc56e1a80a740ee0f8c47611bdfb28cd1849de60660f3799d", size = 247744, upload-time = "2025-10-15T15:13:14.554Z" },
    { url = "https://files.pythonhosted.org/packages/16/02/d500b91f5471b2975947e0629b8980e5e90786fe316b6d7299852c1d793d/coverage-7.11.0-cp312-cp312-musllinux_1_2_riscv64.whl", hash = "sha256:dba82204769d78c3fd31b35c3d5f46e06511936c5019c39f98320e05b08f794d", size = 247325, upload-time = "2025-10-15T15:13:16.438Z" },
    { url = "https://files.pythonhosted.org/packages/77/11/dee0284fbbd9cd64cfce806b827452c6df3f100d9e66188e82dfe771d4af/coverage-7.11.0-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:81b335f03ba67309a95210caf3eb43bd6fe75a4e22ba653ef97b4696c56c7ec2", size = 249180, upload-time = "2025-10-15T15:13:17.959Z" },
    { url = "https://files.pythonhosted.org/packages/59/1b/cdf1def928f0a150a057cab03286774e73e29c2395f0d30ce3d9e9f8e697/coverage-7.11.0-cp312-cp312-win32.whl", hash = "sha256:037b2d064c2f8cc8716fe4d39cb705779af3fbf1ba318dc96a1af858888c7bb5", size = 218479, upload-time = "2025-10-15T15:13:19.608Z" },
    { url = "https://files.pythonhosted.org/packages/ff/55/e5884d55e031da9c15b94b90a23beccc9d6beee65e9835cd6da0a79e4f3a/coverage-7.11.0-cp312-cp312-win_amd64.whl", hash = "sha256:d66c0104aec3b75e5fd897e7940188ea1892ca1d0235316bf89286d6a22568c0", size = 219290, upload-time = "2025-10-15T15:13:21.593Z" },
    { url = "https://files.pythonhosted.org/packages/23/a8/faa930cfc71c1d16bc78f9a19bb73700464f9c331d9e547bfbc1dbd3a108/coverage-7.11.0-cp312-cp312-win_arm64.whl", hash = "sha256:d91ebeac603812a09cf6a886ba6e464f3bbb367411904ae3790dfe28311b15ad", size = 217924, upload-time = "2025-10-15T15:13:23.39Z" },
    { url = "https://files.pythonhosted.org/packages/60/7f/85e4dfe65e400645464b25c036a26ac226cf3a69d4a50c3934c532491cdd/coverage-7.11.0-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:cc3f49e65ea6e0d5d9bd60368684fe52a704d46f9e7fc413918f18d046ec40e1", size = 216129, upload-time = "2025-10-15T15:13:25.371Z" },
    { url = "https://files.pythonhosted.org/packages/96/5d/dc5fa98fea3c175caf9d360649cb1aa3715e391ab00dc78c4c66fabd7356/coverage-7.11.0-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:f39ae2f63f37472c17b4990f794035c9890418b1b8cca75c01193f3c8d3e01be", size = 216380, upload-time = "2025-10-15T15:13:26.976Z" },
    { url = "https://files.pythonhosted.org/packages/b2/f5/3da9cc9596708273385189289c0e4d8197d37a386bdf17619013554b3447/coverage-7.11.0-cp313-cp313-manylinux1_i686.manylinux_2_28_i686.manylinux_2_5_i686.whl", hash = "sha256:7db53b5cdd2917b6eaadd0b1251cf4e7d96f4a8d24e174bdbdf2f65b5ea7994d", size = 247375, upload-time = "2025-10-15T15:13:28.923Z" },
    { url = "https://files.pythonhosted.org/packages/65/6c/f7f59c342359a235559d2bc76b0c73cfc4bac7d61bb0df210965cb1ecffd/coverage-7.11.0-cp313-cp313-manylinux1_x86_64.manylinux_2_28_x86_64.manylinux_2_5_x86_64.whl", hash = "sha256:10ad04ac3a122048688387828b4537bc9cf60c0bf4869c1e9989c46e45690b82", size = 249978, upload-time = "2025-10-15T15:13:30.525Z" },
    { url = "https://files.pythonhosted.org/packages/e7/8c/042dede2e23525e863bf1ccd2b92689692a148d8b5fd37c37899ba882645/coverage-7.11.0-cp313-cp313-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:4036cc9c7983a2b1f2556d574d2eb2154ac6ed55114761685657e38782b23f52", size = 251253, upload-time = "2025-10-15T15:13:32.174Z" },
    { url = "https://files.pythonhosted.org/packages/7b/a9/3c58df67bfa809a7bddd786356d9c5283e45d693edb5f3f55d0986dd905a/coverage-7.11.0-cp313-cp313-manylinux_2_31_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:7ab934dd13b1c5e94b692b1e01bd87e4488cb746e3a50f798cb9464fd128374b", size = 247591, upload-time = "2025-10-15T15:13:34.147Z" },
    { url = "https://files.pythonhosted.org/packages/26/5b/c7f32efd862ee0477a18c41e4761305de6ddd2d49cdeda0c1116227570fd/coverage-7.11.0-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:59a6e5a265f7cfc05f76e3bb53eca2e0dfe90f05e07e849930fecd6abb8f40b4", size = 249411, upload-time = "2025-10-15T15:13:38.425Z" },
    { url = "https://files.pythonhosted.org/packages/76/b5/78cb4f1e86c1611431c990423ec0768122905b03837e1b4c6a6f388a858b/coverage-7.11.0-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:df01d6c4c81e15a7c88337b795bb7595a8596e92310266b5072c7e301168efbd", size = 247303, upload-time = "2025-10-15T15:13:40.464Z" },
    { url = "https://files.pythonhosted.org/packages/87/c9/23c753a8641a330f45f221286e707c427e46d0ffd1719b080cedc984ec40/coverage-7.11.0-cp313-cp313-musllinux_1_2_riscv64.whl", hash = "sha256:8c934bd088eed6174210942761e38ee81d28c46de0132ebb1801dbe36a390dcc", size = 247157, upload-time = "2025-10-15T15:13:42.087Z" },
    { url = "https://files.pythonhosted.org/packages/c5/42/6e0cc71dc8a464486e944a4fa0d85bdec031cc2969e98ed41532a98336b9/coverage-7.11.0-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:5a03eaf7ec24078ad64a07f02e30060aaf22b91dedf31a6b24d0d98d2bba7f48", size = 248921, upload-time = "2025-10-15T15:13:43.715Z" },
    { url = "https://files.pythonhosted.org/packages/e8/1c/743c2ef665e6858cccb0f84377dfe3a4c25add51e8c7ef19249be92465b6/coverage-7.11.0-cp313-cp313-win32.whl", hash = "sha256:695340f698a5f56f795b2836abe6fb576e7c53d48cd155ad2f80fd24bc63a040", size = 218526, upload-time = "2025-10-15T15:13:45.336Z" },
    { url = "https://files.pythonhosted.org/packages/ff/d5/226daadfd1bf8ddbccefbd3aa3547d7b960fb48e1bdac124e2dd13a2b71a/coverage-7.11.0-cp313-cp313-win_amd64.whl", hash = "sha256:2727d47fce3ee2bac648528e41455d1b0c46395a087a229deac75e9f88ba5a05", size = 219317, upload-time = "2025-10-15T15:13:47.401Z" },
    { url = "https://files.pythonhosted.org/packages/97/54/47db81dcbe571a48a298f206183ba8a7ba79200a37cd0d9f4788fcd2af4a/coverage-7.11.0-cp313-cp313-win_arm64.whl", hash = "sha256:0efa742f431529699712b92ecdf22de8ff198df41e43aeaaadf69973eb93f17a", size = 217948, upload-time = "2025-10-15T15:13:49.096Z" },
    { url = "https://files.pythonhosted.org/packages/e5/8b/cb68425420154e7e2a82fd779a8cc01549b6fa83c2ad3679cd6c088ebd07/coverage-7.11.0-cp313-cp313t-macosx_10_13_x86_64.whl", hash = "sha256:587c38849b853b157706407e9ebdca8fd12f45869edb56defbef2daa5fb0812b", size = 216837, upload-time = "2025-10-15T15:13:51.09Z" },
    { url = "https://files.pythonhosted.org/packages/33/55/9d61b5765a025685e14659c8d07037247de6383c0385757544ffe4606475/coverage-7.11.0-cp313-cp313t-macosx_11_0_arm64.whl", hash = "sha256:b971bdefdd75096163dd4261c74be813c4508477e39ff7b92191dea19f24cd37", size = 217061, upload-time = "2025-10-15T15:13:52.747Z" },
    { url = "https://files.pythonhosted.org/packages/52/85/292459c9186d70dcec6538f06ea251bc968046922497377bf4a1dc9a71de/coverage-7.11.0-cp313-cp313t-manylinux1_i686.manylinux_2_28_i686.manylinux_2_5_i686.whl", hash = "sha256:269bfe913b7d5be12ab13a95f3a76da23cf147be7fa043933320ba5625f0a8de", size = 258398, upload-time = "2025-10-15T15:13:54.45Z" },
    { url = "https://files.pythonhosted.org/packages/1f/e2/46edd73fb8bf51446c41148d81944c54ed224854812b6ca549be25113ee0/coverage-7.11.0-cp313-cp313t-manylinux1_x86_64.manylinux_2_28_x86_64.manylinux_2_5_x86_64.whl", hash = "sha256:dadbcce51a10c07b7c72b0ce4a25e4b6dcb0c0372846afb8e5b6307a121eb99f", size = 260574, upload-time = "2025-10-15T15:13:56.145Z" },
    { url = "https://files.pythonhosted.org/packages/07/5e/1df469a19007ff82e2ca8fe509822820a31e251f80ee7344c34f6cd2ec43/coverage-7.11.0-cp313-cp313t-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:9ed43fa22c6436f7957df036331f8fe4efa7af132054e1844918866cd228af6c", size = 262797, upload-time = "2025-10-15T15:13:58.635Z" },
    { url = "https://files.pythonhosted.org/packages/f9/50/de216b31a1434b94d9b34a964c09943c6be45069ec704bfc379d8d89a649/coverage-7.11.0-cp313-cp313t-manylinux_2_31_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:9516add7256b6713ec08359b7b05aeff8850c98d357784c7205b2e60aa2513fa", size = 257361, upload-time = "2025-10-15T15:14:00.409Z" },
    { url = "https://files.pythonhosted.org/packages/82/1e/3f9f8344a48111e152e0fd495b6fff13cc743e771a6050abf1627a7ba918/coverage-7.11.0-cp313-cp313t-musllinux_1_2_aarch64.whl", hash = "sha256:eb92e47c92fcbcdc692f428da67db33337fa213756f7adb6a011f7b5a7a20740", size = 260349, upload-time = "2025-10-15T15:14:02.188Z" },
    { url = "https://files.pythonhosted.org/packages/65/9b/3f52741f9e7d82124272f3070bbe316006a7de1bad1093f88d59bfc6c548/coverage-7.11.0-cp313-cp313t-musllinux_1_2_i686.whl", hash = "sha256:d06f4fc7acf3cabd6d74941d53329e06bab00a8fe10e4df2714f0b134bfc64ef", size = 258114, upload-time = "2025-10-15T15:14:03.907Z" },
    { url = "https://files.pythonhosted.org/packages/0b/8b/918f0e15f0365d50d3986bbd3338ca01178717ac5678301f3f547b6619e6/coverage-7.11.0-cp313-cp313t-musllinux_1_2_riscv64.whl", hash = "sha256:6fbcee1a8f056af07ecd344482f711f563a9eb1c2cad192e87df00338ec3cdb0", size = 256723, upload-time = "2025-10-15T15:14:06.324Z" },
    { url = "https://files.pythonhosted.org/packages/44/9e/7776829f82d3cf630878a7965a7d70cc6ca94f22c7d20ec4944f7148cb46/coverage-7.11.0-cp313-cp313t-musllinux_1_2_x86_64.whl", hash = "sha256:dbbf012be5f32533a490709ad597ad8a8ff80c582a95adc8d62af664e532f9ca", size = 259238, upload-time = "2025-10-15T15:14:08.002Z" },
    { url = "https://files.pythonhosted.org/packages/9a/b8/49cf253e1e7a3bedb85199b201862dd7ca4859f75b6cf25ffa7298aa0760/coverage-7.11.0-cp313-cp313t-win32.whl", hash = "sha256:cee6291bb4fed184f1c2b663606a115c743df98a537c969c3c64b49989da96c2", size = 219180, upload-time = "2025-10-15T15:14:09.786Z" },
    { url = "https://files.pythonhosted.org/packages/ac/e1/1a541703826be7ae2125a0fb7f821af5729d56bb71e946e7b933cc7a89a4/coverage-7.11.0-cp313-cp313t-win_amd64.whl", hash = "sha256:a386c1061bf98e7ea4758e4313c0ab5ecf57af341ef0f43a0bf26c2477b5c268", size = 220241, upload-time = "2025-10-15T15:14:11.471Z" },
    { url = "https://files.pythonhosted.org/packages/d5/d1/5ee0e0a08621140fd418ec4020f595b4d52d7eb429ae6a0c6542b4ba6f14/coverage-7.11.0-cp313-cp313t-win_arm64.whl", hash = "sha256:f9ea02ef40bb83823b2b04964459d281688fe173e20643870bb5d2edf68bc836", size = 218510, upload-time = "2025-10-15T15:14:13.46Z" },
    { url = "https://files.pythonhosted.org/packages/f4/06/e923830c1985ce808e40a3fa3eb46c13350b3224b7da59757d37b6ce12b8/coverage-7.11.0-cp314-cp314-macosx_10_13_x86_64.whl", hash = "sha256:c770885b28fb399aaf2a65bbd1c12bf6f307ffd112d6a76c5231a94276f0c497", size = 216110, upload-time = "2025-10-15T15:14:15.157Z" },
    { url = "https://files.pythonhosted.org/packages/42/82/cdeed03bfead45203fb651ed756dfb5266028f5f939e7f06efac4041dad5/coverage-7.11.0-cp314-cp314-macosx_11_0_arm64.whl", hash = "sha256:a3d0e2087dba64c86a6b254f43e12d264b636a39e88c5cc0a01a7c71bcfdab7e", size = 216395, upload-time = "2025-10-15T15:14:16.863Z" },
    { url = "https://files.pythonhosted.org/packages/fc/ba/e1c80caffc3199aa699813f73ff097bc2df7b31642bdbc7493600a8f1de5/coverage-7.11.0-cp314-cp314-manylinux1_i686.manylinux_2_28_i686.manylinux_2_5_i686.whl", hash = "sha256:73feb83bb41c32811973b8565f3705caf01d928d972b72042b44e97c71fd70d1", size = 247433, upload-time = "2025-10-15T15:14:18.589Z" },
    { url = "https://files.pythonhosted.org/packages/80/c0/5b259b029694ce0a5bbc1548834c7ba3db41d3efd3474489d7efce4ceb18/coverage-7.11.0-cp314-cp314-manylinux1_x86_64.manylinux_2_28_x86_64.manylinux_2_5_x86_64.whl", hash = "sha256:c6f31f281012235ad08f9a560976cc2fc9c95c17604ff3ab20120fe480169bca", size = 249970, upload-time = "2025-10-15T15:14:20.307Z" },
    { url = "https://files.pythonhosted.org/packages/8c/86/171b2b5e1aac7e2fd9b43f7158b987dbeb95f06d1fbecad54ad8163ae3e8/coverage-7.11.0-cp314-cp314-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:e9570ad567f880ef675673992222746a124b9595506826b210fbe0ce3f0499cd", size = 251324, upload-time = "2025-10-15T15:14:22.419Z" },
    { url = "https://files.pythonhosted.org/packages/1a/7e/7e10414d343385b92024af3932a27a1caf75c6e27ee88ba211221ff1a145/coverage-7.11.0-cp314-cp314-manylinux_2_31_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:8badf70446042553a773547a61fecaa734b55dc738cacf20c56ab04b77425e43", size = 247445, upload-time = "2025-10-15T15:14:24.205Z" },
    { url = "https://files.pythonhosted.org/packages/c4/3b/e4f966b21f5be8c4bf86ad75ae94efa0de4c99c7bbb8114476323102e345/coverage-7.11.0-cp314-cp314-musllinux_1_2_aarch64.whl", hash = "sha256:a09c1211959903a479e389685b7feb8a17f59ec5a4ef9afde7650bd5eabc2777", size = 249324, upload-time = "2025-10-15T15:14:26.234Z" },
    { url = "https://files.pythonhosted.org/packages/00/a2/8479325576dfcd909244d0df215f077f47437ab852ab778cfa2f8bf4d954/coverage-7.11.0-cp314-cp314-musllinux_1_2_i686.whl", hash = "sha256:5ef83b107f50db3f9ae40f69e34b3bd9337456c5a7fe3461c7abf8b75dd666a2", size = 247261, upload-time = "2025-10-15T15:14:28.42Z" },
    { url = "https://files.pythonhosted.org/packages/7b/d8/3a9e2db19d94d65771d0f2e21a9ea587d11b831332a73622f901157cc24b/coverage-7.11.0-cp314-cp314-musllinux_1_2_riscv64.whl", hash = "sha256:f91f927a3215b8907e214af77200250bb6aae36eca3f760f89780d13e495388d", size = 247092, upload-time = "2025-10-15T15:14:30.784Z" },
    { url = "https://files.pythonhosted.org/packages/b3/b1/bbca3c472544f9e2ad2d5116b2379732957048be4b93a9c543fcd0207e5f/coverage-7.11.0-cp314-cp314-musllinux_1_2_x86_64.whl", hash = "sha256:cdbcd376716d6b7fbfeedd687a6c4be019c5a5671b35f804ba76a4c0a778cba4", size = 248755, upload-time = "2025-10-15T15:14:32.585Z" },
    { url = "https://files.pythonhosted.org/packages/89/49/638d5a45a6a0f00af53d6b637c87007eb2297042186334e9923a61aa8854/coverage-7.11.0-cp314-cp314-win32.whl", hash = "sha256:bab7ec4bb501743edc63609320aaec8cd9188b396354f482f4de4d40a9d10721", size = 218793, upload-time = "2025-10-15T15:14:34.972Z" },
    { url = "https://files.pythonhosted.org/packages/30/cc/b675a51f2d068adb3cdf3799212c662239b0ca27f4691d1fff81b92ea850/coverage-7.11.0-cp314-cp314-win_amd64.whl", hash = "sha256:3d4ba9a449e9364a936a27322b20d32d8b166553bfe63059bd21527e681e2fad", size = 219587, upload-time = "2025-10-15T15:14:37.047Z" },
    { url = "https://files.pythonhosted.org/packages/93/98/5ac886876026de04f00820e5094fe22166b98dcb8b426bf6827aaf67048c/coverage-7.11.0-cp314-cp314-win_arm64.whl", hash = "sha256:ce37f215223af94ef0f75ac68ea096f9f8e8c8ec7d6e8c346ee45c0d363f0479", size = 218168, upload-time = "2025-10-15T15:14:38.861Z" },
    { url = "https://files.pythonhosted.org/packages/14/d1/b4145d35b3e3ecf4d917e97fc8895bcf027d854879ba401d9ff0f533f997/coverage-7.11.0-cp314-cp314t-macosx_10_13_x86_64.whl", hash = "sha256:f413ce6e07e0d0dc9c433228727b619871532674b45165abafe201f200cc215f", size = 216850, upload-time = "2025-10-15T15:14:40.651Z" },
    { url = "https://files.pythonhosted.org/packages/ca/d1/7f645fc2eccd318369a8a9948acc447bb7c1ade2911e31d3c5620544c22b/coverage-7.11.0-cp314-cp314t-macosx_11_0_arm64.whl", hash = "sha256:05791e528a18f7072bf5998ba772fe29db4da1234c45c2087866b5ba4dea710e", size = 217071, upload-time = "2025-10-15T15:14:42.755Z" },
    { url = "https://files.pythonhosted.org/packages/54/7d/64d124649db2737ceced1dfcbdcb79898d5868d311730f622f8ecae84250/coverage-7.11.0-cp314-cp314t-manylinux1_i686.manylinux_2_28_i686.manylinux_2_5_i686.whl", hash = "sha256:cacb29f420cfeb9283b803263c3b9a068924474ff19ca126ba9103e1278dfa44", size = 258570, upload-time = "2025-10-15T15:14:44.542Z" },
    { url = "https://files.pythonhosted.org/packages/6c/3f/6f5922f80dc6f2d8b2c6f974835c43f53eb4257a7797727e6ca5b7b2ec1f/coverage-7.11.0-cp314-cp314t-manylinux1_x86_64.manylinux_2_28_x86_64.manylinux_2_5_x86_64.whl", hash = "sha256:314c24e700d7027ae3ab0d95fbf8d53544fca1f20345fd30cd219b737c6e58d3", size = 260738, upload-time = "2025-10-15T15:14:46.436Z" },
    { url = "https://files.pythonhosted.org/packages/0e/5f/9e883523c4647c860b3812b417a2017e361eca5b635ee658387dc11b13c1/coverage-7.11.0-cp314-cp314t-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:630d0bd7a293ad2fc8b4b94e5758c8b2536fdf36c05f1681270203e463cbfa9b", size = 262994, upload-time = "2025-10-15T15:14:48.3Z" },
    { url = "https://files.pythonhosted.org/packages/07/bb/43b5a8e94c09c8bf51743ffc65c4c841a4ca5d3ed191d0a6919c379a1b83/coverage-7.11.0-cp314-cp314t-manylinux_2_31_riscv64.manylinux_2_39_riscv64.whl", hash = "sha256:e89641f5175d65e2dbb44db15fe4ea48fade5d5bbb9868fdc2b4fce22f4a469d", size = 257282, upload-time = "2025-10-15T15:14:50.236Z" },
    { url = "https://files.pythonhosted.org/packages/aa/e5/0ead8af411411330b928733e1d201384b39251a5f043c1612970310e8283/coverage-7.11.0-cp314-cp314t-musllinux_1_2_aarch64.whl", hash = "sha256:c9f08ea03114a637dab06cedb2e914da9dc67fa52c6015c018ff43fdde25b9c2", size = 260430, upload-time = "2025-10-15T15:14:52.413Z" },
    { url = "https://files.pythonhosted.org/packages/ae/66/03dd8bb0ba5b971620dcaac145461950f6d8204953e535d2b20c6b65d729/coverage-7.11.0-cp314-cp314t-musllinux_1_2_i686.whl", hash = "sha256:ce9f3bde4e9b031eaf1eb61df95c1401427029ea1bfddb8621c1161dcb0fa02e", size = 258190, upload-time = "2025-10-15T15:14:54.268Z" },
    { url = "https://files.pythonhosted.org/packages/45/ae/28a9cce40bf3174426cb2f7e71ee172d98e7f6446dff936a7ccecee34b14/coverage-7.11.0-cp314-cp314t-musllinux_1_2_riscv64.whl", hash = "sha256:e4dc07e95495923d6fd4d6c27bf70769425b71c89053083843fd78f378558996", size = 256658, upload-time = "2025-10-15T15:14:56.436Z" },
    { url = "https://files.pythonhosted.org/packages/5c/7c/3a44234a8599513684bfc8684878fd7b126c2760f79712bb78c56f19efc4/coverage-7.11.0-cp314-cp314t-musllinux_1_2_x86_64.whl", hash = "sha256:424538266794db2861db4922b05d729ade0940ee69dcf0591ce8f69784db0e11", size = 259342, upload-time = "2025-10-15T15:14:58.538Z" },
    { url = "https://files.pythonhosted.org/packages/e1/e6/0108519cba871af0351725ebdb8660fd7a0fe2ba3850d56d32490c7d9b4b/coverage-7.11.0-cp314-cp314t-win32.whl", hash = "sha256:4c1eeb3fb8eb9e0190bebafd0462936f75717687117339f708f395fe455acc73", size = 219568, upload-time = "2025-10-15T15:15:00.382Z" },
    { url = "https://files.pythonhosted.org/packages/c9/76/44ba876e0942b4e62fdde23ccb029ddb16d19ba1bef081edd00857ba0b16/coverage-7.11.0-cp314-cp314t-win_amd64.whl", hash = "sha256:b56efee146c98dbf2cf5cffc61b9829d1e94442df4d7398b26892a53992d3547", size = 220687, upload-time = "2025-10-15T15:15:02.322Z" },
    { url = "https://files.pythonhosted.org/packages/b9/0c/0df55ecb20d0d0ed5c322e10a441775e1a3a5d78c60f0c4e1abfe6fcf949/coverage-7.11.0-cp314-cp314t-win_arm64.whl", hash = "sha256:b5c2705afa83f49bd91962a4094b6b082f94aef7626365ab3f8f4bd159c5acf3", size = 218711, upload-time = "2025-10-15T15:15:04.575Z" },
    { url = "https://files.pythonhosted.org/packages/5f/04/642c1d8a448ae5ea1369eac8495740a79eb4e581a9fb0cbdce56bbf56da1/coverage-7.11.0-py3-none-any.whl", hash = "sha256:4b7589765348d78fb4e5fb6ea35d07564e387da2fc5efff62e0222971f155f68", size = 207761, upload-time = "2025-10-15T15:15:06.439Z" },
]

[package.optional-dependencies]
toml = [
    { name = "tomli", marker = "python_full_version <= '3.11'" },
]

[[package]]
name = "cryptography"
version = "44.0.3"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "cffi", marker = "python_full_version >= '3.11' and python_full_version < '3.14' and platform_python_implementation != 'PyPy'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/53/d6/1411ab4d6108ab167d06254c5be517681f1e331f90edf1379895bcb87020/cryptography-44.0.3.tar.gz", hash = "sha256:fe19d8bc5536a91a24a8133328880a41831b6c5df54599a8417b62fe015d3053", size = 711096, upload-time = "2025-05-02T19:36:04.667Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/08/53/c776d80e9d26441bb3868457909b4e74dd9ccabd182e10b2b0ae7a07e265/cryptography-44.0.3-cp37-abi3-macosx_10_9_universal2.whl", hash = "sha256:962bc30480a08d133e631e8dfd4783ab71cc9e33d5d7c1e192f0b7c06397bb88", size = 6670281, upload-time = "2025-05-02T19:34:50.665Z" },
    { url = "https://files.pythonhosted.org/packages/6a/06/af2cf8d56ef87c77319e9086601bef621bedf40f6f59069e1b6d1ec498c5/cryptography-44.0.3-cp37-abi3-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:4ffc61e8f3bf5b60346d89cd3d37231019c17a081208dfbbd6e1605ba03fa137", size = 3959305, upload-time = "2025-05-02T19:34:53.042Z" },
    { url = "https://files.pythonhosted.org/packages/ae/01/80de3bec64627207d030f47bf3536889efee8913cd363e78ca9a09b13c8e/cryptography-44.0.3-cp37-abi3-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:58968d331425a6f9eedcee087f77fd3c927c88f55368f43ff7e0a19891f2642c", size = 4171040, upload-time = "2025-05-02T19:34:54.675Z" },
    { url = "https://files.pythonhosted.org/packages/bd/48/bb16b7541d207a19d9ae8b541c70037a05e473ddc72ccb1386524d4f023c/cryptography-44.0.3-cp37-abi3-manylinux_2_28_aarch64.whl", hash = "sha256:e28d62e59a4dbd1d22e747f57d4f00c459af22181f0b2f787ea83f5a876d7c76", size = 3963411, upload-time = "2025-05-02T19:34:56.61Z" },
    { url = "https://files.pythonhosted.org/packages/42/b2/7d31f2af5591d217d71d37d044ef5412945a8a8e98d5a2a8ae4fd9cd4489/cryptography-44.0.3-cp37-abi3-manylinux_2_28_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:af653022a0c25ef2e3ffb2c673a50e5a0d02fecc41608f4954176f1933b12359", size = 3689263, upload-time = "2025-05-02T19:34:58.591Z" },
    { url = "https://files.pythonhosted.org/packages/25/50/c0dfb9d87ae88ccc01aad8eb93e23cfbcea6a6a106a9b63a7b14c1f93c75/cryptography-44.0.3-cp37-abi3-manylinux_2_28_x86_64.whl", hash = "sha256:157f1f3b8d941c2bd8f3ffee0af9b049c9665c39d3da9db2dc338feca5e98a43", size = 4196198, upload-time = "2025-05-02T19:35:00.988Z" },
    { url = "https://files.pythonhosted.org/packages/66/c9/55c6b8794a74da652690c898cb43906310a3e4e4f6ee0b5f8b3b3e70c441/cryptography-44.0.3-cp37-abi3-manylinux_2_34_aarch64.whl", hash = "sha256:c6cd67722619e4d55fdb42ead64ed8843d64638e9c07f4011163e46bc512cf01", size = 3966502, upload-time = "2025-05-02T19:35:03.091Z" },
    { url = "https://files.pythonhosted.org/packages/b6/f7/7cb5488c682ca59a02a32ec5f975074084db4c983f849d47b7b67cc8697a/cryptography-44.0.3-cp37-abi3-manylinux_2_34_x86_64.whl", hash = "sha256:b424563394c369a804ecbee9b06dfb34997f19d00b3518e39f83a5642618397d", size = 4196173, upload-time = "2025-05-02T19:35:05.018Z" },
    { url = "https://files.pythonhosted.org/packages/d2/0b/2f789a8403ae089b0b121f8f54f4a3e5228df756e2146efdf4a09a3d5083/cryptography-44.0.3-cp37-abi3-musllinux_1_2_aarch64.whl", hash = "sha256:c91fc8e8fd78af553f98bc7f2a1d8db977334e4eea302a4bfd75b9461c2d8904", size = 4087713, upload-time = "2025-05-02T19:35:07.187Z" },
    { url = "https://files.pythonhosted.org/packages/1d/aa/330c13655f1af398fc154089295cf259252f0ba5df93b4bc9d9c7d7f843e/cryptography-44.0.3-cp37-abi3-musllinux_1_2_x86_64.whl", hash = "sha256:25cd194c39fa5a0aa4169125ee27d1172097857b27109a45fadc59653ec06f44", size = 4299064, upload-time = "2025-05-02T19:35:08.879Z" },
    { url = "https://files.pythonhosted.org/packages/10/a8/8c540a421b44fd267a7d58a1fd5f072a552d72204a3f08194f98889de76d/cryptography-44.0.3-cp37-abi3-win32.whl", hash = "sha256:3be3f649d91cb182c3a6bd336de8b61a0a71965bd13d1a04a0e15b39c3d5809d", size = 2773887, upload-time = "2025-05-02T19:35:10.41Z" },
    { url = "https://files.pythonhosted.org/packages/b9/0d/c4b1657c39ead18d76bbd122da86bd95bdc4095413460d09544000a17d56/cryptography-44.0.3-cp37-abi3-win_amd64.whl", hash = "sha256:3883076d5c4cc56dbef0b898a74eb6992fdac29a7b9013870b34efe4ddb39a0d", size = 3209737, upload-time = "2025-05-02T19:35:12.12Z" },
    { url = "https://files.pythonhosted.org/packages/34/a3/ad08e0bcc34ad436013458d7528e83ac29910943cea42ad7dd4141a27bbb/cryptography-44.0.3-cp39-abi3-macosx_10_9_universal2.whl", hash = "sha256:5639c2b16764c6f76eedf722dbad9a0914960d3489c0cc38694ddf9464f1bb2f", size = 6673501, upload-time = "2025-05-02T19:35:13.775Z" },
    { url = "https://files.pythonhosted.org/packages/b1/f0/7491d44bba8d28b464a5bc8cc709f25a51e3eac54c0a4444cf2473a57c37/cryptography-44.0.3-cp39-abi3-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:f3ffef566ac88f75967d7abd852ed5f182da252d23fac11b4766da3957766759", size = 3960307, upload-time = "2025-05-02T19:35:15.917Z" },
    { url = "https://files.pythonhosted.org/packages/f7/c8/e5c5d0e1364d3346a5747cdcd7ecbb23ca87e6dea4f942a44e88be349f06/cryptography-44.0.3-cp39-abi3-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:192ed30fac1728f7587c6f4613c29c584abdc565d7417c13904708db10206645", size = 4170876, upload-time = "2025-05-02T19:35:18.138Z" },
    { url = "https://files.pythonhosted.org/packages/73/96/025cb26fc351d8c7d3a1c44e20cf9a01e9f7cf740353c9c7a17072e4b264/cryptography-44.0.3-cp39-abi3-manylinux_2_28_aarch64.whl", hash = "sha256:7d5fe7195c27c32a64955740b949070f21cba664604291c298518d2e255931d2", size = 3964127, upload-time = "2025-05-02T19:35:19.864Z" },
    { url = "https://files.pythonhosted.org/packages/01/44/eb6522db7d9f84e8833ba3bf63313f8e257729cf3a8917379473fcfd6601/cryptography-44.0.3-cp39-abi3-manylinux_2_28_armv7l.manylinux_2_31_armv7l.whl", hash = "sha256:3f07943aa4d7dad689e3bb1638ddc4944cc5e0921e3c227486daae0e31a05e54", size = 3689164, upload-time = "2025-05-02T19:35:21.449Z" },
    { url = "https://files.pythonhosted.org/packages/68/fb/d61a4defd0d6cee20b1b8a1ea8f5e25007e26aeb413ca53835f0cae2bcd1/cryptography-44.0.3-cp39-abi3-manylinux_2_28_x86_64.whl", hash = "sha256:cb90f60e03d563ca2445099edf605c16ed1d5b15182d21831f58460c48bffb93", size = 4198081, upload-time = "2025-05-02T19:35:23.187Z" },
    { url = "https://files.pythonhosted.org/packages/1b/50/457f6911d36432a8811c3ab8bd5a6090e8d18ce655c22820994913dd06ea/cryptography-44.0.3-cp39-abi3-manylinux_2_34_aarch64.whl", hash = "sha256:ab0b005721cc0039e885ac3503825661bd9810b15d4f374e473f8c89b7d5460c", size = 3967716, upload-time = "2025-05-02T19:35:25.426Z" },
    { url = "https://files.pythonhosted.org/packages/35/6e/dca39d553075980ccb631955c47b93d87d27f3596da8d48b1ae81463d915/cryptography-44.0.3-cp39-abi3-manylinux_2_34_x86_64.whl", hash = "sha256:3bb0847e6363c037df8f6ede57d88eaf3410ca2267fb12275370a76f85786a6f", size = 4197398, upload-time = "2025-05-02T19:35:27.678Z" },
    { url = "https://files.pythonhosted.org/packages/9b/9d/d1f2fe681eabc682067c66a74addd46c887ebacf39038ba01f8860338d3d/cryptography-44.0.3-cp39-abi3-musllinux_1_2_aarch64.whl", hash = "sha256:b0cc66c74c797e1db750aaa842ad5b8b78e14805a9b5d1348dc603612d3e3ff5", size = 4087900, upload-time = "2025-05-02T19:35:29.312Z" },
    { url = "https://files.pythonhosted.org/packages/c4/f5/3599e48c5464580b73b236aafb20973b953cd2e7b44c7c2533de1d888446/cryptography-44.0.3-cp39-abi3-musllinux_1_2_x86_64.whl", hash = "sha256:6866df152b581f9429020320e5eb9794c8780e90f7ccb021940d7f50ee00ae0b", size = 4301067, upload-time = "2025-05-02T19:35:31.547Z" },
    { url = "https://files.pythonhosted.org/packages/a7/6c/d2c48c8137eb39d0c193274db5c04a75dab20d2f7c3f81a7dcc3a8897701/cryptography-44.0.3-cp39-abi3-win32.whl", hash = "sha256:c138abae3a12a94c75c10499f1cbae81294a6f983b3af066390adee73f433028", size = 2775467, upload-time = "2025-05-02T19:35:33.805Z" },
    { url = "https://files.pythonhosted.org/packages/c9/ad/51f212198681ea7b0deaaf8846ee10af99fba4e894f67b353524eab2bbe5/cryptography-44.0.3-cp39-abi3-win_amd64.whl", hash = "sha256:5d186f32e52e66994dce4f766884bcb9c68b8da62d61d9d215bfe5fb56d21334", size = 3210375, upload-time = "2025-05-02T19:35:35.369Z" },
    { url = "https://files.pythonhosted.org/packages/7f/10/abcf7418536df1eaba70e2cfc5c8a0ab07aa7aa02a5cbc6a78b9d8b4f121/cryptography-44.0.3-pp310-pypy310_pp73-macosx_10_9_x86_64.whl", hash = "sha256:cad399780053fb383dc067475135e41c9fe7d901a97dd5d9c5dfb5611afc0d7d", size = 3393192, upload-time = "2025-05-02T19:35:37.468Z" },
    { url = "https://files.pythonhosted.org/packages/06/59/ecb3ef380f5891978f92a7f9120e2852b1df6f0a849c277b8ea45b865db2/cryptography-44.0.3-pp310-pypy310_pp73-manylinux_2_28_aarch64.whl", hash = "sha256:21a83f6f35b9cc656d71b5de8d519f566df01e660ac2578805ab245ffd8523f8", size = 3898419, upload-time = "2025-05-02T19:35:39.065Z" },
    { url = "https://files.pythonhosted.org/packages/bb/d0/35e2313dbb38cf793aa242182ad5bc5ef5c8fd4e5dbdc380b936c7d51169/cryptography-44.0.3-pp310-pypy310_pp73-manylinux_2_28_x86_64.whl", hash = "sha256:fc3c9babc1e1faefd62704bb46a69f359a9819eb0292e40df3fb6e3574715cd4", size = 4117892, upload-time = "2025-05-02T19:35:40.839Z" },
    { url = "https://files.pythonhosted.org/packages/dc/c8/31fb6e33b56c2c2100d76de3fd820afaa9d4d0b6aea1ccaf9aaf35dc7ce3/cryptography-44.0.3-pp310-pypy310_pp73-manylinux_2_34_aarch64.whl", hash = "sha256:e909df4053064a97f1e6565153ff8bb389af12c5c8d29c343308760890560aff", size = 3900855, upload-time = "2025-05-02T19:35:42.599Z" },
    { url = "https://files.pythonhosted.org/packages/43/2a/08cc2ec19e77f2a3cfa2337b429676406d4bb78ddd130a05c458e7b91d73/cryptography-44.0.3-pp310-pypy310_pp73-manylinux_2_34_x86_64.whl", hash = "sha256:dad80b45c22e05b259e33ddd458e9e2ba099c86ccf4e88db7bbab4b747b18d06", size = 4117619, upload-time = "2025-05-02T19:35:44.774Z" },
    { url = "https://files.pythonhosted.org/packages/02/68/fc3d3f84022a75f2ac4b1a1c0e5d6a0c2ea259e14cd4aae3e0e68e56483c/cryptography-44.0.3-pp310-pypy310_pp73-win_amd64.whl", hash = "sha256:479d92908277bed6e1a1c69b277734a7771c2b78633c224445b5c60a9f4bc1d9", size = 3136570, upload-time = "2025-05-02T19:35:46.94Z" },
    { url = "https://files.pythonhosted.org/packages/8d/4b/c11ad0b6c061902de5223892d680e89c06c7c4d606305eb8de56c5427ae6/cryptography-44.0.3-pp311-pypy311_pp73-macosx_10_9_x86_64.whl", hash = "sha256:896530bc9107b226f265effa7ef3f21270f18a2026bc09fed1ebd7b66ddf6375", size = 3390230, upload-time = "2025-05-02T19:35:49.062Z" },
    { url = "https://files.pythonhosted.org/packages/58/11/0a6bf45d53b9b2290ea3cec30e78b78e6ca29dc101e2e296872a0ffe1335/cryptography-44.0.3-pp311-pypy311_pp73-manylinux_2_28_aarch64.whl", hash = "sha256:9b4d4a5dbee05a2c390bf212e78b99434efec37b17a4bff42f50285c5c8c9647", size = 3895216, upload-time = "2025-05-02T19:35:51.351Z" },
    { url = "https://files.pythonhosted.org/packages/0a/27/b28cdeb7270e957f0077a2c2bfad1b38f72f1f6d699679f97b816ca33642/cryptography-44.0.3-pp311-pypy311_pp73-manylinux_2_28_x86_64.whl", hash = "sha256:02f55fb4f8b79c1221b0961488eaae21015b69b210e18c386b69de182ebb1259", size = 4115044, upload-time = "2025-05-02T19:35:53.044Z" },
    { url = "https://files.pythonhosted.org/packages/35/b0/ec4082d3793f03cb248881fecefc26015813199b88f33e3e990a43f79835/cryptography-44.0.3-pp311-pypy311_pp73-manylinux_2_34_aarch64.whl", hash = "sha256:dd3db61b8fe5be220eee484a17233287d0be6932d056cf5738225b9c05ef4fff", size = 3898034, upload-time = "2025-05-02T19:35:54.72Z" },
    { url = "https://files.pythonhosted.org/packages/0b/7f/adf62e0b8e8d04d50c9a91282a57628c00c54d4ae75e2b02a223bd1f2613/cryptography-44.0.3-pp311-pypy311_pp73-manylinux_2_34_x86_64.whl", hash = "sha256:978631ec51a6bbc0b7e58f23b68a8ce9e5f09721940933e9c217068388789fe5", size = 4114449, upload-time = "2025-05-02T19:35:57.139Z" },
    { url = "https://files.pythonhosted.org/packages/87/62/d69eb4a8ee231f4bf733a92caf9da13f1c81a44e874b1d4080c25ecbb723/cryptography-44.0.3-pp311-pypy311_pp73-win_amd64.whl", hash = "sha256:5d20cc348cca3a8aa7312f42ab953a56e15323800ca3ab0706b8cd452a3a056c", size = 3134369, upload-time = "2025-05-02T19:35:58.907Z" },
]

[[package]]
name = "debugpy"
version = "1.8.17"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/15/ad/71e708ff4ca377c4230530d6a7aa7992592648c122a2cd2b321cf8b35a76/debugpy-1.8.17.tar.gz", hash = "sha256:fd723b47a8c08892b1a16b2c6239a8b96637c62a59b94bb5dab4bac592a58a8e", size = 1644129, upload-time = "2025-09-17T16:33:20.633Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/38/36/b57c6e818d909f6e59c0182252921cf435e0951126a97e11de37e72ab5e1/debugpy-1.8.17-cp310-cp310-macosx_15_0_x86_64.whl", hash = "sha256:c41d2ce8bbaddcc0009cc73f65318eedfa3dbc88a8298081deb05389f1ab5542", size = 2098021, upload-time = "2025-09-17T16:33:22.556Z" },
    { url = "https://files.pythonhosted.org/packages/be/01/0363c7efdd1e9febd090bb13cee4fb1057215b157b2979a4ca5ccb678217/debugpy-1.8.17-cp310-cp310-manylinux_2_34_x86_64.whl", hash = "sha256:1440fd514e1b815edd5861ca394786f90eb24960eb26d6f7200994333b1d79e3", size = 3087399, upload-time = "2025-09-17T16:33:24.292Z" },
    { url = "https://files.pythonhosted.org/packages/79/bc/4a984729674aa9a84856650438b9665f9a1d5a748804ac6f37932ce0d4aa/debugpy-1.8.17-cp310-cp310-win32.whl", hash = "sha256:3a32c0af575749083d7492dc79f6ab69f21b2d2ad4cd977a958a07d5865316e4", size = 5230292, upload-time = "2025-09-17T16:33:26.137Z" },
    { url = "https://files.pythonhosted.org/packages/5d/19/2b9b3092d0cf81a5aa10c86271999453030af354d1a5a7d6e34c574515d7/debugpy-1.8.17-cp310-cp310-win_amd64.whl", hash = "sha256:a3aad0537cf4d9c1996434be68c6c9a6d233ac6f76c2a482c7803295b4e4f99a", size = 5261885, upload-time = "2025-09-17T16:33:27.592Z" },
    { url = "https://files.pythonhosted.org/packages/d8/53/3af72b5c159278c4a0cf4cffa518675a0e73bdb7d1cac0239b815502d2ce/debugpy-1.8.17-cp311-cp311-macosx_15_0_universal2.whl", hash = "sha256:d3fce3f0e3de262a3b67e69916d001f3e767661c6e1ee42553009d445d1cd840", size = 2207154, upload-time = "2025-09-17T16:33:29.457Z" },
    { url = "https://files.pythonhosted.org/packages/8f/6d/204f407df45600e2245b4a39860ed4ba32552330a0b3f5f160ae4cc30072/debugpy-1.8.17-cp311-cp311-manylinux_2_34_x86_64.whl", hash = "sha256:c6bdf134457ae0cac6fb68205776be635d31174eeac9541e1d0c062165c6461f", size = 3170322, upload-time = "2025-09-17T16:33:30.837Z" },
    { url = "https://files.pythonhosted.org/packages/f2/13/1b8f87d39cf83c6b713de2620c31205299e6065622e7dd37aff4808dd410/debugpy-1.8.17-cp311-cp311-win32.whl", hash = "sha256:e79a195f9e059edfe5d8bf6f3749b2599452d3e9380484cd261f6b7cd2c7c4da", size = 5155078, upload-time = "2025-09-17T16:33:33.331Z" },
    { url = "https://files.pythonhosted.org/packages/c2/c5/c012c60a2922cc91caa9675d0ddfbb14ba59e1e36228355f41cab6483469/debugpy-1.8.17-cp311-cp311-win_amd64.whl", hash = "sha256:b532282ad4eca958b1b2d7dbcb2b7218e02cb934165859b918e3b6ba7772d3f4", size = 5179011, upload-time = "2025-09-17T16:33:35.711Z" },
    { url = "https://files.pythonhosted.org/packages/08/2b/9d8e65beb2751876c82e1aceb32f328c43ec872711fa80257c7674f45650/debugpy-1.8.17-cp312-cp312-macosx_15_0_universal2.whl", hash = "sha256:f14467edef672195c6f6b8e27ce5005313cb5d03c9239059bc7182b60c176e2d", size = 2549522, upload-time = "2025-09-17T16:33:38.466Z" },
    { url = "https://files.pythonhosted.org/packages/b4/78/eb0d77f02971c05fca0eb7465b18058ba84bd957062f5eec82f941ac792a/debugpy-1.8.17-cp312-cp312-manylinux_2_34_x86_64.whl", hash = "sha256:24693179ef9dfa20dca8605905a42b392be56d410c333af82f1c5dff807a64cc", size = 4309417, upload-time = "2025-09-17T16:33:41.299Z" },
    { url = "https://files.pythonhosted.org/packages/37/42/c40f1d8cc1fed1e75ea54298a382395b8b937d923fcf41ab0797a554f555/debugpy-1.8.17-cp312-cp312-win32.whl", hash = "sha256:6a4e9dacf2cbb60d2514ff7b04b4534b0139facbf2abdffe0639ddb6088e59cf", size = 5277130, upload-time = "2025-09-17T16:33:43.554Z" },
    { url = "https://files.pythonhosted.org/packages/72/22/84263b205baad32b81b36eac076de0cdbe09fe2d0637f5b32243dc7c925b/debugpy-1.8.17-cp312-cp312-win_amd64.whl", hash = "sha256:e8f8f61c518952fb15f74a302e068b48d9c4691768ade433e4adeea961993464", size = 5319053, upload-time = "2025-09-17T16:33:53.033Z" },
    { url = "https://files.pythonhosted.org/packages/50/76/597e5cb97d026274ba297af8d89138dfd9e695767ba0e0895edb20963f40/debugpy-1.8.17-cp313-cp313-macosx_15_0_universal2.whl", hash = "sha256:857c1dd5d70042502aef1c6d1c2801211f3ea7e56f75e9c335f434afb403e464", size = 2538386, upload-time = "2025-09-17T16:33:54.594Z" },
    { url = "https://files.pythonhosted.org/packages/5f/60/ce5c34fcdfec493701f9d1532dba95b21b2f6394147234dce21160bd923f/debugpy-1.8.17-cp313-cp313-manylinux_2_34_x86_64.whl", hash = "sha256:3bea3b0b12f3946e098cce9b43c3c46e317b567f79570c3f43f0b96d00788088", size = 4292100, upload-time = "2025-09-17T16:33:56.353Z" },
    { url = "https://files.pythonhosted.org/packages/e8/95/7873cf2146577ef71d2a20bf553f12df865922a6f87b9e8ee1df04f01785/debugpy-1.8.17-cp313-cp313-win32.whl", hash = "sha256:e34ee844c2f17b18556b5bbe59e1e2ff4e86a00282d2a46edab73fd7f18f4a83", size = 5277002, upload-time = "2025-09-17T16:33:58.231Z" },
    { url = "https://files.pythonhosted.org/packages/46/11/18c79a1cee5ff539a94ec4aa290c1c069a5580fd5cfd2fb2e282f8e905da/debugpy-1.8.17-cp313-cp313-win_amd64.whl", hash = "sha256:6c5cd6f009ad4fca8e33e5238210dc1e5f42db07d4b6ab21ac7ffa904a196420", size = 5319047, upload-time = "2025-09-17T16:34:00.586Z" },
    { url = "https://files.pythonhosted.org/packages/de/45/115d55b2a9da6de812696064ceb505c31e952c5d89c4ed1d9bb983deec34/debugpy-1.8.17-cp314-cp314-macosx_15_0_universal2.whl", hash = "sha256:045290c010bcd2d82bc97aa2daf6837443cd52f6328592698809b4549babcee1", size = 2536899, upload-time = "2025-09-17T16:34:02.657Z" },
    { url = "https://files.pythonhosted.org/packages/5a/73/2aa00c7f1f06e997ef57dc9b23d61a92120bec1437a012afb6d176585197/debugpy-1.8.17-cp314-cp314-manylinux_2_34_x86_64.whl", hash = "sha256:b69b6bd9dba6a03632534cdf67c760625760a215ae289f7489a452af1031fe1f", size = 4268254, upload-time = "2025-09-17T16:34:04.486Z" },
    { url = "https://files.pythonhosted.org/packages/86/b5/ed3e65c63c68a6634e3ba04bd10255c8e46ec16ebed7d1c79e4816d8a760/debugpy-1.8.17-cp314-cp314-win32.whl", hash = "sha256:5c59b74aa5630f3a5194467100c3b3d1c77898f9ab27e3f7dc5d40fc2f122670", size = 5277203, upload-time = "2025-09-17T16:34:06.65Z" },
    { url = "https://files.pythonhosted.org/packages/b0/26/394276b71c7538445f29e792f589ab7379ae70fd26ff5577dfde71158e96/debugpy-1.8.17-cp314-cp314-win_amd64.whl", hash = "sha256:893cba7bb0f55161de4365584b025f7064e1f88913551bcd23be3260b231429c", size = 5318493, upload-time = "2025-09-17T16:34:08.483Z" },
    { url = "https://files.pythonhosted.org/packages/b0/d0/89247ec250369fc76db477720a26b2fce7ba079ff1380e4ab4529d2fe233/debugpy-1.8.17-py2.py3-none-any.whl", hash = "sha256:60c7dca6571efe660ccb7a9508d73ca14b8796c4ed484c2002abba714226cfef", size = 5283210, upload-time = "2025-09-17T16:34:25.835Z" },
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
name = "defusedxml"
version = "0.7.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/0f/d5/c66da9b79e5bdb124974bfe172b4daf3c984ebd9c2a06e2b8a4dc7331c72/defusedxml-0.7.1.tar.gz", hash = "sha256:1bb3032db185915b62d7c6209c5a8792be6a32ab2fedacc84e01b52c51aa3e69", size = 75520, upload-time = "2021-03-08T10:59:26.269Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/07/6c/aa3f2f849e01cb6a001cd8554a88d4c77c5c1a31c95bdf1cf9301e6d9ef4/defusedxml-0.7.1-py2.py3-none-any.whl", hash = "sha256:a352e7e428770286cc899e2542b6cdaedb2b4953ff269a210103ec58f6198a61", size = 25604, upload-time = "2021-03-08T10:59:24.45Z" },
]

[[package]]
name = "exceptiongroup"
version = "1.3.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "typing-extensions", marker = "python_full_version < '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/0b/9f/a65090624ecf468cdca03533906e7c69ed7588582240cfe7cc9e770b50eb/exceptiongroup-1.3.0.tar.gz", hash = "sha256:b241f5885f560bc56a59ee63ca4c6a8bfa46ae4ad651af316d4e81817bb9fd88", size = 29749, upload-time = "2025-05-10T17:42:51.123Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/36/f4/c6e662dade71f56cd2f3735141b265c3c79293c109549c1e6933b0651ffc/exceptiongroup-1.3.0-py3-none-any.whl", hash = "sha256:4d111e6e0c13d0644cad6ddaa7ed0261a0b36971f6d23e7ec9b4b9097da78a10", size = 16674, upload-time = "2025-05-10T17:42:49.33Z" },
]

[[package]]
name = "execnet"
version = "2.1.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/bb/ff/b4c0dc78fbe20c3e59c0c7334de0c27eb4001a2b2017999af398bf730817/execnet-2.1.1.tar.gz", hash = "sha256:5189b52c6121c24feae288166ab41b32549c7e2348652736540b9e6e7d4e72e3", size = 166524, upload-time = "2024-04-08T09:04:19.245Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/43/09/2aea36ff60d16dd8879bdb2f5b3ee0ba8d08cbbdcdfe870e695ce3784385/execnet-2.1.1-py3-none-any.whl", hash = "sha256:26dee51f1b80cebd6d0ca8e74dd8745419761d3bef34163928cbebbdc4749fdc", size = 40612, upload-time = "2024-04-08T09:04:17.414Z" },
]

[[package]]
name = "executing"
version = "2.2.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/cc/28/c14e053b6762b1044f34a13aab6859bbf40456d37d23aa286ac24cfd9a5d/executing-2.2.1.tar.gz", hash = "sha256:3632cc370565f6648cc328b32435bd120a1e4ebb20c77e3fdde9a13cd1e533c4", size = 1129488, upload-time = "2025-09-01T09:48:10.866Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/c1/ea/53f2148663b321f21b5a606bd5f191517cf40b7072c0497d3c92c4a13b1e/executing-2.2.1-py2.py3-none-any.whl", hash = "sha256:760643d3452b4d777d295bb167ccc74c64a81df23fb5e08eff250c425a4b2017", size = 28317, upload-time = "2025-09-01T09:48:08.5Z" },
]

[[package]]
name = "fastjsonschema"
version = "2.21.2"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/20/b5/23b216d9d985a956623b6bd12d4086b60f0059b27799f23016af04a74ea1/fastjsonschema-2.21.2.tar.gz", hash = "sha256:b1eb43748041c880796cd077f1a07c3d94e93ae84bba5ed36800a33554ae05de", size = 374130, upload-time = "2025-08-14T18:49:36.666Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/cb/a8/20d0723294217e47de6d9e2e40fd4a9d2f7c4b6ef974babd482a59743694/fastjsonschema-2.21.2-py3-none-any.whl", hash = "sha256:1c797122d0a86c5cace2e54bf4e819c36223b552017172f32c5c024a6b77e463", size = 24024, upload-time = "2025-08-14T18:49:34.776Z" },
]

[[package]]
name = "forbiddenfruit"
version = "0.1.4"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/e6/79/d4f20e91327c98096d605646bdc6a5ffedae820f38d378d3515c42ec5e60/forbiddenfruit-0.1.4.tar.gz", hash = "sha256:e3f7e66561a29ae129aac139a85d610dbf3dd896128187ed5454b6421f624253", size = 43756, upload-time = "2021-01-16T21:03:35.401Z" }

[[package]]
name = "fqdn"
version = "1.5.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/30/3e/a80a8c077fd798951169626cde3e239adeba7dab75deb3555716415bd9b0/fqdn-1.5.1.tar.gz", hash = "sha256:105ed3677e767fb5ca086a0c1f4bb66ebc3c100be518f0e0d755d9eae164d89f", size = 6015, upload-time = "2021-03-11T07:16:29.08Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/cf/58/8acf1b3e91c58313ce5cb67df61001fc9dcd21be4fadb76c1a2d540e09ed/fqdn-1.5.1-py3-none-any.whl", hash = "sha256:3a179af3761e4df6eb2e026ff9e1a3033d3587bf980a0b1b2e1e5d08d7358014", size = 9121, upload-time = "2021-03-11T07:16:28.351Z" },
]

[[package]]
name = "googleapis-common-protos"
version = "1.71.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "protobuf", marker = "python_full_version >= '3.11' and python_full_version < '3.14'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/30/43/b25abe02db2911397819003029bef768f68a974f2ece483e6084d1a5f754/googleapis_common_protos-1.71.0.tar.gz", hash = "sha256:1aec01e574e29da63c80ba9f7bbf1ccfaacf1da877f23609fe236ca7c72a2e2e", size = 146454, upload-time = "2025-10-20T14:58:08.732Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/25/e8/eba9fece11d57a71e3e22ea672742c8f3cf23b35730c9e96db768b295216/googleapis_common_protos-1.71.0-py3-none-any.whl", hash = "sha256:59034a1d849dc4d18971997a72ac56246570afdd17f9369a0ff68218d50ab78c", size = 294576, upload-time = "2025-10-20T14:56:21.295Z" },
]

[[package]]
name = "grpcio"
version = "1.75.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "typing-extensions", marker = "python_full_version >= '3.11' and python_full_version < '3.14'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/9d/f7/8963848164c7604efb3a3e6ee457fdb3a469653e19002bd24742473254f8/grpcio-1.75.1.tar.gz", hash = "sha256:3e81d89ece99b9ace23a6916880baca613c03a799925afb2857887efa8b1b3d2", size = 12731327, upload-time = "2025-09-26T09:03:36.887Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/51/57/89fd829fb00a6d0bee3fbcb2c8a7aa0252d908949b6ab58bfae99d39d77e/grpcio-1.75.1-cp310-cp310-linux_armv7l.whl", hash = "sha256:1712b5890b22547dd29f3215c5788d8fc759ce6dd0b85a6ba6e2731f2d04c088", size = 5705534, upload-time = "2025-09-26T09:00:52.225Z" },
    { url = "https://files.pythonhosted.org/packages/76/dd/2f8536e092551cf804e96bcda79ecfbc51560b214a0f5b7ebc253f0d4664/grpcio-1.75.1-cp310-cp310-macosx_11_0_universal2.whl", hash = "sha256:8d04e101bba4b55cea9954e4aa71c24153ba6182481b487ff376da28d4ba46cf", size = 11484103, upload-time = "2025-09-26T09:00:59.457Z" },
    { url = "https://files.pythonhosted.org/packages/9a/3d/affe2fb897804c98d56361138e73786af8f4dd876b9d9851cfe6342b53c8/grpcio-1.75.1-cp310-cp310-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:683cfc70be0c1383449097cba637317e4737a357cfc185d887fd984206380403", size = 6289953, upload-time = "2025-09-26T09:01:03.699Z" },
    { url = "https://files.pythonhosted.org/packages/87/aa/0f40b7f47a0ff10d7e482bc3af22dac767c7ff27205915f08962d5ca87a2/grpcio-1.75.1-cp310-cp310-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:491444c081a54dcd5e6ada57314321ae526377f498d4aa09d975c3241c5b9e1c", size = 6949785, upload-time = "2025-09-26T09:01:07.504Z" },
    { url = "https://files.pythonhosted.org/packages/a5/45/b04407e44050781821c84f26df71b3f7bc469923f92f9f8bc27f1406dbcc/grpcio-1.75.1-cp310-cp310-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:ce08d4e112d0d38487c2b631ec8723deac9bc404e9c7b1011426af50a79999e4", size = 6465708, upload-time = "2025-09-26T09:01:11.028Z" },
    { url = "https://files.pythonhosted.org/packages/09/3e/4ae3ec0a4d20dcaafbb6e597defcde06399ccdc5b342f607323f3b47f0a3/grpcio-1.75.1-cp310-cp310-musllinux_1_2_aarch64.whl", hash = "sha256:5a2acda37fc926ccc4547977ac3e56b1df48fe200de968e8c8421f6e3093df6c", size = 7100912, upload-time = "2025-09-26T09:01:14.393Z" },
    { url = "https://files.pythonhosted.org/packages/34/3f/a9085dab5c313bb0cb853f222d095e2477b9b8490a03634cdd8d19daa5c3/grpcio-1.75.1-cp310-cp310-musllinux_1_2_i686.whl", hash = "sha256:745c5fe6bf05df6a04bf2d11552c7d867a2690759e7ab6b05c318a772739bd75", size = 8042497, upload-time = "2025-09-26T09:01:17.759Z" },
    { url = "https://files.pythonhosted.org/packages/c3/87/ea54eba931ab9ed3f999ba95f5d8d01a20221b664725bab2fe93e3dee848/grpcio-1.75.1-cp310-cp310-musllinux_1_2_x86_64.whl", hash = "sha256:259526a7159d39e2db40d566fe3e8f8e034d0fb2db5bf9c00e09aace655a4c2b", size = 7493284, upload-time = "2025-09-26T09:01:20.896Z" },
    { url = "https://files.pythonhosted.org/packages/b7/5e/287f1bf1a998f4ac46ef45d518de3b5da08b4e86c7cb5e1108cee30b0282/grpcio-1.75.1-cp310-cp310-win32.whl", hash = "sha256:f4b29b9aabe33fed5df0a85e5f13b09ff25e2c05bd5946d25270a8bd5682dac9", size = 3950809, upload-time = "2025-09-26T09:01:23.695Z" },
    { url = "https://files.pythonhosted.org/packages/a4/a2/3cbfc06a4ec160dc77403b29ecb5cf76ae329eb63204fea6a7c715f1dfdb/grpcio-1.75.1-cp310-cp310-win_amd64.whl", hash = "sha256:cf2e760978dcce7ff7d465cbc7e276c3157eedc4c27aa6de7b594c7a295d3d61", size = 4644704, upload-time = "2025-09-26T09:01:25.763Z" },
    { url = "https://files.pythonhosted.org/packages/0c/3c/35ca9747473a306bfad0cee04504953f7098527cd112a4ab55c55af9e7bd/grpcio-1.75.1-cp311-cp311-linux_armv7l.whl", hash = "sha256:573855ca2e58e35032aff30bfbd1ee103fbcf4472e4b28d4010757700918e326", size = 5709761, upload-time = "2025-09-26T09:01:28.528Z" },
    { url = "https://files.pythonhosted.org/packages/c9/2c/ecbcb4241e4edbe85ac2663f885726fea0e947767401288b50d8fdcb9200/grpcio-1.75.1-cp311-cp311-macosx_11_0_universal2.whl", hash = "sha256:6a4996a2c8accc37976dc142d5991adf60733e223e5c9a2219e157dc6a8fd3a2", size = 11496691, upload-time = "2025-09-26T09:01:31.214Z" },
    { url = "https://files.pythonhosted.org/packages/81/40/bc07aee2911f0d426fa53fe636216100c31a8ea65a400894f280274cb023/grpcio-1.75.1-cp311-cp311-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:b1ea1bbe77ecbc1be00af2769f4ae4a88ce93be57a4f3eebd91087898ed749f9", size = 6296084, upload-time = "2025-09-26T09:01:34.596Z" },
    { url = "https://files.pythonhosted.org/packages/b8/d1/10c067f6c67396cbf46448b80f27583b5e8c4b46cdfbe18a2a02c2c2f290/grpcio-1.75.1-cp311-cp311-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:e5b425aee54cc5e3e3c58f00731e8a33f5567965d478d516d35ef99fd648ab68", size = 6950403, upload-time = "2025-09-26T09:01:36.736Z" },
    { url = "https://files.pythonhosted.org/packages/3f/42/5f628abe360b84dfe8dd8f32be6b0606dc31dc04d3358eef27db791ea4d5/grpcio-1.75.1-cp311-cp311-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:0049a7bf547dafaeeb1db17079ce79596c298bfe308fc084d023c8907a845b9a", size = 6470166, upload-time = "2025-09-26T09:01:39.474Z" },
    { url = "https://files.pythonhosted.org/packages/c3/93/a24035080251324019882ee2265cfde642d6476c0cf8eb207fc693fcebdc/grpcio-1.75.1-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:5b8ea230c7f77c0a1a3208a04a1eda164633fb0767b4cefd65a01079b65e5b1f", size = 7107828, upload-time = "2025-09-26T09:01:41.782Z" },
    { url = "https://files.pythonhosted.org/packages/e4/f8/d18b984c1c9ba0318e3628dbbeb6af77a5007f02abc378c845070f2d3edd/grpcio-1.75.1-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:36990d629c3c9fb41e546414e5af52d0a7af37ce7113d9682c46d7e2919e4cca", size = 8045421, upload-time = "2025-09-26T09:01:45.835Z" },
    { url = "https://files.pythonhosted.org/packages/7e/b6/4bf9aacff45deca5eac5562547ed212556b831064da77971a4e632917da3/grpcio-1.75.1-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:b10ad908118d38c2453ade7ff790e5bce36580c3742919007a2a78e3a1e521ca", size = 7503290, upload-time = "2025-09-26T09:01:49.28Z" },
    { url = "https://files.pythonhosted.org/packages/3b/15/d8d69d10223cb54c887a2180bd29fe5fa2aec1d4995c8821f7aa6eaf72e4/grpcio-1.75.1-cp311-cp311-win32.whl", hash = "sha256:d6be2b5ee7bea656c954dcf6aa8093c6f0e6a3ef9945c99d99fcbfc88c5c0bfe", size = 3950631, upload-time = "2025-09-26T09:01:51.23Z" },
    { url = "https://files.pythonhosted.org/packages/8a/40/7b8642d45fff6f83300c24eaac0380a840e5e7fe0e8d80afd31b99d7134e/grpcio-1.75.1-cp311-cp311-win_amd64.whl", hash = "sha256:61c692fb05956b17dd6d1ab480f7f10ad0536dba3bc8fd4e3c7263dc244ed772", size = 4646131, upload-time = "2025-09-26T09:01:53.266Z" },
    { url = "https://files.pythonhosted.org/packages/3a/81/42be79e73a50aaa20af66731c2defeb0e8c9008d9935a64dd8ea8e8c44eb/grpcio-1.75.1-cp312-cp312-linux_armv7l.whl", hash = "sha256:7b888b33cd14085d86176b1628ad2fcbff94cfbbe7809465097aa0132e58b018", size = 5668314, upload-time = "2025-09-26T09:01:55.424Z" },
    { url = "https://files.pythonhosted.org/packages/c5/a7/3686ed15822fedc58c22f82b3a7403d9faf38d7c33de46d4de6f06e49426/grpcio-1.75.1-cp312-cp312-macosx_11_0_universal2.whl", hash = "sha256:8775036efe4ad2085975531d221535329f5dac99b6c2a854a995456098f99546", size = 11476125, upload-time = "2025-09-26T09:01:57.927Z" },
    { url = "https://files.pythonhosted.org/packages/14/85/21c71d674f03345ab183c634ecd889d3330177e27baea8d5d247a89b6442/grpcio-1.75.1-cp312-cp312-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:bb658f703468d7fbb5dcc4037c65391b7dc34f808ac46ed9136c24fc5eeb041d", size = 6246335, upload-time = "2025-09-26T09:02:00.76Z" },
    { url = "https://files.pythonhosted.org/packages/fd/db/3beb661bc56a385ae4fa6b0e70f6b91ac99d47afb726fe76aaff87ebb116/grpcio-1.75.1-cp312-cp312-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:4b7177a1cdb3c51b02b0c0a256b0a72fdab719600a693e0e9037949efffb200b", size = 6916309, upload-time = "2025-09-26T09:02:02.894Z" },
    { url = "https://files.pythonhosted.org/packages/1e/9c/eda9fe57f2b84343d44c1b66cf3831c973ba29b078b16a27d4587a1fdd47/grpcio-1.75.1-cp312-cp312-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:7d4fa6ccc3ec2e68a04f7b883d354d7fea22a34c44ce535a2f0c0049cf626ddf", size = 6435419, upload-time = "2025-09-26T09:02:05.055Z" },
    { url = "https://files.pythonhosted.org/packages/c3/b8/090c98983e0a9d602e3f919a6e2d4e470a8b489452905f9a0fa472cac059/grpcio-1.75.1-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:3d86880ecaeb5b2f0a8afa63824de93adb8ebe4e49d0e51442532f4e08add7d6", size = 7064893, upload-time = "2025-09-26T09:02:07.275Z" },
    { url = "https://files.pythonhosted.org/packages/ec/c0/6d53d4dbbd00f8bd81571f5478d8a95528b716e0eddb4217cc7cb45aae5f/grpcio-1.75.1-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:a8041d2f9e8a742aeae96f4b047ee44e73619f4f9d24565e84d5446c623673b6", size = 8011922, upload-time = "2025-09-26T09:02:09.527Z" },
    { url = "https://files.pythonhosted.org/packages/f2/7c/48455b2d0c5949678d6982c3e31ea4d89df4e16131b03f7d5c590811cbe9/grpcio-1.75.1-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:3652516048bf4c314ce12be37423c79829f46efffb390ad64149a10c6071e8de", size = 7466181, upload-time = "2025-09-26T09:02:12.279Z" },
    { url = "https://files.pythonhosted.org/packages/fd/12/04a0e79081e3170b6124f8cba9b6275871276be06c156ef981033f691880/grpcio-1.75.1-cp312-cp312-win32.whl", hash = "sha256:44b62345d8403975513af88da2f3d5cc76f73ca538ba46596f92a127c2aea945", size = 3938543, upload-time = "2025-09-26T09:02:14.77Z" },
    { url = "https://files.pythonhosted.org/packages/5f/d7/11350d9d7fb5adc73d2b0ebf6ac1cc70135577701e607407fe6739a90021/grpcio-1.75.1-cp312-cp312-win_amd64.whl", hash = "sha256:b1e191c5c465fa777d4cafbaacf0c01e0d5278022082c0abbd2ee1d6454ed94d", size = 4641938, upload-time = "2025-09-26T09:02:16.927Z" },
    { url = "https://files.pythonhosted.org/packages/46/74/bac4ab9f7722164afdf263ae31ba97b8174c667153510322a5eba4194c32/grpcio-1.75.1-cp313-cp313-linux_armv7l.whl", hash = "sha256:3bed22e750d91d53d9e31e0af35a7b0b51367e974e14a4ff229db5b207647884", size = 5672779, upload-time = "2025-09-26T09:02:19.11Z" },
    { url = "https://files.pythonhosted.org/packages/a6/52/d0483cfa667cddaa294e3ab88fd2c2a6e9dc1a1928c0e5911e2e54bd5b50/grpcio-1.75.1-cp313-cp313-macosx_11_0_universal2.whl", hash = "sha256:5b8f381eadcd6ecaa143a21e9e80a26424c76a0a9b3d546febe6648f3a36a5ac", size = 11470623, upload-time = "2025-09-26T09:02:22.117Z" },
    { url = "https://files.pythonhosted.org/packages/cf/e4/d1954dce2972e32384db6a30273275e8c8ea5a44b80347f9055589333b3f/grpcio-1.75.1-cp313-cp313-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:5bf4001d3293e3414d0cf99ff9b1139106e57c3a66dfff0c5f60b2a6286ec133", size = 6248838, upload-time = "2025-09-26T09:02:26.426Z" },
    { url = "https://files.pythonhosted.org/packages/06/43/073363bf63826ba8077c335d797a8d026f129dc0912b69c42feaf8f0cd26/grpcio-1.75.1-cp313-cp313-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:9f82ff474103e26351dacfe8d50214e7c9322960d8d07ba7fa1d05ff981c8b2d", size = 6922663, upload-time = "2025-09-26T09:02:28.724Z" },
    { url = "https://files.pythonhosted.org/packages/c2/6f/076ac0df6c359117676cacfa8a377e2abcecec6a6599a15a672d331f6680/grpcio-1.75.1-cp313-cp313-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:0ee119f4f88d9f75414217823d21d75bfe0e6ed40135b0cbbfc6376bc9f7757d", size = 6436149, upload-time = "2025-09-26T09:02:30.971Z" },
    { url = "https://files.pythonhosted.org/packages/6b/27/1d08824f1d573fcb1fa35ede40d6020e68a04391709939e1c6f4193b445f/grpcio-1.75.1-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:664eecc3abe6d916fa6cf8dd6b778e62fb264a70f3430a3180995bf2da935446", size = 7067989, upload-time = "2025-09-26T09:02:33.233Z" },
    { url = "https://files.pythonhosted.org/packages/c6/98/98594cf97b8713feb06a8cb04eeef60b4757e3e2fb91aa0d9161da769843/grpcio-1.75.1-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:c32193fa08b2fbebf08fe08e84f8a0aad32d87c3ad42999c65e9449871b1c66e", size = 8010717, upload-time = "2025-09-26T09:02:36.011Z" },
    { url = "https://files.pythonhosted.org/packages/8c/7e/bb80b1bba03c12158f9254762cdf5cced4a9bc2e8ed51ed335915a5a06ef/grpcio-1.75.1-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:5cebe13088b9254f6e615bcf1da9131d46cfa4e88039454aca9cb65f639bd3bc", size = 7463822, upload-time = "2025-09-26T09:02:38.26Z" },
    { url = "https://files.pythonhosted.org/packages/23/1c/1ea57fdc06927eb5640f6750c697f596f26183573069189eeaf6ef86ba2d/grpcio-1.75.1-cp313-cp313-win32.whl", hash = "sha256:4b4c678e7ed50f8ae8b8dbad15a865ee73ce12668b6aaf411bf3258b5bc3f970", size = 3938490, upload-time = "2025-09-26T09:02:40.268Z" },
    { url = "https://files.pythonhosted.org/packages/4b/24/fbb8ff1ccadfbf78ad2401c41aceaf02b0d782c084530d8871ddd69a2d49/grpcio-1.75.1-cp313-cp313-win_amd64.whl", hash = "sha256:5573f51e3f296a1bcf71e7a690c092845fb223072120f4bdb7a5b48e111def66", size = 4642538, upload-time = "2025-09-26T09:02:42.519Z" },
    { url = "https://files.pythonhosted.org/packages/f2/1b/9a0a5cecd24302b9fdbcd55d15ed6267e5f3d5b898ff9ac8cbe17ee76129/grpcio-1.75.1-cp314-cp314-linux_armv7l.whl", hash = "sha256:c05da79068dd96723793bffc8d0e64c45f316248417515f28d22204d9dae51c7", size = 5673319, upload-time = "2025-09-26T09:02:44.742Z" },
    { url = "https://files.pythonhosted.org/packages/c6/ec/9d6959429a83fbf5df8549c591a8a52bb313976f6646b79852c4884e3225/grpcio-1.75.1-cp314-cp314-macosx_11_0_universal2.whl", hash = "sha256:06373a94fd16ec287116a825161dca179a0402d0c60674ceeec8c9fba344fe66", size = 11480347, upload-time = "2025-09-26T09:02:47.539Z" },
    { url = "https://files.pythonhosted.org/packages/09/7a/26da709e42c4565c3d7bf999a9569da96243ce34a8271a968dee810a7cf1/grpcio-1.75.1-cp314-cp314-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:4484f4b7287bdaa7a5b3980f3c7224c3c622669405d20f69549f5fb956ad0421", size = 6254706, upload-time = "2025-09-26T09:02:50.4Z" },
    { url = "https://files.pythonhosted.org/packages/f1/08/dcb26a319d3725f199c97e671d904d84ee5680de57d74c566a991cfab632/grpcio-1.75.1-cp314-cp314-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:2720c239c1180eee69f7883c1d4c83fc1a495a2535b5fa322887c70bf02b16e8", size = 6922501, upload-time = "2025-09-26T09:02:52.711Z" },
    { url = "https://files.pythonhosted.org/packages/78/66/044d412c98408a5e23cb348845979a2d17a2e2b6c3c34c1ec91b920f49d0/grpcio-1.75.1-cp314-cp314-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:07a554fa31c668cf0e7a188678ceeca3cb8fead29bbe455352e712ec33ca701c", size = 6437492, upload-time = "2025-09-26T09:02:55.542Z" },
    { url = "https://files.pythonhosted.org/packages/4e/9d/5e3e362815152aa1afd8b26ea613effa005962f9da0eec6e0e4527e7a7d1/grpcio-1.75.1-cp314-cp314-musllinux_1_2_aarch64.whl", hash = "sha256:3e71a2105210366bfc398eef7f57a664df99194f3520edb88b9c3a7e46ee0d64", size = 7081061, upload-time = "2025-09-26T09:02:58.261Z" },
    { url = "https://files.pythonhosted.org/packages/1e/1a/46615682a19e100f46e31ddba9ebc297c5a5ab9ddb47b35443ffadb8776c/grpcio-1.75.1-cp314-cp314-musllinux_1_2_i686.whl", hash = "sha256:8679aa8a5b67976776d3c6b0521e99d1c34db8a312a12bcfd78a7085cb9b604e", size = 8010849, upload-time = "2025-09-26T09:03:00.548Z" },
    { url = "https://files.pythonhosted.org/packages/67/8e/3204b94ac30b0f675ab1c06540ab5578660dc8b690db71854d3116f20d00/grpcio-1.75.1-cp314-cp314-musllinux_1_2_x86_64.whl", hash = "sha256:aad1c774f4ebf0696a7f148a56d39a3432550612597331792528895258966dc0", size = 7464478, upload-time = "2025-09-26T09:03:03.096Z" },
    { url = "https://files.pythonhosted.org/packages/b7/97/2d90652b213863b2cf466d9c1260ca7e7b67a16780431b3eb1d0420e3d5b/grpcio-1.75.1-cp314-cp314-win32.whl", hash = "sha256:62ce42d9994446b307649cb2a23335fa8e927f7ab2cbf5fcb844d6acb4d85f9c", size = 4012672, upload-time = "2025-09-26T09:03:05.477Z" },
    { url = "https://files.pythonhosted.org/packages/f9/df/e2e6e9fc1c985cd1a59e6996a05647c720fe8a03b92f5ec2d60d366c531e/grpcio-1.75.1-cp314-cp314-win_amd64.whl", hash = "sha256:f86e92275710bea3000cb79feca1762dc0ad3b27830dd1a74e82ab321d4ee464", size = 4772475, upload-time = "2025-09-26T09:03:07.661Z" },
]

[[package]]
name = "grpcio-tools"
version = "1.75.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "grpcio", marker = "python_full_version >= '3.11' and python_full_version < '3.14'" },
    { name = "protobuf", marker = "python_full_version >= '3.11' and python_full_version < '3.14'" },
    { name = "setuptools", marker = "python_full_version >= '3.11' and python_full_version < '3.14'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/7d/76/0cd2a2bb379275c319544a3ab613dc3cea7a167503908c1b4de55f82bd9e/grpcio_tools-1.75.1.tar.gz", hash = "sha256:bb78960cf3d58941e1fec70cbdaccf255918beed13c34112a6915a6d8facebd1", size = 5390470, upload-time = "2025-09-26T09:10:11.948Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/26/b7/7d1b0b7669f993a6a393083a876937f478ca034c283eb23baf6720d8c85a/grpcio_tools-1.75.1-cp310-cp310-linux_armv7l.whl", hash = "sha256:ae0f04d5ec8b8e13476bf516a08fc1de4e58c6bf79f99123a6b964ca7d02c790", size = 2545419, upload-time = "2025-09-26T09:07:44.432Z" },
    { url = "https://files.pythonhosted.org/packages/3e/c0/db5d052d1ba5e859c833d1366960d784c0b44c8330012717aeaa123b6b9f/grpcio_tools-1.75.1-cp310-cp310-macosx_11_0_universal2.whl", hash = "sha256:24a881ad7292e904fc256892b647da17d9137ef2e72faf8b7c8e515314ad1377", size = 5841650, upload-time = "2025-09-26T09:07:50.987Z" },
    { url = "https://files.pythonhosted.org/packages/af/13/ab49230ef106f2b9de156a813bc14049e6fd4fe9c26fa0cde496f0e86a09/grpcio_tools-1.75.1-cp310-cp310-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:1b5810ace274dba12ecfac69ac32c8047c6ee0200a23274cb4885ed4187271f8", size = 2591560, upload-time = "2025-09-26T09:07:52.777Z" },
    { url = "https://files.pythonhosted.org/packages/29/fd/1fd3069fb0559c2f90d85b0fd3a73adc3f63966c6300fee01e4e52740229/grpcio_tools-1.75.1-cp310-cp310-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:ab33993288b97b1180e092fa447a8ce00fbc8c59d67b23553245b88d14fe36bb", size = 2904895, upload-time = "2025-09-26T09:07:55.002Z" },
    { url = "https://files.pythonhosted.org/packages/d6/51/e58fae40132a4589819c388333545a33a89f91b8affcac45623ace9ca659/grpcio_tools-1.75.1-cp310-cp310-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:4cac693621043ef11d3ab2318e811d919779f8cd5011ba8e37f44c178c831d94", size = 2656151, upload-time = "2025-09-26T09:07:56.762Z" },
    { url = "https://files.pythonhosted.org/packages/39/c9/a33736c2a8ceee39991f2c9f67a426ab799c6caf09145120bae6080428d5/grpcio_tools-1.75.1-cp310-cp310-musllinux_1_2_aarch64.whl", hash = "sha256:a09cd5d267b296af67116fe098633ad770bc8c19831a5f3c896f65fad90c1064", size = 3105152, upload-time = "2025-09-26T09:07:58.702Z" },
    { url = "https://files.pythonhosted.org/packages/ac/71/2f09cbe321f057a47a8ae4dacf004bbe8a171fd712b8f7f689ec7b2f1c49/grpcio_tools-1.75.1-cp310-cp310-musllinux_1_2_i686.whl", hash = "sha256:dff4bcb4d16cf9ef745c1984394ed15187e6c23d73d71377377deaf443d11358", size = 3654551, upload-time = "2025-09-26T09:08:00.87Z" },
    { url = "https://files.pythonhosted.org/packages/05/54/91481a5b96cab2a81326ab1041fd7cfc6b6ce0cd82bab14ebcdb6ed78d4e/grpcio_tools-1.75.1-cp310-cp310-musllinux_1_2_x86_64.whl", hash = "sha256:16d5986b37e2a9203f85e456c7ff8705b932718021d408adfe4a79e0f4d95949", size = 3322220, upload-time = "2025-09-26T09:08:02.724Z" },
    { url = "https://files.pythonhosted.org/packages/a2/07/272955f15a35ef0069ebe17a5fc14282c6dc6690edeb4dbe6a81dd2d1efb/grpcio_tools-1.75.1-cp310-cp310-win32.whl", hash = "sha256:3fbac14998bfadc6b9140b6339dbc5f673700ebb4d45ba0c4d4fbe0ffb8559a9", size = 992986, upload-time = "2025-09-26T09:08:04.6Z" },
    { url = "https://files.pythonhosted.org/packages/16/9a/482b05c1277b3385be7e426f000efb921ce3ae76bcb8aa4f9b9f724c58d3/grpcio_tools-1.75.1-cp310-cp310-win_amd64.whl", hash = "sha256:b56e495844eb899de721eb77d9e077192bdeb40842f598481d32a8f6de3db124", size = 1157427, upload-time = "2025-09-26T09:08:06.246Z" },
    { url = "https://files.pythonhosted.org/packages/45/28/71ab934662d41ded4e451d9af0ec6f9aade3525e470fdfd10bd20e588e44/grpcio_tools-1.75.1-cp311-cp311-linux_armv7l.whl", hash = "sha256:f0635231feb70a9d551452829943a1a5fa651283e7a300aadc22df5ea5da696f", size = 2545461, upload-time = "2025-09-26T09:08:08.514Z" },
    { url = "https://files.pythonhosted.org/packages/69/40/d90f6fdb51f51b2a518401207b3920fcfdfa996ed7bca844096f111ed839/grpcio_tools-1.75.1-cp311-cp311-macosx_11_0_universal2.whl", hash = "sha256:626293296ef7e2d87ab1a80b81a55eef91883c65b59a97576099a28b9535100b", size = 5842958, upload-time = "2025-09-26T09:08:11.468Z" },
    { url = "https://files.pythonhosted.org/packages/b4/b7/52e6f32fd0101e3ac9c654a6441b254ba5874f146b543b20afbcb8246947/grpcio_tools-1.75.1-cp311-cp311-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:071339d90f1faab332ce4919c815a10b9c3ed2c09473f550f686bf9cc148579f", size = 2591669, upload-time = "2025-09-26T09:08:13.481Z" },
    { url = "https://files.pythonhosted.org/packages/0a/3c/115c59a5c0c8e9d7d99a40bac8d5e91c05b6735b3bb185265d40e9fc4346/grpcio_tools-1.75.1-cp311-cp311-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:44195f58c052fa935b78c7438c85cbcd4b273dd685028e4f6d4d7b30d47daad1", size = 2904952, upload-time = "2025-09-26T09:08:15.299Z" },
    { url = "https://files.pythonhosted.org/packages/a9/cd/d2a3583a5b1d71da88f7998f20fb5a0b6fe5bb96bb916a610c29269063b6/grpcio_tools-1.75.1-cp311-cp311-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:860fafdb85726029d646c99859ff7bdca5aae61b5ff038c3bd355fc1ec6b2764", size = 2656311, upload-time = "2025-09-26T09:08:17.094Z" },
    { url = "https://files.pythonhosted.org/packages/aa/09/67b9215d39add550e430c9677bd43c9a315da07ab62fa3a5f44f1cf5bb75/grpcio_tools-1.75.1-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:4559547a0cb3d3db1b982eea87d4656036339b400f48127fef932210672fb59e", size = 3105583, upload-time = "2025-09-26T09:08:19.179Z" },
    { url = "https://files.pythonhosted.org/packages/98/d7/d400b90812470f3dc2466964e62fc03592de46b5c824c82ef5303be60167/grpcio_tools-1.75.1-cp311-cp311-musllinux_1_2_i686.whl", hash = "sha256:9af65a310807d7f36a8f7cddea142fe97d6dffba74444f38870272f2e5a3a06b", size = 3654677, upload-time = "2025-09-26T09:08:21.227Z" },
    { url = "https://files.pythonhosted.org/packages/9c/93/edf6de71b4f936b3f09461a3286db1f902c6366c5de06ef19a8c2523034a/grpcio_tools-1.75.1-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:8c1de31aefc0585d2f915a7cd0994d153547495b8d79c44c58048a3ede0b65be", size = 3322147, upload-time = "2025-09-26T09:08:23.08Z" },
    { url = "https://files.pythonhosted.org/packages/80/00/0f8c6204e34070e7d4f344b27e4b1b0320dfdd94574f79738a43504d182e/grpcio_tools-1.75.1-cp311-cp311-win32.whl", hash = "sha256:efaf95fcaa5d3ac1bcfe44ceed9e2512eb95b5c8c476569bdbbe2bee4b59c8a9", size = 993388, upload-time = "2025-09-26T09:08:24.708Z" },
    { url = "https://files.pythonhosted.org/packages/b0/ae/6f738154980f606293988a64ef4bb0ea2bb12029a4529464aac56fe2ab99/grpcio_tools-1.75.1-cp311-cp311-win_amd64.whl", hash = "sha256:7cefe76fc35c825f0148d60d2294a527053d0f5dd6a60352419214a8c53223c9", size = 1157907, upload-time = "2025-09-26T09:08:26.537Z" },
    { url = "https://files.pythonhosted.org/packages/ef/a7/581bb204d19a347303ed5e25b19f7d8c6365a28c242fca013d1d6d78ad7e/grpcio_tools-1.75.1-cp312-cp312-linux_armv7l.whl", hash = "sha256:49b68936cf212052eeafa50b824e17731b78d15016b235d36e0d32199000b14c", size = 2546099, upload-time = "2025-09-26T09:08:28.794Z" },
    { url = "https://files.pythonhosted.org/packages/9f/59/ab65998eba14ff9d292c880f6a276fe7d0571bba3bb4ddf66aca1f8438b5/grpcio_tools-1.75.1-cp312-cp312-macosx_11_0_universal2.whl", hash = "sha256:08cb6e568e58b76a2178ad3b453845ff057131fff00f634d7e15dcd015cd455b", size = 5839838, upload-time = "2025-09-26T09:08:31.038Z" },
    { url = "https://files.pythonhosted.org/packages/7e/65/7027f71069b4c1e8c7b46de8c46c297c9d28ef6ed4ea0161e8c82c75d1d0/grpcio_tools-1.75.1-cp312-cp312-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:168402ad29a249092673079cf46266936ec2fb18d4f854d96e9c5fa5708efa39", size = 2592916, upload-time = "2025-09-26T09:08:33.216Z" },
    { url = "https://files.pythonhosted.org/packages/0f/84/1abfb3c679b78c7fca7524031cf9de4c4c509c441b48fd26291ac16dd1af/grpcio_tools-1.75.1-cp312-cp312-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:bbae11c29fcf450730f021bfc14b12279f2f985e2e493ccc2f133108728261db", size = 2905276, upload-time = "2025-09-26T09:08:35.691Z" },
    { url = "https://files.pythonhosted.org/packages/99/cd/7f9e05f1eddccb61bc0ead1e49eb2222441957b02ed11acfcd2f795b03a8/grpcio_tools-1.75.1-cp312-cp312-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:38c6c7d5d4800f636ee691cd073db1606d1a6a76424ca75c9b709436c9c20439", size = 2656424, upload-time = "2025-09-26T09:08:38.255Z" },
    { url = "https://files.pythonhosted.org/packages/29/1d/8b7852771c2467728341f7b9c3ca4ebc76e4e23485c6a3e6d97a8323ad2a/grpcio_tools-1.75.1-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:626f6a61a8f141dde9a657775854d1c0d99509f9a2762b82aa401a635f6ec73d", size = 3108985, upload-time = "2025-09-26T09:08:40.291Z" },
    { url = "https://files.pythonhosted.org/packages/c2/6a/069da89cdf2e97e4558bfceef5b60bf0ef200c443b465e7691869006dd32/grpcio_tools-1.75.1-cp312-cp312-musllinux_1_2_i686.whl", hash = "sha256:f61a8334ae38d4f98c744a732b89527e5af339d17180e25fff0676060f8709b7", size = 3657940, upload-time = "2025-09-26T09:08:42.437Z" },
    { url = "https://files.pythonhosted.org/packages/c3/e4/ca8dae800c084beb89e2720346f70012d36dfb9df02d8eacd518c06cf4a0/grpcio_tools-1.75.1-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:bd0c3fb40d89a1e24a41974e77c7331e80396ab7cde39bc396a13d6b5e2a750b", size = 3324878, upload-time = "2025-09-26T09:08:45.083Z" },
    { url = "https://files.pythonhosted.org/packages/58/06/cbe923679309bf970923f4a11351ea9e485291b504d7243130fdcfdcb03f/grpcio_tools-1.75.1-cp312-cp312-win32.whl", hash = "sha256:004bc5327593eea48abd03be3188e757c3ca0039079587a6aac24275127cac20", size = 993071, upload-time = "2025-09-26T09:08:46.785Z" },
    { url = "https://files.pythonhosted.org/packages/7c/0c/84d6be007262c5d88a590082f3a1fe62d4b0eeefa10c6cdb3548f3663e80/grpcio_tools-1.75.1-cp312-cp312-win_amd64.whl", hash = "sha256:23952692160b5fe7900653dfdc9858dc78c2c42e15c27e19ee780c8917ba6028", size = 1157506, upload-time = "2025-09-26T09:08:48.844Z" },
    { url = "https://files.pythonhosted.org/packages/47/fa/624bbe1b2ccf4f6044bf3cd314fe2c35f78f702fcc2191dc65519baddca4/grpcio_tools-1.75.1-cp313-cp313-linux_armv7l.whl", hash = "sha256:ca9e116aab0ecf4365fc2980f2e8ae1b22273c3847328b9a8e05cbd14345b397", size = 2545752, upload-time = "2025-09-26T09:08:51.433Z" },
    { url = "https://files.pythonhosted.org/packages/b9/4c/6d884e2337feff0a656e395338019adecc3aa1daeae9d7e8eb54340d4207/grpcio_tools-1.75.1-cp313-cp313-macosx_11_0_universal2.whl", hash = "sha256:9fe87a926b65eb7f41f8738b6d03677cc43185ff77a9d9b201bdb2f673f3fa1e", size = 5838163, upload-time = "2025-09-26T09:08:53.858Z" },
    { url = "https://files.pythonhosted.org/packages/d1/2a/2ba7b6911a754719643ed92ae816a7f989af2be2882b9a9e1f90f4b0e882/grpcio_tools-1.75.1-cp313-cp313-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:45503a6094f91b3fd31c3d9adef26ac514f102086e2a37de797e220a6791ee87", size = 2592148, upload-time = "2025-09-26T09:08:55.86Z" },
    { url = "https://files.pythonhosted.org/packages/88/db/fa613a45c3c7b00f905bd5ad3a93c73194724d0a2dd72adae3be32983343/grpcio_tools-1.75.1-cp313-cp313-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:b01b60b3de67be531a39fd869d7613fa8f178aff38c05e4d8bc2fc530fa58cb5", size = 2905215, upload-time = "2025-09-26T09:08:58.27Z" },
    { url = "https://files.pythonhosted.org/packages/d7/0c/ee4786972bb82f60e4f313bb2227c79c2cd20eb13c94c0263067923cfd12/grpcio_tools-1.75.1-cp313-cp313-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:09e2b9b9488735514777d44c1e4eda813122d2c87aad219f98d5d49b359a8eab", size = 2656251, upload-time = "2025-09-26T09:09:00.249Z" },
    { url = "https://files.pythonhosted.org/packages/77/f1/cc5a50658d705d0b71ff8a4fbbfcc6279d3c95731a2ef7285e13dc40e2fe/grpcio_tools-1.75.1-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:55e60300e62b220fabe6f062fe69f143abaeff3335f79b22b56d86254f3c3c80", size = 3108911, upload-time = "2025-09-26T09:09:02.515Z" },
    { url = "https://files.pythonhosted.org/packages/09/d8/43545f77c4918e778e90bc2c02b3462ac71cee14f29d85cdb69b089538eb/grpcio_tools-1.75.1-cp313-cp313-musllinux_1_2_i686.whl", hash = "sha256:49ce00fcc6facbbf52bf376e55b8e08810cecd03dab0b3a2986d73117c6f6ee4", size = 3657021, upload-time = "2025-09-26T09:09:05.331Z" },
    { url = "https://files.pythonhosted.org/packages/fc/0b/2ae5925374b66bc8df5b828eff1a5f9459349c83dae1773f0aa9858707e6/grpcio_tools-1.75.1-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:71e95479aea868f8c8014d9dc4267f26ee75388a0d8a552e1648cfa0b53d24b4", size = 3324450, upload-time = "2025-09-26T09:09:07.867Z" },
    { url = "https://files.pythonhosted.org/packages/6e/53/9f887bacbecf892ac5b0b282477ca8cfa5b73911b04259f0d88b52e9a055/grpcio_tools-1.75.1-cp313-cp313-win32.whl", hash = "sha256:fff9d2297416eae8861e53154ccf70a19994e5935e6c8f58ebf431f81cbd8d12", size = 992434, upload-time = "2025-09-26T09:09:09.966Z" },
    { url = "https://files.pythonhosted.org/packages/a5/f0/9979d97002edffdc2a88e5f2e0dccea396dd4a6eab34fa2f705fe43eae2f/grpcio_tools-1.75.1-cp313-cp313-win_amd64.whl", hash = "sha256:1849ddd508143eb48791e81d42ddc924c554d1b4900e06775a927573a8d4267f", size = 1157069, upload-time = "2025-09-26T09:09:12.287Z" },
    { url = "https://files.pythonhosted.org/packages/a6/0b/4ff4ead293f2b016668628a240937828444094778c8037d2bbef700e9097/grpcio_tools-1.75.1-cp314-cp314-linux_armv7l.whl", hash = "sha256:f281b594489184b1f9a337cdfed1fc1ddb8428f41c4b4023de81527e90b38e1e", size = 2545868, upload-time = "2025-09-26T09:09:14.716Z" },
    { url = "https://files.pythonhosted.org/packages/0e/78/aa6bf73a18de5357c01ef87eea92150931586b25196fa4df197a37bae11d/grpcio_tools-1.75.1-cp314-cp314-macosx_11_0_universal2.whl", hash = "sha256:becf8332f391abc62bf4eea488b63be063d76a7cf2ef00b2e36c617d9ee9216b", size = 5838010, upload-time = "2025-09-26T09:09:20.415Z" },
    { url = "https://files.pythonhosted.org/packages/99/65/7eaad673bc971af45e079d3b13c20d9ba9842b8788d31953e3234c2e2cee/grpcio_tools-1.75.1-cp314-cp314-manylinux2014_aarch64.manylinux_2_17_aarch64.whl", hash = "sha256:a08330f24e5cd7b39541882a95a8ba04ffb4df79e2984aa0cd01ed26dcdccf49", size = 2593170, upload-time = "2025-09-26T09:09:22.889Z" },
    { url = "https://files.pythonhosted.org/packages/e4/db/57e1e29e9186c7ed223ce8a9b609d3f861c4db015efb643dfe60b403c137/grpcio_tools-1.75.1-cp314-cp314-manylinux2014_i686.manylinux_2_17_i686.whl", hash = "sha256:6bf3742bd8f102630072ed317d1496f31c454cd85ad19d37a68bd85bf9d5f8b9", size = 2905167, upload-time = "2025-09-26T09:09:25.96Z" },
    { url = "https://files.pythonhosted.org/packages/cd/7b/894f891f3cf19812192f8bbf1e0e1c958055676ecf0a5466a350730a006d/grpcio_tools-1.75.1-cp314-cp314-manylinux2014_x86_64.manylinux_2_17_x86_64.whl", hash = "sha256:f26028949474feb380460ce52d9d090d00023940c65236294a66c42ac5850e8b", size = 2656210, upload-time = "2025-09-26T09:09:28.786Z" },
    { url = "https://files.pythonhosted.org/packages/99/76/8e48427da93ef243c09629969c7b5a2c59dceb674b6b623c1f5fbaa5c8c5/grpcio_tools-1.75.1-cp314-cp314-musllinux_1_2_aarch64.whl", hash = "sha256:1bd68fb98bf08f11b6c3210834a14eefe585bad959bdba38e78b4ae3b04ba5bd", size = 3109226, upload-time = "2025-09-26T09:09:31.307Z" },
    { url = "https://files.pythonhosted.org/packages/b3/7e/ecf71c316c2a88c2478b7c6372d0f82d05f07edbf0f31b6da613df99ec7c/grpcio_tools-1.75.1-cp314-cp314-musllinux_1_2_i686.whl", hash = "sha256:f1496e21586193da62c3a73cd16f9c63c5b3efd68ff06dab96dbdfefa90d40bf", size = 3657139, upload-time = "2025-09-26T09:09:35.043Z" },
    { url = "https://files.pythonhosted.org/packages/6f/f3/b2613e81da2085f40a989c0601ec9efc11e8b32fcb71b1234b64a18af830/grpcio_tools-1.75.1-cp314-cp314-musllinux_1_2_x86_64.whl", hash = "sha256:14a78b1e36310cdb3516cdf9ee2726107875e0b247e2439d62fc8dc38cf793c1", size = 3324513, upload-time = "2025-09-26T09:09:37.44Z" },
    { url = "https://files.pythonhosted.org/packages/9a/1f/2df4fa8634542524bc22442ffe045d41905dae62cc5dd14408b80c5ac1b8/grpcio_tools-1.75.1-cp314-cp314-win32.whl", hash = "sha256:0e6f916daf222002fb98f9a6f22de0751959e7e76a24941985cc8e43cea77b50", size = 1015283, upload-time = "2025-09-26T09:09:39.461Z" },
    { url = "https://files.pythonhosted.org/packages/23/4f/f27c973ff50486a70be53a3978b6b0244398ca170a4e19d91988b5295d92/grpcio_tools-1.75.1-cp314-cp314-win_amd64.whl", hash = "sha256:878c3b362264588c45eba57ce088755f8b2b54893d41cc4a68cdeea62996da5c", size = 1189364, upload-time = "2025-09-26T09:09:42.036Z" },
]

[[package]]
name = "h11"
version = "0.16.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/01/ee/02a2c011bdab74c6fb3c75474d40b3052059d95df7e73351460c8588d963/h11-0.16.0.tar.gz", hash = "sha256:4e35b956cf45792e4caa5885e69fba00bdbc6ffafbfa020300e549b208ee5ff1", size = 101250, upload-time = "2025-04-24T03:35:25.427Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/04/4b/29cac41a4d98d144bf5f6d33995617b185d14b22401f75ca86f384e87ff1/h11-0.16.0-py3-none-any.whl", hash = "sha256:63cf8bbe7522de3bf65932fda1d9c2772064ffb3dae62d55932da54b31cb6c86", size = 37515, upload-time = "2025-04-24T03:35:24.344Z" },
]

[[package]]
name = "httpcore"
version = "1.0.9"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "certifi" },
    { name = "h11" },
]
sdist = { url = "https://files.pythonhosted.org/packages/06/94/82699a10bca87a5556c9c59b5963f2d039dbd239f25bc2a63907a05a14cb/httpcore-1.0.9.tar.gz", hash = "sha256:6e34463af53fd2ab5d807f399a9b45ea31c3dfa2276f15a2c3f00afff6e176e8", size = 85484, upload-time = "2025-04-24T22:06:22.219Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/7e/f5/f66802a942d491edb555dd61e3a9961140fd64c90bce1eafd741609d334d/httpcore-1.0.9-py3-none-any.whl", hash = "sha256:2d400746a40668fc9dec9810239072b40b4484b640a8c38fd654a024c7a1bf55", size = 78784, upload-time = "2025-04-24T22:06:20.566Z" },
]

[[package]]
name = "httpx"
version = "0.28.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "anyio" },
    { name = "certifi" },
    { name = "httpcore" },
    { name = "idna" },
]
sdist = { url = "https://files.pythonhosted.org/packages/b1/df/48c586a5fe32a0f01324ee087459e112ebb7224f646c0b5023f5e79e9956/httpx-0.28.1.tar.gz", hash = "sha256:75e98c5f16b0f35b567856f597f06ff2270a374470a5c2392242528e3e3e42fc", size = 141406, upload-time = "2024-12-06T15:37:23.222Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/2a/39/e50c7c3a983047577ee07d2a9e53faf5a69493943ec3f6a384bdc792deb2/httpx-0.28.1-py3-none-any.whl", hash = "sha256:d909fcccc110f8c7faf814ca82a9a4d816bc5a6dbfea25d6591d6985b8ba59ad", size = 73517, upload-time = "2024-12-06T15:37:21.509Z" },
]

[[package]]
name = "idna"
version = "3.11"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/6f/6d/0703ccc57f3a7233505399edb88de3cbd678da106337b9fcde432b65ed60/idna-3.11.tar.gz", hash = "sha256:795dafcc9c04ed0c1fb032c2aa73654d8e8c5023a7df64a53f39190ada629902", size = 194582, upload-time = "2025-10-12T14:55:20.501Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/0e/61/66938bbb5fc52dbdf84594873d5b51fb1f7c7794e9c0f5bd885f30bc507b/idna-3.11-py3-none-any.whl", hash = "sha256:771a87f49d9defaf64091e6e6fe9c18d4833f140bd19464795bc32d966ca37ea", size = 71008, upload-time = "2025-10-12T14:55:18.883Z" },
]

[[package]]
name = "importlib-metadata"
version = "8.7.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "zipp", marker = "python_full_version >= '3.11' and python_full_version < '3.14'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/76/66/650a33bd90f786193e4de4b3ad86ea60b53c89b669a5c7be931fac31cdb0/importlib_metadata-8.7.0.tar.gz", hash = "sha256:d13b81ad223b890aa16c5471f2ac3056cf76c5f10f82d6f9292f0b415f389000", size = 56641, upload-time = "2025-04-27T15:29:01.736Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/20/b0/36bd937216ec521246249be3bf9855081de4c5e06a0c9b4219dbeda50373/importlib_metadata-8.7.0-py3-none-any.whl", hash = "sha256:e5dd1551894c77868a30651cef00984d50e1002d06942a7101d34870c5f02afd", size = 27656, upload-time = "2025-04-27T15:29:00.214Z" },
]

[[package]]
name = "iniconfig"
version = "2.3.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/72/34/14ca021ce8e5dfedc35312d08ba8bf51fdd999c576889fc2c24cb97f4f10/iniconfig-2.3.0.tar.gz", hash = "sha256:c76315c77db068650d49c5b56314774a7804df16fee4402c1f19d6d15d8c4730", size = 20503, upload-time = "2025-10-18T21:55:43.219Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/cb/b1/3846dd7f199d53cb17f49cba7e651e9ce294d8497c8c150530ed11865bb8/iniconfig-2.3.0-py3-none-any.whl", hash = "sha256:f631c04d2c48c52b84d0d0549c99ff3859c98df65b3101406327ecc7d53fbf12", size = 7484, upload-time = "2025-10-18T21:55:41.639Z" },
]

[[package]]
name = "ipykernel"
version = "7.0.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "appnope", marker = "sys_platform == 'darwin'" },
    { name = "comm" },
    { name = "debugpy" },
    { name = "ipython", version = "8.37.0", source = { registry = "https://pypi.org/simple" }, marker = "python_full_version < '3.11'" },
    { name = "ipython", version = "9.6.0", source = { registry = "https://pypi.org/simple" }, marker = "python_full_version >= '3.11'" },
    { name = "jupyter-client" },
    { name = "jupyter-core" },
    { name = "matplotlib-inline" },
    { name = "nest-asyncio" },
    { name = "packaging" },
    { name = "psutil" },
    { name = "pyzmq" },
    { name = "tornado" },
    { name = "traitlets" },
]
sdist = { url = "https://files.pythonhosted.org/packages/a8/4c/9f0024c8457286c6bfd5405a15d650ec5ea36f420ef9bbc58b301f66cfc5/ipykernel-7.0.1.tar.gz", hash = "sha256:2d3fd7cdef22071c2abbad78f142b743228c5d59cd470d034871ae0ac359533c", size = 171460, upload-time = "2025-10-14T16:17:07.325Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/b8/f7/761037905ffdec673533bfa43af8d4c31c859c778dfc3bbb71899875ec18/ipykernel-7.0.1-py3-none-any.whl", hash = "sha256:87182a8305e28954b6721087dec45b171712610111d494c17bb607befa1c4000", size = 118157, upload-time = "2025-10-14T16:17:05.606Z" },
]

[[package]]
name = "ipython"
version = "8.37.0"
source = { registry = "https://pypi.org/simple" }
resolution-markers = [
    "python_full_version < '3.11'",
]
dependencies = [
    { name = "colorama", marker = "python_full_version < '3.11' and sys_platform == 'win32'" },
    { name = "decorator", marker = "python_full_version < '3.11'" },
    { name = "exceptiongroup", marker = "python_full_version < '3.11'" },
    { name = "jedi", marker = "python_full_version < '3.11'" },
    { name = "matplotlib-inline", marker = "python_full_version < '3.11'" },
    { name = "pexpect", marker = "python_full_version < '3.11' and sys_platform != 'emscripten' and sys_platform != 'win32'" },
    { name = "prompt-toolkit", marker = "python_full_version < '3.11'" },
    { name = "pygments", marker = "python_full_version < '3.11'" },
    { name = "stack-data", marker = "python_full_version < '3.11'" },
    { name = "traitlets", marker = "python_full_version < '3.11'" },
    { name = "typing-extensions", marker = "python_full_version < '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/85/31/10ac88f3357fc276dc8a64e8880c82e80e7459326ae1d0a211b40abf6665/ipython-8.37.0.tar.gz", hash = "sha256:ca815841e1a41a1e6b73a0b08f3038af9b2252564d01fc405356d34033012216", size = 5606088, upload-time = "2025-05-31T16:39:09.613Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/91/d0/274fbf7b0b12643cbbc001ce13e6a5b1607ac4929d1b11c72460152c9fc3/ipython-8.37.0-py3-none-any.whl", hash = "sha256:ed87326596b878932dbcb171e3e698845434d8c61b8d8cd474bf663041a9dcf2", size = 831864, upload-time = "2025-05-31T16:39:06.38Z" },
]

[[package]]
name = "ipython"
version = "9.6.0"
source = { registry = "https://pypi.org/simple" }
resolution-markers = [
    "python_full_version >= '3.14'",
    "python_full_version >= '3.11' and python_full_version < '3.14'",
]
dependencies = [
    { name = "colorama", marker = "python_full_version >= '3.11' and sys_platform == 'win32'" },
    { name = "decorator", marker = "python_full_version >= '3.11'" },
    { name = "ipython-pygments-lexers", marker = "python_full_version >= '3.11'" },
    { name = "jedi", marker = "python_full_version >= '3.11'" },
    { name = "matplotlib-inline", marker = "python_full_version >= '3.11'" },
    { name = "pexpect", marker = "python_full_version >= '3.11' and sys_platform != 'emscripten' and sys_platform != 'win32'" },
    { name = "prompt-toolkit", marker = "python_full_version >= '3.11'" },
    { name = "pygments", marker = "python_full_version >= '3.11'" },
    { name = "stack-data", marker = "python_full_version >= '3.11'" },
    { name = "traitlets", marker = "python_full_version >= '3.11'" },
    { name = "typing-extensions", marker = "python_full_version == '3.11.*'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/2a/34/29b18c62e39ee2f7a6a3bba7efd952729d8aadd45ca17efc34453b717665/ipython-9.6.0.tar.gz", hash = "sha256:5603d6d5d356378be5043e69441a072b50a5b33b4503428c77b04cb8ce7bc731", size = 4396932, upload-time = "2025-09-29T10:55:53.948Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/48/c5/d5e07995077e48220269c28a221e168c91123ad5ceee44d548f54a057fc0/ipython-9.6.0-py3-none-any.whl", hash = "sha256:5f77efafc886d2f023442479b8149e7d86547ad0a979e9da9f045d252f648196", size = 616170, upload-time = "2025-09-29T10:55:47.676Z" },
]

[[package]]
name = "ipython-pygments-lexers"
version = "1.1.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "pygments", marker = "python_full_version >= '3.11'" },
]
sdist = { url = "https://files.pythonhosted.org/packages/ef/4c/5dd1d8af08107f88c7f741ead7a40854b8ac24ddf9ae850afbcf698aa552/ipython_pygments_lexers-1.1.1.tar.gz", hash = "sha256:09c0138009e56b6854f9535736f4171d855c8c08a563a0dcd8022f78355c7e81", size = 8393, upload-time = "2025-01-17T11:24:34.505Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/d9/33/1f075bf72b0b747cb3288d011319aaf64083cf2efef8354174e3ed4540e2/ipython_pygments_lexers-1.1.1-py3-none-any.whl", hash = "sha256:a9462224a505ade19a605f71f8fa63c2048833ce50abc86768a0d81d876dc81c", size = 8074, upload-time = "2025-01-17T11:24:33.271Z" },
]

[[package]]
name = "ipywidgets"
version = "8.1.7"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "comm" },
    { name = "ipython", version = "8.37.0", source = { registry = "https://pypi.org/simple" }, marker = "python_full_version < '3.11'" },
    { name = "ipython", version = "9.6.0", source = { registry = "https://pypi.org/simple" }, marker = "python_full_version >= '3.11'" },
    { name = "jupyterlab-widgets" },
    { name = "traitlets" },
    { name = "widgetsnbextension" },
]
sdist = { url = "https://files.pythonhosted.org/packages/3e/48/d3dbac45c2814cb73812f98dd6b38bbcc957a4e7bb31d6ea9c03bf94ed87/ipywidgets-8.1.7.tar.gz", hash = "sha256:15f1ac050b9ccbefd45dccfbb2ef6bed0029d8278682d569d71b8dd96bee0376", size = 116721, upload-time = "2025-05-05T12:42:03.489Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/58/6a/9166369a2f092bd286d24e6307de555d63616e8ddb373ebad2b5635ca4cd/ipywidgets-8.1.7-py3-none-any.whl", hash = "sha256:764f2602d25471c213919b8a1997df04bef869251db4ca8efba1b76b1bd9f7bb", size = 139806, upload-time = "2025-05-05T12:41:56.833Z" },
]

[[package]]
name = "isoduration"
version = "20.11.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "arrow" },
]
sdist = { url = "https://files.pythonhosted.org/packages/7c/1a/3c8edc664e06e6bd06cce40c6b22da5f1429aa4224d0c590f3be21c91ead/isoduration-20.11.0.tar.gz", hash = "sha256:ac2f9015137935279eac671f94f89eb00584f940f5dc49462a0c4ee692ba1bd9", size = 11649, upload-time = "2020-11-01T11:00:00.312Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/7b/55/e5326141505c5d5e34c5e0935d2908a74e4561eca44108fbfb9c13d2911a/isoduration-20.11.0-py3-none-any.whl", hash = "sha256:b2904c2a4228c3d44f409c8ae8e2370eb21a26f7ac2ec5446df141dde3452042", size = 11321, upload-time = "2020-11-01T10:59:58.02Z" },
]

[[package]]
name = "jedi"
version = "0.19.2"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "parso" },
]
sdist = { url = "https://files.pythonhosted.org/packages/72/3a/79a912fbd4d8dd6fbb02bf69afd3bb72cf0c729bb3063c6f4498603db17a/jedi-0.19.2.tar.gz", hash = "sha256:4770dc3de41bde3966b02eb84fbcf557fb33cce26ad23da12c742fb50ecb11f0", size = 1231287, upload-time = "2024-11-11T01:41:42.873Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/c0/5a/9cac0c82afec3d09ccd97c8b6502d48f165f9124db81b4bcb90b4af974ee/jedi-0.19.2-py2.py3-none-any.whl", hash = "sha256:a8ef22bde8490f57fe5c7681a3c83cb58874daf72b4784de3cce5b6ef6edb5b9", size = 1572278, upload-time = "2024-11-11T01:41:40.175Z" },
]

[[package]]
name = "jinja2"
version = "3.1.6"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "markupsafe" },
]
sdist = { url = "https://files.pythonhosted.org/packages/df/bf/f7da0350254c0ed7c72f3e33cef02e048281fec7ecec5f032d4aac52226b/jinja2-3.1.6.tar.gz", hash = "sha256:0137fb05990d35f1275a587e9aee6d56da821fc83491a0fb838183be43f66d6d", size = 245115, upload-time = "2025-03-05T20:05:02.478Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/62/a1/3d680cbfd5f4b8f15abc1d571870c5fc3e594bb582bc3b64ea099db13e56/jinja2-3.1.6-py3-none-any.whl", hash = "sha256:85ece4451f492d0c13c5dd7c13a64681a86afae63a5f347908daf103ce6d2f67", size = 134899, upload-time = "2025-03-05T20:05:00.369Z" },
]

[[package]]
name = "json5"
version = "0.12.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/12/ae/929aee9619e9eba9015207a9d2c1c54db18311da7eb4dcf6d41ad6f0eb67/json5-0.12.1.tar.gz", hash = "sha256:b2743e77b3242f8d03c143dd975a6ec7c52e2f2afe76ed934e53503dd4ad4990", size = 52191, upload-time = "2025-08-12T19:47:42.583Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/85/e2/05328bd2621be49a6fed9e3030b1e51a2d04537d3f816d211b9cc53c5262/json5-0.12.1-py3-none-any.whl", hash = "sha256:d9c9b3bc34a5f54d43c35e11ef7cb87d8bdd098c6ace87117a7b7e83e705c1d5", size = 36119, upload-time = "2025-08-12T19:47:41.131Z" },
]

[[package]]
name = "jsonpatch"
version = "1.33"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "jsonpointer" },
]
sdist = { url = "https://files.pythonhosted.org/packages/42/78/18813351fe5d63acad16aec57f94ec2b70a09e53ca98145589e185423873/jsonpatch-1.33.tar.gz", hash = "sha256:9fcd4009c41e6d12348b4a0ff2563ba56a2923a7dfee731d004e212e1ee5030c", size = 21699, upload-time = "2023-06-26T12:07:29.144Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/73/07/02e16ed01e04a374e644b575638ec7987ae846d25ad97bcc9945a3ee4b0e/jsonpatch-1.33-py2.py3-none-any.whl", hash = "sha256:0ae28c0cd062bbd8b8ecc26d7d164fbbea9652a1a3693f3b956c1eae5145dade", size = 12898, upload-time = "2023-06-16T21:01:28.466Z" },
]

[[package]]
name = "jsonpointer"
version = "3.0.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/6a/0a/eebeb1fa92507ea94016a2a790b93c2ae41a7e18778f85471dc54475ed25/jsonpointer-3.0.0.tar.gz", hash = "sha256:2b2d729f2091522d61c3b31f82e11870f60b68f43fbc705cb76bf4b832af59ef", size = 9114, upload-time = "2024-06-10T19:24:42.462Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/71/92/5e77f98553e9e75130c78900d000368476aed74276eb8ae8796f65f00918/jsonpointer-3.0.0-py2.py3-none-any.whl", hash = "sha256:13e088adc14fca8b6aa8177c044e12701e6ad4b28ff10e65f2267a90109c9942", size = 7595, upload-time = "2024-06-10T19:24:40.698Z" },
]

[[package]]
name = "jsonschema"
version = "4.25.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "attrs" },
    { name = "jsonschema-specifications" },
    { name = "referencing" },
    { name = "rpds-py" },
]
sdist = { url = "https://files.pythonhosted.org/packages/74/69/f7185de793a29082a9f3c7728268ffb31cb5095131a9c139a74078e27336/jsonschema-4.25.1.tar.gz", hash = "sha256:e4a9655ce0da0c0b67a085847e00a3a51449e1157f4f75e9fb5aa545e122eb85", size = 357342, upload-time = "2025-08-18T17:03:50.038Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/bf/9c/8c95d856233c1f82500c2450b8c68576b4cf1c871db3afac5c34ff84e6fd/jsonschema-4.25.1-py3-none-any.whl", hash = "sha256:3fba0169e345c7175110351d456342c364814cfcf3b964ba4587f22915230a63", size = 90040, upload-time = "2025-08-18T17:03:48.373Z" },
]

[package.optional-dependencies]
format-nongpl = [
    { name = "fqdn" },
    { name = "idna" },
    { name = "isoduration" },
    { name = "jsonpointer" },
    { name = "rfc3339-validator" },
    { name = "rfc3986-validator" },
    { name = "rfc3987-syntax" },
    { name = "uri-template" },
    { name = "webcolors" },
]

[[package]]
name = "jsonschema-rs"
version = "0.29.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/b0/b4/33a9b25cad41d1e533c1ab7ff30eaec50628dd1bcb92171b99a2e944d61f/jsonschema_rs-0.29.1.tar.gz", hash = "sha256:a9f896a9e4517630374f175364705836c22f09d5bd5bbb06ec0611332b6702fd", size = 1406679, upload-time = "2025-02-08T21:25:12.639Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/4e/2a/cc53cf158d9ea3629b0c818fdbce9ad2cd8f656f11cafd0e21124487887e/jsonschema_rs-0.29.1-cp310-cp310-macosx_10_12_x86_64.macosx_11_0_arm64.macosx_10_12_universal2.whl", hash = "sha256:7d84c4e1483a103694150b5706d638606443c814662738f14d34ca16948349df", size = 3828955, upload-time = "2025-02-08T21:23:47.957Z" },
    { url = "https://files.pythonhosted.org/packages/83/81/6ff994c6bf223eab69109f367fe8c5551a84f9e17869b6d51682776916fa/jsonschema_rs-0.29.1-cp310-cp310-macosx_10_12_x86_64.whl", hash = "sha256:0e7b72365ae0875c0e99f951ad4cec00c6f5e57b25ed5b8495b8f2c810ad21a6", size = 1968807, upload-time = "2025-02-08T21:23:52.069Z" },
    { url = "https://files.pythonhosted.org/packages/e2/11/28243681ff1337bfe988d5b13acc11bd7f962a16eb5721f4fba86c8c7157/jsonschema_rs-0.29.1-cp310-cp310-manylinux_2_12_i686.manylinux2010_i686.whl", hash = "sha256:7078d3cc635b9832ba282ec4de3af4cdaba4af74691e52aa3ea5f7d1baaa3b76", size = 2066165, upload-time = "2025-02-08T21:23:55.573Z" },
    { url = "https://files.pythonhosted.org/packages/b5/17/263f9dccc3d80b7b43ee564b414155a62f6685eb2e657a3d1ebb68f791af/jsonschema_rs-0.29.1-cp310-cp310-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:30ed43c64e0d732edf32a881f0c1db340c9484f3a28c41f7da96667a49eb0f34", size = 2067619, upload-time = "2025-02-08T21:23:58.528Z" },
    { url = "https://files.pythonhosted.org/packages/3c/a7/00f4fec03168c71206bf9da3ff9d943cb2b44ad95f44255c4ae2efcdd4a0/jsonschema_rs-0.29.1-cp310-cp310-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:24a14806090a5ebf1616a0047eb8049f70f3a830c460e71d5f4a78b300644ec9", size = 2085023, upload-time = "2025-02-08T21:24:00.133Z" },
    { url = "https://files.pythonhosted.org/packages/8f/d9/ae4b6de008b45827e534f90cd68f9f34702bae02f9050101e556683d3c55/jsonschema_rs-0.29.1-cp310-cp310-win32.whl", hash = "sha256:3d739419212e87219e0aa5b9b81eee726e755f606ac63f4795e37efeb9635ed9", size = 1704378, upload-time = "2025-02-08T21:24:02.611Z" },
    { url = "https://files.pythonhosted.org/packages/d1/ca/4b8df88f81f560d712144f1dd2526852fc2d183eb5ae044c524146666d68/jsonschema_rs-0.29.1-cp310-cp310-win_amd64.whl", hash = "sha256:a8fa9007e76cea86877165ebb13ed94648246a185d5eabaf9125e97636bc56e4", size = 1872145, upload-time = "2025-02-08T21:24:05.489Z" },
    { url = "https://files.pythonhosted.org/packages/ad/e2/9c3af8c7d56ff1b6bac88137f60bf02f2814c60d1f658ef06b2ddc2a21b1/jsonschema_rs-0.29.1-cp311-cp311-macosx_10_12_x86_64.macosx_11_0_arm64.macosx_10_12_universal2.whl", hash = "sha256:b4458f1a027ab0c64e91edcb23c48220d60a503e741030bcf260fbbe12979ad2", size = 3828925, upload-time = "2025-02-08T21:24:07.289Z" },
    { url = "https://files.pythonhosted.org/packages/3f/29/f9377e55f10ef173c4cf1c2c88bc30e4a1a4ea1c60659c524903cac85a07/jsonschema_rs-0.29.1-cp311-cp311-macosx_10_12_x86_64.whl", hash = "sha256:faf3d90b5473bf654fd6ffb490bd6fdd2e54f4034f652d1749bee963b3104ce3", size = 1968915, upload-time = "2025-02-08T21:24:09.123Z" },
    { url = "https://files.pythonhosted.org/packages/0f/ae/8c514ebab1d312a2422bece0a1ccca45b82a36131d4cb63e01b4469ac99a/jsonschema_rs-0.29.1-cp311-cp311-manylinux_2_12_i686.manylinux2010_i686.whl", hash = "sha256:e96919483960737ea5cd8d36e0752c63b875459f31ae14b3a6e80df925b74947", size = 2066366, upload-time = "2025-02-08T21:24:10.469Z" },
    { url = "https://files.pythonhosted.org/packages/05/3e/04c6b25ae1b53c8c72eaf35cdda4f84558ca4df011d370b5906a6f56ba7f/jsonschema_rs-0.29.1-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:2e70f1ff7281810327b354ecaeba6cdce7fe498483338207fe7edfae1b21c212", size = 2067599, upload-time = "2025-02-08T21:24:12.006Z" },
    { url = "https://files.pythonhosted.org/packages/1f/78/b9b8934e4db4f43f61e65c5f285432c2d07cb1935ad9df88d5080a4a311b/jsonschema_rs-0.29.1-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:07fef0706a5df7ba5f301a6920b28b0a4013ac06623aed96a6180e95c110b82a", size = 2084926, upload-time = "2025-02-08T21:24:14.544Z" },
    { url = "https://files.pythonhosted.org/packages/5c/ae/676d67d2583cdd50b07b5a0989b501aebf003b12232d14f87fc7fb991f2c/jsonschema_rs-0.29.1-cp311-cp311-win32.whl", hash = "sha256:07524370bdce055d4f106b7fed1afdfc86facd7d004cbb71adeaff3e06861bf6", size = 1704339, upload-time = "2025-02-08T21:24:16.145Z" },
    { url = "https://files.pythonhosted.org/packages/4b/3e/4767dce237d8ea2ff5f684699ef1b9dae5017dc41adaa6f3dc3a85b84608/jsonschema_rs-0.29.1-cp311-cp311-win_amd64.whl", hash = "sha256:36fa23c85333baa8ce5bf0564fb19de3d95b7640c0cab9e3205ddc44a62fdbf0", size = 1872253, upload-time = "2025-02-08T21:24:18.43Z" },
    { url = "https://files.pythonhosted.org/packages/7b/4a/67ea15558ab85e67d1438b2e5da63b8e89b273c457106cbc87f8f4959a3d/jsonschema_rs-0.29.1-cp312-cp312-macosx_10_12_x86_64.macosx_11_0_arm64.macosx_10_12_universal2.whl", hash = "sha256:9fe7529faa6a84d23e31b1f45853631e4d4d991c85f3d50e6d1df857bb52b72d", size = 3825206, upload-time = "2025-02-08T21:24:19.985Z" },
    { url = "https://files.pythonhosted.org/packages/b9/2e/bc75ed65d11ba47200ade9795ebd88eb2e64c2852a36d9be640172563430/jsonschema_rs-0.29.1-cp312-cp312-macosx_10_12_x86_64.whl", hash = "sha256:b5d7e385298f250ed5ce4928fd59fabf2b238f8167f2c73b9414af8143dfd12e", size = 1966302, upload-time = "2025-02-08T21:24:21.673Z" },
    { url = "https://files.pythonhosted.org/packages/95/dd/4a90e96811f897de066c69d95bc0983138056b19cb169f2a99c736e21933/jsonschema_rs-0.29.1-cp312-cp312-manylinux_2_12_i686.manylinux2010_i686.whl", hash = "sha256:64a29be0504731a2e3164f66f609b9999aa66a2df3179ecbfc8ead88e0524388", size = 2062846, upload-time = "2025-02-08T21:24:23.171Z" },
    { url = "https://files.pythonhosted.org/packages/21/91/61834396748a741021716751a786312b8a8319715e6c61421447a07c887c/jsonschema_rs-0.29.1-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:7e91defda5dfa87306543ee9b34d97553d9422c134998c0b64855b381f8b531d", size = 2065564, upload-time = "2025-02-08T21:24:24.574Z" },
    { url = "https://files.pythonhosted.org/packages/f0/2c/920d92e88b9bdb6cb14867a55e5572e7b78bfc8554f9c625caa516aa13dd/jsonschema_rs-0.29.1-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:96f87680a6a1c16000c851d3578534ae3c154da894026c2a09a50f727bd623d4", size = 2083055, upload-time = "2025-02-08T21:24:26.834Z" },
    { url = "https://files.pythonhosted.org/packages/6d/0a/f4c1bea3193992fe4ff9ce330c6a594481caece06b1b67d30b15992bbf54/jsonschema_rs-0.29.1-cp312-cp312-win32.whl", hash = "sha256:bcfc0d52ecca6c1b2fbeede65c1ad1545de633045d42ad0c6699039f28b5fb71", size = 1701065, upload-time = "2025-02-08T21:24:28.282Z" },
    { url = "https://files.pythonhosted.org/packages/5e/89/3f89de071920208c0eb64b827a878d2e587f6a3431b58c02f63c3468b76e/jsonschema_rs-0.29.1-cp312-cp312-win_amd64.whl", hash = "sha256:a414c162d687ee19171e2d8aae821f396d2f84a966fd5c5c757bd47df0954452", size = 1871774, upload-time = "2025-02-08T21:24:30.824Z" },
    { url = "https://files.pythonhosted.org/packages/1b/9b/d642024e8b39753b789598363fd5998eb3053b52755a5df6a021d53741d5/jsonschema_rs-0.29.1-cp313-cp313-macosx_10_12_x86_64.macosx_11_0_arm64.macosx_10_12_universal2.whl", hash = "sha256:0afee5f31a940dec350a33549ec03f2d1eda2da3049a15cd951a266a57ef97ee", size = 3824864, upload-time = "2025-02-08T21:24:32.252Z" },
    { url = "https://files.pythonhosted.org/packages/aa/3d/48a7baa2373b941e89a12e720dae123fd0a663c28c4e82213a29c89a4715/jsonschema_rs-0.29.1-cp313-cp313-macosx_10_12_x86_64.whl", hash = "sha256:c38453a5718bcf2ad1b0163d128814c12829c45f958f9407c69009d8b94a1232", size = 1966084, upload-time = "2025-02-08T21:24:33.8Z" },
    { url = "https://files.pythonhosted.org/packages/1e/e4/f260917a17bb28bb1dec6fa5e869223341fac2c92053aa9bd23c1caaefa0/jsonschema_rs-0.29.1-cp313-cp313-manylinux_2_12_i686.manylinux2010_i686.whl", hash = "sha256:5dc8bdb1067bf4f6d2f80001a636202dc2cea027b8579f1658ce8e736b06557f", size = 2062430, upload-time = "2025-02-08T21:24:35.174Z" },
    { url = "https://files.pythonhosted.org/packages/f5/e7/61353403b76768601d802afa5b7b5902d52c33d1dd0f3159aafa47463634/jsonschema_rs-0.29.1-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:4bcfe23992623a540169d0845ea8678209aa2fe7179941dc7c512efc0c2b6b46", size = 2065443, upload-time = "2025-02-08T21:24:36.778Z" },
    { url = "https://files.pythonhosted.org/packages/40/ed/40b971a09f46a22aa956071ea159413046e9d5fcd280a5910da058acdeb2/jsonschema_rs-0.29.1-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:0f2a526c0deacd588864d3400a0997421dffef6fe1df5cfda4513a453c01ad42", size = 2082606, upload-time = "2025-02-08T21:24:38.388Z" },
    { url = "https://files.pythonhosted.org/packages/bc/59/1c142e1bfb87d57c18fb189149f7aa8edf751725d238d787015278b07600/jsonschema_rs-0.29.1-cp313-cp313-win32.whl", hash = "sha256:68acaefb54f921243552d15cfee3734d222125584243ca438de4444c5654a8a3", size = 1700666, upload-time = "2025-02-08T21:24:40.573Z" },
    { url = "https://files.pythonhosted.org/packages/13/e8/f0ad941286cd350b879dd2b3c848deecd27f0b3fbc0ff44f2809ad59718d/jsonschema_rs-0.29.1-cp313-cp313-win_amd64.whl", hash = "sha256:1c4e5a61ac760a2fc3856a129cc84aa6f8fba7b9bc07b19fe4101050a8ecc33c", size = 1871619, upload-time = "2025-02-08T21:24:42.286Z" },
]

[[package]]
name = "jsonschema-specifications"
version = "2025.9.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "referencing" },
]
sdist = { url = "https://files.pythonhosted.org/packages/19/74/a633ee74eb36c44aa6d1095e7cc5569bebf04342ee146178e2d36600708b/jsonschema_specifications-2025.9.1.tar.gz", hash = "sha256:b540987f239e745613c7a9176f3edb72b832a4ac465cf02712288397832b5e8d", size = 32855, upload-time = "2025-09-08T01:34:59.186Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/41/45/1a4ed80516f02155c51f51e8cedb3c1902296743db0bbc66608a0db2814f/jsonschema_specifications-2025.9.1-py3-none-any.whl", hash = "sha256:98802fee3a11ee76ecaca44429fda8a41bff98b00a0f2838151b113f210cc6fe", size = 18437, upload-time = "2025-09-08T01:34:57.871Z" },
]

[[package]]
name = "jupyter"
version = "1.1.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "ipykernel" },
    { name = "ipywidgets" },
    { name = "jupyter-console" },
    { name = "jupyterlab" },
    { name = "nbconvert" },
    { name = "notebook" },
]
sdist = { url = "https://files.pythonhosted.org/packages/58/f3/af28ea964ab8bc1e472dba2e82627d36d470c51f5cd38c37502eeffaa25e/jupyter-1.1.1.tar.gz", hash = "sha256:d55467bceabdea49d7e3624af7e33d59c37fff53ed3a350e1ac957bed731de7a", size = 5714959, upload-time = "2024-08-30T07:15:48.299Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/38/64/285f20a31679bf547b75602702f7800e74dbabae36ef324f716c02804753/jupyter-1.1.1-py2.py3-none-any.whl", hash = "sha256:7a59533c22af65439b24bbe60373a4e95af8f16ac65a6c00820ad378e3f7cc83", size = 2657, upload-time = "2024-08-30T07:15:47.045Z" },
]

[[package]]
name = "jupyter-client"
version = "8.6.3"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "jupyter-core" },
    { name = "python-dateutil" },
    { name = "pyzmq" },
    { name = "tornado" },
    { name = "traitlets" },
]
sdist = { url = "https://files.pythonhosted.org/packages/71/22/bf9f12fdaeae18019a468b68952a60fe6dbab5d67cd2a103cac7659b41ca/jupyter_client-8.6.3.tar.gz", hash = "sha256:35b3a0947c4a6e9d589eb97d7d4cd5e90f910ee73101611f01283732bd6d9419", size = 342019, upload-time = "2024-09-17T10:44:17.613Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/11/85/b0394e0b6fcccd2c1eeefc230978a6f8cb0c5df1e4cd3e7625735a0d7d1e/jupyter_client-8.6.3-py3-none-any.whl", hash = "sha256:e8a19cc986cc45905ac3362915f410f3af85424b4c0905e94fa5f2cb08e8f23f", size = 106105, upload-time = "2024-09-17T10:44:15.218Z" },
]

[[package]]
name = "jupyter-console"
version = "6.6.3"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "ipykernel" },
    { name = "ipython", version = "8.37.0", source = { registry = "https://pypi.org/simple" }, marker = "python_full_version < '3.11'" },
    { name = "ipython", version = "9.6.0", source = { registry = "https://pypi.org/simple" }, marker = "python_full_version >= '3.11'" },
    { name = "jupyter-client" },
    { name = "jupyter-core" },
    { name = "prompt-toolkit" },
    { name = "pygments" },
    { name = "pyzmq" },
    { name = "traitlets" },
]
sdist = { url = "https://files.pythonhosted.org/packages/bd/2d/e2fd31e2fc41c14e2bcb6c976ab732597e907523f6b2420305f9fc7fdbdb/jupyter_console-6.6.3.tar.gz", hash = "sha256:566a4bf31c87adbfadf22cdf846e3069b59a71ed5da71d6ba4d8aaad14a53539", size = 34363, upload-time = "2023-03-06T14:13:31.02Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/ca/77/71d78d58f15c22db16328a476426f7ac4a60d3a5a7ba3b9627ee2f7903d4/jupyter_console-6.6.3-py3-none-any.whl", hash = "sha256:309d33409fcc92ffdad25f0bcdf9a4a9daa61b6f341177570fdac03de5352485", size = 24510, upload-time = "2023-03-06T14:13:28.229Z" },
]

[[package]]
name = "jupyter-core"
version = "5.9.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "platformdirs" },
    { name = "traitlets" },
]
sdist = { url = "https://files.pythonhosted.org/packages/02/49/9d1284d0dc65e2c757b74c6687b6d319b02f822ad039e5c512df9194d9dd/jupyter_core-5.9.1.tar.gz", hash = "sha256:4d09aaff303b9566c3ce657f580bd089ff5c91f5f89cf7d8846c3cdf465b5508", size = 89814, upload-time = "2025-10-16T19:19:18.444Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/e7/e7/80988e32bf6f73919a113473a604f5a8f09094de312b9d52b79c2df7612b/jupyter_core-5.9.1-py3-none-any.whl", hash = "sha256:ebf87fdc6073d142e114c72c9e29a9d7ca03fad818c5d300ce2adc1fb0743407", size = 29032, upload-time = "2025-10-16T19:19:16.783Z" },
]

[[package]]
name = "jupyter-events"
version = "0.12.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "jsonschema", extra = ["format-nongpl"] },
    { name = "packaging" },
    { name = "python-json-logger" },
    { name = "pyyaml" },
    { name = "referencing" },
    { name = "rfc3339-validator" },
    { name = "rfc3986-validator" },
    { name = "traitlets" },
]
sdist = { url = "https://files.pythonhosted.org/packages/9d/c3/306d090461e4cf3cd91eceaff84bede12a8e52cd821c2d20c9a4fd728385/jupyter_events-0.12.0.tar.gz", hash = "sha256:fc3fce98865f6784c9cd0a56a20644fc6098f21c8c33834a8d9fe383c17e554b", size = 62196, upload-time = "2025-02-03T17:23:41.485Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/e2/48/577993f1f99c552f18a0428731a755e06171f9902fa118c379eb7c04ea22/jupyter_events-0.12.0-py3-none-any.whl", hash = "sha256:6464b2fa5ad10451c3d35fabc75eab39556ae1e2853ad0c0cc31b656731a97fb", size = 19430, upload-time = "2025-02-03T17:23:38.643Z" },
]

[[package]]
name = "jupyter-lsp"
version = "2.3.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "jupyter-server" },
]
sdist = { url = "https://files.pythonhosted.org/packages/eb/5a/9066c9f8e94ee517133cd98dba393459a16cd48bba71a82f16a65415206c/jupyter_lsp-2.3.0.tar.gz", hash = "sha256:458aa59339dc868fb784d73364f17dbce8836e906cd75fd471a325cba02e0245", size = 54823, upload-time = "2025-08-27T17:47:34.671Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/1a/60/1f6cee0c46263de1173894f0fafcb3475ded276c472c14d25e0280c18d6d/jupyter_lsp-2.3.0-py3-none-any.whl", hash = "sha256:e914a3cb2addf48b1c7710914771aaf1819d46b2e5a79b0f917b5478ec93f34f", size = 76687, upload-time = "2025-08-27T17:47:33.15Z" },
]

[[package]]
name = "jupyter-server"
version = "2.17.0"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "anyio" },
    { name = "argon2-cffi" },
    { name = "jinja2" },
    { name = "jupyter-client" },
    { name = "jupyter-core" },
    { name = "jupyter-events" },
    { name = "jupyter-server-terminals" },
    { name = "nbconvert" },
    { name = "nbformat" },
    { name = "overrides", marker = "python_full_version < '3.12'" },
    { name = "packaging" },
    { name = "prometheus-client" },
    { name = "pywinpty", marker = "os_name == 'nt'" },
    { name = "pyzmq" },
    { name = "send2trash" },
    { name = "terminado" },
    { name = "tornado" },
    { name = "traitlets" },
    { name = "websocket-client" },
]
sdist = { url = "https://files.pythonhosted.org/packages/5b/ac/e040ec363d7b6b1f11304cc9f209dac4517ece5d5e01821366b924a64a50/jupyter_server-2.17.0.tar.gz", hash = "sha256:c38ea898566964c888b4772ae1ed58eca84592e88251d2cfc4d171f81f7e99d5", size = 731949, upload-time = "2025-08-21T14:42:54.042Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/92/80/a24767e6ca280f5a49525d987bf3e4d7552bf67c8be07e8ccf20271f8568/jupyter_server-2.17.0-py3-none-any.whl", hash = "sha256:e8cb9c7db4251f51ed307e329b81b72ccf2056ff82d50524debde1ee1870e13f", size = 388221, upload-time = "2025-08-21T14:42:52.034Z" },
]

[[package]]
name = "jupyter-server-terminals"
version = "0.5.3"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "pywinpty", marker = "os_name == 'nt'" },
    { name = "terminado" },
]
sdist = { url = "https://files.pythonhosted.org/packages/fc/d5/562469734f476159e99a55426d697cbf8e7eb5efe89fb0e0b4f83a3d3459/jupyter_server_terminals-0.5.3.tar.gz", hash = "sha256:5ae0295167220e9ace0edcfdb212afd2b01ee8d179fe6f23c899590e9b8a5269", size = 31430, upload-time = "2024-03-12T14:37:03.049Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/07/2d/2b32cdbe8d2a602f697a649798554e4f072115438e92249624e532e8aca6/jupyter_server_terminals-0.5.3-py3-none-any.whl", hash = "sha256:41ee0d7dc0ebf2809c668e0fc726dfaf258fcd3e769568996ca731b6194ae9aa", size = 13656, upload-time = "2024-03-12T14:37:00.708Z" },
]

[[package]]
name = "jupyterlab"
version = "4.4.9"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "async-lru" },
    { name = "httpx" },
    { name = "ipykernel" },
    { name = "jinja2" },
    { name = "jupyter-core" },
    { name = "jupyter-lsp" },
    { name = "jupyter-server" },
    { name = "jupyterlab-server" },
    { name = "notebook-shim" },
    { name = "packaging" },
    { name = "setuptools" },
    { name = "tomli", marker = "python_full_version < '3.11'" },
    { name = "tornado" },
    { name = "traitlets" },
]
sdist = { url = "https://files.pythonhosted.org/packages/45/b2/7dad2d0049a904d17c070226a4f78f81905f93bfe09503722d210ccf9335/jupyterlab-4.4.9.tar.gz", hash = "sha256:ea55aca8269909016d5fde2dc09b97128bc931230183fe7e2920ede5154ad9c2", size = 22966654, upload-time = "2025-09-26T17:28:20.158Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/1f/fd/ac0979ebd1b1975c266c99b96930b0a66609c3f6e5d76979ca6eb3073896/jupyterlab-4.4.9-py3-none-any.whl", hash = "sha256:394c902827350c017430a8370b9f40c03c098773084bc53930145c146d3d2cb2", size = 12292552, upload-time = "2025-09-26T17:28:15.663Z" },
]

[[package]]
name = "jupyterlab-pygments"
version = "0.3.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/90/51/9187be60d989df97f5f0aba133fa54e7300f17616e065d1ada7d7646b6d6/jupyterlab_pygments-0.3.0.tar.gz", hash = "sha256:721aca4d9029252b11cfa9d185e5b5af4d54772bb8072f9b7036f4170054d35d", size = 512900, upload-time = "2023-11-23T09:26:37.44Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/b1/dd/ead9d8ea85bf202d90cc513b533f9c363121c7792674f78e0d8a854b63b4/jupyterlab_pygments-0.3.0-py3-none-any.whl", hash = "sha256:841a89020971da1d8693f1a99997aefc5dc424bb1b251fd6322462a1b8842780", size = 15884, upload-time = "2023-11-23T09:26:34.325Z" },
]

[[package]]
name = "jupyterlab-server"
version = "2.27.3"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "babel" },
    { name = "jinja2" },
    { name = "json5" },
    { name = "jsonschema" },
    { name = "jupyter-server" },
    { name = "packaging" },
    { name = "requests" },
]
sdist = { url = "https://files.pythonhosted.org/packag
```
> [truncated]

---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/config.py
```py
import asyncio
import sys
from typing import Any

from langchain_core.runnables import RunnableConfig
from langchain_core.runnables.config import var_child_runnable_config
from langgraph.store.base import BaseStore

from langgraph._internal._constants import CONF, CONFIG_KEY_RUNTIME
from langgraph.types import StreamWriter


def _no_op_stream_writer(c: Any) -> None:
    pass


def get_config() -> RunnableConfig:
    if sys.version_info < (3, 11):
        try:
            if asyncio.current_task():
                raise RuntimeError(
                    "Python 3.11 or later required to use this in an async context"
                )
        except RuntimeError:
            pass
    if var_config := var_child_runnable_config.get():
        return var_config
    else:
        raise RuntimeError("Called get_config outside of a runnable context")


def get_store() -> BaseStore:
    """Access LangGraph store from inside a graph node or entrypoint task at runtime.

    Can be called from inside any [StateGraph][langgraph.graph.StateGraph] node or
    functional API [task][langgraph.func.task], as long as the StateGraph or the [entrypoint][langgraph.func.entrypoint]
    was initialized with a store, e.g.:

    ```python
    # with StateGraph
    graph = (
        StateGraph(...)
        ...
        .compile(store=store)
    )

    # or with entrypoint
    @entrypoint(store=store)
    def workflow(inputs):
        ...
    ```

    !!! warning "Async with Python < 3.11"

        If you are using Python < 3.11 and are running LangGraph asynchronously,
        `get_store()` won't work since it uses [contextvar](https://docs.python.org/3/library/contextvars.html) propagation (only available in [Python >= 3.11](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task)).


    Example: Using with StateGraph
        ```python
        from typing_extensions import TypedDict
        from langgraph.graph import StateGraph, START
        from langgraph.store.memory import InMemoryStore
        from langgraph.config import get_store

        store = InMemoryStore()
        store.put(("values",), "foo", {"bar": 2})


        class State(TypedDict):
            foo: int


        def my_node(state: State):
            my_store = get_store()
            stored_value = my_store.get(("values",), "foo").value["bar"]
            return {"foo": stored_value + 1}


        graph = (
            StateGraph(State)
            .add_node(my_node)
            .add_edge(START, "my_node")
            .compile(store=store)
        )

        graph.invoke({"foo": 1})
        ```

        ```pycon
        {"foo": 3}
        ```

    Example: Using with functional API
        ```python
        from langgraph.func import entrypoint, task
        from langgraph.store.memory import InMemoryStore
        from langgraph.config import get_store

        store = InMemoryStore()
        store.put(("values",), "foo", {"bar": 2})


        @task
        def my_task(value: int):
            my_store = get_store()
            stored_value = my_store.get(("values",), "foo").value["bar"]
            return stored_value + 1


        @entrypoint(store=store)
        def workflow(value: int):
            return my_task(value).result()


        workflow.invoke(1)
        ```

        ```pycon
        3
        ```
    """
    return get_config()[CONF][CONFIG_KEY_RUNTIME].store


def get_stream_writer() -> StreamWriter:
    """Access LangGraph [StreamWriter][langgraph.types.StreamWriter] from inside a graph node or entrypoint task at runtime.

    Can be called from inside any [StateGraph][langgraph.graph.StateGraph] node or
    functional API [task][langgraph.func.task].

    !!! warning "Async with Python < 3.11"

        If you are using Python < 3.11 and are running LangGraph asynchronously,
        `get_stream_writer()` won't work since it uses [contextvar](https://docs.python.org/3/library/contextvars.html) propagation (only available in [Python >= 3.11](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task)).

    Example: Using with StateGraph
        ```python
        from typing_extensions import TypedDict
        from langgraph.graph import StateGraph, START
        from langgraph.config import get_stream_writer


        class State(TypedDict):
            foo: int


        def my_node(state: State):
            my_stream_writer = get_stream_writer()
            my_stream_writer({"custom_data": "Hello!"})
            return {"foo": state["foo"] + 1}


        graph = (
            StateGraph(State)
            .add_node(my_node)
            .add_edge(START, "my_node")
            .compile(store=store)
        )

        for chunk in graph.stream({"foo": 1}, stream_mode="custom"):
            print(chunk)
        ```

        ```pycon
        {"custom_data": "Hello!"}
        ```

    Example: Using with functional API
        ```python
        from langgraph.func import entrypoint, task
        from langgraph.config import get_stream_writer


        @task
        def my_task(value: int):
            my_stream_writer = get_stream_writer()
            my_stream_writer({"custom_data": "Hello!"})
            return value + 1


        @entrypoint(store=store)
        def workflow(value: int):
            return my_task(value).result()


        for chunk in workflow.stream(1, stream_mode="custom"):
            print(chunk)
        ```

        ```pycon
        {"custom_data": "Hello!"}
        ```
    """
    runtime = get_config()[CONF][CONFIG_KEY_RUNTIME]
    return runtime.stream_writer

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/constants.py
```py
import sys
from typing import Any
from warnings import warn

from langgraph._internal._constants import (
    CONF,
    CONFIG_KEY_CHECKPOINTER,
    TASKS,
)
from langgraph.warnings import LangGraphDeprecatedSinceV10

__all__ = (
    "TAG_NOSTREAM",
    "TAG_HIDDEN",
    "START",
    "END",
    # retained for backwards compatibility (mostly langgraph-api), should be removed in v2 (or earlier)
    "CONF",
    "TASKS",
    "CONFIG_KEY_CHECKPOINTER",
)

# --- Public constants ---
TAG_NOSTREAM = sys.intern("nostream")
"""Tag to disable streaming for a chat model."""
TAG_HIDDEN = sys.intern("langsmith:hidden")
"""Tag to hide a node/edge from certain tracing/streaming environments."""
END = sys.intern("__end__")
"""The last (maybe virtual) node in graph-style Pregel."""
START = sys.intern("__start__")
"""The first (maybe virtual) node in graph-style Pregel."""


def __getattr__(name: str) -> Any:
    if name in ["Send", "Interrupt"]:
        warn(
            f"Importing {name} from langgraph.constants is deprecated. "
            f"Please use 'from langgraph.types import {name}' instead.",
            LangGraphDeprecatedSinceV10,
            stacklevel=2,
        )

        from importlib import import_module

        module = import_module("langgraph.types")
        return getattr(module, name)

    try:
        from importlib import import_module

        private_constants = import_module("langgraph._internal._constants")
        attr = getattr(private_constants, name)
        warn(
            f"Importing {name} from langgraph.constants is deprecated. "
            f"This constant is now private and should not be used directly. "
            "Please let the LangGraph team know if you need this value.",
            LangGraphDeprecatedSinceV10,
            stacklevel=2,
        )
        return attr
    except AttributeError:
        pass

    raise AttributeError(f"module has no attribute '{name}'")

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/errors.py
```py
from __future__ import annotations

from collections.abc import Sequence
from enum import Enum
from typing import Any
from warnings import warn

# EmptyChannelError is re-exported from langgraph.channels.base
from langgraph.checkpoint.base import EmptyChannelError  # noqa: F401
from typing_extensions import deprecated

from langgraph.types import Command, Interrupt
from langgraph.warnings import LangGraphDeprecatedSinceV10

__all__ = (
    "EmptyChannelError",
    "ErrorCode",
    "GraphRecursionError",
    "InvalidUpdateError",
    "GraphBubbleUp",
    "GraphInterrupt",
    "NodeInterrupt",
    "ParentCommand",
    "EmptyInputError",
    "TaskNotFound",
)


class ErrorCode(Enum):
    GRAPH_RECURSION_LIMIT = "GRAPH_RECURSION_LIMIT"
    INVALID_CONCURRENT_GRAPH_UPDATE = "INVALID_CONCURRENT_GRAPH_UPDATE"
    INVALID_GRAPH_NODE_RETURN_VALUE = "INVALID_GRAPH_NODE_RETURN_VALUE"
    MULTIPLE_SUBGRAPHS = "MULTIPLE_SUBGRAPHS"
    INVALID_CHAT_HISTORY = "INVALID_CHAT_HISTORY"


def create_error_message(*, message: str, error_code: ErrorCode) -> str:
    return (
        f"{message}\n"
        "For troubleshooting, visit: https://docs.langchain.com/oss/python/langgraph/"
        f"errors/{error_code.value}"
    )


class GraphRecursionError(RecursionError):
    """Raised when the graph has exhausted the maximum number of steps.

    This prevents infinite loops. To increase the maximum number of steps,
    run your graph with a config specifying a higher `recursion_limit`.

    Troubleshooting guides:

    - [`GRAPH_RECURSION_LIMIT`](https://docs.langchain.com/oss/python/langgraph/GRAPH_RECURSION_LIMIT)

    Examples:

        graph = builder.compile()
        graph.invoke(
            {"messages": [("user", "Hello, world!")]},
            # The config is the second positional argument
            {"recursion_limit": 1000},
        )
    """

    pass


class InvalidUpdateError(Exception):
    """Raised when attempting to update a channel with an invalid set of updates.

    Troubleshooting guides:

    - [`INVALID_CONCURRENT_GRAPH_UPDATE`](https://docs.langchain.com/oss/python/langgraph/INVALID_CONCURRENT_GRAPH_UPDATE)
    - [`INVALID_GRAPH_NODE_RETURN_VALUE`](https://docs.langchain.com/oss/python/langgraph/INVALID_GRAPH_NODE_RETURN_VALUE)
    """

    pass


class GraphBubbleUp(Exception):
    pass


class GraphInterrupt(GraphBubbleUp):
    """Raised when a subgraph is interrupted, suppressed by the root graph.
    Never raised directly, or surfaced to the user."""

    def __init__(self, interrupts: Sequence[Interrupt] = ()) -> None:
        super().__init__(interrupts)


@deprecated(
    "NodeInterrupt is deprecated. Please use [`interrupt`][langgraph.types.interrupt] instead.",
    category=None,
)
class NodeInterrupt(GraphInterrupt):
    """Raised by a node to interrupt execution."""

    def __init__(self, value: Any, id: str | None = None) -> None:
        warn(
            "NodeInterrupt is deprecated. Please use `langgraph.types.interrupt` instead.",
            LangGraphDeprecatedSinceV10,
            stacklevel=2,
        )
        if id is None:
            super().__init__([Interrupt(value=value)])
        else:
            super().__init__([Interrupt(value=value, id=id)])


class ParentCommand(GraphBubbleUp):
    args: tuple[Command]

    def __init__(self, command: Command) -> None:
        super().__init__(command)


class EmptyInputError(Exception):
    """Raised when graph receives an empty input."""

    pass


class TaskNotFound(Exception):
    """Raised when the executor is unable to find a task (for distributed mode)."""

    pass

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/py.typed
```typed

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/runtime.py
```py
from __future__ import annotations

from dataclasses import dataclass, field, replace
from typing import Any, Generic, cast

from langgraph.store.base import BaseStore
from typing_extensions import TypedDict, Unpack

from langgraph._internal._constants import CONF, CONFIG_KEY_RUNTIME
from langgraph.config import get_config
from langgraph.types import _DC_KWARGS, StreamWriter
from langgraph.typing import ContextT

__all__ = ("Runtime", "get_runtime")


def _no_op_stream_writer(_: Any) -> None: ...


class _RuntimeOverrides(TypedDict, Generic[ContextT], total=False):
    context: ContextT
    store: BaseStore | None
    stream_writer: StreamWriter
    previous: Any


@dataclass(**_DC_KWARGS)
class Runtime(Generic[ContextT]):
    """Convenience class that bundles run-scoped context and other runtime utilities.

    !!! version-added "Added in version v0.6.0"

    Example:

    ```python
    from typing import TypedDict
    from langgraph.graph import StateGraph
    from dataclasses import dataclass
    from langgraph.runtime import Runtime
    from langgraph.store.memory import InMemoryStore


    @dataclass
    class Context:  # (1)!
        user_id: str


    class State(TypedDict, total=False):
        response: str


    store = InMemoryStore()  # (2)!
    store.put(("users",), "user_123", {"name": "Alice"})


    def personalized_greeting(state: State, runtime: Runtime[Context]) -> State:
        '''Generate personalized greeting using runtime context and store.'''
        user_id = runtime.context.user_id  # (3)!
        name = "unknown_user"
        if runtime.store:
            if memory := runtime.store.get(("users",), user_id):
                name = memory.value["name"]

        response = f"Hello {name}! Nice to see you again."
        return {"response": response}


    graph = (
        StateGraph(state_schema=State, context_schema=Context)
        .add_node("personalized_greeting", personalized_greeting)
        .set_entry_point("personalized_greeting")
        .set_finish_point("personalized_greeting")
        .compile(store=store)
    )

    result = graph.invoke({}, context=Context(user_id="user_123"))
    print(result)
    # > {'response': 'Hello Alice! Nice to see you again.'}
    ```

    1. Define a schema for the runtime context.
    2. Create a store to persist memories and other information.
    3. Use the runtime context to access the `user_id`.
    """

    context: ContextT = field(default=None)  # type: ignore[assignment]
    """Static context for the graph run, like `user_id`, `db_conn`, etc.
    
    Can also be thought of as 'run dependencies'."""

    store: BaseStore | None = field(default=None)
    """Store for the graph run, enabling persistence and memory."""

    stream_writer: StreamWriter = field(default=_no_op_stream_writer)
    """Function that writes to the custom stream."""

    previous: Any = field(default=None)
    """The previous return value for the given thread.
    
    Only available with the functional API when a checkpointer is provided.
    """

    def merge(self, other: Runtime[ContextT]) -> Runtime[ContextT]:
        """Merge two runtimes together.

        If a value is not provided in the other runtime, the value from the current runtime is used.
        """
        return Runtime(
            context=other.context or self.context,
            store=other.store or self.store,
            stream_writer=other.stream_writer
            if other.stream_writer is not _no_op_stream_writer
            else self.stream_writer,
            previous=self.previous if other.previous is None else other.previous,
        )

    def override(
        self, **overrides: Unpack[_RuntimeOverrides[ContextT]]
    ) -> Runtime[ContextT]:
