# miasm_notes

A concise and structured summary of the key components and APIs in the [Miasm](https://github.com/cea-sec/miasm) reverse engineering framework. This guide is designed as a quick reference to the internal mechanics and scripting capabilities of Miasm.

# Table of Contents
- [LocationDB](#locationdb)


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
