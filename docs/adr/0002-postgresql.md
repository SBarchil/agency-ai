# ADR 0002: PostgreSQL as the only database

Status: Accepted · Date: 2026-10-11

## Decision
PostgreSQL with EF Core (Npgsql). pgvector for AI knowledge search. Managed service in production.

## Why
No license cost; daterange + exclusion constraints enforce contract and season rules; jsonb stores AI output; pgvector avoids a separate vector database.

## Rejected alternatives
SQL Server (license, fewer range features); separate vector database (extra service).
