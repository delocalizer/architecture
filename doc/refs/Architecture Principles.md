# Architecture Principles
Notes and highlights from Juval Löwy's *Righting Software*.

## Chapter 1 — The Method

_The Method_ = System Design + Project Design

## Chapter 2 — Decomposition

### Clients Bloat and Coupling

* Functional decomposition often leads to flattening of the system hierarchy. In particular, if service call orchestration logic ends up in the clients, they become bloated with business logic of the system.
* Hallmarks of bad design are when any change to the system affects the client, or when you have to duplicate business logic across multiple clients.

### Services Bloat and Coupling

* If on the other hand you allow the services to call each other, _they_ become coupled to each other and to the order of the functional calls, and each bloats with code to handle exceptions in the other components.

### When to Use Functional Decomposition?

* During requirements discovery. At this stage of the project you can drive functional decomposition to a very fine level, arranging requirements in a tree and identifying redundancies and mutually exclusive requirements. But don't map this to the design!

### Testability and Design

* Unit tests are not that useful (compared to integration and regression tests). But with tightly coupled design, that's often all you can do.
* With loosely-coupled designs, integration and regression testing is much easier.

### Volatility-Based Decomposition

* Identify areas of potential change (not simply variability) and encapsulate those areas into services or system building blocks.

## Chapter 3 — Structure

### Required Behavior

* A __use case__ is an expression of required behavior — how the system is required to go about accomplishing some work and adding value to the business. Use cases tend to be verbose and descriptive. They can describe end-user interactions, or the system's interactions with other systems, or back-end processing. The bulk of the system is usually 'below the waterline' i.e. not user-facing; and you should produce use cases for it as well.
* Ideally, activity diagrams should be used to represent all use cases, but at the very least they _must_ be used whenever the logic becomes as complex as a nested *if*.

### Layers

Layers allow you to layer encapsulation. Each layer encapsulates its own volatilities from the layers above and below; services inside layers encapsulate volatility from each other.

![Typical layers in The Method](Righting%20Software%20Figure%203-4.png)


### Services

The preferred way of crossing layers is by calling **services**. You can certainly benefit from volatility based decomposition even using regular classes but relying on services gives distinct advantages:

* Scalability
* Security — service oriented platforms treat security as a first class aspect. All calls — not just those from client to service — require authentication and authorization. 
* Throughput and availability (queues)
* Responsiveness (throttling, buffering)
* Reliability
* Consistency (eventual)
* Synchronization

### Client Layer

The top layer in the architecture. Could be user-facing but also includes service-level interaction. Most service-oriented technologies are very strict about the type of data they allow over endpoints; this limits the ability to couple clients to services which is good. Clients may be: apps, command-line interfaces, web portals, APIs, admin applications, etc.

### Business Logic

This layer implements the system's required behavior as expressed in the use cases. It encapsulates volatility in the system's business logic.  **Manager** components encapsulate volatility in *sequence*; **Engine** components encapsulate volatility in *activity*.

### Resource Access

This should encapsulate not just the volatility in how you access a resource, but in the resource itself - e.g. local file system vs object storage vs remote object storage. A well-designed ResourceAccess component exposes atomic *business* verbs around a resource, not low-level implementation-specific actions like 'select', 'insert', or 'delete'.

### Resource

This contains the actual physical resources on which the system relies e.g. database, filesystem, cache, message queue.

### Naming

* Names of services must be two-part compound words written in PascalCase
* The suffix is always the service type - e.g. `Manager`, `Engine` or `Access`
* The prefix for *Managers* should be a noun associated with the encapsulated volatility
* The prefix for *Engines* should be a noun describing the encapsulated activity
* The prefix for *ResourceAccess* should be a noun associated with the *Resource*.
* Gerunds should only be used as a prefix with *Engines*. Elsewhere they smell of functional decomposition.
* Atomic business verbs should not be used in a prefix for a service. These should be confined to operation names in contracts interfacing with the resource access layer.

### The Four Questions

* Who (*Clients*)
* What (*Managers*)
* How (*Engines*, *ResourceAccess*)
* Where (*Resource*)

Asking and answering the four questions is useful at both ends of the design effort. Are all your *Clients* "who" with no trace of "what" in them? Are all the *Managers* "what" without a trace of "who" and "where" in them?

### Key Observations

* Volatility decreases top-down. Clients are very volatile. Managers do change but not as often as clients. Resources are the least volatile.
* Reuse increases top-down. Clients are hardly ever reusable. Resources must be the most reusable element in any well-designed system.
* Almost-Expendable *Managers*. If your response to a request to change a *Manager* is to fight the change then it is too expensive. It is too big and complex, likely due to functional decomposition. Completely expendable *Managers* are always a design flaw and often only exist to satisfy design guidelines without any real need.

