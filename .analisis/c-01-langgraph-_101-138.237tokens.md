                If pipeline mode is not supported, will fall back to using transaction context manager.
        """
        async with _ainternal.get_connection(self.conn) as conn:
            if self.pipe:
                # a connection in pipeline mode can be used concurrently
                # in multiple threads/coroutines, but only one cursor can be
                # used at a time
                try:
                    async with conn.cursor(binary=True, row_factory=dict_row) as cur:
                        yield cur
                finally:
                    if pipeline:
                        await self.pipe.sync()
            elif pipeline:
                # a connection not in pipeline mode can only be used by one
                # thread/coroutine at a time, so we acquire a lock
                if self.supports_pipeline:
                    async with (
                        self.lock,
                        conn.pipeline(),
                        conn.cursor(binary=True, row_factory=dict_row) as cur,
                    ):
                        yield cur
                else:
                    async with (
                        self.lock,
                        conn.transaction(),
                        conn.cursor(binary=True, row_factory=dict_row) as cur,
                    ):
                        yield cur
            else:
                async with (
                    self.lock,
                    conn.cursor(binary=True, row_factory=dict_row) as cur,
                ):
                    yield cur

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint-postgres/langgraph/store/postgres/base.py
```py
from __future__ import annotations

import asyncio
import concurrent.futures
import json
import logging
import threading
from collections import defaultdict
from collections.abc import Callable, Iterable, Iterator, Sequence
from contextlib import contextmanager
from datetime import datetime
from typing import (
    TYPE_CHECKING,
    Any,
    Generic,
    Literal,
    NamedTuple,
    TypeVar,
    cast,
)

import orjson
from langgraph.store.base import (
    BaseStore,
    GetOp,
    IndexConfig,
    Item,
    ListNamespacesOp,
    Op,
    PutOp,
    Result,
    SearchItem,
    SearchOp,
    TTLConfig,
    ensure_embeddings,
    get_text_at_path,
    tokenize_path,
)
from psycopg import Capabilities, Connection, Cursor, Pipeline
from psycopg.rows import DictRow, dict_row
from psycopg.types.json import Jsonb
from psycopg_pool import ConnectionPool
from typing_extensions import TypedDict

from langgraph.checkpoint.postgres import _ainternal as _ainternal
from langgraph.checkpoint.postgres import _internal as _pg_internal

if TYPE_CHECKING:
    from langchain_core.embeddings import Embeddings

logger = logging.getLogger(__name__)


class Migration(NamedTuple):
    """A database migration with optional conditions and parameters."""

    sql: str
    params: dict[str, Any] | None = None
    condition: Callable[[BasePostgresStore], bool] | None = None


MIGRATIONS: Sequence[str] = [
    """
CREATE TABLE IF NOT EXISTS store (
    -- 'prefix' represents the doc's 'namespace'
    prefix text NOT NULL,
    key text NOT NULL,
    value jsonb NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (prefix, key)
);
""",
    """
-- For faster lookups by prefix
CREATE INDEX CONCURRENTLY IF NOT EXISTS store_prefix_idx ON store USING btree (prefix text_pattern_ops);
""",
    """
-- Add expires_at column to store table
ALTER TABLE store
ADD COLUMN IF NOT EXISTS expires_at TIMESTAMP WITH TIME ZONE,
ADD COLUMN IF NOT EXISTS ttl_minutes INT;
""",
    """
-- Add indexes for efficient TTL sweeping
CREATE INDEX IF NOT EXISTS idx_store_expires_at ON store (expires_at)
WHERE expires_at IS NOT NULL;
""",
]

VECTOR_MIGRATIONS: Sequence[Migration] = [
    Migration(
        """
DO $$
BEGIN
    IF NOT EXISTS (SELECT 1 FROM pg_extension WHERE extname = 'vector') THEN
        CREATE EXTENSION vector;
    END IF;
END $$;
""",
    ),
    Migration(
        """
CREATE TABLE IF NOT EXISTS store_vectors (
    prefix text NOT NULL,
    key text NOT NULL,
    field_name text NOT NULL,
    embedding %(vector_type)s(%(dims)s),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (prefix, key, field_name),
    FOREIGN KEY (prefix, key) REFERENCES store(prefix, key) ON DELETE CASCADE
);
""",
        params={
            "dims": lambda store: store.index_config["dims"],
            "vector_type": lambda store: (
                cast(PostgresIndexConfig, store.index_config)
                .get("ann_index_config", {})
                .get("vector_type", "vector")
            ),
        },
    ),
    Migration(
        """
CREATE INDEX CONCURRENTLY IF NOT EXISTS store_vectors_embedding_idx ON store_vectors 
    USING %(index_type)s (embedding %(ops)s)%(index_params)s;
