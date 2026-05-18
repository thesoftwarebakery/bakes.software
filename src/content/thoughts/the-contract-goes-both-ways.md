---
title: "The Contract Goes Both Ways"
description: "On the importance of contract enforcement at the server level"
pubDate: "May 17 2026"
heroImage: "/blog-placeholder-3.jpg"
---

I've long been an advocate of schema-driven design; the writing of a schema (whether it be an OpenAPI schema or anything of the ilk) as part of the scoping phase of work.

I'll be focusing mainly on REST APIs and OpenAPI schemas here, but I believe as a pattern this applies in any scenario where you have data which conforms to a contract.

Schema-based development, as I see it, has several benefits:

- It gives the work to be done a bounded definition which can be agreed in advance by all parties.
- The scope of work becomes clearer. Anybody agreeing to the contract defined by the schema understands the change.
- It forces the schema author to think about the integration point of the system. This thinking happens up-front and early. Issues such as pagination for high-cardinality relationships, over-fetching data on high-request-volume endpoints, or even issues of naming convention in returned headers and fields can be caught early.
- Systems design becomes easier to fathom. Breaking changes can be easily identified beforehand, and release versions can be planned accordingly.
- A descriptive enough schema means that documentation for the relevant version is written before the feature is.
- Dependencies can validate their integration *before work has taken place*.

The last point on this list I believe is perhaps the hardest-landing. From a business perspective, this means that QA can test early, other squads' work is unblocked because they understand the work that _will_ land, and release control becomes simpler (this change is blocked by that change which lands in version `x.x.x`).

The schema is a contract. The analogy works across contexts: in a business sense, imagine if a supplier told you that you'd get a contract to sign _after_ the work has been completed. Sure, they might give you an explanation of the work to be undertaken, but there's nothing binding about an explanation.

## The Tooling Earns Its Keep

