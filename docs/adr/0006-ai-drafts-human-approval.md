# ADR 0006: AI drafts, humans approve

Status: Accepted · Date: 2026-10-11

## Decision
AI may create drafts (contract versions, bookings from emails, expense lines). Only a logged-in user with the right permission can approve. The AI service account cannot call approve endpoints. Guest assistant and MCP tools are read-only except creating leads.

## Why
Contracts and money must never change without a responsible person; AI output can be wrong or manipulated (prompt injection).

## Rejected alternatives
Automatic approval above a confidence score.
