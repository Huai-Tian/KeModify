# README_AGENT.md — AI 协作者契约文档

> **本文档的读者是大语言模型 / 编码 Agent,不是人类。** 人类请阅读 [README.md](README.md) / [README_ZH.md](README_ZH.md)。
>
> 你(Agent)即将在一个**内核态修改框架**项目上工作。这不是普通的应用开发:
> 核心载荷是一个 KernelPatch Module(KPM)——它在内核空间运行,持有跨进程内存读写与
> syscall 拦截能力。一个错误的 sleep、一次错误上下文里的分配、一个越界的内存写,
> 都会直接 panic 内核或损坏任意进程的内存——**手机变砖 / 开机循环是真实的失败模式**。
> 同时,本项目的核心价值是一条安全性质:"目标进程内零驻留"——破坏它比崩溃更糟,
> 因为它静默且难以察觉。
>
> 本文档包含三类知识:
> 1. **红线铁律**(§2):惯性思维最想"优化"但绝对不能碰的代码模式,每条附违例后果;
> 2. **架构与目标状态**(§4-§7):快速建立正确的系统心智模型与 Native API 分类学;
> 3. **验证协议**(§8):改完之后如何证明没有破坏系统。

---

## 0. 阅读协议(元指令)

- **权威顺序**:源码注释 > 本文档 > README。若本文档与代码注释冲突,以代码为准,并视为本文档的 bug。
- 本文档**自包含**:不引用任何仓库外文档(上游 KernelPatch 的事实已内联到 §6)。
- §2 的每条铁律都标注了 `[违例后果]`。这些不是代码风格偏好,是设计层面的安全契约。
- **当前项目状态:设计阶段(pre-alpha)。** 仓库尚无代码——本文档描述的是**已拍板的目标架构**。
  诚实现状:代码实现时以本文档为准绳;实现过程中发现的偏差必须回写本文档(经人类维护者确认),
  而不是让代码悄悄偏离。**严禁把计划中的能力写成已实现**(人类 README 与本文档同受此约束)。
- 你无法在本机运行内核测试(需要 ARM64 Android 设备 + KP 加载器)。你能做的验证是:
  静态编译正确性推理、与本文档契约的一致性核对、协议文档(`docs/native-api.md`)与
  UAPI 头文件的同步性校验。真机验证由人类执行(§8)。
- **生成代码时的默认姿态**:本项目的设计决策经常与"通用驱动开发"或"通用 Xposed 模块开发"的
  直觉相反(§3)。当你觉得"这里明显写错了"时,**先假设是你错了**,去 §2/§3 找依据;
  找不到再向人类提出质疑。

---

## 1. 项目本质(30 秒版)

**KeModify = Android ARM64 内核级修改框架。单个 KPM(`kemodify.kpm`)经私有 syscall 通道向外暴露窄原语集(跨进程内存读写/搜索、syscall 结果塑形、进程事件),通道以 128 位 token + 逐模块能力集鉴权;调用方是非特权用户态二进制模块;root 守护进程 `kemodifyd` 管理模块生命周期并代理特权 ioctl。设计目标:目标进程内零代码驻留——只有被明确修改的数据不同。**

核心事实:

| 事实 | 含义 |
|---|---|
| KPM 形态,非 .ko、非 app | 以 KernelPatch Module 运行于内核空间;加载交给宿主加载器 |
| 内核 = 纯执行器 | 无脚本引擎、无规则解释器、无业务逻辑(eBPF 哲学) |
| 模块 = 非特权二进制 | 共享非特权 uid(默认 9000,低于 Android app 的 10000+ 段);身份靠 token |
| 无 SDK | 模块侧契约 = 版本化 syscall ABI(UAPI 头文件 + 协议文档),零链接关系 |
| 加载器无关 | APatch / SukiSU-Ultra(需 KP 补丁 boot)/ 裸 KernelPatch 均可加载;永不依赖任何管理器 app 或 SukiSU 私有 extension ioctl |
| 与 LSPosed 正交 | 方法级进程内 hook 是 LSPosed 的领域;本框架只做进程外的 syscall 塑形与数据操纵 |
| bug = 砖/开机循环 | KPM 没有用户态容错边界;`KPM_EXIT` 清理路径与 `KPM_INIT` 同等重要 |

