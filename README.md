# EKS Pre-Upgrade Inventory & Compatibility Report

- Cluster: **dev**
- Target Kubernetes version: **1.34**
- Generated: 2026-07-15T07:52:00Z

## 1. Cluster Basics

- Current Kubernetes version: **1.33**
- Platform version: eks.39
- Cluster status: ACTIVE

## 2. EKS-Managed Add-ons

| Add-on | Installed Version | Status | Latest Compatible w/ 1.34 | Action Needed |
|---|---|---|---|---|
| aws-ebs-csi-driver | v1.51.1-eksbuild.1 | ACTIVE | v1.62.0-eksbuild.1 | 🔁 Update to v1.62.0-eksbuild.1 |
| coredns | v1.11.3-eksbuild.2 | ACTIVE | v1.12.4-eksbuild.17 | 🔁 Update to v1.12.4-eksbuild.17 |
| eks-pod-identity-agent | v1.3.8-eksbuild.2 | ACTIVE | v1.3.10-eksbuild.3 | 🔁 Update to v1.3.10-eksbuild.3 |
| kube-proxy | v1.33.3-eksbuild.4 | ACTIVE | v1.34.6-eksbuild.11 | 🔁 Update to v1.34.6-eksbuild.11 |
| vpc-cni | v1.18.2-eksbuild.1 | ACTIVE | v1.21.2-eksbuild.2 | 🔁 Update to v1.21.2-eksbuild.2 |

> Note: only add-ons registered via `aws eks list-addons` show here. Anything installed via Helm/manifests (CNI plugins beyond vpc-cni, ingress controllers, autoscalers, mesh, monitoring, CSI drivers not managed as EKS add-ons, etc.) is covered in the sections below and must be checked against its own upstream compatibility matrix manually.

## 3. Nodes

| Node | Kubelet Version | Container Runtime | OS Image |
|---|---|---|---|
| ip-10-11-52-198.ap-southeast-1.compute.internal | v1.33.13-eks-8f14419 | containerd://2.2.4+unknown | Amazon Linux 2023.12.20260611 |
| ip-10-11-52-208.ap-southeast-1.compute.internal | v1.33.13-eks-8f14419 | containerd://2.2.4+unknown | Amazon Linux 2023.12.20260611 |
| ip-10-11-72-107.ap-southeast-1.compute.internal | v1.33.8-eks-f69f56f | containerd://2.2.1+unknown | Amazon Linux 2023.10.20260302 |
| ip-10-11-73-137.ap-southeast-1.compute.internal | v1.33.13-eks-8f14419 | containerd://2.2.4+unknown | Amazon Linux 2023.12.20260611 |
| ip-10-11-77-44.ap-southeast-1.compute.internal | v1.33.13-eks-8f14419 | containerd://2.2.4+unknown | Amazon Linux 2023.12.20260611 |

> ✅ No nodes detected on containerd 1.x.

## 4. Helm-Installed Components

| Namespace | Release | Chart | App Version | Status |
|---|---|---|---|---|
| argocd | argocd | argo-cd-8.5.8 | v3.1.8 | deployed |
| kube-system | aws-load-balancer-controller | aws-load-balancer-controller-1.14.0 | v2.14.0 | deployed |
| keycloak | corp-keycloak | keycloakx-7.1.9 | 26.5.5 | deployed |
| kube-system | csi-secrets-store | secrets-store-csi-driver-1.5.6 | 1.5.6 | deployed |
| keycloak | dev-keycloak-postgresql | postgresql-18.5.15 | 18.3.0 | deployed |
| drdroid | drd-vpc-agent | drd-vpc-agent-0.1.2 |  | deployed |
| karpenter | karpenter | karpenter-1.7.1 | 1.7.1 | deployed |
| kube-system | karpenter-crd | karpenter-crd-1.7.1 | 1.7.1 | deployed |
| otterize-system | network-mapper | network-mapper-3.0.47 | v3.0.19 | deployed |
| signoz | signoz | signoz-0.99.0 | v0.99.0 | deployed |
| signoz | signoz-agent | k8s-infra-0.14.1 | 0.109.0 | deployed |

