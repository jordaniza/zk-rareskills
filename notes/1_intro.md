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
