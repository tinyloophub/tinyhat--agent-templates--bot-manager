---
name: probe-network
description: Diagnostics-only. Probes sandbox network (env, DNS, multi-host curl) and returns the raw report. Use ONLY when the user asks to debug network or curl(7) failures.
runtime_tier: 1
runtime_requires:
  - "curl"
---

# probe-network

This skill exists for issue #62 diagnostics. It is **not** a
user-facing capability — only invoke it when the user explicitly
asks to probe the network or debug `curl(7)` / `curl(6)`.

## What to do

Run **one shell command** that gathers a comprehensive report on the
sandbox's networking state and tests outbound connectivity against
a fixed list of hosts (a mix of our platform host, GitHub-org hosts
which the OpenAI org allowlist also includes, and one definitely-
unrelated host as a control).

```sh
. /home/oai/skills/session-context-*/scripts/env.sh \
  && {
       echo "=== probe-network v2 ==="
       echo "--- proxy env vars ---"
       echo "HTTP_PROXY=${HTTP_PROXY:-<unset>}"
       echo "HTTPS_PROXY=${HTTPS_PROXY:-<unset>}"
       echo "http_proxy=${http_proxy:-<unset>}"
       echo "https_proxy=${https_proxy:-<unset>}"
       echo "NO_PROXY=${NO_PROXY:-<unset>}"
       echo "no_proxy=${no_proxy:-<unset>}"
       echo "--- /etc/resolv.conf ---"
       cat /etc/resolv.conf 2>&1 || echo "(no resolv.conf)"
       echo "--- /etc/hosts ---"
       cat /etc/hosts 2>&1 || echo "(no /etc/hosts)"
       echo "--- ip route ---"
       (ip route 2>&1 || route -n 2>&1) | head -20
       echo "--- ip addr (interfaces) ---"
       (ip -br addr 2>&1 || ifconfig -a 2>&1) | head -30
       echo "--- container metadata ---"
       echo "uname=$(uname -a 2>&1)"
       echo "container_id=$(hostname 2>&1)"
       echo "agents_sdk=${OPENAI_AGENTS_SDK_VERSION:-<unset>}"
       echo "platform=${TINYLOOP_API_BASE_URL:-<unset>}"
       echo "--- DNS reachability ---"
       for srv in 8.8.8.8 1.1.1.1 169.254.169.254; do
         echo -n "ping ${srv}: "
         (timeout 1 sh -c "echo > /dev/tcp/${srv}/53" 2>&1 \
           && echo "tcp/53 open") || echo "blocked"
       done
       echo "--- per-host probes ---"
       for url in \
         "$TINYLOOP_API_BASE_URL/" \
         "https://api.tinyloop.co/" \
         "https://github.com/" \
         "https://api.github.com/" \
         "https://raw.githubusercontent.com/" \
         "https://example.com/"; do
         host=$(echo "$url" | awk -F/ '{print $3}')
         echo "--- url=$url ---"
         echo -n "getent_hosts: "
         (getent hosts "$host" 2>&1 | head -1) || echo "exit=$?"
         echo -n "curl_proxy_honoured: "
         curl -sS -o /dev/null \
           -w 'http_code=%{http_code} time=%{time_total}\n' \
           --max-time 5 --head "$url" 2>&1 \
           | tr '\n' ' '
         echo " exit=$?"
         echo -n "curl_proxy_bypassed: "
         curl -sS --noproxy '*' -o /dev/null \
           -w 'http_code=%{http_code} time=%{time_total}\n' \
           --max-time 5 --head "$url" 2>&1 \
           | tr '\n' ' '
         echo " exit=$?"
       done
       echo "=== end probe-network v2 ==="
     }
```

## What to put in your final assistant message

Return the **literal stdout** of the probe block above as your
final assistant message. Do not paraphrase, summarise, or add
commentary. The harness reads `final_output` and the operator
expects to see the raw probe text verbatim.

The block is meant to be self-contained and machine-readable —
each section is delimited by `--- <name> ---` lines. Keep that
structure intact.