**名字语义(勿漂移)**:致敬 Windows 内核执行例程族(`Ke*`)。`KeStackAttachProcess`——
附体进程、操纵内存、抽离、不留一物——正是本框架在 Android 上所做的事。
API 分类学因此借用 Windows 内核命名法(§5),这是刻意的身份认同,不是 cosplay 随笔。

---

## 2. 红线铁律(RED LINES — 绝对禁止清单)

每条格式:**规则 / 为什么 / [违例后果]**。编号 RL-xx 供引用。

### A. 零注入纪律(项目存在的理由)

- **RL-01** 永不向目标进程映射任何代码——不注入 dex/ELF、不写 shellcode 进程内执行、
  不改 ArtMethod entrypoint、不远程线程。内核写目标进程内存时,**写入的数据必须是数据**
  (数值、字段、结构体内容),永远不是可执行载荷。
  `[一旦注入,目标进程内出现可扫描的代码痕迹——项目从"无物可扫"退化为"藏猫猫",核心卖点死亡。方法级 hook 需求一律引导到 LSPosed]`

- **RL-02** 目标进程的 maps/fd/threads 列表必须保持原生。不为目标进程创建 fd(不 dup、
  不 SCM_RIGHTS)、不创建线程、不改 VMA 属性。数据修改只在既有页内发生。
  `[新增 fd/线程/VMA 本身就是可检测痕迹,与 RL-01 同级]`

### B. 内核上下文纪律(错误 = panic/砖)

- **RL-03** syscall hook 的 `before`/`after` 回调内**绝不**睡眠/阻塞/获取可能被占的锁/
  `kmalloc(GFP_KERNEL)`。回调运行在 syscall 路径上,被中断的线程可能持有任意锁。
  需要分配的状态一律在 `KPM_INIT` 或命令安装时预分配;确需延迟处理走 workqueue。
  `[死锁/优先级反转/原子上下文睡眠 panic]`

- **RL-04** gateway 分发路径(私有 syscall 的处理函数)与 hook 回调内**绝不**直接
  `printk` 洪水、绝不浮点运算。日志一律限流(同 (op,target) 计数 + 周期采样)。
  `[观测体系自身成为放大器;ARM64 内核态 FP 需显式保存恢复]`

- **RL-05** 跨进程内存写遇到只读目标页时,必须走 `mm/` 内统一的页表遍历路径
  (改 PTE 临时可写 → 写 → 恢复 → flush),该实现只允许存在一份。绝不内联第二份拷贝。
  `[两份实现 = 两处会漂移的页表语义;漏恢复 = 目标进程拿到永久可写 .text = 可检测痕迹 + 数据竞态]`

- **RL-06** 所有 hook(syscall hook、inline hook)**必须**在 `KPM_EXIT` 里对称卸载;
  所有内核分配**必须**对称释放。卸载路径与安装路径同一次提交内完成、同一次评审。
  `[KPM 被卸载后残留 hook = 跳转到已释放内存 = 内核 panic;boot 时加载的 KPM 卸载失败 = 每次开机泄漏]`

### C. 信任模型纪律(模块永远摸不到 root)

- **RL-07** 模块进程**永不**以 root/manager uid 运行。spawn 流程固定:
  校验 manifest → `ObRegisterModule`(内核铸造 token)→ fork → setgroups/setgid/setuid 9000
  → exec,token 经 argv 传递。任何"这个模块比较特殊,给它 root 方便调试"的想法都拒绝——
  调试用 gateway 的调试命令 + daemon 侧日志。
  `[模块拿到 root = 信任模型崩塌 = 恶意/被入侵的模块等于内核本身。这是本框架与"脚本合集"的分界线]`

