# Whalekeep agent

The Whalekeep agent reads what actually runs on your Kubernetes clusters and Linux hosts and reports it to a Whalekeep server, which matches exact versions against distro and upstream advisories.

- **Read-only.** The cluster agent only lists workloads, nodes, services, ingresses and routes. It never reads Secrets or ConfigMaps. The host agent reads the package databases (dpkg, apk, rpm), the running kernel and, if Docker is present, the running containers.
- **No image pulls.** The agent reports image digests. The server scans the images.
- **Push only.** The agent opens HTTPS connections to your Whalekeep server. Nothing connects to the agent.

Releases: [GitHub releases](https://github.com/ipp-net/whalekeep-agent/releases). Image: `ghcr.io/ipp-net/whalekeep-agent`. Helm chart: `oci://ghcr.io/ipp-net/charts/whalekeep-agent`.

## Enrolment

Each agent enrols once with a single-use enrolment token from your Whalekeep server administrator. A token is for one kind of agent, `host` or `cluster`, and expires after 24 hours by default. Enrolment exchanges the token for an agent ID and a per-agent secret, stored in a credential file readable only by its owner:

```
ivt-agent enroll --server https://whalekeep.example.com --kind host --token-file token.txt --credential /etc/ivt-agent/credential
```

Revoking an agent on the server makes its next report fail with `401`, and the agent exits.

## Linux hosts

Static binaries for `linux/amd64` and `linux/arm64`. They have no dependencies and run on any distribution with systemd.

```
VERSION=v0.1.0
ARCH=amd64
curl -fsSLO https://github.com/ipp-net/whalekeep-agent/releases/download/$VERSION/whalekeep-agent_${VERSION}_linux_${ARCH}.tar.gz
curl -fsSLO https://github.com/ipp-net/whalekeep-agent/releases/download/$VERSION/SHA256SUMS
sha256sum --ignore-missing -c SHA256SUMS
tar -xzf whalekeep-agent_${VERSION}_linux_${ARCH}.tar.gz
cd whalekeep-agent_${VERSION}_linux_${ARCH}

sudo install -m 0755 ivt-agent /usr/local/bin/ivt-agent
sudo install -d -m 0700 /etc/ivt-agent
sudo ivt-agent enroll --server https://whalekeep.example.com --kind host --token-file token.txt --credential /etc/ivt-agent/credential

sudo install -m 0644 ivt-agent.service ivt-agent.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now ivt-agent.timer
sudo systemctl start ivt-agent.service
journalctl -u ivt-agent.service -n 20
```

The timer runs the agent 5 minutes after boot and then every 6 hours. The service runs with a read-only view of the system and only the capability to read files. To report once by hand:

```
sudo ivt-agent host --credential /etc/ivt-agent/credential
```

The host is reported under its hostname; `--host-name` overrides it. `--docker-socket ""` turns off container collection.

## Kubernetes

1. Enrol a `cluster` agent on any machine. macOS builds are in the release for this step:

   ```
   ivt-agent enroll --server https://whalekeep.example.com --kind cluster --token-file token.txt --credential credential.json
   ```

2. Store the credential in a Secret. Its keys match the credential file:

   ```
   kubectl create namespace whalekeep
   jq -r 'to_entries[] | "\(.key)=\(.value)"' credential.json > credential.env
   kubectl -n whalekeep create secret generic whalekeep-agent-credential --from-env-file=credential.env
   rm credential.env credential.json
   ```

3. Install the chart. `clusterName` must be unique within your tenant:

   ```
   helm install whalekeep-agent oci://ghcr.io/ipp-net/charts/whalekeep-agent --version 0.1.0 \
     --namespace whalekeep --set clusterName=prod-eu
   ```

With Flux:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: whalekeep-agent
  namespace: whalekeep
spec:
  interval: 1h
  url: oci://ghcr.io/ipp-net/charts/whalekeep-agent
  ref:
    semver: "0.1.x"
  layerSelector:
    mediaType: application/vnd.cncf.helm.chart.content.v1.tar+gzip
    operation: copy
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: whalekeep-agent
  namespace: whalekeep
spec:
  interval: 1h
  chartRef:
    kind: OCIRepository
    name: whalekeep-agent
  values:
    clusterName: prod-eu
```

Chart values:

| Value | Default | Meaning |
|---|---|---|
| `clusterName` | required | Cluster name, unique within the tenant |
| `interval` | `6h` | Time between reports |
| `excludeNamespaces` | empty | Label selector of namespaces to skip, for example `team=sandbox` |
| `credential.existingSecret` | `whalekeep-agent-credential` | Secret holding `server`, `agent_id`, `secret` and `tenant` |
| `image.repository`, `image.tag` | `ghcr.io/ipp-net/whalekeep-agent`, the chart's version | Agent image |
| `resources` | 50m CPU and 64Mi requested, 256Mi limit | Container resources |
| `nodeSelector`, `tolerations`, `affinity`, `priorityClassName` | empty | Scheduling |

The agent runs as a non-root user with a read-only root filesystem and no capabilities. Its ClusterRole grants `get` and `list` on pods, nodes, namespaces, services, workload controllers, jobs, ingresses, endpoint slices, Gateway API and Traefik routes, and `/version`.

## License

Apache License 2.0. See [LICENSE](LICENSE).
