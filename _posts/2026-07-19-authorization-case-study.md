---
title: "Kiểm thử Authorization trong SaaS multi-tenant: Case study 5 lỗ hổng"
date: 2026-07-19 22:00:00 +0700
categories: [Web Security, Authorization]
tags: [authorization, broken-access-control, api-security, multi-tenant, idor, bola, mass-assignment]
pin: false
author: idkwhy07
---

> **Lưu ý:** Đây là một case study giả lập. Tên hệ thống, organization, người dùng, request, response và toàn bộ lỗ hổng đều là hư cấu. Bài viết không mô tả lỗ hổng của một sản phẩm thực tế.

## Tóm tắt nhanh

Bài viết mô phỏng một cuộc kiểm thử **Authorization** trên ứng dụng SaaS multi-tenant có tên **Umber Desk 12**.

Toàn bộ HTTP traffic được ghi lại từ một Flask fixture server tự dựng, khoảng 220 dòng code. Server có hai chế độ vulnerable và fixed để tái hiện lỗ hổng, áp dụng bản sửa và chạy lại cùng bộ request.

Năm finding tập trung vào năm authorization boundary khác nhau:

| Authorization boundary | Hành vi được kiểm thử | Kết quả |
|---|---|---|
| Ownership boundary | Analyst đọc note của người dùng khác | Ben đọc được note của Alex |
| Property-level authorization | Analyst cập nhật `review_status` | Ben tự chuyển note sang `approved` |
| Tenant boundary | Manager đọc case cross-tenant | Daniel đọc được case của `org_02` |
| Parent-child relationship | Thay `evidence_id` trong nested route | Endpoint trả evidence của case khác |
| Indirect access path | Export source object cross-tenant | Worker trả evidence của `org_02` dù direct endpoint trả `403` |

Hai test case về ownership boundary và indirect access path được trình bày đầy đủ theo quy trình:

```text
Hiểu policy
    ↓
Tạo baseline hợp lệ
    ↓
Chỉ thay đổi một giá trị ảnh hưởng đến Authorization
    ↓
Quan sát response và trạng thái trên server
    ↓
Dùng tài khoản thứ hai để xác minh độc lập
    ↓
Phân tích root cause, sửa lỗi và retest
```

Ba test case còn lại sử dụng cùng phương pháp nhưng được rút gọn để bài không lặp lại cùng một cấu trúc quá nhiều lần.

## Bối cảnh hệ thống

### Umber Desk 12 là gì?

Umber Desk 12 là một ứng dụng **SaaS multi-tenant** dành cho các nhóm làm product compliance. Hệ thống giúp doanh nghiệp tập hợp, review và export tài liệu trước khi gửi cho auditor hoặc phục vụ một đợt kiểm tra tuân thủ.

- **SaaS**: nhiều khách hàng sử dụng cùng một ứng dụng do nhà cung cấp vận hành.
- **Multi-tenant**: nhiều khách hàng dùng chung hạ tầng, nhưng dữ liệu của từng khách hàng phải được tách biệt về mặt logic.
- Trong bài viết này, mỗi khách hàng được biểu diễn bằng một `organization`.

Ví dụ, một công ty sản xuất thiết bị y tế và một đơn vị vận tải có thể cùng sử dụng Umber Desk 12. Cả hai cùng gọi một API, nhưng người dùng của công ty thứ nhất không được nhìn thấy case, note hoặc evidence của công ty thứ hai.

Đây là yêu cầu bảo mật quan trọng nhất của hệ thống multi-tenant:

> Dùng chung ứng dụng không có nghĩa là dùng chung dữ liệu.

### Workflow cơ bản

Một quy trình nghiệp vụ thông thường diễn ra như sau:

1. Mỗi organization có các thành viên với role `Analyst`, `Manager` hoặc `Owner`.
2. Hồ sơ compliance được lưu dưới dạng `case`.
3. Mỗi case có thể được assign cho một Analyst.
4. Analyst đọc evidence của case được giao và tạo note của riêng mình.
5. Manager hoặc Owner review note và thay đổi `review_status`.
6. Khi cần gửi tài liệu cho auditor, hệ thống tạo một export job chạy nền.

Authorization của hệ thống không thể chỉ dựa vào việc user đã đăng nhập hoặc đang có một role hợp lệ. Server còn phải xác định:

- User là member của organization nào.
- Role đó có hiệu lực trong organization nào.
- Case đang được assign cho ai.
- Note thuộc sở hữu của ai.
- Evidence thực sự thuộc case nào.
- Property nào được phép cập nhật.
- Worker có còn được phép đọc source object tại thời điểm job bắt đầu chạy hay không.

### Ví dụ đơn giản

Ben và Alex đều là Analyst trong `org_01`:

- Ben được giao `case_01` và sở hữu `note_01`.
- Alex được giao `case_02` và sở hữu `note_02`.

Ben có quyền đọc `note_01`, nhưng không được đọc `note_02`. Việc hai người có cùng role và cùng organization không khiến họ có quyền trên cùng một object.

Daniel là Manager của `org_01`, vì vậy Daniel được đọc cả `case_01` và `case_02`. Tuy nhiên, quyền Manager đó chỉ có hiệu lực trong `org_01`. Daniel không được đọc `case_03` thuộc `org_02`.

Hai ví dụ trên thể hiện hai boundary khác nhau:

- Ownership boundary giữa hai user cùng role.
- Tenant boundary giữa hai khách hàng dùng chung hệ thống.

## Môi trường và dữ liệu kiểm thử

