# Handleiding app.planning.nl API

De planningstool app.planning.nl biedt een API aan om gegevens uit te kunnen wisselen. Met deze API is het mogelijk om vrijwel alle gegevens uit te lezen en aan te passen. De API is geschreven op basis van het OData protocol voor gegevensuitwisseling.

In dit document wordt beschreven hoe je met deze API aan de slag kunt gaan en hoe je deze kunt gebruiken.

## Installatie

Om te kunnen verbinden met onze API is een API token nodig.
1. Log in op app.planning.nl (met rechten op gebruikersbeheer)
2. Ga naar het formulier van de gewenste gebruiker
3. Ga naar het tabje *Authenticatie*
4. Zet hier de checkbox bij *API toegang* (of *Webservice*, afhankelijk van de versie)
5. Ga naar het tabje "Tokens" en druk rechts bovenin op "Nieuw"
6. Geef de token een een omschrijving en vink aan "Kan gebruikt worden als API token" en geef een einddatum (optioneel)
7. Druk op opslaan. Er verschijnt nu een token
8. Kopieer en plak deze token nu achter de url: `https://app.planning.nl/OData/V1/info?token=...`
9. Open deze URL in de browser. Er verschijnt nu een tabel met alle type entiteiten in het systeem

De ingestelde rechten voor deze gebruiker (eventueel via de rol) zijn ook van toepassing op alle lees- en schrijfacties van de API.

## Datamodel
De URL hierboven geeft dus een tabel met alle types in het systeem. Dit zijn er behoorlijk wat, want de types komen overeen met het complete datamodel. Welke types/velden er beschikbaar zijn is afhankelijk van de configuratie van uw omgeving.

Wellicht heeft u voor uw doeleinden maar een paar types nodig. In de voorbeelden verderop zal duidelijker worden welke types voor welke use case van belang zijn.

