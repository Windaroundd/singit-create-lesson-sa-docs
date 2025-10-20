# Lesson Creation Wizard - System Architecture Documentation

## 1. Introduction and Goals

### Business Context

Singit is upgrading the existing lesson creation system with a new wizard-based interface. This redesign breaks down the process into guided steps with AI assistance, making it easier for teachers to create high-quality lessons efficiently.

### Quality Goals (Priority Order)

1. **Backward Compatibility** - Existing lessons must continue working without modification
2. **Data Integrity** - No data loss during draft creation/editing
3. **Performance** - Lesson creation completes within acceptable time
4. **Usability** - Intuitive step-by-step flow with clear progress indication
5. **Scalability** - Support for future enhancements (e.g., more lesson types)

### Stakeholders

- **Teachers** - Primary users creating lessons
- **Students** - End users consuming lessons (no impact)
- **Administrators** - Managing lesson library
- **Development Team** - Implementing and maintaining

---

## 2. Architecture Constraints

### Technical Constraints

- Node.js/Express backend (existing)
- MongoDB database (existing)
- Must support existing REST API consumers
- Authentication via JWT tokens
- Frontend uses React 

### Organizational Constraints

- No breaking changes to existing API endpoints
- Must maintain current database performance
- Deploy without downtime

---

## 3. System Scope and Context

### Business Context

```
┌─────────────┐
│   Teacher   │
│  (Frontend) │
└──────┬──────┘
       │ REST API
       ↓
┌─────────────────────────────────────┐
│     Lesson Creation System          │
│  ┌──────────────────────────────┐   │
│  │ Wizard Flow Controller       │   │
│  └──────────────────────────────┘   │
│  ┌──────────────────────────────┐   │
│  │ Draft Management Service     │   │
│  └──────────────────────────────┘   │
│  ┌──────────────────────────────┐   │
│  │ Lesson Service (Enhanced)    │   │
│  └──────────────────────────────┘   │
└────────┬─────────────┬──────────────┘
         │             │
         ↓             ↓
┌─────────────┐  ┌──────────────┐
│  MongoDB    │  │  AI Service  │
│  (Lessons,  │  │  (Quiz Gen)  │
│   Drafts)   │  │              │
└─────────────┘  └──────────────┘
```

### Impacted External Systems

#### Internal Services (Existing - Need Enhancement)

**1. Track Search Service**
- **Location**: `services/music/track.service.js`, `controllers/music/tracks.controller.js`
- **Current Usage**: Basic song search via `/tracks/search`
- **Required Changes**:
  - Add artist-based filtering logic
  - Add genre-based filtering logic
  - Enhance search to consider wizard preferences (proficiency level, theme)
  - Create `searchTracksWithFilters()` method for wizard-specific queries
- **Impact Level**: MEDIUM - Existing search works, need to add filters

**2. Select Service** 
- **Location**: `models/select.model.js`, `controllers/select.controller.js`, `services/select.service.js`
- **Current Usage**: Dynamic dropdown options via `/select/getSelects`
- **Required Changes**:
  - Add wizard-specific select configurations:
    - English skills options
    - Proficiency levels (A1-A2, B1-B2, C1-C2)
    - Lesson themes
    - Song genres
    - Lesson formats
    - Submission types
    - Lesson durations
  - May need to create new Select documents in database
- **Impact Level**: LOW - Just configuration additions

**3. User Service**
- **Location**: `services/users.service.js`, `controllers/users.controller.js`
- **Current Usage**: User search via `/users/filter`, `/users/myStudentsFilter`, `/users/populateUsers`
- **Required Changes**:
  - Enhance `searchUsers()` for teacher collaboration feature
  - Add filtering by teacher role
  - Support searching for collaborators
- **Impact Level**: LOW - Existing search APIs work, minor enhancements only

**4. Artist & Genre Collections**
- **Location**: `models/music/artist.model.js`, `models/genre.model.js`
- **Current Usage**: Referenced in tracks, used in onboarding
- **Required Changes**:
  - Add `getPopularArtists()` endpoint for wizard step 2
  - May need to add `sortWeight` or popularity metrics if not exists
  - Filter artists by relevance for educational content
- **Impact Level**: LOW - Mostly read-only operations

#### AI Services (Existing - Need New Features)

