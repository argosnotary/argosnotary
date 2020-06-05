#
# Copyright (C) 2019 - 2020 Rabobank Nederland
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#         http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

ARG ARGOS_VERSION

FROM argosnotary/argos-frontend:${ARGOS_VERSION} as frontend

FROM argosnotary/argos-service:${ARGOS_VERSION} as argos-service

FROM argosnotary/argos-integration-test-stub:master as oauth-stub

FROM phusion/baseimage:0.11 as run-server-stage

RUN mkdir /usr/lib/argos

######################################################################
# Begin install mongo
######################################################################

# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
RUN addgroup --system nginx \
    && adduser --system --disabled-login --ingroup nginx --no-create-home --home /nonexistent --gecos "nginx user" --shell /bin/false nginx \
    && groupadd -r mongodb && useradd -r -g mongodb mongodb \
    && groupadd -r argos && useradd -r -g argos argos \
    && groupadd -r oauth && useradd -r -g oauth oauth

RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		ca-certificates \
		jq \
		numactl \
		gettext \
		wget \
	; \
	if ! command -v ps > /dev/null; then \
		apt-get install -y --no-install-recommends procps; \
	fi; \
	rm -rf /var/lib/apt/lists/*

# grab gosu for easy step-down from root (https://github.com/tianon/gosu/releases)
ENV GOSU_VERSION 1.11
# grab "js-yaml" for parsing mongod's YAML config files (https://github.com/nodeca/js-yaml/releases)
ENV JSYAML_VERSION 3.13.0

RUN set -ex; \
	\
	savedAptMark="$(apt-mark showmanual)"; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		wget \
	; \
	if ! command -v gpg > /dev/null; then \
		apt-get install -y --no-install-recommends gnupg dirmngr; \
		savedAptMark="$savedAptMark gnupg dirmngr"; \
	elif gpg --version | grep -q '^gpg (GnuPG) 1\.'; then \
# "This package provides support for HKPS keyservers." (GnuPG 1.x only)
		apt-get install -y --no-install-recommends gnupg-curl; \
	fi; \
	rm -rf /var/lib/apt/lists/*; \
	\
	dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')"; \
	wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch"; \
	wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch.asc"; \
	export GNUPGHOME="$(mktemp -d)"; \
	gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4; \
	gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu; \
	command -v gpgconf && gpgconf --kill all || :; \
	rm -r "$GNUPGHOME" /usr/local/bin/gosu.asc; \
	chmod +x /usr/local/bin/gosu; \
	gosu --version; \
	gosu nobody true; \
	\
	wget -O /js-yaml.js "https://github.com/nodeca/js-yaml/raw/${JSYAML_VERSION}/dist/js-yaml.js"; \
# TODO some sort of download verification here
	\
	apt-mark auto '.*' > /dev/null; \
	apt-mark manual $savedAptMark > /dev/null; \
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false

RUN mkdir /docker-entrypoint-initdb.d

ENV GPG_KEYS E162F504A20CDF15827F718D4B7C549A058F8B6B
RUN set -ex; \
	export GNUPGHOME="$(mktemp -d)"; \
	for key in $GPG_KEYS; do \
		gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
	done; \
	gpg --batch --export $GPG_KEYS > /etc/apt/trusted.gpg.d/mongodb.gpg; \
	command -v gpgconf && gpgconf --kill all || :; \
	rm -r "$GNUPGHOME"; \
	apt-key list

# Allow build-time overrides (eg. to build image with MongoDB Enterprise version)
# Options for MONGO_PACKAGE: mongodb-org OR mongodb-enterprise
# Options for MONGO_REPO: repo.mongodb.org OR repo.mongodb.com
# Example: docker build --build-arg MONGO_PACKAGE=mongodb-enterprise --build-arg MONGO_REPO=repo.mongodb.com .
ARG MONGO_PACKAGE=mongodb-org
ARG MONGO_REPO=repo.mongodb.org
ENV MONGO_PACKAGE=${MONGO_PACKAGE} MONGO_REPO=${MONGO_REPO}

ENV MONGO_MAJOR 4.2
ENV MONGO_VERSION 4.2.3
# bashbrew-architectures:amd64 arm64v8 s390x
RUN echo "deb http://$MONGO_REPO/apt/ubuntu bionic/${MONGO_PACKAGE%-unstable}/$MONGO_MAJOR multiverse" | tee "/etc/apt/sources.list.d/${MONGO_PACKAGE%-unstable}.list"

RUN set -x \
# installing "mongodb-enterprise" pulls in "tzdata" which prompts for input
	&& export DEBIAN_FRONTEND=noninteractive \
	&& apt-get update \
	&& apt-get install -y \
		${MONGO_PACKAGE}=$MONGO_VERSION \
		${MONGO_PACKAGE}-server=$MONGO_VERSION \
		${MONGO_PACKAGE}-shell=$MONGO_VERSION \
		${MONGO_PACKAGE}-mongos=$MONGO_VERSION \
		${MONGO_PACKAGE}-tools=$MONGO_VERSION \
	&& rm -rf /var/lib/apt/lists/* \
	&& rm -rf /var/lib/mongodb \
	&& mv /etc/mongod.conf /etc/mongod.conf.orig

RUN mkdir -p /data/db /data/configdb \
	&& chown -R mongodb:mongodb /data/db /data/configdb
	
VOLUME /data/db /data/configdb

EXPOSE 27017
######################################################################
# End install mongo
######################################################################

######################################################################
# Begin install java
######################################################################

ENV JAVA_VERSION   11.0.6
#ENV JAVA_UBUNTU_VERSION   8u191-b12-2ubuntu0.16.04.1

# see https://bugs.debian.org/775775
# and https://github.com/docker-library/java/issues/19#issuecomment-70546872
ENV CA_CERTIFICATES_JAVA_VERSION 20160321ubuntu1

######################################################################
# End install java
######################################################################

######################################################################
# Begin install oauth stub
######################################################################

ENV WIREMOCK_VERSION 2.26.3

# grab wiremock standalone jar
RUN mkdir -p /var/wiremock/lib/ /usr/lib/argos/oauth/mappings \
  && wget https://repo1.maven.org/maven2/com/github/tomakehurst/wiremock-standalone/$WIREMOCK_VERSION/wiremock-standalone-$WIREMOCK_VERSION.jar \
    -O /var/wiremock/lib/wiremock-standalone.jar

COPY --from=oauth-stub /home/wiremock/mappings /usr/lib/argos/oauth/mappings
COPY --from=oauth-stub /home/wiremock/__files /usr/lib/argos/oauth/__files

######################################################################
# End install oauth stub
######################################################################

RUN set -x \
	&& apt-get update \
	&& apt-get install -y \
		openjdk-11-jre-headless \
		nginx

COPY --from=frontend /usr/share/nginx/html/ /usr/share/nginx/html
COPY --from=frontend /etc/nginx/nginx.conf /etc/nginx/nginx.conf
COPY --from=frontend /image_config/nginx.conf.template /

COPY --from=argos-service /argos-service.jar /usr/lib/argos/argos-service.jar

COPY resources /tmp/resources

ENV BACKEND_URL=http://localhost:8080
      
ENV spring.data.mongodb.uri                                       mongodb://localhost/test
ENV spring.security.oauth2.client.registration.azure.clientId     client-id
ENV spring.security.oauth2.client.registration.azure.clientSecret client-secret
ENV spring.security.oauth2.client.provider.azure.authorizationUri http://localhost:8087/oauth2/v2.0/authorize-manual
ENV spring.security.oauth2.client.provider.azure.tokenUri         http://localhost:8087/oauth2/v2.0/token
ENV spring.security.oauth2.client.provider.azure.userInfoUri      http://localhost:8087/v1.0/me
ENV jwt.token.secret                                              Z2LcVpgCuGtpb5rBDNIIWGe8RXqYGWDpWFjWhYQUS2Z8rjE2/GTJlu0s6BRYG0Mn0kV8StC9cFE83Jkl68RCkw==
ENV auth.frontendRedirectBasePath                                 http://localhost
ENV server.use-forward-headers                                    'true'

# Add scripts for starting runit services
RUN mkdir /etc/service/argos-service
ADD resources/service/argos-service/run.sh /etc/service/argos-service/run
RUN chmod +x /etc/service/argos-service/run

RUN mkdir /etc/service/argos-frontend
ADD resources/service/argos-frontend/run.sh /etc/service/argos-frontend/run
RUN chmod +x /etc/service/argos-frontend/run

RUN mkdir /etc/service/mongo
ADD resources/service/mongo/run.sh /etc/service/mongo/run
RUN chmod +x /etc/service/mongo/run

RUN mkdir /etc/service/oauth
ADD resources/service/oauth/run.sh /etc/service/oauth/run
RUN chmod +x /etc/service/oauth/run

COPY mongo_entry.sh /usr/lib/argos/

RUN chown -R nginx:nginx /var/log/nginx /var/lib/nginx \
    && chmod -R 755 /usr/lib/argos
    
RUN ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log

EXPOSE 80 8080 8087
