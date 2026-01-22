# CodeForge - Interview Deep Dive (Senior SDE Preparation)

This document covers the key technical areas you must understand deeply for a senior-level interview.

---

## 1. PROBLEM & PRODUCT UNDERSTANDING

### 1.1 Real-World Problem
**Problem Statement**: Developers lack an integrated platform to write, execute, and share code snippets across multiple languages without context-switching between different tools.

**Who faces this problem**:
- Developers teaching/learning (need to share working examples)
- Competitive programmers (testing algorithms across languages)
- Code review participants (comparing implementations)
- Open-source contributors (documenting usage patterns)

**Why existing solutions are insufficient**:
- **Local IDEs**: Can't easily share code or see others' implementations
- **Online editors** (LeetCode, HackerRank): Closed ecosystems, can't build on top
- **Gists/Pastebin**: No execution capability
- **Collaboration tools**: Not optimized for quick code testing

**Value Proposition**:
1. **Write** code instantly without setup
2. **Run** code in 11 languages (5 FREE + 6 PRO) without installation
3. **Share** executable code with others (not just text)
4. **Discover** community solutions and implementations
5. **Monetize** via pro subscription (6 advanced languages)

**Supported Languages**:
- **FREE Tier (5)**: JavaScript, C, C++, Python, Java
- **PRO Tier (6)**: Rust, Go, C#, Kotlin, PHP, Ruby

**Success Metrics**:
- Code execution latency < 500ms
- Snippet creation takes < 30 seconds
- Community engagement (comments, stars)
- Pro conversion rate target: 5-10%

### 1.2 Key User Journeys

**Journey 1: Casual User**
```
Land on site → Write JavaScript code → Run locally → Share with friend
```

**Journey 2: Learning Developer**
```
Visit /snippets → Search Python examples → View implementation → 
Add comment → Like interesting solutions → Upgrade to pro
```

**Journey 3: Pro Developer**
```
Write Rust code → Execute → Save to history → 
Review stats → Share with team
```

---

## 2. HIGH-LEVEL ARCHITECTURE & SYSTEM DESIGN

### 2.1 Architecture Layers

