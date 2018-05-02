# Sync Mode
Sync mode is the standard mode of operation for a tap.  During sync mode, the tap is expected to emit Schema, Record, and State messages to stdout.  The [SPEC](https://github.com/singer-io/getting-started/blob/master/SPEC.md) contains detailed explanations for each type of message.  

To run a tap in sync mode:
```bash
tap --config CONFIG [--state STATE] [--catalog CATALOG]
```

`CONFIG` is a required argument that points to a JSON file containing any
configuration parameters the tap needs.

`STATE` is an optional argument pointing to a JSON file that the
tap can use to remember information from the previous invocation,
like, for example, the point where it left off.

`CATALOG` is an optional argument pointing to a JSON file that the
tap can use to filter which streams should be synced.  See
the Discovery document for more info

## Config File
The configuration file contains whatever parameters the tap needs in order to pull data from the source. Typically this will include the credentials for the API or data source.  The format of the configuration will vary by tap, but it must be JSON-encoded and the root of the configuration must be an object.

### Special Fields

`start_date` should be used on first sync to indicate how far back to grab
records. Start dates should conform to the
[RFC3339](https://www.ietf.org/rfc/rfc3339.txt) specification.

`user_agent` should be set to something that includes a contact email
address should the API provider need to contact you for any reason.
(e.g. "Stitch (+support@stitchdata.com)")

### Example
Here is a basic config file:
```json
{
  "api_key" : "ABC123ASDF5432",
  "start_date" : "2017-01-01T00:00:00Z",
  "user_agent" : "Stitch (+support@stitchdata.com)"
}
```

## State File
State is a JSON map used to persist information between invocations of a tap. A tap that wishes to persist state should periodically write STATE messages to stdout as it processes the stream, and should expect the file named by the --state STATE argument to have the same format as the value of the STATE messages it emits.

A common use case of state is to record the spot in the stream where the last invocation left off. For this use case, the state will typically contain values like timestamps that correspond to `last-updated-at` fields from the source. If the Tap is invoked without a --state STATE argument, it should start at the beginning of the stream or at some appropriate default position. If it is invoked with a --state STATE argument it should read in the state file and start from the corresponding position in the stream.

### Bookmarks
State records serve to indicate a Tap's progress through a data source, but they can also provide more granular information about progress through individual streams.

If a Tap's streams can each have an individual state, the state output by the Tap should conform to the following format: the state object contains a top-level property "bookmarks" that maps to an object. The bookmarks object contains stream identifiers as property names, each of which maps to an object describing the respective stream's state.

### Example
Here is an example state file:
```json
{
  "bookmarks": {
    "orders": {
      "last_record": "2017-07-07T10:20:00Z"
    },
    "customers": {
      "last_record": 123
    }
  }
}
```
In the above example, there are two streams that have been bookmarked: `"orders"` and `"customers"`.
Each of those streams has some replication key field, like an `updated_at` timestamp or a sequential
ID for append-only sources. The replication key value persisted as part of the state message allows
the Tap to:

- bookmark its progress through the stream

- serve as a query parameter upon subsequent invocations, (e.g. the equivalent of `SELECT * from
  customers where id >= 123`)

The state record above indicates that the Tap last output an entry from the `"orders"` stream where
the source record's replication key field had the value "2017-07-07T10:20:00Z", and that it last
output an entry from the `"customers"` stream where the source record's replication key field had
the value 123.

### Things to keep in mind
- Write state as early and often as possible, but no sooner and no more often than is required. When a state is written, no data prior to that state will be synced, so do not update the state until all possible exceptions will be thrown.
- Endpoints that do not support filtering by updated timestamps or include updated timestamps do not support saving state.
- If the API supports filtering by updated timestamp, use that for filtering. If the API doesn't support filtering but does return updated timestamps with the data, filter by the timestamp before streaming the data.
- When streaming data in, stream the data in ascending order when possible.
- Jobs can be interrupted at any point. The saved state should never be invalid. Interrupted jobs that save state too early will have data missing. Interrupted jobs that save state too late will cause an increase in duplicate rows being replicated.

## Replication Methods
Taps can replicate data in one of two ways:
1. `INCREMENTAL` - Occurs when the tap saves it's progress via bookmarks or some other mechanism in the state. Only new or updated records are replicated during each sync.

2. `FULL_TABLE` - Occurs when the tap replicates all available records dating back to a `start_date`, defined in the config file, during every sync.

Taps can support both replication methods, and should decide which code path to use by checking the `replication-method` metadata for a stream during the sync.

## Stream/Field Selection
Taps should allow users to choose which streams and fields to replicate. The selection is driven by metadata.  The following metadata should be checked during a sync to decide whether a stream/field should be replicated:

| Metadata Keyword | Description  | 
| ----------------- | ------- |
| `inclusion` | Only applies to fields.  If this is set to `automatic`, the field should be replicated.  If this is set to `unsupported`, the field should not be replicated.  Can be written by a tap during discovery |
| `selected` | If this is set to `True`, the stream (empty breadcrumb), or field should be replicated.  If `False`, the stream or field should be omitted.  This metadata is written by services outside the tap. |
| `selected-by-default` | Only applies to fields.  If there is no `selected` metadata for a  field, this can be set to `True` or `False` to set a default behavior. Can be written by a tap during discovery |

## Metrics
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

#### Example 1: Timer for Successful HTTP GET

```
INFO METRIC: {"type": "timer", "metric": "http_request_duration", "value": 1.23, "tags": {"endpoint": "orders", "http_status_code": 200, "status": "succeeded"}}
```

> We made an HTTP request to an "orders" endpoint that took 1.23 seconds
> and succeeded with a status code of 200.

#### Example 2: Timer for Failed HTTP GET

```
INFO METRIC: {"type": "timer", "metric": "http_request_duration", "value": 30.01, "tags": {"endpoint": "orders", "http_status_code": 500, "status": "failed"}}
```

> We made an HTTP request to an "orders" endpoint that took 30.01 seconds
> and failed with a status code of 500.

#### Example 3: Counter for Records

```
INFO METRIC: {"type": "counter", "metric": "record_count", "value": 100, "tags": {"endpoint: "orders"}}
INFO METRIC: {"type": "counter", "metric": "record_count", "value": 100, "tags": {"endpoint: "orders"}}
INFO METRIC: {"type": "counter", "metric": "record_count", "value": 100, "tags": {"endpoint: "orders"}}
INFO METRIC: {"type": "counter", "metric": "record_count", "value": 14, "tags": {"endpoint: "orders"}}

```

> We fetched a total of 314 records from an "orders" endpoint.