### Incremental  Construction

If the system is relatively simple and small, the business value — that is the execution of the use-cases — will likely require all components. There is no point releasing independent *Engines* or *ResourceAccess* components. The right approach is:

***Design iteratively, build incrementally***

Not only is building iteratively wasteful and difficult, the intermediate iterations have no business value. This argues against some dev methodologies e.g. Agile in its more extreme forms, in which teams build their own pieces completely independently, and iterate on the design and integration cycles.

The ability to build incrementally over time within the confines of the architecture is predicated on the architecture remaining constant and correct. This is only possible with volatility-based decomposition.

### Note on Services

As a general principle, in a well-designed system you should never use the same communications mechanism both internally and externally. 

External services can use HTTP and REST; internal services should rely on fast, high-performance comms channels. e.g. TCP/IP, pipes, sockets, message queues etc.

### Open vs Closed Architecture

In an **open architecture** any component can call any other component regardless of the layer in which the components reside. Up, down, sideways, whatever. This is very flexible but introduces a significant amount of coupling. 

In a **closed architecture** you disallow up- and sideways calls. This promotes decoupling by trading flexibility for encapsulation.

In a **semi-closed** architecture you allow calling more than one layer down, e.g. for performance reasons.

In a closed architecture, *Utilities* pose a challenge because its services cut across all layers of concern. In short there is no good location for *Utilities* among the layers of a closed architecture, so they are represented as orthogonal to all the layers. Don't abuse this! Put only truly generic services in *Utilities*.

### Call Rules

* Make calls only to one layer down.
* *Managers* and *Engines* may both directly call *ResourceAccess*.
* *Managers* may directly call *Engines*.
* *Managers* may queue calls to other *Managers*. If you view the queue call as a *ResourceAccess* and the listener as a *Client*, you're not even technically calling sideways here.

 ### Design "Dont's"
* Clients do not call multiple *Managers* in the same use case. Doing so implies tightly-coupled *Managers* that no longer represent separate families of use cases or slices.
* *Clients* do not call *Engines*. The only entry point to the business logic layer are *Managers*.
* *Managers* do not queue calls to more than one *Manager* in a single use case. The need to have two or more *Managers* respond to a queued call is a sign that more may need to — so you should use a `Pub/Sub` *Utility* instead.
* *Engines* do not receive queued calls. *Engines* are utilitarian and exist to execute a volatile activity for a *Manager*. A queued call by definition executes independently from anything else but *Engines* activities have no independent meaning on their own.
* *ResourceAccess* services do not receive queued calls, for similar reasons. *ResourceAccess* services exist to service a *Manager* or *Engine* and have no meaning on their own. Accessing a *Resource* independently from anything else in the system makes no business sense.
* *Clients* do not publish events. Events represent changes to state about which *Clients* or *Managers* may want to know but a *Client* has no need to notify itself or other *Clients*.
* *Engines* do not publish events. Publishing an event requires noticing and responding to system state change and is typically a step in a use case executed by a *Manager*. An *Engine* performing an activity has no way of knowing much about the activity context or use case.
* *ResourceAccess* services do not publish events. 
* *Resources* do not publish events.
* Corollary: only *Managers* publish events.
* *Engines*, *ResourceAccess* and *Resources* do not subscribe to events. Processing an event is almost always the start of some use case so it must be done in a *Client* or *Manager*. The *Client* may inform the user about something and the *Manager* may execute some backend behavior.
* *Engines* never call each other. In a volatility-based decomposition an *Engine* should already have encapsulated everything to do with an activity.
* *ResourceAccess* services never call each other. Note that a 1:1 mapping between *ResourceAccess* and *Resource* is **not** required. Often two or more *Resources* must be logically joined to implement some business verb. A single *ResourceAccess* service should perform the join rather than inter-*ResourceAccess* services calls.

### Symmetry

Strive for symmetry. Lack of it is a design smell. For example, a *Manager* implements four use cases, three of which publish an event with *Pub/Sub* service and fourth does not — why is that? What are you missing or overdoing?

## Chapter 4 — Composition

Design by decomposition produces a static layout, like a blueprint. How do we know the composition of the elements at runtime will adequately satisfy requirements? Design validation and composition are intimately related. 

### Requirements and Changes

* **Requirements change. Accept it — that's what requirements do.** This is great actually because it means software engineers are never out of a job!
* But many resent change. The reason is, they've designed against requirements and so their design must change when the requirements change. That's painful, destructive and expensive.

