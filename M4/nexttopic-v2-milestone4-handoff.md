# Next Topic — V2 Milestone 4 Handoff: Home Screen

**Package ID:** `com.nexttopic.app`
**Status:** Implemented against real Room data. Not compiled in this environment (no network/Gradle available in sandbox) — see Manual Steps.

---

## Approach

`HomeScreen` and `HomeViewModel` existed as stubs from M2/M3 scaffolding; both were filled in rather than restructured.

- **HomeViewModel** injects `SubjectRepository` (already bound in `RepositoryModule`) and maps its
  `Flow<List<SubjectEntity>>` to a `HomeUiState` (`Loading` / `Empty` / `Success`) via `stateIn`,
  scoped to `viewModelScope` with `WhileSubscribed(5_000)`. No transformation of the entity data —
  `orderIndex` ordering and `colorHex` already come sorted/populated from the DAO (`ORDER BY
  orderIndex ASC`, set in M3), so the ViewModel does no business logic, just Flow → StateFlow.
- **HomeScreen** collects `uiState` and renders one of three states. On `Success`, subjects render
  in a `LazyColumn` of `SubjectCard`, in the order the repository returned them.
- **SubjectCard**, **HomeTopBar**, **EmptyState** extracted as separate composables per the milestone spec.

## Card content vs. mockup

The mockup shows per-subject progress rings and chapter counts. That data doesn't exist in the
schema (no completion/progress columns) and the brief explicitly says not to fabricate it, so each
card shows only what M2/M3 persisted: **name**, **colorHex** (as a small color dot), and a chevron.
Layout, dark surface, spacing, and corner radius follow the mockup; the progress indicator does not.
Same reasoning applies to the mockup's "Good morning, Rohan!" greeting and avatar initial — there's
no user/profile entity, so the top bar shows just the app name and a search action (search itself
is a stub — `onSearchClick = {}` — since search wasn't in scope for M4).

## Navigation

`HomeScreen` now takes `onSubjectClick: (Long) -> Unit` instead of owning a `NavController`.
`NextTopicNavHost` wires it to `navController.navigate(Screen.Subject.createRoute(subjectId))` —
the existing route contract from M2 was reused unchanged.

## Files changed

```
ui/home/HomeViewModel.kt      — real implementation (was empty stub)
ui/home/HomeScreen.kt         — real implementation (was empty stub)
ui/home/HomeTopBar.kt         — new
ui/home/SubjectCard.kt        — new
ui/home/EmptyState.kt         — new
navigation/NextTopicNavHost.kt — Home composable call passes onSubjectClick
MainActivity.kt               — added enableEdgeToEdge()
gradle/libs.versions.toml     — added material-icons-core coordinate
app/build.gradle.kts          — added material-icons-core dependency
```

## Manual Steps

1. **Gradle sync required** — `material-icons-core` was added as an explicit dependency (Compose
   BOM 2024.12+ no longer bundles it transitively via material3; `Icons.Filled.Search` /
   `Icons.AutoMirrored.Filled.KeyboardArrowRight` need it directly). No version pinned — resolved
   via the existing `androidx-compose-bom` platform entry.
2. **Build and run** — could not execute `./gradlew build` in this sandbox (no network/JDK
   toolchain access). Please build locally before merging; flag back if anything doesn't compile.

---

## Next Step: M5 — Subject Screen

- `SubjectScreen` / `SubjectViewModel` are still stubs. `SubjectViewModel` already reads
  `subjectId` from `SavedStateHandle` (from M2 scaffolding) — needs `ChapterRepository.getBySubjectId`
  wiring (check `ChapterRepository`'s existing query shape first).
- Same "don't fabricate" constraint will apply: chapters have `orderIndex` but no progress/
  completion column yet — confirm with product plan whether M5 needs it before adding schema.
- Revisit the M3-flagged `externalId` gap if Subject Screen needs anything keyed by the original
  JSON ids.
