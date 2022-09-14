.. contents::
.. sectnum::

====================
eBPF Instruction Set
====================

The eBPF instruction set consists of eleven 64 bit registers, a program counter,
and 512 bytes of stack space.

Versions
========

The current Instruction Set Architecture (ISA) version, sometimes referred to in other documents
as a "CPU" version, is 3.  This document also covers older versions of the ISA.

   **Note**

   *Clang implementation*: Clang can select the eBPF ISA version using
   ``-mcpu=v2`` for example to select version 2.

Registers and calling convention
================================

eBPF has 10 general purpose registers and a read-only frame pointer register,
all of which are 64-bits wide.

The eBPF calling convention is defined as:

* R0: return value from function calls, and exit value for eBPF programs
* R1 - R5: arguments for function calls
* R6 - R9: callee saved registers that function calls will preserve
* R10: read-only frame pointer to access stack

Registers R0 - R5 are scratch registers, meaning the BPF program needs to either
spill them to the BPF stack or move them to callee saved registers if these
arguments are to be reused across multiple function calls. Spilling means
that the value in the register is moved to the BPF stack. The reverse operation
of moving the variable from the BPF stack to the register is called filling.
The reason for spilling/filling is due to the limited number of registers.

   **Note**

   *Linux implementation*: In the Linux kernel, the exit value for eBPF
   programs is passed as a 32 bit value.

Upon entering execution of an eBPF program, registers R1 - R5 initially can contain
the input arguments for the program (similar to the argc/argv pair for a typical C program).
The actual number of registers used, and their meaning, is defined by the program type;
for example, a networking program might have an argument that includes network packet data
and/or metadata.

   **Note**

   *Linux implementation*: In the Linux kernel, all program types only use
   R1 which contains the "context", which is typically a structure containing all
   the inputs needed.  

Instruction encoding
====================

An eBPF program is a sequence of instructions.

eBPF has two instruction encodings:

* the basic instruction encoding, which uses 64 bits to encode an instruction
* the wide instruction encoding, which appends a second 64-bit immediate (i.e.,
  constant) value after the basic instruction for a total of 128 bits.

The basic instruction encoding is as follows, where MSB and LSB mean the most significant
bits and least significant bits, respectively:

=============  =======  ===============  ====================  ============
32 bits (MSB)  16 bits  4 bits           4 bits                8 bits (LSB)
=============  =======  ===============  ====================  ============
imm            offset   src              dst                   opcode
=============  =======  ===============  ====================  ============

imm         
  signed integer immediate value

offset
  signed integer offset used with pointer arithmetic

src
  the source register number (0-10), except where otherwise specified
  (`64-bit immediate instructions`_ reuse this field for other purposes)

dst
  destination register number (0-10)

opcode
  operation to perform

Note that most instructions do not use all of the fields.
Unused fields must be set to zero.

As discussed below in `64-bit immediate instructions`_, some
instructions use a 64-bit immediate value that is constructed as follows.
The 64 bits following the basic instruction contain a pseudo instruction
using the same format but with opcode, dst, src, and offset all set to zero,
and imm containing the high 32 bits of the immediate value.

=================  ==================
64 bits (MSB)      64 bits (LSB)
=================  ==================
basic instruction  pseudo instruction
=================  ==================

Thus the 64-bit immediate value is constructed as follows:

  imm64 = imm + (next_imm << 32)

where 'next_imm' refers to the imm value of the pseudo instruction
following the basic instruction.

In the remainder of this document 'src' and 'dst' refer to the values of the source
and destination registers, respectively, rather than the register number.

Instruction classes
-------------------

The encoding of the 'opcode' field varies and can be determined from
the three least significant bits (LSB) of the 'opcode' field which holds
the "instruction class", as follows:

=========  =====  ===============================  =======  =================
class      value  description                      version  reference
=========  =====  ===============================  =======  =================
BPF_LD     0x00   non-standard load operations     1        `Load and store instructions`_
BPF_LDX    0x01   load into register operations    1        `Load and store instructions`_
BPF_ST     0x02   store from immediate operations  1        `Load and store instructions`_
BPF_STX    0x03   store from register operations   1        `Load and store instructions`_
BPF_ALU    0x04   32-bit arithmetic operations     3        `Arithmetic and jump instructions`_
BPF_JMP    0x05   64-bit jump operations           1        `Arithmetic and jump instructions`_
BPF_JMP32  0x06   32-bit jump operations           3        `Arithmetic and jump instructions`_
BPF_ALU64  0x07   64-bit arithmetic operations     1        `Arithmetic and jump instructions`_
=========  =====  ===============================  =======  =================

where 'version' indicates the first ISA version in which support for the value was mandatory.

Arithmetic and jump instructions
================================

For arithmetic and jump instructions (``BPF_ALU``, ``BPF_ALU64``, ``BPF_JMP`` and
``BPF_JMP32``), the 8-bit 'opcode' field is divided into three parts:

