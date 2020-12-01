# TAP SUBMISSION REQUIREMENTS
This is the doc that will outline all requirements for tap contributors.

Before working on your project, please check that someone else has not submitted the same project already.
The [Singer website][singer-io] and the [Singer Slack][singer-slack] are great resources for this.


Before submitting your tap for review, make sure that you have completed all of the following:

- [Project Structure](#projectstructure)
- [Implemented stream selection](#stream-selection)
- [Implemented field selection](#field-selection)
- [Handled child streams correctly](#how-to-handle-child-streams)
- [Run pylint](#pylint)
- [Pypi](#pypi)
- [Contributor License Agreement](#contributor-license-agreement)

## Project Structure

The project should have the following structure:

```
tap-<tap_name>
├── LICENSE
├── MANIFEST.in
├── README.md
├── setup.py
├── tap_<tap_name>
│   ├── schemas
│   │   ├── <stream_name>.json
│   │   ├── <stream_name>.json
│   │   └── <stream_name>.json
│   ├── client.py
│   ├── discover.py
│   ├── __init__.py
│   ├── streams.py
│   └── sync.py
└── tests
    └── base.py
```
files/directories to note:
- `schemas`: This folder houses all the json schema files. Each stream has a json schema associated with it, as defined in this file.
- `client.py`: Calls to the api are made here.
- `discover.py`: Discovery code is here.
- `__init__.py`: The main function is in this file. Grab command line arguments and kick off discovery/sync here.
- `streams.py`: If using the class-based model, this is where all the stream classes and their corresponding functions live.
- `sync.py`: This is where the sync function is run


## Correct Catalog
In order for sync to run properly, the tap must correctly output a catalog during discovery.

The catalog structure is described [here][catalog]

An easy way to construct the catalog is by using the following call:

```
        catalog_entry = {
            "stream": stream_name,
            "tap_stream_id": stream_name,
            "schema": schema,
            "metadata": metadata.get_standard_metadata(
                schema=schema,
                key_properties=stream.key_properties,
                valid_replication_keys=stream.replication_keys,
                replication_method=stream.replication_method,
            ),
            "key_properties": stream.key_properties
        }
```
(see [here][adroll-discovery] for the full example)

## Stream Selection
It's required that the tap implements stream selection so that in the Stitch product the user has control over what endpoints are hit.

In order to implement stream selection, the tap must Iterate over only selected streams.

The tap can get a list of the selected stream objects by calling `get_selected_streams()` on a singer-python
`Catalog` object, as is done here:

```
selected_streams = catalog.get_selected_streams(state)

for stream in selected_streams:
    <sync stream>
 ```
(see [here][adroll-sync] for an example)

## Field Selection
It's required that the tap implements field selection so that in the Stitch product the user has control over which fields in the record are returned.

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

(see [here][adroll-transformer] for an example)


## How to handle child streams
If there are child streams, this means that they rely on some piece of information from a corresponding parent. To
handle this, the tap must grab parent ids in child stream sync function.

(see [here][adroll-streams] in the ClientStream sync method)


## Pylint

In order to submit the tap, it's required that pylint is passing at 100% using the following disables:
`broad-except,chained-comparison,empty-docstring,fixme,invalid-name,line-too-long,missing-class-docstring,missing-function-docstring,missing-module-docstring,no-else-raise,no-else-return,too-few-public-methods,too-many-arguments`

The following command should be run from the root of the repo:

```
pylint tap_<tap_name> --disable 'broad-except,chained-comparison,empty-docstring,fixme,invalid-name,line-too-long,missing-class-docstring,missing-function-docstring,missing-module-docstring,no-else-raise,no-else-return,too-few-public-methods,too-many-arguments'
```

If other disables are necessarys, use inline disables:

``` python
def get_format_values(self): # pylint: disable=no-self-use
    return []
```

## Pypi

In order to run in Stitch, we require sole ownership a Pypi repo using the specific naming convention: `tap-<tap-name>`

If you have a Pypi repo under this name, you will either need to:
   - add `singer_io` user as admin and remove yourself as admin
   - delete the Pypi repo

## Contributor License Agreement

To ensure the community can use the repo code, contributors are required to sign Singer CLA.

The CLA can be found [here][singer-cla]




<!-- Links -->
[singer-io]: https://www.singer.io/
[singer-slack]: https://singer-slackin.herokuapp.com/
[adroll-discovery]: https://github.com/singer-io/tap-adroll/blob/138fc92dc4fb17c4b9446a3cf998b34b288b3e4a/tap_adroll/discover.py#L38
[adroll-sync]: https://github.com/singer-io/tap-adroll/blob/138fc92dc4fb17c4b9446a3cf998b34b288b3e4a/tap_adroll/sync.py#L10
[adroll-transformer]: https://github.com/singer-io/tap-adroll/blob/138fc92dc4fb17c4b9446a3cf998b34b288b3e4a/tap_adroll/sync.py#L29
[trello-streams]: https://github.com/singer-io/tap-trello/blob/374b80eca4bfb1263699ea1c4cd746b04dba4320/tap_trello/streams.py#L187
[singer-cla]: https://stitch-cla-enforcer.herokuapp.com/
[catalog]: https://github.com/singer-io/getting-started/blob/master/docs/DISCOVERY_MODE.md#the-catalog
