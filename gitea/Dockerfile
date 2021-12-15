FROM gitea/gitea:1

RUN apk update && \
    apk add --no-cache tzdata ca-certificates

COPY root_ca.crt /usr/local/share/ca-certificates/
RUN update-ca-certificates