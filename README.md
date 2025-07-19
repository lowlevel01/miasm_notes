# miasm_notes

A concise and structured summary of the key components and APIs in the [Miasm](https://github.com/cea-sec/miasm) reverse engineering framework. This guide is designed as a quick reference to the internal mechanics and scripting capabilities of Miasm.

# Table of Contents
- [Expressions](#expressions)
- [LocationDB](#locationdb)


# Expressions

Miasm provides an intermediate representation (IR) to represent the effects of a source code. The benefits of using an IR are:

- A unified representation that does not depend on the source architecture
- A minimal language
- The side effects are explicit (e.g. `A + B` will not implicitly update flags)

Miasm's IR implementation is located in `miasm.expression.expression`, with `Expr*` objects.

## Rules

- Each expression has a size (in bits)
- Some expressions need this size at creation time, others compute it from their arguments

## IR Words

| Word           | Meaning             |
|----------------|---------------------|
| `ExprAssign`   | `A = B`             |
| `ExprInt`      | `0x18`              |
| `ExprId`       | `EAX`               |
| `ExprLoc`      | `label_1`           |
| `ExprCond`     | `A ? B : C`         |
| `ExprMem`      | `@16[ESI]`          |
| `ExprOp`       | `A + B`             |
| `ExprSlice`    | `AH = EAX[8:16]`    |
| `ExprCompose`  | `AX = AH.AL`        |

## ExprOp Rules

- In most cases, arguments must have the same size
- Size of `ExprOp` is the size of its arguments
- Some exceptions (e.g. `==`, `parity`) have size 1
- `+`, `^`, `|`, ... are n-ary operations
- `-` is always unary
- `parity` always has size 1

## ExprAssign

- Represents assignment `dst = src`
- Not a word of the language: cannot be used inside another expression

## ExprCond

- Ternary conditional: `cond ? src1 : src2`
- `src1` and `src2` must have the same size
- `cond` may differ in size

## ExprSlice

- Extracts bit slice from an expression
- Size = `stop - start`

## ExprCompose

- Concatenates two expressions
- Size = sum of argument sizes

## ExprLoc

- Represents a location (jump/call target)
- Wraps a `LocKey`, which contains info like offset or name

## Helpers

- `a.mask` → returns mask expression of same size
- `a.size` → returns expression size
- `repr(expr)` → printable/copyable form
- `expr.zeroExtend(size)` / `signExtend(size)` → size extension
- `expr.msb()` → most significant bit
- `expr.replace_expr({old: new})` → substitution
- Type tests:
  - `is_id`, `is_int`, `is_op`, `is_op("+")`, `is_op("&")`

## Expression Graphs

- Expressions are recursive and can be viewed as graphs
- Use `.graph()` method
- Graph is a `DiGraph` from `miasm.core.graph`
- Supports:
  - `.dot()` for Graphviz
  - node/edge access
  - dominators, successors, etc.

## Expression Simplification

- Implemented in `miasm.expression.simplifications`
- Use `expr_simp` to apply transformation rules
- Simplifies:
  - Constants (e.g. `0x10 + -1 = 0xF`)
  - Bit slicing
  - Algebraic identities (e.g. `a + a - a = a`)
  - Replacements (e.g. evaluate with `a = 0x10`)
- Use `ExpressionSimplifier.enable_passes(...)` to activate additional rules

## Custom Passes Example

- Boolean `ExprCond` transformed to `<`:
  ```python
  ((x - y) ^ ((x ^ y) & ((x - y) ^ x)))[31:32]
  ```
- Enable with:
  ```python
  expr_simp_cond.enable_passes(ExpressionSimplifier.PASS_COND)
  ```

# LocationDB

## Overview
`LocationDB` is Miasm's core symbol management object. It maintains mappings between code/data positions (`Location`) and symbolic identifiers (`LocKey`).

---

## Key Concepts

- **Location**: Represents a code or data position.
- **LocKey**: Unique identifier for a location (like a database primary key).
- Each `Location` has exactly **one associated LocKey**.
- Each `LocKey` is tied to **one `LocationDB`** only.
- A `LocKey`:
  - Can have an **optional offset**.
  - Can have **multiple symbol names** (aliases).
  - Must have a **unique offset** within the `LocationDB`.
  - Must have **unique symbol names** within the `LocationDB`.

---

## LocationDB Operations

### Create a `LocationDB`
```python
from miasm.core.locationdb import LocationDB
loc_db = LocationDB()
print(repr(loc_db))
```

### Add Locations
```python
# Default location (no offset, no name)
loc_a = loc_db.add_location()

# With offset
loc_b = loc_db.add_location(offset=112233)

# With a name
loc_c = loc_db.add_location(name="main")

# Add alias
loc_db.add_location_name(loc_c, "_main_")

# Add another location
loc_d = loc_db.add_location()
```

### Modify Locations
```python
# Associate an offset
loc_db.set_location_offset(loc_a, 0x5678)

# Remove a name
loc_db.remove_location_name(loc_c, "_main_")
```

### Query Information
```python
# Get the offset
offset = loc_db.get_location_offset(loc_a)

# Location with no offset returns None
print(loc_db.get_location_offset(loc_c))  # None

# Pretty display
print(loc_db.pretty_str(loc_a))  # e.g., 'loc_5678'
print(loc_db.pretty_str(loc_b))  # e.g., 'loc_1b669'
print(loc_db.pretty_str(loc_c))  # 'main'
print(loc_db.pretty_str(loc_d))  # e.g., 'loc_key_3'
```

---

## Pretty Format Rules

| Condition                      | Output Format       |
|-------------------------------|---------------------|
| Has a name                    | `"name"`            |
| Has offset but no name        | `"loc_<offset>"`    |
| Has neither name nor offset   | `"loc_key_<index>"` |

---

## IR Integration
```python
# Get a lifter from the machine and the loc_db
lifter = machine.lifter_model_call(loc_db)

# Get IR from asm CFG
ircfg = lifter.new_ircfg_from_asmcfg(asmcfg)

# Access block by location
loc_entry = loc_db.get_offset_location(0)
irblock = ircfg.blocks[loc_entry]

# Display IR block
print(irblock)

# Get IRDst (IR jump target)
dst = irblock.dst
print(dst)
print(repr(dst))

# Decompose conditional destination
src1, src2 = dst.src1, dst.src2
print(repr(src1), repr(src2))

# Access LocKey from ExprLoc
loc = src1.loc_key
print(loc)
```
- Each IR expression has a defined size.
- `Location` itself has **no size**.
- Must use `ExprLoc(loc_key, size)` in IR.