### Velden
Door op een type te klikken wordt zichtbaar welke velden en relaties er beschikbaar zijn:
![image](https://github.com/Planning-nl/app-api-examples/assets/120531/81e933d9-43bf-4a0e-9218-46fc0dcf5583)

De beschikbare types en velden zijn afhankelijk van de configuratie van uw omgeving.

Per veld is er verder een **datatype** (bijvoorbeeld `Edm.String`) gegeven. Dit is een OData type. Voor het grote deel wijst dit zich vanzelf, maar een `Edm.DateTimeOffset` wordt geformatteerd als ISO UTC datetime voor de tijdzone Europe/Amsterdam. Met bijvoorbeeld `2024-01-01T10:00:00Z` wordt dus een locale tijd van 11 uur aangegeven.

### Navigaties
Sommige velden hebben verder een **navigatie**. Dat betekend dat er een relatie bestaat met een andere entiteit. Bijvoorbeeld heeft personeel een relatie naar een *personnel_resourcetype*, genaamd `ResourceTypeEntity`. Er zijn ook relaties beschikbaar naar collecties, die geven andere types die juist verwijzen naar een personeelslid. Een voorbeeld hiervan is `Resourcedepartments_Resource`.

## Gegevens uitlezen

Gegevens kunnen worden uitgelezen met een OData GET request, bijvoorbeeld:
`https://app.planning.nl/OData/V1/personnelcollection`

> De token parameter laten we verder weg in de voorbeelden. Merk op dat deze desgewenst ook meegegeven kan worden als HTTP header via `X-API-KEY`.

*personnelcollection* is de entity set name zoals in de info tabel getoond wordt.
Dit geeft een JSON document met daarin alle personeelsleden binnen het systeem (waar de gebruiker leesrechten op heeft).

Er kan ook een enkel document worden opgevraagd op basis van een id: `/personnelcollection(4)`

Het OData protocol bevat extra parameters om te kunnen filteren, sorteren, limiteren, enzovoorts. Hieronder staan een aantal voorbeelden.

### Filters
`/personnelcollection?$filter=ResourceType eq 2`
Selecteer alle personeelsleden met resource type met id *2*

Merk op dat de query parameter URL-encoded moet worden. Dat is hier echter bewust niet gedaan in verband met de leesbaarheid.

Mocht je met een tool werken (zoals Azure Logic Apps) die de spaties encode naar `+` in plaats van `%20`, dan accepteerd onze API dit gewoonlijk niet. Echter, door de query parameter `&odata-accept-forms-encoding=true` aan de URLs toe te voegen worden de `+` karakters *wel* als spaties beschouwd.

`/personnelcollection?$filter=ResourceTypeEntity/Number eq 'abc'`
Sselecteer alle personeelsleden met resource type met als Number veld 'abc'

`/personnelcollection?$filter=year(Birthdate) ge 2000`
Alle geboren vanaf het jaar 2000

`/personnelcollection?$filter=Resourcedepartments_Resource/any(d:d/DepartmentEntity/Description eq 'abc')`
Personeel voor afdeling met de omschrijving 'abc'

Zie voor een lijst met beschikbare functies: https://docs.oasis-open.org/odata/odata/v4.0/errata03/os/complete/part2-url-conventions/odata-v4.0-errata03-os-part2-url-conventions-complete.html#_Filter_System_Query

### Selecteren
`/personnelcollection?$select=Firstname,Lastname`
Selecteer alleen de velden *Firstname* en *Lastname*

`/personnelcollection?$expand=ResourceTypeEntity($select=Description)`
Voeg het *Description* veld toe van het resource type

`/personnelcollection?$expand=Personnel_absenceassignments_Resource($select=Start,End;$expand=AbsenceTypeEntity($select=Description))`
Voeg alle afwezigheidstoekenningen toe met daarbij het afwezigheidstype

Merk op dat je binnen een expand ook weer kan filteren, sorteren, limiteren.

### Sorteren
`/personnelcollection?$orderby=Firstname`
Sorteer op voornaam

`/personnelcollection?$orderby=ResourceTypeEntity/Description`
Sorteer op resource type omschrijving

### Limiteren

`/personnelcollection?$top=10&$orderby=Firstname`
Sselecteer de eerste 10

`/personnelcollection?$top=10&$skip=10$orderby=Firstname`
Selecteer de volgende 10

`/personnelcollection?$top=0&$count=true`
Vraag alleen het aantal personeelsleden op

### Opvraaglimiet
Er zit een limiet van 10.000 items in de repons. Dit is gedaan om onze server te beschermen tegen aanvragen zonder filters. Als uw opgevraagde set meer items heeft, dan zal er een `@odata.nextLink` veld, met een URL, in de response worden toegevoegd. Deze kan dan als odata request worden uitgevoerd om de volgende 10.000 resultaten te krijgen. Dit kan recursief worden gedaan totdat er geen nextpage link meer in de response staat. Dan is de volledige set binnengehaald.

### OData URL genereren
In app.planning.nl kan je ook een OData URL genereren door een tabel te openen. De filters en getoonde velden (tandwiel-knopje) worden via de knop *exporteren* > *OData* naar een URL geexporteerd. Een voorbeeld:
![image](https://github.com/Planning-nl/app-api-examples/assets/120531/fed13fbe-90f6-4325-b77b-bbaef6ff3061)

Geeft als resultaat: `https://app.planning.nl/odata/departments?$filter=contains(Description, 'afd')&$select=Description,Id,SortIndex,Number,Color,Comments,CreatedAt,LastModifiedAt&$expand=CreatedByEntity($select=Description),LastModifiedByEntity($select=Description),DeletedByEntity($select=Description)`

> Let op! De URL werkt vanuit de applicatie en wijkt daarom net iets af van die voor de API. Vervang `/odata/` voor `/OData/V1/` om deze te converteren naar een URL voor de API.

### Aggregaties met $apply

Met de OData-queryoptie `$apply` kun je bewerkingen zoals aggregaties, groeperingen en berekeningen direct op de server uitvoeren. Dit is handig om bijvoorbeeld totalen, gemiddelden of gegroepeerde resultaten op te halen zonder dat je deze zelf hoeft te berekenen in je eigen code.

#### Wat kun je met `$apply`?
- **Aggregaties**: Som, gemiddelde, minimum, maximum, aantal, enz.
- **Groeperen**: Data groeperen op één of meerdere velden.
- **Berekeningen**: Nieuwe velden berekenen op basis van bestaande data.
- **Combinaties**: Bovenstaande bewerkingen combineren in één query.

#### Voorbeelden
- **Som van uren**:
  ```
  absenceassignments?$apply=aggregate(Hours with sum as TotalHours)
  ```
- **Groeperen en aggregeren**:
  ```
  absenceassignments?$apply=groupby((AbsenceType))/aggregate(Hours with sum as TotalHours)
  ```
- **Combineren van bewerkingen**:
  ```
  absenceassignments?$apply=compute(year(Start) as Year)/groupby((Year),aggregate(Hours with sum as TotalHours))
  ```

## Gegevens schrijven
Het OData protocol omvat ook het aanmaken, updaten en verwijderen van enteiten via POST, PATCH en DELETE Http requests.

### Aanmaken
`curl -X POST -H "Content-Type: application/json; charset=utf-8" -d @body.json "https://app.planning.nl/OData/V1/personnelcollection?token=..."`

Met body.json:

```json
{"Firstname": "John", "Lastname": "Doe"}
```

Er wordt nu een nieuwe entiteit aangemaakt.

### Upserten
Het is ook mogelijk om een bestaande entiteit te updaten via een `POST` request. Je kan op verschillende manieren refereren naar een bestaande entiteit.

#### Id
Als je een `Id` meegeeft dan wordt die bewuste entiteit geupdate. Als het `Id` is gespecificeerd dan moet deze bestaan, anders wordt er een foutmelding gegeven.

#### ExternalId
Als je een `ExternalId` meegeeft dan wordt de entiteit daarvoor gezocht. Als deze niet wordt gevonden wordt er een nieuwe entiteit aangemaakt.

#### Unieke sleutels
Voor diverse enteiten wordt er automatisch gecheckt op bepaalde unieke sleutels.

| Type | Kolomnamen |
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
Je kunt ook een `Matcher` meegeven. Je geeft dan meerdere kolomnamen mee waarop 'gematcht' dient te worden. Als er een entiteit bestaat met exact dezelfde waardes voor de kolommen, dan wordt deze geupdate. Als deze niet gevonden wordt, dan wordt er een nieuwe entiteit aangemaakt.
Je kunt dit bijvoorbeeld gebruiken door te matchen op het `Number` veld (`{"Matcher": "Number"}`). Als je wilt matchen op een relatie kan dit ook, via de kolom voor die relatie:

```json
{
  "ResourceTypeEntity": {"Number": "a", "Matcher": "Number"},
  "Custom_naam": "test",
  "Matcher": "ResourceType,Custom_naam"
}
```

Merk op dat de matcher exact hetzelfde werkt als de unieke sleutels.

### Updaten
`curl -X PATCH -H "Content-Type: application/json; charset=utf-8" -d @body.json "https://app.planning.nl/OData/V1/personnelcollection(123)"`

Met body.json:

```json
{"Firstname": "Richard"}
```

Hierbij wordt het item geupdate met id 123. In de praktijk wordt bij koppelingen meestal een POST gebruikt met een ExternalId om de entiteit te identificeren.

### Verwijderen
`curl -X DELETE "https://app.planning.nl/OData/V1/personnelcollection(123)"`

Het personeelslid met Id 123 wordt verwijderd. Merk op dat dit ook alleen op basis van het id kan, terwijl in de praktijk vaker gebruik wordt gemaakt van ExternalId.
Merk op dat om deze reden vaak eerst het Id moet worden gevonden voor een ExternalId. Dit is foutgevoelig omdat het niet in een transactie gebeurd. Daarom gebruiken wij voor onze eigen tool deze endpoints eigenlijk niet, maar gebruiken we de **Batch API**.

## Batch API
Wat wij in OData nog niet toereikend vonden was het schrijven van meerdere operaties tegelijk, binnen 1 transactie. Daarom hebben we bovenop de standaard OData acties onze eigen **Batch API** toegvoegd, waarmee meerdere OData acties atomisch kunnen worden uitgevoerd. Wij raden alle partijen die zelf een koppeling coderen aan om van deze Batch API gebruik te maken in plaats van standaard OData requests. Merk op dat wij voor onze eigen planningstool ook gebruik maken van deze Batch API. Dit heeft als voordeel dat het goed getest wordt en flexibel genoeg is voor vele toepassingen.

> Tip: in de Chrome Developer netwerk tab kan je zien hoe de OData API intern gebruikt wordt.

De batch API is beschikbaar onder de URL `https://app.planning.nl/OData/V1/batch`. Er moet een token of `X-API-KEY` header worden toegevoegd om te kunnen authenticeren.

Een batch request bestaat uit meerdere items. Voorbeeld:

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
Deze batch request voegt eerst een nieuwe *personeelstype* toe onder `ExternalId` "inleen" (als deze nog niet bestond), en stelt de `Description` in op "Inleen".

Vervolgens upsert hij een personeelslid en refereert daarbij aan het zojuist ingevoerde "inleen" type. Merk op dat dit ook als een **deep upsert** gedaan kan worden zoals bij `ResourceSortEntity`. Bij `Resourcedepartments_Resource` worden in 1 keer alle gekoppelde afdelingen van dit personeelslid *overschreven*, waarbij "Amsterdam" als hoofdafdeling wordt ingesteld.

### Opties

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

`items`: een array met `BatchRequestItem` objecten (later meer)

`rollbackOnError`: standaard `true`, zet op `false` als je falende items wilt negeren (en de rest wel wilt toepassen)

`dryRun`: standaard `false`, zet op `true` om uiteindelijk de transactie te rollbacken (geen effect op de database, handig voor tests)

`fixOptions`: als er validatie-problemen optreden stelt de API soms een (of meerdere) *fixOption* voor, deze kan hier gespecificeerd worden

`skipItemsAfterError`: standaard `true`, zet op `false` als je na een falend item de rest van items toch wilt uitvoeren

### BatchRequestItem

Alle items worden serieel uitgevoerd binnen dezelfde transactie. Je kan het vergelijken met het doen van losse OData requests, maar dan in 1 keer.

Een `BatchRequestItem` kan de volgende items bevatten:

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

Een aantal van de opties zijn ook op globaal niveau aanwezig. In dat geval overschrijven ze het globale niveau voor alleen dit specifieke item.

De `entitySetName` geeft aan op welke set de operatie van toepassing is (bijvoorbeeld *personnelcollection*).

De `BatchMethod` geeft aan *wat* voor operatie er moet worden uitgevoerd. Hieronder worden ze toegelicht.

### Batch Methods

De volgende methods worden ondersteund.

#### UPSERT

Een nieuw item invoegen of een bestaand item updaten. De `BatchRequestItem.entity` is verplicht en geeft de updates die moeten worden uitgevoerd. Deze operatie is gelijk aan de OData POST operatie. `entityId` is optioneel en kan gebruikt worden om een bestaand entity te updaten. Anders wordt de `Id` of `ExternalId` van de entity gebruikt ter identificatie.

> Merk op dat er voor bepaalde types ook andere velden gebruikt kunnen worden ter identificatie. Bijvoorbeeld bij `resourcedepartments` wordt, als je zowel een `Resource` als `Department` gespecificeerd, geprobeerd om een bestaand record te matchen en te updaten.

Voorbeeld:

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

Hetzelfde als `UPSERT`, maar geeft een foutmelding als de entity nog niet bestond.

#### DELETE

Equivalent aan de OData `DELETE` request method. `entityId` of `entity` kan gebruikt worden om te identificeren (net als bij `UPSERT`). Voorbeeld:

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

Met deze method is het mogelijk om op basis van de `entityFilter` (wat een filter is in dezelfde syntax als `$filter` bij een GET request) een set van entiteiten in 1 keer te updaten. De `entity` bevat de properties/navigaties die geupdate moeten worden.

Voorbeeld:

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

Met deze method is het mogelijk om op basis van de `entityFilter` een set van entiteiten in 1 keer te verwijderen.

Voorbeeld:

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

Stel dat je een `projectphase` (vraagblokje) probeert te plaatsen buiten het project. Dan komt er gewoonlijk een validatiefout terug. In de foutmelding wordt getoond wat voor opties er zijn:

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
            "description" : "Vergroot het bereik van de bovenliggende objecten"
          } ],
          "message" : "Vraag ligt buiten de periode van Project",
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