Toàn bộ quá trình kiểm thử được thực hiện trên fixture server tại `api.umber-desk-12.test`. Đây là môi trường giả lập được dựng riêng cho case study, không phải một hệ thống thực tế đang hoạt động.

Để dễ theo dõi thay đổi:

- Các resource có thể chỉnh sửa bắt đầu với `version: 1`.
- Export job đầu tiên có ID `job_01`.
- Audit event đầu tiên có ID `event_01`.
- Không có dữ liệu được chia sẻ giữa hai organization.
- Mỗi request và response đều có `X-Request-Id` riêng.

Cấu trúc resource:

```text
Organization
├── Members
├── Cases
│   ├── Notes
│   └── Evidence
└── Export jobs
```

Test fixture gồm hai organization độc lập:

```text
org_01 — Meldran Biomedical Works
├── usr_01 — Emma Carter — Owner
├── usr_02 — Daniel Reed — Manager
├── usr_03 — Ben Miller — Analyst
├── usr_04 — Alex Turner — Analyst
├── case_01 — Valve Recall Review
│   ├── assigned_analyst_id: usr_03
│   ├── note_01 — owner_id: usr_03 — review_status: draft — version: 1
│   └── evidence_01 — case_id: case_01 — version: 1
└── case_02 — Sterility Audit
    ├── assigned_analyst_id: usr_04
    ├── note_02 — owner_id: usr_04 — review_status: draft — version: 1
    └── evidence_02 — case_id: case_02 — version: 1

org_02 — Ternwick Transit Cooperative
├── usr_05 — Maya Collins — Owner
├── usr_06 — Leo Foster — Analyst
└── case_03 — Brake Sensor Certification
    ├── assigned_analyst_id: usr_06
    ├── note_03 — owner_id: usr_06 — review_status: draft — version: 1
    └── evidence_03 — case_id: case_03 — version: 1
```

Ba relationship quan trọng cần ghi nhớ:

```text
Ben  ──được assign──> case_01 ──chứa──> evidence_01
Ben  ──sở hữu───────> note_01

Alex ──được assign──> case_02 ──chứa──> evidence_02
Alex ──sở hữu───────> note_02

Leo  ──được assign──> case_03 ──chứa──> evidence_03
Leo  ──sở hữu───────> note_03
```

`case_01` và `case_02` cùng thuộc `org_01`, nhưng được giao cho hai Analyst khác nhau. `case_03` thuộc `org_02`, vì vậy user của `org_01` phải bị chặn bởi tenant boundary trước khi có thể truy cập object này.

## API surface được sử dụng trong case study

Case study chỉ tập trung vào các API trực tiếp liên quan đến năm Authorization boundary được kiểm thử. Các endpoint khác của hệ thống không được liệt kê vì không tham gia trực tiếp vào finding hoặc quá trình xác minh.

| Chức năng | Endpoint | Authorization rule cần bảo vệ |
|---|---|---|
| Đọc note | `GET /api/v1/notes/{note_id}` | Analyst chỉ được đọc note do chính mình sở hữu; Manager và Owner được đọc note trong tenant của mình |
| Cập nhật note | `PATCH /api/v1/notes/{note_id}` | Analyst chỉ được cập nhật note của mình và chỉ được sửa `title`, `body` |
| Manager đọc case | `GET /api/v1/manager/cases/{case_id}` | Manager và Owner chỉ được đọc case thuộc organization nơi role của họ có hiệu lực |
| Đọc evidence qua nested route | `GET /api/v1/orgs/{org_id}/cases/{case_id}/evidence/{evidence_id}` | Organization, case và evidence phải có relationship hợp lệ; subject phải có quyền trên case |
| Tạo export | `POST /api/v1/exports` | Người tạo export phải có quyền đọc source object |
| Kiểm tra trạng thái export | `GET /api/v1/exports/{job_id}` | Chỉ actor hợp lệ được xem trạng thái job |
| Tải file export | `GET /api/v1/exports/{job_id}/download` | Quyền trên job và source object phải còn hợp lệ tại thời điểm tải |

Các endpoint đăng nhập, `/api/v1/me`, review queue, direct evidence endpoint và audit stream được dùng để chuẩn bị authenticated session, thiết lập baseline hoặc xác minh server-side state. Chúng không phải access path bị khai thác trực tiếp trong phần lớn finding, nhưng vẫn đóng vai trò positive control hoặc cung cấp bằng chứng độc lập khi cần.

## Session xác định danh tính, Authorization quyết định quyền truy cập

Ứng dụng sử dụng **opaque server-side session**. Khi đăng nhập thành công, trình duyệt nhận một session cookie. Server dùng cookie đó để xác định user, organization mà user đang tham gia, role của user trong từng organization và trạng thái thành viên hiện tại.

Session chỉ giúp server biết request đến từ ai. Nó không tự động cấp quyền trên case, note, evidence hoặc export.

- **Authentication** trả lời: “Người gửi request là ai?”
- **Authorization** trả lời: “Người đó có được thực hiện hành động này trên resource cụ thể này, trong context hiện tại hay không?”

Ví dụ, Ben đã đăng nhập hợp lệ, nhưng với từng request server vẫn phải xác định Ben có được đọc note đó không, có sở hữu note đó không, có được sửa property được gửi lên không, case chứa evidence có được assign cho Ben không và resource có thuộc organization của Ben không. Authenticated request chỉ xác nhận identity; nó không trả lời bất kỳ câu hỏi Authorization nào trong số này.

## Authorization policy

