FROM python:3.8.5-slim

WORKDIR /server

ADD ./server/requirements/prod.txt /server/prod.txt

RUN pip install -r prod.txt

COPY ./server .
