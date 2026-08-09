# syntax=docker/dockerfile:1

# Drop-in replacement for the nginx sidecar image, carrying the Nextcloud code so
# that nginx can serve static assets and resolve try_files without a shared volume.
ARG NEXTCLOUD_IMAGE

FROM ${NEXTCLOUD_IMAGE} AS code

FROM nginx:alpine

COPY --from=code /var/www/html /var/www/html
