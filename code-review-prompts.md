# AI-Agent Audit Prompt Library
### For auditing a baseline application: exception handling, optimization, test coverage, integration coverage, magic strings

A note on sourcing: this is synthesized from current developer/vendor blogs, GitHub/Microsoft/Google documentation, and established software-engineering references (Martin Fowler, JetBrains). Most of it is converging practitioner consensus as of mid-2026, not a formal standard — treat it as a strong starting point, not gospel, and adapt to your Java/Spring Boot/AISDLC context.

---

## 0. Before you run any of these: prime the agent

Every source on this converges on the same failure mode: an agent with no context gives generic, low-value answers ("model amnesia"). Before the seven prompts below, give the agent a short context block once per session:

```
Project context: [Java 17 / Spring Boot 3.x service, e.g. CROP Document
Generation Service]. Regulated financial services — assume PII/financial
data unless told otherwise. Coding standards: [Lombok conventions, your
copilot-instructions.md rules]. Scope for this session: [file/module/package].
```

This single block is what separates a useful review from a shallow one across every source reviewed — it's the same principle behind your `copilot-instructions.md` and declarative agents.

One more cross-cutting rule worth adopting: **run one dimension per prompt, not all seven at once.** Every source that compared results explicitly warned that "review everything" prompts produce shallow, diluted findings on each dimension. Run them as seven separate turns (or seven separate agent invocations), not one mega-prompt.

---

## 1. The seven prompts

Each now includes a worked example row (few-shot) and a one-line self-check before the agent finalizes output — the two gaps identified in review.

### Prompt 1 — Exception handling audit
```
Act as a senior [Java/Spring Boot] reviewer. Review [file/module] for
exception handling ONLY — ignore style, naming, and performance for this pass.

For every method that can fail, check:
1. Does it catch specific exception types, not generic Exception/Throwable?
2. Are any catch blocks empty or silently swallowing the error?
3. Are errors logged and/or correctly propagated to the caller, not lost?
4. Are resources (streams, connections) closed via try-with-resources or
   finally, even on the failure path?
5. Do critical operations (payments, data writes, external submissions)
   have rollback or compensation logic if they fail partway through?

Before finalizing, re-check your own list: for each row, confirm the
line number is real and the issue would actually compile/run as
described — don't include a finding you're not sure is accurate.

Output as a table: File | Line | Issue | Severity | Suggested fix.

Example row (format only — replace with your real findings):
| PaymentService.java | 142 | catch (Exception e) is too broad — swallows
unrelated runtime errors along with the intended IOException | High |
Catch IOException specifically; let unexpected exceptions propagate |
```

### Prompt 2 — Optimization pass with behavior locked
```
Act as a senior [Java/Spring Boot] engineer doing a NON-FUNCTIONAL refactor
pass on [file/module]. Do not change external behavior, method signatures,
or business logic. If you are not certain a change preserves behavior,
flag it instead of making it.

Identify:
1. Algorithmic complexity issues (O(n²) or worse in hot paths)
2. Duplicated logic that could be extracted
3. Unnecessary allocations, N+1 query patterns
4. Outdated / non-idiomatic patterns for [Spring Boot version]

For each finding: current behavior, proposed change, and a one-line
justification for why it's behavior-preserving. Add a column
"Behavior-preserving? Y/N + why."

Example row (format only — replace with your real findings):
| OrderLookupService.java:88 | Loops the full order list per item (O(n²))
to find matches | Replace with a HashMap keyed by order ID before the
loop | Y — same matches returned, only lookup strategy changes |
```

### Prompt 3 — Test coverage gap analysis
```
Act as a QA-minded senior engineer. Cross-reference [source files] against
their existing tests in [test files]. Do NOT write tests yet — this is a
gap report only.

Identify:
1. Methods/functions with no test coverage at all
2. Branches or conditions inside tested methods that aren't exercised
3. Missing edge cases: null/empty input, boundary values, invalid types
4. Error-path tests — are failure/exception scenarios tested, or only
   the happy path?

Before finalizing, re-check: confirm each "untested" claim by actually
looking for a matching test method name or assertion — don't flag
something as untested just because you didn't immediately spot the test.

Output as: Untested Areas | Missing Edge Cases | Under-tested Areas |
Suggested New Test Case (one line each).

Example row (format only — replace with your real findings):
| UserValidator.validateEmail() has no test file reference | Empty
string, string with no @, string over max length | N/A | Add
validateEmail_rejectsMissingAtSymbol_throwsValidationException |
```

