# The InvenTree dockerfile provides two build targets:
#
# production:
# - Required files are copied into the image
# - Runs InvenTree web server under gunicorn
#
# dev:
# - Expects source directories to be loaded as a run-time volume
# - Runs InvenTree web server under django development server
# - Monitors source files for any changes, and live-reloads server

FROM python:3.11-alpine3.20@sha256:520924f35357a374aa1beaa81b867f449f9f12a53f00b69ad03c3d697fdf4aad AS inventree_base

# Build arguments for this image
ARG commit_tag=""
ARG commit_hash=""
ARG commit_date=""

ARG data_dir="data"

ENV PYTHONUNBUFFERED=1
ENV PIP_DISABLE_PIP_VERSION_CHECK=1
ENV INVOKE_RUN_SHELL="/bin/ash"

ENV INVENTREE_DOCKER="true"

# InvenTree paths
ENV INVENTREE_HOME="/home/inventree"
ENV INVENTREE_DATA_DIR="${INVENTREE_HOME}/${data_dir}"
ENV INVENTREE_STATIC_ROOT="${INVENTREE_DATA_DIR}/static"
ENV INVENTREE_MEDIA_ROOT="${INVENTREE_DATA_DIR}/media"
ENV INVENTREE_BACKUP_DIR="${INVENTREE_DATA_DIR}/backup"
ENV INVENTREE_PLUGIN_DIR="${INVENTREE_DATA_DIR}/plugins"

ENV INVENTREE_BACKEND_DIR="${INVENTREE_HOME}/src/backend"

# InvenTree configuration files
ENV INVENTREE_CONFIG_FILE="${INVENTREE_DATA_DIR}/config.yaml"
ENV INVENTREE_SECRET_KEY_FILE="${INVENTREE_DATA_DIR}/secret_key.txt"
ENV INVENTREE_OIDC_PRIVATE_KEY_FILE="${INVENTREE_DATA_DIR}/oidc.pem"
ENV INVENTREE_PLUGIN_FILE="${INVENTREE_DATA_DIR}/plugins.txt"

# Worker configuration (can be altered by user)
ENV INVENTREE_GUNICORN_WORKERS="4"
ENV INVENTREE_BACKGROUND_WORKERS="4"

# Default web server address:port
ENV INVENTREE_WEB_ADDR=0.0.0.0
ENV INVENTREE_WEB_PORT=8000

LABEL org.opencontainers.image.vendor="inventree" \
      org.opencontainers.image.title="InvenTree backend server" \
      org.opencontainers.image.description="InvenTree is the open-source inventory management system" \
      org.opencontainers.image.url="https://inventree.org" \
      org.opencontainers.image.documentation="https://docs.inventree.org" \
      org.opencontainers.image.source="https://github.com/inventree/InvenTree" \
      org.opencontainers.image.revision=${commit_hash} \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.version=${commit_tag}


# Install required system level packages
RUN apk add --no-cache \
    git gettext py-cryptography \
    # Image format support
    libjpeg libwebp zlib \
    # Weasyprint requirements : https://doc.courtbouillon.org/weasyprint/stable/first_steps.html#alpine-3-12
    py3-pip py3-pillow py3-cffi py3-brotli pango poppler-utils openldap \
    # Postgres client (note: backwards compatible with postgres server <= 16)
    postgresql16-client \
    # MySQL / MariaDB client
    mariadb-client mariadb-connector-c \
    && \
    # font support
    apk --update --upgrade --no-cache add fontconfig ttf-freefont font-terminus font-noto font-noto-cjk font-noto-extra \
    && fc-cache -f

EXPOSE 8000

RUN mkdir -p ${INVENTREE_HOME}
WORKDIR ${INVENTREE_HOME}

COPY contrib/container/requirements.txt base_requirements.txt
COPY src/backend/requirements.txt ./
COPY contrib/container/install_build_packages.sh .
RUN chmod +x install_build_packages.sh

# For ARMv7 architecture, add the piwheels repo (for cryptography library)
# Otherwise, we have to build from source, which is difficult
# Ref: https://github.com/inventree/InvenTree/pull/4598
RUN if [ `apk --print-arch` = "armv7" ]; then \
    printf "[global]\nextra-index-url=https://www.piwheels.org/simple\n" > /etc/pip.conf ; \
    fi

COPY tasks.py contrib/container/gunicorn.conf.py contrib/container/init.sh ./
RUN chmod +x init.sh

ENTRYPOINT ["/bin/ash", "./init.sh"]

FROM inventree_base AS prebuild

ENV PATH=/root/.local/bin:$PATH
RUN ./install_build_packages.sh --no-cache --virtual .build-deps && \
    pip install --user --require-hashes -r base_requirements.txt --no-cache && \
    pip install --user --require-hashes -r requirements.txt --no-cache && \
    apk --purge del .build-deps

# Frontend builder image:
FROM prebuild AS frontend

RUN apk add --no-cache --update nodejs npm yarn bash
RUN npm install -g --ignore-scripts n@10.1.0
RUN bash -c "n lts"
RUN yarn config set network-timeout 600000 -g
COPY src ${INVENTREE_HOME}/src
COPY tasks.py ${INVENTREE_HOME}/tasks.py
RUN cd ${INVENTREE_HOME} && invoke int.frontend-compile

# InvenTree production image:
# - Copies required files from local directory
# - Starts a gunicorn webserver
FROM inventree_base AS production

ENV INVENTREE_DEBUG=False

# As .git directory is not available in production image, we pass the commit information via ENV
ENV INVENTREE_COMMIT_HASH="${commit_hash}"
ENV INVENTREE_COMMIT_DATE="${commit_date}"

# use dependencies and compiled wheels from the prebuild image
ENV PATH=/root/.local/bin:$PATH
COPY --from=prebuild /root/.local /root/.local


# Copy source code
COPY src/backend/InvenTree ${INVENTREE_BACKEND_DIR}/InvenTree
COPY src/backend/requirements.txt ${INVENTREE_BACKEND_DIR}/requirements.txt
COPY --from=frontend ${INVENTREE_BACKEND_DIR}/InvenTree/web/static/web ${INVENTREE_BACKEND_DIR}/InvenTree/web/static/web

# Launch the production server
CMD gunicorn -c ./gunicorn.conf.py InvenTree.wsgi -b 0.0.0.0:8000 --chdir ${INVENTREE_BACKEND_DIR}/InvenTree

FROM inventree_base AS dev

# Vite server (for local frontend development)
EXPOSE 5173

# Install packages required for building python packages
RUN ./install_build_packages.sh

RUN pip install --require-hashes -r base_requirements.txt --no-cache

# Install nodejs / npm / yarn

RUN apk add --no-cache --update nodejs npm yarn bash
RUN npm install -g --ignore-scripts n@10.1.0
RUN bash -c "n lts"
RUN yarn config set network-timeout 600000 -g

# The development image requires the source code to be mounted to /home/inventree/
# So from here, we don't actually "do" anything, apart from some file management

ENV INVENTREE_DEBUG=True

# Location for python virtual environment
# If the INVENTREE_PY_ENV variable is set, the entrypoint script will use it!
ENV INVENTREE_PY_ENV="${INVENTREE_DATA_DIR}/env"

WORKDIR ${INVENTREE_HOME}

# Entrypoint ensures that we are running in the python virtual environment
ENTRYPOINT ["/bin/ash", "./contrib/container/init.sh"]

# Launch the development server
CMD ["invoke", "dev.server", "-a", "${INVENTREE_WEB_ADDR}:${INVENTREE_WEB_PORT}"]
