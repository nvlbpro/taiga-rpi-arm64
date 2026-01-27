# This Source Code Form is subject to the terms of the Mozilla Public
# License, v. 2.0. If a copy of the MPL was not distributed with this
# file, You can obtain one at http://mozilla.org/MPL/2.0/.
#
# Copyright (c) 2021-present Kaleidos INC

FROM python:3.11-slim
LABEL maintainer="support@taiga.io (ARM64 overlay by nvlbpro)"

ENV DEBIAN_FRONTEND=noninteractive
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONFAULTHANDLER=1

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY . /taiga-back
WORKDIR /taiga-back

ENV GOSU_VERSION 1.12

RUN set -eux; \
    apt-get update; \
    apt-get install -y \
    build-essential \
    gettext \
    libpq5 \
    libpq-dev \
    git \
    net-tools \
    procps \
    wget \
    gnupg; \
    \
    apt-get install -y --no-install-recommends ca-certificates wget; \
    dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')"; \
    wget -O /usr/local/bin/gosu \
    "https://github.com/tianon/gosu/releases/download/${GOSU_VERSION}/gosu-${dpkgArch}"; \
    wget -O /usr/local/bin/gosu.asc \
    "https://github.com/tianon/gosu/releases/download/${GOSU_VERSION}/gosu-${dpkgArch}.asc"; \
    chmod +x /usr/local/bin/gosu; \
    \
    export GNUPGHOME="$(mktemp -d)"; \
    gpg --batch --keyserver hkps://keys.openpgp.org \
    --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4; \
    gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu; \
    command -v gpgconf && gpgconf --kill all || :; \
    rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc; \
    \
    python -m pip install --upgrade pip; \
    python -m pip install wheel; \
    python -m pip install -r requirements.txt; \
    python -m pip install -r requirements-contribs.txt; \
    python manage.py compilemessages; \
    python manage.py collectstatic --no-input; \
    chmod +x docker/entrypoint.sh; \
    chmod +x docker/async_entrypoint.sh; \
    cp docker/config.py settings/config.py; \
    \
    groupadd --system taiga --gid=999; \
    useradd --system --no-create-home --gid taiga --uid=999 --shell=/bin/bash taiga; \
    mkdir -p /taiga-back/media/exports; \
    chown -R taiga:taiga /taiga-back; \
    \
    apt-get purge -y \
    build-essential \
    gettext \
    git \
    libpq-dev \
    net-tools \
    procps \
    wget \
    gnupg; \
    apt-get autoremove -y; \
    rm -rf /var/lib/apt/lists/* /root/.cache; \
    \
    rm requirements.txt; \
    rm requirements-contribs.txt; \
    find . -name '__pycache__' -type d -exec rm -rf {} +; \
    find . -name '*.pyc' -delete; \
    find . -name '*.po' -delete

ENV DJANGO_SETTINGS_MODULE=settings.config

EXPOSE 8000
ENTRYPOINT ["./docker/entrypoint.sh"]
