# HarmonyOS WPS Open SDK Notes: Demo 工程集成要点

在鸿蒙工程里对照官方快速接入 / Demo「抄」一版打开文档并不难；真正拖慢进度的是工程约定：HAR 怎么管、注册写在哪、文件从哪进沙箱、ToB/ToC 凭据如何隔离。本文把 Demo 调用链收成可复用的仓库约定，便于团队接手与回归。字段语义以官方对接文档为准，示例代码按工程封装改写。

## Problem

Copying Demo button handlers into every page causes concurrent `registerApp` on cold start, mixed Preview / Release secrets, `OpenFileRequest` with non-sandbox paths, and Release logs that print `appSecret`. Docs require registration before other `sendRequest` calls; otherwise the Promise rejects. Treat “SDK ready” as a gate before any open UI.

产品文案也要拆开「已注册」与「已打开」。混在同一个 toast 里，测试同学会把鉴权问题报成打开失败。联调时把注册态和打开态拆开看日志，排障会快很多。

## Layout and HAR

Recommended tree:

```
wps/
  auth.ts       // ensureRegistered
  sandbox.ts    // picker/external → filesDir
  open.ts       // OpenFileRequest
libs/
  wps_sdk.har
```

```json5
{
  "dependencies": {
    "@wps/wps_sdk": "file:./libs/wps_sdk.har"
  }
}
```

Place the delivered HAR under `libs/`, run `ohpm install`. After replacing a HAR batch, clean and reinstall so stale native bits do not shadow new credentials. Record HAR hash, ToB/ToC batch, apply date, and bound `bundleName` internally. Pages only call `openLite`; direct `new RegisterAppRequest` in UI is a review blocker.

ToB and ToC packages differ; credentials are not interchangeable. `SdkConstants.isPersonalSdk()` helps branch behavior, but keys still come from build injection.

## Auth module

```typescript
let ready = false;
let inflight: Promise<void> | null = null;

export async function ensureRegistered(
  ctx: common.UIAbilityContext,
  appKey: string,
  appSecret: string,
  activationSn?: string
): Promise<void> {
  if (ready) return;
  if (inflight) return inflight;
  inflight = (async () => {
    const result = await WPSApi.sendRequest(
      new RegisterAppRequest(ctx, appKey, appSecret)
    );
    if (result.code === ResultCode.ERROR_CODE_AUTH_FAILURE) {
      throw new Error(`auth failure: ${result.msg}`);
    }
    if (result.code !== ResultCode.OK) {
      throw new Error(`register ${result.code}: ${result.msg}`);
    }
    if (activationSn) {
      WPSApi.setWpsFileToken(activationSn);
    }
    ready = true;
  })();
  try {
    await inflight;
  } finally {
    inflight = null;
  }
}
```

For ToB, set the activation SN after OK (channel is usually business/support, not the same as `m_open_sdk@wps.cn`). For ToC, SN is typically unnecessary. Keep one in-flight Promise so double taps share the same registration. Disable open buttons until `ready`. Ability `onCreate` may start registration early, but open entry points must still `await ensureRegistered()`.

## Sandbox open and matrix

```typescript
function copyToSandbox(ctx: common.UIAbilityContext, src: string): string {
  const dir = `${ctx.filesDir}/wps_open`;
  if (!fs.accessSync(dir)) {
    fs.mkdirSync(dir, true);
  }
  const dest = `${dir}/${Date.now()}.docx`;
  fs.copyFileSync(src, dest);
  return dest;
}

export async function openLite(
  ctx: common.UIAbilityContext,
  src: string,
  editable: boolean
) {
  await ensureRegistered(ctx, APP_KEY, APP_SECRET, ACTIVATION_SN);
  const path = copyToSandbox(ctx, src);
  const req = new OpenFileRequest(ctx, path);
  req.enableEdit = editable;
  const result = await WPSApi.sendRequest(req);
  if (result.code !== ResultCode.OK) {
    throw new Error(result.msg ?? `code=${result.code}`);
  }
  return result;
}
```

| Case | Expect |
|------|--------|
| open before register | reject / catch |
| wrong bundle + correct key | 1013 |
| non-sandbox path | ERROR |
| ToC ok credentials, no SN | register OK, can open |
| ToB missing SN (per contract) | auth/open failure exposed early |
| repeated ensure | short-circuit |

Stack watermark / `extraOptions` / `wpsTransferType` only after register + sandbox open are stable. Log prefix: `[WPS][integration]`. CI prints `bundleName`, HAR hash, and whether SN exists (boolean). Never print secrets in Release.

## Boundaries and checklist

Upload retries must not treat auth failures as attachment network errors. Callback file handling (`fileUri` / FD) belongs behind an explicit transfer flag. Key rotation: verify on debug flavor first, then Release. Expired limited credentials and wrong bundle can both look like 1013—log bundle + HAR batch to separate them.

PR checklist: only `auth.ts` touches `RegisterAppRequest`; Release builds strip secret logs; HAR change MRs mention clean; ToB/ToC config comes from the same apply batch as the HAR. Word / spreadsheet samples should each run preview and edit once after a HAR swap.

工程侧再补一条：内部 wiki 固定「申请材料 / 密钥位置 / HAR 批次」三栏，新人先对清单再改 `open.ts`。弱网与冷启动用按钮门禁 + in-flight Promise 双保险后，「点了没反应」类工单会明显下降。Ability `onCreate` 可提前触发注册，但打开入口仍要 `await`，避免竞态。调试包与上架包 Bundle 不同时，两套凭据分目录存放，CI 打印当前 flavor / Bundle / key 后四位。

续期走官方对接文档渠道，补发后同步更新 CI 与本地调试环境。半套残留旧 secret 最容易在商店包才暴露。水印、`extraOptions`、关窗回传都应叠在「注册 + 沙箱打开」之后；一次堆满参数时，`ERROR` 很难归因。把 Demo 的演示价值保留在联调矩阵里，把工程价值落在模块边界上，仓库长期可维护。

若同学问「为啥能打开却没文件回业务库」：先问有没有设 `wpsTransferType`，再问有没有关窗。轻量阶段没开回传时，`data` 为空是预期。开回传后，`OK && !data` 多半是测试只从多任务切回。FD 模式团队若暂时不做，写注释标明「只支持 URI」即可，避免两套半吊子实现。

## Summary

Demo integration on HarmonyOS is three engineering habits: **clean HAR lifecycle, single registration exit, sandbox-before-open**. Keep API facts aligned with the official docs; when renewing keys or changing bundles, re-run the registration and sandbox matrix before touching business upload code. 日常需求尽量只改打开参数或上传接口，把鉴权留在单出口里，是仓库长期可维护的关键。封装稳定后，产品侧再提水印文案或菜单开关，研发只需在已注册链路上下游叠加，不必每次从零打穿 Demo。把打开建立在「已注册」之上，后续叠回传时心态会稳很多。

---
> Based on WPS Open SDK HarmonyOS integration practice for developers.
> Docs: https://365.kdocs.cn/l/clQl5cek2NoT
> Apply HAR/credentials: m_open_sdk@wps.cn
> QQ group: 628436767
