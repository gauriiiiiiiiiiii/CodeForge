# CodeForge - Complete Project Structure & Code Overview

## Project Overview
**CodeForge** is a full-stack web application for writing, executing, and sharing code snippets across 10+ programming languages. It combines Monaco Editor for code editing with the Piston API for code execution, Convex for real-time database, and Clerk for authentication.

---

## 📂 Complete Directory Structure

```
CodeForge/
├── package.json                    # 180 dependencies, build scripts
├── tsconfig.json                   # TypeScript strict mode config
├── next.config.ts                  # Next.js 15 configuration
├── tailwind.config.ts              # Tailwind CSS v3 dark theme
├── postcss.config.mjs              # PostCSS pipeline setup
├── next-env.d.ts                   # Auto-generated Next.js types
│
├── public/                         # Static language logo assets
│   ├── javascript.png
│   ├── typescript.png
│   ├── python.png
│   └── [9 more language logos]
│
├── src/                            # Frontend application
│   ├── app/                        # Next.js 15 App Router
│   │   ├── layout.tsx             # Root layout with Clerk + Convex providers
│   │   ├── globals.css            # Tailwind CSS global styles
│   │   │
│   │   ├── (root)/                # Editor/home pages (route group)
│   │   │   ├── page.tsx           # Main editor interface
│   │   │   └── _components/
│   │   │       ├── EditorPanel.tsx        # Monaco Editor wrapper
│   │   │       ├── EditorPanelSkeleton.tsx
│   │   │       ├── OutputPanel.tsx        # Code execution output display
│   │   │       ├── RunButton.tsx          # Code execution trigger
│   │   │       ├── RunningCodeSkeleton.tsx
│   │   │       ├── Header.tsx             # Navigation + settings
│   │   │       ├── HeaderProfileBtn.tsx   # User profile button
│   │   │       ├── LanguageSelector.tsx   # Language switcher (10+ langs)
│   │   │       ├── ThemeSelector.tsx      # Editor theme toggle
│   │   │       └── ShareSnippetDialog.tsx # Save snippet modal
│   │   │   └── _constants/
│   │   │       └── index.ts        # Language configs, Piston runtimes
│   │   │
│   │   ├── snippets/               # Community code library page
│   │   │   ├── page.tsx            # Snippets feed with filters
│   │   │   └── _components/
│   │   │       ├── SnippetCard.tsx       # Individual snippet preview
│   │   │       └── SnippetsPageSkeleton.tsx
│   │   │   └── [id]/               # Dynamic route for snippet details
│   │   │       ├── page.tsx        # Full snippet view + comments
│   │   │       └── _components/
│   │   │           ├── CodeBlock.tsx            # Code display
│   │   │           ├── Comments.tsx            # Comments thread
│   │   │           ├── Comment.tsx             # Individual comment
│   │   │           ├── CommentContent.tsx      # Rich comment editor
│   │   │           ├── CommentForm.tsx         # Comment input form
│   │   │           ├── CopyButton.tsx          # Copy code button
│   │   │           └── SnippetLoadingSkeleton.tsx
│   │   │
│   │   ├── profile/                # User profile page
│   │   │   ├── page.tsx            # Executions + starred snippets tabs
│   │   │   └── _components/
│   │   │       ├── ProfileHeader.tsx
│   │   │       ├── ProfileHeaderSkeleton.tsx
│   │   │       └── CodeBlock.tsx
│   │   │
│   │   ├── pricing/                # Pro subscription page
│   │   │   ├── page.tsx            # Pricing tiers + upgrade
│   │   │   └── _components/
│   │   │       ├── ProPlanView.tsx
│   │   │       ├── FeatureCategory.tsx
│   │   │       ├── FeatureItem.tsx
│   │   │       └── UpgradeButton.tsx
│   │   │
│   │   └── fonts/                  # Local font files
│   │       ├── GeistVF.woff
│   │       └── GeistMonoVF.woff
│   │
│   ├── components/                 # Shared components
│   │   ├── Footer.tsx              # Copyright + links
│   │   ├── LoginButton.tsx         # Clerk auth button
│   │   ├── NavigationHeader.tsx    # Global navbar
│   │   ├── StarButton.tsx          # Favorite toggle
│   │   └── providers/
│   │       └── ConvexClientProvider.tsx  # Convex + Clerk provider setup
│   │
│   ├── hooks/                      # Custom React hooks
│   │   └── useMounted.tsx          # Hydration safety hook
│   │
│   ├── middleware.ts               # Clerk authentication middleware
│   │
│   ├── store/                      # State management
│   │   └── useCodeEditorStore.ts   # Zustand editor state store
│   │
│   └── types/                      # TypeScript interfaces
│       └── index.ts                # All type definitions
│
├── convex/                         # Backend (Convex serverless DB)
│   ├── schema.ts                   # Database tables & indexes
│   ├── auth.config.ts              # Clerk authentication config
│   │
│   ├── users.ts                    # User mutations & queries
│   ├── snippets.ts                 # Snippet CRUD + comments
│   ├── codeExecutions.ts           # Execution history & stats
│   ├── http.ts                     # HTTP webhooks
│   ├── lemonSqueezy.ts             # Payment webhook verification
│   │
│   └── _generated/                 # Auto-generated Convex types
│       ├── api.d.ts
│       ├── api.js
│       ├── dataModel.d.ts
│       └── server.d.ts
│
└── STUDY/                          # Documentation (this folder)
    ├── 01_PROJECT_STRUCTURE.md     # This file
    ├── 02_BUILD_PROCESS.md
    ├── 03_INTERVIEW_DEEP_DIVE.md
    └── README.md
```

