# Milestone 6 Handoff — Chapter Screen

**Status:** Complete. Chapter Screen implemented following the Subject Screen pattern (M5). Project should compile — not run through Gradle in this sandbox (no network/JDK toolchain access); verify locally before merging.

## What was built

### ChapterViewModel (`ui/chapter/ChapterViewModel.kt`)
- Receives `chapterId` via `SavedStateHandle` (existing nav scaffolding).
- `ChapterUiState` sealed interface: `Loading`, `Success(chapter, topics)`, `Empty(chapter)`.
- `uiState: StateFlow<ChapterUiState>` built by mapping `TopicRepository.getByChapterId(chapterId)` (Room `Flow`, `ORDER BY orderIndex ASC`) against a one-shot `ChapterRepository.getById(chapterId)` lookup, `stateIn`'d with `WhileSubscribed(5_000)`.
- No business logic — pure data shaping, same shape as `SubjectViewModel`.

### ChapterScreen (`ui/chapter/ChapterScreen.kt`)
- `Scaffold` with `ChapterTopBar` showing the chapter's `name` (from `Success`/`Empty` state; blank while `Loading`).
- Body switches on `uiState`: `CircularProgressIndicator` while loading, `EmptyState` when no topics, `LazyColumn` of `TopicRow` in `Success`.
- Topics rendered in `orderIndex` order as returned by the DAO query — no client-side re-sorting needed.
- Takes `onBackClick: () -> Unit` and `onTopicClick: (Long) -> Unit`, no `NavController` owned directly (same convention as `SubjectScreen`).

### New composables
- **ChapterTopBar** — title + back arrow, mirrors `SubjectTopBar` exactly.
- **TopicRow** — displays only `topic.name` and a trailing chevron (`Icons.AutoMirrored.Filled.KeyboardArrowRight`). No completion, progress, difficulty, study time, or stats — none of that exists in the schema, and the brief explicitly disallows fabricating it.
- **EmptyState** (`ui/chapter` package) — "No topics yet", same layout as the Subject Screen's `EmptyState` but scoped locally since Compose empty states aren't shared components in this codebase yet.

### Navigation (`navigation/NextTopicNavHost.kt`)
- `Screen.Chapter` route was already defined (M2 scaffolding) — no route contract changes.
- `ChapterScreen()` call updated to pass `onBackClick = { navController.popBackStack() }` and `onTopicClick = { topicId -> navController.navigate(Screen.Topic.createRoute(topicId)) }`.
- Topic Screen itself untouched — still `Text("Topic")` stub; only the navigation call site was wired.

## ViewModel → Repository → Room data flow

```
ChapterScreen
  → ChapterViewModel.uiState (StateFlow)
    → TopicRepository.getByChapterId(chapterId)   [Flow<List<TopicEntity>>]
        → TopicDao.getByChapterId (SELECT * FROM topics WHERE chapterId = :chapterId ORDER BY orderIndex ASC)
    → ChapterRepository.getById(chapterId)          [one-shot suspend fun]
        → ChapterDao.getById (SELECT * FROM chapters WHERE id = :id)
  → mapped into ChapterUiState.{Loading|Empty|Success}
```

Repository layer is a thin pass-through (unchanged from M2) — no new queries added to either DAO.

## Files changed
```
ui/chapter/ChapterViewModel.kt   — real implementation (was empty stub)
ui/chapter/ChapterScreen.kt      — real implementation (was Text("Chapter") stub)
ui/chapter/ChapterTopBar.kt      — new
ui/chapter/TopicRow.kt           — new
ui/chapter/EmptyState.kt         — new
navigation/NextTopicNavHost.kt   — Chapter composable call now passes onBackClick / onTopicClick
```

## Design/mockup deviations
- Schema field is `name`, not `title` (entities from M2/M3) — used `chapter.name` / `topic.name` throughout; functionally identical to "title" as described in the brief.
- Topic rows show name + chevron only, per spec — no fabricated completion/progress/difficulty/stats.

## Known gaps (unchanged from M5, not in scope for M6)
- If `chapterId` is deleted mid-session, `uiState` sticks on `Loading` — no terminal/error state. Same gap flagged in M5 for `subjectId`.

## Manual steps
1. **Gradle sync/build** — not run in this sandbox (no network/JDK toolchain). Run `./gradlew build` locally before merging.
2. No new dependencies were added this milestone (reused `material-icons-core` and Compose BOM entries already added in M4/M5).

## Next step: Milestone 7 — Topic Screen
- `TopicScreen` / `TopicViewModel` are still stubs. Same pattern applies: `TopicViewModel` should read `topicId` from `SavedStateHandle`, load via `TopicRepository.getById(topicId)`.
- Schema currently has only `name`, `chapterId`, `orderIndex` on `TopicEntity` — no content/body/detail field. Confirm with the product plan what M7's Topic Screen actually needs to display; if it needs body content, a schema addition (new column or entity) will need scoping before implementation.
- Consider whether the M5/M6 stuck-on-Loading-if-deleted gap should finally be addressed as a small shared fix (e.g. a `NotFound` state) once three screens share the same pattern.
