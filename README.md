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
```

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

Returns the user details 

```
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
    "estateOfficerLoginId": "d1o1o1o1o1o-15dc-e711-8115-7be3faabb11",
    "officerId": "d1b1b1b1b1b-15dc-e711-8115-7be3faabb11",
    "firstName": "Shweta",
    "surName": "Sandilya",
    "username": "ssandilya",
    "fullName": "Shweta Sandilya",
    "isManager": false,
    "areaManagerId": null,
    "officerPatchId": "d1c1c1c1c-15dc-e711-8115-7be3faabb11"    
  }
}
   
```

## Get deb items for housing residents

Returns the deb items for a housing resident based on their tag reference. 

```
Get/v1/accounts/getdebitemsbytagreference?tagreference=1234567/89
```
### Parameters
- Tag reference (required)

### Response
A successful response should return a list of deb items corresponding to the required parameters.

```json
{
  "results": [
  {
    "currentBalance": "123456879-15dc-e711-8115-7be3faabb11",
    "debItemCode": "DGR",
    "debItemValue": 1.25,
    "effectiveDate": "2017-01-01T00:00:00Z",
    "propertyReference": "00012345",
    "tagReference": "1234567/89",
    "termDate": "2018-12-12T00:00:00Z"
  },
  {
   ...etc...
  }
 ]
}
   
```

## Get all accounts and notifications

Returns all lease or rent accounts and their notifications entries based on the account type. 

```
Get/v1/accounts/getaccountsandnotifications?type=1
```
### Parameters
- Type (required)

### Response
A successful response should return a list all lease or rent accounts and their notification entries information based on the chosen account type.

```json
{
  "results": [
  {
    "currentBalance": "-29.99",
    "telephone": "0987654321",
    "areNotificationsOn": true,
    "paymentAgreementId": null,
    "paymentAgreementEndDate": null   
  },
  {
   ...etc...
  }
 ]
}
   
```


## Get all estate officers for an area

Returns a list of all estate officers for a given area. 

```
Get/v1/areapatch/getallofficersperarea?areaId=1
```
### Parameters
- Area ID (required)

### Response
A successful response should return a list all officers for a given area.

```json
{
  "results": [
  {
    "propertyAreaPatchId": "1234527c-b205-1231-811c-71234faa1234",
    "estateOfficerPropertyPatchId": "bo11oo1o44-b005-e811-811c-12126faa1212",
    "estateOfficerPropertyPatchName": "Test Officer Patch Name",
    "llpgReferenece": "12345678901",
    "patchId": "1234545f-b4f7-e711-1234-12345faa6a31",
    "patchName": "Test Patch",
    "propetyReference": "010203040",
    "wardName": "Test Ward Name",
    "wardId": 0,
    "areaName": "Test Area Name",
    "areaId": 0,
    "managerPropertyPatchId": "b2o2o2o44-b005-e811-811c-12126faa1212",
    "managerPropertyPatchName": "Test Manager Name",
    "areaManagerName": "Test Area Manager",
    "areaManagerId": "a1a1a1a1a-b4f7-e711-1234-12345faa6a31",
    "isaManager": false,
    "officerId": "a1c1c1c1c-b4f7-e711-1234-12345faa6a31",
    "officerName": "Test Officer Name"
  },
  {
   ...etc...
  }
 ]
}
   
```


## Get citizen index search result

Returns a list of citizens corresponding to a search parameter. 

```
Get/v1/citizenindexsearch?firstname=test&surname=test&addressline12=flat%201%20test&postcode=E82HH
```
### Parameters
- First name (optional)
- Surname (optional)
- Address line 1 (optional)
- Postcode (optional)

Note: At least one of the optional parameters must be present to execute a search.

### Response
A successful response should return a list all matching citizens.

```json
{
  "results": [
  {
    "id": null,
    "hackneyhomesId": null,
    "title": "MR",
    "surname": "Test",
    "firstName": "Testing",
    "dateOfBirth": "1962-01-1",
    "address": null,
    "addressLine1": "1 THE HACKNEY SERVICE CENTRE HILLMAN STREET HACKNEY LONDON HACKNEY E8 1DY",
    "addressLine2": "HILLMAN STREET",
    "addressLine3": "HACKNEY",
    "addressCity": "LONDON",
    "addressCountry": "HACKNEY",
    "postCode": "E8 1DY",
    "systemName": "CitizenIndex",
    "larn": "LARN2TEST260",
    "uprn": "10012345678",
    "usn": "101010",
    "fullAddressSearch": "1THEHACKNEYSERVICECENTREHILLMANSTREETHACKNEYLONDONHACKNEYE81DY",
    "fullAddressDisplay": "1 THE HACKNEY SERVICE CENTRE HILLMAN STREET HACKNEY LONDON HACKNEY E8 1DY",
    "crMcontactId": "00000000-0000-0000-0000-000000000000",
    "fullName": "TESTING TEST"
  },
  {
   ...etc...
  }
 ]
}
   
