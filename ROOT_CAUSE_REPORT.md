SM-A146M (Exynos 1330)
Integrated Report — KVM, EL1/EL2, and H-Arx
Visual Problem Map + Detailed Firmware Investigation
Target Device: Samsung SM-A146M / Exynos 1330 Observed Kernel: Linux 5.15.180
PART I — Visual Map and Investigation Notebook
Summary and map explaining why /dev/kvm is missing.
1. Current Visual Map of the /dev/kvm Problem
                 A146M BOOT SEQUENCE
                          │
                          ▼
             ┌──────────────────────────┐
             │ Samsung Boot Chain       │
             │ BL / firmware / H-Arx    │
             └────────────┬─────────────┘
                          │
                          │
                          ▼
               ┌─────────────────────┐
               │ Linux 5.15.180      │
               │ Starts at EL1       │
               └─────────┬───────────┘
                         │
                         ▼
               CPU: All CPU(s)
                  started at EL1
                         │
                         ▼
               ┌─────────────────────┐
               │ KVM initcall        │
               │ arm_init()          │
               └─────────┬───────────┘
                         │
                         ▼
               kvm_arch_init()
                         │
                         ▼
             is_hyp_mode_available()
                         │
                         ▼
                      FALSE
                         │
                         ▼
             "HYP mode not available"
                         │
                         ▼
                    -ENODEV
                         │
                         ▼
               kvm_init() aborts
                         │
                         ▼
             misc_register(kvm_dev)
                  NEVER executes
                         │
                         ▼
                   /dev/kvm
                DOES NOT EXIST


This is the exact pathway proven experimentally.
2. The issue is NOT that the KVM driver is uncompiled
The kernel contains:
CONFIG_VIRTUALIZATION=y
CONFIG_KVM=y
CONFIG_KVM_VFIO=y
CONFIG_KVM_S2MPU=y
CONFIG_KVM_MMIO=y
CONFIG_KVM_GENERIC_DIRTYLOG_READ_PROTECT=y
CONFIG_KVM_XFER_TO_GUEST_WORK=y


And the KVM module/driver effectively executes initialization arguments passed to the kernel:
INITCALL TEST:
__initstub__kmod_kvm__617_2388_arm_init6

KVM TEST: arm_init ENTER

KVM TEST: kvm_arch_init ENTER


Therefore:
KVM compiled       ✓
KVM initcall      ✓
arm_init()        ✓
kvm_arch_init()   ✓


The failure occurs inside the architectural initialization of KVM.
3. The Exact Point of Failure
The function is:
kvm_arch_init()


It begins by checking:
if (!is_hyp_mode_available()) {
    kvm_info("HYP mode not available\n");
    return -ENODEV;
}


And kernel instrumentation showed:
KVM TEST: hyp_available=0
KVM TEST: kernel_hyp=0
KVM TEST: mode=0

kvm [1]: HYP mode not available

KVM TEST: arm_init EXIT rc=-19


-19 = -ENODEV.
Therefore the chain is:
is_hyp_mode_available()
        │
        └── 0
             │
             ▼
       return -ENODEV
             │
             ▼
       KVM initialization
            aborts


4. Why does is_hyp_mode_available() return false?
In Samsung's kernel source:
static inline bool is_hyp_mode_available(void)
{
    if (is_pkvm_initialized())
        return true;

    return (__boot_cpu_mode[0] == BOOT_CPU_MODE_EL2 &&
            __boot_cpu_mode[1] == BOOT_CPU_MODE_EL2);
}


That is, for the conventional pathway:
CPU 0 started in EL2
        AND
CPU 1 started in EL2
        ↓
HYP available


However, under Termux, the kernel logs report:
CPU: All CPU(s) started at EL1


And runtime testing showed:
hyp_available = 0
kernel_hyp    = 0


Therefore:
Linux is not in EL2
        │
        ▼
EL2 is not available for KVM
        │
        ▼
KVM aborts before registering /dev/kvm


5. This also explains kvm-arm.mode=protected
The kernel command line parameter:
kvm-arm.mode=protected


was kept in CONFIG_CMDLINE.
However, the kernel itself logs:
KVM is not available. Ignoring kvm-arm.mode


This is crucial: it does not mean protected mode causes the issue.
The sequence is approximately:
boot
 │
 ├── Checks HYP availability
 │       │
 │       └── NOT available
 │
 └── "KVM is not available"
          │
          └── Ignores kvm-arm.mode


Subsequently:
kvm_get_mode() = 0


This mode=0 is a consequence of HYP being unavailable, not the primary cause.
6. Why /dev/kvm disappears for a very specific reason
In virt/kvm/kvm_main.c, the sequence is essentially:
kvm_init()
   │
   ▼
kvm_arch_init()
   │
   ├── success → continues
   │
   └── -ENODEV → aborts
   │
   ▼
misc_register(&kvm_dev)
   │
   ▼
/dev/kvm


Since we have:
kvm_arch_init() → -ENODEV


