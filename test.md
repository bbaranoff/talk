---

**Title**
Emulating the baseband

**Subtitle**
About eight months to find a way to make a voice call with osmocom-bb layer1 and a homemade C54x DSP emulation on a virtual 2G network.

**Abstract**
A real GSM phone's baseband DSP ROM now runs, unmodified, on an ordinary PC, and you can stop it at an instruction and see what it is doing. The DSP is the least observable part of a phone: a vendor processor running a mask ROM nobody outside the manufacturer could inspect. Instead of replacing it, this project emulates the TI TMS320C54x closely enough to run the original ROM, next to a QEMU Calypso machine running OsmocomBB's real layer 1. The result is an end-to-end GSM call — attach, SMS, USSD, voice, A5 — with nothing on air, the ROM doing the downlink and part of the uplink. But the call is not the point: the DSP can now be single-stepped. Since April 2010 a `FIXME` in OsmocomBB has asked where a magic 23 came from. The ROM answers it.

**Description**
A Calypso-type baseband has two processors. The ARM runs the protocol stack, and OsmocomBB has made that side free software since 2010. The DSP runs the physical layer — demodulation, channel coding, A5 — from a mask ROM that was never documented outside the vendor and that OsmocomBB drives through a shared-memory API without knowing what happens behind it. For fifteen years, "the DSP does it" was where the free stack stopped. This project removes that wall by emulating the DSP rather than replacing it.

**What runs.** A QEMU fork with a TI Calypso machine (ARM946, TPU, RIF, DMA, shared API RAM) loads OsmocomBB's `layer1.highram` as the real chip would. A native emulator for the TMS320C54x executes the DSP mask ROM, frame-locked with QEMU over the same API RAM a real phone uses. The ROM dump is not redistributed: the one used is identical, word for word, to the public FreeCalypso dump. A bridge carries bursts to osmo-bts-trx, and an Osmocom core network (BSC, MSC, HLR, SIP) completes the call. An end-to-end test bench runs the whole chain — boot, attach, SMS in both directions, USSD, a voice call, a 1 kHz tone recovered through the vocoder — and the 4 October run reports 33 layer-1 functions handled by the chain, 1 degraded (downlink TCH/F quality flag), 1 not handled (frequency hopping, out of scope), with the DSP ROM doing FCCH detection, SCH decoding, all downlink channel decoding, speech decoding and A5. Uplink SDCCH is ROM-coded as well; RACH, FACCH/SACCH on TCH, uplink speech and uplink A5 are still encoded host-side in the full system, and that is the current work. The emulated DSP processes a 4.62 ms TDMA frame in 4.61 ms on a laptop.

**What you can do with it.** In effect, a root shell on each of the two processors of a baseband. The ARM firmware runs under gdb through QEMU's gdbstub. The DSP has its own shell: stop it, dump the shared API RAM with every field of the ARM–DSP interface named and decoded, step it through the ROM. Put a breakpoint at the SB correlator, watch the 50-lag energy table fill, see which lag the ROM keeps and why. Two results from doing exactly that. OsmocomBB's `prim_fbsb.c` has carried `/* FIXME: where did this magic 23 come from? */` since April 2010. The ROM hard-codes a 191-sample SB window (`stm #191,ar3`), and 23 is the only margin for which layer 1's window formula reproduces it: 148 + 2×23 − 3 = 191. The full chain (−23, +75 qbits, 16-qbit target) was then checked against logs from real Calypso phones found in the mailing-list archives: they land on the predicted values. A double subtraction nobody had noticed, present since a May 2010 refactoring, fell out on the way. Second result: the ROM's burst emitter writes the complete access burst into the ABB script — training sequence, tails, I/Q inversion, A5 — bit-exact against libosmocoding. It took two missing instructions in my core (`CMPR`, `LD src,ASM`) to let it.

**Honest state.** The emulated core passes 150 of TI's 230 worked instruction examples; 58 fail and 22 do not assemble yet. Every "the ROM does X" is conditional on that, and the talk says so. Most of the emulator's code was written with an AI assistant; what decides whether it is right is none of that: the unmodified TI ROM, the unmodified OsmocomBB stack, a shadow mode comparing ROM and host encoders bit by bit, and the end-to-end bench.

