# ZenArch - Children's Interactive Learning Portal

## 🎯 Project Vision

A production-grade interactive learning platform for children (9+) that teaches programming through hands-on coding exercises, similar to Flexbox Froggy. The platform features real-time code execution, progressive difficulty, and a comprehensive admin panel for content management.

---

## 🏛️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                       CDN LAYER                              │
│              (Vercel Edge / Cloudflare)                      │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                    FRONTEND LAYER                           │
│  ┌─────────────────┐    ┌──────────────────────────────┐   │
│  │  Student Portal │    │      Admin Dashboard         │   │
│  │   (React SPA)   │    │       (React SPA)            │   │
│  └────────┬─────────┘    └─────────────┬────────────────┘   │
└───────────┼─────────────────────────────┼───────────────────┘
            │                             │
            └──────────────┬──────────────┘
                           │ HTTPS
┌──────────────────────────▼──────────────────────────────────┐
│                     API GATEWAY                              │
│                  (Express.js + Nginx)                        │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐    │
│  │ Rate Limit  │ │ Auth Guard  │ │ Request Validation  │    │
│  └─────────────┘ └─────────────┘ └─────────────────────┘    │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                   SERVICE LAYER                              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐    │
│  │    Auth     │ │   Content   │ │    Code Execution   │    │
│  │   Service   │ │   Service   │ │      Service        │    │
│  └─────────────┘ └─────────────┘ └──────────┬──────────┘    │
│  ┌─────────────┐ ┌─────────────┐            │               │
│  │   Progress  │ │    User     │            │               │
│  │   Service   │ │   Service   │            │               │
│  └─────────────┘ └─────────────┘            │               │
└─────────────────┬───────────────────────────┼───────────────┘
                  │                           │
     ┌────────────▼────────────┐    ┌─────────▼──────────┐
     │      DATA LAYER         │    │  EXTERNAL SERVICES │
     │  ┌─────────┐ ┌───────┐  │    │  ┌──────────────┐  │
     │  │ MongoDB │ │ Redis │  │    │  │  Piston API  │  │
     │  │  Atlas  │ │Upstash│  │    │  │  (Primary)   │  │
     │  └─────────┘ └───────┘  │    │  ├──────────────┤  │
     │                         │    │  │  Judge0 CE   │  │
     │                         │    │  │  (Fallback)  │  │
     └─────────────────────────┘    │  └──────────────┘  │
                                    └────────────────────┘
```

---

## 🗄️ Database Architecture

### MongoDB Collections

#### 1. Users Collection
```javascript
{
  _id: ObjectId,
  email: String,              // unique, indexed
  passwordHash: String,
  role: {                     // enum
    type: String,
    enum: ['student', 'admin', 'parent']
  },
  profile: {
    displayName: String,
    age: Number,
    avatar: String,           // URL or preset ID
    theme: String             // 'light', 'dark', 'colorful'
  },
  parentEmail: String,        // For COPPA compliance (age < 13)
  preferences: {
    soundEnabled: Boolean,
    animationsEnabled: Boolean,
    fontSize: String,
    codeEditorTheme: String
  },
  stats: {
    totalPoints: Number,
    lessonsCompleted: Number,
    streakDays: Number,
    lastActiveAt: Date
  },
  isActive: Boolean,
  isVerified: Boolean,
  createdAt: Date,
  updatedAt: Date
}

// Indexes
{ email: 1 } // unique
{ role: 1 }
{ "stats.totalPoints": -1 } // for leaderboards
{ createdAt: -1 }
```

#### 2. Languages Collection
```javascript
{
  _id: ObjectId,
  name: String,               // "Python", "JavaScript", "HTML/CSS"
  slug: String,               // unique, indexed: "python", "javascript"
  icon: String,               // URL or emoji
  color: String,              // Theme color "#3776AB"
  description: String,
  shortDescription: String,   // For cards
  
  executorConfig: {
    type: String,             // 'piston', 'browser', 'sandbox'
    runtime: String,          // 'python3', 'node'
    version: String,          // '3.10'
    timeout: Number,          // ms
    memoryLimit: Number       // bytes
  },
  
  editorConfig: {
    defaultCode: String,
    theme: String,
    language: String,         // Monaco language ID
    tabSize: Number
  },
  
  metadata: {
    difficulty: Number,       // 1-5
    estimatedHours: Number,
    ageRecommendation: Number,
    prerequisites: [String]   // language slugs
  },
  
  displayOrder: Number,
  isActive: Boolean,
  createdAt: Date,
  updatedAt: Date
}

