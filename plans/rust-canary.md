# Plan: rust-canary

project: rust-canary

## C1: Canary feature

area: canary
depends: seed

One tiny feature that walks a task through bosun (claim, PR, `bosun pr`, risk-check,
review-verdict) on the hull template-rust canary.

### Tasks

- T1: `hello --shout` prints the greeting in upper case
  touch: crates/cli/src/**, crates/cli/tests/**
  size: 30

### Done when

- UC-C1-1: `cli hello --name Ada --shout` prints `HELLO, ADA`, and `--json` reports the same greeting.

## C2: Trim the name

area: canary
depends: C1

Trial task for bosun 0.2.2 on this repo: `hello` trims the whitespace around `--name`.
It came in through `bosun intake` (#15, plan PR #16); this plan task (C2/P) makes T1 ready.

### Tasks

- T1: `hello` trims the whitespace around `--name`
  touch: crates/cli/src/**, crates/cli/tests/**
  size: 20

### Done when

- UC-C2-1: `cli hello --name "  Ada "` prints `hello, Ada`, and `--json` reports the same greeting.
