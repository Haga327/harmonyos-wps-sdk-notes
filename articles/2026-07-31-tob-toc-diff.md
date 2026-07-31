# HarmonyOS WPS Open SDK Notes: 专业版与个人版选型

本文用中英对照笔记形式，记录 HarmonyOS 工程在 **WPS Open SDK** 统一版下如何选型专业版（ToB）与个人版（ToC）：HAR / 凭据、`setWpsFileToken`、不落地边界，以及如何把分叉收进适配层。目标是立项评审就能答对「申请哪套包」，并在仓库里只保留一处打开 Facade。字段语义以官方对接文档为准，请随 SDK 版本逐项核对。

This note explains **ToB vs ToC** selection for WPS Open SDK on HarmonyOS. The API surface is unified; HAR, credentials, activation SN, and localization behavior still fork. Decide the edition before applying for HAR, then keep one Facade so pages never guess branches.

## Decision: three questions

1. Which WPS client is on device — Pro or Personal?  
2. Is no-landing / localization compliance required? → Pro (ToB).  
3. Can you run activation SN flow? Pro usually needs `setWpsFileToken`; Personal usually skips after register OK.

| Item | ToB (Pro) | ToC (Personal) |
|------|-----------|----------------|
| HAR / credentials | Pro batch | Personal batch |
| `registerApp` | required | required |
| `setWpsFileToken` | usually required | usually not |
| `enableLocalization` | effective | no-op |

Do not mix HAR batches. Credentials bind to `bundleName`.

## Adapter sketch

```typescript
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
```

Prefer product flavors for dual lines (each HAR / key pair) over runtime HAR switching. Never print full `appSecret` in Release.

## Shared open path

```typescript
import { common } from '@kit.AbilityKit';
import fs from '@ohos.file.fs';

function toSandbox(ctx: common.UIAbilityContext, src: string): string {
  const dir = `${ctx.filesDir}/wps_sel`;
  fs.mkdirSync(dir, true);
  const dest = `${dir}/${Date.now()}.docx`;
  fs.copyFileSync(src, dest);
  return dest;
}

export async function openDoc(
  ctx: common.UIAbilityContext,
  src: string,
  editable: boolean,
  allowLanding?: boolean
): Promise<Result> {
  await prepareWps(APP_KEY, APP_SECRET, PRO_SN_OR_EMPTY);
  const path = toSandbox(ctx, src);
  const req = new OpenFileRequest(ctx, path);
  req.enableEdit = editable;
  if (allowLanding !== undefined && !SdkConstants.isPersonalSdk()) {
    req.enableLocalization = allowLanding;
  }
  return WPSApi.sendRequest(req);
}
```

Bring-up order: register → read-only → editable → watermark / `extraOptions` / localization / transfer. One stack of all switches makes `ResultCode.ERROR` hard to attribute. 路径先落沙箱；外部 URI 权限不足时常见泛化 ERROR。

## Pitfalls checklist

- Unified API ≠ shared credentials / HAR  
- Copying ToB Token logic into ToC adds noise  
- Parallel `OpenHelper` for “the other edition” drifts fast  
- Product asks no-landing but engineering applied ToC  
- Debug OK / Release `1013` → `bundleName` mismatch; clean after HAR swap  

Keep a weekly对照表: package name, HAR filename, `isPersonalSdk`, Token injected, device WPS build. Search the repo for `new OpenFileRequest` and stray `wpsToken` on Request.

## Notes for dual product lines

If one repo serves both editions, inject `APP_KEY` / `APP_SECRET` / `PRO_SN_OR_EMPTY` via flavor. Pages only call `prepare` / `open`. Freeze parallel helpers in PR templates. After swapping HAR, clean and re-verify formal package registration.

For localization / no-landing acceptance, verify on a ToB build only; ToC settings will not prove compliance. Watermark and close-transfer still share the same `OpenFileRequest` object — extend optional params, do not fork files.

## Engineering rhythm

Treat selection as an engineering gate, not a marketing slide. Before coding, freeze a one-page对照表: package name, HAR filename, ToB/ToC, whether Token is injected, device WPS build. Testers accept against that table. After any HAR swap, clean and re-verify formal package registration to catch `1013` early.

When product later asks for preview watermark or close-transfer, extend `openDoc` optional params only. Do not create `openWithWatermarkTob` / `openTocLite` twins. Search weekly for `new OpenFileRequest` and Request-level `wpsToken`. Log `code` / `msg` / ready / `isPersonalSdk` / sandbox path / `enableEdit` on every failure.

Dual product lines should use flavors for keys and SN, not runtime HAR hot-swap. 路径务必先落沙箱；外部 URI 权限不足时常见泛化 ERROR。Release builds must not print full `appSecret`. No-landing acceptance belongs on ToB builds only — ToC settings will not prove compliance. Put these rules into the PR template so onboarding stays cheap.

再写一段中文备忘，方便国内同事直接粘进周报：选型评审要同时有产品、测试、研发三人签字——产品确认目标 WPS 客户端与是否不落地，测试确认验收机型与包名，研发确认 HAR 批次与是否注入序列号。缺一环就容易在联调第两周才发现申请错了包。统一版的价值是打开代码共用；它不会自动帮你选对交付物。把 Facade、对照表、PR 模板三件套固化后，新人接入周期通常能从「到处问」变成「按表勾选」。换 HAR 后 clean；正式包再验一次注册；全仓 `new OpenFileRequest` 命中数保持为一。字段与错误码继续以官方对接文档为准，随 SDK 小版本更新注释，而不是把整张参数表贴进业务页。

Also keep an English onboarding blurb in the README: apply with edition spelled out, never mix HAR, inject SN only in prepare, one open helper, sandbox paths only, clean after HAR swap, no secrets in Release logs. That short paragraph prevents most ToB/ToC mix-ups we have seen in the wild.

## Summary

选型是客户端形态、合规边界与序列号流程三件事。统一版让打开链路共用 Facade；分叉收在依赖与 `prepare`。申请时注明包名与专业版 / 个人版需求。字段以官方对接文档为准。把对照表与 Facade 纪律坚持几周，联调通常会从猜原因变成对表排查。若产品临时加预览水印或关窗回传，仍然只扩可选参数，不要再开平行 Helper——选型阶段立下的边界，才不会被临时需求冲垮。

Selection is about client flavor, compliance, and SN flow — not API count. Keep one Facade; fork only at dependency + prepare. Apply with package name and edition spelled out. A short对照表 plus weekly repo search beats long slide decks when proving the unified API actually landed in the codebase. Re-check docs after each SDK bump; keep comments describing project conventions rather than pasting entire parameter tables into UI code.

---
> Based on WPS Open SDK HarmonyOS integration notes for developers.  
> Docs: https://365.kdocs.cn/l/clQl5cek2NoT  
> QQ group: 628436767  
> Apply HAR / credentials: m_open_sdk@wps.cn
