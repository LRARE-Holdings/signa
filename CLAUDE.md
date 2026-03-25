# CLAUDE.md — Signa

## What is Signa?

Signa is a proof-of-acknowledgement platform. It lets organisations send a document and receive verifiable, timestamped proof that the recipient opened it and confirmed they have read it.

The recipient does not need to create an account, download an app, or log into a platform. They receive a link, open the document, and acknowledge it. Signa records the event with a timestamp, IP address, and device metadata, then issues a verifiable proof-of-acknowledgement record to the sender.

Signa is NOT an e-signature tool. It does not collect signatures, initials, or any form of legal signature. It is not DocuSign, HelloSign, or BreezeDoc. It does not handle templates, clause libraries, version control, or negotiation workflows.

**One-liner:** "Signa proves that someone received and acknowledged a document."

**Domain:** usesigna.co
**Parent company:** Pellar Holdings Ltd (pellar.co.uk)

---

## Tech stack

| Layer              | Technology                        |
|--------------------|-----------------------------------|
| Framework          | Next.js 14+ (App Router)          |
| Language           | TypeScript (strict mode)          |
| Styling            | Tailwind CSS                      |
| Database           | Supabase (PostgreSQL)             |
| Auth               | Supabase Auth (email/password + passkey/OAuth/authenticator apps) |
| File storage       | Supabase Storage                  |
| Payments           | Stripe                            |
| Transactional email| Resend                            |
| Hosting            | Vercel                            |
| Document rendering | pdf.js (browser-side PDF viewer)  |
| Document conversion| DOCX → PDF server-side on upload (LibreOffice headless or equivalent) |

---

## Project structure

```
signa/
├── app/
│   ├── (auth)/
│   │   ├── login/
│   │   ├── signup/
│   │   └── forgot-password/
│   ├── (dashboard)/
│   │   ├── layout.tsx              # Authenticated layout with sidebar
│   │   ├── page.tsx                # Dashboard home (recent activity)
│   │   ├── documents/
│   │   │   ├── page.tsx            # Document list with status filters
│   │   │   ├── [id]/
│   │   │   │   └── page.tsx        # Single document detail + recipient tracking
│   │   │   └── new/
│   │   │       └── page.tsx        # Upload + add recipients + send
│   │   ├── settings/
│   │   │   └── page.tsx            # Account, billing, team
│   │   └── billing/
│   │       └── page.tsx            # Stripe subscription management
│   ├── (marketing)/
│   │   ├── page.tsx                # Landing page (usesigna.co)
│   │   ├── pricing/
│   │   └── about/
│   ├── d/
│   │   └── [token]/
│   │       └── page.tsx            # Recipient document view + acknowledge (PUBLIC)
│   ├── api/
│   │   ├── documents/
│   │   │   ├── upload/route.ts     # Handle file upload + conversion
│   │   │   └── send/route.ts       # Trigger send to recipients
│   │   ├── acknowledge/
│   │   │   └── route.ts            # Record acknowledgement event
│   │   ├── reminders/
│   │   │   └── route.ts            # Cron-triggered reminder emails
│   │   ├── proof/
│   │   │   └── [id]/route.ts       # Generate proof-of-acknowledgement PDF
│   │   ├── webhooks/
│   │   │   └── stripe/route.ts     # Stripe webhook handler
│   │   └── cron/
│   │       └── reminders/route.ts  # Vercel cron endpoint for reminders
│   ├── layout.tsx                  # Root layout
│   └── globals.css                 # Tailwind base + brand tokens
├── components/
│   ├── ui/                         # Generic UI primitives (button, input, card, badge, etc.)
│   ├── dashboard/                  # Dashboard-specific components
│   ├── documents/                  # Document-related components (upload, list, detail)
│   ├── recipient/                  # Recipient-facing components (viewer, acknowledge button)
│   └── marketing/                  # Landing page components
├── lib/
│   ├── supabase/
│   │   ├── client.ts               # Browser Supabase client
│   │   ├── server.ts               # Server Supabase client
│   │   └── admin.ts                # Service role client (for server-only operations)
│   ├── stripe/
│   │   └── client.ts               # Stripe SDK setup
│   ├── resend/
│   │   └── client.ts               # Resend SDK setup
│   ├── pdf/
│   │   └── convert.ts              # DOCX → PDF conversion logic
│   ├── proof/
│   │   └── generate.ts             # Proof-of-acknowledgement PDF generation
│   ├── tokens.ts                   # Secure token generation for recipient links
│   ├── metadata.ts                 # IP, user agent, device metadata capture
│   └── constants.ts                # App-wide constants
├── types/
│   └── index.ts                    # Shared TypeScript types
├── supabase/
│   └── migrations/                 # SQL migration files
├── public/
│   ├── logo.svg
│   └── og-image.png
├── tailwind.config.ts
├── next.config.ts
├── tsconfig.json
├── package.json
└── CLAUDE.md
```

