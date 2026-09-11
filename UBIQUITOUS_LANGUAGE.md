# Ubiquitous Language

Canonical vocabulary for product, engineering, and customer communication. One term = one concept. Synonyms listed below must be renamed wherever they appear.

**This file** = internal, project-scoped vocabulary: coined terms, shorthand, product concepts, workflow names.

**Domain MCP server** (`mcp__domain__*` tools) = external, standards-scoped reference knowledge: Modbus, SNMPv3, Redfish, IEC 61850, BESS, power economics. Query it when domain specifics matter — protocol register layouts, standard field names, IEC logical node definitions, etc. It is the lookup surface, not the place to invent app-specific names.

---

## Hardware

| Term | Definition | Aliases to avoid |
|---|---|---|
| **module** | A physical 10ft ISO container *and* the informational domain concept it represents — the two are the same entity in this system | container, unit, box, sub-system |
| **equipment** | A piece of gear installed inside a module (cells, BMS, GPU server, chiller, breaker, racks, sensors, etc.). Equipment may itself contain further equipment — depth is unbounded. | sub-module, component, part, equipment unit, sub-component |

Three module types: `compute_module`, `grid_module` (arcnode-fabricated containers), and `bess_module` (BYO — customer's Tesla Megapack, Tesla Megablock, or CATL EnerOne). Arcnode does **not** fabricate a BESS module. There is no `thermal_module` — dry coolers, chillers, and other site cooling infrastructure are not first-class EMS-monitored modules; their telemetry, when collected, is projected through `compute_module` equipment-tier devices (DLC sensors, VFD readings).

**Grid-forming PCS location is conditional on `bess_coupling`, not a fixed template.** Tesla Megapack / Megablock (`ac_coupled`) carry their own grid-forming inverter internally — the PCS lives inside `bess_module`, and `grid_module` is switchgear + metering only for that coupling. CATL EnerOne (`dc_external_pcs`) does not grid-form itself — it pairs with arcnode's own `GRD-PCS-001` (an EPC PD500), which lives inside `grid_module` and does the grid-forming. Don't assume either module always owns the PCS; check `bess_coupling` first.

## Geometry

| Term | Definition | Aliases to avoid |
|---|---|---|
| **equipment envelope** | The simplified solid bounding the outermost surfaces of a piece of equipment. Source: vendor STEP, reduced by the envelope extraction pipeline. Used for: collision detection, container fit verification, mass property calculation. | bounding box, footprint |
| **service envelope** | The directional clearance volume around equipment required for installation, operation, and maintenance. Always per-face (front, rear, top, sides, bottom) — access requirements are not isotropic. Source: vendor datasheet installation section, extracted into `equipment_spec`. Used for: clearance verification, installation sequencing, arc-flash boundary compliance, crane lift planning. | clearance zone, maintenance clearance |

Equipment envelopes check whether things *fit*. Service envelopes check whether installed things can be *worked on*. Both are checked independently in the volumetric verification step; both must pass. A layout satisfying equipment envelopes but violating service envelopes is a maintenance trap.

Pitfalls: (1) merging both into a single bounding box loses directionality — a 36" front and 6" rear clearance cannot be averaged; (2) some service clearances are regulatory (NEC arc-flash, NFPA egress) and non-negotiable; (3) installation-time clearances (crane, rigging) may exceed the operational service envelope — captured separately when relevant.

## Templates

| Term | Definition | Aliases to avoid |
|---|---|---|
| **template** | A versioned YAML definition that declares the canonical measurement/command vocabulary, protocol-binding scaffold, and standards metadata for one type of module or equipment | class, device type, device model, profile (collides with IEC 61850 conformance profiles) |
| **module template** | A template whose definition lives in `edp-api/device_templates/modules/`; always has a `contains:` block | — |
| **equipment template** | A template whose definition lives in `edp-api/device_templates/equipment/` | component template, component class |
| **line template** | A template for cross-cutting infrastructure not bound to a container (`dlr_sensor`, `phase_shift_transformer`, `industrial_gateway`) | — |

Why "template" instead of "class": *class* is OOP jargon and doesn't translate to operators or industrial integrators. *Template* is the standard SCADA/HMI term for "a definition you instantiate." OPC UA's *ObjectType*, IEC 61850's *Logical Node Type*, BACnet's *ObjectType*, and Sparkplug's *Device Definition* all model the same concept.

**Templates encode ARCNODE-engineered hardware *and* ARCNODE-authored integrations.** Compute and Grid containers are arcnode-built end-to-end (GPU servers, NVLink switches, DLC pumps + plate heat exchangers, breakers; AC switchgear, transformer, metering relays, and — *conditionally* — a PCS). BESS templates encode supported third-party gear (Tesla Megapack, Tesla Megablock, CATL EnerOne) — vendor protocol surfaces, register maps, and standards metadata, authored by the ARCNODE team. Templates are opinionated, versioned, PR-gated definitions; codegen and HMI views are designed against the specific shapes templates declare. They are not vendor-agnostic placeholders or user extensibility points. Per-deployment one-off measurements use the DTM's `extra_measurements:` escape hatch, not template-authoring.

**PCS provenance is conditional on `bess_coupling`, not a fixed Grid-container component.** Three couplings carry a PCS (a fourth, `none`, has no BESS and can't island):
- `ac_coupled` (Tesla Megapack, Tesla Megablock): PCS is integrated in the BESS. Lives in `bess_module`; `grid_module` for these deployments is AC switchgear + metering only. Grid-forming confirmed (`EXT-BESS-001/spec.yaml`).
- `dc_external_pcs` (CATL EnerOne, DC output): the BESS does not grid-form itself — it pairs with ARCNODE's own `GRD-PCS-001` inside `grid_module`, which grid-forms. Confirmed via `GRD-PCS-001/spec.yaml`.
- `dc_integrated_pcs` (a distinct CATL EnerOne **AC**-output SKU, PCS integrated in the BESS pad itself): **undocumented.** No `equipment/` spec exists for this SKU yet — grid-forming capability unconfirmed, `reviewed_by: TBD_pending_human_review` same as other in-progress specs. Do not assume it grid-forms until a sourced spec lands.

`bess_coupling: ac_coupled | dc_integrated_pcs | dc_external_pcs | none` on the deployment payload determines which shape applies — the `grid_module` template's `contains:` block is conditional on the selected `bess_coupling`, not a single fixed list.

Templates appear on engineering surfaces (YAML files, code, the `template:` field on every DTM device) and may appear in commissioning UIs. They do not appear in HMI runtime labels, customer-facing docs, or sales material — those use **display name**.

## Devices & Topology

| Term | Definition | Aliases to avoid |
|---|---|---|
| **device** | An instance of a template within a site; identified by `device_id` (snake_case slug, immutable) | node, asset, entity |
| **module-tier device** | A device whose template is a module template; sits at the root of the device tree (`parent` is null) | — |
| **equipment-tier device** | A device whose template is an equipment template; has `parent: <device_id>` pointing to a containing device. The containing device may itself be equipment-tier — depth is unbounded | sub-device, child device |
| **DTM** | Device Topology Manifest — `/etc/ems/dtm.json`; the authoritative per-deployment record of every device instance, its template, its display-name overrides, the site's electrical bus topology (`buses[]`), and a `templates_used` map embedding every template the `devices` block references | topology file, device manifest |
| **bus** | A named electrical node (DC or AC) to which module-tier devices connect. Declared in the DTM `buses[]` block. Not a device — it has no MQTT topics and no template. Corresponds to IEC 61850 `ConnectivityNode`. | bus bar, electrical node, connectivity node |
| **SLD** | Single Line Diagram — served by the EMS from the EDP SVG artifact; loaded by `ems-hmi` at `/modules/sld`. Live MQTT measurements are overlaid on the SVG by binding to element IDs that match `device_id` slugs. Bus connection states are driven by `buses[]` from the DTM. Not a hand-drawn per-deployment artifact — the SVG is generated by `edp-api` from the sizing payload. | one-line diagram, topology diagram |
| **deployment** | One running EMS stack: one CFN stack or one ISO image, one HiveMQ broker, one DTM, one `deployment_uuid` | install, instance, stack |
| **site** | A physical location within a deployment; identified by `site_id` in MQTT topics. MVP: one deployment = one site | location, plant, facility |

MVP note: each deployment contains exactly one site. The `sites/{site_id}/` topic prefix is kept for forward-compatibility with future multi-site deployments.

## MQTT Messaging

| Term | Definition | Aliases to avoid |
|---|---|---|
| **measurement** | What a device can emit — a named, typed data channel with a fixed unit (e.g. `voltage_dc`, `conductor_temp`). Static identifier; lives in the topic | tag, metric, point, reading, state |
| **sample** | One timestamped value on a measurement channel — payload `{ts, value}` where `ts` is RFC3339 / ISO8601 with `Z` suffix. All timestamps system-wide are UTC; no local-time conversion is applied at any layer. | reading, data point, observation |
| **command** | An imperative sent *to* a device. Verb is enum-locked: `set`, `reset`, `clear`, `start`, `stop`, `enable`, `disable` | action, instruction, setpoint (use only for the value, not the message) |
| **target** | The thing a command operates on, e.g. `active_power` in `set/active_power/watts` | subject, object |
| **unit** | Engineering unit carried in the topic slug, locked to the vocabulary in ADR-002 (e.g. `volts`, `amps`, `percent`, `none`) | uom |
| **family** | Slot 4 of every device-scoped topic: `measurements`, `commands`, or `system`; used to dispatch to the correct parser | message_kind, topic type |
| **status** | A regular measurement that every device publishes to indicate communication health; used for time-joined quality lookups | comm_state, health beat, heartbeat, quality flag |

### Topic Structure

```
sites/{site_id}/devices/{device_id}/measurements/{measurement}/{unit}   # 6 segments
sites/{site_id}/devices/{device_id}/commands/{verb}/{target}/{unit}     # 7 segments
system/{event_type}                                                      # 2 segments
```

## Measurement Schema

| Term | Definition | Aliases to avoid |
|---|---|---|
| **severity** | A per-value key on `enum` measurements in the template YAML: `ok \| warn \| alarm`. Flows through to the AsyncAPI spec as `x-severity`. Drives color in HMI `Mode` components. Absent on pure mode enums (e.g. `control_mode`) — absence means the label carries all meaning, not color. | alarm level, priority |
| **register_value** | The integer that a non-native-protocol device (Modbus, DNP3, CAN) places on the wire for a given enum value. The industrial gateway translates this to the string label before publishing to MQTT. The HMI never sees `register_value`. Not needed for `mqtt_native` or `redfish` bindings. | raw value, wire value |
| **synthetic measurement** | A measurement whose value is computed locally by the industrial gateway from other measurements already on the MQTT bus, rather than scraped from a south-side device. Declared in templates via `binding.protocol: synthetic` with a `formula` (`subtract`, `sum`, `mean`, etc.) and `inputs[]` referencing source topics. Attribution: `publisher: gateway`. Example: module-level `import_headroom = envelope.import_limit − bess_module.active_power`. Industry-standard SCADA term (Ignition, OSI PI, AVEVA all use "synthetic tag"). | derived measurement, calculated tag, virtual point, computed channel, rollup |
| **LOTO** | Lockout/Tagout — a per-device state in the EMS with three enforcement properties: (1) command inhibit: the EMS refuses to dispatch any command to the device while LOTO is active; (2) state visibility: the HMI surfaces the LOTO state on the SLD node (padlock icon), module card, and module detail header; (3) audit log: every set and clear is recorded with username, device ID, and UTC timestamp for OSHA compliance and defense/sovereign audit trails. The physical LOTO procedure (physical lock, written tag) is a field activity — the EMS tracks and enforces the software state but does not manage the physical procedure. | lockout, maintenance mode, soft disable |

## HMI Display Primitives

The five components that render measurement samples in `ems-hmi`. Each maps to one `type` in the template YAML. These terms are used by designer, PM, and engineer when discussing what the HMI shows — not what the wire carries.

| Term | Measurement type | What it renders | Aliases to avoid |
|---|---|---|---|
| **Reading** | `float` | Number + unit. `"842.3 kW"`. No data: `"—"`. Note: lowercase "reading" is an alias to avoid for `sample` in MQTT contexts — `Reading` (capital R) is the HMI component name only. | value, display, numeric |
| **Indicator** | `bool` | Colored dot only. Green = ok, red = fault, gray = offline. No label. Color is the entire signal. | status light, LED, dot |
| **Mode** | `enum` | Colored dot + humanized string label. `"● MANUAL"`. Dot color from `x-severity` if present; neutral gray if not. | state indicator, multistate indicator, StateIndicator |
| **Gauge** | bounded `float` | Circular arc showing position in operating range. Props: `min`, `max`, `thresholds[]`. | dial, meter, circular indicator |
| **RangeIndicator** | bounded `float` | Horizontal bar showing position in operating range. Same semantics as `Gauge`, used where vertical space is limited. | progress bar, bar indicator |

## Naming

| Term | Definition | Aliases to avoid |
|---|---|---|
| **canonical name** | Snake_case slug used in topics, code, and the AsyncAPI spec; immutable once a device is provisioned | internal name, system name |
| **display name** | Customer-facing mutable label; lives in DTM under `display_name`, propagated to HMI via `x-display-name` | friendly name, label (ok informally) |

Resolution order: DTM override → template default → humanized canonical.

## Deployment & Configuration

| Term | Definition | Aliases to avoid |
|---|---|---|
| **stage** | The deploy environment a service runs in. Picked by the `ENV` env var. Values: `local` (dev box, mocks everything), `demo` (self-hosted demo, bundled seed), `beta` (every AWS deploy — our smokes AND customer prod, same code path; distinguished by AWS account + DTM contents + per-deploy creds). | env, environment, profile, tier |
| **cfg.defaults.yml** | Per-service YAML baked into the image. One block per stage. The file the loader reads first; pydantic validates after any customer merge. | base config, baseline config, cfg.yml |
| **cfg.customer.yml** | Per-deploy YAML written by platform-api UserData from the `ConfiguratorPayload`. Mounted into the service container; deep-merged over the matching stage block in `cfg.defaults.yml` before validation. | customer overlay, customer override, runtime config |
| **CFG_CUSTOMER_PATH** | Env var holding the path to `cfg.customer.yml`. The contract; the value is platform-supplied per service. | cfg path, overlay path |
| **commissioning POST** | The `POST /topology` call a customer makes after deploy to fill in real device connection info (IP/port/unit_id). Until then, devices render `--` in HMI. | go-live, flip to live, activation |
| **industrial fixtures** | Mock protocol servers (modbus / snmp / redfish / dnp3 / bacnet) shipped from `ems-industrial-fixtures`. Deployed in compose; the `industrial-fixtures.json` DTM points devices at them to exercise the full path through the device boundary. | mocks, sim devices |

Pitfalls: (1) inventing a separate "staging" or "prod" stage — every AWS deploy uses `ENV=beta`; what differs is the AWS account, DTM contents, and per-deploy creds (not a stage); (2) putting per-deploy values in `config.env` instead of `cfg.customer.yml` (the cardinal rule: secrets stay env, ENV stays env, everything else is YAML); (3) hardcoding paths the customer might re-target — use `CFG_CUSTOMER_PATH`, not literal file paths.

## Specs & Standards

| Term | Definition | Aliases to avoid |
|---|---|---|
| **AsyncAPI spec** | The AsyncAPI v3 document served by `ems-device-api` at `GET /asyncapi`; generated from the persisted DTM (which embeds its own `templates_used` map); single source of truth for all topic shapes, payload schemas, and protocol bindings | "the spec", API spec, topic spec |
| **IEC 61850** | The grid-equipment communication standard; DTM is a strict superset of IEC 61850 SCD. Structural SCD enforcement deferred to v1.x; `iec_61850` metadata blocks present in template YAML as annotation. Query the domain MCP server for IEC 61850 logical node definitions. | — |
| **SCD** | Substation Configuration Description — the IEC 61850 XML export format that DTM supersedes for Arcnode devices | — |

---

## Relationships

- A **deployment** contains one or more **sites** (MVP: exactly one site per deployment)
- A **site** contains one or more **devices**
- A **device** is an instance of a **template**; its canonical identity is `{site_id}/{device_id}`
- A **module-tier device** may contain one or more **equipment-tier devices** (each with `parent: <ancestor_device_id>`); depth is unbounded
- A **template** declares the **measurement** and **command** vocabulary for all devices of that type
- A **DTM** declares every **device** instance for a **deployment**, references templates by `${name}.${version}` slug, and embeds the full template definitions in `templates_used` so the payload is self-describing. It also carries a `buses[]` block encoding the site's electrical topology.
- A **bus** connects two or more **module-tier devices** at the same voltage domain. A device that bridges domains (e.g. `grid_module` on both AC and DC) appears in multiple `buses[]` entries, distinguished by an optional `port` label.
- The **AsyncAPI spec** is stamped out from the persisted DTM (which already contains its own `templates_used`)
- A **measurement** identifies a channel (static); a **sample** is one value on that channel (dynamic)
- **status** is a **measurement** — it participates in the same topic structure and payload schema as any other measurement

---

## Example Dialogue

> **Dev:** "When a new compute module is installed, do I add a new device or a new template?"

> **Domain expert:** "New device — you instantiate the existing `compute_module.v1` template. A new template is only needed if this module type has different measurements or commands than anything we've built before."

> **Dev:** "And the display name the customer sees — is that in the template or the DTM?"

> **Domain expert:** "DTM. The template holds the canonical name, like `gpu_core_temp`. The DTM holds the display name override, like 'Rack 3 GPU Temp'. The HMI renders the DTM value; the topic always uses the canonical name."

> **Dev:** "So if the customer renames it, the topic doesn't change?"

> **Domain expert:** "Exactly. Topics are immutable once provisioned. Only display names move."

> **Dev:** "What goes into the AsyncAPI spec — the template or the DTM or both?"

> **Domain expert:** "`ems-device-api` stamps it out from the DTM alone — the DTM already embeds its own `templates_used` map, so device-api doesn't need to read the template directory. It receives, persists, projects."

---

## Flagged Ambiguities

- **"Spec" (generic)** — used loosely across docs and conversation to mean the AsyncAPI spec, the JSON Schema part, or even the IEC 61850 SCD. Always say **AsyncAPI spec** when referring to the `GET /asyncapi` document. Use **JSON Schema** when discussing payload validation schemas within it.

- **"Module" (physical vs informational)** — previously unclear whether "module" referred to the physical container or the software concept. Resolution: they are the **same entity**. A module is simultaneously the 10ft ISO container and the domain object. No split needed. Note: "module" also appears as a software concept in NestJS and Rust — context disambiguates; in domain discussions always means hardware module.

- **"Site" implies multi-site** — the `sites/{site_id}/` topic prefix implies multiple sites per deployment, but MVP is 1:1 (one deployment = one site). The prefix is retained for forward-compatibility. Docs that describe "a site" at MVP mean "the single site in that deployment."

- **"Device" (multi-tier)** — used for both module-tier and equipment-tier instances. Both are correct uses of "device"; distinguish with **module-tier device** / **equipment-tier device** when tier matters. Equipment-tier devices may contain further equipment-tier devices (an `ac_switchgear` containing `breaker`s) — depth is unbounded.

- **"State"** — appeared as a draft third-family name. Not a distinct concept. Anything a device emits — including alarm state, tap position, communication health — is a **measurement**. There is no `states/` family.

- **"reading" (lowercase) vs `Reading` (component)** — the MQTT messaging table lists "reading" as an alias to avoid for `sample`. The HMI component `Reading` (capital R) is a different concept: it is the component that renders a float sample value on screen. Disambiguate by capitalisation and context. In MQTT / protocol discussions: use `sample`. In HMI / design discussions: use `Reading`.

- **"class" (legacy)** — earlier docs and code used *class* for what is now called *template*. The legacy term remains valid only when discussing OOP / TypeScript / Python language constructs. Anywhere else it's a rename target.
