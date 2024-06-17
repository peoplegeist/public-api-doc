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
1. Upload text data to Peoplegeist
2. Query analysis results

## Authentication:
*	Username/password: This will be provided per tenant and should be configurable per tenant.
*	All request should include in the header “Authentication” with basic schema

## 1. Upload Data

See Swagger: https://public-api.peoplegeist.com/doc/index.html?url=campaign-public-swagger.json

![Datamodel](./Slide1.PNG?raw=true)


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
PUT /campaign/databucket/
```
It's an UPSERT endpoint. By sending the following example body before every upload, you can ensure that a databucket exists.
The refeference field is used to re-identify the bucket you uploaded.
You can use for example the LineID from the source system as reference.

```typescript
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

Use this endpoint to submit an array of events to the specified databucket (id_poll):
```
POST /campaign/databucket/{id_poll}/logs
```
 
Example record:

```typescript
[{
  "lineCount": 0,
  "reference": "eventId", // event id from your source system
  "text": "CAN JAMS AT THE RINSER  ",  // free text
  "machine": "Filler", // machine or asset name
  "date": "2023-02-06T12:00:00Z0", // shiftDate
  "downtime": 549, // duration of event in seconds
  "impactQty": 4575, // any waste or loss quantity
  "meta": { // these are optional meta fields you can add. You can add your ownfield
    “Product”: “C153112/BR”,
    “Shift”: “Green”,
    “ReasonCode”: “FILLER CAN JAM”
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

See Swagger: https://public-api.peoplegeist.com/doc/index.html?url=feedback-public-swagger.json

Data is anlyzed as they are uploaded.
The analysis is saved that you can query it instantly.
The depending on the volume of data it can take a few minutes until the analysis is updated to include the latest data.

### 2.1 Lookup problem / solutions

You can lookup solutions to known problems in two steps:
* Get a list of known similar problems
* Get a list of known solutions for a selected problem

### 2.1.1 Get list of known problems that are similar
```
PATCH /feedback/trouble-shooter/problems-find
```
Get you a list of problems that are similar to your problem.

Query parameter: 
* topK = how many results you want to get (default 20)

Request body:
```typescript
{
  onlyWithComments: true;
  id_polls: number[];  // list of databucket-ids you want to search in.
  topic: {
    text: 'This is the problem description',
    sentiment: null,
    threshold: 0.55;
  }; // filter by topic

  metaOptions?: {
    machine: 'Machine Name',
    [index: string]: number[] | string[]
  };  // the metafields of the events can be used to filter


  fromDate?: Date; // if you want to limit the date range of the documents to query
  thruDate?: Date; // Depending on usecase it is useful to only get solutions that are somewhat recent. Say 1 year old.
}
```

Returns
```typescript
{
  queryId: 'uuid4';

  queryExpirationDate: 'ISO 8601 Date';

  problems: [
    {
      id: '1';  
      topic: 'heater makes noise';
      details: [
        'heater makes noise',
        'heater is noisy',
        'heater is loud'
      ];
    
      clusterSize: number;
      members: ConversationMetaHeader[];
    
      topicScore?: number;
    },
    {
      id: '2';
      topic: 'boiler valves are hissing';
      details: [
        'boiler is hissing'
      ];
    
      clusterSize: number;
      members: ConversationMetaHeader[];
    
      topicScore?: number;
    }
  ];
}
```

You get a list of possible problems back.
* topic: contains the header of the problem.
* details: a list of variants/similar phrasings that are grouped together into this problem group.
 
Let the user review the list of possible known problems and let the user select the problem that might be a good match the current problem.

Remember the "queryId" and the "problem.id" field. Use these 2 fields to lookup possible solutions.

### 2.1.2 Get possible solutions for a problem
```
GET /feedback/trouble-shooter/problems-find/{query-id}/{problem-id}/solutions
```

Get the known solutions for the problem.
Each solution may have multiple similar texts.
The user can pick the text that works best.

Response:
```typescript
[
  {
    solutionId: '1';
    details: ['tag1', 'tag2', 'tag3', 'tag3'];
    conversations: [
      {
        id_poll_response: number,
        comment: 'this is the solution text',
        machine: 'Machine Name';
        responseDate: Date;
      }
    ];
  },
]
```
