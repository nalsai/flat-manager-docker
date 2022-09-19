FROM fedora:latest AS builder

WORKDIR /root

RUN dnf -y update && \
    dnf -y install cargo postgresql-devel glib2-devel ostree-devel

RUN cargo --quiet install --git https://github.com/flatpak/flat-manager.git --root=/root
#--tag=0.3.7

RUN cd /root/.cargo/git/checkouts/flat-manager*/* && \
    install -m 644 -D -t /root/etc example-config.json

FROM fedora:latest

RUN dnf -y update && \
    dnf -y install flatpak ostree libpq && \
    dnf clean all

COPY --from=builder /root/bin /usr/bin
COPY --from=builder /root/etc /etc/flat-manager

ADD entrypoint.sh /usr/bin
RUN chown root:root /usr/bin/entrypoint.sh && chmod 755 /usr/bin/entrypoint.sh

ENV HOME /flat-manager
ENV REPO_CONFIG $HOME/config.json
ENV STARTUP_SCRIPT $HOME/startup.sh
ENV RUST_LOG info

EXPOSE 8080

ENTRYPOINT ["/usr/bin/entrypoint.sh"]
CMD ["flat-manager"]
