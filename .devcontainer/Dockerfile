
FROM mcr.microsoft.com/devcontainers/base:ubuntu

ENV container=docker

RUN sudo apt-get update > /dev/null \
	# Node.js
	# && wget -qO- https://deb.nodesource.com/setup_lts.x | sudo sh \
	# Common
	# && sudo apt-get install -y file nodejs \
	&& sudo apt-get install -y file \
	# Arkade
	&& curl -fsSL https://get.arkade.dev | sudo sh

COPY --chmod=755 ./entrypoint.sh /entrypoint.sh

COPY ./kind-cluster-config.yaml /kind-cluster-config.yaml

ENTRYPOINT ["/entrypoint.sh"]
