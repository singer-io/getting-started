# Singer

Singer is an open source standard for moving data between databases, web APIs, files, queues, and just about anything else you can think of. The Singer spec describes how data extraction scripts — called “Taps” — and data loading scripts — called “Targets” — should communicate using a standard JSON-based data format over stdout. By conforming to this spec, Taps and Targets can be used in any combination to move data from any source to any destination.

Join the [Singer Slack channel](https://singer-slackin.herokuapp.com/) to get help from members of the Singer
community.

## Docs
- [Singer Specification](docs/SPEC.md#singer-specification)
  - [Synopsis](docs/SPEC.md#synopsis)
  - [Input](docs/SPEC.md#input)
    - [Config](docs/SPEC.md#config)
    - [State](docs/SPEC.md#state)
    - [Example Invocations](docs/SPEC.md#example-invocations)
  - [Output](docs/SPEC.md#output)
    - [RECORD Message](docs/SPEC.md#record-message)
    - [SCHEMA Message](docs/SPEC.md#schema-message)
    - [STATE Message](docs/SPEC.md#state-message)
  - [Example](docs/SPEC.md#example)
  - [Versioning](docs/SPEC.md#versioning)
- [Running and Developing Singer Taps and Targets](docs/RUNNING_AND_DEVELOPING.md#running-and-developing-singer-taps-and-targets)
  - [Running Singer with Python](docs/RUNNING_AND_DEVELOPING.md#running-singer-with-python)
  - [Developing a Tap](docs/RUNNING_AND_DEVELOPING.md#developing-a-tap)
  - [Developing a Target](docs/RUNNING_AND_DEVELOPING.md#developing-a-target)
- [Config and State](docs/CONFIG_AND_STATE.md#config-and-state)
  - [Config File](docs/CONFIG_AND_STATE.md#config-file)
  - [State File](docs/CONFIG_AND_STATE.md#state-file)
- [Discovery Mode](docs/DISCOVERY_MODE.md#discovery-mode)
  - [Schemas](docs/DISCOVERY_MODE.md#schemas)
  - [The Catalog](docs/DISCOVERY_MODE.md#the-catalog)
  - [Metadata](docs/DISCOVERY_MODE.md#metadata)
- [Sync Mode](docs/SYNC_MODE.md#sync-mode)
  - [Streams](docs/SYNC_MODE.md#streams)
  - [Replication Method](docs/SYNC_MODE.md#replication-method)
  - [Stream/Field Selection](docs/SYNC_MODE.md#streamfield-selection)
  - [Metric Messages](docs/SYNC_MODE.md#metric-messages)
- [Best Practices for Building a Singer Tap](docs/BEST_PRACTICES.md#best-practices-for-building-a-singer-tap)
  - [Rate Limiting](docs/BEST_PRACTICES.md#rate-limiting)
  - [Memory Constraints](docs/BEST_PRACTICES.md#memory-constraints)
  - [Dates](docs/BEST_PRACTICES.md#dates)
  - [Logging and Exception Handling](docs/BEST_PRACTICES.md#logging-and-exception-handling)
  - [Module Structure](docs/BEST_PRACTICES.md#module-structure)
  - [Schemas](docs/BEST_PRACTICES.md#schemas)
  - [Code Quality](docs/BEST_PRACTICES.md#code-quality)
- [FAQ](docs/FAQ.md#FAQ)

---
  Copyright &copy; 2018 Stitch
