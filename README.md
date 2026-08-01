# FindIt On Campus

## Why the original download failed to build

Your component uses **Svelte 5 runes** (`$state`, `$derived`) and the
**Svelte 5 legacy-event-compat helpers** (`onclick={...}`, `stopPropagation`,
`createBubbler` from `'svelte/legacy'`). The `svelte.dev` playground always
runs the newest Svelte 5 compiler, so it Just Works there. But if you
scaffolded your own project with an older template (or one that pulled in
`svelte@4`), the compiler doesn't understand runes or the legacy-compat
imports, and `npm run build` throws a wall of "svelte 4 vs 5" errors.

This project pins known-good, mutually compatible versions:

```
svelte                     ^5.20.0
@sveltejs/kit               ^2.69.3
@sveltejs/vite-plugin-svelte ^5.0.3
@sveltejs/adapter-vercel     ^6.3.4
```

Your component is dropped in unchanged at `src/routes/+page.svelte`.

## Run it locally

```bash
npm install
npm run dev       # http://localhost:5173
npm run build     # sanity-check the production build before pushing
npm run preview   # serve the built output locally
```

If `npm run build` still errors, copy the **exact error text** — it'll say
which file/line and usually names the specific rune or directive it choked
on, which narrows it down fast.

## Deploy to Vercel

1. Push this folder to a new GitHub repo (root of the repo = root of this
   project, i.e. `package.json` should be at the repo root).
2. Go to [vercel.com/new](https://vercel.com/new) → import the repo.
3. Framework preset should auto-detect as **SvelteKit**. Leave build command
   as `npm run build` / output as auto-detected.
4. Deploy. No environment variables are needed for this app as it currently
   stands — the "Microsoft sign-in" is mocked client-side and all data is
   in-memory (resets on page reload).

## Known limitations to fix before real use

These aren't build errors — the app compiles and runs — but worth knowing:

- **No real backend**: `items`, `user`, comments, etc. all live in browser
  memory (`$state`) and vanish on refresh. You'll want a database (e.g.
  Vercel Postgres, Supabase) and real API routes (`+server.js` files in
  SvelteKit) before this is usable by more than one person at a time.
- **Mock sign-in**: `mockMicrosoftSignIn()` fakes a login. Real Microsoft
  auth means wiring up an OAuth flow (e.g. via `@auth/sveltekit` with the
  Microsoft Entra ID provider).
- **Images stored as base64 data URLs** in memory — fine for a prototype,
  but for production you'd upload to real storage (Vercel Blob, S3, etc.)
  and store URLs instead.