==============  ======  =================
4 bits (MSB)    1 bit   3 bits (LSB)
==============  ======  =================
code            source  instruction class
==============  ======  =================

code
  the operation code, whose meaning varies by instruction class

source
  the source operand location, which unless otherwise specified is one of:

  ======  =====  ========================================
  source  value  description
  ======  =====  ========================================
  BPF_K   0x00   use 32-bit 'imm' value as source operand
  BPF_X   0x08   use 'src' register value as source operand
  ======  =====  ========================================

instruction class
  the instruction class (see `Instruction classes`_)

Arithmetic instructions
-----------------------

Instruction class ``BPF_ALU`` uses 32-bit wide operands (zeroing the upper 32 bits
of the destination register) while ``BPF_ALU64`` uses 64-bit wide operands for
otherwise identical operations.

Support for ``BPF_ALU`` is required in ISA version 3, and optional in earlier
versions.

   **Note**

   *Clang implementation*:
   For ISA versions prior to 3, Clang v7.0 and later can enable ``BPF_ALU`` support with
   ``-Xclang -target-feature -Xclang +alu32``.

The 4-bit 'code' field encodes the operation as follows:

========  =====  =================================================
code      value  description
========  =====  =================================================
BPF_ADD   0x00   dst += src
BPF_SUB   0x10   dst -= src
BPF_MUL   0x20   dst \*= src
BPF_DIV   0x30   dst = (src != 0) ? (dst / src) : 0
BPF_OR    0x40   dst \|= src
BPF_AND   0x50   dst &= src
BPF_LSH   0x60   dst <<= src
BPF_RSH   0x70   dst >>= src
BPF_NEG   0x80   dst = ~src
BPF_MOD   0x90   dst = (src != 0) ? (dst % src) : src
BPF_XOR   0xa0   dst ^= src
BPF_MOV   0xb0   dst = src
BPF_ARSH  0xc0   sign extending shift right
BPF_END   0xd0   byte swap operations (see `Byte swap instructions`_ below)
========  =====  =================================================

where 'src' is the source operand value.

Underflow and overflow are allowed during arithmetic operations,
meaning the 64-bit or 32-bit value will wrap.  If
eBPF program execution would result in division by zero,
the destination register is instead set to zero.
If execution would result in module by zero,
the destination register is instead set to the source value.

Examples:

``BPF_ADD | BPF_X | BPF_ALU`` (0x0c) means::

  dst = (uint32_t) (dst + src)

where '(uint32_t)' indicates truncation to 32 bits.

   **Note**

   *Linux implementation*: In the Linux kernel, uint32_t is expressed as u32,
   uint64_t is expressed as u64, etc.  This document uses the standard C terminology
   as the cross-platform specification.

``BPF_ADD | BPF_X | BPF_ALU64`` (0x0f) means::

  dst = dst + src

``BPF_XOR | BPF_K | BPF_ALU`` (0xa4) means::

  src = (uint32_t) src ^ (uint32_t) imm

``BPF_XOR | BPF_K | BPF_ALU64`` (0xa7) means::

  src = src ^ imm


Byte swap instructions
~~~~~~~~~~~~~~~~~~~~~~

The byte swap instructions use an instruction class of ``BPF_ALU`` and a 4-bit
'code' field of ``BPF_END``.

The byte swap instructions operate on the destination register
only and do not use a separate source register or immediate value.

Byte swap instructions use non-default semantics of the 1-bit 'source' field in
the 'opcode' field.  Instead of indicating the source operator, it is instead
used to select what byte order the operation converts from or to:

=========  =====  =================================================
source     value  description
=========  =====  =================================================
BPF_TO_LE  0x00   convert between host byte order and little endian
BPF_TO_BE  0x08   convert between host byte order and big endian
=========  =====  =================================================

   **Note**

   *Linux implementation*:
   ``BPF_FROM_LE`` and ``BPF_FROM_BE`` exist as aliases for ``BPF_TO_LE`` and
   ``BPF_TO_BE`` respectively.

The 'imm' field encodes the width of the swap operations.  The following widths
are supported: 16, 32 and 64. The following table summarizes the resulting
possibilities:

=============================  =========  ===  ========  ==================
opcode construction            opcode     imm  mnemonic  pseudocode
=============================  =========  ===  ========  ==================
BPF_END | BPF_TO_LE | BPF_ALU  0xd4       16   le16 dst  dst = htole16(dst)
BPF_END | BPF_TO_LE | BPF_ALU  0xd4       32   le32 dst  dst = htole32(dst)
BPF_END | BPF_TO_LE | BPF_ALU  0xd4       64   le64 dst  dst = htole64(dst)
BPF_END | BPF_TO_BE | BPF_ALU  0xdc       16   be16 dst  dst = htobe16(dst)
BPF_END | BPF_TO_BE | BPF_ALU  0xdc       32   be32 dst  dst = htobe32(dst)
BPF_END | BPF_TO_BE | BPF_ALU  0xdc       64   be64 dst  dst = htobe64(dst)
=============================  =========  ===  ========  ==================

