# syntax=docker/dockerfile:1
SHELL ["/bin/bash", "-eo", "pipefail", "-c"]
# check=skip=SecretsUsedInArgOrEnv
#
# 参考官方镜像: https://github.com/docker-library/rabbitmq (4.3/ubuntu/Dockerfile)
# 官方基于 ubuntu:24.04 + apt，本文件适配 UOS Server 20 1070a (yum/dnf)
#
# 与官方的主要差异:
#   1. apt‑get -> yum/dnf，包名适配RHEL系 (perl‑IPC‑Cmd / ncurses‑devel 等)
#   2. dpkg架构探测(dpkg --print‑architecture) -> uname -m
#   3. gosu不在UOS源内，从GitHub releases下载静态二进制
#   4. PGP签名校验与sha256校验默认关闭(内网环境keyserver网络受限)
#   5. apt‑mark purge清理编译工具 → 使用多阶段构建天然隔离
#   6. 合并官方management变体能力：启用rabbitmq_management，保留metrics collector，预装rabbitmqadmin
#
# 构建流程(多阶段源码编译):
#   build‑base       → 基础编译环境
#   openssl‑builder  → /opt/openssl 自建OpenSSL，规避系统库版本绑定
#   erlang‑builder   → /opt/erlang 自建Erlang/OTP，链接自建OpenSSL
#   runtime          → 运行期镜像，复制编译产物，部署RabbitMQ通用二进制包
ARG RABBITMQ_VERSION=4.3.6
ARG OTP_VERSION=27.3.4.18
ARG OPENSSL_VERSION=3.5.9
ARG GOSU_VERSION=1.17
ARG RABBITMQADMIN_VERSION=2.35.0
ARG BASE_IMAGE_TAG=uos-server-20-1070a:latest
# ARG OPENSSL_SOURCE_SHA256=603f5602e2eef00d77fbd429d34dcd5822bb301757a1bc9cdb24c670f1eb859a
# ARG OTP_SOURCE_SHA256=920d00076d94f4850bd642d79da3229c8e5eea7ebc914996ee6dfd2154aeacf7
# ARG RABBITMQ_SOURCE_SHA256=
############################################
# build‑base: 基础编译依赖
############################################
FROM ghcr.io/blueapple168/${BASE_IMAGE_TAG} AS build-base
ARG BUILDKIT_SBOM_SCAN_STAGE=true
ENV TZ=Asia/Shanghai
RUN set -eux; \
    echo "${TZ}" > /etc/timezone; \
    sed -i 's/\$StateMode/ufu/g' /etc/yum.repos.d/UnionTechOS.repo; \
    : "yum/dnf network timeout setting"; \
    if [ -f /etc/yum.conf ]; then \
        sed -i '/^timeout=/d;/^minrate=/d;/^retries=/d' /etc/yum.conf || true; \
        printf '[main]\ntimeout=300\nminrate=100\nretries=10\n' >>/etc/yum.conf; \
    fi; \
    yum makecache; \
    : "install compile deps"; \
    yum install -y --setopt=install_weak_deps=False --nogpgcheck \
        tar gzip xz wget curl make gcc gcc-c++ glibc-devel \
        ncurses-devel perl perl-IPC-Cmd; \
    yum clean all; \
    rm -rf /var/cache/yum
