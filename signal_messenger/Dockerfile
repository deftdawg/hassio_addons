# Build signal-cli-rest-api from deftdawg fork source
# This fork adds /api/v1/rpc, /api/v1/events, /api/v1/check endpoints
# for signal-cli daemon HTTP API compatibility (required for openclaw integration)

ARG SIGNAL_CLI_REST_API_REPO=https://github.com/deftdawg/signal-cli-rest-api.git
ARG SIGNAL_CLI_REST_API_BRANCH=master

ARG SIGNAL_CLI_VERSION=0.13.24
ARG LIBSIGNAL_CLIENT_VERSION=0.87.0
ARG SIGNAL_CLI_NATIVE_PACKAGE_VERSION=0.13.24+morph027+1

ARG SWAG_VERSION=1.16.4
ARG GRAALVM_VERSION=25.0.2

ARG BUILD_VERSION_ARG=unset

FROM golang:1.24-bookworm AS buildcontainer

ARG SIGNAL_CLI_REST_API_REPO
ARG SIGNAL_CLI_REST_API_BRANCH
ARG SIGNAL_CLI_VERSION
ARG LIBSIGNAL_CLIENT_VERSION
ARG SWAG_VERSION
ARG GRAALVM_VERSION
ARG BUILD_VERSION_ARG
ARG SIGNAL_CLI_NATIVE_PACKAGE_VERSION

# Clone deftdawg fork of signal-cli-rest-api
RUN git clone --depth 1 -b ${SIGNAL_CLI_REST_API_BRANCH} ${SIGNAL_CLI_REST_API_REPO} /tmp/signal-cli-rest-api

COPY --from=scratch /tmp/signal-cli-rest-api/ext/libraries/libsignal-client/v${LIBSIGNAL_CLIENT_VERSION} /tmp/libsignal-client-libraries
COPY --from=scratch /tmp/signal-cli-rest-api/ext/libraries/libsignal-client/signal-cli-native.patch /tmp/signal-cli-native.patch

# use architecture specific libsignal_jni.so
RUN arch="$(uname -m)"; \
        case "$arch" in \
            aarch64) cp /tmp/signal-cli-rest-api/ext/libraries/libsignal-client/v${LIBSIGNAL_CLIENT_VERSION}/arm64/libsignal_jni.so /tmp/libsignal_jni.so ;; \
            armv7l) cp /tmp/signal-cli-rest-api/ext/libraries/libsignal-client/v${LIBSIGNAL_CLIENT_VERSION}/armv7/libsignal_jni.so /tmp/libsignal_jni.so ;; \
            x86_64) cp /tmp/signal-cli-rest-api/ext/libraries/libsignal-client/v${LIBSIGNAL_CLIENT_VERSION}/x86-64/libsignal_jni.so /tmp/libsignal_jni.so ;; \
            *) echo "Unknown architecture" && exit 1 ;; \
        esac;

