---
layout: post
title: "An approach to building a verified career master with Claude"
subtitle: "A tool to help you build CVs you can defend"
date: 2026-10-01
tags: [career-master, claude, llm, product, job-search]
description: "An open-source Claude plugin that helps you review your career, build a verified master profile and write honest applications."
---

## Why I built it

My CV was not working as well as I expected. The challenge was not a lack of content, but how to condense it and connect earlier roles with what companies ask for today.

Asking an AI to write the CV produced a good-looking result, but the AI does not know your story. Without that context it fills the gaps: it inflates, undersells or invents figures.

So instead of a tool that just produces a CV, I built one that first helps you review your experience, knowledge and results, step by step, and turns them into facts you have confirmed. Only then does it write the CV.

It also maps what you did to the terms recruiters use. Some examples from my own case:

- **Radical Candor and Management 3.0.** I had never heard of them; for me, the way I ran feedback and 1:1s was common sense. Comparing them with my practice, the principles were the same.
- **Transformation and change management.** That is what the reorganisations I led were, including succession planning.
- **Disagree and commit.** Accepting a decision I disagreed with and then working to make it succeed.
- **FinOps.** Bringing a cloud budget under control.

It was a very useful exercise of introspection for me, so I decided to share it. I built it especially for:

- **People returning to the market** after a long break or many years in the same company.
- **Juniors without experience** who need help to write their first CV.

One golden rule: **nothing goes into a document unless you have confirmed it.** It prevents exaggeration and also underselling.

> **Download:** [career-master on GitHub](https://github.com/lgonzalezalo/career-master/releases/latest) (open source, MIT).

## What it does

career-master is an open-source plugin for Claude (Cowork and Claude Code). You open an empty folder, say "let's start", and five skills cover the full journey:

| Skill | Purpose |
|---|---|
| `career-start` | Short onboarding: situation (junior, returning, experienced, career change), document languages and conditions. Creates the folder structure. |
| `master-builder` | Guided interview in 15–20 minute sessions: ingests old CVs, builds the timeline, goes role by role, resolves contradictions, verifies each claim and ends with a private reflection. |
| `skills-matrix` | An honest level per skill (`direct`, `oversaw`, `partial`, `equivalent`, `learning`, `none`), backed by confirmed achievements, plus how to talk about gaps. |
| `star-coach` | STAR interview stories built from confirmed facts, competency coverage and a practice mode. |
| `job-tailor` | Fit score (0–100) for each offer, gaps, tailored CV in PDF, cover letter, form answers and an application tracker. |

Each achievement in the master carries its own rules. Example from the fictional masters in the repository:

```yaml
- id: ventisca-coord-02
  status: confirmed
  text: "I took part in the rollout of the Navision ERP and trained the 5 people in the department."
  verb_level: contributed
  usage_rule: "The project was led by the vendor; I was a key user and internal trainer."
  source: "sources/CV_2013.doc + interview 2026-09-15"
```

- `status`: only confirmed entries can be used.
- `verb_level`: the honest verb (*contributed*, not *led*).
- `usage_rule`: the limits any CV or letter must respect.
- `source`: provenance of the claim.

Before confirming anything strong, the interviewer asks: *"Could you explain and defend this in an interview?"*

## Architecture: agents only where they add value

**The interviewer is the only agent.** A career conversation is open by nature: it must follow answers, detect contradictions, remember previous sessions and adapt to each profile.

**Everything else is a predictable pipeline:**

```text
offer → analyse → score (fixed rubric) → gate → select → [you validate]
      → generate → validate (trace every sentence) → render PDF → tracker
```

Key decisions:

- **Fixed rules.** Fixed scoring weights and thresholds (red below 55, green above 70). Same offer and same master, same result, and failures are easy to locate.
- **Traceability.** Every CV sentence is mapped to a confirmed achievement in a private `trace.md`.
- **Deterministic code where output must be exact.** A Python script renders the PDF (Pandoc + headless Chromium), checks the page count and falls back to printable HTML.
- **Files as the source of truth.** A versioned YAML file, readable by a person, re-read and merged before every write.
- **Humans at the gates.** No claim, selection or deletion without an explicit yes.
- **Referential integrity.** If an achievement is no longer confirmed, everything that depends on it goes back to review.

Today the model applies scoring and trace validation following a written specification. Moving them to code that runs on the user's data is the next step.

## Privacy by design

- **Local first.** Data stays in a folder on your computer, with no telemetry. Conversations go through Claude under your account's terms.
- **Never stored:** date of birth, ID, photo, nationality, marital status, address, health or other people's names, even if old CVs include them.
- **Private reflection.** Never used in a CV, letter or answer.
- **Career breaks on your terms.** It never asks why, you decide what the CV shows, and a break never lowers the score.
- **Sensitive form questions** (nationality, age, health, family) are left for you to answer.
- **Real deletion.** Removed everywhere, in cascade, after confirmation.

## How I checked quality

You cannot review quality into a product without first defining what "correct" means.

- **Review rounds** with Claude agents in parallel, each from one angle: user flows, data contracts, privacy, and UX.
- **Concrete findings.** In a test CV, Pandoc turned "from ~12% to ~5%" into subscript and the approximation disappeared from the PDF: exactly the kind of quiet inflation the project tries to avoid.
- **Automation.** The findings became `tools/check.py`, which runs in GitHub Actions on every push: shared rules identical across skills, valid references, no personal files committable, correct PDF rendering.

The principle: automate everything that can be checked, and spend reviews on what cannot. The same standard I apply to any team I lead.

## Limits

- It only runs inside Claude, and Cowork needs a paid plan.
- Scoring and trace validation depend on the model following the rules, not on code.
- The master takes two to five sessions of 15–20 minutes.
- So far, only one other person apart from me has tried it.

If I started again, I would write the data contracts and checks before the skills: most issues found were inconsistencies between files.

## Next: the pilot

I have used it in my own search for less than a month, so results will come in a separate article.

The next step is a pilot with three to five people, ideally juniors and people returning to the market, to learn how long the master takes, where people get stuck, and whether the final CV represents them. That will decide whether to take it outside Claude.

## Try it

- **Download:** the plugin file (`career-master.plugin`) is in the [latest release](https://github.com/lgonzalezalo/career-master/releases/latest) on GitHub.
- **Install:** in Cowork, add the downloaded file in the plugins section. In Claude Code, run `/plugin marketplace add lgonzalezalo/career-master` and `/plugin install career-master@career-master`.
- **Source code:** the [repository](https://github.com/lgonzalezalo/career-master) is open source (MIT), so you are free to adapt it to your needs.
- **Feedback:** use the [feedback template](https://github.com/lgonzalezalo/career-master/issues/new?template=feedback.md). Issues are public, so please do not include personal data. For private feedback, you can find me on [LinkedIn](https://www.linkedin.com/in/luis-gonzalez-alonso-99b1763a).
