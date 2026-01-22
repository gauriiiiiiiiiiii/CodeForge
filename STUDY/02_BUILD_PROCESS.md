# CodeForge - Complete Build Process (From Scratch)

## Overview
This document describes the complete step-by-step process of building CodeForge from scratch, including architectural decisions, file creation order, and implementation details.

---

## Phase 1: Project Setup (Foundation)

### Step 1.1: Initialize Next.js 15 Project
```bash
npx create-next-app@latest codeforge --typescript --tailwind
cd codeforge
```

**What was set up**:
- Next.js 15.1.0 with App Router
- TypeScript 5 with strict mode
- Tailwind CSS 3.4.1 for styling
- PostCSS for CSS processing

**Files created**:
- `package.json` - Project manifest with 180 dependencies
- `tsconfig.json` - TypeScript strict mode config
- `next.config.ts` - Next.js configuration
- `tailwind.config.ts` - Tailwind dark theme customization
- `postcss.config.mjs` - CSS pipeline
- `src/app/layout.tsx` - Root layout wrapper
- `src/app/globals.css` - Global styles

### Step 1.2: Install Core Dependencies
```bash
npm install convex @clerk/nextjs zustand @monaco-editor/react 
npm install framer-motion lucide-react react-hot-toast 
npm install react-syntax-highlighter svix axios clsx
```

**Why each package**:
- **convex** - Real-time backend database with authentication support
- **@clerk/nextjs** - OAuth authentication (Google, GitHub, etc.)
- **zustand** - Minimal state management for editor state
- **@monaco-editor/react** - Professional code editor component
- **framer-motion** - Smooth UI animations
- **lucide-react** - Icon library for UI
- **react-hot-toast** - Toast notifications for user feedback
- **react-syntax-highlighter** - Syntax highlighting for code display
- **svix** - Webhook verification for secure integrations
- **axios** - HTTP client for API calls
- **clsx** - Conditional className utility

### Step 1.3: Set Up Environment Variables
```env
# .env.local file:
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_xxx
CLERK_SECRET_KEY=sk_test_xxx
NEXT_PUBLIC_CONVEX_URL=https://xxx.convex.cloud
CLERK_WEBHOOK_SECRET=whsec_xxx
LEMON_SQUEEZY_WEBHOOK_SECRET=xxx
```

**Explanation**:
- Clerk variables enable OAuth login
- Convex URL points to backend database
- Webhook secrets verify payment & auth events

---

## Phase 2: Type System & Constants (Architecture Foundation)

### Step 2.1: Create Type Definitions
**File**: `src/types/index.ts`

```typescript
// Define TypeScript interfaces for the entire app
interface CodeEditorState {
  language: string;
  theme: string;
  fontSize: number;
  output: string;
  error: string | null;
  editor: Monaco | null;
  isRunning: boolean;
  executionResult: ExecutionResult | null;
  // ... methods
}

interface Snippet {
  _id: Id<"snippets">;
  userId: string;
  title: string;
  language: string;
  code: string;
  userName: string;
}
```

**Why this first**: Type safety is foundational. All subsequent code can reference these types.

### Step 2.2: Define Language Configurations
**File**: `src/app/(root)/_constants/index.ts` (436 lines)

```typescript
// Specify which languages are supported
LANGUAGE_CONFIG = {
  javascript: {
    id: "javascript",
    label: "JavaScript",
    monacoLanguage: "javascript",
    pistonRuntime: { language: "javascript", version: "18.15.0" },
    defaultCode: `// JavaScript example...`
  },
  python: { ... },
  java: { ... },
  // ... 7 more languages
}