Then:
misc_register(&kvm_dev)


is never executed.
Consequently, the absence of:
/dev/kvm
/sys/class/misc/kvm
/proc/misc → kvm


is a direct architectural result.
7. Findings Regarding Samsung's EL2 Layer
Here emerged the most interesting part of the investigation. The device features a Samsung component named:
Exynos H-Arx


along with related components:
H-Arx
uH
Exynos EL2
S2MPU
RKP/KDP


The bootloader firmware package contains:
harx.bin
uh.bin.lz4


harx.bin contains code explicitly executing in EL2.
For instance, direct register accesses were found:
msr SCTLR_EL2, x0
msr VBAR_EL2, x0
msr HCR_EL2, x0
msr MDCR_EL2, x0


This is direct evidence that Samsung code operates at EL2.
8. H-Arx is NOT Linux KVM
This distinction is fundamental.
We have:
                 EL2
                  │
          ┌───────┴────────┐
          │ Samsung H-Arx  │
          │ / firmware     │
          └────────────────┘


whereas Linux KVM would conceptually require:
                 EL2
                  │
          ┌───────┴────────┐
          │ Linux KVM hyp  │
          └────────────────┘
                  │
                 EL1
                  │
          Normal Linux


Findings indicate EL2 already has a function permanently reserved for Samsung's infrastructure, meaning Linux does not receive EL2 as a conventional HYP execution environment.
9. Exynos EL2 driver does not resolve this
The driver:
drivers/soc/samsung/exynos/exynos-el2.c


is loaded and running:
exynos_el2 ... Live


It uses HVC calls such as:
HVC_FID_SET_EL2_LV3_TABLE_BUF
HVC_FID_CHECK_STAGE2_ENABLE
HVC_FID_GET_EL2_LOG_BUF
HVC_FID_DELIVER_LV3_ALLOC_IRQ_NUM
HVC_FID_SET_EL2_CRASH_INFO_FP_BUF


However, this driver does not perform an EL1 → EL2 transition. It acts as an interface between Linux and Samsung's EL2 infrastructure.
Therefore:
exynos-el2 loaded
        ≠
Linux KVM possesses EL2


10. stage2 is not Linux KVM's Stage-2
Querying sysfs reveals:
/sys/devices/platform/exynos-el2/exynos_stage2_enable


returns:
stage2 is disable


This Stage-2 mechanism belongs to Samsung/H-Arx infrastructure. It cannot be naively interpreted as:
stage2 is disable
        ↓
KVM is disabled


What was already proven previously: Linux simply has no HYP mode available.
11. H-Arx actively manipulates EL2
At the beginning of harx.bin, a significant sequence was found:
msr SCTLR_EL2, x0
...
msr VBAR_EL2, x0
...
msr HCR_EL2, x0
...
msr MDCR_EL2, x0


Additionally, multiple paths feature:
msr SPSR_EL2, ...
msr ELR_EL2, ...
eret


Initially, this appeared to be a potential handoff transition:
EL2 → EL1 Linux


However, after examining the execution paths, the main eret instructions found are associated with:
crash handling
exception handling
context save/restore
SEH
EL2 exception dispatcher


For example, offsets DE14 / DEA0 save:
SPSR_EL2
ELR_EL2
HCR_EL2
ESR_EL2


and later at E1B0 / E1D8, restore context and execute eret.
This represents exception/context return logic, not a normal handoff to the Linux kernel.
Therefore:
Encountering ERET
        ≠
Finding normal handoff (EL2 → Linux EL1)


This hypothesis was discarded as a sufficient explanation for kernel entry.
12. Crucial Discovery in uh.bin Firmware
uh.bin differs structurally from harx.bin. It contains Rust components:
RUSTUH
harx_rlp
src/bsp/chipset/slsi/harx/mod.rs
src/platform/memory_map/physmap.rs
src/platform/guest_mem.rs
src/platform/region_manager.rs
src/app/rkp/...


Along with symbols:
GuestMem
reserved_mem
el2_va
el2_pa


Furthermore, explicit assertion invariants exist:
assertion failed: self.reserved_mem.start.el2_va == 0


and:
assertion failed: crate::ALIGNED!(reserved_mem_start, 2 * MB)


This demonstrates that uH explicitly models reserved memory for EL2 and guest memory spaces, consistent with Samsung maintaining EL2 hardware resources strictly separated from Linux.
13. Strong Device Tree Correlation
On the running device, the kernel imports:
reserved-memory/
    el2_code
    el2_earlymem


configured as:
el2_code:
    base = 0xC1400000
    size = 0x00200000 (2 MiB)

el2_earlymem:
    base = 0x8FF200000
    size = 0x00E00000 (14 MiB)


And:
exynos-el2 {
    compatible = "samsung,exynos-el2";
    interrupts = ...
};


While source DTS trees may omit these nodes, the runtime DTB passed to Linux includes them.
DTS Source
   │
   │ (omits nodes)
   ▼
