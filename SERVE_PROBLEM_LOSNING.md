# Serve-problem og losning

## Problem

Projektet kunne installeres og bygges fint, men `serve` fejlede under opstart af SPFx dev-serveren.

Det synlige symptom var:

```text
[build:webpack] Error: spawn EPERM
Encountered 1 error
  [build:webpack] spawn EPERM
```

Der var ogsa disse beskeder under serve:

```text
[build:lint] Linting isn't currently supported in watch mode
[build:configure-webpack-serve] Placeholder {tenantdomain} was found in serve.json but OS variable SPFX_SERVE_TENANT_DOMAIN is not set.
```

Lint-beskeden er normal for watch mode, og `{tenantdomain}`-beskeden stoppede ikke serveren. Den reelle blocker var `spawn EPERM`, som opstod fordi dev-serveren/browser-starten blev koert i en begranset shell/sandbox-kontekst.

Derudover fejlede et CLI-tjek mod `https://localhost:4321/temp/build/manifests.js` med en Windows Schannel/TLS-fejl, selv efter serveren var startet. Webpack-loggen viste dog at dev-serveren koerte korrekt paa localhost.

## Losning

1. Installer dependencies:

```powershell
npm install
```

2. Trust SPFx dev-certifikatet:

```powershell
npx heft trust-dev-cert
```

3. Kontroller at projektet bygger:

```powershell
npm run build
```

4. Start dev-serveren uden automatisk browser-launch:

```powershell
npx heft start --clean --nobrowser
```

Hvis kommandoen koeres fra et miljo med sandbox/rettighedsbegransninger, skal den koeres uden den begransning. Det var dette der fjernede `spawn EPERM`.

## Resultat

Dev-serveren startede korrekt og lyttede paa:

```text
https://localhost:4321/
```

Webpack viste:

```text
Started Webpack Dev Server at https://[::1]:4321/
webpack 5.95.0 compiled successfully
Waiting for changes. Press CTRL + C to exit...
```

SharePoint-debug URL:

```text
https://strmhansenas.sharepoint.com/sites/IT-Helpdesk/SitePages/AI.aspx?debugManifestsFile=https%3A%2F%2Flocalhost%3A4321%2Ftemp%2Fbuild%2Fmanifests.js&debug=true&noredir=true
```

## Noter

- `npm run build` gennemfoerte og genererede pakken `sharepoint\solution\spfx-ai-ideforum.sppkg`.
- Buildet havde lint-advarsler, men ingen build-fejl.
- `npm install` rapporterede npm audit-advarsler. De blev ikke automatisk rettet, fordi `npm audit fix` kan aendre dependency-versioner og dermed risikere SPFx-kompatibiliteten.
