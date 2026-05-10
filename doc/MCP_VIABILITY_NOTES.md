# Freeciv MCP viability notes

## Scope

This document records an initial viability assessment for replacing the native Freeciv client with an MCP-facing client layer, without implementing the MCP yet.

## Short conclusion

The project looks viable.

The strongest reason is that Freeciv is already organized as a server-authoritative client/server game, and the existing client is intentionally "quite dumb". That means an MCP layer does not need to reproduce core game rules locally; it mostly needs to speak the network protocol, maintain enough mirrored state to reason about the game, and expose that state and the allowed requests as MCP tools/resources.

## Main findings

### 1. The architecture already supports a thin client

- `doc/HACKING` explicitly states that Freeciv is a client/server game and that "almost all calculations are performed on the server".
- The same document says the client is "quite dumb", which is exactly the kind of architecture that makes a protocol-driven MCP replacement realistic.

Implication: the MCP side can focus on transport, state tracking, and request orchestration rather than re-implementing simulation logic.

### 2. The real action surface is broader than `doc/README.actions`

`doc/README.actions` is important, but it is not the full definition of what an MCP client would need.

It documents:

- ruleset-facing action concepts
- enablers and hard requirements
- action names and Lua integration

But the actual interactive surface for a client replacement also includes:

- login/authentication
- pregame selection
- city management
- unit orders and unit actions
- diplomacy
- research/government/rates
- reports and chat/server commands
- editor / scenario packets in privileged modes

The canonical protocol surface is `common/networking/packets.def`, not the docs alone.

### 3. The protocol is centrally defined and code-generated

The most important protocol files are:

- `common/networking/packets.def`
- `common/generate_packets.py`
- `doc/README.delta`
- `doc/HACKING`

`packets.def` defines packet ids, direction (`cs` / `sc`), fields, and flags. `generate_packets.py` produces client/server/common packet code from that definition.

Implication: future MCP work should treat `packets.def` as the primary machine-readable contract.

### 4. There is already an optional JSON protocol

The repository already has a JSON transport mode:

- build option: `meson_options.txt` → `json-protocol`
- build wiring: `meson.build`
- JSON send/receive code: `common/networking/packets_json.[ch]`
- JSON field I/O: `common/networking/dataio_json.c`

Important details:

- JSON mode is optional and disabled by default.
- The first packet can cause the server to switch a connection into JSON mode.
- JSON packets include a `pid` field for packet type.
- The JSON path still uses the same generated packet definitions.

Implication: if enabled in the build, JSON is the most promising path for a Python FastMCP implementation. It avoids reimplementing the raw binary protocol and delta/compression details in Python.

### 5. Login and session establishment are straightforward but mandatory

The initial handshake is easy to locate:

- client send: `client/clinet.c`
- packet definitions: `PACKET_SERVER_JOIN_REQ`, `PACKET_SERVER_JOIN_REPLY`, `PACKET_AUTHENTICATION_REQ`, `PACKET_AUTHENTICATION_REPLY` in `common/networking/packets.def`
- server validation: `server/connecthand.c`

Implication: an MCP client can connect directly to the server, but it must participate in the same version/capability/authentication flow as a normal client.

### 6. Unit actions are discoverable and server-validated

The most important action flow is explicit in the protocol:

- ask what actions are possible: `PACKET_UNIT_GET_ACTIONS`
- ask for extra action parameters/costs: `PACKET_UNIT_ACTION_QUERY`
- execute action: `PACKET_UNIT_DO_ACTION`
- receive answers/probabilities: `PACKET_UNIT_ACTIONS`, `PACKET_UNIT_ACTION_ANSWER`

Relevant handlers:

- client requests: `client/control.c`, `client/packhand.c`
- server handling: `server/unithand.c`

Implication: for unit actions, the MCP should prefer a two-step model:

1. ask the server what is possible
2. expose only the legal or near-legal actions to the LLM

That is safer than hardcoding legality rules in Python.

### 7. Request lifecycle tracking matters

Freeciv frames request processing with:

- `PACKET_PROCESSING_STARTED`
- `PACKET_PROCESSING_FINISHED`

The client uses request ids and waits for processing completion in:

- `client/clinet.c`
- `common/networking/packets.c`
- `common/networking/connection.h`

