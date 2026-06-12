# BAD Python/FastAPI image — full base, baked-in secret, runs as root, no healthcheck.

FROM python:latest                        # DF-13 full image (use -slim) · DF-15 unpinned (`latest`)
WORKDIR /app
ENV SECRET_KEY=my-django-secret-key       # DF-01 secret in ENV
COPY . .                                  # DF-06 source copied before pip install — busts the cache
RUN pip install -r requirements.txt       # DF-18 no --no-cache-dir — wheel cache bloats the image
# no USER                                 # DF-02 runs as root
# no HEALTHCHECK                          # DF-22 long-running server, no healthcheck
# no .dockerignore in this dir            # DF-11
EXPOSE 8000
CMD uvicorn app.main:app --host 0.0.0.0 --port 8000   # DF-04 shell form — sh is PID 1, signals lost
