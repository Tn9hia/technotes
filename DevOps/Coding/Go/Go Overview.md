---
undefined: ""
File: Coding/Go/Go Overview.md
---

## Simple program
```go
package main

import (
	"fmt"
	"math"
)

func main() {
	fmt.Printf("Now you have %g problems.\n", math.Sqrt(7))
}
```
## Package
Every Go program is made up of **packages.**
Programs start running in package `main`.
## Imports
This code groups the imports into a parenthesized, "factored" import statement.
## Exported names

In Go, a name is exported if it begins with a capital letter. For example, `Pizza` is an exported name, as is `Pi`, which is exported from the `math` package.

`pizza` and `pi` do not start with a capital letter, so they are not exported.
## Functions

A function can take **zero or more arguments.**
In this example, `add` takes two parameters of type `int`.
Notice that the type comes _after_ the variable name.
When two or more consecutive named function parameters share a type, you can omit the type from all but the last.

In this example, we shortened
`x int, y int`
to
`x, y int`

```go
package main

import "fmt"

func add(x int, y int) int {
	return x + y
}

func main() {
	fmt.Println(add(42, 13))
}
```

## Syntax
Variable:
```
var tên_biến_1, tên_biến_2, ... kiểu_dữ_liệu
```

## Data type
| Kiểu                                                   | Giá trị mặc định   |
| ------------------------------------------------------ | ------------------ |
| `int`, `float`, `complex`                              | `0`, `0.0`, `0+0i` |
| `bool`                                                 | `false`            |
| `string`                                               | `""` (chuỗi rỗng)  |
| `pointer`, `slice`, `map`, `chan`, `interface`, `func` | `nil`              |
Go's basic types are
```
bool
string
int  int8  int16  int32  int64
uint uint8 uint16 uint32 uint64 uintptr
byte // alias for uint8
rune // alias for int32
     // represents a Unicode code point
float32 float64
complex64 complex128
```

## Flow control
### For statement
Go has only one looping construct, the `for` loop.
The `init` and `post` statements are optional.

```go
package main
import fmt

func main() {
	for i = 0; i<10; i++ {
		fmt.Println(i)
	}
}
```

### If Statement
Go's `if` statements are like its `for` loops; the expression need not be surrounded by parentheses `( )` but the braces `{ }` are required.
```Go
package main

import (
	"fmt"
	"math"
)

func sqrt(x float64) string {
	if x < 0 {
		return sqrt(-x) + "i"
	}
	return fmt.Sprint(math.Sqrt(x))
}

func main() {
	fmt.Println(sqrt(2), sqrt(-4))
}

```