### Prompt 4 — Integration test coverage check
```
Act as a senior engineer checking integration-test coverage on
[module/service]. First, state which definition of "integration test"
you're using for this review — e.g. tests that hit a real or realistic
double of an out-of-process dependency, vs. tests that only exercise an
in-process boundary — so we're aligned before you report anything.

Then check:
1. Which service-to-service or service-to-database calls have no test
   exercising the real (or realistic test-double) dependency?
2. Which API endpoints have no test verifying the full request→response
   path?
3. Which multi-step business workflows are only unit-tested piecemeal and
   never verified end-to-end?
4. Are there tests for failure modes of external dependencies — timeouts,
   5xx responses, DB unavailable?

Output as: Integration Point | Currently Tested? | Test Type Needed | Priority.

Example row (format only — replace with your real findings):
| DocumentGenerationService → OMOS submission API call | No — only mocked
in unit tests | Test with a real/staged OMOS endpoint or WireMock stub
verifying actual request/response contract | High |
```

### Prompt 5 — Magic strings / constants audit
```
Act as a senior [Java] reviewer focused ONLY on magic values — no other
review dimension this pass.

Scan [file/module] for:
1. String literals used as status/role/type identifiers (e.g. compared
   against "ACTIVE", "ADMIN", "PENDING") that appear more than once
2. Numeric literals with no named meaning embedded in business logic
3. Repeated string literals used as config keys, map keys, or event names

For each: suggest a named constant, enum, or config entry, and list every
file/line where the same literal recurs, so nothing is missed when it's
replaced. Do not change any logic — only the representation of these values.

Before finalizing, re-check: search again for each literal you're about
to report, to make sure you've found every recurrence, not just the
first one — a partial list is worse than none, since it leaves two names
for the same concept after a partial fix.

Output as: Literal | Occurrences (file:line list) | Suggested Constant/Enum.

Example row (format only — replace with your real findings):
| "PENDING" | OrderService.java:34, OrderService.java:112,
NotificationJob.java:9 | OrderStatus.PENDING enum value |
```

### Prompt 6 — Boilerplate code audit
```
Act as a senior [Java/Spring Boot] reviewer focused ONLY on boilerplate
code this pass.

Scan [file/module] for:
1. Manually written getters/setters/constructors/equals/hashCode/toString
   that a Lombok annotation (@Data, @Getter, @Builder, @AllArgsConstructor,
   etc.) could replace
2. Repeated null-check + log + rethrow patterns that could become a single
   @NonNull annotation or a shared utility
3. Repetitive DTO↔entity mapping code a mapper (MapStruct or a shared
   mapper class) could replace
4. Duplicated exception-wrapping / try-catch blocks across multiple
   methods that could be centralized (@ControllerAdvice or an AOP aspect)

For each finding: show current code, the replacement, and explicitly
confirm it won't change generated behavior — e.g. Lombok's @Data
generates equals/hashCode from ALL fields, which can silently change
behavior if the class had a hand-written equals with different field
inclusion. Output as: File | Line(s) | Boilerplate Type | Suggested
Replacement | Behavior Risk (Y/N + why).

Example row (format only — replace with your real findings):
| CustomerRecord.java:12-58 | Hand-written getters, setters, and toString
| Replace with @Data | Y — confirm no custom toString formatting or
excluded fields exist before swapping, since @Data includes all fields |
```

### Prompt 7 — Null-check / null-safety audit
```
Act as a senior Java reviewer focused ONLY on null-safety this pass.

Check [file/module] for:
1. Public method parameters and return values that can be null but
   aren't validated or documented as nullable
2. Optional used as a field type or method parameter — Optional is
   conventionally a return type only; flag other uses as anti-pattern
3. Chained calls (a.getB().getC().getD()) with no null-guard on any link
4. Repeated `if (x != null)` boilerplate that could become Optional
   chaining, @NonNull (Lombok), or a single boundary-level check instead
   of scattered internal re-checks
5. Whether nulls are validated at the boundary (input/API layer) rather
   than defensively re-checked at every internal call site

For each finding, show the concrete scenario that would trigger an NPE,
and the suggested fix.

Before finalizing, re-check each chained-call finding specifically —
confirm from the actual method signatures (not assumption) that each
link in the chain can genuinely return null; don't flag a chain as risky
if an earlier method is documented or typed to never return null.

Output as: File | Line | Risk Type | Suggested Fix.

Example row (format only — replace with your real findings):
| ReportBuilder.java:67 | Chained call | user.getAddress().getCity() —
getAddress() can return null for users with no address on file | Add a
null-guard or switch getAddress() to return Optional<Address> |
```

