# ZestCoder - Complete Implementation Roadmap

> **Total Timeline**: 10-14 weeks (solo developer with AI tools)  
> **Skill Level**: MERN stack known, learning deployment & DevOps along the way

---

## 📋 Pre-Development Setup (Day 1-2)

### Step 1: Create Accounts (All Free)

| Service | URL | Purpose | Free Tier |
|---------|-----|---------|-----------|
| MongoDB Atlas | mongodb.com/atlas | Database | 512MB |
| Upstash | upstash.com | Redis cache | 10k cmds/day |
| Render.com | render.com | Backend hosting | 750 hrs/month |
| Vercel | vercel.com | Frontend hosting | 100GB bandwidth |
| Cloudinary | cloudinary.com | Image storage | 25GB |
| SendGrid | sendgrid.com | Emails | 100 emails/day |

### Step 2: Install Development Tools

```bash
# Check Node.js version (need 18+)
node -v

# If not installed or outdated:
# Mac: brew install node
# Or download from nodejs.org

# Install pnpm (faster than npm)
npm install -g pnpm

# Verify
pnpm -v
```

### Step 3: Set Up VS Code Extensions

Install these extensions:
- ESLint
- Prettier
- GitLens
- Thunder Client (API testing)
- MongoDB for VS Code
- ES7+ React/Redux/React-Native snippets

---

## 🏗️ Phase 1: Project Foundation (Week 1-2)

### 1.1 Initialize Monorepo

```bash
# Create project folder
mkdir zestcoder && cd zestcoder

# Initialize git
git init

# Create folder structure
mkdir -p apps/client apps/admin server packages/shared docs

# Initialize root package.json
pnpm init
```

**Edit `package.json`:**
```json
{
  "name": "zestcoder",
  "private": true,
  "scripts": {
    "dev:server": "pnpm --filter server dev",
    "dev:client": "pnpm --filter client dev",
    "dev:admin": "pnpm --filter admin dev",
    "dev": "pnpm run --parallel dev:server dev:client"
  },
  "devDependencies": {
    "prettier": "^3.2.0",
    "eslint": "^8.56.0"
  }
}
```

### 1.2 Set Up Backend Server

```bash
cd server
pnpm init
```

**Install dependencies:**
```bash
pnpm add express mongoose dotenv cors helmet morgan bcryptjs jsonwebtoken cookie-parser express-rate-limit express-validator ioredis

pnpm add -D nodemon
```

**Create `server/src/app.js`:**
```javascript
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');
const cookieParser = require('cookie-parser');
const rateLimit = require('express-rate-limit');

const app = express();

// Security middleware
app.use(helmet());
app.use(cors({
  origin: process.env.CLIENT_URL || 'http://localhost:5173',
  credentials: true
}));

// Rate limiting
app.use(rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100
}));

// Body parsing
app.use(express.json({ limit: '10kb' }));
app.use(cookieParser());

// Logging
app.use(morgan('dev'));

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// API routes will go here
app.use('/api/v1', require('./routes'));

// Error handler
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(err.status || 500).json({
    success: false,
    error: {
      message: err.message || 'Internal Server Error'
    }
  });
});

module.exports = app;
```

**Create `server/src/index.js`:**
```javascript
require('dotenv').config();
const app = require('./app');
const connectDB = require('./config/database');

const PORT = process.env.PORT || 5000;

// Connect to MongoDB
connectDB();

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### 1.3 Database Connection

**Create `server/src/config/database.js`:**
```javascript
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI);
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error: ${error.message}`);
    process.exit(1);
  }
};

module.exports = connectDB;
```

**Create `server/.env`:**
```env
NODE_ENV=development
PORT=5000
MONGODB_URI=mongodb+srv://YOUR_USERNAME:YOUR_PASSWORD@cluster.mongodb.net/zestcoder
JWT_ACCESS_SECRET=your-super-secret-32-char-minimum-key-here
JWT_REFRESH_SECRET=another-super-secret-32-char-minimum-key
CLIENT_URL=http://localhost:5173
```

> ⚠️ **IMPORTANT**: Never commit `.env` files! Add to `.gitignore`

