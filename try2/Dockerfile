FROM python:3.12-slim

COPY --from=ghcr.io/astral-sh/uv:0.7.13 /uv /bin/uv

ENV PYTHONUNBUFFERED=1 \
    PATH="/app/.venv/bin:$PATH"

ENV SERVER_PORT=8000

WORKDIR /app

COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-install-project

COPY . .

RUN uv sync --locked

EXPOSE 8009

CMD ["sh", "-c", "exec uvicorn main:app --host 0.0.0.0 --port \"$SERVER_PORT\""]
