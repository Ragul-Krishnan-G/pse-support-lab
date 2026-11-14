# API restart observations (runtime evidence supplied in chat)

- `docker inspect invoicely-lite-api-1`: `Restarts=2`, `Status=running`, `OOMKilled=false`, `ExitCode=0`.
- `docker compose logs --since=2h api`: startup messages appeared three times in the sampled window.
- Filter for `error|fatal|panic|restart|shutdown|signal|killed` returned no matches in the sampled API logs.
- `docker stats --no-stream`: API 90.33 MiB / 128 MiB (70.57%) at the time of the snapshot.
- `curl -i http://localhost:8080/health` returned 404. This is an invalid path for this service; the implemented health route is `/healthz` in `services/api/internal/httpapi/httpapi.go`.
- Limitation: local container timestamps were on the actual lab machine clock, not the fictional exercise clock. These observations do not establish the cause of the 09:42 exercise alert.