""",
        condition=lambda store: bool(
            store.index_config and _get_index_params(store)[0] != "flat"
        ),
        params={
            "index_type": lambda store: _get_index_params(store)[0],
            "ops": lambda store: _get_vector_type_ops(store),
            "index_params": lambda store: (
                " WITH ("
                + ", ".join(f"{k}={v}" for k, v in _get_index_params(store)[1].items())
                + ")"
                if _get_index_params(store)[1]
                else ""
            ),
        },
    ),
]


C = TypeVar("C", bound=_pg_internal.Conn | _ainternal.Conn)


class PoolConfig(TypedDict, total=False):
    """Connection pool settings for PostgreSQL connections.

    Controls connection lifecycle and resource utilization:
    - Small pools (1-5) suit low-concurrency workloads
    - Larger pools handle concurrent requests but consume more resources
    - Setting max_size prevents resource exhaustion under load
    """

    min_size: int
    """Minimum number of connections maintained in the pool. Defaults to 1."""

    max_size: int | None
    """Maximum number of connections allowed in the pool. None means unlimited."""

    kwargs: dict
    """Additional connection arguments passed to each connection in the pool.
    
    Default kwargs set automatically:
    - autocommit: True
    - prepare_threshold: 0
    - row_factory: dict_row
    """


class ANNIndexConfig(TypedDict, total=False):
    """Configuration for vector index in PostgreSQL store."""

    kind: Literal["hnsw", "ivfflat", "flat"]
    """Type of index to use: 'hnsw' for Hierarchical Navigable Small World, or 'ivfflat' for Inverted File Flat."""
    vector_type: Literal["vector", "halfvec"]
    """Type of vector storage to use.
    Options:
    - 'vector': Regular vectors (default)
    - 'halfvec': Half-precision vectors for reduced memory usage
    """


class HNSWConfig(ANNIndexConfig, total=False):
    """Configuration for HNSW (Hierarchical Navigable Small World) index."""

    kind: Literal["hnsw"]  # type: ignore[misc]
    m: int
    """Maximum number of connections per layer. Default is 16."""
    ef_construction: int
    """Size of dynamic candidate list for index construction. Default is 64."""


class IVFFlatConfig(ANNIndexConfig, total=False):
    """IVFFlat index divides vectors into lists, and then searches a subset of those lists that are closest to the query vector. It has faster build times and uses less memory than HNSW, but has lower query performance (in terms of speed-recall tradeoff).

    Three keys to achieving good recall are:
    1. Create the index after the table has some data
    2. Choose an appropriate number of lists - a good place to start is rows / 1000 for up to 1M rows and sqrt(rows) for over 1M rows
    3. When querying, specify an appropriate number of probes (higher is better for recall, lower is better for speed) - a good place to start is sqrt(lists)
    """

    kind: Literal["ivfflat"]  # type: ignore[misc]
    nlist: int
    """Number of inverted lists (clusters) for IVF index.
    
    Determines the number of clusters used in the index structure.
    Higher values can improve search speed but increase index size and build time.
    Typically set to the square root of the number of vectors in the index.
    """


class PostgresIndexConfig(IndexConfig, total=False):
    """Configuration for vector embeddings in PostgreSQL store with pgvector-specific options.

    Extends EmbeddingConfig with additional configuration for pgvector index and vector types.
    """

    ann_index_config: ANNIndexConfig
    """Specific configuration for the chosen index type (HNSW or IVF Flat)."""
    distance_type: Literal["l2", "inner_product", "cosine"]
    """Distance metric to use for vector similarity search:
    - 'l2': Euclidean distance
    - 'inner_product': Dot product
    - 'cosine': Cosine similarity
    """


class BasePostgresStore(Generic[C]):
    MIGRATIONS = MIGRATIONS
    VECTOR_MIGRATIONS = VECTOR_MIGRATIONS
    conn: C
    _deserializer: Callable[[bytes | orjson.Fragment], dict[str, Any]] | None
    index_config: PostgresIndexConfig | None

    def _get_batch_GET_ops_queries(
        self,
        get_ops: Sequence[tuple[int, GetOp]],
    ) -> list[tuple[str, tuple, tuple[str, ...], list]]:
        """
        Build queries to fetch (and optionally refresh the TTL of) multiple keys per namespace.

        Each returned element is a tuple of:
        (sql_query_string, sql_params, namespace, items_for_this_namespace)

        where items_for_this_namespace is the original list of (idx, key, refresh_ttl).
        """

        namespace_groups = defaultdict(list)
        refresh_ttls = defaultdict(list)
        for idx, op in get_ops:
            namespace_groups[op.namespace].append((idx, op.key))
            refresh_ttls[op.namespace].append(op.refresh_ttl)

        results = []
        for namespace, items in namespace_groups.items():
            _, keys = zip(*items, strict=False)
            this_refresh_ttls = refresh_ttls[namespace]

            query = """
                WITH passed_in AS (
                    SELECT unnest(%s::text[]) AS key,
                        unnest(%s::bool[])  AS do_refresh
                ),
                updated AS (
                    UPDATE store s
                    SET expires_at = NOW() + (s.ttl_minutes || ' minutes')::interval
                    FROM passed_in p
                    WHERE s.prefix = %s
                    AND s.key    = p.key
                    AND p.do_refresh = TRUE
                    AND s.ttl_minutes IS NOT NULL
                    RETURNING s.key
                )
                SELECT s.key, s.value, s.created_at, s.updated_at
                FROM store s
                JOIN passed_in p ON s.key = p.key
                WHERE s.prefix = %s
            """
            ns_text = _namespace_to_text(namespace)
            params = (
                list(keys),  # -> unnest(%s::text[])
                list(this_refresh_ttls),  # -> unnest(%s::bool[])
                ns_text,  # -> prefix = %s (for UPDATE)
                ns_text,  # -> prefix = %s (for final SELECT)
            )
            results.append((query, params, namespace, items))

        return results

    def _prepare_batch_PUT_queries(
        self,
        put_ops: Sequence[tuple[int, PutOp]],
    ) -> tuple[
        list[tuple[str, Sequence]],
        tuple[str, Sequence[tuple[str, str, str, str]]] | None,
    ]:
        dedupped_ops: dict[tuple[tuple[str, ...], str], PutOp] = {}
        for _, op in put_ops:
            dedupped_ops[(op.namespace, op.key)] = op

        inserts: list[PutOp] = []
        deletes: list[PutOp] = []
        for op in dedupped_ops.values():
            if op.value is None:
                deletes.append(op)
            else:
                inserts.append(op)

        queries: list[tuple[str, Sequence]] = []

        if deletes:
            namespace_groups: dict[tuple[str, ...], list[str]] = defaultdict(list)
            for op in deletes:
                namespace_groups[op.namespace].append(op.key)
            for namespace, keys in namespace_groups.items():
                placeholders = ",".join(["%s"] * len(keys))
                query = (
                    f"DELETE FROM store WHERE prefix = %s AND key IN ({placeholders})"
                )
                params = (_namespace_to_text(namespace), *keys)
                queries.append((query, params))
        embedding_request: tuple[str, Sequence[tuple[str, str, str, str]]] | None = None
        if inserts:
            values = []
            insertion_params = []
            vector_values = []
            embedding_request_params = []
            # Handle TTL expiration

            # First handle main store insertions
            for op in inserts:
                if op.ttl is not None:
                    expires_at_str = f"NOW() + INTERVAL '{op.ttl * 60} seconds'"
                    ttl_minutes = op.ttl
                else:
                    expires_at_str = "NULL"
                    ttl_minutes = None

                values.append(
                    f"(%s, %s, %s, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, {expires_at_str}, %s)"
                )
                insertion_params.extend(
                    [
                        _namespace_to_text(op.namespace),
                        op.key,
                        Jsonb(cast(dict, op.value)),
                        ttl_minutes,
                    ]
                )

            # Then handle embeddings if configured
            if self.index_config:
                for op in inserts:
                    if op.index is False:
                        continue
                    value = op.value
                    ns = _namespace_to_text(op.namespace)
                    k = op.key

                    if op.index is None:
                        paths = cast(dict, self.index_config)["__tokenized_fields"]
                    else:
                        paths = [(ix, tokenize_path(ix)) for ix in op.index]

                    for path, tokenized_path in paths:
                        texts = get_text_at_path(value, tokenized_path)
                        for i, text in enumerate(texts):
                            pathname = f"{path}.{i}" if len(texts) > 1 else path
                            vector_values.append(
                                "(%s, %s, %s, %s, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP)"
                            )
                            embedding_request_params.append((ns, k, pathname, text))

            values_str = ",".join(values)
            query = f"""
                INSERT INTO store (prefix, key, value, created_at, updated_at, expires_at, ttl_minutes)
                VALUES {values_str}
                ON CONFLICT (prefix, key) DO UPDATE
                SET value = EXCLUDED.value,
                    updated_at = CURRENT_TIMESTAMP,
                    expires_at = EXCLUDED.expires_at,
                    ttl_minutes = EXCLUDED.ttl_minutes
            """
            queries.append((query, insertion_params))

            if vector_values:
                values_str = ",".join(vector_values)
                query = f"""
                    INSERT INTO store_vectors (prefix, key, field_name, embedding, created_at, updated_at)
                    VALUES {values_str}
                    ON CONFLICT (prefix, key, field_name) DO UPDATE
                    SET embedding = EXCLUDED.embedding,
                        updated_at = CURRENT_TIMESTAMP
                """
                embedding_request = (query, embedding_request_params)

        return queries, embedding_request

    def _prepare_batch_search_queries(
        self,
        search_ops: Sequence[tuple[int, SearchOp]],
    ) -> tuple[
        list[tuple[str, list[None | str | list[float]]]],  # queries, params
        list[tuple[int, str]],  # idx, query_text pairs to embed
    ]:
        """
        Build per-SearchOp SQL queries (with optional TTL refresh) plus embedding requests.
        Returns:
        - queries: list of (SQL, param_list)
        - embedding_requests: list of (original_index_in_search_ops, text_query)
        """

        queries = []
        embedding_requests = []
        for idx, (_, op) in enumerate(search_ops):
            filter_params = []
            filter_clauses = []
            if op.filter:
                for key, value in op.filter.items():
                    if isinstance(value, dict):
                        for op_name, val in value.items():
                            condition, params_ = self._get_filter_condition(
                                key, op_name, val
                            )
                            filter_clauses.append(condition)
                            filter_params.extend(params_)
                    else:
                        filter_clauses.append("value->%s = %s::jsonb")
                        filter_params.extend([key, orjson.dumps(value).decode("utf-8")])

            ns_condition = "TRUE"
            ns_param: Sequence[str] | None = None
            if op.namespace_prefix:
                ns_condition = "store.prefix LIKE %s"
                ns_param = (f"{_namespace_to_text(op.namespace_prefix)}%",)
            else:
                ns_param = ()

            extra_filters = (
                " AND " + " AND ".join(filter_clauses) if filter_clauses else ""
            )

            if op.query and self.index_config:
                # We'll embed the text later, so record the request.
                embedding_requests.append((idx, op.query))

                score_operator, post_operator = get_distance_operator(self)
                post_operator = post_operator.replace("scored", "uniq")
                vector_type = self.index_config.get("ann_index_config", {}).get(
                    "vector_type", "vector"
                )

                # For hamming bit vectors, or “regular” vectors
                if (
                    vector_type == "bit"
                    and cast(dict, self.index_config).get("distance_type") == "hamming"
                ):
                    score_operator = score_operator % (
                        "%s",
                        cast(dict, self.index_config)["dims"],
                    )
                else:
                    score_operator = score_operator % ("%s", vector_type)

                vectors_per_doc_estimate = cast(dict, self.index_config)[
                    "__estimated_num_vectors"
                ]
                expanded_limit = (op.limit * vectors_per_doc_estimate * 2) + 1

                # “sub_scored” does the main vector search
                # Then we do DISTINCT ON to drop duplicates if your store can have them
                # Finally we limit & offset
                vector_search_cte = f"""
                        SELECT store.prefix, store.key, store.value, store.created_at, store.updated_at,
                            {score_operator} AS neg_score
                        FROM store
                        JOIN store_vectors sv ON store.prefix = sv.prefix AND store.key = sv.key
                        WHERE {ns_condition} {extra_filters}
                        ORDER BY {score_operator} ASC
                        LIMIT %s
                    """

                search_results_sql = f"""
                        WITH scored AS (
                            {vector_search_cte}
                        )
                        SELECT uniq.prefix, uniq.key, uniq.value, uniq.created_at, uniq.updated_at,
                            {post_operator} AS score
                        FROM (
                            SELECT DISTINCT ON (scored.prefix, scored.key)
                                scored.prefix, scored.key, scored.value, scored.created_at, scored.updated_at, scored.neg_score
                            FROM scored
                            ORDER BY scored.prefix, scored.key, scored.neg_score ASC
                        ) uniq
                        ORDER BY score DESC
                        LIMIT %s
                        OFFSET %s
                    """

                search_results_params = [
                    PLACEHOLDER,
                    *ns_param,
                    *filter_params,
                    PLACEHOLDER,
                    expanded_limit,
                    op.limit,
                    op.offset,
                ]

            else:
                base_query = f"""
                        SELECT store.prefix, store.key, store.value, store.created_at, store.updated_at, NULL AS score
                        FROM store
                        WHERE {ns_condition} {extra_filters}
                        ORDER BY store.updated_at DESC
                        LIMIT %s
                        OFFSET %s
                    """
                search_results_sql = base_query
                search_results_params = [
                    *ns_param,
                    *filter_params,
                    op.limit,
                    op.offset,
                ]

            if op.refresh_ttl:
                # Wrap entire primary query in a CTE, then perform "update_at"
                final_sql = f"""
                        WITH search_results AS (
                            {search_results_sql}
                        ),
                        updated AS (
                            UPDATE store s
                            SET expires_at = NOW() + (s.ttl_minutes || ' minutes')::interval
                            FROM search_results sr
                            WHERE s.prefix = sr.prefix
                            AND s.key = sr.key
                            AND s.ttl_minutes IS NOT NULL
                        )
                        SELECT sr.prefix, sr.key, sr.value, sr.created_at, sr.updated_at, sr.score
                        FROM search_results sr
                    """
                final_params = search_results_params[:]  # copy
            else:
                final_sql = search_results_sql
                final_params = search_results_params
            queries.append((final_sql, final_params))

        return queries, embedding_requests

    def _get_batch_list_namespaces_queries(
        self,
        list_ops: Sequence[tuple[int, ListNamespacesOp]],
    ) -> list[tuple[str, Sequence]]:
        queries: list[tuple[str, Sequence]] = []
        for _, op in list_ops:
            query = r"""
                SELECT DISTINCT ON (truncated_prefix) truncated_prefix, prefix
                FROM (
                    SELECT
                        prefix,
                        CASE
                            WHEN %s::integer IS NOT NULL THEN
                                (SELECT STRING_AGG(part, '.' ORDER BY idx)
                                 FROM (
                                     SELECT part, ROW_NUMBER() OVER () AS idx
                                     FROM UNNEST(REGEXP_SPLIT_TO_ARRAY(prefix, '\.')) AS part
                                     LIMIT %s::integer
                                 ) subquery
                                )
                            ELSE prefix
                        END AS truncated_prefix
                    FROM store
            """
            params: list[Any] = [op.max_depth, op.max_depth]

            conditions = []
            if op.match_conditions:
                for condition in op.match_conditions:
                    if condition.match_type == "prefix":
                        conditions.append("prefix LIKE %s")
                        params.append(
                            f"{_namespace_to_text(condition.path, handle_wildcards=True)}%"
                        )
                    elif condition.match_type == "suffix":
                        conditions.append("prefix LIKE %s")
                        params.append(
                            f"%{_namespace_to_text(condition.path, handle_wildcards=True)}"
                        )
                    else:
                        logger.warning(
                            f"Unknown match_type in list_namespaces: {condition.match_type}"
                        )

            if conditions:
                query += " WHERE " + " AND ".join(conditions)
            query += ") AS subquery "

            query += " ORDER BY truncated_prefix LIMIT %s OFFSET %s"
            params.extend([op.limit, op.offset])
            queries.append((query, tuple(params)))

        return queries

    def _get_filter_condition(self, key: str, op: str, value: Any) -> tuple[str, list]:
        """Helper to generate filter conditions."""
        if op == "$eq":
            return "value->%s = %s::jsonb", [key, json.dumps(value)]
        elif op == "$gt":
            return "value->>%s > %s", [key, str(value)]
        elif op == "$gte":
            return "value->>%s >= %s", [key, str(value)]
        elif op == "$lt":
            return "value->>%s < %s", [key, str(value)]
        elif op == "$lte":
            return "value->>%s <= %s", [key, str(value)]
        elif op == "$ne":
            return "value->%s != %s::jsonb", [key, json.dumps(value)]
        else:
            raise ValueError(f"Unsupported operator: {op}")


class PostgresStore(BaseStore, BasePostgresStore[_pg_internal.Conn]):
    """Postgres-backed store with optional vector search using pgvector.

    !!! example "Examples"
        Basic setup and usage:
        ```python
        from langgraph.store.postgres import PostgresStore
        from psycopg import Connection

        conn_string = "postgresql://user:pass@localhost:5432/dbname"

        # Using direct connection
        with Connection.connect(conn_string) as conn:
            store = PostgresStore(conn)
            store.setup() # Run migrations. Done once

            # Store and retrieve data
            store.put(("users", "123"), "prefs", {"theme": "dark"})
            item = store.get(("users", "123"), "prefs")
        ```

        Or using the convenient from_conn_string helper:
        ```python
        from langgraph.store.postgres import PostgresStore

        conn_string = "postgresql://user:pass@localhost:5432/dbname"

        with PostgresStore.from_conn_string(conn_string) as store:
            store.setup()

            # Store and retrieve data
            store.put(("users", "123"), "prefs", {"theme": "dark"})
            item = store.get(("users", "123"), "prefs")
        ```

        Vector search using LangChain embeddings:
        ```python
        from langchain.embeddings import init_embeddings
        from langgraph.store.postgres import PostgresStore

        conn_string = "postgresql://user:pass@localhost:5432/dbname"

        with PostgresStore.from_conn_string(
            conn_string,
            index={
                "dims": 1536,
                "embed": init_embeddings("openai:text-embedding-3-small"),
                "fields": ["text"]  # specify which fields to embed. Default is the whole serialized value
            }
        ) as store:
            store.setup() # Do this once to run migrations

            # Store documents
            store.put(("docs",), "doc1", {"text": "Python tutorial"})
            store.put(("docs",), "doc2", {"text": "TypeScript guide"})
            store.put(("docs",), "doc2", {"text": "Other guide"}, index=False) # don't index

            # Search by similarity
            results = store.search(("docs",), query="programming guides", limit=2)
        ```

    Note:
        Semantic search is disabled by default. You can enable it by providing an `index` configuration
        when creating the store. Without this configuration, all `index` arguments passed to
        `put` or `aput`will have no effect.

    Warning:
        Make sure to call `setup()` before first use to create necessary tables and indexes.
        The pgvector extension must be available to use vector search.

    Note:
        If you provide a TTL configuration, you must explicitly call `start_ttl_sweeper()` to begin
        the background thread that removes expired items. Call `stop_ttl_sweeper()` to properly
        clean up resources when you're done with the store.

    """

    __slots__ = (
        "_deserializer",
        "pipe",
        "lock",
        "supports_pipeline",
        "index_config",
        "embeddings",
        "_ttl_sweeper_thread",
        "_ttl_stop_event",
    )
    supports_ttl: bool = True

    def __init__(
        self,
        conn: _pg_internal.Conn,
        *,
        pipe: Pipeline | None = None,
        deserializer: Callable[[bytes | orjson.Fragment], dict[str, Any]] | None = None,
        index: PostgresIndexConfig | None = None,
        ttl: TTLConfig | None = None,
    ) -> None:
        super().__init__()
        self._deserializer = deserializer
        self.conn = conn
        self.pipe = pipe
        self.supports_pipeline = Capabilities().has_pipeline()
        self.lock = threading.Lock()
        self.index_config = index
        if self.index_config:
            self.embeddings, self.index_config = _ensure_index_config(self.index_config)
        else:
            self.embeddings = None
        self.ttl_config = ttl
        self._ttl_sweeper_thread: threading.Thread | None = None
        self._ttl_stop_event = threading.Event()

    @classmethod
    @contextmanager
    def from_conn_string(
        cls,
        conn_string: str,
        *,
        pipeline: bool = False,
        pool_config: PoolConfig | None = None,
        index: PostgresIndexConfig | None = None,
        ttl: TTLConfig | None = None,
    ) -> Iterator[PostgresStore]:
        """Create a new PostgresStore instance from a connection string.

        Args:
            conn_string: The Postgres connection info string.
            pipeline: whether to use Pipeline
            pool_config: Configuration for the connection pool.
                If provided, will create a connection pool and use it instead of a single connection.
                This overrides the `pipeline` argument.
            index: The index configuration for the store.
            ttl: The TTL configuration for the store.

        Returns:
            PostgresStore: A new PostgresStore instance.
        """
        if pool_config is not None:
            pc = pool_config.copy()
            with cast(
                ConnectionPool[Connection[DictRow]],
                ConnectionPool(
                    conn_string,
                    min_size=pc.pop("min_size", 1),
                    max_size=pc.pop("max_size", None),
                    kwargs={
                        "autocommit": True,
                        "prepare_threshold": 0,
                        "row_factory": dict_row,
                        **(pc.pop("kwargs", None) or {}),
                    },
                    **cast(dict, pc),
                ),
            ) as pool:
                yield cls(conn=pool, index=index, ttl=ttl)
        else:
            with Connection.connect(
                conn_string, autocommit=True, prepare_threshold=0, row_factory=dict_row
            ) as conn:
                if pipeline:
                    with conn.pipeline() as pipe:
                        yield cls(conn, pipe=pipe, index=index, ttl=ttl)
                else:
                    yield cls(conn, index=index, ttl=ttl)

    def sweep_ttl(self) -> int:
        """Delete expired store items based on TTL.

        Returns:
            int: The number of deleted items.
        """
        with self._cursor() as cur:
            cur.execute(
                """
                DELETE FROM store
                WHERE expires_at IS NOT NULL AND expires_at < NOW()
                """
            )
            deleted_count = cur.rowcount
            return deleted_count

    def start_ttl_sweeper(
        self, sweep_interval_minutes: int | None = None
    ) -> concurrent.futures.Future[None]:
        """Periodically delete expired store items based on TTL.

        Returns:
            Future that can be waited on or cancelled.
        """
        if not self.ttl_config:
            future: concurrent.futures.Future[None] = concurrent.futures.Future()
            future.set_result(None)
            return future

        if self._ttl_sweeper_thread and self._ttl_sweeper_thread.is_alive():
            logger.info("TTL sweeper thread is already running")
            # Return a future that can be used to cancel the existing thread
            future = concurrent.futures.Future()
            future.add_done_callback(
                lambda f: self._ttl_stop_event.set() if f.cancelled() else None
            )
            return future

        self._ttl_stop_event.clear()

        interval = float(
            sweep_interval_minutes or self.ttl_config.get("sweep_interval_minutes") or 5
        )
        logger.info(f"Starting store TTL sweeper with interval {interval} minutes")

        future = concurrent.futures.Future()

        def _sweep_loop() -> None:
            try:
                while not self._ttl_stop_event.is_set():
                    if self._ttl_stop_event.wait(interval * 60):
                        break

                    try:
                        expired_items = self.sweep_ttl()
                        if expired_items > 0:
                            logger.info(f"Store swept {expired_items} expired items")
                    except Exception as exc:
                        logger.exception(
                            "Store TTL sweep iteration failed", exc_info=exc
                        )
                future.set_result(None)
            except Exception as exc:
                future.set_exception(exc)

        thread = threading.Thread(target=_sweep_loop, daemon=True, name="ttl-sweeper")
        self._ttl_sweeper_thread = thread
        thread.start()

        future.add_done_callback(
            lambda f: self._ttl_stop_event.set() if f.cancelled() else None
        )
        return future

    def stop_ttl_sweeper(self, timeout: float | None = None) -> bool:
        """Stop the TTL sweeper thread if it's running.

        Args:
            timeout: Maximum time to wait for the thread to stop, in seconds.
                If `None`, wait indefinitely.

        Returns:
            bool: True if the thread was successfully stopped or wasn't running,
                False if the timeout was reached before the thread stopped.
        """
        if not self._ttl_sweeper_thread or not self._ttl_sweeper_thread.is_alive():
            return True

        logger.info("Stopping TTL sweeper thread")
        self._ttl_stop_event.set()

        self._ttl_sweeper_thread.join(timeout)
        success = not self._ttl_sweeper_thread.is_alive()

        if success:
            self._ttl_sweeper_thread = None
            logger.info("TTL sweeper thread stopped")
        else:
            logger.warning("Timed out waiting for TTL sweeper thread to stop")

        return success

    def __del__(self) -> None:
        """Ensure the TTL sweeper thread is stopped when the object is garbage collected."""
        if hasattr(self, "_ttl_stop_event") and hasattr(self, "_ttl_sweeper_thread"):
            self.stop_ttl_sweeper(timeout=0.1)

    @contextmanager
    def _cursor(self, *, pipeline: bool = False) -> Iterator[Cursor[DictRow]]:
        """Create a database cursor as a context manager.

        Args:
            pipeline: whether to use pipeline for the DB operations inside the context manager.
                Will be applied regardless of whether the PostgresStore instance was initialized with a pipeline.
                If pipeline mode is not supported, will fall back to using transaction context manager.
        """
        with _pg_internal.get_connection(self.conn) as conn:
            if self.pipe:
                # a connection in pipeline mode can be used concurrently
                # in multiple threads/coroutines, but only one cursor can be
                # used at a time
                try:
                    with conn.cursor(binary=True, row_factory=dict_row) as cur:
                        yield cur
                finally:
                    if pipeline:
                        self.pipe.sync()
            elif pipeline:
                # a connection not in pipeline mode can only be used by one
                # thread/coroutine at a time, so we acquire a lock
                if self.supports_pipeline:
                    with (
                        self.lock,
                        conn.pipeline(),
                        conn.cursor(binary=True, row_factory=dict_row) as cur,
                    ):
                        yield cur
                else:
                    with (
                        self.lock,
                        conn.transaction(),
                        conn.cursor(binary=True, row_factory=dict_row) as cur,
                    ):
                        yield cur
            else:
                with conn.cursor(binary=True, row_factory=dict_row) as cur:
                    yield cur

    def batch(self, ops: Iterable[Op]) -> list[Result]:
        grouped_ops, num_ops = _group_ops(ops)
        results: list[Result] = [None] * num_ops

        with self._cursor(pipeline=True) as cur:
            if GetOp in grouped_ops:
                self._batch_get_ops(
                    cast(Sequence[tuple[int, GetOp]], grouped_ops[GetOp]), results, cur
                )

            if SearchOp in grouped_ops:
                self._batch_search_ops(
                    cast(Sequence[tuple[int, SearchOp]], grouped_ops[SearchOp]),
                    results,
                    cur,
                )

            if ListNamespacesOp in grouped_ops:
                self._batch_list_namespaces_ops(
                    cast(
                        Sequence[tuple[int, ListNamespacesOp]],
                        grouped_ops[ListNamespacesOp],
                    ),
                    results,
                    cur,
                )
            if PutOp in grouped_ops:
                self._batch_put_ops(
                    cast(Sequence[tuple[int, PutOp]], grouped_ops[PutOp]), cur
                )

        return results

    def _batch_get_ops(
        self,
        get_ops: Sequence[tuple[int, GetOp]],
        results: list[Result],
        cur: Cursor[DictRow],
    ) -> None:
        for query, params, namespace, items in self._get_batch_GET_ops_queries(get_ops):
            cur.execute(query, params)
            rows = cast(list[Row], cur.fetchall())
            key_to_row = {row["key"]: row for row in rows}
            for idx, key in items:
                row = key_to_row.get(key)
                if row:
                    results[idx] = _row_to_item(
                        namespace, row, loader=self._deserializer
                    )
                else:
                    results[idx] = None

    def _batch_put_ops(
        self,
        put_ops: Sequence[tuple[int, PutOp]],
        cur: Cursor[DictRow],
    ) -> None:
        queries, embedding_request = self._prepare_batch_PUT_queries(put_ops)
        if embedding_request:
            if self.embeddings is None:
                # Should not get here since the embedding config is required
                # to return an embedding_request above
                raise ValueError(
                    "Embedding configuration is required for vector operations "
                    f"(for semantic search). "
                    f"Please provide an Embeddings when initializing the {self.__class__.__name__}."
                )
            query, txt_params = embedding_request
            # Update the params to replace the raw text with the vectors
            vectors = self.embeddings.embed_documents(
                [param[-1] for param in txt_params]
            )
            queries.append(
                (
                    query,
                    [
                        p
                        for (ns, k, pathname, _), vector in zip(
                            txt_params, vectors, strict=False
                        )
                        for p in (ns, k, pathname, vector)
                    ],
                )
            )

        for query, params in queries:
            cur.execute(query, params)

    def _batch_search_ops(
        self,
        search_ops: Sequence[tuple[int, SearchOp]],
        results: list[Result],
        cur: Cursor[DictRow],
    ) -> None:
        queries, embedding_requests = self._prepare_batch_search_queries(search_ops)

        if embedding_requests and self.embeddings:
            embeddings = self.embeddings.embed_documents(
                [query for _, query in embedding_requests]
            )
            for (idx, _), embedding in zip(
                embedding_requests, embeddings, strict=False
            ):
                _paramslist = queries[idx][1]
                for i in range(len(_paramslist)):
                    if _paramslist[i] is PLACEHOLDER:
                        _paramslist[i] = embedding

        for (idx, _), (query, params) in zip(search_ops, queries, strict=False):
            cur.execute(query, params)
            rows = cast(list[Row], cur.fetchall())
            results[idx] = [
                _row_to_search_item(
                    _decode_ns_bytes(row["prefix"]), row, loader=self._deserializer
                )
                for row in rows
            ]

    def _batch_list_namespaces_ops(
        self,
        list_ops: Sequence[tuple[int, ListNamespacesOp]],
        results: list[Result],
        cur: Cursor[DictRow],
    ) -> None:
        for (query, params), (idx, _) in zip(
            self._get_batch_list_namespaces_queries(list_ops), list_ops, strict=False
        ):
            cur.execute(query, params)
            results[idx] = [_decode_ns_bytes(row["truncated_prefix"]) for row in cur]

    async def abatch(self, ops: Iterable[Op]) -> list[Result]:
        return await asyncio.get_running_loop().run_in_executor(None, self.batch, ops)

    def setup(self) -> None:
        """Set up the store database.

        This method creates the necessary tables in the Postgres database if they don't
        already exist and runs database migrations. It MUST be called directly by the user
        the first time the store is used.
        """

        def _get_version(cur: Cursor[dict[str, Any]], table: str) -> int:
            cur.execute(
                f"""
                CREATE TABLE IF NOT EXISTS {table} (
                    v INTEGER PRIMARY KEY
                )
            """
            )
            cur.execute(f"SELECT v FROM {table} ORDER BY v DESC LIMIT 1")
            row = cast(dict, cur.fetchone())
            if row is None:
                version = -1
            else:
                version = row["v"]
            return version

        with self._cursor() as cur:
            version = _get_version(cur, table="store_migrations")
            for v, sql in enumerate(self.MIGRATIONS[version + 1 :], start=version + 1):
                try:
                    cur.execute(sql)
                    cur.execute("INSERT INTO store_migrations (v) VALUES (%s)", (v,))
                except Exception as e:
                    logger.error(
                        f"Failed to apply migration {v}.\nSql={sql}\nError={e}"
                    )
                    raise

            if self.index_config:
                version = _get_version(cur, table="vector_migrations")
                for v, migration in enumerate(
                    self.VECTOR_MIGRATIONS[version + 1 :], start=version + 1
                ):
                    if migration.condition and not migration.condition(self):
                        continue
                    sql = migration.sql
                    if migration.params:
                        params = {
                            k: v(self) if v is not None and callable(v) else v
                            for k, v in migration.params.items()
                        }
                        sql = sql % params
                    cur.execute(sql)
                    cur.execute("INSERT INTO vector_migrations (v) VALUES (%s)", (v,))


class Row(TypedDict):
    key: str
    value: Any
    prefix: str
    created_at: datetime
    updated_at: datetime


# Private utilities

_DEFAULT_ANN_CONFIG = ANNIndexConfig(
    vector_type="vector",
)


def _get_vector_type_ops(store: BasePostgresStore) -> str:
    """Get the vector type operator class based on config."""
    if not store.index_config:
        return "vector_cosine_ops"

    config = store.index_config
    index_config = config.get("ann_index_config", _DEFAULT_ANN_CONFIG).copy()
    vector_type = cast(str, index_config.get("vector_type", "vector"))
    if vector_type not in ("vector", "halfvec"):
        raise ValueError(
            f"Vector type must be 'vector' or 'halfvec', got {vector_type}"
        )

    distance_type = config.get("distance_type", "cosine")

    # For regular vectors
    type_prefix = {"vector": "vector", "halfvec": "halfvec"}[vector_type]

    if distance_type not in ("l2", "inner_product", "cosine"):
        raise ValueError(
            f"Vector type {vector_type} only supports 'l2', 'inner_product', or 'cosine' distance, got {distance_type}"
        )

    distance_suffix = {
        "l2": "l2_ops",
        "inner_product": "ip_ops",
        "cosine": "cosine_ops",
    }[distance_type]

    return f"{type_prefix}_{distance_suffix}"


def _get_index_params(store: Any) -> tuple[str, dict[str, Any]]:
    """Get the index type and configuration based on config."""
    if not store.index_config:
        return "hnsw", {}

    config = cast(PostgresIndexConfig, store.index_config)
    index_config = config.get("ann_index_config", _DEFAULT_ANN_CONFIG).copy()
    kind = index_config.pop("kind", "hnsw")
    index_config.pop("vector_type", None)
    return kind, index_config


def _namespace_to_text(
    namespace: tuple[str, ...], handle_wildcards: bool = False
) -> str:
    """Convert namespace tuple to text string."""
    if handle_wildcards:
        namespace = tuple("%" if val == "*" else val for val in namespace)
    return ".".join(namespace)


def _row_to_item(
    namespace: tuple[str, ...],
    row: Row,
    *,
    loader: Callable[[bytes | orjson.Fragment], dict[str, Any]] | None = None,
) -> Item:
    """Convert a row from the database into an Item.

    Args:
        namespace: Item namespace
        row: Database row
        loader: Optional value loader for non-dict values
    """
    val = row["value"]
    if not isinstance(val, dict):
        val = (loader or _json_loads)(val)

    kwargs = {
        "key": row["key"],
        "namespace": namespace,
        "value": val,
        "created_at": row["created_at"],
        "updated_at": row["updated_at"],
    }

    return Item(**kwargs)


def _row_to_search_item(
    namespace: tuple[str, ...],
    row: Row,
    *,
    loader: Callable[[bytes | orjson.Fragment], dict[str, Any]] | None = None,
) -> SearchItem:
    """Convert a row from the database into an Item."""
    loader = loader or _json_loads
    val = row["value"]
    score = row.get("score")
    if score is not None:
        try:
            score = float(score)  # type: ignore[arg-type]
        except ValueError:
            logger.warning("Invalid score: %s", score)
            score = None
    return SearchItem(
        value=val if isinstance(val, dict) else loader(val),
        key=row["key"],
        namespace=namespace,
        created_at=row["created_at"],
        updated_at=row["updated_at"],
        score=score,
    )


def _group_ops(ops: Iterable[Op]) -> tuple[dict[type, list[tuple[int, Op]]], int]:
    grouped_ops: dict[type, list[tuple[int, Op]]] = defaultdict(list)
    tot = 0
    for idx, op in enumerate(ops):
        grouped_ops[type(op)].append((idx, op))
        tot += 1
    return grouped_ops, tot


def _json_loads(content: bytes | orjson.Fragment) -> Any:
    if isinstance(content, orjson.Fragment):
        if hasattr(content, "buf"):
            content = content.buf
        else:
            if isinstance(content.contents, bytes):
                content = content.contents
            else:
                content = content.contents.encode()
    return orjson.loads(cast(bytes, content))


def _decode_ns_bytes(namespace: str | bytes | list) -> tuple[str, ...]:
    if isinstance(namespace, list):
        return tuple(namespace)
    if isinstance(namespace, bytes):
        namespace = namespace.decode()[1:]
    return tuple(namespace.split("."))


def get_distance_operator(store: Any) -> tuple[str, str]:
    """Get the distance operator and score expression based on config."""
    # Note: Today, we are not using ANN indices due to restrictions
    # on PGVector's support for mixing vector and non-vector filters
    # To use the index, PGVector expects:
    #  - ORDER BY the operator NOT an expression (even negation blocks it)
    #  - ASCENDING order
    #  - Any WHERE clause should be over a partial index.
    # If we violate any of these, it will use a sequential scan
    # See https://github.com/pgvector/pgvector/issues/216 and the
    # pgvector documentation for more details.
    if not store.index_config:
        raise ValueError(
            "Embedding configuration is required for vector operations "
            f"(for semantic search). "
            f"Please provide an Embeddings when initializing the {store.__class__.__name__}."
        )

    config = cast(PostgresIndexConfig, store.index_config)
    distance_type = config.get("distance_type", "cosine")

    # Return the operator and the score expression
    # The operator is used in the CTE and will be compatible with an ASCENDING ORDER
    # sort clause.
    # The score expression is used in the final query and will be compatible with
    # a DESCENDING ORDER sort clause and the user's expectations of what the similarity score
    # should be.
    if distance_type == "l2":
        # Final: "-(sv.embedding <-> %s::%s)"
        # We return the "l2 similarity" so that the sorting order is the same
        return "sv.embedding <-> %s::%s", "-scored.neg_score"
    elif distance_type == "inner_product":
        # Final: "-(sv.embedding <#> %s::%s)"
        return "sv.embedding <#> %s::%s", "-(scored.neg_score)"
    else:  # cosine similarity
        # Final:  "1 - (sv.embedding <=> %s::%s)"
        return "sv.embedding <=> %s::%s", "1 - scored.neg_score"


def _ensure_index_config(
    index_config: PostgresIndexConfig,
) -> tuple[Embeddings | None, PostgresIndexConfig]:
    index_config = index_config.copy()
    tokenized: list[tuple[str, Literal["$"] | list[str]]] = []
    tot = 0
    fields = index_config.get("fields") or ["$"]
    if isinstance(fields, str):
        fields = [fields]
    if not isinstance(fields, list):
        raise ValueError(f"Text fields must be a list or a string. Got {fields}")
    for p in fields:
        if p == "$":
            tokenized.append((p, "$"))
            tot += 1
        else:
            toks = tokenize_path(p)
            tokenized.append((p, toks))
            tot += len(toks)
    index_config["__tokenized_fields"] = tokenized
    index_config["__estimated_num_vectors"] = tot
    embeddings = ensure_embeddings(
        index_config.get("embed"),
    )
    return embeddings, index_config


PLACEHOLDER = object()

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint-postgres/langgraph/store/postgres/py.typed
```typed

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint-postgres/langgraph/checkpoint/postgres/__init__.py
```py
from __future__ import annotations

import threading
from collections import defaultdict
from collections.abc import Iterator, Sequence
from contextlib import contextmanager
from typing import Any

from langchain_core.runnables import RunnableConfig
from langgraph.checkpoint.base import (
    WRITES_IDX_MAP,
    ChannelVersions,
    Checkpoint,
    CheckpointMetadata,
    CheckpointTuple,
    get_checkpoint_id,
    get_serializable_checkpoint_metadata,
)
from langgraph.checkpoint.serde.base import SerializerProtocol
from psycopg import Capabilities, Connection, Cursor, Pipeline
from psycopg.rows import DictRow, dict_row
from psycopg.types.json import Jsonb
from psycopg_pool import ConnectionPool

from langgraph.checkpoint.postgres import _internal
from langgraph.checkpoint.postgres.base import BasePostgresSaver
from langgraph.checkpoint.postgres.shallow import ShallowPostgresSaver

Conn = _internal.Conn  # For backward compatibility


class PostgresSaver(BasePostgresSaver):
    """Checkpointer that stores checkpoints in a Postgres database."""

    lock: threading.Lock

    def __init__(
        self,
        conn: _internal.Conn,
        pipe: Pipeline | None = None,
        serde: SerializerProtocol | None = None,
    ) -> None:
        super().__init__(serde=serde)
        if isinstance(conn, ConnectionPool) and pipe is not None:
            raise ValueError(
                "Pipeline should be used only with a single Connection, not ConnectionPool."
            )

        self.conn = conn
        self.pipe = pipe
        self.lock = threading.Lock()
        self.supports_pipeline = Capabilities().has_pipeline()

    @classmethod
    @contextmanager
    def from_conn_string(
        cls, conn_string: str, *, pipeline: bool = False
    ) -> Iterator[PostgresSaver]:
        """Create a new PostgresSaver instance from a connection string.

        Args:
            conn_string: The Postgres connection info string.
            pipeline: whether to use Pipeline

        Returns:
            PostgresSaver: A new PostgresSaver instance.
        """
        with Connection.connect(
            conn_string, autocommit=True, prepare_threshold=0, row_factory=dict_row
        ) as conn:
            if pipeline:
                with conn.pipeline() as pipe:
                    yield cls(conn, pipe)
            else:
                yield cls(conn)

    def setup(self) -> None:
        """Set up the checkpoint database asynchronously.

        This method creates the necessary tables in the Postgres database if they don't
        already exist and runs database migrations. It MUST be called directly by the user
        the first time checkpointer is used.
        """
        with self._cursor() as cur:
            cur.execute(self.MIGRATIONS[0])
            results = cur.execute(
                "SELECT v FROM checkpoint_migrations ORDER BY v DESC LIMIT 1"
            )
            row = results.fetchone()
            if row is None:
                version = -1
            else:
                version = row["v"]
            for v, migration in zip(
                range(version + 1, len(self.MIGRATIONS)),
                self.MIGRATIONS[version + 1 :],
                strict=False,
            ):
                cur.execute(migration)
                cur.execute("INSERT INTO checkpoint_migrations (v) VALUES (%s)", (v,))
        if self.pipe:
            self.pipe.sync()

    def list(
        self,
        config: RunnableConfig | None,
        *,
        filter: dict[str, Any] | None = None,
        before: RunnableConfig | None = None,
        limit: int | None = None,
    ) -> Iterator[CheckpointTuple]:
        """List checkpoints from the database.

        This method retrieves a list of checkpoint tuples from the Postgres database based
        on the provided config. The checkpoints are ordered by checkpoint ID in descending order (newest first).

        Args:
            config: The config to use for listing the checkpoints.
            filter: Additional filtering criteria for metadata.
            before: If provided, only checkpoints before the specified checkpoint ID are returned.
            limit: The maximum number of checkpoints to return.

        Yields:
            An iterator of checkpoint tuples.

        Examples:
            >>> from langgraph.checkpoint.postgres import PostgresSaver
            >>> DB_URI = "postgres://postgres:postgres@localhost:5432/postgres?sslmode=disable"
            >>> with PostgresSaver.from_conn_string(DB_URI) as memory:
            ... # Run a graph, then list the checkpoints
            >>>     config = {"configurable": {"thread_id": "1"}}
            >>>     checkpoints = list(memory.list(config, limit=2))
            >>> print(checkpoints)
            [CheckpointTuple(...), CheckpointTuple(...)]

            >>> config = {"configurable": {"thread_id": "1"}}
            >>> before = {"configurable": {"checkpoint_id": "1ef4f797-8335-6428-8001-8a1503f9b875"}}
            >>> with PostgresSaver.from_conn_string(DB_URI) as memory:
            ... # Run a graph, then list the checkpoints
            >>>     checkpoints = list(memory.list(config, before=before))
            >>> print(checkpoints)
            [CheckpointTuple(...), ...]
        """
        where, args = self._search_where(config, filter, before)
        query = self.SELECT_SQL + where + " ORDER BY checkpoint_id DESC"
        if limit:
            query += f" LIMIT {limit}"
        # if we change this to use .stream() we need to make sure to close the cursor
        with self._cursor() as cur:
            cur.execute(query, args)
            values = cur.fetchall()
            if not values:
                return
            # migrate pending sends if necessary
            if to_migrate := [
                v
                for v in values
                if v["checkpoint"]["v"] < 4 and v["parent_checkpoint_id"]
            ]:
                cur.execute(
                    self.SELECT_PENDING_SENDS_SQL,
                    (
                        values[0]["thread_id"],
                        [v["parent_checkpoint_id"] for v in to_migrate],
                    ),
                )
                grouped_by_parent = defaultdict(list)
                for value in to_migrate:
                    grouped_by_parent[value["parent_checkpoint_id"]].append(value)
                for sends in cur:
                    for value in grouped_by_parent[sends["checkpoint_id"]]:
                        if value["channel_values"] is None:
                            value["channel_values"] = []
                        self._migrate_pending_sends(
                            sends["sends"],
                            value["checkpoint"],
                            value["channel_values"],
                        )
            for value in values:
                yield self._load_checkpoint_tuple(value)

    def get_tuple(self, config: RunnableConfig) -> CheckpointTuple | None:
        """Get a checkpoint tuple from the database.

        This method retrieves a checkpoint tuple from the Postgres database based on the
        provided config. If the config contains a `checkpoint_id` key, the checkpoint with
        the matching thread ID and timestamp is retrieved. Otherwise, the latest checkpoint
        for the given thread ID is retrieved.

        Args:
            config: The config to use for retrieving the checkpoint.

        Returns:
            The retrieved checkpoint tuple, or None if no matching checkpoint was found.

        Examples:

            Basic:
            >>> config = {"configurable": {"thread_id": "1"}}
            >>> checkpoint_tuple = memory.get_tuple(config)
            >>> print(checkpoint_tuple)
            CheckpointTuple(...)

            With timestamp:

            >>> config = {
            ...    "configurable": {
            ...        "thread_id": "1",
            ...        "checkpoint_ns": "",
            ...        "checkpoint_id": "1ef4f797-8335-6428-8001-8a1503f9b875",
            ...    }
            ... }
            >>> checkpoint_tuple = memory.get_tuple(config)
            >>> print(checkpoint_tuple)
            CheckpointTuple(...)
        """  # noqa
        thread_id = config["configurable"]["thread_id"]
        checkpoint_id = get_checkpoint_id(config)
        checkpoint_ns = config["configurable"].get("checkpoint_ns", "")
        if checkpoint_id:
            args: tuple[Any, ...] = (thread_id, checkpoint_ns, checkpoint_id)
            where = "WHERE thread_id = %s AND checkpoint_ns = %s AND checkpoint_id = %s"
        else:
            args = (thread_id, checkpoint_ns)
            where = "WHERE thread_id = %s AND checkpoint_ns = %s ORDER BY checkpoint_id DESC LIMIT 1"

        with self._cursor() as cur:
            cur.execute(
                self.SELECT_SQL + where,
                args,
            )
            value = cur.fetchone()
            if value is None:
                return None

            # migrate pending sends if necessary
            if value["checkpoint"]["v"] < 4 and value["parent_checkpoint_id"]:
                cur.execute(
                    self.SELECT_PENDING_SENDS_SQL,
                    (thread_id, [value["parent_checkpoint_id"]]),
                )
                if sends := cur.fetchone():
                    if value["channel_values"] is None:
                        value["channel_values"] = []
                    self._migrate_pending_sends(
                        sends["sends"],
                        value["checkpoint"],
                        value["channel_values"],
                    )

            return self._load_checkpoint_tuple(value)

    def put(
        self,
        config: RunnableConfig,
        checkpoint: Checkpoint,
        metadata: CheckpointMetadata,
        new_versions: ChannelVersions,
    ) -> RunnableConfig:
        """Save a checkpoint to the database.

        This method saves a checkpoint to the Postgres database. The checkpoint is associated
        with the provided config and its parent config (if any).

        Args:
            config: The config to associate with the checkpoint.
            checkpoint: The checkpoint to save.
            metadata: Additional metadata to save with the checkpoint.
            new_versions: New channel versions as of this write.

        Returns:
            RunnableConfig: Updated configuration after storing the checkpoint.

        Examples:

            >>> from langgraph.checkpoint.postgres import PostgresSaver
            >>> DB_URI = "postgres://postgres:postgres@localhost:5432/postgres?sslmode=disable"
            >>> with PostgresSaver.from_conn_string(DB_URI) as memory:
            >>>     config = {"configurable": {"thread_id": "1", "checkpoint_ns": ""}}
            >>>     checkpoint = {"ts": "2024-05-04T06:32:42.235444+00:00", "id": "1ef4f797-8335-6428-8001-8a1503f9b875", "channel_values": {"key": "value"}}
            >>>     saved_config = memory.put(config, checkpoint, {"source": "input", "step": 1, "writes": {"key": "value"}}, {})
            >>> print(saved_config)
            {'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef4f797-8335-6428-8001-8a1503f9b875'}}
        """
        configurable = config["configurable"].copy()
        thread_id = configurable.pop("thread_id")
        checkpoint_ns = configurable.pop("checkpoint_ns")
        checkpoint_id = configurable.pop("checkpoint_id", None)
        copy = checkpoint.copy()
        copy["channel_values"] = copy["channel_values"].copy()
        next_config = {
            "configurable": {
                "thread_id": thread_id,
                "checkpoint_ns": checkpoint_ns,
                "checkpoint_id": checkpoint["id"],
            }
        }

        # inline primitive values in checkpoint table
        # others are stored in blobs table
        blob_values = {}
        for k, v in checkpoint["channel_values"].items():
            if v is None or isinstance(v, (str, int, float, bool)):
                pass
            else:
                blob_values[k] = copy["channel_values"].pop(k)

        with self._cursor(pipeline=True) as cur:
            if blob_versions := {
                k: v for k, v in new_versions.items() if k in blob_values
            }:
                cur.executemany(
                    self.UPSERT_CHECKPOINT_BLOBS_SQL,
                    self._dump_blobs(
                        thread_id,
                        checkpoint_ns,
                        blob_values,
                        blob_versions,
                    ),
                )
            cur.execute(
                self.UPSERT_CHECKPOINTS_SQL,
                (
                    thread_id,
                    checkpoint_ns,
                    checkpoint["id"],
                    checkpoint_id,
                    Jsonb(copy),
                    Jsonb(get_serializable_checkpoint_metadata(config, metadata)),
                ),
            )
        return next_config

    def put_writes(
        self,
        config: RunnableConfig,
        writes: Sequence[tuple[str, Any]],
        task_id: str,
        task_path: str = "",
    ) -> None:
        """Store intermediate writes linked to a checkpoint.

        This method saves intermediate writes associated with a checkpoint to the Postgres database.

        Args:
            config: Configuration of the related checkpoint.
            writes: List of writes to store.
            task_id: Identifier for the task creating the writes.
        """
        query = (
            self.UPSERT_CHECKPOINT_WRITES_SQL
            if all(w[0] in WRITES_IDX_MAP for w in writes)
            else self.INSERT_CHECKPOINT_WRITES_SQL
        )
        with self._cursor(pipeline=True) as cur:
            cur.executemany(
                query,
                self._dump_writes(
                    config["configurable"]["thread_id"],
                    config["configurable"]["checkpoint_ns"],
                    config["configurable"]["checkpoint_id"],
                    task_id,
                    task_path,
                    writes,
                ),
            )

    def delete_thread(self, thread_id: str) -> None:
        """Delete all checkpoints and writes associated with a thread ID.

        Args:
            thread_id: The thread ID to delete.

        Returns:
            None
        """
        with self._cursor(pipeline=True) as cur:
            cur.execute(
                "DELETE FROM checkpoints WHERE thread_id = %s",
                (str(thread_id),),
            )
            cur.execute(
                "DELETE FROM checkpoint_blobs WHERE thread_id = %s",
                (str(thread_id),),
            )
            cur.execute(
                "DELETE FROM checkpoint_writes WHERE thread_id = %s",
                (str(thread_id),),
            )

    @contextmanager
    def _cursor(self, *, pipeline: bool = False) -> Iterator[Cursor[DictRow]]:
        """Create a database cursor as a context manager.

        Args:
            pipeline: whether to use pipeline for the DB operations inside the context manager.
                Will be applied regardless of whether the PostgresSaver instance was initialized with a pipeline.
                If pipeline mode is not supported, will fall back to using transaction context manager.
        """
        with self.lock, _internal.get_connection(self.conn) as conn:
            if self.pipe:
                # a connection in pipeline mode can be used concurrently
                # in multiple threads/coroutines, but only one cursor can be
                # used at a time
                try:
                    with conn.cursor(binary=True, row_factory=dict_row) as cur:
                        yield cur
                finally:
                    if pipeline:
                        self.pipe.sync()
            elif pipeline:
                # a connection not in pipeline mode can only be used by one
                # thread/coroutine at a time, so we acquire a lock
                if self.supports_pipeline:
                    with (
                        conn.pipeline(),
                        conn.cursor(binary=True, row_factory=dict_row) as cur,
                    ):
                        yield cur
                else:
                    # Use connection's transaction context manager when pipeline mode not supported
                    with (
                        conn.transaction(),
                        conn.cursor(binary=True, row_factory=dict_row) as cur,
                    ):
                        yield cur
            else:
                with conn.cursor(binary=True, row_factory=dict_row) as cur:
                    yield cur

    def _load_checkpoint_tuple(self, value: DictRow) -> CheckpointTuple:
        """
        Convert a database row into a CheckpointTuple object.

        Args:
            value: A row from the database containing checkpoint data.

        Returns:
            CheckpointTuple: A structured representation of the checkpoint,
            including its configuration, metadata, parent checkpoint (if any),
            and pending writes.
        """
        return CheckpointTuple(
            {
                "configurable": {
                    "thread_id": value["thread_id"],
                    "checkpoint_ns": value["checkpoint_ns"],
                    "checkpoint_id": value["checkpoint_id"],
                }
            },
            {
                **value["checkpoint"],
                "channel_values": {
                    **(value["checkpoint"].get("channel_values") or {}),
                    **self._load_blobs(value["channel_values"]),
                },
            },
            value["metadata"],
            (
                {
                    "configurable": {
                        "thread_id": value["thread_id"],
                        "checkpoint_ns": value["checkpoint_ns"],
                        "checkpoint_id": value["parent_checkpoint_id"],
                    }
                }
                if value["parent_checkpoint_id"]
                else None
            ),
            self._load_writes(value["pending_writes"]),
        )


__all__ = ["PostgresSaver", "BasePostgresSaver", "ShallowPostgresSaver", "Conn"]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint-postgres/langgraph/checkpoint/postgres/_ainternal.py
```py
"""Shared async utility functions for the Postgres checkpoint & storage classes."""

from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

from psycopg import AsyncConnection
from psycopg.rows import DictRow
from psycopg_pool import AsyncConnectionPool

Conn = AsyncConnection[DictRow] | AsyncConnectionPool[AsyncConnection[DictRow]]


@asynccontextmanager
async def get_connection(
    conn: Conn,
) -> AsyncIterator[AsyncConnection[DictRow]]:
    if isinstance(conn, AsyncConnection):
        yield conn
    elif isinstance(conn, AsyncConnectionPool):
        async with conn.connection() as conn:
            yield conn
    else:
        raise TypeError(f"Invalid connection type: {type(conn)}")

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint-postgres/langgraph/checkpoint/postgres/_internal.py
```py
"""Shared utility functions for the Postgres checkpoint & storage classes."""

from collections.abc import Iterator
from contextlib import contextmanager

from psycopg import Connection
from psycopg.rows import DictRow
from psycopg_pool import ConnectionPool

Conn = Connection[DictRow] | ConnectionPool[Connection[DictRow]]


@contextmanager
def get_connection(conn: Conn) -> Iterator[Connection[DictRow]]:
    if isinstance(conn, Connection):
        yield conn
    elif isinstance(conn, ConnectionPool):
        with conn.connection() as conn:
            yield conn
    else:
        raise TypeError(f"Invalid connection type: {type(conn)}")

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint-postgres/langgraph/checkpoint/postgres/aio.py
```py
from __future__ import annotations

import asyncio
from collections import defaultdict
from collections.abc import AsyncIterator, Iterator, Sequence
from contextlib import asynccontextmanager
from typing import Any

from langchain_core.runnables import RunnableConfig
from langgraph.checkpoint.base import (
    WRITES_IDX_MAP,
    ChannelVersions,
    Checkpoint,
    CheckpointMetadata,
    CheckpointTuple,
    get_checkpoint_id,
    get_serializable_checkpoint_metadata,
)
from langgraph.checkpoint.serde.base import SerializerProtocol
from psycopg import AsyncConnection, AsyncCursor, AsyncPipeline, Capabilities
from psycopg.rows import DictRow, dict_row
from psycopg.types.json import Jsonb
from psycopg_pool import AsyncConnectionPool

from langgraph.checkpoint.postgres import _ainternal
from langgraph.checkpoint.postgres.base import BasePostgresSaver
from langgraph.checkpoint.postgres.shallow import AsyncShallowPostgresSaver

Conn = _ainternal.Conn  # For backward compatibility


class AsyncPostgresSaver(BasePostgresSaver):
    """Asynchronous checkpointer that stores checkpoints in a Postgres database."""

    lock: asyncio.Lock

    def __init__(
        self,
        conn: _ainternal.Conn,
        pipe: AsyncPipeline | None = None,
        serde: SerializerProtocol | None = None,
    ) -> None:
        super().__init__(serde=serde)
        if isinstance(conn, AsyncConnectionPool) and pipe is not None:
            raise ValueError(
                "Pipeline should be used only with a single AsyncConnection, not AsyncConnectionPool."
            )

        self.conn = conn
        self.pipe = pipe
        self.lock = asyncio.Lock()
        self.loop = asyncio.get_running_loop()
        self.supports_pipeline = Capabilities().has_pipeline()

    @classmethod
    @asynccontextmanager
    async def from_conn_string(
        cls,
        conn_string: str,
        *,
        pipeline: bool = False,
        serde: SerializerProtocol | None = None,
    ) -> AsyncIterator[AsyncPostgresSaver]:
        """Create a new AsyncPostgresSaver instance from a connection string.

        Args:
            conn_string: The Postgres connection info string.
            pipeline: whether to use AsyncPipeline

        Returns:
            AsyncPostgresSaver: A new AsyncPostgresSaver instance.
        """
        async with await AsyncConnection.connect(
            conn_string, autocommit=True, prepare_threshold=0, row_factory=dict_row
        ) as conn:
            if pipeline:
                async with conn.pipeline() as pipe:
                    yield cls(conn=conn, pipe=pipe, serde=serde)
            else:
                yield cls(conn=conn, serde=serde)

    async def setup(self) -> None:
        """Set up the checkpoint database asynchronously.

        This method creates the necessary tables in the Postgres database if they don't
        already exist and runs database migrations. It MUST be called directly by the user
        the first time checkpointer is used.
        """
        async with self._cursor() as cur:
            await cur.execute(self.MIGRATIONS[0])
            results = await cur.execute(
                "SELECT v FROM checkpoint_migrations ORDER BY v DESC LIMIT 1"
            )
            row = await results.fetchone()
            if row is None:
                version = -1
            else:
                version = row["v"]
            for v, migration in zip(
                range(version + 1, len(self.MIGRATIONS)),
                self.MIGRATIONS[version + 1 :],
                strict=False,
            ):
                await cur.execute(migration)
                await cur.execute(
                    "INSERT INTO checkpoint_migrations (v) VALUES (%s)", (v,)
                )
        if self.pipe:
            await self.pipe.sync()

    async def alist(
        self,
        config: RunnableConfig | None,
        *,
        filter: dict[str, Any] | None = None,
        before: RunnableConfig | None = None,
        limit: int | None = None,
    ) -> AsyncIterator[CheckpointTuple]:
        """List checkpoints from the database asynchronously.

        This method retrieves a list of checkpoint tuples from the Postgres database based
        on the provided config. The checkpoints are ordered by checkpoint ID in descending order (newest first).

        Args:
            config: Base configuration for filtering checkpoints.
            filter: Additional filtering criteria for metadata.
            before: If provided, only checkpoints before the specified checkpoint ID are returned.
            limit: Maximum number of checkpoints to return.

        Yields:
            An asynchronous iterator of matching checkpoint tuples.
        """
        where, args = self._search_where(config, filter, before)
        query = self.SELECT_SQL + where + " ORDER BY checkpoint_id DESC"
        if limit:
            query += f" LIMIT {limit}"
        # if we change this to use .stream() we need to make sure to close the cursor
        async with self._cursor() as cur:
            await cur.execute(query, args, binary=True)
            values = await cur.fetchall()
            if not values:
                return
            # migrate pending sends if necessary
            if to_migrate := [
                v
                for v in values
                if v["checkpoint"]["v"] < 4 and v["parent_checkpoint_id"]
            ]:
                await cur.execute(
                    self.SELECT_PENDING_SENDS_SQL,
                    (
                        values[0]["thread_id"],
                        [v["parent_checkpoint_id"] for v in to_migrate],
                    ),
                )
                grouped_by_parent = defaultdict(list)
                for value in to_migrate:
                    grouped_by_parent[value["parent_checkpoint_id"]].append(value)
                async for sends in cur:
                    for value in grouped_by_parent[sends["checkpoint_id"]]:
                        if value["channel_values"] is None:
                            value["channel_values"] = []
                        self._migrate_pending_sends(
                            sends["sends"],
                            value["checkpoint"],
                            value["channel_values"],
                        )
            for value in values:
                yield await self._load_checkpoint_tuple(value)

    async def aget_tuple(self, config: RunnableConfig) -> CheckpointTuple | None:
        """Get a checkpoint tuple from the database asynchronously.

        This method retrieves a checkpoint tuple from the Postgres database based on the
        provided config. If the config contains a `checkpoint_id` key, the checkpoint with
        the matching thread ID and "checkpoint_id" is retrieved. Otherwise, the latest checkpoint
        for the given thread ID is retrieved.

        Args:
            config: The config to use for retrieving the checkpoint.

        Returns:
            The retrieved checkpoint tuple, or None if no matching checkpoint was found.
        """
        thread_id = config["configurable"]["thread_id"]
        checkpoint_id = get_checkpoint_id(config)
        checkpoint_ns = config["configurable"].get("checkpoint_ns", "")
        if checkpoint_id:
            args: tuple[Any, ...] = (thread_id, checkpoint_ns, checkpoint_id)
            where = "WHERE thread_id = %s AND checkpoint_ns = %s AND checkpoint_id = %s"
        else:
            args = (thread_id, checkpoint_ns)
            where = "WHERE thread_id = %s AND checkpoint_ns = %s ORDER BY checkpoint_id DESC LIMIT 1"

        async with self._cursor() as cur:
            await cur.execute(
                self.SELECT_SQL + where,
                args,
                binary=True,
            )
            value = await cur.fetchone()
            if value is None:
                return None

            # migrate pending sends if necessary
            if value["checkpoint"]["v"] < 4 and value["parent_checkpoint_id"]:
                await cur.execute(
                    self.SELECT_PENDING_SENDS_SQL,
                    (thread_id, [value["parent_checkpoint_id"]]),
                )
                if sends := await cur.fetchone():
                    if value["channel_values"] is None:
                        value["channel_values"] = []
                    self._migrate_pending_sends(
                        sends["sends"],
                        value["checkpoint"],
                        value["channel_values"],
                    )

            return await self._load_checkpoint_tuple(value)

    async def aput(
        self,
        config: RunnableConfig,
        checkpoint: Checkpoint,
        metadata: CheckpointMetadata,
        new_versions: ChannelVersions,
    ) -> RunnableConfig:
        """Save a checkpoint to the database asynchronously.

        This method saves a checkpoint to the Postgres database. The checkpoint is associated
        with the provided config and its parent config (if any).

        Args:
            config: The config to associate with the checkpoint.
            checkpoint: The checkpoint to save.
            metadata: Additional metadata to save with the checkpoint.
            new_versions: New channel versions as of this write.

        Returns:
            RunnableConfig: Updated configuration after storing the checkpoint.
        """
        configurable = config["configurable"].copy()
        thread_id = configurable.pop("thread_id")
        checkpoint_ns = configurable.pop("checkpoint_ns")
        checkpoint_id = configurable.pop("checkpoint_id", None)

        copy = checkpoint.copy()
        copy["channel_values"] = copy["channel_values"].copy()
        next_config = {
            "configurable": {
                "thread_id": thread_id,
                "checkpoint_ns": checkpoint_ns,
                "checkpoint_id": checkpoint["id"],
            }
        }

        # inline primitive values in checkpoint table
        # others are stored in blobs table
        blob_values = {}
        for k, v in checkpoint["channel_values"].items():
            if v is None or isinstance(v, (str, int, float, bool)):
                pass
            else:
                blob_values[k] = copy["channel_values"].pop(k)

        async with self._cursor(pipeline=True) as cur:
            if blob_versions := {
                k: v for k, v in new_versions.items() if k in blob_values
            }:
                await cur.executemany(
                    self.UPSERT_CHECKPOINT_BLOBS_SQL,
                    await asyncio.to_thread(
                        self._dump_blobs,
                        thread_id,
                        checkpoint_ns,
                        blob_values,
                        blob_versions,
                    ),
                )
            await cur.execute(
                self.UPSERT_CHECKPOINTS_SQL,
                (
                    thread_id,
                    checkpoint_ns,
                    checkpoint["id"],
                    checkpoint_id,
                    Jsonb(copy),
                    Jsonb(get_serializable_checkpoint_metadata(config, metadata)),
                ),
            )
        return next_config

    async def aput_writes(
        self,
        config: RunnableConfig,
        writes: Sequence[tuple[str, Any]],
        task_id: str,
        task_path: str = "",
    ) -> None:
        """Store intermediate writes linked to a checkpoint asynchronously.

        This method saves intermediate writes associated with a checkpoint to the database.

        Args:
            config: Configuration of the related checkpoint.
            writes: List of writes to store, each as (channel, value) pair.
            task_id: Identifier for the task creating the writes.
        """
        query = (
            self.UPSERT_CHECKPOINT_WRITES_SQL
            if all(w[0] in WRITES_IDX_MAP for w in writes)
            else self.INSERT_CHECKPOINT_WRITES_SQL
        )
        params = await asyncio.to_thread(
            self._dump_writes,
            config["configurable"]["thread_id"],
            config["configurable"]["checkpoint_ns"],
            config["configurable"]["checkpoint_id"],
            task_id,
            task_path,
            writes,
        )
        async with self._cursor(pipeline=True) as cur:
            await cur.executemany(query, params)

    async def adelete_thread(self, thread_id: str) -> None:
        """Delete all checkpoints and writes associated with a thread ID.

        Args:
            thread_id: The thread ID to delete.

        Returns:
            None
        """
        async with self._cursor(pipeline=True) as cur:
            await cur.execute(
                "DELETE FROM checkpoints WHERE thread_id = %s",
                (str(thread_id),),
            )
            await cur.execute(
                "DELETE FROM checkpoint_blobs WHERE thread_id = %s",
                (str(thread_id),),
            )
            await cur.execute(
                "DELETE FROM checkpoint_writes WHERE thread_id = %s",
                (str(thread_id),),
            )

    @asynccontextmanager
    async def _cursor(
        self, *, pipeline: bool = False
    ) -> AsyncIterator[AsyncCursor[DictRow]]:
        """Create a database cursor as a context manager.

        Args:
            pipeline: whether to use pipeline for the DB operations inside the context manager.
                Will be applied regardless of whether the AsyncPostgresSaver instance was initialized with a pipeline.
                If pipeline mode is not supported, will fall back to using transaction context manager.
        """
        async with self.lock, _ainternal.get_connection(self.conn) as conn:
            if self.pipe:
                # a connection in pipeline mode can be used concurrently
                # in multiple threads/coroutines, but only one cursor can be
                # used at a time
                try:
                    async with conn.cursor(binary=True, row_factory=dict_row) as cur:
                        yield cur
                finally:
                    if pipeline:
                        await self.pipe.sync()
            elif pipeline:
                # a connection not in pipeline mode can only be used by one
                # thread/coroutine at a time, so we acquire a lock
                if self.supports_pipeline:
                    async with (
                        conn.pipeline(),
                        conn.cursor(binary=True, row_factory=dict_row) as cur,
                    ):
                        yield cur
                else:
                    # Use connection's transaction context manager when pipeline mode not supported
                    async with (
                        conn.transaction(),
                        conn.cursor(binary=True, row_factory=dict_row) as cur,
                    ):
                        yield cur
            else:
                async with conn.cursor(binary=True, row_factory=dict_row) as cur:
                    yield cur

    async def _load_checkpoint_tuple(self, value: DictRow) -> CheckpointTuple:
        """
        Convert a database row into a CheckpointTuple object.

        Args:
            value: A row from the database containing checkpoint data.

        Returns:
            CheckpointTuple: A structured representation of the checkpoint,
            including its configuration, metadata, parent checkpoint (if any),
            and pending writes.
        """
        return CheckpointTuple(
            {
                "configurable": {
                    "thread_id": value["thread_id"],
                    "checkpoint_ns": value["checkpoint_ns"],
                    "checkpoint_id": value["checkpoint_id"],
                }
            },
            {
                **value["checkpoint"],
                "channel_values": {
                    **(value["checkpoint"].get("channel_values") or {}),
                    **self._load_blobs(value["channel_values"]),
                },
            },
            value["metadata"],
            (
                {
                    "configurable": {
                        "thread_id": value["thread_id"],
                        "checkpoint_ns": value["checkpoint_ns"],
                        "checkpoint_id": value["parent_checkpoint_id"],
                    }
                }
                if value["parent_checkpoint_id"]
                else None
            ),
            await asyncio.to_thread(self._load_writes, value["pending_writes"]),
        )

    def list(
        self,
        config: RunnableConfig | None,
        *,
        filter: dict[str, Any] | None = None,
        before: RunnableConfig | None = None,
        limit: int | None = None,
    ) -> Iterator[CheckpointTuple]:
        """List checkpoints from the database.

        This method retrieves a list of checkpoint tuples from the Postgres database based
        on the provided config. The checkpoints are ordered by checkpoint ID in descending order (newest first).

        Args:
            config: Base configuration for filtering checkpoints.
            filter: Additional filtering criteria for metadata.
            before: If provided, only checkpoints before the specified checkpoint ID are returned.
            limit: Maximum number of checkpoints to return.

        Yields:
            An iterator of matching checkpoint tuples.
        """
        try:
            # check if we are in the main thread, only bg threads can block
            # we don't check in other methods to avoid the overhead
            if asyncio.get_running_loop() is self.loop:
                raise asyncio.InvalidStateError(
                    "Synchronous calls to AsyncPostgresSaver are only allowed from a "
                    "different thread. From the main thread, use the async interface. "
                    "For example, use `checkpointer.alist(...)` or `await "
                    "graph.ainvoke(...)`."
                )
        except RuntimeError:
            pass
        aiter_ = self.alist(config, filter=filter, before=before, limit=limit)
        while True:
            try:
                yield asyncio.run_coroutine_threadsafe(
                    anext(aiter_),  # type: ignore[arg-type]  # noqa: F821
                    self.loop,
                ).result()
            except StopAsyncIteration:
                break

    def get_tuple(self, config: RunnableConfig) -> CheckpointTuple | None:
        """Get a checkpoint tuple from the database.

        This method retrieves a checkpoint tuple from the Postgres database based on the
        provided config. If the config contains a `checkpoint_id` key, the checkpoint with
        the matching thread ID and "checkpoint_id" is retrieved. Otherwise, the latest checkpoint
        for the given thread ID is retrieved.

        Args:
            config: The config to use for retrieving the checkpoint.

        Returns:
            The retrieved checkpoint tuple, or None if no matching checkpoint was found.
        """
        try:
            # check if we are in the main thread, only bg threads can block
            # we don't check in other methods to avoid the overhead
            if asyncio.get_running_loop() is self.loop:
                raise asyncio.InvalidStateError(
                    "Synchronous calls to AsyncPostgresSaver are only allowed from a "
                    "different thread. From the main thread, use the async interface. "
                    "For example, use `await checkpointer.aget_tuple(...)` or `await "
                    "graph.ainvoke(...)`."
                )
        except RuntimeError:
            pass
        return asyncio.run_coroutine_threadsafe(
            self.aget_tuple(config), self.loop
        ).result()

    def put(
        self,
        config: RunnableConfig,
        checkpoint: Checkpoint,
        metadata: CheckpointMetadata,
        new_versions: ChannelVersions,
    ) -> RunnableConfig:
        """Save a checkpoint to the database.

        This method saves a checkpoint to the Postgres database. The checkpoint is associated
        with the provided config and its parent config (if any).

        Args:
            config: The config to associate with the checkpoint.
            checkpoint: The checkpoint to save.
            metadata: Additional metadata to save with the checkpoint.
            new_versions: New channel versions as of this write.

        Returns:
            RunnableConfig: Updated configuration after storing the checkpoint.
        """
        return asyncio.run_coroutine_threadsafe(
            self.aput(config, checkpoint, metadata, new_versions), self.loop
        ).result()

    def put_writes(
        self,
        config: RunnableConfig,
        writes: Sequence[tuple[str, Any]],
        task_id: str,
        task_path: str = "",
    ) -> None:
        """Store intermediate writes linked to a checkpoint.

        This method saves intermediate writes associated with a checkpoint to the database.

        Args:
            config: Configuration of the related checkpoint.
            writes: List of writes to store, each as (channel, value) pair.
            task_id: Identifier for the task creating the writes.
            task_path: Path of the task creating the writes.
        """
        return asyncio.run_coroutine_threadsafe(
            self.aput_writes(config, writes, task_id, task_path), self.loop
        ).result()

    def delete_thread(self, thread_id: str) -> None:
        """Delete all checkpoints and writes associated with a thread ID.

        Args:
            thread_id: The thread ID to delete.

        Returns:
            None
        """
        try:
            # check if we are in the main thread, only bg threads can block
            # we don't check in other methods to avoid the overhead
            if asyncio.get_running_loop() is self.loop:
                raise asyncio.InvalidStateError(
                    "Synchronous calls to AsyncPostgresSaver are only allowed from a "
                    "different thread. From the main thread, use the async interface. "
                    "For example, use `await checkpointer.aget_tuple(...)` or `await "
                    "graph.ainvoke(...)`."
                )
        except RuntimeError:
            pass
        return asyncio.run_coroutine_threadsafe(
            self.adelete_thread(thread_id), self.loop
        ).result()


__all__ = ["AsyncPostgresSaver", "AsyncShallowPostgresSaver", "Conn"]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint-postgres/langgraph/checkpoint/postgres/base.py
```py
from __future__ import annotations

import random
import warnings
from collections.abc import Sequence
from importlib.metadata import version as get_version
from typing import Any, cast

from langchain_core.runnables import RunnableConfig
from langgraph.checkpoint.base import (
    WRITES_IDX_MAP,
    BaseCheckpointSaver,
    ChannelVersions,
    get_checkpoint_id,
)
from langgraph.checkpoint.serde.types import TASKS
from psycopg.types.json import Jsonb

MetadataInput = dict[str, Any] | None

try:
    major, minor = get_version("langgraph").split(".")[:2]
    if int(major) == 0 and int(minor) < 5:
        warnings.warn(
            "You're using incompatible versions of langgraph and checkpoint-postgres. Please upgrade langgraph to avoid unexpected behavior.",
            DeprecationWarning,
            stacklevel=2,
        )
except Exception:
    # skip version check if running from source
    pass

"""
To add a new migration, add a new string to the MIGRATIONS list.
The position of the migration in the list is the version number.
"""
MIGRATIONS = [
    """CREATE TABLE IF NOT EXISTS checkpoint_migrations (
    v INTEGER PRIMARY KEY
);""",
    """CREATE TABLE IF NOT EXISTS checkpoints (
    thread_id TEXT NOT NULL,
    checkpoint_ns TEXT NOT NULL DEFAULT '',
    checkpoint_id TEXT NOT NULL,
    parent_checkpoint_id TEXT,
    type TEXT,
    checkpoint JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id)
);""",
    """CREATE TABLE IF NOT EXISTS checkpoint_blobs (
    thread_id TEXT NOT NULL,
    checkpoint_ns TEXT NOT NULL DEFAULT '',
    channel TEXT NOT NULL,
    version TEXT NOT NULL,
    type TEXT NOT NULL,
    blob BYTEA,
    PRIMARY KEY (thread_id, checkpoint_ns, channel, version)
);""",
    """CREATE TABLE IF NOT EXISTS checkpoint_writes (
    thread_id TEXT NOT NULL,
    checkpoint_ns TEXT NOT NULL DEFAULT '',
    checkpoint_id TEXT NOT NULL,
    task_id TEXT NOT NULL,
    idx INTEGER NOT NULL,
    channel TEXT NOT NULL,
    type TEXT,
    blob BYTEA NOT NULL,
    PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id, task_id, idx)
);""",
    "ALTER TABLE checkpoint_blobs ALTER COLUMN blob DROP not null;",
    # NOTE: this is a no-op migration to ensure that the versions in the migrations table are correct.
    # This is necessary due to an empty migration previously added to the list.
    "SELECT 1;",
    """
    CREATE INDEX CONCURRENTLY IF NOT EXISTS checkpoints_thread_id_idx ON checkpoints(thread_id);
    """,
    """
    CREATE INDEX CONCURRENTLY IF NOT EXISTS checkpoint_blobs_thread_id_idx ON checkpoint_blobs(thread_id);
    """,
    """
    CREATE INDEX CONCURRENTLY IF NOT EXISTS checkpoint_writes_thread_id_idx ON checkpoint_writes(thread_id);
    """,
    """ALTER TABLE checkpoint_writes ADD COLUMN IF NOT EXISTS task_path TEXT NOT NULL DEFAULT '';""",
]

SELECT_SQL = """
select
    thread_id,
    checkpoint,
    checkpoint_ns,
    checkpoint_id,
    parent_checkpoint_id,
    metadata,
    (
        select array_agg(array[bl.channel::bytea, bl.type::bytea, bl.blob])
        from jsonb_each_text(checkpoint -> 'channel_versions')
        inner join checkpoint_blobs bl
            on bl.thread_id = checkpoints.thread_id
            and bl.checkpoint_ns = checkpoints.checkpoint_ns
            and bl.channel = jsonb_each_text.key
            and bl.version = jsonb_each_text.value
    ) as channel_values,
    (
        select
        array_agg(array[cw.task_id::text::bytea, cw.channel::bytea, cw.type::bytea, cw.blob] order by cw.task_id, cw.idx)
        from checkpoint_writes cw
        where cw.thread_id = checkpoints.thread_id
            and cw.checkpoint_ns = checkpoints.checkpoint_ns
            and cw.checkpoint_id = checkpoints.checkpoint_id
    ) as pending_writes
