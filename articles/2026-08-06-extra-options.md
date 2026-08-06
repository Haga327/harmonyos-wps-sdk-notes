# HarmonyOS WPS Open SDK Notes: OpenFileExtraOptions Feature Flags

本文记录鸿蒙侧打开文档时用 `extraOptions`（`OpenFileExtraOptions`）做细粒度功能管控的实践：分享、打印、另存、云引导等开关同属打开管控层。核心规则是**仅显式赋值生效**，未赋值保持客户端默认。先跑绿注册、沙箱与只读/可编辑，再叠开关；关窗回传仍是独立一层。字段语义以官方对接文档为准。

This note records how to apply **feature flags** via `OpenFileRequest.extraOptions` when opening a document with WPS Open SDK on HarmonyOS. Only **explicitly assigned** properties take effect; unset fields keep WPS defaults. Field semantics follow the official integration doc.

## Placement in the call chain / 在调用链中的位置

Typical order:

1. `registerApp` → OK  
2. Copy file into app sandbox  
3. `new OpenFileRequest`  
4. Set `enableEdit`  
5. Optionally set watermark / revision  
6. Optionally set `extraOptions`  
7. `sendRequest`  
8. Optionally configure transfer-on-close  

Do not debug feature flags before open/read-only is green. 打开层未绿时讨论开关没有意义。

## Rule: unset ≠ false / 未赋值不是 false

Creating `new OpenFileExtraOptions()` without assigning fields does **not** disable share/print/export. You must set the flags you care about to `true` or `false`, then assign the object to `request.extraOptions`.

```typescript
const opt = new OpenFileExtraOptions();
opt.enableShare = false;
opt.enableHomeShareTab = false;
opt.enableSaveAs = false;
opt.enablePrint = false;
opt.enableExport = false;
opt.enableOpenWithExternalApp = false;
req.extraOptions = opt;
```

Leave unrelated fields unset so the client keeps its defaults. 无关字段不要强行写满。

## Field groups / 字段分组

| Intent | Fields |
|--------|--------|
| Block outbound | `enableShare`, `enableHomeShareTab`, `enableOpenWithExternalApp` |
| Block derivative save | `enableSaveAs`, `enableExport`, `document_save` |
| Reduce cloud UX | `enableCloud`, `enableLogin`, `enableAutoUploadDoc` |
| Tighten edit aids | `enableCopy`, `enablePaste`, `enablePrint`, `enableScreenShot` |

`enableCopy` also affects cut. Screenshot may also exist on the request root—set one place only. 完整列表见官方对接文档 OpenFileExtraOptions 一节。

## Suggested presets / 建议预设

```typescript
export type ExtraPreset = 'preview-locked' | 'no-cloud' | 'edit-open';

function buildExtra(preset: ExtraPreset): OpenFileExtraOptions | undefined {
  if (preset === 'edit-open') return undefined;
  const opt = new OpenFileExtraOptions();
  if (preset === 'preview-locked') {
    opt.enableShare = false;
    opt.enableHomeShareTab = false;
    opt.enableSaveAs = false;
    opt.enablePrint = false;
    opt.enableExport = false;
    opt.enableOpenWithExternalApp = false;
  }
  if (preset === 'no-cloud') {
    opt.enableCloud = false;
    opt.enableLogin = false;
    opt.enableAutoUploadDoc = false;
  }
  return opt;
}

export async function openDoc(
  ctx: UIAbilityContext,
  src: string,
  mode: 'preview' | 'edit',
  preset: ExtraPreset = 'edit-open'
): Promise<Result> {
  await prepareWps(/* key, secret, optional SN */);
  const path = toSandbox(ctx, src);
  const req = new OpenFileRequest(ctx, path);
  req.enableEdit = mode === 'edit';
  const extra = buildExtra(preset);
  if (extra) req.extraOptions = extra;
  return WPSApi.sendRequest(req);
}
```

Transfer-on-close stays separate. 关窗回传与管控无关。全仓 `new OpenFileRequest` 命中保持一处。

## Checklist / 联调清单

| Symptom | Check first |
|---------|-------------|
| Exception | Registration finished? |
| `1013` | Credentials / package name / clean rebuild |
| Flags “ignored” | Explicit assignment + assigned to request? |
| Share still works | Both share flags + real menu QA |
| Generic ERROR | Sandbox path |

Log: `code` / `msg` / ready / sandbox / `enableEdit` / preset / assigned keys. After HAR swap, clean. Do not print secrets in Release. Prefer global token set after register when required.

Suggested staged QA: read-only → disable share → disable save-as/export → editable + no-cloud → transfer if needed. 分步联调比一次堆满更易归因。验收必须点开菜单，不要只看代码赋值。

## Notes for Pro / Personal packaging

Professional and personal packaging share the open API surface; which flags are available depends on HAR and credential agreement. Keep one Facade; inject keys via flavor. Personal packaging typically does not rely on activation SN; Pro packaging usually sets SN after successful register.

专业版与个人版共用打开 API；开关是否可用取决于 HAR / 凭据约定。Facade 一份，密钥用 flavor 注入。个人版通常不依赖激活序列号；专业版多在注册成功后设置序列号。正式包与调试包包名不同时，凭据分开归档。

## Summary / 小结

`extraOptions` are explicit policy flags on `OpenFileRequest`. Green the open mode first, then stack flags; transfer is separate. Prefer named presets over duplicated boolean soup. Official doc is the source of truth for fields.

`extraOptions` 是 Request 上的显式管控字段。先打开模式绿，再叠开关；回传另算。优先命名预设，页面只传业务意图。路径先落沙箱；换 HAR 后 clean；Release 不打印密钥。把「先模式后管控」写进联调清单后，相关误报通常会下降。字段以官方对接文档为准。

Engineering cadence: keep two Friday device cases—preview-locked and no-cloud—and fail review if a second `OpenFileRequest` constructor appears. 产品临时加禁止打印时只改预设，不新开平行 Helper。注释写清约定，比口头「参考 Demo」更耐看。

When product asks for watermark, flags, and transfer together, still ship in stages: open mode → `extraOptions` → watermark → transfer. Each stage has a clear regression scope. Facade can grow presets; do not fork a new open helper per boolean. PR descriptions should list which preset changed. 产品若同时要求水印、管控与回传，仍建议分阶段合入。PR 写清本次改了哪个预设。

QA must click real menus for share / save-as / export / print. After disabling copy, verify cut. Path-copy failures should surface a distinct message so QA does not file “flag ignored” for a permission issue. 验收要点菜单；路径拷贝失败要有独立提示。

Empty `OpenFileExtraOptions` is the top foot-gun: it looks configured but changes nothing. Prefer named presets (`preview-locked`, `no-cloud`) and log assigned keys on every open. Merge patches carefully so a second assignment does not wipe earlier flags. 空对象是最高频踩坑：看起来配了其实没关。优先命名预设，并在每次打开日志里打出已赋值键；合并 patch 时避免第二次赋值覆盖已关闭的项。

Keep open mode, flags, watermark, and transfer as four separable layers in code review. If a page needs one extra boolean, extend the preset map—do not paste a new helper. 代码评审里把打开模式、管控、水印、回传拆成四层；页面多要一个布尔时扩展预设表，不要再贴一套 Helper。

---

Official doc / 官方对接文档: https://365.kdocs.cn/l/clQl5cek2NoT  
Discuss QQ / 交流群: 628436767
