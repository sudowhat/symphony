# Orchestrator — Design Note (rationale beside the profile; operational law lives in orchestrator_profile.md)

## Architecture in one diagram (both lifecycles converge on SRTL)
```
                      requests/[NEW]_REQ-n.md  (human intake, verbatim)
                                 │ spawn
        android-dev              ▼                    content-web
  Architect ─ tickets + route  Architect/Composer   Composer → [DRAFT] ⇄ [REVISION] (Critic loop)
      │                                                  Critic → [FINAL] ─ Designer bridge → *_APPROVED
[APPROVED] ─QA→ [READY_FOR_DEV] ─Dev→ [DONE]            *_APPROVED ─Tester→ *_VERIFIED ─Impl→ *_RFT ─Tester→ *_FIXED
      │              │                  │ HUMAN GATE                                            │ HUMAN GATE
   CANNOT ────────── AUTO resolver ──── └── human adds <T>-SRTL line ──► SRTL review ◄──────────┘
   (Arch/Designer → SRTL → escalate, bounded 3)          *_STALE ──► HALT + human, always
```

## Key decisions & why
0. **Per-project invocation (`init <project> orchestrator`) — user ruling 2026-07-21, reversing the spec's literal single-entry-point reading.** Uniform init grammar (no parser special case), standard one-project Path Integrity, matches the existing per-project daemon scripts; "spanning all projects" is achieved by N parallel instances, which also dissolves the cross-project concurrency design into nothing.
1. **Route-position is derived, not stored.** A saved pointer can desync from renames done by agents/humans; deriving "first line neither complete nor in-flight" from ticket statuses each pass makes restart deterministic and free. Only claims + rings are written state — both constant-size.
2. **Human authorizes SRTL by writing a `<T>-SRTL` route line.** Reuses the existing authority channel (Architect-authored route file) instead of inventing an approvals store; the human IS allowed to append (SRTL lines only) as the gate-opener.
3. **Per-project strict serial = concurrency AND git strategy.** Symphony already mandates one-agent-at-a-time on one shared branch; the Orchestrator enforcing one in-flight line per project satisfies claims, hotspots, and merge-conflict avoidance with zero new git machinery (no branch-per-ticket — rejected as contradicting global-skill's shared-branch workflow).
4. **Rotate-before-spawn** ring (per spec): crash-safe, self-healing, token-free.
5. **CANNOT bounded at 3** (Architect/Designer → SRTL → human): matches existing resolution paths; attempt count derived by counting the ticket's ESCALATION/claim history lines (bounded tail read).
6. **Orchestrator on `haiku4.5-medium`:** the loop is names-only filesystem observation — the cheapest model is the correct model; anything smarter invites unwanted reasoning.

## Corner-case ledger (all 18 spec cases → resolution)
1 collision → claims + per-role serialization · 2 exit-no-transition → claim timeout, retry×2 next-model, escalate · 3 STALE → halt+human · 4 batch→serial → route encodes staging (QA lines then Dev lines) · 5 same-project concurrency → serial; cross-project → parallel · 6 git → decision 3 · 7 interrupt/queue/independent → project overlap + priority + depends (profile §Intake) · 8 REQ depends → hold until dependency `[TICKETED]`+complete · 9 restart → derived position + claims resume · 10 gate timeout → park + daily re-notify; queue continues on other tickets · 11 stuck IN_PROGRESS → timeout escalation, never auto-rename another agent's claim · 12 CANNOT loop → bounded 3 · 13 guard weakening → refuse + escalate (Orchestrator never edits guards; detects via lock-red batch-close rule) · 14 path integrity → refuse project, escalate, continue others · 15 cross-lifecycle request → two linked REQs · 16 Designer bridge → capsule line completes when `*_APPROVED` exists · 17 future roles → 3-line plug-in · 18 SRTL dual mode → route-line entry (review) vs CANNOT-scan entry (unblock).

## Risks
Model-CLI adapter drift (legend placeholders — Grok id unfilled); polling granularity vs agent runtimes (60s is safe); human forgets INBOX (daily re-notify only — deliberate, no nagging); route authored out of oldest-first order per role could make a self-selecting agent grab a different ticket than the line intended (Architect authoring duty — documented in profile; worst case = same-role work done in swapped order, still valid states).

## Phases
P1 (now): profile + wiring + files (this delivery) — human runs the daemon script manually per project. P2: single all-projects daemon `orchestrate.ps1` + Task Scheduler registration. P3: optional capped audit log (model↔ticket) + INBOX viewer.

## Recommendation
Ship P1; drive whatdate's current queue through it as the pilot; adopt P2 after one clean batch.
