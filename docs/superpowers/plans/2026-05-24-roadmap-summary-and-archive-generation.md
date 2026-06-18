# Roadmap Summary and Archive Generation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Generate roadmap progress data from a machine-readable manifest, expose the next dependency-satisfied plan, and snapshot completed roadmap versions into an archive before starting a new one.

**Architecture:** Keep `docs/roadmap/mvp-roadmap.md` as the human-readable index, but move plan metadata into a manifest that scripts can read deterministically. Generate a compact summary file from the manifest so counts and next-plan pointers are derived, not hand-maintained. Treat archived roadmap versions as immutable snapshots and document the usage flow in `AGENTS.md` so agents always read the generated summary before choosing the next plan.

**Tech Stack:** Markdown, JSON, Node.js, pnpm.

---

## Source Documents

- `docs/roadmap/mvp-roadmap.md`
- `docs/roadmap/current-stage.json`
- `AGENTS.md`
- `scripts/verify-roadmap-state.mjs`

## Scope

This plan updates roadmap metadata, generation scripts, archive workflow docs, and the validation gate. It does not change application runtime behavior.

## File Structure

- Create: `docs/roadmap/roadmap.manifest.json`
- Create: `docs/roadmap/current-summary.json`
- Create: `docs/roadmap/archive/`
- Modify: `docs/roadmap/mvp-roadmap.md`
- Modify: `AGENTS.md`
- Create: `scripts/roadmap/render-summary.mjs`
- Create: `scripts/roadmap/archive-version.mjs`
- Modify: `scripts/verify-roadmap-state.mjs`
- Modify: `package.json`

## Task 1: Add a Machine-Readable Roadmap Manifest

**Files:**
- Create: `docs/roadmap/roadmap.manifest.json`
- Modify: `docs/roadmap/mvp-roadmap.md`

- [ ] **Step 1: Define the manifest shape**

Create a roadmap manifest with one top-level version and a `plans` array. Each plan entry must include:

```json
{
  "id": "01",
  "file": "docs/superpowers/plans/2026-05-22-01-project-foundation.md",
  "title": "Project Foundation",
  "order": 1,
  "dependsOn": [],
  "status": "completed",
  "outcome": "Electron, Vite, React, TypeScript, Ant Design, packages/backend, and packages/shared workspace exists and verifies locally"
}
```

Use the existing `mvp-roadmap.md` table as the source for plan files, order, dependencies, and outcomes.

- [ ] **Step 2: Add a manifest note to the human roadmap**

Update `docs/roadmap/mvp-roadmap.md` so readers know the manifest is the machine-readable source and the summary is generated from it.

## Task 2: Generate Summary Data from the Manifest

**Files:**
- Create: `scripts/roadmap/render-summary.mjs`
- Create: `docs/roadmap/current-summary.json`
- Modify: `package.json`

- [ ] **Step 1: Write the renderer**

Implement a Node script that:

1. reads `docs/roadmap/roadmap.manifest.json`
2. computes:
   - `totalCount`
   - `completedCount`
   - `remainingCount`
   - `nextExecutablePlan`
3. writes `docs/roadmap/current-summary.json`

The `nextExecutablePlan` should be the first `pending` plan in roadmap order whose `dependsOn` plans are all `completed`.

- [ ] **Step 2: Expose the renderer as a workspace script**

Add a script entry:

```json
{
  "scripts": {
    "roadmap:render": "node scripts/roadmap/render-summary.mjs"
  }
}
```

- [ ] **Step 3: Render the initial summary**

Run the new renderer once and confirm the generated summary matches the manifest.

## Task 3: Add Archive Snapshot Support

**Files:**
- Create: `scripts/roadmap/archive-version.mjs`
- Create: `docs/roadmap/archive/`
- Modify: `AGENTS.md`

- [ ] **Step 1: Write the archive script**

Implement a Node script that copies the active roadmap set into a timestamped archive folder, including:

- `docs/roadmap/roadmap.manifest.json`
- `docs/roadmap/current-stage.json`
- `docs/roadmap/current-summary.json`
- `docs/roadmap/mvp-roadmap.md`

Use a stable archive path such as `docs/roadmap/archive/<yyyy-mm-dd-or-release-label>/`.

- [ ] **Step 2: Document the workflow in AGENTS.md**

Add a short section that says:

- read the generated roadmap summary before choosing the next plan
- use the next dependency-satisfied pending plan as `nextExecutablePlan`
- archive the active roadmap set before starting a new version
- do not edit archived roadmap snapshots

## Task 4: Harden Validation

**Files:**
- Modify: `scripts/verify-roadmap-state.mjs`
- Modify: `package.json`

- [ ] **Step 1: Validate the manifest and generated summary**

Extend the verifier so it:

1. checks the manifest exists and is well-formed
2. recomputes the summary values from the manifest
3. verifies `docs/roadmap/current-summary.json` matches the recomputed values
4. confirms `nextExecutablePlan` is dependency-satisfied

- [ ] **Step 2: Expose validation as a script**

Keep `pnpm verify:roadmap-state` as the single entry point for the roadmap guardrail.

## Validation Order

- [ ] Run `pnpm roadmap:render`
- [ ] Run `pnpm verify:roadmap-state`
- [ ] Confirm `docs/roadmap/current-summary.json` matches the manifest

## Commit Boundary

Commit the manifest, renderer, archive helper, validation updates, and AGENTS workflow together so the roadmap state machine stays internally consistent.