```
┌─────────────────────────────────────────────────────────────┐
│                       Presentation Layer                      │
│  React Components (EditorPanel, OutputPanel, SnippetCard)   │
│  State Management (Zustand for editor state)                │
└─────────────────────────────────────────────────────────────┘
                           ↕
┌─────────────────────────────────────────────────────────────┐
│                      Middleware Layer                         │
│  Clerk Authentication (JWT tokens)                           │
│  Convex Real-time Sync (optimistic updates)                │
│  Framer Motion animations                                   │
└─────────────────────────────────────────────────────────────┘
                           ↕
┌─────────────────────────────────────────────────────────────┐
│                   Application Layer (Next.js)               │
│  Route handlers (/api/*, /snippets/*, /profile/*)          │
│  HTTP endpoint handlers (webhooks from Clerk & Lemon Squeezy)│
└─────────────────────────────────────────────────────────────┘
                           ↕
┌─────────────────────────────────────────────────────────────┐
│                    Business Logic Layer                       │
│  Convex Mutations (CRUD for snippets, comments, stars)      │
│  Authorization checks (isPro, userId matching)              │
│  Piston integration (code execution coordination)           │
└─────────────────────────────────────────────────────────────┘
                           ↕
┌─────────────────────────────────────────────────────────────┐
│                      Database Layer                           │
│  Convex (managed database with built-in auth)               │
│  Tables: users, snippets, comments, executions, stars      │
│  Indexes optimized for common queries                       │
└─────────────────────────────────────────────────────────────┘
                           ↕
┌─────────────────────────────────────────────────────────────┐
│                    External Services                          │
│  Clerk (authentication)     Piston API (code execution)      │
│  Lemon Squeezy (payments)   localStorage (preferences)      │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 System Architecture Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                    Frontend Browser                             │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │         React App (Next.js 15 App Router)              │  │
│  │                                                         │  │
│  │  EditorPanel ←→ Zustand Store ←→ OutputPanel           │  │
│  │       ↕              ↕              ↕                   │  │
│  │   Monaco Editor   Editor State  Code Results           │  │
│  │                  (language,                             │  │
│  │   localStorage   theme,       Piston API              │  │
│  │   (preferences)  fontSize)    (external)              │  │
│  └─────────────────────────────────────────────────────────┘  │
│         ↕                           ↕                           │
│    localStorage             ClerkProvider                      │
│    (code per                (JWT auth)                         │
│     language)                   ↕                              │
│                        Middleware check                        │
└────────────────────────────────────────────────────────────────┘
         ↕                           ↕
   ┌──────────────┐         ┌──────────────┐
   │   Clerk API   │         │  Piston API  │
   │ (JWT tokens)  │         │   (Execute)  │
   │ (webhooks)    │         │  REST API    │
   └──────────────┘         └──────────────┘
         ↕                           
   (user.created                     
    webhook)                         
         ↕
   ┌─────────────────────────────────────────┐
   │         Convex Backend                  │
   │  ┌───────────────────────────────────┐  │
   │  │  HTTP Handlers (/clerk-webhook)   │  │
   │  │  (Receives auth events)           │  │
   │  └───────────────────────────────────┘  │
   │         ↕                                │
   │  ┌───────────────────────────────────┐  │
   │  │  Mutations & Queries              │  │
   │  │  ├─ syncUser                      │  │
   │  │  ├─ createSnippet                 │  │
   │  │  ├─ addComment                    │  │
   │  │  ├─ saveExecution                 │  │
   │  │  └─ starSnippet                   │  │
   │  └───────────────────────────────────┘  │
   │         ↕                                │
   │  ┌───────────────────────────────────┐  │
   │  │  Database (Managed)               │  │
   │  │  users, snippets, comments        │  │
   │  │  codeExecutions, stars            │  │
   │  └───────────────────────────────────┘  │
   └─────────────────────────────────────────┘
```

### 2.3 Data Flow - Code Execution

```
┌──────────────────────────────────────────────────────────────────┐
│  User clicks "Run Code"                                          │
└──────────────────────────────────────────────────────────────────┘
                            ↓
┌──────────────────────────────────────────────────────────────────┐
│  RunButton component calls useCodeEditorStore.runCode()         │
│  Sets isRunning = true (shows "Executing...")                   │
└──────────────────────────────────────────────────────────────────┘
                            ↓
┌──────────────────────────────────────────────────────────────────┐
│  Store fetches code from Monaco Editor instance                  │
│  Reads pistonRuntime config for current language                │
│  const runtime = LANGUAGE_CONFIG[language].pistonRuntime       │
│  → { language: "python", version: "3.10.0" }                   │
└──────────────────────────────────────────────────────────────────┘
                            ↓
┌──────────────────────────────────────────────────────────────────┐
│  HTTP POST to https://emkc.org/api/v2/piston/execute           │
│                                                                  │
│  Body: {                                                         │
│    language: "python",                                          │
│    version: "3.10.0",                                           │
│    files: [{ content: userCode }]                              │
│  }                                                               │
└──────────────────────────────────────────────────────────────────┘
                            ↓
┌──────────────────────────────────────────────────────────────────┐
│  Piston API sandboxes execution                                  │
│  ├─ Compilation phase (if needed): checks for syntax errors     │
│  ├─ Runtime phase: executes code with timeout                   │
│  └─ Returns: { compile?: {...}, run?: { output, stderr } }     │
└──────────────────────────────────────────────────────────────────┘
                            ↓
┌──────────────────────────────────────────────────────────────────┐
│  Response parsing in Zustand                                     │
│  if (data.compile?.code !== 0) → compilation error              │
│  if (data.run?.code !== 0) → runtime error                      │
│  else → success, extract output                                 │
└──────────────────────────────────────────────────────────────────┘
                            ↓
┌──────────────────────────────────────────────────────────────────┐
│  Update Zustand state:                                          │
│  ├─ output: trimmed output string                               │
│  ├─ error: null (if successful)                                 │
│  ├─ isRunning: false                                            │
│  └─ executionResult: { code, output, error }                    │
└──────────────────────────────────────────────────────────────────┘
                            ↓
┌──────────────────────────────────────────────────────────────────┐
│  OutputPanel re-renders with new output                          │
│  (automatic due to Zustand subscription)                         │
└──────────────────────────────────────────────────────────────────┘
                            ↓
┌──────────────────────────────────────────────────────────────────┐
│  If user is logged in (useUser() returns user):                 │
│  Call saveExecution mutation                                     │
│  Convex stores execution in codeExecutions table                │
└──────────────────────────────────────────────────────────────────┘
```

