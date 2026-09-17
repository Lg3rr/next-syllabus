# Next Topic — V2 Foundation Milestone Handoff

**Milestone:** Project Foundation (compile-ready scaffold, no features)  
**App:** Next Topic — NEET study tracker, offline-first  
**Package ID:** `com.nexttopic.app`  
**Min SDK:** 24 | **Target/Compile SDK:** 35  
**Build Tools / AGP:** 8.6.1  
**Kotlin:** 2.0.21  

---

## Architecture

Single-module MVVM, package-by-layer.

```
UI Layer (Compose screens)
    ↓
ViewModel Layer (HiltViewModel, StateFlow — empty for now)
    ↓
Repository Layer (interfaces only — no impls yet)
    ↓
Data Layer (Room AppDatabase — zero entities, skeleton only)
    ↓
DI Layer (Hilt, SingletonComponent)
```

No Use Case layer was scaffolded. Add one when business logic warrants it (V2.3 recommendation engine is the earliest candidate).

---

## Package Structure

```
com.nexttopic.app/
├── MainActivity.kt
├── NextTopicApp.kt              ← @HiltAndroidApp
├── data/                        ← empty, reserved for data models / JSON assets
├── database/
│   └── AppDatabase.kt           ← @Database(entities=[], version=1) — STUB
├── di/
│   └── DatabaseModule.kt        ← @Provides AppDatabase (singleton)
├── domain/                      ← empty, reserved for use cases
├── navigation/
│   ├── Screen.kt                ← sealed class with typed routes
│   └── NextTopicNavHost.kt      ← NavHost wiring
├── repository/
│   ├── SubjectRepository.kt     ← empty interface
│   └── TopicRepository.kt       ← empty interface
├── ui/
│   ├── chapter/
│   │   ├── ChapterScreen.kt
│   │   └── ChapterViewModel.kt
│   ├── components/              ← empty, reserved for shared Composables
│   ├── home/
│   │   ├── HomeScreen.kt
│   │   └── HomeViewModel.kt
│   ├── settings/
│   │   ├── SettingsScreen.kt
│   │   └── SettingsViewModel.kt
│   ├── subject/
│   │   ├── SubjectScreen.kt
│   │   └── SubjectViewModel.kt
│   ├── theme/
│   │   ├── Color.kt
│   │   ├── SubjectColors.kt     ← CompositionLocal for product subject colors
│   │   ├── Theme.kt             ← NextTopicTheme (light + dark)
│   │   └── Type.kt
│   └── topic/
│       ├── TopicScreen.kt
│       └── TopicViewModel.kt
└── util/                        ← empty, reserved
```

---

## Dependencies & Versions

Managed via `gradle/libs.versions.toml` (version catalog).

| Library | Version |
|---|---|
| AGP | 8.6.1 |
| Kotlin | 2.0.21 |
| KSP | 2.0.21-1.0.28 |
| Compose BOM | 2024.12.01 |
| Navigation Compose | 2.8.5 |
| Hilt | 2.53 |
| Hilt Navigation Compose | 1.2.0 |
| Room | 2.6.1 |
| kotlinx-serialization-json | 1.7.3 |
| kotlinx-coroutines-android | 1.9.0 |
| core-ktx | 1.15.0 |
| lifecycle (runtime + viewmodel-compose) | 2.8.7 |
| activity-compose | 1.9.3 |

**Annotation processors:** KSP only (no kapt). Hilt and Room both use `ksp(...)`.

**Plugins applied in `app/build.gradle.kts`:**
- `com.android.application`
- `org.jetbrains.kotlin.android`
- `org.jetbrains.kotlin.plugin.compose`
- `org.jetbrains.kotlin.plugin.serialization`
- `com.google.dagger.hilt.android`
- `com.google.devtools.ksp`
- `androidx.room` (for `schemaDirectory`)

---

## Navigation Graph

```
NavHost(startDestination = Home)
├── Home          route: "home"
├── Subject       route: "subject/{subjectId}"    arg: subjectId: Long
├── Chapter       route: "chapter/{chapterId}"    arg: chapterId: Long
├── Topic         route: "topic/{topicId}"        arg: topicId: Long
└── Settings      route: "settings"
```

Route constants and `createRoute(id)` helpers are in `Screen.kt` (sealed class `data object`s).

`SubjectViewModel`, `ChapterViewModel`, and `TopicViewModel` each read their nav arg from `SavedStateHandle` on construction.

