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
[JSON Schema] spec.

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
 

[JSON Schema]: http://json-schema.org/