Boot chain / DT modification / Secondary DT
   │
   ▼
Final Device Tree
   │
   ├── el2_code
   ├── el2_earlymem
   └── exynos-el2


14. Connection Between uh.bin and el2_code
The uH plugin header contains memory offsets:
PLUG-IN

C1401000
C1438000
C1438000
C1443000
C1443000
C144D000

harx_rlp


Matching the active Device Tree node el2_code = 0xC1400000.
uH plugin
   │
   ├── C1401000
   ├── C1438000
   ├── C1443000
   └── C144D000
          │
          ▼
el2_code region
C1400000 – C1600000


15. harx.bin Internal References to 0xC1400000
Scanning harx.bin for addresses within 0xC1400000 – 0xC1600000 revealed numerous pointers:
C1438125
C1405294
...


Reinforcing that 0xC1400000 is the primary execution/data region for the H-Arx/uH hypervisor.
16. Probable Meaning of Offset 0x1C4F0
Investigation showed:
[0x1C4F0] = 0xC000013C


And at D874:
ldr x0, [x1]


returns this value.
Then at 0x30:
bl D874
cbz x0, ...
br x0


Where 0xC000013C corresponds to offset 0x13C inside harx.bin. There, EL2 initialization executes again:
msr VBAR_EL2, ...
msr SCTLR_EL2, ...
msr HCR_EL2, ...


This is an internal pointer for H-Arx re-entry/re-initialization, not a pointer to Linux entry points.
17. Unresolved Questions
Central question remaining:
At which exact boot stage is Linux placed in EL1 while EL2 remains locked by H-Arx/firmware?
Uncertain components: BL, ABL, uH, H-Arx, or specific Samsung bootloader handoff code.
18. Comprehensive Architectural Map
                         Samsung BL
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
          harx.bin                    uh.bin
          H-Arx                      Rust uH
              │                           │
              │                           │
              ├── EL2 code                ├── GuestMem
              ├── HCR_EL2                ├── reserved_mem
              ├── SCTLR_EL2              ├── physmap
              ├── VBAR_EL2               ├── region_manager
              ├── H-Arx exceptions       └── RKP/KDP
              │
              ▼
       ┌───────────────────────┐
       │ Reserved EL2 Resources│
       │                       │
       │ el2_code              │
       │ 0xC1400000 / 2 MiB    │
       │                       │
       │ el2_earlymem          │
       │ 0x8FF200000 / 14 MiB  │
       └───────────┬───────────┘
                   │
                   ▼
             EL2 Reserved
           to Samsung Infra
                   │
                   │
                   ▼
          ┌─────────────────┐
          │ Linux ARM64     │
          │                 │
          │ Starts at EL1   │
          └────────┬────────┘
                   │
                   ▼
             KVM initcall
                   │
                   ▼
            kvm_arch_init()
                   │
                   ▼
        is_hyp_mode_available()
                   │
                   X
                  false
                   │
                   ▼
             -ENODEV
                   │
                   ▼
          /dev/kvm not created


19. Core Working Hypothesis
The SM-A146M boot chain/firmware uses EL2 exclusively for H-Arx/uH/security infrastructure, forcing Linux to boot at EL1.
This simultaneously explains:
CPU(s) started at EL1
is_hyp_mode_available() = 0
kvm_arch_init() = -ENODEV
Missing /dev/kvm
Active Samsung infrastructure in EL2
20. SBOOT Analysis Findings
Analyzing sboot.bin revealed:
Specific code handling EL2/H-Arx.
The diagnostic string "CAN NOT enter EL2".
SMC calls validating memory descriptors.
Execution drops through 17B81C → 17B19C → 17B71C → 17AEB4 → 17B268 → 17B608.
17B268 validates a 0x100-byte block containing five region descriptors.
17B608 allocates buffers and invokes SMC -0xF1.
sboot.bin prepares and validates memory, while harx.bin contains the actual MSR HCR_EL2, SCTLR_EL2, VBAR_EL2 instructions.
21. Radare2 Disassembly Analysis of Exception Level Transitions
Commands targeting potential EL3/EL2 → EL1 transitions:
pdf @ 0xEB818
pd 80 @ 0xEB800
pd 80 @ 0xEB9A0
pd 80 @ 0xEC5C0
pdf @ 0xEB9BC
axt @ 0xEB818
axt @ 0xEB85C
axt @ 0xEB9BC


Findings:
0xEB818 configures system registers and executes ERET, setting ELR_EL3 = 0xEB8CC and SPSR_EL3 = 0x3C5 (selecting EL1h).
0xEB85C prepares a return to EL1 when called from EL3 or EL2.
0xEB9BC configures VBAR_EL1 and calls 0xEB818 when detecting execution in EL2 or EL3.
Trace sequence:
pd @ 0xE9000
axt @ 0xEB9BC
axt @ 0xEC5C8
pdf @ 0xEC5C8
pdf @ 0x101348
pdf @ 0x1019F4
axt @ 0xE9000


