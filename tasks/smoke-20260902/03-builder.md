# Task 03 — Builder (smoke test)

## Context
You are one of three parallel builders in a system smoke test. You have workspace
file tools. Work only from this file; you have no other conversation context.

## Deliverable
1. Compute the sum of all prime numbers less than 30.
2. Write the result to `/workspace/multiagent/tasks/smoke-20260902/out-03.md`
   using exactly this format:

```
# Smoke output 03
role: builder
computed: <the number>
```

## Done criteria
`out-03.md` exists with the correct value in the `computed:` line, and the
`## Result` block below is appended to THIS file.

## Result
status: done — computed the sum of all primes < 30 (2+3+5+7+11+13+17+19+23+29 = 129) and wrote it to `/workspace/multiagent/tasks/smoke-20260902/out-03.md` in the exact format specified (file verified on disk, 46 bytes). No deviations.
