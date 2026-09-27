# Module 10: Testing, Documentation & Agile Practices

---

## PART 1: Fundamentals (must know cold)

- **Unit test**: tests one function/class in isolation, with external dependencies mocked — fast, and pinpoints exactly what broke.
- **Integration test**: tests multiple components working together (e.g., a service actually hitting a test database) — slower, catches issues unit tests can't (like a wrong SQL query or a misconfigured connection).
- **Mocking**: replacing a real dependency (an external API, a database) with a fake that returns pre-programmed responses, so a unit test doesn't depend on that dependency being available/fast/deterministic.
- **Test coverage**: the percentage of code exercised by tests — a useful signal, but a target to chase blindly (100% coverage doesn't mean zero bugs; it just means every line executed at least once).
- **Regression test**: a test written specifically to reproduce a bug that was just fixed, so the pipeline blocks that exact bug from silently coming back.
- **Contract testing**: verifying that a service's API actually matches what its consumers expect, without needing a full end-to-end environment — catches breaking changes between services early.
- **Agile/Scrum cadence**: sprints (fixed-length iterations), daily standups, sprint planning, retrospectives — the goal is short feedback loops and frequent, incremental delivery rather than one large release.
- **Documentation as a habit, not an afterthought**: what changed and why, not just what the code does — the "what" is visible in the code; the "why" and "what changed" are what get lost without documentation.

---

## PART 2: Interview Questions & Answers

**Q1: What's the difference between a unit test and an integration test, and when do you write each?**
> A unit test isolates one function or class, mocking out its dependencies, so it's fast and tells you exactly what broke if it fails. An integration test exercises multiple real components together — for example, a service actually querying a real (test) database — which is slower but catches issues a unit test's mocks would hide, like a query that's syntactically fine in isolation but wrong against the real schema. I write unit tests for business logic with many edge cases (fast feedback matters most there) and integration tests for the seams between components — API endpoints, database access layers — where the actual wiring is what needs verifying.

**Q2: How do you decide what to mock in a unit test?**
> I mock anything that's slow, non-deterministic, or external to the thing actually being tested — a third-party API call, a database, the system clock for time-dependent logic. I don't mock the actual logic under test itself, or the test becomes meaningless (testing that the mock returns what I told it to return). If a function is hard to unit test without mocking half of it, that's often a signal the function is doing too much and should be split.

**Q3: How would you add a regression test after fixing a production bug?**
> I'd first write a test that reproduces the exact bug — feeding in the specific input that caused the wrong behavior and asserting the correct expected output — and confirm it fails against the old (buggy) code. Then I apply the fix and confirm the same test now passes. That test stays in the suite permanently, so if a future change accidentally reintroduces the same bug, the pipeline catches it before it reaches production instead of a customer finding it again.

**Q4: How do you use Postman for more than manual API testing?**
> Beyond manually poking an endpoint during development, I'd build out a Postman collection covering the main request/response scenarios for an API — including error cases — and use it as living documentation the whole team can reference. Running that collection via Newman in the CI pipeline turns it into an automated contract check, catching a breaking change to the API's shape before it ships, not just when someone happens to test manually.

**Q5: What does good documentation actually include, beyond describing what the code does?**
> The code itself already shows *what* it does if someone reads it — good documentation covers *why* a non-obvious decision was made (e.g., "we retry 3 times here because the third-party API has intermittent 2-second blips"), and a changelog of what changed and why for anything consumers depend on, like an API contract. For a cross-timezone or Agile team especially, that upfront context prevents a lot of back-and-forth that otherwise costs a full day waiting for the other timezone to wake up and explain a decision.

---

## PART 3: How It Works Internally

**How mocking actually works under the hood**: a mocking library replaces the real object/function at the point your code looks it up (via dependency injection, monkey-patching, or a test double passed in explicitly) with a fake object that records calls made to it and returns pre-configured responses. This is why dependency injection (Module 2) makes testing easier structurally — if a class receives its dependencies via constructor injection rather than creating them internally, a test can simply pass in a mock instead, with no special test-only code path needed inside the class itself.

**How code coverage tools measure "coverage" mechanically**: a coverage tool instruments your code (either at compile time or via a runtime hook) to record which lines/branches actually execute during a test run, then compares that against the total lines/branches in the codebase to produce a percentage. This is exactly why coverage is a weak proxy for correctness — a line "executing" during a test doesn't mean the test actually asserted anything meaningful about its behavior; a test with no assertions at all can still produce 100% coverage of the code it calls.

**How CI pipelines actually enforce these practices as gates**: each stage (lint, unit test, integration test, coverage threshold) runs as a distinct step; a non-zero exit code from any step fails the whole pipeline run, which is typically wired to block merging a PR into the main branch via a required-status-check setting on the repository. This is the mechanism that turns "we should write tests" from a hopeful team norm into something structurally enforced — a PR literally cannot merge if a required test fails, regardless of social pressure or deadline urgency.

**How Agile's short feedback loop actually reduces risk, mechanically (not just philosophically)**: by shipping in small, frequent increments (a sprint) rather than one large release, the amount of *new, unverified* code in front of real users at any given time stays small — so if something goes wrong, the space of recent changes to investigate is small too. A daily standup surfaces a blocker within at most 24 hours instead of it silently costing days before anyone notices; a retrospective is a structural checkpoint for the process itself to improve, not just the product — this is why "Agile" is really a risk-and-feedback-loop-management approach as much as a scheduling method.
