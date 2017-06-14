Best Practices for Building a Singer Tap
============================================

Language
--------

Currently all Singer Taps are written in Python. This Best Practices guide
contains some python-specific recommendations.

Recommended Config Fields
-------------------------

The Singer Specification does not define any standard fields in the
configuration. We recommend supporting the following fields:

* `start_date` - A tap should take a `start_date` field in the config that
  Indicates how far back an integration should sync data in the absence of
  a state entry.

* `user_agent` - A tap should accept a `user_agent` field in the config
  and pass it along in request headers to the API. The caller of a Tap
  should include an email address in the `user_agent` field to allow the
  API provider to contact you if there's an issue with the tap or your
  usage of their API.

Rate Limiting
-------------

Most APIs enforce rate limits. Taps should be written in a way that respects these rate limits.

Rate limits can take the form of quotas (X requests per day), in which case the tap should leave
room for other use of the API, or short-term limits (X requests per Y seconds). For short-term
limits, the entire quota can be used up and the tap should sleep when rate limited.

The singer-python library's utils namespace has a rate limiting decorator for use.

Memory Constraints
------------------

Taps should not rely on large volumes of RAM being available during run and should strive to keep
as little data in memory as required. Use iterators/generators when available.


Dates
-----

All dates should use the RFC3339 format (which includes timezone offsets). UTC is the preferred
timezone for all data and should be used when possible.

Good:
 - 2017-01-01T00:00:00Z (January 1, 2017 12:00:00AM UTC)
 - 2017-01-01T00:00:00-05:00 (January 1, 2017, 12:00:00AM America/New_York)

Bad:
 - 2017-01-01 00:00:00

State
-----

States that are datetimes will all be in RFC3339 format. Using ids or other identifiers for state
is possible but could cause issues if the entities are not immutable and change out of order. If
the API supports filtering by created timestamps but is not immutable (meaning a record can be
updated after creation), DO NOT use the created timestamp for saving state. States using created
timestamps or ids are a sure way to have data discrepencies.

State as early and often as possible, but no sooner and no more often than is required. When a
state is written, no data prior to that state will be synced, so do not update the state until all
possible exceptions will be thrown.

Endpoints that do not support filtering by updated timestamps or include updated timestamps do not
support saving state.

If the API supports filtering by updated timestamp, use that for filtering. If the API doesn't
support filtering but does return updated timestamps with the data, filter by the timestamp before
streaming the data.

When streaming data in, stream the data in ascending order when possible.

Keep in mind that jobs can be interrupted at any point. States should never be in an invalid
state. Interrupted jobs that save state too early will have data missing. Interrupted jobs that
save state too late will cause an increase in duplicate rows being replicated.

The tap's config file must ALWAYS have a `start_date` field indicating the default state.

Logging and Exception Handling
------------------------------

Log every URL + params that will be requested, but be sure to exclude any sensitive information
such as api keys and client secrets. Log the progress of the sync (e.g. Starting entity 1,
Starting entity 2, etc.) When the API returns with an error, log the status code and body.

Allow exceptions to be bubbled up and interrupt the tap. DO NOT wrap the tap code in one large
try/except block and log the exception message. The stack trace is much more useful than the error
message.

If an intermittent error is detected from the API, retry using an exponential backoff (try using
`backoff` library for Python). If an unrecoverable error is detected, exit the script with a
non-zero error code (raise an exception or use `sys.exit(1)`)


Module Structure
----------------

Source code should be in a module (folder with `__init__.py` file) and not just a script (`module.py`
file).


Schemas
-------

Schemas should be stored in a `schemas` folder under the module directory
as JSON files rather than as native dicts in the source code.

Always stream the entity's schema before streaming any records for that
entity.

If the API you're using does not publish a schema, you can use the
`singer-infer-schema` program in [singer-tools] to generate a base schema
to start from.


Testing
-------

Use `singer-tools`'s `singer-check-tap` command to validate that a tap's
output matches the expected format and validate the output against its
schema.

Code Quality
------------

We recommend running pylint on your Tap and making sure that you
understand any issues it reports. You should get your code to the point
where pylint does not report any error-level messages.

