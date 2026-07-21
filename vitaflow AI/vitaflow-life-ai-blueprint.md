# VitaFlow Life AI — Product Blueprint

> Scope note: A 13-domain app (coaching, planning, health, mental wellness, finance, relationships, spirituality, learning, analytics, gamification, community, wearables, voice AI) is a full platform, not an MVP. Building all of it "production-ready" in one pass produces 13 shallow features instead of one that works. This blueprint gives you the full architecture so nothing is blocked later, but scopes the **buildable MVP down to the 20% that proves the core loop**: capture → AI insight → action → score movement. Everything else is architected but sequenced into the roadmap.

---

## 1. Core Loop (the thing that must feel great before anything else does)

```
Quick daily check-in (60 sec)
   → mood + energy + sleep + top 3 priorities
   → AI generates: 1 insight + 1 recommendation + today's plan
   → user acts (completes a block, logs water, journals)
   → Life Balance Score updates + streak increments
   → tomorrow's check-in references yesterday's data ("You said X, how did it go?")
```

If this loop isn't addictive and useful in week 1, no amount of extra modules fixes it. MVP = this loop + Health basics + Mental Wellness basics + the Analytics dashboard that visualizes it. Finance/Relationships/Spirituality/Learning/Community/Gamification are v2–v4 (see roadmap).

---

## 2. MVP Feature Scope (Phase 1 — what actually gets built)

| Included in MVP | Deferred to v1.1–v2 |
|---|---|
| Auth (email, Google, Apple) | Phone OTP |
| Daily check-in (mood/energy/sleep) | Full wearable integration (HealthKit/Health Connect read is v1.1, full sync later) |
| AI Life Coach: daily insight + recommendation | AI voice coach |
| Smart planner: 3 priorities + Pomodoro + deep work block | Full calendar 2-way sync |
| Health basics: water, sleep, steps (manual + HealthKit read) | Nutrition log, heart rate |
| Mental Wellness: gratitude journal, mood calendar, guided breathing | Full meditation library |
| Life Balance Score + 4 sub-scores (Health, Work, Mental, Overall) | Financial/Relationship/Spiritual scores |
| Streaks + XP + levels (lightweight gamification) | Leaderboards, community, challenges |
| Free vs Premium paywall (Stripe/RevenueCat) | Full premium AI meal/workout plans |
| Dark/light mode, offline-first cache | Multi-language |

Finance, Relationship, and Spiritual modules are **schema-complete** below so they slot in later without a data migration.

---

## 3. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  React Native (Expo, TypeScript)                            │
│  - Expo Router (file-based nav)                             │
│  - Zustand (client state) + TanStack Query (server cache)   │
│  - WatermelonDB / SQLite (offline-first local store)         │
│  - Reanimated 3 + Moti (animations)                          │
└───────────────┬───────────────────────────────────────────┘
                │  HTTPS / WSS
┌───────────────▼───────────────────────────────────────────┐
│  Supabase                                                    │
│  ├─ Postgres (Row Level Security per user_id)                │
│  ├─ Auth (Google / Apple / Email / Phone OTP)                │
│  ├─ Edge Functions (Deno) — AI orchestration, scoring, cron   │
│  ├─ Realtime (check-in sync, community later)                 │
│  └─ Storage (avatars, journal attachments)                    │
└───────────────┬───────────────────────────────────────────┘
                │
   ┌────────────┼─────────────┬─────────────┬──────────────┐
   ▼            ▼             ▼             ▼              ▼
 OpenAI API   Stripe /     Firebase FCM   Firebase        Apple Health /
 (coaching,   RevenueCat   (push/smart    Analytics +     Google Health
 insights,    (payments)   reminders)     Mixpanel        Connect (read)
 scoring)
