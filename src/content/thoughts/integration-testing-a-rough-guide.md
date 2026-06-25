---
title: "Integration Testing: A Rough Guide"
description: "A practical approach to making sure your thing works correctly with the other thing"
pubDate: "June 24 2026"
heroImage: "/integration-testing-a-rough-guide.jpg"
---

"Integration testing" is one of those terms with an incredibly broad set of definitions. At one end of the spectrum, it might mean checking that one class correctly calls a method on another. At the other, it's exercising a full Kafka send/receive loop. Somewhere in the middle, it's probably "does my code actually write to the database the way I think it does." Anything along this spectrum is an integration test by some definition, and arguing about which one *really* deserves the name is a somewhat fruitless exercise.

This post isn't going to debate the definition. The thing those examples have in common, and the thing worth talking about, is that at some point your code has to talk to something it doesn't own, and you'd like to know it does that correctly without booting the entire application to find out. Whether your app is some monolith with a MySQL database and outbound calls to Stripe, or several services each with their own infrastructure, that problem is fundamentally the same.

What follows is a walkthrough of an approach that handles it relatively cheaply: [testcontainers](https://testcontainers.com/) for the stateful dependencies you genuinely want to exercise, request-level interception for outbound HTTP, everything else mocked at a sensible boundary, and the whole lot running in-process. The examples are TypeScript with OpenAPI-generated clients, but nothing about the underlying approach is specific to that. Testcontainers exists in most languages. [MSW](https://mswjs.io/) has equivalents in most languages. The principles are language-agnostic.

The goal: tests that ensure CI runs smoothly and the feedback loop is fast, that are focused enough to actually catch bugs, and that strike a balance between power and simplicity. The latter also happens to make them pleasant to write, rather than a chore. In theory, this makes tests like these dual-purpose: both a CI gate and a valuable development tool.

If you'd like to skip the explanation and go straight to a demonstration of these concepts in practice, there's a [companion repository](https://github.com/thesoftwarebakery/integration-testing-a-rough-guide) which does just that.

## The Setup

The model is simpler than the tooling suggests. For any test, you've got three categories of dependency:

1. **Stateful infrastructure you control** - your database, your message broker, your cache. Things with non-trivial behaviour where the actual implementation matters. These get real instances, spun up as containers.
2. **HTTP services you call** - other internal services, third-party APIs, your auth provider, whatever. These get intercepted at the network layer with a typed client and a mock.
3. **Everything else** - bits of your own code you want to isolate from for the test in question, SaaS SDKs whose behaviour you don't need to reproduce, feature flag clients, and so on. Mock these at whatever boundary makes sense.

One cardinal rule, worth stating explicitly because it sounds obvious until you notice how often it's violated: don't try to faithfully reproduce things you don't own. If you find yourself building a "mock" container that pretends to be a third-party SaaS, stop. You're not testing integration with that service any more, you're testing whether your mock of it agrees with your other mock of it. Mock the SDK at its boundary, trust the SaaS vendor's tests, and focus on the parts you own.

## Contracts and Generated Clients

The foundation of all this is taking your service contracts seriously. If your service exposes a REST API, it should have an OpenAPI spec. If it consumes another REST API, it should consume it through a typed client generated from *that* service's OpenAPI spec.

The win here isn't only type safety, it's that the spec becomes the shared artefact between services - a single source of truth defining what one service is allowed to ask of another. Test against the contract, and you're testing the integration. You don't need the other service running to do it.

A representative layout:

```
packages/
  service-a/
    src/
    test/
  service-a-sdk/         # generated from service-a's OpenAPI spec
    src/client.ts        # typed HTTP client
    src/factories.ts     # factory functions for fake requests and responses
    src/handlers.ts      # generated MSW handlers for in-process HTTP interception
  service-b/
  service-b-sdk/
```

Each service ships its own SDK package alongside it. The SDK contains the generated client, factories for constructing valid responses in tests, and a set of default MSW handlers that return reasonable defaults. When service A calls service B, it imports `service-b-sdk` and uses the client. When service A's tests need to simulate service B's responses, they import the factories and handlers from the same package.

Generating the factories, types and SDKs should be an integral part of the build tooling. In a TypeScript world, Nx supports this with its `dependsOn` syntax, but pretty much any build tool you choose will have the capacity to set up similar chaining. For bonus points, ensure that code generation happens _before_ your project is built - that way you can use a service's types generated from its schema in its code.

The factories are the parts that represent the data in flight. A typed factory like:

```ts
export const orderFactory = (overrides: Partial<Order> = {}): Order => ({
  id: 'ord_123',
  status: 'pending',
  total: 4200,
  ...overrides,
});
```

…means tests can express "what happens when service B returns an order with status `cancelled`" in one line, with full type safety, without having to know about service B's internal representation. When service B changes its API, the changed interface in the SDK breaks the tests that depend on it - that's how you know you have an integration failure.

## HTTP Mocking with MSW

MSW intercepts HTTP calls at the request level. You define handlers that match URLs and return responses, and any HTTP client - whether it's fetch, axios, your generated client - hits those handlers instead of the network. Interception happens in-process; no separate server, no extra container, no port juggling.

In a test, it looks roughly like:

```ts
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';
import { orderFactory } from 'service-b-sdk/factories';

const server = setupServer(
  http.get('http://service-b/orders/:id', () =>
    HttpResponse.json(orderFactory({ status: 'cancelled' }))
  )
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('handles cancelled upstream orders correctly', async () => {
  const result = await myService.processOrder('ord_123');
  expect(result.status).toBe('rejected');
});
```

Each test gets to declare exactly what its upstream dependencies will return, scoped to that test. The default handlers from the SDK package cover the boring cases; tests override them for the interesting ones.

There's a wealth of tooling in this area that can enrich the process of creating MSW handlers and factories from an OpenAPI schema and tie together your client, factories and handlers with a suite of types - MSW even ships with its own [OpenAPI handler generation](https://source.mswjs.io/docs/integrations/open-api/), and there are suites of tooling which will generate the whole lot for you, like [Orval](https://orval.dev/docs/guides/faker/), but for the sake of the demonstration, the examples here have been kept simple.

**A note on WireMock**, because someone will ask. WireMock is the language-agnostic standard for HTTP mocking and it's good at what it does. However, it runs as a separate HTTP server. That makes sense when you can't get inside the thing you're testing - black-box deployable artefacts, cross-language contract testing, anything that crosses a network you don't own. But here, the thing you're testing is in-process code that you wrote, and the network hop adds nothing except latency, a process to manage, and a port to allocate. Skipping it costs you nothing and gains you a faster, simpler test suite.

MSW isn't the only option, either. [nock](https://github.com/nock/nock) does essentially the same job in Node and works fine. In other languages, look for whatever does request-level interception in-process. And if you'd rather mock at the function boundary - wrapping HTTP calls in strongly-typed functions and stubbing those directly - that's a perfectly defensible choice with a slightly different trade-off: simpler setup, but you're not testing your client's serialisation behaviour in those tests. Fine, provided you trust the generated client (you probably can) and there's a separate test for the HTTP layer somewhere.

## Testcontainers for the Things that Actually Matter

For stateful dependencies whose actual behaviour you want to exercise, e.g. databases, message brokers, sometimes auth providers - testcontainers gives you the real thing in a container, spun up per test suite, torn down after.

```ts
import { PostgreSqlContainer } from '@testcontainers/postgresql';

let pg: StartedPostgreSqlContainer;
let repo: OrderRepository;

beforeAll(async () => {
  pg = await new PostgreSqlContainer('postgres:16').start();
  const db = createDbClient(pg.getConnectionUri());
  await runMigrations(db);
  repo = new OrderRepository(db);
});

afterAll(() => pg.stop());

test('upserts orders correctly on duplicate id', async () => {
  await repo.upsert({ id: 'ord_1', status: 'pending', total: 100 });
  await repo.upsert({ id: 'ord_1', status: 'paid', total: 100 });

  const result = await repo.findById('ord_1');
  expect(result.status).toBe('paid');
});
```

The repository layer is being tested directly against a real Postgres, in-process, no HTTP layer in sight. If `upsert` has a subtle bug in how it constructs the SQL, this catches it. No web server boot, no request fixtures, no waiting for half a dozen other services to come up before the first assertion runs.

The same applies to message brokers. Spin up Kafka in a container, write a test that sends a message and asserts on what came out the other side, and you've verified actual Kafka behaviour - serialisation, partitioning, consumer group semantics. And here's the useful part: you only need to test that stuff thoroughly once. Once you've established that your Kafka producer/consumer wrapper behaves correctly, most of your other tests can mock at the wrapper boundary and skip the container entirely. The Kafka integration test is a separate concern from "does this business logic handle a particular incoming message correctly."

That selectivity is the point. You're not running every container for every test; you're choosing which dependencies each test genuinely needs to exercise, and mocking the rest.

## What This Unlocks

The thing that surprised me most about this approach, the first time I set it up properly, was how much more I could do in a single test. Tests that exercise an application end-to-end are necessarily blunt instruments: you give it an input, you get an output, and everything in between is opaque. You can assert on the response, maybe peek at a database row afterwards, and that's about your lot.

In-process tests with selective mocking give you the middle. You can inspect that a function was called with particular arguments. You can verify a side effect happened without round-tripping through HTTP. You can test a single layer of your service - just the repository, just the domain logic, just the HTTP handler - by mocking everything around it. A test can assert "given this row in the database, my domain service produces this output" without involving a single HTTP request, and a separate test can cover the HTTP layer with a mocked domain service. Each test is doing one job, on one layer, and only the things relevant to that job are spun up.

That granularity makes the tests themselves more useful. They become documentation for the next engineer who looks at the code: *this is how the OrderService handles a cancelled payment, here's a test for it.* They become a fast feedback loop while developing: change a function, run twenty tests focused on that function in a couple of seconds, see what broke. They become executable contracts: this is what this layer promises to do, written down in a way that can't drift from the implementation without somebody noticing.

## Optimisations and Gotchas

The single biggest performance win, once everything is wired up, is to stop spinning containers up and tearing them down between tests. Each container takes a couple of seconds to start; multiply that by every suite in a moderately-sized codebase and you're well into "go make a coffee" territory before the first assertion fires.

The pattern that works best is one instance of each container type for the entire test run, with data isolation handled inside. Spin up a single Postgres container at the start, run migrations on it once, and give each test suite its own database within that container by cloning a template. The same idea applies elsewhere: topic-per-test inside a single Kafka broker, bucket-per-test using S3Mock. The granularity depends on cost. Cloning a database isn't free, so per-suite tends to be the right balance. Creating a Kafka topic is cheap, so per-test is fine.

The benefit isn't just speed, though that's substantial. Data isolation within a shared container means cleanup mistakes don't cascade. If somebody forgets to delete a row in a teardown, the blast radius is limited to other tests in that suite rather than the entire test run. Tests that pass locally but fail in CI because they happened to run in a different order become a much rarer occurrence. Debugging is easier too: you can drop into a single failing suite, leave its data lying around, and inspect it without polluting anything else.

A few other gotchas worth knowing:

- **MSW handlers leak between tests if you forget to reset them.** Put `server.resetHandlers()` in `afterEach`, set it up once, never think about it again.
- **Generated clients drift from specs unless you enforce regeneration.** Either a CI check that fails when the generated code doesn't match the spec, or don't commit the generated code at all and generate it as part of the build. I prefer the latter - treat generated code as a build artefact, the same way you do your `dist` folder.
- **MSW can intercept testcontainers' calls to the Docker socket.** This one stumped me for a while, and is another argument for spinning up your containers once - and before you register your MSW handlers. MSW's interception can interfere with testcontainers communicating with the Docker socket in various ways, but making sure you've performed all of your Docker operations before registering the MSW handlers avoids this class of problem entirely.

## The TL;DR

The whole approach is underpinned by three principles: mock at the service boundary, do it in-process, and don't simulate things outside of your control. Once the basic structure is in place, you get fast tests, granular assertions, and a feedback loop that makes development feel less like a tax. There is a setup cost, but it's paid once, and the result is a testing setup that scales with your application.

For a demonstration of this in practice, see [the companion GitHub repository](https://github.com/thesoftwarebakery/integration-testing-a-rough-guide)
