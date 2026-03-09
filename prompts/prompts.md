# First analysis

```
This is new offer we got from our customer. Check all content in `official-input/` directory (especially `specification.md`) and try to identify imporant question that can have significant impact on final estimate of the project.

Consider things like
* potential gaps in scope
* clarification of deliveries and post-production support
* testing/security checks requirements

Write the questions into `internal-notes/initial-analysis-questions.md`
```

Then write answers directly to document with `Answer:` prefix

# Initial design phase

```
We want to design high level architecture for the offer we have from our customer. Initial requirements are in `official-input/` directory (check all files but especially `specification.md`). Additionally read all answers in `internal-notes/initial-analysis-questions.md`.

Based on that prepare basic design with high level components we will have to deliver.

Consider not only implementation but also other things like
* CI/CD pipelines
* Testing scripts
* Any relevant documentation

If there are multiple reasonable alternatives in design - write up to 3 most relevant alternatives to the design with pros and cons.

Write the result into `internal-notes/initial-design.md` (optionally with mermaid diagrams when suitable)
```

# Project plan

Write `internal-notes/design-decisions.md` - to select alternatives or give another design instructions

```
Based on `internal-notes/initial-design.md` and `internal-notes/design-decisions.md` make project plan where you plan epics and stories to be implemented to fully deliver the solution.

Please check also `official-input/specification.md` to cross-check if something was forgotten.

Don't forget to add also non-implementation stories (like final testing, writing documentation).

At the end - add summarizing table with epics - stories - brief description.

Write down the project plan to `internal-notes/project-plan.md`
```

Iterate on design decissions while project plan looks reasonable (it contains everything, it is properly structured)

# Estimate

```
Based on `internal-notes/project-plan.md` prepare estimates for each epic.
Read also `internal-notes/initial-design.md` and `internal-notes/design-decisions.md` so that you understand more details.

* Make estimates in man-days assuming it will be done be mid to senior developer (mix of such developers)
* If epic is too big, split estimate into smaller parts
* Use fibonachi numbers for estimates 0,1,2,3,5,8,13,21 (anything more is too big)

Write down the estimates to the `internal-notes/estimates.md` in form of table.
```

# Offer

TBD - don't forget to add all assumptions to offer