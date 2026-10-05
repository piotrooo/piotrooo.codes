---
title: 'What If Your if Statement Could Understand Context? Jev + Spring AI'
date: 2026-10-05
draft: false
url: 'what-if-your-if-statement-could-understand-context-jev-spring-ai'
---

AI models are good at answering questions.

Applications usually need something more specific.

A value. A choice. A score. Something that normal code can use.

That is what made me curious about [Jev](https://docs.typesafe.ai/).

Jev is not a chat model. You give it some state, ask typed questions about that state, and get structured answers back.

I wanted to understand where that fits in a real Java application, so I built a small dependency-review tool with Spring AI and Jev.

{{< figure src="/images/2026/10/05/1-hero.png" title="Figure 1. Hero" >}}

## Watch the video

I also recorded the complete walkthrough, including the real pull request and the CLI output.

<!-- Replace YOUTUBE_VIDEO_ID before publishing -->
{{< youtube YOUTUBE_VIDEO_ID >}}

[Watch it on YouTube](https://youtu.be/YOUTUBE_VIDEO_ID)

## The example

I used a real Dependabot pull request from Asterisk Java:

[Asterisk Java PR #786](https://github.com/asterisk-java/asterisk-java/pull/786)

The pull request updates Guava:

```text
com.google.guava:guava
33.5.0-jre → 33.6.0-jre
```

That already gives me useful information.

I know which dependency changed.

I have the upstream release notes.

But I still do not know what the change means for this application.

For example:

- Does this update probably need changes in the application?
- Could runtime behavior change?
- How important is Guava in this project?
- Is that usage covered by tests?

The release notes describe Guava.

They do not tell me how this repository uses Guava.

That is the gap I wanted to explore.

## The flow

The tool is intentionally small.

{{< figure src="/images/2026/10/05/2-flow.png" title="Figure 2. Flow" >}}

The flow is:

```text
GitHub PR
   ↓
Spring AI
   ↓
Evidence
   ↓
Jev
   ↓
Java
```

Git gives me the change.

Spring AI collects useful evidence.

Jev evaluates that evidence.

Java stays in control.

## Running the tool

The CLI accepts a GitHub pull request directly:

```bash
java -jar build/libs/dependency-review-0.1.0-SNAPSHOT.jar analyze \
  --github-pr https://github.com/asterisk-java/asterisk-java/pull/786
```

For this pull request, the tool detected:

```text
com.google.guava:guava
33.5.0-jre → 33.6.0-jre
```

Then it collected repository evidence, upstream release notes, unknowns, and the Jev evaluation.

The interesting part is not just the final result.

It is how the responsibilities are split.

## Spring AI investigates

The first part is `DependencyInvestigator`.

I do not ask Spring AI one big question like:

> Is this dependency update safe?

Instead, I give it a smaller job.

Find evidence.

The prompt includes rules like these:

```java
- Use ONLY the repository evidence supplied below.
- Do not invent release notes, CVEs, upstream API changes, or library behavior.
- Do not decide whether the change should be merged.
- Separate production usage, configuration usage, and relevant tests.
- If the evidence is insufficient, put that fact in `unknowns` instead of guessing.
```

This matters because I want Spring AI to investigate, not own the decision.

The result is returned as a structured `DependencyEvidence` object:

```java
DependencyEvidence evidence = chatClient.prompt()
        .user(prompt)
        .call()
        .entity(DependencyEvidence.class, EntityParamSpec::validateSchema);
```

Then I verify the evidence:

```java
return evidenceVerifier.verify(evidence, snippets, releaseNotes);
```

The verifier checks that evidence points back to the snippets and release notes that were actually supplied.

So the role of Spring AI is narrow:

```text
repository + release notes
        ↓
Spring AI
        ↓
structured evidence
```

## Repository evidence and upstream evidence are different

The tool keeps two types of evidence separate.

Repository evidence tells me how the application uses the dependency.

Upstream release notes tell me what changed in the dependency itself.

For this run, repository evidence only found the Guava dependency declaration in `pom.xml`.

The release notes, on the other hand, contain changes such as:

- moving some classes from `finalize()` to `PhantomReference`;
- deprecating `CacheBuilder` APIs using `TimeUnit` in favor of `Duration`;
- deserialization changes;
- additions and changes in graph-related APIs.

Those are real upstream changes.

But they do not automatically mean this application is affected.

I still need local evidence.

## Unknowns are part of the result

This is one of my favorite parts of the tool.

For this run it reported:

```text
❓ Unknowns
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Observation                                                                                            │
├────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Specific production code references to Guava classes or methods are absent from the provided snippets. │
│ Specific test code references to Guava classes or methods are absent from the provided snippets.       │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

The wording is important.

It does not say:

> This project does not use Guava.

It says the collected evidence did not contain clear production or test references.

That is a much safer statement.

In AI-assisted workflows, "I do not know" is useful information.

It gives normal code a reason to stop, ask for review, or collect more context.

## Jev evaluates the evidence

After the evidence is collected, I build one state:

```java
Map<String, Object> state = new LinkedHashMap<>();
state.put("dependency", update.coordinate());
state.put("previousVersion", update.fromVersion());
state.put("newVersion", update.toVersion());
state.put("repositoryEvidence", evidence);
```

Then I use `systemOne(...)` and ask several questions about the same state.

```java
SystemOneResponse response = typeSafeClient.systemOne(
        state,
        Map.of(
                "migration_required", Noul.of("""
                        Based on the repository and upstream release-note evidence, does this dependency update likely require
                        changes to the application's existing source code or configuration?
                        """),
                "runtime_behavior_change", Noul.of("""
                        Based on the repository and upstream release-note evidence, could this dependency update materially affect
                        runtime behavior that this application relies on even if the project still compiles?
                        """),
                "usage_impact", Score.of(
                        "How important is the observed dependency usage to this application's runtime behavior?",
                        "Not used by production code",
                        "Used in an isolated non-critical path",
                        "Used by multiple application components",
                        "Used in an important application path",
                        "Widely used or infrastructure-critical"),
                "test_coverage", Score.of(
                        "How well is the observed application usage of this dependency protected by relevant tests?",
                        "No relevant tests found",
                        "Only indirect test coverage",
                        "Partial behavior coverage",
                        "Strong coverage of relevant behavior",
                        "Dedicated compatibility coverage"),
                "primary_affected_area", Choice.builder()
                        .instructions("Which area is most directly affected by the observed dependency usage?")
                        .option("BUILD", "Build tooling or compile-time integration")
                        .option("API_USAGE", "Direct calls to library APIs")
                        .option("CONFIGURATION", "Properties, beans, or framework configuration")
                        .option("RUNTIME", "Runtime infrastructure or application behavior")
                        .option("SECURITY", "Authentication, authorization, cryptography, or security controls")
                        .option("DATA", "Serialization, persistence, schemas, or data formats")
                        .build()));
```

This is the part I find most interesting.

I am not asking Jev:

> Is this update safe?

I am asking smaller questions:

- Is migration probably required?
- Could runtime behavior change?
- How important is the observed usage?
- How well is it tested?
- Which area is affected most?

Each question has one job.

## Noul, Choice, and Score

Jev uses three question shapes: `Noul`, `Choice`, and `Score`.

### Noul

`Noul` is for yes-or-no questions.

The result is a value between `0` and `1`.

For example:

```java
"migration_required", Noul.of("""
        Does this dependency update likely require changes
        to the application's existing source code or configuration?
        """)
```

A value close to `1` leans toward yes.

A value close to `0` leans toward no.

A value around `0.5` means the answer is unclear.

`Noul` does not need a separate confidence value because the result already shows the uncertainty.

### Choice

`Choice` selects one value from a known set.

For this tool:

```text
BUILD
API_USAGE
CONFIGURATION
RUNTIME
SECURITY
DATA
```

The answer also includes confidence.

### Score

`Score` puts the answer on an ordered scale.

For usage impact, my scale is:

```text
0  Not used by production code
1  Used in an isolated non-critical path
2  Used by multiple application components
3  Used in an important application path
4  Widely used or infrastructure-critical
```

The scale is defined before the model evaluates the state.

That is important.

My application decides what the levels mean.

## The real result

For Asterisk Java PR #786, the Jev result was:

```text
📊 Jev evaluation
┌─────────────────────────┬──────────┬────────────┬─────────────────────────────┐
│ Assessment              │ Score    │ Confidence │ Details                     │
├─────────────────────────┼──────────┼────────────┼─────────────────────────────┤
│ Migration required      │ 0.14     │ —          │                             │
│ Runtime behavior change │ 0.22     │ —          │                             │
│ Usage impact            │ 0.04 / 4 │ 0.97       │ Not used by production code │
│ Relevant test coverage  │ 0.00 / 4 │ 1.00       │ No relevant tests found     │
│ Primary affected area   │ BUILD    │ 0.74       │                             │
└─────────────────────────┴──────────┴────────────┴─────────────────────────────┘
```

These are not final approve-or-reject decisions.

They are separate judgments.

### Migration required: 0.14

Based on the supplied evidence, Jev sees little support for the idea that this update requires a migration.

That does not prove migration is impossible.

It only describes the current evidence.

### Runtime behavior change: 0.22

This is also low.

Again, it is based on the evidence supplied to Jev.

### Usage impact: 0.04 / 4

The closest label is:

```text
Not used by production code
```

Confidence is `0.97`.

The safe interpretation is not:

> Asterisk Java does not use Guava.

The safe interpretation is:

> The collected evidence did not show direct production usage, and Jev strongly placed that evidence near the bottom of the defined usage scale.

### Relevant test coverage: 0.00 / 4

The closest label is:

```text
No relevant tests found
```

Confidence is `1.00`.

Again, this describes the collected evidence.

It does not prove that the whole repository has no Guava-related tests.

### Primary affected area: BUILD

Confidence is `0.74`.

The strongest evidence in this run is the Maven dependency declaration, so `BUILD` is a reasonable result.

The lower confidence also tells me this answer is less clear than the usage and test scores.

## Confidence is a separate signal

For `Choice` and `Score`, Jev returns confidence.

The answer tells me what Jev selected.

Confidence tells me how clearly the available options separated for that input.

That is useful for application logic.

A future workflow could:

```text
high confidence
    → continue

low confidence
    → manual review

missing evidence
    → collect more context
```

The important part is that uncertainty stays visible.

The application does not have to treat every AI result as equally strong.

## Why separate questions matter

The useful output from this run is:

```text
Migration       0.14
Runtime         0.22
Usage           0.04 / 4
Tests           0.00 / 4
Area            BUILD
```

These answers describe different things.

That lets Java decide how each signal should be used.

For example, a future policy may treat runtime impact differently from test coverage.

Security updates may have their own rules.

Low confidence may require manual review.

The model evaluates the context.

The application still owns the workflow.

## An intelligent if

The mental model that helped me understand Jev is an intelligent `if`.

Normal Java code can easily do this:

```java
if (majorUpdate) {
    requireReview();
}
```

`majorUpdate` is a fact I already know.

The harder question looks more like this:

```java
if (migrationRequired > 0.7) {
    requireReview();
}
```

`migrationRequired` is not a value I can read directly from the version number.

It depends on context.

Jev evaluates that context.

Java still owns the `if`.

That is the boundary I want.

## Timing

I also measure the main stages.

For this run:

```text
⏱️  Timing
┌──────────────────────────┬──────────┐
│ Stage                    │ Duration │
├──────────────────────────┼──────────┤
│ Git + context collection │ 2.637 s  │
│ Spring AI investigation  │ 3.592 s  │
│ Jev evaluation           │ 0.443 s  │
│ Total                    │ 6.671 s  │
└──────────────────────────┴──────────┘
```

This is one run, not a benchmark.

It is still useful to see where the time goes.

## What this tool does not do

The current version is intentionally limited.

It does not:

- approve or reject the pull request;
- claim that missing evidence means something does not exist;
- let Spring AI invent upstream changes;
- reduce everything to one final decision.

Its current job is smaller:

1. detect the dependency update;
2. collect repository and upstream evidence;
3. expose unknowns;
4. ask Jev a few typed questions;
5. return the result to normal Java code.

That is enough for this experiment.

## What I would improve next

The next things I want to test are:

- better repository context collection;
- stronger support for indirect dependency usage;
- better links between release-note changes and local code;
- more real dependency updates;
- whether the same questions stay useful across different projects.

Only after that would I consider using some of these results in CI.

## Source code and links

The full working example should live in a public repository and match the code shown in this article.

<!-- Replace before publishing -->
- **Example project:** TODO_REPOSITORY_URL
- **Jev documentation:** https://docs.typesafe.ai/
- **Spring AI TypeSafe:** https://github.com/spring-ai-community/spring-ai-typesafe
- **Spring AI TypeSafe docs:** https://spring-ai-community.github.io/spring-ai-typesafe/latest/
- **Asterisk Java PR #786:** https://github.com/asterisk-java/asterisk-java/pull/786
- **Video:** https://youtu.be/YOUTUBE_VIDEO_ID

For me, the interesting part is not that AI can write another dependency review.

It is that I can split a fuzzy problem into small questions, get typed answers back, keep uncertainty visible, and let normal Java code decide what happens next.

Spring AI investigates.

Jev evaluates.

Java stays in control.