Identified chain:
0xE9000 → 0xEC5C8 → 0xEB9BC → 0xEB818 → ERET → EL1 → 0x1019F4


22. Deep Analysis of 0x32000 (Vector Setup and Exception Level Logic)
Disassembly dump at 0x32000:
; XREFS: DATA 0x00032164  DATA 0x00032268  DATA 0x00032510
; XREFS: DATA 0x0003253c  DATA 0x00032630  DATA 0x00032658
┌─< 0x00032000      01000014       b 0x32004
│   ; CODE XREF from fcn.0002e028 @ +0x3fd8(x)
└─> 0x00032004      e07f0010       adr x0, 0x33000
0x00032008      414238d5       mrs x1, currentel
0x0003200c      3f2000f1       cmp x1, 8
┌─< 0x00032010      00010054       b.eq 0x32030
│   0x00032014      3f1000f1       cmp x1, 4
┌──< 0x00032018      00010054       b.eq 0x32038
││   0x0003201c      00c01ed5       msr vbar_el3, x0
││   0x00032020      00113ed5       mrs x0, scr_el3
││   ; CODE XREF from fcn.0003262c @ +0x1f130(x)
││   0x00032024      000c40b2       orr x0, x0, 0xf
││   0x00032028      00111ed5       msr scr_el3, x0
┌───< 0x0003202c      06000014       b 0x32044
│││   ; CODE XREF from fcn.0002e028 @ +0x3fe8(x)
││└─> 0x00032030      00c01cd5       msr vbar_el2, x0
││┌─< 0x00032034      04000014       b 0x32044
│││   ; CODE XREF from fcn.0002e028 @ +0x3ff0(x)
│└──> 0x00032038      00c018d5       msr vbar_el1, x0
│ │   0x0003203c      0006a0d2       mov x0, 0x300000
│ │   0x00032040      401018d5       msr cpacr_el1, x0
│ │   ; CODE XREFS from fcn.0002e028 @ +0x4004(x), +0x400c(x)
└─└─> 0x00032044      001038d5       mrs x0, sctlr_el1
0x00032048      00001432       orr w0, w0, 0x1000
0x0003204c      001018d5       msr sctlr_el1, x0
0x00032050      c0050058       ldr x0, 0x32108             ; [0x32108:8]=0x2036830
0x00032054      e1050058       ldr x1, 0x32110             ; [0x32110:8]=0x2036838
0x00032058      1f0001eb       cmp x0, x1
┌─< 0x0003205c      80000054       b.eq 0x3206c
│   ; CODE XREF from fcn.0002e028 @ +0x4040(x)
┌──> 0x00032060      1f4400b8       str wzr, [x0], 4
╎│   0x00032064      1f0001eb       cmp x0, x1
└──< 0x00032068      c3ffff54       b.lo 0x32060
│   ; CODE XREF from fcn.0002e028 @ +0x4034(x)
└─> 0x0003206c      60050058       ldr x0, 0x32118             ; [0x32118:8]=0x11860804
0x00032070      010040b9       ldr w1, [x0]
0x00032074      a2758152       mov w2, 0xbad
0x00032078      3f00026b       cmp w1, w2
┌─< 0x0003207c      81000054       b.ne 0x3208c
│   0x00032080      00050058       ldr x0, 0x32120             ; [0x32120:8]=0x2039000
│   0x00032084      1fec7c92       and sp, x0, 0xfffffffffffffff0
┌──< 0x00032088      03000014       b 0x32094
││   ; CODE XREF from fcn.0002e028 @ +0x4054(x)
│└─> 0x0003208c      e0040058       ldr x0, 0x32128             ; [0x32128:8]=0x14f85000
│    0x00032090      1fec7c92       and sp, x0, 0xfffffffffffffff0
│    ; CODE XREF from fcn.0002e028 @ +0x4060(x)
└──> 0x00032094      a8000094       bl fcn.00032334
┌─< 0x00032098      7b000014       b 0x32284
│   0x0003209c      a0040058       ldr x0, 0x32130             ; [0x32130:8]=0x11860404
│   0x000320a0      010040b9       ldr w1, [x0]
│   0x000320a4      21000312       and w1, w1, 0x20000000
│   ; DATA XREF from fcn.00009a78 @ +0x1abc(r)
│   0x000320a8      3f000372       tst w1, 0x20000000
┌──< 0x000320ac      80000054       b.eq 0x320bc
││   0x000320b0      40030058       ldr x0, 0x32118             ; [0x32118:8]=0x11860804
││   0x000320b4      01008052       mov w1, 0
││   0x000320b8      010000b9       str w1, [x0]
││   ; CODE XREF from fcn.0002e028 @ +0x4084(x)
└──> 0x000320bc      c0035fd6       ret
│   0x000320c0      80030058       ldr x0, 0x32130             ; [0x32130:8]=0x11860404
│   ; CODE XREF from fcn.0003262c @ +0xf1c8(x)
│   0x000320c4      010040b9       ldr w1, [x0]
│   0x000320c8      21000812       and w1, w1, 0x1000000
│   0x000320cc      3f000872       tst w1, 0x1000000
┌──< 0x000320d0      80000054       b.eq 0x320e0
││   0x000320d4      20020058       ldr x0, 0x32118             ; [0x32118:8]=0x11860804
││   0x000320d8      01008052       mov w1, 0
││   0x000320dc      010000b9       str w1, [x0]
││   ; CODE XREF from fcn.0002e028 @ +0x40a8(x)
└──> 0x000320e0      c0035fd6       ret
│   0x000320e4      60020058       ldr x0, 0x32130             ; [0x32130:8]=0x11860404
│   0x000320e8      010040b9       ldr w1, [x0]
│   0x000320ec      21000c12       and w1, w1, 0x100000
│   0x000320f0      3f000c72       tst w1, 0x100000
┌──< 0x000320f4      80000054       b.eq 0x32104
││   0x000320f8      00010058       ldr x0, 0x32118             ; [0x32118:8]=0x11860804
││   0x000320fc      01008052       mov w1, 0
││   0x00032100      010000b9       str w1, [x0]
││   ; CODE XREF from fcn.0002e028 @ +0x40cc(x)
└──> 0x00032104      c0035fd6       ret
│   ; DATA XREF from fcn.0002e028 @ +0x4028(r)
│   0x00032108      30680302       invalid
│   0x0003210c      00000000       invalid
│   ; DATA XREF from fcn.0002e028 @ +0x402c(r)
│   0x00032110      38680302       invalid
│   0x00032114      00000000       invalid
│   ; DATA XREFS from fcn.0002e028 @ +0x4044(r), +0x4088(r), +0x40ac(r), +0x40d0(r)
│   0x00032118      04088611       invalid
│   0x0003211c      00000000       invalid
│   ; DATA XREF from fcn.0002e028 @ +0x4058(r)
│   0x00032120      00900302       invalid
│   0x00032124      00000000       invalid
│   ; DATA XREF from fcn.0002e028 @ +0x4064(r)
│   0x00032128      0050f814       b 0x3e46128
│   ; CODE XREF from fcn.0003262c @ +0x1f238(x)
│   0x0003212c      00000000       invalid
│   ; DATA XREFS from fcn.0002e028 @ +0x4074(r), +0x4098(r), +0x40bc(r)
│   0x00032130      04048611       invalid
│   0x00032134      00000000       invalid
│   0x00032138      030000d4       smc 0
│   0x0003213c      c0035fd6       ret
│   0x00032140      e2031faa       mov x2, xzr
│   0x00032144      9f3f03d5       dsb sy
│   0x00032148      030000d4       smc 0
│   0x0003214c      e00302aa       mov x0, x2
│   0x00032150      c0035fd6       ret
│   0x00032154      e0031f2a       mov w0, wzr
│   0x00032158      c0035fd6       ret
│   0x0003215c      0a009c52       mov w10, 0xe000
│   0x00032150      0890be52       mov w8, -0xb800000
│   0x00032164      09000090       adrp x9, 0x32000
│   0x00032168      ea8fbe72       movk w10, 0xf47f, lsl 16
│   0x0003216c      2a1904f9       str x10, [x9, 0x830]
│   ; DATA XREFS from fcn.0002c73c @ 0x2c918(r), 0x2c944(r)
│   0x00032170      480100b9       str w8, [x10]
│   0x00032174      221944f9       ldr x2, [x9, 0x830]
│   0x00032178      0a808952       mov w10, 0x4c00
│   0x0003217c      0908a052       mov w9, 0x400000
│   0x00032180      4a40a072       movk w10, 0x202, lsl 16
│   0x00032184      1f800071       cmp w0, 0x20
│   0x00032188      490400b9       str w9, [x2, 4]
│   0x0003218c      4a0400f9       str x10, [x2, 8]


