# HarmonyOS WPS Open SDK Notes: SdkConstants HAR Flavor Check

在 HarmonyOS 上接入 `@wps/wps_sdk` 时，统一入口仍是 `WPSApi`，但交付物是 **两份 HAR**（ToB 专业版 / ToC 个人版）。运行时识别当前包的官方方法是 `SdkConstants.isPersonalSdk()`：`true` → 个人版路径，`false` → 专业版路径。不要解析设备上的 WPS 客户端 `bundleName`。本文把查询收进 adapter，并接到注册、Token 与 `sendRequest`；字段语义以官方对接文档为准。

## 范围

| 层 | 覆盖点 |
|----|--------|
| 构建 | `oh-package.json5` → `file:./libs/wps_sdk.har` |
| 运行时 | `SdkConstants.isPersonalSdk()` |
| 门禁 | `RegisterAppRequest` / 1013 |
| Token | 仅专业版 `setWpsFileToken` |
| 打开 | `OpenFileRequest` + `sendRequest` |

ToC 通常注册即可打开；ToB 常在注册 OK 后注入序列号。`enableLocalization` 在 ToC **赋值不生效**。`wpsUpdateInfo` 是客户端升级弹窗，与 HAR 形态无关。联调原则：先认 flavor，再绿门禁，再打开。一次堆满开关时，`ResultCode.ERROR` 很难归因。

同一仓库兼 ToB/ToC 时，用构建变体注入凭据；页面层不要复制两套 Promise 链。全仓只允许 adapter 调用 `isPersonalSdk()`。换 HAR 后 clean。日志固定：`flavor`、`ready`、包名后缀。

## Adapter

```typescript
import {
  WPSApi,
  RegisterAppRequest,
  OpenFileRequest,
  ResultCode,
  SdkConstants,
} from '@wps/wps_sdk';

let ready = false;

export function sdkFlavor(): 'personal' | 'pro' {
  return SdkConstants.isPersonalSdk() ? 'personal' : 'pro';
}

export async function ensureWps(
  ctx: UIAbilityContext,
  cred: { key: string; secret: string; sn?: string }
): Promise<void> {
  if (ready) return;
  const r = await WPSApi.sendRequest(
    new RegisterAppRequest(ctx, cred.key, cred.secret)
  );
  if (r.code === ResultCode.ERROR_CODE_AUTH_FAILURE) {
    throw new Error(`1013: ${r.msg ?? ''}`);
  }
  if (r.code !== ResultCode.OK) {
    throw new Error(`register ${r.code}`);
  }
  if (!SdkConstants.isPersonalSdk() && cred.sn) {
    WPSApi.setWpsFileToken(cred.sn);
  }
  ready = true;
}
```

包名与凭据绑定。Release 不打印 secret。也可使用回调式 `registerApp`；关键是业务打开前 `ready === true`。打开按钮在 ready 前禁用。限时凭据失效需重新申请。

注册失败不要继续打开。Token 仅在专业版分支注入。Ability 启动阶段完成 ensure。若同时设置全局 Token 与 `request.wpsToken`，以全局设置为准；不推荐每次打开写 `wpsToken`。

## Open facade

```typescript
export async function openDoc(
  ctx: UIAbilityContext,
  path: string,
  opts: { edit?: boolean; allowLand?: boolean } = {}
) {
  await ensureWps(ctx, CRED);
  const req = new OpenFileRequest(ctx, path);
  req.enableEdit = !!opts.edit;
  if (typeof opts.allowLand === 'boolean' && !SdkConstants.isPersonalSdk()) {
    req.enableLocalization = opts.allowLand;
  }
  const result = await WPSApi.sendRequest(req);
  if (result.code !== ResultCode.OK) {
    throw new Error(`${result.code}: ${result.msg ?? ''}`);
  }
  return result;
}
```