**5. OpenAI Integration**
- **Location**: `thirdParty/openAI.js`, `services/aiChat.service.js`
- **Current Usage**: AI chat, translation, speaking feedback
- **Required Changes**:
  - Add `extractVocabularyFromLyrics()` - analyze lyrics and suggest key vocabulary
  - Add `generateQuizQuestions()` - create comprehension questions based on lesson content
  - Both functions use existing OpenAI infrastructure
- **Impact Level**: MEDIUM - New AI prompts, need to design and test

**6. Google AI Integration**
- **Location**: `thirdParty/googleAI.js`
- **Current Usage**: Alternative AI provider for various features
- **Required Changes**:
  - Fallback for vocabulary extraction if OpenAI fails
  - Fallback for quiz generation if OpenAI fails
- **Impact Level**: LOW - Optional fallback implementation

#### Background Processing (EXISTING - Need Enhancement)

**7. Job Queue System**
- **Current State**: Agenda.js with MongoDB backend already exists in `worker.js`
- **Current Usage**: Handles `send-streak-notification`, `sqs-track-pipeline-consumer`, `run-bigquery-etl`, etc.
- **Required Changes**:
  - Add `jobs/generateAIQuiz.job.js` for async quiz generation
  - Add `jobs/cleanupOldWizardDrafts.job.js` for wizard draft maintenance
  - Use existing Agenda.js infrastructure
- **Impact Level**: LOW - Leverage existing infrastructure
- **Implementation**: Extend current job system with new job definitions

#### Event Tracking & Analytics

**8. Customer.io Integration**
- **Location**: Referenced in `controllers/lesson.controller.js` as `cio.createEvent()`
- **Current Usage**: Tracks lesson_create, lesson_update events
- **Required Changes**:
  - Add wizard-specific events:
    - `wizard_step_completed` (with step number)
    - `wizard_lesson_generated` (from draft)
    - `ai_quiz_generation_completed`
    - `vocabulary_extraction_completed`
  - Track wizard completion rate and drop-off points
- **Impact Level**: LOW - Just additional event tracking calls

#### Notification System (OPTIONAL - May Need)

**9. Real-time Notifications**
- **Current State**: Unclear if WebSocket/SSE infrastructure exists
- **Required For**: Notify teacher when AI quiz generation completes
- **Options**:
  - A) Simple polling from frontend (check `/lesson/:id/aiStatus`)
  - B) WebSocket/SSE for real-time updates (if infrastructure exists)
  - C) Email notification (use existing email system if available)
- **Impact Level**: VARIES (LOW for polling, HIGH for WebSocket)
- **Recommendation**: Start with polling for MVP


#### Dependencies Summary

| System | Type | Impact Level | Action Required |
|--------|------|--------------|-----------------|
| Track Search | Enhancement | MEDIUM | Add filtering methods |
| Select Service | Configuration | LOW | Add new select options |
| User Service | Enhancement | LOW | Minor search enhancements |
| Artist/Genre | Enhancement | LOW | Add popularity endpoint |
| OpenAI | New Features | MEDIUM | Add 2 new AI functions |
| Google AI | Fallback | LOW | Optional fallback logic |
| Job Queue | Enhancement | LOW | Extend existing Agenda.js infrastructure |
| Customer.io | Events | LOW | Add event tracking calls |
| Notifications | Optional | VARIES | Choose polling vs WebSocket |

---

## 4. Solution Strategy

### Core Architectural Decisions

#### 4.1 Version-Based Backward Compatibility

**Decision**: Add `version` field to Lesson model (default: 1, new workflow: 2)

**Rationale**:

- Minimal database changes
- Clear distinction between old/new lessons
- Easy to add version-specific business logic
- Supports gradual migration if needed

**Implementation**:

```javascript
version: { type: Number, default: 1, required: true }
// version 1 = legacy form creation
// version 2 = new wizard workflow
```

#### 4.2 Hybrid Draft System Architecture

**Decision**: Create separate `LessonWizardDraft` model while keeping existing `LessonDraft` unchanged

**Rationale**:

- Zero breaking changes to legacy draft system
- Clean separation of concerns between legacy and wizard workflows
- Independent scaling and optimization
- Easier maintenance and debugging

**Implementation**:

- Keep existing `LessonDraft` model completely unchanged
- Create new `LessonWizardDraft` model with wizard-specific fields
- Separate controllers, routes, and services for wizard workflow


#### 4.4 Hybrid Vocabulary Selection

