# Runbook — Sealed Secrets với Argo CD

## 1. Mục tiêu

Quản lý Kubernetes Secret bằng GitOps mà không lưu Secret
plaintext trong Git repository.

## 2. Kiến trúc

- GitOps repository: hoang4325/argocd-helm-lab
- Kubernetes: Kind
- GitOps controller: Argo CD
- Secret encryption: Bitnami Sealed Secrets
- Controller namespace: kube-system
- Demo namespace: sealed-secrets-demo

## 3. Luồng triển khai

1. Tạo Kubernetes Secret thử nghiệm ở file tạm.
2. Lấy public certificate từ Sealed Secrets controller.
3. Sử dụng kubeseal với scope strict để mã hóa Secret.
4. Commit SealedSecret đã mã hóa vào Git.
5. Argo CD phát hiện thay đổi và đồng bộ Application.
6. Sealed Secrets controller giải mã SealedSecret.
7. Kubernetes Secret được tạo trong namespace đích.
8. Deployment đọc Secret thông qua biến môi trường.

## 4. Xác minh

Kiểm tra controller:

kubectl get deployment sealed-secrets-controller -n kube-system

Kiểm tra Application:

kubectl get application sealed-secrets-controller -n argocd
kubectl get application sealed-secrets-demo -n argocd

Kiểm tra SealedSecret:

kubectl get sealedsecret demo-app-secret -n sealed-secrets-demo
kubectl describe sealedsecret demo-app-secret -n sealed-secrets-demo

Kiểm tra Kubernetes Secret mà không in giá trị:

kubectl get secret demo-app-secret -n sealed-secrets-demo \
  -o go-template='name={{.metadata.name}} type={{.type}} keys={{range $k, $v := .data}}{{$k}} {{end}}{{"\n"}}'

Kiểm tra Deployment:

kubectl rollout status deployment/demo-secret-consumer \
  -n sealed-secrets-demo

Kiểm tra biến môi trường có giá trị mà không tiết lộ nội dung:

kubectl exec deployment/demo-secret-consumer \
  -n sealed-secrets-demo -- \
  sh -c 'test -n "$DEMO_USERNAME" && test -n "$DEMO_PASSWORD" && echo PASS'

## 5. Xử lý sự cố

### Không lấy được public certificate

- Kiểm tra kubectl config current-context.
- Kiểm tra Kubernetes API Server.
- Xác minh controller Deployment và Service.
- Dùng đúng kubeconfig, tên controller và namespace.

### SealedSecret không được giải mã

- Kiểm tra kubectl describe sealedsecret.
- Kiểm tra Events trong namespace ứng dụng.
- Xem log của sealed-secrets-controller.
- Xác minh tên và namespace có khớp với scope strict.
- Kiểm tra trạng thái khóa giải mã của controller.

### Application không đồng bộ

- Kiểm tra trạng thái Application và revision.
- Kiểm tra điều kiện và thông báo lỗi.
- Xác minh manifest đã được push lên nhánh main.

## 6. Lưu ý bảo mật

- Không commit Kubernetes Secret plaintext.
- Không in password hoặc giá trị Secret ra log.
- Dùng scope strict khi mã hóa Secret cho một namespace cụ thể.
- Bảo vệ và sao lưu khóa riêng của Sealed Secrets theo quy trình
  an toàn; mất khóa có thể khiến không giải mã được ciphertext
  đã lưu trong Git.
- Không dùng dữ liệu demo làm credentials thực tế.

## 7. Kết quả thực hành

- Controller: Running.
- Argo CD Application: Synced và Healthy.
- SealedSecret: Synced=True.
- Kubernetes Secret: có hai khóa DEMO_USERNAME và DEMO_PASSWORD.
- Deployment: rollout thành công.
- Kiểm tra biến môi trường: PASS.

Kết luận: Đã xác minh luồng GitOps từ manifest mã hóa trong Git
đến Kubernetes Secret được workload sử dụng.
