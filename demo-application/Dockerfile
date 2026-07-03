FROM python:3.13.3-bookworm

ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN useradd -m appuser
USER appuser

EXPOSE 8000

CMD ["gunicorn", "simply.wsgi:application", "--bind", "0.0.0.0:8000"]