// Custom Monaco themes (vs-dark, vs-light, etc.)
defineMonacoThemes(monaco)
```

**Why defined early**: Language support is a core feature decision. All components reference this configuration.

---

## Phase 3: Backend Architecture (Convex Setup)

### Step 3.1: Initialize Convex
```bash
npm run convex
# Creates convex/ folder and _generated/ types
```

### Step 3.2: Define Database Schema
**File**: `convex/schema.ts`

```typescript
export default defineSchema({
  users: defineTable({
    userId: v.string(),     // Clerk user ID
    email: v.string(),
    name: v.string(),
    isPro: v.boolean(),
    proSince: v.optional(v.number()),
    lemonSqueezyCustomerId: v.optional(v.string()),
  }).index("by_user_id", ["userId"]),
  
  snippets: defineTable({
    userId: v.string(),
    title: v.string(),
    language: v.string(),
    code: v.string(),
    userName: v.string(),
  }).index("by_user_id", ["userId"]),
  
  snippetComments: defineTable({
    snippetId: v.id("snippets"),
    userId: v.string(),
    userName: v.string(),
    content: v.string(),
  }).index("by_snippet_id", ["snippetId"]),
  
  codeExecutions: defineTable({
    userId: v.string(),
    language: v.string(),
    code: v.string(),
    output: v.optional(v.string()),
    error: v.optional(v.string()),
  }).index("by_user_id", ["userId"]),
  
  stars: defineTable({
    userId: v.string(),
    snippetId: v.id("snippets"),
  })
    .index("by_user_id", ["userId"])
    .index("by_snippet_id", ["snippetId"])
    .index("by_user_id_and_snippet_id", ["userId", "snippetId"]),
});
```

**Architectural decisions**:
- Denormalize `userName` in snippets/comments for fast reads
- Use compound index (userId, snippetId) to prevent duplicate stars
- Store execution history for analytics & pro-feature enforcement

### Step 3.3: Create User Management Functions
**File**: `convex/users.ts`

```typescript
// Sync user on first Clerk login
export const syncUser = mutation({
  args: { userId, email, name },
  handler: async (ctx, args) => {
    // Check if user exists
    const existing = await ctx.db.query("users")
      .withIndex("by_user_id", q => q.eq("userId", args.userId))
      .first();
    
    if (!existing) {
      // Create new user
      await ctx.db.insert("users", {
        userId: args.userId,
        email: args.email,
        name: args.name,
        isPro: false,
      });
    }
  }
});

// Fetch user profile
export const getUser = query({
  args: { userId },
  handler: async (ctx, args) => {
    return await ctx.db.query("users")
      .withIndex("by_user_id", q => q.eq("userId", args.userId))
      .first();
  }
});

// Upgrade to pro (called by Lemon Squeezy webhook)
export const upgradeToPro = mutation({
  args: { email, lemonSqueezyCustomerId, lemonSqueezyOrderId, amount },
  handler: async (ctx, args) => {
    const user = await ctx.db.query("users")
      .filter(q => q.eq(q.field("email"), args.email))
      .first();
    
    if (user) {
      await ctx.db.patch(user._id, {
        isPro: true,
        proSince: Date.now(),
        lemonSqueezyCustomerId: args.lemonSqueezyCustomerId,
        lemonSqueezyOrderId: args.lemonSqueezyOrderId,
      });
    }
  }
});
```

**Why this order**:
1. Sync user on login (foundation)
2. Fetch user data (for profile pages)
3. Upgrade to pro (payment integration)

### Step 3.4: Implement Snippet Management
**File**: `convex/snippets.ts` (227 lines)

```typescript
// Create snippet (shared code)
export const createSnippet = mutation({
  args: { title, language, code },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Not authenticated");
    
    const user = await ctx.db.query("users")
      .withIndex("by_user_id", q => q.eq("userId", identity.subject))
      .first();
    
    return await ctx.db.insert("snippets", {
      userId: identity.subject,
      userName: user.name,
      title: args.title,
      language: args.language,
      code: args.code,
    });
  }
});

// Delete snippet (cascade delete comments & stars)
export const deleteSnippet = mutation({
  args: { snippetId },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    const snippet = await ctx.db.get(args.snippetId);
    
    // Authorization check
    if (snippet.userId !== identity.subject) {
      throw new Error("Not authorized");
    }
    
    // Cascade delete comments
    const comments = await ctx.db.query("snippetComments")
      .withIndex("by_snippet_id", q => q.eq("snippetId", args.snippetId))
      .collect();
    for (const comment of comments) {
      await ctx.db.delete(comment._id);
    }
    
    // Cascade delete stars
    const stars = await ctx.db.query("stars")
      .withIndex("by_snippet_id", q => q.eq("snippetId", args.snippetId))
      .collect();
    for (const star of stars) {
      await ctx.db.delete(star._id);
    }
    
    // Delete snippet
    await ctx.db.delete(args.snippetId);
  }
});

// Star/unstar toggle
export const starSnippet = mutation({
  args: { snippetId },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    
    const existing = await ctx.db.query("stars")
      .withIndex("by_user_id_and_snippet_id", q =>
        q.eq("userId", identity.subject) && 
        q.eq("snippetId", args.snippetId)
      )
      .first();
    
    if (existing) {
      await ctx.db.delete(existing._id);
    } else {
      await ctx.db.insert("stars", {
        userId: identity.subject,
        snippetId: args.snippetId,
      });
    }
  }
});