### 1.4 Create User Model

**Create `server/src/models/User.js`:**
```javascript
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true
  },
  passwordHash: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['student', 'admin', 'parent'],
    default: 'student'
  },
  profile: {
    displayName: String,
    age: Number,
    avatar: String
  },
  isActive: {
    type: Boolean,
    default: true
  }
}, { timestamps: true });

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('passwordHash')) return next();
  this.passwordHash = await bcrypt.hash(this.passwordHash, 12);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.passwordHash);
};

module.exports = mongoose.model('User', userSchema);
```

### 1.5 Create Auth Service

**Create `server/src/services/auth.service.js`:**
```javascript
const jwt = require('jsonwebtoken');
const User = require('../models/User');

class AuthService {
  generateAccessToken(userId, role) {
    return jwt.sign(
      { userId, role },
      process.env.JWT_ACCESS_SECRET,
      { expiresIn: '15m' }
    );
  }

  generateRefreshToken(userId) {
    return jwt.sign(
      { userId },
      process.env.JWT_REFRESH_SECRET,
      { expiresIn: '7d' }
    );
  }

  async register(email, password, displayName, age) {
    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      throw new Error('Email already registered');
    }

    // Create user
    const user = await User.create({
      email,
      passwordHash: password,
      profile: { displayName, age }
    });

    // Generate tokens
    const accessToken = this.generateAccessToken(user._id, user.role);
    const refreshToken = this.generateRefreshToken(user._id);

    return { user, accessToken, refreshToken };
  }

  async login(email, password) {
    const user = await User.findOne({ email });
    if (!user || !(await user.comparePassword(password))) {
      throw new Error('Invalid email or password');
    }

    const accessToken = this.generateAccessToken(user._id, user.role);
    const refreshToken = this.generateRefreshToken(user._id);

    return { user, accessToken, refreshToken };
  }
}

module.exports = new AuthService();
```

### 1.6 Create Auth Routes

**Create `server/src/routes/auth.routes.js`:**
```javascript
const express = require('express');
const router = express.Router();
const authService = require('../services/auth.service');

// Register
router.post('/register', async (req, res, next) => {
  try {
    const { email, password, displayName, age } = req.body;
    const { user, accessToken, refreshToken } = await authService.register(
      email, password, displayName, age
    );

    // Set refresh token as httpOnly cookie
    res.cookie('refreshToken', refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000 // 7 days
    });

    res.status(201).json({
      success: true,
      data: {
        user: {
          id: user._id,
          email: user.email,
          role: user.role,
          profile: user.profile
        },
        accessToken
      }
    });
  } catch (error) {
    next(error);
  }
});

// Login
router.post('/login', async (req, res, next) => {
  try {
    const { email, password } = req.body;
    const { user, accessToken, refreshToken } = await authService.login(email, password);

    res.cookie('refreshToken', refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000
    });

    res.json({
      success: true,
      data: {
        user: {
          id: user._id,
          email: user.email,
          role: user.role,
          profile: user.profile
        },
        accessToken
      }
    });
  } catch (error) {
    next(error);
  }
});

// Logout
router.post('/logout', (req, res) => {
  res.clearCookie('refreshToken');
  res.json({ success: true, message: 'Logged out successfully' });
});

module.exports = router;
```

**Create `server/src/routes/index.js`:**
```javascript
const express = require('express');
const router = express.Router();

router.use('/auth', require('./auth.routes'));

module.exports = router;
```

### 1.7 Test Backend Locally

```bash
cd server
pnpm dev
```

Test with Thunder Client or curl:
```bash
# Health check
curl http://localhost:5000/health

# Register
curl -X POST http://localhost:5000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"Test123!","displayName":"Test User","age":12}'
```

### 1.8 Set Up Frontend (Student Portal)

```bash
cd apps/client
pnpm create vite . --template react
pnpm install
pnpm add axios react-router-dom zustand @tanstack/react-query
```

**Update `vite.config.js`:**
```javascript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:5000',
        changeOrigin: true
      }
    }
  }
})
```

### 1.9 Create Auth Store (Zustand)

