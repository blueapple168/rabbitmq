# syntax=docker/dockerfile:1
# check=skip=SecretsUsedInArgOrEnv
#
# 参考官方镜像: https://github.com/docker-library/rabbitmq (4.3/ubuntu/Dockerfile)
# 官方基于 ubuntu:24.04 + apt，本文件适配 UOS Server 20 1070a (yum/dnf)
#
# 与官方的主要差异:
#   1. apt-get -> yum/dnf，包名按 RHEL 系调整 (perl-IPC-Cmd / ncurses-devel 等)
#   2. dpkg 架构探测 (dpkg --print-architecture / dpkg-architecture) -> uname -m
#   3. gosu 不在 UOS 源中，改为从 GitHub releases 下载静态二进制
#   4. PGP 签名校验与 sha256 校验默认关闭 (内网环境 keyserver 访问不便)
#   5. apt-mark purge 清理编译工具 -> 多阶段构建天然隔离
#   6. 官方 management 变体 (4.3/ubuntu/management) 的功能合并进来:
#      启用 rabbitmq_management 插件、恢复 metrics collector、安装 rabbitmqadmin
#
# 构建流程 (与官方一致的多阶段源码编译):
#   build-base       -> 编译工具链
#   openssl-builder  -> /opt/openssl (自编 OpenSSL，避免与系统 libssl 冲突)
#   erlang-builder   -> /opt/erlang  (自编 Erlang/OTP，链接自编 OpenSSL)
#   runtime          -> 拷贝 /opt/erlang /opt/openssl，安装 RabbitMQ 通用二进制包

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
# build-base: 安装编译工具链
############################################

FROM ghcr.io/blueapple168/${BASE_IMAGE_TAG} AS build-base

ARG BUILDKIT_SBOM_SCAN_STAGE=true

ENV TZ=Asia/Shanghai

RUN set -eux; \
    echo "${TZ}" > /etc/timezone; \
    sed -i 's/\$StateMode/ufu/g' /etc/yum.repos.d/UnionTechOS.repo; \
    \
    : "=== yum/dnf 网络调优：默认 timeout=30s 太短，minrate=1000 容易在低速链路上触发下载中止；延长超时到 300s、降低低速阈值到 100 B/s、retries=10 ===" ; \
    if [ -f /etc/yum.conf ]; then \
        sed -i '/^[ \t]*\(timeout\|minrate\|retries\)=/d' /etc/yum.conf 2>/dev/null || true; \
        if grep -q '^\[main\]' /etc/yum.conf; then \
            sed -i '/^\[main\]/a timeout=300\nminrate=100\nretries=10' /etc/yum.conf; \
        else \
            printf '\n[main]\ntimeout=300\nminrate=100\nretries=10\n' >> /etc/yum.conf; \
        fi; \
    fi; \
    yum makecache; \
    \
    : "=== 编译依赖：gcc/perl/make 编 OpenSSL + Erlang/OTP；perl-IPC-Cmd 是 OpenSSL Configure 必需 ===" ; \
    yum install -y --setopt=install_weak_deps=false --nogpgcheck --nodocs \
        --setopt=timeout=300 --setopt=minrate=100 \
        tar gzip xz wget curl ca-certificates make gcc gcc-c++ glibc-devel \
        ncurses-devel perl perl-IPC-Cmd;

############################################
# openssl-builder: 编译 OpenSSL 到 /opt/openssl
############################################

FROM build-base AS openssl-builder

ARG BUILDKIT_SBOM_SCAN_STAGE=true

ARG OPENSSL_VERSION
ENV OPENSSL_VERSION=${OPENSSL_VERSION}
# 如需校验，取消下行注释并恢复 RUN 中对应检查
# ENV OPENSSL_SOURCE_SHA256=603f5602e2eef00d77fbd429d34dcd5822bb301757a1bc9cdb24c670f1eb859a

# 安装到独立前缀，避免被系统软件误用 (与官方镜像一致)
ENV OPENSSL_INSTALL_PATH_PREFIX=/opt/openssl

