---

## name: failure-map description: Map realistic failure scenarios for code, workflows, automations, and AI systems. Produces structured incident reports for failures that haven't happened yet - but plausibly will. argument-hint: "\[system, file, workflow, integration, or AI deployment to review\]" disable-model-invocation: true

# Failure Map

You are now in failure-mapping mode.

Your job is to read a system - code, a workflow, an automation, an integration, or an AI deployment - and produce a structured failure analysis written in past tense, as if each incident has already occurred.

This is not a bug hunt. The system may be working correctly today. You are looking for places where a future change, upstream drift, human handoff, or environmental shift could cause a failure that no one saw coming.

The central question for everything you read: what reasonable thing could someone do here - or what could reasonably change - that would break this in a way that isn't immediately obvious?

---

## Scope

$ARGUMENTS

- If the user provides specific files, workflows, or system descriptions, scope your analysis there.
- If nothing is specified, ask the user to define the starting scope. Choose one meaningful subsystem - don't try to cover everything at once.
- For code: focus on production logic, not config files or boilerplate unless they contain logic other systems depend on.
- For workflows and automations: focus on handoff points, state transitions, and trigger conditions.
- For AI systems: focus on input surfaces, output trust boundaries, memory and retrieval layers, and human oversight gaps.

---

## Workflow

1. **Read the full system.** If it's code, read callers and callees, not just the file in isolation. If it's a workflow, trace it from trigger to terminal state. Understand how data moves, where decisions get made, and what assumptions are load-bearing.

2. **Map the fragility.** Use the catalogue below to identify the patterns. For each one, ask: what would a reasonable person do here - or what would a reasonable system change do - that would break this? If you can't name a plausible scenario, move on.

3. **Write the failure reports.** For each fragility, write a fictional incident report in past tense from the perspective of someone investigating after the fact. Be specific. Name the components, the values, the conditions, the consequences.

4. **Produce the output file.** Write all reports to a single markdown file. Ask the user to confirm the output path, defaulting to `FAILURE-MAP.md` in the project root.

---

## Fragility Catalogue

### Code and Systems

**1. Implicit ordering dependencies**Code or initialization that must run in a specific sequence but doesn't enforce it. Setup methods that must be called first. Lists that assume sorted input. Step 3 silently requiring step 1. *Future edit:* Someone reorders calls, adds a step between existing ones, or calls a method before the object is ready.

**2. Semantic coupling through shared mutable state**Two components communicating through a shared object - a dict, a module-level variable, an attribute on a passed-in object - instead of explicit arguments and return values. *Future edit:* Someone modifies one component's use of the shared state without knowing the other depends on it.

**3. Stringly-typed contracts**Logic that depends on the exact value of strings - status fields, dict keys, column names, format strings. These create invisible contracts between producers and consumers with no type enforcement. *Future edit:* Someone renames a status string, adds an enum variant the existing switch doesn't handle, or changes a key in one place but not another.

**4. Assumptions baked into data transformations**A function that processes data assuming a particular shape, range, or distribution - non-empty list, positive value, no nulls, string matching a pattern. Nothing enforces these assumptions. *Future edit:* Someone changes the upstream data source, adds a new code path that feeds different data in, or relaxes upstream validation.

**5. Coincidental correctness**Code that produces the right result for the wrong reason. A condition that works because two variables are always equal today. A loop that doesn't handle the empty case but has never been called with one. *Future edit:* The coincidence stops holding. Input space widens, a new exception type appears, the previously-equal variables diverge.

**6. Non-atomic compound operations**A sequence of operations that should be atomic but isn't - check-then-act patterns, multi-step state updates with no rollback, file operations that assume no concurrent access. *Future edit:* Someone adds concurrency, introduces an early return, or moves the code to a context where interruption is possible.

**7. Invisible invariants**Relationships between data that must be maintained but are enforced only by convention. No assertion, type, or test enforces the relationship. *Future edit:* Someone updates one side of the invariant without knowing about the other.

**8. Load-bearing defaults**Default values - function parameters, config settings, environment variables - that the code subtly depends on. The code would behave incorrectly with a different value, but nothing documents this. *Future edit:* Someone changes the default to something equally reasonable, or a caller starts passing an explicit value nobody anticipated.

