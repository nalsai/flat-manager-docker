This repository is no longer maintained as the [official flat-manager repository](https://github.com/flatpak/flat-manager) now builds working docker images.  
Use ghcr.io/flatpak/flat-manager:6bd4999b68748245abbbca5dbd686e60839f3a67 or later instead.  
You can set the home directory and config file path using environment variables:

```yaml
    environment:
      HOME: /flat-manager
      REPO_CONFIG: /flat-manager/config.json
      RUST_LOG: info
```

# flat-manager-docker

flat-manager serves and maintains a Flatpak repository. You point it
at an ostree repository and it will allow Flatpak clients to install
apps from the repository over HTTP. Additionally, it has an HTTP API
that lets you upload new builds and manage the repository.

You can find flat-manager here: <https://github.com/flatpak/flat-manager>

This Docker image is based on <https://github.com/elementary/flat-manager-docker>,
however it currently doesn't include goofys or catfs and it is based on fedora instead of alpine.
Also, it is built from the latest master. If there is a new release of flat-manager,
I may switch to building from the latest release tag.

You can get the image from ghcr:

```bash
docker pull ghcr.io/nalsai/flat-manager:latest
```

## Setup

You need to run the flat-manager image and a postgres database.

Example docker-compose.yml with the database and labels for traefik.

```yaml
version: "3.9"

services:
  flat-manager:
    image: ghcr.io/nalsai/flat-manager
    restart: unless-stopped
    volumes:
      - ./flatman:/flat-manager
    depends_on:
      - "db"
    ports:
      - 8080:8080
    networks:
      - internal
      - traefik
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.flatpak.rule=Host(`flatpak.example.com`)"
      - "traefik.http.routers.flatpak.entrypoints=websecure"
      - "traefik.http.routers.flatpak.tls.certresolver=letsencrypt"

  db:
    image: postgres:14-alpine
    restart: unless-stopped
    volumes:
      - ./postgresql:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: repo
      POSTGRES_USER: flatmanager
      POSTGRES_PASSWORD: mysecretpassword
    networks:
      - internal

networks:
  traefik:
    external: true
  internal:
```

You need to have a config.json in the folder mounted to /flat-manager (./flatman in the example docker-compose.yml), containing the configuration for flat-manager.  
Here is an example config:

```json
{
    "repos": {
        "stable": {
            "path": "/flat-manager/repo",
            "collection-id": "org.test.Stable",
            "suggested-repo-name": "testrepo",
            "runtime-repo-url": "https://dl.flathub.org/repo/flathub.flatpakrepo",
            "gpg-key": null,
            "base-url": "https://flatpak.example.com/repo/stable",
            "subsets": {
                "all": {
                    "collection-id": "org.test.Stable",
                    "base-url": null
                },
                "free": {
                    "collection-id": "org.test.Stable.test",
                    "base-url": null
                }
            }
        },
        "beta": {
            "path": "/flat-manager/beta-repo",
            "collection-id": "org.test.Beta",
            "suggested-repo-name": "testrepo-beta",
            "runtime-repo-url": "https://dl.flathub.org/repo/flathub.flatpakrepo",
            "gpg-key": null,
            "subsets": {
                "all": {
                    "collection-id": "org.test.Beta",
                    "base-url": null
                },
                "free": {
                    "collection-id": "org.test.Beta.test",
                    "base-url": null
                }
            }
        }
    },
    "base-url": "https://flatpak.example.com",
    "host": "0.0.0.0",
    "port": 8080,
    "delay-update-secs": 10,
    "database-url": "postgres://flatmanager:mysecretpassword@db:5432/repo",
    "build-repo-base": "/flat-manager/build-repo",
    "build-gpg-key": null,
    "gpg-homedir": "/flat-manager/gpg",
    "secret": "mysecretsecret"
}
```

flat-manager maintains a set of repositories specified in the configuration, as well as a set of dynamically generated repositories beneath the configured build-repo-base path.  
For testing with the example configuration, these can be initialized by doing:

```bash
ostree --repo=repo init --mode=archive-z2
ostree --repo=beta-repo init --mode=archive-z2
mkdir build-repo
```

You probably also want to clone the <https://github.com/flatpak/flat-manager> repo and use

```bash
echo "mysecretsecret" | cargo run --bin gentoken -- --base64 --secret-file - --name testtoken  # use the secret from your config.json
```

to generate a token from you secret. This token is used to interact with the flat-manager server.

The official flat-manager repo also contains flat-manager-client, a Python-based client that can be used to talk to the server.  
To run it, you need python3 and python3-aiohttp.

To test adding something to the repository, you can try building a simple app and exporting it to a repository:

```bash
git clone https://github.com/flathub/org.gnome.eog.git test-build
cd test-build
flatpak-builder --install-deps-from=flathub --repo=local-repo builddir org.gnome.eog.yml
cd ..
```

Then you can create a new "build", upload the build to it, commit it and the publish the build like this:

```bash
export REPO_TOKEN=mysecrettoken  # use the token generated earlier
./flat-manager-client create http://127.0.0.1:8080 stable  # this outputs the url you need for the next steps
./flat-manager-client push --commit URL test-build/local-repo
./flat-manager-client publish URL
```

## Configuration

You can use the `RUST_LOG` environment variable in the docker-compose.yml to adjust the log level.

```yaml
    environment:
      RUST_LOG: error
```

If you only want one repo and have it accesssible at /repo/ instead of /repo/name/, you can use something like this:  
DISCLAIMER: I do not know if this config is supported, but it works and I am using it. You need to supply an empty string with "" as the repo name. You can not use any other repos alongside it.

```json
    "repos": {
        "": {
            "path": "/flat-manager/repo",
            "collection-id": "org.test.Stable",
            "suggested-repo-name": "testrepo",
            "runtime-repo-url": "https://dl.flathub.org/repo/flathub.flatpakrepo",
            "gpg-key": null,
            "base-url": "https://flatpak.example.com/repo",
            "subsets": {
                "all": {
                    "collection-id": "org.test.Stable",
                    "base-url": null
                }
            }
        }
    },
```

### GPG

To use gpg you can use the following commands.
Export your key and import it to a new gpg home directory which you give to flat-manager.

```bash
# list keys
gpg --list-secret-keys --keyid-format LONG

# export key
gpg --export -a KEYID > gpg-pub.asc
gpg --export-secret-keys -a KEYID > gpg-sc.asc

# import to new homedir
mkdir gpg
gpg --homedir gpg --import gpg-pub.asc
gpg --homedir gpg --import gpg-sc.asc

# remove the passphrase
gpg --homedir gpg --edit-key KEYID

# permissions
chmod 700 gpg
chmod 600 gpg/*
```

Then you can move the gpg directory to your bind mount and specify the key id in the config.json.
