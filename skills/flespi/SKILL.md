---
name: flespi
description: flespi is an IoT and telematics cloud platform. Provides skills for developing applications and consulting agents that integrate with the flespi REST API and MQTT broker.
user-invocable: true
argument-hint: "<question about flespi platform, API, or MQTT>"
metadata:
  flespi:
    category: telematics
    api_base: https://flespi.io
    mqtt_broker: mqtt.flespi.io
---

# flespi Platform Concepts

## Data Flow

```
Physical Device  -->  Channel  -->  Device  -->  Stream (export)
  (tracker)        (protocol      (message     Calculator (analytics)
                    endpoint)      storage)     Plugin (transform)
```

1. **Physical device** sends data over TCP/UDP/HTTP using a vendor protocol (e.g. Teltonika, Queclink)
2. **Channel** listens on a port, parses raw bytes into normalized JSON messages
3. **Device** receives parsed messages, stores them as time-series data, maintains latest telemetry snapshot
4. **Stream** forwards device messages to external systems in real-time (AWS IoT, Azure, HTTP, MQTT, etc.)
5. **Calculator** processes device messages into analytics intervals (trips, stops, geofence events, fuel consumption)
6. **Plugin** transforms or enriches messages in the pipeline before storage

## Core Entities

### Channel
Inbound protocol endpoint. Each channel binds to one protocol and listens for connections.
- Key fields: `name`, `protocol_id`, `enabled`
- Sub-resources: `/messages`, `/logs`, `/connections`
- Devices link to channels through `device_type_id` (protocol family) and `configuration.ident` (IMEI)

### Device
Logical representation of a physical tracker. Central entity for data storage.
- Key fields: `name`, `device_type_id`, `configuration.ident` (IMEI/serial), `enabled`, `connected`
- Sub-resources: `/messages` (time-series), `/telemetry` (latest snapshot), `/settings`, `/commands_queue`, `/logs`
- `connected` field indicates whether device is currently sending data

### Stream
Forwards device messages to external systems in real-time.
- Key fields: `name`, `type`, `configuration`, `queue_ttl`
- Types: http, mqtt, aws_iot, azure_event_hub, google_pubsub, wialon, and more
- Sub-resources: `/logs` (delivery status and errors)

### Calculator
Real-time analytics engine. Processes device messages into intervals.
- Key fields: `name`, `selectors` (interval boundary conditions), `counters` (what to compute)
- **Selectors** define when intervals start/end (e.g. ignition on/off, speed threshold, geofence enter/exit)
- **Counters** define what to measure (duration, distance, fuel, max speed, etc.)
- Sub-resources: `/devices/{device_id}/intervals/all`

### Plugin
Transforms or enriches messages in the processing pipeline.
- Key fields: `name`, `item_type`, `configuration`, `validate_message`
- Can add/modify/filter message parameters before storage

### Geofence
Geographic area used by calculators and plugins for location-based logic.
- Types: polygon, circle, corridor
- Key fields: `name`, `geometry`

### Group
Organizes devices for bulk assignment to streams, calculators, and plugins.

### Webhook
Reacts to MQTT events with HTTP requests. Event-driven automation.
- Key fields: `name`, `triggers` (MQTT topic filters), `configuration` (HTTP request template)
- Uses `%{expression}%` syntax in request body templates

### Token
API access credential with configurable permissions.
- Access types: Master (type=1, full access), Standard (type=0, telematics access), ACL (type=2, granular permissions)
- Every API request requires `Authorization: FlespiToken <key>` header

### Subaccount
Hierarchical customer structure for multi-tenant deployments.
- Children inherit resource limits from parent
- Operate as a subaccount using `x-flespi-cid` header or `cid` parameter

### Container
Generic message storage (not tied to devices). Used for custom data.

### CDN
File storage service.

## Key Message Parameters

Device messages are JSON objects with normalized parameter names:

