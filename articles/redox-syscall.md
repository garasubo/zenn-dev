---
title: "Redoxにおけるシステムコールの実装を読む 〜　x86_64編"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Rust, osdev, Redox]
published: true
---

この資料はRedoxを読む会 #2のための資料を兼ねています。

https://osdev-jp.connpass.com/event/246485/

RedoxというRustで書かれたOSのシステムコールの実装を読み解きます。Redoxは現在x86_64とaarch64アーキテクチャに対応していますが、
今回はx86_64アーキテクチャの実装を中心に読んでいきます。

Redoxのレポジトリはこちら。読んだときのコミットハッシュは`0a63f024e9c824384344ede006d40805b53909db`、tagの`0.7.0`の直後のコミットになっています。

https://gitlab.redox-os.org/redox-os/redox

主に見るのは`kernel`ディレクトリ以下でこれはGitのサブモジュールとして管理されています。

https://gitlab.redox-os.org/redox-os/kernel

以降、ソースコードの引用時、コメント部分は適宜取り除いたり、筆者によるコメントに置き換えている場合があります。

## syscall呼び出し側
ユーザープログラムからの呼び出しの実装は`syscall`ディレクトリ以下にあります。これは`kernel`以下のGitのサブモジュールとして管理されていて別のクレートになっています。

https://gitlab.redox-os.org/redox-os/syscall

`kernel/syscall/src/call.rs`を見るとどんなシステムコールがあるかを見ることができます。
基本的には引数の数に合わせて`syscall1`や`syscall3`と言った関数があり、それを呼び出すためのラッパになっています。
例として`open`関数を見てみましょう。

```rust
pub fn open<T: AsRef<str>>(path: T, flags: usize) -> Result<usize> {
    unsafe { syscall3(SYS_OPEN, path.as_ref().as_ptr() as usize, path.as_ref().len(), flags) }
}
```

`syscall3`は第一引数にシステムコールの種類を示すIDを受け取り、残りはusizeとして3つのそのシステムコールのための引数を受け取る関数になっています。
`syscall3`そのものはアーキテクチャ依存の実装になっています。x86_64のものは`kernel/syscall/src/arch/x86_64.rs`に実装されていてこのようになっています。

```rust
macro_rules! syscall {
    ($($name:ident($a:ident, $($b:ident, $($c:ident, $($d:ident, $($e:ident, $($f:ident, )?)?)?)?)?);)+) => {
        $(
            pub unsafe fn $name(mut $a: usize, $($b: usize, $($c: usize, $($d: usize, $($e: usize, $($f: usize)?)?)?)?)?) -> Result<usize> {
                asm!(
                    "syscall",
                    inout("rax") $a,
                    $(
                        in("rdi") $b,
                        $(
                            in("rsi") $c,
                            $(
                                in("rdx") $d,
                                $(
                                    in("r10") $e,
                                    $(
                                        in("r8") $f,
                                    )?
                                )?
                            )?
                        )?
                    )?
                    out("rcx") _,
                    out("r11") _,
                    options(nostack),
                );

                Error::demux($a)
            }
        )+
    };
}

syscall! {
    syscall0(a,);
    syscall1(a, b,);
    syscall2(a, b, c,);
    syscall3(a, b, c, d,);
    syscall4(a, b, c, d, e,);
    syscall5(a, b, c, d, e, f,);
}

```

マクロになっていて読みにくいですね。例として`syscall3`がこのマクロによってどのような定義になるか展開します。


```rust
pub unsafe fn sycall3(mut a: usize, b: usize, c: usize, d: usize) -> Result<usize> {
    asm!(
        "syscall",
        inout("rax") a,
        in("rdi") b,
        in("rsi") c,
        in("rdx") d,
        out("rcx") _,
        out("r11") _,
        options(nostack),
    );

    Error::demux(a)
}
```

インラインアセンブラによって`syscall`という命令を呼んでいます。これはユーザーランドからカーネルの機能を呼び出すためのFast System Callsという仕組みのための命令です。
これを呼び出すことによってユーザーランドから特権レベルの特定のルーチンに飛ぶことができます。

