# Lab 01 - Palo Alto Initial Configuration

## Topology
Win VM → e0 → mgmt → PA-VM (PAN-OS 10.0)

## Objective
Configure Palo Alto firewall from scratch including
management IP, admin password, and Web UI access.

## Environment
- EVE-NG
- PAN-OS 10.0
- PA-VM

## Configuration Steps

### 1. Set Static Management IP
set deviceconfig system type static
set deviceconfig system ip-address 192.168.10.1
set deviceconfig system netmask 255.255.255.0

### 2. Set Admin Password
set mgt-config users admin password

### 3. Commit Configuration
commit

### 4. Access Web UI
https://192.168.10.1

## Status
✅ Completed