| Parameter | Description |
|-----------|-------------|
| `timestamp` | Device-reported time (unix UTC) |
| `server.timestamp` | flespi reception time (unix UTC) |
| `position.latitude`, `position.longitude` | GPS coordinates |
| `position.speed` | Speed (km/h) |
| `position.direction` | Heading (degrees) |
| `position.altitude` | Altitude (meters) |
| `position.satellites` | Satellite count |
| `position.valid` | GPS fix validity |
| `ident` | Device IMEI/serial number |
| `device.id`, `device.name` | flespi device identity |
| `device.type.id` | Device type ID |
| `channel.id` | Channel ID that received the message |

If `server.timestamp - timestamp > 3600`, it usually indicates time sync issues or buffered/historical (blackbox) data.

## Expressions

flespi expressions are used across the platform: in calculators, plugins, streams, webhooks, REST API filters, and MQTT topic filters.

- Parameter access: `$param_name` (safe - returns null if missing) or `param_name`
- Nested access: `$position.speed`, `$can.vehicle.mileage`
- Previous value (calculators/plugins only): `previous("param_name")`
- Built-in functions: `mileage()`, `distance_to(lat,lon)`, `duration()`, `geofence("name")`
- Operators: `+`, `-`, `*`, `/`, `%`, `==`, `!=`, `>`, `<`, `>=`, `<=`, `&&`, `||`, `!`, `? :`
- String matching: `~` (wildcard match)
- Webhook templating: `%{$device.name}%`
- REST filter: `{"filter": "position.speed>60&&position.valid==true"}`

# flespi REST API

Base URL: `https://flespi.io`

| Namespace | Contents |
|-----------|----------|
| `/gw/` | Channels, devices, streams, calculators, plugins, geofences, groups, modems, protocols |
| `/storage/` | Containers, CDNs |
| `/platform/` | Customer, tokens, subaccounts, webhooks, grants, realms, logs, statistics |
| `/mqtt/` | MQTT sessions, subscriptions |

## Authentication

Every request requires: `Authorization: FlespiToken <key>`

### Token Types

| Type | `access.type` | Scope |
|------|---------------|-------|
| Master | 1 | Full access including platform API (tokens, subaccounts, webhooks, realms). Can create other tokens. |
| Standard | 0 | Telematics entities only (channels, devices, streams, calcs, plugins, etc.). No platform API. |
| ACL | 2 | Granular per-URI, per-method, per-item permissions. Default-deny - only explicitly allowed actions work. |

### Obtaining a Token

**Via flespi panel**: Tokens section in left menu -> "+" button -> choose type, configure ACL, set expiration.

**Via REST API** (requires Master token):
```
POST /platform/tokens
[{"info": "My token", "ttl": 31536000, "access": {"type": 2, "acl": [{"uri": "gw/devices", "methods": ["GET"]}]}}]
```

**Via realm login** (for realm-managed users):
```
POST /realm/{realm-public-id}/login
{"username": "user", "password": "pass"}
```

**Token validation**: `GET /auth/info` - returns token info if valid, 401 if not.

### Token Expiration

Tokens require either `ttl` (seconds since last use) or `expire` (unix timestamp), or both. Token remains valid as long as at least one is valid. When both expire, the token is auto-deleted.

## Response Format

Success:
```json
{"result": [...]}
```

Error:
```json
{"errors": [{"code": 404, "reason": "not found"}]}
```

HTTP 400 errors often include `schema_hint` with valid field names - use it to retry with correct fields.

## Selectors

Selectors in URL paths address which items to operate on:

| Selector | Example | Description |
|----------|---------|-------------|
| By ID | `/gw/devices/123` | Single item |
| Multiple IDs | `/gw/devices/123,456,789` | Comma-separated |
| All | `/gw/devices/all` | Every item |
| By field | `/gw/devices/configuration.ident=352625066004561` | Exact field match |
| By expression | `/gw/devices/{name=="my tracker" || device_type_id==42}` | Expression filter |