// Comment management
export const addComment = mutation({
  args: { snippetId, content },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    const user = await ctx.db.query("users")
      .withIndex("by_user_id", q => q.eq("userId", identity.subject))
      .first();
    
    return await ctx.db.insert("snippetComments", {
      snippetId: args.snippetId,
      userId: identity.subject,
      userName: user.name,
      content: args.content,
    });
  }
});

// Query all snippets
export const getSnippets = query({
  handler: async (ctx) => {
    return await ctx.db.query("snippets")
      .order("desc")
      .collect();
  }
});
```

**Key architectural choices**:
- Authorization checks in every mutation (security)
- Cascade deletes to maintain referential integrity
- Compound index for efficient star lookups

### Step 3.5: Track Execution History
**File**: `convex/codeExecutions.ts` (97 lines)

```typescript
// Save each code execution
export const saveExecution = mutation({
  args: { language, code, output, error },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    const user = await ctx.db.query("users")
      .withIndex("by_user_id", q => q.eq("userId", identity.subject))
      .first();
    
    // Pro feature: non-free languages require pro subscription
    // FREE: JavaScript, C, C++, Python, Java
    // PRO: Rust, Go, C#, Kotlin, PHP, Ruby
    const FREE_LANGUAGES = ["javascript", "c", "cpp", "python", "java"];
    if (!user.isPro && !FREE_LANGUAGES.includes(args.language)) {
      throw new ConvexError("Pro subscription required");
    }
    
    await ctx.db.insert("codeExecutions", {
      userId: identity.subject,
      language: args.language,
      code: args.code,
      output: args.output,
      error: args.error,
    });
  }
});

// Paginated execution history
export const getUserExecutions = query({
  args: { userId, paginationOpts },
  handler: async (ctx, args) => {
    return await ctx.db.query("codeExecutions")
      .withIndex("by_user_id", q => q.eq("userId", args.userId))
      .order("desc")
      .paginate(args.paginationOpts);
  }
});

// User statistics
export const getUserStats = query({
  args: { userId },
  handler: async (ctx, args) => {
    const executions = await ctx.db.query("codeExecutions")
      .withIndex("by_user_id", q => q.eq("userId", args.userId))
      .collect();
    
    // Calculate language breakdown
    const languageStats = executions.reduce((acc, curr) => {
      acc[curr.language] = (acc[curr.language] || 0) + 1;
      return acc;
    }, {});
    
    return {
      totalExecutions: executions.length,
      languagesCount: Object.keys(languageStats).length,
      favoriteLanguage: /* ... */,
      last24Hours: /* ... */,
    };
  }
});
```

**Why track executions**:
1. Enforce pro-feature limits (non-JS requires subscription)
2. Show user statistics on profile
3. Analytics for future features

### Step 3.6: Set Up Webhooks
**File**: `convex/http.ts` (116 lines)

```typescript
// Clerk webhook: sync user on first login
http.route({
  path: "/clerk-webhook",
  method: "POST",
  handler: httpAction(async (ctx, request) => {
    const webhookSecret = process.env.CLERK_WEBHOOK_SECRET;
    const payload = await request.json();
    
    // Verify webhook signature with Svix
    const wh = new Webhook(webhookSecret);
    const evt = wh.verify(body, headers);
    
    if (evt.type === "user.created") {
      const { id, email_addresses, first_name, last_name } = evt.data;
      const email = email_addresses?.[0]?.email_address;
      const name = `${first_name} ${last_name}`.trim();
      
      // Sync user to Convex
      await ctx.runMutation(api.users.syncUser, {
        userId: id,
        email,
        name,
      });
    }
    
    return new Response("OK", { status: 200 });
  })
});

// Lemon Squeezy webhook: process payment
http.route({
  path: "/lemon-squeezy-webhook",
  method: "POST",
  handler: httpAction(async (ctx, request) => {
    const payload = await request.text();
    const signature = request.headers.get("X-Signature");
    
    // Verify HMAC signature
    const verified = await ctx.runAction(
      internal.lemonSqueezy.verifyWebhook,
      { payload, signature }
    );
    
    if (verified.meta.event_name === "order_created") {
      // Upgrade user to pro
      await ctx.runMutation(api.users.upgradeToPro, {
        email: verified.data.attributes.user_email,
        lemonSqueezyCustomerId: verified.data.attributes.customer_id,
        lemonSqueezyOrderId: verified.data.id,
        amount: verified.data.attributes.total,
      });
    }
    
    return new Response("OK", { status: 200 });
  })
});
```

**Why webhooks**:
1. Clerk webhook syncs user immediately on signup
2. Lemon Squeezy webhook enables payment processing
3. Proper signature verification prevents spoofing

### Step 3.7: Webhook Verification
**File**: `convex/lemonSqueezy.ts` (26 lines)

```typescript
import { createHmac } from "crypto";