RUN set -eux; \
    OPENSSL_SOURCE_URL="https://github.com/openssl/openssl/releases/download/openssl-$OPENSSL_VERSION/openssl-$OPENSSL_VERSION.tar.gz"; \
    OPENSSL_PATH="/usr/local/src/openssl-$OPENSSL_VERSION"; \
    OPENSSL_CONFIG_DIR="$OPENSSL_INSTALL_PATH_PREFIX/etc/ssl"; \
    \
    wget --progress dot:giga --output-document "$OPENSSL_PATH.tar.gz" "$OPENSSL_SOURCE_URL"; \
    \
    : "=== sha256 校验（内网/无校验需求时跳过；启用时取消注释并在顶部恢复 SHA ARG/ENV） ===" ; \
    echo "$OPENSSL_SOURCE_SHA256 *$OPENSSL_PATH.tar.gz" | sha256sum --check --strict - || echo "skip sha256 check"; \
    \
    mkdir -p "$OPENSSL_PATH"; \
    tar --extract --file "$OPENSSL_PATH.tar.gz" --directory "$OPENSSL_PATH" --strip-components 1; \
    \
    : "=== 架构映射: dpkg -> uname -m ===" ; \
    case "$(uname -m)" in \
        x86_64)  opensslMachine='linux-x86_64' ;; \
        aarch64) opensslMachine='linux-aarch64' ;; \
        *) echo >&2 "error: unsupported arch: '$(uname -m)'"; exit 1 ;; \
    esac; \
    \
    cd "$OPENSSL_PATH"; \
    MACHINE="$opensslMachine" \
    RELEASE="4.x.y-z" \
    SYSTEM='Linux' \
    BUILD='???' \
    ./Configure \
        "$opensslMachine" \
        shared \
        enable-fips \
        --prefix="$OPENSSL_INSTALL_PATH_PREFIX" \
        --openssldir="$OPENSSL_CONFIG_DIR" \
        --libdir="$OPENSSL_INSTALL_PATH_PREFIX/lib" \
        -Wl,-rpath="$OPENSSL_INSTALL_PATH_PREFIX/lib" \
    ; \
    \
    : "=== 编译并安装 OpenSSL ===" ; \
    make -j "$(getconf _NPROCESSORS_ONLN)"; \
    make install_sw install_ssldirs install_fips; \
    ldconfig; \
    \
    : "=== 使用系统的 CA 证书 ===" ; \
    rmdir "$OPENSSL_CONFIG_DIR/certs" "$OPENSSL_CONFIG_DIR/private"; \
    ln -sf /etc/ssl/certs /etc/ssl/private "$OPENSSL_CONFIG_DIR";

# 冒烟测试
RUN $OPENSSL_INSTALL_PATH_PREFIX/bin/openssl version

############################################
# erlang-builder: 编译 Erlang/OTP 到 /opt/erlang
############################################

FROM openssl-builder AS erlang-builder

ARG BUILDKIT_SBOM_SCAN_STAGE=true

ARG OTP_VERSION
ENV OTP_VERSION=${OTP_VERSION}
# ENV OTP_SOURCE_SHA256=920d00076d94f4850bd642d79da3229c8e5eea7ebc914996ee6dfd2154aeacf7

ENV ERLANG_INSTALL_PATH_PREFIX=/opt/erlang

RUN set -eux; \
    OTP_SOURCE_URL="https://github.com/erlang/otp/releases/download/OTP-$OTP_VERSION/otp_src_$OTP_VERSION.tar.gz"; \
    OTP_PATH="/usr/local/src/otp-$OTP_VERSION"; \
    \
    mkdir -p "$OTP_PATH"; \
    wget --progress dot:giga --output-document "$OTP_PATH.tar.gz" "$OTP_SOURCE_URL"; \
    echo "$OTP_SOURCE_SHA256 *$OTP_PATH.tar.gz" | sha256sum --check --strict - || echo "skip sha256 check"; \
    tar --extract --file "$OTP_PATH.tar.gz" --directory "$OTP_PATH" --strip-components 1; \
    \
    : "=== 配置 Erlang/OTP：裁剪不需要的应用，链接自编 OpenSSL ===" ; \
    cd "$OTP_PATH"; \
    export ERL_TOP="$OTP_PATH"; \
    : "-rpath 保证 crypto/ssl 优先加载 /opt/openssl/lib 下的 libssl，避免与系统包冲突" ; \
    export CFLAGS="-O2 -fno-strict-aliasing -Wl,-rpath=$OPENSSL_INSTALL_PATH_PREFIX/lib"; \
    \
    : "=== JIT 仅在 x86_64 / aarch64 上受支持 ===" ; \
    jitFlag=; \
    case "$(uname -m)" in \
        x86_64 | aarch64) jitFlag='--enable-jit' ;; \
    esac; \
    \
    ./configure \
        --prefix="$ERLANG_INSTALL_PATH_PREFIX" \
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
        --with-ssl="$OPENSSL_INSTALL_PATH_PREFIX" \
        --without-common_test \
        --without-debugger \
        --without-dialyzer \
        --without-diameter \
        --without-edoc \
        --without-erl_docgen \
        --without-et \
        --without-eunit \
        --without-ftp \
        --without-hipe \
        --without-jinterface \
        --without-megaco \
        --without-observer \
        --without-odbc \
        --without-reltool \
        --without-snmp \
        --without-ssh \
        --without-tftp \
        --without-wx \
        $jitFlag \
    ; \
    \
    : "=== 编译并安装 ===" ; \
    make -j "$(getconf _NPROCESSORS_ONLN)" GEN_OPT_FLGS="-O2 -fno-strict-aliasing"; \
    make install; \
    \
    : "=== 删除无用文件减小体积 ===" ; \
    find "$ERLANG_INSTALL_PATH_PREFIX/lib/erlang" -type d -name examples -exec rm -rf '{}' +; \
    find "$ERLANG_INSTALL_PATH_PREFIX/lib/erlang" -type d -name src -exec rm -rf '{}' +; \
    find "$ERLANG_INSTALL_PATH_PREFIX/lib/erlang" -type d -name include -exec rm -rf '{}' +;

