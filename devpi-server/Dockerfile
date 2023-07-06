FROM python:3.11-alpine
RUN apk add supervisor gcc musl-dev
RUN pip install devpi-server
RUN mkdir /var/lib/devpi-server
WORKDIR /var/lib/devpi-server
COPY config.yml ./
RUN devpi-init -c config.yml
RUN devpi-gen-config -c config.yml
CMD [ "supervisord", "-n", "-c", "gen-config/supervisord.conf" ]
