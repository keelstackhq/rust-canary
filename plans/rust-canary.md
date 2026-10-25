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
