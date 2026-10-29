<!-- syedazeez337/oss-contrib-setup:global/skills/tdd-go/SKILL.md | https://github.com/syedazeez337/oss-contrib-setup/blob/HEAD/global/skills/tdd-go/SKILL.md -->

---
name: tdd-go
description: Test-driven development for Go. Strict RED-GREEN-REFACTOR cycle. Use when implementing any new logic, fixing a bug, or adding behaviour to Go code. No production code without a failing test first.
version: 1.0.0
author: oss-contrib-setup (adapted from NousResearch/hermes-agent)
license: MIT
metadata:
  tags: [tdd, go, testing, methodology, red-green-refactor]
  related_skills: [systematic-debugging, writing-plans]
---

# TDD — Go

## Iron Law

**NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST.**

No exceptions for "trivial" changes. No exceptions for gosec fixes.
No exceptions when the fix "seems obvious". If it is correct, a test will prove it.

---

## The Cycle

### RED — Write a Failing Test

Write the test first. The test must fail before you write any implementation.

```go
func TestConvertPort_Overflow(t *testing.T) {
    tests := []struct {
        name    string
        input   int64
        want    int32
        wantErr bool
    }{
        {name: "valid port", input: 8080, want: 8080, wantErr: false},
        {name: "max valid", input: math.MaxInt32, want: math.MaxInt32, wantErr: false},
        {name: "overflow", input: math.MaxInt32 + 1, want: 0, wantErr: true},
        {name: "negative", input: -1, want: 0, wantErr: true},
    }
    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel()
            got, err := ConvertPort(tc.input)
            if (err != nil) != tc.wantErr {
                t.Errorf("ConvertPort(%d) error = %v, wantErr = %v", tc.input, err, tc.wantErr)
            }
            if !tc.wantErr && got != tc.want {
                t.Errorf("ConvertPort(%d) = %d, want %d", tc.input, got, tc.want)
            }
        })
    }
}
```

Run it. **It must fail with a compilation error or assertion failure.**

```bash
go test -run TestConvertPort_Overflow ./pkg/forward -v -count=1
# Expected output: FAIL (compilation error if function doesn't exist yet, or assertion failure)
```

If it passes before you write implementation → the test is wrong. Fix the test.

---

### GREEN — Write the Minimal Fix

Write the simplest code that makes the test pass. Do not add extra cases.
Do not handle future requirements. Do not clean up yet.

```go
// pkg/forward/forward.go

// ConvertPort converts an int64 port value to int32 with overflow protection.
func ConvertPort(port int64) (int32, error) {
    if port < 0 || port > math.MaxInt32 {
        return 0, fmt.Errorf("port value %d is out of int32 range [0, %d]", port, math.MaxInt32)
    }
    return int32(port), nil
}
```

Run the test:
```bash
go test -run TestConvertPort_Overflow ./pkg/forward -v -count=1
# Expected: PASS
```

Run the full suite to confirm nothing broke:
```bash
go test ./... -count=1
```

---

### REFACTOR — Clean Up

Only after GREEN. The tests must remain green throughout refactoring.

Things to do:
- Remove duplication
- Improve variable names
- Add a doc comment on the exported function
- Extract a helper if it will be reused (not just to reduce line count)

Things NOT to do:
- Add features not covered by any test
- Change the behaviour in any way

```bash
go test ./... -count=1  # must still pass
```

---

## Table-Driven Test Pattern (Default)

This is the standard Go testing pattern. Use it for every function with multiple cases.

```go
func TestFunctionName(t *testing.T) {
    tests := []struct {
        name    string
        // add input fields here
        // add expected output fields here
        wantErr bool
    }{
        // add test cases here
        // include: happy path, boundary values, error cases, zero values, nil inputs
    }
    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel()
            // act
            // assert using t.Errorf or testify/assert
        })
    }
}
```

### Test case naming conventions
- `"valid input"` — happy path
- `"zero value"` — empty/zero input
- `"nil input"` — nil pointer/interface
- `"boundary max"` — exact maximum valid value
- `"overflow"` — one beyond the maximum
- `"negative"` — negative input where not expected
- `"empty string"` — for string inputs

---

## Test File Conventions

- File: `foo_test.go` alongside `foo.go`
- Package: `package foo_test` for black-box testing (preferred for CNCF PRs)
- Use `package foo` only when testing unexported functions is genuinely necessary
- Test function name: `TestFunctionName` or `TestFunctionName_Scenario`
- Benchmark name: `BenchmarkFunctionName`
- Helper name: `assertFoo(t *testing.T, ...)` with `t.Helper()` as first statement

---

## Go Test Commands Reference

```bash
# Run a specific test
go test -run TestConvertPort ./pkg/forward -v -count=1

# Run all tests in a package
go test ./pkg/forward/... -v -count=1

# Run all tests in the repo
go test ./... -count=1

# Run with race detector (always for code touching goroutines/channels)
go test -race ./...

# Run with coverage
go test -cover ./...
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out    # visual browser report
go tool cover -func=coverage.out    # text summary per function

# Run benchmarks
go test -bench=BenchmarkConvertPort -benchmem ./pkg/forward

# Detect flaky tests
go test -run TestFlakyName ./pkg/... -count=20
```

---

## Common Rationalizations — and Why They're Wrong

| What you're thinking | Reality |
|---|---|
| "It's just a gosec lint fix — no behaviour change" | gosec G115 fixes change behaviour at boundary values. Test the boundary. |
| "The existing tests already cover this" | Run `-run TestSpecific` and confirm coverage. They probably don't cover the overflow case. |
| "I'll write tests after the PR is reviewed" | You won't. The PR will merge. Write the test now — it takes 10 minutes. |
| "The test is too simple to be worth writing" | Simple tests catch simple regressions. Simple code gets refactored into bugs. |
| "Writing the test will slow me down" | Writing a test before the fix makes the fix faster. You know exactly when you're done. |

---

## Red Flags — Stop and Correct

- You wrote implementation code before any test exists
- The test passes immediately without any implementation (test is testing the wrong thing)
- You modified a test assertion to make it pass instead of fixing the implementation
- Test file does not exist for the package you're modifying
- You cannot run `go test -run TestName ./pkg -v` and see it fail first
