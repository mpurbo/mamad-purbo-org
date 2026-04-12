+++
date = '2026-04-12T11:13:17+07:00'
draft = false
title = "The Side Effect You Can't See: Perceptual Parochialism in Software Engineering"
tags = ["philosophy", "software engineering", "functional programming"]
toc = true
+++

> This article is a companion to [Perceptual Parochialism: The Scale We Can't See](https://mamad.purbo.org/posts/perceptual-parochialism/). That piece explores why humans, by default, struggle to reason about phenomena at scales beyond direct experience. This article applies the same idea to a specific domain where it causes enormous, measurable damage: software systems with uncontrolled side effects.

## The Pebble and the Mountain Range

A mutable variable in a 200-line script is a pebble in your hand. You can feel its weight, see its shape, track exactly how it moves. A mutable variable in a 2-million-line distributed system is a pebble somewhere in a mountain range. Same pebble. But your ability to locate it, predict its effects, and reason about its interactions has collapsed entirely. And at this scale, a pebble can trigger a landslide.

The side effect didn't get worse. Your cognitive access to its consequences did.

This is the core problem, and it's the same problem described in the companion article on perceptual parochialism. Just as our brains compress geological timescales into a vague blur of "a long time," they compress software complexity into a vague blur of "a big system." The internal structure, the combinatorial explosion of possible interactions, the emergent behaviors that arise from thousands of individually harmless mutations, all of this gets flattened.

And because it gets flattened, we make decisions at small scale that become catastrophic at large scale. Not because we're careless, but because we literally cannot see the consequences.

## Side Effects That Looked Harmless

Let's make this concrete. Here are patterns that every experienced engineer has encountered, and most have shipped.

### The Shared Mutable Cache

At small scale, it looks like this:

```python
# "Just a cache, what could go wrong?"
_user_cache = {}

def get_user(user_id):
    if user_id not in _user_cache:
        _user_cache[user_id] = db.fetch_user(user_id)
    return _user_cache[user_id]
```

In a tightly bounded context, this is acceptable. One process, one thread, predictable access patterns. Every developer who writes this is making a *locally correct* decision. The mutation is visible, the state space is small, and the performance benefit is real.

Now scale it. The application grows. Multiple threads call `get_user` concurrently. The cache becomes a shared mutable structure accessed from request handlers, background workers, and event processors. Someone adds a function that *modifies* the returned user object (they don't realize it's the cached reference). Now the cache silently contains corrupted data. Requests start returning stale or partially modified user records. The bug is intermittent, timing-dependent, and nearly impossible to reproduce locally.

Here's the critical point: the local code looks unchanged. What changed is the *operational context* in which it runs: concurrency, aliasing, lifetime of references. That context grew beyond the developer's ability to reason about it.

A pure alternative would return a fresh copy or an immutable data structure. The performance cost is usually acceptable relative to the debugging cost it prevents, though this is workload-dependent. The reasoning cost saved is enormous.

### The Implicit Ordering Dependency

```java
// Service initialization
public void initialize() {
    configService.load();        // must be first
    authService.connect();       // needs config
    cacheService.warmUp();       // needs auth
    eventBus.start();            // needs cache
    metricsService.register();   // needs eventBus
}
```

At small scale, this is a perfectly readable sequence. Five steps in a clear order. Any developer can understand it.

But this code encodes an implicit dependency graph through *temporal ordering*. The dependencies aren't declared; they're implied by the sequence of side effects. Each `init` call mutates shared state that the next one reads.

At scale, this becomes a minefield. A new engineer adds `notificationService.start()` between `authService` and `cacheService` because it "seems like the right place." It works in dev. It fails in staging because `notificationService` triggers an event that `cacheService` hasn't warmed yet. The failure mode is a null reference three stack frames deep in an event handler.

The fix takes a week. Not because the bug is complex, but because understanding *why* the ordering matters requires tracing invisible state dependencies across five services. The information needed to prevent the bug was never encoded in the code. It existed only in the original developer's head.

A declarative dependency graph (even a simple one) makes these relationships explicit and machine-verifiable. The system can detect the cycle or the missing dependency at startup rather than at 3 AM in production.

### The Non-Idempotent Handler

```python
# "Process payment when order is confirmed"
def handle_order_confirmed(event):
    charge_customer(event.user_id, event.amount)
    deduct_inventory(event.item_id, event.quantity)
    send_confirmation_email(event.user_id, event.order_id)
```

This handler is correct. It does exactly what it should: charge the customer, adjust inventory, send a notification. In development, in staging, and in early production, it works perfectly. Every event arrives once, gets processed once, produces the right outcome.

But most real messaging systems (Kafka, SQS, Pub/Sub, RabbitMQ) provide *at-least-once* delivery, not exactly-once. At small scale, duplicates are so rare they might never occur. You could run this in production for months without seeing one. There's nothing to notice, no signal that anything is wrong with the design.

At scale, duplicates become a certainty. Consumer rebalances, network timeouts, retry storms, partition reassignments: these are routine operational events in a distributed system, and each one can redeliver messages. Now customers get charged twice. Inventory goes negative. Confirmation emails arrive in triplicate. The side effect (mutating state on receipt) was always there and always looked fine. What changed is that the operational context grew into one where duplicate delivery is a statistical guarantee rather than a theoretical possibility.

This is scope insensitivity applied to code. The handler is correct in a world with exactly-once delivery. It's just that the real world, at scale, isn't that world, and the gap between the two is invisible until the system is large enough for duplicates to become routine.

### The Convenient Global Toggle

```python
# Feature flag, simple and useful
ENABLE_NEW_PRICING = True

def calculate_price(item):
    if ENABLE_NEW_PRICING:
        return new_pricing_engine(item)
    return legacy_pricing(item)
```

One flag, one branch. Totally manageable.

Now multiply. After two years, the system has 47 feature flags. Some are checked in request handlers, some in background jobs, some in event processors. Some flags depend on other flags. Some were supposed to be temporary but nobody removed them. The function `calculate_price` now checks three flags, and the interaction between them creates eight possible code paths, only three of which have ever been tested.

The state space has grown combinatorially (2^47 possible configurations in the worst case, even if constraints reduce the effective surface), but the team's mental model is still "we have some feature flags." This is logarithmic cognitive compression applied to configuration complexity. The *felt* complexity of 47 flags is maybe twice the felt complexity of 5 flags. The *actual* complexity is roughly 10^14 times larger.

Any individual flag is a reasonable engineering decision. The aggregate is an unreasonable system that no one can fully understand, and the transition from reasonable to unreasonable was invisible because it happened gradually, one "harmless" flag at a time.

## The Pattern

All four examples share the same structure:

1. **Local innocence.** The side effect is reasonable and even beneficial at the scale where it's introduced.
2. **Invisible accumulation.** As the system grows, side effects interact combinatorially, but this growth is imperceptible because our cognition compresses it.
3. **Threshold collapse.** At some point, the system crosses a complexity threshold where the accumulated side effects overwhelm human reasoning capacity. Bugs become intermittent, root causes become expensive to reconstruct, and refactoring becomes painful because you can't determine what depends on what.
4. **Refactoring resistance.** By the time the problem is visible, the side effects are so entangled with the system's behavior that cleaning them up can force large-scale refactoring if left unchecked. The mutable cache, the implicit ordering, the non-idempotent handlers, the flag combinations: they've become load-bearing parts of the system's behavior, including its *buggy* behavior that other code has been written to compensate for.

This fourth point is especially cruel. Side effects don't just make systems hard to reason about. They make systems hard to *change*, which means they resist the very fix they need.

## Why "Be Careful" Doesn't Scale

The most common response to concerns about side effects is: "Just be disciplined. Document your mutations. Review carefully. Write good tests."

This is the software engineering equivalent of "just be careful with your investments" or, as argued in the companion article, "surely something this complex must have been designed." It's an appeal to human vigilance as a substitute for structural guarantees.

The problem is that human vigilance is itself subject to perceptual parochialism. You can be vigilant about things you can perceive. You cannot be vigilant about combinatorial interactions among 47 feature flags, or timing-dependent race conditions in a cache accessed by 200 concurrent goroutines, or duplicate message delivery across a distributed event pipeline with dozens of consumer groups.

Code review helps, but reviewers suffer from the same cognitive compression as authors. A diff that introduces one new mutation looks fine. The reviewer would need to hold the entire system's state-flow graph in their head to see why it's dangerous, and they can't, because that graph is beyond human working memory.

Testing helps, but example-based tests verify *expected* behaviors. Side-effect bugs are almost by definition *unexpected* behaviors, emergent interactions that nobody anticipated. Property-based testing and fuzzing can catch some of these, but they work best when you can specify invariants clearly, and the hardest side-effect bugs are precisely the ones where the violated invariant was never articulated.

Documentation alone helps least of all, because it requires the person writing the code to anticipate the very scale effects they can't perceive. "This cache must not be accessed from multiple threads" is a constraint that the original author might document. "This cache's reference semantics will interact with the order service's event-driven update path to produce corrupted user records when the inventory adjustment handler runs concurrently under load" is not something anyone documents, because no one sees it until it happens.

## Purity as Scale Invariance

Here's the key insight, and the one I think is underappreciated: referential transparency preserves local reasoning under composition. It is, in effect, *scale-invariant reasoning*.

A pure function means the same thing regardless of where it's called, how many times it's called, or what else is happening in the system. Its behavior is determined entirely by its inputs. This means that your local reasoning about the function remains valid at *any* system scale. You don't need to hold the whole system in your head, because the function's behavior doesn't depend on the whole system.

Uncontrolled impurity breaks this property. A function with ambient side effects, one that reads or writes shared mutable state, depends on timing, or produces different results depending on the system's global condition, has behavior that depends on context: the state of a database, the contents of a cache, the timing of concurrent operations, the values of global flags. At small scale, the context is small enough to reason about. At large scale, it isn't. The function's behavior becomes context-sensitive enough that local inspection is no longer sufficient to predict it.

This is why the pragmatist argument ("just use mutation where it's convenient, be pure where it matters") is structurally identical to the argument against evolution: "at the scale I can see, this works fine." And it does! Mutation is perfectly fine at small scale. Just like intuitive agency detection is perfectly fine at small timescales. The problem isn't that the heuristic is wrong. It's that it doesn't generalize, and we can't perceive the point where it stops working until after it has already stopped.

Functional programming, at its core, isn't about elegance or mathematical purity or academic preference. It's a set of structural constraints that make reasoning *composable* across scales. Pure functions, immutable data, explicit effect management: these extend the system of developer + code + tooling into a composition where scale-invariant reasoning becomes the default, rather than something that depends on individual vigilance.

The resistance to FP is, I believe, a form of the same perceptual parochialism that makes people resist deep time or compound interest. Imperative programming *feels* natural because it mirrors how we experience agency in daily life: do this, then that, change this thing. FP asks you to think in terms of transformations and composition, which is how systems actually behave at scale but is far removed from embodied human experience.

We resist FP for the same reason we resist evolution: both require abandoning the intuition that our experiential frame is the correct one.

## What We Can Do

I'm not arguing that all code should be purely functional, and FP is not the only family of tools that addresses these problems. Actor isolation, explicit state machines, ownership systems, idempotent handler design, transactional semantics: these all attack the same underlying issue from different angles. What they share with FP is that they make the context of effects explicit and structurally constrained rather than implicit and ambient.

What I *am* arguing is that learning to think in terms of purity, composition, and explicit effects gives you something valuable even when you choose a different solution: *awareness*. Once you've internalized why uncontrolled mutation is dangerous at scale, you can reach for mutable state deliberately, understanding the tradeoff, rather than reaching for it reflexively because it feels natural at the scale you're currently looking at. The difference between choosing imperative code with awareness and choosing it out of habit is the same difference the companion article describes between understanding deep time and compressing it into a blur.

Some practical principles:

**Push effects to the edges.** Keep the core logic pure. Let the boundaries of your system (HTTP handlers, database access, event emission) be the only places where side effects occur. This is sometimes called the "pure core, imperative shell" pattern. It gives you a large region of code where local reasoning remains valid at any scale, surrounded by a thin layer where you accept the cognitive cost of effects.

**Make dependencies explicit.** If function B depends on state that function A produces, encode that dependency in types, function signatures, or at minimum a dependency injection framework. Don't rely on temporal ordering (A runs before B) to encode structural relationships.

**Treat shared mutable state as a liability.** This doesn't mean never using mutation. Local mutation inside a tightly bounded function, actor-local state with no aliasing, transactional mutation with strong isolation: these are all fine. What deserves scrutiny is *ambient* shared mutable state, the kind that leaks across boundaries and accumulates invisible dependencies over time.

**Invest in making effects visible.** Whether through type systems (Haskell's IO, algebraic effect systems), ownership and isolation boundaries (Rust's borrow checker for aliasing, actor models for concurrency), architectural patterns (explicit state machines, command/query segregation), or even just conventions (all mutations go through a single state manager), make it so that a developer reading the code can see *where* effects happen without tracing the entire call graph.

The goal isn't purity for its own sake. The goal is building systems where reasoning remains valid as the system grows, where the composition of developer and code doesn't degrade at scale. Because the system will grow. And when it does, the side effects you can't see today will be the outages you can't debug tomorrow.