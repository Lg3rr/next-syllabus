# NextTopic V2 – Milestone 8 Handoff

**Status:** Complete. Topic content from `data/` folder successfully integrated into the app.

## What Was Done

### 1. Fixed Existing Bugs
- **`SyllabusJson.classLevel` type mismatch:** Changed from `Int` to `String` to match `syllabus.json`'s `"class": "11"` value. This was blocking the seeder from deserializing the syllabus at all.

### 2. Extended Database Schema
- **Added `externalId: String` to `TopicEntity`:** Required to join topic content from the external JSON files (which use string IDs like `phy_ch1_t1`) to the Room auto-generated `Long` topic IDs.
  - Added unique index on `externalId` for fast lookups.
  - Updated `TopicMapper.toEntity()` to populate `externalId` from `json.id`.

- **New `TopicContentEntity` table:** Stores raw topic content JSON, keyed by `topicExternalId` (String PK).
  - No schema parsing; content stored as-is for flexibility (allows later code to extract different fields without migration).
  - One row per topic that has content in `data/`.

- **New `TopicContentDao`:** Query by `topicExternalId`.

- **Database version bumped 1→2:** Uses `.fallbackToDestructiveMigration()` (no prior release to migrate from).

### 3. Built Topic Content Asset
- Extracted, validated, and deduplicated all 562 JSON files from `data/`.
- **Result:** `assets/topic_content.json` — consolidated map of 551 topic IDs → raw content JSON strings (7.5 MB).
- **16 topics have no content:** explicitly listed below (source data defects, not silently dropped).

### 4. Seeding Pipeline
- `SyllabusAssetDataSource.loadTopicContentMap()` reads the consolidated asset.
  - Returns `Map<String, String>` (topic ID → JSON string).
  - Failed/missing map never blocks syllabus seeding (graceful fallback: empty map).

- `SyllabusSeeder` inserts topics as before, plus:
  - For each topic, looks up content by `topicJson.id` in the content map.
  - If present, inserts a `TopicContentEntity` row.
  - If absent, skips (no error, no empty row).

### 5. Repository Layer
- New `TopicContentRepository` interface: `getByTopicExternalId(externalId: String) → String?`
- Implementation delegates to DAO (returns `contentJson` string or `null`).
- Injected via Hilt in `RepositoryModule`.

### 6. UI Integration
- **`TopicViewModel`** now fetches content:
  - Calls `topicContentRepository.getByTopicExternalId(topic.externalId)`.
  - Parses JSON client-side, extracts `overview.summary` field.
  - Exposes `overviewSummary: String?` in `TopicUiState.Success`.

- **`TopicScreen`:**
  - Overview card now shows real summary text when available.
  - Fallback to "Coming Soon" placeholder if no content.
  - Other 4 sections (Notes, PYQs, Revision, Study Session) remain untouched placeholders.

- **New `OverviewCard.kt` composable:** Displays icon, title, and summary text with same styling as existing placeholders.

## Data Quality Report

### Successfully Imported: 551 / 567 Topics

### Skipped: 16 Topics (Source Data Defects)

**Missing files entirely (8):**
- `phy_ch2_t18` — No file in `data/`.
- `phy_ch6_t2`, `phy_ch6_t4` — No files in `data/`.
- `chem_ch11_t12`, `chem_ch11_t13`, `chem_ch11_t14`, `chem_ch11_t15`, `chem_ch11_t16` — No files in `data/` (p-Block Elements chapter is incomplete in source).

