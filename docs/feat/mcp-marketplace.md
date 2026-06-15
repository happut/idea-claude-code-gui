# MCP Marketplace – Integrationsanalyse

> Status: **Planung / noch nicht implementiert**
> Branch: `feature/mcp-marketplace`
> Erstellt: 2026-06-15

Dieses Dokument beschreibt, **wo** ein MCP-Marketplace in den Settings integriert
werden muss, **welche Bausteine schon vorhanden** sind und **was fehlt**.

---

## 1. Einstieg in der UI (Settings)

Die MCP-Verwaltung lebt komplett im Webview (React/TS), nicht im Java-Teil.

| Schritt | Datei |
| --- | --- |
| Sidebar-Eintrag `mcp` | `webview/src/components/settings/SettingsSidebar/index.tsx` (Zeile 4 + 17) |
| Routing auf die Section | `webview/src/components/settings/PlaceholderSection/index.tsx` (`type === 'mcp'` → `McpSettingsSection`) |
| Eigentliche MCP-Section | `webview/src/components/mcp/McpSettingsSection.tsx` |

Der Tab `mcp` ist als `SettingsTab` definiert und wird über die `PlaceholderSection`
an die dedizierte Komponente `McpSettingsSection` durchgereicht.

---

## 2. Die zentrale Integrationsstelle (der „Hook")

In `McpSettingsSection.tsx` gibt es bereits einen Button **„Add from MCP market"** im
Dropdown des Add-Buttons. Aktuell ist er aber nur ein Platzhalter:

```tsx
// webview/src/components/mcp/McpSettingsSection.tsx:216
// Add server from marketplace
const handleAddFromMarket = useCallback(() => {
  setShowDropdown(false);
  addToast(t('mcp.marketComingSoon'), 'info');   // <-- nur Toast „coming soon"
}, [t, addToast]);
```

Aufgerufen wird er im Dropdown-Menü:

```tsx
// McpSettingsSection.tsx:364
<div className="dropdown-item" onClick={handleAddFromMarket}>
  <span className="codicon codicon-extensions"></span>
  {t('mcp.addFromMarket')}
</div>
```

**→ Das ist die Stelle, an der der Marketplace angebunden wird.** Statt des
„coming soon"-Toasts muss hier ein Marketplace-Dialog geöffnet werden.

---

## 3. Bereits vorhandenes Gerüst (wiederverwendbar)

### 3.1 `McpPresetDialog` – ein „Mini-Marketplace", der nur verdrahtet werden muss

`webview/src/components/mcp/McpPresetDialog.tsx` existiert bereits und zeigt eine
Liste von kuratierten Servern (fetch, time, memory, sequential-thinking, context7)
mit Icon, Beschreibung, Tags und Typ-Badge an. Er ist **die offensichtliche Basis
für den Marketplace**.

Wichtige Beobachtungen:
- Der State `showPresetDialog` und das Rendering (`McpSettingsSection.tsx:437`) sind
  schon da, **werden aber nie aktiviert** – `setShowPresetDialog(true)` wird
  nirgends aufgerufen. Das Dropdown ruft stattdessen `handleAddFromMarket` (Toast).
- `handleSelectPreset` (`McpSettingsSection.tsx:246`) wandelt ein `McpPreset` bereits
  korrekt in einen `McpServer` um und persistiert ihn via `add_${prefix}mcp_server`.
- Die Presets sind **hartkodiert** in der Komponente, und die Dialog-Texte sind noch
  **hardcodierte chinesische Strings** (Zeilen 134, 142, 178, 182) statt `t(...)`.

### 3.2 Persistenz / Backend

Das Hinzufügen läuft über die Bridge an Java:

```tsx
sendToJava(`add_${messagePrefix}mcp_server`, server);
```

Behandelt in `src/main/java/com/github/claudecodegui/handler/McpServerHandler.java`
(`add_mcp_server`) bzw. `CodexMcpServerHandler.java` für den Codex-Modus.
Hier ist **keine Änderung nötig** – der Marketplace nutzt denselben Add-Pfad.

### 3.3 Typen

`webview/src/types/mcp.ts`:
- `McpPreset` (Zeile 112) – Modell eines Marketplace-Eintrags
- `McpServer` / `McpServerSpec` – Zielformat nach Auswahl

---

## 4. Was für einen echten Marketplace fehlt

