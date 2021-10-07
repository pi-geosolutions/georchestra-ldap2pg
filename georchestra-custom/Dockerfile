FROM debian:buster-slim

LABEL project="georchestra.org"
LABEL app="ldap2pg"
LABEL org.opencontainers.image.authors="jeanpommier@pi-geosolutions.fr"

RUN set -ex; \
    apt-get update ; \
    apt-get install -y --no-install-recommends \
        python3 \
        python3-ldap \
        python3-pip \
        python3-psycopg2 \
        python3-setuptools \
        python3-yaml \
    ; \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*; \
    :

COPY . /ldap2pg

RUN cd /ldap2pg ; \
    pip3 install ldap3 ;\
    python3 setup.py install

RUN cp /ldap2pg/georchestra-custom/ldap2pg.yml /etc/ldap2pg.yml && \
    mv /ldap2pg/georchestra-custom/docker/* / && \
    chmod +x /entrypoint.sh && \
    chmod +x /docker-entrypoint.d/*

ENTRYPOINT ["/entrypoint.sh"]

WORKDIR /workspace

#CMD ["ldap2pg"]