### 2.4 Why This Architecture

**Design Decisions & Justifications**:

1. **Zustand for local state (not Redux)**
   - ✓ Minimal boilerplate
   - ✓ Direct state access (no selectors)
   - ✓ Supports async operations (runCode)
   - ✓ localStorage integration out of the box
   - ✗ Redux would be overkill for editor-only state

2. **Convex as backend (not traditional API)**
   - ✓ Real-time sync without polling
   - ✓ Built-in auth (JWT from Clerk)
   - ✓ Automatic type generation
   - ✓ Serverless (no server management)
   - ✗ GraphQL would require more setup

3. **Piston API for code execution (not in-house)**
   - ✓ Supports 100+ languages/versions
   - ✓ Proper sandboxing
   - ✓ No infrastructure cost
   - ✓ Battle-tested
   - ✗ Self-hosting would be complex

4. **Clerk for authentication (not custom)**
   - ✓ Social login (Google, GitHub)
   - ✓ JWT token management
   - ✓ Webhook support
   - ✓ RBAC ready
   - ✗ Custom auth would require security expertise

5. **Lemon Squeezy for payments (not Stripe)**
   - ✓ Simpler integration
   - ✓ Webhook verification (Svix)
   - ✓ License key support (future)
   - ✗ Stripe has more features but more complex

---

## 3. BACKEND DESIGN & BUSINESS LOGIC

### 3.1 Database Design

**Schema Design Philosophy**:
- Denormalization where reads >> writes
- Indexing on query fields
- Cascade deletes for referential integrity
- Compound indexes for uniqueness constraints

```
users (1)
  ├─ PK: _id
  ├─ FK: userId (Clerk ID)
  ├─ Fields: email, name, isPro, proSince
  ├─ Webhooks: Clerk user.created → syncUser
  └─ Relationships: 1 → many snippets, executions, comments, stars

snippets (many)
  ├─ PK: _id
  ├─ FK: userId
  ├─ Denormalized: userName (for pagination without join)
  ├─ Fields: title, language, code
  ├─ Indexes: by_user_id (for /profile)
  └─ Relationships: 1 snippet → many comments, stars

snippetComments (many)
  ├─ PK: _id
  ├─ FK: snippetId, userId
  ├─ Denormalized: userName (for thread rendering)
  ├─ Fields: content (HTML)
  ├─ Indexes: by_snippet_id (for /snippets/[id])
  └─ Lifecycle: cascade delete with snippet

codeExecutions (many)
  ├─ PK: _id
  ├─ FK: userId
  ├─ Fields: language, code, output, error
  ├─ Indexes: by_user_id (for pagination & stats)
  └─ Use case: analytics, pro limit enforcement

stars (many)
  ├─ PK: _id
  ├─ FK: userId, snippetId
  ├─ Compound index: (userId, snippetId) → prevents duplicates
  ├─ Simple indexes: by_user_id, by_snippet_id
  └─ Use case: bookmark system, star count queries
```

**Query Patterns**:

