---
name: security-fix-verification
description: "Structured verification workflow for security vulnerability patches. Use this skill on any QA issue whose title or description contains 'SQL injection', 'XSS', 'CORS', 'injection', 'security fix', 'vulnerability', 'auth bypass', or 'token', or when a PR patches files under middleware/auth*, routes/, or a service that builds dynamic queries, or when an issue mentions 'scan for similar patterns'. This skill is distinct from general code review and from the security-auditor (which is proactive and full-scope). Use it to verify a targeted fix: confirm the vulnerable pattern is gone, scan for similar patterns still in the codebase, check test coverage for the patched path, and issue a structured PASS/CONDITIONAL PASS/FAIL verdict."
---

# Security Fix Verification

This skill answers: **is this security fix correct, complete, and covered — and are there similar vulnerabilities still in the codebase?**

Run this skill whenever you are verifying a merged or proposed security fix. It does four things that general QA skills do not: checks the fix itself for correctness, scans the full codebase for structurally similar vulnerabilities, checks test coverage for the patched path, and checks for regressions.

---

## When to use this skill

Invoke when any of the following are true:

- The QA issue title or description contains: SQL injection, XSS, CORS, injection, security fix, vulnerability, auth bypass, token validation, rate limit, RBAC, privilege escalation, sanitisation, or similar security terms
- The PR touches `middleware/auth*`, `routes/`, or any file that builds dynamic DB queries or parses HTTP headers
- The issue description mentions "scan for similar patterns" or "check other occurrences"
- You are verifying a fix that was flagged by a security audit or pen test

Skip to `qa-acceptance-testing` only if the change is a documentation update with no code changes.

---

## Step 1 — Fix correctness check

Read the diff (PR diff, or the changed files listed in the issue). For each changed file:

1. **Identify the vulnerable pattern** — what was the original vulnerability? (e.g. raw string interpolation into SQL, unescaped user input reflected into HTML, missing `Authorization` header check, JWT secret hard-coded)

2. **Verify the pattern is removed, not just wrapped** — a wrapped vulnerability is still a vulnerability. Check for:
   - Input is validated/sanitised **before** use, not just caught in a try/catch after
   - Parameterised queries or ORM methods used instead of string concatenation
   - Escaping or encoding applied at the **output** layer for XSS, not just stripped on input
   - Secret material (keys, tokens) read from environment or a secrets manager, not a fallback literal
   - Middleware guard placed **before** the handler, not after

3. **Check for bypasses** — are there alternative code paths that reach the same vulnerable sink without going through the fix? (e.g. the middleware is added to one router but another router serves the same resource)

Document findings as:

```markdown
### Fix Correctness

- Vulnerable pattern: [describe it]
- Fix approach: [parameterised query / output encoding / middleware guard / …]
- Pattern removed (not wrapped): YES / NO — [evidence]
- Bypass paths found: NONE / [list any]
```

If bypass paths exist, the verdict cannot be PASS — stop here and flag them.

---

## Step 2 — Pattern scan

The fix addresses one instance. The question is: are there other instances of the same class of vulnerability?

Run targeted searches based on the vulnerability type. Examples:

| Vulnerability type | What to grep for |
|---|---|
| SQL injection via string concatenation | Template literals containing SELECT/INSERT/UPDATE/DELETE, `query(` calls with string `+` concatenation |
| XSS via unescaped template output | React's raw HTML injection prop, `innerHTML =` assignments, Vue's `v-html` directive, Handlebars triple-brace syntax `{{{ }}}` |
| JWT/auth — missing verification | `jwt.decode(` without a corresponding `.verify(`, `req.headers.authorization` used without validation middleware |
| Hard-coded secrets | Patterns like `secret:` followed by a quoted string of 16+ alphanumeric chars, `password =` with a hard-coded string literal |
| CORS misconfiguration | `origin: "*"` or `Access-Control-Allow-Origin: *` on routes that should be restricted |
| Unvalidated request parameters | `req.query.` or `req.params.` values fed directly into a DB call or template without sanitisation |
| Missing rate limiting | Login, password-reset, OTP, or token-exchange routes without a rate-limit middleware call above the handler |

Adjust the patterns to match the specific vulnerability in this fix. Run grep across the full repo.

Document findings as:

```markdown
### Pattern Scan

Searched for: [pattern(s) used]
Findings:
- [filepath:line] — [brief description of what was found]
- [filepath:line] — [brief description]
- (none found)
```

For each finding, classify it:
- **Same vulnerability** — exploitable in the same way as the patched instance → must create a follow-up issue
- **Same pattern, lower risk** — structurally similar but less exploitable in context → note and flag
- **False positive** — similar-looking code that is safe for a documented reason → note and exclude

