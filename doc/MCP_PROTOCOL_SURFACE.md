# Freeciv MCP protocol surface inventory

## Purpose

This is a compact inventory of the protocol areas that matter most if a future MCP server replaces the normal Freeciv client.

## Primary source of truth

Use `common/networking/packets.def` first.

It defines packet ids, packet direction, fields, and protocol flags. Generated code is produced by `common/generate_packets.py`.

## Packet volume snapshot

From `packets.def`:

- roughly **76** packet definitions are client-to-server (`cs`)
- roughly **134** packet definitions are server-to-client (`sc`)

This confirms that the MCP problem is larger than "unit actions only".

## Core protocol areas

### 1. Session establishment

Key packets:

- `PACKET_SERVER_JOIN_REQ`
- `PACKET_SERVER_JOIN_REPLY`
- `PACKET_AUTHENTICATION_REQ`
- `PACKET_AUTHENTICATION_REPLY`

Key files:

- `client/clinet.c`
- `server/connecthand.c`
- `common/networking/packets.def`

### 2. Request lifecycle

Key packets:

- `PACKET_PROCESSING_STARTED`
- `PACKET_PROCESSING_FINISHED`
- `PACKET_FREEZE_HINT`
- `PACKET_THAW_HINT`

Key files:

- `doc/HACKING`
- `client/clinet.c`
- `common/networking/packets.c`
- `common/networking/connection.h`

Why it matters:

- lets the client correlate updates with a request
- defines when an MCP tool call should consider the server-side operation complete

### 3. Pregame / player selection

Key packets:

- `PACKET_NATION_SELECT_REQ`
- `PACKET_PLAYER_READY`
- `PACKET_SINGLE_WANT_HACK_REQ` (special/privileged flow)

Key files:

- `common/networking/packets.def`
- `server/connecthand.c`
- `doc/HACKING`

### 4. Unit control and unit actions

Key packets:

- `PACKET_UNIT_ORDERS`
- `PACKET_UNIT_CHANGE_ACTIVITY`
- `PACKET_UNIT_GET_ACTIONS`
- `PACKET_UNIT_ACTIONS`
- `PACKET_UNIT_ACTION_QUERY`
- `PACKET_UNIT_ACTION_ANSWER`
- `PACKET_UNIT_DO_ACTION`
- `PACKET_UNIT_TYPE_UPGRADE`
- `PACKET_UNIT_SERVER_SIDE_AGENT_SET`

Key files:

- `client/control.c`
- `client/packhand.c`
- `server/unithand.c`
- `doc/README.actions`

Notes:

- `README.actions` explains ruleset and action semantics.
- `server/unithand.c` is the operational truth for legality checks and action execution.
- `PACKET_UNIT_GET_ACTIONS` is especially important for an LLM-safe tool design.

### 5. City management

Representative packets:

- `PACKET_CITY_SELL`
- `PACKET_CITY_BUY`
- `PACKET_CITY_CHANGE`
- `PACKET_CITY_WORKLIST`
- `PACKET_CITY_MAKE_SPECIALIST`
- `PACKET_CITY_MAKE_WORKER`
- `PACKET_CITY_CHANGE_SPECIALIST`
- `PACKET_CITY_RENAME`
- `PACKET_CITY_OPTIONS_REQ`
- `PACKET_CITY_REFRESH`
- `PACKET_CITY_RALLY_POINT`
- `PACKET_WORKER_TASK`

Why it matters:

- city operations are a major share of meaningful gameplay
- some of these are not "actions" in the `README.actions` sense

### 6. Player-level empire controls

Representative packets:

- `PACKET_PLAYER_PHASE_DONE`
- `PACKET_PLAYER_RATES`
- `PACKET_PLAYER_CHANGE_GOVERNMENT`
- `PACKET_PLAYER_RESEARCH`
- `PACKET_PLAYER_TECH_GOAL`
- `PACKET_PLAYER_MULTIPLIER`

Why it matters:

- these map naturally to high-level MCP tools
- they are central to turn-based play even though they are not unit actions

### 7. Diplomacy

Representative packets:

- `PACKET_DIPLOMACY_INIT_MEETING_REQ`
- `PACKET_DIPLOMACY_CANCEL_MEETING_REQ`
- `PACKET_DIPLOMACY_CREATE_CLAUSE_REQ`
- `PACKET_DIPLOMACY_REMOVE_CLAUSE_REQ`
- `PACKET_DIPLOMACY_ACCEPT_TREATY_REQ`
- `PACKET_DIPLOMACY_CANCEL_PACT`

Why it matters:

- diplomacy is a separate tool family with its own state and request flow

### 8. Reports / queries

Representative packets:

- `PACKET_REPORT_REQ`

Why it matters:

- useful for read-oriented MCP tools and summaries

### 9. Chat and server command path

Representative packets:

- `PACKET_CHAT_MSG_REQ`

Key files:

- `client/chatline_common.c`
- `server/handchat.c`
- `server/stdinhand.c`

Why it matters:

- some setup/admin flows are issued through chat commands rather than dedicated structured packets

### 10. Spaceship / scenario / editor / privileged flows

Representative packets:

- `PACKET_SPACESHIP_LAUNCH`
- `PACKET_SPACESHIP_PLACE`
- `PACKET_RULESET_SELECT`
- `PACKET_SAVE_SCENARIO`
- `PACKET_EDIT_*`

Why it matters:

- probably out of scope for the first playable MCP slice
- should still be tracked so they are not mistaken for missing protocol support later

## Transport options

### Preferred

Use the optional JSON protocol when available.

Relevant files:

- `meson_options.txt`
- `meson.build`
- `common/networking/packets_json.[ch]`
- `common/networking/dataio_json.c`

### Fallback

Implement or reuse raw packet support only if JSON is unavailable or incomplete for the chosen gameplay slice.

## Recommended first MCP tool families

1. connection/session tools
2. turn/state inspection tools
3. unit action discovery tools
4. unit action execution tools
5. city management tools
6. research/government/economy tools
7. minimal diplomacy tools

## Most useful code paths for future analysis

- `server/unithand.c`
- `server/cityhand.c`
- `server/plrhand.c`
- `server/diplhand.c`
- `client/control.c`
- `client/packhand.c`
- `common/networking/packets.def`
- `common/generate_packets.py`