where

* mnenomic indicates a short form that might be displayed by some tools such as disassemblers
* 'htoleNN()' indicates converting a NN-bit value from host byte order to little-endian byte order
* 'htobeNN()' indicates converting a NN-bit value from host byte order to big-endian byte order

Jump instructions
-----------------

Instruction class ``BPF_JMP32`` uses 32-bit wide operands while ``BPF_JMP`` uses 64-bit wide operands for
otherwise identical operations.

Support for ``BPF_JMP32`` is required in ISA version 3, and optional in earlier
versions.

The 4-bit 'code' field encodes the operation as below, where PC is the program counter:

========  =====  ============================  =======  ============
code      value  description                   version  notes
========  =====  ============================  =======  ============
BPF_JA    0x00   PC += offset                  1        BPF_JMP only
BPF_JEQ   0x10   PC += offset if dst == src    1
BPF_JGT   0x20   PC += offset if dst > src     1        unsigned
BPF_JGE   0x30   PC += offset if dst >= src    1        unsigned
BPF_JSET  0x40   PC += offset if dst & src     1
BPF_JNE   0x50   PC += offset if dst != src    1
BPF_JSGT  0x60   PC += offset if dst > src     1        signed
BPF_JSGE  0x70   PC += offset if dst >= src    1        signed
BPF_CALL  0x80   call function imm             1        see `Helper functions`_
BPF_EXIT  0x90   function / program return     1        BPF_JMP only
BPF_JLT   0xa0   PC += offset if dst < src     2        unsigned
BPF_JLE   0xb0   PC += offset if dst <= src    2        unsigned
BPF_JSLT  0xc0   PC += offset if dst < src     2        signed
BPF_JSLE  0xd0   PC += offset if dst <= src    2        signed
========  =====  ============================  =======  ============

where 'version' indicates the first ISA version in which the value was supported.

Helper functions
~~~~~~~~~~~~~~~~
Helper functions are a concept whereby BPF programs can call into
set of function calls exposed by the eBPF runtime.  Each helper
function is identified by an integer used in a ``BPF_CALL`` instruction.
The available helper functions may differ for each eBPF program type.

Conceptually, each helper function is implemented with a commonly shared function
signature defined as:

  uint64_t function(uint64_t r1, uint64_t r2, uint64_t r3, uint64_t r4, uint64_t r5)

In actuality, each helper function is defined as taking between 0 and 5 arguments,
with the remaining registers being ignored.  The definition of a helper function
is responsible for specifying the type (e.g., integer, pointer, etc.) of the value returned,
the number of arguments, and the type of each argument.

Note that ``BPF_CALL | BPF_X | BPF_JMP`` (0x8d), where the helper function integer
would be read from a specified register, is not currently permitted.

   **Note**

   *Clang implementation*:
   Clang will generate this invalid instruction if ``-O0`` is used.

Load and store instructions
===========================

For load and store instructions (``BPF_LD``, ``BPF_LDX``, ``BPF_ST``, and ``BPF_STX``), the
8-bit 'opcode' field is divided as:

============  ======  =================
3 bits (MSB)  2 bits  3 bits (LSB)
============  ======  =================
mode          size    instruction class
============  ======  =================

mode
  one of:

  =============  =====  ====================================  =============
  mode modifier  value  description                           reference
  =============  =====  ====================================  =============
  BPF_IMM        0x00   64-bit immediate instructions         `64-bit immediate instructions`_
  BPF_ABS        0x20   legacy BPF packet access (absolute)   `Legacy BPF Packet access instructions`_
  BPF_IND        0x40   legacy BPF packet access (indirect)   `Legacy BPF Packet access instructions`_
  BPF_MEM        0x60   regular load and store operations     `Regular load and store operations`_
  BPF_ATOMIC     0xc0   atomic operations                     `Atomic operations`_
  =============  =====  ====================================  =============

size
  one of:

  =============  =====  =====================
  size modifier  value  description
  =============  =====  =====================
  BPF_W          0x00   word        (4 bytes)
  BPF_H          0x08   half word   (2 bytes)
  BPF_B          0x10   byte
  BPF_DW         0x18   double word (8 bytes)
  =============  =====  =====================

instruction class
  the instruction class (see `Instruction classes`_)

Regular load and store operations
---------------------------------

The ``BPF_MEM`` mode modifier is used to encode regular load and store
instructions that transfer data between a register and memory.

