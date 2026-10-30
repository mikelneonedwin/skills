# Final Implementation Review Rubric

The final panel exists to decide whether the actual implementation is acceptable, not to provide encouragement. Reviewers should look for reasons the work is **not yet ready**, support findings with evidence, and re-inspect every correction in subsequent rounds.

## Required reviewer lenses

Use independent reviewers with different primary lenses where subagent delegation is available. One reviewer may cover more than one lens only when the environment or task size makes that necessary.

### Product usefulness and interaction

Assess whether the delivered flows solve the requested problem. Verify that navigation, calls to action, forms, filters, search, menus, dialogs, tabs, pagination, and other visible controls perform their promised behavior. Check the clarity of feedback, validation, loading, empty, error, success, disabled, and recovery states. Identify dead ends, misleading affordances, broken flows, unnecessary friction, or UI behavior that is only simulated.

### Visual design, originality, and reference synthesis

Inspect the rendered product at representative viewports. Compare it with the original brief and every accepted reference. Identify which specific lessons were used well, which were lost, which were borrowed too literally, and whether the result is a coherent product rather than a collage.

Look for weak hierarchy, generic composition, default-library appearance where the brief requires a stronger identity, arbitrary colour or spacing, inconsistent component treatments, repetitive cards, purposeless decoration, typography that feels unconsidered, content that looks like filler, and visual inconsistency between routes. Ask whether the app looks credible for its intended users and use case, not merely whether the screenshot looks fashionable.

### Engineering, responsiveness, accessibility, and performance

Inspect routes, component structure, tokens, shared variants, and layout reuse. Look for huge page components, repeated application shells, duplicated styles, unexplained overrides, unnecessary dependencies, brittle responsive logic, and one-off components that should share a pattern. Check whether the design system was customised coherently rather than left stock or replaced with arbitrary local styles.

Test at desktop and mobile widths, plus intermediate widths where likely to reveal breakage. Check overflow, clipping, tap targets, focus order, keyboard operation, accessible names, semantic structure, contrast, reduced-motion behavior, and other issues relevant to the product. Inspect console errors, failed network requests, runtime warnings, and tests where tools permit. Treat performance concerns as evidence-based findings rather than assumptions.

## Evidence standard

For every finding, provide:

- **Location:** Route, viewport, component, state, or workflow.
- **Observation:** What was actually seen or reproduced.
- **Impact:** Why it undermines usability, product fit, quality, accessibility, maintainability, or trust.
- **Severity:** Critical, high, medium, or low.
- **Expected correction:** A concrete change or result that would address the problem.
- **Evidence source:** Browser reproduction, screenshot, test, reference comparison, DOM/style inspection, code location, or other available evidence.

Do not issue generic findings such as “make it more polished,” “improve UX,” or “this looks like AI.” Translate the criticism into specific, observable symptoms. For a subjective concern, explain how it conflicts with the brief or weakens hierarchy, identity, usability, coherence, or credibility.

## Severity guidance

- **Critical:** Core flow is unusable, significant user data or security risk is introduced, a major page is broken, or a fundamental requirement is not met.
- **High:** A primary workflow is unreliable or confusing; a key responsive layout fails; a serious accessibility barrier exists; the visual direction is fundamentally generic, incoherent, or mismatched to the product; major states are missing; or a shared-system defect affects many screens.
- **Medium:** A meaningful but non-blocking interaction, consistency, hierarchy, responsive, performance, or maintainability issue affects quality.
- **Low:** Local polish that does not materially undermine the task, such as a minor spacing or alignment issue.

Severity depends on context. Do not label a preference critical simply to force compliance.

## Anti-slop assessment

“AI slop” is a shorthand for a result that feels generic, unconsidered, derivative, or generated from a familiar template without product-specific thought. It is not a checklist of forbidden CSS patterns.

Challenge signs such as interchangeable SaaS hero sections, gratuitous gradients/glows, repetitive rounded-card layouts, decorative icons that communicate nothing, arbitrary gradients or colours, stock component appearance that clashes with the reference synthesis, inconsistent spacing, generic or inflated copy, excessive empty space, and features that look interactive but do nothing. Evaluate each sign in context: gradients, cards, rounded corners, and other familiar patterns can be correct when they have a clear rationale.

Ask the more important questions: Does the hierarchy reflect real user priorities? Is the visual language consistent and specific to the product? Do details reinforce the intended use? Does the UI work with realistic content? Is there a clear reason for the major design choices? Does it remain useful beyond its first screenshot?

When there is uncertainty about a current pattern or a specific criticism, consult credible and current external examples when available. Do not browse simply to decorate the review with opinions. Explain how any external evidence applies to the actual product.

## Required verdict

End each review with one of these verdicts:

- **NOT ACCEPTABLE:** Critical or high-impact defects remain, or the implementation misses a fundamental requirement. Specify the required corrections.
- **IMPROVED, NOT ACCEPTABLE:** Material progress is evident, but further concrete corrections are required. Identify what remains and verify previously reported defects.
- **ACCEPTABLE:** The brief is substantially met, there are no unresolved critical or high-severity findings, the core journeys work, the rendered product is coherent and responsive, and verification evidence supports the claims. List only remaining low-priority issues or explicit limitations.
- **BLOCKED:** A specific external constraint prevents meaningful review or correction. Explain what cannot be inspected or tested, what was attempted, and what must change to resume. Do not use this verdict for difficulty or ordinary defects.

“ACCEPTABLE” does not mean perfect or free of every possible improvement. It means the product has reached a defensible quality bar for the requested scope. Reviewers must not demand subjective perfection or restart resolved debates without new evidence.

## Correction loop: no fixed pass limit

After every non-acceptable review, the lead agent must consolidate findings, resolve duplicate reports, rank the remaining issues, challenge weak findings with evidence if appropriate, implement meaningful corrections, and rerun relevant browser flows and tests. Then the panel must inspect the updated product again.

At each subsequent review, the panel must verify whether earlier findings are actually resolved, whether the fixes introduced regressions, and whether the improved implementation reveals new issues. Reviewers should not parrot their previous feedback without reinspection.

Continue until **ACCEPTABLE** or a genuine **BLOCKED** condition. There is no arbitrary round limit. If reviewers disagree, resolve the factual question with reproducible behavior, the brief, the references, implementation details, or applicable accessibility/engineering expectations. Do not allow the lead agent to self-approve on behalf of an available independent panel. Conversely, do not let minor taste disputes create an infinite loop after the material acceptance criteria have been met.

If independent subagents are unavailable, conduct the strongest honest role-based self-review possible and disclose that independent approval could not be obtained. Never invent panel participation or a verdict.