**Caution**: `all` in write/delete operations (`PUT`, `PATCH`, `DELETE`) affects every item.

## Limiting Response Size

### Collection endpoints (list entities)

Use `?fields=` query parameter:
```
GET /gw/devices/all?fields=id,name,device_type_id,connected
```
Field names are flat - use `errors`, not `counters.errors`.

### Message/log/interval endpoints

Use `data` parameter (JSON body) instead of `?fields=`:
```
GET /gw/devices/123/messages
data: {"count": 10, "fields": "timestamp,position.latitude,position.longitude,position.speed", "reverse": true}
```

| Parameter | Description |
|-----------|-------------|
| `count` | Max records to return |
| `fields` | Comma-separated field names |
| `reverse` | `true` for latest-first |
| `from` / `to` | Time range (unix UTC) |
| `filter` | flespi expression to filter records |
| `domain` | Limits fields by domain (e.g. `position`, `can`) |

Latest message: `data={"count":1,"reverse":true}`

## CRUD Operations

| Method | Action | Body |
|--------|--------|------|
| `POST` | Create | Array of objects: `[{"name":"My Device","device_type_id":42}]` |
| `GET` | Read | - |
| `PUT` | Replace specified fields | Fields to replace |
| `PATCH` | Patch JSON object fields | Partial object: `{"configuration":{"ident": "new ident"}}` |
| `DELETE` | Remove | - |

POST accepts arrays. Response `result` contains items with assigned `id` fields.

## Pagination

### Collection endpoints

Use `limit` and `offset` query parameters:
```
GET /gw/devices/all?fields=id,name&limit=10&offset=0
```
Response includes `pagination` object with `limit`, `offset`, `count` (total). Increment `offset` by `limit` until done.

### Message/log/interval endpoints

No `limit`/`offset`. Use time-based pagination via `data` body:

1. Fetch batch: `data={"from":1700000000,"to":1700086400,"count":1000}`
2. Take `timestamp` of last message
3. Use as new `from` (or `to` if `reverse:true`) for next request

For calculator intervals, use `begin`/`end` instead of `from`/`to`.

Note: for messages, `count` limits total across all queried devices. For intervals, `count` limits per device.

### Telemetry endpoints

No pagination. Page the device list first, then fetch telemetry per batch of IDs.

## Calculate Method

Server-side aggregation without a persistent calculator:
```
POST /gw/devices/{selector}/calculate
data: {
  "from": 1700000000,
  "to": 1700086400,
  "selectors": [{"type": "datetime"}],
  "counters": [
    {"type": "expression", "name": "mileage", "method": "summary", "expression": "mileage()"},
    {"type": "parameter", "name": "max_speed", "parameter": "position.speed", "method": "max"}
  ]
}
```
This is a POST even though it only reads data. Use it for on-demand aggregation instead of fetching all messages client-side.

## Auxiliary Headers

| Header | Description |
|--------|-------------|
| `x-flespi-cid` | Operate as a specific subaccount (by customer ID) |
| `x-flespi-app` | Application name for logging/tracing (max 63 chars) |
| `x-flespi-trace` | Request trace ID for debugging (max 31 chars) |

## Error Codes

| Code | Meaning |
|------|---------|
| 400 | Bad request (check `schema_hint` in response) |
| 401 | Invalid or expired token |
| 403 | Insufficient permissions (token ACL) |
| 404 | Entity not found |
| 429 | Rate limit exceeded |

## Logs

Most entities have `/logs` sub-resource for operation history:
```
GET /gw/devices/123/logs  data={"count":10,"reverse":true}
GET /gw/streams/456/logs  data={"count":10,"reverse":true}
GET /platform/customer/logs  data={"count":10,"reverse":true}
```

## Gateway API Surface (`/gw/`)

Each entity supports standard CRUD via selectors. Sub-resources provide access to related data.

