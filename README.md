# Internal Form System (Microsoft Forms Replacement)

This document defines the **phase-1 system design only**:
1. Database schema (Supabase PostgreSQL + Prisma)
2. Project folder structure (Next.js + App Router)
3. Step-by-step implementation plan

---

## 1) Database Schema Design

## Core design goals
- Support two roles: **admin** and **production_user**
- Support hierarchical form modeling: **form -> sections -> questions**
- Support polymorphic question types:
  - `text`, `number`, `date`, `dropdown`, `single_choice`, `multiple_choice`, `file_upload`, `image_upload`
- Support incremental submissions (saved **section by section**)
- Keep full audit timestamps and allow soft lifecycle states
- Keep answer data flexible but validated by question metadata
- Support export to Excel with one sheet per section

## Prisma schema (recommended)

> Save as: `prisma/schema.prisma`

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum UserRole {
  admin
  production_user
}

enum FormStatus {
  draft
  published
  archived
}

enum QuestionType {
  text
  number
  date
  dropdown
  single_choice
  multiple_choice
  file_upload
  image_upload
}

enum SubmissionStatus {
  in_progress
  completed
  abandoned
}

model Profile {
  id          String   @id @db.Uuid
  email       String   @unique
  fullName    String?
  role        UserRole @default(production_user)
  isActive    Boolean  @default(true)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  createdForms Form[]       @relation("FormCreatedBy")
  submissions  Submission[]

  @@map("profiles")
}

model Form {
  id              String     @id @default(uuid()) @db.Uuid
  title           String
  description     String?
  status          FormStatus  @default(draft)
  version         Int         @default(1)
  allowEditsAfterSubmit Boolean @default(false)
  createdById     String      @db.Uuid
  publishedAt     DateTime?
  archivedAt      DateTime?
  createdAt       DateTime    @default(now())
  updatedAt       DateTime    @updatedAt

  createdBy       Profile      @relation("FormCreatedBy", fields: [createdById], references: [id])
  sections        Section[]
  submissions     Submission[]

  @@index([status])
  @@index([createdById])
  @@map("forms")
}

model Section {
  id           String    @id @default(uuid()) @db.Uuid
  formId        String    @db.Uuid
  title         String
  description   String?
  orderIndex    Int
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  form          Form      @relation(fields: [formId], references: [id], onDelete: Cascade)
  questions     Question[]
  sectionProgress SectionProgress[]

  @@unique([formId, orderIndex])
  @@index([formId])
  @@map("sections")
}

model Question {
  id             String       @id @default(uuid()) @db.Uuid
  sectionId       String       @db.Uuid
  label           String
  helpText        String?
  type            QuestionType
  isRequired      Boolean      @default(false)
  orderIndex      Int
  configJson      Json?        // min/max, placeholder, regex, date ranges, etc.
  createdAt       DateTime     @default(now())
  updatedAt       DateTime     @updatedAt

  section         Section      @relation(fields: [sectionId], references: [id], onDelete: Cascade)
  options         QuestionOption[]
  answers         Answer[]

  @@unique([sectionId, orderIndex])
  @@index([sectionId])
  @@index([type])
  @@map("questions")
}

model QuestionOption {
  id             String    @id @default(uuid()) @db.Uuid
  questionId      String    @db.Uuid
  label           String
  value           String
  orderIndex      Int
  isActive        Boolean   @default(true)

  question        Question  @relation(fields: [questionId], references: [id], onDelete: Cascade)

  @@unique([questionId, value])
  @@unique([questionId, orderIndex])
  @@index([questionId])
  @@map("question_options")
}

model Submission {
  id              String           @id @default(uuid()) @db.Uuid
  formId           String           @db.Uuid
  userId           String           @db.Uuid
  status           SubmissionStatus @default(in_progress)
  startedAt        DateTime         @default(now())
  submittedAt      DateTime?
  createdAt        DateTime         @default(now())
  updatedAt        DateTime         @updatedAt

  form             Form             @relation(fields: [formId], references: [id], onDelete: Cascade)
  user             Profile          @relation(fields: [userId], references: [id])
  answers          Answer[]
  sectionProgress  SectionProgress[]

  @@unique([formId, userId, status]) // optional: only one active in_progress per user/form (enforce in app if needed)
  @@index([formId])
  @@index([userId])
  @@index([status])
  @@map("submissions")
}

model SectionProgress {
  id             String    @id @default(uuid()) @db.Uuid
  submissionId    String    @db.Uuid
  sectionId       String    @db.Uuid
  isCompleted     Boolean   @default(false)
  completedAt     DateTime?
  updatedAt       DateTime  @updatedAt

  submission      Submission @relation(fields: [submissionId], references: [id], onDelete: Cascade)
  section         Section    @relation(fields: [sectionId], references: [id], onDelete: Cascade)

  @@unique([submissionId, sectionId])
  @@index([sectionId])
  @@map("section_progress")
}

