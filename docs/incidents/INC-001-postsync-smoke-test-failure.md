# RCA — INC-001: PostSync Smoke Test thất bại do sai cổng Service

## 1. Thông tin sự cố

- **Mã sự cố:** INC-001
- **Tên sự cố:** PostSync Smoke Test Failure
- **Môi trường:** Dev
- **Hệ thống:** Kubernetes Kind, Argo CD, Kustomize, Book Info
- **Thành phần bị ảnh hưởng:** Kubernetes Job `bookinfo-smoke-test`
- **Mức độ ảnh hưởng:** Thấp — ảnh hưởng đến kết quả xác minh sau triển khai
- **Loại sự cố:** Diễn tập lỗi có chủ đích trong lab
- **Trạng thái:** Đã khôi phục và xác minh thành công

## 2. Tóm tắt sự cố

Trong quá trình thực hành Resource Hooks của Argo CD, một lỗi được cố tình đưa vào cấu hình Kustomize của môi trường Dev. Cấu hình mới thay đổi lệnh kiểm tra sau triển khai, khiến Kubernetes Job `bookinfo-smoke-test` gửi HTTP request đến cổng `9081` thay vì cổng dịch vụ `9080`.

Do Service không phục vụ tại cổng được chỉ định trong bài kiểm tra, lệnh `curl` bị timeout. Kubernetes Job chuyển sang trạng thái `Failed` và sync operation của Argo CD cũng thất bại tại bước PostSync Hook.

Sau khi thu thập log và xác định nguyên nhân, cấu hình lỗi được loại bỏ qua Git. Argo CD đồng bộ lại cấu hình đã khôi phục và chạy PostSync Hook thành công.

## 3. Ảnh hưởng

Ảnh hưởng được xác nhận trong quá trình diễn tập gồm:

- Kubernetes Job `bookinfo-smoke-test` tại namespace `bookinfo-dev` thất bại.
- Sync operation của Application `bookinfo-dev` chuyển sang `Failed`.
- Kết quả kiểm thử sau triển khai không đạt, khiến quá trình xác nhận phát hành bị đánh dấu thất bại.

Tại thời điểm điều tra, Argo CD có thể hiển thị `Synced` và `Healthy` đồng thời với `phase=Failed`. Điều này xảy ra vì trạng thái đồng bộ và trạng thái operation biểu thị hai khía cạnh khác nhau.

Không có bằng chứng trong log thu thập được cho thấy các Deployment của Book Info bị dừng do lỗi này. Sự cố tập trung vào bước kiểm thử sau triển khai, không phải lỗi khởi động ứng dụng.

Môi trường Staging không được cố tình thay đổi trong bài diễn tập.

## 4. Phát hiện và bằng chứng

Sự cố được phát hiện bằng cách kiểm tra Job, log của container và trạng thái Application trong Argo CD.

### 4.1. Trạng thái Kubernetes Job

Job `bookinfo-smoke-test` chuyển sang `Failed`, với `0/1` completion.

### 4.2. Log của Job

Thông báo lỗi ghi nhận được:

```text
INTENTIONAL FAILURE INJECTION: wrong service port
curl: (28) Connection timed out after 5002 milliseconds
```

Log xác nhận lệnh kiểm thử đã thực thi cấu hình tạo lỗi, đồng thời không thể hoàn thành HTTP request trong thời gian chờ quy định.

### 4.3. Trạng thái Argo CD

Application `bookinfo-dev` hiển thị:

```text
sync=Synced
health=Healthy
phase=Failed
message=one or more synchronization tasks completed unsuccessfully
```

Kết quả này cho thấy trạng thái tài nguyên mong muốn vẫn có thể đồng bộ với Git, nhưng operation thất bại do tác vụ PostSync Hook không hoàn thành thành công.

## 5. Phân tích nguyên nhân gốc

### 5.1. Nguyên nhân trực tiếp

File `apps/bookinfo/overlays/dev/smoke-test-failure-patch.yaml` đã ghi đè lệnh của Job `bookinfo-smoke-test` để gọi endpoint:

```text
http://products:9081/productpage
```

Trong khi đó, Service Book Info đang cung cấp cổng `9080`.

Việc sử dụng sai cổng khiến lệnh `curl` không kết nối được đến endpoint mục tiêu và kết thúc với lỗi timeout.

### 5.2. Nguyên nhân ở cấp quy trình

Bài diễn tập cố tình thay đổi cấu hình kiểm thử trong Git nhằm xác minh khả năng phát hiện lỗi sau triển khai.

PostSync Hook chỉ được thực thi khi có một sync operation phù hợp. Việc làm mới trạng thái bằng annotation `argocd.argoproj.io/refresh=hard` không tự động tạo một sync operation mới.

