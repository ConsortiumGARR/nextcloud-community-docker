# nextcloud-community-docker

Fork of [nextcloud/docker](https://github.com/nextcloud/docker) that bakes the
application code into the image, eliminating the rsync-on-startup pattern of the
official image. Designed for Kubernetes deployments on NFS storage.

`README.md` is upstream's and still applies, except where noted here.

## Key differences from the official image

- **No `VOLUME /var/www/html`**: the application code lives in the image layer. Only
  user-data directories (`config`, `data`, `custom_apps`, `themes`) are declared as
  volumes.
- **No rsync at startup**: code is copied at build time, not at runtime, in the same
  layer as the extraction. Splitting the two would ship the code twice.
- **Supports both Community and Enterprise**: select the edition via build args. Both
  archives go through the same GPG verification before extraction.
- **`NEXTCLOUD_INIT_HTACCESS` honours its value**: upstream only checks whether the
  variable is defined, so setting it to `false` still enables it.

## Build arguments

|         Argument         |  Default  |                                     Description                                     |
| ------------------------ | --------- | ----------------------------------------------------------------------------------- |
| `ENTERPRISE_ARCHIVE_URL` | *(empty)* | If set, downloads the enterprise ZIP from this URL instead of the community tarball |

## Volumes

Only user-generated data needs to persist across image updates:

|            Path             |                   Content                    |
| --------------------------- | -------------------------------------------- |
| `/var/www/html/config`      | Nextcloud configuration (`config.php`, etc.) |
| `/var/www/html/data`        | User files                                   |
| `/var/www/html/custom_apps` | User-installed apps                          |
| `/var/www/html/themes`      | Custom themes                                |

## Upgrade behaviour

The entrypoint detects the installed version from `config/version.php` (written after
each successful install or upgrade) and compares it to the version baked in the image.

- **Fresh install** (`config/version.php` absent): runs `occ maintenance:install`, then
  writes the sentinel.
- **Upgrade** (image version > installed version): runs `occ upgrade`, then updates the
  sentinel.
- **Downgrade**: refused with an error.
- **Cross-replica safety**: the init block is protected by `flock` on
  `config/nextcloud-init-sync.lock` (on the shared `config` volume).

## Publishing

`.github/workflows/publish.yml` publishes to
`ghcr.io/consortiumgarr/nextcloud`.

- **Community**: leave `enterprise_url` empty to build every version and variant in
  `versions.json`.
- **Enterprise**: set `enterprise_url` to build that version across all its variants.
  The version is read from the archive filename and the run fails if it does not match
  `versions.json`.
- **Tags**: each build is pushed twice, as `34.0.2-apache` which pins that exact
  patch, and as `34-apache` which follows the latest patch of the branch. Enterprise
  builds append `-enterprise` to both.
- **nginx**: every run also builds the sidecar image from the `fpm` one it just
  pushed, tagged `34.0.2-nginx` and `34-nginx`.

Upstream's own workflows were removed: `update-sh.yml` pushes to `master` and would
make it diverge, `images.yml` has the upstream repository hardcoded.

## Serving with nginx

The `fpm` and `fpm-alpine` variants need a web server in front, and that web server
needs to see the application tree.

- nginx serves static assets from disk itself, and checks that the PHP file exists
  locally (`try_files $fastcgi_script_name =404`) before passing the request to fpm.
- Upstream deployments mount the same volume into both containers. That does not work
  here, because the code lives in the image layer and the nginx sidecar runs a
  different image.
- `nginx/Dockerfile` therefore builds an nginx image with the same code copied in, to
  be used as the sidecar image instead of the stock `nginx:alpine`.

Nothing else is needed on the chart side: point `nginx.image` at the `-nginx` tag of
the matching version, and the existing `useImageCode` flag already stops the PVC from
being mounted over the code. The `custom_apps` and `themes` mounts stay, since nginx
serves static assets from them.

`useImageCode` is not in the upstream chart. It lives on the `imagecode` branch of
[ConsortiumGARR/nextcloud-community-helm](https://github.com/ConsortiumGARR/nextcloud-community-helm),
proposed upstream as [nextcloud/helm#847](https://github.com/nextcloud/helm/pull/847).

Both images must be kept at the same version, or nginx serves the static assets of one
release while fpm runs another. The publish workflow always builds them together.

## Keeping up with upstream

- `master` tracks upstream untouched and exists only as the sync anchor.
- `garr` holds the changes and is the default branch, so `workflow_dispatch` can find
  the publish workflow.
- Sync with `git fetch upstream && git rebase upstream/master` on `garr`. Rebasing
  keeps `git diff upstream/master...garr` as the answer to what this fork changes.
- Run `./update.sh` after touching any template or `docker-entrypoint.sh`.

## Files

- `Dockerfile-debian.template`, `Dockerfile-alpine.template`: sources for the
  generated Dockerfiles
- `docker-entrypoint.sh`: install/upgrade logic, structurally identical to the
  official community entrypoint
- `update.sh`: generates `32/`, `33/` and `34/` from the templates and `versions.json`
- `.config/`: default PHP config files, baked into the image and also kept in
  `/usr/src/nextcloud/config/` as a reference for drift detection warnings at startup
- `nginx/Dockerfile`: sidecar image, the app image's code copied onto `nginx:alpine`

Everything under `32/`, `33/` and `34/` is generated. Do not edit it by hand.

## Not included

Post-upgrade DB optimization and `SUBSCRIPTION_KEY` were part of the earlier
standalone image. They belong in a `post-upgrade` hook mounted by the Helm chart,
since upstream already runs whatever it finds in `/docker-entrypoint-hooks.d/`.