| Entity | Path | Key Sub-resources |
|--------|------|-------------------|
| Channels | `/gw/channels` | `/messages`, `/logs`, `/connections`, `/idents` |
| Devices | `/gw/devices` | `/messages`, `/telemetry`, `/logs`, `/commands-queue`, `/commands-result`, `/settings`, `/connections`, `/media`, `/packets`, `/sms`, `/geofences`, `/calculate` |
| Streams | `/gw/streams` | `/logs`, `/messages`, `/packets`, `/channels/*`, `/devices/*`, `/groups/*`, `/geofences/*` |
| Calculators | `/gw/calcs` | `/logs`, `/devices/*/intervals`, `/devices/*/calculate`, `/devices/*/recalculate`, `/assets/*`, `/groups/*`, `/geofences/*` |
| Plugins | `/gw/plugins` | `/logs`, `/packets`, `/devices/*`, `/groups/*`, `/geofences/*` |
| Geofences | `/gw/geofences` | `/logs`, `/hittest` |
| Groups | `/gw/groups` | `/logs`, `/devices/*`, `/assets/*`, `/geofences/*` |
| Assets | `/gw/assets` | `/logs`, `/intervals` |
| Modems | `/gw/modems` | `/logs` |
| Protocols | `/gw/channel-protocols` | `/device-types`, `/device-types/*/knowledge`, `/device-types/*/assistance` |
| Message params | `/gw/message-parameters` | - |

CRUD pattern: `POST /gw/{entity}` to create, `GET/PUT/PATCH/DELETE /gw/{entity}/{selector}` to read/update/delete.

Sub-resource assignment pattern (streams, calcs, plugins, groups): `POST /gw/{entity}/{id}/{sub}/{sub_ids}` to assign, `DELETE` to unassign, `GET` to list.

### Other API Namespaces

| Namespace | Path | Key Entities |
|-----------|------|--------------|
| Platform | `/platform/` | `customer`, `tokens`, `subaccounts`, `webhooks`, `grants`, `realms`, `logs`, `statistics` |
| Storage | `/storage/` | `containers` (generic message storage), `cdns` (file storage) |
| MQTT | `/mqtt/` | `sessions`, `subscriptions` |
| AI tools | `/ai/` | `tools/search-api-methods`, `tools/get-api-schema`, `tools/search-flespi-documentation`, `tools/generate-flespi-expression`, `logs` |

## API Discovery

The flespi REST API schema is large (76+ endpoints in `/gw/` alone, plus `/platform/`, `/storage/`, `/mqtt/`, `/ai/`). When a flespi token is available, prefer using the `search-api-methods` and `get-api-schema` MCP tools for on-demand endpoint discovery rather than reading the full OpenAPI specs.

### OpenAPI/Swagger Specifications

Static specs for offline reference:
- `https://flespi.io/gw/api.json` - channels, devices, streams, calcs, plugins, etc.
- `https://flespi.io/mqtt/api.json` - MQTT sessions and subscriptions
- `https://flespi.io/platform/api.json` - platform (tokens, subaccounts, webhooks, realms, etc.)
- `https://flespi.io/ai/api.json` - AI tools

### Search Tools (preferred when token is available)

Find endpoints by semantic search:
```
search-api-methods: queries=["list device types", "get device messages"]
```

Get full schema for a specific endpoint (use links from search results):
```
get-api-schema: link="https://flespi.io/docs/#/gw/devices/get_devices_dev_selector_messages"
```

# flespi MQTT Broker

Broker endpoint: `mqtt.flespi.io` (ports 8883 TLS, 443 WSS, 1883 plain TCP).

## Authentication

Use flespi token as MQTT `username`, leave `password` empty.

For ACL tokens on memory-constrained devices: token ID as `username`, first 16 characters of token key as `password`.

Token type determines MQTT permissions - ACL tokens can restrict which topics a client may subscribe to or publish on.

Note: some protocol channels (e.g. Teltonika MQTT, Positioning Universal) act as their own MQTT broker endpoints with separate channel-level authentication. This is unrelated to flespi broker authentication.

## QoS Levels

