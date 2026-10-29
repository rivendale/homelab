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
- **Measured on:** Docker Engine 29.9.2 with `live-restore` on, 2026-10. Any signal sent
  through the kill API sets the container's `HasBeenManuallyStopped` flag, unless the daemon
  itself is shutting down, and `unless-stopped` skips a container that carries it. `always`
  ignores the flag. `docker inspect` does not show it. The daemon keeps it in the container's
  `config.v2.json` under its data root, which only root can read, and `docker start` clears it.
  That file location comes from the engine source and was not checked on disk.
- **Check:** the flag also blocks restarts while the daemon is running, so you can test it
  without a reboot. Each container below ignores `HUP` and exits after four seconds.
  `--stop-signal TERM` matters: when the signal you send is the container's stop signal, which
  it is for any signal on an image that declares none, the kill also cancels restarts by a
  second route. That route would produce the same result without the flag.

  ```sh
  run() { docker run -d --name "$1" --restart "$2" --stop-signal TERM --network none \
    alpine:3.22 sh -c 'trap "" HUP; sleep 4; exit 1' >/dev/null; }
  run t-kill unless-stopped; run t-always always; run t-exec unless-stopped
  run t-control unless-stopped
  docker kill --signal=HUP t-kill >/dev/null
  docker kill --signal=HUP t-always >/dev/null
  docker exec t-exec kill -HUP 1
  timeout 60 sh -c 'until [ "$(docker inspect -f "{{.RestartCount}}" t-always t-exec t-control \
    | sort -n | head -1)" -ge 1 ]; do sleep 1; done'
  for c in t-kill t-always t-exec t-control; do
    docker inspect "$c" --format '{{.Name}} {{.State.Status}} restarts={{.RestartCount}}'
  done
  docker rm -f t-kill t-always t-exec t-control >/dev/null
  ```

  Measured output: `t-kill` reads `exited restarts=0`. The other three read `restarting` or
  `running`, depending on where the cycle is, with a restart count above zero. `t-always` got
  the same signal as `t-kill`, so the difference is the flag and not the signal. If `t-control`
  reads `restarts=0`, the wait timed out and the run proves nothing.
- **Recovery:** `docker start <container>` clears the flag. Search your runbooks and scripts for
  `docker kill -s`, `docker kill --signal`, `docker container kill` and `docker compose kill`.
  Change only the calls meant to leave the container running, such as a reload, to
  `docker exec <container> kill -<SIGNAL> 1`. That form needs a `kill` binary in the image. A
  call that is meant to stop the container should stay as it is, because the `exec` form lets
  `unless-stopped` start it again. A server with an HTTP reload endpoint, such as Prometheus
  with `--web.enable-lifecycle`, can use that instead. After any reboot, list containers that
  should be up and are not: `docker ps -a --filter status=exited`.
