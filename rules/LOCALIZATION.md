# Localization Rules (JSONC + Vietnamese purity)

> **Cross-cutting rule** — đụng vào `WorkManagement.Domain.Shared/Localization/WorkManagement/*.json` (en/vi/jp). Đọc kèm `MENU_PERMISSION.md` khi thêm `Menu:` / `Permission:` key, và `AI_PROMPT_BACKEND.md` khi thêm `Enum:` key cho domain enum.

## Folder layout

```
WorkManagement.Domain.Shared/Localization/WorkManagement/
├── en.json   ← English (primary)
├── vi.json   ← Vietnamese (primary)
└── jp.json   ← Japanese (secondary, ABP-fallback OK nhưng nên có)
```

WM dùng **JSONC** (JSON with comments) — `//`-style comments được phép trong `//#region` blocks. ABP parse như JSON sau khi strip comment.

## Key naming convention

| Prefix | Dùng cho | Ví dụ |
|---|---|---|
| `Menu:<Module>.<Entity>` | Menu item label | `Menu:ProjectManager.Issue` |
| `Permission:<Module>.<Entity>.<Action>` | Permission label trong role admin | `Permission:ProjectManager.Issue.Approve` |
| `Enum:<EnumType>:<Value>` | Domain enum display | `Enum:IssueStatus:Open` |
| `Label:<Field>` | Form field label, column caption | `Label:CreatedDate` |
| `Button:<Action>` | Action button text | `Button:Save` |
| `Status:<State>` | Status badge text | `Status:Pending` |
| `Notification:<Event>` | Toast/notification message | `Notification:SaveSuccess` |
| `Error:<Code>` | UserFriendlyException message | `Error:Issue.AlreadyClosed` |
| `Confirm:<Action>` | Confirm dialog message | `Confirm:DeleteIssue` |
| `Tab:<Name>` | Tab title | `Tab:History` |
| `Filter:<Field>` | Filter form label | `Filter:SearchText` |

**Key chain phải match constant chain exactly** — `Permission:ProjectManager.Issue.Approve` chỉ render đúng khi constant là `ProjectManagerPermissions.Issue.Approve = "ProjectManager.Issue.Approve"`. Mismatch = silently swallow label.

## JSONC `//#region` discipline

Mọi key liên quan tới **1 module** phải nằm trong cùng 1 `//#region <Module>` block, không floating ở đáy file:

```jsonc
{
  "culture": "vi",
  "texts": {

    //#region ProjectManager
    "Menu:ProjectManager":             "Quản lý dự án",
    "Menu:ProjectManager.Issue":       "Vấn đề",
    "Permission:ProjectManager.Issue.Approve": "Duyệt vấn đề",
    "Enum:IssueStatus:Open":           "Đang mở",
    "Enum:IssueStatus:Closed":         "Đã đóng",
    //#endregion

    //#region WTSManager
    "Menu:WTSManager":                 "Quản lý bảng giờ",
    //#endregion

  }
}
```

- Append-only trong region: thêm key mới ở **cuối** region, ngay trước `//#endregion`.
- Nếu module chưa có region: tạo region mới ở **cuối** `"texts"`, không chen giữa.
- KHÔNG reformat siblings hoặc đổi key order.
- KHÔNG xóa region wrapper khi xóa key cuối — giữ region rỗng để feature sau add lại.

## MUST

- **Mọi key phải tồn tại ở MỌI culture file** — thêm key vào `en.json` thì phải thêm vào `vi.json` và `jp.json` cùng lúc. ABP fallback en→vi sẽ làm UI mix ngôn ngữ nếu sót.
- Match indent của file hiện tại (WM dùng 2-space).
- Giữ trailing comma trên entry cuối nếu file đang dùng — JSONC chấp nhận, đừng phá style.
- Khi thêm `Permission:...` key, **đồng thời** kiểm tra constant trong `<Project>Permissions.cs` khớp exact chain.
- Khi thêm `Enum:<Type>:<Value>` key, **đồng thời** kiểm tra value khớp với `enum` C# (case-sensitive cho name, không phải số).
- Validate JSONC sau edit: strip `//`-style comment rồi `JSON.parse()` — fail = rollback edit file đó và surface error.

## MUST NOT

- Hardcode user-facing string trong code (`"Vấn đề mới"` trong `.cs` / `.cshtml` / `.js`). **Phải** dùng `L["Menu:..."]` / `@L["..."]` / `abp.localization.localize(...)`.
- Bỏ trống key ở culture phụ — nếu chưa có bản dịch, dùng EN value làm fallback **tạm** và flag rõ trong PR description để translator điền sau. KHÔNG để key vắng mặt ở culture đó.
- Đổi value tiếng Anh của key đã release mà không thông báo — frontend có thể đang text-match (anti-pattern nhưng tồn tại trong code legacy).
- Đặt key floating ở đáy `"texts"` không có region. Mọi key thuộc về 1 module.
- Mix 2 module key trong cùng 1 region. Mỗi region 1 module.

## Vietnamese localization purity (MUST — zero mixing)