> Cross-check each chart version against its project's compatibility matrix for Kubernetes 1.34 (e.g. AWS Load Balancer Controller, Karpenter, ingress-nginx, cert-manager, Prometheus stack, service mesh).

## 5. Workloads

| Namespace | Deployments | StatefulSets | DaemonSets |
|---|---|---|---|
| argocd | 4 | 1 | 0 |
| common | 4 | 0 | 0 |
| corporate | 12 | 0 | 0 |
| drdroid | 3 | 0 | 0 |
| karpenter | 1 | 0 | 0 |
| keycloak | 0 | 2 | 0 |
| kube-system | 5 | 0 | 7 |
| medical | 11 | 0 | 0 |
| otterize-system | 1 | 0 | 1 |
| platform | 1 | 0 | 0 |
| signoz | 3 | 3 | 1 |

## 6. Unique Container Images In Use

```
602401143452.dkr.ecr.ap-southeast-1.amazonaws.com/amazon-k8s-cni:v1.18.2-eksbuild.1 602401143452.dkr.ecr.ap-southeast-1.amazonaws.com/amazon/aws-network-policy-agent:v1.1.2-eksbuild.1
602401143452.dkr.ecr.ap-southeast-1.amazonaws.com/eks/aws-ebs-csi-driver:v1.51.1 602401143452.dkr.ecr.ap-southeast-1.amazonaws.com/eks/csi-node-driver-registrar:v2.14.0-eksbuild.5 602401143452.dkr.ecr.ap-southeast-1.amazonaws.com/eks/livenessprobe:v2.16.0-eksbuild.5
602401143452.dkr.ecr.ap-southeast-1.amazonaws.com/eks/aws-ebs-csi-driver:v1.51.1 602401143452.dkr.ecr.ap-southeast-1.amazonaws.com/eks/csi-provisioner:v5.3.0-eksbuild.4 602401143452.dkr.ecr.ap-southeast-1.amazonaws.com/eks/csi-attacher:v4.9.0-eksbuild.4 602401143452.dkr.ecr.ap-southeast-1.amazonaws.com/eks/csi-snapshotter:v8.3.0-eksbuild.2 602401143452.dkr.ecr.ap-southeast-1.amazonaws.com/eks/csi-resizer:v1.14.0-eksbuild.4 602401143452.dkr.ecr.ap-southeast-1.amazonaws.com/eks/livenessprobe:v2.16.0-eksbuild.5
602401143452.dkr.ecr.ap-southeast-1.amazonaws.com/eks/coredns:v1.11.3-eksbuild.2
602401143452.dkr.ecr.ap-southeast-1.amazonaws.com/eks/eks-pod-identity-agent:v0.1.31
602401143452.dkr.ecr.ap-southeast-1.amazonaws.com/eks/kube-proxy:v1.33.3-eksbuild.4
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/eks-dashboard:latest
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/common/nw-chart-service:2.0.0-alpha02
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/common/nw-email-worker:2.15.2
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/common/nw-image-service:1.8.0
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/common/nw-report-worker:2.27.4
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/corporate/b2c-events-app:0.1.3
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/corporate/b2c-events-service:0.1.2
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/corporate/emc-dragonfly-service:0.3.0
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/corporate/emc-dragonfly-service:0.3.3
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/corporate/fullerton-dragonfly-service:0.1.1
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/corporate/nw-apollo-dragon-service:0.19.6
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/corporate/nw-dragonfly-app:1.10.0-dev01
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/corporate/nw-dragonfly-service:1.7.4-alpha01
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/corporate/one-medic-dragonfly-service:0.1.3
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/corporate/one-medic-vietnam-dragonfly-service:0.3.2
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/corporate/pmx-health-dragonfly-service:0.0.8
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/corporate/prodia-dragonfly-service:0.0.6
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/medical/cx-cognifyx-core:4.4.4
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/medical/cx-nhyre-server:2.2.3
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/medical/cx-nhyre-web:3.14.3
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/medical/cx-nhyre-web:3.14.6
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/medical/nw-admin-portal-service:1.2.3
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/medical/nw-apollo-streamlit:0.0.5
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/medical/nw-cygnus-app:0.19.3-dev01
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/medical/nw-nbi-app:0.1.3
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/medical/nw-nbi-service:0.0.19
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/medical/nw-nn001-app:1.0.0
614005182939.dkr.ecr.ap-southeast-1.amazonaws.com/nw/medical/nw-pulse-app:0.0.6
bitnami/kubectl:latest
docker.io/altinity/clickhouse-operator:0.21.2 docker.io/altinity/metrics-exporter:0.21.2
docker.io/clickhouse/clickhouse-server:25.5.6
docker.io/otel/opentelemetry-collector-contrib:0.109.0
docker.io/signoz/signoz-otel-collector:v0.129.8
docker.io/signoz/signoz:v0.99.0
docker.io/signoz/zookeeper:3.7.1
drdroidlab/drd-vpc-agent:latest
drdroidlab/drd-vpc-agent:latest drdroidlab/drd-vpc-agent:latest drdroidlab/drd-vpc-agent:latest
drdroidlab/otterize-network-mapper:droid-latest-v2
drdroidlab/otterize-sniffer:droid-latest-v2
ecr-public.aws.com/docker/library/redis:7.2.8-alpine
public.ecr.aws/aws-secrets-manager/secrets-store-csi-driver-provider-aws:2.1.0
public.ecr.aws/eks/aws-load-balancer-controller:v2.14.0
public.ecr.aws/karpenter/controller:1.7.1@sha256:426c3af05c1dca65454663262a82bec57c74f5d69eec157067166738eab8436e
quay.io/argoproj/argocd:v3.1.8
quay.io/keycloak/keycloak:26.5.6
redis:7-alpine
registry-1.docker.io/bitnami/postgresql:latest
registry.k8s.io/metrics-server/metrics-server:v0.8.1
registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.13.0 registry.k8s.io/csi-secrets-store/driver:v1.5.4 registry.k8s.io/sig-storage/livenessprobe:v2.15.0
```

