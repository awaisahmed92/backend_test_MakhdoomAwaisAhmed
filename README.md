## Recruitment SaaS Database (Supabase)

This repo contains Database-as-Code for a multi-tenant recruitment SaaS on Supabase.

### Prerequisites
- Supabase project (Postgres 15+)
- psql or Supabase SQL editor
- Optional: bash, `psql` CLI, and DATABASE_URL for scripted apply

### Schema Overview
- Tenancy: `organizations`, `organization_members` (+ roles enum `user_role`)
- Core: `jobs`, `candidates`, `candidate_documents`
- AI: `processing_runs`, `ai_summaries`, `matches`
- Collaboration: `notes`, `invitations`
- Integrations: `api_keys`, `webhooks`
- Profiles: `profiles` mirrors auth users metadata

RLS isolates data by organization. Helper functions (`is_org_member`, `has_org_role`) enforce access.

### Apply via SQL editor (manual)
Run these files in order in the Supabase SQL editor:
1. `migrations/0001_init.sql`
2. `migrations/0002_security_and_rls.sql`
3. `migrations/0003_indexes.sql`
4. `migrations/0004_storage.sql`
5. `migrations/0005_soft_delete.sql`
6. `migrations/0006_auditing.sql`
7. `migrations/0007_views_and_functions.sql`
8. `migrations/0008_rpc.sql`
9. `seed/000_seed.sql`
10. `seed/001_enhanced_seed.sql`

### Apply via CLI (psql)
Set your `DATABASE_URL` to the Supabase connection string (service role for full apply):

```bash
export DATABASE_URL="postgres://postgres:<password>@<host>:5432/postgres?sslmode=require"
```

Then run:

```bash
bash scripts/apply.sh
```

Or directly with psql include script:

```bash
psql "$DATABASE_URL" -v ON_ERROR_STOP=1 -f apply_all.sql
```

### Storage
Create a bucket named `candidate-cvs` in Supabase Storage. The policies in `migrations/0004_storage.sql` assume this bucket name.

### Notes
- `seed/000_seed.sql` uses `auth.uid()` for created_by and membership; run as an authenticated user in SQL editor or adjust IDs as needed.
- Never store plaintext API keys; only hashed values are stored in `api_keys`.
- For production, rotate keys, secrets, and review RLS policies.

### Demo queries
- Ranked results for a job:
```sql
select * from public.get_job_results('00000000-0000-0000-0000-000000000001','10000000-0000-0000-0000-000000000003');
```
- View audit trail:
```sql
select occurred_at, table_name, operation, row_id from audit.events order by occurred_at desc limit 20;
```
- List visible matches via view:
```sql
select * from public.v_ranked_matches order by score desc limit 10;
```

### Docs
- ERD (Mermaid): `docs/ERD.mmd`
- Render ERD PNG: `bash scripts/render-erd.sh` → outputs `docs/ERD.png`
- Slide outline: `docs/slides-outline.md`
- Speaker notes (verbatim, ≤5 min): `docs/speaker-notes.md`

### REST and RPC usage (Supabase)
- Get ranked results via RPC:
```bash
curl -s --request POST \
  --url "https://<YOUR-PROJECT-REF>.supabase.co/rest/v1/rpc/api_get_job_results" \
  --header "apikey: <ANON_OR_SERVICE_KEY>" \
  --header "Authorization: Bearer <ANON_OR_SERVICE_KEY_OR_USER_JWT>" \
  --header "Content-Type: application/json" \
  --data '{"p_job_id":"10000000-0000-0000-0000-000000000003"}'
```
- Queue a processing run:
```bash
curl -s --request POST \
  --url "https://<YOUR-PROJECT-REF>.supabase.co/rest/v1/rpc/api_queue_processing_run" \
  --header "apikey: <SERVICE_KEY>" \
  --header "Authorization: Bearer <SERVICE_KEY_OR_USER_JWT>" \
  --header "Content-Type: application/json" \
  --data '{"p_org_id":"00000000-0000-0000-0000-000000000001","p_job_id":"10000000-0000-0000-0000-000000000003"}'
```