レジスタが`inout`などで指定されています。RAXは関数の戻り値を格納するためのレジスタと呼び出し規約ではなっていますが、このシステムコールでは同時にシステムコールのIDを渡す役割も担っているようです。
RDI、RSI、RDXは関数の引数を渡すためのレジスタなのでそのままの使い方ですね。`out`としてRCXとR11が指定されています。
実は`sycall`命令はこれらのレジスタを上書きするのでこれらの値が破壊されることをコンパイラに伝えるための書き方です。
`options(nostack)`はこのインラインアセンブラ内でスタックに値を追加しないことを教えるためのオプションです。

呼び出し側の実装はこのようになっていて、次にカーネル側でどのように処理されるかを見ていきましょう。

## syscallのための初期設定
syscall時にどのような動きをするかはIntel SDM Vol.3 Chapter 5.8.8を見るとわかります。
システムコールは一見すると例外や割り込みの一種に見えるかもしれませんが、x86_64アーキテクチャでは別物であることに注意してください。

`kernel/src/arch/x86_64/interrupt/syscall.rs`より

```rust
pub unsafe fn init() {
    // IA32_STAR[31:0] are reserved.

    // syscallでprivileged level 0になるときのコードセグメントセレクタ
    let syscall_cs_ss_base = (gdt::GDT_KERNEL_CODE as u16) << 3;
    // sysretでprivileged level 3になるときのコードセグメントセレクタ
    let sysret_cs_ss_base = ((gdt::GDT_USER_CODE32_UNUSED as u16) << 3) | 3;
    let star_high = u32::from(syscall_cs_ss_base) | (u32::from(sysret_cs_ss_base) << 16);

    msr::wrmsr(msr::IA32_STAR, u64::from(star_high) << 32);
    msr::wrmsr(msr::IA32_LSTAR, syscall_instruction as u64);
    msr::wrmsr(msr::IA32_FMASK, 0x0300); // Clear trap flag and interrupt enable

    let efer = msr::rdmsr(msr::IA32_EFER);
    msr::wrmsr(msr::IA32_EFER, efer | 1);
}
```

`wrmsr`命令でMSR（Model Specific Register）というレジスタに値を書き込みます。

`syscall_cs_ss_base`はsyscallが呼ばれたときに使われるコードセグメントとスタックセグメントを指定するためのセグメントセレクタです。
コードセグメントはIA32_STAR[47:32]の設定値で、でスタックセグメントはIA32_STAR[47:32]の設定値+8の値となります。そのため、GDTではこの２つのセグメントは連続しています。
逆に`sysret_cs_ss_base`はsysretが呼ばれたときに使われるコードセグメントとスタックセグメントを指定するためのセグメントセレクタです。
実際のコードセグメントはIA32_STAR[63:48]の設定値の+16でスタックセグメントはIA32_STAR[63:48]の設定値+8の値となり、その１つ手前のインデックスを使っています。

セグメントセレクタの構造として[15:3]がインデックスとして使われ、[1:0]が特権レベルを指します。
そのため`syscall_cs_ss_base`ではレベル0に突入するため0になっていますが、
`sysret_cs_ss_base`はレベル3に戻るため3を付け加えています。

ちなみにGDTの定義及びその初期化は`kernel/src/arch/x86_64/gdt.rs`で行われています。
GDTの詳しい解説はSDMの他に英語ですがこちらの資料も参考になります。

https://wiki.osdev.org/Global_Descriptor_Table

FMASKはRFLAGSの現在値をマスクするための値です。第８ビットと第９ビットはそれぞれトラップと割り込みの有効化のビットなのでシステムコール中はこれらを無効にするということになります。
各フィールドについてはIntel SDM Vol.3 Chapter 2.3とFigure 2-5を参照しましょう。

### syscallハンドラ
syscallが発行されたときに呼び出されるコード関数が`syscall_instruction`になります。
長いのでいくつかに分けて順番に見ていきましょう。
まず最初の"magic"と言われている箇所を見ていきます。"you don't need to understand"と書いてありますが、理解します。

```rust
#[naked]
pub unsafe extern "C" fn syscall_instruction() {
    core::arch::asm!(concat!(
    // Yes, this is magic. No, you don't need to understand
    "
        swapgs                    // Set gs segment to TSS
        mov gs:[{sp}], rsp        // Save userspace stack pointer
        mov rsp, gs:[{ksp}]       // Load kernel stack pointer
        push QWORD PTR {ss_sel}   // Push fake userspace SS (resembling iret frame)
        push QWORD PTR gs:[{sp}]  // Push userspace rsp
        push r11                  // Push rflags
        push QWORD PTR {cs_sel}   // Push fake CS (resembling iret stack frame)
        push rcx                  // Push userspace return pointer
    ",
```

