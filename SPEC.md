# Stitch Streamer Specification
### Version 0.1

A *streamer* is an application that takes a *configuration* file and
optional *state* and *structure* files as input, and produces an
ordered stream of *record*, *state*, *schema* and *structure* messages
as output. A *record* is json-encoded data of any kind. A *structure*
message describes the data available to the streamer. A *state*
message is used to persist information between invocations of a
streamer. A *schema* message describes the datatypes of the *record*s
in the stream. A streamer may be implemented in any programming
language.

Streamers are designed to produce a stream of data from sources like
databases and web service APIs for use in a data integration or ETL
pipeline.

## Input

### Synopsis

```
streamer --config CONFIG [--state STATE] [--structure STRUCTURE]

CONFIG is a required argument that points to a JSON file containing any
configuration parameters the streamer needs.

STATE is an optional argument pointing to a JSON file that the
streamer can use to remember information from the previous invocation,
like, for example, the point where it left off.

STRUCTURE is an optional argument pointing to a JSON file containing
extra configuration about the structures that the streamer should
sync.

```

### Command

A streamer should support three modes of operation, `sync`, `check`
and `discover`. In sync mode, the streamer connects to a data source
and writes a stream of messages to stdout.

In check mode, the streamer quickly attempts to determine whether it
can access the data source with the configuration provided. For
example, for a streamer that sources from a web service, the `check`
command might simply read an API key from the config and fetch a
single record from the API to confirm that the API key is correct. A
streamer in check mode should not write any messages to stdout.

In discover mode, the streamer outputs messages about the structure of
the available data to stdout. For example, a streamer that sources
data from a database, would output the available schemas, tables, and
columns in discover mode. Support of the `discover` command is
optional, and streamers should fail if discover is invoked but not
supported.

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
streamer. The state must be encoded in JSON, but beyond that the
format of the state is determined wholely by the streamer. A streamer
that wishes to persist state should periodically write STATE messages
to stdout as it processes the stream, and should expect the file named
by the `--state STATE` argument to have the same format as the value
of the STATE messages it emits.

A common use case of state is to record the spot in the stream where the
last invocation left off. For this use case, the state will typically
contain values like timestamps that correspond to "last-updated-at"
fields from the source. If the streamer is invoked without a `--state
STATE` argument, it should start at the beginning of the stream or at some
appropriate default position. If it is invoked with a `--state STATE`
argument it should read in the state file and start from the corresponding
position in the stream.

### Structure

The structure is an optional configuration file that can be used to
configure which data structures the streamer captures in sync
mode. The structure must be encoded as JSON following the same format
as STRUCTURE messages, defined below, with an additional "is_synced"
key on every node.  If the "is_synced" value of a node is true, the
streamer should emit the data corresponding to that structure node
during sync, and it should not emit the data otherwise.  A streamer
that supports structure specification should fail if an invalid
structure is supplied.


### Example invocations

#### Sync from the beginning without structure selection

```bash
$ ./streamer --config config.json
$ ruby streamer.rb --config config.json
$ java -cp your.jar com.yours.Streamer --config config.json
$ python streamer.py --config config.json
```

#### Sync from the beginning with structure selection

```bash
$ ./streamer --config config.json --structure structure.json
$ ruby streamer.rb --config config.json --structure structure.json
$ java -cp your.jar com.yours.Streamer --config config.json --structure structure.json
$ python streamer.py --config config.json --structure structure.json
```

#### Sync starting from a stored state without structure selection

```bash
$ ./streamer --config config.json --state state.json
$ ruby streamer.rb --config config.json --state state.json
$ java -cp your.jar com.yours.Streamer --config config.json --state state.json
$ python streamer.py --config config.json --state state.json
```

#### Sync starting from a stored state with structure selection

```bash
$ ./streamer --config config.json --state state.json --structure structure.json
$ ruby streamer.rb --config config.json --state state.json --structure structure.json
$ java -cp your.jar com.yours.Streamer --config config.json --state state.json --structure structure.json
$ python streamer.py --config config.json --state state.json --structure structure.json
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

A single streamer may output SCHEMA messages with different stream
names.  If a RECORD message from a stream is not preceded by a
`SCHEMA` message for that stream, it is assumed to be schema-less.

Example:

```json
{"type": "SCHEMA",
 "stream": "users",
  "schema": {"properties":{"id":{"type":"integer"}}}, "record": {"id": 0, "name": "Chris"},
  "key_properties": ["id"]}
```

### STATE

STATE messages contain the state that the streamer wishes to persist.
STATE messages have the following properties:


 - `value` **Required**. The JSON formatted state value

The semantics of a STATE value are not part of the specification,
and should be determined independently by each streamer.

## Example:

```
stitchstream/0.1
Content-Type: jsonline
--
{"type": "SCHEMA", "stream": "users", "key_properties": ["id"], "schema": {"required": ["id"], "type": "object", "properties": {"id": {"type": "integer"}}}}
{"type": "RECORD", "stream": "users", "record": {"id": 1, "name": "Chris"}}
{"type": "RECORD", "stream": "stream": "users", "record": {"id": 2, "name": "Mike"}}
{"type": "SCHEMA", "stream": "locations", "key_properties": ["id"], "schema": {"required": ["id"], "type": "object", "properties": {"id": {"type": "integer"}}}}
{"type": "RECORD", "stream": "locations", "record": {"id": 1, "name": "Philadelphia"}}
{"type": "STATE", "value": {"users": 2, "locations": 1}}
```

### STRUCTURE

STRUCTURE messages describe the data available to the streamer. They
must have the following properties:
 
 - `structure` **Required**. A JSON object with string keys naming the
   top of the structure hierarchy, and values as objects with keys:
   - type: a string containing one of "database", "table", "column" or "other"
   - description: a string with descriptive information about the node
   - supported: a boolean indicating whether or not the streamer can sync this node
   - children: a map containing the next lower level of structure, formatted in the same way as the root

A STRUCTURE message should describe the entirety of the structure
available to the streamer.

Example, for a database:

```json
{"type": "STRUCTURE", "structure": {"my_first_database": {"type": "database", "description": "", "supported": true, "children": {"my_first_databases_first_table": {"type": "table", "description": "", "supported": true, "children": {"id_column": {"type": "column", "description": "INT PRIMARY KEY", "supported": true}, "string_column": {"type": "column", "description": "VARCHAR(255)", "supported": true}}}}}}}
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