**Decision**: AI suggests vocabulary, teacher can modify

**Rationale**:

- Saves teacher time with intelligent suggestions
- Maintains teacher control for educational quality
- Leverages existing lyrics analysis capabilities

---

## 5. Building Block View

### Level 1: System Context

```
┌────────────────────────────────────────────────┐
│           Lesson Creation Module               │
├────────────────────────────────────────────────┤
│  Controllers:                                  │
│    - lessonDraft.controller (UNCHANGED)        │
│    - lesson.controller (EXTENDED)              │
│    - lessonWizardDraft.controller (NEW)        │
│                                                │
│  Services:                                     │
│    - lesson.service (EXTENDED)                 │
│    - vocabularyExtraction.service (NEW)        │
│    - quizGeneration.service (NEW)              │
│                                                │
│  Models:                                       │
│    - lesson.model (EXTENDED)                   │
│    - lessonDraft.model (UNCHANGED)             │
│    - lessonWizardDraft.model (NEW)             │
└────────────────────────────────────────────────┘
```

### Level 2: Component Details

#### New/Modified Components

**1. lessonWizardDraft.controller.js** (NEW)

- Handles wizard-specific operations
- Step validation logic
- Progress tracking
- Draft to lesson transformation

**2. vocabularyExtraction.service.js** (NEW)

- Analyzes song lyrics
- Extracts key vocabulary using AI
- Returns suggested words with context

**3. quizGeneration.service.js** (NEW)

- Generates quiz questions based on lesson content
- Uses OpenAI/Google AI APIs
- Processes asynchronously via existing Agenda.js job queue

**4. lessonWizardDraft.model.js** (NEW)

- Separate model for wizard workflow
- Contains `wizardData` object with nested structure:
  - `personalInfo`: schoolName, gradeLevel, quarter, englishSkills
  - `lessonPreferences`: selectedTrack, proficiencyLevel, lessonTheme, songGenre, favoriteArtists, selectedVocabulary, includeInteractiveExercises, enableAIQuizGeneration, lessonFormat
  - `studentExperience`: submissionType, enableCollaboration, lessonDuration
  - `finalization`: shareWithTeachers, enableStudentFeedback, saveAsTemplate, collaborators
- `currentStep` for progress tracking (Number, default: 1)
- `completedSteps` array for tracking finished steps
- `aiGenerationStatus` object for async AI operations status

**5. lesson.model.js** (EXTENDED)

- Add `version` field (Number, default: 1, required: true)
- Add `wizardMetadata` object containing:
  - `schoolName`, `gradeLevel`, `quarter`, `englishSkills` (from personalInfo)
  - `proficiencyLevel` (A1-A2, B1-B2, C1-C2) - separate from existing difficulty field
  - `lessonTheme`, `songGenre` (ObjectId ref to Genre), `favoriteArtists`, `selectedVocabulary`
  - `includeInteractiveExercises`, `enableAIQuizGeneration`, `lessonFormat`
  - `submissionType`, `enableCollaboration`, `lessonDuration` (15-20, 30-40, 60 minutes)
  - `shareWithTeachers`, `enableStudentFeedback`, `saveAsTemplate`
- Note: `collaborators` field already exists in base model (line 56)
- Note: `duration` field already exists - `lessonDuration` is for wizard preferences

### Detailed Model Structure