=============================  =========  ==================================
opcode construction            opcode     pseudocode
=============================  =========  ==================================
BPF_MEM | BPF_B | BPF_LDX      0x71       dst = *(uint8_t *) (src + offset)  
BPF_MEM | BPF_H | BPF_LDX      0x69       dst = *(uint16_t *) (src + offset)
BPF_MEM | BPF_W | BPF_LDX      0x61       dst = *(uint32_t *) (src + offset)
BPF_MEM | BPF_DW | BPF_LDX     0x79       dst = *(uint64_t *) (src + offset)
BPF_MEM | BPF_B | BPF_ST       0x72       *(uint8_t *) (dst + offset) = imm
BPF_MEM | BPF_H | BPF_ST       0x6a       *(uint16_t *) (dst + offset) = imm
BPF_MEM | BPF_W | BPF_ST       0x62       *(uint32_t *) (dst + offset) = imm
BPF_MEM | BPF_DW | BPF_ST      0x7a       *(uint64_t *) (dst + offset) = imm
BPF_MEM | BPF_B | BPF_STX      0x73       *(uint8_t *) (dst + offset) = src
BPF_MEM | BPF_H | BPF_STX      0x6b       *(uint16_t *) (dst + offset) = src
BPF_MEM | BPF_W | BPF_STX      0x63       *(uint32_t *) (dst + offset) = src
BPF_MEM | BPF_DW | BPF_STX     0x7b       *(uint64_t *) (dst + offset) = src
=============================  =========  ==================================

Atomic operations
-----------------

Atomic operations are operations that operate on memory and can not be
interrupted or corrupted by other access to the same memory region
by other eBPF programs or means outside of this specification.

All atomic operations supported by eBPF are encoded as store operations
that use the ``BPF_ATOMIC`` mode modifier as follows:

* ``BPF_ATOMIC | BPF_W | BPF_STX`` (0xc3) for 32-bit operations
* ``BPF_ATOMIC | BPF_DW | BPF_STX`` (0xdb) for 64-bit operations

Note that 8-bit (``BPF_B``) and 16-bit (``BPF_H``) wide atomic operations are not supported,
nor is ``BPF_ATOMIC | <size> | BPF_ST``.

The 'imm' field is used to encode the actual atomic operation.
Simple atomic operation use a subset of the values defined to encode
arithmetic operations in the 'imm' field to encode the atomic operation:

========  =====  ===========  =======
imm       value  description  version
========  =====  ===========  =======
BPF_ADD   0x00   atomic add   1
BPF_OR    0x40   atomic or    3
BPF_AND   0x50   atomic and   3
BPF_XOR   0xa0   atomic xor   3
========  =====  ===========  =======

where 'version' indicates the first ISA version in which the value was supported.

``BPF_ATOMIC | BPF_W  | BPF_STX`` (0xc3) with 'imm' = BPF_ADD means::

  *(uint32_t *)(dst + offset) += src

``BPF_ATOMIC | BPF_DW | BPF_STX`` (0xdb) with 'imm' = BPF ADD means::

  *(uint64_t *)(dst + offset) += src

``BPF_XADD`` appeared in version 1, but is now considered to be a deprecated alias
for ``BPF_ATOMIC | BPF_ADD``.

In addition to the simple atomic operations above, there also is a modifier and
two complex atomic operations:

===========  ================  ===========================  =======
imm          value             description                  version
===========  ================  ===========================  =======
BPF_FETCH    0x01              modifier: return old value   3
BPF_XCHG     0xe0 | BPF_FETCH  atomic exchange              3
BPF_CMPXCHG  0xf0 | BPF_FETCH  atomic compare and exchange  3
===========  ================  ===========================  =======

The ``BPF_FETCH`` modifier is optional for simple atomic operations, and
always set for the complex atomic operations.  If the ``BPF_FETCH`` flag
is set, then the operation also overwrites ``src`` with the value that
was in memory before it was modified.

The ``BPF_XCHG`` operation atomically exchanges ``src`` with the value
addressed by ``dst + offset``.

The ``BPF_CMPXCHG`` operation atomically compares the value addressed by
``dst + offset`` with ``R0``. If they match, the value addressed by
``dst + offset`` is replaced with ``src``. In either case, the
value that was at ``dst + offset`` before the operation is zero-extended
and loaded back to ``R0``.

   **Note**

   *Clang implementation*:
   Clang can generate atomic instructions by default when ``-mcpu=v3`` is
   enabled. If a lower version for ``-mcpu`` is set, the only atomic instruction
   Clang can generate is ``BPF_ADD`` *without* ``BPF_FETCH``. If you need to enable
   the atomics features, while keeping a lower ``-mcpu`` version, you can use
   ``-Xclang -target-feature -Xclang +alu32``.

64-bit immediate instructions
-----------------------------

Instructions with the ``BPF_IMM`` 'mode' modifier use the wide instruction
encoding defined in `Instruction encoding`_, and use the 'src' field of the
basic instruction to hold an opcode subtype.

The following instructions are defined, and use additional concepts defined below:

