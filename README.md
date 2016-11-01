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
instructions for being run.  For example, the Drift streamer is run with:

```bash
› DRIFT_CLIENT_ID=<drift oauth client id> DRIFT_REDIRECT_URI=<drift oauth callback> DRIFT_USERNAME=<drift username> DRIFT_PASSWORD=<drift_password> python stream_drift.py
```

Notice that this Streamer reads OAuth credentials out of the
environment. This is a common pattern, and means that you'll have to
login to the service in question to retrieve those parameters.  As
much as possible, Streamers will also have instructions about where to
find these values.

Once you have the necessary parameters, you can run the streamer.  If
you see something like this:

```bash
stitchstream/0.1
Content-Type: transit
--
["^ ","type","RECORD","record",["^ ","orgId",9999,"name","Stitch Support","id",9999,"status","ENABLED","createdAt","~m1468435557000","openConversationCount",24,"totalConversationCount",616],"stream","inboxes","key_fields",["id"]]
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

```bash › DRIFT_CLIENT_ID=<Drift OAuth client ID>
DRIFT_REDIRECT_URI=<Drift OAuth callback> DRIFT_USERNAME=<Drift
username> DRIFT_PASSWORD=<Drift password> python stream_drift.py |
persist-stitch -C <your Stitch client ID> -T <the Stitch import API token
id> ```

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
