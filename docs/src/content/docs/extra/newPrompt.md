I want to plan and execute a new major release of lavalink-client with a full architectural refactor focused on extensibility, Bun-native runtime support, API consistency, type safety, removal of circular dependencies, and high performance. The result should be a true breaking-change release with a cleaner and more scalable internal design than the current one.
Main goal

Refactor lavalink-client into a modular, extensible, Bun-native Lavalink v4 client with:

    first-class subclassing/extensibility,

    strong TypeScript generics and IntelliSense,

    cleaner node creation APIs,

    zero circular class references,

    a dedicated REST abstraction on nodes,

    fluent chainable APIs even for async workflows,

    prebuilt node implementations for different backends,

    and an internal design optimized for performance and maintainability.

Current project context

The current library exposes LavalinkManager as the main entry point, creates nodes from manager config, exposes nodeManager events separately, supports queue/player/filter handling, and currently documents that the client itself is not really extendable via plugins. This refactor should deliberately change that and make extensibility a first-class design goal.
Required architectural changes

1. Replace node creation via plain node option objects

Creating nodes should no longer rely on raw nodeOptionsObjects only as the primary pattern. Instead, introduce an explicit and extensible node creation system such as:

    manager.createNode(...),

    manager.nodeManager.create(...),

    or factory-based creation like new LavalinkNode(...) / new NodeLinkNode(...) followed by registration.

Requirements

    Node creation should be class-driven or factory-driven, not just object-driven.

    Raw config objects may still exist for convenience/migration, but they should not be the main internal abstraction.

    The API should support custom node subclasses without hacks.

Desired outcome

The system should feel intentional and type-safe, e.g.:

ts
const node = manager.createNode(new LavalinkNode({...}));
manager.registerNode(node);

or

ts
const node = manager.nodeManager.create(LavalinkNode, {...});

2. All major classes must be extensible

It should be possible to extend all major core classes cleanly:

    LavalinkManager

    NodeManager

    FilterManager

    LavalinkNode / Node

    Player

Requirements

    Replace hardcoded concrete-class assumptions with generic abstractions and injected constructors.

    Avoid internal code that assumes exact base classes everywhere.

    Document the extension points clearly.

    Support subclassing without requiring users to fork internals.

Example direction

A user should be able to do things like:

ts
class CustomPlayer extends Player {
public myMethod() {}
}

class CustomManager extends LavalinkManager<{
player: CustomPlayer;
}> {}

3. Strongly typed manager customization

The main LavalinkManager class should be configurable with custom properties, custom types, and IntelliSense-friendly generics. The goal is that users can attach typed custom metadata and override internal class mappings without losing developer experience.
Requirements

    Allow generic typing for manager-level custom data.

    Allow generic typing for node/player/filter/rest implementations.

    Preserve IntelliSense across manager, manager.getNode(), players, and related collections.

    Avoid any-heavy extension patterns.

Example goal

ts
interface CustomManagerContext {
metrics: MetricsCollector;
customCache: Map<string, unknown>;
}

const manager = new LavalinkManager<CustomManagerContext, CustomPlayer, CustomNode>({
context: {
metrics,
customCache
}
});

4. Export prebuilt node classes

The package should provide ready-to-use exported node classes such as:

    LavalinkNode

    NodeLinkNode

Requirements

    These should be official exports.

    Each should encapsulate backend-specific behavior cleanly.

    The public API should make it obvious which class to use for which server flavor.

    Additional custom node classes should be easy to implement by extending a shared base abstraction.

5. Remove per-node nodeTypeChecks

There should not be repeated nodeTypeChecks or backend-branching logic scattered inside each node implementation.
Requirements

    Move node capability handling into one of:

        class polymorphism,

        capability descriptors,

        strategy objects,

        transport adapters,

        or backend-specific overrides.

    The base node should expose a unified contract.

    Backend differences should be isolated, not leaked through the whole codebase.

Desired outcome

No more code like:

ts
if (this.nodeType === "lavalink") ...
if (this.nodeType === "nodelink") ...

inside unrelated logic. 6. Eliminate circular references

The current architecture appears to suffer from circular object/class chains like:
LavalinkManager -> NodeManager -> LavalinkManager -> ManagerUtils -> LavalinkManager

These circular references should be removed entirely and replaced by cleaner dependency boundaries.
Requirements

    No class should keep unnecessary backreferences to multiple layers of the system.

    Utilities should be stateless or dependency-injected.

    Shared services should be extracted into dedicated modules instead of being reached through deep manager chains.

    Prefer composition over mutual ownership.

Desired outcome

A class should only know about the minimal collaborators it actually needs. 7. Migrate fully to Bun-native runtime support

The project should move fully toward Bun-native operation and avoid reliance on:

    ws

    node:* packages

    Node-specific runtime assumptions that require extra compatibility imports

Requirements

    Use Bun-native WebSocket / fetch / Request / Response primitives where possible.

    Prefer web-standard APIs over Node-only APIs.

    Keep the package ESM-first and runtime-portable where feasible.

    Remove unnecessary runtime dependencies that only exist because of legacy Node assumptions.

Important nuance

If complete runtime neutrality is possible, that is even better than Bun-only. But Bun-native support should be a primary target. 8. Design for high performance

