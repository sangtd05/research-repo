# Thách thức và Hạn chế: Mongoose với AWS DocumentDB 5.0 (Chi tiết)

Chào anh, dựa trên thông tin chi tiết nhất về khả năng tương thích, em xin lọc ra **3 điểm "tử huyệt"** quan trọng nhất mà anh cần đặc biệt lưu tâm. Những thứ khác có thể né được, nhưng 3 thứ này sẽ ảnh hưởng trực tiếp đến "sống còn" của hệ thống khi chạy thực tế.

> [!IMPORTANT]
>
> ## 1. HIỆU NĂNG "NGẦM": Các toán tử không ăn Index
>
> Đây là điều nguy hiểm nhất vì code **không báo lỗi** nhưng hệ thống sẽ **chậm dần đến mức sập** khi data lớn.
>
> - Các toán tử `$ne`, `$nin`, `$exists`, `$not`, `$distinct` sẽ ép DocDB quét toàn bộ collection (Full Scan).
> - **Hệ quả:** Nếu anh có 1 triệu bản ghi, mỗi câu query tìm `status: { $ne: 'deleted' }` sẽ cực kỳ kinh khủng.
> [!WARNING]
>
> ## 2. KIẾN TRÚC: Thiếu Stage `$facet` trong Aggregation
>
> Nếu App của anh có chức năng: "Hiển thị danh sách kết quả + Tổng số bản ghi (Count)" ở cùng một màn hình (rất phổ biến trong admin dashboard), anh thường dùng `$facet`.
>
> - **Hệ quả:** DocDB không hỗ trợ Stage này. Anh sẽ phải viết 2 câu query riêng biệt hoặc xử lý lồng ghép phức tạp, gây tăng tải cho DB và trễ mạng.
> [!CAUTION]
>
> ## 3. TÍNH NĂNG MỞ RỘNG: Không có Text Search (`$text`)
>
> MongoDB dùng `$text` index để tìm kiếm từ khóa gần đúng rất nhanh. DocumentDB không có.
>
> - **Hệ quả:** Anh không thể làm ô "Tìm kiếm nội dung bài viết" nhanh chóng được. Anh buộc phải dùng Operator `$regex` (rất chậm trên volume lớn) hoặc phải cài thêm AWS OpenSearch (tốn thêm rất nhiều tiền và công sức tích hợp).

---

## Chi tiết danh sách "đen" những thứ không hỗ trợ

## 1. Các tính năng (Features) KHÔNG hỗ trợ

- **Capped Collections**: Không hỗ trợ (thường dùng cho log hệ thống).
- **Text & Vector Indexes**: Không hỗ trợ tìm kiếm toàn văn hoặc tìm kiếm vector.
- **GridFS**: Không hỗ trợ (dùng để lưu file lớn).
- **Time-series data**: Không hỗ trợ các collection chuyên dụng cho dữ liệu chuỗi thời gian.
- **Case-insensitive Indexes**: Không hỗ trợ index không phân biệt hoa thường.

## 2. Các Query Operators KHÔNG hỗ trợ

Mongoose Query của anh sẽ bị lỗi nếu dùng:

- `$expr`, `$jsonSchema`, `$text`, `$where`, `$meta`.
- Các toán tử không gian: `$box`, `$center`, `$centerSphere`, `$polygon`, `$near`.

## 3. Các Aggregation Stages & Operators KHÔNG hỗ trợ

Nếu anh dùng `.aggregate()`, các stage sau sẽ **FAIL**:

- `$facet` (phân trang/thống kê song song).
- `$collStats`, `$bucket`, `$bucketAuto`, `$sortByCount`, `$unionWith`, `$graphLookup`, `$merge`.
- Các toán tử tính toán: `$accumulator`, `$stdDevPop`, `$dateDiff`, `$regexFind`, `$rand`, `$getField`.

## 4. Các phương thức Cursor (Cursor Methods) KHÔNG hỗ trợ

Các hàm xích (`chaining`) sau trong Mongoose sẽ không hoạt động:

- `.collation()`, `.max()`, `.min()`, `.noCursorTimeout()`, `.returnKey()`, `.tailable()`.

## 5. CẢNH BÁO: Operator không tận dụng được Index

Đây là lỗi "ngầm" nguy hiểm nhất. DocumentDB **SẼ KHÔNG** dùng Index (dẫn đến quét toàn bộ bảng - CollScan) nếu query chứa:

- `$ne` (không bằng).
- `$nin` (không nằm trong tập hợp).
- `$nor`, `$not`, `$exists`, `$distinct`.
- `$elemMatch` khi dùng trong các truy vấn lồng nhau (nested queries).

> [!CAUTION]
> Nếu database của anh lớn, việc dùng các toán tử trên sẽ khiến App chạy cực chậm vì DocDB phải duyệt từng bản ghi thay vì dùng Index.

## 6. Các lệnh quản trị (Commands) KHÔNG hỗ trợ

Nếu anh dùng các script migration hoặc tool quản trị gọi ngầm các lệnh này, chúng sẽ báo lỗi:

- `collMod`, `reIndex`, `createView`, `copydb`, `connPoolStats`, `dbHash`.

**Lời khuyên từ em:**
Anh hãy kiểm tra ngay code của mình, đặc biệt là phần **Aggregation** và các query dùng `$ne` hoặc `$exists`. Nếu data lớn, anh cần phải cấu trúc lại cách lưu trữ để tránh dùng các toán tử "kháng Index" này.

Anh có muốn em giúp rà soát một đoạn code cụ thể nào đang dùng các toán tử này không?