外部文件先入 `filesDir`。预览与编辑共用入口。不落地开关只在专业版赋值。页面不直接 `new OpenFileRequest`。全仓搜索该构造函数，尽量收敛为一处。

## 联调检查表

1. HAR 文件与申请渠道（pro / personal）一致，clean 重编  
2. `sdkFlavor()` 与预期一致  
3. pro：Token 已设；personal：确认未设  
4. 沙箱只读打开  
5. `enableEdit = true`；（可选）URI 回传并拷贝  
6. 故意未注册确认 catch；正式包包名再验 1013  

| 现象 | 优先查 |
|------|--------|
| 猜包名仍在代码里 | 删掉，改 Constants |
| 1013 | HAR / bundle / key 是否一套 |
| Token 无效 | flavor 与是否调用 `setWpsFileToken` |
| 不落地无效 | 是否 ToC HAR |
| 抛异常 | 未注册就 `sendRequest` |
| 只能预览 | `enableEdit` |

PR 勾选：页面是否 import `SdkConstants`、是否解析 WPS 包名、Release 无密钥、是否走统一 Facade。产品口头「要能打开」时，落到 flavor / 注册 / 沙箱路径。周五正式包再跑 1–5。同一工程多渠道列全 Bundle 名。

新人 Onboarding 顺序：WPSApi 门禁 → `isPersonalSdk()` → `OpenFileRequest` 字段。联调「设了字段没效果」优先查 flavor。若联调机安装多个 WPS 客户端，代码仍只信 Constants，不要加包名「双保险」。

把 flavor 写入崩溃日志（不要写 secret）。客服工单拆层：认错 HAR、1013、路径、Token。策略字段单独 PR，不要和 flavor 分支同一提交。

构建期在 `oh-package.json5` 钉死 HAR 路径；每个 product 使用独立 `CRED` 模块。调试包与正式包包名不同则凭据必须不同。换 HAR 后执行 clean，否则 `sdkFlavor()` 可能仍反映旧包。`OpenFileRequest.wpsToken` 不推荐；漏设专业版 Token 时优先查 adapter 而不是打开参数。页面若需要展示接入类型，只读 `sdkFlavor()`。

真机用例建议命名：`pro-register-ok`、`personal-no-token`、`pro-missing-token`、`unregistered-catch`、`release-bundle-1013`。每步只改一类行为。外部路径先拷贝到 `filesDir`。开启 URI 回传后等待关窗并拷贝 `fileUri`；未开回传时 `data` 为空是正常结果。打开按钮在 `ready` 前禁用。Release 包禁止打印 secret。

Code Review 打回条件：业务 ETS 直接 import `SdkConstants`、存在 WPS 包名字符串判断、忽略 `sendRequest` 的 catch、外部 URI 直传。接入评审三项：HAR 与申请渠道、查询是否单一入口、专业版 OK 后是否 Token。把检查表跑两周，串包类 1013 通常下降。文档字段若有更新，以 WPS 最新交付说明为准，先改 adapter 再改页面。

同一周若必须同时验 ToB 与 ToC，分设备或分日安装对应 WPS 客户端与对应 HAR，不要在同一调试包来回替换 libs 却不 uninstall。反馈邮件注明：`sdkFlavor()`、`bundleName` 后缀、`Result.code` / `msg`、是否已 `setWpsFileToken`（布尔即可）。不要把完整序列号贴到公共 issue。

## 小结

Detecting the current HarmonyOS WPS Open SDK = **`SdkConstants.isPersonalSdk()` in one adapter + Token only on pro + then `sendRequest`**. Do not infer HAR from the WPS client package name or from `wpsUpdateInfo`. Clean after swapping HAR. Re-test `registerApp` on the release `bundleName`.

Keep the checklist for two weeks; mixed-HAR 1013 reports usually drop. Official field semantics: https://365.kdocs.cn/l/clQl5cek2NoT

申请 SDK HAR 与凭据时注明包名与专业版/个人版需求。技术交流 QQ 群：628436767