---

## 2. Why these are shaped this way

**Prompt 1 (exceptions).** The pattern of checking for generic-vs-specific exception types, empty/swallowed catch blocks, error propagation, and rollback on critical operations shows up consistently across a Microsoft .NET engineering blog on reviewing AI-generated code and a field-tested Claude Code prompt collection — both call out exception handling as the area AI-generated code most reliably gets wrong, since models tend to reach for generic exception types and can miss resource-cleanup edge cases.

**Prompt 2 (optimization, behavior-locked).** The "don't change behavior" constraint isn't optional framing — a widely cited prompt-engineering guide for developers specifically documents a case where an unscoped "refactor this" prompt made a structural change the developer hadn't asked for, and it only turned out fine by luck. An arXiv paper on multi-LLM code refinement pipelines formalizes the same idea: their "Refinement Agent" is explicitly instructed to optimize for a target metric *while preserving correctness*, and it verifies that with a correctness check before ever measuring performance. That two-step (verify behavior first, then optimize) is worth carrying into how you sequence Prompt 2 in your pipeline.

**Prompt 3 (test coverage).** The four-part structure (untested functions, uncovered branches, missing edge cases, missing error-path tests) is echoed almost verbatim across a GitLab engineering blog and a dedicated test-coverage prompt template. One QA-tooling vendor blog suggests AI-generated code specifically needs a higher coverage bar (they cite ~85–90% vs. ~70–80% for human-written code) because it tends to pass the happy path and fail on edge cases — flagging that this is one vendor's recommended benchmark, not an industry standard, so treat it as a discussion point for your team rather than a hard target.

**Prompt 4 (integration tests).** This is the one area where I'd genuinely caution you: "integration test" does not have a settled definition. Martin Fowler himself has written that the term is used inconsistently even among experienced engineers — some mean "any test with a real out-of-process dependency" (narrow), others mean "any broad, multi-component test" (broad), and a widely-shared engineering blog post found no two engineers on the same team agreed on a concrete definition. That's why Prompt 4 asks the agent to state its working definition before reporting findings — otherwise you'll get a report that silently uses a different scope than what you meant, and the coverage numbers won't mean what you think they mean. The four check areas (service/DB calls, endpoint request→response paths, end-to-end workflows, and external-dependency failure modes) come from an AI-generated-code audit checklist that specifically flags integration tests for API routes, DB writes, and end-to-end payment/login flows as the highest-value places to add coverage.

**Prompt 5 (magic strings/constants).** This is one of the more settled, non-controversial code smells in the literature — JetBrains' static-analysis guide, several independent code-smell catalogs, and a LinkedIn post from a well-known .NET educator all converge on the same fix (named constants or enums) and the same reasoning: magic strings fail silently on typos, get no IDE rename support, and resist centralized change. The "list every recurrence" instruction in the prompt matters in practice — a partial find-and-replace across a codebase is worse than not fixing it at all, since it leaves two names for the same concept.

**Prompt 6 (boilerplate).** Given you're on Spring Boot with Lombok already, the highest-value boilerplate checks are the ones several Lombok/Spring guides converge on: getters/setters/constructors that `@Data`/`@Builder`/`@AllArgsConstructor` can replace, and repetitive DTO-mapping code. I've deliberately left out the specific "X% reduction" statistics that show up in some of these guides — they're unsourced marketing figures from SEO-oriented blogs, not measurements I can verify, so I'm not passing them off as fact. The one caution worth keeping in the prompt is real, not marketing: Lombok's `@Data` generates `equals`/`hashCode` from every field by default, which is a documented source of subtle behavior changes if you retrofit it onto a class that had a hand-written `equals` — that's why the prompt asks the agent to confirm behavior parity before recommending the swap.

