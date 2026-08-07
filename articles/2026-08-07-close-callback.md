# HarmonyOS WPS Open SDK Notes: Close Transfer (URI / FD)

本文记录鸿蒙侧打开文档后，用关闭回传把编辑结果拿回本应用的实践：设置 `enableTransferFile` 或 `wpsTransferType` 后，`sendRequest` 会等待用户关闭文档，再在 `Result.data` 中给出 URI 或 FD。URI 路径位于 WPS 沙箱，必须拷贝到本应用沙箱后才能给业务使用。字段语义以官方对接文档为准。

This note records **close-transfer** on HarmonyOS with WPS Open SDK: after enabling transfer, `sendRequest` waits until the user closes the document, then returns `ResultData` (URI or FD). Copy WPS-sandbox URIs into your app sandbox before business use.

## Placement / 在调用链中的位置

1. `registerApp` → OK  
2. Copy source into app sandbox  
3. `OpenFileRequest` + `enableEdit`  
4. Set `wpsTransferType` (or `enableTransferFile`)  
5. `await sendRequest` → waits for close when transfer is on  
6. Parse `result.data` → copy to app sandbox → optional cleanup  

Without transfer, OK with empty `data` is normal. 未开回传时 OK 且 data 空属正常。

## Fields / 字段

| Field | Role |
|-------|------|
| `enableTransferFile` | Convenience URI transfer |
| `wpsTransferType` | `URI` or `FD` (**higher priority**) |
| `ResultData.fileUri` | WPS-side path (URI mode) |
| `ResultData.transferFd` | FD mode descriptor |
| parameters / file name/size | FD metadata |

Prefer setting only `wpsTransferType` in app code. 封装建议只写类型枚举。

```typescript
req.enableEdit = true;
req.wpsTransferType = TransferType.URI; // or TransferType.FD
```

## Copy helpers / 拷贝

```typescript
function copyWpsUriToSandbox(ctx: UIAbilityContext, wpsFileUri: string): string | undefined {
  const file = fs.openSync(wpsFileUri, fs.OpenMode.READ_ONLY);
  if (!file.path) return undefined;
  const name = file.path.substring(file.path.lastIndexOf('/') + 1);
  const dir = `${ctx.filesDir}/wps_callback/${Date.now()}/`;
  if (!fs.accessSync(dir)) fs.mkdirSync(dir, true);
  const dest = dir + name;
  fs.copyFileSync(file.fd, dest);
  return fileuri.getUriFromPath(dest);
}
```

For FD: read from descriptor into app file, then close fds. 按凭据约定需要清理临时文件时，拷贝成功后再删 WPS 侧文件；删除失败只打日志。

## Facade sketch / 封装示意

```typescript
export async function openAndCollect(
  ctx: UIAbilityContext,
  src: string,
  transfer: 'none' | 'uri' | 'fd' = 'uri'
): Promise<{ result: Result; localPath?: string }> {
  await prepareWps(/* ... */);
  const path = toSandbox(ctx, src);
  const req = new OpenFileRequest(ctx, path);
  req.enableEdit = true;
  if (transfer !== 'none') {
    req.wpsTransferType = transfer === 'fd' ? TransferType.FD : TransferType.URI;
  }
  const result = await WPSApi.sendRequest(req);
  if (transfer === 'none' || result.code !== ResultCode.OK || !result.data) {
    return { result };
  }
  let localPath: string | undefined;
  if (transfer === 'fd' && result.data.parameters) {
    localPath = copyFdToSandbox(ctx, result.data.parameters);
  } else if (result.data.fileUri) {
    localPath = copyWpsUriToSandbox(ctx, result.data.fileUri);
  }
  return { result, localPath };
}
```

Keep a single `new OpenFileRequest` hit in the app. Preview uses `transfer: 'none'`.

## Checklist / 联调清单

| Symptom | Check first |
|---------|-------------|
| Long pending | Transfer on? User closed doc? |
| OK + empty data | Transfer disabled? (OK) |
| URI unreadable later | Copied to app sandbox? |
| FD issues | Valid fd, parameters, closed? |
| `1013` / throw | Register / credentials / clean |

Log: `code` / `msg` / transfer mode / localPath?. After HAR swap, clean. Do not print secrets. Prefer global token after register when required.

Staged QA: editable without transfer → URI + copy → optional FD. UI should show “waiting for close” when transfer is enabled. 分步联调；开回传时 UI 提示等待关窗。

## Pro / Personal packaging notes

Professional and personal packaging share the open API; transfer is request-driven. Keep one Facade; inject keys via flavor. Personal packaging typically skips activation SN; Pro packaging sets SN after register. Cleanup of WPS temp files depends on credential/no-landing agreement.

专业版与个人版共用打开 API；是否回传由 Request 决定。Facade 一份。临时文件清理按凭据与不落地约定执行。

## Summary / 小结

Close transfer is a wait-on-close result path: set type, parse `ResultData`, copy into app sandbox, then hand to business. Official doc is the source of truth.

关闭回传是关窗等待 + 沙箱拷贝。先打开绿，再开回传，再拷贝。禁止业务长期持有 WPS 沙箱路径。路径先落沙箱；换 HAR 后 clean；Release 不打印密钥。

Engineering cadence: Friday case = editable + URI copy success. Fail review if pages use raw `fileUri` without copy. 产品临时要 FD 时只扩分支。超时/取消与产品对齐。拷贝目录按时间戳隔离，防并发覆盖。

When combining watermark, `extraOptions`, and transfer, ship in stages: open → transfer/copy → policy flags. PR lists which transfer mode changed. 与水印、管控同批需求时仍分阶段合入；PR 写清回传模式变更。

Empty transfer config is a common foot-gun: developers expect `data` after open, but never set `wpsTransferType`. Document the wait-on-close UX in the product spec. Use timestamped copy dirs. Close FDs. Do not block the user on temp-file cleanup failure. 未配置回传却期望有 data，是高频误判。产品说明写清关窗等待。拷贝目录带时间戳。FD 必须关闭。清理临时文件失败不阻断主流程。

Keep open, transfer, and copy as three reviewable layers. Fail CI review if business code consumes raw `fileUri`. Friday device case remains editable + URI copy. 打开、回传、拷贝三层可评审；业务禁止直接用原始 `fileUri`；周五固定用例不变。

Add a short UX state machine in the host app: opening → editing (WPS up) → waiting for close → copying → done/failed. Align product and QA on the same states so “waiting for close” is not filed as “open hang”. Return both `result` and `localPath` from the Facade; do not re-parse `ResultData` in every page. If preview-only, pass `transfer: 'none'` explicitly.

页面侧建议有状态机：打开中 → 编辑中 → 等待关窗 → 拷贝中 → 完成/失败。Facade 同时返回 `result` 与 `localPath`。纯预览显式传不回传。日志只打路径后缀与大小。超时与取消写进需求。并发用时间戳目录。把这些坚持几周，回传相关误报通常会下降。细节始终以官方对接文档为准，字段变更时先改封装再改页面。欢迎在仓库 Issue 补充真机时序。

---

Official doc / 官方对接文档: https://365.kdocs.cn/l/clQl5cek2NoT  
Discuss QQ / 交流群: 628436767
