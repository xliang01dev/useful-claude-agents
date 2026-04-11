---
name: unit-testing
description: Use when writing tests for existing code - analyze implementation, write modular tests, validate correctness without changing code
---

# Writing Tests Correctly

## Overview

Writing a test that **looks right** but is logically flawed wastes time and gives false confidence. The only way to know if a test is correct is to **run it and watch it pass**. If you wrote the test without running it, you don't know if it works.

**Core principle:** Tests are code. They must be verified before considering them complete, just like implementation code must be tested.

## How to Use

```
ANALYZE CODE FIRST. WRITE TESTS THAT VALIDATE CORRECTNESS.
```

1. **Understand** what the code does (signatures, behavior, state, edge cases)
2. **Write** comprehensive modular tests based on actual code behavior
3. **Run** all tests together
4. **If tests fail:** Test assumptions are wrong—fix tests, never the code
5. **Never modify source code** to make tests pass

## When to Use

**STOP and use this skill when:**
- You just wrote test functions but haven't run them yet
- You're about to mark tests as "complete" or "done"
- You wrote tests without tracing through expected behavior
- Tests involve stateful objects (buffers, queues, objects with internal state)

**When NOT to use:**
- You've already run the tests and they pass
- You're debugging failing tests (that's testing, not writing)

## Core Principles

Follow the principles in **testing-principles.md** before writing any tests. These prevent most test flaws before they occur.

## Avoiding & Catching Test Flaws

**Common test mistakes** can be prevented with proper code analysis and caught before running. See **testing-mistakes.md** for detailed explanations, fixes, and red flags.

**Practical techniques** help you spot problems when writing tests—before you run them. See **testing-techniques.md** for detailed examples and when to use each.

## Red Flags - Stop If You See These

**Design & Verification Issues:**
- **You haven't checked function signature and types** → STOP. Read parameter types, return type, error handling first.
- **Function behavior doesn't match its signature** → Code is inconsistent. Document what it actually does before testing.
- **Test name has "and" or "plus" in it** → Split into separate tests. One behavior per test.
- **Test has more than 2-3 assertions** → Probably testing multiple behaviors. Split it.
- **Test setup is longer than the test itself** → Setup is too complex. Simplify or extract helper.

**Execution Issues (Critical):**
- **You've written tests but haven't run them yet** → STOP and run them NOW
- **You're about to move to the next task without running tests** → STOP and run them first

## Implementation Checklist

- [ ] **Check alignment with testing-principles.md** - After writing unit tests, review if your tests follow the principles
- [ ] **Verify tests align with principles on error** - If tests fail, check if they violate any principle in testing-principles.md
- [ ] **Audit for principle violations** - Scan existing tests to ensure none violate testing-principles.md
- [ ] **Avoid common flaws** - Review testing-mistakes.md to prevent contradictions and state issues
- [ ] **Run all tests together** - Execute your complete test suite at once
- [ ] **If tests fail** - Analyze with red flags, involve user in diagnosis. Never assume test is wrong or fix code.
- [ ] **Mark tests complete** - Only after running and passing verification