This refactor should explicitly optimize for high performance, low overhead, and scalable internals. The current project already emphasizes memory efficiency and a developer-friendly API, so the rewrite should preserve that spirit while improving internals.
Requirements

    Reduce allocation-heavy patterns in hot paths.

    Minimize redundant validation and branching.

    Use efficient collection/storage patterns for nodes/players.

    Avoid deep wrapper chains where direct calls are sufficient.

    Ensure event dispatch and state updates remain lightweight.

    Keep transport, REST, and player state management separated enough to avoid unnecessary coupling.

9. Add a dedicated node REST class

Each node should have a dedicated REST abstraction, and it should be accessible via the manager in a natural way such as:

ts
manager.getNode().rest.functionCall(...)

The getNode() function should optionally accept selectors/options to choose a specific node.
Requirements

    Each node owns or exposes a rest instance.

    manager.getNode() should support flexible selection:

        by id,

        by region,

        by strategy,

        by custom predicate,

        or default best-node resolution.

    The REST layer should be typed and extensible.

    REST responsibilities should not be mixed directly into the node core class if they can be isolated cleanly.

Example goal

ts
manager.getNode({ id: "main" }).rest.updateSession(...)
manager.getNode({ strategy: "leastLoad" }).rest.getStats()

10. Async-capable chainable API design

All node/player/etc. methods should support chaining even when async behavior is involved.
Requirements

Design a fluent API that remains ergonomic without becoming misleading.

Possible acceptable directions:

    returning this synchronously while queueing internal async work,

    returning a chainable command builder,

    returning a Promise<this> with helper chaining utilities,

    or introducing a dedicated fluent executor abstraction.

Goal

This should feel natural for operations like:

ts
await player
.connect()
.setVolume(80)
.play()
.setPaused(false);

If true async chaining on instance methods becomes awkward in TypeScript, design an alternative fluent API that preserves readability and type safety.
Expected refactor strategy

Please propose the major-release redesign with the following deliverables:
A. New architecture proposal

Provide a full architecture proposal covering:

    module boundaries,

    dependency direction,

    base abstract classes/interfaces,

    factory/registry system,

    manager/node/player/rest relationships,

    and how extensibility is achieved without circular references.

B. Public API redesign

Show the new public API for:

    manager creation,

    node registration/creation,

    typed manager customization,

    node selection,

    REST access,

    player creation and chaining,

    and subclass extension.

C. Migration plan

Provide a migration plan from the current API to the new one:

    what breaks,

    what gets renamed,

    what can be shimmed,

    what deprecated compatibility helpers could exist for one major cycle.

D. TypeScript design

Explain the generic typing model in detail:

    manager context typing,

    class override typing,

    custom node/player typings,

    collection typings,

    REST typings,

    and IntelliSense preservation.

E. Bun-native runtime plan

Show exactly how the library can drop or reduce:

    ws,

    node:*,

    and Node-only APIs,
    while preserving required Lavalink transport features.

F. Performance plan

List concrete performance improvements:

    hot-path simplifications,

    object lifecycle improvements,

    event handling strategy,

    caching/memoization where appropriate,

    and reduced internal indirection.

G. Example implementation sketch

Provide example code for:

    extending LavalinkManager,

    defining a custom Player,

    exporting and using LavalinkNode / NodeLinkNode,

    accessing manager.getNode().rest,

    and using the new chainable API.

Design principles

The refactor should follow these principles:

    Extensibility first

    Composition over circular ownership

    Polymorphism over nodeTypeChecks

    Web-standard/Bun-native runtime APIs

    Strong TypeScript ergonomics

    Fluent developer experience

    High performance in hot paths

    Minimal magic, predictable abstractions

    Clean separation of transport, REST, state, and orchestration

Non-goals

Avoid solutions that:

    keep the current architecture but only rename things,

    add more internal manager backreferences,

    depend on any for extensibility,

    hide async behavior in confusing ways,

    or preserve node-type branching throughout the codebase.

Final output format I want from you

Please respond with:

    a proposed new architecture,

    the new class/type hierarchy,

    the public API redesign,

    migration examples from the current API,

    Bun-native implementation notes,

    performance recommendations,

    and example TypeScript code for the new design.

Shorter version

If you want a tighter version to paste into Cursor/Claude/GPT, use this:

    I want to design a new major release of lavalink-client with a full architectural refactor. The current project is a TypeScript Lavalink v4 client centered around LavalinkManager, nodeManager, players, queues, and node config objects, and its current docs indicate the client is not really extendable. I want the new version to make extensibility a first-class goal.

    Requirements:

        Stop using raw node option objects as the main node creation model; use class/factory-based node creation instead.

        Make all major classes extensible: LavalinkManager, NodeManager, FilterManager, Node, Player.

        Support typed customization of LavalinkManager with custom properties, generics, and full IntelliSense.

        Export ready-made node classes like LavalinkNode and NodeLinkNode.

        Eliminate scattered nodeTypeChecks inside node logic through polymorphism/capability abstractions.

        Remove circular dependencies/backreferences between manager/classes/utilities.

        Migrate to Bun-native/web-standard runtime APIs and avoid ws and node:* dependencies where possible.

        Optimize the internals for high performance.

        Add a dedicated rest class on nodes so users can do manager.getNode(...).rest.method().

        Allow fluent chaining for node/player methods, including async workflows.

    Please propose the full new architecture, class hierarchy, public API, migration plan from the current version, TypeScript generic model, Bun-native strategy, performance improvements, and example code.
