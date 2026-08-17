# HarmonyOS WPS Open SDK Notes: WPSApi & sendRequest Paradigm

在 HarmonyOS 上接入 `@wps/wps_sdk` 时，对外统一入口是单例 **`WPSApi`**。注册、全局 Token、打开文档都经此入口；`sendRequest` 返回 `Promise<Result>`，未注册成功会抛异常。本文把调用范式映射到可复用 Facade，并给出联调检查表；字段语义以官方对接文档为准。先收稳入口与 Promise 处理，再叠编辑、回传或策略字段，联调会更可归因。

## 范围

| 层级 | 覆盖点 |
|------|--------|
| 入口 | `WPSApi` 单例 |
| 门禁 | `registerApp` / `RegisterAppRequest`、1013 |
| 打开 | `OpenFileRequest` + `sendRequest` |
| 结果 | `Result` / `ResultData`、回传等待语义 |

ToC 通常注册即可打开；ToB 常在注册 OK 后 `setWpsFileToken`。用 `SdkConstants.isPersonalSdk()` 做防御分支。联调原则：先绿门禁，再打开，最后回传。一次堆满开关时，`ResultCode.ERROR` 很难归因。

同一仓库兼 ToB/ToC 时，用构建变体注入凭据；页面层不要复制两套 Promise 链。注释写清「未注册禁止 sendRequest」「必须处理 then/catch」。把范式写进 Onboarding：先讲入口与门禁，再讲字段。联调群里「打不开」优先查 ready，「data 空」优先查是否开回传。

## 注册与 ready 标记

```typescript
import {
  WPSApi,
  RegisterAppRequest,
  OpenFileRequest,
  ResultCode,
  TransferType,
  SdkConstants,
} from '@wps/wps_sdk';

let ready = false;

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

包名与凭据绑定。换 HAR 后 clean。Release 不打印 secret。也可使用回调式 `registerApp`；关键是业务打开前 `ready === true`。打开按钮在 ready 前禁用。限时凭据失效需重新申请。

注册失败不要继续打开。Token 仅在专业版分支注入。Ability 启动阶段完成 ensure，比在点击回调里临时注册更稳。日志固定：`ready`、`isPersonalSdk()`、包名后缀。

## sendRequest Facade

```typescript
export async function openDoc(
  ctx: UIAbilityContext,
  path: string,
  opts: { edit?: boolean; transfer?: 'none' | 'uri' } = {}
) {
  await ensureWps(ctx, CRED);
  const req = new OpenFileRequest(ctx, path);
  req.enableEdit = !!opts.edit;
  if (opts.transfer === 'uri') {
    req.wpsTransferType = TransferType.URI;
  }
  const result = await WPSApi.sendRequest(req);
  if (result.code !== ResultCode.OK) {
    throw new Error(`${result.code}: ${result.msg ?? ''}`);
  }
  // URI: copy result.data.fileUri into app sandbox
  return result;
}
```

外部文件先入 `filesDir`。未开回传时 OK 且 data 空属正常。开了 URI 须等关窗；业务只消费本应用拷贝路径。并发打开时拷贝目录按时间戳隔离。全仓尽量只留一个打开封装。

## 联调与评审

建议顺序：注册 → 未注册 catch 冒烟 → 只读 → 可编辑 →（可选）URI 拷贝 → 正式包复测。

| 现象 | 优先查 |
|------|--------|
| 抛异常 | 未注册 |
| 1013 | key/secret/bundle/HAR |
| 只能预览 | `enableEdit` |
| OK 且 data 空 | 未开回传 |
| 一直转圈 | 回传等待 |

PR 勾选：HAR、包名、Facade、Release 无 secret。产品「要打开」落到：注册？可编辑？要否回传？

日志固定：flavor、ready、`enableEdit`、transfer、`code/msg`、包名后缀。联调反馈缺任一项，群聊容易反复猜。客服工单把「打不开」「不能编辑」「卡住」拆层。

新人按分层 Onboarding：门禁 → 打开 → 回传。真机用例标题写清是否开回传。Code Review 检查是否忽略 catch、是否外部路径直传。混测时把 flavor 写进用例标题，避免政企清单验个人版。

## 小结

调用范式 = `WPSApi` 单例 + 先注册 + `sendRequest` Promise + 按 `Result` 分支。统一 TypeScript API 降低分裂成本；行为差异由 HAR 与字段是否生效决定。以官方对接文档为权威，本文作工程检查表。

每周用商店包包名复测注册与可编辑打开。字段变更先改 Facade。把「未注册禁止 sendRequest」「必须处理 Promise」「回传路径必须先拷贝」写进 Wiki，可减少误报。

技术交流可对照社区实践；上线口径仍以对接文档为准。路径先入沙箱；换 HAR 后 clean。坚持按清单跑两周，入口相关反复提问通常会下降，对接节奏也会更可预期。

再强调：1013 优先核对包名与 HAR；只读与可编辑共用封装；开了回传就要接受等关窗语义。接入评审把对照表贴进联调页，新人合入会稳很多。Release 与调试包密钥分目录存放。范式目标是调用面稳定，策略字段要加就单独建用例与 PR。

把 Facade 覆盖三条冒烟：未注册抛错、OK 且 data 空、URI 拷贝成功。联调页标记当前 flavor 与 ready。串包排查优先重装 HAR。客服工单模板要求带是否已注册、是否开回传、`code/msg`。坚持两周清单，入口类反复提问通常会明显下降。

把对照表与日志字段模板一并放进仓库 docs/，比只存在聊天记录里更可传承。坚持两周清单，入口类反复提问通常会明显下降。
