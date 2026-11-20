# 案例：Adobe Illustrator 2025 / Photoshop 2025 每次启动都弹窗

环境：macOS 26.5.2（Tahoe），Apple Silicon，Adobe Illustrator 29.3.0、Photoshop 2025（用激活工具安装的版本）。

## 现象

每次打开 Illustrator 或 Photoshop 都弹出：

```text
“Adobe Illustrator 2025” would like to access data from other apps.
[Don't Allow] [Allow]
```

点 Allow 之后，下次启动照旧。

## 诊断过程

### 1. 确认服务名

```bash
log show --last 5m --predicate 'process == "tccd"' | grep -i illustrator | grep -E "PROMPTING|RESULT"
```

```text
AUTHREQ_PROMPTING: msgID=7102.39, service=kTCCServiceSystemPolicyAppData,
  subject=Sub:{com.adobe.illustrator}
AUTHREQ_RESULT: msgID=7102.39, authValue=0, authReason=2, …   ← 用户点了 Don't Allow
```

服务是 `kTCCServiceSystemPolicyAppData`（不是 App Management 的 `…AppBundles`）。
请求方是 `sandboxd`，被检查方是 App 本身。

### 2. 确认“Allow 也记不住”

```bash
log show --predicate 'process == "tccd" AND eventMessage CONTAINS "TCCDEvent"'
```

```text
10:25:34  TCCDEvent: type=Create, service=…AppData, identifier=com.adobe.illustrator   ← Allow 写入了
10:26:37  AUTHREQ_PROMPTING: service=…AppData, subject=com.adobe.illustrator           ← 又问了
```

### 3. 找到失配的记录

```bash
log show --predicate 'process == "tccd"' | grep -A2 "Failed to match existing code requirement"
```

```text
Failed to match existing code requirement for subject com.adobe.illustrator
  and service kTCCServiceSystemPolicyAllFiles
    cdhash H"0ca9e38f…" or cdhash H"163f72f8…"   ← 记录里存的（原厂签名时代）
    cdhash H"db9cc3eb…"                            ← 当前运行的
```

### 4. 确认封条已损坏

```bash
codesign --verify --verbose=1 "/Applications/Adobe Illustrator 2025/Adobe Illustrator.app"
codesign -dvv "/Applications/Adobe Illustrator 2025/Adobe Illustrator.app"
```

```text
…a sealed resource is missing or invalid        ← 资源封条失效（本地化目录被裁掉）
Signature=adhoc
TeamIdentifier=not set                          ← ad-hoc 签名，requirement 只能是 cdhash
```

## 修复

```bash
# 1) 备份 + 只重签外层 bundle，恢复合法封条
sudo ./bin/appdata-prompt-fix reseal "/Applications/Adobe Illustrator 2025/Adobe Illustrator.app"
#   → valid on disk / satisfies its Designated Requirement，cdhash 变为 af1465…

# 2) 清掉绑在旧签名上的记录
./bin/appdata-prompt-fix reset com.adobe.illustrator
./bin/appdata-prompt-fix reset com.adobe.Photoshop

# 3) 重建完全磁盘访问权限记录（Touch ID 一次）
./bin/appdata-prompt-fix fda "/Applications/Adobe Illustrator 2025/Adobe Illustrator.app"
./bin/appdata-prompt-fix fda "/Applications/Adobe Photoshop 2025/Adobe Photoshop 2025.app"

# 4) 退出两个 App，重新打开，验证
./bin/appdata-prompt-fix verify com.adobe.illustrator 3
./bin/appdata-prompt-fix verify com.adobe.Photoshop 3
```

Photoshop 不需要重签（Adobe 的 Developer ID 签名仍然完好），只需要第 2、3 步重建记录。

## 验证

| 时间 | 动作 | `AUTHREQ_PROMPTING` 次数 |
|---|---|---|
| 11:02:59 | 重启 Illustrator | 0 |
| 11:07:57 | 两个 App 同时启动 | 0 / 0 |
| 11:09:09 | 重启 Photoshop | 0 |

复现命令：

```bash
log show --last 5m --predicate 'process == "tccd"' | grep -c "AUTHREQ_PROMPTING.*adobe"
# 0 = 已解决
```

## 踩过的坑

- **先重建记录、后修签名** → 记录写入成功但下次仍失配（顺序必须反）。
- 把 `tccutil reset` 当成万能药 → 它只删记录，不修复签名，单独用无效。
- 想直接写 `TCC.db` → macOS 26 下 root、root LaunchDaemon 都被 FDA 门控挡住（实测 `Operation not permitted` / `authorization denied`）。
- 用 `--deep` 重签带注入 dylib 的 App → 可能把子组件的 entitlements 签坏，外层足够。
- 误以为弹窗是 “App Management” → 那条服务是 `…AppBundles`，文案几乎一样，服务名必须从日志里确认。
