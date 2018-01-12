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

## List properties by postcode

Returns a list of properties matching the given criteria.

```
GET /v1/properties/
```

### Parameters

- postcode (required)

### Response

```json
{
  "metadata": {
    "resultset": {
      "count": 5,
      "offset": 0,
      "limit": 10
    },
  },
  "results": [
    {
      "propertyReference": "00032896",
      "postcode": "AB1 1AA",
      "address": "Ross Court 2"
    },
    {
      ...etc...
    }
  ]
}
```


## Fetch property by id

Return details of the given property

```
GET /v1/properties/:propertyReference/
```

### Parameters

- propertyReference: (required) - the Id of the household to retrieve

### Response

This should mostly mirror the individual results from the List properties by
ID endpoint, with the addition of the `maintainable` boolean. This is `true`
when `no_maint` in UH is `false`, and vice versa.

Properties with `maintainable = false` cannot have repairs raised against them.

```json
{
  "propertyReference": "00032896",
  "postcode": "AB1 1AA",
  "address": "Ross Court 2",
  "maintainable": true
}
```

## Create a new repair

Create a repair request, with or without a list of work orders

```
POST /v1/repairs
```

### Request

```json
{
  "propertyReference": "00078345",
  "problemDescription": "The fan is buzzing and sometimes not spinning at
  all",
  "priority": "N",
  "contact": {
    "name": "Al Smith",
    "telephoneNumber": "07876543210",
    "emailAddress": "al.smith@hotmail.com",
    "callbackTime": "8am - 12pm"
  },
  "workOrders": [
    {
       "sorCode":  "20090190"
    }
  ]
}
```

- propertyReference - A Hackney-specific code identifying the property
- problemDescription - A free-text description of the problem which needs to
  be fixed
- priority - A single-character representation of the repair priority:
  - N - Normal
  - U - Urgent
  - I - Immediate
  - E - Emergency
  - G -
  - Z -
  - V -
- contact - The person who should be contacted in relation to the repair
  - callbackTime (optional) - a time which this person has specified that they
    are available to be called back. Don't pass this key at all if it is not
    applicable
- workOrders (optional) - A list of the repair jobs which need to happen to
  fix the resident's problem. Don't pass this key at all if it is not
  applicable
  - sorCode - a "Schedule of Rates" code which describes the problem

### Response

A successful request should return HTTP 201 Created

The response contains the data which was submitted, plus some additional IDs
and data calculated based on the SOR code.

```json
{
  "repairRequestReference": "08912445",
  "propertyReference": "00078345",
  "problemDescription": "The fan is buzzing and sometimes not spinning at
  all",
  "priority": "N",
  "contact": {
    "name": "Al Smith",
    "telephoneNumber": "07876543210",
    "emailAddress": "al.smith@hotmail.com",
    "callbackTime": "8am - 12pm"
  },
  "workOrders": [
    {
      "workOrderReference": "20090190",
      "sorCode":  "20090190",
      "supplierReference": "00000127"
    }
  ]
}
```

- repairRequestReference - an identifier for the repair request made by the resident
- workOrderReference - an identifier for the work to be done by a contractor
- supplierReference - an identifier for the organisation who will carry out
  the work - e.g. the Hackney DLO

If the repair was created without work orders, an empty array of workOrders
should be returned (rather than no key or a null value)

## Retrieve a repair by reference

Retrieve a repair request, with or without a list of work orders

```
GET /v1/repairs/:repairRequestReference
```

### Parameters

- Repair request reference (required)

### Response

The response contains the same data as when the repair was created:

```json
{
"repairRequestReference": "08912445",
  "propertyReference": "00078345",
  "problemDescription": "The fan is buzzing and sometimes not spinning at
    all",
  "priority": "N",
  "contact": {
    "name": "Al Smith",
    "telephoneNumber": "07876543210",
    "emailAddress": "al.smith@hotmail.com",
    "callbackTime": "8am - 12pm"
  },
  "workOrders": [
    {
      "workOrderReference": "20090190",
      "sorCode":  "20090190",
      "supplierReference": "00000127"
    }
  ]
}
```

As above: if the repair was created without work orders, an empty array of
workOrders should be returned (rather than no key or a null value)