When we import your Tap, we'll run it in CircleCI, and we'll include
Pylint in the test phase. The CircleCI buld will fail if Pylint finds any
issues, so we will need to account for all issues by either fixing them or
disabling the message in Pylint. We are flexible about which Pylint issues
are acceptable, but we generally run Pylint with some of the more pedantic
messages disabled. For example, we typically use the following circle.yml
config:

```yaml
machine:
  python:
    version: 3.4.4

dependencies:
  pre:
    - pip install pylint

test:
  post:
    - pylint tap_outbrain -d missing-docstring -d logging-format-interpolation -d too-many-locals -d too-many-arguments
```

Schema Discovery and Property Selection
---------------------------------------

For some data sources, it won't make sense to pull every stream and
property available. For example, suppose we had a Tap for a Postgres
database, and a user only wanted to pull a subset of columns from a subset
of tables. It would be inconvenient if the Tap emitted all columns for all
tables.

To address this, we recommend allowing a Tap to produce a document
containing the "discovered" schemas for its data source, and allowing it
to also accept "desired schemas" that indicate which streams and
properties to sync.

### Schema Discovery

A Tap that wants to support property selection should add an optional
`--discover` flag. When the `--discover` flag is supplied, the Tap should
connect to its data source, find the list of streams available, and print
out a document listing each stream along with the discovered schema. The
discovery output should go to STDOUT, and it should be the only thing
written to STDOUT. If the `--discover` flag is supplied, a tap should not
emit any RECORD, SCHEMA, or STATE messages.

The format of the discovery output is as follows. The top level is an
object, with a single key called "streams", that points to a map where
each key is a stream name and each value is the discovered schema for that
stream.

The discovered schema is in JSON schema format, with one extension.
Properties may optionally contain an `inclusion` attribute, which can have
one of three values:

* automatic - the user cannot select the attribute, it will be automatically included.
* available - the user can select the attribute for inclusion.
* unsupported - the Tap does not support this attribute, and is just reporting it in the schema to indicate that the attribute exists in the source but will not be included.

Here's an example of a discovered schema:

```javascript
{
  "streams": {
    "users": {
      "type": "object",
      "properties": {
        "id": {
          "type": "integer",
          "inclusion": "automatic"
        },
        "name": {
          "type": "object",
          "inclusion": "available",
          "properties": {
            "first_name": {"type": "string", "inclusion": "available"},
            "last_name": {"type": "string", "inclusion": "available"}
          },
        },
        "addresses": {
          "type": "array",
          "inclusion": "available",
          "items": {
            "type": "object",
            "inclusion": "available",
            "properties": {
              "addr1": {"type": "string", "inclusion": "available"},
              "addr2": {"type": "string", "inclusion": "available"},
              "city": {"type": "string", "inclusion": "available"},
              "state": {"type": "string", "inclusion": "available"},
              "zip": {"type": "string", "inclusion": "available"},
            }
          }
        }
      }
    },
    "orders": {
      "type": "object",
      "properties": {
        "id": {"type": "integer"},
        "user_id": {"type": "integer", "inclusion": "available"},
        "amount": {"type": "number", "inclusion": "available"},
        "credit_card_number": {"type": "string", "inclusion": "available"},
      }
    }
  }
}
```

### Property Selection

A tap that supports property selection should accept an optional
`--properties PROPERTIES` option. `PROPERTIES` should point to a file
containing the desired schemas.

The format of PROPERTIES file is similar to the output from discovery. The
top level is an object, with a single key called "streams", that points to
a map where each key is a stream name and each value is the requested
schema for that stream. The caller requests properties by decorating those
properties with a `"selected": true` attribute in the schema.

The Tap should attempt to sync every stream that is listed in the
PROPERTIES file. For each of those streams, it SHOULD include all the
properties that are marked as selected for that stream. If the requested
schema contains a property that does not exist in the data source, a Tap
MAY fail and exit with a non-zero status or it MAY omit the requested
field from the output. The Tap MAY include additional properties that are
not included in the desired schema, if those properties are always
provided by the data source. The tap MUST NOT include in its output any
streams that are not present in desired schemas file.

Note that the Tap should not validate the data against the desired schema.
The purpose of the desired schema is _only_ to communicate which streams
and properties are desired.

For example, suppose we wanted to sync only the users table, and only the
last name and state of each user. Our desired schemas file would look like
this:

