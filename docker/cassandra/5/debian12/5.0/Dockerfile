

FROM golang:1.23.4-bookworm AS gosu-builder

RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		arch-test \
		file \
		patch \
		git \
	; \
	rm -rf /var/lib/apt/lists/*

RUN mkdir -p /usr/src/ \
		&& git clone https://github.com/tianon/gosu.git --branch "1.17" /usr/src/gosu

# note: we cannot add "-s" here because then "govulncheck" does not work (see SECURITY.md); the ~0.2MiB increase (as of 2022-12-16, Go 1.18) is worth it
ENV BUILD_FLAGS="-v -ldflags '-d -w'"

RUN set -eux; \
	{ \
		echo '#!/usr/bin/env bash'; \
		echo 'set -Eeuo pipefail -x'; \
		echo 'eval "go build $BUILD_FLAGS -o /go/bin/gosu-$ARCH"'; \
		echo 'file "/go/bin/gosu-$ARCH"'; \
		echo 'if arch-test "$ARCH"; then'; \
# there's a fun QEMU + Go 1.18+ bug that causes our binaries (especially on ARM arches) to hang indefinitely *sometimes*, hence the "timeout" and looping here
		echo '  try() { for (( i = 0; i < 30; i++ )); do if timeout 1s "$@"; then return 0; fi; done; return 1; }'; \
		echo '  try "/go/bin/gosu-$ARCH" --version'; \
		echo '  try "/go/bin/gosu-$ARCH" nobody id'; \
		echo '  try "/go/bin/gosu-$ARCH" nobody ls -l /proc/self/fd'; \
		echo 'fi'; \
	} > /usr/local/bin/gosu-build-and-test.sh; \
	chmod +x /usr/local/bin/gosu-build-and-test.sh

# disable CGO for ALL THE THINGS (to help ensure no libc)
ENV CGO_ENABLED 0

WORKDIR /go/src/github.com/tianon/gosu
RUN cp /usr/src/gosu/go.mod /usr/src/gosu/go.sum ./
RUN set -eux; \
	go mod download; \
	go mod verify

RUN cp /usr/src/gosu/*.go ./

RUN ARCH=amd64    GOARCH=amd64       gosu-build-and-test.sh

RUN set -eux; ls -lAFh /go/bin/gosu-*; file /go/bin/gosu-*


FROM marketplace.gcr.io/google/debian12

# explicitly set user/group IDs
RUN groupadd -r cassandra --gid=999 && useradd -r -g cassandra --uid=999 cassandra

RUN set -ex; \
	apt-get update; \
	if ! command -v gpg > /dev/null; then \
		apt-get install -y --no-install-recommends \
			gnupg \
			dirmngr \
		; \
	fi ; \
	if ! command -v free > /dev/null; then \
		apt-get install -y --no-install-recommends \
			procps \
		; \
	fi ; \
	rm -rf /var/lib/apt/lists/*;

COPY --from=gosu-builder /go/bin/gosu-amd64 /usr/local/bin/gosu
RUN chmod +x /usr/local/bin/gosu \
			&& gosu nobody true

RUN apt-get update \
	&& apt-get install -y --no-install-recommends \
		ca-certificates \
		# solves warning: "jemalloc shared library could not be preloaded to speed up memory allocations"
		libjemalloc2 \
# "free" is used by cassandra-env.sh
		procps \
# "cqlsh" needs a python interpreter
		python3 \
# "ip" is not required by Cassandra itself, but is commonly used in scripting Cassandra's configuration (since it is so fixated on explicit IP addresses)
		iproute2 \
# Cassandra will automatically use numactl if available
#   https://github.com/apache/cassandra/blob/18bcda2d4c2eba7370a0b21f33eed37cb730bbb3/bin/cassandra#L90-L100
#   https://github.com/apache/cassandra/commit/604c0e87dc67fa65f6904ef9a98a029c9f2f865a
		numactl \
		wget \
	&& rm -rf /var/lib/apt/lists/*

# https://wiki.apache.org/cassandra/DebianPackaging#Adding_Repository_Keys
ENV GPG_KEYS 514A2AD631A57A16DD0047EC749D6EEC0353B12C A26E528B271F19B9E5D8E19EA278B781FE4B2BDA A4C465FEA0C552561A392A61E91335D77E3E87CB

ENV CASSANDRA_VERSION 5.0.2
ENV CASSANDRA_HOME /opt/cassandra
ENV CASSANDRA_CONFIG /etc/cassandra
ENV PATH $CASSANDRA_HOME/bin:$PATH

ENV C2D_RELEASE 5.0.2

RUN set -ex; \
	export DIST_FILE="https://archive.apache.org/dist/cassandra/5.0.2/apache-cassandra-5.0.2-bin.tar.gz"; \
	wget --progress=dot:giga -O "cassandra-bin.tgz" "${DIST_FILE}"

RUN mkdir -p "$CASSANDRA_HOME"; \
	tar --extract --file cassandra-bin.tgz --directory "$CASSANDRA_HOME" --strip-components 1; \
	rm cassandra-bin.tgz*; \
	\
	[ ! -e "$CASSANDRA_CONFIG" ]; \
	mv "$CASSANDRA_HOME/conf" "$CASSANDRA_CONFIG"; \
	ln -sT "$CASSANDRA_CONFIG" "$CASSANDRA_HOME/conf"

ENV JMX_EXPORTER_VERSION 1.0.1
ENV JMX_EXPORTER_PATH /opt/jmx-exporter
ENV JMX_EXPORTER_JAR jmx_prometheus_javaagent-${JMX_EXPORTER_VERSION}.jar
ENV JMX_EXPORTER_AGENT ${JMX_EXPORTER_PATH}/${JMX_EXPORTER_JAR}
ENV JMX_EXPORTER_CONFIG ${JMX_EXPORTER_PATH}/cassandra.yml
ENV JMX_EXPORTER_LICENSE ${JMX_EXPORTER_PATH}/LICENSE
ENV JMX_EXPORTER_NOTICE ${JMX_EXPORTER_PATH}/NOTICE


RUN apt-get update \
	&& apt-get install -y --no-install-recommends curl git ca-certificates wget \
	&& mkdir -p $JMX_EXPORTER_PATH \
	&& wget -O $JMX_EXPORTER_AGENT https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/$JMX_EXPORTER_VERSION/$JMX_EXPORTER_JAR \
	&& wget -O $JMX_EXPORTER_CONFIG https://raw.githubusercontent.com/prometheus/jmx_exporter/main/examples/cassandra.yml \
	&& wget -O $JMX_EXPORTER_LICENSE https://raw.githubusercontent.com/prometheus/jmx_exporter/main/LICENSE \
	&& wget -O $JMX_EXPORTER_NOTICE https://raw.githubusercontent.com/prometheus/jmx_exporter/main/NOTICE \
	&& apt-get purge -y --auto-remove ca-certificates wget \
	&& rm -rf /var/lib/apt/lists/*

RUN set -x; \
		echo "deb http://deb.debian.org/debian unstable main non-free non-free-firmware contrib" > /etc/apt/sources.list.d/debian-unstable.list; \
		apt-get update \
		&& apt-get -y install openjdk-11-jdk

COPY docker-entrypoint.sh /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]

RUN mkdir -p /var/lib/cassandra "$CASSANDRA_CONFIG" /var/log/cassandra \
	&& chown -R cassandra:cassandra /var/lib/cassandra "$CASSANDRA_CONFIG" "$JMX_EXPORTER_PATH" /var/log/cassandra \
	&& chmod 777 /var/lib/cassandra "$CASSANDRA_CONFIG" "$JMX_EXPORTER_PATH" /var/log/cassandra \
	&& ln -sT /var/lib/cassandra "$CASSANDRA_HOME/data" \
	&& ln -sT /var/log/cassandra "$CASSANDRA_HOME/logs" \
	&& chown cassandra:cassandra /docker-entrypoint.sh

VOLUME /var/lib/cassandra

RUN test ${C2D_RELEASE} = $(cassandra -v) \
        || { echo "Version check failed"; exit 1; }


# Download Licenses
COPY components.csv /components.csv
COPY download-licenses.sh /download-licenses.sh
RUN mkdir -p /usr/src && \
		/download-licenses.sh /components.csv /usr/src/licenses

# Download GPL source-code
RUN git clone https://github.com/typetools/checker-framework.git /usr/src/checker-framework \
		&& git clone https://github.com/qos-ch/logback.git /usr/src/logback \
		&& git clone https://github.com/vbmacher/cup-maven-plugin /usr/src/cup-maven-plugin

# 7000: intra-node communication
# 7001: TLS intra-node communication
# 7199: JMX
# 9042: CQL
# 9160: thrift service
# 9404: JMX Exporter for Prometheus metrics
EXPOSE 7000 7001 7199 9042 9160 9404

CMD ["cassandra", "-f"]