// Indexes
{ slug: 1 } // unique
{ displayOrder: 1, isActive: 1 }
```

#### 3. Topics Collection
```javascript
{
  _id: ObjectId,
  languageId: ObjectId,       // ref: languages, indexed
  
  title: String,
  slug: String,               // unique within language
  description: String,
  icon: String,               // emoji or icon name
  
  displayOrder: Number,
  
  metadata: {
    difficulty: Number,       // 1-5
    estimatedMinutes: Number,
    lessonCount: Number,      // denormalized for performance
    prerequisites: [ObjectId] // topic IDs
  },
  
  unlockCriteria: {
    type: String,             // 'none', 'previous', 'points', 'specific'
    value: Mixed              // depends on type
  },
  
  isActive: Boolean,
  createdAt: Date,
  updatedAt: Date
}

// Indexes
{ languageId: 1, displayOrder: 1 }
{ languageId: 1, slug: 1 } // compound unique
{ isActive: 1 }
```

#### 4. Lessons Collection
```javascript
{
  _id: ObjectId,
  topicId: ObjectId,          // ref: topics, indexed
  languageId: ObjectId,       // denormalized for faster queries
  
  title: String,
  slug: String,
  
  type: {
    type: String,
    enum: [
      'tutorial',           // Read + understand
      'interactive',        // Flexbox Froggy style
      'code_playground',    // Free coding
      'code_challenge',     // Complete the task
      'mcq',               // Multiple choice
      'fill_blank',        // Fill in the code
      'debug'              // Fix the bug
    ]
  },
  
  displayOrder: Number,
  difficulty: Number,         // 1-5
  points: Number,             // XP for completion
  
  content: {
    instruction: String,      // Markdown
    theory: String,           // Optional explanation
    hints: [String],          // Progressive hints
    starterCode: String,
    solutionCode: String,
    
    // For interactive lessons (Flexbox Froggy style)
    interactiveConfig: {
      htmlTemplate: String,   // Base HTML
      cssBase: String,        // Non-editable CSS
      targetState: Object,    // What success looks like
      editableProperties: [String],
      visualFeedback: Boolean
    },
    
    // For MCQ
    mcqConfig: {
      question: String,
      options: [{
        id: String,
        text: String,
        isCorrect: Boolean
      }],
      explanation: String
    },
    
    // For fill_blank
    fillBlankConfig: {
      codeTemplate: String,   // With ___BLANK___ markers
      blanks: [{
        id: String,
        answer: String,
        alternateAnswers: [String],
        hint: String
      }]
    }
  },
  
  validation: {
    type: String,             // 'output', 'test_cases', 'visual', 'manual'
    expectedOutput: String,
    testCases: [{
      input: String,
      expectedOutput: String,
      isHidden: Boolean
    }],
    customValidator: String   // JS function as string (advanced)
  },
  
  isActive: Boolean,
  createdAt: Date,
  updatedAt: Date
}

// Indexes
{ topicId: 1, displayOrder: 1 }
{ topicId: 1, slug: 1 } // compound unique
{ languageId: 1, type: 1 }
{ isActive: 1 }
```

#### 5. UserProgress Collection
```javascript
{
  _id: ObjectId,
  userId: ObjectId,           // ref: users
  
  // Can be at different levels
  entityType: String,         // 'language', 'topic', 'lesson'
  entityId: ObjectId,         // the referenced entity
  
  status: {
    type: String,
    enum: ['locked', 'available', 'in_progress', 'completed']
  },
  
  // For lessons specifically
  lessonData: {
    score: Number,            // 0-100
    attempts: Number,
    timeSpent: Number,        // seconds
    hintsUsed: Number,
    bestCode: String,         // Their best solution
    completedAt: Date,
    
    submissions: [{           // Last N submissions
      code: String,
      output: String,
      isCorrect: Boolean,
      timestamp: Date
    }]
  },
  
  // For topics
  topicData: {
    lessonsCompleted: Number,
    totalLessons: Number,
    averageScore: Number,
    startedAt: Date,
    completedAt: Date
  },
  
  // For languages
  languageData: {
    topicsCompleted: Number,
    totalTopics: Number,
    totalPoints: Number,
    startedAt: Date
  },
  
  lastAccessedAt: Date,
  createdAt: Date,
  updatedAt: Date
}

// Indexes - Critical for performance
{ userId: 1, entityType: 1, entityId: 1 } // compound unique
{ userId: 1, status: 1 }
{ userId: 1, "lessonData.completedAt": -1 }
```

#### 6. RefreshTokens Collection
```javascript
{
  _id: ObjectId,
  userId: ObjectId,
  token: String,              // hashed
  userAgent: String,
  ipAddress: String,
  expiresAt: Date,
  createdAt: Date
}

