# **Manual for app.planning.nl API**

The planning tool **app.planning.nl** provides an API for exchanging data. With this API it is possible to read and modify almost all data. The API is built on the **OData protocol** for data exchange.

This document describes how to get started with this API and how to use it.

## **Installation**

To connect to our API, an **API token** is required.

1. Log in to app.planning.nl (with permissions for user management).
2. Go to the form of the desired user.
3. Go to the tab *Authentication*.
4. Check the box *API access* (or *Webservice*, depending on the version).
5. Go to the tab **Tokens** and click **New** in the top right.
6. Give the token a description, check **Can be used as API token**, and set an expiry date (optional).
7. Click **Save**. A token will now appear.
8. Copy and append this token to the URL:
   `https://app.planning.nl/OData/V1/info?token=...`
9. Open this URL in the browser. A table will appear showing all entity types in the system.

The permissions configured for this user (possibly via roles) also apply to all read and write actions through the API.

## **Data model**

The URL above displays a table with all types in the system. There are quite a few, as the types correspond to the complete data model. Which types/fields are available depends on the configuration of your environment.

You may only need a few types for your purposes. In the examples later on, it will become clearer which types are relevant for which use case.

### **Fields**

Clicking on a type shows which fields and relations are available:
*(image)*

The available types and fields depend on your environment’s configuration.

Each field has a **datatype** (e.g., `Edm.String`). This is an OData type. Most are self-explanatory, but an `Edm.DateTimeOffset` is formatted as ISO UTC datetime for the Europe/Amsterdam timezone. For example:
`2024-01-01T10:00:00Z` corresponds to local time 11:00.

### **Navigations**

Some fields also have a **navigation**. This means a relationship exists with another entity. For example, personnel has a relation to a *personnel_resourcetype*, named `ResourceTypeEntity`.
There are also collection relations, which show types that refer back to a personnel member. An example is `Resourcedepartments_Resource`.

## **Reading data**

Data can be retrieved using an OData **GET** request, for example:
`https://app.planning.nl/OData/V1/personnelcollection`

> The token parameter is omitted in the examples below. Note that this can also be provided as an HTTP header using `X-API-KEY`.

*personnelcollection* is the entity set name as shown in the info table.
This returns a JSON document with all personnel records (that the user has read access to).

A single record can also be fetched by ID:
`/personnelcollection(4)`

The OData protocol supports additional parameters for filtering, sorting, limiting, and so on. Below are some examples.

### **Filtering**

`/personnelcollection?$filter=ResourceType eq 2`
Select all personnel members with resource type ID **2**.

Note: The query parameter must be URL-encoded. This is omitted here for readability.

If you use a tool (such as Azure Logic Apps) that encodes spaces as `+` instead of `%20`, our API normally does **not** accept this. However, if you add the parameter `&odata-accept-forms-encoding=true`, then `+` will *be interpreted as spaces*.

`/personnelcollection?$filter=ResourceTypeEntity/Number eq 'abc'`
Select personnel with resource type number `'abc'`.

`/personnelcollection?$filter=year(Birthdate) ge 2000`
All personnel born from the year 2000 onward.

`/personnelcollection?$filter=Resourcedepartments_Resource/any(d:d/DepartmentEntity/Description eq 'abc')`
Personnel linked to a department with description `'abc'`.

