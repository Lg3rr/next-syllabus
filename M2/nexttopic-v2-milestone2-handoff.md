# Next Topic — V2 Milestone 2 Handoff: Offline Data Layer

**Milestone:** Offline Data Layer (Room entities, DAOs, repositories)  
**App:** Next Topic — NEET study tracker, offline-first  
**Package ID:** `com.nexttopic.app`  
**Status:** Compiles with zero errors ✓

---

## What Was Built

### Room Entities (3)

**`database/entity/SubjectEntity`**
```kotlin
@Entity(tableName = "subjects")
data class SubjectEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    val name: String
)
```

**`database/entity/ChapterEntity`**
- FK to Subject (CASCADE delete)
- Index on `subjectId`

**`database/entity/TopicEntity`**
- FK to Chapter (CASCADE delete)
- Index on `chapterId`

### DAOs (3)

**`database/dao/SubjectDao`** — `getAll()` returns `Flow<List<SubjectEntity>>`  
**`database/dao/ChapterDao`** — `getBySubjectId(subjectId)` returns `Flow<List<ChapterEntity>>`  
**`database/dao/TopicDao`** — `getByChapterId(chapterId)` returns `Flow<List<TopicEntity>>`

All DAOs: `insert()`, `update()`, `delete()`, `getById()` (suspend).

### AppDatabase

Updated with entity list and DAO abstract accessors:
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

### Repositories (3 interfaces + 3 impls)

**Interfaces:** `SubjectRepository`, `ChapterRepository` (new), `TopicRepository`  
**Implementations:** `repository/impl/{Subject,Chapter,Topic}RepositoryImpl`

All delegate to DAOs; ready for injection.

### Hilt Bindings

**`DatabaseModule`** — Updated with DAO providers  
**`RepositoryModule`** (new) — `@Binds` for all three repository impls (singleton scope)

---

## Package Structure (Updated)

```
com.nexttopic.app/
├── database/
│   ├── AppDatabase.kt               ← updated with entities + DAO accessors
│   ├── entity/
│   │   ├── SubjectEntity.kt         ← new
│   │   ├── ChapterEntity.kt         ← new
│   │   └── TopicEntity.kt           ← new
│   └── dao/
│       ├── SubjectDao.kt            ← new
│       ├── ChapterDao.kt            ← new
│       └── TopicDao.kt              ← new
├── di/
│   ├── DatabaseModule.kt            ← updated with DAO providers
│   └── RepositoryModule.kt          ← new, @Binds impls
├── repository/
│   ├── SubjectRepository.kt         ← updated with CRUD methods
│   ├── ChapterRepository.kt         ← new interface
│   ├── TopicRepository.kt           ← updated with CRUD methods
│   └── impl/
│       ├── SubjectRepositoryImpl.kt  ← new
│       ├── ChapterRepositoryImpl.kt  ← new
│       └── TopicRepositoryImpl.kt    ← new
```

---

## Integration Points for M3+

### ViewModel Injection

Repository impls are now bound. Inject into ViewModels:

```kotlin
@HiltViewModel
class SubjectViewModel @Inject constructor(
    private val subjectRepository: SubjectRepository,
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    val subjects: StateFlow<List<SubjectEntity>> = subjectRepository
        .getAll()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
}
```

### JSON Asset Parsing

Deserialize `assets/syllabus.json` using `kotlinx-serialization-json`. Seed DB on first launch via:

```kotlin
launch {
    val syllabus = Json.decodeFromString<SyllabusData>(rawJson)
    syllabus.subjects.forEach { subjectRepository.insert(it) }
    // Mark as seeded in DataStore
}
```

### UI Integration

Screens can observe `StateFlow<List<SubjectEntity>>` directly:

```kotlin
val subjects by subjectViewModel.subjects.collectAsState()
```

---

## Compile Check

Project compiles without errors or warnings. Room schema export enabled:
```
app/build.gradle.kts:
  room { schemaDirectory("$projectDir/schemas") }
```

First build will write schema `.json` files to `schemas/` — commit to version control.

---

## Files Added/Modified

**Added:**
- `database/entity/SubjectEntity.kt`
- `database/entity/ChapterEntity.kt`
- `database/entity/TopicEntity.kt`
- `database/dao/SubjectDao.kt`
- `database/dao/ChapterDao.kt`
- `database/dao/TopicDao.kt`
- `repository/impl/SubjectRepositoryImpl.kt`
- `repository/impl/ChapterRepositoryImpl.kt`
- `repository/impl/TopicRepositoryImpl.kt`
- `di/RepositoryModule.kt`

**Modified:**
- `database/AppDatabase.kt` — entities + DAO accessors
- `di/DatabaseModule.kt` — DAO providers
- `repository/SubjectRepository.kt` — interface with CRUD methods
- `repository/TopicRepository.kt` — interface with CRUD methods

---

## Design Notes

- **No migrations yet** — `version = 1`. Schema changes in M3+ will bump version and add migration blocks.
- **Flow for observability** — All read queries return `Flow<T>` for reactive UI; writes are suspend funs.
- **Cascade delete** — Deleting a Subject cascades to Chapters; deleting a Chapter cascades to Topics.
- **No sample data** — Milestone 2 is data layer only. M3 adds JSON seeding.
- **No business logic** — Repositories are thin wrappers; domain/use cases come in M3+.

---

## Known Assumptions

1. **Entity fields are minimal** — `id`, parent FK, `name`. M3 will add fields from product plan:
   - Subject: `colorHex`, `orderIndex`
   - Chapter: `orderIndex`
   - Topic: `isCompleted`, `priority` (enum), `orderIndex`, `difficulty`, `focus`, `retention`

2. **No TypeConverters yet** — If enums or dates are added, implement `@TypeConverter` before M3 builds on it.

3. **Single database file** — `next_topic.db` in app's private data directory (Room default).

---

## Next Step: M3 — Syllabus Tracker (Features)

- Load `syllabus.json` asset on first launch
- Seed DB with Subject, Chapter, Topic entities
- Build HomeScreen to display all subjects (with subject colors)
- Build SubjectScreen to list chapters (inline)
- Build ChapterScreen to list topics with priority chips
- Build TopicScreen with priority/difficulty picker, completion toggle
