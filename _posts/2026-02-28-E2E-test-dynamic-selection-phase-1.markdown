---
layout: post
title: "E2e Test Dynamic Selection: Phase 1 - The Shell Script"
date: 2026-02-28 12:00:00 +0800
categories: [Testing]
tags: [e2e, testing, ci/cd]
---

## Phase 1: The Shell Script (aka Duct Tape That Actually Works)

> **Note**: This is Part 2 of the series "E2E Test Dynamic Selection".  
> You can find the introduction and background in the [previous post](/posts/E2E-test-dynamic-selection).

Look, every good system starts out as an ugly hack. For us, that hack was a Bash script.

### The Idea

If you boil down the "Change A → Test A" concept, it really just takes four brute-force steps:

1. Run `git diff` against the merge base to see what files changed.
2. Map those files to specific **tags** (hardcoded right in the shell script).
3. Mash those tags together into a regex pattern.
4. Shove that pattern into `npx playwright test --grep "<pattern>"`.

No fancy AST parsing, no complex config file. Just string manipulation in Bash, piggybacking off a feature Playwright already has.

### The Repository Structure

To show how this works, I set up two repos: a standard frontend app, and a QA suite pulled in as a git submodule. 

- **[`sample-app`](https://github.com/uestcheng/sample-app)** — a Vite + React frontend
- **[`auto-select-poc`](https://github.com/uestcheng/auto-select-poc)** — the Playwright E2E suite, mounted at `qa/`

I chose a git submodule for POC simplicity, but nothing in the approach depends on it. The only real prerequisite is that the E2E environment can `git diff` the frontend code. A monorepo would make this even more straightforward; a separate repo with CI checkout access works just as well.

The directory structure looks like this:

{% highlight text %}
sample-app/                    ← Frontend repo
├── .github/workflows/
│   └── e2e-playwright.yml     ← CI pipeline
├── src/
│   └── pages/
│       ├── UsersPage.jsx
│       └── ProductsPage.jsx
├── qa/                        ← Git submodule → auto-select-poc
│   ├── scripts/
│   │   └── run-dynamic-selection.sh  ← Phase 1 entry point
│   ├── specs/
│   │   ├── example.spec.ts           ← smoke tests (@smoke)
│   │   └── example.flow.spec.ts      ← flow tests (@flow)
│   ├── src/
│   │   ├── page/                     ← Page Objects
│   │   └── component/                ← Reusable component POs
│   └── playwright.config.ts
├── package.json
└── vite.config.js
{% endhighlight %}

The shell script lives inside `qa/scripts/`, but it gets triggered by the frontend's CI. It basically peeks up a directory, runs `git diff`, and decides what tests to run.

### Running Phase 1 Locally

If you want to poke around Phase 1 locally, just check out the matching tags. The submodule makes this pretty painless:

{% highlight bash %}
# Switch to Phase 1
cd sample-app
git checkout phase-1-frontend
git submodule update --init    # This automatically aligns qa/ to the Phase 1 commit

# Start the frontend app
npm install && npm run dev

# Install QA dependencies
cd qa
npm install && npx playwright install

# Run the dynamic selection script
bash scripts/run-dynamic-selection.sh
{% endhighlight %}

(For the record, Phase 1 uses tags `phase-1-frontend` and `phase-1-qa`. We'll move to `phase-2-*` in the next section).

### How We Abused Playwright Tags

The whole script relies on Playwright's built-in `--grep` flag. It just does a regex match against test titles. So, if you jam some `@` tags into your test names, you get a filtering system for free:

{% highlight typescript %}
test('should display user list @UsersPage @smoke', async ({ page }) => {
  // ...
});

test('should complete checkout flow @ProductsPage @flow', async ({ page }) => {
  // ...
});
{% endhighlight %}

Run `npx playwright test --grep "@UsersPage"` and you only hit the first test. Quick and dirty.

### The Mapping: A Giant `case` Statement

The "brain" of this phase is just a massive file-to-tag mapping inside `run-dynamic-selection.sh`:

{% highlight bash %}
case "$file" in
  src/pages/UsersPage.*)
    add_tag "@UsersPage"
    ;;
  src/pages/ProductsPage.*)
    add_tag "@ProductsPage"
    ;;
  src/components/*)
    add_tag "@UsersPage"
    add_tag "@ProductsPage"
    ;;
esac
{% endhighlight %}

It loops through the changed files, collects the tags, and builds a grep string like `@smoke|@UsersPage|@ProductsPage`. Notice that `@smoke` is always appended—you never want to skip the critical path, regardless of what changed.

### Proof: A Real CI Run

Does it actually work? Yep. On commit `88317b7`, we tweaked `src/pages/ProductsPage.jsx`. If you check the GitHub Actions [log](https://github.com/uestcheng/sample-app/actions/runs/22258647436/job/64393395851#step:10:34), here’s what the script figured out:

{% highlight text %}
Changed files (from ..):
  .github/workflows/e2e-playwright.yml
  node_modules/.package-lock.json
  ...
  src/pages/ProductsPage.jsx

Selected tags: @smoke @ProductsPage
Grep pattern: @smoke|@ProductsPage

Running 3 tests using 1 worker
{% endhighlight %}

Result: Instead of running the whole suite, it grabbed 3 tests, spun up 1 worker, and finished in 45 seconds. Huge win.

### The Catch: Why You Can't Stop Here

Phase 1 proved the core idea is highly valuable: running selective tests based on git diff saves a ton of CI time. But as a long-term solution, it’s a maintenance nightmare waiting to happen:

1.  **Hardcoded mappings suck.** Rename a route? Add a new page? You have to remember to update the Bash script. Spoiler: Nobody remembers.
2.  **The granularity is too coarse.** Touch the "delete user" button, and suddenly all 40 `UsersPage` tests run.
3.  **Test titles become unreadable.** You end up with monstrosities like `'should delete user @UsersPage @AdminFlow @smoke @regression'`—it's a search index pretending to be a sentence.
4.  **Grep is dumb.** If someone names a test `"should not show @smoke warning banner"`, congrats, it's now accidentally part of your smoke suite.
5.  **Shared components nuke the filtering.** Notice how `src/components/*` triggers everything in our script? Touch one shared button, and you're basically running a full regression anyway.

So, the duct tape holds, but it’s not exactly "production-grade". We needed a way to put the mapping closer to the actual test code, keep the test titles clean, and handle shared dependencies without blowing up the CI time.

That’s exactly what Phase 2 is built to fix.