### Design Prime Directive

* **Never design against the requirements**
* Any attempt of gathering the *complete* set of requirements at the start of a project and designing against them is futile.

### Composable Design

The goal of any system design is to be able to satisfy all use cases. Now, and in the future. The hallmark of bad design is that when requirements change, the design has to change as well.

#### Core Use Cases

* Not all use cases are distinct and unique. Most are variations of others. Even the main behavior has numerous permutations — the normal case, the incomplete case, the error case, the locale-ized case.
* There are only **two** types of use cases: **core** use cases, and all the rest.
* **core use cases** represent the essence of the business of the system. While the system may have hundreds of use cases, the saving grace is that it will have only a handful of core use cases. Two or three are common, and empirically, the number rarely exceeds six. Look at a single-page marketing brochure for the system and count the number of bullets. Likely no more than three.
* Core use cases will hardly ever be explicitly presented in the requirements documentation no matter how refined that may be. The small number doesn't mean they're easy to identify, nor easy to agree on with others. Even flawed requirements will however contain the core use cases because they represent the essence of the business.
* Core use cases will almost always be an abstraction of other use cases and may even require a new terminology to differentiate it.
* The whole point of requirements analysis as an architect is to *identify the core use cases*, often via some iterative process.

### The Architect's Mission

* Your mission as an architect is to identify the smallest set of components that you can put together to satisfy all the core use cases. Since all the others are variations on the core ones, regular use cases simply represent a different *interaction* between the components, not a different decomposition. Now when requirements change, your design does not.
***Note***
This observation is about the decomposition into components, not about the implementation of code within components. Integration code mostly resides in *Managers*, and integration code likely changes as a result of requirements change — so *Managers* may change. But this is not an architectural change, it's an implementation change.

* Löwy  calls this **composable design**. A composable design does not aim to satisfy any use case in particular. You do not avoid targeting any particular use case because the use cases were incomplete, faulty, or contradictory, but because *they will change*.

#### Architecture Validation

* Since the goal of any system is to satisfy all requirements, composable design enables something else: *design validation*. Once you can produce an interaction between your services for each core use case you have produced a valid design.
* The act of validating the design can be as straightforward as producing diagrams demonstrating the interactions between the components of the architecture that support the use cases: **call chain** diagrams.
* A call chain demonstrates the interaction between components required to satisfy a particular use case. solid arrows represent synchronous calls; dashed arrows for queued calls. 
* For many cases call chain diagrams are sufficient, but downsides are that they have no notion of order, nor of duration.
* For more complex cases a **sequence diagram** may be used. These are extremely useful for helping define interfaces, methods, and even parameters. If you're going to produce them for detailed design you may as well produce them first for design validation, albeit with fewer details.

![Call chain and sequence diagrams](Righting%20Software%20Figures%204%20-%201,2.png)

### Smallest Set

* Your mission as an architect is not just to identify a set of components that when put together will satisfy the use cases, but the *smallest set* of such. 
* In general you should produce a design that *minimizes the amount of work involved in detailed design and implementation*. Less is more. 
* The smallest set of services is typically of the order of magnitude of 10. e.g. between 5 and 20.
* You cannot validate architectures that have a single component, or hundreds of components. A single large component by definition does everything, and one component per use case also supports all use cases by definition.
* You may spend weeks or even months identifying core use cases and areas of volatility, but the actual design process to produce a valid design from that point on will be of the order of days.

### There is No Feature

* **Features are always and everywhere aspects of integration, not implementation**. This is a universal design rule which governs design and implementation of all systems, not just software systems. (This has implications not just for system design, but for project design — milestones are integration points)

### Handling Change

* Fighting change is tantamount to killing the system. Living systems are systems that customers use, and dead systems are systems that customers do not use.
* Part of responding to change is responding *quickly*.
* The trick to addressing change is not to fight, postpone, or punt it. The trick is containing its effects. And the only really good way to do this proactively, by making good design choices that expect and allow for change.
* Recall that *Managers* should be almost expendable. This enables you to absorb the cost of the change and contain it. Furthermore, the bulk of effort in any system typically goes into the services the *Manager* uses:
  * Implementing *Engines* is expensive. *Engines* represent business activities.
  * Implementing *ResourceAccess* is non-trivial. Identifying the atomic business verbs, translating them to access methods for *Resources*, and exposing them as a *Resource*-neutral interface takes effort.
  * Designing *Resources* that are scalable, reliable, performant and reusable is expensive. e.g. designing data contracts, schemas, cache access policies, partitioning, replication, connection management, timeouts, locks, indexing, normalization, message formats, transactions, delivery failures, poison messages, etc.
  * Implementing *Utilities* always requires top skills — e.g. `Security` component.
  * Designing superior user experience or convenient APIs is expensive. 
