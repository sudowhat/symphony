---
name: question-induction
description: "CONTENT-WEB PROJECTS ONLY — specifically wisdom-capsules. Complete standard operating procedure and workflow for authoring, reviewing, validating, duplicate-chaining, and deploying new questions into the Wisdom Capsules Challenge question bank. Load whenever adding or updating assessment questions. The workflow generalizes to any assessment bank; the paths, npm scripts, and host commands are Wisdom Capsules' own."
---

# Question Induction Skill — Wisdom Capsules Challenge

**What this solves:** adding new assessment questions to a high-stakes, certified exam bank is not just appending a JSON object. It requires strict structural schema conformity, anti-cue distractor calibration, hint safety, exact replica prevention, duplicate-chaining for fair lottery sampling, editorial map synchronization (`qmap.md`), pre-deploy invariant testing, and atomic zero-downtime production deployment.

Reference implementation: `wisdom_capsules-folder` (`quiz/validate-bank.js`, `quiz/replica-check.js`, `quiz/select-questions.js`, `quiz/qmap.md`, and `quiz/question-bank.pilot.json`).

---

## 1. The 6-Step Induction Workflow

```text
[Step 1: Draft] ────────► [Step 2: Replica Gate] ────► [Step 3: Bank Induction]
Author complete question   Run replica-check on JSON    Insert into question-bank.pilot.json
with 4 options & catalysts  Adjudicate replica/chain    Increment questionCount header

[Step 4: QMap Sync] ────► [Step 5: Test Suite] ──────► [Step 6: Atomic Deploy]
Update qmap.md tables      Run all 8 test suites       Copy to /var/lib/ active bank
npm run quiz:replicas:sync Check selection & security  Switch release & restart API
```

---

## 2. Step 1: Question Package Authoring Standards

Every question must be a complete, self-contained evaluation package:

```json
{
  "id": "WC-Q534",
  "prompt": "Concise realistic scenario describing a situation where principles collide (approx 25–45 words).",
  "options": [
    { "id": "A", "text": "Plausible adjacent error representing a common misconception.", "catalyst": "Explains why option A fails the deeper principle without mocking the user." },
    { "id": "B", "text": "Correct nuanced response embodying the core capsule principle.", "catalyst": null },
    { "id": "C", "text": "Overgeneralization or wrong-level application of the lesson.", "catalyst": "Explains why option C is an overextension or premature inference." },
    { "id": "D", "text": "Avoidance or superficial action that misses the core issue.", "catalyst": "Explains why option D does not resolve the underlying conflict." }
  ],
  "correctOption": "B",
  "explanation": "Clear, comprehensive explanation detailing why the correct option applies and how the principle resolves the tension.",
  "sourceCapsules": [
    { "number": 12, "slug": "what-makes-a-nation", "title": "What Makes a Nation?" }
  ],
  "conceptTags": ["nation-building", "shared-identity", "cultural-preservation"],
  "difficulty": "medium",
  "cognitiveType": "application",
  "scenarioDomain": "civic",
  "status": "active"
}
```

### Non-Negotiable Authoring Rules:
1. **Options**: Exactly 4 options (`A`, `B`, `C`, `D`). Exactly one `correctOption`.
2. **Catalysts**: The correct option **must** have `"catalyst": null`. All 3 incorrect options **must** have a non-empty diagnostic catalyst explaining why that choice is flawed.
3. **Anti-Cue Balance**: Do NOT make the correct answer conspicuously the longest or most nuanced option. Avoid "all of the above" or caricature distractors.
4. **Hint Safety**: The first sentence of the prompt/explanation must not give away the correct option ID.
5. **Calibrated Rubrics**:
   - `difficulty`: `easy`, `medium`, `medium-hard`, `hard`
   - `cognitiveType`: `application`, `distinction`, `causal-reasoning`, `synthesis`, `ethical-reasoning`, `epistemic-reasoning`, `interpretation`
   - `scenarioDomain`: `work`, `personal`, `family`, `civic`, `education`, `digital-media`, `community`, `relationships`, `technology`, `society`, etc.

---

## 3. Step 2: Pre-Insertion Replica & Duplicate Gate

Before inserting a question into the main bank, test it against the versioned `replica-v1` detector:

```bash
# Save candidate question to a temporary file
node quiz/replica-check.js --candidate-file ./candidate.json
```

### Possible Outcomes & Required Actions:
- **`CANDIDATE_CLEAR` (Exit 0)**: No duplicate or near-replica found. Proceed to Step 3 as a standalone question.
- **`REPLICA_FOUND` (Exit 1)**: An exact normalized stem match exists. **Forbidden.** Merge any improved phrasing into the existing question ID; discard the new ID.
- **`DECISION_REQUIRED` (Exit 2)**: High-similarity pair detected. Adjudicate:
  - **Legitimate Variant (Duplicate)**: Tests the same judgment in a different context. Assign or extend a duplicate chain (e.g. `"duplicateChain": "D034"`).
  - **Genuinely Distinct**: Mark as unique; no chain needed.

