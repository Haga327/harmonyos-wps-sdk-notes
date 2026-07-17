# HarmonyOS WPS Open SDK Notes: 能力矩阵与 OpenFileRequest 组合封装

本仓库笔记面向已经在用 `@wps/wps_sdk` 的 HarmonyOS 工程：把「二开能力」收敛成可评审的字段矩阵，并给出一份可粘贴的模块化封装。目标不是介绍「SDK 能做什么」的口号，而是让 PR 里可以直接对照：改了哪个字段、期望什么 `Result`、如何回归。字段语义以官方对接文档为准。

## 能力矩阵（Layer → Field）

| Layer | 能力 | API 落点 | 默认行为 | 常见误用 |
|-------|------|----------|----------|----------|
| Bootstrap | 应用注册 | `RegisterAppRequest` | 未成功则后续 `sendRequest` reject | 页面里连点 open |
| Bootstrap | 激活序列号 | `setWpsFileToken` | 按凭据形态需要 | 每次 open 塞 `wpsToken` |
| Open | 只读 / 可编辑 | `enableEdit` | 默认只读 | 以为「打不开」其实只读 |
| Open | 可读路径 | 业务侧 copy → `filesDir` | 外部路径可能 ERROR | 直接传选择器 URI |
| Policy | 水印 | `wpsWaterMarkParams` / `WaterMark` | 不设则无 | 透明度格式写错 |
| Policy | 菜单开关 | `extraOptions` / `OpenFileExtraOptions` | 未赋值保持默认 | 期望全关却只设一项 |
| Policy | 落地约束 | `enableLocalization` | 特定形态默认不落地 | 与 `extraOptions` 冲突时忽略强制关闭 |
| Outcome | 关窗回传 | `wpsTransferType` / `TransferType` | 默认不等待 | `OK` 无 `data` 当失败 |

叠加顺序建议：Bootstrap → Open（路径 + `enableEdit`）→ Policy → Outcome。一次堆满参数时，`ResultCode.ERROR` 很难归因。

## 模块边界

推荐把打开链路收进独立模块，避免 UI 页直接构造请求：

```
src/wps/
  register.ts
  sandbox.ts
  open.ts
  transfer.ts
  types.ts
```

职责边界：

1. `register.ts`：只处理 `1013` / `OK` / token 注入，不感知水印。  
2. `sandbox.ts`：只负责 copy 与扩展名保留，不感知业务上传。  
3. `open.ts`：只允许在这一处 `new OpenFileRequest`。  
4. `transfer.ts`：把 `fileUri` / `transferFd` 落到应用目录，断言与 open 成功拆开。  
5. `types.ts`：场景到字段的类型契约。

## Bootstrap 实现

```typescript
import { common } from '@kit.AbilityKit';
import {
  WPSApi,
  RegisterAppRequest,
  OpenFileRequest,
  OpenFileExtraOptions,
  WaterMark,
  TransferType,
  Result,
  ResultCode,
  SdkConstants,
} from '@wps/wps_sdk';

export let wpsReady = false;

export function logResult(tag: string, r: Result): void {
  console.info(`[WPS][${tag}] type=${r.requestType} code=${r.code} msg=${r.msg ?? ''}`);
}

export async function ensureRegistered(ctx: common.UIAbilityContext): Promise<void> {
  if (wpsReady) {
    return;
  }
  const result = await WPSApi.sendRequest(
    new RegisterAppRequest(ctx, APP_KEY, APP_SECRET)
  );
  logResult('register', result);
  if (result.code === ResultCode.ERROR_CODE_AUTH_FAILURE) {
    throw new Error(`register 1013: ${result.msg ?? ''}`);
  }
  if (result.code !== ResultCode.OK) {
    throw new Error(`register failed: ${result.code}`);
  }
  if (!SdkConstants.isPersonalSdk() && ACTIVATION_SN) {
    WPSApi.setWpsFileToken(ACTIVATION_SN);
  }
  wpsReady = true;
}
```

要点：

- 冷启动注册；UI 用 `wpsReady` 门禁。  
- 激活序列号用全局 `setWpsFileToken`，不要依赖 `OpenFileRequest.wpsToken`。  
- Debug / Release 分凭据文件；打包前打印 `bundleName`，减少「预发正常、上架 1013」。

## Open + Policy 组合