**9. Implicit resource lifecycle**Resources - connections, file handles, locks, threads - created with cleanup that depends on a specific control flow. No context manager or finalizer guarantees cleanup. *Future edit:* Someone adds an early return, raises an exception, or refactors the function and the cleanup code is no longer reached.

**10. Version-coupled assumptions**Code that depends on the behavior of a specific version of a dependency, runtime, or protocol - relies on dict ordering, an undocumented library side effect, or the exact format of a third-party error message. *Future edit:* The dependency upgrades, the runtime changes, or the undocumented behavior shifts.

---

### Workflows and Automations

**11. Handoff gaps**A step passes work to the next step - person to person, system to system, or tool to tool - without confirming receipt or validating the handoff completed. The system assumes handoffs succeed unless told otherwise. *Future edit:* The next step is delayed, skipped, or fails silently. No one knows the handoff didn't land.

**12. Human-in-the-loop assumption**The workflow was designed assuming a human would catch a certain class of error - a bad value, an edge case, a missing field. That assumption isn't written down anywhere. *Future edit:* The human step gets skipped, automated, or handed to someone who doesn't know what to look for.

**13. Approval bypass risk**A workflow can reach a terminal state - sent, published, fulfilled, charged - without passing through a defined approval or review step. The path exists but isn't protected. *Future edit:* Someone adds a shortcut, the approval step gets marked optional, or an automation triggers the terminal state before review completes.

**14. Silent failure path**The workflow fails - an email doesn't send, a record doesn't update, a sync doesn't complete - but no alert fires, no log is written, and no one notices until downstream effects surface. *Future edit:* The failure condition was always there. Volume increases or a system change finally triggers it at scale.

**15. Trigger-state coupling**An automation triggers based on an external state - a tag, a flag, a field value - that was set by a different system or a human. The trigger assumes the state is trustworthy and current. *Future edit:* The upstream system sets the state incorrectly, the human sets the flag too early, or a third system modifies the state value between set and trigger.

**16. Recovery path missing**The system has no defined manual override, rollback, or recovery procedure. When something breaks mid-process, the options are: wait, start over, or improvise. *Future edit:* Something breaks mid-process and the operator has no clear path forward.

---

### AI Systems

**17. Hallucination surface**The AI's output is trusted and acted upon - written to a record, sent to a user, passed to another system - without verification or a defined confidence threshold. *Future edit:* The AI encounters an input type outside its reliable range. The output looks plausible and passes without review.

**18. AI overreach risk**The AI can take actions - write, delete, send, update - that exceed the intended scope of the task it was given. The permissions were granted for convenience, not because the scope was deliberately designed. *Future edit:* The AI interprets a prompt broadly, encounters unexpected input, or is given a task that overlaps with a sensitive operation.

**19. Memory or context staleness**The AI operates on stored context - retrieved memories, injected documents, cached summaries - that may no longer reflect current state. Nothing flags when the context is outdated. *Future edit:* The underlying data changes. The AI continues reasoning from the old version without knowing it.

**20. Retrieval bias**The retrieval layer consistently surfaces certain documents, records, or memories over others - not because they are more relevant, but because of how they were indexed, weighted, or formatted. *Future edit:* A user asks a question that should surface newer or higher-priority context. The old, well-formatted record dominates the results instead.

**21. Data trust drift**The AI system was built assuming upstream data quality stays consistent - clean formats, reliable sources, accurate values. No monitoring watches for drift. *Future edit:* A source changes format, adds nulls, shifts encoding, or degrades in quality. The AI processes it without error but produces degraded output.

---

### Business and Integration Risk

**22. Business rule encoding**A business rule - pricing logic, eligibility criteria, compliance threshold - is baked into a technical system that business stakeholders don't control and may not know exists. *Future edit:* The business rule changes. The technical system doesn't get updated because no one knew it was the source of truth.

**23. Dependency on undocumented behavior**The integration or workflow depends on how a third-party system currently behaves - response format, default field value, pagination pattern - rather than on what the API contract actually guarantees. *Future edit:* The third-party system updates or optimizes an internal behavior. The integration breaks in a way the changelog never mentions.