One of the under-appreciated wins of a well-written schema is that the rest of the ecosystem follows from it almost for free. Type definitions and client SDKs for most major languages can be generated from an OpenAPI document: [Hey API](https://heyapi.dev/), [openapi-generator](https://github.com/OpenAPITools/openapi-generator) and [oapi-codegen](https://github.com/oapi-codegen/oapi-codegen) are all solid options depending on your stack. Documentation sites can be generated and kept in sync automatically with tools like [Redocly](https://redocly.com/), [Scalar](https://scalar.com/) or [Fern](https://buildwithfern.com/). Mock servers, invaluable for the "consumers can build before the functionality exists" benefit I described earlier, fall out of the same schema via tools like [Prism](https://stoplight.io/open-source/prism) or [Mockoon](https://mockoon.com/).

This compounds the argument for schema-first. The schema isn't just a contract; it's a source of artefacts that your consumers actually depend on. Which brings me neatly onto my next point:

## Self-validation is Table Stakes

This, I fear, is where my opinion gets slightly controversial.

Contract compliance is one of the server's core responsibilities. Not only on request, **but also on response**. A server which returns a response which does not validate against its own schema **is a server error** and should be treated as such.

Consider this scenario. The schema says `status` is one of `"pending"`, `"active"`, or `"complete"`. The server returns `"active_v2"`, perhaps because someone added a new state to the database and forgot to update the schema. The response still validates as a string. Downstream, the consumer's switch statement has no branch for it; the code falls through to a default case that nobody thought hard about. The bug surfaces in the consumer, gets triaged as a consumer bug, costs an afternoon to trace, and the actual culprit is sitting in a service nobody thought to look at.

The server lied about its own behaviour, and the consumer paid for it. Go back to the supplier analogy: if a supplier delivered goods that didn't match the spec they themselves had written, nobody would call that a customer error. It would be a breach of contract on the supplier's side. APIs deserve the same logic.

This isn't only about response bodies. The contract covers the whole response; headers, status codes, content types. A `201 Created` that omits the `Location` header, an endpoint that advertises 200/400/404 but quietly returns a 204 when results are empty, a response that declares `Content-Type: application/json` and ships XML. All of these are the server failing to deliver what its schema promised. The semantic argument doesn't change because the violation happened in a header field. The server broke its contract; that's a 5xx.

A few objections often come up:

> But some of the data we return is unstructured

The claim is that you can't validate what you don't constrain: user-provided JSON blobs, free-form metadata, etc.

Free-form data can absolutely be validated. Sure, it's not an enum, but it could be a text blob, an object with `additionalProperties`, an array of items with `type: "object"`. The schema doesn't pretend to constrain the shape, it just documents that the shape is open. There's always _some_ structure worth asserting. Even if it's permissive, it should still be enforced.

> But our endpoints return different shapes depending on...

The argument here is that your schema can't cope with discriminated data shapes.

While I do appreciate that authoring schemas can be a fiddly process, this is absolutely something that OpenAPI and JSON Schema support. `anyOf`, `oneOf` and `discriminator` can all be used in these schema languages. You describe each possible shape and the condition under which that shape is returned.

If you're finding that your endpoint definition is becoming too complex, I would argue that's probably a schema design problem, and an indicator that it's worth considering simplifying.

> But this is just defensive programming

It isn't. Defensive programming is what you do when you don't trust your inputs. Self-validation is what you do when you take your own outputs seriously enough to enforce them. Your language already does this for you inside a single process (Rust won't let you return the wrong type from a function) but at the service boundary, the type system stops checking. The schema is how you put it back.

## A Note on Adjacent Protocols 

gRPC is a different case. Protobuf is schema-first by construction; you cannot meaningfully write a gRPC service without the `.proto` file existing first, and code generation in every supported language falls out of it the same way. The wire format is strongly typed, so a structurally malformed response can't even leave the server. In that sense gRPC has taken the thesis of this post seriously at the protocol level, which is to its credit. What it doesn't catch is the semantic layer - enum values outside the expected set, business invariants, the "active_v2" case from earlier. The structural type system handles part of the job; the rest still needs explicit validation. The argument applies, just to a smaller surface.

And a sympathetic word for anyone still on SOAP. WSDL is, for all its faults, exhaustively schema-first. The tooling generated stubs in both directions, validated both ways, and treated the contract as the source of truth in a way most modern REST stacks still don't approach. The SOAP world solved a lot of this decades ago and then we collectively threw it out for being miserable to work with (understandably). Some of the pendulum-swing back towards strict contracts in REST is, somewhat, an admission that the pendulum swung too far.

It's worth addressing GraphQL briefly, because schema-first is sometimes conflated with it. GraphQL is schema-driven, yes, but in a way that runs in the opposite direction to what I'm describing. The schema is coupled to resolvers; the implementation is what gives the schema meaning. The schema describes what the code does, rather than the code being held to what the schema promised. The arrow points the wrong way.

There's also no clean equivalent of versioned breaking changes. The convention is to evolve additively and deprecate fields in place, which works until it doesn't. The coordination benefits I described above largely don't apply.

## What This Looks Like in Practice

The good news is that adding this to an existing service isn't a heavy lift. Most ecosystems have a validator that takes a schema and a payload and returns pass/fail: `ajv` for Node, `jsonschema` or `pydantic` for Python, the `jsonschema` crate for Rust. Wire it in as middleware on the response path and you're most of the way there.

The harder question is what to do when validation fails. In dev and staging, the answer is obvious: fail loudly. A test suite that catches a broken response before it hits production is exactly what you want.

Production behaviour deserves careful thought, but the choice is starker than it might first appear. You have two options: breach the trust boundary and return invalid data, or accept responsibility and return a 5xx. The trust boundary was put forward by you, the server, and agreed to implicitly by every consumer that took your schema at its word.

It's worth asking what the actual argument for returning invalid data in production even is. "Graceful degradation" doesn't hold up; if you want degraded responses to be valid, declare them so in the schema. "Avoiding cascading failures" doesn't hold up either; that's an argument for better client-side resilience, not for servers lying about their contracts. A malformed response masquerading as a 200 is worse than a 5xx for the same reason a defective product shipped in branded packaging is worse than a delivery failure: the second is a setback, the first is a betrayal of the label.

A 5xx holds the server to a high standard. It says: I broke my contract, here is an honest signal that I did so, handle me accordingly. Returning invalid data instead is just shifting the blame to the victim. The client now has to detect, diagnose and work around a failure that originated upstream, and they have to do so without the courtesy of being told that's what's happening.

Whatever you choose, the 5xx itself is only useful if you actually see it. Wire it into your observability stack with a counter on schema-validation failures, a dashboard, an alert threshold. The schema gives you something most error metrics don't: a precise definition of what "wrong" means, which makes the signal much cleaner than "something 500'd somewhere".

What you do with the signal is up to you. You can treat it with a soft touch - check the dashboard once a month, triage anything that's trending up, fix it on the next sprint. Or you can take a harder line and PagerDuty your favourite engineer at 3am every time a malformed response slips through. Both are defensible. One makes for a calmer team; the other makes for fewer malformed responses. Choose your own adventure.