```

**Why this stack works for MVP speed:** Supabase gives you Postgres + Auth + Edge Functions + Realtime + Storage as one bill and one RLS model, so a solo/small team isn't gluing together five vendors. Edge Functions call OpenAI server-side so the API key never ships in the app bundle.

**Offline-first approach:** all check-ins, journal entries, and tracker logs write to local SQLite first (optimistic UI), then a background sync queue pushes to Supabase and resolves conflicts last-write-wins by `updated_at`. AI insight generation requires connectivity and queues gracefully when offline.

---

## 4. Database Schema (Postgres / Supabase)

Full schema — MVP tables are marked **[MVP]**, others are **[v2+]** but included now so relations don't need to change later. See `/supabase/migrations/0001_init.sql` for the runnable SQL.

### Core
- `profiles` **[MVP]** — user_id, display_name, timezone, onboarding_complete, subscription_tier
- `checkins` **[MVP]** — daily mood, energy, sleep_hours, top priorities (jsonb), created_at
- `ai_insights` **[MVP]** — generated insight text, category, confidence, source_data_range, dismissed
- `life_scores` **[MVP]** — date, overall_score, health_score, work_score, mental_score, finance_score(v2), relationship_score(v2), spiritual_score(v2)

### Planner
- `tasks` **[MVP]** — title, priority, due_date, estimated_minutes, status, is_deep_work
- `pomodoro_sessions` **[MVP]** — task_id, started_at, duration_minutes, completed

### Health
- `health_logs` **[MVP]** — type (water|sleep|exercise|weight|steps), value, unit, logged_at
- `wearable_syncs` **[v1.1]** — provider, external_id, raw_payload

### Mental Wellness
- `journal_entries` **[MVP]** — type (gratitude|reflection|free), content, mood_tag, is_private
- `breathing_sessions` **[MVP]** — pattern, duration_seconds

### Finance **[v2]**
- `expenses`, `budgets`, `savings_goals`

### Relationships **[v2]**
- `important_people`, `check_in_reminders`, `acts_of_kindness`

### Spiritual **[v2]**
- `spiritual_practices`, `spiritual_journal`

### Learning **[v2]**
- `learning_items`, `reading_log`, `skill_roadmaps`

### Gamification
- `user_xp` **[MVP]** — total_xp, level, current_streak, longest_streak
- `badges`, `user_badges` **[MVP-lite]**

### Community **[v3]**
- `groups`, `group_members`, `challenges`, `leaderboard_entries`

All tables use `user_id uuid references auth.users` with **Row Level Security**: `USING (auth.uid() = user_id)` on every table — no cross-user reads without an explicit `groups`/`community` join table.

---

## 5. AI System Design

Three distinct AI jobs, kept separate so each can be tuned/evaluated independently:

1. **Daily Insight Generator** (Edge Function, cron 1x/day per user, or on-demand)
   - Input: last 14 days of checkins + health_logs + tasks completion rate
   - Output (structured JSON via OpenAI function-calling/JSON mode): `{ insight, category, recommendation, confidence }`
   - Pattern detection prompts specifically for the "Your stress increases every Tuesday" style outputs: aggregate by day-of-week, by pre/post calendar-event windows, by sleep-vs-mood correlation — computed in SQL/edge function, *then* handed to the LLM to phrase in plain language (don't let the LLM invent statistics — compute them, then narrate them).

2. **Burnout Detection** (rules + LLM hybrid)
   - Deterministic trigger: declining mood trend (3+ days) + sleep < 6hrs (3+ days) + task completion drop > 30% → flags `burnout_risk = true`
   - LLM only used to phrase the supportive message and suggest one concrete action, never to decide the flag itself (keeps it auditable and safe).

3. **Life Balance Score Engine** (deterministic, NOT an LLM)
   - Each sub-score is a weighted formula over the last 7/30 days (consistency matters more than single-day peaks). Scores must be explainable — show the user the formula, not a black box. Example:
     `health_score = 0.3*sleep_consistency + 0.3*water_goal_hit_rate + 0.2*exercise_freq + 0.2*steps_goal_hit_rate`
   - `overall_score` = weighted average of active sub-scores, re-weighted automatically as modules (finance, relationships) come online.

**Safety rail:** insight/coaching prompts include a system instruction to never give clinical mental-health, medical, or financial advice — only general wellness framing — and to surface crisis resources if language suggests self-harm (route to a static, non-AI-generated crisis card, not a generated response).

---

## 6. User Journey (MVP)

```
Onboarding (90 sec, skippable):
  Welcome → "What areas matter most to you?" (pick 3-5 of 7 domains)
  → baseline check-in (mood/energy/sleep) → notification permission
  → first AI insight shown immediately ("Here's what I noticed already")
  → paywall shown softly (7-day trial), not blocking

Day 1-7 (habit formation window):
  Morning: push notification → 60-sec check-in → today's AI plan
  Throughout day: log water/tasks/breathing as needed (frictionless, 1-tap)
  Evening: gratitude prompt (2 lines) → streak +1, XP +10

Day 7:
  Weekly recap screen: score trend line, 1 standout insight, 1 badge unlocked
  Soft paywall re-prompt tied to a real insight ("Unlock the full pattern behind your Tuesday stress")

