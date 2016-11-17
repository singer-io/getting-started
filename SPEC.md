# Stitch Streamer Specification

A Stitch Streamer is an application that takes configuration and an
optional Bookmark as input, and produces an ordered stream of Records
and Bookmarks as output. A Record is text-encoded data of any kind. A
Bookmark is a value that indicates an offset in the ordered stream. A
Streamer may be implemented in any programming language.

Streamers are designed to produce a stream of data from sources like
databases and web service APIs for use in a data integration or ETL
pipeline.

## Input

### Configuration

A Streamer can accept configuration through environmental variables or
command line arguments. A Streamer can optionally accept one positional
command line argument corresponding to the path to a file containing
the lask Bookmark value.  No other positional command line arguments
are permitted.

Examples:

Good:

```bash
$ ./streamer
$ ruby streamer.rb
$ java -cp your.jar com.yours.Streamer
$ python streamer.py
$ python streamer.py /path/to/bookmark.file
$ DEBUG=1 python streamer.py -i 1 --token=abc /path/to/bookmark.file
```

Bad:

```bash
$ python streamer.py too many positional arguments
```

### Bookmarks

When a positional command line argument is provided, it must be a path
containing a Bookmark value.  When passed a Bookmark value, a Streamer
must only produce data with positions in the stream equal to or after
the position corresponding to that value. The structure and content of
a Bookmark value is determined entirely by the Streamer.  The file
containing the last Bookmark value should contain *only* that value,
and nothing else. A common Bookmark use case is a timestamp
corresponding to the latest modification date of the streamed data.


## Output

A Streamer outputs a header followed by structured messages to
`stdout` in JSON format, one message per line. Logs and other
information can be emitted to `stderr` for aiding debugging. A
Streamer exits with a zero exit code on success, non-zero on failure.

### Header

A Streamer must write a Header to `stdout` prior to any other
output. The first line of the Header MUST define the protocol version.
The last line of the Header MUST be “--”.  Lines in between define
properties of the stream in a “Key: value” format.  Permitted keys
are:

 - Content-Type (required): defines the encoding of the
   body. Permitted values are: jsonline.


### Body

The body contains messages encoded as a JSON map, one message per
line. Each message must contain a `type` attribute. Any message `type`
is permitted, and `type`s are interpretted case-insensitively. The
following `type`s have specific meaning:

#### RECORD

RECORD messages contain the data from the data stream. They must have
the following properties:

 - `record`: *Required*. A JSON map containing a streamed data point
 
 - `schema`: *Required*. A [JSON Schema][schema] describing the
   contents of `record`

 - stream: *Required*. The name of the stream

A single Streamer may output RECORDS messages with different stream
names.  A single RECORDS entry may only contain records for a single
stream.

Example:

```json
{"type": "RECORD", "stream": "users", "schema": {"properties":{"id":{"type":"integer"}}}, "record": {"id": 0, "name": "Chris"}]}
```

[schema]: http://json-schema.org/ "JSON Schema"

#### BOOKMARK

BOOKMARK messages contain a value that identifies the point in the
stream corresponding to the point at which the BOOKMARK entry appears
in the stream.  All RECORD messages prior to a BOOKMARK entry come
*before* or equal to the point in the stream described by the BOOKMARK
value, and all subsequent RECORDS entries come after or equal to the
point in the stream corresponding to the BOOKMARK value. BOOKMARK
messages have the following properties:

 - `value`: *Required*. The JSON formatted Bookmark value

The semantics of a BOOKMARK value are not part of the specification,
and should be determined independently by each Streamer.

### Example

```
stitchstream/0.1
Content-Type: jsonline
--
{"type": "RECORD", "schema": {"required": ["id"], "type": "object", "properties": {"id": {"key": true, "type": "integer"}}}, "stream": "users", "record": {"id": 1, "name": "Chris"}}
{"type": "RECORD", "schema": {"required": ["id"], "type": "object", "properties": {"id": {"key": true, "type": "integer"}}}, "stream": "users", "record": {"id": 2, "name": "Mike"}}
{"type": "RECORD", "schema": {"required": ["id"], "type": "object", "properties": {"id": {"key": true, "type": "integer"}}}, "stream": "locations", "record": {"id": 1, "name": "Philadelphia"}}
{"type": "BOOKMARK", "value": {"users": 2, "locations": 1}}
```

## Versioning

TODO

## Packaging

TODO

## Resource Restrictions

TODO

## Security Restrictions

TODO