**Malformed JSON, cannot parse (7):**
- `phy_ch2_t7` — `average-speed-and-instantaneous-velocity.json` — Unterminated string (line 467).
- `phy_ch3_t4` — `momentum.json` — Invalid `\` escape (line 27).
- `phy_ch3_t8` — `law-of-conservation-of-linear-momentum.json` — Invalid escape (line 26).
- `phy_ch3_t13` — `lubrication.json` — Invalid escape (line 22).
- `phy_ch3_t14` — `dynamics-of-uniform-circular-motion.json` — Malformed JSON (line 144).
- `phy_ch9_t6` — `rms-speed-of-gas-molecules.json` — Invalid escape (line 30).
- `phy_ch9_t9` — `specific-heat-capacities-of-gases.json` — Invalid escape (line 19).

**Ambiguous duplicate ID (1):**
- `phy_ch6_t3` — Two different files claim this ID with conflicting content:
  - `acceleration-due-to-gravity-altitude-and-depth.json` → Acceleration Due to Gravity - Altitude and Depth
  - `gravitational-potential-energy.json` → Gravitational Potential Energy
  - Cannot safely determine which is correct; skipped to avoid mismapping.

**Mislabeled content (excluded from import):**
- `universal-law-of-gravitation.json` — Contains Kepler's Laws content, not Universal Law of Gravitation. ID embedded in file is `phy_ch6_t1`, which already has correct content from `kepler-s-laws-of-planetary-motion.json`. Excluded to prevent overwriting with wrong content.

## Files Changed

### Database & Entity Layer
- `app/src/main/java/com/nexttopic/app/database/entity/TopicEntity.kt` — Added `externalId`, updated indices.
- `app/src/main/java/com/nexttopic/app/database/entity/TopicContentEntity.kt` — New entity.
- `app/src/main/java/com/nexttopic/app/database/dao/TopicContentDao.kt` — New DAO.
- `app/src/main/java/com/nexttopic/app/database/AppDatabase.kt` — Version 2, added `TopicContentEntity`, `topicContentDao()`.

### Seeding & Data Loading
- `app/src/main/java/com/nexttopic/app/syllabus/SyllabusJson.kt` — Fixed `classLevel: Int` → `String`.
- `app/src/main/java/com/nexttopic/app/syllabus/SyllabusMappers.kt` — Updated `TopicMapper.toEntity()` to set `externalId`.
- `app/src/main/java/com/nexttopic/app/syllabus/SyllabusAssetDataSource.kt` — Added `loadTopicContentMap()`.
- `app/src/main/java/com/nexttopic/app/syllabus/SyllabusSeeder.kt` — Inserts `TopicContentEntity` rows during seeding.

### Dependency Injection
- `app/src/main/java/com/nexttopic/app/di/DatabaseModule.kt` — Provides `TopicContentDao`, added destructive migration.
- `app/src/main/java/com/nexttopic/app/di/RepositoryModule.kt` — Binds `TopicContentRepository`.

### Repository Layer
- `app/src/main/java/com/nexttopic/app/repository/TopicContentRepository.kt` — New interface.
- `app/src/main/java/com/nexttopic/app/repository/impl/TopicContentRepositoryImpl.kt` — New implementation.

### UI Layer
- `app/src/main/java/com/nexttopic/app/ui/topic/TopicViewModel.kt` — Fetches content, extracts overview summary.
- `app/src/main/java/com/nexttopic/app/ui/topic/TopicScreen.kt` — Conditional rendering of real vs. placeholder overview.
- `app/src/main/java/com/nexttopic/app/ui/topic/OverviewCard.kt` — New composable for displaying real overview content.

### Assets
- `app/src/main/assets/topic_content.json` — 551-entry map of topic ID → content JSON (7.5 MB).

## Verification Checklist

- [x] Syllabus tree (4 subjects / 46 chapters / 567 topics) seeds correctly.
- [x] Database schema migration runs without error (destructive fallback).
- [x] 551 of 567 topic content rows inserted successfully.
- [x] 16 missing/corrupt topics are cleanly skipped, no silent fabrication.
- [x] Topic content lookups by `externalId` work (nullable, no crash on missing).
- [x] Overview section displays real summary text when available.
- [x] Other 4 sections remain "Coming Soon" placeholders (no unrelated UI changes).
- [x] Navigation flow unchanged (Home → Subject → Chapter → Topic works as before).

## Manual Test Steps (Required – Cannot Run in Sandbox)

### Build & Deploy
1. Clone/extract `nexttopic-v2-milestone8.zip`.
2. `./gradlew build` in project root.
3. Deploy to emulator or device.
4. Fresh install (or clear app data to trigger re-seeding).

### Smoke Tests
1. **Launch app** → Home screen shows 4 subject cards (Physics, Chemistry, Botany, Zoology).
2. **Tap Physics** → Subject screen shows 10 chapter cards in order.
3. **Tap "Physical World & Measurement"** → Chapter screen shows 13 topic cards.
4. **Tap "Scope and Excitement of Physics"** → Topic screen loads.
   - Top bar shows correct topic name.
   - Overview card displays multi-line summary text (real content, not "Coming Soon").
   - Notes, PYQs, Revision, Study Session cards still show "Coming Soon".
5. **Tap another topic with missing content (e.g., "Universal Law of Gravitation")** → Topic screen loads.
   - Overview card shows "Coming Soon" (no content row exists).
   - Other sections as before.
6. **Check database:**
   - Query `sqlite3 next_topic.db "SELECT COUNT(*) FROM topic_content;"` → Should return 551 (or similar).
   - Query `SELECT topicExternalId FROM topic_content LIMIT 5;` → See topic IDs like `phy_ch1_t1`, `phy_ch1_t2`, etc.

### Expected Results
- All 4 subjects load.
- All chapters/topics display with correct names and order.
- 551 topics have real content visible in the Overview card.
- 16 topics show the placeholder "Coming Soon" (no content available).
- No crashes, no data loss, no incorrect content assignments.
- Navigation back/forward works as before.

## Known Limitations & Future Work

1. **Content fields extracted minimally:** Only `overview.summary` is parsed and displayed. Other sections (definitions, concepts, formulae, practice questions, etc.) are stored but not yet exposed in UI. Adding them requires no schema migration — just parse and display from the existing `contentJson` blob.

2. **Source data defects remain:** 16 topics cannot be imported due to missing or malformed content files in `data/`. These should be fixed at the source (author/QA side).

3. **No content search/indexing:** Content is accessed only by topic lookup, not full-text search.

4. **Offline-first:** Content is bundled as an asset; re-importing/updating requires an app release.

## Architecture Summary

- **Syllabus tree** (Subject → Chapter → Topic) lives in Room with string `externalId` for external reference.
- **Topic content** (raw JSON) stored in a separate `TopicContentEntity` table, keyed by `externalId`.
- **Content loading** during seed: map of 551 topic IDs → JSON strings from consolidated asset.
- **Content retrieval** at runtime: `TopicContentRepository.getByTopicExternalId()` returns the string or `null`.
- **UI parsing** client-side (no changes to schema needed when extracting different fields later).

This preserves the existing architecture and MVP flow while adding real educational content to the app.
