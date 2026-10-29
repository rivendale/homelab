# Docker

Containers on a small home server, started by Docker Compose with a restart policy.

## A signal sent with docker kill stops the restart policy

**`docker kill --signal=<any>` marks the container as stopped by hand, so `unless-stopped`
does not start it again after it next exits, including after a reboot;
send a reload signal from inside with `docker exec <container> kill -HUP 1` instead.**

- **Failure (2026-10):** the metrics server stayed down after two reboots in a row while every
  other `unless-stopped` container came back. The daemon logged nothing for it: no start
  attempt and no error. Both times, the last thing done to it beforehand was the documented
  rule reload, `docker kill --signal=HUP <container>`. The signal only reloads the
  configuration and the container keeps running, so nothing looks wrong until the next boot.
  Alerting was dark until someone started it by hand.
- **Measured on:** Docker Engine 29.9.2 with `live-restore` on, 2026-10. Measured with the check
  below: a signal sent with `docker kill` keeps an `unless-stopped` container down after it
  exits, `always` restarts it anyway, and `docker start` on the stopped container clears the
  effect, so it restarts normally after its next exit. Measured separately, against a control
  container: on a container still running after such a kill, `docker start` exits 0 and clears
  nothing, while `docker restart` clears it. From the engine source and not measured
  here: the effect is the container's `HasBeenManuallyStopped` flag, it is not set while the daemon
  itself is shutting down, and it is kept in the container's `config.v2.json` under the
  daemon's data root. Measured: `docker inspect` does not show it, and on a rootful install
  the data root (`/var/lib/docker`) is readable only by root. Rootless Docker keeps it under
  the user's own home.
- **Check:** the flag also blocks restarts while the daemon is running, so you can test it
  without a reboot. Each container below ignores `HUP` and exits after twenty seconds.
  `--stop-signal TERM` matters. When a container has no stop signal configured
  (`docker inspect -f '{{.Config.StopSignal}}' <container>` prints nothing, as for this image),
  or the signal you send is its stop signal, the kill also cancels restarts by a second route.
  Measured for both: a `HUP` from `docker kill` kept even an `always` container down, with no
  stop signal set and with `--stop-signal HUP`.
  That route would produce the same result without the flag. The check runs in a subshell
  that removes its containers on exit and says so if any are left, so an interrupt, a kill or
  a dropped SSH session leaves nothing restarting. Run it in bash, where all of this was
  measured. The run prints its name prefix first. `kill -9`, a reboot or a power loss gets
  past the cleanup, and the restart policies then keep the containers cycling. Remove them with
  `docker rm -f <prefix>-kill <prefix>-always <prefix>-exec <prefix>-control`. If the prefix is
  lost, `docker ps -a --filter name=hupcheck-` lists every run's containers. Remove those while
  no other run of the check is in progress.

  ```sh
  (
  p="hupcheck-$(mktemp -u XXXXXXXX)"   # unique per run: cleanup touches only these four
  echo "prefix $p"
  trap 'trap "" INT TERM HUP
    for c in kill always exec control; do docker rm -f "$p-$c" >/dev/null 2>&1; done
    left=$(docker ps -aq --filter "name=^/?$p-") && [ -z "$left" ] \
      || echo "Cleanup failed: remove the $p containers." >&2' EXIT
  trap 'exit 130' INT TERM HUP
  run() { docker run -d --name "$p-$1" --restart "$2" --stop-signal TERM --network none \
    alpine:3.22 sh -c 'trap "" HUP; sleep 20; exit 1' >/dev/null; }
  run kill unless-stopped && run always always && run exec unless-stopped \
    && run control unless-stopped || { echo 'Could not create the test containers.' >&2; exit 1; }
  docker kill --signal=HUP "$p-kill" >/dev/null && docker kill --signal=HUP "$p-always" >/dev/null \
    && docker exec "$p-exec" kill -HUP 1 \
    && [ "$(docker inspect -f '{{.RestartCount}}' "$p-kill" "$p-always" "$p-exec" | sort -u)" = 0 ] \
    || echo 'A signal missed the first run of its container. This run proves nothing.'
  waitfor() { for _ in $(seq 60); do [ "$(docker inspect -f '{{.RestartCount}}' "$@" \
    | sort -n | head -1)" -ge 1 ] && return; sleep 1; done; return 1; }
  waitfor "$p-always" "$p-exec" "$p-control" \
    || echo 'The wait for the first restarts timed out. This run proves nothing.'
  for c in kill always exec control; do
    docker inspect "$p-$c" --format "$c {{.State.Status}} restarts={{.RestartCount}}"
  done
  docker start "$p-kill" >/dev/null && waitfor "$p-kill" \
    && docker inspect "$p-kill" --format 'kill after docker start: restarts={{.RestartCount}}' \
    || echo 'The docker start step did not finish. It proves nothing.'
  )
  ```

  Measured output: `kill` reads `exited restarts=0`. The other three read `restarting` or
  `running`, depending on where the cycle is, with a restart count above zero. `always` got the
  same signal as `kill`, so the difference is the flag and not the signal. The last line reads
  `kill after docker start: restarts=1`: after a `docker start`, the container restarts
  normally again. The run proves nothing if it printed a line saying a signal missed or a
  wait timed out.
- **Recovery:** on a stopped container, `docker start <container>` clears the flag. On one
  still running after such a kill, `docker start` exits 0 and changes nothing; use
  `docker restart <container>`, which clears it. Search your runbooks and scripts for
  `docker kill -s`, `docker kill --signal`, `docker container kill` and `docker compose kill`.
  Change only the calls meant to leave the container running, such as a reload, to
  `docker exec <container> kill -<SIGNAL> 1`. That form needs a `kill` binary in the image, and
  it reaches the service only when PID 1 is the service or an init that passes signals on.
  Docker's own `--init` does: measured, a `HUP` sent to it reached the service. A wrapper
  script usually does not. `docker exec <container> cat /proc/1/comm` names PID 1 without
  printing its arguments, which can hold a credential. If PID 1 is a wrapper, signal the
  service's own PID instead. Either way, confirm the reload where it lands, for example in the
  service's log or its reported configuration, rather than trusting the exit code. A
  call that is meant to stop the container should stay as it is, because the `exec` form lets
  `unless-stopped` start it again. A server with an HTTP reload endpoint, such as Prometheus
  with `--web.enable-lifecycle`, can use that instead. After any reboot, list exited containers
  whose restart policy should have brought them back:

  ```sh
  if ids=$(docker ps -aq --filter status=exited); then
    printf '%s\n' "$ids" | xargs -r docker inspect --format '{{.Name}} {{.HostConfig.RestartPolicy.Name}}' \
      | awk '$2 == "unless-stopped" || $2 == "always"'
  else
    echo 'Could not query Docker, so an empty list here proves nothing.' >&2; false
  fi
  ```

  Measured: it lists an `unless-stopped` container kept down this way, and leaves out a
  one-shot container with no restart policy that `docker ps -a --filter status=exited` alone
  would include. A container you stopped on purpose before the reboot is listed too, because
  nothing visible tells the two apart, so check each one before starting it.