**Create `apps/client/src/stores/authStore.js`:**
```javascript
import { create } from 'zustand';
import axios from 'axios';

const api = axios.create({
  baseURL: '/api/v1',
  withCredentials: true
});

export const useAuthStore = create((set, get) => ({
  user: null,
  accessToken: null,
  isLoading: false,
  error: null,

  login: async (email, password) => {
    set({ isLoading: true, error: null });
    try {
      const { data } = await api.post('/auth/login', { email, password });
      set({ 
        user: data.data.user, 
        accessToken: data.data.accessToken,
        isLoading: false 
      });
      return data.data;
    } catch (error) {
      set({ error: error.response?.data?.error?.message || 'Login failed', isLoading: false });
      throw error;
    }
  },

  register: async (email, password, displayName, age) => {
    set({ isLoading: true, error: null });
    try {
      const { data } = await api.post('/auth/register', { 
        email, password, displayName, age 
      });
      set({ 
        user: data.data.user, 
        accessToken: data.data.accessToken,
        isLoading: false 
      });
      return data.data;
    } catch (error) {
      set({ error: error.response?.data?.error?.message || 'Registration failed', isLoading: false });
      throw error;
    }
  },

  logout: async () => {
    await api.post('/auth/logout');
    set({ user: null, accessToken: null });
  }
}));
```

### 1.10 Deploy to Production (First Time)

#### Deploy Backend to Render

1. Go to render.com → New → Web Service
2. Connect your GitHub repo
3. Settings:
   - **Name**: zestcoder-api
   - **Region**: Singapore (closest to India)
   - **Branch**: main
   - **Root Directory**: server
   - **Build Command**: `pnpm install`
   - **Start Command**: `node src/index.js`
4. Environment Variables (add all from .env)
5. Click Deploy

#### Deploy Frontend to Vercel

1. Go to vercel.com → New Project
2. Import GitHub repo
3. Settings:
   - **Root Directory**: apps/client
   - **Framework**: Vite
4. Environment Variables:
   - `VITE_API_URL=https://zestcoder-api.onrender.com`
5. Deploy

---

## 📚 Learning Checkpoint #1

Before Phase 2, learn these concepts (1-2 days):

### JWT Authentication
- **Video**: "JWT Authentication Tutorial" by Web Dev Simplified (YouTube, 20 min)
- **Read**: jwt.io/introduction

### React Query
- **Video**: "React Query Tutorial" by Codevolution (YouTube, 1 hour)
- **Docs**: tanstack.com/query

### MongoDB Indexes
- **Course**: MongoDB University M001 (free, 2 hours)

---

## 🔧 Phase 2: Admin Panel (Week 3-4)

### 2.1 Create All MongoDB Models

**Create `server/src/models/Language.js`:**
```javascript
const mongoose = require('mongoose');

const languageSchema = new mongoose.Schema({
  name: { type: String, required: true },
  slug: { type: String, required: true, unique: true },
  icon: String,
  color: String,
  description: String,
  shortDescription: String,
  executorConfig: {
    type: { type: String, enum: ['piston', 'browser', 'sandbox'] },
    runtime: String,
    version: String,
    timeout: { type: Number, default: 10000 },
    memoryLimit: { type: Number, default: 134217728 }
  },
  editorConfig: {
    defaultCode: String,
    theme: { type: String, default: 'vs-dark' },
    language: String,
    tabSize: { type: Number, default: 2 }
  },
  displayOrder: { type: Number, default: 0 },
  isActive: { type: Boolean, default: true }
}, { timestamps: true });

languageSchema.index({ slug: 1 }, { unique: true });
languageSchema.index({ displayOrder: 1, isActive: 1 });

module.exports = mongoose.model('Language', languageSchema);
```

