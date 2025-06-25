# Manual app.planning.nl API

The planning tool app.planning.nl provides an API for data exchange. With this API, it is possible to read and modify nearly all data. The API is based on the OData protocol for data exchange.

This document describes how to get started with this API and how to use it.

## Installation

To connect to our API, an API token is required.
1. Log in to app.planning.nl (with user management rights)
2. Go to the form of the desired user
3. Go to the *Authentication* tab
4. Check the box for *API access* (or *Webservice*, depending on the version)
5. Go to the "Tokens" tab and click "New" at the top right
6. Give the token a description, check "Can be used as API token" and optionally set an expiration date
7. Click save. A token will appear
8. Copy and paste this token after the URL: `https://app.planning.nl/OData/V1/info?token=...`
9. Open this URL in the browser. A table with all entity types in the system will appear

The rights set for this user (possibly via role) also apply to all read and write actions of the API.

## Data Model

The URL above returns a table with all types in the system. There are quite a few because the types correspond to the complete data model. Which types/fields are available depends on the configuration of your environment.

You may only need a few types for your purposes. The examples later will clarify which types are important for which use case.

### Fields

By clicking on a type, you can see which fields and relations are available:  
![image](https://github.com/Planning-nl/app-api-examples/assets/120531/81e933d9-43bf-4a0e-9218-46fc0dcf5583)

The available types and fields depend on your environment configuration.

Each field has a **datatype** (for example `Edm.String`). This is an OData type. Most of these are self-explanatory, but an `Edm.DateTimeOffset` is formatted as ISO UTC datetime for the Europe/Amsterdam timezone. For example, `2024-01-01T10:00:00Z` indicates a local time of 11:00.

### Navigations

Some fields have **navigations**, meaning they have a relationship with another entity. For example, personnel have a relation to a *personnel_resourcetype*, called `ResourceTypeEntity`. There are also relations to collections that reference other types referring to a staff member. An example is `Resourcedepartments_Resource`.

## Reading Data

Data can be read with an OData GET request, for example:  
`https://app.planning.nl/OData/V1/personnelcollection`

> The token parameter is omitted in examples below. Note that it can also be passed as an HTTP header via `X-API-KEY`.

*personnelcollection* is the entity set name as shown in the info table.  
This returns a JSON document containing all personnel in the system (where the user has read rights).

You can also request a single document by id: `/personnelcollection(4)`

The OData protocol contains extra parameters for filtering, sorting, limiting, and more. Some examples follow.

### Filters

`/personnelcollection?$filter=ResourceType eq 2`  
Select all personnel with resource type id *2*

Note the query parameter must be URL-encoded, but it is not done here for readability.

If you use a tool (like Azure Logic Apps) that encodes spaces as `+` instead of `%20`, our API usually does not accept this. However, by adding the query parameter `&odata-accept-forms-encoding=true` to URLs, the `+` characters *are* interpreted as spaces.

`/personnelcollection?$filter=ResourceTypeEntity/Number eq 'abc'`  
Select personnel with resource type having Number field 'abc'

`/personnelcollection?$filter=year(Birthdate) ge 2000`  
All born from year 2000 onward

`/personnelcollection?$filter=Resourcedepartments_Resource/any(d:d/DepartmentEntity/Description eq 'abc')`  
Personnel for department with description 'abc'

See available functions list:  
https://docs.oasis-open.org/odata/odata/v4.0/errata03/os/complete/part2-url-conventions/odata-v4.0-errata03-os-part2-url-conventions-complete.html#_Filter_System_Query

### Selecting

`/personnelcollection?$select=Firstname,Lastname`  
Select only the *Firstname* and *Lastname* fields

`/personnelcollection?$expand=ResourceTypeEntity($select=Description)`  
Add the *Description* field of the resource type

`/personnelcollection?$expand=Personnel_absenceassignments_Resource($select=Start,End;$expand=AbsenceTypeEntity($select=Description))`  
Add all absence assignments with their absence types

Note you can also filter, sort, and limit inside an expand.

### Sorting

`/personnelcollection?$orderby=Firstname`  
Sort by first name

`/personnelcollection?$orderby=ResourceTypeEntity/Description`  
Sort by resource type description

### Limiting

`/personnelcollection?$top=10&$orderby=Firstname`  
Select first 10

`/personnelcollection?$top=10&$skip=10&$orderby=Firstname`  
Select next 10

`/personnelcollection?$top=0&$count=true`  
Request only the count of personnel

### Request Limit

There is a limit of 10,000 items in the response to protect the server from requests without filters. If your request exceeds this, a `@odata.nextLink` field with a URL will be included in the response. This URL can be requested again to fetch the next 10,000 results. This can be done recursively until there is no nextLink, meaning the entire set has been retrieved.

### Generating OData URL

In app.planning.nl you can generate an OData URL by opening a table. Filters and displayed fields (gear icon) are exported via *export* > *OData* to a URL. Example:  
![image](https://github.com/Planning-nl/app-api-examples/assets/120531/fed13fbe-90f6-4325-b77b-bbaef6ff3061)

Result:  
`https://app.planning.nl/odata/departments?$filter=contains(Description, 'afd')&$select=Description,Id,SortIndex,Number,Color,Comments,CreatedAt,LastModifiedAt&$expand=CreatedByEntity($select=Description),LastModifiedByEntity($select=Description),DeletedByEntity($select=Description)`

> Note! The URL works from the application and differs slightly from the API. Replace `/odata/` with `/OData/V1/` to convert it to an API URL.

### Aggregations with $apply

The OData query option `$apply` allows server-side operations like aggregations, grouping, and calculations. This is useful to get totals, averages, or grouped results without computing them yourself.

#### What can you do with `$apply`?

- **Aggregations**: Sum, average, min, max, count, etc.  
- **Grouping**: Group data by one or more fields  
- **Calculations**: Compute new fields based on existing data  
- **Combinations**: Combine these operations in one query

#### Examples

- **Sum of hours**:  
```

absenceassignments?\$apply=aggregate(Hours with sum as TotalHours)

```
- **Group and aggregate**:  
```

absenceassignments?\$apply=groupby((AbsenceType))/aggregate(Hours with sum as TotalHours)

```
- **Combine operations**:  
```

absenceassignments?\$apply=compute(year(Start) as Year)/groupby((Year),aggregate(Hours with sum as TotalHours))

````

## Writing Data

The OData protocol also supports creating, updating, and deleting entities via POST, PATCH, and DELETE HTTP requests.

### Creating

```bash
curl -X POST -H "Content-Type: application/json; charset=utf-8" -d @body.json "https://app.planning.nl/OData/V1/personnelcollection?token=..."
````

With body.json:

```json
{"Firstname": "John", "Lastname": "Doe"}
```

A new entity will be created.

### Upserting

You can update an existing entity via a `POST` request. You can refer to an existing entity in different ways.

#### Id

If you provide an `Id`, that specific entity is updated. The `Id` must exist; otherwise, an error occurs.

#### ExternalId

If you provide an `ExternalId`, the entity is searched for that. If not found, a new entity is created.

#### Unique keys

For various entities, unique keys are checked automatically.

| Type                           | Column names                               |
| ------------------------------ | ------------------------------------------ |
| resourceteam                   | Resource, Team                             |
| resourcecompetence             | Resource, Competence                       |
| resourcedepartment             | Resource, Department                       |
| activityassignmentcontact      | ActivityAssignment, RelationContact        |
| activitycontact                | Activity, RelationContact                  |
| projectdepartment              | Project, Department                        |
| projectcontact                 | Project, RelationContact                   |
| projectrequestplacementcontact | ProjectRequestPlacement, RelationContact   |
| projectphasecontact            | ProjectPhase, RelationContact              |
| projectcompetencevalue         | ProjectCompetence, Competence              |
| projectcompetenceplacement     | ProjectRequestPlacement, ProjectCompetence |

#### Matcher

You can also provide a `Matcher`. You specify multiple column names to match on. If an entity exists with exactly the same values for those columns, it is updated. If not found, a new entity is created.
For example, matching on the `Number` field (`{"Matcher": "Number"}`). You can also match on a relation via the column for that relation:

```json
{
  "ResourceTypeEntity": {"Number": "a", "Matcher": "Number"},
  "Custom_naam": "test",
  "Matcher": "ResourceType,Custom_naam"
}
```

Note the matcher works exactly like unique keys.

### Updating

```bash
curl -X PATCH -H "Content-Type: application/json; charset=utf-8" -d @body.json "https://app.planning.nl/OData/V1/personnelcollection(123)"
```

With body.json:

```json
{"Firstname": "Richard"}
```

This updates the item with id 123. In practice, POST with ExternalId is often used for linked entities.

### Deleting

```bash
curl -X DELETE "https://app.planning.nl/OData/V1/personnelcollection(123)"
```

The personnel with Id 123 is deleted. Note this only works by id, so usually the Id must be found first for an ExternalId. This can be error-prone because it is not transactional. Therefore, our own tool usually does not use these endpoints but uses the **Batch API**.

## Batch API

What we found lacking in standard OData was writing multiple operations at once within one transaction. Therefore, we added our own **Batch API** on top of standard OData actions, which allows multiple OData actions to be executed atomically. We recommend all parties coding integrations to use this Batch API instead of standard OData requests. Note that our own planning tool also uses this Batch API, which is well-tested and flexible.

> Tip: In Chrome Developer Network tab you can see how the OData API is used internally.

The batch API is available at `https://app.planning.nl/OData/V1/batch`. You must add a token or `X-API-KEY` header for authentication.

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

This batch request first adds a new *personnel type* with `ExternalId` "inleen" (if it did not exist) and sets its `Description` to "Inleen". Then it upserts a personnel record referencing the newly created "inleen" type. Note that this is also a **deep upsert**, e.g., with `ResourceSortEntity`. The `Resourcedepartments_Resource` collection overwrites all linked departments of this personnel at once, marking "Amsterdam" as main department.

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

* `items`: array of `BatchRequestItem` objects (see below)
* `rollbackOnError`: default `true`, set to `false` to ignore failing items and apply the rest
* `dryRun`: default `false`, set to `true` to rollback transaction (no DB effect, useful for testing)
* `fixOptions`: if validation problems occur, API sometimes suggests fix options, which can be specified here
* `skipItemsAfterError`: default `true`, set to `false` to continue after a failing item

### BatchRequestItem

All items execute serially within the same transaction. It is similar to sending separate OData requests but in one go.

A `BatchRequestItem` can contain:

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

Some options can also be global; in that case, they override the global setting for this item only.

`entitySetName` specifies the set the operation applies to (e.g. *personnelcollection*).

`BatchMethod` indicates which operation to perform (explained below).

### Batch Methods

Supported methods:

#### UPSERT

Insert a new item or update an existing one. `BatchRequestItem.entity` is required and contains the updates. Equivalent to OData POST. `entityId` is optional; if specified, it updates an existing entity. Otherwise, `Id` or `ExternalId` from the entity is used to identify.

> Note that some types use other fields for identification. For example, in `resourcedepartments`, if both `Resource` and `Department` are specified, it tries to match and update an existing record.

Example:

```json
{
  "items" : [
    {
      "entity" : {
        "ExternalId" : "ab1234",
        "Firstname" : "Hello"
      },
      "entitySetName" : "personnelcollection",
      "method" : "UPSERT"
    }
  ]
}
```

#### UPDATE

Same as `UPSERT`, but returns an error if the entity does not exist.

#### DELETE

Equivalent to OData `DELETE`. `entityId` or `entity` can be used for identification. Example:

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

#### UPSERT\_MULTIPLE

Update a set of entities based on `entityFilter` (filter syntax like `$filter` in GET). `entity` contains properties/navigations to update.

Example:

```json
{
  "items" : [
    {
      "entity" : {
        "Description" : "NewDescription"
      },
      "entityFilter" : "startswith(ExternalId, 'ab')",
      "entitySetName" : "personnel_resourcetypes",
      "method" : "UPSERT_MULTIPLE"
    }
  ]
}
```

#### DELETE\_MULTIPLE

Delete a set of entities based on `entityFilter`.

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

If you try to place a `projectphase` (request block) outside the project, a validation error usually occurs. The error message shows possible options:

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
      "validationInfo" : {
        "errors" : [ {
          "property" : "End",
          "failureCode" : "PROJECTREQUESTPLACEMENT_OUTSIDE_PROJECT_RANGE",
          "possibleSolutions" : [ {
            "solutionCode" : "EXTEND_PARENT_RANGE",
            "description" : "Extend the range of the parent objects"
          } ],
          "message" : "Request lies outside the Project period"
        } ]
      }
    }
  } ]
}
```

Adding the following fix option to the request (or request item) automatically extends the project:

```json
"fixOptions": { "PROJECTREQUESTPLACEMENT_OUTSIDE_PROJECT_RANGE": "EXTEND_PARENT_RANGE" }
```

### Response

A batch request returns a response with the result per item:

```typescript
export interface BatchResponse {
    rollback: boolean;
    noErrors: boolean;
    items: BatchResponseItem[];
}

