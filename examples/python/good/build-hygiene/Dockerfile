# GOOD twin of bad/build-hygiene — a clean builder, no findings.

FROM python:3.12-slim AS builder
WORKDIR /app
RUN apt-get update \
 && apt-get install -y --no-install-recommends gcc libpq-dev \
 && rm -rf /var/lib/apt/lists/*
COPY tool.sha256 ./
RUN curl -fsSL https://example.com/tool.tar.gz -o tool.tar.gz \
 && sha256sum -c tool.sha256 \
 && tar -xzf tool.tar.gz
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
# COPY, not ADD — explicit, no surprise tar-extraction
COPY assets ./assets
COPY . .

FROM python:3.12-slim AS runtime
WORKDIR /app
RUN addgroup --system app && adduser --system --ingroup app app
COPY --from=builder --chown=app:app /app ./
USER app
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://localhost:8000/health').status==200 else 1)"
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
