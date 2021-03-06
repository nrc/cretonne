********************************
Cretonne Meta Language Reference
********************************

.. default-domain:: py
.. highlight:: python
.. module:: cretonne

The Cretonne meta language is used to define instructions for Cretonne. It is a
domain specific language embedded in Python. This document describes the Python
modules that form the embedded DSL.

The meta language descriptions are Python modules under the
:file:`lib/cretonne/meta` directory. The descriptions are processed in two
steps:

1. The Python modules are imported. This has the effect of building static data
   structures in global variables in the modules. These static data structures
   use the classes in the :mod:`cretonne` module to describe instruction sets
   and other properties.

2. The static data structures are processed to produce Rust source code and
   constant tables.

The main driver for this source code generation process is the
:file:`lib/cretonne/meta/build.py` script which is invoked as part of the build
process if anything in the :file:`lib/cretonne/meta` directory has changed
since the last build.


Settings
========

Settings are used by the environment embedding Cretonne to control the details
of code generation. Each setting is defined in the meta language so a compact
and consistent Rust representation can be generated. Shared settings are defined
in the :mod:`cretonne.settings` module. Some settings are specific to a target
ISA, and defined in a `settings` module under the appropriate
:file:`lib/cretonne/meta/isa/*` directory.

Settings can take boolean on/off values, small numbers, or explicitly enumerated
symbolic values. Each type is represented by a sub-class of :class:`Setting`:

.. inheritance-diagram:: Setting BoolSetting NumSetting EnumSetting
    :parts: 1

.. autoclass:: Setting
.. autoclass:: BoolSetting
.. autoclass:: NumSetting
.. autoclass:: EnumSetting

All settings must belong to a *group*, represented by a :class:`SettingGroup`
object.

.. autoclass:: SettingGroup

Normally, a setting group corresponds to all settings defined in a module. Such
a module looks like this::

    group = SettingGroup('example')

    foo = BoolSetting('use the foo')
    bar = BoolSetting('enable bars', True)
    opt = EnumSetting('optimization level', 'Debug', 'Release')

    group.close(globals())


Instruction descriptions
========================

New instructions are defined as instances of the :class:`Instruction`
class. As instruction instances are created, they are added to the currently
open :class:`InstructionGroup`.

.. autoclass:: InstructionGroup
    :members:

The basic Cretonne instruction set described in :doc:`langref` is defined by the
Python module :mod:`cretonne.base`. This module has a global variable
:data:`cretonne.base.instructions` which is an :class:`InstructionGroup`
instance containing all the base instructions.

.. autoclass:: Instruction

An instruction is defined with a set of distinct input and output operands which
must be instances of the :class:`Operand` class.

.. autoclass:: Operand

Cretonne uses two separate type systems for operand kinds and SSA values.

Type variables
--------------

Instruction descriptions can be made polymorphic by using :class:`Operand`
instances that refer to a *type variable* instead of a concrete value type.
Polymorphism only works for SSA value operands. Other operands have a fixed
operand kind.

.. autoclass:: cretonne.typevar.TypeVar
    :members:

If multiple operands refer to the same type variable they will be required to
have the same concrete type. For example, this defines an integer addition
instruction::

    Int = TypeVar('Int', 'A scalar or vector integer type', ints=True, simd=True)
    a = Operand('a', Int)
    x = Operand('x', Int)
    y = Operand('y', Int)

    iadd = Instruction('iadd', 'Integer addition', ins=(x, y), outs=a)

The type variable `Int` is allowed to vary over all scalar and vector integer
value types, but in a given instance of the `iadd` instruction, the two
operands must have the same type, and the result will be the same type as the
inputs.

There are some practical restrictions on the use of type variables, see
:ref:`restricted-polymorphism`.

Immediate operands
------------------

Immediate instruction operands don't correspond to SSA values, but have values
that are encoded directly in the instruction. Immediate operands don't
have types from the :class:`cretonne.ValueType` type system; they often have
enumerated values of a specific type. The type of an immediate operand is
indicated with an instance of :class:`ImmediateKind`.

.. autoclass:: ImmediateKind

.. automodule:: cretonne.immediates
    :members:

