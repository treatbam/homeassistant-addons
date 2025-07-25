# check=skip=SecretsUsedInArgOrEnv
FROM	pihole/pihole:latest AS pihole

ARG	YQ_VERSION=v4.45.4
ARG	SV_VERSION=0.4
ARG	TARGETARCH

SHELL	["/bin/bash", "-e", "-o", "pipefail", "-c"]

# Ensure that build args are given
ARG	BUILD_ARCH
ARG	VERSION
RUN	[ -z "$BUILD_ARCH" ] && echo "Error: Build argument 'BUILD_ARCH' is missing" && exit 1 || true
RUN	[ -z "$VERSION" ] && echo "Error: Build argument 'VERSION' is missing" && exit 1 || true

# Update base image packages and install additional software
RUN <<EOF
	apk upgrade --no-cache
	apk add     --no-cache dumb-init nginx nginx-mod-http-lua patch
EOF

# Install yq
RUN <<EOF
	curl -sSLf -o /usr/bin/yq "https://github.com/mikefarah/yq/releases/download/$YQ_VERSION/yq_linux_$TARGETARCH"
	chmod +x /usr/bin/yq

	# Test binary
	yq --version
EOF

# Install supervisor.sh
RUN <<EOF
	curl -sSLf -o /usr/bin/supervisor.sh               "https://raw.githubusercontent.com/casperklein/supervisor.sh/refs/tags/$SV_VERSION/supervisor.sh"
	curl -sSLf -o /etc/bash_completion.d/supervisor.sh "https://raw.githubusercontent.com/casperklein/supervisor.sh/refs/tags/$SV_VERSION/supervisor-completion.bash"
	chmod +x /usr/bin/supervisor.sh
EOF

# Copy root filesystem
COPY	rootfs /

# Remove docker-pi-hole update instructions
RUN	sedfile -i 's|To install updates, <a href="https://github.com/pi-hole/docker-pi-hole#upgrading-persistence-and-customizations" rel="noopener noreferrer" target="_blank">replace this old container with a fresh upgraded image</a>.||' /var/www/html/admin/scripts/js/footer.js
# Disable update checks completely
# RUN	sedfile -i 's|updateComponentAvailable = true;|updateComponentAvailable = false;|g' /var/www/html/admin/scripts/js/footer.js

# Remove obsolete files
RUN	rm /etc/nginx/http.d/default.conf /usr/bin/sedfile

# Build dnscrypt-proxy from source #################################################################
FROM	golang:bookworm AS dnscrypt

ARG	DNSCRYPT_VERSION="2.1.12"
ENV	URL="https://github.com/dnscrypt/dnscrypt-proxy/archive/$DNSCRYPT_VERSION.tar.gz"

ENV	DEBIAN_FRONTEND="noninteractive"
RUN	apt-get update && apt-get -y upgrade

WORKDIR	/build
RUN	curl -sSLf "$URL" | tar --strip-component 1 -xz
WORKDIR	/build/dnscrypt-proxy

# Compile with static linking (including all library code in the binary); needed for alpine image
# Otherwise --> /dnscrypt-proxy: cannot execute: required file not found
ENV	CGO_ENABLED=0

# Builds a Go binary without symbol tables and debugging information, reducing the file size.
RUN	go build -ldflags="-s -w" -mod vendor

# Test binary
RUN	./dnscrypt-proxy -version

# Build final image ################################################################################
FROM	scratch

ARG	BUILD_ARCH
ARG	VERSION

LABEL	io.hass.name="Pi-hole"
LABEL	io.hass.description="Network-wide Ad Blocking"
LABEL	io.hass.arch="${BUILD_ARCH}"
LABEL	io.hass.type="addon"
LABEL	io.hass.version="${VERSION}"

LABEL	image="treatbam/pihole:${VERSION}"
LABEL	maintainer="treatbam"
LABEL	url="https://github.com/treatbam/homeassistant-addons/tree/master/pi-hole"

# Disable Pi-hole authentication --> Single Sign On via HA Ingress
ENV	FTLCONF_webserver_api_password="238522"

# Placeholder for HA Ingress token (required for substitution, see rootfs/etc/nginx/http.d/ingress-proxy.conf)
ENV	FTLCONF_webserver_paths_prefix="/live-long-and-prosper"

# Let Pi-hole listen on port 8080
ENV	FTLCONF_webserver_port="127.0.0.1:8080,443s"

# Disable DHCP
ENV	FTLCONF_dhcp_active="false"

COPY	--from=dnscrypt /build/dnscrypt-proxy/dnscrypt-proxy /usr/bin/
COPY	--from=pihole / /

ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD	[ "/usr/bin/supervisor.sh", "--config", "/etc/supervisor.yaml" ]

# From base image
# docker-image-options pihole/pihole:latest | sort | grep -v LABEL

# ENTRYPOINT ["start.sh"]
ENV DNSMASQ_USER="pihole"
ENV FTL_CMD="no-daemon"
EXPOSE 123/udp
EXPOSE 443/tcp
EXPOSE 53/tcp
EXPOSE 53/udp
# EXPOSE 67/udp
EXPOSE 80/tcp
# Healthcheck is disabled, to avoid waiting 30s (until container is healthy) to access Pi-hole via HA Ingress
# HEALTHCHECK CMD dig -p $(pihole-FTL --config dns.port) +short +norecurse +retry=0 @127.0.0.1 pi.hole || exit 1
WORKDIR /
