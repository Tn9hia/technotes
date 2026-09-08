## Roadmap Learning
### 1. Fundamentals

- Variables (`var`, `:=`)
- Basic types
- Control flow: `if`, `switch`, `for`
- Functions: parameters, returns, variadic
- Error handling (no exceptions)


---

### 2. Structs & Pointers

- Define struct
- Access & update fields
- Pointer to struct

- Method receivers
    - Value receiver
    - Pointer receiver (mutates original)

---

### 3. Arrays vs Slices

#### Arrays

- Fixed-size
- Passed by value (creates copy)

#### Slices

- Reference to underlying array
- Contains: pointer + length + capacity

#### Key Concepts

- `len` = number of used elements
- `cap` = size before reallocation
- Slicing operator: `a[low:high]`
- Slices share underlying array → modifying one may affect the others
- `append()`
    - Uses original array if `cap` allows
    - Allocates new array if `cap` exceeded
- `make([]T, len, cap)`
    - Pre-allocate capacity to reduce reallocations
- `copy(dst, src)` to avoid shared memory
- Slice of slice → same underlying array unless copied
    

---

### 4. Maps

- Declare & update
- Check key existence using `value, ok := map[key]`
- Map of structs
- Not ordered

---

### 5. Methods & Interfaces

- Define methods on types
- Value vs pointer receiver
- Interfaces behave like duck typing
- Empty interface: `interface{}`
- Type assertion & type switch    

---

### 6. Concurrency

- Goroutines
- Channels (buffered/unbuffered)
- `select`
- Fan-in / Fan-out patterns
- Mutex vs channel-based synchronization
- Context cancellation    

---

### 7. Modules & Project Layout

- `go mod init`
- `go get`
- Standard project structure
- Build, install, run    

---

### 8. Testing

- `testing` package    
- Unit tests
- Benchmark tests
- Example tests
- Basic mocking patterns

---

### 9. Tooling & Best Practices

- `go fmt`, `go vet`    
- Linting
- Profiling & tracing
- Error wrapping with `fmt.Errorf`
- Avoid unnecessary interfaces
- Keep code small and explicit
