# microcode-generator

A generator for the microcode of my custom CPU. This is a language of sorts that
generates hex files to put in ROM for the microcode of a ROM for control signals
of the CPU. This program does assume a few things about the ISA however:

1. Each instruction should begin with an operator instruction to decide which
   routine to enter. For example, if there were 8 instructions (excluding
   parameters of the opcodes), you would need 3 bits to discriminate between the
   instructions. Each of these instructions can be programmed to have a distinct
   routine.
2. The final bits of the full input, called `iter_bits`, are used so an external
   counter is able to increment the memory address of the microcode ROM and
   change the output of the signals for each routine. This does mean that each
   routine is limited to 2^`iter_bits` operations, so choose your inputs wisely.

## V1.0 Language Standard

### Note on Terms

Some terms will be used in this standard which may be ambiguous, so here are the
specific definitions of those terms:

- Identifier: An alphanumeric string that does not start with a number.
- Block: An encapsulated piece of code between two braces (`{` and `}`). The
  function and syntax of code in the block depends on the type of block.
- Instruction: A command which sets the signals to the state necessary for the
  function of the routine.

### Parsing Info

All assignment and instruction operations must be terminated with a semicolon.
Newlines are treated like spaced, so feel free to create newlines as needed (or
not at all if you are really feeling it).

### Main Blocks

#### Config

A `.mcgr` file is parsed with the information from two blocks: the `config`
block, and the `routines` block. The `config` block should contain the following
information:

- `address_len`: The length (in bytes) of the addressing width.
- `output_width`: The length (in bytes) of the output width. Could be different
  than 1 byte if you have more than 8 control signals. Instructions are placed
  in LSB order for writing to ROMs.
- `op_len`: Determines the maximum number of routines you can have. This number
  is the amount of bits needed to represent each distinct routine. For example,
  if you had 8 distinct routines, you would need 3 bits. These operators take
  the most significant bits of the address.
- `inputs`: The number of addressing inputs to the ROM that do not include the
  `op_len` or the `iter_bits`. Each input is a space seperated identifier.
- `signals`: A list of the output signals. Similar to the `inputs` field, each
  signal is an identifier seperated by a space.
- `iter_len`: The length (in bits) of the space allocated to iterate to the next
  instruction. These bits take up the least significant bits of the address.

Each input and signal is tied to a bit index (least significant bit is index 0).
These cannot go out of range of the address or output with, nor can they
interfere with an `op` or `iter` bit. This will cause an error.

An example configuration is shown below

```
config {
    address_len = 1;
    output_width = 1;
    op_len = 3;
    inputs = interupt:3;
    signals = pc_we:0 reg_we:1 reg_sel:2 reg_sel2:3 
              mem_we:4 mem_oe:5 ir_we:6 pc_sel:7;
    iter_len = 3;
}
```

#### Routines

The second main block that should be passed is the `routines` block. This block
contains all of the functional code for the program. The `routines` block is
made up of routines or global conditional statements (we will get to those
later). Each routine is its own block starting with `op <op index>`. For
example, if I wanted to make a routine for the op `000`, the block would start
with `op 001`. Inside of the block, there are two types of statements that can
follow:

##### Instructions

Instructions are the actual units that set the signals. Each instruction is
placed at the routine address prefix, and they are placed in order. The number
of instructions are limited by `iter_len`. An instruction contains a list of
signals, and the states of those signals as an `on` or `off`, or `1` or `0`. Any
signal excluded from the list will automatically be set to off, though you
should include all relevant signals to make the config readable. The signal is
set via a colon. For example, an instruction could look like

`pc_we:on reg_we:off mem_we:off mem_oe:on ir_we:off;`

The state of signals can also be set by inputs

`mem_we:interupt mem_oe:on;`

##### Conditions

Typically, a routine without any condition will appear in every possible state
of the inputs. This makes it so the same routine will run no matter the state of
any signal. However, if you want one part of the routine to run depending on the
state of a input, you can use a condition. Conditions are formatted as
`if <boolean expresssion> {`. This does however mean that if these signals are
used in the if-block, they will be replaced by the compiler and you will get a
warning. An example if-block is shown below

```
if interupt {
    mem_oe:on pc_we:off pc_sel:0
    mem_oe:on pc_we:off pc_sel:1
}
```

#### Absolute Conditions

An absolute condition is a condition where, no matter the op, the instructions
contains in the block are ran. These instructions are copied at every single op
where the signal is set, however this does mean it is pointless to reference the
signal in other blocks if the condition is only an input. The absolute
conditions are formatted the same.

#### Macros

If there is a repreated set of instructions that is run, it can be wrapped in a
macro. The macros are defined aboove where they are used in the `macros` block.
The syntax is simply `<macro name>|(<params>...)| {`, and then a block of
instructions. The parameters are replaced literally by the arguments provided to
the macro's invokation. To reference a macro, prefix the macro's name with `$`.
You can also run `if` statements off of these by starting with `$if <boolean>`
and ending with `$endif`. As with most kinds of macros, the parameters directly
replace signals. Macro `if` statements cannot be used with signals, only with
direct boolean values.

#### A Full Example

```
macros {
    load_pc(pc_sel_arg) {
        mem_oe:on pc_we:on pc_sel:$pc_sel_arg:
        mem_oe:on pc_we:off ir_we:on;

        $if pc_sel_arg

        mem_oe:off;

        $endif
    }
}

config {
    address_len = 1;
    output_width = 1;
    op_len = 3;
    inputs = interupt:3;
    signals = pc_we:0 reg_we:1 reg_sel:2 accum_we
              mem_we:4 mem_oe:5 ir_we:6 pc_sel:7;
    iter_len = 3;
}

routines {                              // The main block for writing logic

    op 000 {                            
        $load_pc(0)
        reg_sel1:0 accum_we;
    }

    if interupt {
        $load_pc(1)
        reg_sel1:0 accum_we;
    }
}
```
