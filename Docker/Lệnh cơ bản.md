

# Docker & Jenkins: Từ Cơ Bản Đến Nâng Cao

Tài liệu này cung cấp các câu lệnh Docker phổ biến và hướng dẫn cách kết hợp Docker vào Pipeline của Jenkins để tự động hóa quy trình Build, Test và Deploy.

---

## 1. Lệnh Docker Cơ Bản (Quản lý Container)

Dành cho người mới bắt đầu để làm quen với vòng đời của một Container.

| Lệnh | Giải thích |
| :--- | :--- |
| `docker --version` | Kiểm tra phiên bản Docker hiện tại. |
| `docker pull <image>` | Tải một Image từ Docker Hub về máy (VD: `docker pull nginx`). |
| `docker run -d --name my_cont -p 8080:80 nginx` | Chạy container từ image nginx ở chế độ chạy ngầm (`-d`), đặt tên là `my_cont` và mapping port 8080 của máy host vào port 80 của container. |
| `docker ps` | Liệt kê các container đang chạy. |
| `docker ps -a` | Liệt kê tất cả container (kể cả đã dừng). |
| `docker stop <id/name>` | Dừng một container đang chạy. |
| `docker start <id/name>` | Khởi động lại một container đã dừng. |
| `docker rm -f <id/name>` | Xóa bỏ hoàn toàn một container (thêm `-f` để ép buộc xóa khi đang chạy). |

---

## 2. Quản lý Images & Dockerfile (Trung cấp)

Lệnh dùng để xây dựng và quản lý các bản đóng gói ứng dụng.

*   **Xây dựng Image từ Dockerfile:**
    ```bash
    docker build -t my-app:v1 .
    ```
    *(Dấu `.` là đường dẫn đến thư mục chứa Dockerfile)*

*   **Quản lý Image:**
    *   `docker images`: Liệt kê các image hiện có.
    *   `docker rmi <image_id>`: Xóa một image.
    *   `docker tag my-app:v1 myrepo/my-app:v1`: Gắn thẻ để chuẩn bị push lên Registry.
    *   `docker push myrepo/my-app:v1`: Đẩy image lên Docker Hub hoặc Private Registry.

*   **Kiểm tra & Debug:**
    *   `docker logs -f <name>`: Xem log của container theo thời gian thực.
    *   `docker exec -it <name> sh`: Truy cập vào terminal bên trong container đang chạy.
    *   `docker inspect <name>`: Xem chi tiết cấu hình (IP, Volume, Network) của container.

---

## 3. Docker Compose (Phức tạp/Đa container)

Dùng để quản lý ứng dụng có nhiều service (VD: Web + DB).

*   **`docker-compose up -d`**: Khởi tạo và chạy tất cả các service định nghĩa trong file `docker-compose.yml`.
*   **`docker-compose down`**: Dừng và xóa bỏ toàn bộ container, network liên quan.
*   **`docker-compose ps`**: Xem trạng thái các service trong cụm compose.

---

## 4. Kết hợp Docker với Jenkins (CI/CD)

Việc kết hợp Docker vào Jenkins giúp môi trường Build luôn sạch sẽ và đồng nhất.

### 4.1. Chạy Jenkins bằng Docker
Để cài đặt Jenkins nhanh nhất bằng Docker:
```bash
docker run -d -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home --name jenkins jenkins/jenkins:lts
```

### 4.2. Docker bên trong Jenkins Pipeline (Declarative Pipeline)
Đây là cách phổ biến nhất để sử dụng Docker trong Jenkins.

**Kịch bản:** Build một ứng dụng Node.js, đóng gói thành Docker Image và push lên Docker Hub.

```groovy
pipeline {
    agent any

    environment {
        // Khai báo Image name và Docker Hub credentials ID
        DOCKER_IMAGE = "username/my-nodejs-app"
        REGISTRY_CREDS = "docker-hub-credentials-id"
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/your-repo/app.git'
            }
        }

        stage('Build Image') {
            steps {
                script {
                    // Build image với tag là số thứ tự của build (BUILD_NUMBER)
                    appImage = docker.build("${DOCKER_IMAGE}:${BUILD_NUMBER}")
                }
            }
        }

        stage('Test Image') {
            steps {
                script {
                    // Chạy test bên trong container vừa build
                    appImage.inside {
                        sh 'npm install && npm test'
                    }
                }
            }
        }

        stage('Push to Registry') {
            steps {
                script {
                    // Login và Push image lên Docker Hub
                    docker.withRegistry('', REGISTRY_CREDS) {
                        appImage.push()
                        appImage.push("latest") // Push thêm bản latest
                    }
                }
            }
        }

        stage('Cleanup') {
            steps {
                // Xóa image ở local để giải phóng dung lượng
                sh "docker rmi ${DOCKER_IMAGE}:${BUILD_NUMBER}"
            }
        }
    }
}
```

---

## 5. Các lệnh tối ưu hóa & Dọn dẹp (Nâng cao)

Khi chạy Jenkins trong thời gian dài, Docker sẽ chiếm rất nhiều dung lượng đĩa cứng. Bạn cần các lệnh này:

1.  **Dọn dẹp hệ thống:**
    *   `docker system prune -f`: Xóa tất cả container đã dừng, network thừa và cache không dùng đến.
    *   `docker image prune -a --filter "until=24h"`: Xóa tất cả các image không dùng đến được tạo trước 24 giờ.

2.  **Giới hạn tài nguyên (Thường dùng trong Jenkins agent):**
    ```bash
    docker run -d --cpus="1.5" --memory="512m" my-app
    ```
    *Lệnh này giới hạn container chỉ dùng tối đa 1.5 CPU và 512MB RAM.*

3.  **Tích hợp Docker-in-Docker (DinD):**
    Nếu Jenkins chạy bên trong container và bạn muốn nó có thể build Docker image, bạn cần mount socket của Docker máy host vào:
    `-v /var/run/docker.sock:/var/run/docker.sock`

---

## Giải thích thuật ngữ quan trọng:
*   **Dockerfile:** File text chứa các chỉ dẫn để tạo ra Image.
*   **Image:** Một bản đóng gói tĩnh chứa code, môi trường chạy.
*   **Container:** Một thực thể (instance) của Image đang chạy.
*   **Jenkinsfile:** File định nghĩa quy trình Pipeline của Jenkins.
*   **Credentials:** Nơi lưu trữ an toàn Username/Password của Docker Hub t