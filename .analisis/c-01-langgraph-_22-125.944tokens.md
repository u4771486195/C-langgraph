    # Then any time a user tries to access this thread or runs within the thread,
    # we can filter by owner
    metadata = value.setdefault("metadata", {})
    metadata["owner"] = ctx.user.identity
    return {"owner": ctx.user.identity}

# Reading a thread. Since this is also more specific than the generic @auth.on handler, and the @auth.on.threads handler,
# it will take precedence for any "read" actions on the "threads" resource
@auth.on.threads.read
async def on_thread_read(
    ctx: Auth.types.AuthContext,
    value: Auth.types.threads.read.value
):
    # Since we are reading (and not creating) a thread,
    # we don't need to set metadata. We just need to
    # return a filter to ensure users can only see their own threads
    return {"owner": ctx.user.identity}

# Run creation, streaming, updates, etc.
# This takes precedenceover the generic @auth.on handler and the @auth.on.threads handler
@auth.on.threads.create_run
async def on_run_create(
    ctx: Auth.types.AuthContext,
    value: Auth.types.threads.create_run.value
):
    metadata = value.setdefault("metadata", {})
    metadata["owner"] = ctx.user.identity
    # Inherit thread's access control
    return {"owner": ctx.user.identity}

# Assistant creation
@auth.on.assistants.create
async def on_assistant_create(
    ctx: Auth.types.AuthContext,
    value: Auth.types.assistants.create.value
):
    if "assistants:create" not in ctx.permissions:
        raise Auth.exceptions.HTTPException(
            status_code=403,
            detail="User lacks the required permissions."
        )
```

:::

:::js

```typescript
import { Auth, HTTPException } from "@langchain/langgraph-sdk/auth";

export const auth = new Auth()
  .authenticate(async (request: Request) => ({
    identity: "user-123",
    permissions: ["threads:write", "threads:read"],
  }))
  .on("*", ({ event, user }) => {
    console.log(`Request for ${event} by ${user.identity}`);
    throw new HTTPException(403, { message: "Forbidden" });
  })

  // Matches the "threads" resource and all actions - create, read, update, delete, search
  // Since this is **more specific** than the generic `on("*")` handler, it will take precedence over the generic handler for all actions on the "threads" resource
  .on("threads", ({ permissions, value, user }) => {
    if (!permissions.includes("write")) {
      throw new HTTPException(403, {
        message: "User lacks the required permissions.",
      });
    }

    // Not all events do include `metadata` property in `value`.
    // So we need to add this type guard.
    if ("metadata" in value) {
      value.metadata ??= {};
      value.metadata.owner = user.identity;
    }

    return { owner: user.identity };
  })

  // Thread creation. This will match only on thread create actions.
  // Since this is **more specific** than both the generic `on("*")` handler and the `on("threads")` handler, it will take precedence for any "create" actions on the "threads" resources
  .on("threads:create", ({ value, user, permissions }) => {
    if (!permissions.includes("write")) {
      throw new HTTPException(403, {
        message: "User lacks the required permissions.",
      });
    }

    // Setting metadata on the thread being created will ensure that the resource contains an "owner" field
    // Then any time a user tries to access this thread or runs within the thread,
    // we can filter by owner
    value.metadata ??= {};
    value.metadata.owner = user.identity;

    return { owner: user.identity };
  })

  // Reading a thread. Since this is also more specific than the generic `on("*")` handler, and the `on("threads")` handler,
  .on("threads:read", ({ user }) => {
    // Since we are reading (and not creating) a thread,
    // we don't need to set metadata. We just need to
    // return a filter to ensure users can only see their own threads.
    return { owner: user.identity };
  })

  // Run creation, streaming, updates, etc.
  // This takes precedence over the generic `on("*")` handler and the `on("threads")` handler
  .on("threads:create_run", ({ value, user }) => {
    value.metadata ??= {};
    value.metadata.owner = user.identity;

    return { owner: user.identity };
  })

  // Assistant creation. This will match only on assistant create actions.
  // Since this is **more specific** than both the generic `on("*")` handler and the `on("assistants")` handler, it will take precedence for any "create" actions on the "assistants" resources
  .on("assistants:create", ({ value, user, permissions }) => {
    if (!permissions.includes("assistants:create")) {
      throw new HTTPException(403, {
        message: "User lacks the required permissions.",
      });
    }

    // Setting metadata on the assistant being created will ensure that the resource contains an "owner" field.
    // Then any time a user tries to access this assistant, we can filter by owner
    value.metadata ??= {};
    value.metadata.owner = user.identity;

    return { owner: user.identity };
  });
```

:::

Notice that we are mixing global and resource-specific handlers in the above example. Since each request is handled by the most specific handler, a request to create a `thread` would match the `on_thread_create` handler but NOT the `reject_unhandled_requests` handler. A request to `update` a thread, however would be handled by the global handler, since we don't have a more specific handler for that resource and action.

### Filter Operations {#filter-operations}

:::python
Authorization handlers can return different types of values:

- `None` and `True` mean "authorize access to all underling resources"
- `False` means "deny access to all underling resources (raises a 403 exception)"
- A metadata filter dictionary will restrict access to resources

A filter dictionary is a dictionary with keys that match the resource metadata. It supports three operators:

- The default value is a shorthand for exact match, or "$eq", below. For example, `{"owner": user_id}` will include only resources with metadata containing `{"owner": user_id}`
- `$eq`: Exact match (e.g., `{"owner": {"$eq": user_id}}`) - this is equivalent to the shorthand above, `{"owner": user_id}`
- `$contains`: List membership (e.g., `{"allowed_users": {"$contains": user_id}}`) The value here must be an element of the list. The metadata in the stored resource must be a list/container type.

A dictionary with multiple keys is treated using a logical `AND` filter. For example, `{"owner": org_id, "allowed_users": {"$contains": user_id}}` will only match resources with metadata whose "owner" is `org_id` and whose "allowed_users" list contains `user_id`.
See the reference [here](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.FilterType) for more information.
:::

:::js
Authorization handlers can return different types of values:

- `null` and `true` mean "authorize access to all underling resources"
- `false` means "deny access to all underling resources (raises a 403 exception)"
- A metadata filter object will restrict access to resources

A filter object is an object with keys that match the resource metadata. It supports three operators:

- The default value is a shorthand for exact match, or "$eq", below. For example, `{ owner: userId}` will include only resources with metadata containing `{ owner: userId }`
- `$eq`: Exact match (e.g., `{ owner: { $eq: userId } }`) - this is equivalent to the shorthand above, `{ owner: userId }`
- `$contains`: List membership (e.g., `{ allowedUsers: { $contains: userId} }`) The value here must be an element of the list. The metadata in the stored resource must be a list/container type.

An object with multiple keys is treated using a logical `AND` filter. For example, `{ owner: orgId, allowedUsers: { $contains: userId} }` will only match resources with metadata whose "owner" is `orgId` and whose "allowedUsers" list contains `userId`.
See the reference [here](../cloud/reference/sdk/typescript_sdk_ref.md#auth.types.FilterType) for more information.
:::

## Common Access Patterns

Here are some typical authorization patterns:

### Single-Owner Resources

This common pattern lets you scope all threads, assistants, crons, and runs to a single user. It's useful for common single-user use cases like regular chatbot-style apps.

:::python

```python
@auth.on
async def owner_only(ctx: Auth.types.AuthContext, value: dict):
    metadata = value.setdefault("metadata", {})
    metadata["owner"] = ctx.user.identity
    return {"owner": ctx.user.identity}
```

:::

:::js

```typescript
export const auth = new Auth()
  .authenticate(async (request: Request) => ({
    identity: "user-123",
    permissions: ["threads:write", "threads:read"],
  }))
  .on("*", ({ value, user }) => {
    if ("metadata" in value) {
      value.metadata ??= {};
      value.metadata.owner = user.identity;
    }
    return { owner: user.identity };
  });
```

:::

### Permission-based Access

This pattern lets you control access based on **permissions**. It's useful if you want certain roles to have broader or more restricted access to resources.

:::python

```python
# In your auth handler:
@auth.authenticate
async def authenticate(headers: dict) -> Auth.types.MinimalUserDict:
    ...
    return {
        "identity": "user-123",
        "is_authenticated": True,
        "permissions": ["threads:write", "threads:read"]  # Define permissions in auth
    }

def _default(ctx: Auth.types.AuthContext, value: dict):
    metadata = value.setdefault("metadata", {})
    metadata["owner"] = ctx.user.identity
    return {"owner": ctx.user.identity}

@auth.on.threads.create
async def create_thread(ctx: Auth.types.AuthContext, value: dict):
    if "threads:write" not in ctx.permissions:
        raise Auth.exceptions.HTTPException(
            status_code=403,
            detail="Unauthorized"
        )
    return _default(ctx, value)


@auth.on.threads.read
async def rbac_create(ctx: Auth.types.AuthContext, value: dict):
    if "threads:read" not in ctx.permissions and "threads:write" not in ctx.permissions:
        raise Auth.exceptions.HTTPException(
            status_code=403,
            detail="Unauthorized"
        )
    return _default(ctx, value)
```

:::

:::js

```typescript
import { Auth, HTTPException } from "@langchain/langgraph-sdk/auth";

export const auth = new Auth()
  .authenticate(async (request: Request) => ({
    identity: "user-123",
    // Define permissions in auth
    permissions: ["threads:write", "threads:read"],
  }))
  .on("threads:create", ({ value, user, permissions }) => {
    if (!permissions.includes("threads:write")) {
      throw new HTTPException(403, { message: "Unauthorized" });
    }

    if ("metadata" in value) {
      value.metadata ??= {};
      value.metadata.owner = user.identity;
    }
    return { owner: user.identity };
  })
  .on("threads:read", ({ user, permissions }) => {
    if (
      !permissions.includes("threads:read") &&
      !permissions.includes("threads:write")
    ) {
      throw new HTTPException(403, { message: "Unauthorized" });
    }

    return { owner: user.identity };
  });
```

:::

## Supported Resources

LangGraph provides three levels of authorization handlers, from most general to most specific:

:::python

1. **Global Handler** (`@auth.on`): Matches all resources and actions
2. **Resource Handler** (e.g., `@auth.on.threads`, `@auth.on.assistants`, `@auth.on.crons`): Matches all actions for a specific resource
3. **Action Handler** (e.g., `@auth.on.threads.create`, `@auth.on.threads.read`): Matches a specific action on a specific resource

The most specific matching handler will be used. For example, `@auth.on.threads.create` takes precedence over `@auth.on.threads` for thread creation.
If a more specific handler is registered, the more general handler will not be called for that resource and action.
:::

:::js

1. **Global Handler** (`on("*")`): Matches all resources and actions
2. **Resource Handler** (e.g., `on("threads")`, `on("assistants")`, `on("crons")`): Matches all actions for a specific resource
3. **Action Handler** (e.g., `on("threads:create")`, `on("threads:read")`): Matches a specific action on a specific resource

The most specific matching handler will be used. For example, `on("threads:create")` takes precedence over `on("threads")` for thread creation.
If a more specific handler is registered, the more general handler will not be called for that resource and action.
:::

:::python
???+ tip "Type Safety"
Each handler has type hints available for its `value` parameter. For example:

    ```python
    @auth.on.threads.create
    async def on_thread_create(
        ctx: Auth.types.AuthContext,
        value: Auth.types.on.threads.create.value  # Specific type for thread creation
    ):
        ...

    @auth.on.threads
    async def on_threads(
        ctx: Auth.types.AuthContext,
        value: Auth.types.on.threads.value  # Union type of all thread actions
    ):
        ...

    @auth.on
    async def on_all(
        ctx: Auth.types.AuthContext,
        value: dict  # Union type of all possible actions
    ):
        ...
    ```

    More specific handlers provide better type hints since they handle fewer action types.

:::

#### Supported actions and types {#supported-actions}

Here are all the supported action handlers:

:::python
| Resource | Handler | Description | Value Type |
|----------|---------|-------------|------------|
| **Threads** | `@auth.on.threads.create` | Thread creation | [`ThreadsCreate`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.ThreadsCreate) |
| | `@auth.on.threads.read` | Thread retrieval | [`ThreadsRead`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.ThreadsRead) |
| | `@auth.on.threads.update` | Thread updates | [`ThreadsUpdate`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.ThreadsUpdate) |
| | `@auth.on.threads.delete` | Thread deletion | [`ThreadsDelete`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.ThreadsDelete) |
| | `@auth.on.threads.search` | Listing threads | [`ThreadsSearch`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.ThreadsSearch) |
| | `@auth.on.threads.create_run` | Creating or updating a run | [`RunsCreate`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.RunsCreate) |
| **Assistants** | `@auth.on.assistants.create` | Assistant creation | [`AssistantsCreate`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.AssistantsCreate) |
| | `@auth.on.assistants.read` | Assistant retrieval | [`AssistantsRead`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.AssistantsRead) |
| | `@auth.on.assistants.update` | Assistant updates | [`AssistantsUpdate`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.AssistantsUpdate) |
| | `@auth.on.assistants.delete` | Assistant deletion | [`AssistantsDelete`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.AssistantsDelete) |
| | `@auth.on.assistants.search` | Listing assistants | [`AssistantsSearch`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.AssistantsSearch) |
| **Crons** | `@auth.on.crons.create` | Cron job creation | [`CronsCreate`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.CronsCreate) |
| | `@auth.on.crons.read` | Cron job retrieval | [`CronsRead`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.CronsRead) |
| | `@auth.on.crons.update` | Cron job updates | [`CronsUpdate`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.CronsUpdate) |
| | `@auth.on.crons.delete` | Cron job deletion | [`CronsDelete`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.CronsDelete) |
| | `@auth.on.crons.search` | Listing cron jobs | [`CronsSearch`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.CronsSearch) |
:::

:::js
| Resource | Event | Description | Value Type |
| -------------- | -------------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| **Threads** | `threads:create` | Thread creation | [`ThreadsCreate`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#threadscreate) |
| | `threads:read` | Thread retrieval | [`ThreadsRead`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#threadsread) |
| | `threads:update` | Thread updates | [`ThreadsUpdate`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#threadsupdate) |
| | `threads:delete` | Thread deletion | [`ThreadsDelete`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#threadsdelete) |
| | `threads:search` | Listing threads | [`ThreadsSearch`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#threadssearch) |
| | `threads:create_run` | Creating or updating a run | [`RunsCreate`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#threadscreate_run) |
| **Assistants** | `assistants:create` | Assistant creation | [`AssistantsCreate`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#assistantscreate) |
| | `assistants:read` | Assistant retrieval | [`AssistantsRead`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#assistantsread) |
| | `assistants:update` | Assistant updates | [`AssistantsUpdate`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#assistantsupdate) |
| | `assistants:delete` | Assistant deletion | [`AssistantsDelete`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#assistantsdelete) |
| | `assistants:search` | Listing assistants | [`AssistantsSearch`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#assistantssearch) |
| **Crons** | `crons:create` | Cron job creation | [`CronsCreate`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#cronscreate) |
| | `crons:read` | Cron job retrieval | [`CronsRead`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#cronsread) |
| | `crons:update` | Cron job updates | [`CronsUpdate`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#cronsupdate) |
| | `crons:delete` | Cron job deletion | [`CronsDelete`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#cronsdelete) |
| | `crons:search` | Listing cron jobs | [`CronsSearch`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#cronssearch) |
:::

???+ note "About Runs"

    Runs are scoped to their parent thread for access control. This means permissions are typically inherited from the thread, reflecting the conversational nature of the data model. All run operations (reading, listing) except creation are controlled by the thread's handlers.

    :::python
    There is a specific `create_run` handler for creating new runs because it had more arguments that you can view in the handler.
    :::

    :::js
    There is a specific `threads:create_run` handler for creating new runs because it had more arguments that you can view in the handler.
    :::

## Next Steps

For implementation details:

- Check out the introductory tutorial on [setting up authentication](../tutorials/auth/getting_started.md)
- See the how-to guide on implementing a [custom auth handlers](../how-tos/auth/custom_auth.md)

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/deployment_options.md
```md
---
search:
  boost: 2
---

# Deployment Options

## Free deployment

[Local](../tutorials/langgraph-platform/local-server.md): Deploy for local testing and development.

## Production deployment

There are 4 main options for deploying with the [LangGraph Platform](langgraph_platform.md):

1. [Cloud SaaS](#cloud-saas)

1. [Self-Hosted Data Plane](#self-hosted-data-plane)

1. [Self-Hosted Control Plane](#self-hosted-control-plane)

1. [Standalone Container](#standalone-container)


A quick comparison:

|                      | **Cloud SaaS** | **Self-Hosted Data Plane** | **Self-Hosted Control Plane** | **Standalone Container** |
|----------------------|----------------|----------------------------|-------------------------------|--------------------------|
| **[Control plane UI/API](../concepts/langgraph_control_plane.md)** | Yes | Yes | Yes | No |
| **CI/CD** | Managed internally by platform | Managed externally by you | Managed externally by you | Managed externally by you |
| **Data/compute residency** | LangChain's cloud | Your cloud | Your cloud | Your cloud |
| **LangSmith compatibility** | Trace to LangSmith SaaS | Trace to LangSmith SaaS | Trace to Self-Hosted LangSmith | Optional tracing |
| **[Pricing](https://www.langchain.com/pricing-langgraph-platform)** | Plus | Enterprise | Enterprise | Enterprise |

## Cloud SaaS

The [Cloud SaaS](./langgraph_cloud.md) deployment option is a fully managed model for deployment where we manage the [control plane](./langgraph_control_plane.md) and [data plane](./langgraph_data_plane.md) in our cloud. This option provides a simple way to deploy and manage your LangGraph Servers.

Connect your GitHub repositories to the platform and deploy your LangGraph Servers from the [control plane UI](./langgraph_control_plane.md#control-plane-ui). The build process (i.e. CI/CD) is managed internally by the platform.

For more information, please see:

* [Cloud SaaS Conceptual Guide](./langgraph_cloud.md)
* [How to deploy to Cloud SaaS](../cloud/deployment/cloud.md)

## Self-Hosted Data Plane

!!! info "Important"
    The Self-Hosted Data Plane deployment option requires an [Enterprise](../concepts/plans.md) plan.

The [Self-Hosted Data Plane](./langgraph_self_hosted_data_plane.md) deployment option is a "hybrid" model for deployment where we manage the [control plane](./langgraph_control_plane.md) in our cloud and you manage the [data plane](./langgraph_data_plane.md) in your cloud. This option provides a way to securely manage your data plane infrastructure, while offloading control plane management to us.

Build a Docker image using the [LangGraph CLI](./langgraph_cli.md) and deploy your LangGraph Server from the [control plane UI](./langgraph_control_plane.md#control-plane-ui).

Supported Compute Platforms: [Kubernetes](https://kubernetes.io/), [Amazon ECS](https://aws.amazon.com/ecs/) (coming soon!)

For more information, please see:

* [Self-Hosted Data Plane Conceptual Guide](./langgraph_self_hosted_data_plane.md)
* [How to deploy the Self-Hosted Data Plane](../cloud/deployment/self_hosted_data_plane.md)

## Self-Hosted Control Plane

!!! info "Important"
    The Self-Hosted Control Plane deployment option requires an [Enterprise](../concepts/plans.md) plan.

The [Self-Hosted Control Plane](./langgraph_self_hosted_control_plane.md) deployment option is a fully self-hosted model for deployment where you manage the [control plane](./langgraph_control_plane.md) and [data plane](./langgraph_data_plane.md) in your cloud. This option gives you full control and responsibility of the control plane and data plane infrastructure.

Build a Docker image using the [LangGraph CLI](./langgraph_cli.md) and deploy your LangGraph Server from the [control plane UI](./langgraph_control_plane.md#control-plane-ui).

Supported Compute Platforms: [Kubernetes](https://kubernetes.io/)

For more information, please see:

* [Self-Hosted Control Plane Conceptual Guide](./langgraph_self_hosted_control_plane.md)
* [How to deploy the Self-Hosted Control Plane](../cloud/deployment/self_hosted_control_plane.md)

## Standalone Container

The [Standalone Container](./langgraph_standalone_container.md) deployment option is the least restrictive model for deployment. Deploy standalone instances of a LangGraph Server in your cloud, using any of the [available](./plans.md) license options.

Build a Docker image using the [LangGraph CLI](./langgraph_cli.md) and deploy your LangGraph Server using the container deployment tooling of your choice. Images can be deployed to any compute platform.

For more information, please see:

* [Standalone Container Conceptual Guide](./langgraph_standalone_container.md)
* [How to deploy a Standalone Container](../cloud/deployment/standalone_container.md)

## Related

For more information, please see:

* [LangGraph Platform plans](./plans.md)
* [LangGraph Platform pricing](https://www.langchain.com/langgraph-platform-pricing)

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/double_texting.md
```md
---
search:
  boost: 2
---

# Double Texting

!!! info "Prerequisites"
    - [LangGraph Server](./langgraph_server.md)

Many times users might interact with your graph in unintended ways. 
For instance, a user may send one message and before the graph has finished running send a second message. 
More generally, users may invoke the graph a second time before the first run has finished.
We call this "double texting".

Currently, LangGraph only addresses this as part of [LangGraph Platform](langgraph_platform.md), not in the open source.
The reason for this is that in order to handle this we need to know how the graph is deployed, and since LangGraph Platform deals with deployment the logic needs to live there.
If you do not want to use LangGraph Platform, we describe the options we have implemented in detail below.

![](img/double_texting.png)

## Reject

This is the simplest option, this just rejects any follow-up runs and does not allow double texting. 
See the [how-to guide](../cloud/how-tos/reject_concurrent.md) for configuring the reject double text option.

## Enqueue

This is a relatively simple option which continues the first run until it completes the whole run, then sends the new input as a separate run. 
See the [how-to guide](../cloud/how-tos/enqueue_concurrent.md) for configuring the enqueue double text option.

## Interrupt

This option interrupts the current execution but saves all the work done up until that point. 
It then inserts the user input and continues from there. 

If you enable this option, your graph should be able to handle weird edge cases that may arise. 
For example, you could have called a tool but not yet gotten back a result from running that tool.
You may need to remove that tool call in order to not have a dangling tool call.

See the [how-to guide](../cloud/how-tos/interrupt_concurrent.md) for configuring the interrupt double text option.

## Rollback

This option interrupts the current execution AND rolls back all work done up until that point, including the original run input. It then sends the new user input in, basically as if it was the original input.

See the [how-to guide](../cloud/how-tos/rollback_concurrent.md) for configuring the rollback double text option.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/durable_execution.md
```md
---
search:
  boost: 2
---

# Durable Execution

**Durable execution** is a technique in which a process or workflow saves its progress at key points, allowing it to pause and later resume exactly where it left off. This is particularly useful in scenarios that require [human-in-the-loop](./human_in_the_loop.md), where users can inspect, validate, or modify the process before continuing, and in long-running tasks that might encounter interruptions or errors (e.g., calls to an LLM timing out). By preserving completed work, durable execution enables a process to resume without reprocessing previous steps -- even after a significant delay (e.g., a week later).

LangGraph's built-in [persistence](./persistence.md) layer provides durable execution for workflows, ensuring that the state of each execution step is saved to a durable store. This capability guarantees that if a workflow is interrupted -- whether by a system failure or for [human-in-the-loop](./human_in_the_loop.md) interactions -- it can be resumed from its last recorded state.

!!! tip

    If you are using LangGraph with a checkpointer, you already have durable execution enabled. You can pause and resume workflows at any point, even after interruptions or failures.
    To make the most of durable execution, ensure that your workflow is designed to be [deterministic](#determinism-and-consistent-replay) and [idempotent](#determinism-and-consistent-replay) and wrap any side effects or non-deterministic operations inside [tasks](./functional_api.md#task). You can use [tasks](./functional_api.md#task) from both the [StateGraph (Graph API)](./low_level.md) and the [Functional API](./functional_api.md).

## Requirements

To leverage durable execution in LangGraph, you need to:

1. Enable [persistence](./persistence.md) in your workflow by specifying a [checkpointer](./persistence.md#checkpointer-libraries) that will save workflow progress.
2. Specify a [thread identifier](./persistence.md#threads) when executing a workflow. This will track the execution history for a particular instance of the workflow.

:::python

3. Wrap any non-deterministic operations (e.g., random number generation) or operations with side effects (e.g., file writes, API calls) inside @[tasks][task] to ensure that when a workflow is resumed, these operations are not repeated for the particular run, and instead their results are retrieved from the persistence layer. For more information, see [Determinism and Consistent Replay](#determinism-and-consistent-replay).

:::

:::js

3. Wrap any non-deterministic operations (e.g., random number generation) or operations with side effects (e.g., file writes, API calls) inside @[tasks][task] to ensure that when a workflow is resumed, these operations are not repeated for the particular run, and instead their results are retrieved from the persistence layer. For more information, see [Determinism and Consistent Replay](#determinism-and-consistent-replay).

:::

## Determinism and Consistent Replay

When you resume a workflow run, the code does **NOT** resume from the **same line of code** where execution stopped; instead, it will identify an appropriate [starting point](#starting-points-for-resuming-workflows) from which to pick up where it left off. This means that the workflow will replay all steps from the [starting point](#starting-points-for-resuming-workflows) until it reaches the point where it was stopped.

As a result, when you are writing a workflow for durable execution, you must wrap any non-deterministic operations (e.g., random number generation) and any operations with side effects (e.g., file writes, API calls) inside [tasks](./functional_api.md#task) or [nodes](./low_level.md#nodes).

To ensure that your workflow is deterministic and can be consistently replayed, follow these guidelines:

- **Avoid Repeating Work**: If a [node](./low_level.md#nodes) contains multiple operations with side effects (e.g., logging, file writes, or network calls), wrap each operation in a separate **task**. This ensures that when the workflow is resumed, the operations are not repeated, and their results are retrieved from the persistence layer.
- **Encapsulate Non-Deterministic Operations:** Wrap any code that might yield non-deterministic results (e.g., random number generation) inside **tasks** or **nodes**. This ensures that, upon resumption, the workflow follows the exact recorded sequence of steps with the same outcomes.
- **Use Idempotent Operations**: When possible ensure that side effects (e.g., API calls, file writes) are idempotent. This means that if an operation is retried after a failure in the workflow, it will have the same effect as the first time it was executed. This is particularly important for operations that result in data writes. In the event that a **task** starts but fails to complete successfully, the workflow's resumption will re-run the **task**, relying on recorded outcomes to maintain consistency. Use idempotency keys or verify existing results to avoid unintended duplication, ensuring a smooth and predictable workflow execution.

:::python
For some examples of pitfalls to avoid, see the [Common Pitfalls](./functional_api.md#common-pitfalls) section in the functional API, which shows
how to structure your code using **tasks** to avoid these issues. The same principles apply to the @[StateGraph (Graph API)][StateGraph].
:::

:::js
For some examples of pitfalls to avoid, see the [Common Pitfalls](./functional_api.md#common-pitfalls) section in the functional API, which shows
how to structure your code using **tasks** to avoid these issues. The same principles apply to the @[StateGraph (Graph API)][StateGraph].
:::

## Durability modes

LangGraph supports three durability modes that allow you to balance performance and data consistency based on your application's requirements. The durability modes, from least to most durable, are as follows:

- [`"exit"`](#exit)
- [`"async"`](#async)
- [`"sync"`](#sync)

A higher durability mode add more overhead to the workflow execution.

!!! version-added "Added in version 0.6.0"

    Use the `durability` parameter instead of `checkpoint_during` (deprecated in v0.6.0) for persistence policy management:
    
    * `durability="async"` replaces `checkpoint_during=True`
    * `durability="exit"` replaces `checkpoint_during=False`
    
    for persistence policy management, with the following mapping:

    * `checkpoint_during=True` -> `durability="async"`
    * `checkpoint_during=False` -> `durability="exit"`

### `"exit"`

Changes are persisted only when graph execution completes (either successfully or with an error). This provides the best performance for long-running graphs but means intermediate state is not saved, so you cannot recover from mid-execution failures or interrupt the graph execution.

### `"async"`

Changes are persisted asynchronously while the next step executes. This provides good performance and durability, but there's a small risk that checkpoints might not be written if the process crashes during execution.

### `"sync"`

Changes are persisted synchronously before the next step starts. This ensures that every checkpoint is written before continuing execution, providing high durability at the cost of some performance overhead.

You can specify the durability mode when calling any graph execution method:

:::python

```python
graph.stream(
    {"input": "test"}, 
    durability="sync"
)
```

:::

## Using tasks in nodes

If a [node](./low_level.md#nodes) contains multiple operations, you may find it easier to convert each operation into a **task** rather than refactor the operations into individual nodes.

:::python
=== "Original"

    ```python
    from typing import NotRequired
    from typing_extensions import TypedDict
    import uuid

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.graph import StateGraph, START, END
    import requests

    # Define a TypedDict to represent the state
    class State(TypedDict):
        url: str
        result: NotRequired[str]

    def call_api(state: State):
        """Example node that makes an API request."""
        # highlight-next-line
        result = requests.get(state['url']).text[:100]  # Side-effect
        return {
            "result": result
        }

    # Create a StateGraph builder and add a node for the call_api function
    builder = StateGraph(State)
    builder.add_node("call_api", call_api)

    # Connect the start and end nodes to the call_api node
    builder.add_edge(START, "call_api")
    builder.add_edge("call_api", END)

    # Specify a checkpointer
    checkpointer = InMemorySaver()

    # Compile the graph with the checkpointer
    graph = builder.compile(checkpointer=checkpointer)

    # Define a config with a thread ID.
    thread_id = uuid.uuid4()
    config = {"configurable": {"thread_id": thread_id}}

    # Invoke the graph
    graph.invoke({"url": "https://www.example.com"}, config)
    ```

=== "With task"

    ```python
    from typing import NotRequired
    from typing_extensions import TypedDict
    import uuid

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.func import task
    from langgraph.graph import StateGraph, START, END
    import requests

    # Define a TypedDict to represent the state
    class State(TypedDict):
        urls: list[str]
        result: NotRequired[list[str]]


    @task
    def _make_request(url: str):
        """Make a request."""
        # highlight-next-line
        return requests.get(url).text[:100]

    def call_api(state: State):
        """Example node that makes an API request."""
        # highlight-next-line
        requests = [_make_request(url) for url in state['urls']]
        results = [request.result() for request in requests]
        return {
            "results": results
        }

    # Create a StateGraph builder and add a node for the call_api function
    builder = StateGraph(State)
    builder.add_node("call_api", call_api)

    # Connect the start and end nodes to the call_api node
    builder.add_edge(START, "call_api")
    builder.add_edge("call_api", END)

    # Specify a checkpointer
    checkpointer = InMemorySaver()

    # Compile the graph with the checkpointer
    graph = builder.compile(checkpointer=checkpointer)

    # Define a config with a thread ID.
    thread_id = uuid.uuid4()
    config = {"configurable": {"thread_id": thread_id}}

    # Invoke the graph
    graph.invoke({"urls": ["https://www.example.com"]}, config)
    ```

:::

:::js
=== "Original"

    ```typescript
    import { StateGraph, START, END } from "@langchain/langgraph";
    import { MemorySaver } from "@langchain/langgraph";
    import { v4 as uuidv4 } from "uuid";
    import { z } from "zod";

    // Define a Zod schema to represent the state
    const State = z.object({
      url: z.string(),
      result: z.string().optional(),
    });

    const callApi = async (state: z.infer<typeof State>) => {
      // highlight-next-line
      const response = await fetch(state.url);
      const text = await response.text();
      const result = text.slice(0, 100); // Side-effect
      return {
        result,
      };
    };

    // Create a StateGraph builder and add a node for the callApi function
    const builder = new StateGraph(State)
      .addNode("callApi", callApi)
      .addEdge(START, "callApi")
      .addEdge("callApi", END);

    // Specify a checkpointer
    const checkpointer = new MemorySaver();

    // Compile the graph with the checkpointer
    const graph = builder.compile({ checkpointer });

    // Define a config with a thread ID.
    const threadId = uuidv4();
    const config = { configurable: { thread_id: threadId } };

    // Invoke the graph
    await graph.invoke({ url: "https://www.example.com" }, config);
    ```

=== "With task"

    ```typescript
    import { StateGraph, START, END } from "@langchain/langgraph";
    import { MemorySaver } from "@langchain/langgraph";
    import { task } from "@langchain/langgraph";
    import { v4 as uuidv4 } from "uuid";
    import { z } from "zod";

    // Define a Zod schema to represent the state
    const State = z.object({
      urls: z.array(z.string()),
      results: z.array(z.string()).optional(),
    });

    const makeRequest = task("makeRequest", async (url: string) => {
      // highlight-next-line
      const response = await fetch(url);
      const text = await response.text();
      return text.slice(0, 100);
    });

    const callApi = async (state: z.infer<typeof State>) => {
      // highlight-next-line
      const requests = state.urls.map((url) => makeRequest(url));
      const results = await Promise.all(requests);
      return {
        results,
      };
    };

    // Create a StateGraph builder and add a node for the callApi function
    const builder = new StateGraph(State)
      .addNode("callApi", callApi)
      .addEdge(START, "callApi")
      .addEdge("callApi", END);

    // Specify a checkpointer
    const checkpointer = new MemorySaver();

    // Compile the graph with the checkpointer
    const graph = builder.compile({ checkpointer });

    // Define a config with a thread ID.
    const threadId = uuidv4();
    const config = { configurable: { thread_id: threadId } };

    // Invoke the graph
    await graph.invoke({ urls: ["https://www.example.com"] }, config);
    ```

:::

## Resuming Workflows

Once you have enabled durable execution in your workflow, you can resume execution for the following scenarios:

:::python

- **Pausing and Resuming Workflows:** Use the @[interrupt][interrupt] function to pause a workflow at specific points and the @[Command] primitive to resume it with updated state. See [**Human-in-the-Loop**](./human_in_the_loop.md) for more details.
- **Recovering from Failures:** Automatically resume workflows from the last successful checkpoint after an exception (e.g., LLM provider outage). This involves executing the workflow with the same thread identifier by providing it with a `None` as the input value (see this [example](../how-tos/use-functional-api.md#resuming-after-an-error) with the functional API).

  :::

:::js

- **Pausing and Resuming Workflows:** Use the @[interrupt][interrupt] function to pause a workflow at specific points and the @[Command] primitive to resume it with updated state. See [**Human-in-the-Loop**](./human_in_the_loop.md) for more details.
- **Recovering from Failures:** Automatically resume workflows from the last successful checkpoint after an exception (e.g., LLM provider outage). This involves executing the workflow with the same thread identifier by providing it with a `null` as the input value (see this [example](../how-tos/use-functional-api.md#resuming-after-an-error) with the functional API).

  :::

## Starting Points for Resuming Workflows

:::python

- If you're using a @[StateGraph (Graph API)][StateGraph], the starting point is the beginning of the [**node**](./low_level.md#nodes) where execution stopped.
- If you're making a subgraph call inside a node, the starting point will be the **parent** node that called the subgraph that was halted.
  Inside the subgraph, the starting point will be the specific [**node**](./low_level.md#nodes) where execution stopped.
- If you're using the Functional API, the starting point is the beginning of the [**entrypoint**](./functional_api.md#entrypoint) where execution stopped.

  :::

:::js

- If you're using a [StateGraph (Graph API)](./low_level.md), the starting point is the beginning of the [**node**](./low_level.md#nodes) where execution stopped.
- If you're making a subgraph call inside a node, the starting point will be the **parent** node that called the subgraph that was halted.
  Inside the subgraph, the starting point will be the specific [**node**](./low_level.md#nodes) where execution stopped.
- If you're using the Functional API, the starting point is the beginning of the [**entrypoint**](./functional_api.md#entrypoint) where execution stopped.

  :::

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/faq.md
```md
---
search:
  boost: 2
---

# FAQ

Common questions and their answers!

## Do I need to use LangChain to use LangGraph? What’s the difference?

No. LangGraph is an orchestration framework for complex agentic systems and is more low-level and controllable than LangChain agents. LangChain provides a standard interface to interact with models and other components, useful for straight-forward chains and retrieval flows.

## How is LangGraph different from other agent frameworks?

Other agentic frameworks can work for simple, generic tasks but fall short for complex tasks. LangGraph provides a more expressive framework to handle your unique tasks without restricting you to a single black-box cognitive architecture.

## Does LangGraph impact the performance of my app?

LangGraph will not add any overhead to your code and is specifically designed with streaming workflows in mind.

## Is LangGraph open source? Is it free?

Yes. LangGraph is an MIT-licensed open-source library and is free to use.

## How are LangGraph and LangGraph Platform different?

LangGraph is a stateful, orchestration framework that brings added control to agent workflows. LangGraph Platform is a service for deploying and scaling LangGraph applications, with an opinionated API for building agent UXs, plus an integrated developer studio.

| Features            | LangGraph (open source)                                   | LangGraph Platform                                                                                     |
| ------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Description         | Stateful orchestration framework for agentic applications | Scalable infrastructure for deploying LangGraph applications                                           |
| SDKs                | Python and JavaScript                                     | Python and JavaScript                                                                                  |
| HTTP APIs           | None                                                      | Yes - useful for retrieving & updating state or long-term memory, or creating a configurable assistant |
| Streaming           | Basic                                                     | Dedicated mode for token-by-token messages                                                             |
| Checkpointer        | Community contributed                                     | Supported out-of-the-box                                                                               |
| Persistence Layer   | Self-managed                                              | Managed Postgres with efficient storage                                                                |
| Deployment          | Self-managed                                              | • Cloud SaaS <br> • Free self-hosted <br> • Enterprise (paid self-hosted)                              |
| Scalability         | Self-managed                                              | Auto-scaling of task queues and servers                                                                |
| Fault-tolerance     | Self-managed                                              | Automated retries                                                                                      |
| Concurrency Control | Simple threading                                          | Supports double-texting                                                                                |
| Scheduling          | None                                                      | Cron scheduling                                                                                        |
| Monitoring          | None                                                      | Integrated with LangSmith for observability                                                            |
| IDE integration     | LangGraph Studio                                          | LangGraph Studio                                                                                       |

## Is LangGraph Platform open source?

No. LangGraph Platform is proprietary software.

There is a free, self-hosted version of LangGraph Platform with access to basic features. The Cloud SaaS deployment option and the Self-Hosted deployment options are paid services. [Contact our sales team](https://www.langchain.com/contact-sales) to learn more.

For more information, see our [LangGraph Platform pricing page](https://www.langchain.com/pricing-langgraph-platform).

## Does LangGraph work with LLMs that don't support tool calling?

Yes! You can use LangGraph with any LLMs. The main reason we use LLMs that support tool calling is that this is often the most convenient way to have the LLM make its decision about what to do. If your LLM does not support tool calling, you can still use it - you just need to write a bit of logic to convert the raw LLM string response to a decision about what to do.

## Does LangGraph work with OSS LLMs?

Yes! LangGraph is totally ambivalent to what LLMs are used under the hood. The main reason we use closed LLMs in most of the tutorials is that they seamlessly support tool calling, while OSS LLMs often don't. But tool calling is not necessary (see [this section](#does-langgraph-work-with-llms-that-dont-support-tool-calling)) so you can totally use LangGraph with OSS LLMs.

## Can I use LangGraph Studio without logging in to LangSmith

Yes! You can use the [development version of LangGraph Server](../tutorials/langgraph-platform/local-server.md) to run the backend locally.
This will connect to the studio frontend hosted as part of LangSmith.
If you set an environment variable of `LANGSMITH_TRACING=false`, then no traces will be sent to LangSmith.

## What does "nodes executed" mean for LangGraph Platform usage?

**Nodes Executed** is the aggregate number of nodes in a LangGraph application that are called and completed successfully during an invocation of the application. If a node in the graph is not called during execution or ends in an error state, these nodes will not be counted. If a node is called and completes successfully multiple times, each occurrence will be counted.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/functional_api.md
```md
---
search:
  boost: 2
---

# Functional API concepts

## Overview

The **Functional API** allows you to add LangGraph's key features — [persistence](./persistence.md), [memory](../how-tos/memory/add-memory.md), [human-in-the-loop](./human_in_the_loop.md), and [streaming](./streaming.md) — to your applications with minimal changes to your existing code.

It is designed to integrate these features into existing code that may use standard language primitives for branching and control flow, such as `if` statements, `for` loops, and function calls. Unlike many data orchestration frameworks that require restructuring code into an explicit pipeline or DAG, the Functional API allows you to incorporate these capabilities without enforcing a rigid execution model.

The Functional API uses two key building blocks:

:::python

- **`@entrypoint`** – Marks a function as the starting point of a workflow, encapsulating logic and managing execution flow, including handling long-running tasks and interrupts.
- **`@task`** – Represents a discrete unit of work, such as an API call or data processing step, that can be executed asynchronously within an entrypoint. Tasks return a future-like object that can be awaited or resolved synchronously.  
  :::

:::js

- **`entrypoint`** – An entrypoint encapsulates workflow logic and manages execution flow, including handling long-running tasks and interrupts.
- **`task`** – Represents a discrete unit of work, such as an API call or data processing step, that can be executed asynchronously within an entrypoint. Tasks return a future-like object that can be awaited or resolved synchronously.  
  :::

This provides a minimal abstraction for building workflows with state management and streaming.

!!! tip

    For information on how to use the functional API, see [Use Functional API](../how-tos/use-functional-api.md).

## Functional API vs. Graph API

For users who prefer a more declarative approach, LangGraph's [Graph API](./low_level.md) allows you to define workflows using a Graph paradigm. Both APIs share the same underlying runtime, so you can use them together in the same application.

Here are some key differences:

- **Control flow**: The Functional API does not require thinking about graph structure. You can use standard Python constructs to define workflows. This will usually trim the amount of code you need to write.
- **Short-term memory**: The **GraphAPI** requires declaring a [**State**](./low_level.md#state) and may require defining [**reducers**](./low_level.md#reducers) to manage updates to the graph state. `@entrypoint` and `@tasks` do not require explicit state management as their state is scoped to the function and is not shared across functions.
- **Checkpointing**: Both APIs generate and use checkpoints. In the **Graph API** a new checkpoint is generated after every [superstep](./low_level.md). In the **Functional API**, when tasks are executed, their results are saved to an existing checkpoint associated with the given entrypoint instead of creating a new checkpoint.
- **Visualization**: The Graph API makes it easy to visualize the workflow as a graph which can be useful for debugging, understanding the workflow, and sharing with others. The Functional API does not support visualization as the graph is dynamically generated during runtime.

## Example

Below we demonstrate a simple application that writes an essay and [interrupts](human_in_the_loop.md) to request human review.

:::python

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import interrupt

@task
def write_essay(topic: str) -> str:
    """Write an essay about the given topic."""
    time.sleep(1) # A placeholder for a long-running task.
    return f"An essay about topic: {topic}"

@entrypoint(checkpointer=InMemorySaver())
def workflow(topic: str) -> dict:
    """A simple workflow that writes an essay and asks for a review."""
    essay = write_essay("cat").result()
    is_approved = interrupt({
        # Any json-serializable payload provided to interrupt as argument.
        # It will be surfaced on the client side as an Interrupt when streaming data
        # from the workflow.
        "essay": essay, # The essay we want reviewed.
        # We can add any additional information that we need.
        # For example, introduce a key called "action" with some instructions.
        "action": "Please approve/reject the essay",
    })

    return {
        "essay": essay, # The essay that was generated
        "is_approved": is_approved, # Response from HIL
    }
```

:::

:::js

```typescript
import { MemorySaver, entrypoint, task, interrupt } from "@langchain/langgraph";

const writeEssay = task("writeEssay", async (topic: string) => {
  // A placeholder for a long-running task.
  await new Promise((resolve) => setTimeout(resolve, 1000));
  return `An essay about topic: ${topic}`;
});

const workflow = entrypoint(
  { checkpointer: new MemorySaver(), name: "workflow" },
  async (topic: string) => {
    const essay = await writeEssay(topic);
    const isApproved = interrupt({
      // Any json-serializable payload provided to interrupt as argument.
      // It will be surfaced on the client side as an Interrupt when streaming data
      // from the workflow.
      essay, // The essay we want reviewed.
      // We can add any additional information that we need.
      // For example, introduce a key called "action" with some instructions.
      action: "Please approve/reject the essay",
    });

    return {
      essay, // The essay that was generated
      isApproved, // Response from HIL
    };
  }
);
```

:::

??? example "Detailed Explanation"

    This workflow will write an essay about the topic "cat" and then pause to get a review from a human. The workflow can be interrupted for an indefinite amount of time until a review is provided.

    When the workflow is resumed, it executes from the very start, but because the result of the `writeEssay` task was already saved, the task result will be loaded from the checkpoint instead of being recomputed.

    :::python
    ```python
    import time
    import uuid
    from langgraph.func import entrypoint, task
    from langgraph.types import interrupt
    from langgraph.checkpoint.memory import InMemorySaver


    @task
    def write_essay(topic: str) -> str:
        """Write an essay about the given topic."""
        time.sleep(1)  # This is a placeholder for a long-running task.
        return f"An essay about topic: {topic}"

    @entrypoint(checkpointer=InMemorySaver())
    def workflow(topic: str) -> dict:
        """A simple workflow that writes an essay and asks for a review."""
        essay = write_essay("cat").result()
        is_approved = interrupt(
            {
                # Any json-serializable payload provided to interrupt as argument.
                # It will be surfaced on the client side as an Interrupt when streaming data
                # from the workflow.
                "essay": essay,  # The essay we want reviewed.
                # We can add any additional information that we need.
                # For example, introduce a key called "action" with some instructions.
                "action": "Please approve/reject the essay",
            }
        )
        return {
            "essay": essay,  # The essay that was generated
            "is_approved": is_approved,  # Response from HIL
        }


    thread_id = str(uuid.uuid4())
    config = {"configurable": {"thread_id": thread_id}}
    for item in workflow.stream("cat", config):
        print(item)
    # > {'write_essay': 'An essay about topic: cat'}
    # > {
    # >     '__interrupt__': (
    # >        Interrupt(
    # >            value={
    # >                'essay': 'An essay about topic: cat',
    # >                'action': 'Please approve/reject the essay'
    # >            },
    # >            id='b9b2b9d788f482663ced6dc755c9e981'
    # >        ),
    # >    )
    # > }
    ```

    An essay has been written and is ready for review. Once the review is provided, we can resume the workflow:

    ```python
    from langgraph.types import Command

    # Get review from a user (e.g., via a UI)
    # In this case, we're using a bool, but this can be any json-serializable value.
    human_review = True

    for item in workflow.stream(Command(resume=human_review), config):
        print(item)
    ```

    ```pycon
    {'workflow': {'essay': 'An essay about topic: cat', 'is_approved': False}}
    ```

    The workflow has been completed and the review has been added to the essay.
    :::

    :::js
    ```typescript
    import { v4 as uuidv4 } from "uuid";
    import { MemorySaver, entrypoint, task, interrupt } from "@langchain/langgraph";

    const writeEssay = task("writeEssay", async (topic: string) => {
      // This is a placeholder for a long-running task.
      await new Promise(resolve => setTimeout(resolve, 1000));
      return `An essay about topic: ${topic}`;
    });

    const workflow = entrypoint(
      { checkpointer: new MemorySaver(), name: "workflow" },
      async (topic: string) => {
        const essay = await writeEssay(topic);
        const isApproved = interrupt({
          // Any json-serializable payload provided to interrupt as argument.
          // It will be surfaced on the client side as an Interrupt when streaming data
          // from the workflow.
          essay, // The essay we want reviewed.
          // We can add any additional information that we need.
          // For example, introduce a key called "action" with some instructions.
          action: "Please approve/reject the essay",
        });

        return {
          essay, // The essay that was generated
          isApproved, // Response from HIL
        };
      }
    );

    const threadId = uuidv4();

    const config = {
      configurable: {
        thread_id: threadId
      }
    };

    for await (const item of workflow.stream("cat", config)) {
      console.log(item);
    }
    ```

    ```console
    { writeEssay: 'An essay about topic: cat' }
    {
      __interrupt__: [{
        value: { essay: 'An essay about topic: cat', action: 'Please approve/reject the essay' },
        resumable: true,
        ns: ['workflow:f7b8508b-21c0-8b4c-5958-4e8de74d2684'],
        when: 'during'
      }]
    }
    ```

    An essay has been written and is ready for review. Once the review is provided, we can resume the workflow:

    ```typescript
    import { Command } from "@langchain/langgraph";

    // Get review from a user (e.g., via a UI)
    // In this case, we're using a bool, but this can be any json-serializable value.
    const humanReview = true;

    for await (const item of workflow.stream(new Command({ resume: humanReview }), config)) {
      console.log(item);
    }
    ```

    ```console
    { workflow: { essay: 'An essay about topic: cat', isApproved: true } }
    ```

    The workflow has been completed and the review has been added to the essay.
    :::

## Entrypoint

:::python
The @[`@entrypoint`][entrypoint] decorator can be used to create a workflow from a function. It encapsulates workflow logic and manages execution flow, including handling _long-running tasks_ and [interrupts](./human_in_the_loop.md).
:::

:::js
The @[`entrypoint`][entrypoint] function can be used to create a workflow from a function. It encapsulates workflow logic and manages execution flow, including handling _long-running tasks_ and [interrupts](./human_in_the_loop.md).
:::

### Definition

:::python
An **entrypoint** is defined by decorating a function with the `@entrypoint` decorator.

The function **must accept a single positional argument**, which serves as the workflow input. If you need to pass multiple pieces of data, use a dictionary as the input type for the first argument.

Decorating a function with an `entrypoint` produces a @[`Pregel`][Pregel.stream] instance which helps to manage the execution of the workflow (e.g., handles streaming, resumption, and checkpointing).

You will usually want to pass a **checkpointer** to the `@entrypoint` decorator to enable persistence and use features like **human-in-the-loop**.

=== "Sync"

    ```python
    from langgraph.func import entrypoint

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(some_input: dict) -> int:
        # some logic that may involve long-running tasks like API calls,
        # and may be interrupted for human-in-the-loop.
        ...
        return result
    ```

=== "Async"

    ```python
    from langgraph.func import entrypoint

    @entrypoint(checkpointer=checkpointer)
    async def my_workflow(some_input: dict) -> int:
        # some logic that may involve long-running tasks like API calls,
        # and may be interrupted for human-in-the-loop
        ...
        return result
    ```

:::

:::js
An **entrypoint** is defined by calling the `entrypoint` function with configuration and a function.

The function **must accept a single positional argument**, which serves as the workflow input. If you need to pass multiple pieces of data, use an object as the input type for the first argument.

Creating an entrypoint with a function produces a workflow instance which helps to manage the execution of the workflow (e.g., handles streaming, resumption, and checkpointing).

You will often want to pass a **checkpointer** to the `entrypoint` function to enable persistence and use features like **human-in-the-loop**.

```typescript
import { entrypoint } from "@langchain/langgraph";

const myWorkflow = entrypoint(
  { checkpointer, name: "workflow" },
  async (someInput: Record<string, any>): Promise<number> => {
    // some logic that may involve long-running tasks like API calls,
    // and may be interrupted for human-in-the-loop
    return result;
  }
);
```

:::

!!! important "Serialization"

    The **inputs** and **outputs** of entrypoints must be JSON-serializable to support checkpointing. Please see the [serialization](#serialization) section for more details.

:::python

### Injectable parameters

When declaring an `entrypoint`, you can request access to additional parameters that will be injected automatically at run time. These parameters include:

| Parameter    | Description                                                                                                                                                        |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **previous** | Access the state associated with the previous `checkpoint` for the given thread. See [short-term-memory](#short-term-memory).                                      |
| **store**    | An instance of [BaseStore][langgraph.store.base.BaseStore]. Useful for [long-term memory](../how-tos/use-functional-api.md#long-term-memory).                      |
| **writer**   | Use to access the StreamWriter when working with Async Python < 3.11. See [streaming with functional API for details](../how-tos/use-functional-api.md#streaming). |
| **config**   | For accessing run time configuration. See [RunnableConfig](https://python.langchain.com/docs/concepts/runnables/#runnableconfig) for information.                  |

!!! important

    Declare the parameters with the appropriate name and type annotation.

??? example "Requesting Injectable Parameters"

    ```python
    from langchain_core.runnables import RunnableConfig
    from langgraph.func import entrypoint
    from langgraph.store.base import BaseStore
    from langgraph.store.memory import InMemoryStore

    in_memory_store = InMemoryStore(...)  # An instance of InMemoryStore for long-term memory

    @entrypoint(
        checkpointer=checkpointer,  # Specify the checkpointer
        store=in_memory_store  # Specify the store
    )
    def my_workflow(
        some_input: dict,  # The input (e.g., passed via `invoke`)
        *,
        previous: Any = None, # For short-term memory
        store: BaseStore,  # For long-term memory
        writer: StreamWriter,  # For streaming custom data
        config: RunnableConfig  # For accessing the configuration passed to the entrypoint
    ) -> ...:
    ```

:::

### Executing

:::python
Using the [`@entrypoint`](#entrypoint) yields a @[`Pregel`][Pregel.stream] object that can be executed using the `invoke`, `ainvoke`, `stream`, and `astream` methods.

=== "Invoke"

    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    my_workflow.invoke(some_input, config)  # Wait for the result synchronously
    ```

=== "Async Invoke"

    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    await my_workflow.ainvoke(some_input, config)  # Await result asynchronously
    ```

=== "Stream"

    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    for chunk in my_workflow.stream(some_input, config):
        print(chunk)
    ```

=== "Async Stream"

    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    async for chunk in my_workflow.astream(some_input, config):
        print(chunk)
    ```

:::

:::js
Using the [`entrypoint`](#entrypoint) function will return an object that can be executed using the `invoke` and `stream` methods.

=== "Invoke"

    ```typescript
    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };
    await myWorkflow.invoke(someInput, config); // Wait for the result
    ```

=== "Stream"

    ```typescript
    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };

    for await (const chunk of myWorkflow.stream(someInput, config)) {
      console.log(chunk);
    }
    ```

:::

### Resuming

:::python
Resuming an execution after an @[interrupt][interrupt] can be done by passing a **resume** value to the @[Command] primitive.

=== "Invoke"

    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    my_workflow.invoke(Command(resume=some_resume_value), config)
    ```

=== "Async Invoke"

    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    await my_workflow.ainvoke(Command(resume=some_resume_value), config)
    ```

=== "Stream"

    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    for chunk in my_workflow.stream(Command(resume=some_resume_value), config):
        print(chunk)
    ```

=== "Async Stream"

    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    async for chunk in my_workflow.astream(Command(resume=some_resume_value), config):
        print(chunk)
    ```

:::

:::js
Resuming an execution after an @[interrupt][interrupt] can be done by passing a **resume** value to the @[`Command`][Command] primitive.

=== "Invoke"

    ```typescript
    import { Command } from "@langchain/langgraph";

    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };

    await myWorkflow.invoke(new Command({ resume: someResumeValue }), config);
    ```

=== "Stream"

    ```typescript
    import { Command } from "@langchain/langgraph";

    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };

    const stream = await myWorkflow.stream(
      new Command({ resume: someResumableValue }),
      config,
    )

    for await (const chunk of stream) {
      console.log(chunk);
    }
    ```

:::

:::python

**Resuming after an error**

To resume after an error, run the `entrypoint` with a `None` and the same **thread id** (config).

This assumes that the underlying **error** has been resolved and execution can proceed successfully.

=== "Invoke"

    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    my_workflow.invoke(None, config)
    ```

=== "Async Invoke"

    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    await my_workflow.ainvoke(None, config)
    ```

=== "Stream"

    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    for chunk in my_workflow.stream(None, config):
        print(chunk)
    ```

=== "Async Stream"

    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    async for chunk in my_workflow.astream(None, config):
        print(chunk)
    ```

:::

:::js

**Resuming after an error**

To resume after an error, run the `entrypoint` with `null` and the same **thread id** (config).

This assumes that the underlying **error** has been resolved and execution can proceed successfully.

=== "Invoke"

    ```typescript
    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };

    await myWorkflow.invoke(null, config);
    ```

=== "Stream"

    ```typescript
    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };

    for await (const chunk of myWorkflow.stream(null, config)) {
      console.log(chunk);
    }
    ```

:::

### Short-term memory

When an `entrypoint` is defined with a `checkpointer`, it stores information between successive invocations on the same **thread id** in [checkpoints](persistence.md#checkpoints).

:::python
This allows accessing the state from the previous invocation using the `previous` parameter.

By default, the `previous` parameter is the return value of the previous invocation.

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(number: int, *, previous: Any = None) -> int:
    previous = previous or 0
    return number + previous

config = {
    "configurable": {
        "thread_id": "some_thread_id"
    }
}

my_workflow.invoke(1, config)  # 1 (previous was None)
my_workflow.invoke(2, config)  # 3 (previous was 1 from the previous invocation)
```

:::

:::js
This allows accessing the state from the previous invocation using the `getPreviousState` function.

By default, the `getPreviousState` function returns the return value of the previous invocation.

```typescript
import { entrypoint, getPreviousState } from "@langchain/langgraph";

const myWorkflow = entrypoint(
  { checkpointer, name: "workflow" },
  async (number: number) => {
    const previous = getPreviousState<number>() ?? 0;
    return number + previous;
  }
);

const config = {
  configurable: {
    thread_id: "some_thread_id",
  },
};

await myWorkflow.invoke(1, config); // 1 (previous was undefined)
await myWorkflow.invoke(2, config); // 3 (previous was 1 from the previous invocation)
```

:::

#### `entrypoint.final`

:::python
@[`entrypoint.final`][entrypoint.final] is a special primitive that can be returned from an entrypoint and allows **decoupling** the value that is **saved in the checkpoint** from the **return value of the entrypoint**.

The first value is the return value of the entrypoint, and the second value is the value that will be saved in the checkpoint. The type annotation is `entrypoint.final[return_type, save_type]`.

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(number: int, *, previous: Any = None) -> entrypoint.final[int, int]:
    previous = previous or 0
    # This will return the previous value to the caller, saving
    # 2 * number to the checkpoint, which will be used in the next invocation
    # for the `previous` parameter.
    return entrypoint.final(value=previous, save=2 * number)

config = {
    "configurable": {
        "thread_id": "1"
    }
}

my_workflow.invoke(3, config)  # 0 (previous was None)
my_workflow.invoke(1, config)  # 6 (previous was 3 * 2 from the previous invocation)
```

:::

:::js
@[`entrypoint.final`][entrypoint.final] is a special primitive that can be returned from an entrypoint and allows **decoupling** the value that is **saved in the checkpoint** from the **return value of the entrypoint**.

The first value is the return value of the entrypoint, and the second value is the value that will be saved in the checkpoint.

```typescript
import { entrypoint, getPreviousState } from "@langchain/langgraph";

const myWorkflow = entrypoint(
  { checkpointer, name: "workflow" },
  async (number: number) => {
    const previous = getPreviousState<number>() ?? 0;
    // This will return the previous value to the caller, saving
    // 2 * number to the checkpoint, which will be used in the next invocation
    // for the `previous` parameter.
    return entrypoint.final({
      value: previous,
      save: 2 * number,
    });
  }
);

const config = {
  configurable: {
    thread_id: "1",
  },
};

await myWorkflow.invoke(3, config); // 0 (previous was undefined)
await myWorkflow.invoke(1, config); // 6 (previous was 3 * 2 from the previous invocation)
```

:::

## Task

A **task** represents a discrete unit of work, such as an API call or data processing step. It has two key characteristics:

- **Asynchronous Execution**: Tasks are designed to be executed asynchronously, allowing multiple operations to run concurrently without blocking.
- **Checkpointing**: Task results are saved to a checkpoint, enabling resumption of the workflow from the last saved state. (See [persistence](persistence.md) for more details).

### Definition

:::python
Tasks are defined using the `@task` decorator, which wraps a regular Python function.

```python
from langgraph.func import task

@task()
def slow_computation(input_value):
    # Simulate a long-running operation
    ...
    return result
```

:::

:::js
Tasks are defined using the `task` function, which wraps a regular function.

```typescript
import { task } from "@langchain/langgraph";

const slowComputation = task("slowComputation", async (inputValue: any) => {
  // Simulate a long-running operation
  return result;
});
```

:::

!!! important "Serialization"

    The **outputs** of tasks must be JSON-serializable to support checkpointing.

### Execution

**Tasks** can only be called from within an **entrypoint**, another **task**, or a [state graph node](./low_level.md#nodes).

Tasks _cannot_ be called directly from the main application code.

:::python
When you call a **task**, it returns _immediately_ with a future object. A future is a placeholder for a result that will be available later.

To obtain the result of a **task**, you can either wait for it synchronously (using `result()`) or await it asynchronously (using `await`).

=== "Synchronous Invocation"

    ```python
    @entrypoint(checkpointer=checkpointer)
    def my_workflow(some_input: int) -> int:
        future = slow_computation(some_input)
        return future.result()  # Wait for the result synchronously
    ```

=== "Asynchronous Invocation"

    ```python
    @entrypoint(checkpointer=checkpointer)
    async def my_workflow(some_input: int) -> int:
        return await slow_computation(some_input)  # Await result asynchronously
    ```

:::

:::js
When you call a **task**, it returns a Promise that can be awaited.

```typescript
const myWorkflow = entrypoint(
  { checkpointer, name: "workflow" },
  async (someInput: number): Promise<number> => {
    return await slowComputation(someInput);
  }
);
```

:::

## When to use a task

**Tasks** are useful in the following scenarios:

- **Checkpointing**: When you need to save the result of a long-running operation to a checkpoint, so you don't need to recompute it when resuming the workflow.
- **Human-in-the-loop**: If you're building a workflow that requires human intervention, you MUST use **tasks** to encapsulate any randomness (e.g., API calls) to ensure that the workflow can be resumed correctly. See the [determinism](#determinism) section for more details.
- **Parallel Execution**: For I/O-bound tasks, **tasks** enable parallel execution, allowing multiple operations to run concurrently without blocking (e.g., calling multiple APIs).
- **Observability**: Wrapping operations in **tasks** provides a way to track the progress of the workflow and monitor the execution of individual operations using [LangSmith](https://docs.smith.langchain.com/).
- **Retryable Work**: When work needs to be retried to handle failures or inconsistencies, **tasks** provide a way to encapsulate and manage the retry logic.

## Serialization

There are two key aspects to serialization in LangGraph:

1. `entrypoint` inputs and outputs must be JSON-serializable.
2. `task` outputs must be JSON-serializable.

:::python
These requirements are necessary for enabling checkpointing and workflow resumption. Use python primitives like dictionaries, lists, strings, numbers, and booleans to ensure that your inputs and outputs are serializable.
:::

:::js
These requirements are necessary for enabling checkpointing and workflow resumption. Use primitives like objects, arrays, strings, numbers, and booleans to ensure that your inputs and outputs are serializable.
:::

Serialization ensures that workflow state, such as task results and intermediate values, can be reliably saved and restored. This is critical for enabling human-in-the-loop interactions, fault tolerance, and parallel execution.

Providing non-serializable inputs or outputs will result in a runtime error when a workflow is configured with a checkpointer.

## Determinism

To utilize features like **human-in-the-loop**, any randomness should be encapsulated inside of **tasks**. This guarantees that when execution is halted (e.g., for human in the loop) and then resumed, it will follow the same _sequence of steps_, even if **task** results are non-deterministic.

LangGraph achieves this behavior by persisting **task** and [**subgraph**](./subgraphs.md) results as they execute. A well-designed workflow ensures that resuming execution follows the _same sequence of steps_, allowing previously computed results to be retrieved correctly without having to re-execute them. This is particularly useful for long-running **tasks** or **tasks** with non-deterministic results, as it avoids repeating previously done work and allows resuming from essentially the same.

While different runs of a workflow can produce different results, resuming a **specific** run should always follow the same sequence of recorded steps. This allows LangGraph to efficiently look up **task** and **subgraph** results that were executed prior to the graph being interrupted and avoid recomputing them.

## Idempotency

Idempotency ensures that running the same operation multiple times produces the same result. This helps prevent duplicate API calls and redundant processing if a step is rerun due to a failure. Always place API calls inside **tasks** functions for checkpointing, and design them to be idempotent in case of re-execution. Re-execution can occur if a **task** starts, but does not complete successfully. Then, if the workflow is resumed, the **task** will run again. Use idempotency keys or verify existing results to avoid duplication.

## Common Pitfalls

### Handling side effects

Encapsulate side effects (e.g., writing to a file, sending an email) in tasks to ensure they are not executed multiple times when resuming a workflow.

=== "Incorrect"

    In this example, a side effect (writing to a file) is directly included in the workflow, so it will be executed a second time when resuming the workflow.

    :::python
    ```python
    @entrypoint(checkpointer=checkpointer)
    def my_workflow(inputs: dict) -> int:
        # This code will be executed a second time when resuming the workflow.
        # Which is likely not what you want.
        # highlight-next-line
        with open("output.txt", "w") as f:
            # highlight-next-line
            f.write("Side effect executed")
        value = interrupt("question")
        return value
    ```
    :::

    :::js
    ```typescript
    import { entrypoint, interrupt } from "@langchain/langgraph";
    import fs from "fs";

    const myWorkflow = entrypoint(
      { checkpointer, name: "workflow },
      async (inputs: Record<string, any>) => {
        // This code will be executed a second time when resuming the workflow.
        // Which is likely not what you want.
        fs.writeFileSync("output.txt", "Side effect executed");
        const value = interrupt("question");
        return value;
      }
    );
    ```
    :::

=== "Correct"

    In this example, the side effect is encapsulated in a task, ensuring consistent execution upon resumption.

    :::python
    ```python
    from langgraph.func import task

    # highlight-next-line
    @task
    # highlight-next-line
    def write_to_file():
        with open("output.txt", "w") as f:
            f.write("Side effect executed")

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(inputs: dict) -> int:
        # The side effect is now encapsulated in a task.
        write_to_file().result()
        value = interrupt("question")
        return value
    ```
    :::

    :::js
    ```typescript
    import { entrypoint, task, interrupt } from "@langchain/langgraph";
    import * as fs from "fs";

    const writeToFile = task("writeToFile", async () => {
      fs.writeFileSync("output.txt", "Side effect executed");
    });

    const myWorkflow = entrypoint(
      { checkpointer, name: "workflow" },
      async (inputs: Record<string, any>) => {
        // The side effect is now encapsulated in a task.
        await writeToFile();
        const value = interrupt("question");
        return value;
      }
    );
    ```
    :::

### Non-deterministic control flow

Operations that might give different results each time (like getting current time or random numbers) should be encapsulated in tasks to ensure that on resume, the same result is returned.

- In a task: Get random number (5) → interrupt → resume → (returns 5 again) → ...
- Not in a task: Get random number (5) → interrupt → resume → get new random number (7) → ...

:::python
This is especially important when using **human-in-the-loop** workflows with multiple interrupts calls. LangGraph keeps a list of resume values for each task/entrypoint. When an interrupt is encountered, it's matched with the corresponding resume value. This matching is strictly **index-based**, so the order of the resume values should match the order of the interrupts.
:::

:::js
This is especially important when using **human-in-the-loop** workflows with multiple interrupt calls. LangGraph keeps a list of resume values for each task/entrypoint. When an interrupt is encountered, it's matched with the corresponding resume value. This matching is strictly **index-based**, so the order of the resume values should match the order of the interrupts.
:::

If order of execution is not maintained when resuming, one `interrupt` call may be matched with the wrong `resume` value, leading to incorrect results.

Please read the section on [determinism](#determinism) for more details.

=== "Incorrect"

    In this example, the workflow uses the current time to determine which task to execute. This is non-deterministic because the result of the workflow depends on the time at which it is executed.

    :::python
    ```python
    from langgraph.func import entrypoint

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(inputs: dict) -> int:
        t0 = inputs["t0"]
        # highlight-next-line
        t1 = time.time()

        delta_t = t1 - t0

        if delta_t > 1:
            result = slow_task(1).result()
            value = interrupt("question")
        else:
            result = slow_task(2).result()
            value = interrupt("question")

        return {
            "result": result,
            "value": value
        }
    ```
    :::

    :::js
    ```typescript
    import { entrypoint, interrupt } from "@langchain/langgraph";

    const myWorkflow = entrypoint(
      { checkpointer, name: "workflow" },
      async (inputs: { t0: number }) => {
        const t1 = Date.now();

        const deltaT = t1 - inputs.t0;

        if (deltaT > 1000) {
          const result = await slowTask(1);
          const value = interrupt("question");
          return { result, value };
        } else {
          const result = await slowTask(2);
          const value = interrupt("question");
          return { result, value };
        }
      }
    );
    ```
    :::

=== "Correct"

    :::python
    In this example, the workflow uses the input `t0` to determine which task to execute. This is deterministic because the result of the workflow depends only on the input.

    ```python
    import time

    from langgraph.func import task

    # highlight-next-line
    @task
    # highlight-next-line
    def get_time() -> float:
        return time.time()

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(inputs: dict) -> int:
        t0 = inputs["t0"]
        # highlight-next-line
        t1 = get_time().result()

        delta_t = t1 - t0

        if delta_t > 1:
            result = slow_task(1).result()
            value = interrupt("question")
        else:
            result = slow_task(2).result()
            value = interrupt("question")

        return {
            "result": result,
            "value": value
        }
    ```
    :::

    :::js
    In this example, the workflow uses the input `t0` to determine which task to execute. This is deterministic because the result of the workflow depends only on the input.

    ```typescript
    import { entrypoint, task, interrupt } from "@langchain/langgraph";

    const getTime = task("getTime", () => Date.now());

    const myWorkflow = entrypoint(
      { checkpointer, name: "workflow" },
      async (inputs: { t0: number }): Promise<any> => {
        const t1 = await getTime();

        const deltaT = t1 - inputs.t0;

        if (deltaT > 1000) {
          const result = await slowTask(1);
          const value = interrupt("question");
          return { result, value };
        } else {
          const result = await slowTask(2);
          const value = interrupt("question");
          return { result, value };
        }
      }
    );
    ```
    :::

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/human_in_the_loop.md
```md
---
search:
  boost: 2
tags:
  - human-in-the-loop
  - hil
  - overview
hide:
  - tags
---

# Human-in-the-loop

To review, edit, and approve tool calls in an agent or workflow, [use LangGraph's human-in-the-loop features](../how-tos/human_in_the_loop/add-human-in-the-loop.md) to enable human intervention at any point in a workflow. This is especially useful in large language model (LLM)-driven applications where model output may require validation, correction, or additional context.

<figure markdown="1">
![image](../concepts/img/human_in_the_loop/tool-call-review.png){: style="max-height:400px"}
</figure>

!!! tip

    For information on how to use human-in-the-loop, see [Enable human intervention](../how-tos/human_in_the_loop/add-human-in-the-loop.md) and [Human-in-the-loop using Server API](../cloud/how-tos/add-human-in-the-loop.md).

## Key capabilities

* **Persistent execution state**: Interrupts use LangGraph's [persistence](./persistence.md) layer, which saves the graph state, to indefinitely pause graph execution until you resume. This is possible because LangGraph checkpoints the graph state after each step, which allows the system to persist execution context and later resume the workflow, continuing from where it left off. This supports asynchronous human review or input without time constraints.

    There are two ways to pause a graph:

    - [Dynamic interrupts](../how-tos/human_in_the_loop/add-human-in-the-loop.md#pause-using-interrupt): Use `interrupt` to pause a graph from inside a specific node, based on the current state of the graph.
    - [Static interrupts](../how-tos/human_in_the_loop/add-human-in-the-loop.md#debug-with-interrupts): Use `interrupt_before` and `interrupt_after` to pause the graph at pre-defined points, either before or after a node executes.

    <figure markdown="1">
    ![image](./img/breakpoints.png){: style="max-height:400px"}
    <figcaption>An example graph consisting of 3 sequential steps with a breakpoint before step_3. </figcaption> </figure>

* **Flexible integration points**: Human-in-the-loop logic can be introduced at any point in the workflow. This allows targeted human involvement, such as approving API calls, correcting outputs, or guiding conversations.

## Patterns

There are four typical design patterns that you can implement using `interrupt` and `Command`:

- [Approve or reject](../how-tos/human_in_the_loop/add-human-in-the-loop.md#approve-or-reject): Pause the graph before a critical step, such as an API call, to review and approve the action. If the action is rejected, you can prevent the graph from executing the step, and potentially take an alternative action. This pattern often involves routing the graph based on the human's input.
- [Edit graph state](../how-tos/human_in_the_loop/add-human-in-the-loop.md#review-and-edit-state): Pause the graph to review and edit the graph state. This is useful for correcting mistakes or updating the state with additional information. This pattern often involves updating the state with the human's input.
- [Review tool calls](../how-tos/human_in_the_loop/add-human-in-the-loop.md#review-tool-calls): Pause the graph to review and edit tool calls requested by the LLM before tool execution.
- [Validate human input](../how-tos/human_in_the_loop/add-human-in-the-loop.md#validate-human-input): Pause the graph to validate human input before proceeding with the next step.
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/langgraph_cli.md
```md
---
search:
  boost: 2
---

# LangGraph CLI

**LangGraph CLI** is a multi-platform command-line tool for building and running the [LangGraph API server](./langgraph_server.md) locally. The resulting server includes all API endpoints for your graph's runs, threads, assistants, etc. as well as the other services required to run your agent, including a managed database for checkpointing and storage.

:::python

## Installation

The LangGraph CLI can be installed via pip or [Homebrew](https://brew.sh/):

=== "pip"

    ```bash
    pip install langgraph-cli
    ```

=== "Homebrew"

    ```bash
    brew install langgraph-cli
    ```
:::

:::js

## Installation

The LangGraph.js CLI can be installed from the NPM registry:

=== "npx"
    ```bash
    npx @langchain/langgraph-cli
    ```

=== "npm"
    ```bash
    npm install @langchain/langgraph-cli
    ```

=== "yarn"
    ```bash
    yarn add @langchain/langgraph-cli
    ```

=== "pnpm"
    ```bash
    pnpm add @langchain/langgraph-cli
    ```

=== "bun"
    ```bash
    bun add @langchain/langgraph-cli
    ```
:::

## Commands

LangGraph CLI provides the following core functionality:

| Command                                                        | Description                                                                                                                                                                                                                                                                            |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`langgraph build`](../cloud/reference/cli.md#build)           | Builds a Docker image for the [LangGraph API server](./langgraph_server.md) that can be directly deployed.                                                                                                                                                                             |
| [`langgraph dev`](../cloud/reference/cli.md#dev)               | Starts a lightweight development server that requires no Docker installation. This server is ideal for rapid development and testing.                                                                                                                                                  |
| [`langgraph dockerfile`](../cloud/reference/cli.md#dockerfile) | Generates a [Dockerfile](https://docs.docker.com/reference/dockerfile/) that can be used to build images for and deploy instances of the [LangGraph API server](./langgraph_server.md). This is useful if you want to further customize the dockerfile or deploy in a more custom way. |
| [`langgraph up`](../cloud/reference/cli.md#up)                 | Starts an instance of the [LangGraph API server](./langgraph_server.md) locally in a docker container. This requires the docker server to be running locally. It also requires a LangSmith API key for local development or a license key for production use.                          |

For more information, see the [LangGraph CLI Reference](../cloud/reference/cli.md).

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/langgraph_cloud.md
```md
---
search:
  boost: 2
---

# Cloud SaaS

To deploy a [LangGraph Server](../concepts/langgraph_server.md), follow the how-to guide for [how to deploy to Cloud SaaS](../cloud/deployment/cloud.md).

## Overview

The Cloud SaaS deployment option is a fully managed model for deployment where we manage the [control plane](./langgraph_control_plane.md) and [data plane](./langgraph_data_plane.md) in our cloud.

|                                    | [Control plane](../concepts/langgraph_control_plane.md)                                                                                     | [Data plane](../concepts/langgraph_data_plane.md)                                                                                                   |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **What is it?**                    | <ul><li>Control plane UI for creating deployments and revisions</li><li>Control plane APIs for creating deployments and revisions</li></ul> | <ul><li>Data plane "listener" for reconciling deployments with control plane state</li><li>LangGraph Servers</li><li>Postgres, Redis, etc</li></ul> |
| **Where is it hosted?**            | LangChain's cloud                                                                                                                           | LangChain's cloud                                                                                                                                   |
| **Who provisions and manages it?** | LangChain                                                                                                                                   | LangChain                                                                                                                                           |

## Architecture

![Cloud SaaS](./img/self_hosted_control_plane_architecture.png)

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/langgraph_components.md
```md
## Components

The LangGraph Platform consists of components that work together to support the development, deployment, debugging, and monitoring of LangGraph applications:

- [LangGraph Server](./langgraph_server.md): The server defines an opinionated API and architecture that incorporates best practices for deploying agentic applications, allowing you to focus on building your agent logic rather than developing server infrastructure.
- [LangGraph CLI](./langgraph_cli.md): LangGraph CLI is a command-line interface that helps to interact with a local LangGraph
- [LangGraph Studio](./langgraph_studio.md): LangGraph Studio is a specialized IDE that can connect to a LangGraph Server to enable visualization, interaction, and debugging of the application locally.
- [Python/JS SDK](./sdk.md): The Python/JS SDK provides a programmatic way to interact with deployed LangGraph Applications.
- [Remote Graph](../how-tos/use-remote-graph.md): A RemoteGraph allows you to interact with any deployed LangGraph application as though it were running locally.
- [LangGraph control plane](./langgraph_control_plane.md): The LangGraph Control Plane refers to the Control Plane UI where users create and update LangGraph Servers and the Control Plane APIs that support the UI experience.
- [LangGraph data plane](./langgraph_data_plane.md): The LangGraph Data Plane refers to LangGraph Servers, the corresponding infrastructure for each server, and the "listener" application that continuously polls for updates from the LangGraph Control Plane.

![LangGraph components](img/lg_platform.png)

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/langgraph_control_plane.md
```md
---
search:
  boost: 2
---

# LangGraph Control Plane

The term "control plane" is used broadly to refer to the control plane UI where users create and update [LangGraph Servers](./langgraph_server.md) (deployments) and the control plane APIs that support the UI experience.

When a user makes an update through the control plane UI, the update is stored in the control plane state. The [LangGraph Data Plane](./langgraph_data_plane.md) "listener" application polls for these updates by calling the control plane APIs.

## Control Plane UI

From the control plane UI, you can:

- View a list of outstanding deployments.
- View details of an individual deployment.
- Create a new deployment.
- Update a deployment.
- Update environment variables for a deployment.
- View build and server logs of a deployment.
- View deployment metrics such as CPU and memory usage.
- Delete a deployment.

The Control Plane UI is embedded in [LangSmith](https://docs.smith.langchain.com/langgraph_cloud).

## Control Plane API

This section describes the data model of the control plane API. The API is used to create, update, and delete deployments. See the [control plane API reference](../cloud/reference/api/api_ref_control_plane.md) for more details.

### Deployment

A deployment is an instance of a LangGraph Server. A single deployment can have many revisions.

### Revision

A revision is an iteration of a deployment. When a new deployment is created, an initial revision is automatically created. To deploy code changes or update secrets for a deployment, a new revision must be created.

## Control Plane Features

This section describes various features of the control plane.

### Deployment Types

For simplicity, the control plane offers two deployment types with different resource allocations: `Development` and `Production`.

| **Deployment Type** | **CPU/Memory**  | **Scaling**       | **Database**                                                                     |
| ------------------- | --------------- | ----------------- | -------------------------------------------------------------------------------- |
| Development         | 1 CPU, 1 GB RAM | Up to 1 replica   | 10 GB disk, no backups                                                           |
| Production          | 2 CPU, 2 GB RAM | Up to 10 replicas | Autoscaling disk, automatic backups, highly available (multi-zone configuration) |

CPU and memory resources are per replica.

!!! warning "Immutable Deployment Type"

    Once a deployment is created, the deployment type cannot be changed.

!!! info "Self-Hosted Deployment"
Resources for [Self-Hosted Data Plane](../concepts/langgraph_self_hosted_data_plane.md) and [Self-Hosted Control Plane](../concepts/langgraph_self_hosted_control_plane.md) deployments can be fully customized. Deployment types are only applicable for [Cloud SaaS](../concepts/langgraph_cloud.md) deployments.

#### Production

`Production` type deployments are suitable for "production" workloads. For example, select `Production` for customer-facing applications in the critical path.

Resources for `Production` type deployments can be manually increased on a case-by-case basis depending on use case and capacity constraints. Contact support@langchain.dev to request an increase in resources.

#### Development

`Development` type deployments are suitable development and testing. For example, select `Development` for internal testing environments. `Development` type deployments are not suitable for "production" workloads.

!!! danger "Preemptible Compute Infrastructure"
`Development` type deployments (API server, queue server, and database) are provisioned on preemptible compute infrastructure. This means the compute infrastructure **may be terminated at any time without notice**. This may result in intermittent...

    - Redis connection timeouts/errors
    - Postgres connection timeouts/errors
    - Failed or retrying background runs

    This behavior is expected. Preemptible compute infrastructure **significantly reduces the cost to provision a `Development` type deployment**. By design, LangGraph Server is fault-tolerant. The implementation will automatically attempt to recover from Redis/Postgres connection errors and retry failed background runs.

    `Production` type deployments are provisioned on durable compute infrastructure, not preemptible compute infrastructure.

Database disk size for `Development` type deployments can be manually increased on a case-by-case basis depending on use case and capacity constraints. For most use cases, [TTLs](../how-tos/ttl/configure_ttl.md) should be configured to manage disk usage. Contact support@langchain.dev to request an increase in resources.

### Database Provisioning

The control plane and [LangGraph Data Plane](./langgraph_data_plane.md) "listener" application coordinate to automatically create a Postgres database for each deployment. The database serves as the [persistence layer](../concepts/persistence.md) for the deployment.

When implementing a LangGraph application, a [checkpointer](../concepts/persistence.md#checkpointer-libraries) does not need to be configured by the developer. Instead, a checkpointer is automatically configured for the graph. Any checkpointer configured for a graph will be replaced by the one that is automatically configured.

There is no direct access to the database. All access to the database occurs through the [LangGraph Server](../concepts/langgraph_server.md).

The database is never deleted until the deployment itself is deleted.

!!! info
A custom Postgres instance can be configured for [Self-Hosted Data Plane](../concepts/langgraph_self_hosted_data_plane.md) and [Self-Hosted Control Plane](../concepts/langgraph_self_hosted_control_plane.md) deployments.

### Asynchronous Deployment

Infrastructure for deployments and revisions are provisioned and deployed asynchronously. They are not deployed immediately after submission. Currently, deployment can take up to several minutes.

- When a new deployment is created, a new database is created for the deployment. Database creation is a one-time step. This step contributes to a longer deployment time for the initial revision of the deployment.
- When a subsequent revision is created for a deployment, there is no database creation step. The deployment time for a subsequent revision is significantly faster compared to the deployment time of the initial revision.
- The deployment process for each revision contains a build step, which can take up to a few minutes.

The control plane and [LangGraph Data Plane](./langgraph_data_plane.md) "listener" application coordinate to achieve asynchronous deployments.

### Monitoring

After a deployment is ready, the control plane monitors the deployment and records various metrics, such as:

- CPU and memory usage of the deployment.
- Number of container restarts.
- Number of replicas (this will increase with [autoscaling](../concepts/langgraph_data_plane.md#autoscaling)).
- [Postgres](../concepts/langgraph_data_plane.md#postgres) CPU, memory usage, and disk usage.
- [LangGraph Server queue](../concepts/langgraph_server.md#persistence-and-task-queue) pending/active run count.
- [LangGraph Server API](../concepts/langgraph_server.md) success response count, error response count, and latency.

These metrics are displayed as charts in the Control Plane UI.

### LangSmith Integration

A [LangSmith](https://docs.smith.langchain.com/) tracing project and LangSmith API key are automatically created for each deployment. The deployment uses the API key to automatically send traces to LangSmith.

- The tracing project has the same name as the deployment.
- The API key has the description `LangGraph Platform: <deployment_name>`.
- The API key is never revealed and cannot be deleted manually.
- When creating a deployment, the `LANGCHAIN_TRACING` and `LANGSMITH_API_KEY`/`LANGCHAIN_API_KEY` environment variables do not need to be specified; they are set automatically by the control plane.

When a deployment is deleted, the traces and the tracing project are not deleted. However, the API will be deleted when the deployment is deleted.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/langgraph_data_plane.md
```md
---
search:
  boost: 2
---

# LangGraph Data Plane

The term "data plane" is used broadly to refer to [LangGraph Servers](./langgraph_server.md) (deployments), the corresponding infrastructure for each server, and the "listener" application that continuously polls for updates from the [LangGraph Control Plane](./langgraph_control_plane.md).

## Server Infrastructure

In addition to the [LangGraph Server](./langgraph_server.md) itself, the following infrastructure components for each server are also included in the broad definition of "data plane":

- Postgres
- Redis
- Secrets store
- Autoscalers

## "Listener" Application

The data plane "listener" application periodically calls [control plane APIs](../concepts/langgraph_control_plane.md#control-plane-api) to:

- Determine if new deployments should be created.
- Determine if existing deployments should be updated (i.e. new revisions).
- Determine if existing deployments should be deleted.

In other words, the data plane "listener" reads the latest state of the control plane (desired state) and takes action to reconcile outstanding deployments (current state) to match the latest state.

## Postgres

Postgres is the persistence layer for all user, run, and long-term memory data in a LangGraph Server. This stores both checkpoints (see more info [here](./persistence.md)), server resources (threads, runs, assistants and crons), as well as items saved in the long-term memory store (see more info [here](./persistence.md#memory-store)).

## Redis

Redis is used in each LangGraph Server as a way for server and queue workers to communicate, and to store ephemeral metadata. No user or run data is stored in Redis.

### Communication

All runs in a LangGraph Server are executed by a pool of background workers that are part of each deployment. In order to enable some features for those runs (such as cancellation and output streaming) we need a channel for two-way communication between the server and the worker handling a particular run. We use Redis to organize that communication.

1. A Redis list is used as a mechanism to wake up a worker as soon as a new run is created. Only a sentinel value is stored in this list, no actual run information. The run information is then retrieved from Postgres by the worker.
2. A combination of a Redis string and Redis PubSub channel is used for the server to communicate a run cancellation request to the appropriate worker.
3. A Redis PubSub channel is used by the worker to broadcast streaming output from an agent while the run is being handled. Any open `/stream` request in the server will subscribe to that channel and forward any events to the response as they arrive. No events are stored in Redis at any time.

### Ephemeral metadata

Runs in a LangGraph Server may be retried for specific failures (currently only for transient Postgres errors encountered during the run). In order to limit the number of retries (currently limited to 3 attempts per run) we record the attempt number in a Redis string when it is picked up. This contains no run-specific info other than its ID, and expires after a short delay.

## Data Plane Features

This section describes various features of the data plane.

### Data Region

!!! info "Only for Cloud SaaS"
    Data regions are only applicable for [Cloud SaaS](../concepts/langgraph_cloud.md) deployments.

Deployments can be created in 2 data regions: US and EU

The data region for a deployment is implied by the data region of the LangSmith organization where the deployment is created. Deployments and the underlying database for the deployments cannot be migrated between data regions.

### Autoscaling

[`Production` type](../concepts/langgraph_control_plane.md#deployment-types) deployments automatically scale up to 10 containers. Scaling is based on 3 metrics:

1. CPU utilization
1. Memory utilization
1. Number of pending (in progress) [runs](./assistants.md#execution)

For CPU utilization, the autoscaler targets 75% utilization. This means the autoscaler will scale the number of containers up or down to ensure that CPU utilization is at or near 75%. For memory utilization, the autoscaler targets 75% utilization as well.

For number of pending runs, the autoscaler targets 10 pending runs. For example, if the current number of containers is 1, but the number of pending runs in 20, the autoscaler will scale up the deployment to 2 containers (20 pending runs / 2 containers = 10 pending runs per container).

Each metric is computed independently and the autoscaler will determine the scaling action based on the metric that results in the largest number of containers.

Scale down actions are delayed for 30 minutes before any action is taken. In other words, if the autoscaler decides to scale down a deployment, it will first wait for 30 minutes before scaling down. After 30 minutes, the metrics are recomputed and the deployment will scale down if the recomputed metrics result in a lower number of containers than the current number. Otherwise, the deployment remains scaled up. This "cool down" period ensures that deployments do not scale up and down too frequently.

### Static IP Addresses

!!! info "Only for Cloud SaaS"
Static IP addresses are only available for [Cloud SaaS](../concepts/langgraph_cloud.md) deployments.

All traffic from deployments created after January 6th 2025 will come through a NAT gateway. This NAT gateway will have several static IP addresses depending on the data region. Refer to the table below for the list of static IP addresses:

| US             | EU             |
| -------------- | -------------- |
| 35.197.29.146  | 34.13.192.67   |
| 34.145.102.123 | 34.147.105.64  |
| 34.169.45.153  | 34.90.22.166   |
| 34.82.222.17   | 34.147.36.213  |
| 35.227.171.135 | 34.32.137.113  |
| 34.169.88.30   | 34.91.238.184  |
| 34.19.93.202   | 35.204.101.241 |
| 34.19.34.50    | 35.204.48.32   |

### Custom Postgres

!!! info
Custom Postgres instances are only available for [Self-Hosted Data Plane](../concepts/langgraph_self_hosted_data_plane.md) and [Self-Hosted Control Plane](../concepts/langgraph_self_hosted_control_plane.md) deployments.

A custom Postgres instance can be used instead of the [one automatically created by the control plane](./langgraph_control_plane.md#database-provisioning). Specify the [`POSTGRES_URI_CUSTOM`](../cloud/reference/env_var.md#postgres_uri_custom) environment variable to use a custom Postgres instance.

Multiple deployments can share the same Postgres instance. For example, for `Deployment A`, `POSTGRES_URI_CUSTOM` can be set to `postgres://<user>:<password>@/<database_name_1>?host=<hostname_1>` and for `Deployment B`, `POSTGRES_URI_CUSTOM` can be set to `postgres://<user>:<password>@/<database_name_2>?host=<hostname_1>`. `<database_name_1>` and `database_name_2` are different databases within the same instance, but `<hostname_1>` is shared. **The same database cannot be used for separate deployments**.

### Custom Redis

!!! info
Custom Redis instances are only available for [Self-Hosted Data Plane](../concepts/langgraph_self_hosted_control_plane.md) and [Self-Hosted Control Plane](../concepts/langgraph_self_hosted_control_plane.md) deployments.

A custom Redis instance can be used instead of the one automatically created by the control plane. Specify the [REDIS_URI_CUSTOM](../cloud/reference/env_var.md#redis_uri_custom) environment variable to use a custom Redis instance.

Multiple deployments can share the same Redis instance. For example, for `Deployment A`, `REDIS_URI_CUSTOM` can be set to `redis://<hostname_1>:<port>/1` and for `Deployment B`, `REDIS_URI_CUSTOM` can be set to `redis://<hostname_1>:<port>/2`. `1` and `2` are different database numbers within the same instance, but `<hostname_1>` is shared. **The same database number cannot be used for separate deployments**.

### LangSmith Tracing

LangGraph Server is automatically configured to send traces to LangSmith. See the table below for details with respect to each deployment option.

| Cloud SaaS                               | Self-Hosted Data Plane                                      | Self-Hosted Control Plane                                          | Standalone Container                                                                         |
| ---------------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| Required<br><br>Trace to LangSmith SaaS. | Optional<br><br>Disable tracing or trace to LangSmith SaaS. | Optional<br><br>Disable tracing or trace to Self-Hosted LangSmith. | Optional<br><br>Disable tracing, trace to LangSmith SaaS, or trace to Self-Hosted LangSmith. |

### Telemetry

LangGraph Server is automatically configured to report telemetry metadata for billing purposes. See the table below for details with respect to each deployment option.

| Cloud SaaS                        | Self-Hosted Data Plane            | Self-Hosted Control Plane                                                                                                           | Standalone Container                                                                                                                |
| --------------------------------- | --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Telemetry sent to LangSmith SaaS. | Telemetry sent to LangSmith SaaS. | Self-reported usage (audit) for air-gapped license key.<br><br>Telemetry sent to LangSmith SaaS for LangGraph Platform License Key. | Self-reported usage (audit) for air-gapped license key.<br><br>Telemetry sent to LangSmith SaaS for LangGraph Platform License Key. |

### Licensing

LangGraph Server is automatically configured to perform license key validation. See the table below for details with respect to each deployment option.

| Cloud SaaS                                          | Self-Hosted Data Plane                              | Self-Hosted Control Plane                                                                  | Standalone Container                                                                       |
| --------------------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------ |
| LangSmith API Key validated against LangSmith SaaS. | LangSmith API Key validated against LangSmith SaaS. | Air-gapped license key or LangGraph Platform License Key validated against LangSmith SaaS. | Air-gapped license key or LangGraph Platform License Key validated against LangSmith SaaS. |

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/langgraph_platform.md
```md
---
search:
  boost: 2
---

# LangGraph Platform

Develop, deploy, scale, and manage agents with **LangGraph Platform** — the purpose-built platform for long-running, agentic workflows.

!!! tip "Get started with LangGraph Platform"

    Check out the [LangGraph Platform quickstart](../tutorials/langgraph-platform/local-server.md) for instructions on how to use LangGraph Platform to run a LangGraph application locally.

## Why use LangGraph Platform?

<div align="center"><iframe width="560" height="315" src="https://www.youtube.com/embed/pfAQxBS5z88?si=XGS6Chydn6lhSO1S" title="What is LangGraph Platform?" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe></div>

LangGraph Platform makes it easy to get your agent running in production —  whether it’s built with LangGraph or another framework — so you can focus on your app logic, not infrastructure. Deploy with one click to get a live endpoint, and use our robust APIs and built-in task queues to handle production scale. 

- **[Streaming Support](../cloud/how-tos/streaming.md)**: As agents grow more sophisticated, they often benefit from streaming both token outputs and intermediate states back to the user. Without this, users are left waiting for potentially long operations with no feedback. LangGraph Server provides multiple streaming modes optimized for various application needs.

- **[Background Runs](../cloud/how-tos/background_run.md)**: For agents that take longer to process (e.g., hours), maintaining an open connection can be impractical. The LangGraph Server supports launching agent runs in the background and provides both polling endpoints and webhooks to monitor run status effectively.
 
- **Support for long runs**: Regular server setups often encounter timeouts or disruptions when handling requests that take a long time to complete. LangGraph Server’s API provides robust support for these tasks by sending regular heartbeat signals, preventing unexpected connection closures during prolonged processes.

- **Handling Burstiness**: Certain applications, especially those with real-time user interaction, may experience "bursty" request loads where numerous requests hit the server simultaneously. LangGraph Server includes a task queue, ensuring requests are handled consistently without loss, even under heavy loads.

- **[Double-texting](../cloud/how-tos/interrupt_concurrent.md)**: In user-driven applications, it’s common for users to send multiple messages rapidly. This “double texting” can disrupt agent flows if not handled properly. LangGraph Server offers built-in strategies to address and manage such interactions.

- **[Checkpointers and memory management](persistence.md#checkpoints)**: For agents needing persistence (e.g., conversation memory), deploying a robust storage solution can be complex. LangGraph Platform includes optimized [checkpointers](persistence.md#checkpoints) and a [memory store](persistence.md#memory-store), managing state across sessions without the need for custom solutions.

- **[Human-in-the-loop support](../cloud/how-tos/human_in_the_loop_breakpoint.md)**: In many applications, users require a way to intervene in agent processes. LangGraph Server provides specialized endpoints for human-in-the-loop scenarios, simplifying the integration of manual oversight into agent workflows.

- **[LangGraph Studio](./langgraph_studio.md)**: Enables visualization, interaction, and debugging of agentic systems that implement the LangGraph Server API protocol. Studio also integrates with LangSmith to enable tracing, evaluation, and prompt engineering.

- **[Deployment](./deployment_options.md)**: There are four ways to deploy on LangGraph Platform: [Cloud SaaS](../concepts/langgraph_cloud.md), [Self-Hosted Data Plane](../concepts/langgraph_self_hosted_data_plane.md), [Self-Hosted Control Plane](../concepts/langgraph_self_hosted_control_plane.md), and [Standalone Container](../concepts/langgraph_standalone_container.md).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/langgraph_self_hosted_control_plane.md
```md
# Self-Hosted Control Plane

There are two versions of the self-hosted deployment: [Self-Hosted Data Plane](./deployment_options.md#self-hosted-data-plane) and [Self-Hosted Control Plane](./deployment_options.md#self-hosted-control-plane).

!!! info "Important"

    The Self-Hosted Control Plane deployment option requires an [Enterprise](plans.md) plan.

## Requirements

- You use the [LangGraph CLI](./langgraph_cli.md) and/or [LangGraph Studio](./langgraph_studio.md) app to test graph locally.
- You use `langgraph build` command to build image.
- You have a Self-Hosted LangSmith instance deployed.
- You are using Ingress for your LangSmith instance. All agents will be deployed as Kubernetes services behind this ingress.

## Self-Hosted Control Plane

The [Self-Hosted Control Plane](./langgraph_self_hosted_control_plane.md) deployment option is a fully self-hosted model for deployment where you manage the [control plane](./langgraph_control_plane.md) and [data plane](./langgraph_data_plane.md) in your cloud. This option gives you full control and responsibility of the control plane and data plane infrastructure.

|                                    | [Control plane](../concepts/langgraph_control_plane.md)                                                                                     | [Data plane](../concepts/langgraph_data_plane.md)                                                                                                   |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **What is it?**                    | <ul><li>Control plane UI for creating deployments and revisions</li><li>Control plane APIs for creating deployments and revisions</li></ul> | <ul><li>Data plane "listener" for reconciling deployments with control plane state</li><li>LangGraph Servers</li><li>Postgres, Redis, etc</li></ul> |
| **Where is it hosted?**            | Your cloud                                                                                                                                  | Your cloud                                                                                                                                          |
| **Who provisions and manages it?** | You                                                                                                                                         | You                                                                                                                                                 |

### Architecture

![Self-Hosted Control Plane Architecture](./img/self_hosted_control_plane_architecture.png)

### Compute Platforms

- **Kubernetes**: The Self-Hosted Control Plane deployment option supports deploying control plane and data plane infrastructure to any Kubernetes cluster.

!!! tip
If you would like to enable this on your LangSmith instance, please follow the [Self-Hosted Control Plane deployment guide](../cloud/deployment/self_hosted_control_plane.md).

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/langgraph_self_hosted_data_plane.md
```md
---
search:
  boost: 2
---

# Self-Hosted Data Plane

There are two versions of the self-hosted deployment: [Self-Hosted Data Plane](./deployment_options.md#self-hosted-data-plane) and [Self-Hosted Control Plane](./deployment_options.md#self-hosted-control-plane).

!!! info "Important"

    The Self-Hosted Data Plane deployment option requires an [Enterprise](plans.md) plan.

## Requirements

- You use `langgraph-cli` and/or [LangGraph Studio](./langgraph_studio.md) app to test graph locally.
- You use `langgraph build` command to build image.

## Self-Hosted Data Plane

The [Self-Hosted Data Plane](../cloud/deployment/self_hosted_data_plane.md) deployment option is a "hybrid" model for deployment where we manage the [control plane](./langgraph_control_plane.md) in our cloud and you manage the [data plane](./langgraph_data_plane.md) in your cloud. This option provides a way to securely manage your data plane infrastructure, while offloading control plane management to us. When using the Self-Hosted Data Plane version, you authenticate with a [LangSmith](https://smith.langchain.com/) API key.

|                                    | [Control plane](../concepts/langgraph_control_plane.md)                                                                                     | [Data plane](../concepts/langgraph_data_plane.md)                                                                                                   |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **What is it?**                    | <ul><li>Control plane UI for creating deployments and revisions</li><li>Control plane APIs for creating deployments and revisions</li></ul> | <ul><li>Data plane "listener" for reconciling deployments with control plane state</li><li>LangGraph Servers</li><li>Postgres, Redis, etc</li></ul> |
| **Where is it hosted?**            | LangChain's cloud                                                                                                                           | Your cloud                                                                                                                                          |
| **Who provisions and manages it?** | LangChain                                                                                                                                   | You                                                                                                                                                 |

For information on how to deploy a [LangGraph Server](../concepts/langgraph_server.md) to Self-Hosted Data Plane, see [Deploy to Self-Hosted Data Plane](../cloud/deployment/self_hosted_data_plane.md)

### Architecture

![Self-Hosted Data Plane Architecture](./img/self_hosted_data_plane_architecture.png)

### Compute Platforms

- **Kubernetes**: The Self-Hosted Data Plane deployment option supports deploying data plane infrastructure to any Kubernetes cluster.
- **Amazon ECS**: Coming soon!

!!! tip
If you would like to deploy to Kubernetes, you can follow the [Self-Hosted Data Plane deployment guide](../cloud/deployment/self_hosted_data_plane.md).

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/langgraph_server.md
```md
---
search:
  boost: 2
---

# LangGraph Server

**LangGraph Server** offers an API for creating and managing agent-based applications. It is built on the concept of [assistants](assistants.md), which are agents configured for specific tasks, and includes built-in [persistence](persistence.md#memory-store) and a **task queue**. This versatile API supports a wide range of agentic application use cases, from background processing to real-time interactions.

Use LangGraph Server to create and manage [assistants](assistants.md), [threads](./persistence.md#threads), [runs](./assistants.md#execution), [cron jobs](../cloud/concepts/cron_jobs.md), [webhooks](../cloud/concepts/webhooks.md), and more.

!!! tip "API reference"
  
    For detailed information on the API endpoints and data models, see [LangGraph Platform API reference docs](../cloud/reference/api/api_ref.html).

## Application structure

To deploy a LangGraph Server application, you need to specify the graph(s) you want to deploy, as well as any relevant configuration settings, such as dependencies and environment variables.

Read the [application structure](./application_structure.md) guide to learn how to structure your LangGraph application for deployment.

## Parts of a deployment

When you deploy LangGraph Server, you are deploying one or more [graphs](#graphs), a database for [persistence](persistence.md), and a task queue.

### Graphs

When you deploy a graph with LangGraph Server, you are deploying a "blueprint" for an [Assistant](assistants.md). 

An [Assistant](assistants.md) is a graph paired with specific configuration settings. You can create multiple assistants per graph, each with unique settings to accommodate different use cases
that can be served by the same graph.

Upon deployment, LangGraph Server will automatically create a default assistant for each graph using the graph's default configuration settings.

!!! note

    We often think of a graph as implementing an [agent](agentic_concepts.md), but a graph does not necessarily need to implement an agent. For example, a graph could implement a simple
    chatbot that only supports back-and-forth conversation, without the ability to influence any application control flow. In reality, as applications get more complex, a graph will often implement a more complex flow that may use [multiple agents](./multi_agent.md) working in tandem.

### Persistence and task queue

LangGraph Server leverages a database for [persistence](persistence.md) and a task queue.

Currently, only [Postgres](https://www.postgresql.org/) is supported as a database for LangGraph Server and [Redis](https://redis.io/) as the task queue.

If you're deploying using [LangGraph Platform](./langgraph_cloud.md), these components are managed for you. If you're deploying LangGraph Server on your own infrastructure, you'll need to set up and manage these components yourself.

Please review the [deployment options](./deployment_options.md) guide for more information on how these components are set up and managed.

## Learn more

* LangGraph [Application Structure](./application_structure.md) guide explains how to structure your LangGraph application for deployment.
* The [LangGraph Platform API Reference](../cloud/reference/api/api_ref.html) provides detailed information on the API endpoints and data models.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/langgraph_standalone_container.md
```md
---
search:
  boost: 2
---

# Standalone Container

To deploy a [LangGraph Server](../concepts/langgraph_server.md), follow the how-to guide for [how to deploy a Standalone Container](../cloud/deployment/standalone_container.md).

## Overview

The Standalone Container deployment option is the least restrictive model for deployment. There is no [control plane](./langgraph_control_plane.md). [Data plane](./langgraph_data_plane.md) infrastructure is managed by you.

|                   | [Control plane](../concepts/langgraph_control_plane.md) | [Data plane](../concepts/langgraph_data_plane.md) |
|-------------------|-------------------|------------|
| **What is it?** | n/a | <ul><li>LangGraph Servers</li><li>Postgres, Redis, etc</li></ul> |
| **Where is it hosted?** | n/a | Your cloud |
| **Who provisions and manages it?** | n/a | You |

!!! warning

      LangGraph Platform should not be deployed in serverless environments. Scale to zero may cause task loss and scaling up will not work reliably.

## Architecture

![Standalone Container](./img/langgraph_platform_deployment_architecture.png)

## Compute Platforms

### Kubernetes

The Standalone Container deployment option supports deploying data plane infrastructure to a Kubernetes cluster.

### Docker

The Standalone Container deployment option supports deploying data plane infrastructure to any Docker-supported compute platform.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/langgraph_studio.md
```md
---
search:
  boost: 2
---

# LangGraph Studio

!!! info "Prerequisites"

    - [LangGraph Platform](./langgraph_platform.md)
    - [LangGraph Server](./langgraph_server.md)
    - [LangGraph CLI](./langgraph_cli.md)

LangGraph Studio is a specialized agent IDE that enables visualization, interaction, and debugging of agentic systems that implement the LangGraph Server API protocol. Studio also integrates with LangSmith to enable tracing, evaluation, and prompt engineering.

![](img/lg_studio.png)

## Features

Key features of LangGraph Studio:

- Visualize your graph architecture
- [Run and interact with your agent](../cloud/how-tos/invoke_studio.md)
- [Manage assistants](../cloud/how-tos/studio/manage_assistants.md)
- [Manage threads](../cloud/how-tos/threads_studio.md)
- [Iterate on prompts](../cloud/how-tos/iterate_graph_studio.md)
- [Run experiments over a dataset](../cloud/how-tos/studio/run_evals.md)
- Manage [long term memory](memory.md)
- Debug agent state via [time travel](time-travel.md)

LangGraph Studio works for graphs that are deployed on [LangGraph Platform](../cloud/quick_start.md) or for graphs that are running locally via the [LangGraph Server](../tutorials/langgraph-platform/local-server.md).

Studio supports two modes:

### Graph mode

Graph mode exposes the full feature-set of Studio and is useful when you would like as many details about the execution of your agent, including the nodes traversed, intermediate states, and LangSmith integrations (such as adding to datasets and playground).

### Chat mode

Chat mode is a simpler UI for iterating on and testing chat-specific agents. It is useful for business users and those who want to test overall agent behavior. Chat mode is only supported for graph's whose state includes or extends [`MessagesState`](https://langchain-ai.github.io/langgraph/how-tos/graph-api/#messagesstate).

## Learn more

- See this guide on how to [get started](../cloud/how-tos/studio/quick_start.md) with LangGraph Studio.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/low_level.md
```md
---
search:
  boost: 2
---

# Graph API concepts

## Graphs

At its core, LangGraph models agent workflows as graphs. You define the behavior of your agents using three key components:

1. [`State`](#state): A shared data structure that represents the current snapshot of your application. It can be any data type, but is typically defined using a shared state schema.

2. [`Nodes`](#nodes): Functions that encode the logic of your agents. They receive the current state as input, perform some computation or side-effect, and return an updated state.

3. [`Edges`](#edges): Functions that determine which `Node` to execute next based on the current state. They can be conditional branches or fixed transitions.

By composing `Nodes` and `Edges`, you can create complex, looping workflows that evolve the state over time. The real power, though, comes from how LangGraph manages that state. To emphasize: `Nodes` and `Edges` are nothing more than functions - they can contain an LLM or just good ol' code.

In short: _nodes do the work, edges tell what to do next_.

LangGraph's underlying graph algorithm uses [message passing](https://en.wikipedia.org/wiki/Message_passing) to define a general program. When a Node completes its operation, it sends messages along one or more edges to other node(s). These recipient nodes then execute their functions, pass the resulting messages to the next set of nodes, and the process continues. Inspired by Google's [Pregel](https://research.google/pubs/pregel-a-system-for-large-scale-graph-processing/) system, the program proceeds in discrete "super-steps."

A super-step can be considered a single iteration over the graph nodes. Nodes that run in parallel are part of the same super-step, while nodes that run sequentially belong to separate super-steps. At the start of graph execution, all nodes begin in an `inactive` state. A node becomes `active` when it receives a new message (state) on any of its incoming edges (or "channels"). The active node then runs its function and responds with updates. At the end of each super-step, nodes with no incoming messages vote to `halt` by marking themselves as `inactive`. The graph execution terminates when all nodes are `inactive` and no messages are in transit.

### StateGraph

The `StateGraph` class is the main graph class to use. This is parameterized by a user defined `State` object.

### Compiling your graph

To build your graph, you first define the [state](#state), you then add [nodes](#nodes) and [edges](#edges), and then you compile it. What exactly is compiling your graph and why is it needed?

Compiling is a pretty simple step. It provides a few basic checks on the structure of your graph (no orphaned nodes, etc). It is also where you can specify runtime args like [checkpointers](./persistence.md) and breakpoints. You compile your graph by just calling the `.compile` method:

:::python

```python
graph = graph_builder.compile(...)
```

:::

:::js

```typescript
const graph = new StateGraph(StateAnnotation)
  .addNode("nodeA", nodeA)
  .addEdge(START, "nodeA")
  .addEdge("nodeA", END)
  .compile();
```

:::

You **MUST** compile your graph before you can use it.

## State

:::python
The first thing you do when you define a graph is define the `State` of the graph. The `State` consists of the [schema of the graph](#schema) as well as [`reducer` functions](#reducers) which specify how to apply updates to the state. The schema of the `State` will be the input schema to all `Nodes` and `Edges` in the graph, and can be either a `TypedDict` or a `Pydantic` model. All `Nodes` will emit updates to the `State` which are then applied using the specified `reducer` function.
:::

:::js
The first thing you do when you define a graph is define the `State` of the graph. The `State` consists of the [schema of the graph](#schema) as well as [`reducer` functions](#reducers) which specify how to apply updates to the state. The schema of the `State` will be the input schema to all `Nodes` and `Edges` in the graph, and can be either a Zod schema or a schema built using `Annotation.Root`. All `Nodes` will emit updates to the `State` which are then applied using the specified `reducer` function.
:::

### Schema

:::python
The main documented way to specify the schema of a graph is by using a [`TypedDict`](https://docs.python.org/3/library/typing.html#typing.TypedDict). If you want to provide default values in your state, use a [`dataclass`](https://docs.python.org/3/library/dataclasses.html). We also support using a Pydantic [BaseModel](../how-tos/graph-api.md#use-pydantic-models-for-graph-state) as your graph state if you want recursive data validation (though note that pydantic is less performant than a `TypedDict` or `dataclass`).

By default, the graph will have the same input and output schemas. If you want to change this, you can also specify explicit input and output schemas directly. This is useful when you have a lot of keys, and some are explicitly for input and others for output. See the [guide here](../how-tos/graph-api.md#define-input-and-output-schemas) for how to use.
:::

:::js
The main documented way to specify the schema of a graph is by using Zod schemas. However, we also support using the `Annotation` API to define the schema of the graph.

By default, the graph will have the same input and output schemas. If you want to change this, you can also specify explicit input and output schemas directly. This is useful when you have a lot of keys, and some are explicitly for input and others for output.
:::

#### Multiple schemas

Typically, all graph nodes communicate with a single schema. This means that they will read and write to the same state channels. But, there are cases where we want more control over this:

- Internal nodes can pass information that is not required in the graph's input / output.
- We may also want to use different input / output schemas for the graph. The output might, for example, only contain a single relevant output key.

It is possible to have nodes write to private state channels inside the graph for internal node communication. We can simply define a private schema, `PrivateState`.

It is also possible to define explicit input and output schemas for a graph. In these cases, we define an "internal" schema that contains _all_ keys relevant to graph operations. But, we also define `input` and `output` schemas that are sub-sets of the "internal" schema to constrain the input and output of the graph. See [this guide](../how-tos/graph-api.md#define-input-and-output-schemas) for more detail.

Let's look at an example:

:::python

```python
class InputState(TypedDict):
    user_input: str

class OutputState(TypedDict):
    graph_output: str

class OverallState(TypedDict):
    foo: str
    user_input: str
    graph_output: str

class PrivateState(TypedDict):
    bar: str

def node_1(state: InputState) -> OverallState:
    # Write to OverallState
    return {"foo": state["user_input"] + " name"}

def node_2(state: OverallState) -> PrivateState:
    # Read from OverallState, write to PrivateState
    return {"bar": state["foo"] + " is"}

def node_3(state: PrivateState) -> OutputState:
    # Read from PrivateState, write to OutputState
    return {"graph_output": state["bar"] + " Lance"}

builder = StateGraph(OverallState,input_schema=InputState,output_schema=OutputState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_node("node_3", node_3)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", "node_3")
builder.add_edge("node_3", END)

graph = builder.compile()
graph.invoke({"user_input":"My"})
# {'graph_output': 'My name is Lance'}
```

:::

:::js

```typescript
const InputState = z.object({
  userInput: z.string(),
});

const OutputState = z.object({
  graphOutput: z.string(),
});

const OverallState = z.object({
  foo: z.string(),
  userInput: z.string(),
  graphOutput: z.string(),
});

const PrivateState = z.object({
  bar: z.string(),
});

const graph = new StateGraph({
  state: OverallState,
  input: InputState,
  output: OutputState,
})
  .addNode("node1", (state) => {
    // Write to OverallState
    return { foo: state.userInput + " name" };
  })
  .addNode("node2", (state) => {
    // Read from OverallState, write to PrivateState
    return { bar: state.foo + " is" };
  })
  .addNode(
    "node3",
    (state) => {
      // Read from PrivateState, write to OutputState
      return { graphOutput: state.bar + " Lance" };
    },
    { input: PrivateState }
  )
  .addEdge(START, "node1")
  .addEdge("node1", "node2")
  .addEdge("node2", "node3")
  .addEdge("node3", END)
  .compile();

await graph.invoke({ userInput: "My" });
// { graphOutput: 'My name is Lance' }
```

:::

There are two subtle and important points to note here:

:::python

1. We pass `state: InputState` as the input schema to `node_1`. But, we write out to `foo`, a channel in `OverallState`. How can we write out to a state channel that is not included in the input schema? This is because a node _can write to any state channel in the graph state._ The graph state is the union of the state channels defined at initialization, which includes `OverallState` and the filters `InputState` and `OutputState`.

2. We initialize the graph with `StateGraph(OverallState,input_schema=InputState,output_schema=OutputState)`. So, how can we write to `PrivateState` in `node_2`? How does the graph gain access to this schema if it was not passed in the `StateGraph` initialization? We can do this because _nodes can also declare additional state channels_ as long as the state schema definition exists. In this case, the `PrivateState` schema is defined, so we can add `bar` as a new state channel in the graph and write to it.
   :::

:::js

1. We pass `state` as the input schema to `node1`. But, we write out to `foo`, a channel in `OverallState`. How can we write out to a state channel that is not included in the input schema? This is because a node _can write to any state channel in the graph state._ The graph state is the union of the state channels defined at initialization, which includes `OverallState` and the filters `InputState` and `OutputState`.

2. We initialize the graph with `StateGraph({ state: OverallState, input: InputState, output: OutputState })`. So, how can we write to `PrivateState` in `node2`? How does the graph gain access to this schema if it was not passed in the `StateGraph` initialization? We can do this because _nodes can also declare additional state channels_ as long as the state schema definition exists. In this case, the `PrivateState` schema is defined, so we can add `bar` as a new state channel in the graph and write to it.
   :::

### Reducers

Reducers are key to understanding how updates from nodes are applied to the `State`. Each key in the `State` has its own independent reducer function. If no reducer function is explicitly specified then it is assumed that all updates to that key should override it. There are a few different types of reducers, starting with the default type of reducer:

#### Default Reducer

These two examples show how to use the default reducer:

**Example A:**

:::python

```python
from typing_extensions import TypedDict

class State(TypedDict):
    foo: int
    bar: list[str]
```

:::

:::js

```typescript
const State = z.object({
  foo: z.number(),
  bar: z.array(z.string()),
});
```

:::

In this example, no reducer functions are specified for any key. Let's assume the input to the graph is:

:::python
`{"foo": 1, "bar": ["hi"]}`. Let's then assume the first `Node` returns `{"foo": 2}`. This is treated as an update to the state. Notice that the `Node` does not need to return the whole `State` schema - just an update. After applying this update, the `State` would then be `{"foo": 2, "bar": ["hi"]}`. If the second node returns `{"bar": ["bye"]}` then the `State` would then be `{"foo": 2, "bar": ["bye"]}`
:::

:::js
`{ foo: 1, bar: ["hi"] }`. Let's then assume the first `Node` returns `{ foo: 2 }`. This is treated as an update to the state. Notice that the `Node` does not need to return the whole `State` schema - just an update. After applying this update, the `State` would then be `{ foo: 2, bar: ["hi"] }`. If the second node returns `{ bar: ["bye"] }` then the `State` would then be `{ foo: 2, bar: ["bye"] }`
:::

**Example B:**

:::python

```python
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: int
    bar: Annotated[list[str], add]
```

In this example, we've used the `Annotated` type to specify a reducer function (`operator.add`) for the second key (`bar`). Note that the first key remains unchanged. Let's assume the input to the graph is `{"foo": 1, "bar": ["hi"]}`. Let's then assume the first `Node` returns `{"foo": 2}`. This is treated as an update to the state. Notice that the `Node` does not need to return the whole `State` schema - just an update. After applying this update, the `State` would then be `{"foo": 2, "bar": ["hi"]}`. If the second node returns `{"bar": ["bye"]}` then the `State` would then be `{"foo": 2, "bar": ["hi", "bye"]}`. Notice here that the `bar` key is updated by adding the two lists together.
:::

:::js

```typescript
import { z } from "zod";
import { withLangGraph } from "@langchain/langgraph/zod";

const State = z.object({
  foo: z.number(),
  bar: withLangGraph(z.array(z.string()), {
    reducer: {
      fn: (x, y) => x.concat(y),
    },
  }),
});
```

In this example, we've used the `withLangGraph` function to specify a reducer function for the second key (`bar`). Note that the first key remains unchanged. Let's assume the input to the graph is `{ foo: 1, bar: ["hi"] }`. Let's then assume the first `Node` returns `{ foo: 2 }`. This is treated as an update to the state. Notice that the `Node` does not need to return the whole `State` schema - just an update. After applying this update, the `State` would then be `{ foo: 2, bar: ["hi"] }`. If the second node returns `{ bar: ["bye"] }` then the `State` would then be `{ foo: 2, bar: ["hi", "bye"] }`. Notice here that the `bar` key is updated by adding the two arrays together.
:::

### Working with Messages in Graph State

#### Why use messages?

:::python
Most modern LLM providers have a chat model interface that accepts a list of messages as input. LangChain's [`ChatModel`](https://python.langchain.com/docs/concepts/#chat-models) in particular accepts a list of `Message` objects as inputs. These messages come in a variety of forms such as `HumanMessage` (user input) or `AIMessage` (LLM response). To read more about what message objects are, please refer to [this](https://python.langchain.com/docs/concepts/#messages) conceptual guide.
:::

:::js
Most modern LLM providers have a chat model interface that accepts a list of messages as input. LangChain's [`ChatModel`](https://js.langchain.com/docs/concepts/#chat-models) in particular accepts a list of `Message` objects as inputs. These messages come in a variety of forms such as `HumanMessage` (user input) or `AIMessage` (LLM response). To read more about what message objects are, please refer to [this](https://js.langchain.com/docs/concepts/#messages) conceptual guide.
:::

#### Using Messages in your Graph

:::python
In many cases, it is helpful to store prior conversation history as a list of messages in your graph state. To do so, we can add a key (channel) to the graph state that stores a list of `Message` objects and annotate it with a reducer function (see `messages` key in the example below). The reducer function is vital to telling the graph how to update the list of `Message` objects in the state with each state update (for example, when a node sends an update). If you don't specify a reducer, every state update will overwrite the list of messages with the most recently provided value. If you wanted to simply append messages to the existing list, you could use `operator.add` as a reducer.

However, you might also want to manually update messages in your graph state (e.g. human-in-the-loop). If you were to use `operator.add`, the manual state updates you send to the graph would be appended to the existing list of messages, instead of updating existing messages. To avoid that, you need a reducer that can keep track of message IDs and overwrite existing messages, if updated. To achieve this, you can use the prebuilt `add_messages` function. For brand new messages, it will simply append to existing list, but it will also handle the updates for existing messages correctly.
:::

:::js
In many cases, it is helpful to store prior conversation history as a list of messages in your graph state. To do so, we can add a key (channel) to the graph state that stores a list of `Message` objects and annotate it with a reducer function (see `messages` key in the example below). The reducer function is vital to telling the graph how to update the list of `Message` objects in the state with each state update (for example, when a node sends an update). If you don't specify a reducer, every state update will overwrite the list of messages with the most recently provided value. If you wanted to simply append messages to the existing list, you could use a function that concatenates arrays as a reducer.

However, you might also want to manually update messages in your graph state (e.g. human-in-the-loop). If you were to use a simple concatenation function, the manual state updates you send to the graph would be appended to the existing list of messages, instead of updating existing messages. To avoid that, you need a reducer that can keep track of message IDs and overwrite existing messages, if updated. To achieve this, you can use the prebuilt `MessagesZodState` schema. For brand new messages, it will simply append to existing list, but it will also handle the updates for existing messages correctly.
:::

#### Serialization

:::python
In addition to keeping track of message IDs, the `add_messages` function will also try to deserialize messages into LangChain `Message` objects whenever a state update is received on the `messages` channel. See more information on LangChain serialization/deserialization [here](https://python.langchain.com/docs/how_to/serialization/). This allows sending graph inputs / state updates in the following format:

```python
# this is supported
{"messages": [HumanMessage(content="message")]}

# and this is also supported
{"messages": [{"type": "human", "content": "message"}]}
```

Since the state updates are always deserialized into LangChain `Messages` when using `add_messages`, you should use dot notation to access message attributes, like `state["messages"][-1].content`. Below is an example of a graph that uses `add_messages` as its reducer function.

```python
from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages
from typing import Annotated
from typing_extensions import TypedDict

class GraphState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

:::

:::js
In addition to keeping track of message IDs, `MessagesZodState` will also try to deserialize messages into LangChain `Message` objects whenever a state update is received on the `messages` channel. This allows sending graph inputs / state updates in the following format:

```typescript
// this is supported
{
  messages: [new HumanMessage("message")];
}

// and this is also supported
{
  messages: [{ role: "human", content: "message" }];
}
```

Since the state updates are always deserialized into LangChain `Messages` when using `MessagesZodState`, you should use dot notation to access message attributes, like `state.messages[state.messages.length - 1].content`. Below is an example of a graph that uses `MessagesZodState`:

```typescript
import { StateGraph, MessagesZodState } from "@langchain/langgraph";

const graph = new StateGraph(MessagesZodState)
  ...
```

`MessagesZodState` is defined with a single `messages` key which is a list of `BaseMessage` objects and uses the appropriate reducer. Typically, there is more state to track than just messages, so we see people extend this state and add more fields, like:

```typescript
const State = z.object({
  messages: MessagesZodState.shape.messages,
  documents: z.array(z.string()),
});
```

:::

:::python

#### MessagesState

Since having a list of messages in your state is so common, there exists a prebuilt state called `MessagesState` which makes it easy to use messages. `MessagesState` is defined with a single `messages` key which is a list of `AnyMessage` objects and uses the `add_messages` reducer. Typically, there is more state to track than just messages, so we see people subclass this state and add more fields, like:

```python
from langgraph.graph import MessagesState

class State(MessagesState):
    documents: list[str]
```

:::

## Nodes

:::python

In LangGraph, nodes are Python functions (either synchronous or asynchronous) that accept the following arguments:

1. `state`: The [state](#state) of the graph
2. `config`: A `RunnableConfig` object that contains configuration information like `thread_id` and tracing information like `tags`
3. `runtime`: A `Runtime` object that contains [runtime `context`](#runtime-context) and other information like `store` and `stream_writer`

Similar to `NetworkX`, you add these nodes to a graph using the @[add_node][add_node] method:

```python
from dataclasses import dataclass
from typing_extensions import TypedDict

from langchain_core.runnables import RunnableConfig
from langgraph.graph import StateGraph
from langgraph.runtime import Runtime

class State(TypedDict):
    input: str
    results: str

@dataclass
class Context:
    user_id: str

builder = StateGraph(State)

def plain_node(state: State):
    return state

def node_with_runtime(state: State, runtime: Runtime[Context]):
    print("In node: ", runtime.context.user_id)
    return {"results": f"Hello, {state['input']}!"}

def node_with_config(state: State, config: RunnableConfig):
    print("In node with thread_id: ", config["configurable"]["thread_id"])
    return {"results": f"Hello, {state['input']}!"}


builder.add_node("plain_node", plain_node)
builder.add_node("node_with_runtime", node_with_runtime)
builder.add_node("node_with_config", node_with_config)
...
```

:::

:::js

In LangGraph, nodes are typically functions (sync or async) that accept the following arguments:

1. `state`: The [state](#state) of the graph
2. `config`: A `RunnableConfig` object that contains configuration information like `thread_id` and tracing information like `tags`

You can add nodes to a graph using the `addNode` method.

```typescript
import { StateGraph } from "@langchain/langgraph";
import { RunnableConfig } from "@langchain/core/runnables";
import { z } from "zod";

const State = z.object({
  input: z.string(),
  results: z.string(),
});

const builder = new StateGraph(State);
  .addNode("myNode", (state, config) => {
    console.log("In node: ", config?.configurable?.user_id);
    return { results: `Hello, ${state.input}!` };
  })
  addNode("otherNode", (state) => {
    return state;
  })
  ...
```

:::

Behind the scenes, functions are converted to [RunnableLambda](https://python.langchain.com/api_reference/core/runnables/langchain_core.runnables.base.RunnableLambda.html)s, which add batch and async support to your function, along with native tracing and debugging.

If you add a node to a graph without specifying a name, it will be given a default name equivalent to the function name.

:::python

```python
builder.add_node(my_node)
# You can then create edges to/from this node by referencing it as `"my_node"`
```

:::

:::js

```typescript
builder.addNode(myNode);
// You can then create edges to/from this node by referencing it as `"myNode"`
```

:::

### `START` Node

The `START` Node is a special node that represents the node that sends user input to the graph. The main purpose for referencing this node is to determine which nodes should be called first.

:::python

```python
from langgraph.graph import START

graph.add_edge(START, "node_a")
```

:::

:::js

```typescript
import { START } from "@langchain/langgraph";

graph.addEdge(START, "nodeA");
```

:::

### `END` Node

The `END` Node is a special node that represents a terminal node. This node is referenced when you want to denote which edges have no actions after they are done.

:::python

```python
from langgraph.graph import END

graph.add_edge("node_a", END)
```

:::

:::js

```typescript
import { END } from "@langchain/langgraph";

graph.addEdge("nodeA", END);
```

:::

### Node Caching

:::python
LangGraph supports caching of tasks/nodes based on the input to the node. To use caching:

- Specify a cache when compiling a graph (or specifying an entrypoint)
- Specify a cache policy for nodes. Each cache policy supports:
  - `key_func` used to generate a cache key based on the input to a node, which defaults to a `hash` of the input with pickle.
  - `ttl`, the time to live for the cache in seconds. If not specified, the cache will never expire.

For example:

```python
import time
from typing_extensions import TypedDict
from langgraph.graph import StateGraph
from langgraph.cache.memory import InMemoryCache
from langgraph.types import CachePolicy


class State(TypedDict):
    x: int
    result: int


builder = StateGraph(State)


def expensive_node(state: State) -> dict[str, int]:
    # expensive computation
    time.sleep(2)
    return {"result": state["x"] * 2}


builder.add_node("expensive_node", expensive_node, cache_policy=CachePolicy(ttl=3))
builder.set_entry_point("expensive_node")
builder.set_finish_point("expensive_node")

graph = builder.compile(cache=InMemoryCache())

print(graph.invoke({"x": 5}, stream_mode='updates'))  # (1)!
[{'expensive_node': {'result': 10}}]
print(graph.invoke({"x": 5}, stream_mode='updates'))  # (2)!
[{'expensive_node': {'result': 10}, '__metadata__': {'cached': True}}]
```

1. First run takes two seconds to run (due to mocked expensive computation).
2. Second run utilizes cache and returns quickly.
   :::

:::js
LangGraph supports caching of tasks/nodes based on the input to the node. To use caching:

- Specify a cache when compiling a graph (or specifying an entrypoint)
- Specify a cache policy for nodes. Each cache policy supports:
  - `keyFunc`, which is used to generate a cache key based on the input to a node.
  - `ttl`, the time to live for the cache in seconds. If not specified, the cache will never expire.

```typescript
import { StateGraph, MessagesZodState } from "@langchain/langgraph";
import { InMemoryCache } from "@langchain/langgraph-checkpoint";

const graph = new StateGraph(MessagesZodState)
  .addNode(
    "expensive_node",
    async () => {
      // Simulate an expensive operation
      await new Promise((resolve) => setTimeout(resolve, 3000));
      return { result: 10 };
    },
    { cachePolicy: { ttl: 3 } }
  )
  .addEdge(START, "expensive_node")
  .compile({ cache: new InMemoryCache() });

await graph.invoke({ x: 5 }, { streamMode: "updates" }); // (1)!
// [{"expensive_node": {"result": 10}}]
await graph.invoke({ x: 5 }, { streamMode: "updates" }); // (2)!
// [{"expensive_node": {"result": 10}, "__metadata__": {"cached": true}}]
```

:::

## Edges

Edges define how the logic is routed and how the graph decides to stop. This is a big part of how your agents work and how different nodes communicate with each other. There are a few key types of edges:

- Normal Edges: Go directly from one node to the next.
- Conditional Edges: Call a function to determine which node(s) to go to next.
- Entry Point: Which node to call first when user input arrives.
- Conditional Entry Point: Call a function to determine which node(s) to call first when user input arrives.

A node can have MULTIPLE outgoing edges. If a node has multiple out-going edges, **all** of those destination nodes will be executed in parallel as a part of the next superstep.

### Normal Edges

:::python
If you **always** want to go from node A to node B, you can use the @[add_edge][add_edge] method directly.

```python
graph.add_edge("node_a", "node_b")
```

:::

:::js
If you **always** want to go from node A to node B, you can use the @[`addEdge`][add_edge] method directly.

```typescript
graph.addEdge("nodeA", "nodeB");
```

:::

### Conditional Edges

:::python
If you want to **optionally** route to 1 or more edges (or optionally terminate), you can use the @[add_conditional_edges][add_conditional_edges] method. This method accepts the name of a node and a "routing function" to call after that node is executed:

```python
graph.add_conditional_edges("node_a", routing_function)
```

Similar to nodes, the `routing_function` accepts the current `state` of the graph and returns a value.

By default, the return value `routing_function` is used as the name of the node (or list of nodes) to send the state to next. All those nodes will be run in parallel as a part of the next superstep.

You can optionally provide a dictionary that maps the `routing_function`'s output to the name of the next node.

```python
graph.add_conditional_edges("node_a", routing_function, {True: "node_b", False: "node_c"})
```

:::

:::js
If you want to **optionally** route to 1 or more edges (or optionally terminate), you can use the @[`addConditionalEdges`][add_conditional_edges] method. This method accepts the name of a node and a "routing function" to call after that node is executed:

```typescript
graph.addConditionalEdges("nodeA", routingFunction);
```

Similar to nodes, the `routingFunction` accepts the current `state` of the graph and returns a value.

By default, the return value `routingFunction` is used as the name of the node (or list of nodes) to send the state to next. All those nodes will be run in parallel as a part of the next superstep.

You can optionally provide an object that maps the `routingFunction`'s output to the name of the next node.

```typescript
graph.addConditionalEdges("nodeA", routingFunction, {
  true: "nodeB",
  false: "nodeC",
});
```

:::

!!! tip

    Use [`Command`](#command) instead of conditional edges if you want to combine state updates and routing in a single function.

### Entry Point

:::python
The entry point is the first node(s) that are run when the graph starts. You can use the @[`add_edge`][add_edge] method from the virtual @[`START`][START] node to the first node to execute to specify where to enter the graph.

```python
from langgraph.graph import START

graph.add_edge(START, "node_a")
```

:::

:::js
The entry point is the first node(s) that are run when the graph starts. You can use the @[`addEdge`][add_edge] method from the virtual @[`START`][START] node to the first node to execute to specify where to enter the graph.

```typescript
import { START } from "@langchain/langgraph";

graph.addEdge(START, "nodeA");
```

:::

### Conditional Entry Point

:::python
A conditional entry point lets you start at different nodes depending on custom logic. You can use @[`add_conditional_edges`][add_conditional_edges] from the virtual @[`START`][START] node to accomplish this.

```python
from langgraph.graph import START

graph.add_conditional_edges(START, routing_function)
```

You can optionally provide a dictionary that maps the `routing_function`'s output to the name of the next node.

```python
graph.add_conditional_edges(START, routing_function, {True: "node_b", False: "node_c"})
```

:::

:::js
A conditional entry point lets you start at different nodes depending on custom logic. You can use @[`addConditionalEdges`][add_conditional_edges] from the virtual @[`START`][START] node to accomplish this.

```typescript
import { START } from "@langchain/langgraph";

graph.addConditionalEdges(START, routingFunction);
```

You can optionally provide an object that maps the `routingFunction`'s output to the name of the next node.

```typescript
graph.addConditionalEdges(START, routingFunction, {
  true: "nodeB",
  false: "nodeC",
});
```

:::

## `Send`

:::python
By default, `Nodes` and `Edges` are defined ahead of time and operate on the same shared state. However, there can be cases where the exact edges are not known ahead of time and/or you may want different versions of `State` to exist at the same time. A common example of this is with [map-reduce](https://langchain-ai.github.io/langgraph/how-tos/map-reduce/) design patterns. In this design pattern, a first node may generate a list of objects, and you may want to apply some other node to all those objects. The number of objects may be unknown ahead of time (meaning the number of edges may not be known) and the input `State` to the downstream `Node` should be different (one for each generated object).

To support this design pattern, LangGraph supports returning @[`Send`][Send] objects from conditional edges. `Send` takes two arguments: first is the name of the node, and second is the state to pass to that node.

```python
def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state['subjects']]

graph.add_conditional_edges("node_a", continue_to_jokes)
```

:::

:::js
By default, `Nodes` and `Edges` are defined ahead of time and operate on the same shared state. However, there can be cases where the exact edges are not known ahead of time and/or you may want different versions of `State` to exist at the same time. A common example of this is with map-reduce design patterns. In this design pattern, a first node may generate a list of objects, and you may want to apply some other node to all those objects. The number of objects may be unknown ahead of time (meaning the number of edges may not be known) and the input `State` to the downstream `Node` should be different (one for each generated object).

To support this design pattern, LangGraph supports returning @[`Send`][Send] objects from conditional edges. `Send` takes two arguments: first is the name of the node, and second is the state to pass to that node.

```typescript
import { Send } from "@langchain/langgraph";

graph.addConditionalEdges("nodeA", (state) => {
  return state.subjects.map((subject) => new Send("generateJoke", { subject }));
});
```

:::

## `Command`

:::python
It can be useful to combine control flow (edges) and state updates (nodes). For example, you might want to BOTH perform state updates AND decide which node to go to next in the SAME node. LangGraph provides a way to do so by returning a @[`Command`][Command] object from node functions:

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        # state update
        update={"foo": "bar"},
        # control flow
        goto="my_other_node"
    )
```

With `Command` you can also achieve dynamic control flow behavior (identical to [conditional edges](#conditional-edges)):

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    if state["foo"] == "bar":
        return Command(update={"foo": "baz"}, goto="my_other_node")
```

:::

:::js
It can be useful to combine control flow (edges) and state updates (nodes). For example, you might want to BOTH perform state updates AND decide which node to go to next in the SAME node. LangGraph provides a way to do so by returning a `Command` object from node functions:

```typescript
import { Command } from "@langchain/langgraph";

graph.addNode("myNode", (state) => {
  return new Command({
    update: { foo: "bar" },
    goto: "myOtherNode",
  });
});
```

With `Command` you can also achieve dynamic control flow behavior (identical to [conditional edges](#conditional-edges)):

```typescript
import { Command } from "@langchain/langgraph";

graph.addNode("myNode", (state) => {
  if (state.foo === "bar") {
    return new Command({
      update: { foo: "baz" },
      goto: "myOtherNode",
    });
  }
});
```

When using `Command` in your node functions, you must add the `ends` parameter when adding the node to specify which nodes it can route to:

```typescript
builder.addNode("myNode", myNode, {
  ends: ["myOtherNode", END],
});
```

:::

!!! important

    When returning `Command` in your node functions, you must add return type annotations with the list of node names the node is routing to, e.g. `Command[Literal["my_other_node"]]`. This is necessary for the graph rendering and tells LangGraph that `my_node` can navigate to `my_other_node`.

Check out this [how-to guide](../how-tos/graph-api.md#combine-control-flow-and-state-updates-with-command) for an end-to-end example of how to use `Command`.

### When should I use Command instead of conditional edges?

- Use `Command` when you need to **both** update the graph state **and** route to a different node. For example, when implementing [multi-agent handoffs](./multi_agent.md#handoffs) where it's important to route to a different agent and pass some information to that agent.
- Use [conditional edges](#conditional-edges) to route between nodes conditionally without updating the state.

### Navigating to a node in a parent graph

:::python
If you are using [subgraphs](./subgraphs.md), you might want to navigate from a node within a subgraph to a different subgraph (i.e. a different node in the parent graph). To do so, you can specify `graph=Command.PARENT` in `Command`:

```python
def my_node(state: State) -> Command[Literal["other_subgraph"]]:
    return Command(
        update={"foo": "bar"},
        goto="other_subgraph",  # where `other_subgraph` is a node in the parent graph
        graph=Command.PARENT
    )
```

!!! note

    Setting `graph` to `Command.PARENT` will navigate to the closest parent graph.

!!! important "State updates with `Command.PARENT`"

    When you send updates from a subgraph node to a parent graph node for a key that's shared by both parent and subgraph [state schemas](#schema), you **must** define a [reducer](#reducers) for the key you're updating in the parent graph state. See this [example](../how-tos/graph-api.md#navigate-to-a-node-in-a-parent-graph).

:::

:::js
If you are using [subgraphs](./subgraphs.md), you might want to navigate from a node within a subgraph to a different subgraph (i.e. a different node in the parent graph). To do so, you can specify `graph: Command.PARENT` in `Command`:

```typescript
import { Command } from "@langchain/langgraph";

graph.addNode("myNode", (state) => {
  return new Command({
    update: { foo: "bar" },
    goto: "otherSubgraph", // where `otherSubgraph` is a node in the parent graph
    graph: Command.PARENT,
  });
});
```

!!! note

    Setting `graph` to `Command.PARENT` will navigate to the closest parent graph.

!!! important "State updates with `Command.PARENT`"

    When you send updates from a subgraph node to a parent graph node for a key that's shared by both parent and subgraph [state schemas](#schema), you **must** define a [reducer](#reducers) for the key you're updating in the parent graph state.

:::

:::js
If you are using [subgraphs](./subgraphs.md), you might want to navigate from a node within a subgraph to a different subgraph (i.e. a different node in the parent graph). To do so, you can specify `graph: Command.PARENT` in `Command`:

```typescript
import { Command } from "@langchain/langgraph";

graph.addNode("myNode", (state) => {
  return new Command({
    update: { foo: "bar" },
    goto: "otherSubgraph", // where `otherSubgraph` is a node in the parent graph
    graph: Command.PARENT,
  });
});
```

!!! note

    Setting `graph` to `Command.PARENT` will navigate to the closest parent graph.

!!! important "State updates with `Command.PARENT`"

    When you send updates from a subgraph node to a parent graph node for a key that's shared by both parent and subgraph [state schemas](#schema), you **must** define a [reducer](#reducers) for the key you're updating in the parent graph state.

:::

This is particularly useful when implementing [multi-agent handoffs](./multi_agent.md#handoffs).

Check out [this guide](../how-tos/graph-api.md#navigate-to-a-node-in-a-parent-graph) for detail.

### Using inside tools

A common use case is updating graph state from inside a tool. For example, in a customer support application you might want to look up customer information based on their account number or ID in the beginning of the conversation.

Refer to [this guide](../how-tos/graph-api.md#use-inside-tools) for detail.

### Human-in-the-loop

:::python
`Command` is an important part of human-in-the-loop workflows: when using `interrupt()` to collect user input, `Command` is then used to supply the input and resume execution via `Command(resume="User input")`. Check out [this conceptual guide](./human_in_the_loop.md) for more information.
:::

:::js
`Command` is an important part of human-in-the-loop workflows: when using `interrupt()` to collect user input, `Command` is then used to supply the input and resume execution via `new Command({ resume: "User input" })`. Check out the [human-in-the-loop conceptual guide](./human_in_the_loop.md) for more information.
:::

## Graph Migrations

LangGraph can easily handle migrations of graph definitions (nodes, edges, and state) even when using a checkpointer to track state.

- For threads at the end of the graph (i.e. not interrupted) you can change the entire topology of the graph (i.e. all nodes and edges, remove, add, rename, etc)
- For threads currently interrupted, we support all topology changes other than renaming / removing nodes (as that thread could now be about to enter a node that no longer exists) -- if this is a blocker please reach out and we can prioritize a solution.
- For modifying state, we have full backwards and forwards compatibility for adding and removing keys
- State keys that are renamed lose their saved state in existing threads
- State keys whose types change in incompatible ways could currently cause issues in threads with state from before the change -- if this is a blocker please reach out and we can prioritize a solution.

:::python

## Runtime Context

When creating a graph, you can specify a `context_schema` for runtime context passed to nodes. This is useful for passing
information to nodes that is not part of the graph state. For example, you might want to pass dependencies such as model name or a database connection.

```python
@dataclass
class ContextSchema:
    llm_provider: str = "openai"

graph = StateGraph(State, context_schema=ContextSchema)
```

:::

:::js

When creating a graph, you can also mark that certain parts of the graph are configurable. This is commonly done to enable easily switching between models or system prompts. This allows you to create a single "cognitive architecture" (the graph) but have multiple different instance of it.

You can optionally specify a config schema when creating a graph.

```typescript
import { z } from "zod";

const ConfigSchema = z.object({
  llm: z.string(),
});

const graph = new StateGraph(State, ConfigSchema);
```

:::

:::python
You can then pass this context into the graph using the `context` parameter of the `invoke` method.

```python
graph.invoke(inputs, context={"llm_provider": "anthropic"})
```

:::

:::js
You can then pass this configuration into the graph using the `configurable` config field.

```typescript
const config = { configurable: { llm: "anthropic" } };

await graph.invoke(inputs, config);
```

:::

You can then access and use this context inside a node or conditional edge:

```python
from langgraph.runtime import Runtime

def node_a(state: State, runtime: Runtime[ContextSchema]):
    llm = get_llm(runtime.context.llm_provider)
    ...
```

See [this guide](../how-tos/graph-api.md#add-runtime-configuration) for a full breakdown on configuration.
:::

:::js

```typescript
graph.addNode("myNode", (state, config) => {
  const llmType = config?.configurable?.llm || "openai";
  const llm = getLlm(llmType);
  return { results: `Hello, ${state.input}!` };
});
```

:::

### Recursion Limit

:::python
The recursion limit sets the maximum number of [super-steps](#graphs) the graph can execute during a single execution. Once the limit is reached, LangGraph will raise `GraphRecursionError`. By default this value is set to 25 steps. The recursion limit can be set on any graph at runtime, and is passed to `.invoke`/`.stream` via the config dictionary. Importantly, `recursion_limit` is a standalone `config` key and should not be passed inside the `configurable` key as all other user-defined configuration. See the example below:

```python
graph.invoke(inputs, config={"recursion_limit": 5}, context={"llm": "anthropic"})
```

Read [this how-to](https://langchain-ai.github.io/langgraph/how-tos/recursion-limit/) to learn more about how the recursion limit works.
:::

:::js
The recursion limit sets the maximum number of [super-steps](#graphs) the graph can execute during a single execution. Once the limit is reached, LangGraph will raise `GraphRecursionError`. By default this value is set to 25 steps. The recursion limit can be set on any graph at runtime, and is passed to `.invoke`/`.stream` via the config object. Importantly, `recursionLimit` is a standalone `config` key and should not be passed inside the `configurable` key as all other user-defined configuration. See the example below:

```typescript
await graph.invoke(inputs, {
  recursionLimit: 5,
  configurable: { llm: "anthropic" },
});
```

:::

## Visualization

It's often nice to be able to visualize graphs, especially as they get more complex. LangGraph comes with several built-in ways to visualize graphs. See [this how-to guide](../how-tos/graph-api.md#visualize-your-graph) for more info.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/mcp.md
```md
# MCP

[Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction) is an open protocol that standardizes how applications provide tools and context to language models. LangGraph agents can use tools defined on MCP servers through the `langchain-mcp-adapters` library.

![MCP](../agents/assets/mcp.png)

Install the `langchain-mcp-adapters` library to use MCP tools in LangGraph:

:::python
```bash
pip install langchain-mcp-adapters
```
:::

:::js
```bash
npm install @langchain/mcp-adapters
```
:::
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/memory.md
```md
---
search:
  boost: 2
---

# Memory

[Memory](../how-tos/memory/add-memory.md) is a system that remembers information about previous interactions. For AI agents, memory is crucial because it lets them remember previous interactions, learn from feedback, and adapt to user preferences. As agents tackle more complex tasks with numerous user interactions, this capability becomes essential for both efficiency and user satisfaction.

This conceptual guide covers two types of memory, based on their recall scope:

- [Short-term memory](#short-term-memory), or [thread](persistence.md#threads)-scoped memory, tracks the ongoing conversation by maintaining message history within a session. LangGraph manages short-term memory as a part of your agent's [state](low_level.md#state). State is persisted to a database using a [checkpointer](persistence.md#checkpoints) so the thread can be resumed at any time. Short-term memory updates when the graph is invoked or a step is completed, and the State is read at the start of each step.

- [Long-term memory](#long-term-memory) stores user-specific or application-level data across sessions and is shared _across_ conversational threads. It can be recalled _at any time_ and _in any thread_. Memories are scoped to any custom namespace, not just within a single thread ID. LangGraph provides [stores](persistence.md#memory-store) ([reference doc](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.BaseStore)) to let you save and recall long-term memories.

![](img/memory/short-vs-long.png)


## Short-term memory

[Short-term memory](../how-tos/memory/add-memory.md#add-short-term-memory) lets your application remember previous interactions within a single [thread](persistence.md#threads) or conversation. A [thread](persistence.md#threads) organizes multiple interactions in a session, similar to the way email groups messages in a single conversation.

LangGraph manages short-term memory as part of the agent's state, persisted via thread-scoped checkpoints. This state can normally include the conversation history along with other stateful data, such as uploaded files, retrieved documents, or generated artifacts. By storing these in the graph's state, the bot can access the full context for a given conversation while maintaining separation between different threads.

### Manage short-term memory

Conversation history is the most common form of short-term memory, and long conversations pose a challenge to today's LLMs. A full history may not fit inside an LLM's context window, resulting in an irrecoverable error. Even if your LLM supports the full context length, most LLMs still perform poorly over long contexts. They get "distracted" by stale or off-topic content, all while suffering from slower response times and higher costs.

Chat models accept context using messages, which include developer provided instructions (a system message) and user inputs (human messages). In chat applications, messages alternate between human inputs and model responses, resulting in a list of messages that grows longer over time. Because context windows are limited and token-rich message lists can be costly, many applications can benefit from using techniques to manually remove or forget stale information.

![](img/memory/filter.png)

For more information on common techniques for managing messages, see the [Add and manage memory](../how-tos/memory/add-memory.md#manage-short-term-memory) guide.

## Long-term memory

[Long-term memory](../how-tos/memory/add-memory.md#add-long-term-memory) in LangGraph allows systems to retain information across different conversations or sessions. Unlike short-term memory, which is **thread-scoped**, long-term memory is saved within custom "namespaces."

Long-term memory is a complex challenge without a one-size-fits-all solution. However, the following questions provide a framework to help you navigate the different techniques:

- [What is the type of memory?](#memory-types) Humans use memories to remember facts ([semantic memory](#semantic-memory)), experiences ([episodic memory](#episodic-memory)), and rules ([procedural memory](#procedural-memory)). AI agents can use memory in the same ways. For example, AI agents can use memory to remember specific facts about a user to accomplish a task.

- [When do you want to update memories?](#writing-memories) Memory can be updated as part of an agent's application logic (e.g., "on the hot path"). In this case, the agent typically decides to remember facts before responding to a user. Alternatively, memory can be updated as a background task (logic that runs in the background / asynchronously and generates memories). We explain the tradeoffs between these approaches in the [section below](#writing-memories).

### Memory types

Different applications require various types of memory. Although the analogy isn't perfect, examining [human memory types](https://www.psychologytoday.com/us/basics/memory/types-of-memory?ref=blog.langchain.dev) can be insightful. Some research (e.g., the [CoALA paper](https://arxiv.org/pdf/2309.02427)) have even mapped these human memory types to those used in AI agents.

| Memory Type | What is Stored | Human Example | Agent Example |
|-------------|----------------|---------------|---------------|
| [Semantic](#semantic-memory) | Facts | Things I learned in school | Facts about a user |
| [Episodic](#episodic-memory) | Experiences | Things I did | Past agent actions |
| [Procedural](#procedural-memory) | Instructions | Instincts or motor skills | Agent system prompt |

#### Semantic memory

[Semantic memory](https://en.wikipedia.org/wiki/Semantic_memory), both in humans and AI agents, involves the retention of specific facts and concepts. In humans, it can include information learned in school and the understanding of concepts and their relationships. For AI agents, semantic memory is often used to personalize applications by remembering facts or concepts from past interactions. 

!!! note

    Semantic memory is different from "semantic search," which is a technique for finding similar content using "meaning" (usually as embeddings). Semantic memory is a term from psychology, referring to storing facts and knowledge, while semantic search is a method for retrieving information based on meaning rather than exact matches.


##### Profile

Semantic memories can be managed in different ways. For example, memories can be a single, continuously updated "profile" of well-scoped and specific information about a user, organization, or other entity (including the agent itself). A profile is generally just a JSON document with various key-value pairs you've selected to represent your domain. 

When remembering a profile, you will want to make sure that you are **updating** the profile each time. As a result, you will want to pass in the previous profile and [ask the model to generate a new profile](https://github.com/langchain-ai/memory-template) (or some [JSON patch](https://github.com/hinthornw/trustcall) to apply to the old profile). This can be become error-prone as the profile gets larger, and may benefit from splitting a profile into multiple documents or **strict** decoding when generating documents to ensure the memory schemas remains valid.

![](img/memory/update-profile.png)

##### Collection

Alternatively, memories can be a collection of documents that are continuously updated and extended over time. Each individual memory can be more narrowly scoped and easier to generate, which means that you're less likely to **lose** information over time. It's easier for an LLM to generate _new_ objects for new information than reconcile new information with an existing profile. As a result, a document collection tends to lead to [higher recall downstream](https://en.wikipedia.org/wiki/Precision_and_recall).

However, this shifts some complexity memory updating. The model must now _delete_ or _update_ existing items in the list, which can be tricky. In addition, some models may default to over-inserting and others may default to over-updating. See the [Trustcall](https://github.com/hinthornw/trustcall) package for one way to manage this and consider evaluation (e.g., with a tool like [LangSmith](https://docs.smith.langchain.com/tutorials/Developers/evaluation)) to help you tune the behavior.

Working with document collections also shifts complexity to memory **search** over the list. The `Store` currently supports both [semantic search](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.SearchOp.query) and [filtering by content](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.SearchOp.filter).

Finally, using a collection of memories can make it challenging to provide comprehensive context to the model. While individual memories may follow a specific schema, this structure might not capture the full context or relationships between memories. As a result, when using these memories to generate responses, the model may lack important contextual information that would be more readily available in a unified profile approach.

![](img/memory/update-list.png)

Regardless of memory management approach, the central point is that the agent will use the semantic memories to [ground its responses](https://python.langchain.com/docs/concepts/rag/), which often leads to more personalized and relevant interactions.

#### Episodic memory

[Episodic memory](https://en.wikipedia.org/wiki/Episodic_memory), in both humans and AI agents, involves recalling past events or actions. The [CoALA paper](https://arxiv.org/pdf/2309.02427) frames this well: facts can be written to semantic memory, whereas *experiences* can be written to episodic memory. For AI agents, episodic memory is often used to help an agent remember how to accomplish a task. 

:::python
In practice, episodic memories are often implemented through [few-shot example prompting](https://python.langchain.com/docs/concepts/few_shot_prompting/), where agents learn from past sequences to perform tasks correctly. Sometimes it's easier to "show" than "tell" and LLMs learn well from examples. Few-shot learning lets you ["program"](https://x.com/karpathy/status/1627366413840322562) your LLM by updating the prompt with input-output examples to illustrate the intended behavior. While various [best-practices](https://python.langchain.com/docs/concepts/#1-generating-examples) can be used to generate few-shot examples, often the challenge lies in selecting the most relevant examples based on user input.
:::

:::js
In practice, episodic memories are often implemented through few-shot example prompting, where agents learn from past sequences to perform tasks correctly. Sometimes it's easier to "show" than "tell" and LLMs learn well from examples. Few-shot learning lets you ["program"](https://x.com/karpathy/status/1627366413840322562) your LLM by updating the prompt with input-output examples to illustrate the intended behavior. While various best-practices can be used to generate few-shot examples, often the challenge lies in selecting the most relevant examples based on user input.
:::

:::python
Note that the memory [store](persistence.md#memory-store) is just one way to store data as few-shot examples. If you want to have more developer involvement, or tie few-shots more closely to your evaluation harness, you can also use a [LangSmith Dataset](https://docs.smith.langchain.com/evaluation/how_to_guides/datasets/index_datasets_for_dynamic_few_shot_example_selection) to store your data. Then dynamic few-shot example selectors can be used out-of-the box to achieve this same goal. LangSmith will index the dataset for you and enable retrieval of few shot examples that are most relevant to the user input based upon keyword similarity ([using a BM25-like algorithm](https://docs.smith.langchain.com/how_to_guides/datasets/index_datasets_for_dynamic_few_shot_example_selection) for keyword based similarity). 

See this how-to [video](https://www.youtube.com/watch?v=37VaU7e7t5o) for example usage of dynamic few-shot example selection in LangSmith. Also, see this [blog post](https://blog.langchain.dev/few-shot-prompting-to-improve-tool-calling-performance/) showcasing few-shot prompting to improve tool calling performance and this [blog post](https://blog.langchain.dev/aligning-llm-as-a-judge-with-human-preferences/) using few-shot example to align an LLMs to human preferences.
:::

:::js
Note that the memory [store](persistence.md#memory-store) is just one way to store data as few-shot examples. If you want to have more developer involvement, or tie few-shots more closely to your evaluation harness, you can also use a LangSmith Dataset to store your data. Then dynamic few-shot example selectors can be used out-of-the box to achieve this same goal. LangSmith will index the dataset for you and enable retrieval of few shot examples that are most relevant to the user input based upon keyword similarity.

See this how-to [video](https://www.youtube.com/watch?v=37VaU7e7t5o) for example usage of dynamic few-shot example selection in LangSmith. Also, see this [blog post](https://blog.langchain.dev/few-shot-prompting-to-improve-tool-calling-performance/) showcasing few-shot prompting to improve tool calling performance and this [blog post](https://blog.langchain.dev/aligning-llm-as-a-judge-with-human-preferences/) using few-shot example to align an LLMs to human preferences.
:::

#### Procedural memory

[Procedural memory](https://en.wikipedia.org/wiki/Procedural_memory), in both humans and AI agents, involves remembering the rules used to perform tasks. In humans, procedural memory is like the internalized knowledge of how to perform tasks, such as riding a bike via basic motor skills and balance. Episodic memory, on the other hand, involves recalling specific experiences, such as the first time you successfully rode a bike without training wheels or a memorable bike ride through a scenic route. For AI agents, procedural memory is a combination of model weights, agent code, and agent's prompt that collectively determine the agent's functionality. 

In practice, it is fairly uncommon for agents to modify their model weights or rewrite their code. However, it is more common for agents to modify their own prompts. 

One effective approach to refining an agent's instructions is through ["Reflection"](https://blog.langchain.dev/reflection-agents/) or meta-prompting. This involves prompting the agent with its current instructions (e.g., the system prompt) along with recent conversations or explicit user feedback. The agent then refines its own instructions based on this input. This method is particularly useful for tasks where instructions are challenging to specify upfront, as it allows the agent to learn and adapt from its interactions.

For example, we built a [Tweet generator](https://www.youtube.com/watch?v=Vn8A3BxfplE) using external feedback and prompt re-writing to produce high-quality paper summaries for Twitter. In this case, the specific summarization prompt was difficult to specify *a priori*, but it was fairly easy for a user to critique the generated Tweets and provide feedback on how to improve the summarization process. 

The below pseudo-code shows how you might implement this with the LangGraph memory [store](persistence.md#memory-store), using the store to save a prompt, the `update_instructions` node to get the current prompt (as well as feedback from the conversation with the user captured in `state["messages"]`), update the prompt, and save the new prompt back to the store. Then, the `call_model` get the updated prompt from the store and uses it to generate a response.

:::python
```python
# Node that *uses* the instructions
def call_model(state: State, store: BaseStore):
    namespace = ("agent_instructions", )
    instructions = store.get(namespace, key="agent_a")[0]
    # Application logic
    prompt = prompt_template.format(instructions=instructions.value["instructions"])
    ...

# Node that updates instructions
def update_instructions(state: State, store: BaseStore):
    namespace = ("instructions",)
    current_instructions = store.search(namespace)[0]
    # Memory logic
    prompt = prompt_template.format(instructions=current_instructions.value["instructions"], conversation=state["messages"])
    output = llm.invoke(prompt)
    new_instructions = output['new_instructions']
    store.put(("agent_instructions",), "agent_a", {"instructions": new_instructions})
    ...
```
:::

:::js
```typescript
// Node that *uses* the instructions
const callModel = async (state: State, store: BaseStore) => {
    const namespace = ["agent_instructions"];
    const instructions = await store.get(namespace, "agent_a");
    // Application logic
    const prompt = promptTemplate.format({ 
        instructions: instructions[0].value.instructions 
    });
    // ...
};

// Node that updates instructions
const updateInstructions = async (state: State, store: BaseStore) => {
    const namespace = ["instructions"];
    const currentInstructions = await store.search(namespace);
    // Memory logic
    const prompt = promptTemplate.format({ 
        instructions: currentInstructions[0].value.instructions, 
        conversation: state.messages 
    });
    const output = await llm.invoke(prompt);
    const newInstructions = output.new_instructions;
    await store.put(["agent_instructions"], "agent_a", { 
        instructions: newInstructions 
    });
    // ...
};
```
:::

![](img/memory/update-instructions.png)

### Writing memories

There are two primary methods for agents to write memories: ["in the hot path"](#in-the-hot-path) and ["in the background"](#in-the-background).

![](img/memory/hot_path_vs_background.png)

#### In the hot path

Creating memories during runtime offers both advantages and challenges. On the positive side, this approach allows for real-time updates, making new memories immediately available for use in subsequent interactions. It also enables transparency, as users can be notified when memories are created and stored.

However, this method also presents challenges. It may increase complexity if the agent requires a new tool to decide what to commit to memory. In addition, the process of reasoning about what to save to memory can impact agent latency. Finally, the agent must multitask between memory creation and its other responsibilities, potentially affecting the quantity and quality of memories created.

As an example, ChatGPT uses a [save_memories](https://openai.com/index/memory-and-new-controls-for-chatgpt/) tool to upsert memories as content strings, deciding whether and how to use this tool with each user message. See our [memory-agent](https://github.com/langchain-ai/memory-agent) template as an reference implementation.

#### In the background

Creating memories as a separate background task offers several advantages. It eliminates latency in the primary application, separates application logic from memory management, and allows for more focused task completion by the agent. This approach also provides flexibility in timing memory creation to avoid redundant work.

However, this method has its own challenges. Determining the frequency of memory writing becomes crucial, as infrequent updates may leave other threads without new context. Deciding when to trigger memory formation is also important. Common strategies include scheduling after a set time period (with rescheduling if new events occur), using a cron schedule, or allowing manual triggers by users or the application logic.

See our [memory-service](https://github.com/langchain-ai/memory-template) template as an reference implementation.

### Memory storage

LangGraph stores long-term memories as JSON documents in a [store](persistence.md#memory-store). Each memory is organized under a custom `namespace` (similar to a folder) and a distinct `key` (like a file name). Namespaces often include user or org IDs or other labels that makes it easier to organize information. This structure enables hierarchical organization of memories. Cross-namespace searching is then supported through content filters.

:::python
```python
from langgraph.store.memory import InMemoryStore


def embed(texts: list[str]) -> list[list[float]]:
    # Replace with an actual embedding function or LangChain embeddings object
    return [[1.0, 2.0] * len(texts)]


# InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production use.
store = InMemoryStore(index={"embed": embed, "dims": 2})
user_id = "my-user"
application_context = "chitchat"
namespace = (user_id, application_context)
store.put(
    namespace,
    "a-memory",
    {
        "rules": [
            "User likes short, direct language",
            "User only speaks English & python",
        ],
        "my-key": "my-value",
    },
)
# get the "memory" by ID
item = store.get(namespace, "a-memory")
# search for "memories" within this namespace, filtering on content equivalence, sorted by vector similarity
items = store.search(
    namespace, filter={"my-key": "my-value"}, query="language preferences"
)
```
:::

:::js
```typescript
import { InMemoryStore } from "@langchain/langgraph";

const embed = (texts: string[]): number[][] => {
    // Replace with an actual embedding function or LangChain embeddings object
    return texts.map(() => [1.0, 2.0]);
};

// InMemoryStore saves data to an in-memory dictionary. Use a DB-backed store in production use.
const store = new InMemoryStore({ index: { embed, dims: 2 } });
const userId = "my-user";
const applicationContext = "chitchat";
const namespace = [userId, applicationContext];

await store.put(
    namespace,
    "a-memory",
    {
        rules: [
            "User likes short, direct language",
            "User only speaks English & TypeScript",
        ],
        "my-key": "my-value",
    }
);

// get the "memory" by ID
const item = await store.get(namespace, "a-memory");

// search for "memories" within this namespace, filtering on content equivalence, sorted by vector similarity
const items = await store.search(
    namespace, 
    { 
        filter: { "my-key": "my-value" }, 
        query: "language preferences" 
    }
);
```
:::

For more information about the memory store, see the [Persistence](persistence.md#memory-store) guide.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/multi_agent.md
```md
# Multi-agent systems

An [agent](./agentic_concepts.md#agent-architectures) is _a system that uses an LLM to decide the control flow of an application_. As you develop these systems, they might grow more complex over time, making them harder to manage and scale. For example, you might run into the following problems:

- agent has too many tools at its disposal and makes poor decisions about which tool to call next
- context grows too complex for a single agent to keep track of
- there is a need for multiple specialization areas in the system (e.g. planner, researcher, math expert, etc.)

To tackle these, you might consider breaking your application into multiple smaller, independent agents and composing them into a **multi-agent system**. These independent agents can be as simple as a prompt and an LLM call, or as complex as a [ReAct](./agentic_concepts.md#tool-calling-agent) agent (and more!).

The primary benefits of using multi-agent systems are:

- **Modularity**: Separate agents make it easier to develop, test, and maintain agentic systems.
- **Specialization**: You can create expert agents focused on specific domains, which helps with the overall system performance.
- **Control**: You can explicitly control how agents communicate (as opposed to relying on function calling).

## Multi-agent architectures

![](./img/multi_agent/architectures.png)

There are several ways to connect agents in a multi-agent system:

- **Network**: each agent can communicate with [every other agent](../tutorials/multi_agent/multi-agent-collaboration.ipynb/). Any agent can decide which other agent to call next.
- **Supervisor**: each agent communicates with a single [supervisor](../tutorials/multi_agent/agent_supervisor.md/) agent. Supervisor agent makes decisions on which agent should be called next.
- **Supervisor (tool-calling)**: this is a special case of supervisor architecture. Individual agents can be represented as tools. In this case, a supervisor agent uses a tool-calling LLM to decide which of the agent tools to call, as well as the arguments to pass to those agents.
- **Hierarchical**: you can define a multi-agent system with [a supervisor of supervisors](../tutorials/multi_agent/hierarchical_agent_teams.ipynb/). This is a generalization of the supervisor architecture and allows for more complex control flows.
- **Custom multi-agent workflow**: each agent communicates with only a subset of agents. Parts of the flow are deterministic, and only some agents can decide which other agents to call next.

### Handoffs

In multi-agent architectures, agents can be represented as graph nodes. Each agent node executes its step(s) and decides whether to finish execution or route to another agent, including potentially routing to itself (e.g., running in a loop). A common pattern in multi-agent interactions is **handoffs**, where one agent _hands off_ control to another. Handoffs allow you to specify:

- **destination**: target agent to navigate to (e.g., name of the node to go to)
- **payload**: [information to pass to that agent](#communication-and-state-management) (e.g., state update)

To implement handoffs in LangGraph, agent nodes can return [`Command`](./low_level.md#command) object that allows you to combine both control flow and state updates:

:::python

```python
def agent(state) -> Command[Literal["agent", "another_agent"]]:
    # the condition for routing/halting can be anything, e.g. LLM tool call / structured output, etc.
    goto = get_next_agent(...)  # 'agent' / 'another_agent'
    return Command(
        # Specify which agent to call next
        goto=goto,
        # Update the graph state
        update={"my_state_key": "my_state_value"}
    )
```

:::

:::js

```typescript
graph.addNode((state) => {
    // the condition for routing/halting can be anything, e.g. LLM tool call / structured output, etc.
    const goto = getNextAgent(...); // 'agent' / 'another_agent'
    return new Command({
      // Specify which agent to call next
      goto,
      // Update the graph state
      update: { myStateKey: "myStateValue" }
    });
})
```

:::

:::python
In a more complex scenario where each agent node is itself a graph (i.e., a [subgraph](./subgraphs.md)), a node in one of the agent subgraphs might want to navigate to a different agent. For example, if you have two agents, `alice` and `bob` (subgraph nodes in a parent graph), and `alice` needs to navigate to `bob`, you can set `graph=Command.PARENT` in the `Command` object:

```python
def some_node_inside_alice(state):
    return Command(
        goto="bob",
        update={"my_state_key": "my_state_value"},
        # specify which graph to navigate to (defaults to the current graph)
        graph=Command.PARENT,
    )
```

:::

:::js
In a more complex scenario where each agent node is itself a graph (i.e., a [subgraph](./subgraphs.md)), a node in one of the agent subgraphs might want to navigate to a different agent. For example, if you have two agents, `alice` and `bob` (subgraph nodes in a parent graph), and `alice` needs to navigate to `bob`, you can set `graph: Command.PARNT` in the `Command` object:

```typescript
alice.addNode((state) => {
  return new Command({
    goto: "bob",
    update: { myStateKey: "myStateValue" },
    // specify which graph to navigate to (defaults to the current graph)
    graph: Command.PARENT,
  });
});
```

:::

!!! note

    :::python

    If you need to support visualization for subgraphs communicating using `Command(graph=Command.PARENT)` you would need to wrap them in a node function with `Command` annotation:
    Instead of this:

    ```python
    builder.add_node(alice)
    ```

    you would need to do this:

    ```python
    def call_alice(state) -> Command[Literal["bob"]]:
        return alice.invoke(state)

    builder.add_node("alice", call_alice)
    ```

    :::

    :::js
    If you need to support visualization for subgraphs communicating using/ `Command({ graph: Command.PARENT })` you would need to wrap them in a node function with `Command` annotation:

    Instead of this:

    ```typescript
    builder.addNode("alice", alice);
    ```

    you would need to do this:

    ```typescript
    builder.addNode("alice", (state) => alice.invoke(state), { ends: ["bob"] });
    ```

    :::

#### Handoffs as tools

One of the most common agent types is a [tool-calling agent](../agents/overview.md). For those types of agents, a common pattern is wrapping a handoff in a tool call:

:::python

```python
from langchain_core.tools import tool

@tool
def transfer_to_bob():
    """Transfer to bob."""
    return Command(
        # name of the agent (node) to go to
        goto="bob",
        # data to send to the agent
        update={"my_state_key": "my_state_value"},
        # indicate to LangGraph that we need to navigate to
        # agent node in a parent graph
        graph=Command.PARENT,
    )
```

:::

:::js

```typescript
import { tool } from "@langchain/core/tools";
import { Command } from "@langchain/langgraph";
import { z } from "zod";

const transferToBob = tool(
  async () => {
    return new Command({
      // name of the agent (node) to go to
      goto: "bob",
      // data to send to the agent
      update: { myStateKey: "myStateValue" },
      // indicate to LangGraph that we need to navigate to
      // agent node in a parent graph
      graph: Command.PARENT,
    });
  },
  {
    name: "transfer_to_bob",
    description: "Transfer to bob.",
    schema: z.object({}),
  }
);
```

:::

This is a special case of updating the graph state from tools where, in addition to the state update, the control flow is included as well.

!!! important

      :::python
      If you want to use tools that return `Command`, you can use the prebuilt @[`create_react_agent`][create_react_agent] / @[`ToolNode`][ToolNode] components, or else implement your own logic:

      ```python
      def call_tools(state):
          ...
          commands = [tools_by_name[tool_call["name"]].invoke(tool_call) for tool_call in tool_calls]
          return commands
      ```
      :::

      :::js
      If you want to use tools that return `Command`, you can use the prebuilt @[`createReactAgent`][create_react_agent] / @[ToolNode] components, or else implement your own logic:

      ```typescript
      graph.addNode("call_tools", async (state) => {
        // ... tool execution logic
        const commands = toolCalls.map((toolCall) =>
          toolsByName[toolCall.name].invoke(toolCall)
        );
        return commands;
      });
      ```
      :::

Let's now take a closer look at the different multi-agent architectures.

### Network

In this architecture, agents are defined as graph nodes. Each agent can communicate with every other agent (many-to-many connections) and can decide which agent to call next. This architecture is good for problems that do not have a clear hierarchy of agents or a specific sequence in which agents should be called.

:::python

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.types import Command
from langgraph.graph import StateGraph, MessagesState, START, END

model = ChatOpenAI()

def agent_1(state: MessagesState) -> Command[Literal["agent_2", "agent_3", END]]:
    # you can pass relevant parts of the state to the LLM (e.g., state["messages"])
    # to determine which agent to call next. a common pattern is to call the model
    # with a structured output (e.g. force it to return an output with a "next_agent" field)
    response = model.invoke(...)
    # route to one of the agents or exit based on the LLM's decision
    # if the LLM returns "__end__", the graph will finish execution
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

def agent_2(state: MessagesState) -> Command[Literal["agent_1", "agent_3", END]]:
    response = model.invoke(...)
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

def agent_3(state: MessagesState) -> Command[Literal["agent_1", "agent_2", END]]:
    ...
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

builder = StateGraph(MessagesState)
builder.add_node(agent_1)
builder.add_node(agent_2)
builder.add_node(agent_3)

builder.add_edge(START, "agent_1")
network = builder.compile()
```

:::

:::js

```typescript
import { StateGraph, MessagesZodState, START, END } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { Command } from "@langchain/langgraph";
import { z } from "zod";

const model = new ChatOpenAI();

const agent1 = async (state: z.infer<typeof MessagesZodState>) => {
  // you can pass relevant parts of the state to the LLM (e.g., state.messages)
  // to determine which agent to call next. a common pattern is to call the model
  // with a structured output (e.g. force it to return an output with a "next_agent" field)
  const response = await model.invoke(...);
  // route to one of the agents or exit based on the LLM's decision
  // if the LLM returns "__end__", the graph will finish execution
  return new Command({
    goto: response.nextAgent,
    update: { messages: [response.content] },
  });
};

const agent2 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return new Command({
    goto: response.nextAgent,
    update: { messages: [response.content] },
  });
};

const agent3 = async (state: z.infer<typeof MessagesZodState>) => {
  // ...
  return new Command({
    goto: response.nextAgent,
    update: { messages: [response.content] },
  });
};

const builder = new StateGraph(MessagesZodState)
  .addNode("agent1", agent1, {
    ends: ["agent2", "agent3", END]
  })
  .addNode("agent2", agent2, {
    ends: ["agent1", "agent3", END]
  })
  .addNode("agent3", agent3, {
    ends: ["agent1", "agent2", END]
  })
  .addEdge(START, "agent1");

const network = builder.compile();
```

:::

### Supervisor

In this architecture, we define agents as nodes and add a supervisor node (LLM) that decides which agent nodes should be called next. We use [`Command`](./low_level.md#command) to route execution to the appropriate agent node based on supervisor's decision. This architecture also lends itself well to running multiple agents in parallel or using [map-reduce](../how-tos/graph-api.md#map-reduce-and-the-send-api) pattern.

:::python

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.types import Command
from langgraph.graph import StateGraph, MessagesState, START, END

model = ChatOpenAI()

def supervisor(state: MessagesState) -> Command[Literal["agent_1", "agent_2", END]]:
    # you can pass relevant parts of the state to the LLM (e.g., state["messages"])
    # to determine which agent to call next. a common pattern is to call the model
    # with a structured output (e.g. force it to return an output with a "next_agent" field)
    response = model.invoke(...)
    # route to one of the agents or exit based on the supervisor's decision
    # if the supervisor returns "__end__", the graph will finish execution
    return Command(goto=response["next_agent"])

def agent_1(state: MessagesState) -> Command[Literal["supervisor"]]:
    # you can pass relevant parts of the state to the LLM (e.g., state["messages"])
    # and add any additional logic (different models, custom prompts, structured output, etc.)
    response = model.invoke(...)
    return Command(
        goto="supervisor",
        update={"messages": [response]},
    )

def agent_2(state: MessagesState) -> Command[Literal["supervisor"]]:
    response = model.invoke(...)
    return Command(
        goto="supervisor",
        update={"messages": [response]},
    )

builder = StateGraph(MessagesState)
builder.add_node(supervisor)
builder.add_node(agent_1)
builder.add_node(agent_2)

builder.add_edge(START, "supervisor")

supervisor = builder.compile()
```

:::

:::js

```typescript
import { StateGraph, MessagesZodState, Command, START, END } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { z } from "zod";

const model = new ChatOpenAI();

const supervisor = async (state: z.infer<typeof MessagesZodState>) => {
  // you can pass relevant parts of the state to the LLM (e.g., state.messages)
  // to determine which agent to call next. a common pattern is to call the model
  // with a structured output (e.g. force it to return an output with a "next_agent" field)
  const response = await model.invoke(...);
  // route to one of the agents or exit based on the supervisor's decision
  // if the supervisor returns "__end__", the graph will finish execution
  return new Command({ goto: response.nextAgent });
};

const agent1 = async (state: z.infer<typeof MessagesZodState>) => {
  // you can pass relevant parts of the state to the LLM (e.g., state.messages)
  // and add any additional logic (different models, custom prompts, structured output, etc.)
  const response = await model.invoke(...);
  return new Command({
    goto: "supervisor",
    update: { messages: [response] },
  });
};

const agent2 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return new Command({
    goto: "supervisor",
    update: { messages: [response] },
  });
};

const builder = new StateGraph(MessagesZodState)
  .addNode("supervisor", supervisor, {
    ends: ["agent1", "agent2", END]
  })
  .addNode("agent1", agent1, {
    ends: ["supervisor"]
  })
  .addNode("agent2", agent2, {
    ends: ["supervisor"]
  })
  .addEdge(START, "supervisor");

const supervisorGraph = builder.compile();
```

:::

:::js

```typescript
import { StateGraph, MessagesZodState, Command, START, END } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { z } from "zod";

const model = new ChatOpenAI();

const supervisor = async (state: z.infer<typeof MessagesZodState>) => {
  // you can pass relevant parts of the state to the LLM (e.g., state.messages)
  // to determine which agent to call next. a common pattern is to call the model
  // with a structured output (e.g. force it to return an output with a "next_agent" field)
  const response = await model.invoke(...);
  // route to one of the agents or exit based on the supervisor's decision
  // if the supervisor returns "__end__", the graph will finish execution
  return new Command({ goto: response.nextAgent });
};

const agent1 = async (state: z.infer<typeof MessagesZodState>) => {
  // you can pass relevant parts of the state to the LLM (e.g., state.messages)
  // and add any additional logic (different models, custom prompts, structured output, etc.)
  const response = await model.invoke(...);
  return new Command({
    goto: "supervisor",
    update: { messages: [response] },
  });
};

const agent2 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return new Command({
    goto: "supervisor",
    update: { messages: [response] },
  });
};

const builder = new StateGraph(MessagesZodState)
  .addNode("supervisor", supervisor, {
    ends: ["agent1", "agent2", END]
  })
  .addNode("agent1", agent1, {
    ends: ["supervisor"]
  })
  .addNode("agent2", agent2, {
    ends: ["supervisor"]
  })
  .addEdge(START, "supervisor");

const supervisorGraph = builder.compile();
```

:::

Check out this [tutorial](../tutorials/multi_agent/agent_supervisor.md) for an example of supervisor multi-agent architecture.

### Supervisor (tool-calling)

In this variant of the [supervisor](#supervisor) architecture, we define a supervisor [agent](./agentic_concepts.md#agent-architectures) which is responsible for calling sub-agents. The sub-agents are exposed to the supervisor as tools, and the supervisor agent decides which tool to call next. The supervisor agent follows a [standard implementation](./agentic_concepts.md#tool-calling-agent) as an LLM running in a while loop calling tools until it decides to stop.

:::python

```python
from typing import Annotated
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import InjectedState, create_react_agent

model = ChatOpenAI()

# this is the agent function that will be called as tool
# notice that you can pass the state to the tool via InjectedState annotation
def agent_1(state: Annotated[dict, InjectedState]):
    # you can pass relevant parts of the state to the LLM (e.g., state["messages"])
    # and add any additional logic (different models, custom prompts, structured output, etc.)
    response = model.invoke(...)
    # return the LLM response as a string (expected tool response format)
    # this will be automatically turned to ToolMessage
    # by the prebuilt create_react_agent (supervisor)
    return response.content

def agent_2(state: Annotated[dict, InjectedState]):
    response = model.invoke(...)
    return response.content

tools = [agent_1, agent_2]
# the simplest way to build a supervisor w/ tool-calling is to use prebuilt ReAct agent graph
# that consists of a tool-calling LLM node (i.e. supervisor) and a tool-executing node
supervisor = create_react_agent(model, tools)
```

:::

:::js

```typescript
import { ChatOpenAI } from "@langchain/openai";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const model = new ChatOpenAI();

// this is the agent function that will be called as tool
// notice that you can pass the state to the tool via config parameter
const agent1 = tool(
  async (_, config) => {
    const state = config.configurable?.state;
    // you can pass relevant parts of the state to the LLM (e.g., state.messages)
    // and add any additional logic (different models, custom prompts, structured output, etc.)
    const response = await model.invoke(...);
    // return the LLM response as a string (expected tool response format)
    // this will be automatically turned to ToolMessage
    // by the prebuilt createReactAgent (supervisor)
    return response.content;
  },
  {
    name: "agent1",
    description: "Agent 1 description",
    schema: z.object({}),
  }
);

const agent2 = tool(
  async (_, config) => {
    const state = config.configurable?.state;
    const response = await model.invoke(...);
    return response.content;
  },
  {
    name: "agent2",
    description: "Agent 2 description",
    schema: z.object({}),
  }
);

const tools = [agent1, agent2];
// the simplest way to build a supervisor w/ tool-calling is to use prebuilt ReAct agent graph
// that consists of a tool-calling LLM node (i.e. supervisor) and a tool-executing node
const supervisor = createReactAgent({ llm: model, tools });
```

:::

### Hierarchical

As you add more agents to your system, it might become too hard for the supervisor to manage all of them. The supervisor might start making poor decisions about which agent to call next, or the context might become too complex for a single supervisor to keep track of. In other words, you end up with the same problems that motivated the multi-agent architecture in the first place.

To address this, you can design your system _hierarchically_. For example, you can create separate, specialized teams of agents managed by individual supervisors, and a top-level supervisor to manage the teams.

:::python

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.types import Command
model = ChatOpenAI()

# define team 1 (same as the single supervisor example above)

def team_1_supervisor(state: MessagesState) -> Command[Literal["team_1_agent_1", "team_1_agent_2", END]]:
    response = model.invoke(...)
    return Command(goto=response["next_agent"])

def team_1_agent_1(state: MessagesState) -> Command[Literal["team_1_supervisor"]]:
    response = model.invoke(...)
    return Command(goto="team_1_supervisor", update={"messages": [response]})

def team_1_agent_2(state: MessagesState) -> Command[Literal["team_1_supervisor"]]:
    response = model.invoke(...)
    return Command(goto="team_1_supervisor", update={"messages": [response]})

team_1_builder = StateGraph(Team1State)
team_1_builder.add_node(team_1_supervisor)
team_1_builder.add_node(team_1_agent_1)
team_1_builder.add_node(team_1_agent_2)
team_1_builder.add_edge(START, "team_1_supervisor")
team_1_graph = team_1_builder.compile()

# define team 2 (same as the single supervisor example above)
class Team2State(MessagesState):
    next: Literal["team_2_agent_1", "team_2_agent_2", "__end__"]

def team_2_supervisor(state: Team2State):
    ...

def team_2_agent_1(state: Team2State):
    ...

def team_2_agent_2(state: Team2State):
    ...

team_2_builder = StateGraph(Team2State)
...
team_2_graph = team_2_builder.compile()


# define top-level supervisor

builder = StateGraph(MessagesState)
def top_level_supervisor(state: MessagesState) -> Command[Literal["team_1_graph", "team_2_graph", END]]:
    # you can pass relevant parts of the state to the LLM (e.g., state["messages"])
    # to determine which team to call next. a common pattern is to call the model
    # with a structured output (e.g. force it to return an output with a "next_team" field)
    response = model.invoke(...)
    # route to one of the teams or exit based on the supervisor's decision
    # if the supervisor returns "__end__", the graph will finish execution
    return Command(goto=response["next_team"])

builder = StateGraph(MessagesState)
builder.add_node(top_level_supervisor)
builder.add_node("team_1_graph", team_1_graph)
builder.add_node("team_2_graph", team_2_graph)
builder.add_edge(START, "top_level_supervisor")
builder.add_edge("team_1_graph", "top_level_supervisor")
builder.add_edge("team_2_graph", "top_level_supervisor")
graph = builder.compile()
```

:::

:::js

```typescript
import { StateGraph, MessagesZodState, Command, START, END } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { z } from "zod";

const model = new ChatOpenAI();

// define team 1 (same as the single supervisor example above)

const team1Supervisor = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return new Command({ goto: response.nextAgent });
};

const team1Agent1 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return new Command({
    goto: "team1Supervisor",
    update: { messages: [response] }
  });
};

const team1Agent2 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return new Command({
    goto: "team1Supervisor",
    update: { messages: [response] }
  });
};

const team1Builder = new StateGraph(MessagesZodState)
  .addNode("team1Supervisor", team1Supervisor, {
    ends: ["team1Agent1", "team1Agent2", END]
  })
  .addNode("team1Agent1", team1Agent1, {
    ends: ["team1Supervisor"]
  })
  .addNode("team1Agent2", team1Agent2, {
    ends: ["team1Supervisor"]
  })
  .addEdge(START, "team1Supervisor");
const team1Graph = team1Builder.compile();

// define team 2 (same as the single supervisor example above)
const team2Supervisor = async (state: z.infer<typeof MessagesZodState>) => {
  // ...
};

const team2Agent1 = async (state: z.infer<typeof MessagesZodState>) => {
  // ...
};

const team2Agent2 = async (state: z.infer<typeof MessagesZodState>) => {
  // ...
};

const team2Builder = new StateGraph(MessagesZodState);
// ... build team2Graph
const team2Graph = team2Builder.compile();

// define top-level supervisor

const topLevelSupervisor = async (state: z.infer<typeof MessagesZodState>) => {
  // you can pass relevant parts of the state to the LLM (e.g., state.messages)
  // to determine which team to call next. a common pattern is to call the model
  // with a structured output (e.g. force it to return an output with a "next_team" field)
  const response = await model.invoke(...);
  // route to one of the teams or exit based on the supervisor's decision
  // if the supervisor returns "__end__", the graph will finish execution
  return new Command({ goto: response.nextTeam });
};

const builder = new StateGraph(MessagesZodState)
  .addNode("topLevelSupervisor", topLevelSupervisor, {
    ends: ["team1Graph", "team2Graph", END]
  })
  .addNode("team1Graph", team1Graph)
  .addNode("team2Graph", team2Graph)
  .addEdge(START, "topLevelSupervisor")
  .addEdge("team1Graph", "topLevelSupervisor")
  .addEdge("team2Graph", "topLevelSupervisor");

const graph = builder.compile();
```

:::

### Custom multi-agent workflow

In this architecture we add individual agents as graph nodes and define the order in which agents are called ahead of time, in a custom workflow. In LangGraph the workflow can be defined in two ways:

- **Explicit control flow (normal edges)**: LangGraph allows you to explicitly define the control flow of your application (i.e. the sequence of how agents communicate) explicitly, via [normal graph edges](./low_level.md#normal-edges). This is the most deterministic variant of this architecture above — we always know which agent will be called next ahead of time.

- **Dynamic control flow (Command)**: in LangGraph you can allow LLMs to decide parts of your application control flow. This can be achieved by using [`Command`](./low_level.md#command). A special case of this is a [supervisor tool-calling](#supervisor-tool-calling) architecture. In that case, the tool-calling LLM powering the supervisor agent will make decisions about the order in which the tools (agents) are being called.

:::python

```python
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, MessagesState, START

model = ChatOpenAI()

def agent_1(state: MessagesState):
    response = model.invoke(...)
    return {"messages": [response]}

def agent_2(state: MessagesState):
    response = model.invoke(...)
    return {"messages": [response]}

builder = StateGraph(MessagesState)
builder.add_node(agent_1)
builder.add_node(agent_2)
# define the flow explicitly
builder.add_edge(START, "agent_1")
builder.add_edge("agent_1", "agent_2")
```

:::

:::js

```typescript
import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { z } from "zod";

const model = new ChatOpenAI();

const agent1 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return { messages: [response] };
};

const agent2 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return { messages: [response] };
};

const builder = new StateGraph(MessagesZodState)
  .addNode("agent1", agent1)
  .addNode("agent2", agent2)
  // define the flow explicitly
  .addEdge(START, "agent1")
  .addEdge("agent1", "agent2");
```

:::

## Communication and state management

The most important thing when building multi-agent systems is figuring out how the agents communicate.

A common, generic way for agents to communicate is via a list of messages. This opens up the following questions:

- Do agents communicate [**via handoffs or via tool calls**](#handoffs-vs-tool-calls)?
- What messages are [**passed from one agent to the next**](#message-passing-between-agents)?
- How are [**handoffs represented in the list of messages**](#representing-handoffs-in-message-history)?
- How do you [**manage state for subagents**](#state-management-for-subagents)?

Additionally, if you are dealing with more complex agents or wish to keep individual agent state separate from the multi-agent system state, you may need to use [**different state schemas**](#using-different-state-schemas).

### Handoffs vs tool calls

What is the "payload" that is being passed around between agents? In most of the architectures discussed above, the agents communicate via [handoffs](#handoffs) and pass the [graph state](./low_level.md#state) as part of the handoff payload. Specifically, agents pass around lists of messages as part of the graph state. In the case of the [supervisor with tool-calling](#supervisor-tool-calling), the payloads are tool call arguments.

![](./img/multi_agent/request.png)

### Message passing between agents

The most common way for agents to communicate is via a shared state channel, typically a list of messages. This assumes that there is always at least a single channel (key) in the state that is shared by the agents (e.g., `messages`). When communicating via a shared message list, there is an additional consideration: should the agents [share the full history](#sharing-full-thought-process) of their thought process or only [the final result](#sharing-only-final-results)?

![](./img/multi_agent/response.png)

#### Sharing full thought process

Agents can **share the full history** of their thought process (i.e., "scratchpad") with all other agents. This "scratchpad" would typically look like a [list of messages](./low_level.md#why-use-messages). The benefit of sharing the full thought process is that it might help other agents make better decisions and improve reasoning ability for the system as a whole. The downside is that as the number of agents and their complexity grows, the "scratchpad" will grow quickly and might require additional strategies for [memory management](../how-tos/memory/add-memory.md).

#### Sharing only final results

Agents can have their own private "scratchpad" and only **share the final result** with the rest of the agents. This approach might work better for systems with many agents or agents that are more complex. In this case, you would need to define agents with [different state schemas](#using-different-state-schemas).

For agents called as tools, the supervisor determines the inputs based on the tool schema. Additionally, LangGraph allows [passing state](../how-tos/tool-calling.md#short-term-memory) to individual tools at runtime, so subordinate agents can access parent state, if needed.

#### Indicating agent name in messages

It can be helpful to indicate which agent a particular AI message is from, especially for long message histories. Some LLM providers (like OpenAI) support adding a `name` parameter to messages — you can use that to attach the agent name to the message. If that is not supported, you can consider manually injecting the agent name into the message content, e.g., `<agent>alice</agent><message>message from alice</message>`.

### Representing handoffs in message history

:::python
Handoffs are typically done via the LLM calling a dedicated [handoff tool](#handoffs-as-tools). This is represented as an [AI message](https://python.langchain.com/docs/concepts/messages/#aimessage) with tool calls that is passed to the next agent (LLM). Most LLM providers don't support receiving AI messages with tool calls **without** corresponding tool messages.
:::

:::js
Handoffs are typically done via the LLM calling a dedicated [handoff tool](#handoffs-as-tools). This is represented as an [AI message](https://js.langchain.com/docs/concepts/messages/#aimessage) with tool calls that is passed to the next agent (LLM). Most LLM providers don't support receiving AI messages with tool calls **without** corresponding tool messages.
:::

You therefore have two options:

:::python

1. Add an extra [tool message](https://python.langchain.com/docs/concepts/messages/#toolmessage) to the message list, e.g., "Successfully transferred to agent X"
2. Remove the AI message with the tool calls
   :::

:::js

1. Add an extra [tool message](https://js.langchain.com/docs/concepts/messages/#toolmessage) to the message list, e.g., "Successfully transferred to agent X"
2. Remove the AI message with the tool calls
:::

In practice, we see that most developers opt for option (1).

### State management for subagents

A common practice is to have multiple agents communicating on a shared message list, but only [adding their final messages to the list](#sharing-only-final-results). This means that any intermediate messages (e.g., tool calls) are not saved in this list.

What if you **do** want to save these messages so that if this particular subagent is invoked in the future you can pass those back in?

There are two high-level approaches to achieve that:

:::python

1. Store these messages in the shared message list, but filter the list before passing it to the subagent LLM. For example, you can choose to filter out all tool calls from **other** agents.
2. Store a separate message list for each agent (e.g., `alice_messages`) in the subagent's graph state. This would be their "view" of what the message history looks like.
:::

:::js

1. Store these messages in the shared message list, but filter the list before passing it to the subagent LLM. For example, you can choose to filter out all tool calls from **other** agents.
2. Store a separate message list for each agent (e.g., `aliceMessages`) in the subagent's graph state. This would be their "view" of what the message history looks like.
:::

### Using different state schemas

An agent might need to have a different state schema from the rest of the agents. For example, a search agent might only need to keep track of queries and retrieved documents. There are two ways to achieve this in LangGraph:

- Define [subgraph](./subgraphs.md) agents with a separate state schema. If there are no shared state keys (channels) between the subgraph and the parent graph, it's important to [add input / output transformations](../how-tos/subgraph.md#different-state-schemas) so that the parent graph knows how to communicate with the subgraphs.
- Define agent node functions with a [private input state schema](../how-tos/graph-api.md#pass-private-state-between-nodes) that is distinct from the overall graph state schema. This allows passing information that is only needed for executing that particular agent.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/persistence.md
```md
---
search:
  boost: 2
---

# Persistence

LangGraph has a built-in persistence layer, implemented through checkpointers. When you compile a graph with a checkpointer, the checkpointer saves a `checkpoint` of the graph state at every super-step. Those checkpoints are saved to a `thread`, which can be accessed after graph execution. Because `threads` allow access to graph's state after execution, several powerful capabilities including human-in-the-loop, memory, time travel, and fault-tolerance are all possible. Below, we'll discuss each of these concepts in more detail.

![Checkpoints](img/persistence/checkpoints.jpg)

!!! info "LangGraph API handles checkpointing automatically"

    When using the LangGraph API, you don't need to implement or configure checkpointers manually. The API handles all persistence infrastructure for you behind the scenes.

## Threads

A thread is a unique ID or thread identifier assigned to each checkpoint saved by a checkpointer. It contains the accumulated state of a sequence of [runs](./assistants.md#execution). When a run is executed, the [state](../concepts/low_level.md#state) of the underlying graph of the assistant will be persisted to the thread.

When invoking a graph with a checkpointer, you **must** specify a `thread_id` as part of the `configurable` portion of the config:

:::python

```python
{"configurable": {"thread_id": "1"}}
```

:::

:::js

```typescript
{
  configurable: {
    thread_id: "1";
  }
}
```

:::

A thread's current and historical state can be retrieved. To persist state, a thread must be created prior to executing a run. The LangGraph Platform API provides several endpoints for creating and managing threads and thread state. See the [API reference](../cloud/reference/api/api_ref.html#tag/threads) for more details.

## Checkpoints

The state of a thread at a particular point in time is called a checkpoint. Checkpoint is a snapshot of the graph state saved at each super-step and is represented by `StateSnapshot` object with the following key properties:

- `config`: Config associated with this checkpoint.
- `metadata`: Metadata associated with this checkpoint.
- `values`: Values of the state channels at this point in time.
- `next` A tuple of the node names to execute next in the graph.
- `tasks`: A tuple of `PregelTask` objects that contain information about next tasks to be executed. If the step was previously attempted, it will include error information. If a graph was interrupted [dynamically](../how-tos/human_in_the_loop/add-human-in-the-loop.md#pause-using-interrupt) from within a node, tasks will contain additional data associated with interrupts.

Checkpoints are persisted and can be used to restore the state of a thread at a later time.

Let's see what checkpoints are saved when a simple graph is invoked as follows:

:::python

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: str
    bar: Annotated[list[str], add]

def node_a(state: State):
    return {"foo": "a", "bar": ["a"]}

def node_b(state: State):
    return {"foo": "b", "bar": ["b"]}


workflow = StateGraph(State)
workflow.add_node(node_a)
workflow.add_node(node_b)
workflow.add_edge(START, "node_a")
workflow.add_edge("node_a", "node_b")
workflow.add_edge("node_b", END)

checkpointer = InMemorySaver()
graph = workflow.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "1"}}
graph.invoke({"foo": ""}, config)
```

:::

:::js

```typescript
import { StateGraph, START, END, MemoryServer } from "@langchain/langgraph";
import { withLangGraph } from "@langchain/langgraph/zod";
import { z } from "zod";

const State = z.object({
  foo: z.string(),
  bar: withLangGraph(z.array(z.string()), {
    reducer: {
      fn: (x, y) => x.concat(y),
    },
    default: () => [],
  }),
});

const workflow = new StateGraph(State)
  .addNode("nodeA", (state) => {
    return { foo: "a", bar: ["a"] };
  })
  .addNode("nodeB", (state) => {
    return { foo: "b", bar: ["b"] };
  })
  .addEdge(START, "nodeA")
  .addEdge("nodeA", "nodeB")
  .addEdge("nodeB", END);

const checkpointer = new MemorySaver();
const graph = workflow.compile({ checkpointer });

const config = { configurable: { thread_id: "1" } };
await graph.invoke({ foo: "" }, config);
```

:::

:::js

```typescript
import { StateGraph, START, END, MemoryServer } from "@langchain/langgraph";
import { withLangGraph } from "@langchain/langgraph/zod";
import { z } from "zod";

const State = z.object({
  foo: z.string(),
  bar: withLangGraph(z.array(z.string()), {
    reducer: {
      fn: (x, y) => x.concat(y),
    },
    default: () => [],
  }),
});

const workflow = new StateGraph(State)
  .addNode("nodeA", (state) => {
    return { foo: "a", bar: ["a"] };
  })
  .addNode("nodeB", (state) => {
    return { foo: "b", bar: ["b"] };
  })
  .addEdge(START, "nodeA")
  .addEdge("nodeA", "nodeB")
  .addEdge("nodeB", END);

const checkpointer = new MemorySaver();
const graph = workflow.compile({ checkpointer });

const config = { configurable: { thread_id: "1" } };
await graph.invoke({ foo: "" }, config);
```

:::

:::python

After we run the graph, we expect to see exactly 4 checkpoints:

- empty checkpoint with `START` as the next node to be executed
- checkpoint with the user input `{'foo': '', 'bar': []}` and `node_a` as the next node to be executed
- checkpoint with the outputs of `node_a` `{'foo': 'a', 'bar': ['a']}` and `node_b` as the next node to be executed
- checkpoint with the outputs of `node_b` `{'foo': 'b', 'bar': ['a', 'b']}` and no next nodes to be executed

Note that we `bar` channel values contain outputs from both nodes as we have a reducer for `bar` channel.

:::

:::js

After we run the graph, we expect to see exactly 4 checkpoints:

- empty checkpoint with `START` as the next node to be executed
- checkpoint with the user input `{'foo': '', 'bar': []}` and `nodeA` as the next node to be executed
- checkpoint with the outputs of `nodeA` `{'foo': 'a', 'bar': ['a']}` and `nodeB` as the next node to be executed
- checkpoint with the outputs of `nodeB` `{'foo': 'b', 'bar': ['a', 'b']}` and no next nodes to be executed

Note that the `bar` channel values contain outputs from both nodes as we have a reducer for the `bar` channel.
:::

### Get state

:::python
When interacting with the saved graph state, you **must** specify a [thread identifier](#threads). You can view the _latest_ state of the graph by calling `graph.get_state(config)`. This will return a `StateSnapshot` object that corresponds to the latest checkpoint associated with the thread ID provided in the config or a checkpoint associated with a checkpoint ID for the thread, if provided.

```python
# get the latest state snapshot
config = {"configurable": {"thread_id": "1"}}
graph.get_state(config)

# get a state snapshot for a specific checkpoint_id
config = {"configurable": {"thread_id": "1", "checkpoint_id": "1ef663ba-28fe-6528-8002-5a559208592c"}}
graph.get_state(config)
```

:::

:::js
When interacting with the saved graph state, you **must** specify a [thread identifier](#threads). You can view the _latest_ state of the graph by calling `graph.getState(config)`. This will return a `StateSnapshot` object that corresponds to the latest checkpoint associated with the thread ID provided in the config or a checkpoint associated with a checkpoint ID for the thread, if provided.

```typescript
// get the latest state snapshot
const config = { configurable: { thread_id: "1" } };
await graph.getState(config);

// get a state snapshot for a specific checkpoint_id
const config = {
  configurable: {
    thread_id: "1",
    checkpoint_id: "1ef663ba-28fe-6528-8002-5a559208592c",
  },
};
await graph.getState(config);
```

:::

:::python
In our example, the output of `get_state` will look like this:

```
StateSnapshot(
    values={'foo': 'b', 'bar': ['a', 'b']},
    next=(),
    config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28fe-6528-8002-5a559208592c'}},
    metadata={'source': 'loop', 'writes': {'node_b': {'foo': 'b', 'bar': ['b']}}, 'step': 2},
    created_at='2024-08-29T19:19:38.821749+00:00',
    parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}}, tasks=()
)
```

:::

:::js
In our example, the output of `getState` will look like this:

```
StateSnapshot {
  values: { foo: 'b', bar: ['a', 'b'] },
  next: [],
  config: {
    configurable: {
      thread_id: '1',
      checkpoint_ns: '',
      checkpoint_id: '1ef663ba-28fe-6528-8002-5a559208592c'
    }
  },
  metadata: {
    source: 'loop',
    writes: { nodeB: { foo: 'b', bar: ['b'] } },
    step: 2
  },
  createdAt: '2024-08-29T19:19:38.821749+00:00',
  parentConfig: {
    configurable: {
      thread_id: '1',
      checkpoint_ns: '',
      checkpoint_id: '1ef663ba-28f9-6ec4-8001-31981c2c39f8'
    }
  },
  tasks: []
}
```

:::

### Get state history

:::python
You can get the full history of the graph execution for a given thread by calling `graph.get_state_history(config)`. This will return a list of `StateSnapshot` objects associated with the thread ID provided in the config. Importantly, the checkpoints will be ordered chronologically with the most recent checkpoint / `StateSnapshot` being the first in the list.

```python
config = {"configurable": {"thread_id": "1"}}
list(graph.get_state_history(config))
```

:::

:::js
You can get the full history of the graph execution for a given thread by calling `graph.getStateHistory(config)`. This will return a list of `StateSnapshot` objects associated with the thread ID provided in the config. Importantly, the checkpoints will be ordered chronologically with the most recent checkpoint / `StateSnapshot` being the first in the list.

```typescript
const config = { configurable: { thread_id: "1" } };
for await (const state of graph.getStateHistory(config)) {
  console.log(state);
}
```

:::

:::python
In our example, the output of `get_state_history` will look like this:

```
[
    StateSnapshot(
        values={'foo': 'b', 'bar': ['a', 'b']},
        next=(),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28fe-6528-8002-5a559208592c'}},
        metadata={'source': 'loop', 'writes': {'node_b': {'foo': 'b', 'bar': ['b']}}, 'step': 2},
        created_at='2024-08-29T19:19:38.821749+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}},
        tasks=(),
    ),
    StateSnapshot(
        values={'foo': 'a', 'bar': ['a']},
        next=('node_b',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}},
        metadata={'source': 'loop', 'writes': {'node_a': {'foo': 'a', 'bar': ['a']}}, 'step': 1},
        created_at='2024-08-29T19:19:38.819946+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f4-6b4a-8000-ca575a13d36a'}},
        tasks=(PregelTask(id='6fb7314f-f114-5413-a1f3-d37dfe98ff44', name='node_b', error=None, interrupts=()),),
    ),
    StateSnapshot(
        values={'foo': '', 'bar': []},
        next=('node_a',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f4-6b4a-8000-ca575a13d36a'}},
        metadata={'source': 'loop', 'writes': None, 'step': 0},
        created_at='2024-08-29T19:19:38.817813+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f0-6c66-bfff-6723431e8481'}},
        tasks=(PregelTask(id='f1b14528-5ee5-579c-949b-23ef9bfbed58', name='node_a', error=None, interrupts=()),),
    ),
    StateSnapshot(
        values={'bar': []},
        next=('__start__',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f0-6c66-bfff-6723431e8481'}},
        metadata={'source': 'input', 'writes': {'foo': ''}, 'step': -1},
        created_at='2024-08-29T19:19:38.816205+00:00',
        parent_config=None,
        tasks=(PregelTask(id='6d27aa2e-d72b-5504-a36f-8620e54a76dd', name='__start__', error=None, interrupts=()),),
    )
]
```

:::

:::js
In our example, the output of `getStateHistory` will look like this:

```
[
  StateSnapshot {
    values: { foo: 'b', bar: ['a', 'b'] },
    next: [],
    config: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28fe-6528-8002-5a559208592c'
      }
    },
    metadata: {
      source: 'loop',
      writes: { nodeB: { foo: 'b', bar: ['b'] } },
      step: 2
    },
    createdAt: '2024-08-29T19:19:38.821749+00:00',
    parentConfig: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f9-6ec4-8001-31981c2c39f8'
      }
    },
    tasks: []
  },
  StateSnapshot {
    values: { foo: 'a', bar: ['a'] },
    next: ['nodeB'],
    config: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f9-6ec4-8001-31981c2c39f8'
      }
    },
    metadata: {
      source: 'loop',
      writes: { nodeA: { foo: 'a', bar: ['a'] } },
      step: 1
    },
    createdAt: '2024-08-29T19:19:38.819946+00:00',
    parentConfig: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f4-6b4a-8000-ca575a13d36a'
      }
    },
    tasks: [
      PregelTask {
        id: '6fb7314f-f114-5413-a1f3-d37dfe98ff44',
        name: 'nodeB',
        error: null,
        interrupts: []
      }
    ]
  },
  StateSnapshot {
    values: { foo: '', bar: [] },
    next: ['node_a'],
    config: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f4-6b4a-8000-ca575a13d36a'
      }
    },
    metadata: {
      source: 'loop',
      writes: null,
      step: 0
    },
    createdAt: '2024-08-29T19:19:38.817813+00:00',
    parentConfig: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f0-6c66-bfff-6723431e8481'
      }
    },
    tasks: [
      PregelTask {
        id: 'f1b14528-5ee5-579c-949b-23ef9bfbed58',
        name: 'node_a',
        error: null,
        interrupts: []
      }
    ]
  },
  StateSnapshot {
    values: { bar: [] },
    next: ['__start__'],
    config: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f0-6c66-bfff-6723431e8481'
      }
    },
    metadata: {
      source: 'input',
      writes: { foo: '' },
      step: -1
    },
    createdAt: '2024-08-29T19:19:38.816205+00:00',
    parentConfig: null,
    tasks: [
      PregelTask {
        id: '6d27aa2e-d72b-5504-a36f-8620e54a76dd',
        name: '__start__',
        error: null,
        interrupts: []
      }
    ]
  }
]
```

:::

![State](img/persistence/get_state.jpg)

### Replay

It's also possible to play-back a prior graph execution. If we `invoke` a graph with a `thread_id` and a `checkpoint_id`, then we will _re-play_ the previously executed steps _before_ a checkpoint that corresponds to the `checkpoint_id`, and only execute the steps _after_ the checkpoint.

- `thread_id` is the ID of a thread.
- `checkpoint_id` is an identifier that refers to a specific checkpoint within a thread.

You must pass these when invoking the graph as part of the `configurable` portion of the config:

:::python

```python
config = {"configurable": {"thread_id": "1", "checkpoint_id": "0c62ca34-ac19-445d-bbb0-5b4984975b2a"}}
graph.invoke(None, config=config)
```

:::

:::js

```typescript
const config = {
  configurable: {
    thread_id: "1",
    checkpoint_id: "0c62ca34-ac19-445d-bbb0-5b4984975b2a",
  },
};
await graph.invoke(null, config);
```

:::

Importantly, LangGraph knows whether a particular step has been executed previously. If it has, LangGraph simply _re-plays_ that particular step in the graph and does not re-execute the step, but only for the steps _before_ the provided `checkpoint_id`. All of the steps _after_ `checkpoint_id` will be executed (i.e., a new fork), even if they have been executed previously. See this [how to guide on time-travel to learn more about replaying](../how-tos/human_in_the_loop/time-travel.md).

![Replay](img/persistence/re_play.png)

### Update state

:::python

In addition to re-playing the graph from specific `checkpoints`, we can also _edit_ the graph state. We do this using `graph.update_state()`. This method accepts three different arguments:

:::

:::js

In addition to re-playing the graph from specific `checkpoints`, we can also _edit_ the graph state. We do this using `graph.updateState()`. This method accepts three different arguments:

:::

#### `config`

The config should contain `thread_id` specifying which thread to update. When only the `thread_id` is passed, we update (or fork) the current state. Optionally, if we include `checkpoint_id` field, then we fork that selected checkpoint.

#### `values`

These are the values that will be used to update the state. Note that this update is treated exactly as any update from a node is treated. This means that these values will be passed to the [reducer](./low_level.md#reducers) functions, if they are defined for some of the channels in the graph state. This means that `update_state` does NOT automatically overwrite the channel values for every channel, but only for the channels without reducers. Let's walk through an example.

Let's assume you have defined the state of your graph with the following schema (see full example above):

:::python

```python
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: int
    bar: Annotated[list[str], add]
```

:::

:::js

```typescript
import { withLangGraph } from "@langchain/langgraph/zod";
import { z } from "zod";

const State = z.object({
  foo: z.number(),
  bar: withLangGraph(z.array(z.string()), {
    reducer: {
      fn: (x, y) => x.concat(y),
    },
    default: () => [],
  }),
});
```

:::

Let's now assume the current state of the graph is

:::python

```
{"foo": 1, "bar": ["a"]}
```

:::

:::js

```typescript
{ foo: 1, bar: ["a"] }
```

:::

If you update the state as below:

:::python

```python
graph.update_state(config, {"foo": 2, "bar": ["b"]})
```

:::

:::js

```typescript
await graph.updateState(config, { foo: 2, bar: ["b"] });
```

:::

Then the new state of the graph will be:

:::python

```
{"foo": 2, "bar": ["a", "b"]}
```

The `foo` key (channel) is completely changed (because there is no reducer specified for that channel, so `update_state` overwrites it). However, there is a reducer specified for the `bar` key, and so it appends `"b"` to the state of `bar`.
:::

:::js

```typescript
{ foo: 2, bar: ["a", "b"] }
```

The `foo` key (channel) is completely changed (because there is no reducer specified for that channel, so `updateState` overwrites it). However, there is a reducer specified for the `bar` key, and so it appends `"b"` to the state of `bar`.
:::

#### `as_node`

:::python
The final thing you can optionally specify when calling `update_state` is `as_node`. If you provided it, the update will be applied as if it came from node `as_node`. If `as_node` is not provided, it will be set to the last node that updated the state, if not ambiguous. The reason this matters is that the next steps to execute depend on the last node to have given an update, so this can be used to control which node executes next. See this [how to guide on time-travel to learn more about forking state](../how-tos/human_in_the_loop/time-travel.md).
:::

:::js
The final thing you can optionally specify when calling `updateState` is `asNode`. If you provide it, the update will be applied as if it came from node `asNode`. If `asNode` is not provided, it will be set to the last node that updated the state, if not ambiguous. The reason this matters is that the next steps to execute depend on the last node to have given an update, so this can be used to control which node executes next. See this [how to guide on time-travel to learn more about forking state](../how-tos/human_in_the_loop/time-travel.md).
:::

![Update](img/persistence/checkpoints_full_story.jpg)

## Memory Store

![Model of shared state](img/persistence/shared_state.png)

A [state schema](low_level.md#schema) specifies a set of keys that are populated as a graph is executed. As discussed above, state can be written by a checkpointer to a thread at each graph step, enabling state persistence.

But, what if we want to retain some information _across threads_? Consider the case of a chatbot where we want to retain specific information about the user across _all_ chat conversations (e.g., threads) with that user!

With checkpointers alone, we cannot share information across threads. This motivates the need for the [`Store`](../reference/store.md#langgraph.store.base.BaseStore) interface. As an illustration, we can define an `InMemoryStore` to store information about a user across threads. We simply compile our graph with a checkpointer, as before, and with our new `in_memory_store` variable.

!!! info "LangGraph API handles stores automatically"

    When using the LangGraph API, you don't need to implement or configure stores manually. The API handles all storage infrastructure for you behind the scenes.

### Basic Usage

First, let's showcase this in isolation without using LangGraph.

:::python

```python
from langgraph.store.memory import InMemoryStore
in_memory_store = InMemoryStore()
```

:::

:::js

```typescript
import { MemoryStore } from "@langchain/langgraph";

const memoryStore = new MemoryStore();
```

:::

Memories are namespaced by a `tuple`, which in this specific example will be `(<user_id>, "memories")`. The namespace can be any length and represent anything, does not have to be user specific.

:::python

```python
user_id = "1"
namespace_for_memory = (user_id, "memories")
```

:::

:::js

```typescript
const userId = "1";
const namespaceForMemory = [userId, "memories"];
```

:::

We use the `store.put` method to save memories to our namespace in the store. When we do this, we specify the namespace, as defined above, and a key-value pair for the memory: the key is simply a unique identifier for the memory (`memory_id`) and the value (a dictionary) is the memory itself.

:::python

```python
memory_id = str(uuid.uuid4())
memory = {"food_preference" : "I like pizza"}
in_memory_store.put(namespace_for_memory, memory_id, memory)
```

:::

:::js

```typescript
import { v4 as uuidv4 } from "uuid";

const memoryId = uuidv4();
const memory = { food_preference: "I like pizza" };
await memoryStore.put(namespaceForMemory, memoryId, memory);
```

:::

We can read out memories in our namespace using the `store.search` method, which will return all memories for a given user as a list. The most recent memory is the last in the list.

:::python

```python
memories = in_memory_store.search(namespace_for_memory)
memories[-1].dict()
{'value': {'food_preference': 'I like pizza'},
 'key': '07e0caf4-1631-47b7-b15f-65515d4c1843',
 'namespace': ['1', 'memories'],
 'created_at': '2024-10-02T17:22:31.590602+00:00',
 'updated_at': '2024-10-02T17:22:31.590605+00:00'}
```

Each memory type is a Python class ([`Item`](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.Item)) with certain attributes. We can access it as a dictionary by converting via `.dict` as above.

The attributes it has are:

- `value`: The value (itself a dictionary) of this memory
- `key`: A unique key for this memory in this namespace
- `namespace`: A list of strings, the namespace of this memory type
- `created_at`: Timestamp for when this memory was created
- `updated_at`: Timestamp for when this memory was updated

:::

:::js

```typescript
const memories = await memoryStore.search(namespaceForMemory);
memories[memories.length - 1];

// {
//   value: { food_preference: 'I like pizza' },
//   key: '07e0caf4-1631-47b7-b15f-65515d4c1843',
//   namespace: ['1', 'memories'],
//   createdAt: '2024-10-02T17:22:31.590602+00:00',
//   updatedAt: '2024-10-02T17:22:31.590605+00:00'
// }
```

The attributes it has are:

- `value`: The value of this memory
- `key`: A unique key for this memory in this namespace
- `namespace`: A list of strings, the namespace of this memory type
- `createdAt`: Timestamp for when this memory was created
- `updatedAt`: Timestamp for when this memory was updated

:::

### Semantic Search

Beyond simple retrieval, the store also supports semantic search, allowing you to find memories based on meaning rather than exact matches. To enable this, configure the store with an embedding model:

:::python

```python
from langchain.embeddings import init_embeddings

store = InMemoryStore(
    index={
        "embed": init_embeddings("openai:text-embedding-3-small"),  # Embedding provider
        "dims": 1536,                              # Embedding dimensions
        "fields": ["food_preference", "$"]              # Fields to embed
    }
)
```

:::

:::js

```typescript
import { OpenAIEmbeddings } from "@langchain/openai";

const store = new InMemoryStore({
  index: {
    embeddings: new OpenAIEmbeddings({ model: "text-embedding-3-small" }),
    dims: 1536,
    fields: ["food_preference", "$"], // Fields to embed
  },
});
```

:::

Now when searching, you can use natural language queries to find relevant memories:

:::python

```python
# Find memories about food preferences
# (This can be done after putting memories into the store)
memories = store.search(
    namespace_for_memory,
    query="What does the user like to eat?",
    limit=3  # Return top 3 matches
)
```

:::

:::js

```typescript
// Find memories about food preferences
// (This can be done after putting memories into the store)
const memories = await store.search(namespaceForMemory, {
  query: "What does the user like to eat?",
  limit: 3, // Return top 3 matches
});
```

:::

You can control which parts of your memories get embedded by configuring the `fields` parameter or by specifying the `index` parameter when storing memories:

:::python

```python
# Store with specific fields to embed
store.put(
    namespace_for_memory,
    str(uuid.uuid4()),
    {
        "food_preference": "I love Italian cuisine",
        "context": "Discussing dinner plans"
    },
    index=["food_preference"]  # Only embed "food_preferences" field
)

# Store without embedding (still retrievable, but not searchable)
store.put(
    namespace_for_memory,
    str(uuid.uuid4()),
    {"system_info": "Last updated: 2024-01-01"},
    index=False
)
```

:::

:::js

```typescript
// Store with specific fields to embed
await store.put(
  namespaceForMemory,
  uuidv4(),
  {
    food_preference: "I love Italian cuisine",
    context: "Discussing dinner plans",
  },
  { index: ["food_preference"] } // Only embed "food_preferences" field
);

// Store without embedding (still retrievable, but not searchable)
await store.put(
  namespaceForMemory,
  uuidv4(),
  { system_info: "Last updated: 2024-01-01" },
  { index: false }
);
```

:::

### Using in LangGraph

:::python
With this all in place, we use the `in_memory_store` in LangGraph. The `in_memory_store` works hand-in-hand with the checkpointer: the checkpointer saves state to threads, as discussed above, and the `in_memory_store` allows us to store arbitrary information for access _across_ threads. We compile the graph with both the checkpointer and the `in_memory_store` as follows.

```python
from langgraph.checkpoint.memory import InMemorySaver

# We need this because we want to enable threads (conversations)
checkpointer = InMemorySaver()

# ... Define the graph ...

# Compile the graph with the checkpointer and store
graph = graph.compile(checkpointer=checkpointer, store=in_memory_store)
```

:::

:::js
With this all in place, we use the `memoryStore` in LangGraph. The `memoryStore` works hand-in-hand with the checkpointer: the checkpointer saves state to threads, as discussed above, and the `memoryStore` allows us to store arbitrary information for access _across_ threads. We compile the graph with both the checkpointer and the `memoryStore` as follows.

```typescript
import { MemorySaver } from "@langchain/langgraph";

// We need this because we want to enable threads (conversations)
const checkpointer = new MemorySaver();

// ... Define the graph ...

// Compile the graph with the checkpointer and store
const graph = workflow.compile({ checkpointer, store: memoryStore });
```

:::

We invoke the graph with a `thread_id`, as before, and also with a `user_id`, which we'll use to namespace our memories to this particular user as we showed above.

:::python

```python
# Invoke the graph
user_id = "1"
config = {"configurable": {"thread_id": "1", "user_id": user_id}}

# First let's just say hi to the AI
for update in graph.stream(
    {"messages": [{"role": "user", "content": "hi"}]}, config, stream_mode="updates"
):
    print(update)
```

:::

:::js

```typescript
// Invoke the graph
const userId = "1";
const config = { configurable: { thread_id: "1", user_id: userId } };

// First let's just say hi to the AI
for await (const update of await graph.stream(
  { messages: [{ role: "user", content: "hi" }] },
  { ...config, streamMode: "updates" }
)) {
  console.log(update);
}
```

:::

:::python
We can access the `in_memory_store` and the `user_id` in _any node_ by passing `store: BaseStore` and `config: RunnableConfig` as node arguments. Here's how we might use semantic search in a node to find relevant memories:

```python
def update_memory(state: MessagesState, config: RunnableConfig, *, store: BaseStore):

    # Get the user id from the config
    user_id = config["configurable"]["user_id"]

    # Namespace the memory
    namespace = (user_id, "memories")

    # ... Analyze conversation and create a new memory

    # Create a new memory ID
    memory_id = str(uuid.uuid4())

    # We create a new memory
    store.put(namespace, memory_id, {"memory": memory})

```

:::

:::js
We can access the `memoryStore` and the `user_id` in _any node_ by accessing `config` and `store` as node arguments. Here's how we might use semantic search in a node to find relevant memories:

```typescript
import {
  LangGraphRunnableConfig,
  BaseStore,
  MessagesZodState,
} from "@langchain/langgraph";
import { z } from "zod";

const updateMemory = async (
  state: z.infer<typeof MessagesZodState>,
  config: LangGraphRunnableConfig,
  store: BaseStore
) => {
  // Get the user id from the config
  const userId = config.configurable?.user_id;

  // Namespace the memory
  const namespace = [userId, "memories"];

  // ... Analyze conversation and create a new memory

  // Create a new memory ID
  const memoryId = uuidv4();

  // We create a new memory
  await store.put(namespace, memoryId, { memory });
};
```

:::

As we showed above, we can also access the store in any node and use the `store.search` method to get memories. Recall the memories are returned as a list of objects that can be converted to a dictionary.

:::python

```python
memories[-1].dict()
{'value': {'food_preference': 'I like pizza'},
 'key': '07e0caf4-1631-47b7-b15f-65515d4c1843',
 'namespace': ['1', 'memories'],
 'created_at': '2024-10-02T17:22:31.590602+00:00',
 'updated_at': '2024-10-02T17:22:31.590605+00:00'}
```

:::

:::js

```typescript
memories[memories.length - 1];
// {
//   value: { food_preference: 'I like pizza' },
//   key: '07e0caf4-1631-47b7-b15f-65515d4c1843',
//   namespace: ['1', 'memories'],
//   createdAt: '2024-10-02T17:22:31.590602+00:00',
//   updatedAt: '2024-10-02T17:22:31.590605+00:00'
// }
```

:::

We can access the memories and use them in our model call.

:::python

```python
def call_model(state: MessagesState, config: RunnableConfig, *, store: BaseStore):
    # Get the user id from the config
    user_id = config["configurable"]["user_id"]

    # Namespace the memory
    namespace = (user_id, "memories")

    # Search based on the most recent message
    memories = store.search(
        namespace,
        query=state["messages"][-1].content,
        limit=3
    )
    info = "\n".join([d.value["memory"] for d in memories])

    # ... Use memories in the model call
```

:::

:::js

```typescript
const callModel = async (
  state: z.infer<typeof MessagesZodState>,
  config: LangGraphRunnableConfig,
  store: BaseStore
) => {
  // Get the user id from the config
  const userId = config.configurable?.user_id;

  // Namespace the memory
  const namespace = [userId, "memories"];

  // Search based on the most recent message
  const memories = await store.search(namespace, {
    query: state.messages[state.messages.length - 1].content,
    limit: 3,
  });
  const info = memories.map((d) => d.value.memory).join("\n");

  // ... Use memories in the model call
};
```

:::

If we create a new thread, we can still access the same memories so long as the `user_id` is the same.

:::python

```python
# Invoke the graph
config = {"configurable": {"thread_id": "2", "user_id": "1"}}

# Let's say hi again
for update in graph.stream(
    {"messages": [{"role": "user", "content": "hi, tell me about my memories"}]}, config, stream_mode="updates"
):
    print(update)
```

:::

:::js

```typescript
// Invoke the graph
const config = { configurable: { thread_id: "2", user_id: "1" } };

// Let's say hi again
for await (const update of await graph.stream(
  { messages: [{ role: "user", content: "hi, tell me about my memories" }] },
  { ...config, streamMode: "updates" }
)) {
  console.log(update);
}
```

:::

When we use the LangGraph Platform, either locally (e.g., in LangGraph Studio) or with LangGraph Platform, the base store is available to use by default and does not need to be specified during graph compilation. To enable semantic search, however, you **do** need to configure the indexing settings in your `langgraph.json` file. For example:

```json
{
    ...
    "store": {
        "index": {
            "embed": "openai:text-embeddings-3-small",
            "dims": 1536,
            "fields": ["$"]
        }
    }
}
```

See the [deployment guide](../cloud/deployment/semantic_search.md) for more details and configuration options.

## Checkpointer libraries

Under the hood, checkpointing is powered by checkpointer objects that conform to @[BaseCheckpointSaver] interface. LangGraph provides several checkpointer implementations, all implemented via standalone, installable libraries:

:::python

- `langgraph-checkpoint`: The base interface for checkpointer savers (@[BaseCheckpointSaver]) and serialization/deserialization interface (@[SerializerProtocol][SerializerProtocol]). Includes in-memory checkpointer implementation (@[InMemorySaver][InMemorySaver]) for experimentation. LangGraph comes with `langgraph-checkpoint` included.
- `langgraph-checkpoint-sqlite`: An implementation of LangGraph checkpointer that uses SQLite database (@[SqliteSaver][SqliteSaver] / @[AsyncSqliteSaver]). Ideal for experimentation and local workflows. Needs to be installed separately.
- `langgraph-checkpoint-postgres`: An advanced checkpointer that uses Postgres database (@[PostgresSaver][PostgresSaver] / @[AsyncPostgresSaver]), used in LangGraph Platform. Ideal for using in production. Needs to be installed separately.

:::

:::js

- `@langchain/langgraph-checkpoint`: The base interface for checkpointer savers (@[BaseCheckpointSaver][BaseCheckpointSaver]) and serialization/deserialization interface (@[SerializerProtocol][SerializerProtocol]). Includes in-memory checkpointer implementation (@[MemorySaver]) for experimentation. LangGraph comes with `@langchain/langgraph-checkpoint` included.
- `@langchain/langgraph-checkpoint-sqlite`: An implementation of LangGraph checkpointer that uses SQLite database (@[SqliteSaver]). Ideal for experimentation and local workflows. Needs to be installed separately.
- `@langchain/langgraph-checkpoint-postgres`: An advanced checkpointer that uses Postgres database (@[PostgresSaver]), used in LangGraph Platform. Ideal for using in production. Needs to be installed separately.

:::

### Checkpointer interface

:::python
Each checkpointer conforms to @[BaseCheckpointSaver] interface and implements the following methods:

- `.put` - Store a checkpoint with its configuration and metadata.
- `.put_writes` - Store intermediate writes linked to a checkpoint (i.e. [pending writes](#pending-writes)).
- `.get_tuple` - Fetch a checkpoint tuple using for a given configuration (`thread_id` and `checkpoint_id`). This is used to populate `StateSnapshot` in `graph.get_state()`.
- `.list` - List checkpoints that match a given configuration and filter criteria. This is used to populate state history in `graph.get_state_history()`

If the checkpointer is used with asynchronous graph execution (i.e. executing the graph via `.ainvoke`, `.astream`, `.abatch`), asynchronous versions of the above methods will be used (`.aput`, `.aput_writes`, `.aget_tuple`, `.alist`).

!!! note

    For running your graph asynchronously, you can use `InMemorySaver`, or async versions of Sqlite/Postgres checkpointers -- `AsyncSqliteSaver` / `AsyncPostgresSaver` checkpointers.

:::

:::js
Each checkpointer conforms to the @[BaseCheckpointSaver][BaseCheckpointSaver] interface and implements the following methods:

- `.put` - Store a checkpoint with its configuration and metadata.
- `.putWrites` - Store intermediate writes linked to a checkpoint (i.e. [pending writes](#pending-writes)).
- `.getTuple` - Fetch a checkpoint tuple using for a given configuration (`thread_id` and `checkpoint_id`). This is used to populate `StateSnapshot` in `graph.getState()`.
- `.list` - List checkpoints that match a given configuration and filter criteria. This is used to populate state history in `graph.getStateHistory()`
  :::

### Serializer

When checkpointers save the graph state, they need to serialize the channel values in the state. This is done using serializer objects.

:::python
`langgraph_checkpoint` defines @[protocol][SerializerProtocol] for implementing serializers provides a default implementation (@[JsonPlusSerializer][JsonPlusSerializer]) that handles a wide variety of types, including LangChain and LangGraph primitives, datetimes, enums and more.

#### Serialization with `pickle`

The default serializer, @[`JsonPlusSerializer`][JsonPlusSerializer], uses ormsgpack and JSON under the hood, which is not suitable for all types of objects.

If you want to fallback to pickle for objects not currently supported by our msgpack encoder (such as Pandas dataframes),
you can use the `pickle_fallback` argument of the `JsonPlusSerializer`:

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.checkpoint.serde.jsonplus import JsonPlusSerializer

# ... Define the graph ...
graph.compile(
    checkpointer=InMemorySaver(serde=JsonPlusSerializer(pickle_fallback=True))
)
```

#### Encryption

Checkpointers can optionally encrypt all persisted state. To enable this, pass an instance of @[`EncryptedSerializer`][EncryptedSerializer] to the `serde` argument of any `BaseCheckpointSaver` implementation. The easiest way to create an encrypted serializer is via @[`from_pycryptodome_aes`][from_pycryptodome_aes], which reads the AES key from the `LANGGRAPH_AES_KEY` environment variable (or accepts a `key` argument):

```python
import sqlite3

from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.checkpoint.sqlite import SqliteSaver

serde = EncryptedSerializer.from_pycryptodome_aes()  # reads LANGGRAPH_AES_KEY
checkpointer = SqliteSaver(sqlite3.connect("checkpoint.db"), serde=serde)
```

```python
from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.checkpoint.postgres import PostgresSaver

serde = EncryptedSerializer.from_pycryptodome_aes()
checkpointer = PostgresSaver.from_conn_string("postgresql://...", serde=serde)
checkpointer.setup()
```

When running on LangGraph Platform, encryption is automatically enabled whenever `LANGGRAPH_AES_KEY` is present, so you only need to provide the environment variable. Other encryption schemes can be used by implementing @[`CipherProtocol`][CipherProtocol] and supplying it to `EncryptedSerializer`.
:::

:::js
`@langchain/langgraph-checkpoint` defines protocol for implementing serializers and provides a default implementation that handles a wide variety of types, including LangChain and LangGraph primitives, datetimes, enums and more.
:::

## Capabilities

### Human-in-the-loop

First, checkpointers facilitate [human-in-the-loop workflows](agentic_concepts.md#human-in-the-loop) workflows by allowing humans to inspect, interrupt, and approve graph steps. Checkpointers are needed for these workflows as the human has to be able to view the state of a graph at any point in time, and the graph has to be to resume execution after the human has made any updates to the state. See [the how-to guides](../how-tos/human_in_the_loop/add-human-in-the-loop.md) for examples.

### Memory

Second, checkpointers allow for ["memory"](../concepts/memory.md) between interactions. In the case of repeated human interactions (like conversations) any follow up messages can be sent to that thread, which will retain its memory of previous ones. See [Add memory](../how-tos/memory/add-memory.md) for information on how to add and manage conversation memory using checkpointers.

### Time Travel

Third, checkpointers allow for ["time travel"](time-travel.md), allowing users to replay prior graph executions to review and / or debug specific graph steps. In addition, checkpointers make it possible to fork the graph state at arbitrary checkpoints to explore alternative trajectories.

### Fault-tolerance

Lastly, checkpointing also provides fault-tolerance and error recovery: if one or more nodes fail at a given superstep, you can restart your graph from the last successful step. Additionally, when a graph node fails mid-execution at a given superstep, LangGraph stores pending checkpoint writes from any other nodes that completed successfully at that superstep, so that whenever we resume graph execution from that superstep we don't re-run the successful nodes.

#### Pending writes

Additionally, when a graph node fails mid-execution at a given superstep, LangGraph stores pending checkpoint writes from any other nodes that completed successfully at that superstep, so that whenever we resume graph execution from that superstep we don't re-run the successful nodes.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/plans.md
```md
---
search:
  boost: 2
---

# LangGraph Platform Plans


## Overview
LangGraph Platform is a solution for deploying agentic applications in production.
There are three different plans for using it.

- **Developer**: All [LangSmith](https://smith.langchain.com/) users have access to this plan. You can sign up for this plan simply by creating a LangSmith account. This gives you access to the [local deployment](./deployment_options.md#free-deployment) option.
- **Plus**: All [LangSmith](https://smith.langchain.com/) users with a [Plus account](https://docs.smith.langchain.com/administration/pricing) have access to this plan. You can sign up for this plan simply by upgrading your LangSmith account to the Plus plan type. This gives you access to the [Cloud](./deployment_options.md#cloud-saas) deployment option.
- **Enterprise**: This is separate from LangSmith plans. You can sign up for this plan by [contacting our sales team](https://www.langchain.com/contact-sales). This gives you access to all [deployment options](./deployment_options.md).


## Plan Details

|                                                                  | Developer                                   | Plus                                                  | Enterprise                                          |
|------------------------------------------------------------------|---------------------------------------------|-------------------------------------------------------|-----------------------------------------------------|
| Deployment Options                                               | Local                          | Cloud SaaS                                         | <ul><li>Cloud SaaS</li><li>Self-Hosted Data Plane</li><li>Self-Hosted Control Plane</li><li>Standalone Container</li></ul> |
| Usage                                                            | Free | See [Pricing](https://www.langchain.com/langgraph-platform-pricing) | Custom                                              |
| APIs for retrieving and updating state and conversational history | ✅                                           | ✅                                                     | ✅                                                   |
| APIs for retrieving and updating long-term memory                | ✅                                           | ✅                                                     | ✅                                                   |
| Horizontally scalable task queues and servers                    | ✅                                           | ✅                                                     | ✅                                                   |
| Real-time streaming of outputs and intermediate steps            | ✅                                           | ✅                                                     | ✅                                                   |
| Assistants API (configurable templates for LangGraph apps)       | ✅                                           | ✅                                                     | ✅                                                   |
| Cron scheduling                                                  | --                                          | ✅                                                     | ✅                                                   |
| LangGraph Studio for prototyping                                 | 	✅                                         | ✅                                                    | ✅                                                  |
| Authentication & authorization to call the LangGraph APIs        | --                                          | Coming Soon!                                          | Coming Soon!                                        |
| Smart caching to reduce traffic to LLM API                       | --                                          | Coming Soon!                                          | Coming Soon!                                        |
| Publish/subscribe API for state                                  | --                                          | Coming Soon!                                          | Coming Soon!                                        |
| Scheduling prioritization                                        | --                                          | Coming Soon!                                          | Coming Soon!                                        |

For pricing information, see [LangGraph Platform Pricing](https://www.langchain.com/langgraph-platform-pricing).

## Related

For more information, please see:

* [Deployment Options conceptual guide](./deployment_options.md)
* [LangGraph Platform Pricing](https://www.langchain.com/langgraph-platform-pricing)
* [LangSmith Plans](https://docs.smith.langchain.com/administration/pricing)

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/pregel.md
```md
---
search:
  boost: 2
---

# LangGraph runtime

:::python
@[Pregel] implements LangGraph's runtime, managing the execution of LangGraph applications.

Compiling a @[StateGraph][StateGraph] or creating an @[entrypoint][entrypoint] produces a @[Pregel] instance that can be invoked with input.
:::

:::js
@[Pregel] implements LangGraph's runtime, managing the execution of LangGraph applications.

Compiling a @[StateGraph][StateGraph] or creating an @[entrypoint][entrypoint] produces a @[Pregel] instance that can be invoked with input.
:::

This guide explains the runtime at a high level and provides instructions for directly implementing applications with Pregel.

:::python

> **Note:** The @[Pregel] runtime is named after [Google's Pregel algorithm](https://research.google/pubs/pub37252/), which describes an efficient method for large-scale parallel computation using graphs.

:::

:::js

> **Note:** The @[Pregel] runtime is named after [Google's Pregel algorithm](https://research.google/pubs/pub37252/), which describes an efficient method for large-scale parallel computation using graphs.

:::

## Overview

In LangGraph, Pregel combines [**actors**](https://en.wikipedia.org/wiki/Actor_model) and **channels** into a single application. **Actors** read data from channels and write data to channels. Pregel organizes the execution of the application into multiple steps, following the **Pregel Algorithm**/**Bulk Synchronous Parallel** model.

Each step consists of three phases:

- **Plan**: Determine which **actors** to execute in this step. For example, in the first step, select the **actors** that subscribe to the special **input** channels; in subsequent steps, select the **actors** that subscribe to channels updated in the previous step.
- **Execution**: Execute all selected **actors** in parallel, until all complete, or one fails, or a timeout is reached. During this phase, channel updates are invisible to actors until the next step.
- **Update**: Update the channels with the values written by the **actors** in this step.

Repeat until no **actors** are selected for execution, or a maximum number of steps is reached.

## Actors

An **actor** is a `PregelNode`. It subscribes to channels, reads data from them, and writes data to them. It can be thought of as an **actor** in the Pregel algorithm. `PregelNodes` implement LangChain's Runnable interface.

## Channels

Channels are used to communicate between actors (PregelNodes). Each channel has a value type, an update type, and an update function – which takes a sequence of updates and modifies the stored value. Channels can be used to send data from one chain to another, or to send data from a chain to itself in a future step. LangGraph provides a number of built-in channels:

:::python

- @[LastValue][LastValue]: The default channel, stores the last value sent to the channel, useful for input and output values, or for sending data from one step to the next.
- @[Topic][Topic]: A configurable PubSub Topic, useful for sending multiple values between **actors**, or for accumulating output. Can be configured to deduplicate values or to accumulate values over the course of multiple steps.
- @[BinaryOperatorAggregate][BinaryOperatorAggregate]: stores a persistent value, updated by applying a binary operator to the current value and each update sent to the channel, useful for computing aggregates over multiple steps; e.g.,`total = BinaryOperatorAggregate(int, operator.add)`
  :::

:::js

- @[LastValue]: The default channel, stores the last value sent to the channel, useful for input and output values, or for sending data from one step to the next.
- @[Topic]: A configurable PubSub Topic, useful for sending multiple values between **actors**, or for accumulating output. Can be configured to deduplicate values or to accumulate values over the course of multiple steps.
- @[BinaryOperatorAggregate]: stores a persistent value, updated by applying a binary operator to the current value and each update sent to the channel, useful for computing aggregates over multiple steps; e.g.,`total = BinaryOperatorAggregate(int, operator.add)`
  :::

## Examples

:::python
While most users will interact with Pregel through the @[StateGraph][StateGraph] API or the @[entrypoint][entrypoint] decorator, it is possible to interact with Pregel directly.
:::

:::js
While most users will interact with Pregel through the @[StateGraph] API or the @[entrypoint] decorator, it is possible to interact with Pregel directly.
:::

Below are a few different examples to give you a sense of the Pregel API.

=== "Single node"

    :::python
    ```python
    from langgraph.channels import EphemeralValue
    from langgraph.pregel import Pregel, NodeBuilder

    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b")
    )

    app = Pregel(
        nodes={"node1": node1},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
        },
        input_channels=["a"],
        output_channels=["b"],
    )

    app.invoke({"a": "foo"})
    ```

    ```con
    {'b': 'foofoo'}
    ```
    :::

    :::js
    ```typescript
    import { EphemeralValue } from "@langchain/langgraph/channels";
    import { Pregel, NodeBuilder } from "@langchain/langgraph/pregel";

    const node1 = new NodeBuilder()
      .subscribeOnly("a")
      .do((x: string) => x + x)
      .writeTo("b");

    const app = new Pregel({
      nodes: { node1 },
      channels: {
        a: new EphemeralValue<string>(),
        b: new EphemeralValue<string>(),
      },
      inputChannels: ["a"],
      outputChannels: ["b"],
    });

    await app.invoke({ a: "foo" });
    ```

    ```console
    { b: 'foofoo' }
    ```
    :::

=== "Multiple nodes"

    :::python
    ```python
    from langgraph.channels import LastValue, EphemeralValue
    from langgraph.pregel import Pregel, NodeBuilder

    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b")
    )

    node2 = (
        NodeBuilder().subscribe_only("b")
        .do(lambda x: x + x)
        .write_to("c")
    )


    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": LastValue(str),
            "c": EphemeralValue(str),
        },
        input_channels=["a"],
        output_channels=["b", "c"],
    )

    app.invoke({"a": "foo"})
    ```

    ```con
    {'b': 'foofoo', 'c': 'foofoofoofoo'}
    ```
    :::

    :::js
    ```typescript
    import { LastValue, EphemeralValue } from "@langchain/langgraph/channels";
    import { Pregel, NodeBuilder } from "@langchain/langgraph/pregel";

    const node1 = new NodeBuilder()
      .subscribeOnly("a")
      .do((x: string) => x + x)
      .writeTo("b");

    const node2 = new NodeBuilder()
      .subscribeOnly("b")
      .do((x: string) => x + x)
      .writeTo("c");

    const app = new Pregel({
      nodes: { node1, node2 },
      channels: {
        a: new EphemeralValue<string>(),
        b: new LastValue<string>(),
        c: new EphemeralValue<string>(),
      },
      inputChannels: ["a"],
      outputChannels: ["b", "c"],
    });

    await app.invoke({ a: "foo" });
    ```

    ```console
    { b: 'foofoo', c: 'foofoofoofoo' }
    ```
    :::

=== "Topic"

    :::python
    ```python
    from langgraph.channels import EphemeralValue, Topic
    from langgraph.pregel import Pregel, NodeBuilder

    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b", "c")
    )

    node2 = (
        NodeBuilder().subscribe_to("b")
        .do(lambda x: x["b"] + x["b"])
        .write_to("c")
    )

    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
            "c": Topic(str, accumulate=True),
        },
        input_channels=["a"],
        output_channels=["c"],
    )

    app.invoke({"a": "foo"})
    ```

    ```pycon
    {'c': ['foofoo', 'foofoofoofoo']}
    ```
    :::

    :::js
    ```typescript
    import { EphemeralValue, Topic } from "@langchain/langgraph/channels";
    import { Pregel, NodeBuilder } from "@langchain/langgraph/pregel";

    const node1 = new NodeBuilder()
      .subscribeOnly("a")
      .do((x: string) => x + x)
      .writeTo("b", "c");

    const node2 = new NodeBuilder()
      .subscribeTo("b")
      .do((x: { b: string }) => x.b + x.b)
      .writeTo("c");

    const app = new Pregel({
      nodes: { node1, node2 },
      channels: {
        a: new EphemeralValue<string>(),
        b: new EphemeralValue<string>(),
        c: new Topic<string>({ accumulate: true }),
      },
      inputChannels: ["a"],
      outputChannels: ["c"],
    });

    await app.invoke({ a: "foo" });
    ```

    ```console
    { c: ['foofoo', 'foofoofoofoo'] }
    ```
    :::

=== "BinaryOperatorAggregate"

    This examples demonstrates how to use the BinaryOperatorAggregate channel to implement a reducer.

    :::python
    ```python
    from langgraph.channels import EphemeralValue, BinaryOperatorAggregate
    from langgraph.pregel import Pregel, NodeBuilder


    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b", "c")
    )

    node2 = (
        NodeBuilder().subscribe_only("b")
        .do(lambda x: x + x)
        .write_to("c")
    )

    def reducer(current, update):
        if current:
            return current + " | " + update
        else:
            return update

    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
            "c": BinaryOperatorAggregate(str, operator=reducer),
        },
        input_channels=["a"],
        output_channels=["c"],
    )

    app.invoke({"a": "foo"})
    ```
    :::

    :::js
    ```typescript
    import { EphemeralValue, BinaryOperatorAggregate } from "@langchain/langgraph/channels";
    import { Pregel, NodeBuilder } from "@langchain/langgraph/pregel";

    const node1 = new NodeBuilder()
      .subscribeOnly("a")
      .do((x: string) => x + x)
      .writeTo("b", "c");

    const node2 = new NodeBuilder()
      .subscribeOnly("b")
      .do((x: string) => x + x)
      .writeTo("c");

    const reducer = (current: string, update: string) => {
      if (current) {
        return current + " | " + update;
      } else {
        return update;
      }
    };

    const app = new Pregel({
      nodes: { node1, node2 },
      channels: {
        a: new EphemeralValue<string>(),
        b: new EphemeralValue<string>(),
        c: new BinaryOperatorAggregate<string>({ operator: reducer }),
      },
      inputChannels: ["a"],
      outputChannels: ["c"],
    });

    await app.invoke({ a: "foo" });
    ```
    :::

=== "Cycle"

    :::python

    This example demonstrates how to introduce a cycle in the graph, by having
    a chain write to a channel it subscribes to. Execution will continue
    until a `None` value is written to the channel.

    ```python
    from langgraph.channels import EphemeralValue
    from langgraph.pregel import Pregel, NodeBuilder, ChannelWriteEntry

    example_node = (
        NodeBuilder().subscribe_only("value")
        .do(lambda x: x + x if len(x) < 10 else None)
        .write_to(ChannelWriteEntry("value", skip_none=True))
    )

    app = Pregel(
        nodes={"example_node": example_node},
        channels={
            "value": EphemeralValue(str),
        },
        input_channels=["value"],
        output_channels=["value"],
    )

    app.invoke({"value": "a"})
    ```

    ```pycon
    {'value': 'aaaaaaaaaaaaaaaa'}
    ```
    :::

    :::js

    This example demonstrates how to introduce a cycle in the graph, by having
    a chain write to a channel it subscribes to. Execution will continue
    until a `null` value is written to the channel.

    ```typescript
    import { EphemeralValue } from "@langchain/langgraph/channels";
    import { Pregel, NodeBuilder, ChannelWriteEntry } from "@langchain/langgraph/pregel";

    const exampleNode = new NodeBuilder()
      .subscribeOnly("value")
      .do((x: string) => x.length < 10 ? x + x : null)
      .writeTo(new ChannelWriteEntry("value", { skipNone: true }));

    const app = new Pregel({
      nodes: { exampleNode },
      channels: {
        value: new EphemeralValue<string>(),
      },
      inputChannels: ["value"],
      outputChannels: ["value"],
    });

    await app.invoke({ value: "a" });
    ```

    ```console
    { value: 'aaaaaaaaaaaaaaaa' }
    ```
    :::

## High-level API

LangGraph provides two high-level APIs for creating a Pregel application: the [StateGraph (Graph API)](./low_level.md) and the [Functional API](functional_api.md).

=== "StateGraph (Graph API)"

    :::python

    The @[StateGraph (Graph API)][StateGraph] is a higher-level abstraction that simplifies the creation of Pregel applications. It allows you to define a graph of nodes and edges. When you compile the graph, the StateGraph API automatically creates the Pregel application for you.

    ```python
    from typing import TypedDict, Optional

    from langgraph.constants import START
    from langgraph.graph import StateGraph

    class Essay(TypedDict):
        topic: str
        content: Optional[str]
        score: Optional[float]

    def write_essay(essay: Essay):
        return {
            "content": f"Essay about {essay['topic']}",
        }

    def score_essay(essay: Essay):
        return {
            "score": 10
        }

    builder = StateGraph(Essay)
    builder.add_node(write_essay)
    builder.add_node(score_essay)
    builder.add_edge(START, "write_essay")

    # Compile the graph.
    # This will return a Pregel instance.
    graph = builder.compile()
    ```
    :::

    :::js

    The @[StateGraph (Graph API)][StateGraph] is a higher-level abstraction that simplifies the creation of Pregel applications. It allows you to define a graph of nodes and edges. When you compile the graph, the StateGraph API automatically creates the Pregel application for you.

    ```typescript
    import { START, StateGraph } from "@langchain/langgraph";

    interface Essay {
      topic: string;
      content?: string;
      score?: number;
    }

    const writeEssay = (essay: Essay) => {
      return {
        content: `Essay about ${essay.topic}`,
      };
    };

    const scoreEssay = (essay: Essay) => {
      return {
        score: 10
      };
    };

    const builder = new StateGraph<Essay>({
      channels: {
        topic: null,
        content: null,
        score: null,
      }
    })
      .addNode("writeEssay", writeEssay)
      .addNode("scoreEssay", scoreEssay)
      .addEdge(START, "writeEssay");

    // Compile the graph.
    // This will return a Pregel instance.
    const graph = builder.compile();
    ```
    :::

    The compiled Pregel instance will be associated with a list of nodes and channels. You can inspect the nodes and channels by printing them.

    :::python
    ```python
    print(graph.nodes)
    ```

    You will see something like this:

    ```pycon
    {'__start__': <langgraph.pregel.read.PregelNode at 0x7d05e3ba1810>,
     'write_essay': <langgraph.pregel.read.PregelNode at 0x7d05e3ba14d0>,
     'score_essay': <langgraph.pregel.read.PregelNode at 0x7d05e3ba1710>}
    ```

    ```python
    print(graph.channels)
    ```

    You should see something like this

    ```pycon
    {'topic': <langgraph.channels.last_value.LastValue at 0x7d05e3294d80>,
     'content': <langgraph.channels.last_value.LastValue at 0x7d05e3295040>,
     'score': <langgraph.channels.last_value.LastValue at 0x7d05e3295980>,
     '__start__': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e3297e00>,
     'write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e32960c0>,
     'score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8ab80>,
     'branch:__start__:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e32941c0>,
     'branch:__start__:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d88800>,
     'branch:write_essay:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e3295ec0>,
     'branch:write_essay:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8ac00>,
     'branch:score_essay:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d89700>,
     'branch:score_essay:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8b400>,
     'start:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8b280>}
    ```
    :::

    :::js
    ```typescript
    console.log(graph.nodes);
    ```

    You will see something like this:

    ```console
    {
      __start__: PregelNode { ... },
      writeEssay: PregelNode { ... },
      scoreEssay: PregelNode { ... }
    }
    ```

    ```typescript
    console.log(graph.channels);
    ```

    You should see something like this

    ```console
    {
      topic: LastValue { ... },
      content: LastValue { ... },
      score: LastValue { ... },
      __start__: EphemeralValue { ... },
      writeEssay: EphemeralValue { ... },
      scoreEssay: EphemeralValue { ... },
      'branch:__start__:__self__:writeEssay': EphemeralValue { ... },
      'branch:__start__:__self__:scoreEssay': EphemeralValue { ... },
      'branch:writeEssay:__self__:writeEssay': EphemeralValue { ... },
      'branch:writeEssay:__self__:scoreEssay': EphemeralValue { ... },
      'branch:scoreEssay:__self__:writeEssay': EphemeralValue { ... },
      'branch:scoreEssay:__self__:scoreEssay': EphemeralValue { ... },
      'start:writeEssay': EphemeralValue { ... }
    }
    ```
    :::

=== "Functional API"

    :::python

    In the [Functional API](functional_api.md), you can use an @[`entrypoint`][entrypoint] to create a Pregel application. The `entrypoint` decorator allows you to define a function that takes input and returns output.

    ```python
    from typing import TypedDict, Optional

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.func import entrypoint

    class Essay(TypedDict):
        topic: str
        content: Optional[str]
        score: Optional[float]


    checkpointer = InMemorySaver()

    @entrypoint(checkpointer=checkpointer)
    def write_essay(essay: Essay):
        return {
            "content": f"Essay about {essay['topic']}",
        }

    print("Nodes: ")
    print(write_essay.nodes)
    print("Channels: ")
    print(write_essay.channels)
    ```

    ```pycon
    Nodes:
    {'write_essay': <langgraph.pregel.read.PregelNode object at 0x7d05e2f9aad0>}
    Channels:
    {'__start__': <langgraph.channels.ephemeral_value.EphemeralValue object at 0x7d05e2c906c0>, '__end__': <langgraph.channels.last_value.LastValue object at 0x7d05e2c90c40>, '__previous__': <langgraph.channels.last_value.LastValue object at 0x7d05e1007280>}
    ```
    :::

    :::js

    In the [Functional API](functional_api.md), you can use an @[`entrypoint`][entrypoint] to create a Pregel application. The `entrypoint` decorator allows you to define a function that takes input and returns output.

    ```typescript
    import { MemorySaver } from "@langchain/langgraph";
    import { entrypoint } from "@langchain/langgraph/func";

    interface Essay {
      topic: string;
      content?: string;
      score?: number;
    }

    const checkpointer = new MemorySaver();

    const writeEssay = entrypoint(
      { checkpointer, name: "writeEssay" },
      async (essay: Essay) => {
        return {
          content: `Essay about ${essay.topic}`,
        };
      }
    );

    console.log("Nodes: ");
    console.log(writeEssay.nodes);
    console.log("Channels: ");
    console.log(writeEssay.channels);
    ```

    ```console
    Nodes:
    { writeEssay: PregelNode { ... } }
    Channels:
    {
      __start__: EphemeralValue { ... },
      __end__: LastValue { ... },
      __previous__: LastValue { ... }
    }
    ```
    :::

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/scalability_and_resilience.md
```md
---
search:
  boost: 2
---

# Scalability & Resilience

LangGraph Platform is designed to scale horizontally with your workload. Each instance of the service is stateless, and keeps no resources in memory. The service is designed to gracefully handle new instances being added or removed, including hard shutdown cases.

## Server scalability

As you add more instances to a service, they will share the HTTP load as long as an appropriate load balancer mechanism is placed in front of them. In most deployment modalities we configure a load balancer for the service automatically. In the “self-hosted without control plane” modality it’s your responsibility to add a load balancer. Since the instances are stateless any load balancing strategy will work, no session stickiness is needed, or recommended. Any instance of the server can communicate with any queue instance (through Redis PubSub), meaning that requests to cancel or stream an in-progress run can be handled by any arbitrary instance.

## Queue scalability

As you add more instances to a service, they will increase run throughput linearly, as each instance is configured to handle a set number of concurrent runs (by default 10). Each attempt for each run will be handled by a single instance, with exactly-once semantics enforced through Postgres’s MVCC model (refer to section below for crash resilience details). Attempts that fail due to transient database errors are retried up to 3 times. We do not make use of long-lived transactions or locks, this enables us to make more efficient use of Postgres resources.

## Resilience

While a run is being handled by a queue instance, a periodic heartbeat timestamp will be recorded in Redis by that queue worker.

When a graceful shutdown request is received (SIGINT) an instance enters shutdown mode, which

- stops accepting new HTTP requests
- gives any in-progress runs a limited number of seconds to finish (if not finished it will be put back in the queue)
- stops the instance from picking up more runs from the queue

If a hard shutdown occurs due to a server crash or an infrastructure failure, any runs that were in progress will be picked up by an internal sweeper task that looks for in-progress runs that have breached their heartbeat window. The sweeper runs every 2 minutes and will put the runs back in the queue for another instance to pick them up.

## Postgres resilience

For deployment modalities where we manage the Postgres database, we have periodic backups and continuously replicated standby replicas for automatic failover. This Postgres configuration is available in the [Cloud SaaS deployment option](../concepts/langgraph_cloud.md) for [`Production` deployment types](../concepts/langgraph_control_plane.md#deployment-types) only.

All communication with Postgres implements retries for retry-able errors. If Postgres is momentarily unavailable, such as during a database restart, most/all traffic should continue to succeed. Prolonged failure of Postgres will render the LangGraph Server unavailable.

## Redis resilience

All data that requires durable storage is stored in Postgres, not Redis. Redis is used only for ephemeral metadata, and communication between instances. Therefore we place no durability requirements on Redis.

All communication with Redis implements retries for retry-able errors. If Redis is momentarily unavailable, such as during a database restart, most/all traffic should continue to succeed. Prolonged failure of Redis will render the LangGraph Server unavailable.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/sdk.md
```md
---
search:
  boost: 2
---

# LangGraph SDK

:::python
LangGraph Platform provides a python SDK for interacting with [LangGraph Server](./langgraph_server.md).

!!! tip "Python SDK reference"

    For detailed information about the Python SDK, see [Python SDK reference docs](../cloud/reference/sdk/python_sdk_ref.md).

## Installation

You can install the LangGraph SDK using the following command:

```bash
pip install langgraph-sdk
```

## Python sync vs. async

The Python SDK provides both synchronous (`get_sync_client`) and asynchronous (`get_client`) clients for interacting with LangGraph Server:

=== "Sync"

    ```python
    from langgraph_sdk import get_sync_client

    client = get_sync_client(url=..., api_key=...)
    client.assistants.search()
    ```

=== "Async"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=..., api_key=...)
    await client.assistants.search()
    ```

## Learn more

- [Python SDK Reference](../cloud/reference/sdk/python_sdk_ref.md)
- [LangGraph CLI API Reference](../cloud/reference/cli.md)
  :::

:::js
LangGraph Platform provides a JS/TS SDK for interacting with [LangGraph Server](./langgraph_server.md).

## Installation

You can add the LangGraph SDK to your project using the following command:

```bash
npm install @langchain/langgraph-sdk
```

## Learn more

- [LangGraph CLI API Reference](../cloud/reference/cli.md)
  :::

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/server-mcp.md
```md
---
tags:
  - mcp
  - platform
hide:
  - tags
---

# MCP endpoint in LangGraph Server

The [Model Context Protocol (MCP)](./mcp.md) is an open protocol for describing tools and data sources in a model-agnostic format, enabling LLMs to discover and use them via a structured API.

[LangGraph Server](./langgraph_server.md) implements MCP using the [Streamable HTTP transport](https://spec.modelcontextprotocol.io/specification/2025-03-26/basic/transports/#streamable-http). This allows LangGraph **agents** to be exposed as **MCP tools**, making them usable with any MCP-compliant client supporting Streamable HTTP.

The MCP endpoint is available at `/mcp` on [LangGraph Server](./langgraph_server.md).

## Requirements

:::python
To use MCP, ensure you have the following dependencies installed:

- `langgraph-api >= 0.2.3`
- `langgraph-sdk >= 0.1.61`

Install them with:

```bash
pip install "langgraph-api>=0.2.3" "langgraph-sdk>=0.1.61"
```

:::

:::js
To use MCP, ensure you have both the api and sdk packages installed.

```bash
npm install @langchain/langgraph-api @langchain/langgraph-sdk
```

:::

## Exposing an agent as MCP tool

When deployed, your agent will appear as a tool in the MCP endpoint
with this configuration:

- **Tool name**: The agent's name.
- **Tool description**: The agent's description.
- **Tool input schema**: The agent's input schema.

### Setting name and description

You can set the name and description of your agent in `langgraph.json`:

:::python

```json
{
  "graphs": {
    "my_agent": {
      "path": "./my_agent/agent.py:graph",
      "description": "A description of what the agent does"
    }
  },
  "env": ".env"
}
```

:::
:::js

```json
{
  "graphs": {
    "my_agent": {
      "path": "./my_agent/agent.ts:graph",
      "description": "A description of what the agent does"
    }
  },
  "env": ".env"
}
```

:::

After deployment, you can update the name and description using the LangGraph SDK.

### Schema

Define clear, minimal input and output schemas to avoid exposing unnecessary internal complexity to the LLM.

:::python
The default [MessagesState](./low_level.md#messagesstate) uses `AnyMessage`, which supports many message types but is too general for direct LLM exposure.
:::

Instead, define **custom agents or workflows** that use explicitly typed input and output structures.

For example, a workflow answering documentation questions might look like this:

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# Define input schema
class InputState(TypedDict):
    question: str

# Define output schema
class OutputState(TypedDict):
    answer: str

# Combine input and output
class OverallState(InputState, OutputState):
    pass

# Define the processing node
def answer_node(state: InputState):
    # Replace with actual logic and do something useful
    return {"answer": "bye", "question": state["question"]}

# Build the graph with explicit schemas
builder = StateGraph(OverallState, input_schema=InputState, output_schema=OutputState)
builder.add_node(answer_node)
builder.add_edge(START, "answer_node")
builder.add_edge("answer_node", END)
graph = builder.compile()

# Run the graph
print(graph.invoke({"question": "hi"}))
```

For more details, see the [low-level concepts guide](https://langchain-ai.github.io/langgraph/concepts/low_level/#state).

## Usage overview

To enable MCP:

- Upgrade to use langgraph-api>=0.2.3. If you are deploying LangGraph Platform, this will be done for you automatically if you create a new revision.
- MCP tools (agents) will be automatically exposed.
- Connect with any MCP-compliant client that supports Streamable HTTP.

### Client

:::python
Use an MCP-compliant client to connect to the LangGraph server. The following example shows how to connect using [langchain-mcp-adapters](https://github.com/langchain-ai/langchain-mcp-adapters).

Install the adapter with:

```bash
pip install langchain-mcp-adapters
```

Here is an example of how to connect to a remote MCP endpoint and use an agent as a tool:

```python
# Create server parameters for stdio connection
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client
import asyncio

from langchain_mcp_adapters.tools import load_mcp_tools
from langgraph.prebuilt import create_react_agent

server_params = {
    "url": "https://mcp-finance-agent.xxx.us.langgraph.app/mcp",
    "headers": {
        "X-Api-Key":"lsv2_pt_your_api_key"
    }
}

async def main():
    async with streamablehttp_client(**server_params) as (read, write, _):
        async with ClientSession(read, write) as session:
            # Initialize the connection
            await session.initialize()

            # Load the remote graph as if it was a tool
            tools = await load_mcp_tools(session)

            # Create and run a react agent with the tools
            agent = create_react_agent("openai:gpt-4.1", tools)

            # Invoke the agent with a message
            agent_response = await agent.ainvoke({"messages": "What can the finance agent do for me?"})
            print(agent_response)

if __name__ == "__main__":
    asyncio.run(main())
```

:::

:::js
Use an MCP-compliant client to connect to the LangGraph server. The following example shows how to connect using [`@langchain/mcp-adapters`](https://npmjs.com/package/@langchain/mcp-adapters).

```bash
npm install @langchain/mcp-adapters
```

Here is an example of how to connect to a remote MCP endpoint and use an agent as a tool:

```typescript
import { MultiServerMCPClient } from "@langchain/mcp-adapters";
import { createReactAgent } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";

async function main() {
  const client = new MultiServerMCPClient({
    mcpServers: {
      "finance-agent": {
        url: "https://mcp-finance-agent.xxx.us.langgraph.app/mcp",
        headers: {
          "X-Api-Key": "lsv2_pt_your_api_key",
        },
      },
    },
  });

  const tools = await client.getTools();

  const model = new ChatOpenAI({
    model: "gpt-4o-mini",
    temperature: 0,
  });

  const agent = createReactAgent({
    model,
    tools,
  });

  const response = await agent.invoke({
    input: "What can the finance agent do for me?",
  });

  console.log(response);
}

main();
```

:::

## Session behavior

The current LangGraph MCP implementation does not support sessions. Each `/mcp` request is stateless and independent.

## Authentication

The `/mcp` endpoint uses the same authentication as the rest of the LangGraph API. Refer to the [authentication guide](./auth.md) for setup details.

## Disable MCP

To disable the MCP endpoint, set `disable_mcp` to `true` in your `langgraph.json` configuration file:

```json
{
  "http": {
    "disable_mcp": true
  }
}
```

This will prevent the server from exposing the `/mcp` endpoint.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/streaming.md
```md
---
search:
boost: 2
---

# Streaming

LangGraph implements a streaming system to surface real-time updates, allowing for responsive and transparent user experiences.

LangGraph’s streaming system lets you surface live feedback from graph runs to your app.  
There are three main categories of data you can stream:

1. **Workflow progress** — get state updates after each graph node is executed.
2. **LLM tokens** — stream language model tokens as they’re generated.
3. **Custom updates** — emit user-defined signals (e.g., “Fetched 10/100 records”).

## What’s possible with LangGraph streaming

- [**Stream LLM tokens**](../how-tos/streaming.md#messages) — capture token streams from anywhere: inside nodes, subgraphs, or tools.
- [**Emit progress notifications from tools**](../how-tos/streaming.md#stream-custom-data) — send custom updates or progress signals directly from tool functions.
- [**Stream from subgraphs**](../how-tos/streaming.md#stream-subgraph-outputs) — include outputs from both the parent graph and any nested subgraphs.
- [**Use any LLM**](../how-tos/streaming.md#use-with-any-llm) — stream tokens from any LLM, even if it's not a LangChain model using the `custom` streaming mode.
- [**Use multiple streaming modes**](../how-tos/streaming.md#stream-multiple-modes) — choose from `values` (full state), `updates` (state deltas), `messages` (LLM tokens + metadata), `custom` (arbitrary user data), or `debug` (detailed traces).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/subgraphs.md
```md
# Subgraphs

A subgraph is a [graph](./low_level.md#graphs) that is used as a [node](./low_level.md#nodes) in another graph — this is the concept of encapsulation applied to LangGraph. Subgraphs allow you to build complex systems with multiple components that are themselves graphs.

![Subgraph](./img/subgraph.png)

Some reasons for using subgraphs are:

- building [multi-agent systems](./multi_agent.md)
- when you want to reuse a set of nodes in multiple graphs
- when you want different teams to work on different parts of the graph independently, you can define each part as a subgraph, and as long as the subgraph interface (the input and output schemas) is respected, the parent graph can be built without knowing any details of the subgraph

The main question when adding subgraphs is how the parent graph and subgraph communicate, i.e. how they pass the [state](./low_level.md#state) between each other during the graph execution. There are two scenarios:

- parent and subgraph have **shared state keys** in their state [schemas](./low_level.md#state). In this case, you can [include the subgraph as a node in the parent graph](../how-tos/subgraph.ipynb#shared-state-schemas)

  :::python

  ```python
  from langgraph.graph import StateGraph, MessagesState, START

  # Subgraph

  def call_model(state: MessagesState):
      response = model.invoke(state["messages"])
      return {"messages": response}

  subgraph_builder = StateGraph(State)
  subgraph_builder.add_node(call_model)
  ...
  # highlight-next-line
  subgraph = subgraph_builder.compile()

  # Parent graph

  builder = StateGraph(State)
  # highlight-next-line
  builder.add_node("subgraph_node", subgraph)
  builder.add_edge(START, "subgraph_node")
  graph = builder.compile()
  ...
  graph.invoke({"messages": [{"role": "user", "content": "hi!"}]})
  ```

  :::

  :::js

  ```typescript
  import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";

  // Subgraph

  const subgraphBuilder = new StateGraph(MessagesZodState).addNode(
    "callModel",
    async (state) => {
      const response = await model.invoke(state.messages);
      return { messages: response };
    }
  );
  // ... other nodes and edges
  // highlight-next-line
  const subgraph = subgraphBuilder.compile();

  // Parent graph

  const builder = new StateGraph(MessagesZodState)
    // highlight-next-line
    .addNode("subgraphNode", subgraph)
    .addEdge(START, "subgraphNode");
  const graph = builder.compile();
  // ...
  await graph.invoke({ messages: [{ role: "user", content: "hi!" }] });
  ```

  :::

- parent graph and subgraph have **different schemas** (no shared state keys in their state [schemas](./low_level.md#state)). In this case, you have to [call the subgraph from inside a node in the parent graph](../how-tos/subgraph.ipynb#different-state-schemas): this is useful when the parent graph and the subgraph have different state schemas and you need to transform state before or after calling the subgraph

  :::python

  ```python
  from typing_extensions import TypedDict, Annotated
  from langchain_core.messages import AnyMessage
  from langgraph.graph import StateGraph, MessagesState, START
  from langgraph.graph.message import add_messages

  class SubgraphMessagesState(TypedDict):
      # highlight-next-line
      subgraph_messages: Annotated[list[AnyMessage], add_messages]

  # Subgraph

  # highlight-next-line
  def call_model(state: SubgraphMessagesState):
      response = model.invoke(state["subgraph_messages"])
      return {"subgraph_messages": response}

  subgraph_builder = StateGraph(SubgraphMessagesState)
  subgraph_builder.add_node("call_model_from_subgraph", call_model)
  subgraph_builder.add_edge(START, "call_model_from_subgraph")
  ...
  # highlight-next-line
  subgraph = subgraph_builder.compile()

  # Parent graph

  def call_subgraph(state: MessagesState):
      response = subgraph.invoke({"subgraph_messages": state["messages"]})
      return {"messages": response["subgraph_messages"]}

  builder = StateGraph(State)
  # highlight-next-line
  builder.add_node("subgraph_node", call_subgraph)
  builder.add_edge(START, "subgraph_node")
  graph = builder.compile()
  ...
  graph.invoke({"messages": [{"role": "user", "content": "hi!"}]})
  ```

  :::

  :::js

  ```typescript
  import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";
  import { z } from "zod";

  const SubgraphState = z.object({
    // highlight-next-line
    subgraphMessages: MessagesZodState.shape.messages,
  });

  // Subgraph

  const subgraphBuilder = new StateGraph(SubgraphState)
    // highlight-next-line
    .addNode("callModelFromSubgraph", async (state) => {
      const response = await model.invoke(state.subgraphMessages);
      return { subgraphMessages: response };
    })
    .addEdge(START, "callModelFromSubgraph");
  // ...
  // highlight-next-line
  const subgraph = subgraphBuilder.compile();

  // Parent graph

  const builder = new StateGraph(MessagesZodState)
    // highlight-next-line
    .addNode("subgraphNode", async (state) => {
      const response = await subgraph.invoke({
        subgraphMessages: state.messages,
      });
      return { messages: response.subgraphMessages };
    })
    .addEdge(START, "subgraphNode");
  const graph = builder.compile();
  // ...
  await graph.invoke({ messages: [{ role: "user", content: "hi!" }] });
  ```

  :::

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/template_applications.md
```md
---
search:
  boost: 2
---

# Template Applications

Templates are open source reference applications designed to help you get started quickly when building with LangGraph. They provide working examples of common agentic workflows that can be customized to your needs.

You can create an application from a template using the LangGraph CLI.

:::python
!!! info "Requirements"

    - Python >= 3.11
    - [LangGraph CLI](https://langchain-ai.github.io/langgraph/cloud/reference/cli/): Requires langchain-cli[inmem] >= 0.1.58

## Install the LangGraph CLI

```bash
pip install "langgraph-cli[inmem]" --upgrade
```

Or via [`uv`](https://docs.astral.sh/uv/getting-started/installation/) (recommended):

```bash
uvx --from "langgraph-cli[inmem]" langgraph dev --help
```

:::

:::js

```bash
npx @langchain/langgraph-cli --help
```

:::

## Available Templates

:::python
| Template | Description | Link |
| -------- | ----------- | ------ |
| **New LangGraph Project** | A simple, minimal chatbot with memory. | [Repo](https://github.com/langchain-ai/new-langgraph-project) |
| **ReAct Agent** | A simple agent that can be flexibly extended to many tools. | [Repo](https://github.com/langchain-ai/react-agent) |
| **Memory Agent** | A ReAct-style agent with an additional tool to store memories for use across threads. | [Repo](https://github.com/langchain-ai/memory-agent) |
| **Retrieval Agent** | An agent that includes a retrieval-based question-answering system. | [Repo](https://github.com/langchain-ai/retrieval-agent-template) |
| **Data-Enrichment Agent** | An agent that performs web searches and organizes its findings into a structured format. | [Repo](https://github.com/langchain-ai/data-enrichment) |

:::

:::js
| Template | Description | Link |
| -------- | ----------- | ------ |
| **New LangGraph Project** | A simple, minimal chatbot with memory. | [Repo](https://github.com/langchain-ai/new-langgraphjs-project) |
| **ReAct Agent** | A simple agent that can be flexibly extended to many tools. | [Repo](https://github.com/langchain-ai/react-agent-js) |
| **Memory Agent** | A ReAct-style agent with an additional tool to store memories for use across threads. | [Repo](https://github.com/langchain-ai/memory-agent-js) |
| **Retrieval Agent** | An agent that includes a retrieval-based question-answering system. | [Repo](https://github.com/langchain-ai/retrieval-agent-template-js) |
| **Data-Enrichment Agent** | An agent that performs web searches and organizes its findings into a structured format. | [Repo](https://github.com/langchain-ai/data-enrichment-js) |
:::

## 🌱 Create a LangGraph App

To create a new app from a template, use the `langgraph new` command.

:::python

```bash
langgraph new
```

Or via [`uv`](https://docs.astral.sh/uv/getting-started/installation/) (recommended):

```bash
uvx --from "langgraph-cli[inmem]" langgraph new
```

:::

:::js

```bash
npm create langgraph
```

:::

## Next Steps

Review the `README.md` file in the root of your new LangGraph app for more information about the template and how to customize it.

After configuring the app properly and adding your API keys, you can start the app using the LangGraph CLI:

:::python

```bash
langgraph dev
```

Or via [`uv`](https://docs.astral.sh/uv/getting-started/installation/) (recommended):

```bash
uvx --from "langgraph-cli[inmem]" --with-editable . langgraph dev
```

!!! info "Missing Local Package?"

    If you are not using `uv` and run into a "`ModuleNotFoundError`" or "`ImportError`", even after installing the local package (`pip install -e .`), it is likely the case that you need to install the CLI into your local virtual environment to make the CLI "aware" of the local package. You can do this by running `python -m pip install "langgraph-cli[inmem]"` and re-activating your virtual environment before running `langgraph dev`.

:::

:::js

```bash
npx @langchain/langgraph-cli dev
```

:::

See the following guides for more information on how to deploy your app:

- **[Launch Local LangGraph Server](../tutorials/langgraph-platform/local-server.md)**: This quick start guide shows how to start a LangGraph Server locally for the **ReAct Agent** template. The steps are similar for other templates.
- **[Deploy to LangGraph Platform](../cloud/quick_start.md)**: Deploy your LangGraph app using LangGraph Platform.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/time-travel.md
```md
---
search:
  boost: 2
---

# Time Travel ⏱️

When working with non-deterministic systems that make model-based decisions (e.g., agents powered by LLMs), it can be useful to examine their decision-making process in detail:

1. 🤔 **Understand reasoning**: Analyze the steps that led to a successful result.
2. 🐞 **Debug mistakes**: Identify where and why errors occurred.
3. 🔍 **Explore alternatives**: Test different paths to uncover better solutions.

LangGraph provides [time travel functionality](../how-tos/human_in_the_loop/time-travel.md) to support these use cases. Specifically, you can resume execution from a prior checkpoint — either replaying the same state or modifying it to explore alternatives. In all cases, resuming past execution produces a new fork in the history.

!!! tip

    For information on how to use time travel, see [Use time travel](../how-tos/human_in_the_loop/time-travel.md) and [Time travel using Server API](../cloud/how-tos/human_in_the_loop_time_travel.md).

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/tools.md
```md
# Tools

Many AI applications interact with users via natural language. However, some use cases require models to interface directly with external systems—such as APIs, databases, or file systems—using structured input. In these scenarios, [tool calling](../how-tos/tool-calling.md) enables models to generate requests that conform to a specified input schema.

:::python
**Tools** encapsulate a callable function and its input schema. These can be passed to compatible [chat models](https://python.langchain.com/docs/concepts/chat_models), allowing the model to decide whether to invoke a tool and with what arguments.
:::

:::js
**Tools** encapsulate a callable function and its input schema. These can be passed to compatible [chat models](https://js.langchain.com/docs/concepts/chat_models), allowing the model to decide whether to invoke a tool and with what arguments.
:::

## Tool calling

![Diagram of a tool call by a model](./img/tool_call.png)

Tool calling is typically **conditional**. Based on the user input and available tools, the model may choose to issue a tool call request. This request is returned in an `AIMessage` object, which includes a `tool_calls` field that specifies the tool name and input arguments:

:::python

```python
llm_with_tools.invoke("What is 2 multiplied by 3?")
# -> AIMessage(tool_calls=[{'name': 'multiply', 'args': {'a': 2, 'b': 3}, ...}])
```

```
AIMessage(
  tool_calls=[
    ToolCall(name="multiply", args={"a": 2, "b": 3}),
    ...
  ]
)
```

:::

:::js

```typescript
await llmWithTools.invoke("What is 2 multiplied by 3?");
```

```
AIMessage {
  tool_calls: [
    ToolCall {
      name: "multiply",
      args: { a: 2, b: 3 },
      ...
    },
    ...
  ]
}
```

:::

If the input is unrelated to any tool, the model returns only a natural language message:

:::python

```python
llm_with_tools.invoke("Hello world!")  # -> AIMessage(content="Hello!")
```

:::

:::js

```typescript
await llmWithTools.invoke("Hello world!"); // { content: "Hello!" }
```

:::

Importantly, the model does not execute the tool—it only generates a request. A separate executor (such as a runtime or agent) is responsible for handling the tool call and returning the result.

See the [tool calling guide](../how-tos/tool-calling.md) for more details.

## Prebuilt tools

LangChain provides prebuilt tool integrations for common external systems including APIs, databases, file systems, and web data.

:::python
Browse the [integrations directory](https://python.langchain.com/docs/integrations/tools/) for available tools.
:::

:::js
Browse the [integrations directory](https://js.langchain.com/docs/integrations/tools/) for available tools.
:::

Common categories:

- **Search**: Bing, SerpAPI, Tavily
- **Code execution**: Python REPL, Node.js REPL
- **Databases**: SQL, MongoDB, Redis
- **Web data**: Scraping and browsing
- **APIs**: OpenWeatherMap, NewsAPI, etc.

## Custom tools

:::python
You can define custom tools using the `@tool` decorator or plain Python functions. For example:

```python
from langchain_core.tools import tool

@tool
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b
```

:::

:::js
You can define custom tools using the `tool` function. For example:

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const multiply = tool(
  (input) => {
    return input.a * input.b;
  },
  {
    name: "multiply",
    description: "Multiply two numbers.",
    schema: z.object({
      a: z.number(),
      b: z.number(),
    }),
  }
);
```

:::

See the [tool calling guide](../how-tos/tool-calling.md) for more details.

## Tool execution

While the model determines when to call a tool, execution of the tool call must be handled by a runtime component.

LangGraph provides prebuilt components for this:

:::python

- @[`ToolNode`][ToolNode]: A prebuilt node that executes tools.
- @[`create_react_agent`][create_react_agent]: Constructs a full agent that manages tool calling automatically.
:::

:::js

- @[ToolNode]: A prebuilt node that executes tools.
- @[`createReactAgent`][create_react_agent]: Constructs a full agent that manages tool calling automatically.
:::

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/tracing.md
```md
# Tracing

Traces are a series of steps that your application takes to go from input to output. Each of these individual steps is represented by a run. You can use [LangSmith](https://smith.langchain.com/) to visualize these execution steps. To use it, [enable tracing for your application](../how-tos/enable-tracing.md). This enables you to do the following:

- [Debug a locally running application](../cloud/how-tos/clone_traces_studio.md).
- [Evaluate the application performance](../agents/evals.md).
- [Monitor the application](https://docs.smith.langchain.com/observability/how_to_guides/dashboards).

To get started, sign up for a free account at [LangSmith](https://smith.langchain.com/).

## Learn more

- [Graph runs in LangSmith](../how-tos/run-id-langsmith.md)
- [LangSmith Observability quickstart](https://docs.smith.langchain.com/observability)
- [Trace with LangGraph](https://docs.smith.langchain.com/observability/how_to_guides/trace_with_langgraph)
- [Tracing conceptual guide](https://docs.smith.langchain.com/observability/concepts#traces)


```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/why-langgraph.md
```md
# Overview

LangGraph is built for developers who want to build powerful, adaptable AI agents. Developers choose LangGraph for:

- **Reliability and controllability.** Steer agent actions with moderation checks and human-in-the-loop approvals. LangGraph persists context for long-running workflows, keeping your agents on course.
- **Low-level and extensible.** Build custom agents with fully descriptive, low-level primitives free from rigid abstractions that limit customization. Design scalable multi-agent systems, with each agent serving a specific role tailored to your use case.
- **First-class streaming support.** With token-by-token streaming and streaming of intermediate steps, LangGraph gives users clear visibility into agent reasoning and actions as they unfold in real time.

## Learn LangGraph basics

To get acquainted with LangGraph's key concepts and features, complete the following LangGraph basics tutorials series:

1. [Build a basic chatbot](../tutorials/get-started/1-build-basic-chatbot.md)
2. [Add tools](../tutorials/get-started/2-add-tools.md)
3. [Add memory](../tutorials/get-started/3-add-memory.md)
4. [Add human-in-the-loop controls](../tutorials/get-started/4-human-in-the-loop.md)
5. [Customize state](../tutorials/get-started/5-customize-state.md)
6. [Time travel](../tutorials/get-started/6-time-travel.md)

In completing this series of tutorials, you will build a support chatbot in LangGraph that can:

* ✅ **Answer common questions** by searching the web
* ✅ **Maintain conversation state** across calls  
* ✅ **Route complex queries** to a human for review  
* ✅ **Use custom state** to control its behavior  
* ✅ **Rewind and explore** alternative conversation paths  

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/img/agent_types.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/img/agent_workflow.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/img/assistants.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/img/breakpoints.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/img/byoc_architecture.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/img/challenge.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/img/double_texting.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/img/langgraph.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/img/langgraph_cloud_architecture.excalidraw
```excalidraw
{
  "type": "excalidraw",
  "version": 2,
  "source": "https://app.excalidraw.com",
  "elements": [
    {
      "type": "rectangle",
      "version": 1732,
      "versionNonce": 275696041,
      "index": "b4w",
      "isDeleted": false,
      "id": "hJT-iHWNgOxBTS-v6NmSA",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 113.39517266127967,
      "y": 190.8320999895953,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "transparent",
      "width": 127.88383573213892,
      "height": 76.53703389977764,
      "seed": 55480532,
      "groupIds": [
        "vvEwxQfV1LZ2VPdanx3xC"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [
        {
          "id": "uqeKYJ6Kvvd6w9OZHyuAQ",
          "type": "arrow"
        }
      ],
      "updated": 1720562263891,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 1335,
      "versionNonce": 957983881,
      "index": "b4y",
      "isDeleted": false,
      "id": "Tik2fyV3DvUyKKUjItR_y",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 0,
      "opacity": 100,
      "angle": 0,
      "x": 117.88320813023455,
      "y": 195.1036697896041,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "#fa5252",
      "width": 5.001953125,
      "height": 5.001953125,
      "seed": 77032404,
      "groupIds": [
        "vvEwxQfV1LZ2VPdanx3xC"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562263891,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 1380,
      "versionNonce": 337982313,
      "index": "b4z",
      "isDeleted": false,
      "id": "4-KStKz4YNm7YxJgvStDr",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 0,
      "opacity": 100,
      "angle": 0,
      "x": 128.14590344273455,
      "y": 195.1036697896041,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "#fab005",
      "width": 5.001953125,
      "height": 5.001953125,
      "seed": 2011353428,
      "groupIds": [
        "vvEwxQfV1LZ2VPdanx3xC"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562263891,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 1438,
      "versionNonce": 1440538185,
      "index": "b50",
      "isDeleted": false,
      "id": "_HzHuZZpsGbx4pgfWrqdG",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 0,
      "opacity": 100,
      "angle": 0,
      "x": 138.85931990908074,
      "y": 195.5543909434503,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "#40c057",
      "width": 5.001953125,
      "height": 5.001953125,
      "seed": 128288468,
      "groupIds": [
        "vvEwxQfV1LZ2VPdanx3xC"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562263891,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 1717,
      "versionNonce": 488385833,
      "index": "b51",
      "isDeleted": false,
      "id": "9BBr2EJfZTdadK4ByuEd1",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 90,
      "angle": 0,
      "x": 154.88638089870796,
      "y": 211.60675982146614,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "#04aaf7",
      "width": 42.72020253937572,
      "height": 42.72020253937572,
      "seed": 1215663188,
      "groupIds": [
        "vvEwxQfV1LZ2VPdanx3xC"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562263891,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 1779,
      "versionNonce": 558800905,
      "index": "b55",
      "isDeleted": false,
      "id": "ql25ZfiIghlA74404nR-R",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 90,
      "angle": 0,
      "x": 169.1146883264293,
      "y": 210.22423019187346,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "transparent",
      "width": 15.528434353116108,
      "height": 44.82230388130942,
      "seed": 699581012,
      "groupIds": [
        "vvEwxQfV1LZ2VPdanx3xC"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [
        {
          "id": "uqeKYJ6Kvvd6w9OZHyuAQ",
          "type": "arrow"
        }
      ],
      "updated": 1720562263891,
      "link": null,
      "locked": false
    },
    {
      "type": "rectangle",
      "version": 2035,
      "versionNonce": 276634153,
      "index": "b5a",
      "isDeleted": false,
      "id": "D3JIlkmsEfkOC-mmRv-iD",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 729.3718643238708,
      "y": 467.40355863727103,
      "strokeColor": "#000000",
      "backgroundColor": "#868e96",
      "width": 125.61758539756245,
      "height": 123.20186260145535,
      "seed": 904330964,
      "groupIds": [
        "5neNifOhXfASMC8FDkUNC",
        "tAnnMEEggyWKb_Nusj5pe"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562091329,
      "link": null,
      "locked": false
    },
    {
      "type": "rectangle",
      "version": 2208,
      "versionNonce": 873065737,
      "index": "b5b",
      "isDeleted": false,
      "id": "rb0EdFdnvkEWMnHZ023fa",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 721.170485431092,
      "y": 459.3471231122537,
      "strokeColor": "#000000",
      "backgroundColor": "#ced4da",
      "width": 125.61758539756245,
      "height": 123.20186260145535,
      "seed": 533202004,
      "groupIds": [
        "5neNifOhXfASMC8FDkUNC",
        "tAnnMEEggyWKb_Nusj5pe"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562091329,
      "link": null,
      "locked": false
    },
    {
      "type": "rectangle",
      "version": 2305,
      "versionNonce": 1864479207,
      "index": "b5c",
      "isDeleted": false,
      "id": "6kqgUFmofb140IbKO0CyJ",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 712.0511318757892,
      "y": 450.3364770827755,
      "strokeColor": "#000000",
      "backgroundColor": "#eeeeee",
      "width": 125.61758539756245,
      "height": 123.20186260145535,
      "seed": 40655316,
      "groupIds": [
        "5neNifOhXfASMC8FDkUNC",
        "tAnnMEEggyWKb_Nusj5pe"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [
        {
          "id": "exADzOCExTS-Bq1vkwRjY",
          "type": "arrow"
        },
        {
          "id": "GRwlh5RbAE7yo1dQav6Vp",
          "type": "arrow"
        },
        {
          "id": "YbBtpu4cnrTXYUrP02_YZ",
          "type": "arrow"
        },
        {
          "id": "ImxR3k_zdYj7JfejUBNDF",
          "type": "arrow"
        }
      ],
      "updated": 1720562100821,
      "link": null,
      "locked": false
    },
    {
      "type": "text",
      "version": 2060,
      "versionNonce": 569742345,
      "index": "b5d",
      "isDeleted": false,
      "id": "lOIooNKCS_TBoRNDfZgJN",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 729.5526487706323,
      "y": 485.73812416908714,
      "strokeColor": "#000000",
      "backgroundColor": "#ced4da",
      "width": 94.21698216343177,
      "height": 46.13739327524151,
      "seed": 149552980,
      "groupIds": [
        "tAnnMEEggyWKb_Nusj5pe"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [
        {
          "id": "YbBtpu4cnrTXYUrP02_YZ",
          "type": "arrow"
        }
      ],
      "updated": 1720562091329,
      "link": null,
      "locked": false,
      "fontSize": 18.454957310096603,
      "fontFamily": 1,
      "text": "LangGraph\nAPI",
      "textAlign": "center",
      "verticalAlign": "top",
      "containerId": null,
      "originalText": "LangGraph\nAPI",
      "autoResize": true,
      "lineHeight": 1.25
    },
    {
      "type": "text",
      "version": 1997,
      "versionNonce": 1535629735,
      "index": "b5z",
      "isDeleted": false,
      "id": "dLlsClCwm2AfX_uVU6tQZ",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 460.9124298095704,
      "y": 566.8833765073015,
      "strokeColor": "#000000",
      "backgroundColor": "transparent",
      "width": 65.29818725585938,
      "height": 18.43911459501469,
      "seed": 1128837484,
      "groupIds": [],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720561631457,
      "link": null,
      "locked": false,
      "fontSize": 14.751291676011752,
      "fontFamily": 1,
      "text": "Public IP",
      "textAlign": "center",
      "verticalAlign": "top",
      "containerId": null,
      "originalText": "Public IP",
      "autoResize": true,
      "lineHeight": 1.25
    },
    {
      "type": "rectangle",
      "version": 3584,
      "versionNonce": 28729127,
      "index": "b5zG",
      "isDeleted": false,
      "id": "dXRSE0bEwHqUShNPc-wE4",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 90,
      "angle": 6.275789871871884,
      "x": 505.87055654031275,
      "y": 485.5584899396523,
      "strokeColor": "#000000",
      "backgroundColor": "#eeeeee",
      "width": 69.69390497296563,
      "height": 70.29012363709795,
      "seed": 2144197996,
      "groupIds": [
        "0y9lSFu7b0xCNAbts8SEy"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [
        {
          "id": "uqeKYJ6Kvvd6w9OZHyuAQ",
          "type": "arrow"
        },
        {
          "id": "_dxg4ciA9Th4HQgp64Lty",
          "type": "arrow"
        },
        {
          "id": "YbBtpu4cnrTXYUrP02_YZ",
          "type": "arrow"
        }
      ],
      "updated": 1720561622176,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 3389,
      "versionNonce": 714646377,
      "index": "b60",
      "isDeleted": false,
      "id": "GtSIdPlfzpdmpSVOsaIsQ",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 523.6501791578276,
      "y": 505.4570133248583,
      "strokeColor": "#000000",
      "backgroundColor": "#868e96",
      "width": 29.390624999999893,
      "height": 32.16406249999993,
      "seed": 1416524396,
      "groupIds": [
        "0y9lSFu7b0xCNAbts8SEy"
      ],
      "frameId": null,
      "roundness": {
        "type": 2
      },
      "boundElements": [
        {
          "id": "uqeKYJ6Kvvd6w9OZHyuAQ",
          "type": "arrow"
        },
        {
          "id": "_dxg4ciA9Th4HQgp64Lty",
          "type": "arrow"
        }
      ],
      "updated": 1720562395007,
      "link": null,
      "locked": false
    },
    {
      "type": "rectangle",
      "version": 477,
      "versionNonce": 1426357543,
      "index": "b62",
      "isDeleted": false,
      "id": "Uh9ZFQgGPS7qCu84blcib",
      "fillStyle": "hachure",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 538.2724609375,
      "y": 165.44585503472217,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "transparent",
      "width": 951.30859375,
      "height": 713.30078125,
      "seed": 978310100,
      "groupIds": [],
      "frameId": null,
      "roundness": null,
      "boundElements": [
        {
          "id": "uqeKYJ6Kvvd6w9OZHyuAQ",
          "type": "arrow"
        },
        {
          "id": "_dxg4ciA9Th4HQgp64Lty",
          "type": "arrow"
        }
      ],
      "updated": 1720562460883,
      "link": null,
      "locked": false
    },
    {
      "type": "text",
      "version": 680,
      "versionNonce": 654253257,
      "index": "b63",
      "isDeleted": false,
      "id": "I7t5MMql-2a4KPJ4P-NZc",
      "fillStyle": "hachure",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 571.3024807884578,
      "y": 198.40908668154765,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "transparent",
      "width": 392.6438293457031,
      "height": 35,
      "seed": 917281260,
      "groupIds": [],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562152692,
      "link": null,
      "locked": false,
      "fontSize": 28,
      "fontFamily": 1,
      "text": "LangGraph Platform Deployment",
      "textAlign": "center",
      "verticalAlign": "top",
      "containerId": null,
      "originalText": "LangGraph Platform Deployment",
      "autoResize": true,
      "lineHeight": 1.25
    },
    {
      "type": "text",
      "version": 618,
      "versionNonce": 1833326985,
      "index": "b64",
      "isDeleted": false,
      "id": "3f9P4tecSo8Zb6I7toQIl",
      "fillStyle": "hachure",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 90.65199855804967,
      "y": 155.27491009424602,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "transparent",
      "width": 173.79986572265625,
      "height": 25,
      "seed": 200224980,
      "groupIds": [],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562269845,
      "link": null,
      "locked": false,
      "fontSize": 20,
      "fontFamily": 1,
      "text": "LangGraph Studio",
      "textAlign": "center",
      "verticalAlign": "top",
      "containerId": null,
      "originalText": "LangGraph Studio",
      "autoResize": true,
      "lineHeight": 1.25
    },
    {
      "type": "rectangle",
      "version": 2322,
      "versionNonce": 1378544551,
      "index": "b65",
      "isDeleted": false,
      "id": "phAGdHKA4dF9rkmGcvy6q",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 701.0313542850588,
      "y": 1084.7585544152464,
      "strokeColor": "#000000",
      "backgroundColor": "#ced4da",
      "width": 165.20091654688943,
      "height": 58.94492796978201,
      "seed": 313335636,
      "groupIds": [
        "oOKAN_Kuyrw0ln-LebimZ",
        "0a0NHNLAqJmP5n2-a1oDq",
        "aGAalId5pY2ko-2u65CHQ"
      ],
      "frameId": null,
      "roundness": {
        "type": 1
      },
      "boundElements": [],
      "updated": 1720562020087,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 2244,
      "versionNonce": 1564270279,
      "index": "b66",
      "isDeleted": false,
      "id": "158rSDD3Iw1pYuwCsAf_f",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 746.7912325773856,
      "y": 1018.057714870496,
      "strokeColor": "#000000",
      "backgroundColor": "#ced4da",
      "width": 111.68512667958727,
      "height": 107.8071708921016,
      "seed": 804656340,
      "groupIds": [
        "oOKAN_Kuyrw0ln-LebimZ",
        "0a0NHNLAqJmP5n2-a1oDq",
        "aGAalId5pY2ko-2u65CHQ"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [
        {
          "id": "GRwlh5RbAE7yo1dQav6Vp",
          "type": "arrow"
        }
      ],
      "updated": 1720562020087,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 2650,
      "versionNonce": 1519411239,
      "index": "b67",
      "isDeleted": false,
      "id": "AJxDZtJg-wDuX-rfC50SK",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 673.8856637726549,
      "y": 1066.9199577928161,
      "strokeColor": "#000000",
      "backgroundColor": "#ced4da",
      "width": 77.55911574971344,
      "height": 75.232342277222,
      "seed": 1938053716,
      "groupIds": [
        "oOKAN_Kuyrw0ln-LebimZ",
        "0a0NHNLAqJmP5n2-a1oDq",
        "aGAalId5pY2ko-2u65CHQ"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562020087,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 2583,
      "versionNonce": 301974343,
      "index": "b68",
      "isDeleted": false,
      "id": "YTnjeziwWQHtTIYmKbw7O",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 715.767586277494,
      "y": 1036.6719026504277,
      "strokeColor": "#000000",
      "backgroundColor": "#ced4da",
      "width": 53.24134991926489,
      "height": 47.85994050340004,
      "seed": 137228244,
      "groupIds": [
        "oOKAN_Kuyrw0ln-LebimZ",
        "0a0NHNLAqJmP5n2-a1oDq",
        "aGAalId5pY2ko-2u65CHQ"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562020087,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 2365,
      "versionNonce": 1742062183,
      "index": "b69",
      "isDeleted": false,
      "id": "BD5jFHFF0eRNm-Ts8WbvX",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 821.2479836971144,
      "y": 1081.6561897852607,
      "strokeColor": "#000000",
      "backgroundColor": "#ced4da",
      "width": 65.92524838725646,
      "height": 62.04729259977069,
      "seed": 1309201748,
      "groupIds": [
        "oOKAN_Kuyrw0ln-LebimZ",
        "0a0NHNLAqJmP5n2-a1oDq",
        "aGAalId5pY2ko-2u65CHQ"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562020087,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 1632,
      "versionNonce": 215932295,
      "index": "b6A",
      "isDeleted": false,
      "id": "u9p-FewX4neDdwGkJ34bz",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 683.7476125381887,
      "y": 1076.4526495387722,
      "strokeColor": "#ced4da",
      "backgroundColor": "#ced4da",
      "width": 68.3086672954735,
      "height": 66.47897085005896,
      "seed": 845875924,
      "groupIds": [
        "0a0NHNLAqJmP5n2-a1oDq",
        "aGAalId5pY2ko-2u65CHQ"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562020087,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 1326,
      "versionNonce": 1683461287,
      "index": "b6B",
      "isDeleted": false,
      "id": "gw9lYoNpan9-pVaEZl-HG",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 775.8423336240497,
      "y": 1050.8368993029696,
      "strokeColor": "#ced4da",
      "backgroundColor": "#ced4da",
      "width": 82.33634004365118,
      "height": 88.43532819503265,
      "seed": 132812884,
      "groupIds": [
        "0a0NHNLAqJmP5n2-a1oDq",
        "aGAalId5pY2ko-2u65CHQ"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562020087,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 1524,
      "versionNonce": 904685511,
      "index": "b6C",
      "isDeleted": false,
      "id": "p8ac8Os3wHyXWhFFYkwtq",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 804.5075779355367,
      "y": 1081.9417388750123,
      "strokeColor": "#ced4da",
      "backgroundColor": "#ced4da",
      "width": 78.67694715282218,
      "height": 60.379982698677395,
      "seed": 1583179220,
      "groupIds": [
        "0a0NHNLAqJmP5n2-a1oDq",
        "aGAalId5pY2ko-2u65CHQ"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562020087,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 1424,
      "versionNonce": 532272871,
      "index": "b6D",
      "isDeleted": false,
      "id": "0CY_N6vUY0JG_06xRQKtP",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 735.5890118249246,
      "y": 1039.2488218153428,
      "strokeColor": "#ced4da",
      "backgroundColor": "#ced4da",
      "width": 65.86907203492086,
      "height": 66.47897085005904,
      "seed": 1524206420,
      "groupIds": [
        "0a0NHNLAqJmP5n2-a1oDq",
        "aGAalId5pY2ko-2u65CHQ"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562020087,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 1377,
      "versionNonce": 1212965383,
      "index": "b6E",
      "isDeleted": false,
      "id": "X3RTSkOQuM9P_WHOcnJgU",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 707.5336663285854,
      "y": 1056.3259886392107,
      "strokeColor": "#ced4da",
      "backgroundColor": "#ced4da",
      "width": 90.87492345558519,
      "height": 82.94623885878921,
      "seed": 263520468,
      "groupIds": [
        "0a0NHNLAqJmP5n2-a1oDq",
        "aGAalId5pY2ko-2u65CHQ"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562020087,
      "link": null,
      "locked": false
    },
    {
      "type": "text",
      "version": 1865,
      "versionNonce": 1352698151,
      "index": "b6F",
      "isDeleted": false,
      "id": "nBnN_oFeoo-IOwN7h4G7L",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 709.4648322159471,
      "y": 1075.6587328291507,
      "strokeColor": "#000000",
      "backgroundColor": "#ced4da",
      "width": 144.66934125859115,
      "height": 45.404379683265596,
      "seed": 1418053204,
      "groupIds": [
        "aGAalId5pY2ko-2u65CHQ"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562020087,
      "link": null,
      "locked": false,
      "fontSize": 18.161751873306237,
      "fontFamily": 1,
      "text": "LangSmith Auth\nService",
      "textAlign": "center",
      "verticalAlign": "top",
      "containerId": null,
      "originalText": "LangSmith Auth Service",
      "autoResize": false,
      "lineHeight": 1.25
    },
    {
      "type": "rectangle",
      "version": 3022,
      "versionNonce": 1337895177,
      "index": "b6Z",
      "isDeleted": false,
      "id": "OAX7xzLor8ovkAfXfqZea",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 337.10710234255197,
      "y": 630.7301375293715,
      "strokeColor": "#000000",
      "backgroundColor": "#eeeeee",
      "width": 103.31005290565567,
      "height": 101.32332111900841,
      "seed": 36673900,
      "groupIds": [
        "mqLvQ6wLJdIAVDE63vzqU",
        "0_5C8FLBLLqaFTaEiRUIN"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562285956,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 2503,
      "versionNonce": 1979097065,
      "index": "b6a",
      "isDeleted": false,
      "id": "7PtpKi0ACyLIsUAOI1yuB",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 378.1680162403772,
      "y": 638.1108336767393,
      "strokeColor": "#495057",
      "backgroundColor": "#868e96",
      "width": 21.98778077452899,
      "height": 21.98778077452899,
      "seed": 1112640492,
      "groupIds": [
        "AcMEjQOzLSSbeqO4P9rXw",
        "mqLvQ6wLJdIAVDE63vzqU",
        "0_5C8FLBLLqaFTaEiRUIN"
      ],
      "frameId": null,
      "roundness": {
        "type": 2
      },
      "boundElements": [],
      "updated": 1720562285956,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 2721,
      "versionNonce": 1137490633,
      "index": "b6b",
      "isDeleted": false,
      "id": "52wlDlAzPni_t4KlYkosc",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 381.7660167307564,
      "y": 641.508945250983,
      "strokeColor": "#495057",
      "backgroundColor": "#ffffff",
      "width": 14.99166870990611,
      "height": 14.99166870990611,
      "seed": 1049701996,
      "groupIds": [
        "AcMEjQOzLSSbeqO4P9rXw",
        "mqLvQ6wLJdIAVDE63vzqU",
        "0_5C8FLBLLqaFTaEiRUIN"
      ],
      "frameId": null,
      "roundness": {
        "type": 2
      },
      "boundElements": [],
      "updated": 1720562285956,
      "link": null,
      "locked": false
    },
    {
      "type": "rectangle",
      "version": 2405,
      "versionNonce": 1776056745,
      "index": "b6c",
      "isDeleted": false,
      "id": "0xedK9k0KDXJsZp2620-v",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 372.97090442094543,
      "y": 653.5022802189081,
      "strokeColor": "#495057",
      "backgroundColor": "#868e96",
      "width": 31.582448748868906,
      "height": 28.184337174623476,
      "seed": 129532140,
      "groupIds": [
        "AcMEjQOzLSSbeqO4P9rXw",
        "mqLvQ6wLJdIAVDE63vzqU",
        "0_5C8FLBLLqaFTaEiRUIN"
      ],
      "frameId": null,
      "roundness": {
        "type": 1
      },
      "boundElements": [],
      "updated": 1720562285956,
      "link": null,
      "locked": false
    },
    {
      "type": "rectangle",
      "version": 2491,
      "versionNonce": 2035375241,
      "index": "b6d",
      "isDeleted": false,
      "id": "f-4nFLLLC5fQbLjbTG_5c",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 0,
      "opacity": 100,
      "angle": 0,
      "x": 377.96812732424786,
      "y": 646.9059459865505,
      "strokeColor": "#868e96",
      "backgroundColor": "#868e96",
      "width": 3.7978894065095457,
      "height": 8.795112309811582,
      "seed": 89228140,
      "groupIds": [
        "AcMEjQOzLSSbeqO4P9rXw",
        "mqLvQ6wLJdIAVDE63vzqU",
        "0_5C8FLBLLqaFTaEiRUIN"
      ],
      "frameId": null,
      "roundness": {
        "type": 1
      },
      "boundElements": [],
      "updated": 1720562285956,
      "link": null,
      "locked": false
    },
    {
      "type": "rectangle",
      "version": 2463,
      "versionNonce": 1374085993,
      "index": "b6e",
      "isDeleted": false,
      "id": "NnkQ05uLEgeb8YO3vTJwC",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 0,
      "opacity": 100,
      "angle": 0,
      "x": 396.5577965245327,
      "y": 646.9059459865505,
      "strokeColor": "#868e96",
      "backgroundColor": "#868e96",
      "width": 3.7978894065095457,
      "height": 10.394223638868224,
      "seed": 497432044,
      "groupIds": [
        "AcMEjQOzLSSbeqO4P9rXw",
        "mqLvQ6wLJdIAVDE63vzqU",
        "0_5C8FLBLLqaFTaEiRUIN"
      ],
      "frameId": null,
      "roundness": {
        "type": 1
      },
      "boundElements": [],
      "updated": 1720562285956,
      "link": null,
      "locked": false
    },
    {
      "type": "rectangle",
      "version": 2415,
      "versionNonce": 1340640841,
      "index": "b6f",
      "isDeleted": false,
      "id": "OWDG579GuwpihpGIA28Tg",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 0,
      "opacity": 100,
      "angle": 0,
      "x": 391.66051807929273,
      "y": 647.8554183381777,
      "strokeColor": "#ffffff",
      "backgroundColor": "#ffffff",
      "width": 4.59744507103787,
      "height": 5.297056277500153,
      "seed": 1572903020,
      "groupIds": [
        "AcMEjQOzLSSbeqO4P9rXw",
        "mqLvQ6wLJdIAVDE63vzqU",
        "0_5C8FLBLLqaFTaEiRUIN"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562285956,
      "link": null,
      "locked": false
    },
    {
      "type": "rectangle",
      "version": 2469,
      "versionNonce": 976346409,
      "index": "b6g",
      "isDeleted": false,
      "id": "M3i-tn7D7iXqgpKtd6MQo",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 0,
      "opacity": 100,
      "angle": 0,
      "x": 382.0658501049553,
      "y": 647.9553627962445,
      "strokeColor": "#ffffff",
      "backgroundColor": "#ffffff",
      "width": 1.5991113290566277,
      "height": 5.297056277500153,
      "seed": 560638700,
      "groupIds": [
        "AcMEjQOzLSSbeqO4P9rXw",
        "mqLvQ6wLJdIAVDE63vzqU",
        "0_5C8FLBLLqaFTaEiRUIN"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562285956,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 2749,
      "versionNonce": 1882083337,
      "index": "b6l",
      "isDeleted": false,
      "id": "T_1Pt_2p1K-v_9am6ECGE",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 379.6565507222151,
      "y": 665.6185202219883,
      "strokeColor": "#000000",
      "backgroundColor": "#ced4da",
      "width": 18.568404421971625,
      "height": 14.319701715249316,
      "seed": 1850103660,
      "groupIds": [
        "AcMEjQOzLSSbeqO4P9rXw",
        "mqLvQ6wLJdIAVDE63vzqU",
        "0_5C8FLBLLqaFTaEiRUIN"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562285956,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 3071,
      "versionNonce": 1473865449,
      "index": "b6m",
      "isDeleted": false,
      "id": "7JlFc-bWpGYPwqGoTtbmt",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 385.6142955866336,
      "y": 657.9377899654851,
      "strokeColor": "#495057",
      "backgroundColor": "#ced4da",
      "width": 6.632995786866073,
      "height": 6.632995786866073,
      "seed": 954239468,
      "groupIds": [
        "AcMEjQOzLSSbeqO4P9rXw",
        "mqLvQ6wLJdIAVDE63vzqU",
        "0_5C8FLBLLqaFTaEiRUIN"
      ],
      "frameId": null,
      "roundness": {
        "type": 2
      },
      "boundElements": [],
      "updated": 1720562285956,
      "link": null,
      "locked": false
    },
    {
      "type": "rectangle",
      "version": 2844,
      "versionNonce": 197042633,
      "index": "b6n",
      "isDeleted": false,
      "id": "gGI7DjTtZFJHEpOrPctLC",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 0,
      "opacity": 100,
      "angle": 0,
      "x": 377.76823840811494,
      "y": 671.4408165237932,
      "strokeColor": "#868e96",
      "backgroundColor": "#868e96",
      "width": 22.187669690661036,
      "height": 9.126842851477583,
      "seed": 1183870060,
      "groupIds": [
        "AcMEjQOzLSSbeqO4P9rXw",
        "mqLvQ6wLJdIAVDE63vzqU",
        "0_5C8FLBLLqaFTaEiRUIN"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562285956,
      "link": null,
      "locked": false
    },
    {
      "type": "text",
      "version": 2311,
      "versionNonce": 1224310953,
      "index": "b6p",
      "isDeleted": false,
      "id": "N-iD-eJgvviywh5AyntKY",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 356.88623361308805,
      "y": 684.2869829568233,
      "strokeColor": "#000000",
      "backgroundColor": "#868e96",
      "width": 63.85595703125,
      "height": 40,
      "seed": 689521004,
      "groupIds": [
        "0_5C8FLBLLqaFTaEiRUIN"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562285956,
      "link": null,
      "locked": false,
      "fontSize": 16,
      "fontFamily": 1,
      "text": "API Key\nAuth",
      "textAlign": "center",
      "verticalAlign": "top",
      "containerId": null,
      "originalText": "API Key\nAuth",
      "autoResize": true,
      "lineHeight": 1.25
    },
    {
      "type": "text",
      "version": 1760,
      "versionNonce": 1156809767,
      "index": "b6q",
      "isDeleted": false,
      "id": "_3VqBGQn1T1xg3L3OXnUb",
      "fillStyle": "hachure",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 5.830698886507291,
      "x": 286.3273036703816,
      "y": 555.7880781372843,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "transparent",
      "width": 164.88632109466232,
      "height": 20.1980825356615,
      "seed": 1715318764,
      "groupIds": [],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562291267,
      "link": null,
      "locked": false,
      "fontSize": 16.158466028529183,
      "fontFamily": 1,
      "text": "LangGraph API calls",
      "textAlign": "center",
      "verticalAlign": "top",
      "containerId": null,
      "originalText": "LangGraph API calls",
      "autoResize": true,
      "lineHeight": 1.25
    },
    {
      "type": "text",
      "version": 1514,
      "versionNonce": 717329063,
      "index": "b6r",
      "isDeleted": false,
      "id": "dlT6gsQDxHLsVXqHWXavI",
      "fillStyle": "hachure",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0.6572320528128994,
      "x": 224.05859996710944,
      "y": 394.9788603845124,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "transparent",
      "width": 167.35105769368803,
      "height": 20.49701717767715,
      "seed": 814625876,
      "groupIds": [],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562281234,
      "link": null,
      "locked": false,
      "fontSize": 16.39761374214174,
      "fontFamily": 1,
      "text": "LangGraph API calls",
      "textAlign": "center",
      "verticalAlign": "top",
      "containerId": null,
      "originalText": "LangGraph API calls",
      "autoResize": true,
      "lineHeight": 1.25
    },
    {
      "type": "text",
      "version": 680,
      "versionNonce": 1762804871,
      "index": "b6s",
      "isDeleted": false,
      "id": "XBrKnd-0devoBKKTHUbZv",
      "fillStyle": "hachure",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 84.52910596636131,
      "y": 583.4949428013393,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "transparent",
      "width": 174.49591064453125,
      "height": 60,
      "seed": 273108588,
      "groupIds": [],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720560939707,
      "link": null,
      "locked": false,
      "fontSize": 16,
      "fontFamily": 1,
      "text": "Client Application\n using LangGraph SDK\n",
      "textAlign": "center",
      "verticalAlign": "top",
      "containerId": null,
      "originalText": "Client Application\n using LangGraph SDK\n",
      "autoResize": true,
      "lineHeight": 1.25
    },
    {
      "type": "text",
      "version": 1396,
      "versionNonce": 398406281,
      "index": "b6u",
      "isDeleted": false,
      "id": "uquS_DsQsMqyTJEUGiF5y",
      "fillStyle": "hachure",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 653.3903818348372,
      "y": 779.5054214719742,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "transparent",
      "width": 126.71083185685349,
      "height": 85.38628472222172,
      "seed": 45973716,
      "groupIds": [],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562325835,
      "link": null,
      "locked": false,
      "fontSize": 17.077256944444322,
      "fontFamily": 1,
      "text": "Each API \nrequest is \nauthenticated\nand authorized",
      "textAlign": "center",
      "verticalAlign": "top",
      "containerId": null,
      "originalText": "Each API request is authenticated and authorized",
      "autoResize": false,
      "lineHeight": 1.25
    },
    {
      "type": "rectangle",
      "version": 2744,
      "versionNonce": 1891848167,
      "index": "b7Zd",
      "isDeleted": false,
      "id": "mLr8h17de1ytpGX4p1EVp",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 905.575985181484,
      "y": 557.0473565430624,
      "strokeColor": "#000000",
      "backgroundColor": "#868e96",
      "width": 96.04882870779743,
      "height": 94.20173584803199,
      "seed": 1856915174,
      "groupIds": [
        "OCE3SeFCdYau8O9CV-PNV",
        "NnwONCWzEe8epkA3Q8Uxa"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562081796,
      "link": null,
      "locked": false
    },
    {
      "type": "rectangle",
      "version": 2920,
      "versionNonce": 197049095,
      "index": "b7Zl",
      "isDeleted": false,
      "id": "oZM58qv5hiR6fIisFeU_u",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 899.3051049225841,
      "y": 550.8873018557446,
      "strokeColor": "#000000",
      "backgroundColor": "#ced4da",
      "width": 96.04882870779743,
      "height": 94.20173584803199,
      "seed": 340818470,
      "groupIds": [
        "OCE3SeFCdYau8O9CV-PNV",
        "NnwONCWzEe8epkA3Q8Uxa"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562081796,
      "link": null,
      "locked": false
    },
    {
      "type": "rectangle",
      "version": 3023,
      "versionNonce": 106938919,
      "index": "b7Zt",
      "isDeleted": false,
      "id": "PXIEaG-wEYWhBqGqWeD5k",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 892.3323293769707,
      "y": 543.9976454888205,
      "strokeColor": "#000000",
      "backgroundColor": "#eeeeee",
      "width": 96.04882870779743,
      "height": 94.20173584803199,
      "seed": 97142118,
      "groupIds": [
        "OCE3SeFCdYau8O9CV-PNV",
        "NnwONCWzEe8epkA3Q8Uxa"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [
        {
          "id": "5UAVBFbYdU9GDrjJfCpVN",
          "type": "arrow"
        }
      ],
      "updated": 1720562081796,
      "link": null,
      "locked": false
    },
    {
      "type": "text",
      "version": 2636,
      "versionNonce": 932282247,
      "index": "b7a",
      "isDeleted": false,
      "id": "lmUoq8BUecgQjLqKl-3QO",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 912.7987374442486,
      "y": 560.8680446628364,
      "strokeColor": "#000000",
      "backgroundColor": "#ced4da",
      "width": 55.56913757324219,
      "height": 37.19209790013146,
      "seed": 342403238,
      "groupIds": [
        "NnwONCWzEe8epkA3Q8Uxa"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720562081796,
      "link": null,
      "locked": false,
      "fontSize": 14.876839160052583,
      "fontFamily": 1,
      "text": "Queue\nWorkers",
      "textAlign": "center",
      "verticalAlign": "top",
      "containerId": null,
      "originalText": "Queue\nWorkers",
      "autoResize": true,
      "lineHeight": 1.25
    },
    {
      "type": "ellipse",
      "version": 5900,
      "versionNonce": 1307837607,
      "index": "b7ph",
      "isDeleted": false,
      "id": "JCX6F31Chm-tWttlh3yep",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 1255.1841824869105,
      "y": 370.17425588898186,
      "strokeColor": "#0a11d3",
      "backgroundColor": "#fff",
      "width": 87.65074610854188,
      "height": 17.72670397681366,
      "seed": 2027853524,
      "groupIds": [
        "w9luoDWKhVKMM96QzOp21",
        "MgkMGR0WNj6mLY8V4qLYe"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720564065349,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 1271,
      "versionNonce": 1052327241,
      "index": "b7pl",
      "isDeleted": false,
      "id": "jDoqgGHvc7g2dB-c2gFva",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 1326.693914787491,
      "y": 394.7500587661154,
      "strokeColor": "#0a11d3",
      "backgroundColor": "#fff",
      "width": 12.846057046979809,
      "height": 13.941904362416096,
      "seed": 1343003732,
      "groupIds": [
        "w9luoDWKhVKMM96QzOp21",
        "MgkMGR0WNj6mLY8V4qLYe"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720564065349,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 1322,
      "versionNonce": 310040519,
      "index": "b7pp",
      "isDeleted": false,
      "id": "o24C6VJHmRl_vjoT-SF4w",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 1326.693914787491,
      "y": 425.3525126251759,
      "strokeColor": "#0a11d3",
      "backgroundColor": "#fff",
      "width": 12.846057046979809,
      "height": 13.941904362416096,
      "seed": 1851650516,
      "groupIds": [
        "w9luoDWKhVKMM96QzOp21",
        "MgkMGR0WNj6mLY8V4qLYe"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720564065349,
      "link": null,
      "locked": false
    },
    {
      "type": "ellipse",
      "version": 1374,
      "versionNonce": 152349737,
      "index": "b7pt",
      "isDeleted": false,
      "id": "0pWke8i6NnjZe5vrKmy3w",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 1326.693914787491,
      "y": 458.6133137996726,
      "strokeColor": "#0a11d3",
      "backgroundColor": "#fff",
      "width": 12.846057046979809,
      "height": 13.941904362416096,
      "seed": 854103892,
      "groupIds": [
        "w9luoDWKhVKMM96QzOp21",
        "MgkMGR0WNj6mLY8V4qLYe"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1720564065349,
      "link": null,
      "locked": false
    },
    {
      "type": "text",
      "version": 1135,
      "versionNonce": 1090143975,
      "index": "b7rO",
      "isDeleted": false,
      "id": "T3nQ0dJaBXZ2gdxhVJgDp",
      "fillStyle": "hachure",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 1236.9355666248207,
      "y": 495.7539450224738,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "transparent",
      "width": 127.01988220214844,
      "height": 25,
      "seed": 1028469356,
      "groupIds": [
        "MgkMGR0WNj6mLY8V4qLYe"
      ],
      "frameId": null,
      "roundness": null,
      "boundElements": [
        {
          "id": "exADzOCExTS-Bq1vkwRjY",
          "type": "arrow"
        },
        {
          "id": "5UAVBFbYdU9GDrjJfCpVN",
          "type": "arrow"
        }
      ],
      "updated": 1720564065349,
      "link": null,
      "locked": false,
      "fontSize": 20,
      "fontFamily": 1,
      "text": "Postgres DB",
      "textAlign": "left",
      "verticalAlign": "top",
      "containerId": null,
      "originalText": "Postgres DB",
      "autoResize": true,
      "lineHeight": 1.25
    },
    {
      "type": "line",
      "version": 1525,
      "versionNonce": 2084294825,
      "index": "bB8",
      "isDeleted": false,
      "id": "lwMAUw6adAmCd08onBv2N",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 112.31957491038446,
      "y": 201.54863438183958,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "transparent",
      "width": 128.84193229844433,
      "height": 0,
      "seed": 1047352916,
      "groupIds": [
        "vvEwxQfV1LZ2VPdanx3xC"
      ],
      "frameId": null,
      "roundness": {
        "type": 2
      },
      "boundElements": [],
      "updated": 1720562263891,
      "link": null,
      "locked": false,
      "startBinding": null,
      "endBinding": null,
      "lastCommittedPoint": null,
      "startArrowhead": null,
      "endArrowhead": null,
      "points": [
        [
          0,
          0
        ],
        [
          128.84193229844433,
          0
        ]
      ]
    },
    {
      "type": "line",
      "version": 2344,
      "versionNonce": 1403900809,
      "index": "bB9",
      "isDeleted": false,
      "id": "AjMNUxtntSOkklfCqbgiB",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 0,
      "opacity": 100,
      "angle": 0,
      "x": 181.96454496068355,
      "y": 227.8311118859238,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "#40c057",
      "width": 28.226201983883442,
      "height": 24.441122842819972,
      "seed": 1277747668,
      "groupIds": [
        "vvEwxQfV1LZ2VPdanx3xC"
      ],
      "frameId": null,
      "roundness": {
        "type": 2
      },
      "boundElements": [],
      "updated": 1720562263891,
      "link": null,
      "locked": false,
      "startBinding": null,
      "endBinding": null,
      "lastCommittedPoint": null,
      "startArrowhead": null,
      "endArrowhead": null,
      "points": [
        [
          0,
          0
        ],
        [
          -1.8305638688608794,
          -0.4146385831511896
        ],
        [
          -7.572250391862635,
          -6.271625431345418
        ],
        [
          -11.35769254157424,
          -4.347633009945142
        ],
        [
          -12.611765400987878,
          2.3733287939365098
        ],
        [
          -11.26889653301009,
          6.740045588043412
        ],
        [
          -17.42450906516472,
          8.886103861507927
        ],
        [
          -17.203202089974113,
          13.643170786196503
        ],
        [
          -12.380895778721076,
          13.349974465892952
        ],
        [
          -8.695178377089249,
          9.477701170522828
        ],
        [
          -3.6201449645383086,
          17.867626643824725
        ],
        [
          -0.415292101592283,
          18.169497411474552
        ],
        [
          -3.7772455950748833,
          4.800439161419844
        ],
        [
          1.3865838260401944,
          4.3476330099451115
        ],
        [
          3.162503997323137,
          5.86739618500973
        ],
        [
          9.236150983110862,
          5.86739618500973
        ],
        [
          10.80169291871872,
          0.6349695457461695
        ],
        [
          4.221225637895673,
          1.1103292603211785
        ],
        [
          5.486227236824892,
          -5.759833037916143
        ],
        [
          2.472627315401658,
          -5.345194454764953
        ],
        [
          -0.23770008446403423,
          -0.9229611976419838
        ],
        [
          0,
          0
        ]
      ]
    },
    {
      "type": "line",
      "version": 1764,
      "versionNonce": 933001833,
      "index": "bBA",
      "isDeleted": false,
      "id": "G8_WgrpvjBi76Oc7X77aE",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 90,
      "angle": 0,
      "x": 155.9454433727355,
      "y": 232.56963338464726,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "#99bcff",
      "width": 42.095115772272244,
      "height": 0,
      "seed": 2044036948,
      "groupIds": [
        "vvEwxQfV1LZ2VPdanx3xC"
      ],
      "frameId": null,
      "roundness": {
        "type": 2
      },
      "boundElements": [],
      "updated": 1720562263891,
      "link": null,
      "locked": false,
      "startBinding": null,
      "endBinding": null,
      "lastCommittedPoint": null,
      "startArrowhead": null,
      "endArrowhead": null,
      "points": [
        [
          0,
          0
        ],
        [
          42.095115772272244,
          0
        ]
      ]
    },
    {
      "type": "line",
      "version": 3971,
      "versionNonce": 953369929,
      "index": "bBB",
      "isDeleted": false,
      "id": "kNw8ac6DfoVHoRsX_sNYz",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 0,
      "opacity": 90,
      "angle": 0,
      "x": 161.04077240250984,
      "y": 217.13510445768105,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "#99bcff",
      "width": 29.31860660384862,
      "height": 5.711199931375845,
      "seed": 1708227796,
      "groupIds": [
        "vvEwxQfV1LZ2VPdanx3xC"
      ],
      "frameId": null,
      "roundness": {
        "type": 2
      },
      "boundElements": [],
      "updated": 1720562263891,
      "link": null,
      "locked": false,
      "startBinding": null,
      "endBinding": null,
      "lastCommittedPoint": null,
      "startArrowhead": null,
      "endArrowhead": null,
      "points": [
        [
          0,
          0
        ],
        [
          0.7724193963150375,
          2.2765797384357125
        ],
        [
          4.103544916365185,
          4.186609221587683
        ],
        [
          8.536129150893453,
          5.343311508306209
        ],
        [
          15.480325949120388,
          5.448716592152419
        ],
        [
          23.583965316012858,
          4.712296432964328
        ],
        [
          28.316582284417855,
          2.033216518393959
        ],
        [
          29.31860660384862,
          -0.2624833392234258
        ]
      ]
    },
    {
      "type": "line",
      "version": 4176,
      "versionNonce": 1118680105,
      "index": "bBC",
      "isDeleted": false,
      "id": "8epSwsufUIpGiETSIwyqA",
      "fillStyle": "solid",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 0,
      "opacity": 90,
      "angle": 0,
      "x": 162.55672391213574,
      "y": 248.87219651612196,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "#99bcff",
      "width": 29.31860660384862,
      "height": 5.896061363392446,
      "seed": 1011832788,
      "groupIds": [
        "vvEwxQfV1LZ2VPdanx3xC"
      ],
      "frameId": null,
      "roundness": {
        "type": 2
      },
      "boundElements": [],
      "updated": 1720562263891,
      "link": null,
      "locked": false,
      "startBinding": null,
      "endBinding": null,
      "lastCommittedPoint": null,
      "startArrowhead": null,
      "endArrowhead": null,
      "points": [
        [
          0,
          0
        ],
        [
          4.103544916365185,
          -4.322122351104391
        ],
        [
          8.536129150893453,
          -5.516265043290966
        ],
        [
          15.480325949120388,
          -5.625081903117008
        ],
        [
          23.583965316012858,
          -4.8648251269605955
        ],
        [
          28.316582284417855,
          -2.0990281379671547
        ],
        [
          29.31860660384862,
          0.2709794602754383
        ]
      ]
    },
    {
      "type": "line",
      "version": 2141,
      "versionNonce": 1783659475,
      "index": "bBD",
      "isDeleted": false,
      "id": "eXhRLcVoRtM1QpcVuYPAd",
      "fillStyle": "hachure",
      "strokeWidth": 1,
      "strokeStyle": "solid",
      "roughness": 1,
      "opacity": 100,
      "angle": 0,
      "x": 180.0938032565306,
      "y": 634.9947947706174,
      "strokeColor": "#881fa3",
      "backgroundColor": "#be4bdb",
      "width": 116.42036295658872,
      "height": 103.65107323746608,
      "seed": 629769964,
      "groupIds": [],
      "frameId": null,
      "roundness": null,
      "boundElements": [],
      "updated": 1718403881901,
      "link": null,
      "locked": false,
      "startBinding": null,
      "endBinding": null,
      "lastCommittedPoint": null,
      "startArrowhead": null,
      "endArrowhead": null,
      "points": [
        [
          0,
          0
        ],
        [
          -62.44191743896485,
          19.19929080548739
        ],
        [
          -63.17668831316513,
          79.43840749607878
        ],
        [
          -7.618334228588694,
          103.65107323746608
        ],
        [
          51.963117173367294,
          79.15871076413049
        ],
        [
          53.24367464342358,
          21.28567723840068
        ],
        [
          0,
          0
        ]
      ]
    },
    {
      "type": "arrow",
      "version": 3351,
      "versionNonce": 400506633,
