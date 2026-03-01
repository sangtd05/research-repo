# Mongoose Migration: Kết nối AWS DocumentDB 5.0 (SSL/TLS)

Chào anh, việc chuyển từ Docker sang AWS DocumentDB 5.0 yêu cầu anh phải thay đổi tư duy từ "tự quản lý" sang "dịch vụ được quản lý". Dưới đây là các điểm mấu chốt:

## 1. Cấu hình Kết nối và SSL (Bắt buộc)

DocumentDB yêu cầu TLS/SSL cực kỳ nghiêm ngặt. Anh không thể dùng chuỗi kết nối đơn giản như khi chạy Docker.

### Bước 1: Tải file chứng chỉ

Anh cần tải file `global-bundle.pem` từ AWS.

```bash
wget https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
```

### Bước 2: Cấu hình trong Mongoose

```javascript
const mongoose = require('mongoose');
const fs = require('fs');

const ca = [fs.readFileSync(__dirname + '/global-bundle.pem')];

mongoose.connect('mongodb://<user>:<password>@<cluster-endpoint>:27017/dbname?tls=true&replicaSet=rs0&readPreference=secondaryPreferred&retryWrites=false', {
  sslValidate: true,
  sslCA: ca,
  useNewUrlParser: true,
  useUnifiedTopology: true
})
.then(() => console.log('Kết nối DocDB 5.0 thành công!'))
.catch(err => console.error('Lỗi kết nối:', err));
```

## 2. Lưu ý về `retryWrites=false`

DocumentDB không hỗ trợ cơ chế *retryable writes* của MongoDB gốc.
> [!IMPORTANT]
> Anh **BẮT BUỘC** phải thêm tham số `retryWrites=false` vào chuỗi kết nối. Nếu không, các hành động `save()` hoặc `update()` của Mongoose sẽ bị crash.

## 3. Tối ưu hóa Indexing

Mongoose mặc định sẽ tự động tạo Index (`autoIndex: true`).

- **Vấn đề:** Trong DocumentDB, việc tạo Index có thể chậm và gây ảnh hưởng hiệu năng khi khởi động App.
- **Giải pháp:** Nên tắt `autoIndex: false` trong môi trường Production và thực hiện tạo Index thủ công qua AWS Console hoặc Script riêng.

## 4. Transactions (Giao dịch)

DocDB 5.0 đã hỗ trợ Transaction. Tuy nhiên:

- Nó chỉ hoạt động nếu Cluster của anh có ít nhất 2 Node (1 Primary, 1 Replica).
- Nếu anh chạy Single Node để tiết kiệm, các lệnh Transaction của Mongoose sẽ báo lỗi.

## 5. Thách thức về tính tương thích

DocumentDB 5.0 không phải là MongoDB 100%. Có rất nhiều Operator và Aggregation stage (như `$facet`, `$expr`) không được hỗ trợ.

> [!WARNING]
> Anh hãy đọc kỹ file [Thách thức và Hạn chế](file:///d:/devops/research-repo/mongoose/challenges-and-limitations.md) để tránh việc App bị crash khi gọi các hàm nâng cao.

## 6. Hạ tầng mạng (VPC)

Đây là điểm "tử huyệt": DocDB nằm trong mạng riêng (VPC).

- Server App Node.js của anh phải nằm cùng VPC hoặc thông qua Peering/VPN.
- Nếu không, Mongoose sẽ báo lỗi **Timeout** dù code của anh hoàn toàn đúng.