export interface BatchResponseItem {
    method: BatchMethod;
    entitySet
```


Name: string;
entityId?: number;
entity?: any;
exception?: ODataExceptionInfo;
entities?: any\[];
}

````

If `rollback` is `true`, nothing was changed in the database, due to errors or a dry run.

If `noErrors` is `true`, the request was successful without errors.

`entity` or `entities` fields contain the resulting entities after the operation.

Errors are shown in the respective response item as `exception`, which could be authorization, validation, or internal errors.

Sometimes, e.g. invalid JSON input, no `BatchResponse` is returned but a more primitive error.

## Sessions

Authentication is done via `token` query parameter or `X-API-KEY` header.

The server is stateless. Sessions are removed immediately after a request. Sending cookies is no longer necessary.

Note our servers support HTTP2.

## Use Cases

### Reading Data

Suppose you want to periodically read all absences from the last 2 weeks from app.planning.nl and import them into another system.

As described above, you can do this with a standard OData URL, for example:  
`https://app.planning.nl/OData/V1/absenceassignments?$filter=End gt (now() sub duration'P14D')`

If the set can be larger than 10,000 items, recursively call the *nextLink* to fetch the entire set.

You will likely need to convert data (e.g., `Start` and `End` to the correct format for the target system) before importing.

### Synchronizing Personnel

Suppose you have a set of objects from an external system to synchronize into app.planning.nl.

It is important that you have a unique field per personnel in the external system, e.g., an ID or personnel number.

The integration periodically reads all data from the external system and converts it into a batch request JSON message. Use the external personnel number as `ExternalId` in the entities. Using the `UPSERT` batch method automatically adds new personnel and updates existing ones.

Note this does not clean up "old" personnel. To do so, add a `DELETE_MULTIPLE` that checks if `ExternalId` is set but excludes the list of existing IDs:

```json
{
  "entityFilter" : "filter=length(ExternalId) gt 0 and not (ExternalId in ('ab123', 'ab234'))",
  "entitySetName" : "personnelcollection",
  "method" : "DELETE_MULTIPLE"
}
````

The list of ExternalIds can be quite large but normally does not cause problems. If issues arise, contact us for advice.

### Creating Project Structure

An external party wanted to automatically create a request block and assignment blocks from their system in the tool. Updates should update the same blocks.

The app.planning.nl project structure:

* From a project, a *request line* can be added
* *Request blocks* with start/end dates can be added to a request line
* Desired *project competences* are specified at the request line
* Competences are attributes that a *resource* (personnel or equipment) can fulfill
* A request block specifies for each project competence the number (or hours) of resources to assign
* *Project assignments* (also blocks with start/end) can be linked to requested competences of a request block
* These are shown on a separate line for the *resource* (personnel or equipment):

![image](https://github.com/Planning-nl/app-api-examples/assets/120531/d12b1125-cc44-4796-ba1d-574e65fbe92f)

All this can be done with the Batch API. Example message:

```js
{
  "items" : [ {
    "entity" : {
      "Description" : "[Project name]",

      // Project start and end dates in Europe/Amsterdam timezone
      "Start" : "2024-02-07T23:00:00Z", 
      "End" : "2024-02-12T23:00:00Z",

      // AllDay indicates the block works with full days, not times
      "AllDay" : true,

      // Identifies the project for upserts
      "ExternalId" : "[Project code]", 
    },
    "entitySetName" : "projects", // Project
    "method" : "UPSERT"
  }, {
    "entity" : {
      "Description" : "",

      // Link to the project created above
      "ProjectEntity" : {
        "ExternalId" : "[Project code]" 
      },

      // Always create/update the same request line
      "ExternalId" : "[Project code]" 
    },
    "entitySetName" : "projectrequests", // Request line
    "method" : "UPSERT"
  }, {
    "entity" : {
      // Per projectcompetence multiple required competences can be specified (*projectcompetencevalues*)
      "Projectcompetencevalues_ProjectCompetence" : [ { 
        "CompetenceEntity" : {
          "ExternalId" : "[equipment]", // For identification
          "Description" : "Equipment" // Competence name

          /*
           * Resource class of competence:
           * 0 = `personnel`
           * 1 = `entity1`
           * 2 = `entity2`
           */
          "ResourceClass" : 2, 
        },
        // Identification
        "ExternalId" : "[Project code]" 
      } ],

      // Always create/update the same competence line
      "ExternalId" : "[Project code]",

      // Link to projectrequest created above
      "ProjectRequestEntity" : {
        "ExternalId" : "[Project code]" 
      },

      /*
       * Used when creating a new block via the tool.
       * (not relevant here, but required field)
       */
      "DefaultAmount" : 1 
    },
    "entitySetName" : "projectcompetences",
    "method" : "UPSERT"
  }, {
    "entity" : {
      "Start" : "2024-02-10T07:00:00Z",
      "End" : "2024-02-10T15:00:00Z",
      "AllDay" : false, // This block works with times, not full days
      "Comments": "[Comments]",

      // Code for the block from the external package
      "ExternalId" : "[Block code]",

      // Link request block to correct request line
      "ProjectRequestEntity" : {
        "ExternalId" : "[Project code]"
      },
    },
    "entitySetName" : "projectrequestplacements",
    "method" : "UPSERT",
    /*
     * If Start/End changes while assignments exist,
     * assignments might fall outside the request block,
     * causing validation errors.
     * One fix option is to automatically shorten project assignments
     * to new Start/End (or remove if completely outside).
     */
    "fixOptions": { "PROJECTASSIGNMENTS_OUTSIDE_RANGE": "CAP_TO_RANGE" },
    /*
     * If Start changes, automatically move all child assignments.
     * This is default behavior and can be omitted.
     * 'extraParameters' usually needed only if indicated by us.
     */  
    "extraParameters": { "moveSubItems": true }
  }, {
    "entity" : {
      /*
       * projectcompetenceplacements can be identified automatically by:
       * - the request block (projectrequestplacement)
       * - and the competence line (projectcompetence)
       */ 
      "ProjectRequestPlacementEntity" : {
        "ExternalId" : "[Block code]"
      },
      "ProjectCompetenceEntity" : {
        "ExternalId" : "[Project code]"
      },

      // Number of required assignments
      "Amount": 1,

      /*
       * Project assignments (blocks) shown.
       * Using deep upsert automatically cleans old assignments for this competence.
       */
      "Projectassignments_ProjectCompetencePlacement": [ {
        "ResourceEntity" : {
          // Optionally auto-create this resource
          "ResourceClass" : 2,
          "ExternalId" : "[Equipment code]",
          "Description" : "[Equipment name]",
          "ResourceTypeEntity": {
            "ResourceClass": 2, // ResourceType also has a ResourceClass
            "ExternalId": "[Equipment type code A]",
            "Description": "[Equipment type A]"
          }
        },

        "Start" : "2024-02-10T07:00:00Z",
        "End" : "2024-02-10T15:00:00Z",
        "AllDay" : false,

        /*
         * HoursPerDay should be 24 if AllDay is false.
         * For full days, HoursPerDay indicates hours per day.
         * HoursTotal can be used to specify fixed hours,
         * regardless of duration (Start, End). Then set HoursPerDay to null.
         */
        "HoursPerDay" : 24,

        /*
         * We must use a composite ExternalId here to be globally unique
         * for projectcompetenceplacements.
         */
        "ExternalId" : "[Block code] [Equipment code]"
      } ]
    },
    "entitySetName" : "projectcompetenceplacements",
    "method" : "UPSERT"
  } ]
}
```

## Questions and Support

For questions about the integration, please contact [support@planning.nl](mailto:support@planning.nl).

