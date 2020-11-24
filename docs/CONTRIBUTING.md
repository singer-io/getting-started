# CONTRIBUTING DOCUMENT
This is the doc that will outline all requirements for contributors


Before submitting your tap for review, make sure that you have completed all of the following:

- [stream selection](#stream-selection)
- [field selection](#field-selection)
- [correct handling of child streams](#how-to-handle-child-streams)

## Stream Selection
Iterate over only selected streams.
The tap can get a list of the selected stream objects by calling `get_selected_streams()` on a singer-python `Catalog` object, as is done here:
 ```
selected_streams = catalog.get_selected_streams(state)

for stream in selected_streams:
    <sync stream>
 ```
(see [here](https://github.com/singer-io/tap-adroll/blob/138fc92dc4fb17c4b9446a3cf998b34b288b3e4a/tap_adroll/discover.py#L38) for an example)

## Field Selection
To filter a record's fields using the selected metadata from the catalog, the supported approach is to pass every record through the transformer with a metadata dictionary, as is done here:
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
(see [here](https://github.com/singer-io/tap-adroll/blob/138fc92dc4fb17c4b9446a3cf998b34b288b3e4a/tap_adroll/sync.py#L10) for an example)


## How to handle child streams
If there are child streams, this means that they rely on some piece of information from a corresponding parent.
To handle this, the tap must grab parent ids in child stream sync function
(see [here](https://github.com/singer-io/tap-adroll/blob/138fc92dc4fb17c4b9446a3cf998b34b288b3e4a/tap_adroll/streams.py#L55) in the ClientStream sync method)
