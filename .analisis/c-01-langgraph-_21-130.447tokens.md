Defined in: node\_modules/typescript/lib/lib.es5.d.ts:1076

##### Inherited from

`Error.name`

***

#### stack?

> `optional` **stack**: `string`

Defined in: node\_modules/typescript/lib/lib.es5.d.ts:1078

##### Inherited from

`Error.stack`

***

#### status

> **status**: `number`

Defined in: [src/auth/error.ts:67](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/error.ts#L67)

***

#### prepareStackTrace()?

> `static` `optional` **prepareStackTrace**: (`err`, `stackTraces`) => `any`

Defined in: node\_modules/@types/node/globals.d.ts:28

Optional override for formatting stack traces

##### Parameters

###### err

`Error`

###### stackTraces

`CallSite`[]

##### Returns

`any`

##### See

https://v8.dev/docs/stack-trace-api#customizing-stack-traces

##### Inherited from

`Error.prepareStackTrace`

***

#### stackTraceLimit

> `static` **stackTraceLimit**: `number`

Defined in: node\_modules/@types/node/globals.d.ts:30

##### Inherited from

`Error.stackTraceLimit`

### Methods

#### captureStackTrace()

> `static` **captureStackTrace**(`targetObject`, `constructorOpt`?): `void`

Defined in: node\_modules/@types/node/globals.d.ts:21

Create .stack property on a target object

##### Parameters

###### targetObject

`object`

###### constructorOpt?

`Function`

##### Returns

`void`

##### Inherited from

`Error.captureStackTrace`


<a name="authinterfacesautheventvaluemapmd"></a>

[**@langchain/langgraph-sdk**](#authreadmemd)

***

[@langchain/langgraph-sdk](#authreadmemd) / AuthEventValueMap

## Interface: AuthEventValueMap

Defined in: [src/auth/types.ts:218](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L218)

### Properties

#### assistants:create

> **assistants:create**: `object`

Defined in: [src/auth/types.ts:226](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L226)

##### assistant\_id?

> `optional` **assistant\_id**: `Maybe`\<`string`\>

##### config?

> `optional` **config**: `Maybe`\<`AssistantConfig`\>

##### graph\_id

> **graph\_id**: `string`

##### if\_exists?

> `optional` **if\_exists**: `Maybe`\<`"raise"` \| `"do_nothing"`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### name?

> `optional` **name**: `Maybe`\<`string`\>

***

#### assistants:delete

> **assistants:delete**: `object`

Defined in: [src/auth/types.ts:229](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L229)

##### assistant\_id

> **assistant\_id**: `string`

***

#### assistants:read

> **assistants:read**: `object`

Defined in: [src/auth/types.ts:227](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L227)

##### assistant\_id

> **assistant\_id**: `string`

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

***

#### assistants:search

> **assistants:search**: `object`

Defined in: [src/auth/types.ts:230](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L230)

##### graph\_id?

> `optional` **graph\_id**: `Maybe`\<`string`\>

##### limit?

> `optional` **limit**: `Maybe`\<`number`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### offset?

> `optional` **offset**: `Maybe`\<`number`\>

***

#### assistants:update

> **assistants:update**: `object`

Defined in: [src/auth/types.ts:228](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L228)

##### assistant\_id

> **assistant\_id**: `string`

##### config?

> `optional` **config**: `Maybe`\<`AssistantConfig`\>

##### graph\_id?

> `optional` **graph\_id**: `Maybe`\<`string`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### name?

> `optional` **name**: `Maybe`\<`string`\>

##### version?

> `optional` **version**: `Maybe`\<`number`\>

***

#### crons:create

> **crons:create**: `object`

Defined in: [src/auth/types.ts:232](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L232)

##### cron\_id?

> `optional` **cron\_id**: `Maybe`\<`string`\>

##### end\_time?

> `optional` **end\_time**: `Maybe`\<`string`\>

##### payload?

> `optional` **payload**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### schedule

> **schedule**: `string`

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

##### user\_id?

> `optional` **user\_id**: `Maybe`\<`string`\>

***

#### crons:delete

> **crons:delete**: `object`

Defined in: [src/auth/types.ts:235](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L235)

##### cron\_id

> **cron\_id**: `string`

***

#### crons:read

> **crons:read**: `object`

Defined in: [src/auth/types.ts:233](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L233)

##### cron\_id

> **cron\_id**: `string`

***

#### crons:search

> **crons:search**: `object`

Defined in: [src/auth/types.ts:236](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L236)

##### assistant\_id?

> `optional` **assistant\_id**: `Maybe`\<`string`\>

##### limit?

> `optional` **limit**: `Maybe`\<`number`\>

##### offset?

> `optional` **offset**: `Maybe`\<`number`\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

***

#### crons:update

> **crons:update**: `object`

Defined in: [src/auth/types.ts:234](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L234)

##### cron\_id

> **cron\_id**: `string`

##### payload?

> `optional` **payload**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### schedule?

> `optional` **schedule**: `Maybe`\<`string`\>

***

#### store:delete

> **store:delete**: `object`

Defined in: [src/auth/types.ts:242](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L242)

##### key

> **key**: `string`

##### namespace?

> `optional` **namespace**: `Maybe`\<`string`[]\>

***

#### store:get

> **store:get**: `object`

Defined in: [src/auth/types.ts:239](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L239)

##### key

> **key**: `string`

##### namespace

> **namespace**: `Maybe`\<`string`[]\>

***

#### store:list\_namespaces

> **store:list\_namespaces**: `object`

Defined in: [src/auth/types.ts:241](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L241)

##### limit?

> `optional` **limit**: `Maybe`\<`number`\>

##### max\_depth?

> `optional` **max\_depth**: `Maybe`\<`number`\>

##### namespace?

> `optional` **namespace**: `Maybe`\<`string`[]\>

##### offset?

> `optional` **offset**: `Maybe`\<`number`\>

##### suffix?

> `optional` **suffix**: `Maybe`\<`string`[]\>

***

#### store:put

> **store:put**: `object`

Defined in: [src/auth/types.ts:238](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L238)

##### key

> **key**: `string`

##### namespace

> **namespace**: `string`[]

##### value

> **value**: `Record`\<`string`, `unknown`\>

***

#### store:search

> **store:search**: `object`

Defined in: [src/auth/types.ts:240](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L240)

##### filter?

> `optional` **filter**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### limit?

> `optional` **limit**: `Maybe`\<`number`\>

##### namespace?

> `optional` **namespace**: `Maybe`\<`string`[]\>

##### offset?

> `optional` **offset**: `Maybe`\<`number`\>

##### query?

> `optional` **query**: `Maybe`\<`string`\>

***

#### threads:create

> **threads:create**: `object`

Defined in: [src/auth/types.ts:219](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L219)

##### if\_exists?

> `optional` **if\_exists**: `Maybe`\<`"raise"` \| `"do_nothing"`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

***

#### threads:create\_run

> **threads:create\_run**: `object`

Defined in: [src/auth/types.ts:224](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L224)

##### after\_seconds?

> `optional` **after\_seconds**: `Maybe`\<`number`\>

##### assistant\_id

> **assistant\_id**: `string`

##### if\_not\_exists?

> `optional` **if\_not\_exists**: `Maybe`\<`"reject"` \| `"create"`\>

##### kwargs

> **kwargs**: `Record`\<`string`, `unknown`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### multitask\_strategy?

> `optional` **multitask\_strategy**: `Maybe`\<`"reject"` \| `"interrupt"` \| `"rollback"` \| `"enqueue"`\>

##### prevent\_insert\_if\_inflight?

> `optional` **prevent\_insert\_if\_inflight**: `Maybe`\<`boolean`\>

##### run\_id

> **run\_id**: `string`

##### status

> **status**: `Maybe`\<`"pending"` \| `"running"` \| `"error"` \| `"success"` \| `"timeout"` \| `"interrupted"`\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

***

#### threads:delete

> **threads:delete**: `object`

Defined in: [src/auth/types.ts:222](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L222)

##### run\_id?

> `optional` **run\_id**: `Maybe`\<`string`\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

***

#### threads:read

> **threads:read**: `object`

Defined in: [src/auth/types.ts:220](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L220)

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

***

#### threads:search

> **threads:search**: `object`

Defined in: [src/auth/types.ts:223](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L223)

##### limit?

> `optional` **limit**: `Maybe`\<`number`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### offset?

> `optional` **offset**: `Maybe`\<`number`\>

##### status?

> `optional` **status**: `Maybe`\<`"error"` \| `"interrupted"` \| `"idle"` \| `"busy"` \| `string` & `object`\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>

##### values?

> `optional` **values**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

***

#### threads:update

> **threads:update**: `object`

Defined in: [src/auth/types.ts:221](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L221)

##### action?

> `optional` **action**: `Maybe`\<`"interrupt"` \| `"rollback"`\>

##### metadata?

> `optional` **metadata**: `Maybe`\<`Record`\<`string`, `unknown`\>\>

##### thread\_id?

> `optional` **thread\_id**: `Maybe`\<`string`\>


<a name="authtype-aliasesauthfiltersmd"></a>

[**@langchain/langgraph-sdk**](#authreadmemd)

***

[@langchain/langgraph-sdk](#authreadmemd) / AuthFilters

## Type Alias: AuthFilters\<TKey\>

> **AuthFilters**\<`TKey`\>: \{ \[key in TKey\]: string \| \{ \[op in "$contains" \| "$eq"\]?: string \} \}

Defined in: [src/auth/types.ts:367](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/auth/types.ts#L367)

### Type Parameters

• **TKey** *extends* `string` \| `number` \| `symbol`


<a name="classesassistantsclientmd"></a>

[**@langchain/langgraph-sdk**](#readmemd)

***

[@langchain/langgraph-sdk](#readmemd) / AssistantsClient

## Class: AssistantsClient

Defined in: [client.ts:294](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L294)

### Extends

- `BaseClient`

### Constructors

#### new AssistantsClient()

> **new AssistantsClient**(`config`?): [`AssistantsClient`](#classesassistantsclientmd)

Defined in: [client.ts:88](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L88)

##### Parameters

###### config?

[`ClientConfig`](#interfacesclientconfigmd)

##### Returns

[`AssistantsClient`](#classesassistantsclientmd)

##### Inherited from

`BaseClient.constructor`

### Methods

#### create()

> **create**(`payload`): `Promise`\<`Assistant`\>

Defined in: [client.ts:359](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L359)

Create a new assistant.

##### Parameters

###### payload

Payload for creating an assistant.

####### assistantId?

`string`

####### config?

`Config`

####### description?

`string`

####### graphId

`string`

####### ifExists?

`OnConflictBehavior`

####### metadata?

`Metadata`

####### name?

`string`

##### Returns

`Promise`\<`Assistant`\>

The created assistant.

***

#### delete()

> **delete**(`assistantId`): `Promise`\<`void`\>

Defined in: [client.ts:415](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L415)

Delete an assistant.

##### Parameters

###### assistantId

`string`

ID of the assistant.

##### Returns

`Promise`\<`void`\>

***

#### get()

> **get**(`assistantId`): `Promise`\<`Assistant`\>

Defined in: [client.ts:301](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L301)

Get an assistant by ID.

##### Parameters

###### assistantId

`string`

The ID of the assistant.

##### Returns

`Promise`\<`Assistant`\>

Assistant

***

#### getGraph()

> **getGraph**(`assistantId`, `options`?): `Promise`\<`AssistantGraph`\>

Defined in: [client.ts:311](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L311)

Get the JSON representation of the graph assigned to a runnable

##### Parameters

###### assistantId

`string`

The ID of the assistant.

###### options?

####### xray?

`number` \| `boolean`

Whether to include subgraphs in the serialized graph representation. If an integer value is provided, only subgraphs with a depth less than or equal to the value will be included.

##### Returns

`Promise`\<`AssistantGraph`\>

Serialized graph

***

#### getSchemas()

> **getSchemas**(`assistantId`): `Promise`\<`GraphSchema`\>

Defined in: [client.ts:325](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L325)

Get the state and config schema of the graph assigned to a runnable

##### Parameters

###### assistantId

`string`

The ID of the assistant.

##### Returns

`Promise`\<`GraphSchema`\>

Graph schema

***

#### getSubgraphs()

> **getSubgraphs**(`assistantId`, `options`?): `Promise`\<`Subgraphs`\>

Defined in: [client.ts:336](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L336)

Get the schemas of an assistant by ID.

##### Parameters

###### assistantId

`string`

The ID of the assistant to get the schema of.

###### options?

Additional options for getting subgraphs, such as namespace or recursion extraction.

####### namespace?

`string`

####### recurse?

`boolean`

##### Returns

`Promise`\<`Subgraphs`\>

The subgraphs of the assistant.

***

#### getVersions()

> **getVersions**(`assistantId`, `payload`?): `Promise`\<`AssistantVersion`[]\>

Defined in: [client.ts:453](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L453)

List all versions of an assistant.

##### Parameters

###### assistantId

`string`

ID of the assistant.

###### payload?

####### limit?

`number`

####### metadata?

`Metadata`

####### offset?

`number`

##### Returns

`Promise`\<`AssistantVersion`[]\>

List of assistant versions.

***

#### search()

> **search**(`query`?): `Promise`\<`Assistant`[]\>

Defined in: [client.ts:426](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L426)

List assistants.

##### Parameters

###### query?

Query options.

####### graphId?

`string`

####### limit?

`number`

####### metadata?

`Metadata`

####### offset?

`number`

####### sortBy?

`AssistantSortBy`

####### sortOrder?

`SortOrder`

##### Returns

`Promise`\<`Assistant`[]\>

List of assistants.

***

#### setLatest()

> **setLatest**(`assistantId`, `version`): `Promise`\<`Assistant`\>

Defined in: [client.ts:481](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L481)

Change the version of an assistant.

##### Parameters

###### assistantId

`string`

ID of the assistant.

###### version

`number`

The version to change to.

##### Returns

`Promise`\<`Assistant`\>

The updated assistant.

***

#### update()

> **update**(`assistantId`, `payload`): `Promise`\<`Assistant`\>

Defined in: [client.ts:388](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L388)

Update an assistant.

##### Parameters

###### assistantId

`string`

ID of the assistant.

###### payload

Payload for updating the assistant.

####### config?

`Config`

####### description?

`string`

####### graphId?

`string`

####### metadata?

`Metadata`

####### name?

`string`

##### Returns

`Promise`\<`Assistant`\>

The updated assistant.


<a name="classesclientmd"></a>

[**@langchain/langgraph-sdk**](#readmemd)

***

[@langchain/langgraph-sdk](#readmemd) / Client

## Class: Client\<TStateType, TUpdateType, TCustomEventType\>

Defined in: [client.ts:1448](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1448)

### Type Parameters

• **TStateType** = `DefaultValues`

• **TUpdateType** = `TStateType`

• **TCustomEventType** = `unknown`

### Constructors

#### new Client()

> **new Client**\<`TStateType`, `TUpdateType`, `TCustomEventType`\>(`config`?): [`Client`](#classesclientmd)\<`TStateType`, `TUpdateType`, `TCustomEventType`\>

Defined in: [client.ts:1484](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1484)

##### Parameters

###### config?

[`ClientConfig`](#interfacesclientconfigmd)

##### Returns

[`Client`](#classesclientmd)\<`TStateType`, `TUpdateType`, `TCustomEventType`\>

### Properties

#### ~ui

> **~ui**: `UiClient`

Defined in: [client.ts:1482](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1482)

**`Internal`**

The client for interacting with the UI.
 Used by LoadExternalComponent and the API might change in the future.

***

#### assistants

> **assistants**: [`AssistantsClient`](#classesassistantsclientmd)

Defined in: [client.ts:1456](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1456)

The client for interacting with assistants.

***

#### crons

> **crons**: [`CronsClient`](#classescronsclientmd)

Defined in: [client.ts:1471](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1471)

The client for interacting with cron runs.

***

#### runs

> **runs**: [`RunsClient`](#classesrunsclientmd)\<`TStateType`, `TUpdateType`, `TCustomEventType`\>

Defined in: [client.ts:1466](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1466)

The client for interacting with runs.

***

#### store

> **store**: [`StoreClient`](#classesstoreclientmd)

Defined in: [client.ts:1476](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1476)

The client for interacting with the KV store.

***

#### threads

> **threads**: [`ThreadsClient`](#classesthreadsclientmd)\<`TStateType`, `TUpdateType`\>

Defined in: [client.ts:1461](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1461)

The client for interacting with threads.


<a name="classescronsclientmd"></a>

[**@langchain/langgraph-sdk**](#readmemd)

***

[@langchain/langgraph-sdk](#readmemd) / CronsClient

## Class: CronsClient

Defined in: [client.ts:197](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L197)

### Extends

- `BaseClient`

### Constructors

#### new CronsClient()

> **new CronsClient**(`config`?): [`CronsClient`](#classescronsclientmd)

Defined in: [client.ts:88](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L88)

##### Parameters

###### config?

[`ClientConfig`](#interfacesclientconfigmd)

##### Returns

[`CronsClient`](#classescronsclientmd)

##### Inherited from

`BaseClient.constructor`

### Methods

#### create()

> **create**(`assistantId`, `payload`?): `Promise`\<`CronCreateResponse`\>

Defined in: [client.ts:238](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L238)

##### Parameters

###### assistantId

`string`

Assistant ID to use for this cron job.

###### payload?

`CronsCreatePayload`

Payload for creating a cron job.

##### Returns

`Promise`\<`CronCreateResponse`\>

***

#### createForThread()

> **createForThread**(`threadId`, `assistantId`, `payload`?): `Promise`\<`CronCreateForThreadResponse`\>

Defined in: [client.ts:205](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L205)

##### Parameters

###### threadId

`string`

The ID of the thread.

###### assistantId

`string`

Assistant ID to use for this cron job.

###### payload?

`CronsCreatePayload`

Payload for creating a cron job.

##### Returns

`Promise`\<`CronCreateForThreadResponse`\>

The created background run.

***

#### delete()

> **delete**(`cronId`): `Promise`\<`void`\>

Defined in: [client.ts:265](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L265)

##### Parameters

###### cronId

`string`

Cron ID of Cron job to delete.

##### Returns

`Promise`\<`void`\>

***

#### search()

> **search**(`query`?): `Promise`\<`Cron`[]\>

Defined in: [client.ts:276](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L276)

##### Parameters

###### query?

Query options.

####### assistantId?

`string`

####### limit?

`number`

####### offset?

`number`

####### threadId?

`string`

##### Returns

`Promise`\<`Cron`[]\>

List of crons.


<a name="classesrunsclientmd"></a>

[**@langchain/langgraph-sdk**](#readmemd)

***

[@langchain/langgraph-sdk](#readmemd) / RunsClient

## Class: RunsClient\<TStateType, TUpdateType, TCustomEventType\>

Defined in: [client.ts:776](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L776)

### Extends

- `BaseClient`

### Type Parameters

• **TStateType** = `DefaultValues`

• **TUpdateType** = `TStateType`

• **TCustomEventType** = `unknown`

### Constructors

#### new RunsClient()

> **new RunsClient**\<`TStateType`, `TUpdateType`, `TCustomEventType`\>(`config`?): [`RunsClient`](#classesrunsclientmd)\<`TStateType`, `TUpdateType`, `TCustomEventType`\>

Defined in: [client.ts:88](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L88)

##### Parameters

###### config?

[`ClientConfig`](#interfacesclientconfigmd)

##### Returns

[`RunsClient`](#classesrunsclientmd)\<`TStateType`, `TUpdateType`, `TCustomEventType`\>

##### Inherited from

`BaseClient.constructor`

### Methods

#### cancel()

> **cancel**(`threadId`, `runId`, `wait`, `action`): `Promise`\<`void`\>

Defined in: [client.ts:1063](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1063)

Cancel a run.

##### Parameters

###### threadId

`string`

The ID of the thread.

###### runId

`string`

The ID of the run.

###### wait

`boolean` = `false`

Whether to block when canceling

###### action

`CancelAction` = `"interrupt"`

Action to take when cancelling the run. Possible values are `interrupt` or `rollback`. Default is `interrupt`.

##### Returns

`Promise`\<`void`\>

***

#### create()

> **create**(`threadId`, `assistantId`, `payload`?): `Promise`\<`Run`\>

Defined in: [client.ts:885](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L885)

Create a run.

##### Parameters

###### threadId

`string`

The ID of the thread.

###### assistantId

`string`

Assistant ID to use for this run.

###### payload?

`RunsCreatePayload`

Payload for creating a run.

##### Returns

`Promise`\<`Run`\>

The created run.

***

#### createBatch()

> **createBatch**(`payloads`): `Promise`\<`Run`[]\>

Defined in: [client.ts:921](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L921)

Create a batch of stateless background runs.

##### Parameters

###### payloads

`RunsCreatePayload` & `object`[]

An array of payloads for creating runs.

##### Returns

`Promise`\<`Run`[]\>

An array of created runs.

***

#### delete()

> **delete**(`threadId`, `runId`): `Promise`\<`void`\>

Defined in: [client.ts:1157](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1157)

Delete a run.

##### Parameters

###### threadId

`string`

The ID of the thread.

###### runId

`string`

The ID of the run.

##### Returns

`Promise`\<`void`\>

***

#### get()

> **get**(`threadId`, `runId`): `Promise`\<`Run`\>

Defined in: [client.ts:1050](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1050)

Get a run by ID.

##### Parameters

###### threadId

`string`

The ID of the thread.

###### runId

`string`

The ID of the run.

##### Returns

`Promise`\<`Run`\>

The run.

***

#### join()

> **join**(`threadId`, `runId`, `options`?): `Promise`\<`void`\>

Defined in: [client.ts:1085](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1085)

Block until a run is done.

##### Parameters

###### threadId

`string`

The ID of the thread.

###### runId

`string`

The ID of the run.

###### options?

####### signal?

`AbortSignal`

##### Returns

`Promise`\<`void`\>

***

#### joinStream()

> **joinStream**(`threadId`, `runId`, `options`?): `AsyncGenerator`\<\{ `data`: `any`; `event`: `StreamEvent`; \}\>

Defined in: [client.ts:1111](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1111)

Stream output from a run in real-time, until the run is done.
Output is not buffered, so any output produced before this call will
not be received here.

##### Parameters

###### threadId

`string`

The ID of the thread.

###### runId

`string`

The ID of the run.

###### options?

Additional options for controlling the stream behavior:
  - signal: An AbortSignal that can be used to cancel the stream request
  - cancelOnDisconnect: When true, automatically cancels the run if the client disconnects from the stream
  - streamMode: Controls what types of events to receive from the stream (can be a single mode or array of modes)
       Must be a subset of the stream modes passed when creating the run. Background runs default to having the union of all
       stream modes enabled.

`AbortSignal` | \{ `cancelOnDisconnect`: `boolean`; `signal`: `AbortSignal`; `streamMode`: `StreamMode` \| `StreamMode`[]; \}

##### Returns

`AsyncGenerator`\<\{ `data`: `any`; `event`: `StreamEvent`; \}\>

An async generator yielding stream parts.

***

#### list()

> **list**(`threadId`, `options`?): `Promise`\<`Run`[]\>

Defined in: [client.ts:1013](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1013)

List all runs for a thread.

##### Parameters

###### threadId

`string`

The ID of the thread.

###### options?

Filtering and pagination options.

####### limit?

`number`

Maximum number of runs to return.
Defaults to 10

####### offset?

`number`

Offset to start from.
Defaults to 0.

####### status?

`RunStatus`

Status of the run to filter by.

##### Returns

`Promise`\<`Run`[]\>

List of runs.

***

#### stream()

Create a run and stream the results.

##### Param

The ID of the thread.

##### Param

Assistant ID to use for this run.

##### Param

Payload for creating a run.

##### Call Signature

> **stream**\<`TStreamMode`, `TSubgraphs`\>(`threadId`, `assistantId`, `payload`?): `TypedAsyncGenerator`\<`TStreamMode`, `TSubgraphs`, `TStateType`, `TUpdateType`, `TCustomEventType`\>

Defined in: [client.ts:781](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L781)

###### Type Parameters

• **TStreamMode** *extends* `StreamMode` \| `StreamMode`[] = `StreamMode`

• **TSubgraphs** *extends* `boolean` = `false`

###### Parameters

####### threadId

`null`

####### assistantId

`string`

####### payload?

`Omit`\<`RunsStreamPayload`\<`TStreamMode`, `TSubgraphs`\>, `"multitaskStrategy"` \| `"onCompletion"`\>

###### Returns

`TypedAsyncGenerator`\<`TStreamMode`, `TSubgraphs`, `TStateType`, `TUpdateType`, `TCustomEventType`\>

##### Call Signature

> **stream**\<`TStreamMode`, `TSubgraphs`\>(`threadId`, `assistantId`, `payload`?): `TypedAsyncGenerator`\<`TStreamMode`, `TSubgraphs`, `TStateType`, `TUpdateType`, `TCustomEventType`\>

Defined in: [client.ts:799](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L799)

###### Type Parameters

• **TStreamMode** *extends* `StreamMode` \| `StreamMode`[] = `StreamMode`

• **TSubgraphs** *extends* `boolean` = `false`

###### Parameters

####### threadId

`string`

####### assistantId

`string`

####### payload?

`RunsStreamPayload`\<`TStreamMode`, `TSubgraphs`\>

###### Returns

`TypedAsyncGenerator`\<`TStreamMode`, `TSubgraphs`, `TStateType`, `TUpdateType`, `TCustomEventType`\>

***

#### wait()

Create a run and wait for it to complete.

##### Param

The ID of the thread.

##### Param

Assistant ID to use for this run.

##### Param

Payload for creating a run.

##### Call Signature

> **wait**(`threadId`, `assistantId`, `payload`?): `Promise`\<`DefaultValues`\>

Defined in: [client.ts:938](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L938)

###### Parameters

####### threadId

`null`

####### assistantId

`string`

####### payload?

`Omit`\<`RunsWaitPayload`, `"multitaskStrategy"` \| `"onCompletion"`\>

###### Returns

`Promise`\<`DefaultValues`\>

##### Call Signature

> **wait**(`threadId`, `assistantId`, `payload`?): `Promise`\<`DefaultValues`\>

Defined in: [client.ts:944](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L944)

###### Parameters

####### threadId

`string`

####### assistantId

`string`

####### payload?

`RunsWaitPayload`

###### Returns

`Promise`\<`DefaultValues`\>


<a name="classesstoreclientmd"></a>

[**@langchain/langgraph-sdk**](#readmemd)

***

[@langchain/langgraph-sdk](#readmemd) / StoreClient

## Class: StoreClient

Defined in: [client.ts:1175](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1175)

### Extends

- `BaseClient`

### Constructors

#### new StoreClient()

> **new StoreClient**(`config`?): [`StoreClient`](#classesstoreclientmd)

Defined in: [client.ts:88](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L88)

##### Parameters

###### config?

[`ClientConfig`](#interfacesclientconfigmd)

##### Returns

[`StoreClient`](#classesstoreclientmd)

##### Inherited from

`BaseClient.constructor`

### Methods

#### deleteItem()

> **deleteItem**(`namespace`, `key`): `Promise`\<`void`\>

Defined in: [client.ts:1296](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1296)

Delete an item.

##### Parameters

###### namespace

`string`[]

A list of strings representing the namespace path.

###### key

`string`

The unique identifier for the item.

##### Returns

`Promise`\<`void`\>

Promise<void>

***

#### getItem()

> **getItem**(`namespace`, `key`, `options`?): `Promise`\<`null` \| `Item`\>

Defined in: [client.ts:1252](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1252)

Retrieve a single item.

##### Parameters

###### namespace

`string`[]

A list of strings representing the namespace path.

###### key

`string`

The unique identifier for the item.

###### options?

####### refreshTtl?

`null` \| `boolean`

Whether to refresh the TTL on this read operation. If null, uses the store's default behavior.

##### Returns

`Promise`\<`null` \| `Item`\>

Promise<Item>

##### Example

```typescript
const item = await client.store.getItem(
  ["documents", "user123"],
  "item456",
  { refreshTtl: true }
);
console.log(item);
// {
//   namespace: ["documents", "user123"],
//   key: "item456",
//   value: { title: "My Document", content: "Hello World" },
//   createdAt: "2024-07-30T12:00:00Z",
//   updatedAt: "2024-07-30T12:00:00Z"
// }
```

***

#### listNamespaces()

> **listNamespaces**(`options`?): `Promise`\<`ListNamespaceResponse`\>

Defined in: [client.ts:1392](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1392)

List namespaces with optional match conditions.

##### Parameters

###### options?

####### limit?

`number`

Maximum number of namespaces to return (default is 100).

####### maxDepth?

`number`

Optional integer specifying the maximum depth of namespaces to return.

####### offset?

`number`

Number of namespaces to skip before returning results (default is 0).

####### prefix?

`string`[]

Optional list of strings representing the prefix to filter namespaces.

####### suffix?

`string`[]

Optional list of strings representing the suffix to filter namespaces.

##### Returns

`Promise`\<`ListNamespaceResponse`\>

Promise<ListNamespaceResponse>

***

#### putItem()

> **putItem**(`namespace`, `key`, `value`, `options`?): `Promise`\<`void`\>

Defined in: [client.ts:1196](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1196)

Store or update an item.

##### Parameters

###### namespace

`string`[]

A list of strings representing the namespace path.

###### key

`string`

The unique identifier for the item within the namespace.

###### value

`Record`\<`string`, `any`\>

A dictionary containing the item's data.

###### options?

####### index?

`null` \| `false` \| `string`[]

Controls search indexing - null (use defaults), false (disable), or list of field paths to index.

####### ttl?

`null` \| `number`

Optional time-to-live in minutes for the item, or null for no expiration.

##### Returns

`Promise`\<`void`\>

Promise<void>

##### Example

```typescript
await client.store.putItem(
  ["documents", "user123"],
  "item456",
  { title: "My Document", content: "Hello World" },
  { ttl: 60 } // expires in 60 minutes
);
```

***

#### searchItems()

> **searchItems**(`namespacePrefix`, `options`?): `Promise`\<`SearchItemsResponse`\>

Defined in: [client.ts:1347](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L1347)

Search for items within a namespace prefix.

##### Parameters

###### namespacePrefix

`string`[]

List of strings representing the namespace prefix.

###### options?

####### filter?

`Record`\<`string`, `any`\>

Optional dictionary of key-value pairs to filter results.

####### limit?

`number`

Maximum number of items to return (default is 10).

####### offset?

`number`

Number of items to skip before returning results (default is 0).

####### query?

`string`

Optional search query.

####### refreshTtl?

`null` \| `boolean`

Whether to refresh the TTL on items returned by this search. If null, uses the store's default behavior.

##### Returns

`Promise`\<`SearchItemsResponse`\>

Promise<SearchItemsResponse>

##### Example

```typescript
const results = await client.store.searchItems(
  ["documents"],
  {
    filter: { author: "John Doe" },
    limit: 5,
    refreshTtl: true
  }
);
console.log(results);
// {
//   items: [
//     {
//       namespace: ["documents", "user123"],
//       key: "item789",
//       value: { title: "Another Document", author: "John Doe" },
//       createdAt: "2024-07-30T12:00:00Z",
//       updatedAt: "2024-07-30T12:00:00Z"
//     },
//     // ... additional items ...
//   ]
// }
```


<a name="classesthreadsclientmd"></a>

[**@langchain/langgraph-sdk**](#readmemd)

***

[@langchain/langgraph-sdk](#readmemd) / ThreadsClient

## Class: ThreadsClient\<TStateType, TUpdateType\>

Defined in: [client.ts:489](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L489)

### Extends

- `BaseClient`

### Type Parameters

• **TStateType** = `DefaultValues`

• **TUpdateType** = `TStateType`

### Constructors

#### new ThreadsClient()

> **new ThreadsClient**\<`TStateType`, `TUpdateType`\>(`config`?): [`ThreadsClient`](#classesthreadsclientmd)\<`TStateType`, `TUpdateType`\>

Defined in: [client.ts:88](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L88)

##### Parameters

###### config?

[`ClientConfig`](#interfacesclientconfigmd)

##### Returns

[`ThreadsClient`](#classesthreadsclientmd)\<`TStateType`, `TUpdateType`\>

##### Inherited from

`BaseClient.constructor`

### Methods

#### copy()

> **copy**(`threadId`): `Promise`\<`Thread`\<`TStateType`\>\>

Defined in: [client.ts:566](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L566)

Copy an existing thread

##### Parameters

###### threadId

`string`

ID of the thread to be copied

##### Returns

`Promise`\<`Thread`\<`TStateType`\>\>

Newly copied thread

***

#### create()

> **create**(`payload`?): `Promise`\<`Thread`\<`TStateType`\>\>

Defined in: [client.ts:511](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L511)

Create a new thread.

##### Parameters

###### payload?

Payload for creating a thread.

####### graphId?

`string`

Graph ID to associate with the thread.

####### ifExists?

`OnConflictBehavior`

How to handle duplicate creation.

**Default**

```ts
"raise"
```

####### metadata?

`Metadata`

Metadata for the thread.

####### supersteps?

`object`[]

Apply a list of supersteps when creating a thread, each containing a sequence of updates.

Used for copying a thread between deployments.

####### threadId?

`string`

ID of the thread to create.

If not provided, a random UUID will be generated.

##### Returns

`Promise`\<`Thread`\<`TStateType`\>\>

The created thread.

***

#### delete()

> **delete**(`threadId`): `Promise`\<`void`\>

Defined in: [client.ts:599](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L599)

Delete a thread.

##### Parameters

###### threadId

`string`

ID of the thread.

##### Returns

`Promise`\<`void`\>

***

#### get()

> **get**\<`ValuesType`\>(`threadId`): `Promise`\<`Thread`\<`ValuesType`\>\>

Defined in: [client.ts:499](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L499)

Get a thread by ID.

##### Type Parameters

• **ValuesType** = `TStateType`

##### Parameters

###### threadId

`string`

ID of the thread.

##### Returns

`Promise`\<`Thread`\<`ValuesType`\>\>

The thread.

***

#### getHistory()

> **getHistory**\<`ValuesType`\>(`threadId`, `options`?): `Promise`\<`ThreadState`\<`ValuesType`\>[]\>

Defined in: [client.ts:752](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L752)

Get all past states for a thread.

##### Type Parameters

• **ValuesType** = `TStateType`

##### Parameters

###### threadId

`string`

ID of the thread.

###### options?

Additional options.

####### before?

`Config`

####### checkpoint?

`Partial`\<`Omit`\<`Checkpoint`, `"thread_id"`\>\>

####### limit?

`number`

####### metadata?

`Metadata`

##### Returns

`Promise`\<`ThreadState`\<`ValuesType`\>[]\>

List of thread states.

***

#### getState()

> **getState**\<`ValuesType`\>(`threadId`, `checkpoint`?, `options`?): `Promise`\<`ThreadState`\<`ValuesType`\>\>

Defined in: [client.ts:659](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L659)

Get state for a thread.

##### Type Parameters

• **ValuesType** = `TStateType`

##### Parameters

###### threadId

`string`

ID of the thread.

###### checkpoint?

`string` | `Checkpoint`

###### options?

####### subgraphs?

`boolean`

##### Returns

`Promise`\<`ThreadState`\<`ValuesType`\>\>

Thread state.

***

#### patchState()

> **patchState**(`threadIdOrConfig`, `metadata`): `Promise`\<`void`\>

Defined in: [client.ts:722](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L722)

Patch the metadata of a thread.

##### Parameters

###### threadIdOrConfig

Thread ID or config to patch the state of.

`string` | `Config`

###### metadata

`Metadata`

Metadata to patch the state with.

##### Returns

`Promise`\<`void`\>

***

#### search()

> **search**\<`ValuesType`\>(`query`?): `Promise`\<`Thread`\<`ValuesType`\>[]\>

Defined in: [client.ts:611](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L611)

List threads

##### Type Parameters

• **ValuesType** = `TStateType`

##### Parameters

###### query?

Query options

####### limit?

`number`

Maximum number of threads to return.
Defaults to 10

####### metadata?

`Metadata`

Metadata to filter threads by.

####### offset?

`number`

Offset to start from.

####### sortBy?

`ThreadSortBy`

Sort by.

####### sortOrder?

`SortOrder`

Sort order.
Must be one of 'asc' or 'desc'.

####### status?

`ThreadStatus`

Thread status to filter on.
Must be one of 'idle', 'busy', 'interrupted' or 'error'.

##### Returns

`Promise`\<`Thread`\<`ValuesType`\>[]\>

List of threads

***

#### update()

> **update**(`threadId`, `payload`?): `Promise`\<`Thread`\<`DefaultValues`\>\>

Defined in: [client.ts:579](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L579)

Update a thread.

##### Parameters

###### threadId

`string`

ID of the thread.

###### payload?

Payload for updating the thread.

####### metadata?

`Metadata`

Metadata for the thread.

##### Returns

`Promise`\<`Thread`\<`DefaultValues`\>\>

The updated thread.

***

#### updateState()

> **updateState**\<`ValuesType`\>(`threadId`, `options`): `Promise`\<`Pick`\<`Config`, `"configurable"`\>\>

Defined in: [client.ts:693](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L693)

Add state to a thread.

##### Type Parameters

• **ValuesType** = `TUpdateType`

##### Parameters

###### threadId

`string`

The ID of the thread.

###### options

####### asNode?

`string`

####### checkpoint?

`Checkpoint`

####### checkpointId?

`string`

####### values

`ValuesType`

##### Returns

`Promise`\<`Pick`\<`Config`, `"configurable"`\>\>


<a name="functionsgetapikeymd"></a>

[**@langchain/langgraph-sdk**](#readmemd)

***

[@langchain/langgraph-sdk](#readmemd) / getApiKey

## Function: getApiKey()

> **getApiKey**(`apiKey`?): `undefined` \| `string`

Defined in: [client.ts:53](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L53)

Get the API key from the environment.
Precedence:
  1. explicit argument
  2. LANGGRAPH_API_KEY
  3. LANGSMITH_API_KEY
  4. LANGCHAIN_API_KEY

### Parameters

#### apiKey?

`string`

Optional API key provided as an argument

### Returns

`undefined` \| `string`

The API key if found, otherwise undefined


<a name="interfacesclientconfigmd"></a>

[**@langchain/langgraph-sdk**](#readmemd)

***

[@langchain/langgraph-sdk](#readmemd) / ClientConfig

## Interface: ClientConfig

Defined in: [client.ts:71](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L71)

### Properties

#### apiKey?

> `optional` **apiKey**: `string`

Defined in: [client.ts:73](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L73)

***

#### apiUrl?

> `optional` **apiUrl**: `string`

Defined in: [client.ts:72](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L72)

***

#### callerOptions?

> `optional` **callerOptions**: `AsyncCallerParams`

Defined in: [client.ts:74](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L74)

***

#### defaultHeaders?

> `optional` **defaultHeaders**: `Record`\<`string`, `undefined` \| `null` \| `string`\>

Defined in: [client.ts:76](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L76)

***

#### timeoutMs?

> `optional` **timeoutMs**: `number`

Defined in: [client.ts:75](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/client.ts#L75)


<a name="reactreadmemd"></a>

**@langchain/langgraph-sdk**

***

## @langchain/langgraph-sdk/react

### Interfaces

- [UseStream](#reactinterfacesusestreammd)
- [UseStreamOptions](#reactinterfacesusestreamoptionsmd)

### Type Aliases

- [MessageMetadata](#reacttype-aliasesmessagemetadatamd)

### Functions

- [useStream](#reactfunctionsusestreammd)


<a name="reactfunctionsusestreammd"></a>

[**@langchain/langgraph-sdk**](#reactreadmemd)

***

[@langchain/langgraph-sdk](#reactreadmemd) / useStream

## Function: useStream()

> **useStream**\<`StateType`, `Bag`\>(`options`): [`UseStream`](#reactinterfacesusestreammd)\<`StateType`, `Bag`\>

Defined in: [react/stream.tsx:618](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L618)

### Type Parameters

• **StateType** *extends* `Record`\<`string`, `unknown`\> = `Record`\<`string`, `unknown`\>

• **Bag** *extends* `object` = `BagTemplate`

### Parameters

#### options

[`UseStreamOptions`](#reactinterfacesusestreamoptionsmd)\<`StateType`, `Bag`\>

### Returns

[`UseStream`](#reactinterfacesusestreammd)\<`StateType`, `Bag`\>


<a name="reactinterfacesusestreammd"></a>

[**@langchain/langgraph-sdk**](#reactreadmemd)

***

[@langchain/langgraph-sdk](#reactreadmemd) / UseStream

## Interface: UseStream\<StateType, Bag\>

Defined in: [react/stream.tsx:507](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L507)

### Type Parameters

• **StateType** *extends* `Record`\<`string`, `unknown`\> = `Record`\<`string`, `unknown`\>

• **Bag** *extends* `BagTemplate` = `BagTemplate`

### Properties

#### assistantId

> **assistantId**: `string`

Defined in: [react/stream.tsx:592](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L592)

The ID of the assistant to use.

***

#### branch

> **branch**: `string`

Defined in: [react/stream.tsx:542](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L542)

The current branch of the thread.

***

#### client

> **client**: `Client`

Defined in: [react/stream.tsx:587](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L587)

LangGraph SDK client used to send request and receive responses.

***

#### error

> **error**: `unknown`

Defined in: [react/stream.tsx:519](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L519)

Last seen error from the thread or during streaming.

***

#### experimental\_branchTree

> **experimental\_branchTree**: `Sequence`\<`StateType`\>

Defined in: [react/stream.tsx:558](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L558)

**`Experimental`**

Tree of all branches for the thread.

***

#### getMessagesMetadata()

> **getMessagesMetadata**: (`message`, `index`?) => `undefined` \| [`MessageMetadata`](#reacttype-aliasesmessagemetadatamd)\<`StateType`\>

Defined in: [react/stream.tsx:579](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L579)

Get the metadata for a message, such as first thread state the message
was seen in and branch information.

##### Parameters

###### message

`Message`

The message to get the metadata for.

###### index?

`number`

The index of the message in the thread.

##### Returns

`undefined` \| [`MessageMetadata`](#reacttype-aliasesmessagemetadatamd)\<`StateType`\>

The metadata for the message.

***

#### history

> **history**: `ThreadState`\<`StateType`\>[]

Defined in: [react/stream.tsx:552](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L552)

Flattened history of thread states of a thread.

***

#### interrupt

> **interrupt**: `undefined` \| `Interrupt`\<`GetInterruptType`\<`Bag`\>\>

Defined in: [react/stream.tsx:563](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L563)

Get the interrupt value for the stream if interrupted.

***

#### isLoading

> **isLoading**: `boolean`

Defined in: [react/stream.tsx:524](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L524)

Whether the stream is currently running.

***

#### messages

> **messages**: `Message`[]

Defined in: [react/stream.tsx:569](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L569)

Messages inferred from the thread.
Will automatically update with incoming message chunks.

***

#### setBranch()

> **setBranch**: (`branch`) => `void`

Defined in: [react/stream.tsx:547](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L547)

Set the branch of the thread.

##### Parameters

###### branch

`string`

##### Returns

`void`

***

#### stop()

> **stop**: () => `void`

Defined in: [react/stream.tsx:529](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L529)

Stops the stream.

##### Returns

`void`

***

#### submit()

> **submit**: (`values`, `options`?) => `void`

Defined in: [react/stream.tsx:534](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L534)

Create and stream a run to the thread.

##### Parameters

###### values

`undefined` | `null` | `GetUpdateType`\<`Bag`, `StateType`\>

###### options?

`SubmitOptions`\<`StateType`, `GetConfigurableType`\<`Bag`\>\>

##### Returns

`void`

***

#### values

> **values**: `StateType`

Defined in: [react/stream.tsx:514](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L514)

The current values of the thread.


<a name="reactinterfacesusestreamoptionsmd"></a>

[**@langchain/langgraph-sdk**](#reactreadmemd)

***

[@langchain/langgraph-sdk](#reactreadmemd) / UseStreamOptions

## Interface: UseStreamOptions\<StateType, Bag\>

Defined in: [react/stream.tsx:408](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L408)

### Type Parameters

• **StateType** *extends* `Record`\<`string`, `unknown`\> = `Record`\<`string`, `unknown`\>

• **Bag** *extends* `BagTemplate` = `BagTemplate`

### Properties

#### apiKey?

> `optional` **apiKey**: `string`

Defined in: [react/stream.tsx:430](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L430)

The API key to use.

***

#### apiUrl?

> `optional` **apiUrl**: `string`

Defined in: [react/stream.tsx:425](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L425)

The URL of the API to use.

***

#### assistantId

> **assistantId**: `string`

Defined in: [react/stream.tsx:415](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L415)

The ID of the assistant to use.

***

#### callerOptions?

> `optional` **callerOptions**: `AsyncCallerParams`

Defined in: [react/stream.tsx:435](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L435)

Custom call options, such as custom fetch implementation.

***

#### client?

> `optional` **client**: `Client`\<`DefaultValues`, `DefaultValues`, `unknown`\>

Defined in: [react/stream.tsx:420](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L420)

Client used to send requests.

***

#### defaultHeaders?

> `optional` **defaultHeaders**: `Record`\<`string`, `undefined` \| `null` \| `string`\>

Defined in: [react/stream.tsx:440](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L440)

Default headers to send with requests.

***

#### messagesKey?

> `optional` **messagesKey**: `string`

Defined in: [react/stream.tsx:448](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L448)

Specify the key within the state that contains messages.
Defaults to "messages".

##### Default

```ts
"messages"
```

***

#### onCustomEvent()?

> `optional` **onCustomEvent**: (`data`, `options`) => `void`

Defined in: [react/stream.tsx:470](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L470)

Callback that is called when a custom event is received.

##### Parameters

###### data

`GetCustomEventType`\<`Bag`\>

###### options

####### mutate

(`update`) => `void`

##### Returns

`void`

***

#### onDebugEvent()?

> `optional` **onDebugEvent**: (`data`) => `void`

Defined in: [react/stream.tsx:494](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L494)

**`Internal`**

Callback that is called when a debug event is received.
 This API is experimental and subject to change.

##### Parameters

###### data

`unknown`

##### Returns

`void`

***

#### onError()?

> `optional` **onError**: (`error`) => `void`

Defined in: [react/stream.tsx:453](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L453)

Callback that is called when an error occurs.

##### Parameters

###### error

`unknown`

##### Returns

`void`

***

#### onFinish()?

> `optional` **onFinish**: (`state`) => `void`

Defined in: [react/stream.tsx:458](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L458)

Callback that is called when the stream is finished.

##### Parameters

###### state

`ThreadState`\<`StateType`\>

##### Returns

`void`

***

#### onLangChainEvent()?

> `optional` **onLangChainEvent**: (`data`) => `void`

Defined in: [react/stream.tsx:488](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L488)

Callback that is called when a LangChain event is received.

##### Parameters

###### data

####### data

`unknown`

####### event

`string` & `object` \| `"on_tool_start"` \| `"on_tool_stream"` \| `"on_tool_end"` \| `"on_chat_model_start"` \| `"on_chat_model_stream"` \| `"on_chat_model_end"` \| `"on_llm_start"` \| `"on_llm_stream"` \| `"on_llm_end"` \| `"on_chain_start"` \| `"on_chain_stream"` \| `"on_chain_end"` \| `"on_retriever_start"` \| `"on_retriever_stream"` \| `"on_retriever_end"` \| `"on_prompt_start"` \| `"on_prompt_stream"` \| `"on_prompt_end"`

####### metadata

`Record`\<`string`, `unknown`\>

####### name

`string`

####### parent_ids

`string`[]

####### run_id

`string`

####### tags

`string`[]

##### Returns

`void`

##### See

https://langchain-ai.github.io/langgraph/cloud/how-tos/stream_events/#stream-graph-in-events-mode for more details.

***

#### onMetadataEvent()?

> `optional` **onMetadataEvent**: (`data`) => `void`

Defined in: [react/stream.tsx:482](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L482)

Callback that is called when a metadata event is received.

##### Parameters

###### data

####### run_id

`string`

####### thread_id

`string`

##### Returns

`void`

***

#### onThreadId()?

> `optional` **onThreadId**: (`threadId`) => `void`

Defined in: [react/stream.tsx:504](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L504)

Callback that is called when the thread ID is updated (ie when a new thread is created).

##### Parameters

###### threadId

`string`

##### Returns

`void`

***

#### onUpdateEvent()?

> `optional` **onUpdateEvent**: (`data`) => `void`

Defined in: [react/stream.tsx:463](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L463)

Callback that is called when an update event is received.

##### Parameters

###### data

##### Returns

`void`

***

#### threadId?

> `optional` **threadId**: `null` \| `string`

Defined in: [react/stream.tsx:499](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L499)

The ID of the thread to fetch history and current values from.


<a name="reacttype-aliasesmessagemetadatamd"></a>

[**@langchain/langgraph-sdk**](#reactreadmemd)

***

[@langchain/langgraph-sdk](#reactreadmemd) / MessageMetadata

## Type Alias: MessageMetadata\<StateType\>

> **MessageMetadata**\<`StateType`\>: `object`

Defined in: [react/stream.tsx:169](https://github.com/langchain-ai/langgraph/blob/d4f644877db6264bd46d0b00fc4c37f174e502d5/libs/sdk-js/src/react/stream.tsx#L169)

### Type Parameters

• **StateType** *extends* `Record`\<`string`, `unknown`\>

### Type declaration

#### branch

> **branch**: `string` \| `undefined`

The branch of the message.

#### branchOptions

> **branchOptions**: `string`[] \| `undefined`

The list of branches this message is part of.
This is useful for displaying branching controls.

#### firstSeenState

> **firstSeenState**: `ThreadState`\<`StateType`\> \| `undefined`

The first thread state the message was seen in.

#### messageId

> **messageId**: `string`

The ID of the message used.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/reference/sdk/python_sdk_ref.md
```md
# Python SDK Reference

::: langgraph_sdk.client
    handler: python


::: langgraph_sdk.schema
    handler: python

::: langgraph_sdk.auth
    handler: python

::: langgraph_sdk.auth.types
    handler: python

::: langgraph_sdk.auth.exceptions
    handler: python
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/concepts/cron_jobs.md
```md
## Cron jobs

There are many situations in which it is useful to run an assistant on a schedule. 

For example, say that you're building an assistant that runs daily and sends an email summary
of the day's news. You could use a cron job to run the assistant every day at 8:00 PM.

LangGraph Platform supports cron jobs, which run on a user-defined schedule. The user specifies a schedule, an assistant, and some input. After that, on the specified schedule, the server will:

- Create a new thread with the specified assistant
- Send the specified input to that thread

Note that this sends the same input to the thread every time. See the [how-to guide](../../cloud/how-tos/cron_jobs.md) for creating cron jobs.

The LangGraph Platform API provides several endpoints for creating and managing cron jobs. See the [API reference](../../cloud/reference/api/api_ref.html#tag/runscreate/POST/threads/{thread_id}/runs/crons) for more details.
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/concepts/data_storage_and_privacy.md
```md
# Data Storage and Privacy

This document describes how data is processed in the LangGraph CLI and the LangGraph Server for both the in-memory server (`langgraph dev`) and the local Docker server (`langgraph up`). It also describes what data is tracked when interacting with the hosted LangGraph Studio frontend.

## CLI

LangGraph **CLI** is the command-line interface for building and running LangGraph applications; see the [CLI guide](../../concepts/langgraph_cli.md) to learn more.

By default, calls to most CLI commands log a single analytics event upon invocation. This helps us better prioritize improvements to the CLI experience. Each telemetry event contains the calling process's OS, OS version, Python version, the CLI version, the command name (`dev`, `up`, `run`, etc.), and booleans representing whether a flag was passed to the command. You can see the full analytics logic [here](https://github.com/langchain-ai/langgraph/blob/main/libs/cli/langgraph_cli/analytics.py). 

You can disable all CLI telemetry by setting `LANGGRAPH_CLI_NO_ANALYTICS=1`.

## LangGraph Server (in-memory & docker)

The [LangGraph Server](../../concepts/langgraph_server.md) provides a durable execution runtime that relies on persisting checkpoints of your application state, long-term memories, thread metadata, assistants, and similar resources to the local file system or a database. Unless you have deliberately customized the storage location, this information is either written to local disk (for `langgraph dev`) or a PostgreSQL database (for `langgraph up` and in all deployments).

### LangSmith Tracing

When running the LangGraph server (either in-memory or in Docker), LangSmith tracing may be enabled to facilitate faster debugging and offer observability of graph state and LLM prompts in production. You can always disable tracing by setting `LANGSMITH_TRACING=false` in your server's runtime environment.

### In-memory development server (`langgraph dev`)

`langgraph dev` runs an [in-memory development server](../../tutorials/langgraph-platform/local-server.md) as a single Python process, designed for quick development and testing. It saves all checkpointing and memory data to disk within a `.langgraph_api` directory in the current working directory. Apart from the telemetry data described in the [CLI](#cli) section, no data leaves the machine unless you have enabled tracing or your graph code explicitly contacts an external service.

### Standalone Container (`langgraph up`)

`langgraph up` builds your local package into a Docker image and runs the server as a [standalone container](../../concepts/deployment_options.md#standalone-container) consisting of three containers: the API server, a PostgreSQL container, and a Redis container. All persistent data (checkpoints, assistants, etc.) are stored in the PostgreSQL database. Redis is used as a pubsub connection for real-time streaming of events. You can encrypt all checkpoints before saving to the database by setting a valid `LANGGRAPH_AES_KEY` environment variable. You can also specify [TTLs](../../how-tos/ttl/configure_ttl.md) for checkpoints and cross-thread memories in `langgraph.json` to control how long data is stored. All persisted threads, memories, and other data can be deleted via the relevant API endpoints.

Additional API calls are made to confirm that the server has a valid license and to track the number of executed runs and tasks. Periodically, the API server validates the provided license key (or API key).

If you've disabled [tracing](#langsmith-tracing), no user data is persisted externally unless your graph code explicitly contacts an external service.

## Studio

[LangGraph Studio](../../concepts/langgraph_studio.md) is a graphical interface for interacting with your LangGraph server. It does not persist any private data (the data you send to your server is not sent to LangSmith). Though the studio interface is served at [smith.langchain.com](https://smith.langchain.com), it is run in your browser and connects directly to your local LangGraph server so that no data needs to be sent to LangSmith.

If you are logged in, LangSmith does collect some usage analytics to help improve studio's user experience. This includes:

- Page visits and navigation patterns
- User actions (button clicks)
- Browser type and version
- Screen resolution and viewport size

Importantly, no application data or code (or other sensitive configuration details) are collected. All of that is stored in the persistence layer of your LangGraph server. When using Studio anonymously, no account creation is required and usage analytics are not collected.

## Quick reference

In summary, you can opt-out of server-side telemetry by turning off CLI analytics and disabling tracing.

| Variable                       | Purpose                   | Default                          |
| ------------------------------ | ------------------------- | -------------------------------- |
| `LANGGRAPH_CLI_NO_ANALYTICS=1` | Disable CLI analytics     | Analytics enabled                |
| `LANGSMITH_API_KEY`            | Enable LangSmith tracing  | Tracing disabled                 |
| `LANGSMITH_TRACING=false`      | Disable LangSmith tracing | Depends on environment           |

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/concepts/webhooks.md
```md
# Webhooks

Webhooks enable event-driven communication from your LangGraph Platform application to external services. For example, you may want to issue an update to a separate service once an API call to LangGraph Platform has finished running.

Many LangGraph Platform endpoints accept a `webhook` parameter. If this parameter is specified by an endpoint that can accept POST requests, LangGraph Platform will send a request at the completion of a run.

See the corresponding [how-to guide](../../cloud/how-tos/webhooks.md) for more detail.
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/deployment/cloud.md
```md
# How to Deploy to Cloud SaaS

Before deploying, review the [conceptual guide for the Cloud SaaS](../../concepts/langgraph_cloud.md) deployment option.

## Prerequisites

1. LangGraph Platform applications are deployed from GitHub repositories. Configure and upload a LangGraph Platform application to a GitHub repository in order to deploy it to LangGraph Platform.
1. [Verify that the LangGraph API runs locally](../../tutorials/langgraph-platform/local-server.md). If the API does not run successfully (i.e. `langgraph dev`), deploying to LangGraph Platform will fail as well.

## Create New Deployment

Starting from the <a href="https://smith.langchain.com/" target="_blank">LangSmith UI</a>...

1. In the left-hand navigation panel, select `LangGraph Platform`. The `LangGraph Platform` view contains a list of existing LangGraph Platform deployments.
1. In the top-right corner, select `+ New Deployment` to create a new deployment.
1. In the `Create New Deployment` panel, fill out the required fields.
    1. `Deployment details`
        1. Select `Import from GitHub` and follow the GitHub OAuth workflow to install and authorize LangChain's `hosted-langserve` GitHub app to access the selected repositories. After installation is complete, return to the `Create New Deployment` panel and select the GitHub repository to deploy from the dropdown menu. **Note**: The GitHub user installing LangChain's `hosted-langserve` GitHub app must be an [owner](https://docs.github.com/en/organizations/managing-peoples-access-to-your-organization-with-roles/roles-in-an-organization#organization-owners) of the organization or account.
        1. Specify a name for the deployment.
        1. Specify the desired `Git Branch`. A deployment is linked to a branch. When a new revision is created, code for the linked branch will be deployed. The branch can be updated later in the [Deployment Settings](#deployment-settings).
        1. Specify the full path to the [LangGraph API config file](../reference/cli.md#configuration-file) including the file name. For example, if the file `langgraph.json` is in the root of the repository, simply specify `langgraph.json`.
        1. Check/uncheck checkbox to `Automatically update deployment on push to branch`. If checked, the deployment will automatically be updated when changes are pushed to the specified `Git Branch`. This setting can be enabled/disabled later in the [Deployment Settings](#deployment-settings).
    1. Select the desired `Deployment Type`.
        1. `Development` deployments are meant for non-production use cases and are provisioned with minimal resources.
        1. `Production` deployments can serve up to 500 requests/second and are provisioned with highly available storage with automatic backups.
    1. Determine if the deployment should be `Shareable through LangGraph Studio`.
        1. If unchecked, the deployment will only be accessible with a valid LangSmith API key for the workspace.
        1. If checked, the deployment will be accessible through LangGraph Studio to any LangSmith user. A direct URL to LangGraph Studio for the deployment will be provided to share with other LangSmith users.
    1. Specify `Environment Variables` and secrets. See the [Environment Variables reference](../reference/env_var.md) to configure additional variables for the deployment.
        1. Sensitive values such as API keys (e.g. `OPENAI_API_KEY`) should be specified as secrets.
        1. Additional non-secret environment variables can be specified as well.
    1. A new LangSmith `Tracing Project` is automatically created with the same name as the deployment.
1. In the top-right corner, select `Submit`. After a few seconds, the `Deployment` view appears and the new deployment will be queued for provisioning.

## Create New Revision

When [creating a new deployment](#create-new-deployment), a new revision is created by default. Subsequent revisions can be created to deploy new code changes.

Starting from the <a href="https://smith.langchain.com/" target="_blank">LangSmith UI</a>...

1. In the left-hand navigation panel, select `LangGraph Platform`. The `LangGraph Platform` view contains a list of existing LangGraph Platform deployments.
1. Select an existing deployment to create a new revision for.
1. In the `Deployment` view, in the top-right corner, select `+ New Revision`.
1. In the `New Revision` modal, fill out the required fields.
    1. Specify the full path to the [LangGraph API config file](../reference/cli.md#configuration-file) including the file name. For example, if the file `langgraph.json` is in the root of the repository, simply specify `langgraph.json`.
    1. Determine if the deployment should be `Shareable through LangGraph Studio`.
        1. If unchecked, the deployment will only be accessible with a valid LangSmith API key for the workspace.
        1. If checked, the deployment will be accessible through LangGraph Studio to any LangSmith user. A direct URL to LangGraph Studio for the deployment will be provided to share with other LangSmith users.
    1. Specify `Environment Variables` and secrets. Existing secrets and environment variables are prepopulated. See the [Environment Variables reference](../reference/env_var.md) to configure additional variables for the revision.
        1. Add new secrets or environment variables.
        1. Remove existing secrets or environment variables.
        1. Update the value of existing secrets or environment variables.
1. Select `Submit`. After a few seconds, the `New Revision` modal will close and the new revision will be queued for deployment.

## View Build and Server Logs

Build and server logs are available for each revision.

Starting from the `LangGraph Platform` view...

1. Select the desired revision from the `Revisions` table. A panel slides open from the right-hand side and the `Build` tab is selected by default, which displays build logs for the revision.
1. In the panel, select the `Server` tab to view server logs for the revision. Server logs are only available after a revision has been deployed.
1. Within the `Server` tab, adjust the date/time range picker as needed. By default, the date/time range picker is set to the `Last 7 days`.

## View Deployment Metrics

Starting from the <a href="https://smith.langchain.com/" target="_blank">LangSmith UI</a>...

1. In the left-hand navigation panel, select `LangGraph Platform`. The `LangGraph Platform` view contains a list of existing LangGraph Platform deployments.
1. Select an existing deployment to monitor.
1. Select the `Monitoring` tab to view the deployment metrics. See a list of [all available metrics](../../concepts/langgraph_control_plane.md#monitoring).
1. Within the `Monitoring` tab, use the date/time range picker as needed. By default, the date/time range picker is set to the `Last 15 minutes`.

## Interrupt Revision

Interrupting a revision will stop deployment of the revision.

!!! warning "Undefined Behavior"
    Interrupted revisions have undefined behavior. This is only useful if you need to deploy a new revision and you already have a revision "stuck" in progress. In the future, this feature may be removed.

Starting from the `LangGraph Platform` view...

1. Select the menu icon (three dots) on the right-hand side of the row for the desired revision from the `Revisions` table.
1. Select `Interrupt` from the menu.
1. A modal will appear. Review the confirmation message. Select `Interrupt revision`.

## Delete Deployment

Starting from the <a href="https://smith.langchain.com/" target="_blank">LangSmith UI</a>...

1. In the left-hand navigation panel, select `LangGraph Platform`. The `LangGraph Platform` view contains a list of existing LangGraph Platform deployments.
1. Select the menu icon (three dots) on the right-hand side of the row for the desired deployment and select `Delete`.
1. A `Confirmation` modal will appear. Select `Delete`.

## Deployment Settings

Starting from the `LangGraph Platform` view...

1. In the top-right corner, select the gear icon (`Deployment Settings`).
1. Update the `Git Branch` to the desired branch.
1. Check/uncheck checkbox to `Automatically update deployment on push to branch`.
1. Branch creation/deletion and tag creation/deletion events will not trigger an update. Only pushes to an existing branch will trigger an update.
1. Pushes in quick succession to a branch will queue subsequent updates. Once a build completes, the most recent commit will begin building and the other queued builds will be skipped.

## Add or Remove GitHub Repositories

After installing and authorizing LangChain's `hosted-langserve` GitHub app, repository access for the app can be modified to add new repositories or remove existing repositories. If a new repository is created, it may need to be added explicitly.

1. From the GitHub profile, navigate to `Settings` > `Applications` > `hosted-langserve` > click `Configure`.
1. Under `Repository access`, select `All repositories` or `Only select repositories`. If `Only select repositories` is selected, new repositories must be explicitly added.
1. Click `Save`.
1. When creating a new deployment, the list of GitHub repositories in the dropdown menu will be updated to reflect the repository access changes.

## Whitelisting IP Addresses

All traffic from `LangGraph Platform` deployments created after January 6th 2025 will come through a NAT gateway.
This NAT gateway will have several static ip addresses depending on the region you are deploying in. Refer to the table below for the list of IP addresses to whitelist:

| US             | EU              |
|----------------|-----------------|
| 35.197.29.146  | 34.90.213.236   |
| 34.145.102.123 | 34.13.244.114   |
| 34.169.45.153  | 34.32.180.189   |
| 34.82.222.17   | 34.34.69.108    |
| 35.227.171.135 | 34.32.145.240   | 
| 34.169.88.30   | 34.90.157.44    |
| 34.19.93.202   | 34.141.242.180  |
| 34.19.34.50    | 34.32.141.108   |

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/deployment/custom_docker.md
```md
# How to customize Dockerfile

Users can add an array of additional lines to add to the Dockerfile following the import from the parent LangGraph image. In order to do this, you simply need to modify your `langgraph.json` file by passing in the commands you want run to the `dockerfile_lines` key. For example, if we wanted to use `Pillow` in our graph you would need to add the following dependencies:

```
{
    "dependencies": ["."],
    "graphs": {
        "openai_agent": "./openai_agent.py:agent",
    },
    "env": "./.env",
    "dockerfile_lines": [
        "RUN apt-get update && apt-get install -y libjpeg-dev zlib1g-dev libpng-dev",
        "RUN pip install Pillow"
    ]
}
```

This would install the system packages required to use Pillow if we were working with `jpeg` or `png` image formats. 
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/deployment/egress.md
```md
# Egress for Subscription Metrics and Operational Metadata

> **Important: Self Hosted Only** 
> This section only applies to customers who are not running in offline mode and assumes you are using a self-hosted LangGraph Platform instance.
> This does not apply to SaaS or Hybrid deployments.

Self-Hosted LangGraph Platform instances store all information locally and will never send sensitive information outside of your network. We currently only track platform usage for billing purposes according to the entitlements in your order. In order to better remotely support our customers, we do require egress to `https://beacon.langchain.com`.

In the future, we will be introducing support diagnostics to help us ensure that the LangGraph Platform is running at an optimal level within your environment.

> **Warning**  
> **This will require egress to `https://beacon.langchain.com` from your network.**
> **If using an API key, you will also need to allow egress to `https://api.smith.langchain.com` or `https://eu.api.smith.langchain.com` for API key verification.**

Generally, data that we send to Beacon can be categorized as follows:

- **Subscription Metrics**
  - Subscription metrics are used to determine level of access and utilization of LangSmith. This includes, but are not limited to:
    - Nodes Executed
    - Runs Executed
    - License Key Verification
- **Operational Metadata**
  - This metadata will contain and collect the above subscription metrics to assist with remote support, allowing the LangChain team to diagnose and troubleshoot performance issues more effectively and proactively.

## Example Payloads

In an effort to maximize transparency, we provide sample payloads here:

### License Verification (If using an Enterprise License)

**Endpoint:**

`POST beacon.langchain.com/v1/beacon/verify`

**Request:**

```json
{
  "license": "<YOUR_LICENSE_KEY>"
}
```

**Response:**

```json
{
  "token": "Valid JWT" // Short-lived JWT token to avoid repeated license checks
}
```

### Api Key Verification (If using a LangSmith API Key)

**Endpoint:**
`POST api.smith.langchain.com/auth`

**Request:**

```json
"Headers": {
  X-Api-Key: <YOUR_API_KEY>
}
```

**Response:**

```json
{
  "org_config": {
    "org_id": "3a1c2b6f-4430-4b92-8a5b-79b8b567bbc1",
    ... // Additional organization details
  }
}
```

### Usage Reporting

**Endpoint:**

`POST beacon.langchain.com/v1/metadata/submit`

**Request:**

```json
{
  "license": "<YOUR_LICENSE_KEY>",
  "from_timestamp": "2025-01-06T09:00:00Z",
  "to_timestamp": "2025-01-06T10:00:00Z",
  "tags": {
    "langgraph.python.version": "0.1.0",
    "langgraph_api.version": "0.2.0",
    "langgraph.platform.revision": "abc123",
    "langgraph.platform.variant": "standard",
    "langgraph.platform.host": "host-1",
    "langgraph.platform.tenant_id": "3a1c2b6f-4430-4b92-8a5b-79b8b567bbc1",
    "langgraph.platform.project_id": "c5b5f53a-4716-4326-8967-d4f7f7799735",
    "langgraph.platform.plan": "enterprise",
    "user_app.uses_indexing": "true",
    "user_app.uses_custom_app": "false",
    "user_app.uses_custom_auth": "true",
    "user_app.uses_thread_ttl": "true",
    "user_app.uses_store_ttl": "false"
  },
  "measures": {
    "langgraph.platform.runs": 150,
    "langgraph.platform.nodes": 450
  },
  "logs": []
}
```

**Response:**

```json
"204 No Content"
```

## Our Commitment

LangChain will not store any sensitive information in the Subscription Metrics or Operational Metadata. Any data collected will not be shared with a third party. If you have any concerns about the data being sent, please reach out to your account team.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/deployment/graph_rebuild.md
```md
# Rebuild Graph at Runtime

You might need to rebuild your graph with a different configuration for a new run. For example, you might need to use a different graph state or graph structure depending on the config. This guide shows how you can do this.

!!! note "Note"
    In most cases, customizing behavior based on the config should be handled by a single graph where each node can read a config and change its behavior based on it

## Prerequisites

Make sure to check out [this how-to guide](./setup.md) on setting up your app for deployment first.

## Define graphs

Let's say you have an app with a simple graph that calls an LLM and returns the response to the user. The app file directory looks like the following:

```
my-app/
|-- requirements.txt
|-- .env
|-- openai_agent.py     # code for your graph
```

where the graph is defined in `openai_agent.py`. 

### No rebuild

In the standard LangGraph API configuration, the server uses the compiled graph instance that's defined at the top level of `openai_agent.py`, which looks like the following:

```python
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, MessageGraph

model = ChatOpenAI(temperature=0)

graph_workflow = MessageGraph()

graph_workflow.add_node("agent", model)
graph_workflow.add_edge("agent", END)
graph_workflow.add_edge(START, "agent")

agent = graph_workflow.compile()
```

To make the server aware of your graph, you need to specify a path to the variable that contains the `CompiledStateGraph` instance in your LangGraph API configuration (`langgraph.json`), e.g.:

```
{
    "dependencies": ["."],
    "graphs": {
        "openai_agent": "./openai_agent.py:agent",
    },
    "env": "./.env"
}
```

### Rebuild

To make your graph rebuild on each new run with custom configuration, you need to rewrite `openai_agent.py` to instead provide a _function_ that takes a config and returns a graph (or compiled graph) instance. Let's say we want to return our existing graph for user ID '1', and a tool-calling agent for other users. We can modify `openai_agent.py` as follows:

```python
from typing import Annotated
from typing_extensions import TypedDict
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, MessageGraph
from langgraph.graph.state import StateGraph
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode
from langchain_core.tools import tool
from langchain_core.messages import BaseMessage
from langchain_core.runnables import RunnableConfig


class State(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]


model = ChatOpenAI(temperature=0)

def make_default_graph():
    """Make a simple LLM agent"""
    graph_workflow = StateGraph(State)
    def call_model(state):
        return {"messages": [model.invoke(state["messages"])]}

    graph_workflow.add_node("agent", call_model)
    graph_workflow.add_edge("agent", END)
    graph_workflow.add_edge(START, "agent")

    agent = graph_workflow.compile()
    return agent


def make_alternative_graph():
    """Make a tool-calling agent"""

    @tool
    def add(a: float, b: float):
        """Adds two numbers."""
        return a + b

    tool_node = ToolNode([add])
    model_with_tools = model.bind_tools([add])
    def call_model(state):
        return {"messages": [model_with_tools.invoke(state["messages"])]}

    def should_continue(state: State):
        if state["messages"][-1].tool_calls:
            return "tools"
        else:
            return END

    graph_workflow = StateGraph(State)

    graph_workflow.add_node("agent", call_model)
    graph_workflow.add_node("tools", tool_node)
    graph_workflow.add_edge("tools", "agent")
    graph_workflow.add_edge(START, "agent")
    graph_workflow.add_conditional_edges("agent", should_continue)

    agent = graph_workflow.compile()
    return agent


# this is the graph making function that will decide which graph to
# build based on the provided config
def make_graph(config: RunnableConfig):
    user_id = config.get("configurable", {}).get("user_id")
    # route to different graph state / structure based on the user ID
    if user_id == "1":
        return make_default_graph()
    else:
        return make_alternative_graph()
```

Finally, you need to specify the path to your graph-making function (`make_graph`) in `langgraph.json`:

```
{
    "dependencies": ["."],
    "graphs": {
        "openai_agent": "./openai_agent.py:make_graph",
    },
    "env": "./.env"
}
```

See more info on LangGraph API configuration file [here](../reference/cli.md#configuration-file)
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/deployment/self_hosted_control_plane.md
```md
# How to Deploy Self-Hosted Control Plane

Before deploying, review the [conceptual guide for the Self-Hosted Control Plane](../../concepts/langgraph_self_hosted_control_plane.md) deployment option.

!!! info "Important"
    The Self-Hosted Control Plane deployment option requires an [Enterprise](../../concepts/plans.md) plan.

## Prerequisites

1. You are using Kubernetes.
1. You have self-hosted LangSmith deployed.
1. Use the [LangGraph CLI](../../concepts/langgraph_cli.md) to [test your application locally](../../tutorials/langgraph-platform/local-server.md).
1. Use the [LangGraph CLI](../../concepts/langgraph_cli.md) to build a Docker image (i.e. `langgraph build`) and push it to a registry your Kubernetes cluster has access to.
1. `KEDA` is installed on your cluster.

         helm repo add kedacore https://kedacore.github.io/charts 
         helm install keda kedacore/keda --namespace keda --create-namespace
1. Ingress Configuration
    1. You must set up an ingress for your LangSmith instance. All agents will be deployed as Kubernetes services behind this ingress.
    1. You can use this guide to [set up an ingress](https://docs.smith.langchain.com/self_hosting/configuration/ingress) for your instance.
1. You have slack space in your cluster for multiple deployments. `Cluster-Autoscaler` is recommended to automatically provision new nodes.
1. A valid Dynamic PV provisioner or PVs available on your cluster. You can verify this by running:

        kubectl get storageclass

1. Egress to `https://beacon.langchain.com` from your network. This is required for license verification and usage reporting if not running in air-gapped mode. See the [Egress documentation](../../cloud/deployment/egress.md) for more details.

## Setup

1. As part of configuring your Self-Hosted LangSmith instance, you enable the `langgraphPlatform` option. This will provision a few key resources.
    1. `listener`: This is a service that listens to the [control plane](../../concepts/langgraph_control_plane.md) for changes to your deployments and creates/updates downstream CRDs.
    1. `LangGraphPlatform CRD`: A CRD for LangGraph Platform deployments. This contains the spec for managing an instance of a LangGraph platform deployment.
    1. `operator`: This operator handles changes to your LangGraph Platform CRDs.
    1. `host-backend`: This is the [control plane](../../concepts/langgraph_control_plane.md).
1. Two additional images will be used by the chart. Use the images that are specified in the latest release.

        hostBackendImage:
          repository: "docker.io/langchain/hosted-langserve-backend"
          pullPolicy: IfNotPresent
        operatorImage:
          repository: "docker.io/langchain/langgraph-operator"
          pullPolicy: IfNotPresent

1. In your config file for langsmith (usually `langsmith_config.yaml`, enable the `langgraphPlatform` option. Note that you must also have a valid ingress setup:

        config:
          langgraphPlatform:
            enabled: true
            langgraphPlatformLicenseKey: "YOUR_LANGGRAPH_PLATFORM_LICENSE_KEY"
1. In your `values.yaml` file, configure the `hostBackendImage` and `operatorImage` options (if you need to mirror images)

1. You can also configure base templates for your agents by overriding the base templates [here](https://github.com/langchain-ai/helm/blob/main/charts/langsmith/values.yaml#L898).
1. You create a deployment from the [control plane UI](../../concepts/langgraph_control_plane.md#control-plane-ui).

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/deployment/self_hosted_data_plane.md
```md
# How to Deploy Self-Hosted Data Plane

Before deploying, review the [conceptual guide for the Self-Hosted Data Plane](../../concepts/langgraph_self_hosted_data_plane.md) deployment option.

!!! info "Important"
    The Self-Hosted Data Plane deployment option requires an [Enterprise](../../concepts/plans.md) plan.

## Prerequisites

1. Use the [LangGraph CLI](../../concepts/langgraph_cli.md) to [test your application locally](../../tutorials/langgraph-platform/local-server.md).
1. Use the [LangGraph CLI](../../concepts/langgraph_cli.md) to build a Docker image (i.e. `langgraph build`) and push it to a registry your Kubernetes cluster or Amazon ECS cluster has access to.

## Kubernetes

### Prerequisites
1. `KEDA` is installed on your cluster.

        helm repo add kedacore https://kedacore.github.io/charts
        helm install keda kedacore/keda --namespace keda --create-namespace

1. A valid `Ingress` controller is installed on your cluster.
1. You have slack space in your cluster for multiple deployments. `Cluster-Autoscaler` is recommended to automatically provision new nodes.
1. You will need to enable egress to two control plane URLs. The listener polls these endpoints for deployments:

        https://api.host.langchain.com
        https://api.smith.langchain.com

### Setup

1. You give us your LangSmith organization ID. We will enable the Self-Hosted Data Plane for your organization.
1. We provide you a [Helm chart](https://github.com/langchain-ai/helm/tree/main/charts/langgraph-dataplane) which you run to setup your Kubernetes cluster. This chart contains a few important components.
    1. `langgraph-listener`: This is a service that listens to LangChain's [control plane](../../concepts/langgraph_control_plane.md) for changes to your deployments and creates/updates downstream CRDs.
    1. `LangGraphPlatform CRD`: A CRD for LangGraph Platform deployments. This contains the spec for managing an instance of a LangGraph Platform deployment.
    1. `langgraph-platform-operator`: This operator handles changes to your LangGraph Platform CRDs.
1. Configure your `langgraph-dataplane-values.yaml` file.

        config:
          langsmithApiKey: "" # API Key of your Workspace
          langsmithWorkspaceId: "" # Workspace ID
          hostBackendUrl: "https://api.host.langchain.com" # Only override this if on EU
          smithBackendUrl: "https://api.smith.langchain.com" # Only override this if on EU

1. Deploy `langgraph-dataplane` Helm chart.

        helm repo add langchain https://langchain-ai.github.io/helm/
        helm repo update
        helm upgrade -i langgraph-dataplane langchain/langgraph-dataplane --values langgraph-dataplane-values.yaml

1. If successful, you will see two services start up in your namespace.

        NAME                                          READY   STATUS              RESTARTS   AGE
        langgraph-dataplane-listener-7fccd788-wn2dx   0/1     Running             0          9s
        langgraph-dataplane-redis-0                   0/1     ContainerCreating   0          9s

1. You create a deployment from the [control plane UI](../../concepts/langgraph_control_plane.md#control-plane-ui).

## Amazon ECS

Coming soon!

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/deployment/semantic_search.md
```md
# How to add semantic search to your LangGraph deployment

This guide explains how to add semantic search to your LangGraph deployment's cross-thread [store](../../concepts/persistence.md#memory-store), so that your agent can search for memories and other documents by semantic similarity.

## Prerequisites

- A LangGraph deployment (see [how to deploy](setup_pyproject.md))
- API keys for your embedding provider (in this case, OpenAI)
- `langchain >= 0.3.8` (if you specify using the string format below)

## Steps

1. Update your `langgraph.json` configuration file to include the store configuration:

```json
{
    ...
    "store": {
        "index": {
            "embed": "openai:text-embedding-3-small",
            "dims": 1536,
            "fields": ["$"]
        }
    }
}
```

This configuration:

- Uses OpenAI's text-embedding-3-small model for generating embeddings
- Sets the embedding dimension to 1536 (matching the model's output)
- Indexes all fields in your stored data (`["$"]` means index everything, or specify specific fields like `["text", "metadata.title"]`)

2. To use the string embedding format above, make sure your dependencies include `langchain >= 0.3.8`:

```toml
# In pyproject.toml
[project]
dependencies = [
    "langchain>=0.3.8"
]
```

Or if using requirements.txt:

```
langchain>=0.3.8
```

## Usage

Once configured, you can use semantic search in your LangGraph nodes. The store requires a namespace tuple to organize memories:

```python
def search_memory(state: State, *, store: BaseStore):
    # Search the store using semantic similarity
    # The namespace tuple helps organize different types of memories
    # e.g., ("user_facts", "preferences") or ("conversation", "summaries")
    results = store.search(
        namespace=("memory", "facts"),  # Organize memories by type
        query="your search query",
        limit=3  # number of results to return
    )
    return results
```

## Custom Embeddings

If you want to use custom embeddings, you can pass a path to a custom embedding function:

```json
{
    ...
    "store": {
        "index": {
            "embed": "path/to/embedding_function.py:embed",
            "dims": 1536,
            "fields": ["$"]
        }
    }
}
```

The deployment will look for the function in the specified path. The function must be async and accept a list of strings:

```python
# path/to/embedding_function.py
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def aembed_texts(texts: list[str]) -> list[list[float]]:
    """Custom embedding function that must:
    1. Be async
    2. Accept a list of strings
    3. Return a list of float arrays (embeddings)
    """
    response = await client.embeddings.create(
        model="text-embedding-3-small",
        input=texts
    )
    return [e.embedding for e in response.data]
```

## Querying via the API

You can also query the store using the LangGraph SDK. Since the SDK uses async operations:

```python
from langgraph_sdk import get_client

async def search_store():
    client = get_client()
    results = await client.store.search_items(
        ("memory", "facts"),
        query="your search query",
        limit=3  # number of results to return
    )
    return results

# Use in an async context
results = await search_store()
```

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/deployment/setup.md
```md
# How to Set Up a LangGraph Application with requirements.txt

A LangGraph application must be configured with a [LangGraph configuration file](../reference/cli.md#configuration-file) in order to be deployed to LangGraph Platform (or to be self-hosted). This how-to guide discusses the basic steps to setup a LangGraph application for deployment using `requirements.txt` to specify project dependencies.

This walkthrough is based on [this repository](https://github.com/langchain-ai/langgraph-example), which you can play around with to learn more about how to setup your LangGraph application for deployment.

!!! tip "Setup with pyproject.toml"
    If you prefer using poetry for dependency management, check out [this how-to guide](./setup_pyproject.md) on using `pyproject.toml` for LangGraph Platform.

!!! tip "Setup with a Monorepo"
    If you are interested in deploying a graph located inside a monorepo, take a look at [this repository](https://github.com/langchain-ai/langgraph-example-monorepo) for an example of how to do so.

The final repository structure will look something like this:

```bash
my-app/
├── my_agent # all project code lies within here
│   ├── utils # utilities for your graph
│   │   ├── __init__.py
│   │   ├── tools.py # tools for your graph
│   │   ├── nodes.py # node functions for you graph
│   │   └── state.py # state definition of your graph
│   ├── requirements.txt # package dependencies
│   ├── __init__.py
│   └── agent.py # code for constructing your graph
├── .env # environment variables
└── langgraph.json # configuration file for LangGraph
```

After each step, an example file directory is provided to demonstrate how code can be organized.

## Specify Dependencies

Dependencies can optionally be specified in one of the following files: `pyproject.toml`, `setup.py`, or `requirements.txt`. If none of these files is created, then dependencies can be specified later in the [LangGraph configuration file](#create-langgraph-configuration-file).

The dependencies below will be included in the image, you can also use them in your code, as long as with a compatible version range:

```
langgraph>=0.3.27
langgraph-sdk>=0.1.66
langgraph-checkpoint>=2.0.23
langchain-core>=0.2.38
langsmith>=0.1.63
orjson>=3.9.7,<3.10.17
httpx>=0.25.0
tenacity>=8.0.0
uvicorn>=0.26.0
sse-starlette>=2.1.0,<2.2.0
uvloop>=0.18.0
httptools>=0.5.0
jsonschema-rs>=0.20.0
structlog>=24.1.0
cloudpickle>=3.0.0
```

Example `requirements.txt` file:

```
langgraph
langchain_anthropic
tavily-python
langchain_community
langchain_openai

```

Example file directory:

```bash
my-app/
├── my_agent # all project code lies within here
│   └── requirements.txt # package dependencies
```

## Specify Environment Variables

Environment variables can optionally be specified in a file (e.g. `.env`). See the [Environment Variables reference](../reference/env_var.md) to configure additional variables for a deployment.

Example `.env` file:

```
MY_ENV_VAR_1=foo
MY_ENV_VAR_2=bar
OPENAI_API_KEY=key
```

Example file directory:

```bash
my-app/
├── my_agent # all project code lies within here
│   └── requirements.txt # package dependencies
└── .env # environment variables
```

## Define Graphs

Implement your graphs! Graphs can be defined in a single file or multiple files. Make note of the variable names of each @[CompiledStateGraph][CompiledStateGraph] to be included in the LangGraph application. The variable names will be used later when creating the [LangGraph configuration file](../reference/cli.md#configuration-file).

Example `agent.py` file, which shows how to import from other modules you define (code for the modules is not shown here, please see [this repository](https://github.com/langchain-ai/langgraph-example) to see their implementation):

```python
# my_agent/agent.py
from typing import Literal
from typing_extensions import TypedDict

from langgraph.graph import StateGraph, END, START
from my_agent.utils.nodes import call_model, should_continue, tool_node # import nodes
from my_agent.utils.state import AgentState # import state

# Define the runtime context
class GraphContext(TypedDict):
    model_name: Literal["anthropic", "openai"]

workflow = StateGraph(AgentState, context_schema=GraphContext)
workflow.add_node("agent", call_model)
workflow.add_node("action", tool_node)
workflow.add_edge(START, "agent")
workflow.add_conditional_edges(
    "agent",
    should_continue,
    {
        "continue": "action",
        "end": END,
    },
)
workflow.add_edge("action", "agent")

graph = workflow.compile()
```

Example file directory:

```bash
my-app/
├── my_agent # all project code lies within here
│   ├── utils # utilities for your graph
│   │   ├── __init__.py
│   │   ├── tools.py # tools for your graph
│   │   ├── nodes.py # node functions for you graph
│   │   └── state.py # state definition of your graph
│   ├── requirements.txt # package dependencies
│   ├── __init__.py
│   └── agent.py # code for constructing your graph
└── .env # environment variables
```

## Create LangGraph Configuration File

Create a [LangGraph configuration file](../reference/cli.md#configuration-file) called `langgraph.json`. See the [LangGraph configuration file reference](../reference/cli.md#configuration-file) for detailed explanations of each key in the JSON object of the configuration file.

Example `langgraph.json` file:

```json
{
  "dependencies": ["./my_agent"],
  "graphs": {
    "agent": "./my_agent/agent.py:graph"
  },
  "env": ".env"
}
```

Note that the variable name of the `CompiledGraph` appears at the end of the value of each subkey in the top-level `graphs` key (i.e. `:<variable_name>`).

!!! warning "Configuration File Location"
    The LangGraph configuration file must be placed in a directory that is at the same level or higher than the Python files that contain compiled graphs and associated dependencies.

Example file directory:

```bash
my-app/
├── my_agent # all project code lies within here
│   ├── utils # utilities for your graph
│   │   ├── __init__.py
│   │   ├── tools.py # tools for your graph
│   │   ├── nodes.py # node functions for you graph
│   │   └── state.py # state definition of your graph
│   ├── requirements.txt # package dependencies
│   ├── __init__.py
│   └── agent.py # code for constructing your graph
├── .env # environment variables
└── langgraph.json # configuration file for LangGraph
```

## Next

After you setup your project and place it in a GitHub repository, it's time to [deploy your app](./cloud.md).

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/deployment/setup_javascript.md
```md
# How to Set Up a LangGraph.js Application

A [LangGraph.js](https://langchain-ai.github.io/langgraphjs/) application must be configured with a [LangGraph configuration file](../reference/cli.md#configuration-file) in order to be deployed to LangGraph Platform (or to be self-hosted). This how-to guide discusses the basic steps to setup a LangGraph.js application for deployment using `package.json` to specify project dependencies.

This walkthrough is based on [this repository](https://github.com/langchain-ai/langgraphjs-studio-starter), which you can play around with to learn more about how to setup your LangGraph application for deployment.

The final repository structure will look something like this:

```bash
my-app/
├── src # all project code lies within here
│   ├── utils # optional utilities for your graph
│   │   ├── tools.ts # tools for your graph
│   │   ├── nodes.ts # node functions for you graph
│   │   └── state.ts # state definition of your graph
│   └── agent.ts # code for constructing your graph
├── package.json # package dependencies
├── .env # environment variables
└── langgraph.json # configuration file for LangGraph
```

After each step, an example file directory is provided to demonstrate how code can be organized.

## Specify Dependencies

Dependencies can be specified in a `package.json`. If none of these files is created, then dependencies can be specified later in the [LangGraph configuration file](#create-langgraph-api-config).

Example `package.json` file:

```json
{
  "name": "langgraphjs-studio-starter",
  "packageManager": "yarn@1.22.22",
  "dependencies": {
    "@langchain/community": "^0.2.31",
    "@langchain/core": "^0.2.31",
    "@langchain/langgraph": "^0.2.0",
    "@langchain/openai": "^0.2.8"
  }
}
```

When deploying your app, the dependencies will be installed using the package manager of your choice, provided they adhere to the compatible version ranges listed below:

```
"@langchain/core": "^0.3.42",
"@langchain/langgraph": "^0.2.57",
"@langchain/langgraph-checkpoint": "~0.0.16",
```

Example file directory:

```bash
my-app/
└── package.json # package dependencies
```

## Specify Environment Variables

Environment variables can optionally be specified in a file (e.g. `.env`). See the [Environment Variables reference](../reference/env_var.md) to configure additional variables for a deployment.

Example `.env` file:

```
MY_ENV_VAR_1=foo
MY_ENV_VAR_2=bar
OPENAI_API_KEY=key
TAVILY_API_KEY=key_2
```

Example file directory:

```bash
my-app/
├── package.json
└── .env # environment variables
```

## Define Graphs

Implement your graphs! Graphs can be defined in a single file or multiple files. Make note of the variable names of each compiled graph to be included in the LangGraph application. The variable names will be used later when creating the [LangGraph configuration file](../reference/cli.md#configuration-file).

Here is an example `agent.ts`:

```ts
import type { AIMessage } from "@langchain/core/messages";
import { TavilySearchResults } from "@langchain/community/tools/tavily_search";
import { ChatOpenAI } from "@langchain/openai";

import { MessagesAnnotation, StateGraph } from "@langchain/langgraph";
import { ToolNode } from "@langchain/langgraph/prebuilt";

const tools = [new TavilySearchResults({ maxResults: 3 })];

// Define the function that calls the model
async function callModel(state: typeof MessagesAnnotation.State) {
  /**
   * Call the LLM powering our agent.
   * Feel free to customize the prompt, model, and other logic!
   */
  const model = new ChatOpenAI({
    model: "gpt-4o",
  }).bindTools(tools);

  const response = await model.invoke([
    {
      role: "system",
      content: `You are a helpful assistant. The current date is ${new Date().getTime()}.`,
    },
    ...state.messages,
  ]);

  // MessagesAnnotation supports returning a single message or array of messages
  return { messages: response };
}

// Define the function that determines whether to continue or not
function routeModelOutput(state: typeof MessagesAnnotation.State) {
  const messages = state.messages;
  const lastMessage: AIMessage = messages[messages.length - 1];
  // If the LLM is invoking tools, route there.
  if ((lastMessage?.tool_calls?.length ?? 0) > 0) {
    return "tools";
  }
  // Otherwise end the graph.
  return "__end__";
}

// Define a new graph.
// See https://langchain-ai.github.io/langgraphjs/how-tos/define-state/#getting-started for
// more on defining custom graph states.
const workflow = new StateGraph(MessagesAnnotation)
  // Define the two nodes we will cycle between
  .addNode("callModel", callModel)
  .addNode("tools", new ToolNode(tools))
  // Set the entrypoint as `callModel`
  // This means that this node is the first one called
  .addEdge("__start__", "callModel")
  .addConditionalEdges(
    // First, we define the edges' source node. We use `callModel`.
    // This means these are the edges taken after the `callModel` node is called.
    "callModel",
    // Next, we pass in the function that will determine the sink node(s), which
    // will be called after the source node is called.
    routeModelOutput,
    // List of the possible destinations the conditional edge can route to.
    // Required for conditional edges to properly render the graph in Studio
    ["tools", "__end__"]
  )
  // This means that after `tools` is called, `callModel` node is called next.
  .addEdge("tools", "callModel");

// Finally, we compile it!
// This compiles it into a graph you can invoke and deploy.
export const graph = workflow.compile();
```

Example file directory:

```bash
my-app/
├── src # all project code lies within here
│   ├── utils # optional utilities for your graph
│   │   ├── tools.ts # tools for your graph
│   │   ├── nodes.ts # node functions for you graph
│   │   └── state.ts # state definition of your graph
│   └── agent.ts # code for constructing your graph
├── package.json # package dependencies
├── .env # environment variables
└── langgraph.json # configuration file for LangGraph
```

## Create LangGraph API Config

Create a [LangGraph configuration file](../reference/cli.md#configuration-file) called `langgraph.json`. See the [LangGraph configuration file reference](../reference/cli.md#configuration-file) for detailed explanations of each key in the JSON object of the configuration file.

Example `langgraph.json` file:

```json
{
  "node_version": "20",
  "dockerfile_lines": [],
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent.ts:graph"
  },
  "env": ".env"
}
```

Note that the variable name of the `CompiledGraph` appears at the end of the value of each subkey in the top-level `graphs` key (i.e. `:<variable_name>`).

!!! info "Configuration Location"

    The LangGraph configuration file must be placed in a directory that is at the same level or higher than the TypeScript files that contain compiled graphs and associated dependencies.

## Next

After you setup your project and place it in a GitHub repository, it's time to [deploy your app](./cloud.md).

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/deployment/setup_pyproject.md
```md
# How to Set Up a LangGraph Application with pyproject.toml

A LangGraph application must be configured with a [LangGraph configuration file](../reference/cli.md#configuration-file) in order to be deployed to LangGraph Platform (or to be self-hosted). This how-to guide discusses the basic steps to setup a LangGraph application for deployment using `pyproject.toml` to define your package's dependencies.

This walkthrough is based on [this repository](https://github.com/langchain-ai/langgraph-example-pyproject), which you can play around with to learn more about how to setup your LangGraph application for deployment.

!!! tip "Setup with requirements.txt"
    If you prefer using `requirements.txt` for dependency management, check out [this how-to guide](./setup.md).

!!! tip "Setup with a Monorepo"
    If you are interested in deploying a graph located inside a monorepo, take a look at [this](https://github.com/langchain-ai/langgraph-example-monorepo) repository for an example of how to do so.

The final repository structure will look something like this:

```bash
my-app/
├── my_agent # all project code lies within here
│   ├── utils # utilities for your graph
│   │   ├── __init__.py
│   │   ├── tools.py # tools for your graph
│   │   ├── nodes.py # node functions for you graph
│   │   └── state.py # state definition of your graph
│   ├── __init__.py
│   └── agent.py # code for constructing your graph
├── .env # environment variables
├── langgraph.json  # configuration file for LangGraph
└── pyproject.toml # dependencies for your project
```

After each step, an example file directory is provided to demonstrate how code can be organized.

## Specify Dependencies

Dependencies can optionally be specified in one of the following files: `pyproject.toml`, `setup.py`, or `requirements.txt`. If none of these files is created, then dependencies can be specified later in the [LangGraph configuration file](#create-langgraph-configuration-file).

The dependencies below will be included in the image, you can also use them in your code, as long as with a compatible version range:

```
langgraph>=0.3.27
langgraph-sdk>=0.1.66
langgraph-checkpoint>=2.0.23
langchain-core>=0.2.38
langsmith>=0.1.63
orjson>=3.9.7,<3.10.17
httpx>=0.25.0
tenacity>=8.0.0
uvicorn>=0.26.0
sse-starlette>=2.1.0,<2.2.0
uvloop>=0.18.0
httptools>=0.5.0
jsonschema-rs>=0.20.0
structlog>=24.1.0
cloudpickle>=3.0.0
```

Example `pyproject.toml` file:

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-agent"
version = "0.0.1"
description = "An excellent agent build for LangGraph Platform."
authors = [
    {name = "Polly the parrot", email = "1223+polly@users.noreply.github.com"}
]
license = {text = "MIT"}
readme = "README.md"
requires-python = ">=3.9"
dependencies = [
    "langgraph>=0.2.0",
    "langchain-fireworks>=0.1.3"
]

[tool.hatch.build.targets.wheel]
packages = ["my_agent"]
```

Example file directory:

```bash
my-app/
└── pyproject.toml   # Python packages required for your graph
```

## Specify Environment Variables

Environment variables can optionally be specified in a file (e.g. `.env`). See the [Environment Variables reference](../reference/env_var.md) to configure additional variables for a deployment.

Example `.env` file:

```
MY_ENV_VAR_1=foo
MY_ENV_VAR_2=bar
FIREWORKS_API_KEY=key
```

Example file directory:

```bash
my-app/
├── .env # file with environment variables
└── pyproject.toml
```

## Define Graphs

Implement your graphs! Graphs can be defined in a single file or multiple files. Make note of the variable names of each @[CompiledStateGraph][CompiledStateGraph] to be included in the LangGraph application. The variable names will be used later when creating the [LangGraph configuration file](../reference/cli.md#configuration-file).

Example `agent.py` file, which shows how to import from other modules you define (code for the modules is not shown here, please see [this repository](https://github.com/langchain-ai/langgraph-example-pyproject) to see their implementation):

```python
# my_agent/agent.py
from typing import Literal
from typing_extensions import TypedDict

from langgraph.graph import StateGraph, END, START
from my_agent.utils.nodes import call_model, should_continue, tool_node # import nodes
from my_agent.utils.state import AgentState # import state

# Define the runtime context
class GraphContext(TypedDict):
    model_name: Literal["anthropic", "openai"]

workflow = StateGraph(AgentState, context_schema=GraphContext)
workflow.add_node("agent", call_model)
workflow.add_node("action", tool_node)
workflow.add_edge(START, "agent")
workflow.add_conditional_edges(
    "agent",
    should_continue,
    {
        "continue": "action",
        "end": END,
    },
)
workflow.add_edge("action", "agent")

graph = workflow.compile()
```

Example file directory:

```bash
my-app/
├── my_agent # all project code lies within here
│   ├── utils # utilities for your graph
│   │   ├── __init__.py
│   │   ├── tools.py # tools for your graph
│   │   ├── nodes.py # node functions for you graph
│   │   └── state.py # state definition of your graph
│   ├── __init__.py
│   └── agent.py # code for constructing your graph
├── .env
└── pyproject.toml
```

## Create LangGraph Configuration File

Create a [LangGraph configuration file](../reference/cli.md#configuration-file) called `langgraph.json`. See the [LangGraph configuration file reference](../reference/cli.md#configuration-file) for detailed explanations of each key in the JSON object of the configuration file.

Example `langgraph.json` file:

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./my_agent/agent.py:graph"
  },
  "env": ".env"
}
```

Note that the variable name of the `CompiledGraph` appears at the end of the value of each subkey in the top-level `graphs` key (i.e. `:<variable_name>`).

!!! warning "Configuration File Location"
    The LangGraph configuration file must be placed in a directory that is at the same level or higher than the Python files that contain compiled graphs and associated dependencies.

Example file directory:

```bash
my-app/
├── my_agent # all project code lies within here
│   ├── utils # utilities for your graph
│   │   ├── __init__.py
│   │   ├── tools.py # tools for your graph
│   │   ├── nodes.py # node functions for you graph
│   │   └── state.py # state definition of your graph
│   ├── __init__.py
│   └── agent.py # code for constructing your graph
├── .env # environment variables
├── langgraph.json  # configuration file for LangGraph
└── pyproject.toml # dependencies for your project
```

## Next

After you setup your project and place it in a GitHub repository, it's time to [deploy your app](./cloud.md).

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/deployment/standalone_container.md
```md
# How to Deploy a Standalone Container

Before deploying, review the [conceptual guide for the Standalone Container](../../concepts/langgraph_standalone_container.md) deployment option.

## Prerequisites

1. Use the [LangGraph CLI](../../concepts/langgraph_cli.md) to [test your application locally](../../tutorials/langgraph-platform/local-server.md).
1. Use the [LangGraph CLI](../../concepts/langgraph_cli.md) to build a Docker image (i.e. `langgraph build`).
1. The following environment variables are needed for a standalone container deployment.
    1. `REDIS_URI`: Connection details to a Redis instance. Redis will be used as a pub-sub broker to enable streaming real time output from background runs. The value of `REDIS_URI` must be a valid [Redis connection URI](https://redis-py.readthedocs.io/en/stable/connections.html#redis.Redis.from_url).

        !!! Note "Shared Redis Instance"
            Multiple self-hosted deployments can share the same Redis instance. For example, for `Deployment A`, `REDIS_URI` can be set to `redis://<hostname_1>:<port>/1` and for `Deployment B`, `REDIS_URI` can be set to `redis://<hostname_1>:<port>/2`.

            `1` and `2` are different database numbers within the same instance, but `<hostname_1>` is shared. **The same database number cannot be used for separate deployments**.

    1. `DATABASE_URI`: Postgres connection details. Postgres will be used to store assistants, threads, runs, persist thread state and long term memory, and to manage the state of the background task queue with 'exactly once' semantics. The value of `DATABASE_URI` must be a valid [Postgres connection URI](https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNSTRING-URIS).

        !!! Note "Shared Postgres Instance"
            Multiple self-hosted deployments can share the same Postgres instance. For example, for `Deployment A`, `DATABASE_URI` can be set to `postgres://<user>:<password>@/<database_name_1>?host=<hostname_1>` and for `Deployment B`, `DATABASE_URI` can be set to `postgres://<user>:<password>@/<database_name_2>?host=<hostname_1>`.

            `<database_name_1>` and `database_name_2` are different databases within the same instance, but `<hostname_1>` is shared. **The same database cannot be used for separate deployments**.

    1. `LANGGRAPH_CLOUD_LICENSE_KEY`: (if using [Enterprise](../../concepts/langgraph_data_plane.md#licensing)) LangGraph Platform license key. This will be used to authenticate ONCE at server start up.
    1. `LANGSMITH_ENDPOINT`: To send traces to a [self-hosted LangSmith](https://docs.smith.langchain.com/self_hosting) instance, set `LANGSMITH_ENDPOINT` to the hostname of the self-hosted LangSmith instance.
1. Egress to `https://beacon.langchain.com` from your network. This is required for license verification and usage reporting if not running in air-gapped mode. See the [Egress documentation](../../cloud/deployment/egress.md) for more details.

## Kubernetes (Helm)

Use this [Helm chart](https://github.com/langchain-ai/helm/blob/main/charts/langgraph-cloud/README.md) to deploy a LangGraph Server to a Kubernetes cluster.

## Docker

Run the following `docker` command:
```shell
docker run \
    --env-file .env \
    -p 8123:8000 \
    -e REDIS_URI="foo" \
    -e DATABASE_URI="bar" \
    -e LANGSMITH_API_KEY="baz" \
    my-image
```

!!! note

    * You need to replace `my-image` with the name of the image you built in the prerequisite steps (from `langgraph build`)
    and you should provide appropriate values for `REDIS_URI`, `DATABASE_URI`, and `LANGSMITH_API_KEY`.
    * If your application requires additional environment variables, you can pass them in a similar way.

## Docker Compose

Docker Compose YAML file:
```yml
volumes:
    langgraph-data:
        driver: local
services:
    langgraph-redis:
        image: redis:6
        healthcheck:
            test: redis-cli ping
            interval: 5s
            timeout: 1s
            retries: 5
    langgraph-postgres:
        image: postgres:16
        ports:
            - "5433:5432"
        environment:
            POSTGRES_DB: postgres
            POSTGRES_USER: postgres
            POSTGRES_PASSWORD: postgres
        volumes:
            - langgraph-data:/var/lib/postgresql/data
        healthcheck:
            test: pg_isready -U postgres
            start_period: 10s
            timeout: 1s
            retries: 5
            interval: 5s
    langgraph-api:
        image: ${IMAGE_NAME}
        ports:
            - "8123:8000"
        depends_on:
            langgraph-redis:
                condition: service_healthy
            langgraph-postgres:
                condition: service_healthy
        env_file:
            - .env
        environment:
            REDIS_URI: redis://langgraph-redis:6379
            LANGSMITH_API_KEY: ${LANGSMITH_API_KEY}
            POSTGRES_URI: postgres://postgres:postgres@langgraph-postgres:5432/postgres?sslmode=disable
```

You can run the command `docker compose up` with this Docker Compose file in the same folder.

This will launch a LangGraph Server on port `8123` (if you want to change this, you can change this by changing the ports in the `langgraph-api` volume). You can test if the application is healthy by running:

```shell
curl --request GET --url 0.0.0.0:8123/ok
```
Assuming everything is running correctly, you should see a response like:

```shell
{"ok":true}
```

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/deployment/img/langgraph_studio.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/deployment/img/quick_start_studio.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/sdk/img/graph_diagram.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/sdk/img/thread_diagram.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/add-human-in-the-loop.md
```md
# Human-in-the-loop using Server API

To review, edit, and approve tool calls in an agent or workflow, use LangGraph's [human-in-the-loop](../../concepts/human_in_the_loop.md) features.

## Dynamic interrupts

=== "Python"

    ```python
    from langgraph_sdk import get_client
    # highlight-next-line
    from langgraph_sdk.schema import Command
    client = get_client(url=<DEPLOYMENT_URL>)

    # Using the graph deployed with the name "agent"
    assistant_id = "agent"

    # create a thread
    thread = await client.threads.create()
    thread_id = thread["thread_id"]

    # Run the graph until the interrupt is hit.
    result = await client.runs.wait(
        thread_id,
        assistant_id,
        input={"some_text": "original text"}   # (1)!
    )

    print(result['__interrupt__']) # (2)!
    # > [
    # >     {
    # >         'value': {'text_to_revise': 'original text'},
    # >         'id': '...',
    # >     }
    # > ]


    # Resume the graph
    print(await client.runs.wait(
        thread_id,
        assistant_id,
        # highlight-next-line
        command=Command(resume="Edited text")   # (3)!
    ))
    # > {'some_text': 'Edited text'}
    ```

    1. The graph is invoked with some initial state.
    2. When the graph hits the interrupt, it returns an interrupt object with the payload and metadata.
    3. The graph is resumed with a `Command(resume=...)`, injecting the human's input and continuing execution.

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

    // Using the graph deployed with the name "agent"
    const assistantID = "agent";

    // create a thread
    const thread = await client.threads.create();
    const threadID = thread["thread_id"];

    // Run the graph until the interrupt is hit.
    const result = await client.runs.wait(
      threadID,
      assistantID,
      { input: { "some_text": "original text" } }   // (1)!
    );

    console.log(result['__interrupt__']); // (2)!
    // > [
    // >     {
    // >         'value': {'text_to_revise': 'original text'},
    // >         'resumable': True,
    // >         'ns': ['human_node:fc722478-2f21-0578-c572-d9fc4dd07c3b'],
    // >         'when': 'during'
    // >     }
    // > ]

    // Resume the graph
    console.log(await client.runs.wait(
        threadID,
        assistantID,
        // highlight-next-line
        { command: { resume: "Edited text" }}   // (3)!
    ));
    // > {'some_text': 'Edited text'}
    ```

    1. The graph is invoked with some initial state.
    2. When the graph hits the interrupt, it returns an interrupt object with the payload and metadata.
    3. The graph is resumed with a `{ resume: ... }` command object, injecting the human's input and continuing execution.

=== "cURL"

    Create a thread:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads \
    --header 'Content-Type: application/json' \
    --data '{}'
    ```

    Run the graph until the interrupt is hit.:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"some_text\": \"original text\"}
    }"
    ```

    Resume the graph:

    ```bash
    curl --request POST \
     --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
     --header 'Content-Type: application/json' \
     --data "{
       \"assistant_id\": \"agent\",
       \"command\": {
         \"resume\": \"Edited text\"
       }
     }"
    ```

??? example "Extended example: using `interrupt`"

    This is an example graph you can run in the LangGraph API server.
    See [LangGraph Platform quickstart](../quick_start.md) for more details.

    ```python
    from typing import TypedDict
    import uuid

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.constants import START
    from langgraph.graph import StateGraph
    # highlight-next-line
    from langgraph.types import interrupt, Command

    class State(TypedDict):
        some_text: str

    def human_node(state: State):
        # highlight-next-line
        value = interrupt( # (1)!
            {
                "text_to_revise": state["some_text"] # (2)!
            }
        )
        return {
            "some_text": value # (3)!
        }


    # Build the graph
    graph_builder = StateGraph(State)
    graph_builder.add_node("human_node", human_node)
    graph_builder.add_edge(START, "human_node")

    graph = graph_builder.compile()
    ```

    1. `interrupt(...)` pauses execution at `human_node`, surfacing the given payload to a human.
    2. Any JSON serializable value can be passed to the `interrupt` function. Here, a dict containing the text to revise.
    3. Once resumed, the return value of `interrupt(...)` is the human-provided input, which is used to update the state.

    Once you have a running LangGraph API server, you can interact with it using
    [LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/)

    === "Python"

        ```python
        from langgraph_sdk import get_client
        # highlight-next-line
        from langgraph_sdk.schema import Command
        client = get_client(url=<DEPLOYMENT_URL>)

        # Using the graph deployed with the name "agent"
        assistant_id = "agent"

        # create a thread
        thread = await client.threads.create()
        thread_id = thread["thread_id"]

        # Run the graph until the interrupt is hit.
        result = await client.runs.wait(
            thread_id,
            assistant_id,
            input={"some_text": "original text"}   # (1)!
        )

        print(result['__interrupt__']) # (2)!
        # > [
        # >     {
        # >         'value': {'text_to_revise': 'original text'},
        # >         'id': '...',
        # >     }
        # > ]


        # Resume the graph
        print(await client.runs.wait(
            thread_id,
            assistant_id,
            # highlight-next-line
            command=Command(resume="Edited text")   # (3)!
        ))
        # > {'some_text': 'Edited text'}
        ```

        1. The graph is invoked with some initial state.
        2. When the graph hits the interrupt, it returns an interrupt object with the payload and metadata.
        3. The graph is resumed with a `Command(resume=...)`, injecting the human's input and continuing execution.

    === "JavaScript"

        ```js
        import { Client } from "@langchain/langgraph-sdk";
        const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

        // Using the graph deployed with the name "agent"
        const assistantID = "agent";

        // create a thread
        const thread = await client.threads.create();
        const threadID = thread["thread_id"];

        // Run the graph until the interrupt is hit.
        const result = await client.runs.wait(
          threadID,
          assistantID,
          { input: { "some_text": "original text" } }   // (1)!
        );

        console.log(result['__interrupt__']); // (2)!
        // > [
        // >     {
        // >         'value': {'text_to_revise': 'original text'},
        // >         'resumable': True,
        // >         'ns': ['human_node:fc722478-2f21-0578-c572-d9fc4dd07c3b'],
        // >         'when': 'during'
        // >     }
        // > ]

        // Resume the graph
        console.log(await client.runs.wait(
            threadID,
            assistantID,
            // highlight-next-line
            { command: { resume: "Edited text" }}   // (3)!
        ));
        // > {'some_text': 'Edited text'}
        ```

        1. The graph is invoked with some initial state.
        2. When the graph hits the interrupt, it returns an interrupt object with the payload and metadata.
        3. The graph is resumed with a `{ resume: ... }` command object, injecting the human's input and continuing execution.

    === "cURL"

        Create a thread:

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
        ```

        Run the graph until the interrupt is hit:

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"input\": {\"some_text\": \"original text\"}
        }"
        ```

        Resume the graph:

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"command\": {
            \"resume\": \"Edited text\"
          }
        }"
        ```

## Static interrupts

Static interrupts (also known as static breakpoints) are triggered either before or after a node executes. 

!!! warning

    Static interrupts are **not** recommended for human-in-the-loop workflows. They are best used for debugging and testing.

You can set static interrupts by specifying `interrupt_before` and `interrupt_after` at compile time:

```python
# highlight-next-line
graph = graph_builder.compile( # (1)!
    # highlight-next-line
    interrupt_before=["node_a"], # (2)!
    # highlight-next-line
    interrupt_after=["node_b", "node_c"], # (3)!
)
```

1. The breakpoints are set during `compile` time.
2. `interrupt_before` specifies the nodes where execution should pause before the node is executed.
3. `interrupt_after` specifies the nodes where execution should pause after the node is executed.

Alternatively, you can set static interrupts at run time:

=== "Python"

    ```python
    # highlight-next-line
    await client.runs.wait( # (1)!
        thread_id,
        assistant_id,
        inputs=inputs,
        # highlight-next-line
        interrupt_before=["node_a"], # (2)!
        # highlight-next-line
        interrupt_after=["node_b", "node_c"] # (3)!
    )
    ```

    1. `client.runs.wait` is called with the `interrupt_before` and `interrupt_after` parameters. This is a run-time configuration and can be changed for every invocation.
    2. `interrupt_before` specifies the nodes where execution should pause before the node is executed.
    3. `interrupt_after` specifies the nodes where execution should pause after the node is executed.

=== "JavaScript"

    ```js
    // highlight-next-line
    await client.runs.wait( // (1)!
        threadID,
        assistantID,
        {
        input: input,
        // highlight-next-line
        interruptBefore: ["node_a"], // (2)!
        // highlight-next-line
        interruptAfter: ["node_b", "node_c"] // (3)!
        }
    )
    ```

    1. `client.runs.wait` is called with the `interruptBefore` and `interruptAfter` parameters. This is a run-time configuration and can be changed for every invocation.
    2. `interruptBefore` specifies the nodes where execution should pause before the node is executed.
    3. `interruptAfter` specifies the nodes where execution should pause after the node is executed.

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
        \"assistant_id\": \"agent\",
        \"interrupt_before\": [\"node_a\"],
        \"interrupt_after\": [\"node_b\", \"node_c\"],
        \"input\": <INPUT>
    }"
    ```

The following example shows how to add static interrupts:

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>)

    # Using the graph deployed with the name "agent"
    assistant_id = "agent"

    # create a thread
    thread = await client.threads.create()
    thread_id = thread["thread_id"]

    # Run the graph until the breakpoint
    result = await client.runs.wait(
        thread_id,
        assistant_id,
        input=inputs   # (1)!
    )

    # Resume the graph
    await client.runs.wait(
        thread_id,
        assistant_id,
        input=None   # (2)!
    )
    ```

    1. The graph is run until the first breakpoint is hit.
    2. The graph is resumed by passing in `None` for the input. This will run the graph until the next breakpoint is hit.

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

    // Using the graph deployed with the name "agent"
    const assistantID = "agent";

    // create a thread
    const thread = await client.threads.create();
    const threadID = thread["thread_id"];

    // Run the graph until the breakpoint
    const result = await client.runs.wait(
      threadID,
      assistantID,
      { input: input }   // (1)!
    );

    // Resume the graph
    await client.runs.wait(
      threadID,
      assistantID,
      { input: null }   // (2)!
    );
    ```

    1. The graph is run until the first breakpoint is hit.
    2. The graph is resumed by passing in `null` for the input. This will run the graph until the next breakpoint is hit.

=== "cURL"

    Create a thread:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads \
    --header 'Content-Type: application/json' \
    --data '{}'
    ```

    Run the graph until the breakpoint:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": <INPUT>
    }"
    ```

    Resume the graph:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\"
    }"
    ```


## Learn more

- [Human-in-the-loop conceptual guide](../../concepts/human_in_the_loop.md): learn more about LangGraph human-in-the-loop features. 
- [Common patterns](../../how-tos/human_in_the_loop/add-human-in-the-loop.md#common-patterns): learn how to implement patterns like approving/rejecting actions, requesting user input, tool call review, and validating human input.
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/background_run.md
```md
# How to kick off background runs
 

This guide covers how to kick off background runs for your agent.
This can be useful for long running jobs.

## Setup

First let's set up our client and thread:

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # Using the graph deployed with the name "agent"
    assistant_id = "agent"
    # create thread
    thread = await client.threads.create()
    print(thread)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // Using the graph deployed with the name "agent"
    const assistantID = "agent";
    // create thread
    const thread = await client.threads.create();
    console.log(thread);
    ```

=== "CURL"

    ```bash
    curl --request POST \
      --url <DEPLOYMENT_URL>/threads \
      --header 'Content-Type: application/json' \
      --data '{}'
    ```

Output:

    {
        'thread_id': '5cb1e8a1-34b3-4a61-a34e-71a9799bd00d',
        'created_at': '2024-08-30T20:35:52.062934+00:00',
        'updated_at': '2024-08-30T20:35:52.062934+00:00',
        'metadata': {},
        'status': 'idle',
        'config': {},
        'values': None
    }

## Check runs on thread

If we list the current runs on this thread, we will see that it's empty:

=== "Python"

    ```python
    runs = await client.runs.list(thread["thread_id"])
    print(runs)
    ```

=== "Javascript"

    ```js
    let runs = await client.runs.list(thread['thread_id']);
    console.log(runs);
    ```

=== "CURL"

    ```bash
    curl --request GET \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs
    ```

Output:

    []

## Start runs on thread

Now let's kick off a run:

=== "Python"

    ```python
    input = {"messages": [{"role": "user", "content": "what's the weather in sf"}]}
    run = await client.runs.create(thread["thread_id"], assistant_id, input=input)
    ```

=== "Javascript"

    ```js
    let input = {"messages": [{"role": "user", "content": "what's the weather in sf"}]};
    let run = await client.runs.create(thread["thread_id"], assistantID, { input });
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <ASSISTANT_ID>
        }'
    ```

The first time we poll it, we can see `status=pending`:

=== "Python"

    ```python
    print(await client.runs.get(thread["thread_id"], run["run_id"]))
    ```

=== "Javascript"

    ```js
    console.log(await client.runs.get(thread["thread_id"], run["run_id"]));
    ```

=== "CURL"

    ```bash
    curl --request GET \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/<RUN_ID>
    ```

Output:

        {
            "run_id": "1ef6a5f8-bd86-6763-bbd6-bff042db7b1b",
            "thread_id": "7885f0cf-94ad-4040-91d7-73f7ba007c8a",
            "assistant_id": "fe096781-5601-53d2-b2f6-0d3403f7e9ca",
            "created_at": "2024-09-04T01:46:47.244887+00:00",
            "updated_at": "2024-09-04T01:46:47.244887+00:00",
            "metadata": {},
            "status": "pending",
            "kwargs": {
                "input": {
                    "messages": [
                        {
                            "role": "user",
                            "content": "what's the weather in sf"
                        }
                    ]
                },
                "config": {
                    "metadata": {
                        "created_by": "system"
                    },
                    "configurable": {
                        "run_id": "1ef6a5f8-bd86-6763-bbd6-bff042db7b1b",
                        "user_id": "",
                        "graph_id": "agent",
                        "thread_id": "7885f0cf-94ad-4040-91d7-73f7ba007c8a",
                        "assistant_id": "fe096781-5601-53d2-b2f6-0d3403f7e9ca",
                        "checkpoint_id": null
                    }
                },
                "webhook": null,
                "temporary": false,
                "stream_mode": [
                    "values"
                ],
                "feedback_keys": null,
                "interrupt_after": null,
                "interrupt_before": null
            },
            "multitask_strategy": "reject"
        }



Now we can join the run, wait for it to finish and check that status again:

=== "Python"

    ```python
    await client.runs.join(thread["thread_id"], run["run_id"])
    print(await client.runs.get(thread["thread_id"], run["run_id"]))
    ```

=== "Javascript"

    ```js
    await client.runs.join(thread["thread_id"], run["run_id"]);
    console.log(await client.runs.get(thread["thread_id"], run["run_id"]));
    ```

=== "CURL"

    ```bash
    curl --request GET \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/<RUN_ID>/join &&
    curl --request GET \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/<RUN_ID>
    ```

Output:

    {
        "run_id": "1ef6a5f8-bd86-6763-bbd6-bff042db7b1b",
        "thread_id": "7885f0cf-94ad-4040-91d7-73f7ba007c8a",
        "assistant_id": "fe096781-5601-53d2-b2f6-0d3403f7e9ca",
        "created_at": "2024-09-04T01:46:47.244887+00:00",
        "updated_at": "2024-09-04T01:46:47.244887+00:00",
        "metadata": {},
        "status": "success",
        "kwargs": {
            "input": {
                "messages": [
                    {
                        "role": "user",
                        "content": "what's the weather in sf"
                    }
                ]
            },
            "config": {
                "metadata": {
                    "created_by": "system"
                },
                "configurable": {
                    "run_id": "1ef6a5f8-bd86-6763-bbd6-bff042db7b1b",
                    "user_id": "",
                    "graph_id": "agent",
                    "thread_id": "7885f0cf-94ad-4040-91d7-73f7ba007c8a",
                    "assistant_id": "fe096781-5601-53d2-b2f6-0d3403f7e9ca",
                    "checkpoint_id": null
                }
            },
            "webhook": null,
            "temporary": false,
            "stream_mode": [
                "values"
            ],
            "feedback_keys": null,
            "interrupt_after": null,
            "interrupt_before": null
        },
        "multitask_strategy": "reject"
    }


Perfect! The run succeeded as we would expect. We can double check that the run worked as expected by printing out the final state:

=== "Python"

    ```python
    final_result = await client.threads.get_state(thread["thread_id"])
    print(final_result)
    ```

=== "Javascript"

    ```js
    let finalResult = await client.threads.getState(thread["thread_id"]);
    console.log(finalResult);
    ```

=== "CURL"

    ```bash
    curl --request GET \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state
    ```

Output:

    {
        "values": {
            "messages": [
                {
                    "content": "what's the weather in sf",
                    "additional_kwargs": {},
                    "response_metadata": {},
                    "type": "human",
                    "name": null,
                    "id": "beba31bf-320d-4125-9c37-cadf526ac47a",
                    "example": false
                },
                {
                    "content": [
                        {
                            "id": "toolu_01AaNPSPzqia21v7aAKwbKYm",
                            "input": {},
                            "name": "tavily_search_results_json",
                            "type": "tool_use",
                            "index": 0,
                            "partial_json": "{\"query\": \"weather in san francisco\"}"
                        }
                    ],
                    "additional_kwargs": {},
                    "response_metadata": {
                        "stop_reason": "tool_use",
                        "stop_sequence": null
                    },
                    "type": "ai",
                    "name": null,
                    "id": "run-f220faf8-1d27-4f73-ad91-6bb3f47e8639",
                    "example": false,
                    "tool_calls": [
                        {
                            "name": "tavily_search_results_json",
                            "args": {
                                "query": "weather in san francisco"
                            },
                            "id": "toolu_01AaNPSPzqia21v7aAKwbKYm",
                            "type": "tool_call"
                        }
                    ],
                    "invalid_tool_calls": [],
                    "usage_metadata": {
                        "input_tokens": 273,
                        "output_tokens": 61,
                        "total_tokens": 334
                    }
                },
                {
                    "content": "[{\"url\": \"https://www.weatherapi.com/\", \"content\": \"{'location': {'name': 'San Francisco', 'region': 'California', 'country': 'United States of America', 'lat': 37.78, 'lon': -122.42, 'tz_id': 'America/Los_Angeles', 'localtime_epoch': 1725052131, 'localtime': '2024-08-30 14:08'}, 'current': {'last_updated_epoch': 1725051600, 'last_updated': '2024-08-30 14:00', 'temp_c': 21.1, 'temp_f': 70.0, 'is_day': 1, 'condition': {'text': 'Partly cloudy', 'icon': '//cdn.weatherapi.com/weather/64x64/day/116.png', 'code': 1003}, 'wind_mph': 11.9, 'wind_kph': 19.1, 'wind_degree': 290, 'wind_dir': 'WNW', 'pressure_mb': 1018.0, 'pressure_in': 30.07, 'precip_mm': 0.0, 'precip_in': 0.0, 'humidity': 59, 'cloud': 25, 'feelslike_c': 21.1, 'feelslike_f': 70.0, 'windchill_c': 18.6, 'windchill_f': 65.5, 'heatindex_c': 18.6, 'heatindex_f': 65.5, 'dewpoint_c': 12.2, 'dewpoint_f': 54.0, 'vis_km': 16.0, 'vis_miles': 9.0, 'uv': 5.0, 'gust_mph': 15.0, 'gust_kph': 24.2}}\"}]",
                    "additional_kwargs": {},
                    "response_metadata": {},
                    "type": "tool",
                    "name": "tavily_search_results_json",
                    "id": "686b2487-f332-4e58-9508-89b3a814cd81",
                    "tool_call_id": "toolu_01AaNPSPzqia21v7aAKwbKYm",
                    "artifact": {
                        "query": "weather in san francisco",
                        "follow_up_questions": null,
                        "answer": null,
                        "images": [],
                        "results": [
                            {
                                "title": "Weather in San Francisco",
                                "url": "https://www.weatherapi.com/",
                                "content": "{'location': {'name': 'San Francisco', 'region': 'California', 'country': 'United States of America', 'lat': 37.78, 'lon': -122.42, 'tz_id': 'America/Los_Angeles', 'localtime_epoch': 1725052131, 'localtime': '2024-08-30 14:08'}, 'current': {'last_updated_epoch': 1725051600, 'last_updated': '2024-08-30 14:00', 'temp_c': 21.1, 'temp_f': 70.0, 'is_day': 1, 'condition': {'text': 'Partly cloudy', 'icon': '//cdn.weatherapi.com/weather/64x64/day/116.png', 'code': 1003}, 'wind_mph': 11.9, 'wind_kph': 19.1, 'wind_degree': 290, 'wind_dir': 'WNW', 'pressure_mb': 1018.0, 'pressure_in': 30.07, 'precip_mm': 0.0, 'precip_in': 0.0, 'humidity': 59, 'cloud': 25, 'feelslike_c': 21.1, 'feelslike_f': 70.0, 'windchill_c': 18.6, 'windchill_f': 65.5, 'heatindex_c': 18.6, 'heatindex_f': 65.5, 'dewpoint_c': 12.2, 'dewpoint_f': 54.0, 'vis_km': 16.0, 'vis_miles': 9.0, 'uv': 5.0, 'gust_mph': 15.0, 'gust_kph': 24.2}}",
                                "score": 0.976148,
                                "raw_content": null
                            }
                        ],
                        "response_time": 3.07
                    },
                    "status": "success"
                },
                {
                    "content": [
                        {
                            "text": "\n\nThe search results provide the current weather conditions in San Francisco. According to the data, as of 2:00 PM on August 30, 2024, the temperature in San Francisco is 70\u00b0F (21.1\u00b0C) with partly cloudy skies. The wind is blowing from the west-northwest at around 12 mph (19 km/h). The humidity is 59% and visibility is 9 miles (16 km). Overall, it looks like a nice late summer day in San Francisco with comfortable temperatures and partly sunny conditions.",
                            "type": "text",
                            "index": 0
                        }
                    ],
                    "additional_kwargs": {},
                    "response_metadata": {
                        "stop_reason": "end_turn",
                        "stop_sequence": null
                    },
                    "type": "ai",
                    "name": null,
                    "id": "run-8fecc61d-3d9f-4e16-8e8a-92f702be498a",
                    "example": false,
                    "tool_calls": [],
                    "invalid_tool_calls": [],
                    "usage_metadata": {
                        "input_tokens": 837,
                        "output_tokens": 124,
                        "total_tokens": 961
                    }
                }
            ]
        },
        "next": [],
        "tasks": [],
        "metadata": {
            "step": 3,
            "run_id": "1ef67140-eb23-684b-8253-91d4c90bb05e",
            "source": "loop",
            "writes": {
                "agent": {
                    "messages": [
                        {
                            "id": "run-8fecc61d-3d9f-4e16-8e8a-92f702be498a",
                            "name": null,
                            "type": "ai",
                            "content": [
                                {
                                    "text": "\n\nThe search results provide the current weather conditions in San Francisco. According to the data, as of 2:00 PM on August 30, 2024, the temperature in San Francisco is 70\u00b0F (21.1\u00b0C) with partly cloudy skies. The wind is blowing from the west-northwest at around 12 mph (19 km/h). The humidity is 59% and visibility is 9 miles (16 km). Overall, it looks like a nice late summer day in San Francisco with comfortable temperatures and partly sunny conditions.",
                                    "type": "text",
                                    "index": 0
                                }
                            ],
                            "example": false,
                            "tool_calls": [],
                            "usage_metadata": {
                                "input_tokens": 837,
                                "total_tokens": 961,
                                "output_tokens": 124
                            },
                            "additional_kwargs": {},
                            "response_metadata": {
                                "stop_reason": "end_turn",
                                "stop_sequence": null
                            },
                            "invalid_tool_calls": []
                        }
                    ]
                }
            },
            "user_id": "",
            "graph_id": "agent",
            "thread_id": "5cb1e8a1-34b3-4a61-a34e-71a9799bd00d",
            "created_by": "system",
            "assistant_id": "fe096781-5601-53d2-b2f6-0d3403f7e9ca"
        },
        "created_at": "2024-08-30T21:09:00.079909+00:00",
        "checkpoint_id": "1ef67141-3ca2-6fae-8003-fe96832e57d6",
        "parent_checkpoint_id": "1ef67141-2129-6b37-8002-61fc3bf69cb5"
    }

We can also just print the content of the last AIMessage:

=== "Python"

    ```python
    print(final_result['values']['messages'][-1]['content'][0]['text'])
    ```

=== "Javascript"

    ```js
    console.log(finalResult['values']['messages'][finalResult['values']['messages'].length-1]['content'][0]['text']);
    ```

=== "CURL"

    ```bash
    curl --request GET \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state | jq -r '.values.messages[-1].content.[0].text'
    ```

Output:

    The search results provide the current weather conditions in San Francisco. According to the data, as of 2:00 PM on August 30, 2024, the temperature in San Francisco is 70°F (21.1°C) with partly cloudy skies. The wind is blowing from the west-northwest at around 12 mph (19 km/h). The humidity is 59% and visibility is 9 miles (16 km). Overall, it looks like a nice late summer day in San Francisco with comfortable temperatures and partly sunny conditions.
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/clone_traces_studio.md
```md
# Debug LangSmith traces

This guide explains how to open LangSmith traces in LangGraph Studio for interactive investigation and debugging.

## Open deployed threads

1. Open the LangSmith trace, selecting the root run.
2. Click "Run in Studio".

This will open LangGraph Studio connected to the associated LangGraph Platform deployment with the trace's parent thread selected.

## Testing local agents with remote traces

This section explains how to test a local agent against remote traces from LangSmith. This enables you to use production traces as input for local testing, allowing you to debug and verify agent modifications in your development environment.

### Requirements

- A LangSmith traced thread
- A locally running agent. See [here](../how-tos/studio/quick_start.md#local-development-server) for setup
  instructions.

!!! info "Local agent requirements"

    - langgraph>=0.3.18
    - langgraph-api>=0.0.32
    - Contains the same set of nodes present in the remote trace

### Cloning Thread

1. Open the LangSmith trace, selecting the root run.
2. Click the dropdown next to "Run in Studio".
3. Enter your local agent's URL.
4. Select "Clone thread locally".
5. If multiple graphs exist, select the target graph.

A new thread will be created in your local agent with the thread history inferred and copied from the remote thread, and you will be navigated to LangGraph Studio for your locally running application.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/configurable_headers.md
```md
# Configurable Headers

LangGraph allows runtime configuration to modify agent behavior and permissions dynamically. When using the [LangGraph Platform](../quick_start.md), you can pass this configuration in the request body (`config`) or specific request headers. This enables adjustments based on user identity or other request data.

For privacy, control which headers are passed to the runtime configuration via the `http.configurable_headers` section in your `langgraph.json` file.

Here's how to customize the included and excluded headers:

```json
{
  "http": {
    "configurable_headers": {
      "include": ["x-user-id", "x-organization-id", "my-prefix-*"],
      "exclude": ["authorization", "x-api-key"]
    }
  }
}
```


The `include` and `exclude` lists accept exact header names or patterns using `*` to match any number of characters. For your security, no other regex patterns are supported.

## Using within your graph

You can access the included headers in your graph using the `config` argument of any node.

```python
def my_node(state, config):
  organization_id = config["configurable"].get("x-organization-id")
  ...
```

Or by fetching from context (useful in tools and or within other nested functions).

```python
from langgraph.config import get_config

def search_everything(query: str):
  organization_id = get_config()["configurable"].get("x-organization-id")
  ...
```


You can even use this to dynamically compile the graph.

```python
# my_graph.py.
import contextlib

@contextlib.asynccontextmanager
async def generate_agent(config):
  organization_id = config["configurable"].get("x-organization-id")
  if organization_id == "org1":
    graph = ...
    yield graph
  else:
    graph = ...
    yield graph

```

```json
{
  "graphs": {"agent": "my_grph.py:generate_agent"}
}
```

### Opt-out of configurable headers

If you'd like to opt-out of configurable headers, you can simply set a wildcard pattern in the `exclude` list:

```json
{
  "http": {
    "configurable_headers": {
      "exclude": ["*"]
    }
  }
}
```

This will exclude all headers from being added to your run's configuration.

Note that exclusions take precedence over inclusions.
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/configuration_cloud.md
```md
# Manage assistants

In this guide we will show how to create, configure, and manage an [assistant](../../concepts/assistants.md).

First, as a brief refresher on the concept of runtime context, consider the following simple `call_model` node and context schema. Observe that this node tries to read and use the `model_provider` as defined by the `Runtime` object's `context` property.

=== "Python"

    ```python
    @dataclass
    class ContextSchema:
        llm_provider: str = "anthropic"

    builder = StateGraph(AgentState, context_schema=ContextSchema)

    def call_model(state, runtime: Runtime[ContextSchema]):
        messages = state["messages"]
        model = _get_model(runtime.context.llm_provider)
        response = model.invoke(messages)
        # We return a list, because this will get added to the existing list
        return {"messages": [response]}
    ```

=== "Javascript"

    ```js
    import { Annotation } from "@langchain/langgraph";

    const ConfigSchema = Annotation.Root({
        model_name: Annotation<string>,
        system_prompt:
    });

    const builder = new StateGraph(AgentState, ConfigSchema)

    function callModel(state: State, config: RunnableConfig) {
      const messages = state.messages;
      const modelName = config.configurable?.model_name ?? "anthropic";
      const model = _getModel(modelName);
      const response = model.invoke(messages);
      // We return a list, because this will get added to the existing list
      return { messages: [response] };
    }
    ```

:::python
For more information on runtime context, [see here](../../concepts/low_level.md#runtime-context).
:::

## Create an assistant

### LangGraph SDK

To create an assistant, use the [LangGraph SDK](../../concepts/sdk.md) `create` method. See the [Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.AssistantsClient.create) and [JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#create) SDK reference docs for more information.

This example uses the same configuration schema as above, and creates an assistant with `model_name` set to `openai`.

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    openai_assistant = await client.assistants.create(
        # "agent" is the name of a graph we deployed
        "agent", config={"configurable": {"model_name": "openai"}}, name="Open AI Assistant"
    )

    print(openai_assistant)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    const openAIAssistant = await client.assistants.create({
        graphId: 'agent',
        name: "Open AI Assistant",
        config: { "configurable": { "model_name": "openai" } },
    });

    console.log(openAIAssistant);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/assistants \
        --header 'Content-Type: application/json' \
        --data '{"graph_id":"agent", "config":{"configurable":{"model_name":"openai"}}, "name": "Open AI Assistant"}'
    ```

Output:

    {
        "assistant_id": "62e209ca-9154-432a-b9e9-2d75c7a9219b",
        "graph_id": "agent",
        "name": "Open AI Assistant"
        "config": {
            "configurable": {
                "model_name": "openai"
            }
        },
        "metadata": {}
        "created_at": "2024-08-31T03:09:10.230718+00:00",
        "updated_at": "2024-08-31T03:09:10.230718+00:00",
    }

### LangGraph Platform UI

You can also create assistants from the LangGraph Platform UI.

Inside your deployment, select the "Assistants" tab. This will load a table of all of the assistants in your deployment, across all graphs.

To create a new assistant, select the "+ New assistant" button. This will open a form where you can specify the graph this assistant is for, as well as provide a name, description, and the desired configuration for the assistant based on the configuration schema for that graph.

To confirm, click "Create assistant". This will take you to [LangGraph Studio](../../concepts/langgraph_studio.md) where you can test the assistant. If you go back to the "Assistants" tab in the deployment, you will see the newly created assistant in the table.

## Use an assistant

### LangGraph SDK

We have now created an assistant called "Open AI Assistant" that has `model_name` defined as `openai`. We can now use this assistant with this configuration:

=== "Python"

    ```python
    thread = await client.threads.create()
    input = {"messages": [{"role": "user", "content": "who made you?"}]}
    async for event in client.runs.stream(
        thread["thread_id"],
        # this is where we specify the assistant id to use
        openai_assistant["assistant_id"],
        input=input,
        stream_mode="updates",
    ):
        print(f"Receiving event of type: {event.event}")
        print(event.data)
        print("\n\n")
    ```

=== "Javascript"

    ```js
    const thread = await client.threads.create();
    const input = { "messages": [{ "role": "user", "content": "who made you?" }] };

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      // this is where we specify the assistant id to use
      openAIAssistant["assistant_id"],
      {
        input,
        streamMode: "updates"
      }
    );

    for await (const event of streamResponse) {
      console.log(`Receiving event of type: ${event.event}`);
      console.log(event.data);
      console.log("\n\n");
    }
    ```

=== "CURL"

    ```bash
    thread_id=$(curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}' | jq -r '.thread_id') && \
    curl --request POST \
        --url "<DEPLOYMENT_URL>/threads/${thread_id}/runs/stream" \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <OPENAI_ASSISTANT_ID>,
            "input": {
                "messages": [
                    {
                        "role": "user",
                        "content": "who made you?"
                    }
                ]
            },
            "stream_mode": [
                "updates"
            ]
        }' | \
        sed 's/\r$//' | \
        awk '
        /^event:/ {
            if (data_content != "") {
                print data_content "\n"
            }
            sub(/^event: /, "Receiving event of type: ", $0)
            printf "%s...\n", $0
            data_content = ""
        }
        /^data:/ {
            sub(/^data: /, "", $0)
            data_content = $0
        }
        END {
            if (data_content != "") {
                print data_content "\n\n"
            }
        }
    '
    ```

Output:

    ```
    Receiving event of type: metadata
    {'run_id': '1ef6746e-5893-67b1-978a-0f1cd4060e16'}



    Receiving event of type: updates
    {'agent': {'messages': [{'content': 'I was created by OpenAI, a research organization focused on developing and advancing artificial intelligence technology.', 'additional_kwargs': {}, 'response_metadata': {'finish_reason': 'stop', 'model_name': 'gpt-4o-2024-05-13', 'system_fingerprint': 'fp_157b3831f5'}, 'type': 'ai', 'name': None, 'id': 'run-e1a6b25c-8416-41f2-9981-f9cfe043f414', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]}}
    ```

### LangGraph Platform UI

Inside your deployment, select the "Assistants" tab. For the assistant you would like to use, click the "Studio" button. This will open LangGraph Studio with the selected assistant. When you submit an input (either in Graph or Chat mode), the selected assistant and its configuration will be used.

## Create a new version for your assistant

### LangGraph SDK

To edit the assistant, use the `update` method. This will create a new version of the assistant with the provided edits. See the [Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.AssistantsClient.update) and [JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#update) SDK reference docs for more information.

!!! note "Note"

    You must pass in the ENTIRE config (and metadata if you are using it). The update endpoint creates new versions completely from scratch and does not rely on previous versions.

For example, to update your assistant's system prompt:

=== "Python"

    ```python
    openai_assistant_v2 = await client.assistants.update(
        openai_assistant["assistant_id"],
        config={
            "configurable": {
                "model_name": "openai",
                "system_prompt": "You are an unhelpful assistant!",
            }
        },
    )
    ```

=== "Javascript"

    ```js
    const openaiAssistantV2 = await client.assistants.update(
        openai_assistant["assistant_id"],
        {
            config: {
                configurable: {
                    model_name: 'openai',
                    system_prompt: 'You are an unhelpful assistant!',
                },
        },
    });
    ```

=== "CURL"

    ```bash
    curl --request PATCH \
    --url <DEPOLYMENT_URL>/assistants/<ASSISTANT_ID> \
    --header 'Content-Type: application/json' \
    --data '{
    "config": {"model_name": "openai", "system_prompt": "You are an unhelpful assistant!"}
    }'
    ```

This will create a new version of the assistant with the updated parameters and set this as the active version of your assistant. If you now run your graph and pass in this assistant id, it will use this latest version.

### LangGraph Platform UI

You can also edit assistants from the LangGraph Platform UI.

Inside your deployment, select the "Assistants" tab. This will load a table of all of the assistants in your deployment, across all graphs.

To edit an existing assistant, select the "Edit" button for the specified assistant. This will open a form where you can edit the assistant's name, description, and configuration.

Additionally, if using LangGraph Studio, you can edit the assistants and create new versions via the "Manage Assistants" button.

## Use a previous assistant version

### LangGraph SDK

You can also change the active version of your assistant. To do so, use the `setLatest` method.

In the example above, to rollback to the first version of the assistant:

=== "Python"

    ```python
    await client.assistants.set_latest(openai_assistant['assistant_id'], 1)
    ```

=== "Javascript"

    ```js
    await client.assistants.setLatest(openaiAssistant['assistant_id'], 1);
    ```

=== "CURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/assistants/<ASSISTANT_ID>/latest \
    --header 'Content-Type: application/json' \
    --data '{
    "version": 1
    }'
    ```

If you now run your graph and pass in this assistant id, it will use the first version of the assistant.

### LangGraph Platform UI

If using LangGraph Studio, to set the active version of your assistant, click the "Manage Assistants" button and locate the assistant you would like to use. Select the assistant and the version, and then click the "Active" toggle. This will update the assistant to make the selected version active.

!!! warning "Deleting Assistants"
Deleting as assistant will delete ALL of its versions. There is currently no way to delete a single version, but by pointing your assistant to the correct version you can skip any versions that you don't wish to use.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/cron_jobs.md
```md
# Use cron jobs

Sometimes you don't want to run your graph based on user interaction, but rather you would like to schedule your graph to run on a schedule - for example if you wish for your graph to compose and send out a weekly email of to-dos for your team. LangGraph Platform allows you to do this without having to write your own script by using the `Crons` client. To schedule a graph job, you need to pass a [cron expression](https://crontab.cronhub.io/) to inform the client when you want to run the graph. `Cron` jobs are run in the background and do not interfere with normal invocations of the graph.

## Setup

First, let's set up our SDK client, assistant, and thread:

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # Using the graph deployed with the name "agent"
    assistant_id = "agent"
    # create thread
    thread = await client.threads.create()
    print(thread)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // Using the graph deployed with the name "agent"
    const assistantId = "agent";
    // create thread
    const thread = await client.threads.create();
    console.log(thread);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/assistants/search \
        --header 'Content-Type: application/json' \
        --data '{
            "limit": 10,
            "offset": 0
        }' | jq -c 'map(select(.config == null or .config == {})) | .[0].graph_id' && \
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
    ```

Output:

    {
        'thread_id': '9dde5490-2b67-47c8-aa14-4bfec88af217', 
        'created_at': '2024-08-30T23:07:38.242730+00:00', 
        'updated_at': '2024-08-30T23:07:38.242730+00:00', 
        'metadata': {}, 
        'status': 'idle', 
        'config': {}, 
        'values': None
    }

## Cron job on a thread 

To create a cron job associated with a specific thread, you can write:


=== "Python"

    ```python
    # This schedules a job to run at 15:27 (3:27PM) every day
    cron_job = await client.crons.create_for_thread(
        thread["thread_id"],
        assistant_id,
        schedule="27 15 * * *",
        input={"messages": [{"role": "user", "content": "What time is it?"}]},
    )
    ```

=== "Javascript"

    ```js
    // This schedules a job to run at 15:27 (3:27PM) every day
    const cronJob = await client.crons.create_for_thread(
      thread["thread_id"],
      assistantId,
      {
        schedule: "27 15 * * *",
        input: { messages: [{ role: "user", content: "What time is it?" }] }
      }
    );
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/crons \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <ASSISTANT_ID>,
        }'
    ```

Note that it is **very** important to delete `Cron` jobs that are no longer useful. Otherwise you could rack up unwanted API charges to the LLM! You can delete a `Cron` job using the following code:

=== "Python"

    ```python
    await client.crons.delete(cron_job["cron_id"])
    ```

=== "Javascript"

    ```js
    await client.crons.delete(cronJob["cron_id"]);
    ```

=== "CURL"

    ```bash
    curl --request DELETE \
        --url <DEPLOYMENT_URL>/runs/crons/<CRON_ID>
    ```

## Cron job stateless

You can also create stateless cron jobs by using the following code:

=== "Python"

    ```python
    # This schedules a job to run at 15:27 (3:27PM) every day
    cron_job_stateless = await client.crons.create(
        assistant_id,
        schedule="27 15 * * *",
        input={"messages": [{"role": "user", "content": "What time is it?"}]},
    )
    ```

=== "Javascript"

    ```js
    // This schedules a job to run at 15:27 (3:27PM) every day
    const cronJobStateless = await client.crons.create(
      assistantId,
      {
        schedule: "27 15 * * *",
        input: { messages: [{ role: "user", content: "What time is it?" }] }
      }
    );
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/runs/crons \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <ASSISTANT_ID>,
        }'
    ```

Again, remember to delete your job once you are done with it!

=== "Python"

    ```python
    await client.crons.delete(cron_job_stateless["cron_id"])
    ```

=== "Javascript"

    ```js
    await client.crons.delete(cronJobStateless["cron_id"]);
    ```

=== "CURL"

    ```bash
    curl --request DELETE \
        --url <DEPLOYMENT_URL>/runs/crons/<CRON_ID>
    ```

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/datasets_studio.md
```md
# Add node to dataset

This guide shows how to add examples to [LangSmith datasets](https://docs.smith.langchain.com/evaluation/how_to_guides#dataset-management) from nodes in the thread log. This is useful to evaluate individual steps of the agent.

1. Select a thread.
2. Click on the `Add to Dataset` button.
3. Select nodes whose input/output you want to add to a dataset.
4. For each selected node, select the target dataset to create the example in. By default a dataset for the specific assistant and node will be selected. If this dataset does not yet exist, it will be created.
5. Edit the example's input/output as needed before adding it to the dataset.
6. Select "Add to dataset" at the bottom of the page to add all selected nodes to their respective datasets.

See [Evaluating intermediate steps](https://docs.smith.langchain.com/evaluation/how_to_guides/langgraph#evaluating-intermediate-steps) for more details on how to evaluate intermediate steps.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/enqueue_concurrent.md
```md
# Enqueue

This guide assumes knowledge of what double-texting is, which you can learn about in the [double-texting conceptual guide](../../concepts/double_texting.md).

The guide covers the `enqueue` option for double texting, which adds the interruptions to a queue and executes them in the order they are received by the client. Below is a quick example of using the `enqueue` option.

## Setup

First, we will define a quick helper function for printing out JS and CURL model outputs (you can skip this if using Python):

=== "Javascript"

    ```js
    function prettyPrint(m) {
      const padded = " " + m['type'] + " ";
      const sepLen = Math.floor((80 - padded.length) / 2);
      const sep = "=".repeat(sepLen);
      const secondSep = sep + (padded.length % 2 ? "=" : "");
      
      console.log(`${sep}${padded}${secondSep}`);
      console.log("\n\n");
      console.log(m.content);
    }
    ```

=== "CURL"

    ```bash
    # PLACE THIS IN A FILE CALLED pretty_print.sh
    pretty_print() {
      local type="$1"
      local content="$2"
      local padded=" $type "
      local total_width=80
      local sep_len=$(( (total_width - ${#padded}) / 2 ))
      local sep=$(printf '=%.0s' $(eval "echo {1.."${sep_len}"}"))
      local second_sep=$sep
      if (( (total_width - ${#padded}) % 2 )); then
        second_sep="${second_sep}="
      fi

      echo "${sep}${padded}${second_sep}"
      echo
      echo "$content"
    }
    ```

Then, let's import our required packages and instantiate our client, assistant, and thread.

=== "Python"

    ```python
    import asyncio

    import httpx
    from langchain_core.messages import convert_to_messages
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # Using the graph deployed with the name "agent"
    assistant_id = "agent"
    thread = await client.threads.create()
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // Using the graph deployed with the name "agent"
    const assistantId = "agent";
    const thread = await client.threads.create();
    ```
  
=== "CURL"

    ```bash
    curl --request POST \
      --url <DEPLOYMENT_URL>/threads \
      --header 'Content-Type: application/json' \
      --data '{}'
    ```

## Create runs

Now let's start two runs, with the second interrupting the first one with a multitask strategy of "enqueue":

=== "Python"

    ```python
    first_run = await client.runs.create(
        thread["thread_id"],
        assistant_id,
        input={"messages": [{"role": "user", "content": "what's the weather in sf?"}]},
    )
    second_run = await client.runs.create(
        thread["thread_id"],
        assistant_id,
        input={"messages": [{"role": "user", "content": "what's the weather in nyc?"}]},
        multitask_strategy="enqueue",
    )
    ```

=== "Javascript"

    ```js
    const firstRun = await client.runs.create(
      thread["thread_id"],
      assistantId,
      input={"messages": [{"role": "user", "content": "what's the weather in sf?"}]},
    )

    const secondRun = await client.runs.create(
      thread["thread_id"],
      assistantId,
      input={"messages": [{"role": "user", "content": "what's the weather in nyc?"}]},
      multitask_strategy="enqueue",
    )
    ```

=== "CURL"

    ```bash
    curl --request POST \
    --url <DEPLOY<ENT_URL>>/threads/<THREAD_ID>/runs \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"what\'s the weather in sf?\"}]},
    }" && curl --request POST \
    --url <DEPLOY<ENT_URL>>/threads/<THREAD_ID>/runs \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"what\'s the weather in nyc?\"}]},
      \"multitask_strategy\": \"enqueue\"
    }"
    ```

## View run results

Verify that the thread has data from both runs:

=== "Python"

    ```python
    # wait until the second run completes
    await client.runs.join(thread["thread_id"], second_run["run_id"])

    state = await client.threads.get_state(thread["thread_id"])

    for m in convert_to_messages(state["values"]["messages"]):
        m.pretty_print()
    ```

=== "Javascript"

    ```js
    await client.runs.join(thread["thread_id"], secondRun["run_id"]);

    const state = await client.threads.getState(thread["thread_id"]);

    for (const m of state["values"]["messages"]) {
      prettyPrint(m);
    }
    ```

=== "CURL"

    ```bash
    source pretty_print.sh && curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/<RUN_ID>/join && \
    curl --request GET --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state | \
    jq -c '.values.messages[]' | while read -r element; do
        type=$(echo "$element" | jq -r '.type')
        content=$(echo "$element" | jq -r '.content | if type == "array" then tostring else . end')
        pretty_print "$type" "$content"
    done
    ```

Output:

    ================================ Human Message =================================
    
    what's the weather in sf?
    ================================== Ai Message ==================================
    
    [{'id': 'toolu_01Dez1sJre4oA2Y7NsKJV6VT', 'input': {'query': 'weather in san francisco'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
    Tool Calls:
      tavily_search_results_json (toolu_01Dez1sJre4oA2Y7NsKJV6VT)
     Call ID: toolu_01Dez1sJre4oA2Y7NsKJV6VT
      Args:
        query: weather in san francisco
    ================================= Tool Message =================================
    Name: tavily_search_results_json
    
    [{"url": "https://www.accuweather.com/en/us/san-francisco/94103/weather-forecast/347629", "content": "Get the current and future weather conditions for San Francisco, CA, including temperature, precipitation, wind, air quality and more. See the hourly and 10-day outlook, radar maps, alerts and allergy information."}]
    ================================== Ai Message ==================================
    
    According to AccuWeather, the current weather conditions in San Francisco are:
    
    Temperature: 57°F (14°C)
    Conditions: Mostly Sunny
    Wind: WSW 10 mph
    Humidity: 72%
    
    The forecast for the next few days shows partly sunny skies with highs in the upper 50s to mid 60s F (14-18°C) and lows in the upper 40s to low 50s F (9-11°C). Typical mild, dry weather for San Francisco this time of year.
    
    Some key details from the AccuWeather forecast:
    
    Today: Mostly sunny, high of 62°F (17°C)
    Tonight: Partly cloudy, low of 49°F (9°C) 
    Tomorrow: Partly sunny, high of 59°F (15°C)
    Saturday: Mostly sunny, high of 64°F (18°C)
    Sunday: Partly sunny, high of 61°F (16°C)
    
    So in summary, expect seasonable spring weather in San Francisco over the next several days, with a mix of sun and clouds and temperatures ranging from the upper 40s at night to the low 60s during the days. Typical dry conditions with no rain in the forecast.
    ================================ Human Message =================================
    
    what's the weather in nyc?
    ================================== Ai Message ==================================
    
    [{'text': 'Here are the current weather conditions and forecast for New York City:', 'type': 'text'}, {'id': 'toolu_01FFft5Sx9oS6AdVJuRWWcGp', 'input': {'query': 'weather in new york city'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
    Tool Calls:
      tavily_search_results_json (toolu_01FFft5Sx9oS6AdVJuRWWcGp)
     Call ID: toolu_01FFft5Sx9oS6AdVJuRWWcGp
      Args:
        query: weather in new york city
    ================================= Tool Message =================================
    Name: tavily_search_results_json
    
    [{"url": "https://www.weatherapi.com/", "content": "{'location': {'name': 'New York', 'region': 'New York', 'country': 'United States of America', 'lat': 40.71, 'lon': -74.01, 'tz_id': 'America/New_York', 'localtime_epoch': 1718734479, 'localtime': '2024-06-18 14:14'}, 'current': {'last_updated_epoch': 1718733600, 'last_updated': '2024-06-18 14:00', 'temp_c': 29.4, 'temp_f': 84.9, 'is_day': 1, 'condition': {'text': 'Sunny', 'icon': '//cdn.weatherapi.com/weather/64x64/day/113.png', 'code': 1000}, 'wind_mph': 2.2, 'wind_kph': 3.6, 'wind_degree': 158, 'wind_dir': 'SSE', 'pressure_mb': 1025.0, 'pressure_in': 30.26, 'precip_mm': 0.0, 'precip_in': 0.0, 'humidity': 63, 'cloud': 0, 'feelslike_c': 31.3, 'feelslike_f': 88.3, 'windchill_c': 28.3, 'windchill_f': 82.9, 'heatindex_c': 29.6, 'heatindex_f': 85.3, 'dewpoint_c': 18.4, 'dewpoint_f': 65.2, 'vis_km': 16.0, 'vis_miles': 9.0, 'uv': 7.0, 'gust_mph': 16.5, 'gust_kph': 26.5}}"}]
    ================================== Ai Message ==================================
    
    According to the weather data from WeatherAPI:
    
    Current Conditions in New York City (as of 2:00 PM local time):
    - Temperature: 85°F (29°C)
    - Conditions: Sunny
    - Wind: 2 mph (4 km/h) from the SSE
    - Humidity: 63%
    - Heat Index: 85°F (30°C)
    
    The forecast shows sunny and warm conditions persisting over the next few days:
    
    Today: Sunny, high of 85°F (29°C)
    Tonight: Clear, low of 68°F (20°C)
    Tomorrow: Sunny, high of 88°F (31°C) 
    Thursday: Mostly sunny, high of 90°F (32°C)
    Friday: Partly cloudy, high of 87°F (31°C)
    
    So New York City is experiencing beautiful sunny weather with seasonably warm temperatures in the mid-to-upper 80s Fahrenheit (around 30°C). Humidity is moderate in the 60% range. Overall, ideal late spring/early summer conditions for being outdoors in the city over the next several days.


```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/generative_ui_react.md
```md
# How to implement Generative User Interfaces with LangGraph

!!! info "Prerequisites"

    - [LangGraph Platform](../../concepts/langgraph_platform.md)
    - [LangGraph Server](../../concepts/langgraph_server.md)
    - [`useStream()` React Hook](./use_stream_react.md)

Generative user interfaces (Generative UI) allows agents to go beyond text and generate rich user interfaces. This enables creating more interactive and context-aware applications where the UI adapts based on the conversation flow and AI responses.

![Generative UI Sample](./img/generative_ui_sample.jpg)

LangGraph Platform supports colocating your React components with your graph code. This allows you to focus on building specific UI components for your graph while easily plugging into existing chat interfaces such as [Agent Chat](https://agentchat.vercel.app) and loading the code only when actually needed.

## Tutorial

### 1. Define and configure UI components

First, create your first UI component. For each component you need to provide an unique identifier that will be used to reference the component in your graph code.

```tsx title="src/agent/ui.tsx"
const WeatherComponent = (props: { city: string }) => {
  return <div>Weather for {props.city}</div>;
};

export default {
  weather: WeatherComponent,
};
```

Next, define your UI components in your `langgraph.json` configuration:

=== "Python agent"

    ```json title="langgraph.json"
    {
      "node_version": "20",
      "graphs": {
        "agent": "./src/agent.py:graph"
      },
      "ui": {
        "agent": "./src/agent/ui.tsx"
      }
    }
    ```

=== "JS agent"

    ```json title="langgraph.json"
    {
      "node_version": "20",
      "graphs": {
        "agent": "./src/agent/index.ts:graph"
      },
      "ui": {
        "agent": "./src/agent/ui.tsx"
      }
    }
    ```

The `ui` section points to the UI components that will be used by graphs. By default, we recommend using the same key as the graph name, but you can split out the components however you like, see [Customise the namespace of UI components](#customise-the-namespace-of-ui-components) for more details.

LangGraph Platform will automatically bundle your UI components code and styles and serve them as external assets that can be loaded by the `LoadExternalComponent` component. Some dependencies such as `react` and `react-dom` will be automatically excluded from the bundle.

CSS and Tailwind 4.x is also supported out of the box, so you can freely use Tailwind classes as well as `shadcn/ui` in your UI components.

=== "`src/agent/ui.tsx`"

    ```tsx
    import "./styles.css";

    const WeatherComponent = (props: { city: string }) => {
      return <div className="bg-red-500">Weather for {props.city}</div>;
    };

    export default {
      weather: WeatherComponent,
    };
    ```

=== "`src/agent/styles.css`"

    ```css
    @import "tailwindcss";
    ```

### 2. Send the UI components in your graph

=== "Python"

    ```python title="src/agent.py"
    import uuid
    from typing import Annotated, Sequence, TypedDict

    from langchain_core.messages import AIMessage, BaseMessage
    from langchain_openai import ChatOpenAI
    from langgraph.graph import StateGraph
    from langgraph.graph.message import add_messages
    from langgraph.graph.ui import AnyUIMessage, ui_message_reducer, push_ui_message


    class AgentState(TypedDict):  # noqa: D101
        messages: Annotated[Sequence[BaseMessage], add_messages]
        ui: Annotated[Sequence[AnyUIMessage], ui_message_reducer]


    async def weather(state: AgentState):
        class WeatherOutput(TypedDict):
            city: str

        weather: WeatherOutput = (
            await ChatOpenAI(model="gpt-4o-mini")
            .with_structured_output(WeatherOutput)
            .with_config({"tags": ["nostream"]})
            .ainvoke(state["messages"])
        )

        message = AIMessage(
            id=str(uuid.uuid4()),
            content=f"Here's the weather for {weather['city']}",
        )

        # Emit UI elements associated with the message
        push_ui_message("weather", weather, message=message)
        return {"messages": [message]}


    workflow = StateGraph(AgentState)
    workflow.add_node(weather)
    workflow.add_edge("__start__", "weather")
    graph = workflow.compile()
    ```

=== "JS"

    Use the `typedUi` utility to emit UI elements from your agent nodes:

    ```typescript title="src/agent/index.ts"
    import {
      typedUi,
      uiMessageReducer,
    } from "@langchain/langgraph-sdk/react-ui/server";

    import { ChatOpenAI } from "@langchain/openai";
    import { v4 as uuidv4 } from "uuid";
    import { z } from "zod";

    import type ComponentMap from "./ui.js";

    import {
      Annotation,
      MessagesAnnotation,
      StateGraph,
      type LangGraphRunnableConfig,
    } from "@langchain/langgraph";

    const AgentState = Annotation.Root({
      ...MessagesAnnotation.spec,
      ui: Annotation({ reducer: uiMessageReducer, default: () => [] }),
    });

    export const graph = new StateGraph(AgentState)
      .addNode("weather", async (state, config) => {
        // Provide the type of the component map to ensure
        // type safety of `ui.push()` calls as well as
        // pushing the messages to the `ui` and sending a custom event as well.
        const ui = typedUi<typeof ComponentMap>(config);

        const weather = await new ChatOpenAI({ model: "gpt-4o-mini" })
          .withStructuredOutput(z.object({ city: z.string() }))
          .withConfig({ tags: ["nostream"] })
          .invoke(state.messages);

        const response = {
          id: uuidv4(),
          type: "ai",
          content: `Here's the weather for ${weather.city}`,
        };

        // Emit UI elements associated with the AI message
        ui.push({ name: "weather", props: weather }, { message: response });

        return { messages: [response] };
      })
      .addEdge("__start__", "weather")
      .compile();
    ```

### 3. Handle UI elements in your React application

On the client side, you can use `useStream()` and `LoadExternalComponent` to display the UI elements.

```tsx title="src/app/page.tsx"
"use client";

import { useStream } from "@langchain/langgraph-sdk/react";
import { LoadExternalComponent } from "@langchain/langgraph-sdk/react-ui";

export default function Page() {
  const { thread, values } = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
  });

  return (
    <div>
      {thread.messages.map((message) => (
        <div key={message.id}>
          {message.content}
          {values.ui
            ?.filter((ui) => ui.metadata?.message_id === message.id)
            .map((ui) => (
              <LoadExternalComponent key={ui.id} stream={thread} message={ui} />
            ))}
        </div>
      ))}
    </div>
  );
}
```

Behind the scenes, `LoadExternalComponent` will fetch the JS and CSS for the UI components from LangGraph Platform and render them in a shadow DOM, thus ensuring style isolation from the rest of your application.

## How-to guides

### Provide custom components on the client side

If you already have the components loaded in your client application, you can provide a map of such components to be rendered directly without fetching the UI code from LangGraph Platform.

```tsx
const clientComponents = {
  weather: WeatherComponent,
};

<LoadExternalComponent
  stream={thread}
  message={ui}
  components={clientComponents}
/>;
```

### Show loading UI when components are loading

You can provide a fallback UI to be rendered when the components are loading.

```tsx
<LoadExternalComponent
  stream={thread}
  message={ui}
  fallback={<div>Loading...</div>}
/>
```

### Customise the namespace of UI components.

By default `LoadExternalComponent` will use the `assistantId` from `useStream()` hook to fetch the code for UI components. You can customise this by providing a `namespace` prop to the `LoadExternalComponent` component.

=== "`src/app/page.tsx`"

    ```tsx
    <LoadExternalComponent
      stream={thread}
      message={ui}
      namespace="custom-namespace"
    />
    ```

=== "`langgraph.json`"

    ```json
    {
      "ui": {
        "custom-namespace": "./src/agent/ui.tsx"
      }
    }
    ```

### Access and interact with the thread state from the UI component

You can access the thread state inside the UI component by using the `useStreamContext` hook.

```tsx
import { useStreamContext } from "@langchain/langgraph-sdk/react-ui";

const WeatherComponent = (props: { city: string }) => {
  const { thread, submit } = useStreamContext();
  return (
    <>
      <div>Weather for {props.city}</div>

      <button
        onClick={() => {
          const newMessage = {
            type: "human",
            content: `What's the weather in ${props.city}?`,
          };

          submit({ messages: [newMessage] });
        }}
      >
        Retry
      </button>
    </>
  );
};
```

### Pass additional context to the client components

You can pass additional context to the client components by providing a `meta` prop to the `LoadExternalComponent` component.

```tsx
<LoadExternalComponent stream={thread} message={ui} meta={{ userId: "123" }} />
```

Then, you can access the `meta` prop in the UI component by using the `useStreamContext` hook.

```tsx
import { useStreamContext } from "@langchain/langgraph-sdk/react-ui";

const WeatherComponent = (props: { city: string }) => {
  const { meta } = useStreamContext<
    { city: string },
    { MetaType: { userId?: string } }
  >();

  return (
    <div>
      Weather for {props.city} (user: {meta?.userId})
    </div>
  );
};
```

### Streaming UI messages from the server

You can stream UI messages before the node execution is finished by using the `onCustomEvent` callback of the `useStream()` hook. This is especially useful when updating the UI component as the LLM is generating the response.

```tsx
import { uiMessageReducer } from "@langchain/langgraph-sdk/react-ui";

const { thread, submit } = useStream({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  onCustomEvent: (event, options) => {
    options.mutate((prev) => {
      const ui = uiMessageReducer(prev.ui ?? [], event);
      return { ...prev, ui };
    });
  },
});
```

Then you can push updates to the UI component by calling `ui.push()` / `push_ui_message()` with the same ID as the UI message you wish to update.

=== "Python"

    ```python
    from typing import Annotated, Sequence, TypedDict

    from langchain_anthropic import ChatAnthropic
    from langchain_core.messages import AIMessage, AIMessageChunk, BaseMessage
    from langgraph.graph import StateGraph
    from langgraph.graph.message import add_messages
    from langgraph.graph.ui import AnyUIMessage, push_ui_message, ui_message_reducer


    class AgentState(TypedDict):  # noqa: D101
        messages: Annotated[Sequence[BaseMessage], add_messages]
        ui: Annotated[Sequence[AnyUIMessage], ui_message_reducer]


    class CreateTextDocument(TypedDict):
        """Prepare a document heading for the user."""

        title: str


    async def writer_node(state: AgentState):
        model = ChatAnthropic(model="claude-3-5-sonnet-latest")
        message: AIMessage = await model.bind_tools(
            tools=[CreateTextDocument],
            tool_choice={"type": "tool", "name": "CreateTextDocument"},
        ).ainvoke(state["messages"])

        tool_call = next(
            (x["args"] for x in message.tool_calls if x["name"] == "CreateTextDocument"),
            None,
        )

        if tool_call:
            ui_message = push_ui_message("writer", tool_call, message=message)
            ui_message_id = ui_message["id"]

            # We're already streaming the LLM response to the client through UI messages
            # so we don't need to stream it again to the `messages` stream mode.
            content_stream = model.with_config({"tags": ["nostream"]}).astream(
                f"Create a document with the title: {tool_call['title']}"
            )

            content: AIMessageChunk | None = None
            async for chunk in content_stream:
                content = content + chunk if content else chunk

                push_ui_message(
                    "writer",
                    {"content": content.text()},
                    id=ui_message_id,
                    message=message,
                    # Use `merge=rue` to merge props with the existing UI message
                    merge=True,
                )

        return {"messages": [message]}
    ```

=== "JS"

    ```tsx
    import {
      Annotation,
      MessagesAnnotation,
      type LangGraphRunnableConfig,
    } from "@langchain/langgraph";
    import { z } from "zod";
    import { ChatAnthropic } from "@langchain/anthropic";
    import {
      typedUi,
      uiMessageReducer,
    } from "@langchain/langgraph-sdk/react-ui/server";
    import type { AIMessageChunk } from "@langchain/core/messages";

    import type ComponentMap from "./ui";

    const AgentState = Annotation.Root({
      ...MessagesAnnotation.spec,
      ui: Annotation({ reducer: uiMessageReducer, default: () => [] }),
    });

    async function writerNode(
      state: typeof AgentState.State,
      config: LangGraphRunnableConfig
    ): Promise<typeof AgentState.Update> {
      const ui = typedUi<typeof ComponentMap>(config);

      const model = new ChatAnthropic({ model: "claude-3-5-sonnet-latest" });
      const message = await model
        .bindTools(
          [
            {
              name: "create_text_document",
              description: "Prepare a document heading for the user.",
              schema: z.object({ title: z.string() }),
            },
          ],
          { tool_choice: { type: "tool", name: "create_text_document" } }
        )
        .invoke(state.messages);

      type ToolCall = { name: "create_text_document"; args: { title: string } };
      const toolCall = message.tool_calls?.find(
        (tool): tool is ToolCall => tool.name === "create_text_document"
      );

      if (toolCall) {
        const { id, name } = ui.push(
          { name: "writer", props: { title: toolCall.args.title } },
          { message }
        );

        const contentStream = await model
          // We're already streaming the LLM response to the client through UI messages
          // so we don't need to stream it again to the `messages` stream mode.
          .withConfig({ tags: ["nostream"] })
          .stream(`Create a short poem with the topic: ${message.text}`);

        let content: AIMessageChunk | undefined;
        for await (const chunk of contentStream) {
          content = content?.concat(chunk) ?? chunk;

          ui.push(
            { id, name, props: { content: content?.text } },
            // Use `merge: true` to merge props with the existing UI message
            { message, merge: true }
          );
        }
      }

      return { messages: [message] };
    }
    ```

=== "`ui.tsx`"

    ```tsx
    function WriterComponent(props: { title: string; content?: string }) {
      return (
        <article>
          <h2>{props.title}</h2>
          <p style={{ whiteSpace: "pre-wrap" }}>{props.content}</p>
        </article>
      );
    }

    export default {
      weather: WriterComponent,
    };
    ```

### Remove UI messages from state

Similar to how messages can be removed from the state by appending a RemoveMessage you can remove an UI message from the state by calling `remove_ui_message` / `ui.delete` with the ID of the UI message.

=== "Python"

    ```python
    from langgraph.graph.ui import push_ui_message, delete_ui_message

    # push message
    message = push_ui_message("weather", {"city": "London"})

    # remove said message
    delete_ui_message(message["id"])
    ```

=== "JS"

    ```tsx
    // push message
    const message = ui.push({ name: "weather", props: { city: "London" } });

    // remove said message
    ui.delete(message.id);
    ```

## Learn more

- [JS/TS SDK Reference](../reference/sdk/js_ts_sdk_ref.md)

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/human_in_the_loop_time_travel.md
```md
# Time travel using Server API

LangGraph provides the [**time travel**](../../concepts/time-travel.md) functionality to resume execution from a prior checkpoint, either replaying the same state or modifying it to explore alternatives. In all cases, resuming past execution produces a new fork in the history.

To time travel using the LangGraph Server API (via the LangGraph SDK):

1. **Run the graph** with initial inputs using [LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/)'s @[`client.runs.wait`][client.runs.wait] or @[`client.runs.stream`][client.runs.stream] APIs.
2. **Identify a checkpoint in an existing thread**: Use @[`client.threads.get_history`][client.threads.get_history] method to retrieve the execution history for a specific `thread_id` and locate the desired `checkpoint_id`.
   Alternatively, set a [breakpoint](./human_in_the_loop_breakpoint.md) before the node(s) where you want execution to pause. You can then find the most recent checkpoint recorded up to that breakpoint.
3. **(Optional) modify the graph state**: Use the @[`client.threads.update_state`][client.threads.update_state] method to modify the graph’s state at the checkpoint and resume execution from alternative state.
4. **Resume execution from the checkpoint**: Use the @[`client.runs.wait`][client.runs.wait] or @[`client.runs.stream`][client.runs.stream] APIs with an input of `None` and the appropriate `thread_id` and `checkpoint_id`.

## Use time travel in a workflow

??? example "Example graph"

    ```python
    from typing_extensions import TypedDict, NotRequired
    from langgraph.graph import StateGraph, START, END
    from langchain.chat_models import init_chat_model
    from langgraph.checkpoint.memory import InMemorySaver

    class State(TypedDict):
        topic: NotRequired[str]
        joke: NotRequired[str]

    llm = init_chat_model(
        "anthropic:claude-3-7-sonnet-latest",
        temperature=0,
    )

    def generate_topic(state: State):
        """LLM call to generate a topic for the joke"""
        msg = llm.invoke("Give me a funny topic for a joke")
        return {"topic": msg.content}

    def write_joke(state: State):
        """LLM call to write a joke based on the topic"""
        msg = llm.invoke(f"Write a short joke about {state['topic']}")
        return {"joke": msg.content}

    # Build workflow
    builder = StateGraph(State)

    # Add nodes
    builder.add_node("generate_topic", generate_topic)
    builder.add_node("write_joke", write_joke)

    # Add edges to connect nodes
    builder.add_edge(START, "generate_topic")
    builder.add_edge("generate_topic", "write_joke")

    # Compile
    graph = builder.compile()
    ```

### 1. Run the graph

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>)

    # Using the graph deployed with the name "agent"
    assistant_id = "agent"

    # create a thread
    thread = await client.threads.create()
    thread_id = thread["thread_id"]

    # Run the graph
    result = await client.runs.wait(
        thread_id,
        assistant_id,
        input={}
    )
    ```

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

    // Using the graph deployed with the name "agent"
    const assistantID = "agent";

    // create a thread
    const thread = await client.threads.create();
    const threadID = thread["thread_id"];

    // Run the graph
    const result = await client.runs.wait(
      threadID,
      assistantID,
      { input: {}}
    );
    ```

=== "cURL"

    Create a thread:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads \
    --header 'Content-Type: application/json' \
    --data '{}'
    ```

    Run the graph:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {}
    }"
    ```

### 2. Identify a checkpoint

=== "Python"

    ```python
    # The states are returned in reverse chronological order.
    states = await client.threads.get_history(thread_id)
    selected_state = states[1]
    print(selected_state)
    ```

=== "JavaScript"

    ```js
    // The states are returned in reverse chronological order.
    const states = await client.threads.getHistory(threadID);
    const selectedState = states[1];
    console.log(selectedState);
    ```

=== "cURL"

    ```bash
    curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/history \
    --header 'Content-Type: application/json'
    ```

### 3. Update the state (optional)

`update_state` will create a new checkpoint. The new checkpoint will be associated with the same thread, but a new checkpoint ID.

=== "Python"

    ```python
    new_config = await client.threads.update_state(
        thread_id,
        {"topic": "chickens"},
        # highlight-next-line
        checkpoint_id=selected_state["checkpoint_id"]
    )
    print(new_config)
    ```

=== "JavaScript"

    ```js
    const newConfig = await client.threads.updateState(
      threadID,
      {
        values: { "topic": "chickens" },
        checkpointId: selectedState["checkpoint_id"]
      }
    );
    console.log(newConfig);
    ```

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"checkpoint_id\": <CHECKPOINT_ID>,
      \"values\": {\"topic\": \"chickens\"}
    }"
    ```

### 4. Resume execution from the checkpoint

=== "Python"

    ```python
    await client.runs.wait(
        thread_id,
        assistant_id,
        # highlight-next-line
        input=None,
        # highlight-next-line
        checkpoint_id=new_config["checkpoint_id"]
    )
    ```

=== "JavaScript"

    ```js
    await client.runs.wait(
      threadID,
      assistantID,
      {
        // highlight-next-line
        input: null,
        // highlight-next-line
        checkpointId: newConfig["checkpoint_id"]
      }
    );
    ```

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"checkpoint_id\": <CHECKPOINT_ID>
    }"
    ```

## Learn more

- [**LangGraph time travel guide**](../../how-tos/human_in_the_loop/time-travel.md): learn more about using time travel in LangGraph.
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/interrupt_concurrent.md
```md
# How to use the interrupt option

This guide assumes knowledge of what double-texting is, which you can learn about in the [double-texting conceptual guide](../../concepts/double_texting.md).

The guide covers the `interrupt` option for double texting, which interrupts the prior run of the graph and starts a new one with the double-text. This option does not delete the first run, but rather keeps it in the database but sets its status to `interrupted`. Below is a quick example of using the `interrupt` option.

## Setup

First, we will define a quick helper function for printing out JS and CURL model outputs (you can skip this if using Python):

=== "Javascript"

    ```js
    function prettyPrint(m) {
      const padded = " " + m['type'] + " ";
      const sepLen = Math.floor((80 - padded.length) / 2);
      const sep = "=".repeat(sepLen);
      const secondSep = sep + (padded.length % 2 ? "=" : "");
      
      console.log(`${sep}${padded}${secondSep}`);
      console.log("\n\n");
      console.log(m.content);
    }
    ```

=== "CURL"

    ```bash
    # PLACE THIS IN A FILE CALLED pretty_print.sh
    pretty_print() {
      local type="$1"
      local content="$2"
      local padded=" $type "
      local total_width=80
      local sep_len=$(( (total_width - ${#padded}) / 2 ))
      local sep=$(printf '=%.0s' $(eval "echo {1.."${sep_len}"}"))
      local second_sep=$sep
      if (( (total_width - ${#padded}) % 2 )); then
        second_sep="${second_sep}="
      fi

      echo "${sep}${padded}${second_sep}"
      echo
      echo "$content"
    }
    ```

Now, let's import our required packages and instantiate our client, assistant, and thread.

=== "Python"

    ```python
    import asyncio

    from langchain_core.messages import convert_to_messages
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # Using the graph deployed with the name "agent"
    assistant_id = "agent"
    thread = await client.threads.create()
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // Using the graph deployed with the name "agent"
    const assistantId = "agent";
    const thread = await client.threads.create();
    ```

=== "CURL"

    ```bash
    curl --request POST \
      --url <DEPLOYMENT_URL>/threads \
      --header 'Content-Type: application/json' \
      --data '{}'
    ```

## Create runs

Now we can start our two runs and join the second one until it has completed:

=== "Python"

    ```python
    # the first run will be interrupted
    interrupted_run = await client.runs.create(
        thread["thread_id"],
        assistant_id,
        input={"messages": [{"role": "user", "content": "what's the weather in sf?"}]},
    )
    # sleep a bit to get partial outputs from the first run
    await asyncio.sleep(2)
    run = await client.runs.create(
        thread["thread_id"],
        assistant_id,
        input={"messages": [{"role": "user", "content": "what's the weather in nyc?"}]},
        multitask_strategy="interrupt",
    )
    # wait until the second run completes
    await client.runs.join(thread["thread_id"], run["run_id"])
    ```

=== "Javascript"

    ```js
    // the first run will be interrupted
    let interruptedRun = await client.runs.create(
      thread["thread_id"],
      assistantId,
      { input: { messages: [{ role: "human", content: "what's the weather in sf?" }] } }
    );
    // sleep a bit to get partial outputs from the first run
    await new Promise(resolve => setTimeout(resolve, 2000)); 

    let run = await client.runs.create(
      thread["thread_id"],
      assistantId,
      { 
        input: { messages: [{ role: "human", content: "what's the weather in nyc?" }] },
        multitaskStrategy: "interrupt" 
      }
    );

    // wait until the second run completes
    await client.runs.join(thread["thread_id"], run["run_id"]);
    ```

=== "CURL"

    ```bash
    curl --request POST \
    --url <DEPLOY<ENT_URL>>/threads/<THREAD_ID>/runs \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"what\'s the weather in sf?\"}]},
    }" && sleep 2 && curl --request POST \
    --url <DEPLOY<ENT_URL>>/threads/<THREAD_ID>/runs \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"what\'s the weather in nyc?\"}]},
      \"multitask_strategy\": \"interrupt\"
    }" && curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/<RUN_ID>/join
    ```

## View run results

We can see that the thread has partial data from the first run + data from the second run


=== "Python"

    ```python
    state = await client.threads.get_state(thread["thread_id"])

    for m in convert_to_messages(state["values"]["messages"]):
        m.pretty_print()
    ```

=== "Javascript"

    ```js
    const state = await client.threads.getState(thread["thread_id"]);

    for (const m of state['values']['messages']) {
      prettyPrint(m);
    }
    ```

=== "CURL"

    ```bash
    source pretty_print.sh && curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state | \
    jq -c '.values.messages[]' | while read -r element; do
        type=$(echo "$element" | jq -r '.type')
        content=$(echo "$element" | jq -r '.content | if type == "array" then tostring else . end')
        pretty_print "$type" "$content"
    done
    ```

Output:

    ================================ Human Message =================================
    
    what's the weather in sf?
    ================================== Ai Message ==================================
    
    [{'id': 'toolu_01MjNtVJwEcpujRGrf3x6Pih', 'input': {'query': 'weather in san francisco'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
    Tool Calls:
      tavily_search_results_json (toolu_01MjNtVJwEcpujRGrf3x6Pih)
     Call ID: toolu_01MjNtVJwEcpujRGrf3x6Pih
      Args:
        query: weather in san francisco
    ================================= Tool Message =================================
    Name: tavily_search_results_json
    
    [{"url": "https://www.wunderground.com/hourly/us/ca/san-francisco/KCASANFR2002/date/2024-6-18", "content": "High 64F. Winds W at 10 to 20 mph. A few clouds from time to time. Low 49F. Winds W at 10 to 20 mph. Temp. San Francisco Weather Forecasts. Weather Underground provides local & long-range weather ..."}]
    ================================ Human Message =================================
    
    what's the weather in nyc?
    ================================== Ai Message ==================================
    
    [{'id': 'toolu_01KtE1m1ifPLQAx4fQLyZL9Q', 'input': {'query': 'weather in new york city'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
    Tool Calls:
      tavily_search_results_json (toolu_01KtE1m1ifPLQAx4fQLyZL9Q)
     Call ID: toolu_01KtE1m1ifPLQAx4fQLyZL9Q
      Args:
        query: weather in new york city
    ================================= Tool Message =================================
    Name: tavily_search_results_json
    
    [{"url": "https://www.accuweather.com/en/us/new-york/10021/june-weather/349727", "content": "Get the monthly weather forecast for New York, NY, including daily high/low, historical averages, to help you plan ahead."}]
    ================================== Ai Message ==================================
    
    The search results provide weather forecasts and information for New York City. Based on the top result from AccuWeather, here are some key details about the weather in NYC:
    
    - This is a monthly weather forecast for New York City for the month of June.
    - It includes daily high and low temperatures to help plan ahead.
    - Historical averages for June in NYC are also provided as a reference point.
    - More detailed daily or hourly forecasts with precipitation chances, humidity, wind, etc. can be found by visiting the AccuWeather page.
    
    So in summary, the search provides a convenient overview of the expected weather conditions in New York City over the next month to give you an idea of what to prepare for if traveling or making plans there. Let me know if you need any other details!


Verify that the original, interrupted run was interrupted

=== "Python"

    ```python
    print((await client.runs.get(thread["thread_id"], interrupted_run["run_id"]))["status"])
    ```

=== "Javascript"

    ```js
    console.log((await client.runs.get(thread['thread_id'], interruptedRun["run_id"]))["status"])
    ```

Output:

    ```
    'interrupted'
    ```


```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/invoke_studio.md
```md
# Run application

!!!info  "Prerequisites"
    - [Running agents](../../agents/run_agents.md#running-agents)

This guide shows how to submit a [run](../../concepts/assistants.md#execution) to your application.

## Graph mode

### Specify input
First define the input to your graph with in the "Input" section on the left side of the page, below the graph interface.

Studio will attempt to render a form for your input based on the graph's defined [state schema](../../concepts/low_level.md/#schema). To disable this, click the "View Raw" button, which will present you with a JSON editor.

Click the up/down arrows at the top of the "Input" section to toggle through and use previously submitted inputs.

### Run settings

#### Assistant

To specify the [assistant](../../concepts/assistants.md) that is used for the run click the settings button in the bottom left corner. If an assistant is currently selected the button will also list the assistant name. If no assistant is selected it will say "Manage Assistants".

Select the assistant to run and click the "Active" toggle at the top of the modal to activate it. [See here](./studio/manage_assistants.md) for more information on managing assistants.

#### Streaming
Click the dropdown next to "Submit" and click the toggle to enable/disable streaming.

#### Breakpoints
To run your graph with breakpoints, click the "Interrupt" button. Select a node and whether to pause before and/or after that node has executed. Click "Continue" in the thread log to resume execution.


For more information on breakpoints see [here](../../concepts/human_in_the_loop.md).

### Submit run

To submit the run with the specified input and run settings, click the "Submit" button. This will add a [run](../../concepts/assistants.md#execution) to the existing selected [thread](../../concepts/persistence.md#threads). If no thread is currently selected, a new one will be created.

To cancel the ongoing run, click the "Cancel" button.


## Chat mode
Specify the input to your chat application in the bottom of the conversation panel. Click the "Send message" button to submit the input as a Human message and have the response streamed back.

To cancel the ongoing run, click the "Cancel" button. Click the "Show tool calls" toggle to hide/show tool calls in the conversation.

## Learn more

To run your application from a specific checkpoint in an existing thread, see [this guide](./threads_studio.md#edit-thread-history).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/iterate_graph_studio.md
```md
# Iterate on prompts

## Overview

LangGraph Studio supports two methods for modifying prompts in your graph: direct node editing and the LangSmith Playground interface.

## Direct Node Editing

Studio allows you to edit prompts used inside individual nodes, directly from the graph interface.

!!! info "Prerequisites"

    - [Assistants overview](../../concepts/assistants.md)

### Graph Configuration

Define your [configuration](https://langchain-ai.github.io/langgraph/how-tos/configuration/) to specify prompt fields and their associated nodes using `langgraph_nodes` and `langgraph_type` keys.

#### Configuration Reference

##### `langgraph_nodes`

- **Description**: Specifies which nodes of the graph a configuration field is associated with.
- **Value Type**: Array of strings, where each string is the name of a node in your graph.
- **Usage Context**: Include in the `json_schema_extra` dictionary for Pydantic models or the `metadata["json_schema_extra"]` dictionary for dataclasses.
- **Example**:
  ```python
  system_prompt: str = Field(
      default="You are a helpful AI assistant.",
      json_schema_extra={"langgraph_nodes": ["call_model", "other_node"]},
  )
  ```

##### `langgraph_type`

- **Description**: Specifies the type of configuration field, which determines how it's handled in the UI.
- **Value Type**: String
- **Supported Values**:
  - `"prompt"`: Indicates the field contains prompt text that should be treated specially in the UI.
- **Usage Context**: Include in the `json_schema_extra` dictionary for Pydantic models or the `metadata["json_schema_extra"]` dictionary for dataclasses.
- **Example**:
  ```python
  system_prompt: str = Field(
      default="You are a helpful AI assistant.",
      json_schema_extra={
          "langgraph_nodes": ["call_model"],
          "langgraph_type": "prompt",
      },
  )
  ```

#### Example Configuration

```python
## Using Pydantic
from pydantic import BaseModel, Field
from typing import Annotated, Literal

class Configuration(BaseModel):
    """The configuration for the agent."""

    system_prompt: str = Field(
        default="You are a helpful AI assistant.",
        description="The system prompt to use for the agent's interactions. "
        "This prompt sets the context and behavior for the agent.",
        json_schema_extra={
            "langgraph_nodes": ["call_model"],
            "langgraph_type": "prompt",
        },
    )

    model: Annotated[
        Literal[
            "anthropic/claude-3-7-sonnet-latest",
            "anthropic/claude-3-5-haiku-latest",
            "openai/o1",
            "openai/gpt-4o-mini",
            "openai/o1-mini",
            "openai/o3-mini",
        ],
        {"__template_metadata__": {"kind": "llm"}},
    ] = Field(
        default="openai/gpt-4o-mini",
        description="The name of the language model to use for the agent's main interactions. "
        "Should be in the form: provider/model-name.",
        json_schema_extra={"langgraph_nodes": ["call_model"]},
    )

## Using Dataclasses
from dataclasses import dataclass, field

@dataclass(kw_only=True)
class Configuration:
    """The configuration for the agent."""

    system_prompt: str = field(
        default="You are a helpful AI assistant.",
        metadata={
            "description": "The system prompt to use for the agent's interactions. "
            "This prompt sets the context and behavior for the agent.",
            "json_schema_extra": {"langgraph_nodes": ["call_model"]},
        },
    )

    model: Annotated[str, {"__template_metadata__": {"kind": "llm"}}] = field(
        default="anthropic/claude-3-5-sonnet-20240620",
        metadata={
            "description": "The name of the language model to use for the agent's main interactions. "
            "Should be in the form: provider/model-name.",
            "json_schema_extra": {"langgraph_nodes": ["call_model"]},
        },
    )

```

### Editing prompts in UI

1. Locate the gear icon on nodes with associated configuration fields
2. Click to open the configuration modal
3. Edit the values
4. Save to update the current assistant version or create a new one

## LangSmith Playground

The [LangSmith Playground](https://
docs.smith.langchain.com/prompt_engineering/how_to_guides#playground) interface allows testing individual LLM calls without running the full graph:

1. Select a thread
2. Click "View LLM Runs" on a node. This lists all the LLM calls (if any) made inside the node.
3. Select an LLM run to open in Playground
4. Modify prompts and test different model and tool settings
5. Copy updated prompts back to your graph

For advanced Playground features, click the expand button in the top right corner.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/reject_concurrent.md
```md
# Reject

This guide assumes knowledge of what double-texting is, which you can learn about in the [double-texting conceptual guide](../../concepts/double_texting.md).

The guide covers the `reject` option for double texting, which rejects the new run of the graph by throwing an error and continues with the original run until completion. Below is a quick example of using the `reject` option.

## Setup

First, we will define a quick helper function for printing out JS and CURL model outputs (you can skip this if using Python):

=== "Javascript"

    ```js
    function prettyPrint(m) {
      const padded = " " + m['type'] + " ";
      const sepLen = Math.floor((80 - padded.length) / 2);
      const sep = "=".repeat(sepLen);
      const secondSep = sep + (padded.length % 2 ? "=" : "");
      
      console.log(`${sep}${padded}${secondSep}`);
      console.log("\n\n");
      console.log(m.content);
    }
    ```

=== "CURL"

    ```bash
    # PLACE THIS IN A FILE CALLED pretty_print.sh
    pretty_print() {
      local type="$1"
      local content="$2"
      local padded=" $type "
      local total_width=80
      local sep_len=$(( (total_width - ${#padded}) / 2 ))
      local sep=$(printf '=%.0s' $(eval "echo {1.."${sep_len}"}"))
      local second_sep=$sep
      if (( (total_width - ${#padded}) % 2 )); then
        second_sep="${second_sep}="
      fi

      echo "${sep}${padded}${second_sep}"
      echo
      echo "$content"
    }
    ```

Now, let's import our required packages and instantiate our client, assistant, and thread.

=== "Python"

    ```python
    import httpx
    from langchain_core.messages import convert_to_messages
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # Using the graph deployed with the name "agent"
    assistant_id = "agent"
    thread = await client.threads.create()
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // Using the graph deployed with the name "agent"
    const assistantId = "agent";
    const thread = await client.threads.create();
    ```

=== "CURL"

    ```bash
    curl --request POST \
      --url <DEPLOYMENT_URL>/threads \
      --header 'Content-Type: application/json' \
      --data '{}'
    ```

## Create runs

Now we can run a thread and try to run a second one with the "reject" option, which should fail since we have already started a run:


=== "Python"

    ```python
    run = await client.runs.create(
        thread["thread_id"],
        assistant_id,
        input={"messages": [{"role": "user", "content": "what's the weather in sf?"}]},
    )
    try:
        await client.runs.create(
            thread["thread_id"],
            assistant_id,
            input={
                "messages": [{"role": "user", "content": "what's the weather in nyc?"}]
            },
            multitask_strategy="reject",
        )
    except httpx.HTTPStatusError as e:
        print("Failed to start concurrent run", e)
    ```

=== "Javascript"

    ```js
    const run = await client.runs.create(
      thread["thread_id"],
      assistantId,
      input={"messages": [{"role": "user", "content": "what's the weather in sf?"}]},
    );
    
    try {
      await client.runs.create(
        thread["thread_id"],
        assistantId,
        { 
          input: {"messages": [{"role": "user", "content": "what's the weather in nyc?"}]},
          multitask_strategy:"reject"
        },
      );
    } catch (e) {
      console.error("Failed to start concurrent run", e);
    }
    ```

=== "CURL"

    ```bash
    curl --request POST \
    --url <DEPLOY<ENT_URL>>/threads/<THREAD_ID>/runs \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"what\'s the weather in sf?\"}]},
    }" && curl --request POST \
    --url <DEPLOY<ENT_URL>>/threads/<THREAD_ID>/runs \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"what\'s the weather in nyc?\"}]},
      \"multitask_strategy\": \"reject\"
    }" || { echo "Failed to start concurrent run"; echo "Error: $?" >&2; }
    ```

Output:

    Failed to start concurrent run Client error '409 Conflict' for url 'http://localhost:8123/threads/f9e7088b-8028-4e5c-88d2-9cc9a2870e50/runs'
    For more information check: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/409

## View run results

We can verify that the original thread finished executing:

=== "Python"

    ```python
    # wait until the original run completes
    await client.runs.join(thread["thread_id"], run["run_id"])

    state = await client.threads.get_state(thread["thread_id"])

    for m in convert_to_messages(state["values"]["messages"]):
        m.pretty_print()
    ```

=== "Javascript"

    ```js
    await client.runs.join(thread["thread_id"], run["run_id"]);

    const state = await client.threads.getState(thread["thread_id"]);

    for (const m of state["values"]["messages"]) {
      prettyPrint(m);
    }
    ```

=== "CURL"

    ```bash
    source pretty_print.sh && curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/<RUN_ID>/join && \
    curl --request GET --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state | \
    jq -c '.values.messages[]' | while read -r element; do
        type=$(echo "$element" | jq -r '.type')
        content=$(echo "$element" | jq -r '.content | if type == "array" then tostring else . end')
        pretty_print "$type" "$content"
    done
    ```

Output:

    ================================ Human Message =================================
    
    what's the weather in sf?
    ================================== Ai Message ==================================
    
    [{'id': 'toolu_01CyewEifV2Kmi7EFKHbMDr1', 'input': {'query': 'weather in san francisco'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
    Tool Calls:
      tavily_search_results_json (toolu_01CyewEifV2Kmi7EFKHbMDr1)
     Call ID: toolu_01CyewEifV2Kmi7EFKHbMDr1
      Args:
        query: weather in san francisco
    ================================= Tool Message =================================
    Name: tavily_search_results_json
    
    [{"url": "https://www.accuweather.com/en/us/san-francisco/94103/june-weather/347629", "content": "Get the monthly weather forecast for San Francisco, CA, including daily high/low, historical averages, to help you plan ahead."}]
    ================================== Ai Message ==================================
    
    According to the search results from Tavily, the current weather in San Francisco is:
    
    The average high temperature in San Francisco in June is around 65°F (18°C), with average lows around 54°F (12°C). June tends to be one of the cooler and foggier months in San Francisco due to the marine layer of fog that often blankets the city during the summer months.
    
    Some key points about the typical June weather in San Francisco:
    
    - Mild temperatures with highs in the 60s F and lows in the 50s F
    - Foggy mornings that often burn off to sunny afternoons
    - Little to no rainfall, as June falls in the dry season
    - Breezy conditions, with winds off the Pacific Ocean
    - Layers are recommended for changing weather conditions
    
    So in summary, you can expect mild, foggy mornings giving way to sunny but cool afternoons in San Francisco this time of year. The marine layer keeps temperatures moderate compared to other parts of California in June.


```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/rollback_concurrent.md
```md
# How to use the Rollback option


This guide assumes knowledge of what double-texting is, which you can learn about in the [double-texting conceptual guide](../../concepts/double_texting.md).

The guide covers the `rollback` option for double texting, which interrupts the prior run of the graph and starts a new one with the double-text. This option is very similar to the `interrupt` option, but in this case the first run is completely deleted from the database and cannot be restarted. Below is a quick example of using the `rollback` option.

## Setup

First, we will define a quick helper function for printing out JS and CURL model outputs (you can skip this if using Python):

=== "Javascript"

    ```js
    function prettyPrint(m) {
      const padded = " " + m['type'] + " ";
      const sepLen = Math.floor((80 - padded.length) / 2);
      const sep = "=".repeat(sepLen);
      const secondSep = sep + (padded.length % 2 ? "=" : "");
      
      console.log(`${sep}${padded}${secondSep}`);
      console.log("\n\n");
      console.log(m.content);
    }
    ```

=== "CURL"

    ```bash
    # PLACE THIS IN A FILE CALLED pretty_print.sh
    pretty_print() {
      local type="$1"
      local content="$2"
      local padded=" $type "
      local total_width=80
      local sep_len=$(( (total_width - ${#padded}) / 2 ))
      local sep=$(printf '=%.0s' $(eval "echo {1.."${sep_len}"}"))
      local second_sep=$sep
      if (( (total_width - ${#padded}) % 2 )); then
        second_sep="${second_sep}="
      fi

      echo "${sep}${padded}${second_sep}"
      echo
      echo "$content"
    }
    ```

Now, let's import our required packages and instantiate our client, assistant, and thread.

=== "Python"

    ```python
    import asyncio

    import httpx
    from langchain_core.messages import convert_to_messages
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # Using the graph deployed with the name "agent"
    assistant_id = "agent"
    thread = await client.threads.create()
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // Using the graph deployed with the name "agent"
    const assistantId = "agent";
    const thread = await client.threads.create();
    ```

=== "CURL"

    ```bash
    curl --request POST \
      --url <DEPLOYMENT_URL>/threads \
      --header 'Content-Type: application/json' \
      --data '{}'
    ```

## Create runs

Now let's run a thread with the multitask parameter set to "rollback":

=== "Python"

    ```python
    # the first run will be rolled back
    rolled_back_run = await client.runs.create(
        thread["thread_id"],
        assistant_id,
        input={"messages": [{"role": "user", "content": "what's the weather in sf?"}]},
    )
    run = await client.runs.create(
        thread["thread_id"],
        assistant_id,
        input={"messages": [{"role": "user", "content": "what's the weather in nyc?"}]},
        multitask_strategy="rollback",
    )
    # wait until the second run completes
    await client.runs.join(thread["thread_id"], run["run_id"])
    ```

=== "Javascript"

    ```js
    // the first run will be interrupted
    let rolledBackRun = await client.runs.create(
      thread["thread_id"],
      assistantId,
      { input: { messages: [{ role: "human", content: "what's the weather in sf?" }] } }
    );

    let run = await client.runs.create(
      thread["thread_id"],
      assistant_id,
      { 
        input: { messages: [{ role: "human", content: "what's the weather in nyc?" }] },
        multitaskStrategy: "rollback" 
      }
    );

    // wait until the second run completes
    await client.runs.join(thread["thread_id"], run["run_id"]);
    ```

=== "CURL"

    ```bash
    curl --request POST \
    --url <DEPLOY<ENT_URL>>/threads/<THREAD_ID>/runs \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"what\'s the weather in sf?\"}]},
    }" && curl --request POST \
    --url <DEPLOY<ENT_URL>>/threads/<THREAD_ID>/runs \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"what\'s the weather in nyc?\"}]},
      \"multitask_strategy\": \"rollback\"
    }" && curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/<RUN_ID>/join
    ```

## View run results

We can see that the thread has data only from the second run

=== "Python"

    ```python
    state = await client.threads.get_state(thread["thread_id"])

    for m in convert_to_messages(state["values"]["messages"]):
        m.pretty_print()
    ```

=== "Javascript"

    ```js
    const state = await client.threads.getState(thread["thread_id"]);

    for (const m of state['values']['messages']) {
      prettyPrint(m);
    }
    ```

=== "CURL"

    ```bash
    source pretty_print.sh && curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state | \
    jq -c '.values.messages[]' | while read -r element; do
        type=$(echo "$element" | jq -r '.type')
        content=$(echo "$element" | jq -r '.content | if type == "array" then tostring else . end')
        pretty_print "$type" "$content"
    done
    ```

Output:

    ================================ Human Message =================================
    
    what's the weather in nyc?
    ================================== Ai Message ==================================
    
    [{'id': 'toolu_01JzPqefao1gxwajHQ3Yh3JD', 'input': {'query': 'weather in nyc'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
    Tool Calls:
      tavily_search_results_json (toolu_01JzPqefao1gxwajHQ3Yh3JD)
     Call ID: toolu_01JzPqefao1gxwajHQ3Yh3JD
      Args:
        query: weather in nyc
    ================================= Tool Message =================================
    Name: tavily_search_results_json
    
    [{"url": "https://www.weatherapi.com/", "content": "{'location': {'name': 'New York', 'region': 'New York', 'country': 'United States of America', 'lat': 40.71, 'lon': -74.01, 'tz_id': 'America/New_York', 'localtime_epoch': 1718734479, 'localtime': '2024-06-18 14:14'}, 'current': {'last_updated_epoch': 1718733600, 'last_updated': '2024-06-18 14:00', 'temp_c': 29.4, 'temp_f': 84.9, 'is_day': 1, 'condition': {'text': 'Sunny', 'icon': '//cdn.weatherapi.com/weather/64x64/day/113.png', 'code': 1000}, 'wind_mph': 2.2, 'wind_kph': 3.6, 'wind_degree': 158, 'wind_dir': 'SSE', 'pressure_mb': 1025.0, 'pressure_in': 30.26, 'precip_mm': 0.0, 'precip_in': 0.0, 'humidity': 63, 'cloud': 0, 'feelslike_c': 31.3, 'feelslike_f': 88.3, 'windchill_c': 28.3, 'windchill_f': 82.9, 'heatindex_c': 29.6, 'heatindex_f': 85.3, 'dewpoint_c': 18.4, 'dewpoint_f': 65.2, 'vis_km': 16.0, 'vis_miles': 9.0, 'uv': 7.0, 'gust_mph': 16.5, 'gust_kph': 26.5}}"}]
    ================================== Ai Message ==================================
    
    The weather API results show that the current weather in New York City is sunny with a temperature of around 85°F (29°C). The wind is light at around 2-3 mph from the south-southeast. Overall it looks like a nice sunny summer day in NYC.


Verify that the original, rolled back run was deleted

=== "Python"

    ```python
    try:
        await client.runs.get(thread["thread_id"], rolled_back_run["run_id"])
    except httpx.HTTPStatusError as _:
        print("Original run was correctly deleted")
    ```

=== "Javascript"

    ```js
    try {
      await client.runs.get(thread["thread_id"], rolledBackRun["run_id"]);
    } catch (e) {
      console.log("Original run was correctly deleted");
    }
    ```

Output:

    Original run was correctly deleted


```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/same-thread.md
```md
# How to run multiple agents on the same thread

In LangGraph Platform, a thread is not explicitly associated with a particular agent.
This means that you can run multiple agents on the same thread, which allows a different agent to continue from an initial agent's progress.

In this example, we will create two agents and then call them both on the same thread.
You'll see that the second agent will respond using information from the [checkpoint](https://langchain-ai.github.io/langgraph/concepts/low_level/#checkpointer-state) generated in the thread by the first agent as context.

## Setup

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)

    openai_assistant = await client.assistants.create(
        graph_id="agent", config={"configurable": {"model_name": "openai"}}
    )

    # There should always be a default assistant with no configuration
    assistants = await client.assistants.search()
    default_assistant = [a for a in assistants if not a["config"]][0]
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    
    const openAIAssistant = await client.assistants.create(
      { graphId: "agent", config: {"configurable": {"model_name": "openai"}}}
    );

    const assistants = await client.assistants.search();
    const defaultAssistant = assistants.find(a => !a.config);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/assistants \
        --header 'Content-Type: application/json' \
        --data '{
            "graph_id": "agent",
            "config": { "configurable": { "model_name": "openai" } }
        }' && \
    curl --request POST \
        --url <DEPLOYMENT_URL>/assistants/search \
        --header 'Content-Type: application/json' \
        --data '{
            "limit": 10,
            "offset": 0
        }' | jq -c 'map(select(.config == null or .config == {})) | .[0]'
    ```

We can see that these agents are different:

=== "Python"

    ```python
    print(openai_assistant)
    ```

=== "Javascript"

    ```js
    console.log(openAIAssistant);
    ```

=== "CURL"

    ```bash
    curl --request GET \
        --url <DEPLOYMENT_URL>/assistants/<OPENAI_ASSISTANT_ID>
    ```

Output:

    {
        "assistant_id": "db87f39d-b2b1-4da8-ac65-cf81beb3c766",
        "graph_id": "agent",
        "created_at": "2024-08-30T21:18:51.850581+00:00",
        "updated_at": "2024-08-30T21:18:51.850581+00:00",
        "config": {
            "configurable": {
                "model_name": "openai"
            }
        },
        "metadata": {}
    }

=== "Python"

    ```python
    print(default_assistant)
    ```

=== "Javascript"

    ```js
    console.log(defaultAssistant);
    ```

=== "CURL"

    ```bash
    curl --request GET \
        --url <DEPLOYMENT_URL>/assistants/<DEFAULT_ASSISTANT_ID>
    ```

Output:

    {
        "assistant_id": "fe096781-5601-53d2-b2f6-0d3403f7e9ca",
        "graph_id": "agent",
        "created_at": "2024-08-08T22:45:24.562906+00:00",
        "updated_at": "2024-08-08T22:45:24.562906+00:00",
        "config": {},
        "metadata": {
            "created_by": "system"
        }
    }

## Run assistants on thread

### Run OpenAI assistant

We can now run the OpenAI assistant on the thread first.

=== "Python"

    ```python
    thread = await client.threads.create()
    input = {"messages": [{"role": "user", "content": "who made you?"}]}
    async for event in client.runs.stream(
        thread["thread_id"],
        openai_assistant["assistant_id"],
        input=input,
        stream_mode="updates",
    ):
        print(f"Receiving event of type: {event.event}")
        print(event.data)
        print("\n\n")
    ```

=== "Javascript"

    ```js
    const thread = await client.threads.create();
    let input =  {"messages": [{"role": "user", "content": "who made you?"}]}

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      openAIAssistant["assistant_id"],
      {
        input,
        streamMode: "updates"
      }
    );
    for await (const event of streamResponse) {
      console.log(`Receiving event of type: ${event.event}`);
      console.log(event.data);
      console.log("\n\n");
    }
    ```

=== "CURL"

    ```bash
    thread_id=$(curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}' | jq -r '.thread_id') && \
    curl --request POST \
        --url "<DEPLOYMENT_URL>/threads/${thread_id}/runs/stream" \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <OPENAI_ASSISTANT_ID>,
            "input": {
                "messages": [
                    {
                        "role": "user",
                        "content": "who made you?"
                    }
                ]
            },
            "stream_mode": [
                "updates"
            ]
        }' | \
        sed 's/\r$//' | \
        awk '
        /^event:/ {
            if (data_content != "") {
                print data_content "\n"
            }
            sub(/^event: /, "Receiving event of type: ", $0)
            printf "%s...\n", $0
            data_content = ""
        }
        /^data:/ {
            sub(/^data: /, "", $0)
            data_content = $0
        }
        END {
            if (data_content != "") {
                print data_content "\n\n"
            }
        }
    '
    ```

Output:

    Receiving event of type: metadata
    {'run_id': '1ef671c5-fb83-6e70-b698-44dba2d9213e'}


    Receiving event of type: updates
    {'agent': {'messages': [{'content': 'I was created by OpenAI, a research organization focused on developing and advancing artificial intelligence technology.', 'additional_kwargs': {}, 'response_metadata': {'finish_reason': 'stop', 'model_name': 'gpt-4o-2024-05-13', 'system_fingerprint': 'fp_157b3831f5'}, 'type': 'ai', 'name': None, 'id': 'run-f5735b86-b80d-4c71-8dc3-4782b5a9c7c8', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]}}

### Run default assistant

Now, we can run it on the default assistant and see that this second assistant is aware of the initial question, and can answer the question, "and you?":

=== "Python"

    ```python
    input = {"messages": [{"role": "user", "content": "and you?"}]}
    async for event in client.runs.stream(
        thread["thread_id"],
        default_assistant["assistant_id"],
        input=input,
        stream_mode="updates",
    ):
        print(f"Receiving event of type: {event.event}")
        print(event.data)
        print("\n\n")
    ```

=== "Javascript"

    ```js
    let input =  {"messages": [{"role": "user", "content": "and you?"}]}

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      defaultAssistant["assistant_id"],
      {
        input,
        streamMode: "updates"
      }
    );
    for await (const event of streamResponse) {
      console.log(`Receiving event of type: ${event.event}`);
      console.log(event.data);
      console.log("\n\n");
    }
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <DEFAULT_ASSISTANT_ID>,
            "input": {
                "messages": [
                    {
                        "role": "user",
                        "content": "and you?"
                    }
                ]
            },
            "stream_mode": [
                "updates"
            ]
        }' | \
        sed 's/\r$//' | \
        awk '
        /^event:/ {
            if (data_content != "") {
                print data_content "\n"
            }
            sub(/^event: /, "Receiving event of type: ", $0)
            printf "%s...\n", $0
            data_content = ""
        }
        /^data:/ {
            sub(/^data: /, "", $0)
            data_content = $0
        }
        END {
            if (data_content != "") {
                print data_content "\n\n"
            }
        }
    '
    ```

Output:

    Receiving event of type: metadata
    {'run_id': '1ef6722d-80b3-6fbb-9324-253796b1cd13'}


    Receiving event of type: updates
    {'agent': {'messages': [{'content': [{'text': 'I am an artificial intelligence created by Anthropic, not by OpenAI. I should not have stated that OpenAI created me, as that is incorrect. Anthropic is the company that developed and trained me using advanced language models and AI technology. I will be more careful about providing accurate information regarding my origins in the future.', 'type': 'text', 'index': 0}], 'additional_kwargs': {}, 'response_metadata': {'stop_reason': 'end_turn', 'stop_sequence': None}, 'type': 'ai', 'name': None, 'id': 'run-ebaacf62-9dd9-4165-9535-db432e4793ec', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': {'input_tokens': 302, 'output_tokens': 72, 'total_tokens': 374}}]}}




```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/stateless_runs.md
```md
# Stateless Runs

Most of the time, you provide a `thread_id` to your client when you run your graph in order to keep track of prior runs through the persistent state implemented in LangGraph Platform. However, if you don't need to persist the runs you don't need to use the built in persistent state and can create stateless runs.

## Setup

First, let's setup our client:

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # Using the graph deployed with the name "agent"
    assistant_id = "agent"
    # create thread
    thread = await client.threads.create()
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // Using the graph deployed with the name "agent"
    const assistantId = "agent";
    // create thread
    const thread = await client.threads.create();
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/assistants/search \
        --header 'Content-Type: application/json' \
        --data '{
            "limit": 10,
            "offset": 0
        }' | jq -c 'map(select(.config == null or .config == {})) | .[0].graph_id' && \
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
    ```

## Stateless streaming

We can stream the results of a stateless run in an almost identical fashion to how we stream from a run with the state attribute, but instead of passing a value to the `thread_id` parameter, we pass `None`:

=== "Python"

    ```python
    input = {
        "messages": [
            {"role": "user", "content": "Hello! My name is Bagatur and I am 26 years old."}
        ]
    }

    async for chunk in client.runs.stream(
        # Don't pass in a thread_id and the stream will be stateless
        None,
        assistant_id,
        input=input,
        stream_mode="updates",
    ):
        if chunk.data and "run_id" not in chunk.data:
            print(chunk.data)
    ```

=== "Javascript"

    ```js
    let input = {
      messages: [
        { role: "user", content: "Hello! My name is Bagatur and I am 26 years old." }
      ]
    };

    const streamResponse = client.runs.stream(
      // Don't pass in a thread_id and the stream will be stateless
      null,
      assistantId,
      {
        input,
        streamMode: "updates"
      }
    );
    for await (const chunk of streamResponse) {
      if (chunk.data && !("run_id" in chunk.data)) {
        console.log(chunk.data);
      }
    }
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/runs/stream \
        --header 'Content-Type: application/json' \
        --data "{
            \"assistant_id\": \"agent\",
            \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"Hello! My name is Bagatur and I am 26 years old.\"}]},
            \"stream_mode\": [
                \"updates\"
            ]
        }" | jq -c 'select(.data and (.data | has("run_id") | not)) | .data'
    ```

Output:

    {'agent': {'messages': [{'content': "Hello Bagatur! It's nice to meet you. Thank you for introducing yourself and sharing your age. Is there anything specific you'd like to know or discuss? I'm here to help with any questions or topics you're interested in.", 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-489ec573-1645-4ce2-a3b8-91b391d50a71', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]}}

## Waiting for stateless results

In addition to streaming, you can also wait for a stateless result by using the `.wait` function like follows:

=== "Python"

    ```python
    stateless_run_result = await client.runs.wait(
        None,
        assistant_id,
        input=input,
    )
    print(stateless_run_result)
    ```

=== "Javascript"

    ```js
    let statelessRunResult = await client.runs.wait(
      null,
      assistantId,
      { input: input }
    );
    console.log(statelessRunResult);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/runs/wait \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <ASSISTANT_IDD>,
        }'
    ```

Output:

    {
        'messages': [
            {
                'content': 'Hello! My name is Bagatur and I am 26 years old.',
                'additional_kwargs': {},
                'response_metadata': {},
                'type': 'human',
                'name': None,
                'id': '5e088543-62c2-43de-9d95-6086ad7f8b48',
                'example': False}
            ,
            {
                'content': "Hello Bagatur! It's nice to meet you. Thank you for introducing yourself and sharing your age. Is there anything specific you'd like to know or discuss? I'm here to help with any questions or topics you'd like to explore.",
                'additional_kwargs': {},
                'response_metadata': {},
                'type': 'ai',
                'name': None,
                'id': 'run-d6361e8d-4d4c-45bd-ba47-39520257f773',
                'example': False,
                'tool_calls': [],
                'invalid_tool_calls': [],
                'usage_metadata': None
            }
        ]
    }
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/streaming.md
```md
# Streaming API

[LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/) allows you to [stream outputs](../../concepts/streaming.md) from the LangGraph API server.

!!! note

    LangGraph SDK and LangGraph Server are a part of [LangGraph Platform](../../concepts/langgraph_platform.md).

## Basic usage

Basic usage example:

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>, api_key=<API_KEY>)

    # Using the graph deployed with the name "agent"
    assistant_id = "agent"

    # create a thread
    thread = await client.threads.create()
    thread_id = thread["thread_id"]

    # create a streaming run
    # highlight-next-line
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input=inputs,
        stream_mode="updates"
    ):
        print(chunk.data)
    ```

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL>, apiKey: <API_KEY> });

    // Using the graph deployed with the name "agent"
    const assistantID = "agent";

    // create a thread
    const thread = await client.threads.create();
    const threadID = thread["thread_id"];

    // create a streaming run
    // highlight-next-line
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input,
        streamMode: "updates"
      }
    );
    for await (const chunk of streamResponse) {
      console.log(chunk.data);
    }
    ```

=== "cURL"

    Create a thread:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads \
    --header 'Content-Type: application/json' \
    --data '{}'
    ```

    Create a streaming run:

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
    --header 'Content-Type: application/json' \
    --header 'x-api-key: <API_KEY>'
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": <inputs>,
      \"stream_mode\": \"updates\"
    }"
    ```

??? example "Extended example: streaming updates"

    This is an example graph you can run in the LangGraph API server.
    See [LangGraph Platform quickstart](../quick_start.md) for more details.

    ```python
    # graph.py
    from typing import TypedDict
    from langgraph.graph import StateGraph, START, END

    class State(TypedDict):
        topic: str
        joke: str

    def refine_topic(state: State):
        return {"topic": state["topic"] + " and cats"}

    def generate_joke(state: State):
        return {"joke": f"This is a joke about {state['topic']}"}

    graph = (
        StateGraph(State)
        .add_node(refine_topic)
        .add_node(generate_joke)
        .add_edge(START, "refine_topic")
        .add_edge("refine_topic", "generate_joke")
        .add_edge("generate_joke", END)
        .compile()
    )
    ```

    Once you have a running LangGraph API server, you can interact with it using
    [LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/)

    === "Python"

        ```python
        from langgraph_sdk import get_client
        client = get_client(url=<DEPLOYMENT_URL>)

        # Using the graph deployed with the name "agent"
        assistant_id = "agent"

        # create a thread
        thread = await client.threads.create()
        thread_id = thread["thread_id"]

        # create a streaming run
        # highlight-next-line
        async for chunk in client.runs.stream(  # (1)!
            thread_id,
            assistant_id,
            input={"topic": "ice cream"},
            # highlight-next-line
            stream_mode="updates"  # (2)!
        ):
            print(chunk.data)
        ```

        1. The `client.runs.stream()` method returns an iterator that yields streamed outputs.
        2. Set `stream_mode="updates"` to stream only the updates to the graph state after each node. Other stream modes are also available. See [supported stream modes](#supported-stream-modes) for details.

    === "JavaScript"

        ```js
        import { Client } from "@langchain/langgraph-sdk";
        const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

        // Using the graph deployed with the name "agent"
        const assistantID = "agent";

        // create a thread
        const thread = await client.threads.create();
        const threadID = thread["thread_id"];

        // create a streaming run
        // highlight-next-line
        const streamResponse = client.runs.stream(  // (1)!
          threadID,
          assistantID,
          {
            input: { topic: "ice cream" },
            // highlight-next-line
            streamMode: "updates"  // (2)!
          }
        );
        for await (const chunk of streamResponse) {
          console.log(chunk.data);
        }
        ```

        1. The `client.runs.stream()` method returns an iterator that yields streamed outputs.
        2. Set `streamMode: "updates"` to stream only the updates to the graph state after each node. Other stream modes are also available. See [supported stream modes](#supported-stream-modes) for details.

    === "cURL"

        Create a thread:

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
        ```

        Create a streaming run:

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"input\": {\"topic\": \"ice cream\"},
          \"stream_mode\": \"updates\"
        }"
        ```

    ```output
    {'run_id': '1f02c2b3-3cef-68de-b720-eec2a4a8e920', 'attempt': 1}
    {'refine_topic': {'topic': 'ice cream and cats'}}
    {'generate_joke': {'joke': 'This is a joke about ice cream and cats'}}
    ```


### Supported stream modes

| Mode                             | Description                                                                                                                                                                         | LangGraph Library Method                                                                                 |
|----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| [`values`](#stream-graph-state)  | Stream the full graph state after each [super-step](../../concepts/low_level.md#graphs).                                                                                            | `.stream()` / `.astream()` with [`stream_mode="values"`](../../how-tos/streaming.md#stream-graph-state)  |
| [`updates`](#stream-graph-state) | Streams the updates to the state after each step of the graph. If multiple updates are made in the same step (e.g., multiple nodes are run), those updates are streamed separately. | `.stream()` / `.astream()` with [`stream_mode="updates"`](../../how-tos/streaming.md#stream-graph-state) |
| [`messages-tuple`](#messages)    | Streams LLM tokens and metadata for the graph node where the LLM is invoked (useful for chat apps).                                                                                 | `.stream()` / `.astream()` with [`stream_mode="messages"`](../../how-tos/streaming.md#messages)          |
| [`debug`](#debug)                | Streams as much information as possible throughout the execution of the graph.                                                                                                      | `.stream()` / `.astream()` with [`stream_mode="debug"`](../../how-tos/streaming.md#stream-graph-state)   |
| [`custom`](#stream-custom-data)  | Streams custom data from inside your graph                                                                                                                                          | `.stream()` / `.astream()` with [`stream_mode="custom"`](../../how-tos/streaming.md#stream-custom-data)  |
| [`events`](#stream-events)       | Stream all events (including the state of the graph); mainly useful when migrating large LCEL apps.                                                                                 | `.astream_events()`                                                                                      |

### Stream multiple modes

You can pass a list as the `stream_mode` parameter to stream multiple modes at once.

The streamed outputs will be tuples of `(mode, chunk)` where `mode` is the name of the stream mode and `chunk` is the data streamed by that mode.

=== "Python"

    ```python
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input=inputs,
        stream_mode=["updates", "custom"]
    ):
        print(chunk)
    ```

=== "JavaScript"

    ```js
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input,
        streamMode: ["updates", "custom"]
      }
    );
    for await (const chunk of streamResponse) {
      console.log(chunk);
    }
    ```

=== "cURL"

    ```bash
    curl --request POST \
     --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
     --header 'Content-Type: application/json' \
     --data "{
       \"assistant_id\": \"agent\",
       \"input\": <inputs>,
       \"stream_mode\": [
         \"updates\"
         \"custom\"
       ]
     }"
    ```

## Stream graph state

Use the stream modes `updates` and `values` to stream the state of the graph as it executes.

* `updates` streams the **updates** to the state after each step of the graph.
* `values` streams the **full value** of the state after each step of the graph.

??? example "Example graph"

    ```python
    from typing import TypedDict
    from langgraph.graph import StateGraph, START, END

    class State(TypedDict):
      topic: str
      joke: str

    def refine_topic(state: State):
        return {"topic": state["topic"] + " and cats"}

    def generate_joke(state: State):
        return {"joke": f"This is a joke about {state['topic']}"}

    graph = (
      StateGraph(State)
      .add_node(refine_topic)
      .add_node(generate_joke)
      .add_edge(START, "refine_topic")
      .add_edge("refine_topic", "generate_joke")
      .add_edge("generate_joke", END)
      .compile()
    )
    ```

!!! note "Stateful runs"

    Examples below assume that you want to **persist the outputs** of a streaming run in the [checkpointer](../../concepts/persistence.md) DB and have created a thread. To create a thread:

    === "Python"

        ```python
        from langgraph_sdk import get_client
        client = get_client(url=<DEPLOYMENT_URL>)

        # Using the graph deployed with the name "agent"
        assistant_id = "agent"
        # create a thread
        thread = await client.threads.create()
        thread_id = thread["thread_id"]
        ```

    === "JavaScript"

        ```js
        import { Client } from "@langchain/langgraph-sdk";
        const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

        // Using the graph deployed with the name "agent"
        const assistantID = "agent";
        // create a thread
        const thread = await client.threads.create();
        const threadID = thread["thread_id"]
        ```

    === "cURL"

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
        ```

    If you don't need to persist the outputs of a run, you can pass `None` instead of `thread_id` when streaming.

=== "updates"

    Use this to stream only the **state updates** returned by the nodes after each step. The streamed outputs include the name of the node as well as the update.

    === "Python"

        ```python
        async for chunk in client.runs.stream(
            thread_id,
            assistant_id,
            input={"topic": "ice cream"},
            # highlight-next-line
            stream_mode="updates"
        ):
            print(chunk.data)
        ```

    === "JavaScript"

        ```js
        const streamResponse = client.runs.stream(
          threadID,
          assistantID,
          {
            input: { topic: "ice cream" },
            // highlight-next-line
            streamMode: "updates"
          }
        );
        for await (const chunk of streamResponse) {
          console.log(chunk.data);
        }
        ```

    === "cURL"

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"input\": {\"topic\": \"ice cream\"},
          \"stream_mode\": \"updates\"
        }"
        ```

===  "values"

    Use this to stream the **full state** of the graph after each step.

    === "Python"

        ```python
        async for chunk in client.runs.stream(
            thread_id,
            assistant_id,
            input={"topic": "ice cream"},
            # highlight-next-line
            stream_mode="values"
        ):
            print(chunk.data)
        ```

    === "JavaScript"

        ```js
        const streamResponse = client.runs.stream(
          threadID,
          assistantID,
          {
            input: { topic: "ice cream" },
            // highlight-next-line
            streamMode: "values"
          }
        );
        for await (const chunk of streamResponse) {
          console.log(chunk.data);
        }
        ```

    === "cURL"

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"input\": {\"topic\": \"ice cream\"},
          \"stream_mode\": \"values\"
        }"
        ```


## Subgraphs

To include outputs from [subgraphs](../../concepts/subgraphs.md) in the streamed outputs, you can set `subgraphs=True` in the `.stream()` method of the parent graph. This will stream outputs from both the parent graph and any subgraphs.

```python
for chunk in client.runs.stream(
    thread_id,
    assistant_id,
    input={"foo": "foo"},
    # highlight-next-line
    stream_subgraphs=True, # (1)!
    stream_mode="updates",
):
    print(chunk)
```

1. Set `stream_subgraphs=True` to stream outputs from subgraphs.

??? example "Extended example: streaming from subgraphs"

    This is an example graph you can run in the LangGraph API server.
    See [LangGraph Platform quickstart](../quick_start.md) for more details.

    ```python
    # graph.py
    from langgraph.graph import START, StateGraph
    from typing import TypedDict

    # Define subgraph
    class SubgraphState(TypedDict):
        foo: str  # note that this key is shared with the parent graph state
        bar: str

    def subgraph_node_1(state: SubgraphState):
        return {"bar": "bar"}

    def subgraph_node_2(state: SubgraphState):
        return {"foo": state["foo"] + state["bar"]}

    subgraph_builder = StateGraph(SubgraphState)
    subgraph_builder.add_node(subgraph_node_1)
    subgraph_builder.add_node(subgraph_node_2)
    subgraph_builder.add_edge(START, "subgraph_node_1")
    subgraph_builder.add_edge("subgraph_node_1", "subgraph_node_2")
    subgraph = subgraph_builder.compile()

    # Define parent graph
    class ParentState(TypedDict):
        foo: str

    def node_1(state: ParentState):
        return {"foo": "hi! " + state["foo"]}

    builder = StateGraph(ParentState)
    builder.add_node("node_1", node_1)
    builder.add_node("node_2", subgraph)
    builder.add_edge(START, "node_1")
    builder.add_edge("node_1", "node_2")
    graph = builder.compile()
    ```

    Once you have a running LangGraph API server, you can interact with it using
    [LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/)

    === "Python"

        ```python
        from langgraph_sdk import get_client
        client = get_client(url=<DEPLOYMENT_URL>)

        # Using the graph deployed with the name "agent"
        assistant_id = "agent"

        # create a thread
        thread = await client.threads.create()
        thread_id = thread["thread_id"]
    
        async for chunk in client.runs.stream(
            thread_id,
            assistant_id,
            input={"foo": "foo"},
            # highlight-next-line
            stream_subgraphs=True, # (1)!
            stream_mode="updates",
        ):
            print(chunk)
        ```
        
        1. Set `stream_subgraphs=True` to stream outputs from subgraphs.

    === "JavaScript"

        ```js
        import { Client } from "@langchain/langgraph-sdk";
        const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

        // Using the graph deployed with the name "agent"
        const assistantID = "agent";

        // create a thread
        const thread = await client.threads.create();
        const threadID = thread["thread_id"];

        // create a streaming run
        const streamResponse = client.runs.stream(
          threadID,
          assistantID,
          {
            input: { foo: "foo" },
            // highlight-next-line
            streamSubgraphs: true,  // (1)!
            streamMode: "updates"
          }
        );
        for await (const chunk of streamResponse) {
          console.log(chunk);
        }
        ```

        1. Set `streamSubgraphs: true` to stream outputs from subgraphs.

    === "cURL"

        Create a thread:

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
        ```

        Create a streaming run:

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"input\": {\"foo\": \"foo\"},
          \"stream_subgraphs\": true,
          \"stream_mode\": [
            \"updates\"
          ]
        }"
        ```

    **Note** that we are receiving not just the node updates, but we also the namespaces which tell us what graph (or subgraph) we are streaming from.

## Debugging {#debug}

Use the `debug` streaming mode to stream as much information as possible throughout the execution of the graph. The streamed outputs include the name of the node as well as the full state.

=== "Python"

    ```python
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input={"topic": "ice cream"},
        # highlight-next-line
        stream_mode="debug"
    ):
        print(chunk.data)
    ```

=== "JavaScript"

    ```js
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input: { topic: "ice cream" },
        // highlight-next-line
        streamMode: "debug"
      }
    );
    for await (const chunk of streamResponse) {
      console.log(chunk.data);
    }
    ```

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"topic\": \"ice cream\"},
      \"stream_mode\": \"debug\"
    }"
    ```

## LLM tokens {#messages}

Use the `messages-tuple` streaming mode to stream Large Language Model (LLM) outputs **token by token** from any part of your graph, including nodes, tools, subgraphs, or tasks.

The streamed output from [`messages-tuple` mode](#supported-stream-modes) is a tuple `(message_chunk, metadata)` where:

- `message_chunk`: the token or message segment from the LLM.
- `metadata`: a dictionary containing details about the graph node and LLM invocation.
 
??? example "Example graph"

    ```python
    from dataclasses import dataclass

    from langchain.chat_models import init_chat_model
    from langgraph.graph import StateGraph, START

    @dataclass
    class MyState:
        topic: str
        joke: str = ""

    llm = init_chat_model(model="openai:gpt-4o-mini")

    def call_model(state: MyState):
        """Call the LLM to generate a joke about a topic"""
        # highlight-next-line
        llm_response = llm.invoke( # (1)!
            [
                {"role": "user", "content": f"Generate a joke about {state.topic}"}
            ]
        )
        return {"joke": llm_response.content}

    graph = (
        StateGraph(MyState)
        .add_node(call_model)
        .add_edge(START, "call_model")
        .compile()
    )
    ```

    1. Note that the message events are emitted even when the LLM is run using `.invoke` rather than `.stream`.

=== "Python"

    ```python
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input={"topic": "ice cream"},
        # highlight-next-line
        stream_mode="messages-tuple",
    ):
        if chunk.event != "messages":
            continue

        message_chunk, metadata = chunk.data  # (1)!
        if message_chunk["content"]:
            print(message_chunk["content"], end="|", flush=True)
    ```

    1. The "messages-tuple" stream mode returns an iterator of tuples `(message_chunk, metadata)` where `message_chunk` is the token streamed by the LLM and `metadata` is a dictionary with information about the graph node where the LLM was called and other information.

=== "JavaScript"

    ```js
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input: { topic: "ice cream" },
        // highlight-next-line
        streamMode: "messages-tuple"
      }
    );
    for await (const chunk of streamResponse) {
      if (chunk.event !== "messages") {
        continue;
      }
      console.log(chunk.data[0]["content"]);  // (1)!
    }
    ```

    1. The "messages-tuple" stream mode returns an iterator of tuples `(message_chunk, metadata)` where `message_chunk` is the token streamed by the LLM and `metadata` is a dictionary with information about the graph node where the LLM was called and other information.

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"topic\": \"ice cream\"},
      \"stream_mode\": \"messages-tuple\"
    }"
    ```

### Filter LLM tokens

* To filter the streamed tokens by LLM invocation, you can [associate `tags` with LLM invocations](../../how-tos/streaming.md#filter-by-llm-invocation).
* To stream tokens only from specific nodes, use `stream_mode="messages"` and [filter the outputs by the `langgraph_node` field](../../how-tos/streaming.md#filter-by-node) in the streamed metadata.

## Stream custom data

To send **custom user-defined data**:

=== "Python"

    ```python
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input={"query": "example"},
        # highlight-next-line
        stream_mode="custom"
    ):
        print(chunk.data)
    ```

=== "JavaScript"

    ```js
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input: { query: "example" },
        // highlight-next-line
        streamMode: "custom"
      }
    );
    for await (const chunk of streamResponse) {
      console.log(chunk.data);
    }
    ```

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"query\": \"example\"},
      \"stream_mode\": \"custom\"
    }"
    ```

## Stream events

To stream all events, including the state of the graph:

=== "Python"

    ```python
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input={"topic": "ice cream"},
        # highlight-next-line
        stream_mode="events"
    ):
        print(chunk.data)
    ```

=== "JavaScript"

    ```js
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input: { topic: "ice cream" },
        // highlight-next-line
        streamMode: "events"
      }
    );
    for await (const chunk of streamResponse) {
      console.log(chunk.data);
    }
    ```

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"topic\": \"ice cream\"},
      \"stream_mode\": \"events\"
    }"
    ```

## Stateless runs

If you don't want to **persist the outputs** of a streaming run in the [checkpointer](../../concepts/persistence.md) DB, you can create a stateless run without creating a thread:

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>, api_key=<API_KEY>)

    async for chunk in client.runs.stream(
        # highlight-next-line
        None,  # (1)!
        assistant_id,
        input=inputs,
        stream_mode="updates"
    ):
        print(chunk.data)
    ```

    1. We are passing `None` instead of a `thread_id` UUID.

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL>, apiKey: <API_KEY> });

    // create a streaming run
    // highlight-next-line
    const streamResponse = client.runs.stream(
      // highlight-next-line
      null,  // (1)!
      assistantID,
      {
        input,
        streamMode: "updates"
      }
    );
    for await (const chunk of streamResponse) {
      console.log(chunk.data);
    }
    ```

    1. We are passing `None` instead of a `thread_id` UUID.

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/runs/stream \
    --header 'Content-Type: application/json' \
    --header 'x-api-key: <API_KEY>'
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": <inputs>,
      \"stream_mode\": \"updates\"
    }"
    ```

## Join and stream

LangGraph Platform allows you to join an active [background run](../how-tos/background_run.md) and stream outputs from it. To do so, you can use [LangGraph SDK's](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/) `client.runs.join_stream` method:

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>, api_key=<API_KEY>)

    # highlight-next-line
    async for chunk in client.runs.join_stream(
        thread_id,
        # highlight-next-line
        run_id,  # (1)!
    ):
        print(chunk)
    ```

    1. This is the `run_id` of an existing run you want to join.


=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL>, apiKey: <API_KEY> });

    // highlight-next-line
    const streamResponse = client.runs.joinStream(
      threadID,
      // highlight-next-line
      runId  // (1)!
    );
    for await (const chunk of streamResponse) {
      console.log(chunk);
    }
    ```

    1. This is the `run_id` of an existing run you want to join.

=== "cURL"

    ```bash
    curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/<RUN_ID>/stream \
    --header 'Content-Type: application/json' \
    --header 'x-api-key: <API_KEY>'
    ```

!!! warning "Outputs not buffered"

    When you use `.join_stream`, output is not buffered, so any output produced before joining will not be received.

## API Reference

For API usage and implementation, refer to the [API reference](../reference/api/api_ref.html#tag/thread-runs/POST/threads/{thread_id}/runs/stream). 

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/threads_studio.md
```md
# Manage threads

Studio allows you to view [threads](../../concepts/persistence.md#threads) from the server and edit their state.

## View threads

### Graph mode

1. In the top of the right-hand pane, select the dropdown menu to view existing threads.
1. Select the desired thread, and the thread history will populate in the right-hand side of the page.
1. To create a new thread, click `+ New Thread` and [submit a run](../how-tos/invoke_studio.md#graph-mode).

To view more granular information in the thread, drag the slider at the top of the page to the right. To view less information, drag the slider to the left. Additionally, collapse or expand individual turns, nodes, and keys of the state.

Switch between `Pretty` and `JSON` mode for different rendering formats.

### Chat mode

1. View all threads in the right-hand pane of the page.
2. Select the desired thread and the thread history will populate in the center panel.
3. To create a new thread, click the plus button and [submit a run](../how-tos/invoke_studio.md#chat-mode).

## Edit thread history

### Graph mode

To edit the state of the thread, select "edit node state" next to the desired node. Edit the node's output as desired and click "fork" to confirm. This will create a new forked run from the checkpoint of the selected node.

If you instead want to re-run the thread from a given checkpoint without editing the state, click the "Re-run from here". This will again create a new forked run from the selected checkpoint. This is useful for re-running with changes that are not specific to the state, such as the selected assistant.

### Chat mode

To edit a human message in the thread, click the edit button below the human message. Edit the message as desired and submit. This will create a new fork of the conversation history. To re-generate an AI message, click the retry icon below the AI message.

## Learn more

For more information about time travel, [see here](../../concepts/time-travel.md).

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/use_stream_react.md
```md
# How to integrate LangGraph into your React application

!!! info "Prerequisites"

    - [LangGraph Platform](../../concepts/langgraph_platform.md)
    - [LangGraph Server](../../concepts/langgraph_server.md)

The `useStream()` React hook provides a seamless way to integrate LangGraph into your React applications. It handles all the complexities of streaming, state management, and branching logic, letting you focus on building great chat experiences.

Key features:

- Messages streaming: Handle a stream of message chunks to form a complete message
- Automatic state management for messages, interrupts, loading states, and errors
- Conversation branching: Create alternate conversation paths from any point in the chat history
- UI-agnostic design: bring your own components and styling

Let's explore how to use `useStream()` in your React application.

The `useStream()` provides a solid foundation for creating bespoke chat experiences. For pre-built chat components and interfaces, we also recommend checking out [CopilotKit](https://docs.copilotkit.ai/coagents/quickstart/langgraph) and [assistant-ui](https://www.assistant-ui.com/docs/runtimes/langgraph).

## Installation

```bash
npm install @langchain/langgraph-sdk @langchain/core
```

## Example

```tsx
"use client";

import { useStream } from "@langchain/langgraph-sdk/react";
import type { Message } from "@langchain/langgraph-sdk";

export default function App() {
  const thread = useStream<{ messages: Message[] }>({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    messagesKey: "messages",
  });

  return (
    <div>
      <div>
        {thread.messages.map((message) => (
          <div key={message.id}>{message.content as string}</div>
        ))}
      </div>

      <form
        onSubmit={(e) => {
          e.preventDefault();

          const form = e.target as HTMLFormElement;
          const message = new FormData(form).get("message") as string;

          form.reset();
          thread.submit({ messages: [{ type: "human", content: message }] });
        }}
      >
        <input type="text" name="message" />

        {thread.isLoading ? (
          <button key="stop" type="button" onClick={() => thread.stop()}>
            Stop
          </button>
        ) : (
          <button keytype="submit">Send</button>
        )}
      </form>
    </div>
  );
}
```

## Customizing Your UI

The `useStream()` hook takes care of all the complex state management behind the scenes, providing you with simple interfaces to build your UI. Here's what you get out of the box:

- Thread state management
- Loading and error states
- Interrupts
- Message handling and updates
- Branching support

Here are some examples on how to use these features effectively:

### Loading States

The `isLoading` property tells you when a stream is active, enabling you to:

- Show a loading indicator
- Disable input fields during processing
- Display a cancel button

```tsx
export default function App() {
  const { isLoading, stop } = useStream<{ messages: Message[] }>({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    messagesKey: "messages",
  });

  return (
    <form>
      {isLoading && (
        <button key="stop" type="button" onClick={() => stop()}>
          Stop
        </button>
      )}
    </form>
  );
}
```

### Resume a stream after page refresh

The `useStream()` hook can automatically resume an ongoing run upon mounting by setting `reconnectOnMount: true`. This is useful for continuing a stream after a page refresh, ensuring no messages and events generated during the downtime are lost.

```tsx
const thread = useStream<{ messages: Message[] }>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  reconnectOnMount: true,
});
```

By default the ID of the created run is stored in `window.sessionStorage`, which can be swapped by passing a custom storage in `reconnectOnMount` instead. The storage is used to persist the in-flight run ID for a thread (under `lg:stream:${threadId}` key).

```tsx
const thread = useStream<{ messages: Message[] }>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  reconnectOnMount: () => window.localStorage,
});
```

You can also manually manage the resuming process by using the run callbacks to persist the run metadata and the `joinStream` function to resume the stream. Make sure to pass `streamResumable: true` when creating the run; otherwise some events might be lost.

```tsx
import type { Message } from "@langchain/langgraph-sdk";
import { useStream } from "@langchain/langgraph-sdk/react";
import { useCallback, useState, useEffect, useRef } from "react";

export default function App() {
  const [threadId, onThreadId] = useSearchParam("threadId");

  const thread = useStream<{ messages: Message[] }>({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",

    threadId,
    onThreadId,

    onCreated: (run) => {
      window.sessionStorage.setItem(`resume:${run.thread_id}`, run.run_id);
    },
    onFinish: (_, run) => {
      window.sessionStorage.removeItem(`resume:${run?.thread_id}`);
    },
  });

  // Ensure that we only join the stream once per thread.
  const joinedThreadId = useRef<string | null>(null);
  useEffect(() => {
    if (!threadId) return;

    const resume = window.sessionStorage.getItem(`resume:${threadId}`);
    if (resume && joinedThreadId.current !== threadId) {
      thread.joinStream(resume);
      joinedThreadId.current = threadId;
    }
  }, [threadId]);

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        const form = e.target as HTMLFormElement;
        const message = new FormData(form).get("message") as string;
        thread.submit(
          { messages: [{ type: "human", content: message }] },
          { streamResumable: true }
        );
      }}
    >
      <div>
        {thread.messages.map((message) => (
          <div key={message.id}>{message.content as string}</div>
        ))}
      </div>
      <input type="text" name="message" />
      <button type="submit">Send</button>
    </form>
  );
}

// Utility method to retrieve and persist data in URL as search param
function useSearchParam(key: string) {
  const [value, setValue] = useState<string | null>(() => {
    const params = new URLSearchParams(window.location.search);
    return params.get(key) ?? null;
  });

  const update = useCallback(
    (value: string | null) => {
      setValue(value);

      const url = new URL(window.location.href);
      if (value == null) {
        url.searchParams.delete(key);
      } else {
        url.searchParams.set(key, value);
      }

      window.history.pushState({}, "", url.toString());
    },
    [key]
  );

  return [value, update] as const;
}
```

### Thread Management

Keep track of conversations with built-in thread management. You can access the current thread ID and get notified when new threads are created:

```tsx
const [threadId, setThreadId] = useState<string | null>(null);

const thread = useStream<{ messages: Message[] }>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",

  threadId: threadId,
  onThreadId: setThreadId,
});
```

We recommend storing the `threadId` in your URL's query parameters to let users resume conversations after page refreshes.

### Messages Handling

The `useStream()` hook will keep track of the message chunks received from the server and concatenate them together to form a complete message. The completed message chunks can be retrieved via the `messages` property.

By default, the `messagesKey` is set to `messages`, where it will append the new messages chunks to `values["messages"]`. If you store messages in a different key, you can change the value of `messagesKey`.

```tsx
import type { Message } from "@langchain/langgraph-sdk";
import { useStream } from "@langchain/langgraph-sdk/react";

export default function HomePage() {
  const thread = useStream<{ messages: Message[] }>({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    messagesKey: "messages",
  });

  return (
    <div>
      {thread.messages.map((message) => (
        <div key={message.id}>{message.content as string}</div>
      ))}
    </div>
  );
}
```

Under the hood, the `useStream()` hook will use the `streamMode: "messages-tuple"` to receive a stream of messages (i.e. individual LLM tokens) from any LangChain chat model invocations inside your graph nodes. Learn more about messages streaming in the [streaming](../how-tos/streaming.md#messages) guide.

### Interrupts

The `useStream()` hook exposes the `interrupt` property, which will be filled with the last interrupt from the thread. You can use interrupts to:

- Render a confirmation UI before executing a node
- Wait for human input, allowing agent to ask the user with clarifying questions

Learn more about interrupts in the [How to handle interrupts](../../how-tos/human_in_the_loop/wait-user-input.ipynb) guide.

```tsx
const thread = useStream<{ messages: Message[] }, { InterruptType: string }>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});

if (thread.interrupt) {
  return (
    <div>
      Interrupted! {thread.interrupt.value}
      <button
        type="button"
        onClick={() => {
          // `resume` can be any value that the agent accepts
          thread.submit(undefined, { command: { resume: true } });
        }}
      >
        Resume
      </button>
    </div>
  );
}
```

### Branching

For each message, you can use `getMessagesMetadata()` to get the first checkpoint from which the message has been first seen. You can then create a new run from the checkpoint preceding the first seen checkpoint to create a new branch in a thread.

A branch can be created in following ways:

1. Edit a previous user message.
2. Request a regeneration of a previous assistant message.

```tsx
"use client";

import type { Message } from "@langchain/langgraph-sdk";
import { useStream } from "@langchain/langgraph-sdk/react";
import { useState } from "react";

function BranchSwitcher({
  branch,
  branchOptions,
  onSelect,
}: {
  branch: string | undefined;
  branchOptions: string[] | undefined;
  onSelect: (branch: string) => void;
}) {
  if (!branchOptions || !branch) return null;
  const index = branchOptions.indexOf(branch);

  return (
    <div className="flex items-center gap-2">
      <button
        type="button"
        onClick={() => {
          const prevBranch = branchOptions[index - 1];
          if (!prevBranch) return;
          onSelect(prevBranch);
        }}
      >
        Prev
      </button>
      <span>
        {index + 1} / {branchOptions.length}
      </span>
      <button
        type="button"
        onClick={() => {
          const nextBranch = branchOptions[index + 1];
          if (!nextBranch) return;
          onSelect(nextBranch);
        }}
      >
        Next
      </button>
    </div>
  );
}

function EditMessage({
  message,
  onEdit,
}: {
  message: Message;
  onEdit: (message: Message) => void;
}) {
  const [editing, setEditing] = useState(false);

  if (!editing) {
    return (
      <button type="button" onClick={() => setEditing(true)}>
        Edit
      </button>
    );
  }

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        const form = e.target as HTMLFormElement;
        const content = new FormData(form).get("content") as string;

        form.reset();
        onEdit({ type: "human", content });
        setEditing(false);
      }}
    >
      <input name="content" defaultValue={message.content as string} />
      <button type="submit">Save</button>
    </form>
  );
}

export default function App() {
  const thread = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    messagesKey: "messages",
  });

  return (
    <div>
      <div>
        {thread.messages.map((message) => {
          const meta = thread.getMessagesMetadata(message);
          const parentCheckpoint = meta?.firstSeenState?.parent_checkpoint;

          return (
            <div key={message.id}>
              <div>{message.content as string}</div>

              {message.type === "human" && (
                <EditMessage
                  message={message}
                  onEdit={(message) =>
                    thread.submit(
                      { messages: [message] },
                      { checkpoint: parentCheckpoint }
                    )
                  }
                />
              )}

              {message.type === "ai" && (
                <button
                  type="button"
                  onClick={() =>
                    thread.submit(undefined, { checkpoint: parentCheckpoint })
                  }
                >
                  <span>Regenerate</span>
                </button>
              )}

              <BranchSwitcher
                branch={meta?.branch}
                branchOptions={meta?.branchOptions}
                onSelect={(branch) => thread.setBranch(branch)}
              />
            </div>
          );
        })}
      </div>

      <form
        onSubmit={(e) => {
          e.preventDefault();

          const form = e.target as HTMLFormElement;
          const message = new FormData(form).get("message") as string;

          form.reset();
          thread.submit({ messages: [message] });
        }}
      >
        <input type="text" name="message" />

        {thread.isLoading ? (
          <button key="stop" type="button" onClick={() => thread.stop()}>
            Stop
          </button>
        ) : (
          <button key="submit" type="submit">
            Send
          </button>
        )}
      </form>
    </div>
  );
}
```

For advanced use cases you can use the `experimental_branchTree` property to get the tree representation of the thread, which can be used to render branching controls for non-message based graphs.

### Optimistic Updates

You can optimistically update the client state before performing a network request to the agent, allowing you to provide immediate feedback to the user, such as showing the user message immediately before the agent has seen the request.

```tsx
const stream = useStream({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});

const handleSubmit = (text: string) => {
  const newMessage = { type: "human" as const, content: text };

  stream.submit(
    { messages: [newMessage] },
    {
      optimisticValues(prev) {
        const prevMessages = prev.messages ?? [];
        const newMessages = [...prevMessages, newMessage];
        return { ...prev, messages: newMessages };
      },
    }
  );
};
```

### Cached Thread Display

Use the `initialValues` option to display cached thread data immediately while the history is being loaded from the server. This improves user experience by showing cached data instantly when navigating to existing threads.

```tsx
import { useStream } from "@langchain/langgraph-sdk/react";

const CachedThreadExample = ({ threadId, cachedThreadData }) => {
  const stream = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    threadId,
    // Show cached data immediately while history loads
    initialValues: cachedThreadData?.values,
    messagesKey: "messages",
  });

  return (
    <div>
      {stream.messages.map((message) => (
        <div key={message.id}>{message.content as string}</div>
      ))}
    </div>
  );
};
```

### Optimistic Thread Creation

Use the `threadId` option in `submit` function to enable optimistic UI patterns where you need to know the thread ID before the thread is actually created.

```tsx
import { useState } from "react";
import { useStream } from "@langchain/langgraph-sdk/react";

const OptimisticThreadExample = () => {
  const [threadId, setThreadId] = useState<string | null>(null);
  const [optimisticThreadId] = useState(() => crypto.randomUUID());

  const stream = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    threadId,
    onThreadId: setThreadId, // (3) Updated after thread has been created.
    messagesKey: "messages",
  });

  const handleSubmit = (text: string) => {
    // (1) Perform a soft navigation to /threads/${optimisticThreadId}
    // without waiting for thread creation.
    window.history.pushState({}, "", `/threads/${optimisticThreadId}`);

    // (2) Submit message to create thread with the predetermined ID.
    stream.submit(
      { messages: [{ type: "human", content: text }] },
      { threadId: optimisticThreadId }
    );
  };

  return (
    <div>
      <p>Thread ID: {threadId ?? optimisticThreadId}</p>
      {/* Rest of component */}
    </div>
  );
};
```

### TypeScript

The `useStream()` hook is friendly for apps written in TypeScript and you can specify types for the state to get better type safety and IDE support.

```tsx
// Define your types
type State = {
  messages: Message[];
  context?: Record<string, unknown>;
};

// Use them with the hook
const thread = useStream<State>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});
```

You can also optionally specify types for different scenarios, such as:

- `ConfigurableType`: Type for the `config.configurable` property (default: `Record<string, unknown>`)
- `InterruptType`: Type for the interrupt value - i.e. contents of `interrupt(...)` function (default: `unknown`)
- `CustomEventType`: Type for the custom events (default: `unknown`)
- `UpdateType`: Type for the submit function (default: `Partial<State>`)

```tsx
const thread = useStream<
  State,
  {
    UpdateType: {
      messages: Message[] | Message;
      context?: Record<string, unknown>;
    };
    InterruptType: string;
    CustomEventType: {
      type: "progress" | "debug";
      payload: unknown;
    };
    ConfigurableType: {
      model: string;
    };
  }
>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});
```

If you're using LangGraph.js, you can also reuse your graph's annotation types. However, make sure to only import the types of the annotation schema in order to avoid importing the entire LangGraph.js runtime (i.e. via `import type { ... }` directive).

```tsx
import {
  Annotation,
  MessagesAnnotation,
  type StateType,
  type UpdateType,
} from "@langchain/langgraph/web";

const AgentState = Annotation.Root({
  ...MessagesAnnotation.spec,
  context: Annotation<string>(),
});

const thread = useStream<
  StateType<typeof AgentState.spec>,
  { UpdateType: UpdateType<typeof AgentState.spec> }
>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});
```

## Event Handling

The `useStream()` hook provides several callback options to help you respond to different events:

- `onError`: Called when an error occurs.
- `onFinish`: Called when the stream is finished.
- `onUpdateEvent`: Called when an update event is received.
- `onCustomEvent`: Called when a custom event is received. See the [streaming](../../how-tos/streaming.md#stream-custom-data) guide to learn how to stream custom events.
- `onMetadataEvent`: Called when a metadata event is received, which contains the Run ID and Thread ID.

## Learn More

- [JS/TS SDK Reference](../reference/sdk/js_ts_sdk_ref.md)

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/use_threads.md
```md
# Use threads

In this guide, we will show how to create, view, and inspect [threads](../../concepts/persistence.md#threads).

## Create a thread

To run your graph and the state persisted, you must first create a thread.

### Empty thread

To create a new thread, use the [LangGraph SDK](../../concepts/sdk.md) `create` method. See the [Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.ThreadsClient.create) and [JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#create_3) SDK reference docs for more information.

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    thread = await client.threads.create()

    print(thread)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    const thread = await client.threads.create();

    console.log(thread);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
    ```

Output:

    {
      "thread_id": "123e4567-e89b-12d3-a456-426614174000",
      "created_at": "2025-05-12T14:04:08.268Z",
      "updated_at": "2025-05-12T14:04:08.268Z",
      "metadata": {},
      "status": "idle",
      "values": {}
    }

### Copy thread

Alternatively, if you already have a thread in your application whose state you wish to copy, you can use the `copy` method. This will create an independent thread whose history is identical to the original thread at the time of the operation. See the [Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.ThreadsClient.copy) and [JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#copy) SDK reference docs for more information.

=== "Python"

    ```python
    copied_thread = await client.threads.copy(<THREAD_ID>)
    ```

=== "Javascript"

    ```js
    const copiedThread = await client.threads.copy(<THREAD_ID>);
    ```

=== "CURL"

    ```bash
    curl --request POST --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/copy \
    --header 'Content-Type: application/json'
    ```

### Prepopulated State

Finally, you can create a thread with an arbitrary pre-defined state by providing a list of `supersteps` into the `create` method. The `supersteps` describe a list of a sequence of state updates. For example:

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    thread = await client.threads.create(
      graph_id="agent",
      supersteps=[
        {
          updates: [
            {
              values: {},
              as_node: '__input__',
            },
          ],
        },
        {
          updates: [
            {
              values: {
                messages: [
                  {
                    type: 'human',
                    content: 'hello',
                  },
                ],
              },
              as_node: '__start__',
            },
          ],
        },
        {
          updates: [
            {
              values: {
                messages: [
                  {
                    content: 'Hello! How can I assist you today?',
                    type: 'ai',
                  },
                ],
              },
              as_node: 'call_model',
            },
          ],
        },
      ])

    print(thread)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    const thread = await client.threads.create({
        graphId: 'agent',
        supersteps: [
        {
          updates: [
            {
              values: {},
              asNode: '__input__',
            },
          ],
        },
        {
          updates: [
            {
              values: {
                messages: [
                  {
                    type: 'human',
                    content: 'hello',
                  },
                ],
              },
              asNode: '__start__',
            },
          ],
        },
        {
          updates: [
            {
              values: {
                messages: [
                  {
                    content: 'Hello! How can I assist you today?',
                    type: 'ai',
                  },
                ],
              },
              asNode: 'call_model',
            },
          ],
        },
      ],
    });

    console.log(thread);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{"metadata":{"graph_id":"agent"},"supersteps":[{"updates":[{"values":{},"as_node":"__input__"}]},{"updates":[{"values":{"messages":[{"type":"human","content":"hello"}]},"as_node":"__start__"}]},{"updates":[{"values":{"messages":[{"content":"Hello\u0021 How can I assist you today?","type":"ai"}]},"as_node":"call_model"}]}]}'
    ```

Output:

    {
        "thread_id": "f15d70a1-27d4-4793-a897-de5609920b7d",
        "created_at": "2025-05-12T15:37:08.935038+00:00",
        "updated_at": "2025-05-12T15:37:08.935046+00:00",
        "metadata": {"graph_id": "agent"},
        "status": "idle",
        "config": {},
        "values": {
            "messages": [
                {
                    "content": "hello",
                    "additional_kwargs": {},
                    "response_metadata": {},
                    "type": "human",
                    "name": null,
                    "id": "8701f3be-959c-4b7c-852f-c2160699b4ab",
                    "example": false
                },
                {
                    "content": "Hello! How can I assist you today?",
                    "additional_kwargs": {},
                    "response_metadata": {},
                    "type": "ai",
                    "name": null,
                    "id": "4d8ea561-7ca1-409a-99f7-6b67af3e1aa3",
                    "example": false,
                    "tool_calls": [],
                    "invalid_tool_calls": [],
                    "usage_metadata": null
                }
            ]
        }
    }

## List threads

### LangGraph SDK

To list threads, use the [LangGraph SDK](../../concepts/sdk.md) `search` method. This will list the threads in the application that match the provided filters. See the [Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.ThreadsClient.search) and [JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#search_2) SDK reference docs for more information.

#### Filter by thread status

Use the `status` field to filter threads based on their status. Supported values are `idle`, `busy`, `interrupted`, and `error`. See [here](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/?h=thread+status#langgraph_sdk.auth.types.ThreadStatus) for information on each status. For example, to view `idle` threads:

=== "Python"

    ```python
    print(await client.threads.search(status="idle",limit=1))
    ```

=== "Javascript"

    ```js
    console.log(await client.threads.search({ status: "idle", limit: 1 }));
    ```

=== "CURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/search \
    --header 'Content-Type: application/json' \
    --data '{"status": "idle", "limit": 1}'
    ```

Output:

    [
      {
        'thread_id': 'cacf79bb-4248-4d01-aabc-938dbd60ed2c',
        'created_at': '2024-08-14T17:36:38.921660+00:00',
        'updated_at': '2024-08-14T17:36:38.921660+00:00',
        'metadata': {'graph_id': 'agent'},
        'status': 'idle',
        'config': {'configurable': {}}
      }
    ]

#### Filter by metadata

The `search` method allows you to filter on metadata:

=== "Python"

    ```python
    print((await client.threads.search(metadata={"graph_id":"agent"},limit=1)))
    ```

=== "Javascript"

    ```js
    console.log((await client.threads.search({ metadata: { "graph_id": "agent" }, limit: 1 })));
    ```

=== "CURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/search \
    --header 'Content-Type: application/json' \
    --data '{"metadata": {"graph_id":"agent"}, "limit": 1}'
    ```

Output:

    [
      {
        'thread_id': 'cacf79bb-4248-4d01-aabc-938dbd60ed2c',
        'created_at': '2024-08-14T17:36:38.921660+00:00',
        'updated_at': '2024-08-14T17:36:38.921660+00:00',
        'metadata': {'graph_id': 'agent'},
        'status': 'idle',
        'config': {'configurable': {}}
      }
    ]

#### Sorting

The SDK also supports sorting threads by `thread_id`, `status`, `created_at`, and `updated_at` using the `sort_by` and `sort_order` params.

### LangGraph Platform UI

You can also view threads in a deployment via the LangGraph Platform UI.

Inside your deployment, select the "Threads" tab. This will load a table of all of the threads in your deployment.

To filter by thread status, select a status in the top bar. To sort by a supported property, click on the arrow icon for the desired column.

## Inspect threads

### LangGraph SDK

#### Get Thread

To view a specific thread given its `thread_id`, use the `get` method:

=== "Python"

    ```python
    print((await client.threads.get(<THREAD_ID>)))
    ```

=== "Javascript"

    ```js
    console.log((await client.threads.get(<THREAD_ID>)));
    ```

=== "CURL"

    ```bash
    curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID> \
    --header 'Content-Type: application/json'
    ```

Output:

    {
      'thread_id': 'cacf79bb-4248-4d01-aabc-938dbd60ed2c',
      'created_at': '2024-08-14T17:36:38.921660+00:00',
      'updated_at': '2024-08-14T17:36:38.921660+00:00',
      'metadata': {'graph_id': 'agent'},
      'status': 'idle',
      'config': {'configurable': {}}
    }

#### Inspect Thread State

To view the current state of a given thread, use the `get_state` method:

=== "Python"

    ```python
    print((await client.threads.get_state(<THREAD_ID>)))
    ```

=== "Javascript"

    ```js
    console.log((await client.threads.getState(<THREAD_ID>)));
    ```

=== "CURL"

    ```bash
    curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state \
    --header 'Content-Type: application/json'
    ```

Output:

    {
        "values": {
            "messages": [
                {
                    "content": "hello",
                    "additional_kwargs": {},
                    "response_metadata": {},
                    "type": "human",
                    "name": null,
                    "id": "8701f3be-959c-4b7c-852f-c2160699b4ab",
                    "example": false
                },
                {
                    "content": "Hello! How can I assist you today?",
                    "additional_kwargs": {},
                    "response_metadata": {},
                    "type": "ai",
                    "name": null,
                    "id": "4d8ea561-7ca1-409a-99f7-6b67af3e1aa3",
                    "example": false,
                    "tool_calls": [],
                    "invalid_tool_calls": [],
                    "usage_metadata": null
                }
            ]
        },
        "next": [],
        "tasks": [],
        "metadata": {
            "thread_id": "f15d70a1-27d4-4793-a897-de5609920b7d",
            "checkpoint_id": "1f02f46f-7308-616c-8000-1b158a9a6955",
            "graph_id": "agent_with_quite_a_long_name",
            "source": "update",
            "step": 1,
            "writes": {
                "call_model": {
                    "messages": [
                        {
                            "content": "Hello! How can I assist you today?",
                            "type": "ai"
                        }
                    ]
                }
            },
            "parents": {}
        },
        "created_at": "2025-05-12T15:37:09.008055+00:00",
        "checkpoint": {
            "checkpoint_id": "1f02f46f-733f-6b58-8001-ea90dcabb1bd",
            "thread_id": "f15d70a1-27d4-4793-a897-de5609920b7d",
            "checkpoint_ns": ""
        },
        "parent_checkpoint": {
            "checkpoint_id": "1f02f46f-7308-616c-8000-1b158a9a6955",
            "thread_id": "f15d70a1-27d4-4793-a897-de5609920b7d",
            "checkpoint_ns": ""
        },
        "checkpoint_id": "1f02f46f-733f-6b58-8001-ea90dcabb1bd",
        "parent_checkpoint_id": "1f02f46f-7308-616c-8000-1b158a9a6955"
    }

Optionally, to view the state of a thread at a given checkpoint, simply pass in the checkpoint id (or the entire checkpoint object):

=== "Python"

    ```python
    thread_state = await client.threads.get_state(
      thread_id=<THREAD_ID>
      checkpoint_id=<CHECKPOINT_ID>
    )
    ```

=== "Javascript"

    ```js
    const threadState = await client.threads.getState(<THREAD_ID>, <CHECKPOINT_ID>);
    ```

=== "CURL"

    ```bash
    curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state/<CHECKPOINT_ID> \
    --header 'Content-Type: application/json'
    ```

#### Inspect Full Thread History

To view a thread's history, use the `get_history` method. This returns a list of every state the thread experienced. For more information see the [Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/?h=thread+status#langgraph_sdk.client.ThreadsClient.get_history) and [JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#gethistory) reference docs.

### LangGraph Platform UI

You can also view threads in a deployment via the LangGraph Platform UI.

Inside your deployment, select the "Threads" tab. This will load a table of all of the threads in your deployment.

Select a thread to inspect its current state. To view its full history and for further debugging, open the thread in [LangGraph Studio](../../concepts//langgraph_studio.md).

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/webhooks.md
```md
# Use webhooks

When working with LangGraph Platform, you may want to use webhooks to receive updates after an API call completes. Webhooks are useful for triggering actions in your service once a run has finished processing. To implement this, you need to expose an endpoint that can accept `POST` requests and pass this endpoint as a `webhook` parameter in your API request.

Currently, the SDK does not provide built-in support for defining webhook endpoints, but you can specify them manually using API requests.

## Supported endpoints

The following API endpoints accept a `webhook` parameter:

| Operation            | HTTP Method | Endpoint                          |
|----------------------|-------------|-----------------------------------|
| Create Run           | `POST`      | `/thread/{thread_id}/runs`        |
| Create Thread Cron   | `POST`      | `/thread/{thread_id}/runs/crons`  |
| Stream Run           | `POST`      | `/thread/{thread_id}/runs/stream` |
| Wait Run             | `POST`      | `/thread/{thread_id}/runs/wait`   |
| Create Cron          | `POST`      | `/runs/crons`                     |
| Stream Run Stateless | `POST`      | `/runs/stream`                    |
| Wait Run Stateless   | `POST`      | `/runs/wait`                      |

In this guide, we’ll show how to trigger a webhook after streaming a run.

## Set up your assistant and thread

Before making API calls, set up your assistant and thread.

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    assistant_id = "agent"
    thread = await client.threads.create()
    print(thread)
    ```

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    const assistantID = "agent";
    const thread = await client.threads.create();
    console.log(thread);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/assistants/search \
        --header 'Content-Type: application/json' \
        --data '{ "limit": 10, "offset": 0 }' | jq -c 'map(select(.config == null or .config == {})) | .[0]' && \
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
    ```

Example response:

```json
{
    "thread_id": "9dde5490-2b67-47c8-aa14-4bfec88af217",
    "created_at": "2024-08-30T23:07:38.242730+00:00",
    "updated_at": "2024-08-30T23:07:38.242730+00:00",
    "metadata": {},
    "status": "idle",
    "config": {},
    "values": null
}
```

## Use a webhook with a graph run

To use a webhook, specify the `webhook` parameter in your API request. When the run completes, LangGraph Platform sends a `POST` request to the specified webhook URL.

For example, if your server listens for webhook events at `https://my-server.app/my-webhook-endpoint`, include this in your request:

=== "Python"

    ```python
    input = { "messages": [{ "role": "user", "content": "Hello!" }] }

    async for chunk in client.runs.stream(
        thread_id=thread["thread_id"],
        assistant_id=assistant_id,
        input=input,
        stream_mode="events",
        webhook="https://my-server.app/my-webhook-endpoint"
    ):
        pass
    ```

=== "JavaScript"

    ```js
    const input = { messages: [{ role: "human", content: "Hello!" }] };

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      assistantID,
      {
        input: input,
        webhook: "https://my-server.app/my-webhook-endpoint"
      }
    );

    for await (const chunk of streamResponse) {
      // Handle stream output
    }
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <ASSISTANT_ID>,
            "input": {"messages": [{"role": "user", "content": "Hello!"}]},
            "webhook": "https://my-server.app/my-webhook-endpoint"
        }'
    ```

## Webhook payload

LangGraph Platform sends webhook notifications in the format of a [Run](../../concepts/assistants.md#execution). See the [API Reference](https://langchain-ai.github.io/langgraph/cloud/reference/api/api_ref.html#model/run) for details. The request payload includes run input, configuration, and other metadata in the `kwargs` field.

## Secure webhooks

To ensure only authorized requests hit your webhook endpoint, consider adding a security token as a query parameter:

```
https://my-server.app/my-webhook-endpoint?token=YOUR_SECRET_TOKEN
```

Your server should extract and validate this token before processing requests.

## Disable webhooks

As of `langgraph-api>=0.2.78`, developers can disable webhooks in the `langgraph.json` file:

```json
{
  "http": {
    "disable_webhooks": true
  }
}
```

This feature is primarily intended for self-hosted deployments, where platform administrators or developers may prefer to disable webhooks to simplify their security posture—especially if they are not configuring firewall rules or other network controls. Disabling webhooks helps prevent untrusted payloads from being sent to internal endpoints.

For full configuration details, refer to the [configuration file reference](https://langchain-ai.github.io/langgraph/cloud/reference/cli/?h=disable_webhooks#configuration-file).

## Test webhooks

You can test your webhook using online services like:

- **[Beeceptor](https://beeceptor.com/)** – Quickly create a test endpoint and inspect incoming webhook payloads.
- **[Webhook.site](https://webhook.site/)** – View, debug, and log incoming webhook requests in real time.

These tools help you verify that LangGraph Platform is correctly triggering and sending webhooks to your service.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/click_create_assistant.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/create_assistant.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/create_assistant_view.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/create_new_version.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/edit_created_assistant.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/generative_ui_sample.jpg
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/run_in_studio.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/see_new_version.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/see_version_history.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/select_different_version.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/studio_datasets.jpg
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/studio_forks.mp4
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/studio_forks_poster.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/studio_graph_with_configuration.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/studio_input.mp4
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/studio_input_poster.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/studio_node_configuration.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/studio_playground.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/studio_screenshot.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/studio_threads.mp4
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/studio_threads_poster.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/studio_usage.mp4
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/img/studio_usage_poster.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/studio/manage_assistants.md
```md
# Manage assistants

!!! info "Prerequisites"

    - [Assistants Overview](../../../concepts/assistants.md)

LangGraph Studio lets you view, edit, and update your assistants, and allows you to run your graph using these assistant configurations.

## Graph mode

To view your assistants, click the "Manage Assistants" button in the bottom left corner.

This opens a modal for you to view all the assistants for the selected graph. Specify the assistant and its version you would like to mark as "Active", and this assistant will be used when submitting runs.

By default, the "Default configuration" option will be active. This option reflects the default configuration defined in your graph. Edits made to this configuration will be used to update the run-time configuration, but will not update or create a new assistant unless you click "Create new assistant".

## Chat mode

Chat mode enables you to switch through the different assistants in your graph via the dropdown selector at the top of the page. To create, edit, or delete assistants, use Graph mode.
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/studio/quick_start.md
```md
!!! info "Prerequisites"

    - [LangGraph Studio Overview](../../../concepts/langgraph_studio.md)

LangGraph Studio supports connecting to two types of graphs:

- Graphs deployed on [LangGraph Platform](../../../cloud/quick_start.md)
- Graphs running locally via the [LangGraph Server](../../../tutorials/langgraph-platform/local-server.md).

LangGraph Studio is accessed from the LangSmith UI, within the LangGraph Platform Deployments tab.

## Deployed application

For applications that are [deployed](../../quick_start.md) on LangGraph Platform, you can access Studio as part of that deployment. To do so, navigate to the deployment in LangGraph Platform within the LangSmith UI and click the "LangGraph Studio" button.

This will load the Studio UI connected to your live deployment, allowing you to create, read, and update the [threads](../../../concepts/persistence.md#threads), [assistants](../../../concepts/assistants.md), and [memory](../../../concepts//memory.md) in that deployment.

## Local development server

To test your locally running application using LangGraph Studio, ensure your application is set up following [this guide](https://langchain-ai.github.io/langgraph/cloud/deployment/setup/).

!!! info "LangSmith Tracing"
    For local development, if you do not wish to have data traced to LangSmith, set `LANGSMITH_TRACING=false` in your application's `.env` file. With tracing disabled, no data will leave your local server.

Next, install the [LangGraph CLI](../../../concepts/langgraph_cli.md):

```
pip install -U "langgraph-cli[inmem]"
```

and run:

```
langgraph dev
```

!!! warning "Browser Compatibility"
    Safari blocks `localhost` connections to Studio. To work around this, run the above command with `--tunnel` to access Studio via a secure tunnel.

This will start the LangGraph Server locally, running in-memory. The server will run in watch mode, listening for and automatically restarting on code changes. Read this [reference](https://langchain-ai.github.io/langgraph/cloud/reference/cli/#dev) to learn about all the options for starting the API server.

If successful, you will see the following logs:

> Ready!
>
> - API: [http://localhost:2024](http://localhost:2024/)
>
> - Docs: http://localhost:2024/docs
>
> - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024

Once running, you will automatically be directed to LangGraph Studio.

For an already running server, access Studio by either:

1.  Directly navigate to the following URL: `https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024`.
2.  Within LangSmith, navigate to the LangGraph Platform Deployments tab, click the "LangGraph Studio" button, enter `http://127.0.0.1:2024` and click "Connect".

If running your server at a different host or port, simply update the `baseUrl` to match.

### (Optional) Attach a debugger

For step-by-step debugging with breakpoints and variable inspection:

```bash
# Install debugpy package
pip install debugpy

# Start server with debugging enabled
langgraph dev --debug-port 5678
```

Then attach your preferred debugger:

=== "VS Code"

    Add this configuration to `launch.json`:

    ```json
    {
        "name": "Attach to LangGraph",
        "type": "debugpy",
        "request": "attach",
        "connect": {
          "host": "0.0.0.0",
          "port": 5678
        }
    }
    ```

=== "PyCharm" 

    1. Go to Run → Edit Configurations 
    2. Click + and select "Python Debug Server" 
    3. Set IDE host name: `localhost` 
    4. Set port: `5678` (or the port number you chose in the previous step) 
    5. Click "OK" and start debugging

## Troubleshooting

For issues getting started, please see this [troubleshooting guide](../../../troubleshooting/studio.md).

## Next steps

See the following guides for more information on how to use Studio:

- [Run application](../invoke_studio.md)
- [Manage assistants](./manage_assistants.md)
- [Manage threads](../threads_studio.md)
- [Iterate on prompts](../iterate_graph_studio.md)
- [Debug LangSmith traces](../clone_traces_studio.md)
- [Add node to dataset](../datasets_studio.md)

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/cloud/how-tos/studio/run_evals.md
```md
# Run experiments over a dataset

LangGraph Studio supports evaluations by allowing you to run your assistant over a pre-defined LangSmith dataset. This enables you to understand how your application performs over a variety of inputs, compare the results to reference outputs, and score the results using [evaluators](../../../agents/evals.md).

This guide shows you how to run an experiment end-to-end from Studio.

---

## Prerequisites

Before running an experiment, ensure you have the following:

1.  **A LangSmith dataset**: Your dataset should contain the inputs you want to test and optionally, reference outputs for comparison.

    - The schema for the inputs must match the required input schema for the assistant. For more information on schemas, see [here](../../../concepts/low_level.md#schema).
    - For more on creating datasets, see [How to Manage Datasets](https://docs.smith.langchain.com/evaluation/how_to_guides/manage_datasets_in_application#set-up-your-dataset).

2.  **(Optional) Evaluators**: You can attach evaluators (e.g., LLM-as-a-Judge, heuristics, or custom functions) to your dataset in LangSmith. These will run automatically after the graph has processed all inputs.

    - To learn more, read about [Evaluation Concepts](https://docs.smith.langchain.com/evaluation/concepts#evaluators).

3.  **A running application**: The experiment can be run against:
    - An application deployed on [LangGraph Platform](../../quick_start.md).
    - A locally running application started via the [langgraph-cli](../../../tutorials/langgraph-platform/local-server.md).

---

## Step-by-step guide

### 1. Launch the experiment

Click the **Run experiment** button in the top right corner of the Studio page.

### 2. Select your dataset

In the modal that appears, select the dataset (or a specific dataset split) to use for the experiment and click **Start**.

### 3. Monitor the progress

All of the inputs in the dataset will now be run against the active assistant. Monitor the experiment's progress via the badge in the top right corner.

You can continue to work in Studio while the experiment runs in the background. Click the arrow icon button at any time to navigate to LangSmith and view the detailed experiment results.

---

## Troubleshooting

### "Run experiment" button is disabled

If the "Run experiment" button is disabled, check the following:

- **Deployed application**: If your application is deployed on LangGraph Platform, you may need to create a new revision to enable this feature.
- **Local development server**: If you are running your application locally, make sure you have upgraded to the latest version of the `langgraph-cli` (`pip install -U langgraph-cli`). Additionally, ensure you have tracing enabled by setting the `LANGSMITH_API_KEY` in your project's `.env` file.

### Evaluator results are missing

When you run an experiment, any attached evaluators are scheduled for execution in a queue. If you don't see results immediately, it likely means they are still pending.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/examples/index.md
```md
# Examples

The pages in this section provide end-to-end examples for the following topics:

## General 

- [Template Applications](../concepts/template_applications.md): Create a LangGraph application from a template.
- [Agentic RAG](../tutorials/rag/langgraph_agentic_rag.md): Build a retrieval agent that can decide when to use a retriever tool.
- [Agent Supervisor](../tutorials/multi_agent/agent_supervisor.md): Build a supervisor agent that can manage a team of agents.
- [SQL agent](../tutorials/sql/sql-agent.md): Build a SQL agent that can execute SQL queries and return the results.
- [Prebuilt chat UI](../agents/ui.md): Use a prebuilt chat UI to interact with any LangGraph agent.
- [Graph runs in LangSmith](../how-tos/run-id-langsmith.md): Use LangSmith to track and analyze graph runs.
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/static/favicon.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/static/values_vs_updates.png
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/static/wordmark_dark.svg
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/static/wordmark_light.svg
```
Content skipped (binary or ignored type).
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/.meta.yml
```yml
tags:
  - reference
  - api
  - api-reference

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/agents.md
```md
# Agents

::: langgraph.prebuilt.chat_agent_executor
    options:
      members:
        - AgentState
        - create_react_agent

::: langgraph.prebuilt.tool_node.ToolNode
    options:
      show_if_no_docstring: true
      show_root_heading: true
      show_root_full_path: false
      inherited_members: false
      members:
        - inject_tool_args

::: langgraph.prebuilt.tool_node
    options:
      members:
        - InjectedState
        - InjectedStore
        - tools_condition

::: langgraph.prebuilt.tool_validator.ValidationNode
    options:
      show_if_no_docstring: true
      show_root_heading: true
      show_root_full_path: false
      inherited_members: false
      members: false

::: langgraph.prebuilt.interrupt
    options:
      members:
        - HumanInterruptConfig
        - ActionRequest
        - HumanInterrupt
        - HumanResponse
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/cache.md
```md
## Caching

::: langgraph.cache.base
::: langgraph.cache.memory
::: langgraph.cache.sqlite

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/channels.md
```md
# Channels

::: langgraph.channels.base
    options:
      members:
        - BaseChannel

::: langgraph.channels
    options:
      members:
        - Topic
        - LastValue
        - EphemeralValue
        - BinaryOperatorAggregate
        - AnyValue

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/checkpoints.md
```md
# Checkpointers

::: langgraph.checkpoint.base
    options:
      members:
        - CheckpointMetadata
        - Checkpoint
        - BaseCheckpointSaver
        - create_checkpoint

::: langgraph.checkpoint.serde.base
    options:
      members:
        - SerializerProtocol
        - CipherProtocol

::: langgraph.checkpoint.serde.jsonplus
    options:
      members:
        - JsonPlusSerializer

::: langgraph.checkpoint.serde.encrypted
    options:
      members:
        - EncryptedSerializer

::: langgraph.checkpoint.memory

::: langgraph.checkpoint.sqlite

::: langgraph.checkpoint.sqlite.aio

::: langgraph.checkpoint.postgres
    options:
      members:
        - PostgresSaver

::: langgraph.checkpoint.postgres.aio
    options:
      members:
        - AsyncPostgresSaver

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/config.md
```md
::: langgraph.config
    options:
      members:
        - get_store
        - get_stream_writer
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/constants.md
```md
::: langgraph.constants
    options:
      members:
        - TAG_HIDDEN
        - TAG_NOSTREAM
        - START
        - END

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/errors.md
```md
# Errors

::: langgraph.errors
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/func.md
```md
::: langgraph.func
    options:
      members:
        - task
        - entrypoint
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/graphs.md
```md
# Graph Definitions

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

::: langgraph.graph.state.CompiledStateGraph
    options:
      show_if_no_docstring: true
      show_root_heading: true
      show_root_full_path: false
      members:
        - stream
        - astream
        - invoke
        - ainvoke
        - get_state
        - aget_state
        - get_state_history
        - aget_state_history
        - update_state
        - aupdate_state
        - bulk_update_state
        - abulk_update_state
        - get_graph
        - aget_graph
        - get_subgraphs
        - aget_subgraphs
        - with_config

::: langgraph.graph.message
    options:
      members:
        - add_messages
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/index.md
```md
---
title: Reference
description: API reference for LangGraph
search:
  boost: 0.5
---

<style>
.md-sidebar {
  display: block !important;
}
</style>

# Reference

Welcome to the LangGraph reference docs! These pages detail the core interfaces you will use when building with LangGraph. Each section covers a different part of the ecosystem.

!!! tip

    If you are just getting started, see [LangGraph basics](../concepts/why-langgraph.md) for an introduction to the main concepts and usage patterns.


## LangGraph

The core APIs for the LangGraph open source library.

- [Graphs](graphs.md): Main graph abstraction and usage.
- [Functional API](func.md): Functional programming interface for graphs.
- [Pregel](pregel.md): Pregel-inspired computation model.
- [Checkpointing](checkpoints.md): Saving and restoring graph state.
- [Storage](store.md): Storage backends and options.
- [Caching](cache.md): Caching mechanisms for performance.
- [Types](types.md): Type definitions for graph components.
- [Config](config.md): Configuration options.
- [Errors](errors.md): Error types and handling.
- [Constants](constants.md): Global constants.
- [Channels](channels.md): Message passing and channels.

## Prebuilt components

Higher-level abstractions for common workflows, agents, and other patterns.

- [Agents](agents.md): Built-in agent patterns.
- [Supervisor](supervisor.md): Orchestration and delegation.
- [Swarm](swarm.md): Multi-agent collaboration.
- [MCP Adapters](mcp.md): Integrations with external systems.

## LangGraph Platform

Tools for deploying and connecting to the LangGraph Platform.

- [SDK (Python)](../cloud/reference/sdk/python_sdk_ref.md): Python SDK for interacting with instances of the LangGraph Server.
- [SDK (JS/TS)](../cloud/reference/sdk/js_ts_sdk_ref.md): JavaScript/TypeScript SDK for interacting with instances of the LangGraph Server.
- [RemoteGraph](remote_graph.md): `Pregel` abstraction for connecting to LangGraph Server instances.

See the [LangGraph Platform reference](https://docs.langchain.com/langgraph-platform/reference-overview) for more reference documentation.
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/mcp.md
```md
# LangChain Model Context Protocol (MCP) Adapters

::: langchain_mcp_adapters.client
    options:
      members:
        - MultiServerMCPClient

::: langchain_mcp_adapters.tools
    options:
      members:
        - load_mcp_tools

::: langchain_mcp_adapters.prompts
    options:
      members:
        - load_mcp_prompt

::: langchain_mcp_adapters.resources
    options:
      members:
        - load_mcp_resources
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/pregel.md
```md
# Pregel

::: langgraph.pregel.NodeBuilder
    options:
      show_if_no_docstring: true
      show_root_heading: true
      show_root_full_path: false
      members:
        - subscribe_only
        - subscribe_to
        - read_from
        - do
        - write_to
        - meta
        - retry
        - cache
        - build

::: langgraph.pregel.Pregel
    options:
      show_if_no_docstring: true
      show_root_heading: true
      show_root_full_path: false
      members:
        - stream
        - astream
        - invoke
        - ainvoke
        - get_state
        - aget_state
        - get_state_history
        - aget_state_history
        - update_state
        - aupdate_state
        - bulk_update_state
        - abulk_update_state
        - get_graph
        - aget_graph
        - get_subgraphs
        - aget_subgraphs
        - with_config
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/remote_graph.md
```md
# RemoteGraph

::: langgraph.pregel.remote
    options:
      members:
        - RemoteGraph

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/runtime.md
```md
# Runtime

::: langgraph.runtime.Runtime
    options:
      show_root_heading: true
      show_root_full_path: false
      members:
        - context
        - store
        - stream_writer
        - previous

::: langgraph.runtime
    options:
      members:
        - get_runtime



```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/store.md
```md
# Storage

::: langgraph.store.base
    

::: langgraph.store.postgres
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/supervisor.md
```md
# LangGraph Supervisor

::: langgraph_supervisor.supervisor
    options:
      members:
        - create_supervisor

::: langgraph_supervisor.handoff
    options:
      members:
        - create_handoff_tool
        - create_forward_message_tool
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/swarm.md
```md
# LangGraph Swarm

::: langgraph_swarm.swarm
    options:
      members:
        - SwarmState
        - create_swarm
        - add_active_agent_router

::: langgraph_swarm.handoff
    options:
      members:
        - create_handoff_tool
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/reference/types.md
```md
# Types

::: langgraph.types
    options:
      members:
        - All
        - StreamMode
        - StreamWriter
        - RetryPolicy
        - CachePolicy
        - Interrupt
        - PregelTask
        - StateSnapshot
        - Send
        - Command
        - interrupt

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/assets/react-agent-graphs.json
```json
{
    "0000": "graph TD;\n\t__start__ --> agent;\n\tagent --> __end__;",
    "0001": "graph TD;\n\t__start__ --> agent;\n\tagent -.-> __end__;\n\tagent -.-> tools;\n\ttools --> agent;",
    "0010": "graph TD;\n\t__start__ --> pre_model_hook;\n\tpre_model_hook --> agent;\n\tagent --> __end__;",
    "0011": "graph TD;\n\t__start__ --> pre_model_hook;\n\tagent -.-> __end__;\n\tagent -.-> tools;\n\tpre_model_hook --> agent;\n\ttools --> pre_model_hook;",
    "0100": "graph TD;\n\t__start__ --> agent;\n\tagent --> post_model_hook;\n\tpost_model_hook --> __end__;",
    "0101": "graph TD;\n\t__start__ --> agent;\n\tagent --> post_model_hook;\n\tpost_model_hook -.-> __end__;\n\tpost_model_hook -.-> agent;\n\tpost_model_hook -.-> tools;\n\ttools --> agent;",
    "0110": "graph TD;\n\t__start__ --> pre_model_hook;\n\tagent --> post_model_hook;\n\tpre_model_hook --> agent;\n\tpost_model_hook --> __end__;",
    "0111": "graph TD;\n\t__start__ --> pre_model_hook;\n\tagent --> post_model_hook;\n\tpost_model_hook -.-> __end__;\n\tpost_model_hook -.-> pre_model_hook;\n\tpost_model_hook -.-> tools;\n\tpre_model_hook --> agent;\n\ttools --> pre_model_hook;",
    "1000": "graph TD;\n\t__start__ --> agent;\n\tagent --> generate_structured_response;\n\tgenerate_structured_response --> __end__;",
    "1001": "graph TD;\n\t__start__ --> agent;\n\tagent -.-> generate_structured_response;\n\tagent -.-> tools;\n\ttools --> agent;\n\tgenerate_structured_response --> __end__;",
    "1010": "graph TD;\n\t__start__ --> pre_model_hook;\n\tagent --> generate_structured_response;\n\tpre_model_hook --> agent;\n\tgenerate_structured_response --> __end__;",
    "1011": "graph TD;\n\t__start__ --> pre_model_hook;\n\tagent -.-> generate_structured_response;\n\tagent -.-> tools;\n\tpre_model_hook --> agent;\n\ttools --> pre_model_hook;\n\tgenerate_structured_response --> __end__;",
    "1100": "graph TD;\n\t__start__ --> agent;\n\tagent --> post_model_hook;\n\tpost_model_hook --> generate_structured_response;\n\tgenerate_structured_response --> __end__;",
    "1101": "graph TD;\n\t__start__ --> agent;\n\tagent --> post_model_hook;\n\tpost_model_hook -.-> agent;\n\tpost_model_hook -.-> generate_structured_response;\n\tpost_model_hook -.-> tools;\n\ttools --> agent;\n\tgenerate_structured_response --> __end__;",
    "1110": "graph TD;\n\t__start__ --> pre_model_hook;\n\tagent --> post_model_hook;\n\tpost_model_hook --> generate_structured_response;\n\tpre_model_hook --> agent;\n\tgenerate_structured_response --> __end__;",
    "1111": "graph TD;\n\t__start__ --> pre_model_hook;\n\tagent --> post_model_hook;\n\tpost_model_hook -.-> generate_structured_response;\n\tpost_model_hook -.-> pre_model_hook;\n\tpost_model_hook -.-> tools;\n\tpre_model_hook --> agent;\n\ttools --> pre_model_hook;\n\tgenerate_structured_response --> __end__;"
}
  
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/.meta.yml
```yml
tags:
  - concepts
  - conceptual guide
  - explanation
```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/agentic_concepts.md
```md
---
search:
  boost: 2
---

# Agent architectures

Many LLM applications implement a particular control flow of steps before and / or after LLM calls. As an example, [RAG](https://github.com/langchain-ai/rag-from-scratch) performs retrieval of documents relevant to a user question, and passes those documents to an LLM in order to ground the model's response in the provided document context. 

Instead of hard-coding a fixed control flow, we sometimes want LLM systems that can pick their own control flow to solve more complex problems! This is one definition of an [agent](https://blog.langchain.dev/what-is-an-agent/): *an agent is a system that uses an LLM to decide the control flow of an application.* There are many ways that an LLM can control application:

- An LLM can route between two potential paths
- An LLM can decide which of many tools to call
- An LLM can decide whether the generated answer is sufficient or more work is needed

As a result, there are many different types of [agent architectures](https://blog.langchain.dev/what-is-a-cognitive-architecture/), which give an LLM varying levels of control. 

![Agent Types](img/agent_types.png)

## Router

A router allows an LLM to select a single step from a specified set of options. This is an agent architecture that exhibits a relatively limited level of control because the LLM usually focuses on making a single decision and produces a specific output from a limited set of pre-defined options. Routers typically employ a few different concepts to achieve this.

### Structured Output

Structured outputs with LLMs work by providing a specific format or schema that the LLM should follow in its response. This is similar to tool calling, but more general. While tool calling typically involves selecting and using predefined functions, structured outputs can be used for any type of formatted response. Common methods to achieve structured outputs include:

1. Prompt engineering: Instructing the LLM to respond in a specific format via the system prompt.
2. Output parsers: Using post-processing to extract structured data from LLM responses.
3. Tool calling: Leveraging built-in tool calling capabilities of some LLMs to generate structured outputs.

Structured outputs are crucial for routing as they ensure the LLM's decision can be reliably interpreted and acted upon by the system. Learn more about [structured outputs in this how-to guide](https://python.langchain.com/docs/how_to/structured_output/).

## Tool-calling agent

While a router allows an LLM to make a single decision, more complex agent architectures expand the LLM's control in two key ways:

1. Multi-step decision making: The LLM can make a series of decisions, one after another, instead of just one.
2. Tool access: The LLM can choose from and use a variety of tools to accomplish tasks.

[ReAct](https://arxiv.org/abs/2210.03629) is a popular general purpose agent architecture that combines these expansions, integrating three core concepts. 

1. [Tool calling](#tool-calling): Allowing the LLM to select and use various tools as needed.
2. [Memory](#memory): Enabling the agent to retain and use information from previous steps.
3. [Planning](#planning): Empowering the LLM to create and follow multi-step plans to achieve goals.

This architecture allows for more complex and flexible agent behaviors, going beyond simple routing to enable dynamic problem-solving with multiple steps. Unlike the original [paper](https://arxiv.org/abs/2210.03629), today's agents rely on LLMs' [tool calling](#tool-calling) capabilities and operate on a list of [messages](./low_level.md#why-use-messages).

In LangGraph, you can use the prebuilt [agent](../agents/agents.md#2-create-an-agent) to get started with tool-calling agents.

### Tool calling

Tools are useful whenever you want an agent to interact with external systems. External systems (e.g., APIs) often require a particular input schema or payload, rather than natural language. When we bind an API, for example, as a tool, we give the model awareness of the required input schema. The model will choose to call a tool based upon the natural language input from the user and it will return an output that adheres to the tool's required schema. 

[Many LLM providers support tool calling](https://python.langchain.com/docs/integrations/chat/) and [tool calling interface](https://blog.langchain.dev/improving-core-tool-interfaces-and-docs-in-langchain/) in LangChain is simple: you can simply pass any Python `function` into `ChatModel.bind_tools(function)`.

![Tools](img/tool_call.png)

### Memory

[Memory](../how-tos/memory/add-memory.md) is crucial for agents, enabling them to retain and utilize information across multiple steps of problem-solving. It operates on different scales:

1. [Short-term memory](../how-tos/memory/add-memory.md#add-short-term-memory): Allows the agent to access information acquired during earlier steps in a sequence.
2. [Long-term memory](../how-tos/memory/add-memory.md#add-long-term-memory): Enables the agent to recall information from previous interactions, such as past messages in a conversation.

LangGraph provides full control over memory implementation:

- [`State`](./low_level.md#state): User-defined schema specifying the exact structure of memory to retain.
- [`Checkpointer`](./persistence.md#checkpoints): Mechanism to store state at every step across different interactions within a session.
- [`Store`](./persistence.md#memory-store): Mechanism to store user-specific or application-level data across sessions.

This flexible approach allows you to tailor the memory system to your specific agent architecture needs. Effective memory management enhances an agent's ability to maintain context, learn from past experiences, and make more informed decisions over time. For a practical guide on adding and managing memory, see [Memory](../how-tos/memory/add-memory.md).

### Planning

In a tool-calling [agent](../agents/overview.md#what-is-an-agent), an LLM is called repeatedly in a while-loop. At each step the agent decides which tools to call, and what the inputs to those tools should be. Those tools are then executed, and the outputs are fed back into the LLM as observations. The while-loop terminates when the agent decides it has enough information to solve the user request and it is not worth calling any more tools.

## Custom agent architectures

While routers and tool-calling agents (like ReAct) are common, [customizing agent architectures](https://blog.langchain.dev/why-you-should-outsource-your-agentic-infrastructure-but-own-your-cognitive-architecture/) often leads to better performance for specific tasks. LangGraph offers several powerful features for building tailored agent systems:

### Human-in-the-loop

Human involvement can significantly enhance agent reliability, especially for sensitive tasks. This can involve:

- Approving specific actions
- Providing feedback to update the agent's state
- Offering guidance in complex decision-making processes

Human-in-the-loop patterns are crucial when full automation isn't feasible or desirable. Learn more in our [human-in-the-loop guide](./human_in_the_loop.md).

### Parallelization 

Parallel processing is vital for efficient multi-agent systems and complex tasks. LangGraph supports parallelization through its [Send](./low_level.md#send) API, enabling:

- Concurrent processing of multiple states
- Implementation of map-reduce-like operations
- Efficient handling of independent subtasks

For practical implementation, see our [map-reduce tutorial](../how-tos/graph-api.md#map-reduce-and-the-send-api)

### Subgraphs

[Subgraphs](./subgraphs.md) are essential for managing complex agent architectures, particularly in [multi-agent systems](./multi_agent.md). They allow:

- Isolated state management for individual agents
- Hierarchical organization of agent teams
- Controlled communication between agents and the main system

Subgraphs communicate with the parent graph through overlapping keys in the state schema. This enables flexible, modular agent design. For implementation details, refer to our [subgraph how-to guide](../how-tos/subgraph.md).

### Reflection

Reflection mechanisms can significantly improve agent reliability by:

1. Evaluating task completion and correctness
2. Providing feedback for iterative improvement
3. Enabling self-correction and learning

While often LLM-based, reflection can also use deterministic methods. For instance, in coding tasks, compilation errors can serve as feedback. This approach is demonstrated in [this video using LangGraph for self-corrective code generation](https://www.youtube.com/watch?v=MvNdgmM7uyc).

By leveraging these features, LangGraph enables the creation of sophisticated, task-specific agent architectures that can handle complex workflows, collaborate effectively, and continuously improve their performance.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/application_structure.md
```md
---
search:
  boost: 2
---

# Application Structure

## Overview

A LangGraph application consists of one or more graphs, a configuration file (`langgraph.json`), a file that specifies dependencies, and an optional `.env` file that specifies environment variables.

This guide shows a typical structure of an application and shows how the required information to deploy an application using the LangGraph Platform is specified.

## Key Concepts

To deploy using the LangGraph Platform, the following information should be provided:

1. A [LangGraph configuration file](#configuration-file-concepts) (`langgraph.json`) that specifies the dependencies, graphs, and environment variables to use for the application.
2. The [graphs](#graphs) that implement the logic of the application.
3. A file that specifies [dependencies](#dependencies) required to run the application.
4. [Environment variables](#environment-variables) that are required for the application to run.

## File Structure

Below are examples of directory structures for applications:

:::python
=== "Python (requirements.txt)"

    ```plaintext
    my-app/
    ├── my_agent # all project code lies within here
    │   ├── utils # utilities for your graph
    │   │   ├── __init__.py
    │   │   ├── tools.py # tools for your graph
    │   │   ├── nodes.py # node functions for your graph
    │   │   └── state.py # state definition of your graph
    │   ├── __init__.py
    │   └── agent.py # code for constructing your graph
    ├── .env # environment variables
    ├── requirements.txt # package dependencies
    └── langgraph.json # configuration file for LangGraph
    ```

=== "Python (pyproject.toml)"

    ```plaintext
    my-app/
    ├── my_agent # all project code lies within here
    │   ├── utils # utilities for your graph
    │   │   ├── __init__.py
    │   │   ├── tools.py # tools for your graph
    │   │   ├── nodes.py # node functions for your graph
    │   │   └── state.py # state definition of your graph
    │   ├── __init__.py
    │   └── agent.py # code for constructing your graph
    ├── .env # environment variables
    ├── langgraph.json  # configuration file for LangGraph
    └── pyproject.toml # dependencies for your project
    ```

:::

:::js

```plaintext
my-app/
├── src # all project code lies within here
│   ├── utils # optional utilities for your graph
│   │   ├── tools.ts # tools for your graph
│   │   ├── nodes.ts # node functions for your graph
│   │   └── state.ts # state definition of your graph
│   └── agent.ts # code for constructing your graph
├── package.json # package dependencies
├── .env # environment variables
└── langgraph.json # configuration file for LangGraph
```

:::

!!! note

    The directory structure of a LangGraph application can vary depending on the programming language and the package manager used.

## Configuration File {#configuration-file-concepts}

The `langgraph.json` file is a JSON file that specifies the dependencies, graphs, environment variables, and other settings required to deploy a LangGraph application.

See the [LangGraph configuration file reference](../cloud/reference/cli.md#configuration-file) for details on all supported keys in the JSON file.

!!! tip

    The [LangGraph CLI](./langgraph_cli.md) defaults to using the configuration file `langgraph.json` in the current directory.

### Examples

:::python

- The dependencies involve a custom local package and the `langchain_openai` package.
- A single graph will be loaded from the file `./your_package/your_file.py` with the variable `variable`.
- The environment variables are loaded from the `.env` file.

```json
{
  "dependencies": ["langchain_openai", "./your_package"],
  "graphs": {
    "my_agent": "./your_package/your_file.py:agent"
  },
  "env": "./.env"
}
```

:::

:::js

- The dependencies will be loaded from a dependency file in the local directory (e.g., `package.json`).
- A single graph will be loaded from the file `./your_package/your_file.js` with the function `agent`.
- The environment variable `OPENAI_API_KEY` is set inline.

```json
{
  "dependencies": ["."],
  "graphs": {
    "my_agent": "./your_package/your_file.js:agent"
  },
  "env": {
    "OPENAI_API_KEY": "secret-key"
  }
}
```

:::

## Dependencies

:::python
A LangGraph application may depend on other Python packages.
:::

:::js
A LangGraph application may depend on other TypeScript/JavaScript libraries.
:::

You will generally need to specify the following information for dependencies to be set up correctly:

:::python

1. A file in the directory that specifies the dependencies (e.g. `requirements.txt`, `pyproject.toml`, or `package.json`).
   :::

:::js

1. A file in the directory that specifies the dependencies (e.g. `package.json`).
   :::

2. A `dependencies` key in the [LangGraph configuration file](#configuration-file-concepts) that specifies the dependencies required to run the LangGraph application.
3. Any additional binaries or system libraries can be specified using `dockerfile_lines` key in the [LangGraph configuration file](#configuration-file-concepts).

## Graphs

Use the `graphs` key in the [LangGraph configuration file](#configuration-file-concepts) to specify which graphs will be available in the deployed LangGraph application.

You can specify one or more graphs in the configuration file. Each graph is identified by a name (which should be unique) and a path for either: (1) the compiled graph or (2) a function that makes a graph is defined.

## Environment Variables

If you're working with a deployed LangGraph application locally, you can configure environment variables in the `env` key of the [LangGraph configuration file](#configuration-file-concepts).

For a production deployment, you will typically want to configure the environment variables in the deployment environment.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/assistants.md
```md
# Assistants

**Assistants** allow you to manage configurations (like prompts, LLM selection, tools) separately from your graph's core logic, enabling rapid changes that don't alter the graph architecture. It is a way to create multiple specialized versions of the same graph architecture, each optimized for different use cases through context/configuration variations rather than structural changes.

For example, imagine a general-purpose writing agent built on a common graph architecture. While the structure remains the same, different writing styles—such as blog posts and tweets—require tailored configurations to optimize performance. To support these variations, you can create multiple assistants (e.g., one for blogs and another for tweets) that share the underlying graph but differ in model selection and system prompt.

![assistant versions](img/assistants.png)

The LangGraph Cloud API provides several endpoints for creating and managing assistants and their versions. See the [API reference](../cloud/reference/api/api_ref.html#tag/assistants) for more details.

!!! info

    Assistants are a [LangGraph Platform](langgraph_platform.md) concept. They are not available in the open source LangGraph library.

## Configuration

:::python
Assistants build on the LangGraph open source concepts of configuration and [runtime context](low_level.md#runtime-context).
:::

While these features are available in the open source LangGraph library, assistants are only present in [LangGraph Platform](langgraph_platform.md). This is due to the fact that assistants are tightly coupled to your deployed graph. Upon deployment, LangGraph Server will automatically create a default assistant for each graph using the graph's default context and configuration settings.

In practice, an assistant is just an _instance_ of a graph with a specific configuration. Therefore, multiple assistants can reference the same graph but can contain different configurations (e.g. prompts, models, tools). The LangGraph Server API provides several endpoints for creating and managing assistants. See the [API reference](../cloud/reference/api/api_ref.html) and [this how-to](../cloud/how-tos/configuration_cloud.md) for more details on how to create assistants.

## Versioning

Assistants support versioning to track changes over time.
Once you've created an assistant, subsequent edits to that assistant will create new versions. See [this how-to](../cloud/how-tos/configuration_cloud.md#create-a-new-version-for-your-assistant) for more details on how to manage assistant versions.

## Execution

A **run** is an invocation of an assistant. Each run may have its own input, configuration, context, and metadata, which may affect execution and output of the underlying graph. A run can optionally be executed on a [thread](./persistence.md#threads).

The LangGraph Platform API provides several endpoints for creating and managing runs. See the [API reference](../cloud/reference/api/api_ref.html#tag/thread-runs/) for more details.

```
---
https://github.com/langchain-ai/langgraph/blob/main/docs/docs/concepts/auth.md
```md
---
search:
  boost: 2
---

# Authentication & Access Control

LangGraph Platform provides a flexible authentication and authorization system that can integrate with most authentication schemes.

## Core Concepts

### Authentication vs Authorization

While often used interchangeably, these terms represent distinct security concepts:

- [**Authentication**](#authentication) ("AuthN") verifies _who_ you are. This runs as middleware for every request.
- [**Authorization**](#authorization) ("AuthZ") determines _what you can do_. This validates the user's privileges and roles on a per-resource basis.

:::python
In LangGraph Platform, authentication is handled by your [`@auth.authenticate`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.Auth.authenticate) handler, and authorization is handled by your [`@auth.on`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.Auth.on) handlers.
:::

:::js
In LangGraph Platform, authentication is handled by your [`@auth.authenticate`](../cloud/reference/sdk/typescript_sdk_ref.md#auth.authenticate) handler, and authorization is handled by your [`@auth.on`](../cloud/reference/sdk/typescript_sdk_ref.md#auth.on) handlers.
:::

## Default Security Models

LangGraph Platform provides different security defaults:

### LangGraph Platform

- Uses LangSmith API keys by default
- Requires valid API key in `x-api-key` header
- Can be customized with your auth handler

!!! note "Custom auth"

   Custom auth **is supported** for all plans in LangGraph Platform.

### Self-Hosted

- No default authentication
- Complete flexibility to implement your security model
- You control all aspects of authentication and authorization

## System Architecture

A typical authentication setup involves three main components:

1. **Authentication Provider** (Identity Provider/IdP)

   - A dedicated service that manages user identities and credentials
   - Handles user registration, login, password resets, etc.
   - Issues tokens (JWT, session tokens, etc.) after successful authentication
   - Examples: Auth0, Supabase Auth, Okta, or your own auth server

2. **LangGraph Backend** (Resource Server)

   - Your LangGraph application that contains business logic and protected resources
   - Validates tokens with the auth provider
   - Enforces access control based on user identity and permissions
   - Doesn't store user credentials directly

3. **Client Application** (Frontend)

   - Web app, mobile app, or API client
   - Collects time-sensitive user credentials and sends to auth provider
   - Receives tokens from auth provider
   - Includes these tokens in requests to LangGraph backend

Here's how these components typically interact:

```mermaid
sequenceDiagram
    participant Client as Client App
    participant Auth as Auth Provider
    participant LG as LangGraph Backend

    Client->>Auth: 1. Login (username/password)
    Auth-->>Client: 2. Return token
    Client->>LG: 3. Request with token
    Note over LG: 4. Validate token (@auth.authenticate)
    LG-->>Auth:  5. Fetch user info
    Auth-->>LG: 6. Confirm validity
    Note over LG: 7. Apply access control (@auth.on.*)
    LG-->>Client: 8. Return resources
```

:::python
Your [`@auth.authenticate`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.Auth.authenticate) handler in LangGraph handles steps 4-6, while your [`@auth.on`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.Auth.on) handlers implement step 7.
:::

:::js
Your [`auth.authenticate`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#authenticate) handler in LangGraph handles steps 4-6, while your [`auth.on`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#on>) handlers implement step 7.
:::

## Authentication

:::python
Authentication in LangGraph runs as middleware on every request. Your [`@auth.authenticate`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.Auth.authenticate) handler receives request information and should:

1. Validate the credentials
2. Return [user info](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.MinimalUserDict) containing the user's identity and user information if valid
3. Raise an [HTTPException](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.exceptions.HTTPException) or AssertionError if invalid

```python
from langgraph_sdk import Auth

auth = Auth()

@auth.authenticate
async def authenticate(headers: dict) -> Auth.types.MinimalUserDict:
    # Validate credentials (e.g., API key, JWT token)
    api_key = headers.get("x-api-key")
    if not api_key or not is_valid_key(api_key):
        raise Auth.exceptions.HTTPException(
            status_code=401,
            detail="Invalid API key"
        )

    # Return user info - only identity and is_authenticated are required
    # Add any additional fields you need for authorization
    return {
        "identity": "user-123",        # Required: unique user identifier
        "is_authenticated": True,      # Optional: assumed True by default
        "permissions": ["read", "write"] # Optional: for permission-based auth
        # You can add more custom fields if you want to implement other auth patterns
        "role": "admin",
        "org_id": "org-456"

    }
```

The returned user information is available:

- To your authorization handlers via [`ctx.user`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.AuthContext)
- In your application via `config["configuration"]["langgraph_auth_user"]`
  :::

:::js
Authentication in LangGraph runs as middleware on every request. Your [`authenticate`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#authenticate>) handler receives request information and should:

1. Validate the credentials
2. Return user information containing the user's identity and user information if valid
3. Raise an [HTTPException](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#class-httpexception>) if invalid

```typescript
import { Auth, HTTPException } from "@langchain/langgraph-sdk";

export const auth = new Auth();

auth.authenticate(async (request) => {
  // Validate credentials (e.g., API key, JWT token)
  const apiKey = request.headers.get("x-api-key");
  if (!apiKey || !isValidKey(apiKey)) {
    throw new HTTPException(401, "Invalid API key");
  }

  // Return user info - only identity and isAuthenticated are required
  // Add any additional fields you need for authorization
  return {
    identity: "user-123", // Required: unique user identifier
    isAuthenticated: true, // Optional: assumed true by default
    permissions: ["read", "write"], // Optional: for permission-based auth
    // You can add more custom fields if you want to implement other auth patterns
    role: "admin",
    orgId: "org-456",
  };
});
```

The returned user information is available:

- To your authorization handlers via the `user` property in a [callback handler](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#on)
- In your application via `config.configurable.langgraph_auth_user`
  :::

??? tip "Supported Parameters"

    :::python
    The [`@auth.authenticate`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.Auth.authenticate) handler can accept any of the following parameters by name:

    * request (Request): The raw ASGI request object
    * body (dict): The parsed request body
    * path (str): The request path, e.g., "/threads/abcd-1234-abcd-1234/runs/abcd-1234-abcd-1234/stream"
    * method (str): The HTTP method, e.g., "GET"
    * path_params (dict[str, str]): URL path parameters, e.g., {"thread_id": "abcd-1234-abcd-1234", "run_id": "abcd-1234-abcd-1234"}
    * query_params (dict[str, str]): URL query parameters, e.g., {"stream": "true"}
    * headers (dict[bytes, bytes]): Request headers
    * authorization (str | None): The Authorization header value (e.g., "Bearer <token>")
    :::

    :::js
    The [`authenticate`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#authenticate) handler can accept any of the following parameters:

    * request (Request): The raw request object
    * body (object): The parsed request body
    * path (string): The request path, e.g., "/threads/abcd-1234-abcd-1234/runs/abcd-1234-abcd-1234/stream"
    * method (string): The HTTP method, e.g., "GET"
    * pathParams (Record<string, string>): URL path parameters, e.g., {"threadId": "abcd-1234-abcd-1234", "runId": "abcd-1234-abcd-1234"}
    * queryParams (Record<string, string>): URL query parameters, e.g., {"stream": "true"}
    * headers (Record<string, string>): Request headers
    * authorization (string | null): The Authorization header value (e.g., "Bearer <token>")
    :::

    In many of our tutorials, we will just show the "authorization" parameter to be concise, but you can opt to accept more information as needed
    to implement your custom authentication scheme.

### Agent authentication

Custom authentication permits delegated access. The values you return in `@auth.authenticate` are added to the run context, giving agents user-scoped credentials lets them access resources on the user’s behalf.

```mermaid
sequenceDiagram
  %% Actors
  participant ClientApp as Client
  participant AuthProv  as Auth Provider
  participant LangGraph as LangGraph Backend
  participant SecretStore as Secret Store
  participant ExternalService as External Service

  %% Platform login / AuthN
  ClientApp  ->> AuthProv: 1. Login (username / password)
  AuthProv   -->> ClientApp: 2. Return token
  ClientApp  ->> LangGraph: 3. Request with token

  Note over LangGraph: 4. Validate token (@auth.authenticate)
  LangGraph  -->> AuthProv: 5. Fetch user info
  AuthProv   -->> LangGraph: 6. Confirm validity

  %% Fetch user tokens from secret store
  LangGraph  ->> SecretStore: 6a. Fetch user tokens
  SecretStore -->> LangGraph: 6b. Return tokens

  Note over LangGraph: 7. Apply access control (@auth.on.*)

  %% External Service round-trip
  LangGraph  ->> ExternalService: 8. Call external service (with header)
  Note over ExternalService: 9. External service validates header and executes action
  ExternalService  -->> LangGraph: 10. Service response

  %% Return to caller
  LangGraph  -->> ClientApp: 11. Return resources
```

After authentication, the platform creates a special configuration object that is passed to your graph and all nodes via the configurable context.
This object contains information about the current user, including any custom fields you return from your [`@auth.authenticate`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.Auth.authenticate) handler.

To enable an agent to act on behalf of the user, use [custom authentication middleware](../how-tos/auth/custom_auth.md). This will allow the agent to interact with external systems like MCP servers, external databases, and even other agents on behalf of the user.

For more information, see the [Use custom auth](../how-tos/auth/custom_auth.md#enable-agent-authentication) guide.

### Agent authentication with MCP

For information on how to authenticate an agent to an MCP server, see the [MCP conceptual guide](../concepts/mcp.md).

## Authorization

After authentication, LangGraph calls your authorization handlers to control access to specific resources (e.g., threads, assistants, crons). These handlers can:

1. Add metadata to be saved during resource creation by mutating the metadata. See the [supported actions table](#supported-actions) for the list of types the value can take for each action.
2. Filter resources by metadata during search/list or read operations by returning a [filter](#filter-operations).
3. Raise an HTTP exception if access is denied.

If you want to just implement simple user-scoped access control, you can use a single authorization handler for all resources and actions. If you want to have different control depending on the resource and action, you can use [resource-specific handlers](#resource-specific-handlers). See the [Supported Resources](#supported-resources) section for a full list of the resources that support access control.

:::python
Your [`@auth.on`](../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.Auth.on) handlers control access by mutating the `value["metadata"]` dictionary directly and returning a [filter dictionary](#filter-operations).

```python
@auth.on
async def add_owner(
    ctx: Auth.types.AuthContext,
    value: dict  # The payload being sent to this access method
) -> dict:  # Returns a filter dict that restricts access to resources
    """Authorize all access to threads, runs, crons, and assistants.

    This handler does two things:
        - Adds a value to resource metadata (to persist with the resource so it can be filtered later)
        - Returns a filter (to restrict access to existing resources)

    Args:
        ctx: Authentication context containing user info, permissions, the path, and
        value: The request payload sent to the endpoint. For creation
              operations, this contains the resource parameters. For read
              operations, this contains the resource being accessed.

    Returns:
        A filter dictionary that LangGraph uses to restrict access to resources.
        See [Filter Operations](#filter-operations) for supported operators.
    """
    # Create filter to restrict access to just this user's resources
    filters = {"owner": ctx.user.identity}

    # Get or create the metadata dictionary in the payload
    # This is where we store persistent info about the resource
    metadata = value.setdefault("metadata", {})

    # Add owner to metadata - if this is a create or update operation,
    # this information will be saved with the resource
    # So we can filter by it later in read operations
    metadata.update(filters)

    # Return filters to restrict access
    # These filters are applied to ALL operations (create, read, update, search, etc.)
    # to ensure users can only access their own resources
    return filters
```

:::

:::js
You can granularly control access by mutating the `value.metadata` object directly and returning a [filter object](#filter-operations) when registering an [`on()`](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#on) handler.

```typescript
import { Auth, HTTPException } from "@langchain/langgraph-sdk/auth";

export const auth = new Auth()
  .authenticate(async (request: Request) => ({
    identity: "user-123",
    permissions: [],
  }))
  .on("*", ({ value, user }) => {
    // Create filter to restrict access to just this user's resources
    const filters = { owner: user.identity };

    // If the operation supports metadata, add the user identity
    // as metadata to the resource.
    if ("metadata" in value) {
      value.metadata ??= {};
      value.metadata.owner = user.identity;
    }

    // Return filters to restrict access
    // These filters are applied to ALL operations (create, read, update, search, etc.)
    // to ensure users can only access their own resources
    return filters;
  });
```

:::

### Resource-Specific Handlers {#resource-specific-handlers}

You can register handlers for specific resources and actions by chaining the resource and action names together with the authorization decorator.
When a request is made, the most specific handler that matches that resource and action is called. Below is an example of how to register handlers for specific resources and actions. For the following setup:

1. Authenticated users are able to create threads, read threads, and create runs on threads
2. Only users with the "assistants:create" permission are allowed to create new assistants
3. All other endpoints (e.g., e.g., delete assistant, crons, store) are disabled for all users.

!!! tip "Supported Handlers"

    For a full list of supported resources and actions, see the [Supported Resources](#supported-resources) section below.

:::python

```python
# Generic / global handler catches calls that aren't handled by more specific handlers
@auth.on
async def reject_unhandled_requests(ctx: Auth.types.AuthContext, value: Any) -> False:
    print(f"Request to {ctx.path} by {ctx.user.identity}")
    raise Auth.exceptions.HTTPException(
        status_code=403,
        detail="Forbidden"
    )

# Matches the "thread" resource and all actions - create, read, update, delete, search
# Since this is **more specific** than the generic @auth.on handler, it will take precedence
# over the generic handler for all actions on the "threads" resource
@auth.on.threads
async def on_thread_create(
    ctx: Auth.types.AuthContext,
    value: Auth.types.threads.create.value
):
    if "write" not in ctx.permissions:
        raise Auth.exceptions.HTTPException(
            status_code=403,
            detail="User lacks the required permissions."
        )
    # Setting metadata on the thread being created
    # will ensure that the resource contains an "owner" field
    # Then any time a user tries to access this thread or runs within the thread,
    # we can filter by owner
    metadata = value.setdefault("metadata", {})
    metadata["owner"] = ctx.user.identity
    return {"owner": ctx.user.identity}

# Thread creation. This will match only on thread create actions
# Since this is **more specific** than both the generic @auth.on handler and the @auth.on.threads handler,
# it will take precedence for any "create" actions on the "threads" resources
@auth.on.threads.create
async def on_thread_create(
    ctx: Auth.types.AuthContext,
    value: Auth.types.threads.create.value
):
    # Setting metadata on the thread being created
    # will ensure that the resource contains an "owner" field
