# HarmonyOS WPS Open SDK Notes: 水印与安全策略（security-tob）

本文记录 HarmonyOS 应用集成 `@wps/wps_sdk` 时，如何把水印、修订、文档不落地与功能开关收成可复用的策略层。事实依据官方对接文档，示例按工程封装改写，便于仓库内对照联调。政企合规场景常见的验收点——可见归属、改动追溯、外泄面收敛、关窗回传清理——都可以映射到 `OpenFileRequest` 字段，而不是另起一套编辑器。

## 目标与调用链

策略层要回答四件事：谁打开过（水印）、改动如何追溯（修订）、文件会不会在 WPS 侧持久化（`enableLocalization`）、菜单级外泄能力如何关（`extraOptions`）。可选地，关窗后把文件拷回业务沙箱并删除 WPS 临时路径。

推荐时序：`RegisterAppRequest` 成功 → 可选 `setWpsFileToken` → 将选择器文件 copy 到 `filesDir` → 构造 `OpenFileRequest` 并写入策略字段 → `sendRequest` → 若开启 `TransferType.URI/FD`，拷贝后再按需 `unlink`。不要一天写满所有开关；顺序应为注册 → 沙箱只读 → 可编辑 → 水印/修订 → 落地与菜单 → 回传。

| 层级 | 职责 | 主要字段 |
|------|------|----------|
| 接入 | 注册 / 序列号 | `RegisterAppRequest` |
| 打开 | 路径 / 只读可编辑 | `enableEdit` |
| 策略 | 水印修订落地菜单 | `wpsWaterMarkParams` 等 |
| 结果 | 回传与清理 | `wpsTransferType` |

## 水印与修订参数

`WaterMark`：`Enable`、`WaterMaskText`、`Angle`、`FontColor`、`FontSize`。`Revision`：`UserName`、`EnterReviseMode`、`ShowRevisionPanel`、`EnterRevisionSilent`。水印文案建议来自登录态；预览与编辑共用打开函数。修订不等于禁打印，外泄控制仍依赖落地策略或 `extraOptions`。

```typescript
function buildWatermark(text: string): WaterMark {
  const mark = new WaterMark();
  mark.Enable = true;
  mark.WaterMaskText = text;
  mark.Angle = -30;
  mark.FontColor = '#19000000';
  mark.FontSize = 24;
  return mark;
}

function buildRevision(user: string): Revision {
  const rev = new Revision();
  rev.UserName = user;
  rev.EnterReviseMode = true;
  rev.ShowRevisionPanel = true;
  rev.EnterRevisionSilent = true;
  return rev;
}
```

## enableLocalization 与 extraOptions

| `enableLocalization` | 落地行为 | 敏感菜单 |
|----------------------|----------|----------|
| 未设置 / `false` | ToB 不落地 | 常被强制关闭 |
| `true` | 允许落地 | 由 `extraOptions` 配置 |

ToC 上设置 `enableLocalization` **不会**触发专业版不落地逻辑。不落地时云文档、分享、另存为、打印、导出、复制粘贴、截图等常被强制关闭，即使 `extraOptions` 开启也无效。`OpenFileExtraOptions` 仅显式赋值字段生效。

```typescript
async function openSecure(
  ctx: common.UIAbilityContext,
  sandboxPath: string,
  operator: string,
  allowPersist: boolean
) {
  const req = new OpenFileRequest(ctx, sandboxPath);
  req.enableEdit = true;
  req.enableLocalization = allowPersist;
  req.wpsWaterMarkParams = buildWatermark(`UID:${operator}`);
  req.wpsRevisionParams = buildRevision(operator);

  if (allowPersist) {
    const extra = new OpenFileExtraOptions();
    extra.enableShare = false;
    extra.enablePrint = false;
    extra.enableExport = false;
    extra.enableSaveAs = false;
    extra.enableScreenShot = false;
    req.extraOptions = extra;
  }

  req.wpsTransferType = TransferType.URI;
  const res = await WPSApi.sendRequest(req);
  if (res.code !== ResultCode.OK) {
    throw new Error(res.msg ?? String(res.code));
  }
  return res.data;
}
```

方案评审应先确认是否允许落地，再讨论菜单矩阵，否则真机上会出现「改了开关却没变化」。URI 回传场景下，拷贝到己方沙箱后再删除 WPS 临时文件，是常见合规动作；是否删除写进安全基线，而不是页面临场发挥。

## 联调清单与常见坑

未注册就打开应 reject；水印在预览与编辑均可见；修订作者与登录名一致；ToB 默认不落地时打印/分享不可用；`allowPersist=true` 后 `extraOptions` 才有细粒度空间；ToC HAR 不落地标志无效；Release 不打 secret；CI 打印 `bundleName` / HAR 批次 / `allowPersist` 布尔。

常见坑：外部 URI 未 copy；不落地仍狂调 `extraOptions`；忘记 `enableEdit`；水印写死在按钮；把注册失败算进附件重试。模块建议拆成 `auth` / `sandbox` / `policy` / `open`，页面只调 `openSecure(...)`。

弱网与冷启动要测按钮门禁；换 HAR 必须 clean；续期后同步 CI 与本地。Word 与表格小文件各跑 preview/edit。封装稳定后，产品改水印文案或菜单布尔，只需动 `policy` 模块。

## 小结与参考

安全策略可收敛为：水印可见归属、修订可追溯、不落地收紧外泄面、允许落地后再用 `extraOptions` 精细化。ToB/ToC 接口统一，但 HAR 与不落地语义必须匹配。字段与申请渠道以官方文档为准。

工程上建议把 `auth` / `sandbox` / `policy` / `open` 拆开，页面只调用 `openSecure`。内部文档写清水印模板、落地默认值与菜单基线，换 HAR 必须 clean，续期后同步 CI 与本地调试环境。启动竞态用按钮门禁加 in-flight Promise；注册失败不要进入附件重试队列。联调按「注册 → 沙箱只读加水印 → 可编辑加修订 → 验证落地与菜单 → 可选回传清理」推进，一次堆满参数时 `ResultCode.ERROR` 很难归因。

Release 包禁止打印 `appSecret`。CI 至少输出 `bundleName`、HAR 批次名、是否允许落地。Word 与表格小文件都要覆盖预览与编辑。封装稳定后，产品改水印文案或开关布尔，只需修改策略模块，不必每次从零打穿 Demo 按钮逻辑。

- 官方对接文档：https://365.kdocs.cn/l/clQl5cek2NoT  
- 申请 HAR 与凭据：m_open_sdk@wps.cn（注明包名与专业版/个人版需求）  
- 技术交流 QQ 群：628436767
