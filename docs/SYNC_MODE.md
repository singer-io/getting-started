# Sync Mode
Sync mode is the standard mode of operation for a Tap.  During sync mode, the Tap is expected to emit Schema, Record, and State messages to stdout.  The [SPEC](SPEC.md) contains detailed explanations for each type of message.

To run a Tap in sync mode:
```bash
tap --config CONFIG [--state STATE] [--catalog CATALOG]
```

- `CONFIG` is a required argument that points to a JSON file containing any
configuration parameters the Tap needs.

- `STATE` is an optional argument pointing to a JSON file that the
Tap can use to remember information from the previous invocation,
like, for example, the point where it left off.

- `CATALOG` is an optional argument pointing to a JSON file that the
Tap can use to filter which streams should be synced.

Note: Some legacy taps use `--properties PROPERTIES` instead of `--catalog CATALOG` where `PROPERTIES` points to the catalog.

## Streams
The [Catalog](DISCOVERY_MODE.md#the-catalog) provided to the Tap contains the streams that are available to sync.  Each stream's [metadata](DISCOVERY_MODE.md#metadata) contains information that can be used to control sync behavior.

## Replication Method
Taps can support two replication methods, and should decide which to use by checking the `replication-method` metadata for a stream, which should be one of the following:

- `INCREMENTAL` - The Tap saves it's progress via bookmarks or some other mechanism in the state. Only new or updated records are replicated during each sync.

- `FULL_TABLE` - The Tap replicates all available records dating back to a `start_date`, defined in the config file, during every sync.


## Stream/Field Selection
Taps should allow users to choose which streams and fields to replicate. The following metadata should be checked to decide whether a stream/field should be replicated:

| Metadata Keyword | Description  |
| ----------------- | ------- |
| `inclusion` | Only applies to fields.  If this is set to `automatic`, the field should be replicated.  If this is set to `unsupported`, the field should not be replicated.  Can be written by a tap during discovery |
| `selected` | If this is set to `True`, the stream (empty breadcrumb), or field should be replicated.  If `False`, the stream or field should be omitted.  This metadata is written by services outside the tap. |
| `available` | The field may or may not be `selected`. |

Streams shouldn’t be selected by default. We want them to have an `inclusion: available` property. Each field needs to have its own metadata object that labels the field as `inclusion: automatic` or `inclusion: available`.

## Suggested Pattern for implementing stream selection
You can get all the proper inclusion metadata during discovery by calling `get_standard_metadata()` as is done here:
```
 mdata = metadata.get_standard_metadata(
            schema=<schema>,
            key_properties=<key_properties>,
            valid_replication_keys=<replication_keys>,
            replication_method=<replication_method>,
        )
```
The `mdata` object can then be passed to the catalog object.

And then in the `sync` function, we'll want to iterate over only selected streams.
We can get a list of the selected stream objects by calling `get_selected_streams()` on a singer-python `Catalog` object, as is done here:
 ```
selected_streams = catalog.get_selected_streams(state)

for stream in selected_streams:
    <sync stream>
 ```

## Suggested Pattern for implementing field selection
You can get all the proper inclusion metadata during discovery by calling `get_standard_metadata()` as is done here:
```
 mdata = metadata.get_standard_metadata(
            schema=<schema>,
            key_properties=<key_properties>,
            valid_replication_keys=<replication_keys>,
            replication_method=<replication_method>,
        )
```
The `mdata` object can then be passed to the catalog object.

And then in the `sync` function, we'lll want to pass every record through the transformer, as is done here:
 ```
with Transformer() as transformer:
    for rec in stream_object.sync():
        singer.write_record(
            <tap stream id>,
            transformer.transform(
                <record>, <schema dict>, <metadata dict>,
            )
        )
```


## How to handle child streams
If there are child streams, this means that they rely on some piece of information from a corresponding parent.
if child stream `CHILD` relies on parent stream `PARENT`'s id, then if we sync `CHILD` without `PARENT` being selected, the python process will error out with an unclear error message.
The tap can do one of two things to handle this:
    - grab parent ids in child stream sync function:
      - in the function to sync `CHILD`, we can grab all id's for and iterate through that list. This will allow us to sync `CHILD` without selecting `PARENT`
    - log a clear error that parent stream `PARENT` must be selected if we want to sync `CHILD`
      - make it clear in the logs: "in order to sync `CHILD`, you must also select `PARENT`

#### Example of Stream/Field Selection
Here is an example catalog with a selected stream that has two fields selected and one field unselected
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
            ]
          },
          "name": {
            "type": [
              "null",
              "string"
            ]
          },
          "date_modified": {
            "type": [
              "null",
              "string"
            ],
            "format": "date-time",
            "selected": true,
          }
        }
      },
      "metadata": [
        {
          "breadcrumb": [],
          "metadata": {
            "selected": "true"
          }
        },{
          "breadcrumb": [
            "properties",
            "id"
          ],
          "metadata": {
            "inclusion": "automatic",
            "selected": "true"
          }
        },
        {
          "breadcrumb": [
            "properties",
            "name"
          ],
          "metadata": {
            "inclusion": "available",
            "selected": "false"
          }
        },
        {
          "breadcrumb": [
            "properties",
            "date_modified"
          ],
          "metadata": {
            "inclusion": "available",
            "selected": "true"
          }
        }
      ]
    }
  ]
}
```

### Legacy Stream/Field Selection (DEPRECATED)
Some legacy Taps handle stream and field selection by looking for `"selected": true` directly in the stream's schema in the catalog.json file (called properties.json in legacy taps).

## Metric Messages
A Tap should periodically emit structured log messages containing metrics about read operations. Consumers of the tap logs can parse these metrics out of the logs for monitoring or analysis.
```
INFO METRIC: <metrics-json>
```
`<metrics-json>` should be a JSON object with the following keys:

| Metric Key | Description | 
| ---------- | ----------- |
| `type` | The type of the metric. Indicates how consumers of the data should interpret the value field. There are two types of metrics: </br></br> `counter` - The value should be interpreted as a number that is added to a cumulative or running total </br></br> `timer` - The value is the duration in seconds of some operation. | 
| `metric` | The name of the metric. This should consist only of letters, numbers, underscore, and dash characters. For example, "http_request_duration".|
| `value` | The value of the datapoint, either an integer or a float. For example, "1234" or "1.234" | 
| `tags` | Mapping of tags describing the data. The keys can be any strings consisting solely of letters, numbers, underscores, and dashes. For consistency's sake, we recommend using the following tags when they are relevant.  Note that for many metrics, many of those tags will not be relevant. </br></br> `endpoint` - For a Tap that pulls data from an HTTP API, this should be a descriptive name for the endpoint, such as "users" or "deals" or "orders". </br></br> `http_status_code` - The HTTP status code. For example, 200 or 500. </br></br> `job_type` - For a process that we are timing, some description of the type of the job. For example, if we have a Tap that does a POST to an HTTP API to generate a report and then polls with a GET until the report is done, we could use a job type of "run_report".</br></br>`status` - Either "succeeded" or "failed". |

### Examples

Here are some examples of metrics and how those metrics should be
interpreted.

#### Timer for Successful HTTP GET

```
INFO METRIC: {"type": "timer", "metric": "http_request_duration", "value": 1.23, "tags": {"endpoint": "orders", "http_status_code": 200, "status": "succeeded"}}
```

> We made an HTTP request to an "orders" endpoint that took 1.23 seconds
> and succeeded with a status code of 200.

#### Timer for Failed HTTP GET

```
INFO METRIC: {"type": "timer", "metric": "http_request_duration", "value": 30.01, "tags": {"endpoint": "orders", "http_status_code": 500, "status": "failed"}}
```

> We made an HTTP request to an "orders" endpoint that took 30.01 seconds
> and failed with a status code of 500.

#### Counter for Records

```
INFO METRIC: {"type": "counter", "metric": "record_count", "value": 100, "tags": {"endpoint: "orders"}}
INFO METRIC: {"type": "counter", "metric": "record_count", "value": 100, "tags": {"endpoint: "orders"}}
INFO METRIC: {"type": "counter", "metric": "record_count", "value": 100, "tags": {"endpoint: "orders"}}
INFO METRIC: {"type": "counter", "metric": "record_count", "value": 14, "tags": {"endpoint: "orders"}}

```

> We fetched a total of 314 records from an "orders" endpoint.
