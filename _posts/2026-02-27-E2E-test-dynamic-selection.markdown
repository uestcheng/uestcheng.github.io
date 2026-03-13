layout: post
title: "E2e Test Dynamic Selection: Part 1: The Problem — Why \"Test Everything\" Doesn't Scale"
date: 2026-02-28 12:00:00 +0800
categories: [Testing]
tags: [e2e, testing, ci/cd]
---

## Part 1: The Problem — Why "Test Everything" Doesn't Scale

### 1.1 The CI/CD Bottleneck Nobody Wants to Talk About

Let's be honest about E2E test suites: they grow linearly with your product scope. What starts as a harmless 5-minute smoke test quietly bloats into a 30-minute regression gate. Usually, nobody pays attention until the entire engineering team is blocked, staring at a spinning CI pipeline.

It’s frustrating because we know a simple copy change in a login form shouldn't trigger a full checkout flow regression. But in most setups today, it absolutely does. Why? Because the CI pipeline is blind—it doesn't know the difference.

What we actually want is a surgical strike: *Change A → Test A*. Instead, we’re stuck with the sledgehammer approach: *Change A → Test Everything*.

### 1.2 Why This Is Harder Than It Looks

You might think, "Just run the tests related to the changed files." If only it were that simple. The relationship between a changed frontend file and the E2E spec that verifies it is notoriously opaque.

Think about it:
* If you change `LoginPage.vue`, you know you should run `login.spec.ts`. But what about `onboarding.spec.ts`? That flow probably starts with a login, too.
* What if you touch a shared `Button.vue` component? You might suddenly need to run dozens of seemingly unrelated specs.
* And if you refactor something like `api/user.ts`? Good luck. You basically need to run every spec that touches user state.

The mapping between source code and E2E tests is rarely 1:1, it’s not static, and it’s certainly not something you can easily guess just by looking at the file tree.
You might think a dependency graph tool (like `dependency-cruiser` or Nx's `affected`) could untangle this automatically. But here's the thing: in many real-world setups—including ours—the frontend application and the E2E test suite live in **separate repositories**. There is no shared import graph to traverse. The frontend code has no idea the test code exists, and vice versa. No matter how sophisticated your static analysis is, it cannot cross a repository boundary that has no code-level link.
### 1.3 The Real Cost of E2E: It's Not Just "Slow"

We all know E2E tests are slow, but the actual cost hits you in three dimensions: compute budget, engineering hours, and pipeline trust.

#### The Compute Bill (CI Minutes = Real Money)

Every minute your tests run costs money. Let's look at a standard Linux runner (like GitHub Actions at $0.008/min or GitLab CI at $0.005/min).

If you have a 30-minute full regression suite running on every single PR, the math gets ugly fast. A small team doing 10 PRs a day burns about $48 a month just on E2E compute. Scale that up to a 30-person team doing 60 PRs a day (and factor in the inevitable retries for flaky tests), and you’re easily looking at $576+ a month. And remember, that’s *just* for the E2E stage.

Now, ask yourself: if 80% of those E2E runs are exercising code paths completely untouched by the PR, aren't you just burning cash to buy confidence you already secured with your unit tests?

#### The Maintenance Tax and Flakiness

As Martin Fowler’s [Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html) points out, E2E tests are notoriously flaky. The more complex your UI—with browser quirks, network timing, and random animations—the more likely a test will fail for a reason completely unrelated to your code change.

Every random failure acts as a tax on your engineering team. Someone has to stop what they're doing to investigate if it's a real bug or a false positive. Often, the "fix" is just hitting the retry button, which doubles that CI cost we just talked about. Worse, when tests fail randomly enough, the team loses trust in the pipeline entirely. Red builds just become background noise.

#### The Pyramid Tells Us *What*, But Not *When*

The Test Pyramid principle is pretty straightforward: write lots of small, fast unit tests, and keep your slow, expensive E2E tests to a bare minimum. If a lower-level test confidently covers a condition, don't duplicate it at the E2E level. Ignoring this leads to the dreaded "Test Ice Cream Cone"—an inverted pyramid that guarantees slow feedback loops and massive maintenance headaches.

However, most CI pipelines today ignore the spirit of the pyramid. They might have fewer E2E tests, but they still run *all* of them on *every* PR.

The Test Pyramid answers the question: **"How many E2E tests should I write?"**
But it leaves a crucial gap that we need to fill: **"Given the E2E tests I have, which ones should I actually run for *this specific PR*?"**

Our goal isn't to run all of them, nor is it to run none of them. We just want to run the *right* ones.

---

## Part 2: Prior Art — What's Been Tried (and Where It Breaks)

### The Static Scanning Approach (Lessons from GitLab)

We aren't the first to try solving this. Back at GitLab, we tackled this by statically scanning the codebase for test identifiers (like `data-testid`). The idea was to reverse-map changed selectors directly to the specs that used them.

It was a clever approach, and it worked decently for atomic components. It was cheap to run because there was no runtime overhead.

But it hit a hard ceiling. Static scanning is incredibly brittle if you use dynamic or computed IDs. More importantly, it completely misses *logical* connections. If you change a Page Object that affects a multi-step user journey across different pages, a simple selector scan won't catch it. It tells you *what element* changed, but it has no idea *which user flow* is actually impacted.

Expensive automation requires precision. And precision requires understanding the semantic relationship between your frontend code and your test code—something file paths and regex scans simply can't grasp.

---

## Part 3: The Insight — Moving from Implicit to Explicit

We realized that trying to magically *infer* the link between source code and test code is a losing battle. Even the most advanced dependency analysis tools assume the code lives in a single, connected graph. But when your frontend app and your E2E suite are in separate repos, that graph simply doesn't exist. Every approach—whether it's regex scanning, AST parsing, or import graph traversal—eventually hits the same wall: someone, somewhere, has to explicitly say "this frontend file is tested by that spec."

Once we accepted that some level of manual declaration is unavoidable, the question shifted from *"how do we eliminate human input?"* to *"how do we make human input as cheap and hard to mess up as possible?"*

So, we stopped guessing.

**The core principle we landed on:** If the machine can't reliably figure out the mapping, we need to make that mapping a first-class, maintained artifact in our codebase. It has to be cheap to write, easy to validate, and impossible to silently drift out of sync.

In the next part, I'll show you exactly how we evolved from brittle shell-script hacks into a config-driven, fixture-aware engine that dynamically selects the right E2E specs for every PR—and how we minimized human intervention at every step of the way.