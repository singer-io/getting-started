# Singer Specification

### Version 0.3.0

A *Tap* is an application that takes a *configuration* file and optional 
*state* and *catalog* files as input and produces an ordered stream of 
*record*, *state* and *schema* messages as output. A *record* is json-encoded
data of any kind. A *state* message is used to persist information between
invocations of a Tap. A *schema* message describes the datatypes of
the *record*s in the stream. A Tap may be implemented in any
programming language.

Taps are designed to produce a stream of data from sources like
databases and web service APIs for use in a data integration or ETL
pipeline.

## Synopsis

```
tap --config CONFIG [--state STATE] [--catalog CATALOG]

CONFIG is a required argument that points to a JSON file containing any
configuration parameters the Tap needs.

STATE is an optional argument pointing to a JSON file that the
Tap can use to remember information from the previous invocation,
like, for example, the point where it left off.

CATALOG is an optional argument pointing to a JSON file that the
Tap can use to filter which streams should be synced.

```

## Input

### Config

The configuration contains whatever parameters the Tap needs in order to
pull data from the source. Typically this will include the credentials for
the API or data source. The format of the configuration will vary by Tap,
but it must be JSON-encoded and the root of the configuration must be an
object.

For taps where the configuration needs to be changed during a run, the tap
should write changes back to the supplied config file, so that the changes
can be used on subsequent runs.

See the [Config File](CONFIG_AND_STATE.md#config-file) section for more
information.

### State

The JSON encoded state is used to persist information between invocations of a Tap.

A Tap that wishes to persist state should periodically write STATE
messages to stdout as it processes the stream, and should expect the file
named by the `--state STATE` argument to have the same format as the value
of the STATE messages it emits.

A common use case of state is to record the spot in the stream where the
last invocation left off. If the Tap is invoked without a `--state STATE`
argument, it should start at the beginning of the stream or at some
appropriate default position. If it is invoked with a `--state STATE`
argument it should read in the state file and start from the corresponding
position in the stream.

See the [State File](CONFIG_AND_STATE.md#state-file) section for more information.

### Catalog

The catalog is a JSON encoded file that lists the available streams and
their schemas. The top level is an object, with a single key called
"streams" that points to an array of stream objects. The metadata for each
stream object can be modified to select whether a stream and/or its fields
should be replicated, and how the data should be replicated (FULL TABLE vs
INCREMENTAL).

See the [Catalog](DISCOVERY_MODE.md#the-catalog) section for more information.

### Example Invocations

Sync from the beginning without catalog

```bash
$ tap --config config.json
```

Sync starting from a stored state with catalog

```bash
$ tap --config config.json --state state.json --catalog catalog.json
```

## Output

A Tap outputs structured messages to `stdout` in JSON format, one
message per line. Logs and other information can be emitted to `stderr`
for aiding debugging. A streamer exits with a zero exit code on success,
non-zero on failure.

The body contains messages encoded as a JSON map, one message per
line. Each message must contain a `type` attribute. Any message `type`
is permitted, and `type`s are interpreted case-insensitively. The
following `type`s have specific meaning:

### RECORD Message

RECORD messages contain the data from the data stream. They must have
the following properties:

 - `record` **Required**. A JSON map containing a streamed data point

 - `stream` **Required**. The string name of the stream

 - `time_extracted` **Optional**. The time this record was observed in the
   source. This should be an
   [RFC3339](https://www.ietf.org/rfc/rfc3339.txt) formatted date-time,
   like "2017-11-20T16:45:33.000Z".

A single Tap may output RECORDs messages with different stream
names.  A single RECORD entry may only contain records for a single
stream.

Example:

*Note: Every message must be on its own line, but the examples here use
 multiple lines for readability.*

```json
{
  "type": "RECORD",
  "stream": "users",
  "time_extracted": "2017-11-20T16:45:33.000Z",
  "record": {
    "id": 0,
    "name": "Chris"
  }
}
```

### SCHEMA Message

SCHEMA messages describe the datatypes of data in the stream. They
must have the following properties:

 - `schema` **Required**. A [JSON Schema] describing the
   `data` property of RECORDs from the same `stream`

 - `stream` **Required**. The string name of the stream that this
   schema describes

 - `key_properties` **Required**. A list of strings indicating which
   properties make up the primary key for this stream. Each item in the
   list must be the name of a top-level property defined in the schema. A
   value for `key_properties` must be provided, but it may be an empty
   list to indicate that there is no primary key.

 - `bookmark_properties` **Optional**. A list of strings indicating which
   properties the tap is using as bookmarks. Each item in the list must be
   the name of a top-level property defined in the schema.

A single Tap may output SCHEMA messages with different stream
names.  If a RECORD message from a stream is not preceded by a
`SCHEMA` message for that stream, it is assumed to be schema-less.

Example:

*Note: Every message must be on its own line, but the examples here use
 multiple lines for readability.*

```json
{
  "type": "SCHEMA",
  "stream": "users",
  "schema": {
    "properties": {
      "id": {
        "type": "integer"
      },
      "name": {
        "type": "string"
      },
      "updated_at": {
        "type": "string",
        "format": "date-time"
      }
    }
  },
  "key_properties": ["id"],
  "bookmark_properties": ["updated_at"]
}
```

### STATE Message

STATE messages contain the state that the Tap wishes to persist.
STATE messages have the following properties:


 - `value` **Required**. The JSON formatted state value

The semantics of a STATE value are not part of the specification,
and should be determined independently by each Tap.

## Example

```
{"type": "SCHEMA", "stream": "users", "key_properties": ["id"], "schema": {"required": ["id"], "type": "object", "properties": {"id": {"type": "integer"}}}}
{"type": "RECORD", "stream": "users", "record": {"id": 1, "name": "Chris"}}
{"type": "RECORD", "stream": "users", "record": {"id": 2, "name": "Mike"}}
{"type": "SCHEMA", "stream": "locations", "key_properties": ["id"], "schema": {"required": ["id"], "type": "object", "properties": {"id": {"type": "integer"}}}}
{"type": "RECORD", "stream": "locations", "record": {"id": 1, "name": "Philadelphia"}}
{"type": "STATE", "value": {"users": 2, "locations": 1}}
```

## Versioning

A Tap's API encompasses its input and output - including its
configuration, how it interprets state, and how the data it
produces is structured and interpreted. Taps should follow
[Semantic Versioning], meaning that breaking changes to any of
these should be a new MAJOR version, and backwards-compatible changes
should be a new MINOR version.

[JSON Schema]: http://json-schema.org/ "JSON Schema"
[Semantic Versioning]: http://semver.org/ "Semantic Versioning"
