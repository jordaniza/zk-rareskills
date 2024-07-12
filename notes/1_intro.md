# Intro

This document covers chapters [1](https://www.rareskills.io/post/p-vs-np) & [2](https://www.rareskills.io/post/arithmetic-circuit)

# Terminology

- **witness**: proof to a solution - this could be the solution itself but does not have to be.
- **===**: Comparison using circom syntax in arithmetic circuit.
- **signal**: variable in an arithmetic circuit.
- **constraints**: conditions on our signals our witness must satisfy.

# P vs. NP

Classic compsci, plenty of materials here.

- Define polynomial time as O(n^c).
- Define exponential time as O(c^n).

Where c is a constant.

In english: exponential time problems are much, much harder than polynomial problems.

| Type   | Verification | Computation | Note               |
| ------ | ------------ | ----------- | ------------------ |
| P      | O(n^c)       | O(n^c)      |
| NP     | O(n^c)       | O(c^n)      |
| PSpace | O(c^n)       | O(c^n)      | Finite memory      |
| ESpace | O(c^n)       | O(c^n)      | Exponential memory |

Critically, an NP problem is (assumed to be) one that can be quickly verified but not computed. It takes me ages to guess a number that resolves to a SHA256 hash of some state in the bitcoin blockchain with 6 leading zeroes, but I can easily check the result hashes to a correct value.

# Boolean Problems

We can model comparisons as booleans using truth tables and binary notation, for example:

| A   | B   | A > B |
| --- | --- | ----- |
| 0   | 0   | 0     |
| 1   | 1   | 0     |
| 1   | 0   | 1     |
| 0   | 1   | 0     |

| A   | B   | A = B |
| --- | --- | ----- |
| 0   | 0   | 1     |
| 1   | 1   | 1     |
| 1   | 0   | 0     |
| 0   | 1   | 0     |

A > B === `(A ∧ ¬B)`  
A = B === `¬(A ∨ B) ∨ (A ∧ B)`

(Not that logical NOT inverses the AND/OR operator in the bracket)

## Comparing numbers

We can establish A >= B by doing a bitwise comparison starting from the most significant bit.

For example:

```
11 =   1 0 1 1
7  =   0 1 1 1
---------------
       T
```

```
6 =    0 1 1 0
7 =    0 1 1 1
---------------
       - - - F
```

So essentially you perform a comparison over every bit and see if `(A_i ∨ ¬B_i)`

Specifically you can model this as below (from the book)

```
We number the bits in P as p₄, p₃, p₂, p₁ and the bits in Q as q₄, q₃, q₂, q₁.

If P ≥ Q then one of the following must be true:

p₄ > q₄
p₄ = q₄ and p₃ > q₃
p₄ = q₄ and p₃ = q₃ and p₂ > q₂
p₄ = q₄ and p₃ = q₃ and p₂ = q₂ and (p₁ > q₁ or p₁ = q₁)
```

If we rewrite this using our boolean comparisons for A and B, we get:

```
(p₄ ∧ ¬q₄) ∨

(((p₄ ∧ q₄) ∨ ¬(p₄ ∨ q₄)) ∧ (p₃ ∧ ¬q₃)) ∨

(((p₄ ∧ q₄) ∨ ¬(p₄ ∨ q₄)) ∧ ((p₃ ∧ q₃) ∨ ¬(p₃ ∨ q₃)) ∧ (p₂ ∧ ¬q₂)) ∨

(((p₄ ∧ q₄) ∨ ¬(p₄ ∨ q₄)) ∧ ((p₃ ∧ q₃) ∨ ¬(p₃ ∨ q₃)) ∧ ((p₂ ∧ q₂) ∨ ¬(p₂ ∨ q₂)) ∧ ((p₁ ∧ ¬q₁) ∨ ((p₁ ∧ q₁) ∨ ¬(p₁ ∨ q₁))))
```

Which is to say:

- Each new line is separated by an OR statement, because each statement contains the one before it.
- We expand the A > B for each boolean, then add A = B at the end.
- Inside the lines, we use AND statements for the individual booleans.

## How can we use this

You can transform _any_ P or NP problem into boolean arithmetic for _verification_. Solutions no, but you can model the _conditions_ that determine if a given witness is valid.

### Validating a list is sorted

Using the above formula we can see if A >= B, so we simply start from the first element and do the bitwise comparisons of i, i+1 for _all_ the elements. The answer is the AND of all the pairs.

So for a 3 element list its:

COMPARE(i0, i1) ∧ COMPARE(i1, i2)

So:

| i1 >= i0 | i2 >= i1 | isSorted                |
| -------- | -------- | ----------------------- |
| TRUE     | TRUE     | TRUE AND TRUE = TRUE    |
| FALSE    | FALSE    | FALSE AND FALSE = FALSE |
| TRUE     | FALSE    | RUE AND FALSE = FALSE   |
| FALSE    | TRUE     | FALSE AND TRUE = FALSE  |

Meaning the list is only sorted if all comparisons are TRUE.

### 3 Colour problem

See the post for graphics but what you're doing is verifying if a map of australia can be coloured with 3 colours, the rule being that the same colour can't appear adjacent to itself.

So there are 2 constraints here:

1. Colour constraint: each country must have exactly one colour
2. The border constraint: each country must not share the same colour as its neighbour

(1) can be expressed as

{Region}\_Red AND NOT {Region}\_Blue AND NOT {Region}\_Green

OR

NOT {Region}\_Red AND {Region}\_Blue AND NOT {Region}\_Green

OR

NOT {Region}\_Red AND NOT {Region}\_Blue AND {Region}\_Green

(NOT changes each line)

You repeat this for each region, so for region `WA`:

```
(WA_G ∧ ¬WA_B ∧ ¬WA_R) ∨ (¬WA_G ∧ WA_B ∧ ¬WA_R) ∨ (¬WA_G ∧ ¬WA_B ∧ WA_R)
```

(2) Can be expressed by defining the region (R) and the neighbours (N1, N2 .... Nn)

R_red AND NOT N1_red AND NOT N2_red ... === R_red AND NOT (N1_red OR N2_red ...)

You do this for EVERY colour AND region.

As you can see, this gets pretty chunky, pretty fast, so we use arithmetic circuits as an alternative way of expressing the problem that seeks to more elegantly convert our boolean verification of the witness into something more concise.

# Arithmetic Circuits

We can express booleans using arithmetic circuit notation. It's more or less what you'd expect:

a + b === 2
ab === 1

requires that we submit {a,b} as {1,1} (unless you're Terence Howard)
We don't have subtraction nor division but (I assume) we can use negative numbers and numbers 0 < x < 1 to create equivalents.

## Compared to Booleans

- Booleans use AND, OR, NOT, ACs use + & x
- A witness in the boolean case is an assignment to all boolean variables that resolves to TRUE
  - In an AC, it's an assignment to the signals that satisfy all equality constraints.
- The two are equivalent, and this can be proved, this means they can be used to verify all NP problems

## Using for colour maps

In the 3 color case, we can use ACs in conjunction with enums to express the color constraints as follows:

```
enum Colors {
  1: Red,
  2: Blue,
  3: Green
}
```

### Color Constraint

Then, to express "this territory should contain only one color" we define the constraint:

```
0 === (1 - x) * (2 - x) * (3 - x)
```

Then, given that x is a SINGLE value, the above expression will _only_ resolve to zero if `x => {1,2,3}`

So, for 3 territories, we would write

{t0, t1, t2}

CC(t): 0 === (1 - t) _ (2 - t) _ (3 - t)

We need all CC to hold for all t:

CC(t0)
CC(t1)
CC(t2)

### Border constraint

Here, we look at the pairwise case. If 2 regions have the same color, the product of the regions' colour will be C^2, or (1,4,9)

The other possible values are {2,3,6}

So you can express this as:

```c
c 0 === (2 - xy) _ (3 - xy) _ (6 - xy)
```

### All together

You can actually represent this in a sane way:

- Each letter represents the colour of a province of australia
- Each number expresses the colour constraint

```c
// color constraints
0 === (1 - WA) * (2 - WA) * (3 - WA)
0 === (1 - SA) * (2 - SA) * (3 - SA)
0 === (1 - NT) * (2 - NT) * (3 - NT)
0 === (1 - Q) * (2 - Q) * (3 - Q)
0 === (1 - NSW) * (2 - NSW) * (3 - NSW)
0 === (1 - V) * (2 - V) * (3 - V)

// boundary constraints
0 === (2 - WA * SA) * (3 - WA * SA) * (6 - WA * SA)
0 === (2 - WA * NT) * (3 - WA * NT) * (6 - WA * NT)
0 === (2 - NT * SA) * (3 - NT * SA) * (6 - NT * SA)
0 === (2 - NT * Q) * (3 - NT * Q) * (6 - NT * Q)
0 === (2 - SA * Q) * (3 - SA * Q) * (6 - SA * Q)
0 === (2 - SA * NSW) * (3 - SA * NSW) * (6 - SA * NSW)
0 === (2 - SA * V) * (3 - SA * V) * (6 - SA * V)
0 === (2 - Q * NSW) * (3 - Q * NSW) * (6 - Q * NSW)
0 === (2 - NSW * V) * (3 - NSW * V) * (6 - NSW * V)
```

## Sorting a list

### Expressing binary notation as a constraint

We can take any number `x` and express it as a sum of powers of 2, to convert it to binary:

```c
x = 13

2^3 * 1
2^2 * 0
2^1 * 1
2^0 * 1

8 * 1
4 * 0
2 * 1
1 * 1

x = 8 + 0 + 2 + 1 = 13
```

The generalised version of this is:

`x = x_n*(2^n) + x_(n-1)*2^(n-1) + ... x_1*(2^1) + x_0*(2^0)`

If you want to constrain x to a specifically sized number, note that the largest term is removed, so for a 4bit number:

`x(size=4bits) = x_3*(2^3) + x_2*(2^2) + x_1*(2^1) + x_0*(2^0)`

Meaning, in a generalised format:

`x(s=size) = x_(s-1)*2^(s-1) + ... x_1*(2^1) + x_0*(2^0)`

What's super cool to remember is that there are some specific operations in binary that we can leverage when building out circuits and constraints:

_Max sizing_: `2^n - 1` represents a binary number of all 1s, or the max possible value for a value of size `n` (in bits).

_Overflow_: `2^n` therefore represents the smallest possible value of `n+1` bits.

_Midpoint_: `2^(n-1)` (aka, 2^2 for a 4 bit number) represents the **halfway point** of the number for n bits, in terms of 1s and 0s:

```c
n = 3 bits

0 0 0 = 0
0 0 1 = 1
0 1 0 = 2
0 1 1 = 3
--------- // leading zero before, leading 1 after
1 0 0 = 4 // 2^2 === 2^(n-1)
1 0 1 = 5
1 1 0 = 6
1 1 1 = 7 // 2^3 - 1 === (2^n -1)
```

### Range constraints

You can start to see where this is going. Until now, we haven't demonstrated a way to compare 2 values in an AC.

However, using the above binary logic we can say a few things:

1. We can constrain a value to be a certain size using the following constraint:

choose (v,x0,x1,x2) st:

```c
// v must be expressable as a sum of powers of 2
v === 4x2 + 2x1 + x0

// all x values must equal 0, 1
x0(1-x0) === 0
x1(1-x1) === 0
x2(1-x2) === 0
```

### Putting it together

Recall the properties of `2^(n-1)`:

- It is the halfway point between `2^n-1 ` and 0
- It is the first value who's MSB is 1

Take some arbitrary delta `d`, then write:

`2^(n-1)` + `d`

For `d_n`:

- If `d_n` === 1, `d` < `2^(n-1)`
- If `d_n` === 0, `d` >= `2^(n-1)`

And recall that as a constraint:

```c
(v, x5, x4, x3, x2, x1, x0)
(u, y5, y4, y3, y2, y1, y0)

v === 16x4 + 8x3 + 4x2 + 2x1 + x0
u === 16y4 + 8y3 + 4y2 + 2y1 + y0

x0(1-x0) === 0
x1(1-x1) === 0
x2(1-x2) === 0
x3(1-x3) === 0
x4(1-x4) === 0

y0(1-y0) === 0
y1(1-y1) === 0
y2(1-y2) === 0
y3(1-y3) === 0
y4(1-y4) === 0
```

Now, to compare v & u, we compute:

`2^(n-1)` + `(v + u)` = `z`

if z_n\* === 0 =>
u is GTE v, because v+u must be negative

if z_n\* === 1 =>
u is LT v, becuse v+u is positive

### Overflow

Notice this creates an issue with larger values of v and u because of underflow and overflow.
In binary, adding 1 bit will _always_ provide room for all numbers below, so a simple further constraint is to say that, for `2^(n-1)`, v can only be at most `n-1` bits (same with u).

This means that for n=5, we can check equality for max 4 bit numbers.

### Complete examples

```c

// 4 bit number
n = 4

// constrain to 3 bit comparisons
v = (x2,x1,x0)
u = (y2,y1,y0)

// binary 1s and zeroes
v = x2(x2-1) + x1(x1-1) + x0(x0-1)
y = y2(y2-1) + y1(y1-1) + y0(y0-1)

// expressable as powers of 2
v = 4x2 + 2x1 + x0
u = 4y2 + 2y1 + y0

// to say v >= u we first diff the 2^3 == 8 against v + u
check = 8 + (v + u)

// finding the MSB of check
check = 8z3 + 4z2 + 2z1 + z0

// implies
z3 === 1
```

The above is then used for each element in the list, to check if it is sorted.

# Boolean Circuits -> Arithmetic Circuits

We saw how to constrain a value to 0, 1:

`x(x-1) === 0`

Mandates that x => {0,1}

How then, can we model different boolean gates (AND, OR, NOT). Recalling compsci 101, these basic building blocks allow us to craft all other bitwise operations and theoretically do anything.

## AND

v AND u has the following TT:

| v   | u   | output |
| --- | --- | ------ |
| 0   | 0   | 0      |
| 1   | 0   | 0      |
| 0   | 1   | 0      |
| 1   | 1   | 1      |

So we need a constraint that ONLY suffices for v === 1 and u === 1

Option 1:

v === 1
u === 1

Or

`1 - uv === 0`

for a given `x` then, replace 1 with x:

`x === uv`

## OR

| v   | u   | output |
| --- | --- | ------ |
| 0   | 0   | 0      |
| 1   | 0   | 1      |
| 0   | 1   | 1      |
| 1   | 1   | 1      |

Bit harder, we need to rule out v, u === 0, but can't set both

We know:

| v   | u   | vu  | v + u | v + u - vu |
| --- | --- | --- | ----- | ---------- |
| 0   | 0   | 0   | 0     | 0          |
| 1   | 0   | 0   | 1     | 1          |
| 0   | 1   | 0   | 1     | 1          |
| 1   | 1   | 1   | 2     | 1          |

So if vu === v + u, this is not allowed

so we could say:

`v + u - vu === 1`

For an arbitrary x this is

t === u + v - uv

## NOT

For v = NOT x

x(x-1) === 0 (constrain x to 1 and 0)
v === 1 - x

## A more complex example:

```c
out = (x ∧ ¬ y) ∨ z
```

1. Bound the values to binary

x(x - 1) === 0
y(y - 1) === 0
z(z - 1) === 0

2. Apply the above logic

```c
// Replace ¬ y with the the arithmetic circuit for NOT
// this is just a replacement for 1 - y
out = (x ∧ (1 - y)) ∨ z

// Replace ∧ with the arithmetic circuit for AND
// a AND b === ab (iff a and b are binary)
out = (x(1 - y)) ∨ z

// Replace ∨ with the arithmetic circuit for OR
// u + v - uv
out = (x(1 - y)) + z - (x(1 - y))z
```

Thus you have:

```c
x(x - 1) === 0
y(y - 1) === 0
z(z - 1) === 0
out === (x(1 - y)) + z - (x(1 - y))z

=== x - xy + z - z(x - xy)
=== x - xy + z - zx + zxy

// or alternatively:
out === x - xy + z - xz + xyz
```
