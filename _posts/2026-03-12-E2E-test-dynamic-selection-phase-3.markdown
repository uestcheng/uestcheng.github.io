---
layout: post
title: "E2E Test Dynamic Selection Phase 3 – Zero-Config Auto Mapping"
date: 2026-03-11
categories: [Testing]
tags: [e2e, testing, ci/cd, playwright]
---

In Part 2, we built what seemed like a solid dynamic test selection pipeline. We used `git diff` to feed a `mapping.yaml`, which then filtered down a `--test-list` for Playwright. It worked beautifully. We were ready to ship it.

Except for one glaring problem: **`mapping.yaml` is basically a lie waiting to happen.**

It’s a manually maintained file that *claims* to know exactly which frontend file affects which E2E test. But we all know how this story ends. The moment a developer adds a new page, refactors a component, or just renames a spec file without remembering to update that YAML, the mapping silently drifts. Suddenly, your pipeline stops selecting the right tests, and you don't even know it's broken until a bug hits production.

In this post, we are taking a sledgehammer to that YAML file. Phase 3 eliminates manual configuration entirely by introducing a static scanner. Instead of relying on humans to remember things, we just read the real source of truth: the Page Object declarations that are already sitting right there in our QA codebase.

---

## The Mapping Problem (and the "Aha!" Moment)

Let's look at what Phase 2's `mapping.yaml` looked like:

{% highlight yaml %}
rules:
  - pattern: "^src/pages/UsersPage\\.jsx$"
    specs:
      - specs/example.spec.ts
  - pattern: "^src/pages/ProductsPage\\.jsx$"
    specs:
      - specs/example.spec.ts
{% endhighlight %}

Every time a new page, component, or spec was introduced, someone had to touch this file. For a proof-of-concept with two pages, sure, it’s fine. But scale this to a real project with dozens of routes and hundreds of specs, and it becomes an unmaintainable ticking time bomb.

Then came the insight: **The QA codebase already knows this information.** If you have a well-structured Page Object (PO) model, the links between your frontend files and your tests are already explicitly written in your TypeScript code. We don't need to duplicate that knowledge into a YAML file; we just need to write a script that can read it.

---

## The Declaration Chain (Connecting the Dots)

Phase 3 relies on tracing a four-link chain that naturally exists in our QA codebase. Here is how we follow the breadcrumbs:

**1. Page Object → Frontend File**

Every good Page Object should know what page it represents. We enforce this by passing the `pageName` directly in the `super()` call:

{% highlight typescript %}
// src/page/users.page.ts
export class UsersPage extends BasePage {
  constructor(page: Page) {
    super(page, 'src/pages/UsersPage.jsx')  // ← Here is our first anchor!
    // ...
  }
}
{% endhighlight %}

**2. Fixture → PO Class**

Next, Playwright’s custom test fixtures wire those PO classes to specific property names:

{% highlight typescript %}
// src/fixtures/test.fixture.ts
export const test = base.extend<AppFixtures>({
  usersPage: async ({ page }, use) => {
    const usersPage = new UsersPage(page)
    await use(usersPage)
  },
  productsPage: async ({ page }, use) => {
    const productsPage = new ProductsPage(page)
    await use(productsPage)
  },
})
{% endhighlight %}

**3. Spec → Fixture Props**

In the actual tests, developers only destructure the specific fixtures they need. This is the crucial link:

{% highlight typescript %}
// specs/example.users-flow.spec.ts
test('create user only', async ({ page, usersPage }) => {
  // If you only ask for `usersPage`, we know this test only cares about `UsersPage.jsx`.
})
{% endhighlight %}

**4. Component → PO Import**

What about shared components? Same logic. Component-level Page Objects declare their `componentName` and are imported by the parent page POs:

{% highlight typescript %}
// src/component/table.component.ts
export class UsersTableComponent extends BaseTableComponent {
  constructor(root: Locator, componentName = 'src/components/UsersTable.jsx') {
    super(root, componentName)
  }
}
{% endhighlight %}

{% highlight typescript %}
// src/page/users.page.ts
import { UsersTableComponent } from '../component/table.component.js'

export class UsersPage extends BasePage {
  readonly userTable: UsersTableComponent
  // ...
}
{% endhighlight %}

**The complete trace:** If someone modifies `UsersTable.jsx`, our scanner sees that `UsersTableComponent` targets it. It sees `UsersPage` imports that component. It sees the `usersPage` fixture instantiates that page. And finally, it sees which specs destructure `usersPage`. Boom. We know exactly which tests to run.

---

## Building the Scanner (`generate-mapping.ts`)

Instead of maintaining a YAML file, we now run one script before the tests start that generates the mapping on the fly:

{% highlight typescript %}
function main(): void {
  const classToFrontendPath = scanPageClassToFrontendPath()
  const fixturePropToClassName = scanFixturePropToClassName()
  const specToFixtureProps = scanSpecToFixtureProps()
  const poClassToImports = scanPoClassToImportedNames()
  const frontendComponents = scanFrontendComponents()
  
  const mapping = buildMappingConfig(
    classToFrontendPath,
    fixturePropToClassName,
    specToFixtureProps,
    poClassToImports,
    frontendComponents,
  )

  mkdirSync(dirname(outputPath), { recursive: true })
  writeFileSync(outputPath, `${JSON.stringify(mapping, null, 2)}\n`, 'utf-8')
}
{% endhighlight %}

