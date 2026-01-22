# CodeForge - Share & Execute Code Snippets

CodeForge is a web application where developers can write, execute, and share code snippets across 10+ programming languages in real-time.

## ✨ Features

- **Write Code**: Monaco Editor with syntax highlighting for 10+ languages
- **Execute Code**: Run code instantly using Piston API (JavaScript, Python, Rust, Go, Java, C++, and more)
- **Share Snippets**: Save code to community library for others to discover
- **Discover Code**: Browse snippets, search by language, star favorites
- **Community**: Comment on snippets, interact with other developers
- **Pro Features**: Unlock all languages with pro subscription
- **Profile**: Track execution history and statistics

## 🚀 Quick Start

### Prerequisites
- Node.js 18+
- npm or yarn

### Installation

1. **Set environment variables** in `.env.local`:
```env
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=your_clerk_key
CLERK_SECRET_KEY=your_clerk_secret
NEXT_PUBLIC_CONVEX_URL=your_convex_url
CLERK_WEBHOOK_SECRET=your_webhook_secret
LEMON_SQUEEZY_WEBHOOK_SECRET=your_payment_secret
```

2. **Install dependencies**:
```bash
npm install
```

3. **Run development server**:
```bash
npm run dev
```

## 🏗️ Architecture

- **Frontend**: React 18 + Next.js 15 + Zustand
- **Backend**: Convex (serverless DB)
- **Auth**: Clerk (OAuth + JWT)
- **Code Execution**: Piston API
- **Styling**: Tailwind CSS + Framer Motion
- **Payments**: Lemon Squeezy

## 📊 Key Technologies

**Languages**: JavaScript, Python, Rust, Go, Java, C++, Ruby, PHP, C#, Kotlin  
**Database**: Convex (WebSocket real-time sync)  
**Hosting**: Vercel (frontend) + Convex (backend)  
**Dependencies**: 180 npm packages (all latest)  
**Code**: 40+ files, fully typed, 0 errors  

## 🔑 Demonstration

This project demonstrates:
- Full-stack architecture (frontend + backend)
- Real-time database patterns (Convex WebSocket)
- OAuth & JWT authentication (Clerk)
- Microservice integration (Piston API)
- Webhook security (HMAC verification)
- Scalable component architecture
- Database optimization patterns
- Production deployment strategies

## 🐛 Development

```bash
npm run dev      # Start dev server (port 3000)
npm run build    # Build production bundle
npm start        # Start production server
npm run lint     # Run ESLint
npx convex codegen  # Generate Convex types
```

