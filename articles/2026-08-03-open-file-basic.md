# HarmonyOS WPS Open SDK Notes: OpenFileRequest 打开文档

本文用中英对照笔记形式，记录 HarmonyOS 工程用 **WPS Open SDK** 的 `OpenFileRequest` 打开本地文档：注册时序、沙箱路径、`enableEdit`、以及如何把打开收敛到一处 Facade。目标是半天跑通真机打开，并避免仓库里长出平行 Helper。字段语义以官方对接文档为准，请随 SDK 版本逐项核对。

This note covers **opening a local document** via `OpenFileRequest` on HarmonyOS: wait for register OK, copy into the app sandbox, set `enableEdit`, then `sendRequest`. Keep one Facade so pages never call `new OpenFileRequest` directly.

## Minimum loop

Goal: import `@wps/wps_sdk` → `registerApp` OK → sandbox read-only open → `enableEdit = true`. Defer watermark / `extraOptions` / localization / close-transfer until this loop is green. 一次堆满开关很难归因。

ToB and ToC share the same open API; fork only at HAR / credentials and optional `setWpsFileToken`. Do not duplicate open helpers per edition.

## Prepare + sandbox open

```typescript
import { common } from '@kit.AbilityKit';
import fs from '@ohos.file.fs';
import {
  WPSApi,
  OpenFileRequest,
  Result,
  ResultCode,
  SdkConstants,
} from '@wps/wps_sdk';

let ready = false;

export function prepareWps(
  key: string,
  secret: string,
  proSn?: string
): Promise<void> {
  return new Promise((resolve, reject) => {
    if (ready) {
      resolve();
      return;
    }
    WPSApi.registerApp(key, secret, {
      onCallback: (result: Result): void => {
        if (result.code === ResultCode.ERROR_CODE_AUTH_FAILURE) {
          reject(new Error(`1013: ${result.msg ?? ''}`));
          return;
        }
        if (result.code !== ResultCode.OK) {
          reject(new Error(`register ${result.code}`));
          return;
        }
        if (!SdkConstants.isPersonalSdk() && proSn) {
          WPSApi.setWpsFileToken(proSn);
        }
        ready = true;
        resolve();
      },
    });
  });
}

function toSandbox(ctx: common.UIAbilityContext, src: string): string {
  const dir = `${ctx.filesDir}/wps_open`;
  fs.mkdirSync(dir, true);
  const dest = `${dir}/${Date.now()}.docx`;
  fs.copyFileSync(src, dest);
  return dest;
}

export async function openDoc(
  ctx: common.UIAbilityContext,
  src: string,
  editable: boolean
): Promise<Result> {
  await prepareWps(APP_KEY, APP_SECRET, PRO_SN_OR_EMPTY);
  const path = toSandbox(ctx, src);
  const req = new OpenFileRequest(ctx, path);
  req.enableEdit = editable;
  return WPSApi.sendRequest(req);
}
```

`enableEdit` defaults to read-only; only explicit `true` edits. Transfer (`wpsTransferType`) is independent. Without transfer, OK with empty `data` is expected — do not toast “saved”.

## Pitfalls checklist

| Symptom | Check first |
|---------|-------------|
| Promise reject | wait for `prepareWps` |
| `1013` | key / secret / `bundleName` |
| Generic ERROR | sandbox path / Context |
| Release-only fail | clean; formal package name |
| Strategy no-op | explicit assign; HAR allows it |

Never print full `appSecret` in Release. Prefer global `setWpsFileToken` over Request-level `wpsToken`. After swapping HAR, clean before rebuild.

## Engineering rhythm

Freeze a one-page对照表: HAR filename, credential env, Token injected?, sandbox dir, device WPS build. Testers accept against that table. PR template: one `new OpenFileRequest` in the repo; pages only call `prepare` / `open`; clean after HAR swap; verify formal package registration.

When product asks for preview watermark or close-transfer later, extend `openDoc` optional params only. Weekly search for stray `new OpenFileRequest` and Request-level `wpsToken`. Log `code` / `msg` / ready / sandbox / `enableEdit` on every failure.

Dual product lines should inject keys via flavor, not runtime HAR hot-swap. 路径务必先落沙箱；外部 URI 权限不足时常见泛化 ERROR。正式包与调试包包名不同时，凭据批次分开归档，避免「本地绿、上架红」。

把「先绿再叠」写进联调清单：注册 → 只读 → 可编辑 → 策略 → 回传。Facade 纪律坚持几周后，联调通常会从猜原因变成对表排查。字段与错误码继续以官方对接文档为准，随 SDK 小版本更新注释，而不是把整张参数表贴进业务页。

Also keep a short English README blurb: register before open, sandbox paths only, one open helper, clean after HAR swap, no secrets in Release logs. That paragraph prevents most open-path mix-ups we have seen. Prefer product flavors for dual lines rather than runtime HAR switching. After any HAR swap, re-verify formal package registration to catch `1013` early. Keep the open button disabled until `ready` is true so users never tap into a half-initialized SDK session.

## From Demo to dual product lines

Demo proves “it opens.” A product repo proves “it stays maintainable.” Migrate only three things: dependency layout, `prepareWps` / `openDoc`, and the bring-up checklist. Do not copy UI layouts or embed secrets in page params. For dual editions, inject keys via flavor and keep one Facade so pages only see `prepare` / `open`.

再写一段中文备忘：打开评审要同时确认 HAR 文件名、凭据环境、是否注入序列号、沙箱目录约定、真机 WPS 构建号。缺一环就容易在联调中段才发现路径或凭据问题。统一版减少的是页面层分裂，它不会自动把选择器 URI 变成沙箱路径。把 Facade、对照表、PR 模板三件套固化后，新人接入周期通常能从「到处问」变成「按表勾选」。换 HAR 后 clean；正式包再验一次注册；全仓 `new OpenFileRequest` 命中数保持为一。

If product later asks for preview watermark or close-transfer, extend optional params only — never spawn `openWithWatermark` twins. Search weekly. Log failures with `code` / `msg` / ready / sandbox / `enableEdit`. 路径务必先落沙箱；外部 URI 权限不足时常见泛化 ERROR。正式包与调试包包名不同时，凭据批次分开归档。

## Summary

`OpenFileRequest` 打开文档的本质是四句话：注册等到 OK、文件进沙箱、只读先绿再开编辑、结果用 `code`/`msg` 归因。统一版让打开链路共用 Facade；分叉收在依赖与 `prepare`。申请时注明包名与交付形态。字段以官方对接文档为准。把对照表与 Facade 纪律坚持几周，联调通常会从猜原因变成对表排查。

Opening is register → sandbox → read-only → editable, then layer strategy. Keep one Facade; fork only at dependency + prepare. Re-check docs after each SDK bump; keep project conventions in comments rather than pasting entire parameter tables into UI code. A short对照表 plus weekly repo search beats long slide decks when proving the open path actually landed.

---
> Based on WPS Open SDK HarmonyOS integration notes for developers.  
> Docs: https://365.kdocs.cn/l/clQl5cek2NoT  
> QQ group: 628436767  
> Apply HAR / credentials: m_open_sdk@wps.cn
