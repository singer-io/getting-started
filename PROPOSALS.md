# Add structure detection and selection

STRUCTURE is an optional argument pointing to a JSON file containing
extra configuration about the structures that the streamer should
sync.

In discover mode, the streamer outputs messages about the structure of
the available data to stdout. For example, a streamer that sources
data from a database, would output the available schemas, tables, and
columns in discover mode. Support of the `discover` command is
optional, and streamers should fail if discover is invoked but not
supported.

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




# Add "check" mode

In check mode, the streamer quickly attempts to determine whether it
can access the data source with the configuration provided. For
example, for a streamer that sources from a web service, the `check`
command might simply read an API key from the config and fetch a
single record from the API to confirm that the API key is correct. A
streamer in check mode should not write any messages to stdout.

