# k3s One-Node Upgrade Runbook

This runbook upgrades k3s one node at a time using `upgrade-k3s-one-node.yml`.
It is intentionally narrower than `site.yml`: it only replaces `/usr/local/bin/k3s`,
restarts the local k3s service when the binary changes, and verifies the local target version.

## Scope

Use this for k3s patch upgrades where the inventory `k3s_version` is already set to the desired target.
Do not use it for Calico, MetalLB, kube-vip, node OS upgrades, cluster rebuilds, or reset operations.

The playbook requires `--limit` to select exactly one host. It refuses to run against zero, multiple, or all hosts.
The one-host guard runs before fact gathering, so an accidental unbounded run fails before connecting to every node.

## Preflight

Export the SSH agent used for node access before running Ansible:

```bash
export SSH_AUTH_SOCK=/tmp/ssh-BatCCCSiZa0r/agent.10452
export SSH_AGENT_PID=10453
ssh-add -l
```

Confirm the target version in the private inventory:

```bash
rg -n '^k3s_version:' inventory/haplolabs/group_vars/all.yml
```

Run safe checks:

```bash
ansible-inventory -i inventory/haplolabs/hosts.yml --graph
ansible-playbook --syntax-check -i inventory/haplolabs/hosts.yml upgrade-k3s-one-node.yml
ansible-playbook --list-hosts -i inventory/haplolabs/hosts.yml upgrade-k3s-one-node.yml --limit k8s-srv-0
kubectl get nodes -o wide
kubectl get --raw='/readyz?verbose'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded -o wide
```

Stop if any node is not Ready, API readiness fails, or unexpected pods are failing.

## Upgrade Order

Upgrade server/control-plane nodes first, one at a time. Then upgrade worker nodes one at a time.

Recommended order for this cluster:

```text
k8s-srv-0
k8s-srv-1
k8s-srv-2
k8s-worker-0
k8s-worker-1
k8s-worker-2
```

## Per-Node Command

Run exactly one host at a time:

```bash
ansible-playbook -i inventory/haplolabs/hosts.yml upgrade-k3s-one-node.yml --limit k8s-srv-0
```

Repeat with the next host only after validation passes.

## Validation After Each Node

Run these after every node:

```bash
kubectl get nodes -o wide
kubectl get --raw='/readyz?verbose'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded -o wide
kubectl -n kube-system get pods -l name=kube-vip-ds -o wide
kubectl -n calico-system get deployment,daemonset,pods -o wide
kubectl get tigerastatuses.operator.tigera.io -o wide
kubectl -n metallb-system get deployment,daemonset,pods -o wide
kubectl -n metallb-system get servicel2statuses -o wide
kubectl -n vault-prod get pods -o wide
kubectl -n vault-prod get endpoints vault-active vault-active-lb vault-standby -o wide
kubectl -n longhorn-system get pods -o wide
kubectl -n longhorn-system get volumes.longhorn.io -o wide
```

Expected result:

- The upgraded node returns to Ready.
- The upgraded node reports the target k3s version.
- API readiness passes.
- No unexpected failed pods appear.
- kube-vip remains 3/3 on control-plane nodes.
- Calico/Tigera is Available and not Degraded.
- MetalLB controller and speakers are Ready.
- Vault active endpoint and active LoadBalancer endpoint remain present.
- Longhorn pods and volumes remain healthy.

## Stop Conditions

Stop before moving to the next node if any of these occur:

- A node does not return to Ready.
- `/readyz` fails, especially etcd readiness.
- Vault active endpoint or `vault-active-lb` endpoint disappears unexpectedly.
- Longhorn reports unhealthy volumes or unavailable system pods.
- Calico/Tigera reports Degraded or unavailable components.
- MetalLB loses `ServiceL2Status` for Vault or ingress.
- kube-vip DaemonSet drops below 3/3 after a control-plane node upgrade.
- The upgraded node does not report the target k3s version.
