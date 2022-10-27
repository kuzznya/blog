---
title: "Overview of authentication protocols"
date: 2022-10-27T20:55:00+03:00
categories:
  - System Design
tags:
  - authentication
  - security
draft: false
---

## Definitions

**Identification** is the ability to identify uniquely a client (a user or an application) of a system.

**Authentication** is the process of verifying one's identity, 
e.g. when subject provides a password, so that system can verify his identity by comparing it to the saved one.
Authentication verifies that someone or something is who they say they are, so authentication enables identification.

**Authorization** is the mechanism to determine access levels or user/client privileges related to system resources or actions. 
Basically, authorization is the process of verifying what a user has access to.

## Authentication factors

- Knowledge factor  
  Users are expected to know some credentials: 
  password, PIN or answers to security questions
- Posession factor  
  Users are expected to provide evidence of posessing some physical item, e.g. SIM card, hardware OTP token (one-time password generator)
- Inherence factor  
  Users are expected to provide evidence inherent to unique features, e.g. fingerprint scans or facial recognition.

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
- Forms-based
- TLS (for certificate-based authentication)
- SAML
- OpenID Connect (based on OAuth 2.0)

## Most simple and least secure

In HTTP authentications, web server should respond to unauthorized client 
with status `401 Unauthorized` and header `WWW-Authenticate` with schema definition and authentication parameters.
In that case web browser automatically shows authentication dialog to the user. 
For all subsequent requests browser adds header `Authorization` with authentication data. 
Server should use this header to authenticate the client.

### Basic authentication

Basic authentication is really basic: username and password are simply encoded using Base64 and : as a separator.

```plantuml
Client -> Server : GET /resource
Server -> Client : 401 Unauthorized \n WWW-Authenticate: Basic
Client -> Server : GET /resource \n Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
Server -> Client : 200 OK
```

`dXNlcm5hbWU6cGFzc3dvcmQ=` is base64 encoded string `username:password`.

This authentication can be used only for HTTPs (encrypted) connections, otherwise intermediate nodes can easily access credentials.

### Digest authentication

Digest authentication is a more secure alternative to basic authentication.
To avoid direct password tramission, server sends unique value 'nonce',
then web browser encrypts MD5 password hash using 'nonce'.

This authentication is better than basic authentication, but it is still vulnerable to man-in-the-middle (MITM) attacks.
Also, this authentication does not allow server to store passwords using strong hash algorithms like bcrypt, 
because password should be recoverable.

Both Basic and Digest authentications do not allow server to log user off (after a certain period of inactivity).
The only way for user to log out is to close all the tabs of the website.

## Forms-based authentication

Forms-based authentication is not a standard but a common implementation. To implement it, the application should include an HTML form that the user fills in and sends to the server using POST request 
(so that credentials would be passed in the request body, not in query params).
Server validates this form and then adds session token to browser cookies. 
In subsequent queries session token is sent to server automatically.