.. currentmodule:: cretonne

Entity references
-----------------

Instruction operands can also refer to other entties in the same function. This
can be extended basic blocks, or entities declared in the function preamble.

.. autoclass:: EntityRefKind

.. automodule:: cretonne.entities
    :members:

.. currentmodule:: cretonne

Value types
-----------

Concrete value types are represented as instances of :class:`cretonne.ValueType`. There are
subclasses to represent scalar and vector types.

.. autoclass:: ValueType
.. inheritance-diagram:: ValueType ScalarType VectorType IntType FloatType BoolType
    :parts: 1
.. autoclass:: ScalarType
    :members:
.. autoclass:: VectorType
    :members:
.. autoclass:: IntType
    :members:
.. autoclass:: FloatType
    :members:
.. autoclass:: BoolType
    :members:

.. automodule:: cretonne.types
    :members:

.. currentmodule:: cretonne

There are no predefined vector types, but they can be created as needed with
the :func:`ScalarType.by` function.


Instruction representation
==========================

The Rust in-memory representation of instructions is derived from the
instruction descriptions. Part of the representation is generated, and part is
written as Rust code in the `cretonne.instructions` module. The instruction
representation depends on the input operand kinds and whether the instruction
can produce multiple results.

.. autoclass:: OperandKind
.. inheritance-diagram:: OperandKind ImmediateKind EntityRefKind

Since all SSA value operands are represented as a `Value` in Rust code, value
types don't affect the representation. Two special operand kinds are used to
represent SSA values:

.. autodata:: value
.. autodata:: variable_args

When an instruction description is created, it is automatically assigned a
predefined instruction format which is an instance of
:class:`InstructionFormat`:

.. autoclass:: InstructionFormat


.. _restricted-polymorphism:

Restricted polymorphism
-----------------------

The instruction format strictly controls the kinds of operands on an
instruction, but it does not constrain value types at all. A given instruction
description typically does constrain the allowed value types for its value
operands. The type variables give a lot of freedom in describing the value type
constraints, in practice more freedom than what is needed for normal instruction
set architectures. In order to simplify the Rust representation of value type
constraints, some restrictions are imposed on the use of type variables.

A polymorphic instruction has a single *controlling type variable*. For a given
opcode, this type variable must be the type of the first result or the type of
the input value operand designated by the `typevar_operand` argument to the
:py:class:`InstructionFormat` constructor. By default, this is the first value
operand, which works most of the time.

The value types of instruction results must be one of the following:

1. A concrete value type.
2. The controlling type variable.
3. A type variable derived from the controlling type variable.

This means that all result types can be computed from the controlling type
variable.

Input values to the instruction are allowed a bit more freedom. Input value
types must be one of:

1. A concrete value type.
2. The controlling type variable.
3. A type variable derived from the controlling type variable.
4. A free type variable that is not used by any other operands.

This means that the type of an input operand can either be computed from the
controlling type variable, or it can vary independently of the other operands.


Encodings
=========

Encodings describe how Cretonne instructions are mapped to binary machine code
for the target architecture. After the lealization pass, all remaining
instructions are expected to map 1-1 to native instruction encodings. Cretonne
instructions that can't be encoded for the current architecture are called
:term:`illegal instruction`\s.

Some instruction set architectures have different :term:`CPU mode`\s with
incompatible encodings. For example, a modern ARMv8 CPU might support three
different CPU modes: *A64* where instructions are encoded in 32 bits, *A32*
where all instuctions are 32 bits, and *T32* which has a mix of 16-bit and
32-bit instruction encodings. These are incompatible encoding spaces, and while
an :cton:inst:`iadd` instruction can be encoded in 32 bits in each of them, it's
not the same 32 bits. It's a judgement call if CPU modes should be modelled as
separate targets, or as sub-modes of the same target. In the ARMv8 case, the
different register banks means that it makes sense to model A64 as a separate
target architecture, while A32 and T32 are CPU modes of the 32-bit ARM target.

In a given CPU mode, there may be multiple valid encodings of the same
instruction. Both RISC-V and ARMv8's T32 mode have 32-bit encodings of all
instructions with 16-bit encodings available for some opcodes if certain
constraints are satisfied.

