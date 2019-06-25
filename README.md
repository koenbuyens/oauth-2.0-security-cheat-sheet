# OAuth 2.0 Security Cheat Sheet

- [OAuth 2.0 Cheat Sheet](#OAuth-20-Cheat-Sheet)
  - [Introduction](#Introduction)
  - [Architectural Decisions](#Architectural-Decisions)
    - [Use the Authorization Code Grant for Classic Web Applications and Native Mobile Apps](#Use-the-Authorization-Code-Grant-for-Classic-Web-Applications-and-Native-Mobile-Apps)
    - [Use Refresh Tokens When You Trust the Client to Store Them Securely](#Use-Refresh-Tokens-When-You-Trust-the-Client-to-Store-Them-Securely)
    - [Use Handle-Based Tokens Outside Your Network](#Use-Handle-Based-Tokens-Outside-Your-Network)
  - [Client Credentials](#Client-Credentials)
    - [Server: Generate the client credentials using a cryptographically strong random number generator](#Server-Generate-the-client-credentials-using-a-cryptographically-strong-random-number-generator)
    - [Server: Implement rate limiting on the exchange/token endpoint](#Server-Implement-rate-limiting-on-the-exchangetoken-endpoint)
    - [Server: Use a Cryptographic hashing algorithm that is appropriate for storing secrets](#Server-Use-a-Cryptographic-hashing-algorithm-that-is-appropriate-for-storing-secrets)
    - [Client: Store the client secret securely on the client](#Client-Store-the-client-secret-securely-on-the-client)
  - [Tokens](#Tokens)
    - [Server: Store handle-based access and refresh tokens securely](#Server-Store-handle-based-access-and-refresh-tokens-securely)
    - [Server: Expire access and refresh tokens](#Server-Expire-access-and-refresh-tokens)
    - [Client: Store handle-based access and refresh tokens securely](#Client-Store-handle-based-access-and-refresh-tokens-securely)
  - [Authorization Code Grant](#Authorization-Code-Grant)
    - [Client: Use the state parameter](#Client-Use-the-state-parameter)
    - [Server: Expire Authorization Codes](#Server-Expire-Authorization-Codes)
    - [Server: Invalidate authorization codes after use](#Server-Invalidate-authorization-codes-after-use)
    - [Server: Generate strong authorization codes](#Server-Generate-strong-authorization-codes)
    - [Server: Bind client to authorization code](#Server-Bind-client-to-authorization-code)
    - [Server: Validate the redirect URI](#Server-Validate-the-redirect-URI)
    - [Server: Hash authorization codes](#Server-Hash-authorization-codes)
  - [PKCE](#PKCE)
    - [Use PKCE when AuthZ Code Grant](#Use-PKCE-when-AuthZ-Code-Grant)
    - [Use the SHA-256 Challenge Method](#Use-the-SHA-256-Challenge-Method)
  - [Resource Owner Password Grant](#Resource-Owner-Password-Grant)
    - [Only Use Resource Owner Password Grant for First-Party Apps](#Only-Use-Resource-Owner-Password-Grant-for-First-Party-Apps)
    - [Follow Password-AuthN Best Practices at OAuth Endpoint](#Follow-Password-AuthN-Best-Practices-at-OAuth-Endpoint)
    - [Store Refresh Token Rather Than User Passwords](#Store-Refresh-Token-Rather-Than-User-Passwords)
  - [Client Credentials Grant](#Client-Credentials-Grant)
  - [Open ID/Connect](#Open-IDConnect)

## Introduction

OAuth 2.0 is a standard that enables users to give websites access to their data/services at other websites. For instance, a user gives a photo printing website access to her pictures on Flickr. Before performing a deep-dive into the specifics of OAuth 2.0, we introduce some definitions (taken from auth0):

- **Resource Owner**: the entity that can grant access to a protected resource. Typically this is the end-user.
- **Client**: an application requesting access to a protected resource on behalf of the Resource Owner. This is also called a Relying Party.
- **Resource Server**: the server hosting the protected resources. This is the API you want to access.
- **Authorization Server**: the server that authenticates the Resource Owner, and issues access tokens after getting proper authorization. This is also called an identity provider (IdP).
- **User Agent**: the agent used by the Resource Owner to interact with the Client, for example a browser or a mobile application.

In OAuth 2.0, the interactions between the user and her client, the Authorization Server, and the Resource Server can be performed in four different flows. These flows enable the clients to obtain Access Tokens and the clients use these access tokens to access an API. The flows are as follows.

- **Authorization Code Grant**: the Client redirects the user (Resource Owner) to an Authorization Server to ask the user whether the Client can access her Resources. After the user confirms, the Client obtains an Authorization Code that the Client can exchange for an Access Token. This Access Token enables the Client to access the Resources of the Resource Owner.
- **Implicit Grant**: this grant is a simplification of the authorization code grant. The Client obtains the Access Token directly rather than being issued an Authorization Code.
- **Resource Owner Password Credentials Grant**: this grant enables the Client to obtain an Access Token by using the username and password of the Resource Owner.
- **Client Credentials Grant**: this grant enables the Client to obtain an Access Token by using its own credentials.

OAuth 2.0 uses two types of tokens, namely Access Tokens and Refresh Tokens.

- **Access Tokens** are tokens that the Client gives to the API to access Resources.
- **Refresh Tokens** are tokens that the Client can use to obtain a new Access Token once the old one expires.

Think of Access Tokens like a session that is created once you authenticate to a website. As long as that session is valid, we can interact with that website without needing to login again. Once the session times out, we would need to login again with our username and password. Refresh tokens are like that password, as they allow a Client to create a new session.

There are two ways to pass the above tokens throughout the system, namely:

- **Handle-based Tokens**  (opaque tokens, reference tokens) are typically random strings that can be used to retrieve the data associated with that token. This is similar to passing a variable by reference in a programming language.
- **Self-contained Tokens** are tokens that contain all the data associated with that token. This is similar to passing a variable by value in a programming language. This is typically expressed as a JWT.

Refresh tokens are handle-based, access tokens can be either, and identity tokens (OpenID/Connect) are self contained.

**OpenID/Connect** is an authentication mechanism built on top of the authZ code grant. The server also returns an identity token (JWT) containing information about the user.

There are two types of clients:
- A **Public Client** is an Oauth Client that cannot store client secrets securely. This category includes mobile applications, single page applications, and desktop clients as an attacker is able to extract the secrets by downloading those applications.
- A **Confidential Client** is an OAuth Client that is able to store secrets securely. This category includes classic web applications. 

## Architectural Decisions

### Use the Authorization Code Grant for Classic Web Applications and Native Mobile Apps

A major design decision is deciding which flows to support. This largely depends on the type of clients the application supports. [Auth0](https://auth0.com/docs/api-auth/which-oauth-flow-to-use) provides an excellent flow chart that helps making a good decision. In summary, if the Client is:

- A classic web application, use the Authorization Code Grant.
- A single page application, use the ~~Implicit Grant~~ Authorization Code Grant without secrets.
- A native mobile application, use the Authorization Code Grant with PKCE.
- A client that is absolutely trusted with user credentials (i.e. the Facebook app accessing Facebook), use the Resource Owner Password Grant.
- A client that is the owner of the data, use the Client Credentials Grant.

### Use Refresh Tokens When You Trust the Client to Store Them Securely

Just like passwords, it is important that the Client stores these Refresh Tokens securely. If you do not trust that the client will store those tokens securely, do not issue Refresh Tokens. An attacker with access to the Refresh Tokens can obtain new Access Tokens and use those Access tokens to access a user's Resources. The main downside of not using Refresh Tokens is that users/clients would need to re-authenticate every time the Access Token expires.

### Use Handle-Based Tokens Outside Your Network

To increase maintainability, use self-contained tokens within your network, but use handle-based tokens outside of it. The main reason is that Access Tokens are meant to be consumed by the API itself, not by the Client. If we share self-contained tokens with Clients, they might consume these tokens in ways that we had not intended. Changing the content (or structure) of our self-contained tokens might thus break lots of Client applications (in ways we had not foreseen).

If Client applications require access to data that is in the self-contained token, offer an API that enables Client applications to obtain information related to that Access Token. If we want to use self-contained tokens and prevent clients from accessing the contents, encrypt those tokens.

## Client Credentials

### Server: Generate the client credentials using a cryptographically strong random number generator

If the client secrets are weak, an attacker may be able to guess them at the token endpoint.

To remediate this, generate secrets with a length of at least 128 bit using a secure pseudo-random number generator that is seeded properly. Most mature OAuth 2.0 frameworks implement this correctly.

### Server: Implement rate limiting on the exchange/token endpoint

To prevent bruteforcing, OAuth 2.0 endpoints should implement rate limiting as it slows down an attacker.

### Server: Use a Cryptographic hashing algorithm that is appropriate for storing secrets

If the client secrets are stored as plain text, an attacker may be able to obtain them from the database at the resource server or the token endpoint.

To remediate this, store the client secrets like you would store user passwords: hashed with a strong hashing algorithm such as bcrypt, scrypt, or pbkdf2. When validating the secret, hash the incoming secret and compare it against the one stored in the database for that client.

### Client: Store the client secret securely on the client

An attacker may be able to extract secrets from client (and uses them to spoof the client).

To remediate this, store the secrets using secure storage offered by the technology stack (typically encrypted). Keep these secrets out of version repositories.

Note: public clients should NOT have secrets.

## Tokens

### Server: Store handle-based access and refresh tokens securely

If the handle-based tokens are stored as plain text, an attacker may be able to obtain them from the database at the resource server or the token endpoint.

To remediate this, hash the tokens before storing them using a strong hashing algorithm. When validating the token, hash the incoming token and validate whether that hashed value exists in the database.

### Server: Expire access and refresh tokens

Expiring access and refresh tokens limits the window in which an attacker can use captured or guessed tokens.

To remediate this, expire access tokens 15-30 minutes after they have been generated. Refresh tokens can be valid for much longer. The actual amount depends on the risk profile of the application. Anything between a couple of hours or a year might be acceptable.

### Client: Store handle-based access and refresh tokens securely

If the handle-based tokens are stored as plain text in a database, an attacker may be able to obtain them from the database at the client.

To remediate this, keep the access tokens in memory and store the refresh tokens using secure storage offered by the technology stack (typically encrypted).

## Authorization Code Grant

Authorization code grant is used by various clients to exchange an authorization code for an access token.

The flow is illustrated below.
![Authorization code grant](https://raw.githubusercontent.com/koenbuyens/Damn-Vulnerable-OAuth-2.0-Applications/master/pics/AuthorizationCodeGrant.png)

1. A user, let's call her Vivian, navigates to the printing website, photoprint. This website is called the Client. Vivian uploaded the pictures to picture gallery site (gallery). The printing website offers the possibility to obtain pictures from the gallery site via a button that says “Print pictures from Gallery”. Vivian clicks that button.
2. The client redirects her to an Authorization Server (AS; Authorization Endpoint). In our case, hosted by gallery. The URI contains the parameters redirect_uri, scope, response_type, and client_id. The redirect_uri is where gallery will redirect Vivian after having created an authorization code. The scope is the access level that the client needs (view_gallery is a custom scope that enables clients to view the pictures from a user's gallery). The response_type is code as we want to use the authorization code grant. The client_id is an identifier that represents the photoprint application.
That server allows Vivian to authenticate to the gallery site and asks her if she consents to the Client photoprint accessing her pictures.
3. Assuming that Vivian gives her consent, the AS generates an Authorization Code (Authorization Grant) and sends it back to Vivian’s browser with a redirect command toward the return URL specified by the Client photoprint (in step 2).
4. The browser honors the redirect and passes the Authorization Code to the Client (photoprint).
5. The Client photoprint forwards that Authorization Code together with its own credentials to the AS (Token Endpoint) at gallery. From this step on, the interactions are server-to-server. The Authorization Code proves that Vivian consented to the actions that the Client photoprint wants to do. Moreover, the message contains the Client’s own credentials (the Client ID and the Client Secret).
6. Assuming that the Client photoprint is allowed to make requests, the Token Endpoint at the Authorization Server gallery issues the Client photoprint an Access Token. The AS may also issue a Refresh Token. The refresh token enables the Client photoprint to obtain new access tokens; e.g. when the old ones expire. Typically, refresh tokens are issued when offline_access is added to the scope in step 2.
7. The Client photoprint then uses the access token to access a Protected Resource, Vivian's pictures, on the Resource Server gallery. The gallery application then validates the access token and processes the request.

### Client: Use the state parameter

An attacker will be able to associate his/her OAuth 2.0 identity with the user's account at the client application if the client does not use the state parameter.

The optional 'state' parameter should be used. The client application should generate a secure random string, store the secure random string in the user's session, and send it to the authorization server using the 'state' parameter via the the user's browser. The authorization server will send this parameter back after the authorization request via the user's browser. The client application should then validate whether the value stored in the requesting user's session matches the received value.

### Server: Expire Authorization Codes

Attackers that steal or brute force unused authorization codes will be able to use them regardless of how long ago they were issued.

To mitigate this, expire authorization codes after 10-15 minutes if they have not been used.

### Server: Invalidate authorization codes after use

Attackers can reuse authorization codes when intercepted. Clients exchange authorization codes together with a client ID and client secret for access tokens. These access tokens then grant the client access to the victim's resources. Attackers can also do this if the client does not use a client secret (e.g. public client) or if the client secret is compromised as an attacker can obtain access tokens with the intercepted authorization code(s).

When an authorization code is exchanged for an access code, the authorization server should invalidate the authorization code and not issue any more access codes against it.

### Server: Generate strong authorization codes

An attacker may be able to guess the authorization code if they are weak.

Generate authorization codes with a length of at least 128 bits using a cryptographically secure pseudo-random number generator (CSPRNG) that is seeded properly.

### Server: Bind client to authorization code

An attacker that obtains authorization codes provided by the application to a particular client will be able to exchange them for access tokens using his/her own client ID and client secret. These access tokens will then grant the attacker access to the victim's resources.

Verify that the client that executes the exchange request is the same as the client that was provided with the authorization code.

### Server: Validate the redirect URI

If the redirect URI is not validated, an attacker may be able to perform an open redirect or steal authorization codes. Attackers can redirect the user to a website they control by initially providing the URL of a website they trust (the authorization server). As the authorization code is a parameter appended to the redirect URI by the server, an attacker may be able to steal the authorization code. This authorization code can be used to get access tokens if the client secret is compromised or not needed.

The authorization server should verify whether the provided redirecturi is one of the redirecturis that the clients provided during the registration process. It is important that the match is an exact string match, as otherwise attackers may be able to circumvent the URI validation logic or perform other types of technology-specific attacks.

### Server: Hash authorization codes

Attackers may steal authorization codes from the database using SQL injection.

Hash authorization codes when stored in the database on the authorization server.

## PKCE

The OAuth 2.0 protocol aims at authorizing applications to access users' resources (stored at the resource server) on their behalf without the users having to provide their credentials to those applications. Within the grant authorization flow, the authorization server generates an authorization code, which is a token that the OAuth 2.0 client can exchange (together with its credentials) for an access token.

The PKCE extension uses a challenge-response protocol to prevent attackers from using intercepted authorization codes. The public client generates a code verifier (which is a cryptographically random string between 43 and 128 characters in length), calculates a challenge (which is either the code verifier itself or a SHA256 hash of the code verifier) and sends the challenge to the Authorization Server in the authorization request. The authorization server generates an authorization code and associates it with the received challenge. The server then returns the authorization code to the client. This application subsequently exchanges the code verifier together with the authorization code for an access token by contacting the Authorization Server. The Authorization Server verifies the challenge using the received code verifier. If verification succeeds (i.e. the challenge matches the code verifier for the plain code challenge method, or the challenge matches a SHA256 hash of the code verifier for the SHA256 code challenge method), the authorization server returns a new access token. If verification fails, it rejects the request

### Use PKCE when AuthZ Code Grant

An attacker might be able to inject (replay) authorization codes into the authorization response.

To mitigate this, Use PKCE when using the authorization code flow.

### Use the SHA-256 Challenge Method

An attacker that can steal an authorization code and brute force its code verifier will be able to exchange it for an access token.

To mitigate this, use the SHA256 code challenge method.

## Resource Owner Password Grant

The resource owner password credentials grant is a simplified flow in which the client uses the resource owner password credentials (username and password) to obtain an access token. It consists of the following steps.

A user, let's call her Vivian, uses the OAuth Client to upload pictures to a gallery site (Gallery). (e.g. mobile app).
1. The client submits Vivian's credentials to the Authorization Server. 
2. The AS generates an Access Token and sends it back to the Client. 
3. The Client can then access the Resource Server to upload Vivian's pictures with the Access Token.

### Only Use Resource Owner Password Grant for First-Party Apps

A malicious client may be able to obtain the user passwords or obtain powerful tokens (scopes) without End-User authorization.

To avoid this, only use the resource owner password grant when you trust the client with the credentials (e.g. facebook app accessing facebook API).

### Follow Password-AuthN Best Practices at OAuth Endpoint

An attacker might be able to guess the user passwords.

To avoid this, implement account lockout (and strong user passwords) at the password endpoint at the authorization server.

### Store Refresh Token Rather Than User Passwords

An attacker might be able to obtain the user password at client.

Do not store the user credentials on the device. Store the refresh token instead.

## Client Credentials Grant

When the authorization scope is limited to protected resources under the control of the client or to protected resources previously arranged with the authorization server, then the client credentials can be used as a grant. This is typically used in B2B scenarios. The flow is as follows.

1. The client submits her credentials to the Authorization Server.
2. The AS generates an Access Token and sends it back to the Client.
3. The Client can access the resources with the Access Token.

Follow the best practices around client credentials and tokens.

## Open ID/Connect

Extension of Authorization Code Grant to authenticate users. The authorization server returns an identity token (JWT) signed by the identity provider together with the access token. Clients can use this token to identify the user.

The best practices for the Authorization Code Grant, JWT, and OAuth Credentials apply.