from checkpoints """

SELECT_PENDING_SENDS_SQL = f"""
select
    checkpoint_id,
    array_agg(array[type::bytea, blob] order by task_path, task_id, idx) as sends
from checkpoint_writes
where thread_id = %s
    and checkpoint_id = any(%s)
    and channel = '{TASKS}'
group by checkpoint_id
"""

UPSERT_CHECKPOINT_BLOBS_SQL = """
    INSERT INTO checkpoint_blobs (thread_id, checkpoint_ns, channel, version, type, blob)
    VALUES (%s, %s, %s, %s, %s, %s)
    ON CONFLICT (thread_id, checkpoint_ns, channel, version) DO NOTHING
"""

UPSERT_CHECKPOINTS_SQL = """
    INSERT INTO checkpoints (thread_id, checkpoint_ns, checkpoint_id, parent_checkpoint_id, checkpoint, metadata)
    VALUES (%s, %s, %s, %s, %s, %s)
    ON CONFLICT (thread_id, checkpoint_ns, checkpoint_id)
    DO UPDATE SET
        checkpoint = EXCLUDED.checkpoint,
        metadata = EXCLUDED.metadata;
"""

UPSERT_CHECKPOINT_WRITES_SQL = """
    INSERT INTO checkpoint_writes (thread_id, checkpoint_ns, checkpoint_id, task_id, task_path, idx, channel, type, blob)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON CONFLICT (thread_id, checkpoint_ns, checkpoint_id, task_id, idx) DO UPDATE SET
        channel = EXCLUDED.channel,
        type = EXCLUDED.type,
        blob = EXCLUDED.blob;
"""

INSERT_CHECKPOINT_WRITES_SQL = """
    INSERT INTO checkpoint_writes (thread_id, checkpoint_ns, checkpoint_id, task_id, task_path, idx, channel, type, blob)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON CONFLICT (thread_id, checkpoint_ns, checkpoint_id, task_id, idx) DO NOTHING
"""


class BasePostgresSaver(BaseCheckpointSaver[str]):
    SELECT_SQL = SELECT_SQL
    SELECT_PENDING_SENDS_SQL = SELECT_PENDING_SENDS_SQL
    MIGRATIONS = MIGRATIONS
    UPSERT_CHECKPOINT_BLOBS_SQL = UPSERT_CHECKPOINT_BLOBS_SQL
    UPSERT_CHECKPOINTS_SQL = UPSERT_CHECKPOINTS_SQL
    UPSERT_CHECKPOINT_WRITES_SQL = UPSERT_CHECKPOINT_WRITES_SQL
    INSERT_CHECKPOINT_WRITES_SQL = INSERT_CHECKPOINT_WRITES_SQL

    supports_pipeline: bool

    def _migrate_pending_sends(
        self,
        pending_sends: list[tuple[bytes, bytes]],
        checkpoint: dict[str, Any],
        channel_values: list[tuple[bytes, bytes, bytes]],
    ) -> None:
        if not pending_sends:
            return
        # add to values
        enc, blob = self.serde.dumps_typed(
            [self.serde.loads_typed((c.decode(), b)) for c, b in pending_sends],
        )
        channel_values.append((TASKS.encode(), enc.encode(), blob))
        # add to versions
        checkpoint["channel_versions"][TASKS] = (
            max(checkpoint["channel_versions"].values())
            if checkpoint["channel_versions"]
            else self.get_next_version(None, None)
        )

    def _load_blobs(
        self, blob_values: list[tuple[bytes, bytes, bytes]]
    ) -> dict[str, Any]:
        if not blob_values:
            return {}
        return {
            k.decode(): self.serde.loads_typed((t.decode(), v))
            for k, t, v in blob_values
            if t.decode() != "empty"
        }

    def _dump_blobs(
        self,
        thread_id: str,
        checkpoint_ns: str,
        values: dict[str, Any],
        versions: ChannelVersions,
    ) -> list[tuple[str, str, str, str, str, bytes | None]]:
        if not versions:
            return []

        return [
            (
                thread_id,
                checkpoint_ns,
                k,
                cast(str, ver),
                *(
                    self.serde.dumps_typed(values[k])
                    if k in values
                    else ("empty", None)
                ),
            )
            for k, ver in versions.items()
        ]

    def _load_writes(
        self, writes: list[tuple[bytes, bytes, bytes, bytes]]
    ) -> list[tuple[str, str, Any]]:
        return (
            [
                (
                    tid.decode(),
                    channel.decode(),
                    self.serde.loads_typed((t.decode(), v)),
                )
                for tid, channel, t, v in writes
            ]
            if writes
            else []
        )

    def _dump_writes(
        self,
        thread_id: str,
        checkpoint_ns: str,
        checkpoint_id: str,
        task_id: str,
        task_path: str,
        writes: Sequence[tuple[str, Any]],
    ) -> list[tuple[str, str, str, str, str, int, str, str, bytes]]:
        return [
            (
                thread_id,
                checkpoint_ns,
                checkpoint_id,
                task_id,
                task_path,
                WRITES_IDX_MAP.get(channel, idx),
                channel,
                *self.serde.dumps_typed(value),
            )
            for idx, (channel, value) in enumerate(writes)
        ]

    def get_next_version(self, current: str | None, channel: None) -> str:
        if current is None:
            current_v = 0
        elif isinstance(current, int):
            current_v = current
        else:
            current_v = int(current.split(".")[0])
        next_v = current_v + 1
        next_h = random.random()
        return f"{next_v:032}.{next_h:016}"

    def _search_where(
        self,
        config: RunnableConfig | None,
        filter: MetadataInput,
        before: RunnableConfig | None = None,
    ) -> tuple[str, list[Any]]:
        """Return WHERE clause predicates for alist() given config, filter, before.

        This method returns a tuple of a string and a tuple of values. The string
        is the parametered WHERE clause predicate (including the WHERE keyword):
        "WHERE column1 = $1 AND column2 IS $2". The list of values contains the
        values for each of the corresponding parameters.
        """
        wheres = []
        param_values = []

        # construct predicate for config filter
        if config:
            wheres.append("thread_id = %s ")
            param_values.append(config["configurable"]["thread_id"])
            checkpoint_ns = config["configurable"].get("checkpoint_ns")
            if checkpoint_ns is not None:
                wheres.append("checkpoint_ns = %s")
                param_values.append(checkpoint_ns)

            if checkpoint_id := get_checkpoint_id(config):
                wheres.append("checkpoint_id = %s ")
                param_values.append(checkpoint_id)

        # construct predicate for metadata filter
        if filter:
            wheres.append("metadata @> %s ")
            param_values.append(Jsonb(filter))

        # construct predicate for `before`
        if before is not None:
            wheres.append("checkpoint_id < %s ")
            param_values.append(get_checkpoint_id(before))

        return (
            "WHERE " + " AND ".join(wheres) if wheres else "",
            param_values,
        )

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint-postgres/langgraph/checkpoint/postgres/py.typed
```typed

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint-postgres/langgraph/checkpoint/postgres/shallow.py
```py
import asyncio
import threading
import warnings
from collections.abc import AsyncIterator, Iterator, Sequence
from contextlib import asynccontextmanager, contextmanager
from typing import Any

from langchain_core.runnables import RunnableConfig
from langgraph.checkpoint.base import (
    WRITES_IDX_MAP,
    ChannelVersions,
    Checkpoint,
    CheckpointMetadata,
    CheckpointTuple,
    get_serializable_checkpoint_metadata,
)
from langgraph.checkpoint.serde.base import SerializerProtocol
from langgraph.checkpoint.serde.types import TASKS
from psycopg import (
    AsyncConnection,
    AsyncCursor,
    AsyncPipeline,
    Capabilities,
    Connection,
    Cursor,
    Pipeline,
)
from psycopg.rows import DictRow, dict_row
from psycopg.types.json import Jsonb
from psycopg_pool import AsyncConnectionPool, ConnectionPool

from langgraph.checkpoint.postgres import _ainternal, _internal
from langgraph.checkpoint.postgres.base import BasePostgresSaver

"""
To add a new migration, add a new string to the MIGRATIONS list.
The position of the migration in the list is the version number.
"""
MIGRATIONS = [
    """CREATE TABLE IF NOT EXISTS checkpoint_migrations (
    v INTEGER PRIMARY KEY
);""",
    """CREATE TABLE IF NOT EXISTS checkpoints (
    thread_id TEXT NOT NULL,
    checkpoint_ns TEXT NOT NULL DEFAULT '',
    type TEXT,
    checkpoint JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    PRIMARY KEY (thread_id, checkpoint_ns)
);""",
    """CREATE TABLE IF NOT EXISTS checkpoint_blobs (
    thread_id TEXT NOT NULL,
    checkpoint_ns TEXT NOT NULL DEFAULT '',
    channel TEXT NOT NULL,
    type TEXT NOT NULL,
    blob BYTEA,
    PRIMARY KEY (thread_id, checkpoint_ns, channel)
);""",
    """CREATE TABLE IF NOT EXISTS checkpoint_writes (
    thread_id TEXT NOT NULL,
    checkpoint_ns TEXT NOT NULL DEFAULT '',
    checkpoint_id TEXT NOT NULL,
    task_id TEXT NOT NULL,
    idx INTEGER NOT NULL,
    channel TEXT NOT NULL,
    type TEXT,
    blob BYTEA NOT NULL,
    PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id, task_id, idx)
);""",
    """
    CREATE INDEX CONCURRENTLY IF NOT EXISTS checkpoints_thread_id_idx ON checkpoints(thread_id);
    """,
    """
    CREATE INDEX CONCURRENTLY IF NOT EXISTS checkpoint_blobs_thread_id_idx ON checkpoint_blobs(thread_id);
    """,
    """
    CREATE INDEX CONCURRENTLY IF NOT EXISTS checkpoint_writes_thread_id_idx ON checkpoint_writes(thread_id);
    """,
    """
    ALTER TABLE checkpoint_writes ADD COLUMN IF NOT EXISTS task_path TEXT NOT NULL DEFAULT '';
    """,
]

SELECT_SQL = f"""
select
    thread_id,
    checkpoint,
    checkpoint_ns,
    metadata,
    (
        select array_agg(array[bl.channel::bytea, bl.type::bytea, bl.blob])
        from jsonb_each_text(checkpoint -> 'channel_versions')
        inner join checkpoint_blobs bl
            on bl.thread_id = checkpoints.thread_id
            and bl.checkpoint_ns = checkpoints.checkpoint_ns
            and bl.channel = jsonb_each_text.key
    ) as channel_values,
    (
        select
        array_agg(array[cw.task_id::text::bytea, cw.channel::bytea, cw.type::bytea, cw.blob] order by cw.task_id, cw.idx)
        from checkpoint_writes cw
        where cw.thread_id = checkpoints.thread_id
            and cw.checkpoint_ns = checkpoints.checkpoint_ns
            and cw.checkpoint_id = (checkpoint->>'id')
    ) as pending_writes,
    (
        select array_agg(array[cw.type::bytea, cw.blob] order by cw.task_path, cw.task_id, cw.idx)
        from checkpoint_writes cw
        where cw.thread_id = checkpoints.thread_id
            and cw.checkpoint_ns = checkpoints.checkpoint_ns
            and cw.channel = '{TASKS}'
    ) as pending_sends
from checkpoints """

UPSERT_CHECKPOINT_BLOBS_SQL = """
    INSERT INTO checkpoint_blobs (thread_id, checkpoint_ns, channel, type, blob)
    VALUES (%s, %s, %s, %s, %s)
    ON CONFLICT (thread_id, checkpoint_ns, channel) DO UPDATE SET
        type = EXCLUDED.type,
        blob = EXCLUDED.blob;
"""

UPSERT_CHECKPOINTS_SQL = """
    INSERT INTO checkpoints (thread_id, checkpoint_ns, checkpoint, metadata)
    VALUES (%s, %s, %s, %s)
    ON CONFLICT (thread_id, checkpoint_ns)
    DO UPDATE SET
        checkpoint = EXCLUDED.checkpoint,
        metadata = EXCLUDED.metadata;
"""

UPSERT_CHECKPOINT_WRITES_SQL = """
    INSERT INTO checkpoint_writes (thread_id, checkpoint_ns, checkpoint_id, task_id, task_path, idx, channel, type, blob)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON CONFLICT (thread_id, checkpoint_ns, checkpoint_id, task_id, idx) DO UPDATE SET
        channel = EXCLUDED.channel,
        type = EXCLUDED.type,
        blob = EXCLUDED.blob;
"""

INSERT_CHECKPOINT_WRITES_SQL = """
    INSERT INTO checkpoint_writes (thread_id, checkpoint_ns, checkpoint_id, task_id, task_path, idx, channel, type, blob)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON CONFLICT (thread_id, checkpoint_ns, checkpoint_id, task_id, idx) DO NOTHING