Authorization policy của Umber Desk 12 được rút gọn thành các quy tắc sau:

1. Analyst chỉ được đọc case được assign cho chính mình trong organization của họ.
2. Analyst chỉ được đọc evidence thuộc case được assign cho họ.
3. Analyst chỉ được tạo, đọc và cập nhật note do chính mình sở hữu.
4. Với note của mình, Analyst chỉ được sửa `title` và `body`.
5. Chỉ Manager hoặc Owner mới được thay đổi `review_status`, approve hoặc reject note cùng tenant.
6. Manager được đọc mọi case, note và evidence trong organization của mình.
7. Không role nào tự động có quyền truy cập organization khác.
8. Nested route chỉ hợp lệ khi toàn bộ relationship trong URL đều đúng.
9. Export phải áp dụng cùng object policy như direct endpoint.
10. Theo policy của hệ thống này, API và worker đều phải dùng thông tin thành viên, role và resource state hiện tại để authorize.

Mô hình Authorization kết hợp ba nhóm điều kiện:

- **RBAC**: quyền phụ thuộc vào role.
- **ABAC**: quyền phụ thuộc vào các thuộc tính như organization, owner, assignment và trạng thái.
- **Relationship-based access control**: quyền phụ thuộc vào quan hệ giữa organization, case, note và evidence.

Một authorization decision chỉ đầy đủ khi server có đủ `subject + action + resource + context`. Ví dụ, `role == "manager"` là chưa đủ nếu server không kiểm tra case đang được đọc có thuộc organization mà role đó có hiệu lực hay không. Tương tự, xác nhận Ben sở hữu một note vẫn chưa đủ nếu backend cho phép Ben cập nhật `review_status`, là property chỉ dành cho Manager hoặc Owner.

## Authorization matrix

Matrix được viết trước khi chỉnh sửa request. Đây là bước quan trọng vì một response `200 OK` chỉ là lỗ hổng khi product policy yêu cầu request đó phải bị từ chối.

| Subject | Action | Resource | Điều kiện | Kết quả mong đợi |
|---|---|---|---|---|
| Ben, Analyst `org_01` | Read | `note_01` | Ben sở hữu note | Allow |
| Ben, Analyst `org_01` | Read | `note_02` | Alex sở hữu note | Deny |
| Ben, Analyst `org_01` | Update `body` | `note_01` | Ben sở hữu note; property hợp lệ | Allow |
| Ben, Analyst `org_01` | Update `review_status` | `note_01` | Property chỉ dành cho Manager/Owner | Deny |
| Ben, Analyst `org_01` | Read | `evidence_01` | Evidence thuộc case được assign cho Ben | Allow |
| Ben, Analyst `org_01` | Read | `evidence_02` | Evidence thuộc case của Alex | Deny |
| Ben, Analyst `org_01` | Read | `evidence_02` qua path `case_01` | Child không thuộc parent trong URL | Deny |
| Ben, Analyst `org_01` | Export | `evidence_01` | Ben được phép đọc source object | Allow |
| Ben, Analyst `org_01` | Export | `evidence_03` | Source object thuộc tenant khác | Deny |
| Daniel, Manager `org_01` | Read | `case_01` | Case cùng tenant | Allow |
| Daniel, Manager `org_01` | Read | `case_02` | Case cùng tenant | Allow |
| Daniel, Manager `org_01` | Read | `case_03` | Case khác tenant | Deny |
| Daniel, Manager `org_01` | Approve | `note_01` | Note cùng tenant | Allow |
| Leo, Analyst `org_02` | Read | `case_03` | Case được assign cho Leo | Allow |
| Maya, Owner `org_02` | Read | `case_03` | Case thuộc tenant của Maya | Allow |

Matrix tách riêng hai câu hỏi thường bị gộp nhầm:

1. Subject có được phép sử dụng chức năng này không?
2. Subject có được phép sử dụng chức năng đó trên **resource cụ thể này** không?

Ben được phép dùng chức năng update note, nhưng Ben chỉ được update note của chính mình và chỉ được sửa các property đã được policy cho phép.

## Chuẩn bị authenticated session

Ben đăng nhập để tạo session dùng trong quá trình kiểm thử:

```http
POST /api/v1/session HTTP/1.1
Host: api.umber-desk-12.test
Content-Type: application/json

{
  "email": "ben.miller@meldran.test",
  "password": "fixture-password"
}
```

```http
HTTP/1.1 201 Created
Set-Cookie: umberdesk12_session=sess_bm_1; Path=/; HttpOnly; Secure; SameSite=Lax
X-Request-Id: req_01

{
  "user_id": "usr_03",
  "session_state": "fully_authenticated"
}
```

Các session còn lại:

| Người dùng | Role | Session |
|---|---|---|
| Emma Carter | Owner `org_01` | `sess_ec_1` |
| Daniel Reed | Manager `org_01` | `sess_dr_1` |
| Ben Miller | Analyst `org_01` | `sess_bm_1` |
| Alex Turner | Analyst `org_01` | `sess_at_1` |
| Maya Collins | Owner `org_02` | `sess_mc_1` |
| Leo Foster | Analyst `org_02` | `sess_lf_1` |

Trong mỗi test, session của actor được giữ nguyên. Chỉ một giá trị ảnh hưởng đến Authorization, chẳng hạn resource ID hoặc property trong request, được thay đổi tại một thời điểm.

## Ownership boundary — Analyst đọc note của người dùng khác