`#[naked]`はRust特有の呼び出し規約によるレジスタ退避のロジックを余計に埋め込まれるのを防ぐためのアトリビュートです。
このインラインアセンブラは`noreturn`がつけられていて、このインラインアセンブラ内で呼び出し元に復帰します。そのため、Rustが勝手にレジスタ退避のロジックを埋め込んでしまうと、それらを取り出す処理が入らなくなりスタックに余計なものが積まれた状態で呼び出し元に復帰してしまいます。

`swapgs`はGSというセグメンテーションレジスタとIA32_GS_BASEというMSRのフィールドをスワップする命令です。セグメンテーションレジスタは他にもいくつかありますが、このような命令が備わっているのはGSのみです。
SDMの解説でもこれはsyscallのような場面で使うことを想定されているようです。
カーネルがsyscallを処理する時、カーネルのスタックポインタを取得するすべがないのでこのような命令が用意されています。
ちなみにGSの初期値やIA32_GS_BASEの設定は`kernel/src/arch/x86_64/gdt.rs`の`init_paging`で行われています。

`{sp}`といった書き方はRustにおけるformatマクロなどと同じルールで、後ろの方で渡される値が代入されます。
値が渡されているところだけ先に見ましょう。

```rust
    sp = const(offset_of!(gdt::ProcessorControlRegion, user_rsp_tmp)),
    ksp = const(offset_of!(gdt::ProcessorControlRegion, tss) + offset_of!(TaskStateSegment, rsp)),
    ss_sel = const(SegmentSelector::new(gdt::GDT_USER_DATA as u16, x86::Ring::Ring3).bits()),
    cs_sel = const(SegmentSelector::new(gdt::GDT_USER_CODE as u16, x86::Ring::Ring3).bits()),
```

`offset_of`マクロは第一引数に何らかの構造体を受け取り、第二引数はその構造体のメンバーを受け取り、その構造体におけるそのメンバーまでのオフセットを計算してくれます。
`ProcessorControlRegion`の定義はこのようになっています。

```rust
#[repr(C, align(16))]
pub struct ProcessorControlRegion {
    pub tcb_end: usize,
    pub user_rsp_tmp: usize,
    pub tss: TssWrapper,
}
```

`TssWrapper`は`TaskStateSegment`のラッパーになっています。

```rust
#[repr(C, align(16))]
pub struct TssWrapper(pub TaskStateSegment);
```

```rust
#[derive(Clone, Copy, Debug, Default)]
#[repr(C, packed)]
pub struct TaskStateSegment {
    pub reserved: u32,
    /// The full 64-bit canonical forms of the stack pointers (RSP) for privilege levels 0-2.
    pub rsp: [u64; 3],
    pub reserved2: u64,
    /// The full 64-bit canonical forms of the interrupt stack table (IST) pointers.
    pub ist: [u64; 7],
    pub reserved3: u64,
    pub reserved4: u16,
    /// The 16-bit offset to the I/O permission bit map from the 64-bit TSS base.
    pub iomap_base: u16,
}
```

これはTask State SegmentをRustで扱うための構造体です。Task State Segmentについての解説はこちらを参照してください。

https://wiki.osdev.org/Task_State_Segment

`#[repr(C, align(16))]`というのは構造体のABIをC言語と同じにして、更にそのアドレスを16バイトでアラインメントさせるためのアトリビュートです。
`#[repr(packed)]`は構想体のフィールドに余計なパディングなどを挟ませないようにするものです。
`repr`についての詳しい説明は以下のドキュメントが詳しいです。

https://doc.rust-lang.org/nomicon/other-reprs.html

つまり、GSは`ProcessorControlRegion`へのポインタになっているので、
`mov gs:[{sp}], rsp`はGSにおける`user_rsp_tmp`にRSPの値を入れる、つまりユーザースタックポインタの値を保存する操作にあたります。
`mov rsp, gs:[{ksp}]`はGSにおける`tss.rsp[0]`の値をRSPに代入する、つまりカーネルのスタックポインタを持ってくる操作です。