**24. User impact without user signal**A failure affects end users - wrong data shown, missing confirmation, incorrect fulfillment - but the failure doesn't generate an error the system can detect. Users experience the problem silently. *Future edit:* The failure condition was always present at low frequency. Volume or a new user segment brings it to detectable levels, but the signal comes from support tickets, not from the system itself.

---

## Failure Report Format

Write each report as a self-contained section in past tense, as if the incident has already happened.

```markdown
### <Short incident title>

**Severity:** Critical | High | Medium | Low
**Domain:** Code | Workflow | Automation | AI System | Integration | Business Process
**Component:** <specific file, step, trigger, or system element involved>
**Fragility type:** <category from the catalogue>

#### What happened

<2-4 sentences describing the failure as if it already occurred. What did users, operators, or the team observe? Be specific - name the symptom.>

#### The change that caused it

<Describe the edit, upstream change, or environmental shift that triggered the failure. It should sound reasonable - something that would pass review or go unnoticed. Include a plausible motivation.>

#### Why it broke

<Explain the hidden assumption or fragility that the change violated. Reference the specific component, field, rule, or behavior that created the risk.>

#### How it was caught

<How did this surface? Monitoring? A user complaint? A downstream data problem? A support ticket? Be honest - if nothing would have caught it proactively, say so.>

#### Impact

<Who was affected and how? Include business impact, user impact, data integrity risk, or trust damage as applicable.>

#### Hardening suggestions

<1-3 concrete, actionable steps. What should become a test, an SOP, an alert, a validation rule, or a refactor? Be specific enough that someone could implement the suggestion directly.>
```

---

## Calibration

Aim for 3-7 reports per system or module, depending on complexity. Each one should be genuinely plausible.

**Avoid:**

- Current bugs. If you find something that's broken today, surface it separately - don't write a fictional report about it.
- Adversarial scenarios. Every imagined change should be something a reasonable person would do with good intentions.
- Unlikely rewrites. The imagined change should be small and local - a feature addition, a config update, a dependency bump, a new team member following their own judgment.
- Generic observations. "This step has no error handling" is a note. A failure map report names the specific scenario where that gap causes a specific, consequential failure.
- Severity inflation. Not everything is Critical. Silent data corruption is Critical. A clear error in an uncommon path is Low.

**Aim for:**

- Cause and effect are non-obvious. The change is in one place; the failure shows up somewhere else.
- Fragilities that are structural, not surface-level. A missing null check is less interesting than a load-bearing assumption woven through three systems.
- Reports that make a reader say "I wouldn't have thought of that" - not "well, obviously."

---

## Output Structure

```markdown
# Failure Map Report

**System:** <what was reviewed>
**Date:** <today's date>
**Reviewed by:** failure-map

## Summary

<A short paragraph on the overall fragility posture. How many reports? What are the dominant themes? Are the risks independent or systemic?>

## Failure Reports

### 1. <title>
...

### 2. <title>
...

## Themes and Recommendations

<After all reports, step back. Identify cross-cutting themes. If several reports point to the same structural weakness, call it out. Suggest the one or two changes that would address the most risk at once.>

## What Should Become a Test, SOP, Alert, or Refactor

<A summary table of hardening suggestions organized by action type so a team can prioritize without re-reading the full report.>
```

---

## Critical Rules

- Read before writing. Never write failure reports for something you haven't traced thoroughly.
- Be specific. Every report must reference actual components, fields, states, or trigger conditions - not abstract descriptions of a class of problem.
- Be plausible. Every imagined scenario must have a motivation a reasonable person would recognize.
- Don't fix it. Your job is the report, not the remediation. Hardening suggestions describe what to do - you don't implement them unless asked.
- Separate real bugs. If you find something actually broken, surface it immediately - don't embed it in a fictional report.
- Ask when uncertain. If a pattern seems fragile but you're not sure, ask before including it.

---

*failure-map is part of The Cartographer Method - a consulting framework for mapping, stabilizing, and deploying AI-augmented systems.Originally inspired by Matthew Honnibal's* `claude-skills` *pre-mortem command, then rewritten and expanded for operational workflow, AI, and systems-integration review.*