![](https://arcnode.io/assets/banner.svg)
![](https://img.shields.io/badge/site-arcnode.io-27a699)
![](https://img.shields.io/badge/license-open-gray)


## About

Arcnode is the top-level catalog and overview repo for the [arcnode-io](https://gitlab.com/arcnode-io) organization. The system is built from four 10ft ISO container modules. Each is independently autonomous. They combine to form deployments from a single BESS node to a multi-MW AI compute site. The only custom-fabricated mechanical part in the entire system is the inter-container interface plate. 

The full software stack is open. The engineering templates, the EMS, and the sizing engine are publicly available.




## How It Works

```plantuml
actor operator
participant configurator
participant platform_api
participant edp_api
database platform_s3
actor epc_integrator
participant ems

operator -> configurator: load + site + context inputs
configurator -> platform_api: POST /platform-api/orders
platform_api --> configurator: 202 "check your inbox"
platform_api -> edp_api: POST /edp-api/jobs (ConfiguratorPayload)
edp_api --> platform_api: 202 { job_id }
platform_api -> edp_api: GET /edp-api/jobs/{job_id} (poll)
edp_api --> platform_api: edp_artifacts[] + ems_delivery
platform_api -> platform_s3: archive artifacts + render index.html
platform_api -> operator: email (portal link)
operator -> platform_s3: open portal\n(EDP downloads, CFN deep link or ISO, F-Droid link)
operator -> epc_integrator: hand off EDP
operator -> ems: spin up (sim mode → live on commissioning)
```

1. Operator enters load requirements, site constraints, and deployment context into the **System Configurator**.
2. The configurator submits to **`platform-api`**, which forwards the sizing payload to **`edp-api`** and waits for the 8 EDP artifacts.
3. `platform-api` archives the EDP artifacts to S3 and renders an HTML portal listing them alongside the EMS deployment link (CloudFormation deep link or air-gapped ISO, both embedding the DTM) and a link to the Android EMS app on F-Droid with DTM upload instructions.
4. Operator's inbox receives a link to the portal. EDP goes to the electrical integrator. Operator clicks the EMS deployment link, which spins up EMS in simulation mode and switches to live on commissioning. No vendor involvement required.

## Repositories

### EMS Suite (12 repos — runs on a deployed stack)

- [`ems`](https://gitlab.com/arcnode-io/ems) — umbrella + project overview
- [`ems-device-api`](https://gitlab.com/arcnode-io/ems-device-api) — device topology + topic provisioning
- [`ems-hmi`](https://gitlab.com/arcnode-io/ems-hmi) — web + mobile HMI
- [`ems-industrial-fixtures`](https://gitlab.com/arcnode-io/ems-industrial-fixtures) — mock industrial protocol fixtures
- [`ems-industrial-gateway`](https://gitlab.com/arcnode-io/ems-industrial-gateway) — protocol → MQTT bridge
- [`ems-line-controller`](https://gitlab.com/arcnode-io/ems-line-controller) — DLR + tap control feedback loop
- [`ems-line-controller-dlr`](https://gitlab.com/arcnode-io/ems-line-controller-dlr) — DLR sensors + IEEE 738 calc
- [`ems-line-controller-pst`](https://gitlab.com/arcnode-io/ems-line-controller-pst) — phase shift transformer tap control
- [`ems-analyst-api`](https://gitlab.com/arcnode-io/ems-analyst-api) — unified historical + ML + chat
- [`ems-analyst-model`](https://gitlab.com/arcnode-io/ems-analyst-model) — solar forecasting models
- [`ems-analyst-agent`](https://gitlab.com/arcnode-io/ems-analyst-agent) — energy analyst agent (RAG + KG)
- [`ems-analyst-server`](https://gitlab.com/arcnode-io/ems-analyst-server) — FastAPI service unifying the above

### EDP Toolchain (2 repos — engineering deployment packages)

- [`edp-api`](https://gitlab.com/arcnode-io/edp-api) — sizing engine + EDP artifact generator (single responsibility)
- [`edp-interface-plates`](https://gitlab.com/arcnode-io/edp-interface-plates) — CAD source for the inter-container interface plates

### Custom Hardware (1 repo)

- [`ems-line-controller-dlr-pcb`](https://gitlab.com/arcnode-io/ems-line-controller-dlr-pcb) — Pi HAT PCB for the solar-powered DLR sensor RTU

### Public Surface (3 repos)

- [`arcnode`](https://gitlab.com/arcnode-io/arcnode) — top-level catalog (this repo)
- [`platform-api`](https://gitlab.com/arcnode-io/platform-api) — order intake, delivery orchestration, email dispatch
- [`website`](https://gitlab.com/arcnode-io/website) — arcnode.io marketing site
