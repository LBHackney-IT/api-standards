# Hackney API Standards

This is a DRAFT standard for building APIs in Hackney council - it is subject
to review and change.

Many of the standards set out in this document are based on the
[White House Web API Standards](https://github.com/WhiteHouse/api-standards),
with some specifics from the
[Google JSON StyleGuide](https://google.github.io/styleguide/jsoncstyleguide.xml#Date_Property_Values)

## Data Properties

- Property names should be in lower camel case
  - Good: `contactTelephoneNumber`
  - Good: `priority`

- Property names should use the language that Hackney uses rather than language used by any back end system:
  - Good: `sorCode`
  - Bad: `jobCode`
- Property names should use full words, not abbreviations
  - Good: `propertyReference`
  - Bad: `propertyRef`

## Data Formats

- Dates and Times should be formatted as RFC3339:
  - Don’t include milliseconds
  - Always use UTC (represented with Z)
  - Where just a date is required, still provide a full time object, with the time as 00:00:00
  - Example time: `"2017-11-06T16:34:41Z"`
  - Example date: `"2017-11-06T00:00:00Z"`
- Durations should be represented as a number of minutes:
  - Good: `"duration": 60`
- Numbers should be represented as JSON numbers. Zero-padded numeric identifiers are strings:
  - Good: `123`
  - Bad: `"123"`
  - Good: `"00003542"`
- Booleans should be represented as JSON booleans:
  - Good: `true`
  - Bad: `"true"`
  - Bad: `1`

## Requests

- POST, PUT and PATCH requests should pass the request parameters as a
  JSON-formatted body

## Responses

All responses should be provided in JSON format, with the Content-Type HTTP header set to application/json.

Successful requests should also have a 200 OK HTTP response in the case of GET requests.

201 Created or 202 Accepted are more applicable to POST,PUT and PATCH requests.

### Response bodies for Create/Update

POST, PUT and PATCH requests should return the full created or modified
resource in the response. This response should be identical to the response
returned by a request which fetches a single resource by ID.

An advantage of this approach is that clients of the API can treat the
response of creating a resource and showing that same resource in exactly the
same way.

This is the standard adopted by the
[GitHub](https://developer.github.com/v3/issues/comments/#create-a-comment),
[Stripe](https://stripe.com/docs/api#create_charge),
and
[Twilio](https://www.twilio.com/docs/api/rest/account#code-suspend-a-subaccount)
APIs.

### List responses

This section applies to requests which return a list of results.

Successful responses for requests which return multiple results should contain
a results key, which contains an array of results. Even queries that contain a
single item should return an array.

```json
{
  "results": [
    ...
  ]
}
```

## Pagination

Whee an endpoint returns many results, in order to prevent either the API
server or client from getting overwhelmed, the results should not be returned
in one go, instead being split between multiple pages.  You can request
multiple pages of results by incrementing the offset, e.g.

```
/households/?offset=50&limit=25
```
meaning: "return 25 results, starting at #50."

When returning a collection (i.e. more than one result from a query),
metadata should be included in the response, e.g:

```json
{
  "metadata": {
    "resultset": {
      "count": 1200,
      "offset": 125,
      "limit": 25
    }
  }
}
```

- count - Total number of items in the resulting collection
- offset - Skip the first n items
- limit - Return only a maximum n items

## Error Messages

In the event of an error, a response should always be returned in JSON format, with the correct HTTP status code:

- 404 - the resource could not be found
- 400 - the request could not be processed (e.g. because of incorrect syntax, required parameters were missing or were incorrectly formatted)
- 500 - the server was unable to respond to the request (internal server error)
- 504 - a timeout occurred when retrieving the requested data (gateway timeout)

Response:

```json
{
	"errors": [
		{
			"developerMessage": "....",
			"userMessage": "...."
		},
		{
			"developerMessage": "....",
			"userMessage": "...."
		}
		.....
	]
}
```

- developerMessage - an error message that contains information that is useful
  to a developer.
- userMessage - a more friendly error message that may be given to a user of
  the service that is consuming the API.
  - IMPORTANT - This should not leak information about the internals of the
    API or exactly what happened in one of the back-end systems

