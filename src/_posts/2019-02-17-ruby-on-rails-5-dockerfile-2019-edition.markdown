---
layout: post
title:  "Ruby on Rails 5 Dockerfile: 2019 Edition"
date:   2019-02-17 10:56:55 -0300
categories: ruby rails docker
---
Most of other posts online about creating a production Docker image for Rails applications seem to be a little basic. Maybe the majority of developers have migrated to other platforms? I needed to dockerize a Rails application for a production deployment, which has a couple of requirements that other guides do not seem to mention:

- It needs to be as small as possible
- It needs to use a production webserver (we use Passenger, but puma would be fine too)
- It needs to compile assets using webpack, and install dependencies using yarn (as Webpacker is being used)
- It needs to be able to pull gems from private github repositories.


I'm still getting used to Docker, I'm not an expert by any means, but I hope my learnings below are useful. I still need to do some work specifically in the configuration, as we are going to be using Kubernetes (and I've read it has a tool called configMap), but for now I'm overriding files like `config.yml` and `database.yml` with versions that are checked-in.

# Small as possible
For this requirement, I wanted to use the Alpine distro, as it's pretty well known in the docker world, and even has official ruby images, which are way more lightweight than Ubuntu.

There's always the ability to improve further in this area, so I'm open for suggestions.

# Production Webserver (Passenger)
Installing passenger was tricky as it doesn't have any official repositories for Alpine, and simply letting the `passenger` rubygem on the Gemfile attempt to install it, fails.

# Webpack
This application's frontend is built using React, Redux, and other modern frontend libraries, and we use the Webpacker gem (which has made working with modern JS frontend in Rails a breeze). Therefore, the docker image needs to install all frontend dependencies and be able to compile the assets.

I started getting some early exits with the status code 137 when trying to compile assets now and then, and eventually found that this was caused by the massive about of working memory that the container requires when compiling assets (it wasn't strange to see it using 2-3GB of RAM). Increasing the amount of memory available to Docker for Desktop fixed this issue, but I would like to find a way to reduce this footprint in the future (maybe removing Sprockets completely or moving to Webpack 4 would make things better? I don't know).

# Private gems
The issue with pulling gems from a private repository is that you either need an SSH key that can read from that repo, or you need to provide credentials for access via HTTPS, as these are the two strategies that Bundler supports. I discarded the SSH key strategy as that was too complicated (needed to provide it to the image at build time, needed to install an ssh agent), and opted instead for the HTTPS version.

Doing some reading I found that it's possible to not require to provide username and password but instead an [OAUTH token ](https://developer.github.com/v3/auth/#via-oauth-tokens), which could be generated from a shared account for deployment, or any developer could generate their own to build the image locally.

Bundler supports [credentials for gem sources natively](https://bundler.io/v1.15/bundle_config.html#CREDENTIALS-FOR-GEM-SOURCES), so it's possible to run `bundle config GITHUB__COM abc123` with the generated Github token as part of the build. The problem with this approach is that you would be checking the token in the Dockerfile repository, unless you used [Docker build-time variables](https://docs.docker.com/engine/reference/commandline/build/#set-build-time-variables---build-arg), which would be an improvement, but this would _still_ persist the token inside the image, viewable by running `bundle config`.

It is possible to provide an argument to a docker image, which can be used by bundler to authenticate with Github, and not have this token end up in the final image, by using a combination of Docker build-time variables, and Bundler support for credentials via ENV variables. It looks like this:

{% highlight dockerfile %}
# ...
ARG GITHUB_ACCESS_TOKEN
RUN BUNDLE_GITHUB__COM="$GITHUB_ACCESS_TOKEN:x-oauth-basic" bundle install --without development test
# ...
{% endhighlight %}

And to build the image:
{% highlight shell %}
docker build -t myapp-prod --build-arg GITHUB_ACCESS_TOKEN=abc123 .
{% endhighlight %}

# Dockerfile
After a couple of days of tweaks to attempt to leverage Docker caching layers as much as possible, this is the end result:

{% highlight dockerfile %}
FROM ruby:2.6.1-alpine3.9

RUN mkdir -p /opt/www/myapp
WORKDIR /opt/www/myapp

RUN apk add --no-cache --update build-base \
  linux-headers \
  git \
  postgresql-dev \
  mariadb-dev \
  nodejs \
  tzdata \
  git \
  openssh \
  build-base \
  libxml2-dev \
  libxslt-dev \
  yarn \
  curl-dev

ENV PASSENGER_VERSION="6.0.1"
ENV PATH="/opt/passenger/bin:$PATH"
ENV VERBOSE=1

RUN PACKAGES="ca-certificates procps curl pcre libstdc++ libexecinfo" && \
  BUILD_PACKAGES="build-base linux-headers pcre-dev libexecinfo-dev" && \
  apk add --update $PACKAGES $BUILD_PACKAGES && \
  # download and extract
  mkdir -p /opt && \
  curl -L https://s3.amazonaws.com/phusion-passenger/releases/passenger-$PASSENGER_VERSION.tar.gz | tar -xzf - -C /opt && \
  mv /opt/passenger-$PASSENGER_VERSION /opt/passenger && \
  export EXTRA_PRE_CFLAGS='-O' EXTRA_PRE_CXXFLAGS='-O' EXTRA_LDFLAGS='-lexecinfo' && \
  # compile agent
  passenger-config compile-agent --auto --optimize && \
  passenger-config install-standalone-runtime --auto --url-root=fake --connect-timeout=60 && \
  passenger-config build-native-support

# Cleanup passenger src directory
RUN passenger-config validate-install --auto && \
  apk del $BUILD_PACKAGES

RUN rm -rf /var/cache/apk/*

ENV RAILS_ENV production
ENV RAILS_SERVE_STATIC_FILES true

ENV NODE_ENV production

# Gems installation
COPY Gemfile Gemfile.lock ./
ARG GITHUB_ACCESS_TOKEN
RUN bundle config --global frozen 1
RUN BUNDLE_GITHUB__COM="$GITHUB_ACCESS_TOKEN:x-oauth-basic" bundle install --without development test

# NPM packages installation
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile --non-interactive

COPY . /opt/www/myapp

COPY config/database.docker.yml config/database.yml

RUN bundle exec rake assets:clobber assets:precompile --trace
RUN yarn cache clean

EXPOSE 3000
CMD ["passenger", "start", "-b", "0.0.0.0"]

{% endhighlight %}