**Quy tắc tuyệt đối:** trong `vi.json` và mọi user-facing string render ra cho người dùng cuối tiếng Việt, **CẤM 100% mix tiếng Anh với tiếng Việt** trong cùng 1 cụm từ. Phải Việt hoá triệt để.

### CẤM (Bad)

```jsonc
"Label:CreatedDate":     "Ngày tạo batch",        // ❌ "batch" English
"Label:FileName":        "Tên file",              // ❌ "file" English
"Label:UploadFile":      "Upload tệp",            // ❌ "upload" English
"Label:ItemList":        "Danh sách item",        // ❌ "item" English
"Notification:Done":     "Đã save thành công",    // ❌ "save" English
"Button:ExportExcel":    "Export ra Excel",       // ❌ "export" English (Excel OK — tên riêng)
"Status:Pending":        "Đang pending",          // ❌ "pending" English
"Tab:History":           "Tab lịch sử",           // ❌ "tab" English
"Filter:SearchText":     "Search keyword",        // ❌ cả cụm English nhưng key VI
"Action:Delete":         "Xoá item này?",         // ❌ "item" English
```

### ĐÚNG (Good)

```jsonc
"Label:CreatedDate":     "Ngày tạo nhóm",         // ✓ batch → nhóm/lô
"Label:FileName":        "Tên tệp",               // ✓ file → tệp
"Label:UploadFile":      "Tải tệp lên",           // ✓ upload → tải lên
"Label:ItemList":        "Danh sách mục",         // ✓ item → mục
"Notification:Done":     "Đã lưu thành công",     // ✓ save → lưu
"Button:ExportExcel":    "Xuất ra Excel",         // ✓ export → xuất; Excel giữ (tên riêng)
"Status:Pending":        "Đang chờ duyệt",        // ✓ pending → chờ duyệt
"Tab:History":           "Thẻ lịch sử",           // ✓ tab → thẻ
"Filter:SearchText":     "Tìm theo từ khoá",      // ✓
"Action:Delete":         "Xoá mục này?",          // ✓
```

### Bảng dịch bắt buộc

| English (CẤM) | Tiếng Việt (PHẢI dùng) |
|---|---|
| batch | nhóm / lô |
| file | tệp |
| folder | thư mục |
| upload | tải lên |
| download | tải xuống / tải về |
| import | nhập (từ tệp) |
| export | xuất (ra tệp) |
| item | mục |
| list | danh sách |
| group | nhóm |
| status | trạng thái |
| pending | chờ duyệt / đang chờ |
| approved | đã duyệt |
| rejected | đã từ chối |
| action | thao tác / hành động |
| button | nút |
| tab | thẻ |
| page | trang |
| log | nhật ký |
| dashboard | bảng điều khiển |
| report | báo cáo |
| save | lưu |
| update | cập nhật |
| create | tạo |
| delete / remove | xoá |
| edit | chỉnh sửa / sửa |
| search | tìm kiếm / tìm |
| filter | bộ lọc / lọc |
| reset | đặt lại |
| confirm | xác nhận |
| cancel | huỷ |
| settings | cài đặt |
| user | người dùng |
| role | vai trò |
| permission | quyền |
| login / sign in | đăng nhập |
| logout / sign out | đăng xuất |
| password | mật khẩu |
| username | tên đăng nhập |
| profile | hồ sơ |
| notification | thông báo |
| error | lỗi |
| warning | cảnh báo |
| success | thành công |
| failed | thất bại |
| loading | đang tải |
| processing | đang xử lý |
| total | tổng |
| count | số lượng |
| code | mã |
| name | tên |
| description | mô tả |
| date | ngày |
| time | giờ / thời gian |
| from / to | từ / đến |
| start / end | bắt đầu / kết thúc |
| issue | vấn đề (WM-specific) |
| task / subtask | công việc / công việc con (WM-specific) |
| release | phiên bản phát hành (WM-specific) |
| project | dự án (WM-specific) |
| skill | kỹ năng (WM-specific) |
| timesheet | bảng giờ (WM-specific) |

### Ngoại lệ — được phép giữ nguyên

Chỉ giữ nguyên tiếng Anh khi là **tên riêng / acronym / thuật ngữ kỹ thuật chưa có bản dịch chính thức**:

- **Tên riêng / sản phẩm:** Excel, Word, PDF, Google Drive, SQL Server, PostgreSQL, Docker, GitHub, GitLab, Slack, ABP, OCR, BOM, ERP, MES, ML, AI, DevExtreme, RRC, WTS, SIT.
- **Acronym kỹ thuật quốc tế:** API, URL, ID, JSON, XML, CSV, HTTP, IP, UI, UX, CSS, HTML, DTO, CRUD.
- **Đơn vị kỹ thuật:** KB, MB, GB, ms, px.
- **Dùng nguyên cụm trong dấu ngoặc khi cần làm rõ ngữ cảnh:** `"Mã đơn (Order ID)"` — tránh khi không cần thiết.

### MUST NOT (zero tolerance trong vi.json)

