# appdata-prompt-fix

> 彻底关掉 macOS 那句烦人的 —— `<App> would like to access data from other apps.`
> 每次启动都弹、点了 Allow 也没用的那种。

[![Platform](https://img.shields.io/badge/platform-macOS%2015%2B-black)](#supported-systems)
[![Shell](https://img.shields.io/badge/shell-bash%203.2%2B-4EAA25)](#supported-systems)
[![License](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

macOS 15 Sequoia 起引入了一个新的 TCC 服务 `kTCCServiceSystemPolicyAppData`（“访问其他 App 的数据”）。只要一个 App 在启动时读了别的 App 目录里的东西（字体、预设、插件、容器数据），系统就会弹一次授权框。

对**官方签名、从未被改动过**的 App，点一次 Allow 就能记住。但对**重新签名 / 打过补丁 / 被裁过资源（.lproj 被删）**的 App，授权永远记不住——因为 TCC 把授权绑定在 App 的**代码要求（code requirement）**上，而那个要求和实际运行的二进制对不上，于是每次启动都重新问。

本工具做三件事：

1. **诊断** —— 把 App 的 cdhash、designated requirement、封条状态、以及最近 TCC 决策从统一日志里捞出来，直接指出是哪一环断了；
2. **重建记录** —— 清掉绑定在旧签名上的失效 TCC 记录；
3. **用 Full Disk Access 收口** —— FDA 是比 App Data 更强的授权，记录会绑定到 App **当前**签名，从此不再询问。

---

## 三十秒用法

```bash
git clone https://github.com/ifryan/macos-appdata-prompt-fix.git
cd macos-appdata-prompt-fix
chmod +x bin/appdata-prompt-fix

# 1) 先看看到底哪一环坏了
./bin/appdata-prompt-fix diagnose com.adobe.illustrator

# 2) 如果封条坏了（"sealed resource is missing or invalid"），先修复签名
sudo ./bin/appdata-prompt-fix reseal "/Applications/Adobe Illustrator 2025/Adobe Illustrator.app"

# 3) 清掉失效记录
./bin/appdata-prompt-fix reset com.adobe.illustrator

# 4) 重新授权（会弹出 Touch ID / 密码，需要你确认一次）
./bin/appdata-prompt-fix fda "/Applications/Adobe Illustrator 2025/Adobe Illustrator.app"

# 5) 退出 App，重新打开，然后验证
./bin/appdata-prompt-fix verify com.adobe.illustrator 3
```

第 5 步输出 `PASS — nothing is asking for permission any more.` 就结束了。

> `fda` 全自动点按钮需要给你的终端 App 开 **辅助功能（Accessibility）** 权限；没给也没关系，它会打印手动四步，30 秒点完。

---

## 原理：为什么点了 Allow 还是不记住

| 层 | 发生了什么 |
|---|---|
| 请求方 | `sandboxd` 代表 App 发起 `TCCAccessRequest(service=kTCCServiceSystemPolicyAppData)` |
| TCC 记录 | 每条记录都带一份 **code requirement**（签名里的 designated requirement；ad-hoc 签名则是 `cdhash H"…"`） |
| 匹配 | 每次请求都拿**当前运行代码**去验证记录里的 requirement |
| 断点 | App 被改过 → 记录的 requirement 是旧签名/原厂签名 → 验证永远失败 → 重新弹出 |

实测日志长这样（`log show --predicate 'process == "tccd"'`）：

```text
Failed to match existing code requirement for subject com.adobe.illustrator
  and service kTCCServiceSystemPolicyAllFiles
    cdhash H"0ca9e38f…" or cdhash H"163f72f8…"     ← 记录里存的（原厂签名时代）
    cdhash H"db9cc3eb…"                              ← 实际运行的
```

也就是说：TCC 里那条授权是“上一辈子”的，和现在这个 App 不是同一个代码身份。此时：

- 点 **Don't Allow** → 记一条 deny，下次照样问（App Data 的拒绝本来就不持久）；
- 点 **Allow** → 会成功写入一条记录（日志里能看到 `TCCDEvent: type=Create`），但下一次启动仍然 `Failed to match`，因为没有封条修复的 App 连“自己稳定地是自己”都做不到。

**修复顺序很关键：先把签名搞成合法、稳定的代码对象，再重建记录。** 顺序颠倒就会出现“授权了但还是弹”。

### 为什么最后用 Full Disk Access

`kTCCServiceSystemPolicyAllFiles`（FDA）是一个更宽的授权，`sandboxd` 在评估 App Data 访问时会先看它。而且**新写入的 FDA 记录绑定的是 App 当前的代码要求**，只要 App 不再被改动就能一直匹配。因此 FDA 是唯一能稳定压掉这个询问的开关。

对比一下几个容易混淆的 TCC 服务：

| 服务 | 弹窗文案 | 管什么 |
|---|---|---|
| `kTCCServiceSystemPolicyAppBundles` | “would like to access data from other apps” 的另一种，常见于 “App Management” | 修改其他 App 的 bundle |
| **`kTCCServiceSystemPolicyAppData`** | **“would like to access data from other apps”** ← 本文主角 | 读取其他 App 的数据/容器 |
| `kTCCServiceSystemPolicyAllFiles` | “完全磁盘访问权限” | 全盘读写，覆盖上面两项 |

---

## 命令参考

```text
appdata-prompt-fix diagnose <app-path|bundle-id>   诊断：身份、封条、最近的 TCC 决策
appdata-prompt-fix reset    <bundle-id> [--all]    清掉失效的 TCC 记录
appdata-prompt-fix reseal   <app-path> [--deep]    修复代码封条（自动备份，可回滚）
appdata-prompt-fix fda      <app-path> [--manual]  加入完全磁盘访问权限
appdata-prompt-fix verify   <bundle-id> [minutes]  统计最近 N 分钟的弹窗次数，PASS/FAIL
```

`reseal` 会先备份 `Contents/_CodeSignature` 和主可执行文件到
`~/Library/Application Support/appdata-prompt-fix/backups/<bundle-id>-<时间戳>/`，
失败自动还原，输出的最后一行给出回滚命令。

---

## 常见问题

**Q：这是不是给 App 开了太大的权限？**
FDA 确实很宽。如果你介意，可以只在“必须用它压弹窗”的 App 上开；也可以用 `reset` 之后**只**点弹窗里的 Allow 试试——如果 App 是原厂签名且未被改动，Allow 本身就够用，本工具就不需要了。

**Q：我的 App 是官方正版，为什么也会反复弹？**
Adobe Illustrator 29.x/30.x 有已知问题：启动时反复请求该权限，macOS 不记住选择（见 [Illustrator UserVoice](https://illustrator.uservoice.com/forums/601447-illustrator-desktop-bugs/suggestions/51432865)）。Adobe 官方给出的绕过方式也是授予完全磁盘访问权限。

**Q：reseal 会不会把 App 弄坏？**
它只重签**外层 bundle**（不递归子组件），保留原 entitlements / flags / runtime，并先做备份。已在 Adobe Illustrator 2025、Photoshop 2025 上验证仍可正常启动。对依赖自身签名做校验的 App（部分商业软件、带 DRM 的），请先自行评估——备份与回滚命令都给了。

**Q：Adobe 更新之后又弹了怎么办？**
更新会替换二进制 → 新 cdhash → 旧 FDA 记录失配。重跑 `reset` → `fda`（必要时 `reseal`）即可。

**Q：能不能完全绕过 TCC 直接写数据库？**
macOS 26 上不行：`~/Library/Application Support/com.apple.TCC/TCC.db` 受 FDA 自身门控，root 进程、root LaunchDaemon 都读不到（本项目实测过）。所以正道只有本工具这条：修好签名 → 重建记录。

---

## English quickstart

macOS 15+ asks *"<App> would like to access data from other apps."* on every launch when an app is re-signed, patched, or shipped with a broken code seal. TCC stores each grant against the app's **code requirement**; when the running binary no longer satisfies that requirement, the grant can never match again, so the prompt returns forever.

```bash
git clone https://github.com/ifryan/macos-appdata-prompt-fix.git
cd macos-appdata-prompt-fix && chmod +x bin/appdata-prompt-fix

./bin/appdata-prompt-fix diagnose com.adobe.illustrator
sudo ./bin/appdata-prompt-fix reseal "/Applications/Adobe Illustrator 2025/Adobe Illustrator.app"  # only if the seal is broken
./bin/appdata-prompt-fix reset com.adobe.illustrator
./bin/appdata-prompt-fix fda  "/Applications/Adobe Illustrator 2025/Adobe Illustrator.app"
# relaunch the app
./bin/appdata-prompt-fix verify com.adobe.illustrator 3
```

Full Disk Access supersedes the App Data gate, and a freshly written FDA record is bound to the app's *current* requirement — which is why this is the only reliable way to silence the prompt for a binary that no longer matches its original signature.

---

## 支持范围

- macOS **15 Sequoia / 26 Tahoe**（`kTCCServiceSystemPolicyAppData` 从 15 开始存在）
- bash 3.2+（macOS 自带版本即可），无第三方依赖
- Intel 与 Apple Silicon 均可

只处理你**自己机器上**的本地 App；需要管理员密码与一次 Touch ID，这些只能由本人完成，脚本不会、也无法代替。

## License

[MIT](LICENSE)
