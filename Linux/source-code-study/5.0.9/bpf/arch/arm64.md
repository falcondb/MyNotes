### arm64/net/bpf_jit.h
* 5-bit Register Operand  see arm64/include/asm/insn.h
```#define A64_R(x)	AARCH64_INSN_REG_##x
#define A64_FP		AARCH64_INSN_REG_FP
#define A64_LR		AARCH64_INSN_REG_LR
#define A64_ZR		AARCH64_INSN_REG_ZR
#define A64_SP		AARCH64_INSN_REG_SP

enum aarch64_insn_register {
	AARCH64_INSN_REG_0  = 0,
	AARCH64_INSN_REG_1  = 1,
  ...
} ```

* all the definitions of AARCH64_XXX staff
```
enum aarch64_insn_imm_type
enum aarch64_insn_register_type
enum aarch64_insn_encoding_class
enum aarch64_insn_condition
enum aarch64_insn_branch_type
enum aarch64_insn_size_type
enum aarch64_insn_ldst_type
enum aarch64_insn_data2_type
enum aarch64_insn_logic_type

```

```
#define	__AARCH64_INSN_FUNCS(abbr, mask, val)				\
static __always_inline bool aarch64_insn_is_##abbr(u32 code)		\
{									\
	BUILD_BUG_ON(~(mask) & (val));					\
	return (code & (mask)) == (val);				\
}									\
static __always_inline u32 aarch64_insn_get_##abbr##_value(void)	\
{									\
	return (val);							\
}

__AARCH64_INSN_FUNCS(adr,	0x9F000000, 0x10000000) \\ insn.h
...

```

* kprobe manager
```
patch_text() ==> aarch64_insn_patch_text() ==> aarch64_insn_patch_text_cb() ==>
aarch64_insn_patch_text_nosync() ==> aarch64_insn_write() ==> patch_map(); probe_kernel_write() ==>
probe_write_common()  ==> __copy_to_user_inatomic()  ==> raw_copy_to_user() ==>
/* arm64 */ __arch_copy_to_user()  ==> copy_touser.S

__kprobes *patch_map(void *addr, int fixmap) {
	unsigned long uintaddr = (uintptr_t) addr;
	page = phys_to_page(__pa_symbol(addr));
  return (void *)set_fixmap_offset(fixmap, page_to_phys(page) +
    (uintaddr & ~PAGE_MASK));
}
```
