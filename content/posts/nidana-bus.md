+++
date = '2026-04-05T11:13:21+07:00'
draft = false
title = 'Nidana Bus: What If Your Frontend Treated Data Flow Like Infrastructure?'
tags = ["mobile", "web", "functional programming", "software architecture"]
toc = true
+++

I've been dabbling in bus-based architecture for the frontend for over 10 years, and have successfully applied it in one form or another to production apps with millions of users. This is the ideal architecture as far as I can envision (with the help of Claude Opus 4.6). I intend to build complete libraries for multiple platforms based on this [Nidana Bus Reference Architecture](https://github.com/Samsara-Stream/nidana-bus-dart/blob/main/docs/nidana-bus-ref-arch-v0_11.md).

---

## The Problem Nobody Talks About

Here's a question: where do side effects live in your app?

If you're honest, the answer is "everywhere." Your `ViewModel` fetches data, mutates state, and triggers navigation. Your BLoC subscribes to a stream, calls a service, catches an error, and emits a new state, all in the same class. Your React component fires off a fetch in a `useEffect`, writes to a Zustand store, and conditionally redirects the user.

This works fine for toy projects. It stops working when:

- A background token refresh needs to trigger re-fetches across five independent pages.
- A connectivity change must pause all outgoing requests, queue analytics events, and show a banner, without any of those features knowing about each other.
- A logout event needs to clear the cart, reset onboarding progress, disconnect a websocket, and navigate to login.

The standard response is ad-hoc wiring: shared singletons, callback chains, context providers that everything re-renders through, or a global Redux store that becomes a coordination bottleneck. None of these make the data flow *visible*. None of them make side effects *controllable*. And none of them scale gracefully to large teams working on independent features.

The deeper issue is **purity**, or rather, the lack of it. When business logic, I/O, state mutation, and rendering are interleaved in the same component, you cannot test the logic without mocking the I/O, you cannot reason about state without simulating the lifecycle, and you cannot change one concern without risking another. Every design principle that matters in practice (declarative composition, loose coupling, testability) is really just the practical expression of one underlying concern: *separate the pure from the impure, and push side effects to the boundaries.*

Nidana Bus is an architecture built around that separation.

## The Idea: Kafka, But In-Memory and For Your UI

Backend engineers solved the coordination problem years ago. Services don't call each other directly; they publish to typed channels (Kafka topics, RabbitMQ queues) and subscribe to the channels they care about. The result: loose coupling, independent scaling, and observable data flow.

Nidana Bus applies this model to frontend applications. A reactive event bus sits at the center of the app. **Topics** are typed channels. **Topologies** are declarative descriptions of how topics relate to each other through pure transformations. **Services** (network, persistence) and **UI pages** (rendering, gestures) sit at the boundaries as imperative shells that publish to and subscribe from topics.

The frontend context actually makes this *simpler* than backend messaging. The bus is in-memory within a single process, so stateful stream operations like join, merge, and windowing are cheap. There's no serialization overhead, no partitioning complexity, no distributed consensus. You get the architectural benefits of event-driven coordination without the infrastructure cost.

## Three Layers, One Rule

The architecture has three layers:

1. **Upper Shell (Services):** I/O side effects. Network calls, disk reads, sensor access. Each service defines a topology and publishes results to topics.
2. **Pure Core (Bus + Topologies):** No domain side effects. Topics hold typed state. Topologies declare stream transformations as pure functions wired together. The bus activates and manages them.
3. **Lower Shell (UI Pages):** Rendering side effects. Pages subscribe to topic output and publish user interactions back.

The rule: **side effects only happen at the shells.** The core is declarative. A topology says "combine auth state and cart items into checkout UI state" and that transformation is a pure function you can call directly in a unit test.

```
// The topology is just wiring
topology("checkout-flow") {
  val uiState = combine(
    read(CheckoutTopics.cartItems),
    read(AuthTopics.state),
    ::buildCheckoutUI     // function reference, not inline closure
  )
  write(CheckoutTopics.uiState, uiState)
}

// The transformer is a pure function, testable with zero framework involvement
fun buildCheckoutUI(cart: CartItems, auth: AuthState): CheckoutUIState {
  return CheckoutUIState(
    items = cart.items,
    canCheckout = auth.isLoggedIn && cart.items.isNotEmpty()
  )
}
```

Services don't know about each other. Pages don't know about services. The only shared contract is the **type** carried by a topic. `PaymentService` reads from `Topic<AuthState>` and doesn't know `AuthService` exists. Replace the auth implementation entirely, and payment keeps working as long as the `AuthState` type stays the same.