**Why it matters.** 2G is still carried in tunnels, mines, railways (GSM-R) and in most of the world's fallback paths. Its baseband has been a black box for twenty-five years. With this setup, anyone can watch one run, without hardware, without a licence, and check every claim against a public ROM dump and public source. The lab ships as a bootable ISO and Docker images, with srsRAN (4G over ZeroMQ, CSFB to the 2G core) and a multi-operator SS7 setup alongside.

**What the audience sees.** Live: the bench from boot to a voice call (about four minutes), with three consoles on screen — the DSP shell, gdb on layer 1, the phone's VTY:

```
P:9002 D> d@ d_task_d 20
    W0    [0800]=000d  d_task_d           task 13 = TCHT
    W0    [0801]=0000  d_burst_d

P:9002 D> d@ 0862 20
    API   [0862]=3c92
    API   [0863]=3f92
    API   [0864]=3c92
```

```
Reading symbols from /opt/GSM/firmware/board/compal_e88/layer1.highram.elf...
The target architecture is set to "armv5te".
Remote debugging using 127.0.0.1:1234
l1a_l23_handler () at layer1/l23_api.c:650
650		msg = msgb_dequeue(&l23_rx_queue);
Continuing.
[cmd.gdb] osmocom panel loaded: help_osmo
```

```
% (MS 1)
% Call is answered
```

Then the inspection tools: the DSP paused at a frame boundary with its registers and the decoded ARM–DSP mailbox on screen, and gdb writing a string into the ARM's RAM that the firmware prints on its own console. Slides only for the architecture and the two results.

**What you will know on leaving.** How a 2G baseband DSP is actually driven, what its ROM does at the instruction level for sync and channel decoding, how to build an emulator good enough to find bugs in your own emulator by reading the vendor's code, and where the remaining walls are.

**Notes to reviewers** (not published)
Everything claimed is reproducible from public material: the four repositories (osmo-operator, qosmo, c54x_exe, grgsm_exe, all under github.com/bbaranoff), the FreeCalypso ROM dump (`ftp://ftp.freecalypso.org/pub/GSM/Calypso/dsp-rom-3606-dump.txt`, not redistributed; the PROM0 used is identical to it word for word), and a binutils `tic54x` disassembler. Each sentence of the project pages is backed by a dated claims file (`c54x_exe/docs/CLAIMS-2026-10-04.md`) marking it VERIFIED / CORRECTED / OPEN with file and line. A one-line installer for the whole lab is at pl4y.store. Honest state: uplink is mixed (SDCCH in ROM, the rest host-side); the C54x core fails 58 of 230 of TI's worked examples; downlink SACCH/8 often fails its FIRE check; two of the DSP unit tests in `dsp_tester` (`nb`, `fb0/fb1`) fail, while the end-to-end bench itself passes 25/25. Demo risk: everything runs on one laptop, no network needed; a recorded run is available as fallback (the linked video). I can do 40 or 60 minutes; 40 + Q&A is the natural length (one demo, two results). First talk at a Congress. English with a French accent; slides and all repositories in English.

**Biography**
I'm an independent telecom engineer in Perpignan, France, about ten years in GSM and radio. Over the past eight months I built, alone, a software lab that runs the TI C54x DSP mask ROM of a Calypso baseband next to OsmocomBB's layer 1 under QEMU, and used it to answer — I think — a fifteen-year-old `FIXME` in the OsmocomBB source. I've taught computer science and networking at university and spent a stint in a radio-security company. My code is on GitHub as bbaranoff; this is my first Congress talk.

**Links**
- https://github.com/bbaranoff/c54x_exe
- https://github.com/bbaranoff/osmo-operator
- https://github.com/bbaranoff/qosmo
- https://github.com/bbaranoff/grgsm_exe
- Proof: https://github.com/bbaranoff/c54x_exe/blob/main/proof_toa_23.md
- Video: https://www.youtube.com/watch?v=-UY50ZZ_mLc

**Form**
Track: Hardware. Length: 40 min + Q&A.

---

Relis la bio une dernière fois (l'enseignement, la boîte), puis envoie.
