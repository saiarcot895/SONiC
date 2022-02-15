# DHCP Server

# High Level Design Document

# Table of Contents
* [Scope](#scope)
* [Definition](#definition)
* [Overview](#overview)
* [DHCPv4 Configuration](#dhcpv4-configuration)
* [Requirements](#requirements)
* [Topology](#topology)
* [Design](#design)
  - [Feature table](#feature-table)
  - [CoPP manager](#copp-manager)
  - [Source IP](#source-ip)
* [Performance](#performance)
* [Testing](#testing)

# Scope

This document describes high level design details of SONiC's DHCP server.

# Definition

DHCP: Dynamic Host Configuration Protocol

# Overview

SONiC currently has a DHCPv4 relay and DHCPv6 relay for forwarding DHCP requests from downstream client devices to upstream servers. However, some networks may want to have a DHCP server running directly on the switch itself, instead of forwarding it to a separate server. This could be because with the network topology in use, the IP addresses used for some network are all contained within this switch, and so it may be simpler to have the DHCP server running locally. Additionally, this allows the DHCP server configuration to be linked to the configuration and current state of the switch, such as quickly changing DHCP network configurations based on ports that are up or down, or changing static DHCP configuration based on devices that are present. For these purpose, a DHCPv4 server can be configured to run on the switch, and then configured via the minigraph and/or a python script.

In the existing DHCP relay container, we are already using ISC's DHCP relay application (from the isc-dhcp-relay package), with some changes done for SONiC. There is also a DHCP server application available (from the isc-dhcp-server package). However, the configuration for the server cannot be quickly updated without restarting the process. This means that if the configuration needs to be changed due to the state or configuration of the switch changing, then the process would need to be fully restarted. Because this is running within a SONiC container, and because of our supervisord process, this would mean restarting the container entirely. To avoid this, dnsmasq will be used instead to act as a DHCPv4 server. Dnsmasq supports rereading certain configuration options (specifically, those related to static DHCP hosts and DHCP options to use in responses) by sending a SIGHUP signal to the process.

# DHCPv4 Configuration

Dnsmasq supports reading its configuration from multiple sources:

1. The configuration settings can be directly specified on the command line as arguments
2. The configuration settings are read from `/etc/dnsmasq.conf`
3. The confiugration settings can be placed in multiple files in a directory (such as `/etc/dnsmasq.d`)

In SONiC, the configuration options will be placed in multiple files in `/etc/dnsmasq.d`. This is primarily for convenience, but this could potentially allow some sort of logical separation of config options in the future.

The config file will specify a range of DHCP addresses to allocate from for each VLAN defined in the minigraph. It will also instruct dnsmasq to bind to the interfaces it needs to listen on, instead of using an unbound socket, so that only the traffic for the defined VLANs are handled by dnsmasq. Finally, a file that will contain static DHCP host entries (`/etc/dnsmasq.hosts`) will be specified here. This file will be reread by dnsmasq every time it receives a SIGHUP. This file isn't required to exist, and it can be an empty file; in either case, dnsmasq will assume there's no static DHCP host entries to process and proceed.

# Requirements

- Configured and running DHCPv4 client
- Connectivity between the DHCPv4 client and the switch
- Client UDP port: 67
- Server UDP port: 68

# Topology

TODO

# Design

## Feature table

Adding to existing DHCP relay container. No new feature table added

## CoPP manager

Control Plane Policing manager is configured to trap DHCPv4 packets.

TODO: Add when this trap is installed

## Source IP

VLAN SVI IP

Configurable option to use loopback address for dual ToR

# Performance

The DHCP server is not expected to be handling a high number of packets, so performance should not be impacted.

# Testing

TODO
