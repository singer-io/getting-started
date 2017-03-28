Best Practices for Submitting Taps to Stitch
============================================

Here are some best practices for writing Taps that will be run in Stitch's platform.

Language
--------

Currently all Stitch Taps are written in Python. This Best Practices guide
contains some python-specific recommendations. If you want to write a Tap
in another language and submit it to be run in Stitch's platform, please
contact us.

Recommended Config Fields
-------------------------

The Singer Specification does not define any standard fields in the
configuration, but in order to run a Tap in Stitch's platform, we
recommend supporting these two fields:

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

For some data sources, it won't make sense to pull every property
available. For example, suppose we had a Tap for a Postgres database, and
a user only wanted to pull a subset of columns from a subset of tables. It
would be inconvenient if the Tap emitted all columns for all tables.

To address this, we recommend allowing a Tap to produce a document
containing the "discovered" schemas for its data source, and allowing it
to also accept "annotated schemas" that indicate which streams and
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
Properties may optionally contain a `selectable` attribute, which means
that a user can decide whether to include those properties in the output.
Certain fields may not be selectable and will always be included if their
parent object is included. For example, for a database source the primary
key of each table must be included if the table is selected. An HTTP API
might always emit some fields, while allowing other fields to be selected.

Here's an example of a discovered schema:

```javascript
{
  "streams": {
    "users": {
      "type": "object",
      "properties": {
        "id": {"type": "integer"},
        "first_name": {"type": "string", "selectable": true},
        "last_name": {"type": "string", "selectable": true},
      }
    },
    "orders": {
      "type": "object",
      "properties": {
        "id": {"type": "integer"},
        "user_id": {"type": "integer", "selectable": true},
        "amount": {"type": "number", "selectable": true},
        "credit_card_number": {"type": "string", "selectable": true},
      }
    }
  }
}
```

2. Add `--properties PROPERTIES` option.

    A tap that supports property selection should accept an optional
    `--properties PROPERTIES` option. `PROPERTIES` should point to a file.
    The Tap should limit its output to the streams present in the
    `PROPERTIES` file, and further limit the fields in each stream to the
    ones marked as "selected" in the PROPERTIES file.

### Discovered Schema and Annotated Schema Format


```javascript
{"streams": [
  {"name": "orders",
   "schema": {
     "type": "object",
     "properties": {
       "user_id": {"type": "integer", "selected": true},
       "amount": {"type": "number", "selected": true},
     }
   }
  }
 ]
}
```



