FROM debian:trixie

ARG PHP_VERSION=8.4
ENV PHPVER="$PHP_VERSION"

COPY mikenopa-docker-utils /tmp/mikenopa-docker-utils
COPY installer /tmp/installer
RUN /tmp/installer

COPY bin/ /usr/bin/
COPY php/ /etc/php/$PHPVER/
COPY nginx-config/ /etc/nginx/

EXPOSE 80
WORKDIR /app
CMD ["entrypoint.sh"]