**Create `server/src/models/Topic.js`:**
```javascript
const mongoose = require('mongoose');

const topicSchema = new mongoose.Schema({
  languageId: { 
    type: mongoose.Schema.Types.ObjectId, 
    ref: 'Language', 
    required: true 
  },
  title: { type: String, required: true },
  slug: { type: String, required: true },
  description: String,
  icon: String,
  displayOrder: { type: Number, default: 0 },
  metadata: {
    difficulty: { type: Number, min: 1, max: 5, default: 1 },
    estimatedMinutes: Number,
    lessonCount: { type: Number, default: 0 }
  },
  isActive: { type: Boolean, default: true }
}, { timestamps: true });

topicSchema.index({ languageId: 1, displayOrder: 1 });
topicSchema.index({ languageId: 1, slug: 1 }, { unique: true });

module.exports = mongoose.model('Topic', topicSchema);
```

**Create `server/src/models/Lesson.js`:**
```javascript
const mongoose = require('mongoose');

const lessonSchema = new mongoose.Schema({
  topicId: { 
    type: mongoose.Schema.Types.ObjectId, 
    ref: 'Topic', 
    required: true 
  },
  languageId: { 
    type: mongoose.Schema.Types.ObjectId, 
    ref: 'Language', 
    required: true 
  },
  title: { type: String, required: true },
  slug: { type: String, required: true },
  type: {
    type: String,
    enum: ['tutorial', 'interactive', 'code_playground', 'code_challenge', 'mcq', 'fill_blank', 'debug'],
    required: true
  },
  displayOrder: { type: Number, default: 0 },
  difficulty: { type: Number, min: 1, max: 5, default: 1 },
  points: { type: Number, default: 10 },
  content: {
    instruction: String,
    theory: String,
    hints: [String],
    starterCode: String,
    solutionCode: String,
    interactiveConfig: mongoose.Schema.Types.Mixed,
    mcqConfig: {
      question: String,
      options: [{
        id: String,
        text: String,
        isCorrect: Boolean
      }],
      explanation: String
    },
    fillBlankConfig: {
      codeTemplate: String,
      blanks: [{
        id: String,
        answer: String,
        alternateAnswers: [String],
        hint: String
      }]
    }
  },
  validation: {
    type: { type: String, enum: ['output', 'test_cases', 'visual', 'manual'] },
    expectedOutput: String,
    testCases: [{
      input: String,
      expectedOutput: String,
      isHidden: Boolean
    }]
  },
  isActive: { type: Boolean, default: true }
}, { timestamps: true });

lessonSchema.index({ topicId: 1, displayOrder: 1 });
lessonSchema.index({ topicId: 1, slug: 1 }, { unique: true });

module.exports = mongoose.model('Lesson', lessonSchema);
```

### 2.2 Create Admin Middleware

**Create `server/src/middleware/auth.js`:**
```javascript
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.authenticate = async (req, res, next) => {
  try {
    const authHeader = req.headers.authorization;
    if (!authHeader?.startsWith('Bearer ')) {
      return res.status(401).json({ success: false, error: { message: 'No token provided' } });
    }

    const token = authHeader.split(' ')[1];
    const decoded = jwt.verify(token, process.env.JWT_ACCESS_SECRET);
    
    req.user = await User.findById(decoded.userId).select('-passwordHash');
    if (!req.user) {
      return res.status(401).json({ success: false, error: { message: 'User not found' } });
    }

    next();
  } catch (error) {
    return res.status(401).json({ success: false, error: { message: 'Invalid token' } });
  }
};

exports.requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ success: false, error: { message: 'Admin access required' } });
  }
  next();
};
```

### 2.3 Create Admin CRUD Routes

**Create `server/src/routes/admin/language.routes.js`:**
```javascript
const express = require('express');
const router = express.Router();
const Language = require('../../models/Language');
const { authenticate, requireAdmin } = require('../../middleware/auth');

// Apply auth to all routes
router.use(authenticate, requireAdmin);

// GET all languages
router.get('/', async (req, res, next) => {
  try {
    const languages = await Language.find().sort('displayOrder');
    res.json({ success: true, data: languages });
  } catch (error) {
    next(error);
  }
});

// POST create language
router.post('/', async (req, res, next) => {
  try {
    const language = await Language.create(req.body);
    res.status(201).json({ success: true, data: language });
  } catch (error) {
    next(error);
  }
});

// PUT update language
router.put('/:id', async (req, res, next) => {
  try {
    const language = await Language.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    );
    if (!language) {
      return res.status(404).json({ success: false, error: { message: 'Language not found' } });
    }
    res.json({ success: true, data: language });
  } catch (error) {
    next(error);
  }
});

// DELETE language (soft delete)
router.delete('/:id', async (req, res, next) => {
  try {
    const language = await Language.findByIdAndUpdate(
      req.params.id,
      { isActive: false },
      { new: true }
    );
    res.json({ success: true, data: language });
  } catch (error) {
    next(error);
  }
});

module.exports = router;
```

