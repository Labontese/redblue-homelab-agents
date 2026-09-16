# Hemmalabbets red/blue team-arbetsyta

Två Claude Code-agenter som jobbar mot varandra i en **purple team-loop** mot
ÄGARENS EGNA tillgångar: hemnätverket + ett auktoriserat webbmål (t.ex. `example.com`).
Allt out-of-scope (t.ex. separata produktionsprojekt) och allt ägaren inte äger är
**out of scope** (se `SCOPE.md`).

> ⚠️ **Endast för egna system.** Kör bara mot tillgångar du äger och uttryckligen
> auktoriserar i `SCOPE.md`. Detta är ett verktyg för att härda din egen lab.

## Agenterna (installeras i `~/.claude/agents/`)
- **redteam-homelab** 🔴 — offensiv: hittar & bevisar säkerhetsluckor (full PoC).
- **blueteam-homelab** 🔵 — defensiv: detekterar, härdar & stänger luckorna.

## Så startar du dem
Be Claude Code använda en agent, t.ex.:
- "Använd **redteam-homelab** för att reka mitt hemnät (`<LAN-CIDR>`) och `example.com`"
- "Använd **blueteam-homelab** för att bygga detektion + fix för öppna fynd"
- Purple team: "Kör en red/blue-runda mot `example.com`" → red hittar, blue stänger,
  red försöker kringgå, blue stärker.

## Katalogstruktur
- `SCOPE.md` — Rules of Engagement (LÄS FÖRST). **Fyll i inventariet.**
- `agents/` — de två agentdefinitionerna (kopieras till `~/.claude/agents/`).
- `engagements/` — tidsstämplade loggar per körning.
- `recon/` — rå rekon-/skanningsdata.
- `findings/` — red teams fynd (mall: `_TEMPLATE-finding.md`, exempel: `EXAMPLE-F-0001-weak-csp.md`).
- `defenses/` — blue teams detektioner + fixar (mall: `_TEMPLATE-defense.md`, exempel: `EXAMPLE-F-0001-weak-csp.md`).
- `loot/` — bevis/PoC-artefakter (håll känsligt lokalt, dela inte).

## Exempelrapporter
`findings/EXAMPLE-F-0001-weak-csp.md` + `defenses/EXAMPLE-F-0001-weak-csp.md` visar en
komplett red→blue-runda (svag CSP → nonce-baserad fix + detektion) med den pedagogiska
nivån agenterna skriver i. **Syntetisk data** (`example.com`) — radera dem eller behåll
som referens.

## Innan första körning
1. Följ `INSTALL.md` för att installera agenterna och sätta din workspace-sökväg.
2. Öppna `SCOPE.md` och fyll i inventariet (LAN-CIDR, router-IP, webbmål-host, m.m.).
3. Se till att offensiva verktyg finns (WSL/Kali eller Docker) om red team ska köra dem.
