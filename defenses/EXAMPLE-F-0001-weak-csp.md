---
finding_id: F-0001
date: 2026-01-01
control_type: både   # detektion + härdning
author: blueteam-homelab
target: https://example.com (HTTP-svarsheaders)
---

# F-0001 (EXEMPEL) — Stark Content-Security-Policy med nonce

> 📚 **EXEMPEL — syntetisk data.** Påhittad demonstration mot `example.com` som visar
> hur ett blue team-försvar dokumenteras (detektion + härdning + pedagogik). Kodexemplen
> är ramverks-neutral pseudokod — anpassa till din stack (Phoenix-plug, Express-middleware,
> Django-middleware, ASP.NET, Nginx m.m.).

## Baslinje (den svaga policyn från findingen)
```
content-security-policy: base-uri 'self'; frame-ancestors 'self';
```
Många ramverk sätter en minimal default-CSP. Fixen är att **överskugga** den headern med en
fullständig, restriktiv policy.

**App-fakta som styr policyn (exempel — kartlägg motsvarande i din app):**
- WebSocket-realtid → `wss://example.com` behövs i `connect-src`.
- App-JS laddas från egen origin (`/assets/...`) → täcks av `'self'`.
- Self-hostad analytics på `https://analytics.example.com`, initierad via ett **inline
  `<script>`** → behöver `'nonce-...'` på inline-taggen + hosten i `script-src` och `connect-src`.

---

## Så fungerar det — mekanismen (för lärlingen)

**CSP är en allowlist som webbläsaren tvingar igenom.** Sidan deklarerar exakt *varifrån*
resurser (script, stilar, bilder, anslutningar) får laddas, och webbläsaren vägrar allt
annat. Det är ett *andra lager under* korrekt kodning: om en XSS-lucka ändå smyger in ska
CSP:n hindra att den gör skada.

**Varför en nonce stoppar injicerat script:** med `script-src 'self' 'nonce-<slump>'` kör
webbläsaren bara (a) script från din egen origin och (b) inline-script som bär *exakt* det
slumpmässiga nonce-värde servern satte just denna request. Nonce genereras om per sidladdning
och är oförutsägbart. En angripare som lyckas injicera `<script>…</script>` via t.ex. ett
kommentarsfält vet inte nonce-värdet → webbläsaren vägrar köra scriptet. Det är därför vi
*inte* använder `'unsafe-inline'` för script — det skulle säga "kör allt inline" och öppna
precis den dörr vi vill stänga.

**Varför report-only först:** `Content-Security-Policy-Report-Only` får webbläsaren att
*rapportera* vad som *skulle* ha blockerats — utan att faktiskt blockera. Så hittar du
legitima resurser du glömt (en font-host, en WS-endpoint) *innan* du sätter policyn skarp och
riskerar att bryta sidan.

**connect-src och realtid:** en WebSocket-baserad realtidskanal (`wss://example.com`) styrs av
`connect-src` (som reglerar vilka URL:er JS får öppna via fetch/XHR/WebSocket). Utan den bryts
realtidsfunktionen. Analytics som skickar events via fetch måste också stå i `connect-src`.

---

## Härdning / Fix

### Målpolicyn (rulla ut i report-only först)
```
default-src 'self';
script-src 'self' 'nonce-<per-request>' https://analytics.example.com;
style-src 'self' 'unsafe-inline';
img-src 'self' data: https:;
font-src 'self' data:;
connect-src 'self' wss://example.com https://analytics.example.com;
worker-src 'self';
manifest-src 'self';
frame-src 'none';
frame-ancestors 'self';
base-uri 'self';
form-action 'self';
object-src 'none';
report-uri /csp-reports;
report-to csp-endpoint;
```

### Val av plats: **i appen** (inte som statisk edge-header) om du använder nonce
En nonce kräver ett per-request-genererat värde som matchar inline-scriptets `nonce`-attribut.
Bara appen kan generera det. En statisk header vid CDN/reverse-proxy (t.ex. Cloudflare Transform
Rule eller Nginx `add_header`) kan inte stödja nonce → passar bara om du tar bort alla inline-scripts.

