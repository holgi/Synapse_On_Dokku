FROM matrixdotorg/synapse:latest

ENV SYNAPSE_SERVER_NAME matrix.example.org
ENV SYNAPSE_REPORT_STATS yes
ENV SYNAPSE_CONFIG_DIR /config
ENV UID 1000
ENV GID 1000

ADD nginx.conf.sigil /
ADD --chown=1000:1000 homeserver.yaml /config/
ADD --chown=1000:1000 matrix.example.org.log.config /config/
ADD --chown=1000:1000 matrix.example.org.signing.key /config/