```typescript
// Profile page execution history
SELECT * FROM codeExecutions 
WHERE userId = ? 
ORDER BY _creationTime DESC 
LIMIT 5;

// Snippet detail page with auth check
SELECT * FROM snippets WHERE _id = ?;
SELECT * FROM snippetComments WHERE snippetId = ? ORDER BY _creationTime DESC;
SELECT COUNT(*) FROM stars WHERE snippetId = ? AND userId = ?;

// Pro feature enforcement
// FREE: JavaScript, C, C++, Python, Java
// PRO: Rust, Go, C#, Kotlin, PHP, Ruby
const FREE_LANGUAGES = ["javascript", "c", "cpp", "python", "java"];
if (!user.isPro && !FREE_LANGUAGES.includes(args.language)) {
  throw new ConvexError("Pro subscription required");
}
```

### 3.2 Authorization Model

**Authentication Flow**:
```
1. User signs up → Clerk creates JWT
2. Frontend sends JWT with Convex requests
3. Convex ctx.auth.getUserIdentity() validates JWT
4. Returns Clerk user object with userId (subject)
```

**Authorization Checks**:
```typescript
// Check 1: User authenticated
const identity = await ctx.auth.getUserIdentity();
if (!identity) throw new Error("Not authenticated");

// Check 2: User owns resource
const snippet = await ctx.db.get(snippetId);
if (snippet.userId !== identity.subject) {
  throw new Error("Not authorized to delete");
}

// Check 3: Pro feature access
const user = await ctx.db.query("users")
  .withIndex("by_user_id", q => q.eq("userId", identity.subject))
  .first();
const FREE_LANGUAGES = ["javascript", "c", "cpp", "python", "java"];
if (!user.isPro && !FREE_LANGUAGES.includes(args.language)) {
  throw new ConvexError("Pro subscription required");
}
```

### 3.3 Webhook Security

**Clerk Webhook Verification**:
```typescript
const webhookSecret = process.env.CLERK_WEBHOOK_SECRET;
const svix_id = request.headers.get("svix-id");
const svix_signature = request.headers.get("svix-signature");
const svix_timestamp = request.headers.get("svix-timestamp");

const wh = new Webhook(webhookSecret);
const evt = wh.verify(body, {
  "svix-id": svix_id,
  "svix-timestamp": svix_timestamp,
  "svix-signature": svix_signature,
});
```

**Lemon Squeezy Webhook Verification**:
```typescript
import { createHmac } from "crypto";

const computedSignature = createHmac("sha256", webhookSecret)
  .update(payload)
  .digest("hex");

if (signature !== computedSignature) {
  throw new Error("Invalid signature");
}
```

**Why verification**:
- Prevents spoofed webhooks
- Ensures only legitimate events processed
- Avoids giving pro access to attackers
- Part of OAuth 2.0 / webhook best practices

### 3.4 Business Logic

**Execution History Enforcement**:
```typescript
// Save execution (pro limits)
// FREE: JavaScript, C, C++, Python, Java
// PRO: Rust, Go, C#, Kotlin, PHP, Ruby
const FREE_LANGUAGES = ["javascript", "c", "cpp", "python", "java"];
if (!user.isPro && !FREE_LANGUAGES.includes(args.language)) {
  throw new ConvexError("Pro subscription required");
}
```

**Cascade Delete Pattern**:
```typescript
// When deleting snippet, must delete:
// 1. All comments on that snippet
// 2. All stars on that snippet
// 3. The snippet itself

for (const comment of comments) {
  await ctx.db.delete(comment._id);
}
// Comments deleted ✓
// Stars deleted ✓
// Snippet deleted ✓
```

**Statistics Calculation**:
```typescript
const executions = await ctx.db
  .query("codeExecutions")
  .withIndex("by_user_id", q => q.eq("userId", userId))
  .collect();

const languageStats = executions.reduce((acc, curr) => {
  acc[curr.language] = (acc[curr.language] || 0) + 1;
  return acc;
}, {});

const favoriteLanguage = languages.reduce((a, b) =>
  languageStats[a] > languageStats[b] ? a : b
);
```

---

## 4. FRONTEND ARCHITECTURE & UX

### 4.1 State Management Strategy