=========================  ======  ====  =====================================  ===========  ==============
opcode construction        opcode  src   pseudocode                             imm type     dst type
=========================  ======  ====  =====================================  ===========  ==============
BPF_IMM | BPF_DW | BPF_LD  0x18    0x00  dst = imm64                            integer      integer
BPF_IMM | BPF_DW | BPF_LD  0x18    0x01  dst = map_by_fd(imm)                   map fd       map
BPF_IMM | BPF_DW | BPF_LD  0x18    0x02  dst = mva(map_by_fd(imm)) + next_imm   map fd       data pointer
BPF_IMM | BPF_DW | BPF_LD  0x18    0x03  dst = variable_addr(imm)               variable id  data pointer
BPF_IMM | BPF_DW | BPF_LD  0x18    0x04  dst = code_addr(imm)                   integer      code pointer  
BPF_IMM | BPF_DW | BPF_LD  0x18    0x05  dst = mva(map_by_idx(imm))             map index    map
BPF_IMM | BPF_DW | BPF_LD  0x18    0x06  dst = mva(map_by_idx(imm)) + next_imm  map index    data pointer
=========================  ======  ====  =====================================  ===========  ==============

where

* map_by_fd(fd) means to convert a 32-bit POSIX file descriptor into an address of a map object (see `Map objects`_)
* map_by_index(index) means to convert a 32-bit index into an address of a map object
* mva(map) gets the address of the memory region expressed by a given map object
* variable_addr(id) gets the address of a variable (see `Variables`_) with a given id
* code_addr(offset) gets the address of the instruction at a specified relative offset in units of 64-bit blocks
* the 'imm type' can be used by disassemblers for display
* the 'dst type' can be used for verification and JIT compilation purposes

Map objects
~~~~~~~~~~~

Maps are shared memory regions accessible by eBPF programs, where we use the term "map object"
to refer to an object containing the data and metadata (e.g., size) about the memory region.
A map can have various semantics as defined in a separate document, and may or may not have a single
contiguous memory region, but the 'mva(map)' is currently only defined for maps that do have a single
contiguous memory region.

   **Note**

   *Linux implementation*: Linux only supports the 'mva(map)' operation on array maps with a single element.

Each map object can have a POSIX file descriptor (fd) if supported by the platform,
where 'map_by_fd(fd)' means to get the map with the specified file descriptor.
Each eBPF program can also be defined to use a set of maps associated with the program
at load time, and 'map_by_index(index)' means to get the map with the given index in the set
associated with the eBPF program containing the instruction.

Variables
~~~~~~~~~

Variables are memory regions, identified by integer ids, accessible by eBPF programs on
some platforms.  The 'variable_addr(id)' operation means to get the address of the memory region
identified by the given id.

   **Note**

   *Linux implementation*: Linux uses BTF ids to identify variables.

Legacy BPF Packet access instructions
-------------------------------------

Linux introduced special instructions for access to packet data that were
carried over from classic BPF. However, these instructions are
deprecated and should no longer be used in any version of the ISA.

   **Note**

   *Linux implementation*: Details can be found in the `Linux historical notes <https://github.com/dthaler/ebpf-docs/blob/update/isa/kernel.org/linux-historical-notes.rst#legacy-bpf-packet-access-instructions>`_.

Appendix
========

For reference, the following table lists opcodes in order by value.

