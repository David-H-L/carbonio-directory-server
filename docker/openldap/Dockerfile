
# syntax=docker/dockerfile:1.7

# ==========================================
# ETAPA 1: Compilación de esquemas Carbonio
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
# ETAPA 2: Obtener Carbonio OpenLDAP
# ==========================================

FROM --platform=$BUILDPLATFORM \
    docker.io/library/ubuntu:jammy AS openldap-installer

ARG TARGETARCH
ARG BUILDARCH

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      ca-certificates \
      openssl \
      gnupg && \
    rm -rf /var/lib/apt/lists/*

# Configurar arquitectura adicional si se usa
# una arquitectura distinta de la del builder.
RUN set -eu; \
    if [ "$TARGETARCH" != "$BUILDARCH" ]; then \
      dpkg --add-architecture "$TARGETARCH"; \
      printf \
        'Types: deb\nURIs: http://ports.ubuntu.com/ubuntu-ports/\nSuites: jammy jammy-updates jammy-security\nComponents: main restricted universe multiverse\nArchitectures: %s\nSigned-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg\n' \
        "$TARGETARCH" \
        > /etc/apt/sources.list.d/ports-target.sources; \
    fi

# Repositorio de desarrollo Zextras.
# Solo para laboratorio.
# En producción se requiere validar la firma
# del repositorio y fijar versiones.
RUN set -eu; \
    printf \
      'deb [arch=%s trusted=yes] https://repo.zextras.tools/repository/ubuntu-devel-jammy jammy main\n' \
      "$TARGETARCH" \
      > /etc/apt/sources.list.d/zextras.list

# Descargar paquetes uno por uno y extraerlos.
RUN set -eu; \
    apt-get update; \
    mkdir -p /staging /tmp/debs; \
    cd /tmp/debs; \
    \
    echo "=== DESCARGANDO PAQUETES ==="; \
    for package in \
      carbonio-openldap \
      carbonio-openssl \
      carbonio-cyrus-sasl \
      carbonio-libsodium \
      libcap2-bin \
      libltdl7 \
      libsystemd0 \
      netcat-openbsd \
      libbsd0 \
      libmd0 \
      openssl; do \
        echo "Descargando ${package}:${TARGETARCH}"; \
        apt-get download "${package}:${TARGETARCH}"; \
    done; \
    \
    echo "=== EXTRAYENDO PAQUETES ==="; \
    for deb in /tmp/debs/*.deb; do \
      echo "Extrayendo $deb"; \
      dpkg -x "$deb" /staging; \
    done; \
    \
    echo "=== VERIFICANDO EJECUTABLE ==="; \
    test -x /staging/opt/zextras/common/libexec/slapd; \
    \
    echo "=== PREPARANDO BIBLIOTECAS ==="; \
    mkdir -p /staging/usr/local/lib; \
    for directory in /staging/lib /staging/usr/lib; do \
      if [ -d "$directory" ]; then \
        find "$directory" -type f -name '*.so*' \
          -exec cp -P {} /staging/usr/local/lib/ \;; \
      fi; \
    done; \
    \
    echo "=== LIMPIEZA ==="; \
    rm -rf /tmp/debs /var/lib/apt/lists/*

# ==========================================
# ETAPA 3: Imagen final para OpenShift
# ==========================================

FROM --platform=$TARGETPLATFORM \
    docker.io/library/ubuntu:jammy

ENV DEBIAN_FRONTEND=noninteractive

# Dependencias base del sistema.
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

# Copiar Carbonio OpenLDAP.
COPY --from=openldap-installer \
    /staging/opt/ /opt/

# Bibliotecas extraídas.
COPY --from=openldap-installer \
    /staging/usr/local/lib/ /usr/local/lib/

# Actualizar caché de bibliotecas.
RUN ldconfig

# Esquemas generados por Maven.
COPY --from=build \
    /build/src/ldap/generated/ \
    /opt/zextras/common/etc/openldap/zimbra/

# Herramientas auxiliares.
COPY docker/openldap/ldap-utils/ /ldap-utils/

# Script de inicio.
COPY docker/openldap/entrypoint.sh /entrypoint.sh

# Variables del entorno.
ENV LD_LIBRARY_PATH=/usr/local/lib

ENV PATH=/opt/zextras/common/bin:/opt/zextras/common/sbin:$PATH

ENV LDAP_PORT=1389

# Crear directorios necesarios.
RUN mkdir -p \
    /opt/zextras/data/ldap/config \
    /opt/zextras/data/ldap/mdb/db \
    /opt/zextras/data/ldap/state/run \
    /opt/zextras/conf/ca \
    /opt/zextras/ldap/custom-ldifs \
    /run/carbonio

# Permisos para UID arbitrario de OpenShift.
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

# Preparar scripts.
RUN chmod 755 /entrypoint.sh && \
    chmod 755 /ldap-utils/zmldapschema && \
    chmod 755 /ldap-utils/replace-olcRootPw

# No realizar chown durante la inicialización
# de esquemas como usuario sin privilegios.
RUN sed -i \
    '/^[[:space:]]*chown zextras:zextras /d' \
    /ldap-utils/zmldapschema

# Evitar impresión de material sensible.
RUN sed -i \
    '/echo "base64 is /d' \
    /ldap-utils/replace-olcRootPw

# Confirmar existencia del ejecutable.
RUN test -x /opt/zextras/common/libexec/slapd

# Puerto LDAP no privilegiado.
EXPOSE 1389/tcp

# UID no privilegiado para pruebas locales.
# OpenShift puede asignar otro UID.
USER 1001

ENTRYPOINT ["/entrypoint.sh"]
