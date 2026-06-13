# Contribution #1: [Feature Request]: Automatically change workflow tab layout on mobile screen

**Contribution Number:** 1

**Student:** Greg Kimatov

**Issue:** https://github.com/Comfy-Org/ComfyUI_frontend/issues/2891 

**Status:** Phase II Complete

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

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

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
