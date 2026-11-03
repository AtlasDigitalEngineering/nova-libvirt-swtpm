ARG KOLLA_BASE=quay.io/openstack.kolla/nova-libvirt:2025.1-ubuntu-noble
FROM ${KOLLA_BASE}

USER root

# Ubuntu's swtpm package makes /var/lib/swtpm-localca owned by swtpm, but
# libvirt in Kolla 20.x runs swtpm as tss and 20.x has no qemu.conf override.
RUN set -eux; \
    mkdir -p /var/lib/swtpm-localca; \
    chown -R tss:tss /var/lib/swtpm-localca; \
    su -s /bin/sh tss -c 'test -w /var/lib/swtpm-localca'