**Prompt 7 (null checks).** This is a genuinely contested area, worth knowing going in. Java's `NullPointerException` is a long-standing pain point, and `Optional` (Java 8) is the mainstream mitigation — but "use `Optional` as a return type, not as a field or parameter type" is Oracle's own stated design intent for the class, not just a style preference. At the same time, I found real pushback in the literature against reflexive "check null everywhere" defensive coding — one engineer's writeup argues scattering null-checks across every method just relocates the problem rather than fixing it, and favors validating once at the boundary instead. Prompt 7 leans toward that boundary-validation view (check 5), but this is a legitimate style debate on your team's part, not a settled fact — worth confirming it matches how your team already thinks about it before you run it at scale.

---

## 3. Other AI-SDLC feedback (kept brief, since you asked me to stay on topic)

A few small additions that would strengthen this specific prompt set, worth a look later rather than now:
- **Severity/priority column** on every prompt's output (a pattern several sources use) makes these gap reports directly triageable in a sprint, rather than just descriptive.
- Given you're building declarative agents already, the Google Antigravity codelab's pattern is worth stealing: have a coordinator agent write the audit *plan* to a file first and pause for your review before it runs any of the five checks — cheap human-in-the-loop gate before burning Copilot credits on a big scan.

---

## 4. Sources consulted

- [.NET Blog – Reviewing AI-Generated Code in .NET](https://devblogs.microsoft.com/dotnet/developer-and-ai-code-reviewer-reviewing-ai-generated-code-in-dotnet/) (Microsoft)
- [Apiyi – 25 Practical Prompts for Code Review with Claude Code](https://help.apiyi.com/en/claude-code-code-review-prompts-collection-guide-en.html)
- [Graphite – Effective prompt engineering for AI code reviews](https://graphite.com/guides/effective-prompt-engineering-ai-code-reviews)
- [DEV Community – Building Effective Prompts for AI Code Review](https://dev.to/learnairesource/building-effective-prompts-for-ai-code-review-what-actually-works-5hao)
- [Addy Osmani – The Prompt Engineering Playbook for Programmers](https://addyo.substack.com/p/the-prompt-engineering-playbook-for)
- [arXiv – Multi-LLM Orchestration for High-Quality Code Generation](https://arxiv.org/pdf/2510.01379)
- [GitLab Blog – 10 AI prompts to speed your team's software delivery](https://about.gitlab.com/blog/10-ai-prompts-to-speed-your-teams-software-delivery/)
- [ContextQA – How to Test AI Generated Code: A QA Checklist for 2026](https://contextqa.com/blog/what-is-ai-generated-code-testing-checklist/)
- [GitHub Docs – Increasing test coverage with GitHub Copilot](https://docs.github.com/en/copilot/tutorials/roll-out-at-scale/drive-downstream-impact/increase-test-coverage)
- [Martin Fowler – bliki: Test Pyramid](https://martinfowler.com/bliki/TestPyramid.html)
- [Jay Freestone – Defining 'integration' tests](https://www.jayfreestone.com/writing/integration-tests/)
- [Aatvi AI – AI-Generated Code Audit Checklist](https://aatvi.ai/en/insights/ai-generated-code-audit-checklist)
- [JetBrains Qodana – What is a Code Smell?](https://www.jetbrains.com/pages/static-code-analysis-guide/code-smells/)
- [CodeClarityLab – Magic Strings glossary](https://codeclaritylab.com/glossary/magic_strings)
- [DEV Community – Code Smell 249: Constants as Numbers](https://dev.to/mcsee/code-smell-249-constants-as-numbers-568)
- [Google Codelabs – Multi-Language Code Auditor with Antigravity](https://codelabs.developers.google.com/multi-language-code-auditor-antigravity)
- [GeeksforGeeks – Using Lombok to Reduce Boilerplate Code in Spring Boot](https://www.geeksforgeeks.org/advance-java/using-lombok-to-reduce-boilerplate-code-in-spring-boot/)
- [DZone – Reducing Boilerplate Code With Annotations](https://dzone.com/articles/removing-boilerplate-code-with-lombok)
- [Oracle – Tired of Null Pointer Exceptions? Consider Using Java SE 8's Optional](https://www.oracle.com/technical-resources/articles/java/java8-optional.html)
- [Medium – The Problem With Null Pointer (case against reflexive defensive null-checking)](https://medium.com/@guanqing/the-problem-with-null-pointer-7d9215bbceab)

*Verify each prompt against your own agent's actual behavior before relying on it — I haven't run these against your codebase, so treat the "why it works" section as grounded reasoning, not a guarantee of results for your specific stack.*
