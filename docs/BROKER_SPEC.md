# Broker Specification (v0.1.0)

## Context

The Singer specification is pretty mature and allow a lot of 
end-users to extract and load data from one source to a 
specific target very simply, and allow developer to easily
implement new taps without worrying about targets and vice-versa.

However, it is not straightforward, as of now, to run Singer channels (taps + targets)
in a production environment.
Because logs and metrics, by default when using the [singer-python](https://github.com/singer-io/singer-python) lib, 
are send to [stderr](https://github.com/singer-io/singer-python/blob/master/singer/logging.conf#L12), it is very difficult
and not straightforward to get to them to exploit them.

To bring the specification one step further, we need to improve on this point.
One possible implementation to resolve this problem, would be the implementation of a *broker* type and *LOG* messages into the specification.

## Definition

### LOG Message

A LOG message contain useful information relatives to the run of the tap/broker/target.


A LOG message have the following properties:

- `level` **Required**. The level of the LOG message. Should be one of `CRITICAL`, `ERROR`, `WARNING`, `INFO`, `DEBUG`
- `value` **Required**. The JSON formatted log

The semantics of a LOG value are not part of the specification,
and should be determined independently by each tap/broker/target.


Example
```
{"type": "LOG", "type": "DEBUG", "value": {"message": "All is good there"}}
{"type": "LOG", "type": "INFO", "value": {"metric": {"type": "counter", "metric": "record_count", "value": 10000, "tags": {"a": "tag"}}}}
{"type": "LOG", "type": "INFO", "value": {"metric": {"type": "timer", "metric": "job_duration", "value": 10.0, "tags": {"a": "tag"}}}}
{"type": "LOG", "type": "ERROR", "value": {"message": "I'm a teapot", "code": "418"}}
```

### Broker

A *Broker* is an application that read from stdin as input.
It should have a *configuration* file to be able to change
its behaviour when run.

A broker, unlike a tap, should not have any *state* and be agnostic
of the invocation environment.
A broker could be implemented in any programming language.

Usage
```
broker --config CONFIG

CONFIG is a required argument that points to a JSON file containing any
configuration parameters the Broker needs.
```

Theoretically, you could have an infinite sequence of brokers,
one after another, without perturbing the flow of a tap/target.

Example of a theoretical flow:
```
tap | broker_1 | broker_2 | ... | broker_n | target | broker_m | ... | broker_z
```

Broker are designed so Singer users could be able to add custom actions
on a Singer flow in a transparent way.
A broker could operate on in input, or letting it flow transparently to the next pipe.

Typical usecases could cover:

* Operations on `RECORD`
  * Filter records unwanted on the target destination
  * Enrich records on the fly with additionnal data
  * Transform records on the fly (lowercase, data masking, etc...)

* Operations on `SCHEMA`
  * Enrich schema on the fly (if new fields are detected or to make more explicit some patternProperties)

* Operations on `STATE`
  * Store/Load states in a remote storage (AWS/GCS/FTP/etc...)

* Operations on `LOG`
  * Store/Load  logs in a remote storage (AWS/GCS/FTP/etc...)
  * Send an error code and/or an error message to a external tool for debugging or alerting
  * Send metrics to a remote monitoring system and/or time series databases for deeper analysis and monitoring

Example of potentials brokers
* broker-statsd to send metrics to statsd
* broker-prometheus to send metrics to prometheus
* broker-elastic to send logs to elasticsearch
* broker-pagerduty to send an alert on pagerduty 
* broker-datadog to send logs and metrics on datadog

## Example of a new production-ready Singer channel

From 
```
tap-exchangeratesapi | target-csv
```

to

```
tap-exchangeratesapi | broker-statsd --config stats.json | broker-elastic --config elastic.json | target-csv | broker-pagerduty --config pagerduty.json
```
