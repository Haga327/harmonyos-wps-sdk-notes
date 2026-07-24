# HarmonyOS WPS Open SDK Notes: 统一接口设计（专业版 / 个人版）

本笔记面向在 HarmonyOS 工程中接入 `@wps/wps_sdk` 的二开同学，说明统一版「一套接口覆盖专业版与个人版」的工程含义：对外模型统一，HAR/凭据与少数字段生效范围按场景分叉。事实依据为官方对接文档，代码示例按可复用 Facade 改写，便于仓库内对照联调与 Code Review。

## 背景与调用链

鸿蒙应用要在端内拉起 WPS 打开、编辑本地文档时，接入面应尽量薄。统一版在成熟专业版（ToB）能力之上，扩容了个人版（ToC）打开路径，并在 API 层保持一致：单例 `WPSApi`、`registerApp`、`OpenFileRequest` + `sendRequest`、`Result` / `ResultCode`。

厂商按场景申请对应 SDK HAR 与凭据；业务侧用同一套封装，在适配层处理序列号与「仅专业版生效」的字段。

```
申请 HAR + appKey/appSecret（注明 ToB 或 ToC）
        ↓
ohpm 集成 @wps/wps_sdk
        ↓
registerApp 成功
        ↓
[ToB] setWpsFileToken(激活序列号)
        ↓
OpenFileRequest(沙箱路径) → sendRequest
        ↓
[可选] 关窗回传 Result.data
```

硬约束：`registerApp` 未成功前调用 `sendRequest` 会抛异常。凭据与三方应用 `bundleName` 绑定，专业版与个人版凭据不可混用。

## 双版本差异（同一模型）

| 项 | 专业版 ToB | 个人版 ToC |
|----|------------|------------|
| 客户端 | WPS 专业版等 | WPS 个人版 |
| registerApp | 必须 | 必须 |
| setWpsFileToken | 通常必须 | 不需要 |
| 打开/编辑/水印/回传 | 支持 | 支持 |
| enableLocalization 不落地 | 生效（默认不落地） | 设置不生效 |

运行时用 `SdkConstants.isPersonalSdk()` 区分形态，避免解析 WPS 包名。统一版消掉的是「两套平行 API」；产品仍需按场景选对客户端与 HAR。

专业版默认不落地（`enableLocalization` 未设或为 `false`）时，云文档、分享、打印、导出等可能被强制关闭，即便 `extraOptions` 写成开启也可能无效。允许落地后再用 `extraOptions` 精细关闭菜单。个人版写入 `enableLocalization` 不会触发不落地逻辑。

## 推荐封装

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

export class WpsUnified {
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
    const dir = `${ctx.filesDir}/wps_unified`;
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

说明：页面只依赖 `prepare` / `open`，不要为 ToB/ToC 复制两套 Helper；激活序列号优先全局 `setWpsFileToken`，不推荐每次在 `OpenFileRequest.wpsToken` 赋值；文件先拷贝到应用沙箱再打开，减少权限类泛化错误。构建变体分别注入 key / secret / 可选 sn，运行时仍走同一类。

## 联调、排错与支持

联调顺序建议：注册 → 只读 → 可编辑 → 水印/`extraOptions` → 回传 →（ToB）验证不落地。一次堆满开关时，`ResultCode.ERROR` 很难归因。

| 现象 | 排查 |
|------|------|
| sendRequest 抛异常 | 是否已 registerApp 成功 |
| 1013 AUTH_FAILURE | key/secret、bundleName 是否匹配申请信息 |
| 打开失败 | 沙箱路径、Context |
| 策略无变化 | 字段是否显式赋值；当前是否 ToB 且能力生效 |

换 flavor 后务必 clean 再装，核对正式包名与申请凭据一致。`extraOptions` 仅显式赋值字段生效；未赋值不要假设默认关闭。

申请与支持渠道：

- 对接文档：https://365.kdocs.cn/l/clQl5cek2NoT
- SDK HAR / 凭据：`m_open_sdk@wps.cn`（注明包名与专业版或个人版）
- 专业版激活序列号：联系 WPS 商务 / 技术支持（与 SDK 凭据渠道不同）
- 技术交流 QQ 群：628436767

## 小结

统一接口设计解决的是开发模型分裂：ToB 与 ToC 共用 `WPSApi` 与 `OpenFileRequest`，差异收敛到 HAR、凭据、序列号与字段生效范围。仓库侧建议只保留一份 Facade，用构建变体注入配置，避免平行复制打开逻辑。字段语义以官方对接文档为准，随 SDK 版本核对后再合入业务分支。

同一 App 若同时维护两种 flavor，UI 文案可以区分场景，但打开链路应共用。回归用例建议复用「注册成功 / 沙箱只读 / 可编辑 / 回传」骨架，再按专业版增量覆盖不落地与菜单强制关闭。这样文档、Demo 与业务工程对齐的是同一套调用面，而不是三份彼此漂移的打开函数。预览与编辑入口也应共用 `open`，仅差布尔与可选策略对象，避免字段在复制粘贴中丢失。合入前再核对对接文档版本号与当前 HAR 说明，防止示例与交付包行为不一致。
