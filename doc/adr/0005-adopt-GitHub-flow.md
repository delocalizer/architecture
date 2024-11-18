# 5. Adopt github flow

Date: 2024-11-18

## Status

Accepted

## Context

We would like a consistent way of working with git version control.

## Decision

We will use [GitHub flow](https://docs.github.com/en/get-started/using-github/github-flow). Long-lived branches (other than `main`) will be the exception.

## Consequences

Simplicity is the key win from GitHub Flow. Deployability and CI/CD are more streamlined. Merge conflicts are avoided when there are few or one long-lived branch. There may be specific projects or scenarios where a more complex strategy is required; these can be approved on an ad-hoc basis.