See the list of available functions:
[https://docs.oasis-open.org/odata/odata/v4.0/errata03/os/complete/part2-url-conventions/odata-v4.0-errata03-os-part2-url-conventions-complete.html#_Filter_System_Query](https://docs.oasis-open.org/odata/odata/v4.0/errata03/os/complete/part2-url-conventions/odata-v4.0-errata03-os-part2-url-conventions-complete.html#_Filter_System_Query)

### **Selecting**

`/personnelcollection?$select=Firstname,Lastname`
Select only the *Firstname* and *Lastname* fields.

`/personnelcollection?$expand=ResourceTypeEntity($select=Description)`
Add the *Description* field of the resource type.

`/personnelcollection?$expand=Personnel_absenceassignments_Resource($select=Start,End;$expand=AbsenceTypeEntity($select=Description))`
Add all absence assignments including absence type information.

You can filter, sort, and limit *within* an expand.

### **Sorting**

`/personnelcollection?$orderby=Firstname`
Sort by first name.

`/personnelcollection?$orderby=ResourceTypeEntity/Description`
Sort by resource type description.

### **Limiting**

`/personnelcollection?$top=10&$orderby=Firstname`
Select the first 10 results.

`/personnelcollection?$top=10&$skip=10&$orderby=Firstname`
Select the next 10.

`/personnelcollection?$top=0&$count=true`
Retrieve only the number of personnel items.

### **Request limit**

There is a limit of **10,000 items** per response. This protects the server against unfiltered requests.
If your result set is larger, an `@odata.nextLink` field will be added with a URL to retrieve the next 10,000 items. Continue fetching until no nextLink remains.

> **Note:** Do *not* use large `$skip` values!
> The system must still iterate through all records to check for deleted items, which is slower and heavier than using nextLink.

## **Generating OData URLs**

In app.planning.nl you can generate OData URLs by opening a table. Filters and shown fields (gear icon) are exported via *export* > *OData*.
Example output:

`https://app.planning.nl/odata/departments?$filter=contains(Description, 'afd')&$select=Description,Id,SortIndex,Number,Color,Comments,CreatedAt,LastModifiedAt&$expand=CreatedByEntity($select=Description),LastModifiedByEntity($select=Description),DeletedByEntity($select=Description)`

> Note: This URL works inside the application and differs slightly from the API.
> Replace `/odata/` with `/OData/V1/` to convert it for API usage.

## **Aggregations with $apply**

The OData `$apply` option allows server-side operations such as aggregations, grouping, and calculations.

### What you can do with `$apply`:

* **Aggregations:** sum, average, min, max, count, etc.
* **Grouping:** group by one or more fields
* **Calculations:** compute derived fields
* **Combinations:** mix multiple operations in one query

### Examples

**Sum of hours:**

```
absenceassignments?$apply=aggregate(Hours with sum as TotalHours)
```

**Group and aggregate:**

```
absenceassignments?$apply=groupby((AbsenceType))/aggregate(Hours with sum as TotalHours)
```

**Combined operations:**

```
absenceassignments?$apply=compute(year(Start) as Year)/groupby((Year),aggregate(Hours with sum as TotalHours))
```

## **Writing data**

The OData protocol supports POST, PATCH, and DELETE for creating, updating, and deleting entities.

### **Create**

```
curl -X POST -H "Content-Type: application/json; charset=utf-8" -d @body.json "https://app.planning.nl/OData/V1/personnelcollection?token=..."
```

`body.json`:

```json
{"Firstname": "John", "Lastname": "Doe"}
```

This creates a new entity.

### **Upsert**

You can also update an existing entity via `POST`. Identification can be done by:

#### **Id**

If an `Id` is included, that specific entity is updated.
If the ID does not exist, an error occurs.

#### **ExternalId**

If an `ExternalId` is provided, the system searches for an entity with that ID.
If not found, a new entity is created.

#### **Unique keys**

For some entities, unique keys are enforced:

| Type                           | Columns                                    |
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

#### **Matcher**

You can also include a `Matcher` specifying which columns must match.
If an entity exists with exactly those matching fields, it is updated; otherwise a new one is created.

Example:

```json
{
  "ResourceTypeEntity": {"Number": "a", "Matcher": "Number"},
  "Custom_naam": "test",
  "Matcher": "ResourceType,Custom_naam"
}
```

### **Update**

```
curl -X PATCH -H "Content-Type: application/json; charset=utf-8" -d @body.json "https://app.planning.nl/OData/V1/personnelcollection(123)"
```

`body.json`:

```json
{"Firstname": "Richard"}
```

Updates the item with ID 123.

### **Delete**

```
curl -X DELETE "https://app.planning.nl/OData/V1/personnelcollection(123)"
```

Deletes the personnel member with ID 123.

Note: This can only be done using the ID. As many integrations work with ExternalId, often the ID must first be resolved. This is error-prone, because it is not done within a transaction.

For this reason we recommend using the **Batch API** instead.

## **Batch API**

Because standard OData operations lack transactional grouping, we added our own **Batch API**, allowing multiple OData operations to be executed **atomically**.

We recommend all integrators use the Batch API.
Our own planning tool uses it as well, ensuring it is well-tested and flexible.

Batch API URL:
`https://app.planning.nl/OData/V1/batch`

Authentication via token or `X-API-KEY`.

### **Batch request example**

```json
{
  "items" : [ 
    {
      "entity" : {
        "Description" : "Inloan",
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

## **Options**

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

* **items**: an array of `BatchRequestItem` objects (explained later)
* **rollbackOnError**: default `true`. Set to `false` if you want to ignore failing items while applying the rest.
* **dryRun**: default `false`. Set to `true` to roll back the entire transaction (no database changes—useful for tests).
* **fixOptions**: if validation errors occur, the API may propose one or more *fixOptions*. These can be specified here.
* **skipItemsAfterError**: default `true`. Set to `false` if you want remaining items to execute even after a failure.

## **BatchRequestItem**

All items are executed **sequentially** within the same transaction.
It is comparable to performing multiple OData requests, but bundled into one atomic operation.

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

Some options also exist at the global request level. If they are specified on the item level, they override the global settings for that single item.

* `entitySetName`: The target entity set (e.g., *personnelcollection*).
* `method`: The **type of operation** to perform. Explained below.

## **Batch Methods**

The following methods are supported:

### **UPSERT**

Insert a new item or update an existing one.
`BatchRequestItem.entity` is required and contains the fields to update.

This behaves like the OData **POST** method.

* `entityId` is optional and, if provided, is used to update an existing entity.
* Otherwise the system uses `Id` or `ExternalId` inside the entity.

Some types support additional matching fields. Example:
For `resourcedepartments`, if both `Resource` and `Department` are provided, the system tries to match and update an existing record.

**Example:**

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

### **UPDATE**

Same as UPSERT, but throws an error if the entity does **not** yet exist.

### **DELETE**

Equivalent to OData **DELETE**.

`entityId` or `entity` can be used for identification.

**Example:**

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

### **UPSERT_MULTIPLE**

Updates multiple entities at once using an `entityFilter` (same syntax as `$filter` in GET requests).
The `entity` contains the properties/navigations to update.

**Example:**

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

### **DELETE_MULTIPLE**

Deletes multiple entities at once based on an `entityFilter`.

**Example:**

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

## **Fix options**

Suppose you attempt to create a `projectphase` (question block) outside the project’s date range.
A validation error will be returned. The error response includes possible fix options.

Example error:

```json
{
  "rollback" : true,
  "noErrors" : false,
  "items" : [ {
    "method" : "UPSERT",
    "entitySetName" : "projectrequestplacements",
    "exception" : {
      "status" : 400,
      "code" : "invalid",
      "message" : "Validation failed",
      "validationInfo" : {
        "errors" : [ {
          "property" : "End",
          "failureCode" : "PROJECTREQUESTPLACEMENT_OUTSIDE_PROJECT_RANGE",
          "possibleSolutions" : [ {
            "solutionCode" : "EXTEND_PARENT_RANGE",
            "description" : "Increase the range of the parent objects"
          } ]
        } ]
      }
    }
  } ]
}
```

You can apply the suggested fix automatically by adding:

```json
"fixOptions": { "PROJECTREQUESTPLACEMENT_OUTSIDE_PROJECT_RANGE": "EXTEND_PARENT_RANGE" }
```

## **Response**

A batch request returns a response with results for each item:

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

* If `rollback = true`, no changes were committed (due to errors or because `dryRun` was enabled).
* `noErrors = true` indicates all operations succeeded.
* `entity` or `entities` contain the final resulting entities.
* If any errors occurred, the details appear under `exception`.

In some cases—such as invalid JSON—a more primitive error message may be returned.

## **Sessions**

Authentication is done via the `token` query parameter or the `X-API-KEY` header.

The server is stateless. Sessions are deleted immediately after a request.
Cookies are **no longer needed**.

Our servers support **HTTP/2**.

## **Use Cases**

### **Reading data**

Suppose you want to periodically retrieve all absences from the last 2 weeks from app.planning.nl and import them into another system.

You can use a standard OData URL:
`https://app.planning.nl/OData/V1/absenceassignments?$filter=End gt (now() sub duration'P14D')`

If the result set can exceed 10,000 items, you must follow the `nextLink` recursively until the full dataset has been retrieved.

You may need to convert fields for the target system (e.g., convert `Start` and `End` into the required date/time format).

### **Synchronizing personnel**

Suppose you have a set of objects from an external system and want to synchronize them into app.planning.nl.

It is important that the external system uses a unique value per employee—such as an ID or personnel number.

The integration periodically reads all external data and converts it into a batch request.
The `ExternalId` field is set to the external personnel number.

Using `UPSERT`, new personnel are automatically created and existing ones updated.

However, old personnel that no longer exist externally are **not** automatically removed.
Therefore, a `DELETE_MULTIPLE` operation is added:

```json
{
  "entityFilter" : "filter=length(ExternalId) gt 0 and not (ExternalId in ('ab123', 'ab234'))",
  "entitySetName" : "personnelcollection",
  "method" : "DELETE_MULTIPLE"
}
```

The list can be large, but this is usually not an issue.
If needed, contact us for advice.

### **Creating a project structure**

An external party wanted to automatically create a question block and assignment blocks in the tool based on their system.
When updated, the same block should be modified.

The project structure in app.planning.nl works as follows:

* From a project, you can add a *question line*
* A question line can have *question blocks* (with start/end date)
* The question line specifies required *project competences*
* Competences describe properties that a *resource* (personnel or materials) can fulfill
* A question block specifies the number of resources needed per competence
* Assignments can be linked to these required competences
* Assignments appear on a resource line for the assigned personnel or materials.

![image](https://github.com/Planning-nl/app-api-examples/assets/120531/d12b1125-cc44-4796-ba1d-574e65fbe92f)

This can all be handled with the Batch API.

```js
{
  "items" : [ {
    "entity" : {
      "Description" : "[Projectnaam]",

      // Start- en einddatum project in Europe/Amsterdam
      "Start" : "2024-02-07T23:00:00Z", 
      "End" : "2024-02-12T23:00:00Z",

      // AllDay geeft aan dat het blokje met hele dagen en niet met tijden werkt
      "AllDay" : true,

      // Identificeert het project voor upserts
      "ExternalId" : "[Code project]", 
    },
    "entitySetName" : "projects", // Project
    "method" : "UPSERT"
  }, {
    "entity" : {
      "Description" : "",

      // Koppelen aan het aangemaakte project hierboven
      "ProjectEntity" : {
        "ExternalId" : "[Code project]" 
      },

      // Maak/bewerk steeds dezelfde vraagregel
      "ExternalId" : "[Code project]" 
    },
    "entitySetName" : "projectrequests", // Vraagregel
    "method" : "UPSERT"
  }, {
    "entity" : {
      // Per projectcompetence kunnen meerdere benodigde competenties worden opgegeven (*projectcompetencevalues*).
      "Projectcompetencevalues_ProjectCompetence" : [ { 
        "CompetenceEntity" : {
          "ExternalId" : "[materieel]", // Ter identificatie
          "Description" : "Materieel" // Naam van de competentieregel

          /*
           * De *resource class* van de competentie:
           * 0 = `personeel`
           * 1 = `entity1`
           * 2 = `entity2`
           */
          "ResourceClass" : 2, 
        },
        // Ter identificatie
        "ExternalId" : "[Code project]" 
      } ],

      // Maak/bewerk steeds dezelfde competentieregel
      "ExternalId" : "[Code project]",

       // Koppelen aan de aangemaakte projectrequest hierboven 
      "ProjectRequestEntity" : {
        "ExternalId" : "[Code project]" 
      },

      /*
       * Dit wordt gebruikt als je een nieuw blokje aanmaakt via de tool.
       * (in dit geval niet van belang, maar wel een verplicht veld)
       */
      "DefaultAmount" : 1 
    },
    "entitySetName" : "projectcompetences",
    "method" : "UPSERT"
  }, {
    "entity" : {
      "Start" : "2024-02-10T07:00:00Z",
      "End" : "2024-02-10T15:00:00Z",
      "AllDay" : false, // Dit blokje werkt niet op hele dagen, maar met tijden.
      "Comments": "[Opmerkingen]",

      // De code voor het blokje uit het externe pakket.
      "ExternalId" : "[Code blokje]",

      // Zet vraagblokje op de juiste vraagregel.
      "ProjectRequestEntity" : {
        "ExternalId" : "[Code project]"
      },
    },
    "entitySetName" : "projectrequestplacements",
    "method" : "UPSERT",
    /*
     * Stel dat Start/End veranderd terwijl er al toekenningen zijn.
     * Dan kan het gebeuren dat toekenningen buiten dit vraagblokje komen te liggen.
     * Dat is niet tegestaan waardoor er een validatiefout ontstaat.
     * 1 van de 'fix options' is om de projecttoekenningen automatisch in te korten
     * tot de nieuwe Start/End (of eventueel te verwijderen als ze er helemaal buiten
     * komen te vallen).
     */ 
    "fixOptions": { "PROJECTASSIGNMENTS_OUTSIDE_RANGE": "CAP_TO_RANGE" },
    /*
     * Als Start veranderd, verplaats dan automatisch alle onderliggende toekenningen mee.
     * Merk op dat dit de 'default' is dus eigenlijk achterwege kan worden gelaten.
     * 'extraParamaters' is gewoonlijk alleen nodig als wij dat aangeven in overleg.
     */  
    "extraParameters": { "moveSubItems": true }
  }, {
    "entity" : {
      /*
       * projectcompetenceplacements kunnen automatisch worden geidentificeerd op basis van:
       * - het vraagblokje (projectrequestplacement)
       * - en de competentieregel (projectcompetence)
       */ 
      "ProjectRequestPlacementEntity" : {
        "ExternalId" : "[Code blokje]"
      },
      "ProjectCompetenceEntity" : {
        "ExternalId" : "[Code project]"
      },

      // Het aantal benodigde toekenningen.
      "Amount": 1,

      /*
       * De projectassignments (blokjes) die worden getoond.
       * Door een deep upsert te gebruiken worden eventuele 'oude' toekenningen voor deze 
       * competentie automatisch opgeruimd.
       */
      "Projectassignments_ProjectCompetencePlacement": [ {
        "ResourceEntity" : {
          // Eventueel automatisch deze resource aanmaken.
          "ResourceClass" : 2,
          "ExternalId" : "[Code Materiaal]",
          "Description" : "[Materiaal naam]",
          "ResourceTypeEntity": {
            "ResourceClass": 2, // Ook een ResourceType heeft een ResourceClass.
            "ExternalId": "[Code Materiaaltype A]",
            "Description": "[Materiaaltype A]"
          }
        },

        "Start" : "2024-02-10T07:00:00Z",
        "End" : "2024-02-10T15:00:00Z",
        "AllDay" : false,

        /*
         * HoursPerDay moet op 24 staan als AllDay false is.
         * Bij hele dagen geet HoursPerDay het aantal uren per dag.
         * HoursTotal kan worden gebruikt om een vast aantal uren te specificeren,
         * onafhankelijk van de duur (Start, End). Dan moet HoursPerDay op null worden ingesteld.
         */
        "HoursPerDay" : 24,

        /*
         * We moeten hier een samengestelde ExternalId gebruiken zodat deze globaal uniek
         * is voor projectcompetenceplacements.
         */
        "ExternalId" : "[Code blokje] [Code Materiaal]"
      } ]
    },
    "entitySetName" : "projectcompetenceplacements",
    "method" : "UPSERT"
  } ]
}
```

## **Questions and support**

For questions about the integration, you can contact **[support@planning.nl](mailto:support@planning.nl)**.
