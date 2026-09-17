# Milestone 7 Handoff — Topic Details Screen

**Status:** Complete. Same pattern as M5/M6, extended one level deeper (Topic → Chapter → Subject). Not run through Gradle in this sandbox (no network/JDK toolchain) — verify locally.

## Implementation approach
`TopicScreen` needed data from three tables, but only one-shot `getById` lookups exist for `Chapter`/`Subject` — there's no `Flow` to observe a single topic. So `TopicViewModel.uiState` is built with `flow { ... }.stateIn(...)` instead of `Repository.flow.map { }` (the M5/M6 shape): it emits `Loading` initially, then does three sequential one-shot repository calls and emits `Success` or `Empty`. Still a single `StateFlow`, still zero business logic — just data shaping, same as prior milestones.

## ViewModel → Repository → Room data flow
```
TopicScreen
  → TopicViewModel.uiState (StateFlow)
    → TopicRepository.getById(topicId)        [one-shot] → TopicDao.getById
    → ChapterRepository.getById(chapterId)     [one-shot] → ChapterDao.getById
    → SubjectRepository.getById(subjectId)     [one-shot] → SubjectDao.getById
  → mapped into TopicUiState.{Loading|Empty|Success(topic, chapterName, subjectName)}
```
No new DAO queries — all three `getById` methods already existed.

## Files changed
```
ui/topic/TopicViewModel.kt   — real implementation (was empty stub)
ui/topic/TopicScreen.kt      — real implementation (was Text("Topic") stub)
ui/topic/TopicTopBar.kt      — new, mirrors ChapterTopBar/SubjectTopBar exactly
ui/topic/TopicHeader.kt      — new: topic title + "Subject • Chapter" line
ui/topic/PlaceholderCard.kt  — new: reused for all 5 placeholder sections
navigation/NextTopicNavHost.kt — TopicScreen call now passes onBackClick
```

## Placeholder sections
Overview, Notes, PYQs, Revision, Study Session — each rendered via the shared non-interactive `PlaceholderCard(icon, label)`, all showing a "Coming Soon" label. No `onClick`, no navigation, no fabricated content.

## Design/mockup deviations
- No mockup image was supplied with this milestone (only the M6 handoff and prior code). Layout follows the established M5/M6 visual language: `Scaffold` + `TopAppBar` + `LazyColumn`, Material 3 defaults, same card/spacing conventions as `ChapterCard`.
- Icons use only `material-icons-core` (Info, Edit, List, Refresh, PlayArrow) — the extended icon set isn't a project dependency, same constraint noted in M4.

## Known gaps (unchanged from M5/M6)
- If `topicId`/`chapterId`/`subjectId` is deleted mid-session, `uiState` returns `Empty`/"Topic not found" rather than a distinct terminal error state — same class of gap as M5/M6, now surfaced explicitly via the `Empty` branch instead of sticking on `Loading`.

## Manual steps
1. Run `./gradlew build` locally — not executed in this sandbox.
2. No new dependencies added.
3. If an actual approved mockup exists for this screen, compare visually and adjust spacing/typography in `TopicHeader`/`PlaceholderCard` — none was provided for this milestone.

## Next step: Milestone 8
- All four core screens (Home, Subject, Chapter, Topic) are now real. Consider addressing the shared "entity deleted mid-session" gap with a common `NotFound` state pattern across ViewModels.
- Placeholder sections (Overview, Notes, PYQs, Revision, Study Session) are stubs only — each is a future milestone's schema + feature work.
