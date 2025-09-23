---
title: OAuth Part 2 - Authorization Code Flow
date: 2026-01-9 8:00:00 +/-0200
categories: [OAuth]
tags: [OAuth, Security, Authorization] # TAG names should always be lowercase
image: /assets/img/posts/code_flow/redirect.jpg
---
<!-- the blog flow
1. why starting with the flow 
2. introduction to the specifications هتلكم عن كل جزء لحد الرابع
3. اقول نبذه علي الفلو من غير اللوجن
4. اشرح كل جزء من الفلو الصغير ده من الدوكس 
5. واشرح الاتاكس لكل جزء برده
6. صوره الفلو الكبير
7. اكسر الفلو الكبير

✅ 1. Authorization Code Interception Attack
What is it?
An attacker intercepts the authorization code while it’s being redirected to the client.

Prevention via Code Flow (with PKCE):

PKCE (Proof Key for Code Exchange): The client sends a code_challenge with the authorization request and a code_verifier when exchanging the code for a token.

The token endpoint checks if they match, preventing use of intercepted codes by attackers.

✅ 2. Redirect URI Manipulation
What is it?
An attacker changes the redirect URI to steal the code or tokens.

Prevention via Code Flow:

The authorization server only redirects to pre-registered redirect URIs, avoiding redirection to malicious URLs.

It should strictly match the redirect URI to the one registered.

✅ 3. CSRF (Cross-Site Request Forgery)
What is it?
An attacker tricks a user into making an unwanted authorization request.

Prevention via Code Flow:

Use of a random state parameter to bind the authorization request to the user’s session.

The client must verify the state value in the redirect to prevent CSRF.

✅ 4. Token Leakage via Front Channel
What is it?
Tokens (especially access tokens) are leaked in the browser URL or referer headers.

Prevention via Code Flow:

The access token is never exposed in the front channel (browser), unlike the Implicit Flow.

Tokens are only sent via secure backchannel communication.

✅ 5. Phishing Attacks
What is it?
Users are tricked into providing credentials to fake login pages.

Prevention via Code Flow:

OAuth doesn't directly prevent phishing, but using OpenID Connect with trusted identity providers and educating users helps.

FIDO2/WebAuthn can also reduce this risk when used with OAuth.

✅ 6. Replay Attacks
What is it?
An attacker reuses an old authorization code or request.

Prevention via Code Flow:

Authorization codes are one-time use only and expire quickly.

The use of PKCE ensures that even reused codes without a valid code_verifier will be rejected.

✅ 7. Client Impersonation
What is it?
An attacker impersonates a legitimate client to get tokens.

Prevention via Code Flow:

Confidential clients must authenticate with client_id + client_secret at the token endpoint.

Public clients use PKCE to provide proof of origin.

✅ 8. Misuse of Access Token
What is it?
An attacker uses a token issued for one client (or scope) in another context.

Prevention via Code Flow:

Scopes and audience are strictly validated by the resource server.

Use JWT validation to ensure tokens are issued for the correct client/resource. -->

<h3 id='intro' style="font-weight: bold;">Introduction .. Why Starting With Authorization Code Flow ..?</h3>

The Authorization Code Flow is OAuth 2.0's most fundamental and secure flow - which is why everyone teaching OAuth begins here. 
While it has more steps than alternatives like the **Implicit Flow**, these extra steps provide critical security benefits.

You'll use this flow in about **90%** of real-world implementations. Master this one first, and the others (which are essentially simplified variants with fewer steps) will be much easier to understand. All the core concepts and terminology you learn here apply across all OAuth flows.


### **First Let’s Dive Into the OAuth 2.0 Specification** [**Flow RFC**](https://datatracker.ietf.org/doc/html/rfc6749)

- The first part is the introduction, where all the terminology is explained.

![Image](/assets/img/posts/code_flow/rfc_introduction.PNG)

- The second part covers client registration. When you want to use an authorization server and register your client (e.g., adding "Login with Google"), Google provides you with a **client ID** and **client secret**. You also need to specify the **redirect URL**.

![Image](/assets/img/posts/code_flow/rfc_client_registration.PNG)
- The third part discusses the endpoints we need to implement but focuses more on the token endpoint rather than the authorization endpoint. This section provides the base implementation, while section 4 contains the additional details required for each flow, as seen in `4.1.3 Token Endpoint Extension`

![Image](/assets/img/posts/code_flow/rfc_auth_enpoints.PNG)

- The fourth part describes the different authorization flows and how to obtain an access token and the implementation.

![Image](/assets/img/posts/code_flow/rfc_auth_flows.PNG)






### **Explaining the Flow from a High Level Perspective**
Let’s break down the flow step by step from a High Level view, without getting too deep into technical details just yet. The idea here is to understand the big picture of what’s happening when a user logs in using the Authorization Code Flow.

1. **The user wants to log in** – They click a “Login with…” button on your app.
2. **Your app redirects them to the authorization server** – Along with this redirect, it sends some important metadata like:
   - `client_id`: a unique identifier for your app
   - `redirect_uri`: where the server should send the response after authentication
   - `response_type=code`: to indicate that you want an authorization code
   - `scope`: what kind of access/Privilages your app is requesting (e.g., email, profile)
   - `state`: a random string to prevent CSRF attacks, is are using PKCE This parameter would be useless
   - (optionally in Oauth2.0) `code_challenge` for PKCE, which improves security in public clients , in Oauth2.1 this is Required

3. **The user logs in and gives consent** – If it’s their first time, they’ll be asked to approve access to their data.
4. **The authorization server redirects back to your app** – It sends an authorization `code` to the `redirect_uri` you provided.
5. **Your backend exchanges that code for tokens** – Your backend sends a secure request to the token endpoint, including:
   - `client_id` and `client_secret` (if confidential client)
   - The same `code` it received
   - `redirect_uri` again (to match the original request)
   - `code_verifier` if you used PKCE

   In return, your backend receives an **access token**, and optionally a **refresh token**

