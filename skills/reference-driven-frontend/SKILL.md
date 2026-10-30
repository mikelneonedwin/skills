---
name: reference-driven-frontend
description: Use this skill when asked to build a new frontend, turn a product idea into a working UI, implement a substantial feature or multi-page flow, or meaningfully redesign an existing interface. Trigger for requests such as “build this UI”, “make this look and feel better”, “implement this dashboard”, “create the app from scratch”, or “finish the frontend”. Use it when the agent is expected to make strong design decisions with little UI direction from the user. Do not use the full workflow for isolated copy, tiny styling fixes, or routine maintenance that does not materially affect the interface.
---

# Reference-Driven Frontend Craft

## Mission

Take a product idea or substantial UI task from brief to a distinctive, polished, responsive, fully working implementation with minimal design-direction burden on the user. Operate as a product designer, visual researcher, frontend engineer, browser tester, and implementation lead. Research real products and visual concepts, extract specific lessons, synthesize them into a coherent design system, build the requested experience end to end, and keep correcting it until an independent review panel judges it acceptable or a genuine blocker makes further progress impossible.

A page that compiles is not necessarily good. A screenshot that looks attractive is not necessarily functional. A passing test suite is not evidence of visual quality. Completion requires evidence across design, behavior, responsiveness, accessibility, performance, and maintainability.

## Non-negotiable principles

- **Own the design work.** Do not push basic design decisions back to the user when the brief and context support a reasonable choice. Infer, record assumptions, and proceed. Ask only when a missing decision materially affects product direction, data integrity, destructive actions, access, or an external commitment.
- **Research before inventing.** Use existing products to learn interaction patterns and visual references to learn art direction. Build a synthesis, not a clone.
- **Judge the rendered product.** Use an actual browser and available testing tools throughout implementation. Do not claim to have inspected, clicked, measured, or tested anything you did not actually inspect, click, measure, or test.
- **Use independent reviewers when available.** The review panel must bring independent criticism, not staged agreement. Never pretend subagents were used if they were not available or did not run.
- **Keep reviewing until acceptable.** The formal reference-selection review is capped at three passes. The final implementation review has no arbitrary pass limit: fix, verify, and return to the panel until it approves or a real blocker prevents progress.
- **Customise the design system, preserve its coherence.** Change theme tokens and shared primitives to fit the product. Do not scatter arbitrary one-off versions of shared UI across the app.
- **Finish the requested scope.** Build complete user journeys and the necessary loading, empty, error, success, and disabled states. Do not mistake an attractive mockup for a working product.
- **Respect the repository.** Inspect existing conventions, dependencies, routes, design tokens, and tests before changing architecture. Preserve unrelated work. Do not overwrite user changes or perform unrelated cleanup.
- **Do not fake production behavior.** If a real service, credential, or permission is unavailable, make the integration boundary clear and finish the rest. Never present a local mock as a completed real-world operation.

## Adapt to the environment

Before acting, identify the capabilities actually available in this agent harness: web search, browser control, screenshots, DOM/style inspection, network/console inspection, image search, code search, test runners, and delegated subagents. Use the strongest available combination. Tool names and access differ between environments; do not assume a specific MCP, browser, framework, package manager, or subagent API exists.

Prefer a browser connected to the user's real development environment for implementation testing. For web research, use search plus direct navigation and interaction when possible. If browser automation or delegated agents are unavailable, use the best available alternative, reduce claims to match the evidence, and disclose material verification limitations in the final report. Do not let a missing optional tool become an excuse to abandon work that can still be completed.

If the environment exposes skills for frontend design, accessibility, framework conventions, or browser automation, use them selectively when relevant. Do not invoke a broad design skill just to satisfy a ritual if it adds no value or conflicts with this workflow. This skill is the orchestrator for substantial UI work.

## Workflow

### 1. Understand the request and inspect the codebase

Read the user's request and inspect the current repository before making a design plan. Identify the framework, routing model, existing visual system, component library, package manager, relevant routes, existing functionality, available tests, and any local instructions such as `AGENTS.md`. Follow repository instructions and established patterns unless they directly prevent the requested result or create a clear defect.

Infer the product's purpose, target users, main jobs-to-be-done, primary user journeys, required screens, important data states, and visual personality. Distinguish explicit requirements from assumptions. Avoid long discovery interviews. Ask at most a focused question when a decision cannot reasonably be inferred and would materially change the implementation; otherwise state the assumption internally and continue.

