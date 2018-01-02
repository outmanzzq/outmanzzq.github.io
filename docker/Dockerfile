FROM ruby:2.3.1-alpine

MAINTAINER outmanzzq <gmoutmanzzq@gmail.com>

ENV TZ 'Asia/Shanghai'

RUN apk upgrade --no-cache && \
    apk add --no-cache bash tzdata nodejs py-pygments git gcc musl-dev make && \
    git clone https://github.com/outmanzzq/outmanzzq.github.io.git /root/outmanzzq.github.io && \
    cd /root/outmanzzq.github.io && \
    gem install --no-document bundle && \
    bundle install && \
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone && \
    apk del gcc musl-dev make && \
    rm -rf /var/cache/apk/*