Door nu in de request (of in het request item) de volgende fix option toe te voegen wordt het project automatisch 'opgerekt':

```json
"fixOptions": { "PROJECTREQUESTPLACEMENT_OUTSIDE_PROJECT_RANGE": "EXTEND_PARENT_RANGE" }
```

### Response

Op een batch request komt er een response terug. Deze bevat per item het eindresultaat:

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

Als het veld `rollback` op `true` staat, dan is er niks gewijzigd in de database. Dat kan gebeuren doordat er fouten opgetreden zijn, of doordat de `dryRun` op `true` stond.

Als `noErrors` `true` is dan geeft dit aan dat de request OK was en zonder fouten is afgehandeld.

De `entity` of `entities` velden bevatten de resultaat entities na de operatie.

Als er fouten zijn opgetreden staat dat gewoonlijk bij het bewuste response item als `exception`. Dit kan bijvoorbeeld een authorizatiefout, een validatiefout of een interne fout zijn.

In sommige gevallen, bijvoorbeeld als er invalide json wordt aangeleverd, wordt er geen `BatchResponse` teruggestuurd maar een 'primitievere' foutmelding.

## Sessies

Authenticatie gaat via de `token` query parameter of de `X-API-KEY` header.

De server is stateless. Sessies worden direct na een request weer verwijderd. Het is niet (meer) nodig om cookies mee te sturen.

