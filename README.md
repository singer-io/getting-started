# Getting Started with Stitch Streams

Stitch Streams are the best way to get new data sources into Stitch.
A Stitch Stream comprises two applications - a *Streamer* that
extracts data from a data source, and a *Persister* that saves the
data to a destination.

## Using an existing Streamer

### Running a Streamer

[Drift](https://github.com/stitchstreams/stream-drift),
[Chargebee](https://github.com/stitchstreams/stream-chargebee),
[PagerDuty](https://github.com/stitchstreams/stream-pagerduty) and
[GitHub](https://github.com/stitchstreams/stream-github) are a few
examples of the data sources that have already been built with Stitch
Streams.

Start by following the installation instructions provided by the
Streamer you intend to use.  The Streamer will also have instructions
for being run.  For example, the GitHub Streamer is run with:

```bash
› GITHUB_ACCESS_TOKEN=<token> GITHUB_REPO_PATH=<repo> python stream_github.py
```

Notice that this Streamer reads a token out of the environment. This
is a common pattern, and means that you'll have to login to the
service in question to retrieve those parameters.  As much as
possible, Streamers will also have instructions about where to find
these values.

Once you have the necessary parameters, you can run the streamer.  If
you see something like this:

```bash
stitchstream/0.1
Content-Type: jsonline
--
{"type": "SCHEMA", "stream": "commits", "schema": {"required": ["sha"], "type": "object", "properties": {"sha": {"key": true, "type": "string"}}}}
{"type": "RECORD", "stream": "commits", "record": {"author": {"id": 207186, "site_admin": false, "login": "cmerrick", "type": "User", "html_url": "https://github.com/cmerrick", ...}, "sha": "7c2f541396ff5b25d3f842ce617295ce50b027de", "url": "https://api.github.com/repos/StitchStreams/getting-started/commits/7c2f541396ff5b25d3f842ce617295ce50b027de", "commit": {"comment_count": 0, "author": {"date": "2016-11-01T17:15:02Z", "email": "cmerrick@rjmetrics.com", "name": "Christopher Merrick"}, "message": "add using bookmarks section", ...}, "committer": { ... }, ...}}
...
```

then, good news - it's working!

### Persisting to Stitch

Streaming data into your console isn't very useful. To get the
data to Stitch, you need to install the [Stitch
Persister](https://github.com/stitchstreams/persist-stitch), following
the instructions in its repository.

To send your Streamer's data to Stitch, [generate an Stitch
Import API
token](https://docs.stitchdata.com/hc/en-us/articles/223759228-Getting-Started-with-the-Import-API#accesstoken),
and pipe the output of the Streamer into the Stitch Persister, like
this:

```bash
› STITCH_TOKEN=<token> STITCH_CLIENT_ID=<number> GITHUB_ACCESS_TOKEN=<token> GITHUB_REPO_PATH=<repo> persist-stitch python stream_github.py
```

In about 20 minutes or less, you'll have the data in your data
warehouse.

### Using Bookmarks

Many Streamers output Bookmark values to keep track of what data has
been replicated.  A common example of a Bookmark value is an *updated
at* field on a record - if a Streamer knows that it has replicated all
data prior to a certain value of the *updated at* field, then, the
next time it runs, it only needs to replicate the data updated after
that value.

Bookmark values are output to the same *stdout* stream that you saw
when you ran the Streamer.  When the Stitch Persister encounters a
Bookmark, it writes the value to *stdout* once it has successfully
persisted all prior records.  Streamers that support bookmarks will
take a filename containing the last bookmark as an input parameter. To
provide this value to the Streamer, pipe the Persister's stdout to
`tail -1` and redirect that to a file - that's the file you should
pass to the Streamer on the next run.

## Building a new Streamer

### Overview

If you can't find an existing Streamer for the data source you want to
replicate, then it's time to build your own Streamer.  A Streamer is
just a program, written in any language, that outputs data to *stdout*
in the Stitch Stream format.  You can read a detailed description of
the format [here](format.html), but most of what you need to know can
be seen in this simple example, which is the commit data from this
repository produced by the
[stream-github](https://github.com/stitchstreams/stream-github)
Streamer:

```json
stitchstream/0.1
Content-Type: jsonline
--
{"type": "SCHEMA", "stream": "commits", "schema": {"required": ["sha"], "type": "object", "properties": {"sha": {"key": true, "type": "string"}}}}
{"type": "RECORD", "stream": "commits", "record": {"author": {"id": 207186, "site_admin": false, "login": "cmerrick", "type": "User", "html_url": "https://github.com/cmerrick", ...}, "sha": "7c2f541396ff5b25d3f842ce617295ce50b027de", "url": "https://api.github.com/repos/StitchStreams/getting-started/commits/7c2f541396ff5b25d3f842ce617295ce50b027de", "commit": {"comment_count": 0, "author": {"date": "2016-11-01T17:15:02Z", "email": "cmerrick@rjmetrics.com", "name": "Christopher Merrick"}, "message": "add using bookmarks section", ...}, "committer": { ... }, ...}}
{"type": "RECORD", "stream": "commits", "record": {"author": {"id": 207186, "site_admin": false, "login": "cmerrick", "type": "User", "html_url": "https://github.com/cmerrick", ...}, "sha": "4f961d199e8f96c4eb4c7bc071e5028b75640271", "url": "https://api.github.com/repos/StitchStreams/getting-started/commits/4f961d199e8f96c4eb4c7bc071e5028b75640271", "commit": {"comment_count": 0, "author": {"date": "2016-11-01T14:12:51Z", "email": "cmerrick@rjmetrics.com", "name": "Christopher Merrick"}, "message": "persisting to Stitch section", ...}, "committer": { ... }, ...}}
{"type": "RECORD", "stream": "commits", "record": {"author": {"id": 207186, "site_admin": false, "login": "cmerrick", "type": "User", "html_url": "https://github.com/cmerrick", ...}, "sha": "55b7d7590ba0f0f56e4e0d9bafccf549cb56744e", "url": "https://api.github.com/repos/StitchStreams/getting-started/commits/55b7d7590ba0f0f56e4e0d9bafccf549cb56744e", "commit": {"comment_count": 0, "author": {"date": "2016-10-31T17:29:13Z", "email": "cmerrick@rjmetrics.com", "name": "Christopher Merrick"}, "message": "using existing pt1", ...}, "committer": { ... }, ...}}
{"type": "RECORD", "stream": "commits", "record": {"author": {"id": 207186, "site_admin": false, "login": "cmerrick", "type": "User", "html_url": "https://github.com/cmerrick", ... }, "sha": "beb2d759464a067c44aebebe772c6e6c0f26c252", "url": "https://api.github.com/repos/StitchStreams/getting-started/commits/beb2d759464a067c44aebebe772c6e6c0f26c252", "commit": {"comment_count": 0, "author": {"date": "2016-10-31T16:55:48Z", "email": "cmerrick@rjmetrics.com", "name": "Christopher Merrick"}, "message": "first commit", ...}, "committer": { ... }, ...}}
{"type": "BOOKMARK", "value": "2016-11-01T17:15:02Z"}
```

You can see that there's a two line header, a separator, and a bunch
of JSON encoded data. JSON encoding is explicitly specified in the
`Content-Type` header, and `jsonline` is currently the only supported
data format.  `jsonline` means that each message is on a separate
line, and there are three `type`s of message: `RECORD`, `SCHEMA` and
`BOOKMARK`.

`RECORD` messages contain actual data points, and have
these properties:

 - `stream` is the name of the data stream, which will eventually get
   mapped to a table in your data warehouse.  A single Streamer can
   produce messages with any number of streams, and different data
   tables or collections in a data source are frequently separated
   into different streams.

 - `record` is the actual data point encoded as a JSON map

`SCHEMA` messages describe the structure of `data` in `RECORD`
messages, and have these properties:

-  `stream` is the name of the data stream being described

- `schema` is a [JSON Schema](http://json-schema.org/) that describes
  the structure of of the data. The `schema` can be used to indicate
  data types (included extended types like date times), key fields,
  and required fields.

`BOOKMARK` messages allow the Streamer to checkpoint its progress.
Use of BOOKMARKs is optional, but strongly encouraged for efficiency
and fault tolerance. When a persister encounters a BOOKMARK message,
it holds on to it, and then outputs that message once it has
successfully persisted all RECORD messages that occurred prior to the
BOOKMARK. The structure of a BOOKMARK message's `value` property is
entirely up to the Streamer that creates it - as long as it is
JSON-encoded and below 1MB.  A BOOKMARK value should apply to the
*entire* Streamer, not an individual sub-stream, so if the Streamer
replicates multi sub-streams that all have different check points, it
is common to put the check point values into a map keyed by the stream
name.

### Example - the GitHub Streamer

Lets look at how these messages are created using the [GitHub
Streamer](https://github.com/stitchstreams/stream-github) - written in
Python - as an example. Although Streamers can be written in any
programming language that can send data to *stdout*, we intend to
publish libraries and guides for Python first, since it is commonly
used for data processing.

It's only about 75 lines of code, so lets walk through it starting at
the beginning:

```python
import os
import argparse
import logging
import requests
import stitchstream
```

The `os` and `argparse` modules are used to read environment variables
and command-line variables, respectively.  `requests` is a fantastic
Python HTTP library that we'll use to make requests to the GitHub API.
`stitchstream` is a library that we've written to make it easy to
write `RECORD` and `BOOKMARK` messages. Using the `logging` library is
a good practice, and helps ensure that we don't `print` any lines to
stdout that would corrupt our data output.

```python
session = requests.Session()
logger = logging.getLogger()
```

Initialize the logger and a [Requests
session](http://docs.python-requests.org/en/master/user/advanced/#session-objects).

```python
def get_env_or_throw(key):
    value = os.environ.get(key)

    if value == None:
        raise Exception('Missing ' + key + ' environment variable!')

    return value
```

This is a helper method to read environment variables, and fail hard
if one is not set.

```python
def configure_logging(level=logging.DEBUG):
    global logger
    logger.setLevel(level)
    ch = logging.StreamHandler()
    ch.setLevel(level)
    formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
    ch.setFormatter(formatter)
    logger.addHandler(ch)
```

This is a helper method to properly configure the logger.

```python
def authed_get(url):
    return session.request(method='get', url=url)

def authed_get_all_pages(url):
    while True:
        r = authed_get(url)
        yield r
        if 'next' in r.links:
            url = r.links['next']['url']
        else:
            break
```

These methods make requests to the GitHub API using the previously
established Requests session.  The `authed_get_all_pages` helper
method is a [generator](https://wiki.python.org/moin/Generators) that
automatically reads all pages of a paginated result set.

```python

commit_schema = {'type': 'object',
                 'properties': {
                     'sha': {
                         'type': 'string',
                         'key': True
                     },
                     'commit': {
                         'type': 'object',
                         'properties': {
                             'committer': {
                                 'type': 'object',
                                 'properties': {
                                     'date': {
                                         'type': 'string',
                                         'format': 'date-time'
                                     }
                                 }
                             }
                         }
                     }
                 },
                 'required': ['sha']
             }
```

This is a partial schema of the commit data returned by the GitHub
API.  It doesn't capture all of the properties of the data, but it
does ensure that the `sha` field is required and treated as a key, and
that the `commit.committer.date` field is treated like a date time,
meaning it will be loaded into a `timestamp` field in the data
warehouse.

```python
def get_all_commits(repo_path, since_date):
    query_string = '?since={}'.format(since_date) if since_date else ''
    latest_commit_time = None

    for response in authed_get_all_pages('https://api.github.com/repos/{}/commits{}'.format(repo_path, query_string)):
        commits = response.json()
        stitchstream.write_records('commits', commit_schema, commits)
        if not latest_commit_time:
            latest_commit_time = commits[0]['commit']['committer']['date']

    stitchstream.write_bookmark(latest_commit_time)
```

This is the meat - it calls out to the appropriate GitHub API
endpoint, using a `?since=` parameter to get only values that have
occurred since a certain date - which lays the groundwork for this
Streamer to use a bookmark.  It encodes the API response as JSON, and
uses the `stitchstream.write_records(stream_name, schema, record)`
function to write each record to *stdout* in the appropriate
format. After it has written all records, it writes a bookmark, with
the value of the latest commit time of the repository.  Since the
GitHub API responds with commits ordered with the most recent first,
that value is grabbed from the very first commit that the API returns.

```python

if __name__ == '__main__':
    parser = argparse.ArgumentParser(prog='GitHub Streamer')
    parser.add_argument('FILENAME', help='File containing the last bookmark value', nargs='?')
    args = parser.parse_args()

    configure_logging()

    access_token = get_env_or_throw('GITHUB_ACCESS_TOKEN')
    repo_path = get_env_or_throw('GITHUB_REPO_PATH')

    session.headers.update({'authorization': 'token ' + access_token})

    bookmark = None
    if args.FILENAME:
        with open(args.FILENAME, 'r') as file:
            for line in file:
                bookmark = line.strip()

    if bookmark:
        logger.info('Replicating commits since %s from %s', bookmark, repo_path)
    else:
        logger.info('Replicating all commits from %s', repo_path)

    get_all_commits(repo_path, bookmark)

```

Finally, the `main` function:

- reads the token and repository path out of the environment, setting the
token onto the Requests session to properly authenticate with the
GitHub API.

- if a bookmark file is passed in, reads the bookmark value out the last line of the file

- kicks off the `get_all_commits` function
