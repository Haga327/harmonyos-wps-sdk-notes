# HarmonyOS WPS Open SDK Notes: 联调痛点与 registerApp/OpenFileRequest 收敛写法

在鸿蒙工程里接 `@wps/wps_sdk`，卡关往往不是「不会写打开代码」，而是几类现象反复出现：冷启动连点直接 reject、正式包 1013、选择器路径一打开就 ERROR、编辑按钮却只能预览、以及 `OK` 却没有 `data` 被当成失败。本文把这些痛点落到接口语义上，用一套可复用的 TypeScript 封装说明如何收敛。字段含义以官方对接文档为准。

## 痛点地图：现象对齐到接口层

| 联调现象 | 常见根因 | 接口侧落点 |
|----------|----------|------------|
| Promise reject | 进程内尚未 `registerApp` 成功 | 调用时序 |
| `code === 1013` | 凭据 / `bundleName` / HAR 不一致 | `RegisterAppRequest` |
| `code === ERROR` | 路径不可读或参数不完整 | 沙箱路径 |
| 只能预览 | 未显式 `enableEdit = true` | `OpenFileRequest` |
| `OK && !data` | 未配置回传却期待文件 | `wpsTransferType` |

推荐时序保持固定：

```
onCreate → ensureRegistered
按钮 → copyToSandbox → buildOpenRequest → sendRequest
      → then(assertOk) / catch(reject)
```

把「策略参数」与「最小打开」拆开：先保证注册与路径稳定，再叠水印、`extraOptions`、URI/FD 回传。一次堆满参数时，ERROR 很难归因。

## 痛点：未注册就 open

对接文档写得很清楚：`registerApp` 成功之前调用其它 `sendRequest` 会 reject，不会进入 `result.code`。页面里散落 `new OpenFileRequest` 最容易踩这一枪。

```typescript
import { common } from '@kit.AbilityKit';
import {
  WPSApi,
  RegisterAppRequest,
  Result,
  ResultCode,
  SdkConstants,
} from '@wps/wps_sdk';

export let wpsReady = false;

export function logResult(tag: string, r: Result): void {
  console.info(`[${tag}] type=${r.requestType} code=${r.code} msg=${r.msg ?? ''}`);
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
    throw new Error(`register code=${result.code}`);
  }
  // 仅当凭据包需要激活序列号时注入；用 SDK 公开方法分支，勿猜 WPS 包名
  if (!SdkConstants.isPersonalSdk() && ACTIVATION_SN) {
    WPSApi.setWpsFileToken(ACTIVATION_SN);
  }
  wpsReady = true;
}
```

冷启动在 `Application.onCreate` 里 `await ensureRegistered`；首页按钮在 `wpsReady` 前禁用。Release 日志不要打完整 secret。多 flavor 时核对当前 `bundleName` 与申请凭据一致，可消掉大半 1013。

## 痛点：路径 ERROR 与默认只读

系统选择器拿到的 URI 直接塞给 `OpenFileRequest`，联调里非常常见 `ResultCode.ERROR`。建议先 `copyFileSync` 到 `filesDir`，再构造请求。另一类「看起来像 SDK 坏了」的问题是：忘记设 `enableEdit`，文档以只读打开。

```typescript
import fs from '@ohos.file.fs';
import { OpenFileRequest, WPSApi, ResultCode } from '@wps/wps_sdk';

export function copyToSandbox(ctx: common.UIAbilityContext, src: string): string {
  const dir = `${ctx.filesDir}/wps_open`;
  fs.mkdirSync(dir, true);
  const name = src.split('/').pop() ?? 'doc.docx';
  const dest = `${dir}/${Date.now()}_${name}`;
  fs.copyFileSync(src, dest);
  return dest;
}

export type OpenMode = 'preview' | 'edit';

export async function openByMode(
  ctx: common.UIAbilityContext,
  pickerPath: string,
  mode: OpenMode
): Promise<Result> {
  await ensureRegistered(ctx);
  const path = copyToSandbox(ctx, pickerPath);
  const req = new OpenFileRequest(ctx, path);
  req.enableEdit = mode === 'edit';
  const result = await WPSApi.sendRequest(req);
  logResult(mode, result);
  if (result.code === ResultCode.ERROR) {
    throw new Error(`${mode} ERROR: ${result.msg ?? 'check sandbox path'}`);
  }
  if (result.code !== ResultCode.OK) {
    throw new Error(`${mode} code=${result.code}`);
  }
  return result;
}
```

预览与编辑共用入口，只差布尔语义，code review 时一眼能看出有没有漏设 `enableEdit`。拷贝时保留扩展名，避免 open 成功却识别不了格式。

