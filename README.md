# udecimal

[![build](https://github.com/quagmt/udecimal/actions/workflows/ci.yaml/badge.svg)](https://github.com/quagmt/udecimal/actions/workflows/ci.yaml)
[![Go Report Card](https://goreportcard.com/badge/github.com/quagmt/udecimal)](https://goreportcard.com/report/github.com/quagmt/udecimal)
[![codecov](https://codecov.io/gh/quagmt/udecimal/graph/badge.svg?token=662ET843EZ)](https://codecov.io/gh/quagmt/udecimal)
[![GoDoc](https://pkg.go.dev/badge/github.com/quagmt/udecimal)](https://pkg.go.dev/github.com/quagmt/udecimal)

High performance, high precision, zero allocation fixed-point decimal number for financial applications.

## Installation

```sh
go get github.com/quagmt/udecimal
```

## Features

- **High Precision**: Supports up to 19 decimal places with no precision loss during arithmetic operations.
- **Optimized for Speed**: Designed for high performance with zero memory allocation in most cases (see [Benchmarks](benchmarks/BENCHMARKS.md) and [How it works](#how-it-works)).
- **Panic-Free**: All errors are returned as values, ensuring no unexpected panics.
- **Immutable**: All arithmetic operations return a new `Decimal` value, preserving the original value and safe for concurrent use.
- **Versatile Rounding Methods**: Includes HALF AWAY FROM ZERO, HALF TOWARD ZERO, and Banker's rounding.
  <br/>

**NOTE**: This library does not perform implicit rounding. If the result of an operation exceeds the maximum precision, extra digits are truncated. All rounding methods must be explicitly invoked. (see [Rounding Methods](#rounding-methods) for more details)

## Documentation

- Checkout [documentation](https://pkg.go.dev/github.com/quagmt/udecimal) for more information.

## Usage

```go
package main

import (
	"fmt"

	"github.com/quagmt/udecimal"
)

func main() {
	// Create a new decimal number
	a, _ := udecimal.NewFromInt64(123456, 3)              // a = 123.456
	b, _ := udecimal.NewFromInt64(-123456, 4)             // b = -12.3456
	c, _ := udecimal.NewFromFloat64(1.2345)               // c = 1.2345
	d, _ := udecimal.Parse("4123547.1234567890123456789") // d = 4123547.1234567890123456789

	// Basic arithmetic operations
	fmt.Println(a.Add(b)) // 123.456 - 12.3456 = 111.1104
	fmt.Println(a.Sub(b)) // 123.456 + 12.3456 = 135.8016
	fmt.Println(a.Mul(b)) // 123.456 * -12.3456 = -1524.1383936
	fmt.Println(a.Div(b)) // 123.456 / -12.3456 = -10
	fmt.Println(a.Div(d)) // 123.456 / 4123547.1234567890123456789 = 0.0000299392722585176

	// Divide with precision, extra digits are truncated
	fmt.Println(a.DivExact(c, 10)) // 123.456 / 1.2345 = 100.0048602673

	// Rounding
	fmt.Println(c.RoundBank(3)) // banker's rounding: 1.2345 -> 1.234
	fmt.Println(c.RoundHAZ(3))  // half away from zero: 1.2345 -> 1.235
	fmt.Println(c.RoundHTZ(3))  // half towards zero: 1.2345 -> 1.234
	fmt.Println(c.Trunc(2))     // truncate: 1.2345 -> 1.23
	fmt.Println(c.Floor())      // floor: 1.2345 -> 1
	fmt.Println(c.Ceil())       // ceil: 1.2345 -> 2

	// Display
	fmt.Println(a.String())         // 123.456
	fmt.Println(a.StringFixed(10))  // 123.4560000000
	fmt.Println(a.InexactFloat64()) // 123.456
}
```

## Rounding Methods

Rounding numbers can often be challenging and confusing due to the [variety of methods](https://www.mathsisfun.com/numbers/rounding-methods.html) available. Each method serves specific purposes, and it's common for developers to make mistakes or incorrect assumptions about how rounding should be performed. For example, the result of `round(1.45)` could be either 1.4 or 1.5, depending on the rounding method used.

This issue is particularly critical in financial applications, where even minor rounding errors can accumulate and lead to significant financial losses. To mitigate such errors, this library intentionally avoids implicit rounding. If the result of an operation exceeds the maximum precision specified by developers beforehand, **extra digits are truncated**. Developers need to explicitly choose the rounding method they want to use. The supported rounding methods are:

- [Banker's rounding](https://en.wikipedia.org/wiki/Rounding#Rounding_half_to_even) or round half to even
- [Half away from zero](https://en.wikipedia.org/wiki/Rounding#Rounding_half_away_from_zero) (HAZ)
- [Half toward zero](https://en.wikipedia.org/wiki/Rounding#Rounding_half_toward_zero) (HTZ)

### Examples:

```go
package main

import (
	"fmt"

	"github.com/quagmt/udecimal"
)

func main() {
	// Create a new decimal number
	a, _ := udecimal.NewFromFloat64(1.5) // a = 1.5

	// Rounding
	fmt.Println(a.RoundBank(0)) // banker's rounding: 1.5 -> 2
	fmt.Println(a.RoundHAZ(0))  // half away from zero: 1.5 -> 2
	fmt.Println(a.RoundHTZ(0))  // half towards zero: 1.5 -> 1
}
```

## How it works

As mentioned above, this library is not always memory allocation free. However, those cases where we need to allocate memory are incredily rare. To understand why, let's take a look at how the `Decimal` type is implemented.

The `Decimal` type represents a fixed-point decimal number. It consists of three components: sign, coefficient, and prec. The number is represented as:

```go
// decimal value = (neg == true ? -1 : 1) * coef * 10^(-prec)
type Decimal struct {
	neg bool
	coef bint
	prec uint8 // 0 <= prec <= 19
}

// Example:
// 123.456 = 123456 * 10^-3
// -> neg = false, coef = 123456, prec = 3

// -123.456 = -123456 / 10^-3
// -> neg = true, coef = 123456, prec = 3
```

<br/>

You can notice that `coef` data type is `bint`, which is a custom data type:

```go
type bint struct {
	// Indicates if the coefficient exceeds 128-bit limit
	overflow bool

	// Indicates if the coefficient exceeds 128-bit limit
	u128 u128

	// For coefficients exceeding 128-bit
	bigInt *big.Int
}
```

The `bint` type can store coefficients up to `2^128 - 1` using `u128`. Arithmetic operations with `u128` are fast and require no memory allocation. If result of an arithmetic operation exceeds u128 capacity, the whole operation will be performed using `big.Int` API. Such operations are slower and do involve memory allocation. However, those cases are rare in financial applications due to the extensive range provided by a 128-bit unsigned integer, for example:

- If precision is 0, the decimal range it can store is:
  `[-340282366920938463463374607431768211455, 340282366920938463463374607431768211456]`(approximately -340 to 340 undecillion)

- If precision is 19, the decimal range becomes:
  `[-34028236692093846346.3374607431768211455, 34028236692093846346.3374607431768211455]` (approximately -34 to 34 quintillion)

Therefore, in most cases you can expect high performance and no memory allocation when using this library.

## Credits

This library is inspired by [govalues/decimal](https://github.com/govalues/decimal) and [lukechampine/uint128](https://github.com/lukechampine/uint128)

## License

**MIT**