```typescript
import fs from '@ohos.file.fs';

export interface CapabilityOptions {
  editable: boolean;
  watermarkText?: string;
  lockOutbound?: boolean;
  waitTransfer?: boolean;
  allowLocalization?: boolean;
}

export function copyToSandbox(
  ctx: common.UIAbilityContext,
  src: string,
  ext = 'docx'
): string {
  const dir = `${ctx.filesDir}/wps_inbox`;
  fs.mkdirSync(dir, true);
  const dest = `${dir}/${Date.now()}.${ext}`;
  fs.copyFileSync(src, dest);
  return dest;
}

export async function openWithCapability(
  ctx: common.UIAbilityContext,
  src: string,
  opts: CapabilityOptions
): Promise<Result> {
  await ensureRegistered(ctx);
  const path = copyToSandbox(ctx, src);
  const req = new OpenFileRequest(ctx, path);
  req.enableEdit = opts.editable;

  if (opts.allowLocalization === true) {
    req.enableLocalization = true;
  }

  if (opts.watermarkText) {
    const wm = new WaterMark();
    wm.Enable = true;
    wm.WaterMaskText = opts.watermarkText;
    wm.Angle = -30;
    wm.FontColor = '#19000000';
    wm.FontSize = 16;
    req.wpsWaterMarkParams = wm;
  }

  if (opts.lockOutbound) {
    const extra = new OpenFileExtraOptions();
    extra.enableShare = false;
    extra.enableCloud = false;
    extra.enablePrint = false;
    extra.enableExport = false;
    req.extraOptions = extra;
  }

  if (opts.waitTransfer) {
    req.wpsTransferType = TransferType.URI;
  }

  const result = await WPSApi.sendRequest(req);
  logResult('open', result);
  if (result.code !== ResultCode.OK) {
    throw new Error(result.msg ?? `open ${result.code}`);
  }
  return result;
}
```

`OpenFileExtraOptions` 只对赋值字段生效。若当前为不落地约束，云文档 / 分享 / 打印等可能被 SDK 强制关闭——这是配置交互，不是「开关坏了」。真机菜单与预期不符时，先查 `enableLocalization`，再查 `extraOptions`。

调用示例：

```typescript
await openWithCapability(ctx, pickerPath, { editable: false });

await openWithCapability(ctx, pickerPath, {
  editable: true,
  watermarkText: `${uid}|内部`,
  lockOutbound: true,
});

await openWithCapability(ctx, pickerPath, {
  editable: true,
  waitTransfer: true,
});
```

## Transfer 层：与 open 成功拆断言

```typescript
export async function collectAfterClose(
  ctx: common.UIAbilityContext,
  src: string,
  opts: Omit<CapabilityOptions, 'waitTransfer'>
): Promise<string | null> {
  const result = await openWithCapability(ctx, src, {
    ...opts,
    editable: true,
    waitTransfer: true,
  });
  if (!result.data?.fileUri) {
    return null;
  }
  const handle = fs.openSync(result.data.fileUri, fs.OpenMode.READ_ONLY);
  const outDir = `${ctx.filesDir}/wps_out`;
  fs.mkdirSync(outDir, true);
  const dest = `${outDir}/${Date.now()}.docx`;
  fs.copyFileSync(handle.fd, dest);
  fs.closeSync(handle);
  return dest;
}
```

约定：

1. 未开回传时，`OK && !data` 是预期。  
2. 开了回传仍无 `data`，优先确认是否关文档窗（切回应用 ≠ 关窗）。  
3. FD 模式读 `transferFd`，校验 `transferFileSize`，用完关闭。  
4. 上传成功 toast 不要和 open 成功混用。

## PR Review Checklist

提交涉及 WPS 打开链路的改动时，至少核对：

1. 是否经过 `ensureRegistered`，而不是页面直调 `sendRequest`？  
2. 选择器文件是否已 copy 到沙箱，且扩展名保留？  
3. 可编辑流是否显式 `enableEdit = true`？  
4. `extraOptions` 是否只赋值需要的字段？  
5. 回传是否与 open 成功分开断言？  
6. HAR 升级后是否对照官方表刷新了本地能力矩阵？

最小回归矩阵：`preview/edit` × `docx/xlsx`；加回传后再测「关窗 / 仅切回应用」对照。弱网单独回归注册超时与 1013。

## Summary

HarmonyOS 上的 WPS Open SDK 二开能力，实质是 `OpenFileRequest` 的组合语义加上注册时序约束。仓库侧用能力矩阵 + 单一 `openWithCapability` 入口，可以把产品需求翻译成可 review 的字段变更，并控制 HAR / flavor 切换时的回归范围。权威字段表与强制关闭列表仍以官方对接文档为准。

---
> 基于 WPS Open SDK 鸿蒙版实践整理。  
> 官方对接文档：https://365.kdocs.cn/l/clQl5cek2NoT  
> QQ 群：628436767