---

## Data model

### `users` (managed by Supabase Auth)

Supabase Auth handles the users table. Extended profile data lives in `profiles`.

### `profiles`

| Column       | Type      | Notes                          |
|--------------|-----------|--------------------------------|
| id           | uuid (PK) | References auth.users(id)      |
| full_name    | text      |                                |
| company_name | text      | nullable                       |
| plan         | text      | 'starter' / 'professional' / 'business' |
| stripe_customer_id | text | nullable                    |
| stripe_subscription_id | text | nullable               |
| created_at   | timestamptz |                              |
| updated_at   | timestamptz |                              |

### `documents`

| Column       | Type      | Notes                          |
|--------------|-----------|--------------------------------|
| id           | uuid (PK) |                                |
| user_id      | uuid (FK) | References profiles(id)        |
| title        | text      | Display name                   |
| original_filename | text |                                |
| original_path | text     | Supabase Storage path (original file) |
| pdf_path     | text      | Supabase Storage path (converted PDF) |
| file_type    | text      | 'pdf' / 'docx' / 'image'      |
| status       | text      | 'draft' / 'sent' / 'partially_acknowledged' / 'fully_acknowledged' |
| reminder_enabled | boolean | Default true                 |
| reminder_interval_days | int | Default 3                  |
| expires_at   | timestamptz | nullable                    |
| created_at   | timestamptz |                              |
| updated_at   | timestamptz |                              |

### `recipients`

| Column       | Type      | Notes                          |
|--------------|-----------|--------------------------------|
| id           | uuid (PK) |                                |
| document_id  | uuid (FK) | References documents(id)       |
| email        | text      |                                |
| name         | text      |                                |
| token        | text (unique) | Secure random token for the recipient link |
| status       | text      | 'pending' / 'sent' / 'opened' / 'acknowledged' |
| sent_at      | timestamptz | nullable                    |
| opened_at    | timestamptz | nullable                    |
| acknowledged_at | timestamptz | nullable                 |
| ip_address   | text      | nullable, captured on acknowledge |
| user_agent   | text      | nullable, captured on acknowledge |
| device_info  | jsonb     | nullable, parsed device metadata |
| reminder_count | int     | Default 0                      |
| last_reminder_at | timestamptz | nullable               |
| created_at   | timestamptz |                              |

### `audit_log`

| Column       | Type      | Notes                          |
|--------------|-----------|--------------------------------|
| id           | uuid (PK) |                                |
| document_id  | uuid (FK) |                                |
| recipient_id | uuid (FK) | nullable                       |
| event        | text      | 'created' / 'sent' / 'opened' / 'acknowledged' / 'reminder_sent' / 'proof_downloaded' |
| metadata     | jsonb     | IP, user agent, any additional context |
| created_at   | timestamptz |                              |

---

## Row Level Security (RLS)

Every table must have RLS enabled. Policies:

- **profiles**: Users can read/update their own profile only.
- **documents**: Users can CRUD their own documents only.
- **recipients**: Users can CRUD recipients on their own documents only.
- **audit_log**: Users can read audit log entries for their own documents only. Insert via service role only.

The recipient-facing routes (`/d/[token]`) use the Supabase service role client (admin) to look up recipient records by token. Recipients are never authenticated — the token IS the access control.

---

## Key flows

### Sender: Upload and send

1. Sender uploads a file (PDF, DOCX, or image) via `/api/documents/upload`
2. File is stored in Supabase Storage under `documents/{user_id}/{document_id}/original`
3. If DOCX, server converts to PDF and stores under `documents/{user_id}/{document_id}/rendered.pdf`
4. If image, server converts to PDF and stores similarly
5. Sender adds recipients (name + email) on the document detail page
6. Sender clicks "Send" → hits `/api/documents/send`
7. For each recipient, a secure token is generated and stored
8. Resend sends an email to each recipient with a link: `usesigna.co/d/{token}`
9. Recipient status set to 'sent', audit log entry created