Merk op dat onze servers HTTP2 ondersteunen.

## Use cases

### Gegevens uitlezen
Stel je wilt periodiek alle afwezigheden vanaf de laatste 2 weken uitlezen uit app.planning.nl en importeren in een ander systeem.

Zoals hierboven beschreven kan je dat via een standaard OData URL doen. Een voorbeeld zou kunnen zijn: `https://app.planning.nl/OData/V1/absenceassignments?$filter=End gt (now() sub duration'P14D')`

Het is van belang om, als de set groter kan worden dan 10.000 items, de *nextLink* recursief aan te roepen om de gehele set op te kunnen halen.

Er zal waarschijnlijk een conversieslag moeten worden uitgevoerd (bijvoorbeeld de `Start` en `End` converteren naar het juiste formaat voor het doelsysteem), voordat de items in het doelsysteem worden geimporteerd.

### Personeelsleden synchroniseren

Stel je hebt een set objecten vanuit een extern systeem en wilt deze synchroniseren in app.planning.nl.

Het is van belang dat je aan de externe kant een uniek veld per personeelslid hebt. Dit kan bijvoorbeeld een Id uit dat pakket zijn of een personeelsnummer.

De koppeling zal periodiek alle gegevens uit het externe pakket moeten inlezen en converteren naar een batch request json bericht. Als `ExternalId` wordt bij de entiteiten het externe personeelsnummer gebruikt. Door de batch method `UPSERT` te gebruiken worden automatisch nieuwe personeelsleden toegevoegd, en bestaande geupdate.

