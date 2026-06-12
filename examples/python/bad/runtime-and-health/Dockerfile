# BAD Python runtime — missing --chown, a broken healthcheck, duplicate CMD.

FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

FROM python:3.12-slim AS runtime
WORKDIR /app
RUN addgroup --system app && adduser --system --ingroup app app
COPY --from=builder /app ./                       # DF-21 COPY without --chown before USER app → unwritable as `app`
USER app
EXPOSE 8000
# slim has no curl → DF-16 fires on this curl healthcheck
HEALTHCHECK CMD curl -f http://localhost:8000/health || exit 1
CMD ["uvicorn", "app.main:app", "--port", "8000"]   # DF-24 with the next line — two CMD
CMD ["uvicorn", "app.main:app", "--port", "8001"]   # DF-24 only the last CMD runs