---

## 4. Step 3: Bank Induction & ID Invariants

1. Add the question to `quiz/question-bank.pilot.json` under the `questions` array.
2. Increment the `questionCount` field in the header:
   ```json
   {
     "bankVersion": "unified-reviewed-2026-08-26",
     "status": "active",
     "questionCount": 534,
     "questions": [ ... ]
   }
   ```
3. Invariant: Question IDs must be contiguous (`WC-Q001` through `WC-Q534`) with zero gaps or duplicate IDs.

---

## 5. Step 4: Synchronize Question Map (`quiz/qmap.md`)

1. Update the **Reviewed scope** header in `quiz/qmap.md` (e.g. `WC-Q001 through WC-Q534`).
2. Update the **Review summary & Statistics**:
   - Total questions (e.g. 534)
   - Duplicate chains count
   - Chained questions count
   - Standalone questions count
   - Independent sampling units (`samplingUnits = standalone + duplicateChains`)
3. Add the row to the **ID Map** table:
   ```markdown
   | WC-Q534 | 12 | — | medium | FROZEN |
   ```
   *(or specify `D034` in the Duplicate Chain column if chained)*.
4. **Synchronize Fingerprints**:
   ```bash
   npm run quiz:replicas:sync
   npm run quiz:replicas:check-qmap
   ```
   *Note: Never edit the machine-generated SHA-256 fingerprint hashes in `qmap.md` by hand; always use `sync`.*

---

## 6. Step 5: Full Regression Test Gate

Run the complete validation suite. Every check must pass before any code is committed:

```bash
npm run quiz:replicas:test
npm run quiz:replicas:check-qmap
npm run quiz:validate
npm run quiz:release:validate
npm run quiz:selection:test
npm run quiz:server:test
npm run quiz:security:test
npm run quiz:test
npm run seo:test
npm audit --omit=dev
```

### What These Guards Enforce:
- `quiz:validate` & `release:validate`: Schema integrity, option counts, catalyst rules, active status.
- `quiz:selection:test`: Validates the sampling invariant (`duplicateChain ?? question.id`), ensuring chains get exactly 1 lottery ticket, no attempt receives >1 member of the same chain, and all units remain selectable without frequency bias.
- `quiz:test`: Guards against secret leakage, ensures `question-bank.pilot.json` is never bundled into `dist/`, and validates frontend source boundaries.

---

## 7. Step 6: Atomic Production Deployment

1. **Commit & Push**:
   ```bash
   git add quiz/question-bank.pilot.json quiz/qmap.md
   git commit -m "content: add WC-Q534 to canonical Challenge bank"
   git push origin master
   ```

2. **Deploy Release to Host**:
   - Create new release directory at `/srv/wisdom-capsules/releases/<SHA>`.
   - Run `npm run build` on host (generates static pages).
   - Write `RELEASE` marker file.

3. **Atomically Install Active Bank**:
   - Copy release question bank to `/var/lib/wisdom-capsules/question-bank.active.json.tmp`.
   - Set ownership and permissions:
     ```bash
     chown wisdomcapsules:wisdomdbbackup /var/lib/wisdom-capsules/question-bank.active.json.tmp
     chmod 0640 /var/lib/wisdom-capsules/question-bank.active.json.tmp
     ```
   - Atomically rename over active bank:
     ```bash
     mv -f /var/lib/wisdom-capsules/question-bank.active.json.tmp /var/lib/wisdom-capsules/question-bank.active.json
     ```
   - Verify SHA-256 hash match:
     ```bash
     sha256sum /var/lib/wisdom-capsules/question-bank.active.json /srv/wisdom-capsules/releases/<SHA>/quiz/question-bank.pilot.json
     ```

4. **Switch Symlink & Restart API**:
   ```bash
   ln -sfn /srv/wisdom-capsules/releases/<SHA> /srv/wisdom-capsules/current_tmp && mv -Tf /srv/wisdom-capsules/current_tmp /srv/wisdom-capsules/current
   systemctl restart wisdom-capsules-challenge
   systemctl is-active wisdom-capsules-challenge
   ```

5. **Verify Perimeter**:
   - Check journal logs (`journalctl -u wisdom-capsules-challenge`).
   - Confirm port 3001 is loopback only (`ss -ltnp '( sport = :3001 )'`).
   - Public smoke test: `curl -sSI https://wisdomcapsules.com/challenge/`.
   - Verify active bank metadata directly on host.
