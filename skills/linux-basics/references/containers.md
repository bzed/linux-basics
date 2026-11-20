# Containers with podman: least access, hard limits, no leftovers

Read this before running anything with `podman` (most of it applies to `docker` too).

## Contents

- Ground rules
- Volumes: mount only what's needed
- Limits: CPU, memory, processes, privileges
- Networking: only the access that's needed
- Cleaning up
- A template

## Ground rules

A container is only as isolated as you make it. Mount your home directory and the container can
read SSH keys, browser profiles and tokens. Give it the host network and it can reach every
local service. Forget to remove it and it lingers for months. Treat the three points below as
defaults. Deviate only with a concrete reason, and tell the user when you do.

Use **rootless** podman (your normal user). Rootful podman (`sudo podman`) means becoming root,
which needs the user's explicit permission (`SKILL.md`, section 1). Check with
`podman info --format '{{.Host.Security.Rootless}}'`.

## Volumes: mount only what's needed

- **Never mount `$HOME`, `/`, `/etc`, `/var`, `/run` or other system directories**, and never
  `$PWD` when it is `$HOME` or a large parent directory. Mount the specific project or data
  directory the task needs, and nothing more.
- Mount **read-only** (`:ro`) unless the container must write. If it must write, give it a
  dedicated output directory (e.g. a `mktemp -d` directory) rather than write access to the
  source tree.
- Never mount the podman/docker socket (`/run/user/$UID/podman/podman.sock`,
  `/var/run/docker.sock`): that gives the container full control over all containers and, with
  rootful setups, the host.
- Don't pass whole credential sets in. No `~/.ssh`, `~/.aws`, `~/.config/gcloud`, `~/.gnupg`,
  no `--env-host`. Pass individual variables with `-e NAME` only when needed.
- SELinux hosts (Fedora/RHEL): `:z`/`:Z` relabel the mounted directory **on the host**. Use them
  only on a dedicated directory, never on `$HOME`, a system directory or anything shared.
  Debian doesn't use SELinux by default, so they're not needed there.
- Rootless podman maps your UID to root inside the container. Files the container writes end up
  owned by a subordinate UID you can't easily delete. `--userns=keep-id` keeps your own UID
  inside the container and avoids that.
- No `--privileged`, no `--device` you don't need, no `--pid=host`/`--ipc=host`.

## Limits: CPU, memory, processes, privileges

Always set limits. Unlimited containers can take over the machine just like a host process (see
`resources.md`: never all CPU cores unless granted).

```sh
--cpus=4 --memory=4g --pids-limit=512     # pick values from nproc/free -h, not "all"
--timeout=3600                            # kill the container if it runs too long
--cap-drop=all --security-opt=no-new-privileges
--read-only                               # root filesystem read-only (tmpfs for /tmp etc. is added automatically)
```

Add back only the capabilities a workload really needs (`--cap-add=NET_BIND_SERVICE`, …).

## Networking: only the access that's needed

- **Avoid `--network=host`.** It removes network isolation completely: the container sees and
  can reach every service on the host, including ones bound to `127.0.0.1`.
- **No network needed** (builds from local sources, data processing, tests without downloads):
  `--network=none`. The container then only has a loopback interface.
- **Only talking to other containers**: create an internal network with no outside access and
  put them on it:
  `podman network create --internal mytask-net` then `--network=mytask-net`.
- **Needs internet access** (downloads): use the default rootless network (pasta), which gives
  outbound access but no access to host services. Don't add `--network=host` "to make it work".
  Find the actual problem instead.
- **Publishing ports**: bind to localhost only, `-p 127.0.0.1:8080:80`. Plain `-p 8080:80`
  listens on all interfaces, which exposes the service to the network. Publish only the ports
  that are needed.

## Cleaning up

- **Use `--rm`** for every container that doesn't have to outlive the command. It removes the
  container and its anonymous volumes on exit.
- **Label everything you create** so you can find exactly your own resources later:
  `--label agent-task=<name>` on containers, and give networks and volumes a recognisable name.
- **When the task is done, delete what you created**: containers, networks, named volumes, and
  images you built or pulled only for this task.

  ```sh
  podman ps -a --filter label=agent-task=<name> --format '{{.ID}} {{.Names}}'  # look first
  podman rm -f $(podman ps -aq --filter label=agent-task=<name>)
  podman network rm mytask-net
  podman volume rm mytask-data
  podman rmi localhost/mytask-image
  ```

- **Never** run `podman system prune`, `podman container prune`, `podman image prune -a`,
  `podman volume prune` or `podman system reset`. They delete the **user's** stopped containers,
  images and volumes too, which may hold data they care about. Delete only by name, ID or your
  own label.
- Long-running containers you leave running deliberately: tell the user the name and how to
  stop and remove them.

## A template

```sh
work=$(mktemp -d /tmp/mytask.XXXXXX)
podman run --rm --label agent-task=mytask \
  --network=none \
  --cpus=4 --memory=4g --pids-limit=512 --timeout=3600 \
  --cap-drop=all --security-opt=no-new-privileges --read-only \
  --userns=keep-id \
  -v "$PWD/src:/src:ro" -v "$work:/out" \
  docker.io/library/debian:trixie \
  sh -c 'cd /src && ./build.sh /out'
```

Fully qualify image names (`docker.io/library/debian:trixie`): short names can be resolved to
different registries depending on configuration and may prompt for a choice.
