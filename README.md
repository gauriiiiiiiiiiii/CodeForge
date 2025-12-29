<h1 align="center">✨  CodeForge - Craft Code, Forge Solutions ✨</h1>

### Highlights:
- 🚀 Tech stack: Next.js 15 + Convex + Clerk + TypeScript
- 💻 Online IDE with multi-language support (10 languages)
- 🎨 Customizable experience with 5 VSCode themes
- ✨ Smart output handling with Success & Error states
- 💎 Flexible pricing with Free & Pro plans
- 🤝 Community-driven code sharing system
- 🔍 Advanced filtering & search capabilities
- 👤 Personal profile with execution history tracking
- 📊 Comprehensive statistics dashboard
- ⚙️ Customizable font size controls
- 🔗 Webhook integration support
- 🌟 Professional deployment walkthrough

### Setup .env file

```js
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=
CLERK_SECRET_KEY=
CONVEX_DEPLOYMENT=
NEXT_PUBLIC_CONVEX_URL=
```

### Add these env to Convex Dashboard

```js
CLERK_WEBHOOK_SECRET=
LEMON_SQUEEZY_WEBHOOK_SECRET=
```

### Run the app

```shell
npm run dev
```

Important local notes:
- Create a `.env.local` file (do NOT commit) from the variables above.
- To run Convex locally, run `npx convex dev` in a separate terminal — this will provision a dev deployment and set `CONVEX_DEPLOYMENT` and `NEXT_PUBLIC_CONVEX_URL` in `.env.local`.
- If you don't have Clerk keys during local development the app will run in a limited mode (authentication disabled) — set `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` and `CLERK_SECRET_KEY` to enable full auth.

Deployment checklist:
- Set the environment variables in your hosting platform (Vercel, Render, etc.): `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`, `CLERK_SECRET_KEY`, `CONVEX_DEPLOYMENT`, `NEXT_PUBLIC_CONVEX_URL`, `CLERK_WEBHOOK_SECRET`, `LEMON_SQUEEZY_WEBHOOK_SECRET`.
- Deploy Convex functions with `npx convex deploy` and update `CONVEX_DEPLOYMENT` to the production deployment name.
- Rotate any keys you exposed while testing and keep secrets out of source control.

Quick build & run for production:
```bash
npm run build
npm run start
```
