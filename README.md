# Getting Started with Streams

Streams are the best way to get new data sources into Stitch.  A
Stream is produced by a Streamer application, and consumed by a
*Persister* application.

Persisters typically save data somewhere - anywhere - like S3, your
local filesystem, a queue or a database. We built the [Stitch
Persister](https://github.com/stitchstreams/persist-stitch) to send
data to the Stitch Import API.

Topics:

 - [Using Streams to get data into Stitch](#using-streams-to-get-data-into-stitch)
 - [Using Streams to get data into other destinations](#using-streams-to-get-data-into-other-destinations)
 - [Developing a Streamer](#developing-a-streamer)
 - [Running your Streamer from Stitch](#running-your-streamer-from-stitch)

## Using Streams to get data into Stitch

Stitch is built on top of the Streams framework, and soon you'll be
able to run your own custom Streamers from within the Stitch Platform,
which will take care of provisioning hardware, scheduling,
configuration and monitoring.  Until then, you can run a Streamer on
your own hardware with the Stitch Persister to send data to Stitch
from your machine.
[Drift](https://github.com/stitchstreams/stream-drift),
[Chargebee](https://github.com/stitchstreams/stream-chargebee),
[PagerDuty](https://github.com/stitchstreams/stream-pagerduty) and
[GitHub](https://github.com/stitchstreams/stream-github) are a few
examples of the data sources that have already been built with
Streams.

### Example - Sending GitHub data to Stitch with the GitHub Streamer

You can run the [GitHub
Streamer](https://github.com/stitchstreams/stream-github) with the
[Stitch Persister](https://github.com/stitchstreams/persist-stitch) to
send data about a GitHub repository activity to Stitch. The steps
required to do this are:

 - [Setup a Python 3 environment](#setup-a-python-3-environment)
 - [Install the Stitch Persister](#install-the-stitch-persister)
 - [Install the GitHub Streamer](#install-the-github-streamer)
 - [Create a new Stitch Connection for your GitHub data](#create-a-new-stitch-connection-for-your-github-data)
 - [Run the GitHub Streamer with the Stitch Persister](#run-the-github-streamer-with-the-stitch-persister)
 - [Save and Use Bookmarks](#save-and-use-bookmarks)

#### Setup a Python 3 environment

The Stitch Persister requires Python 3.5.0 or greater. You can check
your Python version with:

```bash
› python --version
Python 3.5.2
```

If you aren't on a compatible version, install the latest version of
Python 3.  If you're on macOS, we recommend using [homebrew][homebrew]
to `brew install python3`.  Installation instructions for other
platforms can be found
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

Confirm that pip is linked to your python 3 installation, and if not,
check to see if it's aliased to `pip3`.  Installation instructions can
be found in the [pip docs](https://pip.readthedocs.io/en/stable/) if
yours is missing.

#### Install the Stitch Persister

The Stitch Persister is available as a pip package.  Install it with:

```bash
› pip install persist-stitch
```

This installs an executable called `persist-stitch`.

#### Install the GitHub Streamer

Clone the [GitHub Streamer
repository](https://github.com/stitchstreams/stream-github) and
install its pip dependencies:

```bash
› git clone git@github.com:stitchstreams/stream-github
...
› cd stream-github
› pip install -r requirements.txt
```

#### Create a GitHub access token

Login to your GitHub account, go to the [Personal Access
Tokens](https://github.com/settings/tokens) settings page, and
generate a new token with at least the `repo` scope.  Temporarily
record this token somewhere secure, you'll need it in just a minute.

#### Create a new Stitch Connection for your GitHub data

Login to your [Stitch account](https://app.stitchdata.com) and
[generate an Stitch Import API
token](https://docs.stitchdata.com/hc/en-us/articles/223759228-Getting-Started-with-the-Import-API#accesstoken).
You'll need to provide a name for the connection - something like
'github' is probably appropriate - and temporarily record the token
somewhere secure.

#### Run the GitHub Streamer with the Stitch Persister

To run the streamer and persister together you'll need the following
values:

 - Stitch Client ID: This is the number in the URL when you login to Stitch
 - Stitch API Token: The token you saved temporarily when you created the Import API connection.
 - GitHub Token: The token you saved temporarily from GitHub
 - GitHub Repo Path: The relative path of the repository you want to pull activity data for. For example, to pull activity for this repository - http://github.com/stitchstreams/getting-started - the path would be `stitchstreams/getting-started`.

Add these values to your environment:

```bash
› export STITCH_CLIENT_ID=<stitch client id>
› export STITCH_TOKEN=<stitch token>
› export GITHUB_ACCESS_TOKEN=<github token>
› export GITHUB_REPO_PATH=<github repo path>
```

Then, running the Streamer and Persister together is as simple as:

```bash
› persist-stitch python stream_github.py
```

There are two parts to this command: running `persist-stitch`, and
passing it the command to run `stream_github.py`.  If successful,
you'll see output like this:

```bash
› persist-stitch python stream_github.py
INFO:root:Subprocess started 48434
2016-12-08 16:41:55,969 - root - INFO - Replicating all commits from stitchstreams/getting-started
2016-12-08 16:41:55,998 - requests.packages.urllib3.connectionpool - INFO - Starting new HTTPS connection (1): api.github.com
2016-12-08 16:41:56,558 - requests.packages.urllib3.connectionpool - DEBUG - "GET /repos/stitchstreams/getting-started/commits HTTP/1.1" 200 None
2016-12-08 16:41:56,613 - requests.packages.urllib3.connectionpool - DEBUG - "GET /repos/stitchstreams/getting-started/issues?sort=updated&direction=asc HTTP/1.1" 200 None
INFO:root:Waiting for process 48434 to complete
INFO:root:Subprocess 48434 returned code 0
INFO:root:Persisted 15 records to Stitch
{"issues": "2016-11-05T04:00:33Z", "commits": "2016-12-08T14:56:00Z"}
```

#### Save and Use Bookmarks

When `persist-stitch` is run as above, it writes log lines to
`stderr`, but `stdout` is reserved for outputting *bookmarks*. The
last line in the output above is an example of a bookmark - it
contains the date of the latest issue and commit extracted by the
streamer. A bookmark is a JSON-formatted string that is injected into
the data stream by the streamer, and then output to `stdout` by the
persister once it has persisted all data prior to the bookmark.

Streamers like the GitHub streamer can also accept a *FILE* argument
that, if present, points to a file containing the last persisted
bookmark value.  This enables Streams to work incrementally - the
bookmark checkpoints the last value that was persisted, and the next
time the streamer is run it picks up from that point.

To run the GitHub streamer incrementally, point it to a bookmark file
and capture the persister's `stdout` like this:

```bash
› persist-stitch python stream_github.py bookmark.out >> bookmark.out
```

Notice that we're appending the bookmark to the same bookmark file
we're reading in, meaning the latest bookmark will be the *last* line
in that file. The streamer is already configured to look only at the
last line, so that's OK, and this means the bookmark file will serve
as a log of the bookmarks that were used, which may be helpful for
debugging. It also means that the bookmark file will grow without
bounds, so you may occasionally want to archive or delete lines prior
to the last line in it.


## Using Streams to get data into other destinations

TODO

## Developing a Streamer

If you can't find an existing Streamer for your data source, then it's
time to build your own.

### Hello, world

A Streamer is just a program, written in any language, that outputs
data to *stdout* in the [Stream
format](https://github.com/StitchStreams/getting-started/blob/master/SPEC.md).
In fact, your first Streamer can be written from the command line,
without any programming at all:

```bash
› printf 'stitchstream/0.1\ncontent-type: jsonline\n--\n{"type":"RECORD","stream":"hello","record":{"value":"world"}}\n'
```

That data can be sent to Stitch by running the command from the Stitch
Persister:

```bash
› export STITCH_TOKEN=redacted
› export STITCH_CLIENT_ID=redacted
› persist-stitch printf 'stitchstream/0.1\ncontent-type: jsonline\n--\n{"type":"RECORD","stream":"hello","record":{"value":"world"}}\n'
...
INFO:root:Persisted 1 records to Stitch
```

### A Python Streamer

To move beyond "Hello, world" you'll need a real programming language.
Although any language will do, we have built a python library to help
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
Schema](http://json-schema.org/).

```python
with urllib.request.urlopen('http://icanhazip.com') as response:
    ip = response.read().decode('utf-8').strip()
    stitchstream.write_schema('my_ip', schema)
    stitchstream.write_records('my_ip', [{'timestamp': now,'ip': ip}])
```

Finally, we make the HTTP request, parse the response, and then make
two calls to the `stitchstream` library: first, to write the schema of
the `'my_ip'` stream, and then to write a record to that stream.

We can send this data to Stitch by running our new Streamer with the
Stitch Persister:

```bash
› persist-stitch python stream_ip.py
```

## Running your Streamer from Stitch

Soon, the Stitch Platform will be able to run *your* streamers.
You'll submit your code, along with a manifest describing how to run
it, and we'll do the rest - configuration, hardware provisioning,
scheduling, monitoring, and bookmark handling.  If you'd like to
participate in the closed beta of the Platform, contact us in our
[Slack channel][]