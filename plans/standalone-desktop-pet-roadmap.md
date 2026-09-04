# newlai_pet 独立桌宠施工蓝图

计划版本：`1.0`
状态：`reviewed`
日期：2026-09-04
计划负责人：仓库维护者 `g5n-dev`
目标：用一套可测试的核心代码，持续交付 Windows 与 macOS 原生桌宠；不依赖 Codex，运行时默认不联网。

## 1. 产品与平台承诺

| 平台 | 首发范围 | 安装产物 | 最低目标 |
|---|---|---|---|
| Windows | x64 | NSIS `Setup.exe`，附加 WiX `.msi` | Windows 10 22H2、Windows 11；首次安装在缺少 WebView2 时允许安装器下载 bootstrapper，安装后可离线运行 |
| macOS | Apple Silicon、Intel | `.app.tar.gz`、`.dmg` | macOS 12+；GitHub 分发，不进入 Mac App Store |

应用身份在 G0a 冻结，并由 S1 写入工程：

- 产品名：`NewLai Pet`
- 可执行文件名：`newlai-pet`
- bundle identifier：`io.github.g5ndev.newlaipet`
- 数据目录 schema：`newlai-pet/v1`
- 主应用 tag：严格使用 `desktop-vMAJOR.MINOR.PATCH[-prerelease]`
- 独立修复器 tag：保留 `repair-vMAJOR.MINOR.PATCH[-prerelease]` 命名空间，不与主应用版本混用

G0a 同时冻结许可分层：新写的 `desktop/` 程序代码使用独立代码许可证与版权声明；角色图集、图标、预览和营销图只按 rights manifest 的素材授权分发，不能因代码开源而被自动再许可。S13 的 SBOM/THIRD-PARTY-NOTICES 覆盖依赖，S14 Release 同时携带代码许可证、素材许可与署名。

“独立”验收含义：全新用户目录、未安装 Codex、断网运行时均可启动；应用不读取 `~/.codex`，不调用 Codex，也不上传、记录或持久化全局指针坐标。Windows 旧机器仅在缺少 WebView2 的安装阶段可能需要网络；后续另评估体积较大的 offline installer。

“跟随鼠标”的产品语义固定为：角色用 16 向视线/朝向响应全局指针，并可因拖拽播放左右跑动；桌宠窗口不会自动追着光标跑，只有用户拖拽才改变位置。这样保留陪伴感，不抢鼠标、不遮挡正在点击的目标。

首版不承诺跨所有虚拟桌面显示：macOS/Windows 均只保证当前工作区可用。`visibleOnAllWorkspaces` 不作为跨平台契约。

## 2. 硬门禁与全局不变量

### G0a/G0b — 产品身份与独立分发权利批准（fail-closed）

当前根目录 `spritesheet.webp`、`assets/`、`previews/` 的上游许可只明确覆盖个人、非商业 **Codex pet** 使用。它们不得因为名称、格式或路径变化而进入独立 EXE/MSI/APP/DMG。

G0a 必须先于 S1：选择并冻结产品路线、产品名、角色名和 bundle identifier。路线 A 使用获准的上游衍生角色；路线 B 使用经过非衍生审查的全新角色。若路线 B 需要更名，必须在 S1 之前完成，不能在 alpha 后迁移身份。

G0b 的证据收集可从 G0a 后立即进行；仓库 PR 要等 G10 建立保护后再提交，并可与 S2–S11 通用运行时开发并行，但必须先于 S12。G0b 不是可由普通 PR 自证的布尔值。通过条件必须同时满足：

1. 路线 A：`sadlywo` 的可核验书面授权明确允许修改并以免费、非商业独立 Windows/macOS 桌宠分发；或路线 B：产品负责人批准改用权利完全自有、经过非衍生审查的新角色。
2. 路线 A 还必须证明当前衍生图集每次修改/生成的作者授权、工具条款、输入来源和输出哈希；上游许可不能替代对新增衍生字节的来源证明。路线 B 对全部视觉字节提供同等级 clean-room chain。
3. 独立权利审阅者核对授权原件、衍生来源或全新素材来源链；实现者不能自批。
4. `rights-review.json` 保存独立审阅结论、被审阅证据摘要与审阅者身份；实现者、素材作者和该次发布触发者不得担任此独立审阅者。
5. 仓库保存机器可读清单；授权原件可私下保存，但必须保存不可变摘要与审阅者可核验位置。该 PR 由 branch rules 强制要求 rights CODEOWNER 批准，普通字段修改不能绕过。

`docs/desktop/rights/rights-manifest.json` 最小结构：

```json
{
  "schemaVersion": 1,
  "productIdentity": "authorized-upstream",
  "evidence": [
    {
      "id": "stable evidence id",
      "type": "upstream-permission | derivative-provenance | clean-room-provenance",
      "sha256": "64 hex",
      "reviewLocator": "reviewer-resolvable locator",
      "reviewedBy": "independent reviewer",
      "reviewedAt": "ISO-8601"
    }
  ],
  "grants": {
    "platforms": ["windows", "macos"],
    "media": ["exe", "msi", "app", "app.tar.gz", "dmg"],
    "modification": true,
    "commercial": false
  },
  "assets": [
    {
      "path": "desktop/assets/pet/example.webp",
      "sha256": "64 hex",
      "purpose": "atlas | app-icon | tray-icon | dmg-background | screenshot | preview",
      "licenseRefs": ["one or more stable evidence ids"]
    }
  ]
}
```

`productIdentity` 是严格 enum：`authorized-upstream` 或 `clean-room-original`，不是包含分隔符的自由文本。授权 grants 必须同时覆盖产品名/角色名；路线 B 另附非衍生审查摘要。

门禁分三种模式，禁止混用：

- **`pre-approval`**：S1–S11 期间，`desktop/assets/pet/` 不得含生产视觉文件；只能使用 Canvas 程序化占位，不从根目录动态读取素材。
- **`evidence-approved`**：G0b 只验证外部证据、授权范围、独立审阅信息和“预期资产哈希清单”；允许这些生产字节尚未进入仓库，因此不与 S12 成环。
- **`source-assets-present` / `bundle-inventory`**：S12 导入后验证源资产 exact allowlist；S13/S14 对解包后的应用再次枚举 Vite/Tauri 全部资源。验证器不只看扩展名：递归解包受支持容器并以 magic bytes/MIME、SVG/XML、CSS `url()`、JS/HTML data/blob URL 和 bundle resource table 识别视觉/音视频/字体资源；拒绝 symlink、目录外路径、构建时下载、重命名伪装、内嵌字节和任何清单外资源，实际 path/purpose/hash 集合必须完全一致。

`CODEOWNERS` 与 branch rules 必须覆盖：全部 `validation/desktop/**` 证据、`update-channels/**`、rights manifest/review/verifier、`docs/desktop/{signing,keys,adr}/**`、Environment/ruleset policy、`desktop/assets/`、`repair/assets/`、应用/修复器图标、预览/营销图、Tauri bundle 配置及所有 build/release/repair workflows。Environment 审批只对一次具体 deployment job 有效，不作为 G0b 的持久状态；workflow 引用一个不存在的 Environment 会自动创建无保护环境，也不算配置成功。

G14 必须预先用 API 创建并回读 `desktop-rights-approved`、`desktop-release-approved`、`desktop-windows-signing`、`desktop-macos-signing`、`desktop-updater-signing-k1`，并在 GitHub UI 对每个环境关闭 admin bypass 后执行负向演练。每个环境均要求命名 reviewer、`prevent_self_review=true`、禁止 admin bypass，deployment policy **只允许 protected `main`**；release tag 是最终 publish job 的输出，不是读取 Environment secrets 的执行 ref。rights 与 release reviewer 必须不同。`rights_gate`、`release_gate` 串行执行；`sign_windows`、`sign_macos` 分别直接绑定自己的签名 Environment，OS 签名授权只对对应 job 可见：Windows 私钥留在 K14os 批准的 HSM/远程签名服务中，macOS 凭据只在 macOS Environment；repo/org scope 不保存私钥。K15a 完成后 `updater_sign`/`channel_sign_k1` 才可直接绑定 K1 updater Environment。后续验证、证明和发布 job 显式消费 gate 与 artifact outputs。单个 Environment 多 reviewer 不能被当成“双人同时批准”。

若选择路线 B，G0a 必须先冻结新角色名称、视觉差异、参考来源、生成工具条款和运行时资产交付合同；这会 supersede 任何使用旧产品身份的在途 S1 分支。纯 TS 核心不得预先绑定当前牛来的尺寸或行列。

### 运行时不变量

1. `desktop/src/core/**` 不导入 DOM、Node、Tauri/Rust，不直接读时钟、随机数、文件或网络。
2. 核心使用版本化、资产无关的 `RuntimeManifest`；当前 8×11 牛来契约只在获准素材集成步骤写入。
3. 动画以单调时间 `now - enteredAt` 求帧，不用 `setInterval` 累加。
4. Renderer 只裁切清单指定格，不旋转、插值或重绘 atlas。
5. 透明穿透启动时永远为关闭状态，且不持久化。
6. 自定义 placement 是窗口位置唯一真源；不安装 `tauri-plugin-window-state`。
7. 生产 CSP 默认 `default-src 'self'; connect-src 'none'; img-src 'self' asset:; script-src 'self'; style-src 'self'`，禁止远程导航，关闭生产 devtools。
8. 指针采样只在内存中使用；日志、store、crash report 与网络请求都不得包含原始坐标。
9. 所有 Actions 以完整 commit SHA 固定，构建使用 `npm ci` 与 `cargo --locked`。
10. 已公开 Release 视为不可变；故障通过标记撤回并发布更高版本 forward-fix，不覆盖 tag/asset。

## 3. 核心协议

### 3.1 资产无关运行时清单

```ts
interface RuntimeManifestV1 {
  schemaVersion: 1;
  atlas: {
    path: string;
    sha256: string;
    width: number;
    height: number;
    columns: number;
    rows: number;
    cellWidth: number;
    cellHeight: number;
  };
  neutralFrame: { row: number; column: number };
  clips: Record<string, {
    row: number;
    columns: readonly number[];
    durationsMs: readonly number[];
    mode: "loop" | "once" | "hold";
  }>;
  look: {
    clockwiseFromUp: true;
    stepDegrees: number;
    frames: readonly { row: number; column: number }[];
  };
  productProfile: {
    requiredClips: readonly [
      "idle", "running-right", "running-left", "waving", "jumping",
      "failed", "waiting", "running", "review"
    ];
    requiredLookFrameCount: 16;
    requiredLookStepDegrees: 22.5;
  };
}
```

`columns.length === durationsMs.length` 是唯一帧数真相。`product-profile` validator 必须拒绝缺失的九个必需 clip、非 16 个 look frame 或非 22.5° 步长；本产品不使用静默 fallback。fixture 必须覆盖非 8×11、非 192×208 图集，证明核心不绑定当前资产。

### 3.2 指针采样协议

Rust 最多 30Hz 发送采样，但不得按“方向未变”做语义去重：

```ts
interface PointerSampleV1 {
  schemaVersion: 1;
  sampledAtMonotonicMs: number;
  sampleSequence: number;
  motionSequence: number;
  coordinateSpace: "global-physical-px-v1";
  cursor: { x: number; y: number };
  cursorMonitorId: string;
  cursorScaleFactor: number;
  windowGazeAnchor: { x: number; y: number };
  windowMonitorId: string;
  windowScaleFactor: number;
}
```

- 任一 cursor、anchor、monitor 或 scale 变化都产生新采样；静止时仍发低频 heartbeat，使 1000ms 停止边界可判定。
- `motionSequence` 从“上一次 accepted physical point”累计位移；连续多个亚阈值移动累积达到等价 1.5 DIP 后必须增加，不能逐样本丢弃。cursor 所在屏的 `cursorMonitorId/cursorScaleFactor` 随 payload 提供；`sampleSequence` 每个采样增加。
- 方向、距离、24/36 DIP neutral 滞回、3° 扇区滞回和停止判断全部留在纯 TS 核心。
- 只有 renderer 可以去重相同最终视觉帧；transport 不得丢掉同扇区持续移动或窗口移动产生的采样。

坐标类型必须显式区分：

- `GlobalPhysicalPx`：全局鼠标与窗口 gaze anchor；凝视向量只在这个连续空间计算。
- `MonitorLocalDip`：窗口尺寸、work area、边距与 placement；使用目标 monitor 的 scale 转换。

禁止构造跨混合 DPI 显示器的“全局 DIP 平面”。Tauri 已返回桌面左上原点坐标，不再做第二次 macOS 原点翻转。

角度定义：

```ts
angle = normalize360(atan2(dx, -dy) * 180 / Math.PI)
index = floor((angle + stepDegrees / 2) / stepDegrees) % frameCount
```

0°=上，90°=屏幕右，180°=下，270°=屏幕左；恰好位于半格边界时进入顺时针下一格。

### 3.3 状态与交互协议

视觉优先级：

```text
drag > failed > reaction > semantic > look > ambient > idle
```

- 单击：`waving` once。
- 双击：`jumping` once，并抢占尚未完成的 wave。
- 拖拽阈值：累计移动 `>=6 DIP`；拖拽后不再触发 click。
- 语义状态通过托盘的开发/用户动作入口可达：`running`、`waiting`、`review`、`failed`；未来外部事件仍走同一 typed port。
- 单次动作完成携带实例 token，旧 token 不能结束新动作；结束后恢复被抢占的下层状态。
- 鼠标停止 1000ms 后回 idle；距离 ≤24 DIP 进入 neutral，距离 ≥36 DIP 才退出。
- 环境动作仅在 8–20 秒无交互后触发，RNG 必须注入；reduced motion 时禁用环境动作并固定稳定帧。

### 3.4 Placement 与生命周期协议

保存 `{displayFingerprint, normalizedX, normalizedY, edgeMarginDip}`：

```text
u = (x - workArea.x) / max(1, workArea.width - pet.width)
v = (y - workArea.y) / max(1, workArea.height - pet.height)
```

持久化记录分开保存稳定身份与可变几何：`displayDeviceId` 优先使用平台持久 ID；另存 `lastMonitorPhysicalBounds`、`lastMonitorWorkAreaDip`、`lastScaleFactor`、`lastWindowPhysicalRect` 与 topology snapshot。不得把会随 Dock/任务栏、DPI 或分辨率变化的 work area/scale 混入稳定 ID。设备 ID 匹配失败时，使用已保存的旧 bounds/window rect执行最大矩形交集、最近屏、主屏回退；覆盖同型号显示器交换、负坐标、上下排列和混合 DPI。

启动顺序唯一：

```text
single-instance first → migrate/sanitize settings → enumerate monitors
→ reconstruct/clamp position → create tray → register recovery shortcut
→ position hidden window → show
```

第二实例必须执行：show、unminimize、focus、关闭穿透、重新 clamp。另提供 `--reset-interaction` 安全入口。

只有托盘创建与 `CommandOrControl+Shift+P` 注册都成功后，才允许启用穿透；任一失败则拒绝启用并提示。快捷键冲突时允许用户从托盘选择替代组合，但在成功注册前仍不可穿透。

### 3.5 性能预算

参考机：Apple M1/8GB/macOS 12+ 与 4 核/8GB Windows 11 x64；预热 5 分钟、采样 30 分钟。

- idle 平均 CPU <2%，p95 <5%。
- 活跃凝视平均 CPU <4%，p95 <8%。
- 30 分钟 RSS 增量 <10 MiB，增长斜率 <1 MiB/10min。
- pointer IPC ≤30/s；静止 heartbeat 可计入但不得超过该上限。
- 5 分钟窗口内 renderer >50ms long task 为 0（启动阶段除外）。

阈值需要两平台证据；超出时阻止公共 beta，不能写“可接受”代替数字。

### 3.6 更新渠道与传输合同

客户端不使用 GitHub API/Token，也不依赖自建可变服务器。K1 客户端只读 `https://raw.githubusercontent.com/g5n-dev/newlai_pet/main/update-channels/k1/{canary|beta|stable}.json`，bridge/K2 客户端只读同路径的 `k2` 命名空间；每个 channel 是一个完整 JSON 文件，因此一次受保护 main commit 就是原子切换点。raw CDN 短暂返回旧提交只会延迟升级；validator 接受仍有效的旧 generation，但拒绝倒退、过期或同 generation 换内容。

每份 manifest 是锁定插件版本所支持的 **Tauri static JSON 严格超集**，使用 RFC 8785 JSON Canonicalization Scheme（JCS）；签名器、Rust 和 TS 用同一组官方/边界 test vectors，size/generation 限于非负安全整数。顶层同时包含 Tauri 要求的单一 SemVer `version`、`notes`、RFC 3339 `pub_date` 与 `platforms`，以及 `schemaVersion`、`keyId`、`channel`、`status: "update" | "noUpdate"`、严格递增 `generation`、`issuedAt/notBefore/expiresAt`、`artifactSetDigest`、`releaseArtifactSetDigest`、`attestationId` 与 `manifestSignature`。`platforms` 精确包含 `windows-x86_64`、`darwin-aarch64`、`darwin-x86_64`；每项用 Tauri 原生字段 `url` 与 `signature` 承载不可变 GitHub Release asset URL 和 detached artifact signature，并追加 `assetId`、`fileName`、`size`、`sha256` 与 `osSigningIdentity`。Win10 22H2/Win11 是同一 Windows target 的两类测试主机。