---

## Theme & Colors

### Subject colors (from product plan)

| Subject | Hex | Kotlin constant |
|---|---|---|
| Physics | `#3B82F6` | `PhysicsBlue` |
| Chemistry | `#22C55E` | `ChemistryGreen` |
| Botany | `#A855F7` | `BotanyPurple` |
| Zoology | `#EC4899` | `ZoologyPink` |

### Priority chip colors

| Priority | Hex | Kotlin constant |
|---|---|---|
| Low | `#22C55E` | `PriorityLow` |
| Medium | `#F59E0B` | `PriorityMedium` |
| High | `#EF4444` | `PriorityHigh` |
| Skipped | `#9CA3AF` | `PrioritySkipped` |

### M3 scheme

Light/dark `ColorScheme`s are hand-authored in `Color.kt` (no dynamic color). Primary seed = `#3B82F6` (Physics Blue).

Subject and priority colors are exposed via `LocalSubjectColors` (`staticCompositionLocalOf`) — **not** part of `MaterialTheme.colorScheme`. Access with:

```kotlin
val subjectColors = LocalSubjectColors.current
subjectColors.physics   // Color
```

`NextTopicTheme` wraps `CompositionLocalProvider(LocalSubjectColors provides SubjectColors())` around `MaterialTheme`. Dark mode reads `isSystemInDarkTheme()` automatically.

---

## Hilt Setup

**Application class:** `NextTopicApp : Application()` annotated `@HiltAndroidApp`.  
**Entry points:** `MainActivity` annotated `@AndroidEntryPoint`.  
**All ViewModels:** `@HiltViewModel` with `@Inject constructor(...)`.

### Modules

| Module | Location | Provides |
|---|---|---|
| `DatabaseModule` | `di/DatabaseModule.kt` | `AppDatabase` (singleton, `@ApplicationContext`) |

No repository bindings yet — interfaces have no implementations. Add a `RepositoryModule` when Room entities and DAOs exist.

---

## ViewModel Structure