Create similar routes for Topics and Lessons following the same pattern.

### 2.4 Set Up Admin Frontend

```bash
cd apps/admin
pnpm create vite . --template react
pnpm install
pnpm add axios react-router-dom zustand @tanstack/react-query
pnpm add @monaco-editor/react react-markdown
```

### 2.5 Build Admin Dashboard UI

Key pages to build:
1. **Login Page** - Admin-only login
2. **Dashboard** - Stats overview
3. **Languages List** - Table with CRUD
4. **Language Form** - Create/Edit form
5. **Topics List** - Nested under languages
6. **Lessons List** - Nested under topics
7. **Lesson Editor** - Monaco editor + preview

> **AI Prompt for building UI:**
> "Create a React admin dashboard component with a sidebar navigation, header with logout button, and main content area. Use CSS modules for styling. Include routes for Dashboard, Languages, Topics, Lessons, and Users."

---

## ⚡ Phase 3: Code Execution (Week 5-6)

### 3.1 Integrate Piston API

**Create `server/src/services/codeExecution.service.js`:**
```javascript
const axios = require('axios');

class CodeExecutionService {
  constructor() {
    this.pistonUrl = 'https://emkc.org/api/v2/piston';
  }

  async execute(language, code, input = '') {
    try {
      // Get runtime info
      const runtimeMap = {
        'javascript': { language: 'javascript', version: '18.15.0' },
        'python': { language: 'python', version: '3.10.0' },
        'html': { type: 'browser' }, // Handle client-side
        'css': { type: 'browser' }
      };

      const runtime = runtimeMap[language.toLowerCase()];
      
      if (!runtime) {
        throw new Error(`Unsupported language: ${language}`);
      }

      if (runtime.type === 'browser') {
        return { type: 'browser', code };
      }

      const response = await axios.post(`${this.pistonUrl}/execute`, {
        language: runtime.language,
        version: runtime.version,
        files: [{ content: code }],
        stdin: input,
        run_timeout: 10000
      });

      return {
        output: response.data.run.stdout,
        error: response.data.run.stderr,
        exitCode: response.data.run.code,
        executionTime: response.data.run.time
      };
    } catch (error) {
      throw new Error(`Code execution failed: ${error.message}`);
    }
  }
}

module.exports = new CodeExecutionService();
```

**Create `server/src/routes/code.routes.js`:**
```javascript
const express = require('express');
const router = express.Router();
const codeExecutionService = require('../services/codeExecution.service');
const { authenticate } = require('../middleware/auth');
const rateLimit = require('express-rate-limit');

// Rate limit code execution
const codeRateLimit = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 30, // 30 requests per minute
  message: { success: false, error: { message: 'Too many requests, slow down!' } }
});

router.use(authenticate);
router.use(codeRateLimit);

router.post('/execute', async (req, res, next) => {
  try {
    const { language, code, input } = req.body;
    
    if (!language || !code) {
      return res.status(400).json({ 
        success: false, 
        error: { message: 'Language and code are required' } 
      });
    }

    if (code.length > 50000) {
      return res.status(400).json({ 
        success: false, 
        error: { message: 'Code too long (max 50KB)' } 
      });
    }

    const result = await codeExecutionService.execute(language, code, input);
    res.json({ success: true, data: result });
  } catch (error) {
    next(error);
  }
});

module.exports = router;
```

### 3.2 Build Code Editor Component

