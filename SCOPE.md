# Rules of Engagement (ROE) — hemmalabb red/blue

> **Båda agenterna (redteam-homelab och blueteam-homelab) MÅSTE läsa denna fil
> före varje uppdrag och följa den till punkt och pricka.**
> Uppdaterad: <ÅÅÅÅ-MM-DD>

## Auktorisering
Ägaren (`<owner@example.com>`) auktoriserar offensiv och defensiv säkerhetstestning
mot ENDAST de tillgångar som listas som IN SCOPE nedan. Detta är testning av egna
system i syfte att hitta och åtgärda säkerhetsluckor.

## ✅ IN SCOPE (får testas)
- **Hemmalabbet / hemnätverket** — alla hostar, tjänster, VM, containrar,
  router/brandvägg, VLAN, trådlöst (SSID), NAS, IoT-enheter som ägaren äger och driftar.
- **`<DITT-WEBBMÅL>`** (t.ex. `example.com`) — domän och tillhörande hostar/tjänster som ägaren äger.

### Inventarie (fyll i — agenterna frågar om detta saknas)
| Tillgång | IP / CIDR / värdnamn | Typ | Anteckning |
|----------|----------------------|-----|------------|
| Hemnät (LAN) | `<LAN-CIDR>` (t.ex. 10.0.0.0/24) | nätverk | fyll i |
| Router/brandvägg | `<ROUTER-IP>` | gateway | t.ex. pfSense/OPNsense |
| `<DITT-WEBBMÅL>` | | webb/domän | |
| ... | | | |

## ⛔ OUT OF SCOPE (får ALDRIG röras)
- **`<OUT-OF-SCOPE-PROJEKT>`** (t.ex. ett separat produktionsprojekt och dess servrar) — helt exkluderad.
- Alla domäner, IP, hostar och tjänster som ägaren **inte** äger.
- Tredjepartsleverantörer, molnkontroll-plan utanför egna kontogränser, ISP-utrustning
  som inte ägs, grannars nät, publika tjänster.
- Andra personers data.

## Regler
1. **Verifiera mål mot IN SCOPE innan varje aktiv handling.** Är ägarskap oklart → STOPP, fråga.
2. **Ingen pivotering utanför scope**, även om det är tekniskt möjligt.
3. **Undvik DoS/destruktivitet.** Ratebegränsa. Ta snapshot/backup före potentiellt
   destruktiva tester. Allt ska vara reversibelt.
4. **Hög-impact-handlingar** (radering, kryptering, persistens som överlever reboot,
   ändring av auth) kräver explicit bekräftelse per handling.
5. **Loggning:** all aktivitet dokumenteras i engagements/ med tidsstämpel och mål.
6. **Legit trafik:** ingen testning som stör ägarens produktion/familjens nätanvändning
   utan avisering.

## Arbetsflöde (purple team-loop)
- Red team → skriver fynd till `findings/` och taggar för blue team.
- Blue team → läser `findings/`, bygger detektion + härdning i `defenses/`, verifierar fix.
- Båda → loggar i `engagements/` och refererar varandras artefakter.