```

## Add a new citizen

Creates a record of a new citizen.

```
POST /v1/contacts
```
### Request
```json
{
  "crMcontactId": null,
  "title": "MR",
  "dateOfBirth": "2018-01-01",
  "lastName": "Testing",
  "firstName": "Test",
  "email": "test@test.com",
  "address1": "MAURICE BISHOP HOUSE E8 1HH",
  "address2": "17 READING LANE",
  "address3": "HACKNEY",
  "city": "LONDON",
  "postCode": "E8 1HH",
  "telephone1": "0912345678",
  "telephone2": null,
  "telephone3": null,
  "larn": "LARN2TEST129",
  "housingId": "1234567",
  "usn": "121212",
  "createdByOfficer": "a1c1c1c1c-b4f7-e711-1234-12345faa6a31",
  "uprn": null,
  "fullAddressSearch": "MAURICEBISHOPHOUSEE81HH17READINGLANEHACKNEYLONDONE81HH",
  "fullAddressDisplay": "MAURICE BISHOP HOUSE E8 1HH 17 READING LANE HACKNEY LONDON E8 1HH",
  "fullName": "Test Testing"
}
   
```

- crMcontactId - used if the citizen already exists in CRM2011 so the new citizen can be created with the same contact ID in Dynamics 365 CRM
- title - citizen's title
- dateOfBirth - citizen's date of birth
- lastName - citizen's last name (mandatory)
- firstName - citizen's first name (mandatory)
- email - citizen's email
- address1 - citizen's address line 1
- address2 - citizen's address line 2
- address3 - citizen's address line 3
- city - citizen's city
- postCode - citizen's postcode
- telehpone1 - citizen's primary telephone
- telehpone2 - citizen's telephone 2
- telehpone3 - citizen's telephone 3
- larn - citizen's LARN (as returned from Citizen Index)
- housingId - citizen's housingId (as returned from Citizen Index)
- usn - citizen's usn
- createdByOfficer - the ID of the officer who is currently logged into the system and is creating the new citizen contact (mandatory)
- uprn - citizen's urpn
- fullAddressSearch - a concatinated string with no space used with Citizen Index search
- fullAddressDisplay - the full address of the citizen as to be displayed
- fullName - citizen's full name

Note: At least one of the optional parameters must be present to execute a search.

### Response
A successful POST request should have the following response:

```json
{
    "contactid": "8c12345a-fd0b-e811-811d-123456a6a11",
    "firstName": "Test",
    "lastName": "Testing",
    "fullName": "Test Testing",
    "dateOfBirth": "2018-01-01",
    "email": "test@test.com",
    "address1": "MAURICE BISHOP HOUSE E8 1HH",
    "address2": "17 READING LANE",
    "address3": "HACKNEY",
    "city": "LONDON",
    "postCode": "E8 1HH",
    "telephone1": "0912345678",
    "larn": "LARN2TEST129",
    "housingId": "1234567",
    "usn": "121212"
}
   
```

## Get notifications

Returns notifications information for one account based on the tag reference provided. 

```
Get /v1/notifications?tagreference=123456%2F78
```

### Parameters
- Tag reference (mandatory)

### Response
A successful response should return the notifications record corresponding to the provided tag reference.

```json
{
  "result": {
    "directDebit": null,
    "saffRentAccount": "28812345",
    "accountId": "75e56bod-07fa-e123-8110-12345faaf8c1",
    "rentAccountNotificationId": "34c456a7-be01-e678-811b-905416faa6a11",
    "isRentAccountNotification": true,
    "phone": "",
    "email": ""
  }
}
   
```

## Create a notification record

Creates a notification record for an account. 

```
POST /v1/notifications
```
### Request
```json
{
  "tagReference": "123456/78",
  "email": "test@test.com",
  "mobilePhone": "",
  "areNotificationsOn": false,
  "accountReference": "75e56bod-07fa-e123-8110-12345faaf8c1",
  "accountType": "1",
}
   
```
- tagReference - customer's tag reference
- email - customer's email
- mobilePhone - customer's mobile phone
- areNotificationsOn - a boolean representing whether the customer's notifications are on or off
- accountReference - the account ID of the customer
- accountType - "1" for Rent account and "2" for Lease account

### Response
A successful POST request should have the following response:

```json
{
    "email": "test@test.com",
    "telephone": "",
    "areNotificationsOn": false,
    "tagReference": "123456/78",
    "accountType": "1"
}
   
```

## Update a notification record

Updates a notification record for an account. 

```
PUT /v1/notifications?notificationId=1c1c2c3cd-07fa-e123-8110-12345faaf8c1
```
### Request
```json
{
  "email": "test@testing.com",
  "mobilePhone": "12345678900",
  "areNotificationsOn": true
}
   
```
- email - customer's email
- mobilePhone - customer's mobile phone
- areNotificationsOn - a boolean representing whether the customer's notifications are on or off
- notificationId - the ID of the notification record to be updated

### Response
A successful PUT request should have the following response:

```json
{
    "email": "test@testing.com",
    "telephone": "12345678900",
    "areNotificationsOn": true,
    "notificationId": "1c1c2c3cd-07fa-e123-8110-12345faaf8c1"
}
   