Mục tiêu của test đầu tiên là xác định endpoint đọc note có kiểm tra ownership hay chỉ xác nhận người gọi là member đang hoạt động trong organization chứa note. Nếu API coi mọi Analyst trong cùng tenant là tương đương, Ben có thể đọc note của Alex dù hai người sở hữu các object khác nhau.

**Giả thuyết:** Endpoint có thể truy vấn note bằng ID do client cung cấp, sau đó chỉ kiểm tra người gọi có phải là member trong organization của note hay không. Nếu thiếu ownership check, Ben sẽ đọc được `note_02` mà không cần đổi session, role, tenant, HTTP method hoặc header.

**Baseline hợp lệ:** Ben đọc note của chính mình.

```http
GET /api/v1/notes/note_01 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Accept: application/json
```

```http
HTTP/1.1 200 OK
ETag: "note_01-v1"
X-Request-Id: req_03

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

Đây là positive control hợp lệ: `usr_03` đang đọc note do chính `usr_03` sở hữu trong `org_01`.

**Request đã chỉnh sửa:** Tôi giữ nguyên toàn bộ request và chỉ đổi object ID từ `note_01` thành `note_02`.

```http
GET /api/v1/notes/note_02 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Accept: application/json
```

**Kết quả:** API trả về note của Alex.

```http
HTTP/1.1 200 OK
ETag: "note_02-v1"
X-Request-Id: req_04

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

Ben và Alex có cùng role Analyst, nhưng Ben đã vượt qua ownership boundary để đọc resource của một Analyst khác. Đây là **horizontal object-level authorization failure**.

**Xác minh độc lập:** Alex cập nhật `note_02` bằng session của chính mình.

```http
PATCH /api/v1/notes/note_02 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_at_1
Content-Type: application/json
If-Match: "note_02-v1"

{
  "body": "Lot 7 requires a second sample. Verification marker: DP-1."
}
```

```http
HTTP/1.1 200 OK
ETag: "note_02-v2"
X-Request-Id: req_05

{
  "id": "note_02",
  "owner_id": "usr_04",
  "body": "Lot 7 requires a second sample. Verification marker: DP-1.",
  "review_status": "draft",
  "version": 2
}
```

Ben lặp lại unauthorized request và nhận đúng marker mới cùng `version: 2`:

```http
GET /api/v1/notes/note_02 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Accept: application/json
```

```http
HTTP/1.1 200 OK
ETag: "note_02-v2"
X-Request-Id: req_06

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

Kết quả xác nhận endpoint đang trả về trạng thái hiện tại của object trên server, không phải cache cũ hoặc dữ liệu được dựng riêng cho response. Lỗi này thường được gọi là IDOR, nhưng ID dễ đoán không phải nguyên nhân gốc. Dù dùng UUID, lỗ hổng vẫn tồn tại nếu server truy vấn object theo ID nhưng không kiểm tra relationship giữa subject và note.

**Tác động:** Một Analyst có thể đọc nội dung note thuộc người dùng khác trong cùng tenant. Tùy dữ liệu được lưu trong note, lỗ hổng có thể làm lộ kết quả phân tích, nhận xét nội bộ hoặc thông tin compliance chưa được công bố.

## Property-level authorization — Analyst tự approve note

Update endpoint kiểm tra đúng việc Ben có sở hữu `note_01`, nhưng có thể tự động bind toàn bộ JSON body vào model. Nếu `review_status` không được allowlist theo role, Analyst có thể tự thực hiện state transition chỉ dành cho Manager hoặc Owner.

**Baseline và request đã chỉnh sửa:** Ben cập nhật `body` của `note_01` và nhận `200 OK`, đúng với policy. Sau đó, tôi giữ nguyên session, object và endpoint, chỉ thêm property `review_status` vào JSON body.

```http
PATCH /api/v1/notes/note_01 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Content-Type: application/json
If-Match: "note_01-v2"

{
  "body": "Seal inspection complete.",
  "review_status": "approved"
}
```

**Kết quả:** Server vẫn chấp nhận request.

```http
HTTP/1.1 200 OK
ETag: "note_01-v3"
X-Request-Id: req_08

{
  "id": "note_01",
  "owner_id": "usr_03",
  "review_status": "approved",
  "version": 3
}
```

Object-level authorization ở đây hoạt động đúng vì Ben thực sự sở hữu `note_01`. Lỗi nằm ở property-level: Ben được phép cập nhật note nhưng không được phép ghi `review_status`. Vì server vẫn chấp nhận property này, Analyst có thể thực hiện state transition chỉ dành cho Manager hoặc Owner, tạo thành **vertical privilege escalation qua mass assignment**.

Daniel đọc cùng note qua review controller và xác nhận:

```text
review_status: approved
status_changed_by: usr_03
version: 3
```

Frontend không hiển thị nút approve cho Analyst, nhưng giới hạn giao diện không phải access control. Khi backend vẫn chấp nhận trực tiếp `review_status`, user có thể tự tạo request bằng Burp Suite, curl hoặc một HTTP client khác.

**Tác động:** Analyst có thể tự đánh dấu nội dung của mình là đã được review, làm sai workflow phê duyệt và khiến dữ liệu chưa được Manager kiểm tra xuất hiện như một kết quả hợp lệ.

## Tenant boundary — Manager đọc case cross-tenant

Manager endpoint có thể chỉ kiểm tra `role in {manager, owner}`, sau đó truy vấn case bằng global ID. Nếu query không được scope theo organization nơi role đó có hiệu lực, Daniel có thể đọc `case_03` thuộc `org_02`.

**Baseline và request đã chỉnh sửa:** Daniel đọc `case_01` thuộc `org_01` và nhận `200 OK`. Sau đó, tôi giữ nguyên session, route và method, chỉ đổi `case_id` từ `case_01` sang `case_03`.

```http
GET /api/v1/manager/cases/case_03 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_dr_1
```

**Kết quả:** Endpoint trả về case của organization khác.

```http
HTTP/1.1 200 OK
ETag: "case_03-v1"
X-Request-Id: req_11