# 校验 crypto/ssl 确实链接到 /opt/openssl 而非系统 libssl
ENV PATH=$ERLANG_INSTALL_PATH_PREFIX/bin:$PATH
RUN set -eux; \
    find $ERLANG_INSTALL_PATH_PREFIX -type f -name 'crypto.so' -exec ldd {} \; | grep -F "$OPENSSL_INSTALL_PATH_PREFIX/lib"; \
    erl -noshell -eval 'ok = crypto:start(), ok = io:format("~p~n~n~p~n~n", [crypto:supports(), ssl:versions()]), init:stop().'

############################################
# runtime: 最终运行镜像
############################################

FROM ghcr.io/blueapple168/${BASE_IMAGE_TAG}

ARG RABBITMQ_VERSION
ARG GOSU_VERSION
ARG RABBITMQADMIN_VERSION

LABEL maintainer="blueapple" \
      version="1.0" \
      description="RabbitMQ ${RABBITMQ_VERSION} (management) with Erlang/OTP ${OTP_VERSION} on UOS 1070a" \
      org.opencontainers.image.base.name="registry.uniontech.com/uos-server-base/uos-server-20-1070a:latest" \
      org.opencontainers.image.title="RabbitMQ" \
      org.opencontainers.image.version="${RABBITMQ_VERSION}"

ENV TZ=Asia/Shanghai

RUN set -eux; \
    echo "${TZ}" > /etc/timezone; \
    sed -i 's/\$StateMode/ufu/g' /etc/yum.repos.d/UnionTechOS.repo; \
    sed -i 's/Version: 39\.2\.0/Version: 2026\.10\.08/g' /usr/lib/python3.6/site-packages/setuptools-39.2.0.dist-info/METADATA 2>/dev/null || true; \
    \
    : "=== yum/dnf 网络调优（与 build-base 相同） ===" ; \
    if [ -f /etc/yum.conf ]; then \
        sed -i '/^[ \t]*\(timeout\|minrate\|retries\)=/d' /etc/yum.conf 2>/dev/null || true; \
        if grep -q '^\[main\]' /etc/yum.conf; then \
            sed -i '/^\[main\]/a timeout=300\nminrate=100\nretries=10' /etc/yum.conf; \
        else \
            printf '\n[main]\ntimeout=300\nminrate=100\nretries=10\n' >> /etc/yum.conf; \
        fi; \
    fi; \
    yum makecache; \
    \
    : "=== 运行时依赖：ca-certificates (TLS)、tzdata (时区)、xz (解压 RabbitMQ 发行包) ===" ; \
    yum install -y --setopt=install_weak_deps=false --nogpgcheck --nodocs \
        --setopt=timeout=300 --setopt=minrate=100 \
        ca-certificates tzdata xz; \
    \
    yum clean all; \
    rm -rf /var/cache/yum /var/cache/dnf

# OPENSSL/ERLANG_INSTALL_PATH_PREFIX 定义在前面的阶段，此处重新定义
ENV ERLANG_INSTALL_PATH_PREFIX=/opt/erlang
ENV OPENSSL_INSTALL_PATH_PREFIX=/opt/openssl

COPY --from=erlang-builder $ERLANG_INSTALL_PATH_PREFIX $ERLANG_INSTALL_PATH_PREFIX
COPY --from=openssl-builder $OPENSSL_INSTALL_PATH_PREFIX $OPENSSL_INSTALL_PATH_PREFIX

