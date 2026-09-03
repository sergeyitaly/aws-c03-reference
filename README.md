# AWS SAA-C03 Exam Prep Reference

A single self-contained, bilingual (English / Ukrainian) study tool for the
AWS Certified Solutions Architect – Associate (SAA-C03) exam. No build step,
no dependencies — one HTML file you can open directly in a browser, publish
as a Claude Artifact, or host as a static site.

**Live site:** https://sergeyitaly.github.io/aws-c03-reference/
**Published artifact:** https://claude.ai/code/artifact/1278e1e1-4e79-464d-af7b-484f39af0fc3

## What's in it

- **21 reference sections** (flashcards, diagrams, comparison tables)
  covering every SAA-C03 domain — IAM, EC2, storage, databases, networking,
  serverless, containers, data & analytics, ML, monitoring, security, and
  migration/cost — mirroring the exam guide's own section breakdown.
- **100-question practice quiz**, domain-tagged, with instant per-answer
  explanations, a live sidebar progress bar per category (Identity /
  Compute / Data / Networking / Storage / Integration / Ops / Security),
  and an end-of-quiz results summary showing which topics need more study.
- **Two architecture games**: *Build the Architecture* (drag services into
  the right slot for a given priority) and *Spot the Flaw* (find the
  architectural mistake in a described design).
- **Sorting Rush**: a falling-item drag-and-drop game with 500 configured
  AWS service scenarios across 5 categories (cost-effective, least-effort,
  high-availability, security, performance), three difficulty levels, a
  cross-session spaced-repetition "review missed" mode, and a per-item
  adaptive difficulty rating that makes historically-hard items reappear
  more often.
- **Sandbox**: a free-form architecture board — drag any of **95 AWS
  services** (from a radial Services Tree catalog) onto a canvas, connect
  them (only documented, real service pairings link — hold on a node to
  see its possible connections first), group services inside a VPC/Subnet
  container that visually encloses them, then read the auto-generated
  architecture-workflow diagram. Includes zoom/pan/fit-to-view, drag-to-
  edge auto-scroll, full undo/redo/clear history, and a **"Load a
  reference architecture..."** dropdown covering **27 curated, exam-
  relevant patterns** (three-tier, serverless API, event-driven, data
  lake, streaming, ML inference, containers, hybrid networking, landing
  zone, DR/backup, multi-region active-active, chatbot, fraud detection,
  and more) that populates the board with real services and connections —
  each pattern also shows how closely your own board matches it, with a
  verified real AWS reference-architecture URL on a 100% match.
- **Exam Traps**: 35 "sounds right but isn't" exam gotchas (SCP never
  blocking the management account, IGW vs NAT Gateway direction, Multi-AZ
  vs cross-Region, Gateway vs Interface VPC endpoints, and more), grouped
  by the real domain weighting (Secure 30% / Resilient 26% /
  High-Performing 24% / Cost-Optimized 20%) — each one rendered as a small
  box-and-arrow diagram contrasting the wrong assumption (red) with the
  correct one (green), not just a text card.
- **Glossary**, exam-pattern cheat sheet, and a service-limits reference.
- Dark/light theme toggle, background music, and sound effects.
- Responsive mobile layout for every mode, including the Sandbox board.

## Files

| File | Purpose |
|---|---|
| `SAA-C03-reference.html` | The entire application — open this in a browser or publish it as an Artifact. |
| `index.html` | A one-line redirect to `SAA-C03-reference.html`, so the GitHub Pages root URL works without renaming the canonical file. |
| `SAA-C03-service-reference.md` | The source-of-truth outline the HTML content was built from (21 numbered sections). Consult this before adding new content, to keep the HTML aligned with the original plan. |
| `.claude/skills/manage-saa-c03-reference/SKILL.md` | How to safely extend this file — bilingual parity rules, validation workflow, exam-scope gotchas. **Read this before editing.** |

## Using it

Just open `SAA-C03-reference.html` in any browser. Everything — quiz
progress, game scores, Sorting Rush ratings and missed-item list, language
and theme preference — is saved to your browser's local storage, so it
persists across sessions on the same device/browser.

## Extending it

See the **manage-saa-c03-reference** skill for the full guide. The short
version:

1. Check the confirmed out-of-scope AWS service list in the skill before
   adding a service that isn't already covered.
2. Every piece of content exists in parallel `_EN` and `_UK` arrays that
   must stay the same length and same conceptual order.
3. Validate before publishing: a `node --check` syntax pass, a structural
   Node script (id/parity/domain-distribution checks), and a `jsdom`
   functional smoke test — the skill has ready-to-adapt snippets for all
   three.
