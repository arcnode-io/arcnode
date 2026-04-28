![](https://arcnode.io/assets/banner.svg)
![](https://img.shields.io/badge/spec-yaml-gray)


## About

Versioned **device templates** consumed by `edp-api` (sole runtime reader — sizes deployments and embeds the templates each DTM uses), and by downstream consumers (`ems-device-api`, `ems-hmi`, gateway, analyst) through the DTM payload itself. Each template declares the canonical measurement vocabulary, command vocabulary, protocol-binding template, and standards-compliance metadata (IEC 61850, Redfish, BACnet — only where applicable) for one type of module or component.

We use *template* rather than *class* deliberately — *class* is OOP jargon that doesn't translate to operators or industrial integrators. *Template* is the standard SCADA/HMI term for "a definition you instantiate." OPC UA's *ObjectType*, IEC 61850's *Logical Node Type*, BACnet's *ObjectType*, and Sparkplug's *Device Definition* all model the same concept; we picked the term that reads cleanly across operator UI, ADR text, and code.

The per-deployment DTM declares *instances*, electrical bus topology, and a `templates_used` map of every template the `devices` block references:

```json
{
  "deployment_uuid": "...",
  "devices": {
    "bess_01":        { "template": "bess_module.v1",   "display_name": "BESS 1" },
    "bess_02":        { "template": "bess_module.v1",   "display_name": "BESS 2" },
    "grid_module_01": { "template": "grid_module.v1",   "display_name": "Grid Module" },
    "compute_01":     { "template": "compute_module.v1","display_name": "Compute" }
  },
  "buses": [
    {
      "id": "dc_bus_1",
      "type": "dc",
      "members": [
        { "device_id": "bess_01" },
        { "device_id": "bess_02" },
        { "device_id": "grid_module_01", "port": "dc" }
      ]
    },
    {
      "id": "ac_bus_1",
      "type": "ac",
      "members": [
        { "device_id": "grid_module_01", "port": "ac" },
        { "device_id": "compute_01" }
      ]
    }
  ],
  "templates_used": {
    "bess_module.v1":    { "template": "bess_module", "version": "v1", "...": "..." },
    "grid_module.v1":    { "template": "grid_module", "version": "v1", "...": "..." },
    "compute_module.v1": { "template": "compute_module", "version": "v1", "...": "..." }
  }
}
```

`buses[]` field semantics:

| Field | Mandatory | Notes |
|---|---|---|
| `id` | yes | Snake_case slug; IEC 61850 ConnectivityNode equivalent |
| `type` | yes | `dc` \| `ac` |
| `members[].device_id` | yes | Must resolve to a key in `devices` |
| `members[].port` | no | Required only when a device bridges two buses |

`buses[]` is authored by `edp-api` from the sizing payload — not hand-edited. `templates_used[]` is also populated by `edp-api` — sole reader of this directory. The spec generator (`ems-device-api`) reads `templates_used` directly off the DTM payload, never from disk.

See [`UBIQUITOUS_LANGUAGE.md`](../UBIQUITOUS_LANGUAGE.md) for the canonical meaning of *module*, *component*, and *template*.
See [`ems/topic_structure_adr.md`](../../ems/topic_structure_adr.md) for the topic shape, payload conventions, naming rules, and unit vocabulary that templates consume.


## Template Tiers

Three tiers, mirroring the hardware and the ubiquitous language.

### Module Templates (one per 10ft ISO container type)

| Template | Marketing name | Notes |
|---|---|---|
| `bess_module.v1`    | BESS Module    | Battery storage, autonomous BMS, parallel-stackable |
| `compute_module.v1` | Compute Module | GPU-dense AI compute (H100/B200), NVLink, NERC CIP monitoring |
| `thermal_module.v1` | Thermal Module | Direct liquid cooling, 1:1 to compute, climate-adaptive |
| `grid_module.v1`    | Grid Module    | PCS + interconnect, IEEE 1547, optional (off-grid when absent) |

### Equipment Templates (composed by modules)

```
bess_module      ⇒ bess_rack, bess_bms, bess_cell, bess_inverter, bess_contactor, bess_thermal_sensor
compute_module   ⇒ gpu_server, nvlink_switch, pdu, server_bmc
thermal_module   ⇒ chiller, pump, flow_sensor, coolant_temp_sensor, leak_detector
grid_module      ⇒ pcs, grid_transformer, grid_meter, breaker, ground_fault_relay
```

Equipment containment is **arbitrary depth**. A `bess_module` may contain `bess_rack`s, which in turn contain `bess_bms`/`bess_inverter`/`bess_cell`. The DTM expresses depth via `parent: device_id` links per ADR-002 §7. Multi-level applies equally to compute (cluster → chassis → server → GPU) and thermal (plant → chiller → pump/CRAH).

### Line Templates (cross-cutting, not container-bound)

```
dlr_sensor                # transmission-line dynamic line rating sensor
phase_shift_transformer   # PST tap controller
industrial_gateway        # protocol bridge (Modbus/DNP3/SNMP/Redfish/CAN → MQTT)
```


## File Layout

```
arcnode/device_templates/
├── readme.md
├── modules/
│   ├── bess_module.v1.yaml
│   ├── compute_module.v1.yaml
│   ├── thermal_module.v1.yaml
│   └── grid_module.v1.yaml
├── equipment/
│   ├── bess_rack.v1.yaml
│   ├── bess_bms.v1.yaml
│   ├── bess_cell.v1.yaml
│   ├── bess_inverter.v1.yaml
│   ├── gpu_server.v1.yaml
│   ├── chiller.v1.yaml
│   ├── pcs.v1.yaml
│   └── ...
└── line/
    ├── dlr_sensor.v1.yaml
    └── phase_shift_transformer.v1.yaml
```


## Template Schema

```yaml
template: bess_module       # snake_case slug, matches file basename
version:  v1                # vN integer suffix, matches file suffix

# Standards-compliance metadata — all blocks optional, include only where applicable.
# A template typically maps to ONE standard depending on what it touches.

iec_61850:                  # grid-touching templates only (PCS, breaker, meter, BESS w/ DER ext.)
  logical_nodes: [ZBAT, ZGEN, MMXU, CSWI]

redfish:                    # compute-side templates only (servers, BMCs)
  schema_refs: [ComputerSystem.v1_19_0, Power.v1_5_0, Thermal.v1_7_0]

bacnet:                     # thermal/HVAC templates only
  object_types: [analog_input, binary_output, multi_state_value]

# Module-tier only — composition rules. Children may declare further `contains:` to chain depth.
contains:
  required:
    - { template: bess_bms.v1,      count: 1 }
    - { template: bess_inverter.v1, count: 1 }
  scalable:
    - { template: bess_cell.v1, count_min: 1, count_max: 256 }

# At least one of measurements: or commands: required
measurements:
  voltage_dc:
    unit: volts
    type: float
    poll_rate_hz: 0.5
    display_name_default: "DC Voltage"
    iec_61850_ref: "MMXU1.PhV.cVal.mag.f"      # optional, only if iec_61850 block present
  alarm_state:
    unit: none
    type: enum
    values:
      ok:    { severity: ok,    register_value: 0 }
      warn:  { severity: warn,  register_value: 1 }
      fault: { severity: alarm, register_value: 2 }
    display_name_default: "Alarm"
  control_mode:
    unit: none
    type: enum
    values:
      AUTO:   { register_value: 0 }
      MANUAL: { register_value: 1 }
      RUNPQ:  { register_value: 2 }
    display_name_default: "Control Mode"

commands:
  set_active_power:
    verb: set
    target: active_power
    unit: watts
    payload: float
    display_name_default: "Active Power Setpoint"
  reset_alarm:
    verb: reset
    target: alarm
    unit: none
    payload: trigger
    display_name_default: "Reset Alarm"

subscribes_to:              # optional cross-instance wiring (implicit by template)
  - template: dlr_sensor.v1
    measurement: dynamic_rating
    scope: same_site         # same_site | same_module | same_deployment

protocol_binding_template:  # optional gateway-side scaffold; instance fills host/unit_id
  type: modbus_tcp           # modbus_tcp | dnp3 | snmp | redfish | canbus | mqtt_native
  register_map:
    voltage_dc: { addr: 3000, type: int16, scale: 0.1 }
```


## Versioning

Template versions follow the same semver rules as the AsyncAPI spec ([`ems/topic_structure_adr.md`](../../ems/topic_structure_adr.md) §10).

- **patch** — display defaults, standards-ref tweaks, protocol-binding scale fixes, poll-rate adjustments
- **minor** — new measurement, new command, new `subscribes_to` wire
- **major** — measurement renamed/removed, unit changed, command verb-target changed

Major bumps create a new template file alongside the old (`bess_module.v1.yaml`, `bess_module.v2.yaml`) — the DTM picks the version per instance, allowing in-place migrations.


## Validation

CI validates that:

- File basename matches `template:` field; file suffix matches `version:` field
- `template:` and `version:` always present
- At least one of `measurements:` or `commands:` present
- Module-tier templates (in `modules/`) have a `contains:` block
- Every measurement has a `unit` from the locked vocabulary in the topic ADR + a `type` in `{float, bool, enum}`
- `type=enum` measurements have a `values:` map where keys are the string labels published on MQTT and used as the AsyncAPI `enum` constraint
- When the template has a non-native protocol binding (`modbus_tcp`, `dnp3`, `snmp`, `canbus`), every `values:` entry must include `register_value: <uint>` — the gateway uses this to translate raw integers to string labels before publishing
- `register_value` is optional (omit) for `mqtt_native` and `redfish` bindings since those protocols carry string enums natively
- Every entry with a `severity` key must have a value in `{ok, warn, alarm}`
- Every command has `verb` from the locked enum + `target` (snake_case slug) + `unit` + `payload` in `{float, bool, enum, trigger}`
- `iec_61850_ref` strings match the IEC 61850 dotted-path syntax
- `parent:` references in DTM resolve to same-site devices whose template is allowed by the parent's `contains:` block
- Every `template:` reference in any DTM resolves to a file in this directory
- DTM `buses[].members[].device_id` values must resolve to keys in `devices`
- DTM `buses[].type` must be `dc` or `ac`
- DTM `templates_used` contains exactly the templates referenced by `devices[*].template` — no missing, no extras
