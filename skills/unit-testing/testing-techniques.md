# Techniques: How to Catch Test Flaws Before Running

Four practical techniques to spot problems when writing tests—before you run them.

---

## Technique 1: Read Top-to-Bottom Like a Debugger

Trace through your test step-by-step, numbering each operation.

```
FUNCTION test_sequence_matching()
    matcher = create_matcher(patterns: ["ab", "cd"])
    
    // Operation 1: Match first character
    ASSERT matcher.match('a') EQUALS false   ✓
    // Operation 2: Match second character
    ASSERT matcher.match('b') EQUALS true    ✓
    // Operation 3: Match third character
    ASSERT matcher.match('c') EQUALS false   ✓
    // Operation 4: Match fourth character
    ASSERT matcher.match('d') EQUALS true    ✓
END FUNCTION
```

### Why This Works

Numbering forces you to track which operation you're actually testing. Many test flaws come from losing track of call order—you think you're testing call 1 & 2, but assertions are checking calls 3 & 4.

### When to Use

Use this technique when:
- You're testing a sequence of operations
- The same method is called multiple times with different inputs
- Results depend on the order of execution
- State changes between calls

---

## Technique 2: Add Explicit State Comments

Write what state you're in before each assertion.

```
FUNCTION test_accumulator()
    counter = create_counter()
    
    // Initial state: no items
    ASSERT counter.increment() EQUALS 1  // After first increment
    ASSERT counter.increment() EQUALS 2  // After second increment
    
    // Check: does reset clear state?
    counter.reset()
    ASSERT counter.get() EQUALS 0  // After reset, back to 0
    
    // Verify state after reset works correctly
    ASSERT counter.increment() EQUALS 1  // Increment still works post-reset
END FUNCTION
```

### Why This Works

State comments force you to think about what state you're actually in. This catches assumptions about stateful objects that contradict each other.

Example: If you assume `increment()` always returns the same value, comments make that assumption explicit—then you'll catch the flaw before running.

### When to Use

Use this technique when:
- Testing stateful objects (ones that change behavior based on internal state)
- Testing sequences where state transitions matter
- Testing reset/initialization functions
- Testing methods called multiple times where behavior changes

---

## Technique 3: Separate Setup, Action, and Assertion

Make test structure visually clear with distinct phases.

```
FUNCTION test_single_operation()
    // SETUP: Create object and initialize state
    matcher = create_matcher(patterns: ["hello"])
    
    // ACTION: Perform the operation being tested
    result = matcher.match('h')
    
    // ASSERTION: Verify the result
    ASSERT result EQUALS false
END FUNCTION
```

### Why This Works

When setup and assertions are visually separated, contradictions jump out immediately. If you find yourself mixing setup and assertions together, you'll spot it because they're in the wrong section.

This also clarifies your test intent: setup prepares the condition, action tests it, assertion verifies it.

### When to Use

Use this technique when:
- Writing simple, single-operation tests
- Testing a specific behavior in isolation
- You want to make test intent crystal clear
- For beginners learning test structure

---

## Technique 4: The "What Am I Testing?" Comment

Before writing assertions, write a comment describing your goal.

```
FUNCTION test_list_removal()
    list = create_list()
    list.add('apple')
    list.add('banana')
    list.add('orange')
    
    // Goal: Verify remove() deletes the correct item and returns true
    ASSERT list.remove('banana') EQUALS true    // Item exists, should succeed
    ASSERT list.size() EQUALS 2                 // Size decreased by 1
    ASSERT list.contains('banana') EQUALS false // Item is gone
    
    // Goal: Verify remove() returns false for non-existent items
    ASSERT list.remove('banana') EQUALS false   // Already removed, should fail
END FUNCTION
```

### Why This Works

Goal comments force you to articulate what you're testing before you write assertions. If your goal doesn't match your assertions, you've caught the flaw **before running**.

This is preventive: you can't write contradictory assertions if your goal is explicit.

### When to Use

Use this technique when:
- Testing complex operations with multiple side effects
- Testing both success and failure paths in one test
- Testing operations that should have multiple outcomes
- You want to be explicit about test intent upfront

---

## Summary Table

| Technique | Best For | Key Benefit |
|-----------|----------|------------|
| Numbering Operations | Multi-step sequences | Prevents losing track of call order |
| State Comments | Stateful objects | Forces explicit state thinking |
| Separate Phases | Simple, focused tests | Makes intent crystal clear |
| Goal Comments | Complex operations | Catches contradictions before running |
