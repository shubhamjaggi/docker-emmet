# BAD Python builder — no WORKDIR, sudo, an unverified download, dirty apt layers.

FROM python:3.12-slim AS builder
# DF-23 no WORKDIR in either stage — everything runs from /
# DF-12 ADD for a local copy — use COPY
ADD ./assets /build/assets
# DF-08 update split from install
RUN apt-get update
# DF-07 cache not cleaned · DF-09 no --no-install-recommends · DF-10 first install
RUN apt-get install -y gcc
# DF-07 · DF-09 · DF-10 second separate install
RUN apt-get install -y libpq-dev
# DF-03 sudo in container · DF-05 download with no checksum
RUN sudo curl -fsSL https://example.com/tool.tar.gz -o tool.tar.gz \
    && tar -xzf tool.tar.gz
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

FROM python:3.12-slim AS runtime
COPY --from=builder --chown=1000:1000 /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder --chown=1000:1000 /app /srv
USER 1000
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://localhost:8000/health').status==200 else 1)"
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
