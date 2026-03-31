# Migrations

Claude Code uses a versioned migration system to update stored settings, preferences, and model aliases as the product evolves. On startup, `runMigrations()` in `main.tsx` checks `CURRENT_MIGRATION_VERSION = 11` against the stored version and runs any pending migrations.

---

## Migration Mechanism

```typescript
const CURRENT_MIGRATION_VERSION = 11

function runMigrations() {
  const storedVersion = getStoredMigrationVersion()  // from global config

  // Run all migrations from storedVersion+1 to CURRENT_MIGRATION_VERSION
  for (let v = storedVersion + 1; v <= CURRENT_MIGRATION_VERSION; v++) {
    runMigration(v)
  }

  // Async, non-blocking migration
  migrateChangelogFromConfig()

  // Save new version
  saveGlobalConfig({ migrationVersion: CURRENT_MIGRATION_VERSION })
}
```

---

## All 11 Migrations

### Migration 1: `migrateAutoUpdatesToSettings.ts`
**What:** Moves the `autoUpdates: false` preference from wherever it was previously stored into `settings.json` as `env.DISABLE_AUTOUPDATER`.

**Why:** Consolidated all user preferences into a single settings file.

**Telemetry:** `tengu_migrate_autoupdates_to_settings`

---

### Migration 2: `migrateBypassPermissionsAcceptedToSettings.ts`
**What:** Migrates the bypass permissions accepted flag into the settings file.

**Why:** Permission preference needed to live in a durable location.

---

### Migration 3: `migrateEnableAllProjectMcpServersToSettings.ts`
**What:** Migrates MCP server enable/disable state into settings.

**Why:** MCP server management was refactored to use the unified settings format.

---

### Migration 4: `migrateFennecToOpus.ts` (Ant-only)
**What:** Updates internal model aliases:
- `fennec-latest` → `claude-opus-4-6` (Opus)
- `fennec-latest[1m]` → `claude-opus-4-6[1m]` (Opus with 1M context)
- `fennec-fast-latest` → `claude-opus-4-6[1m]` + `fastMode: true`

**Why:** "Fennec" was an internal codename. Production users should never see this.

---

### Migration 5: `migrateLegacyOpusToCurrent.ts`
**What:** Maps legacy Opus model string identifiers to current default model.

**Why:** Model versioning cleanup — ensures no one is stuck on a deprecated model ID.

---

### Migration 6: `migrateOpusToOpus1m.ts`
**What:** For Max/Team Premium users: upgrades `opus` → `opus[1m]` (1 million token context).

**Why:** Opus with 1M context became the default for eligible tiers.

**Gate:** `isOpus1mMergeEnabled()`

---

### Migration 7: `migrateReplBridgeEnabledToRemoteControlAtStartup.ts`
**What:** Migrates bridge mode preference from one field to another.

**Why:** Bridge mode was renamed and the field moved.

---

### Migration 8: `migrateSonnet1mToSonnet45.ts`
**What:** For Pro users: `claude-sonnet[1m]` → `claude-sonnet-4-5[1m]`.

**Why:** Model version update — Sonnet 4.5 released and became the default.

---

### Migration 9: `migrateSonnet45ToSonnet46.ts`
**What:** For Pro/Max/Team Premium users: `claude-sonnet-4-5` → `claude-sonnet-4-6`.

**Why:** Sonnet 4.6 released and became the preferred default for all tiers.

**Telemetry:** `tengu_sonnet45_to_46_migration`

---

### Migration 10: `resetAutoModeOptInForDefaultOffer.ts`
**What:** Resets auto mode opt-in state for users who had previously opted in.

**Why:** Auto mode offer terms changed and users need to re-opt-in.

---

### Migration 11: `resetProToOpusDefault.ts`
**What:** Resets Pro tier default model back to Opus if it was changed.

**Why:** Default model reset needed after tier/offering changes.

---

## Non-Blocking Async Migration

`migrateChangelogFromConfig()` runs after the version-based migrations, fire-and-forget. It handles changelog data that doesn't need to block startup.

---

## Model String Reference

The migrations above map between these model IDs:

| Alias / Old Name | Current ID |
|---|---|
| `fennec-latest` | `claude-opus-4-6` |
| `fennec-fast-latest` | `claude-opus-4-6[1m]` + fastMode |
| `claude-opus-4-6[1m]` | Opus 4.6 with 1M context window |
| `claude-sonnet-4-5` | Sonnet 4.5 |
| `claude-sonnet-4-6` | Sonnet 4.6 (current default) |
| `claude-haiku-4-5-20251001` | Haiku 4.5 |
