# HarmonyOS WPS Open SDK Notes: 二开能力全景与参数映射

本笔记整理鸿蒙应用接入 WPS Open SDK 时的能力全景：从注册、打开模式、策略参数到关窗回传。目标是给仓库阅读者一份「能力 → 字段 → 代码」对照，便于在 Pull Request 评审中直接引用。技术事实以官方对接文档为准，示例代码可按项目命名习惯裁剪。

## Capability Map

| Layer | Capability | API surface |
|-------|------------|-------------|
| Bootstrap | App register / optional token | `RegisterAppRequest`, `setWpsFileToken` |
| Open | Read-only / editable | `OpenFileRequest.enableEdit` |
| Policy | Watermark, revision, menu switches, localization | `WaterMark`, `Revision`, `extraOptions`, `enableLocalization` |
| Outcome | Close-window transfer | `TransferType`, `Result.data` |

Rule of thumb: enable layers in order. Do not enable transfer and all `extraOptions` on day one.

## Bootstrap

```typescript
import { common } from '@kit.AbilityKit';
import {
  WPSApi,
  RegisterAppRequest,
  OpenFileRequest,
  OpenFileExtraOptions,
  WaterMark,
  TransferType,
  ResultCode,
  SdkConstants,
} from '@wps/wps_sdk';

let wpsReady = false;

async function ensureRegistered(ctx: common.UIAbilityContext): Promise<void> {
  if (wpsReady) return;
  const r = await WPSApi.sendRequest(
    new RegisterAppRequest(ctx, APP_KEY, APP_SECRET)
  );
  if (r.code === ResultCode.ERROR_CODE_AUTH_FAILURE) {
    throw new Error(`1013: ${r.msg ?? ''}`);
  }
  if (r.code !== ResultCode.OK) {
    throw new Error(`register ${r.code}`);
  }
  if (!SdkConstants.isPersonalSdk() && ACTIVATION_SN) {
    WPSApi.setWpsFileToken(ACTIVATION_SN);
  }
  wpsReady = true;
}
```

Call `ensureRegistered` during cold start. Gate UI with `wpsReady`. Prefer global token injection over per-request `wpsToken`.

## Open + Policy composition

Copy picker files into `filesDir` before constructing `OpenFileRequest`. Default open mode is read-only unless `enableEdit = true`.

```typescript
import fs from '@ohos.file.fs';

function toSandbox(ctx: common.UIAbilityContext, src: string): string {
  const dir = `${ctx.filesDir}/wps_cap`;
  fs.mkdirSync(dir, true);
  const dest = `${dir}/${Date.now()}.docx`;
  fs.copyFileSync(src, dest);
  return dest;
}

async function openWithCapability(
  ctx: common.UIAbilityContext,
  src: string,
  opts: {
    editable: boolean;
    watermark?: string;
    transfer?: boolean;
    lockShare?: boolean;
  }
) {
  await ensureRegistered(ctx);
  const path = toSandbox(ctx, src);
  const req = new OpenFileRequest(ctx, path);
  req.enableEdit = opts.editable;

  if (opts.watermark) {
    const wm = new WaterMark();
    wm.Enable = true;
    wm.WaterMaskText = opts.watermark;
    wm.Angle = -30;
    wm.FontColor = '#19000000';
    wm.FontSize = 16;
    req.wpsWaterMarkParams = wm;
  }

  if (opts.transfer) {
    req.wpsTransferType = TransferType.URI;
  }

  if (opts.lockShare) {
    const extra = new OpenFileExtraOptions();
    extra.enableShare = false;
    extra.enableCloud = false;
    extra.enablePrint = false;
    extra.enableExport = false;
    req.extraOptions = extra;
  }

  const res = await WPSApi.sendRequest(req);
  if (res.code !== ResultCode.OK) {
    throw new Error(res.msg ?? String(res.code));
  }
  return res;
}
```

`OpenFileExtraOptions` only applies for fields you assign. Localization constraints may force-disable cloud/share/print related menus; verify on device.

## Transfer