- **RL-08** token 鉴权与能力检查**必须**在任何参数解析之前完成:先查 token → 再查该模块
  allowed_ops 是否含此 op → 然后才碰用户指针。越权返回 `-EACCES`;`-EPERM` 保留给
  框架级拒绝(如安全模式)。token 比较必须常数时间;token 一次性铸造、永不复用、
  卸载即失效。
  `[先解析参数再鉴权 = 未授权调用者可以触发 copy_from_user 的 bug 面;非常数时间比较 = 时序侧信道]`

- **RL-09** token **绝不**落盘、绝不写日志、绝不经任何世界可读文件传递。
  内核注册表是运行时状态——重启后由 daemon 从磁盘上的**非密**注册信息(uid/能力集/模块 id)
  重放,内核铸造**新** token。
  `[落盘 token = 任何 root 程序可窃取模块身份;这也是为什么持久化属于 daemon 而不属于 KP kstorage]`

### D. 契约纪律(ABI 是唯一接口)

- **RL-10** **不做 SDK。** 模块侧唯一契约是 UAPI 头文件 + `docs/native-api.md`。
  不提供客户端库、不提供 wrapper、不发布示例以外的"官方运行时"。
  `[SDK = 链接关系 = 许可证传染争议 + 版本耦合;syscall ABI 是 arm's-length 边界,Linux 世界 30 年验证过的模式。模块闭源合法性依赖此边界]`

- **RL-11** 命令码**永不重编号**。分配区间:`0x1000+` Ke、`0x2000+` Mm、`0x3000+` Ps、
  `0x4000+` Zw、`0x5000+` Ob。废弃的码保留取值并返回 `-EOPNOTSUPP`;新增只追加。
  任何 struct 布局变更必须升 `KEMODIFY_PROTO_VERSION`(gateway 对不匹配返回 `-EPROTO`)。
  `[无 SDK 保护的生态里,ABI 破坏 = 所有第三方模块同时死亡;协议文档与 UAPI 头文件必须同 commit 变更]`

- **RL-12** 所有 UAPI 结构体:定宽类型(`u32/u64`)、显式 `__attribute__((packed))`(布局敏感处)、
  预留字段、统一以 `u16 proto_ver; u16 cmd;` 开头。错误码是 Linux errno,
  `docs/native-api.md` 的错误表是规范。
  `[跨编译器/跨语言(模块可能是 Rust/C/Go)的结构体布局必须显式钉死]`

### E. 生态隔离纪律(防污染是双向的)

- **RL-13** 框架模块的清单文件**必须**叫 `manifest.toml`,**绝不**叫 `module.prop`;
  模块目录**必须**在 `/data/adb/modules/kemodify/modules/` 之下。
  `[叫 module.prop = KSU/LSPosed manager 把我们的模块误认成自己的模块 = 双向污染;嵌套目录对 KSU 顶层扫描不可见,卸载框架 = 连带清空]`

- **RL-14** 框架**绝不**依赖任何宿主私有接口:SukiSU 的 extension ioctl、APatch 的管理器
  app、任何 `me.bmax.*`/`me.weishu.*` 的包名约定。加载是唯一与宿主的接触点。
  `[依赖宿主私有接口 = 框架被绑死在单一管理器上,与"加载器无关"的定位矛盾]`

### F. 许可证纪律

- **RL-15** 内核与 daemon 源文件 SPDX 为 `GPL-3.0`;`uapi/` 头文件为
  `GPL-3.0 OR Apache-2.0` 双许可;协议文档同样双许可。KernelPatch 上游是
  `GPL-2.0-or-later`(已核实其源码 SPDX),GPL-3.0 调用其导出符号兼容。
  人类 README 的非商业声明与 GPL-3.0-only 立场是 owner 的既定法律姿态,不得弱化。
  `[双许可 UAPI 是闭源模块合法性的支点;单方面收紧/放宽都是生态事故]`

---

## 3. 反直觉设计决策("看起来错"但正确的裁决)

当你产生"这段该重构/修复/补充"的冲动时,先查此表:

