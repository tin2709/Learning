

# 🚀 7 Dockerfile Patterns for Python: From Minutes to Seconds

Tài liệu này tổng hợp 7 mẫu Dockerfile thực chiến giúp tối ưu hóa thời gian build, giảm kích thước Image và tăng tính bảo mật cho ứng dụng Python.

---

## 🏗️ Triết lý: "Tòa nhà 3 tầng"
Để hiểu tại sao các mẫu dưới đây lại nhanh, hãy tưởng tượng Dockerfile như một tòa nhà:
1.  **Tầng 1 - Móng (Base Image):** Hệ điều hành và công cụ hệ thống (Ít thay đổi).
2.  **Tầng 2 - Tường (Dependencies):** Thư viện Python (Thay đổi vài tuần một lần).
3.  **Tầng 3 - Nội thất (Source Code):** Code ứng dụng (Thay đổi liên tục).

**Nguyên tắc vàng:** Đặt những thứ ít thay đổi lên đầu, hay thay đổi xuống cuối.

---

## 1. Tối ưu thứ tự Layer & Base Image nhẹ
Sử dụng `python-slim` để giảm dung lượng và sắp xếp `requirements.txt` trước khi `COPY` mã nguồn.

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-slim AS base

ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1

# Cài thư viện hệ thống trước (ít đổi)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
```

---

## 2. Multi-stage Build (Biên dịch một lần)
Tách biệt quá trình biên dịch (build) và quá trình chạy (runtime) để loại bỏ các công cụ thừa như `gcc`.

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /wheels
COPY requirements.txt .
RUN pip wheel --no-cache-dir --wheel-dir=/wheels -r requirements.txt

FROM python:3.12-slim AS runtime
WORKDIR /app
COPY --from=builder /wheels /wheels
RUN pip install --no-cache-dir --no-index --find-links=/wheels /wheels/*
COPY . .
```

---

## 3. Sử dụng Cache của BuildKit
Tận dụng tính năng mount cache để tránh tải lại thư viện npm/pip mỗi lần build.

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-slim AS base
WORKDIR /app

# Bật cache cho pip và apt
RUN --mount=type=cache,target=/root/.cache/pip \
    --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y curl

COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

---

## 4. Khóa phiên bản với Constraints
Sử dụng `constraints.txt` để đảm bảo tính nhất quán của layer cache.

```dockerfile
COPY requirements.txt constraints.txt ./
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt -c constraints.txt
```

---

## 5. Sử dụng `uv` - Công cụ siêu tốc
`uv` (viết bằng Rust) nhanh hơn pip gấp nhiều lần trong việc giải quyết dependencies.

```dockerfile
FROM python:3.12-slim AS base
WORKDIR /app
RUN --mount=type=cache,target=/root/.cache/uv \
    pip install uv

COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/uv \
    uv pip install -r requirements.txt --system
```

---

## 6. Multi-target (Dev & Prod trong một file)
Quản lý cả môi trường phát triển và môi trường thực tế một cách gọn gàng.

```dockerfile
FROM python:3.12-slim AS base
WORKDIR /app
COPY requirements.txt ./
RUN pip install -r requirements.txt

# Target cho Development
FROM base AS dev
RUN pip install watchfiles ipython
COPY . .
CMD ["python", "-m", "app", "--reload"]

# Target cho Production
FROM base AS prod
RUN useradd -m appuser
USER appuser
COPY . .
CMD ["python", "-m", "app"]
```

---

## 7. Vũ khí bí mật: `.dockerignore`
Loại bỏ file rác để tránh làm hỏng cache vô lý.

**File `.dockerignore`:**
```text
.git
__pycache__/
*.pyc
.env
tests/
Dockerfile
```

---

## 🏆 Mẫu Dockerfile Hoàn Chỉnh (Production Ready)

Đây là mẫu kết hợp tất cả các kỹ thuật trên để đạt hiệu suất cao nhất:

```dockerfile
# syntax=docker/dockerfile:1.7

# Giai đoạn 1: Build Wheels
FROM python:3.12-slim AS wheels
WORKDIR /w
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y build-essential

COPY requirements.txt constraints.txt ./
RUN --mount=type=cache,target=/root/.cache/pip \
    pip wheel --no-cache-dir --wheel-dir=/w -r requirements.txt -c constraints.txt

# Giai đoạn 2: Runtime
FROM python:3.12-slim AS runtime
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app

RUN useradd -m appuser
COPY --from=wheels /w /wheels
COPY requirements.txt constraints.txt ./

RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-index --find-links=/wheels -r requirements.txt -c constraints.txt

COPY src/ ./src/
USER appuser

HEALTHCHECK --interval=30s --timeout=3s \
  CMD python -c "import socket; s=socket.socket(); s.connect(('localhost', 8000))" || exit 1

CMD ["python", "-m", "src.main"]
```

---

## 💡 Kết quả thực tế
| Kỹ thuật | Thời gian Build lại (sau khi sửa code) |
| :--- | :--- |
| **Kiểu cũ (Copy toàn bộ)** | ~ 2 - 3 phút |
| **Sắp xếp lại Layer** | ~ 30 - 45 giây |
| **Dùng BuildKit + uv** | **~ 6 - 10 giây** |

---