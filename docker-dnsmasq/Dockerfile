FROM alpine:edge
LABEL maintainer="Bill Clark <36345117+brav0charlie@users.noreply.github.com>"
LABEL github_url="https://github.com/brav0charlie/docker-dnsmasq"

RUN apk --no-cache add dnsmasq=2.85-r1
EXPOSE 53/udp
EXPOSE 67/udp
EXPOSE 68/udp

ENTRYPOINT ["dnsmasq", "-k"]
