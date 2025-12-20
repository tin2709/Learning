

# 🚀 CI/CD Pipeline Documentation: Next.js + Jenkins + Docker

Tài liệu này mô tả quy trình Tự động hóa Tích hợp và Triển khai (CI/CD) cho dự án **AI Resume Builder**. Hệ thống sử dụng **Jenkins** chạy trên Docker, tự động build ứng dụng Next.js, đóng gói thành **Docker Image** và đẩy lên **Docker Hub**.

---

## 🛠️ 1. Kiến trúc hệ thống
Quy trình hoạt động như sau:
1.  **Dev:** Push code lên GitHub.
2.  **Trigger:** GitHub Webhook (qua Ngrok) báo tín hiệu cho Jenkins.
3.  **Jenkins:**
    *   Kéo code về (Checkout).
    *   Cài đặt thư viện (npm install).
    *   Xây dựng Docker Image (kèm xử lý biến môi trường Prisma).
    *   Đăng nhập và đẩy (Push) Image lên Docker Hub.
4.  **Result:** Sản phẩm cuối cùng là một Docker Image sẵn sàng deploy ở bất cứ đâu.

---

## ⚙️ 2. Cài đặt môi trường Jenkins

Jenkins được chạy bằng Docker trên máy Local, nhưng được cấp quyền đặc biệt để có thể gọi lệnh Docker của máy chủ (Docker-in-Docker technique).

### Lệnh khởi chạy Jenkins Server:
```bash
docker run -d \
  -p 8080:8080 -p 50000:50000 \
  --name my-jenkins \
  -u root \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts-jdk17
```

**Giải thích:**
*   `-u root`: Chạy với quyền cao nhất để được phép cài đặt phần mềm.
*   `-v /var/run/docker.sock:/var/run/docker.sock`: **Quan trọng nhất.** Cho phép Jenkins "mượn" Docker Daemon của máy chủ để thực hiện lệnh build image.
*   `-v jenkins_home...`: Map volume để giữ lại dữ liệu khi tắt container.

### Cài đặt Docker Client bên trong Jenkins:
Để Jenkins hiểu được lệnh `docker build`, cần cài đặt CLI bên trong container:
```bash
docker exec -it -u root my-jenkins bash -c "apt-get update && apt-get install -y docker.io"
```

---

## 📝 3. Cấu hình Project

### A. Dockerfile
File cấu hình để đóng gói ứng dụng. Đặt tại thư mục gốc của source code (`ai-resume-builder/Dockerfile`).
*Lưu ý: Cần xử lý `ARG` để nhận biến môi trường lúc build.*

```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm install
COPY . .

# Nhận biến DB URL từ Jenkins để Prisma chạy được
ARG DATABASE_URL
ENV DATABASE_URL=$DATABASE_URL

RUN npx prisma generate
RUN npm run build

# ... (Phần Runner để chạy ứng dụng)
```

### B. Jenkinsfile (Pipeline Script)
Kịch bản chạy tự động của Jenkins.

**Các điểm nhấn kỹ thuật:**
1.  **Credentials:** Sử dụng `docker-hub-login` (Token) để bảo mật, không lộ password trong code.
2.  **Build Args:** Truyền `DATABASE_URL` vào Docker để bypass lỗi Prisma Generate.
3.  **Clean Workspace:** Tự động dọn dẹp file rác trước khi build.

```groovy
pipeline {
    agent any
    tools { nodejs 'node-22' } // Cấu hình NodeJS Tool trong Jenkins
    
    environment {
        DOCKER_USER = 'trungtin2003' 
        IMAGE_NAME = 'ai-resume-builder'
        // URL giả lập hoặc thật để Prisma generate schema
        DATABASE_URL="mongodb+srv://admin:..." 
    }
    
    stages {
        stage('Clean Workspace') { steps { cleanWs() } }
        
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/tin2709/CVBuilder.git'
            }
        }
        
        stage('Build & Push Docker Image') {
            steps {
                dir('ai-resume-builder') {
                    script {
                        withCredentials([usernamePassword(credentialsId: 'docker-hub-login', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER_LOGIN')]) {
                            
                            // 1. Login Docker Hub
                            sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER_LOGIN --password-stdin"
                            
                            // 2. Build Image (Truyền biến môi trường vào)
                            sh "docker build --build-arg DATABASE_URL='${DATABASE_URL}' -t $DOCKER_USER/$IMAGE_NAME:latest ."
                            
                            // 3. Push Image
                            sh "docker push $DOCKER_USER/$IMAGE_NAME:latest"
                        }
                    }
                }
            }
        }
    }
}
```

---

## 🔐 4. Quản lý Bảo mật (Credentials)

Để Jenkins đăng nhập Docker Hub mà không lộ mật khẩu:
1.  Trên Docker Hub: Tạo **Access Token** (Settings -> Security -> New Access Token).
2.  Trên Jenkins: Vào **Manage Jenkins -> Credentials**.
    *   Kind: Username with password.
    *   ID: `docker-hub-login`.
    *   Password: Dán chuỗi Token vừa tạo vào đây.

---

## 🔄 5. Tự động hóa (Webhook)

Để Jenkins tự chạy khi có code mới (Push event):
1.  **Ngrok:** Mở đường hầm kết nối Localhost ra Internet.
    ```bash
    ngrok http 8080
    ```
2.  **GitHub Repo Settings:**
    *   Vào **Webhooks** -> Add Webhook.
    *   URL: `https://<ngrok-url>/github-webhook/`
    *   Content type: `application/json`.
3.  **Jenkins Job Configuration:**
    *   Tick chọn: `GitHub hook trigger for GITScm polling`.

---

## ✅ Kết quả

Sau khi pipeline chạy thành công (**SUCCESS**):
*   Docker Image được đẩy lên: [https://hub.docker.com/repositories/trungtin2003](https://hub.docker.com/repositories/trungtin2003)
*   Để chạy ứng dụng trên bất kỳ máy nào, chỉ cần gõ lệnh:
    ```bash
    docker run -p 3000:3000 trungtin2003/ai-resume-builder:latest
    ```

---
*Documented by Pham Trung Tin - 2025*