export const verifyWebhook = internalAction({
  args: { payload, signature },
  handler: async (ctx, args) => {
    const webhookSecret = process.env.LEMON_SQUEEZY_WEBHOOK_SECRET;
    
    // Recreate HMAC signature
    const hmac = createHmac("sha256", webhookSecret);
    const computedSignature = hmac
      .update(args.payload)
      .digest("hex");
    
    // Verify signature matches
    if (args.signature !== computedSignature) {
      throw new Error("Invalid signature");
    }
    
    return JSON.parse(args.payload);
  }
});
```

**Security note**: HMAC prevents unauthorized webhook calls.

---

## Phase 4: Frontend - State Management

### Step 4.1: Create Zustand Editor Store
**File**: `src/store/useCodeEditorStore.ts` (162 lines)

```typescript
export const useCodeEditorStore = create<CodeEditorState>((set, get) => {
  // Load from localStorage
  const initialState = getInitialState();
  
  return {
    ...initialState,
    output: "",
    isRunning: false,
    error: null,
    editor: null,
    
    // Setters
    setLanguage: (language: string) => {
      const currentCode = get().editor?.getValue();
      // Save old language's code before switching
      if (currentCode) {
        localStorage.setItem(`editor-code-${get().language}`, currentCode);
      }
      
      set({ language, output: "", error: null });
    },
    
    // Code execution
    runCode: async () => {
      const { language, getCode } = get();
      const code = getCode();
      
      set({ isRunning: true, error: null, output: "" });
      
      try {
        // Call Piston API
        const response = await fetch(
          "https://emkc.org/api/v2/piston/execute",
          {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
              language: LANGUAGE_CONFIG[language].pistonRuntime.language,
              version: LANGUAGE_CONFIG[language].pistonRuntime.version,
              files: [{ content: code }],
            }),
          }
        );
        
        const data = await response.json();
        
        // Handle compilation errors
        if (data.compile?.code !== 0) {
          const error = data.compile.stderr || data.compile.output;
          set({ error, executionResult: { code, output: "", error } });
          return;
        }
        
        // Handle runtime errors
        if (data.run?.code !== 0) {
          const error = data.run.stderr || data.run.output;
          set({ error, executionResult: { code, output: "", error } });
          return;
        }
        
        // Success
        const output = data.run.output.trim();
        set({
          output,
          error: null,
          executionResult: { code, output, error: null },
        });
      } catch (error) {
        set({
          error: error.message,
          executionResult: { code, output: "", error: error.message },
        });
      } finally {
        set({ isRunning: false });
      }
    },
  };
});
```

**Why Zustand**:
- Minimal boilerplate compared to Redux
- Direct state access without selectors
- localStorage integration for persistence
- Supports async actions (runCode)

---

## Phase 5: Frontend - Provider Setup

### Step 5.1: Create Authentication & Database Provider
**File**: `src/components/providers/ConvexClientProvider.tsx`

```typescript
const convex = new ConvexReactClient(
  process.env.NEXT_PUBLIC_CONVEX_URL!
);

export default function ConvexClientProvider({
  children,
}: {
  children: React.ReactNode;
}) {
  if (process.env.NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY) {
    return (
      <ClerkProvider
        publishableKey={process.env.NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY}
      >
        <ConvexProviderWithClerk
          client={convex}
          useAuth={useAuth}
        >
          {children}
        </ConvexProviderWithClerk>
      </ClerkProvider>
    );
  }
  
  // Fallback if Clerk not configured
  return <ConvexProvider client={convex}>{children}</ConvexProvider>;
}
```

**Architecture**:
- `ConvexProviderWithClerk` connects Clerk auth to Convex
- Automatic JWT token passing from Clerk to Convex
- Fallback for local development without auth

### Step 5.2: Update Root Layout
**File**: `src/app/layout.tsx`

```typescript
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <ClerkProvider>
      <html lang="en" suppressHydrationWarning>
        <body>
          <ConvexClientProvider>
            {children}
          </ConvexClientProvider>
          
          <Footer />
          <Toaster /> {/* Toast notifications */}
        </body>
      </html>
    </ClerkProvider>
  );
}
```

**Why this structure**:
1. ClerkProvider at top (auth initialization)
2. ConvexClientProvider next (database access)
3. Children get access to both auth + DB
4. Toaster catches notifications globally

### Step 5.3: Add Middleware for Protected Routes
**File**: `src/middleware.ts`

```typescript
import { clerkMiddleware } from "@clerk/nextjs/server";