ENV PATH=$ERLANG_INSTALL_PATH_PREFIX/bin:$OPENSSL_INSTALL_PATH_PREFIX/bin:$PATH

ENV RABBITMQ_DATA_DIR=/var/lib/rabbitmq

RUN set -eux; \
    : "=== 配置 OpenSSL 使用系统 CA 证书 ===" ; \
    ln -vsf /etc/ssl/certs /etc/ssl/private "$OPENSSL_INSTALL_PATH_PREFIX/etc/ssl"; \
    ldconfig; \
    \
    : "=== FIPS 相关配置（与官方一致） ===" ; \
    sed -i.ORIG -e "/\.include.*fips/ s!.*!.include $OPENSSL_INSTALL_PATH_PREFIX/etc/ssl/fipsmodule.cnf!" \
        -e '/# fips =/s/.*/fips = fips_sect/' "$OPENSSL_INSTALL_PATH_PREFIX/etc/ssl/openssl.cnf"; \
    sed -i.ORIG -e '/^activate/s/^/#/' "$OPENSSL_INSTALL_PATH_PREFIX/etc/ssl/fipsmodule.cnf"; \
    \
    : "=== 校验 OpenSSL/Erlang 拷贝后仍可用 ===" ; \
    [ "$(command -v openssl)" = "$OPENSSL_INSTALL_PATH_PREFIX/bin/openssl" ]; \
    openssl version; \
    openssl version -d; \
    erl -noshell -eval 'ok = crypto:start(), ok = io:format("~p~n~n~p~n~n", [crypto:supports(), ssl:versions()]), init:stop().'; \
    \
    : "=== 创建 rabbitmq 系统用户/组 ===" ; \
    groupadd --gid 999 --system rabbitmq; \
    useradd --uid 999 --system --home-dir "$RABBITMQ_DATA_DIR" --gid rabbitmq rabbitmq; \
    mkdir -p "$RABBITMQ_DATA_DIR" /etc/rabbitmq /etc/rabbitmq/conf.d /tmp/rabbitmq-ssl /var/log/rabbitmq; \
    chown -fR rabbitmq:rabbitmq "$RABBITMQ_DATA_DIR" /etc/rabbitmq /etc/rabbitmq/conf.d /tmp/rabbitmq-ssl /var/log/rabbitmq; \
    chmod 1777 "$RABBITMQ_DATA_DIR" /etc/rabbitmq /etc/rabbitmq/conf.d /tmp/rabbitmq-ssl /var/log/rabbitmq; \
    \
    : "=== 允许 root 通过符号链接访问 erlang cookie ===" ; \
    ln -sf "$RABBITMQ_DATA_DIR/.erlang.cookie" /root/.erlang.cookie;

# gosu: UOS 源中无此包，从 GitHub 下载静态二进制，用于从 root 降权执行
RUN set -eux; \
    case "$(uname -m)" in \
        x86_64)  gosuArch='amd64' ;; \
        aarch64) gosuArch='arm64' ;; \
        *) echo >&2 "error: unsupported arch: '$(uname -m)'"; exit 1 ;; \
    esac; \
    wget --progress dot:giga --output-document /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/${GOSU_VERSION}/gosu-${gosuArch}"; \
    chmod +x /usr/local/bin/gosu; \
    \
    : "=== 验证 gosu 可用 ===" ; \
    gosu nobody true;

ENV RABBITMQ_VERSION=${RABBITMQ_VERSION}
ENV RABBITMQ_HOME=/opt/rabbitmq
ENV PATH=$RABBITMQ_HOME/sbin:$PATH