Set a brief-specific definition of done. Include behavior, visual quality, responsive layouts, maintainability, accessibility, and performance. Use this to guide research and later judge the implementation. Keep work in the requested scope; do not expand a single feature into an invented product roadmap.

### 2. Research functional and visual references

Research before committing to a visual direction. Aim for **five strong functional references and three strong visual references** when the task has enough relevant material. The user's original minimum is three per category, so never pad the set with weak candidates simply to hit a target. A task with limited relevant examples can use fewer when the reason is explicit. Aim for the strongest useful evidence, not the largest pile of links.

**Functional references** are real websites or applications that teach something about product structure, navigation, interaction, information density, workflow, data presentation, forms, responsive behavior, or a related use case. Search broadly, inspect candidate sites directly, and click through the relevant flows. Do not assess a product from its homepage alone when the useful behavior is deeper in the app. If login or a paywall blocks inspection, record the limitation and move on to accessible evidence rather than inventing what lies behind it.

**Visual references** are images or design presentations from sources such as Dribbble, Behance, Awwwards, design portfolios, screenshot collections, or other credible sources. Inspect the actual image at sufficient size when the tool allows it. Look for composition, typography, hierarchy, spacing, colour relationships, density, illustration, motion cues, and overall art direction. Prefer references that can translate into a real responsive product, not only a static concept render.

For each retained reference, record its URL/source, why it is relevant, the exact pattern worth learning from it, what should not be copied, and where the lesson may fit in the target product. Capture screenshots or notes when tools allow it. Do not claim to have inspected source code unless it was genuinely available. Rendered DOM, computed styles, accessible names, fonts, measurements, and network behavior may be inspectable; arbitrary production source code often is not. Use public repositories or publicly exposed assets only when available and appropriate.

Use the browser and search tools honestly. Search results alone are discovery leads, not proof that a candidate's actual interaction or visual quality has been inspected. Do not spend indefinitely searching once the selection has enough varied, relevant evidence. Stop research when the target is met and the references cover the material design questions, or explain why suitable options are scarce.

Do not copy a reference wholesale, reuse its distinctive branding, reproduce proprietary copy, or download assets without appropriate rights. Extract transferable principles and combine ideas from multiple sources into a coherent, original product identity.

### 3. Validate the reference set with a panel

Before implementation, ask an independent panel of subagents to assess the proposed references and synthesis plan. Use genuinely independent perspectives where the environment supports delegation. Give reviewers the original brief, direct reference links or captured evidence, and the proposed mapping from references to product needs. Do not give them only the lead agent's persuasive summary.

The panel should test whether the selected set is relevant, varied, technically feasible, visually credible, and useful for the target users. It should identify weak or redundant references, poor fit, inaccessible evidence, unworkable ideas, and risks of generic or derivative design. Read `references/reference-selection-rubric.md` for the selection rubric and expected response format.

Cap **reference-selection review at three passes total**: initial assessment plus at most two refinement rounds. The lead agent may challenge panel feedback when it is vague, contradictory, impractical, or unsupported, but must cite concrete evidence and reasoning. Do not enter prolonged taste debates. If the third pass ends with some uncertainty, make a reasoned decision and proceed; this cap applies only to reference selection, not final product review.

If true subagents are unavailable, do not fabricate a panel. Use distinct reviewer roles only if the harness provides a genuine way to run separate evaluations; otherwise conduct an explicitly labelled self-review using the same rubric and note that an independent panel was unavailable. This fallback is weaker evidence and must not be represented as independent approval.

### 4. Synthesize the design direction and system

Turn the accepted references into a brief implementation plan. Map patterns to specific product needs: for example, one source may inform navigation, another information hierarchy, another a form flow, and a visual reference may inform type scale or composition. Explain internally why the choices belong together. Avoid a collage of unrelated ideas.

Establish or revise the actual design system **before the UI sprawls**. Choose a coherent visual identity suited to the product rather than automatically accepting framework defaults. Define global tokens and shared patterns for typography, colour, background/surfaces, borders, radii, spacing, content widths, breakpoints, shadows, focus states, interaction states, and motion where relevant. Ensure contrast and readable hierarchy. Let product context guide whether the result is quiet, technical, editorial, playful, utilitarian, expressive, or something else.

When using shadcn/ui, treat it as a set of adaptable primitives, not a finished visual identity. Modify its theme tokens, CSS variables, base component styles, variants, and shared components as justified. Changes should be intentional and propagated through the system. Establish distinct variants only when they express a real semantic or functional difference. Prefer one shared button style with deliberate variants over unrelated button styling on individual pages; the same principle applies to cards, inputs, badges, tables, dialogs, navigation, and other repeated patterns.