```javascript
{
  "streams": {
    "users": {
      "type": "object",
      "selected": true,
      "properties": {
        "id": {"type": "integer"},
        "name": {
          "type": "object",
          "selected": true,
          "properties": {
            "last_name": {"type": "string"}
          },
        },
        "addresses": {
          "type": "array",
          "selected": true,
          "items": {
            "type": "object",
            "properties": {
              "state": {
                "type": "string",
                "selected": true
              }
            }
          }
        }
      }
    }
  }
}
```

Since the Tap will ignore the type information on the desired schema, we
can prune out the "type" fields. The following is sufficient for a Tap to
know that it should sync the "users.last_name" and "addresses.state"
properties of the "users" stream:

```javascript
{
  "streams": {
    "users": {
      "selected": true
      "properties": {
        "name": {
          "selected": true,
          "properties": {
            "last_name": {"selected": true},
            "first_name": {"selected": true}
          }
        },
        "addresses": {
          "selected": true,
          "items": {
            "properties": {
              "state": {
                "selected": true
              }
            }
          }
        }
      }
    }
  }
}
```

Metrics
-------

A Tap should periodically emit structured log messages containing metrics
about read operations.

```
INFO METRIC: <metrics-json>
```

`<metrics-json>` should be a JSON object where the keys are limited to the following:

* type
* metric
* value
* tags

### Fields

#### source

* `type` - The type of the metric. Indicates how consumers of the data
  should interpret the "value" field. There are two types of metrics,
  "counter" and "timer":
  
    * `counter` - the value should be interpreted as a number that is added
      to a cumulative or running total.
      
    * `timer` - the value is the duration in seconds of some operation.
  
* `metric` - The name of the metric. This should consist only of letters,
  numbers, underscore, and dash characters. For example,
  `"http_request_duration"`.

* `value` - The value of the datapoint, either an integer or a float. For
  example, `1234` or `1.234`.

* `tags` - Mapping of tags describing the data. The keys can be any
  strings consisting solely of letters, numbers, underscores, and dashes.
  For consistency's sake, we recommend using the following tags when they
  are relevant.
  
    * `endpoint` - For a Tap that pulls data from an HTTP API, this should
      be a descriptive name for the endpoint, such as `"users"` or `"deals"`
      or `"orders"`.

    * `http_status_code` - The HTTP status code. For example, `200` or
      `500`.

    * `job_type` - For a process that we are timing, some description of
      the type of the job. For example, if we have a Tap that does a POST
      to an HTTP API to generate a report and then polls with a GET until
      the report is done, we could use a job type of `"run_report"`.
    
    * `status` - Either `"succeeded"` or `"failed"`.

  Note that for many metrics, many of those tags will _not_ be relevant.
  
### Examples

Here are some examples of metrics and how those metrics should be
interpreted.

#### Example 1: Timer for Successful HTTP GET

```
INFO STATS: {"type": "timer",\
             "metric": "http_request_duration",\
             "value": 1.23,\
             "tags": {\
               "endpoint": "orders",\
               "http_status_code": 200,\
               "status": "succeeded"\
             }\
            }
```

> We made an HTTP request to an "orders" endpoint that took 1.23,
> succeeded with a status code of 200, and returned a body with 12345
> bytes containing 100 records.

#### Example 2: Failed HTTP GET

```
INFO STATS: {"source": "orders", "status": "failed", "http_status_code": 400, "duration": 1.23}
```

> We made an HTTP request to an "orders" endpoint that took 1.23 seconds
> and failed with a status code of 400.

#### Example 3: Successful streaming operation

```
INFO STATS: {"source": "orders", "status": "running", "duration": 1.2, "record_count": 100}
INFO STATS: {"source": "orders", "status": "running", "duration": 0.9, "record_count": 100}
INFO STATS: {"source": "orders", "status": "running", "duration": 2.3, "record_count": 100}
INFO STATS: {"source": "orders", "status": "succeeded", "duration": 0.1, "record_count": 14}
```

We fetched a total of 314 records from an "orders" endpoint in 4.5 seconds.

#### Example 4: Successful HTTP GET

```
INFO STATS: {"source": "orders", "status": "succeeded", "http_status_code": 200, "duration": 1.23}
```

> We made an HTTP request that succeeded and returned an unspecified
> number of records.

If a Tap hits a lot of different HTTP endpoints with different
response structures, and it would be too cumbersome to discern and
report accurate record counts, a Tap can simply omit that field.
