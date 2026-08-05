# HarmonyOS WPS Open SDK Notes: Watermark & Revision on Open

本文记录鸿蒙侧打开文档时配置水印与修订模式的实践要点：二者同属 `OpenFileRequest` 策略层，不另开启动接口。先跑绿注册、沙箱与只读/可编辑，再叠 `WaterMark` / `Revision`；关窗回传仍是独立一层。字段语义以官方对接文档为准，下文示例仅作工程封装参考。

This note records how to attach **watermark** and **revision mode** when opening a document with WPS Open SDK on HarmonyOS. Both belong to the **open policy** layer on `OpenFileRequest`, not separate launch APIs. Field semantics follow the official integration doc.

## Placement in the call chain / 在调用链中的位置

Typical order:

1. `registerApp` → OK  
2. Copy file into app sandbox  
3. `new OpenFileRequest`  
4. Set `enableEdit`  
5. Optionally set `wpsWaterMarkParams` / `wpsRevisionParams`  
6. `sendRequest`  
7. Optionally configure transfer-on-close  

Do not debug watermark/revision before open/read-only is green. 打开层未绿时讨论策略没有意义：未注册完成会抛异常，外部路径权限不足常见泛化 ERROR，二者都容易被误判成「水印没生效」。

Keep open mode and policy as separate concerns in code review. Prefer one Facade that accepts optional policy fields instead of parallel helpers per page.

## WaterMark fields / 水印字段

| Field | Role |
|-------|------|
| `Enable` | Turn watermark on |
| `WaterMaskText` | Text |
| `Angle` | Rotation |
| `FontColor` | Color incl. alpha, e.g. `#19000000` |
| `FontSize` | Size |

Common miss: object created but `Enable` left unset, or never assigned to the request. Very light alpha can also look like “no watermark” during QA—confirm logic with a stronger color first.

```typescript
const wm = new WaterMark();
wm.Enable = true;
wm.WaterMaskText = 'INTERNAL';
wm.Angle = -30;
wm.FontColor = '#19000000';
wm.FontSize = 24;
req.wpsWaterMarkParams = wm;
```

Watermark can combine with read-only preview. 只读预览也可以带水印，不必先开编辑。

## Revision fields / 修订字段

| Field | Role |
|-------|------|
| `UserName` | Author name for revisions |
| `EnterReviseMode` | Open in revise mode |
| `ShowRevisionPanel` | Show panel |
| `EnterRevisionSilent` | Enter without prompt |

Usually set `enableEdit = true` together with revise mode. Read-only + revision often feels “not working” to users.

```typescript
const rev = new Revision();
rev.UserName = 'reviewer';
rev.EnterReviseMode = true;
rev.ShowRevisionPanel = true;
rev.EnterRevisionSilent = true;
req.wpsRevisionParams = rev;
req.enableEdit = true;
```

`EnterRevisionSilent` reduces prompts; for first integration, verify panel behavior with silent off, then enable silent if product requires it. 修订作者名建议与登录态对齐，便于追溯。

## Suggested Facade / 建议封装

Keep one open entry; pass optional policy:

```typescript
export async function openDoc(
  ctx: UIAbilityContext,
  src: string,
  mode: 'preview' | 'edit',
  policy?: { watermark?: string; revisionUser?: string; silent?: boolean }
): Promise<Result> {
  await prepareWps(/* key, secret, optional SN */);
  const path = toSandbox(ctx, src);
  const req = new OpenFileRequest(ctx, path);
  req.enableEdit = mode === 'edit';
  if (policy?.watermark) {
    const wm = new WaterMark();
    wm.Enable = true;
    wm.WaterMaskText = policy.watermark;
    req.wpsWaterMarkParams = wm;
  }
  if (policy?.revisionUser) {
    const rev = new Revision();
    rev.UserName = policy.revisionUser;
    rev.EnterReviseMode = true;
    rev.ShowRevisionPanel = true;
    rev.EnterRevisionSilent = !!policy.silent;
    req.wpsRevisionParams = rev;
  }
  return WPSApi.sendRequest(req);
}
```

Transfer-on-close stays a separate optional concern. 关窗回传与水印/修订无关；未开回传时 OK 且 `data` 为空属正常。全仓 `new OpenFileRequest` 命中保持一处。

## Checklist / 联调清单

| Symptom | Check first |
|---------|-------------|
| Exception | Registration finished? |
| `1013` | Credentials / package name / clean rebuild |
| No watermark | `Enable`, text, assigned to request |
| No revision UX | `enableEdit` + `EnterReviseMode` |
| Generic ERROR | Sandbox path |

Log: `code` / `msg` / ready / sandbox / `enableEdit` / watermark? / revision?. After HAR swap, clean. Do not print secrets in Release. Prefer global token set after register when required.

Suggested staged QA: read-only → read-only + watermark → editable → editable + revision → transfer if needed. 分步联调比一次堆满所有开关更易归因。

## Notes for Pro / Personal packaging

Professional and personal packaging share the open API surface; which policy features are available depends on HAR and credential agreement. Keep one Facade; inject keys via flavor. Personal packaging typically does not rely on activation SN; Pro packaging usually sets SN after successful register.

专业版与个人版共用打开 API；策略能力是否可用取决于 HAR / 凭据约定。Facade 一份，密钥用 flavor 注入。个人版通常不依赖激活序列号；专业版多在注册成功后设置序列号。正式包与调试包包名不同时，凭据分开归档。

## Summary / 小结

Watermark and revision are explicit policy objects on `OpenFileRequest`. Green the open mode first, then stack policy; transfer is separate. Prefer one Facade over duplicated helpers. Official doc is the source of truth for fields.

水印与修订是 Request 上的显式策略对象。先打开模式绿，再叠策略；回传另算。优先统一 Facade，页面只传业务意图。路径先落沙箱；换 HAR 后 clean；Release 不打印密钥。把「先模式后策略」写进联调清单后，策略相关误报通常会下降。字段以官方对接文档为准。

Engineering cadence tip: keep two fixed device cases every Friday—preview with watermark, edit with revision—and fail CI review if a second `OpenFileRequest` constructor appears. 产品临时加预览水印时只扩可选参数，不新开平行 Helper。注释写清约定，比口头「参考 Demo」更耐看。

When product asks for watermark, revision, share locks, and transfer together, still ship in stages: policy fields first, then `extraOptions`, then transfer. Each stage has a clear regression scope. Facade can grow optional args; do not fork a new open helper per switch. PR descriptions should list which policy knobs were added. 产品若同时要求水印、修订、功能开关与回传，仍建议分阶段合入：先策略两字段，再细粒度开关，最后回传。Facade 可扩可选参数，但不要为每个开关复制打开函数。PR 写清本次新增了哪些策略开关。

Log watermark text length instead of full string if content may be sensitive. Align revision author with login identity for audit trails. Path-copy failures should surface a distinct message so QA does not file “watermark missing” for a permission issue. 水印日志可只打长度；修订作者与登录态对齐；路径拷贝失败要有独立提示，避免与策略问题混单。

---

Official doc / 官方对接文档: https://365.kdocs.cn/l/clQl5cek2NoT  
Discuss QQ / 交流群: 628436767