| QoS | Supported | Behavior |
|-----|-----------|----------|
| 0 | Yes | At most once, no delivery guarantee |
| 1 | Yes | At least once, broker stores and redelivers unacknowledged messages |
| 2 | No | Not supported |

Use QoS 1 with persistent sessions (`clean=false`) for reliable message delivery.

## Topic Structure

All flespi topics follow the pattern: `flespi/{category}/{module}/{entity}/{id}/...`

## Payload

All flespi topics contain valid UTF-8 string: number, boolean, string, JSON

### Categories

| Category | Description | Retained |
|----------|-------------|----------|
| `message` | New data (device messages, chat, container messages) | No |
| `state` | Entity state and telemetry (full config, field values) | Yes |
| `log` | Operation logs (CRUD, connections, errors, commands) | No |
| `interval` | Calculator interval lifecycle (created, updated, deleted) | No |

## Comma Subscriptions

flespi extension: use comma-separated values in a single topic level to match multiple items without separate subscriptions:

```
flespi/state/gw/devices/123,456,789/connected
flespi/message/gw/channels/10,20/+
```

## Shared Subscriptions

Distribute messages across multiple clients for load balancing:

```
$share/group_name/flespi/message/gw/devices/+
```

Messages are distributed randomly among members of `group_name`. Requirements:
- QoS 1 subscriptions (for delivery guarantees)
- Persistent sessions (`clean=false`) recommended to prevent message loss when all subscribers offline

### Sticky Shared Subscriptions

Ensure messages from the same source always go to the same subscriber (useful for local caches/state):

**By topic prefix length**: `$share/name:count/topic`
- `count` = number of topic levels from the beginning used for hash
- `$share/by_ident:6/flespi/message/gw/channels/+/+` - routes by ident (6th level)

**By topic position**: `$share/name:from:count/topic`
- `from` = starting topic level (zero-based), `count` = number of levels for hash
- `$share/by_device:4:1/flespi/+/gw/devices/+/#` - routes by device ID (level 4)

## `cid` User Property

Filter messages by subaccount ID in hierarchical account structures:

**As topic prefix**:
```
$filter/cid=SUBACCOUNT_ID/flespi/message/gw/devices/+
```

**As MQTT v5 User Property** in SUBSCRIBE packet:
- Key: `cid`, Value: subaccount ID

## Message Filtering

Server-side filtering using `$filter/` prefix reduces traffic to the client:

```
$filter/<filter-spec>/original-topic
```

### Filter Types

**By subaccount**:
```
$filter/cid=123/flespi/message/gw/devices/+
```

**By publication time**:
```
$filter/modified_since>1700000000/flespi/message/gw/devices/+
```

**By payload** (flespi expression, URL-encoded):
```
$filter/payload=position.speed%3E60/flespi/message/gw/devices/+
```

**By message object** (access topic, cid, timestamp, user_properties, payload):
```
$filter/message=cid%3D%3D123%26%26tonumber(payload)%3E10/some/topic
```

Filters can be combined with shared subscriptions:
```
$share/workers/$filter/cid=123/flespi/message/gw/devices/+
```

# flespi MCP Tools Guide

How to effectively use flespi MCP tools. Choose the right tool for the task to minimize cost and latency.

## Available Tools

| Tool | Cost | Description |
|------|------|-------------|
| `flespi-api-read` | 0 | Execute GET requests to flespi REST API |
| `flespi-api-write` | 0 | Execute POST/PUT/PATCH/DELETE requests (requires user approval) |
| `search-api-methods` | 0 | Discover API endpoints by semantic search |
| `get-api-schema` | 0 | Get full OpenAPI schema for a specific endpoint |
| `search-flespi-documentation` | 5 | Search knowledge base, blog, API docs for concepts and guidance |
| `search-device-documentation` | 10 | Search device/protocol-specific documentation (hardware specs, commands, wiring) |
| `consult-flespi-account` | 30 | Delegate complex analysis to a platform expert with account access |

