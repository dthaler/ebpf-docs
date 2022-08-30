.. contents::
.. sectnum::

====================
eBPF Instruction Set
====================

Registers and calling convention
================================

eBPF has 10 general purpose registers and a read-only frame pointer register,
all of which are 64-bits wide.

The eBPF calling convention is defined as:

 * R0: return value from function calls, and exit value for eBPF programs
 * R1 - R5: arguments for function calls
 * R6 - R9: callee saved registers that function calls will preserve
 * R10: read-only frame pointer to access stack

R0 - R5 are scratch registers and eBPF programs need to spill/fill them if
necessary across calls.

Instruction encoding
====================

An eBPF program is a sequence of 64-bit instructions.

eBPF has two instruction encodings:

 * the basic instruction encoding, which uses 64 bits to encode an instruction
 * the wide instruction encoding, which appends a second 64-bit immediate value
   (imm64) after the basic instruction for a total of 128 bits.

The basic instruction encoding is as follows:

 =============  =======  ===============  ====================  ============
 32 bits (MSB)  16 bits  4 bits           4 bits                8 bits (LSB)
 =============  =======  ===============  ====================  ============
 imm            offset   src              dst                   opcode
 =============  =======  ===============  ====================  ============

imm         
  integer immediate value

offset
  integer offset

src
  source register number

dst
  destination register number

opcode
  operation to perform

Note that most instructions do not use all of the fields.
Unused fields must be set to zero.

As discussed below in `64-bit immediate instructions`_, some basic
instructions denote that a 64-bit immediate value follows.  Thus
the wide instruction encoding is as follows:

 =================  =============
 64 bits (MSB)      64 bits (LSB)
 =================  =============
 basic instruction  imm64
 =================  =============

where MSB and LSB mean the most significant bits and least significant bits, respectively.

In the remainder of this document 'src' and 'dst' refer to the values of the source
and destination registers, respectively, rather than the register number.

Instruction classes
-------------------

The encoding of the 'opcode' field varies and can be determined from
the three least significant bits (LSB) of the 'opcode' field which holds
the "instruction class", as follows:

  =========  =====  ===============================  =================
  class      value  description                      reference
  =========  =====  ===============================  =================
  BPF_LD     0x00   non-standard load operations     `Load and store instructions`_
  BPF_LDX    0x01   load into register operations    `Load and store instructions`_
  BPF_ST     0x02   store from immediate operations  `Load and store instructions`_
  BPF_STX    0x03   store from register operations   `Load and store instructions`_
  BPF_ALU    0x04   32-bit arithmetic operations     `Arithmetic and jump instructions`_
  BPF_JMP    0x05   64-bit jump operations           `Arithmetic and jump instructions`_
  BPF_JMP32  0x06   32-bit jump operations           `Arithmetic and jump instructions`_
  BPF_ALU64  0x07   64-bit arithmetic operations     `Arithmetic and jump instructions`_
  =========  =====  ===============================  =================

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
  the source operand, which unless otherwise specified is one of:

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
The 4-bit 'code' field encodes the operation as follows:

  ========  =====  =================================================
  code      value  description
  ========  =====  =================================================
  BPF_ADD   0x00   dst += src
  BPF_SUB   0x10   dst -= src
  BPF_MUL   0x20   dst \*= src
  BPF_DIV   0x30   dst /= src
  BPF_OR    0x40   dst \|= src
  BPF_AND   0x50   dst &= src
  BPF_LSH   0x60   dst <<= src
  BPF_RSH   0x70   dst >>= src
  BPF_NEG   0x80   dst = ~src
  BPF_MOD   0x90   dst %= src
  BPF_XOR   0xa0   dst ^= src
  BPF_MOV   0xb0   dst = src
  BPF_ARSH  0xc0   sign extending shift right
  BPF_END   0xd0   byte swap operations (see `Byte swap instructions`_ below)
  ========  =====  =================================================

Examples:

``BPF_ADD | BPF_X | BPF_ALU`` (0x0c) means::

  dst = (uint32_t) (dst + src);

where '(uint32_t)' indicates truncation to 32 bits.

