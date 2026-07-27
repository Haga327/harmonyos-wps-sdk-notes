# HarmonyOS WPS Open SDK Notes: 从旧版 SDK 迁移到统一版

本笔记面向已在 HarmonyOS 工程接入过 `@wps/wps_sdk`、准备替换为统一版 HAR 的二开同学。统一版在专业版（ToB）能力之上扩容个人版（ToC）打开路径，对外仍保持 `WPSApi` + `OpenFileRequest` 同一模型。迁移重点是交付对齐与封装收拢，而不是重写业务页面。事实依据为官方对接文档，代码示例按可复用 Facade 改写，便于仓库内对照联调与 Code Review。

## 迁移目标与调用链

目标：换 HAR / 凭据后，业务继续走同一套 `prepare` → `open`，用构建变体消化 ToB / ToC 差异。

```
替换 HAR + 对齐 appKey/appSecret/bundleName
        ↓
ohpm install + clean
        ↓
registerApp 成功
        ↓
[ToB] setWpsFileToken
        ↓
OpenFileRequest（沙箱路径）→ sendRequest
        ↓
[可选] 关窗回传 Result.data
```

硬约束不变：未注册成功调用 `sendRequest` 会抛异常。专业版与个人版凭据不可混用。

## 依赖与凭据

```json5
{
  "dependencies": {
    "@wps/wps_sdk": "file:./libs/wps_sdk.har"
  }
}
```

| 检查项 | 说明 |
|--------|------|
| HAR 文件 | 使用统一版交付包覆盖 `libs/wps_sdk.har` |
| 凭据 | 与申请包名、所需版本一致 |
| clean | 避免旧二进制残留导致偶发 1013 |
| 激活序列号 | ToB 通常必须；ToC 不需要；渠道与 SDK 凭据分开 |

运行时用 `SdkConstants.isPersonalSdk()` 区分形态，避免解析 WPS 包名。

## 推荐迁移后的封装

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

export class WpsMigrateClient {
  private ready = false;

  prepare(appKey: string, appSecret: string, proSn?: string): Promise<void> {
    return new Promise((resolve, reject) => {
      if (this.ready) {
        resolve();
        return;
      }
      WPSApi.registerApp(appKey, appSecret, {
        onCallback: (result: Result): void => {
          if (result.code !== ResultCode.OK) {
            reject(new Error(`${result.code}:${result.msg ?? ''}`));
            return;
          }
          if (!SdkConstants.isPersonalSdk() && proSn) {
            WPSApi.setWpsFileToken(proSn);
          }
          this.ready = true;
          resolve();
        }
      });
    });
  }

  async open(
    ctx: common.UIAbilityContext,
    src: string,
    options: {
      edit?: boolean;
      transfer?: boolean;
      allowLocalization?: boolean;
    } = {}
  ): Promise<Result> {
    if (!this.ready) {
      throw new Error('prepare required');
    }
    const dir = `${ctx.filesDir}/wps_migrate`;
    fs.mkdirSync(dir, true);
    const dest = `${dir}/${Date.now()}.docx`;
    fs.copyFileSync(src, dest);

    const req = new OpenFileRequest(ctx, dest);
    req.enableEdit = !!options.edit;
    if (options.transfer) {
      req.wpsTransferType = TransferType.TRANSFER_TYPE_URI;
    }
    if (
      typeof options.allowLocalization === 'boolean' &&
      !SdkConstants.isPersonalSdk()
    ) {
      req.enableLocalization = options.allowLocalization;
    }
    return WPSApi.sendRequest(req);
  }
}
```

从旧代码迁出时：删除 `OpenFileRequest.wpsToken` 逐次赋值；全局 `setWpsFileToken` 与 Request 字段同时存在时以全局为准。页面不要为 ToB/ToC 复制两套 Helper。

## 联调、排错与支持

联调顺序：注册 → 只读 → 可编辑 → 水印/`extraOptions` → 回传 →（ToB）不落地。专业版不落地时，云文档、分享、打印等可能被强制关闭，覆盖 `extraOptions` 中的开启配置。一次堆满开关时，`ResultCode.ERROR` 很难归因，迁移验收应强制「一次只开一类能力」。

| 现象 | 排查 |
|------|------|
| sendRequest 抛异常 | 是否已 registerApp 成功 |
| 1013 AUTH_FAILURE | key/secret、bundleName、HAR 是否同批次 |
| 策略无变化 | 字段是否显式赋值；是否 ToB 且能力生效 |
| 换包后行为怪异 | 是否 clean；是否残留旧 Helper |

从旧分支合并时，注意不要把逐次 `wpsToken` 赋值又带回来。Code Review 可把「全仓 `OpenFileRequest` 构造点数量」作为门禁：理想状态是一处 Facade。

申请与支持：

- 对接文档：https://365.kdocs.cn/l/clQl5cek2NoT
- SDK HAR / 凭据：`m_open_sdk@wps.cn`（注明包名与专业版或个人版）
- 专业版激活序列号：联系 WPS 商务 / 技术支持
- 技术交流 QQ 群：628436767

## 小结

迁移统一版解决的是交付与维护分裂：ToB / ToC 共用 API，差异收敛到 HAR、凭据、序列号与字段生效范围。仓库侧保留一份 Facade，用构建变体注入配置，避免平行打开逻辑。合入前核对对接文档版本与当前 HAR 说明，保证示例与交付包行为一致。

同一 App 若同时维护两种 flavor，UI 文案可以区分场景，但打开链路应共用。回归用例建议复用「注册成功 / 沙箱只读 / 可编辑 / 回传」骨架，再按专业版增量覆盖不落地与菜单强制关闭。这样文档、Demo 与业务工程对齐的是同一套调用面，而不是三份彼此漂移的打开函数。预览与编辑入口也应共用 `open`，仅差布尔与可选策略对象，避免字段在复制粘贴中丢失。从旧代码迁出时，把「是否注入序列号」收进 `prepare`，把「是否允许落地」收进 `open` 的可选参数，页面层就不必再感知 HAR 形态差异。迁移窗口内建议全仓搜索 `wpsToken` 与重复的 `OpenFileRequest` 构造，列成 checklist 逐项关掉。`extraOptions` 仅显式赋值字段生效；未赋值不要假设默认关闭。换 flavor 后务必 clean 再装，核对正式包名与申请凭据一致后再合入主干。若迁移前后能力矩阵有变化，在 PR 描述里写清「哪些字段从 Request 挪到了全局 / 适配层」，方便评审对照联调结果，也减少「看起来换了包、实际还在跑旧 Helper」的情况。