**Zustand Store Structure**:
```typescript
useCodeEditorStore = {
  // Persistent state (survives page reload)
  language: "javascript",
  theme: "vs-dark",
  fontSize: 16,
  
  // Runtime state (cleared on reload)
  output: "",
  error: null,
  isRunning: false,
  editor: null,
  executionResult: null,
  
  // Actions
  setLanguage: () => { /* switch & persist */ },
  setTheme: () => { /* change theme & persist */ },
  runCode: () => { /* execute via Piston API */ },
  
  // Computed
  getCode: () => editor.getValue(),
};
```

**Why not Redux**:
- No action creators/reducers boilerplate
- No middleware setup
- Direct state access
- Smaller bundle size

**Persistence Pattern**:
```typescript
// On language change: save current language's code
const currentCode = get().editor?.getValue();
localStorage.setItem(`editor-code-${get().language}`, currentCode);

// On mount: restore code for current language
const saved = localStorage.getItem(`editor-code-${language}`);
if (saved) editor.setValue(saved);
```

### 4.2 Component Composition

**Smart vs Presentational**:
- **Smart**: EditorPanel, SnippetsPage, ProfilePage (use hooks, queries)
- **Presentational**: SnippetCard, CodeBlock, OutputPanel (pure components)

**Props Drilling Avoidance**:
```typescript
// ✓ Use Zustand in EditorPanel, pass props to child
function EditorPanel() {
  const { editor, setEditor } = useCodeEditorStore();
  return <Editor onMount={setEditor} />;
}

// ✗ Don't drill through multiple levels
function GrandParent() {
  const { editor } = useCodeEditorStore();
  return <Parent editor={editor} />;
}
function Parent({ editor }) {
  return <Child editor={editor} />;
}
```

**Hydration Safety**:
```typescript
// Problem: Monaco Editor renders on server, adds to client bundle
// Solution: useMounted hook
const mounted = useMounted();
return mounted ? <Editor /> : <EditorPanelSkeleton />;
```

### 4.3 User Experience Optimizations

**Skeleton Loading**:
```typescript
if (snippets === undefined) {
  return <SnippetsPageSkeleton />;
}
// Show realistic placeholder while loading
```

**Optimistic Updates**:
```typescript
// Star button responds immediately
const handleStar = async () => {
  setStarred(!starred); // optimistic
  try {
    await starSnippet({ snippetId });
  } catch (error) {
    setStarred(!starred); // revert
  }
};
```

**Smooth Animations**:
```typescript
<motion.button
  whileHover={{ scale: 1.02 }}
  whileTap={{ scale: 0.98 }}
  onClick={handleRun}
>
  Run Code
</motion.button>
```

---

## 5. AUTHENTICATION, AUTHORIZATION & SECURITY

### 5.1 Authentication Flow

**Complete Flow**:
```
1. User clicks "Sign In"
   ↓
2. Redirects to Clerk login page
   ↓
3. User authenticates with Google/GitHub
   ↓
4. Clerk returns JWT token in cookie
   ↓
5. Middleware (src/middleware.ts) validates JWT
   ↓
6. useUser() hook retrieves user info
   ↓
7. JWT sent with Convex requests
   ↓
8. Convex verifies JWT signature
   ↓
9. ctx.auth.getUserIdentity() available in mutations
```

**JWT Token Contents**:
```json
{
  "iss": "https://accounts.clerk.com",
  "sub": "user_abc123",          // Clerk user ID
  "aud": "my-convex-app",
  "exp": 1704067200,              // Expires in 1 hour
  "iat": 1704063600,
  "email": "user@example.com",
  "email_verified": true
}
```

### 5.2 Authorization Model

**Resource-Based Access Control**:
```typescript
// Only owner can delete their snippet
const snippet = await ctx.db.get(snippetId);
if (snippet.userId !== identity.subject) {
  throw new Error("Unauthorized");
}
```

**Feature-Based Access Control**:
```typescript
// Only pro users can execute non-JS languages
const user = await ctx.db.query("users")
  .withIndex("by_user_id", q => q.eq("userId", identity.subject))
  .first();

if (!user.isPro && args.language !== "javascript") {
  throw new ConvexError("Upgrade to pro");
}
```