| 直觉冲动 | 为什么是错的 | 正确认知 |
|---|---|---|
| "内核侧加个小型脚本引擎/规则解释器,模块就简单了" | 内核=执行器是安全模型本体,不是实现偏好 | 决策逻辑住用户态模块;内核只认命令码(RL 见 §2.A/§1) |
| "给模块 SDK 会更友好" | SDK 制造链接关系 = 许可证传染 + 版本耦合 | ABI + 头文件是唯一契约(RL-10) |
| "token 存 kstorage 持久化更省事" | kstorage 重启即失 + token 落盘=可窃取 | daemon 侧重放注册信息,内核每次铸造新 token(RL-09) |
| "模块共用一个 uid 不如每个模块独立 uid 隔离" | 正确的方向,但 v1 故意不做 | MVP 共享 uid 9000 + token 区分身份;协议预留 per-module uid 演进位,不实现 |
| "hook 每个目标 syscall 一次,粒度细控制准" | per-syscall hook 产生 per-syscall trampoline 时序指纹 | 用 `hook_syscalln` 自动策略:优先全局挂 `invoke_syscall` 单点分发(§6) |
| "detected 进程自哈希数据段 = 框架失败" | 数据修改后自校验确实可发现 | 这是文档化的残余对抗面;校验输入可在 syscall 层反向塑形,纵深在内核侧。诚实标注,不假装零痕 |
| "方法级 hook 也顺便支持一下,扩大生态" | 与 RL-01 直接冲突 | 引导到 LSPosed;两框架攻击面不相交、共存即是答案 |
| "gateway 用 binder/socket 更标准" | 新增 fd/socket = 目标无关但环境可检测的痕迹;且 binder 在 KPM 里不可用 | 冷门 syscall 号 + 参数帧(版本+命令+token)是唯一通道形态 |
| "加载失败自动降级成 .ko 试试" | 框架不碰加载(RL-14) | 加载失败 = 如实报错给用户,让宿主解决 |
| "给 WebUI 加远程访问方便调试" | 攻击面 | WebUI 仅 localhost(KSU WebUI 惯例) |

---

## 4. 架构地图(目标状态)

```
kemodify/
├── README.md / README_ZH.md / README_AGENT.md   # 本三件套
├── LICENSE                                       # GPL-3.0
├── kpm/                        # 内核态:C,KernelPatch Module
│   ├── Makefile                # 对照 kernelpatch/kpms/demo-hello 的构建环境
│   ├── gateway/                # 私有 syscall 通道、token 鉴权、命令分发
│   ├── ke/                     # attach/detach(地址空间切换)、执行层原语
│   ├── mm/                     # 跨进程 read/write/pattern-search(唯一页表写路径在
│   │                           #   这里,RL-05)
│   ├── ps/                     # 进程定位、spawn/exit 事件通知
│   ├── zw/                     # syscall 塑形(构建于 hook_syscalln 之上)
│   ├── ob/                     # 模块注册表:token、能力集、撤销
│   └── include/kemodify/       # 内核内部头
├── daemon/                     # kemodifyd(Rust 优先;root 运行)
│   ├── install/                # 模块 zip 校验、解包、内核注册
│   ├── spawn/                  # fork → drop uid → exec 模块二进制(RL-07 流程)
│   ├── registry/               # 磁盘注册表(非密信息;重启重放用,RL-09)
│   └── webui/                  # localhost HTTP,服务框架与各模块 webroot/
├── uapi/                       # Native API 头文件(双许可,稳定 ABI,RL-12)
│   ├── kemodify.h              # 命令码、结构体布局、协议版本
│   └── capability.h            # 能力位定义
├── module-template/            # 全原语演示模块 = 集成测试载体
├── docs/
│   └── native-api.md           # KeModify Native API Reference(协议规范)
└── (KSU 模块包装:service.sh、customize.sh、metainfo——随发布打包)
```

关键数据流(一屏版):

