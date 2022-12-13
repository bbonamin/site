---
layout: post
title: "Ruby on Rails 5 Dockerfile: Slimming Down Further"
date: 2019-02-24 11:58:00 -0300
categories: ruby rails docker
---

In my [previous post]({{site.baseurl}}{% post_url 2019-02-17-ruby-on-rails-5-dockerfile-2019-edition %}) I attempted to generate a Docker image for a Rails application I'm working on, to be as small as possible.

This week I've been able to improve that output even more, after finding out about [Docker multistage builds](https://docs.docker.com/develop/develop-images/multistage-build/).

Using the Dockerfile described at the end of my last post, an image that weighed 1.2GB was built. Not crazy, but still a lot. Running a `shell` in the image (`docker run -ti myapp /bin/sh`) showed that the actual files were using around 500MB of disk space, so where was the rest coming from?

It turns out, that even if you delete dependencies (like compilers, etc); unless you do everything in the same command as the installation, the "layering" feature of Docker keeps a copy of that software, as an intermediate state. If you think about it, it makes sense, you might change a more recent layer (like decide to stop uninstalling certain dependencies), and Docker would still be able to re-use the cache from all previous layers.

If you use the Multistage feature, the previous layers remain as part of the "builder" images, and are not carried over to the final image.

## Using Multistage prevents sharing secret tokens

I also discovered that with my previous approach, my claim

> It is possible to provide an argument to a docker image, which can be used by bundler to authenticate with Github, and not have this token end up in the final image, by using a combination of Docker build-time variables, and Bundler support for credentials via ENV variables

was false, as you were still able to see the Github token by checking the history of the image:

{% highlight shell %}
❯ docker history myapp-prod
IMAGE CREATED CREATED BY SIZE COMMENT
24575fd555a9 5 days ago /bin/sh -c #(nop) CMD ["passenger" "start" … 0B
d54a9ded8035 5 days ago /bin/sh -c #(nop) EXPOSE 3000 0B
226768ca118a 5 days ago |1 GITHUB_ACCESS_TOKEN=ABC!123456… 176MB

# ...

{% endhighlight %}

The good news is that by using multistage builds (_and provided that only the final image is uploaded to the registry_), this "token" is not added to the final image's history, as the `bundle` phase is replaced by a `copy`.

{% highlight shell %}
❯ docker history myapp
IMAGE CREATED CREATED BY SIZE COMMENT
8341d5aa2089 2 days ago /bin/sh -c #(nop) CMD ["passenger" "start" … 0B
f85d7f3ce49d 2 days ago /bin/sh -c #(nop) EXPOSE 3000 0B
00eebda1d110 2 days ago /bin/sh -c #(nop) COPY dir:3eb6adbc9858fec2a… 65.4MB
e7c54cae075a 3 days ago /bin/sh -c #(nop) COPY dir:19e889591bcced183… 100MB
6956c2999c40 3 days ago /bin/sh -c #(nop) COPY dir:b1a0f67106288fb02… 1.17MB
bafaa2b3c7c1 3 days ago /bin/sh -c #(nop) COPY dir:a4daf71b677d4c5bc… 147MB
cf56f0a1b196 3 days ago /bin/sh -c #(nop) COPY dir:8ec1839535939889b… 158MB
9fec28c02120 3 days ago /bin/sh -c #(nop) COPY dir:1445209a55d41971e… 83.3MB

# ...

{% endhighlight %}

I still need to do decide exactly which binaries and libraries to copy from `/usr/lib` instead of the complete directory, but with this change I was able to reduce the image size to 600MB, roughly a 50% improvement.

## Dockerfile

{% highlight dockerfile %}

# ---------------------------------------------------

# Phase 1 - Webserver install

# ---------------------------------------------------

FROM ruby:2.6.1-alpine3.9 as webserver-builder

RUN mkdir -p /opt/www/myapp
WORKDIR /opt/www/myapp

ENV PASSENGER_VERSION="6.0.1"
ENV PATH="/opt/passenger/bin:$PATH"
ENV VERBOSE=1

RUN apk add --no-cache --update \
 ca-certificates \
 procps \
 curl \
 pcre \
 libstdc++ \
 libexecinfo \

build-base \
 curl-dev \
 linux-headers \
 pcre-dev \
 libexecinfo-dev

RUN mkdir -p /opt && \
 curl -L https://s3.amazonaws.com/phusion-passenger/releases/passenger-$PASSENGER_VERSION.tar.gz | tar -xzf - -C /opt && \
 mv /opt/passenger-$PASSENGER_VERSION /opt/passenger && \
 export EXTRA_PRE_CFLAGS='-O' EXTRA_PRE_CXXFLAGS='-O' EXTRA_LDFLAGS='-lexecinfo' && \
 # compile agent
passenger-config compile-agent --auto --optimize && \
 passenger-config install-standalone-runtime --auto --url-root=fake --connect-timeout=60 && \
 passenger-config build-native-support && \
 passenger-config validate-install --auto

# ---------------------------------------------------

# Phase 2 - Gems installation and assets compilation.

# ---------------------------------------------------

FROM ruby:2.6.1-alpine3.9 as builder
RUN apk add --no-cache --update \
 build-base \
 postgresql-dev \
 mariadb-dev \
 git \
 nodejs-current \
 yarn \
 tzdata \
 libxml2-dev \
 libxslt-dev \
 python

WORKDIR /app
ENV RAILS_ENV production
ENV NODE_ENV production

# Gems installation

COPY Gemfile Gemfile.lock ./
ARG GITHUB_ACCESS_TOKEN
RUN bundle config --global frozen 1 && \
 BUNDLE_GITHUB\_\_COM="$GITHUB_ACCESS_TOKEN:x-oauth-basic" bundle install --jobs 4 --without development test && \
 rm -rf /usr/local/bundle/cache/_.gem && \
 find /usr/local/bundle/gems/ -name "_.c" -delete && \
 find /usr/local/bundle/gems/ -name "\*.o" -delete

# NPM packages installation

COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile --non-interactive --production

ADD . /app

# Copy a dummy database.yml and config.yml to allow asset compilation, otherwise Rails blows up.

# Real configuration files are written by configMaps.

COPY config/database.k8s-dummy.yml config/database.yml
COPY config/config.k8s-dummy.yml config/config.yml

RUN bundle exec rake assets:clobber assets:precompile --trace && \
 yarn cache clean && \
 rm -rf node_modules tmp/cache app/assets vendor/assets spec

# ---------------------------------------------------

# Phase 3 - Final phase

# ---------------------------------------------------

FROM ruby:2.6.1-alpine3.9

RUN mkdir -p /opt/www/myapp
WORKDIR /opt/www/myapp

ENV RAILS_ENV production
ENV NODE_ENV production
ENV RAILS_SERVE_STATIC_FILES true
ENV PATH="/opt/passenger/bin:$PATH"

# Some native extensions required by the webserver

COPY --from=webserver-builder /usr/lib /usr/lib

# Passenger and its binary files.

COPY --from=webserver-builder /opt/passenger /opt/passenger

# Some native extensions required by gems such as pg or mysql2.

COPY --from=builder /usr/lib /usr/lib

# Timezone data is required at runtime

COPY --from=builder /usr/share/zoneinfo/ /usr/share/zoneinfo/

# Ruby gems

COPY --from=builder /usr/local/bundle /usr/local/bundle

COPY --from=builder /app /opt/www/myapp

EXPOSE 3000
CMD ["passenger", "start", "--no-install-runtime", "--no-compile-runtime", "-b", "0.0.0.0"]

{% endhighlight %}
