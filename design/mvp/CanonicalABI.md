# Canonical ABI Explainer

* Link to AST for function definition
* What this explainer covers
* Python pseudo-code

Contents:
* [Statics](#Statics)
  * [Type flattening](#type-flattening)
  * [`canon.lift` validation](#canonlift-validation)
  * [`canon.lower` validation](#canonlower-validation)
* [Dynamics](#Dynamics)
  * [Value lifting](#value-lifting)
  * [Value lowering](#value-lowering)
  * [`canon.lift` execution](#canonlift-execution)
  * [`canon.lower` execution](#canonlower-execution)
  * [Calling host imports from components](#calling-host-imports-from-components)
  * [Calling component exports from the host](#calling-component-exports-from-the-host)

## Statics

### Type Flattening

```
MAX_INLINE = 4

def flatten(type):
  match type:
    case Unit():               return []
    case Bool():               return [I32()]
    case S8() | S16() | S32(): return [I32()]
    case U8() | U16() | U32(): return [I32()]
    case S64() | U64():        return [I64()]
    case Float32():            return [F32()]
    case Float64():            return [F64()]
    case Char():               return [I32()]
    case String() | List(_):   return [I32(), I32()]
    case Record(fields):       return flatten_record(map(lambda f: f.type, fields))
    case Variant(cases):       return flatten_variant(map(lambda c: c.type, cases))
    case Tuple(fields):        return flatten_record(fields)
    case Flags(cases):         return [I32()] * num_packed_i32s(cases)
    case Enum(cases):          return [I32()]
    case Union(cases):         return flatten_variant(cases)
    case Option(type):         return flatten_variant([type])
    case Expected(ok, fail):   return flatten_variant([ok, fail])

def flatten_record(field_types):
  flat = [x for ft in field_types for x in flatten(ft)]
  if len(flat) > MAX_INLINE:
    return [I32()]
  return flat

def flatten_variant(case_types):
  flat = []
  for ct in case_types:
    for i, t in enumerate(flatten(ct)):
      if i < len(flat):
        flat[i] = join(flat[i], t)
      else:
        flat.append(t)
  flat = [I32()] + flat
  if len(flat) > MAX_INLINE
    return [I32()]
  return flat

def join(a, b):
  match (a, b):
    case (F32(), F32()):                                   return F32()
    case (F64(), F64()):                                   return F64()
    case (I32(), I32()) | (I32(), F32()) | (F32(), I32()): return I32()
    case _:                                                return I64()

def num_packed_i32s(cases):
  n = math.ceil(len(cases) / 32)
  if n > MAX_INLINE:
    return [I32()]
  return n
```
* validation requires number of variant cases < 2<sup>32</sup>
* give example


### `canon.lift` validation

For a function definition:
```
(func $f (canon.lift (func (param T*) (result U) <canonopt>* $callee:<funcidx>))
```
 * `$f` is given type `(func (param T*) (result U))`
 * validation requires `$callee` has type `(func (param flatten(T)*) (result flatten(U)*))`


### `canon.lower` validation

For a function definition:
```
(func $f (canon.lower <canonopt>* $callee:<funcidx>))
```
where `$callee` has type `(func (param T*) (result U))`:
* `$f` is given type `(func (param flatten(T)*) (result flatten(U)*))`


## Dynamics

### Value Lifting

```
def lift(instance, type, values):
  match type:
    case Unit():             return
    case Bool():             return bool(values.next(I32))
    case S8():               return to_signed(values.next(I32), 8)
    case S16():              return to_signed(values.next(I32), 16)
    case S32():              return to_signed(values.next(I32), 32)
    case S64():              return to_signed(values.next(I64), 64)
    case U8():               return to_unsigned(values.next(I32), 8)
    case U16():              return to_unsigned(values.next(I32), 16)
    case U32():              return to_unsigned(values.next(I32), 32)
    case U64():              return to_unsigned(values.next(I64), 64)
    case Float32():          return to_float(values.next(F32))
    case Float64():          return to_float(values.next(F64))
    case Char():             return 
    case String() | List(_): return 
    case Record(fields):     return 
    case Variant(cases):     return 
    case Tuple(fields):      return 
    case Flags(cases):       return 
    case Enum(cases):        return 
    case Union(cases):       return 
    case Option(type):       return 
    case Expected(ok, fail): return 

def to_signed(x, bits):
  assert(x >= 0)
  trap_if(x >= (1 << bits))
  if x >= (1 << (bits - 1)):
    return x - (1 << bits)
  return x

def to_unsigned(x, bits):
  assert(x >= 0)
  trap_if(x >= (1 << bits))
  return x

def to_float(f):
  if math.isnan(f):
    return float("nan") # 0x7ff8000000000000
  return f

```

Lifting generator

### Value Lowering

Lowering consumer

### `canon.lift` execution

Accept lifting generator, call lowering consumer
Call
Return lifting generator, wait for post-return

### `canon.lower` execution

Pass lifting generator
Call
Accept lifting generator, calling lowering consumer
Call post-return

### Calling host imports from components

### Calling component exports from the host

