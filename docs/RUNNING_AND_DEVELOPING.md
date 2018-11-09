# Running and Developing Singer Taps and Targets
Since most Singer Taps and Targets are written in Python, the examples given on this page will all be written in Python.  

## Running Singer with Python
To run Singer with Python, first make sure Python 3 is installed on your system or follow these installation instructions for [Mac](http://docs.python-guide.org/en/latest/starting/install3/osx/) or [Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-install-python-3-and-set-up-a-local-programming-environment-on-ubuntu-16-04).  

We recommend installing each Tap and Target in a separate Python virtual environment.  This will insure that you won't have conflicting dependencies between any Taps and Targets.  

### Running a Singer Tap
1. Create and activate a Python 3 virtual environment for the Tap, which we'll call `tap-foo`.  When you run this yourself, change the tap name in the angle brackets `< >`
```bash
python3 -m venv ~/.virtualenvs/<tap-foo>
source ~/.virtualenvs/<tap-foo>/bin/activate
```
2. Install the Tap using pip:
```bash
pip install <tap-foo>
```
3. Edit the Tap's [config file](CONFIG_AND_STATE.md#config-file) (tap_config.json) to include any necessary credentials or parameters.

4. If the tap supports [discovery mode](DISCOVERY_MODE.md), run it to obtain the catalog:
```bash
~/.virtualenvs/<tap-foo>/bin/<tap-foo> --config tap_config.json --discover > catalog.json
```
5. Depending on what features the Tap supports, you may need to add metadata in the catalog for [stream/field selection](SYNC_MODE.md#streamfield-selection) or [replication-method](SYNC_MODE.md#replication-method).  

6. Run the Tap in sync mode:
```bash
~/.virtualenvs/<tap-foo>/bin/<tap-foo> --config tap_config.json --catalog catalog.json
``` 
The output should consist of [SCHEMA](SPEC.md#schema-message), [RECORD](SPEC.md#record-message), [STATE](SPEC.md#state-message), and [METRIC](SYNC_MODE.md#metric-messages) messages.  

### Running a Singer Tap with a Singer Target
To run a Singer Tap with a Singer Target, follow steps 1-5 above to set up the Tap, and then continue with the following steps.

1. Create and activate a Python 3 virtual environment for the Target, which we'll call `target-bar`.  Again, when you run this yourself, change the target name in the angle brackets `< >`
```bash
python3 -m venv ~/.virtualenvs/<target-bar>
source ~/.virtualenvs/<target-bar>/bin/activate
```
2. Install the Target in its virtual environment using pip:
```bash
pip install <target-bar>
```
3. Edit the Target's config file (target_config.json) to include any necessary credentials or parameters.

4. Run the Singer Tap, pipe the output to the Singer Target, and save the final state emitted by the target for future runs:
```bash
~/.virtualenvs/<tap-foo>/bin/<tap-foo> --config tap_config.json --catalog catalog.json | ~/.virtualenvs/<target-bar>/bin/<target-bar> --config target_config.json >> state.json

tail -1 state.json > state.json.tmp && mv state.json.tmp state.json
```

### Example - Using Singer to populate Google Sheets
The [Google Sheets Target](https://github.com/singer-io/target-gsheet) can be combined with any Singer Tap to populate a Google Sheet with data. This example will use currency exchange rate data from the [Fixer.io Tap](https://github.com/singer-io/tap-fixerio). [Fixer.io](http://fixer.io) is a free API for current and historical foreign exchange rates published by the European Central Bank.

The steps are:
 1. [Activate the Google Sheets API](#step-1---activate-the-google-sheets-api)
 1. [Configure the Target](#step-2---configure-the-target)
 1. [Install](#step-3---install)
 1. [Run](#step-4---run)
 1. [Save State (optional)](#step-5---save-state-optional)

#### Step 1 - Activate the Google Sheets API

 (originally found in the [Google API docs](https://developers.google.com/sheets/api/quickstart/python))
 
 1. Use [this
 wizard](https://console.developers.google.com/start/api?id=sheets.googleapis.com) to create or select a project in the Google Developers Console and activate the Sheets API. Click Continue, then Go to credentials.

 2. On the **Add credentials to your project** page, click the
 **Cancel** button.

 3. At the top of the page, select the **OAuth consent screen**
 tab. Select an **Email address**, enter a **Product name** if not
 already set, and click the **Save** button.

 4. Select the **Credentials** tab, click the **Create credentials**
 button and select **OAuth client ID**.

 5. Select the application type **Other**, enter the name "Singer
 Sheets Target", and click the **Create** button.

 6. Click **OK** to dismiss the resulting dialog.

 7. Click the Download button to the right of the client ID.

 8. Move this file to your working directory and rename it `client_secret.json`.

#### Step 2 - Configure the Target

Created a file called `config.json` in your working directory,
following [config.sample.json](https://github.com/singer-io/target-gsheet/blob/master/config.sample.json). The required
`spreadsheet_id` parameter is the value between the "/d/" and the "/edit" in the URL of your spreadsheet. For example, consider the following URL that references a Google Sheets spreadsheet:

```
https://docs.google.com/spreadsheets/d/1qpyC0XzvTcKT6EISywvqESX3A0MwQoFDE8p-Bll4hps/edit#gid=0
```

The ID of this spreadsheet is `1qpyC0XzvTcKT6EISywvqESX3A0MwQoFDE8p-Bll4hps`.


#### Step 3 - Install
`target-gsheet` can be run with any [Singer Tap](https://singer.io) to move data from sources like [Braintree](https://github.com/singer-io/tap-braintree), [Freshdesk](https://github.com/singer-io/tap-freshdesk) and [Hubspot](https://github.com/singer-io/tap-hubspot) to Google Sheets. We'll use the [Fixer.io Tap]() - which pulls currency exchange rate data from a public data set - as an example.

These commands will install `tap-fixerio` and `target-gsheet` with pip in their own virtual environments:

```bash
# Install tap-fixerio in its own virtualenv
python3 -m venv ~/.virtualenvs/tap-fixerio
source ~/.virtualenvs/tap-fixerio/bin/activate
pip install tap-fixerio
deactivate

# Install target-gsheet in its own virtualenv
python3 -m venv ~/.virtualenvs/target-gsheet
source ~/.virtualenvs/target-gsheet/bin/activate
pip install target-gsheet
deactivate
```

#### Step 4 - Run

This command will pipe the output of `tap-fixerio` to `target-gsheet`, using the configuration file created in Step 2:

```bash
~/.virtualenvs/tap-fixerio/bin/tap-fixerio | ~/.virtualenvs/target-gsheet/bin/target-gsheet -c config.json
```

`target-gsheet` will attempt to open a new window or tab in your default browser to perform authentication. If this fails, copy the URL from the console and manually open it in your browser.

If you are not already logged into your Google account, you will be prompted to log in. If you are logged into multiple Google accounts, you will be asked to select one account to use for the
authorization. Click the **Accept** button to allow `target-gsheet` to access your Google Sheet.  You can close the tab after the signup flow is complete.

Each stream generated by the Tap will be written to a different sheet in your Google Sheet. For the [Fixer.io Tap](https://github.com/singer-io/tap-fixerio) you'll see a single sheet named `exchange_rate`.

#### Step 5 - Save State (optional)

Taps like the [Fixer.io Tap](https://github.com/singer-io/tap-fixerio) can also accept a `--state` argument that, if present, points to a file containing the last persisted State value.  This enables Taps to work incrementally - the State checkpoints the last value that was handled by the Target, and the next time the Tap is run it should pick up from that point.

To run the [Fixer.io Tap](https://github.com/singer-io/tap-fixerio) incrementally, point it to a State file and capture the persister's `stdout` like this:

```bash
~/.virtualenvs/tap-fixerio/bin/tap-fixerio | ~/.virtualenvs/target-gsheet/bin/target-gsheet -c config.json >> state.json

tail -1 state.json > state.json.tmp && mv state.json.tmp state.json
(rinse and repeat)
```

## Developing a Tap

If you can't find an existing Tap for your data source, then it's time
to build your own.

**Topics**:
 - [Hello, world](#hello-world)
 - [A Python Tap](#a-python-tap)
 - [Tap Template](#tap-template)
 
### Hello, world

A Tap is just a program, written in any language, that outputs data to
`stdout` according to the [Singer spec]. In fact, your first Tap can
be written from the command line, without any programming at all:

```bash
printf '{"type":"SCHEMA", "stream":"hello","key_properties":[],"schema":{"type":"object", "properties":{"value":{"type":"string"}}}}\n{"type":"RECORD","stream":"hello","schema":"hello","record":{"value":"world"}}\n'
```

This writes the datapoint `{"value":"world"}` to the *hello*
stream along with a schema indicating that `value` is a string. 
That data can be piped into any Target, like the [Google Sheets
Target], over `stdin`:

```bash
printf '{"type":"SCHEMA", "stream":"hello","key_properties":[],"schema":{"type":"object", "properties":{"value":{"type":"string"}}}}\n{"type":"RECORD","stream":"hello","schema":"hello","record":{"value":"world"}}\n' | target-gsheet -c config.json
```

### A Python Tap

To move beyond *Hello, world* you'll need a real programming language.
Although any language will do, we have built a Python library to help
you get up and running quickly.

Let's write a Tap called `tap_ip.py` that retrieves the current
 IP using icanhazip.com, and writes that data with a timestamp.

First, install the [Singer helper library](https://github.com/singer-io/singer-python) with `pip`:

```bash
pip install singer-python
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
Schema](http://json-schema.org/).

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

We can send this data to Google Sheets by running our new Tap
with the [Google Sheets Target](https://github.com/singer-io/target-gsheet):

```bash
python tap_ip.py | target-gsheet -c config.json
```

### Tap Template

The python-based [singer-tap-template](https://github.com/singer-io/singer-tap-template) provides some skeleton code for a new tap, so you can spend less time on setup and more time coding!

## Developing a Target
A Singer Target should read lines from `stdin` and process [Schema](SPEC#schema-message), [Record](SPEC#record-message), and [State](SPEC#state-message) Messages.

### Guidelines
- Schema validation should be performed to ensure that the records being written for a stream comform to the schema defined for that stream.
- Write State messages to stdout once all data that appeared in the stream before the State message has been processed by the Target. Note that although the State message is sent into the target, in most cases the Target's process won't actually store it anywhere or do anything with it other than repeat it back to stdout.
- Targets should be able to handle records with nested fields.  Depending on the type of Target, the record may need to be flattened (de-nested).  See target-csv's [flatten()](https://github.com/singer-io/target-csv/blob/1f4ac68cf78a16f9107de0c615809bb26b15a32d/target_csv.py#L28) function for an example.

### Target Template
The python-based [singer-target-template](https://github.com/singer-io/singer-target-template) provides some skeleton code for a new target, so you can spend less time on setup and more time coding!
