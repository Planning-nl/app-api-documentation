# API app.planning.nl

De planningstool app.planning.nl biedt een API aan om gegevens uit te kunnen wisselen. Met deze API is het mogelijk om vrijwel alle gegevens uit te lezen en aan te passen. De API is geschreven op basis van het Odata protocol voor gegevensuitwisseling.
In dit document wordt beschreven hoe je met deze API aan de slag kunt gaan en hoe je deze kunt gebruiken.

## Installatie
Om de API aan te zetten logt u als beheerder in de applicatie in. Bij de *Gebruikers* maakt u een nieuwe gebruiker aan (of selecteert een bestaande gebruiker) die mag inloggen bij de API. De ingestelde rechten voor deze gebruiker (eventueel via de rol) zijn ook van toepassing op alle lees- en schrijfacties van de API.

Om te kunnen verbinden met onze API is een API token nodig.
1. Log in op app.planning.nl (met rechten op gebruikersbeheer)
2. Ga naar het formulier van de gewenste gebruiker
3. Ga naar het tabje *Authenticatie*
4. Zet hier de checkbox bij *API toegang* (of *Webservice*, afhankelijk van de versie)
5. op *Genereer API token* (of *Genereer webservice token*). Er verschijnt nu een token
6. Kopieer en plak deze token nu achter de url: `https://app.planning.nl/OData/V1/info?token=...`
7. Open deze URL in de browser. Er verschijnt nu een tabel met alle type entiteiten in het systeem

## Datamodel
De URL hierboven geeft dus een tabel met alle types in het systeem. Dit zijn er behoorlijk wat, want de types komen overeen met het complete datamodel. Welke types/velden er beschikbaar zijn is afhankelijk van de configuratie van uw omgeving. 
Wellicht heeft u voor uw doeleinden maar een paar types nodig. In de voorbeelden verderop zal duidelijker worden welke types voor welke use case van belang zijn.

