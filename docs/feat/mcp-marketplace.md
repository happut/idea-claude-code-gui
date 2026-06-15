# MCP Marketplace – Integrationsanalyse & Stand

> Status: **Rahmenwerk als Patch vorhanden (`codriver-mcp-marketplace.patch`), noch nicht angewendet**
> Branch: `feature/mcp-marketplace`
> Erstellt: 2026-06-15 · Aktualisiert: 2026-06-15
>
> Architektur-Leitlinie: **reines Java-Backend, Webview nur zur Anzeige.**
> Der Patch setzt genau diese Linie um.

Dieses Dokument beschreibt, **wo** der MCP-Marketplace in den Settings andockt,
**was der vorbereitete Patch bereits liefert** und **was noch fehlt**.

---

## 1. Einstieg in der UI (Settings)

| Schritt | Datei |
| --- | --- |
| Sidebar-Eintrag `mcp` | `webview/src/components/settings/SettingsSidebar/index.tsx` (Zeile 4 + 17) |
| Routing auf die Section | `webview/src/components/settings/PlaceholderSection/index.tsx` (`type === 'mcp'` → `McpSettingsSection`) |
| Eigentliche MCP-Section | `webview/src/components/mcp/McpSettingsSection.tsx` |

Der Tab `mcp` ist als `SettingsTab` definiert und wird über die `PlaceholderSection`
an `McpSettingsSection` durchgereicht.

---

## 2. Die zentrale Integrationsstelle (der „Hook")

In `McpSettingsSection.tsx` öffnet das Dropdown des Add-Buttons den Eintrag
**„Add from MCP market"** über `handleAddFromMarket`. Vor dem Patch zeigte das nur
einen `mcp.marketComingSoon`-Toast; **der Patch biegt es auf den Marketplace-Dialog um**:

```tsx
// McpSettingsSection.tsx (nach Patch)
const handleAddFromMarket = useCallback(() => {
  setShowDropdown(false);
  setShowMarketplaceDialog(true);   // statt Coming-soon-Toast
}, []);
```

`onSelect` des Dialogs ist `handleSaveServer` – es läuft also derselbe Add-Pfad wie
beim manuellen Anlegen (`add_${prefix}mcp_server`). Kein neuer Persistenzpfad nötig.

---

## 3. Was der Patch liefert (`codriver-mcp-marketplace.patch`)

> `git apply --check` läuft **sauber** durch (nur neue Dateien + 4 kleine Einschübe).

### 3.1 Java-Backend (reine Logik, neues Package `mcp/marketplace`)

| Datei | Aufgabe |
| --- | --- |
| `handler/marketplace/McpMarketplaceHandler.java` | Bridge-Handler: `get_mcp_marketplace_sources`, `search_mcp_marketplace` → `window.updateMcpMarketplace*` |
| `mcp/marketplace/McpMarketplaceService.java` | Orchestriert Quellen: laden, **dedupe**, Suche/Filter, Sortierung (official → installable → Name), Cap 250 |
| `mcp/marketplace/McpMarketplaceSource.java` | Quellen-Modell + `defaults()` (siehe 3.3) |
| `mcp/marketplace/BuiltInMcpMarketplaceClient.java` | Die bisherigen 5 Presets als Built-in-Quelle |
| `mcp/marketplace/RegistryMarketplaceClient.java` | MCP-Registry v0.1 (paginiert via `next_cursor`) |
| `mcp/marketplace/GitHubOrgMarketplaceClient.java` | GitHub-Org-Repos (sterne-sortiert, paginiert) |
| `mcp/marketplace/McpRegistryEntryMapper.java` | Normalisiert Registry-/GitHub-JSON → Entry inkl. Install-Optionen: npm→`npx -y`, pypi→`uvx`, docker→`docker run`; Secret-/Header-**Platzhalter** |
| `mcp/marketplace/McpMarketplaceHttpClient.java` | HTTP GET + Platten-Cache (TTL 1 h, Stale-Fallback, optional `GITHUB_TOKEN`) |
| `mcp/marketplace/McpMarketplaceEntry.java` / `McpInstallOption.java` / `McpMarketplaceJson.java` | Modelle + JSON-Helper |
| `ui/ChatWindowDelegate.java` | Registriert `McpMarketplaceHandler` neben `McpServerHandler` |

