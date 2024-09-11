FROM ruby:3.3.5

ARG RENDER_EXTERNAL_HOSTNAME
ARG CUSTOM_DOMAIN
ARG DISCOURSE_WORKER_CONCURRENCY

ENV DISCOURSE_HOSTNAME=${CUSTOM_DOMAIN:-$RENDER_EXTERNAL_HOSTNAME}
ENV RAILS_ENV=production
ENV DISCOURSE_VERSION=v3.3.1
ENV DISCOURSE_ROOT=/var/www/discourse
ENV DISCOURSE_WORKER_CONCURRENCY=${DISCOURSE_WORKER_CONCURRENCY:-2}

WORKDIR $DISCOURSE_ROOT

RUN echo "DISCOURSE_HOSTNAME is: $DISCOURSE_HOSTNAME"

RUN git clone https://github.com/discourse/discourse.git --branch $DISCOURSE_VERSION --single-branch $DISCOURSE_ROOT

RUN apt-get update && apt-get install -y \
    brotli \
    libbrotli-dev \ 
    gifsicle \
    jhead \
    jpegoptim \
    postgresql-client \
    vim

# Install Node.js 20.x
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
RUN apt-get install -y nodejs

RUN npm install -g svgo

# Install project dependencies
RUN bundle config set path vendor/bundle/ && bundle install
RUN gem install foreman

# Install esbuild
RUN npm install -g esbuild

# Configure NGINX
COPY install-nginx.sh /tmp/install-nginx.sh
RUN ldconfig && \
    /tmp/install-nginx.sh && \
    cp config/nginx.sample.conf /etc/nginx/conf.d/discourse.conf && \
    sed -i "s/^  server_name enter\\.your\\.web\\.hostname\\.here;/  server_name $DISCOURSE_HOSTNAME;/" /etc/nginx/conf.d/discourse.conf && \
    mkdir -p /var/nginx/cache/

# Install Redis
COPY install-redis.sh /tmp/install-redis.sh
COPY redis.conf .
RUN /tmp/install-redis.sh

# Render offers free, fully managed SSL, so set default web setting to force HTTPS
RUN perl -i -p0e 's/  force_https:\n    default: false/  force_https:\n    default: true/' config/site_settings.yml

# Ensure socket, pid, and log files exist
RUN mkdir -p tmp/sockets/ tmp/pids/
RUN touch log/production.log

COPY puma.rb sidekiq.yml ./config/
COPY entrypoint.sh Procfile ./

RUN sed -i "s/^  :concurrency: 2/  :concurrency: ${DISCOURSE_WORKER_CONCURRENCY}/" config/sidekiq.yml

ENTRYPOINT [ "bash", "entrypoint.sh" ]
