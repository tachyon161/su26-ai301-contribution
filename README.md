# Contribution #1: [Feature Request]: Automatically change workflow tab layout on mobile screen

**Contribution Number:** 1

**Student:** Greg Kimatov

**Issue:** https://github.com/Comfy-Org/ComfyUI_frontend/issues/2891 

**Status:** Phase III Complete

---

## Why I Chose This Issue


This issue stood out to me because responsive UI development is an area where I have direct experience. I've built component-driven interfaces using Lightning Blits and TypeScript, which gives me a solid foundation for implementing breakpoint-aware layout logic. The feature request is well-defined and maps closely to patterns I've worked with before, making it a strong candidate for a first contribution to this codebase.

I'm also motivated by the broader context this work lives in. ComfyUI is a widely-used tool in the AI image generation space, and improving its accessibility on mobile devices directly expands who can use and experiment with these models. As someone interested in how AI tools reach end users, contributing to the frontend experience feels meaningful beyond the technical problem itself. I'm looking forward to engaging with a production-scale codebase in this space and developing a stronger familiarity with the project's architecture and conventions.

---

## Understanding the Issue

### Problem Description

There's a poor mobile user experience when managing multiple workflows. 

### Expected Behavior

Multiple workflows should remain easily accessible without requiring precise scrolling. Tab layout should adapt to screen size while maintaining the horizontal tab paradigm.


### Current Behavior

No visual indication of additional tabs beyond the visible area.


<img width="784" height="361" alt="Screenshot 2026-06-13 at 11 18 47 AM" src="https://github.com/user-attachments/assets/589ef991-912a-4b26-a872-8d54c45669bc" />


### Affected Components

The primary edits: src/components/topbar/WorkflowTabs.vue (responsive logic + CSS), and the parent that mounts it (GraphView.vue / topbar layout) for the conditional render. Likely touched: src/components/topbar/WorkflowTab.vue (drag behaviour), src/components/sidebar/tabs/WorkflowsSidebarTab.vue (if it becomes the mobile surface), and a new src/composables/.../useResponsiveLayout.ts. Possibly: the settings definitions file if you add a layout-mode setting, plus i18n locale files for any new setting labels.

---

## Reproduction Process

### Environment Setup



### Steps to Reproduce

1. Install pnpm
2. Run pnpm dev:cloud to proxy all API requests to the test cloud environment
3. Login and navigate to main workflow window
4. Connect to the app on a mobile device as well
5. Notice the poor UX on mobile devices

### Reproduction Evidence

- **Commit showing reproduction:** https://github.com/tachyon161/ComfyUI_frontend/tree/fix-issue-2891
- **Screenshots/logs:**
  <img width="1509" height="776" alt="Screenshot 2026-06-13 at 11 25 31 AM" src="https://github.com/user-attachments/assets/71997885-8b6a-4055-9e1b-2fea46678e34" />

- **My findings:** The app wasn’t running on mobile devices at first because I needed to allow dev server access from remote IP addresses in the .env file. Also, logging in through a separate device only worked if I used an email. It did not work with GItHub Auth because of a Firebase Auth error, which is expected behavior.

---

## Solution Approach

### Analysis

The root cause is a layout that assumes horizontal space that doesn't exist on mobile. WorkflowTabs.vue renders open workflows as a horizontal strip of PrimeVue SelectButton toggles inside a ScrollPanel, and its scoped CSS pins each tab to min-width: 90px. On a ~380px phone viewport, three or four tabs already saturate the bar, so the component falls back to its overflow behaviour — chevron scroll arrows plus a WorkflowOverflowMenu dropdown. That overflow path is a desktop affordance (mouse-wheel horizontal scroll, hold-to-scroll arrows); on touch it's cramped and awkward. Nothing in the component is viewport-aware — its only conditional branches key off distribution (isDesktop/isCloud/isNightly) and a settings toggle (Comfy.UI.TabBarLayout), never screen width. So the layout never adapts no matter how narrow the screen gets.

### Proposed Solution

Make the workflow navigation viewport-aware. Below a mobile breakpoint, stop rendering the horizontal tab strip and surface open workflows through the existing WorkflowsSidebarTab.vue instead, which is what the issue author suggested. This reuses an already-built component rather than inventing a new mobile-only UI, and a vertical list scrolls naturally under touch and scales to any number of workflows. Gate the switch behind a small reactive isMobile flag (and ideally an overridable setting) so tablet users aren't forced into one mode.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** When the viewport drops below a mobile/tablet breakpoint, the horizontal workflow tab bar becomes unusable — tabs squash to their 90px minimum and spill into a fiddly scroll-arrow overflow. The fix is to detect narrow viewports and switch workflow switching to a vertical sidebar layout automatically.

**Match:** The codebase already contains every pattern this needs:
  - Reactive environment branching. The component imports isDesktop/isCloud/isNightly and already conditionally renders chunks of template (v-if="isDesktop", v-if="isIntegratedTabBar"). The same v-if approach extends cleanly to an isMobile flag.
  - VueUse is already a dependency. useScroll and useOverflowObserver are imported here, so useMediaQuery is available with no new package.
  - A settings-driven layout toggle. Comfy.UI.TabBarLayout ('Legacy' vs integrated) is the established precedent for letting layout mode be configurable.
  - An existing sidebar workflow list. WorkflowsSidebarTab.vue already renders open/saved workflows in a vertical tree, so the mobile surface mostly exists.

