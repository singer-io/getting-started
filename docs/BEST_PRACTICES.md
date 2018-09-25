# Best Practices for Building a Singer Tap

Developers should try to adhere to these best practices to help maintain
consistency and quality between Singer Taps and Targets.

## Rate Limiting

Most APIs enforce rate limits. Taps should be written in a way that
respects these rate limits.

Rate limits can take the form of quotas (X requests per day), in which
case the tap should leave room for other use of the API, or short-term
limits (X requests per Y seconds). For short-term limits, the entire quota
can be used up and the tap should sleep when rate limited.

The singer-python library's utils namespace has a rate limiting decorator
for use.

## Memory Constraints

Taps should not rely on large volumes of RAM being available during run
and should strive to keep as little data in memory as required. Use
iterators/generators when available.


## Dates

All dates should use the RFC3339 format (which includes timezone offsets).
UTC is the preferred timezone for all data and should be used when
possible.

Good:
 - 2017-01-01T00:00:00Z (January 1, 2017 12:00:00AM UTC)
 - 2017-01-01T00:00:00-05:00 (January 1, 2017, 12:00:00AM America/New_York)

Bad:
 - 2017-01-01 00:00:00

When parsing datetimes or converting between timezones, the library accepted for use by singer taps is pytz. This project has more accurate timezone data than the builtin datetime.timezone.

## Logging and Exception Handling

During tap execution, log every URL + params that will be requested, but
be sure to exclude any sensitive information such as api keys and client
secrets. Log the progress of the sync (e.g. Starting entity 1, Starting
entity 2, etc.) When the API returns with an error, log the status code
and body.

If an error causes the tap or target to exit, log the error at the
CRITICAL or FATAL level just before exiting with a non-zero status. The
expectation is that if the tap or target fails, a calling script can look
for lines that begin with CRITICAL or FATAL in order to get the most
relevant error message. If the fatal error is one that the tap or target
explicitly raises, consider omitting the stack trace. However if it's an
Exception from an unknown origin, log the full stack trace in addition to
logging the message at the CRITICAL or FATAL level.

If an intermittent error is detected from the API, retry using an
exponential backoff (try using `backoff` library for Python). If an
unrecoverable error is detected, exit the script with a non-zero error
code (raise an exception or use `sys.exit(1)`)


## Module Structure

Source code should be in a module (folder with `__init__.py` file) and not
just a script (`module.py` file).


## Schemas

If the schema is static, it should be stored in a `schemas` folder under
the module directory as JSON files rather than as native dicts in the
source code.

Please avoid vague schemas:

1. Do not use the empty schema `{}`. An empty schema means the target will
   be unable to do validation or data type transformation.

2. Set `"additionalProperties": false` for all "object" schemas. If you do
   not specify `"additionalProperties": false`, the target will be unable
   to do any validation or data type transformation on properties that
   aren't defined in the schema.


## Code Quality

We recommend running pylint on your Tap and making sure that you
understand any issues it reports. You should get your code to the point
where pylint does not report any error-level messages.

When we import your Tap, we'll run it in CircleCI, and we'll include
Pylint in the test phase. The CircleCI buld will fail if Pylint finds any
issues, so we will need to account for all issues by either fixing them or
disabling the message in Pylint. We are flexible about which Pylint issues
are acceptable, but we generally run Pylint with some of the more pedantic
messages disabled. See any of our recently adopted taps for an example.
