---
title: "Overview of authentication protocols"
date: 2022-10-25T16:36:54+03:00
categories:
  - System Design
tags:
  - authentication
  - security
draft: true
---

## Definitions

**Identification** is the ability to identify uniquely a client (a user or an application) of a system.

**Authentication** is the process of verifying one's identity, 
e.g. when subject provides a password, so that system can verify his identity by comparing it to the saved one.
Authentication verifies that someone or something is who they say they are, so authentication enables identification.

**Authorization** is the mechanism to determine access levels or user/client privileges related to system resources or actions. 
Basically, authorization is the process of verifying what a user has access to.

## Authentication methods

- Passwords
- Certificates
- 2FA (single use passwords, SMS code, phone call, code in email)
- access keys (e.g. API keys)
- tokens (JWT, SAML, etc.)

## Main protocols

- HTTP authentication
  - Basic
  - Digest
  - Windows related authentications
- Forms authentication
- 
