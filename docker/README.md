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
- **Measured on:** Docker Engine 29.9.2 with `live-restore` on, 2026-10. Measured with the check
  below: a signal sent with `docker kill` keeps an `unless-stopped` container down after it
  exits, `always` restarts it anyway, and a later `docker start` clears the effect, so the
  container restarts normally after its next exit. From the engine source and not measured
  here: the effect is the container's `HasBeenManuallyStopped` flag, it is not set while the
  daemon itself is shutting down, and it is kept in the container's `config.v2.json` under the
  daemon's data root, which only root can read. Measured: `docker inspect` does not show it.
- **Check:** the flag also blocks restarts while the daemon is running, so you can test it
  without a reboot. Each container below ignores `HUP` and exits after twenty seconds.
  `--stop-signal TERM` matters: when the signal you send is the container's stop signal, which
  it is for any signal on an image that declares none, the kill also cancels restarts by a
  second route. That route would produce the same result without the flag.

  ```sh
  p="hupcheck-$$"   # unique names, so the cleanup cannot touch a container of yours
  run() { docker run -d --name "$p-$1" --restart "$2" --stop-signal TERM --network none \
    alpine:3.22 sh -c 'trap "" HUP; sleep 20; exit 1' >/dev/null; }
  run kill unless-stopped; run always always; run exec unless-stopped; run control unless-stopped
  docker kill --signal=HUP "$p-kill" >/dev/null && docker kill --signal=HUP "$p-always" >/dev/null \
    && docker exec "$p-exec" kill -HUP 1 \
    && [ "$(docker inspect -f '{{.RestartCount}}' "$p-kill" "$p-always" "$p-exec" | sort -u)" = 0 ] \
    || echo 'A signal missed the first run of its container. This run proves nothing.'
  waitfor() { timeout 60 sh -c 'until [ "$(docker inspect -f "{{.RestartCount}}" "$@" \
    | sort -n | head -1)" -ge 1 ]; do sleep 1; done' _ "$@"; }
  waitfor "$p-always" "$p-exec" "$p-control"
  for c in kill always exec control; do
    docker inspect "$p-$c" --format "$c {{.State.Status}} restarts={{.RestartCount}}"
  done
  docker start "$p-kill" >/dev/null && waitfor "$p-kill"
  docker inspect "$p-kill" --format 'kill after docker start: restarts={{.RestartCount}}'
  docker rm -f "$p-kill" "$p-always" "$p-exec" "$p-control" >/dev/null
  ```

  Measured output: `kill` reads `exited restarts=0`. The other three read `restarting` or
  `running`, depending on where the cycle is, with a restart count above zero. `always` got the
  same signal as `kill`, so the difference is the flag and not the signal. The last line reads
  `kill after docker start: restarts=1`: after a `docker start`, the container restarts
  normally again. If `control` reads `restarts=0`, the wait timed out and the run proves
  nothing. The same is true if the run printed the line saying a signal missed.
- **Recovery:** `docker start <container>` clears the flag. Search your runbooks and scripts for
  `docker kill -s`, `docker kill --signal`, `docker container kill` and `docker compose kill`.
  Change only the calls meant to leave the container running, such as a reload, to
  `docker exec <container> kill -<SIGNAL> 1`. That form needs a `kill` binary in the image. A
  call that is meant to stop the container should stay as it is, because the `exec` form lets
  `unless-stopped` start it again. A server with an HTTP reload endpoint, such as Prometheus
  with `--web.enable-lifecycle`, can use that instead. After any reboot, list exited containers
  whose restart policy should have brought them back:

  ```sh
  docker ps -a --filter status=exited -q \
    | xargs -r docker inspect --format '{{.Name}} {{.HostConfig.RestartPolicy.Name}}' \
    | awk '$2 == "unless-stopped" || $2 == "always"'
  ```

  Measured: it lists an `unless-stopped` container kept down this way, and leaves out a
  one-shot container with no restart policy that `docker ps -a --filter status=exited` alone
  would include. A container you stopped on purpose before the reboot is listed too, because
  nothing visible tells the two apart, so check each one before starting it.