{
  "id": "case_03",
  "organization_id": "org_02",
  "title": "Brake Sensor Certification",
  "assigned_analyst_id": "usr_06",
  "version": 1
}
```

Daniel đúng là Manager, nhưng quyền Manager của Daniel chỉ có hiệu lực trong `org_01`. Endpoint đã kiểm tra role nhưng không kiểm tra case có thuộc đúng organization nơi role đó có hiệu lực hay không. Maya, Owner của `org_02`, sau đó cập nhật `case_03`; khi Daniel lặp lại request, response chứa marker và version mới, xác nhận endpoint đang trả về trạng thái hiện tại của object thuộc tenant khác.

**Tác động:** Một Manager có thể đọc hồ sơ compliance của khách hàng khác. Đây là cross-tenant data exposure và có thể phá vỡ yêu cầu cô lập dữ liệu cốt lõi của hệ thống SaaS multi-tenant.

## Parent-child relationship — Nested route trả về evidence của case khác

Controller có thể authorize Ben dựa trên `org_01` và `case_01`, nhưng truy vấn evidence cuối cùng bằng một global `evidence_id`. Nếu server không kiểm tra `evidence.case_id == case_id`, một child không được phép truy cập có thể “đi ké” dưới path của một parent hợp lệ.

**Baseline và request đã chỉnh sửa:** Ben trước tiên đọc `evidence_01` qua đúng path.

```http
GET /api/v1/orgs/org_01/cases/case_01/evidence/evidence_01 HTTP/1.1
```

Response trả `200 OK`. Sau đó, tôi giữ nguyên `org_01` và `case_01`, chỉ đổi ID cuối path từ `evidence_01` sang `evidence_02`:

```http
GET /api/v1/orgs/org_01/cases/case_01/evidence/evidence_02 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
```

**Kết quả:** Endpoint vẫn trả về evidence.

```http
HTTP/1.1 200 OK
ETag: "evidence_02-v1"
X-Request-Id: req_15

{
  "id": "evidence_02",
  "organization_id": "org_01",
  "case_id": "case_02",
  "label": "Sterility sample manifest",
  "version": 1
}
```

Response tự cho thấy relationship không khớp:

- URL chứa `case_01`.
- Object được trả về có `case_id: case_02`.

Controller đã authorize parent path nhưng chưa xác nhận child thực sự thuộc parent đó. Alex sau đó cập nhật `evidence_02` qua đúng path của `case_02`; khi Ben lặp lại request với path sai, response chứa label và version mới, xác nhận evidence thực tế thuộc một case khác.

**Tác động:** Analyst có thể đọc evidence của case không được assign cho mình bằng cách đặt `evidence_id` trái phép dưới một nested path hợp lệ.

## Indirect access path — Export worker đọc evidence cross-tenant

Test cuối cùng kiểm tra cùng một object policy có được áp dụng nhất quán ở cả direct endpoint và export worker hay không. Direct evidence endpoint đã kiểm tra assignment và tenant, trong khi export API tạo một job để worker xử lý sau.

**Giả thuyết:** Export API có thể chỉ kiểm tra Ben đã đăng nhập và có quyền tạo export job, sau đó chuyển `source_id` cho worker. Nếu worker truy vấn evidence bằng global ID mà không authorize source object, direct endpoint có thể từ chối `evidence_03` nhưng export vẫn đọc và trả về object đó.

**Baseline hợp lệ:** Ben tạo export cho `evidence_01`, là object Ben được phép đọc.

```http
POST /api/v1/exports HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Content-Type: application/json

{
  "format": "json",
  "source_type": "evidence",
  "source_id": "evidence_01"
}
```

```http
HTTP/1.1 202 Accepted
Location: /api/v1/exports/job_01
X-Request-Id: req_18

{
  "job_id": "job_01",
  "state": "queued",
  "version": 1
}
```

Job hoàn tất bình thường:

```http
GET /api/v1/exports/job_01 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
```

```http
HTTP/1.1 200 OK
X-Request-Id: req_19

{
  "job_id": "job_01",
  "state": "completed",
  "source_id": "evidence_01",
  "download_path": "/api/v1/exports/job_01/download",
  "version": 2
}
```

**Request đã chỉnh sửa:** Tôi giữ nguyên session, endpoint, format và `source_type`. Giá trị duy nhất được thay đổi là `source_id`.

```http
POST /api/v1/exports HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Content-Type: application/json

{
  "format": "json",
  "source_type": "evidence",
  "source_id": "evidence_03"
}
```

`evidence_03` thuộc `org_02`, trong khi Ben chỉ là member của `org_01`.

**Kết quả:** API vẫn tạo `job_02`.

```http
HTTP/1.1 202 Accepted
Location: /api/v1/exports/job_02
X-Request-Id: req_20

{
  "job_id": "job_02",
  "state": "queued",
  "version": 1
}
```

Worker sau đó hoàn tất job:

```http
GET /api/v1/exports/job_02 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
```

```http
HTTP/1.1 200 OK
X-Request-Id: req_21