Xref query:
[0x00000000]> axt @ 0x32094
(nofunc) 0x32088 [CODE:--x] b 0x32094


Hex dumps at 0x32000 and MMIO address 0x11862D04:
[0x00000000]> px 128 @ 0x32000

- offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
  0x00032000  0100 0014 e07f 0010 4142 38d5 3f20 00f1  ........AB8.? ..
  0x00032010  0001 0054 3f10 00f1 0001 0054 00c0 1ed5  ...T?......T....
  0x00032020  0011 3ed5 000c 40b2 0011 1ed5 0600 0014  ..>...@.........
  0x00032030  00c0 1cd5 0400 0014 00c0 18d5 0006 a0d2  ................
  0x00032040  4010 18d5 0010 38d5 0000 1432 0010 18d5  @.....8....2....
  0x00032050  c005 0058 e105 0058 1f00 01eb 8000 0054  ...X...X.......T
  0x00032060  1f44 00b8 1f00 01eb c3ff ff54 6005 0058  .D.........T`..X
  0x00032070  0100 40b9 a275 8152 3f00 026b 8100 0054  ..@..u.R?..k...T

[0x00000000]> px 128 @ 0x11862d04
- offset -   4 5  6 7  8 9  A B  C D  E F 1011 1213  456789ABCDEF0123
  0x11862d04  ffff ffff ffff ffff ffff ffff ffff ffff  ................
  0x11862d14  ffff ffff ffff ffff ffff ffff ffff ffff  ................
  0x11862d24  ffff ffff ffff ffff ffff ffff ffff ffff  ................
  0x11862d34  ffff ffff ffff ffff ffff ffff ffff ffff  ................
  0x11862d44  ffff ffff ffff ffff ffff ffff ffff ffff  ................
  0x11862d54  ffff ffff ffff ffff ffff ffff ffff ffff  ................
  0x11862d64  ffff ffff ffff ffff ffff ffff ffff ffff  ................
  0x11862d74  ffff ffff ffff ffff ffff ffff ffff ffff  ................


Key Breakdown:
0x32004–0x32040: Reads CurrentEL and sets VBAR_EL3, VBAR_EL2, or VBAR_EL1 dynamically to vector base 0x33000.
0x32044–0x3204C: Enables bit 0x1000 in SCTLR_EL1.
0x32050–0x32068: Clears memory between 0x2036830 and 0x2036838.
0x3206C–0x32094: Evaluates control register 0x11860804 to select stack pointers before calling fcn.00032334.
PART II — Detailed Technical Analysis of SBOOT and H-Arx
Consolidation of reverse engineering, static analysis (sboot.bin, harx.bin), and runtime corroboration documenting the complete chain from chip reset in EL3 to Linux kernel handoff in EL1.
1. Executive Summary
Linux receives control at EL1, never at EL2. This is an architectural policy enforced by boot firmware, not a bug, conditional failure, or hardware defect.
Key Discoveries:
force_EL1 (0xEB818 / 0xEB85C in sboot.bin) exclusively generates SPSR = EL1h. No alternate branches produce EL2h.
The final kernel jump (blr x8 at 0x11412C) does not use eret. Zero eret instructions exist globally between 0xEB8C8 and 0x114130.
EL2 is permanently occupied by Samsung's proprietary hypervisor (H-Arx), configuring VBAR_EL2, HCR_EL2, SCTLR_EL2, and protecting memory via S2MPU.
SBOOT probes H-Arx for EL2 resource availability via HVC 0x82002060. H-Arx returns -1. SBOOT accepts this and proceeds.
Runtime Corroboration: Linux init_kernel_el (head.S) reads CurrentEL and logs:
CPU: All CPU(s) started at EL1


2. Methodology & Tooling
Analysis phases:
Raw Disassembly Scans: String/grep pattern matching across raw instructions.
Radare2 (r2 -a arm -b 64): Used for pdf, axt, px. Noted non-deterministic boundary analysis across sessions.
Capstone Python Engine: Direct byte-level deterministic disassembly. Corrected axt cross-reference discrepancies.
3. Confirmed Boot Sequence Pipeline
Reset (EL3 @ sboot.bin 0x20)
 │  Header size=0x98, magic "head", branch to 0x20
 │  SCTLR_EL3 = 0x30C50830
 │  MPIDR_EL1 CPU gate (traps non-primary cores)
 │  Embedded public key trust root (~0x92E0+)
 │  VBAR_EL3 → 0xB000 (defensive "b self")
 │  SCR_EL3 = 0x238 (EA, SIF — NO RW bit, NO HCE)
 ▼
force_EL1() @ 0xEB818 / 0xEB85C
 │  SCR_EL3 |= 0x400 (Bit RW set)
 │  HCR_EL2 = 0x80000000, CPTR_EL2 = 0x33FF, CPACR_EL1 = 0x300000
 │  SPSR = 0x3C5 (EL1h, DAIF masked) — ONLY possible target
 │  eret
 ▼
EL1h Execution — ZERO eret between 0xEB8C8 and 0x114130
 │  sboot/LK routines: boot.img parsing
 │  Diagnostic log "CAN NOT enter EL2" @ 0x112FF4
 │  HVC queries to H-Arx
 │  Kernel load address calculation via SMC #0 (0xC200101C)
 ▼
blr x8 @ 0x11412C
 │  x0 = 0xA0000000 (Physical DTB address)
 │  x8 = Kernel entry point
 │  NO eret — Direct jump in EL1
 ▼
Linux Kernel Entry (EL1)


Parallel EL2 Resident Architecture:
H-Arx (harx.bin) — Active at EL2 before SBOOT execution
 │  Entry: b 0x20
 │  SCTLR_EL2 = 0x30C50830
 │  VBAR_EL2 → 0xF000 (Internal exception table)
 │  HCR_EL2 = 0x8000800000000020 (RW=1, AMO=1, FIEN=1, HCD=0)
 ▼
Handles SBOOT HVC requests @ VBAR+0x400
 │  Routes SMCCC Owner 6 (Vendor Hypervisor)
 │  Dispatches via fcn.172c


4. EL3 Initialization Sequence (sboot.bin offset 0x20)
0x20-0x38: Stack setup from static pointers
0x3C:      bl fcn.158 (Zero-BSS / Data copy)
0x50-0x58: msr SCTLR_EL3, x0 (x0 = 0x30C50830)
0x64-0x68: cbz x0, 0x6c / br x0 (Warm-boot/PSCI CPU_ON check)
0x6C-0x70: adr x0, 0xb000 / msr VBAR_EL3, x0
0x9C:      msr SCR_EL3, x0 (x0 = 0x238)
0xBC:      msr MDCR_EL3, x0
0xE0:      msr CPTR_EL3, x0
0xF8:      DIT instruction
0xFC-0x144: Sequence of bl calls → b 0x872c (main loop)


fcn.149c/fcn.1444 extract MPIDR_EL1 affinity values, looping non-primary cores indefinitely (0x14c4).
5. force_EL1: Closing the Door to EL2
At 0xEB818 / 0xEB85C:
eb818: mrs x0, SCR_EL3
eb81c: orr x0, x0, #0x400       ; Sets RW bit (64-bit lower EL)
eb820: msr SCR_EL3, x0
eb824: mov x0, #0x80000000
eb828: msr HCR_EL2, x0
eb82c: mov x0, #0x33ff
eb830: msr CPTR_EL2, x0
eb834: mov x0, #0x300000
eb838: msr CPACR_EL1, x0
eb83c: mov x0, sp
eb840: msr SP_EL1, x0
eb844: adr x0, 0xeb8cc
eb848: msr ELR_EL3, x0
eb84c: mov x0, #0x3c5            ; EL1h (DAIF masked)
eb850: msr SPSR_EL3, x0
eb858: eret


Entry point 0xEB85C checks CurrentEL (EL1/EL2/EL3), converging unconditionally on SPSR = 0x3C5 and eret at 0xEB8C8.
6. SBOOT / LK Execution & "CAN NOT enter EL2"
0x112FF4–0x113050 runs three validation checks (0x10b288, 0x10b76c, 0x10b534). The third validates region el2_code (0xC1400000). If checks fail, it logs "CAN NOT enter EL2" and returns 0. It is a non-fatal diagnostic.
SMC #0 invocation (0xC200101C, SiP Owner) calculates RAM ranges and populates [0x75ED04].
7. Final Kernel Jump (0x11412C)
114100: adrp x0, 0x257000 ; "Starting kernel..."
114108: bl   0x148d78     ; Logging
11410c: adrp x8, 0x75e000
114110: ldr  w8, [x8, #0xd04]
114114: mov  w9, #0x80000000
114118: mov  w0, #0xA0000000   ; x0 = DTB physical pointer
114120: add  x8, x8, x9        ; x8 = Kernel entry point
11412c: blr  x8                ; DIRECT JUMP — NO ERET


8. H-Arx Hypervisor Deep Dive
8.1 Strings in harx.bin
Exynos_H-Arx
[H-Arx Version] This image's H-Arx version ID [%s]
H-Arx Start
harx_ver@
exynos_s2mpu_protect_kernel_text
exynos_s2mpu_set_kernel_text_info
exynos_s2mpu_blacklist_init
stage2_device_init_s2mpu
H-Arx OEM Plug-in
H-Arx SIP Plug-in
H-Arx resume


8.2 EL2 Setup Routine (harx.bin offset 0x00–0x40)
0x00: b 0x20
0x20: msr SCTLR_EL2, x0  ; x0 = 0x30C50830
0x30: bl fcn.d874        ; Check cold vs warm boot
0x34: cbz x0, 0x3c       ; 0 = Cold boot
0x38: br x0              ; Warm boot jump
0x3c: adr x0, 0xf000     ; x0 = VBAR_EL2 address
0x40: msr vbar_el2, x0
0x44: isb
...
0x70: msr hcr_el2, x0   ; x0 = 0x0000800080000020
0x80: msr mdcr_el2, x0  ; Clear debug traps
0x8c: msr cptr_el2, x0  ; 0x33FF (No FP/SIMD traps)


Vector table located at 0xF000 inside harx.bin.
9. HVC Routing: SBOOT to H-Arx Interface
SBOOT issues HVC calls via wrapper fcn.107ee4:
107ee4: dsb sy
107ee8: hvc #0
107eec: ret


HVC Dispatch Mapping:
Function ID (x0)
SBOOT Caller
Arguments
H-Arx Handler
Action
0xC6000105
0x10af98
x1 = Base, x2 = Size, x3 = 0
0x1cf8 → fcn.54b4
Protect Kernel Text
0xC6000010
0x10b704
x1 = Base, x2/x3 from stack
0x19b4 → 0x319c
Validate/Register EL3
0xC6000040
0x10b428
x1 = 0x10b920, x2 = 0x743000
0x19c8
Validate with fcn.1178
0xC6000109
0x10b500
Unspecified
0x1888 → fcn.5b6c
Subsystem config
0xC6000020
0x10b080+
x1 = 0,1,2,5
0x17c8 → fcn.e14
Sub-command dispatch
0xC6000108
0x10ade4+
x1 = 0/1, x2 = w19
0x1888
General H-Arx call
0x82002060
0x10b1b8+
x1 = 1 (1st try), x1 = 2 (2nd try)
0x1d84
Returns -1

H-Arx dispatch function fcn.172c processes incoming calls through a cmp/b.eq cascade.
Kernel Text Protection Trace (0xC6000105):
SBOOT (0x10af98): hvc 0xC6000105
H-Arx Vector (0xF400): Lower EL Sync trap -> 0xDD88
EC Router (0xDE50): EC 0x16 (HVC64) -> fcn.172c
fcn.172c: Matches 0xC6000105 -> Calls fcn.54b4(base, size)
fcn.54b4: Aligns address to 64KB, invokes validation fcn.1178
fcn.1178: Verifies DRAM window bounds (0xC0000000–0xC0200000, 0xC1200000–0xC1800000)
SMC Target: Issues SMC #0 (x0 = 0x8200007D) to EL3 monitor for Stage-2 lockdown.
10. Probe 0x82002060: The Definitive Rejection
SBOOT executes function 0x10b160.
Checks conditions [0x742000+0x989] == 1 and [0x742000+0x98a] != 0.
Issues HVC 0x82002060 (x1 = 1).
H-Arx compares against valid function IDs. 0x82002060 falls through to default handler 0x1D84.
Handler writes -1 (0xFFFFFFFFFFFFFFFF) to x0 and returns.
SBOOT retries with x1 = 2. Result: -1.
SBOOT writes -1 to output buffers and returns success (w0 = 0).
Caller (0x1139C0) logs the result without branching. Boot flow continues unaffected.
11. Evidentiary Matrix
Statement / Finding
Status
Evidence Source
Core resets in EL3 at 0x20 in sboot.bin
Confirmed
Binary disassembly
blr x8 at 0x11412C enters kernel without level change
Confirmed
sboot.bin static disassembly
force_EL1 only outputs SPSR = EL1h
Confirmed
sboot.bin (0xEB818 / 0xEB85C)
Zero eret instructions between 0xEB8C8 and 0x114130
Confirmed
Exhaustive binary byte scan
Kernel log CPU: All CPU(s) started at EL1
Confirmed
Runtime dmesg log
"CAN NOT enter EL2" is non-fatal
Confirmed
Function returns 0 unconditionally
H-Arx active at EL2 with distinct vector tables
Confirmed
harx.bin disassembly (0x20–0x80)
SBOOT invokes H-Arx via HVC to protect kernel text
Confirmed
0xC6000105 trace in both binaries
H-Arx requests EL3 S2MPU lock via SMC 0x8200007D
Confirmed
harx.bin disassembly
Probe HVC 0x82002060 returns -1 without halting boot
Confirmed
SBOOT function 0x10b160 analysis
Identity of stage initializing H-Arx prior to SBOOT
Unresolved
Pre-SBOOT BL1 / ROM phase

12. Final Root Cause Determination
The Linux kernel on the Samsung SM-A146M (Exynos 1330) receives control in EL1 because the entire boot chain drops execution privileges to EL1 early in the boot sequence (force_EL1).
EL2 is permanently reserved for Samsung's H-Arx hypervisor, which initializes prior to SBOOT to manage Stage-2 Memory Protection (S2MPU) and kernel text integrity.
When SBOOT probes H-Arx for EL2 resource availability via HVC 0x82002060, H-Arx rejects the request (-1). SBOOT treats this rejection as expected behavior and continues booting Linux at EL1.
As a direct result, is_hyp_mode_available() returns false, kvm_arch_init() aborts with -ENODEV, and /dev/kvm is never registered.
13. Remaining Open Questions
Initial H-Arx Entry Point: The exact pre-SBOOT stage (BootROM, FWBL1, or dedicated EL3 monitor) that first loads and jumps into harx.bin at EL2 remains unmapped.
Specific HVC Sub-commands: Full functional specifications for secondary HVC function IDs (0xC6000040, 0xC6000109, sub-ops of 0xC6000020) remain unparsed, though they do not affect the EL1/EL2 handoff conclusion.
Exact SBOOT Partition Layout: Internal boundary offsets for concatenated components (fwbl1, epbl, bl2, lk) inside sboot.bin (totaling 8 MB, signed via AVB footer at offset 0x5C9330) are not individually demarcated.
14. Final Conclusion
Hardware-assisted virtualization via KVM cannot be enabled on the Samsung SM-A146M / SM-A146B through kernel modifications, configuration tweaks, or command-line parameters. The reservation of ARM64 EL2 by Samsung's H-Arx hypervisor is an immutable architectural boundary enforced by hardware-rooted Secure Boot.
