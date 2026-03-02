---
layout: post
title: "E2E Test Dynamic Selection: Phase 2 - Configs and Sanity"
date: 2026-03-01 12:00:00 +0800
categories: [Testing]
tags: [e2e, testing, ci/cd, playwright]
---

## Phase 2: Ditching the Script for Configs (and Sanity)

> **Note**: This is Part 3 of the series "E2E Test Dynamic Selection".  
> Read [Part 1: Why full E2E doesn't scale](/posts/E2E-test-dynamic-selection) and [Part 2: The Shell Script phase](/posts/E2E-test-dynamic-selection-phase-1) first.

Phase 1 proved we weren't crazy: dynamic selection works. But a giant Bash `case` statement is a ticking time bomb. For Phase 2, we needed something a team could actually maintain without hating their lives.

Architecturally, we made three major upgrades:

1. **From Bash to TypeScript**: We moved to a language that can actually be unit-tested.
2. **From Code to Config**: Hardcoded mapping rules were ripped out and moved into YAML files.
3. **From `--grep` to `--test-list`**: We stopped relying on Playwright regex filtering and built a precise, auditable selection pipeline.

### The Configuration Files

We split the brain of the operation into two YAML files.

**`suites.yaml`** handles the broad strokes (for example, defining what belongs in the default smoke suite):

{% highlight yaml %}
defaultSuite: smoke

suites:
  smoke:
    tags:
      - "@smoke"
{% endhighlight %}

**`mapping.yaml`** is where the actual mapping happens. Instead of mapping a file to a broad tag, we map regex patterns to specific test case IDs (`TC_*`):

{% highlight yaml %}
rules:
  - pattern: "src/pages/UsersPage\\."
    tests:
      - "TC_USERS_CREATE"

  - pattern: "src/pages/ProductsPage\\."
    tests:
      - "TC_PRODUCTS_ORDER"

  - pattern: "src/components/"
    tests:
      - "TC_USERS_CREATE"
      - "TC_PRODUCTS_ORDER"
{% endhighlight %}

To make this work, we embed unique TC IDs directly into test titles:

{% highlight typescript %}
test('flow - create user only TC_USERS_CREATE @flow', async ({ page }) => {
  // ...
});

test('flow - place order with current active user TC_PRODUCTS_ORDER @flow', async ({ page }) => {
  // ...
});
{% endhighlight %}

The strategy is simple: we still keep tags (like `@smoke`) to guarantee critical paths always run, but for surgical precision we rely on TC IDs. This is not either/or—we union both sets.

### The Execution Pipeline

Here is how the Phase 2 pipeline flows under the hood:

{% highlight text %}
git diff → changedFiles
            ↓
     mapping.yaml pattern match → mappedTests (TC ID set)
     suites.yaml tags           → suiteTags (tag set)
            ↓
  npx playwright test --list    → listedTests (full test inventory)
            ↓
  filter(listedTests, suiteTags ∪ mappedTests)
            ↓
  write → .tmp/dynamic-test-list.txt
            ↓
  npx playwright test --test-list ".tmp/dynamic-test-list.txt"
{% endhighlight %}

### Why `--list` + `--test-list`? (The Magic Trick)

In Phase 1, we used Playwright's `--grep` to filter tests. That works for simple tag matches, but it quickly falls apart for complex logic.

You cannot maintain a single readable regex that says:

- run anything with `@smoke`,
- OR exactly match `TC_USERS_CREATE`,
- but absolutely ignore `TC_USERS_CREATE_EXTENDED`.

So we took control back.

`npx playwright test --list` gives us the full test inventory as plain text. We parse it in TypeScript, apply arbitrary matching logic, write the selected tests to a file, then pass that file back to Playwright with `--test-list`.

Here is the core filtering logic in `resolve-tests.ts`:

{% highlight typescript %}
export function resolveSelectedTests(
  listedTests: string[],
  suiteTags: Set<string>,
  mappedTests: Set<string>,
): string[] {
  return listedTests.filter((line) => {
    const hitSuite = [...suiteTags].some((tag) => line.includes(tag));

    const hitMapped = [...mappedTests].some((testId) =>
      new RegExp(`(?:^|\\W)${escapeRegex(testId)}(?:$|\\W)`).test(line)
    );

    return hitSuite || hitMapped;
  });
}
{% endhighlight %}

Notice the boundary check (`\\W`) around TC IDs. That prevents `TC_PRODUCTS_ORDER` from accidentally matching `TC_PRODUCTS_ORDER_EXTENDED`.

Because this is pure TypeScript now, we can unit-test selection behavior directly (for example in `resolve-tests.unit.spec.ts`):

{% highlight typescript %}
test('resolveSelectedTests should include suite-tag and mapped-TC matches', () => {
  const listedTests = [
    'example.spec.ts › users to products ... @smoke',
    'example.spec.ts › flow - create user only TC_USERS_CREATE',
    'example.spec.ts › flow - place order ... TC_PRODUCTS_ORDER',
    'example.spec.ts › should not match TC_PRODUCTS_ORDER_EXTENDED',
  ];

  const suiteTags = new Set(['@smoke']);
  const mappedTests = new Set(['TC_PRODUCTS_ORDER']);

  const result = resolveSelectedTests(listedTests, suiteTags, mappedTests);

  expect(result).toEqual([
    'example.spec.ts › users to products ... @smoke',
    'example.spec.ts › flow - place order ... TC_PRODUCTS_ORDER',
  ]);
});
{% endhighlight %}

### CI Integration: No More Black Boxes

With this setup, CI shrinks to one clear npm entry point:

{% highlight yaml %}
# .github/workflows/e2e-playwright.yml
- name: Run dynamic tests
  run: npm run test:dynamic   # -> tsx ./scripts/resolve-tests.ts
{% endhighlight %}

And the logs are now understandable for PR reviewers:

{% highlight text %}
┌─── Dynamic Test Selection (Phase 2) ───
│ Repository:   ..
│ Changed files:
│   src/pages/UsersPage.jsx
│ Suite tags:   @smoke
│ Mapped tests: TC_USERS_CREATE
│ Test list:    .tmp/dynamic-test-list.txt
│ Selected:     3 tests
└────────────────────────────────────────
{% endhighlight %}

You can see exactly what files changed, what rules matched, and why those specific tests were selected.

### Validation CI Run (Phase 2)

Phase 2 only matters if it can survive outside local experiments, so we validated it in a real pipeline run.

The key log point for this post is here:
[GitHub Actions Job Evidence (step)](https://github.com/uestcheng/sample-app/actions/runs/22467801509/job/65077797036#step:10:34).

What this run tells us is exactly what we wanted from the Phase 2 design:

- it works in the default team workflow (regular push-triggered CI);
- it keeps feedback tight (successful run in roughly one minute, not a full-suite drag);
- and it remains reviewable (selection behavior is visible in logs, with `playwright-report` as traceable output).

That is the real signal here: Phase 2 is not just “green in CI”, but **usable in day-to-day engineering** because it is fast, understandable, and operationally reliable.

### A Quick Detour: The Page Object Layer

Dynamic selection is the star of this post, but none of this matters if your tests are flaky. Selective execution requires a stable Page Object (PO) foundation:

{% highlight text %}
BasePage                     ← visit(url), waitForLoadState
  ├── UsersPage              ← open(), addUser(), selectUser(), ...
  └── ProductsPage           ← open(), placeOrder(), addProduct(), ...
TableComponent               ← generic table ops: getRows(), getCell()
{% endhighlight %}

We strictly use Playwright `Locator` APIs (`getByRole`, `getByTestId`) and avoid raw CSS selectors. Reusable components like `TableComponent` receive a root `Locator` for scoping, so multiple tables on the same page do not bleed into each other.

If your Page Objects are brittle, dynamic selection just means you fail faster.

### The Catch (Because There's Always One)

Phase 2 is a massive leap forward: maintainable YAML configs, exact TC ID matching, and an auditable TypeScript pipeline.

But it still has one fatal flaw: `mapping.yaml` is maintained by humans.

Every time an engineer adds a new page or a new test, they must remember to update YAML. If they forget, the new test becomes invisible to the dynamic selector. It either gets skipped on related PRs or forces a fallback to full regression.

Which leads to the final evolution.

In Phase 3, we stop relying on humans entirely. We remove manual YAML mapping and use AST (Abstract Syntax Tree) scanning to derive `frontend file -> spec file` relationships directly from code.

(More on Phase 3 in the next post.)
