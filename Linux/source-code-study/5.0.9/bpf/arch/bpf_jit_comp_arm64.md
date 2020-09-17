* build_insn
```
static int build_insn(const struct bpf_insn *insn, struct jit_ctx *ctx, bool extra_pass)
	const u8 code = insn->code;
  switch (code) {
    case BPF_ALU64 | BPF_MOV | BPF_X:
      emit(A64_MOV(is64, dst, src), ctx);
    ...
  }


  static inline void emit(const u32 insn, struct jit_ctx *ctx)
  {
  	if (ctx->image != NULL)
  		ctx->image[ctx->idx] = cpu_to_le32(insn);

  	ctx->idx++;
  }

  // in bpf_jit.h
  // #define A64_MOV(sf, Rd, Rn) A64_ADD_I(sf, Rd, Rn, 0)
  // #define A64_ADD_I(sf, Rd, Rn, imm12) A64_ADDSUB_IMM(sf, Rd, Rn, imm12, ADD)
  // #define A64_ADDSUB_IMM(sf, Rd, Rn, imm12, type)
  // aarch64_insn_gen_add_sub_imm(Rd, Rn, imm12,
  // A64_VARIANT(sf), AARCH64_INSN_ADSB_##type)

  // aarch64_insn_gen_XXX_XXX retruns the instruction code of arm64


  // Map BPF registers to A64 registers
  static const int bpf2a64[] = {
  	/* return value from in-kernel function, and exit value from eBPF */
  	[BPF_REG_0] = A64_R(7),
    ...
  }
```