Ongoing:
  Analytics dashboard becomes the "why I keep opening this app" screen
  Score movement + specific causal insights = retention driver
```

---

## 7. Folder Structure

```
life-balance-ai/
├── app/                          # Expo Router (file-based routing)
│   ├── (auth)/
│   │   ├── sign-in.tsx
│   │   ├── sign-up.tsx
│   │   └── onboarding.tsx
│   ├── (tabs)/
│   │   ├── index.tsx             # Home / Today
│   │   ├── planner.tsx
│   │   ├── health.tsx
│   │   ├── wellness.tsx
│   │   ├── insights.tsx          # Analytics dashboard
│   │   └── profile.tsx
│   ├── checkin/[step].tsx
│   ├── journal/[id].tsx
│   └── _layout.tsx
├── src/
│   ├── components/
│   │   ├── ui/                   # Button, Card, Sheet, ScoreRing, etc.
│   │   └── charts/
│   ├── features/
│   │   ├── checkin/
│   │   ├── planner/
│   │   ├── health/
│   │   ├── wellness/
│   │   ├── insights/
│   │   └── gamification/
│   ├── lib/
│   │   ├── supabase.ts
│   │   ├── queryClient.ts
│   │   ├── offlineSync.ts
│   │   └── theme.ts               # light/dark tokens
│   ├── stores/                    # Zustand stores
│   ├── hooks/
│   └── types/
├── supabase/
│   ├── migrations/
│   │   └── 0001_init.sql
│   └── functions/
│       ├── generate-daily-insight/
│       ├── compute-life-scores/
│       ├── burnout-check/
│       └── stripe-webhook/
├── docs/
├── app.json
├── package.json
└── tsconfig.json
```

---

## 8. API / Edge Function Endpoints

| Endpoint (Edge Function) | Method | Purpose |
|---|---|---|
| `/generate-daily-insight` | POST | Pulls last 14d data, calls OpenAI, writes to `ai_insights` |
| `/compute-life-scores` | POST (cron nightly) | Recomputes all sub-scores + overall score for active users |
| `/burnout-check` | POST (cron nightly) | Runs rule engine, writes flag + notification trigger |
| `/checkin` | POST | Validates + writes check-in, triggers XP award |
| `/tasks` | GET/POST/PATCH/DELETE | Planner CRUD (could also be direct Supabase client calls via RLS — Edge Function only needed if server logic required) |
| `/stripe-webhook` | POST | Handles subscription lifecycle events |
| `/stripe-create-checkout` | POST | Creates Stripe/RevenueCat checkout session |
| `/push-notification-dispatch` | POST (cron) | Smart reminder timing based on user's historical active hours |

Most simple CRUD (tasks, health_logs, journal_entries) should go **directly through the Supabase client with RLS**, not custom Edge Functions — reserve Edge Functions for anything touching OpenAI, Stripe, or cross-table aggregation.

---

## 9. Monetization Strategy

**Model:** Freemium subscription, standard for wellness apps (Headspace/Calm/Fabulous pattern).

| Tier | Price (anchor) | Includes |
|---|---|---|
| Free | $0 | Daily check-in, basic planner, health/mental wellness logging, 1 AI insight/day, basic score |
| Premium | $9.99/mo or $59.99/yr | Unlimited AI coaching, full analytics + historical trends, personalized meal/workout generation, wearable sync, AI voice coach, priority insights |
| Premium+ (later) | $14.99/mo | Community challenges, accountability partner matching, human-coach add-ons |

- 7-day free trial, soft paywall (show value before asking — first insight always free, deeper pattern analysis behind paywall).
- Annual plan pushed at 50%+ discount to lock LTV early.
- B2B2C channel later: corporate wellness (sell to HR teams for employee burnout prevention) — high-margin expansion, not MVP.

**Unit economics target:** CAC via organic/content + App Store search should stay under $15; at $60/yr LTV with 12-month average retention, aim for 4:1 LTV:CAC before paid acquisition scales.

---

## 10. 12-Month Product Roadmap

| Phase | Months | Focus |
|---|---|---|
| **Phase 0 — MVP build** | 0–2 | Core loop, health/mental wellness basics, scoring engine, paywall |
| **Phase 1 — Retention** | 3–4 | Push notification tuning, streak/XP polish, weekly recap, HealthKit/Health Connect read sync |
| **Phase 2 — Finance + Relationships** | 5–6 | Expense tracker, budget planner, relationship check-in reminders, sub-scores go live |
| **Phase 3 — Learning + Spiritual** | 7–8 | Reading/course tracker, spiritual journal, habit streak polish across all domains |
| **Phase 4 — Community** | 9–10 | Accountability partners, private groups, challenges, leaderboards |
| **Phase 5 — Premium AI depth** | 11 | AI voice coach, personalized meal/workout generation, full wearable 2-way sync |
| **Phase 6 — Scale** | 12 | Corporate/B2B2C pilot, localization, performance/cost optimization on AI spend |

---

## 11. Marketing Plan (lean, pre-Series-A)

1. **Content/SEO:** founder-led content on burnout, productivity science, "life balance score" as a shareable/quiz-like hook (build a free web quiz that captures email + drives app installs).
2. **Short-form video:** before/after score-improvement stories, "AI told me this about my Tuesdays" style hooks — this app's insights are inherently shareable screenshots.
3. **App Store Optimization:** target keywords around "burnout," "life balance," "AI coach," "habit tracker" — screenshots should show the score ring + one real insight, not generic feature lists.
4. **Community seeding:** Reddit (r/productivity, r/burnout, r/getdisciplined), Indie Hackers build-in-public thread for founder credibility + early adopters.
5. **Referral loop:** "invite an accountability partner" ties directly to Phase 4 community feature — build the referral mechanic early even before groups ship (waitlist + invite credits).
6. **Partnerships:** wellness newsletters, therapist/coach affiliates (they recommend the tracking, you don't claim to replace them).

---

## 12. Investor Pitch Deck Outline

1. **Title / Vision** — "VitaFlow Life AI: the AI coach for the life you're actually living, not just your calendar."
2. **Problem** — Burnout, fragmented point-solutions (5 separate apps for sleep/finance/mood/tasks), no one shows the *connections* between domains.
3. **Solution** — One AI that sees mood + sleep + work + money + relationships together and tells you the causal pattern, not just the data.
4. **Product demo** — Screenshots: check-in → insight → score movement.
5. **Market size** — Wellness app market TAM/SAM/SOM (cite current figures at fundraise time, don't hardcode stale numbers in the deck).
6. **Business model** — Freemium subscription + future B2B2C.
7. **Traction** — MVP metrics: D7/D30 retention, insight engagement rate, conversion to paid.
8. **Competitive landscape** — vs. single-domain apps (Calm, YNAB, Headspace, Rise) — differentiation is cross-domain correlation, not another single tracker.
9. **Go-to-market** — content + ASO + referral loop (see Marketing Plan).
10. **Team** — founder/domain expertise.
11. **Financials/ask** — burn rate, runway, use of funds (engineering, AI cost optimization, growth).
12. **Roadmap** — 12-month plan (see above), condensed to 4 milestones.

---

## 13. MVP Development Roadmap (build order, ~8–10 weeks, small team)

| Week | Deliverable |
|---|---|
| 1 | Supabase project + schema + RLS, Expo app skeleton, auth (email/Google/Apple) |
| 2 | Onboarding flow, daily check-in UI + local-first save |
| 3 | Planner (tasks, priorities, Pomodoro timer) |
| 4 | Health basics (water/sleep/steps manual + HealthKit/Health Connect read) |
| 5 | Mental wellness (gratitude journal, mood calendar, guided breathing) |
| 6 | AI insight Edge Function + burnout rule engine + score computation |
| 7 | Analytics dashboard (score ring, trend charts, insight feed) |
| 8 | Gamification (XP/streaks/badges), dark/light theming polish, animations |
| 9 | Stripe/RevenueCat paywall, push notifications (FCM), offline sync hardening |
| 10 | QA, accessibility pass, App Store / Play Store submission prep |

---

## 14. Source Code

Full production source for every module isn't realistic to hand-write in one pass — but a working, runnable foundation is included alongside this document:
- `supabase/migrations/0001_init.sql` — complete MVP schema with RLS
- `supabase/functions/generate-daily-insight/index.ts` — real OpenAI-calling Edge Function
- `supabase/functions/compute-life-scores/index.ts` — deterministic scoring engine
- `src/lib/supabase.ts`, `src/lib/theme.ts` — app foundation
- `app/(tabs)/index.tsx`, `app/checkin/[step].tsx`, `src/components/ui/ScoreRing.tsx` — working core-loop screens

Everything else in the folder structure above is the scaffold to extend module-by-module following the same patterns.
