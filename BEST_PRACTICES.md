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
timestamps or ids are a sure way to have data discrepancies.

Write state as early and often as possible, but no sooner and no more often than is required. When a
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

The Tap's config file can include a `start_date` field indicating the default state.

### Stream bookmark format

State records serve to indicate a Tap's progress through a data source, but they can also provide
more granular information about progress through individual streams.

If a Tap's streams can each have an individual state, the state output by the Tap should conform to
the following format: the state object contains a top-level property `"bookmarks"` that maps to an
object. The bookmarks object contains stream identifiers as property names, each of which maps to an
object describing the respective stream's state. As a concrete example:

```
{
  "bookmarks": {
    "orders": {
      "last_record": "2017-07-07T10:20:00Z"
    },
    "customers": {
      "last_record": 123
    }
  }
}
```

In the above example, there are two streams that have been bookmarked: `"orders"` and `"customers"`.
Each of those streams has some replication key field, like an `updated_at` timestamp or a sequential
ID for append-only sources. The replication key value persisted as part of the state message allows
the Tap to:

- bookmark its progress through the stream
- serve as a query parameter upon subsequent invocations, (e.g. the equivalent of `SELECT * from
  customers where id >= 123`)

The state record above indicates that the Tap last output an entry from the `"orders"` stream where
the source record's replication key field had the value "2017-07-07T10:20:00Z", and that it last
output an entry from the `"customers"` stream where the source record's replication key field had
the value 123.


Logging and Exception Handling
------------------------------

Log every URL + params that will be requested, but be sure to exclude any sensitive information
such as api keys and client secrets. Log the progress of the sync (e.g. Starting entity 1,
Starting entity 2, etc.) When the API returns with an error, log the status code and body.

If an error causes the tap or target to exit, log the error at the
CRITICAL or FATAL level just before exiting with a non-zero status. The
expectation is that if the tap or target fails, a calling script can look
for lines that begin with CRITICAL or FATAL in order to get the most
relevant error message. If the fatal error is one that the tap or target
explicitly raises, consider omitting the stack trace. However if it's an
Exception from an unknown origin, log the full stack trace in addition to
logging the message at the CRITICAL or FATAL level.

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

Allowing Users to Select Streams to Sync
----------------------------------------

For some data sources, it won't make sense to pull every stream and
property available. For example, suppose we had a Tap for a Postgres
database, and a user only wanted to pull a subset of columns from a
subset of tables. It would be inconvenient if the Tap emitted data
from all tables and columns.

To address this, we recommend extending the Tap to use a document called a
_catalog_. A _catalog_ is a mechanism for a Tap to indicate what streams
it makes available, and for the user to select certain streams and
properties within those streams.

A Tap that supports a catalog should provide two additional options:

* `--discover` - indicates that the Tap should not sync data, but should
  just write its catalog to stdout.

* `--catalog CATALOG` - the Tap should sync data, based on the selections
  made in the provided CATALOG file.

### Catalog Format

The format of the catalog is as follows. The top level is an object,
with a single key called `"streams"` that points to an array of
objects, each having the following fields:

| Property          | type               | required? | Description                    |
| ----------------- |--------------------|-----------|--------------------------------|
| `tap_stream_id`   | string             | required  | The unique identifier for the stream. This is allowed to be different from the name of the stream in order to allow for sources that have duplicate stream names. |
| `stream`          | string             | required  | The name that will be used for the stream in the data produced by this Tap. |
| `key_properties`  | array of strings   | optional  |  List of key properties. |
| `schema`          | object             | required  | The JSON schema for the stream.  |
| `replication_key` | string             | optional  | The name of a property in the source to use as a "bookmark". For example, this will often be an "updated-at" field or an auto-incrementing primary key. |
| `is_view`         | boolean            | optional  | For a database source, indicates that the source is a view. |
| `database_name`   | string             | optional  | For a database source, the name of the database. |
| `table_name`      | string             | optional  | For a database source, the name of the table. |
| `row_count`       | integer            | optional  | The number of rows in the source data, for taps that have access to that information. |

### JSON Schema Extensions

