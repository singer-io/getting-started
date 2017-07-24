# Singer Specification

### Version 0.1

A *Tap* is an application that takes a *configuration* file and an
optional *state* file as input and produces an ordered stream of *record*,
*state* and *schema* messages as output. A *record* is json-encoded data
of any kind. A *state* message is used to persist information between
invocations of a Tap. A *schema* message describes the datatypes of
the *record*s in the stream. A Tap may be implemented in any
programming language.

Taps are designed to produce a stream of data from sources like
databases and web service APIs for use in a data integration or ETL
pipeline.

## Synopsis

```
tap --config CONFIG [--state STATE]

CONFIG is a required argument that points to a JSON file containing any
configuration parameters the Tap needs.

STATE is an optional argument pointing to a JSON file that the
Tap can use to remember information from the previous invocation,
like, for example, the point where it left off.

```

## Input

### Configuration

The configuration contains whatever parameters the Tap needs in order
to pull data from the source. Typically this will include the credentials
for the API or data source.

#### Special Fields

`start_date` should be used on first sync to indicate how far back to grab
records. Start dates should conform to the
[RFC3339](https://www.ietf.org/rfc/rfc3339.txt) specification.

`user_agent` should be set to something that includes a contact email
address should the API provider need to contact you for any reason.
(e.g. "Stitch (+support@stitchdata.com)")

#### Examples

The format of the configuration will vary by Tap, but it must be
JSON-encoded and the root of the configuration must be an object. For
many sources, the configuration may just be a single value like an API
key. This should still be encoded as JSON. For example:

```json
{
  "api_key" : "ABC123ASDF5432",
  "start_date" : "2017-01-01T00:00:00Z",
  "user_agent" : "Stitch (+support@stitchdata.com)"
}
```

### State

The state is used to persist information between invocations of a
Tap. The state must be encoded in JSON, but beyond that the
format of the state is determined wholely by the Tap. A Tap
that wishes to persist state should periodically write STATE messages
to stdout as it processes the stream, and should expect the file named
by the `--state STATE` argument to have the same format as the value
of the STATE messages it emits.

A common use case of state is to record the spot in the stream where the
last invocation left off. For this use case, the state will typically
contain values like timestamps that correspond to "last-updated-at"
fields from the source. If the Tap is invoked without a `--state
STATE` argument, it should start at the beginning of the stream or at some
appropriate default position. If it is invoked with a `--state STATE`
argument it should read in the state file and start from the corresponding
position in the stream.

### Example invocations

Sync from the beginning

```bash
$ ./tap --config config.json
```

Sync starting from a stored state

```bash
$ ./tap --config config.json --state state.json
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

### RECORD

RECORD messages contain the data from the data stream. They must have
the following properties:

 - `record` **Required**. A JSON map containing a streamed data point

 - `stream` **Required**. The string name of the stream

A single Tap may output RECORDs messages with different stream
names.  A single RECORD entry may only contain records for a single
stream.

Example:

```json
{"type": "RECORD", "stream": "users", "record": {"id": 0, "name": "Chris"}}
```

### SCHEMA

SCHEMA messages describe the datatypes of data in the stream. They
must have the following properties:

 - `schema` **Required**. A [JSON Schema](http://json-schema.org/) describing the
   `data` property of RECORDs from the same `stream`

 - `stream` **Required**. The string name of the stream that this
   schema describes

 - `key_properties` **Required**. A list of strings indicating which
   properties make up the primary key for this stream. Each item in the
   list must be the name of a top-level property defined in the schema. A
   value for `key_properties` must be provided, but it may be an empty
   list to indicate that there is no primary key.

A single Tap may output SCHEMA messages with different stream
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

STATE messages contain the state that the Tap wishes to persist.
STATE messages have the following properties:


 - `value` **Required**. The JSON formatted state value

The semantics of a STATE value are not part of the specification,
and should be determined independently by each Tap.

## Example:

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



# Data Types and Schemas

JSON is used to represent data because it is ubiquitous, readable, and
especially appropriate for the large universe of sources that expose data
as JSON like web APIs. However, JSON is far from perfect:

 - it has a limited type system, without support for common types like
   dates, and no distinction between integers and floating point numbers
 
 - while its flexibility makes it easy to use, it can also cause
   compatibility problems

*Schemas* are used to solve these problems. Generally speaking, a schema
is anything that describes how data is structured. In Streams, schemas are
written by streamers in *SCHEMA* messages, formatted following the
[JSON Schema](http://json-schema.org/) spec.

Schemas solve the limited data types problem by providing more information
about how to interpret JSON's basic types. For example, the [JSON Schema]
spec distinguishes between `integer` and `number` types, where the latter
is appropriately interpretted as a floating point. Additionally, it
defines a string format called `date-time` that can be used to indicate
when a data point is expected to be a
[properly formatted](https://tools.ietf.org/html/rfc3339) timestamp
string.

Schemas mitigate JSON's compatibility problem by providing an easy way to
validate the structure of a set of data points. Streams deploys this
concept by encouraging use of only a single schema for each substream, and
validating each data point against its schema prior to persistence. This
forces the streamer author to think about how to resolve schema evolution
and compatibility questions, placing that responsibility as close to the
original data source as possible, and freeing downstream systems from
making uninformed assumptions to resolve these issues.

Schemas are required, but they can be defined in the broadest terms - a
JSON Schema of '{}' validates all data points. However, it is a best
practice for streamer authors to define schemas as narrowly as possible.

## Schemas in Stitch

The Stitch persister and Stitch API use schemas as follows:

 - the Stitch persister fails when it encounters a data point that doesn't
   validate against its stream's latest schema
 - schemas must be an 'object' at the top level
 - Stitch supports schemas with objects nested to any depth, and arrays of
   objects nested to any depth - more info in the
   [Stitch docs](https://www.stitchdata.com/docs/data-structure/nested-data-structures-row-count-impact)
 - properties of type `string` and format `date-time` are converted to
   the appropriate timestamp or datetime type in the destination database
 - properties of type `integer` are converted to integer in the destination
   database
 - properties of type `number` are converted to decimal or numeric in the
   destination database
 - (soon) the `maxLength` parameter of a property of type `string` is used
   to define the width of the corresponding varchar column in the
   destination database
 - when Stitch encounters a schema for a stream that is incompatible with
   the table that stream is to be loaded into in the destination database,
   it adds the data to the
   [reject pile](https://www.stitchdata.com/docs/data-structure/identifying-rejected-records)
 
