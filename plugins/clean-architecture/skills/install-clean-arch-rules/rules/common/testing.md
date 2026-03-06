# Testing Requirements

## Minimum Test Coverage: 80%

Test Types (ALL required):
1. **Unit Tests** - Individual functions, utilities, components
2. **Integration Tests** - API endpoints, database operations
3. **E2E Tests** - Critical user flows (framework chosen per language)

## Test-Driven Development

MANDATORY workflow:
1. Write test first (RED)
2. Run test - it should FAIL
3. Write minimal implementation (GREEN)
4. Run test - it should PASS
5. Refactor (IMPROVE)
6. Verify coverage (80%+)

## Test Class Naming

Test class names MUST follow the pattern: `<ClassUnderTest>Tests`

Examples:
- `UserService` -> `UserServiceTests`
- `OrderValidator` -> `OrderValidatorTests`
- `ClaimDataMapper` -> `ClaimDataMapperTests`

## Test Method Naming

Test methods MUST follow one of these patterns:

1. **`<MethodName>_Should<Result>_When<Condition>`**
2. **`<MethodName>_Should<Result>_Given<Condition>`**

Examples:
```
GetUserAsync_ShouldReturnUser_WhenUserExists
CreateOrderAsync_ShouldThrowValidationException_WhenOrderNumberIsEmpty
ProcessClaimAsync_ShouldUpdateStatus_GivenValidStatusTransition
```

Make test intent clear from the method name alone. Use descriptive, specific condition descriptions.

## AAA Pattern (Arrange-Act-Assert)

EVERY test MUST follow the Arrange/Act/Assert pattern with clearly marked comment sections:

```
// Arrange
<setup test data, configure mocks>

// Act
<invoke method under test>

// Assert
<verify results and mock interactions>
```

## Mock Standards

### Strict Mocks - MANDATORY

All mocks MUST use strict behavior to ensure:
- All interactions are explicitly defined
- No unexpected calls go unnoticed
- Tests fail if setup is incomplete

### Setup Verification - MANDATORY

Every mock setup call MUST be chained with verification for the expected number of invocations. This replaces ad-hoc `Verify()` calls scattered through the Assert section.

### VerifyAll() - MANDATORY

Every test MUST call `VerifyAll()` on:
1. The service/class mock itself (when mocking the SUT)
2. ALL dependency mocks that have setup calls

This ensures every setup was actually invoked the expected number of times.

### Parameter Matching - Maximize Specificity

Avoid wildcard/any-match parameters when specific values can be checked:

- **Avoid**: Matching any argument when the exact value is known
- **Prefer**: Passing the exact expected value
- **Best**: Use predicate matching for complex objects

## Exception Testing

Use assertion methods to capture and verify exceptions. Do NOT use declarative exception attributes (e.g., `[ExpectedException]`) because they:
- Cannot verify the exception message
- Cannot verify exception properties
- Cannot assert state after the exception

Instead, use the framework's `ThrowsException` / `ThrowsExceptionAsync` assertion methods to capture the exception, then verify its message and properties.

## Virtual Method Testing

When the class under test has virtual methods:

1. **Testing the virtual method itself**: Configure the mock to call the real implementation (call-base) and verify invocation
2. **Testing a method that calls a virtual method**: Mock the virtual method's return value to isolate the method under test
3. **Testing a non-virtual method**: No special setup needed for the method under test

## Time-Dependent Testing

When testing code that depends on current time:
- Use a fake/testable time provider instead of the system clock
- Set a fixed time in test setup to ensure deterministic results
- Inject the time provider as a dependency

## Troubleshooting Test Failures

1. Use **tdd-guide** agent
2. Check test isolation
3. Verify mocks are correct
4. Fix implementation, not tests (unless tests are wrong)

## Agent Support

- **tdd-guide** - Use PROACTIVELY for new features, enforces write-tests-first