model Answer {
  id              String    @id @default(uuid()) @db.Uuid
  submissionId     String    @db.Uuid
  questionId       String    @db.Uuid

  textValue        String?
  numberValue      Decimal?  @db.Decimal(18, 4)
  dateValue        DateTime?
  selectedOptions  Json?     // array of option values (for dropdown/single/multiple)
  filePath         String?   // Supabase Storage path
  fileUrl          String?   // optional signed/public URL snapshot

  createdAt        DateTime  @default(now())
  updatedAt        DateTime  @updatedAt

  submission       Submission @relation(fields: [submissionId], references: [id], onDelete: Cascade)
  question         Question   @relation(fields: [questionId], references: [id], onDelete: Cascade)

  @@unique([submissionId, questionId])
  @@index([questionId])
  @@map("answers")
}
```

## Supabase/Auth notes
- `profiles.id` should match `auth.users.id` (UUID).
- Create DB trigger/function to auto-create a `profiles` row on user signup.
- Restrict admin actions with role checks (`profiles.role = 'admin'`).

## Storage bucket design
- Bucket: `form-uploads`
- Path pattern:
  - `form-uploads/{formId}/{submissionId}/{questionId}/{filename}`
- Store path in `answers.filePath`.

## RLS (Row-Level Security) policy plan
- `profiles`: users read/update own profile; admins can read all.
- `forms/sections/questions/options`: production users can read only `published` forms; admins CRUD all.
- `submissions/answers/section_progress`:
  - production user can CRUD only own submission data.
  - admin can read all submissions for exports/review.

---

## 2) Project Structure (Next.js + Supabase + Prisma + shadcn/ui)

```text
forms-app/
  prisma/
    schema.prisma
    migrations/

  src/
    app/
      (auth)/
        login/page.tsx
        callback/route.ts

      (admin)/
        admin/
          forms/page.tsx                  # list forms
          forms/new/page.tsx              # create form
          forms/[formId]/edit/page.tsx    # form builder (sections/questions)
          forms/[formId]/submissions/page.tsx
          forms/[formId]/export/route.ts  # Excel download endpoint

      (production)/
        forms/page.tsx                    # published forms list
        forms/[formId]/start/page.tsx
        forms/[formId]/sections/[sectionId]/page.tsx
        forms/[formId]/review/page.tsx
        forms/[formId]/submitted/page.tsx

      api/
        forms/route.ts
        forms/[formId]/route.ts
        submissions/route.ts
        submissions/[submissionId]/answers/route.ts
        submissions/[submissionId]/sections/[sectionId]/complete/route.ts

      layout.tsx
      page.tsx

    components/
      ui/                                 # shadcn/ui generated components
      forms/
        form-builder/
          form-editor.tsx
          section-editor.tsx
          question-editor.tsx
          question-type-config.tsx
        form-runtime/
          question-renderer.tsx
          section-progress.tsx
          answer-inputs/
            text-input.tsx
            number-input.tsx
            date-input.tsx
            select-input.tsx
            radio-input.tsx
            checkbox-input.tsx
            file-upload-input.tsx
            image-upload-input.tsx
        admin/
          submissions-table.tsx
          submission-detail-drawer.tsx

    lib/
      auth/
        roles.ts
        guards.ts
      db/
        prisma.ts
      supabase/
        client.ts
        server.ts
        middleware.ts
      forms/
        validators.ts                     # zod schemas for payloads
        question-type.ts
      excel/
        export-sections.ts                # ExcelJS sheet mapping logic
      storage/
        upload.ts
      utils/
        date.ts

    server/
      repositories/
        form-repository.ts
        submission-repository.ts
      services/
        form-service.ts
        submission-service.ts
        export-service.ts

    types/
      form.ts
      submission.ts

    middleware.ts

  supabase/
    migrations/
    seed.sql

  scripts/
    seed.ts

  package.json
  tsconfig.json
  next.config.ts
  .env.example
```

---

## 3) Step-by-step Implementation Plan

## Phase 1 - Foundation
1. Initialize Next.js app (App Router, TypeScript, ESLint).
2. Install core dependencies:
   - `@supabase/supabase-js`, `@supabase/ssr`
   - `@prisma/client`, `prisma`
   - `exceljs`
   - `zod`, `react-hook-form`
   - shadcn/ui stack
3. Configure environment variables (`DATABASE_URL`, Supabase URL/keys, storage bucket).

## Phase 2 - Database + Auth backbone
4. Implement `schema.prisma` from this design.
5. Create and run Prisma migrations against Supabase Postgres.
6. Add signup trigger to populate `profiles` table from `auth.users`.
7. Create initial admin user and set `profiles.role = admin`.
8. Enable and apply RLS policies for each table.

## Phase 3 - Admin form builder (MVP)
9. Build admin guard middleware/layout.
10. Implement form CRUD (draft/published states).
11. Implement section CRUD with ordering.
12. Implement question CRUD with ordering and type config.
13. Implement option CRUD for dropdown/single/multiple question types.

## Phase 4 - Production form filling flow
14. Build published forms list page for production users.
15. Implement “start submission” flow (`Submission` row creation).
16. Render section-by-section questionnaire from schema.
17. Save answers per section and upsert `Answer` records.
18. Mark section completion in `SectionProgress`.
19. Finalize submission (`status = completed`, `submittedAt = now`).

## Phase 5 - Admin submission review
20. Build admin submissions list per form.
21. Build submission detail view grouped by section/question.
22. Add filters (date range, user, completion status).

## Phase 6 - Excel export
23. Implement export service using ExcelJS:
   - one worksheet per section
   - columns: submitter, submission timestamp, and section question columns
24. Add export API route and admin UI button.
25. Validate file/image answers are exported as storage paths or signed URLs.

## Phase 7 - Hardening
26. Add server-side payload validation with Zod.
27. Add optimistic locking/version checks for form edits.
28. Add audit logs (optional table: `activity_logs`).
29. Add tests for repositories/services and key API routes.
30. Add seed data for realistic test form templates.

---

## Notes on answer modeling choices
- `Answer` table uses type-specific columns for efficient querying + light validation.
- `selectedOptions` stored as JSON array to support both single and multiple choice consistently.
- `Question.configJson` allows future extensibility (e.g., numeric min/max, file size limits, regex).

If you want, next step can be: **build the Prisma schema and first migration files only**, without UI.
