# HarmonyOS WPS Open SDK Notes: 快速入门

本文用中英对照笔记形式，记录 HarmonyOS 工程接入 **WPS Open SDK** 的最短路径：HAR 依赖、`registerApp`、沙箱 `OpenFileRequest`，以及 ToB / ToC 差异如何收在适配层。目标是半天跑通真机打开，并把打开封装收敛到一处 Facade。字段语义以官方对接文档为准，请随 SDK 版本逐项核对。

This note is a **quick-start** for integrating WPS Open SDK on HarmonyOS: install HAR, wait for register OK, copy files into the app sandbox, then open via one `OpenFileRequest` helper. Keep one Facade early so the repo does not grow parallel open helpers.

## Dependency and minimum loop

Goal: `@wps/wps_sdk` imports → `registerApp` OK → sandbox read-only open → `enableEdit = true`. Defer watermark / `extraOptions` / localization / close-transfer until this loop is green. 一次堆满开关很难归因。

```json5
{
  "dependencies": {
    "@wps/wps_sdk": "file:./libs/wps_sdk.har"
  }
}
```

Place `wps_sdk.har` under `./libs/`, run `ohpm install`. After swapping HAR batches, clean before rebuild. Credentials bind to `bundleName`; debug and release packages often need separate applications or an explicit choice.

## Prepare + open

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

export class QuickStartFacade {
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
    const dir = `${ctx.filesDir}/wps_qs`;
    fs.mkdirSync(dir, true);
    const dest = `${dir}/${Date.now()}.docx`;
    fs.copyFileSync(src, dest);
    return dest;
  }

  async open(
    ctx: common.UIAbilityContext,
    src: string,
    editable: boolean,
    needTransfer = false
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

Calling `sendRequest` before register OK throws. Prefer global `setWpsFileToken` over per-request `wpsToken`. 路径先落沙箱；可编辑与回传是独立开关。

## ToB / ToC (quick facts)

| Item | Professional (ToB) | Personal (ToC) |
|------|--------------------|----------------|
| HAR / credentials | Must match | Must match |
| `registerApp` | Required | Required |
| Activation token | Usually `setWpsFileToken` | Usually skip after OK |
| Localization-related | Per doc | Some fields no-op |

Keep `isPersonalSdk()` inside the adapter. Unified API ≠ identical field effectiveness.

## Failure table

| Symptom | Check first |
|---------|-------------|
| Exception on `sendRequest` | Register finished? |
| `1013` | key / secret / `bundleName` |
| Generic `ERROR` | Sandbox path / Context |
| Release-only fail | clean; release package name |

Do not print full `appSecret` in Release. Search the repo for `new OpenFileRequest` and `wpsToken` weekly—hit count is the real adoption signal.

## Integration narrative

从申请到可联调：邮件申请 HAR 与 `appKey` / `appSecret`（注明专业版或个人版、包名）→ 工程依赖 → Ability 冷启动 `prepare` → 业务入口 `open` → 真机只读再可编辑。From application to smoke test: request HAR + credentials → add package → cold-start `prepare` → business `open` → verify read-only then edit.

若同一仓库服务多种交付，用 flavor 注入 `APP_KEY` / `APP_SECRET` / `PRO_SN_OR_EMPTY`，不要复制 Facade 类。If one repo serves multiple deliveries, inject via flavor config—never clone the Facade. 页面层永远只看见 `prepare` / `open`。

联调日志固定：`code`、`msg`、是否 `ready`、路径是否沙箱前缀、本次是否赋值 `enableEdit`。A stable log line beats “打开失败”. 当后续叠 `extraOptions`「改了没效果」时，先确认显式赋值，再确认当前 HAR 是否允许该能力。

合入前再核对：全仓只有一处 `new OpenFileRequest`、无逐次 `request.wpsToken`、Release 不打印完整 secret、正式包包名与凭据一致、真机验过只读与可编辑。Freeze parallel helpers; extend the Facade.

## From demo to production

Demo proves “it opens”. Production proves “it stays maintainable”. Migrate three things only: HAR path convention, `prepare` / `open`, and the ordered checklist. Do not migrate page layout or hard-code secrets into UI props.

If one repo serves multiple deliveries, inject `APP_KEY` / `APP_SECRET` / `PRO_SN_OR_EMPTY` via flavor. Keep a single Facade class. Pages only see `prepare` / `open`. Diffs should land in config, not in every button handler.

Log line suggestion: `code`, `msg`, ready flag, sandbox path prefix, `enableEdit`. When later `extraOptions` “do nothing”, check explicit assignment first, then package effectiveness. 快速入门的工程纪律是：从今天起全仓只允许一处 `new OpenFileRequest`。两周后再搜命中数，比写长 README 更能证明落地。

Also verify close-transfer only after the basic loop is green. Prefer global token setup. Never print full secrets in Release builds. Keep the official doc link next to the Facade so newcomers do not paste stale parameter tables into comments. 字段与错误码随官方文档核对；申请 HAR / 凭据时注明专业版或个人版需求。

## Summary

Quick-start on HarmonyOS WPS Open SDK: install HAR, await register, sandbox path, one open helper. 专业版与个人版共用 API，差异收在适配层。Keep searching for duplicate open sites—hit count down means the quick-start actually landed in the repo.

补充工程提醒：快速入门的验收不只是「Demo 能开」，而是依赖目录、注册出口、沙箱打开函数成为团队约定。合入前用正式包包名验注册，真机各跑只读与可编辑；日志打全 `code` / `msg`。预览水印或关窗回传只扩 Facade 可选参数。换 HAR 后 clean；Release 不打印完整 secret；不要逐次 `wpsToken`。把检查表写进 PR 模板，两周后再搜 `new OpenFileRequest`，命中数仍为一才算入门成功。Also keep the official doc URL next to the Facade comments so parameter semantics stay synchronized with the current HAR. 路径先落沙箱；外部 URI 权限不足时常见泛化 ERROR。正式包与调试包包名不同时，凭据批次也要分开归档。

---
> Based on WPS Open SDK HarmonyOS integration practice. For reference only.
> Docs: https://365.kdocs.cn/l/clQl5cek2NoT
> QQ: 628436767 · Email: m_open_sdk@wps.cn
