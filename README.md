# Integration API for Peoplegeist Text AI

## Intro
Peoplegeist TextAI is a text analysis solution for manufacturing companies.
This software understands your shiftlogs, maintenancelogs, equipmentlogs, qa-reports, ... other freetext data and shows you to:
* Reduce downtime
* Increase asset availability

With peoplegeist Text AI you can find in your manufacturing text data:
* Repeating issues, potentially systematic issues
* Biggest loss drivers / failure modes
* Regularly recurring maintenance issues
* Root cause hints
* Tips how to solve the equipment issue

More: https://www.peoplegeist.com

## Overview
There are two parts to using this integration
* Upload text data to Peoplegeist
* Query analysis results

## Authentication:
*	Username/password: This will be provided per tenant and should be configurable per tenant.
*	All request should include in the header “Authentication” with basic schema

## 1. Upload Data

### 1.1 Ensure Databucket

Text data or production events are uploaded in Peoplegeist into “Databuckets”.
A databucket is a database.
You can have as many databuckets as you want.
During analysis you can also select which databucket you want to query.

The suggested pattern is to use:
* Per Line one databucket.
* Per work center one databucket

If you have multiple sites, you can nane your datatucket for example: "Site 1: Line 1"

You can create a databucket with the following endpoint:

```
POST / 
```
It's an UPSERT endpoint. By sending the following example body before every upload, you can ensure that a databucket exists.
The refeference field is used to re-identify the bucket you uploaded.
You can use for example the LineID from the source system as reference.

```
{
  "id_poll": 0,
  "status": "running",
  "name": "Line CAN 1",
  "reference": "your system reference"
}
```
On upsert you receive the object with the “id_poll” that’s the databucket id.
You will need that id_poll in the next step.

### 1.2 Upload text data / production events

There are no format requirements for the text comments.
* Each event should be an own record
* Machine property is mandatory per record
* You can add meta-field values, that can be later used to filter analysis results.

Use this endpoint to submit an array of events to the specified databucket (id_poll)
 
Example record:

```
[{
  "lineCount": 0,
  "reference": "eventId", // event id from your source system
  "text": 3 CAN JAMS AT THE RINSER  ",  // free text
  "machine": "Infeed Conveyor", // machine or asset name
  "date": "2023-02-06T12:00:00Z0", // shiftDate
  "downtime": 549, // duration of event in seconds
  "impactQty": 4575, // any waste or loss quantity
  "meta": { // these are optional meta fields you can add. You can add your ownfield
    “Product”: “C153112/BR”,
    “Shift”: “Green”,
    “ReasonCode”: “EMPTY CAN RINSER”
    “Responsible”: “Operations”,
    “MyOwnFieldName”: “Something”
  },
  "metaSerial": { // non recurring or lookup fields, that are just informational but not useful for grouping
    “OrderNo”: 51924”
  },
  "langHint": "en"
}]
```

### 1.3 Group your uploads: Upload History

Optionally uploaded events can be grouped together that they can be re-identified using the Peoplegeist UI.
The object "Upload History" helps you group events together.
Each databucket can have unlimited "upload histories".
Each event can belong to 1 (one) "upload history".
A "Upload History" can only be used within one databucket.

Before uploading data you can crate an "upload-history".
While uploading the event data, you pass on the "upload-history"'s id.

It is good practice to group your data uploads by upload date.

Example:
- Create uploadHistory: "Plant A - Line 1: 2024-06-10"
- Upload all data of that day assigned to that uploadHistory

## 2. Query analysis results.

Data is anlyzed as upload happen.
Dhe analysis is saved that you can query it instantly.
The depending on the volume of data it can take a few minutes until the analysis is updated to include the latest data.

### 2.1 Lookup problem / solutions

You can lookup solutions to production problems in two steps:
* Get a list of known similar problems
* Get a list of possible solutions for a selected problem

### 2.1.1 Get list of known problems that are similar
```
PATCH get-top-problems
```
Get you a list of problems that are similar to your problem.

Query parameter: 
* topK = (default 20)
Request body:
* the problem as string. Example "The light has gone dark"

You get a list of possible problems back.
Let the user review the list of possible known problems and let the user select the problem that might be a good match the current problem.

Each time you query you get back a "query-id"
This "queryId" will be needed to lookup the solutions.
The "queryId" remains valid for 60min. After that you need to resubmit a new query.

### 2.1.2 Get possible solutions for a problem
```
GET /{query-id}/problem/{problem-id}/solutions
```

Get the known solutions for the problem.
Each solution may have multiple similar texts.
The user can pick the text that works best for him/her.