Vì thế, sau khi sửa cấu hình, cần bảo đảm Argo CD thực hiện một sync mới để kiểm tra lại Hook. Trong lab này, một thay đổi cấu hình được đưa vào Git để kích hoạt quá trình tự đồng bộ.

## 6. Quá trình khắc phục

Quá trình khắc phục được thực hiện qua Git theo mô hình GitOps.

**Bước 1 — Khôi phục cấu hình Dev**

Xóa patch tạo lỗi khỏi overlay Dev, khôi phục file `kustomization.yaml` và loại bỏ lệnh gọi tới cổng `9081`.

**Bước 2 — Kiểm tra manifest**

Sử dụng `kubectl kustomize` để render manifest sau khi sửa. Kết quả xác nhận cấu hình không còn chứa `INTENTIONAL FAILURE INJECTION` hoặc `products:9081`, đồng thời lệnh kiểm thử trở về `products:9080`.

**Bước 3 — Commit và push**

Bản sửa được đưa lên GitHub trong commit:

```text
e58e33d fix(gitops): restore dev hook and trigger auto-sync
```

**Bước 4 — Đồng bộ lại bằng Argo CD**

Argo CD nhận revision mới và chạy lại PostSync Hook trên môi trường Dev.

## 7. Xác minh sau khắc phục

Các kết quả thu thập được xác nhận sự cố đã được khôi phục.

### 7.1. Cấu hình đã sửa

Manifest Kustomize sau khi render có:

```text
http://products:9080/productpage
```

Không còn lệnh failure injection hoặc lời gọi tới cổng `9081`.

### 7.2. Kubernetes Job

Job chuyển sang trạng thái:

```text
STATUS: Complete
COMPLETIONS: 1/1
```

Log của Job:

```text
Starting Book Info smoke test
PASS: Book Info returned HTTP success
```

### 7.3. Trạng thái Argo CD

Application `bookinfo-dev` đạt trạng thái:

```text
sync=Synced
health=Healthy
phase=Succeeded
```

Revision được xác nhận là:

```text
e58e33d516dc9e8c645d9e3d5818f0a49eef6d43
```

Những kết quả này xác nhận cấu hình đã khôi phục, PostSync Hook chạy thành công và sync operation mới hoàn tất.

## 8. Biện pháp phòng ngừa

### 8.1. Kiểm tra cấu hình trước khi push

Trước khi commit thay đổi trong Kustomize, thực hiện:

```bash
kubectl kustomize apps/bookinfo/overlays/dev \
  > /tmp/bookinfo-dev-rendered.yaml
```

Kiểm tra endpoint smoke test trong manifest đã render để phát hiện sai cổng trước khi triển khai.

### 8.2. Theo dõi đồng thời Sync, Health và Operation

Không chỉ dựa vào trạng thái `Synced` hoặc `Healthy`. Khi điều tra một lần triển khai thất bại, cần kiểm tra thêm:

- `status.operationState.phase`
- `status.operationState.message`
- Trạng thái Kubernetes Job
- Log của container trong Job

### 8.3. Xử lý lỗi thông qua Git

Thực hiện khắc phục bằng cách sửa manifest, commit và push lên Git. Không chỉnh sửa trực tiếp Deployment để che giấu sai lệch cấu hình trong GitOps.

### 8.4. Xác minh phục hồi sau mỗi lần sửa

Sau khi triển khai lại, xác nhận Job hoàn tất, log chứa thông báo PASS và operation mới nhất của Argo CD thành công.

## 9. Bài học rút ra

Sự cố diễn tập cho thấy một ứng dụng đang chạy bình thường không đồng nghĩa với quy trình phát hành đã thành công.

PostSync Hook cung cấp một bước kiểm tra sau triển khai. Nếu bước này thất bại, Argo CD có thể đánh dấu sync operation thất bại ngay cả khi các Deployment vẫn hoạt động.

Ngoài ra, `refresh=hard` chỉ làm mới trạng thái nguồn và phép so sánh; nó không tự chạy lại một PostSync Hook. Muốn Hook được thực thi, cần có một sync operation mới.

Qua bài thực hành này, quy trình GitOps được kiểm chứng từ thay đổi cấu hình trong Git, phát hiện lỗi, thu thập bằng chứng, sửa cấu hình và xác minh phục hồi.

## 10. Kết luận

INC-001 được đóng sau khi cấu hình smoke test trong môi trường Dev được khôi phục về đúng cổng Service, Job hoàn tất thành công và Application trở lại `Synced`, `Healthy`, với operation mới nhất ở trạng thái `Succeeded`.

Bài diễn tập hoàn thành mục tiêu kiểm chứng cơ chế PostSync Hook của Argo CD và cung cấp một tình huống thực hành có bằng chứng để sử dụng trong tài liệu Incident, RCA và portfolio DevOps/SRE.