# Common Testing Mistakes

Three patterns that cause most test failures. Each is preventable with proper analysis and techniques.

---

## Mistake 1: Contradictory Assertions

**Problem:** Same operation, different expected results

```
FUNCTION test_stream_matching()
    matcher = create_matcher(patterns: ["word"])
    
    ASSERT matcher.match(character) EQUALS false  // ❌ Call 1: returns false
    ASSERT matcher.match(character) EQUALS true   // ❌ Call 2: returns true?
END FUNCTION
```

**Issue:** You called `match(character)` twice. Same input, different assertions = one will fail.

### The Fix

Trace what you're actually testing:

```
FUNCTION test_stream_matching()
    matcher = create_matcher(patterns: ["word"])
    
    // Setup: process characters in sequence
    ASSERT matcher.match(char_1) EQUALS false
    ASSERT matcher.match(char_2) EQUALS false
    ASSERT matcher.match(char_3) EQUALS false
    
    // Test: match should succeed after sequence
    ASSERT matcher.match(char_4) EQUALS true
    
    // Only ONE match call for char_4, matches ONE expected result
END FUNCTION
```

### Red Flags for This Mistake

- **Calling the same method twice with same inputs but different expected results** — You're testing the same operation with conflicting expectations
- **Comments saying "should return X" but assertion says "== Y"** — Your intent doesn't match your assertion

---

## Mistake 2: State Mutation in Setup vs Assertions

**Problem:** You call the method multiple times without realizing it

```
FUNCTION test_processor()
    processor = create_processor()
    processor.process(data)      // Setup: call 1
    processor.process(data)      // Setup: call 2
    
    // You think you're testing calls 1 & 2
    // You're actually testing calls 3 & 4
    ASSERT processor.process(data) EQUALS false  // Call 3
    ASSERT processor.process(data) EQUALS true   // Call 4
END FUNCTION
```

**Issue:** Setup and assertions both call the method. You lose track of which call you're testing.

### The Fix

Capture results in variables:

```
FUNCTION test_processor()
    processor = create_processor()
    
    result_1 = processor.process(data)  // Call 1
    result_2 = processor.process(data)  // Call 2
    
    ASSERT result_1 EQUALS false
    ASSERT result_2 EQUALS true
END FUNCTION
```

This makes state transitions explicit and obvious. You test exactly the calls you intended.

### Red Flags for This Mistake

- **Setup that modifies state, then assertions that modify more state** — You're calling the method in both places
- **Assertion on a method you already called in setup** — Capture results in variables instead

---

## Mistake 3: Forgetting Stateful Object Behavior

**Problem:** Assuming a method returns the same thing when called repeatedly

```
FUNCTION test_container()
    container = create_container()
    
    // What does this return? Depends on internal state!
    ASSERT container.store(value) EQUALS false
    ASSERT container.store(value) EQUALS true  // ❌ Different state now
END FUNCTION
```

**Issue:** You assumed `store()` always returns the same value. But state changes with each call—the second call has different state than the first.

### The Fix

Understand the object's behavior before writing tests:
- Does `store()` return based on container state?
- Does state change with each call?
- Do I need to trace through multiple calls?

Document your assumptions:

```
FUNCTION test_container()
    container = create_container()
    
    // store() returns true when full, false when adding
    ASSERT container.store(value) EQUALS false  // Container not full yet
    ASSERT container.store(value) EQUALS false  // Still not full
    
    // Only after reaching max capacity does it return true
    container.fill_to_capacity()
    ASSERT container.store(value) EQUALS true   // Now full, returns true
END FUNCTION
```

### Red Flags for This Mistake

- **Calling the same method twice with same inputs but different expected results** — State may have changed between calls
- **Test for stateful object without tracing state** — You haven't documented how internal state changes with each call