**Plan:** 
1. Add a useResponsiveLayout.ts composable wrapping useMediaQuery('(max-width: 768px)') to expose a reactive isMobile, so the flag is reusable beyond this one component.
2. In WorkflowTabs.vue, consume isMobile and wrap the horizontal strip (the ScrollPanel + overflow arrows + overflow menu) in v-if="!isMobile". Move the per-tab min-width: 90px rule behind a non-mobile selector so it can't force overflow on small screens.
3. In the parent that mounts the tab bar (GraphView.vue / topbar layout), branch the render: horizontal tabs on desktop; on mobile, suppress them and ensure the workflows sidebar is reachable (auto-register or auto-open WorkflowsSidebarTab at narrow widths).
4. Adapt WorkflowTab.vue's Pragmatic Drag-and-Drop — horizontal reorder logic conflicts with vertical touch scrolling, so either switch to vertical reordering inside the sidebar or disable reorder on mobile.
5. (Optional, on-brand) Add a Comfy.UI.WorkflowLayout setting (Auto / Tabs / Sidebar) mirroring the existing TabBarLayout precedent, so tablet users can override the automatic switch. Requires a settings-definition entry and i18n locale strings.
6. Update tests: extend the existing tab tests (tests-ui for the composable; browser_tests Topbar.ts/SidebarTab.ts fixtures) with a narrow-viewport case asserting the strip is hidden and the sidebar surfaces the workflows.

**Implement:** https://github.com/tachyon161/ComfyUI_frontend/tree/fix-issue-2891

**Review:** Confirm against the project's contribution guidelines — TypeScript throughout (the repo is TS + <script setup lang="ts">); no new runtime dependencies (VueUse already present); new user-facing strings added to i18n locale files, not hardcoded; the change is additive and desktop behaviour is byte-for-byte unchanged when isMobile is false; CSS changes scoped, not global; lint/format pass (the repo enforces both).

**Evaluate:** Verify by running the dev server with --listen and loading from a phone (or Chrome DevTools device emulation) at several breakpoints — 375px, 768px, 1024px — opening 5+ workflows and confirming the strip disappears on mobile, the sidebar lists and switches workflows, and the desktop strip with its scroll-arrow overflow is untouched above the breakpoint. Test the resize boundary live (drag the window across 768px) to confirm the flag is reactive and nothing flickers or loses the active workflow. Run the unit and browser test suites.

---

## Testing Strategy

### Unit Tests

- [x] Test case 1 — honors the user setting on a wide viewport: with useMediaQuery stubbed false (wide), position resolves to whatever Comfy.Workflow.WorkflowTabsPosition returns ('Topbar' → 'Topbar', 'Sidebar' → 'Sidebar'). Asserts the existing setting is still authoritative on desktop.
- [x] Test case 2 — narrow viewport forces Sidebar regardless of setting: with useMediaQuery stubbed true and the setting 'Topbar', position resolves to 'Sidebar'. Asserts the mobile override wins.
- [x] Test case 3 — reactive flip across the breakpoint: toggling the media-query ref false → true → false flips position 'Topbar' → 'Sidebar' → 'Topbar' on the same computed. Asserts reactivity without losing the underlying setting.

### Integration Tests

- [ ] Integration scenario 1 — strip hidden, sidebar surfaced on narrow viewport: render the canvas at ≤767px with several open workflows; assert [data-testid="topbar-workflow-tabs"] is absent and the workflows sidebar badge shows the open count. Not yet written (browser_tests).
- [ ] Integration scenario 2 — desktop unchanged + reactive boundary: at ≥768px assert the topbar strip renders as before; resize across 768px and assert the layout flips without losing the active workflow. Not yet written (browser_tests).

### Manual Testing

                     Before                                      After (below 768px)
    Topbar strip     Cramped horizontal toggles + scroll-arrows	 Gone — [data-testid="topbar-workflow-tabs"] not rendered
    Canvas space	   Reduced by the strip	                       Reclaimed
    Workflow switching	Tap tiny tabs / chevrons	               Via the workflows sidebar tab (full-height, touch-friendly list)
    Sidebar badge	   Hidden unless setting was Sidebar	         Shows open-workflow count
    Desktop (≥768px)	—	                                         Byte-for-byte identical
  


The horizontal tab strip disappears, the canvas gets that vertical space back, and workflows move into the existing sidebar panel (reached via its toggle, with a count badge as the cue). 

This approach trades a cramped-but-visible strip for a clean-but-hidden one. That's a genuine product judgment, not a clear win, so I'll follow up with the community regarding the product direction we want to head in.

---

## Implementation Notes

### Week 3 Progress

