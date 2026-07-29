# Node Failure Runbook

*This will be filled in as procedures are tested.*

## Worker Node Failure (Qotom)

**Symptom:** Worker shows `NotReady` or is unreachable.

**Check:**
```bash
k3s kubectl get nodes
ssh remy@worker_node1
systemctl status k3s-agent
```

**Resolution:**
1. SSH into worker node
2. Check k3s-agent status: `systemctl status k3s-agent`
3. Restart if needed: `systemctl restart k3s-agent`
4. If the node is dead, pods will reschedule on the master (if resources allow)
5. If the node cannot be recovered, remove it:
   ```bash
   k3s kubectl delete node worker_node1
   ```
6. Prepare replacement node, add to inventory, re-run Ansible

## Control Plane Failure (HP EliteDesk)

**Symptom:** Entire cluster inaccessible.

**Check:**
```bash
ssh remy@control_node
systemctl status k3s
```

**Resolution:**
1. SSH into master node
2. Check k3s server: `systemctl status k3s`
3. Restart if needed: `systemctl restart k3s`
4. If hardware failure, the cluster is down until replaced — this is a single-control-plane setup
5. Build replacement node, run Ansible playbook, rejoin worker

## Notes

- The 8GB worker has limited capacity; if it goes down, the master must handle all workloads
- Critical workloads should be scheduled on the master or have anti-affinity rules
- Longhorn replicas on both nodes provide data redundancy if one node fails
