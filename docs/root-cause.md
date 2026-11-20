# 根因分析：为什么 “访问其他 App 的数据” 授权记不住

本文记录 `appdata-prompt-fix` 背后的机制与实测证据。所有日志片段来自 macOS 26.5.2（Tahoe）/ Apple Silicon。

## 1. 弹窗到底是谁发的

弹窗文案固定为：

```text
“<App Name>” would like to access data from other apps.
Keeping app data separate makes it easier to manage your privacy and security.
[Don't Allow] [Allow]
```

统一日志里对应的是：

```text
AUTHREQ_PROMPTING: msgID=7102.126, service=kTCCServiceSystemPolicyAppData,
  subject=Sub:{com.adobe.illustrator}
  Resp:{TCCDProcess: identifier=com.adobe.illustrator, pid=24329, …}
```

注意两个字段：

- `service=kTCCServiceSystemPolicyAppData` —— 这不是 “App Management”（那是 `…AppBundles`），文案却几乎一样，很容易查错方向；
- 请求的发起方是 **`sandboxd`**，被检查的是 App 本身（`accessing=com.adobe.illustrator`）。也就是说，App 自己并没有“跳出沙盒”的能力，是 sandboxd 在它的文件访问路径上做拦截并询问 TCC。

## 2. TCC 记录是怎么匹配的

TCC 表的每条记录包含：

| 字段 | 含义 |
|---|---|
| `service` | `kTCCServiceSystemPolicyAppData` 等 |
| `client` / `client_type` | bundle id（`client_type=0`）或绝对路径 |
| `auth_value` | `0`=拒绝，`1`=未知/待定，`2`=允许 |
| `auth_reason` | 用户点选、系统策略、提示超时等 |
| `csreq` | **code requirement blob**，匹配时用 |

每次访问都要用**当前运行代码**去满足 `csreq`。满足 → 直接放行，不问；不满足 → 走提示流程。

`csreq` 的来源是 App 的 designated requirement：

- 正常 Developer ID 签名的 App：`anchor apple generic and certificate leaf[…6.1.9] … and certificate leaf[subject.OU] = <TEAMID>`（与具体二进制哈希无关，重装小版本仍可匹配）；
- **ad-hoc 签名**（`Signature=adhoc`，破解/重打包最常见）：`cdhash H"<sha1>"`——只认这一个二进制本体。

所以：

```text
原厂签名时代写入的记录  : cdhash H"0ca9e38f…" or cdhash H"163f72f8…"
当前实际运行的代码      : cdhash H"db9cc3eb…"
→ Failed to match existing code requirement
```

这条日志是判定“记录和代码不是同一个身份”的铁证：

```text
Failed to match existing code requirement for subject com.adobe.illustrator
  and service kTCCServiceSystemPolicyAllFiles
    cdhash H"0ca9e38f7b1ffb469c2f728eca42ee28edc1d37e" or cdhash H"163f72f8e8c7e05e32e409d95cee1a4745211d09"
    cdhash H"db9cc3ebe95b92fc52bfefbfb5abdfa2e7f6d977"
```

## 3. 实测：Allow 确实写入了，但下一次仍然弹

时间线（同一台机器，连续复现）：

| 时间 | 事件 |
|---|---|
| 10:21:10 | `AUTHREQ_PROMPTING`（AppData） |
| 10:21:19 | `AUTHREQ_RESULT: authValue=0, authReason=2` —— 用户点了 Don't Allow |
| 10:25:25 | 再次 `AUTHREQ_PROMPTING` |
| 10:25:34 | `TCCDEvent: type=Create, service=…AppData, identifier=com.adobe.illustrator` —— 允许并写入记录 |
| 10:26:37 | **又一次** `AUTHREQ_PROMPTING`（说明 10:25 的记录没被用上） |
| 10:52:24 | 用 `codesign --force --sign -` 修复外层封条 → `valid on disk`，cdhash 变为 `af1465…` |
| 10:52:50 | Allow → 写入新记录 |
| 10:53:22 | **仍然** 弹（说明“有合法签名”还不够，必须让记录与当前身份匹配） |
| 11:07:26 | 重建 Full Disk Access 记录（绑定 `af1465…`） |
| 11:07:57 / 11:09:09 | 两个 App 同时重启：`AUTHREQ_PROMPTING` 计数 **0** |

三个可复用的结论：

1. **App Data 的拒绝不持久**：`authValue=0` 之后还会继续问；
2. **App Data 的允许也可能不生效**：当 App 的代码对象本身不合法（封条损坏 `a sealed resource is missing or invalid`）或记录绑定的是另一份代码时，写入成功 ≠ 匹配成功；
3. **Full Disk Access 是最稳的收口**：新 FDA 记录的 `csreq` 来自**当前**代码，且 FDA 在评估 App Data 时被优先考虑。

## 4. 为什么“损坏的封条”是硬伤

`codesign --verify` 对这类 App 会报：

```text
a sealed resource is missing or invalid
```

常见成因：安装包被裁剪（删掉几十个 `.lproj` 本地化目录）、往 `Contents/` 里塞了第三方 dylib/脚本、或用非标准工具打过补丁。

封条一坏，App 的静态代码对象就不合法。TCC 在新服务（App Data / App Management）上对代码合法性要求更严格，于是即便你能点 Allow，记录的可用性也不稳定。`reseal` 做的事就是：**只重签外层 bundle**，让 `_CodeSignature/CodeResources` 重新覆盖当前文件集合，从而恢复 `valid on disk` 与 `satisfies its Designated Requirement`，同时不动子组件（避免把注入的 dylib、helper 的 entitlements 搞错）。

## 5. 为什么不能直接改 TCC.db

很多人第一反应是 `sqlite3` 直接写 `~/Library/Application Support/com.apple.TCC/TCC.db`。在 macOS 26 上实测全部失败：

```text
$ sudo -s; cp "$DB" /tmp/
cp: …/TCC.db: Operation not permitted          # root 也不行，FDA 门控对 root 同样生效

$ # root LaunchDaemon 方式同样失败
Error: unable to open database "…/TCC.db": authorization denied
```

原因是 FDA 的判定基于**发起访问的进程身份**，root 身份换不来 FDA。唯一“合法写”的入口就是 GUI 里的系统设置（需要本人 Touch ID / 密码），或者由已经拿到 FDA 的进程去写——后者是鸡生蛋问题。

因此本项目的思路不是“绕过”，而是**把 App 的代码身份修成可匹配的，再让系统自己写一条能长久的记录**。

## 6. 排查清单

按顺序检查，基本不会走偏：

1. `codesign --verify --verbose=1 <app>` → 是否 `valid on disk`？是否 `sealed resource is missing or invalid`？
2. `codesign -dvv <app>` → `Signature=adhoc`？`TeamIdentifier` 有没有？
3. `codesign -d -r- <app>` → designated requirement 是 `cdhash` 还是 `anchor apple generic …`？
4. `log show --last 5m --predicate 'process == "tccd"' | grep -E "AUTHREQ_PROMPTING|Failed to match"` → 是否有 `Failed to match existing code requirement`？比对两行 cdhash 是否不同。
5. `System Settings ▸ Privacy & Security ▸ Full Disk Access` 里该 App 是否已在列表、开关是否打开（在列表里但 `csreq` 过期同样无效）。
