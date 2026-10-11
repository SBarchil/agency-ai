# ADR 0003: React for back office and partner portal, Next.js for the public site

Status: Accepted · Date: 2026-10-11

## Decision
React + TypeScript + Vite single-page apps for staff and partners; Next.js for the public catalogue; shared UI package; API client generated from OpenAPI.

## Why
Large ecosystem for data-heavy screens; static files run easily on Kubernetes; server rendering needed for search engines on the public site.

## Rejected alternatives
Blazor Server (sticky sessions and live connections on Kubernetes).