## Decision Tree

```
What do you need?
|
+-- Simple data retrieval (list items, check status, get messages)
|   --> flespi-api-read directly
|
+-- Don't know the endpoint path
|   --> search-api-methods, then flespi-api-read
|
+-- Need exact field names or request body format
|   --> get-api-schema (requires link from search-api-methods)
|
+-- Data aggregation or analytics (total mileage, trip count, max speed)
|   --> search-flespi-documentation FIRST (learn about calculate method/calculators)
|   --> then flespi-api-write for calculate, or flespi-api-read for existing intervals
|
+-- Complex configuration (streams, plugins, webhooks, calculators)
|   --> search-flespi-documentation to understand the feature
|   --> get-api-schema for exact field format
|   --> flespi-api-write to create (with user approval)
|
+-- Write flespi expressions (selectors, counters, filters, templates)
|   --> flespi-api-write: POST /ai/tools/generate-flespi-expression {"question": "..."}
|
+-- Device hardware, wiring, firmware, SMS commands
|   --> search-device-documentation (requires protocol_name + device_type_name)
|
+-- Multi-step account analysis or debugging
|   --> consult-flespi-account (expensive, use only when simpler tools won't suffice)
|
+-- API call returned unexpected results or errors
|   --> search-api-methods or get-api-schema to verify endpoint details
|   --> Check schema_hint in error response for valid field names
```

## Tool Details

### flespi-api-read
Execute read-only GET requests. Authenticated automatically with the user's token.

**Parameters:**
- `url` (required): API path with selector and query params
- `data` (optional): JSON body for message/log/interval queries
- `cid` (optional): Customer ID for subaccount access

**Best practices:**
- Always add `?fields=field1,field2` to collection endpoints
- For messages/logs, use `data` with `count`, `fields`, `reverse`
- Use selectors to narrow results: `/gw/devices/123` not `/gw/devices/all`
- To get latest data: `data={"count":1,"reverse":true}`

**Examples:**
```
url: /gw/devices/all?fields=id,name,connected

url: /gw/devices/123/messages
data: {"count":5, "fields":"timestamp,position.speed,position.latitude,position.longitude", "reverse":true}

url: /gw/devices/configuration.ident=352625066004561?fields=id,name

url: /gw/devices/123/telemetry

url: /gw/calcs/456/devices/123/intervals/all
data: {"count":3, "reverse":true}

url: /platform/customer?fields=id,name,limits,counters
```

### flespi-api-write
Execute write operations. **Always confirm with the user before calling.**

**Parameters:**
- `method` (required): POST, PUT, PATCH, or DELETE
- `url` (required): API path
- `data` (optional): Request body (object or array)
- `cid` (optional): Customer ID for subaccount access

**Examples:**
```
method: POST
url: /gw/devices
data: [{"name": "My Tracker", "device_type_id": 42}]

method: PATCH
url: /gw/devices/123
data: {"name": "Renamed Device"}

method: DELETE
url: /gw/devices/123

method: POST
url: /gw/devices/123/calculate
data: {"from":1700000000, "to":1700086400, "selectors":[{"type":"datetime"}], "counters":[{"type":"expression","name":"mileage","method":"summary","expression":"mileage()"}]}
```

### search-api-methods
Find API endpoints by natural language. Returns method links, titles, descriptions, and available selector/response fields.

**Parameters:**
- `queries` (required): Array of 1-3 search strings

**Examples:**
```
queries: ["list device types"]
queries: ["get device messages", "device telemetry"]
queries: ["calculate method for devices"]
```

Use the returned `link` field with `get-api-schema` for full details.

### get-api-schema
Get full OpenAPI schema for a specific endpoint. Use links from `search-api-methods`.

**Parameters:**
- `link` (required): ApiBox documentation URL starting with `https://flespi.io/docs/#/`

### search-flespi-documentation
Search knowledge base for platform concepts, configuration guidance, and best practices.

