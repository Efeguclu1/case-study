FROM python:3.11-slim

RUN groupadd -r etl && useradd -r -g etl etl

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY --chown=etl:etl ETL.py .

USER etl
CMD ["python", "ETL.py"]