Implication: an MCP adapter should preserve this concept. A tool call should usually wait until the related server processing window completes before returning a final result.

### 8. Server commands are also part of the reachable surface

Some operations are issued as chat/server commands rather than dedicated gameplay packets:

- `PACKET_CHAT_MSG_REQ`
- client helpers in `client/chatline_common.c`
- server command handling in `server/handchat.c` and `server/stdinhand.c`

Implication: the MCP surface may need two lanes:

- structured gameplay tools backed by dedicated packets
- controlled administrative/chat-command tools for setup and server management

### 9. There is precedent for non-native clients

Two existing precedents reduce risk:

- `client/gui-stub/` shows that the client core is already separable from a real GUI
- `FREECIV_WEB` / freeciv-web specific packets show the ecosystem already supports alternate client forms

Implication: building a new machine-facing client layer is aligned with the codebase direction.

## What seems most viable

### Best near-term path

Build or run a server/client configuration with JSON protocol enabled, then implement a Python FastMCP transport and state mirror on top of that protocol.

Why this is the best path:

- lower implementation complexity than binary packet support
- protocol still anchored in `packets.def`
- easier debugging and observability
- easier mapping from packet fields to MCP resources/tools

### Lower-priority alternative

Reuse or wrap part of the existing C client core and expose it to Python.

This is possible, but it is likely heavier than a JSON-based Python client and would create a more complex mixed-language architecture early.

## Main risks

1. **JSON protocol is optional**
   - It is not enabled by default, so the deployment/build story must be decided early.

2. **Protocol compatibility is version/capability-sensitive**
   - Login verifies capability strings and version compatibility.

3. **The MCP cannot rely on docs alone**
   - `README.actions` is necessary but insufficient as a complete action inventory.

4. **Client state mirroring is still substantial**
   - Even with a dumb client, the MCP side must maintain enough world/player/unit/city state to ask good questions and present useful tools.

5. **Not all interactions are "actions"**
   - A useful MCP will need broader gameplay/session tooling than only unit action execution.

## Recommended phased plan

### Phase 1 — Lock down the protocol strategy

- Confirm whether JSON protocol will be required for the MCP path.
- Confirm whether the target runtime controls the Freeciv build configuration.
- Treat `packets.def` as the source of truth for request/response inventory.

### Phase 2 — Define the minimum playable MCP slice

Start with a narrow vertical slice instead of full feature parity:

- connect/authenticate
- choose/take player
- observe map/unit/city state
- get unit actions for a target
- perform a small set of safe unit and city actions
- end turn

### Phase 3 — Build a protocol/state catalog

Create a machine-usable mapping from:

- packet type
- direction
- handler file
- gameplay concept
- MCP exposure candidate

Use `packets.def` plus server handlers as the primary sources.

### Phase 4 — Design the MCP around server truth

The MCP should expose:

- **resources** for read-mostly world/player/city/unit state
- **tools** for state-changing requests
- **guarded tools** that first query the server for legality where possible

### Phase 5 — Add orchestration semantics

Model each tool call around:

1. send request
2. collect resulting packets until processing is finished
3. update mirrored state
4. return structured result to the LLM

### Phase 6 — Expand by packet category

Suggested order:

1. session / pregame
2. turn control
3. unit actions
4. city management
5. diplomacy
6. research / government / economy
7. reports / admin / scenario tools

## Suggested next investigation targets

1. Enumerate the minimum packet set needed for a playable single-player or observer flow.
2. Inspect which server-to-client packets are essential for maintaining a useful mirrored world state.
3. Validate whether JSON mode covers all packet field types exercised by the intended gameplay slice.
4. Decide whether server commands sent through chat should be first-class MCP tools or stay out of scope initially.

## Key files

- `doc/HACKING`
- `doc/README.actions`
- `doc/README.delta`
- `common/networking/packets.def`
- `common/generate_packets.py`
- `common/networking/packets_json.c`
- `common/networking/packets_json.h`
- `common/networking/dataio_json.c`
- `common/networking/packets.c`
- `common/networking/connection.h`
- `client/clinet.c`
- `client/control.c`
- `client/packhand.c`
- `server/connecthand.c`
- `server/unithand.c`
- `server/handchat.c`
- `server/stdinhand.c`