./eks-inventory-dev-2026-07-15_132200/images-full-list.txt written separately for grepping.

## 7. Custom Resource Definitions

```
applicationnetworkpolicies.networking.k8s.aws
applications.argoproj.io
applicationsets.argoproj.io
appprojects.argoproj.io
clickhouseinstallations.clickhouse.altinity.com
clickhouseinstallationtemplates.clickhouse.altinity.com
clickhouseoperatorconfigurations.clickhouse.altinity.com
clusternetworkpolicies.networking.k8s.aws
clusterpolicyendpoints.networking.k8s.aws
cninodes.vpcresources.k8s.aws
ec2nodeclasses.karpenter.k8s.aws
eniconfigs.crd.k8s.amazonaws.com
ingressclassparams.elbv2.k8s.aws
listenerruleconfigurations.gateway.k8s.aws
loadbalancerconfigurations.gateway.k8s.aws
nodeclaims.karpenter.sh
nodeoverlays.karpenter.sh
nodepools.karpenter.sh
policyendpoints.networking.k8s.aws
secretproviderclasses.secrets-store.csi.x-k8s.io
secretproviderclasspodstatuses.secrets-store.csi.x-k8s.io
securitygrouppolicies.vpcresources.k8s.aws
targetgroupbindings.elbv2.k8s.aws
targetgroupconfigurations.gateway.k8s.aws
```

## 8. Admission Webhooks

**Mutating:**
```
0500-amazon-eks-fargate-mutation.amazonaws.com
aws-load-balancer-webhook
pod-identity-webhook
vpc-resource-mutating-webhook
```
**Validating:**
```
aws-load-balancer-webhook
vpc-resource-validating-webhook
```

