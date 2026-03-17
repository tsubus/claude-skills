# Testing and Benchmarking

## Installation

```bash
go get github.com/stretchr/testify
```

## Table-Driven Tests

```go
package math

import (
    "testing"

    "github.com/stretchr/testify/assert"
)

func Add(a, b int) int {
    return a + b
}

func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -2, -3, -5},
        {"mixed signs", -2, 3, 1},
        {"zeros", 0, 0, 0},
        {"large numbers", 1000000, 2000000, 3000000},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            assert.Equal(t, tt.expected, result, "Add(%d, %d)", tt.a, tt.b)
        })
    }
}
```

## Subtests and Parallel Execution

```go
func TestParallel(t *testing.T) {
    tests := []struct {
        name  string
        input string
        want  string
    }{
        {"lowercase", "hello", "HELLO"},
        {"uppercase", "WORLD", "WORLD"},
        {"mixed", "HeLLo", "HELLO"},
    }

    for _, tt := range tests {
        tt := tt // Capture range variable for parallel tests
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // Run subtests in parallel

            result := strings.ToUpper(tt.input)
            assert.Equal(t, tt.want, result)
        })
    }
}
```

## Test Helpers and Setup/Teardown

```go
func TestWithSetup(t *testing.T) {
    // Setup
    db := setupTestDB(t)
    defer cleanupTestDB(t, db)

    tests := []struct {
        name string
        user User
    }{
        {"valid user", User{Name: "John", Email: "john@example.com"}},
        {"empty name", User{Name: "", Email: "test@example.com"}},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := db.SaveUser(tt.user)
            require.NoError(t, err, "SaveUser failed")
        })
    }
}

// Helper function (doesn't show in stack trace)
func setupTestDB(t *testing.T) *DB {
    t.Helper()

    db, err := NewDB(":memory:")
    require.NoError(t, err, "failed to create test DB")
    return db
}

func cleanupTestDB(t *testing.T, db *DB) {
    t.Helper()

    require.NoError(t, db.Close(), "failed to close DB")
}
```

## Mocking with Interfaces

```go
// Interface to mock
type EmailSender interface {
    Send(to, subject, body string) error
}

// Mock implementation
type MockEmailSender struct {
    mock.Mock
}

func (m *MockEmailSender) Send(to, subject, body string) error {
    args := m.Called(to, subject, body)
    return args.Error(0)
}

// Test using mock
func TestUserService_Register(t *testing.T) {
    mockSender := new(MockEmailSender)
    service := NewUserService(mockSender)

    mockSender.On("Send", "user@example.com", mock.Anything, mock.Anything).
        Return(nil)

    err := service.Register("user@example.com")
    require.NoError(t, err)

    mockSender.AssertExpectations(t)
}
```

## Suite Package for Test Suites

```go
import (
    "testing"

    "github.com/stretchr/testify/suite"
)

type UserServiceTestSuite struct {
    suite.Suite
    db     *DB
    service *UserService
}

func (s *UserServiceTestSuite) SetupSuite() {
    // Run once before all tests
    s.db = setupDB()
}

func (s *UserServiceTestSuite) TearDownSuite() {
    // Run once after all tests
    s.db.Close()
}

func (s *UserServiceTestSuite) SetupTest() {
    // Run before each test
    s.service = NewUserService(s.db)
}

func (s *UserServiceTestSuite) TestCreateUser() {
    user, err := s.service.CreateUser("john@example.com")
    s.NoError(err)
    s.NotNil(user)
    s.Equal("john@example.com", user.Email)
}

func (s *UserServiceTestSuite) TestCreateUser_InvalidEmail() {
    user, err := s.service.CreateUser("invalid")
    s.Error(err)
    s.Nil(user)
    s.Contains(err.Error(), "invalid email")
}

func TestUserServiceTestSuite(t *testing.T) {
    suite.Run(t, new(UserServiceTestSuite))
}
```

## Benchmarking

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(100, 200)
    }
}

// Benchmark with subtests
func BenchmarkStringOperations(b *testing.B) {
    benchmarks := []struct {
        name  string
        input string
    }{
        {"short", "hello"},
        {"medium", strings.Repeat("hello", 10)},
        {"long", strings.Repeat("hello", 100)},
    }

    for _, bm := range benchmarks {
        b.Run(bm.name, func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                _ = strings.ToUpper(bm.input)
            }
        })
    }
}

// Benchmark with setup
func BenchmarkMapOperations(b *testing.B) {
    m := make(map[string]int)
    for i := 0; i < 1000; i++ {
        m[fmt.Sprintf("key%d", i)] = i
    }

    b.ResetTimer() // Don't count setup time

    for i := 0; i < b.N; i++ {
        _ = m["key500"]
    }
}

// Parallel benchmark
func BenchmarkConcurrentAccess(b *testing.B) {
    var counter int64

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            atomic.AddInt64(&counter, 1)
        }
    })
}

// Memory allocation benchmark
func BenchmarkAllocation(b *testing.B) {
    b.ReportAllocs() // Report allocations

    for i := 0; i < b.N; i++ {
        s := make([]int, 1000)
        _ = s
    }
}
```

## Fuzzing (Go 1.18+)

```go
func FuzzReverse(f *testing.F) {
    // Seed corpus
    testcases := []string{"hello", "world", "123", ""}
    for _, tc := range testcases {
        f.Add(tc)
    }

    f.Fuzz(func(t *testing.T, input string) {
        reversed := Reverse(input)
        doubleReversed := Reverse(reversed)

        assert.Equal(t, input, doubleReversed, "Reverse(Reverse(x)) should equal x")
    })
}