#### lessonWizardDraft.model.js Structure:
```javascript
// NEW MODEL - Separate from existing LessonDraft
const LessonWizardDraftSchema = new Schema({
  // Basic fields
  title: { type: String, required: true },
  createdBy: { type: ObjectId, ref: 'User', required: true },
  
  // Wizard-specific fields
  currentStep: { type: Number, default: 1, min: 1, max: 4 },
  completedSteps: [{ type: Number }],
  
  // Wizard data structure
  wizardData: {
    personalInfo: {
      schoolName: { type: String },
      gradeLevel: { type: String },
      quarter: { type: String },
      englishSkills: [{ type: String }]
    },
    lessonPreferences: {
      selectedTrack: { type: ObjectId, ref: 'Track' },
      proficiencyLevel: { type: String, enum: ['A1-A2', 'B1-B2', 'C1-C2'] },
      lessonTheme: { type: String },
      songGenre: { type: ObjectId, ref: 'Genre' },
      favoriteArtists: [{ type: String }],
      selectedVocabulary: [{ type: String }],
      includeInteractiveExercises: { type: Boolean, default: false },
      enableAIQuizGeneration: { type: Boolean, default: false },
      lessonFormat: { type: String }
    },
    studentExperience: {
      submissionType: { type: String },
      enableCollaboration: { type: Boolean, default: false },
      lessonDuration: { type: String, enum: ['15-20', '30-40', '60'] }
    },
    finalization: {
      shareWithTeachers: { type: Boolean, default: false },
      enableStudentFeedback: { type: Boolean, default: false },
      saveAsTemplate: { type: Boolean, default: false },
      collaborators: [{ type: ObjectId, ref: 'User' }]
    }
  },
  
  // AI generation status
  aiGenerationStatus: {
    quizQuestions: { type: String, enum: ['pending', 'processing', 'completed', 'failed'], default: 'pending' },
    vocabularyExtraction: { type: String, enum: ['pending', 'processing', 'completed', 'failed'], default: 'pending' }
  },
  
  // Reference to final lesson when created
  generatedLessonId: { type: ObjectId, ref: 'Lesson' }
}, {
  timestamps: { createdAt: 'created', updatedAt: 'lastModified' }
});
```

#### lesson.model.js Extensions:
```javascript
version: { type: Number, default: 1, required: true },
wizardMetadata: {
  // Personal Info
  schoolName: { type: String },
  gradeLevel: { type: String },
  quarter: { type: String },
  englishSkills: [{ type: String }],
  
  // Lesson Preferences
  proficiencyLevel: { type: String, enum: ['A1-A2', 'B1-B2', 'C1-C2'] },
  lessonTheme: { type: String },
  songGenre: { type: ObjectId, ref: 'Genre' },
  favoriteArtists: [{ type: String }],
  selectedVocabulary: [{ type: String }],
  includeInteractiveExercises: { type: Boolean, default: false },
  enableAIQuizGeneration: { type: Boolean, default: false },
  lessonFormat: { type: String },
  
  // Student Experience
  submissionType: { type: String },
  enableCollaboration: { type: Boolean, default: false },
  lessonDuration: { type: String, enum: ['15-20', '30-40', '60'] },
  
  // Finalization
  shareWithTeachers: { type: Boolean, default: false },
  enableStudentFeedback: { type: Boolean, default: false },
  saveAsTemplate: { type: Boolean, default: false }
}
// Note: collaborators field already exists in base model
```

---

## 6. Runtime View

### Scenario 1: Teacher Creates New Lesson via Wizard

```
Teacher          Frontend         API Gateway      Draft Service     Lesson Service    AI Service
   │                │                  │                │                 │               │
   │─── Start ─────>│                  │                │                 │               │
   │                │                  │                │                 │               │
   │                │── POST /lessonWizard/start ──────>│                 │               │
   │                │                  │                │─ Create Wizard  │               │
   │                │                  │                │  Draft          │               │
   │                │<───── draftId + step1 config ─────│                 │               │
   │                │                  │                │                 │               │
   │─ Fill Step1 ──>│                  │                │                 │               │
   │─ Click Next ──>│                  │                │                 │               │
   │                │                  │                │                 │               │
   │                │── POST /lessonWizard/:id/updateStep ───────────────>│               │
   │                │                  │                │─ Save wizardData│               │
   │                │<───── success + step2 config ─────│                 │               │
   │                │                  │                │                 │               │
   │                ├─────────────────────────────────────────────────────┤               │
   │                │  ... Repeat for Steps 2, 3, 4 ...                   │               │
   │                ├─────────────────────────────────────────────────────┤               │
   │                │                  │                │                 │               │
   │─ Generate ────>│                  │                │                 │               │
   │   Lesson       │                  │                │                 │               │
   │                │                  │                │                 │               │
   │                │── POST /lessonWizard/:id/generate ─────────────────>│               │
   │                │                  │                │                 │               │
   │                │                  │                │── Transform ───>│               │
   │                │                  │                │   wizardData    │               │
   │                │                  │                │                 │               │
   │                │                  │                │                 │─ Create v2    │
   │                │                  │                │                 │  Lesson       │
   │                │                  │                │                 │               │
   │                │                  │                │                 │─ Trigger ────>│
   │                │                  │                │                 │  AI Quiz Job  │
   │                │                  │                │                 │  (Agenda.js)  │
   │                │                  │                │                 │               │
   │                │<────────────────── lessonId ──────────────────────────              │
   │                │                  │                │                 │               │
   │                │                  │                │─ Delete Wizard  │               │
   │                │                  │                │  Draft          │               │
   │<─── Success ───│                  │                │                 │               │
   │                │                  │                │                 │               │
   │                │                  │                │                 │               │
   │                │                  │                │                 │<─ Questions ──│
   │                │                  │                │                 │  Generated    │
   │                │                  │                │                 │               │
   │<─ Notification─│<────────────────── Webhook───────────────────────────               │
   │   (Quiz Ready) │                  │                │                 │               │
```

