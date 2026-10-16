# Runbook — Helm Application với Argo CD GitOps

## 1. Mục tiêu

Triển khai một Helm chart bằng Argo CD và kiểm chứng cơ chế
GitOps khi thay đổi cấu hình mong muốn trong Git.

## 2. Môi trường

- GitOps repository: https://github.com/hoang4325/argocd-helm-lab
- Git branch: main
- Kubernetes: Kind
- GitOps controller: Argo CD
- Application: helm-demo
- Namespace: helm-demo
- Helm chart: charts/helm-demo

## 3. Cấu trúc chart

- Chart.yaml: metadata và phiên bản chart.
- values.yaml: các giá trị cấu hình.
- templates/deployment.yaml: template Deployment.
- templates/service.yaml: template Service.
- helm-demo-app.yaml: khai báo Argo CD Application.

## 4. Kiểm tra chart tại local

Kiểm tra chất lượng chart:

helm lint ./charts/helm-demo

Render template:

helm template helm-demo ./charts/helm-demo

Kết quả ban đầu:
- Deployment có replicas = 1.
- Service sử dụng ClusterIP.
- Service port = 80.
- Container image = nginx:1.28-alpine.

## 5. Triển khai qua GitOps

Argo CD Application sử dụng:
- Repository: argocd-helm-lab
- Branch: main
- Path: charts/helm-demo
- Helm release name: helm-demo
- Destination namespace: helm-demo

Application được quản lý bởi bookinfo-root.

## 6. Diễn tập thay đổi replica

Thay đổi trong charts/helm-demo/values.yaml:

replicaCount: 1

thành:

replicaCount: 2

Sau đó commit và push thay đổi lên main.

Argo CD phát hiện cấu hình mới, render Helm chart và đồng bộ
Deployment trong Kubernetes.

Không chạy helm upgrade thủ công trong quá trình kiểm chứng.

## 7. Xác minh

Kiểm tra Application:

kubectl get application helm-demo -n argocd -o wide

Kiểm tra Deployment:

kubectl get deployment helm-demo -n helm-demo

Kiểm tra trạng thái rollout:

kubectl rollout status deployment/helm-demo \
  -n helm-demo --timeout=120s

Kiểm tra số replica:

kubectl get deployment helm-demo -n helm-demo \
  -o jsonpath='desired={.spec.replicas} ready={.status.readyReplicas} available={.status.availableReplicas}{"\n"}'

Kiểm tra Pod:

kubectl get pods -n helm-demo -o wide

## 8. Kết quả thực hành

- Helm lint thành công, không có chart error.
- Helm template render Deployment và Service.
- Argo CD Application helm-demo đạt Synced và Healthy.
- Deployment ban đầu có 1 replica sẵn sàng.
- Sau thay đổi values.yaml qua Git, số replica tăng lên 2.
- Người thực hành đã quan sát thấy hai replica trong cluster.

## 9. Bài học rút ra

Helm quản lý template và values dùng để render Kubernetes
manifests. Argo CD theo dõi Git và đồng bộ trạng thái thực tế
theo cấu hình mong muốn.

Thay đổi values.yaml và push lên Git là đủ để kích hoạt quy
trình triển khai khi Application đã được cấu hình auto-sync.

Cần phân biệt:
- helm lint: kiểm tra chart.
- helm template: render template thành manifest.
- Argo CD: theo dõi Git và reconcile trạng thái Kubernetes.
- Kubernetes Deployment: duy trì số replica theo cấu hình.

## 10. Kết luận

Bài lab đã xác minh Helm chart được quản lý bằng Argo CD và
thay đổi replica thông qua GitOps thành công.