### Recipient: View and acknowledge

1. Recipient clicks link → `/d/[token]`
2. Page loads: look up recipient by token (service role), fetch document PDF path
3. Document renders in-browser via pdf.js (full document, scrollable)
4. Below the viewer: acknowledgement section
5. Fixed text: "I acknowledge I have received and read this document."
6. Single button: "Acknowledge"
7. On click → POST to `/api/acknowledge` with token
8. Server records: timestamp, IP address, user agent, device metadata
9. Recipient status set to 'acknowledged', audit log entry created
10. Sender sees status update in real-time on the dashboard

### Reminders

1. Vercel cron job runs daily (or configurable interval)
2. Queries recipients where status != 'acknowledged' AND sent_at is past the reminder interval
3. Sends reminder email via Resend
4. Increments reminder_count, updates last_reminder_at

### Proof of acknowledgement

1. Sender clicks "Download proof" on a fully acknowledged document
2. Server generates a PDF containing: document title, recipient name, recipient email, acknowledgement timestamp, IP address, device metadata, unique proof ID
3. PDF is generated on the fly (not stored) and returned as a download

---

## Routes

### Public (no auth)

| Route | Purpose |
|-------|---------|
| `/` | Marketing landing page |
| `/pricing` | Pricing page |
| `/login` | Login |
| `/signup` | Signup |
| `/forgot-password` | Password reset |
| `/d/[token]` | Recipient document view + acknowledge |

### Authenticated (sender dashboard)

| Route | Purpose |
|-------|---------|
| `/documents` | All documents with status filters |
| `/documents/new` | Upload + add recipients + send |
| `/documents/[id]` | Document detail + recipient tracking |
| `/settings` | Account settings |
| `/billing` | Stripe billing portal |

### API

| Route | Method | Purpose |
|-------|--------|---------|
| `/api/documents/upload` | POST | File upload + conversion |
| `/api/documents/send` | POST | Send document to recipients |
| `/api/acknowledge` | POST | Record acknowledgement (public, token-based) |
| `/api/proof/[id]` | GET | Generate proof-of-acknowledgement PDF |
| `/api/cron/reminders` | GET | Cron-triggered reminder check |
| `/api/webhooks/stripe` | POST | Stripe webhook handler |

---

## Brand and design

### Colours

```
/* Primary palette */
--ink: #0a0a0a;         /* Wordmark, headings, UI text */
--zinc: #52525b;        /* Body copy, secondary text */
--stone: #e4e4e7;       /* Borders, dividers, cards */
--paper: #fafaf9;       /* Backgrounds, surfaces */

/* Status colours */
--acknowledged: #22c55e;  /* green-500 */
--pending: #f59e0b;       /* amber-500 */
--overdue: #ef4444;       /* red-500 */
--sent: #3b82f6;          /* blue-500 */

/* Extended neutrals: zinc scale from Tailwind */
```

### Typography

- **Primary:** Plus Jakarta Sans (Google Fonts)
- **Mono:** JetBrains Mono (labels, timestamps, metadata)
- **Wordmark:** Plus Jakarta Sans 800, lowercase, letter-spacing -2px

### Tailwind config