**Create `apps/client/src/features/codeEditor/Editor.jsx`:**
```jsx
import { useState, useRef } from 'react';
import MonacoEditor from '@monaco-editor/react';
import axios from 'axios';
import styles from './Editor.module.css';

export function CodeEditor({ language, starterCode, onSubmit }) {
  const [code, setCode] = useState(starterCode || '');
  const [output, setOutput] = useState('');
  const [isRunning, setIsRunning] = useState(false);
  const editorRef = useRef(null);

  const handleRun = async () => {
    setIsRunning(true);
    setOutput('Running...');

    try {
      const { data } = await axios.post('/api/v1/code/execute', {
        language,
        code
      });

      if (data.data.error) {
        setOutput(`Error:\n${data.data.error}`);
      } else {
        setOutput(data.data.output || 'No output');
      }
    } catch (error) {
      setOutput(`Error: ${error.response?.data?.error?.message || error.message}`);
    } finally {
      setIsRunning(false);
    }
  };

  return (
    <div className={styles.container}>
      <div className={styles.editorPane}>
        <MonacoEditor
          height="400px"
          language={language}
          value={code}
          onChange={setCode}
          theme="vs-dark"
          options={{
            minimap: { enabled: false },
            fontSize: 14,
            tabSize: 2
          }}
          onMount={(editor) => { editorRef.current = editor; }}
        />
      </div>
      
      <div className={styles.controls}>
        <button onClick={handleRun} disabled={isRunning}>
          {isRunning ? '⏳ Running...' : '▶️ Run Code'}
        </button>
        {onSubmit && (
          <button onClick={() => onSubmit(code)} className={styles.submitBtn}>
            ✅ Submit
          </button>
        )}
      </div>
      
      <div className={styles.outputPane}>
        <h4>Output:</h4>
        <pre>{output}</pre>
      </div>
    </div>
  );
}
```

---

## 🎮 Phase 4: Student Portal (Week 7-8)

### 4.1 Build Main Pages

Create these pages in `apps/client/src/pages/`:

1. **Home.jsx** - Welcome + language cards
2. **Languages.jsx** - Grid of available languages
3. **Topics.jsx** - Topics for selected language
4. **Lessons.jsx** - Lessons in topic
5. **LessonView.jsx** - Actual lesson with editor

### 4.2 Create Progress Tracking

**Create `server/src/models/UserProgress.js`:**
```javascript
const mongoose = require('mongoose');

const userProgressSchema = new mongoose.Schema({
  userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  lessonId: { type: mongoose.Schema.Types.ObjectId, ref: 'Lesson', required: true },
  status: { 
    type: String, 
    enum: ['not_started', 'in_progress', 'completed'], 
    default: 'not_started' 
  },
  score: { type: Number, default: 0 },
  attempts: { type: Number, default: 0 },
  completedAt: Date,
  bestCode: String
}, { timestamps: true });

userProgressSchema.index({ userId: 1, lessonId: 1 }, { unique: true });

module.exports = mongoose.model('UserProgress', userProgressSchema);
```

### 4.3 Add Interactive Lesson Types

Create components for each lesson type:
- `TutorialLesson.jsx` - Read + understand
- `MCQLesson.jsx` - Multiple choice
- `FillBlankLesson.jsx` - Fill in code
- `CodeChallenge.jsx` - Full coding
- `InteractiveLesson.jsx` - Visual (like Flexbox Froggy)

---

## 📚 Learning Checkpoint #2

Before Phase 5, learn these concepts:

### WebSockets (for real-time features)
- **Video**: "WebSockets in 100 Seconds" by Fireship (YouTube)

### CSS Animations
- **Course**: "CSS Animation" on web.dev (free)

### Testing
- **Video**: "Jest Tutorial" by Traversy Media (YouTube)

---

## ✨ Phase 5: Polish & Features (Week 9-10)

### 5.1 Add Gamification

- Points system
- Streak counter
- Achievement badges
- Progress bars
- Celebration animations (confetti on completion)

### 5.2 Child-Friendly UI

- Large buttons (min 48px)
- Bright colors
- Fun animations
- Sound effects (optional)
- Avatar customization

### 5.3 Add Hints System

Progressive hints that unlock after failed attempts.

---

## 🧪 Phase 6: Testing & Optimization (Week 11-12)

