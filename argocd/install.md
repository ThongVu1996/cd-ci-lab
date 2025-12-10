# **Cài đặt Argo CD trên Kubernetes (K8S)**

## **1. Giới thiệu về Argo CD**

- Argo CD là một công cụ **Continuous Delivery (CD)** dành cho Kubernetes, được xây dựng theo triết lý **GitOps**. Thay vì chạy lệnh kubectl apply thủ công, Argo CD sẽ tự động đồng bộ (sync) trạng thái thực tế của cluster với trạng thái mong muốn được khai báo trong Git repository.
- Argo CD chỉ làm nhiệm vụ CD (không build image, không chạy test).
- CI (build image, chạy test, push image…) bạn thường dùng Jenkins, GitLab CI, GitHub Actions… để làm.
- Argo CD sẽ pull manifest từ Git và tự động deploy/update ứng dụng lên cluster K8S.

## **2. Yêu cầu chuẩn bị (Prerequisites)**

- Một cụm Kubernetes cluster đã hoạt động (tối thiểu 1 master, 2 worker).
- Đã cài kubectl và kết nối được với cluster (file ~/.kube/config hoạt động bình thường).
  <image src="./1.png">
- Đã cấu hình CoreDNS và kiểm tra lại bằng lệnh:
  ```
  kubectl get pods \-n kube-system
  ```
  <image src="./2.png">
- Nên có sẵn Ingress Nginx nếu bạn muốn truy cập Argo CD qua domain (có thể xem lại bài hướng dẫn Ingress của bạn).
  
## **3. Các bước cài đặt chi tiết**

- **Bước 1**:
  -  Tại máy k8s-master-1 (máy là master trong cụm k8s) bạn tiến hành chạy lệnh như bên dưới.  
      ```
         kubectl create namespace argocd
         kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
      ```

- **Bước 2**: 
  - Kiểm tra các thành phần của argocd cài đặt trên máy đã được hay chưa tại máy k8s-master-1: 
      ```
         kubectl get all -n argocd
      ```
      <image src="./3.png">

 **Bước 3**: Cấu hình Ingress cho ArgoCD
 - Bạn tiến hành tạo nội dung yaml triển khai file argocd-ingress.yaml trong thư mục home của máy chủ k8s-master-1 với nội dung:
   ```
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      metadata:
      name: argocd-ingress
      namespace: argocd #
      annotations:
         # Tells Ingress Controller the backend protocol is HTTP
         nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
         # ENABLE gRPC: Required for Argo CD CLI and some UI features
         nginx.ingress.kubernetes.io/grpc-backend: "true"
      spec:
      ingressClassName: nginx # Based on your K8s cluster configuration
      rules:
      - host: "argocd.tonytechlab.com"
         http:
            paths:
            - path: /
            pathType: Prefix
            backend:
               service:
                  name: argocd-server # Service name from Rancher
                  port:
                  number: 80 # HTTP port of the service
   ```

- Do ta đang chạy ssl bằng self-cert lên ta sẽ cấu hình đẻ cho phép inscure chạy bằng lệnh
  ```
    kubectl patch configmap argocd-cmd-params-cm -n argocd -p '{"data":{"server.insecure":"true"}}'
  ```

- Khởi động lại (Restart) Argo CD Server
  ```
     kubectl rollout restart deployment argocd-server -n argocd
  ```

 **Bước 4**: Cấu hình dns, npm
 - Cấu hình dns như hình
   <image src ="./4.png">
 - Cấu hình npm như hình
   <image src ="./5.png">
 - Truy cập vào argocd bằng link [argocd](https://argocd.thongdev.site)
   <image src ="./6.png">