const isClerkConfigured =
  !!process.env.CLERK_SECRET_KEY &&
  !!process.env.NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY;

export default isClerkConfigured
  ? clerkMiddleware()
  : function middleware() {
      return NextResponse.next();
    };

export const config = {
  matcher: [
    "/((?!_next|favicon.ico|.*\\.(?:svg|png|jpg)).*)",
    "/(api|trpc)(.*)",
  ],
};
```

**Why**:
- Clerk middleware checks auth on every request
- Gracefully disables if Clerk not configured
- Allows local development without auth

---

## Phase 6: Frontend - Core Components

### Step 6.1: Build Editor Component
**File**: `src/app/(root)/_components/EditorPanel.tsx` (200 lines)

```typescript
function EditorPanel() {
  const { language, theme, fontSize, editor, setFontSize, setEditor } =
    useCodeEditorStore();
  const [isShareDialogOpen, setIsShareDialogOpen] = useState(false);
  const mounted = useMounted();
  
  useEffect(() => {
    // Load saved code for this language
    const savedCode = localStorage.getItem(`editor-code-${language}`);
    if (editor) {
      editor.setValue(
        savedCode || LANGUAGE_CONFIG[language].defaultCode
      );
    }
  }, [language, editor]);
  
  const handleEditorChange = (value: string) => {
    // Persist code to localStorage as user types
    localStorage.setItem(`editor-code-${language}`, value);
  };
  
  return (
    <div className="bg-[#12121a] rounded-xl p-6">
      {/* Header with controls */}
      <div className="flex justify-between mb-4">
        <div className="flex gap-3">
          <div>
            <h2>Code Editor</h2>
            <p>Write and execute your code</p>
          </div>
        </div>
        
        <div className="flex gap-3">
          {/* Font size control */}
          <input
            type="range"
            min="12"
            max="24"
            value={fontSize}
            onChange={(e) =>
              handleFontSizeChange(parseInt(e.target.value))
            }
          />
          
          {/* Reset button */}
          <button onClick={handleRefresh}>
            Reset
          </button>
          
          {/* Share button */}
          <button onClick={() => setIsShareDialogOpen(true)}>
            Share
          </button>
        </div>
      </div>
      
      {/* Monaco Editor */}
      {mounted && (
        <Editor
          height="600px"
          language={LANGUAGE_CONFIG[language].monacoLanguage}
          theme={theme}
          value={/* current code */}
          onChange={handleEditorChange}
          onMount={(editor) => setEditor(editor)}
          options={{
            minimap: { enabled: false },
            fontSize,
            automaticLayout: true,
            fontFamily: '"Fira Code", Consolas, monospace',
            fontLigatures: true,
          }}
        />
      )}
      
      {/* Share dialog */}
      {isShareDialogOpen && (
        <ShareSnippetDialog
          onClose={() => setIsShareDialogOpen(false)}
        />
      )}
    </div>
  );
}
```

**Key features**:
- `useMounted()` prevents hydration mismatch
- localStorage persists code per language
- Monaco Editor with custom config
- Share dialog modal

### Step 6.2: Build Output Component
**File**: `src/app/(root)/_components/OutputPanel.tsx`

```typescript
function OutputPanel() {
  const { output, error, isRunning } = useCodeEditorStore();
  
  const handleCopy = async () => {
    await navigator.clipboard.writeText(error || output);
    // Show toast: "Copied!"
  };
  
  return (
    <div className="bg-[#181825] rounded-xl p-4">
      <div className="flex justify-between mb-3">
        <span>Output</span>
        {(error || output) && (
          <button onClick={handleCopy}>Copy</button>
        )}
      </div>
      
      <div className="bg-[#1e1e2e] rounded-xl p-4 h-[600px] overflow-auto">
        {isRunning ? (
          <div>Running...</div>
        ) : error ? (
          <div className="text-red-400">
            <AlertTriangle />
            <pre>{error}</pre>
          </div>
        ) : output ? (
          <div className="text-emerald-400">
            <CheckCircle />
            <pre>{output}</pre>
          </div>
        ) : (
          <div className="text-gray-500">
            Run your code to see output...
          </div>
        )}
      </div>
    </div>
  );
}
```

**Displays**:
- Compilation errors in red
- Successful output in green
- Copy button for output
- Loading state while executing

### Step 6.3: Build Run Button
**File**: `src/app/(root)/_components/RunButton.tsx`

```typescript
function RunButton() {
  const { runCode, isRunning, language } = useCodeEditorStore();
  const { user } = useUser();
  const saveExecution = useMutation(api.codeExecutions.saveExecution);
  
  const handleRun = async () => {
    // Run code locally
    await runCode();
    
    // Save to Convex if logged in
    if (user) {
      const result = getExecutionResult();
      await saveExecution({
        language,
        code: result.code,
        output: result.output,
        error: result.error,
      });
    }
  };
  
  return (
    <button
      onClick={handleRun}
      disabled={isRunning}
      className="px-5 py-2.5 bg-blue-600 text-white rounded-xl"
    >
      {isRunning ? (
        <>
          <Loader2 />
          Executing...
        </>
      ) : (
        <>
          <Play />
          Run Code
        </>
      )}
    </button>
  );
}
```

**Flow**:
1. Click Run
2. Zustand executes code via Piston API
3. If logged in, save execution to Convex
4. Display result in OutputPanel

### Step 6.4: Create Share Dialog
**File**: `src/app/(root)/_components/ShareSnippetDialog.tsx`

```typescript
function ShareSnippetDialog({ onClose }) {
  const { editor, language } = useCodeEditorStore();
  const { user } = useUser();
  const createSnippet = useMutation(api.snippets.createSnippet);
  const [title, setTitle] = useState("");
  
  const handleShare = async () => {
    if (!user) {
      // Show: "Please log in first"
      return;
    }
    
    const code = editor.getValue();
    
    try {
      const snippetId = await createSnippet({
        title,
        language,
        code,
      });
      
      // Redirect to snippet page
      router.push(`/snippets/${snippetId}`);
      onClose();
    } catch (error) {
      console.error("Error sharing snippet:", error);
    }
  };
  
  return (
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
      <div className="bg-[#12121a] rounded-xl p-6 max-w-md">
        <h2 className="text-xl font-bold mb-4">Share Snippet</h2>
        
        <input
          type="text"
          placeholder="Give your snippet a title..."
          value={title}
          onChange={(e) => setTitle(e.target.value)}
          className="w-full mb-4 px-4 py-2 bg-[#1e1e2e] rounded-lg"
        />
        
        <div className="flex gap-3">
          <button onClick={onClose} className="flex-1">
            Cancel
          </button>
          <button
            onClick={handleShare}
            className="flex-1 bg-blue-600 text-white rounded-lg"
          >
            Share
          </button>
        </div>
      </div>
    </div>
  );
}
```

**User flow**:
1. User writes code
2. Clicks "Share"
3. Modal prompts for title
4. Saves to Convex
5. Redirects to shareable URL

---

## Phase 7: Frontend - Community Features

### Step 7.1: Create Snippets Feed Page
**File**: `src/app/snippets/page.tsx` (223 lines)

```typescript
function SnippetsPage() {
  const snippets = useQuery(api.snippets.getSnippets);
  const [searchQuery, setSearchQuery] = useState("");
  const [selectedLanguage, setSelectedLanguage] = useState(null);
  
  const filteredSnippets = snippets.filter((snippet) => {
    const matchesSearch = snippet.title.toLowerCase().includes(
      searchQuery.toLowerCase()
    );
    const matchesLanguage =
      !selectedLanguage || snippet.language === selectedLanguage;
    
    return matchesSearch && matchesLanguage;
  });
  
  return (
    <div>
      <NavigationHeader />
      
      {/* Search */}
      <input
        type="text"
        placeholder="Search snippets..."
        value={searchQuery}
        onChange={(e) => setSearchQuery(e.target.value)}
      />
      
      {/* Language filters */}
      {languages.map((lang) => (
        <button
          key={lang}
          onClick={() => setSelectedLanguage(lang)}
          className={
            selectedLanguage === lang ? "bg-blue-500" : ""
          }
        >
          {lang}
        </button>
      ))}
      
      {/* Snippet grid */}
      <div className="grid grid-cols-3 gap-4">
        {filteredSnippets.map((snippet) => (
          <SnippetCard key={snippet._id} snippet={snippet} />
        ))}
      </div>
    </div>
  );
}
```

**Features**:
- Real-time search across all snippets
- Filter by language
- Grid/list view toggle
- Lazy load more snippets

### Step 7.2: Create Snippet Detail Page
**File**: `src/app/snippets/[id]/page.tsx`

```typescript
function SnippetDetailPage({ params }) {
  const snippet = useQuery(api.snippets.getSnippetById, {
    snippetId: params.id,
  });
  const comments = useQuery(api.snippets.getComments, {
    snippetId: params.id,
  });
  const isStarred = useQuery(api.snippets.isSnippetStarred, {
    snippetId: params.id,
  });
  
  return (
    <div>
      <NavigationHeader />
      
      {/* Snippet code */}
      <CodeBlock code={snippet.code} language={snippet.language} />
      
      {/* Star button */}
      <StarButton snippetId={params.id} isStarred={isStarred} />
      
      {/* Comments section */}
      <Comments
        snippetId={params.id}
        comments={comments}
      />
      
      {/* Comment form */}
      <CommentForm snippetId={params.id} />
    </div>
  );
}
```

**Features**:
- Display code with syntax highlighting
- Star/unstar functionality
- Comment thread
- Edit own comments

### Step 7.3: Create Profile Page
**File**: `src/app/profile/page.tsx` (297 lines)

```typescript
function ProfilePage() {
  const { user } = useUser();
  const [activeTab, setActiveTab] = useState("executions");
  
  // Fetch user stats
  const userStats = useQuery(api.codeExecutions.getUserStats, {
    userId: user?.id,
  });
  
  // Fetch execution history (paginated)
  const { results: executions, loadMore } = usePaginatedQuery(
    api.codeExecutions.getUserExecutions,
    { userId: user?.id },
    { initialNumItems: 5 }
  );
  
  // Fetch starred snippets
  const starredSnippets = useQuery(api.snippets.getStarredSnippets);
  
  return (
    <div>
      <NavigationHeader />
      
      {/* Profile header with stats */}
      <ProfileHeader userStats={userStats} user={user} />
      
      {/* Tabs */}
      <div className="flex gap-4">
        <button
          onClick={() => setActiveTab("executions")}
          className={activeTab === "executions" ? "text-blue-400" : ""}
        >
          Executions
        </button>
        <button
          onClick={() => setActiveTab("starred")}
          className={activeTab === "starred" ? "text-blue-400" : ""}
        >
          Starred
        </button>
      </div>
      
      {/* Tab content */}
      {activeTab === "executions" && (
        <div>
          {executions.map((execution) => (
            <ExecutionCard key={execution._id} execution={execution} />
          ))}
          {/* Load more button */}
        </div>
      )}
      
      {activeTab === "starred" && (
        <div>
          {starredSnippets.map((snippet) => (
            <SnippetCard key={snippet._id} snippet={snippet} />
          ))}
        </div>
      )}
    </div>
  );
}
```

**Features**:
- User statistics dashboard
- Execution history with pagination
- Starred snippets collection
- Pro subscription status display

### Step 7.4: Create Pricing Page
**File**: `src/app/pricing/page.tsx`

```typescript
function PricingPage() {
  const { user } = useUser();
  
  const handleUpgrade = async () => {
    // Redirect to Lemon Squeezy checkout
    window.location.href = process.env.NEXT_PUBLIC_LEMON_SQUEEZY_CHECKOUT_URL;
  };
  
  return (
    <div>
      <NavigationHeader />
      
      <div className="max-w-5xl mx-auto py-12">
        <h1>Unlock All Languages</h1>
        
        {/* Free tier */}
        <div className="p-8 border rounded-xl">
          <h2>Free</h2>
          <p>$0/month</p>
          <ul>
            <li>✓ JavaScript execution</li>
            <li>✓ Share snippets</li>
            <li>✓ Comment on snippets</li>
          </ul>
        </div>
        
        {/* Pro tier */}
        <div className="p-8 border-2 border-blue-500 rounded-xl">
          <h2>Pro</h2>
          <p>$9.99/month</p>
          <ul>
            <li>✓ 10+ programming languages</li>
            <li>✓ Unlimited executions</li>
            <li>✓ Advanced features</li>
          </ul>
          
          {user?.unsafeMetadata?.isPro ? (
            <p>You're already a pro member!</p>
          ) : (
            <button onClick={handleUpgrade}>
              Upgrade to Pro
            </button>
          )}
        </div>
      </div>
    </div>
  );
}
```

**Monetization**:
- Free tier: JavaScript only
- Pro tier: All 10+ languages
- Payment via Lemon Squeezy

---

## Phase 8: Frontend - Supporting Features

### Step 8.1: Create Language Selector
**File**: `src/app/(root)/_components/LanguageSelector.tsx`

```typescript
function LanguageSelector() {
  const { language, setLanguage } = useCodeEditorStore();
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <div className="relative">
      <button onClick={() => setIsOpen(!isOpen)}>
        <img src={`/${language}.png`} alt={language} />
        {language}
      </button>
      
      {isOpen && (
        <div className="absolute top-full mt-2 bg-[#1e1e2e] rounded-lg p-3">
          {Object.values(LANGUAGE_CONFIG).map((lang) => (
            <button
              key={lang.id}
              onClick={() => {
                setLanguage(lang.id);
                setIsOpen(false);
              }}
              className={
                language === lang.id ? "bg-blue-500" : ""
              }
            >
              <img src={lang.logoPath} alt={lang.label} />
              {lang.label}
            </button>
          ))}
        </div>
      )}
    </div>
  );
}
```

### Step 8.2: Create Theme Selector
**File**: `src/app/(root)/_components/ThemeSelector.tsx`

```typescript
function ThemeSelector() {
  const { theme, setTheme } = useCodeEditorStore();
  
  const themes = ["vs-dark", "vs-light", "hc-black"];
  
  return (
    <select
      value={theme}
      onChange={(e) => setTheme(e.target.value)}
    >
      {themes.map((t) => (
        <option key={t} value={t}>
          {t}
        </option>
      ))}
    </select>
  );
}
```

### Step 8.3: Create Navigation Header
**File**: `src/components/NavigationHeader.tsx`

```typescript
function NavigationHeader() {
  const { user } = useUser();
  const { signOut } = useClerk();
  
  return (
    <nav className="border-b border-gray-800 px-4 py-4">
      <div className="max-w-7xl mx-auto flex justify-between items-center">
        {/* Logo */}
        <Link href="/" className="text-2xl font-bold">
          Code Craft
        </Link>
        
        {/* Navigation links */}
        <div className="flex gap-8">
          <Link href="/">Editor</Link>
          <Link href="/snippets">Snippets</Link>
          <Link href="/pricing">Pricing</Link>
        </div>
        
        {/* User menu */}
        {user ? (
          <div className="flex gap-3">
            <Link href="/profile">
              {user.firstName}
            </Link>
            <button onClick={() => signOut()}>
              Logout
            </button>
          </div>
        ) : (
          <LoginButton />
        )}
      </div>
    </nav>
  );
}
```

### Step 8.4: Create Footer
**File**: `src/components/Footer.tsx`

```typescript
function Footer() {
  return (
    <footer className="border-t border-gray-800 mt-auto py-8">
      <div className="max-w-7xl mx-auto px-4 text-center text-gray-400">
        <p>© 2024 CodeForge. Built by developers, for developers.</p>
        <div className="flex gap-4 justify-center mt-4">
          <a href="/privacy">Privacy</a>
          <a href="/terms">Terms</a>
          <a href="https://github.com">GitHub</a>
        </div>
      </div>
    </footer>
  );
}
```

---

## Phase 9: Polish & Optimization

### Step 9.1: Add Animations
- Framer Motion for smooth transitions
- Button hover effects
- Page transitions
- Loading skeletons

### Step 9.2: Improve Error Handling
- Try/catch blocks in mutations
- User-friendly error messages
- Toast notifications for feedback
- Graceful fallbacks

### Step 9.3: Performance Optimization
- Code splitting per page
- Monaco Editor lazy loading
- Zustand selector memoization
- Image optimization

### Step 9.4: Testing
- Manual testing of all features
- Browser console for errors
- Network tab for API calls
- Cross-browser testing

---

## Phase 10: Deployment

### Step 10.1: Environment Setup
```bash
# Set environment variables in deployment platform
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=...
CLERK_SECRET_KEY=...
NEXT_PUBLIC_CONVEX_URL=...
CLERK_WEBHOOK_SECRET=...
LEMON_SQUEEZY_WEBHOOK_SECRET=...
```

### Step 10.2: Deploy Frontend
```bash
npm run build
npm start  # or deploy to Vercel/Netlify
```

### Step 10.3: Deploy Backend
```bash
npx convex deploy  # Deploys to Convex cloud
```

### Step 10.4: Configure Webhooks
- Clerk: Point webhook URL to deployment
- Lemon Squeezy: Point webhook URL to `/lemon-squeezy-webhook`

---

## Summary: Complete Build Timeline

1. **Setup** → Next.js + TypeScript + Tailwind + Dependencies
2. **Types** → Type definitions + Language configs
3. **Backend** → Convex schema + CRUD mutations
4. **Webhooks** → Clerk + Lemon Squeezy integration
5. **State** → Zustand store for editor state
6. **Providers** → Clerk + Convex wiring
7. **Core Components** → Editor + Output + Run button
8. **Community** → Snippets feed + Comments + Stars
9. **Profile** → User dashboard + Statistics
10. **Polish** → Animations + Error handling + Optimization

This structure ensures that:
- Backend is ready before frontend consumes it
- Types guide all implementations
- Reusable components build larger features
- Data flows predictably through Zustand → Convex → UI
