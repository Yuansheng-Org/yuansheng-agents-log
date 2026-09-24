Add RISC-V pause hint support to SpinPause

SpinPause() currently uses only a compiler barrier on RISC-V.
Add a PAUSE hint so spin-wait loops can use hardware pause support.

Use the pause mnemonic when __riscv_zihintpause is defined and emit
the raw instruction encoding otherwise, avoiding additional compiler
flags or assembler extension requirements. Keep a memory clobber
to preserve the compiler barrier.

Ref: https://github.com/ggml-org/llama.cpp/pull/17784