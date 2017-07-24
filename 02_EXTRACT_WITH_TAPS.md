# 🍺 All about TAPS 🍺

## Taps extract data from any source and write that data to a standard stream in a JSON-based format.

Be Check out our [official](05_MAKE_IT_OFFICIAL.md) and [unofficial](04_COOL_UNOFFICIAL_CLUB.md) pages before creating your own since it might save you some time in the long run.

### Making Taps

If a tap for your use case doesn't exist yet have no fear! This documentation will help. Let's get started:
 
### 👩🏽‍💻 👨🏻‍💻 Hello, world

A Tap is just a program, written in any language, that outputs data to `stdout` according to the [Singer spec](07_SPEC.md). 

In fact, your first Tap can be written from the command line, without any programming at all:

```bash
› printf '{"type":"SCHEMA", "stream":"hello","key_properties":[],"schema":{"type":"object", "properties":{"value":{"type":"string"}}}}\n{"type":"RECORD","stream":"hello","schema":"hello","record":{"value":"world"}}\n'
```

This writes the datapoint `{"value":"world"}` to the *hello* stream along with a schema indicating that `value` is a string. 

That data can be piped into any Target, like the [Google Sheets Target], over `stdin`:

```bash
› printf '{"type":"SCHEMA", "stream":"hello","key_properties":[],"schema":{"type":"object", "properties":{"value":{"type":"string"}}}}\n{"type":"RECORD","stream":"hello","schema":"hello","record":{"value":"world"}}\n' | target-gsheet -c config.json
```

### 🐍🐍🐍 A Python Tap

To move beyond *Hello, world* you'll need a real programming language. Although any language will do, we have built a Python library to help you get up and running quickly. This is because Python is the defacto standard for data engineers or folks interested in moving data like yourself.

If you need help ramping up or getting started with Python there's fantastic community support [here](https://www.python.org/about/gettingstarted/).

Let's write a Tap called `tap_ip.py` that retrieves the current IP using icanhazip.com, and writes that data with a timestamp.

First, install the Singer helper library with `pip`:

```bash
› pip install singer-python
```

Then, open up a new file called `tap_ip.py` in your favorite editor.

```python
import singer
import urllib.request
from datetime import datetime, timezone
```

We'll use the `datetime` module to get the current timestamp, the
`singer` module to write data to `stdout` in the correct format, and
the `urllib.request` module to make a request to icanhazip.com.

```python
now = datetime.now(timezone.utc).isoformat()
schema = {
    'properties':   {
        'ip': {'type': 'string'},
        'timestamp': {'type': 'string', 'format': 'date-time'},
    },
}

```

This sets up some of the data we'll need - the current time, and the
schema of the data we'll be writing to the stream formatted as a [JSON
Schema].

```python
with urllib.request.urlopen('http://icanhazip.com') as response:
    ip = response.read().decode('utf-8').strip()
    singer.write_schema('my_ip', schema, 'timestamp')
    singer.write_records('my_ip', [{'timestamp': now, 'ip': ip}])
```

Finally, we make the HTTP request, parse the response, and then make
two calls to the `singer` library:

 - `singer.write_schema` which writes the schema of the `my_ip` stream and defines its primary key
 - `singer.write_records` to write a record to that stream

We can send this data to Google Sheets as an example by running our new Tap
with the Google Sheets Target:

```
› python tap_ip.py | target-gsheet -c config.json
```

Alternatively you could send it to a csv just as easy by doing this:

```
› python tap_ip.py | target-csv -c config.json
```

## To summarize the formula for pulling with a tap and sending to a target is:

```
› python YOUR_TAP_FILE.py -c TAP_CONFIG_FILE_HERE.json | TARGET-NAME -c TARGET_CONFIG_FILE_HERE.json
To summarize the formula for pulling with a tap and sending to a target is:

```
› python YOUR_TAP_FILE.py -c TAP_CONFIG_FILE_HERE.json | TARGET-TYPE -c TARGET_CONFIG_FILE_HERE.json
```

You might not always need config files, in which case it would just be:

```
› python YOUR_TAP_FILE.py | TARGET-NAME 
```

More simply the formula is:
```
› python YOUR_TAP_FILE.py | TARGET-TYPE 
```

This assumes your target is intalled locally. Which you can read more about by heading over to the [targets page](03_SEND_TO_TARGETS). 