// Indexes
{ token: 1 }
{ userId: 1 }
{ expiresAt: 1 } // TTL index
```

#### 7. AnalyticsEvents Collection (Optional - Phase 2)
```javascript
{
  _id: ObjectId,
  userId: ObjectId,
  sessionId: String,
  
  eventType: String,          // 'lesson_start', 'code_run', 'hint_used', etc.
  eventData: Object,
  
  metadata: {
    userAgent: String,
    platform: String,
    screenSize: String
  },
  
  timestamp: Date
}

// Indexes
{ userId: 1, timestamp: -1 }
{ eventType: 1, timestamp: -1 }
{ timestamp: -1 } // TTL after 90 days
```

---

## 📁 Project Structure

### Monorepo Structure (Recommended for Solo Dev)

```
zenarch/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
│
├── apps/
│   ├── client/                    # Student Portal
│   │   ├── public/
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── App.jsx
│   │   │   │   ├── Router.jsx
│   │   │   │   └── providers/
│   │   │   │
│   │   │   ├── pages/
│   │   │   │   ├── Home/
│   │   │   │   ├── Languages/
│   │   │   │   ├── Topics/
│   │   │   │   ├── Lesson/
│   │   │   │   ├── Profile/
│   │   │   │   └── Auth/
│   │   │   │
│   │   │   ├── features/
│   │   │   │   ├── auth/
│   │   │   │   │   ├── components/
│   │   │   │   │   ├── hooks/
│   │   │   │   │   ├── store.js
│   │   │   │   │   └── api.js
│   │   │   │   │
│   │   │   │   ├── codeEditor/
│   │   │   │   │   ├── Editor.jsx
│   │   │   │   │   ├── OutputPanel.jsx
│   │   │   │   │   ├── InteractivePreview.jsx
│   │   │   │   │   ├── Console.jsx
│   │   │   │   │   └── hooks/
│   │   │   │   │       └── useCodeExecution.js
│   │   │   │   │
│   │   │   │   ├── lessons/
│   │   │   │   │   ├── LessonViewer.jsx
│   │   │   │   │   ├── MCQQuestion.jsx
│   │   │   │   │   ├── FillBlank.jsx
│   │   │   │   │   ├── InteractiveLesson.jsx
│   │   │   │   │   └── ProgressBar.jsx
│   │   │   │   │
│   │   │   │   └── gamification/
│   │   │   │       ├── PointsBadge.jsx
│   │   │   │       ├── StreakCounter.jsx
│   │   │   │       └── Achievements.jsx
│   │   │   │
│   │   │   ├── shared/
│   │   │   │   ├── components/
│   │   │   │   │   ├── ui/            # Button, Input, Modal, Card
│   │   │   │   │   ├── layout/        # Header, Footer, Sidebar
│   │   │   │   │   └── feedback/      # Toast, Loader, Error
│   │   │   │   │
│   │   │   │   ├── hooks/
│   │   │   │   ├── utils/
│   │   │   │   └── constants/
│   │   │   │
│   │   │   ├── services/
│   │   │   │   └── api.js             # Axios instance
│   │   │   │
│   │   │   ├── stores/
│   │   │   │   └── index.js           # Zustand root store
│   │   │   │
│   │   │   ├── styles/
│   │   │   │   ├── globals.css
│   │   │   │   ├── themes/
│   │   │   │   └── animations/
│   │   │   │
│   │   │   └── main.jsx
│   │   │
│   │   ├── index.html
│   │   ├── vite.config.js
│   │   └── package.json
│   │
│   └── admin/                     # Admin Dashboard
│       ├── src/
│       │   ├── pages/
│       │   │   ├── Dashboard/
│       │   │   ├── Languages/
│       │   │   │   ├── LanguageList.jsx
│       │   │   │   ├── LanguageForm.jsx
│       │   │   │   └── LanguagePreview.jsx
│       │   │   │
│       │   │   ├── Topics/
│       │   │   ├── Lessons/
│       │   │   │   ├── LessonList.jsx
│       │   │   │   ├── LessonForm.jsx
│       │   │   │   ├── LessonPreview.jsx
│       │   │   │   └── editors/
│       │   │   │       ├── TutorialEditor.jsx
│       │   │   │       ├── MCQEditor.jsx
│       │   │   │       ├── FillBlankEditor.jsx
│       │   │   │       └── InteractiveEditor.jsx
│       │   │   │
│       │   │   └── Users/
│       │   │
│       │   └── [similar structure to client]
│       │
│       └── package.json
│
├── packages/
│   └── shared/                    # Shared utilities
│       ├── constants/
│       ├── types/
│       ├── validators/
│       └── package.json
│
├── server/
│   ├── src/
│   │   ├── config/
│   │   │   ├── index.js           # Central config
│   │   │   ├── database.js        # MongoDB connection
│   │   │   ├── redis.js           # Redis connection
│   │   │   └── env.js             # Environment validation
│   │   │
│   │   ├── middleware/
│   │   │   ├── auth.js            # JWT verification
│   │   │   ├── roleCheck.js       # RBAC middleware
│   │   │   ├── rateLimiter.js     # Rate limiting
│   │   │   ├── validate.js        # Request validation
│   │   │   ├── errorHandler.js    # Global error handler
│   │   │   └── requestLogger.js   # HTTP logging
│   │   │
│   │   ├── models/
│   │   │   ├── User.js
│   │   │   ├── Language.js
│   │   │   ├── Topic.js
│   │   │   ├── Lesson.js
│   │   │   ├── UserProgress.js
│   │   │   ├── RefreshToken.js
│   │   │   └── index.js
│   │   │
│   │   ├── services/
│   │   │   ├── auth.service.js
│   │   │   ├── user.service.js
│   │   │   ├── language.service.js
│   │   │   ├── topic.service.js
│   │   │   ├── lesson.service.js
│   │   │   ├── progress.service.js
│   │   │   ├── codeExecution.service.js
│   │   │   ├── cache.service.js
│   │   │   └── email.service.js
│   │   │
│   │   ├── controllers/
│   │   │   ├── auth.controller.js
│   │   │   ├── admin/
│   │   │   │   ├── language.controller.js
│   │   │   │   ├── topic.controller.js
│   │   │   │   ├── lesson.controller.js
│   │   │   │   └── user.controller.js
│   │   │   │
│   │   │   └── user/
│   │   │       ├── content.controller.js
│   │   │       ├── progress.controller.js
│   │   │       └── profile.controller.js
│   │   │
│   │   ├── routes/
│   │   │   ├── index.js
│   │   │   ├── auth.routes.js
│   │   │   ├── admin.routes.js
│   │   │   └── user.routes.js
│   │   │
│   │   ├── utils/
│   │   │   ├── validators/
│   │   │   ├── helpers/
│   │   │   ├── ApiError.js
│   │   │   └── ApiResponse.js
│   │   │
│   │   └── app.js
│   │
│   ├── tests/
│   │   ├── unit/
│   │   ├── integration/
│   │   └── setup.js
│   │
│   ├── scripts/
│   │   ├── seed.js               # Seed sample data
│   │   └── migrate.js            # DB migrations
│   │
│   ├── package.json
│   └── nodemon.json
│
├── docker/
│   ├── Dockerfile.server
│   ├── Dockerfile.client
│   └── docker-compose.yml
│
├── docs/
│   ├── API.md
│   ├── DEPLOYMENT.md
│   └── CONTRIBUTING.md
│
├── .env.example
├── .gitignore
├── package.json                  # Root package.json for workspaces
└── README.md
```

---

## 🔌 API Design

### Base URL Structure
```
Production: https://api.zenarch.dev/v1
Development: http://localhost:5000/api/v1
```

### Response Format
```javascript
// Success Response
{
  success: true,
  data: { ... },
  message: "Operation successful",
  meta: {
    pagination: {
      page: 1,
      limit: 20,
      total: 100,
      pages: 5
    }
  }
}