{
  "job_id": "job_02",
  "state": "completed",
  "source_id": "evidence_03",
  "download_path": "/api/v1/exports/job_02/download",
  "version": 2
}
```

File download chứa toàn bộ evidence record của `org_02`:

```http
GET /api/v1/exports/job_02/download HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Disposition: attachment; filename="evidence_03.json"
X-Request-Id: req_22

{
  "id": "evidence_03",
  "organization_id": "org_02",
  "case_id": "case_03",
  "label": "Brake sensor calibration log",
  "sha256": "fixture-evidence-03",
  "version": 1
}
```

Trong khi đó, direct endpoint áp dụng đúng policy và từ chối cùng object với cùng session:

```http
GET /api/v1/evidence/evidence_03 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_bm_1
Accept: application/json
```

```http
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json
X-Request-Id: req_23

{
  "title": "Forbidden",
  "status": 403
}
```

Với cùng một actor và cùng một source object, direct path trả `403` nhưng indirect path qua export worker lại trả dữ liệu. Điều này cho thấy authorization policy đã tồn tại nhưng không được áp dụng nhất quán giữa các code path.

**Xác minh độc lập:** Maya truy vấn audit stream của `org_02`.

```http
GET /api/v1/orgs/org_02/audit-events?resource_id=evidence_03 HTTP/1.1
Host: api.umber-desk-12.test
Cookie: umberdesk12_session=sess_mc_1
Accept: application/json
```

```http
HTTP/1.1 200 OK
X-Request-Id: req_24

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

Audit state của tenant thứ hai xác nhận worker đã xử lý một protected object cho actor thuộc tenant khác.

**Tác động:** Attacker có thể dùng export như một đường truy cập thay thế để lấy dữ liệu mà direct endpoint đã từ chối. Vì export thường trả về dữ liệu đầy đủ hơn màn hình thông thường, tác động có thể lớn hơn một lỗi đọc object đơn lẻ.

## Tổng hợp finding

| Boundary | Request chứng minh lỗi | Hành vi mong đợi | Hành vi thực tế | Phân loại |
|---|---|---|---|---|
| Ownership boundary | Ben gọi `GET /api/v1/notes/note_02` | Deny | Trả trạng thái hiện tại của note thuộc Alex | Horizontal object-level authorization failure |
| Property-level authorization | Ben gửi `review_status: approved` khi update `note_01` | Từ chối property | Chuyển note sang `approved` | Broken object property-level authorization / mass assignment |
| Tenant boundary | Daniel gọi `GET /api/v1/manager/cases/case_03` | Deny | Trả case của `org_02` | Missing tenant-scope check |
| Parent-child relationship | Ben gọi path `case_01/evidence/evidence_02` | Deny | Trả evidence của `case_02` | Unvalidated parent-child relationship |
| Indirect access path | Ben export `evidence_03` | Không tạo job | Worker hoàn tất và trả file | Indirect-path authorization bypass |

Các lỗi ở ownership boundary, tenant boundary, parent-child relationship và indirect access path đều là object-level authorization failure nhưng xảy ra tại những boundary khác nhau. Lỗi property-level authorization xảy ra khi actor có quyền trên object nhưng không có quyền trên property được gửi lên.

## Root cause chung

Năm finding đều bắt nguồn từ cùng một lỗi thiết kế:

> Client được phép chọn resource hoặc property, hệ thống truy vấn chúng bằng lớp truy cập dữ liệu dùng chung, nhưng Authorization chỉ được bổ sung bằng một số check rời rạc tại từng route.

Một authorization decision đầy đủ phải dựa trên:

```text
allow = policy(
  subject=current user and current role in the organization,
  action=read | update | approve | export,
  resource=the actual note, case, or evidence,
  context={
    organization,
    ownership,
    assignment,
    parent-child relationships,
    mutable properties,
    current resource state
  }
)
```

Trong mỗi vulnerable path, hệ thống bỏ sót ít nhất một dữ kiện:

| Authorization boundary | Dữ kiện bị thiếu |
|---|---|
| Ownership boundary | Ownership của note |
| Property-level authorization | Quyền trên property và state transition |
| Tenant boundary | Tenant của resource so với organization nơi role của user có hiệu lực |
| Parent-child relationship | Relationship giữa evidence và case |
| Indirect access path | Quyền trên source object tại API, worker và download endpoint |

Codebase sử dụng các method như `get_note(id)`, `get_case(id)`, `get_evidence(id)` hoặc `model.update(request.json)` trước khi thu thập đủ authorization context. Việc enforcement bị phân tán khiến mỗi code path chỉ nhìn thấy một phần của context: một controller chỉ kiểm tra người gọi có phải là member của organization hay không, controller khác chỉ kiểm tra role hoặc assignment, còn worker chỉ kiểm tra job ownership. Mỗi phép kiểm tra riêng lẻ có thể đúng, nhưng không phép kiểm tra nào đủ để bảo vệ toàn bộ operation.

Điều này giải thích vì sao một số control gần đó vẫn hoạt động:

- Analyst bị chặn khỏi Manager route nhưng vẫn đọc được object qua endpoint khác.
- Direct evidence endpoint trả `403` nhưng export lại thành công.
- Ownership được kiểm tra khi update note nhưng protected property vẫn được chấp nhận.

Đổi ID tuần tự thành UUID không sửa được lỗi thiết kế này. UUID chỉ làm identifier khó đoán hơn. Khi ID xuất hiện trong traffic, log, search result, link hoặc export, attacker vẫn có thể sử dụng nó nếu backend tiếp tục thực hiện unscoped lookup.