Session token can be an identifier of a session in some server storage (in-memory or key-value storage like Redis),
or it can be encrypted or signed user data (this method is described in [token-based authentication section](#token-based-authentication))

```plantuml
Client -> Server : GET /login
Server -> Client : 200 OK <html>
Client -> Server : POST /login \n Content-Type: application/x-www-form-urlencoded \n username=user1&password=pass
Server -> Client : 200 OK \n Set-Cookie: token=<token>
Client -> Server : GET /resource \n Cookie: token=<token>
```

The credentials are sent non-encrypted with this type of authentication, so it also requires HTTPs to be secure.

## Certificate-based authentication

This type of authentication is not widely used, but it is a lot more secure than password-based authentications.

Certificate-based authentication is part of the TLS protocol and happens during the connection to the server.
Server checks that the certificate passed by the client is signed by trusted certificate authority, 
nonexpired and not revoked.

```plantuml
node Client
node Server
node "Certificate Authority" as ca

Server -> ca : trusts
ca -down-> Client : issues certificate
Client -> Server : uses certificate
```

Certificate-based authentication is hard to implement and to support.

## Two-factor authentication

Two-factor authentication (2FA) is actually not a standard but a concept 
that user should provide two different authentication factors to prove his identity.
For example, the application can combine forms-based authentication with single use password from SMS.

```plantuml
participant Client
participant Server
participant "Mobile Operator" as operator
Client -> Server : username + password
Server -> Server : validate credentials
Server -> operator : request SMS sending
operator -> Client : send SMS
Server --> Client : request 2nd factor, i.e. SMS code
Client -> Server : provide code from SMS
Server -> Client : resource
```

## Access key authentication

Access keys are the replacement of username/password pair.
An access key is usually a long unique string made from random characters.
This makes acess key more secure, because it is almost impossible to brute force it.
The key is usually easier to revoke if it was compromised.

Usually the `Authorization` header is used: `Authorization: Bearer <API key>` or a custom header, e.g. `APIKey: <API key>`.

Access keys can be emulated when [token-based authentication](#token-based-authentication) is used, 
in that case an access key is a token containing some data inside of itself.

## Token-based authentication

### JWT

JWT - JSON Web Token - is a de-facto standard of authentication tokens.
It consists of three parts:
1. Header - metadata about token itself in JSON format, e.g. what algorithm is used for signing
2. Payload - contents of token itself in JSON format
3. Signature - HMAC (with secret), RSA or ECDSA (with public/private key pair) algorithm

Header and payload are also encoded using Base64.

The killer feature of this format is that it contain the real data: role of the user, his permissions, his email, etc.

Example of JWT:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNjY2MjM5MDIyLCJleHAiOjE2NjYyNDkwMjIsInJvbGUiOiJBRE1JTiIsImVtYWlsIjoic29tZUBtYWlsLmNvbSJ9
.
oflWKWY28JX_1_Kl7kBG1y6KPXChqd3uHw2f4dZwJBk
```

(line breaks added only to beautify)

```json
{
  "alg": "HS256",
  "typ": "JWT"
}

{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1666239022,
  "exp": 1666249022,
  "role": "ADMIN",
  "email": "some@mail.com"
}
```

The information in token can be trusted because it is digitally signed, 
so anyone can read it, but no one can modify the content of the token - the signature will not match the content.
Also, without knowing the secret or the private key, no one can generate the token by himself.

JWT is a perfect fit to achieve stateless backend application, because authentication data can be passed along with each request,
so the server does not need to store session information, which allows easier scalability. 
Distributed architectures can also benefit a lot from JWTs, 
because every component can validate JWT and get authentication data without the need to query any other component, 
all what is need is either secret (for HMAC algorithm) or public key (for RSA or ECDSA).

The expiration timestamp is usually written inside of the JWT, so server can easily validate that the token is not expired yet.

JWTs are used in OAuth 2.0 and OpenID Connect, described below,
but they can also be used with Forms-based authentication.

**Refresh tokens**

JWT-based authentication is oftenly extended by the use of refresh tokens.
It is a special token type that is used to refresh usual access token. It should not be used for anything besides that.

The authentication with refresh tokens allow access tokens to expire quickly (in a couple of minutes), so even if it is compromised, 
the attacker can use it only for a short period of time.
Refresh tokens, on the other side, should live longer (days or weeks), so that user should not provide username and password every time.

Refresh token should be single use, so if a refresh token is compromised, then the server can detect reuse of the same refresh token and revoke all user tokens.

Refresh token should be stored securely in HttpOnly cookie (without JavaScript code access) for the domain of authentication system (identity provider). 
Access token can be accessed by JavaScript code to be able to send it to any API that uses the same authentication system.

### SAML

Security Assertion Markup Language (SAML) is an authentication and authorization protocol. It uses XML to transfer data.
It consists of two parties: service provider (the application) and identity provider (the service that authenticates users).

SAML has a mechanism of verifying client posession of the token (JWT does not have this feature, so an attacker can use compromised JWT to access the application).

SAML provides Single Sign-On (SSO) that allows a user to use same identity for multiple sites if they use shared identity provider.

SAML signs tokens in XML format like JWT does.

It is a complex protocol compared to OpenID Connect, so currently OpenID Connect is more common.

## OAuth 2.0 + OpenID Connect

This will be covered in the next article or later here...