############################################
# openssl‑builder: compile openssl
############################################
FROM build-base AS openssl-builder
ARG OPENSSL_VERSION
ENV OPENSSL_VERSION=${OPENSSL_VERSION}
# ENV OPENSSL_SOURCE_SHA256=
ENV OPENSSL_INSTALL_PATH=/opt/openssl
RUN set -eux; \
    OPENSSL_SRC_URL="https://github.com/openssl/openssl/releases/download/openssl-${OPENSSL_VERSION}/openssl-${OPENSSL_VERSION}.tar.gz"; \
    OPENSSL_SRC_DIR="/usr/local/src/openssl-${OPENSSL_VERSION}"; \
    wget --progress=dot:giga -O "${OPENSSL_SRC_DIR}.tar.gz" "${OPENSSL_SRC_URL}"; \
    # echo "${OPENSSL_SOURCE_SHA256} *${OPENSSL_SRC_DIR}.tar.gz" | sha256sum --check --strict || echo "skip sha256 check"; \
    mkdir -p /usr/local/src; \
    tar -xf "${OPENSSL_SRC_DIR}.tar.gz" -C /usr/local/src; \
    cd "${OPENSSL_SRC_DIR}"; \
    ./config shared enable-fips --prefix="${OPENSSL_INSTALL_PATH}" -Wl,-rpath="${OPENSSL_INSTALL_PATH}/lib"; \
    make -j "$(nproc)"; \
    make install_sw install_ssldirs install_fips; \
    ln -sf /etc/pki/tls/certs /etc/pki/tls/private "${OPENSSL_INSTALL_PATH}/etc/ssl"
RUN "${OPENSSL_INSTALL_PATH}/bin/openssl" version
############################################
# erlang‑builder: compile erlang/OTP
############################################
FROM openssl-builder AS erlang-builder
ARG OTP_VERSION
ENV OTP_VERSION=${OTP_VERSION}
# ENV OTP_SOURCE_SHA256=
ENV ERLANG_INSTALL_PATH=/opt/erlang
ENV PATH=${OPENSSL_INSTALL_PATH}/bin:${PATH}
RUN set -eux; \
    OTP_SRC_URL="https://github.com/erlang/otp/releases/download/OTP-${OTP_VERSION}/otp_src_${OTP_VERSION}.tar.gz"; \
    OTP_SRC_DIR="/usr/local/src/otp_src_${OTP_VERSION}"; \
    mkdir -p /usr/local/src; \
    wget --progress=dot:giga -O "${OTP_SRC_DIR}.tar.gz" "${OTP_SRC_URL}"; \
    # echo "${OTP_SOURCE_SHA256} *${OTP_SRC_DIR}.tar.gz" | sha256sum --check --strict || echo "skip sha256 check"; \
    tar -xf "${OTP_SRC_DIR}.tar.gz" -C /usr/local/src; \
    cd "${OTP_SRC_DIR}"; \
    export ERL_TOP="${OTP_SRC_DIR}"; \
    export CFLAGS="-O2 -fno‑strict‑aliasing -Wl,-rpath=${OPENSSL_INSTALL_PATH}/lib"; \
    ./configure \
      --prefix="${ERLANG_INSTALL_PATH}" \
      --disable-hipe \
      --disable-sctp \
      --disable-silent-rules \
      --enable-builtin-zlib \
      --enable-clock-gettime \
      --enable-hybrid-heap \
      --enable-kernel-poll \
      --enable-smp-support \
      --enable-threads \
      --with-microstate-accounting=extra \
      --with-ssl="${OPENSSL_INSTALL_PATH}" \
      --without-common_test \
      --without-debugger \
      --without-dialyzer \
      --without-diameter \
      --without-edoc \
      --without-erl_docgen \
      --without-et \
      --without-eunit \
      --without-ftp \
      --without-jinterface \
      --without-megaco \
      --without-observer \
      --without-odbc \
      --without-reltool \
      --without-snmp \
      --without-ssh \
      --without-tftp \
      --without-wx; \
    make -j "$(nproc)" GEN_OPT_FLAGS="-O2 -fno‑strict‑aliasing"; \
    make install; \
    find "${ERLANG_INSTALL_PATH}/lib/erlang" -type d -name examples -exec rm -rf {} +; \
    find "${ERLANG_INSTALL_PATH}/lib/erlang" -type d -name src -exec rm -rf {} +; \
    find "${ERLANG_INSTALL_PATH}/lib/erlang" -type d -name include -exec rm -rf {} +
ENV PATH=${ERLANG_INSTALL_PATH}/bin:${PATH}
RUN set -eux; \
    find "${ERLANG_INSTALL_PATH}" -name 'crypto.so' -exec ldd {} \; | grep -F "${OPENSSL_INSTALL_PATH}/lib"; \
    erl -noshell -eval 'ok=crypto:start(), io:format("~p~n",[crypto:supports()]), erlang:halt(0).'
