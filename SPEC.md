# Stitch Streamer Specification
### Version 0.1

A *streamer* is an application that takes a *configuration* file and an
optional *state* file as input, and produces an ordered stream of *record*
and *state* messages as output. A *record* is json-encoded data of any
kind. A *state* object is a value that indicates an offset in the ordered
stream. A streamer may be implemented in any programming language.

Streamers are designed to produce a stream of data from sources like
databases and web service APIs for use in a data integration or ETL
pipeline.

## Input

### Synopsis

```
streamer COMMAND --config CONFIG [--state STATE]

COMMAND must be one of:

  sync - Stream data from the source and write it to stdout
  check - Quickly test whether we can access the source

CONFIG is a required argument that points to a JSON file containing any
configuration parameters the streamer needs.

STATE is an optional argument pointing to a JSON file that the streamer
can use to remember where it left off in the previous invocation. 
```

### Command

A streamer should support two modes of operation, `sync` and `check`. In
sync mode, the streamer connects to a data source and writes a stream of
records to stdout. In check mode, the streamer quickly attempts to
determine whether it can access the data source with the configuration
provided.

### Configuration

The configuration contains whatever parameters the streamer needs in order
to pull data from the source. Typically this will include the credentials
for the API or data source.

#### Examples

The format of the configuration will vary by streamer, but it must be
JSON-encoded and the root of the configuration must be an object. For
many sources, the configuration may just be a single value like an API
key. This should still be encoded as JSON. For example:

```json
{
  "api_key" : "ABC123ASDF5432"
}
```

### State

The state is used to persist information between invocations of a
streamer. The state must be encoded in JSON, but beyond that the structure
of the state is determined wholely by the streamer. A streamer that wishes
to persist state should periodically write STATE messages to stdout as it
processesses the stream, and should expect the file named by the `--state
STATE` argument to have the same format as the value of the STATE messages
it emits.

A common use case of state is to record the spot in the stream where the
last invocation left off. For this use case, the state will typically
contain values that like timestamps that correspond to "last-updated-at"
fields from the source. If the streamer is invoked without a `--state
STATE` argument, it should start at the beginning of the stream or at some
appropriate default position. If it is invoked with a `--state STATE`
argument it should read in the state file and start from the corresponding
position in the stream.

### Example invocations

#### Check the connection information

```bash
$ ./streamer check --config config.json
$ ruby streamer.rb check --config config.json
$ java -cp your.jar check com.yours.Streamer --config config.json
$ python streamer.py check --config config.json
```

#### Sync from the beginning

```bash
$ ./streamer sync --config config.json
$ ruby streamer.rb sync --config config.json
$ java -cp your.jar sync com.yours.Streamer --config config.json
$ python streamer.py sync --config config.json
```

#### Sync starting from a stored state

```bash
$ ./streamer sync --config config.json --state state.json
$ ruby streamer.rb sync --config config.json --state state.json
$ java -cp your.jar sync com.yours.Streamer --config config.json --state state.json
$ python streamer.py sync --config config.json --state state.json
```

## Output

A streamer outputs structured messages to `stdout` in JSON format, one
message per line. Logs and other information can be emitted to `stderr`
for aiding debugging. A streamer exits with a zero exit code on success,
non-zero on failure.

The body contains messages encoded as a JSON map, one message per
line. Each message must contain a `type` attribute. Any message `type`
is permitted, and `type`s are interpretted case-insensitively. The
following `type`s have specific meaning:

### RECORD

RECORD messages contain the data from the data stream. They must have
the following properties:

 - `record` **Required**. A JSON map containing a streamed data point

 - `stream` **Required**. The string name of the stream

A single streamer may output RECORDS messages with different stream
names.  A single RECORDS entry may only contain records for a single
stream.

Example:

```json
{"type": "RECORD", "stream": "users", "record": {"id": 0, "name": "Chris"}}
```

### SCHEMA

SCHEMA messages describe the structure of data in the stream. They
must have the following properties:
 
 - `schema` **Required**. A [JSON Schema] describing the
   `data` property of RECORDs from the same `stream`

 - `stream` **Required**. The string name of the stream that this
   schema describes

A single streamer may output SCHEMA messages with different stream
names.  If a RECORD message from a stream is not preceded by a
`SCHEMA` message for that stream, it is assumed to be schema-less.

Example:

```json
{"type": "SCHEMA", "stream": "users", "schema": {"properties":{"id":{"type":"integer"}}}, "record": {"id": 0, "name": "Chris"}}
```

### STATE

STATE messages contain a value that identifies the point in the
stream corresponding to the point at which the STATE entry appears
in the stream.  All RECORD messages prior to a STATE entry come
*before* or equal to the point in the stream described by the STATE
value, and all subsequent RECORDS entries come after or equal to the
point in the stream corresponding to the STATE value. STATE
messages have the following properties:

 - `value` **Required**. The JSON formatted state value

The semantics of a STATE value are not part of the specification,
and should be determined independently by each streamer.

## Example output

```
stitchstream/0.1
Content-Type: jsonline
--
{"type": "SCHEMA", "stream": "users", "schema": {"required": ["id"], "type": "object", "properties": {"id": {"key": true, "type": "integer"}}}}
{"type": "RECORD", "stream": "users", "record": {"id": 1, "name": "Chris"}}
{"type": "RECORD", "stream": "stream": "users", "record": {"id": 2, "name": "Mike"}}
{"type": "SCHEMA", "stream": "locations", "schema": {"required": ["id"], "type": "object", "properties": {"id": {"key": true, "type": "integer"}}}}
{"type": "RECORD", "stream": "locations", "record": {"id": 1, "name": "Philadelphia"}}
{"type": "STATE", "value": {"users": 2, "locations": 1}}
```

## Versioning

A streamer's API encompasses its input and output - including its
configuration, how it interprets state, and how the data it
produces is structured and interpretted. Streamers should follow
[Semantic Versioning], meaning that breaking changes to any of
these should be a new MAJOR version, and backwards-compatible changes
should be a new MINOR version.

## Packaging

TODO

## Resource Restrictions

TODO

## Security Restrictions

TODO

[JSON Schema]: http://json-schema.org/ "JSON Schema"
[Semantic Versioning]: http://semver.org/ "Semantic Versioning"
