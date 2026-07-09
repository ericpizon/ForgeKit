# Security Policy

## What ForgeKit is

ForgeKit is an experimental AI-SDLC protocol — a set of markdown command files for Claude Code. It is not production compliance software, a certified audit platform, or a security control system. It does not store credentials, transmit data, or run server-side processes.

Use ForgeKit to structure your AI-assisted software delivery process. Do not use it as a substitute for formal compliance certification, legal review, or security audits.

## Reporting a vulnerability

If you find a security issue in ForgeKit — including prompt injection risks, command logic that could cause harmful output, or patterns that could mislead users into unsafe practices — please report it responsibly.

**Do not file a public GitHub issue for security vulnerabilities.**

Instead, email: **ericpizon@gmail.com**

Include:
- A description of the issue and the affected file(s)
- Steps to reproduce or a proof-of-concept scenario
- Your assessment of severity and potential impact

You will receive an acknowledgment within 5 business days. If the issue is confirmed, a fix will be prioritized based on severity.

## Scope

In scope:
- Security logic errors in command files (e.g., a security review command that misses a known vulnerability class)
- Prompt patterns that could produce harmful or misleading output
- Documentation that could lead users to make unsafe security decisions

Out of scope:
- Vulnerabilities in Claude itself or in Claude Code — report those to Anthropic
- Issues in target projects that happen to use ForgeKit
- Theoretical risks with no practical exploitation path

## Disclosure

Once a fix is available, the vulnerability will be disclosed in a GitHub Security Advisory. Credit will be given to the reporter unless anonymity is requested.