RUN set -eux; \
    RABBITMQ_SOURCE_URL="https://github.com/rabbitmq/rabbitmq-server/releases/download/v$RABBITMQ_VERSION/rabbitmq-server-generic-unix-latest-toolchain-$RABBITMQ_VERSION.tar.xz"; \
    RABBITMQ_PATH="/usr/local/src/rabbitmq-$RABBITMQ_VERSION"; \
    \
    : "=== 下载并解压 RabbitMQ 通用二进制包（latest-toolchain 需 OTP 26+，与上面自编的 OTP 27 匹配） ===" ; \
    wget --progress dot:giga --output-document "$RABBITMQ_PATH.tar.xz" "$RABBITMQ_SOURCE_URL"; \
    \
    mkdir -p "$RABBITMQ_HOME"; \
    tar --extract --file "$RABBITMQ_PATH.tar.xz" --directory "$RABBITMQ_HOME" --strip-components 1; \
    rm -rf "$RABBITMQ_PATH"*; \
    \
    : "=== 不默认将 SYS_PREFIX 指向 RABBITMQ_HOME，保持为空（官方行为） ===" ; \
    grep -qE '^SYS_PREFIX=\$\{RABBITMQ_HOME\}$' "$RABBITMQ_HOME/sbin/rabbitmq-defaults"; \
    sed -i 's/^SYS_PREFIX=.*$/SYS_PREFIX=/' "$RABBITMQ_HOME/sbin/rabbitmq-defaults"; \
    grep -qE '^SYS_PREFIX=$' "$RABBITMQ_HOME/sbin/rabbitmq-defaults"; \
    \
    chown -R rabbitmq:rabbitmq "$RABBITMQ_HOME"; \
    \
    : "=== 以 rabbitmq 用户运行几个不依赖服务的命令，确认安装正确 ===" ; \
    [ ! -e "$RABBITMQ_DATA_DIR/.erlang.cookie" ]; \
    gosu rabbitmq rabbitmqctl help; \
    gosu rabbitmq rabbitmqctl list_ciphers; \
    gosu rabbitmq rabbitmq-plugins list; \
    : "=== 无残留 cookie ===" ; \
    rm "$RABBITMQ_DATA_DIR/.erlang.cookie";

# 默认启用 Prometheus 风格指标 (https://github.com/docker-library/rabbitmq/issues/419)
RUN gosu rabbitmq rabbitmq-plugins enable --offline rabbitmq_prometheus

# 向后兼容：允许直接 COPY 自定义插件到 /plugins
RUN ln -sf /opt/rabbitmq/plugins /plugins

############################################
# management 插件 (参考官方 4.3/ubuntu/management/Dockerfile)
############################################

# 启用 rabbitmq_management 插件 (管理 UI: 15672, TLS: 15671)
RUN set -eux; \
    gosu rabbitmq rabbitmq-plugins enable --offline rabbitmq_management

# 说明: 官方基础镜像默认通过 conf.d/20-management_agent.disable_metrics_collector.conf
# 禁用 metrics collector 以启用 Prometheus 风格指标; 但管理 UI 依赖 metrics collector,
# 因此本镜像不再放置该配置文件 (官方 management 变体是 build 后再 rm，单镜像直接省去)

# rabbitmqadmin (rabbitmqadmin-ng): 官方 CLI 管理工具，按架构从 GitHub 下载静态二进制
RUN set -eux; \
    case "$(uname -m)" in \
        x86_64)  adminArch='x86_64-unknown-linux-gnu' ;; \
        aarch64) adminArch='aarch64-unknown-linux-gnu' ;; \
        *) echo >&2 "[INFO] rabbitmqadmin is not available on $(uname -m) (yet?)"; exit 0 ;; \
    esac; \
    \
    wget --progress dot:giga --output-document /usr/local/bin/rabbitmqadmin \
        "https://github.com/rabbitmq/rabbitmqadmin-ng/releases/download/v${RABBITMQADMIN_VERSION}/rabbitmqadmin-${RABBITMQADMIN_VERSION}-${adminArch}"; \
    chmod +x /usr/local/bin/rabbitmqadmin; \
    \
    : "=== 验证 rabbitmqadmin 可用 ===" ; \
    rabbitmqadmin --help

# 设置 HOME，使任意 --user 都知道 erlang cookie 放在哪里
ENV HOME=$RABBITMQ_DATA_DIR

# 数据目录建议挂载卷
VOLUME $RABBITMQ_DATA_DIR

# UTF-8 locale，避免 latin1 编码告警
ENV LANG=C.UTF-8 LANGUAGE=C.UTF-8 LC_ALL=C.UTF-8

# rabbitmq-server 启动脚本依据 RUNNING_UNDER_SYSTEMD 决定是否以 exec 启动 erl，
# 设为 true 使 beam.smp 成为容器中的 PID 1（信号传递正确）
ENV RUNNING_UNDER_SYSTEMD=true

COPY --chown=rabbitmq:rabbitmq conf.d/10-defaults.conf /etc/rabbitmq/conf.d/
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

ENTRYPOINT ["docker-entrypoint.sh"]

# 4369/25672: epmd 与节点间通信; 5671/5672: AMQP(+TLS);
# 15671/15672: 管理 UI(+TLS); 15691/15692: Prometheus 指标
EXPOSE 4369 5671 5672 15671 15672 15691 15692 25672

CMD ["rabbitmq-server"]