```
开机: 宿主加载器加载 kemodify.kpm → KPM_INIT(gateway 自挂、能力表初始化)
  → KSU service.sh 拉起 kemodifyd → 从磁盘注册表重放模块注册(内核铸造新 token)
  → 逐个 spawn 模块(非特权 uid + token)

模块调用: 模块二进制 → syscall(gateway_nr, 帧{proto_ver, cmd, token, args})
  → 内核: 验 token → 验能力集(RL-08) → 分发到 ke/mm/ps/zw/ob 例程
  → 例程经 access_process_vm / hook_syscalln 作用于目标进程

syscall 塑形: 目标进程调 openat/read/... → KP invoke_syscall 全局分发
  → Kemodify Zw 层 before/after 回调(受 RL-03 约束)→ 结果塑形 → 返回目标

卸载: 用户在宿主管理器卸载 KSU 模块 → daemon 停止 → ObRevokeToken × N + 杀模块进程
  → KPM_EXIT 对称卸载全部 hook(RL-06)
```

---

## 5. Native API 分类学(协议的身份认同)

分类学整体移植 Windows 内核例程族——这不是装饰,是协议文档的章节结构、命令码的分配
依据、以及生态的命名词汇表。**前缀语义必须用对**(全塞进 Ke 族 = 一眼穿帮的 cosplay):

| 族 | Windows 语义 | KeModify 对应 | 命令码区间 |
|---|---|---|---|
| `Ke` | 内核执行层核心 | attach/detach、通道管理、执行原语 | `0x1000+` |
| `Mm` | 内存管理 | 跨进程读/写/特征码搜索、页属性操作 | `0x2000+` |
| `Ps` | 进程管理 | 进程定位、spawn/exit 事件注册 | `0x3000+` |
| `Zw` | 系统调用层 | syscall 结果塑形的安装/卸载/枚举 | `0x4000+` |
| `Ob` | 对象管理器 | 模块注册表:注册/撤销/能力查询 | `0x5000+` |

命名映射示例(第三列是 Windows 真实 API,语义吻合是选择它的原因):

| KeModify 例程 | Windows 原型 | 语义 |
|---|---|---|
| `KeAttachProcess(pid)` | `KeStackAttachProcess` | 附体目标进程地址空间 |
| `KeDetachProcess()` | `KeUnstackDetachProcess` | 脱离 |
| `MmCopyVirtualMemory(pid, src, dst, len)` | `MmCopyVirtualMemory` | 跨进程内存拷贝 |
| `MmScanVirtualMemory(pid, pattern, mask)` | —(Mm 族自造) | 特征码搜索 |
| `PsLookupProcessByProcessId(pid)` | `PsLookupProcessByProcessId` | 按_pid_定位进程 |
| `PsSetCreateProcessNotifyRoutine(cb)` | `PsSetCreateProcessNotifyRoutine` | spawn/exit 事件 |
| `ZwHookSyscall(nr, match, transform)` | Zw 族(syscall 层) | syscall 塑形 |
| `ObRegisterModule(token, caps)` | Ob 族(对象管理器) | 模块注册 |

协议文档《KeModify Native API Reference》按族分章;Nt/Zw 二象性可作协议彩蛋
(模块侧调用约定 = Nt 视角,内核侧处理 = Zw 视角),不作为功能面。

---

## 6. 内核侧实现约束(KP 运行时事实,已对源码核实)

内核代码运行在 KernelPatch(当前 0.13.9)运行时内。以下是**已核实**的上游事实,
新 Agent 不需要重新考古(来源:KernelPatch 仓库 `doc/en/*.md` 与源码):

- **生命周期宏**:`KPM_NAME("kemodify")`(全内核唯一,控制/卸载的标识符)、
  `KPM_VERSION/LICENSE/AUTHOR/DESCRIPTION`、`KPM_INIT(fn)`、`KPM_CTL0(fn)`(用户态
  控制消息)、`KPM_EXIT(fn)`。init 返回非 0 = 加载失败。
- **syscall hook**:用 `hook_syscalln(nr, narg, before, after, udata)`——自动策略,
  优先在 `invoke_syscall`(所有 syscall 的公共必经点)挂**单个**全局 hook,syscall 表
  零修改、无 per-syscall 时序指纹;`invoke_syscall` 不可用时回退 `el0_svc_common`,
  再回退 per-syscall 的 `fp_hook_syscalln` / `inline_hook_syscalln`。
  **要 `skip_origin`(吞掉原 syscall)必须用 `hook_syscalln_override`**;
  `hook_syscalln` 不认 skip_origin。