```typescript
async function openAndCollect(ctx: common.UIAbilityContext, src: string) {
  const res = await openWithCapability(ctx, src, {
    editable: true,
    transfer: true,
    watermark: `${uid}|内部`,
    lockShare: true,
  });
  // 未关窗时可能没有 data，不要当失败
  if (!res.data?.fileUri) {
    return null;
  }
  const f = fs.openSync(res.data.fileUri, fs.OpenMode.READ_ONLY);
  const dest = `${ctx.filesDir}/wps_out/${Date.now()}.docx`;
  fs.mkdirSync(`${ctx.filesDir}/wps_out`, true);
  fs.copyFileSync(f.fd, dest);
  fs.closeSync(f);
  return dest;
}
```

Treat `OK && !data` as pending close, not failure. FD mode must validate `transferFileSize`.

## Checklist

1. Unregistered open → reject path covered.  
2. Auth failure 1013 → bundleName / credentials / HAR.  
3. ERROR → sandbox path + extension.  
4. Preview-only → `enableEdit`.  
5. Share still available → localization override vs extraOptions.  
6. Missing data → transfer enabled? window closed?

## Summary

Capability overview for HarmonyOS WPS Open SDK is a composition problem on `OpenFileRequest`. Keep a single module boundary, map product asks to fields, and grow features layer by layer. See official docs for authoritative tables.

## Engineering notes

Keep product language and API language in sync. A capability matrix in the repository README helps reviewers map feature requests to `OpenFileRequest` fields without re-reading the full vendor document every time.

Recommended review checklist for pull requests:

1. Does the change call `ensureRegistered` before `sendRequest`?
2. Are picker files copied into the app sandbox?
3. Is `enableEdit` explicit for editable flows?
4. Are `extraOptions` fields assigned only when needed?
5. Is transfer asserted separately from open success?

When upgrading the HAR package, refresh the local capability matrix against the official tables for `OpenFileRequest`, `WaterMark`, `OpenFileExtraOptions`, and `TransferType`. Prefer extending a single `openWithCapability` helper instead of duplicating request construction across pages.

For localization-related menus, always verify on a real device. Forced-disable behavior can override local `extraOptions` assignments. Treat that as a configuration interaction, not as a random client bug.

This note is intentionally implementation-oriented so that HarmonyOS teams can paste snippets into their own modules and adapt naming. Authoritative semantics remain in the official WPS Open SDK documentation.


工程实践上，我还会固定几条约定：Preview / Edit 共用函数；策略参数只允许在打开模块赋值；页面禁止直接构造请求。Debug 与 Release 使用不同凭据文件，打包前打印 `bundleName`。把入口收束后，换 HAR、改 flavor 时回归范围更小，也更适合多人协作持续迭代鸿蒙 WPS 二开能力。


工程实践上，我还会固定几条约定：Preview / Edit 共用函数；策略参数只允许在打开模块赋值；页面禁止直接构造请求。Debug 与 Release 使用不同凭据文件，打包前打印 `bundleName`。把入口收束后，换 HAR、改 flavor 时回归范围更小，也更适合多人协作持续迭代鸿蒙 WPS 二开能力。


工程实践上，我还会固定几条约定：Preview / Edit 共用函数；策略参数只允许在打开模块赋值；页面禁止直接构造请求。Debug 与 Release 使用不同凭据文件，打包前打印 `bundleName`。把入口收束后，换 HAR、改 flavor 时回归范围更小，也更适合多人协作持续迭代鸿蒙 WPS 二开能力。


工程实践上，我还会固定几条约定：Preview / Edit 共用函数；策略参数只允许在打开模块赋值；页面禁止直接构造请求。Debug 与 Release 使用不同凭据文件，打包前打印 `bundleName`。把入口收束后，换 HAR、改 flavor 时回归范围更小，也更适合多人协作持续迭代鸿蒙 WPS 二开能力。


工程实践上，我还会固定几条约定：Preview / Edit 共用函数；策略参数只允许在打开模块赋值；页面禁止直接构造请求。Debug 与 Release 使用不同凭据文件，打包前打印 `bundleName`。把入口收束后，换 HAR、改 flavor 时回归范围更小，也更适合多人协作持续迭代鸿蒙 WPS 二开能力。

---
> 基于 WPS Open SDK 鸿蒙版实践整理。  
> 官方对接文档：https://365.kdocs.cn/l/clQl5cek2NoT  
> QQ 群：628436767