```

## Get tenancy management interactions

Returns a list of tenancy management interactions. 

```
Get /v1/tenancymanagementinteractions?contactid=12345b678-d901-e511-b5a2-12121298417b&personType=contact
```

### Parameters
- Contact ID (required)
- Person Type (required)
  - contact
  - officer
  - manager

### Response
A successful response should return a list of all tenancy management interactions that are assigned to the provided contact id.

```json
{
 "results": [
    {
      "incidentId": "1c2c3b4d-ef0f-e811-8114-1c2bfaaf8c1",
      "ticketNumber": "CAS-31234-W1F7S2",
      "stateCode": 1,
      "nccOfficersId": "9912345-da01-e234-8112-71236faaf8c1",
      "nccEstateOfficer": "Bhavesh Test",
      "createdon": "2018-02-12T12:22:26Z",
      "nccOfficerUpdatedById": "12345567-da01-e811-1234-34567faaf8c1",
      "nccOfficerUpdatedByName": "Bhavesh Test",
      "natureOfEnquiryId": 3,
      "natureOfEnquiry": "Estate Managment",
      "enquirySubjectId": 100000005,
      "enquirysubject": "Joint tenancy application",
      "interactionId": "11c2b3d-ef0f-e811-123d-70106faa6a11",
      "areaManagerId": "1b2b3bc4d-b005-e811-811c-71236faa6a11",
      "areaManagerName": "Mirela Estate Manager Test",
      "officerPatchId": "1b2b3b4bcb005-e811-811c-70106faa6a11",
      "officerPatchName": "Bhavesh Patch",
      "areaName": "Central Panel",
      "handledBy": "Estate Officer",
      "requestCallBack": true,
      "contactId": "1b2bcb3d4f-ed0f-e123-811d-70106faa6a11",
      "contactName": "Will Test",
      "contactPostcode": "E8 2LN",
      "contactAddressLine1": "47, ABERSHAM ROAD, HACKNEY, LONDON, E8 2LN",
      "contactAddressLine2": null,
      "contactAddressLine3": null,
      "contactAddressCity": null,
      "contactBirthDate": null,
      "contactTelephone": "1114567890",
      "contactEmailAddress": null,
      "contactLarn": null,
      "AnnotationList": [
        {
          "noteText": "Test logged on  12/02/2018 12:38:21 by Bhavesh Test",
          "annotationId": "ee1bb2vv-f10f-123vc-8111-e0071b7fe041",
          "noteCreatedOn": "12/02/2018T12:38Z"
        },
	{
         ...etc...
        }
      ]
    },
    {
     ...etc...
    }
  ]
}
   
```

## Get tenancy management interactions by area

Returns a list of tenancy management interactions based on a selected area. 

```
Get /v1/tenancymanagementinteractions/getareatrayinteractions?officeid=1
```

### Parameters
- Office ID (required)

### Response
A successful response should return a list of all tenancy management interactions that are corresponding to the selected area.

```json
{
 "results": [
    {
      "incidentId": "1c2c3b4d-ef0f-e811-8114-1c2bfaaf8c1",
      "ticketNumber": "CAS-31234-W1F7S2",
      "stateCode": 1,
      "nccOfficersId": "9912345-da01-e234-8112-71236faaf8c1",
      "nccEstateOfficer": "Bhavesh Test",
      "createdon": "2018-02-12T12:22:26Z",
      "nccOfficerUpdatedById": "12345567-da01-e811-1234-34567faaf8c1",
      "nccOfficerUpdatedByName": "Bhavesh Test",
      "natureOfEnquiryId": 3,
      "natureOfEnquiry": "Estate Managment",
      "enquirySubjectId": 100000005,
      "enquirysubject": "Joint tenancy application",
      "interactionId": "11c2b3d-ef0f-e811-123d-70106faa6a11",
      "areaManagerId": "1b2b3bc4d-b005-e811-811c-71236faa6a11",
      "areaManagerName": "Mirela Estate Manager Test",
      "officerPatchId": "1b2b3b4bcb005-e811-811c-70106faa6a11",
      "officerPatchName": "Bhavesh Patch",
      "areaName": "Central Panel",
      "handledBy": "Estate Officer",
      "requestCallBack": true,
      "contactId": "1b2bcb3d4f-ed0f-e123-811d-70106faa6a11",
      "contactName": "Will Test",
      "contactPostcode": "E8 2LN",
      "contactAddressLine1": "47, ABERSHAM ROAD, HACKNEY, LONDON, E8 2LN",
      "contactAddressLine2": null,
      "contactAddressLine3": null,
      "contactAddressCity": null,
      "contactBirthDate": null,
      "contactTelephone": "1114567890",
      "contactEmailAddress": null,
      "contactLarn": null,
      "AnnotationList": [
        {
          "noteText": "Test logged on  12/02/2018 12:38:21 by Bhavesh Test",
          "annotationId": "ee1bb2vv-f10f-123vc-8111-e0071b7fe041",
          "noteCreatedOn": "12/02/2018T12:38Z"
        },
	{
         ...etc...
        }
      ]
    },
    {
     ...etc...
    }
  ]
}
   
```