- **参数访问**:一律 `syscall_argn(args, n)` / `set_syscall_argn(args, n, val)`,
  绝不裸读 `args->argN`——带 syscall wrapper 的内核首个参数是 `pt_regs *`。
- **用户态访问**:hook 回调内用 `compat_copy_to_user` / `compat_strncpy_from_user`
  (`kputils.h`),不是裸 copy_to_user。
- **inline hook**:`hook_wrap(func, argno, before, after, udata)` 链式(每函数上限 16
  项,插入序,无优先级);同一回调指针重复注册返回 `HOOK_DUPLICATED`。
  卸载用 `hook_unwrap(func, before, after)` 传原指针。
- **进程定位**:`find_get_task_by_vpid(pid)` / `get_task_mm(task)`——KP 在
  `kernel/linux/include/linux/sched/` 提供了 kfunc 封装头,**优先用这些头**;
  不在封装内的内核函数(如 `access_process_vm`)用
  `kallsyms_lookup_name("...")` 取址后按函数指针调用。
- **运行时版本常量**:`kver`(内核版本码)、`kpver`(KP 版本码)可直接引用。
- **热路径纪律**:syscall hook 的 before/after 是热路径——RL-03/RL-04 的来源。

已知边界(上游文档明示):hook 框架 ARM64-only;fp 链上限 32 项;
`HOOK_CHAIN_FULL`(16 项满)是真实错误码,KeModify 自身的 Zw 塑形层要做配额管理,
不能把 KP 的链配额吃光。

---

## 7. Gateway 协议规范(通道层)

- **通道形态**:冷门 syscall 号(具体取值定稿时写入 `docs/native-api.md` 并冻结),
  参数携带帧:`{u16 proto_ver, u16 cmd, u128 token, ...}`。帧里显式带协议版本,
  未来任何变更可被对端感知(RL-11)。
- **鉴权顺序**(实现为不可重排的三步,RL-08):验 token → 验能力位 → 解析参数。
- **特权侧入口**:daemon 经 ioctl(经 KP supercall 或宿主提供的等价机制)持有
  管理通道——注册/撤销模块、查运行态。管理通道同样过 gateway 层,cmd 落 `Ob` 族。
- **能力位**:`uapi/capability.h` 定义(mem_read/mem_write/mem_search/syscall_shape/
  process_events …)。能力粒度到**原语族**,不做参数级能力(过度设计)。
- **错误码约定**:`-EACCES` 模块越权;`-EPERM` 框架级拒绝(安全模式);`-EPROTO`
  协议版本不匹配;`-EOPNOTSUPP` 命令不存在/已废弃;`-ESRCH` 目标进程不存在;
  `-EFAULT` 用户指针无效。语义表是协议规范的一部分,变更需同步文档。

## 8. 验证协议

- **构建**:KPM 在 KernelPatch 检出环境内构建(镜像 `kpms/demo-hello/Makefile`),
  工具链 `aarch64-none-elf-`;产物 `kemodify.kpm`。daemon 用 NDK 静态构建。
- **CI 边界(诚实)**:CI 只能做协议一致性检查——UAPI 头与 `docs/native-api.md`
  的命令码/结构体/错误码三向同步校验(脚本比对),外加 daemon/模块模板的构建。
  **内核代码无法在 CI 验证**;每个触碰 kpm/ 的 PR 必须在 PR 描述里写明:
  测试设备、内核版本、加载器及版本、KPM_EXIT 后设备是否存活。
