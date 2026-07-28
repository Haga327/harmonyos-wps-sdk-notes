# HarmonyOS WPS Open SDK Notes: 能力周回顾

本文用中英对照笔记形式，回顾 HarmonyOS 工程接入 WPS Open SDK 时每周应盯的主线：注册、打开、策略与关窗回传。目标不是堆参数表，而是确认仓库里是否仍只有一处 Facade 在演进字段，并把联调顺序固化成可执行清单。字段语义以官方对接文档为准。

This note summarizes a practical weekly recap for integrating **WPS Open SDK** on **HarmonyOS**: what must stay on the single call chain, what can be layered as options, and how to keep one Facade instead of parallel helpers.

## Call chain (source of truth)

1. Integrate HAR as `@wps/wps_sdk`
2. `WPSApi.registerApp(appKey, appSecret, callback)` — required before any `sendRequest`
3. Optionally `WPSApi.setWpsFileToken(sn)` in the success callback (credential-dependent)
4. Build `OpenFileRequest`, call `WPSApi.sendRequest`
5. Optionally read `Result.data` after close-transfer

| Layer | Focus | Common failure |
|-------|--------|----------------|
| Dependency | HAR + credentials + `bundleName` | Release-only `1013` |
| Register | Wait for OK | Exception: not registered |
| Open | Sandbox path + `enableEdit` | Generic `ERROR` on bad URI |
| Policy | Watermark / `extraOptions` / localization-related | Switch assigned but no effect |
| Result | Transfer type + `Result.data` | Empty data / wrong field branch |

ToB and ToC share the same API surface; differences are HAR, credentials, token requirement, and which fields take effect. Prefer `SdkConstants.isPersonalSdk()` inside the adapter, not in every page. 统一版的价值是接口统一，不是抹掉交付差异；周回顾时先问「全仓还有几处 `new OpenFileRequest`」。

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

export class RecapFacade {
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

  async open(
    ctx: common.UIAbilityContext,
    src: string,
    editable: boolean,
    transfer: boolean
  ): Promise<Result> {
    if (!this.ready) {
      throw new Error('prepare first');
    }
    const dir = `${ctx.filesDir}/wps_recap`;
    fs.mkdirSync(dir, true);
    const dest = `${dir}/${Date.now()}.docx`;
    fs.copyFileSync(src, dest);

    const req = new OpenFileRequest(ctx, dest);
    req.enableEdit = editable;
    if (transfer) {
      req.wpsTransferType = TransferType.TRANSFER_TYPE_URI;
    }
    return WPSApi.sendRequest(req);
  }
}
```

Do not assign `request.wpsToken` on every open if a global token is already set; global wins. Keep preview and edit on the same function — only toggles differ. 路径务必先落到应用沙箱；外部 URI 权限不足时，常见现象是泛化 `ResultCode.ERROR`，日志里又看不到明确「权限」文案。

## Layering checklist for weekly review

1. Register OK on the Release package name  
2. Read-only open from sandbox path  
3. `enableEdit = true`  
4. Watermark / revision if required  
5. `extraOptions` only for fields you intentionally set  
6. Localization-related fields only after confirming current HAR contract  
7. Close-transfer and parse `Result.data` (URI vs FD branches)  
8. Grep the repo: one `new OpenFileRequest` site; no leftover parallel helpers  

`extraOptions` applies only to explicitly assigned fields. Stacking every switch at once makes `ResultCode.ERROR` hard to attribute. 建议把上述清单写进 PR 模板：周一扫依赖与凭据，周三跑正式包注册，周五真机验证回传。命中数下降，比「感觉改完了」更可靠。

## Engineering notes

- Clean after HAR replacement; stale binaries cause intermittent auth or behavior mismatch.  
- Never log full `appSecret` in Release.  
- Edit and transfer are independent flags — combine by entry semantics.  
- Product chooses client flavor / compliance; engineering keeps one Facade and injects keys via build variants.  

多人并行接入时，冻结新增平行 Helper，统一走 Facade 合入。水印、修订、`extraOptions` 若原先散落在页面按钮回调里，复盘窗口正好收进可选参数，避免继续长出第三套打开路径。合入前再核对对接文档版本号与当前 HAR 说明，避免示例与交付包行为不一致。多 flavor 时构建变体分别注入 key / secret / 可选 sn，运行时共用同一客户端类；UI 文案可区分场景，但不要分叉 `OpenFileRequest` 构造路径。也可以把「注册成功 / 沙箱只读 / 可编辑 / 回传」写成固定回归用例，每次换 HAR 后跑一遍。真机关窗至少验证一次 URI 或 FD 回传字段是否非空。Release 日志不要打印完整 secret。调试包正常、正式包 `1013` 时，优先查 `bundleName` 与凭据批次。未注册就 `sendRequest` 会抛异常——这是统一约束，不是偶发 bug。把 Facade 约定写进 README 两三句话，周复盘时对照命中数，比贴整张参数表更耐维护。若团队有多模块并行接入文档能力，复盘窗口内应冻结新增平行打开封装，统一合入后再加字段，减少回退成本。

## Summary

A weekly recap for HarmonyOS WPS Open SDK should reaffirm the single call chain, credential alignment, and Facade discipline — not rewrite APIs. Keep differences in the adapter; pages only call `prepare` / `open`. Field semantics follow the official integration doc for your SDK drop.

周回顾的结论应能落到仓库：全仓一处打开封装、凭据与包名对齐、联调按层叠加。把这些写进 README 两三句话，比口头交接更不容易回退到旧写法。下一迭代加字段时，先改 Facade 再改页面，命中数就不会反弹。细节仍以官方对接文档与当前 HAR 交付说明为准。

Apply for HAR and credentials via `m_open_sdk@wps.cn` (include bundle name and required SDK flavor).  
QQ group: 628436767

---
> Based on WPS Open SDK HarmonyOS integration practice for developers.
> Official doc: https://365.kdocs.cn/l/clQl5cek2NoT