// Error Response
{
  success: false,
  error: {
    code: "VALIDATION_ERROR",
    message: "Invalid input data",
    details: [
      { field: "email", message: "Invalid email format" }
    ]
  }
}
```

### Endpoints

#### Health Check
```
GET    /health                   # Basic health check
GET    /health/ready             # Ready check (DB + Redis)
```

#### Authentication
```
POST   /api/v1/auth/register         # Create account
POST   /api/v1/auth/login            # Get tokens
POST   /api/v1/auth/logout           # Invalidate refresh token
POST   /api/v1/auth/refresh          # Refresh access token
POST   /api/v1/auth/forgot-password  # Send reset email
POST   /api/v1/auth/reset-password   # Set new password
GET    /api/v1/auth/me               # Get current user
```

#### Admin - Languages
```
GET    /api/v1/admin/languages              # List all (with inactive)
POST   /api/v1/admin/languages              # Create language
GET    /api/v1/admin/languages/:id          # Get single
PUT    /api/v1/admin/languages/:id          # Update language
DELETE /api/v1/admin/languages/:id          # Soft delete
PATCH  /api/v1/admin/languages/:id/toggle   # Toggle active
PATCH  /api/v1/admin/languages/reorder      # Reorder languages
```

#### Admin - Topics
```
GET    /api/v1/admin/topics                 # List with filters
POST   /api/v1/admin/topics                 # Create topic
GET    /api/v1/admin/topics/:id             # Get single with stats
PUT    /api/v1/admin/topics/:id             # Update topic
DELETE /api/v1/admin/topics/:id             # Soft delete
PATCH  /api/v1/admin/topics/reorder         # Reorder within language
```

#### Admin - Lessons
```
GET    /api/v1/admin/lessons                # List with filters
POST   /api/v1/admin/lessons                # Create lesson
GET    /api/v1/admin/lessons/:id            # Get full lesson
PUT    /api/v1/admin/lessons/:id            # Update lesson
DELETE /api/v1/admin/lessons/:id            # Soft delete
POST   /api/v1/admin/lessons/:id/preview    # Test lesson
POST   /api/v1/admin/lessons/:id/duplicate  # Clone lesson
```

#### Admin - Users
```
GET    /api/v1/admin/users                  # List with pagination
GET    /api/v1/admin/users/:id              # Get user details + progress
PATCH  /api/v1/admin/users/:id/status       # Activate/Deactivate
DELETE /api/v1/admin/users/:id              # Soft delete
GET    /api/v1/admin/users/:id/progress     # Full progress details
```

#### Student - Content
```
GET    /api/v1/content/languages            # Active languages only
GET    /api/v1/content/languages/:slug      # Language with topics
GET    /api/v1/content/topics/:id           # Topic with lessons list
GET    /api/v1/content/lessons/:id          # Full lesson content
```

#### Student - Code Execution
```
POST   /api/v1/code/execute                 # Run code
  Body: { language, code, input }
  Response: { output, error, executionTime }