两个摘要不可混用。`artifactSetDigest = SHA256(JCS(按 platform key 排序的 [platformKey, fileName, size, sha256]))` 只覆盖 updater 实际安装的三个 OS 签名 artifact：Windows NSIS `Setup.exe` 与两种架构的 macOS `.app.tar.gz`；MSI、DMG 和 detached `.sig` 不属于 updater target。每次 promotion 另生成 canonical `release-manifest-v1.json`，以 `releaseArtifactSetDigest = SHA256(JCS(按 [role,target,fileName] 排序的 [role,target,fileName,size,sha256]))` 覆盖该 Release **全部公开 payload assets**：main profile 包括 NSIS、MSI、两份 `.app.tar.gz`、两份 DMG、该合同版本启用的全部 `.sig`、SBOM、THIRD-PARTY-NOTICES、代码/素材许可与署名；repair profile 包括 Windows x64 ZIP、macOS universal DMG、SBOM 与许可说明。为避免自引用，只有 `release-manifest-v1.json` 本身和 GitHub attestation envelope 不进入该集合；attestation 必须同时绑定 manifest 文件 SHA-256 与 `releaseArtifactSetDigest`，publish 拒绝 manifest 列表之外的任意 asset。两个集合都不依赖尚未创建的 Release asset ID；发布后按 manifest 对每个 asset ID/URL/name/size/hash 做 exact readback。全文其余“final digest”均指 `releaseArtifactSetDigest`，单个产物字节仍以各自 SHA-256 校验。

`noUpdate` 仍携带一套字段完整、可由 Tauri 解析的 previous-good `platforms`，但自定义 policy 永不下载。`pub_date` 必须等于 `issuedAt`；`manifestSignature` 由对应 K1/K2 Environment 对除自身字段外的 canonical bytes 签名。客户端先验证 manifest signature/channel/key/time，再验证 target/arch、SemVer、hash/length 与选中 target metadata，并从三个 `platforms` entry **独立重算** `artifactSetDigest`。客户端只检查 `releaseArtifactSetDigest`/`attestationId` 的格式、确认二者被 `manifestSignature` 覆盖，并把它们作为 channel signer 认证的 release-lineage 引用持久化；由于单次 channel JSON 不含 MSI/DMG/SBOM/许可的完整 entry 或 attestation envelope，客户端不声称独立复算完整 Release 摘要或验证证明内容。该复算由 promotion、channel signer 与无密钥 channel verifier 完成。同 generation 仅允许逐字节相同的 manifest。`status=update` 要求 `version` 严格高于已安装版本及该 key/channel 的 version high-water mark；`status=noUpdate` 的 `version` 是不低于既有 high-water mark 的停发上界，previous-good targets 只为满足 Tauri 静态 schema，绝不产生下载/install effect，恢复发布必须使用更高 SemVer。全部通过后才按 key/channel 持久化最后接受的 generation/version/两个 digest，并仅在 `status=update` 时调用 Tauri 的 artifact signature 验证与下载；`status=noUpdate` 只提交新的 generation/high-water mark。Release asset/tag 已由 S14 设为 immutable，所以能控制 raw 文件但拿不到 K1/K2 与 OS signing authority 的主体最多造成 freeze/replay，不能伪造 generation、跨 channel 提前晋级或注入新二进制；rollback policy 对任何 `update` 仍拒绝低版本。

channel 变更必须来自固定 `desktop-channel.yml`，顺序为 `desktop-release-approved` 决策 gate → 只绑定对应 K1/K2 Environment 的 `channel_sign` → 无密钥 verifier。`channel_sign` 不接受任意文件/URL/payload，只能从已验证 rollout decision、immutable Release asset IDs 和 previous generation 构造固定 canonical manifest；它只新增 manifest signature，绝不重签 artifact。受保护 deployment 完成后，操作者提交 `desktop/<id>-channel-evidence` PR，只能把该步骤唯一拥有的一个 `update-channels/...json` 与自己的 `validation/desktop/<ID>/**` 改成 candidate 的逐字节内容；PR verifier 从 deployment/run ID 重取 candidate、Release assets 和 rollout decision 复算，独立 release CODEOWNER 批准后合入。workflow 不直接 push main；禁止一次 PR 同时改两个阶段/两把 key 的 channel。紧急停发同样经过 release→key approval 与受保护 PR，发布 **更高 generation**、正确 key/channel 签名且携带 previous-good `platforms` 的 `noUpdate`；不得发布低版本 `update`、恢复成客户端已见过的低 generation，或删除 Release。

所有请求由窄 Rust updater adapter 发起，前端 CSP 继续 `connect-src 'none'`。默认 `updaterEnabled=false`；只有用户 opt-in 后才访问 `raw.githubusercontent.com`，以及 S15a1 冻结且无 wildcard 的 Release 下载/redirect host allowlist（初始为 `github.com`、`objects.githubusercontent.com`、`release-assets.githubusercontent.com`）；任一重定向出 allowlist 即拒绝。请求不发送设备 ID、指针、日志或使用统计。超时、TLS/JSON/签名失败均 fail closed，且不影响离线启动。

## 4. 用户旅程

1. 未安装 Codex 的新用户安装并离线启动，透明桌宠出现在当前 work area 内。
2. 鼠标移动时按 16 向凝视；同一扇区跨 neutral 阈值仍更新；停住 1000ms 恢复 idle。
3. 拖拽时使用左右移动动画，松手后 clamp 并只写一次位置。
4. 单击 wave、双击 jump；托盘可触发 running/waiting/review/failed，并能恢复下层状态。
5. 托盘可控制显示、置顶、凝视、穿透、scale、opacity、reduced motion、开机启动、重置与退出。
6. 穿透下托盘、已注册全局快捷键、第二实例与 `--reset-interaction` 均能恢复。
7. 睡眠唤醒、缩放变化、显示器拔插和负坐标多屏后窗口仍可见，不补播历史帧。
8. 平台能力失败时安全降级：无托盘/快捷键则禁止穿透；无全局鼠标则 idle 仍可用；损坏设置回默认。

### 4.1 双平台迭代节奏

每一轮都是 Windows + macOS 的纵向切片，不接受“先把 Mac 做完、最后再移植 Windows”。涉及原生行为的 PR 必须同时有两平台契约测试；到达相应 gate 时再补真实系统证据。

| 迭代 | 对应步骤 | 用户可见增量 | 分发边界 |
|---|---|---|---|
| Engineering 0 | S1–S5 | 透明置顶窗口、占位角色、托盘与退出 | 仅 CI smoke，不发布 |
| Alpha 1 | S6–S7 | 16 向凝视、拖拽、单/双击、可恢复穿透 | 程序化占位，仅内部测试 artifact |
| Alpha 2 | S8–S11 | 设置、多屏/DPI、autostart、性能收敛 | 程序化占位，仅内部测试 artifact |
| Alpha 3 | G0b + S12–S13 | 获准角色与完整 Win/mac 安装体验 | Actions artifact，不公开 |
| Beta | K14os + G14 + K14v + S14 | Windows/macOS 正式签名、公证的公开 prerelease | GitHub immutable Release |
| RC/1.0 | S15a1–S15c2 + G15/G15b | opt-in updater、crash-loop 自救、外部修复器 | 同一字节依次通过 canary→beta→stable 门禁 |

任一平台在该轮失败，该轮整体不升级版本；平台特有实现留在 adapter，核心动作与状态语义保持一致。

## 5. 唯一依赖表

此表是依赖关系唯一真源；正文不得另写冲突依赖。

| ID | 硬前置 | 消费产物 | Owned paths | 禁止修改 | 可并行组 |
|---|---|---|---|---|---|
| G0a | 无 | 上游许可现状、产品决策 | `docs/desktop/product-decision.md` | 代码、素材字节 | 首个串行门禁 |
| G0b | G0a,G10 | 固定产品路线、外部证据/clean-room 来源 | `docs/desktop/rights/**`, `scripts/verify-desktop-rights.mjs` | CODEOWNERS、生产 bundle、根素材 | R（可与 S2–S11 并行） |
| S1 | G0a | 冻结产品身份、本计划 | `desktop/package*.json`, `desktop/*config*`, `desktop/index.html`, `desktop/src/main.ts`, `desktop/src-tauri/{Cargo*,build.rs,tauri.conf.json}`, `desktop/src-tauri/src/{lib.rs,main.rs}`, 基础 capability/目录 | 根宠物包 | 串行 |
| S10a | S1 | 稳定 scripts/lockfiles | `.github/workflows/{ci,native-webview,native-os}.yml`、`.github/{CODEOWNERS,branch-protection-main.json,pull_request_template.md}`、`.github/checklists/{_schema,_templates}/**`、`docs/desktop/{review-roles,test-hosts}/**`、`scripts/native-test/{engine,drivers,probes,schemas,controller}/**`、`scripts/verify-branch-protection/**`、`validation/desktop/S10a/**` | feature scenarios/checklists、package/lock/Cargo/Tauri config | CI 代码步骤 |
| G10 | S10a | 实际出现的 check contexts | `validation/desktop/G10/**`；GitHub main protection 外部状态 | 应用代码、check 名称 | 管理员门禁；必须先于 S2 |
| S2 | S1,G10 | RuntimeManifest 契约、强制 CI readback | `desktop/src/core/atlas/**`, `desktop/src/core/animation/**`, 对应 tests | package/lock/config | 串行 |
| S3 | S2 | atlas/timeline exports | `desktop/src/core/gaze/**`, 对应 tests | Rust、renderer、配置 | 串行 |
| S4 | S3 | gaze exports | `desktop/src/core/state/**`, `interaction/**`, `ports.ts`, 对应 tests | Rust、renderer、配置 | 串行 |
| S5 | S4 | typed ports/effects | `desktop/src-tauri/src/{lifecycle/**,tray/**}`、`desktop/src/adapters/tauri/lifecycle*`、组合根、`scripts/native-test/scenarios/S5/**`、`.github/checklists/S5/**`、tests | pointer/interaction feature、harness engine | 串行组合根所有者 |
| G5 | S5 | main 上的 S5 scenario 与 artifact | `validation/desktop/G5/**` | 代码、artifact | post-merge native gate |
| S6 | G5 | lifecycle/window handle | `desktop/src-tauri/src/pointer/**`、`desktop/src/adapters/tauri/pointer*`、组合根、`scripts/native-test/scenarios/S6/**`、`.github/checklists/S6/**`、tests | core state/renderer、harness engine | 串行组合根所有者 |
| G6 | S6 | main 上的 S6 scenario 与 artifact | `validation/desktop/G6/**` | 代码、artifact | post-merge native gate |
| S7 | G6 | pointer protocol | `desktop/src-tauri/src/interaction/**`、`desktop/src/adapters/tauri/interaction*`、组合根、`scripts/native-test/scenarios/S7/**`、`.github/checklists/S7/**`、tests | core gaze/state、harness engine | 串行组合根所有者 |
| G7 | S7 | main 上的 S7 scenario 与 artifact | `validation/desktop/G7/**` | 代码、artifact | post-merge native gate |
| S8 | G7 | ports/effects | `desktop/src/core/settings/**`、`desktop/src/adapters/tauri/store*`、`desktop/src-tauri/src/settings/**`、组合根与 tests | placement feature、harness engine | 串行组合根所有者 |
| S9a | S8 | sanitized settings | `desktop/src/core/geometry/**`、`desktop/src-tauri/src/placement/**`、组合根、`scripts/native-test/scenarios/S9a/**`、`.github/checklists/S9a/**`、tests | startup/autostart、harness engine | 串行组合根所有者 |
| G9a | S9a | main 上的 S9a scenario 与 artifact | `validation/desktop/G9a/**` | 代码、artifact | post-merge native gate |
| S9b | G9a | placement API | `desktop/src-tauri/src/startup/**`、`desktop/src/adapters/tauri/startup*`、组合根、`scripts/native-test/scenarios/S9b/**`、`.github/checklists/S9b/**`、tests | renderer、harness engine | 串行组合根所有者 |
| G9b | S9b | main 上的 S9b scenario 与 artifact | `validation/desktop/G9b/**` | 代码、artifact | post-merge native gate |
| S11 | G9b | complete adapters | `desktop/src/renderer/**`、`desktop/src/main.ts`、`desktop/styles/**`、`scripts/native-test/scenarios/S11/**`、`.github/checklists/S11/**`、perf/tests | 生产素材、harness engine | 串行 |
| G11 | S11 | main 上的 S11 scenario 与 artifact | `validation/desktop/G11/**` | 代码、artifact | post-merge native/perf gate |
| S12 | G0b,G11 | evidence-approved manifest、renderer | `desktop/assets/pet/**`、`desktop/runtime-manifest.json`、`desktop/src-tauri/icons/**`、`desktop/src-tauri/tauri.conf.json`、视觉验证 | root 原资产（除非 manifest 明确批准同一哈希） | 串行组合根所有者 |
| S13 | S12 | 完整应用与 rights verifier | `.github/workflows/desktop-build.yml`、`scripts/{verify-desktop-bundle,desktop-transport,native-test/scenarios/S13}/**`、`.github/checklists/S13/**`、packaging docs | core、素材字节、harness engine | 串行；输出 canonical unpatched binary/bundle inputs |
| G13 | S13 | main 上的 S13 installers/digest | `validation/desktop/G13/**` | 代码、artifact | post-merge installer gate |
| K14os | G13,G10 | 双平台内部 installers、产品身份、签名账号/服务资格 | `docs/desktop/signing/os-identities.json`、`docs/desktop/adr/windows-signing-backend.md`、`validation/desktop/K14os/**`；CA/Apple/Azure 资格外部状态 | 应用/素材、私钥/认证值、Environment 配置 | OS 签名 profile/资格门禁 |
| G14 | K14os | S13 artifact digest、已冻结的 signing profile、job-to-Environment/角色合同 | `.github/environments/**`、`.github/rulesets/**`、`.github/workflows/{updater-key-challenge,os-signing-probe}.yml`、`docs/desktop/admin-roles.json`、`scripts/configure-release-controls/**`、`validation/desktop/G14/**`；GitHub Environment/ruleset/immutable-release 外部状态 | 应用/素材、secret/identity 写入、release artifact | 空 Environment 管理员门禁 |
| K14v | G14 | 空且受保护的签名 Environments、固定 probe workflow、K14os profile | `validation/desktop/K14v/**`；CA/Apple/Azure identity/federation/Environment secret 外部状态 | 应用/素材、私钥值、workflow | OS 签名 identity 激活与 probe 门禁 |
| S14 | K14v | 同一 canonical unsigned binary/bundle inputs、真实机证据、已验证签名身份 | `.github/workflows/desktop-promote.yml`、`scripts/{signing,promotion}/**`、`scripts/native-test/scenarios/S14/**`、`.github/checklists/S14/**`、`desktop/src-tauri/config/release/**`、`docs/desktop/release/**` | core、素材字节、harness engine、既有 tag | 串行 |
| S15a1 | S14 | 固定身份、updater 协议 | `desktop/src/core/updater/**`、`desktop/schemas/update-channel-v1.schema.json`、纯 TS tests | Tauri/Rust/workflow、旧公开 Release | 串行 |
| K15a | S15a1,G14 | 空的受保护 K1 Environment | `desktop/src-tauri/updater-keys/K1.pub`、`docs/desktop/keys/K1.json`、`validation/desktop/K15a/**`；K1 Environment secret 外部状态 | 应用行为、私钥文件 | 离线密钥仪式 |
| S15a2 | K15a | updater trust/manifest core、K1 pubkey | `desktop/src/updater/**`、`desktop/src-tauri/src/updater/**`、`desktop/vendor/tauri-plugin-updater/**`、组合根、`scripts/native-test/scenarios/S15a2/**`、`.github/checklists/S15a2/**`、tests | channel workflow、OS/updater 私钥、harness engine | 串行组合根所有者；受审 vendored patch |
| G15a2 | S15a2 | main 上的升级 scenario/artifact | `validation/desktop/G15a2/**` | 代码、artifact | post-merge native gate |
| S15b | G15a2 | updater result/health contract | `desktop/src/core/health/**`、`desktop/src-tauri/src/health/**`、组合根、`scripts/native-test/scenarios/S15b/**`、`.github/checklists/S15b/**`、tests | channel 签名、harness engine | 串行组合根所有者 |
| G15h | S15b | main 上的 health scenario/artifact | `validation/desktop/G15h/**` | 代码、artifact | post-merge native gate |
| S15a3 | G15h | 可恢复 updater client、K1、S13 build/S14 promotion contracts | `.github/workflows/{desktop-channel,desktop-promote}.yml`、`scripts/{channel,updater-sign}/**`、`update-channels/k1/canary.json`、`docs/desktop/{keys,channels}/**` 与对应 tests；新 protected-main build run | core、OS 签名私钥、旧公开 Release | 只发布 canary |
| G15 | S15a3 | canary artifacts/channel | `validation/desktop/G15/**`、签署的 rollout decision | 代码、既有 artifact/manifest | 7 天 canary 门禁 |
| S15a4 | G15 | 通过 canary 的同一 final digest | `update-channels/k1/beta.json` | build/sign/core、artifact 字节 | canary→beta deployment |
| G15b | S15a4 | beta manifest 与同一 final digest | `validation/desktop/G15b/**`、签署的 rollout decision | 代码、既有 artifact/manifest | 14 天 beta 门禁 |
| S15a5 | G15b | 通过 beta 的同一 final digest | `update-channels/k1/stable.json` | build/sign/core、artifact 字节 | beta→stable deployment |
| A15c | S15a5 | safe-mode/backup 与发布模型 | `docs/desktop/adr/repair-tool.md` | repair 实现、主应用 | 冻结的 Rust CLI 架构门禁 |
| S15c1 | A15c | 已批准 repair ADR | `repair/src/**`、`repair/{Cargo*,tests/**}`、本地恢复 fixtures/docs | 主应用、签名 workflow | 串行 |
| G15cR | S15c1 | repair bytes/图标/文档 | `docs/desktop/rights/**`、`repair/assets/**`、`validation/desktop/G15cR/**` | 主应用素材、bundle/release | repair rights amendment |
| S15c2 | G15cR | 可测试 repair source/resources、rights amendment、S14 `repair-cli-v1` profile | `.github/workflows/repair-build.yml`、`scripts/native-test/scenarios/S15c2/**`、`.github/checklists/S15c2/**`、`docs/desktop/repair/{packaging,signing}.md` | `desktop-promote.yml`、主应用、G15cR 后新增视觉字节、旧公开 Release | 串行 |
| K15b | S15a5 | 稳定 K1 系统与轮换决定 | `.github/environments/desktop-updater-signing-k2.json`、`desktop/src-tauri/updater-keys/K2.pub`、`docs/desktop/{admin-roles.json,keys/K2.json}`、`validation/desktop/K15b/**`；独立 K2 Environment/secret 外部状态 | K1 私钥、应用行为 | 可选离线 K2 仪式 |
| S15r1 | K15b | K2 pubkey/fingerprint、S13 build/S14 promotion contracts | `desktop/src/updater/**`、`desktop/src-tauri/src/updater/**`、组合根、`scripts/native-test/scenarios/S15r1/**`、`.github/checklists/S15r1/**`、`update-channels/k1/canary.json`、bridge tests；新 protected-main build run | channel workflow、K1/K2 私钥、harness engine | 只发布 K1-signed bridge canary |
| G15r1c | S15r1 | bridge canary artifacts/channel | `validation/desktop/G15r1c/**`、签署的 rollout decision | 代码、既有 artifact/manifest | 7 天 bridge canary 门禁 |
| S15r1b | G15r1c | 通过 bridge canary 的同一 final digest | `update-channels/k1/beta.json` | build/sign/core、artifact 字节 | bridge canary→beta deployment |
| G15r1b | S15r1b | bridge beta manifest 与同一 final digest | `validation/desktop/G15r1b/**`、签署的 rollout decision | 代码、既有 artifact/manifest | 14 天 bridge beta 门禁 |
| S15r1s | G15r1b | 通过 bridge beta 的同一 final digest | `update-channels/k1/stable.json` | build/sign/core、artifact 字节 | bridge beta→stable deployment |
| G15r | S15r1s | bridge stable final digest、K1 feed | `validation/desktop/G15r/**`、rotation decision | 代码、既有 artifact | ≥180 天轮换门禁 |
| S15r2 | G15r | 批准的 rotation decision、稳定 bridge、S14/S15a3 promotion contract | `.github/workflows/{desktop-channel,desktop-promote}.yml`、`scripts/{channel,updater-sign}/**`、组合根、`scripts/native-test/scenarios/S15r2/**`、`.github/checklists/S15r2/**`、`update-channels/k2/canary.json`、rotation tests | core feature、两把私钥、harness engine、既有 artifact | K2 release-path PR 后只发布 K2 canary；保留 K1 bridge feed |
| G15r2c | S15r2 | K2 canary artifacts/channel | `validation/desktop/G15r2c/**`、签署的 rollout decision | 代码、既有 artifact/manifest | 7 天 K2 canary 门禁 |
| S15r2b | G15r2c | 通过 K2 canary 的同一 final digest | `update-channels/k2/beta.json` | build/sign/core、artifact 字节 | K2 canary→beta deployment |
| G15r2b | S15r2b | K2 beta manifest 与同一 final digest | `validation/desktop/G15r2b/**`、签署的 rollout decision | 代码、既有 artifact/manifest | 14 天 K2 beta 门禁 |
| S15r2s | G15r2b | 通过 K2 beta 的同一 final digest | `update-channels/k2/stable.json` | build/sign/core、artifact 字节 | K2 beta→stable deployment |