* But when a change happens to the *Manager* you get to salvage and re-use all the effort that went into the other services, and focus only on reintegration in the *Manager*.

## Appendix C — Design Standard

### Directives

1. Avoid functional decomposition
2. Decompose based on volatility
3. Provide a composable design
4. Offer features as aspects of integration, not implementation
5. Design iteratively, build incrementally
6. Design the project to build the system
7. Drive educated decisions with viable options that differ by schedule,  cost, and risk
8. Build the project along the critical path
9. Be on time throughout the project

### System Design Guidelines

1. Requirements
    * Capture required behavior, not required functionality
    * Describe required behavior with use cases
    * Document all use cases that contain nested conditions with [activity diagrams](https://en.wikipedia.org/wiki/Activity_diagram)
    * Eliminate solutions masquerading as requirements
    * Validate the system design by ensuring it supports all *core* use cases
2. Cardinality
    * Avoid more than 5 *Managers* in a system without subsystems
    * Avoid more than a handful of subsystems
    * Avoid more than 3 *Managers* per subsystem
    * Strive for a golden ratio of *Engines* to *Managers*
    * Allow *ResourceAccess* components to access more than one *Resource* if necessary
 1. Attributes
     * Volatility should decrease top-down[^1]
     * Reuse should increase top-down
     * Do not encapsulate changes to the nature of the business
     * *Managers* should be almost expendable
     * Design should be symmetric
     * Never use public communications channels for internal system interactions[^2]
 1. Layers
     * Avoid open architecture
     * Avoid semi-closed/semi-open architecture
     * Prefer a closed architecture
         - Do not call up
         - Do not call sideways (except queued calls between *Managers*)
         - Do not call more than one layer down
         - Resolve attempts at opening architecture by using queued calls or async event publishing
      * Extend the system by implementing subsystem
3. Interaction rules
    * All components can call *Utilities*[^3]
    * *Managers* and *Engines* can call *ResourceAccess*
    * *Managers* can queue calls to another *Manager*
4. Interaction **dont's**
    * *Clients* do not call multiple *Managers* in the same use case
    * *Managers* do not queue calls to more than one *Manager* in the same use case
    * *Engines* do not receive queued calls
    * *ResourceAccess* components do not receive queued calls
    * *Clients* do not publish events
    * *Engines* do not publish events
    * *ResourceAccess* components do not publish events
    * *Resources* do not publish events
    * *Engines*, *ResourceAccess*, and *Resources* do not subscribe to events
 
[^1]: Relatedly, recall "*D is for dependency inversion* " from SOLID principles: at the service level, high-level modules should not depend on low-level modules, and both should depend on abstractions.
[^2]: Note that he doesn't argue against SOA or microservices. Just that not everything should be talking over HTTP. See also ESB vs iPaaS
[^3]: e.g. **Logging**, **Message Bus**, **Security** components.

*[SOA]: Service Oriented Architecture
*[ESB]: Enterprise Services Bus
*[iPaaS]: Integration Platform As A Service



<!--stackedit_data:
eyJwcm9wZXJ0aWVzIjoidGl0bGU6IEFyY2hpdGVjdHVyZSBQcm
luY2lwbGVzXG5cbmV4dGVuc2lvbnM6XG4gIGVtb2ppOlxuICAg
ICMgRW5hYmxlIGVtb2ppIHNob3J0Y3V0cyBsaWtlIDopIDotKF
xuICAgIHNob3J0Y3V0czogdHJ1ZVxuIiwiaGlzdG9yeSI6Wy0x
NjM3OTM1ODkyLC0xODU5ODgzNTM4LC0xNzczNzg3NzgzLC0zMD
k4MTM3MjMsNzI3ODY5MTkxLC0xMzE3Njg0NTAsLTMyMDU4MzQx
MiwtMTIxMDc3NDk3OCwxMTMzODY5ODA5LDc2MDcwMTcyMiwtMz
A4OTUwNTgsMTg0Njk3OTYzMywxNzMwODAxMzE0LDM3MjM5Mjg0
NSwtNDg3ODk0NTQ2LC01NzAyMTk4NDgsLTcyMzE0Njk4MywxNj
Q0ODQ2NTE3LC0xMzUzMjgwMDIyLDExMDYzNjUyODddfQ==
-->
