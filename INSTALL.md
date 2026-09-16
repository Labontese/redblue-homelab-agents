# Installation — red/blue homelab-agenter

Detta paket innehåller **ramverket** (två agenter + arbetsyta), inte någon riktig
engagement-data. Följ stegen nedan för att ta det i bruk mot din EGNA lab.

## 1. Välj en workspace-katalog
Kopiera hela detta paket dit du vill ha din arbetsyta, t.ex. `C:\sec\redsoc`
(Windows) eller `~/sec/redsoc` (Linux/macOS). Den katalogen kallas nedan `<WORKSPACE>`.

## 2. Installera agenterna
Kopiera de två filerna i `agents/` till din globala Claude Code-agentkatalog:

- **Global (alla projekt):** `~/.claude/agents/`
- **Endast detta projekt:** `<WORKSPACE>/.claude/agents/`

PowerShell (global):
```powershell
Copy-Item .\agents\redteam-homelab.md,.\agents\blueteam-homelab.md "$HOME\.claude\agents\"
```

bash (global):
```bash
cp agents/redteam-homelab.md agents/blueteam-homelab.md ~/.claude/agents/
```

## 3. Sätt din workspace-sökväg i agenterna
Agentfilerna refererar till arbetsytan som platshållaren `<WORKSPACE>`. Ersätt den
med din faktiska sökväg i **båda** agentfilerna (där du installerade dem).

PowerShell (byt ut sökvägen på rad 1):
```powershell
$ws = "C:\sec\redsoc"                       # <-- din workspace
Get-ChildItem "$HOME\.claude\agents\redteam-homelab.md","$HOME\.claude\agents\blueteam-homelab.md" |
  ForEach-Object {
    (Get-Content $_ -Raw).Replace('<WORKSPACE>', $ws) | Set-Content $_ -Encoding utf8
  }
```

bash:
```bash
WS="/home/you/sec/redsoc"                    # <-- din workspace
sed -i "s#<WORKSPACE>#${WS}#g" ~/.claude/agents/redteam-homelab.md ~/.claude/agents/blueteam-homelab.md
```

> Tips: du kan i stället låta `<WORKSPACE>` stå kvar och bara säga till agenten
> i chatten var arbetsytan ligger — men explicit sökväg är mer robust.

## 4. Fyll i scope
Öppna `<WORKSPACE>/SCOPE.md` och fyll i:
- Ditt LAN-CIDR och router-IP under **Inventarie**.
- Ditt webbmål (ersätt `<DITT-WEBBMÅL>` / `example.com`).
- Ägar-e-post och datum.
- Vad som är **out of scope** (ersätt `<OUT-OF-SCOPE-PROJEKT>`).

Agenterna vägrar aktiva handlingar mot mål som inte är tydligt IN SCOPE här.

## 5. Kör
I Claude Code:
- "Använd **redteam-homelab** för att reka mitt hemnät och `<DITT-WEBBMÅL>`"
- "Använd **blueteam-homelab** för att bygga detektion + fix för öppna fynd"

## Vad som medvetet INTE ingår
- Riktiga findings, recon, defenses, engagement-loggar (de innehöll skarp måldata).
- Alla domännamn, IP-adresser, DNS-poster, e-postadresser och personliga sökvägar
  från originaluppsättningen — ersatta med platshållare.

`engagements/`, `recon/` och `loot/` levereras tomma (endast `.gitkeep`).
`findings/` och `defenses/` innehåller bara sina mallar.

## Anpassa modell
Båda agenterna är satta till `model: opus` i frontmatter. Ändra till t.ex.
`sonnet` om du vill köra billigare/snabbare.
