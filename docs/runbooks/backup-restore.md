# Backup & Restore Runbook

*This will be filled in as backup solutions are implemented.*

## What to Back Up

| Component   | Tool              | Frequency | Target         |
| ----------- | ----------------- | --------- | -------------- |
| etcd        | k3s etcd snapshot | Daily     | (TBD)          |
| K8s objects | Velero            | Daily     | S3-compatible  |
| Volumes     | Longhorn backup   | Daily     | S3/NFS         |
| Manifests   | Git (this repo)   | On change | GitHub         |

## Backup Procedures

*(To be documented as each tool is configured)*

## Restore Procedures

*(To be documented as each tool is configured)*

## Tested Restores

| Date | Restore Type | Result | Notes |
| ---- | ------------ | ------ | ----- |
| —    | —            | —      | —     |