RUN dpkg-reconfigure debconf --frontend=noninteractive \
	&& apt-get update \
	&& apt-get -y install --no-install-recommends \
		wget software-properties-common git locales zip unzip \
		file build-essential libz-dev zlib1g-dev \
	&& rm -rf /var/lib/apt/lists/* 

RUN sed -i -e 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen && \
    dpkg-reconfigure --frontend=noninteractive locales && \
    update-locale LANG=en_US.UTF-8

ENV JAVA_OPTS="-Djdk.lang.Process.launchMechanism=vfork"

ENV LANG en_US.UTF-8

RUN go install github.com/swaggo/swag/cmd/swag@v${SWAG_VERSION}

RUN cd /tmp/ \
	&& wget -nv https://github.com/AsamK/signal-cli/releases/download/v${SIGNAL_CLI_VERSION}/signal-cli-${SIGNAL_CLI_VERSION}.tar.gz -O /tmp/signal-cli.tar.gz \
	&& tar xf signal-cli.tar.gz

# build native image with graalvm

RUN arch="$(uname -m)"; \
        case "$arch" in \
            aarch64) wget -nv https://github.com/graalvm/graalvm-ce-builds/releases/download/jdk-${GRAALVM_VERSION}/graalvm-community-jdk-${GRAALVM_VERSION}_linux-aarch64_bin.tar.gz -O /tmp/gvm.tar.gz ;; \
            armv7l) echo "GRAALVM doesn't support 32bit" ;; \
            x86_64) wget -nv https://github.com/graalvm/graalvm-ce-builds/releases/download/jdk-${GRAALVM_VERSION}/graalvm-community-jdk-${GRAALVM_VERSION}_linux-x64_bin.tar.gz -O /tmp/gvm.tar.gz ;; \
			*) echo "Invalid architecture" ;; \
        esac;

RUN if [ "$(uname -m)" = "x86_64" ]; then \
		cd /tmp \
		&& git clone https://github.com/AsamK/signal-cli.git signal-cli-${SIGNAL_CLI_VERSION}-source \
		&& cd signal-cli-${SIGNAL_CLI_VERSION}-source \
		&& git checkout -q v${SIGNAL_CLI_VERSION} \
		&& cd /tmp && mkdir -p /tmp/graalvm && tar xf gvm.tar.gz -C /tmp/graalvm --strip-components=1 \
		&& export GRAALVM_HOME=/tmp/graalvm \
		&& export PATH=/tmp/graalvm/bin:$PATH \
		&& cd /tmp/signal-cli-${SIGNAL_CLI_VERSION}-source \
		&& sed -i 's/Signal-Android\/5.22.3/Signal-Android\/5.51.7/g' src/main/java/org/asamk/signal/BaseConfig.java \
		&& ./gradlew build \
		&& ./gradlew installDist \
		&& ls build/install/signal-cli/lib/libsignal-client-${LIBSIGNAL_CLIENT_VERSION}.jar || (echo "\n\nsignal-client jar file with version ${LIBSIGNAL_CLIENT_VERSION} not found. Maybe the version needs to be bumped in the signal-cli-rest-api Dockerfile?\n\n" && echo "Available version: \n" && ls build/install/signal-cli/lib/libsignal-client-* && echo "\n\n" && exit 1) \
		&& cd /tmp \
		&& cp signal-cli-${SIGNAL_CLI_VERSION}-source/build/install/signal-cli/lib/libsignal-client-${LIBSIGNAL_CLIENT_VERSION}.jar libsignal-client.jar \
		&& zip -qu libsignal-client.jar libsignal_jni.so \
		&& cd /tmp/signal-cli-${SIGNAL_CLI_VERSION}-source \
		&& git apply /tmp/signal-cli-rest-api/ext/libraries/libsignal-client/signal-cli-native.patch \
		&& ./gradlew -q nativeCompile; \
	elif [ "$(uname -m)" = "aarch64" ] ; then \
		echo "Use native image from @morph027 (https://packaging.gitlab.io/signal-cli/) for arm64 - many thanks to @morph027" \
		&& curl -fsSL https://packaging.gitlab.io/signal-cli/gpg.key | apt-key add - \
		&& echo "deb https://packaging.gitlab.io/signal-cli focal main" > /etc/apt/sources.list.d/morph027-signal-cli.list \
		&& mkdir -p /tmp/signal-cli-native \
		&& cd /tmp/signal-cli-native \
		&& apt-get update \
		&& apt-get download signal-cli-native=${SIGNAL_CLI_NATIVE_PACKAGE_VERSION} \
		&& ar x *.deb \
		&& tar xf data.tar.gz \
		&& mkdir -p /tmp/signal-cli-${SIGNAL_CLI_VERSION}-source/build/native/nativeCompile \
		&& cp /tmp/signal-cli-native/usr/bin/signal-cli-native  /tmp/signal-cli-${SIGNAL_CLI_VERSION}-source/build/native/nativeCompile/signal-cli; \
    elif [ "$(uname -m)" = "armv7l" ] ; then \
		echo "GRAALVM doesn't support 32bit" \
		&& echo "Creating temporary file, otherwise the below copy doesn't work for armv7" \
		&& mkdir -p /tmp/signal-cli-${SIGNAL_CLI_VERSION}-source/build/native/nativeCompile \
		&& touch /tmp/signal-cli-${SIGNAL_CLI_VERSION}-source/build/native/nativeCompile/signal-cli; \
    else \
		echo "Unknown architecture"; \
    fi;

# replace libsignal-client

RUN ls /tmp/signal-cli-${SIGNAL_CLI_VERSION}/lib/libsignal-client-${LIBSIGNAL_CLIENT_VERSION}.jar || (echo "\n\nsignal-client jar file with version ${LIBSIGNAL_CLIENT_VERSION} not found. Maybe the version needs to be bumped in the signal-cli-rest-api Dockerfile?\n\n" && echo "Available version: \n" && ls /tmp/signal-cli-${SIGNAL_CLI_VERSION}/lib/libsignal-client-* && echo "\n\n" && exit 1)

# workaround until upstream is fixed
RUN cd /tmp/signal-cli-${SIGNAL_CLI_VERSION}/lib \
	&& unzip signal-cli-${SIGNAL_CLI_VERSION}.jar \
	&& sed -i 's/Signal-Android\/5.22.3/Signal-Android\/5.51.7/g' org/asamk/signal/BaseConfig.class \
	&& zip -r signal-cli-${SIGNAL_CLI_VERSION}.jar org/ META-INF/ \
	&& rm -rf META-INF \
	&& rm -rf org

RUN cd /tmp/ \
	&& zip -qu /tmp/signal-cli-${SIGNAL_CLI_VERSION}/lib/libsignal-client-${LIBSIGNAL_CLIENT_VERSION}.jar libsignal_jni.so \
	&& zip -qr signal-cli-${SIGNAL_CLI_VERSION}.zip signal-cli-${SIGNAL_CLI_VERSION}/* \
    && unzip -q /tmp/signal-cli-${SIGNAL_CLI_VERSION}.zip -d /opt \
	&& rm -f /tmp/signal-cli-${SIGNAL_CLI_VERSION}.zip

# build signal-cli-rest-api from deftdawg fork
RUN ls -la /tmp/signal-cli-rest-api/src
RUN cd /tmp/signal-cli-rest-api/src && ${GOPATH}/bin/swag init
RUN cd /tmp/signal-cli-rest-api/src && go build -o signal-cli-rest-api main.go
RUN cd /tmp/signal-cli-rest-api/src && go test ./client -v && go test ./utils -v

# build supervisorctl_config_creator
RUN cd /tmp/signal-cli-rest-api/src/scripts && go build -o jsonrpc2-helper 

# build plugin_loader
RUN cd /tmp/signal-cli-rest-api/src && go build -buildmode=plugin -o signal-cli-rest-api_plugin_loader.so plugin_loader.go

# Start a fresh container for release container
FROM ubuntu:jammy

ENV GIN_MODE=release

ENV PORT=8080

ARG SIGNAL_CLI_VERSION
ARG BUILD_VERSION_ARG

ENV BUILD_VERSION=$BUILD_VERSION_ARG
ENV SIGNAL_CLI_REST_API_PLUGIN_SHARED_OBJ_DIR=/usr/bin/

RUN dpkg-reconfigure debconf --frontend=noninteractive \
	&& apt-get update \
	&& apt-get install -y --no-install-recommends util-linux supervisor netcat openjdk-21-jre curl locales jq \
	&& rm -rf /var/lib/apt/lists/* 

COPY --from=buildcontainer /tmp/signal-cli-rest-api/src/signal-cli-rest-api /usr/bin/signal-cli-rest-api
COPY --from=buildcontainer /opt/signal-cli-${SIGNAL_CLI_VERSION} /opt/signal-cli-${SIGNAL_CLI_VERSION}
COPY --from=buildcontainer /tmp/signal-cli-${SIGNAL_CLI_VERSION}-source/build/native/nativeCompile/signal-cli /opt/signal-cli-${SIGNAL_CLI_VERSION}/bin/signal-cli-native
COPY --from=buildcontainer /tmp/signal-cli-rest-api/src/scripts/jsonrpc2-helper /usr/bin/jsonrpc2-helper
COPY --from=buildcontainer /tmp/signal-cli-rest-api/src/signal-cli-rest-api_plugin_loader.so /usr/bin/signal-cli-rest-api_plugin_loader.so
COPY --from=buildcontainer /tmp/signal-cli-rest-api/entrypoint.sh /entrypoint.sh

COPY options.sh /options.sh
RUN chmod +x /options.sh

RUN groupadd -g 1000 signal-api \
	&& useradd --no-log-init -M -d /home -s /bin/bash -u 1000 -g 1000 signal-api \
	&& ln -s /opt/signal-cli-${SIGNAL_CLI_VERSION}/bin/signal-cli /usr/bin/signal-cli \
	&& ln -s /opt/signal-cli-${SIGNAL_CLI_VERSION}/bin/signal-cli-native /usr/bin/signal-cli-native \
	&& mkdir -p /signal-cli-config/ \
	&& mkdir -p /home/.local/share/signal-cli

# remove the temporary created signal-cli-native on armv7, as GRAALVM doesn't support 32bit
RUN arch="$(uname -m)"; \
        case "$arch" in \
            armv7l) echo "GRAALVM doesn't support 32bit" && rm /opt/signal-cli-${SIGNAL_CLI_VERSION}/bin/signal-cli-native /usr/bin/signal-cli-native  ;; \
        esac;

RUN sed -i -e 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen && \
    dpkg-reconfigure --frontend=noninteractive locales && \
    update-locale LANG=en_US.UTF-8

ENV LANG en_US.UTF-8

EXPOSE ${PORT}

ENV SIGNAL_CLI_CONFIG_DIR=/home/.local/share/signal-cli
ENV SIGNAL_CLI_UID=1000
ENV SIGNAL_CLI_GID=1000
ENV SIGNAL_CLI_CHOWN_ON_STARTUP=true

VOLUME ["/config"]

ENTRYPOINT ["/options.sh"]

# Build arguments for Home Assistant addon
ARG BUILD_ARCH
ARG BUILD_DATE
ARG BUILD_REF
ARG BUILD_VERSION

# Labels
LABEL \
    io.hass.name="Signal Messenger" \
    io.hass.description="Signal Messenger (deftdawg fork with openclaw support)" \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.version="${BUILD_VERSION}" \
    io.hass.type="addon" \
    maintainer="deftdawg" \
    org.label-schema.description="Signal Messenger" \
    org.label-schema.build-date="${BUILD_DATE}" \
    org.label-schema.name="Signal Messenger" \
    org.label-schema.schema-version="${BUILD_VERSION}" \
    org.label-schema.url="https://github.com/deftdawg/signal-addon" \
    org.label-schema.usage="https://github.com/deftdawg/signal-addon" \
    org.label-schema.vcs-ref="${BUILD_REF}" \
    org.label-schema.vcs-url="https://github.com/deftdawg/" \
    org.label-schema.vendor="deftdawg"

HEALTHCHECK --interval=20s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:${PORT}/v1/health || exit 1
