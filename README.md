# Getting Started with Stitch Streams

Stitch Streams are the best way to get new data sources into Stitch.
A Stitch Stream comprises two applications - a *Streamer* that
extracts data from a data source, and a *Persister* that saves the
data to a destination.

## Using an existing Streamer

### Running a Streamer

[Drift](https://github.com/stitchstreams/stream-drift),
[Chargebee](https://github.com/stitchstreams/stream-chargebee) and
[GitHub](https://github.com/stitchstreams/stream-github) are a few
examples of the data sources that have already been built with Stitch
Streams.

Start by installing the Streamer you intend to use, following the
instructions provided by that Streamer.  The Streamer will also have
instructions for being run.  For example, the GitHub Streamer is run with:

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
{"type": "RECORD", "key_fields": ["sha"], "stream": "commits", "record": {"author": {"id": 207186, "site_admin": false, "login": "cmerrick", "type": "User", "html_url": "https://github.com/cmerrick", ...}, "sha": "7c2f541396ff5b25d3f842ce617295ce50b027de", "url": "https://api.github.com/repos/StitchStreams/getting-started/commits/7c2f541396ff5b25d3f842ce617295ce50b027de", "commit": {"comment_count": 0, "author": {"date": "2016-11-01T17:15:02Z", "email": "cmerrick@rjmetrics.com", "name": "Christopher Merrick"}, "message": "add using bookmarks section", ...}, "committer": { ... }, ...}}
...
```

then, good news - it's working!

### Persisting to Stitch

But, streaming data into your console isn't very useful. To get the
data to Stitch, you need to install the [Stitch
Persister](https://github.com/stitchstreams/persist-stitch), following
the instructions in its repository.

Then, to send your Streamer's data to Stitch, [generate an Stitch
Import API
token](https://docs.stitchdata.com/hc/en-us/articles/223759228-Getting-Started-with-the-Import-API#accesstoken),
and pipe the output of the Streamer into the Stitch Persister, like
this:

```bash
› GITHUB_ACCESS_TOKEN=<token> GITHUB_REPO_PATH=<repo> python stream_github.py | persist-stitch -C <your Stitch client ID> -T <the Stitch import API token>
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
just a program, written in any language, that outputs data records and
bookmarks to *stdout* in the Stitch Stream format.  You can read a
detailed description of the format [here](format.html), but most of
what you need to know can be seen in this simple example, which is the
commit data from this repository produced by the
[stream-github](https://github.com/stitchstreams/stream-github)
Streamer:

```json
stitchstream/0.1
Content-Type: jsonline
--
{"type": "RECORD", "key_fields": ["sha"], "stream": "commits", "record": {"author": {"id": 207186, "site_admin": false, "login": "cmerrick", "type": "User", "html_url": "https://github.com/cmerrick", ...}, "sha": "7c2f541396ff5b25d3f842ce617295ce50b027de", "url": "https://api.github.com/repos/StitchStreams/getting-started/commits/7c2f541396ff5b25d3f842ce617295ce50b027de", "commit": {"comment_count": 0, "author": {"date": "2016-11-01T17:15:02Z", "email": "cmerrick@rjmetrics.com", "name": "Christopher Merrick"}, "message": "add using bookmarks section", ...}, "committer": { ... }, ...}}
{"type": "RECORD", "key_fields": ["sha"], "stream": "commits", "record": {"author": {"id": 207186, "site_admin": false, "login": "cmerrick", "type": "User", "html_url": "https://github.com/cmerrick", ...}, "sha": "4f961d199e8f96c4eb4c7bc071e5028b75640271", "url": "https://api.github.com/repos/StitchStreams/getting-started/commits/4f961d199e8f96c4eb4c7bc071e5028b75640271", "commit": {"comment_count": 0, "author": {"date": "2016-11-01T14:12:51Z", "email": "cmerrick@rjmetrics.com", "name": "Christopher Merrick"}, "message": "persisting to Stitch section", ...}, "committer": { ... }, ...}}
{"type": "RECORD", "key_fields": ["sha"], "stream": "commits", "record": {"author": {"id": 207186, "site_admin": false, "login": "cmerrick", "type": "User", "html_url": "https://github.com/cmerrick", ...}, "sha": "55b7d7590ba0f0f56e4e0d9bafccf549cb56744e", "url": "https://api.github.com/repos/StitchStreams/getting-started/commits/55b7d7590ba0f0f56e4e0d9bafccf549cb56744e", "commit": {"comment_count": 0, "author": {"date": "2016-10-31T17:29:13Z", "email": "cmerrick@rjmetrics.com", "name": "Christopher Merrick"}, "message": "using existing pt1", ...}, "committer": { ... }, ...}}
{"type": "RECORD", "key_fields": ["sha"], "stream": "commits", "record": {"author": {"id": 207186, "site_admin": false, "login": "cmerrick", "type": "User", "html_url": "https://github.com/cmerrick", ... }, "sha": "beb2d759464a067c44aebebe772c6e6c0f26c252", "url": "https://api.github.com/repos/StitchStreams/getting-started/commits/beb2d759464a067c44aebebe772c6e6c0f26c252", "commit": {"comment_count": 0, "author": {"date": "2016-10-31T16:55:48Z", "email": "cmerrick@rjmetrics.com", "name": "Christopher Merrick"}, "message": "first commit", ...}, "committer": { ... }, ...}}
{"type": "BOOKMARK", "value": "2016-11-01T17:15:02Z"}
```

You can see that there's a two line header, a separator, and a bunch
of JSON encoded data. JSON encoding is explicitly specified in the
`Content-Type` header, and `jsonline` is currently the only supported
data format (and we're working on providing support for richer data
types - stay tuned).  `jsonline` means that each message is on a
separate line, and there are two `type`s of message: `RECORD` and
`BOOKMARK`.

`RECORD` messages contain actual data points, and have
these properties:

 - `stream` is the name of the data stream, which will eventually get
   mapped to a table in your data warehouse.  A single Streamer can
   produce messages with any number of streams, and different data
   tables or collections in a data source are frequently separated
   into different streams.

 - `record` is the actual data point encoded as a JSON map

 - `key_fields` is an array of field names from the record that
   uniquely identify that record, analogous to the concept of a
   primary key on a database table.

`BOOKMARK` messages allow the Streamer to checkpoint its progress.
Use of BOOKMARKs is optional, but strongly encouraged for efficiency
and fault tolerance. When a persister encounters a BOOKMARK message,
it holds on to it, and then outputs that message once it has
successfully persisted all RECORDs messages that occurred prior to the
BOOKMARK. The structure of a BOOKMARK message's `value` property is
entirely up to the Streamer that creates it - as long as it is
JSON-encoded and below 1MB.  A BOOKMARK value should apply to the
*entire* Streamer, not an individual sub-stream, so if the Streamer
replicates multi sub-streams that all have different check points, it
is common to put the check point values into a map keyed by the stream
name.

### Example - the GitHub Streamer