Extend Tailwind with the brand colours above. Use the zinc scale as the default neutral. Default to the `paper` background (#fafaf9) not pure white.

### Design principles

- Monochrome by default. Colour only enters through status states.
- The product should feel like paper and ink, not a SaaS dashboard.
- Generous whitespace. Restrained palette.
- No gradients, no shadows deeper than `shadow-sm`, no rounded corners larger than `rounded-lg`.
- UI components should feel closer to Linear/Notion than to a traditional enterprise app.

---

## Coding conventions

### General

- TypeScript strict mode. No `any` types unless absolutely unavoidable and documented with a comment.
- Use `async/await` everywhere, never raw `.then()` chains.
- Prefer named exports over default exports (except for page/layout components which Next.js requires as default).
- Use `const` by default, `let` only when reassignment is needed, never `var`.
- Destructure props and function parameters.
- Keep files under 200 lines. If a component exceeds this, split it.

### Next.js

- Use the App Router (`app/` directory) exclusively.
- Server Components by default. Add `'use client'` only when the component needs interactivity, hooks, or browser APIs.
- Use `loading.tsx` and `error.tsx` boundary files in route segments.
- Data fetching happens in Server Components or Route Handlers. Never fetch data in Client Components directly — pass it as props or use Server Actions.
- Use Next.js `metadata` export for SEO on every page.

### Components

- One component per file.
- UI primitives go in `components/ui/`. These are generic and reusable (Button, Input, Card, Badge, etc.).
- Feature components go in their domain folder (`components/dashboard/`, `components/documents/`, etc.).
- Props interfaces are defined in the same file, above the component, named `{ComponentName}Props`.
- Use `cn()` utility (clsx + tailwind-merge) for conditional class names.

### Supabase

- Three client variants:
  - `lib/supabase/client.ts` — browser client (uses `createBrowserClient`)
  - `lib/supabase/server.ts` — server client for authenticated requests (uses `createServerClient` with cookies)
  - `lib/supabase/admin.ts` — service role client for server-only operations (recipient lookups, audit log writes)
- Always use the typed Supabase client with generated types from `supabase gen types`.
- Migrations go in `supabase/migrations/` with descriptive names: `001_create_profiles.sql`, `002_create_documents.sql`, etc.

### API routes

- Validate all input at the top of every route handler. Use zod for schema validation.
- Return consistent JSON responses: `{ data, error }`.
- Use appropriate HTTP status codes.
- Log errors server-side, return user-friendly messages client-side.

### Email (Resend)

- All email templates live in `components/emails/` as React components (using @react-email/components).
- Email sending logic lives in `lib/resend/`.
- Every email must have a plain text fallback.

### Error handling

- Never silently swallow errors.
- Use try/catch in async functions, log the error, and return a meaningful response.
- Client-side errors show a toast notification, not an alert.

---

## Security

- Recipient tokens must be cryptographically random (use `crypto.randomUUID()` or equivalent), minimum 32 characters.
- Recipient tokens are the sole access control for the `/d/[token]` route. Treat them like passwords: do not log them, do not expose them in URLs that get indexed.
- The `/d/[token]` route must set `noindex` meta tag and `X-Robots-Tag: noindex` header.
- Rate limit the `/api/acknowledge` endpoint to prevent abuse.
- Validate file types and sizes on upload. Max file size: 20MB. Allowed types: PDF, DOCX, PNG, JPG.
- Strip EXIF data from uploaded images.
- Sanitise all user input. Never render unsanitised HTML.

---

## Environment variables

```
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=

# Resend
RESEND_API_KEY=

# App
NEXT_PUBLIC_APP_URL=https://usesigna.co
```

---

## Testing

- Use Vitest for unit tests.
- Use Playwright for end-to-end tests.
- Critical paths that must have E2E coverage:
  1. Sender uploads a document and adds recipients
  2. Recipient opens the link and acknowledges
  3. Sender sees updated status on the dashboard
  4. Proof of acknowledgement generates correctly

---

## Deployment

- Vercel for hosting. Connected to the GitHub repo.
- Supabase project for database, auth, and storage.
- Stripe for payments (test mode until launch).
- Resend for transactional email.
- Vercel Cron for scheduled reminder checks.

---

## What Signa will NEVER have

These features are explicitly out of scope. If a request or PR introduces any of these, it should be rejected.

- Signature fields or e-signature capture
- Document editing or annotation
- Template builder or clause library
- Contract negotiation workflows
- Long-term document storage or archiving
- HR platform features (employee management, onboarding)
- SSO, SCIM, or enterprise identity management (not in MVP)
- Rich text editor for documents
- In-app messaging between sender and recipient

---

## Voice and copy

All user-facing copy should be:

- Conversational and direct. No buzzwords.
- No em dashes.
- Sentence case everywhere (buttons, headings, labels).
- Error messages should be human: "We couldn't upload that file. Try a PDF, DOCX, or image under 20MB." not "Error: invalid file type."
- Success states should be brief: "Document sent." not "Your document has been successfully sent to all recipients!"

---

## Current status

Project initialised. Next.js App Router scaffolded but empty. This CLAUDE.md is the starting point for all development work.