その後いろいろなものをスタックに積んでいます。これはsyscallの呼び出し元への復帰が`sysretq`ではなく`iretq`で行われる場合を考慮しているためです。
これはintel固有の`sysretq`の脆弱性に対処するためのものです。

https://xenproject.org/2012/06/13/the-intel-sysret-privilege-escalation/

簡単に説明すると、`sysretq`の実行時、`RCX`レジスタに不正なアドレスが入っている場合、General Protection例外が発生するのですが、intel製のチップの場合、この不正アドレスのチェックのタイミングの違いにより本来ユーザー権限で実行されるべきなのに特権レベルで実行されてしまうという脆弱性があるためです。


次のパートがシステムコールを処理する部分です。

```rust
    // Push context registers
    "push rax\n",
    push_scratch!(),
    push_preserved!(),

    // Call inner funtion
    "mov rdi, rsp\n",
    "call __inner_syscall_instruction\n",

    // Pop context registers
    pop_preserved!(),
    pop_scratch!(),
```

`push_scratch`はスクラッチレジスタ、つまりcaller-saved（呼び出し元が保存するべき）レジスタを保存し`push_preserved`はcallee-saved（呼び出し先が保存するべき）なレジスタを保存しています。`pop_preserved`と`pop_scratch`はその逆です。
callee-savedなレジスタは本来は保存する必要ないはずですが、`__inner_syscall_instruction`内で使う場面
`push rax`とRAXだけ別に保存してありますが、これは単に`push_scratch`で保存するものの中にraxが含まれていないので追加しているので深い意味はないです。

`__inner_syscall_instruction`を呼び出していて、この先を追うとシステムコールがどのように処理されるかがわかります。
直前の`mov`は現在のスタック値を関数の引数として渡すためのものです。RDIが第一引数として使われます。
呼び出し先の処理を詳しく見る前に、先に残りのユーザースペースに戻る部分のコードを見ていきます。

```rust
    "
        // Set ZF iff forbidden bits 63:47 (i.e. the bits that must be sign extended) of the pushed
        // RCX are set.
        test DWORD PTR [rsp + 4], 0xFFFF8000

        // If ZF was set, i.e. the address was invalid higher-half, so jump to the slower iretq and
        // handle the error without being able to execute attacker-controlled code!
        jnz 1f

        // Otherwise, continue with the fast sysretq.

        pop rcx                 // Pop userspace return pointer
        add rsp, 8              // Pop fake userspace CS
        pop r11                 // Pop rflags
        pop QWORD PTR gs:[{sp}] // Pop userspace stack pointer
        mov rsp, gs:[{sp}]      // Restore userspace stack pointer
        swapgs                  // Restore gs from TSS to user data
        sysretq                 // Return into userspace; RCX=>RIP,R11=>RFLAGS

1:

        // Slow iretq
        xor rcx, rcx
        xor r11, r11
        swapgs
        iretq
    "),

    sp = const(offset_of!(gdt::ProcessorControlRegion, user_rsp_tmp)),
    ksp = const(offset_of!(gdt::ProcessorControlRegion, tss) + offset_of!(TaskStateSegment, rsp)),
    ss_sel = const(SegmentSelector::new(gdt::GDT_USER_DATA as u16, x86::Ring::Ring3).bits()),
    cs_sel = const(SegmentSelector::new(gdt::GDT_USER_CODE as u16, x86::Ring::Ring3).bits()),

    options(noreturn),
    );
}
```

これで`syscall_instruction`全部です。
最初の`test`と`jnz`の分岐の部分が冒頭述べた脆弱性対策のRCXに対する値のチェックです。これが不正な値だった場合は`1:`にジャンプして`iretq`によってリターンします。
そうでない場合は、余計にpushしていた値も含めレジスタの値をもとに戻しつつ`swapgs`でGSの値を再び交換して`sysretq`で帰ります。
このときRCXの値は自動的にRIP、すなわちプログラムカウンタに、R11の値がRFLAGSに書き戻されます。

さて、`__inner_syscall_instruction`を見ていきましょう。

