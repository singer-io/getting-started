# CONTRIBUTING DOCUMENT
This is the doc that will outline all requirements for contributors.

Before working on your project, please check that someone else has not already submitted the same project already.
The [Singer website][singer-io] and the [Singer Slack][singer-slack] are great resources for this.


Before submitting your tap for review, make sure that you have completed all of the following:

- [Implemented stream selection](#stream-selection)
- [Implemented field selection](#field-selection)
- [Handled child streams correctly](#how-to-handle-child-streams)

## Stream Selection
Iterate over only selected streams.

The tap can get a list of the selected stream objects by calling `get_selected_streams()` on a singer-python
`Catalog` object, as is done here:

```
selected_streams = catalog.get_selected_streams(state)

for stream in selected_streams:
    <sync stream>
 ```
(see [here][adroll-discovery] for an example)

## Field Selection
To filter a record's fields using the selected metadata from the catalog, the supported approach is to pass every
record through the transformer with a metadata dictionary, as is done here:

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

(see [here][adroll-sync] for an example)


## How to handle child streams
If there are child streams, this means that they rely on some piece of information from a corresponding parent. To
handle this, the tap must grab parent ids in child stream sync function.

(see [here][adroll-streams] in the ClientStream sync method)

<!-- Links -->
[singer-io]: https://www.singer.io/
[singer-slack]: https://singer-slackin.herokuapp.com/
[adroll-discovery]: https://github.com/singer-io/tap-adroll/blob/v1.0.0/tap_adroll/discover.py#L38
[adroll-sync]: https://github.com/singer-io/tap-adroll/blob/v1.0.0/tap_adroll/sync.py#L10
[adroll-streams]: https://github.com/singer-io/tap-adroll/blob/v1.0.0/tap_adroll/streams.py#L55