## 痛点：把「拉起成功」当成「回传成功」

未设置 `wpsTransferType` / `enableTransferFile` 时，`sendRequest` 在 WPS 拉起成功后即可返回 `OK`，`data` 常为空。这是轻量打开的预期行为，不是保存失败。需要关窗拿文件时再打开回传：

```typescript
import { TransferType, WaterMark, OpenFileExtraOptions } from '@wps/wps_sdk';

export function decorateAfterLite(req: OpenFileRequest, stamp?: string): void {
  req.wpsTransferType = TransferType.URI;
  if (stamp) {
    const wm = new WaterMark();
    wm.Enable = true;
    wm.WaterMarkText = stamp;
    wm.Angle = -30;
    wm.FontColor = '#19000000';
    wm.FontSize = 16;
    req.wpsWaterMarkParams = wm;
  }
  const opt = new OpenFileExtraOptions();
  opt.enableShare = false;
  opt.enableExport = false;
  req.extraOptions = opt;
}
```

拿到 `result.data.fileUri` 后，用只读 `openSync` + `copyFileSync` 落到本应用目录再上传。切回应用不等于关窗；`OK && !data` 不要当失败 toast。FD 模式再读 `transferFd`，并校验 `transferFileSize`。

## 工程收敛清单

联调按序打勾，比同时改五个参数便宜：

1. 冷启动未注册时 open：应进 `catch`（reject）。
2. 1013：包名、key、HAR 批次；换 HAR 后 clean。
3. ERROR：是否已进 `filesDir`、扩展名是否保留。
4. 只能预览：检查 `enableEdit === true`。
5. 无 `data`：当前是否故意未开回传。
6. Debug 固定打 `requestType/code/msg`；单测 Mock 1013 / ERROR / OK。

激活序列号推荐在注册成功后用 `setWpsFileToken` 全局注入，不要每次 `OpenFileRequest.wpsToken`。若同时设置，以全局为准。不落地相关模式下部分菜单会被强制关闭，以真机为准，不要只信本地 `extraOptions` 赋值。

弱网或企业内网下，可单独回归 `registerApp` 超时与 1013；WPS 客户端升级后抽测只读与可编辑各一次。若业务后续需要断点上传，只需在拿到回传路径后再接上传队列，不必改注册与 `enableEdit` 分支。页面层继续只暴露 `preview` / `edit`，策略细节收在 `buildRequest` 一类函数中，便于单测 Mock `Result`。

再补一条对日志与测试的约定：每条 `sendRequest` 打同一前缀（例如 `[WPS][open]`），检索时不和业务上传日志缠在一起。错误展示给用户用短句（注册失败 / 文件路径无效 / 打开失败），详细 `msg` 进 HiLog。单测至少覆盖 `ensureRegistered` 的 1013、重复调用短路（`wpsReady` 已 true 直接 return），以及 `assertOpenOk` 对 ERROR / OK 的分支。机型覆盖建议至少一台完整版 WPS、一台菜单项可能被裁剪的环境；菜单与预期不一致时优先查不落地模式与 `extraOptions`，而不是改注册。文档字段以对接文档为准，HAR 升级后对照 `OpenFileRequest` 与 `Result` 各扫一遍即可。

工程上我还会固定几条约定，避免团队各自加字段：Preview / Edit 两个入口永远走同一函数；策略类参数只允许在打开模块内部赋值；页面禁止直接 `new OpenFileRequest`。这样 code review 时只需盯注册时序与路径拷贝。Debug 与 Release 用不同 rawfile 存 key，CI 在打包前打印 `bundleName`，减少「调试包正常、正式包 1013」。把入口与断言收进 `wps/` 目录后，换 HAR、改 flavor 凭据时回归范围也更小。

## 小结

鸿蒙侧 WPS Open SDK 的联调痛点，多半可以归到时序、路径、模式与回传语义四类。用 `ensureRegistered` 挡住 reject 与 1013，用沙箱挡住 ERROR，用 `enableEdit` 分清预览/编辑，用是否配置 `wpsTransferType` 解释空 `data`——这条收敛写进工程后，日常需求通常只改水印文案或上传层，不必再打穿整条链路。字段表与枚举以官方对接文档为准，HAR 升级后对照 `OpenFileRequest` / `Result` 扫一遍即可。把入口收束后，多人协作也不容易再次把鉴权逻辑摊回页面生命周期里。

---
> 基于 WPS Open SDK 鸿蒙版实践整理。  
> 官方对接文档：https://365.kdocs.cn/l/clQl5cek2NoT  
> QQ 群：628436767