======  ====  ====  ===================================================  ========================================
opcode  imm   src   description                                          reference 
======  ====  ====  ===================================================  ========================================
0x00    any   0x00  (additional immediate value)                         `64-bit immediate instructions`_
0x04    any   0x00  dst = (uint32_t)(dst + imm)                          `Arithmetic instructions`_
0x05    0x00  0x00  goto +offset                                         `Jump instructions`_
0x07    any   0x00  dst += imm                                           `Arithmetic instructions`_
0x0c    0x00  any   dst = (uint32_t)(dst + src)                          `Arithmetic instructions`_
0x0f    0x00  any   dst += src                                           `Arithmetic instructions`_
0x14    any   0x00  dst = (uint32_t)(dst - imm)                          `Arithmetic instructions`_
0x15    any   0x00  if dst == imm goto +offset                           `Jump instructions`_
0x16    any   0x00  if (uint32_t)dst == imm goto +offset                 `Jump instructions`_
0x17    any   0x00  dst -= imm                                           `Arithmetic instructions`_
0x18    0x00  0x00  dst = imm64                                          `64-bit immediate instructions`_
0x18    0x00  0x01  dst = map_by_fd(imm)                                 `64-bit immediate instructions`_
0x18    0x00  0x02  dst = mva(map_by_fd(imm)) + next_imm                 `64-bit immediate instructions`_
0x18    0x00  0x03  dst = variable_addr(imm)                             `64-bit immediate instructions`_
0x18    0x00  0x04  dst = code_addr(imm)                                 `64-bit immediate instructions`_
0x18    0x00  0x05  dst = mva(map_by_idx(imm))                           `64-bit immediate instructions`_
0x18    0x00  0x06  dst = mva(map_by_idx(imm)) + next_imm                `64-bit immediate instructions`_
0x1c    0x00  any   dst = (uint32_t)(dst - src)                          `Arithmetic instructions`_
0x1d    0x00  any   if dst == src goto +offset                           `Jump instructions`_
0x1e    0x00  any   if (uint32_t)dst == (uint32_t)src goto +offset       `Jump instructions`_
0x1f    0x00  any   dst -= src                                           `Arithmetic instructions`_
0x20    any   any   (deprecated, implementation-specific)                `Legacy BPF Packet access instructions`_
0x24    any   0x00  dst = (uint32_t)(dst \* imm)                         `Arithmetic instructions`_
0x25    any   0x00  if dst > imm goto +offset                            `Jump instructions`_
0x26    any   0x00  if (uint32_t)dst > imm goto +offset                  `Jump instructions`_
0x27    any   0x00  dst \*= imm                                          `Arithmetic instructions`_
0x28    any   any   (deprecated, implementation-specific)                `Legacy BPF Packet access instructions`_
0x2c    0x00  any   dst = (uint32_t)(dst \* src)                         `Arithmetic instructions`_
0x2d    0x00  any   if dst > src goto +offset                            `Jump instructions`_
0x2e    0x00  any   if (uint32_t)dst > (uint32_t)src goto +offset        `Jump instructions`_
0x2f    0x00  any   dst \*= src                                          `Arithmetic instructions`_
0x30    any   any   (deprecated, implementation-specific)                `Legacy BPF Packet access instructions`_
0x34    any   0x00  dst = (uint32_t)((imm != 0) ? (dst / imm) : 0)       `Arithmetic instructions`_
0x35    any   0x00  if dst >= imm goto +offset                           `Jump instructions`_
0x36    any   0x00  if (uint32_t)dst >= imm goto +offset                 `Jump instructions`_
0x37    any   0x00  dst = (imm != 0) ? (dst / imm) : 0                   `Arithmetic instructions`_
0x38    any   any   (deprecated, implementation-specific)                `Legacy BPF Packet access instructions`_
0x3c    0x00  any   dst = (uint32_t)((imm != 0) ? (dst / src) : 0)       `Arithmetic instructions`_
0x3d    0x00  any   if dst >= src goto +offset                           `Jump instructions`_
0x3e    0x00  any   if (uint32_t)dst >= (uint32_t)src goto +offset       `Jump instructions`_
0x3f    0x00  any   dst = (src !+ 0) ? (dst / src) : 0                   `Arithmetic instructions`_
0x40    any   any   (deprecated, implementation-specific)                `Legacy BPF Packet access instructions`_
0x44    any   0x00  dst = (uint32_t)(dst \| imm)                         `Arithmetic instructions`_
0x45    any   0x00  if dst & imm goto +offset                            `Jump instructions`_
0x46    any   0x00  if (uint32_t)dst & imm goto +offset                  `Jump instructions`_
0x47    any   0x00  dst \|= imm                                          `Arithmetic instructions`_
0x48    any   any   (deprecated, implementation-specific)                `Legacy BPF Packet access instructions`_
0x4c    0x00  any   dst = (uint32_t)(dst \| src)                         `Arithmetic instructions`_
0x4d    0x00  any   if dst & src goto +offset                            `Jump instructions`_
0x4e    0x00  any   if (uint32_t)dst & (uint32_t)src goto +offset        `Jump instructions`_
0x4f    0x00  any   dst \|= src                                          `Arithmetic instructions`_
0x50    any   any   (deprecated, implementation-specific)                `Legacy BPF Packet access instructions`_
0x54    any   0x00  dst = (uint32_t)(dst & imm)                          `Arithmetic instructions`_
0x55    any   0x00  if dst != imm goto +offset                           `Jump instructions`_
0x56    any   0x00  if (uint32_t)dst != imm goto +offset                 `Jump instructions`_
0x57    any   0x00  dst &= imm                                           `Arithmetic instructions`_
0x58    any   any   (deprecated, implementation-specific)                `Legacy BPF Packet access instructions`_
0x5c    0x00  any   dst = (uint32_t)(dst & src)                          `Arithmetic instructions`_
0x5d    0x00  any   if dst != src goto +offset                           `Jump instructions`_
0x5e    0x00  any   if (uint32_t)dst != (uint32_t)src goto +offset       `Jump instructions`_
0x5f    0x00  any   dst &= src                                           `Arithmetic instructions`_
0x61    0x00  any   dst = \*(uint32_t \*)(src + offset)                  `Load and store instructions`_
0x62    any   0x00  \*(uint32_t \*)(dst + offset) = imm                  `Load and store instructions`_
0x63    0x00  any   \*(uint32_t \*)(dst + offset) = src                  `Load and store instructions`_
0x64    any   0x00  dst = (uint32_t)(dst << imm)                         `Arithmetic instructions`_
0x65    any   0x00  if dst s> imm goto +offset                           `Jump instructions`_
0x66    any   0x00  if (int32_t)dst s> (int32_t)imm goto +offset         `Jump instructions`_
0x67    any   0x00  dst <<= imm                                          `Arithmetic instructions`_
0x69    0x00  any   dst = \*(uint16_t \*)(src + offset)                  `Load and store instructions`_
0x6a    any   0x00  \*(uint16_t \*)(dst + offset) = imm                  `Load and store instructions`_
0x6b    0x00  any   \*(uint16_t \*)(dst + offset) = src                  `Load and store instructions`_
0x6c    0x00  any   dst = (uint32_t)(dst << src)                         `Arithmetic instructions`_
0x6d    0x00  any   if dst s> src goto +offset                           `Jump instructions`_
0x6e    0x00  any   if (int32_t)dst s> (int32_t)src goto +offset         `Jump instructions`_
0x6f    0x00  any   dst <<= src                                          `Arithmetic instructions`_
0x71    0x00  any   dst = \*(uint8_t \*)(src + offset)                   `Load and store instructions`_
0x72    any   0x00  \*(uint8_t \*)(dst + offset) = imm                   `Load and store instructions`_
0x73    0x00  any   \*(uint8_t \*)(dst + offset) = src                   `Load and store instructions`_
0x74    any   0x00  dst = (uint32_t)(dst >> imm)                         `Arithmetic instructions`_
0x75    any   0x00  if dst s>= imm goto +offset                          `Jump instructions`_
0x76    any   0x00  if (int32_t)dst s>= (int32_t)imm goto +offset        `Jump instructions`_
0x77    any   0x00  dst >>= imm                                          `Arithmetic instructions`_
0x79    0x00  any   dst = \*(uint64_t \*)(src + offset)                  `Load and store instructions`_
0x7a    any   0x00  \*(uint64_t \*)(dst + offset) = imm                  `Load and store instructions`_
0x7b    0x00  any   \*(uint64_t \*)(dst + offset) = src                  `Load and store instructions`_
0x7c    0x00  any   dst = (uint32_t)(dst >> src)                         `Arithmetic instructions`_
0x7d    0x00  any   if dst s>= src goto +offset                          `Jump instructions`_
0x7e    0x00  any   if (int32_t)dst s>= (int32_t)src goto +offset        `Jump instructions`_
0x7f    0x00  any   dst >>= src                                          `Arithmetic instructions`_
0x84    0x00  0x00  dst = (uint32_t)-dst                                 `Arithmetic instructions`_
0x85    any   0x00  call imm                                             `Jump instructions`_
0x87    0x00  0x00  dst = -dst                                           `Arithmetic instructions`_
0x94    any   0x00  dst = (uint32_t)((imm != 0) ? (dst % imm) : imm)     `Arithmetic instructions`_
0x95    0x00  0x00  return                                               `Jump instructions`_
0x97    any   0x00  dst = (imm != 0) ? (dst % imm) : imm                 `Arithmetic instructions`_
0x9c    0x00  any   dst = (uint32_t)((src != 0) ? (dst % src) : src)     `Arithmetic instructions`_
0x9f    0x00  any   dst = (src != 0) ? (dst % src) : src                 `Arithmetic instructions`_
0xa4    any   0x00  dst = (uint32_t)(dst ^ imm)                          `Arithmetic instructions`_
0xa5    any   0x00  if dst < imm goto +offset                            `Jump instructions`_
0xa6    any   0x00  if (uint32_t)dst < imm goto +offset                  `Jump instructions`_
0xa7    any   0x00  dst ^= imm                                           `Arithmetic instructions`_
0xac    0x00  any   dst = (uint32_t)(dst ^ src)                          `Arithmetic instructions`_
0xad    0x00  any   if dst < src goto +offset                            `Jump instructions`_
0xae    0x00  any   if (uint32_t)dst < (uint32_t)src goto +offset        `Jump instructions`_
0xaf    0x00  any   dst ^= src                                           `Arithmetic instructions`_
0xb4    any   0x00  dst = (uint32_t) imm                                 `Arithmetic instructions`_
0xb5    any   0x00  if dst <= imm goto +offset                           `Jump instructions`_
0xa6    any   0x00  if (uint32_t)dst <= imm goto +offset                 `Jump instructions`_
0xb7    any   0x00  dst = imm                                            `Arithmetic instructions`_
0xbc    0x00  any   dst = (uint32_t) src                                 `Arithmetic instructions`_
0xbd    0x00  any   if dst <= src goto +offset                           `Jump instructions`_
0xbe    0x00  any   if (uint32_t)dst <= (uint32_t)src goto +offset       `Jump instructions`_
0xbf    0x00  any   dst = src                                            `Arithmetic instructions`_
0xc3    0x00  any   lock \*(uint32_t \*)(dst + offset) += src            `Atomic operations`_
0xc3    0x01  any   lock::                                               `Atomic operations`_

                        *(uint32_t *)(dst + offset) += src
                        src = *(uint32_t *)(dst + offset)
