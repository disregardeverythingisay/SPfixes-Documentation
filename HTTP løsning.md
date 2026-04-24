# SharePoint REST API – HTTP-fejl og løsninger

Dokumentation af de tre SharePoint REST API-problemer der opstod under deployment af AI Idéforum webparten, og hvordan de blev løst.

---

## Problem 1: Listeoprettelse returnerer HTTP 500 i stedet for 409

### Symptom
```
Kunne ikke oprette liste/bibliotek 'AIIndhold'. HTTP 500
```

### Årsag
SharePoint Online burde returnere **HTTP 409 Conflict** når en liste allerede eksisterer, men returnerer i stedet **HTTP 500** på visse tenants. Det betød at koden fejlede, selvom listen var til stede.

### Løsning
Tilføj en verify-step efter 500: foretag et nyt GET-kald for at bekræfte at listen faktisk eksisterer. Behandl både 200 og 406 som "listen eksisterer".

```ts
if (createResponse.status === 500) {
  const verifyResponse = await this._spHttpClient.get(
    `.../_api/web/lists/getbytitle('${safeTitle}')?$select=Id`,
    SPHttpClient.configurations.v1,
    {}
  );
  if (verifyResponse.ok || verifyResponse.status === 406) return; // listen eksisterer
}
```

---

## Problem 2: Feltkontrol returnerer tomt ved brug af `$filter`

### Symptom
```
Kunne ikke oprette felt 'AIIdeaPayload'. HTTP 404
```

Fejlen opstod selvom feltet allerede eksisterede (bekræftet i SharePoint-kolonnevisningen).

### Årsag
`$filter=InternalName eq '...'` på fields-samlingen returnerer fejlagtigt et tomt array i visse SharePoint Online-tenants, selv når feltet er til stede. Koden fortolkede det som "felt mangler" og forsøgte at oprette det — `addfieldasxml` returnerede 404 fordi POST-body-formatet var forkert (manglede `__metadata`).

### Løsning
Erstat `$filter`-opslaget med `getbyinternalnameortitle()` — et direkte opslag der returnerer 200 (eksisterer) eller 404 (mangler):

```ts
// Før (upålidelig):
GET .../fields?$filter=InternalName eq 'AIIdeaPayload'&$select=InternalName&$top=1

// Efter (pålidelig):
GET .../fields/getbyinternalnameortitle('AIIdeaPayload')?$select=InternalName
```

Derudover kræver `addfieldasxml` at POST-body bruger `odata=verbose`-format med `__metadata`:

```ts
// Forkert (mangler __metadata):
{ parameters: { SchemaXml: "...", Options: 0 } }

// Korrekt:
{
  parameters: {
    __metadata: { type: 'SP.XmlSchemaFieldCreationInformation' },
    SchemaXml: "...",
    Options: 0
  }
}
// Headers: Accept + Content-Type skal begge sættes til application/json;odata=verbose
```

---

## Problem 3: Alle GET- og POST-kald returnerer HTTP 406

### Symptom
```
Kunne ikke hente opslag. HTTP 406
Kunne ikke oprette opslag. HTTP 406
The HTTP header ACCEPT is missing or its value is invalid.
```

### Årsag
`SPHttpClient.configurations.v1` sætter automatisk headeren `Accept: application/json;odata=nometadata`. Koden satte *samme* header igen i `_getHeaders()` og `_jsonHeaders()`. Den duplikerede `Accept`-header resulterede i en ugyldig headerværdi, som SharePoint afviste med 406.

### Løsning
Fjern `Accept` fra alle custom headers. Lad `SPHttpClient.configurations.v1` håndtere det alene. Angiv kun `Content-Type` på POST-kald:

```ts
// Før:
private _getHeaders(): ISPHttpClientOptions {
  return { headers: { Accept: 'application/json;odata=nometadata' } };
}
private _jsonHeaders(): ISPHttpClientOptions {
  return { headers: { Accept: 'application/json;odata=nometadata', 'Content-Type': 'application/json;odata=nometadata' } };
}

// Efter:
private _getHeaders(): ISPHttpClientOptions {
  return {}; // SPHttpClient.configurations.v1 sender Accept automatisk
}
private _jsonHeaders(): ISPHttpClientOptions {
  return { headers: { 'Content-Type': 'application/json;odata=nometadata' } };
}
```

---

## Oversigt

| Problem | HTTP-kode | Årsag | Løsning |
|---|---|---|---|
| Listeoprettelse fejler selvom listen eksisterer | 500 | SP returnerer 500 i stedet for 409 | Verify-GET efter 500 |
| Feltkontrol finder ikke eksisterende felter | 404 | `$filter` på fields er upålidelig | Brug `getbyinternalnameortitle()` |
| Alle API-kald afvises | 406 | Duplikeret `Accept`-header | Fjern Accept fra custom headers |