---

## Step 3 — Test coverage check

Security fixes without test coverage tend to regress. Verify:

1. **Is there at least one test that exercises the patched code path with a malicious or boundary input?**
   - For SQL injection: a test that passes a `'; DROP TABLE` or `' OR 1=1` style value and asserts it is rejected or sanitised
   - For XSS: a test that passes a script-injection string and asserts it is escaped in the output
   - For auth bypass: a test that sends a request without a valid token and asserts 401/403
   - For hard-coded secrets: a test or startup check that asserts the app fails to start when the required env var is absent

2. **Is the test asserting the right thing?** A test that passes the malicious input but only checks HTTP 200 (not the output content) is not a meaningful security test.

3. **Does the test run in CI?** Check if the test file is in the standard test directory and follows the project's test runner conventions.

Document findings as:

```markdown
### Test Coverage

- Patched code path: [file and function/route]
- Test found: YES / NO
  - Test file: [path]
  - Malicious input tested: [what input]
  - Assertion: [what the test asserts]
- Test is in CI: YES / NO / UNKNOWN
- Coverage verdict: ADEQUATE / MISSING / PARTIAL
```

If coverage is MISSING, the overall verdict must be CONDITIONAL PASS or FAIL — not PASS — and a follow-up test issue must be created.

---

## Step 4 — Regression check

Confirm the fix has not broken existing functionality:

1. **Check if existing tests still pass** — if you can run the test suite, run it and report the result. If you cannot run tests, look for CI run results linked in the PR or issue.

2. **Look for tests that the fix may have invalidated** — e.g. a test that previously relied on the now-blocked behaviour (a test that expected a certain response that came from an unsanitised path)

3. **Check the happy path** — the endpoint or function that was fixed should still work correctly for legitimate inputs. If you can exercise it, do so.

Document findings as:

```markdown
### Regression Check

- Test suite run: YES (all pass) / YES (N failures) / NO (CI results used) / NO (unable to run)
- CI result: [pass/fail/unknown, with link if available]
- Tests invalidated by the fix: NONE / [list any]
- Happy path verified: YES / NO / UNABLE
```

---

## Step 5 — Verdict

Combine the findings from Steps 1–4 into a structured verdict.

```markdown
## Security Fix Verification Verdict

**Issue:** [issue identifier and title]
**Fix:** [one-sentence summary of what was patched]

### Summary

| Check | Result |
|---|---|
| Fix correctness | PASS / FAIL — [reason] |
| Bypass paths | NONE / [count] found |
| Pattern scan | CLEAN / [N] similar findings |
| Test coverage | ADEQUATE / MISSING / PARTIAL |
| Regression check | PASS / FAIL / UNKNOWN |

### Verdict: PASS / CONDITIONAL PASS / FAIL

**PASS** — all checks pass, no similar patterns found, test coverage adequate, no regressions.

**CONDITIONAL PASS** — fix is correct and no regressions, but one or more of the following:
- Similar patterns found → follow-up issues required (list them)
- Test coverage is missing or partial → follow-up test issue required

**FAIL** — fix is incorrect or incomplete, bypass exists, or regressions introduced.

### Required follow-up issues

[List any follow-up issues that must be created before this verdict can be upgraded]
```

---

## Follow-up issue creation

For every **Same vulnerability** finding in the pattern scan, create a follow-up QA or security issue:

- Title: `[Security] [Vulnerability type] in [filepath] — similar pattern to [original issue identifier]`
- Description: include the file path, line number, the vulnerable pattern found, and a link to the original issue
- Priority: match the priority of the original issue
- Assignee: the same agent or team that owns the original fix

For missing test coverage, create a separate issue:

- Title: `[Security test] Add regression test for [fix description] ([original issue identifier])`
- Description: specify the patched file/route, the malicious input that should be tested, and the expected assertion
- Priority: one level below the fix issue (if fix was high, test is medium)

---

## Related skills

| Skill | When to use it alongside this skill |
|---|---|
| `regression-scope-assessment` | Before this skill when the security fix touches multiple layers — to scope what else could have regressed |
| `qa-acceptance-testing` | After this skill to run the full AC-based acceptance test on the fixed feature |
| `security-constraint-test-standards` | When writing new security tests as a result of Step 3 finding missing coverage |
| `security-auditor` | When the pattern scan reveals a systemic class of vulnerability that warrants a full audit |
| `completion-evidence-gate` | Before issuing a PASS verdict — confirms evidence requirements are met |

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*