Merk op dat op deze manier geen 'oude' personeelsleden opgeruimd worden. Omdat dit wel gewenst is wordt er een `DELETE_MULTIPLE` toegevoegd die wel checkt of het ExternalId is ingevuld, maar de lijst met nog bestaande Ids uitsluit:

```json
    {
      "entityFilter" : "filter=length(ExternalId) gt 0 and not (ExternalId in ('ab123', 'ab234'))",
      "entitySetName" : "personnelcollection",
      "method" : "DELETE_MULTIPLE"
    }
```

Merk op dat de lijst met ExternalIds in de filter vrij groot kan worden, maar gewoonlijk levert dit geen problemen op. Mocht dit wel problemen geven dan kunt u bij ons terecht voor advies.

### Projectstructuur aanmaken

Een externe partij wilde graag vanuit hun systeem automatisch een vraagblokje en toekenningsblokjes aanmaken in de tool. Bij wijzigingen zou ditzelfde blokje moeten worden geupdate.

De project structuur van app.planning.nl:
* vanuit een project kan een *vraagregel* worden toegevoegd
* aan een vraagregel kunnen *vraagblokjes* worden toegevoegd met een start/eind datum
* bij een vraagregel geef je de gewenste *projectcompetenties* op
* competenties zijn eigenschappen waaraan een *resource* (personeel of materieel) kan voldoen
* een vraagblokje specificeert voor elke projectcompetentie een aantal (of uren) toe te kennen resources
* aan gevraagde competenties van een vraagblokje kunnen *projecttoekenningen* (ook blokjes met start/eind) worden gekoppeld
* deze worden op een aparte regel voor de *resource* (personeelslid of materieel) getoond:

![image](https://github.com/Planning-nl/app-api-examples/assets/120531/d12b1125-cc44-4796-ba1d-574e65fbe92f)

Dit kan allemaal met de Batch API. Een voorbeeld bericht:

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

## Vragen en support

Voor vragen omtrent de koppeling kunt u terecht bij support@planning.nl.
