# Proposals

These are changes to the spec that may be added in the future.

## Add structure detection and selection

### Use case

For some data sources, it would be desirable to provide a more where the
tap can detect what tables or fields are available, and to allow a user to
select which fields to retrieve.

### Solution

Add an optional `--discover` flag that causes the tap to print out a
"STRUCTURE" message describing the data structures that are available to
the tap. Add a `--structure STRUCTURE` option that indicates that the tap
should read the STRUCTURE file in and use that to determine which data structures to sync.

Support for these two options should be optional, and a tap should fail if
it is invoked with options it doesn't support.

#### Structure File

The structure is an optional configuration file that can be used to
configure which data structures the streamer captures in sync
mode. The structure must be encoded as JSON following the same format
as STRUCTURE messages, defined below, with an additional "is_synced"
key on every node.  If the "is_synced" value of a node is true, the
streamer should emit the data corresponding to that structure node
during sync, and it should not emit the data otherwise.  A streamer
that supports structure specification should fail if an invalid
structure is supplied.

#### Structure Message

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
{"type": "STRUCTURE",
 "structure": {
   "my_first_database": {
     "type": "database",
     "description": "",
     "supported": true,
     "children": {
       "my_first_databases_first_table": {
         "type": "table",
         "description": "",
         "supported": true,
         "children": {
           "id_column": {
             "type": "column",
             "description": "INT PRIMARY KEY",
             "supported": true
           },
           "string_column": {
             "type": "column",
             "description": "VARCHAR(255)",
             "supported": true
           }
         }
       }
     }
   }
 }
}
```

## Add connection check support

### Use case

For some sources it might be good to provide an option to quickly test
whether the authentication details provided in the config file are valid.
This would allow a user to determine whether they can authenticate without
actually initiating a sync.

### Solution

Add an optional `--check` flag. If the `--check` flag is provided, the tap
should attempt to authenticate, then print the details of the
authentication attempt to stderr and exit 0 on success or 1 on failure.
