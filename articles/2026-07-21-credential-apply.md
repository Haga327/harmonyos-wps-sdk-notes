# HarmonyOS WPS Open SDK Notes: 接入凭据与 SDK 申请流程

在鸿蒙应用里接入 `@wps/wps_sdk` 之前，打开样例往往不是最大阻塞；真正拖慢进度的是**接入前门禁**：如何申请 `appKey` / `appSecret` 与 HAR、如何与 Bundle 包名绑定、如何把 `registerApp` 做成可复用模块。本文整理申请材料、交付物清单、注册封装、鉴权失败归因与联调矩阵，便于仓库内复用。

## Problem

未注册成功时调用 `sendRequest` 会 reject；凭据与包名不一致时常见 `ERROR_CODE_AUTH_FAILURE`（文档示例码 1013）。若业务层只处理打开结果、不区分注册态，排障会把鉴权问题误判成路径或客户端版本问题。

另有两套易混淆材料：

| Material | Role |
|----------|------|
| appKey / appSecret | SDK 接入资格 |
| Activation SN | 产品授权（按交付约定；渠道常与凭据邮件不同） |

Teams often paste Demo code into every page. That makes cold-start double registration, scattered 1013 handling, and secret leaks in Release logs much more likely. Prefer one auth module, one in-flight Promise, and explicit UI gating before any open entry is enabled.

Product copy should also split “SDK registered” from “document opened”. Mixing them in toast text trains testers to report the wrong bug class.

## Apply checklist

Email for HAR + credentials (per docs): `m_open_sdk@wps.cn`

Include:

- App name / purpose  
- **Final Bundle name**  
- ToB or ToC (HAR must match; credentials are not interchangeable)  
- Contact  

Activation SN for pro scenarios: contact **WPS business / support**, not the same mailbox as SDK credentials.

Archive the apply receipt together with the expected `bundleName` and HAR batch id. Store secrets in CI or a local vault—never in a public git history. When the debug package and the store package use different bundles, apply twice or explicitly document which credential set belongs to which flavor.

A short internal wiki page that lists “apply materials / secret location / HAR file hash” saves more time than another copy of the demo project.

## Integrate HAR

```json5
{
  "dependencies": {
    "@wps/wps_sdk": "file:./libs/wps_sdk.har"
  }
}
```

Place `wps_sdk.har` under `./libs/`, run `ohpm install`. After replacing HAR, clean the project so old native bits do not shadow new credentials. If types compile but devices still fail auth, suspect a stale binary before you rewrite the open page.
## Register module

```typescript
let ready = false;
let inflight: Promise<void> | null = null;

async function ensureRegistered(
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
      throw new Error(`register failed: ${result.code}`);
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

Prefer global `setWpsFileToken` after OK. Avoid passing `wpsToken` on every `OpenFileRequest`. Share one in-flight Promise so concurrent taps do not start multiple registrations.

UI tip: keep the open button disabled until `ready`, and still `await ensureRegistered()` inside the open path as a second gate.

## Failure matrix

| Symptom | Likely cause |
|---------|----------------|
| reject on open | not registered |
| incomplete params | empty key/secret |
| 1013 | wrong credentials or bundle mismatch |
| expired limited credential | renew via official channel |
| open ERROR after register OK | path / client / request params |
| works on debug, fails on release | flavor injected the wrong secret file |

Log prefix: `[WPS][auth]`. Never print full secrets in Release. Logging `bundleName` + HAR filename + last four chars of the key is usually enough to compare against the apply receipt.
## Flavor, layout, and test matrix

Debug and release bundles often differ. Keep separate credential sets and inject via build profiles. CI should print `bundleName` and HAR hash before packaging.

Suggested layout:

```
wps/
  auth.ts
  sandbox.ts
  open.ts
secrets/   # not committed
libs/wps_sdk.har
```

Test cases:

- Unregistered open → reject  
- Wrong bundle + valid key → 1013  
- Correct ToC path without SN → register OK  
- ToB path missing required SN → fails per product rules  
- Double `ensureRegistered` → short-circuit  
- Wrong flavor secret file → auth failure with matching bundle log  

Then proceed to sandbox path + readonly open + `enableEdit`. Watermark / `extraOptions` / transfer come later. After auth is stable, day-to-day work is usually renewing credentials or switching flavors, not rewriting every open button.

Startup races are common: `onCreate` may still be registering while the open button is already tappable. Disable the button until ready, and always `await ensureRegistered()` inside the open path so concurrent taps share one in-flight Promise.

When rotating keys, validate on the debug bundle first, then cut over release. Both expired timed credentials and wrong bundle names can surface as auth failure—log `bundleName` plus HAR name (never the full secret) so you can tell “typo” from “expired”. Keep upload-retry logic out of auth failures; retries should only cover network errors after a business `filePath` already exists.

PR checklist ideas: only `auth.ts` constructs `RegisterAppRequest`; Release builds strip secret logs; HAR upgrades mention clean; ToB/ToC materials come from the same apply batch. These gates are cheaper than rediscovering 1013 on a device farm.

Also keep product copy honest: “opened editor” is not “registered SDK”. Auth telemetry and open telemetry should use different event names so dashboards do not mix reject noise into open success rates.

Renewal runbook (short): confirm expire date on the apply receipt → request renewal via `m_open_sdk@wps.cn` with the same bundle name → update CI secrets → smoke “register → readonly open” on both debug and release flavors → only then enable edit/transfer suites. If 1013 remains, compare HAR hash before rewriting open parameters.

For multi-app vendors, keep a table of package → credential set → HAR batch. Mixing rows is the usual root cause when “the same key works in project A but not B”.

If you inherit a repo with credentials inlined across pages, migrate in this order: extract `ensureRegistered`, gate every open entry, move secrets to build profiles, then delete duplicate `registerApp` calls. Do not start with watermark or transfer refactors—auth noise will hide the real regressions.

## Summary

Stable callability = matched HAR + bundle-bound credentials + successful `registerApp` (+ SN when required). Keep apply materials, secrets, and HAR batch documented together. Treat registration as a hard gate before any open or transfer work. Field meanings and channels follow the official docs: https://365.kdocs.cn/l/clQl5cek2NoT  

Once the auth module is stable, most follow-up work is flavor switching or credential renewal—not reopening every page that used to call `registerApp` inline. Put that boundary in the README so the next owner does not undo it under schedule pressure. When in doubt, re-run the auth matrix before touching transfer or watermark code. Keep the matrix short enough that CI can run it on every HAR bump without extra drama.

Tech QQ group: 628436767