## Get appointment booked for a Work Order

Returns the appointment booked for a work order

```
GET /v1/work_orders/:workOrderReference/appointments/
```

### Parameters

- Work order reference (required)

### Response

```json
{
  "beginDate": "2017-10-18T08:00:00Z",
  "endDate": "2017-10-18T12:00:00Z",
}
```

## Book an appointment for a Work Order

Creates the appointment in DRS and returns the booked appointment

```
POST /v1/work_orders/:workOrderReference/appointments/
```

### Parameters

#### Specified in path

  - workOrderReference - Work order reference (required)

#### Specified in JSON body

  - beginDate - The start time for the desired appointment (required)
  - endDate - The end time for the desired appointment (required)

### Response

A successful response should book the appointment in DRS

```json
{
  "beginDate": "2017-10-18T08:00:00Z",
  "endDate": "2017-10-18T12:00:00Z",
}
```

## Get available appointments for Work Order

Returns a list of available appointments for a work order

```
GET /v1/work_orders/:workOrderReference/available_appointments/
```

### Parameters

- Work order reference (required)

### Response
A successful response should create the Work order in DRS and get the list of available appointments.

```json
{
  "metadata": {
    "resultset": {
      "count": 5,
      "offset": 0,
      "limit": 10
    },
  },
  "results": [
    {
      "beginDate": "2017-10-18T08:00:00Z",
      "endDate": "2017-10-18T12:00:00Z",
      "bestSlot": true
    },
    {
      "beginDate": "2017-10-18T12:00:00Z",
      "endDate": "2017-10-18T16:15:00Z",
      "bestSlot": false
    },
    {
      "beginDate": "2017-10-19T10:00:00Z",
      "endDate": "2017-10-19T14:30:00Z",
      "bestSlot": false
    },
    {
      "beginDate": "2017-10-20T08:00:00Z",
      "endDate": "2017-10-20T16:15:00Z",
      "bestSlot": false
    },
    {
      "beginDate": "2017-10-20T16:00:00Z",
      "endDate": "2017-10-20T18:00:00Z",
      "bestSlot": false
    },
    {
      ...etc...
    }
  ]
}
```
## Get account and address for housing residents

Returns a list of account and address information

```
Get /v1/accounts/verifyhousingaccountlogindetail?parisReference=123434470&postcode=E8 1HH
```

### Parameters
- Paris reference (required)
- Postcode (required)

### Response
A successful response should get a list of account and address information corresponding to the required parameters.

```json
{
  "results": [
     {
      "parisReferenceNumber": "123403470",
      "postcode": "E8 1HH",
      "address": "Maurice Bishop House"
     },
     {
      ...etc...
     }
   ]
 }
```

## Get housing transactions for residents

Returns a list of transactions

```
Get /v1/ transactions?tagReference=123456/01
```

### Parameters
- Tag Reference (required)

### Response
A successful response should get a list of transactions based on the tag reference provided.

```json
{
  "results": [
    {
      "tagReference": "123456/01",
      "propertyReference": "01234513",
      "transactionSid": null,
      "houseReference": "000123",
      "transactionType": "RPO",
      "postDate": "2017-11-28T00:00:00",
      "realValue": -10,
      "transactionID": "49ct627-e9d4-e711-8109-zzz71b7fe041",
      "debDesc": "PayPoint/Post Office"
    },
    {
      "tagReference": "123456/01",
      "propertyReference": "01234513",
      "transactionSid": null,
      "houseReference": "123456",
      "transactionType": "RTB",
      "postDate": "2017-11-27T00:00:00",
      "realValue": -110.95,
      "transactionID": "c4396c29-87o3-e711-8109-zzz71b7fe041",
      "debDesc": "Housing Benefit"
    },
    {
      ...etc...
    }
  ]
}
```

## Get payment agreement information for housing residents

Returns a list of payment agreement information

```
Get /v1/accountpaymentagreement?TagRef=12345/01
```

### Parameters
- Tag reference (required)