POST   /api/v1/code/validate                # Validate against expected
  Body: { lessonId, code }
  Response: { isCorrect, feedback, score }
```

#### Student - Progress
```
GET    /api/v1/progress                     # Overall dashboard
GET    /api/v1/progress/language/:id        # Language progress
GET    /api/v1/progress/topic/:id           # Topic progress
GET    /api/v1/progress/lesson/:id          # Lesson attempts

POST   /api/v1/progress/lesson/:id/start    # Mark as started
POST   /api/v1/progress/lesson/:id/submit   # Submit answer
POST   /api/v1/progress/lesson/:id/complete # Mark complete
```

#### Student - Profile
```
GET    /api/v1/profile                      # Get profile
PUT    /api/v1/profile                      # Update profile
GET    /api/v1/profile/stats                # Stats & achievements
PUT    /api/v1/profile/preferences          # Update preferences
```

---

## ⚡ Code Execution Strategy

### Multi-Provider Approach (Critical for Reliability)

```javascript
// codeExecution.service.js

const providers = [
  {
    name: 'piston',
    url: 'https://emkc.org/api/v2/piston',
    priority: 1,
    // Piston public API - no auth required, generous limits
    // Self-hostable for production: github.com/engineer-man/piston
    limits: { rpm: 200 }  // requests per minute (public instance)
  },
  {
    name: 'judge0-ce',
    // Self-host Judge0 CE for production (free)
    // Or use RapidAPI: judge0-ce.p.rapidapi.com (100 req/day free)
    url: process.env.JUDGE0_URL || 'https://judge0-ce.p.rapidapi.com',
    priority: 2,
    limits: { rpm: 30 }
  },
  {
    name: 'browser-sandbox',  // For HTML/CSS/JS only
    type: 'client-side',      // Runs entirely in user's browser
    priority: 3               // Fallback - always available
  }
];

// Fallback logic
async function executeCode(language, code, input) {
  for (const provider of sortedProviders) {
    try {
      if (await isProviderHealthy(provider)) {
        return await executeWithProvider(provider, language, code, input);
      }
    } catch (error) {
      logProviderError(provider, error);
      continue;  // Try next provider
    }
  }
  throw new CodeExecutionError('All providers unavailable');
}
```

### Language-Specific Handlers

```javascript
// For HTML/CSS/JS - Use in-browser execution
{
  type: 'browser',
  handler: 'WebContainerPreview'  // sandboxed iframe
}

