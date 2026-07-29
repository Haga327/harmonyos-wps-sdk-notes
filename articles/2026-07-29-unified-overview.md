# HarmonyOS WPS Open SDK Notes: 统一版概述

本文用中英对照笔记形式，概述 HarmonyOS 工程接入 **WPS Open SDK** 统一版时该抓住的主线：HAR 集成、单例 `WPSApi`、`registerApp` → `OpenFileRequest` → `Result`，以及 ToB / ToC 差异如何收在适配层。目标不是堆卖点，而是确认仓库里是否仍只有一处 Facade 在演进字段。字段语义以官方对接文档为准。

This note is an overview of the **unified** HarmonyOS WPS Open SDK surface: one API model for professional (ToB) and personal (ToC) packages, with differences expressed as credentials, optional activation token, and field effectiveness—not parallel helpers.

## Why “unified” matters

| Pain (before) | Unified approach |
|---------------|------------------|
| Two nearly identical open helpers | One `OpenFileRequest` Facade |
| Two demos / two checklists | One call chain, variant config |
| Unclear which credential a build uses | HAR + key/secret bound to `bundleName` |

统一版 = 接口统一 + 参数统一 + 流程统一。专业版能力保留，个人版打开能力扩容；厂商按场景申请对应 HAR / 凭据。Unified does **not** mean every field behaves identically on every package—read the doc for effectiveness notes.

## Call chain (source of truth)

1. Integrate HAR as `@wps/wps_sdk`
2. `WPSApi.registerApp(appKey, appSecret, callback)` — required before any `sendRequest`
3. Optionally `WPSApi.setWpsFileToken(sn)` in the success callback (ToB / credential-dependent)
4. Build `OpenFileRequest`, call `WPSApi.sendRequest`
5. Optionally read `Result.data` after close-transfer

| Layer | Focus | Common failure |
|-------|--------|----------------|
| Dependency | HAR + credentials + `bundleName` | Release-only `1013` |
| Register | Wait for OK | Exception: not registered |
| Open | Sandbox path + `enableEdit` | Generic `ERROR` on bad URI |
| Policy | Watermark / `extraOptions` / localization-related | Switch assigned but no effect |
| Result | Transfer type + `Result.data` | Empty data / wrong field branch |

Prefer `SdkConstants.isPersonalSdk()` inside the adapter, not in every page. 统一版的价值是接口统一，不是抹掉交付差异；概述阶段先问「全仓还有几处 `new OpenFileRequest`」。

## Minimal prepare + open

```typescript
import { common } from '@kit.AbilityKit';
import fs from '@ohos.file.fs';
import {
  WPSApi,
  OpenFileRequest,
  Result,
  ResultCode,
  SdkConstants,
  TransferType,
} from '@wps/wps_sdk';

export class OverviewFacade {
  private ready = false;

  prepare(key: string, secret: string, sn?: string): Promise<void> {
    return new Promise((resolve, reject) => {
      if (this.ready) {
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
          if (!SdkConstants.isPersonalSdk() && sn) {
            WPSApi.setWpsFileToken(sn);
          }
          this.ready = true;
          resolve();
        },
      });
    });
  }

  private toSandbox(ctx: common.UIAbilityContext, src: string): string {
    const dir = `${ctx.filesDir}/wps_overview`;
    fs.mkdirSync(dir, true);
    const dest = `${dir}/${Date.now()}.docx`;
    fs.copyFileSync(src, dest);
    return dest;
  }

  async open(
    ctx: common.UIAbilityContext,
    src: string,
    editable: boolean,
    needTransfer: boolean
  ): Promise<Result> {
    await this.prepare(APP_KEY, APP_SECRET, PRO_SN_OR_EMPTY);
    const path = this.toSandbox(ctx, src);
    const req = new OpenFileRequest(ctx, path);
    req.enableEdit = editable;
    if (needTransfer) {
      req.wpsTransferType = TransferType.TRANSFER_TYPE_URI;
    }
    return WPSApi.sendRequest(req);
  }
}
```

