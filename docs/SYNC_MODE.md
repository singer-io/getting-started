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

Note: Some legacy taps use `--propertes PROPERTIES` instead of `--catalog CATALOG` where `PROPERTIES` points to the catalog.

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
        "selected": true,
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