### 5.3 Security Considerations

**Protected Data**:
- User credentials: Clerk handles (never stored)
- Execution results: Temporary (cleared after response)
- Pro status: Verified server-side on each request
- Payment info: Lemon Squeezy handles (never stored)

**XSS Prevention**:
- User content (snippets, comments) not evaluated
- Syntax highlighting escapes HTML
- No eval() or dangerouslySetInnerHTML

**CSRF Prevention**:
- Convex uses same-origin cookies
- Clerk JWT in Authorization header
- Lemon Squeezy webhook signature verification

**SQL Injection Prevention**:
- Convex uses structured queries (not string concatenation)
- No raw SQL execution
- Parameter binding automatic

---

## 6. PERFORMANCE OPTIMIZATION & SCALABILITY

### 6.1 Code Execution Performance

**Optimization**:
```
User writes code → Piston API sandboxes → Returns in ~100-500ms
```

**Bottlenecks**:
1. Network latency to Piston (~50-100ms)
2. Compilation time (varies by language)
3. Execution time (user's code)

**Solutions**:
- Timeout on Piston requests (prevent hanging)
- Show loading skeleton while executing
- Disable Run button during execution
- Cache language configs

### 6.2 Database Query Performance

**Indexed Queries**:
```typescript
// ✓ Fast: uses index
const user = await ctx.db
  .query("users")
  .withIndex("by_user_id", q => q.eq("userId", userId))
  .first();

// ✗ Slow: table scan
const user = await ctx.db
  .query("users")
  .filter(q => q.eq(q.field("userId"), userId))
  .first();
```

**Pagination for Large Result Sets**:
```typescript
// Profile page: 5 executions per page
const { results, loadMore } = usePaginatedQuery(
  api.codeExecutions.getUserExecutions,
  { userId },
  { initialNumItems: 5 }
);
```

**Denormalization for Read Performance**:
```typescript
// snippets table includes userName (denormalized)
// Query: SELECT * FROM snippets WHERE _id = ?
// Returns: { _id, userId, userName, title, code }
// Result: No need to JOIN users table
```

### 6.3 Frontend Performance

**Code Splitting**:
```typescript
// Each page loaded on-demand (Next.js automatic)
app/(root)/page.tsx      // /
app/snippets/page.tsx    // /snippets
app/profile/page.tsx     // /profile
```

**Monaco Editor Lazy Loading**:
```typescript
const mounted = useMounted();
if (!mounted) return <EditorPanelSkeleton />;
return <Editor />; // Load only on client
```

**Memoization**:
```typescript
// Prevent unnecessary re-renders
const SnippetCard = memo(({ snippet }) => {
  return <div>{snippet.title}</div>;
});
```

### 6.4 Scalability Analysis

**Current Limits**:
- Convex: 5,000 documents/month free tier (sufficient for MVP)
- Piston API: Rate limited (~100 req/sec per IP)
- Clerk: Unlimited users on hobby plan

**Scaling Approach**:
1. **Vertical**: Pay-as-you-go Convex plan
2. **Caching**: Redis for popular snippets
3. **CDN**: Static assets (language logos) via Cloudflare
4. **Regional**: Multi-region Piston proxies

**Estimated Capacity**:
- 100 concurrent users: ✓ No problem
- 10K daily executions: ✓ No problem
- 1M monthly users: Need to scale Convex, add caching

---

## 7. RELIABILITY, ERROR HANDLING & OBSERVABILITY

### 7.1 Error Handling

**Client-Side**:
```typescript
try {
  await runCode();
} catch (error) {
  set({
    error: error.message,
    output: "",
    isRunning: false,
  });
  // User sees error in OutputPanel
}
```

**Server-Side**:
```typescript
try {
  const evt = wh.verify(body, headers);
  // Process event
} catch (err) {
  console.error("Webhook verification failed:", err);
  return new Response("Error occurred", { status: 400 });
}
```

**User-Facing**:
```typescript
// Compilation error (from Piston)
"Error: Unexpected token '}' at line 5"

// Runtime error (from Piston)
"ReferenceError: variable 'x' is not defined"

// Auth error (from Convex)
"Not authenticated"

// Pro feature error
"Pro subscription required"
```

### 7.2 Observability

**Logging**:
```typescript
console.log("data back from piston:", data);
console.log("Error running code:", error);
console.error("Webhook verification failed:", err);
```

**Monitoring Points**:
1. Piston API response times
2. Webhook failures (Clerk, Lemon Squeezy)
3. Database query performance
4. User authentication failures
5. Payment webhook processing

**What to Monitor**:
- Execution latency (target: < 500ms)
- Error rates (target: < 1%)
- Webhook delivery (target: 100% success)
- Code execution memory usage

---

## 8. TESTING STRATEGY & CODE QUALITY

### 8.1 Testing Approach

**Unit Tests** (not currently implemented, but should test):
- Zustand store actions (setLanguage, runCode)
- Language config lookups
- Statistics calculations

**Integration Tests**:
- Create snippet + verify appears in feed
- Star snippet + verify star count updates
- Execute code + verify execution saved

**Manual Testing Checklist**:
```
✓ Write JavaScript code → Run → See output
✓ Switch language → Code persists
✓ Logout → Re-login → Code still there
✓ Share snippet → Can view via URL
✓ Comment on snippet → Author can delete
✓ Upgrade to pro → Execute Python
✓ Webhook from Clerk → User synced to DB
✓ Webhook from Lemon Squeezy → User marked pro
```

### 8.2 Code Quality

**TypeScript Strict Mode**: All types checked
```typescript
// ✓ Type-safe
const user: User = await getUser(userId);

// ✗ Would fail strict mode
const user = await getUser(userId); // Missing type annotation
```

**No Unused Code**:
- ESLint catches unused imports
- Dead code removal during build
- Tree-shaking removes unused dependencies

---

## 9. DEPLOYMENT, CI/CD & ENVIRONMENT MANAGEMENT

### 9.1 Deployment Architecture

```
Dev Environment
  ├─ Local: npm run dev (localhost:3000)
  ├─ .env.local: Local Clerk/Convex keys
  └─ Piston API: https://emkc.org (public)

Production Environment
  ├─ Frontend: Vercel (npm run build → npm start)
  ├─ Backend: Convex Cloud (npx convex deploy)
  ├─ Auth: Clerk production keys
  ├─ Payments: Lemon Squeezy live keys
  └─ Webhooks: Public URLs for Clerk, Lemon Squeezy
```

### 9.2 Environment Variables

**Client-Side** (prefixed with NEXT_PUBLIC_):
```env
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_live_xxx
NEXT_PUBLIC_CONVEX_URL=https://xxx.convex.cloud
NEXT_PUBLIC_LEMON_SQUEEZY_CHECKOUT_URL=https://xxx.lemonsqueezy.com/checkout
```

**Server-Side**:
```env
CLERK_SECRET_KEY=sk_live_xxx
CLERK_WEBHOOK_SECRET=whsec_xxx
LEMON_SQUEEZY_WEBHOOK_SECRET=...
```

### 9.3 Deployment Process

```
1. Commit code to Git
2. GitHub/GitLab webhook triggers CI/CD
3. Run tests (if any)
4. Build Next.js: npm run build
5. Deploy to Vercel (frontend)
6. Deploy to Convex: npx convex deploy (backend)
7. Configure webhooks in Clerk/Lemon Squeezy
8. Test in production
```

---

## 10. PROJECT EVOLUTION, TRADE-OFFS & LEARNINGS

### 10.1 Technical Debt

**What's Working Well**:
- ✓ Real-time database with Convex (no polling)
- ✓ Simple authentication with Clerk (no custom auth)
- ✓ Zustand store (minimal boilerplate)
- ✓ Monaco Editor (professional code editor)

**What Could Be Improved**:
1. **Testing**: No unit/integration tests currently
   - Should add Jest + React Testing Library
   
2. **Error Boundaries**: No React error boundaries
   - Should wrap components to catch render errors
   
3. **Logging**: console.log in production
   - Should use proper logging library (LogRocket, Sentry)
   
4. **Rate Limiting**: No rate limit on Piston API calls
   - Should add client-side throttling
   
5. **Caching**: No caching of popular snippets
   - Could improve performance with Redis

### 10.2 Trade-Offs Made

| Decision | Chosen | Alternative | Why |
|----------|--------|-------------|-----|
| State Management | Zustand | Redux | Less boilerplate, faster development |
| Backend | Convex | Firebase | Convex has better auth integration |
| Code Execution | Piston API | Docker | Piston handles all languages, no infra |
| Payments | Lemon Squeezy | Stripe | Simpler webhook verification |
| CSS | Tailwind | Styled Components | Smaller bundle, better performance |

### 10.3 Architectural Decisions

**Decision 1: Denormalize userName in snippets**
```
Pro: Can show snippet title + author without joining users table
Con: Requires updating userName when user changes name
Resolution: userName is rarely updated; acceptable trade-off
```

**Decision 2: Store code in Convex (not S3)**
```
Pro: Simpler architecture, no extra service
Con: Convex has storage limits (~100KB per document)
Limitation: Only works for code < 100KB (acceptable for snippets)
```

**Decision 3: localStorage for code (not Convex)**
```
Pro: Fast access, works offline, reduces server load
Con: Lost if cookies cleared
Compromise: Auto-save to Convex when sharing (best of both)
```

### 10.4 Lessons Learned

**What Went Well**:
1. Convex real-time sync makes collaborative features easy
2. Clerk webhooks provide seamless user sync
3. Zustand store keeps editor logic simple
4. Piston API handles language complexity

**What Was Challenging**:
1. Hydration mismatches required useMounted() hook
2. JWT token lifecycle required careful handling
3. Cascade deletes in Convex needed manual implementation
4. Testing webhooks requires dev environment setup

**If Rebuilding**:
1. Start with tests from day 1
2. Use error boundaries for stability
3. Implement proper logging early
4. Add rate limiting to Piston calls
5. Document webhook setup better

---

## Interview Talking Points

### For System Design Questions:
- "I chose Convex for real-time sync instead of traditional REST API"
- "Zustand is minimal for editor-only state (not Redux complexity)"
- "Piston API abstracts away language execution complexity"
- "Denormalized schema prioritizes read performance"

### For Architectural Questions:
- "Real-time updates via Convex WebSocket instead of polling"
- "Webhooks enable passive integration with Clerk/Lemon Squeezy"
- "Zustand with localStorage balances persistence and performance"
- "Cascade deletes maintain referential integrity in Convex"

### For Scalability Questions:
- "Pagination prevents loading entire execution history"
- "Indexed queries ensure O(log n) lookups"
- "Lazy-loaded Monaco Editor reduces initial bundle"
- "Piston API scales horizontally via load balancing"

### For Security Questions:
- "Clerk JWT prevents session hijacking"
- "HMAC signature verification prevents webhook spoofing"
- "Resource-based authorization (userId matching) in every mutation"
- "Pro feature enforcement server-side (never trust client)"

### For Trade-offs:
- "Chose Lemon Squeezy over Stripe for simpler webhooks"
- "Store code in Convex (simpler) vs S3 (more scalable)"
- "Zustand (simpler) vs Redux (more testable)"
- "Piston public API (free) vs self-hosted (control)"

---

## Summary

CodeForge demonstrates:
1. **Full-stack architecture**: Frontend (React), Backend (Convex), Auth (Clerk), Payments (Lemon Squeezy)
2. **Real-time features**: WebSocket sync, optimistic updates
3. **Community features**: Snippets, comments, stars, discovery
4. **Monetization**: Free (JavaScript) vs Pro (all languages)
5. **Production concerns**: Security, performance, reliability, scalability

Master these concepts, and you'll confidently discuss CodeForge in any senior-level interview.
