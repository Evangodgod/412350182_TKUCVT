# ---------- builder stage ----------
FROM python:3.12 AS builder
WORKDIR /build
COPY app/requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ---------- runtime stage ----------
FROM python:3.12-slim AS runtime
WORKDIR /app

# 把 /install 底下的編譯依賴搬到 runtime 乾淨的對應位置
COPY --from=builder /install/bin /usr/local/bin
COPY --from=builder /install/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY app/ .

# 安全伏筆：切換成非 root 用戶
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 80
ENTRYPOINT ["python"]
CMD ["app.py"]
