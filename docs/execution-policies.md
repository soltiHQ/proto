---
title: Execution policies
description: Configure attempt deadlines, restart behavior, retry limits, backoff, and jitter without confusing logical and physical completion.
---

# Execution policies

`TaskSpec` controls one desired Task across one or more execution attempts. Timeout, restart, retry, and backoff fields answer different questions and must be configured together.

## Distinguish a Task generation from an attempt

```text
desired generation 7 ──► attempt 1 ──► failure
                              │
                              └── backoff ──► attempt 2 ──► timeout
                                                   │
                                                   └── backoff ──► attempt 3 ──► success
```

`metadata.generation` changes with desired state. `TaskStatus.attempt` and `TaskRunInfo.attempt` identify attempts within an observed generation.

Run history records both values. Use the pair together when correlating status and output.

## Set an attempt deadline

`timeout_ms` is the deadline for each attempt and must be greater than zero. A timeout records a logical `TIMEOUT` phase. Retry policy can still start another attempt.

A timeout or force-aborted outcome does not guarantee immediate physical exit. Code that performs external side effects needs its own idempotency, acknowledgement, or fencing protocol.

## Choose a restart policy

| Policy | Contract |
|---|---|
| `RESTART_POLICY_NEVER` | Do not restart after an attempt ends. |
| `RESTART_POLICY_ON_FAILURE` | Restart after failure or a non-zero exit. |
| `RESTART_POLICY_ALWAYS` | Restart after an attempt ends and wait `restart_interval_ms` between runs. |
| `RESTART_POLICY_UNSPECIFIED` | Zero-value sentinel; it does not select a restart policy. |

`restart_interval_ms` applies only to `ALWAYS`. Its presence is represented separately from a numeric value because the field is optional.

## Bound failure retries

`max_retries` is a failure-retry budget:

- omitted means unlimited retries under the selected restart policy;
- a positive value limits retries after failure;
- zero is invalid.

The initial attempt is not itself a retry. The schema does not turn an invalid combination into a different policy; the serving application reports validation failure.

## Configure backoff

`BackoffPolicy` defines the delay used between failure retries:

| Field | Requirement |
|---|---|
| `first_ms` | Initial delay in milliseconds; greater than zero. |
| `max_ms` | Delay cap in milliseconds; greater than or equal to `first_ms`. |
| `factor` | Finite growth multiplier; greater than or equal to `1.0`. |
| `jitter` | Randomness policy applied to the computed base delay. |

Jitter policies are:

| Policy | Delay behavior |
|---|---|
| `JITTER_POLICY_NONE` | Use the computed base exactly. |
| `JITTER_POLICY_FULL` | Choose uniformly from zero through the base. |
| `JITTER_POLICY_EQUAL` | Choose a value centered near half of the base. |
| `JITTER_POLICY_DECORRELATED` | Use the declared decorrelated strategy capped by `max_ms`. |
| `JITTER_POLICY_UNSPECIFIED` | Zero-value sentinel; it does not select a jitter policy. |

Backoff and `restart_interval_ms` are separate fields. Backoff belongs to failure retries. The restart interval belongs to the `ALWAYS` policy.

## Interpret terminal-looking phases carefully

`SUCCEEDED`, `FAILED`, `TIMEOUT`, `CANCELED`, and `EXHAUSTED` are recorded logical phases. A later reconciliation or allowed retry can start another attempt.

`EXHAUSTED` means an eligible non-fatal failure stopped because the restart policy allowed no further attempt or the retry budget was reached. `FAILED` alone does not mean the retry budget is exhausted.

Read [Task observation](task-observation.md) for the complete phase table and [Task resources](task-resources.md) for cancellation and deletion semantics.
