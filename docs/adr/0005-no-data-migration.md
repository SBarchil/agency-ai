# ADR 0005: Start with an empty system

Status: Accepted · Date: 2026-10-11

## Decision
No data is imported from the old system. Reference data, partners, contracts and circuit templates are entered fresh (contracts and templates with AI help). On go-live day only cash counts and bank balances are entered as starting figures.

## Why
Old data has no keys, duplicated contract versions and float amounts; cleaning it would cost more than re-entering current data.

## Rejected alternatives
Importer from the old SQL Server databases.
