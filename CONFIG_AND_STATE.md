# Config and State
The config and state files help provide context to Taps and Targets by providing authentication information, start times, and information about previous invocations.

## Config File
The config file contains whatever parameters the Tap needs in order to pull data from the source. Typically this will include the credentials for the API or data source.

#### Special Fields
`start_date` should be used on first sync to indicate how far back to grab
records. Start dates should conform to the
[RFC3339](https://www.ietf.org/rfc/rfc3339.txt) specification.

`user_agent` should be set to something that includes a contact email
address should the API provider need to contact you for any reason.
(e.g. "Stitch (+support@stitchdata.com)")

The format of the configuration will vary by Tap, but it must be
JSON-encoded and the root of the configuration must be an object. 

#### Example
Here is an example config file:

```json
{
  "api_key" : "ABC123ASDF5432",
  "start_date" : "2017-01-01T00:00:00Z",
  "user_agent" : "Stitch (+support@stitchdata.com)"
}
```

There is a single key provided to access the API, and the start_date indicates how far back to sync records from.

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
There are two streams that have been bookmarked: `"orders"` and `"customers"`.
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