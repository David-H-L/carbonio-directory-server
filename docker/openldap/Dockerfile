
# ==========================================
# ETAPA 1: Compilar los esquemas de Carbonio
# ==========================================

FROM --platform=$BUILDPLATFORM \
  docker.io/library/maven:3-eclipse-temurin-21-alpine AS build

RUN apk add --no-cache git

WORKDIR /build

COPY .git .git
COPY pom.xml pom.xml
COPY src src

RUN --mount=type=cache,target=/root/.m2 \
    mvn clean install -DskipTests


# ==========================================
# ETAPA 2: Obtener paquetes Carbonio LDAP
# ==========================================

FROM --platform=$BUILDPLATFORM \
  docker.io/library/ubuntu:jammy AS openldap-installer

ARG TARGETARCH=amd64
ARG BUILDARCH=amd64

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      ca-certificates \
      openssl \
      gnupg && \
    rm -rf /var/lib/apt/lists/*

# Arquitectura adicional cuando corresponda
RUN if [ "$TARGETARCH" != "$BUILDARCH" ]; then \
      dpkg --add-architecture "$TARGETARCH"; \
      printf \
        'Types: deb\nURIs: http://ports.ubuntu.com/ubuntu-ports/\nSuites: jammy jammy-updates jammy-security\nComponents: main restricted universe multiverse\nArchitectures: %s\nSigned-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg\n' \
        "$TARGETARCH" \
        > /etc/apt/sources.list.d/ports-target.sources; \
    fi

# Repositorio de desarrollo de Zextras.
# SOLO PARA LABORATORIO.
# Antes de producción: verificar firma y fijar versiones.
RUN printf \
    'deb [arch=%s trusted=yes] https://repo.zextras.tools/repository/ubuntu-devel-jammy jammy main\n' \
    "$TARGETARCH" \
    > /etc/apt/sources.list.d/zextras.list

RUN apt-get update && \
    mkdir -p /staging /tmp/debs && \
    cd /tmp/debs && \
    apt-get download \
      carbonio-openldap:${TARGETARCH} \
      carbonio-openssl:${TARGETARCH} \
      carbonio-cyrus-sasl:${TARGETARCH} \
      carbonio-libsodium:${TARGETARCH} \
      libcap2-bin:${TARGETARCH} \
      libltdl7:${TARGETARCH} \
      libsystemd0:${TARGETARCH} \
      netcat-openbsd:${TARGETARCH} \
      libbsd0:${TARGETARCH} \
      libmd0:${TARGETARCH} \
      openssl:${TARGETARCH} && \
    for deb in /tmp/debs/*.deb; do \
      dpkg -x "$deb" /staging; \
    done && \
    mkdir -p /staging/usr/local/lib && \
    find /staging/lib /staging/usr/lib \
      -name '*.so*' -not -path '*/opt/*' \
      -exec cp -P {} /staging/usr/local/lib/ \; && \
    rm -rf /tmp/debs /var/lib/apt/lists/*

# ==========================================
# ETAPA 3: Imagen final para OpenShift
# ==========================================

FROM --platform=$TARGETPLATFORM \
  docker.io/library/ubuntu:jammy

ENV DEBIAN_FRONTEND=noninteractive

# Utilidades necesarias en tiempo de ejecución.
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      bash \
      ca-certificates \
      openssl \
      netcat-openbsd \
      libcap2-bin \
      libltdl7 \
      libsystemd0 \
      libbsd0 \
      libmd0 \
      libsodium23 \
      procps && \
    rm -rf /var/lib/apt/lists/*

# Archivos extraídos de Carbonio OpenLDAP
COPY --from=openldap-installer /staging/opt /opt

COPY --from=openldap-installer \
  /staging/usr/local/lib/ /usr/local/lib/

# Esquemas generados durante la compilación
COPY --from=build \
  /build/src/ldap/generated/ \
  /opt/zextras/common/etc/openldap/zimbra/

# Scripts del proyecto
COPY docker/openldap/ldap-utils/ /ldap-utils/

COPY docker/openldap/entrypoint.sh /entrypoint.sh

ENV LD_LIBRARY_PATH=/usr/local/lib

ENV PATH=/opt/zextras/common/bin:/opt/zextras/common/sbin:$PATH

ENV LDAP_PORT=1389

# Crear rutas requeridas
RUN mkdir -p \
      /opt/zextras/data/ldap/config \
      /opt/zextras/data/ldap/mdb/db \
      /opt/zextras/data/ldap/state/run \
      /opt/zextras/conf/ca \
      /opt/zextras/ldap/custom-ldifs \
      /run/carbonio

# Preparar permisos compatibles con OpenShift
RUN chgrp -R 0 \
      /opt/zextras/data \
      /opt/zextras/conf \
      /opt/zextras/ldap \
      /run/carbonio \
      /opt/zextras/common/etc/openldap && \
    chmod -R g+rwX \
      /opt/zextras/data \
      /opt/zextras/conf \
      /opt/zextras/ldap \
      /run/carbonio \
      /opt/zextras/common/etc/openldap

# El script del fabricante utiliza chown en
# la inicialización de los esquemas.
# Retirar esa operación para ejecución sin root.
RUN sed -i '/^[[:space:]]*chown zextras:zextras /d' \
      /ldap-utils/zmldapschema

# Evitar que el auxiliar imprima material
# derivado de la contraseña de cn=config.
RUN sed -i '/echo "base64 is /d' \
      /ldap-utils/replace-olcRootPw

RUN chmod 755 /entrypoint.sh && \
    chmod 755 /ldap-utils/zmldapschema && \
    chmod 755 /ldap-utils/replace-olcRootPw

# Verificar que el binario existe.
RUN test -x /opt/zextras/common/libexec/slapd

# Puerto LDAP no privilegiado
EXPOSE 1389/tcp

# No fijar UID del namespace.
# OpenShift podrá asignar uno permitido.
USER 1001

ENTRYPOINT ["/entrypoint.sh"]