// Fuzz with multiple parameters
func FuzzAdd(f *testing.F) {
    f.Add(1, 2)
    f.Add(0, 0)
    f.Add(-1, 1)

    f.Fuzz(func(t *testing.T, a, b int) {
        result := Add(a, b)

        // Properties that should always hold
        if b >= 0 {
            assert.GreaterOrEqual(t, result, a, "result should be >= a when b >= 0")
        }
    })
}
```

## Test Coverage

```go
// Run tests with coverage:
// go test -cover
// go test -coverprofile=coverage.out
// go tool cover -html=coverage.out

func TestCalculate(t *testing.T) {
    tests := []struct {
        name     string
        input    int
        expected int
    }{
        {"zero", 0, 0},
        {"positive", 5, 25},
        {"negative", -3, 9},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Calculate(tt.input)
            assert.Equal(t, tt.expected, result, "Calculate(%d)", tt.input)
        })
    }
}
```

## Race Detector

```go
// Run with: go test -race

func TestConcurrentAccess(t *testing.T) {
    var counter int
    var wg sync.WaitGroup

    // This will fail with -race if not synchronized
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++ // Data race!
        }()
    }

    wg.Wait()
}

// Fixed version with mutex
func TestConcurrentAccessSafe(t *testing.T) {
    var counter int
    var mu sync.Mutex
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()
            counter++
            mu.Unlock()
        }()
    }

    wg.Wait()

    assert.Equal(t, 10, counter)
}
```

## Golden Files

```go
import (
    "flag"
    "os"
    "path/filepath"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestRenderHTML(t *testing.T) {
    data := Data{Title: "Test", Content: "Hello"}
    result := RenderHTML(data)

    goldenFile := filepath.Join("testdata", "expected.html")

    if *update {
        // Update golden file: go test -update
        os.WriteFile(goldenFile, []byte(result), 0644)
    }

    expected, err := os.ReadFile(goldenFile)
    require.NoError(t, err, "failed to read golden file")

    assert.Equal(t, string(expected), result, "output doesn't match golden file")
}

var update = flag.Bool("update", false, "update golden files")
```

## Integration Tests

```go
// integration_test.go
//go:build integration

package myapp

import (
    "testing"
    "time"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestIntegration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test in short mode")
    }

    // Long-running integration test
    server := startTestServer(t)
    defer server.Stop()

    time.Sleep(100 * time.Millisecond) // Wait for server

    client := NewClient(server.URL)
    resp, err := client.Get("/health")
    require.NoError(t, err, "health check failed")

    assert.Equal(t, "ok", resp.Status)
}

// Run: go test -tags=integration
// Run short tests only: go test -short
```

## Testable Examples

```go
// Example tests that appear in godoc
func ExampleAdd() {
    result := Add(2, 3)
    fmt.Println(result)
    // Output: 5
}

func ExampleAdd_negative() {
    result := Add(-2, -3)
    fmt.Println(result)
    // Output: -5
}

// Unordered output
func ExampleKeys() {
    m := map[string]int{"a": 1, "b": 2, "c": 3}
    keys := Keys(m)
    for _, k := range keys {
        fmt.Println(k)
    }
    // Unordered output:
    // a
    // b
    // c
}
```

## Common Assertions Quick Reference

| Assertion | Description |
|-----------|-------------|
| `assert.Equal(t, expected, actual)` | Values are equal |
| `assert.NotEqual(t, unexpected, actual)` | Values are not equal |
| `assert.True(t, condition)` | Condition is true |
| `assert.False(t, condition)` | Condition is false |
| `assert.Nil(t, obj)` | Object is nil |
| `assert.NotNil(t, obj)` | Object is not nil |
| `require.Error(t, err)` | Error is not nil |
| `require.NoError(t, err)` | Error is nil |
| `assert.Contains(t, slice, item)` | Slice/map contains item |
| `assert.Len(t, slice, length)` | Slice has length |
| `assert.Empty(t, obj)` | Object is empty |
| `assert.Greater(t, a, b)` | a > b |
| `assert.Less(t, a, b)` | a < b |
| `assert.Regexp(t, pattern, str)` | String matches regex |
| `assert.JSONEq(t, expected, actual)` | JSON strings are equal |
| `assert.ElementsMatch(t, a, b)` | Slices have same elements |
| `assert.InDelta(t, expected, actual, delta)` | Floats within delta |

**Note:** Use `require.*` variants to stop test immediately on failure.

## Quick Reference

| Command | Description |
|---------|-------------|
| `go test` | Run tests |
| `go test -v` | Verbose output |
| `go test -run TestName` | Run specific test |
| `go test -bench .` | Run benchmarks |
| `go test -cover` | Show coverage |
| `go test -race` | Run race detector |
| `go test -short` | Skip long tests |
| `go test -fuzz FuzzName` | Run fuzzing |
| `go test -cpuprofile cpu.prof` | CPU profiling |
| `go test -memprofile mem.prof` | Memory profiling |
