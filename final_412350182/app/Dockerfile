FROM python:3.12-slim

WORKDIR /app

# 建立非 root 使用者與群組
RUN groupadd -g 1000 appuser && \
    useradd -r -u 1000 -g appuser appuser

# 【快取最佳化】先複製不常變動的相依性宣告
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 【快取最佳化】最後才複製會頻繁修改的原始碼
COPY app.py .

# 切換至非 root 使用者
USER appuser

EXPOSE 8080

CMD ["python", "app.py"]