Instead of a new setting, I made the existing one viewport-aware: a small composable resolves the effective position, forcing 'Sidebar' below the md breakpoint (≤767px) and otherwise deferring to the stored setting. The three call sites that read the raw setting now read the resolved value, so the topbar strip, the sidebar "Open" section, and the count badge all respond to viewport consistently.

Challenges / things that shaped the result:

- The drag-reorder reconciliation turned out to be unnecessary: WorkflowTab.vue is used only inside the topbar strip, which GraphCanvas removes from the DOM entirely below 768px. The handlers unmount; there's nothing to guard.
- The forced sidebar-open was deliberately skipped — even today's explicit-Sidebar setting doesn't auto-open the panel; it relies on the badge as a cue. Matching that keeps the change consistent and non-intrusive.
- One test-authoring snag: vi.hoisted(() => ref(...)) failed because hoisting runs before the vue import. Fixed by letting the vi.mock factory reference a module-level ref lazily (only read when useMediaQuery is actually called at test runtime).
- An IDE diagnostic flagged the WorkflowsSidebarTab.vue import as unresolvable — a stale .vue false positive; vue-tsc (pnpm typecheck) confirmed it's clean.


### Weeks 4-5 Progress

Shipped a viewport-responsive fix for the workflow tab strip, pushed to fix-issue-2891 on my fork (commit a914a5070). Two files:

`src/components/topbar/WorkflowTabs.vue` — Added mobile detection via useBreakpoints(breakpointsTailwind).smaller('md') (the repo's existing pattern, matching MiniMap.vue). Below 768px, tabs now get enlarged touch targets (min-width: 120px, min-height: 44px) and stop shrinking (flex-shrink: 0), so they stay tappable and overflow into the existing scroll-chevron/overflow-menu affordance instead of squishing. Desktop is untouched — all mobile rules are scoped behind a workflow-tabs-container-mobile class.

`browser_tests/tests/workflowTabsMobile.spec.ts` — New @mobile e2e test (runs on the Pixel 5 / 393px project): opens several workflows, asserts every tab's bounding box is ≥120×44px and that the strip stays visible and scrolls on overflow.

All quality gates pass: format, typecheck, lint, knip, typecheck:browser, browser-test eslint/oxlint, and the 4 unit tests.

Challenges / things that shaped the result:

- I initially built the wrong solution: I hid the tab strip on mobile and surface workflows through the sidebar (a new useResponsiveLayout composable + reusing the existing WorkflowTabsPosition setting). It passed all gates. It turned out the issue author proposed the opposite solution: keep the horizontal strip and enhance it for mobile (VueUse breakpoints, bigger touch targets, overflow indicators). The entire first implementation was scrapped.

- A unit test that violated the project's own testing rules. I first wrote unit tests asserting the mobile CSS class appears on the container. Lint flagged them (testing-library/no-container), and they were exactly the style/change-detector tests AGENTS.md prohibits. Touch-target sizing is a layout concern happy-dom can't verify anyway. Removed them and moved that verification to the Playwright e2e layer, which is the correct place for it.

### Code Changes

- **Files modified:**
WorkflowTabs.vue:
  - Added useBreakpoints import + isMobile detection (smaller('md'))
  - Bound a workflow-tabs-container-mobile class on the container
  - Scoped mobile CSS: tabs min-width: 120px, min-height: 44px, flex-shrink: 0; overflow arrows min-width: 44px

- **Files added:**
workFlowTabsMobile.spec.ts:
  - @mobile e2e test asserting touch-target sizing and overflow-scroll behavior

- **Key commits:**
https://github.com/tachyon161/ComfyUI_frontend/commit/a914a5070120719f1478e378cf71d909ca628807#diff-fdeeb0f02d5a417a00d49a6e2e2190132f3f326af8d68cf615f523c053af2a81
https://github.com/tachyon161/ComfyUI_frontend/commit/fd0295b75a1b629716a97ac32408356ff2cd6417 (no longer exists)

- **Approach decisions:**
  - Enhanced-strip over sidebar-replacement — align with the issue author's actual request; keeps open workflows always visible (your core concern) rather than hiding them behind a toggle.
  - Reuse the repo's useBreakpoints pattern — no new composable, no new setting; consistent with MiniMap, AssetBrowserModal, etc.
  - Didn't redesign the overflow menu into a "+X count" badge — the author listed it as a nice-to-have; the existing chevrons + overflow menu already surface hidden tabs and now engage on mobile because tabs no longer shrink. Kept scope to the bug.
  - Desktop byte-for-byte unchanged — original min-width: 90px retained; every mobile rule scoped behind the new class.

(OLD DECISIONS, not relevant anymore)
  - Reuse the existing setting, don't add a parallel one — avoids duplicate/conflicting UX and keeps the diff to one composable + three one-line call-site swaps.
  - Centralize resolution in one composable — the three consumers were independently calling settingStore.get('Comfy.Workflow.WorkflowTabsPosition'); DRYing them behind useWorkflowTabsPosition() means the responsive rule lives in exactly one place.
  - Breakpoint (max-width: 767px) — pairs with Tailwind's md (≥768px) so "narrow" and "desktop" partition cleanly with no overlap at the boundary.

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