In order to allow a Tap to indicate which fields are selectable and to
allow the user to make their selections, we extend JSON Schema by adding
two additional properties. Note that since JSON Schema is recursive, these
properties may appear on the top-level schema or on properties within the
schema:

* `inclusion`: Either `available`, `automatic`, or `unsupported`.

    * `"available"` means that the field is available for selection, and that
      the Tap will only emit values for that field if it is marked with
      `"selected": true`.
    * `"automatic"` means that the Tap may emit values for the field, but it
      is not up to the user to select it.
    * `"unsupported"` means that the field exists in the source data but the
      Tap is unable to provide it.
* `selected`: Either `true` or `false`. For a top-level schema, `true`
   indicates that the stream should be synced, and `false` indicates it
   should be omitted entirely. For a property within a stream, `true`
   means include the property, `false` means leave it out.

Here's an example of a discovered catalog

```javascript
{
  "streams": [
    {
      "stream": "users",
      "tap_stream_id": "users",
      "schema": {
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
      }
    },
    {
      "stream": "orders",
      "tap_stream_id": "orders",
      "schema": {
        "type": "object",
        "properties": {
          "id": {"type": "integer"},
          "user_id": {"type": "integer", "inclusion": "available"},
          "amount": {"type": "number", "inclusion": "available"},
          "credit_card_number": {"type": "string", "inclusion": "available"},
        }
      }
    }
  ]
}
```

### Discovery Mode

A Tap that wants to support property selection should add an optional
`--discover` flag. When the `--discover` flag is supplied, the Tap
should connect to its data source, find the list of streams available,
and print out the catalog to stdout. The discovery output should go to
STDOUT, and it should be the only thing written to STDOUT. If the
`--discover` flag is supplied, a tap should not emit any RECORD,
SCHEMA, or STATE messages.

### Sync Mode

A tap that supports property selection should accept an optional
`--catalog CATALOG` option. `CATALOG` should point to a file containing
the catalog, annotated with the user's "selected" choices.

The Tap should attempt to sync every stream that is listed in the
PROPERTIES file where the "selected" property of the stream's schema is
`true`. For each of those streams, it should include all the properties
that are marked as selected for that stream. If the requested schema
contains a property that does not exist in the data source, a Tap may fail
and exit with a non-zero status or it may omit the requested field from
the output. The Tap may include additional properties that are not
included in the catalog, if those properties are always provided by the
data source. The tap should not include in its output any streams that are
not present in the catalog.

Metrics
-------

A Tap should periodically emit structured log messages containing metrics
about read operations. Consumers of the Tap logs can parse these metrics
out of the logs for monitoring or analysis.

```
INFO METRIC: <metrics-json>
```

`<metrics-json>` should be a JSON object with the following keys;

* `type` - The type of the metric. Indicates how consumers of the data
  should interpret the `value` field. There are two types of metrics:

    * `counter` - The value should be interpreted as a number that is added
      to a cumulative or running total.

    * `timer` - The value is the duration in seconds of some operation.

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
INFO METRIC: {"type": "timer", "metric": "http_request_duration", "value": 1.23, "tags": {"endpoint": "orders", "http_status_code": 200, "status": "succeeded"}}
```

> We made an HTTP request to an "orders" endpoint that took 1.23 seconds
> and succeeded with a status code of 200.

#### Example 2: Timer for Failed HTTP GET

```
INFO METRIC: {"type": "timer", "metric": "http_request_duration", "value": 30.01, "tags": {"endpoint": "orders", "http_status_code": 500, "status": "failed"}}
```

> We made an HTTP request to an "orders" endpoint that took 30.01 seconds
> and failed with a status code of 500.

#### Example 3: Counter for Records

```
INFO METRIC: {"type": "counter", "metric": "record_count", "value": 100, "tags": {"endpoint: "orders"}}
INFO METRIC: {"type": "counter", "metric": "record_count", "value": 100, "tags": {"endpoint: "orders"}}
INFO METRIC: {"type": "counter", "metric": "record_count", "value": 100, "tags": {"endpoint: "orders"}}
INFO METRIC: {"type": "counter", "metric": "record_count", "value": 14, "tags": {"endpoint: "orders"}}

```

> We fetched a total of 314 records from an "orders" endpoint.
