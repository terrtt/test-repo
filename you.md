# Production EKS Upgrade Runbook — 1.33 → 1.35

## 0. Critical framing before you start

**You cannot upgrade EKS directly from 1.33 to 1.35.** Amazon EKS only allows one minor-version step at a time on the control plane. This is a **two-stage upgrade**:

```
1.33 → 1.34 → 1.35
```

Each stage is its own full cycle: pre-checks → control plane → add-ons → node data plane → validation. Do not start stage 2 until stage 1 is fully validated and stable (recommend a minimum "bake" period of a few days to a week in production before moving on).

Treat this as **two separate maintenance windows**, not one.

---

## 1. Key considerations specific to this path

| Item | Detail |
|---|---|
| Sequential upgrade requirement | Control plane can only move one minor version at a time. No skipping. |
| Rollback window | EKS now supports **Kubernetes version rollback** within 7 days of a control-plane upgrade (one minor version back at a time). Good safety net, but don't rely on it as your only plan — some node/addon changes may not be cleanly reversible. |
| containerd 1.x EOL at 1.35 | **1.35 is the last EKS version supporting containerd 1.x.** You must be on containerd 2.0+ before you attempt the *next* upgrade past 1.35. Plan this now if you're on older AMIs. |
| cgroup v1 removed in 1.35 | Kubelet will refuse to start on cgroup v1 nodes by default in 1.35. AL2023/Bottlerocket default to cgroup v2 already; if you've manually forced cgroup v1 anywhere, fix that before the 1.34→1.35 step. |
| AL2 AMIs not available from 1.33+ | AWS stopped releasing EKS-optimized AL2 AMIs starting at 1.33. If any node groups are still on AL2, migrate to AL2023 as part of this project (should already be true since you're on 1.33, but verify — self-managed/custom AMIs may have been pinned). |
| `--pod-infra-container-image` kubelet flag removed in 1.35 | If you use **custom AMIs** with custom kubelet config, remove this flag before upgrading nodes to 1.35. |
| IPVS mode deprecated in kube-proxy | Not removed yet, but flagged for removal in 1.36. If you use `--proxy-mode=ipvs`, start planning a move to iptables or nftables. |
| Node/control-plane version skew | Kubelet can be up to 3 minor versions behind kube-apiserver (from 1.25+). Don't let node groups linger far behind — keep it tight in production. |
| kubectl version | Must be within 1 minor version of the cluster's Kubernetes version — update client tooling per stage. |

---

## 2. Pre-upgrade planning (do this for **each** stage: 1.33→1.34 and 1.34→1.35)

### 2.1 Inventory and compatibility check
1. Confirm current versions:
```bash
aws eks describe-cluster --name <cluster-name> \
  --query "cluster.{Version:version,PlatformVersion:platformVersion,Status:status}"
```
2. Confirm the target version is available and check support windows:
```bash
aws eks describe-cluster-versions \
  --query "clusterVersions[*].{Version:clusterVersion,Status:versionStatus,StandardSupportEnds:endOfStandardSupportDate,ExtendedSupportEnds:endOfExtendedSupportDate}" \
  --output table
```
3. Run **EKS Upgrade Insights** — this is the single most valuable pre-check AWS gives you. It scans 30 days of audit logs for deprecated API usage and flags add-on/version compatibility issues:
```bash
aws eks list-insights --cluster-name <cluster-name>
aws eks describe-insight --cluster-name <cluster-name> --id <insight-id>
```
   Or via console: Cluster → Upgrade Insights tab. **Do not skip this.** Fix flagged deprecated API usage before proceeding — note the 30-day audit window means a fix won't clear the warning immediately.

4. Cross-check deprecated/removed API usage independently with `kubent` (Kube No Trouble) or `pluto`:
```bash
kubent --target-version 1.34   # then rerun for 1.35 in stage 2
```

5. Review official version-specific release notes for breaking changes for **both** hops:
   - https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions-standard.html (covers 1.34 and 1.35 change notes)
   - Upstream Kubernetes CHANGELOG for 1.34 and 1.35

### 2.2 Add-on inventory (do this every stage — don't assume last stage's versions still apply)

List everything currently installed:
```bash
aws eks list-addons --cluster-name <cluster-name> --output table
```

For each add-on, check what's compatible with the **target** version:
```bash
aws eks describe-addon-versions \
  --addon-name vpc-cni \
  --kubernetes-version 1.34 \
  --query "addons[0].addonVersions[*].{Version:addonVersion,Default:compatibilities[0].defaultVersion}" \
  --output table
```
Repeat for each managed add-on you run, typically:
- `vpc-cni`
- `coredns`
- `kube-proxy`
- `aws-ebs-csi-driver`
- `aws-efs-csi-driver`
- `adot` (if used)
- `eks-pod-identity-agent` (if used)
- any others you've enabled (GuardDuty agent, Metrics Server via addon, etc.)

Also inventory **self-managed / Helm-installed** components that need separate compatibility review — these are *not* covered by `list-addons`:
- Cluster Autoscaler / Karpenter
- AWS Load Balancer Controller
- Ingress controllers (nginx, etc.) — note: **Ingress NGINX was retired in March 2026, no further security fixes** — if you're still on it, treat this upgrade as a forcing function to plan a Gateway API migration (Envoy Gateway or Cilium).
- Cert-manager
- Service mesh (Istio/App Mesh/Linkerd)
- Prometheus/Grafana stack, Fluent Bit/Fluentd
- Any custom admission webhooks/CRDs/operators

Check upstream compatibility matrices for each against the target Kubernetes version (most projects publish a support matrix in their docs/GitHub).

### 2.3 Cluster health & readiness
- Cluster is free of existing health issues: `aws eks describe-cluster` status healthy, no degraded nodegroups.
- Confirm at least 5 free IP addresses in the control-plane subnets (AWS requirement for the upgrade process).
- Review PodDisruptionBudgets — overly strict PDBs can stall node rotation during data-plane upgrade.
- Confirm etcd/control-plane logging is enabled (CloudWatch) so you have visibility during the upgrade:
```bash
aws eks update-cluster-config --name <cluster-name> \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
```

### 2.4 Backup
- Velero (or AWS Backup) snapshot of cluster resources and, if applicable, persistent volumes.
```bash
velero backup create pre-upgrade-1-34-backup --include-cluster-resources=true
```
- Export raw manifests as a secondary safety net:
```bash
kubectl get all --all-namespaces -o yaml > pre-upgrade-manifests-$(date +%F).yaml
```
- Confirm your IaC (Terraform/CloudFormation/CDK/eksctl config) reflects the *current* state so you're not upgrading out-of-band from your source of truth. Update the IaC version pin as part of the change, don't just click/CLI it live if you manage this cluster via IaC.

### 2.5 Staging/lower-environment validation
- Run the exact same upgrade path (1.33→1.34→1.35) against a staging/pre-prod cluster first.
- Run your full regression/integration test suite post-upgrade there.
- Time the actual upgrade duration in staging so you can set an accurate production maintenance window.

---

## 3. Execution — Stage 1: 1.33 → 1.34

### 3.1 Upgrade control plane
```bash
aws eks update-cluster-version \
  --name <cluster-name> \
  --kubernetes-version 1.34

# Monitor
aws eks describe-update --name <cluster-name> --update-id <UPDATE_ID>
```
This is AWS-managed and in-place — no workload downtime expected, though API server clients may need to reconnect briefly as API server instances are replaced (build retry logic into CI/CD and controllers where relevant).

### 3.2 Update add-ons to 1.34-compatible versions
```bash
aws eks update-addon --cluster-name <cluster-name> --addon-name vpc-cni \
  --addon-version <version> --resolve-conflicts OVERWRITE

aws eks update-addon --cluster-name <cluster-name> --addon-name coredns \
  --addon-version <version> --resolve-conflicts OVERWRITE

aws eks update-addon --cluster-name <cluster-name> --addon-name kube-proxy \
  --addon-version <version> --resolve-conflicts OVERWRITE

# repeat for ebs-csi, efs-csi, pod-identity-agent, adot, etc.
```
Check status:
```bash
aws eks describe-addon --cluster-name <cluster-name> --addon-name vpc-cni \
  --query "addon.{Name:addonName,Version:addonVersion,Status:status}"
```

### 3.3 Upgrade the data plane (nodes)
- **Managed node groups**: Console/CLI update triggers automatic rolling replacement respecting your configured max-unavailable / PDBs.
```bash
aws eks update-nodegroup-version --cluster-name <cluster-name> --nodegroup-name <ng-name>
```
- **Karpenter-provisioned nodes**: update the Karpenter-managed AMI/EC2NodeClass reference; Karpenter will drift-detect and roll nodes per your disruption budgets.
- **Self-managed node groups / ASGs**: update launch template AMI to the 1.34 EKS-optimized AMI, trigger instance refresh.
- **Fargate**: no manual node upgrade — pods pick up the new version on next restart/redeploy. If a Fargate pod is behind, delete/redeploy it explicitly before you touch the control plane if you want it in sync ahead of time (order matters less here than for managed nodes, but don't leave old Fargate pods running indefinitely).

### 3.4 Upgrade cluster-adjacent tooling
- Bump `kubectl` client to a 1.33/1.34-compatible version.
- Upgrade Helm-managed controllers (Cluster Autoscaler/Karpenter, ALB Controller, ingress, cert-manager, mesh, monitoring stack) to versions certified for 1.34.

### 3.5 Validate stage 1
- `kubectl get nodes -o wide` — confirm all nodes report 1.34.
- `kubectl get pods -A | grep -v Running` — check for anything crash-looping post-upgrade.
- Check controller/webhook logs for errors tied to API changes.
- Run your app-level smoke tests / synthetic checks.
- Watch CloudWatch Container Insights / your APM for anomalies (latency, error rate, OOMKills) for at least 24–48h before proceeding.
- Re-run Upgrade Insights against the now-1.34 cluster to get a clean read ahead of the 1.35 hop.

**Bake period recommended: several days minimum in production before Stage 2.**

---

## 4. Execution — Stage 2: 1.34 → 1.35

Repeat the full cycle above, targeting 1.35, **plus these 1.35-specific actions**:

### 4.1 Pre-checks specific to 1.35
- **cgroup v1 audit**: confirm no nodes are forced onto cgroup v1. AL2023/Bottlerocket default to v2; if you overrode this anywhere, either migrate the workload to tolerate v2 or explicitly set `failCgroupV1: false` in kubelet config as a stopgap — but plan the real migration, since this is a dead end at the next hop.
- **containerd version audit**: confirm your node AMIs are on containerd 2.x, or have a concrete plan to move off 1.x immediately after this upgrade (1.35 is the last version where 1.x works).
- **Custom AMI kubelet flags**: strip `--pod-infra-container-image` from kubelet config if you maintain custom AMIs/bootstrap scripts.
- **IPVS check**: if kube-proxy is running in IPVS mode, note the upcoming (1.36) removal and start migration planning now, even though it's not yet broken in 1.35.
- Windows node groups (if any): 1.35 adds Windows Server 2025 support — good moment to also confirm your Windows AMI strategy.

### 4.2 Control plane, add-ons, nodes, tooling
Same command patterns as Section 3, targeting `--kubernetes-version 1.35` and 1.35-compatible add-on versions.

### 4.3 Validate stage 2
Same validation steps as 3.5, targeting 1.35.

---

## 5. Rollback plan

- Within **7 days** of a control-plane version bump, you can roll back one minor version:
```bash
# check readiness/insights first — console surfaces "Rollback insights"
aws eks rollback-cluster-version --name <cluster-name> --kubernetes-version 1.33
```
- Node-level rollback via version rollback is only automatic for **EKS Auto Mode**; for standard managed node groups/self-managed/Karpenter you'll need to manually roll nodes back to prior AMIs if you go this route.
- After 7 days, rollback is not available — recovery path becomes "stand up a new cluster on the prior version and migrate workloads" (blue/green), which is why staging validation and a real bake period between stages matters so much.
- Have your blue/green migration runbook (Velero restore into a parallel cluster) ready as the deeper fallback regardless.

---

## 6. Post-upgrade checklist (after both stages complete)

- [ ] Both control plane and all node groups report 1.35
- [ ] All managed add-ons on 1.35-compatible versions, status `ACTIVE`
- [ ] All Helm-managed controllers on certified-compatible versions
- [ ] No deprecated API usage flagged by Upgrade Insights / kubent
- [ ] containerd 2.x confirmed on all nodes (since 1.35 was the last hop supporting 1.x)
- [ ] No cgroup v1 dependencies remaining
- [ ] IPVS migration plan documented if still in use
- [ ] IaC updated and applied (not just live-patched) to reflect 1.35 + new add-on versions
- [ ] Monitoring/alerting baseline re-confirmed normal for 48–72h
- [ ] Runbook/docs updated with lessons learned for the next cycle (1.35 exits standard support in ~14 months — put that date on the calendar now)

---

## 7. Suggested timeline for a production cluster

| Phase | Duration |
|---|---|
| Inventory, insights, staging test — Stage 1 | 1–2 weeks |
| Stage 1 execution + bake | Maintenance window + 3–7 days bake |
| Inventory, insights, staging test — Stage 2 | 1 week (much of Stage 1's prep is reusable) |
| Stage 2 execution + bake | Maintenance window + 3–7 days bake |

Total realistic window for a careful production upgrade: **~4–6 weeks** end to end, not counting emergency buffer.
