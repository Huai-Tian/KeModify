# KePosed

[简体中文](README_ZH.md) | English | [AI collaboration doc](README_AGENT.md)

## 📖 Introduction

**KePosed** is a kernel-level modification framework for Android (ARM64).

It provides cross-process memory read/write, syscall result shaping, and **zero-injection method-level hooks** (shadow-page dual view: original bytes never change, no resident code) — **entirely from kernel space**. The target process receives no resident code, no extra mapping, no resident library. **There is nothing to scan, because nothing is there.**

The name joins two lineages: `Ke` is a **deliberate double meaning** — the abbreviation of **Kernel** (the framework's fundamental identity: kernel-level), and the Windows kernel executive routine family (`KeStackAttachProcess` — attach to a process, manipulate its memory, detach, leave nothing behind); `Posed`, the Xposed philosophy (method-level hooks, scope-based authorization). Same goals as the Xposed family, different methods. **KePosed does not run Xposed modules** — its modules are native binaries against a versioned kernel UAPI, not Java bridge modules; a philosophical descendant, not an API descendant.

User-space hook frameworks (the Xposed family) must place executable code inside the target process. However hard they hide — magic wiping, classloader disguises, maps obfuscation — the payload *exists in memory*, and memory can be scanned. It is an arms race the hider eventually loses. KePosed breaks the loop by refusing to enter the arena:

| | User-space hook frameworks | KePosed |
|---|---|---|
| Payload location | Inside target process | Kernel space only |
| Detectable artifacts | dex/ELF blobs, classloaders, entrypoint hooks, anonymous exec pages | **None resident** — original bytes never change (hash-checks pass); only the data you explicitly modify differs |
| Detection cost | Memory scan (cheap, getting cheaper) | Requires kernel introspection (unavailable to sandboxed apps) |

> **Using an AI assistant (Copilot / Claude / GPT ...) for development on this project?** Read [README_AGENT.md](README_AGENT.md) first — it is written specifically for AI and contains the project's **non-negotiable design decisions, kernel-context red lines, and the Native API taxonomy**. Skipping it and editing code directly is very likely to produce bootloop-grade defects.

## ✨ Features

- **Zero in-process footprint**
  All hooks run in kernel space (a KernelPatch Module). The target process has no framework code, no module code, no new mappings, no anonymous executable pages. In-process scanning — however thorough — finds nothing, because nothing is there.

- **Kernel-only execution, userspace decision logic (the eBPF philosophy)**
  The kernel side exposes narrow, auditable primitives only — no scripting engine, no rule interpreter, no business logic. Decision logic lives in unprivileged userspace modules. A crashing module kills its own process — never the kernel, never the target.

- **Unprivileged binary modules with capability-scoped handles**
  Modules are statically linked binaries running as a **non-root uid** — like Xposed modules are unprivileged apps. Every privileged action must be expressed as a KePosed primitive call through its kernel-issued handle (fd); the kernel enforces each module's registered capability set (mem_read / mem_write / mem_search / syscall_shape ...). A module literally has no path to root that does not pass through the gatekeeper.

- **Native API in the Windows kernel taxonomy**
  The kernel↔module protocol borrows the Windows kernel routine classification, organized by prefix family: `Ke` (execution), `Mm` (memory), `Ps` (process), `Zw` (syscall shaping), `Ob` (module registry). **No SDK — headers and a protocol document only.** The ABI is a stable, versioned contract (ioctl over a kernel-issued handle); modules may be closed-source, because the interface is the only contact surface.

- **Loader-agnostic**
  `keposed.kpm` is a standard KernelPatch Module and works with any conforming loader — [APatch](https://github.com/bmax121/APatch) (native KPM support), [SukiSU-Ultra](https://github.com/SukiSU-Ultra/SukiSU-Ultra) (KPM bridge), or raw [KernelPatch](https://github.com/bmax121/KernelPatch) (`sc_kpm_load` / boot-image embedding). Loading is the loader's business, not the framework's; the framework never depends on any manager app.

- **Modules isolated from each other**
  All modules share one identity on purpose (least privilege), so isolation comes from construction instead: each module is born into its own process-namespace and mount-namespace **cell** — sibling modules simply do not exist in each other's world (no ptrace, no fd theft, no file snooping between modules; the paths are absent, not blocked). The kernel-side rules a module registers can never exceed the capabilities and scope the user granted at install.

- **Complementary to LSPosed — same goal, different methods**
  LSPosed hooks in-process (ecosystem compatibility, the largest module ecosystem); KePosed hooks out-of-process from kernel space (zero footprint, detection-immune). Method-level hooks exist on both paths — resident form there, zero-injection shadow-page form here. The two can run on one device without conflict. The userspace part ships as a KernelSU module whose module directory is deliberately namespaced (manifest deliberately not named `module.prop`) so no manager ever mis-identifies the other's modules.

- **Manager: an APK that is never installed**
  The KePosed manager is an Android application in APK form, but it never goes through system installation — no package record exists. It is launched by the root daemon (keposedd) when needed, in the tradition of LSPosed's uninstalled manager, but by our own means: **launching is not injection**. The manager runs as its own process and never enters anyone else's process — the zero-injection identity is untouched. What we borrow from LSPosed is the idea (an invisible manager) and the distribution/runtime experience; the implementation is our own. Module configuration pages are hosted inside the manager, under the framework's control.

## 📐 Architecture

```
┌───────────────────────────────────────────────────────────┐
│  User space                                                │
│                                                            │
│  keposedd (root daemon)      WebUI       Module binaries   │
│  CONTROL PLANE only:        (HTTP)      UNPRIVILEGED,      │
│  install (mint handle), event-spawn,    any license        │
│  net on/off switch (netns),                                │
│  registry, WebUI config                                    │
└──────────────┬─────────────────────────────▲───────────────┘
               │ control plane               │ DATA PLANE: ioctl on handle
               │ (handle mint & delivery)    │ (daemon NOT in the loop)
┌──────────────▼─────────────────────────────┴───────────────┐
│  Kernel space — keposed.kpm (KernelPatch Module)           │
│                                                            │
│  Gateway     handle auth, per-module capability enforcement│
│  Ke routines  attach / detach, execution primitives        │
│  Mm routines  cross-process read / write / pattern search  │
│  Ps routines  process lookup, spawn/exit event notification│
│  Zw routines  syscall result shaping                       │
│  Ob routines  module registry: handles, caps, revocation   │
└──────────────┬─────────────────────────────────────────────┘
               │ access_process_vm / hook_syscalln
┌──────────────▼─────────────────────────────────────────────┐
│  Target process — no code, no library, no mapping.         │
│  Only its data changes, when a module decides so.          │
└─────────────────────────────────────────────────────────────┘
```

**Data plane vs control plane.** Every hot-path primitive — memory read/write, pattern search, method hooks, watch rules — is issued by the module **directly to the kernel** via ioctl on its handle. The daemon is never in the data path: no added latency, no load, no coupling. It owns the control plane only: minting and delivering handles at install time, spawning modules on kernel events, the per-module network switch, and the persistent module registry. If the daemon dies, running modules and their handles are unaffected — a handle's lifetime is bound to the module process, not the daemon.

**Module networking.** Networking is a module's native capability — **on by default**: the module connects directly, with no middleman. The user can flip it off at any time. The cutoff is physical: the module is **born** into an empty network namespace (no interfaces, no routes, no DNS) — sockets still open normally, but not a single byte ever leaves (the error is `ENETUNREACH`, exactly what a machine with no network returns — no "blocked" fingerprint, nothing to bypass). Pre-stocked sockets are structurally impossible: a process born offline never had a network to hoard from. Flipping the switch back simply respawns the module with normal networking. The kernel-side rules a module registered (watch rules, method hooks) live in the kernel and survive the restart.

## 🧩 Module system (designed, not yet shipped)

Modules are **archive packages** (format undecided) carrying a native binary, installed under the framework's own directory — nested below the framework's KSU module directory, invisible to KSU's top-level scan, and removed together with the framework. The manifest is deliberately **not** named `module.prop`, so KSU/LSPosed managers never mis-identify framework modules. Exact manifest fields (including capability bits) will be finalized in the protocol document — nothing here is an API promise yet.

At design, the daemon registers an installed module with the kernel (which mints a per-module handle — an anonymous fd). When a target process in the module's scope appears, the daemon spawns the module binary: the child is born into its isolation namespaces (and into an empty network namespace if the user has switched networking off), then dropped to an unprivileged uid, with the handle passed via fd inheritance. Every gateway command is checked against the module's capability set **before** any argument parsing; violations return `-EACCES`. Uninstall revokes the handle and kills the process — credentials die instantly.

A module may ship an optional configuration page (built with web front-end technology, in the KernelSU WebUI tradition) — some configuration logic may be closed-source, so the page is authored by the module developer and hosted inside the KePosed manager under the framework's control.

## 📡 Native API (taxonomy decided; names TBD)

The kernel↔module protocol borrows the Windows kernel routine classification, organized by prefix families (command-code ranges already reserved): `Ke` (execution), `Mm` (memory), `Ps` (process), `Zw` (syscall shaping), `Ob` (module registry), `Ldr` (cross-process symbol resolution), plus a reserved family for method hooks. **No SDK — a versioned ABI (headers + protocol document) is the only contract.** The exact routine names and structures are **not final yet** — they will be defined in the protocol document; nothing on this page is an API reference. Modules may be closed-source, because the interface is the only contact surface.

## ⚠️ Honest capability boundary

KePosed is **not** a universal Xposed replacement. What it can and cannot do, stated plainly:

- ✅ **Clean (zero artifacts)** — syscall result shaping (path hiding, environment spoofing) and data-level memory patches (values, fields, property areas). No code enters the target, ever.
- ✅ **Method-level hooks (v1, zero-injection form)** — shadow-page dual view: the original page is never modified (reads and hash-checks always see original bytes); at call time the target thread briefly executes the module's callback via the module's own memory — the image belongs to the module process, is invisible to the target from any angle, and is never resident anywhere in the target. Two equal forms: simple rules (always-return-X, always-skip) are the module's will expressed as data; complex callbacks are the same will expressed as code — the driver executes both faithfully. Covers AOT/JIT/native targets, hot paths included (a few microseconds per call); interpreted-only and inlined methods are documented residual gaps (README_AGENT.md §6.8). LSPosed remains the complementary path — same goal, different methods.
- ✅ **Scope covers native executables too** — a scope entry can be a package name or the path of a directly-run binary; hooking which process is the user's and the module developer's choice.
- 🛡️ **Residual surface** — a target can hash its *own data* to detect data patches (true of every modification tool — no one is immune); but the verification logic's inputs can themselves be shaped at the syscall layer. What KePosed guarantees constructively is the second level: no resident code, no byte changes — there is no "culprit" to find. Defense in depth favors the kernel side. Hardware-level introspection (JTAG, hypervisor-level DMA scanning) is explicitly out of scope.

## 📊 Project Status

**Pre-alpha — design phase.** The repository currently holds the design and the Native API taxonomy; **no code exists yet** — the gateway protocol and primitives are designed (README_AGENT.md is the authoritative, up-to-date record), implementation has not started. **v1 is the endgame** — capabilities either ship in v1 or are cut with the reason recorded; there is no "next version will fix it". Expect the ABI to change until v0.1.0. A kernel-space bug can bootloop the device — always test on a disposable device.

## 🚫 Non-Commercial Statement

This project is initiated by the developer out of personal interest and for technical research purposes, and is **non-commercial** in nature:

- **Permanently Free**:
This project is completely free, with **no paid features, memberships, subscriptions, or in-app purchases**. All features are fully accessible to all users.

- **No Sponsorship Channels**:
The author has **never opened any sponsorship channels**, nor does the author **accept any financial donations** — to maintain the project's neutrality and purity.

- **Non-Profit Purpose**:
This project involves no commercial operations, and the author derives no direct or indirect financial benefit from it.

- **Research-Oriented**:
This project is consistently positioned for **security research, kernel development, and software testing** — providing a research tool for the community, not a commercial product. Any commercial use of this project is the user's own initiative and is unrelated to this project.

- **License is GPL-3.0 only — no commercial exceptions.**
This project is offered under the terms of the GPL-3.0 (see LICENSE),
and every use must comply with that license in full. What GPL requires —
source availability and the same license for derivatives — is exactly what
it means to use this project. Commercial use that cannot accept GPL
terms does not have the author's authorization: the author does not
offer, and will not negotiate, dual licensing, commercial exceptions,
or proprietary redistribution. Reselling this project for profit while
ignoring GPL obligations is copyright infringement.

- **Attribution and statement integrity:**
Redistribution of unmodified builds is permitted only together with
this statement and proper attribution. Removing, altering, or obscuring
this non-commercial statement when redistributing is prohibited.

- **Official channels only:**
Obtain this project only from this repository (GitHub) or other
officially designated channels. Builds from any other source are
unofficial, unverified, and used entirely at the downloader's own risk.

## ⚖️ Disclaimer

- **Purpose Limitation**:
This project is intended for **security research, kernel development, software testing, and educational purposes** only.
Do not use this project for any illegal purposes (including but not limited to cheating, anti-fraud evasion, data theft, and system compromise).

- **Consequences Warning**:
Modifying other processes with this framework **may violate the terms of service of third-party applications and the laws of your jurisdiction**.
You should assess the risks before using it. The developer and contributors **are not responsible for any account bans, legal liabilities, or other consequences** arising from such use.

- **System Stability**:
A kernel-level module operates with the highest privilege on your device. **A bug can bootloop the device or corrupt data** — recovery may require reflashing the boot image. Always test on a disposable device, and keep backups.

- **No Warranty**:
This software is provided under the terms of its license, **without any express or implied warranties**, including but not limited to the warranties of merchantability, fitness for a particular purpose, and non-infringement.

- **Compatibility Disclaimer**:
This software **does not guarantee full compatibility with all Android versions, kernel builds, or loaders**. The developer assumes no responsibility for functional issues or losses caused by system updates, OEM kernel changes, or other uncontrollable factors.

- **Limitation of Liability**:
To the fullest extent permitted by applicable law, **in no event shall the author or contributors be liable** for any direct, indirect, incidental, special, or consequential damages arising out of or in connection with the use or inability to use this software, even if advised of the possibility of such damages.

- **User Responsibility**:
Users assume all legal responsibilities arising from the use of this project.

- **Final Interpretation**:
The final interpretation of this disclaimer belongs to the author of this project.

## 💬 Contact

You are welcome to submit issues, suggestions, and bug reports via GitHub Issues.

## ⭐ Support the Project

If you find this project helpful, or if you recognize its value in technical research, consider giving it a ⭐ on GitHub.

Your support helps more people discover this project, and also lets the author feel the significance of continued maintenance.

Thank you for your recognition.