### 3.2 Webview (nur Anzeige)

- `components/mcp/McpMarketplaceDialog.tsx` – Browser mit Suchfeld, **Quellen-Dropdown**,
  Liste, Detail-Panel, Install-Optionen-Auswahl und **Config-Preview** vor dem Hinzufügen.
- `McpSettingsSection.tsx` – `showMarketplaceDialog`-State, Dialog verdrahtet, `onSelect = handleSaveServer`.
- `types/mcp.ts` – `McpMarketplaceSource`, `McpInstallOption`, `McpMarketplaceEntry`, `McpMarketplaceSearchResponse`.
- `global.d.ts` – Callbacks `updateMcpMarketplaceSources` / `updateMcpMarketplaceEntries`.
- `styles/less/components/mcp.less` – Styles für den Dialog.

Der alte `McpPresetDialog` **bleibt unangetastet** – additiv, nichts wird kaputtgemacht.

### 3.3 Quellen (getrennt, im Dropdown wählbar) – `McpMarketplaceSource.defaults()`

| id | Name | Typ | URL |
| --- | --- | --- | --- |
| `built-in` | Built-in Presets | `BUILT_IN` | (lokal) |
| `official-registry` | Official MCP Registry | `REGISTRY` | `registry.modelcontextprotocol.io` |
| `github-mcp-registry` | GitHub MCP Registry | `REGISTRY` | `api.mcp.github.com` |
| `official-github-org` | MCP Official GitHub Org | `GITHUB_ORG` | `github.com/modelcontextprotocol` |

Dazu die Pseudo-Quelle **„All sources"** (`all`) im Dropdown, die quellenübergreifend sucht.

---

## 4. Abgleich mit den Anforderungen

| Anforderung | Status | Hinweis |
| --- | --- | --- |
| Reines Java-Backend, Webview nur Anzeige | ✅ erfüllt | Logik komplett in `mcp/marketplace/*` |
| Quellen getrennt | ✅ erfüllt | `SourceType {BUILT_IN, REGISTRY, GITHUB_ORG}` |
| Dropdown zur Quellenwahl | ✅ erfüllt | `marketplace-source-select` im Dialog |
| Vorhandenes nicht kaputtmachen | ✅ erfüllt | rein additiv, `McpPresetDialog` bleibt |
| **Favorit bleibt selektiert** (über Sessions/Öffnen) | ❌ **offen** | siehe 5. |

---

## 5. Offene Punkte

### 5.1 Persistenz der Quellenwahl (die einzige echte Lücke)

`selectedSourceId` im `McpMarketplaceDialog` ist reiner Component-State und startet
bei **jedem Öffnen** auf `DEFAULT_SOURCE_ID = 'built-in'`. Die Anforderung „mein
Favorit (z. B. Anthropic) bleibt ausgewählt, egal wie oft ich den Marketplace öffne"
ist damit **nicht** erfüllt.

Zwei Optionen:
- **localStorage im Webview (empfohlen):** `selectedSourceId` aus
  `localStorage` initialisieren und bei Änderung zurückschreiben
  (Key z. B. `codriver.mcp.marketplace.lastSourceId`). Folgt exakt dem bestehenden
  Muster `localStorage.setItem(cacheKeys.LAST_SERVER_ID, …)` in `McpSettingsSection`.
  Hält die Java-Schicht frei von View-Zustand. ~3 Zeilen.
- **Java-Settings-Service** (`CodemossSettingsService`): passt zur „alles in Java"-Linie,
  überlebt das Leeren des Webview-Storage – aber mehr Aufwand (Settings-Key +
  Lade-/Speicher-Roundtrip beim Öffnen). Begründung gegen die UI-Präferenz: es ist
  reiner Ansichtszustand, kein fachliches Setting.