Minimal-Variante (Presets sichtbar machen):
1. `handleAddFromMarket` umstellen auf `setShowPresetDialog(true)`.
2. `McpPresetDialog` i18n-isieren (chinesische Hardcodes → `t('mcp.market.*')`).
3. i18n-Keys ergänzen in `webview/src/i18n/locales/*.json`
   (bestehend: `mcp.addFromMarket`, `mcp.marketComingSoon` → ggf. ersetzen;
   `mcp.presets.*` existieren bereits).

Vollwertiger Marketplace (über statische Presets hinaus):
4. **Datenquelle/Registry** definieren → siehe Abschnitt 5.
5. **Suche/Filter/Kategorien** im Dialog (Tags sind im Modell schon vorhanden).
6. **Konfigurierbare Felder** vor dem Hinzufügen (env/headers/args), da viele
   Server Secrets/API-Keys brauchen – ggf. `McpServerDialog` vorbefüllt öffnen
   statt direkt zu persistieren.
7. Optional: „installiert"-Status (Abgleich mit `servers` aus `useServerData`).

---

## 5. Datenquelle – Entscheidung & Begründung

### Ausgangslage: „offiziell" ist hier nicht provider-gebunden

MCP ist ein **offener Standard**, den sowohl Anthropic (Claude) als auch OpenAI
(Codex/ChatGPT, Agents SDK) adoptiert haben. Dieses Plugin ist ohnehin
**multi-provider** (`apps: { claude, codex, gemini }` in `McpServer`). Es gibt also
keine konkurrierenden „Anthropic-" vs. „OpenAI-Registries" – beide sind nur
*Clients* desselben Protokolls. Damit ist die *neutrale* MCP-Quelle die richtige.

Zwei Kandidaten für „offiziell":

| Quelle | Art | Bewertung |
| --- | --- | --- |
| `github.com/modelcontextprotocol/servers` | Kuratierte README-Liste + Referenz-Server | „Offiziell", aber **keine API** (Markdown). Quelle der bestehenden 5 Presets. |
| `registry.modelcontextprotocol.io` | Kanonische **Registry mit REST-API** (offene Governance unter der `modelcontextprotocol`-Org) | **Das „offiziell" mit API.** Herstellerneutral, source-of-truth für Sub-Registries. |

### Entscheidung: **Hybrid (Bundle + offizielle Registry)**

- **Bundle** (erweiterte Preset-Liste, lokal): Offline-Fallback & schneller Start.
- **Live** aus `registry.modelcontextprotocol.io`: aktueller, neutraler Katalog,
  **kein Drittanbieter-API-Key** (anders als Smithery/mcp.so).

Verworfen:
- *Nur Bundle*: veraltet schnell, manuelle Pflege.
- *Smithery/mcp.so*: größter Katalog, aber Drittanbieter-Abhängigkeit + API-Key/Rate-Limits.

### Registry-Datenmodell → Mapping

Die Registry liefert Einträge mit u. a. `name`, `description`, `repository`,
`packages[]` (npm/pypi/oci …) und `remotes[]` (http/sse-URLs). Mapping auf `McpPreset`:

| Registry-Feld | → `McpPreset` |
| --- | --- |
| `name` / Title | `name`, `id` |
| `description` | `description` |
| `packages[].registry_type` + Name | `server.type='stdio'`, `command` (`npx`/`uvx`), `args` |
| `remotes[].url` (+ `type`) | `server.type='http'\|'sse'`, `url` |
| `repository.url` | `homepage` / `docs` |
| Kategorien/Keywords | `tags` |

> Hinweis: Aus einem npm/pypi-Paket den `command/args` ableiten (npm→`npx -y <pkg>`,
> pypi→`uvx <pkg>`). Diese Normalisierung gehört in einen `registryToPreset()`-Mapper.

### Transport: über Java, nicht direkt aus dem Webview

Das Webview läuft im JCEF und empfängt Java-Antworten über globale Callbacks
(`window.updateMcpServers(...)`, siehe `useServerData.ts:269-282`). Den
Registry-Fetch **über einen neuen Java-Handler** laufen lassen (statt `fetch()` im
Webview): umgeht CORS/Proxy-Themen, nutzt den vorhandenen HTTP-Stack und ist
konsistent mit der bestehenden Architektur.

---

## 6. Konkreter Implementierungsplan

### Phase 0 – Quick Win: Preset-Dialog sichtbar machen (rein Webview)
- `handleAddFromMarket` → `setShowPresetDialog(true)` statt Coming-soon-Toast
  (`McpSettingsSection.tsx:216`).
- `McpPresetDialog.tsx` i18n-isieren: hardcodierte CN-Strings (Z. 134, 142, 178, 182)
  → `t('mcp.market.*')`. Neue i18n-Keys in `webview/src/i18n/locales/*.json`.
