# P1 — Identity and audit (time-box: 1.5 weeks)

> Place at `docs/prompts/P1_IDENTITY_AUDIT.md`. Read `CLAUDE.md` and `docs/prompts/00_MASTER_PROMPT.md` first.
> **This prompt is self-contained.** `docs/design/*.md` are optional references; where they differ from this prompt, this prompt wins.
> Branches: `phase-1/prep` (exists), `phase-1/identity`, `phase-1/audit`, `phase-1/facades`. One commit per step.
> **Exit gate:** every role can log in and reach its own facade; every write and every login, logout, failure and denial is audited; CI green on the PR; PR **not merged** until the STOP D review approves it.

Scope: FR-AUTH-01..05, FR-AUDIT-01..02, FR-API-01..04 (identity part), MU-02, MU-05, MU-12.
ACs proven in P1: **AC-AUTH-01, AC-AUTH-02, AC-AUTH-03, AC-AUTH-04, AC-SEC-01, AC-API-01**.
AC-SEC-02 (another student's marks) needs marks, so it is proven in P5; P1 builds and unit-tests the `OwnershipGuard` it will use.

---

## Step 0 — Inputs check (no commit)

Before any code, report in ≤ 10 lines and stop only if something is missing:
- `main` is at `8e54e71` or later; `phase-1/prep` has `2e58842`.
- Test cluster or local PostgreSQL 18 reachable with `UMS_TEST_DB_*` and all role passwords set.
- `gh` CLI available or not (affects how the PR is opened).
- `docs/design/*.md` present or not (informational only).

## Step 1 — CI maintenance · on `phase-1/prep`

- `ci: upgrade actions to node 24 majors` — move `actions/checkout`, `actions/setup-java`, `actions/upload-artifact` to their current Node-24 major versions (verify the version tags exist on GitHub Marketplace; pin by major tag).
- `ci: pin runner to ubuntu-24.04` — replace every `ubuntu-latest`.
- Push `phase-1/prep`, open a PR to `main`, wait for green. **Do not merge.** Report the run URL in STOP A.

## Step 2 — Spec sync · `docs: p1 spec additions` (on `phase-1/identity`, branched from `main` after I merge `phase-1/prep`; if I have not merged yet, branch from `phase-1/prep`)

Apply these approved additions to `docs/prompts/00_MASTER_PROMPT.md` and `CLAUDE.md`:
- `CLAUDE.md` §0: "`docs/design/*.md` are optional references. Phase prompts are self-contained and take precedence."
- Master §10: add error code **`PASSWORD_CHANGE_REQUIRED`** (area `auth`) with message key and English text.
- Master §5: add `AccountApi` (any role: `profile()`, `updateContact(ContactUpdate)`, `changePassword(char[] current, char[] next)`), obtained via `Session.account()`; `Session.changePassword` delegates to it. Add `@FacadeRole(Role)` annotation (type-level) in `ums-api/annotation`. `UserSummary` gains `UserId id`.
- Master §6: P1 tables below.

---

## Step 3 — Identity schema · `feat(db): identity tables and console column grants`

`V3__p1_identity.sql` (never edit V1/V2). All constraints and indexes named.

```sql
CREATE TABLE app_user (
  id                    BIGINT GENERATED ALWAYS AS IDENTITY,
  username              VARCHAR(64)  NOT NULL,
  password_hash         VARCHAR(72)  NOT NULL,
  role                  VARCHAR(16)  NOT NULL,
  full_name             VARCHAR(128) NOT NULL,
  email                 VARCHAR(254),
  phone                 VARCHAR(32),
  active                BOOLEAN      NOT NULL DEFAULT TRUE,
  failed_attempts       INT          NOT NULL DEFAULT 0,
  locked_until          TIMESTAMPTZ,
  must_change_password  BOOLEAN      NOT NULL DEFAULT TRUE,
  password_changed_at   TIMESTAMPTZ,
  version               BIGINT       NOT NULL DEFAULT 0,
  created_at            TIMESTAMPTZ  NOT NULL DEFAULT now(),
  updated_at            TIMESTAMPTZ  NOT NULL DEFAULT now(),
  CONSTRAINT pk_app_user PRIMARY KEY (id),
  CONSTRAINT uq_app_user_username UNIQUE (username),
  CONSTRAINT ck_app_user_username_lower CHECK (username = lower(username)),
  CONSTRAINT ck_app_user_role CHECK (role IN ('STUDENT','FACULTY','ADMIN','DEV')),
  CONSTRAINT ck_app_user_failed_attempts CHECK (failed_attempts >= 0)
);
CREATE INDEX idx_app_user_role_active ON app_user (role, active);

CREATE TABLE user_session (
  id            UUID        NOT NULL,
  user_id       BIGINT      NOT NULL,
  role          VARCHAR(16) NOT NULL,
  state         VARCHAR(8)  NOT NULL DEFAULT 'ACTIVE',
  login_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_seen_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_action   VARCHAR(64),
  ended_at      TIMESTAMPTZ,
  end_reason    VARCHAR(16),
  CONSTRAINT pk_user_session PRIMARY KEY (id),
  CONSTRAINT fk_user_session_user FOREIGN KEY (user_id) REFERENCES app_user (id),
  CONSTRAINT ck_user_session_state CHECK (state IN ('ACTIVE','EXPIRED','CLOSED')),
  CONSTRAINT ck_user_session_end_reason CHECK (end_reason IS NULL OR end_reason IN
    ('LOGOUT','IDLE','RESTART','DEACTIVATED','ROLE_CHANGED','PASSWORD_RESET'))
);
CREATE INDEX idx_user_session_active ON user_session (user_id) WHERE state = 'ACTIVE';
CREATE INDEX idx_user_session_login ON user_session (login_at DESC, id);

ALTER TABLE activity_log   ADD CONSTRAINT fk_activity_log_user    FOREIGN KEY (user_id)  REFERENCES app_user (id);
ALTER TABLE change_history ADD CONSTRAINT fk_change_history_actor FOREIGN KEY (actor_id) REFERENCES app_user (id);

-- console role must never read password hashes (MU-09)
REVOKE SELECT ON app_user FROM ums_dev_console;
GRANT SELECT (id, username, role, full_name, email, phone, active, failed_attempts, locked_until,
              must_change_password, password_changed_at, version, created_at, updated_at)
  ON app_user TO ums_dev_console;
```

- Add `fk_*` → `NOT_FOUND` already covered by 23503; register `ck_app_user_role`, `ck_app_user_username_lower` → `VALIDATION_FAILED` in `ConstraintErrorMap`.
- ITs: catalog check of names/prefixes; `ums_dev_console` gets `42501` on `SELECT password_hash` but can read the other columns; `ums_app` can DML both tables; V3 idempotency (migrate twice).

## Step 4 — Domain and repositories · `feat(auth): user and session domain with repositories`

Package `edu.ums.core.auth`:
- `domain/`: `User` (record), `Username` (value object: trim, lower-case, 3–64 chars `[a-z0-9._-]`, char-loop validation, no regex), `AccountState` enum `ACTIVE LOCKED DEACTIVATED` (**derived**: `!active → DEACTIVATED`, `locked_until > now → LOCKED`, else `ACTIVE`), `SessionState` enum `ACTIVE EXPIRED CLOSED`, `EndReason` enum.
- `domain/PasswordPolicy`: min 10 characters, **max 72 UTF-8 bytes** (BCrypt truncates beyond 72 — reject, don't truncate), not equal to username, new ≠ current → else `PASSWORD_POLICY` with detail `rule`.
- `AccountLifecycle` and `SessionLifecycle` as `StateMachine` constants:
  - Account: `ACTIVE→DEACTIVATED` (guard: not the last active admin), `DEACTIVATED→ACTIVE`, `LOCKED→ACTIVE` (admin unlock), `LOCKED→DEACTIVATED` (same guard).
  - Session: `ACTIVE→EXPIRED`, `ACTIVE→CLOSED`.
- `repository/UserRepository` (+ `JdbcUserRepository`), `repository/SessionRepository` (+ Jdbc). Every SQL statement `PreparedStatement`; list queries keyset by `(username)` (unique) or `(login_at DESC, id)`.
- `edu.ums.platform.security`: `PasswordHasher` (jBCrypt cost 12; `char[]` in, zeroed after; constant-time verify; a fixed dummy hash for unknown users so timing is uniform), `SecureTokens` (session ids via `SecureRandom`-backed `UUID`).
- Tests: jqwik for `Username` and `PasswordPolicy` (including multi-byte input at the 72-byte edge); every (from, to) pair for both machines + 1,000-try property tests.

## Step 5 — Login, lockout and bootstrap · `feat(auth): login with lockout and first-admin bootstrap`

`AuthService.login(String username, char[] password)` — exact order:
1. Normalise username (invalid format → `BAD_CREDENTIALS`, never `VALIDATION_FAILED`, so format doesn't leak).
2. **One write transaction that always commits**, returning a sealed `LoginOutcome` (`Success(user)` | `Failure(ErrorCode)`); the exception is thrown only **after** commit — otherwise the rollback would undo the failed-attempt counter.
   - Unknown user → verify against the dummy hash (equal timing) → `Failure(BAD_CREDENTIALS)`.
   - Inactive → `Failure(BAD_CREDENTIALS)` (uniform).
   - `locked_until > now()` → `Failure(ACCOUNT_LOCKED)` without checking the password.
   - Wrong password → one conditional update, no read-then-write in Java:
     ```sql
     UPDATE app_user
        SET failed_attempts = CASE WHEN locked_until IS NOT NULL AND locked_until <= now() THEN 1
                                   ELSE failed_attempts + 1 END,
            locked_until    = CASE WHEN (CASE WHEN locked_until IS NOT NULL AND locked_until <= now() THEN 1
                                              ELSE failed_attempts + 1 END) >= ?  -- lockout threshold (5)
                                   THEN now() + ?::interval ELSE NULL END,     -- lockout duration (15 min)
            updated_at = now()
      WHERE id = ?
      RETURNING failed_attempts, locked_until
     ```
     → the attempt that reaches the threshold returns `Failure(ACCOUNT_LOCKED)`, earlier ones `Failure(BAD_CREDENTIALS)`.
   - Correct password → reset `failed_attempts = 0, locked_until = NULL`; insert `user_session`; `Success`.
3. Audit after commit: `LOGIN` (SUCCESS), `LOGIN_FAILED` (FAILURE, role `ANONYMOUS` when the user is unknown, details `{reason}` only — **never** the attempted password; attempted username truncated to 64 chars), `ACCOUNT_LOCKED` when the lock is set.

First-admin bootstrap (needed to log in at all):
- `Main --create-admin <username>` reads the password from env `UMS_BOOTSTRAP_ADMIN_PASSWORD` (never from argv), applies `PasswordPolicy`, and succeeds **only if no active ADMIN exists**; creates the user with `must_change_password = TRUE`; audits `USER_CREATED` with role `SYSTEM`. Otherwise exits non-zero with a clear message. Document in README.

Tests: AC-AUTH-01, AC-AUTH-02 (and that lock expiry after 15 min allows login — use an injectable `Clock`/DB time offset, no sleeping), MU-02 (unknown vs wrong password: same code, and median timing within 25 %), **concurrency**: 20 simultaneous wrong-password logins for one user → `failed_attempts` exactly matches the attempts up to the threshold, lock set exactly once, no lost update; failure injection: connection killed mid-login → `DB_UNAVAILABLE`, counter unchanged.

### STOP A — "Identity core"
Include the V3 file, the login SQL, timing evidence for MU-02, the concurrency test result, Step 1's CI run URL. Await review.

---

## Step 6 — Sessions · `feat(auth): session registry, idle expiry and sweeper` (on `phase-1/audit`)

- `SessionRegistry`: `ConcurrentHashMap<SessionId, SessionState>`, **capped** (config `ums.session.max-active`, default 5,000; beyond → login fails `BUSY_TRY_AGAIN`). `SessionState` holds user id, role, `AtomicLong lastSeenMillis`, `AtomicLong lastPersistedMillis`, volatile state.
- Touch on every call: in-memory always; DB `last_seen_at` / `last_action` written **at most once per 60 s per session** (CAS on `lastPersistedMillis`), in its own short transaction.
- Idle > 30 min → `EXPIRED`: removed from the registry, DB row CAS `ACTIVE→EXPIRED` (`end_reason IDLE`), audit `SESSION_EXPIRED`, caller gets `SESSION_EXPIRED`.
- Scheduler job every 5 min sweeps idle sessions (bounded batch of 500).
- On startup: every DB row still `ACTIVE` is set `CLOSED`, `end_reason RESTART` (the registry is in memory).
- Admin deactivate, role change and password reset close all of that user's sessions (`DEACTIVATED`, `ROLE_CHANGED`, `PASSWORD_RESET`).
- Uses an injectable `Clock` everywhere; tests never sleep for timeouts.

## Step 7 — Interceptors · `feat(platform): session, authorisation and rate-limit interceptors`

- **SESSION** (slot 2): first checks `Readiness` (`DB_UNAVAILABLE` until ready); resolves the session bound to this facade proxy; expiry check; touch; puts a `CallerContext` (user id, role, session id) in a ThreadLocal for the call, cleared in `finally`; sets MDC `userId`, `role`.
- **AUTHORISATION** (slot 3): if the facade interface has `@FacadeRole`, it must equal the caller's role (an interface without it, i.e. `AccountApi`, allows any authenticated role), else `ACCESS_DENIED` **and** an audit event `ACCESS_DENIED` (FAILURE, details `{facade, method}`) — this interceptor records it itself, because denied calls never reach the AUDIT slot. If `must_change_password` is set, every call except `AccountApi.changePassword` and `Session.close` fails with `PASSWORD_CHANGE_REQUIRED`.
- **RATE_LIMIT** (slot 5): `@RateLimited(perMinute)`; per-session `TokenBucket` (capacity = perMinute, refill per minute, `synchronized` per bucket, removed when the session ends) → `RATE_LIMITED`. No production method uses it until P3; prove it with a test facade.
- Facade proxies are created **per session** (at most 5 cached in the session: account, student, faculty, admin, dev), so the SESSION interceptor knows its session without a lookup by argument.
- `OwnershipGuard` (in `edu.ums.core.common`): `requireSelf(UserId)`, plus an extension point for later phases (`requireOwnEnrolment`, `requireTeachesSection` added in P3/P5). Unit-tested now.

## Step 8 — Audit pipeline · `feat(audit): async activity writer, change history and error reporter`

- `AuditService.record(AuditEvent)` — O(1) `offer` into `ArrayBlockingQueue` (capacity 10,000); full → `droppedCount++` and a rate-limited WARN log (at most once per minute). Never blocks, never throws.
- `AuditEvent` captures `occurred_at` (from `Clock`), user id, username, role, session id, action, entity type/id, outcome, details (small `Map`, serialised with Jackson, ≤ 4 KB, else truncated with a flag), trace id.
- `AuditWriter`: one `audit-writer` thread; `poll(500 ms)` then `drainTo` up to 100; one JDBC batch insert as `ums_app` in its own transaction; on DB failure → batch counted as dropped, ERROR logged once per minute, keep running; registered in the **DRAINERS** shutdown stage (drain ≤ 5 s, then count the rest as dropped).
- **AUDIT** interceptor (slot 7): for methods with `@Audited(action)`, after the chain returns (i.e. after commit) record SUCCESS; on exception record FAILURE with `{code}`.
- `ChangeHistoryWriter.record(actor, entityType, entityId, before, after, reason)` — **synchronous, inside the current transaction** (requires an active tx; else `INTERNAL_ERROR`). Used now for user role/active/unlock/reset changes.
- `ErrorReporter` replaces the no-op `UncaughtErrorSink`: bounded queue 1,000 drained by the **same** `audit-writer` thread into `app_error_log` (level, source, message without row data, stack trace, trace id). `ApiErrorTranslator` reports every `INTERNAL_ERROR` with its incident id.
- Tests: 100 threads × 100 events → all persisted when under capacity; queue full → callers return within 1 ms p99 and drops are counted exactly; DB down during writing (FailureInjector) → app keeps serving, drops counted, writer recovers when DB returns; shutdown drains within 5 s; MU-12: capture all log output during login/reset tests and assert no password, hash or temporary password appears.

### STOP B — "Sessions, interceptors and audit"
Include complexity table (registry, bucket, writer, interceptors), concurrency notes (new threads: `audit-writer`; shutdown order now ACCEPT_STOP → DRAINERS(audit-writer) → SCHEDULER → LISTENERS → POOLS), drop/latency evidence. Await review.

---

## Step 9 — Facades · `feat(api): client, session, account and admin user management` (on `phase-1/facades`)

`ums-api` (additive only):
- `id/UserId(long value)`; `UserSummary(UserId id, String username, String displayName, Role role)`.
- `dto/UserDto(UserId id, String username, String fullName, Role role, String email, String phone, boolean active, AccountStateDto state, boolean mustChangePassword, long version)`; `AccountStateDto` enum mirrors `AccountState` with a plain-text `label()`.
- `dto/NewUser(String username, String fullName, Role role, String email, String phone)`; `dto/UserUpdate(UserId id, String fullName, String email, String phone, Role role, long version)`; `dto/ContactUpdate(String email, String phone, long version)`; `dto/UserFilter(Optional<Role> role, Optional<Boolean> active, Optional<String> text)`.
- `dto/IssuedPassword(UserDto user, char[] temporaryPassword)` — returned **once**; never stored or logged; the caller clears it.
- `AccountApi` (`@FacadeRole` not applied — any role): `profile()`, `updateContact(ContactUpdate)`, `changePassword(char[], char[])`.
- `AdminApi` (`@FacadeRole(ADMIN)`), all `@Writes @Audited` except search: `createUser(NewUser)` → `IssuedPassword` (16-char random temp password, `must_change_password = TRUE`); `updateUser(UserUpdate)`; `deactivate(UserId, long version, String reason)`; `reactivate(UserId, long version)`; `unlock(UserId, long version)`; `resetPassword(UserId, long version)` → `IssuedPassword`; `searchUsers(UserFilter, PageRequest)` → `Page<UserDto>` (keyset on `username`; `text` matches username/full name prefix, case-insensitive, via `lower(...) LIKE ? || '%'` with an index `idx_app_user_full_name_lower` added in **V4** if needed).
- `StudentApi`, `FacultyApi`, `DevApi` get `@FacadeRole` and stay otherwise empty.
- `UmsClient.login` → `Session`; `Session.account()`, `student()`, `faculty()`, `admin()`, `dev()` (wrong role → `ACCESS_DENIED`, audited), `onNotice` returns a working `Subscription` that receives nothing until P4, `close()` → `CLOSED`, `LOGOUT`, idempotent.

`ums-core` implementations: `AccountApiImpl`, `AdminApiImpl`, `UserAdminService`:
- **Last-admin rule (MU-05, AC-AUTH-04):** any change that removes an active admin (deactivate, role change away from ADMIN) first runs `SELECT id FROM app_user WHERE role = 'ADMIN' AND active ORDER BY id FOR UPDATE`, counts in Java (bounded by number of admins), and fails `LAST_ADMIN` if it would reach zero. Ordered locking prevents deadlock.
- Every change writes `change_history` (before/after of role, active, lock, must_change_password; **never** hashes) and closes sessions where Step 6 says so.
- Optimistic `version` on every write → `STALE_DATA`.
- Wire everything in `AppContext`; `UmsRuntime` exposes `UmsClient`.

## Step 10 — Activity listener · `feat(audit): activity listener and event bus`

- `ActivityListener`: dedicated `ums_listener` connection; `LISTEN ums_activity`; loop `getNotifications(5000)`; on any notification fetch `WHERE id > ? ORDER BY id LIMIT 200` until empty; starts from `max(id)` at startup; reconnects with backoff 1, 2, 4 … 30 s and catches up from `lastSeenId` (no gaps, no duplicates); registered in the **LISTENERS** shutdown stage.
- `ActivityEventBus`: subscribers in `CopyOnWriteArrayList`; dispatch on `listener-dispatch` (2 threads); each subscriber has a bounded queue of 1,000, drop-oldest, with a per-subscriber dropped counter. A throwing subscriber is logged and never affects others.
- No facade exposes it yet (DevApi feed is P6); prove it with ITs: insert → subscriber within 2 s; kill the listener backend (`FailureInjector`) → reconnect + catch-up delivers the missed rows exactly once; slow subscriber doesn't delay a fast one.

## Step 11 — Acceptance tests · `test: p1 acceptance and misuse tests` (in `ums-app/src/test`)

- `AC_AUTH_01_loginReachesRoleFacade` — for all four roles.
- `AC_AUTH_02_fifthFailureLocksFor15Minutes`
- `AC_AUTH_03_idleSessionExpires` (injectable clock)
- `AC_AUTH_04_lastAdminCannotBeDeactivated` — plus role-change variant, plus the concurrency variant: two admins deactivate each other simultaneously → exactly one succeeds, the other gets `LAST_ADMIN`.
- `AC_SEC_01_studentCannotCallAdminOrDev` — every `AdminApi` method and `Session.admin()/dev()`/`faculty()` from a student session → `ACCESS_DENIED`, nothing changes, an `ACCESS_DENIED` row exists.
- `AC_API_01_failuresCarryCodeAndKeyOnly` — for a sample of failures (validation, stale, not found, injected SQL error, injected `NullPointerException`): `UmsApiException` has code + key, message has no SQL/stack/hash, `INTERNAL_ERROR` has an incident id that exists in `app_error_log`.
- `MU_02`, `MU_05`, `MU_09_consoleCannotReadPasswordHash`, `MU_12_noSecretsInLogs`.
- **`MUST_CHANGE_PASSWORD`**: a new or reset user can only change password or close the session; after changing, all facades work.

### STOP C — "Facades and listener"
Full acceptance table (AC ID → test → result), error mapping table for all new codes, state machine tables for Account and Session. Await review.

---

## Step 12 — Docs and PR · `docs: p1 readme, adrs and changelog`

- README: bootstrap admin, session settings, audit behaviour.
- ADR 013 "Login commits failed-attempt counters before throwing"; ADR 014 "Per-session facade proxies and CallerContext"; ADR 015 "Activity log is best-effort, change history is transactional" (supersedes nothing; refines 008).
- `docs/runbook.md`: "Locked-out user", "Last admin lost" (bootstrap only works when no active admin exists), "Audit drops > 0".
- CHANGELOG `[Unreleased]`.
- Push `phase-1/identity`, `phase-1/audit`, `phase-1/facades`; open **one** PR `phase-1/facades` → `main`. CI must be green. **Do not merge.**

### STOP D — "P1 complete" (final)
CLAUDE.md §8 report plus: PR URL and CI run URLs; coverage per module (core service/domain ≥ 80 %); PIT on `ums-core` (informational); carried-forward decisions; a separate **"Process deviations"** subsection; retrospective (what to change in P2's prompt). Await review — I merge after approval.

---

## P1 must not

- Store, log or return a password, hash or temporary password anywhere except the single `IssuedPassword` return value.
- Read-then-write `failed_attempts` in Java, or roll back the counter on a failed login.
- Let any denied, expired or locked call skip the audit log.
- Expose `ActivityEventBus` through any facade (P6), or add ORG/TERM tables (P2).
- Use `Thread.sleep` in tests for time-based behaviour — use the injectable `Clock`.
- Merge any PR.
