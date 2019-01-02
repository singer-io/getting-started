# Discovery Mode
Discovery mode provides a way for a tap to describe the data streams it supports.  JSON schema is used to describe the structure and type of data for each stream.  The implementation of discovery mode will depend on the tap's data source. Some taps will hard code the schema for each stream, while others will connect to an API that provides a desription of the available streams.  When discovery mode is run, the tap should write to stdout a list of streams, known as the catalog, with each entry containing some basic information about the stream and a JSON schema describing the stream's data.

To run a tap in discovery mode, the `--discover` flag should be provided:
```bash
tap --config CONFIG --discover
```

Discovery is typically run with the output redirected to a file so it can be passed into the tap in sync mode:
```bash
tap --config CONFIG --discover > catalog.json
```

Note that some legacy taps use `properties.json` as the catalog.

## Schemas
JSON is used to represent data because it is ubiquitous, readable, and especially appropriate for the large universe of sources that expose data as JSON like web APIs. However, JSON is far from perfect:

- it has a limited type system, without support for common types like dates, and no distinction between integers and floating point numbers

-  while its flexibility makes it easy to use, it can also cause compatibility problems

*Schemas* are used to solve these problems. Generally speaking, a schema is anything that describes how data is structured. Schemas are written by Taps in *SCHEMA* messages, formatted following the [JSON Schema](http://json-schema.org/) spec.

Schemas solve the limited data types problem by providing more information
about how to interpret JSON's basic types. For example, the [JSON Schema](http://json-schema.org/) spec distinguishes between `integer` and `number` types, where the latter is appropriately interpretted as a floating point. Additionally, it defines a string format called `date-time` that can be used to indicate when a data point is expected to be a [properly formatted](https://tools.ietf.org/html/rfc3339) timestamp string.

Schemas mitigate JSON's compatibility problem by providing an easy way to
validate the structure of a set of data points. Taps deploy this
concept by encouraging use of only a single schema for each stream, and
validating each data point against its schema prior to persistence. This
forces the Tap author to think about how to resolve schema evolution
and compatibility questions, placing that responsibility as close to the
original data source as possible, and freeing downstream systems from
making uninformed assumptions to resolve these issues.

Schemas are required, but they can be defined in the broadest terms - a
JSON Schema of '{}' validates all data points. However, it is a best
practice for Tap authors to define schemas as narrowly as possible.

### Schemas in Stitch

The Stitch Target and Stitch API use schemas as follows:

 - the Stitch Target fails when it encounters a data point that doesn't
   validate against its stream's latest schema
 - schemas must be an 'object' at the top level
 - Stitch supports schemas with objects nested to any depth, and arrays of
   objects nested to any depth - more info in the
   [Stitch docs](https://www.stitchdata.com/docs/data-structure/nested-data-structures-row-count-impact)
 - References using the JSON Schema [`$ref` feature](https://tools.ietf.org/html/draft-zyp-json-schema-04#section-7) must be fully resolved and replaced before constructing a `SCHEMA` message. The spec does not support a method of passing extra schemas to serve as reference resolution.
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

### Example
Here is a basic example of a JSON schema:
```json
{
  "type": [
    "null", 
    "object"
  ],
  "additionalProperties": false,
  "properties": {
    "id": {
      "type": [
        "null",
        "string"
      ],
    },
    "name": {
      "type": [
        "null",
        "string"
      ],
    },
    "date_modified": {
      "type": [
        "null",
        "string"
      ],
      "format": "date-time",
    }
  }
}
```

## The Catalog
The output of discovery mode should be a list of the data streams a Tap supports.  This JSON formatted list is known as the catalog.  The top level is an object, with a single key called `"streams"` that points to an array of objects, each having the following fields:

| Property          | type               | required? | Description                    |
| ----------------- |--------------------|-----------|--------------------------------|
| `tap_stream_id`   | string             | required  | The unique identifier for the stream. This is allowed to be different from the name of the stream in order to allow for sources that have duplicate stream names. |
| `schema`          | object             | required  | The JSON schema for the stream.  |
| `table_name`      | string             | optional  | For a database source, the name of the table. |
| `metadata`        | array of metadata            | optional  | See metadata below for an explanation |

### Example
Here is an example catalog with one simple stream and no metadata:
```json
{
  "streams": [
    {
      "tap_stream_id": "users",
      "stream": "users",
      "schema": {
        "type": ["null", "object"],
        "additionalProperties": false,
        "properties": {
          "id": {
            "type": [
              "null",
              "string"
            ],
          },
          "name": {
            "type": [
              "null",
              "string"
            ],
          },
          "date_modified": {
            "type": [
              "null",
              "string"
            ],
            "format": "date-time",
          }
        }
      }
    }
  ]
}
```

## Metadata
Metadata is the preferred mechanism for associating extra information about nodes in the schema.

Certain metadata should be written _and_ read by a tap.  This metadata is known as `discoverable` metadata.  Other metadata will be written by other systems such as a UI and therefore should only be read by the tap.  This type of metadata is called `non-discoverable` metadata.

 A tap is free to write ANY type of metadata they feel is useful for describing fields in the schema, although several reserved keywords exist.  A tap that extracts data from a database should use additional metadata to describe the properties of the database.

|  Keyword                    |  Tap Type  |  Discoverable?      |  Description  |
| ----------------------------|------------|---------------------|---------------|
| `selected`                  | any        | non-discoverable    | Either `true` or `false`.  Indicates that this node in the schema has been selected by the user for replication. |
| `replication-method`        | any        | non-discoverable    | Either `FULL_TABLE`, `INCREMENTAL`, or `LOG_BASED`. The replication method to use for a stream.|
| `replication-key`           | any        | non-discoverable    | The name of a property in the source to use as a "bookmark".  For example, this will often be an "updated-at" field or an auto-incrementing primary key (requires `replication-method`).|
| `view-key-properties`       | database   | non-discoverable    | List of key properties for a database view. |
| `inclusion`                 | any        | discoverable        | Either `available`, `automatic`, or `unsupported`. </br></br>`available` means the field is available for selection, and the tap will only emit values for that field if it is marked with `"selected": true`. </br></br>`automatic` means that the tap will emit values for the field.  </br></br>`unsupported` means that the field exists in the source data but the tap is unable to provide it.|
| `selected-by-default`       | any        | discoverable        | Either `true` or `false`.  Indicates if a node in the schema should be replicated _if_ a user has not expressed any opinion on whether or not to replicate it.|
| `valid-replication-keys`    | any        | discoverable        | List of the fields that could be used as replication keys.|
| `schema-name`               | any        | discoverable        | The name of the stream.|
| `forced-replication-method` | any        | discoverable        | Used to force the replication method to either `FULL_TABLE` or `INCREMENTAL`.|
| `table-key-properties`      | database   | discoverable        | List of key properties for a database table.|
| `is-view`                   | database   | discoverable        | Either `true` or `false`.  Indicates whether a stream corresponds to a database view.|
| `row-count`                 | database   | discoverable        | Number of rows in a database table/view.|
| `database-name`             | database   | discoverable        | Name of database.|
| `sql-datatype`              | database   | discoverable        | Represents the datatype of a database column.|


Each piece of metadata has the following canonical shape:
```json
{
  "metadata" : {
    "selected" : true,
    "some-other-metadata" : "whatever"
  },
  "breadcrumb" : ["properties", "some-field-name"]
}
```

The breadcrumb object above defines the path into the schema to the node to which the metadata belongs.  Metadata for a stream will have an empty breadcrumb.

The `metadata` module in singer-python provides several utility functions for working with and writing metadata.

### Example 
Here is an example of the catalog from the previous section with metadata:
```json
{
  "streams": [
    {
      "tap_stream_id": "users",
      "stream": "users",
      "schema": {
        "type": ["null", "object"],
        "additionalProperties": false,
        "properties": {
          "id": {
            "type": [
              "null",
              "string"
            ],
          },
          "name": {
            "type": [
              "null",
              "string"
            ],
          },
          "date_modified": {
            "type": [
              "null",
              "string"
            ],
            "format": "date-time",
          }
        }
      },
      "metadata": [
        {
          "metadata": {
            "inclusion": "available",
            "table-key-properties": ["id"],
            "selected-by-default": true,
            "valid-replication-keys": ["date_modified"],
            "schema-name": "users",
          },
          "breadcrumb": []
        },
        {
          "metadata": {
            "inclusion": "automatic",
          },
          "breadcrumb": ["properties", "id"]
        },
        {
          "metadata": {
            "inclusion": "available",
            "selected-by-default": true,
          },
          "breadcrumb": ["properties", "name"]
        },
        {
          "metadata": {
            "inclusion": "automatic",
          },
          "breadcrumb": ["properties", "date_modified"]
        }
      ]
    }
  ]
}
```
