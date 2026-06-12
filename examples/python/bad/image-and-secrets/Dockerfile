# BAD Python/FastAPI image — full base, baked-in secret, runs as root, no healthcheck.

# DF-13 full image (use -slim) · DF-15 unpinned (`latest`)
FROM python:latest
WORKDIR /app
# DF-01 secret in ENV
ENV SECRET_KEY=my-django-secret-key
# DF-06 source copied before pip install — busts the cache
COPY . .
# DF-18 no --no-cache-dir — wheel cache bloats the image
RUN pip install -r requirements.txt
# no USER                                 # DF-02 runs as root
# no HEALTHCHECK                          # DF-22 long-running server, no healthcheck
# no .dockerignore in this dir            # DF-11
EXPOSE 8000
# DF-04 shell form — sh is PID 1, signals lost
CMD uvicorn app.main:app --host 0.0.0.0 --port 8000
