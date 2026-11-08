# KeModify

[简体中文](README_ZH.md) | English | [AI collaboration doc](README_AGENT.md)

## 📖 Introduction

**KeModify** is a kernel-level modification framework for Android (ARM64).

It provides cross-process memory read/write and syscall result shaping — **entirely from kernel space**. The target process receives no injected code, no extra mapping, no resident library. **There is nothing to scan, because nothing is there.**

The name is a tribute to the Windows kernel executive routine family (`Ke*`): `KeStackAttachProcess` — attach to a process, manipulate its memory, detach, leave nothing behind — is precisely what this framework does on Android.

User-space hook frameworks (the Xposed family) must place executable code inside the target process. However hard they hide — magic wiping, classloader disguises, maps obfuscation — the payload *exists in memory*, and memory can be scanned. It is an arms race the hider eventually loses. KeModify breaks the loop by refusing to enter the arena:

| | User-space hook frameworks | KeModify |
|---|---|---|
| Payload location | Inside target process | Kernel space only |
| Detectable artifacts | dex/ELF blobs, classloaders, entrypoint hooks, anonymous exec pages | **None** — target memory untouched except data you explicitly modify |
| Detection cost | Memory scan (cheap, getting cheaper) | Requires kernel introspection (unavailable to sandboxed apps) |

> **Using an AI assistant (Copilot / Claude / GPT ...) for development on this project?** Read [README_AGENT.md](README_AGENT.md) first — it is written specifically for AI and contains the project's **non-negotiable design decisions, kernel-context red lines, and the Native API taxonomy**. Skipping it and editing code directly is very likely to produce bootloop-grade defects.

## ✨ Features

- **Zero in-process footprint**
  All hooks run in kernel space (a KernelPatch Module). The target process has no framework code, no module code, no new mappings, no anonymous executable pages. In-process scanning — however thorough — finds nothing, because nothing is there.

- **Kernel-only execution, userspace decision logic (the eBPF philosophy)**
  The kernel side exposes narrow, auditable primitives only — no scripting engine, no rule interpreter, no business logic. Decision logic lives in unprivileged userspace modules. A crashing module kills its own process — never the kernel, never the target.

- **Unprivileged binary modules with capability tokens**
  Modules are statically linked binaries running as a **non-root uid** — like Xposed modules are unprivileged apps. Every privileged action must be expressed as a KeModify primitive call through a token-authenticated channel; the kernel enforces each module's registered capability set (mem_read / mem_write / mem_search / syscall_shape ...). A module literally has no path to root that does not pass through the gatekeeper.

- **Native API in the Windows kernel taxonomy**
  The kernel↔module protocol borrows the Windows kernel routine classification, organized by prefix family: `Ke` (execution), `Mm` (memory), `Ps` (process), `Zw` (syscall shaping), `Ob` (module registry). **No SDK — headers and a protocol document only.** The ABI is a stable, versioned syscall contract; modules may be closed-source, because the interface is the only contact surface.

- **Loader-agnostic**
  `kemodify.kpm` is a standard KernelPatch Module and works with any conforming loader — [APatch](https://github.com/bmax121/APatch) (native KPM support), [SukiSU-Ultra](https://github.com/SukiSU-Ultra/SukiSU-Ultra) (KPM bridge), or raw [KernelPatch](https://github.com/bmax121/KernelPatch) (`sc_kpm_load` / boot-image embedding). Loading is the loader's business, not the framework's; the framework never depends on any manager app.

- **Coexists with LSPosed — attack surfaces do not intersect**
  Method-level in-process hooks remain LSPosed's domain; KeModify owns syscall shaping and data-level memory manipulation from outside the process. The two can run on one device without conflict. The userspace part ships as a KernelSU module whose module directory is deliberately namespaced (`manifest.toml`, not `module.prop`) so no manager ever mis-identifies the other's modules.

## 📐 Architecture

```
┌───────────────────────────────────────────────────────────┐
│  User space                                                │
│                                                            │
│  kemodifyd (root daemon)      WebUI       Module binaries  │
│  module lifecycle, config,   (HTTP)      UNPRIVILEGED,    │
│  privilege broker                         any license     │
└──────────────┬─────────────────────────────▲───────────────┘
               │ ioctl (privileged)          │ private syscall channel
               │                             │ (token-authenticated)
┌──────────────▼─────────────────────────────┴───────────────┐
│  Kernel space — kemodify.kpm (KernelPatch Module)          │
│                                                            │
│  Gateway      token auth, per-module capability enforcement│
│  Ke routines  attach / detach, execution primitives        │
│  Mm routines  cross-process read / write / pattern search  │
│  Ps routines  process lookup, spawn/exit event notification│
│  Zw routines  syscall result shaping                       │
│  Ob routines  module registry: tokens, caps, revocation    │
└──────────────┬─────────────────────────────────────────────┘
               │ access_process_vm / hook_syscalln
┌──────────────▼─────────────────────────────────────────────┐
│  Target process — no code, no library, no mapping.         │
│  Only its data changes, when a module decides so.          │
└─────────────────────────────────────────────────────────────┘
```

## 🧩 Module system

Modules are zip packages installed under `/data/adb/modules/kemodify/modules/<module-id>/` — nested below the framework's own KSU module directory, invisible to KSU's top-level scan, and removed together with the framework. The manifest is `manifest.toml` (deliberately **not** `module.prop`), so KSU/LSPosed managers never mis-identify framework modules.

```toml
id = "example"
name = "Example Module"
version = "1.0.0"

[capabilities]
mem_read = true
mem_search = true
mem_write = true
syscall_shape = false
```

At install time the daemon registers the module with the kernel (which mints a 128-bit token), then spawns `bin/<module-id>` dropped to an unprivileged uid, token passed via argv. Every gateway command is checked against the module's capability set **before** any argument parsing; violations return `-EACCES`. Uninstall revokes the token and kills the process — credentials die instantly.

## 📡 Native API

Full protocol reference: [docs/native-api.md](docs/native-api.md) (the *KeModify Native API Reference*).

| Family | Purpose | Representative routines |
|---|---|---|
| `Ke` | execution layer primitives | `KeAttachProcess`, `KeDetachProcess` |
| `Mm` | memory management | `MmCopyVirtualMemory`, `MmScanVirtualMemory` |
| `Ps` | process management | `PsLookupProcessByProcessId`, `PsSetCreateProcessNotifyRoutine` |
| `Zw` | syscall shaping | `ZwHookSyscall`, `ZwUnhookSyscall` |
| `Ob` | module registry & credentials | `ObRegisterModule`, `ObRevokeToken` |

## ⚠️ Honest capability boundary

KeModify is **not** a universal Xposed replacement. What it can and cannot do, stated plainly:

- ✅ **Clean (zero artifacts)** — syscall result shaping (path hiding, environment spoofing) and data-level memory patches (values, fields, property areas). No code enters the target, ever.
- ⚠️ **Out of scope** — method-level logic hooks (arguments, return values, call chains) require in-process execution, which contradicts the core design. Use LSPosed for that; both frameworks coexist.
- 🛡️ **Residual surface** — a target can hash its *own data* to detect data patches; but the verification logic's inputs can themselves be shaped at the syscall layer. Defense in depth favors the kernel side. Hardware-level introspection (JTAG, hypervisor-level DMA scanning) is explicitly out of scope.

## 📊 Project Status

**Pre-alpha — design phase.** The repository currently holds the architecture, the Native API taxonomy, and the licensing groundwork. The gateway protocol and primitive set are being implemented. Expect the ABI to change until v0.1.0. A kernel-space bug can bootloop the device — always test on a disposable device.

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