```rust
#[no_mangle]
pub unsafe extern "C" fn __inner_syscall_instruction(stack: *mut InterruptStack) {
    let _guard = ptrace::set_process_regs(stack);
    with_interrupt_stack!(|stack| {
        // Set a restore point for clone
        let rbp;
        core::arch::asm!("mov {}, rbp", out(reg) rbp);

        let scratch = &stack.scratch;
        syscall::syscall(scratch.rax, scratch.rdi, scratch.rsi, scratch.rdx, scratch.r10, scratch.r8, rbp, stack)
    });
}
```

引数として渡されていたのは呼び出し直前のカーネルスタックのアドレスでした。`InterruptStack`の構造はこのようになっています。

```rust
#[derive(Default)]
#[repr(packed)]
pub struct InterruptStack {
    pub preserved: PreservedRegisters,
    pub scratch: ScratchRegisters,
    pub iret: IretRegisters,
}

#[derive(Default)]
#[repr(packed)]
pub struct PreservedRegisters {
    pub r15: usize,
    pub r14: usize,
    pub r13: usize,
    pub r12: usize,
    pub rbp: usize,
    pub rbx: usize,
}

#[derive(Default)]
#[repr(packed)]
pub struct ScratchRegisters {
    pub r11: usize,
    pub r10: usize,
    pub r9: usize,
    pub r8: usize,
    pub rsi: usize,
    pub rdi: usize,
    pub rdx: usize,
    pub rcx: usize,
    pub rax: usize,
}

#[derive(Default)]
#[repr(packed)]
pub struct IretRegisters {
    pub rip: usize,
    pub cs: usize,
    pub rflags: usize,
    pub rsp: usize,
    pub ss: usize
}
```

`ptrace`に関するパートは深く突っ込まないことにします。これを解説するのはきっと他の人がやってくれるはずです。
大雑把に説明すると`proc:`スキーム用のクレートでブレイクポイント用の処理をいろいろやっています。

`with_interrupt_stack`というマクロが使われているので、中身を見てみましょう。

```rust
macro_rules! with_interrupt_stack {
    (|$stack:ident| $code:block) => {{
        let allowed = ptrace::breakpoint_callback(PTRACE_STOP_PRE_SYSCALL, None)
            .and_then(|_| ptrace::next_breakpoint().map(|f| !f.contains(PTRACE_FLAG_IGNORE)));

        if allowed.unwrap_or(true) {
            let $stack = &mut *$stack;
            (*$stack).scratch.rax = $code;
        }

        ptrace::breakpoint_callback(PTRACE_STOP_POST_SYSCALL, None);
    }}
}
```

`code`の実行結果を`stack`ポインタを利用してユーザー側のRAXが退避されているフィールドに代入しています。RAXは戻り値を代入するためのレジスタなので、システムコールの値を代入しているというわけです。
他の部分は`ptrace`関連なのでとりあえず今回は深く突っ込まずにいます。

では`$code:block`として渡されている部分を改めて見ていきましょう。
RBPを`rbp`という変数に退避させています。このレジスタはフレームポインタといって関数の呼び出し元のスタックポインタが入っています。
通常であれば関数の呼び出し時・リターン時に自動的に更新されるものです。コメントには`clone`システムコールのために保存していると書いてあります。
その後、`syscall::syscall`を呼び出しています。ここから先はアーキテクチャ依存のないRustの世界です。
引数として渡しているのはスタック上に積まれたユーザー側のスクラッチレジスタの値です。
呼び出し元でRAXにはシステムコールのIDが入っていましたね。RDIからR8は普通の引数で、RBPの値が入った`rbp`とユーザー側の状態にアクセスするためのスタックポインタが引数として渡されています。

## アキーテクチャー非依存の処理を眺める

`syscall::syscall`の中身を見ていくのですが、基本的にはsyscallのIDでの条件分岐で、数が多くややこしいので適宜省略して見ていきます。

