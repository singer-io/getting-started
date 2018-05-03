# Singer Docs

Singer is an open source standard for moving data between databases, web APIs, files, queues, and just about anything else you can think of. The Singer spec describes how data extraction scripts — called “Taps” — and data loading scripts — called “Targets” — should communicate using a standard JSON-based data format over stdout. By conforming to this spec, Taps and Targets can be used in any combination to move data from any source to any destination.

Join the [Singer Slack channel](https://singer-slackin.herokuapp.com/) to get help from members of the Singer
community.

## Contents
- [Singer Specification](docs/SPEC#singer-specification)
  - [Synopsis](docs/SPEC#synopsis)
  - [Input](docs/SPEC#input)
    - [Config](docs/SPEC#config)
    - [State](docs/SPEC#state)
    - [Example Invocations](docs/SPEC#example-invocations)
  - [Output](docs/SPEC#output)
    - [RECORD Message](docs/SPEC#record-message)
    - [SCHEMA Message](docs/SPEC#schema-message)
    - [STATE Message](docs/SPEC#state-message)
  - [Example](docs/SPEC#example)
  - [Versioning](docs/SPEC#versioning)
- [Config and State](docs/CONFIG_AND_STATE#config-and-state)
  - [Config File](docs/CONFIG_AND_STATE#config-file)
  - [State File](docs/CONFIG_AND_STATE#state-file)
- [Discovery Mode](docs/DISCOVERY_MODE#discovery-mode)
  - [Schemas](docs/DISCOVERY_MODE#schemas)
  - [The Catalog](docs/DISCOVERY_MODE#the-catalog)
  - [Metadata](docs/DISCOVERY_MODE#metadata)
- [Sync Mode](docs/SYNC_MODE#sync-mode)
  - [Streams](docs/SYNC_MODE#streams)
  - [Replication Method](docs/SYNC_MODE#replication-method)
  - [Stream/Field Selection](docs/SYNC_MODE#streamfield-selection)
  - [Metric Messages](docs/SYNC_MODE#metric-messages)
- [Best Practices for Building a Singer Tap](docs/BEST_PRACTICES#best-practices-for-building-a-singer-tap)
  - [Rate Limiting](docs/BEST_PRACTICES#rate-limiting)
  - [Memory Constraints](docs/BEST_PRACTICES#memory-constraints)
  - [Dates](docs/BEST_PRACTICES#dates)
  - [Logging and Exception Handling](docs/BEST_PRACTICES#logging-and-exception-handling)
  - [Module Structure](docs/BEST_PRACTICES#module-structure)
  - [Schemas](docs/BEST_PRACTICES#schemas)
  - [Code Quality](docs/BEST_PRACTICES#code-quality)

  Copyright &copy; 2018 Stitch