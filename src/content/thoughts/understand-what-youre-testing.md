---
title: "Understand What You're Testing"
description: "On the different questions tests answer, the contract that holds them together, and why the laptop is a bad place to answer CI's questions"
pubDate: "August 11 2026"
heroImage: "/understand-what-youre-testing.jpg"
---

Software tests answer different questions, and treating them as one is where most of the pain comes from.

Some tests are narrow: given these inputs to this function, does it produce this output? Some are broader: does my service correctly use its database, its message broker, the other services it calls? Some are broader still: does the whole system work when wired together as it would be in production? Between and around these are plenty of others - contract tests, characterisation tests, property-based tests, and so on - each answering something slightly different again. The distinctions I want to draw here aren't a taxonomy; they're a rough map of the questions worth being honest about.

Because the pain most teams have with their test suite isn't that they don't test. It's that they haven't clearly decided what each test is *for*. What they end up with is one big bucket labelled "tests", a vague sense that more is better, and a slow, flaky suite that answers whichever question happens to be asked loudest.

When you don't distinguish the questions, you get tests that try to answer several at once. Tests that boot the entire system to verify a single business rule. Tests that mock so much of their surroundings that they no longer verify anything about the real code. End-to-end tests that run against a fake environment, which is neither end-to-end nor testing much of anything.

The fix isn't more tests. It's clarity about what each test is trying to prove, and picking the right shape for that job.

## Testing the Whole System

The most clearly-scoped question of the lot, and the one that gets abused least, is whether the whole system works when it's put together. That's end-to-end testing. Which isn't to say it's easy, or cheap, or fast - it's usually none of those things. But its purpose is straightforward: exercise your real deployed system to make sure the pieces are wired together correctly.

The rule for end-to-end tests is embarrassingly obvious to state and violated everywhere: test the real infrastructure against itself. Not a compose file. Not a scale-model of production running on your laptop. Not a "close enough" staging environment where half the services are stubbed and the auth provider is a mock. Actual staging, or a per-PR preview environment, running on the same kind of infrastructure your production runs on, talking to the same kind of dependencies over the same kind of networks.

The moment you start faking parts of the environment, you're not answering the end-to-end question any more. You're answering some other question. "Does my compose file boot?" is a valid question, but it isn't "does the system work in production?", and confusing the two costs you every time production surprises you.

End-to-end tests should be few in number, exercise the critical paths, and be treated as a last line of confidence rather than a primary test suite. If you're running hundreds of them, you're probably using them to answer questions that would be answered faster and more reliably at a lower level.

## The Importance of Fast Failures

Testing against a production-like (or production) environment carries inherent risk. Failures will be presented to a consumer - whether that's another developer using your test environment, or a real user in production. No amount of testing at lower levels catches everything; a real breakage will eventually surface in a live environment. This isn't something to be afraid of, but something to be well-prepared for. The ideal is that the breakage is presented to as few users as possible, caught by your observability platform, and automatically rolled back to the latest stable version.

The intricacies of this are worth their own blog post, but the fundamentals lie in blue/green or canary deployment, thorough analysis runs, and well-defined metrics in your observability platform to act on. There's no more realistic place to test production than production - but ensuring that any change has a small blast radius and can be rolled back safely and quickly is the only way to make that test safe.

## Testing Across Boundaries

A narrower question, but where most of the interesting work happens: does your service correctly use the things around it - its database, its message broker, the other services it calls - without needing the entire system booted to find out? This is what most people mean when they say "integration test", though the term is stretched enough that it's worth being specific about the question rather than the label.

That's the question [my previous post](https://bakes.software/thoughts/integration-testing-a-rough-guide/) was about, and I won't rehearse the whole thing here. The short version: use real instances of the stateful things you own, intercept HTTP calls to services you don't own at the request level, mock everything else at whatever boundary makes sense, and keep the whole lot running in-process. Fast, granular and focused.

## Contracts: The Seam that Makes it Work

None of this hangs together without contracts. This is the thing that makes the separation between questions possible, and its absence is why so many attempts at testing across boundaries collapse into either full-stack simulation or bare-bones mock objects that verify nothing.

