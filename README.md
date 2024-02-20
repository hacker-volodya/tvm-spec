# TVM Instructions Specification
There is ~700 instructions in TVM, so it's quite painful to implement tools such as disassembler, decompiler, etc.
This repo's goal is to provide machine-readable (JSON) description of TVM instructions: bytecode format, value flow and control flow information.

| Codepage | Specification
| -------- | -------------
| 0        | [cp0.json](./cp0.json)

JSON Schema of specifications is available in [schema.json](./schema.json). Generators such as [quicktype.io](https://app.quicktype.io/) can be used to generate wrappers for specific programming language.

Based on [instructions.csv](https://github.com/ton-community/ton-docs/blob/main/docs/learn/tvm-instructions/instructions.csv).

## Features

| JSON key     | Status      | Description
| ------------ | ----------- | -----------
| doc          | ✅ Implemented | Provides human-readable information about instructions. Can be useful to provide integrated docs to user, for example, in disassembler.
| bytecode     | ✅ Implemented | Describes instruction encoding. It contains information to determine, which instruction we are currently decoding and how to parse its operands.
| value_flow   | ✅ Implemented | Describes how instruction changes current stack. This part of specification allows to analyze how instructions interact with each other, so it becomes possible to implement high-level tools such as decompilers.
| control_flow | ✅ Implemented | Describes code flow (operations with cc register). It helps to reconstruct a control flow graph. This part mainly contains semantics of cont_* category instructions. For example, both JMPX and CALLX transfers execution to continuation on stack, but only CALLX returns and JMPX is not.
| aliases | ✅ Implemented | Specifies instruction aliases. Can be used to provide to user information about special cases (for example, SWAP is a special case of XCHG_0i with i = 1).

## Projects based on tvm-spec
1. [tvm-spec-example](https://github.com/hacker-volodya/tvm-spec-example), tiny TVM disassembler

## Instruction Specification
### Example
```json
[
    {
        "mnemonic": "LDU",
        "doc": {
            "category": "cell_parse",
            "description": "Loads an unsigned `cc+1`-bit integer `x` from _Slice_ `s`.",
            "gas": "26",
            "fift": "[cc+1] LDU"
        },
        "bytecode": {
            "doc_opcode": "D3cc",
            "tlb": "#D3 cc:uint8",
            "prefix": "D3",
            "operands": [
                {
                    "name": "c",
                    "loader": "uint",
                    "loader_args": {
                        "size": 8
                    }
                }
            ]
        },
        "value_flow": {
            "doc_stack": "s - x s'",
            "inputs": {
                "stack": [
                    {
                        "type": "simple",
                        "name": "s",
                        "value_types": ["Slice"]
                    }
                ]
            },
            "outputs": {
                "stack": [
                    {
                        "type": "simple",
                        "name": "x",
                        "value_types": ["Integer"]
                    },
                    {
                        "type": "simple",
                        "name": "s2",
                        "value_types": ["Slice"]
                    }
                ]
            }
        }
    },
    {
        "mnemonic": "EXECUTE",
        "doc": {
            "category": "cont_basic",
            "description": "_Calls_, or _executes_, continuation `c`.",
            "gas": "18",
            "fift": "EXECUTE\nCALLX"
        },
        "bytecode": {
            "doc_opcode": "D8",
            "tlb": "#D8",
            "prefix": "D8",
            "operands": []
        },
        "value_flow": {
            "doc_stack": "c - ",
            "inputs": {
                "stack": [
                    {
                        "type": "simple",
                        "name": "c",
                        "value_types": ["Continuation"]
                    }
                ]
            },
            "outputs": {
                "stack": [
                ]
            }
        },
        "control_flow": {
            "branches": [
                {
                    "type": "variable",
                    "var_name": "c",
                    "save": {
                        "c0": {
                            "type": "cc",
                            "save": {
                                "c0": { "type": "register", "index": 0 }
                            }
                        }
                    }
                }
            ]
        }
    }
]
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
| value_flow.inputs | Incoming values constraints. Input is unconstrained if absent.
| value_flow.inputs.stack | Defines how current stack is used by instruction. Instruction must not operate on stack entries other than defined here. Required.
| value_flow.inputs.stack[i] | Stack entry or group of stack entries.
| value_flow.inputs.stack[i].type | Type of stack entry. Can be one of "simple", "const", "conditional", "array". Required. 
| value_flow.inputs.stack[i].* | Properties for stack entries of each type are described below.
| value_flow.outputs | Outgoing values constraints. Output is unconstrained if absent. Identical to value_flow.inputs.
| control_flow | Information related to current cc modification by instruction. Optional.
| control_flow.branches | Array of possible branches of an instruction. Specifies all possible values of cc after instruction execution. Empty by default (no branches can be taken by instruction). Each branch described by a `Continuation` object described below.
| control_flow.nobranch | Can this instruction not perform any of specified branches in certain cases (do not modify cc)? False by default.

### Loaders Specification and Examples
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

### Stack Entry Specification and Examples

#### Simple
```json
{
    "type": "simple",
    "name": "c",
    "value_types": ["Continuation"]
}
```
Specifies a single stack entry, attaches `name` to it and optionally constraints its type by `value_types`. Possible types are `Integer`, `Cell`, `Builder`, `Slice`, `Tuple`, `Continuation`, `Null`. For example, dict is actually `"value_types": ["Slice", "Null"]` as it can be either Slice or Null. Absence of `value_types` means that this stack entry may be of any type.

#### Const
```json
{
    "type": "const",
    "value": 32,
    "value_type": "Integer"
}
```
Specifies a constant value on stack. `value_type` is either `Integer` or `Null`, with `number` type value for `Integer` and `null` value for `Null`.

#### Conditional
```json
[
    {
        "type": "conditional",
        "name": "status",
        "match": [
            {
                "value": 0, 
                "stack": []
            },
            {
                "value": -1, 
                "stack": [
                    {
                        "type": "simple",
                        "name": "c",
                        "value_types": ["Cell"]
                    }
                ]
            }
        ]
    },
    {
        "type": "simple",
        "name": "status",
        "value_types": ["Integer"]
    }
]
```
_Stack notation: `0 or c -1`_
```json
[
    {
        "type": "conditional",
        "name": "x",
        "match": [
            {"value": 0, "stack": []}
        ],
        "else": [
            {
                "type": "const",
                "value": null,
                "value_type": "Null"
            }
        ]
    },
    {
        "type": "simple",
        "name": "x",
        "value_types": ["Integer"]
    }
]
```
_Stack notation: `x or null x`_

Depending on `value` of variable specified in `name`, put values specified in corresponding arm of `match`. If `else` arm is not present, then input for this conditional may be only of specified values.

#### Array
```json
{
    "type": "array",
    "name": "key_message_pairs",
    "length_var": "n",
    "array_entry": [
        {
            "type": "simple",
            "name": "pk",
            "value_types": ["Slice"]
        },
        {
            "type": "simple",
            "name": "msg",
            "value_types": ["Slice"]
        }
    ]
}
```
_Stack notation: `pk_1 msg_1 ... pk_n msg_n`_

Specifies a bunch of stack entries with length from variable `length_var`, usually noted as `x_1 ... x_n`. Each part of array, such as `x_i` or `x_i y_i` is described in `array_entry`. Used in tuple, continuation arguments and crypto operations.

#### Notes
1. Each variable name is unique across `operands`, `value_flow`, and `control_flow` sections of each instruction. Assumed that variables are immutable, so if variable `x` is defined both in inputs and outputs, it goes to output without any modification.
2. Value flow describes only `cc` stack usage before the actual jump or call. Subsequent continuations may have a separate stack, so this will be defined in control flow section of this spec.

### Continuations Specification and Examples
Each object represents a continuation, which can be constructed in several ways: 
* from a variable (operand or stack)
* from cc register
* from c0-c3 registers
* by "extraordinary continuation" constructors (as in [tvm.pdf](https://docs.ton.org/tvm.pdf), p.50: "4.1.5. Extraordinary continuations.")

Savelist can be defined using `save` property. Keys are `c0`, `c1`, `c2`, `c3` and values are continuation objects (in fact, continuation is a recursive type, representing a tree of continuations). Please note that savelist defined here will not override already saved continuation registers (that's the standard behaviour of TVM when saving registers).

#### variable
```json
{
    "type": "variable",
    "var_name": "c",
    "save": {
        "c0": {
            "type": "cc",
            "save": {
                "c0": { "type": "register", "index": 0 }
            }
        }
    }
}
```
Specifies a variable-backed continuation. Variable may be not referenced previously (in `operands` or `value_flow` section), assuming that it is defined somewhere in the actual implementation of the instruction. Example of such instruction is `DICTIGETJMP`: internally it does dictionary lookup for continuation retrieval, so `x` in control flow is expected to be defined in the implementation of `DICTIGETJMP`.

#### cc
```json
{
    "type": "cc",
    "save": {
        "c0": { "type": "register", "index": 0 }
    }
}
```
Specifies a continuation, which is constructed from cc code and `save` registers. In C++ implementation this operation is implemented in [VmState::extract_cc](https://github.com/ton-blockchain/ton/blob/17c3477f7191fe6e5db22b71631b5c7472046c2f/crypto/vm/vm.cpp#L356).

#### register
```json
{
    "type": "register",
    "index": 3,
    "save": {
        "c0": {
            "type": "cc",
            "save": {
                "c0": { "type": "register", "index": 0 }
            }
        }
    }
}
```
Specifies continuation from register with `index` (`"index": 3` => take continuation from `c3`).

#### special
```json
{
    "type": "special",
    "name": "repeat",
    "args": {
        "count": "n",
        "body": {
            "type": "variable",
            "var_name": "c"
        },
        "after": {
            "type": "cc",
            "save": {
                "c0": { "type": "register", "index": 0 }
            }
        }
    }
}
```

Specifies extraordinary continuation. `save` property does not apply here.

| name | args | description
| ---- | ---- | -----------
| repeat | string count; continuation body, after | Count is the name of a count variable. Jumps to `body` with `c0 = repeat(count - 1, body, after)`, jumps to `after` if `count <= 0`.
| until | continuation body, after | Pops boolean from stack, jumps to `after` if it is `true`, otherwise jumps to `body` with `c0 = until(body, after)`.
| while | continuation cond, body, after | Pops boolean from stack, jumps to `after` if it is `false`, otherwise jumps to `body` with `c0 = cond` and `cond.c0 = while(cond, body, after)`.
| again | continuation body | Jumps to body with `c0 = again(body)`.
| pushint | continuation next; integer value | Push `value` on stack, jump to `next`.

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
