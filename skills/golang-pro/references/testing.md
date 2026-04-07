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

## Mocking with testify/mock

```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
    "github.com/stretchr/testify/require"
)

// Interface to mock
type EmailSender interface {
    Send(to, subject, body string) error
}

// Mock implementation using testify/mock
type MockEmailSender struct {
    mock.Mock
}

func (m *MockEmailSender) Send(to, subject, body string) error {
    args := m.Called(to, subject, body)
    return args.Error(0)
}

// Test using mock with assertions
func TestUserService_Register(t *testing.T) {
    mockSender := new(MockEmailSender)
    service := NewUserService(mockSender)

    // Set up expectations
    mockSender.On("Send", "user@example.com", mock.Anything, mock.Anything).
        Return(nil).
        Once() // Expect exactly one call

    err := service.Register("user@example.com")
    require.NoError(t, err)

    // Verify all expectations were met
    mockSender.AssertExpectations(t)
}

// Mock with return value variation
func TestUserService_Register_SendFails(t *testing.T) {
    mockSender := new(MockEmailSender)
    service := NewUserService(mockSender)

    mockSender.On("Send", mock.Anything, mock.Anything, mock.Anything).
        Return(assert.AnError).
        Once()

    err := service.Register("user@example.com")
    require.Error(t, err)
    assert.Contains(t, err.Error(), "failed to send")

    mockSender.AssertExpectations(t)
}

// Mock with specific argument matching
func TestUserService_Register_MultipleRecipients(t *testing.T) {
    mockSender := new(MockEmailSender)
    service := NewUserService(mockSender)

    // Expect two different calls with different arguments
    mockSender.On("Send", "user1@example.com", "Welcome", mock.Anything).
        Return(nil).
        Once()
    mockSender.On("Send", "user2@example.com", "Welcome", mock.Anything).
        Return(nil).
        Once()

    err := service.RegisterMultiple([]string{"user1@example.com", "user2@example.com"})
    require.NoError(t, err)

    mockSender.AssertNumberOfCalls(t, "Send", 2)
}

// Mock with Run to inspect/modify arguments
func TestUserService_Register_InspectCall(t *testing.T) {
    mockSender := new(MockEmailSender)
    service := NewUserService(mockSender)

    var capturedBody string
    mockSender.On("Send", mock.Anything, mock.Anything, mock.Anything).
        Run(func(args mock.Arguments) {
            capturedBody = args.Get(2).(string)
        }).
        Return(nil).
        Once()

    err := service.Register("user@example.com")
    require.NoError(t, err)
    assert.Contains(t, capturedBody, "Welcome")

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

## HTTP Testing with testify

```go
import (
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestHandler(t *testing.T) {
    tests := []struct {
        name       string
        method     string
        path       string
        wantStatus int
        wantBody   string
    }{
        {
            name:       "GET success",
            method:     http.MethodGet,
            path:       "/api/users",
            wantStatus: http.StatusOK,
            wantBody:   `{"users":[]}`,
        },
        {
            name:       "POST invalid body",
            method:     http.MethodPost,
            path:       "/api/users",
            wantStatus: http.StatusBadRequest,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            req := httptest.NewRequest(tt.method, tt.path, nil)
            rec := httptest.NewRecorder()

            handler := NewUserHandler()
            handler.ServeHTTP(rec, req)

            assert.Equal(t, tt.wantStatus, rec.Code, "status code mismatch")
            if tt.wantBody != "" {
                assert.JSONEq(t, tt.wantBody, rec.Body.String(), "response body mismatch")
            }
        })
    }
}

// Test with request body
func TestCreateUserHandler(t *testing.T) {
    body := `{"name":"John","email":"john@example.com"}`
    req := httptest.NewRequest(http.MethodPost, "/api/users", strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    rec := httptest.NewRecorder()

    handler := NewUserHandler()
    handler.ServeHTTP(rec, req)

    require.Equal(t, http.StatusCreated, rec.Code)

    var resp UserResponse
    err := json.Unmarshal(rec.Body.Bytes(), &resp)
    require.NoError(t, err)

    assert.NotEmpty(t, resp.ID, "user ID should be generated")
    assert.Equal(t, "John", resp.Name)
    assert.Equal(t, "john@example.com", resp.Email)
}

// Test with headers and cookies
func TestAuthenticatedHandler(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "/api/protected", nil)
    req.AddCookie(&http.Cookie{Name: "session", Value: "valid-token"})
    rec := httptest.NewRecorder()

    handler := NewProtectedHandler()
    handler.ServeHTTP(rec, req)

    require.Equal(t, http.StatusOK, rec.Code)
    assert.Equal(t, "application/json", rec.Header().Get("Content-Type"))
}
```

## testify/assert vs testify/require

Both packages provide the same assertions, but differ in failure behavior:

```go
import (
    "github.com/stretchr/testify/assert"  // Continue on failure
    "github.com/stretchr/testify/require" // Stop immediately on failure
)