表中“组合根”严格只展开为 `desktop/src-tauri/src/lib.rs`、`desktop/src/main.ts`、`desktop/src-tauri/Cargo.toml`、`desktop/src-tauri/Cargo.lock`、`desktop/package.json`、`desktop/package-lock.json`、`desktop/src-tauri/tauri.conf.json` 与 `desktop/src-tauri/capabilities/**`；步骤只能改实现其本功能所必需的子集，不能借此修改其他 feature。

为避免每行重复，所有 ID 的 Owned paths 自动且仅追加 `validation/desktop/<ID>/**`；表内已经显式写出的同一路径只是强调。任何 ID 都不能写入另一个 ID 的 validation 目录。

生成文件初始所有者：`package-lock.json`、`Cargo.lock` 和基础配置由 S1 创建。只有表中明确标为“组合根所有者”的串行步骤可更新上述共享文件；必须先提交该步骤的独占 `dependency-prep`，其他在途分支 rebase 后才能继续。任一步骤需要扩大 owned paths 时必须先修改本表并审阅，不能用表外例外覆盖“禁止修改”。

唯一 DAG：

```text
G0a → S1 → S10a → G10 → S2 → S3 → S4 → S5 → G5 → S6 → G6 → S7 → G7 → S8
S8 → S9a → G9a → S9b → G9b → S11 → G11
(G0a, G10) → G0b
(G0b, G11) → S12 → S13 → G13
(G13, G10) → K14os → G14 → K14v → S14
S14 → S15a1 → K15a → S15a2 → G15a2 → S15b → G15h → S15a3 → G15 → S15a4 → G15b → S15a5
S15a5 → A15c → S15c1 → G15cR → S15c2
S15a5 → K15b → S15r1 → G15r1c → S15r1b → G15r1b → S15r1s → G15r
G15r → S15r2 → G15r2c → S15r2b → G15r2b → S15r2s  # 可选、独立的完整 K2 rotation train
```

上图只为可读性省略已经由更长路径保证的传递边（例如 S1→S2、G14→K15a）；不得据此删除依赖表中的直接硬前置，执行器只解析表。

## 6. 分步 PR 施工卡

所有实现分支都从“硬前置已合并后的最新 `main`”创建。每个实现 PR 必须写 `validation/desktop/<ID>/red-green.md`，记录先失败测试、失败原因、最小实现与转绿命令。回滚默认使用 `git revert <merge-sha>`；不得删除后续共享目录。

标为“非代码 gate”或“受保护 deployment”的步骤先在外部状态上执行，再用 `desktop/<id>-evidence` PR 只提交该行 owned `validation/desktop/<ID>/**` 内的签署结果、run/deployment ID、输入/输出 digest 与 readback；不得夹带 workflow、脚本、应用或素材变更。唯一例外是依赖表明确分配了一个 `update-channels/...json` 的步骤：它使用 §3.6 的 channel/evidence PR，并且只能额外提交该一个由固定 workflow 生成的 manifest。固定 verifier 在 PR 上重新下载不可变证据并复算，独立 CODEOWNER 批准、PR 合入 protected main、且 live readback 仍匹配后，该步骤才算完成。workflow 本身不直接 push main，Actions artifact 或口头确认都不能替代此闭环。

### G0a — 路线与产品身份冻结

**分支**：`desktop/00a-product-decision`。**模型**：默认 + 产品负责人。
**输入**：根 `LICENSE`、`SOURCE_ATTRIBUTION.md`、固定上游提交；不得假设用户拥有额外权利。

先记录失败事实：现有授权文本没有覆盖独立 Windows/macOS 桌宠。随后只做二选一决策：路线 A 保留 `NewLai Pet` 并获取上游授权；路线 B 改用 clean-room original 角色，并在此处冻结最终产品名、角色名、bundle ID、视觉差异准则和素材交付合同。决策文件包含负责人、日期、依据和 superseded 分支列表。

退出：产品负责人签署 `docs/desktop/product-decision.md`，身份字段无占位符；S1 才可创建。此步不代表素材权利已批准。
回滚：S1 前可用新 G0a 决策 supersede；S1 后变更身份必须另立迁移步骤，不能原地改历史。

### G0b — Rights Evidence Approval

**分支**：`desktop/00b-rights-gate`。**模型**：默认 + 独立人工法律/权利审阅。
**输入**：G0a 路线、G10 已保护 main、外部授权原件或 clean-room 来源链；证据收集可提前，PR 可与 S2–S11 通用运行时并行。

先写 verifier 的失败 fixture，再建立 rights manifest/schema、`rights-review.json` 与 CODEOWNERS。此阶段只验证证据和预期哈希；生产视觉字节仍不得进入 desktop 输入。GitHub Environment 的实际创建与回读归 G14，不能在此用文档假装已配置。

验证与预期：

```bash
node scripts/verify-desktop-rights.mjs --phase pre-approval
# expected: pass only when no production visual asset exists under desktop inputs

node scripts/verify-desktop-rights.mjs --phase evidence-approved \
  --manifest docs/desktop/rights/rights-manifest.json \
  --review docs/desktop/rights/rights-review.json
# expected: pass only with route-consistent grant/source chain, independent review and expected hashes;
# source asset bytes may still be absent
```

退出：rights CODEOWNER 批准证据 PR，机器检查通过；不存在可由实现者自填并绕过的字段。Environment 留到每次 S14 promotion 现场审批。
回滚：revert G0b PR；S1–S11 可保留占位运行时，但 S12 及任何独立 bundle 继续被拒绝。

### S1 — 工程骨架、身份与安全默认值

**分支**：`desktop/01-scaffold`。**模型**：默认。
**输入**：G0a 已冻结的身份、本计划 §1–2；程序化 Canvas 占位，不读取任何图像文件。

先写测试：Tauri 配置满足透明/无边框/置顶/不可缩放/无阴影；macOS 明确 `app.macOSPrivateApi=true`，Windows 明确 `noRedirectionBitmap=true`；bundle ID 固定；生产 CSP/导航/devtools/capabilities 最小；renderer 有 canvas、可访问名称和错误回退。使用 macOS private API 是透明窗口的必要条件，因此产品固定为 Developer ID/GitHub 分发，不进入 Mac App Store。

实现 Tauri 2 + Vanilla TS/Vite、Vitest/V8/fast-check、Playwright(renderer-contract)、WebdriverIO Tauri service(webview native test)、ESLint/Prettier。WDIO embedded WebDriver 只在 Rust `e2e` feature + debug/test profile 同时满足时编译和注册；production capability、前端入口和 bundle 不得包含它。一次性提交全部基础依赖和两个 lockfile；只有依赖表中后续“串行组合根所有者”可为自己的功能提交独占 dependency-prep，其中 S15a2 对受审 updater fork 的引入是显式例外。设置 core per-file/all-files 四项 80% 阈值。Rust 通过官方 rustup 安装，文档记录版本。

命令：

```bash
cd desktop
npm ci
npm run format:check
npm run lint
npm run typecheck
npm run test:coverage
cargo test --locked --manifest-path src-tauri/Cargo.toml
```

退出：命令全绿；`desktop/assets/pet/` 无生产素材；release-profile 静态检查证明无 WebDriver 插件符号、ACL 和监听端口；`npm audit`/`cargo audit` 的高危项为零。
输出契约：稳定 scripts、lockfiles、app identity、目录边界。
回滚：revert S1 merge；根 Codex 包不受影响。

### S10a — PR CI、Native Harness 与 red→green 审计

**分支**：`desktop/10a-ci-harness`。**模型**：强。
**输入**：S1 稳定 scripts。S10a/G10 完成前不得创建 S2 分支。

实现三条 workflow：

- `Desktop CI`：`static`、`core-coverage`、`rust`、`renderer-contract`、`workflow-policy`；Actions 全 SHA 固定，Node/Rust lock 构建，core all-files/per-file 80%。
- `Desktop Native WebView`：在 GitHub-hosted `windows-2025`、`macos-15`（arm64）、`macos-15-intel` 运行 `test:native:webview`；触发为 PR、main push 与手动运行。
- `Desktop Native OS`：只允许 protected-main 的手动/promotion 调用，不对 fork PR 执行；主机池基线为 `[self-hosted,newlai-pet,windows-10-22h2,x64,interactive]`、`[self-hosted,newlai-pet,windows-11,x64,interactive]`、`[self-hosted,newlai-pet,macos-12,arm64,interactive]`、`[self-hosted,newlai-pet,macos-12,x64,interactive]`，实际 job 还必须附加 run/phase 唯一 label。基线标签代表实际最低版本主机，不得把新系统兼容模式冒充最低版本；四类之外再用当前 Windows 11/macOS 最新版做非阻塞前瞻。输入必须包含 commit、artifact digest 和场景版本；四类 host/controller inventory 由 `g5n-dev` runner 管理员登记，每阶段从已知快照启动 one-shot runner、无发布 secrets，结束后注销并销毁或还原，不能留下可接下一任务的共享 registration。

每台 OS 测试主机必须拆成两个信任域：随系统启动、无桌面交互权限的固定 out-of-band controller，以及只在专用测试用户登录会话运行的最小 UI witness；Actions worker 本身不得作为跨重启 controller。真实注销/重启采用版本化两阶段协议和 **run-scoped、one-shot JIT/`--ephemeral` runner registrations**：Phase A 使用只匹配本次 run/host/nonce 的 `phase-a` label，生成并上传绑定 run ID、host ID、nonce、commit、artifact digest、scenario version 与原 boot/session ID 的 ticket；A job 成功后该 runner 必须自动注销，controller 从 GitHub/control plane 双向确认 job success 且 registration 已 removed/offline，才允许 logout/reboot。Phase B 虽可因 `needs` 先进入队列，但只匹配不同的本次 `phase-b` label；controller 只有在新 boot、专用用户新登录会话和 UI witness 握手全部就绪后，才用工作区外的窄 runner-registration authority 创建新的 one-shot registration，因此 B 不可能在重启前获派。B 从 Actions 取回 ticket，证明 registration/boot/session ID 已变化、UI witness hash/version 匹配、应用由 OS autostart 恰好启动一次且 PID/start time/签名/窗口位置合法。卸载清理用新 nonce 与全新的 A/B registrations 再跑一轮，证明下次登录不再启动。禁止“同一 persistent runner/label + 普通 `needs`”充当调度屏障。controller 与 witness 的安装包/hash、IPC schema、JIT label/token scope、超时、重放拒绝和恢复流程写入 `docs/desktop/test-hosts/**`；二者不持 release secrets、不接受 PR ref/任意命令或任意路径，registration authority 不进入 repo、job secret 或 artifact。

所有 jobs 默认 `permissions: contents: read`，按需只给 artifact job `actions: read`；不使用 `pull_request_target`，不读取 PR secrets。S10a 只拥有版本化 harness engine、click-probe、平台 UI driver、schema/template、两阶段 controller/ticket 协议与真实登录周期驱动；后续步骤分别拥有 `scripts/native-test/scenarios/<ID>/**` 和 `.github/checklists/<ID>/**`，不能修改 engine/controller 或另建未登记 workflow。S10a 同时建立基础 `CODEOWNERS`，在任何权利 PR 出现前就覆盖 §2 列出的全部未来敏感路径，以及 workflows、branch/ruleset manifests、harness engine、`CODEOWNERS` 自身和 `validation/desktop/**`，避免在同一 PR 新增规则并自证；`review-roles.json` 必须写真实 GitHub login/ID，并为 rights/evidence PR 指定有仓库 review 权限、不同于被审步骤实现者/运行触发者的 reviewer，拒绝占位符或同一身份兼任。

required check context 精确固定为：`Desktop CI / static`、`Desktop CI / core-coverage`、`Desktop CI / rust`、`Desktop CI / renderer-contract`、`Desktop CI / workflow-policy`、`Desktop Native WebView / windows-x64`、`Desktop Native WebView / macos-arm64`、`Desktop Native WebView / macos-x64`。workflow/job display name 变更必须先同步 protection manifest。

验证：对 workflow 做静态测试，拒绝浮动 `uses:`；用失败测试 PR 证明 CI 非假绿；保存成功/失败 run ID 与 runner label inventory。
退出：八个 required contexts 在 S10a PR 实际出现且通过；失败 fixture 为非 0；四类 OS runner 的空载 probe 均成功；基础 CODEOWNERS 与 reviewer 身份可解析；workflow/config 合并 main。
回滚：revert S10a；已登记 runner 保持禁用 PR 代码，等待修复。

