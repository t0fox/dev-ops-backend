FROM python:3.12-slim

RUN useradd --create-home --uid 1000 appuser

WORKDIR /app

COPY app.py .

EXPOSE 8080

USER appuser

CMD ["python", "app.py"]