## Thiết kế Authorization an toàn hơn

Hướng khắc phục là đưa Authorization vào service layer bắt buộc, thay vì để từng controller tự quyết định có kiểm tra hay không.

Route có thể parse input và tạo response, nhưng không được trực tiếp dùng unrestricted repository để lấy protected resource.

### Principal dùng chung cho mọi authorization decision

Trong phần code bên dưới, `Membership` là tên của object lưu ba thông tin: user thuộc organization nào, giữ role gì và trạng thái thành viên hiện tại có còn `active` hay không. Tên class được giữ bằng tiếng Anh vì đây là thuật ngữ trong code; phần giải thích còn lại sẽ dùng cách diễn đạt tự nhiên hơn.

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

Trong mỗi request, principal phải được tạo từ session cùng thông tin hiện tại trong database: user thuộc organization nào, có role gì và trạng thái thành viên đang là `active`, `suspended` hay đã bị thu hồi.

### Khắc phục ownership boundary: kiểm tra ownership khi đọc note

Handler dễ mắc lỗi khi thực hiện global lookup rồi chỉ xác nhận user là member của organization:

```python
note = notes.get_by_id(note_id)
require(current_user.is_member_of(note.organization_id))
return serialize(note)
```

Service sau khi sửa thu thập đủ organization, role và ownership trước khi trả object:

```python
def read_note(principal: Principal, note_id: str) -> dict:
    note = notes.get_by_id(note_id)
    if note is None:
        raise NotFound()

    membership = principal.active_membership(note.organization_id)
    if membership is None:
        raise NotFound()

    if membership.role is Role.ANALYST:
        if note.owner_id != principal.user_id:
            raise NotFound()
    elif membership.role not in {Role.MANAGER, Role.OWNER}:
        raise Forbidden()

    return serialize_note(note)
```

Bản sửa cố ý trả `404 Not Found` khi Analyst không sở hữu note để không làm lộ sự tồn tại của object. Hệ thống có thể chọn `403` thay thế, miễn mọi unauthorized relationship đều bị từ chối nhất quán.

### Khắc phục property-level authorization: allowlist field và tách approve thành action riêng

Handler dễ mắc lỗi khi authorize quyền update note rồi bind toàn bộ JSON body vào model:

```python
note = notes.get_by_id(note_id)
require(can_update_note(principal, note))
note.update(request.json)
notes.save(note)
```

Sau khi sửa, schema dành cho Analyst chỉ chấp nhận các property hợp lệ:

```python
from pydantic import BaseModel, ConfigDict

class AnalystNotePatch(BaseModel):
    model_config = ConfigDict(extra="forbid")

    title: str | None = None
    body: str | None = None
```

Action approve được tách khỏi update note và có policy riêng:

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

Thiết kế này tránh việc một quyền update object quá rộng vô tình trở thành quyền ghi mọi field của database model.

### Khắc phục tenant boundary: đưa tenant scope vào query

Manager endpoint dễ mắc lỗi khi chỉ kiểm tra role rồi thực hiện global lookup:

```python
require(current_membership.role in {Role.MANAGER, Role.OWNER})
case = cases.get_by_id(case_id)
return serialize(case)
```

Sau khi sửa, organization context phải được xác định từ organization mà server đã xác nhận user đang là member hợp lệ.

Endpoint được đổi thành:

```http
GET /api/v1/orgs/{organization_id}/manager/cases/{case_id}
```

Service:

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

`organization_id` trong URL chỉ là selector. Nó không tự tạo quyền. Server vẫn phải xác nhận user thực sự là member đang hoạt động trong organization đó và role hiện tại cho phép thực hiện action.

### Khắc phục parent-child relationship: ràng buộc đầy đủ nested path

Vulnerable pattern:

```python
case = cases.get_by_id(case_id)
require(can_read_case(principal, case))

evidence = evidence_repository.get_by_id(evidence_id)
return serialize(evidence)
```

Query sau khi sửa phải ràng buộc organization, case và evidence:

```sql
SELECT e.*
FROM evidence AS e
JOIN cases AS c ON c.id = e.case_id
WHERE e.id = :evidence_id
  AND c.id = :case_id
  AND c.organization_id = :organization_id;
```

Evidence chỉ được trả về khi đồng thời khớp `evidence_id + case_id + organization_id`, vì vậy client không còn có thể gắn một child không liên quan vào một parent hợp lệ. Sau khi resolve đúng `case` và `evidence` theo toàn bộ relationship trong URL, service vẫn phải gọi `can_read_evidence(principal, case, evidence)` để kiểm tra tenant, role và assignment của subject trước khi trả dữ liệu.

### Khắc phục indirect access path: authorize tại API, worker và download endpoint

Theo policy của Umber Desk 12, worker phải đánh giá lại quyền tại thời điểm thực thi. API trước hết authorize source object rồi mới tạo job:

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
        source_id=evidence.id,
        organization_id=evidence.organization_id,
        export_format=export_format,
    )

    queue.publish(job.id)
    return serialize_export_job(job)
```

Khi thực thi, worker load lại principal và resource state hiện tại:

```python
def run_export_job(job_id: str) -> None:
    job = export_jobs.get_by_id(job_id)
    if job is None:
        return

    principal = principals.load_current(job.actor_user_id)
    evidence = evidence_repository.get_by_id(job.source_id)
    case = cases.get_by_id(evidence.case_id) if evidence else None

    if evidence is None or case is None:
        export_jobs.fail(job.id, reason="source_not_found")
        return

    if not can_read_evidence(principal, case, evidence):
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

