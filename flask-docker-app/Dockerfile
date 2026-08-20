FROM python:3.8-slim

WORKDIR /app

COPY . .

RUN apt-get update && apt-get install -y \
    pkg-config \
    default-libmysqlclient-dev \
    build-essential

RUN pip install flask mysqlclient

EXPOSE 5002

CMD ["python", "app.py"]