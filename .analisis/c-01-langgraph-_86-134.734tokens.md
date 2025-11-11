"""

import logging
import re
from typing import Optional

from _scripts.link_map import SCOPE_LINK_MAPS

logger = logging.getLogger(__name__)


def _transform_link(
    link_name: str,
    scope: str,
    file_path: str,
    line_number: int,
    custom_title: Optional[str] = None,
) -> Optional[str]:
    """Transform a cross-reference link based on the current scope.

    Args:
        link_name: The name of the link to transform (e.g., "StateGraph").
        scope: The current scope context ("global", "python", "js", etc.).
        file_path: The file path for error reporting.
        line_number: The line number for error reporting.
        custom_title: Optional custom title for the link. If `None`, uses link_name.

    Returns:
        A formatted markdown link if the link is found in the scope mapping,
        None otherwise.

    Example:
        >>> _transform_link("StateGraph", "python", "file.md", 5)
        "[StateGraph](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.StateGraph)"

        >>> _transform_link("StateGraph", "python", "file.md", 5, "Custom Title")
        "[Custom Title](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.StateGraph)"

        >>> _transform_link("unknown-link", "python", "file.md", 5)
        None
    """
    if scope == "global":
        # Special scope that is composed of both Python and JS links
        # For now, we will substitute in the python scope!
        # But we need to add support for handling both scopes.
        scope = "python"
        logger.error(
            "Encountered unhandled 'global' scope. Defaulting to 'python'."
            "In file: %s, line %d, link_name: %s",
            file_path,
            line_number,
            link_name,
        )
    link_map = SCOPE_LINK_MAPS.get(scope, {})
    url = link_map.get(link_name)

    if url:
        title = custom_title if custom_title is not None else link_name
        return f"[{title}]({url})"
    else:
        # Log error with file location information
        logger.info(
            # Using %s
            "Link '%s' not found in scope '%s'. "
            "In file: %s, line %d. Available links in scope: %s",
            link_name,
            scope,
            file_path,
            line_number,
            list(link_map.keys() if link_map else []),
        )
        return None


CONDITIONAL_FENCE_PATTERN = re.compile(
    r"""
    ^                       # Start of line
    (?P<indent>[ \t]*)      # Optional indentation (spaces or tabs)
    :::                     # Literal fence marker
    (?P<language>\w+)?      # Optional language identifier (named group: language)
    \s*                     # Optional trailing whitespace
    $                       # End of line
    """,
    re.VERBOSE,
)
CROSS_REFERENCE_PATTERN = re.compile(
    r"""
    (?:                     # Non-capturing group for two possible formats:
        @\[                 # @ symbol followed by opening bracket for title
        (?P<title>[^\]]+)   # Custom title - one or more non-bracket characters
        \]                  # Closing bracket for title
        \[                  # Opening bracket for link name
        (?P<link_name_with_title>[^\]]+)  # Link name - one or more non-bracket characters
        \]                  # Closing bracket for link name
        |                   # OR
        @\[                 # @ symbol followed by opening bracket
        (?P<link_name>[^\]]+)   # Link name - one or more non-bracket characters
        \]                  # Closing bracket
    )
    """,
    re.VERBOSE,
)


def _replace_autolinks(
    markdown: str, file_path: str, *, default_scope: str = "python"
) -> str:
    """Preprocess markdown lines to handle @[links] with conditional fence scopes.

    This function processes markdown content to transform @[link_name] references
    based on the current conditional fence scope. Conditional fences use the
    syntax :::language to define scope boundaries.

    Args:
        markdown: The markdown content to process.
        file_path: The file path for error reporting.
        default_scope: The default scope to use if no scope is matched.

    Returns:
        Processed markdown content with @[references] transformed to proper
        markdown links or left unchanged if not found.

    Example:
        Input:
            "@[StateGraph]\\n:::python\\n@[Command]\\n:::\\n"
        Output:
            "[StateGraph](url)\\n:::python\\n[Command](url)\\n:::\\n"
    """
    # Track the current scope context
    current_scope = default_scope
    lines = markdown.splitlines(keepends=True)
    processed_lines = []

    for line_number, line in enumerate(lines, 1):
        line_stripped = line.strip()

        # Check if this line defines a new conditional fence scope
        fence_match = CONDITIONAL_FENCE_PATTERN.match(line_stripped)
        if fence_match:
            language = fence_match.group("language")
            # Set scope to the specified language, or reset to global if no language
            current_scope = language.lower() if language else default_scope
            processed_lines.append(line)
            continue

        # Transform all @[link_name] references in this line based on current scope
        def replace_cross_reference(match: re.Match[str]) -> str:
            """Replace a single @[link_name] with the scoped equivalent."""
            # Check if this is the @[title][ref] format or @[ref] format
            title = match.group("title")
            if title is not None:
                # This is @[title][ref] format
                link_name = match.group("link_name_with_title")
                custom_title = title
            else:
                # This is @[ref] format
                link_name = match.group("link_name")
                custom_title = None

            transformed = _transform_link(
                link_name, current_scope, file_path, line_number, custom_title
            )
            return transformed if transformed is not None else match.group(0)

        transformed_line = CROSS_REFERENCE_PATTERN.sub(replace_cross_reference, line)
        processed_lines.append(transformed_line)

    return "".join(processed_lines)

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/link_map.py
```py
"""Link mapping for cross-reference resolution across different scopes.

This module provides link mappings for different language/framework scopes
to resolve @[link_name] references to actual URLs.
"""

# Python-specific link mappings
PYTHON_LINK_MAP = {
    "StateGraph": "reference/graphs/#langgraph.graph.StateGraph",
    "add_conditional_edges": "reference/graphs/#langgraph.graph.state.StateGraph.add_conditional_edges",
    "add_edge": "reference/graphs/#langgraph.graph.state.StateGraph.add_edge",
    "add_node": "reference/graphs/#langgraph.graph.state.StateGraph.add_node",
    "add_messages": "reference/graphs/#langgraph.graph.message.add_messages",
    "ToolNode": "reference/agents/#langgraph.prebuilt.tool_node.ToolNode",
    "CompiledStateGraph.astream": "reference/graphs/#langgraph.graph.state.CompiledStateGraph.astream",
    "Pregel.astream": "reference/pregel/#langgraph.pregel.Pregel.astream",
    "AsyncPostgresSaver": "reference/checkpoints/#langgraph.checkpoint.postgres.aio.AsyncPostgresSaver",
    "AsyncSqliteSaver": "reference/checkpoints/#langgraph.checkpoint.sqlite.aio.AsyncSqliteSaver",
    "BaseCheckpointSaver": "reference/checkpoints/#langgraph.checkpoint.base.BaseCheckpointSaver",
    "BaseStore": "reference/store/#langgraph.store.base.BaseStore",
    "BaseStore.put": "reference/store/#langgraph.store.base.BaseStore.put",
    "BinaryOperatorAggregate": "reference/pregel/#langgraph.pregel.Pregel--advanced-channels-context-and-binaryoperatoraggregate",
    "CipherProtocol": "reference/checkpoints/#langgraph.checkpoint.serde.base.CipherProtocol",
    "client.runs.stream": "cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.RunsClient.stream",
    "client.runs.wait": "cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.RunsClient.wait",
    "client.threads.get_history": "cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.ThreadsClient.get_history",
    "client.threads.update_state": "cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.ThreadsClient.update_state",
    "Command": "reference/types/#langgraph.types.Command",
    "CompiledStateGraph": "reference/graphs/#langgraph.graph.state.CompiledStateGraph",
    "create_react_agent": "reference/prebuilt/#langgraph.prebuilt.chat_agent_executor.create_react_agent",
    "create_supervisor": "reference/supervisor/#langgraph_supervisor.supervisor.create_supervisor",
    "EncryptedSerializer": "reference/checkpoints/#langgraph.checkpoint.serde.encrypted.EncryptedSerializer",
    "entrypoint.final": "reference/func/#langgraph.func.entrypoint.final",
    "entrypoint": "reference/func/#langgraph.func.entrypoint",
    "from_pycryptodome_aes": "reference/checkpoints/#langgraph.checkpoint.serde.encrypted.EncryptedSerializer.from_pycryptodome_aes",
    "get_state_history": "reference/graphs/#langgraph.graph.state.CompiledStateGraph.get_state_history",
    "get_stream_writer": "reference/config/#langgraph.config.get_stream_writer",
    "HumanInterrupt": "reference/prebuilt/#langgraph.prebuilt.interrupt.HumanInterrupt",
    "InjectedState": "reference/agents/#langgraph.prebuilt.tool_node.InjectedState",
    "InMemorySaver": "reference/checkpoints/#langgraph.checkpoint.memory.InMemorySaver",
    "interrupt": "reference/types/#langgraph.types.Interrupt",
    "CompiledStateGraph.invoke": "reference/graphs/#langgraph.graph.state.CompiledStateGraph.invoke",
    "JsonPlusSerializer": "reference/checkpoints/#langgraph.checkpoint.serde.jsonplus.JsonPlusSerializer",
    "langgraph.json": "cloud/reference/cli/#configuration-file",
    "LastValue": "reference/channels/#langgraph.channels.LastValue",
    "PostgresSaver": "reference/checkpoints/#langgraph.checkpoint.postgres.PostgresSaver",
    "Pregel": "reference/pregel/",
    "Pregel.stream": "reference/pregel/#langgraph.pregel.Pregel.stream",
    "pre_model_hook": "reference/prebuilt/#langgraph.prebuilt.chat_agent_executor.create_react_agent",
    "protocol": "reference/checkpoints/#langgraph.checkpoint.serde.base.SerializerProtocol",
    "Send": "reference/types/#langgraph.types.Send",
    "SerializerProtocol": "reference/checkpoints/#langgraph.checkpoint.serde.base.SerializerProtocol",
    "SqliteSaver": "reference/checkpoints/#langgraph.checkpoint.sqlite.SqliteSaver",
    "START": "reference/constants/#langgraph.constants.START",
    "CompiledStateGraph.stream": "reference/graphs/#langgraph.graph.state.CompiledStateGraph.stream",
    "task": "reference/func/#langgraph.func.task",
    "Topic": "reference/channels/#langgraph.channels.Topic",
    "update_state": "reference/graphs/#langgraph.graph.state.CompiledStateGraph.update_state",
}

# JavaScript-specific link mappings
JS_LINK_MAP = {
    "Auth": "reference/classes/sdk_auth.Auth.html",
    "StateGraph": "reference/classes/langgraph.StateGraph.html",
    "add_conditional_edges": "/reference/classes/langgraph.StateGraph.html#addConditionalEdges",
    "add_edge": "reference/classes/langgraph.StateGraph.html#addEdge",
    "add_node": "reference/classes/langgraph.StateGraph.html#addNode",
    "add_messages": "reference/modules/langgraph.html#addMessages",
    "ToolNode": "reference/classes/langgraph_prebuilt.ToolNode.html",
    "BaseCheckpointSaver": "reference/classes/checkpoint.BaseCheckpointSaver.html",
    "BaseStore": "reference/classes/checkpoint.BaseStore.html",
    "BaseStore.put": "reference/classes/checkpoint.BaseStore.html#put",
    "BinaryOperatorAggregate": "reference/classes/langgraph.BinaryOperatorAggregate.html",
    "client.runs.stream": "reference/classes/sdk_client.RunsClient.html#stream",
    "client.runs.wait": "reference/classes/sdk_client.RunsClient.html#wait",
    "client.threads.get_history": "reference/classes/sdk_client.ThreadsClient.html#getHistory",
    "client.threads.update_state": "reference/classes/sdk_client.ThreadsClient.html#updateState",
    "Command": "reference/classes/langgraph.Command.html",
    "CompiledStateGraph": "reference/classes/langgraph.CompiledStateGraph.html",
    "create_react_agent": "reference/functions/langgraph_prebuilt.createReactAgent.html",
    "create_supervisor": "reference/functions/langgraph_supervisor.createSupervisor.html",
    "entrypoint.final": "reference/functions/langgraph.entrypoint.html#final",
    "entrypoint": "reference/functions/langgraph.entrypoint.html",
    "getContextVariable": "https://v03.api.js.langchain.com/functions/_langchain_core.context.getContextVariable.html",
    "get_state_history": "reference/classes/langgraph.CompiledStateGraph.html#getStateHistory",
    "HumanInterrupt": "reference/interfaces/langgraph_prebuilt.HumanInterrupt.html",
    "interrupt": "reference/functions/langgraph.interrupt-2.html",
    "CompiledStateGraph.invoke": "reference/classes/langgraph.CompiledStateGraph.html#invoke",
    "langgraph.json": "cloud/reference/cli/#configuration-file",
    "MemorySaver": "reference/classes/checkpoint.MemorySaver.html",
    "messagesStateReducer": "reference/functions/langgraph.messagesStateReducer.html",
    "PostgresSaver": "reference/classes/checkpoint_postgres.PostgresSaver.html",
    "Pregel": "reference/classes/langgraph.Pregel.html",
    "Pregel.stream": "reference/classes/langgraph.Pregel.html#stream",
    "pre_model_hook": "reference/functions/langgraph_prebuilt.createReactAgent.html",
    "protocol": "reference/interfaces/checkpoint.SerializerProtocol.html",
    "Send": "reference/classes/langgraph.Send.html",
    "SerializerProtocol": "reference/interfaces/checkpoint.SerializerProtocol.html",
    "SqliteSaver": "reference/classes/checkpoint_sqlite.SqliteSaver.html",
    "START": "reference/variables/langgraph.START.html",
    "CompiledStateGraph.stream": "reference/classes/langgraph.CompiledStateGraph.html#stream",
    "task": "reference/functions/langgraph.task.html",
    ## TODO (hntrl): export Topic from langgraphjs
    # "Topic": "reference/classes/langgraph_channels.Topic.html",
    "update_state": "reference/classes/langgraph.CompiledStateGraph.html#updateState",
}

# TODO: Allow updating these to localhost for local development
PY_REFERENCE_HOST = "https://langchain-ai.github.io/langgraph/"
JS_REFERENCE_HOST = "https://langchain-ai.github.io/langgraphjs/"

for key, value in PYTHON_LINK_MAP.items():
    # Ensure the link is absolute
    if not value.startswith("http"):
        PYTHON_LINK_MAP[key] = f"{PY_REFERENCE_HOST}{value}"

for key, value in JS_LINK_MAP.items():
    # Ensure the link is absolute
    if not value.startswith("http"):
        JS_LINK_MAP[key] = f"{JS_REFERENCE_HOST}{value}"

# Global scope is assembled from the Python and JS mappings
# Combined mapping by scope
SCOPE_LINK_MAPS = {
    "python": PYTHON_LINK_MAP,
    "js": JS_LINK_MAP,
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/notebook_convert.py
```py
"""Convert Jupyter notebooks to markdown with custom processing."""

import ast
import os
import re
from typing import Literal

import nbformat
from nbconvert.exporters import MarkdownExporter
from nbconvert.preprocessors import Preprocessor


def _uses_input(source: str) -> bool:
    """Parse the source code to determine if it uses the input() function."""
    try:
        tree = ast.parse(source)
    except SyntaxError:
        # If there's a syntax error, assume input() might be present to be safe.
        return False

    for node in ast.walk(tree):
        if isinstance(node, ast.Call):
            # Check if the function called is named 'input'
            if isinstance(node.func, ast.Name) and node.func.id == "input":
                return True
    return False


def _rewrite_cell_magic(code: str) -> str:
    """Process a code block that uses cell magic.

    - Lines starting with "%%capture" are ignored.
    - Lines starting with "%pip" are rewritten by removing the leading "%" character.
    - Any other non-empty line causes a NotImplementedError.

    Args:
        code (str): The original code block.

    Returns:
        str: The transformed code block.

    Raises:
        NotImplementedError: If a line doesn't start with either "%%capture" or "%pip".
    """
    rewritten_lines = []

    for line in code.splitlines():
        stripped = line.strip()
        # Skip empty lines
        if not stripped:
            continue
        # Ignore %%capture lines
        if stripped.startswith("%%capture"):
            continue
        # Rewrite %pip lines by dropping the '%'
        elif stripped.startswith("%") or stripped.startswith("!"):
            # Drop the leading '%' character and then drop all leading whitespace
            stripped = stripped.lstrip("%! \t")
            # Check if the line starts with "pip"
            if stripped.startswith("pip"):
                rewritten_lines.append(stripped)
            else:
                raise NotImplementedError(f"Unhandled line: {line}")
        else:
            raise NotImplementedError(f"Unhandled line: {line}")

    return "\n".join(rewritten_lines)


class PrintCallVisitor(ast.NodeVisitor):
    """
    This visitor sets self.has_print to True if it encounters a call
    to a print within the global scope.

    This should catch calls to print(), print_stream(), etc. (Prefixed with "print").

    May have some false positives, but it's not meant to be perfect.

    Temporary code for notebook conversion.
    """

    def __init__(self):
        self.has_print = False
        self.scope_level = 0  # counter to track whether we're inside a def/lambda

    def visit_FunctionDef(self, node):
        self.scope_level += 1
        self.generic_visit(node)
        self.scope_level -= 1

    def visit_AsyncFunctionDef(self, node):
        self.scope_level += 1
        self.generic_visit(node)
        self.scope_level -= 1

    def visit_Lambda(self, node):
        self.scope_level += 1
        self.generic_visit(node)
        self.scope_level -= 1

    def visit_ClassDef(self, node):
        self.scope_level += 1
        self.generic_visit(node)
        self.scope_level -= 1

    def visit_Call(self, node):
        # Only consider calls when not inside a function definition.
        if self.scope_level == 0:
            if isinstance(node.func, ast.Name) and node.func.id.startswith("print"):
                self.has_print = True
        self.generic_visit(node)


def _has_output(source: str) -> bool:
    """Determine if the code block is expected to produce output.

    Args:
        source (str): The source code of the code block.

    Returns:
        True if the code block is expected to produce output, False otherwise.

        Must meet the following conditions:

        1. There is a call to a printing function (name starts with "print")
           that is not inside a function definition.
        2. The last top-level statement is an expression that is valid if:
         - It is any expression (including calls) AND
         - It is NOT a call to `display(...)`.

        `display` isn't handled currently by markdown-exec
    """
    try:
        tree = ast.parse(source)
    except SyntaxError:
        return False

    # Condition (1): Check for a global print-like call.
    visitor = PrintCallVisitor()
    visitor.visit(tree)
    condition_a = visitor.has_print

    # Condition (2): Check the last top-level statement.
    condition_b = False
    if tree.body:
        last_stmt = tree.body[-1]
        if isinstance(last_stmt, ast.Expr):
            # If the expression is a call, ensure it's not a call to "display"
            if isinstance(last_stmt.value, ast.Call):
                if (
                    isinstance(last_stmt.value.func, ast.Name)
                    and last_stmt.value.func.id == "display"
                ):
                    condition_b = False  # exclude display-wrapped expressions
                else:
                    condition_b = True
            else:
                # Any other expression qualifies.
                condition_b = True

    return condition_a or condition_b


def _convert_links_in_markdown(markdown: str) -> str:
    """Convert links present in notebook markdown cells to standardized format.

    We want to update markdown links code cells by linking to markdown
    files rather than assuming that the link is to the finalized HTML.

    This code is needed temporarily since the markdown links that are present
    in ipython notebooks do not follow the same conventions as regular markdown
    files in mkdocs (which should link to a .md file).
    """

    # Define the regex pattern in parts for clarity:
    pattern = (
        r"(?<!!)"  # Negative lookbehind: ensure the link is not an image (i.e., doesn't start with "!")
        r"\["  # Literal '[' indicating the start of the link text.
        r"(?P<text>[^\]]*)"  # Named group 'text': match any characters except ']', representing the link text.
        r"\]"  # Literal ']' indicating the end of the link text.
        r"\("  # Literal '(' indicating the start of the URL.
        r"(?![^\)]*//)"  # Negative lookahead: ensure that the URL does not contain '//' (skip absolute URLs).
        r"(?P<url>[^)]*)"  # Named group 'url': match any characters except ')', representing the URL.
        r"\)"  # Literal ')' indicating the end of the URL.
    )

    def custom_replacement(match):
        """logic will correct the link format used in ipython notebooks

        Ipython notebooks were being converted directly into HTML links
        instead of markdown links that retain the markdown extension.

        It needs to handle the following cases:
        - optional fragments (e.g., `#section`)
            e.g., `[text](url/#section)` -> `[text](url.md#section)`
            e.g., `[text](url#section)` -> `[text](url.md#section)`
        - relative paths (e.g., `../path/to/file`) need to be denested by 1 level
        """
        text = match.group("text")
        url = match.group("url")

        if url.startswith("../"):
            # we strip the "../" from the start of the URL
            # We only need to denest one level.
            url = url[3:]

        url = url.rstrip("/")  # Strip `/` from the end of the URL

        # if url has a fragment
        if "#" in url:
            url, fragment = url.split("#")
            url = url.rstrip("/")
            # Strip `/` from the end of the URL
            return f"[{text}]({url}.md#{fragment})"
        # Otherwise add the .md extension
        return f"[{text}]({url}.md)"

    return re.sub(
        pattern,
        custom_replacement,
        markdown,
    )


class HideCellTagPreprocessor(Preprocessor):
    """
    Removes cells that have '# hide-cell' at the beginning of the cell content.
    This allows authors to include cells in the notebook that should not
    appear in the generated markdown output.
    """

    def preprocess(self, nb, resources):
        # Filter out cells with the '# hide-cell' comment at the beginning
        nb.cells = [
            cell
            for cell in nb.cells
            if not (cell.source.strip().startswith("# hide-cell"))
        ]

        return nb, resources


class EscapePreprocessor(Preprocessor):
    def __init__(self, markdown_exec_migration: bool = False, **kwargs) -> None:
        super().__init__(**kwargs)
        self.markdown_exec_migration = markdown_exec_migration

    def preprocess_cell(self, cell, resources, cell_index):
        if cell.cell_type == "markdown":
            if not self.markdown_exec_migration:
                # Old logic is to convert ipynb links to HTML links
                cell.source = re.sub(
                    r"(?<!!)\[([^\]]*)\]\((?![^\)]*//)([^)]*)(?:\.ipynb)?\)",
                    r'<a href="\2">\1</a>',
                    cell.source,
                )
            else:
                cell.source = _convert_links_in_markdown(cell.source)

            # Fix image paths in <img> tags
            cell.source = re.sub(
                r'<img\s+src="\.?/img/([^"]+)"', r'<img src="../img/\1"', cell.source
            )

        elif cell.cell_type == "code":
            # Determine if the cell has bash or cell magic
            source = cell.source
            is_exec = not (
                source.startswith("%") or source.startswith("!") or _uses_input(source)
            )
            cell.metadata["exec"] = is_exec

            # For markdown exec migration we'll re-write cell magic as bash commands
            if source.startswith("%%"):
                cell.source = _rewrite_cell_magic(source)
                cell.metadata["language"] = "shell"

            # Remove noqa comments
            cell.source = re.sub(r"#\s*noqa.*$", "", cell.source, flags=re.MULTILINE)
            # escape ``` in code
            # This is needed because the markdown exporter will wrap code blocks in
            # triple backticks, which will break the markdown output if the code block
            # contains triple backticks.
            cell.source = cell.source.replace("```", r"\`\`\`")
            # escape ``` in output
            if "outputs" in cell:
                filter_out = set()
                for i, output in enumerate(cell["outputs"]):
                    if "text" in output:
                        if not output["text"].strip():
                            filter_out.add(i)
                            continue

                        value = output["text"].replace("```", r"\`\`\`")
                        # handle a funky case w/ references in text
                        value = re.sub(r"\[(\d+)\](?=\[(\d+)\])", r"[\1]\\", value)
                        output["text"] = value
                    elif "data" in output:
                        for key, value in output["data"].items():
                            if isinstance(value, str):
                                value = value.replace("```", r"\`\`\`")
                                # handle a funky case w/ references in text
                                output["data"][key] = re.sub(
                                    r"\[(\d+)\](?=\[(\d+)\])", r"[\1]\\", value
                                )
                cell["outputs"] = [
                    output
                    for i, output in enumerate(cell["outputs"])
                    if i not in filter_out
                ]

        return cell, resources


class ExtractAttachmentsPreprocessor(Preprocessor):
    """
    Extracts all of the outputs from the notebook file.  The extracted
    outputs are returned in the 'resources' dictionary.
    """

    def preprocess_cell(self, cell, resources, cell_index):
        """
        Apply a transformation on each cell,
        Parameters
        ----------
        cell : NotebookNode cell
            Notebook cell being processed
        resources : dictionary
            Additional resources used in the conversion process.  Allows
            preprocessors to pass variables into the Jinja engine.
        cell_index : int
            Index of the cell being processed (see base.py)
        """

        # Get files directory if it has been specified

        # Make sure outputs key exists
        if not isinstance(resources["outputs"], dict):
            resources["outputs"] = {}

        # Loop through all of the attachments in the cell
        for name, attach in cell.get("attachments", {}).items():
            for mime, data in attach.items():
                if mime not in {
                    "image/png",
                    "image/jpeg",
                    "image/svg+xml",
                    "application/pdf",
                }:
                    continue

                # attachments are pre-rendered. Only replace markdown-formatted
                # images with the following logic
                attach_str = f"({name})"
                if attach_str in cell.source:
                    data = f"(data:{mime};base64,{data})"
                    cell.source = cell.source.replace(attach_str, data)

        return cell, resources


exporter = MarkdownExporter(
    preprocessors=[
        HideCellTagPreprocessor,
        EscapePreprocessor,
        ExtractAttachmentsPreprocessor,
    ],
    template_name="mdoutput",
    extra_template_basedirs=[
        os.path.join(os.path.dirname(__file__), "notebook_convert_templates")
    ],
)


def convert_notebook(
    notebook_path: str,
    mode: Literal["markdown", "exec"] = "markdown",
) -> str:
    with open(notebook_path) as f:
        nb = nbformat.read(f, as_version=4)

    nb.metadata.mode = mode
    body, _ = exporter.from_notebook_node(nb)
    return body

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/notebook_hooks.py
```py
"""mkdocs hooks for adding custom logic to documentation pipeline.

Lifecycle events: https://www.mkdocs.org/dev-guide/plugins/#events
"""

import json
import logging
import os
import posixpath
import re
from typing import Any, Dict

from bs4 import BeautifulSoup
from mkdocs.config.defaults import MkDocsConfig
from mkdocs.structure.files import Files, File
from mkdocs.structure.pages import Page

from _scripts.generate_api_reference_links import update_markdown_with_imports
from _scripts.handle_auto_links import _replace_autolinks
from _scripts.notebook_convert import convert_notebook

logger = logging.getLogger(__name__)
logging.basicConfig()
logger.setLevel(logging.INFO)
DISABLED = os.getenv("DISABLE_NOTEBOOK_CONVERT") in ("1", "true", "True")


REDIRECT_MAP = {
    # lib redirects
    "how-tos/stream-values.ipynb": "https://docs.langchain.com/oss/python/langgraph/streaming",
    "how-tos/stream-updates.ipynb": "https://docs.langchain.com/oss/python/langgraph/streaming",
    "how-tos/streaming-content.ipynb": "https://docs.langchain.com/oss/python/langgraph/streaming",
    "how-tos/stream-multiple.ipynb": "https://docs.langchain.com/oss/python/langgraph/streaming",
    "how-tos/streaming-tokens-without-langchain.ipynb": "https://docs.langchain.com/oss/python/langgraph/streaming",
    "how-tos/streaming-from-final-node.ipynb": "https://docs.langchain.com/oss/python/langgraph/streaming",
    "how-tos/streaming-events-from-within-tools-without-langchain.ipynb": "https://docs.langchain.com/oss/python/langgraph/streaming",
    # graph-api
    "how-tos/state-reducers.ipynb": "https://docs.langchain.com/oss/python/langgraph/graph-api#define-and-update-state",
    "how-tos/sequence.ipynb": "https://docs.langchain.com/oss/python/langgraph/graph-api#create-a-sequence-of-steps",
    "how-tos/branching.ipynb": "https://docs.langchain.com/oss/python/langgraph/graph-api#create-branches",
    "how-tos/recursion-limit.ipynb": "https://docs.langchain.com/oss/python/langgraph/graph-api#create-and-control-loops",
    "how-tos/visualization.ipynb": "https://docs.langchain.com/oss/python/langgraph/graph-api#visualize-your-graph",
    "how-tos/input_output_schema.ipynb": "https://docs.langchain.com/oss/python/langgraph/graph-api#define-input-and-output-schemas",
    "how-tos/pass_private_state.ipynb": "https://docs.langchain.com/oss/python/langgraph/graph-api#pass-private-state-between-nodes",
    "how-tos/state-model.ipynb": "https://docs.langchain.com/oss/python/langgraph/graph-api#use-pydantic-models-for-graph-state",
    "how-tos/map-reduce.ipynb": "https://docs.langchain.com/oss/python/langgraph/graph-api#map-reduce-and-the-send-api",
    "how-tos/command.ipynb": "https://docs.langchain.com/oss/python/langgraph/graph-api#combine-control-flow-and-state-updates-with-command",
    "how-tos/configuration.ipynb": "https://docs.langchain.com/oss/python/langgraph/graph-api#add-runtime-configuration",
    "how-tos/node-retries.ipynb": "https://docs.langchain.com/oss/python/langgraph/graph-api#add-retry-policies",
    "how-tos/return-when-recursion-limit-hits.ipynb": "https://docs.langchain.com/oss/python/langgraph/graph-api#impose-a-recursion-limit",
    "how-tos/async.ipynb": "https://docs.langchain.com/oss/python/langgraph/graph-api#async",
    # memory how-tos
    "how-tos/memory/manage-conversation-history.ipynb": "https://docs.langchain.com/oss/python/langgraph/add-memory",
    "how-tos/memory/delete-messages.ipynb": "https://docs.langchain.com/oss/python/langgraph/add-memory#delete-messages",
    "how-tos/memory/add-summary-conversation-history.ipynb": "https://docs.langchain.com/oss/python/langgraph/add-memory#summarize-messages",
    "how-tos/memory.ipynb": "https://docs.langchain.com/oss/python/langgraph/add-memory",
    "agents/memory.ipynb": "https://docs.langchain.com/oss/python/langgraph/add-memory",
    # subgraph how-tos
    "how-tos/subgraph-transform-state.ipynb": "https://docs.langchain.com/oss/python/langgraph/use-subgraphs#different-state-schemas",
    "how-tos/subgraphs-manage-state.ipynb": "https://docs.langchain.com/oss/python/langgraph/use-subgraphs#add-persistence",
    # persistence how-tos
    "how-tos/persistence_postgres.ipynb": "https://docs.langchain.com/oss/python/langgraph/add-memory#use-in-production",
    "how-tos/persistence_mongodb.ipynb": "https://docs.langchain.com/oss/python/langgraph/add-memory#use-in-production",
    "how-tos/persistence_redis.ipynb": "https://docs.langchain.com/oss/python/langgraph/add-memory#use-in-production",
    "how-tos/subgraph-persistence.ipynb": "https://docs.langchain.com/oss/python/langgraph/add-memory#use-with-subgraphs",
    "how-tos/cross-thread-persistence.ipynb": "https://docs.langchain.com/oss/python/langgraph/add-memory#add-long-term-memory",
    "cloud/how-tos/copy_threads": "https://docs.langchain.com/langsmith/use-threads",
    "cloud/how-tos/check-thread-status": "https://docs.langchain.com/langsmith/use-threads",
    "cloud/concepts/threads.md": "https://docs.langchain.com/oss/python/langgraph/persistence#threads",
    "how-tos/persistence.ipynb": "https://docs.langchain.com/oss/python/langgraph/add-memory",
    # tool calling how-tos
    "how-tos/tool-calling-errors.ipynb": "https://docs.langchain.com/oss/python/langgraph/workflows-agents",
    "how-tos/pass-config-to-tools.ipynb": "https://docs.langchain.com/oss/python/langgraph/workflows-agents",
    "how-tos/pass-run-time-values-to-tools.ipynb": "https://docs.langchain.com/oss/python/langgraph/workflows-agents",
    "how-tos/update-state-from-tools.ipynb": "https://docs.langchain.com/oss/python/langgraph/workflows-agents",
    "agents/tools.md": "https://docs.langchain.com/oss/python/langgraph/workflows-agents",
    # multi-agent how-tos
    "how-tos/agent-handoffs.ipynb": "https://docs.langchain.com/oss/python/langgraph/graph-api",
    "how-tos/multi-agent-network.ipynb": "https://docs.langchain.com/oss/python/langgraph/graph-api",
    "how-tos/multi-agent-multi-turn-convo.ipynb": "https://docs.langchain.com/oss/python/langgraph/graph-api",
    # cloud redirects
    "cloud/index.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "cloud/how-tos/index.md": "https://docs.langchain.com/langsmith/home",
    "cloud/concepts/api.md": "https://docs.langchain.com/langsmith/agent-server",
    "cloud/concepts/cloud.md": "https://docs.langchain.com/langsmith/cloud",
    "cloud/faq/studio.md": "https://docs.langchain.com/langsmith/studio",
    "cloud/how-tos/human_in_the_loop_edit_state.md": "https://docs.langchain.com/langsmith/add-human-in-the-loop",
    "cloud/how-tos/human_in_the_loop_user_input.md": "https://docs.langchain.com/langsmith/add-human-in-the-loop",
    "concepts/platform_architecture.md": "https://docs.langchain.com/langsmith/cloud#architecture",
    # cloud streaming redirects
    "cloud/how-tos/stream_values.md": "https://docs.langchain.com/langsmith/streaming",
    "cloud/how-tos/stream_updates.md": "https://docs.langchain.com/langsmith/streaming",
    "cloud/how-tos/stream_messages.md": "https://docs.langchain.com/langsmith/streaming",
    "cloud/how-tos/stream_events.md": "https://docs.langchain.com/langsmith/streaming",
    "cloud/how-tos/stream_debug.md": "https://docs.langchain.com/langsmith/streaming",
    "cloud/how-tos/stream_multiple.md": "https://docs.langchain.com/langsmith/streaming",
    "cloud/concepts/streaming.md": "https://docs.langchain.com/oss/python/langgraph/streaming",
    "agents/streaming.md": "https://docs.langchain.com/oss/python/langgraph/streaming",
    # prebuilt redirects
    "how-tos/create-react-agent.ipynb": "https://docs.langchain.com/oss/python/langchain/agents#basic-configuration",
    "how-tos/create-react-agent-memory.ipynb": "https://docs.langchain.com/oss/python/langgraph/add-memory",
    "how-tos/create-react-agent-system-prompt.ipynb": "https://docs.langchain.com/oss/python/langgraph/add-memory",
    "how-tos/create-react-agent-structured-output.ipynb": "https://docs.langchain.com/oss/python/langchain/agents#structured-output",
    # misc
    "prebuilt.md": "https://docs.langchain.com/oss/python/langchain/agents",
    "reference/prebuilt.md": "https://reference.langchain.com/python/langgraph/agents/",
    "concepts/high_level.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "concepts/index.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "concepts/v0-human-in-the-loop.md": "https://docs.langchain.com/oss/python/langgraph/interrupts",
    "how-tos/index.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "tutorials/introduction.ipynb": "https://docs.langchain.com/oss/python/langgraph/overview",
    "agents/deployment.md": "https://docs.langchain.com/oss/python/langgraph/local-server",
    # deployment redirects
    "how-tos/deploy-self-hosted.md": "https://docs.langchain.com/langsmith/platform-setup",
    "concepts/self_hosted.md": "https://docs.langchain.com/langsmith/platform-setup",
    "tutorials/deployment.md": "https://docs.langchain.com/langsmith/deployments",
    # assistant redirects
    "cloud/how-tos/assistant_versioning.md": "https://docs.langchain.com/langsmith/configuration-cloud",
    "cloud/concepts/runs.md": "https://docs.langchain.com/langsmith/assistants#execution",
    # hitl redirects
    "how-tos/wait-user-input-functional.ipynb": "https://docs.langchain.com/oss/python/langgraph/functional-api",
    "how-tos/review-tool-calls-functional.ipynb": "https://docs.langchain.com/oss/python/langgraph/functional-api",
    "how-tos/create-react-agent-hitl.ipynb": "https://docs.langchain.com/oss/python/langgraph/interrupts",
    "agents/human-in-the-loop.md": "https://docs.langchain.com/oss/python/langgraph/interrupts",
    "how-tos/human_in_the_loop/dynamic_breakpoints.ipynb": "https://docs.langchain.com/oss/python/langgraph/interrupts",
    "concepts/breakpoints.md": "https://docs.langchain.com/oss/python/langgraph/interrupts",
    "how-tos/human_in_the_loop/breakpoints.md": "https://docs.langchain.com/oss/python/langgraph/interrupts",
    "cloud/how-tos/human_in_the_loop_breakpoint.md": "https://docs.langchain.com/langsmith/add-human-in-the-loop",
    "how-tos/human_in_the_loop/edit-graph-state.ipynb": "https://docs.langchain.com/oss/python/langgraph/use-time-travel",

    # LGP mintlify migration redirects
    "tutorials/auth/getting_started.md": "https://docs.langchain.com/langsmith/auth",
    "tutorials/auth/resource_auth.md": "https://docs.langchain.com/langsmith/resource-auth",
    "tutorials/auth/add_auth_server.md": "https://docs.langchain.com/langsmith/add-auth-server",
    "how-tos/use-remote-graph.md": "https://docs.langchain.com/langsmith/use-remote-graph",
    "how-tos/autogen-integration.md": "https://docs.langchain.com/langsmith/autogen-integration",
    "cloud/how-tos/use_stream_react.md": "https://docs.langchain.com/langsmith/use-stream-react",
    "cloud/how-tos/generative_ui_react.md": "https://docs.langchain.com/langsmith/generative-ui-react",
    "concepts/langgraph_platform.md": "https://docs.langchain.com/langsmith/deployments",
    "concepts/langgraph_components.md": "https://docs.langchain.com/langsmith/components",
    "concepts/langgraph_server.md": "https://docs.langchain.com/langsmith/agent-server",
    "concepts/langgraph_data_plane.md": "https://docs.langchain.com/langsmith/data-plane",
    "concepts/langgraph_control_plane.md": "https://docs.langchain.com/langsmith/control-plane",
    "concepts/langgraph_cli.md": "https://docs.langchain.com/langsmith/cli",
    "concepts/langgraph_studio.md": "https://docs.langchain.com/langsmith/studio",
    "cloud/how-tos/studio/quick_start.md": "https://docs.langchain.com/langsmith/quick-start-studio",
    "cloud/how-tos/invoke_studio.md": "https://docs.langchain.com/langsmith/use-studio#run-application",
    "cloud/how-tos/studio/manage_assistants.md": "https://docs.langchain.com/langsmith/use-studio#manage-assistants",
    "cloud/how-tos/threads_studio.md": "https://docs.langchain.com/langsmith/use-studio#manage-threads",
    "cloud/how-tos/iterate_graph_studio.md": "https://docs.langchain.com/langsmith/observability-studio#iterate-on-prompts",
    "cloud/how-tos/studio/run_evals.md": "https://docs.langchain.com/langsmith/observability-studio#run-experiments-over-a-dataset",
    "cloud/how-tos/clone_traces_studio.md": "https://docs.langchain.com/langsmith/observability-studio#debug-langsmith-traces",
    "cloud/how-tos/datasets_studio.md": "https://docs.langchain.com/langsmith/observability-studio#add-node-to-dataset",
    "concepts/sdk.md": "https://docs.langchain.com/langsmith/sdk",
    "concepts/plans.md": "https://langchain.com/pricing",
    "concepts/application_structure.md": "https://docs.langchain.com/langsmith/application-structure",
    "concepts/scalability_and_resilience.md": "https://docs.langchain.com/langsmith/scalability-and-resilience",
    "concepts/auth.md": "https://docs.langchain.com/langsmith/authentication-methods",
    "how-tos/auth/custom_auth.md": "https://docs.langchain.com/langsmith/custom-auth",
    "how-tos/auth/openapi_security.md": "https://docs.langchain.com/langsmith/openapi-security",
    "concepts/assistants.md": "https://docs.langchain.com/langsmith/assistants",
    "cloud/how-tos/configuration_cloud.md": "https://docs.langchain.com/langsmith/cloud",
    "cloud/how-tos/use_threads.md": "https://docs.langchain.com/langsmith/use-threads",
    "cloud/how-tos/background_run.md": "https://docs.langchain.com/langsmith/background-run",
    "cloud/how-tos/same-thread.md": "https://docs.langchain.com/langsmith/same-thread",
    "cloud/how-tos/stateless_runs.md": "https://docs.langchain.com/langsmith/stateless-runs",
    "cloud/how-tos/configurable_headers.md": "https://docs.langchain.com/langsmith/configurable-headers",
    "concepts/double_texting.md": "https://docs.langchain.com/langsmith/double-texting",
    "cloud/how-tos/interrupt_concurrent.md": "https://docs.langchain.com/langsmith/interrupt-concurrent",
    "cloud/how-tos/rollback_concurrent.md": "https://docs.langchain.com/langsmith/rollback-concurrent",
    "cloud/how-tos/reject_concurrent.md": "https://docs.langchain.com/langsmith/reject-concurrent",
    "cloud/how-tos/enqueue_concurrent.md": "https://docs.langchain.com/langsmith/enqueue-concurrent",
    "cloud/concepts/webhooks.md": "https://docs.langchain.com/langsmith/use-webhooks",
    "cloud/how-tos/webhooks.md": "https://docs.langchain.com/langsmith/use-webhooks",
    "cloud/concepts/cron_jobs.md": "https://docs.langchain.com/langsmith/cron-jobs",
    "cloud/how-tos/cron_jobs.md": "https://docs.langchain.com/langsmith/cron-jobs",
    "how-tos/http/custom_lifespan.md": "https://docs.langchain.com/langsmith/custom-lifespan",
    "how-tos/http/custom_middleware.md": "https://docs.langchain.com/langsmith/custom-middleware",
    "how-tos/http/custom_routes.md": "https://docs.langchain.com/langsmith/custom-routes",
    "cloud/concepts/data_storage_and_privacy.md": "https://docs.langchain.com/langsmith/data-storage-and-privacy",
    "cloud/deployment/semantic_search.md": "https://docs.langchain.com/langsmith/semantic-search",
    "how-tos/ttl/configure_ttl.md": "https://docs.langchain.com/langsmith/configure-ttl",
    "concepts/deployment_options.md": "https://docs.langchain.com/langsmith/platform-setup",
    "cloud/quick_start.md": "https://docs.langchain.com/langsmith/deployment-quickstart",
    "cloud/deployment/setup.md": "https://docs.langchain.com/langsmith/setup-app-requirements-txt",
    "cloud/deployment/setup_pyproject.md": "https://docs.langchain.com/langsmith/setup-pyproject",
    "cloud/deployment/setup_javascript.md": "https://docs.langchain.com/langsmith/setup-javascript",
    "cloud/deployment/custom_docker.md": "https://docs.langchain.com/langsmith/custom-docker",
    "cloud/deployment/graph_rebuild.md": "https://docs.langchain.com/langsmith/graph-rebuild",
    "concepts/langgraph_cloud.md": "https://docs.langchain.com/langsmith/cloud",
    "concepts/langgraph_self_hosted_data_plane.md": "https://docs.langchain.com/langsmith/hybrid",
    "concepts/langgraph_self_hosted_control_plane.md": "https://docs.langchain.com/langsmith/self-hosted",
    "concepts/langgraph_standalone_container.md": "https://docs.langchain.com/langsmith/self-hosted#standalone-server",
    "cloud/deployment/cloud.md": "https://docs.langchain.com/langsmith/cloud",
    "cloud/deployment/self_hosted_data_plane.md": "https://docs.langchain.com/langsmith/deploy-hybrid",
    "cloud/deployment/self_hosted_control_plane.md": "https://docs.langchain.com/langsmith/deploy-self-hosted-full-platform",
    "cloud/deployment/standalone_container.md": "https://docs.langchain.com/langsmith/deploy-standalone-server",
    "concepts/server-mcp.md": "https://docs.langchain.com/langsmith/server-mcp",
    "cloud/how-tos/human_in_the_loop_time_travel.md": "https://docs.langchain.com/langsmith/human-in-the-loop-time-travel",
    "cloud/how-tos/add-human-in-the-loop.md": "https://docs.langchain.com/langsmith/add-human-in-the-loop",
    "cloud/deployment/egress.md": "https://docs.langchain.com/langsmith/env-var",
    "cloud/how-tos/streaming.md": "https://docs.langchain.com/langsmith/streaming",
    "cloud/reference/api/api_ref.md": "https://docs.langchain.com/langsmith/server-api-ref",
    "cloud/reference/langgraph_server_changelog.md": "https://docs.langchain.com/langsmith/agent-server-changelog",
    "cloud/reference/api/api_ref_control_plane.md": "https://docs.langchain.com/langsmith/api-ref-control-plane",
    "cloud/reference/cli.md": "https://docs.langchain.com/langsmith/cli",
    "cloud/reference/env_var.md": "https://docs.langchain.com/langsmith/env-var",
    "troubleshooting/studio.md": "https://docs.langchain.com/langsmith/troubleshooting-studio",

    # LangGraph mintlify migration redirects
    "index.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "agents/agents.md": "https://docs.langchain.com/oss/python/langchain/agents",
    "concepts/why-langgraph.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "tutorials/get-started/1-build-basic-chatbot.md": "https://docs.langchain.com/oss/python/langgraph/quickstart",
    "tutorials/get-started/2-add-tools.md": "https://docs.langchain.com/oss/python/langgraph/quickstart",
    "tutorials/get-started/3-add-memory.md": "https://docs.langchain.com/oss/python/langgraph/quickstart",
    "tutorials/get-started/4-human-in-the-loop.md": "https://docs.langchain.com/oss/python/langgraph/quickstart",
    "tutorials/get-started/5-customize-state.md": "https://docs.langchain.com/oss/python/langgraph/quickstart",
    "tutorials/get-started/6-time-travel.md": "https://docs.langchain.com/oss/python/langgraph/quickstart",
    "tutorials/langsmith/local-server.md": "https://docs.langchain.com/oss/python/langgraph/local-server",
    "tutorials/workflows.md": "https://docs.langchain.com/oss/python/langgraph/workflows-agents",
    "concepts/agentic_concepts.md": "https://docs.langchain.com/oss/python/langgraph/workflows-agents",
    "guides/index.md": "https://docs.langchain.com/oss/python/langchain/overview",
    "agents/overview.md": "https://docs.langchain.com/oss/python/langchain/agents",
    "agents/run_agents.md": "https://docs.langchain.com/oss/python/langgraph/quickstart",
    "concepts/low_level.md": "https://docs.langchain.com/oss/python/langgraph/graph-api",
    "how-tos/graph-api.md": "https://docs.langchain.com/oss/python/langgraph/graph-api",
    "concepts/functional_api.md": "https://docs.langchain.com/oss/python/langgraph/functional-api",
    "how-tos/use-functional-api.md": "https://docs.langchain.com/oss/python/langgraph/functional-api",
    "concepts/pregel.md": "https://docs.langchain.com/oss/python/langgraph/pregel",
    "concepts/streaming.md": "https://docs.langchain.com/oss/python/langgraph/streaming",
    "how-tos/streaming.md": "https://docs.langchain.com/oss/python/langgraph/streaming",
    "concepts/persistence.md": "https://docs.langchain.com/oss/python/langgraph/persistence",
    "concepts/durable_execution.md": "https://docs.langchain.com/oss/python/langgraph/durable-execution",
    "concepts/memory.md": "https://docs.langchain.com/oss/python/langgraph/memory",
    "how-tos/memory/add-memory.md": "https://docs.langchain.com/oss/python/langgraph/add-memory",
    "agents/context.md": "https://docs.langchain.com/oss/python/langgraph/add-memory",
    "agents/models.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "concepts/tools.md": "https://docs.langchain.com/oss/python/langgraph/workflows-agents",
    "how-tos/tool-calling.md": "https://docs.langchain.com/oss/python/langgraph/workflows-agents",
    "concepts/human_in_the_loop.md": "https://docs.langchain.com/oss/python/langgraph/interrupts",
    "how-tos/human_in_the_loop/add-human-in-the-loop.md": "https://docs.langchain.com/oss/python/langgraph/interrupts",
    "concepts/time-travel.md": "https://docs.langchain.com/oss/python/langgraph/persistence",
    "how-tos/human_in_the_loop/time-travel.md": "https://docs.langchain.com/oss/python/langgraph/use-time-travel",
    "concepts/subgraphs.md": "https://docs.langchain.com/oss/python/langgraph/use-subgraphs",
    "how-tos/subgraph.md": "https://docs.langchain.com/oss/python/langgraph/use-subgraphs",
    "concepts/multi_agent.md": "https://docs.langchain.com/oss/python/langgraph/graph-api",
    "agents/multi-agent.md": "https://docs.langchain.com/oss/python/langchain/multi-agent",
    "how-tos/multi_agent.md": "https://docs.langchain.com/oss/python/langgraph/graph-api",
    "concepts/mcp.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "agents/mcp.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "concepts/tracing.md": "https://docs.langchain.com/oss/python/langgraph/observability",
    "how-tos/enable-tracing.md": "https://docs.langchain.com/oss/python/langgraph/observability",
    "agents/evals.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "examples/index.md": "https://docs.langchain.com/oss/python/langgraph/case-studies",
    "concepts/template_applications.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "tutorials/rag/langgraph_agentic_rag.md": "https://docs.langchain.com/oss/python/langgraph/agentic-rag",
    "tutorials/multi_agent/agent_supervisor.md": "https://docs.langchain.com/oss/python/langgraph/workflows-agents",
    "tutorials/sql/sql-agent.md": "https://docs.langchain.com/oss/python/langgraph/sql-agent",
    "agents/ui.md": "https://docs.langchain.com/oss/python/langgraph/ui",
    "how-tos/run-id-langsmith.md": "https://docs.langchain.com/oss/python/langgraph/observability",
    "troubleshooting/errors/index.md": "https://docs.langchain.com/oss/python/langgraph/common-errors",
    "troubleshooting/errors/INVALID_CHAT_HISTORY.md": "https://docs.langchain.com/oss/python/langgraph/INVALID_CHAT_HISTORY",
    "troubleshooting/errors/INVALID_LICENSE.md": "https://docs.langchain.com/oss/python/langgraph/common-errors",
    "adopters.md": "https://docs.langchain.com/oss/python/langgraph/case-studies",
    "concepts/faq.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "agents/prebuilt.md": "https://docs.langchain.com/oss/python/langchain/agents",
    "reference/index.md": "https://reference.langchain.com/python/langgraph/",
    "reference/graphs.md": "https://reference.langchain.com/python/langgraph/graphs/",
    "reference/func.md": "https://reference.langchain.com/python/langgraph/func/",
    "reference/pregel.md": "https://reference.langchain.com/python/langgraph/pregel/",
    "reference/checkpoints.md": "https://reference.langchain.com/python/langgraph/checkpoints/",
    "reference/store.md": "https://reference.langchain.com/python/langgraph/store/",
    "reference/cache.md": "https://reference.langchain.com/python/langgraph/cache/",
    "reference/types.md": "https://reference.langchain.com/python/langgraph/types/",
    "reference/runtime.md": "https://reference.langchain.com/python/langgraph/runtime/",
    "reference/config.md": "https://reference.langchain.com/python/langgraph/config/",
    "reference/errors.md": "https://reference.langchain.com/python/langgraph/errors/",
    "reference/constants.md": "https://reference.langchain.com/python/langgraph/constants/",
    "reference/channels.md": "https://reference.langchain.com/python/langgraph/channels/",
    "reference/agents.md": "https://reference.langchain.com/python/langgraph/agents/",
    "reference/supervisor.md": "https://reference.langchain.com/python/langgraph/supervisor/",
    "reference/swarm.md": "https://reference.langchain.com/python/langgraph/swarm/",
    "reference/mcp.md": "https://reference.langchain.com/python/langgraph/mcp/",
    "cloud/reference/sdk/python_sdk_ref.md": "https://reference.langchain.com/python/langsmith/deployment/sdk/",
    "reference/remote_graph.md": "https://reference.langchain.com/python/langsmith/deployment/remote_graph/",

    # additional exclude-search entries from mkdocs.yml
    "additional-resources/index.md": "https://docs.langchain.com/oss/python/langchain/overview",
    "cloud/concepts/cron_jobs.md": "https://docs.langchain.com/langsmith/cron-jobs",
    "cloud/concepts/data_storage_and_privacy.md": "https://docs.langchain.com/langsmith/data-storage-and-privacy",
    "cloud/concepts/webhooks.md": "https://docs.langchain.com/langsmith/use-webhooks",
    "cloud/deployment/cloud.md": "https://docs.langchain.com/langsmith/cloud",
    "cloud/deployment/custom_docker.md": "https://docs.langchain.com/langsmith/custom-docker",
    "cloud/deployment/egress.md": "https://docs.langchain.com/langsmith/env-var",
    "cloud/deployment/graph_rebuild.md": "https://docs.langchain.com/langsmith/graph-rebuild",
    "cloud/deployment/self_hosted_control_plane.md": "https://docs.langchain.com/langsmith/platform-setup",
    "cloud/deployment/self_hosted_data_plane.md": "https://docs.langchain.com/langsmith/platform-setup",
    "cloud/deployment/semantic_search.md": "https://docs.langchain.com/langsmith/semantic-search",
    "cloud/deployment/setup_javascript.md": "https://docs.langchain.com/langsmith/setup-javascript",
    "cloud/deployment/setup_pyproject.md": "https://docs.langchain.com/langsmith/setup-pyproject",
    "cloud/deployment/setup.md": "https://docs.langchain.com/langsmith/setup-app-requirements-txt",
    "cloud/deployment/standalone_container.md": "https://docs.langchain.com/langsmith/docker",
    "cloud/how-tos/add-human-in-the-loop.md": "https://docs.langchain.com/langsmith/add-human-in-the-loop",
    "cloud/how-tos/background_run.md": "https://docs.langchain.com/langsmith/background-run",
    "cloud/how-tos/clone_traces_studio.md": "https://docs.langchain.com/langsmith/observability",
    "cloud/how-tos/configurable_headers.md": "https://docs.langchain.com/langsmith/configurable-headers",
    "cloud/how-tos/configuration_cloud.md": "https://docs.langchain.com/langsmith/configuration-cloud",
    "cloud/how-tos/cron_jobs.md": "https://docs.langchain.com/langsmith/cron-jobs",
    "cloud/how-tos/datasets_studio.md": "https://docs.langchain.com/langsmith/use-studio",
    "cloud/how-tos/enqueue_concurrent.md": "https://docs.langchain.com/langsmith/enqueue-concurrent",
    "cloud/how-tos/generative_ui_react.md": "https://docs.langchain.com/langsmith/generative-ui-react",
    "cloud/how-tos/human_in_the_loop_time_travel.md": "https://docs.langchain.com/langsmith/human-in-the-loop-time-travel",
    "cloud/how-tos/interrupt_concurrent.md": "https://docs.langchain.com/langsmith/interrupt-concurrent",
    "cloud/how-tos/invoke_studio.md": "https://docs.langchain.com/langsmith/use-studio",
    "cloud/how-tos/iterate_graph_studio.md": "https://docs.langchain.com/langsmith/use-studio",
    "cloud/how-tos/reject_concurrent.md": "https://docs.langchain.com/langsmith/reject-concurrent",
    "cloud/how-tos/rollback_concurrent.md": "https://docs.langchain.com/langsmith/rollback-concurrent",
    "cloud/how-tos/same-thread.md": "https://docs.langchain.com/langsmith/same-thread",
    "cloud/how-tos/stateless_runs.md": "https://docs.langchain.com/langsmith/stateless-runs",
    "cloud/how-tos/streaming.md": "https://docs.langchain.com/langsmith/streaming",
    "cloud/how-tos/studio/manage_assistants.md": "https://docs.langchain.com/langsmith/use-studio",
    "cloud/how-tos/studio/quick_start.md": "https://docs.langchain.com/langsmith/quick-start-studio",
    "cloud/how-tos/studio/run_evals.md": "https://docs.langchain.com/langsmith/observability",
    "cloud/how-tos/threads_studio.md": "https://docs.langchain.com/langsmith/use-threads",
    "cloud/how-tos/use_stream_react.md": "https://docs.langchain.com/langsmith/use-stream-react",
    "cloud/how-tos/use_threads.md": "https://docs.langchain.com/langsmith/use-threads",
    "cloud/how-tos/webhooks.md": "https://docs.langchain.com/langsmith/use-webhooks",
    "cloud/quick_start.md": "https://docs.langchain.com/langsmith/deployment-quickstart",
    "cloud/reference/api/api_ref_control_plane.md": "https://docs.langchain.com/langsmith/api-ref-control-plane",
    "cloud/reference/api/api_ref.md": "https://docs.langchain.com/langsmith/server-api-ref",
    "cloud/reference/cli.md": "https://docs.langchain.com/langsmith/cli",
    "cloud/reference/env_var.md": "https://docs.langchain.com/langsmith/env-var",
    "cloud/reference/langgraph_server_changelog.md": "https://docs.langchain.com/langsmith/agent-server-changelog",
    "cloud/reference/sdk/js_ts_sdk_ref.md": "https://reference.langchain.com/javascript/modules/langsmith.html",
    "concepts/application_structure.md": "https://docs.langchain.com/langsmith/application-structure",
    "concepts/assistants.md": "https://docs.langchain.com/langsmith/assistants",
    "concepts/auth.md": "https://docs.langchain.com/langsmith/auth",
    "concepts/deployment_options.md": "https://docs.langchain.com/langsmith/deployments",
    "concepts/double_texting.md": "https://docs.langchain.com/langsmith/double-texting",
    "concepts/faq.md": "https://docs.langchain.com/langsmith/faq",
    "concepts/langgraph_cli.md": "https://docs.langchain.com/langsmith/cli",
    "concepts/langgraph_cloud.md": "https://docs.langchain.com/langsmith/cloud",
    "concepts/langgraph_components.md": "https://docs.langchain.com/langsmith/components",
    "concepts/langgraph_control_plane.md": "https://docs.langchain.com/langsmith/control-plane",
    "concepts/langgraph_data_plane.md": "https://docs.langchain.com/langsmith/data-plane",
    "concepts/langgraph_platform.md": "https://docs.langchain.com/langsmith/home",
    "concepts/langgraph_self_hosted_control_plane.md": "https://docs.langchain.com/langsmith/platform-setup",
    "concepts/langgraph_self_hosted_data_plane.md": "https://docs.langchain.com/langsmith/platform-setup",
    "concepts/langgraph_server.md": "https://docs.langchain.com/langsmith/agent-server",
    "concepts/langgraph_standalone_container.md": "https://docs.langchain.com/langsmith/docker",
    "concepts/langgraph_studio.md": "https://docs.langchain.com/langsmith/studio",
    "concepts/plans.md": "https://docs.langchain.com/langsmith/home",
    "concepts/scalability_and_resilience.md": "https://docs.langchain.com/langsmith/scalability-and-resilience",
    "concepts/sdk.md": "https://docs.langchain.com/langsmith/sdk",
    "concepts/server-mcp.md": "https://docs.langchain.com/langsmith/server-mcp",
    "concepts/template_applications.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "concepts/why-langgraph.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "examples/index.md": "https://docs.langchain.com/oss/python/langgraph/case-studies",
    "guides/index.md": "https://docs.langchain.com/oss/python/langchain/overview",
    "how-tos/auth/custom_auth.md": "https://docs.langchain.com/langsmith/custom-auth",
    "how-tos/auth/openapi_security.md": "https://docs.langchain.com/langsmith/openapi-security",
    "how-tos/autogen-integration.md": "https://docs.langchain.com/langsmith/autogen-integration",
    "how-tos/http/custom_lifespan.md": "https://docs.langchain.com/langsmith/custom-lifespan",
    "how-tos/http/custom_middleware.md": "https://docs.langchain.com/langsmith/custom-middleware",
    "how-tos/http/custom_routes.md": "https://docs.langchain.com/langsmith/custom-routes",
    "how-tos/ttl/configure_ttl.md": "https://docs.langchain.com/langsmith/configure-ttl",
    "how-tos/use-remote-graph.md": "https://docs.langchain.com/langsmith/use-remote-graph",
    "index.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "snippets/chat_model_tabs.md": "https://docs.langchain.com/oss/python/langchain/overview",
    "troubleshooting/errors/GRAPH_RECURSION_LIMIT.md": "https://docs.langchain.com/oss/python/langgraph/GRAPH_RECURSION_LIMIT",
    "troubleshooting/errors/index.md": "https://docs.langchain.com/oss/python/langgraph/common-errors",
    "troubleshooting/errors/INVALID_CHAT_HISTORY.md": "https://docs.langchain.com/oss/python/langgraph/INVALID_CHAT_HISTORY",
    "troubleshooting/errors/INVALID_CONCURRENT_GRAPH_UPDATE.md": "https://docs.langchain.com/oss/python/langgraph/INVALID_CONCURRENT_GRAPH_UPDATE",
    "troubleshooting/errors/INVALID_GRAPH_NODE_RETURN_VALUE.md": "https://docs.langchain.com/oss/python/langgraph/INVALID_GRAPH_NODE_RETURN_VALUE",
    "troubleshooting/errors/INVALID_LICENSE.md": "https://docs.langchain.com/oss/python/langgraph/common-errors",
    "troubleshooting/errors/MULTIPLE_SUBGRAPHS.md": "https://docs.langchain.com/oss/python/langgraph/MULTIPLE_SUBGRAPHS",
    "troubleshooting/studio.md": "https://docs.langchain.com/langsmith/troubleshooting-studio",
    "tutorials/auth/add_auth_server.md": "https://docs.langchain.com/langsmith/add-auth-server",
    "tutorials/auth/getting_started.md": "https://docs.langchain.com/langsmith/auth",
    "tutorials/auth/resource_auth.md": "https://docs.langchain.com/langsmith/resource-auth",
    "agents/agents.md": "https://docs.langchain.com/oss/python/langchain/agents",
    "concepts/why-langgraph.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "tutorials/langsmith/local-server.md": "https://docs.langchain.com/oss/python/langgraph/local-server",
    "tutorials/workflows.md": "https://docs.langchain.com/oss/python/langgraph/workflows-agents",
    "concepts/agentic_concepts.md": "https://docs.langchain.com/oss/python/langgraph/workflows-agents",
    "guides/index.md": "https://docs.langchain.com/oss/python/langchain/overview",
    "agents/overview.md": "https://docs.langchain.com/oss/python/langchain/agents",
    "concepts/agentic_concepts.md": "https://docs.langchain.com/oss/python/langgraph/workflows-agents",
    "agents/run_agents.md": "https://docs.langchain.com/oss/python/langgraph/quickstart",
    "concepts/low_level.md": "https://docs.langchain.com/oss/python/langgraph/graph-api",
    "how-tos/graph-api.md": "https://docs.langchain.com/oss/python/langgraph/graph-api",
    "concepts/functional_api.md": "https://docs.langchain.com/oss/python/langgraph/functional-api",
    "how-tos/use-functional-api.md": "https://docs.langchain.com/oss/python/langgraph/functional-api",
    "concepts/pregel.md": "https://docs.langchain.com/oss/python/langgraph/pregel",
    "concepts/streaming.md": "https://docs.langchain.com/oss/python/langgraph/streaming",
    "how-tos/streaming.md": "https://docs.langchain.com/oss/python/langgraph/streaming",
    "concepts/persistence.md": "https://docs.langchain.com/oss/python/langgraph/persistence",
    "concepts/durable_execution.md": "https://docs.langchain.com/oss/python/langgraph/durable-execution",
    "concepts/memory.md": "https://docs.langchain.com/oss/python/langgraph/memory",
    "how-tos/memory/add-memory.md": "https://docs.langchain.com/oss/python/langgraph/add-memory",
    "agents/context.md": "https://docs.langchain.com/oss/python/langgraph/add-memory",
    "agents/models.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "concepts/tools.md": "https://docs.langchain.com/oss/python/langgraph/workflows-agents",
    "how-tos/tool-calling.md": "https://docs.langchain.com/oss/python/langgraph/workflows-agents",
    "concepts/human_in_the_loop.md": "https://docs.langchain.com/oss/python/langgraph/interrupts",
    "how-tos/human_in_the_loop/add-human-in-the-loop.md": "https://docs.langchain.com/oss/python/langgraph/interrupts",
    "concepts/time-travel.md": "https://docs.langchain.com/oss/python/langgraph/persistence",
    "how-tos/human_in_the_loop/time-travel.md": "https://docs.langchain.com/oss/python/langgraph/use-time-travel",
    "concepts/subgraphs.md": "https://docs.langchain.com/oss/python/langgraph/use-subgraphs",
    "how-tos/subgraph.md": "https://docs.langchain.com/oss/python/langgraph/use-subgraphs",
    "concepts/multi_agent.md": "https://docs.langchain.com/oss/python/langgraph/graph-api",
    "agents/multi-agent.md": "https://docs.langchain.com/oss/python/langchain/multi-agent",
    "how-tos/multi_agent.md": "https://docs.langchain.com/oss/python/langgraph/graph-api",
    "concepts/mcp.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "agents/mcp.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "concepts/tracing.md": "https://docs.langchain.com/oss/python/langgraph/observability",
    "how-tos/enable-tracing.md": "https://docs.langchain.com/oss/python/langgraph/observability",
    "agents/evals.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "examples/index.md": "https://docs.langchain.com/oss/python/langgraph/case-studies",
    "concepts/template_applications.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "tutorials/rag/langgraph_agentic_rag.md": "https://docs.langchain.com/oss/python/langgraph/agentic-rag",
    "tutorials/multi_agent/agent_supervisor.md": "https://docs.langchain.com/oss/python/langgraph/workflows-agents",
    "tutorials/sql/sql-agent.md": "https://docs.langchain.com/oss/python/langgraph/sql-agent",
    "agents/ui.md": "https://docs.langchain.com/oss/python/langgraph/ui",
    "how-tos/run-id-langsmith.md": "https://docs.langchain.com/oss/python/langgraph/observability",
    "troubleshooting/errors/index.md": "https://docs.langchain.com/oss/python/langgraph/common-errors",
    "troubleshooting/errors/GRAPH_RECURSION_LIMIT.md": "https://docs.langchain.com/oss/python/langgraph/GRAPH_RECURSION_LIMIT",
    "troubleshooting/errors/INVALID_CONCURRENT_GRAPH_UPDATE.md": "https://docs.langchain.com/oss/python/langgraph/INVALID_CONCURRENT_GRAPH_UPDATE",
    "troubleshooting/errors/INVALID_GRAPH_NODE_RETURN_VALUE.md": "https://docs.langchain.com/oss/python/langgraph/INVALID_GRAPH_NODE_RETURN_VALUE",
    "troubleshooting/errors/MULTIPLE_SUBGRAPHS.md": "https://docs.langchain.com/oss/python/langgraph/MULTIPLE_SUBGRAPHS",
    "troubleshooting/errors/INVALID_CHAT_HISTORY.md": "https://docs.langchain.com/oss/python/langgraph/INVALID_CHAT_HISTORY",
    "troubleshooting/errors/INVALID_LICENSE.md": "https://docs.langchain.com/oss/python/langgraph/common-errors",
    "adopters.md": "https://docs.langchain.com/oss/python/langgraph/case-studies",
    "concepts/faq.md": "https://docs.langchain.com/oss/python/langgraph/overview",
    "agents/prebuilt.md": "https://docs.langchain.com/oss/python/langchain/agents",
}


class NotebookFile(File):
    def is_documentation_page(self):
        return True


def on_files(files: Files, **kwargs: Dict[str, Any]):
    if DISABLED:
        return files
    new_files = Files([])
    for file in files:
        if file.src_path.endswith(".ipynb"):
            new_file = NotebookFile(
                path=file.src_path,
                src_dir=file.src_dir,
                dest_dir=file.dest_dir,
                use_directory_urls=file.use_directory_urls,
            )
            new_files.append(new_file)
        else:
            new_files.append(file)
    return new_files


def _add_path_to_code_blocks(markdown: str, page: Page) -> str:
    """Add the path to the code blocks."""
    code_block_pattern = re.compile(
        r"(?P<indent>[ \t]*)```(?P<language>\w+)[ ]*(?P<attributes>[^\n]*)\n"
        r"(?P<code>((?:.*\n)*?))"  # Capture the code inside the block using named group
        r"(?P=indent)```"  # Match closing backticks with the same indentation
    )

    def replace_code_block_header(match: re.Match) -> str:
        indent = match.group("indent")
        language = match.group("language")
        attributes = match.group("attributes").rstrip()

        if 'exec="on"' not in attributes:
            # Return original code block
            return match.group(0)

        code = match.group("code")
        return f'{indent}```{language} {attributes} path="{page.file.src_path}"\n{code}{indent}```'

    return code_block_pattern.sub(replace_code_block_header, markdown)


# Compiled regex patterns for better performance and readability


def _apply_conditional_rendering(md_text: str, target_language: str) -> str:
    if target_language not in {"python", "js"}:
        raise ValueError("target_language must be 'python' or 'js'")

    pattern = re.compile(
        r"(?P<indent>[ \t]*):::(?P<language>\w+)\s*\n"
        r"(?P<content>((?:.*\n)*?))"  # Capture the content inside the block
        r"(?P=indent)[ \t]*:::"  # Match closing with the same indentation + any additional whitespace
    )

    def replace_conditional_blocks(match: re.Match) -> str:
        """Keep active conditionals."""
        language = match.group("language")
        content = match.group("content")

        if language not in {"python", "js"}:
            # If the language is not supported, return the original block
            return match.group(0)

        if language == target_language:
            return content

        # If the language does not match, return an empty string
        return ""

    processed = pattern.sub(replace_conditional_blocks, md_text)
    return processed


def _highlight_code_blocks(markdown: str) -> str:
    """Find code blocks with highlight comments and add hl_lines attribute.

    Args:
        markdown: The markdown content to process.

    Returns:
        updated Markdown code with code blocks containing highlight comments
        updated to use the hl_lines attribute.
    """
    # Pattern to find code blocks with highlight comments and without
    # existing hl_lines for Python and JavaScript
    # Pattern to find code blocks with highlight comments, handling optional indentation
    code_block_pattern = re.compile(
        r"(?P<indent>[ \t]*)```(?P<language>\w+)[ ]*(?P<attributes>[^\n]*)\n"
        r"(?P<code>((?:.*\n)*?))"  # Capture the code inside the block using named group
        r"(?P=indent)```"  # Match closing backticks with the same indentation
    )

    def replace_highlight_comments(match: re.Match) -> str:
        indent = match.group("indent")
        language = match.group("language")
        code_block = match.group("code")
        attributes = match.group("attributes").rstrip()

        # Account for a case where hl_lines is manually specified
        if "hl_lines" in attributes:
            # Return original code block
            return match.group(0)

        lines = code_block.split("\n")
        highlighted_lines = []

        # Skip initial empty lines
        while lines and not lines[0].strip():
            lines.pop(0)

        lines_to_keep = []

        comment_syntax = (
            "# highlight-next-line"
            if language in ["py", "python"]
            else "// highlight-next-line"
        )

        for line in lines:
            if comment_syntax in line:
                count = len(lines_to_keep) + 1
                highlighted_lines.append(str(count))
            else:
                lines_to_keep.append(line)

        # Reconstruct the new code block
        new_code_block = "\n".join(lines_to_keep)

        # Construct the full code block that also includes
        # the fenced code block syntax.
        opening_fence = f"```{language}"

        if attributes:
            opening_fence += f" {attributes}"

        if highlighted_lines:
            opening_fence += f' hl_lines="{" ".join(highlighted_lines)}"'

        return (
            # The indent and opening fence
            f"{indent}{opening_fence}\n"
            # The indent and terminating \n is already included in the code block
            f"{new_code_block}"
            f"{indent}```"
        )

    # Replace all code blocks in the markdown
    markdown = code_block_pattern.sub(replace_highlight_comments, markdown)
    return markdown


def _save_page_output(markdown: str, output_path: str):
    """Save markdown content to a file, creating parent directories if needed.

    Args:
        markdown: The markdown content to save
        output_path: The file path to save to
    """
    # Create parent directories recursively if they don't exist
    os.makedirs(os.path.dirname(output_path), exist_ok=True)
    
    # Write the markdown content to the file
    with open(output_path, "w", encoding="utf-8") as f:
        f.write(markdown)


def _on_page_markdown_with_config(
    markdown: str,
    page: Page,
    *,
    add_api_references: bool = True,
    remove_base64_images: bool = False,
    **kwargs: Any,
) -> str:
    if DISABLED:
        return markdown

    if page.file.src_path.endswith(".ipynb"):
        # logger.info("Processing Jupyter notebook: %s", page.file.src_path)
        markdown = convert_notebook(page.file.abs_src_path)

    target_language = kwargs.get(
        "target_language",
        os.environ.get("TARGET_LANGUAGE", "python")
    )

    # Apply cross-reference preprocessing to all markdown content
    markdown = _replace_autolinks(markdown, page.file.src_path, default_scope=target_language)

    # Append API reference links to code blocks
    if add_api_references:
        markdown = update_markdown_with_imports(markdown, page.file.abs_src_path)
    # Apply highlight comments to code blocks
    markdown = _highlight_code_blocks(markdown)

    # Apply conditional rendering for code blocks
    markdown = _apply_conditional_rendering(markdown, target_language)

    # Add file path as an attribute to code blocks that are executable.
    # This file path is used to associate fixtures with the executable code
    # which can be used in CI to test the docs without making network requests.
    markdown = _add_path_to_code_blocks(markdown, page)

    if remove_base64_images:
        # Remove base64 encoded images from markdown
        markdown = re.sub(r"!\[.*?\]\(data:image/[^;]+;base64,[^)]+\)", "", markdown)

    return markdown


def on_page_markdown(markdown: str, page: Page, **kwargs: Dict[str, Any]):
    finalized_markdown = _on_page_markdown_with_config(
        markdown,
        page,
        add_api_references=True,
        **kwargs,
    )
    page.meta["original_markdown"] = finalized_markdown

    output_path = os.environ.get("MD_OUTPUT_PATH")
    if output_path:
        file_path = os.path.join(output_path, page.file.src_path)
        _save_page_output(finalized_markdown, file_path)

    return finalized_markdown


# redirects

HTML_TEMPLATE = """
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>Redirecting...</title>
    <link rel="canonical" href="{url}">
    <meta name="robots" content="noindex">
    <script>var anchor=window.location.hash.substr(1);location.href="{url}"+(anchor?"#"+anchor:"")</script>
    <meta http-equiv="refresh" content="0; url={url}">
</head>
<body>
Redirecting...
</body>
</html>
"""


def _write_html(site_dir, old_path, new_path):
    """Write an HTML file in the site_dir with a meta redirect to the new page"""
    # Determine all relevant paths
    old_path_abs = os.path.join(site_dir, old_path)
    old_dir_abs = os.path.dirname(old_path_abs)

    # Create parent directories if they don't exist
    if not os.path.exists(old_dir_abs):
        os.makedirs(old_dir_abs)

    # Write the HTML redirect file in place of the old file
    content = HTML_TEMPLATE.format(url=new_path)
    with open(old_path_abs, "w", encoding="utf-8") as f:
        f.write(content)


def _inject_gtm(html: str) -> str:
    """Inject Google Tag Manager code into the HTML.

    Code to inject Google Tag Manager noscript tag immediately after <body>.

    This is done via hooks rather than via a template because the MkDocs material
    theme does not seem to allow placing the code immediately after the <body> tag
    without modifying the template files directly.

    Args:
        html: The HTML content to modify.

    Returns:
        The modified HTML content with GTM code injected.
    """
    # Code was copied from Google Tag Manager setup instructions.
    gtm_code = """
<!-- Google Tag Manager (noscript) -->
<noscript><iframe src="https://www.googletagmanager.com/ns.html?id=GTM-T35S4S46"
height="0" width="0" style="display:none;visibility:hidden"></iframe></noscript>
<!-- End Google Tag Manager (noscript) -->
"""
    soup = BeautifulSoup(html, "html.parser")
    body = soup.body
    if body:
        # Insert the GTM code as raw HTML at the top of <body>
        body.insert(0, BeautifulSoup(gtm_code, "html.parser"))
        return str(soup)
    else:
        return html  # fallback if no <body> found


def _inject_markdown_into_html(html: str, page: Page) -> str:
    """Inject the original markdown content into the HTML page as JSON."""
    original_markdown = page.meta.get("original_markdown", "")
    if not original_markdown:
        return html
    markdown_data = {
        "markdown": original_markdown,
        "title": page.title or "Page Content",
        "url": page.url or "",
    }

    # Properly escape the JSON for HTML
    json_content = json.dumps(markdown_data, ensure_ascii=False)

    json_content = (
        json_content.replace("</", "\\u003c/")
        .replace("<script", "\\u003cscript")
        .replace("</script", "\\u003c/script")
    )

    script_content = (
        f'<script id="page-markdown-content" '
        f'type="application/json">{json_content}</script>'
    )

    # Insert before </head> if it exists, otherwise before </body>
    if "</head>" not in html:
        raise ValueError(
            "HTML does not contain </head> tag. Cannot inject markdown content."
        )
    return html.replace("</head>", f"{script_content}</head>")


def on_post_page(html: str, page: Page, config: MkDocsConfig) -> str:
    """Inject Google Tag Manager noscript tag immediately after <body>.

    Args:
        html: The HTML output of the page.
        page: The page instance.
        config: The MkDocs configuration object.

    Returns:
        modified HTML output with GTM code injected.
    """
    html = _inject_markdown_into_html(html, page)
    return _inject_gtm(html)


# Create HTML files for redirects after site dir has been built
def on_post_build(config):
    use_directory_urls = config.get("use_directory_urls")
    site_dir = config["site_dir"]

    # Track which paths have explicit redirects
    redirected_paths = set()

    # Process explicit redirects from REDIRECT_MAP
    for page_old, page_new in REDIRECT_MAP.items():
        # Convert .ipynb to .md for path calculation
        page_old = page_old.replace(".ipynb", ".md")

        # Calculate the HTML path for the old page (whether it exists or not)
        if use_directory_urls:
            # With directory URLs: /path/to/page/ becomes /path/to/page/index.html
            if page_old.endswith(".md"):
                old_html_path = page_old[:-3] + "/index.html"
            else:
                old_html_path = page_old + "/index.html"
        else:
            # Without directory URLs: /path/to/page.md becomes /path/to/page.html
            if page_old.endswith(".md"):
                old_html_path = page_old[:-3] + ".html"
            else:
                old_html_path = page_old + ".html"

        # Track this path as redirected
        redirected_paths.add(old_html_path)

        if isinstance(page_new, str) and page_new.startswith("http"):
            # Handle external redirects
            _write_html(site_dir, old_html_path, page_new)
        else:
            # Handle internal redirects
            page_new = page_new.replace(".ipynb", ".md")
            page_new_before_hash, hash, suffix = page_new.partition("#")

            # Try to get the new path using File class, but fallback to manual calculation
            try:
                new_html_path = File(page_new_before_hash, "", "", True).url
                new_html_path = (
                    posixpath.relpath(new_html_path, start=posixpath.dirname(old_html_path))
                    + hash
                    + suffix
                )
            except:
                # Fallback: calculate relative path manually
                if use_directory_urls:
                    if page_new_before_hash.endswith(".md"):
                        new_html_path = page_new_before_hash[:-3] + "/"
                    else:
                        new_html_path = page_new_before_hash + "/"
                else:
                    if page_new_before_hash.endswith(".md"):
                        new_html_path = page_new_before_hash[:-3] + ".html"
                    else:
                        new_html_path = page_new_before_hash + ".html"
                new_html_path += hash + suffix

            _write_html(site_dir, old_html_path, new_html_path)

    # Create root index.html redirect
    root_redirect_html = """<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>Redirecting to LangGraph Documentation</title>
    <link rel="canonical" href="https://docs.langchain.com/oss/python/langgraph/overview">
    <meta name="robots" content="noindex">
    <script>var anchor=window.location.hash.substr(1);location.href="https://docs.langchain.com/oss/python/langgraph/overview"+(anchor?"#"+anchor:"")</script>
    <meta http-equiv="refresh" content="0; url=https://docs.langchain.com/oss/python/langgraph/overview">
</head>
<body>
<h1>Documentation has moved</h1>
<p>The LangGraph documentation has moved to <a href="https://docs.langchain.com/oss/python/langgraph/overview">docs.langchain.com</a>.</p>
<p>Redirecting you now...</p>
</body>
</html>
"""

    root_index_path = os.path.join(site_dir, "index.html")
    with open(root_index_path, "w", encoding="utf-8") as f:
        f.write(root_redirect_html)

    # Create server-side catch-all redirect file for Netlify/Cloudflare Pages
    # This handles any pages not explicitly mapped in REDIRECT_MAP
    # Note: This won't work on GitHub Pages, but kept for potential future use
    redirects_content = """# Netlify/Cloudflare Pages redirect rules
# Specific redirects are handled by individual HTML redirect pages
# This is the catch-all for any unmapped pages

# Exclude reference docs from catch-all
/reference/*  200

# Catch-all: redirect any page not explicitly mapped
/*  https://docs.langchain.com/oss/python/langgraph/overview  301
"""

    redirects_path = os.path.join(site_dir, "_redirects")
    with open(redirects_path, "w", encoding="utf-8") as f:
        f.write(redirects_content)

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/prepare_notebooks_for_ci.py
```py
"""Preprocess notebooks for CI. Currently adds VCR cassettes and optionally removes pip install cells."""

import logging
import os
import json
import click
import nbformat
import re

logger = logging.getLogger(__name__)
NOTEBOOK_DIRS = ("docs/how-tos","docs/tutorials")
DOCS_PATH = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
CASSETTES_PATH = os.path.join(DOCS_PATH, "cassettes")

BLOCKLIST_COMMANDS = (
    # skip if has WebBaseLoader to avoid caching web pages
    "WebBaseLoader",
    # skip if has draw_mermaid_png to avoid generating mermaid images via API
    "draw_mermaid_png",
)

NOTEBOOKS_NO_CASSETTES = (
    "docs/how-tos/visualization.ipynb",
)

NOTEBOOKS_NO_EXECUTION = [
    # this uses a user provided project name for langsmith
    "docs/tutorials/tnt-llm/tnt-llm.ipynb",
    # this uses langsmith datasets
    "docs/tutorials/chatbot-simulation-evaluation/langsmith-agent-simulation-evaluation.ipynb",
    # this uses browser APIs
    "docs/tutorials/web-navigation/web_voyager.ipynb",
    # these RAG guides use an ollama model
    "docs/tutorials/rag/langgraph_adaptive_rag_local.ipynb",
    "docs/tutorials/rag/langgraph_crag_local.ipynb",
    "docs/tutorials/rag/langgraph_self_rag_local.ipynb",
    # this loads a massive dataset from gcp
    "docs/tutorials/usaco/usaco.ipynb",
    # TODO: figure out why autogen notebook is not runnable (they are just hanging. possible due to code execution?)
    "docs/how-tos/autogen-integration.ipynb",
    "docs/how-tos/autogen-integration-functional.ipynb",
    # TODO: need to update these notebooks to make sure they are runnable in CI
    "docs/tutorials/storm/storm.ipynb",  # issues only when running with VCR
    "docs/tutorials/lats/lats.ipynb",  # issues only when running with VCR
    "docs/tutorials/rag/langgraph_crag.ipynb",  # flakiness from tavily
    "docs/tutorials/rag/langgraph_adaptive_rag.ipynb",  # flakiness only when running in GHA 
    "docs/tutorials/rag/langgraph_self_rag.ipynb",  # flakiness only when running in GHA
    "docs/tutorials/rag/langgraph_agentic_rag.ipynb",  # flakiness only when running in GHA
    "docs/how-tos/map-reduce.ipynb",  # flakiness from structured output, only when running with VCR
    "docs/tutorials/tot/tot.ipynb",
    "docs/how-tos/visualization.ipynb",
    "docs/how-tos/streaming-specific-nodes.ipynb",
    "docs/tutorials/llm-compiler/LLMCompiler.ipynb",
    "docs/tutorials/customer-support/customer-support.ipynb",  # relies on openai embeddings, doesn't play well w/ VCR
    "docs/how-tos/many-tools.ipynb",  # relies on openai embeddings, doesn't play well w/ VCR
]


def comment_install_cells(notebook: nbformat.NotebookNode) -> nbformat.NotebookNode:
    for cell in notebook.cells:
        if cell.cell_type != "code":
            continue

        if "pip install" in cell.source:
            # Comment out the lines in cells containing "pip install"
            cell.source = "\n".join(
                f"# {line}" if line.strip() else line
                for line in cell.source.splitlines()
            )

    return notebook


def is_magic_command(code: str) -> bool:
    return code.strip().startswith("%") or code.strip().startswith("!")


def is_comment(code: str) -> bool:
    return code.strip().startswith("#")


def has_blocklisted_command(code: str, metadata: dict) -> bool:
    if 'hide_from_vcr' in metadata:
        return True
    
    code = code.strip()
    for blocklisted_pattern in BLOCKLIST_COMMANDS:
        if blocklisted_pattern in code:
            return True
    return False

MERMAID_PATTERN = re.compile(r'display\(Image\((\w+)\.get_graph\(\)\.draw_mermaid_png\(\)\)\)')

def remove_mermaid(code: str) -> str:
    return MERMAID_PATTERN.sub('print()', code)


def add_vcr_to_notebook(
    notebook: nbformat.NotebookNode, cassette_prefix: str
) -> nbformat.NotebookNode:
    """Inject `with vcr.cassette` into each code cell of the notebook."""

    uses_langsmith = False
    # Inject VCR context manager into each code cell
    for idx, cell in enumerate(notebook.cells):
        if cell.cell_type != "code":
            continue

        lines = cell.source.splitlines()
        # remove the special tag for hidden cells
        lines = [line for line in lines if not line.strip().startswith("# hide-cell")]
        # skip if empty cell
        if not lines:
            continue

        are_magic_lines = [is_magic_command(line) for line in lines]

        # skip if all magic
        if all(are_magic_lines):
            continue

        if any(are_magic_lines):
            raise ValueError(
                "Cannot process code cells with mixed magic and non-magic code."
            )

        # skip if just comments
        if all(is_comment(line) or not line.strip() for line in lines):
            continue

        if has_blocklisted_command(cell.source, cell.metadata):
            continue

        cell_id = cell.get("id", idx)
        cassette_name = f"{cassette_prefix}_{cell_id}.msgpack.zlib"
        cell.source = f"with custom_vcr.use_cassette('{cassette_name}', filter_headers=['x-api-key', 'authorization'], record_mode='once', serializer='advanced_compressed'):\n" + "\n".join(
            f"    {line}" for line in lines
        )

        if any("hub.pull" in line or "from langsmith import" in line for line in lines):
            uses_langsmith = True

    # Add import statement
    vcr_import_lines = []
    if uses_langsmith:
        vcr_import_lines.extend([
            # patch urllib3 to handle vcr errors, see more here:
            # https://github.com/langchain-ai/langsmith-sdk/blob/main/python/langsmith/_internal/_patch.py
            "import sys",
            f"sys.path.insert(0, '{os.path.join(DOCS_PATH, '_scripts')}')",
            "import _patch as patch_urllib3",
            "patch_urllib3.patch_urllib3()",
        ])

    vcr_import_lines.extend([
        "import nest_asyncio",
        "nest_asyncio.apply()",
        "import vcr",
        "import msgpack",
        "import base64",
        "import zlib",
        "import os",
        "os.environ.pop(\"LANGCHAIN_TRACING_V2\", None)",
        "custom_vcr = vcr.VCR()",
        "",
        "def compress_data(data, compression_level=9):",
        "    packed = msgpack.packb(data, use_bin_type=True)",
        "    compressed = zlib.compress(packed, level=compression_level)",
        "    return base64.b64encode(compressed).decode('utf-8')",
        "",
        "def decompress_data(compressed_string):",
        "    decoded = base64.b64decode(compressed_string)",
        "    decompressed = zlib.decompress(decoded)",
        "    return msgpack.unpackb(decompressed, raw=False)",
        "",
        "class AdvancedCompressedSerializer:",
        "    def serialize(self, cassette_dict):",
        "        return compress_data(cassette_dict)",
        "",
        "    def deserialize(self, cassette_string):",
        "        return decompress_data(cassette_string)",
        "",
        "custom_vcr.register_serializer('advanced_compressed', AdvancedCompressedSerializer())",
        "custom_vcr.serializer = 'advanced_compressed'",
    ])

    import_cell = nbformat.v4.new_code_cell(source="\n".join(vcr_import_lines))
    import_cell.pop("id", None)
    notebook.cells.insert(0, import_cell)
    return notebook


def remove_mermaid_from_notebook(notebook: nbformat.NotebookNode) -> nbformat.NotebookNode:
    for cell in notebook.cells:
        if cell.cell_type != "code":
            continue

        cell.source = remove_mermaid(cell.source)

        # skip the cell entirely if it contains PYPPETEER
        if "PYPPETEER" in cell.source:
            cell.source = ""

    return notebook


def process_notebooks(should_comment_install_cells: bool) -> None:
    for directory in NOTEBOOK_DIRS:
        for root, _, files in os.walk(directory):
            for file in files:
                if not file.endswith(".ipynb") or "ipynb_checkpoints" in root:
                    continue

                notebook_path = os.path.join(root, file)
                try:
                    notebook = nbformat.read(notebook_path, as_version=4)

                    if should_comment_install_cells:
                        notebook = comment_install_cells(notebook)

                    base_filename = os.path.splitext(os.path.basename(file))[0]
                    cassette_prefix = os.path.join(CASSETTES_PATH, base_filename)
                    if notebook_path not in NOTEBOOKS_NO_CASSETTES:
                        notebook = add_vcr_to_notebook(
                            notebook, cassette_prefix=cassette_prefix
                        )

                    notebook = remove_mermaid_from_notebook(notebook)

                    if notebook_path in NOTEBOOKS_NO_EXECUTION:
                        # Add a cell at the beginning to indicate that this notebook should not be executed
                        warning_cell = nbformat.v4.new_markdown_cell(
                            source="**Warning:** This notebook is not meant to be executed automatically."
                        )
                        notebook.cells.insert(0, warning_cell)

                        # Add a special tag to the first code cell
                        if notebook.cells and notebook.cells[1].cell_type == "code":
                            notebook.cells[1].metadata["tags"] = notebook.cells[1].metadata.get("tags", []) + ["no_execution"]

                    nbformat.write(notebook, notebook_path)
                    logger.info(f"Processed: {notebook_path}")
                except Exception as e:
                    logger.error(f"Error processing {notebook_path}: {e}")
    
    with open("notebooks_no_execution.json", "w") as f:
        json.dump(NOTEBOOKS_NO_EXECUTION, f)


@click.command()
@click.option(
    "--comment-install-cells",
    is_flag=True,
    default=False,
    help="Whether to comment out install cells",
)
def main(comment_install_cells):
    process_notebooks(should_comment_install_cells=comment_install_cells)
    logger.info("All notebooks processed successfully.")


if __name__ == "__main__":
    main()

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/notebook_convert_templates/mdoutput/conf.json
```json
{
  "mimetypes": {
    "text/markdown": true
  }
}
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/notebook_convert_templates/mdoutput/index.md.j2
```j2
{% extends 'markdown/index.md.j2' %}

{% block input %}{# cell.metadata.language is an addition of our docs pipeline. #}
```{%- if 'language' in cell.metadata -%}
{{ cell.metadata.language }}
{%- elif 'magics_language' in cell.metadata -%}
{{ cell.metadata.magics_language }}
{%- elif 'name' in nb.metadata.get('language_info', {}) -%}
{{ nb.metadata.language_info.name }}
{%- endif %}
{{ cell.source }}
```
{% endblock input %}


{%- block traceback_line -%}
```output
{{ line.rstrip() | strip_ansi }}
```
{%- endblock traceback_line -%}

{%- block stream -%}
```output
{{ output.text.rstrip() | strip_ansi }}
```
{%- endblock stream -%}

{%- block data_text scoped -%}
```output
{{ output.data['text/plain'].rstrip() | strip_ansi }}
```
{%- endblock data_text -%}

{%- block data_html scoped -%}
```html
{{ output.data['text/html'] | safe }} 
```
{%- endblock data_html -%}

{%- block data_jpg scoped -%}
<p>
<img src="data:image/jpg;base64,{{ output.data['image/jpeg'] }}" />
</p>
{%- endblock data_jpg -%}

{%- block data_png scoped -%}
<p>
<img src="data:image/png;base64,{{ output.data['image/png'] }}" />
</p>
{%- endblock data_png -%}

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/js_translation/add_translation.py
```py
"""Translate Python markdown to TypeScript and/or consolidate Python-JS markdown into a single document."""

import argparse

import requests
from langchain_anthropic import ChatAnthropic
from textwrap import dedent


# Load reference TypeScript snippets
URL = "https://gist.githubusercontent.com/dqbd/b35d49e2ceec80e654fe1c5ab61ec477/raw/f4768aeedb67628190a4e06d063a938afc8e7672/snippets.md"
response = requests.get(URL)
response.raise_for_status()
reference_snippets = response.text

# Initialize model
model = ChatAnthropic(model="claude-sonnet-4-0", max_tokens=64_000)


FLUENT_INTERFACE_PROMPT = (
    "CRITICAL: Always use method chaining (fluent interface) for StateGraph operations in TypeScript. "
    "Never create separate variables for the graph builder or call methods individually. "
    "The fluent interface provides better type safety and is the preferred pattern.\n\n"
    "CORRECT examples with fluent interface:\n"
    + dedent(
        """
        ```typescript
        const graph = new StateGraph(MyState)
          .addNode('node1', node1)
          .addNode('node2', node2)
          .addEdge(START, 'node1')
          .addEdge('node1', 'node2')
          .addEdge('node2', END)
          .compile()
        ```
        
        ```typescript
        const graph = new StateGraph(MyState)
          .addNode('chatbot', chatbot)
          .addEdge(START, 'chatbot')
          .addEdge('chatbot', END)
          .compile()
        ```
        
        ```typescript
        const graph = new StateGraph(MyState)
          .addNode('chatbot', chatbot)
          .addEdge(START, 'chatbot')
          .addEdge('chatbot', END)
          .compile()
        ```
        """
    )
    + "\n"
    + "INCORRECT examples to avoid:\n"
    + dedent(
        """
        ```typescript
        // WRONG: Creating separate builder variable
        const graphBuilder = new StateGraph(MyState)
        graphBuilder.addNode('node1', node1)
        graphBuilder.addEdge(START, 'node1')
        const graph = graphBuilder.compile()
        ```
        
        ```typescript
        // WRONG: Using Python-style method names
        const workflow = new StateGraph(MyState)
        workflow.add_node('node1', node1)
        workflow.add_edge(START, 'node1')
        const graph = workflow.compile()
        ```
        
        ```typescript
        // WRONG: Calling methods individually
        const graphBuilder = new StateGraph(MyState)
        graphBuilder.addNode('chatbot', chatbot)
        graphBuilder.addEdge(START, 'chatbot')
        graphBuilder.addEdge('chatbot', END)
        const graph = graphBuilder.compile()
        ```
        """
    )
    + "\n"
    + "Key rules:\n"
    + "- Always chain methods directly on the StateGraph constructor\n"
    + "- Use camelCase method names (addNode, addEdge, not add_node, add_edge)\n"
    + "- Always end with .compile()\n"
    + "- Never store the builder in a separate variable\n"
)


TRANSLATION_PROMPT = (
    "You are a helpful assistant that translates Python-based technical "
    "documentation written in Markdown to equivalent TypeScript-based documentation. "
    "The input is a Markdown file written in mkdocs format. It contains "
    "Python code snippets embedded in prose. "
    "Your task is to rewrite the content by translating the Python code to "
    "idiomatic TypeScript, using the provided TypeScript reference snippets "
    "to ensure accurate and consistent usage (e.g., correct imports, function "
    "names, and patterns). "
    "Remove the original Python code and replace it with the corresponding "
    "TypeScript version. "
    "Do not alter the surrounding prose unless a change is necessary to "
    "reflect differences between Python and TypeScript. "
    "Preserve the structure and formatting of the original Markdown document. "
    "Do not make stylistic or structural changes unless they directly support "
    "the translation. "
    "Use the reference TypeScript snippets as guidance whenever possible to "
    "maintain alignment with existing conventions.\n\n"
    "IMPORTANT REQUIREMENTS:\n"
    "- Use Zod for state definition for StateGraph. Avoid using Annotation since it will be deprecated in the future.\n"
    "- ALWAYS use fluent interface (method chaining) for StateGraph operations - this is CRITICAL\n"
    "- Never create separate variables for graph builders\n"
    "- Always chain methods directly on the StateGraph constructor and end with .compile()\n\n"
    f"{FLUENT_INTERFACE_PROMPT}\n\n"
    f"Here are the reference TypeScript snippets:\n\n{reference_snippets}\n\n"
)

CONSOLIDATION_PROMPT = (
    "You are a helpful assistant that consolidates parallel Python and JavaScript (TypeScript) technical documentation "
    "written in Markdown into a single unified Markdown document. "
    "The input consists of two documents: the first is for Python users, and the second is for JavaScript/TypeScript users. "
    "Your task is to merge these into one Markdown file using language-specific fenced blocks to separate the content where needed. "
    "Use the following syntax to distinguish content for each language:\n\n"
    ":::python\n"
    "# Python-specific content\n"
    ":::\n\n"
    ":::js\n"
    "# JavaScript/TypeScript-specific content\n"
    ":::\n\n"
    "Follow these consolidation rules:\n"
    "- When content (prose or code) is the same or nearly identical in both versions, include it only once—outside of any fenced block.\n"
    "- When content differs between the Python and JS versions, wrap each version in its corresponding fenced block.\n"
    "- Prefer **paragraph-level separation** of language-specific content. Do not combine Python and JS snippets or terminology in the same sentence or paragraph using conditional phrases.\n"
    "  For example, avoid inline constructs like:\n"
    "  `The :::python add_messages ::: :::js reducer ::: function...`\n"
    "  Instead, write two distinct paragraphs:\n\n"
    "  :::python\n"
    "  The `add_messages` function in our `State` will append the LLM's response messages to whatever messages are already in the state.\n"
    "  ::: \n\n"
    "  :::js\n"
    "  The `reducer` function in our `StateAnnotation` will append the LLM's response messages to whatever messages are already in the state.\n"
    "  :::\n\n"
    "- Preserve the overall structure, ordering, and formatting of the original Markdown documents.\n"
    "- Do not rephrase or unify content unless it is logically and semantically identical.\n"
    "- Use the fenced blocks for both prose and code as needed, and ensure output is clean, readable Markdown suitable for tools that parse these directives.\n"
    "Your goal is to produce a cleanly merged documentation file that serves both Python and JavaScript users without redundancy, while maximizing clarity and separation of language-specific details."
)


def translate_python_to_ts(markdown_content: str) -> str:
    response = model.invoke(
        [
            {
                "role": "system",
                "content": TRANSLATION_PROMPT,
                "cache_control": {"type": "ephemeral"},
            },
            {"role": "user", "content": markdown_content},
        ]
    )
    return response.content


def consolidate_python_and_ts(combined_content: str) -> str:
    response = model.invoke(
        [
            {
                "role": "system",
                "content": CONSOLIDATION_PROMPT,
                "cache_control": {"type": "ephemeral"},
            },
            {"role": "user", "content": combined_content},
        ]
    )
    return response.content


def main(file_path: str, translate_only: bool, consolidate_only: bool) -> None:
    with open(file_path, "r", encoding="utf-8") as f:
        markdown_content = f.read()

    if translate_only:
        translated = translate_python_to_ts(markdown_content)
        output_path = file_path.replace(".md", ".translated.md")
        with open(output_path, "w", encoding="utf-8") as f:
            f.write(translated)
        print(f"Translated JS/TS version written to: {output_path}")

    elif consolidate_only:
        consolidated = consolidate_python_and_ts(markdown_content)
        with open(file_path, "w", encoding="utf-8") as f:
            f.write(consolidated)
        print(f"Consolidated content written to: {file_path}")

    else:
        # Default behavior: translate first, then consolidate both
        translated = translate_python_to_ts(markdown_content)
        combined = f"{markdown_content.strip()}\n\n\n{translated.strip()}"
        consolidated = consolidate_python_and_ts(combined)
        with open(file_path, "w", encoding="utf-8") as f:
            f.write(consolidated)
        print(f"Translated and consolidated content written to: {file_path}")


if __name__ == "__main__":
    parser = argparse.ArgumentParser(
        description=(
            "Translate Python markdown to TypeScript and/or consolidate "
            "Python-JS markdown into one file."
        )
    )
    parser.add_argument("file_path", type=str, help="Path to the markdown file.")
    parser.add_argument(
        "--translate-only",
        action="store_true",
        help="Only generate the JS translation.",
    )
    parser.add_argument(
        "--consolidate-only",
        action="store_true",
        help="Only consolidate pre-paired Python and JS content.",
    )
    args = parser.parse_args()

    if args.translate_only and args.consolidate_only:
        raise ValueError(
            "Cannot use both --translate-only and --consolidate-only at the same time."
        )

    main(
        args.file_path,
        translate_only=args.translate_only,
        consolidate_only=args.consolidate_only,
    )

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/js_translation/extract_codeblocks.py
```py
#!/usr/bin/env python
"""Extracts typescript code blocks from a markdown file."""

import argparse
import json
import re
import os
from typing import List, TypedDict, Literal


class CodeBlock(TypedDict):
    """A code block extracted from a markdown file."""
    starting_line: int
    """The line number where the code block starts in the source file"""
    ending_line: int
    """The line number where the code block ends in the source file"""
    indentation: int
    """Number of spaces/tabs used for indentation of the code block"""
    source_file: str
    """Path to the markdown file containing this code block"""
    frontmatter: str
    """Any metadata or frontmatter specified after the opening code fence"""
    code: str
    """The actual code content within the code block"""
    language: str
    """The language of the code block (e.g. typescript, javascript)"""


def extract_code_blocks(markdown_content: str, source_file: str) -> List[CodeBlock]:
    """Extracts code blocks from a markdown file.

    Args:
        markdown_content: The content of the markdown file.
        source_file: The path to the markdown file.

    Returns:
        A list of TypedDicts, where each dict represents a code block.
    """
    # Regex to find code blocks with specified languages, capturing indentation
    # and frontmatter.
    pattern = re.compile(
        r"^(?P<indentation>\s*)```(?P<language>typescript|javascript|ts|js)(?P<frontmatter>[^\n]*)\n(?P<code>.*?)\n^(?P=indentation)```\s*$",
        re.DOTALL | re.MULTILINE,
    )

    code_blocks: List[CodeBlock] = []
    for match in pattern.finditer(markdown_content):
        start_pos = match.start()

        # Calculate line numbers
        starting_line = markdown_content.count("\n", 0, start_pos) + 1
        ending_line = starting_line + match.group(0).count("\n")

        indentation_str = match.group("indentation")

        code_block: CodeBlock = {
            "starting_line": starting_line,
            "ending_line": ending_line,
            "indentation": len(indentation_str),
            "source_file": source_file,
            "frontmatter": match.group("frontmatter").strip(),
            "code": match.group("code"),
            "language": match.group("language"),
        }
        code_blocks.append(code_block)

    return code_blocks

def dump_code_blocks(input_file: str, output_file: str, format: Literal["json", "inline"]) -> None:
    """Function to extract and save code blocks from a markdown file.

    Args:
        input_file: Path to the input markdown file.
        output_file: Path to the output JSON file for the extracted code blocks.
        format: Output format - either "json" or "inline"
    """
    with open(input_file, "r", encoding="utf-8") as f:
        markdown_content = f.read()

    extracted_code = extract_code_blocks(markdown_content, input_file)

    if len(extracted_code) == 0:
        print(f"No code blocks found in {input_file}")
        return
    
    if format == "json":
        with open(output_file, "w", encoding="utf-8") as f:
            json.dump(extracted_code, f, indent=2)
    elif format == "inline":
        with open(output_file, "w", encoding="utf-8") as f:
            for code_block in extracted_code:
                f.write(f"// {json.dumps({k:v for k,v in code_block.items() if k != 'code'})}\n")
                f.write("\n")
                f.write(code_block["code"])
                f.write("\n")
    print(f"Extracted {len(extracted_code)} code blocks from {input_file} to {output_file}")

def main(input_path: str, output_path: str, format: Literal["json", "inline"]) -> None:
    """Main function to extract code blocks from a markdown file.

    Args:
        input_file: Path to the input markdown file.
        output_file: Path to the output JSON file for the extracted code blocks.
        format: Output format - either "json" or "inline"
    """
    # Check if input path is a directory
    if os.path.isdir(input_path):
        if os.path.isfile(output_path):
            raise ValueError("If input_path is a directory, output_path must also be a directory")
        if not os.path.isdir(output_path):
            os.makedirs(output_path, exist_ok=True)
        
        # Process each markdown file in the directory recursively
        for root, _, files in os.walk(input_path):
            for filename in files:
                if filename.endswith(".md"):
                    # Get relative path to maintain directory structure
                    rel_path = os.path.relpath(root, input_path)
                    input_file = os.path.join(root, filename)
                    # Create output directory if it doesn't exist
                    output_dir = os.path.join(output_path, rel_path)
                    os.makedirs(output_dir, exist_ok=True)
                    output_file = os.path.join(output_dir, filename.replace(".md", ".ts"))
                    dump_code_blocks(input_file, output_file, format)
    else:
        # Process single file
        dump_code_blocks(input_path, output_path, format)


if __name__ == "__main__":
    parser = argparse.ArgumentParser(
        description="Extract typescript code blocks from a markdown file."
    )
    parser.add_argument(
        "input_file",
        help="Path to the input markdown file.",
    )
    parser.add_argument(
        "output_file",
        help="Path to the output JSON file for the extracted code blocks.",
    )
    parser.add_argument(
        "--format",
        choices=["json", "inline"],
        default="json",
        help="Output format - either 'json' or 'inline'",
    )
    args = parser.parse_args()

    main(args.input_file, args.output_file, args.format)

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/js_translation/codeblocks/.gitkeep
```text
.prettierrc
.eslint.config.mjs
package.json
README.md
tsconfig.json
yarn.lock
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/js_translation/codeblocks/.prettierrc
```text
{
  "$schema": "https://json.schemastore.org/prettierrc",
  "printWidth": 80,
  "tabWidth": 2,
  "useTabs": false,
  "semi": true,
  "singleQuote": false,
  "quoteProps": "as-needed",
  "jsxSingleQuote": false,
  "trailingComma": "es5",
  "bracketSpacing": true,
  "arrowParens": "always",
  "requirePragma": false,
  "insertPragma": false,
  "proseWrap": "preserve",
  "htmlWhitespaceSensitivity": "css",
  "vueIndentScriptAndStyle": false,
  "endOfLine": "lf"
}
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/js_translation/codeblocks/README.md
```md
# \_codeblocks

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/js_translation/codeblocks/eslint.config.mjs
```mjs
import js from "@eslint/js";
import globals from "globals";
import tseslint from "typescript-eslint";
import { defineConfig } from "eslint/config";

export default defineConfig([
  {
    files: ["**/*.{js,mjs,cjs,ts,mts,cts}"],
    plugins: { js },
    extends: ["js/recommended"],
    languageOptions: { globals: globals.browser },
  },
  tseslint.configs.recommended,
]);

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/js_translation/codeblocks/package.json
```json
{
  "name": "_codeblocks",
  "packageManager": "yarn@4.6.0",
  "scripts": {
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write .",
    "format:fix": "prettier --write . --fix"
  },
  "dependencies": {
    "@langchain/anthropic": "^0.3.24",
    "@langchain/core": "^0.3.66",
    "@langchain/langgraph": "^0.3.11",
    "@langchain/langgraph-api": "^0.0.52",
    "@langchain/langgraph-sdk": "^0.0.102",
    "@langchain/openai": "^0.6.3",
    "zod": "^4.0.10"
  },
  "devDependencies": {
    "@eslint/js": "^9.32.0",
    "eslint": "^9.32.0",
    "globals": "^16.3.0",
    "jiti": "^2.5.1",
    "typescript": "^5.8.3",
    "typescript-eslint": "^8.38.0"
  }
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/js_translation/codeblocks/tsconfig.json
```json
{
  "compilerOptions": {
    /* Visit https://aka.ms/tsconfig to read more about this file */

    /* Projects */
    // "incremental": true,                              /* Save .tsbuildinfo files to allow for incremental compilation of projects. */
    // "composite": true,                                /* Enable constraints that allow a TypeScript project to be used with project references. */
    // "tsBuildInfoFile": "./.tsbuildinfo",              /* Specify the path to .tsbuildinfo incremental compilation file. */
    // "disableSourceOfProjectReferenceRedirect": true,  /* Disable preferring source files instead of declaration files when referencing composite projects. */
    // "disableSolutionSearching": true,                 /* Opt a project out of multi-project reference checking when editing. */
    // "disableReferencedProjectLoad": true,             /* Reduce the number of projects loaded automatically by TypeScript. */

    /* Language and Environment */
    "target": "esnext",                                  /* Set the JavaScript language version for emitted JavaScript and include compatible library declarations. */
    // "lib": [],                                        /* Specify a set of bundled library declaration files that describe the target runtime environment. */
    // "jsx": "preserve",                                /* Specify what JSX code is generated. */
    // "libReplacement": true,                           /* Enable lib replacement. */
    // "experimentalDecorators": true,                   /* Enable experimental support for legacy experimental decorators. */
    // "emitDecoratorMetadata": true,                    /* Emit design-type metadata for decorated declarations in source files. */
    // "jsxFactory": "",                                 /* Specify the JSX factory function used when targeting React JSX emit, e.g. 'React.createElement' or 'h'. */
    // "jsxFragmentFactory": "",                         /* Specify the JSX Fragment reference used for fragments when targeting React JSX emit e.g. 'React.Fragment' or 'Fragment'. */
    // "jsxImportSource": "",                            /* Specify module specifier used to import the JSX factory functions when using 'jsx: react-jsx*'. */
    // "reactNamespace": "",                             /* Specify the object invoked for 'createElement'. This only applies when targeting 'react' JSX emit. */
    // "noLib": true,                                    /* Disable including any library files, including the default lib.d.ts. */
    // "useDefineForClassFields": true,                  /* Emit ECMAScript-standard-compliant class fields. */
    // "moduleDetection": "auto",                        /* Control what method is used to detect module-format JS files. */

    /* Modules */
    "module": "nodenext",                                /* Specify what module code is generated. */
    // "rootDir": "./",                                  /* Specify the root folder within your source files. */
    "moduleResolution": "nodenext",                     /* Specify how TypeScript looks up a file from a given module specifier. */
    // "baseUrl": "./",                                  /* Specify the base directory to resolve non-relative module names. */
    // "paths": {},                                      /* Specify a set of entries that re-map imports to additional lookup locations. */
    // "rootDirs": [],                                   /* Allow multiple folders to be treated as one when resolving modules. */
    // "typeRoots": [],                                  /* Specify multiple folders that act like './node_modules/@types'. */
    // "types": [],                                      /* Specify type package names to be included without being referenced in a source file. */
    // "allowUmdGlobalAccess": true,                     /* Allow accessing UMD globals from modules. */
    // "moduleSuffixes": [],                             /* List of file name suffixes to search when resolving a module. */
    // "allowImportingTsExtensions": true,               /* Allow imports to include TypeScript file extensions. Requires '--moduleResolution bundler' and either '--noEmit' or '--emitDeclarationOnly' to be set. */
    // "rewriteRelativeImportExtensions": true,          /* Rewrite '.ts', '.tsx', '.mts', and '.cts' file extensions in relative import paths to their JavaScript equivalent in output files. */
    // "resolvePackageJsonExports": true,                /* Use the package.json 'exports' field when resolving package imports. */
    // "resolvePackageJsonImports": true,                /* Use the package.json 'imports' field when resolving imports. */
    // "customConditions": [],                           /* Conditions to set in addition to the resolver-specific defaults when resolving imports. */
    // "noUncheckedSideEffectImports": true,             /* Check side effect imports. */
    // "resolveJsonModule": true,                        /* Enable importing .json files. */
    // "allowArbitraryExtensions": true,                 /* Enable importing files with any extension, provided a declaration file is present. */
    // "noResolve": true,                                /* Disallow 'import's, 'require's or '<reference>'s from expanding the number of files TypeScript should add to a project. */

    /* JavaScript Support */
    // "allowJs": true,                                  /* Allow JavaScript files to be a part of your program. Use the 'checkJS' option to get errors from these files. */
    // "checkJs": true,                                  /* Enable error reporting in type-checked JavaScript files. */
    // "maxNodeModuleJsDepth": 1,                        /* Specify the maximum folder depth used for checking JavaScript files from 'node_modules'. Only applicable with 'allowJs'. */

    /* Emit */
    // "declaration": true,                              /* Generate .d.ts files from TypeScript and JavaScript files in your project. */
    // "declarationMap": true,                           /* Create sourcemaps for d.ts files. */
    // "emitDeclarationOnly": true,                      /* Only output d.ts files and not JavaScript files. */
    // "sourceMap": true,                                /* Create source map files for emitted JavaScript files. */
    // "inlineSourceMap": true,                          /* Include sourcemap files inside the emitted JavaScript. */
    // "noEmit": true,                                   /* Disable emitting files from a compilation. */
    // "outFile": "./",                                  /* Specify a file that bundles all outputs into one JavaScript file. If 'declaration' is true, also designates a file that bundles all .d.ts output. */
    // "outDir": "./",                                   /* Specify an output folder for all emitted files. */
    // "removeComments": true,                           /* Disable emitting comments. */
    // "importHelpers": true,                            /* Allow importing helper functions from tslib once per project, instead of including them per-file. */
    // "downlevelIteration": true,                       /* Emit more compliant, but verbose and less performant JavaScript for iteration. */
    // "sourceRoot": "",                                 /* Specify the root path for debuggers to find the reference source code. */
    // "mapRoot": "",                                    /* Specify the location where debugger should locate map files instead of generated locations. */
    // "inlineSources": true,                            /* Include source code in the sourcemaps inside the emitted JavaScript. */
    // "emitBOM": true,                                  /* Emit a UTF-8 Byte Order Mark (BOM) in the beginning of output files. */
    // "newLine": "crlf",                                /* Set the newline character for emitting files. */
    // "stripInternal": true,                            /* Disable emitting declarations that have '@internal' in their JSDoc comments. */
    // "noEmitHelpers": true,                            /* Disable generating custom helper functions like '__extends' in compiled output. */
    // "noEmitOnError": true,                            /* Disable emitting files if any type checking errors are reported. */
    // "preserveConstEnums": true,                       /* Disable erasing 'const enum' declarations in generated code. */
    // "declarationDir": "./",                           /* Specify the output directory for generated declaration files. */

    /* Interop Constraints */
    // "isolatedModules": true,                          /* Ensure that each file can be safely transpiled without relying on other imports. */
    // "verbatimModuleSyntax": true,                     /* Do not transform or elide any imports or exports not marked as type-only, ensuring they are written in the output file's format based on the 'module' setting. */
    // "isolatedDeclarations": true,                     /* Require sufficient annotation on exports so other tools can trivially generate declaration files. */
    // "erasableSyntaxOnly": true,                       /* Do not allow runtime constructs that are not part of ECMAScript. */
    // "allowSyntheticDefaultImports": true,             /* Allow 'import x from y' when a module doesn't have a default export. */
    "esModuleInterop": true,                             /* Emit additional JavaScript to ease support for importing CommonJS modules. This enables 'allowSyntheticDefaultImports' for type compatibility. */
    // "preserveSymlinks": true,                         /* Disable resolving symlinks to their realpath. This correlates to the same flag in node. */
    "forceConsistentCasingInFileNames": true,            /* Ensure that casing is correct in imports. */

    /* Type Checking */
    "strict": false,                                      /* Enable all strict type-checking options. */
    // "noImplicitAny": true,                            /* Enable error reporting for expressions and declarations with an implied 'any' type. */
    // "strictNullChecks": true,                         /* When type checking, take into account 'null' and 'undefined'. */
    // "strictFunctionTypes": true,                      /* When assigning functions, check to ensure parameters and the return values are subtype-compatible. */
    // "strictBindCallApply": true,                      /* Check that the arguments for 'bind', 'call', and 'apply' methods match the original function. */
    // "strictPropertyInitialization": true,             /* Check for class properties that are declared but not set in the constructor. */
    // "strictBuiltinIteratorReturn": true,              /* Built-in iterators are instantiated with a 'TReturn' type of 'undefined' instead of 'any'. */
    // "noImplicitThis": true,                           /* Enable error reporting when 'this' is given the type 'any'. */
    // "useUnknownInCatchVariables": true,               /* Default catch clause variables as 'unknown' instead of 'any'. */
    // "alwaysStrict": true,                             /* Ensure 'use strict' is always emitted. */
    // "noUnusedLocals": true,                           /* Enable error reporting when local variables aren't read. */
    // "noUnusedParameters": true,                       /* Raise an error when a function parameter isn't read. */
    // "exactOptionalPropertyTypes": true,               /* Interpret optional property types as written, rather than adding 'undefined'. */
    // "noImplicitReturns": true,                        /* Enable error reporting for codepaths that do not explicitly return in a function. */
    // "noFallthroughCasesInSwitch": true,               /* Enable error reporting for fallthrough cases in switch statements. */
    // "noUncheckedIndexedAccess": true,                 /* Add 'undefined' to a type when accessed using an index. */
    // "noImplicitOverride": true,                       /* Ensure overriding members in derived classes are marked with an override modifier. */
    // "noPropertyAccessFromIndexSignature": true,       /* Enforces using indexed accessors for keys declared using an indexed type. */
    // "allowUnusedLabels": true,                        /* Disable error reporting for unused labels. */
    // "allowUnreachableCode": true,                     /* Disable error reporting for unreachable code. */

    /* Completeness */
    // "skipDefaultLibCheck": true,                      /* Skip type checking .d.ts files that are included with TypeScript. */
    "skipLibCheck": true                                 /* Skip type checking all .d.ts files. */
    ""
  }
}

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/js_translation/codeblocks/yarn.lock
```lock
# This file is generated by running "yarn install" inside your project.
# Manual changes might be lost - proceed with caution!

__metadata:
  version: 8
  cacheKey: 10c0

"@alloc/quick-lru@npm:^5.2.0":
  version: 5.2.0
  resolution: "@alloc/quick-lru@npm:5.2.0"
  checksum: 10c0/7b878c48b9d25277d0e1a9b8b2f2312a314af806b4129dc902f2bc29ab09b58236e53964689feec187b28c80d2203aff03829754773a707a8a5987f1b7682d92
  languageName: node
  linkType: hard

"@ampproject/remapping@npm:^2.3.0":
  version: 2.3.0
  resolution: "@ampproject/remapping@npm:2.3.0"
  dependencies:
    "@jridgewell/gen-mapping": "npm:^0.3.5"
    "@jridgewell/trace-mapping": "npm:^0.3.24"
  checksum: 10c0/81d63cca5443e0f0c72ae18b544cc28c7c0ec2cea46e7cb888bb0e0f411a1191d0d6b7af798d54e30777d8d1488b2ec0732aac2be342d3d7d3ffd271c6f489ed
  languageName: node
  linkType: hard

"@anthropic-ai/sdk@npm:^0.56.0":
  version: 0.56.0
  resolution: "@anthropic-ai/sdk@npm:0.56.0"
  bin:
    anthropic-ai-sdk: bin/cli
  checksum: 10c0/b8506daa740b3700c56cf7e7cd16c5f3c092b96ad0bca893530d3e12ed543bdae174ea5e34b270ba86958f193a8ac31559f17ed4a79ba9219771d3c457e15c06
  languageName: node
  linkType: hard

"@babel/code-frame@npm:^7.26.2":
  version: 7.27.1
  resolution: "@babel/code-frame@npm:7.27.1"
  dependencies:
    "@babel/helper-validator-identifier": "npm:^7.27.1"
    js-tokens: "npm:^4.0.0"
    picocolors: "npm:^1.1.1"
  checksum: 10c0/5dd9a18baa5fce4741ba729acc3a3272c49c25cb8736c4b18e113099520e7ef7b545a4096a26d600e4416157e63e87d66db46aa3fbf0a5f2286da2705c12da00
  languageName: node
  linkType: hard

"@babel/helper-validator-identifier@npm:^7.27.1":
  version: 7.27.1
  resolution: "@babel/helper-validator-identifier@npm:7.27.1"
  checksum: 10c0/c558f11c4871d526498e49d07a84752d1800bf72ac0d3dad100309a2eaba24efbf56ea59af5137ff15e3a00280ebe588560534b0e894a4750f8b1411d8f78b84
  languageName: node
  linkType: hard

"@cfworker/json-schema@npm:^4.0.2":
  version: 4.1.1
  resolution: "@cfworker/json-schema@npm:4.1.1"
  checksum: 10c0/b5253486d346b7de6feec9c73954f612b11019dacb9023d710a5666df2f5fc145dd88b6b913c88726c6d97e2e258a515fa2cab177f58b18da6bac3738cbc4739
  languageName: node
  linkType: hard

"@colors/colors@npm:1.6.0, @colors/colors@npm:^1.6.0":
  version: 1.6.0
  resolution: "@colors/colors@npm:1.6.0"
  checksum: 10c0/9328a0778a5b0db243af54455b79a69e3fb21122d6c15ef9e9fcc94881d8d17352d8b2b2590f9bdd46fac5c2d6c1636dcfc14358a20c70e22daf89e1a759b629
  languageName: node
  linkType: hard

"@commander-js/extra-typings@npm:^13.0.0":
  version: 13.1.0
  resolution: "@commander-js/extra-typings@npm:13.1.0"
  peerDependencies:
    commander: ~13.1.0
  checksum: 10c0/ff799f0641f68855aa73c976912a607c25d564df34fd8e262927a80b19f6cccd882fe7ce098a0e072a497fd0020cbd19fd1e4d5cc98a461cd6623abf8ed5f4e7
  languageName: node
  linkType: hard

"@dabh/diagnostics@npm:^2.0.2":
  version: 2.0.3
  resolution: "@dabh/diagnostics@npm:2.0.3"
  dependencies:
    colorspace: "npm:1.1.x"
    enabled: "npm:2.0.x"
    kuler: "npm:^2.0.0"
  checksum: 10c0/a5133df8492802465ed01f2f0a5784585241a1030c362d54a602ed1839816d6c93d71dde05cf2ddb4fd0796238c19774406bd62fa2564b637907b495f52425fe
  languageName: node
  linkType: hard

"@emnapi/core@npm:^1.4.3":
  version: 1.4.5
  resolution: "@emnapi/core@npm:1.4.5"
  dependencies:
    "@emnapi/wasi-threads": "npm:1.0.4"
    tslib: "npm:^2.4.0"
  checksum: 10c0/da4a57f65f325d720d0e0d1a9c6618b90c4c43a5027834a110476984e1d47c95ebaed4d316b5dddb9c0ed9a493ffeb97d1934f9677035f336d8a36c1f3b2818f
  languageName: node
  linkType: hard

"@emnapi/runtime@npm:^1.4.3":
  version: 1.4.5
  resolution: "@emnapi/runtime@npm:1.4.5"
  dependencies:
    tslib: "npm:^2.4.0"
  checksum: 10c0/37a0278be5ac81e918efe36f1449875cbafba947039c53c65a1f8fc238001b866446fc66041513b286baaff5d6f9bec667f5164b3ca481373a8d9cb65bfc984b
  languageName: node
  linkType: hard

"@emnapi/wasi-threads@npm:1.0.4, @emnapi/wasi-threads@npm:^1.0.2":
  version: 1.0.4
  resolution: "@emnapi/wasi-threads@npm:1.0.4"
  dependencies:
    tslib: "npm:^2.4.0"
  checksum: 10c0/2c91a53e62f875800baf035c4d42c9c0d18e5afd9a31ca2aac8b435aeaeaeaac386b5b3d0d0e70aa7a5a9852bbe05106b1f680cd82cce03145c703b423d41313
  languageName: node
  linkType: hard

"@esbuild/aix-ppc64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/aix-ppc64@npm:0.25.8"
  conditions: os=aix & cpu=ppc64
  languageName: node
  linkType: hard

"@esbuild/android-arm64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/android-arm64@npm:0.25.8"
  conditions: os=android & cpu=arm64
  languageName: node
  linkType: hard

"@esbuild/android-arm@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/android-arm@npm:0.25.8"
  conditions: os=android & cpu=arm
  languageName: node
  linkType: hard

"@esbuild/android-x64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/android-x64@npm:0.25.8"
  conditions: os=android & cpu=x64
  languageName: node
  linkType: hard

"@esbuild/darwin-arm64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/darwin-arm64@npm:0.25.8"
  conditions: os=darwin & cpu=arm64
  languageName: node
  linkType: hard

"@esbuild/darwin-x64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/darwin-x64@npm:0.25.8"
  conditions: os=darwin & cpu=x64
  languageName: node
  linkType: hard

"@esbuild/freebsd-arm64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/freebsd-arm64@npm:0.25.8"
  conditions: os=freebsd & cpu=arm64
  languageName: node
  linkType: hard

"@esbuild/freebsd-x64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/freebsd-x64@npm:0.25.8"
  conditions: os=freebsd & cpu=x64
  languageName: node
  linkType: hard

"@esbuild/linux-arm64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/linux-arm64@npm:0.25.8"
  conditions: os=linux & cpu=arm64
  languageName: node
  linkType: hard

"@esbuild/linux-arm@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/linux-arm@npm:0.25.8"
  conditions: os=linux & cpu=arm
  languageName: node
  linkType: hard

"@esbuild/linux-ia32@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/linux-ia32@npm:0.25.8"
  conditions: os=linux & cpu=ia32
  languageName: node
  linkType: hard

"@esbuild/linux-loong64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/linux-loong64@npm:0.25.8"
  conditions: os=linux & cpu=loong64
  languageName: node
  linkType: hard

"@esbuild/linux-mips64el@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/linux-mips64el@npm:0.25.8"
  conditions: os=linux & cpu=mips64el
  languageName: node
  linkType: hard

"@esbuild/linux-ppc64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/linux-ppc64@npm:0.25.8"
  conditions: os=linux & cpu=ppc64
  languageName: node
  linkType: hard

"@esbuild/linux-riscv64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/linux-riscv64@npm:0.25.8"
  conditions: os=linux & cpu=riscv64
  languageName: node
  linkType: hard

"@esbuild/linux-s390x@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/linux-s390x@npm:0.25.8"
  conditions: os=linux & cpu=s390x
  languageName: node
  linkType: hard

"@esbuild/linux-x64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/linux-x64@npm:0.25.8"
  conditions: os=linux & cpu=x64
  languageName: node
  linkType: hard

"@esbuild/netbsd-arm64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/netbsd-arm64@npm:0.25.8"
  conditions: os=netbsd & cpu=arm64
  languageName: node
  linkType: hard

"@esbuild/netbsd-x64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/netbsd-x64@npm:0.25.8"
  conditions: os=netbsd & cpu=x64
  languageName: node
  linkType: hard

"@esbuild/openbsd-arm64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/openbsd-arm64@npm:0.25.8"
  conditions: os=openbsd & cpu=arm64
  languageName: node
  linkType: hard

"@esbuild/openbsd-x64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/openbsd-x64@npm:0.25.8"
  conditions: os=openbsd & cpu=x64
  languageName: node
  linkType: hard

"@esbuild/openharmony-arm64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/openharmony-arm64@npm:0.25.8"
  conditions: os=openharmony & cpu=arm64
  languageName: node
  linkType: hard

"@esbuild/sunos-x64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/sunos-x64@npm:0.25.8"
  conditions: os=sunos & cpu=x64
  languageName: node
  linkType: hard

"@esbuild/win32-arm64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/win32-arm64@npm:0.25.8"
  conditions: os=win32 & cpu=arm64
  languageName: node
  linkType: hard

"@esbuild/win32-ia32@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/win32-ia32@npm:0.25.8"
  conditions: os=win32 & cpu=ia32
  languageName: node
  linkType: hard

"@esbuild/win32-x64@npm:0.25.8":
  version: 0.25.8
  resolution: "@esbuild/win32-x64@npm:0.25.8"
  conditions: os=win32 & cpu=x64
  languageName: node
  linkType: hard

"@eslint-community/eslint-utils@npm:^4.2.0, @eslint-community/eslint-utils@npm:^4.7.0":
  version: 4.7.0
  resolution: "@eslint-community/eslint-utils@npm:4.7.0"
  dependencies:
    eslint-visitor-keys: "npm:^3.4.3"
  peerDependencies:
    eslint: ^6.0.0 || ^7.0.0 || >=8.0.0
  checksum: 10c0/c0f4f2bd73b7b7a9de74b716a664873d08ab71ab439e51befe77d61915af41a81ecec93b408778b3a7856185244c34c2c8ee28912072ec14def84ba2dec70adf
  languageName: node
  linkType: hard

"@eslint-community/regexpp@npm:^4.10.0, @eslint-community/regexpp@npm:^4.12.1":
  version: 4.12.1
  resolution: "@eslint-community/regexpp@npm:4.12.1"
  checksum: 10c0/a03d98c246bcb9109aec2c08e4d10c8d010256538dcb3f56610191607214523d4fb1b00aa81df830b6dffb74c5fa0be03642513a289c567949d3e550ca11cdf6
  languageName: node
  linkType: hard

"@eslint/config-array@npm:^0.21.0":
  version: 0.21.0
  resolution: "@eslint/config-array@npm:0.21.0"
  dependencies:
    "@eslint/object-schema": "npm:^2.1.6"
    debug: "npm:^4.3.1"
    minimatch: "npm:^3.1.2"
  checksum: 10c0/0ea801139166c4aa56465b309af512ef9b2d3c68f9198751bbc3e21894fe70f25fbf26e1b0e9fffff41857bc21bfddeee58649ae6d79aadcd747db0c5dca771f
  languageName: node
  linkType: hard

"@eslint/config-helpers@npm:^0.3.0":
  version: 0.3.0
  resolution: "@eslint/config-helpers@npm:0.3.0"
  checksum: 10c0/013ae7b189eeae8b30cc2ee87bc5c9c091a9cd615579003290eb28bebad5d78806a478e74ba10b3fe08ed66975b52af7d2cd4b4b43990376412b14e5664878c8
  languageName: node
  linkType: hard

"@eslint/core@npm:^0.15.0, @eslint/core@npm:^0.15.1":
  version: 0.15.1
  resolution: "@eslint/core@npm:0.15.1"
  dependencies:
    "@types/json-schema": "npm:^7.0.15"
  checksum: 10c0/abaf641940776638b8c15a38d99ce0dac551a8939310ec81b9acd15836a574cf362588eaab03ab11919bc2a0f9648b19ea8dee33bf12675eb5b6fd38bda6f25e
  languageName: node
  linkType: hard

"@eslint/eslintrc@npm:^3.3.1":
  version: 3.3.1
  resolution: "@eslint/eslintrc@npm:3.3.1"
  dependencies:
    ajv: "npm:^6.12.4"
    debug: "npm:^4.3.2"
    espree: "npm:^10.0.1"
    globals: "npm:^14.0.0"
    ignore: "npm:^5.2.0"
    import-fresh: "npm:^3.2.1"
    js-yaml: "npm:^4.1.0"
    minimatch: "npm:^3.1.2"
    strip-json-comments: "npm:^3.1.1"
  checksum: 10c0/b0e63f3bc5cce4555f791a4e487bf999173fcf27c65e1ab6e7d63634d8a43b33c3693e79f192cbff486d7df1be8ebb2bd2edc6e70ddd486cbfa84a359a3e3b41
  languageName: node
  linkType: hard

"@eslint/js@npm:9.32.0, @eslint/js@npm:^9.32.0":
  version: 9.32.0
  resolution: "@eslint/js@npm:9.32.0"
  checksum: 10c0/f71e8f9146638d11fb15238279feff98801120a4d4130f1c587c4f09b024ff5ec01af1ba88e97ba6b7013488868898a668f77091300cc3d4394c7a8ed32d2667
  languageName: node
  linkType: hard

"@eslint/object-schema@npm:^2.1.6":
  version: 2.1.6
  resolution: "@eslint/object-schema@npm:2.1.6"
  checksum: 10c0/b8cdb7edea5bc5f6a96173f8d768d3554a628327af536da2fc6967a93b040f2557114d98dbcdbf389d5a7b290985ad6a9ce5babc547f36fc1fde42e674d11a56
  languageName: node
  linkType: hard

"@eslint/plugin-kit@npm:^0.3.4":
  version: 0.3.4
  resolution: "@eslint/plugin-kit@npm:0.3.4"
  dependencies:
    "@eslint/core": "npm:^0.15.1"
    levn: "npm:^0.4.1"
  checksum: 10c0/64331ca100f62a0115d10419a28059d0f377e390192163b867b9019517433d5073d10b4ec21f754fa01faf832aceb34178745924baab2957486f8bf95fd628d2
  languageName: node
  linkType: hard

"@hono/node-server@npm:^1.12.0":
  version: 1.17.1
  resolution: "@hono/node-server@npm:1.17.1"
  peerDependencies:
    hono: ^4
  checksum: 10c0/a313e389a34782dbf310fc180dbd62bc3b32f41eeb5810718b560416122cdf81cd1bec1c31a3117e7fae26b596f9d8ffb2f7ac37a3a0194e2db89d4d4126998d
  languageName: node
  linkType: hard

"@hono/zod-validator@npm:^0.2.2":
  version: 0.2.2
  resolution: "@hono/zod-validator@npm:0.2.2"
  peerDependencies:
    hono: ">=3.9.0"
    zod: ^3.19.1
  checksum: 10c0/3d6d03d28287e6f05e4cf5b86f3fa5fa386429a4212881f7344fe93272a69732ca8dd98634eb51df434b002230d2be2f3c6822f3b1ab320676932583856b30a5
  languageName: node
  linkType: hard

"@humanfs/core@npm:^0.19.1":
  version: 0.19.1
  resolution: "@humanfs/core@npm:0.19.1"
  checksum: 10c0/aa4e0152171c07879b458d0e8a704b8c3a89a8c0541726c6b65b81e84fd8b7564b5d6c633feadc6598307d34564bd53294b533491424e8e313d7ab6c7bc5dc67
  languageName: node
  linkType: hard

"@humanfs/node@npm:^0.16.6":
  version: 0.16.6
  resolution: "@humanfs/node@npm:0.16.6"
  dependencies:
    "@humanfs/core": "npm:^0.19.1"
    "@humanwhocodes/retry": "npm:^0.3.0"
  checksum: 10c0/8356359c9f60108ec204cbd249ecd0356667359b2524886b357617c4a7c3b6aace0fd5a369f63747b926a762a88f8a25bc066fa1778508d110195ce7686243e1
  languageName: node
  linkType: hard

"@humanwhocodes/module-importer@npm:^1.0.1":
  version: 1.0.1
  resolution: "@humanwhocodes/module-importer@npm:1.0.1"
  checksum: 10c0/909b69c3b86d482c26b3359db16e46a32e0fb30bd306a3c176b8313b9e7313dba0f37f519de6aa8b0a1921349e505f259d19475e123182416a506d7f87e7f529
  languageName: node
  linkType: hard

"@humanwhocodes/retry@npm:^0.3.0":
  version: 0.3.1
  resolution: "@humanwhocodes/retry@npm:0.3.1"
  checksum: 10c0/f0da1282dfb45e8120480b9e2e275e2ac9bbe1cf016d046fdad8e27cc1285c45bb9e711681237944445157b430093412b4446c1ab3fc4bb037861b5904101d3b
  languageName: node
  linkType: hard

"@humanwhocodes/retry@npm:^0.4.2":
  version: 0.4.3
  resolution: "@humanwhocodes/retry@npm:0.4.3"
  checksum: 10c0/3775bb30087d4440b3f7406d5a057777d90e4b9f435af488a4923ef249e93615fb78565a85f173a186a076c7706a81d0d57d563a2624e4de2c5c9c66c486ce42
  languageName: node
  linkType: hard

"@isaacs/cliui@npm:^8.0.2":
  version: 8.0.2
  resolution: "@isaacs/cliui@npm:8.0.2"
  dependencies:
    string-width: "npm:^5.1.2"
    string-width-cjs: "npm:string-width@^4.2.0"
    strip-ansi: "npm:^7.0.1"
    strip-ansi-cjs: "npm:strip-ansi@^6.0.1"
    wrap-ansi: "npm:^8.1.0"
    wrap-ansi-cjs: "npm:wrap-ansi@^7.0.0"
  checksum: 10c0/b1bf42535d49f11dc137f18d5e4e63a28c5569de438a221c369483731e9dac9fb797af554e8bf02b6192d1e5eba6e6402cf93900c3d0ac86391d00d04876789e
  languageName: node
  linkType: hard

"@isaacs/fs-minipass@npm:^4.0.0":
  version: 4.0.1
  resolution: "@isaacs/fs-minipass@npm:4.0.1"
  dependencies:
    minipass: "npm:^7.0.4"
  checksum: 10c0/c25b6dc1598790d5b55c0947a9b7d111cfa92594db5296c3b907e2f533c033666f692a3939eadac17b1c7c40d362d0b0635dc874cbfe3e70db7c2b07cc97a5d2
  languageName: node
  linkType: hard

"@jridgewell/gen-mapping@npm:^0.3.5":
  version: 0.3.12
  resolution: "@jridgewell/gen-mapping@npm:0.3.12"
  dependencies:
    "@jridgewell/sourcemap-codec": "npm:^1.5.0"
    "@jridgewell/trace-mapping": "npm:^0.3.24"
  checksum: 10c0/32f771ae2467e4d440be609581f7338d786d3d621bac3469e943b9d6d116c23c4becb36f84898a92bbf2f3c0511365c54a945a3b86a83141547a2a360a5ec0c7
  languageName: node
  linkType: hard

"@jridgewell/resolve-uri@npm:^3.1.0":
  version: 3.1.2
  resolution: "@jridgewell/resolve-uri@npm:3.1.2"
  checksum: 10c0/d502e6fb516b35032331406d4e962c21fe77cdf1cbdb49c6142bcbd9e30507094b18972778a6e27cbad756209cfe34b1a27729e6fa08a2eb92b33943f680cf1e
  languageName: node
  linkType: hard

"@jridgewell/sourcemap-codec@npm:^1.4.14, @jridgewell/sourcemap-codec@npm:^1.5.0":
  version: 1.5.4
  resolution: "@jridgewell/sourcemap-codec@npm:1.5.4"
  checksum: 10c0/c5aab3e6362a8dd94ad80ab90845730c825fc4c8d9cf07ebca7a2eb8a832d155d62558800fc41d42785f989ddbb21db6df004d1786e8ecb65e428ab8dff71309
  languageName: node
  linkType: hard

"@jridgewell/trace-mapping@npm:^0.3.24":
  version: 0.3.29
  resolution: "@jridgewell/trace-mapping@npm:0.3.29"
  dependencies:
    "@jridgewell/resolve-uri": "npm:^3.1.0"
    "@jridgewell/sourcemap-codec": "npm:^1.4.14"
  checksum: 10c0/fb547ba31658c4d74eb17e7389f4908bf7c44cef47acb4c5baa57289daf68e6fe53c639f41f751b3923aca67010501264f70e7b49978ad1f040294b22c37b333
  languageName: node
  linkType: hard

"@langchain/anthropic@npm:^0.3.24":
  version: 0.3.24
  resolution: "@langchain/anthropic@npm:0.3.24"
  dependencies:
    "@anthropic-ai/sdk": "npm:^0.56.0"
    fast-xml-parser: "npm:^4.4.1"
  peerDependencies:
    "@langchain/core": ">=0.3.58 <0.4.0"
  checksum: 10c0/6b042413881929262e43a10b8fb0d4960c1cac00434813036e7495801e362934f759d66a7067d6a9ae912263b5d3355906564614290259aa8cc7eccb125e948f
  languageName: node
  linkType: hard

"@langchain/core@npm:^0.3.66":
  version: 0.3.66
  resolution: "@langchain/core@npm:0.3.66"
  dependencies:
    "@cfworker/json-schema": "npm:^4.0.2"
    ansi-styles: "npm:^5.0.0"
    camelcase: "npm:6"
    decamelize: "npm:1.2.0"
    js-tiktoken: "npm:^1.0.12"
    langsmith: "npm:^0.3.46"
    mustache: "npm:^4.2.0"
    p-queue: "npm:^6.6.2"
    p-retry: "npm:4"
    uuid: "npm:^10.0.0"
    zod: "npm:^3.25.32"
    zod-to-json-schema: "npm:^3.22.3"
  checksum: 10c0/c6689e860ac037e799cb41f6ca756086b8673f5c1ae9090db1a060934755c832827f6bb317ec0fe7677505549b1ae40e1bf0bd45778f8bab34bb2e52e87bef11
  languageName: node
  linkType: hard

"@langchain/langgraph-api@npm:^0.0.52":
  version: 0.0.52
  resolution: "@langchain/langgraph-api@npm:0.0.52"
  dependencies:
    "@babel/code-frame": "npm:^7.26.2"
    "@hono/node-server": "npm:^1.12.0"
    "@hono/zod-validator": "npm:^0.2.2"
    "@langchain/langgraph-ui": "npm:0.0.52"
    "@types/json-schema": "npm:^7.0.15"
    "@typescript/vfs": "npm:^1.6.0"
    dedent: "npm:^1.5.3"
    dotenv: "npm:^16.4.7"
    exit-hook: "npm:^4.0.0"
    hono: "npm:^4.5.4"
    langsmith: "npm:^0.3.33"
    open: "npm:^10.1.0"
    semver: "npm:^7.7.1"
    stacktrace-parser: "npm:^0.1.10"
    superjson: "npm:^2.2.2"
    tsx: "npm:^4.19.3"
    uuid: "npm:^10.0.0"
    winston: "npm:^3.17.0"
    winston-console-format: "npm:^1.0.8"
    zod: "npm:^3.23.8"
  peerDependencies:
    "@langchain/core": ^0.3.59
    "@langchain/langgraph": ^0.2.57 || ^0.3.0
    "@langchain/langgraph-checkpoint": ~0.0.16
    "@langchain/langgraph-sdk": ~0.0.70
    typescript: ^5.5.4
  peerDependenciesMeta:
    "@langchain/langgraph-sdk":
      optional: true
  checksum: 10c0/018e65b39ce4324dd956da1e5c14040a7293bc5b4affc1297f821c09016c4ae6d0170b9ae86454166bb87485218d76ca475b9fa4f558e6a754a10755fcb349ce
  languageName: node
  linkType: hard

"@langchain/langgraph-checkpoint@npm:~0.0.18":
  version: 0.0.18
  resolution: "@langchain/langgraph-checkpoint@npm:0.0.18"
  dependencies:
    uuid: "npm:^10.0.0"
  peerDependencies:
    "@langchain/core": ">=0.2.31 <0.4.0"
  checksum: 10c0/d0a1525a7e45044953ddaafeeb702f7ddfb3d30632ac93842f8f940b1fe410f6e132080db6f6843e0a19bc59f8e4fef6b55d59f2812bc708f50cbec5fdcd6ce6
  languageName: node
  linkType: hard

"@langchain/langgraph-sdk@npm:^0.0.102, @langchain/langgraph-sdk@npm:~0.0.100":
  version: 0.0.102
  resolution: "@langchain/langgraph-sdk@npm:0.0.102"
  dependencies:
    "@types/json-schema": "npm:^7.0.15"
    p-queue: "npm:^6.6.2"
    p-retry: "npm:4"
    uuid: "npm:^9.0.0"
  peerDependencies:
    "@langchain/core": ">=0.2.31 <0.4.0"
    react: ^18 || ^19
    react-dom: ^18 || ^19
  peerDependenciesMeta:
    "@langchain/core":
      optional: true
    react:
      optional: true
    react-dom:
      optional: true
  checksum: 10c0/5567fe250e1f90f1d302f1a37dbc109e3a880743788026e35b7e395bd70aab8d8409a4e5b0d2ac09af4d02ffe045e33aed372aa108aa733ad8cfb76889c6bb36
  languageName: node
  linkType: hard

"@langchain/langgraph-ui@npm:0.0.52":
  version: 0.0.52
  resolution: "@langchain/langgraph-ui@npm:0.0.52"
  dependencies:
    "@commander-js/extra-typings": "npm:^13.0.0"
    commander: "npm:^13.0.0"
    esbuild: "npm:^0.25.0"
    esbuild-plugin-tailwindcss: "npm:^2.0.1"
    zod: "npm:^3.23.8"
  bin:
    langgraphjs-ui: ./dist/cli.mjs
  checksum: 10c0/1ce1e0a16234c4e0603578a41b9c66dc24b99a655f9ef8d08642d9265d9da8f237078ef1b82ccb7dee2b1d06ccb46a28caba46eb80068b126054ca4060bcbc74
  languageName: node
  linkType: hard

"@langchain/langgraph@npm:^0.3.11":
  version: 0.3.11
  resolution: "@langchain/langgraph@npm:0.3.11"
  dependencies:
    "@langchain/langgraph-checkpoint": "npm:~0.0.18"
    "@langchain/langgraph-sdk": "npm:~0.0.100"
    uuid: "npm:^10.0.0"
    zod: "npm:^3.25.32"
  peerDependencies:
    "@langchain/core": ">=0.3.58 < 0.4.0"
    zod-to-json-schema: ^3.x
  peerDependenciesMeta:
    zod-to-json-schema:
      optional: true
  checksum: 10c0/55122bcb6e76fd1debafc9e57ecb5cdc47eefa983aba2468649d80abb588f98a9090fbe87bf460ce4a88a4aa9b4250af4fe1e9ed85ca436e22862bd1c2cb6ec8
  languageName: node
  linkType: hard

"@langchain/openai@npm:^0.6.3":
  version: 0.6.3
  resolution: "@langchain/openai@npm:0.6.3"
  dependencies:
    js-tiktoken: "npm:^1.0.12"
    openai: "npm:^5.3.0"
    zod: "npm:^3.25.32"
  peerDependencies:
    "@langchain/core": ">=0.3.58 <0.4.0"
  checksum: 10c0/cda9199446d241ba67da9bcf33571955043b69e5f21883e8f9052cfb5063871cac2e1a30d889b7b04e5cf4ab8f55643fe3cd13c9cfb5ab8ab514a190fa8104f0
  languageName: node
  linkType: hard

"@napi-rs/wasm-runtime@npm:^0.2.11":
  version: 0.2.12
  resolution: "@napi-rs/wasm-runtime@npm:0.2.12"
  dependencies:
    "@emnapi/core": "npm:^1.4.3"
    "@emnapi/runtime": "npm:^1.4.3"
    "@tybys/wasm-util": "npm:^0.10.0"
  checksum: 10c0/6d07922c0613aab30c6a497f4df297ca7c54e5b480e00035e0209b872d5c6aab7162fc49477267556109c2c7ed1eb9c65a174e27e9b87568106a87b0a6e3ca7d
  languageName: node
  linkType: hard

"@nodelib/fs.scandir@npm:2.1.5":
  version: 2.1.5
  resolution: "@nodelib/fs.scandir@npm:2.1.5"
  dependencies:
    "@nodelib/fs.stat": "npm:2.0.5"
    run-parallel: "npm:^1.1.9"
  checksum: 10c0/732c3b6d1b1e967440e65f284bd06e5821fedf10a1bea9ed2bb75956ea1f30e08c44d3def9d6a230666574edbaf136f8cfd319c14fd1f87c66e6a44449afb2eb
  languageName: node
  linkType: hard

"@nodelib/fs.stat@npm:2.0.5, @nodelib/fs.stat@npm:^2.0.2":
  version: 2.0.5
  resolution: "@nodelib/fs.stat@npm:2.0.5"
  checksum: 10c0/88dafe5e3e29a388b07264680dc996c17f4bda48d163a9d4f5c1112979f0ce8ec72aa7116122c350b4e7976bc5566dc3ddb579be1ceaacc727872eb4ed93926d
  languageName: node
  linkType: hard

"@nodelib/fs.walk@npm:^1.2.3":
  version: 1.2.8
  resolution: "@nodelib/fs.walk@npm:1.2.8"
  dependencies:
    "@nodelib/fs.scandir": "npm:2.1.5"
    fastq: "npm:^1.6.0"
  checksum: 10c0/db9de047c3bb9b51f9335a7bb46f4fcfb6829fb628318c12115fbaf7d369bfce71c15b103d1fc3b464812d936220ee9bc1c8f762d032c9f6be9acc99249095b1
  languageName: node
  linkType: hard

"@npmcli/agent@npm:^3.0.0":
  version: 3.0.0
  resolution: "@npmcli/agent@npm:3.0.0"
  dependencies:
    agent-base: "npm:^7.1.0"
    http-proxy-agent: "npm:^7.0.0"
    https-proxy-agent: "npm:^7.0.1"
    lru-cache: "npm:^10.0.1"
    socks-proxy-agent: "npm:^8.0.3"
  checksum: 10c0/efe37b982f30740ee77696a80c196912c274ecd2cb243bc6ae7053a50c733ce0f6c09fda085145f33ecf453be19654acca74b69e81eaad4c90f00ccffe2f9271
  languageName: node
  linkType: hard

"@npmcli/fs@npm:^4.0.0":
  version: 4.0.0
  resolution: "@npmcli/fs@npm:4.0.0"
  dependencies:
    semver: "npm:^7.3.5"
  checksum: 10c0/c90935d5ce670c87b6b14fab04a965a3b8137e585f8b2a6257263bd7f97756dd736cb165bb470e5156a9e718ecd99413dccc54b1138c1a46d6ec7cf325982fe5
  languageName: node
  linkType: hard

"@pkgjs/parseargs@npm:^0.11.0":
  version: 0.11.0
  resolution: "@pkgjs/parseargs@npm:0.11.0"
  checksum: 10c0/5bd7576bb1b38a47a7fc7b51ac9f38748e772beebc56200450c4a817d712232b8f1d3ef70532c80840243c657d491cf6a6be1e3a214cff907645819fdc34aadd
  languageName: node
  linkType: hard

"@tailwindcss/node@npm:4.1.11":
  version: 4.1.11
  resolution: "@tailwindcss/node@npm:4.1.11"
  dependencies:
    "@ampproject/remapping": "npm:^2.3.0"
    enhanced-resolve: "npm:^5.18.1"
    jiti: "npm:^2.4.2"
    lightningcss: "npm:1.30.1"
    magic-string: "npm:^0.30.17"
    source-map-js: "npm:^1.2.1"
    tailwindcss: "npm:4.1.11"
  checksum: 10c0/1a433aecd80d0c6d07d468ed69b696e4e02996e6b77cc5ed66e3c91b02f5fa9a26320fb321e4b1aa107003b401d7a4ffeb2986966dc022ec329a44e54493a2aa
  languageName: node
  linkType: hard

"@tailwindcss/oxide-android-arm64@npm:4.1.11":
  version: 4.1.11
  resolution: "@tailwindcss/oxide-android-arm64@npm:4.1.11"
  conditions: os=android & cpu=arm64
  languageName: node
  linkType: hard

"@tailwindcss/oxide-darwin-arm64@npm:4.1.11":
  version: 4.1.11
  resolution: "@tailwindcss/oxide-darwin-arm64@npm:4.1.11"
  conditions: os=darwin & cpu=arm64
  languageName: node
  linkType: hard

"@tailwindcss/oxide-darwin-x64@npm:4.1.11":
  version: 4.1.11
  resolution: "@tailwindcss/oxide-darwin-x64@npm:4.1.11"
  conditions: os=darwin & cpu=x64
  languageName: node
  linkType: hard

"@tailwindcss/oxide-freebsd-x64@npm:4.1.11":
  version: 4.1.11
  resolution: "@tailwindcss/oxide-freebsd-x64@npm:4.1.11"
  conditions: os=freebsd & cpu=x64
  languageName: node
  linkType: hard

"@tailwindcss/oxide-linux-arm-gnueabihf@npm:4.1.11":
  version: 4.1.11
  resolution: "@tailwindcss/oxide-linux-arm-gnueabihf@npm:4.1.11"
  conditions: os=linux & cpu=arm
  languageName: node
  linkType: hard

"@tailwindcss/oxide-linux-arm64-gnu@npm:4.1.11":
  version: 4.1.11
  resolution: "@tailwindcss/oxide-linux-arm64-gnu@npm:4.1.11"
  conditions: os=linux & cpu=arm64 & libc=glibc
  languageName: node
  linkType: hard

"@tailwindcss/oxide-linux-arm64-musl@npm:4.1.11":
  version: 4.1.11
  resolution: "@tailwindcss/oxide-linux-arm64-musl@npm:4.1.11"
  conditions: os=linux & cpu=arm64 & libc=musl
  languageName: node
  linkType: hard

"@tailwindcss/oxide-linux-x64-gnu@npm:4.1.11":
  version: 4.1.11
  resolution: "@tailwindcss/oxide-linux-x64-gnu@npm:4.1.11"
  conditions: os=linux & cpu=x64 & libc=glibc
  languageName: node
  linkType: hard

"@tailwindcss/oxide-linux-x64-musl@npm:4.1.11":
  version: 4.1.11
  resolution: "@tailwindcss/oxide-linux-x64-musl@npm:4.1.11"
  conditions: os=linux & cpu=x64 & libc=musl
  languageName: node
  linkType: hard

"@tailwindcss/oxide-wasm32-wasi@npm:4.1.11":
  version: 4.1.11
  resolution: "@tailwindcss/oxide-wasm32-wasi@npm:4.1.11"
  dependencies:
    "@emnapi/core": "npm:^1.4.3"
    "@emnapi/runtime": "npm:^1.4.3"
    "@emnapi/wasi-threads": "npm:^1.0.2"
    "@napi-rs/wasm-runtime": "npm:^0.2.11"
    "@tybys/wasm-util": "npm:^0.9.0"
    tslib: "npm:^2.8.0"
  conditions: cpu=wasm32
  languageName: node
  linkType: hard

"@tailwindcss/oxide-win32-arm64-msvc@npm:4.1.11":
  version: 4.1.11
  resolution: "@tailwindcss/oxide-win32-arm64-msvc@npm:4.1.11"
  conditions: os=win32 & cpu=arm64
  languageName: node
  linkType: hard

"@tailwindcss/oxide-win32-x64-msvc@npm:4.1.11":
  version: 4.1.11
  resolution: "@tailwindcss/oxide-win32-x64-msvc@npm:4.1.11"
  conditions: os=win32 & cpu=x64
  languageName: node
  linkType: hard

"@tailwindcss/oxide@npm:4.1.11":
  version: 4.1.11
  resolution: "@tailwindcss/oxide@npm:4.1.11"
  dependencies:
    "@tailwindcss/oxide-android-arm64": "npm:4.1.11"
    "@tailwindcss/oxide-darwin-arm64": "npm:4.1.11"
    "@tailwindcss/oxide-darwin-x64": "npm:4.1.11"
    "@tailwindcss/oxide-freebsd-x64": "npm:4.1.11"
    "@tailwindcss/oxide-linux-arm-gnueabihf": "npm:4.1.11"
    "@tailwindcss/oxide-linux-arm64-gnu": "npm:4.1.11"
    "@tailwindcss/oxide-linux-arm64-musl": "npm:4.1.11"
    "@tailwindcss/oxide-linux-x64-gnu": "npm:4.1.11"
    "@tailwindcss/oxide-linux-x64-musl": "npm:4.1.11"
    "@tailwindcss/oxide-wasm32-wasi": "npm:4.1.11"
    "@tailwindcss/oxide-win32-arm64-msvc": "npm:4.1.11"
    "@tailwindcss/oxide-win32-x64-msvc": "npm:4.1.11"
    detect-libc: "npm:^2.0.4"
    tar: "npm:^7.4.3"
  dependenciesMeta:
    "@tailwindcss/oxide-android-arm64":
      optional: true
    "@tailwindcss/oxide-darwin-arm64":
      optional: true
    "@tailwindcss/oxide-darwin-x64":
      optional: true
    "@tailwindcss/oxide-freebsd-x64":
      optional: true
    "@tailwindcss/oxide-linux-arm-gnueabihf":
      optional: true
    "@tailwindcss/oxide-linux-arm64-gnu":
      optional: true
    "@tailwindcss/oxide-linux-arm64-musl":
      optional: true
    "@tailwindcss/oxide-linux-x64-gnu":
      optional: true
    "@tailwindcss/oxide-linux-x64-musl":
      optional: true
    "@tailwindcss/oxide-wasm32-wasi":
      optional: true
    "@tailwindcss/oxide-win32-arm64-msvc":
      optional: true
    "@tailwindcss/oxide-win32-x64-msvc":
      optional: true
  checksum: 10c0/0455483b0e52885a3f36ecbec5409c360159bb0ee969f3a64c2d93dbd94d0d769c1351b7031f4d4b9d8bed997d04d685ca9519160714f432d63f4e824ce1406d
  languageName: node
  linkType: hard

"@tailwindcss/postcss@npm:^4.0.5":
  version: 4.1.11
  resolution: "@tailwindcss/postcss@npm:4.1.11"
  dependencies:
    "@alloc/quick-lru": "npm:^5.2.0"
    "@tailwindcss/node": "npm:4.1.11"
    "@tailwindcss/oxide": "npm:4.1.11"
    postcss: "npm:^8.4.41"
    tailwindcss: "npm:4.1.11"
  checksum: 10c0/e449e1992d0723061aa9452979cd01727db4d1e81b2c16762b01899d06a6c9015792d10d3db4cb553e2e59f307593dc4ccf679ef1add5f774da73d3a091f7227
  languageName: node
  linkType: hard

"@tybys/wasm-util@npm:^0.10.0":
  version: 0.10.0
  resolution: "@tybys/wasm-util@npm:0.10.0"
  dependencies:
    tslib: "npm:^2.4.0"
  checksum: 10c0/044feba55c1e2af703aa4946139969badb183ce1a659a75ed60bc195a90e73a3f3fc53bcd643497c9954597763ddb051fec62f80962b2ca6fc716ba897dc696e
  languageName: node
  linkType: hard

"@tybys/wasm-util@npm:^0.9.0":
  version: 0.9.0
  resolution: "@tybys/wasm-util@npm:0.9.0"
  dependencies:
    tslib: "npm:^2.4.0"
  checksum: 10c0/f9fde5c554455019f33af6c8215f1a1435028803dc2a2825b077d812bed4209a1a64444a4ca0ce2ea7e1175c8d88e2f9173a36a33c199e8a5c671aa31de8242d
  languageName: node
  linkType: hard

"@types/estree@npm:^1.0.6":
  version: 1.0.8
  resolution: "@types/estree@npm:1.0.8"
  checksum: 10c0/39d34d1afaa338ab9763f37ad6066e3f349444f9052b9676a7cc0252ef9485a41c6d81c9c4e0d26e9077993354edf25efc853f3224dd4b447175ef62bdcc86a5
  languageName: node
  linkType: hard

"@types/json-schema@npm:^7.0.15":
  version: 7.0.15
  resolution: "@types/json-schema@npm:7.0.15"
  checksum: 10c0/a996a745e6c5d60292f36731dd41341339d4eeed8180bb09226e5c8d23759067692b1d88e5d91d72ee83dfc00d3aca8e7bd43ea120516c17922cbcb7c3e252db
  languageName: node
  linkType: hard

"@types/retry@npm:0.12.0":
  version: 0.12.0
  resolution: "@types/retry@npm:0.12.0"
  checksum: 10c0/7c5c9086369826f569b83a4683661557cab1361bac0897a1cefa1a915ff739acd10ca0d62b01071046fe3f5a3f7f2aec80785fe283b75602dc6726781ea3e328
  languageName: node
  linkType: hard

"@types/triple-beam@npm:^1.3.2":
  version: 1.3.5
  resolution: "@types/triple-beam@npm:1.3.5"
  checksum: 10c0/d5d7f25da612f6d79266f4f1bb9c1ef8f1684e9f60abab251e1261170631062b656ba26ff22631f2760caeafd372abc41e64867cde27fba54fafb73a35b9056a
  languageName: node
  linkType: hard

"@types/uuid@npm:^10.0.0":
  version: 10.0.0
  resolution: "@types/uuid@npm:10.0.0"
  checksum: 10c0/9a1404bf287164481cb9b97f6bb638f78f955be57c40c6513b7655160beb29df6f84c915aaf4089a1559c216557dc4d2f79b48d978742d3ae10b937420ddac60
  languageName: node
  linkType: hard

"@typescript-eslint/eslint-plugin@npm:8.38.0":
  version: 8.38.0
  resolution: "@typescript-eslint/eslint-plugin@npm:8.38.0"
  dependencies:
    "@eslint-community/regexpp": "npm:^4.10.0"
    "@typescript-eslint/scope-manager": "npm:8.38.0"
    "@typescript-eslint/type-utils": "npm:8.38.0"
    "@typescript-eslint/utils": "npm:8.38.0"
    "@typescript-eslint/visitor-keys": "npm:8.38.0"
    graphemer: "npm:^1.4.0"
    ignore: "npm:^7.0.0"
    natural-compare: "npm:^1.4.0"
    ts-api-utils: "npm:^2.1.0"
  peerDependencies:
    "@typescript-eslint/parser": ^8.38.0
    eslint: ^8.57.0 || ^9.0.0
    typescript: ">=4.8.4 <5.9.0"
  checksum: 10c0/199b82e9f0136baecf515df7c31bfed926a7c6d4e6298f64ee1a77c8bdd7a8cb92a2ea55a5a345c9f2948a02f7be6d72530efbe803afa1892b593fbd529d0c27
  languageName: node
  linkType: hard

"@typescript-eslint/parser@npm:8.38.0":
  version: 8.38.0
  resolution: "@typescript-eslint/parser@npm:8.38.0"
  dependencies:
    "@typescript-eslint/scope-manager": "npm:8.38.0"
    "@typescript-eslint/types": "npm:8.38.0"
    "@typescript-eslint/typescript-estree": "npm:8.38.0"
    "@typescript-eslint/visitor-keys": "npm:8.38.0"
    debug: "npm:^4.3.4"
  peerDependencies:
    eslint: ^8.57.0 || ^9.0.0
    typescript: ">=4.8.4 <5.9.0"
  checksum: 10c0/5580c2a328f0c15f85e4a0961a07584013cc0aca85fe868486187f7c92e9e3f6602c6e3dab917b092b94cd492ed40827c6f5fea42730bef88eb17592c947adf4
  languageName: node
  linkType: hard

"@typescript-eslint/project-service@npm:8.38.0":
  version: 8.38.0
  resolution: "@typescript-eslint/project-service@npm:8.38.0"
  dependencies:
    "@typescript-eslint/tsconfig-utils": "npm:^8.38.0"
    "@typescript-eslint/types": "npm:^8.38.0"
    debug: "npm:^4.3.4"
  peerDependencies:
    typescript: ">=4.8.4 <5.9.0"
  checksum: 10c0/87d2f55521e289bbcdc666b1f4587ee2d43039cee927310b05abaa534b528dfb1b5565c1545bb4996d7fbdf9d5a3b0aa0e6c93a8f1289e3fcfd60d246364a884
  languageName: node
  linkType: hard

"@typescript-eslint/scope-manager@npm:8.38.0":
  version: 8.38.0
  resolution: "@typescript-eslint/scope-manager@npm:8.38.0"
  dependencies:
    "@typescript-eslint/types": "npm:8.38.0"
    "@typescript-eslint/visitor-keys": "npm:8.38.0"
  checksum: 10c0/ceaf489ea1f005afb187932a7ee363dfe1e0f7cc3db921283991e20e4c756411a5e25afbec72edd2095d6a4384f73591f4c750cf65b5eaa650c90f64ef9fe809
  languageName: node
  linkType: hard

"@typescript-eslint/tsconfig-utils@npm:8.38.0, @typescript-eslint/tsconfig-utils@npm:^8.38.0":
  version: 8.38.0
  resolution: "@typescript-eslint/tsconfig-utils@npm:8.38.0"
  peerDependencies:
    typescript: ">=4.8.4 <5.9.0"
  checksum: 10c0/1a90da16bf1f7cfbd0303640a8ead64a0080f2b1d5969994bdac3b80abfa1177f0c6fbf61250bae082e72cf5014308f2f5cc98edd6510202f13420a7ffd07a84
  languageName: node
  linkType: hard

"@typescript-eslint/type-utils@npm:8.38.0":
  version: 8.38.0
  resolution: "@typescript-eslint/type-utils@npm:8.38.0"
  dependencies:
    "@typescript-eslint/types": "npm:8.38.0"
    "@typescript-eslint/typescript-estree": "npm:8.38.0"
    "@typescript-eslint/utils": "npm:8.38.0"
    debug: "npm:^4.3.4"
    ts-api-utils: "npm:^2.1.0"
  peerDependencies:
    eslint: ^8.57.0 || ^9.0.0
    typescript: ">=4.8.4 <5.9.0"
  checksum: 10c0/27795c4bd0be395dda3424e57d746639c579b7522af1c17731b915298a6378fd78869e8e141526064b6047db2c86ba06444469ace19c98cda5779d06f4abd37c
  languageName: node
  linkType: hard

"@typescript-eslint/types@npm:8.38.0, @typescript-eslint/types@npm:^8.38.0":
  version: 8.38.0
  resolution: "@typescript-eslint/types@npm:8.38.0"
  checksum: 10c0/f0ac0060c98c0f3d1871f107177b6ae25a0f1846ca8bd8cfc7e1f1dd0ddce293cd8ac4a5764d6a767de3503d5d01defcd68c758cb7ba6de52f82b209a918d0d2
  languageName: node
  linkType: hard

"@typescript-eslint/typescript-estree@npm:8.38.0":
  version: 8.38.0
  resolution: "@typescript-eslint/typescript-estree@npm:8.38.0"
  dependencies:
    "@typescript-eslint/project-service": "npm:8.38.0"
    "@typescript-eslint/tsconfig-utils": "npm:8.38.0"
    "@typescript-eslint/types": "npm:8.38.0"
    "@typescript-eslint/visitor-keys": "npm:8.38.0"
    debug: "npm:^4.3.4"
    fast-glob: "npm:^3.3.2"
    is-glob: "npm:^4.0.3"
    minimatch: "npm:^9.0.4"
    semver: "npm:^7.6.0"
    ts-api-utils: "npm:^2.1.0"
  peerDependencies:
    typescript: ">=4.8.4 <5.9.0"
  checksum: 10c0/00a00f6549877f4ae5c2847fa5ac52bf42cbd59a87533856c359e2746e448ed150b27a6137c92fd50c06e6a4b39e386d6b738fac97d80d05596e81ce55933230
  languageName: node
  linkType: hard

"@typescript-eslint/utils@npm:8.38.0":
  version: 8.38.0
  resolution: "@typescript-eslint/utils@npm:8.38.0"
  dependencies:
    "@eslint-community/eslint-utils": "npm:^4.7.0"
    "@typescript-eslint/scope-manager": "npm:8.38.0"
    "@typescript-eslint/types": "npm:8.38.0"
    "@typescript-eslint/typescript-estree": "npm:8.38.0"
  peerDependencies:
    eslint: ^8.57.0 || ^9.0.0
    typescript: ">=4.8.4 <5.9.0"
  checksum: 10c0/e97a45bf44f315f9ed8c2988429e18c88e3369c9ee3227ee86446d2d49f7325abebbbc9ce801e178f676baa986d3e1fd4b5391f1640c6eb8944c123423ae43bb
  languageName: node
  linkType: hard

"@typescript-eslint/visitor-keys@npm:8.38.0":
  version: 8.38.0
  resolution: "@typescript-eslint/visitor-keys@npm:8.38.0"
  dependencies:
    "@typescript-eslint/types": "npm:8.38.0"
    eslint-visitor-keys: "npm:^4.2.1"
  checksum: 10c0/071a756e383f41a6c9e51d78c8c64bd41cd5af68b0faef5fbaec4fa5dbd65ec9e4cd610c2e2cdbe9e2facc362995f202850622b78e821609a277b5b601a1d4ec
  languageName: node
  linkType: hard

"@typescript/vfs@npm:^1.6.0":
  version: 1.6.1
  resolution: "@typescript/vfs@npm:1.6.1"
  dependencies:
    debug: "npm:^4.1.1"
  peerDependencies:
    typescript: "*"
  checksum: 10c0/3878686aff4bf26813dad9242aa8e01c5c9734f4d37f31035f93e9c8b850f15ec6a4480f04cf3a3a1cbf78a4e796ae1be5d6c54f7f7c91556eafee913a8d0da4
  languageName: node
  linkType: hard

"_codeblocks@workspace:.":
  version: 0.0.0-use.local
  resolution: "_codeblocks@workspace:."
  dependencies:
    "@eslint/js": "npm:^9.32.0"
    "@langchain/anthropic": "npm:^0.3.24"
    "@langchain/core": "npm:^0.3.66"
    "@langchain/langgraph": "npm:^0.3.11"
    "@langchain/langgraph-api": "npm:^0.0.52"
    "@langchain/langgraph-sdk": "npm:^0.0.102"
    "@langchain/openai": "npm:^0.6.3"
    eslint: "npm:^9.32.0"
    globals: "npm:^16.3.0"
    jiti: "npm:^2.5.1"
    typescript: "npm:^5.8.3"
    typescript-eslint: "npm:^8.38.0"
    zod: "npm:^4.0.10"
  languageName: unknown
  linkType: soft

"abbrev@npm:^3.0.0":
  version: 3.0.1
  resolution: "abbrev@npm:3.0.1"
  checksum: 10c0/21ba8f574ea57a3106d6d35623f2c4a9111d9ee3e9a5be47baed46ec2457d2eac46e07a5c4a60186f88cb98abbe3e24f2d4cca70bc2b12f1692523e2209a9ccf
  languageName: node
  linkType: hard

"acorn-jsx@npm:^5.3.2":
  version: 5.3.2
  resolution: "acorn-jsx@npm:5.3.2"
  peerDependencies:
    acorn: ^6.0.0 || ^7.0.0 || ^8.0.0
  checksum: 10c0/4c54868fbef3b8d58927d5e33f0a4de35f59012fe7b12cf9dfbb345fb8f46607709e1c4431be869a23fb63c151033d84c4198fa9f79385cec34fcb1dd53974c1
  languageName: node
  linkType: hard

"acorn@npm:^8.15.0":
  version: 8.15.0
  resolution: "acorn@npm:8.15.0"
  bin:
    acorn: bin/acorn
  checksum: 10c0/dec73ff59b7d6628a01eebaece7f2bdb8bb62b9b5926dcad0f8931f2b8b79c2be21f6c68ac095592adb5adb15831a3635d9343e6a91d028bbe85d564875ec3ec
  languageName: node
  linkType: hard

"agent-base@npm:^7.1.0, agent-base@npm:^7.1.2":
  version: 7.1.4
  resolution: "agent-base@npm:7.1.4"
  checksum: 10c0/c2c9ab7599692d594b6a161559ada307b7a624fa4c7b03e3afdb5a5e31cd0e53269115b620fcab024c5ac6a6f37fa5eb2e004f076ad30f5f7e6b8b671f7b35fe
  languageName: node
  linkType: hard

"ajv@npm:^6.12.4":
  version: 6.12.6
  resolution: "ajv@npm:6.12.6"
  dependencies:
    fast-deep-equal: "npm:^3.1.1"
    fast-json-stable-stringify: "npm:^2.0.0"
    json-schema-traverse: "npm:^0.4.1"
    uri-js: "npm:^4.2.2"
  checksum: 10c0/41e23642cbe545889245b9d2a45854ebba51cda6c778ebced9649420d9205f2efb39cb43dbc41e358409223b1ea43303ae4839db682c848b891e4811da1a5a71
  languageName: node
  linkType: hard

"ansi-regex@npm:^5.0.1":
  version: 5.0.1
  resolution: "ansi-regex@npm:5.0.1"
  checksum: 10c0/9a64bb8627b434ba9327b60c027742e5d17ac69277960d041898596271d992d4d52ba7267a63ca10232e29f6107fc8a835f6ce8d719b88c5f8493f8254813737
  languageName: node
  linkType: hard

"ansi-regex@npm:^6.0.1":
  version: 6.1.0
  resolution: "ansi-regex@npm:6.1.0"
  checksum: 10c0/a91daeddd54746338478eef88af3439a7edf30f8e23196e2d6ed182da9add559c601266dbef01c2efa46a958ad6f1f8b176799657616c702b5b02e799e7fd8dc
  languageName: node
  linkType: hard

"ansi-styles@npm:^4.0.0, ansi-styles@npm:^4.1.0":
  version: 4.3.0
  resolution: "ansi-styles@npm:4.3.0"
  dependencies:
    color-convert: "npm:^2.0.1"
  checksum: 10c0/895a23929da416f2bd3de7e9cb4eabd340949328ab85ddd6e484a637d8f6820d485f53933446f5291c3b760cbc488beb8e88573dd0f9c7daf83dccc8fe81b041
  languageName: node
  linkType: hard

"ansi-styles@npm:^5.0.0":
  version: 5.2.0
  resolution: "ansi-styles@npm:5.2.0"
  checksum: 10c0/9c4ca80eb3c2fb7b33841c210d2f20807f40865d27008d7c3f707b7f95cab7d67462a565e2388ac3285b71cb3d9bb2173de8da37c57692a362885ec34d6e27df
  languageName: node
  linkType: hard

"ansi-styles@npm:^6.1.0":
  version: 6.2.1
  resolution: "ansi-styles@npm:6.2.1"
  checksum: 10c0/5d1ec38c123984bcedd996eac680d548f31828bd679a66db2bdf11844634dde55fec3efa9c6bb1d89056a5e79c1ac540c4c784d592ea1d25028a92227d2f2d5c
  languageName: node
  linkType: hard

"argparse@npm:^2.0.1":
  version: 2.0.1
  resolution: "argparse@npm:2.0.1"
  checksum: 10c0/c5640c2d89045371c7cedd6a70212a04e360fd34d6edeae32f6952c63949e3525ea77dbec0289d8213a99bbaeab5abfa860b5c12cf88a2e6cf8106e90dd27a7e
  languageName: node
  linkType: hard

"async@npm:^3.2.3":
  version: 3.2.6
  resolution: "async@npm:3.2.6"
  checksum: 10c0/36484bb15ceddf07078688d95e27076379cc2f87b10c03b6dd8a83e89475a3c8df5848859dd06a4c95af1e4c16fc973de0171a77f18ea00be899aca2a4f85e70
  languageName: node
  linkType: hard

"autoprefixer@npm:^10.4.20":
  version: 10.4.21
  resolution: "autoprefixer@npm:10.4.21"
  dependencies:
    browserslist: "npm:^4.24.4"
    caniuse-lite: "npm:^1.0.30001702"
    fraction.js: "npm:^4.3.7"
    normalize-range: "npm:^0.1.2"
    picocolors: "npm:^1.1.1"
    postcss-value-parser: "npm:^4.2.0"
  peerDependencies:
    postcss: ^8.1.0
  bin:
    autoprefixer: bin/autoprefixer
  checksum: 10c0/de5b71d26d0baff4bbfb3d59f7cf7114a6030c9eeb66167acf49a32c5b61c68e308f1e0f869d92334436a221035d08b51cd1b2f2c4689b8d955149423c16d4d4
  languageName: node
  linkType: hard

"balanced-match@npm:^1.0.0":
  version: 1.0.2
  resolution: "balanced-match@npm:1.0.2"
  checksum: 10c0/9308baf0a7e4838a82bbfd11e01b1cb0f0cf2893bc1676c27c2a8c0e70cbae1c59120c3268517a8ae7fb6376b4639ef81ca22582611dbee4ed28df945134aaee
  languageName: node
  linkType: hard

"base64-js@npm:^1.5.1":
  version: 1.5.1
  resolution: "base64-js@npm:1.5.1"
  checksum: 10c0/f23823513b63173a001030fae4f2dabe283b99a9d324ade3ad3d148e218134676f1ee8568c877cd79ec1c53158dcf2d2ba527a97c606618928ba99dd930102bf
  languageName: node
  linkType: hard

"brace-expansion@npm:^1.1.7":
  version: 1.1.12
  resolution: "brace-expansion@npm:1.1.12"
  dependencies:
    balanced-match: "npm:^1.0.0"
    concat-map: "npm:0.0.1"
  checksum: 10c0/975fecac2bb7758c062c20d0b3b6288c7cc895219ee25f0a64a9de662dbac981ff0b6e89909c3897c1f84fa353113a721923afdec5f8b2350255b097f12b1f73
  languageName: node
  linkType: hard

"brace-expansion@npm:^2.0.1":
  version: 2.0.2
  resolution: "brace-expansion@npm:2.0.2"
  dependencies:
    balanced-match: "npm:^1.0.0"
  checksum: 10c0/6d117a4c793488af86b83172deb6af143e94c17bc53b0b3cec259733923b4ca84679d506ac261f4ba3c7ed37c46018e2ff442f9ce453af8643ecd64f4a54e6cf
  languageName: node
  linkType: hard

"braces@npm:^3.0.3":
  version: 3.0.3
  resolution: "braces@npm:3.0.3"
  dependencies:
    fill-range: "npm:^7.1.1"
  checksum: 10c0/7c6dfd30c338d2997ba77500539227b9d1f85e388a5f43220865201e407e076783d0881f2d297b9f80951b4c957fcf0b51c1d2d24227631643c3f7c284b0aa04
  languageName: node
  linkType: hard

"browserslist@npm:^4.24.4":
  version: 4.25.1
  resolution: "browserslist@npm:4.25.1"
  dependencies:
    caniuse-lite: "npm:^1.0.30001726"
    electron-to-chromium: "npm:^1.5.173"
    node-releases: "npm:^2.0.19"
    update-browserslist-db: "npm:^1.1.3"
  bin:
    browserslist: cli.js
  checksum: 10c0/acba5f0bdbd5e72dafae1e6ec79235b7bad305ed104e082ed07c34c38c7cb8ea1bc0f6be1496958c40482e40166084458fc3aee15111f15faa79212ad9081b2a
  languageName: node
  linkType: hard

"bundle-name@npm:^4.1.0":
  version: 4.1.0
  resolution: "bundle-name@npm:4.1.0"
  dependencies:
    run-applescript: "npm:^7.0.0"
  checksum: 10c0/8e575981e79c2bcf14d8b1c027a3775c095d362d1382312f444a7c861b0e21513c0bd8db5bd2b16e50ba0709fa622d4eab6b53192d222120305e68359daece29
  languageName: node
  linkType: hard

"cacache@npm:^19.0.1":
  version: 19.0.1
  resolution: "cacache@npm:19.0.1"
  dependencies:
    "@npmcli/fs": "npm:^4.0.0"
    fs-minipass: "npm:^3.0.0"
    glob: "npm:^10.2.2"
    lru-cache: "npm:^10.0.1"
    minipass: "npm:^7.0.3"
    minipass-collect: "npm:^2.0.1"
    minipass-flush: "npm:^1.0.5"
    minipass-pipeline: "npm:^1.2.4"
    p-map: "npm:^7.0.2"
    ssri: "npm:^12.0.0"
    tar: "npm:^7.4.3"
    unique-filename: "npm:^4.0.0"
  checksum: 10c0/01f2134e1bd7d3ab68be851df96c8d63b492b1853b67f2eecb2c37bb682d37cb70bb858a16f2f0554d3c0071be6dfe21456a1ff6fa4b7eed996570d6a25ffe9c
  languageName: node
  linkType: hard

"callsites@npm:^3.0.0":
  version: 3.1.0
  resolution: "callsites@npm:3.1.0"
  checksum: 10c0/fff92277400eb06c3079f9e74f3af120db9f8ea03bad0e84d9aede54bbe2d44a56cccb5f6cf12211f93f52306df87077ecec5b712794c5a9b5dac6d615a3f301
  languageName: node
  linkType: hard

"camelcase@npm:6":
  version: 6.3.0
  resolution: "camelcase@npm:6.3.0"
  checksum: 10c0/0d701658219bd3116d12da3eab31acddb3f9440790c0792e0d398f0a520a6a4058018e546862b6fba89d7ae990efaeb97da71e1913e9ebf5a8b5621a3d55c710
  languageName: node
  linkType: hard

"caniuse-lite@npm:^1.0.30001702, caniuse-lite@npm:^1.0.30001726":
  version: 1.0.30001727
  resolution: "caniuse-lite@npm:1.0.30001727"
  checksum: 10c0/f0a441c05d8925d728c2d02ce23b001935f52183a3bf669556f302568fe258d1657940c7ac0b998f92bc41383e185b390279a7d779e6d96a2b47881f56400221
  languageName: node
  linkType: hard

"chalk@npm:^4.0.0, chalk@npm:^4.1.2":
  version: 4.1.2
  resolution: "chalk@npm:4.1.2"
  dependencies:
    ansi-styles: "npm:^4.1.0"
    supports-color: "npm:^7.1.0"
  checksum: 10c0/4a3fef5cc34975c898ffe77141450f679721df9dde00f6c304353fa9c8b571929123b26a0e4617bde5018977eb655b31970c297b91b63ee83bb82aeb04666880
  languageName: node
  linkType: hard

"chownr@npm:^3.0.0":
  version: 3.0.0
  resolution: "chownr@npm:3.0.0"
  checksum: 10c0/43925b87700f7e3893296c8e9c56cc58f926411cce3a6e5898136daaf08f08b9a8eb76d37d3267e707d0dcc17aed2e2ebdf5848c0c3ce95cf910a919935c1b10
  languageName: node
  linkType: hard

"color-convert@npm:^1.9.3":
  version: 1.9.3
  resolution: "color-convert@npm:1.9.3"
  dependencies:
    color-name: "npm:1.1.3"
  checksum: 10c0/5ad3c534949a8c68fca8fbc6f09068f435f0ad290ab8b2f76841b9e6af7e0bb57b98cb05b0e19fe33f5d91e5a8611ad457e5f69e0a484caad1f7487fd0e8253c
  languageName: node
  linkType: hard

"color-convert@npm:^2.0.1":
  version: 2.0.1
  resolution: "color-convert@npm:2.0.1"
  dependencies:
    color-name: "npm:~1.1.4"
  checksum: 10c0/37e1150172f2e311fe1b2df62c6293a342ee7380da7b9cfdba67ea539909afbd74da27033208d01d6d5cfc65ee7868a22e18d7e7648e004425441c0f8a15a7d7
  languageName: node
  linkType: hard

"color-name@npm:1.1.3":
  version: 1.1.3
  resolution: "color-name@npm:1.1.3"
  checksum: 10c0/566a3d42cca25b9b3cd5528cd7754b8e89c0eb646b7f214e8e2eaddb69994ac5f0557d9c175eb5d8f0ad73531140d9c47525085ee752a91a2ab15ab459caf6d6
  languageName: node
  linkType: hard

"color-name@npm:^1.0.0, color-name@npm:~1.1.4":
  version: 1.1.4
  resolution: "color-name@npm:1.1.4"
  checksum: 10c0/a1a3f914156960902f46f7f56bc62effc6c94e84b2cae157a526b1c1f74b677a47ec602bf68a61abfa2b42d15b7c5651c6dbe72a43af720bc588dff885b10f95
  languageName: node
  linkType: hard

"color-string@npm:^1.6.0":
  version: 1.9.1
  resolution: "color-string@npm:1.9.1"
  dependencies:
    color-name: "npm:^1.0.0"
    simple-swizzle: "npm:^0.2.2"
  checksum: 10c0/b0bfd74c03b1f837f543898b512f5ea353f71630ccdd0d66f83028d1f0924a7d4272deb278b9aef376cacf1289b522ac3fb175e99895283645a2dc3a33af2404
  languageName: node
  linkType: hard

"color@npm:^3.1.3":
  version: 3.2.1
  resolution: "color@npm:3.2.1"
  dependencies:
    color-convert: "npm:^1.9.3"
    color-string: "npm:^1.6.0"
  checksum: 10c0/39345d55825884c32a88b95127d417a2c24681d8b57069413596d9fcbb721459ef9d9ec24ce3e65527b5373ce171b73e38dbcd9c830a52a6487e7f37bf00e83c
  languageName: node
  linkType: hard

"colors@npm:^1.4.0":
  version: 1.4.0
  resolution: "colors@npm:1.4.0"
  checksum: 10c0/9af357c019da3c5a098a301cf64e3799d27549d8f185d86f79af23069e4f4303110d115da98483519331f6fb71c8568d5688fa1c6523600044fd4a54e97c4efb
  languageName: node
  linkType: hard

"colorspace@npm:1.1.x":
  version: 1.1.4
  resolution: "colorspace@npm:1.1.4"
  dependencies:
    color: "npm:^3.1.3"
    text-hex: "npm:1.0.x"
  checksum: 10c0/af5f91ff7f8e146b96e439ac20ed79b197210193bde721b47380a75b21751d90fa56390c773bb67c0aedd34ff85091883a437ab56861c779bd507d639ba7e123
  languageName: node
  linkType: hard

"commander@npm:^13.0.0":
  version: 13.1.0
  resolution: "commander@npm:13.1.0"
  checksum: 10c0/7b8c5544bba704fbe84b7cab2e043df8586d5c114a4c5b607f83ae5060708940ed0b5bd5838cf8ce27539cde265c1cbd59ce3c8c6b017ed3eec8943e3a415164
  languageName: node
  linkType: hard

"concat-map@npm:0.0.1":
  version: 0.0.1
  resolution: "concat-map@npm:0.0.1"
  checksum: 10c0/c996b1cfdf95b6c90fee4dae37e332c8b6eb7d106430c17d538034c0ad9a1630cb194d2ab37293b1bdd4d779494beee7786d586a50bd9376fd6f7bcc2bd4c98f
  languageName: node
  linkType: hard

"console-table-printer@npm:^2.12.1":
  version: 2.14.6
  resolution: "console-table-printer@npm:2.14.6"
  dependencies:
    simple-wcswidth: "npm:^1.0.1"
  checksum: 10c0/af4f7f18d2a70130ea9fd76ffd9c56351329665fe3bec96cfab26d71dd2de6b983b09ec163c9346c72fa6fb7fdce350e09ee132b1c89db83ac50b23742cdb10f
  languageName: node
  linkType: hard

"copy-anything@npm:^3.0.2":
  version: 3.0.5
  resolution: "copy-anything@npm:3.0.5"
  dependencies:
    is-what: "npm:^4.1.8"
  checksum: 10c0/01eadd500c7e1db71d32d95a3bfaaedcb839ef891c741f6305ab0461398056133de08f2d1bf4c392b364e7bdb7ce498513896e137a7a183ac2516b065c28a4fe
  languageName: node
  linkType: hard

"cross-spawn@npm:^7.0.6":
  version: 7.0.6
  resolution: "cross-spawn@npm:7.0.6"
  dependencies:
    path-key: "npm:^3.1.0"
    shebang-command: "npm:^2.0.0"
    which: "npm:^2.0.1"
  checksum: 10c0/053ea8b2135caff68a9e81470e845613e374e7309a47731e81639de3eaeb90c3d01af0e0b44d2ab9d50b43467223b88567dfeb3262db942dc063b9976718ffc1
  languageName: node
  linkType: hard

"cssesc@npm:^3.0.0":
  version: 3.0.0
  resolution: "cssesc@npm:3.0.0"
  bin:
    cssesc: bin/cssesc
  checksum: 10c0/6bcfd898662671be15ae7827120472c5667afb3d7429f1f917737f3bf84c4176003228131b643ae74543f17a394446247df090c597bb9a728cce298606ed0aa7
  languageName: node
  linkType: hard

"debug@npm:4, debug@npm:^4.1.1, debug@npm:^4.3.1, debug@npm:^4.3.2, debug@npm:^4.3.4":
  version: 4.4.1
  resolution: "debug@npm:4.4.1"
  dependencies:
    ms: "npm:^2.1.3"
  peerDependenciesMeta:
    supports-color:
      optional: true
  checksum: 10c0/d2b44bc1afd912b49bb7ebb0d50a860dc93a4dd7d946e8de94abc957bb63726b7dd5aa48c18c2386c379ec024c46692e15ed3ed97d481729f929201e671fcd55
  languageName: node
  linkType: hard

"decamelize@npm:1.2.0":
  version: 1.2.0
  resolution: "decamelize@npm:1.2.0"
  checksum: 10c0/85c39fe8fbf0482d4a1e224ef0119db5c1897f8503bcef8b826adff7a1b11414972f6fef2d7dec2ee0b4be3863cf64ac1439137ae9e6af23a3d8dcbe26a5b4b2
  languageName: node
  linkType: hard

"dedent@npm:^1.5.3":
  version: 1.6.0
  resolution: "dedent@npm:1.6.0"
  peerDependencies:
    babel-plugin-macros: ^3.1.0
  peerDependenciesMeta:
    babel-plugin-macros:
      optional: true
  checksum: 10c0/671b8f5e390dd2a560862c4511dd6d2638e71911486f78cb32116551f8f2aa6fcaf50579ffffb2f866d46b5b80fd72470659ca5760ede8f967619ef7df79e8a5
  languageName: node
  linkType: hard

"deep-is@npm:^0.1.3":
  version: 0.1.4
  resolution: "deep-is@npm:0.1.4"
  checksum: 10c0/7f0ee496e0dff14a573dc6127f14c95061b448b87b995fc96c017ce0a1e66af1675e73f1d6064407975bc4ea6ab679497a29fff7b5b9c4e99cb10797c1ad0b4c
  languageName: node
  linkType: hard

"default-browser-id@npm:^5.0.0":
  version: 5.0.0
  resolution: "default-browser-id@npm:5.0.0"
  checksum: 10c0/957fb886502594c8e645e812dfe93dba30ed82e8460d20ce39c53c5b0f3e2afb6ceaec2249083b90bdfbb4cb0f34e1f73fde3d68cac00becdbcfd894156b5ead
  languageName: node
  linkType: hard

"default-browser@npm:^5.2.1":
  version: 5.2.1
  resolution: "default-browser@npm:5.2.1"
  dependencies:
    bundle-name: "npm:^4.1.0"
    default-browser-id: "npm:^5.0.0"
  checksum: 10c0/73f17dc3c58026c55bb5538749597db31f9561c0193cd98604144b704a981c95a466f8ecc3c2db63d8bfd04fb0d426904834cfc91ae510c6aeb97e13c5167c4d
  languageName: node
  linkType: hard

"define-lazy-prop@npm:^3.0.0":
  version: 3.0.0
  resolution: "define-lazy-prop@npm:3.0.0"
  checksum: 10c0/5ab0b2bf3fa58b3a443140bbd4cd3db1f91b985cc8a246d330b9ac3fc0b6a325a6d82bddc0b055123d745b3f9931afeea74a5ec545439a1630b9c8512b0eeb49
  languageName: node
  linkType: hard

"detect-libc@npm:^2.0.3, detect-libc@npm:^2.0.4":
  version: 2.0.4
  resolution: "detect-libc@npm:2.0.4"
  checksum: 10c0/c15541f836eba4b1f521e4eecc28eefefdbc10a94d3b8cb4c507689f332cc111babb95deda66f2de050b22122113189986d5190be97d51b5a2b23b938415e67c
  languageName: node
  linkType: hard

"dotenv@npm:^16.4.7":
  version: 16.6.1
  resolution: "dotenv@npm:16.6.1"
  checksum: 10c0/15ce56608326ea0d1d9414a5c8ee6dcf0fffc79d2c16422b4ac2268e7e2d76ff5a572d37ffe747c377de12005f14b3cc22361e79fc7f1061cce81f77d2c973dc
  languageName: node
  linkType: hard

"eastasianwidth@npm:^0.2.0":
  version: 0.2.0
  resolution: "eastasianwidth@npm:0.2.0"
  checksum: 10c0/26f364ebcdb6395f95124fda411f63137a4bfb5d3a06453f7f23dfe52502905bd84e0488172e0f9ec295fdc45f05c23d5d91baf16bd26f0fe9acd777a188dc39
  languageName: node
  linkType: hard

"electron-to-chromium@npm:^1.5.173":
  version: 1.5.191
  resolution: "electron-to-chromium@npm:1.5.191"
  checksum: 10c0/26b22ec2ae2a152da09f062d8582e54384a15ddc2a27149cdc2747a0c3f46154370a37b9e687de2d6d71ea1ebc1319f8394283ffb1581f1d4495cdefffd7a2a6
  languageName: node
  linkType: hard

"emoji-regex@npm:^8.0.0":
  version: 8.0.0
  resolution: "emoji-regex@npm:8.0.0"
  checksum: 10c0/b6053ad39951c4cf338f9092d7bfba448cdfd46fe6a2a034700b149ac9ffbc137e361cbd3c442297f86bed2e5f7576c1b54cc0a6bf8ef5106cc62f496af35010
  languageName: node
  linkType: hard

"emoji-regex@npm:^9.2.2":
  version: 9.2.2
  resolution: "emoji-regex@npm:9.2.2"
  checksum: 10c0/af014e759a72064cf66e6e694a7fc6b0ed3d8db680427b021a89727689671cefe9d04151b2cad51dbaf85d5ba790d061cd167f1cf32eb7b281f6368b3c181639
  languageName: node
  linkType: hard

"enabled@npm:2.0.x":
  version: 2.0.0
  resolution: "enabled@npm:2.0.0"
  checksum: 10c0/3b2c2af9bc7f8b9e291610f2dde4a75cf6ee52a68f4dd585482fbdf9a55d65388940e024e56d40bb03e05ef6671f5f53021fa8b72a20e954d7066ec28166713f
  languageName: node
  linkType: hard

"encoding@npm:^0.1.13":
  version: 0.1.13
  resolution: "encoding@npm:0.1.13"
  dependencies:
    iconv-lite: "npm:^0.6.2"
  checksum: 10c0/36d938712ff00fe1f4bac88b43bcffb5930c1efa57bbcdca9d67e1d9d6c57cfb1200fb01efe0f3109b2ce99b231f90779532814a81370a1bd3274a0f58585039
  languageName: node
  linkType: hard

"enhanced-resolve@npm:^5.18.1":
  version: 5.18.2
  resolution: "enhanced-resolve@npm:5.18.2"
  dependencies:
    graceful-fs: "npm:^4.2.4"
    tapable: "npm:^2.2.0"
  checksum: 10c0/2a45105daded694304b0298d1c0351a981842249a9867513d55e41321a4ccf37dfd35b0c1e9ceae290eab73654b09aa7a910d618ea6f9441e97c52bc424a2372
  languageName: node
  linkType: hard

"env-paths@npm:^2.2.0":
  version: 2.2.1
  resolution: "env-paths@npm:2.2.1"
  checksum: 10c0/285325677bf00e30845e330eec32894f5105529db97496ee3f598478e50f008c5352a41a30e5e72ec9de8a542b5a570b85699cd63bd2bc646dbcb9f311d83bc4
  languageName: node
  linkType: hard

"err-code@npm:^2.0.2":
  version: 2.0.3
  resolution: "err-code@npm:2.0.3"
  checksum: 10c0/b642f7b4dd4a376e954947550a3065a9ece6733ab8e51ad80db727aaae0817c2e99b02a97a3d6cecc648a97848305e728289cf312d09af395403a90c9d4d8a66
  languageName: node
  linkType: hard

"esbuild-plugin-tailwindcss@npm:^2.0.1":
  version: 2.0.1
  resolution: "esbuild-plugin-tailwindcss@npm:2.0.1"
  dependencies:
    "@tailwindcss/postcss": "npm:^4.0.5"
    autoprefixer: "npm:^10.4.20"
    postcss: "npm:^8.5.1"
    postcss-modules: "npm:^6.0.1"
  checksum: 10c0/74a7ea474c87f4b99cd31873bc56ca269d48990bd9084d473c71d81503b2cebd365062316e8647c3d293425377b315578268d19769c4c3b461bc05e7b5417b6c
  languageName: node
  linkType: hard

"esbuild@npm:^0.25.0, esbuild@npm:~0.25.0":
  version: 0.25.8
  resolution: "esbuild@npm:0.25.8"
  dependencies:
    "@esbuild/aix-ppc64": "npm:0.25.8"
    "@esbuild/android-arm": "npm:0.25.8"
    "@esbuild/android-arm64": "npm:0.25.8"
    "@esbuild/android-x64": "npm:0.25.8"
    "@esbuild/darwin-arm64": "npm:0.25.8"
    "@esbuild/darwin-x64": "npm:0.25.8"
    "@esbuild/freebsd-arm64": "npm:0.25.8"
    "@esbuild/freebsd-x64": "npm:0.25.8"
    "@esbuild/linux-arm": "npm:0.25.8"
    "@esbuild/linux-arm64": "npm:0.25.8"
    "@esbuild/linux-ia32": "npm:0.25.8"
    "@esbuild/linux-loong64": "npm:0.25.8"
    "@esbuild/linux-mips64el": "npm:0.25.8"
    "@esbuild/linux-ppc64": "npm:0.25.8"
    "@esbuild/linux-riscv64": "npm:0.25.8"
    "@esbuild/linux-s390x": "npm:0.25.8"
    "@esbuild/linux-x64": "npm:0.25.8"
    "@esbuild/netbsd-arm64": "npm:0.25.8"
    "@esbuild/netbsd-x64": "npm:0.25.8"
    "@esbuild/openbsd-arm64": "npm:0.25.8"
    "@esbuild/openbsd-x64": "npm:0.25.8"
    "@esbuild/openharmony-arm64": "npm:0.25.8"
    "@esbuild/sunos-x64": "npm:0.25.8"
    "@esbuild/win32-arm64": "npm:0.25.8"
    "@esbuild/win32-ia32": "npm:0.25.8"
    "@esbuild/win32-x64": "npm:0.25.8"
  dependenciesMeta:
    "@esbuild/aix-ppc64":
      optional: true
    "@esbuild/android-arm":
      optional: true
    "@esbuild/android-arm64":
      optional: true
    "@esbuild/android-x64":
      optional: true
    "@esbuild/darwin-arm64":
      optional: true
    "@esbuild/darwin-x64":
      optional: true
    "@esbuild/freebsd-arm64":
      optional: true
    "@esbuild/freebsd-x64":
      optional: true
    "@esbuild/linux-arm":
      optional: true
    "@esbuild/linux-arm64":
      optional: true
    "@esbuild/linux-ia32":
      optional: true
    "@esbuild/linux-loong64":
      optional: true
    "@esbuild/linux-mips64el":
      optional: true
    "@esbuild/linux-ppc64":
      optional: true
    "@esbuild/linux-riscv64":
      optional: true
    "@esbuild/linux-s390x":
      optional: true
    "@esbuild/linux-x64":
      optional: true
    "@esbuild/netbsd-arm64":
      optional: true
    "@esbuild/netbsd-x64":
      optional: true
    "@esbuild/openbsd-arm64":
      optional: true
    "@esbuild/openbsd-x64":
      optional: true
    "@esbuild/openharmony-arm64":
      optional: true
    "@esbuild/sunos-x64":
      optional: true
    "@esbuild/win32-arm64":
      optional: true
    "@esbuild/win32-ia32":
      optional: true
    "@esbuild/win32-x64":
      optional: true
  bin:
    esbuild: bin/esbuild
  checksum: 10c0/43747a25e120d5dd9ce75c82f57306580d715647c8db4f4a0a84e73b04cf16c27572d3937d3cfb95d5ac3266a4d1bbd3913e3d76ae719693516289fc86f8a5fd
  languageName: node
  linkType: hard

"escalade@npm:^3.2.0":
  version: 3.2.0
  resolution: "escalade@npm:3.2.0"
  checksum: 10c0/ced4dd3a78e15897ed3be74e635110bbf3b08877b0a41be50dcb325ee0e0b5f65fc2d50e9845194d7c4633f327e2e1c6cce00a71b617c5673df0374201d67f65
  languageName: node
  linkType: hard

"escape-string-regexp@npm:^4.0.0":
  version: 4.0.0
  resolution: "escape-string-regexp@npm:4.0.0"
  checksum: 10c0/9497d4dd307d845bd7f75180d8188bb17ea8c151c1edbf6b6717c100e104d629dc2dfb687686181b0f4b7d732c7dfdc4d5e7a8ff72de1b0ca283a75bbb3a9cd9
  languageName: node
  linkType: hard

"eslint-scope@npm:^8.4.0":
  version: 8.4.0
  resolution: "eslint-scope@npm:8.4.0"
  dependencies:
    esrecurse: "npm:^4.3.0"
    estraverse: "npm:^5.2.0"
  checksum: 10c0/407f6c600204d0f3705bd557f81bd0189e69cd7996f408f8971ab5779c0af733d1af2f1412066b40ee1588b085874fc37a2333986c6521669cdbdd36ca5058e0
  languageName: node
  linkType: hard

"eslint-visitor-keys@npm:^3.4.3":
  version: 3.4.3
  resolution: "eslint-visitor-keys@npm:3.4.3"
  checksum: 10c0/92708e882c0a5ffd88c23c0b404ac1628cf20104a108c745f240a13c332a11aac54f49a22d5762efbffc18ecbc9a580d1b7ad034bf5f3cc3307e5cbff2ec9820
  languageName: node
  linkType: hard

"eslint-visitor-keys@npm:^4.2.1":
  version: 4.2.1
  resolution: "eslint-visitor-keys@npm:4.2.1"
  checksum: 10c0/fcd43999199d6740db26c58dbe0c2594623e31ca307e616ac05153c9272f12f1364f5a0b1917a8e962268fdecc6f3622c1c2908b4fcc2e047a106fe6de69dc43
  languageName: node
  linkType: hard

"eslint@npm:^9.32.0":
  version: 9.32.0
  resolution: "eslint@npm:9.32.0"
  dependencies:
    "@eslint-community/eslint-utils": "npm:^4.2.0"
    "@eslint-community/regexpp": "npm:^4.12.1"
    "@eslint/config-array": "npm:^0.21.0"
    "@eslint/config-helpers": "npm:^0.3.0"
    "@eslint/core": "npm:^0.15.0"
    "@eslint/eslintrc": "npm:^3.3.1"
    "@eslint/js": "npm:9.32.0"
    "@eslint/plugin-kit": "npm:^0.3.4"
    "@humanfs/node": "npm:^0.16.6"
    "@humanwhocodes/module-importer": "npm:^1.0.1"
    "@humanwhocodes/retry": "npm:^0.4.2"
    "@types/estree": "npm:^1.0.6"
    "@types/json-schema": "npm:^7.0.15"
    ajv: "npm:^6.12.4"
    chalk: "npm:^4.0.0"
    cross-spawn: "npm:^7.0.6"
    debug: "npm:^4.3.2"
    escape-string-regexp: "npm:^4.0.0"
    eslint-scope: "npm:^8.4.0"
    eslint-visitor-keys: "npm:^4.2.1"
    espree: "npm:^10.4.0"
    esquery: "npm:^1.5.0"
    esutils: "npm:^2.0.2"
    fast-deep-equal: "npm:^3.1.3"
    file-entry-cache: "npm:^8.0.0"
    find-up: "npm:^5.0.0"
    glob-parent: "npm:^6.0.2"
    ignore: "npm:^5.2.0"
    imurmurhash: "npm:^0.1.4"
    is-glob: "npm:^4.0.0"
    json-stable-stringify-without-jsonify: "npm:^1.0.1"
    lodash.merge: "npm:^4.6.2"
    minimatch: "npm:^3.1.2"
    natural-compare: "npm:^1.4.0"
    optionator: "npm:^0.9.3"
  peerDependencies:
    jiti: "*"
  peerDependenciesMeta:
    jiti:
      optional: true
  bin:
    eslint: bin/eslint.js
  checksum: 10c0/e8a23924ec5f8b62e95483002ca25db74e25c23bd9c6d98a9f656ee32f820169bee3bfdf548ec728b16694f198b3db857d85a49210ee4a035242711d08fdc602
  languageName: node
  linkType: hard

"espree@npm:^10.0.1, espree@npm:^10.4.0":
  version: 10.4.0
  resolution: "espree@npm:10.4.0"
  dependencies:
    acorn: "npm:^8.15.0"
    acorn-jsx: "npm:^5.3.2"
    eslint-visitor-keys: "npm:^4.2.1"
  checksum: 10c0/c63fe06131c26c8157b4083313cb02a9a54720a08e21543300e55288c40e06c3fc284bdecf108d3a1372c5934a0a88644c98714f38b6ae8ed272b40d9ea08d6b
  languageName: node
  linkType: hard

"esquery@npm:^1.5.0":
  version: 1.6.0
  resolution: "esquery@npm:1.6.0"
  dependencies:
    estraverse: "npm:^5.1.0"
  checksum: 10c0/cb9065ec605f9da7a76ca6dadb0619dfb611e37a81e318732977d90fab50a256b95fee2d925fba7c2f3f0523aa16f91587246693bc09bc34d5a59575fe6e93d2
  languageName: node
  linkType: hard

"esrecurse@npm:^4.3.0":
  version: 4.3.0
  resolution: "esrecurse@npm:4.3.0"
  dependencies:
    estraverse: "npm:^5.2.0"
  checksum: 10c0/81a37116d1408ded88ada45b9fb16dbd26fba3aadc369ce50fcaf82a0bac12772ebd7b24cd7b91fc66786bf2c1ac7b5f196bc990a473efff972f5cb338877cf5
  languageName: node
  linkType: hard

"estraverse@npm:^5.1.0, estraverse@npm:^5.2.0":
  version: 5.3.0
  resolution: "estraverse@npm:5.3.0"
  checksum: 10c0/1ff9447b96263dec95d6d67431c5e0771eb9776427421260a3e2f0fdd5d6bd4f8e37a7338f5ad2880c9f143450c9b1e4fc2069060724570a49cf9cf0312bd107
  languageName: node
  linkType: hard

"esutils@npm:^2.0.2":
  version: 2.0.3
  resolution: "esutils@npm:2.0.3"
  checksum: 10c0/9a2fe69a41bfdade834ba7c42de4723c97ec776e40656919c62cbd13607c45e127a003f05f724a1ea55e5029a4cf2de444b13009f2af71271e42d93a637137c7
  languageName: node
  linkType: hard

"eventemitter3@npm:^4.0.4":
  version: 4.0.7
  resolution: "eventemitter3@npm:4.0.7"
  checksum: 10c0/5f6d97cbcbac47be798e6355e3a7639a84ee1f7d9b199a07017f1d2f1e2fe236004d14fa5dfaeba661f94ea57805385e326236a6debbc7145c8877fbc0297c6b
  languageName: node
  linkType: hard

"exit-hook@npm:^4.0.0":
  version: 4.0.0
  resolution: "exit-hook@npm:4.0.0"
  checksum: 10c0/7fb33eaeb9050aee9479da9c93d42b796fb409c40e1d2b6ea2f40786ae7d7db6dc6a0f6ecc7bc24e479f957b7844bcb880044ded73320334743c64e3ecef48d7
  languageName: node
  linkType: hard

"exponential-backoff@npm:^3.1.1":
  version: 3.1.2
  resolution: "exponential-backoff@npm:3.1.2"
  checksum: 10c0/d9d3e1eafa21b78464297df91f1776f7fbaa3d5e3f7f0995648ca5b89c069d17055033817348d9f4a43d1c20b0eab84f75af6991751e839df53e4dfd6f22e844
  languageName: node
  linkType: hard

"fast-deep-equal@npm:^3.1.1, fast-deep-equal@npm:^3.1.3":
  version: 3.1.3
  resolution: "fast-deep-equal@npm:3.1.3"
  checksum: 10c0/40dedc862eb8992c54579c66d914635afbec43350afbbe991235fdcb4e3a8d5af1b23ae7e79bef7d4882d0ecee06c3197488026998fb19f72dc95acff1d1b1d0
  languageName: node
  linkType: hard

"fast-glob@npm:^3.3.2":
  version: 3.3.3
  resolution: "fast-glob@npm:3.3.3"
  dependencies:
    "@nodelib/fs.stat": "npm:^2.0.2"
    "@nodelib/fs.walk": "npm:^1.2.3"
    glob-parent: "npm:^5.1.2"
    merge2: "npm:^1.3.0"
    micromatch: "npm:^4.0.8"
  checksum: 10c0/f6aaa141d0d3384cf73cbcdfc52f475ed293f6d5b65bfc5def368b09163a9f7e5ec2b3014d80f733c405f58e470ee0cc451c2937685045cddcdeaa24199c43fe
  languageName: node
  linkType: hard

"fast-json-stable-stringify@npm:^2.0.0":
  version: 2.1.0
  resolution: "fast-json-stable-stringify@npm:2.1.0"
  checksum: 10c0/7f081eb0b8a64e0057b3bb03f974b3ef00135fbf36c1c710895cd9300f13c94ba809bb3a81cf4e1b03f6e5285610a61abbd7602d0652de423144dfee5a389c9b
  languageName: node
  linkType: hard

"fast-levenshtein@npm:^2.0.6":
  version: 2.0.6
  resolution: "fast-levenshtein@npm:2.0.6"
  checksum: 10c0/111972b37338bcb88f7d9e2c5907862c280ebf4234433b95bc611e518d192ccb2d38119c4ac86e26b668d75f7f3894f4ff5c4982899afced7ca78633b08287c4
  languageName: node
  linkType: hard

"fast-xml-parser@npm:^4.4.1":
  version: 4.5.3
  resolution: "fast-xml-parser@npm:4.5.3"
  dependencies:
    strnum: "npm:^1.1.1"
  bin:
    fxparser: src/cli/cli.js
  checksum: 10c0/bf9ccadacfadc95f6e3f0e7882a380a7f219cf0a6f96575149f02cb62bf44c3b7f0daee75b8ff3847bcfd7fbcb201e402c71045936c265cf6d94b141ec4e9327
  languageName: node
  linkType: hard

"fastq@npm:^1.6.0":
  version: 1.19.1
  resolution: "fastq@npm:1.19.1"
  dependencies:
    reusify: "npm:^1.0.4"
  checksum: 10c0/ebc6e50ac7048daaeb8e64522a1ea7a26e92b3cee5cd1c7f2316cdca81ba543aa40a136b53891446ea5c3a67ec215fbaca87ad405f102dd97012f62916905630
  languageName: node
  linkType: hard

"fdir@npm:^6.4.4":
  version: 6.4.6
  resolution: "fdir@npm:6.4.6"
  peerDependencies:
    picomatch: ^3 || ^4
  peerDependenciesMeta:
    picomatch:
      optional: true
  checksum: 10c0/45b559cff889934ebb8bc498351e5acba40750ada7e7d6bde197768d2fa67c149be8ae7f8ff34d03f4e1eb20f2764116e56440aaa2f6689e9a4aa7ef06acafe9
  languageName: node
  linkType: hard

"fecha@npm:^4.2.0":
  version: 4.2.3
  resolution: "fecha@npm:4.2.3"
  checksum: 10c0/0e895965959cf6a22bb7b00f0bf546f2783836310f510ddf63f463e1518d4c96dec61ab33fdfd8e79a71b4856a7c865478ce2ee8498d560fe125947703c9b1cf
  languageName: node
  linkType: hard

"file-entry-cache@npm:^8.0.0":
  version: 8.0.0
  resolution: "file-entry-cache@npm:8.0.0"
  dependencies:
    flat-cache: "npm:^4.0.0"
  checksum: 10c0/9e2b5938b1cd9b6d7e3612bdc533afd4ac17b2fc646569e9a8abbf2eb48e5eb8e316bc38815a3ef6a1b456f4107f0d0f055a614ca613e75db6bf9ff4d72c1638
  languageName: node
  linkType: hard

"fill-range@npm:^7.1.1":
  version: 7.1.1
  resolution: "fill-range@npm:7.1.1"
  dependencies:
    to-regex-range: "npm:^5.0.1"
  checksum: 10c0/b75b691bbe065472f38824f694c2f7449d7f5004aa950426a2c28f0306c60db9b880c0b0e4ed819997ffb882d1da02cfcfc819bddc94d71627f5269682edf018
  languageName: node
  linkType: hard

"find-up@npm:^5.0.0":
  version: 5.0.0
  resolution: "find-up@npm:5.0.0"
  dependencies:
    locate-path: "npm:^6.0.0"
    path-exists: "npm:^4.0.0"
  checksum: 10c0/062c5a83a9c02f53cdd6d175a37ecf8f87ea5bbff1fdfb828f04bfa021441bc7583e8ebc0872a4c1baab96221fb8a8a275a19809fb93fbc40bd69ec35634069a
  languageName: node
  linkType: hard

"flat-cache@npm:^4.0.0":
  version: 4.0.1
  resolution: "flat-cache@npm:4.0.1"
  dependencies:
    flatted: "npm:^3.2.9"
    keyv: "npm:^4.5.4"
  checksum: 10c0/2c59d93e9faa2523e4fda6b4ada749bed432cfa28c8e251f33b25795e426a1c6dbada777afb1f74fcfff33934fdbdea921ee738fcc33e71adc9d6eca984a1cfc
  languageName: node
  linkType: hard

"flatted@npm:^3.2.9":
  version: 3.3.3
  resolution: "flatted@npm:3.3.3"
  checksum: 10c0/e957a1c6b0254aa15b8cce8533e24165abd98fadc98575db082b786b5da1b7d72062b81bfdcd1da2f4d46b6ed93bec2434e62333e9b4261d79ef2e75a10dd538
  languageName: node
  linkType: hard

"fn.name@npm:1.x.x":
  version: 1.1.0
  resolution: "fn.name@npm:1.1.0"
  checksum: 10c0/8ad62aa2d4f0b2a76d09dba36cfec61c540c13a0fd72e5d94164e430f987a7ce6a743112bbeb14877c810ef500d1f73d7f56e76d029d2e3413f20d79e3460a9a
  languageName: node
  linkType: hard

"foreground-child@npm:^3.1.0":
  version: 3.3.1
  resolution: "foreground-child@npm:3.3.1"
  dependencies:
    cross-spawn: "npm:^7.0.6"
    signal-exit: "npm:^4.0.1"
  checksum: 10c0/8986e4af2430896e65bc2788d6679067294d6aee9545daefc84923a0a4b399ad9c7a3ea7bd8c0b2b80fdf4a92de4c69df3f628233ff3224260e9c1541a9e9ed3
  languageName: node
  linkType: hard

"fraction.js@npm:^4.3.7":
  version: 4.3.7
  resolution: "fraction.js@npm:4.3.7"
  checksum: 10c0/df291391beea9ab4c263487ffd9d17fed162dbb736982dee1379b2a8cc94e4e24e46ed508c6d278aded9080ba51872f1bc5f3a5fd8d7c74e5f105b508ac28711
  languageName: node
  linkType: hard

"fs-minipass@npm:^3.0.0":
  version: 3.0.3
  resolution: "fs-minipass@npm:3.0.3"
  dependencies:
    minipass: "npm:^7.0.3"
  checksum: 10c0/63e80da2ff9b621e2cb1596abcb9207f1cf82b968b116ccd7b959e3323144cce7fb141462200971c38bbf2ecca51695069db45265705bed09a7cd93ae5b89f94
  languageName: node
  linkType: hard

"fsevents@npm:~2.3.3":
  version: 2.3.3
  resolution: "fsevents@npm:2.3.3"
  dependencies:
    node-gyp: "npm:latest"
  checksum: 10c0/a1f0c44595123ed717febbc478aa952e47adfc28e2092be66b8ab1635147254ca6cfe1df792a8997f22716d4cbafc73309899ff7bfac2ac3ad8cf2e4ecc3ec60
  conditions: os=darwin
  languageName: node
  linkType: hard

"fsevents@patch:fsevents@npm%3A~2.3.3#optional!builtin<compat/fsevents>":
  version: 2.3.3
  resolution: "fsevents@patch:fsevents@npm%3A2.3.3#optional!builtin<compat/fsevents>::version=2.3.3&hash=df0bf1"
  dependencies:
    node-gyp: "npm:latest"
  conditions: os=darwin
  languageName: node
  linkType: hard

"generic-names@npm:^4.0.0":
  version: 4.0.0
  resolution: "generic-names@npm:4.0.0"
  dependencies:
    loader-utils: "npm:^3.2.0"
  checksum: 10c0/4e2be864535fadceed4e803fefc1df7f85447d9479d51e611a8a43a2c96533422b62c8fae84d9eb10cc21ee3de569a8c29d5ba68978ae930cccc9cb43b9a36d1
  languageName: node
  linkType: hard

"get-tsconfig@npm:^4.7.5":
  version: 4.10.1
  resolution: "get-tsconfig@npm:4.10.1"
  dependencies:
    resolve-pkg-maps: "npm:^1.0.0"
  checksum: 10c0/7f8e3dabc6a49b747920a800fb88e1952fef871cdf51b79e98db48275a5de6cdaf499c55ee67df5fa6fe7ce65f0063e26de0f2e53049b408c585aa74d39ffa21
  languageName: node
  linkType: hard

"glob-parent@npm:^5.1.2":
  version: 5.1.2
  resolution: "glob-parent@npm:5.1.2"
  dependencies:
    is-glob: "npm:^4.0.1"
  checksum: 10c0/cab87638e2112bee3f839ef5f6e0765057163d39c66be8ec1602f3823da4692297ad4e972de876ea17c44d652978638d2fd583c6713d0eb6591706825020c9ee
  languageName: node
  linkType: hard

"glob-parent@npm:^6.0.2":
  version: 6.0.2
  resolution: "glob-parent@npm:6.0.2"
  dependencies:
    is-glob: "npm:^4.0.3"
  checksum: 10c0/317034d88654730230b3f43bb7ad4f7c90257a426e872ea0bf157473ac61c99bf5d205fad8f0185f989be8d2fa6d3c7dce1645d99d545b6ea9089c39f838e7f8
  languageName: node
  linkType: hard

"glob@npm:^10.2.2":
  version: 10.4.5
  resolution: "glob@npm:10.4.5"
  dependencies:
    foreground-child: "npm:^3.1.0"
    jackspeak: "npm:^3.1.2"
    minimatch: "npm:^9.0.4"
    minipass: "npm:^7.1.2"
    package-json-from-dist: "npm:^1.0.0"
    path-scurry: "npm:^1.11.1"
  bin:
    glob: dist/esm/bin.mjs
  checksum: 10c0/19a9759ea77b8e3ca0a43c2f07ecddc2ad46216b786bb8f993c445aee80d345925a21e5280c7b7c6c59e860a0154b84e4b2b60321fea92cd3c56b4a7489f160e
  languageName: node
  linkType: hard

"globals@npm:^14.0.0":
  version: 14.0.0
  resolution: "globals@npm:14.0.0"
  checksum: 10c0/b96ff42620c9231ad468d4c58ff42afee7777ee1c963013ff8aabe095a451d0ceeb8dcd8ef4cbd64d2538cef45f787a78ba3a9574f4a634438963e334471302d
  languageName: node
  linkType: hard

"globals@npm:^16.3.0":
  version: 16.3.0
  resolution: "globals@npm:16.3.0"
  checksum: 10c0/c62dc20357d1c0bf2be4545d6c4141265d1a229bf1c3294955efb5b5ef611145391895e3f2729f8603809e81b30b516c33e6c2597573844449978606aad6eb38
  languageName: node
  linkType: hard

"graceful-fs@npm:^4.2.4, graceful-fs@npm:^4.2.6":
  version: 4.2.11
  resolution: "graceful-fs@npm:4.2.11"
  checksum: 10c0/386d011a553e02bc594ac2ca0bd6d9e4c22d7fa8cfbfc448a6d148c59ea881b092db9dbe3547ae4b88e55f1b01f7c4a2ecc53b310c042793e63aa44cf6c257f2
  languageName: node
  linkType: hard

"graphemer@npm:^1.4.0":
  version: 1.4.0
  resolution: "graphemer@npm:1.4.0"
  checksum: 10c0/e951259d8cd2e0d196c72ec711add7115d42eb9a8146c8eeda5b8d3ac91e5dd816b9cd68920726d9fd4490368e7ed86e9c423f40db87e2d8dfafa00fa17c3a31
  languageName: node
  linkType: hard

"has-flag@npm:^4.0.0":
  version: 4.0.0
  resolution: "has-flag@npm:4.0.0"
  checksum: 10c0/2e789c61b7888d66993e14e8331449e525ef42aac53c627cc53d1c3334e768bcb6abdc4f5f0de1478a25beec6f0bd62c7549058b7ac53e924040d4f301f02fd1
  languageName: node
  linkType: hard

"hono@npm:^4.5.4":
  version: 4.10.3
  resolution: "hono@npm:4.10.3"
  checksum: 10c0/bdcc4c7066c74ba7cfa63ed6550768a0f43a420286c8f8f74b7012ea4901b8b06778fa8e98264b46f1a86920f056b7ede1f07814da4934912f9945def4977c29
  languageName: node
  linkType: hard

"http-cache-semantics@npm:^4.1.1":
  version: 4.2.0
  resolution: "http-cache-semantics@npm:4.2.0"
  checksum: 10c0/45b66a945cf13ec2d1f29432277201313babf4a01d9e52f44b31ca923434083afeca03f18417f599c9ab3d0e7b618ceb21257542338b57c54b710463b4a53e37
  languageName: node
  linkType: hard

"http-proxy-agent@npm:^7.0.0":
  version: 7.0.2
  resolution: "http-proxy-agent@npm:7.0.2"
  dependencies:
    agent-base: "npm:^7.1.0"
    debug: "npm:^4.3.4"
  checksum: 10c0/4207b06a4580fb85dd6dff521f0abf6db517489e70863dca1a0291daa7f2d3d2d6015a57bd702af068ea5cf9f1f6ff72314f5f5b4228d299c0904135d2aef921
  languageName: node
  linkType: hard

"https-proxy-agent@npm:^7.0.1":
  version: 7.0.6
  resolution: "https-proxy-agent@npm:7.0.6"
  dependencies:
    agent-base: "npm:^7.1.2"
    debug: "npm:4"
  checksum: 10c0/f729219bc735edb621fa30e6e84e60ee5d00802b8247aac0d7b79b0bd6d4b3294737a337b93b86a0bd9e68099d031858a39260c976dc14cdbba238ba1f8779ac
  languageName: node
  linkType: hard

"iconv-lite@npm:^0.6.2":
  version: 0.6.3
  resolution: "iconv-lite@npm:0.6.3"
  dependencies:
    safer-buffer: "npm:>= 2.1.2 < 3.0.0"
  checksum: 10c0/98102bc66b33fcf5ac044099d1257ba0b7ad5e3ccd3221f34dd508ab4070edff183276221684e1e0555b145fce0850c9f7d2b60a9fcac50fbb4ea0d6e845a3b1
  languageName: node
  linkType: hard

"icss-utils@npm:^5.0.0, icss-utils@npm:^5.1.0":
  version: 5.1.0
  resolution: "icss-utils@npm:5.1.0"
  peerDependencies:
    postcss: ^8.1.0
  checksum: 10c0/39c92936fabd23169c8611d2b5cc39e39d10b19b0d223352f20a7579f75b39d5f786114a6b8fc62bee8c5fed59ba9e0d38f7219a4db383e324fb3061664b043d
  languageName: node
  linkType: hard

"ignore@npm:^5.2.0":
  version: 5.3.2
  resolution: "ignore@npm:5.3.2"
  checksum: 10c0/f9f652c957983634ded1e7f02da3b559a0d4cc210fca3792cb67f1b153623c9c42efdc1c4121af171e295444459fc4a9201101fb041b1104a3c000bccb188337
  languageName: node
  linkType: hard

"ignore@npm:^7.0.0":
  version: 7.0.5
  resolution: "ignore@npm:7.0.5"
  checksum: 10c0/ae00db89fe873064a093b8999fe4cc284b13ef2a178636211842cceb650b9c3e390d3339191acb145d81ed5379d2074840cf0c33a20bdbd6f32821f79eb4ad5d
  languageName: node
  linkType: hard

"import-fresh@npm:^3.2.1":
  version: 3.3.1
  resolution: "import-fresh@npm:3.3.1"
  dependencies:
    parent-module: "npm:^1.0.0"
    resolve-from: "npm:^4.0.0"
  checksum: 10c0/bf8cc494872fef783249709385ae883b447e3eb09db0ebd15dcead7d9afe7224dad7bd7591c6b73b0b19b3c0f9640eb8ee884f01cfaf2887ab995b0b36a0cbec
  languageName: node
  linkType: hard

"imurmurhash@npm:^0.1.4":
  version: 0.1.4
  resolution: "imurmurhash@npm:0.1.4"
  checksum: 10c0/8b51313850dd33605c6c9d3fd9638b714f4c4c40250cff658209f30d40da60f78992fb2df5dabee4acf589a6a82bbc79ad5486550754bd9ec4e3fc0d4a57d6a6
  languageName: node
  linkType: hard

"inherits@npm:^2.0.3":
  version: 2.0.4
  resolution: "inherits@npm:2.0.4"
  checksum: 10c0/4e531f648b29039fb7426fb94075e6545faa1eb9fe83c29f0b6d9e7263aceb4289d2d4557db0d428188eeb449cc7c5e77b0a0b2c4e248ff2a65933a0dee49ef2
  languageName: node
  linkType: hard

"ip-address@npm:^9.0.5":
  version: 9.0.5
  resolution: "ip-address@npm:9.0.5"
  dependencies:
    jsbn: "npm:1.1.0"
    sprintf-js: "npm:^1.1.3"
  checksum: 10c0/331cd07fafcb3b24100613e4b53e1a2b4feab11e671e655d46dc09ee233da5011284d09ca40c4ecbdfe1d0004f462958675c224a804259f2f78d2465a87824bc
  languageName: node
  linkType: hard

"is-arrayish@npm:^0.3.1":
  version: 0.3.2
  resolution: "is-arrayish@npm:0.3.2"
  checksum: 10c0/f59b43dc1d129edb6f0e282595e56477f98c40278a2acdc8b0a5c57097c9eff8fe55470493df5775478cf32a4dc8eaf6d3a749f07ceee5bc263a78b2434f6a54
  languageName: node
  linkType: hard

"is-docker@npm:^3.0.0":
  version: 3.0.0
  resolution: "is-docker@npm:3.0.0"
  bin:
    is-docker: cli.js
  checksum: 10c0/d2c4f8e6d3e34df75a5defd44991b6068afad4835bb783b902fa12d13ebdb8f41b2a199dcb0b5ed2cb78bfee9e4c0bbdb69c2d9646f4106464674d3e697a5856
  languageName: node
  linkType: hard

"is-extglob@npm:^2.1.1":
  version: 2.1.1
  resolution: "is-extglob@npm:2.1.1"
  checksum: 10c0/5487da35691fbc339700bbb2730430b07777a3c21b9ebaecb3072512dfd7b4ba78ac2381a87e8d78d20ea08affb3f1971b4af629173a6bf435ff8a4c47747912
  languageName: node
  linkType: hard

"is-fullwidth-code-point@npm:^3.0.0":
  version: 3.0.0
  resolution: "is-fullwidth-code-point@npm:3.0.0"
  checksum: 10c0/bb11d825e049f38e04c06373a8d72782eee0205bda9d908cc550ccb3c59b99d750ff9537982e01733c1c94a58e35400661f57042158ff5e8f3e90cf936daf0fc
  languageName: node
  linkType: hard

"is-glob@npm:^4.0.0, is-glob@npm:^4.0.1, is-glob@npm:^4.0.3":
  version: 4.0.3
  resolution: "is-glob@npm:4.0.3"
  dependencies:
    is-extglob: "npm:^2.1.1"
  checksum: 10c0/17fb4014e22be3bbecea9b2e3a76e9e34ff645466be702f1693e8f1ee1adac84710d0be0bd9f967d6354036fd51ab7c2741d954d6e91dae6bb69714de92c197a
  languageName: node
  linkType: hard

"is-inside-container@npm:^1.0.0":
  version: 1.0.0
  resolution: "is-inside-container@npm:1.0.0"
  dependencies:
    is-docker: "npm:^3.0.0"
  bin:
    is-inside-container: cli.js
  checksum: 10c0/a8efb0e84f6197e6ff5c64c52890fa9acb49b7b74fed4da7c95383965da6f0fa592b4dbd5e38a79f87fc108196937acdbcd758fcefc9b140e479b39ce1fcd1cd
  languageName: node
  linkType: hard

"is-number@npm:^7.0.0":
  version: 7.0.0
  resolution: "is-number@npm:7.0.0"
  checksum: 10c0/b4686d0d3053146095ccd45346461bc8e53b80aeb7671cc52a4de02dbbf7dc0d1d2a986e2fe4ae206984b4d34ef37e8b795ebc4f4295c978373e6575e295d811
  languageName: node
  linkType: hard

"is-stream@npm:^2.0.0":
  version: 2.0.1
  resolution: "is-stream@npm:2.0.1"
  checksum: 10c0/7c284241313fc6efc329b8d7f08e16c0efeb6baab1b4cd0ba579eb78e5af1aa5da11e68559896a2067cd6c526bd29241dda4eb1225e627d5aa1a89a76d4635a5
  languageName: node
  linkType: hard

"is-what@npm:^4.1.8":
  version: 4.1.16
  resolution: "is-what@npm:4.1.16"
  checksum: 10c0/611f1947776826dcf85b57cfb7bd3b3ea6f4b94a9c2f551d4a53f653cf0cb9d1e6518846648256d46ee6c91d114b6d09d2ac8a07306f7430c5900f87466aae5b
  languageName: node
  linkType: hard

"is-wsl@npm:^3.1.0":
  version: 3.1.0
  resolution: "is-wsl@npm:3.1.0"
  dependencies:
    is-inside-container: "npm:^1.0.0"
  checksum: 10c0/d3317c11995690a32c362100225e22ba793678fe8732660c6de511ae71a0ff05b06980cf21f98a6bf40d7be0e9e9506f859abe00a1118287d63e53d0a3d06947
  languageName: node
  linkType: hard

"isexe@npm:^2.0.0":
  version: 2.0.0
  resolution: "isexe@npm:2.0.0"
  checksum: 10c0/228cfa503fadc2c31596ab06ed6aa82c9976eec2bfd83397e7eaf06d0ccf42cd1dfd6743bf9aeb01aebd4156d009994c5f76ea898d2832c1fe342da923ca457d
  languageName: node
  linkType: hard

"isexe@npm:^3.1.1":
  version: 3.1.1
  resolution: "isexe@npm:3.1.1"
  checksum: 10c0/9ec257654093443eb0a528a9c8cbba9c0ca7616ccb40abd6dde7202734d96bb86e4ac0d764f0f8cd965856aacbff2f4ce23e730dc19dfb41e3b0d865ca6fdcc7
  languageName: node
  linkType: hard

"jackspeak@npm:^3.1.2":
  version: 3.4.3
  resolution: "jackspeak@npm:3.4.3"
  dependencies:
    "@isaacs/cliui": "npm:^8.0.2"
    "@pkgjs/parseargs": "npm:^0.11.0"
  dependenciesMeta:
    "@pkgjs/parseargs":
      optional: true
  checksum: 10c0/6acc10d139eaefdbe04d2f679e6191b3abf073f111edf10b1de5302c97ec93fffeb2fdd8681ed17f16268aa9dd4f8c588ed9d1d3bffbbfa6e8bf897cbb3149b9
  languageName: node
  linkType: hard

"jiti@npm:^2.4.2, jiti@npm:^2.5.1":
  version: 2.5.1
  resolution: "jiti@npm:2.5.1"
  bin:
    jiti: lib/jiti-cli.mjs
  checksum: 10c0/f0a38d7d8842cb35ffe883038166aa2d52ffd21f1a4fc839ae4076ea7301c22a1f11373f8fc52e2667de7acde8f3e092835620dd6f72a0fbe9296b268b0874bb
  languageName: node
  linkType: hard

"js-tiktoken@npm:^1.0.12":
  version: 1.0.20
  resolution: "js-tiktoken@npm:1.0.20"
  dependencies:
    base64-js: "npm:^1.5.1"
  checksum: 10c0/846c9257b25efa153695bb2a4a85f35e2e747fa76f03d04dcaffee5b64ce22ccd61db83af099492558375eabf18c8070369d0f88de07a9fcf4cb494bb9759dbd
  languageName: node
  linkType: hard

"js-tokens@npm:^4.0.0":
  version: 4.0.0
  resolution: "js-tokens@npm:4.0.0"
  checksum: 10c0/e248708d377aa058eacf2037b07ded847790e6de892bbad3dac0abba2e759cb9f121b00099a65195616badcb6eca8d14d975cb3e89eb1cfda644756402c8aeed
  languageName: node
  linkType: hard

"js-yaml@npm:^4.1.0":
  version: 4.1.0
  resolution: "js-yaml@npm:4.1.0"
  dependencies:
    argparse: "npm:^2.0.1"
  bin:
    js-yaml: bin/js-yaml.js
  checksum: 10c0/184a24b4eaacfce40ad9074c64fd42ac83cf74d8c8cd137718d456ced75051229e5061b8633c3366b8aada17945a7a356b337828c19da92b51ae62126575018f
  languageName: node
  linkType: hard

"jsbn@npm:1.1.0":
  version: 1.1.0
  resolution: "jsbn@npm:1.1.0"
  checksum: 10c0/4f907fb78d7b712e11dea8c165fe0921f81a657d3443dde75359ed52eb2b5d33ce6773d97985a089f09a65edd80b11cb75c767b57ba47391fee4c969f7215c96
  languageName: node
  linkType: hard

"json-buffer@npm:3.0.1":
  version: 3.0.1
  resolution: "json-buffer@npm:3.0.1"
  checksum: 10c0/0d1c91569d9588e7eef2b49b59851f297f3ab93c7b35c7c221e288099322be6b562767d11e4821da500f3219542b9afd2e54c5dc573107c1126ed1080f8e96d7
  languageName: node
  linkType: hard

"json-schema-traverse@npm:^0.4.1":
  version: 0.4.1
  resolution: "json-schema-traverse@npm:0.4.1"
  checksum: 10c0/108fa90d4cc6f08243aedc6da16c408daf81793bf903e9fd5ab21983cda433d5d2da49e40711da016289465ec2e62e0324dcdfbc06275a607fe3233fde4942ce
  languageName: node
  linkType: hard

"json-stable-stringify-without-jsonify@npm:^1.0.1":
  version: 1.0.1
  resolution: "json-stable-stringify-without-jsonify@npm:1.0.1"
  checksum: 10c0/cb168b61fd4de83e58d09aaa6425ef71001bae30d260e2c57e7d09a5fd82223e2f22a042dedaab8db23b7d9ae46854b08bb1f91675a8be11c5cffebef5fb66a5
  languageName: node
  linkType: hard

"keyv@npm:^4.5.4":
  version: 4.5.4
  resolution: "keyv@npm:4.5.4"
  dependencies:
    json-buffer: "npm:3.0.1"
  checksum: 10c0/aa52f3c5e18e16bb6324876bb8b59dd02acf782a4b789c7b2ae21107fab95fab3890ed448d4f8dba80ce05391eeac4bfabb4f02a20221342982f806fa2cf271e
  languageName: node
  linkType: hard

"kuler@npm:^2.0.0":
  version: 2.0.0
  resolution: "kuler@npm:2.0.0"
  checksum: 10c0/0a4e99d92ca373f8f74d1dc37931909c4d0d82aebc94cf2ba265771160fc12c8df34eaaac80805efbda367e2795cb1f1dd4c3d404b6b1cf38aec94035b503d2d
  languageName: node
  linkType: hard

"langsmith@npm:^0.3.33, langsmith@npm:^0.3.46":
  version: 0.3.49
  resolution: "langsmith@npm:0.3.49"
  dependencies:
    "@types/uuid": "npm:^10.0.0"
    chalk: "npm:^4.1.2"
    console-table-printer: "npm:^2.12.1"
    p-queue: "npm:^6.6.2"
    p-retry: "npm:4"
    semver: "npm:^7.6.3"
    uuid: "npm:^10.0.0"
  peerDependencies:
    "@opentelemetry/api": "*"
    "@opentelemetry/exporter-trace-otlp-proto": "*"
    "@opentelemetry/sdk-trace-base": "*"
    openai: "*"
  peerDependenciesMeta:
    "@opentelemetry/api":
      optional: true
    "@opentelemetry/exporter-trace-otlp-proto":
      optional: true
    "@opentelemetry/sdk-trace-base":
      optional: true
    openai:
      optional: true
  checksum: 10c0/7152988844b04403c6e5b1e7b321eba12d62f273c9fcd554f04cfc5f905df85c126cb81238315210a282e53b6cafe11af53878ab4a82c1cef3ad196974419782
  languageName: node
  linkType: hard

"levn@npm:^0.4.1":
  version: 0.4.1
  resolution: "levn@npm:0.4.1"
  dependencies:
    prelude-ls: "npm:^1.2.1"
    type-check: "npm:~0.4.0"
  checksum: 10c0/effb03cad7c89dfa5bd4f6989364bfc79994c2042ec5966cb9b95990e2edee5cd8969ddf42616a0373ac49fac1403437deaf6e9050fbbaa3546093a59b9ac94e
  languageName: node
  linkType: hard

"lightningcss-darwin-arm64@npm:1.30.1":
  version: 1.30.1
  resolution: "lightningcss-darwin-arm64@npm:1.30.1"
  conditions: os=darwin & cpu=arm64
  languageName: node
  linkType: hard

"lightningcss-darwin-x64@npm:1.30.1":
  version: 1.30.1
  resolution: "lightningcss-darwin-x64@npm:1.30.1"
  conditions: os=darwin & cpu=x64
  languageName: node
  linkType: hard

"lightningcss-freebsd-x64@npm:1.30.1":
  version: 1.30.1
  resolution: "lightningcss-freebsd-x64@npm:1.30.1"
  conditions: os=freebsd & cpu=x64
  languageName: node
  linkType: hard

"lightningcss-linux-arm-gnueabihf@npm:1.30.1":
  version: 1.30.1
  resolution: "lightningcss-linux-arm-gnueabihf@npm:1.30.1"
  conditions: os=linux & cpu=arm
  languageName: node
  linkType: hard

"lightningcss-linux-arm64-gnu@npm:1.30.1":
  version: 1.30.1
  resolution: "lightningcss-linux-arm64-gnu@npm:1.30.1"
  conditions: os=linux & cpu=arm64 & libc=glibc
  languageName: node
  linkType: hard

"lightningcss-linux-arm64-musl@npm:1.30.1":
  version: 1.30.1
  resolution: "lightningcss-linux-arm64-musl@npm:1.30.1"
  conditions: os=linux & cpu=arm64 & libc=musl
  languageName: node
  linkType: hard

"lightningcss-linux-x64-gnu@npm:1.30.1":
  version: 1.30.1
  resolution: "lightningcss-linux-x64-gnu@npm:1.30.1"
  conditions: os=linux & cpu=x64 & libc=glibc
  languageName: node
  linkType: hard

"lightningcss-linux-x64-musl@npm:1.30.1":
  version: 1.30.1
  resolution: "lightningcss-linux-x64-musl@npm:1.30.1"
  conditions: os=linux & cpu=x64 & libc=musl
  languageName: node
  linkType: hard

"lightningcss-win32-arm64-msvc@npm:1.30.1":
  version: 1.30.1
  resolution: "lightningcss-win32-arm64-msvc@npm:1.30.1"
  conditions: os=win32 & cpu=arm64
  languageName: node
  linkType: hard

"lightningcss-win32-x64-msvc@npm:1.30.1":
  version: 1.30.1
  resolution: "lightningcss-win32-x64-msvc@npm:1.30.1"
  conditions: os=win32 & cpu=x64
  languageName: node
  linkType: hard

"lightningcss@npm:1.30.1":
  version: 1.30.1
  resolution: "lightningcss@npm:1.30.1"
  dependencies:
    detect-libc: "npm:^2.0.3"
    lightningcss-darwin-arm64: "npm:1.30.1"
    lightningcss-darwin-x64: "npm:1.30.1"
    lightningcss-freebsd-x64: "npm:1.30.1"
    lightningcss-linux-arm-gnueabihf: "npm:1.30.1"
    lightningcss-linux-arm64-gnu: "npm:1.30.1"
    lightningcss-linux-arm64-musl: "npm:1.30.1"
    lightningcss-linux-x64-gnu: "npm:1.30.1"
    lightningcss-linux-x64-musl: "npm:1.30.1"
    lightningcss-win32-arm64-msvc: "npm:1.30.1"
    lightningcss-win32-x64-msvc: "npm:1.30.1"
  dependenciesMeta:
    lightningcss-darwin-arm64:
      optional: true
    lightningcss-darwin-x64:
      optional: true
    lightningcss-freebsd-x64:
      optional: true
    lightningcss-linux-arm-gnueabihf:
      optional: true
    lightningcss-linux-arm64-gnu:
      optional: true
    lightningcss-linux-arm64-musl:
      optional: true
    lightningcss-linux-x64-gnu:
      optional: true
    lightningcss-linux-x64-musl:
      optional: true
    lightningcss-win32-arm64-msvc:
      optional: true
    lightningcss-win32-x64-msvc:
      optional: true
  checksum: 10c0/1e1ad908f3c68bf39d964a6735435a8dd5474fb2765076732d64a7b6aa2af1f084da65a9462443a9adfebf7dcfb02fb532fce1d78697f2a9de29c8f40f09aee3
  languageName: node
  linkType: hard

"loader-utils@npm:^3.2.0":
  version: 3.3.1
  resolution: "loader-utils@npm:3.3.1"
  checksum: 10c0/f2af4eb185ac5bf7e56e1337b666f90744e9f443861ac521b48f093fb9e8347f191c8960b4388a3365147d218913bc23421234e7788db69f385bacfefa0b4758
  languageName: node
  linkType: hard

"locate-path@npm:^6.0.0":
  version: 6.0.0
  resolution: "locate-path@npm:6.0.0"
  dependencies:
    p-locate: "npm:^5.0.0"
  checksum: 10c0/d3972ab70dfe58ce620e64265f90162d247e87159b6126b01314dd67be43d50e96a50b517bce2d9452a79409c7614054c277b5232377de50416564a77ac7aad3
  languageName: node
  linkType: hard

"lodash.camelcase@npm:^4.3.0":
  version: 4.3.0
  resolution: "lodash.camelcase@npm:4.3.0"
  checksum: 10c0/fcba15d21a458076dd309fce6b1b4bf611d84a0ec252cb92447c948c533ac250b95d2e00955801ebc367e5af5ed288b996d75d37d2035260a937008e14eaf432
  languageName: node
  linkType: hard

"lodash.merge@npm:^4.6.2":
  version: 4.6.2
  resolution: "lodash.merge@npm:4.6.2"
  checksum: 10c0/402fa16a1edd7538de5b5903a90228aa48eb5533986ba7fa26606a49db2572bf414ff73a2c9f5d5fd36b31c46a5d5c7e1527749c07cbcf965ccff5fbdf32c506
  languageName: node
  linkType: hard

"logform@npm:^2.2.0, logform@npm:^2.7.0":
  version: 2.7.0
  resolution: "logform@npm:2.7.0"
  dependencies:
    "@colors/colors": "npm:1.6.0"
    "@types/triple-beam": "npm:^1.3.2"
    fecha: "npm:^4.2.0"
    ms: "npm:^2.1.1"
    safe-stable-stringify: "npm:^2.3.1"
    triple-beam: "npm:^1.3.0"
  checksum: 10c0/4789b4b37413c731d1835734cb799240d31b865afde6b7b3e06051d6a4127bfda9e88c99cfbf296d084a315ccbed2647796e6a56b66e725bcb268c586f57558f
  languageName: node
  linkType: hard

"lru-cache@npm:^10.0.1, lru-cache@npm:^10.2.0":
  version: 10.4.3
  resolution: "lru-cache@npm:10.4.3"
  checksum: 10c0/ebd04fbca961e6c1d6c0af3799adcc966a1babe798f685bb84e6599266599cd95d94630b10262f5424539bc4640107e8a33aa28585374abf561d30d16f4b39fb
  languageName: node
  linkType: hard

"magic-string@npm:^0.30.17":
  version: 0.30.17
  resolution: "magic-string@npm:0.30.17"
  dependencies:
    "@jridgewell/sourcemap-codec": "npm:^1.5.0"
  checksum: 10c0/16826e415d04b88378f200fe022b53e638e3838b9e496edda6c0e086d7753a44a6ed187adc72d19f3623810589bf139af1a315541cd6a26ae0771a0193eaf7b8
  languageName: node
  linkType: hard

"make-fetch-happen@npm:^14.0.3":
  version: 14.0.3
  resolution: "make-fetch-happen@npm:14.0.3"
  dependencies:
    "@npmcli/agent": "npm:^3.0.0"
    cacache: "npm:^19.0.1"
    http-cache-semantics: "npm:^4.1.1"
    minipass: "npm:^7.0.2"
    minipass-fetch: "npm:^4.0.0"
    minipass-flush: "npm:^1.0.5"
    minipass-pipeline: "npm:^1.2.4"
    negotiator: "npm:^1.0.0"
    proc-log: "npm:^5.0.0"
    promise-retry: "npm:^2.0.1"
    ssri: "npm:^12.0.0"
  checksum: 10c0/c40efb5e5296e7feb8e37155bde8eb70bc57d731b1f7d90e35a092fde403d7697c56fb49334d92d330d6f1ca29a98142036d6480a12681133a0a1453164cb2f0
  languageName: node
  linkType: hard

"merge2@npm:^1.3.0":
  version: 1.4.1
  resolution: "merge2@npm:1.4.1"
  checksum: 10c0/254a8a4605b58f450308fc474c82ac9a094848081bf4c06778200207820e5193726dc563a0d2c16468810516a5c97d9d3ea0ca6585d23c58ccfff2403e8dbbeb
  languageName: node
  linkType: hard

"micromatch@npm:^4.0.8":
  version: 4.0.8
  resolution: "micromatch@npm:4.0.8"
  dependencies:
    braces: "npm:^3.0.3"
    picomatch: "npm:^2.3.1"
  checksum: 10c0/166fa6eb926b9553f32ef81f5f531d27b4ce7da60e5baf8c021d043b27a388fb95e46a8038d5045877881e673f8134122b59624d5cecbd16eb50a42e7a6b5ca8
  languageName: node
  linkType: hard

"minimatch@npm:^3.1.2":
  version: 3.1.2
  resolution: "minimatch@npm:3.1.2"
  dependencies:
    brace-expansion: "npm:^1.1.7"
  checksum: 10c0/0262810a8fc2e72cca45d6fd86bd349eee435eb95ac6aa45c9ea2180e7ee875ef44c32b55b5973ceabe95ea12682f6e3725cbb63d7a2d1da3ae1163c8b210311
  languageName: node
  linkType: hard

"minimatch@npm:^9.0.4":
  version: 9.0.5
  resolution: "minimatch@npm:9.0.5"
  dependencies:
    brace-expansion: "npm:^2.0.1"
  checksum: 10c0/de96cf5e35bdf0eab3e2c853522f98ffbe9a36c37797778d2665231ec1f20a9447a7e567cb640901f89e4daaa95ae5d70c65a9e8aa2bb0019b6facbc3c0575ed
  languageName: node
  linkType: hard

"minipass-collect@npm:^2.0.1":
  version: 2.0.1
  resolution: "minipass-collect@npm:2.0.1"
  dependencies:
    minipass: "npm:^7.0.3"
  checksum: 10c0/5167e73f62bb74cc5019594709c77e6a742051a647fe9499abf03c71dca75515b7959d67a764bdc4f8b361cf897fbf25e2d9869ee039203ed45240f48b9aa06e
  languageName: node
  linkType: hard

"minipass-fetch@npm:^4.0.0":
  version: 4.0.1
  resolution: "minipass-fetch@npm:4.0.1"
  dependencies:
    encoding: "npm:^0.1.13"
    minipass: "npm:^7.0.3"
    minipass-sized: "npm:^1.0.3"
    minizlib: "npm:^3.0.1"
  dependenciesMeta:
    encoding:
      optional: true
  checksum: 10c0/a3147b2efe8e078c9bf9d024a0059339c5a09c5b1dded6900a219c218cc8b1b78510b62dae556b507304af226b18c3f1aeb1d48660283602d5b6586c399eed5c
  languageName: node
  linkType: hard

"minipass-flush@npm:^1.0.5":
  version: 1.0.5
  resolution: "minipass-flush@npm:1.0.5"
  dependencies:
    minipass: "npm:^3.0.0"
  checksum: 10c0/2a51b63feb799d2bb34669205eee7c0eaf9dce01883261a5b77410c9408aa447e478efd191b4de6fc1101e796ff5892f8443ef20d9544385819093dbb32d36bd
  languageName: node
  linkType: hard

"minipass-pipeline@npm:^1.2.4":
  version: 1.2.4
  resolution: "minipass-pipeline@npm:1.2.4"
  dependencies:
    minipass: "npm:^3.0.0"
  checksum: 10c0/cbda57cea20b140b797505dc2cac71581a70b3247b84480c1fed5ca5ba46c25ecc25f68bfc9e6dcb1a6e9017dab5c7ada5eab73ad4f0a49d84e35093e0c643f2
  languageName: node
  linkType: hard

"minipass-sized@npm:^1.0.3":
  version: 1.0.3
  resolution: "minipass-sized@npm:1.0.3"
  dependencies:
    minipass: "npm:^3.0.0"
  checksum: 10c0/298f124753efdc745cfe0f2bdfdd81ba25b9f4e753ca4a2066eb17c821f25d48acea607dfc997633ee5bf7b6dfffb4eee4f2051eb168663f0b99fad2fa4829cb
  languageName: node
  linkType: hard

"minipass@npm:^3.0.0":
  version: 3.3.6
  resolution: "minipass@npm:3.3.6"
  dependencies:
    yallist: "npm:^4.0.0"
  checksum: 10c0/a114746943afa1dbbca8249e706d1d38b85ed1298b530f5808ce51f8e9e941962e2a5ad2e00eae7dd21d8a4aae6586a66d4216d1a259385e9d0358f0c1eba16c
  languageName: node
  linkType: hard

"minipass@npm:^5.0.0 || ^6.0.2 || ^7.0.0, minipass@npm:^7.0.2, minipass@npm:^7.0.3, minipass@npm:^7.0.4, minipass@npm:^7.1.2":
  version: 7.1.2
  resolution: "minipass@npm:7.1.2"
  checksum: 10c0/b0fd20bb9fb56e5fa9a8bfac539e8915ae07430a619e4b86ff71f5fc757ef3924b23b2c4230393af1eda647ed3d75739e4e0acb250a6b1eb277cf7f8fe449557
  languageName: node
  linkType: hard

"minizlib@npm:^3.0.1":
  version: 3.0.2
  resolution: "minizlib@npm:3.0.2"
  dependencies:
    minipass: "npm:^7.1.2"
  checksum: 10c0/9f3bd35e41d40d02469cb30470c55ccc21cae0db40e08d1d0b1dff01cc8cc89a6f78e9c5d2b7c844e485ec0a8abc2238111213fdc5b2038e6d1012eacf316f78
  languageName: node
  linkType: hard

"mkdirp@npm:^3.0.1":
  version: 3.0.1
  resolution: "mkdirp@npm:3.0.1"
  bin:
    mkdirp: dist/cjs/src/bin.js
  checksum: 10c0/9f2b975e9246351f5e3a40dcfac99fcd0baa31fbfab615fe059fb11e51f10e4803c63de1f384c54d656e4db31d000e4767e9ef076a22e12a641357602e31d57d
  languageName: node
  linkType: hard

"ms@npm:^2.1.1, ms@npm:^2.1.3":
  version: 2.1.3
  resolution: "ms@npm:2.1.3"
  checksum: 10c0/d924b57e7312b3b63ad21fc5b3dc0af5e78d61a1fc7cfb5457edaf26326bf62be5307cc87ffb6862ef1c2b33b0233cdb5d4f01c4c958cc0d660948b65a287a48
  languageName: node
  linkType: hard

"mustache@npm:^4.2.0":
  version: 4.2.0
  resolution: "mustache@npm:4.2.0"
  bin:
    mustache: bin/mustache
  checksum: 10c0/1f8197e8a19e63645a786581d58c41df7853da26702dbc005193e2437c98ca49b255345c173d50c08fe4b4dbb363e53cb655ecc570791f8deb09887248dd34a2
  languageName: node
  linkType: hard

"nanoid@npm:^3.3.11":
  version: 3.3.11
  resolution: "nanoid@npm:3.3.11"
  bin:
    nanoid: bin/nanoid.cjs
  checksum: 10c0/40e7f70b3d15f725ca072dfc4f74e81fcf1fbb02e491cf58ac0c79093adc9b0a73b152bcde57df4b79cd097e13023d7504acb38404a4da7bc1cd8e887b82fe0b
  languageName: node
  linkType: hard

"natural-compare@npm:^1.4.0":
  version: 1.4.0
  resolution: "natural-compare@npm:1.4.0"
  checksum: 10c0/f5f9a7974bfb28a91afafa254b197f0f22c684d4a1731763dda960d2c8e375b36c7d690e0d9dc8fba774c537af14a7e979129bca23d88d052fbeb9466955e447
  languageName: node
  linkType: hard

"negotiator@npm:^1.0.0":
  version: 1.0.0
  resolution: "negotiator@npm:1.0.0"
  checksum: 10c0/4c559dd52669ea48e1914f9d634227c561221dd54734070791f999c52ed0ff36e437b2e07d5c1f6e32909fc625fe46491c16e4a8f0572567d4dd15c3a4fda04b
  languageName: node
  linkType: hard

"node-gyp@npm:latest":
  version: 11.2.0
  resolution: "node-gyp@npm:11.2.0"
  dependencies:
    env-paths: "npm:^2.2.0"
    exponential-backoff: "npm:^3.1.1"
    graceful-fs: "npm:^4.2.6"
    make-fetch-happen: "npm:^14.0.3"
    nopt: "npm:^8.0.0"
    proc-log: "npm:^5.0.0"
    semver: "npm:^7.3.5"
    tar: "npm:^7.4.3"
    tinyglobby: "npm:^0.2.12"
    which: "npm:^5.0.0"
  bin:
    node-gyp: bin/node-gyp.js
  checksum: 10c0/bd8d8c76b06be761239b0c8680f655f6a6e90b48e44d43415b11c16f7e8c15be346fba0cbf71588c7cdfb52c419d928a7d3db353afc1d952d19756237d8f10b9
  languageName: node
  linkType: hard

"node-releases@npm:^2.0.19":
  version: 2.0.19
  resolution: "node-releases@npm:2.0.19"
  checksum: 10c0/52a0dbd25ccf545892670d1551690fe0facb6a471e15f2cfa1b20142a5b255b3aa254af5f59d6ecb69c2bec7390bc643c43aa63b13bf5e64b6075952e716b1aa
  languageName: node
  linkType: hard

"nopt@npm:^8.0.0":
  version: 8.1.0
  resolution: "nopt@npm:8.1.0"
  dependencies:
    abbrev: "npm:^3.0.0"
  bin:
    nopt: bin/nopt.js
  checksum: 10c0/62e9ea70c7a3eb91d162d2c706b6606c041e4e7b547cbbb48f8b3695af457dd6479904d7ace600856bf923dd8d1ed0696f06195c8c20f02ac87c1da0e1d315ef
  languageName: node
  linkType: hard

"normalize-range@npm:^0.1.2":
  version: 0.1.2
  resolution: "normalize-range@npm:0.1.2"
  checksum: 10c0/bf39b73a63e0a42ad1a48c2bd1bda5a07ede64a7e2567307a407674e595bcff0fa0d57e8e5f1e7fa5e91000797c7615e13613227aaaa4d6d6e87f5bd5cc95de6
  languageName: node
  linkType: hard

"one-time@npm:^1.0.0":
  version: 1.0.0
  resolution: "one-time@npm:1.0.0"
  dependencies:
    fn.name: "npm:1.x.x"
  checksum: 10c0/6e4887b331edbb954f4e915831cbec0a7b9956c36f4feb5f6de98c448ac02ff881fd8d9b55a6b1b55030af184c6b648f340a76eb211812f4ad8c9b4b8692fdaa
  languageName: node
  linkType: hard

"open@npm:^10.1.0":
  version: 10.2.0
  resolution: "open@npm:10.2.0"
  dependencies:
    default-browser: "npm:^5.2.1"
    define-lazy-prop: "npm:^3.0.0"
    is-inside-container: "npm:^1.0.0"
    wsl-utils: "npm:^0.1.0"
  checksum: 10c0/5a36d0c1fd2f74ce553beb427ca8b8494b623fc22c6132d0c1688f246a375e24584ea0b44c67133d9ab774fa69be8e12fbe1ff12504b1142bd960fb09671948f
  languageName: node
  linkType: hard

"openai@npm:^5.3.0":
  version: 5.10.2
  resolution: "openai@npm:5.10.2"
  peerDependencies:
    ws: ^8.18.0
    zod: ^3.23.8
  peerDependenciesMeta:
    ws:
      optional: true
    zod:
      optional: true
  bin:
    openai: bin/cli
  checksum: 10c0/c08896a5d20722f3bd1a72522768ffc7c053a4df94f7b7f064c9946110070187264e6e3bf01c3abeb861e5602c65329d6aec485989338bae8ce8d3d81160fca9
  languageName: node
  linkType: hard

"optionator@npm:^0.9.3":
  version: 0.9.4
  resolution: "optionator@npm:0.9.4"
  dependencies:
    deep-is: "npm:^0.1.3"
    fast-levenshtein: "npm:^2.0.6"
    levn: "npm:^0.4.1"
    prelude-ls: "npm:^1.2.1"
    type-check: "npm:^0.4.0"
    word-wrap: "npm:^1.2.5"
  checksum: 10c0/4afb687a059ee65b61df74dfe87d8d6815cd6883cb8b3d5883a910df72d0f5d029821f37025e4bccf4048873dbdb09acc6d303d27b8f76b1a80dd5a7d5334675
  languageName: node
  linkType: hard

"p-finally@npm:^1.0.0":
  version: 1.0.0
  resolution: "p-finally@npm:1.0.0"
  checksum: 10c0/6b8552339a71fe7bd424d01d8451eea92d379a711fc62f6b2fe64cad8a472c7259a236c9a22b4733abca0b5666ad503cb497792a0478c5af31ded793d00937e7
  languageName: node
  linkType: hard

"p-limit@npm:^3.0.2":
  version: 3.1.0
  resolution: "p-limit@npm:3.1.0"
  dependencies:
    yocto-queue: "npm:^0.1.0"
  checksum: 10c0/9db675949dbdc9c3763c89e748d0ef8bdad0afbb24d49ceaf4c46c02c77d30db4e0652ed36d0a0a7a95154335fab810d95c86153105bb73b3a90448e2bb14e1a
  languageName: node
  linkType: hard

"p-locate@npm:^5.0.0":
  version: 5.0.0
  resolution: "p-locate@npm:5.0.0"
  dependencies:
    p-limit: "npm:^3.0.2"
  checksum: 10c0/2290d627ab7903b8b70d11d384fee714b797f6040d9278932754a6860845c4d3190603a0772a663c8cb5a7b21d1b16acb3a6487ebcafa9773094edc3dfe6009a
  languageName: node
  linkType: hard

"p-map@npm:^7.0.2":
  version: 7.0.3
  resolution: "p-map@npm:7.0.3"
  checksum: 10c0/46091610da2b38ce47bcd1d8b4835a6fa4e832848a6682cf1652bc93915770f4617afc844c10a77d1b3e56d2472bb2d5622353fa3ead01a7f42b04fc8e744a5c
  languageName: node
  linkType: hard

"p-queue@npm:^6.6.2":
  version: 6.6.2
  resolution: "p-queue@npm:6.6.2"
  dependencies:
    eventemitter3: "npm:^4.0.4"
    p-timeout: "npm:^3.2.0"
  checksum: 10c0/5739ecf5806bbeadf8e463793d5e3004d08bb3f6177bd1a44a005da8fd81bb90f80e4633e1fb6f1dfd35ee663a5c0229abe26aebb36f547ad5a858347c7b0d3e
  languageName: node
  linkType: hard

"p-retry@npm:4":
  version: 4.6.2
  resolution: "p-retry@npm:4.6.2"
  dependencies:
    "@types/retry": "npm:0.12.0"
    retry: "npm:^0.13.1"
  checksum: 10c0/d58512f120f1590cfedb4c2e0c42cb3fa66f3cea8a4646632fcb834c56055bb7a6f138aa57b20cc236fb207c9d694e362e0b5c2b14d9b062f67e8925580c73b0
  languageName: node
  linkType: hard

"p-timeout@npm:^3.2.0":
  version: 3.2.0
  resolution: "p-timeout@npm:3.2.0"
  dependencies:
    p-finally: "npm:^1.0.0"
  checksum: 10c0/524b393711a6ba8e1d48137c5924749f29c93d70b671e6db761afa784726572ca06149c715632da8f70c090073afb2af1c05730303f915604fd38ee207b70a61
  languageName: node
  linkType: hard

"package-json-from-dist@npm:^1.0.0":
  version: 1.0.1
  resolution: "package-json-from-dist@npm:1.0.1"
  checksum: 10c0/62ba2785eb655fec084a257af34dbe24292ab74516d6aecef97ef72d4897310bc6898f6c85b5cd22770eaa1ce60d55a0230e150fb6a966e3ecd6c511e23d164b
  languageName: node
  linkType: hard

"parent-module@npm:^1.0.0":
  version: 1.0.1
  resolution: "parent-module@npm:1.0.1"
  dependencies:
    callsites: "npm:^3.0.0"
  checksum: 10c0/c63d6e80000d4babd11978e0d3fee386ca7752a02b035fd2435960ffaa7219dc42146f07069fb65e6e8bf1caef89daf9af7535a39bddf354d78bf50d8294f556
  languageName: node
  linkType: hard

"path-exists@npm:^4.0.0":
  version: 4.0.0
  resolution: "path-exists@npm:4.0.0"
  checksum: 10c0/8c0bd3f5238188197dc78dced15207a4716c51cc4e3624c44fc97acf69558f5ebb9a2afff486fe1b4ee148e0c133e96c5e11a9aa5c48a3006e3467da070e5e1b
  languageName: node
  linkType: hard

"path-key@npm:^3.1.0":
  version: 3.1.1
  resolution: "path-key@npm:3.1.1"
  checksum: 10c0/748c43efd5a569c039d7a00a03b58eecd1d75f3999f5a28303d75f521288df4823bc057d8784eb72358b2895a05f29a070bc9f1f17d28226cc4e62494cc58c4c
  languageName: node
  linkType: hard

"path-scurry@npm:^1.11.1":
  version: 1.11.1
  resolution: "path-scurry@npm:1.11.1"
  dependencies:
    lru-cache: "npm:^10.2.0"
    minipass: "npm:^5.0.0 || ^6.0.2 || ^7.0.0"
  checksum: 10c0/32a13711a2a505616ae1cc1b5076801e453e7aae6ac40ab55b388bb91b9d0547a52f5aaceff710ea400205f18691120d4431e520afbe4266b836fadede15872d
  languageName: node
  linkType: hard

"picocolors@npm:^1.1.1":
  version: 1.1.1
  resolution: "picocolors@npm:1.1.1"
  checksum: 10c0/e2e3e8170ab9d7c7421969adaa7e1b31434f789afb9b3f115f6b96d91945041ac3ceb02e9ec6fe6510ff036bcc0bf91e69a1772edc0b707e12b19c0f2d6bcf58
  languageName: node
  linkType: hard

"picomatch@npm:^2.3.1":
  version: 2.3.1
  resolution: "picomatch@npm:2.3.1"
  checksum: 10c0/26c02b8d06f03206fc2ab8d16f19960f2ff9e81a658f831ecb656d8f17d9edc799e8364b1f4a7873e89d9702dff96204be0fa26fe4181f6843f040f819dac4be
  languageName: node
  linkType: hard

"picomatch@npm:^4.0.2":
  version: 4.0.3
  resolution: "picomatch@npm:4.0.3"
  checksum: 10c0/9582c951e95eebee5434f59e426cddd228a7b97a0161a375aed4be244bd3fe8e3a31b846808ea14ef2c8a2527a6eeab7b3946a67d5979e81694654f939473ae2
  languageName: node
  linkType: hard

"postcss-modules-extract-imports@npm:^3.1.0":
  version: 3.1.0
  resolution: "postcss-modules-extract-imports@npm:3.1.0"
  peerDependencies:
    postcss: ^8.1.0
  checksum: 10c0/402084bcab376083c4b1b5111b48ec92974ef86066f366f0b2d5b2ac2b647d561066705ade4db89875a13cb175b33dd6af40d16d32b2ea5eaf8bac63bd2bf219
  languageName: node
  linkType: hard

"postcss-modules-local-by-default@npm:^4.0.5":
  version: 4.2.0
  resolution: "postcss-modules-local-by-default@npm:4.2.0"
  dependencies:
    icss-utils: "npm:^5.0.0"
    postcss-selector-parser: "npm:^7.0.0"
    postcss-value-parser: "npm:^4.1.0"
  peerDependencies:
    postcss: ^8.1.0
  checksum: 10c0/b0b83feb2a4b61f5383979d37f23116c99bc146eba1741ca3cf1acca0e4d0dbf293ac1810a6ab4eccbe1ee76440dd0a9eb2db5b3bba4f99fc1b3ded16baa6358
  languageName: node
  linkType: hard

"postcss-modules-scope@npm:^3.2.0":
  version: 3.2.1
  resolution: "postcss-modules-scope@npm:3.2.1"
  dependencies:
    postcss-selector-parser: "npm:^7.0.0"
  peerDependencies:
    postcss: ^8.1.0
  checksum: 10c0/bd2d81f79e3da0ef6365b8e2c78cc91469d05b58046b4601592cdeef6c4050ed8fe1478ae000a1608042fc7e692cb51fecbd2d9bce3f4eace4d32e883ffca10b
  languageName: node
  linkType: hard

"postcss-modules-values@npm:^4.0.0":
  version: 4.0.0
  resolution: "postcss-modules-values@npm:4.0.0"
  dependencies:
    icss-utils: "npm:^5.0.0"
  peerDependencies:
    postcss: ^8.1.0
  checksum: 10c0/dd18d7631b5619fb9921b198c86847a2a075f32e0c162e0428d2647685e318c487a2566cc8cc669fc2077ef38115cde7a068e321f46fb38be3ad49646b639dbc
  languageName: node
  linkType: hard

"postcss-modules@npm:^6.0.1":
  version: 6.0.1
  resolution: "postcss-modules@npm:6.0.1"
  dependencies:
    generic-names: "npm:^4.0.0"
    icss-utils: "npm:^5.1.0"
    lodash.camelcase: "npm:^4.3.0"
    postcss-modules-extract-imports: "npm:^3.1.0"
    postcss-modules-local-by-default: "npm:^4.0.5"
    postcss-modules-scope: "npm:^3.2.0"
    postcss-modules-values: "npm:^4.0.0"
    string-hash: "npm:^1.1.3"
  peerDependencies:
    postcss: ^8.0.0
  checksum: 10c0/b82230693cb257b69db486df8835626d96632481ec6a8777b51ae7a530a56fa0ed399cbc8c2c777525f31fefab5a2d12ea7331a748fdfddde9f16cf3fff3bc58
  languageName: node
  linkType: hard

"postcss-selector-parser@npm:^7.0.0":
  version: 7.1.0
  resolution: "postcss-selector-parser@npm:7.1.0"
  dependencies:
    cssesc: "npm:^3.0.0"
    util-deprecate: "npm:^1.0.2"
  checksum: 10c0/0fef257cfd1c0fe93c18a3f8a6e739b4438b527054fd77e9a62730a89b2d0ded1b59314a7e4aaa55bc256204f40830fecd2eb50f20f8cb7ab3a10b52aa06c8aa
  languageName: node
  linkType: hard

"postcss-value-parser@npm:^4.1.0, postcss-value-parser@npm:^4.2.0":
  version: 4.2.0
  resolution: "postcss-value-parser@npm:4.2.0"
  checksum: 10c0/f4142a4f56565f77c1831168e04e3effd9ffcc5aebaf0f538eee4b2d465adfd4b85a44257bb48418202a63806a7da7fe9f56c330aebb3cac898e46b4cbf49161
  languageName: node
  linkType: hard

"postcss@npm:^8.4.41, postcss@npm:^8.5.1":
  version: 8.5.6
  resolution: "postcss@npm:8.5.6"
  dependencies:
    nanoid: "npm:^3.3.11"
    picocolors: "npm:^1.1.1"
    source-map-js: "npm:^1.2.1"
  checksum: 10c0/5127cc7c91ed7a133a1b7318012d8bfa112da9ef092dddf369ae699a1f10ebbd89b1b9f25f3228795b84585c72aabd5ced5fc11f2ba467eedf7b081a66fad024
  languageName: node
  linkType: hard

"prelude-ls@npm:^1.2.1":
  version: 1.2.1
  resolution: "prelude-ls@npm:1.2.1"
  checksum: 10c0/b00d617431e7886c520a6f498a2e14c75ec58f6d93ba48c3b639cf241b54232d90daa05d83a9e9b9fef6baa63cb7e1e4602c2372fea5bc169668401eb127d0cd
  languageName: node
  linkType: hard

"proc-log@npm:^5.0.0":
  version: 5.0.0
  resolution: "proc-log@npm:5.0.0"
  checksum: 10c0/bbe5edb944b0ad63387a1d5b1911ae93e05ce8d0f60de1035b218cdcceedfe39dbd2c697853355b70f1a090f8f58fe90da487c85216bf9671f9499d1a897e9e3
  languageName: node
  linkType: hard

"promise-retry@npm:^2.0.1":
  version: 2.0.1
  resolution: "promise-retry@npm:2.0.1"
  dependencies:
    err-code: "npm:^2.0.2"
    retry: "npm:^0.12.0"
  checksum: 10c0/9c7045a1a2928094b5b9b15336dcd2a7b1c052f674550df63cc3f36cd44028e5080448175b6f6ca32b642de81150f5e7b1a98b728f15cb069f2dd60ac2616b96
  languageName: node
  linkType: hard

"punycode@npm:^2.1.0":
  version: 2.3.1
  resolution: "punycode@npm:2.3.1"
  checksum: 10c0/14f76a8206bc3464f794fb2e3d3cc665ae416c01893ad7a02b23766eb07159144ee612ad67af5e84fa4479ccfe67678c4feb126b0485651b302babf66f04f9e9
  languageName: node
  linkType: hard

"queue-microtask@npm:^1.2.2":
  version: 1.2.3
  resolution: "queue-microtask@npm:1.2.3"
  checksum: 10c0/900a93d3cdae3acd7d16f642c29a642aea32c2026446151f0778c62ac089d4b8e6c986811076e1ae180a694cedf077d453a11b58ff0a865629a4f82ab558e102
  languageName: node
  linkType: hard

"readable-stream@npm:^3.4.0, readable-stream@npm:^3.6.2":
  version: 3.6.2
  resolution: "readable-stream@npm:3.6.2"
  dependencies:
    inherits: "npm:^2.0.3"
    string_decoder: "npm:^1.1.1"
    util-deprecate: "npm:^1.0.1"
  checksum: 10c0/e37be5c79c376fdd088a45fa31ea2e423e5d48854be7a22a58869b4e84d25047b193f6acb54f1012331e1bcd667ffb569c01b99d36b0bd59658fb33f513511b7
  languageName: node
  linkType: hard

"resolve-from@npm:^4.0.0":
  version: 4.0.0
  resolution: "resolve-from@npm:4.0.0"
  checksum: 10c0/8408eec31a3112ef96e3746c37be7d64020cda07c03a920f5024e77290a218ea758b26ca9529fd7b1ad283947f34b2291c1c0f6aa0ed34acfdda9c6014c8d190
  languageName: node
  linkType: hard

"resolve-pkg-maps@npm:^1.0.0":
  version: 1.0.0
  resolution: "resolve-pkg-maps@npm:1.0.0"
  checksum: 10c0/fb8f7bbe2ca281a73b7ef423a1cbc786fb244bd7a95cbe5c3fba25b27d327150beca8ba02f622baea65919a57e061eb5005204daa5f93ed590d9b77463a567ab
  languageName: node
  linkType: hard

"retry@npm:^0.12.0":
  version: 0.12.0
  resolution: "retry@npm:0.12.0"
  checksum: 10c0/59933e8501727ba13ad73ef4a04d5280b3717fd650408460c987392efe9d7be2040778ed8ebe933c5cbd63da3dcc37919c141ef8af0a54a6e4fca5a2af177bfe
  languageName: node
  linkType: hard

"retry@npm:^0.13.1":
  version: 0.13.1
  resolution: "retry@npm:0.13.1"
  checksum: 10c0/9ae822ee19db2163497e074ea919780b1efa00431d197c7afdb950e42bf109196774b92a49fc9821f0b8b328a98eea6017410bfc5e8a0fc19c85c6d11adb3772
  languageName: node
  linkType: hard

"reusify@npm:^1.0.4":
  version: 1.1.0
  resolution: "reusify@npm:1.1.0"
  checksum: 10c0/4eff0d4a5f9383566c7d7ec437b671cc51b25963bd61bf127c3f3d3f68e44a026d99b8d2f1ad344afff8d278a8fe70a8ea092650a716d22287e8bef7126bb2fa
  languageName: node
  linkType: hard

"run-applescript@npm:^7.0.0":
  version: 7.0.0
  resolution: "run-applescript@npm:7.0.0"
  checksum: 10c0/bd821bbf154b8e6c8ecffeaf0c33cebbb78eb2987476c3f6b420d67ab4c5301faa905dec99ded76ebb3a7042b4e440189ae6d85bbbd3fc6e8d493347ecda8bfe
  languageName: node
  linkType: hard

"run-parallel@npm:^1.1.9":
  version: 1.2.0
  resolution: "run-parallel@npm:1.2.0"
  dependencies:
    queue-microtask: "npm:^1.2.2"
  checksum: 10c0/200b5ab25b5b8b7113f9901bfe3afc347e19bb7475b267d55ad0eb86a62a46d77510cb0f232507c9e5d497ebda569a08a9867d0d14f57a82ad5564d991588b39
  languageName: node
  linkType: hard

"safe-buffer@npm:~5.2.0":
  version: 5.2.1
  resolution: "safe-buffer@npm:5.2.1"
  checksum: 10c0/6501914237c0a86e9675d4e51d89ca3c21ffd6a31642efeba25ad65720bce6921c9e7e974e5be91a786b25aa058b5303285d3c15dbabf983a919f5f630d349f3
  languageName: node
  linkType: hard

"safe-stable-stringify@npm:^2.3.1":
  version: 2.5.0
  resolution: "safe-stable-stringify@npm:2.5.0"
  checksum: 10c0/baea14971858cadd65df23894a40588ed791769db21bafb7fd7608397dbdce9c5aac60748abae9995e0fc37e15f2061980501e012cd48859740796bea2987f49
  languageName: node
  linkType: hard

"safer-buffer@npm:>= 2.1.2 < 3.0.0":
  version: 2.1.2
  resolution: "safer-buffer@npm:2.1.2"
  checksum: 10c0/7e3c8b2e88a1841c9671094bbaeebd94448111dd90a81a1f606f3f67708a6ec57763b3b47f06da09fc6054193e0e6709e77325415dc8422b04497a8070fa02d4
  languageName: node
  linkType: hard

"semver@npm:^7.3.5, semver@npm:^7.6.0, semver@npm:^7.6.3, semver@npm:^7.7.1":
  version: 7.7.2
  resolution: "semver@npm:7.7.2"
  bin:
    semver: bin/semver.js
  checksum: 10c0/aca305edfbf2383c22571cb7714f48cadc7ac95371b4b52362fb8eeffdfbc0de0669368b82b2b15978f8848f01d7114da65697e56cd8c37b0dab8c58e543f9ea
  languageName: node
  linkType: hard

"shebang-command@npm:^2.0.0":
  version: 2.0.0
  resolution: "shebang-command@npm:2.0.0"
  dependencies:
    shebang-regex: "npm:^3.0.0"
  checksum: 10c0/a41692e7d89a553ef21d324a5cceb5f686d1f3c040759c50aab69688634688c5c327f26f3ecf7001ebfd78c01f3c7c0a11a7c8bfd0a8bc9f6240d4f40b224e4e
  languageName: node
  linkType: hard

"shebang-regex@npm:^3.0.0":
  version: 3.0.0
  resolution: "shebang-regex@npm:3.0.0"
  checksum: 10c0/1dbed0726dd0e1152a92696c76c7f06084eb32a90f0528d11acd764043aacf76994b2fb30aa1291a21bd019d6699164d048286309a278855ee7bec06cf6fb690
  languageName: node
  linkType: hard

"signal-exit@npm:^4.0.1":
  version: 4.1.0
  resolution: "signal-exit@npm:4.1.0"
  checksum: 10c0/41602dce540e46d599edba9d9860193398d135f7ff72cab629db5171516cfae628d21e7bfccde1bbfdf11c48726bc2a6d1a8fb8701125852fbfda7cf19c6aa83
  languageName: node
  linkType: hard

"simple-swizzle@npm:^0.2.2":
  version: 0.2.2
  resolution: "simple-swizzle@npm:0.2.2"
  dependencies:
    is-arrayish: "npm:^0.3.1"
  checksum: 10c0/df5e4662a8c750bdba69af4e8263c5d96fe4cd0f9fe4bdfa3cbdeb45d2e869dff640beaaeb1ef0e99db4d8d2ec92f85508c269f50c972174851bc1ae5bd64308
  languageName: node
  linkType: hard

"simple-wcswidth@npm:^1.0.1":
  version: 1.1.2
  resolution: "simple-wcswidth@npm:1.1.2"
  checksum: 10c0/0db23ffef39d81a018a2354d64db1d08a44123c54263e48173992c61d808aaa8b58e5651d424e8c275589671f35e9094ac6fa2bbf2c98771b1bae9e007e611dd
  languageName: node
  linkType: hard

"smart-buffer@npm:^4.2.0":
  version: 4.2.0
  resolution: "smart-buffer@npm:4.2.0"
  checksum: 10c0/a16775323e1404dd43fabafe7460be13a471e021637bc7889468eb45ce6a6b207261f454e4e530a19500cc962c4cc5348583520843b363f4193cee5c00e1e539
  languageName: node
  linkType: hard

"socks-proxy-agent@npm:^8.0.3":
  version: 8.0.5
  resolution: "socks-proxy-agent@npm:8.0.5"
  dependencies:
    agent-base: "npm:^7.1.2"
    debug: "npm:^4.3.4"
    socks: "npm:^2.8.3"
  checksum: 10c0/5d2c6cecba6821389aabf18728325730504bf9bb1d9e342e7987a5d13badd7a98838cc9a55b8ed3cb866ad37cc23e1086f09c4d72d93105ce9dfe76330e9d2a6
  languageName: node
  linkType: hard

"socks@npm:^2.8.3":
  version: 2.8.6
  resolution: "socks@npm:2.8.6"
  dependencies:
    ip-address: "npm:^9.0.5"
    smart-buffer: "npm:^4.2.0"
  checksum: 10c0/15b95db4caa359c80bfa880ff3e58f3191b9ffa4313570e501a60ee7575f51e4be664a296f4ee5c2c40544da179db6140be53433ce41ec745f9d51f342557514
  languageName: node
  linkType: hard

"source-map-js@npm:^1.2.1":
  version: 1.2.1
  resolution: "source-map-js@npm:1.2.1"
  checksum: 10c0/7bda1fc4c197e3c6ff17de1b8b2c20e60af81b63a52cb32ec5a5d67a20a7d42651e2cb34ebe93833c5a2a084377e17455854fee3e21e7925c64a51b6a52b0faf
  languageName: node
  linkType: hard

"sprintf-js@npm:^1.1.3":
  version: 1.1.3
  resolution: "sprintf-js@npm:1.1.3"
  checksum: 10c0/09270dc4f30d479e666aee820eacd9e464215cdff53848b443964202bf4051490538e5dd1b42e1a65cf7296916ca17640aebf63dae9812749c7542ee5f288dec
  languageName: node
  linkType: hard

"ssri@npm:^12.0.0":
  version: 12.0.0
  resolution: "ssri@npm:12.0.0"
  dependencies:
    minipass: "npm:^7.0.3"
  checksum: 10c0/caddd5f544b2006e88fa6b0124d8d7b28208b83c72d7672d5ade44d794525d23b540f3396108c4eb9280dcb7c01f0bef50682f5b4b2c34291f7c5e211fd1417d
  languageName: node
  linkType: hard

"stack-trace@npm:0.0.x":
  version: 0.0.10
  resolution: "stack-trace@npm:0.0.10"
  checksum: 10c0/9ff3dabfad4049b635a85456f927a075c9d0c210e3ea336412d18220b2a86cbb9b13ec46d6c37b70a302a4ea4d49e30e5d4944dd60ae784073f1cde778ac8f4b
  languageName: node
  linkType: hard

"stacktrace-parser@npm:^0.1.10":
  version: 0.1.11
  resolution: "stacktrace-parser@npm:0.1.11"
  dependencies:
    type-fest: "npm:^0.7.1"
  checksum: 10c0/4633d9afe8cd2f6c7fb2cebdee3cc8de7fd5f6f9736645fd08c0f66872a303061ce9cc0ccf46f4216dc94a7941b56e331012398dc0024dc25e46b5eb5d4ff018
  languageName: node
  linkType: hard

"string-hash@npm:^1.1.3":
  version: 1.1.3
  resolution: "string-hash@npm:1.1.3"
  checksum: 10c0/179725d7706b49fbbc0a4901703a2d8abec244140879afd5a17908497e586a6b07d738f6775450aefd9f8dd729e4a0abd073fbc6fa3bd020b7a1d2369614af88
  languageName: node
  linkType: hard

"string-width-cjs@npm:string-width@^4.2.0, string-width@npm:^4.1.0":
  version: 4.2.3
  resolution: "string-width@npm:4.2.3"
  dependencies:
    emoji-regex: "npm:^8.0.0"
    is-fullwidth-code-point: "npm:^3.0.0"
    strip-ansi: "npm:^6.0.1"
  checksum: 10c0/1e525e92e5eae0afd7454086eed9c818ee84374bb80328fc41217ae72ff5f065ef1c9d7f72da41de40c75fa8bb3dee63d92373fd492c84260a552c636392a47b
  languageName: node
  linkType: hard

"string-width@npm:^5.0.1, string-width@npm:^5.1.2":
  version: 5.1.2
  resolution: "string-width@npm:5.1.2"
  dependencies:
    eastasianwidth: "npm:^0.2.0"
    emoji-regex: "npm:^9.2.2"
    strip-ansi: "npm:^7.0.1"
  checksum: 10c0/ab9c4264443d35b8b923cbdd513a089a60de339216d3b0ed3be3ba57d6880e1a192b70ae17225f764d7adbf5994e9bb8df253a944736c15a0240eff553c678ca
  languageName: node
  linkType: hard

"string_decoder@npm:^1.1.1":
  version: 1.3.0
  resolution: "string_decoder@npm:1.3.0"
  dependencies:
    safe-buffer: "npm:~5.2.0"
  checksum: 10c0/810614ddb030e271cd591935dcd5956b2410dd079d64ff92a1844d6b7588bf992b3e1b69b0f4d34a3e06e0bd73046ac646b5264c1987b20d0601f81ef35d731d
  languageName: node
  linkType: hard

"strip-ansi-cjs@npm:strip-ansi@^6.0.1, strip-ansi@npm:^6.0.0, strip-ansi@npm:^6.0.1":
  version: 6.0.1
  resolution: "strip-ansi@npm:6.0.1"
  dependencies:
    ansi-regex: "npm:^5.0.1"
  checksum: 10c0/1ae5f212a126fe5b167707f716942490e3933085a5ff6c008ab97ab2f272c8025d3aa218b7bd6ab25729ca20cc81cddb252102f8751e13482a5199e873680952
  languageName: node
  linkType: hard

"strip-ansi@npm:^7.0.1":
  version: 7.1.0
  resolution: "strip-ansi@npm:7.1.0"
  dependencies:
    ansi-regex: "npm:^6.0.1"
  checksum: 10c0/a198c3762e8832505328cbf9e8c8381de14a4fa50a4f9b2160138158ea88c0f5549fb50cb13c651c3088f47e63a108b34622ec18c0499b6c8c3a5ddf6b305ac4
  languageName: node
  linkType: hard

"strip-json-comments@npm:^3.1.1":
  version: 3.1.1
  resolution: "strip-json-comments@npm:3.1.1"
  checksum: 10c0/9681a6257b925a7fa0f285851c0e613cc934a50661fa7bb41ca9cbbff89686bb4a0ee366e6ecedc4daafd01e83eee0720111ab294366fe7c185e935475ebcecd
  languageName: node
  linkType: hard

"strnum@npm:^1.1.1":
  version: 1.1.2
  resolution: "strnum@npm:1.1.2"
  checksum: 10c0/a0fce2498fa3c64ce64a40dada41beb91cabe3caefa910e467dc0518ef2ebd7e4d10f8c2202a6104f1410254cae245066c0e94e2521fb4061a5cb41831952392
  languageName: node
  linkType: hard

"superjson@npm:^2.2.2":
  version: 2.2.2
  resolution: "superjson@npm:2.2.2"
  dependencies:
    copy-anything: "npm:^3.0.2"
  checksum: 10c0/aa49ebe6653e963020bc6a1ed416d267dfda84cfcc3cbd3beffd75b72e44eb9df7327215f3e3e77528f6e19ad8895b16a4964fdcd56d1799d14350db8c92afbc
  languageName: node
  linkType: hard

"supports-color@npm:^7.1.0":
  version: 7.2.0
  resolution: "supports-color@npm:7.2.0"
  dependencies:
    has-flag: "npm:^4.0.0"
  checksum: 10c0/afb4c88521b8b136b5f5f95160c98dee7243dc79d5432db7efc27efb219385bbc7d9427398e43dd6cc730a0f87d5085ce1652af7efbe391327bc0a7d0f7fc124
  languageName: node
  linkType: hard

"tailwindcss@npm:4.1.11":
  version: 4.1.11
  resolution: "tailwindcss@npm:4.1.11"
  checksum: 10c0/e23eed0a0d6557b3aff8ba320b82758988ca67c351ee9b33dfc646e83a64f6eaeca6183dfc97e931f7b2fab46e925090066edd697d2ede3f396c9fdeb4af24c1
  languageName: node
  linkType: hard

"tapable@npm:^2.2.0":
  version: 2.2.2
  resolution: "tapable@npm:2.2.2"
  checksum: 10c0/8ad130aa705cab6486ad89e42233569a1fb1ff21af115f59cebe9f2b45e9e7995efceaa9cc5062510cdb4ec673b527924b2ab812e3579c55ad659ae92117011e
  languageName: node
  linkType: hard

"tar@npm:^7.4.3":
  version: 7.4.3
  resolution: "tar@npm:7.4.3"
  dependencies:
    "@isaacs/fs-minipass": "npm:^4.0.0"
    chownr: "npm:^3.0.0"
    minipass: "npm:^7.1.2"
    minizlib: "npm:^3.0.1"
    mkdirp: "npm:^3.0.1"
    yallist: "npm:^5.0.0"
  checksum: 10c0/d4679609bb2a9b48eeaf84632b6d844128d2412b95b6de07d53d8ee8baf4ca0857c9331dfa510390a0727b550fd543d4d1a10995ad86cdf078423fbb8d99831d
  languageName: node
  linkType: hard

"text-hex@npm:1.0.x":
  version: 1.0.0
  resolution: "text-hex@npm:1.0.0"
  checksum: 10c0/57d8d320d92c79d7c03ffb8339b825bb9637c2cbccf14304309f51d8950015c44464b6fd1b6820a3d4821241c68825634f09f5a2d9d501e84f7c6fd14376860d
  languageName: node
  linkType: hard

"tinyglobby@npm:^0.2.12":
  version: 0.2.14
  resolution: "tinyglobby@npm:0.2.14"
  dependencies:
    fdir: "npm:^6.4.4"
    picomatch: "npm:^4.0.2"
  checksum: 10c0/f789ed6c924287a9b7d3612056ed0cda67306cd2c80c249fd280cf1504742b12583a2089b61f4abbd24605f390809017240e250241f09938054c9b363e51c0a6
  languageName: node
  linkType: hard

"to-regex-range@npm:^5.0.1":
  version: 5.0.1
  resolution: "to-regex-range@npm:5.0.1"
  dependencies:
    is-number: "npm:^7.0.0"
  checksum: 10c0/487988b0a19c654ff3e1961b87f471702e708fa8a8dd02a298ef16da7206692e8552a0250e8b3e8759270f62e9d8314616f6da274734d3b558b1fc7b7724e892
  languageName: node
  linkType: hard

"triple-beam@npm:^1.3.0":
  version: 1.4.1
  resolution: "triple-beam@npm:1.4.1"
  checksum: 10c0/4bf1db71e14fe3ff1c3adbe3c302f1fdb553b74d7591a37323a7badb32dc8e9c290738996cbb64f8b10dc5a3833645b5d8c26221aaaaa12e50d1251c9aba2fea
  languageName: node
  linkType: hard

"ts-api-utils@npm:^2.1.0":
  version: 2.1.0
  resolution: "ts-api-utils@npm:2.1.0"
  peerDependencies:
    typescript: ">=4.8.4"
  checksum: 10c0/9806a38adea2db0f6aa217ccc6bc9c391ddba338a9fe3080676d0d50ed806d305bb90e8cef0276e793d28c8a929f400abb184ddd7ff83a416959c0f4d2ce754f
  languageName: node
  linkType: hard

"tslib@npm:^2.4.0, tslib@npm:^2.8.0":
  version: 2.8.1
  resolution: "tslib@npm:2.8.1"
  checksum: 10c0/9c4759110a19c53f992d9aae23aac5ced636e99887b51b9e61def52611732872ff7668757d4e4c61f19691e36f4da981cd9485e869b4a7408d689f6bf1f14e62
  languageName: node
  linkType: hard

"tsx@npm:^4.19.3":
  version: 4.20.3
  resolution: "tsx@npm:4.20.3"
  dependencies:
    esbuild: "npm:~0.25.0"
    fsevents: "npm:~2.3.3"
    get-tsconfig: "npm:^4.7.5"
  dependenciesMeta:
    fsevents:
      optional: true
  bin:
    tsx: dist/cli.mjs
  checksum: 10c0/6ff0d91ed046ec743fac7ed60a07f3c025e5b71a5aaf58f3d2a6b45e4db114c83e59ebbb078c8e079e48d3730b944a02bc0de87695088aef4ec8bbc705dc791b
  languageName: node
  linkType: hard

"type-check@npm:^0.4.0, type-check@npm:~0.4.0":
  version: 0.4.0
  resolution: "type-check@npm:0.4.0"
  dependencies:
    prelude-ls: "npm:^1.2.1"
  checksum: 10c0/7b3fd0ed43891e2080bf0c5c504b418fbb3e5c7b9708d3d015037ba2e6323a28152ec163bcb65212741fa5d2022e3075ac3c76440dbd344c9035f818e8ecee58
  languageName: node
  linkType: hard

"type-fest@npm:^0.7.1":
  version: 0.7.1
  resolution: "type-fest@npm:0.7.1"
  checksum: 10c0/ce6b5ef806a76bf08d0daa78d65e61f24d9a0380bd1f1df36ffb61f84d14a0985c3a921923cf4b97831278cb6fa9bf1b89c751df09407e0510b14e8c081e4e0f
  languageName: node
  linkType: hard

"typescript-eslint@npm:^8.38.0":
  version: 8.38.0
  resolution: "typescript-eslint@npm:8.38.0"
  dependencies:
    "@typescript-eslint/eslint-plugin": "npm:8.38.0"
    "@typescript-eslint/parser": "npm:8.38.0"
    "@typescript-eslint/typescript-estree": "npm:8.38.0"
    "@typescript-eslint/utils": "npm:8.38.0"
  peerDependencies:
    eslint: ^8.57.0 || ^9.0.0
    typescript: ">=4.8.4 <5.9.0"
  checksum: 10c0/486b9862ee08f7827d808a2264ce03b58087b11c4c646c0da3533c192a67ae3fcb4e68d7a1e69d0f35a1edc274371a903a50ecfe74012d5eaa896cb9d5a81e0b
  languageName: node
  linkType: hard

"typescript@npm:^5.8.3":
  version: 5.8.3
  resolution: "typescript@npm:5.8.3"
  bin:
    tsc: bin/tsc
    tsserver: bin/tsserver
  checksum: 10c0/5f8bb01196e542e64d44db3d16ee0e4063ce4f3e3966df6005f2588e86d91c03e1fb131c2581baf0fb65ee79669eea6e161cd448178986587e9f6844446dbb48
  languageName: node
  linkType: hard

"typescript@patch:typescript@npm%3A^5.8.3#optional!builtin<compat/typescript>":
  version: 5.8.3
  resolution: "typescript@patch:typescript@npm%3A5.8.3#optional!builtin<compat/typescript>::version=5.8.3&hash=5786d5"
  bin:
    tsc: bin/tsc
    tsserver: bin/tsserver
  checksum: 10c0/39117e346ff8ebd87ae1510b3a77d5d92dae5a89bde588c747d25da5c146603a99c8ee588c7ef80faaf123d89ed46f6dbd918d534d641083177d5fac38b8a1cb
  languageName: node
  linkType: hard

"unique-filename@npm:^4.0.0":
  version: 4.0.0
  resolution: "unique-filename@npm:4.0.0"
  dependencies:
    unique-slug: "npm:^5.0.0"
  checksum: 10c0/38ae681cceb1408ea0587b6b01e29b00eee3c84baee1e41fd5c16b9ed443b80fba90c40e0ba69627e30855570a34ba8b06702d4a35035d4b5e198bf5a64c9ddc
  languageName: node
  linkType: hard

"unique-slug@npm:^5.0.0":
  version: 5.0.0
  resolution: "unique-slug@npm:5.0.0"
  dependencies:
    imurmurhash: "npm:^0.1.4"
  checksum: 10c0/d324c5a44887bd7e105ce800fcf7533d43f29c48757ac410afd42975de82cc38ea2035c0483f4de82d186691bf3208ef35c644f73aa2b1b20b8e651be5afd293
  languageName: node
  linkType: hard

"update-browserslist-db@npm:^1.1.3":
  version: 1.1.3
  resolution: "update-browserslist-db@npm:1.1.3"
  dependencies:
    escalade: "npm:^3.2.0"
    picocolors: "npm:^1.1.1"
  peerDependencies:
    browserslist: ">= 4.21.0"
  bin:
    update-browserslist-db: cli.js
  checksum: 10c0/682e8ecbf9de474a626f6462aa85927936cdd256fe584c6df2508b0df9f7362c44c957e9970df55dfe44d3623807d26316ea2c7d26b80bb76a16c56c37233c32
  languageName: node
  linkType: hard

"uri-js@npm:^4.2.2":
  version: 4.4.1
  resolution: "uri-js@npm:4.4.1"
  dependencies:
    punycode: "npm:^2.1.0"
  checksum: 10c0/4ef57b45aa820d7ac6496e9208559986c665e49447cb072744c13b66925a362d96dd5a46c4530a6b8e203e5db5fe849369444440cb22ecfc26c679359e5dfa3c
  languageName: node
  linkType: hard

"util-deprecate@npm:^1.0.1, util-deprecate@npm:^1.0.2":
  version: 1.0.2
  resolution: "util-deprecate@npm:1.0.2"
  checksum: 10c0/41a5bdd214df2f6c3ecf8622745e4a366c4adced864bc3c833739791aeeeb1838119af7daed4ba36428114b5c67dcda034a79c882e97e43c03e66a4dd7389942
  languageName: node
  linkType: hard

"uuid@npm:^10.0.0":
  version: 10.0.0
  resolution: "uuid@npm:10.0.0"
  bin:
    uuid: dist/bin/uuid
  checksum: 10c0/eab18c27fe4ab9fb9709a5d5f40119b45f2ec8314f8d4cf12ce27e4c6f4ffa4a6321dc7db6c515068fa373c075b49691ba969f0010bf37f44c37ca40cd6bf7fe
  languageName: node
  linkType: hard

"uuid@npm:^9.0.0":
  version: 9.0.1
  resolution: "uuid@npm:9.0.1"
  bin:
    uuid: dist/bin/uuid
  checksum: 10c0/1607dd32ac7fc22f2d8f77051e6a64845c9bce5cd3dd8aa0070c074ec73e666a1f63c7b4e0f4bf2bc8b9d59dc85a15e17807446d9d2b17c8485fbc2147b27f9b
  languageName: node
  linkType: hard

"which@npm:^2.0.1":
  version: 2.0.2
  resolution: "which@npm:2.0.2"
  dependencies:
    isexe: "npm:^2.0.0"
  bin:
    node-which: ./bin/node-which
  checksum: 10c0/66522872a768b60c2a65a57e8ad184e5372f5b6a9ca6d5f033d4b0dc98aff63995655a7503b9c0a2598936f532120e81dd8cc155e2e92ed662a2b9377cc4374f
  languageName: node
  linkType: hard

"which@npm:^5.0.0":
  version: 5.0.0
  resolution: "which@npm:5.0.0"
  dependencies:
    isexe: "npm:^3.1.1"
  bin:
    node-which: bin/which.js
  checksum: 10c0/e556e4cd8b7dbf5df52408c9a9dd5ac6518c8c5267c8953f5b0564073c66ed5bf9503b14d876d0e9c7844d4db9725fb0dcf45d6e911e17e26ab363dc3965ae7b
  languageName: node
  linkType: hard

"winston-console-format@npm:^1.0.8":
  version: 1.0.8
  resolution: "winston-console-format@npm:1.0.8"
  dependencies:
    colors: "npm:^1.4.0"
    logform: "npm:^2.2.0"
    triple-beam: "npm:^1.3.0"
  checksum: 10c0/67839ac8f533617747ea3c22a14f2b3cd5bb07dcf91bb25c87f1ad3aa9850d30e5960c44e92726a9cd4c239611dc5171f6b0f4d3d9fcf58ca1ad5323a1fb81c5
  languageName: node
  linkType: hard

"winston-transport@npm:^4.9.0":
  version: 4.9.0
  resolution: "winston-transport@npm:4.9.0"
  dependencies:
    logform: "npm:^2.7.0"
    readable-stream: "npm:^3.6.2"
    triple-beam: "npm:^1.3.0"
  checksum: 10c0/e2990a172e754dbf27e7823772214a22dc8312f7ec9cfba831e5ef30a5d5528792e5ea8f083c7387ccfc5b2af20e3691f64738546c8869086110a26f98671095
  languageName: node
  linkType: hard

"winston@npm:^3.17.0":
  version: 3.17.0
  resolution: "winston@npm:3.17.0"
  dependencies:
    "@colors/colors": "npm:^1.6.0"
    "@dabh/diagnostics": "npm:^2.0.2"
    async: "npm:^3.2.3"
    is-stream: "npm:^2.0.0"
    logform: "npm:^2.7.0"
    one-time: "npm:^1.0.0"
    readable-stream: "npm:^3.4.0"
    safe-stable-stringify: "npm:^2.3.1"
    stack-trace: "npm:0.0.x"
    triple-beam: "npm:^1.3.0"
    winston-transport: "npm:^4.9.0"
  checksum: 10c0/ec8eaeac9a72b2598aedbff50b7dac82ce374a400ed92e7e705d7274426b48edcb25507d78cff318187c4fb27d642a0e2a39c57b6badc9af8e09d4a40636a5f7
  languageName: node
  linkType: hard

"word-wrap@npm:^1.2.5":
  version: 1.2.5
  resolution: "word-wrap@npm:1.2.5"
  checksum: 10c0/e0e4a1ca27599c92a6ca4c32260e8a92e8a44f4ef6ef93f803f8ed823f486e0889fc0b93be4db59c8d51b3064951d25e43d434e95dc8c960cc3a63d65d00ba20
  languageName: node
  linkType: hard

"wrap-ansi-cjs@npm:wrap-ansi@^7.0.0":
  version: 7.0.0
  resolution: "wrap-ansi@npm:7.0.0"
  dependencies:
    ansi-styles: "npm:^4.0.0"
    string-width: "npm:^4.1.0"
    strip-ansi: "npm:^6.0.0"
  checksum: 10c0/d15fc12c11e4cbc4044a552129ebc75ee3f57aa9c1958373a4db0292d72282f54373b536103987a4a7594db1ef6a4f10acf92978f79b98c49306a4b58c77d4da
  languageName: node
  linkType: hard

"wrap-ansi@npm:^8.1.0":
  version: 8.1.0
  resolution: "wrap-ansi@npm:8.1.0"
  dependencies:
    ansi-styles: "npm:^6.1.0"
    string-width: "npm:^5.0.1"
    strip-ansi: "npm:^7.0.1"
  checksum: 10c0/138ff58a41d2f877eae87e3282c0630fc2789012fc1af4d6bd626eeb9a2f9a65ca92005e6e69a75c7b85a68479fe7443c7dbe1eb8fbaa681a4491364b7c55c60
  languageName: node
  linkType: hard

"wsl-utils@npm:^0.1.0":
  version: 0.1.0
  resolution: "wsl-utils@npm:0.1.0"
  dependencies:
    is-wsl: "npm:^3.1.0"
  checksum: 10c0/44318f3585eb97be994fc21a20ddab2649feaf1fbe893f1f866d936eea3d5f8c743bec6dc02e49fbdd3c0e69e9b36f449d90a0b165a4f47dd089747af4cf2377
  languageName: node
  linkType: hard

"yallist@npm:^4.0.0":
  version: 4.0.0
  resolution: "yallist@npm:4.0.0"
  checksum: 10c0/2286b5e8dbfe22204ab66e2ef5cc9bbb1e55dfc873bbe0d568aa943eb255d131890dfd5bf243637273d31119b870f49c18fcde2c6ffbb7a7a092b870dc90625a
  languageName: node
  linkType: hard

"yallist@npm:^5.0.0":
  version: 5.0.0
  resolution: "yallist@npm:5.0.0"
  checksum: 10c0/a499c81ce6d4a1d260d4ea0f6d49ab4da09681e32c3f0472dee16667ed69d01dae63a3b81745a24bd78476ec4fcf856114cb4896ace738e01da34b2c42235416
  languageName: node
  linkType: hard

"yocto-queue@npm:^0.1.0":
  version: 0.1.0
  resolution: "yocto-queue@npm:0.1.0"
  checksum: 10c0/dceb44c28578b31641e13695d200d34ec4ab3966a5729814d5445b194933c096b7ced71494ce53a0e8820685d1d010df8b2422e5bf2cdea7e469d97ffbea306f
  languageName: node
  linkType: hard

"zod-to-json-schema@npm:^3.22.3":
  version: 3.24.6
  resolution: "zod-to-json-schema@npm:3.24.6"
  peerDependencies:
    zod: ^3.24.1
  checksum: 10c0/b907ab6d057100bd25a37e5545bf5f0efa5902cd84d3c3ec05c2e51541431a47bd9bf1e5e151a244273409b45f5986d55b26e5d207f98abc5200702f733eb368
  languageName: node
  linkType: hard

"zod@npm:^3.23.8, zod@npm:^3.25.32":
  version: 3.25.76
  resolution: "zod@npm:3.25.76"
  checksum: 10c0/5718ec35e3c40b600316c5b4c5e4976f7fee68151bc8f8d90ec18a469be9571f072e1bbaace10f1e85cf8892ea12d90821b200e980ab46916a6166a4260a983c
  languageName: node
  linkType: hard

"zod@npm:^4.0.10":
  version: 4.0.10
  resolution: "zod@npm:4.0.10"
  checksum: 10c0/8d1145e767c22b571a7967c198632f69ef15ce571b5021cdba84cf31d9af2ca40b033ea2fcbe5797cfd2da9c67b3a6ebe435938eabfbb1d1f3ab2f17f00f443b
  languageName: node
  linkType: hard

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/js_translation/codeblocks/.yarn/install-state.gz
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/third_party_page/__init__.py
```py

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/third_party_page/create_third_party_page.py
```py
#!/usr/bin/env python
"""Create the third party page for the documentation."""

import argparse
from typing import List
from typing import TypedDict

import yaml

MARKDOWN = """\
[//]: # (This file is automatically generated using a script in docs/_scripts. Do not edit this file directly!)
# Community Agents

If you’re looking for other prebuilt libraries, explore the community-built options 
below. These libraries can extend LangGraph's functionality in various ways.

## 📚 Available Libraries
[//]: # (This file is automatically generated using a script in docs/_scripts. Do not edit this file directly!)

:::python
{python_library_list}

## ✨ Contributing Your Library

Have you built an awesome open-source library using LangGraph? We'd love to feature 
your project on the official LangGraph documentation pages! 🏆

To share your project, simply open a Pull Request adding an entry for your package in our [packages.yml]({langgraph_url}) file.

**Guidelines**

- Your repo must be distributed as an installable package on PyPI 📦
- The repo should either use the Graph API (exposing a `StateGraph` instance) or 
  the Functional API (exposing an `entrypoint`).
- The package must include documentation (e.g., a `README.md` or docs site) 
  explaining how to use it.

We'll review your contribution and merge it in!

Thanks for contributing! 🚀
:::

:::js
{js_library_list}

## ✨ Contributing Your Library

Have you built an awesome open-source library using LangGraph? We'd love to feature 
your project on the official LangGraph documentation pages! 🏆

To share your project, simply open a Pull Request adding an entry for your package in our [packages.yml]({langgraph_url}) file.

**Guidelines**

- Your repo must be distributed as an installable package on npm 📦
- The repo should either use the Graph API (exposing a `StateGraph` instance) or 
  the Functional API (exposing an `entrypoint`).
- The package must include documentation (e.g., a `README.md` or docs site) 
  explaining how to use it.

We'll review your contribution and merge it in!

Thanks for contributing! 🚀
:::
"""


class ResolvedPackage(TypedDict):
    name: str
    """The name of the package."""
    repo: str
    """Repository ID within github. Format is: [orgname]/[repo_name]."""
    monorepo_path: str | None
    """Optional: The path to the package in the monorepo. Must be relative to the root of the monorepo."""
    language: str
    """The language of the package. (either 'python' or 'js')"""
    weekly_downloads: int | None
    """The weekly download count of the package."""
    description: str
    """A brief description of what the package does."""

def generate_package_table(resolved_packages: List[ResolvedPackage]) -> str:
    """Generate the package table for the third party page.
    """
    sorted_packages = sorted(
        resolved_packages, key=lambda p: p["weekly_downloads"] or 0, reverse=True
    )
    rows = [
        "| Name | GitHub URL | Description | Weekly Downloads | Stars |",
        "| --- | --- | --- | --- | --- |",
    ]
    for package in sorted_packages:
        name = f"**{package['name']}**"
        
        monorepo_path = package.get("monorepo_path", "")
        if monorepo_path:
            monorepo_path = monorepo_path[1:] if monorepo_path.startswith('/') else monorepo_path
            repo_url_suffix = f"/tree/main/{monorepo_path}"
        else:
            repo_url_suffix = ""
        repo_url = f"https://github.com/{package['repo']}{repo_url_suffix}"

        stars_badge = (
            f"https://img.shields.io/github/stars/{package['repo']}?style=social"
        )
        stars = f"![GitHub stars]({stars_badge})"
        downloads = package["weekly_downloads"] or "-"
        row = f"| {name} | {repo_url} | {package['description']} | {downloads} | {stars}"
        rows.append(row)
    return "\n".join(rows)

def generate_markdown(resolved_packages: List[ResolvedPackage]) -> str:
    """Generate the markdown content for the third party page.

    Args:
        resolved_packages: A list of resolved package information.

    Returns:
        The markdown content as a string.
    """
    # Update the URL to the actual file once the initial version is merged
    langgraph_url = (
        "https://github.com/langchain-ai/langgraph/blob/main/docs"
        "/_scripts/third_party_page/packages.yml"
    )

    python_library_list = generate_package_table(
        [p for p in resolved_packages if p["language"] == "python"]
    )
    js_library_list = generate_package_table(
        [p for p in resolved_packages if p["language"] == "js"]
    )
    
    markdown_content = MARKDOWN.format(
        python_library_list=python_library_list,
        js_library_list=js_library_list,
        langgraph_url=langgraph_url,
    )
    return markdown_content


def main(input_file: str, output_file: str) -> None:
    """Main function to create the third party page.

    Args:
        input_file: Path to the input YAML file containing resolved package information.
        output_file: Path to the output file for the third party page.
        language: The language for which to generate the third party page.
    """
    # Parse the input YAML file
    with open(input_file, "r") as f:
        resolved_packages: List[ResolvedPackage] = yaml.safe_load(f)

    markdown_content = generate_markdown(resolved_packages)

    # Write the markdown content to the output file
    with open(output_file, "w", encoding="utf-8") as f:
        f.write(markdown_content)


if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Create the third party page.")
    parser.add_argument(
        "input_file",
        help="Path to the input YAML file containing resolved package information.",
    )
    parser.add_argument(
        "output_file", help="Path to the output file for the third party page."
    )
    args = parser.parse_args()

    main(args.input_file, args.output_file)

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/third_party_page/get_download_stats.py
```py
#!/usr/bin/env python
"""Retrieve download count for a list of Python packages from PyPI."""

import argparse
from datetime import datetime
from typing import TypedDict
import pathlib

import requests
import yaml


class Package(TypedDict):
    name: str
    """The name of the package."""
    repo: str
    """Repository ID within github. Format is: [orgname]/[repo_name]."""
    monorepo_path: str | None
    """The path to the package in the monorepo. Only used for JS packages."""
    description: str
    """A brief description of what the package does."""


class ResolvedPackage(Package):
    weekly_downloads: int | None
    """The weekly download count of the package."""
    language: str
    """The language of the package. (either 'python' or 'js')"""


HERE = pathlib.Path(__file__).parent
PACKAGES_FILE = HERE / "packages.yml"
PACKAGES = yaml.safe_load(PACKAGES_FILE.read_text())["packages"]


def _get_pypi_downloads(package: Package) -> int:
    """Retrieve the weekly download count for a package from PyPIStats."""

    # First check if package exists on PyPI
    pypi_url = f"https://pypi.org/pypi/{package['name']}/json"
    try:
        pypi_response = requests.get(pypi_url)
        pypi_response.raise_for_status()
    except requests.exceptions.HTTPError:
        raise AssertionError(f"Package {package['name']} does not exist on PyPI")

    # Get first release date
    pypi_data = pypi_response.json()
    releases = pypi_data["releases"]
    first_release_date = None
    for version_releases in releases.values():
        if version_releases:  # Some versions may be empty lists
            upload_time = datetime.fromisoformat(version_releases[0]["upload_time"])
            if first_release_date is None or upload_time < first_release_date:
                first_release_date = upload_time

    if first_release_date is None:
        raise AssertionError(f"Package {package['name']} has no releases yet")

    # If package was published in last 48 hours, skip download stats
    if (datetime.now() - first_release_date).total_seconds() >= 48 * 3600:
        url = f"https://pypistats.org/api/packages/{package['name']}/overall"

        response = requests.get(url)
        response.raise_for_status()
        data = response.json()

        sorted_data = sorted(
            data["data"],
            key=lambda x: datetime.strptime(x["date"], "%Y-%m-%d"),
            reverse=True,
        )

        # Sum the last 7 days of downloads
        return sum(entry["downloads"] for entry in sorted_data[:7])
    else:
        return None


def _get_npm_downloads(package: Package) -> int:
    """Retrieve the weekly download count for a package on the npm registry."""

    # Check if package exists on the npm registry
    npm_url = f"https://registry.npmjs.org/{package['name']}"
    try:
        npm_response = requests.get(npm_url)
        npm_response.raise_for_status()
    except requests.exceptions.HTTPError:
        raise AssertionError(
            f"Package {package['name']} does not exist on npm registry"
        )

    npm_data = npm_response.json()

    # Retrieve the first publish date using the 'created' timestamp from the 'time' field.
    created_str = npm_data.get("time", {}).get("created")
    if created_str is None:
        raise AssertionError(
            f"Package {package['name']} has no creation time in registry data"
        )
    # Remove the trailing 'Z' if present and parse the ISO format timestamp
    first_publish_date = datetime.fromisoformat(created_str.rstrip("Z"))

    # If package was published more than 48 hours ago, fetch download stats.
    if (datetime.now() - first_publish_date).total_seconds() >= 48 * 3600:
        stats_url = f"https://api.npmjs.org/downloads/point/last-week/{package['name']}"
        stats_response = requests.get(stats_url)
        stats_response.raise_for_status()
        stats_data = stats_response.json()
        return stats_data.get("downloads", None)
    else:
        return None


def _get_weekly_downloads(
    packages: dict[str, list[Package]], fake: bool
) -> list[ResolvedPackage]:
    """Retrieve the weekly download count for a dictionary of python or js packages."""
    resolved_packages: list[ResolvedPackage] = []

    if fake:
        # To avoid making network requests during testing, return fake download counts
        for language, package_list in packages.items():
            for package in package_list:
                resolved_packages.append(
                    {
                        "name": package["name"],
                        "repo": package["repo"],
                        "monorepo_path": package.get("monorepo_path", None),
                        "language": language,
                        "description": package["description"],
                        "weekly_downloads": -12345,
                    }
                )
        return resolved_packages

    for language, package_list in packages.items():
        for package in package_list:
            if language == "python":
                num_downloads = _get_pypi_downloads(package)
            elif language == "js":
                num_downloads = _get_npm_downloads(package)
            else:
                num_downloads = None

            resolved_packages.append(
                {
                    "name": package["name"],
                    "repo": package["repo"],
                    "monorepo_path": package.get("monorepo_path", None),
                    "language": language,
                    "description": package["description"],
                    "weekly_downloads": num_downloads,
                }
            )

    return resolved_packages


def main(output_file: str, fake: bool) -> None:
    """Main function to generate package download information.

    Args:
        output_file: Path to the output YAML file.
        fake: If `True`, use fake download counts for testing purposes.
    """
    resolved_packages: list[ResolvedPackage] = _get_weekly_downloads(PACKAGES, fake)

    if not output_file.endswith(".yml"):
        raise ValueError("Output file must have a .yml extension")

    with open(output_file, "w") as f:
        f.write("# This file is auto-generated. Do not edit.\n")
        yaml.dump(resolved_packages, f)


if __name__ == "__main__":
    parser = argparse.ArgumentParser(
        description="Generate package download information."
    )
    parser.add_argument(
        "output_file",
        help=(
            "Path to the output YAML file. Example: python generate_downloads.py "
            "downloads.yml"
        ),
    )
    parser.add_argument(
        "--fake",
        default=False,
        action="store_true",
        help=(
            "Generate fake download counts for testing purposes. "
            "This option will not make any network requests."
        ),
    )
    args = parser.parse_args()

    main(args.output_file, args.fake)

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/_scripts/third_party_page/packages.yml
```yml
#A list of third-party packages to surface on the third-party page.
packages:
  python:
    - name: "trustcall"
      repo: "hinthornw/trustcall"
      description: "Tenacious tool calling built on LangGraph."
    - name: "breeze-agent"
      repo: "andrestorres123/breeze-agent"
      description: "A streamlined research system built inspired on STORM and built on LangGraph."
    - name: "langgraph-supervisor"
      repo: "langchain-ai/langgraph-supervisor-py"
      description: "Build supervisor multi-agent systems with LangGraph."
    - name: "langmem"
      repo: "langchain-ai/langmem"
      description: "Build agents that learn and adapt from interactions over time."
    - name: "langchain-mcp-adapters"
      repo: "langchain-ai/langchain-mcp-adapters"
      description: "Make Anthropic Model Context Protocol (MCP) tools compatible with LangGraph agents."
    - name: "open-deep-research"
      repo: "langchain-ai/open_deep_research"
      description: "Open source assistant for iterative web research and report writing."
    - name: "langgraph-swarm"
      repo: "langchain-ai/langgraph-swarm-py"
      description: "Build swarm-style multi-agent systems using LangGraph."
    - name: "delve-taxonomy-generator"
      repo: "andrestorres123/delve"
      description: "A taxonomy generator for unstructured data"
    - name: "nodeology"
      repo: "xyin-anl/Nodeology"
      description: "Enable researcher to build scientific workflows easily with simplified interface."
    - name: "langgraph-bigtool"
      repo: "langchain-ai/langgraph-bigtool"
      description: "Build LangGraph agents with large numbers of tools."
    - name: "ai-data-science-team"
      repo: "business-science/ai-data-science-team"
      description: "An AI-powered data science team of agents to help you perform common data science tasks 10X faster."
    - name: "langgraph-reflection"
      repo: "langchain-ai/langgraph-reflection"
      description: "LangGraph agent that runs a reflection step."
    - name: "langgraph-codeact"
      repo: "langchain-ai/langgraph-codeact"
      description: "LangGraph implementation of CodeAct agent that generates and executes code instead of tool calling."
  js:
    - name: "@langchain/mcp-adapters"
      repo: "langchain-ai/langchainjs"
      description: "Make Anthropic Model Context Protocol (MCP) tools compatible with LangGraph agents."
    - name: "@langchain/langgraph-supervisor"
      repo: "langchain-ai/langgraphjs"
      monorepo_path: "libs/langgraph-supervisor"
      description: "Build supervisor multi-agent systems with LangGraph"
    - name: "@langchain/langgraph-swarm"
      repo: "langchain-ai/langgraphjs"
      monorepo_path: "libs/langgraph-swarm"
      description: "Build multi-agent swarms with LangGraph"
    - name: "@langchain/langgraph-cua"
      repo: "langchain-ai/langgraphjs"
      monorepo_path: "libs/langgraph-cua"
      description: "Build computer use agents with LangGraph"

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/tests/__init__.py
```py

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/tests/unit_tests/__init__.py
```py

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/tests/unit_tests/test_api_reference.py
```py
"""Test generation of links into the API reference."""

import pytest

from _scripts.generate_api_reference_links import (
    update_markdown_with_imports,
    get_imports,
)

MARKDOWN_IMPORTS = """\
```python
from langgraph.types import interrupt
```
"""

EXPECTED_MARKDOWN = """\
<sup><i>API Reference: <a href="https://langchain-ai.github.io/langgraph/reference/types/#langgraph.types.interrupt">interrupt</a></i></sup>

```python
from langgraph.types import interrupt
```
"""


def test_update_markdown_with_imports() -> None:
    """Light weight end-to-end test."""
    assert (
        update_markdown_with_imports(MARKDOWN_IMPORTS, "some_path") == EXPECTED_MARKDOWN
    )


@pytest.mark.parametrize(
    "code_block, expected_imports",
    [
        (
            "from langgraph.types import interrupt",
            [
                {
                    "docs": "https://langchain-ai.github.io/langgraph/reference/types/#langgraph.types.interrupt",
                    "imported": "interrupt",
                    "path": "some_path",
                    "source": "langgraph.types",
                }
            ],
        ),
        (
            "from langgraph.types import ( interrupt )",
            [
                {
                    "docs": "https://langchain-ai.github.io/langgraph/reference/types/#langgraph.types.interrupt",
                    "imported": "interrupt",
                    "path": "some_path",
                    "source": "langgraph.types",
                }
            ],
        ),
        (
            "from langgraph.types import interrupt as foo",
            [
                {
                    "docs": "https://langchain-ai.github.io/langgraph/reference/types/#langgraph.types.interrupt",
                    "imported": "interrupt",
                    "path": "some_path",
                    "source": "langgraph.types",
                }
            ],
        ),
    ],
)
def test_get_imports(code_block: str, expected_imports: list) -> None:
    """Get imports from a code block."""
    assert (
        get_imports(code_block, "some_path") == expected_imports
    ), f"Failed for code_block=`{code_block}`"


@pytest.mark.parametrize(
    "code, expected_imports",
    [
        # Single import without parenthesis
        (
            "from langgraph.types import interrupt",
            [
                {
                    "source": "langgraph.types",
                    "imported": "interrupt",
                }
            ],
        ),
        # Multiple imports
        (
            (
                "from langgraph.types import interrupt\n"
                "from langgraph.func import task"
            ),
            [
                {
                    "source": "langgraph.types",
                    "imported": "interrupt",
                },
                {
                    "source": "langgraph.func",
                    "imported": "task",
                },
            ],
        ),
        # Single import with parenthesis and extra whitespace
        (
            "from langgraph.types import ( interrupt )",
            [
                {
                    "source": "langgraph.types",
                    "imported": "interrupt",
                }
            ],
        ),
        # Single import with an alias
        (
            "from langgraph.types import interrupt as foo",
            [
                {
                    "source": "langgraph.types",
                    "imported": "interrupt",
                }
            ],
        ),
        # Multiple imports on one line with an alias
        (
            "from langgraph.types import interrupt, StreamWriter as bar",
            [
                {
                    "source": "langgraph.types",
                    "imported": "interrupt",
                },
                {
                    "source": "langgraph.types",
                    "imported": "StreamWriter",
                },
            ],
        ),
        # Multiple imports without aliases
        (
            "from langgraph.types import interrupt, StreamWriter",
            [
                {
                    "source": "langgraph.types",
                    "imported": "interrupt",
                },
                {
                    "source": "langgraph.types",
                    "imported": "StreamWriter",
                },
            ],
        ),
        # Multiline import with parenthesis and trailing comma
        (
            """from langgraph.types import (
            interrupt,
            StreamWriter as foo,
            Command,
        )""",
            [
                {
                    "source": "langgraph.types",
                    "imported": "interrupt",
                },
                {
                    "source": "langgraph.types",
                    "imported": "StreamWriter",
                },
                {
                    "source": "langgraph.types",
                    "imported": "Command",
                },
            ],
        ),
        # Multiline import with parenthesis and trailing comma
        (
            (
                "from langgraph.types import (\n"
                "        interrupt,\n"
                "        StreamWriter as foo\n,"
                "        Command,\n"
                ")\n"
                "def foo():\n"
                "    pass\n"
                ""
            ),
            [
                {
                    "source": "langgraph.types",
                    "imported": "interrupt",
                },
                {
                    "source": "langgraph.types",
                    "imported": "StreamWriter",
                },
                {
                    "source": "langgraph.types",
                    "imported": "Command",
                },
            ],
        ),
    ],
)
def test_regexp_matching(code: str, expected_imports: list) -> None:
    results = get_imports(code, "some_path")
    for result in results:
        del result["docs"]
        del result["path"]

    assert results == expected_imports

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/tests/unit_tests/test_auto_links.py
```py
"""Unit tests for cross-reference preprocessing functionality."""

from unittest.mock import patch

import pytest

from _scripts.handle_auto_links import _transform_link, _replace_autolinks


@pytest.fixture
def mock_link_maps():
    """Fixture providing mock link maps for testing."""
    mock_scope_maps = {
        "python": {"py-link": "https://example.com/python"},
        "js": {"js-link": "https://example.com/js"},
    }

    with patch("_scripts.handle_auto_links.SCOPE_LINK_MAPS", mock_scope_maps):
        yield mock_scope_maps


def test_transform_link_basic(mock_link_maps) -> None:
    """Test basic link transformation."""
    # Test with a known link
    result = _transform_link("py-link", "python", "test.md", 1)
    assert result == "[py-link](https://example.com/python)"

    # Test with an unknown link (returns None)
    result = _transform_link("unknown-link", "global", "test.md", 1)
    assert result is None


def test_transform_link_with_custom_title(mock_link_maps) -> None:
    """Test link transformation with custom title."""
    # Test with a known link and custom title
    result = _transform_link("py-link", "python", "test.md", 1, "Custom Python Link")
    assert result == "[Custom Python Link](https://example.com/python)"

    # Test with unknown link and custom title (should still return None)
    result = _transform_link("unknown-link", "python", "test.md", 1, "Custom Title")
    assert result is None


def test_no_cross_refs(mock_link_maps) -> None:
    """Test markdown with no @[references]."""
    lines = ["# Title\n", "Regular text.\n"]
    markdown = "".join(lines)
    result = _replace_autolinks(markdown, "test.md")
    expected = "".join(["# Title\n", "Regular text.\n"])
    assert result == expected


def test_global_cross_refs(mock_link_maps) -> None:
    """Test @[references] in global scope (no conditional blocks)."""
    lines = ["@[global-link]\n", "Text with @[unknown-link].\n"]
    markdown = "".join(lines)
    result = _replace_autolinks(markdown, "test.md")
    expected = "".join(["@[global-link]\n", "Text with @[unknown-link].\n"])
    assert result == expected


def test_python_conditional_block(mock_link_maps) -> None:
    """Test @[references] inside Python conditional block."""
    lines = [":::python\n", "@[py-link]\n", ":::\n"]
    markdown = "".join(lines)
    result = _replace_autolinks(markdown, "test.md")
    expected = "".join(
        [":::python\n", "[py-link](https://example.com/python)\n", ":::\n"]
    )
    assert result == expected


def test_js_conditional_block(mock_link_maps) -> None:
    """Test @[references] inside JavaScript conditional block."""
    lines = [":::js\n", "@[js-link]\n", ":::\n"]
    markdown = "".join(lines)
    result = _replace_autolinks(markdown, "test.md")
    expected = "".join([":::js\n", "[js-link](https://example.com/js)\n", ":::\n"])
    assert result == expected


def test_all_scopes(mock_link_maps) -> None:
    """Test @[references] in global, Python, and JavaScript scopes."""
    lines = [
        "@[global-link]\n",
        ":::python\n",
        "@[py-link]\n",
        ":::\n",
        "@[global-link]\n",
        ":::js\n",
        "@[js-link]\n",
        ":::\n",
        "@[global-link]\n",
    ]
    markdown = "".join(lines)
    result = _replace_autolinks(markdown, "test.md")
    expected = "".join(
        [
            "@[global-link]\n",
            ":::python\n",
            "[py-link](https://example.com/python)\n",
            ":::\n",
            "@[global-link]\n",
            ":::js\n",
            "[js-link](https://example.com/js)\n",
            ":::\n",
            "@[global-link]\n",
        ]
    )
    assert result == expected


def test_fence_resets_to_global(mock_link_maps) -> None:
    """Test that closing fence resets scope to global."""
    lines = [":::python\n", "@[py-link]\n", ":::\n", "@[global-link]\n"]
    markdown = "".join(lines)
    result = _replace_autolinks(markdown, "test.md")
    expected = "".join(
        [
            ":::python\n",
            "[py-link](https://example.com/python)\n",
            ":::\n",
            "@[global-link]\n",
        ]
    )
    assert result == expected


def test_indented_conditional_fences(mock_link_maps) -> None:
    """Test @[references] inside indented conditional fences (e.g., in tabs or admonitions)."""
    lines = [
        "@[global-link]\n",
        "    :::python\n",
        "    @[py-link]\n",
        "    :::\n",
        "@[global-link]\n",
        "\t\t:::js\n",
        "\t\t@[js-link]\n",
        "\t\t:::\n",
        "@[global-link]\n",
    ]
    markdown = "".join(lines)
    result = _replace_autolinks(markdown, "test.md")
    expected = "".join(
        [
            "@[global-link]\n",
            "    :::python\n",
            "    [py-link](https://example.com/python)\n",
            "    :::\n",
            "@[global-link]\n",
            "\t\t:::js\n",
            "\t\t[js-link](https://example.com/js)\n",
            "\t\t:::\n",
            "@[global-link]\n",
        ]
    )
    assert result == expected


def test_custom_title_syntax(mock_link_maps) -> None:
    """Test @[title][ref] syntax with custom titles."""
    lines = [
        ":::python\n",
        "@[Custom Python Title][py-link]\n",
        ":::\n",
        ":::js\n", 
        "@[Custom JS Title][js-link]\n",
        ":::\n"
    ]
    markdown = "".join(lines)
    result = _replace_autolinks(markdown, "test.md")
    expected = "".join([
        ":::python\n",
        "[Custom Python Title](https://example.com/python)\n",
        ":::\n",
        ":::js\n",
        "[Custom JS Title](https://example.com/js)\n",
        ":::\n"
    ])
    assert result == expected


def test_mixed_syntax_compatibility(mock_link_maps) -> None:
    """Test that both @[ref] and @[title][ref] syntax work together."""
    lines = [
        ":::python\n",
        "@[py-link]\n",  # Old syntax
        "@[Custom Title][py-link]\n",  # New syntax
        ":::\n"
    ]
    markdown = "".join(lines)
    result = _replace_autolinks(markdown, "test.md")
    expected = "".join([
        ":::python\n",
        "[py-link](https://example.com/python)\n",
        "[Custom Title](https://example.com/python)\n",
        ":::\n"
    ])
    assert result == expected


def test_custom_title_with_unknown_link(mock_link_maps) -> None:
    """Test @[title][ref] syntax with unknown reference."""
    lines = [
        ":::python\n",
        "@[Custom Title][unknown-link]\n",
        ":::\n"
    ]
    markdown = "".join(lines)
    result = _replace_autolinks(markdown, "test.md")
    expected = "".join([
        ":::python\n",
        "@[Custom Title][unknown-link]\n",  # Should remain unchanged
        ":::\n"
    ])
    assert result == expected

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/tests/unit_tests/test_conditional_rendering.py
```py
from _scripts.notebook_hooks import _apply_conditional_rendering


CONDITIONAL_RENDERING = """
above
:::js
js-content
:::
between
:::python
python-content
:::
below
"""


def test_conditional_rendering() -> None:
    """Test logic for conditional rendering of content."""
    output = _apply_conditional_rendering(CONDITIONAL_RENDERING, "js")
    assert output.strip() == "above\njs-content\n\nbetween\n\nbelow"
    output = _apply_conditional_rendering(CONDITIONAL_RENDERING, "python")
    assert output.strip() == "above\n\nbetween\npython-content\n\nbelow"

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/tests/unit_tests/test_highlights.py
```py
from mkdocs.config.defaults import MkDocsConfig
from mkdocs.structure.files import File
from mkdocs.structure.pages import Page

from _scripts.notebook_hooks import _highlight_code_blocks, on_page_markdown

NO_OP_INPUT_1 = """\
This is a plain text without any code blocks.

```python
print("Hello, World!")
```
"""

NO_OP_INPUT_2 = """\

=== "Python"

    ```python
        def foo():
            pass
    print("Hello, World!")
    ```
"""


def test_highlight_code_blocks_no_op() -> None:
    assert _highlight_code_blocks(NO_OP_INPUT_1) == NO_OP_INPUT_1
    assert _highlight_code_blocks(NO_OP_INPUT_2) == NO_OP_INPUT_2


# Examples are written in multiline style to make sure that whitespace
# is easy to interpret.
INPUT_HIGHLIGHT_1 = """\
This is a plain text without any code blocks.

```python
# highlight-next-line
print("Hello, World!")
```
"""

EXPECTED_HIGHLIGHT_1 = """\
This is a plain text without any code blocks.

```python hl_lines="1"
print("Hello, World!")
```
"""

INPUT_HIGHLIGHT_2 = """\
This is a plain text without any code blocks.

```python
# highlight-next-line
print("Hello, World!")

x = 5

# highlight-next-line
print("Hello, World!")

```
"""

EXPECTED_HIGHLIGHT_2 = """\
This is a plain text without any code blocks.

```python hl_lines="1 5"
print("Hello, World!")

x = 5

print("Hello, World!")

```
"""


# Test end-to-end behavior of on_page_markdown
INPUT_HIGHLIGHT_3 = """\
```python exec="on" source="below"
print("Hello, World!")
# highlight-next-line
print("Hello, World!")
```
"""

EXPECTED_HIGHLIGHT_3 = """\
```python exec="on" source="below" hl_lines="2"
print("Hello, World!")
print("Hello, World!")
```
"""


def test_highlight_code_blocks() -> None:
    """Test that code blocks are highlighted correctly."""
    assert _highlight_code_blocks(INPUT_HIGHLIGHT_1) == EXPECTED_HIGHLIGHT_1
    assert _highlight_code_blocks(INPUT_HIGHLIGHT_2) == EXPECTED_HIGHLIGHT_2
    assert _highlight_code_blocks(INPUT_HIGHLIGHT_3) == EXPECTED_HIGHLIGHT_3


END_TO_END_INPUT_HIGHLIGHT_1 = """\
```python exec="on" source="below"
print("Hello, World!")
# highlight-next-line
print("Hello, World!")
```
"""


END_TO_END_INPUT_HIGHLIGHT_1_EXPECT = """\
```python exec="on" source="below" hl_lines="2" path="dummy.md"
print("Hello, World!")
print("Hello, World!")
```
"""


def test_on_page_markdown_highlights() -> None:
    """Test that on page markdown behaves correctly."""
    # Create a dummy MkDocs File and Page object.
    dummy_file = File("dummy.md", "dummy.md", "placeholder", use_directory_urls=False)
    dummy_page = Page("Test Page", dummy_file, config=MkDocsConfig())

    assert (
        on_page_markdown(END_TO_END_INPUT_HIGHLIGHT_1, dummy_page)
        == END_TO_END_INPUT_HIGHLIGHT_1_EXPECT
    )

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/tests/unit_tests/test_notebook_conversion.py
```py
import os
import tempfile

import nbformat
import pytest

from _scripts.notebook_convert import (
    _convert_links_in_markdown,
    _has_output,
    convert_notebook,
)


def test_has_output() -> None:
    """Test if a given code block is expected to have output."""
    assert _has_output("print('Hello, world!')") is True
    assert _has_output("print_stream(some_iterable)") is True
    assert _has_output("foo.y") is True
    assert _has_output("display(x)") is False
    assert _has_output("assert 1 == 1") is False
    assert _has_output("def foo(): pass") is False
    assert _has_output("import foobar") is False


@pytest.mark.parametrize(
    "source, expected",
    [
        (
            "This is a [link](https://example.com).",
            "This is a [link](https://example.com).",
        ),
        ("This is a [link](../foo).", "This is a [link](foo.md)."),
        ("This is a [link](../foo#hello).", "This is a [link](foo.md#hello)."),
        ("This is a [link](../foo/#hello).", "This is a [link](foo.md#hello)."),
    ],
)
def test_link_conversion(source: str, expected: str) -> None:
    """Test logic to convert links in markdown cells."""
    assert _convert_links_in_markdown(source) == expected


EXPECTED_OUTPUT = """\
```shell
pip install -U langgraph
```


```python
print('Hello')
```\

"""


def test_converting_cell_magic() -> None:
    """Test converting cell magic to code blocks."""
    with tempfile.TemporaryDirectory() as tmpdir:
        nb_path = os.path.join(tmpdir, "test_notebook.ipynb")

        # Create a minimal notebook object
        nb = nbformat.v4.new_notebook()
        nb.cells = [
            nbformat.v4.new_code_cell(
                "%%capture --no-stderr\n"
                "%pip install -U langgraph"
            ),
            nbformat.v4.new_code_cell("print('Hello')"),
        ]
        nb.metadata["language_info"] = {"name": "python"}

        # Write to file
        with open(nb_path, "w", encoding="utf-8") as f:
            nbformat.write(nb, f)

        # Run the conversion
        converted = convert_notebook(nb_path)
        assert converted == EXPECTED_OUTPUT

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/templates/python/material/function.html.jinja
```jinja
{#- Template for Python functions.
   
This template renders a Python function or method.

Context:
  function (griffe.Function): The function to render.
  root (bool): Whether this is the root object, injected with `:::` in a Markdown page.
  heading_level (int): The HTML heading level to use.
  config (dict): The configuration options.
-#}

{% block logs scoped %}
  {{ log.debug("Rendering " + function.path) }}
{% endblock logs %}

{% import "language"|get_template as lang with context %}
{#- Language module providing the `t` translation method. -#}

<div class="doc doc-object doc-function">
  {% with obj = function, html_id = function.path %}
    {% if root %}
      {% set show_full_path = config.show_root_full_path %}
      {% set root_members = True %}
    {% elif root_members %}
      {% set show_full_path = config.show_root_members_full_path or config.show_object_full_path %}
      {% set root_members = False %}
    {% else %}
      {% set show_full_path = config.show_object_full_path %}
    {% endif %}

    {% set function_name = function.path if show_full_path else function.name %}
    {#- Brief or full function name depending on configuration. -#}
    {% set symbol_type = "method" if function.parent.is_class else "function" %}
    {#- Symbol type: method when parent is a class, function otherwise. -#}

    {% if not root or config.show_root_heading %}
      {% filter heading(
          heading_level,
          role="function",
          id=html_id,
          class="doc doc-heading",
          toc_label=(('<code class="doc-symbol doc-symbol-toc doc-symbol-' + symbol_type + '"></code>&nbsp;')|safe if config.show_symbol_type_toc else '') + function.name,
        ) %}
      
        {% block heading scoped %}
          {% if config.show_symbol_type_heading %}<code class="doc-symbol doc-symbol-heading doc-symbol-{{ symbol_type }}"></code>{% endif %}
          {% if config.separate_signature %}
            <span class="doc doc-object-name doc-function-name">{{ config.heading if config.heading and root else function_name }}</span>
          {% else %}
            {%+ filter highlight(language="python", inline=True) %}
              {{ function_name }}{% include "signature"|get_template with context %}
            {% endfilter %}
          {% endif %}
        {% endblock heading %}

        {% block labels scoped %}
          {% with labels = function.labels %}
            {% include "labels"|get_template with context %}
          {% endwith %}
        {% endblock labels %}

      {% endfilter %}

      {% block signature scoped %}
        {#- Signature block.
        
        This block renders only the main signature and deliberately omits the overloads.
        -#}
