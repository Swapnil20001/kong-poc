FROM kong:3.1.1-alpine
USER root
ENV KONG_ADMIN_LISTEN=admin_listen
ENV KONG_DATABASE=kong_database
ENV KONG_PG_HOST=pg_host
ENV KONG_PG_PASSWORD=pg_password
ENV KONG_PG_USER=pg_user
ENV KONG_PG_DATABASE=pg_database
ENV KONG_TRUSTED_IPS="0.0.0.0/0,::/0"
ENV KONG_REAL_IP_RECURSIVE="on"
ENV KONG_REAL_IP_HEADER="X-Forwarded-For"
ENV KONG_PROXY_LISTEN="0.0.0.0:8443,0.0.0.0:8000"
ENV KONG_HEADERS="off"
ENV KONG_NGINX_WORKER_PROCESSES="1"
ENV KONG_SSL="off"
RUN apk update && apk add git unzip luarocks
RUN luarocks install kong-plugin-static-response
ENV KONG_PLUGINS=bundled,static-response
RUN mkdir /opt/logs
RUN chmod 777 /opt/logs/