"""


def _dump_blobs(
    serde: SerializerProtocol,
    thread_id: str,
    checkpoint_ns: str,
    values: dict[str, Any],
    versions: ChannelVersions,
) -> list[tuple[str, str, str, str, bytes | None]]:
    if not versions:
        return []

    return [
        (
            thread_id,
            checkpoint_ns,
            k,
            *(serde.dumps_typed(values[k]) if k in values else ("empty", None)),
        )
        for k in versions
    ]


class ShallowPostgresSaver(BasePostgresSaver):
    """A checkpoint saver that uses Postgres to store checkpoints.

    This checkpointer ONLY stores the most recent checkpoint and does NOT retain any history.
    It is meant to be a light-weight drop-in replacement for the PostgresSaver that
    supports most of the LangGraph persistence functionality with the exception of time travel.
    """

    SELECT_SQL = SELECT_SQL
    MIGRATIONS = MIGRATIONS
    UPSERT_CHECKPOINT_BLOBS_SQL = UPSERT_CHECKPOINT_BLOBS_SQL
    UPSERT_CHECKPOINTS_SQL = UPSERT_CHECKPOINTS_SQL
    UPSERT_CHECKPOINT_WRITES_SQL = UPSERT_CHECKPOINT_WRITES_SQL
    INSERT_CHECKPOINT_WRITES_SQL = INSERT_CHECKPOINT_WRITES_SQL

    lock: threading.Lock

    def __init__(
        self,
        conn: _internal.Conn,
        pipe: Pipeline | None = None,
        serde: SerializerProtocol | None = None,
    ) -> None:
        warnings.warn(
            "ShallowPostgresSaver is deprecated as of version 2.0.20 and will be removed in 3.0.0. "
            "Use PostgresSaver instead, and invoke the graph with `graph.invoke(..., durability='exit')`.",
            DeprecationWarning,
            stacklevel=2,
        )
        super().__init__(serde=serde)
        if isinstance(conn, ConnectionPool) and pipe is not None:
            raise ValueError(
                "Pipeline should be used only with a single Connection, not ConnectionPool."
            )

        self.conn = conn
        self.pipe = pipe
        self.lock = threading.Lock()
        self.supports_pipeline = Capabilities().has_pipeline()

    @classmethod
    @contextmanager
    def from_conn_string(
        cls, conn_string: str, *, pipeline: bool = False
    ) -> Iterator["ShallowPostgresSaver"]:
        """Create a new ShallowPostgresSaver instance from a connection string.

        Args:
            conn_string: The Postgres connection info string.
            pipeline: whether to use Pipeline

        Returns:
            ShallowPostgresSaver: A new ShallowPostgresSaver instance.
        """
        with Connection.connect(
            conn_string, autocommit=True, prepare_threshold=0, row_factory=dict_row
        ) as conn:
            if pipeline:
                with conn.pipeline() as pipe:
                    yield cls(conn, pipe)
            else:
                yield cls(conn)

    def setup(self) -> None:
        """Set up the checkpoint database asynchronously.

        This method creates the necessary tables in the Postgres database if they don't
        already exist and runs database migrations. It MUST be called directly by the user
        the first time checkpointer is used.
        """
        with self._cursor() as cur:
            cur.execute(self.MIGRATIONS[0])
            results = cur.execute(
                "SELECT v FROM checkpoint_migrations ORDER BY v DESC LIMIT 1"
            )
            row = results.fetchone()
            if row is None:
                version = -1
            else:
                version = row["v"]
            for v, migration in zip(
                range(version + 1, len(self.MIGRATIONS)),
                self.MIGRATIONS[version + 1 :],
                strict=False,
            ):
                cur.execute(migration)
                cur.execute("INSERT INTO checkpoint_migrations (v) VALUES (%s)", (v,))
        if self.pipe:
            self.pipe.sync()

    def list(
        self,
        config: RunnableConfig | None,
        *,
        filter: dict[str, Any] | None = None,
        before: RunnableConfig | None = None,
        limit: int | None = None,
    ) -> Iterator[CheckpointTuple]:
        """List checkpoints from the database.

        This method retrieves a list of checkpoint tuples from the Postgres database based
        on the provided config. For ShallowPostgresSaver, this method returns a list with
        ONLY the most recent checkpoint.
        """
        where, args = self._search_where(config, filter, before)
        query = self.SELECT_SQL + where
        if limit:
            query += f" LIMIT {limit}"
        with self._cursor() as cur:
            cur.execute(self.SELECT_SQL + where, args, binary=True)
            for value in cur:
                checkpoint: Checkpoint = {
                    **value["checkpoint"],
                    "channel_values": self._load_blobs(value["channel_values"]),
                    "pending_sends": [
                        self.serde.loads_typed((t.decode(), v))
                        for t, v in value["pending_sends"]
                    ]
                    if value["pending_sends"]
                    else [],
                }
                yield CheckpointTuple(
                    config={
                        "configurable": {
                            "thread_id": value["thread_id"],
                            "checkpoint_ns": value["checkpoint_ns"],
                            "checkpoint_id": checkpoint["id"],
                        }
                    },
                    checkpoint=checkpoint,
                    metadata=value["metadata"],
                    pending_writes=self._load_writes(value["pending_writes"]),
                )

    def get_tuple(self, config: RunnableConfig) -> CheckpointTuple | None:
        """Get a checkpoint tuple from the database.

        This method retrieves a checkpoint tuple from the Postgres database based on the
        provided config (matching the thread ID in the config).

        Args:
            config: The config to use for retrieving the checkpoint.

        Returns:
            The retrieved checkpoint tuple, or None if no matching checkpoint was found.

        Examples:

            Basic:
            >>> config = {"configurable": {"thread_id": "1"}}
            >>> checkpoint_tuple = memory.get_tuple(config)
            >>> print(checkpoint_tuple)
            CheckpointTuple(...)

            With timestamp:

            >>> config = {
            ...    "configurable": {
            ...        "thread_id": "1",
            ...        "checkpoint_ns": "",
            ...        "checkpoint_id": "1ef4f797-8335-6428-8001-8a1503f9b875",
            ...    }
            ... }
            >>> checkpoint_tuple = memory.get_tuple(config)
            >>> print(checkpoint_tuple)
            CheckpointTuple(...)
        """  # noqa
        thread_id = config["configurable"]["thread_id"]
        checkpoint_ns = config["configurable"].get("checkpoint_ns", "")
        args = (thread_id, checkpoint_ns)
        where = "WHERE thread_id = %s AND checkpoint_ns = %s"

        with self._cursor() as cur:
            cur.execute(
                self.SELECT_SQL + where,
                args,
                binary=True,
            )

            for value in cur:
                checkpoint: Checkpoint = {
                    **value["checkpoint"],
                    "channel_values": self._load_blobs(value["channel_values"]),
                    "pending_sends": [
                        self.serde.loads_typed((t.decode(), v))
                        for t, v in value["pending_sends"]
                    ]
                    if value["pending_sends"]
                    else [],
                }
                return CheckpointTuple(
                    config={
                        "configurable": {
                            "thread_id": thread_id,
                            "checkpoint_ns": checkpoint_ns,
                            "checkpoint_id": checkpoint["id"],
                        }
                    },
                    checkpoint=checkpoint,
                    metadata=value["metadata"],
                    pending_writes=self._load_writes(value["pending_writes"]),
                )

    def put(
        self,
        config: RunnableConfig,
        checkpoint: Checkpoint,
        metadata: CheckpointMetadata,
        new_versions: ChannelVersions,
    ) -> RunnableConfig:
        """Save a checkpoint to the database.

        This method saves a checkpoint to the Postgres database. The checkpoint is associated
        with the provided config. For ShallowPostgresSaver, this method saves ONLY the most recent
        checkpoint and overwrites a previous checkpoint, if it exists.

        Args:
            config: The config to associate with the checkpoint.
            checkpoint: The checkpoint to save.
            metadata: Additional metadata to save with the checkpoint.
            new_versions: New channel versions as of this write.

        Returns:
            RunnableConfig: Updated configuration after storing the checkpoint.

        Examples:

            >>> from langgraph.checkpoint.postgres import ShallowPostgresSaver
            >>> DB_URI = "postgres://postgres:postgres@localhost:5432/postgres?sslmode=disable"
            >>> with ShallowPostgresSaver.from_conn_string(DB_URI) as memory:
            >>>     config = {"configurable": {"thread_id": "1", "checkpoint_ns": ""}}
            >>>     checkpoint = {"ts": "2024-05-04T06:32:42.235444+00:00", "id": "1ef4f797-8335-6428-8001-8a1503f9b875", "channel_values": {"key": "value"}}
            >>>     saved_config = memory.put(config, checkpoint, {"source": "input", "step": 1, "writes": {"key": "value"}}, {})
            >>> print(saved_config)
            {'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef4f797-8335-6428-8001-8a1503f9b875'}}
        """
        configurable = config["configurable"].copy()
        thread_id = configurable.pop("thread_id")
        checkpoint_ns = configurable.pop("checkpoint_ns")

        copy = checkpoint.copy()
        next_config = {
            "configurable": {
                "thread_id": thread_id,
                "checkpoint_ns": checkpoint_ns,
                "checkpoint_id": checkpoint["id"],
            }
        }

        with self._cursor(pipeline=True) as cur:
            cur.execute(
                """DELETE FROM checkpoint_writes
                WHERE thread_id = %s AND checkpoint_ns = %s AND checkpoint_id NOT IN (%s, %s)""",
                (
                    thread_id,
                    checkpoint_ns,
                    checkpoint["id"],
                    configurable.get("checkpoint_id", ""),
                ),
            )
            cur.executemany(
                self.UPSERT_CHECKPOINT_BLOBS_SQL,
                _dump_blobs(
                    self.serde,
                    thread_id,
                    checkpoint_ns,
                    copy.pop("channel_values"),  # type: ignore[misc]
                    new_versions,
                ),
            )
            cur.execute(
                self.UPSERT_CHECKPOINTS_SQL,
                (
                    thread_id,
                    checkpoint_ns,
                    Jsonb(copy),
                    Jsonb(get_serializable_checkpoint_metadata(config, metadata)),
                ),
            )
        return next_config

    def put_writes(
        self,
        config: RunnableConfig,
        writes: Sequence[tuple[str, Any]],
        task_id: str,
        task_path: str = "",
    ) -> None:
        """Store intermediate writes linked to a checkpoint.

        This method saves intermediate writes associated with a checkpoint to the Postgres database.

        Args:
            config: Configuration of the related checkpoint.
            writes: List of writes to store.
            task_id: Identifier for the task creating the writes.
        """
        query = (
            self.UPSERT_CHECKPOINT_WRITES_SQL
            if all(w[0] in WRITES_IDX_MAP for w in writes)
            else self.INSERT_CHECKPOINT_WRITES_SQL
        )
        with self._cursor(pipeline=True) as cur:
            cur.executemany(
                query,
                self._dump_writes(
                    config["configurable"]["thread_id"],
                    config["configurable"]["checkpoint_ns"],
                    config["configurable"]["checkpoint_id"],
                    task_id,
                    task_path,
                    writes,
                ),
            )

    @contextmanager
    def _cursor(self, *, pipeline: bool = False) -> Iterator[Cursor[DictRow]]:
        """Create a database cursor as a context manager.

        Args:
            pipeline: whether to use pipeline for the DB operations inside the context manager.
                Will be applied regardless of whether the ShallowPostgresSaver instance was initialized with a pipeline.
                If pipeline mode is not supported, will fall back to using transaction context manager.
        """
        with _internal.get_connection(self.conn) as conn:
            if self.pipe:
                # a connection in pipeline mode can be used concurrently
                # in multiple threads/coroutines, but only one cursor can be
                # used at a time
                try:
                    with conn.cursor(binary=True, row_factory=dict_row) as cur:
                        yield cur
                finally:
                    if pipeline:
                        self.pipe.sync()
            elif pipeline:
                # a connection not in pipeline mode can only be used by one
                # thread/coroutine at a time, so we acquire a lock
                if self.supports_pipeline:
                    with (
                        self.lock,
                        conn.pipeline(),
                        conn.cursor(binary=True, row_factory=dict_row) as cur,
                    ):
                        yield cur
                else:
                    # Use connection's transaction context manager when pipeline mode not supported
                    with (
                        self.lock,
                        conn.transaction(),
                        conn.cursor(binary=True, row_factory=dict_row) as cur,
                    ):
                        yield cur
            else:
                with self.lock, conn.cursor(binary=True, row_factory=dict_row) as cur:
                    yield cur


class AsyncShallowPostgresSaver(BasePostgresSaver):
    """A checkpoint saver that uses Postgres to store checkpoints asynchronously.

    This checkpointer ONLY stores the most recent checkpoint and does NOT retain any history.
    It is meant to be a light-weight drop-in replacement for the AsyncPostgresSaver that
    supports most of the LangGraph persistence functionality with the exception of time travel.
    """

    SELECT_SQL = SELECT_SQL
    MIGRATIONS = MIGRATIONS
    UPSERT_CHECKPOINT_BLOBS_SQL = UPSERT_CHECKPOINT_BLOBS_SQL
    UPSERT_CHECKPOINTS_SQL = UPSERT_CHECKPOINTS_SQL
    UPSERT_CHECKPOINT_WRITES_SQL = UPSERT_CHECKPOINT_WRITES_SQL
    INSERT_CHECKPOINT_WRITES_SQL = INSERT_CHECKPOINT_WRITES_SQL
    lock: asyncio.Lock

    def __init__(
        self,
        conn: _ainternal.Conn,
        pipe: AsyncPipeline | None = None,
        serde: SerializerProtocol | None = None,
    ) -> None:
        warnings.warn(
            "AsyncShallowPostgresSaver is deprecated as of version 2.0.20 and will be removed in 3.0.0. "
            "Use AsyncPostgresSaver instead, and invoke the graph with `await graph.ainvoke(..., durability='exit')`.",
            DeprecationWarning,
            stacklevel=2,
        )
        super().__init__(serde=serde)
        if isinstance(conn, AsyncConnectionPool) and pipe is not None:
            raise ValueError(
                "Pipeline should be used only with a single AsyncConnection, not AsyncConnectionPool."
            )

        self.conn = conn
        self.pipe = pipe
        self.lock = asyncio.Lock()
        self.loop = asyncio.get_running_loop()
        self.supports_pipeline = Capabilities().has_pipeline()

    @classmethod
    @asynccontextmanager
    async def from_conn_string(
        cls,
        conn_string: str,
        *,
        pipeline: bool = False,
        serde: SerializerProtocol | None = None,
    ) -> AsyncIterator["AsyncShallowPostgresSaver"]:
        """Create a new AsyncShallowPostgresSaver instance from a connection string.

        Args:
            conn_string: The Postgres connection info string.
            pipeline: whether to use AsyncPipeline

        Returns:
            AsyncShallowPostgresSaver: A new AsyncShallowPostgresSaver instance.
        """
        async with await AsyncConnection.connect(
            conn_string, autocommit=True, prepare_threshold=0, row_factory=dict_row
        ) as conn:
            if pipeline:
                async with conn.pipeline() as pipe:
                    yield cls(conn=conn, pipe=pipe, serde=serde)
            else:
                yield cls(conn=conn, serde=serde)

    async def setup(self) -> None:
        """Set up the checkpoint database asynchronously.

        This method creates the necessary tables in the Postgres database if they don't
        already exist and runs database migrations. It MUST be called directly by the user
        the first time checkpointer is used.
        """
        async with self._cursor() as cur:
            await cur.execute(self.MIGRATIONS[0])
            results = await cur.execute(
                "SELECT v FROM checkpoint_migrations ORDER BY v DESC LIMIT 1"
            )
            row = await results.fetchone()
            if row is None:
                version = -1
            else:
                version = row["v"]
            for v, migration in zip(
                range(version + 1, len(self.MIGRATIONS)),
                self.MIGRATIONS[version + 1 :],
                strict=False,
            ):
                await cur.execute(migration)
                await cur.execute(
                    "INSERT INTO checkpoint_migrations (v) VALUES (%s)", (v,)
                )
        if self.pipe:
            await self.pipe.sync()

    async def alist(
        self,
        config: RunnableConfig | None,
        *,
        filter: dict[str, Any] | None = None,
        before: RunnableConfig | None = None,
        limit: int | None = None,
    ) -> AsyncIterator[CheckpointTuple]:
        """List checkpoints from the database asynchronously.

        This method retrieves a list of checkpoint tuples from the Postgres database based
        on the provided config. For ShallowPostgresSaver, this method returns a list with
        ONLY the most recent checkpoint.
        """
        where, args = self._search_where(config, filter, before)
        query = self.SELECT_SQL + where
        if limit:
            query += f" LIMIT {limit}"
        async with self._cursor() as cur:
            await cur.execute(self.SELECT_SQL + where, args, binary=True)
            async for value in cur:
                checkpoint: Checkpoint = {
                    **value["checkpoint"],
                    "channel_values": self._load_blobs(value["channel_values"]),
                    "pending_sends": [
                        self.serde.loads_typed((t.decode(), v))
                        for t, v in value["pending_sends"]
                    ]
                    if value["pending_sends"]
                    else [],
                }
                yield CheckpointTuple(
                    config={
                        "configurable": {
                            "thread_id": value["thread_id"],
                            "checkpoint_ns": value["checkpoint_ns"],
                            "checkpoint_id": checkpoint["id"],
                        }
                    },
                    checkpoint=checkpoint,
                    metadata=value["metadata"],
                    pending_writes=await asyncio.to_thread(
                        self._load_writes, value["pending_writes"]
                    ),
                )

    async def aget_tuple(self, config: RunnableConfig) -> CheckpointTuple | None:
        """Get a checkpoint tuple from the database asynchronously.

        This method retrieves a checkpoint tuple from the Postgres database based on the
        provided config (matching the thread ID in the config).

        Args:
            config: The config to use for retrieving the checkpoint.

        Returns:
            The retrieved checkpoint tuple, or None if no matching checkpoint was found.
        """
        thread_id = config["configurable"]["thread_id"]
        checkpoint_ns = config["configurable"].get("checkpoint_ns", "")
        args = (thread_id, checkpoint_ns)
        where = "WHERE thread_id = %s AND checkpoint_ns = %s"

        async with self._cursor() as cur:
            await cur.execute(
                self.SELECT_SQL + where,
                args,
                binary=True,
            )

            async for value in cur:
                checkpoint: Checkpoint = {
                    **value["checkpoint"],
                    "channel_values": self._load_blobs(value["channel_values"]),
                    "pending_sends": [
                        self.serde.loads_typed((t.decode(), v))
                        for t, v in value["pending_sends"]
                    ]
                    if value["pending_sends"]
                    else [],
                }
                return CheckpointTuple(
                    config={
                        "configurable": {
                            "thread_id": thread_id,
                            "checkpoint_ns": checkpoint_ns,
                            "checkpoint_id": checkpoint["id"],
                        }
                    },
                    checkpoint=checkpoint,
                    metadata=value["metadata"],
                    pending_writes=await asyncio.to_thread(
                        self._load_writes, value["pending_writes"]
                    ),
                )

    async def aput(
        self,
        config: RunnableConfig,
        checkpoint: Checkpoint,
        metadata: CheckpointMetadata,
        new_versions: ChannelVersions,
    ) -> RunnableConfig:
        """Save a checkpoint to the database asynchronously.

        This method saves a checkpoint to the Postgres database. The checkpoint is associated
        with the provided config. For AsyncShallowPostgresSaver, this method saves ONLY the most recent
        checkpoint and overwrites a previous checkpoint, if it exists.

        Args:
            config: The config to associate with the checkpoint.
            checkpoint: The checkpoint to save.
            metadata: Additional metadata to save with the checkpoint.
            new_versions: New channel versions as of this write.

        Returns:
            RunnableConfig: Updated configuration after storing the checkpoint.
        """
        configurable = config["configurable"].copy()
        thread_id = configurable.pop("thread_id")
        checkpoint_ns = configurable.pop("checkpoint_ns")

        copy = checkpoint.copy()
        next_config = {
            "configurable": {
                "thread_id": thread_id,
                "checkpoint_ns": checkpoint_ns,
                "checkpoint_id": checkpoint["id"],
            }
        }

        async with self._cursor(pipeline=True) as cur:
            await cur.execute(
                """DELETE FROM checkpoint_writes
                WHERE thread_id = %s AND checkpoint_ns = %s AND checkpoint_id NOT IN (%s, %s)""",
                (
                    thread_id,
                    checkpoint_ns,
                    checkpoint["id"],
                    configurable.get("checkpoint_id", ""),
                ),
            )
            await cur.executemany(
                self.UPSERT_CHECKPOINT_BLOBS_SQL,
                _dump_blobs(
                    self.serde,
                    thread_id,
                    checkpoint_ns,
                    copy.pop("channel_values"),  # type: ignore[misc]
                    new_versions,
                ),
            )
            await cur.execute(
                self.UPSERT_CHECKPOINTS_SQL,
                (
                    thread_id,
                    checkpoint_ns,
                    Jsonb(copy),
                    Jsonb(get_serializable_checkpoint_metadata(config, metadata)),
                ),
            )
        return next_config

    async def aput_writes(
        self,
        config: RunnableConfig,
        writes: Sequence[tuple[str, Any]],
        task_id: str,
        task_path: str = "",
    ) -> None:
        """Store intermediate writes linked to a checkpoint asynchronously.

        This method saves intermediate writes associated with a checkpoint to the database.

        Args:
            config: Configuration of the related checkpoint.
            writes: List of writes to store, each as (channel, value) pair.
            task_id: Identifier for the task creating the writes.
        """
        query = (
            self.UPSERT_CHECKPOINT_WRITES_SQL
            if all(w[0] in WRITES_IDX_MAP for w in writes)
            else self.INSERT_CHECKPOINT_WRITES_SQL
        )
        params = await asyncio.to_thread(
            self._dump_writes,
            config["configurable"]["thread_id"],
            config["configurable"]["checkpoint_ns"],
            config["configurable"]["checkpoint_id"],
            task_id,
            task_path,
            writes,
        )
        async with self._cursor(pipeline=True) as cur:
            await cur.executemany(query, params)

    @asynccontextmanager
    async def _cursor(
        self, *, pipeline: bool = False
    ) -> AsyncIterator[AsyncCursor[DictRow]]:
        """Create a database cursor as a context manager.

        Args:
            pipeline: whether to use pipeline for the DB operations inside the context manager.
                Will be applied regardless of whether the AsyncShallowPostgresSaver instance was initialized with a pipeline.
                If pipeline mode is not supported, will fall back to using transaction context manager.
        """
        async with _ainternal.get_connection(self.conn) as conn:
            if self.pipe:
                # a connection in pipeline mode can be used concurrently
                # in multiple threads/coroutines, but only one cursor can be
                # used at a time
                try:
                    async with conn.cursor(binary=True, row_factory=dict_row) as cur:
                        yield cur
                finally:
                    if pipeline:
                        await self.pipe.sync()
            elif pipeline:
                # a connection not in pipeline mode can only be used by one
                # thread/coroutine at a time, so we acquire a lock
                if self.supports_pipeline:
                    async with (
                        self.lock,
                        conn.pipeline(),
                        conn.cursor(binary=True, row_factory=dict_row) as cur,
                    ):
                        yield cur
                else:
                    # Use connection's transaction context manager when pipeline mode not supported
                    async with (
                        self.lock,
                        conn.transaction(),
                        conn.cursor(binary=True, row_factory=dict_row) as cur,
                    ):
                        yield cur
            else:
                async with (
                    self.lock,
                    conn.cursor(binary=True, row_factory=dict_row) as cur,
                ):
                    yield cur

    def list(
        self,
        config: RunnableConfig | None,
        *,
        filter: dict[str, Any] | None = None,
        before: RunnableConfig | None = None,
        limit: int | None = None,
    ) -> Iterator[CheckpointTuple]:
        """List checkpoints from the database.

        This method retrieves a list of checkpoint tuples from the Postgres database based
        on the provided config. For ShallowPostgresSaver, this method returns a list with
        ONLY the most recent checkpoint.
        """
        aiter_ = self.alist(config, filter=filter, before=before, limit=limit)
        while True:
            try:
                yield asyncio.run_coroutine_threadsafe(
                    anext(aiter_),  # type: ignore[arg-type]  # noqa: F821
                    self.loop,
                ).result()
            except StopAsyncIteration:
                break

    def get_tuple(self, config: RunnableConfig) -> CheckpointTuple | None:
        """Get a checkpoint tuple from the database.

        This method retrieves a checkpoint tuple from the Postgres database based on the
        provided config (matching the thread ID in the config).

        Args:
            config: The config to use for retrieving the checkpoint.

        Returns:
            The retrieved checkpoint tuple, or None if no matching checkpoint was found.
        """
        try:
            # check if we are in the main thread, only bg threads can block
            # we don't check in other methods to avoid the overhead
            if asyncio.get_running_loop() is self.loop:
                raise asyncio.InvalidStateError(
                    "Synchronous calls to AsyncShallowPostgresSaver are only allowed from a "
                    "different thread. From the main thread, use the async interface."
                    "For example, use `await checkpointer.aget_tuple(...)` or `await "
                    "graph.ainvoke(...)`."
                )
        except RuntimeError:
            pass
        return asyncio.run_coroutine_threadsafe(
            self.aget_tuple(config), self.loop
        ).result()

    def put(
        self,
        config: RunnableConfig,
        checkpoint: Checkpoint,
        metadata: CheckpointMetadata,
        new_versions: ChannelVersions,
    ) -> RunnableConfig:
        """Save a checkpoint to the database.

        This method saves a checkpoint to the Postgres database. The checkpoint is associated
        with the provided config. For AsyncShallowPostgresSaver, this method saves ONLY the most recent
        checkpoint and overwrites a previous checkpoint, if it exists.

        Args:
            config: The config to associate with the checkpoint.
            checkpoint: The checkpoint to save.
            metadata: Additional metadata to save with the checkpoint.
            new_versions: New channel versions as of this write.

        Returns:
            RunnableConfig: Updated configuration after storing the checkpoint.
        """
        return asyncio.run_coroutine_threadsafe(
            self.aput(config, checkpoint, metadata, new_versions), self.loop
        ).result()

    def put_writes(
        self,
        config: RunnableConfig,
        writes: Sequence[tuple[str, Any]],
        task_id: str,
        task_path: str = "",
    ) -> None:
        """Store intermediate writes linked to a checkpoint.

        This method saves intermediate writes associated with a checkpoint to the database.

        Args:
            config: Configuration of the related checkpoint.
            writes: List of writes to store, each as (channel, value) pair.
            task_id: Identifier for the task creating the writes.
            task_path: Path of the task creating the writes.
        """
        return asyncio.run_coroutine_threadsafe(
            self.aput_writes(config, writes, task_id, task_path), self.loop
        ).result()

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint-postgres/tests/__init__.py
```py

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint-postgres/tests/compose-postgres.yml
```yml
services:
  postgres-test:
    image: pgvector/pgvector:pg${POSTGRES_VERSION:-16}
    ports:
      - "5441:5432"
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    command: ["postgres", "-c", "shared_preload_libraries=vector"]
    healthcheck:
      test: pg_isready -U postgres
      start_period: 10s
      timeout: 1s
      retries: 5
      interval: 60s
      start_interval: 1s

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint-postgres/tests/conftest.py
```py
from collections.abc import AsyncIterator

import pytest
from psycopg import AsyncConnection
from psycopg.errors import UndefinedTable
from psycopg.rows import DictRow, dict_row

from tests.embed_test_utils import CharacterEmbeddings

DEFAULT_POSTGRES_URI = "postgres://postgres:postgres@localhost:5441/"
DEFAULT_URI = "postgres://postgres:postgres@localhost:5441/postgres?sslmode=disable"


@pytest.fixture(scope="function")
async def conn() -> AsyncIterator[AsyncConnection[DictRow]]:
    async with await AsyncConnection.connect(
        DEFAULT_URI, autocommit=True, prepare_threshold=0, row_factory=dict_row
    ) as conn:
        yield conn


@pytest.fixture(scope="function", autouse=True)
async def clear_test_db(conn: AsyncConnection[DictRow]) -> None:
    """Delete all tables before each test."""
    try:
        await conn.execute("DELETE FROM checkpoints")
        await conn.execute("DELETE FROM checkpoint_blobs")
        await conn.execute("DELETE FROM checkpoint_writes")
        await conn.execute("DELETE FROM checkpoint_migrations")
    except UndefinedTable:
        pass
    try:
        await conn.execute("DELETE FROM store_migrations")
        await conn.execute("DELETE FROM store")
    except UndefinedTable:
        pass


@pytest.fixture
def fake_embeddings() -> CharacterEmbeddings:
    return CharacterEmbeddings(dims=500)


VECTOR_TYPES = ["vector", "halfvec"]

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint-postgres/tests/embed_test_utils.py
```py
"""Embedding utilities for testing."""

import math
import random
from collections import Counter, defaultdict
from typing import Any

from langchain_core.embeddings import Embeddings


class CharacterEmbeddings(Embeddings):
    """Simple character-frequency based embeddings using random projections."""

    def __init__(self, dims: int = 50, seed: int = 42):
        """Initialize with embedding dimensions and random seed."""
        self._rng = random.Random(seed)
        self.dims = dims
        # Create projection vector for each character lazily
        self._char_projections: defaultdict[str, list[float]] = defaultdict(
            lambda: [
                self._rng.gauss(0, 1 / math.sqrt(self.dims)) for _ in range(self.dims)
            ]
        )

    def _embed_one(self, text: str) -> list[float]:
        """Embed a single text."""
        counts = Counter(text)
        total = sum(counts.values())

        if total == 0:
            return [0.0] * self.dims

        embedding = [0.0] * self.dims
        for char, count in counts.items():
            weight = count / total
            char_proj = self._char_projections[char]
            for i, proj in enumerate(char_proj):
                embedding[i] += weight * proj

        norm = math.sqrt(sum(x * x for x in embedding))
        if norm > 0:
            embedding = [x / norm for x in embedding]

        return embedding

    def embed_documents(self, texts: list[str]) -> list[list[float]]:
        """Embed a list of documents."""
        return [self._embed_one(text) for text in texts]

    def embed_query(self, text: str) -> list[float]:
        """Embed a query string."""
        return self._embed_one(text)

    def __eq__(self, other: Any) -> bool:
        return isinstance(other, CharacterEmbeddings) and self.dims == other.dims

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint-postgres/tests/test_async.py
```py
# type: ignore

from contextlib import asynccontextmanager
from typing import Any
from uuid import uuid4

import pytest
from langchain_core.runnables import RunnableConfig
from langgraph.checkpoint.base import (
    EXCLUDED_METADATA_KEYS,
    Checkpoint,
    CheckpointMetadata,
    create_checkpoint,
    empty_checkpoint,
)
from langgraph.checkpoint.serde.types import TASKS
from psycopg import AsyncConnection
from psycopg.rows import dict_row
from psycopg_pool import AsyncConnectionPool

from langgraph.checkpoint.postgres.aio import (
    AsyncPostgresSaver,
    AsyncShallowPostgresSaver,
)
from tests.conftest import DEFAULT_POSTGRES_URI


def _exclude_keys(config: dict[str, Any]) -> dict[str, Any]:
    return {k: v for k, v in config.items() if k not in EXCLUDED_METADATA_KEYS}


@asynccontextmanager
async def _pool_saver():
    """Fixture for pool mode testing."""
    database = f"test_{uuid4().hex[:16]}"
    # create unique db
    async with await AsyncConnection.connect(
        DEFAULT_POSTGRES_URI, autocommit=True
    ) as conn:
        await conn.execute(f"CREATE DATABASE {database}")
    try:
        # yield checkpointer
        async with AsyncConnectionPool(
            DEFAULT_POSTGRES_URI + database,
            max_size=10,
            kwargs={"autocommit": True, "row_factory": dict_row},
        ) as pool:
            checkpointer = AsyncPostgresSaver(pool)
            await checkpointer.setup()
            yield checkpointer
    finally:
        # drop unique db
        async with await AsyncConnection.connect(
            DEFAULT_POSTGRES_URI, autocommit=True
        ) as conn:
            await conn.execute(f"DROP DATABASE {database}")


@asynccontextmanager
async def _pipe_saver():
    """Fixture for pipeline mode testing."""
    database = f"test_{uuid4().hex[:16]}"
    # create unique db
    async with await AsyncConnection.connect(
        DEFAULT_POSTGRES_URI, autocommit=True
    ) as conn:
        await conn.execute(f"CREATE DATABASE {database}")
    try:
        async with await AsyncConnection.connect(
            DEFAULT_POSTGRES_URI + database,
            autocommit=True,
            prepare_threshold=0,
            row_factory=dict_row,
        ) as conn:
            checkpointer = AsyncPostgresSaver(conn)
            await checkpointer.setup()
            async with conn.pipeline() as pipe:
                checkpointer = AsyncPostgresSaver(conn, pipe=pipe)
                yield checkpointer
    finally:
        # drop unique db
        async with await AsyncConnection.connect(
            DEFAULT_POSTGRES_URI, autocommit=True
        ) as conn:
            await conn.execute(f"DROP DATABASE {database}")


@asynccontextmanager
async def _base_saver():
    """Fixture for regular connection mode testing."""
    database = f"test_{uuid4().hex[:16]}"
    # create unique db
    async with await AsyncConnection.connect(
        DEFAULT_POSTGRES_URI, autocommit=True
    ) as conn:
        await conn.execute(f"CREATE DATABASE {database}")
    try:
        async with await AsyncConnection.connect(
            DEFAULT_POSTGRES_URI + database,
            autocommit=True,
            prepare_threshold=0,
            row_factory=dict_row,
        ) as conn:
            checkpointer = AsyncPostgresSaver(conn)
            await checkpointer.setup()
            yield checkpointer
    finally:
        # drop unique db
        async with await AsyncConnection.connect(
            DEFAULT_POSTGRES_URI, autocommit=True
        ) as conn:
            await conn.execute(f"DROP DATABASE {database}")


@asynccontextmanager
async def _shallow_saver():
    """Fixture for shallow connection mode testing."""
    database = f"test_{uuid4().hex[:16]}"
    # create unique db
    async with await AsyncConnection.connect(
        DEFAULT_POSTGRES_URI, autocommit=True
    ) as conn:
        await conn.execute(f"CREATE DATABASE {database}")
    try:
        async with await AsyncConnection.connect(
            DEFAULT_POSTGRES_URI + database,
            autocommit=True,
            prepare_threshold=0,
            row_factory=dict_row,
        ) as conn:
            checkpointer = AsyncShallowPostgresSaver(conn)
            await checkpointer.setup()
            yield checkpointer
    finally:
        # drop unique db
        async with await AsyncConnection.connect(
            DEFAULT_POSTGRES_URI, autocommit=True
        ) as conn:
            await conn.execute(f"DROP DATABASE {database}")


@asynccontextmanager
async def _saver(name: str):
    if name == "base":
        async with _base_saver() as saver:
            yield saver
    elif name == "shallow":
        async with _shallow_saver() as saver:
            yield saver
    elif name == "pool":
        async with _pool_saver() as saver:
            yield saver
    elif name == "pipe":
        async with _pipe_saver() as saver:
            yield saver


@pytest.fixture
def test_data():
    """Fixture providing test data for checkpoint tests."""
    config_1: RunnableConfig = {
        "configurable": {
            "thread_id": "thread-1",
            "checkpoint_id": "1",
            "checkpoint_ns": "",
        }
    }
    config_2: RunnableConfig = {
        "configurable": {
            "thread_id": "thread-2",
            "checkpoint_id": "2",
            "checkpoint_ns": "",
        }
    }
    config_3: RunnableConfig = {
        "configurable": {
            "thread_id": "thread-2",
            "checkpoint_id": "2-inner",
            "checkpoint_ns": "inner",
        }
    }

    chkpnt_1: Checkpoint = empty_checkpoint()
    chkpnt_2: Checkpoint = create_checkpoint(chkpnt_1, {}, 1)
    chkpnt_3: Checkpoint = empty_checkpoint()

    metadata_1: CheckpointMetadata = {
        "source": "input",
        "step": 2,
        "score": 1,
    }
    metadata_2: CheckpointMetadata = {
        "source": "loop",
        "step": 1,
        "score": None,
    }
    metadata_3: CheckpointMetadata = {}

    return {
        "configs": [config_1, config_2, config_3],
        "checkpoints": [chkpnt_1, chkpnt_2, chkpnt_3],
        "metadata": [metadata_1, metadata_2, metadata_3],
    }


@pytest.mark.parametrize("saver_name", ["base", "pool", "pipe", "shallow"])
async def test_combined_metadata(saver_name: str, test_data) -> None:
    async with _saver(saver_name) as saver:
        config = {
            "configurable": {
                "thread_id": "thread-2",
                "checkpoint_ns": "",
                "__super_private_key": "super_private_value",
            },
            "metadata": {"run_id": "my_run_id"},
        }
        chkpnt: Checkpoint = create_checkpoint(empty_checkpoint(), {}, 1)
        metadata: CheckpointMetadata = {
            "source": "loop",
            "step": 1,
            "score": None,
        }
        await saver.aput(config, chkpnt, metadata, {})
        checkpoint = await saver.aget_tuple(config)
        assert checkpoint.metadata == {
            **metadata,
            "run_id": "my_run_id",
        }


@pytest.mark.parametrize("saver_name", ["base", "pool", "pipe", "shallow"])
async def test_asearch(saver_name: str, test_data) -> None:
    async with _saver(saver_name) as saver:
        configs = test_data["configs"]
        checkpoints = test_data["checkpoints"]
        metadata = test_data["metadata"]

        await saver.aput(configs[0], checkpoints[0], metadata[0], {})
        await saver.aput(configs[1], checkpoints[1], metadata[1], {})
        await saver.aput(configs[2], checkpoints[2], metadata[2], {})

        # call method / assertions
        query_1 = {"source": "input"}  # search by 1 key
        query_2 = {
            "step": 1,
        }  # search by multiple keys
        query_3: dict[str, Any] = {}  # search by no keys, return all checkpoints
        query_4 = {"source": "update", "step": 1}  # no match

        search_results_1 = [c async for c in saver.alist(None, filter=query_1)]
        assert len(search_results_1) == 1
        assert search_results_1[0].metadata == {
            **_exclude_keys(configs[0]["configurable"]),
            **metadata[0],
        }

        search_results_2 = [c async for c in saver.alist(None, filter=query_2)]
        assert len(search_results_2) == 1
        assert search_results_2[0].metadata == {
            **_exclude_keys(configs[1]["configurable"]),
            **metadata[1],
        }

        search_results_3 = [c async for c in saver.alist(None, filter=query_3)]
        assert len(search_results_3) == 3

        search_results_4 = [c async for c in saver.alist(None, filter=query_4)]
        assert len(search_results_4) == 0

        # search by config (defaults to checkpoints across all namespaces)
        search_results_5 = [
            c async for c in saver.alist({"configurable": {"thread_id": "thread-2"}})
        ]
        assert len(search_results_5) == 2
        assert {
            search_results_5[0].config["configurable"]["checkpoint_ns"],
            search_results_5[1].config["configurable"]["checkpoint_ns"],
        } == {"", "inner"}


@pytest.mark.parametrize("saver_name", ["base", "pool", "pipe", "shallow"])
async def test_null_chars(saver_name: str, test_data) -> None:
    async with _saver(saver_name) as saver:
        config = await saver.aput(
            test_data["configs"][0],
            test_data["checkpoints"][0],
            {"my_key": "\x00abc"},
            {},
        )
        assert (await saver.aget_tuple(config)).metadata["my_key"] == "abc"  # type: ignore
        assert [c async for c in saver.alist(None, filter={"my_key": "abc"})][
            0
        ].metadata["my_key"] == "abc"


@pytest.mark.parametrize("saver_name", ["base", "pool", "pipe"])
async def test_pending_sends_migration(saver_name: str) -> None:
    async with _saver(saver_name) as saver:
        config = {
            "configurable": {
                "thread_id": "thread-1",
                "checkpoint_ns": "",
            }
        }

        # create the first checkpoint
        # and put some pending sends
        checkpoint_0 = empty_checkpoint()
        config = await saver.aput(config, checkpoint_0, {}, {})
        await saver.aput_writes(
            config, [(TASKS, "send-1"), (TASKS, "send-2")], task_id="task-1"
        )
        await saver.aput_writes(config, [(TASKS, "send-3")], task_id="task-2")

        # check that fetching checkpoint_0 doesn't attach pending sends
        # (they should be attached to the next checkpoint)
        tuple_0 = await saver.aget_tuple(config)
        assert tuple_0.checkpoint["channel_values"] == {}
        assert tuple_0.checkpoint["channel_versions"] == {}

        # create the second checkpoint
        checkpoint_1 = create_checkpoint(checkpoint_0, {}, 1)
        config = await saver.aput(config, checkpoint_1, {}, {})

        # check that pending sends are attached to checkpoint_1
        tuple_1 = await saver.aget_tuple(config)
        assert tuple_1.checkpoint["channel_values"] == {
            TASKS: ["send-1", "send-2", "send-3"]
        }
        assert TASKS in tuple_1.checkpoint["channel_versions"]

        # check that list also applies the migration
        search_results = [
            c async for c in saver.alist({"configurable": {"thread_id": "thread-1"}})
        ]
        assert len(search_results) == 2
        assert search_results[-1].checkpoint["channel_values"] == {}
        assert search_results[-1].checkpoint["channel_versions"] == {}
        assert search_results[0].checkpoint["channel_values"] == {
            TASKS: ["send-1", "send-2", "send-3"]
        }
        assert TASKS in search_results[0].checkpoint["channel_versions"]


@pytest.mark.parametrize("saver_name", ["base", "pool", "pipe"])
async def test_get_checkpoint_no_channel_values(
    monkeypatch, saver_name: str, test_data
) -> None:
    """Backwards compatibility test that verifies a checkpoint with no channel_values key can be retrieved without throwing an error."""
    async with _saver(saver_name) as saver:
        config = {
            "configurable": {
                "thread_id": "thread-2",
                "checkpoint_ns": "",
                "__super_private_key": "super_private_value",
            },
            "metadata": {"run_id": "my_run_id"},
        }
        chkpnt: Checkpoint = create_checkpoint(empty_checkpoint(), {}, 1)
        await saver.aput(config, chkpnt, {}, {})

        load_checkpoint_tuple = saver._load_checkpoint_tuple

        def patched_load_checkpoint_tuple(value):
            value["checkpoint"].pop("channel_values", None)
            return load_checkpoint_tuple(value)

        monkeypatch.setattr(
            saver, "_load_checkpoint_tuple", patched_load_checkpoint_tuple
        )

        checkpoint = await saver.aget_tuple(config)
        assert checkpoint.checkpoint["channel_values"] == {}

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint-postgres/tests/test_async_store.py
```py
# type: ignore
from __future__ import annotations

import asyncio
import itertools
import uuid
from collections.abc import AsyncIterator
from concurrent.futures import ThreadPoolExecutor
from contextlib import asynccontextmanager
from typing import Any

import pytest
from langchain_core.embeddings import Embeddings
from langgraph.store.base import (
    GetOp,
    Item,
    ListNamespacesOp,
    PutOp,
    SearchOp,
)
from psycopg import AsyncConnection

from langgraph.store.postgres import AsyncPostgresStore
from tests.conftest import (
    DEFAULT_URI,
    VECTOR_TYPES,
    CharacterEmbeddings,
)

TTL_SECONDS = 6
TTL_MINUTES = TTL_SECONDS / 60


@pytest.fixture(scope="function", params=["default", "pipe", "pool"])
async def store(request) -> AsyncIterator[AsyncPostgresStore]:
    database = f"test_{uuid.uuid4().hex[:16]}"
    uri_parts = DEFAULT_URI.split("/")
    uri_base = "/".join(uri_parts[:-1])
    query_params = ""
    if "?" in uri_parts[-1]:
        db_name, query_params = uri_parts[-1].split("?", 1)
        query_params = "?" + query_params

    conn_string = f"{uri_base}/{database}{query_params}"
    admin_conn_string = DEFAULT_URI
    ttl_config = {
        "default_ttl": TTL_MINUTES,
        "refresh_on_read": True,
        "sweep_interval_minutes": TTL_MINUTES / 2,
    }
    async with await AsyncConnection.connect(
        admin_conn_string, autocommit=True
    ) as conn:
        await conn.execute(f"CREATE DATABASE {database}")
    try:
        async with AsyncPostgresStore.from_conn_string(
            conn_string, ttl=ttl_config
        ) as store:
            store.MIGRATIONS = [
                (
                    mig.replace("ttl_minutes INT;", "ttl_minutes FLOAT;")
                    if isinstance(mig, str)
                    else mig
                )
                for mig in store.MIGRATIONS
            ]
            await store.setup()
            async with store._cursor() as cur:
                # drop the migration index
                await cur.execute("DROP TABLE IF EXISTS store_migrations")
            await store.setup()  # Will fail if migrations aren't idempotent

        if request.param == "pipe":
            async with AsyncPostgresStore.from_conn_string(
                conn_string, pipeline=True, ttl=ttl_config
            ) as store:
                await store.start_ttl_sweeper()
                yield store
                await store.stop_ttl_sweeper()
        elif request.param == "pool":
            async with AsyncPostgresStore.from_conn_string(
                conn_string, pool_config={"min_size": 1, "max_size": 10}, ttl=ttl_config
            ) as store:
                await store.start_ttl_sweeper()
                yield store
                await store.stop_ttl_sweeper()
        else:  # default
            async with AsyncPostgresStore.from_conn_string(
                conn_string, ttl=ttl_config
            ) as store:
                await store.start_ttl_sweeper()
                yield store
                await store.stop_ttl_sweeper()
    finally:
        async with await AsyncConnection.connect(
            admin_conn_string, autocommit=True
        ) as conn:
            await conn.execute(f"DROP DATABASE {database}")


async def test_no_running_loop(store: AsyncPostgresStore) -> None:
    with pytest.raises(asyncio.InvalidStateError):
        store.put(("foo", "bar"), "baz", {"val": "baz"})
    with pytest.raises(asyncio.InvalidStateError):
        store.get(("foo", "bar"), "baz")
    with pytest.raises(asyncio.InvalidStateError):
        store.delete(("foo", "bar"), "baz")
    with pytest.raises(asyncio.InvalidStateError):
        store.search(("foo", "bar"))
    with pytest.raises(asyncio.InvalidStateError):
        store.list_namespaces(prefix=("foo",))
    with pytest.raises(asyncio.InvalidStateError):
        store.batch([PutOp(namespace=("foo", "bar"), key="baz", value={"val": "baz"})])
    with ThreadPoolExecutor(max_workers=1) as executor:
        future = executor.submit(store.put, ("foo", "bar"), "baz", {"val": "baz"})
        result = await asyncio.wrap_future(future)
        assert result is None
        future = executor.submit(store.get, ("foo", "bar"), "baz")
        result = await asyncio.wrap_future(future)
        assert result.value == {"val": "baz"}
        result = await asyncio.wrap_future(
            executor.submit(store.list_namespaces, prefix=("foo",))
        )


async def test_large_batches(request: Any, store: AsyncPostgresStore) -> None:
    N = 100  # less important that we are performant here
    M = 10

    with ThreadPoolExecutor(max_workers=10) as executor:
        futures = []
        for m in range(M):
            for i in range(N):
                futures += [
                    executor.submit(
                        store.put,
                        ("test", "foo", "bar", "baz", str(m % 2)),
                        f"key{i}",
                        value={"foo": "bar" + str(i)},
                    ),
                    executor.submit(
                        store.get,
                        ("test", "foo", "bar", "baz", str(m % 2)),
                        f"key{i}",
                    ),
                    executor.submit(
                        store.list_namespaces,
                        prefix=None,
                        max_depth=m + 1,
                    ),
                    executor.submit(
                        store.search,
                        ("test",),
                    ),
                    executor.submit(
                        store.put,
                        ("test", "foo", "bar", "baz", str(m % 2)),
                        f"key{i}",
                        value={"foo": "bar" + str(i)},
                    ),
                    executor.submit(
                        store.put,
                        ("test", "foo", "bar", "baz", str(m % 2)),
                        f"key{i}",
                        None,
                    ),
                ]

        results = await asyncio.gather(
            *(asyncio.wrap_future(future) for future in futures)
        )
    assert len(results) == M * N * 6


async def test_large_batches_async(store: AsyncPostgresStore) -> None:
    N = 1000
    M = 10
    coros = []
    for m in range(M):
        for i in range(N):
            coros.append(
                store.aput(
                    ("test", "foo", "bar", "baz", str(m % 2)),
                    f"key{i}",
                    value={"foo": "bar" + str(i)},
                )
            )
            coros.append(
                store.aget(
                    ("test", "foo", "bar", "baz", str(m % 2)),
                    f"key{i}",
                )
            )
            coros.append(
                store.alist_namespaces(
                    prefix=None,
                    max_depth=m + 1,
                )
            )
            coros.append(
                store.asearch(
                    ("test",),
                )
            )
            coros.append(
                store.aput(
                    ("test", "foo", "bar", "baz", str(m % 2)),
                    f"key{i}",
                    value={"foo": "bar" + str(i)},
                )
            )
            coros.append(
                store.adelete(
                    ("test", "foo", "bar", "baz", str(m % 2)),
                    f"key{i}",
                )
            )

    results = await asyncio.gather(*coros)
    assert len(results) == M * N * 6


async def test_abatch_order(store: AsyncPostgresStore) -> None:
    # Setup test data
    await store.aput(("test", "foo"), "key1", {"data": "value1"})
    await store.aput(("test", "bar"), "key2", {"data": "value2"})

    ops = [
        GetOp(namespace=("test", "foo"), key="key1"),
        PutOp(namespace=("test", "bar"), key="key2", value={"data": "value2"}),
        SearchOp(
            namespace_prefix=("test",), filter={"data": "value1"}, limit=10, offset=0
        ),
        ListNamespacesOp(match_conditions=None, max_depth=None, limit=10, offset=0),
        GetOp(namespace=("test",), key="key3"),
    ]

    results = await store.abatch(ops)
    assert len(results) == 5
    assert isinstance(results[0], Item)
    assert isinstance(results[0].value, dict)
    assert results[0].value == {"data": "value1"}
    assert results[0].key == "key1"
    assert results[1] is None
    assert isinstance(results[2], list)
    assert len(results[2]) == 1
    assert isinstance(results[3], list)
    assert ("test", "foo") in results[3] and ("test", "bar") in results[3]
    assert results[4] is None

    ops_reordered = [
        SearchOp(namespace_prefix=("test",), filter=None, limit=5, offset=0),
        GetOp(namespace=("test", "bar"), key="key2"),
        ListNamespacesOp(match_conditions=None, max_depth=None, limit=5, offset=0),
        PutOp(namespace=("test",), key="key3", value={"data": "value3"}),
        GetOp(namespace=("test", "foo"), key="key1"),
    ]

    results_reordered = await store.abatch(ops_reordered)
    assert len(results_reordered) == 5
    assert isinstance(results_reordered[0], list)
    assert len(results_reordered[0]) == 2
    assert isinstance(results_reordered[1], Item)
    assert results_reordered[1].value == {"data": "value2"}
    assert results_reordered[1].key == "key2"
    assert isinstance(results_reordered[2], list)
    assert ("test", "foo") in results_reordered[2] and (
        "test",
        "bar",
    ) in results_reordered[2]
    assert results_reordered[3] is None
    assert isinstance(results_reordered[4], Item)
    assert results_reordered[4].value == {"data": "value1"}
    assert results_reordered[4].key == "key1"


async def test_batch_get_ops(store: AsyncPostgresStore) -> None:
    # Setup test data
    await store.aput(("test",), "key1", {"data": "value1"})
    await store.aput(("test",), "key2", {"data": "value2"})

    ops = [
        GetOp(namespace=("test",), key="key1"),
        GetOp(namespace=("test",), key="key2"),
        GetOp(namespace=("test",), key="key3"),
    ]

    results = await store.abatch(ops)

    assert len(results) == 3
    assert results[0] is not None
    assert results[1] is not None
    assert results[2] is None
    assert results[0].key == "key1"
    assert results[1].key == "key2"


async def test_batch_put_ops(store: AsyncPostgresStore) -> None:
    ops = [
        PutOp(namespace=("test",), key="key1", value={"data": "value1"}),
        PutOp(namespace=("test",), key="key2", value={"data": "value2"}),
        PutOp(namespace=("test",), key="key3", value=None),
    ]

    results = await store.abatch(ops)

    assert len(results) == 3
    assert all(result is None for result in results)

    # Verify the puts worked
    items = await store.asearch(["test"], limit=10)
    assert len(items) == 2  # key3 had None value so wasn't stored


async def test_batch_search_ops(store: AsyncPostgresStore) -> None:
    # Setup test data
    await store.aput(("test", "foo"), "key1", {"data": "value1"})
    await store.aput(("test", "bar"), "key2", {"data": "value2"})

    ops = [
        SearchOp(
            namespace_prefix=("test",), filter={"data": "value1"}, limit=10, offset=0
        ),
        SearchOp(namespace_prefix=("test",), filter=None, limit=5, offset=0),
    ]

    results = await store.abatch(ops)

    assert len(results) == 2
    assert len(results[0]) == 1  # Filtered results
    assert len(results[1]) == 2  # All results


async def test_batch_list_namespaces_ops(store: AsyncPostgresStore) -> None:
    # Setup test data
    await store.aput(("test", "namespace1"), "key1", {"data": "value1"})
    await store.aput(("test", "namespace2"), "key2", {"data": "value2"})

    ops = [ListNamespacesOp(match_conditions=None, max_depth=None, limit=10, offset=0)]

    results = await store.abatch(ops)

    assert len(results) == 1
    assert len(results[0]) == 2
    assert ("test", "namespace1") in results[0]
    assert ("test", "namespace2") in results[0]


@asynccontextmanager
async def _create_vector_store(
    vector_type: str,
    distance_type: str,
    fake_embeddings: CharacterEmbeddings,
    text_fields: list[str] | None = None,
) -> AsyncIterator[AsyncPostgresStore]:
    """Create a store with vector search enabled."""

    database = f"test_{uuid.uuid4().hex[:16]}"
    uri_parts = DEFAULT_URI.split("/")
    uri_base = "/".join(uri_parts[:-1])
    query_params = ""
    if "?" in uri_parts[-1]:
        db_name, query_params = uri_parts[-1].split("?", 1)
        query_params = "?" + query_params

    conn_string = f"{uri_base}/{database}{query_params}"
    admin_conn_string = DEFAULT_URI

    index_config = {
        "dims": fake_embeddings.dims,
        "embed": fake_embeddings,
        "ann_index_config": {
            "vector_type": vector_type,
        },
        "distance_type": distance_type,
        "fields": text_fields,
    }

    async with await AsyncConnection.connect(
        admin_conn_string, autocommit=True
    ) as conn:
        await conn.execute(f"CREATE DATABASE {database}")
    try:
        async with AsyncPostgresStore.from_conn_string(
            conn_string,
            index=index_config,
        ) as store:
            await store.setup()
            yield store
    finally:
        async with await AsyncConnection.connect(
            admin_conn_string, autocommit=True
        ) as conn:
            await conn.execute(f"DROP DATABASE {database}")


@pytest.fixture(
    scope="function",
    params=[
        (vector_type, distance_type)
        for vector_type in VECTOR_TYPES
        for distance_type in (
            ["hamming"] if vector_type == "bit" else ["l2", "inner_product", "cosine"]
        )
    ],
    ids=lambda p: f"{p[0]}_{p[1]}",
)
async def vector_store(
    request,
    fake_embeddings: CharacterEmbeddings,
) -> AsyncIterator[AsyncPostgresStore]:
    """Create a store with vector search enabled."""
    vector_type, distance_type = request.param
    async with _create_vector_store(
        vector_type, distance_type, fake_embeddings
    ) as store:
        yield store


async def test_vector_store_initialization(
    vector_store: AsyncPostgresStore, fake_embeddings: CharacterEmbeddings
) -> None:
    """Test store initialization with embedding config."""
    assert vector_store.index_config is not None
    assert vector_store.index_config["dims"] == fake_embeddings.dims
    if isinstance(vector_store.index_config["embed"], Embeddings):
        assert vector_store.index_config["embed"] == fake_embeddings


async def test_vector_insert_with_auto_embedding(
    vector_store: AsyncPostgresStore,
) -> None:
    """Test inserting items that get auto-embedded."""
    docs = [
        ("doc1", {"text": "short text"}),
        ("doc2", {"text": "longer text document"}),
        ("doc3", {"text": "longest text document here"}),
        ("doc4", {"description": "text in description field"}),
        ("doc5", {"content": "text in content field"}),
        ("doc6", {"body": "text in body field"}),
    ]

    for key, value in docs:
        await vector_store.aput(("test",), key, value)

    results = await vector_store.asearch(("test",), query="long text")
    assert len(results) > 0

    doc_order = [r.key for r in results]
    assert "doc2" in doc_order
    assert "doc3" in doc_order


async def test_vector_update_with_embedding(vector_store: AsyncPostgresStore) -> None:
    """Test that updating items properly updates their embeddings."""
    await vector_store.aput(("test",), "doc1", {"text": "zany zebra Xerxes"})
    await vector_store.aput(("test",), "doc2", {"text": "something about dogs"})
    await vector_store.aput(("test",), "doc3", {"text": "text about birds"})

    results_initial = await vector_store.asearch(("test",), query="Zany Xerxes")
    assert len(results_initial) > 0
    assert results_initial[0].key == "doc1"
    initial_score = results_initial[0].score

    await vector_store.aput(("test",), "doc1", {"text": "new text about dogs"})

    results_after = await vector_store.asearch(("test",), query="Zany Xerxes")
    after_score = next((r.score for r in results_after if r.key == "doc1"), 0.0)
    assert after_score < initial_score

    results_new = await vector_store.asearch(("test",), query="new text about dogs")
    for r in results_new:
        if r.key == "doc1":
            assert r.score > after_score

    # Don't index this one
    await vector_store.aput(
        ("test",), "doc4", {"text": "new text about dogs"}, index=False
    )
    results_new = await vector_store.asearch(
        ("test",), query="new text about dogs", limit=3
    )
    assert not any(r.key == "doc4" for r in results_new)


async def test_vector_search_with_filters(vector_store: AsyncPostgresStore) -> None:
    """Test combining vector search with filters."""
    docs = [
        ("doc1", {"text": "red apple", "color": "red", "score": 4.5}),
        ("doc2", {"text": "red car", "color": "red", "score": 3.0}),
        ("doc3", {"text": "green apple", "color": "green", "score": 4.0}),
        ("doc4", {"text": "blue car", "color": "blue", "score": 3.5}),
    ]

    for key, value in docs:
        await vector_store.aput(("test",), key, value)

    results = await vector_store.asearch(
        ("test",), query="apple", filter={"color": "red"}
    )
    assert len(results) == 2
    assert results[0].key == "doc1"

    results = await vector_store.asearch(
        ("test",), query="car", filter={"color": "red"}
    )
    assert len(results) == 2
    assert results[0].key == "doc2"

    results = await vector_store.asearch(
        ("test",), query="bbbbluuu", filter={"score": {"$gt": 3.2}}
    )
    assert len(results) == 3
    assert results[0].key == "doc4"

    results = await vector_store.asearch(
        ("test",), query="apple", filter={"score": {"$gte": 4.0}, "color": "green"}
    )
    assert len(results) == 1
    assert results[0].key == "doc3"


async def test_vector_search_pagination(vector_store: AsyncPostgresStore) -> None:
    """Test pagination with vector search."""
    for i in range(5):
        await vector_store.aput(
            ("test",), f"doc{i}", {"text": f"test document number {i}"}
        )

    results_page1 = await vector_store.asearch(("test",), query="test", limit=2)
    results_page2 = await vector_store.asearch(
        ("test",), query="test", limit=2, offset=2
    )

    assert len(results_page1) == 2
    assert len(results_page2) == 2
    assert results_page1[0].key != results_page2[0].key

    all_results = await vector_store.asearch(("test",), query="test", limit=10)
    assert len(all_results) == 5


async def test_vector_search_edge_cases(vector_store: AsyncPostgresStore) -> None:
    """Test edge cases in vector search."""
    await vector_store.aput(("test",), "doc1", {"text": "test document"})

    perfect_match = await vector_store.asearch(("test",), query="text test document")
    perfect_score = perfect_match[0].score

    results = await vector_store.asearch(("test",), query="")
    assert len(results) == 1
    assert results[0].score is None

    results = await vector_store.asearch(("test",), query=None)
    assert len(results) == 1
    assert results[0].score is None

    long_query = "foo " * 100
    results = await vector_store.asearch(("test",), query=long_query)
    assert len(results) == 1
    assert results[0].score < perfect_score

    special_query = "test!@#$%^&*()"
    results = await vector_store.asearch(("test",), query=special_query)
    assert len(results) == 1
    assert results[0].score < perfect_score


@pytest.mark.parametrize(
    "vector_type,distance_type",
    [
        *itertools.product(["vector", "halfvec"], ["cosine", "inner_product", "l2"]),
    ],
)
async def test_embed_with_path(
    request: Any,
    fake_embeddings: CharacterEmbeddings,
    vector_type: str,
    distance_type: str,
) -> None:
    """Test vector search with specific text fields in Postgres store."""
    async with _create_vector_store(
        vector_type,
        distance_type,
        fake_embeddings,
        text_fields=["key0", "key1", "key3"],
    ) as store:
        # This will have 2 vectors representing it
        doc1 = {
            # Omit key0 - check it doesn't raise an error
            "key1": "xxx",
            "key2": "yyy",
            "key3": "zzz",
        }
        # This will have 3 vectors representing it
        doc2 = {
            "key0": "uuu",
            "key1": "vvv",
            "key2": "www",
            "key3": "xxx",
        }
        await store.aput(("test",), "doc1", doc1)
        await store.aput(("test",), "doc2", doc2)

        # doc2.key3 and doc1.key1 both would have the highest score
        results = await store.asearch(("test",), query="xxx")
        assert len(results) == 2
        assert results[0].key != results[1].key
        ascore = results[0].score
        bscore = results[1].score
        assert ascore == pytest.approx(bscore, abs=1e-3)

        results = await store.asearch(("test",), query="uuu")
        assert len(results) == 2
        assert results[0].key != results[1].key
        assert results[0].key == "doc2"
        assert results[0].score > results[1].score
        assert ascore == pytest.approx(results[0].score, abs=1e-3)

        # Un-indexed - will have low results for both. Not zero (because we're projecting)
        # but less than the above.
        results = await store.asearch(("test",), query="www")
        assert len(results) == 2
        assert results[0].score < ascore
        assert results[1].score < ascore


@pytest.mark.parametrize(
    "vector_type,distance_type",
    [
        *itertools.product(["vector", "halfvec"], ["cosine", "inner_product", "l2"]),
    ],
)
async def test_search_sorting(
    request: Any,
    fake_embeddings: CharacterEmbeddings,
    vector_type: str,
    distance_type: str,
) -> None:
    """Test operation-level field configuration for vector search."""
    async with _create_vector_store(
        vector_type,
        distance_type,
        fake_embeddings,
        text_fields=["key1"],  # Default fields that won't match our test data
    ) as store:
        amatch = {
            "key1": "mmm",
        }

        await store.aput(("test", "M"), "M", amatch)
        N = 100
        for i in range(N):
            await store.aput(("test", "A"), f"A{i}", {"key1": "no"})
        for i in range(N):
            await store.aput(("test", "Z"), f"Z{i}", {"key1": "no"})

        results = await store.asearch(("test",), query="mmm", limit=10)
        assert len(results) == 10
        assert len(set(r.key for r in results)) == 10
        assert results[0].key == "M"
        assert results[0].score > results[1].score


async def test_store_ttl(store):
    # Assumes a TTL of 1 minute = 60 seconds
    ns = ("foo",)
    await store.start_ttl_sweeper()
    await store.aput(
        ns,
        key="item1",
        value={"foo": "bar"},
        ttl=TTL_MINUTES,  # type: ignore
    )
    await asyncio.sleep(TTL_SECONDS - 2)
    res = await store.aget(ns, key="item1", refresh_ttl=True)
    assert res is not None
    await asyncio.sleep(TTL_SECONDS - 2)
    results = await store.asearch(ns, query="foo", refresh_ttl=True)
    assert len(results) == 1
    await asyncio.sleep(TTL_SECONDS - 2)
    res = await store.aget(ns, key="item1", refresh_ttl=False)
    assert res is not None
    await asyncio.sleep(TTL_SECONDS - 1)
    # Now has been (TTL_SECONDS-2)*2 > TTL_SECONDS + TTL_SECONDS/2
    results = await store.asearch(ns, query="bar", refresh_ttl=False)
    assert len(results) == 0

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint-postgres/tests/test_store.py
```py
# type: ignore
from __future__ import annotations

import re
import time
from contextlib import contextmanager
from typing import Any
from uuid import uuid4

import pytest
from langchain_core.embeddings import Embeddings
from langgraph.store.base import (
    GetOp,
    Item,
    ListNamespacesOp,
    MatchCondition,
    PutOp,
    SearchOp,
)
from psycopg import Connection

from langgraph.store.postgres import PostgresStore
from tests.conftest import (
    DEFAULT_URI,
    VECTOR_TYPES,
    CharacterEmbeddings,
)

TTL_SECONDS = 6
TTL_MINUTES = TTL_SECONDS / 60


@pytest.fixture(scope="function", params=["default", "pipe", "pool"])
def store(request) -> PostgresStore:
    database = f"test_{uuid4().hex[:16]}"
    uri_parts = DEFAULT_URI.split("/")
    uri_base = "/".join(uri_parts[:-1])
    query_params = ""
    if "?" in uri_parts[-1]:
        _, query_params = uri_parts[-1].split("?", 1)
        query_params = "?" + query_params

    conn_string = f"{uri_base}/{database}{query_params}"
    admin_conn_string = DEFAULT_URI
    ttl_config = {
        "default_ttl": TTL_MINUTES,
        "refresh_on_read": True,
        "sweep_interval_minutes": TTL_MINUTES / 2,
    }
    with Connection.connect(admin_conn_string, autocommit=True) as conn:
        conn.execute(f"CREATE DATABASE {database}")
    try:
        with PostgresStore.from_conn_string(conn_string, ttl=ttl_config) as store:
            store.MIGRATIONS = [
                (
                    mig.replace("ttl_minutes INT;", "ttl_minutes FLOAT;")
                    if isinstance(mig, str)
                    else mig
                )
                for mig in store.MIGRATIONS
            ]
            store.setup()

        if request.param == "pipe":
            with PostgresStore.from_conn_string(
                conn_string,
                pipeline=True,
                ttl=ttl_config,
            ) as store:
                store.start_ttl_sweeper()
                yield store

                store.stop_ttl_sweeper()
        elif request.param == "pool":
            with PostgresStore.from_conn_string(
                conn_string,
                pool_config={"min_size": 1, "max_size": 10},
                ttl=ttl_config,
            ) as store:
                store.start_ttl_sweeper()
                yield store

                store.stop_ttl_sweeper()
        else:  # default
            with PostgresStore.from_conn_string(conn_string, ttl=ttl_config) as store:
                store.start_ttl_sweeper()
                yield store

                store.stop_ttl_sweeper()
    finally:
        with Connection.connect(admin_conn_string, autocommit=True) as conn:
            conn.execute(f"DROP DATABASE {database}")


def test_batch_order(store: PostgresStore) -> None:
    # Setup test data
    store.put(("test", "foo"), "key1", {"data": "value1"})
    store.put(("test", "bar"), "key2", {"data": "value2"})

    ops = [
        GetOp(namespace=("test", "foo"), key="key1"),
        PutOp(namespace=("test", "bar"), key="key2", value={"data": "value2"}),
        SearchOp(
            namespace_prefix=("test",), filter={"data": "value1"}, limit=10, offset=0
        ),
        ListNamespacesOp(match_conditions=None, max_depth=None, limit=10, offset=0),
        GetOp(namespace=("test",), key="key3"),
    ]

    results = store.batch(ops)
    assert len(results) == 5
    assert isinstance(results[0], Item)
    assert isinstance(results[0].value, dict)
    assert results[0].value == {"data": "value1"}
    assert results[0].key == "key1"
    assert results[1] is None  # Put operation returns None
    assert isinstance(results[2], list)
    assert len(results[2]) == 1
    assert isinstance(results[3], list)
    assert len(results[3]) > 0  # Should contain at least our test namespaces
    assert results[4] is None  # Non-existent key returns None

    # Test reordered operations
    ops_reordered = [
        SearchOp(namespace_prefix=("test",), filter=None, limit=5, offset=0),
        GetOp(namespace=("test", "bar"), key="key2"),
        ListNamespacesOp(match_conditions=None, max_depth=None, limit=5, offset=0),
        PutOp(namespace=("test",), key="key3", value={"data": "value3"}),
        GetOp(namespace=("test", "foo"), key="key1"),
    ]

    results_reordered = store.batch(ops_reordered)
    assert len(results_reordered) == 5
    assert isinstance(results_reordered[0], list)
    assert len(results_reordered[0]) >= 2  # Should find at least our two test items
    assert isinstance(results_reordered[1], Item)
    assert results_reordered[1].value == {"data": "value2"}
    assert results_reordered[1].key == "key2"
    assert isinstance(results_reordered[2], list)
    assert len(results_reordered[2]) > 0
    assert results_reordered[3] is None  # Put operation returns None
    assert isinstance(results_reordered[4], Item)
    assert results_reordered[4].value == {"data": "value1"}
    assert results_reordered[4].key == "key1"


def test_batch_get_ops(store: PostgresStore) -> None:
    # Setup test data
    store.put(("test",), "key1", {"data": "value1"})
    store.put(("test",), "key2", {"data": "value2"})

    ops = [
        GetOp(namespace=("test",), key="key1"),
        GetOp(namespace=("test",), key="key2"),
        GetOp(namespace=("test",), key="key3"),  # Non-existent key
    ]

    results = store.batch(ops)

    assert len(results) == 3
    assert results[0] is not None
    assert results[1] is not None
    assert results[2] is None
    assert results[0].key == "key1"
    assert results[1].key == "key2"


def test_batch_put_ops(store: PostgresStore) -> None:
    ops = [
        PutOp(namespace=("test",), key="key1", value={"data": "value1"}),
        PutOp(namespace=("test",), key="key2", value={"data": "value2"}),
        PutOp(namespace=("test",), key="key3", value=None),  # Delete operation
    ]

    results = store.batch(ops)
    assert len(results) == 3
    assert all(result is None for result in results)

    # Verify the puts worked
    item1 = store.get(("test",), "key1")
    item2 = store.get(("test",), "key2")
    item3 = store.get(("test",), "key3")

    assert item1 and item1.value == {"data": "value1"}
    assert item2 and item2.value == {"data": "value2"}
    assert item3 is None


def test_batch_search_ops(store: PostgresStore) -> None:
    # Setup test data
    test_data = [
        (("test", "foo"), "key1", {"data": "value1", "tag": "a"}),
        (("test", "bar"), "key2", {"data": "value2", "tag": "a"}),
        (("test", "baz"), "key3", {"data": "value3", "tag": "b"}),
    ]
    for namespace, key, value in test_data:
        store.put(namespace, key, value)

    ops = [
        SearchOp(namespace_prefix=("test",), filter={"tag": "a"}, limit=10, offset=0),
        SearchOp(namespace_prefix=("test",), filter=None, limit=2, offset=0),
        SearchOp(namespace_prefix=("test", "foo"), filter=None, limit=10, offset=0),
    ]

    results = store.batch(ops)
    assert len(results) == 3

    # First search should find items with tag "a"
    assert len(results[0]) == 2
    assert all(item.value["tag"] == "a" for item in results[0])

    # Second search should return first 2 items
    assert len(results[1]) == 2

    # Third search should only find items in test/foo namespace
    assert len(results[2]) == 1
    assert results[2][0].namespace == ("test", "foo")


def test_batch_list_namespaces_ops(store: PostgresStore) -> None:
    # Setup test data with various namespaces
    test_data = [
        (("test", "documents", "public"), "doc1", {"content": "public doc"}),
        (("test", "documents", "private"), "doc2", {"content": "private doc"}),
        (("test", "images", "public"), "img1", {"content": "public image"}),
        (("prod", "documents", "public"), "doc3", {"content": "prod doc"}),
    ]
    for namespace, key, value in test_data:
        store.put(namespace, key, value)

    ops = [
        ListNamespacesOp(match_conditions=None, max_depth=None, limit=10, offset=0),
        ListNamespacesOp(match_conditions=None, max_depth=2, limit=10, offset=0),
        ListNamespacesOp(
            match_conditions=[MatchCondition("suffix", "public")],
            max_depth=None,
            limit=10,
            offset=0,
        ),
    ]

    results = store.batch(ops)
    assert len(results) == 3

    # First operation should list all namespaces
    assert len(results[0]) == len(test_data)

    # Second operation should only return namespaces up to depth 2
    assert all(len(ns) <= 2 for ns in results[1])

    # Third operation should only return namespaces ending with "public"
    assert all(ns[-1] == "public" for ns in results[2])


def test_basic_store_ops(store) -> None:
    namespace = ("test", "documents")
    item_id = "doc1"
    item_value = {"title": "Test Document", "content": "Hello, World!"}

    store.put(namespace, item_id, item_value)
    item = store.get(namespace, item_id)

    assert item
    assert item.namespace == namespace
    assert item.key == item_id
    assert item.value == item_value

    # Test update
    updated_value = {"title": "Updated Document", "content": "Hello, Updated!"}
    store.put(namespace, item_id, updated_value)
    updated_item = store.get(namespace, item_id)

    assert updated_item.value == updated_value
    assert updated_item.updated_at > item.updated_at

    # Test get from non-existent namespace
    different_namespace = ("test", "other_documents")
    item_in_different_namespace = store.get(different_namespace, item_id)
    assert item_in_different_namespace is None

    # Test delete
    store.delete(namespace, item_id)
    deleted_item = store.get(namespace, item_id)
    assert deleted_item is None


def test_list_namespaces(store) -> None:
    # Create test data with various namespaces
    test_namespaces = [
        ("test", "documents", "public"),
        ("test", "documents", "private"),
        ("test", "images", "public"),
        ("test", "images", "private"),
        ("prod", "documents", "public"),
        ("prod", "documents", "private"),
    ]

    # Insert test data
    for namespace in test_namespaces:
        store.put(namespace, "dummy", {"content": "dummy"})

    # Test listing with various filters
    all_namespaces = store.list_namespaces()
    assert len(all_namespaces) == len(test_namespaces)

    # Test prefix filtering
    test_prefix_namespaces = store.list_namespaces(prefix=["test"])
    assert len(test_prefix_namespaces) == 4
    assert all(ns[0] == "test" for ns in test_prefix_namespaces)

    # Test suffix filtering
    public_namespaces = store.list_namespaces(suffix=["public"])
    assert len(public_namespaces) == 3
    assert all(ns[-1] == "public" for ns in public_namespaces)

    # Test max depth
    depth_2_namespaces = store.list_namespaces(max_depth=2)
    assert all(len(ns) <= 2 for ns in depth_2_namespaces)

    # Test pagination
    paginated_namespaces = store.list_namespaces(limit=3)
    assert len(paginated_namespaces) == 3

    # Cleanup
    for namespace in test_namespaces:
        store.delete(namespace, "dummy")


def test_search(store) -> None:
    # Create test data
    test_data = [
        (
            ("test", "docs"),
            "doc1",
            {"title": "First Doc", "author": "Alice", "tags": ["important"]},
        ),
        (
            ("test", "docs"),
            "doc2",
            {"title": "Second Doc", "author": "Bob", "tags": ["draft"]},
        ),
        (
            ("test", "images"),
            "img1",
            {"title": "Image 1", "author": "Alice", "tags": ["final"]},
        ),
    ]

    for namespace, key, value in test_data:
        store.put(namespace, key, value)

    # Test basic search
    all_items = store.search(["test"])
    assert len(all_items) == 3

    # Test namespace filtering
    docs_items = store.search(["test", "docs"])
    assert len(docs_items) == 2
    assert all(item.namespace == ("test", "docs") for item in docs_items)

    # Test value filtering
    alice_items = store.search(["test"], filter={"author": "Alice"})
    assert len(alice_items) == 2
    assert all(item.value["author"] == "Alice" for item in alice_items)

    # Test pagination
    paginated_items = store.search(["test"], limit=2)
    assert len(paginated_items) == 2

    offset_items = store.search(["test"], offset=2)
    assert len(offset_items) == 1

    # Cleanup
    for namespace, key, _ in test_data:
        store.delete(namespace, key)


@contextmanager
def _create_vector_store(
    vector_type: str,
    distance_type: str,
    fake_embeddings: Embeddings,
    text_fields: list[str] | None = None,
    enable_ttl: bool = True,
) -> PostgresStore:
    """Create a store with vector search enabled."""
    database = f"test_{uuid4().hex[:16]}"
    uri_parts = DEFAULT_URI.split("/")
    uri_base = "/".join(uri_parts[:-1])
    query_params = ""
    if "?" in uri_parts[-1]:
        db_name, query_params = uri_parts[-1].split("?", 1)
        query_params = "?" + query_params

    conn_string = f"{uri_base}/{database}{query_params}"
    admin_conn_string = DEFAULT_URI

    index_config = {
        "dims": fake_embeddings.dims,
        "embed": fake_embeddings,
        "ann_index_config": {
            "vector_type": vector_type,
        },
        "distance_type": distance_type,
        "fields": text_fields,
    }

    with Connection.connect(admin_conn_string, autocommit=True) as conn:
        conn.execute(f"CREATE DATABASE {database}")
    try:
        with PostgresStore.from_conn_string(
            conn_string,
            index=index_config,
            ttl={"default_ttl": 2, "refresh_on_read": True} if enable_ttl else None,
        ) as store:
            store.setup()
            with store._cursor() as cur:
                # drop the migration index
                cur.execute("DROP TABLE IF EXISTS store_migrations")
            store.setup()  # Will fail if migrations aren't idempotent
            yield store
    finally:
        with Connection.connect(admin_conn_string, autocommit=True) as conn:
            conn.execute(f"DROP DATABASE {database}")


_vector_params = [
    (vector_type, distance_type, True)
    for vector_type in VECTOR_TYPES
    for distance_type in (
        ["hamming"] if vector_type == "bit" else ["l2", "inner_product", "cosine"]
    )
]
_vector_params += [(*_vector_params[-1][:2], False)]


@pytest.fixture(
    scope="function",
    params=_vector_params,
    ids=lambda p: f"{p[0]}_{p[1]}",
)
def vector_store(
    request,
    fake_embeddings: Embeddings,
) -> PostgresStore:
    """Create a store with vector search enabled."""
    vector_type, distance_type, enable_ttl = request.param
    with _create_vector_store(
        vector_type, distance_type, fake_embeddings, enable_ttl=enable_ttl
    ) as store:
        yield store


def test_vector_store_initialization(
    vector_store: PostgresStore, fake_embeddings: CharacterEmbeddings
) -> None:
    """Test store initialization with embedding config."""
    # Store should be initialized with embedding config
    assert vector_store.index_config is not None
    assert vector_store.index_config["dims"] == fake_embeddings.dims
    assert vector_store.index_config["embed"] == fake_embeddings


def test_vector_insert_with_auto_embedding(vector_store: PostgresStore) -> None:
    """Test inserting items that get auto-embedded."""
    docs = [
        ("doc1", {"text": "short text"}),
        ("doc2", {"text": "longer text document"}),
        ("doc3", {"text": "longest text document here"}),
        ("doc4", {"description": "text in description field"}),
        ("doc5", {"content": "text in content field"}),
        ("doc6", {"body": "text in body field"}),
    ]

    for key, value in docs:
        vector_store.put(("test",), key, value)

    results = vector_store.search(("test",), query="long text")
    assert len(results) > 0

    doc_order = [r.key for r in results]
    assert "doc2" in doc_order
    assert "doc3" in doc_order


def test_vector_update_with_embedding(vector_store: PostgresStore) -> None:
    """Test that updating items properly updates their embeddings."""
    vector_store.put(("test",), "doc1", {"text": "zany zebra Xerxes"})
    vector_store.put(("test",), "doc2", {"text": "something about dogs"})
    vector_store.put(("test",), "doc3", {"text": "text about birds"})

    results_initial = vector_store.search(("test",), query="Zany Xerxes")
    assert len(results_initial) > 0
    assert results_initial[0].key == "doc1"
    initial_score = results_initial[0].score

    vector_store.put(("test",), "doc1", {"text": "new text about dogs"})

    results_after = vector_store.search(("test",), query="Zany Xerxes")
    after_score = next((r.score for r in results_after if r.key == "doc1"), 0.0)
    assert after_score < initial_score

    results_new = vector_store.search(("test",), query="new text about dogs")
    for r in results_new:
        if r.key == "doc1":
            assert r.score > after_score

    # Don't index this one
    vector_store.put(("test",), "doc4", {"text": "new text about dogs"}, index=False)
    results_new = vector_store.search(("test",), query="new text about dogs", limit=3)
    assert not any(r.key == "doc4" for r in results_new)


@pytest.mark.parametrize("refresh_ttl", [True, False])
def test_vector_search_with_filters(
    vector_store: PostgresStore, refresh_ttl: bool
) -> None:
    """Test combining vector search with filters."""
    # Insert test documents
    docs = [
        ("doc1", {"text": "red apple", "color": "red", "score": 4.5}),
        ("doc2", {"text": "red car", "color": "red", "score": 3.0}),
        ("doc3", {"text": "green apple", "color": "green", "score": 4.0}),
        ("doc4", {"text": "blue car", "color": "blue", "score": 3.5}),
    ]

    for key, value in docs:
        vector_store.put(("test",), key, value)

    results = vector_store.search(
        ("test",), query="apple", filter={"color": "red"}, refresh_ttl=refresh_ttl
    )
    assert len(results) == 2
    assert results[0].key == "doc1"

    results = vector_store.search(
        ("test",), query="car", filter={"color": "red"}, refresh_ttl=refresh_ttl
    )
    assert len(results) == 2
    assert results[0].key == "doc2"

    results = vector_store.search(
        ("test",),
        query="bbbbluuu",
        filter={"score": {"$gt": 3.2}},
        refresh_ttl=refresh_ttl,
    )
    assert len(results) == 3
    assert results[0].key == "doc4"

    # Multiple filters
    results = vector_store.search(
        ("test",), query="apple", filter={"score": {"$gte": 4.0}, "color": "green"}
    )
    assert len(results) == 1
    assert results[0].key == "doc3"


def test_vector_search_pagination(vector_store: PostgresStore) -> None:
    """Test pagination with vector search."""
    # Insert multiple similar documents
    for i in range(5):
        vector_store.put(("test",), f"doc{i}", {"text": f"test document number {i}"})

    # Test with different page sizes
    results_page1 = vector_store.search(("test",), query="test", limit=2)
    results_page2 = vector_store.search(("test",), query="test", limit=2, offset=2)

    assert len(results_page1) == 2
    assert len(results_page2) == 2
    assert results_page1[0].key != results_page2[0].key

    # Get all results
    all_results = vector_store.search(("test",), query="test", limit=10)
    assert len(all_results) == 5


def test_vector_search_edge_cases(vector_store: PostgresStore) -> None:
    """Test edge cases in vector search."""
    vector_store.put(("test",), "doc1", {"text": "test document"})

    results = vector_store.search(("test",), query="")
    assert len(results) == 1

    results = vector_store.search(("test",), query=None)
    assert len(results) == 1

    long_query = "test " * 100
    results = vector_store.search(("test",), query=long_query)
    assert len(results) == 1

    special_query = "test!@#$%^&*()"
    results = vector_store.search(("test",), query=special_query)
    assert len(results) == 1


@pytest.mark.parametrize(
    "vector_type,distance_type",
    [
        ("vector", "cosine"),
        ("vector", "inner_product"),
        ("halfvec", "cosine"),
        ("halfvec", "inner_product"),
    ],
)
def test_embed_with_path_sync(
    request: Any,
    fake_embeddings: CharacterEmbeddings,
    vector_type: str,
    distance_type: str,
) -> None:
    """Test vector search with specific text fields in Postgres store."""
    with _create_vector_store(
        vector_type,
        distance_type,
        fake_embeddings,
        text_fields=["key0", "key1", "key3"],
    ) as store:
        # This will have 2 vectors representing it
        doc1 = {
            # Omit key0 - check it doesn't raise an error
            "key1": "xxx",
            "key2": "yyy",
            "key3": "zzz",
        }
        # This will have 3 vectors representing it
        doc2 = {
            "key0": "uuu",
            "key1": "vvv",
            "key2": "www",
            "key3": "xxx",
        }
        store.put(("test",), "doc1", doc1)
        store.put(("test",), "doc2", doc2)

        # doc2.key3 and doc1.key1 both would have the highest score
        results = store.search(("test",), query="xxx")
        assert len(results) == 2
        assert results[0].key != results[1].key
        ascore = results[0].score
        bscore = results[1].score
        assert ascore == pytest.approx(bscore, abs=1e-3)

        # ~Only match doc2
        results = store.search(("test",), query="uuu")
        assert len(results) == 2
        assert results[0].key != results[1].key
        assert results[0].key == "doc2"
        assert results[0].score > results[1].score
        assert ascore == pytest.approx(results[0].score, abs=1e-3)

        # ~Only match doc1
        results = store.search(("test",), query="zzz")
        assert len(results) == 2
        assert results[0].key != results[1].key
        assert results[0].key == "doc1"
        assert results[0].score > results[1].score
        assert ascore == pytest.approx(results[0].score, abs=1e-3)

        # Un-indexed - will have low results for both. Not zero (because we're projecting)
        # but less than the above.
        results = store.search(("test",), query="www")
        assert len(results) == 2
        assert results[0].key != results[1].key
        assert results[0].score < ascore
        assert results[1].score < ascore


@pytest.mark.parametrize(
    "vector_type,distance_type",
    [
        ("vector", "cosine"),
        ("vector", "inner_product"),
        ("halfvec", "cosine"),
        ("halfvec", "inner_product"),
    ],
)
def test_embed_with_path_operation_config(
    request: Any,
    fake_embeddings: CharacterEmbeddings,
    vector_type: str,
    distance_type: str,
) -> None:
    """Test operation-level field configuration for vector search."""

    with _create_vector_store(
        vector_type,
        distance_type,
        fake_embeddings,
        text_fields=["key17"],  # Default fields that won't match our test data
    ) as store:
        doc3 = {
            "key0": "aaa",
            "key1": "bbb",
            "key2": "ccc",
            "key3": "ddd",
        }
        doc4 = {
            "key0": "eee",
            "key1": "bbb",  # Same as doc3.key1
            "key2": "fff",
            "key3": "ggg",
        }

        store.put(("test",), "doc3", doc3, index=["key0", "key1"])
        store.put(("test",), "doc4", doc4, index=["key1", "key3"])

        results = store.search(("test",), query="aaa")
        assert len(results) == 2
        assert results[0].key == "doc3"
        assert len(set(r.key for r in results)) == 2
        assert results[0].score > results[1].score

        results = store.search(("test",), query="ggg")
        assert len(results) == 2
        assert results[0].key == "doc4"
        assert results[0].score > results[1].score

        results = store.search(("test",), query="bbb")
        assert len(results) == 2
        assert results[0].key != results[1].key
        assert results[0].score == pytest.approx(results[1].score, abs=1e-3)

        results = store.search(("test",), query="ccc")
        assert len(results) == 2
        assert all(
            r.score < 0.9 for r in results
        )  # Unindexed field should have low scores

        # Test index=False behavior
        doc5 = {
            "key0": "hhh",
            "key1": "iii",
        }
        store.put(("test",), "doc5", doc5, index=False)
        results = store.search(("test",))
        assert len(results) == 3
        assert all(r.score is None for r in results), f"{results}"
        assert any(r.key == "doc5" for r in results)

        results = store.search(("test",), query="hhh")
        # TODO: We don't currently fill in additional results if there are not enough
        # returned during vector search.
        # assert len(results) == 3
        # doc5_result = next(r for r in results if r.key == "doc5")
        # assert doc5_result.score is None


def _cosine_similarity(X: list[float], Y: list[list[float]]) -> list[float]:
    """
    Compute cosine similarity between a vector X and a matrix Y.
    Lazy import numpy for efficiency.
    """

    similarities = []
    for y in Y:
        dot_product = sum(a * b for a, b in zip(X, y, strict=False))
        norm1 = sum(a * a for a in X) ** 0.5
        norm2 = sum(a * a for a in y) ** 0.5
        similarity = dot_product / (norm1 * norm2) if norm1 > 0 and norm2 > 0 else 0.0
        similarities.append(similarity)

    return similarities


def _inner_product(X: list[float], Y: list[list[float]]) -> list[float]:
    """
    Compute inner product between a vector X and a matrix Y.
    Lazy import numpy for efficiency.
    """

    similarities = []
    for y in Y:
        similarity = sum(a * b for a, b in zip(X, y, strict=False))
        similarities.append(similarity)

    return similarities


def _neg_l2_distance(X: list[float], Y: list[list[float]]) -> list[float]:
    """
    Compute l2 distance between a vector X and a matrix Y.
    Lazy import numpy for efficiency.
    """

    similarities = []
    for y in Y:
        similarity = sum((a - b) ** 2 for a, b in zip(X, y, strict=False)) ** 0.5
        similarities.append(-similarity)

    return similarities


@pytest.mark.parametrize(
    "vector_type,distance_type",
    [
        ("vector", "cosine"),
        ("vector", "inner_product"),
        ("halfvec", "l2"),
    ],
)
@pytest.mark.parametrize("query", ["aaa", "bbb", "ccc", "abcd", "poisson"])
def test_scores(
    fake_embeddings: CharacterEmbeddings,
    vector_type: str,
    distance_type: str,
    query: str,
) -> None:
    """Test operation-level field configuration for vector search."""
    with _create_vector_store(
        vector_type,
        distance_type,
        fake_embeddings,
        text_fields=["key0"],
    ) as store:
        doc = {
            "key0": "aaa",
        }
        store.put(("test",), "doc", doc, index=["key0", "key1"])

        results = store.search((), query=query)
        vec0 = fake_embeddings.embed_query(doc["key0"])
        vec1 = fake_embeddings.embed_query(query)
        if distance_type == "cosine":
            similarities = _cosine_similarity(vec1, [vec0])
        elif distance_type == "inner_product":
            similarities = _inner_product(vec1, [vec0])
        elif distance_type == "l2":
            similarities = _neg_l2_distance(vec1, [vec0])

        assert len(results) == 1
        assert results[0].score == pytest.approx(similarities[0], abs=1e-3)


def test_nonnull_migrations() -> None:
    _leading_comment_remover = re.compile(r"^/\*.*?\*/")
    for migration in PostgresStore.MIGRATIONS:
        statement = _leading_comment_remover.sub("", migration).split()[0]
        assert statement.strip()


def test_store_ttl(store):
    # Assumes a TTL of 1 minute = 60 seconds
    ns = ("foo",)
    store.put(
        ns,
        key="item1",
        value={"foo": "bar"},
        ttl=TTL_MINUTES,  # type: ignore
    )
    time.sleep(TTL_SECONDS - 2)
    res = store.get(ns, key="item1", refresh_ttl=True)
    assert res is not None
    time.sleep(TTL_SECONDS - 2)
    results = store.search(ns, query="foo", refresh_ttl=True)
    assert len(results) == 1
    time.sleep(TTL_SECONDS - 2)
    res = store.get(ns, key="item1", refresh_ttl=False)
    assert res is not None
    time.sleep(TTL_SECONDS - 1)
    # Now has been (TTL_SECONDS-2)*2 > TTL_SECONDS + TTL_SECONDS/2
    res = store.search(ns, query="bar", refresh_ttl=False)
    assert len(res) == 0


@pytest.mark.parametrize(
    "vector_type,distance_type",
    [
        ("vector", "cosine"),
        ("vector", "inner_product"),
        ("halfvec", "cosine"),
        ("halfvec", "inner_product"),
    ],
)
def test_non_ascii(
    request: Any,
    fake_embeddings: CharacterEmbeddings,
    vector_type: str,
    distance_type: str,
) -> None:
    """Test support for non-ascii characters"""
    with _create_vector_store(vector_type, distance_type, fake_embeddings) as store:
        store.put(("user_123", "memories"), "1", {"text": "这是中文"})  # Chinese
        store.put(
            ("user_123", "memories"), "2", {"text": "これは日本語です"}
        )  # Japanese
        store.put(("user_123", "memories"), "3", {"text": "이건 한국어야"})  # Korean
        store.put(("user_123", "memories"), "4", {"text": "Это русский"})  # Russian
        store.put(("user_123", "memories"), "5", {"text": "यह रूसी है"})  # Hindi

        result1 = store.search(("user_123", "memories"), query="这是中文")
        result2 = store.search(("user_123", "memories"), query="これは日本語です")
        result3 = store.search(("user_123", "memories"), query="이건 한국어야")
        result4 = store.search(("user_123", "memories"), query="Это русский")
        result5 = store.search(("user_123", "memories"), query="यह रूसी है")

        assert result1[0].key == "1"
        assert result2[0].key == "2"
        assert result3[0].key == "3"
        assert result4[0].key == "4"
        assert result5[0].key == "5"

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint-postgres/tests/test_sync.py
```py
# type: ignore

import re
from contextlib import contextmanager
from typing import Any
from uuid import uuid4

import pytest
from langchain_core.runnables import RunnableConfig
from langgraph.checkpoint.base import (
    EXCLUDED_METADATA_KEYS,
    Checkpoint,
    CheckpointMetadata,
    create_checkpoint,
    empty_checkpoint,
)
from langgraph.checkpoint.serde.types import TASKS
from psycopg import Connection
from psycopg.rows import dict_row
from psycopg_pool import ConnectionPool

from langgraph.checkpoint.postgres import PostgresSaver, ShallowPostgresSaver
from tests.conftest import DEFAULT_POSTGRES_URI


def _exclude_keys(config: dict[str, Any]) -> dict[str, Any]:
    return {k: v for k, v in config.items() if k not in EXCLUDED_METADATA_KEYS}


@contextmanager
def _pool_saver():
    """Fixture for pool mode testing."""
    database = f"test_{uuid4().hex[:16]}"
    # create unique db
    with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
        conn.execute(f"CREATE DATABASE {database}")
    try:
        # yield checkpointer
        with ConnectionPool(
            DEFAULT_POSTGRES_URI + database,
            max_size=10,
            kwargs={"autocommit": True, "row_factory": dict_row},
        ) as pool:
            checkpointer = PostgresSaver(pool)
            checkpointer.setup()
            yield checkpointer
    finally:
        # drop unique db
        with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
            conn.execute(f"DROP DATABASE {database}")


@contextmanager
def _pipe_saver():
    """Fixture for pipeline mode testing."""
    database = f"test_{uuid4().hex[:16]}"
    # create unique db
    with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
        conn.execute(f"CREATE DATABASE {database}")
    try:
        with Connection.connect(
            DEFAULT_POSTGRES_URI + database,
            autocommit=True,
            prepare_threshold=0,
            row_factory=dict_row,
        ) as conn:
            checkpointer = PostgresSaver(conn)
            checkpointer.setup()
            with conn.pipeline() as pipe:
                checkpointer = PostgresSaver(conn, pipe=pipe)
                yield checkpointer
    finally:
        # drop unique db
        with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
            conn.execute(f"DROP DATABASE {database}")


@contextmanager
def _base_saver():
    """Fixture for regular connection mode testing."""
    database = f"test_{uuid4().hex[:16]}"
    # create unique db
    with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
        conn.execute(f"CREATE DATABASE {database}")
    try:
        with Connection.connect(
            DEFAULT_POSTGRES_URI + database,
            autocommit=True,
            prepare_threshold=0,
            row_factory=dict_row,
        ) as conn:
            checkpointer = PostgresSaver(conn)
            checkpointer.setup()
            yield checkpointer
    finally:
        # drop unique db
        with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
            conn.execute(f"DROP DATABASE {database}")


@contextmanager
def _shallow_saver():
    """Fixture for regular connection mode testing with a shallow checkpointer."""
    database = f"test_{uuid4().hex[:16]}"
    # create unique db
    with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
        conn.execute(f"CREATE DATABASE {database}")
    try:
        with Connection.connect(
            DEFAULT_POSTGRES_URI + database,
            autocommit=True,
            prepare_threshold=0,
            row_factory=dict_row,
        ) as conn:
            checkpointer = ShallowPostgresSaver(conn)
            checkpointer.setup()
            yield checkpointer
    finally:
        # drop unique db
        with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
            conn.execute(f"DROP DATABASE {database}")


@contextmanager
def _saver(name: str):
    if name == "base":
        with _base_saver() as saver:
            yield saver
    elif name == "shallow":
        with _shallow_saver() as saver:
            yield saver
    elif name == "pool":
        with _pool_saver() as saver:
            yield saver
    elif name == "pipe":
        with _pipe_saver() as saver:
            yield saver


@pytest.fixture
def test_data():
    """Fixture providing test data for checkpoint tests."""
    config_1: RunnableConfig = {
        "configurable": {
            "thread_id": "thread-1",
            "checkpoint_id": "1",
            "checkpoint_ns": "",
        }
    }
    config_2: RunnableConfig = {
        "configurable": {
            "thread_id": "thread-2",
            "checkpoint_id": "2",
            "checkpoint_ns": "",
        }
    }
    config_3: RunnableConfig = {
        "configurable": {
            "thread_id": "thread-2",
            "checkpoint_id": "2-inner",
            "checkpoint_ns": "inner",
        }
    }

    chkpnt_1: Checkpoint = empty_checkpoint()
    chkpnt_2: Checkpoint = create_checkpoint(chkpnt_1, {}, 1)
    chkpnt_3: Checkpoint = empty_checkpoint()

    metadata_1: CheckpointMetadata = {
        "source": "input",
        "step": 2,
        "score": 1,
    }
    metadata_2: CheckpointMetadata = {
        "source": "loop",
        "step": 1,
        "score": None,
    }
    metadata_3: CheckpointMetadata = {}

    return {
        "configs": [config_1, config_2, config_3],
        "checkpoints": [chkpnt_1, chkpnt_2, chkpnt_3],
        "metadata": [metadata_1, metadata_2, metadata_3],
    }


@pytest.mark.parametrize("saver_name", ["base", "pool", "pipe", "shallow"])
def test_combined_metadata(saver_name: str, test_data) -> None:
    with _saver(saver_name) as saver:
        config = {
            "configurable": {
                "thread_id": "thread-2",
                "checkpoint_ns": "",
                "__super_private_key": "super_private_value",
            },
            "metadata": {"run_id": "my_run_id"},
        }
        chkpnt: Checkpoint = create_checkpoint(empty_checkpoint(), {}, 1)
        metadata: CheckpointMetadata = {
            "source": "loop",
            "step": 1,
            "score": None,
        }
        saver.put(config, chkpnt, metadata, {})
        checkpoint = saver.get_tuple(config)
        assert checkpoint.metadata == {
            **metadata,
            "run_id": "my_run_id",
        }


@pytest.mark.parametrize("saver_name", ["base", "pool", "pipe", "shallow"])
def test_search(saver_name: str, test_data) -> None:
    with _saver(saver_name) as saver:
        configs = test_data["configs"]
        checkpoints = test_data["checkpoints"]
        metadata = test_data["metadata"]

        saver.put(configs[0], checkpoints[0], metadata[0], {})
        saver.put(configs[1], checkpoints[1], metadata[1], {})
        saver.put(configs[2], checkpoints[2], metadata[2], {})

        # call method / assertions
        query_1 = {"source": "input"}  # search by 1 key
        query_2 = {
            "step": 1,
        }  # search by multiple keys
        query_3: dict[str, Any] = {}  # search by no keys, return all checkpoints
        query_4 = {"source": "update", "step": 1}  # no match

        search_results_1 = list(saver.list(None, filter=query_1))
        assert len(search_results_1) == 1
        assert search_results_1[0].metadata == {
            **_exclude_keys(configs[0]["configurable"]),
            **metadata[0],
        }

        search_results_2 = list(saver.list(None, filter=query_2))
        assert len(search_results_2) == 1
        assert search_results_2[0].metadata == {
            **_exclude_keys(configs[1]["configurable"]),
            **metadata[1],
        }

        search_results_3 = list(saver.list(None, filter=query_3))
        assert len(search_results_3) == 3

        search_results_4 = list(saver.list(None, filter=query_4))
        assert len(search_results_4) == 0

        # search by config (defaults to checkpoints across all namespaces)
        search_results_5 = list(saver.list({"configurable": {"thread_id": "thread-2"}}))
        assert len(search_results_5) == 2
        assert {
            search_results_5[0].config["configurable"]["checkpoint_ns"],
            search_results_5[1].config["configurable"]["checkpoint_ns"],
        } == {"", "inner"}


@pytest.mark.parametrize("saver_name", ["base", "pool", "pipe", "shallow"])
def test_null_chars(saver_name: str, test_data) -> None:
    with _saver(saver_name) as saver:
        config = saver.put(
            test_data["configs"][0],
            test_data["checkpoints"][0],
            {"my_key": "\x00abc"},
            {},
        )
        assert saver.get_tuple(config).metadata["my_key"] == "abc"  # type: ignore
        assert (
            list(saver.list(None, filter={"my_key": "abc"}))[0].metadata["my_key"]
            == "abc"
        )


def test_nonnull_migrations() -> None:
    _leading_comment_remover = re.compile(r"^/\*.*?\*/")
    for migration in PostgresSaver.MIGRATIONS:
        statement = _leading_comment_remover.sub("", migration).split()[0]
        assert statement.strip()


@pytest.mark.parametrize("saver_name", ["base", "pool", "pipe"])
def test_pending_sends_migration(saver_name: str) -> None:
    with _saver(saver_name) as saver:
        config = {
            "configurable": {
                "thread_id": "thread-1",
                "checkpoint_ns": "",
            }
        }

        # create the first checkpoint
        # and put some pending sends
        checkpoint_0 = empty_checkpoint()
        config = saver.put(config, checkpoint_0, {}, {})
        saver.put_writes(
            config, [(TASKS, "send-1"), (TASKS, "send-2")], task_id="task-1"
        )
        saver.put_writes(config, [(TASKS, "send-3")], task_id="task-2")

        # check that fetching checkpoint_0 doesn't attach pending sends
        # (they should be attached to the next checkpoint)
        tuple_0 = saver.get_tuple(config)
        assert tuple_0.checkpoint["channel_values"] == {}
        assert tuple_0.checkpoint["channel_versions"] == {}

        # create the second checkpoint
        checkpoint_1 = create_checkpoint(checkpoint_0, {}, 1)
        config = saver.put(config, checkpoint_1, {}, {})

        # check that pending sends are attached to checkpoint_1
        checkpoint_1 = saver.get_tuple(config)
        assert checkpoint_1.checkpoint["channel_values"] == {
            TASKS: ["send-1", "send-2", "send-3"]
        }
        assert TASKS in checkpoint_1.checkpoint["channel_versions"]

        # check that list also applies the migration
        search_results = [
            c for c in saver.list({"configurable": {"thread_id": "thread-1"}})
        ]
        assert len(search_results) == 2
        assert search_results[-1].checkpoint["channel_values"] == {}
        assert search_results[-1].checkpoint["channel_versions"] == {}
        assert search_results[0].checkpoint["channel_values"] == {
            TASKS: ["send-1", "send-2", "send-3"]
        }
        assert TASKS in search_results[0].checkpoint["channel_versions"]


@pytest.mark.parametrize("saver_name", ["base", "pool", "pipe"])
def test_get_checkpoint_no_channel_values(
    monkeypatch, saver_name: str, test_data
) -> None:
    """Backwards compatibility test that verifies a checkpoint with no channel_values key can be retrieved without throwing an error."""
    with _saver(saver_name) as saver:
        config = {
            "configurable": {
                "thread_id": "thread-2",
                "checkpoint_ns": "",
                "__super_private_key": "super_private_value",
            },
        }
        chkpnt: Checkpoint = create_checkpoint(empty_checkpoint(), {}, 1)
        saver.put(config, chkpnt, {}, {})

        load_checkpoint_tuple = saver._load_checkpoint_tuple

        def patched_load_checkpoint_tuple(value):
            value["checkpoint"].pop("channel_values", None)
            return load_checkpoint_tuple(value)

        monkeypatch.setattr(
            saver, "_load_checkpoint_tuple", patched_load_checkpoint_tuple
        )

        checkpoint = saver.get_tuple(config)
        assert checkpoint.checkpoint["channel_values"] == {}

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint/LICENSE
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
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint/Makefile
```text
.PHONY: test test_watch lint format

######################
# TESTING AND COVERAGE
######################

TEST ?= .

test:
	uv run pytest $(TEST)

test_watch:
	uv run ptw $(TEST)

######################
# LINTING AND FORMATTING
######################

# Define a variable for Python and notebook files.
PYTHON_FILES=.
MYPY_CACHE=.mypy_cache
lint format: PYTHON_FILES=.
lint_diff format_diff: PYTHON_FILES=$(shell git diff --name-only --relative --diff-filter=d main . | grep -E '\.py$$|\.ipynb$$')
lint_package: PYTHON_FILES=langgraph
lint_tests: PYTHON_FILES=tests
lint_tests: MYPY_CACHE=.mypy_cache_test

lint lint_diff lint_package lint_tests:
	uv run ruff check .
	[ "$(PYTHON_FILES)" = "" ] || uv run ruff format $(PYTHON_FILES) --diff
	[ "$(PYTHON_FILES)" = "" ] || uv run ruff check --select I $(PYTHON_FILES)
	[ "$(PYTHON_FILES)" = "" ] || mkdir -p $(MYPY_CACHE)
	[ "$(PYTHON_FILES)" = "" ] || uv run mypy $(PYTHON_FILES) --cache-dir $(MYPY_CACHE)

format format_diff:
	uv run ruff format $(PYTHON_FILES)
	uv run ruff check --select I --fix $(PYTHON_FILES)

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint/README.md
```md
# LangGraph Checkpoint

This library defines the base interface for LangGraph checkpointers. Checkpointers provide a persistence layer for LangGraph. They allow you to interact with and manage the graph's state. When you use a graph with a checkpointer, the checkpointer saves a _checkpoint_ of the graph state at every superstep, enabling several powerful capabilities like human-in-the-loop, "memory" between interactions and more.

## Key concepts

### Checkpoint

Checkpoint is a snapshot of the graph state at a given point in time. Checkpoint tuple refers to an object containing checkpoint and the associated config, metadata and pending writes.

### Thread

Threads enable the checkpointing of multiple different runs, making them essential for multi-tenant chat applications and other scenarios where maintaining separate states is necessary. A thread is a unique ID assigned to a series of checkpoints saved by a checkpointer. When using a checkpointer, you must specify a `thread_id` and optionally `checkpoint_id` when running the graph.

- `thread_id` is simply the ID of a thread. This is always required.
- `checkpoint_id` can optionally be passed. This identifier refers to a specific checkpoint within a thread. This can be used to kick off a run of a graph from some point halfway through a thread.

You must pass these when invoking the graph as part of the configurable part of the config, e.g.

```python
{"configurable": {"thread_id": "1"}}  # valid config
{"configurable": {"thread_id": "1", "checkpoint_id": "0c62ca34-ac19-445d-bbb0-5b4984975b2a"}}  # also valid config
```

### Serde

`langgraph_checkpoint` also defines protocol for serialization/deserialization (serde) and provides an default implementation (`langgraph.checkpoint.serde.jsonplus.JsonPlusSerializer`) that handles a wide variety of types, including LangChain and LangGraph primitives, datetimes, enums and more.

### Pending writes

When a graph node fails mid-execution at a given superstep, LangGraph stores pending checkpoint writes from any other nodes that completed successfully at that superstep, so that whenever we resume graph execution from that superstep we don't re-run the successful nodes.

## Interface

Each checkpointer should conform to `langgraph.checkpoint.base.BaseCheckpointSaver` interface and must implement the following methods:

- `.put` - Store a checkpoint with its configuration and metadata.
- `.put_writes` - Store intermediate writes linked to a checkpoint (i.e. pending writes).
- `.get_tuple` - Fetch a checkpoint tuple using for a given configuration (`thread_id` and `checkpoint_id`).
- `.list` - List checkpoints that match a given configuration and filter criteria.
- `.delete_thread()` - Delete all checkpoints and writes associated with a thread.
- `.get_next_version()` - Generate the next version ID for a channel.

If the checkpointer will be used with asynchronous graph execution (i.e. executing the graph via `.ainvoke`, `.astream`, `.abatch`), checkpointer must implement asynchronous versions of the above methods (`.aput`, `.aput_writes`, `.aget_tuple`, `.alist`). Similarly, the checkpointer must implement `.adelete_thread()` if asynchronous thread cleanup is desired. The base class provides a default implementation of `.get_next_version()` that generates an integer sequence starting from 1, but this method should be overridden for custom versioning schemes.

## Usage

```python
from langgraph.checkpoint.memory import InMemorySaver

write_config = {"configurable": {"thread_id": "1", "checkpoint_ns": ""}}
read_config = {"configurable": {"thread_id": "1"}}

checkpointer = InMemorySaver()
checkpoint = {
    "v": 4,
    "ts": "2024-07-31T20:14:19.804150+00:00",
    "id": "1ef4f797-8335-6428-8001-8a1503f9b875",
    "channel_values": {
      "my_key": "meow",
      "node": "node"
    },
    "channel_versions": {
      "__start__": 2,
      "my_key": 3,
      "start:node": 3,
      "node": 3
    },
    "versions_seen": {
      "__input__": {},
      "__start__": {
        "__start__": 1
      },
      "node": {
        "start:node": 2
      }
    },
}

# store checkpoint
checkpointer.put(write_config, checkpoint, {}, {})

# load checkpoint
checkpointer.get(read_config)

# list checkpoints
list(checkpointer.list(read_config))
```

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint/pyproject.toml
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "langgraph-checkpoint"
version = "3.0.1"
description = "Library with base interfaces for LangGraph checkpoint savers."
authors = []
requires-python = ">=3.10"
readme = "README.md"
license = "MIT"
license-files = ['LICENSE']
dependencies = [
    "langchain-core>=0.2.38",
    "ormsgpack>=1.12.0",
]

[project.urls]
Source = "https://github.com/langchain-ai/langgraph/tree/main/libs/checkpoint"
Twitter = "https://x.com/LangChainAI"
Slack = "https://www.langchain.com/join-community"
Reddit = "https://www.reddit.com/r/LangChain/"

[dependency-groups]
test = [
  "pytest",
  "pytest-asyncio",
  "pytest-mock",
  "pytest-watcher",
  "dataclasses-json",
  "numpy",
  "pandas",
  "pandas-stubs>=2.2.2.240807",
  "redis",
]
lint = [
  "ruff",
  "codespell",
  "mypy",
]
dev = [
  {include-group = "test"},
  {include-group = "lint"},
]

[tool.hatch.build.targets.wheel]
include = ["langgraph"]

[tool.pytest.ini_options]
addopts = "--strict-markers --strict-config --durations=5 -vv"
asyncio_mode = "auto"

[tool.ruff]
lint.select = [
  "E",  # pycodestyle
  "F",  # Pyflakes
  "UP", # pyupgrade
  "B",  # flake8-bugbear
  "I",  # isort
  "UP", # pyupgrade
]
lint.ignore = ["E501", "B008"]
target-version = "py310"

[tool.pytest-watcher]
now = true
delay = 0.1
runner_args = ["--ff", "-v", "--tb", "short"]
patterns = ["*.py"]

[tool.mypy]
# https://mypy.readthedocs.io/en/stable/config_file.html
disallow_untyped_defs = "True"
explicit_package_bases = "True"
warn_no_return = "False"
warn_unused_ignores = "True"
warn_redundant_casts = "True"
allow_redefinition = "True"
disable_error_code = "typeddict-item, return-value"

```
---
https://github.com/langchain-ai/langgraph/blob/main/libs/checkpoint/uv.lock
```lock
version = 1
revision = 3
requires-python = ">=3.10"
resolution-markers = [
    "python_full_version >= '3.12'",
    "python_full_version == '3.11.*'",
    "python_full_version < '3.11'",
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
name = "async-timeout"
version = "5.0.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/a5/ae/136395dfbfe00dfc94da3f3e136d0b13f394cba8f4841120e34226265780/async_timeout-5.0.1.tar.gz", hash = "sha256:d9321a7a3d5a6a5e187e824d2fa0793ce379a202935782d555d6e9d2735677d3", size = 9274, upload-time = "2024-11-06T16:41:39.6Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/fe/ba/e2081de779ca30d473f21f5b30e0e737c438205440784c7dfc81efc2b029/async_timeout-5.0.1-py3-none-any.whl", hash = "sha256:39e3809566ff85354557ec2398b55e096c8364bacac9405a7a1fa429e77fe76c", size = 6233, upload-time = "2024-11-06T16:41:37.9Z" },
]

[[package]]
name = "backports-asyncio-runner"
version = "1.2.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/8e/ff/70dca7d7cb1cbc0edb2c6cc0c38b65cba36cccc491eca64cabd5fe7f8670/backports_asyncio_runner-1.2.0.tar.gz", hash = "sha256:a5aa7b2b7d8f8bfcaa2b57313f70792df84e32a2a746f585213373f900b42162", size = 69893, upload-time = "2025-07-02T02:27:15.685Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/a0/59/76ab57e3fe74484f48a53f8e337171b4a2349e506eabe136d7e01d059086/backports_asyncio_runner-1.2.0-py3-none-any.whl", hash = "sha256:0da0a936a8aeb554eccb426dc55af3ba63bcdc69fa1a600b5bb305413a4477b5", size = 12313, upload-time = "2025-07-02T02:27:14.263Z" },
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
name = "codespell"
version = "2.4.1"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/15/e0/709453393c0ea77d007d907dd436b3ee262e28b30995ea1aa36c6ffbccaf/codespell-2.4.1.tar.gz", hash = "sha256:299fcdcb09d23e81e35a671bbe746d5ad7e8385972e65dbb833a2eaac33c01e5", size = 344740, upload-time = "2025-01-28T18:52:39.411Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/20/01/b394922252051e97aab231d416c86da3d8a6d781eeadcdca1082867de64e/codespell-2.4.1-py3-none-any.whl", hash = "sha256:3dadafa67df7e4a3dbf51e0d7315061b80d265f9552ebd699b3dd6834b47e425", size = 344501, upload-time = "2025-01-28T18:52:37.057Z" },
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
name = "iniconfig"
version = "2.3.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/72/34/14ca021ce8e5dfedc35312d08ba8bf51fdd999c576889fc2c24cb97f4f10/iniconfig-2.3.0.tar.gz", hash = "sha256:c76315c77db068650d49c5b56314774a7804df16fee4402c1f19d6d15d8c4730", size = 20503, upload-time = "2025-10-18T21:55:43.219Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/cb/b1/3846dd7f199d53cb17f49cba7e651e9ce294d8497c8c150530ed11865bb8/iniconfig-2.3.0-py3-none-any.whl", hash = "sha256:f631c04d2c48c52b84d0d0549c99ff3859c98df65b3101406327ecc7d53fbf12", size = 7484, upload-time = "2025-10-18T21:55:41.639Z" },
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
name = "langchain-core"
version = "1.0.3"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "jsonpatch" },
    { name = "langsmith" },
    { name = "packaging" },
    { name = "pydantic" },
    { name = "pyyaml" },
    { name = "tenacity" },
    { name = "typing-extensions" },
]
sdist = { url = "https://files.pythonhosted.org/packages/41/15/dfe0c2af463d63296fe18608a06570ce3a4b245253d4f26c301481380f7d/langchain_core-1.0.3.tar.gz", hash = "sha256:10744945d21168fb40d1162a5f1cf69bf0137ff6ad2b12c87c199a5297410887", size = 770278, upload-time = "2025-11-03T14:32:09.712Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/f2/1b/b0a37674bdcbd2931944e12ea742fd167098de5212ee2391e91dce631162/langchain_core-1.0.3-py3-none-any.whl", hash = "sha256:64f1bd45f04b174bbfd54c135a8adc52f4902b347c15a117d6383b412bf558a5", size = 469927, upload-time = "2025-11-03T14:32:08.322Z" },
]

[[package]]
name = "langgraph-checkpoint"
version = "3.0.1"
source = { editable = "." }
dependencies = [
    { name = "langchain-core" },
    { name = "ormsgpack" },
]

[package.dev-dependencies]
dev = [
    { name = "codespell" },
    { name = "dataclasses-json" },
    { name = "mypy" },
    { name = "numpy", version = "2.2.6", source = { registry = "https://pypi.org/simple" }, marker = "python_full_version < '3.11'" },
    { name = "numpy", version = "2.3.4", source = { registry = "https://pypi.org/simple" }, marker = "python_full_version >= '3.11'" },
    { name = "pandas" },
    { name = "pandas-stubs" },
    { name = "pytest" },
    { name = "pytest-asyncio" },
    { name = "pytest-mock" },
    { name = "pytest-watcher" },
    { name = "redis" },
    { name = "ruff" },
]
lint = [
    { name = "codespell" },
    { name = "mypy" },
    { name = "ruff" },
]
test = [
    { name = "dataclasses-json" },
    { name = "numpy", version = "2.2.6", source = { registry = "https://pypi.org/simple" }, marker = "python_full_version < '3.11'" },
    { name = "numpy", version = "2.3.4", source = { registry = "https://pypi.org/simple" }, marker = "python_full_version >= '3.11'" },
    { name = "pandas" },
    { name = "pandas-stubs" },
    { name = "pytest" },
    { name = "pytest-asyncio" },
    { name = "pytest-mock" },
    { name = "pytest-watcher" },
    { name = "redis" },
]

[package.metadata]
requires-dist = [
    { name = "langchain-core", specifier = ">=0.2.38" },
    { name = "ormsgpack", specifier = ">=1.12.0" },
]

[package.metadata.requires-dev]
dev = [
    { name = "codespell" },
    { name = "dataclasses-json" },
    { name = "mypy" },
    { name = "numpy" },
    { name = "pandas" },
    { name = "pandas-stubs", specifier = ">=2.2.2.240807" },
    { name = "pytest" },
    { name = "pytest-asyncio" },
    { name = "pytest-mock" },
    { name = "pytest-watcher" },
    { name = "redis" },
    { name = "ruff" },
]
lint = [
    { name = "codespell" },
    { name = "mypy" },
    { name = "ruff" },
]
test = [
    { name = "dataclasses-json" },
    { name = "numpy" },
    { name = "pandas" },
    { name = "pandas-stubs", specifier = ">=2.2.2.240807" },
    { name = "pytest" },
    { name = "pytest-asyncio" },
    { name = "pytest-mock" },
    { name = "pytest-watcher" },
    { name = "redis" },
]

[[package]]
name = "langsmith"
version = "0.4.40"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "httpx" },
    { name = "orjson", marker = "platform_python_implementation != 'PyPy'" },
    { name = "packaging" },
    { name = "pydantic" },
    { name = "requests" },
    { name = "requests-toolbelt" },
    { name = "zstandard" },
]
sdist = { url = "https://files.pythonhosted.org/packages/5d/68/988272a0f0ddf1f07c6e34176104f687b2f45a2d59a16371b17ac739d12c/langsmith-0.4.40.tar.gz", hash = "sha256:bd7704bd72c560dbcf5f72f81b3a02ace27ae8f15e0d5e46f9c4568b4908d0c2", size = 950190, upload-time = "2025-11-04T01:21:56.76Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/b3/8a/de38127895f185f677778930478c6884b087c2cab5744dd7e27c0188fbec/langsmith-0.4.40-py3-none-any.whl", hash = "sha256:1ec1a4981c39d4cee85b623ac86dd79e14b1b236ddc6357aec72a58940ecbfff", size = 399341, upload-time = "2025-11-04T01:21:54.407Z" },
]

[[package]]
name = "marshmallow"
version = "3.26.1"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "packaging" },
]
sdist = { url = "https://files.pythonhosted.org/packages/ab/5e/5e53d26b42ab75491cda89b871dab9e97c840bf12c63ec58a1919710cd06/marshmallow-3.26.1.tar.gz", hash = "sha256:e6d8affb6cb61d39d26402096dc0aee12d5a26d490a121f118d2e81dc0719dc6", size = 221825, upload-time = "2025-02-03T15:32:25.093Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/34/75/51952c7b2d3873b44a0028b1bd26a25078c18f92f256608e8d1dc61b39fd/marshmallow-3.26.1-py3-none-any.whl", hash = "sha256:3350409f20a70a7e4e11a27661187b77cdcaeb20abca41c1454fe33636bea09c", size = 50878, upload-time = "2025-02-03T15:32:22.295Z" },
]

[[package]]
name = "mypy"
version = "1.18.2"
source = { registry = "https://pypi.org/simple" }
dependencies = [
    { name = "mypy-extensions" },
    { name = "pathspec" },
    { name = "tomli", marker = "python_full_version < '3.11'" },
    { name = "typing-extensions" },
]
sdist = { url = "https://files.pythonhosted.org/packages/c0/77/8f0d0001ffad290cef2f7f216f96c814866248a0b92a722365ed54648e7e/mypy-1.18.2.tar.gz", hash = "sha256:06a398102a5f203d7477b2923dda3634c36727fa5c237d8f859ef90c42a9924b", size = 3448846, upload-time = "2025-09-19T00:11:10.519Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/03/6f/657961a0743cff32e6c0611b63ff1c1970a0b482ace35b069203bf705187/mypy-1.18.2-cp310-cp310-macosx_10_9_x86_64.whl", hash = "sha256:c1eab0cf6294dafe397c261a75f96dc2c31bffe3b944faa24db5def4e2b0f77c", size = 12807973, upload-time = "2025-09-19T00:10:35.282Z" },
    { url = "https://files.pythonhosted.org/packages/10/e9/420822d4f661f13ca8900f5fa239b40ee3be8b62b32f3357df9a3045a08b/mypy-1.18.2-cp310-cp310-macosx_11_0_arm64.whl", hash = "sha256:7a780ca61fc239e4865968ebc5240bb3bf610ef59ac398de9a7421b54e4a207e", size = 11896527, upload-time = "2025-09-19T00:10:55.791Z" },
    { url = "https://files.pythonhosted.org/packages/aa/73/a05b2bbaa7005f4642fcfe40fb73f2b4fb6bb44229bd585b5878e9a87ef8/mypy-1.18.2-cp310-cp310-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:448acd386266989ef11662ce3c8011fd2a7b632e0ec7d61a98edd8e27472225b", size = 12507004, upload-time = "2025-09-19T00:11:05.411Z" },
    { url = "https://files.pythonhosted.org/packages/4f/01/f6e4b9f0d031c11ccbd6f17da26564f3a0f3c4155af344006434b0a05a9d/mypy-1.18.2-cp310-cp310-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:f9e171c465ad3901dc652643ee4bffa8e9fef4d7d0eece23b428908c77a76a66", size = 13245947, upload-time = "2025-09-19T00:10:46.923Z" },
    { url = "https://files.pythonhosted.org/packages/d7/97/19727e7499bfa1ae0773d06afd30ac66a58ed7437d940c70548634b24185/mypy-1.18.2-cp310-cp310-musllinux_1_2_x86_64.whl", hash = "sha256:592ec214750bc00741af1f80cbf96b5013d81486b7bb24cb052382c19e40b428", size = 13499217, upload-time = "2025-09-19T00:09:39.472Z" },
    { url = "https://files.pythonhosted.org/packages/9f/4f/90dc8c15c1441bf31cf0f9918bb077e452618708199e530f4cbd5cede6ff/mypy-1.18.2-cp310-cp310-win_amd64.whl", hash = "sha256:7fb95f97199ea11769ebe3638c29b550b5221e997c63b14ef93d2e971606ebed", size = 9766753, upload-time = "2025-09-19T00:10:49.161Z" },
    { url = "https://files.pythonhosted.org/packages/88/87/cafd3ae563f88f94eec33f35ff722d043e09832ea8530ef149ec1efbaf08/mypy-1.18.2-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:807d9315ab9d464125aa9fcf6d84fde6e1dc67da0b6f80e7405506b8ac72bc7f", size = 12731198, upload-time = "2025-09-19T00:09:44.857Z" },
    { url = "https://files.pythonhosted.org/packages/0f/e0/1e96c3d4266a06d4b0197ace5356d67d937d8358e2ee3ffac71faa843724/mypy-1.18.2-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:776bb00de1778caf4db739c6e83919c1d85a448f71979b6a0edd774ea8399341", size = 11817879, upload-time = "2025-09-19T00:09:47.131Z" },
    { url = "https://files.pythonhosted.org/packages/72/ef/0c9ba89eb03453e76bdac5a78b08260a848c7bfc5d6603634774d9cd9525/mypy-1.18.2-cp311-cp311-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:1379451880512ffce14505493bd9fe469e0697543717298242574882cf8cdb8d", size = 12427292, upload-time = "2025-09-19T00:10:22.472Z" },
    { url = "https://files.pythonhosted.org/packages/1a/52/ec4a061dd599eb8179d5411d99775bec2a20542505988f40fc2fee781068/mypy-1.18.2-cp311-cp311-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:1331eb7fd110d60c24999893320967594ff84c38ac6d19e0a76c5fd809a84c86", size = 13163750, upload-time = "2025-09-19T00:09:51.472Z" },
    { url = "https://files.pythonhosted.org/packages/c4/5f/2cf2ceb3b36372d51568f2208c021870fe7834cf3186b653ac6446511839/mypy-1.18.2-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:3ca30b50a51e7ba93b00422e486cbb124f1c56a535e20eff7b2d6ab72b3b2e37", size = 13351827, upload-time = "2025-09-19T00:09:58.311Z" },
    { url = "https://files.pythonhosted.org/packages/c8/7d/2697b930179e7277529eaaec1513f8de622818696857f689e4a5432e5e27/mypy-1.18.2-cp311-cp311-win_amd64.whl", hash = "sha256:664dc726e67fa54e14536f6e1224bcfce1d9e5ac02426d2326e2bb4e081d1ce8", size = 9757983, upload-time = "2025-09-19T00:10:09.071Z" },
    { url = "https://files.pythonhosted.org/packages/07/06/dfdd2bc60c66611dd8335f463818514733bc763e4760dee289dcc33df709/mypy-1.18.2-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:33eca32dd124b29400c31d7cf784e795b050ace0e1f91b8dc035672725617e34", size = 12908273, upload-time = "2025-09-19T00:10:58.321Z" },
    { url = "https://files.pythonhosted.org/packages/81/14/6a9de6d13a122d5608e1a04130724caf9170333ac5a924e10f670687d3eb/mypy-1.18.2-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:a3c47adf30d65e89b2dcd2fa32f3aeb5e94ca970d2c15fcb25e297871c8e4764", size = 11920910, upload-time = "2025-09-19T00:10:20.043Z" },
    { url = "https://files.pythonhosted.org/packages/5f/a9/b29de53e42f18e8cc547e38daa9dfa132ffdc64f7250e353f5c8cdd44bee/mypy-1.18.2-cp312-cp312-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:5d6c838e831a062f5f29d11c9057c6009f60cb294fea33a98422688181fe2893", size = 12465585, upload-time = "2025-09-19T00:10:33.005Z" },
    { url = "https://files.pythonhosted.org/packages/77/ae/6c3d2c7c61ff21f2bee938c917616c92ebf852f015fb55917fd6e2811db2/mypy-1.18.2-cp312-cp312-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:01199871b6110a2ce984bde85acd481232d17413868c9807e95c1b0739a58914", size = 13348562, upload-time = "2025-09-19T00:10:11.51Z" },
    { url = "https://files.pythonhosted.org/packages/4d/31/aec68ab3b4aebdf8f36d191b0685d99faa899ab990753ca0fee60fb99511/mypy-1.18.2-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:a2afc0fa0b0e91b4599ddfe0f91e2c26c2b5a5ab263737e998d6817874c5f7c8", size = 13533296, upload-time = "2025-09-19T00:10:06.568Z" },
    { url = "https://files.pythonhosted.org/packages/9f/83/abcb3ad9478fca3ebeb6a5358bb0b22c95ea42b43b7789c7fb1297ca44f4/mypy-1.18.2-cp312-cp312-win_amd64.whl", hash = "sha256:d8068d0afe682c7c4897c0f7ce84ea77f6de953262b12d07038f4d296d547074", size = 9828828, upload-time = "2025-09-19T00:10:28.203Z" },
    { url = "https://files.pythonhosted.org/packages/5f/04/7f462e6fbba87a72bc8097b93f6842499c428a6ff0c81dd46948d175afe8/mypy-1.18.2-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:07b8b0f580ca6d289e69209ec9d3911b4a26e5abfde32228a288eb79df129fcc", size = 12898728, upload-time = "2025-09-19T00:10:01.33Z" },
    { url = "https://files.pythonhosted.org/packages/99/5b/61ed4efb64f1871b41fd0b82d29a64640f3516078f6c7905b68ab1ad8b13/mypy-1.18.2-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:ed4482847168439651d3feee5833ccedbf6657e964572706a2adb1f7fa4dfe2e", size = 11910758, upload-time = "2025-09-19T00:10:42.607Z" },
    { url = "https://files.pythonhosted.org/packages/3c/46/d297d4b683cc89a6e4108c4250a6a6b717f5fa96e1a30a7944a6da44da35/mypy-1.18.2-cp313-cp313-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:c3ad2afadd1e9fea5cf99a45a822346971ede8685cc581ed9cd4d42eaf940986", size = 12475342, upload-time = "2025-09-19T00:11:00.371Z" },
    { url = "https://files.pythonhosted.org/packages/83/45/4798f4d00df13eae3bfdf726c9244bcb495ab5bd588c0eed93a2f2dd67f3/mypy-1.18.2-cp313-cp313-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:a431a6f1ef14cf8c144c6b14793a23ec4eae3db28277c358136e79d7d062f62d", size = 13338709, upload-time = "2025-09-19T00:11:03.358Z" },
    { url = "https://files.pythonhosted.org/packages/d7/09/479f7358d9625172521a87a9271ddd2441e1dab16a09708f056e97007207/mypy-1.18.2-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:7ab28cc197f1dd77a67e1c6f35cd1f8e8b73ed2217e4fc005f9e6a504e46e7ba", size = 13529806, upload-time = "2025-09-19T00:10:26.073Z" },
    { url = "https://files.pythonhosted.org/packages/71/cf/ac0f2c7e9d0ea3c75cd99dff7aec1c9df4a1376537cb90e4c882267ee7e9/mypy-1.18.2-cp313-cp313-win_amd64.whl", hash = "sha256:0e2785a84b34a72ba55fb5daf079a1003a34c05b22238da94fcae2bbe46f3544", size = 9833262, upload-time = "2025-09-19T00:10:40.035Z" },
    { url = "https://files.pythonhosted.org/packages/5a/0c/7d5300883da16f0063ae53996358758b2a2df2a09c72a5061fa79a1f5006/mypy-1.18.2-cp314-cp314-macosx_10_13_x86_64.whl", hash = "sha256:62f0e1e988ad41c2a110edde6c398383a889d95b36b3e60bcf155f5164c4fdce", size = 12893775, upload-time = "2025-09-19T00:10:03.814Z" },
    { url = "https://files.pythonhosted.org/packages/50/df/2cffbf25737bdb236f60c973edf62e3e7b4ee1c25b6878629e88e2cde967/mypy-1.18.2-cp314-cp314-macosx_11_0_arm64.whl", hash = "sha256:8795a039bab805ff0c1dfdb8cd3344642c2b99b8e439d057aba30850b8d3423d", size = 11936852, upload-time = "2025-09-19T00:10:51.631Z" },
    { url = "https://files.pythonhosted.org/packages/be/50/34059de13dd269227fb4a03be1faee6e2a4b04a2051c82ac0a0b5a773c9a/mypy-1.18.2-cp314-cp314-manylinux2014_aarch64.manylinux_2_17_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:6ca1e64b24a700ab5ce10133f7ccd956a04715463d30498e64ea8715236f9c9c", size = 12480242, upload-time = "2025-09-19T00:11:07.955Z" },
    { url = "https://files.pythonhosted.org/packages/5b/11/040983fad5132d85914c874a2836252bbc57832065548885b5bb5b0d4359/mypy-1.18.2-cp314-cp314-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:d924eef3795cc89fecf6bedc6ed32b33ac13e8321344f6ddbf8ee89f706c05cb", size = 13326683, upload-time = "2025-09-19T00:09:55.572Z" },
    { url = "https://files.pythonhosted.org/packages/e9/ba/89b2901dd77414dd7a8c8729985832a5735053be15b744c18e4586e506ef/mypy-1.18.2-cp314-cp314-musllinux_1_2_x86_64.whl", hash = "sha256:20c02215a080e3a2be3aa50506c67242df1c151eaba0dcbc1e4e557922a26075", size = 13514749, upload-time = "2025-09-19T00:10:44.827Z" },
    { url = "https://files.pythonhosted.org/packages/25/bc/cc98767cffd6b2928ba680f3e5bc969c4152bf7c2d83f92f5a504b92b0eb/mypy-1.18.2-cp314-cp314-win_amd64.whl", hash = "sha256:749b5f83198f1ca64345603118a6f01a4e99ad4bf9d103ddc5a3200cc4614adf", size = 9982959, upload-time = "2025-09-19T00:10:37.344Z" },
    { url = "https://files.pythonhosted.org/packages/87/e3/be76d87158ebafa0309946c4a73831974d4d6ab4f4ef40c3b53a385a66fd/mypy-1.18.2-py3-none-any.whl", hash = "sha256:22a1748707dd62b58d2ae53562ffc4d7f8bcc727e8ac7cbc69c053ddc874d47e", size = 2352367, upload-time = "2025-09-19T00:10:15.489Z" },
]

[[package]]
name = "mypy-extensions"
version = "1.1.0"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/a2/6e/371856a3fb9d31ca8dac321cda606860fa4548858c0cc45d9d1d4ca2628b/mypy_extensions-1.1.0.tar.gz", hash = "sha256:52e68efc3284861e772bbcd66823fde5ae21fd2fdb51c62a211403730b916558", size = 6343, upload-time = "2025-04-22T14:54:24.164Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/79/7b/2c79738432f5c924bef5071f933bcc9efd0473bac3b4aa584a6f7c1c8df8/mypy_extensions-1.1.0-py3-none-any.whl", hash = "sha256:1be4cccdb0f2482337c4743e60421de3a356cd97508abadd57d47403e94f5505", size = 4963, upload-time = "2025-04-22T14:54:22.983Z" },
]

[[package]]
name = "numpy"
version = "2.2.6"
source = { registry = "https://pypi.org/simple" }
resolution-markers = [
    "python_full_version < '3.11'",
]
sdist = { url = "https://files.pythonhosted.org/packages/76/21/7d2a95e4bba9dc13d043ee156a356c0a8f0c6309dff6b21b4d71a073b8a8/numpy-2.2.6.tar.gz", hash = "sha256:e29554e2bef54a90aa5cc07da6ce955accb83f21ab5de01a62c8478897b264fd", size = 20276440, upload-time = "2025-05-17T22:38:04.611Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/9a/3e/ed6db5be21ce87955c0cbd3009f2803f59fa08df21b5df06862e2d8e2bdd/numpy-2.2.6-cp310-cp310-macosx_10_9_x86_64.whl", hash = "sha256:b412caa66f72040e6d268491a59f2c43bf03eb6c96dd8f0307829feb7fa2b6fb", size = 21165245, upload-time = "2025-05-17T21:27:58.555Z" },
    { url = "https://files.pythonhosted.org/packages/22/c2/4b9221495b2a132cc9d2eb862e21d42a009f5a60e45fc44b00118c174bff/numpy-2.2.6-cp310-cp310-macosx_11_0_arm64.whl", hash = "sha256:8e41fd67c52b86603a91c1a505ebaef50b3314de0213461c7a6e99c9a3beff90", size = 14360048, upload-time = "2025-05-17T21:28:21.406Z" },
    { url = "https://files.pythonhosted.org/packages/fd/77/dc2fcfc66943c6410e2bf598062f5959372735ffda175b39906d54f02349/numpy-2.2.6-cp310-cp310-macosx_14_0_arm64.whl", hash = "sha256:37e990a01ae6ec7fe7fa1c26c55ecb672dd98b19c3d0e1d1f326fa13cb38d163", size = 5340542, upload-time = "2025-05-17T21:28:30.931Z" },
    { url = "https://files.pythonhosted.org/packages/7a/4f/1cb5fdc353a5f5cc7feb692db9b8ec2c3d6405453f982435efc52561df58/numpy-2.2.6-cp310-cp310-macosx_14_0_x86_64.whl", hash = "sha256:5a6429d4be8ca66d889b7cf70f536a397dc45ba6faeb5f8c5427935d9592e9cf", size = 6878301, upload-time = "2025-05-17T21:28:41.613Z" },
    { url = "https://files.pythonhosted.org/packages/eb/17/96a3acd228cec142fcb8723bd3cc39c2a474f7dcf0a5d16731980bcafa95/numpy-2.2.6-cp310-cp310-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:efd28d4e9cd7d7a8d39074a4d44c63eda73401580c5c76acda2ce969e0a38e83", size = 14297320, upload-time = "2025-05-17T21:29:02.78Z" },
    { url = "https://files.pythonhosted.org/packages/b4/63/3de6a34ad7ad6646ac7d2f55ebc6ad439dbbf9c4370017c50cf403fb19b5/numpy-2.2.6-cp310-cp310-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:fc7b73d02efb0e18c000e9ad8b83480dfcd5dfd11065997ed4c6747470ae8915", size = 16801050, upload-time = "2025-05-17T21:29:27.675Z" },
    { url = "https://files.pythonhosted.org/packages/07/b6/89d837eddef52b3d0cec5c6ba0456c1bf1b9ef6a6672fc2b7873c3ec4e2e/numpy-2.2.6-cp310-cp310-musllinux_1_2_aarch64.whl", hash = "sha256:74d4531beb257d2c3f4b261bfb0fc09e0f9ebb8842d82a7b4209415896adc680", size = 15807034, upload-time = "2025-05-17T21:29:51.102Z" },
    { url = "https://files.pythonhosted.org/packages/01/c8/dc6ae86e3c61cfec1f178e5c9f7858584049b6093f843bca541f94120920/numpy-2.2.6-cp310-cp310-musllinux_1_2_x86_64.whl", hash = "sha256:8fc377d995680230e83241d8a96def29f204b5782f371c532579b4f20607a289", size = 18614185, upload-time = "2025-05-17T21:30:18.703Z" },
    { url = "https://files.pythonhosted.org/packages/5b/c5/0064b1b7e7c89137b471ccec1fd2282fceaae0ab3a9550f2568782d80357/numpy-2.2.6-cp310-cp310-win32.whl", hash = "sha256:b093dd74e50a8cba3e873868d9e93a85b78e0daf2e98c6797566ad8044e8363d", size = 6527149, upload-time = "2025-05-17T21:30:29.788Z" },
    { url = "https://files.pythonhosted.org/packages/a3/dd/4b822569d6b96c39d1215dbae0582fd99954dcbcf0c1a13c61783feaca3f/numpy-2.2.6-cp310-cp310-win_amd64.whl", hash = "sha256:f0fd6321b839904e15c46e0d257fdd101dd7f530fe03fd6359c1ea63738703f3", size = 12904620, upload-time = "2025-05-17T21:30:48.994Z" },
    { url = "https://files.pythonhosted.org/packages/da/a8/4f83e2aa666a9fbf56d6118faaaf5f1974d456b1823fda0a176eff722839/numpy-2.2.6-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:f9f1adb22318e121c5c69a09142811a201ef17ab257a1e66ca3025065b7f53ae", size = 21176963, upload-time = "2025-05-17T21:31:19.36Z" },
    { url = "https://files.pythonhosted.org/packages/b3/2b/64e1affc7972decb74c9e29e5649fac940514910960ba25cd9af4488b66c/numpy-2.2.6-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:c820a93b0255bc360f53eca31a0e676fd1101f673dda8da93454a12e23fc5f7a", size = 14406743, upload-time = "2025-05-17T21:31:41.087Z" },
    { url = "https://files.pythonhosted.org/packages/4a/9f/0121e375000b5e50ffdd8b25bf78d8e1a5aa4cca3f185d41265198c7b834/numpy-2.2.6-cp311-cp311-macosx_14_0_arm64.whl", hash = "sha256:3d70692235e759f260c3d837193090014aebdf026dfd167834bcba43e30c2a42", size = 5352616, upload-time = "2025-05-17T21:31:50.072Z" },
    { url = "https://files.pythonhosted.org/packages/31/0d/b48c405c91693635fbe2dcd7bc84a33a602add5f63286e024d3b6741411c/numpy-2.2.6-cp311-cp311-macosx_14_0_x86_64.whl", hash = "sha256:481b49095335f8eed42e39e8041327c05b0f6f4780488f61286ed3c01368d491", size = 6889579, upload-time = "2025-05-17T21:32:01.712Z" },
    { url = "https://files.pythonhosted.org/packages/52/b8/7f0554d49b565d0171eab6e99001846882000883998e7b7d9f0d98b1f934/numpy-2.2.6-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:b64d8d4d17135e00c8e346e0a738deb17e754230d7e0810ac5012750bbd85a5a", size = 14312005, upload-time = "2025-05-17T21:32:23.332Z" },
    { url = "https://files.pythonhosted.org/packages/b3/dd/2238b898e51bd6d389b7389ffb20d7f4c10066d80351187ec8e303a5a475/numpy-2.2.6-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:ba10f8411898fc418a521833e014a77d3ca01c15b0c6cdcce6a0d2897e6dbbdf", size = 16821570, upload-time = "2025-05-17T21:32:47.991Z" },
    { url = "https://files.pythonhosted.org/packages/83/6c/44d0325722cf644f191042bf47eedad61c1e6df2432ed65cbe28509d404e/numpy-2.2.6-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:bd48227a919f1bafbdda0583705e547892342c26fb127219d60a5c36882609d1", size = 15818548, upload-time = "2025-05-17T21:33:11.728Z" },
    { url = "https://files.pythonhosted.org/packages/ae/9d/81e8216030ce66be25279098789b665d49ff19eef08bfa8cb96d4957f422/numpy-2.2.6-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:9551a499bf125c1d4f9e250377c1ee2eddd02e01eac6644c080162c0c51778ab", size = 18620521, upload-time = "2025-05-17T21:33:39.139Z" },
    { url = "https://files.pythonhosted.org/packages/6a/fd/e19617b9530b031db51b0926eed5345ce8ddc669bb3bc0044b23e275ebe8/numpy-2.2.6-cp311-cp311-win32.whl", hash = "sha256:0678000bb9ac1475cd454c6b8c799206af8107e310843532b04d49649c717a47", size = 6525866, upload-time = "2025-05-17T21:33:50.273Z" },
    { url = "https://files.pythonhosted.org/packages/31/0a/f354fb7176b81747d870f7991dc763e157a934c717b67b58456bc63da3df/numpy-2.2.6-cp311-cp311-win_amd64.whl", hash = "sha256:e8213002e427c69c45a52bbd94163084025f533a55a59d6f9c5b820774ef3303", size = 12907455, upload-time = "2025-05-17T21:34:09.135Z" },
    { url = "https://files.pythonhosted.org/packages/82/5d/c00588b6cf18e1da539b45d3598d3557084990dcc4331960c15ee776ee41/numpy-2.2.6-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:41c5a21f4a04fa86436124d388f6ed60a9343a6f767fced1a8a71c3fbca038ff", size = 20875348, upload-time = "2025-05-17T21:34:39.648Z" },
    { url = "https://files.pythonhosted.org/packages/66/ee/560deadcdde6c2f90200450d5938f63a34b37e27ebff162810f716f6a230/numpy-2.2.6-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:de749064336d37e340f640b05f24e9e3dd678c57318c7289d222a8a2f543e90c", size = 14119362, upload-time = "2025-05-17T21:35:01.241Z" },
    { url = "https://files.pythonhosted.org/packages/3c/65/4baa99f1c53b30adf0acd9a5519078871ddde8d2339dc5a7fde80d9d87da/numpy-2.2.6-cp312-cp312-macosx_14_0_arm64.whl", hash = "sha256:894b3a42502226a1cac872f840030665f33326fc3dac8e57c607905773cdcde3", size = 5084103, upload-time = "2025-05-17T21:35:10.622Z" },
    { url = "https://files.pythonhosted.org/packages/cc/89/e5a34c071a0570cc40c9a54eb472d113eea6d002e9ae12bb3a8407fb912e/numpy-2.2.6-cp312-cp312-macosx_14_0_x86_64.whl", hash = "sha256:71594f7c51a18e728451bb50cc60a3ce4e6538822731b2933209a1f3614e9282", size = 6625382, upload-time = "2025-05-17T21:35:21.414Z" },
    { url = "https://files.pythonhosted.org/packages/f8/35/8c80729f1ff76b3921d5c9487c7ac3de9b2a103b1cd05e905b3090513510/numpy-2.2.6-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:f2618db89be1b4e05f7a1a847a9c1c0abd63e63a1607d892dd54668dd92faf87", size = 14018462, upload-time = "2025-05-17T21:35:42.174Z" },
    { url = "https://files.pythonhosted.org/packages/8c/3d/1e1db36cfd41f895d266b103df00ca5b3cbe965184df824dec5c08c6b803/numpy-2.2.6-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:fd83c01228a688733f1ded5201c678f0c53ecc1006ffbc404db9f7a899ac6249", size = 16527618, upload-time = "2025-05-17T21:36:06.711Z" },
    { url = "https://files.pythonhosted.org/packages/61/c6/03ed30992602c85aa3cd95b9070a514f8b3c33e31124694438d88809ae36/numpy-2.2.6-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:37c0ca431f82cd5fa716eca9506aefcabc247fb27ba69c5062a6d3ade8cf8f49", size = 15505511, upload-time = "2025-05-17T21:36:29.965Z" },
    { url = "https://files.pythonhosted.org/packages/b7/25/5761d832a81df431e260719ec45de696414266613c9ee268394dd5ad8236/numpy-2.2.6-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:fe27749d33bb772c80dcd84ae7e8df2adc920ae8297400dabec45f0dedb3f6de", size = 18313783, upload-time = "2025-05-17T21:36:56.883Z" },
    { url = "https://files.pythonhosted.org/packages/57/0a/72d5a3527c5ebffcd47bde9162c39fae1f90138c961e5296491ce778e682/numpy-2.2.6-cp312-cp312-win32.whl", hash = "sha256:4eeaae00d789f66c7a25ac5f34b71a7035bb474e679f410e5e1a94deb24cf2d4", size = 6246506, upload-time = "2025-05-17T21:37:07.368Z" },
    { url = "https://files.pythonhosted.org/packages/36/fa/8c9210162ca1b88529ab76b41ba02d433fd54fecaf6feb70ef9f124683f1/numpy-2.2.6-cp312-cp312-win_amd64.whl", hash = "sha256:c1f9540be57940698ed329904db803cf7a402f3fc200bfe599334c9bd84a40b2", size = 12614190, upload-time = "2025-05-17T21:37:26.213Z" },
    { url = "https://files.pythonhosted.org/packages/f9/5c/6657823f4f594f72b5471f1db1ab12e26e890bb2e41897522d134d2a3e81/numpy-2.2.6-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:0811bb762109d9708cca4d0b13c4f67146e3c3b7cf8d34018c722adb2d957c84", size = 20867828, upload-time = "2025-05-17T21:37:56.699Z" },
    { url = "https://files.pythonhosted.org/packages/dc/9e/14520dc3dadf3c803473bd07e9b2bd1b69bc583cb2497b47000fed2fa92f/numpy-2.2.6-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:287cc3162b6f01463ccd86be154f284d0893d2b3ed7292439ea97eafa8170e0b", size = 14143006, upload-time = "2025-05-17T21:38:18.291Z" },
    { url = "https://files.pythonhosted.org/packages/4f/06/7e96c57d90bebdce9918412087fc22ca9851cceaf5567a45c1f404480e9e/numpy-2.2.6-cp313-cp313-macosx_14_0_arm64.whl", hash = "sha256:f1372f041402e37e5e633e586f62aa53de2eac8d98cbfb822806ce4bbefcb74d", size = 5076765, upload-time = "2025-05-17T21:38:27.319Z" },
    { url = "https://files.pythonhosted.org/packages/73/ed/63d920c23b4289fdac96ddbdd6132e9427790977d5457cd132f18e76eae0/numpy-2.2.6-cp313-cp313-macosx_14_0_x86_64.whl", hash = "sha256:55a4d33fa519660d69614a9fad433be87e5252f4b03850642f88993f7b2ca566", size = 6617736, upload-time = "2025-05-17T21:38:38.141Z" },
    { url = "https://files.pythonhosted.org/packages/85/c5/e19c8f99d83fd377ec8c7e0cf627a8049746da54afc24ef0a0cb73d5dfb5/numpy-2.2.6-cp313-cp313-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:f92729c95468a2f4f15e9bb94c432a9229d0d50de67304399627a943201baa2f", size = 14010719, upload-time = "2025-05-17T21:38:58.433Z" },
    { url = "https://files.pythonhosted.org/packages/19/49/4df9123aafa7b539317bf6d342cb6d227e49f7a35b99c287a6109b13dd93/numpy-2.2.6-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:1bc23a79bfabc5d056d106f9befb8d50c31ced2fbc70eedb8155aec74a45798f", size = 16526072, upload-time = "2025-05-17T21:39:22.638Z" },
    { url = "https://files.pythonhosted.org/packages/b2/6c/04b5f47f4f32f7c2b0e7260442a8cbcf8168b0e1a41ff1495da42f42a14f/numpy-2.2.6-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:e3143e4451880bed956e706a3220b4e5cf6172ef05fcc397f6f36a550b1dd868", size = 15503213, upload-time = "2025-05-17T21:39:45.865Z" },
    { url = "https://files.pythonhosted.org/packages/17/0a/5cd92e352c1307640d5b6fec1b2ffb06cd0dabe7d7b8227f97933d378422/numpy-2.2.6-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:b4f13750ce79751586ae2eb824ba7e1e8dba64784086c98cdbbcc6a42112ce0d", size = 18316632, upload-time = "2025-05-17T21:40:13.331Z" },
    { url = "https://files.pythonhosted.org/packages/f0/3b/5cba2b1d88760ef86596ad0f3d484b1cbff7c115ae2429678465057c5155/numpy-2.2.6-cp313-cp313-win32.whl", hash = "sha256:5beb72339d9d4fa36522fc63802f469b13cdbe4fdab4a288f0c441b74272ebfd", size = 6244532, upload-time = "2025-05-17T21:43:46.099Z" },
    { url = "https://files.pythonhosted.org/packages/cb/3b/d58c12eafcb298d4e6d0d40216866ab15f59e55d148a5658bb3132311fcf/numpy-2.2.6-cp313-cp313-win_amd64.whl", hash = "sha256:b0544343a702fa80c95ad5d3d608ea3599dd54d4632df855e4c8d24eb6ecfa1c", size = 12610885, upload-time = "2025-05-17T21:44:05.145Z" },
    { url = "https://files.pythonhosted.org/packages/6b/9e/4bf918b818e516322db999ac25d00c75788ddfd2d2ade4fa66f1f38097e1/numpy-2.2.6-cp313-cp313t-macosx_10_13_x86_64.whl", hash = "sha256:0bca768cd85ae743b2affdc762d617eddf3bcf8724435498a1e80132d04879e6", size = 20963467, upload-time = "2025-05-17T21:40:44Z" },
    { url = "https://files.pythonhosted.org/packages/61/66/d2de6b291507517ff2e438e13ff7b1e2cdbdb7cb40b3ed475377aece69f9/numpy-2.2.6-cp313-cp313t-macosx_11_0_arm64.whl", hash = "sha256:fc0c5673685c508a142ca65209b4e79ed6740a4ed6b2267dbba90f34b0b3cfda", size = 14225144, upload-time = "2025-05-17T21:41:05.695Z" },
    { url = "https://files.pythonhosted.org/packages/e4/25/480387655407ead912e28ba3a820bc69af9adf13bcbe40b299d454ec011f/numpy-2.2.6-cp313-cp313t-macosx_14_0_arm64.whl", hash = "sha256:5bd4fc3ac8926b3819797a7c0e2631eb889b4118a9898c84f585a54d475b7e40", size = 5200217, upload-time = "2025-05-17T21:41:15.903Z" },
    { url = "https://files.pythonhosted.org/packages/aa/4a/6e313b5108f53dcbf3aca0c0f3e9c92f4c10ce57a0a721851f9785872895/numpy-2.2.6-cp313-cp313t-macosx_14_0_x86_64.whl", hash = "sha256:fee4236c876c4e8369388054d02d0e9bb84821feb1a64dd59e137e6511a551f8", size = 6712014, upload-time = "2025-05-17T21:41:27.321Z" },
    { url = "https://files.pythonhosted.org/packages/b7/30/172c2d5c4be71fdf476e9de553443cf8e25feddbe185e0bd88b096915bcc/numpy-2.2.6-cp313-cp313t-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:e1dda9c7e08dc141e0247a5b8f49cf05984955246a327d4c48bda16821947b2f", size = 14077935, upload-time = "2025-05-17T21:41:49.738Z" },
    { url = "https://files.pythonhosted.org/packages/12/fb/9e743f8d4e4d3c710902cf87af3512082ae3d43b945d5d16563f26ec251d/numpy-2.2.6-cp313-cp313t-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:f447e6acb680fd307f40d3da4852208af94afdfab89cf850986c3ca00562f4fa", size = 16600122, upload-time = "2025-05-17T21:42:14.046Z" },
    { url = "https://files.pythonhosted.org/packages/12/75/ee20da0e58d3a66f204f38916757e01e33a9737d0b22373b3eb5a27358f9/numpy-2.2.6-cp313-cp313t-musllinux_1_2_aarch64.whl", hash = "sha256:389d771b1623ec92636b0786bc4ae56abafad4a4c513d36a55dce14bd9ce8571", size = 15586143, upload-time = "2025-05-17T21:42:37.464Z" },
    { url = "https://files.pythonhosted.org/packages/76/95/bef5b37f29fc5e739947e9ce5179ad402875633308504a52d188302319c8/numpy-2.2.6-cp313-cp313t-musllinux_1_2_x86_64.whl", hash = "sha256:8e9ace4a37db23421249ed236fdcdd457d671e25146786dfc96835cd951aa7c1", size = 18385260, upload-time = "2025-05-17T21:43:05.189Z" },
    { url = "https://files.pythonhosted.org/packages/09/04/f2f83279d287407cf36a7a8053a5abe7be3622a4363337338f2585e4afda/numpy-2.2.6-cp313-cp313t-win32.whl", hash = "sha256:038613e9fb8c72b0a41f025a7e4c3f0b7a1b5d768ece4796b674c8f3fe13efff", size = 6377225, upload-time = "2025-05-17T21:43:16.254Z" },
    { url = "https://files.pythonhosted.org/packages/67/0e/35082d13c09c02c011cf21570543d202ad929d961c02a147493cb0c2bdf5/numpy-2.2.6-cp313-cp313t-win_amd64.whl", hash = "sha256:6031dd6dfecc0cf9f668681a37648373bddd6421fff6c66ec1624eed0180ee06", size = 12771374, upload-time = "2025-05-17T21:43:35.479Z" },
    { url = "https://files.pythonhosted.org/packages/9e/3b/d94a75f4dbf1ef5d321523ecac21ef23a3cd2ac8b78ae2aac40873590229/numpy-2.2.6-pp310-pypy310_pp73-macosx_10_15_x86_64.whl", hash = "sha256:0b605b275d7bd0c640cad4e5d30fa701a8d59302e127e5f79138ad62762c3e3d", size = 21040391, upload-time = "2025-05-17T21:44:35.948Z" },
    { url = "https://files.pythonhosted.org/packages/17/f4/09b2fa1b58f0fb4f7c7963a1649c64c4d315752240377ed74d9cd878f7b5/numpy-2.2.6-pp310-pypy310_pp73-macosx_14_0_x86_64.whl", hash = "sha256:7befc596a7dc9da8a337f79802ee8adb30a552a94f792b9c9d18c840055907db", size = 6786754, upload-time = "2025-05-17T21:44:47.446Z" },
    { url = "https://files.pythonhosted.org/packages/af/30/feba75f143bdc868a1cc3f44ccfa6c4b9ec522b36458e738cd00f67b573f/numpy-2.2.6-pp310-pypy310_pp73-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:ce47521a4754c8f4593837384bd3424880629f718d87c5d44f8ed763edd63543", size = 16643476, upload-time = "2025-05-17T21:45:11.871Z" },
    { url = "https://files.pythonhosted.org/packages/37/48/ac2a9584402fb6c0cd5b5d1a91dcf176b15760130dd386bbafdbfe3640bf/numpy-2.2.6-pp310-pypy310_pp73-win_amd64.whl", hash = "sha256:d042d24c90c41b54fd506da306759e06e568864df8ec17ccc17e9e884634fd00", size = 12812666, upload-time = "2025-05-17T21:45:31.426Z" },
]

[[package]]
name = "numpy"
version = "2.3.4"
source = { registry = "https://pypi.org/simple" }
resolution-markers = [
    "python_full_version >= '3.12'",
    "python_full_version == '3.11.*'",
]
sdist = { url = "https://files.pythonhosted.org/packages/b5/f4/098d2270d52b41f1bd7db9fc288aaa0400cb48c2a3e2af6fa365d9720947/numpy-2.3.4.tar.gz", hash = "sha256:a7d018bfedb375a8d979ac758b120ba846a7fe764911a64465fd87b8729f4a6a", size = 20582187, upload-time = "2025-10-15T16:18:11.77Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/60/e7/0e07379944aa8afb49a556a2b54587b828eb41dc9adc56fb7615b678ca53/numpy-2.3.4-cp311-cp311-macosx_10_9_x86_64.whl", hash = "sha256:e78aecd2800b32e8347ce49316d3eaf04aed849cd5b38e0af39f829a4e59f5eb", size = 21259519, upload-time = "2025-10-15T16:15:19.012Z" },
    { url = "https://files.pythonhosted.org/packages/d0/cb/5a69293561e8819b09e34ed9e873b9a82b5f2ade23dce4c51dc507f6cfe1/numpy-2.3.4-cp311-cp311-macosx_11_0_arm64.whl", hash = "sha256:7fd09cc5d65bda1e79432859c40978010622112e9194e581e3415a3eccc7f43f", size = 14452796, upload-time = "2025-10-15T16:15:23.094Z" },
    { url = "https://files.pythonhosted.org/packages/e4/04/ff11611200acd602a1e5129e36cfd25bf01ad8e5cf927baf2e90236eb02e/numpy-2.3.4-cp311-cp311-macosx_14_0_arm64.whl", hash = "sha256:1b219560ae2c1de48ead517d085bc2d05b9433f8e49d0955c82e8cd37bd7bf36", size = 5381639, upload-time = "2025-10-15T16:15:25.572Z" },
    { url = "https://files.pythonhosted.org/packages/ea/77/e95c757a6fe7a48d28a009267408e8aa382630cc1ad1db7451b3bc21dbb4/numpy-2.3.4-cp311-cp311-macosx_14_0_x86_64.whl", hash = "sha256:bafa7d87d4c99752d07815ed7a2c0964f8ab311eb8168f41b910bd01d15b6032", size = 6914296, upload-time = "2025-10-15T16:15:27.079Z" },
    { url = "https://files.pythonhosted.org/packages/a3/d2/137c7b6841c942124eae921279e5c41b1c34bab0e6fc60c7348e69afd165/numpy-2.3.4-cp311-cp311-manylinux_2_27_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:36dc13af226aeab72b7abad501d370d606326a0029b9f435eacb3b8c94b8a8b7", size = 14591904, upload-time = "2025-10-15T16:15:29.044Z" },
    { url = "https://files.pythonhosted.org/packages/bb/32/67e3b0f07b0aba57a078c4ab777a9e8e6bc62f24fb53a2337f75f9691699/numpy-2.3.4-cp311-cp311-manylinux_2_27_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:a7b2f9a18b5ff9824a6af80de4f37f4ec3c2aab05ef08f51c77a093f5b89adda", size = 16939602, upload-time = "2025-10-15T16:15:31.106Z" },
    { url = "https://files.pythonhosted.org/packages/95/22/9639c30e32c93c4cee3ccdb4b09c2d0fbff4dcd06d36b357da06146530fb/numpy-2.3.4-cp311-cp311-musllinux_1_2_aarch64.whl", hash = "sha256:9984bd645a8db6ca15d850ff996856d8762c51a2239225288f08f9050ca240a0", size = 16372661, upload-time = "2025-10-15T16:15:33.546Z" },
    { url = "https://files.pythonhosted.org/packages/12/e9/a685079529be2b0156ae0c11b13d6be647743095bb51d46589e95be88086/numpy-2.3.4-cp311-cp311-musllinux_1_2_x86_64.whl", hash = "sha256:64c5825affc76942973a70acf438a8ab618dbd692b84cd5ec40a0a0509edc09a", size = 18884682, upload-time = "2025-10-15T16:15:36.105Z" },
    { url = "https://files.pythonhosted.org/packages/cf/85/f6f00d019b0cc741e64b4e00ce865a57b6bed945d1bbeb1ccadbc647959b/numpy-2.3.4-cp311-cp311-win32.whl", hash = "sha256:ed759bf7a70342f7817d88376eb7142fab9fef8320d6019ef87fae05a99874e1", size = 6570076, upload-time = "2025-10-15T16:15:38.225Z" },
    { url = "https://files.pythonhosted.org/packages/7d/10/f8850982021cb90e2ec31990291f9e830ce7d94eef432b15066e7cbe0bec/numpy-2.3.4-cp311-cp311-win_amd64.whl", hash = "sha256:faba246fb30ea2a526c2e9645f61612341de1a83fb1e0c5edf4ddda5a9c10996", size = 13089358, upload-time = "2025-10-15T16:15:40.404Z" },
    { url = "https://files.pythonhosted.org/packages/d1/ad/afdd8351385edf0b3445f9e24210a9c3971ef4de8fd85155462fc4321d79/numpy-2.3.4-cp311-cp311-win_arm64.whl", hash = "sha256:4c01835e718bcebe80394fd0ac66c07cbb90147ebbdad3dcecd3f25de2ae7e2c", size = 10462292, upload-time = "2025-10-15T16:15:42.896Z" },
    { url = "https://files.pythonhosted.org/packages/96/7a/02420400b736f84317e759291b8edaeee9dc921f72b045475a9cbdb26b17/numpy-2.3.4-cp312-cp312-macosx_10_13_x86_64.whl", hash = "sha256:ef1b5a3e808bc40827b5fa2c8196151a4c5abe110e1726949d7abddfe5c7ae11", size = 20957727, upload-time = "2025-10-15T16:15:44.9Z" },
    { url = "https://files.pythonhosted.org/packages/18/90/a014805d627aa5750f6f0e878172afb6454552da929144b3c07fcae1bb13/numpy-2.3.4-cp312-cp312-macosx_11_0_arm64.whl", hash = "sha256:c2f91f496a87235c6aaf6d3f3d89b17dba64996abadccb289f48456cff931ca9", size = 14187262, upload-time = "2025-10-15T16:15:47.761Z" },
    { url = "https://files.pythonhosted.org/packages/c7/e4/0a94b09abe89e500dc748e7515f21a13e30c5c3fe3396e6d4ac108c25fca/numpy-2.3.4-cp312-cp312-macosx_14_0_arm64.whl", hash = "sha256:f77e5b3d3da652b474cc80a14084927a5e86a5eccf54ca8ca5cbd697bf7f2667", size = 5115992, upload-time = "2025-10-15T16:15:50.144Z" },
    { url = "https://files.pythonhosted.org/packages/88/dd/db77c75b055c6157cbd4f9c92c4458daef0dd9cbe6d8d2fe7f803cb64c37/numpy-2.3.4-cp312-cp312-macosx_14_0_x86_64.whl", hash = "sha256:8ab1c5f5ee40d6e01cbe96de5863e39b215a4d24e7d007cad56c7184fdf4aeef", size = 6648672, upload-time = "2025-10-15T16:15:52.442Z" },
    { url = "https://files.pythonhosted.org/packages/e1/e6/e31b0d713719610e406c0ea3ae0d90760465b086da8783e2fd835ad59027/numpy-2.3.4-cp312-cp312-manylinux_2_27_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:77b84453f3adcb994ddbd0d1c5d11db2d6bda1a2b7fd5ac5bd4649d6f5dc682e", size = 14284156, upload-time = "2025-10-15T16:15:54.351Z" },
    { url = "https://files.pythonhosted.org/packages/f9/58/30a85127bfee6f108282107caf8e06a1f0cc997cb6b52cdee699276fcce4/numpy-2.3.4-cp312-cp312-manylinux_2_27_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:4121c5beb58a7f9e6dfdee612cb24f4df5cd4db6e8261d7f4d7450a997a65d6a", size = 16641271, upload-time = "2025-10-15T16:15:56.67Z" },
    { url = "https://files.pythonhosted.org/packages/06/f2/2e06a0f2adf23e3ae29283ad96959267938d0efd20a2e25353b70065bfec/numpy-2.3.4-cp312-cp312-musllinux_1_2_aarch64.whl", hash = "sha256:65611ecbb00ac9846efe04db15cbe6186f562f6bb7e5e05f077e53a599225d16", size = 16059531, upload-time = "2025-10-15T16:15:59.412Z" },
    { url = "https://files.pythonhosted.org/packages/b0/e7/b106253c7c0d5dc352b9c8fab91afd76a93950998167fa3e5afe4ef3a18f/numpy-2.3.4-cp312-cp312-musllinux_1_2_x86_64.whl", hash = "sha256:dabc42f9c6577bcc13001b8810d300fe814b4cfbe8a92c873f269484594f9786", size = 18578983, upload-time = "2025-10-15T16:16:01.804Z" },
    { url = "https://files.pythonhosted.org/packages/73/e3/04ecc41e71462276ee867ccbef26a4448638eadecf1bc56772c9ed6d0255/numpy-2.3.4-cp312-cp312-win32.whl", hash = "sha256:a49d797192a8d950ca59ee2d0337a4d804f713bb5c3c50e8db26d49666e351dc", size = 6291380, upload-time = "2025-10-15T16:16:03.938Z" },
    { url = "https://files.pythonhosted.org/packages/3d/a8/566578b10d8d0e9955b1b6cd5db4e9d4592dd0026a941ff7994cedda030a/numpy-2.3.4-cp312-cp312-win_amd64.whl", hash = "sha256:985f1e46358f06c2a09921e8921e2c98168ed4ae12ccd6e5e87a4f1857923f32", size = 12787999, upload-time = "2025-10-15T16:16:05.801Z" },
    { url = "https://files.pythonhosted.org/packages/58/22/9c903a957d0a8071b607f5b1bff0761d6e608b9a965945411f867d515db1/numpy-2.3.4-cp312-cp312-win_arm64.whl", hash = "sha256:4635239814149e06e2cb9db3dd584b2fa64316c96f10656983b8026a82e6e4db", size = 10197412, upload-time = "2025-10-15T16:16:07.854Z" },
    { url = "https://files.pythonhosted.org/packages/57/7e/b72610cc91edf138bc588df5150957a4937221ca6058b825b4725c27be62/numpy-2.3.4-cp313-cp313-macosx_10_13_x86_64.whl", hash = "sha256:c090d4860032b857d94144d1a9976b8e36709e40386db289aaf6672de2a81966", size = 20950335, upload-time = "2025-10-15T16:16:10.304Z" },
    { url = "https://files.pythonhosted.org/packages/3e/46/bdd3370dcea2f95ef14af79dbf81e6927102ddf1cc54adc0024d61252fd9/numpy-2.3.4-cp313-cp313-macosx_11_0_arm64.whl", hash = "sha256:a13fc473b6db0be619e45f11f9e81260f7302f8d180c49a22b6e6120022596b3", size = 14179878, upload-time = "2025-10-15T16:16:12.595Z" },
    { url = "https://files.pythonhosted.org/packages/ac/01/5a67cb785bda60f45415d09c2bc245433f1c68dd82eef9c9002c508b5a65/numpy-2.3.4-cp313-cp313-macosx_14_0_arm64.whl", hash = "sha256:3634093d0b428e6c32c3a69b78e554f0cd20ee420dcad5a9f3b2a63762ce4197", size = 5108673, upload-time = "2025-10-15T16:16:14.877Z" },
    { url = "https://files.pythonhosted.org/packages/c2/cd/8428e23a9fcebd33988f4cb61208fda832800ca03781f471f3727a820704/numpy-2.3.4-cp313-cp313-macosx_14_0_x86_64.whl", hash = "sha256:043885b4f7e6e232d7df4f51ffdef8c36320ee9d5f227b380ea636722c7ed12e", size = 6641438, upload-time = "2025-10-15T16:16:16.805Z" },
    { url = "https://files.pythonhosted.org/packages/3e/d1/913fe563820f3c6b079f992458f7331278dcd7ba8427e8e745af37ddb44f/numpy-2.3.4-cp313-cp313-manylinux_2_27_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:4ee6a571d1e4f0ea6d5f22d6e5fbd6ed1dc2b18542848e1e7301bd190500c9d7", size = 14281290, upload-time = "2025-10-15T16:16:18.764Z" },
    { url = "https://files.pythonhosted.org/packages/9e/7e/7d306ff7cb143e6d975cfa7eb98a93e73495c4deabb7d1b5ecf09ea0fd69/numpy-2.3.4-cp313-cp313-manylinux_2_27_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:fc8a63918b04b8571789688b2780ab2b4a33ab44bfe8ccea36d3eba51228c953", size = 16636543, upload-time = "2025-10-15T16:16:21.072Z" },
    { url = "https://files.pythonhosted.org/packages/47/6a/8cfc486237e56ccfb0db234945552a557ca266f022d281a2f577b98e955c/numpy-2.3.4-cp313-cp313-musllinux_1_2_aarch64.whl", hash = "sha256:40cc556d5abbc54aabe2b1ae287042d7bdb80c08edede19f0c0afb36ae586f37", size = 16056117, upload-time = "2025-10-15T16:16:23.369Z" },
    { url = "https://files.pythonhosted.org/packages/b1/0e/42cb5e69ea901e06ce24bfcc4b5664a56f950a70efdcf221f30d9615f3f3/numpy-2.3.4-cp313-cp313-musllinux_1_2_x86_64.whl", hash = "sha256:ecb63014bb7f4ce653f8be7f1df8cbc6093a5a2811211770f6606cc92b5a78fd", size = 18577788, upload-time = "2025-10-15T16:16:27.496Z" },
    { url = "https://files.pythonhosted.org/packages/86/92/41c3d5157d3177559ef0a35da50f0cda7fa071f4ba2306dd36818591a5bc/numpy-2.3.4-cp313-cp313-win32.whl", hash = "sha256:e8370eb6925bb8c1c4264fec52b0384b44f675f191df91cbe0140ec9f0955646", size = 6282620, upload-time = "2025-10-15T16:16:29.811Z" },
    { url = "https://files.pythonhosted.org/packages/09/97/fd421e8bc50766665ad35536c2bb4ef916533ba1fdd053a62d96cc7c8b95/numpy-2.3.4-cp313-cp313-win_amd64.whl", hash = "sha256:56209416e81a7893036eea03abcb91c130643eb14233b2515c90dcac963fe99d", size = 12784672, upload-time = "2025-10-15T16:16:31.589Z" },
    { url = "https://files.pythonhosted.org/packages/ad/df/5474fb2f74970ca8eb978093969b125a84cc3d30e47f82191f981f13a8a0/numpy-2.3.4-cp313-cp313-win_arm64.whl", hash = "sha256:a700a4031bc0fd6936e78a752eefb79092cecad2599ea9c8039c548bc097f9bc", size = 10196702, upload-time = "2025-10-15T16:16:33.902Z" },
    { url = "https://files.pythonhosted.org/packages/11/83/66ac031464ec1767ea3ed48ce40f615eb441072945e98693bec0bcd056cc/numpy-2.3.4-cp313-cp313t-macosx_10_13_x86_64.whl", hash = "sha256:86966db35c4040fdca64f0816a1c1dd8dbd027d90fca5a57e00e1ca4cd41b879", size = 21049003, upload-time = "2025-10-15T16:16:36.101Z" },
    { url = "https://files.pythonhosted.org/packages/5f/99/5b14e0e686e61371659a1d5bebd04596b1d72227ce36eed121bb0aeab798/numpy-2.3.4-cp313-cp313t-macosx_11_0_arm64.whl", hash = "sha256:838f045478638b26c375ee96ea89464d38428c69170360b23a1a50fa4baa3562", size = 14302980, upload-time = "2025-10-15T16:16:39.124Z" },
    { url = "https://files.pythonhosted.org/packages/2c/44/e9486649cd087d9fc6920e3fc3ac2aba10838d10804b1e179fb7cbc4e634/numpy-2.3.4-cp313-cp313t-macosx_14_0_arm64.whl", hash = "sha256:d7315ed1dab0286adca467377c8381cd748f3dc92235f22a7dfc42745644a96a", size = 5231472, upload-time = "2025-10-15T16:16:41.168Z" },
    { url = "https://files.pythonhosted.org/packages/3e/51/902b24fa8887e5fe2063fd61b1895a476d0bbf46811ab0c7fdf4bd127345/numpy-2.3.4-cp313-cp313t-macosx_14_0_x86_64.whl", hash = "sha256:84f01a4d18b2cc4ade1814a08e5f3c907b079c847051d720fad15ce37aa930b6", size = 6739342, upload-time = "2025-10-15T16:16:43.777Z" },
    { url = "https://files.pythonhosted.org/packages/34/f1/4de9586d05b1962acdcdb1dc4af6646361a643f8c864cef7c852bf509740/numpy-2.3.4-cp313-cp313t-manylinux_2_27_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:817e719a868f0dacde4abdfc5c1910b301877970195db9ab6a5e2c4bd5b121f7", size = 14354338, upload-time = "2025-10-15T16:16:46.081Z" },
    { url = "https://files.pythonhosted.org/packages/1f/06/1c16103b425de7969d5a76bdf5ada0804b476fed05d5f9e17b777f1cbefd/numpy-2.3.4-cp313-cp313t-manylinux_2_27_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:85e071da78d92a214212cacea81c6da557cab307f2c34b5f85b628e94803f9c0", size = 16702392, upload-time = "2025-10-15T16:16:48.455Z" },
    { url = "https://files.pythonhosted.org/packages/34/b2/65f4dc1b89b5322093572b6e55161bb42e3e0487067af73627f795cc9d47/numpy-2.3.4-cp313-cp313t-musllinux_1_2_aarch64.whl", hash = "sha256:2ec646892819370cf3558f518797f16597b4e4669894a2ba712caccc9da53f1f", size = 16134998, upload-time = "2025-10-15T16:16:51.114Z" },
    { url = "https://files.pythonhosted.org/packages/d4/11/94ec578896cdb973aaf56425d6c7f2aff4186a5c00fac15ff2ec46998b46/numpy-2.3.4-cp313-cp313t-musllinux_1_2_x86_64.whl", hash = "sha256:035796aaaddfe2f9664b9a9372f089cfc88bd795a67bd1bfe15e6e770934cf64", size = 18651574, upload-time = "2025-10-15T16:16:53.429Z" },
    { url = "https://files.pythonhosted.org/packages/62/b7/7efa763ab33dbccf56dade36938a77345ce8e8192d6b39e470ca25ff3cd0/numpy-2.3.4-cp313-cp313t-win32.whl", hash = "sha256:fea80f4f4cf83b54c3a051f2f727870ee51e22f0248d3114b8e755d160b38cfb", size = 6413135, upload-time = "2025-10-15T16:16:55.992Z" },
    { url = "https://files.pythonhosted.org/packages/43/70/aba4c38e8400abcc2f345e13d972fb36c26409b3e644366db7649015f291/numpy-2.3.4-cp313-cp313t-win_amd64.whl", hash = "sha256:15eea9f306b98e0be91eb344a94c0e630689ef302e10c2ce5f7e11905c704f9c", size = 12928582, upload-time = "2025-10-15T16:16:57.943Z" },
    { url = "https://files.pythonhosted.org/packages/67/63/871fad5f0073fc00fbbdd7232962ea1ac40eeaae2bba66c76214f7954236/numpy-2.3.4-cp313-cp313t-win_arm64.whl", hash = "sha256:b6c231c9c2fadbae4011ca5e7e83e12dc4a5072f1a1d85a0a7b3ed754d145a40", size = 10266691, upload-time = "2025-10-15T16:17:00.048Z" },
    { url = "https://files.pythonhosted.org/packages/72/71/ae6170143c115732470ae3a2d01512870dd16e0953f8a6dc89525696069b/numpy-2.3.4-cp314-cp314-macosx_10_15_x86_64.whl", hash = "sha256:81c3e6d8c97295a7360d367f9f8553973651b76907988bb6066376bc2252f24e", size = 20955580, upload-time = "2025-10-15T16:17:02.509Z" },
    { url = "https://files.pythonhosted.org/packages/af/39/4be9222ffd6ca8a30eda033d5f753276a9c3426c397bb137d8e19dedd200/numpy-2.3.4-cp314-cp314-macosx_11_0_arm64.whl", hash = "sha256:7c26b0b2bf58009ed1f38a641f3db4be8d960a417ca96d14e5b06df1506d41ff", size = 14188056, upload-time = "2025-10-15T16:17:04.873Z" },
    { url = "https://files.pythonhosted.org/packages/6c/3d/d85f6700d0a4aa4f9491030e1021c2b2b7421b2b38d01acd16734a2bfdc7/numpy-2.3.4-cp314-cp314-macosx_14_0_arm64.whl", hash = "sha256:62b2198c438058a20b6704351b35a1d7db881812d8512d67a69c9de1f18ca05f", size = 5116555, upload-time = "2025-10-15T16:17:07.499Z" },
    { url = "https://files.pythonhosted.org/packages/bf/04/82c1467d86f47eee8a19a464c92f90a9bb68ccf14a54c5224d7031241ffb/numpy-2.3.4-cp314-cp314-macosx_14_0_x86_64.whl", hash = "sha256:9d729d60f8d53a7361707f4b68a9663c968882dd4f09e0d58c044c8bf5faee7b", size = 6643581, upload-time = "2025-10-15T16:17:09.774Z" },
    { url = "https://files.pythonhosted.org/packages/0c/d3/c79841741b837e293f48bd7db89d0ac7a4f2503b382b78a790ef1dc778a5/numpy-2.3.4-cp314-cp314-manylinux_2_27_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:bd0c630cf256b0a7fd9d0a11c9413b42fef5101219ce6ed5a09624f5a65392c7", size = 14299186, upload-time = "2025-10-15T16:17:11.937Z" },
    { url = "https://files.pythonhosted.org/packages/e8/7e/4a14a769741fbf237eec5a12a2cbc7a4c4e061852b6533bcb9e9a796c908/numpy-2.3.4-cp314-cp314-manylinux_2_27_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:d5e081bc082825f8b139f9e9fe42942cb4054524598aaeb177ff476cc76d09d2", size = 16638601, upload-time = "2025-10-15T16:17:14.391Z" },
    { url = "https://files.pythonhosted.org/packages/93/87/1c1de269f002ff0a41173fe01dcc925f4ecff59264cd8f96cf3b60d12c9b/numpy-2.3.4-cp314-cp314-musllinux_1_2_aarch64.whl", hash = "sha256:15fb27364ed84114438fff8aaf998c9e19adbeba08c0b75409f8c452a8692c52", size = 16074219, upload-time = "2025-10-15T16:17:17.058Z" },
    { url = "https://files.pythonhosted.org/packages/cd/28/18f72ee77408e40a76d691001ae599e712ca2a47ddd2c4f695b16c65f077/numpy-2.3.4-cp314-cp314-musllinux_1_2_x86_64.whl", hash = "sha256:85d9fb2d8cd998c84d13a79a09cc0c1091648e848e4e6249b0ccd7f6b487fa26", size = 18576702, upload-time = "2025-10-15T16:17:19.379Z" },
    { url = "https://files.pythonhosted.org/packages/c3/76/95650169b465ececa8cf4b2e8f6df255d4bf662775e797ade2025cc51ae6/numpy-2.3.4-cp314-cp314-win32.whl", hash = "sha256:e73d63fd04e3a9d6bc187f5455d81abfad05660b212c8804bf3b407e984cd2bc", size = 6337136, upload-time = "2025-10-15T16:17:22.886Z" },
    { url = "https://files.pythonhosted.org/packages/dc/89/a231a5c43ede5d6f77ba4a91e915a87dea4aeea76560ba4d2bf185c683f0/numpy-2.3.4-cp314-cp314-win_amd64.whl", hash = "sha256:3da3491cee49cf16157e70f607c03a217ea6647b1cea4819c4f48e53d49139b9", size = 12920542, upload-time = "2025-10-15T16:17:24.783Z" },
    { url = "https://files.pythonhosted.org/packages/0d/0c/ae9434a888f717c5ed2ff2393b3f344f0ff6f1c793519fa0c540461dc530/numpy-2.3.4-cp314-cp314-win_arm64.whl", hash = "sha256:6d9cd732068e8288dbe2717177320723ccec4fb064123f0caf9bbd90ab5be868", size = 10480213, upload-time = "2025-10-15T16:17:26.935Z" },
    { url = "https://files.pythonhosted.org/packages/83/4b/c4a5f0841f92536f6b9592694a5b5f68c9ab37b775ff342649eadf9055d3/numpy-2.3.4-cp314-cp314t-macosx_10_15_x86_64.whl", hash = "sha256:22758999b256b595cf0b1d102b133bb61866ba5ceecf15f759623b64c020c9ec", size = 21052280, upload-time = "2025-10-15T16:17:29.638Z" },
    { url = "https://files.pythonhosted.org/packages/3e/80/90308845fc93b984d2cc96d83e2324ce8ad1fd6efea81b324cba4b673854/numpy-2.3.4-cp314-cp314t-macosx_11_0_arm64.whl", hash = "sha256:9cb177bc55b010b19798dc5497d540dea67fd13a8d9e882b2dae71de0cf09eb3", size = 14302930, upload-time = "2025-10-15T16:17:32.384Z" },
    { url = "https://files.pythonhosted.org/packages/3d/4e/07439f22f2a3b247cec4d63a713faae55e1141a36e77fb212881f7cda3fb/numpy-2.3.4-cp314-cp314t-macosx_14_0_arm64.whl", hash = "sha256:0f2bcc76f1e05e5ab58893407c63d90b2029908fa41f9f1cc51eecce936c3365", size = 5231504, upload-time = "2025-10-15T16:17:34.515Z" },
    { url = "https://files.pythonhosted.org/packages/ab/de/1e11f2547e2fe3d00482b19721855348b94ada8359aef5d40dd57bfae9df/numpy-2.3.4-cp314-cp314t-macosx_14_0_x86_64.whl", hash = "sha256:8dc20bde86802df2ed8397a08d793da0ad7a5fd4ea3ac85d757bf5dd4ad7c252", size = 6739405, upload-time = "2025-10-15T16:17:36.128Z" },
    { url = "https://files.pythonhosted.org/packages/3b/40/8cd57393a26cebe2e923005db5134a946c62fa56a1087dc7c478f3e30837/numpy-2.3.4-cp314-cp314t-manylinux_2_27_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:5e199c087e2aa71c8f9ce1cb7a8e10677dc12457e7cc1be4798632da37c3e86e", size = 14354866, upload-time = "2025-10-15T16:17:38.884Z" },
    { url = "https://files.pythonhosted.org/packages/93/39/5b3510f023f96874ee6fea2e40dfa99313a00bf3ab779f3c92978f34aace/numpy-2.3.4-cp314-cp314t-manylinux_2_27_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:85597b2d25ddf655495e2363fe044b0ae999b75bc4d630dc0d886484b03a5eb0", size = 16703296, upload-time = "2025-10-15T16:17:41.564Z" },
    { url = "https://files.pythonhosted.org/packages/41/0d/19bb163617c8045209c1996c4e427bccbc4bbff1e2c711f39203c8ddbb4a/numpy-2.3.4-cp314-cp314t-musllinux_1_2_aarch64.whl", hash = "sha256:04a69abe45b49c5955923cf2c407843d1c85013b424ae8a560bba16c92fe44a0", size = 16136046, upload-time = "2025-10-15T16:17:43.901Z" },
    { url = "https://files.pythonhosted.org/packages/e2/c1/6dba12fdf68b02a21ac411c9df19afa66bed2540f467150ca64d246b463d/numpy-2.3.4-cp314-cp314t-musllinux_1_2_x86_64.whl", hash = "sha256:e1708fac43ef8b419c975926ce1eaf793b0c13b7356cfab6ab0dc34c0a02ac0f", size = 18652691, upload-time = "2025-10-15T16:17:46.247Z" },
    { url = "https://files.pythonhosted.org/packages/f8/73/f85056701dbbbb910c51d846c58d29fd46b30eecd2b6ba760fc8b8a1641b/numpy-2.3.4-cp314-cp314t-win32.whl", hash = "sha256:863e3b5f4d9915aaf1b8ec79ae560ad21f0b8d5e3adc31e73126491bb86dee1d", size = 6485782, upload-time = "2025-10-15T16:17:48.872Z" },
    { url = "https://files.pythonhosted.org/packages/17/90/28fa6f9865181cb817c2471ee65678afa8a7e2a1fb16141473d5fa6bacc3/numpy-2.3.4-cp314-cp314t-win_amd64.whl", hash = "sha256:962064de37b9aef801d33bc579690f8bfe6c5e70e29b61783f60bcba838a14d6", size = 13113301, upload-time = "2025-10-15T16:17:50.938Z" },
    { url = "https://files.pythonhosted.org/packages/54/23/08c002201a8e7e1f9afba93b97deceb813252d9cfd0d3351caed123dcf97/numpy-2.3.4-cp314-cp314t-win_arm64.whl", hash = "sha256:8b5a9a39c45d852b62693d9b3f3e0fe052541f804296ff401a72a1b60edafb29", size = 10547532, upload-time = "2025-10-15T16:17:53.48Z" },
    { url = "https://files.pythonhosted.org/packages/b1/b6/64898f51a86ec88ca1257a59c1d7fd077b60082a119affefcdf1dd0df8ca/numpy-2.3.4-pp311-pypy311_pp73-macosx_10_15_x86_64.whl", hash = "sha256:6e274603039f924c0fe5cb73438fa9246699c78a6df1bd3decef9ae592ae1c05", size = 21131552, upload-time = "2025-10-15T16:17:55.845Z" },
    { url = "https://files.pythonhosted.org/packages/ce/4c/f135dc6ebe2b6a3c77f4e4838fa63d350f85c99462012306ada1bd4bc460/numpy-2.3.4-pp311-pypy311_pp73-macosx_11_0_arm64.whl", hash = "sha256:d149aee5c72176d9ddbc6803aef9c0f6d2ceeea7626574fc68518da5476fa346", size = 14377796, upload-time = "2025-10-15T16:17:58.308Z" },
    { url = "https://files.pythonhosted.org/packages/d0/a4/f33f9c23fcc13dd8412fc8614559b5b797e0aba9d8e01dfa8bae10c84004/numpy-2.3.4-pp311-pypy311_pp73-macosx_14_0_arm64.whl", hash = "sha256:6d34ed9db9e6395bb6cd33286035f73a59b058169733a9db9f85e650b88df37e", size = 5306904, upload-time = "2025-10-15T16:18:00.596Z" },
    { url = "https://files.pythonhosted.org/packages/28/af/c44097f25f834360f9fb960fa082863e0bad14a42f36527b2a121abdec56/numpy-2.3.4-pp311-pypy311_pp73-macosx_14_0_x86_64.whl", hash = "sha256:fdebe771ca06bb8d6abce84e51dca9f7921fe6ad34a0c914541b063e9a68928b", size = 6819682, upload-time = "2025-10-15T16:18:02.32Z" },
    { url = "https://files.pythonhosted.org/packages/c5/8c/cd283b54c3c2b77e188f63e23039844f56b23bba1712318288c13fe86baf/numpy-2.3.4-pp311-pypy311_pp73-manylinux_2_27_aarch64.manylinux_2_28_aarch64.whl", hash = "sha256:957e92defe6c08211eb77902253b14fe5b480ebc5112bc741fd5e9cd0608f847", size = 14422300, upload-time = "2025-10-15T16:18:04.271Z" },
    { url = "https://files.pythonhosted.org/packages/b0/f0/8404db5098d92446b3e3695cf41c6f0ecb703d701cb0b7566ee2177f2eee/numpy-2.3.4-pp311-pypy311_pp73-manylinux_2_27_x86_64.manylinux_2_28_x86_64.whl", hash = "sha256:13b9062e4f5c7ee5c7e5be96f29ba71bc5a37fed3d1d77c37390ae00724d296d", size = 16760806, upload-time = "2025-10-15T16:18:06.668Z" },
    { url = "https://files.pythonhosted.org/packages/95/8e/2844c3959ce9a63acc7c8e50881133d86666f0420bcde695e115ced0920f/numpy-2.3.4-pp311-pypy311_pp73-win_amd64.whl", hash = "sha256:81b3a59793523e552c4a96109dde028aa4448ae06ccac5a76ff6532a85558a7f", size = 12973130, upload-time = "2025-10-15T16:18:09.397Z" },
]

[[package]]
name = "orjson"
version = "3.11.4"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "https://files.pythonhosted.org/packages/c6/fe/ed708782d6709cc60eb4c2d8a361a440661f74134675c72990f2c48c785f/orjson-3.11.4.tar.gz", hash = "sha256:39485f4ab4c9b30a3943cfe99e1a213c4776fb69e8abd68f66b83d5a0b0fdc6d", size = 5945188, upload-time = "2025-10-24T15:50:38.027Z" }
wheels = [
    { url = "https://files.pythonhosted.org/packages/e0/30/5aed63d5af1c8b02fbd2a8d83e2a6c8455e30504c50dbf08c8b51403d873/orjson-3.11.4-cp310-cp310-macosx_10_15_x86_64.macosx_11_0_arm64.macosx_10_15_universal2.whl", hash = "sha256:e3aa2118a3ece0d25489cbe48498de8a5d580e42e8d9979f65bf47900a15aba1", size = 243870, upload-time = "2025-10-24T15:48:28.908Z" },
    { url = "https://files.pythonhosted.org/packages/44/1f/da46563c08bef33c41fd63c660abcd2184b4d2b950c8686317d03b9f5f0c/orjson-3.11.4-cp310-cp310-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:a69ab657a4e6733133a3dca82768f2f8b884043714e8d2b9ba9f52b6efef5c44", size = 130622, upload-time = "2025-10-24T15:48:31.361Z" },
    { url = "https://files.pythonhosted.org/packages/02/bd/b551a05d0090eab0bf8008a13a14edc0f3c3e0236aa6f5b697760dd2817b/orjson-3.11.4-cp310-cp310-manylinux_2_17_armv7l.manylinux2014_armv7l.whl", hash = "sha256:3740bffd9816fc0326ddc406098a3a8f387e42223f5f455f2a02a9f834ead80c", size = 129344, upload-time = "2025-10-24T15:48:32.71Z" },
    { url = "https://files.pythonhosted.org/packages/87/6c/9ddd5e609f443b2548c5e7df3c44d0e86df2c68587a0e20c50018cdec535/orjson-3.11.4-cp310-cp310-manylinux_2_17_i686.manylinux2014_i686.whl", hash = "sha256:65fd2f5730b1bf7f350c6dc896173d3460d235c4be007af73986d7cd9a2acd23", size = 136633, upload-time = "2025-10-24T15:48:34.128Z" },
    { url = "https://files.pythonhosted.org/packages/95/f2/9f04f2874c625a9fb60f6918c33542320661255323c272e66f7dcce14df2/orjson-3.11.4-cp310-cp310-manylinux_2_17_ppc64le.manylinux2014_ppc64le.whl", hash = "sha256:9fdc3ae730541086158d549c97852e2eea6820665d4faf0f41bf99df41bc11ea", size = 137695, upload-time = "2025-10-24T15:48:35.654Z" },
    { url = "https://files.pythonhosted.org/packages/d2/c2/c7302afcbdfe8a891baae0e2cee091583a30e6fa613e8bdf33b0e9c8a8c7/orjson-3.11.4-cp310-cp310-manylinux_2_17_s390x.manylinux2014_s390x.whl", hash = "sha256:e10b4d65901da88845516ce9f7f9736f9638d19a1d483b3883dc0182e6e5edba", size = 136879, upload-time = "2025-10-24T15:48:37.483Z" },
    { url = "https://files.pythonhosted.org/packages/c6/3a/b31c8f0182a3e27f48e703f46e61bb769666cd0dac4700a73912d07a1417/orjson-3.11.4-cp310-cp310-manylinux_2_17_x86_64.manylinux2014_x86_64.whl", hash = "sha256:fb6a03a678085f64b97f9d4a9ae69376ce91a3a9e9b56a82b1580d8e1d501aff", size = 136374, upload-time = "2025-10-24T15:48:38.624Z" },
    { url = "https://files.pythonhosted.org/packages/29/d0/fd9ab96841b090d281c46df566b7f97bc6c8cd9aff3f3ebe99755895c406/orjson-3.11.4-cp310-cp310-musllinux_1_2_aarch64.whl", hash = "sha256:2c82e4f0b1c712477317434761fbc28b044c838b6b1240d895607441412371ac", size = 140519, upload-time = "2025-10-24T15:48:39.756Z" },
    { url = "https://files.pythonhosted.org/packages/d6/ce/36eb0f15978bb88e33a3480e1a3fb891caa0f189ba61ce7713e0ccdadabf/orjson-3.11.4-cp310-cp310-musllinux_1_2_armv7l.whl", hash = "sha256:d58c166a18f44cc9e2bad03a327dc2d1a3d2e85b847133cfbafd6bfc6719bd79", size = 406522, upload-time = "2025-10-24T15:48:41.198Z" },
    { url = "https://files.pythonhosted.org/packages/85/11/e8af3161a288f5c6a00c188fc729c7ba193b0cbc07309a1a29c004347c30/orjson-3.11.4-cp310-cp310-musllinux_1_2_i686.whl", hash = "sha256:94f206766bf1ea30e1382e4890f763bd1eefddc580e08fec1ccdc20ddd95c827", size = 149790, upload-time = "2025-10-24T15:48:42.664Z" },
    { url = "https://files.pythonhosted.org/packages/ea/96/209d52db0cf1e10ed48d8c194841e383e23c2ced5a2ee766649fe0e32d02/orjson-3.11.4-cp310-cp310-musllinux_1_2_x86_64.whl", hash = "sha256:41bf25fb39a34cf8edb4398818523277ee7096689db352036a9e8437f2f3ee6b", size = 140040, upload-time = "2025-10-24T15:48:44.042Z" },
    { url = "https://files.pythonhosted.org/packages/ef/0e/526db1395ccb74c3d59ac1660b9a325017096dc5643086b38f27662b4add/orjson-3.11.4-cp310-cp310-win32.whl", hash = "sha256:fa9627eba4e82f99ca6d29bc967f09aba446ee2b5a1ea728949ede73d313f5d3", size = 135955, upload-time = "2025-10-24T15:48:45.495Z" },
    { url = "https://files.pythonhosted.org/packages/e6/69/18a778c9de3702b19880e73c9866b91cc85f904b885d816ba1ab318b223c/orjson-3.11.4-cp310-cp310-win_amd64.whl", hash = "sha256:23ef7abc7fca96632d8174ac115e668c1e931b8fe4dde586e92a500bf1914dcc", size = 131577, upload-time = "2025-10-24T15:48:46.609Z" },
    { url = "https://files.pythonhosted.org/packages/63/1d/1ea6005fffb56715fd48f632611e163d1604e8316a5bad2288bee9a1c9eb/orjson-3.11.4-cp311-cp311-macosx_10_15_x86_64.macosx_11_0_arm64.macosx_10_15_universal2.whl", hash = "sha256:5e59d23cd93ada23ec59a96f215139753fbfe3a4d989549bcb390f8c00370b39", size = 243498, upload-time = "2025-10-24T15:48:48.101Z" },
    { url = "https://files.pythonhosted.org/packages/37/d7/ffed10c7da677f2a9da307d491b9eb1d0125b0307019c4ad3d665fd31f4f/orjson-3.11.4-cp311-cp311-macosx_15_0_arm64.whl", hash = "sha256:5c3aedecfc1beb988c27c79d52ebefab93b6c3921dbec361167e6559aba2d36d", size = 128961, upload-time = "2025-10-24T15:48:49.571Z" },
    { url = "https://files.pythonhosted.org/packages/a2/96/3e4d10a18866d1368f73c8c44b7fe37cc8a15c32f2a7620be3877d4c55a3/orjson-3.11.4-cp311-cp311-manylinux_2_17_aarch64.manylinux2014_aarch64.whl", hash = "sha256:da9e5301f1c2caa2a9a4a303480d79c9ad73560b2e7761de742ab39fe59d9175", size = 130321, upload-time = "2025-10-24T15:48:50.713Z" },
