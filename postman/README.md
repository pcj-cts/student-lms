# Postman collection — Student LMS Backend

A complete Postman collection covering all ~50 endpoints across the 8 services, with chained variables so requests flow into each other (login auto-stores tokens, create-course stores the new `courseId`, etc.).

## Files

| File | Purpose |
|---|---|
| `student-lms-be.postman_collection.json` | The collection itself (one folder per service + a Setup folder) |
| `student-lms-be.postman_environment.json` | Local environment with `host=localhost` + persona credentials |

## Import

In Postman:

1. **File → Import** → drop both JSON files
2. Top-right environment dropdown → select **LMS — Local**
3. Make sure all 8 services are running on `localhost:8081–8088`

## How to use

### Run order — first time

Open the **`00 — Setup (run first)`** folder and execute requests 0.1 through 0.6 **in order**:

| # | Request | What it stores |
|---|---|---|
| 0.1 | Register Admin | `adminId` |
| 0.2 | Register Instructor | `instructorId` |
| 0.3 | Register Student | `studentId` |
| 0.4 | Login Admin | `adminToken`, `adminRefreshToken` |
| 0.5 | Login Instructor | `instructorToken`, `instructorRefreshToken` |
| 0.6 | Login Student | `studentToken`, `studentRefreshToken` |

After these six requests, every other request in the collection has a valid bearer token populated automatically.

### After tokens expire (15 min)

Either:
- Re-run **0.4 / 0.5 / 0.6** (login again), or
- Use **`01 — Auth Service → Refresh student token`** to rotate (it'll update both `studentToken` and `studentRefreshToken`).

### Happy-path walkthrough

After the Setup folder, this sequence exercises a meaningful slice of the system:

1. `03 → [Instructor] Create course` — stores `courseId`
2. `03 → [Instructor] Create module` — stores `moduleId`
3. `03 → [Instructor] Create lesson` — stores `lessonId`
4. `03 → [Instructor] Submit course for approval`
5. `03 → [Admin] Approve course` → course is now PUBLISHED
6. `04 → [Student] Enrol in course` — stores `enrollmentId`
7. `06 → [Student] Mark lesson complete`
8. `06 → Get progress detail` → check completion %
9. `05 → [Instructor] Create quiz` — stores `assessmentId`
10. `05 → [Student] Submit quiz answers` — stores `submissionId`, score auto-populated
11. `05 → [Instructor] Grade submission` (only if quiz had short-answer questions)
12. `05 → [Instructor] Release grade`
13. `07 → [Admin] Dispatch notification` → check service logs for delivery
14. `08 → [Admin] Write log entry` + `[Admin] List logs`

### Things to actively look for

| Behaviour | How to surface it |
|---|---|
| **JWT cross-service** | The token from `0.6 Login Student` (issued by 8081) works on 8082-8088 |
| **404, not 403, hides existence** | Anonymous `GET /api/v1/courses/{id}` for a DRAFT course returns 404 |
| **Idempotency** | Run `06 → Mark lesson complete` twice — second call is a no-op |
| **Two-step grade** | Student sees `score: null` until `PATCH /release` fires |
| **RBAC** | Student calling `02 → [Admin] List users` returns 403 |
| **Cert event** | When a student crosses 100% completion, progress-service logs `DomainEvent published type=certificate-required` |

## Variables stored automatically

Every create/login response is intercepted by a `pm.collectionVariables.set(...)` test script. You should never need to copy-paste IDs manually.

| Variable | Set by |
|---|---|
| `adminId`, `instructorId`, `studentId` | Register requests (0.1–0.3) or Login requests (0.4–0.6) |
| `adminToken`, `instructorToken`, `studentToken` | Login requests |
| `adminRefreshToken`, `instructorRefreshToken`, `studentRefreshToken` | Login requests |
| `courseId` | `03 → Create course` |
| `moduleId` | `03 → Create module` |
| `lessonId` | `03 → Create lesson` |
| `announcementId` | `03 → Post announcement` |
| `enrollmentId` | `04 → Enrol in course` |
| `assessmentId` | `05 → Create quiz/assignment` |
| `submissionId` | `05 → Submit quiz/assignment` |
| `noteId`, `bookmarkId` | `06 → Create note/bookmark` |
| `logId` | `08 → Write log entry` |

## Gotchas

- **Service must be running.** If a request hangs or returns `ECONNREFUSED`, the corresponding service (port in the URL) isn't up. Check with:
  ```powershell
  curl.exe -s -o NUL -w "%{http_code}`n" http://localhost:8081/actuator/health
  ```
- **JWT_SECRET must match across services.** All services in this repo default to the same placeholder value in `application.yml`, so tokens issued by auth-service are accepted by all other services. If you've overridden `JWT_SECRET` env on one service but not the others, you'll see 401s on cross-service calls.
- **Tokens expire after 15 minutes.** Just re-run the login requests in folder 00.
- **Folder 00 register requests fail with 409** if you've already run them once (email is unique). That's expected — just skip to the login requests (0.4–0.6) which always work.
- **Postman runs requests serially when you "Run Collection".** The Setup folder must finish before anything else, because all other requests depend on the variables it sets.

## Running the whole collection as a smoke test

**Runner → New Run → select the collection → uncheck folders you don't want → Run.** Postman's collection runner executes top-to-bottom and shows pass/fail per assertion (we have minimal assertions for now — just the variable-extraction scripts). Useful as a "did everything come up healthy" sanity check.

For a stricter smoke test, add `pm.test("status is 200", ...)` blocks to each request's `tests` array — easy to extend.