A contract, in the sense I mean it here, is a specification of what one component promises to accept and what it promises to return. An OpenAPI spec is a contract. A protobuf definition is a contract. A well-defined function signature between two internal modules is, at a smaller scale, a contract. What makes it a contract rather than just documentation is that both sides can be checked against it programmatically. The caller can generate a typed client from it. The server can validate its own responses against it. Drift between the two becomes something a machine notices rather than something a customer discovers.

When you have that, the boundary between components becomes something you can test *through*. Your service calls another by way of a typed client generated from its contract. Your tests simulate the other service's responses by returning valid-according-to-the-contract data through that client. If the contract holds, and both sides honour it, then integration holds - and you don't need to run the other service to verify it.

When you don't have that, the boundary becomes something you have to simulate. And simulation is where things go wrong. You start building fakes that "behave like" the real thing. Then you build fakes of the fakes for tests. Then someone builds a mock container for a SaaS dependency whose entire value is being a managed external service, and you find yourself in the position of testing that your mocks agree with your other mocks. This isn't a strawman - it's what happens when the seam between components isn't a contract, because you have no principled way to say what a valid interaction looks like other than by imitating one that happened once.

Worth restating a point [from an earlier post](https://bakes.software/thoughts/schema-first-schema-last/): for contracts to actually do this work, both sides have to honour them. It isn't enough for the client to be typed. The server needs to validate that its own responses match its own schema, and treat a mismatch as its own bug rather than the client's problem. When servers don't do this, contracts become aspirational, and everything downstream - including any testing strategy built on top of them - degrades in reliability.

The contract is the seam that makes clean testing possible. Once it's in place, tests at the boundary can be fast and focused because they're testing through a well-defined boundary rather than around a poorly-defined one. Once it isn't, the pressure to simulate leaks in everywhere - which is where the next problem starts.

## Running Everything Locally

Which brings us to running your app locally.

If you've read this far expecting the "don't do that" section, it's not quite that. Running services locally is a perfectly fine development aid. Being able to spin something up, poke at it, watch how it responds to a real request - that's genuinely useful when you're building. There's a case for it as a way to sense-check behaviour that's awkward to unit test, or to explore how something feels before writing anything down.

But a lot of local-everything setups aren't really development aids any more. They're attempts to answer the end-to-end question - "does the whole thing work?" - on a laptop. And a laptop is the wrong place to answer that question, because a laptop isn't production. No matter how faithfully you compose the stack, you're not testing against a service mesh, a real load balancer, real DNS, real inter-region networking, or the actual SaaS services you depend on. You're testing against a scale model, and every time production surprises you, the scale model was where the surprise wasn't caught.

Teams end up here because they don't have somewhere better to answer the end-to-end question. Staging is missing, or perpetually broken, or shared with QA in a way that makes it useless for verification. Per-PR preview environments aren't a thing. So the laptop becomes the substitute - and then, inevitably, so does CI, because whatever you build locally, someone will try to reproduce in a pipeline.

The problem isn't the local stack itself. It's what it's being asked to do. If you're using Docker Compose to check that your service starts and talks to a local Postgres while you develop, that's fine. If you're using it as your primary means of verifying that a change works, you're using it as a testing strategy - and the trade-off is that you're spending real engineering time maintaining an increasingly faithful fake of production, on a machine that will never be production. At some point the effort to keep the local stack alive exceeds the value it provides. The honest question is whether that line has already been crossed.

## Clicking Around Isn't Testing

The clean version of the argument comes down to this: clicking around on a locally-running app is a fine development aid. It's how you sense-check what you're building. It gives you a fast, informal loop for exploring how something behaves. What it isn't is a test.

A test is something that survives after you've stopped clicking. It runs in CI without you. It fails when the thing it's checking regresses. It documents what the code is supposed to do in a form the next engineer can read. Clicking around and concluding "seems fine" satisfies none of these. It's verification-shaped, but it isn't verification, because nothing is preserved and nothing runs again without your hands on the keyboard.

Which is fine, as long as you don't confuse the two. Develop however you like. Boot the whole stack if it helps you think. Click around, watch the logs, satisfy yourself that the thing works. Then write the tests that let a machine confirm it, at the level that answers the right question. Narrow tests for narrow questions. Boundary tests for boundary questions. End-to-end tests, sparingly, for the ones only real infrastructure can answer.

Each test answering its own question. None of them pretending to answer someone else's.

The strategy isn't complicated. It just requires some understanding of what each test is for.