```rust
pub fn syscall(a: usize, b: usize, c: usize, d: usize, e: usize, f: usize, bp: usize, stack: &mut InterruptStack) -> usize {
    #[inline(always)]
    fn inner(a: usize, b: usize, c: usize, d: usize, e: usize, f: usize, bp: usize, stack: &mut InterruptStack) -> Result<usize> {
        //SYS_* is declared in kernel/syscall/src/number.rs
        match a & SYS_CLASS {
            SYS_CLASS_FILE => {
                let fd = FileHandle::from(b);
                match a & SYS_ARG {
                    SYS_ARG_SLICE => match a {
                        SYS_FMAP if b == !0 => {
                            MemoryScheme::fmap_anonymous(unsafe { validate_ref(c as *const Map, d)? })
                        },
                        _ => file_op_slice(a, fd, validate_slice(c as *const u8, d)?),
                    }
                    SYS_ARG_MSLICE => match a {
                        SYS_FSTAT => fstat(fd, unsafe { validate_ref_mut(c as *mut Stat, d)? }),
                        _ => file_op_mut_slice(a, fd, validate_slice_mut(c as *mut u8, d)?),
                    },
                    _ => match a {
                        SYS_CLOSE => close(fd),
                        // 省略
                    }
                }
            },
            SYS_CLASS_PATH => match a {
                SYS_OPEN => open(validate_str(b as *const u8, c)?, d).map(FileHandle::into),
                SYS_CHMOD => chmod(validate_str(b as *const u8, c)?, d as u16),
                SYS_RMDIR => rmdir(validate_str(b as *const u8, c)?),
                SYS_UNLINK => unlink(validate_str(b as *const u8, c)?),
                _ => Err(Error::new(ENOSYS))
            },
            _ => match a {
                SYS_YIELD => sched_yield(),
                // 省略
                SYS_CLONE => {
                    let b = CloneFlags::from_bits_truncate(b);

                    #[cfg(not(target_arch = "x86_64"))]
                    {
                        //TODO: CLONE_STACK
                        let ret = clone(b, bp).map(ContextId::into);
                        ret
                    }

                    #[cfg(target_arch = "x86_64")]
                    {
                        let old_rsp = stack.iret.rsp;
                        if b.contains(flag::CLONE_STACK) {
                            stack.iret.rsp = c;
                        }
                        let ret = clone(b, bp).map(ContextId::into);
                        stack.iret.rsp = old_rsp;
                        ret
                    }
                },
                // 省略
                _ => Err(Error::new(ENOSYS))
            }
        }
    }


    // The next lines set the current syscall in the context struct, then once the inner() function
    // completes, we set the current syscall to none.
    //
    // When the code below falls out of scope it will release the lock
    // see the spin crate for details
    {
        let contexts = crate::context::contexts();
        if let Some(context_lock) = contexts.current() {
            let mut context = context_lock.write();
            context.syscall = Some((a, b, c, d, e, f));
        }
    }

    let result = inner(a, b, c, d, e, f, bp, stack);

    {
        let contexts = crate::context::contexts();
        if let Some(context_lock) = contexts.current() {
            let mut context = context_lock.write();
            context.syscall = None;
        }
    }

    // errormux turns Result<usize> into -errno
    Error::mux(result)
}
```

まず関数内で`inner`という関数を定義しています。この関数は`Result`型を返しますが、最終的に`Error::mux`によりエラーコードを示す`usize`型に変換されています。
IDにマスクをかけた結果によっての条件分岐がまず入ります。
これはファイルディスクリプタを扱うものか、ファイルのパスを扱うものか、それ以外かで分岐しているのがわかります。
今回は代表として`close`と`open`を軽く見ていきましょう。突っ込みすぎると本質から外れていくのと自分の資料つくる時間が足りないので…

```rust
pub fn close(fd: FileHandle) -> Result<usize> {
    let file = {
        let contexts = context::contexts();
        let context_lock = contexts.current().ok_or(Error::new(ESRCH))?;
        let context = context_lock.read();
        context.remove_file(fd).ok_or(Error::new(EBADF))?
    };

    file.close()
}
```

`context`というのはコンテキストスイッチのコンテキストでカーネルの様々な状態が格納されているものです。コンテキストの中には現在開いているファイルが`Vec`で管理されています。
`remove_file`の定義はこうなっています。

```rust
    pub fn remove_file(&self, i: FileHandle) -> Option<FileDescriptor> {
        let mut files = self.files.write();
        if i.into() < files.len() {
            files[i.into()].take()
        } else {
            None
        }
    }
```

`FileHandle`が`usize`に変換されて、そのインデックスのファイルが`take`によって取り出されています。

続いて`open`を見ましょう。

