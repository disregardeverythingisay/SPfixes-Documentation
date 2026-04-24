# Hyperlink-funktion – Automatisk linkgenkendelse

Dokumentation af implementeringen der automatisk gør URL'er klikbare i beskrivelser og kommentarer i AI Idéforum.

---

## Hvad løser det

Når en bruger skriver en URL i en beskrivelse eller kommentar (f.eks. `https://teams.microsoft.com/...`), blev den tidligere vist som almindelig tekst — ikke klikbar. Med denne funktion bliver URL'er automatisk omdannet til klikbare links med forkortet visningstekst.

---

## Implementering

### Ny fil: `linkify.tsx`

Kernen er en funktion `linkify(text)` der:

1. Søger efter URL-mønstre (`https://...`) i teksten med et regulært udtryk
2. Splitter teksten i skift mellem almindelig tekst og URL'er
3. Returnerer React-elementer — `<a>`-tags for links, plain strings for resten
4. **Bruger ikke `dangerouslySetInnerHTML`** — ingen XSS-risiko

```ts
export function linkify(text: string): React.ReactNode {
  const re = /https?:\/\/[^\s<>"]+/g;
  // ... splitter tekst og renderer <a>-tags
}
```

### URL-forkortelse

Visningsnavnet på linket forkortes automatisk til maks 42 tegn:

- `www.` fjernes fra domænet
- Domæne + sti vises (query string inkluderet)
- Overskydende tegn erstattes med `…`

```
https://www.sharepoint.com/sites/IT-Helpdesk/Lists/AIIndhold
→ sharepoint.com/sites/IT-Helpdesk/Lists/AIIndhold

https://teams.microsoft.com/l/channel/19%3a.../GenereltKanal
→ teams.microsoft.com/l/channel/19%3a...…
```

### Tegnsætning

Afsluttende tegnsætning der sandsynligvis tilhører sætningen og ikke URL'en strippes automatisk:

```
"Se mere her: https://example.com/page."
→ Link til https://example.com/page  (punktum tælles ikke med)
```

Tegn der strippes fra slutningen: `. , ; : ! ? ) " '`

### Sikkerhed

| Egenskab | Værdi |
|---|---|
| `target` | `_blank` — åbner i nyt faneblad |
| `rel` | `noreferrer noopener` — forhindrer den nye side i at tilgå `window.opener` |
| XSS | Ingen `dangerouslySetInnerHTML` — React escaper indholdet automatisk |
| Protokol | Kun `http://` og `https://` genkendes — ikke `javascript:` eller andre |

---

## Hvor bruges det

Funktionen er integreret i `IdeaDetailPanel.tsx` på to steder:

| Element | Placering |
|---|---|
| Beskrivelse | `<p className={panelSummary}>{linkify(idea.summary)}</p>` |
| Kommentarer | `<div className={commentText}>{linkify(item.text)}</div>` |

---

## Eksempler

| Brugerens input | Visning i panelet |
|---|---|
| `https://www.google.com` | `google.com` (klikbart) |
| `https://teams.microsoft.com/l/channel/abc123def456/Kanal` | `teams.microsoft.com/l/channel/abc…` (klikbart) |
| `Tjek https://example.com/side — super nyttigt!` | `Tjek ` + `example.com/side` + ` — super nyttigt!` |
| Almindelig tekst uden URL | Uændret |
