# Menu & Permission Rules (ABP)

> **Cross-cutting rule** — đụng vào C# (`Permissions/*.cs`, `Menus/*MenuContributor.cs`) hoặc JS (FE permission gate). Đọc kèm `AI_PROMPT_BACKEND.md` cho `.cs` và `AI_PROMPT_FRONTEND.md` cho `.js`.

## Purpose

Mỗi feature **phải sinh ra cùng lúc 3 mảnh khớp với nhau**: 1 permission, 1 menu entry gated bởi permission đó, và 1 localization key. Thiếu 1 trong 3 = feature half-released (accessible cho sai người, invisible với đúng người, hoặc không có label).

## Principles

- **Permission first, menu second, label third.** Constant trước → provider trước → menu trước → page trước. Localization key thêm song song khi tạo permission/menu.
- **2-tier menu là default.** Mỗi business module = 1 parent menu item; pages là children. **Tránh 3 cấp** — split module nếu sâu hơn 2.
- **Provider gates the menu; AppService gates the API.** Cả hai phải khớp. Page-level JS còn gọi `abp.auth.isGranted` để hide UI sớm cho user thiếu quyền (defensive, không thay thế server gate).

## Permission constants — nested static-class pattern

Trong `WorkManagement.Application.Contracts/Permissions/WorkManagementPermissions.cs` (hoặc file permission tương ứng của module):

```csharp
public static class <Module>Permissions
{
    public const string GroupName = "<Module>";

    public class <Entity>
    {
        public const string Default = GroupName + ".<Entity>";
        public const string Create  = Default + ".Create";
        public const string Update  = Default + ".Update";
        public const string Delete  = Default + ".Delete";
        // domain-specific actions giữ cùng pattern
        public const string Approve = Default + ".Approve";
        public const string Close   = Default + ".Close";
        public const string Export  = Default + ".Export";
    }
}
```

## Permission provider — AddGroup / AddPermission / AddChild

Trong `WorkManagement.Application.Contracts/Permissions/WorkManagementPermissionDefinitionProvider.cs`:

```csharp
#region <Module>

var <module>Group = context.AddGroup(<Module>Permissions.GroupName, L("Permission:<Module>"));

var <entity>Per = <module>Group.AddPermission(
    <Module>Permissions.<Entity>.Default,
    L("Permission:<Module>.<Entity>"));

<entity>Per.AddChild(<Module>Permissions.<Entity>.Create, L("Permission:<Module>.<Entity>.Create"));
<entity>Per.AddChild(<Module>Permissions.<Entity>.Update, L("Permission:<Module>.<Entity>.Update"));
<entity>Per.AddChild(<Module>Permissions.<Entity>.Delete, L("Permission:<Module>.<Entity>.Delete"));
<entity>Per.AddChild(<Module>Permissions.<Entity>.Approve, L("Permission:<Module>.<Entity>.Approve"));

#endregion <Module>
```

## Menu — 2-tier pattern

Trong `WorkManagement.Web/Menus/WorkManagementMenuContributor.cs`:

```csharp
#region <Module>

var <module>Menu = new ApplicationMenuItem(
    "<Module>",
    l["Menu:<Module>"],
    icon: "fa fa-cogs"
).RequirePermissions(<Module>Permissions.<Entity>.Default);   // parent gated on .Default

<module>Menu.AddItem(new ApplicationMenuItem(
    "<Module>.<Entity>",
    l["Menu:<Module>.<Entity>"],
    url: "/<AreaPath>/<Module>/<Entity>",
    icon: "fa fa-list"
));   // child không cần gate thêm khi cùng permission

<module>Menu.AddItem(new ApplicationMenuItem(
    "<Module>.<Entity>Config",
    l["Menu:<Module>.<Entity>Config"],
    url: "/<AreaPath>/<Module>/<Entity>Config",
    icon: "fa fa-sliders-h"
).RequirePermissions(<Module>Permissions.<Entity>.Update));   // child có gate chặt hơn

context.Menu.AddItem(<module>Menu);

#endregion <Module>
```

## Frontend permission gate (jQuery + ABP)

Trong page-level `.js`, cache permission flags **1 lần ở top file** — KHÔNG gọi `abp.auth.isGranted(...)` inline trong render/cellTemplate vì sẽ chạy lại mỗi lần render:

```js
// ─── PERMISSIONS ─────────────────────────────────────────────────
const canCreate = abp.auth.isGranted('<Module>.<Entity>.Create');
const canUpdate = abp.auth.isGranted('<Module>.<Entity>.Update');
const canDelete = abp.auth.isGranted('<Module>.<Entity>.Delete');
const canApprove = abp.auth.isGranted('<Module>.<Entity>.Approve');

// ─── INIT ─────────────────────────────────────────────────────────
function renderToolbar() {
    if (canCreate) $('#toolbar').append('<button data-act="create">...</button>');
    if (canApprove) $('#toolbar').append('<button data-act="approve">...</button>');
}
```

Razor server-side dùng:

```cshtml
@if (await AuthorizationService.IsGrantedAsync(<Module>Permissions.<Entity>.Approve))
{
    <button>Approve</button>
}
```

## MUST