*Linux implementation note*: In the Linux kernel, uint32_t is expressed as u32,
uint64_t is expressed as u64, etc.  This document uses the standard C terminology
as the cross-platform specification.

``BPF_ADD | BPF_X | BPF_ALU64`` (0x0f) means::

  dst = dst + src

``BPF_XOR | BPF_K | BPF_ALU`` (0xa4) means::

  src = (uint32_t) src ^ (uint32_t) imm

``BPF_XOR | BPF_K | BPF_ALU64`` (0xa7) means::

  src = src ^ imm


Byte swap instructions
----------------------

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

*Linux implementation note*:
``BPF_FROM_LE`` and ``BPF_FROM_BE`` exist as aliases for ``BPF_TO_LE`` and
``BPF_TO_BE`` respectively.

The 'imm' field encodes the width of the swap operations.  The following widths
are supported: 16, 32 and 64. The following table summarizes the resulting
possibilities:

  =============================  =========  ===  ========  =================
  opcode construction            opcode     imm  mnemonic  pseudocode
  =============================  =========  ===  ========  =================
  BPF_ALU | BPF_TO_LE | BPF_END  0xd4       16   le16 dst  dst = htole16(dst)
  BPF_ALU | BPF_TO_LE | BPF_END  0xd4       32   le32 dst  dst = htole32(dst)
  BPF_ALU | BPF_TO_LE | BPF_END  0xd4       64   le64 dst  dst = htole64(dst)
  BPF_ALU | BPF_TO_BE | BPF_END  0xdc       16   be16 dst  dst = htobe16(dst)
  BPF_ALU | BPF_TO_BE | BPF_END  0xdc       32   be32 dst  dst = htobe32(dst)
  BPF_ALU | BPF_TO_BE | BPF_END  0xdc       64   be64 dst  dst = htobe64(dst)
  =============================  =========  ===  ========  =================

where
  * mnenomic indicates a short form that might be displayed by some tools such as disassemblers
  * 'htoleNN()' indicates converting a NN-bit value from host byte order to little-endian byte order
  * 'htobeNN()' indicates converting a NN-bit value from host byte order to big-endian byte order

Jump instructions
-----------------

Instruction class ``BPF_JMP32`` uses 32-bit wide operands while ``BPF_JMP`` uses 64-bit wide operands for
otherwise identical operations.
The 4-bit 'code' field encodes the operation as below:

  ========  =====  ============================  ============
  code      value  description                   notes
  ========  =====  ============================  ============
  BPF_JA    0x00   PC += offset                  BPF_JMP only
  BPF_JEQ   0x10   PC += offset if dst == src
  BPF_JGT   0x20   PC += offset if dst > src     unsigned
  BPF_JGE   0x30   PC += offset if dst >= src    unsigned
  BPF_JSET  0x40   PC += offset if dst & src
  BPF_JNE   0x50   PC += offset if dst != src
  BPF_JSGT  0x60   PC += offset if dst > src     signed
  BPF_JSGE  0x70   PC += offset if dst >= src    signed
  BPF_CALL  0x80   function call
  BPF_EXIT  0x90   function / program return     BPF_JMP only
  BPF_JLT   0xa0   PC += offset if dst < src     unsigned
  BPF_JLE   0xb0   PC += offset if dst <= src    unsigned
  BPF_JSLT  0xc0   PC += offset if dst < src     signed
  BPF_JSLE  0xd0   PC += offset if dst <= src    signed
  ========  =====  ============================  ============

The eBPF verifier is responsible for verifying that the
eBPF program stores the return value into register R0 before doing a
``BPF_EXIT``.


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
  BPF_ATOMIC     0xc0   atomic operations                     `Atomic operations`
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

``BPF_MEM | <size> | BPF_STX`` means::

  *(size *) (dst + offset) = src

``BPF_MEM | <size> | BPF_ST`` means::

  *(size *) (dst + offset) = imm

``BPF_MEM | <size> | BPF_LDX`` means::

  dst = *(size *) (src + offset)

Where size is one of: ``BPF_B``, ``BPF_H``, ``BPF_W``, or ``BPF_DW``.

