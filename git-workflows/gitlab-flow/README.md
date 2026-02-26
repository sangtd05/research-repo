# GitLab Flow

## 1. Tổng quan
GitLab Flow ra đời với định vị "Người lấp khoảng trống". Nó giải quyết sự phức tạp của Gitflow trong khi vá lại rào cản từ sự quá đơn giản của GitHub Flow. Tư duy của GitLab Flow bao gồm **nhánh (branch) gắn chặt với môi trường triển khai (environment)** hoặc hướng theo quy trình **phát hành (release)** bài bản. 
Mô hình này làm rõ câu hỏi *"Code nào đang chạy ở server nào?"*.

## 2. Các biến thể phổ biến

### Biến thể 1: Hướng Môi Trường (Environment-Driven Flow)
Áp dụng cho SaaS hoặc các ứng dụng Web/Microservices cần nhiều tầng quản lý chất lượng (QA) trước khi nhả thẳng ra ngoài tự nhiên:
*   **`main`**: Giống GitHub Flow. Developer merge code từ tính năng (`feature/*`) vào nhánh này thông qua Merge Request. Code hợp lệ với `main` cũng tự động Deploy xuống server **Staging / Test** để đội ngũ QA kiểm thử sơ bộ.
*   **`pre-production`**: Sau khi code ở `main` được kiểm tra kỹ càng qua đội BA/QA, nó dồn vào nhánh chuẩn bị phát hành. Ngay khi merge vào `pre-production`, CI/CD đưa app lên server **UAT (User Acceptance Test)** để khách hàng đánh giá hoặc đánh giá hiệu năng (Performance Test).
*   **`production`**: Điểm dừng cuối cùng. Code trên `pre-production` được merge thẳng vào `production`. Một đoạn kịch bản tự động sẽ đẩy app lên server thật, phục vụ người dùng thật.

### Biến thể 2: Hướng Phát Hành (Release-Driven Flow)
Áp dụng cho App iOS / Android, API Public Versioned hoặc Game/Desktop Software:
* Bắt đầu làm tính năng trên một nhánh, cuối cùng merge hết vào nhánh gốc **`main`**.
* Nếu muốn chốt hạ Phiên bản "1.0", quản trị viên tạo hẳn 1 nhánh mang định danh **`1-0-stable`** được tách từ `main`. Khách hàng cứ việc tải bản này.
* Team vẫn tiếp tục cày cuốc trên `main` cho tính năng mới. 2 tháng sau chốt ra bản "2.0", team tạo nhánh **`2-0-stable`**.
* Sửa lỗi (Fix bug): Dev không bao giờ sửa trực tiếp trên nhánh `stable`. Developer FIX vào nhánh `main` trước. Nếu lỗi đó là quá khẩn cấp với các người dùng V1.0, Dev dùng lệnh **`git cherry-pick`** nhúp một commit fix duy nhất đó ném ngược vào nhánh `1-0-stable` để release bản Patch (Bản vá 1.0.1). 

## 3. Sơ đồ hoạt động bằng Mermaid (Mẫu Môi Trường)

```mermaid
gitGraph
  commit id: "Initial code"
  commit id: "Base system ready"
  
  %% Team makes features
  branch feature/dashboard
  checkout feature/dashboard
  commit id: "Build API for dashboard"
  commit id: "Build UI for dashboard"
  checkout main
  merge feature/dashboard id: "Merge request #1: Dashboard"
  
  %% Auto Deploy to Staging...
  
  %% Move code to Pre-production Environment for client check
  branch pre-production
  checkout pre-production
  merge main tag: "ready-for-UAT"
  
  %% Client discovers a bug, DEV must fix in MAIN first
  checkout main
  branch bugfix/dashboard-crash
  checkout bugfix/dashboard-crash
  commit id: "Fix crash missing data"
  checkout main
  merge bugfix/dashboard-crash id: "Fix MR #2"
  
  %% Promo to pre-prod again
  checkout pre-production
  merge main tag: "retry-UAT"
  
  %% Client happy! Promo to Prod!
  branch production
  checkout production
  merge pre-production tag: "Release v3.4.1"
```

## 4. Đặc thù và Cách xử lý lệnh Git (Cho Release Driven)
**1. Phát hành Version mới (Ví dụ Version 2.4.0)**
```bash
# Ở nhánh origin main (vẫn cập nhật tính năng mới liên tục)
git checkout main

# Tách nhánh dành riêng cho Version 2.4 (Để đó, ko merge lại)
git checkout -b 2-4-stable

# Có thể cập nhật nhẹ metadata
git commit -am "Update release note for v2.4.0"

# Đánh mã Tag chính thức và push
git tag -a v2.4.0 -m "Release 2.4"
git push origin 2-4-stable --tags
```

**2. Vá nhanh 1 lỗi nguy cấp cho bản `2-4-stable`**
```bash
# Developer bắt buộc cập nhật code mới, Sửa lỗi tạo ở MÁY CHỦ (Main)
git checkout main
# Cập nhật thay đổi, ví dụ thay "Auth token expried" -> 012a9f (ID COMMIT)
git commit -am "Fix authentication issue for all"

# Dịch chuyển về nhánh version cũ đang rắc rối
git checkout 2-4-stable

# Cherry-pick (nhặt lấy mỗi 1 cục gạch/commit này) áp vào version cũ
git cherry-pick 012a9f

# Lúc này ta ra 1 bản Patch v2.4.1 xịn xò
git tag -a v2.4.1 -m "Hotfix for auth Issue"
git push origin 2-4-stable --tags
```

## 5. Ưu điểm và Nhược điểm

**Ưu điểm:**
- Cấu trúc "môi trường" trực quan 1-1 cực khớp rọi với những hạ tầng Cloud thực tế, giúp không bao giờ bị loạn "Code này đang test hay code thật".
- Rất hoàn hảo cho GitLab CI, hỗ trợ tạo đường dẫn, phân quyền cực chi tiết (Leader hold Production, QA hold Pre-prod, Dev xài Main).
- Càn quét được cả Web App deploy liên tục lẫn Boxed Software với hai biến thể ở trên (linh hoạt hơn GitFlow 10 lần).

**Nhược điểm:**
- Phương pháp `cherry-pick` (Upstream First) bắt lỗi phải fix từ ngọn rồi lùi lại cành dưới đi ngược cấu trúc suy nghĩ bản địa, thường khiến Developer gây ra **xung đột Conflict cực kì mệt mỏi** khi file của `main` và nhánh stable cũ không giống nhau.
- Overhead (Lực cản thao tác) vì nhiều cấp độ. Code từ lúc mới hoàn thành đến tay người dùng phải trải qua quá nhiều bước Click "Merge Request", mất nhiều thời gian lách vòng.

## 6. Khi nào nên áp dụng?
- Tổ chức Doanh Nghiệp (Enterprise) phát triển sản phẩm lớn, vừa yêu cầu tốc độ vừa bảo mật kỹ lượng và quy trình kiểm duyệt phê duyệt thủ công.
- Có chia cấu trúc server rõ rệt Staging - UAT/Test - Production.
- Tận dụng sức mạnh kết hợp tuyệt đối với mảng CI/CD và Issue Tracking sẵn có do GitLab hỗ trợ.