```rust
pub fn open(path: &str, flags: usize) -> Result<FileHandle> {
    let (mut path_canon, uid, gid, scheme_ns, umask) = {
        let contexts = context::contexts();
        let context_lock = contexts.current().ok_or(Error::new(ESRCH))?;
        let context = context_lock.read();
        (context.canonicalize(path), context.euid, context.egid, context.ens, context.umask)
    };

    let flags = (flags & (!0o777)) | ((flags & 0o777) & (!(umask & 0o777)));


    for _level in 0..32 { // XXX What should the limit be?

        let mut parts = path_canon.splitn(2, ':');
        let scheme_name_opt = parts.next();
        let reference_opt = parts.next();

        let (scheme_id, file_id) = {
            let scheme_name = scheme_name_opt.ok_or(Error::new(ENODEV))?;
            let (scheme_id, scheme) = {
                let schemes = scheme::schemes();
                let (scheme_id, scheme) = schemes.get_name(scheme_ns, scheme_name).ok_or(Error::new(ENODEV))?;
                (scheme_id, Arc::clone(&scheme))
            };
            let reference = reference_opt.unwrap_or("");
            let file_id = match scheme.open(reference, flags, uid, gid) {
                Ok(ok) => ok,
                Err(err) => if err.errno == EXDEV {
                    let resolve_flags = O_CLOEXEC | O_SYMLINK | O_RDONLY;
                    let resolve_id = scheme.open(reference, resolve_flags, uid, gid)?;

                    let mut buf = [0; 4096];
                    let res = scheme.read(resolve_id, &mut buf);

                    let _ = scheme.close(resolve_id);

                    let count = res?;

                    let buf_str = str::from_utf8(&buf[..count]).map_err(|_| Error::new(EINVAL))?;

                    let contexts = context::contexts();
                    let context_lock = contexts.current().ok_or(Error::new(ESRCH))?;
                    let context = context_lock.read();
                    path_canon = context.canonicalize(buf_str);

                    continue;
                } else {
                    return Err(err);
                }
            };
            (scheme_id, file_id)
        };

        let contexts = context::contexts();
        let context_lock = contexts.current().ok_or(Error::new(ESRCH))?;
        let context = context_lock.read();
        return context.add_file(FileDescriptor {
            description: Arc::new(RwLock::new(FileDescription {
                namespace: scheme_ns,
                scheme: scheme_id,
                number: file_id,
                flags: flags & !O_CLOEXEC,
            })),
            cloexec: flags & O_CLOEXEC == O_CLOEXEC,
        }).ok_or(Error::new(EMFILE));
    }
    Err(Error::new(ELOOP))
}
```

`context.canonicalize(path)`は`path`の相対パスを絶対パスに変換するものです。このときスキーマも解決されます。
この絶対パスを元にスキーマとファイルの実体をとってこようとしています。
スキーマごとに`open`が実装されていたのは、前回の発表で見ましたね。`Scheme`トレイトのメソッドの1つです。
`EXDEV`でのエラーが発生した場合、`continue`して再度`open`を試みています。
これはCross-device linkという別のデバイスのファイルにハードリンクされている場合発生するエラーのようです。
このへんの処理はちょっとよくわからなかったので、次以降の人の解説に期待です。

最終的には`context`に新しいファイルディスクリプタを追加しておしまいです。

## aarch64向けの実装は？
かなり軽くしか見ていないのですが、aarch64アーキテクチャではシステムコールは例外の一種であるので、普通の例外ハンドリングと大差ない感じで実装されていました。
そのため、x86_64よりはいろいろと楽そうに見えました。

## 参考になる資料

- [opv86](https://hikalium.github.io/opv86/)
- [Intel SDM Vol.3 Chapter 5.8](https://www.intel.co.jp/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-system-programming-manual-325384.pdf#G10.1604)
- [Intel SDM Vol.4 Model-Specific Registers](https://www.intel.com/content/dam/develop/external/us/en/documents/335592-sdm-vol-4.pdf)
- マイナビ出版「ゼロからのOS自作入門」　第20章　システムコール

## 次の人向けのおもしろそうなネタ

- `ptrace`。デバッグ周りの実装がどうなっているかを調べるのは楽しそう
- コンテキストスイッチやスケジューリング周り
- `funmap`システムコール。メモリ管理周りのシステムコールは少し覗いたがややこしそう
- その他特定のシステムコールを掘り下げるだけでも1回分になりそう