Atomic operations
-----------------

Atomic operations are operations that operate on memory and can not be
interrupted or corrupted by other access to the same memory region
by other eBPF programs or means outside of this specification.

All atomic operations supported by eBPF are encoded as store operations
that use the ``BPF_ATOMIC`` mode modifier as follows:

  * ``BPF_ATOMIC | BPF_W | BPF_STX`` for 32-bit operations
  * ``BPF_ATOMIC | BPF_DW | BPF_STX`` for 64-bit operations

Note that 8-bit (``BPF_B``) and 16-bit (``BPF_H``) wide atomic operations are not supported,
nor is ``BPF_ATOMIC | <size> | BPF_ST``.

The 'imm' field is used to encode the actual atomic operation.
Simple atomic operation use a subset of the values defined to encode
arithmetic operations in the 'imm' field to encode the atomic operation:

  ========  =====  ===========  =======
  imm       value  description  version
  ========  =====  ===========  =======
  BPF_ADD   0x00   atomic add   v1
  BPF_OR    0x40   atomic or    v3
  BPF_AND   0x50   atomic and   v3
  BPF_XOR   0xa0   atomic xor   v3
  ========  =====  ===========  =======

**TODO**: Confirm the versions above. And add a section introducing the version concept.

``BPF_ATOMIC | BPF_W  | BPF_STX`` with 'imm' = BPF_ADD means::

  *(uint32_t *)(dst + offset) += src

``BPF_ATOMIC | BPF_DW | BPF_STX`` with 'imm' = BPF ADD means::

  *(uint64_t *)(dst + offset) += src

*Linux implementation note*: ``BPF_XADD`` is a deprecated name for ``BPF_ATOMIC | BPF_ADD``.

In addition to the simple atomic operations above, there also is a modifier and
two complex atomic operations:

  ===========  ================  ===========================  =======
  imm          value             description                  version
  ===========  ================  ===========================  =======
  BPF_FETCH    0x01              modifier: return old value   v3
  BPF_XCHG     0xe0 | BPF_FETCH  atomic exchange              v3
  BPF_CMPXCHG  0xf0 | BPF_FETCH  atomic compare and exchange  v3
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

*Clang implementation note*:
Clang can generate atomic instructions by default when ``-mcpu=v3`` is
enabled. If a lower version for ``-mcpu`` is set, the only atomic instruction
Clang can generate is ``BPF_ADD`` *without* ``BPF_FETCH``. If you need to enable
the atomics features, while keeping a lower ``-mcpu`` version, you can use
``-Xclang -target-feature -Xclang +alu32``.

64-bit immediate instructions
-----------------------------

Instructions with the ``BPF_IMM`` 'mode' modifier use the wide instruction
encoding for an extra imm64 value.

There is currently only one such instruction.

``BPF_LD | BPF_DW | BPF_IMM`` means::

  dst = imm64


Legacy BPF Packet access instructions
-------------------------------------

eBPF has special instructions for access to packet data that have been
carried over from classic BPF to retain the performance of legacy socket
filters running in an eBPF interpreter.

The instructions come in two forms: ``BPF_ABS | <size> | BPF_LD`` and
``BPF_IND | <size> | BPF_LD``.

These instructions are used to access packet data and can only be used when
the program context contains a pointer to a networking packet.  ``BPF_ABS``
accesses packet data at an absolute offset specified by the immediate data
and ``BPF_IND`` access packet data at an offset that includes the value of
a register in addition to the immediate data.

These instructions have seven implicit operands:

 * Register R6 is an implicit input that must contain a pointer to a
   context structure with a packet data pointer.
 * Register R0 is an implicit output which contains the data fetched from
   the packet.
 * Registers R1-R5 are scratch registers that are clobbered by the
   instruction.

*Linux implementation note*: In Linux, R6 references a struct sk_buff.

These instructions have an implicit program exit condition as well. If an
eBPF program attempts access data beyond the packet boundary, the
program execution must be gracefully aborted.

**TODO**: Is the verifier required to allow such programs, or is it free to
reject them?

``BPF_ABS | BPF_W | BPF_LD`` means::

  R0 = ntohl(*(uint32_t *) (R6->data + imm))

