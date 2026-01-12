# MACFIND — McMaster Lost & Found MVP

By Nehan Mohammed, Rian Sen Majumder, and Simon Akhter

Vite + React + TypeScript single-page app that matches the requested McMaster-themed lost & found marketplace:

- Landing hero with McMaster maroon/gold palette
- Marketplace `/lost` with search + filters, blurred thumbnails, AI metadata chips
- Found dashboard `/found` to post items (photo upload + location hint) and see your postings
- Item detail `/items/:id` with claim flow and chat entry
- Chat `/chat/:itemId/:otherUserId` with safe-meetup banner

If Supabase environment vars are missing, the app runs in **demo mode** with seeded data so the UX still works offline. Supabase integration is wired and will activate once env vars are provided.

## Getting started

```bash
npm install
npm run dev
```

Then open the printed local URL.

## Environment

Create a `.env` (or `.env.local`) with:

```
VITE_SUPABASE_URL=https://YOUR_PROJECT.supabase.co
VITE_SUPABASE_ANON_KEY=YOUR_ANON_KEY
```

Buckets/tables expected:

- Bucket `item-images`
- Tables: `items`, `item_images`, `ai_image_analyses`, `messages`
- Suggested schema tweaks: `alter table items add column if not exists category text;` and `alter table item_images add column if not exists is_blurred boolean default true;`

## Auth

Supabase Auth (Email magic link) with enforcement:

- Login prompts for a `@mcmaster.ca` email, then sends a magic link via `supabase.auth.signInWithOtp({ email, options: { emailRedirectTo } })`
- After session, users without `@mcmaster.ca` are signed out with an error message.
- Demo mode signs in a stub user when Supabase is not configured.

## AI (Gemini) prompt to keep outputs consistent

Use this JSON contract after uploading the image to storage:

```
Return JSON with keys:
category: one of ["electronics","clothing","id","keys","bag","book","water_bottle","sports","other"]
vague_label: short 2-5 word human-friendly label
tags: array of keywords (brand/model/object/material)
colors: array of main colors
materials: array of materials (optional)
confidence: number 0-1 for the category choice

If multiple items, pick the dominant lost item. Do not include any extra text.
```

Use the JSON to insert `ai_image_analyses` and update `items.vague_description`, `items.metadata`, `items.category`, and `items.best_ai_confidence`.

## Key routes/components

- `/` — Landing hero + CTAs (login, “I lost”, “I found”), blur messaging, AI note.
- `/lost` — Search + filters (category, location, time), marketplace grid with blurred cards.
- `/found` — Upload photo, location hint, optional description/quiz hint; shows “My posted items.”
- `/items/:id` — Blurred image, metadata, AI confidence, claim form, owner chat links.
- `/chat/:itemId/:otherUserId` — Message thread view with safe meetup suggestions.

## Notes

- Images are blurred via CSS (`filter: blur(12px)`) per MVP requirement.
- Demo content lives in `src/data/demo.ts` so the app feels alive without a backend.

## Backend (Gemini + Supabase ingest)

Location: `backend/`

What it does:
- POST `/api/upload-item` with multipart field `image` → Gemini vision extracts `{object_type, color}` → inserts into `items` table.

Setup:
1) Copy `backend/.env.example` to `backend/.env` and fill:
   - `SUPABASE_URL`
   - `SUPABASE_SERVICE_KEY` (service role; keep server-only)
   - `GEMINI_API_KEY`
   - `PORT` (optional, defaults to 3001)
2) Install deps and run:
   ```bash
   cd backend
   npm install
   npm run start   # or npm run dev for watch
   ```
3) Ensure Supabase table `items` has columns `object_type` and `color` (or adjust insert logic in `backend/server.js`).

Endpoint usage example (curl):
```bash
curl -F "image=@/path/to/photo.jpg" http://localhost:3001/api/upload-item
```
