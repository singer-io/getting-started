# Getting Started with Streams

Streams are the best way to get new data sources into Stitch.  A
Stream is produced by a *streamer* application, and consumed by a
*persister* application.

Persisters typically save data somewhere - anywhere - like S3, your
local filesystem, a queue or a database. We built the [Stitch
persister] to send data to the Stitch Import API.

**Topics**

 - [Using Streams to get data into Stitch](#using-streams-to-get-data-into-stitch)
 - [Using Streams to get data into other destinations](#using-streams-to-get-data-into-other-destinations)
 - [Developing a streamer](#developing-a-streamer)
 - [Running your streamer from Stitch](#running-your-streamer-from-stitch)

## Using Streams to get data into Stitch

Stitch is built on top of the Streams framework, and soon you'll be
able to run your own custom streamers from within the Stitch Platform,
which will take care of provisioning hardware, scheduling,
configuration and monitoring.  Until then, you can run a streamer on
your own hardware, and connect it to the [Stitch persister] to send data
to Stitch.
[Drift](https://github.com/stitchstreams/stream-drift),
[Chargebee](https://github.com/stitchstreams/stream-chargebee),
[PagerDuty](https://github.com/stitchstreams/stream-pagerduty) and
[GitHub](GitHub streamer) are a few
examples of the data sources that have already been built with
Streams.

### Example - Sending GitHub data to Stitch with the GitHub streamer

You can run the [GitHub streamer] with the [Stitch persister] to send
data about a GitHub repository activity to Stitch. The steps required
are:

 - [Setup a Python 3 environment](#setup-a-python-3-environment)
 - [Install the Stitch persister](#install-the-stitch-persister)
 - [Install the GitHub streamer](#install-the-github-streamer)
 - [Create a GitHub access token](#create-a-github-access-token)
 - [Create a new Stitch Connection](#create-a-new-stitch-connection)
 - [Configure your environment](#configure-your-environment)
 - [Run the GitHub streamer with the Stitch persister](#run-the-github-streamer-with-the-stitch-persister)
 - [Save and Use State](#save-and-use-state)

#### Setup a Python 3 environment

The [Stitch persister] requires Python 3.5.0 or greater. You can check
your Python version with:

```bash
› python --version
Python 3.5.2
```

If you aren't on a compatible version, install the latest version of
Python 3.  If you're on macOS, we recommend using [homebrew] to `brew
install python3`.  Installation instructions for other platforms can
be found
[here](http://www.diveintopython3.net/installing-python.html).

Depending on how you perform the install, you might end up with Python
3 aliased as `python3` in your PATH.  Be sure to confirm your `python
--version`, and swap `python3` for `python` in the commands below if
necessary.

pip - the python package manager - should have been installed along
with python.  You can check for pip with:

```bash
› pip --version
pip 9.0.1 from /Users/cmerrick/.virtualenvs/stitchstream/lib/python3.5/site-packages (python 3.5)
```

Confirm that `pip` is linked to your Python 3 installation, and if not,
check to see if it's aliased to `pip3`.  Installation instructions can
be found in the [pip docs](https://pip.readthedocs.io/en/stable/) if
yours is missing.

#### Install the Stitch persister

The [Stitch persister] is available as a pip package.  Install it with:

```bash
› pip install persist-stitch
```

This installs an executable called `persist-stitch`.

#### Install the GitHub streamer

Clone the [GitHub streamer] repository and install it:

```bash
› git clone git@github.com:stitchstreams/stream-github
› cd stream-github
› python setup.py install
```

#### Create a GitHub access token

Login to your GitHub account, go to the [Personal Access
Tokens](https://github.com/settings/tokens) settings page, and
generate a new token with at least the `repo` scope.  Temporarily
record this token somewhere secure, you'll need it later.

#### Create a new Stitch Connection

Login to your [Stitch] account and [generate an Stitch Import API
token](https://docs.stitchdata.com/hc/en-us/articles/223759228-Getting-Started-with-the-Import-API#accesstoken).
You'll need to provide a name for the connection - something like
'github' is probably appropriate - and temporarily record the token
somewhere secure. Stitch will put the data into a schema of the same
name in your warehouse.  We recommend creating a new connection for
each streamer, to avoid table name collisions.

#### Configure the streamer

To run the streamer you'll need to create a JSON file containing the
access token you just created and the path to the repository to stream.
The repo path is relative to `https://github.com/`. For example the path
for this repository is `stitchstreams/stream-github`.

```json
{"access_token": "your-access-token",
 "repo_path": "path-to/repository"}
```

Create this file and name it something like `github-streamer-config.json`.

#### Configure the persister

To run the persister you'll need to create a file contianing your Stitch
client id and access token. For example:

```json
{"client_id": 1234,
 "token": "asdfasdfasdfasdfasdfasdfasdfasdfadsf"}
```

#### Run the GitHub streamer with the Stitch persister

Run the streamer and pipe the output into the persister, like this:

```bash
› python stream_github.py sync --config streamer-config.json | persist-stitch sync --config persister-config.json
```

There are two parts to this command: running `persist-stitch`, and
passing it the command to run the streamer.  If successful,
you'll see output like this:

```bash
> stream-github sync --config streamer-config.json | persist-stitch sync --config persister-config.json
  INFO Replicating all commits from StitchStreams/getting-started
  INFO Persisted batch of 41 records to Stitch
{"commits": "2017-01-17T20:32:05Z", "issues": null}
```

#### Save and Use State

When `persist-stitch` is run as above, it writes log lines to
`stderr`, but `stdout` is reserved for outputting *state*. The last
line in the output above is an example of a state message - it
contains the date of the latest issue and commit extracted by the
streamer, as a JSON-formatted map. This message is put into the data
stream by the streamer with a value that describes its location in the
stream. Read more about state in the [Stream format] documentation.

The Stitch persister writes the state message to `stdout` once it has
persisted all data that appeared in the stream before the state
message. Note that although the state message is sent into the Stitch
persister, the persister process doesn't actually store it anywhere or
send it to the Stitch API, it just repeats it back to *stdout*.

Streamers like the GitHub streamer can also accept a `--state` argument
that, if present, points to a file containing the last persisted
state value.  This enables streamers to work incrementally - the
state checkpoints the last value that was persisted, and the next
time the streamer is run it should pick up from that point.

To run the GitHub streamer incrementally, point it to a state file
and capture the persister's `stdout` like this:

```bash
› stream-github sync --config streamer-config.json --state state.json | persist-stitch >> state.json
```

## Using Streams to get data into other destinations

[Stitch] provides a hosted pipeline for loading data into multiple
data warehouse platforms. We wrote the Stitch persister to send data
to the Stitch API, so that Stitch can handle processing and loading
the data.  But, different persisters can be used to send streams to
different destinations - like files, S3 buckets, databases, queues,
and APIs. If you'd like to build a different persister, check out the
[Stitch persister] for guidance, and join our [Slack channel] to get
help from our team.

## Developing a streamer

If you can't find an existing streamer for your data source, then it's
time to build your own.

**Topics**:
 - [Hello, world](#hello-world)
 - [A Python streamer](#a-python-streamer)
 
### Hello, world

A streamer is just a program, written in any language, that outputs
data to *stdout* in the [Stream format]. In fact, your first streamer
can be written from the command line, without any programming at all:

```bash
› printf '{"type":"RECORD","stream":"hello","record":{"value":"world"}}\n'
```

This streams the datapoint `{"value":"world"}` to the "hello"
stream. That data can be sent to Stitch by running the command from
the [Stitch persister]:

```bash
› export STITCH_TOKEN=redacted
› export STITCH_CLIENT_ID=redacted
› printf '{"type":"RECORD","stream":"hello","record":{"value":"world"}}\n' | persist-stitch
...
INFO:root:Persisted 1 records to Stitch
```

### A Python streamer

To move beyond *Hello, world* you'll need a real programming language.
Although any language will do, we have built a Python library to help
you get up and running quickly.

Let's write a streamer called `stream_ip.py` that retrieves the
current public IP using icanhazip.com, and writes that data with a
timestamp.

First, install the `stitchstreams` helper library with `pip`:

```bash
› pip install stitchstreams
```

Then, open up a new file called `stream_ip.py` in your favorite editor.

```python
import stitchstream
import urllib.request
from datetime import datetime, timezone
```

We'll use the `datetime` module to get the current timestamp, the
`stitchstream` module to write data to `stdout` in the correct format,
and the `urllib.request` module to make a request to icanhazip.com.

```python
now = datetime.now(timezone.utc).isoformat()
schema = {'properties':	
	  {'ip': {'type': 'string'},
           'timestamp': {'type': 'string',
           	         'format': 'date-time'}}}
```

This sets up some of the data we'll need - the current time, and the
schema of the data we'll be writing to the stream formatted as a [JSON
Schema].

```python
with urllib.request.urlopen('http://icanhazip.com') as response:
    ip = response.read().decode('utf-8').strip()
    stitchstream.write_schema('my_ip', schema)
    stitchstream.write_records('my_ip', [{'timestamp': now,'ip': ip}])
```

Finally, we make the HTTP request, parse the response, and then make
two calls to the `stitchstream` library: first, to write the schema of
the `'my_ip'` stream, and then to write a record to that stream.

We can send this data to Stitch by running our new streamer with the
[Stitch persister]:

```bash
› python stream_ip.py | persist-stitch
```

## Running your streamer from Stitch

Soon, the Stitch Platform will be able to run *your* streamers.
You'll submit your code, along with a manifest describing how to run
it, and we'll do the rest - configuration, hardware provisioning,
scheduling, monitoring, and state handling.  If you'd like to
participate in the closed beta of the Platform, contact us in our
[Slack channel]

[Slack channel]: https://stitch-streams-slack.herokuapp.com/
[Stitch persister]: https://github.com/stitchstreams/persist-stitch
[GitHub streamer]: https://github.com/stitchstreams/stream-github
[JSON Schema]: http://json-schema.org/
[Stream format]: https://github.com/StitchStreams/getting-started/blob/master/SPEC.md
[homebrew]: http://brew.sh
[Stitch]: https://app.stitchdata.com