### Velden
Door op een type te klikken wordt zichtbaar welke velden en relaties er beschikbaar zijn:
![image](https://github.com/Planning-nl/app-api-examples/assets/120531/81e933d9-43bf-4a0e-9218-46fc0dcf5583)

De beschikbare types en velden zijn afhankelijk van de configuratie van uw omgeving. 
Per veld is er verder een datatype (bijvoorbeeld Edm.String) gegeven. Dit is een Odata type. Voor het grote deel wijst dit zich vanzelf, maar het Edm.DateTimeOffset wordt geformatteerd als ISO UTC datetime voor de tijdzone Europe/Amsterdam. Hier is dus mogelijk een conversie nodig.

### Navigaties
Sommige velden hebben verder een **navigatie**. Dat betekend dat er een relatie bestaat met een andere entiteit. Bijvoorbeeld heeft personeel een relatie naar een ‘personnel_resourcetype’. Er zijn ook relaties beschikbaar naar collecties, die geven andere types die juist verwijzen naar een personeelslid.

## Gegevens uitlezen

Gegevens kunnen worden uitgelezen met een Odata GET request, bijvoorbeeld:
https://app.planning.nl/OData/V1/personnelcollection
(merk op dat we de token parameter verder weglaten)

*personnelcollection* is de entity set name zoals in de info tabel getoond wordt.
Dit geeft een JSON document met daarin alle personeelsleden binnen het systeem (waar de gebruiker leesrechten op heeft).

Er kan ook een enkel document worden opgevraagd op basis van een id: `/personnelcollection(4)`

Het Odata protocol bevat extra parameters om te kunnen filteren, sorteren, limiteren, enzovoorts. Hieronder staan een aantal voorbeelden.

### Filters
`/personnelcollection?$filter=ResourceType eq 2` 
Selecteer alle personeelsleden met resource type met id *2*

`/personnelcollection?$filter=ResourceTypeEntity/Number eq 'abc'` 
Sselecteer alle personeelsleden met resource type met als Number veld 'abc'

`/personnelcollection?$filter=year(Birthdate) ge 2000` 
Alle geboren vanaf het jaar 2000

`/personnelcollection?$filter=Resourcedepartments_Resource/any(d:d/DepartmentEntity/Description eq 'abc')` 
Personeel voor afdeling 'abc'

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
Er zit een limiet van 10’000 items in de repons. Dit is gedaan om onze server te beschermen tegen aanvragen zonder filters. Als uw opgevraagde set meer items heeft, dan zal er een `@odata.nextLink` veld, met een URL, in de response worden toegevoegd. Deze kan dan als odata request worden uitgevoerd om de volgende pagina met resultaten te krijgen. Dit kan recursief worden gedaan totdat er geen nextpage link meer in de respons staat.

### Odata URL genereren
In het nieuwe planbord kan je ook een Odata URL genereren door een tabel te openen. De filters en getoonde velden (tandwiel-knopje) worden via de knop *exporteren* > *Odata* naar een URL geexporteerd. Een voorbeeld:
![image](https://github.com/Planning-nl/app-api-examples/assets/120531/fed13fbe-90f6-4325-b77b-bbaef6ff3061)

Geeft als resultaat: `https://app.planning.nl/odata/departments?$filter=contains(Description, 'afd')&$select=Description,Id,SortIndex,Number,Color,Comments,CreatedAt,LastModifiedAt&$expand=CreatedByEntity($select=Description),LastModifiedByEntity($select=Description),DeletedByEntity($select=Description)`

## Gegevens schrijven
Het Odata protocol omvat ook het aanmaken, updaten en verwijderen van enteiten via POST, PATCH en DELETE Http requests.

### Aanmaken
`curl -X POST -H "Content-Type: application/json; charset=utf-8" -d @body.json "https://app.planning.nl/Odata/V1/personnelcollection?token=..."`

Met body.json:

`{"Firstname": "John", "Lastname": "Doe", "ExternalId": "951"}`

Merk op dat de ExternalId kan worden gebruikt om later te refereren naar dezelfde entiteit. Dit kan je dus gebruiken om het id van het 'andere' systeem in te zetten. Als bij een POST request dit ExternalId al gevonden wordt voor de aangegeven set (personnelcollection) dan wordt in plaats van een nieuw item ingevoegd, de bestaande geupdate.

### Updaten
`curl -X PATCH -H "Content-Type: application/json; charset=utf-8" -d @body.json "https://app.planning.nl/Odata/V1/personnelcollection(123)"`

Met body.json:

`{"Firstname": "Richard"}`

Hierbij wordt het item geupdate met id 123. In de praktijk wordt meestal een POST gebruikt met een ExternalId om de entiteit te identificeren.

### Verwijderen
`curl -X DELETE "https://app.planning.nl/Odata/V1/personnelcollection(123)"`

Het iem wordt verwijderd. Merk op dat dit ook alleen op basis van het id kan, terwijl in de praktijk vaker gebruik wordt gemaakt van ExternalId.
Merk op dat om deze reden vaak eerst het Id moet worden gevonden voor een ExternalId. Dit is foutgevoelig omdat het niet in een transactie gebeurd. Daarom gebruiken wij voor onze eigen tool deze endpoints eigenlijk niet, maar gebruiken we de **Batch API**.

### Batch API
Wat wij in Odata nog niet toereikend vonden was het schrijven van meerdere operaties tegelijk, binnen 1 transactie. Daarom hebben we bovenop de standaard Odata acties onze eigen **Batch API** toegvoegd, waarmee meerdere Odata acties atomisch kunnen worden uitgevoerd. Wij raden alle partijen die zelf een koppeling coderen aan om van deze Batch API gebruik te maken in plaats van standaard Odata requests. Merk op dat wij voor onze eigen tool ook gebruik maken van de Batch API. Dit heeft als voordeel dat het goed getest wordt. 

> Tip: in de Chrome Developer netwerk tab kan je zien hoe de Odata API gebruikt wordt.

Een batch request bestaat uit meerdere items:

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
        "Lastname" : "Doe2",
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
  "dryRun": true,
  "fixOptions": {}
}
```

### Opties

`items`: een array met `BatchRequestItem` objecten (later meer)

`rollbackOnError`: standaard `true`, zet op `false` als je falende items wilt negeren (en de rest wel wilt toepassen)

`skipItemsAfterError`: standaard `true`, zet op `false` als je na een falend item de rest van items toch wilt uitvoeren

`dryRun`: standaard `false`, zet op `true` om uiteindelijk de transactie te rollbacken (geen effect op de database, handig voor tests)

`fixOptions`: als er validatie-problemen optreden stelt de API soms een (of meerdere) *fixOption* voor, deze kan hier gespecificeerd worden

### BatchRequestItem

Alle items worden serieel uitgevoerd binnen dezelfde transactie. Je kan het vergelijken met het doen van losse Odata requests, maar dan in 1 keer.

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


### Fix options

Stel dat je een `projectphase` (vraagblokje) probeert te plaatsen buiten het project. Dan komt er gewoonlijk een validatiefout terug:

### Respons
