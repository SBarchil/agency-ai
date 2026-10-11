# ADR 0001: Modular monolith with Clean Architecture

Status: Accepted · Date: 2026-10-11

## Decision
One solution and one database, split into modules (Catalogue, Contracts, Pricing, Bookings, Partners, Billing, Treasury, GuestAssistant, Identity) inside Clean Architecture layers. Each host (back office, partner API, public site, worker, MCP server, migrator) is its own container and Kubernetes deployment.

## Why
Small team; business rules are tightly connected (pricing, bookings, billing); microservices would add network, data-consistency and deployment cost with no benefit now.

## Rejected alternatives
Microservices (too costly for one developer); classic layered app without module borders (old system's problem).
