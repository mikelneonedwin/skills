# Implementation and Browser Verification Checklist

Use this as a practical minimum. Adapt checks to the actual application; do not mechanically test irrelevant features. Never claim a check was completed unless it was run.

## Baseline and architecture

- [ ] Read the task-specific repository instructions and inspect the current application before modifying it.
- [ ] Identify existing routes, shared layouts, design tokens, component primitives, data contracts, tests, and package scripts.
- [ ] Confirm the app starts with the repository's actual commands and tools, when possible.
- [ ] Keep global styling and repeated component behavior in the design system; audit exceptions for a real reason.
- [ ] Keep page files focused, reuse appropriate route layouts, and avoid both giant files and unnecessary micro-components.

## Visual quality and responsive behavior

- [ ] Inspect the rendered application in a browser, not only source code or a static mockup.
- [ ] Inspect primary routes at desktop and mobile widths; check intermediate breakpoints when appropriate.
- [ ] Check overflow, clipping, alignment, content density, type hierarchy, contrast, spacing, and consistency between routes.
- [ ] Compare the output with accepted references and verify that useful lessons have been translated rather than copied wholesale.
- [ ] Verify realistic content lengths, long labels, empty result sets, and other likely stress cases.
- [ ] Ensure skeletons represent the real content structure and do not create layout shifts unnecessarily.
- [ ] Confirm animations are purposeful, lightweight, and respect reduced-motion preferences when relevant.

## Interaction and states

- [ ] Exercise every primary user journey from entry point to completion.
- [ ] Click all meaningful visible controls: navigation, buttons, menus, tabs, dropdowns, dialogs, filters, sorting, pagination, search, and other task-specific controls.
- [ ] Submit forms with valid data, invalid data, missing required fields, and relevant boundary cases.
- [ ] Verify loading, empty, error, success, and disabled states where applicable.
- [ ] Confirm feedback is understandable and actions have visible consequences.
- [ ] Verify keyboard operation, focus visibility/order, semantic controls, and accessible names for important interactions.
- [ ] Check failed network requests, console errors, runtime warnings, and navigation dead ends when the tools permit.
- [ ] Confirm no real integration is falsely represented by a local mock or hard-coded success state.

## Regression and delivery checks

- [ ] Rerun tests relevant to the changed flows after implementation and after meaningful corrections.
- [ ] Run available type checks, lint, build, and other repository checks appropriate to the scope.
- [ ] Recheck affected pages after global token or shared-component changes; shared changes can create cross-route regressions.
- [ ] Confirm fixes resolve the reported issue rather than merely hiding its symptom.
- [ ] Look for unnecessary dependencies, oversized assets, avoidable client-side JavaScript, expensive animations, and obvious bottlenecks.
- [ ] Record which checks actually ran and any relevant commands that could not be executed.

## Evidence notes

For meaningful findings, capture the route, viewport, state, expected behavior, observed result, and a screenshot or test reference when available. If a tool is unavailable, describe the limitation rather than implying verification occurred. Browser evidence, automated tests, and code inspection complement one another; none alone proves the entire application is correct.