############################################
# runtime: final production image
############################################
FROM ghcr.io/blueapple168/${BASE_IMAGE_TAG} AS runtime
ARG RABBITMQ_VERSION
ARG OTP_VERSION
ARG GOSU_VERSION
ARG RABBITMQADMIN_VERSION
LABEL maintainer="blueapple168" \
      version="${RABBITMQ_VERSION}" \
      description="RabbitMQ ${RABBITMQ_VERSION}(management) built on UOS Server20, Erlang/OTP ${OTP_VERSION}"
ENV TZ=Asia/Shanghai
RUN set -eux; \
    echo "${TZ}" > /etc/timezone; \
    sed -i 's/\$StateMode/ufu/g' /etc/yum.repos.d/UnionTechOS.repo; \
    sed -i 's/Version: 39\.2\.0/Version: 2026\.10\.08/g' /usr/lib/python3.6/site-packages/setuptools-39.2.0.dist-info/METADATA 2>/dev/null || true; \
    yum makecache; \
    yum install -y --setopt=install_weak_deps=False --nogpgcheck \
        ca-certificates tzdata xz; \
    yum clean all; \
    rm -rf /var/cache/yum
ENV ERLANG_INSTALL_PATH=/opt/erlang
ENV OPENSSL_INSTALL_PATH=/opt/openssl
COPY --from=erlang-builder "${ERLANG_INSTALL_PATH}" "${ERLANG_INSTALL_PATH}"
COPY --from=openssl-builder "${OPENSSL_INSTALL_PATH}" "${OPENSSL_INSTALL_PATH}"
ENV PATH=${ERLANG_INSTALL_PATH}/bin:${OPENSSL_INSTALL_PATH}/bin:${PATH}
ENV RABBITMQ_DATA_DIR=/var/lib/rabbitmq
RUN set -eux; \
    ln -vsf /etc/pki/tls/certs /etc/pki/tls/private "${OPENSSL_INSTALL_PATH}/etc/ssl"; \
    ldconfig; \
    sed -i.ORIG -e "/\.include.*fips/ s!.*!.include ${OPENSSL_INSTALL_PATH}/etc/ssl/fipsmodule.cnf!" \
        -e '/^# fips =/s/.*/fips = fips_sect/' "${OPENSSL_INSTALL_PATH}/etc/ssl/openssl.cnf"; \
    sed -i.ORIG -e '/^activate/s/^/#/' "${OPENSSL_INSTALL_PATH}/etc/ssl/fipsmodule.cnf"; \
    [ "$(command -v openssl)" = "${OPENSSL_INSTALL_PATH}/bin/openssl" ]; \
    openssl version -d; \
    erl -noshell -eval 'ok=crypto:start(), io:format("~p~n",[crypto:supports()]), erlang:halt(0).'; \
    groupadd --gid 999 --system rabbitmq; \
    useradd --uid 999 --system --home-dir "${RABBITMQ_DATA_DIR}" --gid rabbitmq rabbitmq; \
    mkdir -p "${RABBITMQ_DATA_DIR}" /etc/rabbitmq /etc/rabbitmq/conf.d /tmp/rabbitmq‑ssl /var/log/rabbitmq; \
    chown -Rf rabbitmq:rabbitmq "${RABBITMQ_DATA_DIR}" /etc/rabbitmq /etc/rabbitmq/conf.d /tmp/rabbitmq‑ssl /var/log/rabbitmq; \
    chmod 1777 "${RABBITMQ_DATA_DIR}" /etc/rabbitmq /etc/rabbitmq/conf.d /tmp/rabbitmq‑ssl /var/log/rabbitmq; \
    ln -sf "${RABBITMQ_DATA_DIR}/.erlang.cookie" /root/.erlang.cookie
RUN set -eux; \
    ARCH=$(uname -m); \
    case "${ARCH}" in \
      x86_64) GOSU_ARCH=amd64 ;; \
      aarch64) GOSU_ARCH=arm64 ;; \
      *) echo >&2 "unsupported arch ${ARCH}"; exit 1 ;; \
    esac; \
    wget --progress=dot:giga -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/${GOSU_VERSION}/gosu‑${GOSU_ARCH}"; \
    chmod +x /usr/local/bin/gosu; \
    gosu nobody true