### Response
A successful response should get a list of payment agreement information corresponding to the given tag reference.
```json
{
  "results": [
     {
      "agreementAmount": "47.18",
      "agreementFrequency": "1",
      "agreementId": "471po9ia-dcd4-e711-8109-e00zzz7fe0bn"
     },
     {
      ...etc...
     }
   ]
 }
...

## Get account information for housing residents

```
Get  /v1/accounts/accountdetailsbyparisreference?parisReference=123409789
```

### Parameters
- Parisreferencenumber (required)

### Response
A successful response should get a list of account information corresponding to the given Parisreferencenumber.
```json
{
  "results": [
     {
      "propertyReferenceNumber": "1234528",
      "benefit": "30",
      "tagReferenceNumber": "123456/01",
      "accountid": "93d621ae-hgc7-e711-8111-70106faa6a11",
      "currentBalance": "45.35",
      "rent": "9.04",
      "housingReferenceNumber": "145656",
      "directdebit": null,
      "personNumber": null,
      "responsible": false,
      "title": "Mr",
      "forename": "Andy",
      "surname": "Benj"
     },
     {
      ...etc...
     }
   ]
 }

```
## Authenticate users based on Username and Password

Returns the user detail 
Get/v1/login/authenticatenhoofficers?username=uaccount&password=hackney
```
### Parameters
- Username (required)
- Password (required)

### Response
A successful response should get the detail of Authenticated neighbourhood officers corresponding to the required parameters.

```json
{
  "result": {
    "userId": "de98e4b6-15dc-e711-8115-701brfaabb11",
    "firstName": "Shweta",
    "surName": "Sandilya",
    "activeDirectoryUserName": "ssandilya"
  }
}

...
## Create a Service Request 
Creates the service request and returns the Service Request

POST v1/tenancymanagementinteractions/servicerequest
```
Specified in JSON body
### Parameters
contactId:- contact id of the customer
title:-title of the Service Request
description:-description pf the Service Request
subject:-Subject of the service request
housing_requestcallback: call back to the customer.

Specified in path
### Response
A successful response create interaction

{
  "id": "f0446d97-04f2-e711-810c-e0071b7fe041",
  "title": "Tenancy Management",
  "description": "Enquiry Created By Estate Officer",
  "contactId": "463adffe-61a5-db11-882c-0014c260c5fa",
  "parentCaseId": null,
  "subject": "c1f72d01-28dc-e711-8115-70106faa6a11",
  "createdDate": null,
  "enquiryType": null,
  "ticketNumber": "CAS-00105-P9R5R7",
  "requestCallback": true,
  "transferred": false,
  "createdBy": null,
  "childRequests": null
}

## Create a Tenancy Management Interaction
Creates the service request and Tenancy Management Interaction and returns the Tenancy Management Interaction

POST v1/tenancymanagementinteractions/createTenancyManagementInteraction
```
Specified in JSON body
### Parameters
contactId:- contact id of the customer
enquirySubject:-subject of the enquiry
estateOfficerId:-Estate OfficerId
subject :-subject of the interaction
adviceGiven:-advice given to the customer
estateOffice:-Estate Office name
source:-source of the enquiry
NatureofEnquiry :- Nature of Enquiry
CRMServiceRequest :-ServiceRequest object

Specified in path
Response
A successful response create interaction

{
"interactionId": "40021783-01f2-e711-810c-e0071b7fe041",
"contactId": "463adffe-61a5-db11-882c-0014c260c5fa",
"enquirySubject": "100000005",
"estateOfficerId": "284216e9-d365-e711-80f9-70106faaead1",
"subject": "c1f72d01-28dc-e711-8115-70106faa6a11",
"adviceGiven": "test 123",
"estateOffice": "5",
"source": "1",
"natureofEnquiry": "3",
"serviceRequest": {
"id": "1da17a83-01f2-e711-8111-70106faa6a31",
"title": "Tenancy Management",
"description": "Enquiry Created By Estate Officer",
"contactId": "463adffe-61a5-db11-882c-0014c260c5fa",
"parentCaseId": null,
"subject": "c1f72d01-28dc-e711-8115-70106faa6a11",
"createdDate": null,
"enquiryType": null,
"ticketNumber": "CAS-00104-N5S4C3",
"requestCallback": false,
"transferred": false,
"createdBy": null,
"childRequests": null
}
}

...
