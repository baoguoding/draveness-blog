FROM ruby:2.4-alpine
MAINTAINER draveness <i@draveness.me>

RUN mkdir -p /tmp
WORKDIR /tmp
COPY Gemfile Gemfile
COPY Gemfile.lock Gemfile.lock
RUN bundle install

RUN apk update && apk add --update nodejs nodejs-npm

# install gitbook versions
RUN npm install gitbook-cli -g

ENV APP_DIR=/app

WORKDIR $APP_DIR
ADD book.json book.json
RUN gitbook install

ADD . $APP_DIR

RUN mkdir -p "/var/www/gitbook"
VOLUME ["/var/www/gitbook"]

CMD ["gitbook", "build", ".", "/var/www/gitbook"]
