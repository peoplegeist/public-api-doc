# Integration API for Peoplegeist Text AI

This is the documentation for the API:

https://public-api.peoplegeist.com/doc/

## Overview
There are two parts to using this API
* Upload data to Peoplegeist
* Query analysis results

There are no format requirements for the text comments.
* Each event should be an own record
* Machine property is mandatory per record
* You can add meta-field values, that can be later used to filter analysis results.


1. Authentication:
•	Username/password: This will be provided per tenant and should be configurable per tenant.
•	All request should include in the header “Authentication” with basic schema

2. Upload Data
2.1 Ensure Databucket

Events are uploaded in Peoplegeist into “Databuckets”.
My thinking is, that per “work center” we create a “databucket”.

Which “work center” a tenant has activated for Peoplegeist should also be configurable per tenant.

You can create a databucket with this:
Before an upload you can submit all “workcenters” of the tenant. The API will then make sure that the buckets exist.
 
{
  "id_poll": 0,
  "status": "running",
  "name": "Line CAN 1",
  "reference": "OEE-systems ID"
}

On upsert you receive the object with the “id_poll” that’s the databucket id.
You will need that id_poll in the next step.

2.2 Upload Data

Use this endpoint to submit an array of events to the specified databucket (id_poll)
 

The field descriptions are here:



















Example record based on the excel we used during the demo (see attached file)
 

[{
  "lineCount": 0,
  "reference": "eventId", // was not in the excel
  "text": 3 CAN JAMS AT THE RINSER  ",  // NOTE
  "machine": "Infeed Conveyor", // Machine
  "resolution": "", // Resolution + CMMSTechComment
  "date": "2023-02-06T12:00:00Z0", // shiftDate
  "downtime": 549, // OeeLossTimeMin
  "impactQty": 4575, // OeeLossQty
  "meta": {
    “OrderNo”: “51924”,
    “Product”: “C153112/BR”,
    “Shift”: “Green”,
    “ReasonCode”: “EMPTY CAN RINSER”
    “Responsible”: “Operations”,
    “DowntimeType”: “UD”,
    “Tech”: “”
  },
  "metaSerial": {
    “xLink”: 171638”
  },
  "langHint": "en",
  “invalidate”: true
}]