// For Python, Java, C++ - Use Piston/Judge0
{
  type: 'api',
  handler: 'PistonExecutor'
}
```

### Security Measures

1. **Request Size Limit**: 50KB max code size
2. **Execution Timeout**: 10 seconds max
3. **Memory Limit**: 128MB per execution
4. **Rate Limiting**: 30 executions per user per minute
5. **Input Sanitization**: Strip potentially dangerous patterns
6. **No Network Access**: Execution environments isolated

---

## 🔐 Authentication Flow

### Token Strategy

```
Access Token:
  - Type: JWT
  - Expires: 15 minutes
  - Stored: Memory (React state/Zustand)
  - Contains: userId, role, permissions

Refresh Token:
  - Type: Random string (hashed in DB)
  - Expires: 7 days
  - Stored: httpOnly cookie
  - Rotation: New token on each refresh
```

### Flow Diagrams

```
REGISTRATION:
User → POST /auth/register → Validate → Hash password → 
Create user → Generate tokens → Set cookie → Return access token

LOGIN:
User → POST /auth/login → Validate → Check password → 
Generate tokens → Store refresh in Redis → Set cookie → Return access token

TOKEN REFRESH:
Frontend (401) → POST /auth/refresh (includes cookie) → 
Validate refresh token → Rotate token → Return new access token

LOGOUT:
User → POST /auth/logout → Delete refresh from Redis → 
Clear cookie → Return success
```

### Middleware Chain

```javascript
// Protected route middleware stack
router.use('/api/v1/admin/*', [
  rateLimiter({ windowMs: 60000, max: 100 }),
  authenticate,              // Verify JWT
  requireRole('admin'),      // Check role
  validateRequest(schema),   // Validate input
  controller                 // Handle request
]);
```

---

## 💾 Caching Strategy

### Redis Cache Layers

```javascript
// Cache keys pattern
const cacheKeys = {
  // Static content (TTL: 1 hour)
  'languages:active': 'all active languages',
  'language:{slug}': 'single language with config',
  'topic:{id}': 'topic with lessons preview',
  'lesson:{id}:content': 'full lesson content',
  
  // User-specific (TTL: 15 minutes)
  'user:{id}:profile': 'user profile data',
  'user:{id}:progress:summary': 'progress overview',
  
  // Session (TTL: matches JWT)
  'session:{userId}': 'active session data',
  'refresh:{token}': 'refresh token validation'
};
```

### Cache Invalidation

```javascript
// On admin content update
async function invalidateContentCache(entityType, entityId) {
  const patterns = {
    language: [`language:*`, `languages:*`],
    topic: [`topic:${entityId}`, `language:*`],
    lesson: [`lesson:${entityId}:*`, `topic:*`]
  };
  
  await redis.del(patterns[entityType]);
}
```

---

## 🚀 Deployment Architecture

### Free Tier Stack (2024 Options)

> **Note**: Railway removed their free tier in 2023. Use Render.com instead.

```
┌─────────────────────────────────────────────────────────┐
│                    VERCEL (Frontend)                     │
│        Client SPA + Admin SPA (Separate deploys)        │
│              Free: 100GB bandwidth/month                 │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│                  RENDER.COM (Backend)                    │
│           Node.js API (sleeps after 15min idle)         │
│        Free: 750 hours/month, auto-wake on request       │
└────────────┬─────────────────────────────┬──────────────┘
             │                             │
┌────────────▼────────────┐   ┌────────────▼────────────┐
│   MONGODB ATLAS         │   │      UPSTASH REDIS      │
│   Free: 512MB storage   │   │   Free: 10k cmds/day    │
│   Shared M0 cluster     │   │   256MB storage         │
└─────────────────────────┘   └─────────────────────────┘
```

### Alternative Hosting Options

| Service | Free Tier | Best For | Limitations |
|---------|-----------|----------|-------------|
| **Render.com** | 750 hrs/month | Primary backend | Sleeps after 15min idle |
| **Fly.io** | 3 shared VMs | Low latency | 256MB RAM per VM |
| **Cyclic.sh** | Unlimited | Serverless | 10 sec max execution |
| **Koyeb** | 2 nano instances | Always-on | Limited regions |
| **Adaptable.io** | Always free | Node.js | 256MB RAM |

### Environment Configuration

```bash
# .env.example

# Server
NODE_ENV=production
PORT=5000
API_VERSION=v1

# Database
MONGODB_URI=mongodb+srv://...
REDIS_URL=redis://...

# JWT
JWT_ACCESS_SECRET=your-32-char-secret
JWT_REFRESH_SECRET=your-32-char-secret
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=7d

# Code Execution
PISTON_API_URL=https://emkc.org/api/v2/piston
CODE_EXECUTION_TIMEOUT=10000
CODE_EXECUTION_MEMORY=134217728

