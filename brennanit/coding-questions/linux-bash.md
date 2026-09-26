# Linux & Bash Automation

## 1. Common operational tasks you'd script

- Rotate/compress logs older than N days.
- Check service health endpoints and alert on failure.
- Dump and restore DB for local reproduction.
- Sync configuration from Key Vault to local env file (dev only).
- Run evaluation harness and post results to dashboard.
- Trigger CI pipeline via API for a branch.

## 2. Bash patterns for robust scripts

```bash
#!/usr/bin/env bash
set -euo pipefail  # exit on error, unset var, pipe failure

log() { echo "[$(date -Is)] $*"; }

main() {
  local input_file="${1:-}"
  [[ -f "$input_file" ]] || { log "ERROR: file not found: $input_file"; exit 1; }
  
  while IFS= read -r line; do
    process_item "$line" || log "WARN: failed: $line"
  done < "$input_file"
}

process_item() {
  local item="$1"
  # idempotent operation
  do_work "$item"
}

main "$@"
```

**Principles:** `set -euo pipefail`, functions, local variables, explicit error handling, idempotent operations, log with timestamps.

## 3. Inspecting a running container / service

- `docker logs -f <container>` — follow logs.
- `docker exec -it <container> bash` — interactive shell.
- `kubectl logs -f <pod> -c <container>` — K8s logs.
- `kubectl exec -it <pod> -- bash` — K8s shell.
- `curl localhost:8080/health` — health endpoint.
- `netstat -tlnp` or `ss -tlnp` — listening ports.
- `ps auxf` — process tree.

## 4. Automating a CI/CD step in Bash

```bash
#!/usr/bin/env bash
set -euo pipefail

# Trigger pipeline and wait
run_id=$(curl -s -X POST "$CI_API/pipelines" -d "ref=$BRANCH" | jq -r .id)
log "Triggered pipeline $run_id"

while true; do
  status=$(curl -s "$CI_API/pipelines/$run_id" | jq -r .status)
  [[ "$status" == "success" ]] && { log "Pipeline passed"; exit 0; }
  [[ "$status" == "failed" ]] && { log "Pipeline failed"; exit 1; }
  sleep 30
done
```

## 5. File and data processing one-liners

- `jq '.items[] | select(.status=="failed") | .id' file.json` — extract failed IDs.
- `awk -F, 'NR>1 && $3>100 {print $1}' data.csv` — filter CSV.
- `find /var/log -name "*.log" -mtime +7 -delete` — cleanup old logs.
- `grep -r "ERROR" /var/log/app/ --include="*.log" | wc -l` — count errors.