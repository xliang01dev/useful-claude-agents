# Core Testing Principles

Foundational knowledge for writing correct tests. These principles prevent most test flaws before they occur.

---

## Never Change Source Code

**When writing tests, you NEVER modify the source code you're testing. Ever.**

If a test fails, it's a flawed test—fix the test assumptions, not the code. Your job is to write correct tests for code that already exists. That's it.

---

## Verify Function Signatures & Types Before Testing

**Read and understand what the code promises before writing tests:**

- **Check function name** - Does it clearly describe what it does?
- **Check parameter types** - What types are accepted? Are they documented?
- **Check parameter defaults** - What's the default behavior if optional?
- **Check return type** - What does it return? Null? Error? Promise?
- **Check error handling** - Does it throw? Return error objects? Silent fail?
- **Verify consistency** - Does the code match what the signature promises?

If the function signature is unclear or inconsistent with its behavior, that's the first problem to surface.

---

## Understand Code Behavior

**Read the actual implementation to understand what it does:**

- **Read the code** - Line by line, understand the logic flow
- **Read the comments** - Developers document assumptions, edge cases, and behaviors there
- **What does the function/method do?** - Purpose and overall behavior
- **What are its inputs and expected outputs?** - Range of values, null handling, error cases
- **What state does it manage?** - Does calling it multiple times produce different results?
- **What edge cases are handled?** - Boundary conditions, special cases mentioned in comments
- **Check for undocumented behaviors** - TODOs, FIXMEs, known limitations

**Pay special attention to comments** - They reveal:
- Why code works a certain way (not just what it does)
- Limitations or gotchas developers discovered
- Expected behavior that isn't obvious from code
- State assumptions that affect testing
- Type constraints and edge cases

**Example:** If you don't know that `store()` returns `true` only when full (something a comment would clarify), you'll write assertions expecting it to always return the same value. Test flaws stem from assumption mismatches, not code mistakes.

---

## Analyze Code First, Then Write Tests

Test flaws come from incomplete code analysis. Never write tests without fully understanding what code does, how it behaves, and what state it manages. This prevents 90% of test contradictions and flawed assumptions.

---

## Write Modular Tests: One Behavior Per Test

**Break down complexity into distinct test cases:**

Each test should validate ONE behavior or scenario. Don't combine multiple conditions, edge cases, or state changes into a single test.

**Anti-pattern:**
```
FUNCTION test_user_validation()
    // Tests empty email, invalid format, duplicate account, weak password
    // Too much, impossible to debug failures
END FUNCTION
```

**Better:**
```
FUNCTION test_rejects_empty_email()
    // Only tests: missing email
END FUNCTION

FUNCTION test_rejects_invalid_email_format()
    // Only tests: malformed email
END FUNCTION
```

**Benefits of modular tests:**
- **Clear intent** - Test name matches exactly what it validates
- **Easy debugging** - Failed test immediately tells you which case broke
- **Simpler setup** - Each test only sets up what it needs
- **Better assertions** - One or two assertions per test, not five
- **Reusable** - Other tests can build on similar setup

**Rule:** If your test name has "and" in it, split it into multiple tests.
