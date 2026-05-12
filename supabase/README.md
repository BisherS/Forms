# Supabase Setup Notes (Phase 2)

## 1. Create project
1. Create a new Supabase project.
2. Copy project URL and API keys into `.env.local` from `.env.example`.

## 2. Database + Prisma
1. Set `DATABASE_URL` and `DIRECT_URL` using Supabase Postgres connection string.
2. Run:
   - `npm run prisma:generate`
   - `npm run prisma:migrate`

## 3. Auth profile bootstrap
Create a SQL function + trigger so every new `auth.users` row creates `public.profiles`:

```sql
create or replace function public.handle_new_user()
returns trigger
language plpgsql
security definer
set search_path = public
as $$
begin
  insert into public.profiles (id, email)
  values (new.id, new.email)
  on conflict (id) do nothing;
  return new;
end;
$$;

create trigger on_auth_user_created
after insert on auth.users
for each row execute procedure public.handle_new_user();
```

## 4. Admin bootstrap
Promote an existing user:

```sql
update public.profiles
set role = 'admin'
where email = 'admin@company.com';
```

## 5. Storage
1. Create bucket `form-uploads`.
2. Save uploads in path format:
   - `{formId}/{submissionId}/{questionId}/{filename}`

## 6. RLS checklist
Enable RLS and add policies:
- forms/sections/questions/options: production users can read published forms; admins full CRUD.
- submissions/answers/section_progress: users access own rows; admins read all.
- profiles: users manage own profile; admins can read all profiles.
