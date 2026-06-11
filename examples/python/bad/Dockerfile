# BAD Python/FastAPI Dockerfile — `/docker-emmet examples/python/bad`.

FROM python:3.12                          # DF-13 full image — use -slim for runtime

WORKDIR /app

ENV SECRET_KEY=my-django-secret-key       # DF-01 secret in ENV

COPY . .                                  # DF-06 source copied before pip install
RUN pip install -r requirements.txt       # DF-18 no --no-cache-dir — wheel cache bloats image

# no USER                                 # DF-02 runs as root
# no HEALTHCHECK on a long-running server # DF-22
# no .dockerignore                        # DF-11

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
