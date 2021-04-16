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

When parsing datetimes or converting between timezones, the library
accepted for use by singer taps is `pytz`. This project has more accurate
timezone data than the builtin datetime.timezone.

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

When reasonable for the data source, it is preferable to write schemas as
explicitly as possible. For example:

1. Explicitly named fields in object schemas instead of
   `patternProperties` if they are well defined.
2. Explicit types associated with fields instead of `{}` if the data type
   is consistent and documented.
3. Specifying `additionalProperties: false` when the tap should fail if
   extra properties are added.

In general, any valid JSON Schema is acceptable, and there are use cases
that may work best when being more or less strict on validation. Targets
should handle valid JSON Schemas and provide a best effort approach to
load the data in a reasonable format.

**NOTE:** When modifying an existing schema, use extra caution when
adding properties such as `additionalProperties: false` or an explicit
type like `integer`. Schema changes such as this can be backwards
incompatible. In Singer, we use [SemVer](https://semver.org/) to the best
of our ability, so making the schema more restrictive (unless provable
beyond question) will result in a major version release that will require
all users of the tap to take special care when upgrading.

Naturally, a major version release as a result of a small field addition
is preferably avoided unless absolutely necessary (for example, if a
REST API changes the primary key's name).

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

## Dependency Versioning

When writing a new tap, all dependencies should be pinned to a specific
version. Loose dependency versions leave your tap susceptible to
dependency updates breaking functionality. Pinning dependencies to a
version ensure that the tap works the same way it did during development.

Good:

```
install_requires=[
    "singer-python==5.2.3"
]
```

Bad:
```
install_requires=[
    "singer-python>=5.2.3"
]
```

Worst:
```
install_requires=[
    "singer-python"
]
```

## Parent-Child Relationships

One of the most challenging API structures for ELT, common with REST APIs,
is the existence of a parent-child relationship. For example, resources
can form a tree like a structure of `organization -> project -> ticket ->
comment`, where each sub-object requires the ID of the parent object. A
request for `tickets` in this structure may look something like `GET
/organizations/{org_id}/projects/{project_id}/tickets`.

This is an efficient design for applications, but can pose challenges for
bulk extraction. The specific challenges are very specific to an
individual API's underlying implementation, but there are some best
practices that can provide guidance in many cases.

1. **Reduce Stream Dependencies** - Keep logic for child streams separate
   from parent streams. Each stream should be able to be extracted without
   requiring the parent to be selected and without complex boolean logic
   to determine what data to emit.
   - This opens up options in implementation that can provide efficiencies
     like only requesting the parent object's ID field for the child
     object's sync, and generally results in more readable code.
   - For REST APIs that provide a client library, the library functions
     usually try to make efficient use of this pattern by lazy loading the
     parent object and only making HTTP requests as necessary.
   - [Example
   tap-shopify](https://github.com/singer-io/tap-shopify/blob/v1.2.9/tap_shopify/streams/order_refunds.py#L28-L31)
2. **Signpost State Keeping** - Set a maximum bookmark value before the
   sync begins and store bookmarks only up to that value. For large data
   sets, records are being updated as the tap is requesting all of the
   data. As the tap reads records that may be changing in real
   time, setting a "signpost" bookmark as a
   maximum value to be stored can help with many race conditions and
   sorting issues. This signpost prevents the bookmark from advancing to
   a point where subsequent executions of the tap would miss records 
   that were modified while the extraction process was still running.
   - For timestamp bookmarks, this is usually something like
   `datetime.utcnow()`, but it can also be a value like a max event ID or
   log position queried before the sync begins.
   - [Date-Time Example
   tap-pardot](https://github.com/singer-io/tap-pardot/blob/v1.3.1/tap_pardot/streams.py#L231-L243)
   - [Log ID Example tap-postgres](https://github.com/singer-io/tap-postgres/blob/v0.2.0/tap_postgres/__init__.py#L650-L652)
3. **Limit State Growth** - Only store a constant set of bookmark keys for
   the deepest child object, rather than an unbounded list of bookmarks
   per parent-id. Along with #1 above, this pattern removes the need for
   API features such as update propagation or sorting based on certain
   fields. For many APIs and their backing data sources, these features
   just aren't feasible without a lot of rework.
   - Adding in bookmark keys at each level of the hierarchy trades
     complexity in tap code for a more efficient usage of the API
   - [Single Child Bookmark tap-square](https://github.com/singer-io/tap-square/blob/v1.3.0/tap_square/streams.py#L250)
   - [Complex Multi-Bookmark tap-trello](https://github.com/singer-io/tap-trello/blob/v1.0.0/tap_trello/streams.py#L356)