### 5.2 i18n

`McpMarketplaceDialog.tsx` enthält hartkodierte englische Strings („Add Server",
„All sources" …). Restliche MCP-UI nutzt `t(...)`. Vor dem finalen Merge
i18n-isieren (`mcp.market.*` in `webview/src/i18n/locales/*.json`). Kein Blocker.

### 5.3 Endpoint-Verifikation `github-mcp-registry`

Die Quelle `github-mcp-registry` (`api.mcp.github.com`) wird als `REGISTRY`-Typ
(v0.1-Schema) behandelt. Endpoint/Schema vor dem Merge verifizieren – sonst landet
die Quelle still im Empty-/Stale-Fall.

---

## 6. Empfohlenes Vorgehen

1. Patch anwenden: `git apply codriver-mcp-marketplace.patch`.
2. **Persistenz** (5.1) per localStorage ergänzen → erfüllt die „Favorit bleibt"-Anforderung.
3. i18n (5.2) nachziehen.
4. `github-mcp-registry`-Endpoint (5.3) verifizieren oder Quelle vorerst deaktivieren.
5. Smoke-Test: Dialog öffnen, Quelle wechseln, suchen, Server hinzufügen (Claude **und** Codex).

> Spätere Politur: „installiert"-Badge (Abgleich mit `servers` aus `useServerData`),
> Secret-Eingabe über vorbefüllten `McpServerDialog` statt direktem Hinzufügen bei
> Einträgen mit Platzhaltern, Codex-spezifisches Mapping (`CodexMcpServerSpec`:
> `http_headers`, `bearer_token_env_var`).

---

## 7. „Offiziell" im MCP-Kontext (Hintergrund zur Quellenwahl)

MCP ist ein **offener Standard**, den sowohl Anthropic (Claude) als auch OpenAI
(Codex/ChatGPT) als *Clients* nutzen – dieses Plugin ist ohnehin multi-provider
(`apps: { claude, codex, gemini }`). Es gibt daher keine konkurrierenden
„Anthropic-" vs. „OpenAI-Registries". „Offiziell" meint hier die **neutrale** Quelle
des MCP-Projekts:

| Quelle | Art | Rolle im Patch |
| --- | --- | --- |
| `github.com/modelcontextprotocol/servers` (README-Liste) | kuratiert, **keine API** | Basis der `built-in`-Presets |
| `registry.modelcontextprotocol.io` | kanonische **Registry mit REST-API** | `official-registry`-Quelle |

Der Patch realisiert damit faktisch einen **Hybrid**: Built-in-Bundle als
Offline-Seed **plus** Live-Registry – ohne Drittanbieter-API-Key (anders als
Smithery/mcp.so, die bewusst nicht aufgenommen wurden).

---

## 8. Datei-Referenzen (Kurzfassung)

- Patch: `codriver-mcp-marketplace.patch` (Repo-Root)
- Hook-Punkt: `webview/src/components/mcp/McpSettingsSection.tsx` (`handleAddFromMarket`)
- Neuer Dialog: `webview/src/components/mcp/McpMarketplaceDialog.tsx` (Quellen-Dropdown, Persistenz-Lücke)
- Java-Backend: `src/main/java/com/github/claudecodegui/mcp/marketplace/*`
- Java-Handler: `src/main/java/com/github/claudecodegui/handler/marketplace/McpMarketplaceHandler.java`
- Handler-Registrierung: `src/main/java/com/github/claudecodegui/ui/ChatWindowDelegate.java`
- Add-Pfad (unverändert): `src/main/java/com/github/claudecodegui/handler/McpServerHandler.java`
- Typen: `webview/src/types/mcp.ts` (Marketplace-Typen am Dateiende)
- i18n (offen): `webview/src/i18n/locales/*.json` (`mcp.market.*`)
