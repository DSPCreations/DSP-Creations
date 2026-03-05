# AI Resume + Job Application Assistant (Weekend MVP)

This document tailors your blueprint for a **code-first build with Next.js + Supabase + OpenAI** so you can ship quickly and still scale.

## 1) Scope for V1 (ship this first)

### Must-have
- Auth (email + magic link or OTP)
- Profile builder (personal, education, experience, projects, skills)
- Optional Job Description input
- AI generation:
  - ATS resume
  - Cover letter
- Resume editor with live preview
- PDF download
- Job application tracker (manual add/update)

### Exclude from V1
- Auto-apply flows
- LinkedIn import
- Salary predictor
- Interview prep generation

---

## 2) Recommended stack (fast dev + maintainable)

- **Frontend:** Next.js (App Router) + Tailwind CSS + shadcn/ui
- **Backend:** Next.js Route Handlers / Server Actions
- **Auth + DB:** Supabase (Postgres + Row Level Security)
- **AI:** OpenAI Responses API
- **PDF:** Playwright print-to-PDF from server-rendered HTML
- **Billing (v1.1):** Razorpay or Stripe

---

## 3) Data model (Supabase SQL-ready)

```sql
create table users (
  id uuid primary key references auth.users(id) on delete cascade,
  email text unique not null,
  name text,
  plan text default 'free' check (plan in ('free','pro','premium')),
  created_at timestamptz default now()
);

create table profiles (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references users(id) on delete cascade,
  phone text,
  location text,
  linkedin text,
  portfolio text,
  summary text,
  skills text[] default '{}',
  languages text[] default '{}',
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

create table education (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references users(id) on delete cascade,
  degree text not null,
  college text not null,
  year text,
  grade text,
  created_at timestamptz default now()
);

create table experience (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references users(id) on delete cascade,
  title text not null,
  company text not null,
  start_date date,
  end_date date,
  responsibilities text[] default '{}',
  created_at timestamptz default now()
);

create table projects (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references users(id) on delete cascade,
  name text not null,
  tech_stack text[] default '{}',
  description text,
  link text,
  created_at timestamptz default now()
);

create table job_applications (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references users(id) on delete cascade,
  company text not null,
  role text not null,
  link text,
  status text default 'Applied' check (status in ('Applied','Interview','Offer','Rejected')),
  date_applied date,
  notes text,
  created_at timestamptz default now()
);

create table generated_docs (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references users(id) on delete cascade,
  type text not null check (type in ('resume','cover_letter')),
  jd_text text,
  output_json jsonb,
  output_html text,
  pdf_url text,
  created_at timestamptz default now()
);
```

### RLS baseline
- Enable RLS on every table.
- Policy pattern: `user_id = auth.uid()` for select/insert/update/delete.
- `users` table policy: `id = auth.uid()`.

---

## 4) Pages and routes (exact MVP IA)

- `/` Landing
- `/auth` Login / signup
- `/dashboard` Overview cards
- `/profile` Multi-step profile builder
- `/generate` Resume/Cover generation inputs
- `/editor/[docId]` JSON section editor + live HTML preview
- `/tracker` Job tracker table
- `/api/generate/resume`
- `/api/generate/cover-letter`
- `/api/score`
- `/api/pdf/[docId]`

---

## 5) Prompt pack (production-safe)

### Prompt A — ATS Resume JSON

**System**
```
You are an expert ATS resume writer.
Return valid JSON only. Do not wrap in markdown.
Never invent employers, dates, projects, or degrees.
```

**User**
```
Create an ATS-friendly resume from candidate data:
{PROFILE_JSON}

Job description (optional):
{JD_TEXT}

Rules:
- 0-5 years: target one page; otherwise max two pages.
- Use action verbs and measurable outcomes.
- Add JD keywords naturally; avoid keyword stuffing.
- Reverse chronological order for experience.
- Maximum 5 bullets per experience entry.

Return strictly this shape:
{
  "header": {"name":"","email":"","phone":"","location":"","links":[]},
  "summary": "",
  "skills": {"technical":[],"tools":[],"soft":[]},
  "experience": [{"company":"","title":"","start":"","end":"","bullets":[]}],
  "projects": [{"name":"","bullets":[],"tech":[]}],
  "education": [{"degree":"","school":"","year":"","details":[]}]
}
```

### Prompt B — Cover letter

**System**
```
You are a professional career coach.
```

**User**
```
Write a tailored cover letter.
Candidate: {PROFILE_JSON}
Role: {ROLE}
Company: {COMPANY}
JD: {JD_TEXT}
Tone: confident, concise, human.
Length: 250-350 words.
Do not fabricate achievements.
```

### Prompt C — Resume score + fixes

**System**
```
You are an ATS evaluator.
Return concise, actionable feedback.
```

**User**
```
Score this resume vs JD from 0-100.
Return JSON:
{
  "score": 0,
  "missing_keywords": [],
  "rewrite_suggestions": [],
  "formatting_issues": []
}
Resume: {RESUME_TEXT}
JD: {JD_TEXT}
```

---

## 6) Guardrails and reliability

- Validate JSON schema on AI response.
- Auto-retry once on invalid JSON.
- Enforce max 5 bullets per role in backend.
- Basic profanity / unsafe text check before rendering.
- Hard rule in prompts: no fabricated experience.
- Keep model temperature low (0.2–0.4) for consistency.

---

## 7) Template system (scalable from day 1)

- Store template as HTML + CSS partials.
- Render pipeline: `profile + ai_json -> normalized view model -> template -> PDF`.
- ATS template rules:
  - single column
  - no icons/tables
  - standard fonts (Arial/Inter)
  - semantic headings and bullet lists

---

## 8) 2-day build checklist

### Day 1
- [ ] Set up Next.js app + Supabase auth
- [ ] Create DB schema + RLS
- [ ] Build profile builder forms
- [ ] Implement resume generation API
- [ ] Save generated JSON + render preview

### Day 2
- [ ] Build editor with section-level edits
- [ ] Add PDF generation endpoint
- [ ] Build job tracker CRUD
- [ ] Add free-tier limit (1 download)
- [ ] Add basic analytics (events: generate, download, apply-add)

---

## 9) Monetization starter

- **Free:** 1 resume download, 1 template
- **Pro (₹499/month):** unlimited downloads, JD optimization, cover letters, scoring
- **Premium (₹999/month):** multiple templates + v2 features

North-star target: `2000 * ₹499 ≈ ₹10L MRR`.

---

## 10) Next implementation step

If you want, the next artifact to add is:
1. `schema.sql` (direct Supabase migration)
2. `lib/prompts.ts` (typed prompt pack)
3. `app/api/generate/resume/route.ts` (JSON-validated generator)
4. `components/templates/ats-basic.tsx` (print-safe HTML template)