Do **not** preserve stock styling merely because it exists. Do **not** replace it with a pile of arbitrary local overrides either. If a design decision repeats, encode it as a token, primitive, variant, or reusable component. Use a one-off exception only when a page genuinely requires it and existing variants cannot express the requirement cleanly. Avoid both giant page components and pointless fragmentation: components should represent meaningful UI sections, behavior, or reuse.

Plan the route and layout hierarchy. Use the framework's layout or nested-route mechanisms for persistent shells, navigation, headers, sidebars, and footers when appropriate. Avoid mounting identical shared chrome manually on every page. Keep page files focused on composing sections and data rather than dumping hundreds of lines of unrelated markup into them. Preserve the repository's existing architecture unless there is a concrete reason to change it.

Before widespread implementation, settle the main information architecture, shared shell, key page compositions, and interaction patterns. Build a thin but representative vertical slice early and view it in the browser; use that evidence to correct the direction before replicating it across every screen.

### 5. Implement the complete experience

Build the requested scope end to end. Use existing components and dependencies where appropriate, add dependencies only when their benefit justifies their cost, and follow the project's language, routing, data-fetching, and state-management conventions. Do not rewrite unrelated functionality or introduce a new architecture without a concrete need.

Break large pages into clear, maintainable components. Use shared route layouts for common structure. Keep component APIs purposeful and avoid needless abstraction. Maintain consistent tokens and variants across all routes. Make the UI data-aware rather than hard-coding a screenshot-shaped page when the product requires real data.

Implement real behavior for the interface elements present. Buttons must perform their stated action. Links must navigate correctly. Forms must validate, submit, report success or failure, and prevent confusing duplicate submissions where relevant. Search, sorting, filters, tabs, pagination, menus, dialogs, keyboard controls, uploads, and other interactions must work as intended, not merely appear clickable. Use realistic state management and connect to the existing backend or APIs when the scope calls for it.

Implement relevant loading states, layout-matched skeletons, empty states, errors, success feedback, disabled states, and recovery paths. Use natural, concise interface copy grounded in the product. Avoid generic AI-style marketing filler, placeholder content left behind, repetitive card grids, and labels that do not explain the user's next action. Animations should be purposeful, lightweight, and respectful of reduced-motion preferences; do not add motion merely to make a screen look busy.

Make responsive behavior deliberate rather than applying a desktop layout and hoping it wraps. Verify key breakpoints and touch targets. Consider keyboard use, focus visibility, semantic HTML, accessible names, and contrast as part of normal implementation, not as a final cosmetic pass.

Backend work is in scope when required to deliver the requested flow, but frontend quality remains the primary purpose of this skill. Inspect and preserve existing data contracts. Never fake successful integration with a real service. If a necessary credential, permission, infrastructure dependency, or external service is unavailable, complete all feasible work and report the precise blocker.

### 6. Use the browser continuously

Start the app and inspect it in a real browser as soon as a meaningful vertical slice exists. Continue browser verification throughout implementation and after meaningful changes. Do not wait until the end to discover that the composition, responsive behavior, or interaction model is wrong.

Exercise actual user journeys, not just route loading. Click controls, submit forms, navigate between relevant states, test validation and error recovery, and inspect success behavior. Check console errors, failed requests, runtime warnings, and layout issues using whatever browser tools are available. Use screenshots to judge the rendered interface and compare it with the accepted references. Re-check desktop and mobile layouts, including intermediate widths when layout transitions are likely to fail.

Use the project's existing test suite and browser automation framework when available. Add or run focused tests for consequential behavior. Verify keyboard and focus behavior, semantic/accessibility basics, and reduced motion where relevant. Check performance for obvious problems such as oversized images, unnecessary JavaScript, expensive rerenders, excessive dependencies, blocking resources, or unneeded animation. Optimize based on evidence; do not guess at speed claims.

After correcting a defect, retest the affected flow and any likely regression surface. Before declaring completion, rerun relevant tests, type checks, lint/build commands when available and appropriate, and browser checks. Do not claim all tests passed if they were not run. Read `references/verification-checklist.md` for the minimum evidence to gather.

### 7. Conduct the final adversarial review panel