Those five `scan*` functions just use simple regular expressions against the TypeScript AST/source code. 

They output a temporary, ephemeral `.tmp/mapping.json` that looks exactly like the rules we used to write by hand. Four perfectly derived rules, completely auto-generated. Zero human maintenance required.

---

## Killing the Test IDs and `--grep`

In Phase 2, we heavily relied on embedding weird `TC_ID` strings into our test titles and matching them with `--grep`:

{% highlight typescript %}
// Phase 2 (R.I.P.)
test('[TC_USERS_001] create user only', ...)
{% endhighlight %}

Phase 3 kills this anti-pattern entirely. Tests go back to having clean, readable titles. Selection is now handled purely through fixture dependencies. If your spec destructures `usersPage`, it's implicitly tied to `UsersPage.jsx`. No grep, no IDs, no arbitrary naming conventions. The dependency is declared by the code itself.

---

## Spec Path Matching and The `›` Separator Trap

With Playwright's `--list` output, we needed a reliable way to filter tests based on the file paths our scanner found. We opted for spec path prefix matching:

{% highlight typescript %}
function matchSpecPathPrefix(line: string, specPath: string): boolean {
  const normalizedSpecPath = normalizeSpecPath(specPath)
  const filename = basename(normalizedSpecPath)

  return line.startsWith(`${normalizedSpecPath} ›`) ||
         line.startsWith(`${filename} ›`)
}
{% endhighlight %}

Sounds simple, right? Except it wasn't. 

My first pass at parsing Playwright's test list output used this regex:

{% highlight typescript %}
// Bug: This splits on the > character too!
const segments = withoutProject.split(/\s*[›>]\s*/)
{% endhighlight %}

Guess what happened? Someone named a test `users->products order flow`. The regex saw the `>` character, split the title in half, and completely broke the matching logic. 

**The fix:** Only ever split on the specific Playwright delimiter `›` (U+203A), never the standard `>`:

{% highlight typescript %}
const segments = withoutProject.split(/\s*›\s*/)
{% endhighlight %}

One single character difference cost me three hours of debugging. Lesson learned: Test names are user input. Never assume they won't contain your regex delimiters.

---

## Smarter Triggers: The Trivial Change Filter

We quickly realized that not every `git diff` should actually trigger a test run. If a developer fixes a typo in a comment inside `UsersPage.jsx`, the behavior hasn't changed:

{% highlight diff %}
- // fetch users on mount
+ // fetch users on component mount
{% endhighlight %}

To prevent wasting CI minutes on comment tweaks, Phase 3 introduces a trivial change filter. We run `git diff -U0` and inspect every added or removed line:

{% highlight typescript %}
export function isTrivialLine(line: string): boolean {
  const trimmed = line.trim()
  if (trimmed === '') return true
  if (trimmed.startsWith('//')) return true
  if (/^(\/\*|\*\/|\*)/.test(trimmed)) return true
  if (/^\{\s*\/\*.*\*\/\s*\}$/.test(trimmed)) return true
  if (trimmed.startsWith('')) return true
  return false
}
{% endhighlight %}

If *every* line in a file's diff is deemed trivial, the file is entirely excluded from the mapping. The dry-run output proudly marks it:

```text
│ Changed files:
│   src/pages/UsersPage.jsx  (trivial – skipped)
```

---

## A Note on File-Level Segregation

During early testing, I had a single massive file called example.flow.spec.ts that destructured both usersPage and productsPage:

{% highlight typescript %}
// example.flow.spec.ts (old, merged)
test('create user only', async ({ page, usersPage }) => { ... })
test('place order', async ({ page, productsPage }) => { ... })
{% endhighlight %}

The problem: Our scanner resolves dependencies at the file level. Because both fixture props appeared anywhere in that file, a change to UsersPage.jsx would trigger both tests—even the one that only tested products.

The fix: We split the specs into example.users-flow.spec.ts and example.products-flow.spec.ts. Keep your specs focused on the domain they actually test. If a test legitimately crosses boundaries (like an end-to-end checkout flow), it correctly gets selected if either page changes.

This isn't really a limitation; it's a design constraint that encourages better test organization.

---

## The New Pipeline at a Glance

With Phase 3 complete, our pipeline looks like this:
{% highlight plaintext %}

generate-mapping.ts  →  .tmp/mapping.json
                              ↓
resolve-tests.ts  ←  git diff + suites.yaml
       ↓
  filter trivial changes
       ↓
  match mapped specs + suite specs
       ↓
  playwright --list  →  path prefix match
       ↓
  .tmp/dynamic-test-list.txt
       ↓
  playwright --test-list

{% endhighlight %}

Two scripts. Zero configuration files to maintain. The mapping logic is generated fresh, accurately, and automatically on every single PR run.

---

## What's Next? (Phase 4)

We've made the mapping self-maintaining. The next logical step is making it self-validating.

In Phase 4, we'll tackle production hardening:

- **PR Full Diff Coverage**: Switching the diff range to `origin/main...HEAD` to cover all commits in a PR, not just the latest push.
- **TestID Contract Validation**: Statically scanning PO `getByTestId()` calls against actual frontend `data-testid` attributes, allowing us to fail fast on missing IDs before tests even boot up.
- **pageName Linting**: Enforcing that every PO strictly declares a `pageName` so we never have silent gaps in our auto-mapping.

Stay tuned.