All ViewModels are empty shells. Pattern to follow when adding state:

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val subjectRepository: SubjectRepository   // inject when bound
) : ViewModel() {
    private val _uiState = MutableStateFlow(HomeUiState())
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()
}
```

ViewModels with nav args (`Subject`, `Chapter`, `Topic`) already inject `SavedStateHandle` and read their ID:

```kotlin
private val subjectId: Long = savedStateHandle.get<Long>("subjectId") ?: -1L
```

---

## Gradle Configuration

**`settings.gradle.kts`** — standard, `rootProject.name = "NextTopic"`, single `:app` module.  
**`gradle.properties`** — `nonTransitiveRClass=true`, Xmx 2048m.  
**`app/build.gradle.kts`** — `compileOptions` and `kotlinOptions` set to Java 17.  
**`room { schemaDirectory("$projectDir/schemas") }`** — schema exports enabled; `/schemas/` will be written on first build. Commit this directory to version control.  
**`gradle/wrapper/gradle-wrapper.properties`** — Gradle 8.9-bin.

> **`gradlew` script and `gradle-wrapper.jar` are not included.** Android Studio generates both automatically on first project open. Alternatively run `gradle wrapper --gradle-version 8.9` from the project root if Gradle is installed locally.

---

## Assumptions Made

1. **No `ic_launcher` drawable** is included — Android Studio's default launcher icon will be used. Replace before release.
2. **`@Database(entities = [])` is intentional** — a necessary build stub. Room requires a concrete `@Database` class for Hilt to provide. This will be replaced with real entities in the next milestone.
3. **No repository implementations** — Hilt's `DatabaseModule` provides `AppDatabase` but no DAOs or repository impls are bound. `SubjectRepository` and `TopicRepository` interfaces exist but have no `@Binds` module yet.
4. **`domain/` package is empty** — no use cases scaffolded. Add when first real business logic is needed.
5. **Kotlin Serialization** is a declared dependency but unused. It will be needed for loading the NEET syllabus from a bundled JSON asset in the next milestone.
6. **XML bridge theme** (`themes.xml`) uses `android:Theme.Material.Light.NoActionBar` — this only governs the window before Compose renders. Not used for any Compose styling.

---

## Files Created (22 Kotlin + config)

```
settings.gradle.kts
build.gradle.kts
gradle.properties
gradle/libs.versions.toml
gradle/wrapper/gradle-wrapper.properties
app/build.gradle.kts
app/src/main/AndroidManifest.xml
app/src/main/res/values/strings.xml
app/src/main/res/values/themes.xml
app/src/main/java/com/nexttopic/app/NextTopicApp.kt
app/src/main/java/com/nexttopic/app/MainActivity.kt
app/src/main/java/com/nexttopic/app/database/AppDatabase.kt
app/src/main/java/com/nexttopic/app/di/DatabaseModule.kt
app/src/main/java/com/nexttopic/app/navigation/Screen.kt
app/src/main/java/com/nexttopic/app/navigation/NextTopicNavHost.kt
app/src/main/java/com/nexttopic/app/repository/SubjectRepository.kt
app/src/main/java/com/nexttopic/app/repository/TopicRepository.kt
app/src/main/java/com/nexttopic/app/ui/theme/Color.kt
app/src/main/java/com/nexttopic/app/ui/theme/SubjectColors.kt
app/src/main/java/com/nexttopic/app/ui/theme/Theme.kt
app/src/main/java/com/nexttopic/app/ui/theme/Type.kt
app/src/main/java/com/nexttopic/app/ui/home/HomeScreen.kt
app/src/main/java/com/nexttopic/app/ui/home/HomeViewModel.kt
app/src/main/java/com/nexttopic/app/ui/subject/SubjectScreen.kt
app/src/main/java/com/nexttopic/app/ui/subject/SubjectViewModel.kt
app/src/main/java/com/nexttopic/app/ui/chapter/ChapterScreen.kt
app/src/main/java/com/nexttopic/app/ui/chapter/ChapterViewModel.kt
app/src/main/java/com/nexttopic/app/ui/topic/TopicScreen.kt
app/src/main/java/com/nexttopic/app/ui/topic/TopicViewModel.kt
app/src/main/java/com/nexttopic/app/ui/settings/SettingsScreen.kt
app/src/main/java/com/nexttopic/app/ui/settings/SettingsViewModel.kt
```

---

## Known Limitations

- `AppDatabase(entities = [])` will generate a warning from Room ("No entities found") — expected, not an error.
- No `gradlew` / `gradle-wrapper.jar` — Android Studio generates these on first sync.
- No launcher icon resource — uses AS default; will show a blank/robot icon.
- Build not verified in CI — no Android SDK or Gradle available in the generation environment. First build must be run in Android Studio or a machine with the Android SDK installed.

---

## What the Next Milestone Builds On

**Next milestone: V2 — Syllabus Tracker (data layer + UI)**

The foundation provides these integration points:

### Room
Replace `AppDatabase(entities = [])` with real entities:
```kotlin
@Database(
    entities = [SubjectEntity::class, ChapterEntity::class, TopicEntity::class],
    version = 1,
    exportSchema = true
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun subjectDao(): SubjectDao
    abstract fun chapterDao(): ChapterDao
    abstract fun topicDao(): TopicDao
}
```

Add DAOs. Add `@Binds` implementations for `SubjectRepository` and `TopicRepository` in a new `RepositoryModule`.

### Entity fields to implement (per product plan)
- `SubjectEntity`: `id`, `name`, `colorHex`, `orderIndex`
- `ChapterEntity`: `id`, `subjectId`, `name`, `orderIndex`
- `TopicEntity`: `id`, `chapterId`, `name`, `isCompleted`, `priority` (enum: LOW/MEDIUM/HIGH/SKIPPED/NONE), `orderIndex`, `difficulty`, `focus`, `retention` (nullable Ints, used in V2.1)

### JSON asset loader
Bundled NEET syllabus JSON goes in `app/src/main/assets/syllabus.json`. Parse on first launch using `kotlinx-serialization-json` (already declared). Seed Room DB once, gate with a `DataStore<Preferences>` flag.

### Navigation
`HomeScreen` → `SubjectScreen(subjectId)` → `ChapterScreen(chapterId)` → `TopicScreen(topicId)`. Nav calls go through the NavController passed down or via a shared `NavigationViewModel`.

### ViewModels
Inject repository impls into each ViewModel. Use `viewModelScope.launch` + `StateFlow` for reactive UI. Pattern:
```kotlin
val subjects: StateFlow<List<Subject>> = subjectRepository
    .getAllSubjects()
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
```

### Theme
`LocalSubjectColors.current` is already available everywhere inside `NextTopicTheme`. Use `subjectColors.physics` etc. for subject-tinted UI elements (progress rings, chips, headers).
