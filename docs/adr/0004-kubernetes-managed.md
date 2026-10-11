# ADR 0004: Managed Kubernetes with managed data services

Status: Accepted · Date: 2026-10-11

## Decision
Run on a managed Kubernetes service in a European region close to Morocco; PostgreSQL and Redis as managed services outside the cluster; Helm charts; GitHub Actions CI/CD.

## Why
Removes cluster and database operations work for a small team; backups and point-in-time recovery for billing data. Confirm data-protection rules (Law 09-08 / CNDP) before choosing the region.

## Rejected alternatives
Self-managed cluster on customer servers (only if data must stay on-site).