Download endpoint cũng phải kiểm tra job ownership và quyền hiện tại trên source object. Biết job ID hoặc từng tạo job không đồng nghĩa user vẫn có quyền tải output.

## Retest

Trước khi retest, fixture được reset về trạng thái ban đầu:

- Tất cả resource trở lại `version: 1`.
- Chưa có export job.
- Chưa có audit event.

Trong bản sửa, `404 Not Found` được dùng khi object hoặc relationship không nằm trong phạm vi mà subject có thể nhìn thấy; `403 Forbidden` được dùng khi action đã được xác định nhưng subject không có quyền thực hiện; `422 Unprocessable Content` được dùng khi request chứa property không được schema của action chấp nhận.

| Retest | Trước khi sửa | Sau khi sửa | Xác minh |
|---|---|---|---|
| Ben đọc `note_02` | `200`, trả note của Alex | `404`, không trả note body | Alex vẫn đọc được `note_02` |
| Ben update `note_01` với `review_status` | `200`, status thành `approved` | `422`, từ chối property | Status vẫn là `draft` |
| Daniel đọc `case_03` | `200`, trả case của `org_02` | `404` | Maya vẫn đọc được `case_03` |
| Ben gọi path `case_01/evidence/evidence_02` | `200`, trả child của `case_02` | `404` | Alex vẫn đọc được evidence qua đúng path |
| Ben export `evidence_03` | `202`, job hoàn tất | `403`, không tạo job | Không có audit event cross-tenant |
| Ben đọc `note_01` | `200` | `200` | Positive control vẫn hoạt động |
| Ben update `note_01.body` | `200` | `200` | Property hợp lệ vẫn được cập nhật |
| Daniel đọc `case_02` trong `org_01` | `200` | `200` | Manager vẫn đọc được case cùng tenant |
| Ben đọc `evidence_01` qua đúng path | `200` | `200` | Relationship hợp lệ vẫn hoạt động |
| Ben export `evidence_01` | `202`, hoàn tất | `202`, hoàn tất | Export hợp lệ không bị phá vỡ |
| Daniel approve `note_01` | `200` | `200` | Manager vẫn thực hiện được state transition |

Bản build sau khi sửa đã chặn toàn bộ request dùng để chứng minh lỗi, đồng thời giữ nguyên các workflow hợp lệ.

## Bài học thực tế

1. Authentication chỉ xác định subject; mỗi action vẫn cần một authorization decision riêng.
2. Luôn đưa tenant scope vào resource query.
3. Kiểm tra quyền trên từng property, không chỉ trên toàn bộ object.
4. Validate mọi relationship xuất hiện trong nested path.
5. Áp dụng cùng một object policy cho direct và indirect access path.
6. Trong mỗi test, chỉ thay đổi một giá trị ảnh hưởng đến Authorization.
7. Dùng tài khoản thứ hai và trạng thái trên server để xác minh tác động.
8. UUID là identifier khó đoán hơn, không phải access control.
9. Retest phải kiểm tra cả request trái phép lẫn positive control hợp lệ.

## Kết luận

Cuộc đánh giá bắt đầu bằng việc xác định product policy và viết authorization matrix. Sau đó, năm boundary được kiểm thử riêng: ownership, protected property, tenant scope, parent-child relationship và direct/indirect access path. Trong mỗi test, chỉ một giá trị ảnh hưởng đến Authorization được thay đổi.

Năm finding khác nhau cùng chỉ về một lỗi thiết kế: hệ thống truy vấn resource quá rộng và bind property tự động, trong khi Authorization được triển khai bằng các check cục bộ thiếu context.

Bản sửa sử dụng một `Principal` chứa organization, role và trạng thái thành viên hiện tại, kết hợp với scoped query, property allowlist, relationship-aware lookup và reauthorization tại worker lẫn download endpoint. Retest cho thấy các unauthorized request đã bị chặn mà workflow hợp lệ vẫn hoạt động.

Đây là tiêu chí quan trọng của một bản sửa Authorization đúng:

> Chặn sai quyền nhưng không phá vỡ đúng quyền.

## Thuật ngữ sử dụng trong bài

- **Baseline**: request hợp lệ dùng làm mốc so sánh.
- **Positive control**: hành vi hợp lệ phải tiếp tục hoạt động sau khi sửa.
- **Mass assignment**: backend tự động bind nhiều field từ request vào model mà không allowlist rõ ràng.
- **State transition**: thay đổi trạng thái nghiệp vụ, ví dụ từ `draft` sang `approved`.
- **Scoped query**: query giới hạn resource theo tenant, owner hoặc relationship cần thiết.
- **Fail closed**: khi không đủ dữ kiện hoặc authorization thất bại, hệ thống từ chối thao tác.
- **`ETag` / `If-Match`**: cơ chế giúp tránh ghi đè khi object đã bị thay đổi bởi request khác.

## Tài liệu tham khảo

- [OWASP API Security Top 10 — API1:2023 Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [OWASP API Security Top 10 — API3:2023 Broken Object Property Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa3-broken-object-property-level-authorization/)
- [OWASP API Security Top 10 — API5:2023 Broken Function Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa5-broken-function-level-authorization/)
- [PortSwigger Web Security Academy — Access control vulnerabilities and privilege escalation](https://portswigger.net/web-security/access-control)
- [PortSwigger Web Security Academy — Insecure direct object references](https://portswigger.net/web-security/access-control/idor)