**Parameters:**
- `question` (required): Natural language question

**When to use:**
- Before any data aggregation task (calculators, calculate method)
- For unfamiliar features (streams, plugins, webhooks, geofences)
- For expression syntax and patterns
- For MQTT topics and integration patterns
- For troubleshooting guidance

### search-device-documentation
Search device manufacturer documentation and protocol specifications.

**Parameters:**
- `protocol_name` (required): Protocol name (e.g. `teltonika`, `queclink`)
- `device_type_name` (required): Device type in lowercase (e.g. `fmb120`, `gl300`)
- `question` (required): Question about the device

To find valid protocol/device names:
```
flespi-api-read: url=/gw/channel-protocols/{protocol}/device-types/all?fields=title,device_type_name
```

Not all protocols have device documentation (e.g. software protocols like wialon-ips, mqtt).

### consult-flespi-account
Delegate complex questions to a platform expert that can read the user's account data.

**Parameters:**
- `question` (required): Detailed question

**Use sparingly (30 credits).** Only when:
- Multiple API calls and cross-referencing are needed
- Deep platform expertise is required for analysis
- Simpler tools cannot answer the question

**Do NOT use for:**
- Simple data retrieval (use flespi-api-read)
- Documentation questions (use search-flespi-documentation)
- Listing or status checks

## Common Patterns

### Check device connectivity
```
flespi-api-read: url=/gw/devices/{id}/telemetry
--> Look at server.timestamp to see when device last reported
```

### Find device by IMEI
```
flespi-api-read: url=/gw/devices/configuration.ident=IMEI_VALUE?fields=id,name,connected
```

### Get recent messages with position
```
flespi-api-read: url=/gw/devices/{id}/messages
  data={"count":10,"reverse":true,"fields":"timestamp,position.latitude,position.longitude,position.speed"}
```

### Check stream health
```
flespi-api-read: url=/gw/streams/{id}?fields=id,name,enabled
flespi-api-read: url=/gw/streams/{id}/logs  data={"count":10,"reverse":true}
```

### Get calculator intervals
```
flespi-api-read: url=/gw/calcs/{calc_id}/devices/{device_id}/intervals/all
  data={"count":10,"reverse":true}
```

### Check account limits
```
flespi-api-read: url=/platform/customer?fields=limits,counters
```

### Create a device
```
flespi-api-write: method=POST url=/gw/devices
  data=[{"name":"My Tracker","device_type_id":42,"configuration":{"ident":"352625066004561"}}]
```
To find device_type_id: `flespi-api-read: url=/gw/channel-protocols/all?fields=id,title`

### Send a command to device
```
flespi-api-write: method=POST url=/gw/devices/123/commands
  data=[{"name":"getinfo","properties":{}}]
```
Check available commands: `flespi-api-read: url=/gw/channel-protocols/{protocol_id}/device-types/{type_id}?fields=commands`

### Generate a flespi expression
```
flespi-api-write: method=POST url=/ai/tools/generate-flespi-expression
  data={"question":"Filter devices with speed greater than 80 km/h"}
```
Use for: calculator selectors/counters, webhook templates, plugin expressions, REST API filters, stream filters.

### On-demand analytics (calculate method)
```
flespi-api-write: method=POST url=/gw/devices/123/calculate
  data={"from":1700000000,"to":1700086400,"selectors":[{"type":"datetime"}],"counters":[{"type":"expression","name":"mileage","method":"summary","expression":"mileage()"}]}
```

## Key Principles

1. **Prefer server-side operations** - use API selectors, expressions, `?fields=`, and the calculate method instead of fetching everything and filtering locally
2. **Search docs before aggregation** - flespi has powerful server-side analytics; learn them before iterating client-side
3. **Minimize response size** - always use `?fields=` for collections and `count`+`fields` for messages
4. **Escalate gradually** - start with free tools (api-read, search-api-methods), use paid tools only when needed
5. **Never write without approval** - always present planned mutations to the user first