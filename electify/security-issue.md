# WorkspacePermissonRequiredOnUrlTarget
Security issue I found (existing, not changed): 
WorkspacePermissonRequiredOnUrlTarget checks if (!allowed) against a profiler result object, which is always truthy. Cross-workspace access is still blocked, but role-based action limits are never enforced on any route that uses this decorator. The fix is to unwrap .result, but that changes authorization across the whole app, so it's your call. My new endpoints check the result correctly.

Lint: npm run lint can't run because the repo has no eslint.config.js for ESLint 10.

The main new code is in src/apps/worker-api/jobs/, src/core/repositories/job.repo/, src/core/repositories/job-queue.repo/ and src/core/services/job-cancellation.service.ts. Next in the plan is message sending (§3a), then MessageBatch (§3).





# IpAddressRepository.upsert
Possible security bug (not about speed)
I think IpAddressRepository.upsert (ip-address.repo/index.ts:111-126) un-bans IPs. It passes status: ALLOWED to TypeORM's upsert. As I understand TypeORM, that turns into ON CONFLICT DO UPDATE SET status = EXCLUDED.status, ..., and skipUpdateIfNoValuesChanged only skips the update when nothing differs, which a banned IP always does. So the next request from a banned IP would reset it to ALLOWED before the BANNED check runs. I haven't run it to confirm; a quick test (ban an IP, then send one request) would settle it. The single-query rewrite in item 5 fixes this as well.

None of these changes the schema, so you won't need makemigrations. Want me to make changes 1–5 and the IP fix?
![[Pasted image 20261123163846.png]]


- [ ] ⏫ Testing Task
