FROM ubuntu:22.04

RUN set -x \
   && apt-get update && apt-get install -y --no-install-recommends \
   unbound \
   curl \
   ca-certificates \
   && apt-get clean \
   && rm -r /var/lib/apt/lists/*

RUN curl -o /etc/unbound/root.hints https://www.internic.net/domain/named.cache
RUN chown -R unbound:unbound /etc/unbound

# unbound-anchor: DNSSEC検証のためのルートトラストアンカーの設置と更新
RUN unbound-anchor -4 -a "/var/lib/unbound/root.key" ; ls -al /var/lib/unbound/root.key

# unbound-checkconf: unboundの設定のエラーを検査
RUN unbound-checkconf