# External Services
CLOUDINARY_URL=cloudinary://...
SENDGRID_API_KEY=SG...

# Frontend URLs (for CORS)
CLIENT_URL=https://app.zenarch.dev
ADMIN_URL=https://admin.zenarch.dev
```

### CORS Configuration

```javascript
// server/src/config/cors.js
const corsOptions = {
  origin: [
    process.env.CLIENT_URL,
    process.env.ADMIN_URL,
    // Allow localhost in development
    ...(process.env.NODE_ENV === 'development' 
      ? ['http://localhost:5173', 'http://localhost:5174'] 
      : [])
  ],
  credentials: true,  // Allow cookies (refresh tokens)
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400  // 24 hours preflight cache
};

// In app.js
app.use(cors(corsOptions));
```

### Cold Start Handling (Render.com)

> **Important**: Render's free tier sleeps after 15 minutes of inactivity.
> First request after sleep takes 30-50 seconds to "wake up".

**Solution:**
```javascript
// Frontend: Add loading state for cold starts
const [isWaking, setIsWaking] = useState(false);

async function fetchWithWakeup(url) {
  const timeout = setTimeout(() => setIsWaking(true), 3000);
  try {
    const res = await fetch(url);
    return res;
  } finally {
    clearTimeout(timeout);
    setIsWaking(false);
  }
}

// Show "Server is waking up..." message if isWaking is true
```

---

## 🎮 Interactive Lesson Types

### Type 1: Tutorial
```
├── Theory explanation (Markdown)
├── Code example with syntax highlighting
├── Try It Yourself button
└── Next lesson button
```

### Type 2: Code Playground
```
├── Instructions panel
├── Code editor (Monaco/CodeMirror)
├── Run button
├── Output/Console panel
└── Hints (progressive reveal)
```

### Type 3: Interactive (Flexbox Froggy Style)
```
├── Visual game area (animated)
├── CSS/Code input area
├── Apply button
├── Visual feedback (success animation)
├── Level progression
└── Celebration on complete
```

### Type 4: MCQ
```
├── Question text
├── Options (radio buttons)
├── Submit button
├── Instant feedback
├── Explanation reveal
```

### Type 5: Fill in the Blanks
```
├── Code with blanks (input fields)
├── Hints per blank
├── Check button
├── Run to verify
└── Solution reveal option
```

### Type 6: Debug Challenge
```
├── Buggy code displayed
├── Error message shown
├── Editable code area
├── Run to test fix
├── Success criteria
```

---

## 🎨 Child-Friendly UX Guidelines

### Design Principles
1. **Large touch targets** (min 48x48px)
2. **Colorful but not overwhelming**
3. **Immediate feedback** for all actions
4. **Progress visualization** everywhere
5. **Celebration animations** on achievements
6. **Voice/sound feedback** (optional)
7. **Simple navigation** (max 3 levels deep)
8. **Undo/retry** always available

### Theme Options
```css
/* Default: Colorful */
--primary: #6366f1;  /* Indigo */
--success: #22c55e;  /* Green */
--warning: #f59e0b;  /* Amber */
--error: #ef4444;    /* Red */

/* Dark Mode */
/* High contrast with neon accents */

