SSH Config Parsing: A Definitive Guide
This document outlines the various scenarios and complexities involved in parsing OpenSSH configuration files (.ssh/config). The primary goal is to reliably differentiate between Connectable Hosts, which should be displayed in a user interface, and Configuration Rules, which apply settings in the background.

## 1. Core Concepts
### 🎯 Connectable Hosts
These are specific, named entries that a user can directly connect to. Each `Host` block is expected to declare a single label, and the parser treats each label as its own connection. Multi-label alias lists (for example, `Host webapp webapp-backup`) are unsupported.

### 📜 Configuration Rules
These are patterns or conditional blocks that apply settings to a class of connections. They are not directly connectable and should be hidden from a primary server list.

## 2. Scenarios for Connectable Hosts
These entries should be parsed and presented to the user as valid connection targets. Each `Host` block should contain exactly one label; alias lists are unsupported and must be split into discrete entries.

> ⚠️ Unsupported: `Host webapp webapp-backup` (alias list). Create separate blocks for each label so that every connection can be listed individually.

**Example: Single Host Label**

Host webapp
  HostName 192.168.1.50
  User admin
  Port 2222
Classification: Connectable

Key Info: The label `webapp` identifies the connection. Use additional `Host` blocks if you need `webapp-backup` or other names.

**Example: Implicit Hostname**

Host prod-server.example.com
  User ubuntu
  IdentityFile ~/.ssh/prod.key
Classification: Connectable

Key Info: The label and hostname are both prod-server.example.com. A parser that requires HostName to be present will incorrectly classify this as a rule.

**Example: Host with a Proxy or Bastion**

Host private-db
  HostName 10.0.50.15
  ProxyJump bastion-host
Classification: Connectable

Key Info: The user wants to connect to private-db. The SSH client handles the jump through bastion-host automatically.

## 3. Scenarios for Configuration Rules
These entries should be parsed to apply their settings but hidden from the main UI list.

### Scenario: Wildcard Patterns
A Host entry containing * or ? is a pattern, not a specific host.

Code snippet

Host *.internal.network
  User internal_user
  ForwardAgent yes
Classification: Rule

Reason: It applies to an infinite number of potential hosts (e.g., server1.internal.network, db.internal.network).

### Scenario: Multi-Pattern Declaration
A single Host line can contain multiple patterns, separated by spaces.

Code snippet

Host server* *.dev *.staging !prod
  User devuser
  StrictHostKeyChecking no
Classification: Rule

Reason: It defines a set of conditions for applying settings, not a single destination. The parser must split the line into individual patterns, and alias lists (multiple plain labels) should be considered unsupported for connectable hosts.

### Scenario: Negated Patterns
A pattern starting with ! is an exception. It matches everything except the pattern.

Code snippet

Host !bastion-host
  ProxyCommand /usr/bin/corp_proxy %h %p
Classification: Rule

Reason: This is a conditional rule for all connections that are not to bastion-host.

### Scenario: The Match Directive
The Match keyword introduces a conditional block that is more powerful than Host. It is always a rule.

Code snippet

# Applies settings to any connection by the 'root' user to any host
Match user root
  IdentityFile ~/.ssh/id_rsa_root
Classification: Rule

Reason: Match blocks are purely for applying settings based on criteria like user, host, or even the execution of a command.

### Scenario: The Include Directive
This directive tells the SSH client to load and parse another configuration file.

Code snippet

# Load all work-related configurations
Include ~/.ssh/config.d/work/*
Classification: Meta-Rule

Reason: This is a command for the parser itself. It is not a host or a rule, but it's essential for discovering all other hosts and rules.

## 4. Parser Requirements & Recommended Strategy
### ✅ Minimum Parser Requirements
A custom parser must be able to:

Differentiate Top-Level Keywords: Recognize Host, Match, and Include as distinct block starters.

Handle Multi-Patterns: Split Host lines into multiple patterns.

Detect Rules: Identify wildcards (*, ?) and negation (!).

Handle Quotes: Correctly parse quoted arguments.

Be Case-Insensitive: Treat keywords like Host, User, HostName as case-insensitive.

Process Include: Recursively read and parse included files.

### ⭐ The Gold Standard: Offload to ssh -G
The most robust and accurate method is to avoid writing a complex custom parser.

Discover Hosts: Use a simple regex like ^\s*Host\s+([^\s*?!]+)$ to find candidate labels for your UI. This is a "best-effort" discovery step that intentionally ignores `Host` lines with multiple labels because alias lists are unsupported.

Evaluate Configuration: For a selected label (e.g., webapp), run the command ssh -G webapp.

Parse the Output: The command produces a simple, definitive key value list of the final, effective configuration for that host. This output accounts for all Host blocks, Match blocks, Include files, and precedence rules.

This strategy guarantees 100% accuracy with the user's native SSH environment and is future-proof against new ssh_config features.