### Middleware — pseudokod (anpassa till ditt ramverk)
```
# Körs per request, EFTER ev. default-header-middleware så den överskuggar:
function security_headers(request, response):
    nonce = base64url(random_bytes(18))          # nytt per request
    request.context.csp_nonce = nonce            # exponera till templaten

    csp = join("; ", [
        "default-src 'self'",
        "script-src 'self' 'nonce-" + nonce + "' https://analytics.example.com",
        "style-src 'self' 'unsafe-inline'",
        "img-src 'self' data: https:",
        "connect-src 'self' wss://example.com https://analytics.example.com",
        "object-src 'none'", "base-uri 'self'", "frame-ancestors 'self'",
        "form-action 'self'", "report-uri /csp-reports"
    ])

    # ROLLOUT-toggle: report-only först, enforce sen (styr via env-flagga)
    header = CSP_ENFORCE ? "Content-Security-Policy" : "Content-Security-Policy-Report-Only"
    response.set_header(header, csp)
```

### Sätt nonce på inline-scriptet i din template
```html
<script nonce="{{ csp_nonce }}">
  /* ...befintlig inline-snippet (t.ex. analytics-init)... */
</script>
```
Gäller ALLA inline `<script>`. Externa `<script src="/assets/...">` behöver ingen nonce (täcks av `'self'`).

- **Backup-plats:** den gamla CSP-strängen antecknad i baslinjen ovan + git-historik.
- **Reversibel:** Ja — ta bort middlewaren (default-CSP återgår) och revertera template-ändringen.

### Alternativ om nonce känns invasivt (hash)
Är inline-snippet statiskt → beräkna `sha256` av innehållet och lägg `'sha256-<base64>'` i
script-src istället. Nackdel: hashen måste uppdateras varje gång snippet ändras.

---

## Utrullningsordning (säker)
1. Deploya med report-only-flaggan **av** enforce → headern blir `Content-Security-Policy-Report-Only`.
   Sidan påverkas inte; brott rapporteras.
2. Använd appen normalt i några dagar. Samla in violations (se Detektion).
3. Rätta ev. brott (lägg till host/nonce där något legitimt blockerades).
4. När rapporterna är rena: slå på enforce → `Content-Security-Policy`.

---

## Detektion (defense-in-depth — XSS-försök/felkonfig ska synas)

### 1. CSP violation-rapportering
Policyn skickar brott till `/csp-reports` (report-uri + report-to). Ta emot dem i appen och
logga dem, eller peka mot en gratis collector (t.ex. report-uri.com free / Sentry) för
dashboard + larm utan egen kod.

Pseudokod för egen endpoint:
```
POST /csp-reports:
    body = read_raw_body(limit = 1MB)
    log.warning("[csp-violation] " + body)
    return 204
```

### 2. Sigma-regel (host/applog) på CSP-violations
```yaml
title: CSP-violation på webbappen (möjlig XSS eller felkonfig)
id: 00000000-example-csp-0001
status: experimental
description: Triggar på CSP violation-rapporter loggade av webbappen.
logsource:
  product: webapp
  service: application
detection:
  selection:
    message|contains: '[csp-violation]'
  # Höj allvar om ett script-src-brott pekar på extern origin (exfil-försök):
  suspicious:
    message|contains:
      - '"violated-directive":"script-src'
      - '"blocked-uri":"http'
  condition: selection
  # för larm: selection and suspicious
level: medium
falsepositives:
  - Nyligen tillagd legitim tredjepart som ännu ej whitelistats (justera policyn).
  - Browser-extensions som injicerar script (vanligt brus — filtrera bort inline-eval).
```
- **Var den körs:** över appens logg i logg-pipelinen.
- **Triggar på:** varje CSP-brott; höjd nivå vid script-src-brott mot extern URL.
- **False-positive-risk:** medel — browser-extensions genererar mycket brus i report-only.
  Filtrera på `violated-directive` = script-src och externa `blocked-uri`.

---

## Detektionslogik förklarad

`report-uri`/`report-to` säger till webbläsaren: "om du blockerar något pga CSP, POSTa en
JSON-rapport hit." Varje rapport innehåller bl.a. `violated-directive` (vilken regel som
bröts), `blocked-uri` (vad som försökte laddas) och `document-uri` (var det hände).