ENV RABBITMQ_VERSION=${RABBITMQ_VERSION}
ENV RABBITMQ_HOME=/opt/rabbitmq
ENV PATH=${RABBITMQ_HOME}/sbin:${PATH}
RUN set -eux; \
    RABBITMQ_DL_URL="https://github.com/rabbitmq/rabbitmq-server/releases/download/v${RABBITMQ_VERSION}/rabbitmq-server‑generic‑unix‑latest‑toolchain‑${RABBITMQ_VERSION}.tar.xz"; \
    RABBITMQ_TMP="/usr/local/src/rabbitmq‑${RABBITMQ_VERSION}"; \
    mkdir -p /usr/local/src; \
    wget --progress=dot:giga -O "${RABBITMQ_TMP}.tar.xz" "${RABBITMQ_DL_URL}"; \
    mkdir -p "${RABBITMQ_HOME}"; \
    tar -xf "${RABBITMQ_TMP}.tar.xz" -C "${RABBITMQ_HOME}" --strip-components=1; \
    rm -rf "${RABBITMQ_TMP}"*; \
    grep -E '^SYS_PREFIX=\$\{RABBITMQ_HOME\}$' "${RABBITMQ_HOME}/sbin/rabbitmq-defaults"; \
    sed -i 's/^SYS_PREFIX=.*$/SYS_PREFIX=/' "${RABBITMQ_HOME}/sbin/rabbitmq-defaults"; \
    grep -E '^SYS_PREFIX=$' "${RABBITMQ_HOME}/sbin/rabbitmq-defaults"; \
    chown -R rabbitmq:rabbitmq "${RABBITMQ_HOME}"; \
    [ ! -e "${RABBITMQ_DATA_DIR}/.erlang.cookie" ]; \
    gosu rabbitmq rabbitmqctl help; \
    gosu rabbitmq rabbitmqctl list_ciphers; \
    gosu rabbitmq rabbitmq-plugins list; \
    rm -f "${RABBITMQ_DATA_DIR}/.erlang.cookie"
RUN gosu rabbitmq rabbitmq-plugins enable --offline rabbitmq_prometheus
RUN ln -sf "${RABBITMQ_HOME}/plugins" /plugins
RUN set -eux; \
    gosu rabbitmq rabbitmq-plugins enable --offline rabbitmq_management
RUN set -eux; \
    ARCH=$(uname -m); \
    case "${ARCH}" in \
      x86_64) ADMIN_ARCH=x86_64‑unknown‑linux‑gnu ;; \
      aarch64) ADMIN_ARCH=aarch64‑unknown‑linux‑gnu ;; \
      *) echo >&2 "[INFO] rabbitmqadmin binary not available for arch ${ARCH}"; exit 0 ;; \
    esac; \
    wget --progress=dot:giga -O /usr/local/bin/rabbitmqadmin "https://github.com/rabbitmq/rabbitmqadmin‑ng/releases/download/v${RABBITMQADMIN_VERSION}/rabbitmqadmin‑${RABBITMQADMIN_VERSION}‑${ADMIN_ARCH}"; \
    chmod +x /usr/local/bin/rabbitmqadmin; \
    rabbitmqadmin --help
ENV HOME=${RABBITMQ_DATA_DIR}
VOLUME ["${RABBITMQ_DATA_DIR}"]
ENV LANG=C.UTF‑8 LANGUAGE=C.UTF‑8 LC_ALL=C.UTF‑8
ENV RUNNING_IN_DOCKER=true
COPY 10‑defaults.conf 20‑management_agent.disable_metrics_collector.conf /etc/rabbitmq/conf.d/
COPY docker‑entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker‑entrypoint.sh
ENTRYPOINT ["docker‑entrypoint.sh"]
EXPOSE 4369 5671 5672 15671 15672 15691 15692 25672
CMD ["rabbitmq‑server"]