---

## 🔑 Key Files & Their Purpose

### Root Configuration

| File | Purpose |
|------|---------|
| **package.json** | Dependencies, build scripts, project metadata |
| **tsconfig.json** | TypeScript strict mode, path aliases (@/*) |
| **next.config.ts** | Next.js 15 configuration (API routes, redirects) |
| **tailwind.config.ts** | Dark theme colors, custom utilities |
| **postcss.config.mjs** | CSS processing pipeline |

### Frontend Entry Points

| File | What It Does |
|------|--------------|
| **src/app/layout.tsx** | Root wrapper: Clerk provider → Convex provider → Child routes |
| **src/app/(root)/page.tsx** | Main editor interface, 2-column layout |
| **src/app/snippets/page.tsx** | Community snippets feed with search/filters |
| **src/app/snippets/[id]/page.tsx** | Full snippet view with comments |
| **src/app/profile/page.tsx** | User's execution history + starred snippets |
| **src/app/pricing/page.tsx** | Pro subscription page with Lemon Squeezy integration |

### Core Components

#### Editor Interface
```
EditorPanel.tsx
  ├─ Monaco Editor (code input)
  ├─ Language Selector
  ├─ Theme Selector
  ├─ Font Size Control
  └─ Share Button

OutputPanel.tsx
  ├─ Execution Result Display
  ├─ Error Messages
  └─ Copy Output Button

RunButton.tsx
  └─ Triggers code execution via Piston API

Header.tsx
  ├─ NavigationHeader (navbar)
  ├─ HeaderProfileBtn (user menu)
  ├─ LanguageSelector
  └─ ThemeSelector

ShareSnippetDialog.tsx
  └─ Modal to save snippet to Convex
```

#### Page Components
```
SnippetCard.tsx
  ├─ Snippet preview card
  ├─ Star count
  └─ User info

Comments.tsx & Comment.tsx
  ├─ Comment thread rendering
  ├─ Author info
  └─ Edit/delete buttons

ProfileHeader.tsx
  ├─ User stats (executions, starred)
  ├─ Language breakdown
  └─ Pro subscription status

CodeBlock.tsx
  └─ Syntax-highlighted code display
```

### State Management

**useCodeEditorStore.ts** (Zustand)
```typescript
State:
  - language: current language (javascript, python, etc.)
  - theme: editor theme (vs-dark, etc.)
  - fontSize: editor font size (12-24)
  - output: execution result
  - error: execution error message
  - editor: Monaco Editor instance
  - executionResult: { code, output, error }
  - isRunning: execution in progress flag

Actions:
  - setLanguage(lang) → switch language + persist to localStorage
  - setTheme(theme) → change editor theme
  - setFontSize(size) → adjust font size
  - runCode() → execute code via Piston API
  - setEditor(editor) → initialize editor instance
  - getCode() → get current editor content
```

**Persistence**: Uses localStorage to save:
- `editor-code-{language}` → code for each language
- `editor-language` → currently selected language
- `editor-theme` → current theme
- `editor-font-size` → font size preference

### Backend (Convex)

#### Database Schema (convex/schema.ts)

```typescript
users table:
  - userId (string, indexed)    // Clerk user ID
  - email (string)
  - name (string)
  - isPro (boolean)             // Pro subscription flag
  - proSince (optional number)  // Timestamp of pro upgrade
  - lemonSqueezyCustomerId (optional)
  - lemonSqueezyOrderId (optional)

snippets table:
  - userId (string, indexed)    // Owner ID
  - userName (string)           // Owner name (denormalized)
  - title (string)
  - language (string)
  - code (string)
  - _creationTime (auto)

snippetComments table:
  - snippetId (id, indexed)     // Reference to snippet
  - userId (string)             // Comment author
  - userName (string)           // Author name (denormalized)
  - content (string)            // HTML content

codeExecutions table:
  - userId (string, indexed)    // Who ran it
  - language (string)
  - code (string)
  - output (optional string)
  - error (optional string)
  - _creationTime (auto)

stars table:
  - userId (string, indexed)
  - snippetId (id, indexed)
  - Compound index: (userId, snippetId) for duplicate prevention
```

#### Backend Functions

**users.ts**
```typescript
syncUser(userId, email, name)
  → Create user on first Clerk login

getUser(userId)
  → Fetch user profile by ID

upgradeToPro(email, customerId, orderId, amount)
  → Mark user as pro (Lemon Squeezy callback)
```

**snippets.ts**
```typescript
createSnippet(title, language, code)
  → Save code snippet to community library

deleteSnippet(snippetId)
  → Owner-only deletion (cascade delete comments & stars)

starSnippet(snippetId)
  → Toggle star (add/remove from stars table)

addComment(snippetId, content)
  → Add comment to snippet thread

deleteComment(commentId)
  → Author-only deletion

getSnippets()
  → Fetch all snippets (paginated)

getSnippetById(snippetId)
  → Fetch single snippet with full details

getComments(snippetId)
  → Fetch comments for snippet

isSnippetStarred(snippetId)
  → Check if current user starred snippet

getSnippetStarCount(snippetId)
  → Get star count for snippet
```

**codeExecutions.ts**
```typescript
saveExecution(language, code, output, error)
  → Save execution to history (pro check for non-JS)

getUserExecutions(userId, paginationOpts)
  → Fetch user's execution history (paginated)

getUserStats(userId)
  → Calculate user stats:
    * totalExecutions
    * favoriteLanguage
    * mostStarredLanguage
    * languageBreakdown
    * last24Hours
```

**http.ts**
```
POST /clerk-webhook
  → Listen for user.created events
  → Sync new user to Convex DB

POST /lemon-squeezy-webhook
  → Listen for order_created events
  → Upgrade user to pro
```

**lemonSqueezy.ts**
```typescript
verifyWebhook(payload, signature)
  → HMAC-SHA256 verification of payment webhook
  → Prevent unauthorized webhook manipulation
```

### Middleware & Providers

**src/middleware.ts**
- Clerk authentication middleware
- Protects routes based on auth state
- Graceful fallback if Clerk not configured

**src/components/providers/ConvexClientProvider.tsx**
- ClerkProvider wrapper (auth)
- ConvexProviderWithClerk (database + auth integration)
- Falls back to ConvexProvider if Clerk not configured
- Handles hydration with useAuth hook

### Hooks

**useMounted.tsx**
```typescript
// Returns true after component mounts on client
// Prevents hydration mismatches between server & client
// Used in EditorPanel to delay Monaco Editor render
```

### Constants

**src/app/(root)/_constants/index.ts** (436 lines)
```typescript
LANGUAGE_CONFIG object:
  - 10+ languages configured:
    * javascript, typescript, python, java, c, cpp, 
      rust, go, ruby, php, csharp, kotlin, etc.
  
  Each language has:
    - id, label, logoPath
    - pistonRuntime: { language, version }
      → Specifies which runtime version Piston API uses
    - monacoLanguage: language ID for Monaco Editor
    - defaultCode: starter template for that language

defineMonacoThemes() function:
  → Registers custom editor themes in Monaco
  → Defines colors for syntax highlighting
  → Sets up dark mode theme variables
```

### Types

**src/types/index.ts**
```typescript
Theme { id, label, color }
Language { id, label, logoPath, monacoLanguage, defaultCode, pistonRuntime }
LanguageRuntime { language, version }
ExecuteCodeResponse { compile?, run? }
ExecutionResult { code, output, error }
CodeEditorState { language, output, isRunning, error, theme, fontSize, editor, ... }
Snippet { _id, _creationTime, userId, language, code, title, userName }
```

---

## 🔄 Data Flow Architecture

### 1. Code Execution Flow
```
User writes code in EditorPanel
  ↓
User clicks RunButton
  ↓
useCodeEditorStore.runCode()
  ↓
Fetch to https://emkc.org/api/v2/piston/execute
  ├─ language: "javascript"
  ├─ version: "18.15.0"
  └─ files: [{ content: userCode }]
  ↓
Piston API executes code
  ↓
Response: { compile?, run? { output, stderr, code } }
  ↓
Store result in Zustand (output/error state)
  ↓
Render in OutputPanel
  ↓
If user logged in:
  SaveExecution mutation → Convex DB
```

### 2. Snippet Sharing Flow
```
User clicks Share Button
  ↓
ShareSnippetDialog opens
  ↓
User enters title, selects language
  ↓
createSnippet mutation (convex/snippets.ts)
  ├─ Verify auth via ctx.auth.getUserIdentity()
  ├─ Fetch user from Convex
  └─ Insert to snippets table
  ↓
Snippet saved with userId + userName
  ↓
Redirect to snippets feed
```

### 3. Community Interaction Flow
```
User visits /snippets
  ↓
getSnippets query fetches all snippets
  ├─ Rendered as SnippetCards
  └─ Search/filter on client side
  ↓
User clicks snippet
  ↓
Navigate to /snippets/[id]
  ↓
getSnippetById query + getComments query
  ↓
Display code + comment thread
  ↓
User can:
  ├─ Star snippet → starSnippet mutation
  ├─ Add comment → addComment mutation
  └─ Delete own comment → deleteComment mutation
```

### 4. Authentication Flow
```
User lands on /
  ↓
Middleware checks Clerk auth
  ↓
If not authenticated:
  ├─ Show LoginButton
  └─ Can use editor but can't share/star
  ↓
If authenticated:
  ├─ Clerk webhook triggers
  ├─ HTTP POST to /clerk-webhook
  ├─ syncUser mutation creates user in Convex
  └─ User can share + interact
```

### 5. Payment Flow
```
User clicks "Upgrade to Pro" on /pricing
  ↓
Redirect to Lemon Squeezy checkout
  ↓
User completes payment
  ↓
Lemon Squeezy sends order_created webhook
  ↓
HTTP POST to /lemon-squeezy-webhook
  ↓
verifyWebhook validates HMAC signature
  ↓
upgradeToPro mutation:
  ├─ Find user by email
  ├─ Set isPro = true
  ├─ Set proSince = Date.now()
  └─ Store Lemon Squeezy IDs
  ↓
User can now execute non-JavaScript languages
```

---

## 💾 Storage Strategy

### Client-Side (Browser)
- **localStorage**: Editor preferences, code per language
- **Zustand**: Runtime state (output, language, theme, fontSize)
- **React state**: UI state (dialogs, modals, loading states)

### Server-Side (Convex)
- **users**: Authentication + billing info
- **snippets**: User-created code snippets
- **snippetComments**: Community interaction
- **codeExecutions**: Execution history (for stats & pro limits)
- **stars**: Bookmark system (many-to-many)

### External APIs
- **Clerk**: User authentication + JWT tokens
- **Lemon Squeezy**: Payment processing
- **Piston**: Code execution (ephemeral)

---

## 🔐 Security Model

### Authentication
- Clerk handles JWT token generation
- Tokens verified by Convex middleware
- ctx.auth.getUserIdentity() checks in mutations

### Authorization
- User data scoped to userId
- Delete/edit checks: verify userId matches
- Pro features: isPro flag in users table

### Webhook Security
- Clerk webhooks: Svix signature verification
- Lemon Squeezy webhooks: HMAC-SHA256 signature
- Both use environment variable secrets

### Code Execution
- Code runs in sandboxed Piston environment
- No access to user data or database
- Timeout limits on execution time

---

## 📊 Key Metrics & Stats

- **10+ supported languages** (JavaScript, Python, Java, Rust, Go, etc.)
- **5 main database tables** with strategic indexing
- **27 React components** (reusable UI patterns)
- **180 npm dependencies** (fully managed & pinned)
- **600px Monaco Editor** for comfortable coding
- **Piston API** supports 100+ language versions

---

## 🚀 Performance Optimizations

1. **Lazy Loading**: Editor only renders after Clerk is loaded
2. **Code Splitting**: Pages loaded on-demand via Next.js
3. **Local Caching**: Code persisted per language in localStorage
4. **Pagination**: Execution history paginated (5 per page initially)
5. **Indexing**: Database queries use indexed fields (userId, snippetId)
6. **CDN**: Static assets (logos) served from public folder
7. **Debouncing**: Font size slider debounced on input

---

## 🎨 UI Component Hierarchy

```
Root Layout
├─ Clerk Provider
├─ Convex Provider
├─ Footer
├─ Toaster (notifications)
└─ Children Pages
    ├─ Home Page (/):
    │  ├─ Header (nav)
    │  ├─ EditorPanel (Monaco)
    │  └─ OutputPanel (results)
    │
    ├─ Snippets Page (/snippets):
    │  ├─ NavigationHeader
    │  ├─ Search + Filters
    │  └─ SnippetCard[] (grid/list)
    │
    ├─ Snippet Detail (/snippets/[id]):
    │  ├─ CodeBlock
    │  ├─ Comments
    │  └─ StarButton
    │
    └─ Profile Page (/profile):
       ├─ ProfileHeader (stats)
       ├─ Execution History Tab
       └─ Starred Snippets Tab
```

---

## 📝 Summary

CodeForge is a **full-stack application** with:
- **Frontend**: React 18 + Next.js 15 + TypeScript + Zustand
- **Backend**: Convex (serverless DB with real-time)
- **Auth**: Clerk (JWT-based)
- **Code Execution**: Piston API (sandboxed)
- **Payments**: Lemon Squeezy (webhooks + HMAC verification)
- **UI**: Tailwind + Framer Motion + Monaco Editor

Each component has a clear responsibility, and data flows through well-defined channels (Zustand for local state, Convex for persistent data, props for UI composition).