**Vad du letar efter — och varför:**
- Ett **`script-src`-brott** med `blocked-uri` mot en *extern* domän du inte känner igen →
  någon försöker ladda skript utifrån. På en produktionssida med intrimmad policy är det en
  stark XSS-indikator: en injicerad script-tagg som pekar på angriparens server (T1059.007).
- **Upprepade** brott mot samma externa host → antingen en attack eller en tredjepart du
  glömt whitelista.

**Så resonerar du kring false positives:**
- Browser-extensions och antivirus injicerar ofta egna script/stilar i sidan. De ger
  CSP-brott som *ser* läskiga ut men är ofarliga och kommer från *klientens* dator, inte din
  app. Därför är report-only initialt bullrigt.
- Sållningen: larma på `violated-directive: script-src` **och** extern `blocked-uri` med
  http(s)-schema; ignorera brott med scheman som `chrome-extension:` / `moz-extension:` och
  rena `inline`/`eval`-brott från extensions. Logga resten utan att larma.

---

## Verifiering
```bash
# Report-only-fasen:
curl -sSI https://example.com/login | grep -i 'content-security-policy-report-only'
# Enforce-fasen (ska visa fullständig policy med default-src/script-src):
curl -sSI https://example.com/login | grep -i 'content-security-policy:'
```
- Bekräfta i webbläsarens DevTools → Console att inga legitima resurser blockeras
  (realtids-WS ansluter, analytics laddar) innan enforce.
- **PoC-retest (stänger luckan):** ett injicerat `<script src="https://evil.example/x.js">`
  eller inline `<script>alert(1)</script>` ska i enforce blockeras av CSP (syns som
  violation, körs ej). Nonce hindrar godtyckliga inline-scripts; `script-src` utan
  wildcard hindrar extern skriptinladdning utanför de whitelistade hostarna.

## Kvarvarande risk
- **`style-src 'unsafe-inline'`** behålls pragmatiskt (många JS-ramverk sätter inline-stilar).
  Låg risk — stilar kör inte kod. Kan senare skärpas med nonce/hash.
- **`img-src ... https:`** tillåter bilder från valfri https-origin. Låg risk (bilder
  exekverar ej), men medger exfil-pixel-mönster. Kan skärpas när alla bild-hosts är kända.
- CSP är **defense-in-depth**, inte ersättning för korrekt output-encoding. Fortsätt sanera/
  encoda användarinnehåll korrekt vid rendering.
- En self-hostad, whitelistad analytics-host blir en betrodd källa — håll den patchad och
  åtkomstskyddad.

## Rekommendation till ägaren
1. Deploya middleware + nonce i **report-only**, kör några dagar.
2. Rätta ev. brott, växla till **enforce**.
3. Aktivera `/csp-reports` + Sigma/larm för löpande insyn.

## Väg till `status: mitigated`
Ägaren deployar report-only → verifierar inga legitima brott → växlar till enforce →
blueteam bekräftar fullständig CSP via `curl -I` + DevTools → red team försöker
skriptinjektion och bekräftar att extern/inline-script blockeras. Då `mitigated`.

---

## Begrepp & lärdomar
- **CSP** — webbläsar-allowlist för resurser; defense-in-depth mot XSS.
- **Nonce vs hash** — två sätt att godkänna *specifika* inline-script utan `'unsafe-inline'`.
- **`'unsafe-inline'` / `'unsafe-eval'`** — de "farliga" nyckelorden vi medvetet undviker för script.
- **`'strict-dynamic'`** — låter ett betrott script ladda mer script; ignorerar host/`'self'`
  (kraftfullt men lätt att sätta fel — testa noga).
- **report-only vs enforce** — rapportera-utan-att-blockera vs faktiskt blockera.
- **Direktiv** — `default-src` (fallback), `script-src`, `style-src`, `connect-src`,
  `frame-ancestors`, `object-src` m.fl.
- **Lärdom:** CSP *ersätter inte* korrekt output-encoding — det är ett skyddsnät, inte fixen för XSS.

## Läs mer
- **MDN — Content Security Policy:** https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP
- **OWASP Secure Headers Project:** https://owasp.org/www-project-secure-headers/
- **OWASP CSP Cheat Sheet** — praktiska mönster och fallgropar.
