# GOOD twin of bad/image-and-secrets — `/docker-emmet examples/python/good` reports no issues.

FROM python:3.12.7-slim AS base
WORKDIR /app
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1
# manifest first → install layer stays cached
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM base AS runtime
RUN addgroup --system app && adduser --system --ingroup app app
# owned by the runtime user — no permission errors at startup
COPY --chown=app:app . .
USER app
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://localhost:8000/health').status==200 else 1)"
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
# SECRET_KEY injected at runtime by the orchestrator, never baked into the image
