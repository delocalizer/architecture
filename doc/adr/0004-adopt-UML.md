# 4. Adopt uml

Date: 2024-11-11

## Status

Accepted

## Context

We want a consistent visual language across the diagram artefacts generated for the
program: from business analysis through system design and deployment. We want to
leverage concepts from existing diagram types rather than reinvent our own each time.

## Decision

We will use the appropriate diagram types from UML. For example:

* Activity Diagram to represent business processes or flow of control in an operation
* Sequence Diagram to represent message flow or interactions other over time
* Deployment Diagram to represent deployment of software components on hardware devices

Swimlanes should be used to represent jurisdictional or network boundaries, to
show which actor, component or entity is responsible for a step in a process.

## Consequences

Using the standard shape and line notations from one of the existing UML diagram types
saves time and increases the portability of the generated artefacts. The UML diagram
types are well-documented and complement each other nicely in scope. There will be a
small overhead in learning a new diagram type when it is unfamiliar. There is little
risk.