0xc3    0x40  any   \*(uint32_t \*)(dst + offset) \|= src                `Atomic operations`_
0xc3    0x41  any   lock::                                               `Atomic operations`_

                        *(uint32_t *)(dst + offset) |= src
                        src = *(uint32_t *)(dst + offset)
0xc3    0x50  any   \*(uint32_t \*)(dst + offset) &= src                 `Atomic operations`_
0xc3    0x51  any   lock::                                               `Atomic operations`_

                        *(uint32_t *)(dst + offset) &= src
                        src = *(uint32_t *)(dst + offset)
0xc3    0xa0  any   \*(uint32_t \*)(dst + offset) ^= src                 `Atomic operations`_
0xc3    0xa1  any   lock::                                               `Atomic operations`_

                        *(uint32_t *)(dst + offset) ^= src
                        src = *(uint32_t *)(dst + offset)
0xc3    0xe1  any   lock::                                               `Atomic operations`_

                        temp = *(uint32_t *)(dst + offset)
                        *(uint32_t *)(dst + offset) = src
                        src = temp
0xc3    0xf1  any   lock::                                               `Atomic operations`_

                        temp = *(uint32_t *)(dst + offset)
                        if *(uint32_t)(dst + offset) == R0
                           *(uint32_t)(dst + offset) = src
                        R0 = temp