Once the implementation is functionally complete and has been tested, submit the **working product**, not just a code diff or screenshot, to an independent review panel. This formal review repeats as many times as necessary. It is not subject to the three-pass reference-selection limit.

Use multiple independent subagents when supported. Assign different critical lenses so the panel does not produce three copies of the same generic review. At minimum, assess product/usefulness and interaction, visual direction and originality, and engineering quality including responsive behavior/accessibility/performance. Every reviewer should have the original brief, the accepted functional and visual references, the current implementation or browser, and the relevant test evidence. Ask reviewers to inspect actual pages and flows wherever their tools allow it. Do not ask them merely whether they like the design.

Tell reviewers to actively try to disprove that the work is ready. They must compare the actual implementation against the product brief and **every accepted reference**, explain what translates well, what was lost, what should not have been borrowed, and whether the resulting app has its own coherent identity. They should look for generic or AI-slop-like visual patterns as concrete symptoms, not as a vague insult. Relevant examples include interchangeable SaaS layouts, arbitrary gradients or glow, repetitive rounded-card grids, weak type hierarchy, decorative icons without purpose, generic filler copy, excessive whitespace or density, inconsistent spacing, stock component styling that contradicts the intended direction, and styling choices without a product rationale. These are warning signs, not automatic violations; judge each in context. If the panel is uncertain about a current design convention or a suspected generic pattern, it may consult credible current examples or design criticism online, then explain how that evidence applies.

The panel must also try real user journeys and stress cases: narrow viewports, empty/loading/error/populated states, awkward but valid input, failed requests where feasible, keyboard navigation, and any high-risk behavior. Check for broken controls, dead ends, misleading affordances, clipping, overflow, inaccessible contrast, inconsistent components, slow or jarring transitions, and defects hidden outside the first viewport.

Require each finding to state the affected page/component/flow, observed evidence, why it matters, severity, and a concrete expected correction. Use `references/final-review-rubric.md` for the full criteria and verdict format.

#### Mandatory correction loop

After every panel review:

1. Consolidate duplicate findings without discarding distinct evidence. Resolve factual disagreements where possible.
2. Rank findings by impact. Fix critical and high-severity behavior, layout, accessibility, visual coherence, or regression problems before minor polish.
3. Challenge weak or incorrect findings when necessary. A challenge must cite the implementation, brief, reference, reproducible behavior, or a relevant standard; do not reject a finding because it is inconvenient or subjective alone.
4. Implement corrections. Revisit the design direction or shared system when repeated symptoms indicate a systemic problem. Do not patch each page independently if the shared token or component is the actual cause.
5. Run the relevant browser interactions and tests again. Confirm the finding is resolved and check for regressions caused by the correction.
6. Return the updated product and evidence to the panel for another independent judgment.

The loop continues until the panel issues an explicit **ACCEPTABLE** verdict and there are no unresolved critical or high-severity findings, or a genuine blocker prevents further meaningful progress. Reviewers must not repeat old criticisms without rechecking them, but they should identify newly visible problems as the work improves. The lead agent cannot declare acceptance on the panel's behalf. If a material disagreement remains, resolve it with evidence. Subjective preferences that do not materially affect the brief, usability, coherence, or quality should not cause an endless stalemate.

A genuine blocker is an obstacle that cannot reasonably be resolved with the available tools, permissions, information, or environment. Difficulty, a failed test, or a substantial design correction is not by itself a blocker. If blocked, complete all remaining feasible work, document attempts and evidence, state exactly what intervention is needed, and do not misrepresent the product as accepted.

If no real subagent panel can be run, perform the strongest honest alternative available. Clearly distinguish self-review from independent panel review in the final report. Never manufacture reviewer identities, quotes, verdicts, or approval.

### 8. Finish with an evidence-based handoff

When the panel accepts the product, summarize the delivered experience and the meaningful design-system or architectural changes. Briefly record the references used and the lessons synthesized when useful. State which tests, build/type/lint checks, and browser flows were actually run, and call out any checks that could not be performed. Mention unresolved low-priority issues or real blockers; do not hide them beneath a general claim that everything works.

Do not end with only “done,” a screenshot, or a list of files changed. The result should make clear what was built, what works, what was verified, and whether the final panel accepted it.

## Supporting references

Read `references/reference-selection-rubric.md` during reference selection and panel assessment. Read `references/verification-checklist.md` while implementing and verifying browser behavior. Read `references/final-review-rubric.md` for every formal final review and its correction loop. These files contain phase-specific criteria; do not load every checklist repeatedly when it is not relevant.
