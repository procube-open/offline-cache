FROM nginx:latest

RUN openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes \
  -keyout /etc/ssl/private/domain.key \
  -out /etc/ssl/certs/signed.crt \
  -subj "/CN=registry-1.docker.io" \
  -addext "subjectAltName=DNS:registry-1.docker.io" \
  -addext "extendedKeyUsage=serverAuth"

COPY default.conf /etc/nginx/conf.d/default.conf