Do not print full `appSecret` in Release. After swapping HAR or flavor, clean and re-verify Release `bundleName`. 路径先落沙箱；可编辑与回传是独立开关。

## Capability map (short)

| Capability | Field / API | Note |
|------------|-------------|------|
| Register | `registerApp` | Must succeed before open |
| Activation | `setWpsFileToken` | Usually ToB; skip on personal SDK |
| Read/edit | `enableEdit` | Default read-only |
| Watermark | `wpsWaterMarkParams` | Inject at open |
| Menus | `extraOptions` | Only explicit fields apply |
| No-landing | `enableLocalization` | Primarily ToB behavior |
| Close transfer | `wpsTransferType` | Independent of edit flag |

联调顺序：注册 → 只读 → 可编辑 → 水印/`extraOptions` → 回传 → 落地相关。一次堆满开关很难归因。Recommended order keeps failures attributable.

## Checklist before merge

- Only one `new OpenFileRequest` site in the repo
- No per-open `request.wpsToken` leftovers
- Release does not log full secret
- Release `bundleName` matches credential application
- Close-transfer verified on device (`Result.data`)

Freeze parallel helpers; extend the Facade. Product still chooses client package and compliance constraints; engineering keeps one call chain. 字段与错误码以官方对接文档为准；申请 HAR / 凭据时注明专业版或个人版需求。

## Integration narrative (longer)

从申请到可联调，通常是这条路径：邮件申请 HAR 与 `appKey` / `appSecret`（注明专业版或个人版、包名）→ 工程依赖 `@wps/wps_sdk` → Ability 冷启动 `prepare` → 业务入口 `open` → 真机验证只读与可编辑 → 再叠水印 / `extraOptions` / 回传。From application to a green smoke test, the path is the same: request HAR + credentials (state ToB/ToC and `bundleName`) → add the package → cold-start `prepare` → business `open` → device verify read-only then edit → then watermark / menus / transfer.

个人版路径更轻：注册成功即可打开，跳过序列号。专业版在成功回调里全局设置 token，后续打开自动携带。Personal builds skip token; professional builds set it once globally. Do not mix packages and credentials across variants.

若同一仓库要服务多种交付，用 product flavor / build config 注入 `APP_KEY`、`APP_SECRET`、`PRO_SN_OR_EMPTY`，而不是复制 `OverviewFacade`。If one repo serves multiple deliveries, inject keys via flavor config—never clone the Facade class. 页面层永远只看见 `prepare` / `open`；是否 `isPersonalSdk` 留在 Facade 内部。

联调日志建议固定格式：`code`、`msg`、当前是否 `ready`、路径是否沙箱前缀、本次是否赋值 `enableEdit` / `wpsTransferType`。A stable log line beats “打开失败” every time. 当 `extraOptions`「改了没效果」时，先确认字段是否显式赋值，再确认当前 HAR 是否允许该能力（不落地模式下部分菜单会被强制关闭）。When a menu switch “does nothing”, check explicit assignment first, then package effectiveness.

周节奏可以很短：搜 `new OpenFileRequest`、搜 `wpsToken`、用 Release 包名跑一次注册、真机关一次带回传的编辑。Weekly: search hits, Release register, one transfer close. 命中数下降才算统一版真正落到工程。Hit count down = unified for real.

## Summary

Unified HarmonyOS WPS Open SDK overview in three lines: single `WPSApi`, shared `OpenFileRequest`, differences in adapter config. 把注册和打开收成 `prepare` / `open` 之后，预览、编辑、后续加水印都走同一条路。Keep searching for duplicate open sites weekly—hit count is a better signal than “we already unified.” 字段与错误码随官方文档核对；申请渠道与 QQ / 邮箱见文末。Keep the doc link next to the Facade so newcomers do not paste stale parameter tables into comments.

---
> Based on WPS Open SDK HarmonyOS integration practice. For reference only.
> Docs: https://365.kdocs.cn/l/clQl5cek2NoT
> QQ: 628436767 · Email: m_open_sdk@wps.cn