where `ntohl()` converts a 32-bit value from network byte order to host byte order.

``BPF_IND | BPF_W | BPF_LD`` means::

  R0 = ntohl(*(uint32_t *) (R6->data + src + imm))

Appendix
========

For reference, the following table lists opcodes in order by value.

======  =================================================  =============
opcode  description                                        reference 
======  =================================================  =============
0x04    dst = (uint32_t)(dst + imm)                        `Arithmetic instructions`_
0x05    goto +offset                                       `Jump instructions`_
0x07    dst += imm                                         `Arithmetic instructions`_
0x0c    dst = (uint32_t)(dst + src)                        `Arithmetic instructions`_
0x0f    dst += src                                         `Arithmetic instructions`_
0x14    dst = (uint32_t)(dst - imm)                        `Arithmetic instructions`_
0x15    if dst == imm goto +offset                         `Jump instructions`_
0x17    dst -= imm                                         `Arithmetic instructions`_
0x18    dst = imm                                          `Load and store instructions`_
0x1c    dst = (uint32_t)(dst - src)                        `Arithmetic instructions`_
0x1d    if dst == src goto +offset                         `Jump instructions`_
0x1f    dst -= src                                         `Arithmetic instructions`_
0x20    dst = ntohl(*(uint32_t *)(R6->data + imm))         `Load and store instructions`_
0x24    dst = (uint32_t)(dst * imm)                        `Arithmetic instructions`_
0x25    if dst > imm goto +offset                          `Jump instructions`_
0x27    dst *= imm                                         `Arithmetic instructions`_
0x28    dst = ntohs(*(uint16_t *)(R6->data + imm))         `Load and store instructions`_
0x2c    dst = (uint32_t)(dst * src)                        `Arithmetic instructions`_
0x2d    if dst > src goto +offset                          `Jump instructions`_
0x2f    dst *= src                                         `Arithmetic instructions`_
0x30    dst = (*(uint8_t *)(R6->data + imm))               `Load and store instructions`_
0x34    dst = (uint32_t)(dst / imm)                        `Arithmetic instructions`_
0x35    if dst >= imm goto +offset                         `Jump instructions`_
0x37    dst /= imm                                         `Arithmetic instructions`_
0x38    dst = ntohll(*(uint64_t *)(R6->data + imm))        `Load and store instructions`_
0x3c    dst = (uint32_t)(dst / src)                        `Arithmetic instructions`_
0x3d    if dst >= src goto +offset                         `Jump instructions`_
0x3f    dst /= src                                         `Arithmetic instructions`_
0x40    dst = ntohl(*(uint32_t *)(R6->data + src + imm))   `Load and store instructions`_
0x44    dst = (uint32_t)(dst \| imm)                       `Arithmetic instructions`_
0x45    if dst & imm goto +offset                          `Jump instructions`_
0x47    dst |= imm                                         `Arithmetic instructions`_
0x48    dst = ntohs(*(uint16_t *)(R6->data + src + imm))   `Load and store instructions`_
0x4c    dst = (uint32_t)(dst \| src)                       `Arithmetic instructions`_
0x4d    if dst & src goto +offset                          `Jump instructions`_
0x4f    dst |= src                                         `Arithmetic instructions`_
0x50    dst = *(uint8_t *)(R6->data + src + imm))          `Load and store instructions`_
0x54    dst = (uint32_t)(dst & imm)                        `Arithmetic instructions`_
0x55    if dst != imm goto +offset                         `Jump instructions`_
0x57    dst &= imm                                         `Arithmetic instructions`_
0x58    dst = ntohll(*(uint64_t *)(R6->data + src + imm))  `Load and store instructions`_
0x5c    dst = (uint32_t)(dst & src)                        `Arithmetic instructions`_
0x5d    if dst != src goto +offset                         `Jump instructions`_
0x5f    dst &= src                                         `Arithmetic instructions`_
0x61    dst = *(uint32_t *)(src + offset)                  `Load and store instructions`_
0x62    *(uint32_t *)(dst + offset) = imm                  `Load and store instructions`_
0x63    *(uint32_t *)(dst + offset) = src                  `Load and store instructions`_
0x64    dst = (uint32_t)(dst << imm)                       `Arithmetic instructions`_
0x65    if dst s> imm goto +offset                         `Jump instructions`_
0x67    dst <<= imm                                        `Arithmetic instructions`_
0x69    dst = *(uint16_t *)(src + offset)                  `Load and store instructions`_
0x6a    *(uint16_t *)(dst + offset) = imm                  `Load and store instructions`_
0x6b    *(uint16_t *)(dst + offset) = src                  `Load and store instructions`_
0x6c    dst = (uint32_t)(dst << src)                       `Arithmetic instructions`_
0x6d    if dst s> src goto +offset                         `Jump instructions`_
0x6f    dst <<= src                                        `Arithmetic instructions`_
0x71    dst = *(uint8_t *)(src + offset)                   `Load and store instructions`_
0x72    *(uint8_t *)(dst + offset) = imm                   `Load and store instructions`_
0x73    *(uint8_t *)(dst + offset) = src                   `Load and store instructions`_
0x74    dst = (uint32_t)(dst >> imm)                       `Arithmetic instructions`_
0x75    if dst s>= imm goto +offset                        `Jump instructions`_
0x77    dst >>= imm                                        `Arithmetic instructions`_
0x79    dst = *(uint64_t *)(src + offset)                  `Load and store instructions`_
0x7a    *(uint64_t *)(dst + offset) = imm                  `Load and store instructions`_
0x7b    *(uint64_t *)(dst + offset) = src                  `Load and store instructions`_
0x7c    dst = (uint32_t)(dst >> src)                       `Arithmetic instructions`_
0x7d    if dst s>= src goto +offset                        `Jump instructions`_
0x7f    dst >>= src                                        `Arithmetic instructions`_
0x84    dst = (uint32_t)-dst                               `Arithmetic instructions`_
0x85    call imm                                           `Jump instructions`_
0x87    dst = -dst                                         `Arithmetic instructions`_
0x94    dst = (uint32_t)(dst % imm)                        `Arithmetic instructions`_
0x95    exit                                               `Jump instructions`_
0x97    dst %= imm                                         `Arithmetic instructions`_
0x9c    dst = (uint32_t)(dst % src)                        `Arithmetic instructions`_
0x9f    dst %= src                                         `Arithmetic instructions`_
0xa4    dst = (uint32_t)(dst ^ imm)                        `Arithmetic instructions`_
0xa5    if dst < imm goto +offset                          `Jump instructions`_
0xa7    dst ^= imm                                         `Arithmetic instructions`_
0xac    dst = (uint32_t)(dst ^ src)                        `Arithmetic instructions`_
0xad    if dst < src goto +offset                          `Jump instructions`_
0xaf    dst ^= src                                         `Arithmetic instructions`_
0xb4    dst = (uint32_t) imm                               `Arithmetic instructions`_
0xb5    if dst <= imm goto +offset                         `Jump instructions`_
0xb7    dst = imm                                          `Arithmetic instructions`_
0xbc    dst = (uint32_t) src                               `Arithmetic instructions`_
0xbd    if dst <= src goto +offset                         `Jump instructions`_
0xbf    dst = src                                          `Arithmetic instructions`_
0xc4    dst = (uint32_t)(dst s>> imm)                      `Arithmetic instructions`_
0xc5    if dst s< imm goto +offset                         `Jump instructions`_
0xc7    dst s>>= imm                                       `Arithmetic instructions`_
0xcc    dst = (uint32_t)(dst s>> src)                      `Arithmetic instructions`_
0xcd    if dst s< src goto +offset                         `Jump instructions`_
0xcf    dst s>>= src                                       `Arithmetic instructions`_
0xd4    dst = htole.imm(dst)                               `Byte swap instructions`_
0xd5    if dst s<= imm goto +offset                        `Jump instructions`_
0xdc    dst = htobe.imm(dst)                               `Byte swap instructions`_
0xdd    if dst s<= src goto +offset                        `Jump instructions`_
======  =================================================  =============