## What This Actually Buys You

### Taming complexity at scale

For a two-screen app, this is overkill. For an app with 50 screens, 15 services, and 8 feature teams, the explicit topology declarations are the only thing that keeps data flow comprehensible. Every relationship is visible, typed, and auditable. A CI step can detect when a PR adds an unexpected dependency between features.

### Measurably more stable apps

Immutable data on topics eliminates shared-mutable-state races by construction. Topology isolation prevents cascading failures (an error in payment doesn't crash analytics). Scoped lifecycle management prevents subscription leaks. These aren't aspirational; they're structural properties. The remaining failure categories (main-thread blocking, circular dependencies) are confined to well-defined boundaries where they're visible and fixable.

### Testability as a first-class property

This is arguably the strongest practical selling point. Pure transformers are tested by calling them directly: `buildCheckoutUI(testCart, testAuth)` and assert. No mocking frameworks, no lifecycle simulation, no DI container setup. Topology wiring is tested by publishing synthetic values on a test bus and asserting output. UI testing collapses to: inject a test bus, publish known state to input topics, assert the UI renders correctly. The entire test pyramid becomes simpler because purity is enforced by the architecture, not by discipline.

### Elegant cross-cutting concerns

Analytics, error handling, connectivity monitoring: these are just services with topologies, like everything else. An analytics service reads from multiple event topics and sends data to a backend. A connectivity service observes an OS API and writes to `Topic<Connectivity>`. No special interceptor framework, no AOP, no middleware chain. The same pattern handles every concern.

### AI-agent compatibility

This is forward-looking but increasingly practical. A pure transformer with a typed signature is an ideal unit for AI-assisted development: bounded context, clear contract, fast feedback loop. An AI agent modifying one topology cannot accidentally break another because the isolation is structural, not dependent on the agent understanding the full system. Multiple agents can work on different topologies in parallel with zero coordination.

## The Honest Trade-offs

**Learning curve.** The conceptual model (topics, topologies, scopes, the shell/core separation) requires upfront investment. Developers comfortable with "just call the service" will find the indirection unfamiliar. The payoff is downstream, and it's real, but the ramp-up cost is nonzero.

**Overhead for small projects.** If your app has three screens and one API, this architecture adds ceremony without proportional benefit. It's designed for apps that are growing and will continue to grow.

**Reactive programming exposure.** The topology DSL abstracts most of the complexity, but debugging stream pipelines still requires some understanding of reactive semantics (backpressure, subscription timing, error propagation). Teams unfamiliar with reactive programming will need time to build intuition.

**Ecosystem adoption.** Nidana Bus is a new architecture. It doesn't yet have the ecosystem depth (DevTools, community plugins, Stack Overflow answers) that established patterns have. Early adopters will need to invest in building some of that infrastructure themselves.

**Debugging indirection.** When something goes wrong, the stack trace passes through the bus runtime and reactive operators rather than a direct call chain. The architecture provides tools to compensate (correlation IDs, causation chains, topology graph visualization), but "just step through the debugger" is less straightforward than with direct service calls.

## Where It Fits

Nidana Bus is **not** a replacement for your component-level state management. Zustand, Jotai, or local `useState` still handle ephemeral UI state (which tab is selected, what the form values are). It's not a replacement for dependency injection either; you still need DI to provide platform infrastructure.

Nidana Bus replaces the ad-hoc wiring *between* features. The shared singletons, the callback chains, the "just inject ServiceA into ServiceB so it can call `getToken()`" patterns that accumulate as an app grows. It makes that coordination layer explicit, typed, testable, and observable.

The adoption path is incremental. Start with one cross-cutting concern as a service topology (connectivity monitoring is a good candidate). Expose it via a framework hook (`useTopic` in React, `async` pipe in Angular). Existing state management continues working unchanged. As more features need cross-feature coordination, add topologies. Migration is gradual, not all-or-nothing.

If your app has outgrown the point where "just wire it together" is maintainable, and you care about testability, stability, and keeping data flow visible as complexity grows, Nidana Bus is worth a serious look.

---

*The full reference architecture (covering lifecycle management, data contracts, platform mappings for Dart/Flutter, Kotlin/Android, Swift/iOS, and TypeScript/Web, formal properties, and more) is available in the [Nidana Bus Reference Architecture](https://github.com/Samsara-Stream/nidana-bus-dart/blob/main/docs/nidana-bus-ref-arch-v0_11.md).*