依据：[GitHub ephemeral/JIT self-hosted runners](https://docs.github.com/en/actions/reference/runners/self-hosted-runners#ephemeral-runners-for-autoscaling)。

### G10 — Main Protection 管理员门禁

**操作负责人**：`g5n-dev` 仓库管理员；需要 repository `Administration: write`。
**输入**：S10a 已合并、八个 check context 已实际出现、基础 CODEOWNERS/reviewer readback、`.github/branch-protection-main.json`。

执行并回读：

```bash
gh api -H 'X-GitHub-Api-Version: 2026-03-10' \
  --method PUT repos/g5n-dev/newlai_pet/branches/main/protection \
  --input .github/branch-protection-main.json
gh api -H 'X-GitHub-Api-Version: 2026-03-10' \
  repos/g5n-dev/newlai_pet/branches/main/protection \
  > validation/desktop/G10/branch-protection.json
node scripts/verify-branch-protection/index.mjs \
  --snapshot validation/desktop/G10/branch-protection.json \
  --expected .github/branch-protection-main.json
```

manifest/验证器精确要求上述八个 contexts、`strict=true`、`enforce_admins=true`、至少一次批准、CODEOWNER review、dismiss stale review、last-push approval、conversation resolution、禁止 force-push/delete。2026-09-04 只读基线为 main 尚未保护；因此 G10 当前明确未完成。

退出：live verifier 为 0；回读 JSON 与 SHA 存入 `validation/desktop/G10/` 并经非操作人审阅。调用返回 403/404、context 缺失或 readback 不匹配时状态为 `blocked`；不得降低规则。S2 必须消费 snapshot SHA，并在合并前再做一次 live readback。
回滚：错误规则通过审计过的 manifest forward-fix；不得为合并 S2 临时关闭保护。

### G5/G6/G7/G9a/G9b/G11/G13/G15a2/G15h — Post-merge Native Gates

这些是非代码 gate，解决 self-hosted runner 的信任边界：功能 PR 先靠 required unit/Rust/WebView checks 合并；随后只从 protected `main` 启动 S10a 固定的 `Desktop Native OS` workflow，构建/下载该 merge commit 的 artifact，并读取已经进入 main 的版本化 scenario。workflow 不 checkout 未合并 ref、不执行 PR 提供的脚本、无发布 secrets。下一实现步骤硬依赖对应 gate，因此真机失败不能继续流水线。

| Gate | 生产者 | 必跑场景 | 阻塞后继 |
|---|---|---|---|
| G5 | S5 | 四类主机 tray/lifecycle smoke | S6 |
| G6 | S6 | global pointer、heartbeat、DPI payload、隐私 | S7 |
| G7 | S7 | drag、下层 click-probe、四种穿透恢复 | S8 |
| G9a | S9a | 多屏交换/拔插、负坐标、125/150/200% | S9b |
| G9b | S9b | 第二实例、CLI reset、同机 Phase A→logout/reboot→Phase B autostart/卸载清理 | S11 |
| G11 | S11 | 完整用户旅程与 30 分钟性能预算 | S12 |
| G13 | S13 | Win10 22H2/Win11、macOS 12 arm64/x64 的 NSIS/MSI/APP/DMG 内部安装、升级、卸载；WebView2 两路径 | K14os |
| G15a2 | S15a2 | 测试 K1 的 NSIS/macOS N→N+1 与中断 | S15b |
| G15h | S15b | 三连 crash、safe mode、三份备份恢复 | S15a3 |

每个 gate 的 evidence-only PR 保存 workflow run ID、被测 merge SHA、artifact/final digest、scenario version、四类 runner image/session、机器结果和独立见证者到 `validation/desktop/<Gate>/`；PR verifier 必须从 run ID 重取 artifact 并复算。任一必需平台失败或证据 digest 不一致即 `blocked`；修复必须走新实现 PR 合并 main 后重新运行，再提交新的 evidence PR，不能重写旧证据。

### S2 — Manifest 与时间线核心

**分支**：`desktop/02-atlas-timeline`。**模型**：强。
**输入**：S1/S10a 已合并、G10 snapshot SHA 与 live readback 均通过；§3.1；使用任意尺寸的程序化 manifests。

先写测试：schema 错误、格子越界、columns/durations 不等长、neutral 越界；timeline 的 0/边界/总时长/超大 elapsed、loop/once/hold、倍速、睡眠跳跃与完成 token。

实现 atlas source rect、manifest validator、单调时间 timeline。不得出现 `192/208/8/11` 硬编码。

验证：`npm run test:coverage -- atlas timeline`；对应文件 per-file ≥80%。
退出：属性测试保证任意合法 elapsed 只返回合法格；无平台导入。
回滚：revert S2 merge。

### S3 — 凝视核心

**分支**：`desktop/03-gaze-core`。**模型**：强。
**输入**：§3.2、S2 manifest API。

先写测试：四基数/四对角/全部中心角、11.25° 与 348.75°、零向量；24/36 DIP 边界；3° 扇区滞回；同扇区持续移动；连续多次亚 1.5 DIP 位移最终累计 accepted；同方向跨 deadzone；窗口移动而鼠标静止；999/1000ms；睡眠大跳跃；cursor/window 分处不同 scale 屏、负坐标与上下排列。

实现纯 TS `PointerSampleV1` reducer 和视觉 frame resolver。transport 信息完整保留，只有最终 frame 可去重。

退出：fast-check 保证任意有限向量返回合法方向或 neutral/idle；全部边界测试绿。
回滚：feature flag `gaze=false`（本 PR 引入）后 revert。

### S4 — 手势与视觉状态机

**分支**：`desktop/04-state-gesture`。**模型**：强。
**输入**：§3.3、S2/S3 exports。

先写测试：`<6/==6 DIP`、拖拽后无 click、double-click 抢占 wave、pointercancel/失焦；完整优先级；旧 token；once 完成恢复 semantic；固定 RNG；reduced motion；托盘四个语义状态可达。

实现 typed events、reducer `[state,effects]`、gesture classifier、ambient scheduler 与 Clock/Rng ports。

退出：所有状态可由公开 typed event 达到；无计时器/IO/平台调用。
回滚：feature flags 分别关闭 reactions/ambient，随后 revert。

### S5 — 窗口生命周期与托盘

**分支**：`desktop/05-window-tray`。**模型**：强。
**输入**：冻结 app ID、S4 ports。single-instance 插件必须最先注册。

先写 Rust 测试与 `scripts/native-test/scenarios/S5/**`：托盘命令 allowlist、未知命令拒绝、single-instance 最小回调 effect；配置测试检查 macOS accessory policy、Windows `noRedirectionBitmap`。

实现 hidden-first 透明窗口、托盘、show/hide/always-on-top/reset/quit、renderer 崩溃时原生退出路径。此步不实现穿透。

验证：`test:native:webview` 证明真实 Tauri WebView/IPC；`test:native:os` 使用平台 UI 自动化点击真实托盘并退出。证据写 `validation/desktop/S5/<os>-<arch>/`；Playwright 仍只称 renderer-contract。

PR 退出：unit/Rust 与三个 GitHub-hosted WebView jobs 全绿，scenario/checklist 已评审后合并；合并后的四类受控主机结果归 G5，G5 通过前 S6 被阻塞。
回滚：revert S5；无持久数据迁移。

### S6 — 全局指针桥

**分支**：`desktop/06-pointer-bridge`。**模型**：强。
**输入**：G5 通过；§3.2 payload；Tauri 全局 cursor/window/monitor physical coordinates。

先写 Rust/contract 测试：30Hz 上限、heartbeat、相对最后 accepted point 的累计 `motionSequence`、连续小位移、同扇区移动不丢、cursor 与 anchor 的 monitor/scale 独立变化、负坐标、上下排列、多 DPI；隐私测试检查坐标不进日志/store/network。

实现 Rust 采样器与窄 IPC；全局鼠标失败时发 capability unavailable 并回 idle。

PR 退出：pointer IPC ≤30/s、999/1000ms contract 与隐私测试通过；合并后的真实 OS 场景归 G6，G6 通过前 S7 被阻塞。
回滚：`gaze=false`，revert S6。

### S7 — 拖拽与穿透安全闭锁

**分支**：`desktop/07-interaction-safety`。**模型**：强。
**输入**：G6 通过；§3.3–3.4、S5 tray、S6 pointer。

先写 native 测试：拖拽方向/丢失捕获；托盘失败、快捷键冲突、renderer crash、睡眠、穿透中第二实例、`--reset-interaction`；启动永远 interactive。点击穿透必须由桌宠下方的独立 click-probe 进程收到事件来判定，不能由 WebView 自报。

实现 native drag、跑动速度事件、整窗穿透。仅在托盘与 `CommandOrControl+Shift+P` 均成功后开放穿透；第二实例和 CLI 恢复路径始终存在。

PR 退出：Rust/WebView/失败注入 tests 与 S7 scenario/checklist 通过评审后合并；真实 enable→probe→四种恢复路径归 G7，G7 通过前 S8 被阻塞。
回滚：设置 feature flag `clickThrough=false`，发布 forward-fix；随后 revert。

### S8 — 设置 schema 与持久化

**分支**：`desktop/08-settings-store`。**模型**：默认。
**输入**：G7 通过、S4 ports；设置版本 `v1`。

先写测试：空/坏 JSON、缺字段、NaN/Infinity、越界、未知未来版本、v0→v1；250ms debounce、拖动期间不写、退出 flush；绝不保存穿透和指针坐标。

实现 sanitize/migrate、store adapter。设置入口先由托盘子菜单提供：scale、opacity、gaze mode、reduced motion、ambient。

退出：损坏 store 不阻止启动；隐私字段 allowlist 测试通过。
回滚：忽略旧设置并使用 defaults，不删除用户文件；revert S8。

### S9a — 多屏 Placement 与 DPI

**分支**：`desktop/09a-placement-dpi`。**模型**：强。
**输入**：§3.4、S8 store。禁止 `window-state`。

先写测试：稳定 display ID 优先；最大旧矩形交集/最近屏/主屏回退；同型号显示器交换；负原点、上下排列、125/150/200%、不同 DPI 接缝、taskbar/Dock 变化、拔屏、窗口大于 work area、ID 匹配失败与 `u/v` round-trip。

实现 placement 唯一真源，保存稳定 ID 与旧 bounds/work area/scale/window rect/topology；显示器/DPI 事件统一 reconstruct + clamp。应用固定在当前虚拟桌面，不宣称 all-workspaces。

PR 退出：geometry fixtures、Rust/WebView 与 S9a scenario/checklist 通过后合并；真实 DPI/显示器证据归 G9a，G9a 通过前 S9b 被阻塞。
回滚：启动落主屏右下默认位置；保持旧设置向前兼容；revert S9a。

### S9b — 启动、Autostart 与 Single-instance

**分支**：`desktop/09b-startup-lifecycle`。**模型**：强。
**输入**：G9a 通过、S9a placement API 与 §3.4 唯一启动顺序。

先写测试：single-instance 必须在插件注册顺序首位；hidden-first 严格顺序；autostart 启停幂等、升级路径、卸载清理；第二实例执行 show/unminimize/focus/关闭穿透/re-clamp；`--reset-interaction` 在损坏设置下仍生效。

实现完整启动编排、autostart 和 single-instance callback。Windows 与 macOS 的 autostart 不是配置 mock：S9b scenario 必须使用 S10a 固定的同机两阶段 ticket 协议，Phase A 启用 autostart 后由 controller 在 job 返回后真实注销/重启，Phase B 在新的 boot/session 中验证 UI witness 所见的单次启动、位置与签名；随后禁用/卸载并再跨一次登录周期证明无残留。任一阶段 host ID、nonce、artifact digest 不一致，或只是重启应用而未改变 boot/session ID，G9b 都失败。

PR 退出：启动顺序/插件测试与 S9b scenario/checklist 通过后合并；真实登录周期、第二实例与 CLI 证据归 G9b，G9b 通过前 S11 被阻塞。
回滚：关闭 autostart，保留手动启动与安全恢复；revert S9b。

### S11 — Renderer、反应体验与性能

**分支**：`desktop/11-renderer-polish`。**模型**：默认。
**输入**：G9b 通过、S4–S9b 的 typed ports；仍使用程序化占位。

先写测试：Canvas HiDPI source rect、相同 frame 不重绘、单击/双击/拖拽旅程、所有托盘设置、错误回退、reduced motion、无远程请求。WebdriverIO 只负责带 `e2e` feature 的实际 Tauri WebView/IPC；平台 UI 与 click-probe 负责 OS 行为；Playwright 只测 renderer contract。

实现 canvas renderer、交互反馈和性能采样。按 §3.5 在两个参考平台采 30 分钟。

PR 退出：renderer-contract/WebView、release-profile 隔离及性能采样器测试通过，S11 scenario/checklist 合并；两平台完整旅程与 §3.5 数值性能证据归 G11，G11 通过前 S12 被阻塞。
回滚：关闭 ambient/高级渲染 flag 并 forward-fix；revert 不删除用户设置。

### S12 — 获准视觉资产集成

**分支**：`desktop/12-approved-assets`。**模型**：强 + 独立视觉审阅。
**输入**：G0b `evidence-approved` rights manifest、G11 通过的 renderer。路线 A/B 已冻结。

先写测试：所有视觉源资产、图标、托盘图、DMG 背景、截图/预览均在 exact allowlist；hash/尺寸/格子/时序合法；必需九个 clip、16 向与 22.5° profile 严格满足；无 symlink/base64/目录外资源。

实现获准 atlas、runtime manifest、图标与署名。若路线 A 批准当前图集，明确：8×11、192×208；idle 为 row0 col0–5；neutral 为 row0 col6；row0 col7 为空；16 向位于 row9–10。否则使用新素材自己的 manifest，不改 core。

验证：`verify-desktop-rights --phase source-assets-present` 对实际字节与预期清单做集合相等；独立视觉 reviewer 核对角色、动作与授权路线，不能由实现者自批。

退出：source-assets-present 检查、两平台 renderer 快照与独立视觉 QA 全过；Environment 留给 S14 的具体发布。
回滚：revert S12，应用回程序化占位；不删除权利审计历史。

### S13 — 无写权限原生构建与内部 Alpha

**分支**：`desktop/13-native-build`。**模型**：强。
**输入**：S12 完整应用、rights verifier。

build jobs：Windows x64 先以锁定 Tauri CLI 执行 `tauri build --no-bundle`，输出 canonical **unpatched** application PE，再用同一 PE、同一 source/config/resource inputs 执行 `tauri bundle --no-sign` 分别生成仅供内部验证的 NSIS/MSI；producer manifest 固定 base PE、bundle inputs、Tauri CLI/bundler 版本与两种 unsigned installer/inner marker digest。S13 不把 NSIS/MSI 内两个经 package-type patch 的 inner EXE误称为同一字节。macOS arm64/x64 显式设置 `MACOSX_DEPLOYMENT_TARGET=12.0`，同时保留 unsigned inner `.app`，并输出仅供内部验证的 ad-hoc `.app.tar.gz`/DMG，不宣称身份认证。macOS verifier 用 Mach-O load commands/linked SDK 检查每个 executable 与 nested library 的 deployment target ≤12.0，拒绝只改 Info.plist 的伪兼容。

macOS producer **不得把 `.app` 目录或 raw CLI 文件直接交给 `actions/upload-artifact`**。S13 的 `scripts/desktop-transport/**` 在 producer 主机上先把每个 unsigned `.app` 封装为单文件、结构保真的 transport tar；repair producer 同一工具把 arm64/x64 raw CLI 分别封装后再上传。每个 envelope 附 canonical tree manifest 与 tree digest：按规范化 relative path 排序，逐项记录 path、`regular-file | directory | symlink`、POSIX mode；regular file 记录 size/SHA-256，symlink 记录原始 target。生成端拒绝绝对路径、空/`.`/`..` segment、NUL、路径碰撞、越界 symlink、hardlink、device/FIFO/socket、setuid/setgid/sticky 位，并显式断言 `.app/Contents/MacOS/*`、nested executable 与 repair CLI 的 executable bit。tar 的 uid/gid/mtime 等非语义元数据规范化，外层 tar 自身的 size/SHA-256 与 tree digest 一同进入 producer manifest。GitHub artifact 只运输这些单文件 envelope、manifest 与其他本来就是单文件的产物；任何接收端不得用推测性 `chmod` 或重建 symlink 修补差异。

build job 仅 `contents: read`，无 release token，输出有固定 retention 的 Actions artifacts、SBOM、bundle rights inventory、transport/tree digest 与 provenance input；禁止创建或上传 Draft Release。

先写 transport fixtures：含主 executable、nested executable、普通文件、目录与 framework 风格 symlink 的树必须经 Actions upload/download 后逐字节、逐 type/mode/target 还原；篡改 mode/hash、绝对/逃逸路径、越界链接、hardlink 与特殊文件必须在解包落盘前或空 staging 目录内 fail closed，且失败目录不可被签名阶段消费。

安装测试覆盖：无 Codex、全新用户目录、运行时断网、安装/升级/卸载、托盘、穿透恢复、autostart；Windows 另分“已装 WebView2 的离线运行”和“缺失 WebView2 时 NSIS 在线 bootstrap 后再断网运行”，MSI 文档明确其 WebView2 前置条件。WDIO 测试插件仅存在于显式 `e2e` debug artifact；候选 release artifact 必须通过无插件符号、无 test ACL、无 WebDriver HTTP 监听端口断言。产物只进入 Actions artifact，不公开 prerelease。

PR 退出：三平台 build jobs、下载后 SHA/解包 rights check 与 S13 scenario/checklist 通过后合并；最终内部 installers 的安装证据归 G13，G13 通过前 K14os 被阻塞。
回滚：保留失败构建审计，关闭 workflow dispatch；不移动 tag。

依据：[GitHub Actions artifact 的权限/链接传输限制](https://github.com/actions/upload-artifact#permission-loss)。

### K14os — Windows/macOS 签名 Profile 与服务资格门禁

**分支**：`desktop/k14os-signing-profile` 与 `desktop/k14os-evidence`；**负责人**：产品所有者、key custodian 与独立见证者。
**输入**：G13 双平台内部 installers、固定产品身份、Apple/Windows 账号、组织类型、服务区域与预算现状；不得假设证书、云签名资格或付费账号已经存在。

Windows 不把可导出的生产私钥/PFX 长期放入 GitHub。ADR 必须根据供应商的真实 eligibility/region/cost 证据冻结且只选一种后端：优先 `azure-artifact-signing-oidc`（Public Trust profile、HSM 托管私钥）；若账号或地区不满足，则选经独立安全审阅、私钥不可导出的 CA cloud/HSM signer。profile 记录 provider、region/endpoint、account/profile、预期 GitHub OIDC subject 或 runner/KMS policy、证书 subject/chain 要求、timestamp URL、固定 action/CLI SHA/version、最小 RBAC、撤销/轮换和每月预算；禁止“临时自签”冒充公开身份。此步只冻结合同与资格，不执行需要 `desktop-windows-signing` Environment 的 probe。SmartScreen 声誉不是门禁可保证的结果。

macOS profile 必须证明 Apple Developer Program 资格并冻结 `Developer ID Application`、Team ID、notary API credential 类型、hardened runtime、secure timestamp、最小 entitlements 且 `get-task-allow=false`；不用 Mac App Distribution/ad-hoc 身份。仓库此时只保存账号/Team 的非秘密资格摘要、预期证书约束和 credential 名称，不保存 `.p12`、API private key 或密码；真实 certificate fingerprint、notary submission 与 Gatekeeper probe 延后到 K14v。

`os-identities.json` 冻结两个签名/probe job 的 runner、permissions、Environment secret/variable **名称** allowlist：Windows OIDC 方案可额外拥有 `id-token: write` 且长期 secret 列表为空；remote-signer 方案列出最小 broker secret 名称与 runner/KMS 约束；macOS job 只允许列出的 certificate/notary secret 名称。任何 backend 变化都需新的 superseding K14os，不能在 G14/K14v/S14 临场切换。

退出：profile PR 已合入 protected main；资格证据证明所选 Windows trusted-signing 服务可开通，Apple Developer Program/Team 可签发 Developer ID 并可使用 notary service；provider、成本、角色、secret-name allowlist、预期 OIDC subject/runner policy 与撤销 runbook 均无 TBD。若资格或预算未落实，K14os 保持 `blocked`：S13 内部 artifact 可用，但不得通过 G14 进入公开签名链。
回滚：在任何 identity 激活前可用新 ADR supersede；激活后走 K14v 的吊销/轮换合同。

依据：[Tauri Windows signing](https://v2.tauri.app/distribute/sign/windows/)、[Microsoft Artifact Signing](https://learn.microsoft.com/en-us/azure/artifact-signing/overview)、[GitHub→Azure OIDC](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-azure)、[Apple Developer ID](https://developer.apple.com/support/developer-id/)、[Apple notarization](https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution)。

### G14 — GitHub Release Controls 管理员门禁

**操作负责人**：`g5n-dev` 仓库管理员 + 已写入 `docs/desktop/admin-roles.json` 的独立 rights/release/signing reviewers。
**输入**：K14os 已批准的 backend/identity/secret allowlist、G13 通过的 S13 run/digest、G10 readback、五份 Environment policy、不可变 desktop tag ruleset；角色文件必须是实际 GitHub login/ID，拒绝占位符、重复的 rights/release 身份和触发者自批。

G14 分成不可颠倒的 config/evidence 两阶段。第一阶段先提交无 secret 的 config PR，把五份 Environment policy、tag ruleset、管理员脚本、`updater-key-challenge.yml` 与 `os-signing-probe.yml` 合入 protected main；PR head 不能执行管理员脚本、读取 Environment 或修改外部状态。第二阶段管理员只能从该 merge SHA 运行已审阅脚本，先核对 workflow blob SHA，再配置 GitHub；所有 API/UI 回读与负向演练通过后才提交 evidence-only PR。禁止“先点 UI/跑 API，之后再补配置文件”。

管理员脚本使用 GitHub REST `2026-03-10`，对五个环境逐一执行准确的 `PUT /repos/g5n-dev/newlai_pet/environments/{name}`，payload 只使用官方 schema：named reviewer、`prevent_self_review=true`、`deployment_branch_policy={protected_branches:false,custom_branch_policies:true}`；随后逐一执行 `POST .../deployment-branch-policies`，每个环境只加入 `{name:"main",type:"branch"}`。desktop tag ruleset 为 active、无 bypass actor，分别匹配 `refs/tags/desktop-v*` 与 `refs/tags/repair-v*`，允许受控 publish 创建新 tag，但禁止删除或更新既有 tag；promotion preflight 遇到同名 tag 已存在必须失败，不能复用或移动。

```bash
# 下列五个 PUT 均使用各自 JSON 中已经解析成数字 ID 的真实 reviewer。
gh api -H 'X-GitHub-Api-Version: 2026-03-10' \
  --method PUT repos/g5n-dev/newlai_pet/environments/desktop-rights-approved \
  --input .github/environments/desktop-rights-approved.json
gh api -H 'X-GitHub-Api-Version: 2026-03-10' \
  --method PUT repos/g5n-dev/newlai_pet/environments/desktop-release-approved \
  --input .github/environments/desktop-release-approved.json
gh api -H 'X-GitHub-Api-Version: 2026-03-10' \
  --method PUT repos/g5n-dev/newlai_pet/environments/desktop-windows-signing \
  --input .github/environments/desktop-windows-signing.json
gh api -H 'X-GitHub-Api-Version: 2026-03-10' \
  --method PUT repos/g5n-dev/newlai_pet/environments/desktop-macos-signing \
  --input .github/environments/desktop-macos-signing.json
gh api -H 'X-GitHub-Api-Version: 2026-03-10' \
  --method PUT repos/g5n-dev/newlai_pet/environments/desktop-updater-signing-k1 \
  --input .github/environments/desktop-updater-signing-k1.json

# 配置脚本随后对每个环境调用同一路径下的 deployment-branch-policies；
# 仅允许 protected main，并幂等删除/拒绝任何其他 branch/tag policy。
node scripts/configure-release-controls/index.mjs --apply \
  --repo g5n-dev/newlai_pet \
  --roles docs/desktop/admin-roles.json
gh api -H 'X-GitHub-Api-Version: 2026-03-10' \
  --method POST repos/g5n-dev/newlai_pet/rulesets \
  --input .github/rulesets/desktop-tags.json
gh api -H 'X-GitHub-Api-Version: 2026-03-10' \
  --method PUT repos/g5n-dev/newlai_pet/immutable-releases
gh api -H 'X-GitHub-Api-Version: 2026-03-10' \
  repos/g5n-dev/newlai_pet/environments \
  > validation/desktop/G14/environments.json
gh api -H 'X-GitHub-Api-Version: 2026-03-10' \
  repos/g5n-dev/newlai_pet/rulesets \
  > validation/desktop/G14/rulesets.json
gh api -H 'X-GitHub-Api-Version: 2026-03-10' \
  repos/g5n-dev/newlai_pet/immutable-releases \
  > validation/desktop/G14/immutable-releases.json
node scripts/configure-release-controls/index.mjs --verify \
  --repo g5n-dev/newlai_pet \
  --evidence validation/desktop/G14
```

GitHub REST `2026-03-10` 的 Environment PUT/GET schema 当前都不提供稳定的 `can_admins_bypass` 字段，因此禁止把它塞进 payload、把未知字段被忽略当成功，或声称 GET 能回读。管理员必须在 GitHub Settings UI 逐一取消 **Allow administrators to bypass configured protection rules**；另一见证者保存含仓库、环境名、UTC 时间和完整设置状态的截图/录屏，并由环境管理员从独立浏览器会话触发一个 harmless pending deployment，证明界面不提供 bypass 且触发者不能自批。该证据进入 `validation/desktop/G14/admin-bypass/<environment>/`，每次公开 promotion 前重复 preflight；GitHub 将来若提供稳定 API，必须先按 §9 更新合同。只写 workflow 中的 `environment:` 名称不算成功，因为不存在的 Environment 会被无保护地自动创建。

`updater-key-challenge.yml` 必须在 G14 随受保护 `main` 合并，之后任何密钥仪式只能运行这份已审阅的默认分支版本。它只接受 `slot=K1|K2` 与 32-byte hex nonce，由两个硬编码 job 分别绑定 K1/K2 Environment；job 自行构造固定长度、带 domain separator 的 `NEWLAI_KEY_PROOF_V1` 消息并签名，禁止输入文件、URL、路径或任意 payload，防止把 challenge 通道变成更新包签名 oracle。K2 Environment 尚不存在时其 job 不运行。无密钥 verifier job 只用仓库公钥验证消息、签名、slot、run ID、受保护 main SHA 与 workflow blob SHA，且 workflow 运行产物不可作为仓库证据，必须经独立 evidence PR 固化。

`os-signing-probe.yml` 同样只能从 protected main 运行，且 Windows/macOS 是两个硬编码 job：各自直接绑定自己的 signing Environment，只接受仓库内固定 harmless fixture ID 与一次性 nonce，不能接受路径、URL、artifact 或任意待签 payload。G14 只安装这份尚无凭据可用的固定 workflow；真正的 identity/federation/secret 写入与 sign→verify/notary probe 全部归后继 K14v，避免 Environment 尚未存在时反向依赖 probe。

G14 创建的是 **五个空 Environment**：rights/release/Windows/macOS/K1 的 secret 与 variable 列表都必须为空，repo/org scope 也不得出现任一受控名称。Windows OIDC federation、remote-signer broker credential、Apple certificate/notary credential 都只能在 K14v 写入或激活；K1 必须等 K15a 的公钥 PR 合并后才写入。G14 只回读 Environment/branch policy/role/tag ruleset/immutable-release/workflow 身份，不用“非空”冒充配置完成。2026-09-04 只读基线为 Environment 数量 0、immutable releases `enabled=false`，因此 G14 当前明确未完成。

退出：config PR merge SHA/blob SHA 可复算；五个空 Environment、五组 main-only deployment policies、双 tag namespace ruleset、immutable releases `enabled=true`、固定 challenge/probe workflows、reviewer/防自批与全 scope 空 secret/variable 列表全部 API readback 匹配；五个环境的无 admin bypass 分别有 UI 状态证据与负向 deployment 演练，并由非操作人审阅。任一偏差即 `blocked`。
回滚：只允许通过审计变更替换 reviewer 或轮换 secret；不得删除 gate 来抢发版本。

依据：[GitHub Environments](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments)、[Repository rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository)、[Immutable releases](https://docs.github.com/en/code-security/concepts/supply-chain-security/immutable-releases)。

### K14v — 双平台发布身份激活与签名探针

**负责人**：key custodian + 独立 signing witness；非功能代码 gate。
**输入**：G14 已验证的空 Windows/macOS Environments 与 protected-main `os-signing-probe.yml`、K14os 固定 profile/allowlist。

先按 profile 激活唯一后端。Windows OIDC 路线创建的 federated credential 默认 subject 精确为 `repo:g5n-dev/newlai_pet:environment:desktop-windows-signing`，Environment 的 main-only policy 再限制执行 ref；若使用 remote signer，则 broker/KMS policy 同时固定 repo、Environment、protected-main workflow blob 与 runner identity。macOS 只向 `desktop-macos-signing` 写入 allowlist 中的 Developer ID/notary credential 名称和值。所有写入由 custodian 从工作区外执行，值不进入 repo、shell trace、artifact 或日志；随后回读 Environment secret/variable **名称集合**、OIDC subject/RBAC 或 runner/KMS policy，与 K14os exact allowlist 比较，任何多余权限都失败。

见证者从 protected main 手动运行固定 probe workflow。Windows job 只签内置 harmless EXE，要求 SHA-256 secure timestamp，并在 Win10 22H2/Win11 执行 `signtool verify /pa /all`。macOS job 对内置 arm64/x64 harmless app 与 raw CLI 使用 Developer ID、hardened runtime、secure timestamp、最小 entitlements：app 在 nested code/app 签名后用 `ditto -c -k --keepParent` 生成**仅供提交**的临时 ZIP，`notarytool submit --wait` 返回 Accepted 且日志无错误后，把 ticket staple/validate 到原 `.app`；CLI 先合成并签名 universal Mach-O，再只把它封装进 signed DMG，提交、Accepted 后 staple/validate 该 DMG。随后在 macOS 12 arm64/x64 分别运行 `codesign --verify --deep --strict`、`spctl --assess` 与断网 ticket 检查。不能直接提交 `.app` bundle，不能 staple 临时 ZIP/裸 CLI，也不能把只有 Accepted 日志但没有 stapled ticket 的最终容器当作离线验证通过。证据保存证书 public chain/fingerprint/expiry、Team ID、notary submission/log hash、timestamp、workflow/run/runner identity 和 nonce，不保存任何秘密值。另执行一次禁用 credential 的失败演练与恢复，证明撤销 runbook 可用。

退出：两平台 probe、最低系统验证、精确 scope readback、失败演练与独立签署全部进入 `validation/desktop/K14v/**` 的 evidence-only PR 并合入；缺 Windows trusted identity、Apple Developer ID、notary Accepted 或任一最小权限证据即 `blocked`，S14 不得公开 EXE/MSI/APP/DMG。
回滚：吊销/禁用尚未用于 Release 的 identity 并提交 superseding K14os/K14v；已签公开产物不可改写，只能撤回 channel 并 forward-fix。

### S14 — 受保护签名与公开 Beta

**分支**：`desktop/14-signed-beta`。**模型**：最强 + 人工安全审阅。
**输入**：K14v identity/probe evidence、G14 环境/ruleset readback、S13 workflow run ID、同一 canonical unsigned binary/bundle-input digest、真实机证据、G0b review attestation。

流水线必须分为八个受控阶段，并以 digest/manifest/inventory 显式交接：

1. `intake`：仅 `contents: read`/`actions: read`，`artifactProfile` 只允许两个硬编码值与 producer：`main-app-v1 → .github/workflows/desktop-build.yml`、`repair-cli-v1 → .github/workflows/repair-build.yml`。调用者只能给 run ID，不能给 workflow 路径；intake 从 Actions API 回读 producer workflow path/blob SHA、protected-main source SHA、artifact schema/digest、SBOM、tests 与 provenance。`main-app-v1` 必须消费已过 G13/对应后续 gate 的 `desktop-app-artifact-v1`，`repair-cli-v1` 必须消费已过 G15cR 且由 S15c2 main merge 产生的 `repair-artifact-v1`；禁止未知 producer/profile、tag 重建或把两种 schema 互换。对 macOS 输入，intake 只能接收 S13 合同产生的单文件 transport tar + canonical tree manifest，先核 outer size/SHA-256/tree digest，再在 macOS 上用拒绝绝对路径、`..`、路径碰撞、特殊文件、hardlink 与越界 symlink 的安全解包器落入空目录；逐项复算 type/mode/file hash/symlink target 集合完全相等，并再次断言主 Mach-O、nested executable 与 repair CLI 的 executable bit。任一差异立即失败，禁止 `chmod`、补链接或从源重建来“修复”producer artifact。
2. `rights_gate`：直接绑定 `desktop-rights-approved` Environment，核对 evidence、source/bundle inventory 与输入 digest；输出不可变批准记录。
3. `release_gate`：`needs: rights_gate`，直接绑定 `desktop-release-approved` Environment，由另一 reviewer 核对版本、真机证据和 rollout；签名 jobs 显式依赖两个 gate。
4. 签名阶段拆为两个可并行 job：`sign_windows` 直接绑定 `desktop-windows-signing`，`sign_macos` 直接绑定 `desktop-macos-signing`；二者基线只有 `contents: read`/`actions: read`，仅当 K14os 冻结 Windows OIDC backend 时，`sign_windows` 额外获得 `id-token: write`。授权严格来自各自 Environment 与 K14os/K14v 外部 policy。`main-app-v1` 的 Windows job 从 S13 同一 canonical unpatched PE 与 exact bundle inputs 出发，由**锁定版本的 Tauri bundler**按 package type 各复制一次：`patch(NSIS marker) → signCommand 签 patched inner → package NSIS → 签 Setup.exe`，以及 `patch(MSI marker) → signCommand 签另一 patched inner → package WiX MSI → 签 MSI`；两份 inner 字节/hash 预期不同但必须回链同一 producer digest，禁止签一个公共 inner 后再 patch，也禁止自行仿造 marker。release config 将 `beforeBundleCommand` 固定为空，sign job 禁止 Rust/前端重新编译，只运行 bundle/sign；Tauri 内置插件/uninstaller 等可执行内容也必须走同一 allowlisted signer。`sign_macos` 只使用 intake 已安全解包且 tree manifest 完全一致的 app/CLI 树，并在第一条 codesign/lipo 命令前独立复算同一合同；macOS 主应用顺序为签全部 nested code 与 `.app` → `ditto` 生成仅供 notary 提交的临时 ZIP → `notarytool submit --wait`/日志验证 → 对原 `.app` staple/validate → 从该 stapled app 生成 `.app.tar.gz` 与 DMG → 签 DMG、提交公证并 staple/validate；临时 ZIP 不发布。`repair-cli-v1` 的 Windows 顺序为签 raw EXE → 连同许可证/校验说明生成确定性 ZIP（ZIP 自身不可 Authenticode，必须验证内部 EXE）；macOS 顺序为从已验证 mode/hash 的 arm64/x64 raw Mach-O 以 `lipo` 生成 universal binary → 对 universal CLI 做 Developer ID+hardened runtime+timestamp 签名 → 生成只读 signed DMG → DMG notary Accepted、staple/validate，裸 CLI 不单独提交或 staple。签名改变字节，因此此阶段只输出待复核产物，不能自称最终 digest。
5. `final_verify`：基础 v1/repair 的 `needs` 是 `[sign_windows, sign_macos]`；启用 updater 的 v2 则还必须依赖无秘密的 `os_verify` 与 `updater_verify`。该 job 不持 secrets/写权限，重新下载签名产物，按 profile 验证内部代码与外层可签容器、公证/staple、架构/最低系统；NSIS/MSI 必须分别解包，验证各自 bundle marker、不同 patched-inner hash、inner/outer Authenticode 与共同 producer lineage。随后生成 exact `release-manifest-v1.json`，验证 SBOM 自身内容及其 manifest entry，重算 `releaseArtifactSetDigest`，执行 `bundle-inventory` 集合相等，并拒绝 e2e 符号/ACL/端口。`repair-cli-v1` 额外要求 ZIP exact allowlist、内部 Authenticode EXE、DMG 与 universal Mach-O 两个 slice 全部匹配 producer digest；禁止在 G15cR 后注入图标、背景、README 图片或其他视觉字节。启用 updater 的后续 contract 必须把 detached `.sig` 生成/验证步骤插在这次 definitive `final_verify` **之前**，使其进入 release set。
6. `signed_native_test`：`needs: final_verify`，把 final digest 对应产物送到 S10a/G10 登记的 Windows 10 22H2、Windows 11、macOS 12 arm64/x64 受控主机。`main-app-v1` 从干净用户状态真实安装 NSIS/MSI/DMG/APP，运行无 Codex/断网启动、WebView2 两路径、托盘、凝视、拖拽、下层 click-probe、穿透恢复、S10a Phase A/B autostart 登录周期、升级/卸载与 Gatekeeper/签名检查；`repair-cli-v1` 从损坏且无法启动的主应用 fixture 出发，解压签名 EXE ZIP 或挂载已 staple DMG，执行 S15c2 固定恢复/卸载场景，并在断网状态验证 Gatekeeper/签名。证据签署并绑定 `releaseArtifactSetDigest`；失败时不得 attest/publish。
7. `attest`：`needs: signed_native_test`，只有 `contents: read`、`id-token: write`、`attestations: write`，不持签名密钥，为通过真机测试的 `releaseArtifactSetDigest` 与 `release-manifest-v1.json` 文件 SHA-256 生成证明。
8. `publish`：`needs: attest`，只有 `contents: write`，不持签名密钥，只发布 manifest 精确列出的已验签 payload 加该 manifest；拒绝任意额外 asset，并在发布后逐项回读 immutable asset ID/name/size/hash 与 attestation。

`desktop-promote.yml` 只提供 protected-main 的版本化 `workflow_dispatch` contract，不暴露 `workflow_call`、`push` 或 `release` 入口；后续步骤完成无秘密 producer run 后，必须另起这次受保护 dispatch，不能在 build workflow 内继承或提升权限。dispatcher 只能提供 producer run ID、release tag、预期 producer digest 与上述 enum `artifactProfile`，不能提供 producer path，也不能关闭 gates、final_verify、signed_native_test 或 attest。合同把 profile→producer workflow→artifact schema→tag namespace→签名/测试场景全部硬编码；结果只通过本次 promotion run 的受证明 outputs/artifacts 暴露 final digest、attestation ID 与经验证 artifact IDs，供 S15a3/repair 读取。S14 必须为两种 profile 写 policy/fixture 测试；`repair-cli-v1` 在 G15cR/S15c2 前没有合法 producer run，调用必然 fail closed。

`main-app-v1` 的 “S13 run” 是 `desktop-build.yml` 在某个受保护 main SHA 上的不可变调用，不是永远复用 Alpha 3 的第一批 app 字节。首次 S14 必须消费 G13 已批准的那次 run；以后每个会改变应用/版本的 S15a3、S15r1、S15r2 都在自身 PR 合入后，从 protected main 重新调用同一固定 build contract，产生包含本次代码的 canonical unpatched PE、unsigned `.app` 的 transport envelope/tree manifest 与 exact bundle inputs，再把该 run ID/digest 交给 promotion。`intake` 校验 build workflow blob SHA、source SHA、required gates、app tree/version、tests 与 provenance；后续只从这些被结构清单固定的字节做 package-type patch/bundle/sign，不从 tag 重建、不重新编译。仅修改 release workflow 的 PR 还必须证明应用 subtree 与最近一次 native gate 测过的 tree digest 相同；否则先补新的 post-merge native gate。

`repair-cli-v1` 只能由 S15c2 合入 protected main 后运行的 `repair-build.yml` 生产；该 workflow 只编译/测试 Windows x64 与 macOS arm64/x64 raw CLI，并把两份 macOS CLI 转成 transport envelope/tree manifest，生成 `repair-artifact-v1` manifest/SBOM/digest，无 Environment、`actions: write`、签名或发布权限。它结束后，由独立的 protected-main `desktop-promote.yml` dispatch 只传该 run ID/digest，而不是由 build job 内联调用或自建第二条发布链；S15c2 无权修改 `desktop-promote.yml`。

首个 `workflow_identity` job 无 secrets/写权限，必须解析 canonical profile version，并严格断言 `github.event_name=workflow_dispatch`、`github.ref=refs/heads/main`、`github.workflow_ref=g5n-dev/newlai_pet/.github/workflows/desktop-promote.yml@refs/heads/main`、`github.workflow_sha=github.sha`；在 step 内再断言 `job.workflow_repository`、`job.workflow_file_path`、`job.workflow_ref`、`job.workflow_sha` 指向同一仓库/文件/ref/SHA，并通过 API 证明该 SHA 可达 protected main、文件 blob 与受审版本一致。上述期望值全部硬编码，不能作为 input；只有最后一条断言成功后才输出固定 `verdict=NEWLAI_PROMOTION_IDENTITY_V1_OK`。

workflow-policy 静态检查要求 identity 之外的每个 intake/gate/sign/verify/test/attest/publish job 都把 `workflow_identity` **直接**列入 `needs`，并使用精确成功条件 `if: ${{ success() && needs.workflow_identity.result == 'success' && needs.workflow_identity.outputs.verdict == 'NEWLAI_PROMOTION_IDENTITY_V1_OK' }}`。这些受保护 job 禁止 `always()`、`failure()`、`cancelled()`、`continue-on-error`、空值 fallback 或任何覆盖默认成功传播的等价表达式；只允许一个完全无 secrets、无 Environment、无 OIDC、无写权限且不被任何后继消费的 diagnostics job 用 `if: ${{ always() }}` 收集失败元数据。入口另要求 `main-app-v1` 的 input tag 等于 `desktop-v${version}`、`repair-cli-v1` 等于 `repair-v${version}`、同名 tag/release 均不存在，且 producer source SHA 与 run API 回读一致并可达 protected main；设置按 profile+version 分区的 concurrency，并在创建 draft 前重新 GET 断言 G14 已开启 immutable releases。`publish` 在单一 job 内以该 producer source SHA 为 `target_commitish` 创建 tag/draft，附齐并回读所有已验产物后一次发布，再回读 tag 指向；发布后不得再增删 asset。OS 签名 secrets 不传入 identity/intake、final_verify、signed_native_test、attest 或 publish。

公开前的 `signed_native_test` 覆盖 Windows 10 22H2/Windows 11 x64、macOS 12 arm64/x64、125/150/200% DPI、负坐标多屏、睡眠唤醒、安装/卸载、签名与 Gatekeeper/SmartScreen 状态；另在当前最新 macOS/Windows 做前瞻 smoke。Release notes 说明 Windows 新文件仍可能受声誉提示。

退出：签名链、公证/staple、最终 digest、attestation 与解包 rights inventory 全部可验证，P0/P1 为零；rights reviewer、release reviewer 与触发者三者分离。
回滚：公开 Release 保留但标记 withdrawn；发布更高 SemVer forward-fix，不替换资产/tag。

依据：[Tauri bundler 的 package-type patch/sign 顺序](https://github.com/tauri-apps/tauri/blob/270c63f117eb1f4ff0a653ca63b2ca61e9175663/crates/tauri-bundler/src/bundle.rs#L38-L187)、[Tauri 独立 bundle 命令](https://v2.tauri.app/distribute/)、[Apple notarization workflow](https://developer.apple.com/documentation/security/customizing-the-notarization-workflow)、[GitHub workflow/job identity contexts](https://docs.github.com/en/enterprise-cloud@latest/actions/reference/workflows-and-actions/contexts#job-context)。

### S15a1 — Updater Manifest 与信任核心

**分支**：`desktop/15a1-updater-core`。**模型**：强。
**输入**：S14 已签名 beta 的固定 app ID/version/target 约定；只使用测试公钥和纯 fixture。

先写纯 TS 测试：Tauri static JSON 严格超集 schema、三个精确 platform key、SemVer 单调性、channel/target/arch 严格匹配、过期 manifest、hash/长度、客户端可复算的 `artifactSetDigest` 与只作 signed-lineage 引用的 `releaseArtifactSetDigest`/`attestationId`、promotion 侧 release manifest/attestation 绑定、未知字段、重复 artifact、错误签名结果、同 generation 换内容、同 generation 同内容重试、`noUpdate` high-water mark/零 install effect、rollback 拒绝与 K1→K2 状态转换。定义版本化 channel manifest、trust state 和 `[state,effects]`；不导入 Tauri、DOM、网络、文件或私钥。

退出：属性测试保证任何未通过 target/version/hash/signature policy 的条目都不会产生 install effect；core per-file ≥80%。
回滚：revert S15a1；已发布 beta 不含 updater 行为。

### K15a — 生产 K1 信任根仪式

**分支**：`desktop/k15a-k1-public` 与 `desktop/k15a-k1-evidence`；**操作负责人**：updater key custodian + 独立见证者；密钥生成在无录屏、无 shell history 同步、无 CI 的离线工作站执行。
**输入**：S15a1 trust contract、G14 已保护但为空的 `desktop-updater-signing-k1`。

使用锁定版本的 Tauri signer 离线生成加密 K1；私钥先只写入两份受控离线介质，不进入工作区、日志、clipboard history、CI 或 repo。第一阶段只提交 `K1.pub` 与 `K1.json` 的 public PR；后者包含 algorithm、Key ID、公钥 SHA-256 fingerprint、生成工具/version、日期、custodian/见证者、两份加密备份的不可变 locator/hash 和撤销步骤。该 PR 被独立审阅并合入 protected main 后，custodian 才把 K1 写入 `desktop-updater-signing-k1` 的 `TAURI_SIGNING_PRIVATE_KEY`/password secrets。

第二阶段由见证者生成一次性 32-byte nonce，手工触发 **protected main 上已由 G14 固定** 的 `updater-key-challenge.yml` K1 job；任何 ceremony branch 或 PR head 均无权读取 Environment secret。无 secrets verifier 用 main 上的 `K1.pub` 验证 domain-separated message。下载的 run artifact、run/commit/workflow blob 身份、Environment GET 与 secret-name metadata 经哈希后进入 evidence PR；`validation/desktop/K15a/` 只保存 nonce、canonical message/hash、detached signature、verify command/result 和上述元数据，不保存私钥或密码。K15a verifier 比较 `K1.pub` 与 `K1.json` fingerprint；S15a2 再负责比较最终嵌入 app config 的公钥字节。

退出：public PR 已合入 protected main；challenge sign→verify 成功且运行的 workflow blob 正是 G14 已审阅版本；evidence PR 已由独立见证者批准并合入；公钥/Key ID/fingerprint 可复算；两份备份可解密演练；Environment 仍满足 reviewer/防自批/main-only policy，并重新执行 G14 定义的无 admin bypass UI preflight。任一条件缺失保持 `blocked`。
回滚：K1 尚未进入客户端前可销毁并重做仪式；进入客户端后只能走 K15b/S15r 轮换，不得静默替换公钥。

### S15a2 — Tauri Updater Adapter、UI 与原生升级

**分支**：`desktop/15a2-updater-adapter`。**模型**：强 + 安全审阅。
**输入**：K15a 通过、S15a1 ports、固定 K1 公钥/Key ID/fingerprint；生产私钥不可用。

NSIS 是 Windows canonical auto-update 安装形态；MSI 只用于手动/企业部署。Windows adapter 在任何网络请求前读取锁定 Tauri bundler 写入的 package-type marker，只有 `NSIS` 可进入 updater，`MSI`、unknown 或直接复制的 EXE 都返回 `updater-disabled-for-install-type` 且网络请求计数必须为零。macOS target/arch 必须精确匹配。Updater 默认 opt-in，UI 明示版本、channel、下载大小和重启；断网仍能正常启动。用本地 HTTPS fixture 与仅供测试的临时密钥先写 N→N+1、错误签名/架构、MSI 零请求、损坏包、断网、下载/安装中断及取消测试，再实现窄 Tauri adapter。

安全边界只暴露自定义 Rust commands，不给 WebView 通用 `updater:allow-*`。S15a2 的 dependency-prep 从固定上游 commit vendor `tauri-plugin-updater`，保留其 MIT/Apache-2.0 声明、记录 upstream tree SHA 与最小补丁；补丁只增加流式 `check_bounded(max_manifest_bytes)` 和同一 `Update` 上的 `download_bounded(expected_size,max_asset_bytes)`，每个 decoded chunk 在 `Vec` 扩容前用 `checked_add` 拒绝越界。产品常量首版固定 manifest ≤256 KiB、artifact ≤256 MiB，artifact 同时不得超过已签 manifest 的 `size`；无/虚假 `Content-Length`、chunked、压缩膨胀、超大首 chunk 与提前断线都必须在硬上限内失败。`configure_client(...)` **只**固定 TLS、timeout、proxy/redirect host policy，不能被描述成 body-size 防护。若受审 fork 未实现这些语义，S15a2 保持 blocked；以后迁回官方版也必须先用同一 fixtures 证明等价，不能静默换依赖。

`check_update` 从编译目标的 exact allowlist 计算唯一 `platformKey`：`windows-x86_64 | darwin-aarch64 | darwin-x86_64`，并在联网前显式调用 `.target(platformKey)`；不使用默认只返回 `windows`/`darwin` 的 `Update.target` 语义，也不允许 `{os}-{arch}-{installer}` fallback。随后设置 comparator 只负责让已被 Tauri 解析的 static manifest 暴露给自定义策略，不授权 rollback；以固定 raw endpoint 的 `check_bounded(256 KiB)` **单次**拉取，把返回的 `Update.raw_json` 交给纯 core 做策略判断。Rust 端独立验证 canonical manifest signature、status、key/channel/time/generation/SemVer，重算 `artifactSetDigest`，并要求 `Update.target == platformKey`、`platforms[platformKey]` 与 `Update.{download_url,signature}` 逐字段相等；`releaseArtifactSetDigest`/`attestationId` 只做格式、签名覆盖和 lineage 持久化，不谎称客户端已复算/取回验证。合法 `noUpdate` 只提交 generation/version high-water mark 并结束。对 `update`，Rust 把原 `Update`、platformKey 与 manifest SHA-256 存入内存 opaque token。`install_update(token, expectedHash)` 只能消费同一个对象，不再请求 manifest；调用该对象的 `download_bounded(signedSize, 256 MiB)`，在硬限内完成 Tauri artifact signature 验证后，再核 length/SHA-256/`artifactSetDigest` 才调用 `Update::install(bytes)`。token 单次、短时、重启失效；last accepted generation 只在整份 manifest 与选中 target 均通过后持久化。锁定依赖的合约测试还必须证明附加字段保留在 `raw_json`、Tauri 必需字段完整、MSI/unknown 零网络、custom target exact match、字段不一致、二次使用、token substitution、redirect 越界及 `status=update` 的低 SemVer 即使 comparator 放行也全部 fail closed；不可降级为 stock 无界 API、双重 manifest fetch，或额外拉取 release manifest/attestation 后仍宣称单次 fetch。

生产 `tauri build` 在此及 S14 均不得获得 `TAURI_SIGNING_PRIVATE_KEY`；只嵌入公钥。release isolation 检查 secrets 名、日志和产物中无私钥材料。

PR 退出：测试 feed 的 unit/WebView/integration、release isolation 与 S15a2 scenario/checklist 通过后合并；Windows 10 22H2/11 NSIS 与 macOS 12 arm64/x64 的真实 N→N+1 证据归 G15a2，G15a2 通过前 S15b 被阻塞。
回滚：远端 feed 为空且本地 `updaterEnabled=false`，随后 revert S15a2。

依据：[Tauri updater](https://v2.tauri.app/plugin/updater/)、[`Update.raw_json`/`download`/`install`](https://docs.rs/tauri-plugin-updater/latest/tauri_plugin_updater/struct.Update.html) 与[受审上游源码基线（默认 target lookup、无界 manifest/artifact 收集）](https://github.com/tauri-apps/plugins-workspace/blob/a21555ddd2eaadfed23848912fe2802c2ba7579e/plugins/updater/src/updater.rs)。

### S15b — 启动健康、Crash-loop 与设置备份

**分支**：`desktop/15b-health-recovery`。**模型**：强。
**输入**：G15a2 通过的 update result、S8 版本化设置。

先写状态机测试：启动前写 pending、达到健康窗口后 commit；连续 3 次在 30 秒内失败进入 safe mode；正常退出不计 crash；睡眠/强制关机不误判；设置迁移前原子备份，成功后保留最近 3 份，损坏恢复不覆盖唯一好副本。

实现本地健康标记、safe mode（禁用 autostart/凝视/环境动作/穿透，仍保留托盘与重置）和设置备份。不得自动安装未验签旧二进制。

PR 退出：health/backup unit、Rust/WebView 与 S15b scenario/checklist 通过后合并；两平台 crash/safe-mode/恢复证据归 G15h，G15h 通过前 S15a3 被阻塞。
回滚：忽略新健康字段仍可按旧版本启动；保留备份，revert S15b。

### S15a3 — Canary Channel 与隔离 K1 签名

**分支**：`desktop/15a3-canary-channel`。**模型**：最强 + 人工密钥审查。
**输入**：G15h 通过的可恢复客户端与被测 app tree digest、K15a、G14 五个受保护 Environment、S13 build/S14 promotion contracts。

此步把 S14 promotion contract 从 v1 升为 v2，而不建立旁路。`main-app` 的 immutable artifact 链固定为 rights→release→OS sign→`os_verify`→`updater_sign_artifacts`→`updater_verify`→definitive `final_verify`/release manifest→signed-native artifact/update-fixture test→attest→publish prerelease；这样三份 detached `.sig` 在计算 `releaseArtifactSetDigest` 前已经产生并验证。取得不可变 Release asset IDs 后，activation 链再执行 `channel_sign_k1`→`channel_verify`→四类主机 public-URL N→N+1→channel/evidence PR。自动更新客户端在最后一个 PR 合入前看不到该版本；若 public-URL 测试失败，只把尚未进入 channel 的 prerelease 标记 withdrawn。`repair` profile 禁用所有 updater/channel 步骤但不能禁用基础 gate。

PR 合入且 verifier 证明 app subtree 未越过 G15h digest 后，从该 merge SHA 调用 S13 build contract；build run 结束后再独立 dispatch protected-main promotion，而不是让 build job 继承权限。`updater_sign_artifacts` 与 `channel_sign_k1` 分别在需要时直接绑定 `desktop-updater-signing-k1`，且只获得 K1 私钥；它们不重新 build、不获得 Windows/Apple 凭据。前者用 `tauri signer sign` 对已通过 OS 签名的 NSIS 与两份 `.app.tar.gz` 生成三个 detached artifact signatures；后者只能按 §3.6 从已发布的 immutable asset IDs、`artifactSetDigest`、`releaseArtifactSetDigest` 与 attestation ID 构造并签 canary manifest。两个无密钥 verifier 分别核对 OS 签名/公证、artifact signature、两个 set digest、manifest signature、hash、target 和 rights inventory。

发布事务顺序：上传内容寻址、不可变 artifacts/signatures → 下载重算 hash 并双重验签 → 对各 target 解包复核 → 核对 `release-manifest-v1.json`、两个 set digest 与 attestation → 发布 immutable GitHub prerelease → 最后按 §3.6 合入包含 K1 artifact signatures 的 **K1 canary manifest**。此步无权写 beta/stable manifest；任何失败都不得留下指向半套 artifacts 的 canary。

退出：测试覆盖 canary 原子性、紧急停发、K1 challenge 回归和无旁路发布；OS code-signing 与 K1 updater-signing jobs/Environments/审阅者/日志完全分离。
回滚：原子停发 manifest 只阻止新更新；已安装版本通过更高 SemVer forward-fix，不声称自动降级。

### G15 — 7 天 Canary 运营门禁

**负责人**：release reviewer；不是代码 PR。
**输入**：S15a3 发布的 canary manifest、artifact/final digest、签名与每次更新结果。

连续 7 天、至少 6 个独立测试主机，覆盖 Win10 22H2 x64、Win11 x64、macOS 12 arm64、macOS 12 x64；至少 100 次更新且每类平台 ≥20 次，成功率 ≥98%；至少 200 个受控会话，crash-free ≥99.5%；P0/P1=0。任一签名失败、任一 P0/P1 或滚动更新失败率 >2% 自动停止 channel 推进。数据只来自实验室/明确 opt-in 测试，不改变默认离线与隐私承诺。

退出：`validation/desktop/G15/rollout-decision.json` 绑定 canary final digest、样本明细、统计窗口和 reviewer 签署结论；不达标保持 `blocked`，不得改阈值追认。
回滚：保持 beta/stable manifest 不变，canary 停发，发布更高版本修复后重新完整计时。

### S15a4 — Canary → Beta 同字节晋级

**操作负责人**：release reviewer；受保护 deployment，不是新 build。
**输入**：G15 签署的 `rollout-decision.json`、canary final digest 与 immutable artifact IDs。

`promote_channel` gate 先绑定 `desktop-release-approved`，验证决策签名、统计窗口、artifact/OS/updater signatures、两个 set digest 与 attestation；随后 `channel_sign_k1` 单独绑定 K1 Environment，只从这些已验证输入构造更高 generation 的 beta manifest 并签名。channel/evidence PR 再把 **同一组 artifact 字节** 原子晋级到 beta。禁止 rebuild、重签 artifact、复制后变更或同时写 stable；新增 channel manifest signature 不改变两个既有 set digest。

退出：重新下载 beta 指向的每个 target，其 hash 分别等于 release manifest 对应 entry，且整份 `artifactSetDigest`/`releaseArtifactSetDigest` 与 G15 完全相同；`validation/desktop/S15a4/` 保存 manifest before/after、deployment ID 与 verifier 结果。
回滚：原子移除 beta 新指针并保持旧 stable；已安装版本仍只接受更高 SemVer。

### G15b — 14 天 Beta 运营门禁

**负责人**：另一名 release reviewer；不是代码 PR。
**输入**：S15a4 beta manifest 与同一 canary final digest。

连续 14 天；至少 10 个独立主机覆盖四类平台；至少 250 次更新且每类 ≥50 次，成功率 ≥99%；至少 500 个受控/明确 opt-in 会话，crash-free ≥99.5%；签名错误=0、P0/P1=0。任一签名失败、P0/P1 或滚动更新失败率 >1% 自动阻止 stable。

退出：`validation/desktop/G15b/rollout-decision.json` 绑定与 G15 完全相同的 artifact digest、样本与独立 reviewer 签署；不达标保持 beta 或撤回，不得换字节后沿用旧统计。
回滚：stable manifest 保持原值，修复版从 canary 重新开始完整周期。

### S15a5 — Beta → Stable 同字节晋级

**操作负责人**：release reviewer；受保护 deployment。
**输入**：G15b 签署决策、beta manifest 与相同 final digest。

同 S15a4 的 release approval→`channel_sign_k1`→无密钥 verifier→channel/evidence PR，且 stable manifest 只允许指向已通过 G15/G15b 的同一 immutable artifact IDs。任何 digest、target、artifact/manifest signature 或 rollout decision 不一致均 fail-closed。

退出：stable manifest 下载回读、双签名与 final digest 全部匹配；证据写 `validation/desktop/S15a5/`。
回滚：停止新 stable 提示并恢复旧 manifest 指针；对已安装坏版本发布更高 SemVer forward-fix。

### A15c — Repair Tool ADR

**分支**：`desktop/a15c-repair-adr`。**模型**：强 + 人工安全审阅。
**输入**：S15a5 stable 结论、S15b safe-mode/backup contract、两平台安装路径与签名身份。

ADR 记录而不是重选以下冻结决策：零 WebView、零网络、永不提权的 Rust CLI；Windows 名为 `newlai-pet-repair.exe`，以含 Authenticode-signed EXE 的 ZIP 分发；macOS 名为 `newlai-pet-repair`，将 arm64/x64 合成 Developer ID-signed universal CLI 并放入 signed/notarized/stapled DMG。两者均为独立 artifact，不随主安装器捆绑、不自更新；选择可 staple 的 DMG 而不是把 macOS ZIP 冒充离线可验证容器。工具只访问平台 API 返回的当前用户 app-data root（Windows Known Folder LocalAppData；macOS Application Support + bundle ID），禁止接受自定义 root。

命令固定为 `status`、`doctor`、`backup --output <用户明确选择的新文件>`、`reset-settings`、`disable-autostart`、`launch-safe-mode`。修改前展示解析后的目标并要求交互确认或显式 `--yes`；只对 bundle ID/签名/路径均匹配的宠物进程先请求正常退出，再有限时终止。exit codes 固定：0 成功、2 用法错误、3 未安装、4 权限不足、5 部分完成、10 完整性失败。

退出：`docs/desktop/adr/repair-tool.md` 无 TBD，列出上述 threat model、精确平台 paths/commands/exit codes、独立交付/签名/卸载合同和回滚；独立 reviewer 批准。若未来要 GUI、提权、随主安装器捆绑或换 runtime，必须先按 §9 supersede A15c 并修改唯一表/DAG/后继 owned paths，不能让 S15c1 临场选择。
回滚：实现前可用完整新 ADR supersede；实现后只能迁移或另建工具。

### S15c1 — 外部修复工具实现

**分支**：`desktop/15c1-repair-implementation`。**模型**：强。
**输入**：已批准 A15c，不得在 PR 内重选架构。

先写测试并实现 ADR 限定功能：停止宠物进程、备份/重置设置、关闭 autostart、启动主应用 safe mode；所有路径先解析到固定 app data/install roots，不接受任意命令或 URL。覆盖损坏主应用/设置、锁文件残留、权限不足、旧/新 schema、symlink/path traversal 和恶意参数。

退出：无 WebView/主应用依赖的 binary 在 Windows/macOS fixture 完成恢复；单元/集成覆盖率 ≥80%，无管理员权限需求。
回滚：revert S15c1；用户备份不删除。

### G15cR — Repair Rights Amendment

**分支**：`desktop/g15cr-repair-rights`。**模型**：默认 + 独立 rights reviewer。
**输入**：S15c1 最终 binary/resource 需求、G0b/S12 rights schema；实现者不能自批。

首发 repair profile 固定为无自定义视觉素材的文字型 CLI：`repair/assets/**` 必须为空，README/打包说明不得含截图、data URL、字体、背景或图标，只允许纯文本许可证/说明与系统默认容器外观。独立 reviewer 对 source/resource inventory 做空集证明，并把 repair profile amendment 写入 rights manifest；S15c2 后再跑 `bundle-inventory`，拒绝 symlink、内嵌视觉字节和未列出转码。未来若要加入任何视觉素材，必须先用新的 G15cR amendment 提供 clean-room 来源链或明确授权、exact SHA-256、purpose 与 licenseRef，不能把它塞进既有 S15c2。

退出：rights CODEOWNER 批准 amendment，机器 verifier 通过，`validation/desktop/G15cR/` 保存 review attestation/hash；失败则 S15c2 blocked。
回滚：revert amendment 和未发布 repair assets；不改主应用既有权利记录。

### S15c2 — Repair 双平台签名、打包与发布

**分支**：`desktop/15c2-repair-package`。**模型**：强 + 人工安全审阅。
**输入**：G15cR 通过的 S15c1 binary/resources、G14 环境/ruleset、S14 可复用 promotion contract。

`repair-build.yml` 只在 S15c2 PR 合入 protected main 后以无 secrets、`contents: read` 权限编译/测试 Windows x64、macOS arm64/x64 raw binaries，输出固定 `repair-artifact-v1` manifest/SBOM/digest；Windows raw EXE 可作为单文件输入，macOS 的 arm64/x64 raw CLI 必须在各自 producer 主机上用 S13 固定的 `scripts/desktop-transport/**` 分别封装为结构保真 transport tar，并生成 canonical tree manifest/outer digest 后才上传。它自己不签名、不创建 tag/Release，也不拥有 `actions: write`。build run 完成后另行 dispatch protected-main `desktop-promote.yml`，只把该 run ID/digest 与 G15cR attestation 交给 S14 已冻结的 `repair-cli-v1` profile，复用 transport/tree exact 验证→rights_gate→release_gate→`sign_windows`/`sign_macos`→final_verify→signed_native_test→attest→publish 链。签名 jobs 直接绑定原有 Windows/macOS signing Environments，其他 jobs 无 OS 私钥，不另建仓库 secret 或旁路 Release，S15c2 也不得修改 `desktop-promote.yml`。

Windows 最终 artifact 是确定性 ZIP，内部 EXE 的 Authenticode/timestamp、外层 SHA-256 与 exact file allowlist 都要验证；macOS 最终 artifact 是含 universal signed CLI 的 DMG，要求两个 Mach-O slice、deployment target、Developer ID、notary Accepted 与 staple 全部通过。G15cR 之后只允许加入机器生成的 manifest/SBOM/checksum 与纯文本许可说明，禁止新增或转码任何图标、背景、截图、字体或其他视觉字节；两平台公开 payload 连同 SBOM/许可条目共同进入一个 `releaseArtifactSetDigest` 与 attestation，不为每个平台伪造互不关联的“final digest”。

真实测试从主应用无法启动开始，Windows 从 ZIP 解压签名 EXE，macOS 在断网状态挂载已 staple DMG，分别执行设置备份、关闭 autostart、safe-mode 恢复及卸载；证据绑定最终签名字节。
退出：两平台验签/公证、恢复、清理全部通过；`repair-v*` GitHub Release 只含已走完整受保护链的不可变 artifact。
回滚：标记该修复器版本 withdrawn 并发更高版本；不删除用户备份或覆盖旧公开资产。

### K15b — 可选 K2 轮换密钥仪式

**分支**：`desktop/k15b-k2-public` 与 `desktop/k15b-k2-evidence`；**操作负责人**：不同于 K1 日常操作者的 key custodian + 独立见证者。
**输入**：S15a5 的稳定 K1 更新链、批准的 rotation reason/date。

复用 K15a 两阶段流程离线生成 K2。先把 K2 public metadata、K2 Environment policy 和角色变更作为无 secrets PR 合入 protected main；管理员再用 G14 已固定的配置脚本创建独立 `desktop-updater-signing-k2` Environment，GET 回读 named reviewer、prevent-self-review 与 main-only policy，并按 G14 的 UI 证据+负向演练确认无 admin bypass，最后才写入 K2 secret。见证者从 protected main 运行 G14 固定的 K2 challenge job，第二个 evidence PR 固化 workflow blob SHA、run、domain-separated message、signature 与 verify evidence。验证器强制 K1≠K2；K1/K2 使用硬编码的不同 Environment jobs，任何 job 不同时读取两把私钥。

退出：public/config PR 与 evidence PR 均已合入；K2 challenge 从固定 main workflow 通过；两份备份演练、Environment/secret metadata readback 与独立签署齐全。K1 公钥、bridge artifact 和旧 feed 不删除。
回滚：K2 尚未进入 bridge 客户端前可销毁重做；进入后必须完成或 forward-fix rotation train。

### S15r1 — K1 签名的 K2 Bridge Client

**分支**：`desktop/15r1-k2-bridge-client`。**模型**：最强 + 密钥审阅。
**输入**：K15b 的 K2 pubkey/fingerprint、稳定 K1 客户端与 S15a3 promotion contract。

单独 PR 把 bridge 版本的内嵌 trust root/目标 feed 改为 K2，并拥有相应 updater config/组合根；bridge 安装包仍经完整 OS 签名链，detached updater signature **只由 K1 Environment** 生成，使旧 K1 客户端能够安装。安装后的 bridge 只查询 K2 feed、只信任 K2。先写 K1-client→bridge→K2-fixture、错误 K2、重放和跨架构测试；release isolation 证明 bridge job 无 K2 私钥。PR 合入后，从该 merge SHA 调用 S13 build contract，再用 S14/S15a3 的固定 promotion contract 生成更高 SemVer、双平台签名/公证/真机验证/attest 的 immutable bridge，并且只写 K1 bridge canary manifest。

退出：K1 bridge canary 下载回读后，OS 签名、公证、K1 updater signature、rights inventory、target 与 final digest 全部匹配；此步无权写 bridge beta/stable。
回滚：移除 bridge canary 指针并保留旧 K1 stable；如已有客户端安装 bridge，只能在 K2 feed 发更高版本修复。

### G15r1c — 7 天 Bridge Canary 门禁

**负责人**：release reviewer；非代码 gate。**输入**：S15r1 bridge canary 与独立 final digest。

为 bridge 独立重新计时：连续 7 天、至少 6 个主机覆盖 Win10 22H2 x64、Win11 x64、macOS 12 arm64/x64；至少 100 次 K1→bridge 更新且每类 ≥20、成功率 ≥98%；至少 200 个受控会话、crash-free ≥99.5%；签名错误/P0/P1=0。退出决策绑定 bridge digest 与完整样本，不能复用初始 K1 G15 证据。

### S15r1b — Bridge Canary → Beta 同字节晋级

**操作负责人**：release reviewer；受保护 deployment。**输入**：G15r1c 决策、bridge canary immutable artifact IDs/final digest。

复用 S15a4 的无 build、无 artifact re-sign 晋级器，经 release approval 与 `channel_sign_k1` 生成更高 generation 的 K1 bridge beta manifest，只指向同一组字节；下载回读与证据进入 `validation/desktop/S15r1b/`。失败则 beta/stable 保持原值。

### G15r1b — 14 天 Bridge Beta 门禁

**负责人**：另一名 release reviewer；非代码 gate。**输入**：S15r1b bridge beta 与同一 digest。

连续 14 天、至少 10 个主机覆盖四类平台；至少 250 次更新且每类 ≥50、成功率 ≥99%；至少 500 个会话、crash-free ≥99.5%；签名错误/P0/P1=0。独立决策绑定同一 bridge digest，不能复用 G15b 或 G15r1c 的统计。

### S15r1s — Bridge Beta → Stable 同字节晋级

**操作负责人**：release reviewer；受保护 deployment。**输入**：G15r1b 决策、bridge beta immutable artifact IDs/final digest。

复用 S15a5 的 release approval→`channel_sign_k1`→verifier→channel/evidence PR，把 K1 stable 原子指向同一 bridge 字节并重新下载验签；证据进入 `validation/desktop/S15r1s/`。K2 feed 此时仍为空或保持旧值。

### G15r — K2 Rotation 观察门禁

**负责人**：key custodian + release reviewer；非代码 gate。
**输入**：S15r1s bridge stable final digest、K1 feed/签名、K2 challenge 与 G15r1c/G15r1b bridge rollout 证据。

从 bridge stable 起至少保留 K1 feed 180 天；四类受控主机 100% 完成 K1→bridge→K2 fixture，明确 opt-in updater 客户中 bridge 覆盖率 ≥99%，签名错误/P0/P1=0。无法测量的离线客户端不作为删除旧 feed 的理由：K1 feed 与 bridge artifact 继续长期只读保留。

退出：rotation decision 绑定 K1/K2 fingerprints、bridge digest、起止日期、覆盖数据和两名 reviewer；任一阈值未达则继续 K1，不切换。
回滚：不触碰 K2 stable feed；保持 K1 正常服务 bridge。

### S15r2 — 切换 K2 Feed

**分支**：`desktop/15r2-k2-release-path`；**操作负责人**：release reviewer；PR + 受保护 deployment。
**输入**：G15r 签署 decision、既有 bridge/K1 feed、K2 Environment。

PR 只增加已冻结 contract 下的 K2 硬编码签名 job、隔离验证、版本提升和对应 scenario/checklist；PR head 不读取任何 Environment secret。合入 protected main 后，从该 merge SHA 调用 S13 build contract，build 完成后另行 dispatch 固定 promotion chain 消费这次 run 的 canonical unsigned inputs 并生成更高 SemVer：rights/release→OS sign/notarize→`os_verify`→只绑定 `desktop-updater-signing-k2` 的 job 生成 NSIS/两份 `.app.tar.gz` detached signatures→`updater_verify`→definitive `final_verify`/release manifest；无密钥验证和四类主机 update fixture 通过后才 attest/publish immutable prerelease。取得 Release asset IDs 后，独立的 `channel_sign_k2` 以两个 set digest 与 attestation ID 构造并签 K2 canary manifest；无密钥 verifier 与四类主机 public-URL N→N+1 通过，最后才用 channel/evidence PR 原子写 K2 canary。K1 feed 永远只指向原 bridge artifact，不用 K2 覆盖、不要求旧 K1 客户端直接验证 K2；任何 job 不同时持 OS/updater 或 K1/K2 secrets。

退出：K1 客户端可达 bridge、bridge 可发现并安装 K2 canary；两条 feed/artifact 均可独立重放验证；K2 canary 的最终字节已通过四类平台真实安装/更新；此步无权写 K2 beta/stable。
回滚：停止 K2 manifest 前进，bridge 继续查询上一个 K2 good version；K1 bridge 路径保持可用。

### G15r2c — 7 天 K2 Canary 门禁

**负责人**：release reviewer；非代码 gate。**输入**：S15r2 K2 canary 与独立 final digest。

按 G15 的固定数字为 K2 重新采样：连续 7 天、≥6 主机、四类平台、≥100 次更新且每类 ≥20、成功率 ≥98%、≥200 会话、crash-free ≥99.5%、签名错误/P0/P1=0。决策只绑定本次 K2 digest，K1 或 bridge 阶段的证据无效。

### S15r2b — K2 Canary → Beta 同字节晋级

**操作负责人**：release reviewer；受保护 deployment。**输入**：G15r2c 决策、K2 canary immutable artifact IDs/final digest。

无 build、无 artifact re-sign，经 release approval 与 `channel_sign_k2` 生成更高 generation 的 K2 beta manifest，再原子晋级同一字节；重新下载核对 OS/K2 artifact+manifest signatures、target 与 final digest，证据进入 `validation/desktop/S15r2b/`。

### G15r2b — 14 天 K2 Beta 门禁

**负责人**：另一名 release reviewer；非代码 gate。**输入**：S15r2b K2 beta 与同一 digest。

连续 14 天、≥10 主机、四类平台、≥250 次更新且每类 ≥50、成功率 ≥99%、≥500 会话、crash-free ≥99.5%、签名错误/P0/P1=0。独立决策绑定同一 K2 digest；不能复用任何先前 rollout 统计。

### S15r2s — K2 Beta → Stable 同字节晋级

**操作负责人**：release reviewer；受保护 deployment。**输入**：G15r2b 决策、K2 beta immutable artifact IDs/final digest。

无 build、无 artifact re-sign，经 release approval 与 `channel_sign_k2` 生成更高 generation 的 K2 stable manifest，再把 stable 原子指向同一字节并下载回读；证据进入 `validation/desktop/S15r2s/`。K1 feed、K1 公钥与 immutable bridge 继续长期只读保留，离线旧客户端不会被强迫直接跨越信任根。

退出：K2 stable 的 manifest、artifact IDs、OS/K2 signatures 与 G15r2c/G15r2b 决策全部绑定同一 final digest。
回滚：停止 K2 stable 提示、恢复前一个 K2 manifest；已安装版本只用更高 SemVer forward-fix，K1 bridge 路径保持不变。

## 7. 全局验收命令与证据

每个实现 PR：

```bash
cd desktop
npm ci
npm run format:check
npm run lint
npm run typecheck
npm run test:coverage
npm run test:renderer-contract
cargo test --locked --manifest-path src-tauri/Cargo.toml
npm run verify:release-isolation
```

涉及原生适配的功能 PR 运行 WebView suite；合并 protected main 后，由对应 G gate 运行 OS suite：

```bash
# GitHub 原生 Windows/macOS runner；仅 debug/test + Cargo e2e feature
npm run test:native:webview

# 仅 post-merge G gate；有真实登录桌面会话的受控 Windows/macOS 主机
npm run test:native:os -- --require-real-session
```

`test:native:webview` 只覆盖真实 WebView/IPC。`test:native:os` 只能由 S10a 固定 workflow 从 protected main 调用，必须覆盖平台 UI 自动化、下层 click-probe、全局快捷键、第二实例；凡要求真实 logout/login 或 reboot 的场景必须展开成不同 run-scoped ephemeral label 的 Phase A/Phase B 两个 job，controller 在确认 A registration 已注销后才重启，并只在新 boot/session+witness ready 后注册 B。禁止假设中断 step 会续跑，或用同一 persistent label 的普通 `needs` 充当屏障。无法自动化的系统提示由独立见证者按版本化 checklist 记录，不能由开发者勾选代替。证据写入：

```text
validation/desktop/<step>/<os>-<arch>/
├── environment.json
├── result.json
├── logs/
└── screenshots/
```

`environment.json` 至少记录 OS build、架构、真实/虚拟主机类型、host/boot/login-session ID、controller/UI-witness hash、显示器布局、DPI/scale、app commit、artifact SHA、测试 feature 和 WebDriver isolation 状态；`result.json` 分别记录 webview、probe、tray/shortcut、Phase A/B ticket、login-cycle 和见证者。公开发布必须从 GitHub 下载产物后重算 SHA、验签、检查无 e2e 端口/ACL/符号，并解包运行 rights verifier；checksum 一致不等于 bit-for-bit reproducible，不作该声称。

## 8. 反模式清单

- 用仓库内一个 `true` 或实现者自批替代外部授权证据。
- 把某次 GitHub Environment job 的批准当成永久权利批准，或用单个 Environment 冒充两个独立角色。
- 只在 workflow 写一个尚不存在的 Environment 名称，任其自动创建为无 reviewer 的空门禁。
- 在 G14 创建并保护签名 Environment 之前运行需要该 Environment 的签名 probe，形成“probe 通过才建环境、建环境又依赖 probe”的循环。
- 只扫描固定 atlas 路径，漏掉图标、预览、营销图、转码或 base64 内嵌。
- 在 core 前按方向去重，丢失距离、活动时间或窗口移动。
- 把同方向采样当静止，或没有 heartbeat 却要求 1000ms 停止检测。
- 把当前 8×11/192×208 固化进资产无关核心。
- 构造跨混合 DPI 屏幕的全局 DIP 平面。
- 同时使用 window-state 与自定义 normalized placement。
- 未确认托盘和快捷键成功就开启穿透，或持久化穿透状态。
- 把 Playwright/WDIO 的 WebView 结果称为托盘、穿透、autostart 等 OS 级 E2E。
- 让 embedded WebDriver 插件、HTTP 端口或测试 ACL 进入 release profile。
- 在 self-hosted OS runner 上 checkout/执行未合并 PR 脚本，或让下一步骤绕过失败的 post-merge native gate。
- 首次 Windows 真机验证拖到发布阶段。
- 两个“并行”分支共同修改 lockfile/config。
- 使用浮动 Action tag，让 build/sign job 持有 Release 写权限，或让 publish job 接触签名私钥。
- 只签 Windows installer 外壳而不签 inner EXE，或在签名前生成所谓最终 digest。
- 先签一个公共 Windows inner EXE，再让 Tauri 为 NSIS/MSI 写 package-type marker；patch 会破坏签名且两个 inner 本就不是同一字节。
- 用 updater 的三目标 `artifactSetDigest` 冒充覆盖 MSI/DMG/SBOM/许可/`.sig` 的完整 Release 摘要。
- 客户端只拿到三个 platform entry，却声称已独立复算完整 `releaseArtifactSetDigest` 或验证 GitHub attestation；它只能验证签名覆盖这些 lineage 引用。
- 把 macOS `.app` 目录或 raw CLI 直接上传 Actions artifact，丢失 executable mode/符号链接后再用 `chmod` 或重建链接猜测性修补；必须先传单文件结构保真 envelope，并对 canonical tree manifest 做 exact 验证。
- 用 `contents: read` job 创建 Draft Release。
- 把未签名公开 alpha 伪装成普通安装体验。
- 把 MSI 放进 NSIS updater feed，或混用 OS code-signing key 与 updater key。
- 只说 MSI “不进 feed”却仍让 MSI 安装来源联网并 fallback 到通用 Windows key。
- 在持有 OS 签名凭据的 `tauri build` 中注入 updater 私钥，而不是对既有 OS 签名 artifact 做隔离 detached signing。
- 声称 `configure_client` 能限制 response body，或在无 `Content-Length`/chunked/压缩膨胀时先无限收集再做后验 size 检查。
- 把默认值为 `windows`/`darwin` 的 `Update.target` 与 `windows-x86_64`/`darwin-*` static key 直接比较。
- 把裸 `.app` 上传给 notary service，或尝试 staple 临时 ZIP/裸 CLI。
- 让一个 Actions step 跨 logout/reboot 续跑，而没有同机 ticket、持久 controller 和重连后的 Phase B。
- 让 Phase A/B 共用 persistent runner label，使 B 在 A 注销与真正重启之间被抢先派发；必须用不同的 run-scoped one-shot registrations。
- 在 reusable workflow 内只用属于 caller 的 `github.workflow_ref` 证明 called workflow 身份；本计划直接禁用 promotion 的 `workflow_call` 入口。
- 只给权限 job 添加 `needs: workflow_identity`，却允许 `always()`/`failure()`/`continue-on-error` 绕过失败传播。
- 没有可复算公钥 fingerprint/challenge 证明就声称 Environment secret 是正确 K1/K2。
- 先写 beta/stable manifest 再补 canary 统计，或换了 artifact 字节却沿用旧 rollout decision。
- 让 updater 或 repair 建立绕过 rights/release/sign/verify/native-test/attest 的第二条发布通道。
- 把停止 updater endpoint 描述成已安装二进制回滚。
- 使用“明显”“可接受”“应该没问题”作为退出条件。

## 9. 计划变更协议

计划状态：`draft → reviewed → active → superseded | complete`。负责人唯一为仓库维护者；每次变更追加新记录，不改写已完成步骤的历史结论。

每条变更必须写：计划版本、日期、触发证据、受影响步骤、依赖表变化、在途分支处理、回滚影响、批准角色。

- **拆分**：步骤跨超过两个架构层或无法一个 PR 完成时必须拆分。
- **插入**：许可、安全、数据丢失或供应链风险可插入阻塞 gate。
- **并行**：只有 owned paths 无交集、无输出依赖、可独立满足退出条件三项同时成立。
- **重排**：先更新唯一依赖表并验证 DAG 无环，再重排。
- **在途分支**：依赖或契约变化时明确 `rebase / cancel / supersede`；不得默默继续。
- **已完成步骤**：只能新增 superseding step 或 revert，不能原地重写事实。
- **并行日志**：各分支写自己的 `validation/desktop/<ID>/`，避免同时追加本文件。

## 10. 变更记录

- `0.1`（2026-09-04）：初稿，选择 Tauri 2 并设置基础权利门禁。
- `0.2`（2026-09-04）：根据对抗审查修订。G0 改为人工、受保护、全资产 fail-closed 门禁；指针改为含时间与 motion sequence 的持续采样；拆分过大 PR；建立唯一依赖表/owned paths；移除 window-state；区分 GlobalPhysicalPx 与 MonitorLocalDip；增加穿透失败闭锁、原生 E2E、供应链最小权限、数值性能预算、签名 promotion 与 updater forward-fix 模型。
- `0.3`（2026-09-04）：第二轮对抗审查后修订。拆分 G0a/G0b、S9a/S9b、S15a/S15b/S15c；将 S10 设为 S2 合并硬前置；明确组合根串行所有权、三阶段权利验证、WebView/OS 两类原生测试及生产 WebDriver 隔离；固化 Windows/macOS 内外层签名顺序、签后 digest、双 Environment、NSIS updater、K1→K2 桥接与数值 canary 门禁。
- `0.4`（2026-09-04）：第三轮对抗审查后修订。拆出 S10a/G10 与 G14 管理员门禁，记录当前 main 未保护且 Environment 为零的真实基线；为 CI/native harness 指定 workflow、runner、check context 与 readback；S14 只消费 S13 同一 inner bytes，并在签名后跑绑定 final digest 的真实安装旅程；签名 jobs 直接绑定各自 Environment；updater 拆为 core/adapter/health/channel 与独立 G15，使用 OS 签名后再由隔离 updater Environment 生成 detached signature；repair 增加 ADR、实现、受保护打包三阶段。
- `0.5`（2026-09-04）：第四轮对抗审查后修订。所有 OS 行为改为 protected-main 的 post-merge G5/G6/G7/G9a/G9b/G11/G13/G15a2/G15h 门禁，各功能步骤独占自己的 scenario/checklist；S15a3 只能发布 canary，G15/S15a4/G15b/S15a5 强制以同一字节逐级晋升；加入 K15a 生产 K1 仪式及独立 K15b/S15r K2 轮换链；A15c 冻结为无网络/无提权 Rust CLI，并新增 G15cR repair rights amendment。
- `0.6`（2026-09-04）：最终对抗审查前收敛。K1/K2 challenge 只能由 protected main 的固定、domain-separated workflow 执行，public/evidence PR 分阶段避免未合并代码接触私钥；bridge 与 K2 版本各自拥有独立 canary→beta→stable 同字节门禁；新增 K14os，要求 Windows trusted signing identity 与 macOS Developer ID/notary probe 后才能公开；补齐受签名且兼容 Tauri static JSON 的 K1/K2 raw channel manifest、单次 `raw_json`→download→install 合同、每轮新 protected-main build、main-only Environment policy、无 admin bypass UI 负向演练和 GitHub immutable releases；并为最低 OS、WebView2 两路径、evidence-only/channel PR、基础 CODEOWNERS/reviewer 及 native harness/scenario 所有权建立可执行合同。
- `0.7`（2026-09-04）：最终对抗审查修订。将 K14os 限定为签名 profile/账号资格，把 G14 改为先合入固定 probe workflow、再创建五个空且受保护的 Environment，新增 K14v 负责凭据激活和双平台真实 sign/verify/notary probe，消除对不存在 Environment 的循环依赖；S14 冻结 `main-app-v1` 与 `repair-cli-v1` 两个 producer/schema/tag profile，使 S15c2 只能把无秘密的 repair build run 送入同一受保护 promotion 链；repair 首发固定为零视觉素材，Windows 发布含签名 EXE 的 ZIP，macOS 发布含 Developer ID-signed universal CLI 的 signed/notarized/stapled DMG；同时将跨平台 `artifactSetDigest` 定义为不依赖尚未创建的 Release asset ID 的 canonical 集合摘要。
- `0.8`（2026-09-04）：跨平台发布合同纠错。Windows 改为从同一 canonical unpatched PE 按 NSIS/MSI 分别执行 Tauri marker patch→inner sign→bundle→outer sign；macOS `.app` 改经临时 ZIP 提交公证后 staple 原 app，repair 只 staple 最终 DMG；拆分 updater 三目标 `artifactSetDigest` 与覆盖全部公开 payload 的 `releaseArtifactSetDigest`；S15a2 显式 custom target、MSI 零网络，并引入带 256 KiB/256 MiB 流式硬限的受审 vendored updater patch；原生 autostart 改为持久 controller + UI witness 的跨 boot/session Phase A/B；promotion 禁用 `workflow_call`，只允许 protected-main `workflow_dispatch` 并双重校验 GitHub workflow/job identity。
- `0.9`（2026-09-04）：最终 fail-closed 修订。明确客户端只独立复算三平台 `artifactSetDigest`，而把完整 Release 摘要/attestation ID 作为被 channel signature 覆盖的 lineage 引用，完整证明仍由 promotion/channel verifier 核验；所有 promotion 权限 job 直接依赖 identity、校验固定 verdict，并禁止 `always()` 等失败传播绕过；原生重启门禁改用 Phase A/Phase B 不同的 run-scoped JIT/ephemeral runner registration，A 注销且 control plane 确认 offline 后才重启，新 boot/session+witness ready 后才注册 B，消除持久 label 的派发竞态。
- `1.0`（2026-09-04）：macOS artifact 传输闭环。所有 unsigned `.app` 与 repair raw CLI 在 producer 主机先封装为单文件结构保真 transport tar，并用 canonical tree manifest 固定 path/type/mode/hash/symlink target；intake 与 `sign_macos` 在 macOS 上安全解包、逐项复算和断言 executable bit，拒绝 GitHub artifact 权限/链接损失及任何事后 `chmod` 猜测修补。