- Wrap mỗi module's permission block trong provider và mỗi module's menu block trong contributor bằng `#region <Module> ... #endregion <Module>`. Append-only trong region; không reformat siblings.
- Parent menu's `RequirePermissions` = module's `.Default` permission. User có **bất kỳ** child permission nào của module phải thấy được parent → set parent guard là permission **rộng nhất** (`.Default`), để children narrow xuống.
- Localize **cả menu titles** (`l["Menu:<Module>.<Entity>"]`) và **permission labels** (`L("Permission:<Module>.<Entity>")`). Thêm keys vào **mọi** culture file (en + vi + jp tối thiểu cho WM).
- Constant `Module.Entity.Action` chain phải match localization key prefix `Permission:Module.Entity.Action` **chính xác** — mismatch sẽ silently swallow label.
- Dùng cùng icon family với menu hiện tại (Font Awesome `fa-*` trong WM codebase).
- FE: cache permission flag với prefix `can<Action>` (camelCase) ở top file. Gate cả toolbar button, popup footer Save/Delete, action-column cellTemplate, conditional section render.
- AppService method nào public-facing phải có `[Authorize(<Module>Permissions.<Entity>.<Action>)]` — server gate là **bắt buộc**, FE gate chỉ là defensive.

## MUST NOT

- Thêm menu items ngoài `#region` — chúng tụt xuống đáy file và mất thứ tự.
- Dùng child permission làm parent gate. User có `.Approve` nhưng không có `.Default` vẫn phải thấy parent → set parent gate ở permission rộng hơn.
- Hardcode menu titles hoặc permission display strings. **Phải** localize.
- Thêm permission vào constants class **mà không** thêm provider `.AddChild(...)` line tương ứng. ABP cần cả 2 để permission hiện ra trong role management.
- 3 cấp menu lồng nhau. Nếu nghĩ cần thì split parent thành 2 module thay vào đó.
- Dùng icon không có trong menu file hiện tại. Pick từ icons đang dùng để giữ visual consistency.
- Gọi `abp.auth.isGranted(...)` inline trong cellTemplate / render function — bị lặp call mỗi render, lift lên top file.
- Bỏ `[Authorize]` ở AppService method khi refactor — server gate **không bao giờ** được skip.

## Hide-other-menus pattern

Khi release feature cho 1 role cụ thể, **đừng xóa menu items khác**. Thay vào đó:

1. Permission-gate menu items mà role không nên thấy.
2. Cho role chỉ permissions của items role được thấy.
3. Items role thiếu permission → ABP auto-hide.

Nếu phải hide built-in ABP menu (vd. Tenant Management) cho 1 tenant:

```csharp
context.Menu.Items.RemoveAll(item => item.Name == "AbpAccount.Tenant");
```

— làm trong `ConfigureMainMenuAsync` 1 lần, có comment giải thích lý do, không rải rác.

## Examples

### Good — matched permission, menu, và localization

Permission constants:
```csharp
public class Issue
{
    public const string Default  = GroupName + ".Issue";
    public const string Approve  = Default + ".Approve";
    public const string Close    = Default + ".Close";
}
```

Provider:
```csharp
#region ProjectManager
var pmGroup    = context.AddGroup("ProjectManager", L("Permission:ProjectManager"));
var issuePer   = pmGroup.AddPermission(ProjectManagerPermissions.Issue.Default, L("Permission:ProjectManager.Issue"));
issuePer.AddChild(ProjectManagerPermissions.Issue.Approve, L("Permission:ProjectManager.Issue.Approve"));
issuePer.AddChild(ProjectManagerPermissions.Issue.Close,   L("Permission:ProjectManager.Issue.Close"));
#endregion ProjectManager
```

Menu:
```csharp
#region ProjectManager
var pmMenu = new ApplicationMenuItem("ProjectManager", l["Menu:ProjectManager"], icon: "fa fa-folder-open")
    .RequirePermissions(ProjectManagerPermissions.Issue.Default);
pmMenu.AddItem(new ApplicationMenuItem("ProjectManager.Issue",
    l["Menu:ProjectManager.Issue"], url: "/ProjectManager/Issues"));
context.Menu.AddItem(pmMenu);
#endregion ProjectManager
```

Localization (`en.json` + `vi.json` + `jp.json`, inside `//#region ProjectManager`):
```jsonc
"Menu:ProjectManager":              "Project Manager",
"Menu:ProjectManager.Issue":        "Issues",
"Permission:ProjectManager":        "Project Manager",
"Permission:ProjectManager.Issue":  "Issues",
"Permission:ProjectManager.Issue.Approve": "Approve issue",
"Permission:ProjectManager.Issue.Close":   "Close issue",
```

### Bad — orphan permission, hardcoded labels, no region

```csharp
// Permissions.cs — typo: missing .Default
public class Issue { public const string Approve = "ProjectManager.Issue.Approve"; }

// Provider — quên AddChild ⇒ permission không bao giờ hiện ra trong role admin

// MenuContributor.cs — append ở đáy file, không region, hardcoded label
context.Menu.AddItem(new ApplicationMenuItem("Iss", "Issues", url: "/iss"));
//                                                ^^^ không localize, không permission gate
```

## See also

- `AI_PROMPT_BACKEND.md` — `[Authorize]` per method là convention, không gate class-level.
- `LOCALIZATION.md` — JSONC `//#region` discipline, Vietnamese purity rules cho `Menu:`/`Permission:` labels.
- `REFACTOR.md` — region-edit discipline trong shared files (MenuContributor, PermissionProvider).
