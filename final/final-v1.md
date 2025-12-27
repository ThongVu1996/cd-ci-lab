# <center>BÀI LAB CUỐI KHÓA </center>

---

## Table of Content
- [BÀI LAB CUỐI KHÓA ](#bài-lab-cuối-khóa-)
  - [Table of Content](#table-of-content)
  - [**1. TỔNG QUAN DỰ ÁN (PROJECT OVERVIEW)**](#1-tổng-quan-dự-án-project-overview)
    - [**Stack Công nghệ**](#stack-công-nghệ)
  - [**2. KIẾN TRÚC HỆ THỐNG VÀ LUỒNG DỮ LIỆU (ARCHITECTURE \& DATA FLOW)**](#2-kiến-trúc-hệ-thống-và-luồng-dữ-liệu-architecture--data-flow)
    - [**2.1. Luồng truy cập của người dùng (User Request Flow)**](#21-luồng-truy-cập-của-người-dùng-user-request-flow)
  - [**3. YÊU CẦU HẠ TẦNG KỸ THUẬT (INFRASTRUCTURE REQUIREMENTS)**](#3-yêu-cầu-hạ-tầng-kỹ-thuật-infrastructure-requirements)
    - [**3.1. Primary Site - Cloud (AWS)**](#31-primary-site---cloud-aws)
    - [**3.2. Disaster Recovery (DR) Site - On-Premise**](#32-disaster-recovery-dr-site---on-premise)
  - [**4. QUY TRÌNH CI/CD \& GITOPS (CI/CD PIPELINE)**](#4-quy-trình-cicd--gitops-cicd-pipeline)
    - [**4.1. Sơ đồ quy trình (Pipeline Diagram)**](#41-sơ-đồ-quy-trình-pipeline-diagram)
    - [**4.2. Chiến lược triển khai (Deployment Strategy)**](#42-chiến-lược-triển-khai-deployment-strategy)
    - [**4.3. Chi tiết các bước trong Pipeline**](#43-chi-tiết-các-bước-trong-pipeline)
  - [**5. KỊCH BẢN ỨNG PHÓ SỰ CỐ (DISASTER RECOVERY PLAN)**](#5-kịch-bản-ứng-phó-sự-cố-disaster-recovery-plan)
    - [**5.1. Điều kiện kích hoạt (Trigger Condition)**](#51-điều-kiện-kích-hoạt-trigger-condition)
    - [**5.2. Quy trình Failover (Chuyển đổi dự phòng)**](#52-quy-trình-failover-chuyển-đổi-dự-phòng)
  - [**6. THÔNG TIN MÃ NGUỒN \& TÀI NGUYÊN (RESOURCES)**](#6-thông-tin-mã-nguồn--tài-nguyên-resources)
  - [Triển khai ở Local](#triển-khai-ở-local)
    - [Cài đặt Cloudflare Agent](#cài-đặt-cloudflare-agent)
    - [Cài đặt Nginx Ingress](#cài-đặt-nginx-ingress)
    - [Thiết lập biến môi trường](#thiết-lập-biến-môi-trường)
    - [Thiết lập jenkins](#thiết-lập-jenkins)
    - [Kiểm tra Harbor](#kiểm-tra-harbor)
    - [Kiểm tra ECR](#kiểm-tra-ecr)
    - [Kiểm tra minifest github](#kiểm-tra-minifest-github)
    - [Tiến hành sử dụng argoCD để deploy ứng dụng lên cụm K8s](#tiến-hành-sử-dụng-argocd-để-deploy-ứng-dụng-lên-cụm-k8s)
  - [Triển khai ở Cloud](#triển-khai-ở-cloud)
  - [Kịch bản DR](#kịch-bản-dr)
    - [Phân tích](#phân-tích)
    - [Thực hành](#thực-hành)
  - [Phân tích chuyên sâu:](#phân-tích-chuyên-sâu)

---

## **1\. TỔNG QUAN DỰ ÁN (PROJECT OVERVIEW)**

Tài liệu này mô tả kiến trúc kỹ thuật, hạ tầng và quy trình triển khai tự động (CI/CD) cho ứng dụng Web (ReactJS \+ Laravel). Hệ thống được thiết kế theo mô hình **Hybrid Cloud**, đảm bảo tính sẵn sàng cao (High Availability) với cơ chế dự phòng thảm họa (Disaster Recovery \- DR) chuyển đổi linh hoạt giữa AWS Cloud và On-Premise server.

### **Stack Công nghệ**

* **Frontend:** ReactJS  
* **Backend:** Laravel  
* **Database:** MySQL (AWS RDS cho Cloud, MySQL Container cho Local)  
* **Orchestration:** Kubernetes (EKS & Local K8s)  
* **CI/CD & GitOps:** Jenkins, GitLab, ArgoCD, Harbor

## **2\. KIẾN TRÚC HỆ THỐNG VÀ LUỒNG DỮ LIỆU (ARCHITECTURE & DATA FLOW)**

### **2.1. Luồng truy cập của người dùng (User Request Flow)**

Hệ thống sử dụng CloudFlare làm điểm nhập (Entry point) để điều phối lưu lượng truy cập giữa Primary Site (Cloud) và DR Site (On-Premise).

**Sơ đồ luồng dữ liệu:**

```
flowchart LR
    %% Style Definitions
    classDef aws fill:#fff0e6,stroke:#f66,stroke-width:1px,stroke-dasharray: 5 5;
    classDef local fill:#e6f3ff,stroke:#33f,stroke-width:1px,stroke-dasharray: 5 5;
    classDef proxy fill:#fff5cc,stroke:#d4a017,stroke-width:2px,rx:5,ry:5;
    classDef user fill:#2d3748,stroke:#1a202c,stroke-width:2px,color:white,rx:10,ry:10;

    User(User):::user -->|Truy cập Domain| CF{CloudFlare}:::proxy
    
    %% AWS Branch (Primary)
    CF == Primary Route ==> AWS_ALB[AWS ALB]
    
    subgraph AWS_Cloud [☁️ Primary Site - AWS Cloud]
        direction LR
        AWS_ALB:::aws --> AWS_Ingress[Ingress Controller]:::aws
        AWS_Ingress --> AWS_Svc[K8s Service]:::aws
        AWS_Svc --> AWS_Pod[App Pods]:::aws
    end
    
    %% Local Branch (Failover)
    CF -. Failover / DR Mode .-> Tunnel[CloudFlare Tunnel]
    
    subgraph On_Premise [🏠 DR Site - On-Premise]
        direction LR
        Tunnel:::local --> CF_Agent[CloudFlare Agent]:::local
        CF_Agent --> Local_Ingress[Ingress Nginx]:::local
        Local_Ingress --> Local_Svc[K8s Service]:::local
        Local_Svc --> Local_Pod[App Pods]:::local
    end

    %% Link Styles for emphasis
    linkStyle 1 stroke:#48bb78,stroke-width:2px,color:#2f855a
    linkStyle 5 stroke:#e53e3e,stroke-width:2px,stroke-dasharray: 5 5,color:#c53030
```


**Quy trình xử lý chi tiết:**

1. **User Request:** Người dùng truy cập website thông qua tên miền.  
2. **DNS & Routing:** CloudFlare tiếp nhận request.  
   * *Trạng thái bình thường:* Route về AWS.  
   * *Trạng thái DR:* Route về Local thông qua CloudFlare Tunnel/Agent.  
3. **Ingress Layer:**  
   * **AWS:** Request đi qua AWS ALB (Application Load Balancer).  
   * **On-Premise:** Request đi qua CloudFlare Agent tại cụm K8s \-\> Ingress Nginx.  
4. **Service Layer:** Ingress điều hướng đến các Kubernetes Service tương ứng (Frontend/Backend).  
5. **Pod Execution:** Request được xử lý tại các Pod ứng dụng.

**Tham khảo:** Chi tiết về Ingress và Service trong Kubernetes [tại đây](https://www.google.com/search?q=./basic.md%23k8s).

## **3\. YÊU CẦU HẠ TẦNG KỸ THUẬT (INFRASTRUCTURE REQUIREMENTS)**

Hệ thống được chia thành hai môi trường vật lý riêng biệt để đảm bảo tính dự phòng.

### **3.1. Primary Site \- Cloud (AWS)**

Đây là môi trường Production chính phục vụ người dùng cuối.

* **Computing:** Cụm Amazon EKS (Elastic Kubernetes Service). ([Hướng dẫn cài đặt](https://github.com/ThongVu1996/cd-ci-lab/blob/master/aws/install.md))  
* **Container Registry:** Amazon ECR (Elastic Container Registry). ([Tài liệu tham khảo](https://www.google.com/search?q))  
* **Source Control (Mirror):** GitHub Repo. ([Tài liệu tham khảo](https://github.com/ThongVu1996/lab-final))  
* **Database:** Amazon RDS (MySQL).

### **3.2. Disaster Recovery (DR) Site \- On-Premise**

Môi trường dự phòng và cũng là nơi đặt hệ thống CI/CD trung tâm.

* **Orchestration:** Kubernetes Local Cluster. ([Hướng dẫn cài đặt](https://www.google.com/search?q))  
* **CI/CD Tooling:**  
  * Jenkins (Automation Server).  
  * GitLab (Source Code Management \- Private). ([Hướng dẫn cài đặt](https://www.google.com/search?q))  
* **Container Registry (Private):** Harbor.  
  * Cài đặt trên chip Intel: [Xem hướng dẫn](https://tonynguyen.top/harbor-registry-phan-1-cai-dat-harbor-registry-tren-ubuntu/)  
  * Cài đặt trên chip ARM: [Xem hướng dẫn](https://github.com/ThongVu1996/cd-ci-lab/blob/master/argocd/install-harbor.md)  
* **Network:** Tài khoản CloudFlare và Tên miền (Domain) đã cấu hình CloudFlare Tunnel.

## **4\. QUY TRÌNH CI/CD & GITOPS (CI/CD PIPELINE)**

Chúng ta tuân thủ nguyên tắc **GitOps**: Git là "nguồn chân lý duy nhất" (Single Source of Truth) cho trạng thái của hệ thống.

### **4.1. Sơ đồ quy trình (Pipeline Diagram)**

%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '\#5e97f6', 'edgeLabelBackground':'\#ffffff', 'fontFamily': 'arial', 'fontSize': '13px'}}}%%  
flowchart LR  
    %% Style Definitions  
    classDef person fill:\#2d3748,stroke:\#1a202c,stroke-width:2px,color:white,rx:10,ry:10;  
    classDef system fill:\#edf2f7,stroke:\#a0aec0,stroke-width:1px,color:\#2d3748,rx:5,ry:5;  
    classDef storage fill:\#ebf8ff,stroke:\#4299e1,stroke-width:2px,color:\#2b6cb0,shape:cylinder;  
    classDef k8s fill:\#3182ce,stroke:\#2c5282,stroke-width:2px,color:white,shape:hexagon;  
    classDef jenkins fill:\#fff5f5,stroke:\#fc8181,stroke-width:2px,color:\#c53030,rx:5,ry:5;

    %% Nodes  
    Dev(👨‍💻 Developer):::person  
    CTO(🤵 CTO/Manager):::person

    subgraph OnPrem \[🏠 On-Premise Infrastructure\]  
        GL\[GitLab Local\]:::system  
          
        subgraph Jenkins\_Server \[Jenkins Pipeline\]  
            direction TB  
            JenBuild\[Stage 1: Build & Local\]:::jenkins  
            JenDeploy\[Stage 2: Cloud Deploy\]:::jenkins  
        end  
          
        Har\[(Harbor Registry)\]:::storage  
        ArgoLoc\[ArgoCD Local\]:::system  
        K8sLoc{{K8s Local}}:::k8s  
    end

    subgraph Cloud \[☁️ AWS Cloud Infrastructure\]  
        GitOps\[GitOps Repo\]:::system  
        ECR\[(AWS ECR)\]:::storage  
        ArgoCloud\[ArgoCD Cloud\]:::system  
        EKS{{AWS EKS}}:::k8s  
    end

    %% Connections \- Linear Flow  
    Dev \==\>|1. Push Code| GL  
    GL \==\>|2. Webhook| JenBuild  
      
    %% Local Path  
    JenBuild \--\>|3. Build & Push| Har  
    Har \--\>|4. Pull| ArgoLoc  
    ArgoLoc \--\>|5. Auto Deploy| K8sLoc

    %% Approval Bridge \- KEY CHANGE HERE  
    JenBuild \-.-\>|6. Request Approval| CTO  
    CTO \-.-\>|7. Approve| JenDeploy

    %% Cloud Path (Only starts from JenDeploy)  
    JenDeploy \--\>|8. Push Image| ECR  
    JenDeploy \--\>|9. Update Manifest| GitOps  
      
    GitOps \--\>|10. Sync| ArgoCloud  
    ArgoCloud \--\>|11. Rolling Update| EKS  
      
    %% Link Styles  
    linkStyle 5,6 stroke:\#ed8936,stroke-width:2px,stroke-dasharray: 5 5;

### **4.2. Chiến lược triển khai (Deployment Strategy)**

* **Mô hình:** Pull-based Deployment.  
* **Quy tắc:**  
  * **Local:** Triển khai tự động ngay sau khi build thành công để Developer kiểm tra.  
  * **Cloud (Production):** Việc deploy lên môi trường Production (AWS) yêu cầu phê duyệt thủ công (Manual Approval).

### **4.3. Chi tiết các bước trong Pipeline**

1. **Code Commit:** Developer đẩy mã nguồn (Push code) lên GitLab Local.  
2. **Trigger Build:** GitLab gửi Webhook kích hoạt Job trên Jenkins.  
3. **Build & Local Release:**  
   * Jenkins build Docker Images và Helm Charts.  
   * Jenkins đẩy Artifacts (Image \+ Chart) lên **Harbor** (Local Registry).  
4. **Local Deployment (Tự động):**  
   * ArgoCD (Local) phát hiện Artifact mới trên Harbor (hoặc qua repo config).  
   * Tự động đồng bộ và triển khai lên **K8s Local** để kiểm thử.  
5. **Approval Gate (Quality Gate):**  
   * Jenkins tạm dừng và gửi yêu cầu phê duyệt đến CTO/PM.  
6. **Cloud Release (Sau khi Approve):**  
   * Sau khi CTO nhấn "Approve" trên Jenkins:  
   * Jenkins đẩy Image từ Harbor lên **AWS ECR**.  
   * Jenkins cập nhật Manifest/Helm values lên **GitOps Repo** (GitLab/GitHub).  
7. **Cloud Deployment (CD):**  
   * ArgoCD (quản lý cụm Cloud) phát hiện thay đổi trên GitOps Repo.  
   * Tự động đồng bộ (Sync) và cập nhật Rolling Update lên **AWS EKS**.

## **5\. KỊCH BẢN ỨNG PHÓ SỰ CỐ (DISASTER RECOVERY PLAN)**

Kịch bản này được kích hoạt khi Primary Site (AWS) gặp sự cố nghiêm trọng không thể phục hồi ngay lập tức.

### **5.1. Điều kiện kích hoạt (Trigger Condition)**

* Ứng dụng trên AWS không phản hồi (Time out).  
* Giả lập sự cố: Tắt cụm EKS hoặc scale replica của ứng dụng về 0\.

### **5.2. Quy trình Failover (Chuyển đổi dự phòng)**

1. **Phát hiện sự cố:** Hệ thống giám sát cảnh báo AWS Down.  
2. **Điều hướng Traffic:**  
   * *Trong môi trường Lab:* Quản trị viên truy cập CloudFlare Dashboard, trỏ DNS/Tunnel traffic về cụm Kubernetes Local.  
   * *Trong thực tế:* Sử dụng **Cloudflare Load Balancing** với Health Check để tự động điều hướng traffic về Local khi AWS không phản hồi.  
3. **Khôi phục:** Khi AWS hoạt động trở lại, traffic được điều hướng ngược lại Primary Site.

## **6\. THÔNG TIN MÃ NGUỒN & TÀI NGUYÊN (RESOURCES)**

Dưới đây là liên kết đến các kho lưu trữ mã nguồn và cấu hình phục vụ cho việc triển khai dự án:

| Hạng mục | Mô tả | Liên kết Repository |
| :---- | :---- | :---- |
| **Frontend** | Mã nguồn ReactJS | [Github FE Repo](https://github.com/ThongVu1996/lab-final-fe) |
| **Backend** | Mã nguồn Laravel | [Github BE Repo](https://github.com/ThongVu1996/lab-final-be) |
| **Cloud Config** | Mã nguồn Setup CI/CD cho Cloud | [Github Cloud Infra](https://github.com/ThongVu1996/lab-final) |
| **Full Project** | Mã nguồn đầy đủ (Local \+ Cloud) | [Github Full Lab](https://github.com/ThongVu1996/lab-final-full) |

--- 

## Triển khai ở Local
### Cài đặt Cloudflare Agent 
 - Đầu tiên chúng ta phải cài Helm Chart lên k8s trước [tại đây](https://github.com/ThongVu1996/cd-ci-lab/blob/master/argocd/argocd-with-helm.md#t%E1%BA%A1o-helm-chart-tr%C3%AAn-k8s) 
    ```bash
          helm repo add cloudflare https://cloudflare.github.io/helm-charts
          helm repo update
          helm install cloudflare-agent cloudflare/cloudflare-utils \
            --namespace cloudflare-agent --create-namespace \
            --set cloudflared.token=<YOUR_TUNNEL_TOKEN>
    ```
  -  `YOUR_TUNNEL_TOKEN` có thể được lấy từ đoạn `Install and run a connector` như trong ảnh

     ![token-cloud-flare-tunel](./token-cloud-flare-tunel.png)
 - Kiểm tra Cloudflare Agent đã được cài thành công chưa ta dùng lệnh
    ```bash
        kubectl get pods -A | grep cloud
    ```
 
   ![check-cloud-flare-agent-in-local](./check-cloud-flare-agent-in-local.png)

### Cài đặt Nginx Ingress 
 ```bash
    helm install ingress-nginx ingress-nginx/ingress-nginx \
      --namespace ingress-nginx --create-namespace \
      --set controller.service.type=ClusterIP \
      --set controller.watchIngressWithoutClass=false \
      --set controller.ingressClassResource.name=nginx \
      --set controller.ingressClassResource.enabled=true \
      --set controller.ingressClassResource.default=true \
      --timeout 30m0s
 ```
  - `--set controller.ingressClassResource.name=nginx` sẽ giúp Nginx sẽ chỉ lắng nghe những Ingress có khai báo đúng tên lớp (class name) mà nó quản lý.

 - Kiểm tra kết quả chúng ta dùng lệnh 
    ```bash
    kubectl get svc -n ingress-nginx
    ```
  
    ![check ingress-nginx in local](./check-ingress-nginx-local.png)

 - Ở đây chúng ta tạo ra controller với type là ClusterIP mà không phải là NodePort vì nhắm làm tăng tính bảo mật cho hệ thống.
 - Bởi vì với type là NodePort, Kubernetes sẽ mở một cổng tĩnh (thường từ 30000-32767) trên tất cả các Node (bao gồm cả Worker và đôi khi là Master) trong cụm. Hacker chỉ cần tìm ra IP của một Node bất kỳ là có thể quét cổng và tấn công trực tiếp vào dịch vụ của bạn.
 - NodePort yêu cầu bạn phải quản lý thủ công việc mở cổng trên tường lửa (Firewall/Security Group) của hệ thống hạ tầng. Nếu bạn lỡ tay mở "0.0.0.0/0", bất kỳ ai trên thế giới cũng có thể kết nối vào.
 - Trong môi trường nhiều người dùng (Multi-tenant), các đội nhóm khác nhau có thể vô tình mở các cổng NodePort trùng nhau hoặc để lộ các dịch vụ nhạy cảm (như Database) ra ngoài mà không biết.
 - ClusterIP: Dịch vụ chỉ có một địa chỉ IP ảo nội bộ. Không có cổng nào được mở trên máy vật lý/máy ảo (Node). Nó hoàn toàn "vô hình" trước mọi quét cổng từ bên ngoài Internet.
 - ClusterIP bắt buộc mọi traffic phải đi qua Cloudflare Tunnel, nơi đã có sẵn các lớp bảo vệ cực mạnh của Cloudflare trước khi đến được cụm k8s.
 - Với type là Cluster thì nó chỉ cho phép Traffic đi vào Nginx Ingress nếu nó đến từ Pod có nhãn là app: cloudflared. Tất cả các Pod khác trong cụm đều bị chặn.
 - Để cấu hình Cloudflare Tunnel cũng không cần phải đưa ra địa chỉ IP private (địa chỉ IP local trong cụm k8s), mà chỉ cần điền vào ô `URL` giá trị là `http://ingress-nginx-controller.ingress-nginx.svc.cluster.local:80`.

    ![config-cloud-flare-tunel](./token-cloud-flare-tunel.png)

- `<service-name>.<namespace>.svc.cluster.local` (đây chính là địa chỉ điển vào URL)
  - <service-name> (Tên Service): Đây là giá trị nằm trong cột NAME.
  - <namespace> (Không gian tên): Đây là giá trị bạn đã điền sau tham số -n khi chạy lệnh.

### Thiết lập biến môi trường
- Với laravel là backend khi triển khai chúng ta cần sử dụng biến môi trường qua file `.env`.
- Để làm được điều đó chúng ta sẽ phải tạo secret key cho k8s, và các biến đó sẽ được đọc trong các file minifest.
  ```bash
        kubectl create namespace yorisoi-local

        kubectl create secret generic yorisoi-secret \
          --namespace yorisoi-local \
          --from-literal=APP_ENV=‘production’ \
          --from-literal=APP_DEBUG='false’ \
          --from-literal=APP_URL=’domain’ \
          --from-literal=APP_KEY='base64:Thay_The_Bang_Key_Cua_Ban_Vao_Day' \
          --from-literal=JWT_SECRET=‘jwt_tao_bang_lenh_ php artisan jwt:secret\
          --from-literal=DB_CONNECTION='mysql' \
          --from-literal=DB_HOST='mysql-svc' \
          --from-literal=DB_PORT='3306' \
          --from-literal=DB_DATABASE='yorisoi_db' \
          --from-literal=DB_USERNAME='yorisoi_user' \
          --from-literal=DB_PASSWORD='MatKhauDbCuaBan' \
          --from-literal=MYSQL_ROOT_PASSWORD='MatKhauRootCuaBan' \
          --from-literal=MYSQL_PASSWORD='MatKhauDbCuaBan' \
          --from-literal=MYSQL_DATABASE='yorisoi_db' \
          --from-literal=MYSQL_USER='yorisoi_user'
  ```
  - Kiểm tra bằng lệnh
    ```bash
       kubectl describe secret yorisoi-secret -n yorisoi-local
    ```
  - Với AWS do ta triển khai DB bằng RDS nên các thông số sẽ ít hơn
    ```bash
      kubectl create secret generic yorisoi-secret \
      --namespace yorisoi-prod \
      --from-literal=APP_ENV='production' \
      --from-literal=APP_KEY='base64:eZ5f9kN7uDSUsnyxoQwISBdgsfHb3XJj4UW4Be7YBlE=' \
      --from-literal=JWT_SECRET='xAR12UlxQenjBfOPMTDIjRewTUJlKRu8sjU7gyJ6A8fYkS7v6PpXPI1xEMlKZ9M0' \
      --from-literal=DB_CONNECTION='mysql' \
      --from-literal=DB_HOST='lab-final-db.cn46i6qw2flt.ap-southeast-1.rds.amazonaws.com' \
      --from-literal=DB_DATABASE='yorisoi_db' \
      --from-literal=DB_PORT='3306' \
      --from-literal=DB_USERNAME='yorisoi_user' \
      --from-literal=DB_PASSWORD='thaolinh123'
    ```
  - Kiểm tra bằng lệnh tương tự ở trên chỉ thay namespace thành yorisoi-prod

### Thiết lập jenkins
- Ta đã biết cách cài đặt jenkins
- Với nội dụng Jenkins ta xem [tại đây](https://github.com/ThongVu1996/lab-final-full/blob/main/Jenkinsfile)
- Sau khi code được đẩy lên gitlab -> Jenkins sẽ tiến hành build
- Ta có thể vào `Open Blue Ocean` ở trong ảnh để xem quá trình build.

  ![blue-ocean](./blue-ocean.png)
- Kết quả của quá trình build.

  ![jenkins-build](./jenkins-build.png)

### Kiểm tra Harbor
- Ta sẽ thấy images và helm chart được đẩy lên tương ứng với version trong Jenkins

   ![harbor-project](./harbor-project.png)

   ![harbor-images](./harbor-images.png)

   ![multilpe-plate-form](./multilpe-plate-form.png)

 - Trong ảnh ta có thể thấy images có hai phiên bản là amd và arm. Vì trong quá trình build đã sử dụng buildx để build multiple plataform.
 - Với 2 iamges được tạo ra như vậy thì khi k8s chạy nó sẽ tự biết phải lấy bản nào để có thể dùng được (dựa trên chip của máy host đang cài k8s).
 - Tuy nhiên buildx sẽ làm tốc độ giảm đi, nên với môi trường product chúng ta nên sử dụng máy host có kernal là chip amd vì thường thì các cloud đa phần chỉ support phiên bản chip amd.

### Kiểm tra ECR
- Kiểm tra các repo

   ![ECR-list-repo](./ECR-list-repo.png)

   ![ECR-images](./ECR-images.png)

### Kiểm tra minifest github
 - Kiểm tra repo ta sẽ thấy code được đẩy lên

   ![manifest-cloud](./manifest-cloud.png)

   ![mainifest-values-config](./mainifest-values-config.png)

### Tiến hành sử dụng argoCD để deploy ứng dụng lên cụm K8s
 - Nhớ tạo nơi chứa data cho Mysql ở cụm node quy định trong file config ở đây là k8s-master-2
  ```bash
    # Đây là do cấu hình vậy
    mkdir /data/mysql-pv
  ```
- Cách kết nối repo và tạo application thì các bài lab trước đã có hướng dẫn rồi
  (tham khảo [tại đây](https://github.com/ThongVu1996/cd-ci-lab/blob/master/argocd/argocd-with-helm.md), search `Bước 4: Triển khai với helm lưu trữ trên Harbor bằng ArgoCD` cho nhanh thấy)
- Với application ngoài cách tạo bằng tay ta hoàn toàn có thể sử dụng 1 file cấu hình bằng yaml rồi chạy lệnh
  ```bash
      kubectl apply -f ten_file_config.yaml
  ```

  ```bash
      apiVersion: argoproj.io/v1alpha1
      kind: Application
      metadata:
        name: yorisoi-local
        namespace: argocd
      spec:
        project: default
        source:
          # 1. Trỏ về Harbor
          repoURL: 'harbor.local.thongdev.site/lab-final'
          # 2. Chọn CHART GÓI (Wrapper) thay vì Chart Gốc
          chart: yorisoi-local 
          # 3. Version động
          targetRevision: '0.1.*'
          
          # KHÔNG CẦN helm: values Ở ĐÂY NỮA
          # Vì mọi thứ đã nằm trong yorisoi-local/values.yaml rồi
        
        destination:
          server: 'https://kubernetes.default.svc'
          namespace: yorisoi-local
        syncPolicy:
          automated:
            prune: true
            selfHeal: true
          syncOptions:
            - CreateNamespace=true
  ```
- Kết quả như hình 
  
   ![argocd-local-app](./argocd-local-app.png)

   ![argocd-local-app-detail](./argocd-local-app-detail.png)
 - Truy cập vào trang web ta sẽ thấy kết quả 
  - Thành công:
    
    ![app-local-success](./app-local-success.png)
  - Thất bại
   
    ![app-local-fail](./app-local-fail.png)
  - Kiểm tra đúng là app đã chạy ở local ta dùng lệnh sau để theo dõi
      ```bash
        kubectl logs -f -l app.kubernetes.io/name=ingress-nginx -n ingress-nginx
      ```
   - Nó sẽ hiển thị thêm log mỗi khi bạn F5 trang
  
     ![log-app-local](./log-app-local.png)
---

## Triển khai ở Cloud
- Ở phía local chúng ta triển khai DB là mysql lên container, còn ở trên AWS chúng ta triển khai nó lên AWS RDS
- Chi tiết về cách cài AWS RDS xem [tại đây](https://github.com/ThongVu1996/cd-ci-lab/blob/master/final/insall-AWS-RDS.md)
  ![RDS](./RDS.png)

- Trong code ở [repo](https://github.com/ThongVu1996/lab-final) đã bao gồm việc các file minifest để cấu hình cho k8s trên AWS. Nên ở đây chúng ta chỉ cần tạo app và rồi trỏ cloudflare về AWS là xong
- Đầu tiên ta cũng lấy address của AWS ELB 
  ```bash
      kubectl get ingress -A
  ```
 
  ![address-aws-alb](./address-aws-alb.png)
 - Ở trên cloudflare ta tạo 1 bản ghi với type là CNAME với Name là sub domain (eg: dr) và Target chính là địa chỉ ta lấy ở trên, sau đó ấn Save là được.
 - Kết quả cũng sẽ như hình ở bên local thôi

 --- 

## Kịch bản DR
### Phân tích
- Như đầu bài lab ta có để cập đến thì AWS chết -> đưa nó về local
- Nhưng ở đây ta tiến hành deploy local trước nên chúng ta làm ngược lại là đưa từ local lên AWS (kết quả cũng sẽ tương đương nhau).
- Mục tiêu là người dùng chỉ cần truy cập vào trang web vẫn thấy dùng được, chứ họ không hề biết là hệ thống đang được chạy ở AWS hay local.
- Để đạt được điều đó thì 1 lưu ý quan trọng là tại thời điểm trang web chỉ trỏ lưu lượng về một nơi duy nhất.

### Thực hành
- Bước 1: Ta vào bên trong bản ghi của cloudflare tunnel và chuyển subdomain sang giá trị khác như hình

  ![change-subdomain-local](./change-subdomain-local.png)
  
  ![results-change-subdomain](./results-change-subdomain.png)
 - Bước 2: Kiểm tra xem trang web đã chết chưa. Kết quả như hình là đúng.
  
  ![link-app-die](./link-app-die.png)
 - Bước 3: Tiến hành tạo bản ghi CNAME với tên subdomain của trang web là targer là kết quả lấy được như đã để cập ở phần triển khai Cloud ở trên (Vào DNS -> Records)
 
    ![record-for-aws](./record-for-aws.png)

    ![record-for-aws-1](./record-for-aws-1.png)

 - Bước 4: Kiểm tra lại trang web xem đã lên chưa, đợi khoảng 30-60s để cloudflare cập nhật, kết quả trang web lên như hình.

   ![results-app-aws-1](./results-app-aws-1.png)

   ![results-app-aws-2](./results-app-aws-2.png)

- Bước 5: Tương tự với bên local, để kiểm tra nó đang thực sự dùng của AWS thì ta cũng dùng lệnh (chạy ở máy kết nối với AWS EKS)
    ```bash
    kubectl logs -f -l app.kubernetes.io/name=ingress-nginx -n ingress-nginx
     ```
    
   ![confirm-connect-aws](./confirm-connect-aws.png)
   - Mỗi lần ta đăng nhập vào hệ thống để thì log sẽ in ra thêm 
   
   ![aws-logs](./aws-logs.png)

--- 

## Phân tích chuyên sâu:
- Vì khi triển khai ở local ta deploy DB vào 1 container, nên ta cần tạo ra 1 thư mục nhằm mount data ra máy local tránh trường hợp mất dữ liệu khi pod chết.
- Ở đây chúng ta dùng backend lả Laravel, vì vậy khi deploy code lên nó sẽ cần luôn phải chạy lệnh `php artisan migrate --seed --force` để cập nhật các trường mới trong DB nếu code có update.
- Chính vì vậy chúng ta cần tạo ra 1 job, và job đó nó sẽ chay sau khi mà ta đã tạo xong service dành cho db và be (laravel). Ta có thể xem kỹ nó [tại đây](https://github.com/ThongVu1996/lab-final-full/blob/main/charts/yorisoi-stack/templates/migration-job.yaml)
- Nginx có vai trò nhận request từ client -> chuyển đến cho server. Vì vậy khi xây dựng dockerfile chúng ta phải có nginx trong đó.
  - Tuy nhiên sẽ có sự khác nhau giữa nginx của BE và FE
   - Với FE thì các file các file tĩnh và chúng không thể tự chạy (cung cấp data cho trình duyệt) mà chúng cần có 1 webserver để làm việc đó, mà ở đây là Nginx.
   - Việc gộp chung FE vào image Nginx thực chất là copy các file tĩnh vào thư mục mặc định của Nginx để nó "giao hàng" cho người dùng.
   - Còn với backend thì khác, nó cần một môi trường để chạy code. Còn nginx đóng vai trò là webserver để nhận request -> gửi đến backend -> backend chạy code -> trả cho nginx -> trả lại cho client
   - Chính vì vậy mà nginx và backend tách thành 2 images riêng, giúp images nhẹ đi, khi cần scale backend khi lưu lượng lớn cũng dễ dàng vì lúc đó nó không kèm nginx đi kèm.
   - Khi nhìn vào trong argoCD ta sẽ thấy trong POD backend sẽ có 2/2 (1 container be, 1 container nginx), còn fe thì chỉ có 1 là vậy.
   
    ![backend-pod](./backend-pod.png)
   
    ![frontend-pod](./frontend-pod.png)