func TestBothPackages(t *testing.T) {
    obj, err := CreateObject()

    // Use require for prerequisites - test stops if these fail
    require.NoError(t, err, "failed to create object")
    require.NotNil(t, obj, "object should not be nil")

    // Use assert for multiple validations - all run even if one fails
    assert.Equal(t, "expected", obj.Name)
    assert.Greater(t, obj.Count, 0)
    assert.NotEmpty(t, obj.Items)
}
```

**Guideline:**
- Use `require.*` for setup/preconditions (no point continuing if these fail)
- Use `assert.*` for multiple validations (see all failures at once)

## Advanced testify Features

### Eventually - Async Assertions

```go
import (
    "testing"
    "time"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestEventually(t *testing.T) {
    // Test that condition becomes true within timeout
    done := make(chan bool)
    go func() {
        time.Sleep(100 * time.Millisecond)
        close(done)
    }()

    require.Eventually(t, func() bool {
        select {
        case <-done:
            return true
        default:
            return false
        }
    }, 500*time.Millisecond, 10*time.Millisecond, "channel should close within timeout")
}

func TestNever(t *testing.T) {
    // Test that condition never becomes true within timeout
    counter := 0
    assert.Never(t, func() bool {
        counter++
        return counter > 100
    }, 50*time.Millisecond, 10*time.Millisecond, "counter should not exceed 100")
}
```

### Error Assertions

```go
func TestErrorAssertions(t *testing.T) {
    // Check error is returned
    err := DoSomethingThatFails()
    require.Error(t, err)

    // Check specific error type
    var customErr *CustomError
    assert.ErrorAs(t, err, &customErr)

    // Check error message contains substring
    assert.ErrorContains(t, err, "connection refused")

    // Check error is exactly the expected error
    expectedErr := errors.New("not found")
    assert.ErrorIs(t, err, expectedErr)
}
```

### Collection Assertions

```go
func TestCollectionAssertions(t *testing.T) {
    users := []User{
        {ID: 1, Name: "Alice"},
        {ID: 2, Name: "Bob"},
        {ID: 3, Name: "Charlie"},
    }

    // Subset checking
    assert.Subset(t, users, []User{{ID: 1, Name: "Alice"}})

    // Empty assertions
    assert.NotEmpty(t, users)
    assert.Empty(t, []int{})

    // Length
    assert.Len(t, users, 3)

    // Elements match (order independent)
    assert.ElementsMatch(t, []int{1, 2, 3}, []int{3, 2, 1})

    // Contains
    assert.Contains(t, users, User{ID: 1, Name: "Alice"})

    // Contains with element predicate
    assert.True(t, slices.ContainsFunc(users, func(u User) bool {
        return u.Name == "Bob"
    }))
}
```

### Object Comparison

```go
func TestObjectComparison(t *testing.T) {
    user1 := User{ID: 1, Name: "Alice", Email: "alice@example.com"}
    user2 := User{ID: 1, Name: "Alice", Email: "alice@example.com"}
    user3 := User{ID: 2, Name: "Bob"}

    // Equality
    assert.Equal(t, user1, user2)
    assert.NotEqual(t, user1, user3)

    // Same pointer
    ptr1 := &user1
    ptr2 := &user1
    ptr3 := &user2
    assert.Same(t, ptr1, ptr2)      // Same underlying object
    assert.NotSame(t, ptr1, ptr3)   // Different objects

    // Type assertions
    var i interface{} = "hello"
    assert.IsType(t, "", i)

    // Implements interface
    var _ io.Reader = new(bytes.Buffer)
    assert.Implements(t, (*io.Reader)(nil), new(bytes.Buffer))
}
```

## Common Assertions Quick Reference

| Assertion | Package | Description |
|-----------|---------|-------------|
| `assert.Equal(t, expected, actual)` | assert/require | Values are equal |
| `assert.NotEqual(t, unexpected, actual)` | assert/require | Values are not equal |
| `assert.True(t, condition)` | assert/require | Condition is true |
| `assert.False(t, condition)` | assert/require | Condition is false |
| `assert.Nil(t, obj)` | assert/require | Object is nil |
| `assert.NotNil(t, obj)` | assert/require | Object is not nil |
| `assert.Error(t, err)` | assert/require | Error is not nil |
| `assert.NoError(t, err)` | assert/require | Error is nil |
| `assert.ErrorContains(t, err, substr)` | assert/require | Error contains substring |
| `assert.ErrorIs(t, err, target)` | assert/require | Error is/is wrapped target |
| `assert.ErrorAs(t, err, &target)` | assert/require | Error can be assigned to target type |
| `assert.Contains(t, slice, item)` | assert/require | Slice/map/string contains item |
| `assert.NotContains(t, slice, item)` | assert/require | Slice/map/string does not contain item |
| `assert.Len(t, slice, length)` | assert/require | Slice/map has length |
| `assert.Empty(t, obj)` | assert/require | Object is zero value |
| `assert.NotEmpty(t, obj)` | assert/require | Object is not zero value |
| `assert.Greater(t, a, b)` | assert/require | a > b |
| `assert.GreaterOrEqual(t, a, b)` | assert/require | a >= b |
| `assert.Less(t, a, b)` | assert/require | a < b |
| `assert.LessOrEqual(t, a, b)` | assert/require | a <= b |
| `assert.Regexp(t, pattern, str)` | assert/require | String matches regex |
| `assert.JSONEq(t, expected, actual)` | assert/require | JSON strings are equal (ignores key order) |
| `assert.ElementsMatch(t, a, b)` | assert/require | Slices have same elements (order independent) |
| `assert.InDelta(t, expected, actual, delta)` | assert/require | Floats within delta |
| `assert.Same(t, a, b)` | assert/require | Pointers refer to same object |
| `assert.NotSame(t, a, b)` | assert/require | Pointers refer to different objects |
| `assert.Implements(t, iface, obj)` | assert/require | Object implements interface |
| `assert.IsType(t, expectedType, obj)` | assert/require | Object is of expected type |
| `assert.Subset(t, slice, subset)` | assert/require | Slice contains all elements of subset |
| `assert.Eventually(t, fn, timeout, tick)` | require | Condition becomes true within timeout |
| `assert.Never(t, fn, timeout, tick)` | assert | Condition never becomes true within timeout |

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