### 6.1 Write Tests

```bash
cd server
pnpm add -D jest supertest
```

**Create `server/tests/auth.test.js`:**
```javascript
const request = require('supertest');
const app = require('../src/app');

describe('Auth Endpoints', () => {
  it('should register a new user', async () => {
    const res = await request(app)
      .post('/api/v1/auth/register')
      .send({
        email: 'test@example.com',
        password: 'Test123!',
        displayName: 'Test User',
        age: 12
      });
    expect(res.statusCode).toBe(201);
    expect(res.body.success).toBe(true);
  });
});
```

### 6.2 Performance Optimization

- Add Redis caching for languages/topics
- Lazy load components
- Optimize images (WebP)
- Add loading skeletons

### 6.3 Security Audit

- [ ] All passwords hashed
- [ ] Rate limiting active
- [ ] Input validation everywhere
- [ ] CORS configured correctly
- [ ] No secrets in code

---

## 🚀 Phase 7: Launch (Week 13-14)

### 7.1 Pre-Launch Checklist

- [ ] All tests passing
- [ ] Mobile responsive
- [ ] Error pages (404, 500)
- [ ] Loading states
- [ ] Analytics added (Google Analytics)
- [ ] Error tracking (Sentry)

### 7.2 Create Seed Data

Create 1 complete language with:
- 3 topics
- 5 lessons per topic
- Mix of lesson types

### 7.3 Final Deployment

1. Merge all changes to main
2. Verify Render auto-deploy
3. Verify Vercel auto-deploy
4. Test production URLs
5. Share with beta testers

---

## 🛠️ AI Tool Usage Guide

### For Each Phase, Tell Your AI Assistant:

**Phase 1 (Foundation):**
> "I'm building a children's coding learning platform with MERN stack. Help me set up Express server with JWT authentication, MongoDB connection, and User model. Follow REST API best practices."

**Phase 2 (Admin):**
> "Create CRUD API routes for Languages, Topics, and Lessons collections. Include admin-only middleware. Return consistent JSON responses with success/error format."

**Phase 3 (Code Execution):**
> "Integrate Piston API for code execution. Create a service that takes language and code, sends to Piston, and returns formatted output. Add rate limiting."

**Phase 4 (Student Portal):**
> "Build React pages for browsing languages, topics, and lessons. Include Monaco code editor with run button. Track user progress in MongoDB."

**Phase 5 (Polish):**
> "Add gamification features: points, streaks, achievements. Create child-friendly UI with large buttons, bright colors, and celebration animations."

---

## ⚠️ Common Issues & Solutions

| Issue | Solution |
|-------|----------|
| CORS errors | Check CLIENT_URL in backend .env matches frontend URL |
| MongoDB connection fails | Check IP whitelist in MongoDB Atlas (add 0.0.0.0/0 for dev) |
| JWT expired immediately | Check system time is correct, use longer expiry for testing |
| Render cold starts slow | Add a health check ping from UptimeRobot every 14 minutes |
| Piston API fails | Check if language runtime is correct, try fallback |
| Code not running | Check code size limit, timeout settings |

---

## 📞 Getting Help

1. **Stack Overflow** - Tag with `mern`, `mongodb`, `react`
2. **Discord** - Join Reactiflux, MongoDB, or Node.js servers
3. **GitHub Issues** - Check if others faced same issue
4. **AI Tools** - Paste error message and ask for explanation

---

## 🎯 Success Milestones

| Week | Milestone | How to Verify |
|------|-----------|---------------|
| 2 | Auth working | Can register/login, see user in MongoDB |
| 4 | Admin panel ready | Can create language → topic → lesson |
| 6 | Code runs | Can write Python/JS and see output |
| 8 | Full lesson flow | Student can complete a lesson |
| 10 | Polished UI | Looks professional, works on mobile |
| 12 | Production ready | All tests pass, no console errors |

---

**You've got this! 🚀**

Start with Phase 1, complete each step, and don't move forward until the current phase works. Use AI tools to speed up coding, but understand what each piece does.

*Built for ZestCoder Learning Platform*
