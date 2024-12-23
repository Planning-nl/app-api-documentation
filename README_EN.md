# User Guide for app.planning.nl API

The planning tool app.planning.nl offers an API for data exchange. This API allows you to read and modify nearly all data. The API is based on the OData protocol for data exchange.

This document describes how to get started with the API and how to use it.

## Installation

To connect to our API, you need an API token.

1. Log in to app.planning.nl (with user management permissions).
2. Go to the form of the desired user.
3. Navigate to the *Authentication* tab.
4. Check the box for *API Access* (or *Webservice*, depending on the version).
5. Click the button named *Generate API Token* (or *Generate Webservice Token*). A token will now appear.
6. Copy and paste this token into the URL: `https://app.planning.nl/OData/V1/info?token=...`
7. Open this URL in a browser. A table with all entity types in the system will appear.

The configured rights for this user (possibly via roles) also apply to all read and write actions of the API.

## Data Model

The URL above provides a table of all types in the system. These represent the complete data model. The available types/fields depend on your environment configuration.

For your purposes, you may only need a few types. The examples later on will clarify which types are relevant for specific use cases.

### Fields

Clicking on a type shows the available fields and relationships:
![image](https://github.com/Planning-nl/app-api-examples/assets/120531/81e933d9-43bf-4a0e-9218-46fc0dcf5583)

The available types and fields depend on your environment configuration.

Each field also specifies a **data type** (e.g., `Edm.String`), which is an OData type. For the most part, these are straightforward, but an `Edm.DateTimeOffset` is formatted as an ISO UTC datetime for the Europe/Amsterdam timezone. For example, `2024-01-01T10:00:00Z` represents a local time of 11:00.

### Navigations

Some fields also have a **navigation**, meaning there is a relationship with another entity. For instance, personnel has a relationship to a *personnel_resourcetype*, called `ResourceTypeEntity`. There are also relationships to collections, which reference a personnel member. An example is `Resourcedepartments_Resource`.

## Reading Data

Data can be read using an OData GET request, e.g.,:
`https://app.planning.nl/OData/V1/personnelcollection`

> In the examples, the token parameter is omitted for simplicity. Note that it can also be provided as an HTTP header via `X-API-KEY`.

*personnelcollection* is the entity set name shown in the info table. This returns a JSON document containing all personnel in the system (for which the user has read permissions).

A single document can also be requested by its ID: `/personnelcollection(4)`.

The OData protocol includes extra parameters for filtering, sorting, limiting, etc. Below are some examples.

### Filters

`/personnelcollection?$filter=ResourceType eq 2`  
Select all personnel with resource type ID *2*.

Note that the query parameter must be URL-encoded. For readability, this has not been done here.

If you are using a tool (like Azure Logic Apps) that encodes spaces as `+` instead of `%20`, our API generally does not accept this. However, by adding the query parameter `&odata-accept-forms-encoding=true` to the URLs, `+` characters are interpreted as spaces.

`/personnelcollection?$filter=ResourceTypeEntity/Number eq 'abc'`  
Select all personnel with a resource type whose Number field equals 'abc'.

`/personnelcollection?$filter=year(Birthdate) ge 2000`  
All personnel born from the year 2000 onwards.

`/personnelcollection?$filter=Resourcedepartments_Resource/any(d:d/DepartmentEntity/Description eq 'abc')`  
Personnel in a department with the description 'abc'.

For a list of available functions, see: https://docs.oasis-open.org/odata/odata/v4.0/errata03/os/complete/part2-url-conventions/odata-v4.0-errata03-os-part2-url-conventions-complete.html#_Filter_System_Query 

### Selecting

`/personnelcollection?$select=Firstname,Lastname`  
Select only the fields *Firstname* and *Lastname*.

`/personnelcollection?$expand=ResourceTypeEntity($select=Description)`  
Include the *Description* field of the resource type.

`/personnelcollection?$expand=Personnel_absenceassignments_Resource($select=Start,End;$expand=AbsenceTypeEntity($select=Description))`  
Include all absence assignments along with the absence type.

Note that within an expand, you can also filter, sort, and limit.

### Sorting

`/personnelcollection?$orderby=Firstname`  
Sort by first name.

`/personnelcollection?$orderby=ResourceTypeEntity/Description`  
Sort by resource type description.

### Limiting

`/personnelcollection?$top=10&$orderby=Firstname`  
Select the first 10 items.

`/personnelcollection?$top=10&$skip=10$orderby=Firstname`  
Select the next 10 items.

`/personnelcollection?$top=0&$count=true`  
Retrieve only the count of personnel.

### Query Limit

There is a limit of 10,000 items in the response. This is to protect our server against unfiltered requests. If your requested set exceeds this limit, a `@odata.nextLink` field with a URL will be included in the response. This URL can be used as an OData request to retrieve the next 10,000 results. This can be done recursively until no next page link is included in the response, indicating that the entire set has been retrieved.

### Generating OData URLs

In app.planning.nl, you can also generate an OData URL by opening a table. The filters and displayed fields (gear icon) are exported to a URL via the *export* > *OData* button. Example:

![image](https://github.com/Planning-nl/app-api-examples/assets/120531/fed13fbe-90f6-4325-b77b-bbaef6ff3061)

Resulting in: `https://app.planning.nl/odata/departments?$filter=contains(Description, 'afd')&$select=Description,Id,SortIndex,Number,Color,Comments,CreatedAt,LastModifiedAt&$expand=CreatedByEntity($select=Description),LastModifiedByEntity($select=Description),DeletedByEntity($select=Description)`

> Note: The URL works from the application and therefore slightly differs from the API URL. Replace `/odata/` with `/OData/V1/` to convert it to an API URL.

## Writing Data

The OData protocol also supports creating, updating, and deleting entities via POST, PATCH, and DELETE HTTP requests.

### Creating

`curl -X POST -H "Content-Type: application/json; charset=utf-8" -d @body.json "https://app.planning.nl/OData/V1/personnelcollection?token=..."`

With body.json:

```json
{"Firstname": "John", "Lastname": "Doe"}
```

A new entity is now created.

### Upserting

Existing entities can be updated via a `POST` request. There are several ways to reference an existing entity:

#### ID
If an `Id` is provided, the specific entity is updated. If the `Id` is specified but does not exist, an error is returned.

#### ExternalId
If an `ExternalId` is provided, the entity is searched for. If not found, a new entity is created.

#### Unique Keys
For certain entities, specific unique keys are automatically checked.

| Type | Column Names |
|---|---|
| resourceteam | Resource, Team |
| resourcecompetence | Resource, Competence  |
| resourcedepartment | Resource, Department |
| activityassignmentcontact | ActivityAssignment, RelationContact |
| activitycontact | Activity, RelationContact |
| projectdepartment | Project, Department |
| projectcontact | Project, RelationContact |
| projectrequestplacementcontact | ProjectRequestPlacement, RelationContact |
| projectphasecontact | ProjectPhase, RelationContact |
| projectcompetencevalue | ProjectCompetence, Competence |
| projectcompetenceplacement | ProjectRequestPlacement, ProjectCompetence |

#### Matcher
A `Matcher` can also be provided. Specify multiple column names to match against. If an entity exists with the exact values for the columns, it is updated; otherwise, a new entity is created. For example:

```json
{
  "ResourceTypeEntity": {"Number": "a", "Matcher": "Number"},
  "Custom_naam": "test",
  "Matcher": "ResourceType,Custom_naam"
}
```

Note that the matcher works exactly like unique keys.

### Updating

`curl -X PATCH -H "Content-Type: application/json; charset=utf-8" -d @body.json "https://app.planning.nl/OData/V1/personnelcollection(123)"`

With body.json:

```json
{"Firstname": "Richard"}
```

The item with ID 123 is updated. Typically, a POST with an ExternalId is used to identify the entity.

### Deleting

`curl -X DELETE "https://app.planning.nl/OData/V1/personnelcollection(123)"`

The personnel member with ID 123 is deleted. Note that this can only be based on the ID, while in practice ExternalId is more often used. For this reason, it is often necessary to first find the ID for an ExternalId. This is error-prone as it does not happen in a transaction. Therefore, we do not use these endpoints for our own tool but rely on the **Batch API**.

## Batch API

The standard OData actions did not meet our needs for writing multiple operations at once within a single transaction. Therefore, we added our own **Batch API**, allowing multiple OData actions to be executed atomically. We recommend all parties coding their own integration to use this Batch API instead of standard OData requests. Note that our planning tool also uses this Batch API, ensuring it is well-tested and flexible enough for many applications.

> Tip: In the Chrome Developer network tab, you can see how the OData API is used internally.

The Batch API is available at the URL `https://app.planning.nl/OData/V1/batch`. A token or `X-API-KEY` header must be added for authentication.

### Batch Request Structure

A batch request consists of multiple items. Example:

```json
{
  "items" : [ 
    {
      "entity" : {
        "Description" : "Inleen",
        "ExternalId" : "inleen"
      },
      "entitySetName" : "personnel_resourcetypes",
      "method" : "UPSERT"
    },
    {
      "entity" : {
        "Firstname" : "John",
        "Lastname" : "Doe",
        "ExternalId" : "ab1234",
        "Resourcedepartments_Resource": [
          {
            "DepartmentEntity": { 
              "ExternalId": "010", 
              "Description": "Rotterdam" 
            }
          },
          {
            "DepartmentEntity": { 
              "ExternalId": "020", 
              "Description": "Amsterdam" 
            }, 
            "MainDepartment": true
          }
        ],
        "ResourceSortEntity" : {
          "ExternalId" : "zzp",
          "Description" : "ZZP"
        },
        "ResourceTypeEntity": { "ExternalId": "inleen" }
      },
      "entitySetName" : "personnelcollection",
      "method" : "UPSERT"
    } 
  ],
  "rollbackOnError": true,
  "skipItemsAfterError": true,
  "dryRun": false,
  "fixOptions": {}
}
```

### Options

```typescript
export interface BatchRequest {
    items: BatchRequestItem[];
    rollbackOnError?: boolean;
    dryRun?: boolean;
    fullValidation?: boolean;
    fixOptions?: { [P in FailureCode]?: SolutionCode };
    skipItemsAfterError?: boolean;
}
```

`items`: An array of `BatchRequestItem` objects (described later).

`rollbackOnError`: Default is `true`. Set to `false` if you want to ignore failing items and apply the rest.

`dryRun`: Default is `false`. Set to `true` to roll back the transaction (no database changes, useful for testing).

`fixOptions`: If validation issues arise, the API may suggest one or more *fix options*, which can be specified here.

`skipItemsAfterError`: Default is `true`. Set to `false` to execute the remaining items after a failing item.

### Batch Request Item

All items are executed serially within the same transaction. This is like performing individual OData requests but in one go.

A `BatchRequestItem` can contain the following:

```typescript
export interface BatchRequestItem {
    method: BatchMethod;
    entitySetName: string;
    entityId?: number;
    entityFilter?: string;
    entity?: any;
    rollbackOnError?: boolean;
    fixOptions?: { [P in FailureCode]?: SolutionCode };
}
```

Some options are also present at the global level. In such cases, they override the global level for only this specific item.

The `entitySetName` indicates the set to which the operation applies (e.g., *personnelcollection*).

The `BatchMethod` specifies *what* operation to perform. These methods are explained below.

### Batch Methods

The following methods are supported:

#### UPSERT

Insert a new item or update an existing one. The `BatchRequestItem.entity` is required and specifies the updates to perform. This operation is equivalent to the OData POST operation. `entityId` is optional and can be used to update an existing entity. Otherwise, the `Id` or `ExternalId` of the entity is used for identification.

> Note: For certain types, other fields can also be used for identification. For example, in `resourcedepartments`, if both a `Resource` and `Department` are specified, an existing record is matched and updated.

Example:

```json
{
  "items" : [
    {
      "entity" : {
        "ExternalId" : "ab1234",
        "Firstname" : "Hello",
      },
      "entitySetName" : "personnelcollection",
      "method" : "UPSERT"
    }
  ]
}
```

#### UPDATE

Same as `UPSERT`, but returns an error if the entity did not exist.

#### DELETE

Equivalent to the OData `DELETE` request method. `entityId` or `entity` can be used for identification (as with `UPSERT`). Example:

```json
{
  "items" : [
    {
      "entity" : {
        "ExternalId" : "ab1234"
      },
      "entitySetName" : "personnelcollection",
      "method" : "DELETE"
    }
  ]
}
```

#### UPSERT_MULTIPLE

This method allows updating a set of entities at once based on the `entityFilter` (a filter with the same syntax as `$filter` in a GET request). The `entity` contains the properties/navigations to update.

Example:

```json
{
  "items" : [
    {
      "entity" : {
        "Description" : "NewDescription",
      },
      "entityFilter" : "startswith(ExternalId, 'ab')",
      "entitySetName" : "personnel_resourcetypes",
      "method" : "UPSERT_MULTIPLE"
    }
  ]
}
```

#### DELETE_MULTIPLE

This method allows deleting a set of entities at once based on the `entityFilter`.

Example:

```json
{
  "items" : [
    {
      "entityFilter" : "startswith(ExternalId, 'ab')",
      "entitySetName" : "personnelcollection",
      "method" : "DELETE_MULTIPLE"
    }
  ]
}
```

### Fix options

Suppose you attempt to place a `projectphase` (query block) outside the project. A validation error is typically returned, showing the available options:

```json
{
  "rollback" : true,
  "noErrors" : false,
  "items" : [ {
    "method" : "UPSERT",
    "entitySetName" : "projectrequestplacements",
    "entityId" : null,
    "entity" : null,
    "exception" : {
      "status" : 400,
      "code" : "invalid",
      "message" : "Validation failed",
      "entityInfo" : {
        "entityTypeName" : "projectrequestplacement",
        "entity" : { },
        "operation" : "UPSERT"
      },
      "toOneInfo" : null,
      "toManyInfo" : null,
      "hookInfo" : null,
      "postponedValidations" : null,
      "validationInfo" : {
        "errors" : [ {
          "property" : "End",
          "failureCode" : "PROJECTREQUESTPLACEMENT_OUTSIDE_PROJECT_RANGE",
          "possibleSolutions" : [ {
            "solutionCode" : "EXTEND_PARENT_RANGE",
            "description" : "Expand the range of the parent objects"
          } ],
          "message" : "Request lies outside the project's period",
          "failureCodeId" : "projectrequestplacementOutsideOfProjectRange",
          "possibleSolutionsIds" : [ "extendParentRange" ]
        } ]
      },
      "authorizationInfo" : null
    },
    "entities" : null
  } ]
}
```

By including the following fix option in the request (or request item), the project is automatically "stretched":

```json
"fixOptions": { "PROJECTREQUESTPLACEMENT_OUTSIDE_PROJECT_RANGE": "EXTEND_PARENT_RANGE" }
```

### Response

A batch request returns a response containing the final result for each item:

```typescript
export interface BatchResponse {
    rollback: boolean;
    noErrors: boolean;
    items: BatchResponseItem[];
}

export interface BatchResponseItem {
    method: BatchMethod;
    entitySetName: string;
    entityId?: number;
    entity?: any;
    exception?: ODataExceptionInfo;
    entities?: any[];
}
```

If the `rollback` field is `true`, no changes were made to the database. This can happen due to errors or if `dryRun` was set to `true`.

If `noErrors` is `true`, the request was processed successfully without errors.

The `entity` or `entities` fields contain the result entities after the operation.

If errors occurred, they are usually included in the respective response item as `exception`. This might indicate an authorization error, validation error, or an internal error.

In some cases, such as invalid JSON, no `BatchResponse` is returned, and a more primitive error message is provided instead.

## Sessions

Authentication is performed via the `token` query parameter or the `X-API-KEY` header.

The server is stateless. Sessions are immediately removed after a request. Cookies no longer need to be sent.

Note that our servers support HTTP2.

## Use cases

### Reading Data

Suppose you want to periodically retrieve all absences from the last two weeks from app.planning.nl and import them into another system.

As described above, this can be done using a standard OData URL. An example might be:

`https://app.planning.nl/OData/V1/absenceassignments?$filter=End gt (now() sub duration'P14D')`

It is essential, if the set can exceed 10,000 items, to recursively call the *nextLink* to retrieve the entire set.

A conversion process will likely need to be performed (e.g., converting `Start` and `End` to the correct format for the target system) before importing the items into the target system.

### Synchronizing Personnel

Suppose you have a set of objects from an external system and want to synchronize them with app.planning.nl.

It is crucial that you have a unique field for each personnel member on the external side. This could be an ID from that system or an employee number.

The integration should periodically read all data from the external system and convert it to a batch request JSON message. The `ExternalId` in the entities should use the external employee number. By using the batch method `UPSERT`, new personnel members are automatically added, and existing ones are updated.

Note that this approach does not clean up "old" personnel. To address this, a `DELETE_MULTIPLE` can be added, which checks if `ExternalId` is populated but excludes the list of existing IDs:

```json
{
  "entityFilter" : "filter=length(ExternalId) gt 0 and not (ExternalId in ('ab123', 'ab234'))",
  "entitySetName" : "personnelcollection",
  "method" : "DELETE_MULTIPLE"
}
```

Note that the list of ExternalIds in the filter can become quite large, but this usually does not cause issues. If problems arise, you can contact us for advice.

### Creating Project Structures

An external party wanted to automatically create a query block and assignment blocks in the tool from their system. Any changes should update the same block.

The project structure of app.planning.nl:

- A *query line* can be added from a project.
- *Query blocks* with start/end dates can be added to a query line.
- Desired *project competencies* can be specified for a query line.
- Competencies are attributes that a *resource* (personnel or equipment) can meet.
- A query block specifies for each project competency a number (or hours) of resources to be allocated.
- Requested competencies of a query block can have *project assignments* (also blocks with start/end) linked.
- These are displayed on a separate line for the *resource* (personnel or equipment):

![image](https://github.com/Planning-nl/app-api-examples/assets/120531/d12b1125-cc44-4796-ba1d-574e65fbe92f)

All of this can be done with the Batch API. Example message:

```js
{
  "items" : [ {
    "entity" : {
      "Description" : "[Project Name]",

      // Start and end date of project in Europe/Amsterdam
      "Start" : "2024-02-07T23:00:00Z", 
      "End" : "2024-02-12T23:00:00Z",

      // AllDay indicates that the block works with whole days and not times
      "AllDay" : true,

      // Identifies the project for upserts
      "ExternalId" : "[Project Code]", 
    },
    "entitySetName" : "projects", // Project
    "method" : "UPSERT"
  }, {
    "entity" : {
      "Description" : "",

      // Link to the created project above
      "ProjectEntity" : {
        "ExternalId" : "[Project Code]" 
      },

      // Always edit the same query line
      "ExternalId" : "[Project Code]" 
    },
    "entitySetName" : "projectrequests", // Query Line
    "method" : "UPSERT"
  }, {
    "entity" : {
      // Multiple required competencies can be specified per project competence (*projectcompetencevalues*).
      "Projectcompetencevalues_ProjectCompetence" : [ { 
        "CompetenceEntity" : {
          "ExternalId" : "[equipment]", // For identification
          "Description" : "Equipment" // Name of the competence line

          /*
           * The *resource class* of the competence:
           * 0 = `personnel`
           * 1 = `entity1`
           * 2 = `entity2`
           */
          "ResourceClass" : 2, 
        },
        // For identification
        "ExternalId" : "[Project Code]" 
      } ],

      // Always edit the same competence line
      "ExternalId" : "[Project Code]",

       // Link to the created project request above 
      "ProjectRequestEntity" : {
        "ExternalId" : "[Project Code]" 
      },

      /*
       * This is used when creating a new block via the tool.
       * (not relevant in this case but still a required field)
       */
      "DefaultAmount" : 1 
    },
    "entitySetName" : "projectcompetences",
    "method" : "UPSERT"
  }, {
    "entity" : {
      "Start" : "2024-02-10T07:00:00Z",
      "End" : "2024-02-10T15:00:00Z",
      "AllDay" : false, // This block works with times, not whole days.
      "Comments": "[Comments]",

      // Code for the block from the external package.
      "ExternalId" : "[Block Code]",

      // Set query block on the correct query line.
      "ProjectRequestEntity" : {
        "ExternalId" : "[Project Code]"
      },
    },
    "entitySetName" : "projectrequestplacements",
    "method" : "UPSERT",
    "fixOptions": { "PROJECTASSIGNMENTS_OUTSIDE_RANGE": "CAP_TO_RANGE" },
    "extraParameters": { "moveSubItems": true }
  } ]
}
```

## Questions and Support

For questions about the integration, contact support@planning.nl.
