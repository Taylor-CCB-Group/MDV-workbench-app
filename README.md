# MDV on Verily Workbench

This repository defines [MDV](https://github.com/Taylor-CCB-Group/MDV), the Multi-Dimensional
Viewer, as a [Verily Workbench](https://workbench.verily.com) custom app. Each member of a
workspace launches their own MDV instance from the workspace's Apps tab.

The repository holds only the app definition. The MDV code is built into a container image,
which Workbench pulls from Artifact Registry when the app launches.

## Why this repository is public

Workbench clones the app definition anonymously at launch, so a public repository needs no
GitHub account link and no GitHub App installation. For the same reason, nothing in it names a
bucket, a workspace or a credential. Settings that differ between workspaces come from
environment variables, including the `WORKBENCH_<resource_name>` variables Workbench sets for
each workspace resource.

## Files

The repository holds 5 files:

- `.devcontainer.json`: runs Verily's startup scripts as root for user `pn`, installs the Java
  and gcloud tools those scripts need, and points the Workbench proxy at port 5055
- `docker-compose.yaml`: the `application-server` container, its single published port, the
  fuse capabilities that gcsfuse needs, the `/app/mdv` volume and MDV's environment
- `devcontainer-template.json`: the template options Workbench fills in at launch, such as the
  container's memory limit
- `mdv-config.json`: the MDV extensions to load, which leaves out the chat extension
- `README.md`: this file

## How MDV is configured

The compose file sets MDV up for a single user per instance:

- authentication is off, because the Workbench proxy decides who can reach the app and
  workspace membership decides who can see its projects
- MDV keeps its working copy of projects and the SQLite database in `/app/mdv`, on the
  instance
- per-file database sync is off, so startup does not walk every file in every project
- the chat extension is not loaded

The image must run gunicorn with a single worker, which the MDV Dockerfile does. MDV registers
projects in process memory, so a second worker would not see projects the first one created.

## Before the first launch

Replace `REPLACE_WITH_ARTIFACT_REGISTRY_IMAGE` in `docker-compose.yaml` with the image's path
in Artifact Registry.

## What persists

Workbench app storage is ephemeral, so treat everything on the instance as temporary. That
includes the `mdv-data` volume holding the projects and the SQLite database, which can be lost
when the app stops or restarts and is always lost when the app is deleted.

The durable copy of a project belongs in the workspace bucket. MDV keeps a working copy in
`/app/mdv` because its HDF5 and SQLite files need local disk, and a sync between that working
copy and the bucket is planned. Until the sync is in place, projects do not outlive the
instance.

## Running it locally

You can run the app definition with Docker on your own machine to check a change before
launching it in Workbench:

1. Create the network Workbench would provide, with `docker network create app-network`.
2. Copy the repository to a scratch directory, and in the copy of `docker-compose.yaml` replace
   the image placeholder and each `${templateOption:...}` value, for example `64m` for
   `shmSize` and `8192m` for `memoryLimit`.
3. If your machine has no `/dev/fuse`, which is the case on macOS, remove the `devices` entry
   from the copy.
4. If port 5055 is already in use, change the host side of the port mapping, for example to
   `5099:5055`.
5. Run `docker compose up -d` in the copy and open `http://localhost:5055`, or the port you
   chose.

Verily's startup scripts do not run locally, because they need a Workbench instance. To check
that MDV works behind a prefix-stripping proxy like Workbench's, put a reverse proxy in front
that serves the app under a path such as `/app/test/proxy/5055/` and strips that path before
forwarding.

## References

- [Verily Workbench custom app examples](https://github.com/verily-src/workbench-app-devcontainers)
- [MDV](https://github.com/Taylor-CCB-Group/MDV)
