# Docker

Containers on a small home server, started by Docker Compose with a restart policy.

## A signal sent with docker kill stops the restart policy

**`docker kill --signal=<any>` marks the container as stopped by hand, so `unless-stopped`
does not start it again at the next daemon start or reboot; send a reload signal from inside
with `docker exec <container> kill -HUP 1` instead.**

- **Failure (2026-10):** the metrics server stayed down after two reboots in a row while every
  other `unless-stopped` container came back. The daemon logged nothing for it: no start
  attempt and no error. Both times, the last thing done to it beforehand was the documented
  rule reload, `docker kill --signal=HUP <container>`. The signal only reloads the
  configuration and the container keeps running, so nothing looks wrong until the next boot.
  Alerting was dark until someone started it by hand.
- **Measured on:** Docker Engine 29.9.2 with `live-restore` on, 2026-10. In the engine source,
  the kill path sets the container's `HasBeenManuallyStopped` flag for any signal, not only
  `SIGKILL`, unless the daemon itself is shutting down. `unless-stopped` skips a container
  carrying that flag. The flag is kept in the daemon's own state and `docker inspect` does not
  show it. A manual `docker start` clears it, so after the fact there is nothing left to see.
- **Check:** the same flag also stops restarts while the daemon is running, so you can test it
  without a reboot. Run three throwaway containers that ignore `HUP` and exit after a few
  seconds. Send one `HUP` with `docker kill`, one with `docker exec`, and nothing to the third:

  ```sh
  for c in t-kill t-exec t-control; do
    docker run -d --name "$c" --restart unless-stopped --network none alpine:3.22 \
      sh -c 'trap "" HUP; sleep 4; exit 1' >/dev/null
  done
  docker kill --signal=HUP t-kill >/dev/null
  docker exec t-exec kill -HUP 1
  timeout 30 docker wait t-control >/dev/null; timeout 30 docker wait t-control >/dev/null
  for c in t-kill t-exec t-control; do
    docker inspect "$c" --format '{{.Name}} {{.State.Status}} restarts={{.RestartCount}}'
  done
  docker rm -f t-kill t-exec t-control >/dev/null
  ```

  Measured output: `t-kill` reads `exited restarts=0`, while `t-exec` and `t-control` read
  `restarting` or `running`, depending on where the cycle is, with a restart count above zero. The control is what makes this a test. Without it, a
  zero could just mean you looked too early.
- **Recovery:** `docker start <container>` clears the flag. Search your runbooks and scripts for
  `docker kill -s`, `docker kill --signal` and `docker container kill`, then switch each to
  `docker exec <container> kill -<SIGNAL> 1`. That form needs a `kill` binary in the image. A
  server with an HTTP reload endpoint, such as Prometheus with `--web.enable-lifecycle`, can
  use that instead. After any reboot, list containers that should be up and are not:
  `docker ps -a --filter status=exited`.