- **CẤM** mix 1 từ tiếng Anh trong câu tiếng Việt — kể cả từ ngắn (`upload`, `save`, `file`, `item`, `tab`).
- **CẤM** dịch nửa vời kiểu `"Tệp file"` (lặp), `"Danh sách list"` (lặp), `"Cập nhật update"` (lặp).
- **CẤM** giữ nguyên tên action button tiếng Anh (`Save`, `Delete`, `Edit`, `Cancel`) trong vi.json — phải dịch (`Lưu`, `Xoá`, `Sửa`, `Huỷ`).
- **CẤM** notification / popup confirm / error message mix Anh-Việt — đặc biệt từ `abp.notify.*` và `UserFriendlyException(L["..."])`.
- **CẤM** column caption / form label / tab title mix — dịch toàn bộ hoặc ngoại lệ rõ ràng theo bảng trên.

### Quy trình kiểm tra khi thêm key vi.json

1. Đọc value tiếng Việt to lên — nếu thấy 1 từ tiếng Anh không nằm trong **bảng ngoại lệ** → SAI, dịch lại.
2. So với vi.json hiện có cùng module — đảm bảo nhất quán terminology (vd. nếu module dùng `nhóm` cho `batch` thì mọi key trong module dùng `nhóm`, không mix `lô`).
3. Nếu phát hiện key cũ trong vi.json vi phạm trong scope task hiện tại → fix luôn (theo scope guard, không refactor ngoài scope).

## Procedure khi thêm localization key mới

1. **List tuples** — mỗi key 1 row: `Key | EN | VI | JP`. Nếu thiếu JP, có thể fallback EN nhưng flag rõ.
2. **Pre-write filter VI** — scan từng VI value qua bảng dịch ở trên, auto-fix các vi phạm phổ biến (`batch → nhóm`, `file → tệp`, ...). Output các fix dạng `[VI fix] "<orig>" → "<fixed>"`.
3. **Edit all 3 files** đồng bộ trong cùng 1 commit — `en.json`, `vi.json`, `jp.json`.
4. **Region placement:** tìm `//#region <Module>` trong mỗi file:
   - Có rồi → append ở cuối region, ngay trước `//#endregion`.
   - Chưa có → tạo region mới ở cuối `"texts"` object (không floating).
5. **Validate JSONC** sau edit từng file: strip `//`-comment rồi `JSON.parse()`. Fail → rollback file đó.
6. **Self-review checklist:**
   - Mọi key tồn tại ở cả 3 file? (no orphans)
   - Permission key chain khớp exact constant chain?
   - JSONC parse thành công?
   - Entries nằm trong region đúng module?
   - VI value pass purity filter?

## Examples

### Good — full coverage cho 1 feature mới

`en.json` (inside `//#region ProjectManager`):
```jsonc
"Menu:ProjectManager.Issue.Close":          "Close issue",
"Permission:ProjectManager.Issue.Close":    "Close issue",
"Button:CloseIssue":                        "Close issue",
"Confirm:CloseIssue":                       "Are you sure you want to close this issue?",
"Notification:IssueClosed":                 "Issue closed successfully",
"Error:Issue.AlreadyClosed":                "This issue is already closed",
```

`vi.json` (inside `//#region ProjectManager`):
```jsonc
"Menu:ProjectManager.Issue.Close":          "Đóng vấn đề",
"Permission:ProjectManager.Issue.Close":    "Đóng vấn đề",
"Button:CloseIssue":                        "Đóng vấn đề",
"Confirm:CloseIssue":                       "Bạn chắc chắn muốn đóng vấn đề này?",
"Notification:IssueClosed":                 "Đã đóng vấn đề thành công",
"Error:Issue.AlreadyClosed":                "Vấn đề này đã được đóng",
```

`jp.json` (inside `//#region ProjectManager`):
```jsonc
"Menu:ProjectManager.Issue.Close":          "課題を閉じる",
"Permission:ProjectManager.Issue.Close":    "課題を閉じる",
"Button:CloseIssue":                        "課題を閉じる",
"Confirm:CloseIssue":                       "この課題を閉じてもよろしいですか？",
"Notification:IssueClosed":                 "課題を閉じました",
"Error:Issue.AlreadyClosed":                "この課題は既に閉じられています",
```

### Bad — orphan key, mix language, hardcoded string

```jsonc
// vi.json — mix Anh-Việt
"Button:CloseIssue":   "Close vấn đề",        // ❌ "Close" English

// en.json — có key, vi.json không có → ABP fallback en, UI mix ngôn ngữ
"Notification:IssueClosed": "Issue closed",  // ✓ en
// vi.json — quên thêm                       // ❌ orphan

// Repository code — hardcode chuỗi
return new { isSuccess = true, data = new { }, msg = "Đóng vấn đề thành công" };  // ❌ phải dùng localizer
```

## See also

- `MENU_PERMISSION.md` — chain `Menu:` / `Permission:` key phải khớp constant.
- `AI_PROMPT_BACKEND.md` — repository dùng localizer khi tạo `msg`; AppService không tạo response.
- `AI_PROMPT_FRONTEND.md` — `abp.localization.localize("Key", "WorkManagement")` từ JS, `@L["Key"]` trong Razor.
