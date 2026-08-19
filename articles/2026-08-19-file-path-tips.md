# HarmonyOS WPS Open SDK Notes: filePath After Close Transfer

在 HarmonyOS 上接入 `@wps/wps_sdk` 时，对外入口仍是 `WPSApi.sendRequest`。仅打开文档并不等于拿到可上传文件：设置 `wpsTransferType` 之后 Promise 会等到用户关窗，`Result.data.fileUri` 与 `transferFd` 仍位于 WPS 沙箱，必须拷贝到本应用 `filesDir` 才是业务 `filePath`。本文把解析收进 Facade，并给出联调检查表；字段语义以官方对接文档为准。ToC 与 ToB 都要拷贝，不落地场景的临时文件删除只放专业版分支，避免页面散落判断。

## 范围

| 层 | 覆盖点 |
|----|--------|
| 门禁 | `RegisterAppRequest`、1013 |
| 打开 | `OpenFileRequest` + `enableEdit` |
| 回传 | `TransferType.URI` / `FD` |
| 落盘 | copy URI / read FD → 本应用路径 |

未开回传：`ResultCode.OK` 且 `data` 空 = 仅拉起。开了回传却上传构造参数路径 = 旧文件。`wpsTransferType` 优先于 `enableTransferFile`。联调先绿门禁，再打开，最后回传拷贝。一次堆满开关时 ERROR 很难归因。

同一仓库兼 ToB/ToC 时，构建变体注入凭据；页面只调 `openAndCollect`。用 `SdkConstants.isPersonalSdk()` 决定拷贝后是否 `unlink` WPS 临时文件。换 HAR 后 clean。日志：ready、transfer、code、localPath。

## Adapter

```typescript
import {
  WPSApi,
  RegisterAppRequest,
  OpenFileRequest,
  TransferType,
  ResultCode,
  SdkConstants,
} from '@wps/wps_sdk';

export async function openAndCollect(
  ctx: UIAbilityContext,
  path: string,
  opts: { transfer?: 'uri' | 'fd' } = {}
) {
  await ensureWps(ctx, CRED);
  const req = new OpenFileRequest(ctx, path);
  req.enableEdit = true;
  if (opts.transfer === 'fd') {
    req.wpsTransferType = TransferType.FD;
  } else if (opts.transfer === 'uri') {
    req.wpsTransferType = TransferType.URI;
  }
  const result = await WPSApi.sendRequest(req);
  if (result.code !== ResultCode.OK) {
    throw new Error(`${result.code}: ${result.msg ?? ''}`);
  }
  const localPath = resolveLocalPath(ctx, result);
  if (localPath && result.data?.fileUri && !SdkConstants.isPersonalSdk()) {
    fs.unlink(result.data.fileUri, () => {});
  }
  return { result, localPath };
}
```

未注册会抛异常。打开按钮在 ready 前禁用。外部文件先入 `filesDir`。Release 不打印 secret。Token 仅专业版在注册 OK 后注入。

`resolveLocalPath`：合法 `transferFd` 优先，否则 `fileUri` 拷贝，否则 `undefined`。页面禁止直接消费 WPS 路径。全仓搜索把 `fileUri` 传给上传 API 的调用，一律收回 Facade。

## Copy helpers

URI：`openSync` 只读 → `mkdirSync` 带时间戳目录 → `copyFileSync` → `fileuri.getUriFromPath` → close 源。FD：从 `parameters` 取 fd/name/size，64KB 循环读写，size 不一致则失败，close 两端 fd。大文件 TaskPool。

并发打开时目录隔离。清理 WPS 临时文件失败不挡住 `localPath`。未开回传不要提示保存成功。UI 开了 transfer 必须提示关窗。

## 联调检查表

1. 未开回传：OK + 空 data  
2. URI：关窗后拷贝，内容 hash 变化  
3. FD：size 匹配并 close  
4. 旧路径上传应被测试拒绝  
5. 未注册 catch；正式包 1013  

| 现象 | 优先查 |
|------|--------|
| 转圈 | 等待关窗 |
| data 空 | 未设 transfer |
| 旧文件 | 未用 localPath |
| 拷贝失败 | 目录 / URI 可读性 |
| 抛异常 | 未注册 |
| 1013 | HAR / bundle / key |

PR 勾选：页面是否引用 `fileUri`、FD close、Release 无密钥。产品「编辑完要上传」落到：transfer / 关窗 / 拷贝。周五正式包跑 1–2。多渠道列全 Bundle 名。

新人顺序：WPSApi 门禁 → 等待语义 → 拷贝。客服工单拆层。策略字段单独 PR。把 localPath 写入非敏感诊断，不要写 secret。

真机用例：`uri-copy-ok`、`no-transfer-empty-data`、`fd-size-mismatch`、`unregistered-catch`。每步只改一类。换 HAR 后 clean。限时凭据失效重新申请。

同一周验 ToB 与 ToC 时分设备。不落地 unlink 只在 adapter。代码不解析 WPS 包名。文档字段若更新，以 WPS 最新交付说明为准。

构建期钉死 HAR；每个 product 使用独立 `CRED`。调试包与正式包包名不同则凭据必须不同。`OpenFileRequest.wpsToken` 不推荐。页面若展示「已保存」，只读 `localPath`。Ability 启动阶段完成 ensure。打开按钮在 ready 前禁用。外部路径先拷贝到 `filesDir`。只读预览不要设置 transfer。可编辑与预览共用入口，用 opts 区分。

真机用例建议命名：`uri-copy-ok`、`no-transfer-empty-data`、`fd-size-mismatch`、`unregistered-catch`、`release-bundle-1013`。Code Review 打回：业务 ETS 直接拿 `fileUri` 上传、忽略 catch、外部 URI 直传、FD 未 close。接入评审三项：是否开回传、是否拷贝、正式包是否复测注册。把检查表跑两周，旧文件误传通常下降。反馈邮件注明：是否关窗、`Result.code` / `msg`、是否已得到 localPath（布尔即可），不要把完整 secret 贴到公共 issue。

并发两路编辑时拷贝目录必须隔离。unlink WPS 临时文件放在确认 localPath 非空之后。清理失败只记日志。客服工单把「打不开」「转圈」「旧文件」拆层。Onboarding 按门禁 → 打开 → 回传拷贝写，不要从 Demo 一次性复制全部开关。预览不要带 transfer。上传入参只用 localPath。

## 小结

Usable filePath after close = **`wpsTransferType` + wait for close + copy WPS sandbox into app `filesDir`**. Never persist `fileUri`. Handle then/catch. Re-test registerApp on the release bundleName.

Keep the checklist two weeks; “uploaded old file” reports usually drop. Official semantics: https://365.kdocs.cn/l/clQl5cek2NoT 换 HAR 后必须 clean 再编，正式包包名再验注册。

申请 SDK HAR 与凭据时注明包名与专业版/个人版需求。技术交流 QQ 群：628436767