/* Dyslexia-Friendly */
/* OpenDyslexic font, higher contrast */
```

---

## 📊 Development Phases

### Phase 1: Foundation (Weeks 1-2)
- [ ] Project setup (monorepo, ESLint, Prettier)
- [ ] MongoDB Atlas + Redis Upstash setup
- [ ] User model + authentication (register, login, refresh)
- [ ] Basic Express server with middleware
- [ ] Deploy backend to Render.com
- [ ] Deploy empty frontend to Vercel
- [ ] Configure CORS for cross-origin requests

### Phase 2: Admin CRUD (Weeks 3-4)
- [ ] Admin authentication (role check)
- [ ] Language CRUD + API
- [ ] Topic CRUD + API
- [ ] Lesson CRUD (tutorial type only) + API
- [ ] Basic admin UI with forms
- [ ] Rich text editor integration

### Phase 3: Student Portal (Weeks 5-6)
- [ ] Language selection page
- [ ] Topic grid view
- [ ] Lesson list view
- [ ] Code editor integration
- [ ] Code execution service (Piston)
- [ ] Output display panel

### Phase 4: Progress System (Week 7)
- [ ] Progress tracking backend
- [ ] Progress UI components
- [ ] Dashboard with stats
- [ ] Completion states

### Phase 5: Interactive Features (Weeks 8-9)
- [ ] MCQ lesson type
- [ ] Fill-in-the-blank type
- [ ] Visual feedback components
- [ ] Hints system

### Phase 6: Polish (Weeks 10-12)
- [ ] Child-friendly UI refinements
- [ ] Animations and transitions
- [ ] Mobile responsiveness
- [ ] Testing
- [ ] Documentation

---

## 📚 Learning Resources

### Docker & Containerization
- **Start here**: [Docker 101 Tutorial](https://www.docker.com/101-tutorial/)
- **Video**: "Docker Tutorial for Beginners" - TechWorld with Nana (YouTube)
- **Practice**: [Play with Docker](https://labs.play-with-docker.com/)

### CI/CD & GitHub Actions
- **Official**: [GitHub Actions Documentation](https://docs.github.com/en/actions)
- **Video**: "GitHub Actions Tutorial" - Fireship (YouTube, 10 min)
- **Example**: `.github/workflows` in popular open-source MERN projects

### Linux & Server Management
- **Interactive**: [Linux Journey](https://linuxjourney.com/) - Free course
- **Video**: "Linux for Programmers" - NetworkChuck (YouTube)
- **Cheatsheet**: [TLDR pages](https://tldr.sh/)

### Redis
- **Free Course**: [Redis University](https://university.redis.com/)
- **Quick Tutorial**: "Redis Crash Course" - Web Dev Simplified (YouTube)
- **Practice**: Redis CLI in [Try Redis](https://try.redis.io/)

### System Design
- **Free GitHub Repo**: [System Design Primer](https://github.com/donnemartin/system-design-primer)
- **Video Channel**: ByteByteGo on YouTube
- **For this project**: Focus on caching, rate limiting, and async processing

### Security Best Practices
- **OWASP**: [Top 10 Web Application Security Risks](https://owasp.org/www-project-top-ten/)
- **Node.js Specific**: [Node.js Security Best Practices](https://github.com/goldbergyoni/nodebestpractices#6-security-best-practices)

---

## ✅ Pre-Launch Checklist

### Security
- [ ] All passwords hashed with bcrypt (cost factor 12)
- [ ] JWT secrets are 32+ characters
- [ ] Rate limiting on all endpoints
- [ ] Input validation on all endpoints
- [ ] CORS configured correctly
- [ ] HTTP headers secured (Helmet.js)
- [ ] No secrets in codebase

### Performance
- [ ] Database indexes created
- [ ] Redis caching implemented
- [ ] Pagination on all list endpoints
- [ ] Images optimized (WebP)
- [ ] Frontend code split

### Reliability
- [ ] Error handling on all endpoints
- [ ] Logging configured
- [ ] Health check endpoint
- [ ] Graceful shutdown
- [ ] Connection retry logic

### User Experience
- [ ] Loading states everywhere
- [ ] Error messages user-friendly
- [ ] Mobile responsive
- [ ] Keyboard accessible
- [ ] Works without JavaScript (graceful degradation)

---

## 🆘 Common Pitfalls & Solutions

| Pitfall | Solution |
|---------|----------|
| Running user code on your server | Always use sandboxed execution (Piston/Judge0) |
| Storing passwords in plain text | Use bcrypt with 12+ rounds |
| Committing .env files | Add to .gitignore immediately |
| No rate limiting | Implement from day 1 with express-rate-limit |
| Trusting frontend validation | Always validate on backend |
| N+1 query problems | Use MongoDB populate wisely, denormalize where needed |
| No error handling | Global error handler + try-catch in async routes |
| Hardcoding URLs | Use environment variables |

---

## 🏆 Success Metrics (Track from Day 1)

1. **Lessons completed per user** (engagement)
2. **Time to complete first lesson** (onboarding)
3. **Return rate after 1 day** (retention)
4. **Code execution success rate** (reliability)
5. **API response times** (performance)
6. **Error rates** (stability)

---

## 💡 Final Tips

1. **Start with seed data** - Create 1 complete language with 2 topics and 5 lessons before building features
2. **Test code execution first** - This is your riskiest integration
3. **Build admin before student** - You need to create content to test student features
4. **Mobile-first CSS** - Children use tablets more than desktops
5. **Celebration moments** - Add confetti/animations for completions early
6. **Version your API** - Start with `/v1` from day 1

---

*This architecture is designed to be:*
- ✅ **Production-ready** - Follows industry patterns
- ✅ **Scalable** - Can handle 10,000+ users with minimal changes
- ✅ **Free to deploy** - All services have adequate free tiers
- ✅ **Solo-dev friendly** - Achievable in 10-12 weeks with AI assistance
- ✅ **Extensible** - Easy to add gamification, multiplayer, and more later

*Built for ZenArch Learning Platform*