.. autoclass:: CPUMode

Encodings are guarded by :term:`sub-target predicate`\s. For example, the RISC-V
"C" extension which specifies the compressed encodings may not be supported, and
a predicate would be used to disable all of the 16-bit encodings in that case.
This can also affect whether an instruction is legal. For example, x86 has a
predicate that controls the SSE 4.1 instruction encodings. When that predicate
is false, the SSE 4.1 instructions are not available.

Encodings also have a :term:`instruction predicate` which depends on the
specific values of the instruction's immediate fields. This is used to ensure
that immediate address offsets are within range, for example. The instructions
in the base Cretonne instruction set can often represent a wider range of
immediates than any specific encoding. The fixed-size RISC-style encodings tend
to have more range limitations than CISC-style variable length encodings like
x86.

The diagram below shows the relationship between the classes involved in
specifying instruction encodings:

.. digraph:: encoding

    node [shape=record]
    EncRecipe -> SubtargetPred
    EncRecipe -> InstrFormat
    EncRecipe -> InstrPred
    Encoding [label="{Encoding|Opcode+TypeVars}"]
    Encoding -> EncRecipe [label="+EncBits"]
    Encoding -> CPUMode
    Encoding -> SubtargetPred
    Encoding -> InstrPred
    Encoding -> Opcode
    Opcode -> InstrFormat
    CPUMode -> Target

An :py:class:`Encoding` instance specifies the encoding of a concrete
instruction. The following properties are used to select instructions to be
encoded:

- An opcode, i.e. :cton:inst:`iadd_imm`, that must match the instruction's
  opcode.
- Values for any type variables if the opcode represents a polymorphic
  instruction.
- An :term:`instruction predicate` that must be satisfied by the instruction's
  immediate operands.
- The CPU mode that must be active.
- A :term:`sub-target predicate` that must be satisfied by the currently active
  sub-target.
- :term:`Register constraint`\s that must be satisfied by the instruction's value
  operands and results.

An encoding specifies an *encoding recipe* along with some *encoding bits* that
the recipe can use for native opcode fields etc. The encoding recipe has
additional constraints that must be satisfied:

- An :py:class:`InstructionFormat` that must match the format required by the
  opcodes of any encodings that use this recipe.
- An additional :term:`instruction predicate`.
- An additional :term:`sub-target predicate`.

The additional predicates in the :py:class:`EncRecipe` are merged with the
per-encoding predicates when generating the encoding matcher code. Often
encodings only need the recipe predicates.

.. autoclass:: EncRecipe


Targets
=======

Cretonne can be compiled with support for multiple target instruction set
architectures. Each ISA is represented by a :py:class:`cretonne.TargetISA` instance.

.. autoclass:: TargetISA

The definitions for each supported target live in a package under
:file:`lib/cretonne/meta/isa`.

.. automodule:: isa
    :members:

.. automodule:: isa.riscv


Glossary
========

.. glossary::

    Illegal instruction
        An instruction is considered illegal if there is no encoding available
        for the current CPU mode. The legality of an instruction depends on the
        value of :term:`sub-target predicate`\s, so it can't always be
        determined ahead of time.

    CPU mode
        Every target defines one or more CPU modes that determine how the CPU
        decodes binary instructions. Some CPUs can switch modes dynamically with
        a branch instruction (like ARM/Thumb), while other modes are
        process-wide (like x86 32/64-bit).

    Sub-target predicate
        A predicate that depends on the current sub-target configuration.
        Examples are "Use SSE 4.1 instructions", "Use RISC-V compressed
        encodings". Sub-target predicates can depend on both detected CPU
        features and configuration settings.

    Instruction predicate
        A predicate that depends on the immediate fields of an instruction. An
        example is "the load address offset must be a 10-bit signed integer".
        Instruction predicates do not depend on the registers selected for value
        operands.

    Register constraint
        Value operands and results correspond to machine registers. Encodings may
        constrain operands to either a fixed register or a register class. There
        may also be register constraints between operands, for example some
        encodings require that the result register is one of the input
        registers.