- **真机判据序列**(每次 kpm/ 改动后全过,人类执行):
  1. 加载:dmesg 横幅版本号与源码 `KEMODIFY_BUILD_TAG` 一致(防旧二进制上机);
  2. gateway:module-template 全原语跑通(attach→read→search→write→detach,
     每步校验读回一致);
  3. 塑形:Zw 层安装 openat 塑形 → 目标进程读路径拿到塑形结果 → 卸载后恢复原生;
  4. 越权:砍掉模块 write 能力位 → write 返回 `-EACCES`(鉴权在参数解析前的证明);
  5. 生命周期:卸载框架 KSU 模块 → 模块进程死、token 失效(重放调用失败)、
     KPM 干净卸载、设备存活。
- **崩溃取证**:KPM panic → 保留 pstore/dmesg,对照 §2 逐条归类;修复必须回写
  判例到本文档(这是本文档的活契约属性)。

## 9. 已知边界与残余臂(文档化接受,勿当 bug 修)

| 项 | 状态 | 说明 |
|---|---|---|
| 数据补丁可被自哈希发现 | 残余对抗面 | 校验输入可在 syscall 层反向塑形;纵深占优但非零痕。README 已如实标注 |
| 方法级逻辑 hook | 明确不做 | RL-01 的推论;引导到 LSPosed |
| 硬件级自省(JTAG/hypervisor DMA) | 明确不管 | 超出软件能力域 |
| 共享 uid 9000 的模块间隔离 | v1 已知弱项 | per-module uid 是 v2 演进位,协议已预留 |
| KP 链配额(16/32 项) | 上游约束 | Zw 塑形层做配额管理,预留半数给其他 KPM 共存 |
| kstorage 不做持久化 | 设计使然 | RL-09 的推论;daemon 侧磁盘注册表是唯一持久层 |

## 10. 术语表

| 术语 | 含义 |
|---|---|
| KPM | KernelPatch Module;ELF 格式内核模块,由 KP 运行时加载执行 |
| Gateway | kemodify.kpm 内的私有 syscall 通道 + 鉴权 + 分发层 |
| 模块(Module) | 扩展 KeModify 的非特权用户态二进制 |
| Token | 内核铸造的 128 位模块凭据;一次性、常数时间比较、卸载即失效 |
| 塑形(Shaping) | 修改目标进程看到的 syscall 结果(返回值/出参缓冲) |
| 附体(Attach) | 内核侧进入目标进程地址上下文执行原语(致敬 KeStackAttachProcess) |
| 能力位(Capability bits) | 模块注册时声明的原语族授权集 |
| 宿主(Host) | 加载 KPM 的方案:APatch / SukiSU-Ultra / 裸 KernelPatch |
| 元模块清单 | `manifest.toml`(刻意避开 module.prop 命名,RL-13) |

## 11. 给接手 Agent 的行为准则

1. **动手前**:读完本文档;对照 §3 确认你要做的事不在"反直觉裁决"里;
   涉及内核代码的,逐条过 §2 的 RL 清单。
2. **写协议时**:任何 UAPI 变更 = 头文件 + `docs/native-api.md` + 协议版本号三处
   同一 commit(RL-11);先想清楚再动——无 SDK 生态里 ABI 错误没有回滚窗口。
3. **写内核代码时**:预分配优先于运行时分配;`KPM_EXIT` 与安装对称;每条 hook 都有
   unhook;页表写只走 `mm/` 的唯一实现(RL-05)。
4. **想加东西时**:先问"这是执行原语还是决策逻辑"——决策逻辑进模块,不进内核;
   再问"这会不会在目标进程留下任何痕迹"——会,则拒绝(RL-01/02)。
5. **想"优化"时**:对照 §3。如果你觉得某条原则"明显更好",大概率它已经在那里被
   推翻过一次;仍确信,带着论据找人类维护者。
6. **诚实性**:没上机验证过的不写"已验证";计划中的能力不写成已实现;
   PR 描述里的测试记录按 §8 判据序列逐项填写。
7. **交付给下一个 Agent 时**:更新本文档的"当前状态"(§0),像 handoff 一样诚实记录
   做了什么、什么没做完、为什么。

---

*本文档随代码演进同步维护。若你(Agent)发现文档与代码冲突:以代码为准,并在你的
修改说明中指出文档偏差。修改 §2 红线、§3 裁决表、§7 协议规范需要人类维护者确认。*