That’s the general idea. 




- _**Note:**_ For More Clarification Think about the "Client" as MVC Application the return view as it do actions,

<!--  
![Image](/assets/img/posts/code_flow/code_flow.svg) -->



## Explaining the Code Flow
![Image](/assets/img/posts/code_flow/code_flow_1.svg)

### First Part Starting The Autherization Process
**①** First, the user interacts with their user agent (opens their browser and navigates to the desired site, for example www.client.com). This website helps archive contacts from different sources like Gmail.

------

**②** There's a button Get Gmail Contacts that retrieves contacts from Gmail and lists them. When we click it:

---------

③ Your user agent (browser) sends a GET request to the client endpoint https://client.com/gmail/contacts

---------

**④** The client see that you are dont have valid token or don't have a session , so it  responds with a Location header to redirect to the auth server's authorize endpoint https://auth.com/authorize

```perl
HTTP/1.1 302 Found
Location: https://auth.com/authorize?
  response_type=code
  &client_id=YOUR_CLIENT_ID
  &redirect_uri=https%3A%2F%2Fclient.com%2Fcallback
  &scope=gmail.read%20contacts
  &state=RANDOM_CSRF_TOKEN
  &code_challenge=BASE64URLENCODED_SHA256_OF_VERIFIER
  &code_challenge_method=S256
```
--------- 

### Flow Starting
**⑤** Your browser sends a GET request to the authorization endpoint, including all the required query parameters:
client_id, redirect_uri, scope, state, code_challenge, code_challenge_method, and others.

`client_id` is used to identify your client application.

`scopes` define the permissions your application is requesting.
In this example, the scope is contacts because we want access to the user's Gmail contacts.

In this flow, we’re using `PKCE`, which stands for `Proof Key for Code Exchange`.
It’s a security enhancement. While `PKCE` is optional in OAuth 2.0, it is required in OAuth 2.1

To implement `PKCE`, we need to include two additional query parameters when calling the authorization endpoint:

`code_challenge`

`code_challenge_method`

So what happens behind the scenes?

Before redirecting the user to the authorization endpoint, the client generates a random string called the `code_verifier`, it’s just a random string and is stored (persisted) on the client side.

This code_verifier is then hashed using the method defined in `code_challenge_method` (typically S256).
The result of that hash is the `code_challenge`.

To summarize:

The client stores the `code_verifier`.

It sends the `code_challenge` and `code_challenge_method` to the authorization endpoint, along with other metadata such as `client_id`, `scope`, etc.

We’ll explain in the upcoming steps why this extra step exists and what benefits it brings — but for now, just remember it.

**⑥**After hitting the authorize endpoint, the auth server notices you're not authenticated (skipped if logged in) and returns a redirect response with Location header to the login page
```perl
HTTP/1.1 302 Found
Location: https://auth.com/login?
  redirect_to=%2Fauthorize%3Fresponse_type%3Dcode%26client_id%3DYOUR_CLIENT_ID%26redirect_uri%3Dhttps%253A%252F%252Fclient.com%252Fcallback%26scope%3Dgmail.read%2520contacts%26state%3DRANDOM_STATE_TOKEN%26code_challenge%3D...%26code_challenge_method%3DS256
Set-Cookie: session=LONG_RANDOM_SESSION_ID; Secure; HttpOnly; SameSite=Lax

```

⑦ Your browser sends a GET request to retrieve the login page https://auth.com/login

⑧ The auth server returns the login page/View as response

⑨ The user enters their credentials (email and password).
Note: OAuth 2.0/2.1 doesn't specify authentication methods, only authorization steps to obtain a token. That's why discussions about OAuth always mention "authorization" not Authentication.

⑩ The browser sends a POST request to the login endpoint with user credentials. If wrong, returns an error (implementation-specific)

⑪ If credentials are correct, as a final authentication step the auth server shows a consent screen asking "Are you sure you want to give access to the client website?"

Clicking "No" returns forbidden (or handles rejection) and stops the auth flow

Clicking "Allow" sends request with consent approval

⑫ After clicking "Allow":

⑬ The system logs you in and creates a cookie representing your authenticated session

⑭ The login endpoint returns a redirect to the browser pointing back to the auth server's authorize endpoint to continue the auth flow

Note: The consent screen step is optional. If authenticating your own client, you can make the consent step implicit (basically skipping it)

⑮ The browser redirects again to the authorize endpoint, but now you're authenticated so it generates an authorization code instead of redirecting to login

⑯ The authorize endpoint returns a response redirecting to the client's callback URL with the code

⑰ The browser redirects to the client callback with the code

⑱ The client uses a backchannel to exchange the code at the token endpoint, sending both the code and code_verifier for validation

⑲ The server returns an access token to the client if the code and code_verifier are valid

Note: You might wonder why the auth server returns a code instead of the token directly, and why you need to send the code_verifier

⑳ The client stores the token

㉑ The client redirects to the original page where the flow started (https://client.com), returning a redirect response to the browser

㉒ The browser redirects and sends a GET request for that page

㉓ The client returns the page so the user can continue their original operation (calling Gmail to get contacts)

Note: At this point, the OAuth code flow has ended

㉔ Now the user can continue their original operation and click Get Gmail Contacts

㉕ The browser sends a GET request to https://client.com/gmail/contacts

㉖ The client uses the stored token to call Gmail's API to fetch contacts

㉗ Gmail returns the contacts

㉘ The client returns the contacts to the browser for display