### Scenario 2: Load Existing Draft

```
Teacher          Frontend         API Gateway      Draft Service
   │                │                  │                │
   │─ View Drafts ─>│                  │                │
   │                │                  │                │
   │                │── GET /lessonWizard/myDrafts ────>│
   │                │                  │                │
   │                │<────────────── wizardDrafts[] ────│
   │                │                  │                │
   │                │                  │                │
   │─ Edit Draft ──>│                  │                │
   │   (select)     │                  │                │
   │                │                  │                │
   │                │── GET /lessonWizard/:id/details ─>│
   │                │                  │                │
   │                │<──── wizardDraft + wizardData ────│
   │                │      (currentStep data)           │
   │                │                  │                │
   │<─ Show Step X ─│                  │                │
   │   (resume)     │                  │                │
```

---


## 7. Cross-cutting Concepts

### Data Transformation Logic

**Draft → Lesson Conversion**:

```javascript
// Transform wizard draft to version 2 lesson
function transformDraftToLesson(draft) {
  return {
    version: 2,
    
    // Core lesson fields (existing)
    title: draft.lesson.title || generateTitleFromPreferences(draft.wizardData),
    description: draft.lesson.description,
    duration: parseDuration(draft.wizardData.studentExperience.lessonDuration),
    ageGrade: parseGradeLevel(draft.wizardData.personalInfo.gradeLevel),
    difficulty: mapProficiencyToDifficulty(draft.wizardData.lessonPreferences.proficiencyLevel),
    trackId: draft.wizardData.lessonPreferences.selectedTrack,
    active: true,
    public: draft.wizardData.finalization.shareWithTeachers,
    
    // Wizard metadata
    wizardMetadata: {
      schoolName: draft.wizardData.personalInfo.schoolName,
      gradeLevel: draft.wizardData.personalInfo.gradeLevel,
      quarter: draft.wizardData.personalInfo.quarter,
      englishSkills: draft.wizardData.personalInfo.englishSkills,
      proficiencyLevel: draft.wizardData.lessonPreferences.proficiencyLevel,
      lessonTheme: draft.wizardData.lessonPreferences.lessonTheme,
      songGenre: draft.wizardData.lessonPreferences.songGenre,
      favoriteArtists: draft.wizardData.lessonPreferences.favoriteArtists,
      selectedVocabulary: draft.wizardData.lessonPreferences.selectedVocabulary,
      includeInteractiveExercises: draft.wizardData.lessonPreferences.includeInteractiveExercises,
      enableAIQuizGeneration: draft.wizardData.lessonPreferences.enableAIQuizGeneration,
      lessonFormat: draft.wizardData.lessonPreferences.lessonFormat,
      submissionType: draft.wizardData.studentExperience.submissionType,
      enableCollaboration: draft.wizardData.studentExperience.enableCollaboration,
      lessonDuration: draft.wizardData.studentExperience.lessonDuration,
      shareWithTeachers: draft.wizardData.finalization.shareWithTeachers,
      enableStudentFeedback: draft.wizardData.finalization.enableStudentFeedback,
      saveAsTemplate: draft.wizardData.finalization.saveAsTemplate
    },
    
    // Assignments (initially empty, populated by AI if enabled)
    preAssignments: [],
    whileAssignments: [],
    assignments: [],
    
    // Collaborators
    collaborators: draft.wizardData.finalization.collaborators || [],
    
    // Creator info
    createdBy: draft.createdBy,
    createdByName: draft.createdByName
  };
}
```

### Validation Rules

**Step 1 (Personal Information)**: Optional fields, all can be skipped

**Step 2 (Lesson Preferences)**:

- `selectedTrack` - REQUIRED
- `proficiencyLevel` - REQUIRED

**Step 3 (Student Experience)**:

