# Data Center Colocation Visit & Work Permit System

This repository contains the high-level software architecture and system design specification for the **Data Center Colocation Visit & Work Permit System**.

## Overview

The system is a dedicated operational intake and workflow automation platform designed to manage the end-to-end lifecycle of tenant physical data center access requests:
- **Client & Tenant Intake:** Multi-day visit scheduling, company-to-room/rack mapping, vehicle registration, and technician roster management.
- **Ticketing Orchestration:** Automated Request Tracker (RT) integration via `rt-ticket-colo-maker`, supporting deduplication, batch consecutive date scheduling, and parent-child ticket hierarchies for 30-day blanket access passes.
- **Dual-Audience Notifications:** Automated dispatch of client confirmation emails and internal DC Ops / NOC alerts with one-click printable links.
- **Physical Custody Tracking:** Generation of standardized, single-page A4 work order permits for on-site clipboard escort, room verification, and DC tool custody audit.

## Documentation

For the complete architectural specification, operational invariants, system diagrams, and printable work order layout, see:
- [High-Level Software Architecture Specification](datacenter_colo_visit_software_architecture.md)
