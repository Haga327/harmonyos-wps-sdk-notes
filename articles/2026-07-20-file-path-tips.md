# HarmonyOS WPS Open SDK Notes: 关闭回传后 filePath 获取

在鸿蒙应用里接入 `@wps/wps_sdk` 之后，打开与编辑往往先完成；真正拖慢联调的是关窗回写。业务上传、版本号、二次打开，都依赖**本应用沙箱内的 filePath**。开启关闭回传后，`ResultData.fileUri` 仍落在 WPS 沙箱，不能直接当持久路径。本文整理请求开启方式、URI/FD 解析顺序、落盘实现要点与联调清单，便于工程内复用。

## Problem

`sendRequest` 在开启 `wpsTransferType` 后，关窗回调里的 `fileUri` 位于 **WPS 沙箱**。若业务直接持久化或上传该 URI，会出现权限失败、进程结束后失效、或正式包表现与调试包不一致。正确做法是拷贝到本应用 `filesDir`，再使用新路径。

同时要区分三种路径：选择器原始路径、交给 `OpenFileRequest` 的打开路径、关窗拷贝后的最终路径。线上「上传旧文件」几乎都是把打开路径误当成最终路径。

## Enable transfer

| Field | Notes |
|-------|--------|
| `enableTransferFile` | `true` → URI style |
| `wpsTransferType` | `TransferType.URI` / `FD`，优先级更高 |

Prefer setting only `wpsTransferType`. Pair with `enableEdit = true` when the user must modify content.

```typescript
const req = new OpenFileRequest(ctx, sandboxPath);
req.enableEdit = true;
req.wpsTransferType = TransferType.URI;
const result = await WPSApi.sendRequest(req);
```

Without transfer fields, `OK` with empty `data` only means WPS launched. UI should not toast “saved to server” in that case.

注册必须先于打开：`registerApp` 未成功时其它请求会 reject，进不了 `result.code`。选择器文件建议先 copy 进 `filesDir` 再打开，避免打开层 ERROR 掩盖回传问题。

## Resolve order

1. Fail if `code !== OK`.  
2. Return `null` if `!data` (opened, not collected).  
3. Prefer FD when `transferFd >= 0`.  
4. Else copy `fileUri` into app sandbox.  
5. Business code must use the resolved path, never the pre-open path.

```typescript
function resolveCallbackFilePath(ctx: UIAbilityContext, result: Result): string | null {
  if (result.code !== ResultCode.OK || !result.data) return null;
  const data = result.data;
  if (data.transferFd !== undefined && data.transferFd >= 0 && data.parameters) {
    return copyFdToSandbox(ctx, data.parameters) ?? null;
  }
  if (data.fileUri) {
    return copyWpsUriToSandbox(ctx, data.fileUri) ?? null;
  }
  return null;
}
```

### URI copy sketch

```typescript
function copyWpsUriToSandbox(ctx: UIAbilityContext, wpsFileUri: string): string {
  const file = fs.openSync(wpsFileUri, fs.OpenMode.READ_ONLY);
  const dir = `${ctx.filesDir}/wps_callback/${Date.now()}/`;
  fs.mkdirSync(dir, true);
  const name = (file.path ?? 'edited.bin').split('/').pop()!;
  const dest = dir + name;
  fs.copyFileSync(file.fd, dest);
  fs.closeSync(file);
  return fileuri.getUriFromPath(dest);
}
```

### FD notes

- Read in chunks (e.g. 64 KiB).  
- Validate against `transferFileSize` when present.  
- Always `closeSync` both local and transfer fds.  
- Optional: run heavy IO on `TaskPool`.  
- Some builds nest FD details under `parameters`; confirm shape before hard-coding.

不落地相关约束若生效，拷贝完成后可按安全策略异步删除 WPS 临时文件；是否删除不要和「有没有拿到 filePath」绑死。

## Pitfalls and UI states

| Symptom | Likely cause |
|---------|----------------|
| Upload is old content | Still using open-time path |
| `OK` but no path | User switched task instead of closing WPS |
| Intermittent open failures | FD not closed |
| Copy throws | Missing `mkdirSync` |
| Truncated file | No size check on FD path |

建议拆分状态：`opened` 与 `collected`。`resolve` 返回 `null` 时提示「请关闭 WPS 完成保存」；有路径后再入上传队列。超时允许取消，避免用户卡在转圈。上传失败重试时继续用最终路径，禁止回退到打开前路径。

## Test matrix and modules

- Small docx / large xlsx  
- URI and FD  
- Close window vs app switch only  
- Re-open resolved path in-app and hash content  
- HAR / flavor change with clean rebuild  

Keep watermark / `extraOptions` orthogonal; verify transfer alone first, then layer other fields.

Suggested layout:

```
wps/
  open.ts       // build OpenFileRequest
  transfer.ts   // resolveCallbackFilePath + copy helpers
  types.ts
```

Pages call `openAndCollect()` → `string | null` only. Code review checklist: single resolve entry, no open-path upload, FD closed, mkdir present, unit tests for empty data / URI / FD / size mismatch.

统一日志前缀 `[WPS][transfer]`，输出 `code/hasData/mode`。Debug 可打印最终路径；Release 可截断敏感目录段。把「无 data」与「拷贝失败」分开计数，便于看回归。

工程上再固定几条：Preview / Edit 共用入口；禁止页面直接 `new OpenFileRequest`；换 HAR 必须 clean；CI 打印 `bundleName` 避免鉴权问题被误判成回传失败。不落地约束若要求最小化残留，在确认上传成功后再删 WPS 临时文件与本应用中间件。表格大文件优先覆盖 FD；Word 小文件用 URI 即可作为日常冒烟。

把 `resolveCallbackFilePath` 做成可单测的纯函数（注入 Mock `Result`），比只在真机点关窗更快暴露分支漏洞。真机矩阵仍不可省：至少覆盖关窗成功、只切回、拷贝目录只读失败三类。字段与枚举以官方对接文档为准，HAR 升级后扫一遍 `ResultData` / `TransferType` 即可。

## Summary

Final `filePath` is an **app-sandbox** artifact produced after close-transfer, not the raw WPS `fileUri`. Declare transfer on the request, resolve once in a shared helper, and feed uploads only with the resolved path. Field semantics follow the official integration doc. After the helper lands in `transfer.ts`，日常需求通常只改上传或命名规则，而不必再打穿每个打开按钮。把打开路径与业务路径拆开，是这类联调问题里最值回票价的一刀。字段与枚举变更时，优先更新 `transfer.ts` 单测，再上真机冒烟关窗链路。

---
> 基于 WPS Open SDK 鸿蒙版对接实践整理，仅供开发者参考。  
> 官方对接文档：https://365.kdocs.cn/l/clQl5cek2NoT  
> 技术交流 QQ 群：628436767