- `submissionType` - REQUIRED
- `lessonDuration` - REQUIRED

**Step 4 (Finalization)**: All optional

---

## 8. Architecture Decisions

### ADR-001: Use Version Field for Backward Compatibility

**Status**: 

**Context**: Need to distinguish between old and new lesson creation workflows without breaking existing functionality.

**Decision**: Add `version` field to Lesson model (default: 1, wizard: 2).

**Consequences**:

- Simple to implement
- Clear version distinction
- Easy to query lessons by version
- Supports gradual migration
- Adds one field to every lesson document

**Alternatives Considered**:

- Separate collection → More complex, data duplication
- Workflow field → Less semantic, harder to extend

---

### ADR-002: Asynchronous AI Quiz Generation

**Status**: 

**Context**: AI quiz generation takes 5-15 seconds, blocking UI is poor UX.

**Decision**: Generate quiz questions asynchronously after lesson creation using background jobs.

**Consequences**:

- Better user experience (no waiting)
- Better error handling
- Lesson creation succeeds even if AI fails
- Requires job queue infrastructure
- More complex state management

**Implementation**:

- Use existing job infrastructure (if available) or implement simple queue
- Store generation status in lesson document
- Notify teacher when questions are ready

---

### ADR-003: Hybrid Draft System Architecture

**Status**: 
 
**Context**: Need to store wizard-specific data during lesson creation without breaking existing functionality.

**Decision**: Create separate `LessonWizardDraft` model while keeping existing `LessonDraft` completely unchanged.

**Consequences**:

- Zero breaking changes to legacy draft system
- Clean separation of concerns between workflows
- Independent scaling and optimization
- Easier maintenance and debugging
- Can rollback wizard feature independently
- Slightly more code to maintain

---



## Appendix A: API Endpoints Specification

### New Endpoints

#### Wizard Management

```
POST   /lessonWizard/start
GET    /lessonWizard/config
GET    /lessonWizard/myDrafts
GET    /lessonWizard/:id/details
POST   /lessonWizard/:id/updateStep
POST   /lessonWizard/:id/generate
DELETE /lessonWizard/:id
```

#### Vocabulary & AI

```
POST   /lessonWizard/extractVocabulary
GET    /lessonWizard/:id/aiStatus
```

#### Artists & Song Selection

```
GET    /artists/popular
POST   /tracks/searchWithFilters
```

### Modified Endpoints

```
POST   /lesson/create (add version support)
POST   /lesson/:id/update (version-aware)
GET    /lesson/:id/details (include wizardMetadata)
```

---

## Appendix B: Database Indexes

### Performance Indexes

```javascript
// lessons collection
db.lessons.createIndex({ version: 1 });
db.lessons.createIndex({ createdBy: 1, version: 1 });
db.lessons.createIndex({ "wizardMetadata.proficiencyLevel": 1 });
db.lessons.createIndex({ "wizardMetadata.songGenre": 1 });

// lesson-wizard-drafts collection
db['lesson-wizard-drafts'].createIndex({ createdBy: 1, lastModified: -1 });
db['lesson-wizard-drafts'].createIndex({ currentStep: 1 });
db['lesson-wizard-drafts'].createIndex({ "aiGenerationStatus.quizQuestions": 1 });
```

---

## Appendix C: Impacted Files Summary

### Files to CREATE (Estimated 12 new files)

**Models:**

- `models/lessonWizardDraft.model.js`

**Controllers:**

- `controllers/lessonWizardDraft.controller.js`

**Services:**

- `services/vocabularyExtraction.service.js`
- `services/quizGeneration.service.js`

**Routes:**

- `routes/lessonWizard.route.js`

**Jobs:**

- `jobs/generateAIQuiz.job.js`
- `jobs/cleanupOldWizardDrafts.job.js`

**Utilities:**

- `utility/wizardValidation.js`
- `utility/draftTransformer.js`


### Files to MODIFY (Estimated 6 existing files)

**Models:**

- `models/lesson.model.js` - Add version field, wizardMetadata object

**Controllers:**

- `controllers/lesson.controller.js` - Version-aware create/update

**Services:**

- `services/lesson.service.js` - Add version handling logic
- `services/select.service.js` - Add wizard-specific selects

**Routes:**

- `routesConfig.js` - Register wizard routes

**Configuration:**

- `worker.js` - Add new job definitions and scheduling

