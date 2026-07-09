FROM python:3.11-slim-bookworm

WORKDIR /app

RUN apt-get update && apt-get install --no-install-recommends -y dnsutils && \
    rm -rf /var/lib/apt/lists/*

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

RUN python -m pip install --no-cache-dir --upgrade pip
COPY requirements.txt requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

COPY . /app/
EXPOSE 8000
RUN python3 /app/manage.py migrate
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers","6", "pygoat.wsgi"]