## 9. Deprecated/Removed API Usage

> Neither `kubent` nor `pluto` found. Install one of them and rerun this check, or use EKS Upgrade Insights below (recommended anyway).
>
> Install kubent: `curl -sSL https://git.io/install-kubent | sh`

## 10. EKS Upgrade Insights (AWS-native)

**Insight: 57145b86-fd6d-4078-8ace-2387f5acf22c**
```
-------------------------------------------------------------------------------
|                               DescribeInsight                               |
+----------------------+-----------------------+------------------------------+
|       Category       |   KubernetesVersion   |            Name              |
+----------------------+-----------------------+------------------------------+
|  UPGRADE_READINESS   |  1.34                 |  kube-proxy version skew     |
+----------------------+-----------------------+------------------------------+
||                                  Status                                   ||
|+----------------------------------------------------------------+----------+|
||                             reason                             | status   ||
|+----------------------------------------------------------------+----------+|
||  kube-proxy versions match the cluster control plane version.  |  PASSING ||
|+----------------------------------------------------------------+----------+|
```
**Insight: cf157532-a0b0-4a9f-9a50-d359d7b2bf52**
```
------------------------------------------------------------------------------------------------
|                                        DescribeInsight                                       |
+---------------------------------+------------------------------------------------------------+
|  Category                       |  UPGRADE_READINESS                                         |
|  KubernetesVersion              |  1.34                                                      |
|  Name                           |  EKS add-on version compatibility                          |
+---------------------------------+------------------------------------------------------------+
||                                           Status                                           ||
|+--------+-----------------------------------------------------------------------------------+|
||  reason|  All installed EKS add-on versions are compatible with next Kubernetes version.   ||
||  status|  PASSING                                                                          ||
|+--------+-----------------------------------------------------------------------------------+|
```
**Insight: 4ebb4afb-246a-4fca-a6e6-ddbab708df44**
```
----------------------------------------------------------------------------
|                              DescribeInsight                             |
+-------------------+---------------------+--------------------------------+
|     Category      |  KubernetesVersion  |             Name               |
+-------------------+---------------------+--------------------------------+
|  UPGRADE_READINESS|  1.34               |  Amazon Linux 2 compatibility  |
+-------------------+---------------------+--------------------------------+
||                                 Status                                 ||
|+-------------------------------------------------------+----------------+|
||                        reason                         |    status      ||
|+-------------------------------------------------------+----------------+|
||  No Amazon Linux 2 nodes detected.                    |  PASSING       ||
|+-------------------------------------------------------+----------------+|
```
**Insight: ffd98726-8ff1-4154-a423-a4075636519e**
```
--------------------------------------------------------------------------------
|                                DescribeInsight                               |
+-----------------------------------+------------------------------------------+
|  Category                         |  UPGRADE_READINESS                       |
|  KubernetesVersion                |  1.34                                    |
|  Name                             |  Kubelet version skew                    |
+-----------------------------------+------------------------------------------+
||                                   Status                                   ||
|+--------+-------------------------------------------------------------------+|
||  reason|  Node kubelet versions match the cluster control plane version.   ||
||  status|  PASSING                                                          ||
|+--------+-------------------------------------------------------------------+|
```
**Insight: b1c77c5e-3050-4175-8d62-622d867eb7da**
```
---------------------------------------------------------------------
|                          DescribeInsight                          |
+-------------------+---------------------+-------------------------+
|     Category      |  KubernetesVersion  |          Name           |
+-------------------+---------------------+-------------------------+
|  UPGRADE_READINESS|  1.34               |  Cluster health issues  |
+-------------------+---------------------+-------------------------+
||                             Status                              ||
|+--------------------------------------------------+--------------+|
||                      reason                      |   status     ||
|+--------------------------------------------------+--------------+|
||  No cluster health issues detected.              |  PASSING     ||
|+--------------------------------------------------+--------------+|
```
