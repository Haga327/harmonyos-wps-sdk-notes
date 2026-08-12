# HarmonyOS WPS Open SDK Notes: Access FAQ (Credentials / Flavor / Register)

在 HarmonyOS 上集成 `@wps/wps_sdk` 时，反复出现的问题多半不在「会不会写 OpenFileRequest」，而在交付物配对、双版本差异、注册时序与打开默认值。本文把这些常见疑问收成工程清单与可复用 Facade，字段语义以官方对接文档为准；英文标识仅用于代码与表头，叙述以中文为主，便于团队 Wiki 直接粘贴。

## 范围与分层

| 层级 | 本文覆盖的问题 |
|------|----------------|
| 交付 | HAR + appKey/appSecret + bundleName 是否成对 |
| 版本 | ToB / ToC 行为差异与 `isPersonalSdk` |
| 注册 | `registerApp` 时序、1013、Token 注入 |
| 打开 | `enableEdit`、沙箱路径、ERROR 与 reject |

水印样式、`extraOptions` 细表、FD 边界另见其它日更笔记。把 FAQ 先映射到层级，联调才不会在群里循环「为什么打不开」。

## 版本关系与凭据申请

统一版在保留专业版（ToB）能力基础上新增个人版（ToC）打开支持。申请时仍须明确 HAR/凭据版本；API 对齐，但 Token 与不落地行为不同。

| 项 | 专业版 | 个人版 |
|----|--------|--------|
| 客户端 | WPS 专业版构建 | WPS 个人版 |
| `registerApp` | 必须 | 必须 |
| `setWpsFileToken` | 通常必须 | 不需要 |
| `enableLocalization` | 生效 | 设置无效 |

```typescript
import { SdkConstants } from '@wps/wps_sdk';

export type Flavor = 'pro' | 'personal';

export function detectFlavor(): Flavor {
  return SdkConstants.isPersonalSdk() ? 'personal' : 'pro';
}
```

不要根据桌面图标猜测版本。邮件申请发至 `m_open_sdk@wps.cn`，写清：应用名、精确 Bundle 名、目标版本、联系人、用途。包名变更须重申。专业版激活序列号走商务/技术支持，与 SDK 凭据渠道不同。

| 交付物 | 绑定对象 | 错配时常见表现 |
|--------|----------|----------------|
| HAR | 版本 | 行为不符 / 鉴权噪声 |
| appKey/secret | 包名 | 1013 |
| 激活序列号 | 专业版客户端 | 注册 OK 仍打不开 |

多环境（调试/正式）包名不同则凭据分开。密钥用构建注入，禁止提交明文。

## 注册 Facade 与打开默认值

```typescript
import {
  WPSApi,
  RegisterAppRequest,
  OpenFileRequest,
  ResultCode,
  SdkConstants,
} from '@wps/wps_sdk';

let ready = false;

export async function prepareWps(
  ctx: UIAbilityContext,
  cred: { key: string; secret: string; sn?: string }
): Promise<void> {
  if (ready) return;
  const result = await WPSApi.sendRequest(
    new RegisterAppRequest(ctx, cred.key, cred.secret)
  );
  if (result.code === ResultCode.ERROR_CODE_AUTH_FAILURE) {
    throw new Error(`1013: verify key/secret/bundle/HAR (${result.msg ?? ''})`);
  }
  if (result.code !== ResultCode.OK) {
    throw new Error(`register ${result.code}: ${result.msg ?? ''}`);
  }
  if (!SdkConstants.isPersonalSdk() && cred.sn) {
    WPSApi.setWpsFileToken(cred.sn);
  }
  ready = true;
}

export async function openEditable(
  ctx: UIAbilityContext,
  sandboxPath: string,
  cred: { key: string; secret: string; sn?: string }
) {
  await prepareWps(ctx, cred);
  const req = new OpenFileRequest(ctx, sandboxPath);
  req.enableEdit = true;
  try {
    return await WPSApi.sendRequest(req);
  } catch (e) {
    console.error('sendRequest rejected (register?)', e);
    throw e;
  }
}
```

注册成功前调用 `sendRequest` 会 **reject**，必须 try/catch。Token 推荐在注册 OK 后全局 `setWpsFileToken`，避免每次 `wpsToken`。

| 现象 | 优先原因 | 处理 |
|------|----------|------|
| 只能预览 | `enableEdit` 未设/false | 显式 `true` |
| 路径笼统 ERROR | 外部 URI 权限 | 先入 `filesDir` |
| Promise 抛异常 | 未注册完成 | `await prepareWps` |
| OK 且 `data` 空 | 未开回传 | 未开回传时属正常 |
| 长时间 pending | 开了回传在等关窗 | UI 提示关闭文档 |

预览与编辑共用一个入口，只差布尔。水印 / `extraOptions` / 回传等可编辑打开跑绿后再叠。

## 联调顺序与日志

1. 集成匹配的 HAR，换包后 clean  
2. 冷启动 `prepareWps`  
3. 文件入沙箱  
4. 可编辑打开（先不开回传）  
5. 可选 URI 回传并拷贝出 WPS 沙箱  
6. 用正式包包名复测注册与打开  

日志固定：flavor、是否已设 Token、`enableEdit`、transfer 模式、`code/msg`、包名后缀。Release 不打印 secret。同一仓库双 flavor 时，用构建变体注入密钥，业务页只调 Facade，避免复制两套打开函数。

## 评审门禁与协作

PR 建议勾选：凭据来源（环境/版本）、包名与申请一致、无散落 `new OpenFileRequest`、强制注册后打开、正式包复测 1013。产品问「能否覆盖个人版用户」时，先确认已申请哪套 HAR，再谈能力表——勿口头承诺个人版不落地。

联调反馈模板：当前 flavor、包名末段、是否注册 OK、是否已设 Token、打开参数、`code/msg`。缺项会导致群聊反复猜。客服工单把「打不开」与「不能编辑」拆开，分别对应注册/交付与 `enableEdit`。

新人 Onboarding 可按三天：注册+只读 → 可编辑 → 回传拷贝。比首日堆满策略字段更少挫败。字段变更先改 Facade，再改页面文案。

## 小结

接入 FAQ 大多可收敛为：交付物成对、注册时序正确、打开默认值理解到位。统一 TypeScript 表面减少重复；版本分叉收在 `prepareWps`，而不是每个页面。以官方对接文档为权威，本文作工程检查表。

凭据或 HAR 变更后，每周用商店包包名复测注册与可编辑打开。注释写清「注册完成前禁止打开」「序列号仅在注册成功后注入」。坚持按清单跑两周，1013 与只读类误报通常会下降，对接节奏也会更可预期。

技术交流可对照社区实践与 QQ 群讨论，但上线口径仍以对接文档为准。

---
> Practice notes for HarmonyOS WPS Open SDK integrators.  
> Docs: https://365.kdocs.cn/l/clQl5cek2NoT  
> QQ: 628436767