- Ergebnis: funktionierender „Marketplace" auf Basis der 5 Bundle-Presets.
- Risiko: niedrig, keine Backend-Änderung.

### Phase 1 – Marketplace-Komponente & Modell
- `McpPresetDialog` → umbenennen/erweitern zu `McpMarketplaceDialog` (oder
  Marketplace als Tab innerhalb des Dialogs; Presets = „Featured").
- Preset-Liste aus der Komponente in eigene Datei auslagern:
  `webview/src/components/mcp/marketplace/bundledCatalog.ts`.
- Modell erweitern (`types/mcp.ts`): `McpMarketEntry` (= `McpPreset` +
  `category?`, `source: 'bundled' | 'registry'`, `installed?`, `secretsSchema?`).
- UI: Suchfeld + Kategorie-/Tag-Filter + „installiert"-Badge (Abgleich mit
  `servers` aus `useServerData`).

### Phase 2 – Offizielle Registry anbinden (Java + Webview)
- **Java**: neuer Handler `McpRegistryHandler` (analog `McpServerHandler`),
  Typen: `search_mcp_registry` (Query/Pagination) → HTTP GET gegen
  `registry.modelcontextprotocol.io`, Antwort via `window.updateMcpMarketplace(...)`.
  Registrieren wie die anderen Handler in der Handler-Registry.
- **Webview**: Hook `useMarketplaceData` (analog `useServerData`): sendet
  `search_mcp_registry`, registriert `window.updateMcpMarketplace`, merged
  Bundle + Registry, dedupliziert per `id`.
- **Mapper**: `registryToPreset()` (npm→`npx`, pypi→`uvx`, remotes→http/sse).
- Fehler/Offline → still auf Bundle zurückfallen.

### Phase 3 – Sicheres Hinzufügen (Secrets)
- Statt direkt `add_mcp_server`: bei Einträgen mit `env`/`headers`/`secretsSchema`
  den bestehenden `McpServerDialog` **vorbefüllt** öffnen, damit der Nutzer
  API-Keys eintragen kann, bevor persistiert wird.
- `handleSelectPreset` (`McpSettingsSection.tsx:246`) entsprechend verzweigen:
  Secrets nötig → Dialog; sonst → direkt hinzufügen (heutiges Verhalten).

### Phase 4 – Politur (optional)
- Caching des Registry-Ergebnisses (vgl. `utils/cacheManager.ts`).
- Detail-Ansicht pro Server (README/Tools-Preview).
- Codex-spezifisches Mapping prüfen (`CodexMcpServerSpec` weicht ab:
  `http_headers`, `bearer_token_env_var` etc.).

### Betroffene/neue Dateien (Überblick)
- ✏️ `webview/src/components/mcp/McpSettingsSection.tsx` (Hook verdrahten, Secrets-Branch)
- ✏️ `webview/src/components/mcp/McpPresetDialog.tsx` → `marketplace/McpMarketplaceDialog.tsx`
- ➕ `webview/src/components/mcp/marketplace/bundledCatalog.ts`
- ➕ `webview/src/components/mcp/marketplace/registryMapper.ts`
- ➕ `webview/src/components/mcp/hooks/useMarketplaceData.ts`
- ✏️ `webview/src/types/mcp.ts` (`McpMarketEntry`)
- ✏️ `webview/src/i18n/locales/*.json` (`mcp.market.*`)
- ➕ `src/main/java/com/github/claudecodegui/handler/McpRegistryHandler.java`
- ✏️ Handler-Registrierung (dort, wo `McpServerHandler` registriert wird)

---

## 7. Datei-Referenzen (Kurzfassung)

- Hook-Punkt: `webview/src/components/mcp/McpSettingsSection.tsx:216` (`handleAddFromMarket`)
- Dialog-Basis: `webview/src/components/mcp/McpPresetDialog.tsx`
- Auswahl→Server: `webview/src/components/mcp/McpSettingsSection.tsx:246` (`handleSelectPreset`)
- Sidebar: `webview/src/components/settings/SettingsSidebar/index.tsx`
- Routing: `webview/src/components/settings/PlaceholderSection/index.tsx`
- Typen: `webview/src/types/mcp.ts` (`McpPreset`, `McpServer`)
- Backend (Add-Pfad): `src/main/java/com/github/claudecodegui/handler/McpServerHandler.java`
- i18n: `webview/src/i18n/locales/*.json` (`mcp.addFromMarket`, `mcp.marketComingSoon`, `mcp.presets.*`)
