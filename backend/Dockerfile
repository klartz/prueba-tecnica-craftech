FROM python:3.8.3-alpine

WORKDIR /usr/src/app

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

RUN pip install --upgrade pip
COPY ./requirements.txt .
RUN apk add --update --no-cache postgresql-client
RUN apk add --update --no-cache --virtual .tmp-build-deps \
  gcc \
  libc-dev \
  linux-headers \
  postgresql-dev \
  musl-dev \
  zlib zlib-dev \
  && pip install -r requirements.txt \
  && apk del --no-cache .tmp-build-deps

COPY ./entrypoint.sh .
COPY . .

ENTRYPOINT "sh" "/usr/src/app/entrypoint.sh"
