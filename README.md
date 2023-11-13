# TVM Instructions Specification
There is ~700 instructions in TVM, so it's quite painful to implement tools such as disassembler, decompiler, etc.
This repo's goal is to provide machine-readable (JSON) description of TVM instructions: bytecode format, value flow and control flow information.

| Codepage | Specification
| -------- | -------------
| 0        | [cp0.json](./cp0.json)

Based on [instructions.csv](https://github.com/ton-community/ton-docs/blob/main/docs/learn/tvm-instructions/instructions.csv).

## Features

| JSON key     | Status      | Description
| ------------ | ----------- | -----------
| doc          | ✅ Implemented | Provides human-readable information about instructions. Can be useful to provide integrated docs to user, for example, in disassembler.
| bytecode     | ✅ Implemented | Describes instruction encoding. It contains information to determine, which instruction we are currently decoding and how to parse its operands.
| value_flow   | ❌ Not implemented | Describes how instruction changes stack and registers. This part of specification allows to analyze how instructions interact with each other, so it becomes possible to implement high-level tools such as decompilers.
| control_flow | ❌ Not implemented | Describes code flow. It helps to reconstruct a control flow graph. This part mainly contains semantics of cont_* category instructions. For example, both JMPX and CALLX transfers execution to continuation on stack, but only CALLX returns and JMPX is not.
| aliases | ✅ Implemented | Specifies instruction aliases. Can be used to provide to user information about special cases (for example, SWAP is a special case of XCHG_0i with i = 1).

## Instruction Specification
### Example
```json
{
    "mnemonic": "XCHG_0I",
    "doc": {
        "category": "stack_basic",
        "description": "Interchanges `s0` with `s[i]`, `1 <= i <= 15`.",
        "gas": "18",
        "fift": "s[i] XCHG0"
    },
    "bytecode": {
        "doc_opcode": "0i",
        "tlb": "#0 i:(## 4) {1 <= i}",
        "prefix": "0",
        "operands_range_check": {
            "length": 4,
            "from": 1,
            "to": 15
        },
        "operands": [
            {
                "name": "i",
                "loader": "uint",
                "loader_args": {
                    "size": 4
                }
            }
        ]
    },
    "value_flow": {
        "doc_stack": ""
    }
}
```

### Documentation
| Key | Description |
| --- | ----------- |
| mnemonic | How instruction is named in [original TVM implementation](https://github.com/ton-blockchain/ton/blob/master/crypto/vm). Not necessarily unique (currently only DEBUG is not unique). Required.
| doc | Free-form human-friendly information which should be used for documentation purposes only.
| doc.category | Category of instruction (examples: cont_loops, stack_basic, dict_get). Required.
| doc.description | Free-form markdown description of instruction. Optional.
| doc.gas | Free-form description of gas amount used by instruction. Optional.
| doc.fift | Free-form fift usage description. Optional.
| doc.fift_examples | Optional array of usage examples in Fift. Each array element is object with `fift` and `description` keys.
| bytecode | Information related to bytecode format of an instruction. Assuming that each instruction has format `prefix || operand_1 || operand_2 || ...` (also some operands may be refs, not bitstring part).
| bytecode.doc_opcode | Free-form bytecode format description. Optional.
| bytecode.tlb | TL-b bytecode format description. Required.
| bytecode.prefix | Prefix to determine next instruction to parse. It is a hex bitstring as in TL-b (suffixed with `_` if bit length is not divisible by 4, trailing `'1' + '0' * x` must be removed). Required.
| bytecode.operands_range_check | In TVM, it is possible for instructions to have overlapping prefixes, so to determine actual instruction it is required to read next `length` bits after prefix as uint `i` and check `from <= i <= to`. Optional, there is no operands check in case of absence.
| bytecode.operands | Describes how to parse operands. Order of objects in this array represents the actual order of operands in instruction. Optional, no operands in case of absence.
| bytecode.operands[i].name | Operand variable name. Allowed chars are `a-zA-Z0-9_`, must not begin with digit or underscore and must not end with underscore. Required.
| bytecode.operands[i].loader | Loader function for operand. Must be one of `int`, `uint`, `ref`, `pushint_long`, `subslice`. Loaders are described below. Required.
| bytecode.operands[i].loader_args | Arguments for loader function, specified below. Optional, no arguments in case of absence.
| value_flow | Information related to usage of stack and registers by instruction. Optional.
| value_flow.doc_stack | Free-form description of stack inputs and outputs. Usually the form is `[inputs] - [outputs]` where `[inputs]` are consumed stack values and `outputs` are produced stack values (top of stack is the last value). Optional.

### Loaders specification and examples
#### uint
```json
{
    "name": "i",
    "loader": "uint",
    "loader_args": {
        "size": 4
    }
}
```
Loads unsigned `size`-bit int from bytecode. Argument `size` of type `integer` is required.
#### int
```json
{
    "name": "i",
    "loader": "int",
    "loader_args": {
        "size": 4
    }
}
```
Loads signed `size`-bit int from bytecode. Argument `size` of type `integer` is required.
#### ref
```json
{
    "name": "c",
    "loader": "ref"
}
```
Loads a single reference from bytecode. Unlike `subslice` with `refs_add = 1`, should dereference instead of returning empty cell with a single reference. No arguments are required.
#### pushint_long
```json
{
    "name": "x",
    "loader": "pushint_long"
}
```
Special loader which currently is used only in `PUSHINT_LONG` instruction. Loads 5-bit uint `size` and then loads and returns integer of bit size `8 * size + 19`. No arguments are required.
#### subslice
```json
[
    {
        "name": "r",
        "loader": "uint",
        "loader_args": {
            "size": 2
        }
    },
    {
        "name": "x",
        "loader": "uint",
        "loader_args": {
            "size": 5
        }
    },
    {
        "name": "slice",
        "loader": "subslice",
        "loader_args": {
            "bits_length_var": "x",
            "bits_padding": 1,
            "refs_length_var": "r",
            "refs_add": 1,
            "completion_tag": true
        }
    }
]
```

Loads subslice of bit length `{bits_length_var} * 8 + bits_padding` and ref count `{refs_length_var} + refs_add`. If `completion_tag` argument with value `true` is passed, remove completion tag from bitstring (trailing `'1' + '0' * x`).

| Argument | Description
| -------- | -----------
| bits_length_var | Name of (previously parsed) operand which contains bit length to load. Optional, assuming this part of bit length is 0 if absent.
| bits_padding | Constant integer value to add to length of bitstring to load. Optional, assuming 0 if absent.
| refs_length_var | Name of (previously parsed) operand which contains ref count to load. Optional, assuming this part of ref count is 0 if absent.
| refs_add | Constant integer value to add to ref count. Optional, assuming 1 if absent.
| completion_tag | Boolean flag, tells to remove trailing `'1' + '0' * x` from bitstring if true. Optional, assuming false if absent.

## Alias Specification
### Example
```json
{
    "mnemonic": "ROT2",
    "alias_of": "BLKSWAP",
    "doc_fift": "ROT2\n2ROT",
    "doc_stack": "a b c d e f - c d e f a b",
    "description": "Rotates the three topmost pairs of stack entries.",
    "operands": {
        "i": 1,
        "j": 3
    }
}
```

### Documentation
| Key | Description
| --- | -----------
| mnemonic | Alias name. Required.
| alias_of | Mnemonic of aliased instruction. Required.
| doc_fift | Free-form fift usage description. Optional.
| doc_stack | Free-form description of stack inputs and outputs. Usually the form is `[inputs] - [outputs]` where `[inputs]` are consumed stack values and `outputs` are produced stack values (top of stack is the last value).
| description | Free-form markdown description of alias. Optional.
| operands | Values of original instruction operands which are fixed in this alias. Currently it can be integer or slice without references which is represented by string of '0' and '1's. Type should be inferred from original instruction operand loaders. Required.