0xc4    any   0x00  dst = (uint32_t)(dst s>> imm)                        `Arithmetic instructions`_
0xc5    any   0x00  if dst s< imm goto +offset                           `Jump instructions`_
0xc6    any   0x00  if (int32_t)dst s< (int32_t)imm goto +offset         `Jump instructions`_
0xc7    any   0x00  dst s>>= imm                                         `Arithmetic instructions`_
0xcc    0x00  any   dst = (uint32_t)(dst s>> src)                        `Arithmetic instructions`_
0xcd    0x00  any   if dst s< src goto +offset                           `Jump instructions`_
0xce    0x00  any   if (int32_t)dst s< (int32_t)src goto +offset         `Jump instructions`_
0xcf    0x00  any   dst s>>= src                                         `Arithmetic instructions`_
0xd4    0x10  0x00  dst = htole16(dst)                                   `Byte swap instructions`_
0xd4    0x20  0x00  dst = htole32(dst)                                   `Byte swap instructions`_
0xd4    0x40  0x00  dst = htole64(dst)                                   `Byte swap instructions`_
0xd5    any   0x00  if dst s<= imm goto +offset                          `Jump instructions`_
0xd6    any   0x00  if (int32_t)dst s<= (int32_t)imm goto +offset        `Jump instructions`_
0xdb    0x00  any   lock \*(uint64_t \*)(dst + offset) += src            `Atomic operations`_
0xdb    0x01  any   lock::                                               `Atomic operations`_

                        *(uint64_t *)(dst + offset) += src
                        src = *(uint64_t *)(dst + offset)
0xdb    0x40  any   \*(uint64_t \*)(dst + offset) \|= src                `Atomic operations`_
0xdb    0x41  any   lock::                                               `Atomic operations`_

                        *(uint64_t *)(dst + offset) |= src
                        lock src = *(uint64_t *)(dst + offset)
0xdb    0x50  any   \*(uint64_t \*)(dst + offset) &= src                 `Atomic operations`_
0xdb    0x51  any   lock::                                               `Atomic operations`_

                        *(uint64_t *)(dst + offset) &= src
                        src = *(uint64_t *)(dst + offset)
0xdb    0xa0  any   \*(uint64_t \*)(dst + offset) ^= src                 `Atomic operations`_
0xdb    0xa1  any   lock::                                               `Atomic operations`_

                        *(uint64_t *)(dst + offset) ^= src
                        src = *(uint64_t *)(dst + offset)
0xdb    0xe1  any   lock::                                               `Atomic operations`_

                        temp = *(uint64_t *)(dst + offset)
                        *(uint64_t *)(dst + offset) = src
                        src = temp
0xdb    0xf1  any   lock::                                               `Atomic operations`_

                        temp = *(uint64_t *)(dst + offset)
                        if *(uint64_t)(dst + offset) == R0
                           *(uint64_t)(dst + offset) = src
                        R0 = temp
0xdc    0x10  0x00  dst = htobe16(dst)                                   `Byte swap instructions`_
0xdc    0x20  0x00  dst = htobe32(dst)                                   `Byte swap instructions`_
0xdc    0x40  0x00  dst = htobe64(dst)                                   `Byte swap instructions`_
0xdd    0x00  any   if dst s<= src goto +offset                          `Jump instructions`_
0xde    0x00  any   if (int32_t)dst s<= (int32_t)src goto +offset        `Jump instructions`_
======  ====  ====  ===================================================  ========================================
