---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "Seamless Native-to-Browser Sessions with OAuth 2.0 Tokens"
abbrev: "Native to Browser Session Transfer"
category: info

docname: draft-ubique-oidc-native-to-browser-session-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "connect"
keyword:
 - native-to-browser
 - session transfer
 - in-app browser tab

author:
 -
    fullname: Fabian Aggeler
    organization: Ubique Innovation 
    email: aggeler@ubique.ch
 -  fullname: Patrick Amrein
    organization: Ubique Innovation 
    email: amrein@ubique.ch

normative:

informative:


--- abstract

This specification defines a short-lived session token mechanism to transfer a user’s native OAuth session into in-app browser tabs, ensuring consistent, secure sign-on and minimizing re-authentication prompts.


--- middle

# Introduction

In many native (mobile) applications, users authenticate via an OpenID Connect flow—launching a system or in-app browser to sign in at the OpenID Provider (OP). The app receives ID tokens, refresh tokens, and access tokens. Yet, when the same app later opens a web page (in a custom tab or SFSafariViewController), there is no straightforward mechanism to transfer the established “app session” into a browser session. This often forces re-authentication or leaves the user in an unauthenticated state in the browser context.

To bridge this gap, we propose a short-lived, single-use Session Transfer Token that an OIDC Client (the native app) can request and then pass to a new web context, allowing the browser to seamlessly create or refresh an SSO session at the OP. This approach leverages existing OIDC components and session management concepts while improving user experience and security.

# Session Transfer Token

1. **Issue**: Once the user has authenticated in the native app (and holds valid tokens issued by the Authorization Server), the app can request an STT from the Authorization Server. The STT is time-limited (e.g., 1–5 minutes) and can be redeemed only once.

2. **Redemption**: When the native app opens a web page (in an in-app browser or external browser), it attaches the STT to the request (for instance, via query parameter or a custom scheme). Upon receiving the STT, the server (or directly the Authorization Server) checks its validity. If valid, it establishes a session for the user in the browser context—usually by setting a secure, HTTP-only cookie.

3. **Single-Use & Expiration**: The Authorization Server marks the STT as consumed when it is redeemed, preventing reuse or replay attacks. The token’s short validity window further limits exposure if it is ever intercepted.


# Redemtion Flows

There are two primary ways to redeem a short-lived, single-use session token (STT) to establish or refresh a user’s authenticated session in a web context:

1. **RP-Initiated Flow**:  
   The Relying Party (RP) begins a standard OpenID Connect (OIDC) or OAuth 2.0 flow, passing the STT to the Identity Provider (IdP) (e.g., via `login_hint`) so that user interaction is minimized or avoided entirely.

2. **IdP-Initiated Flow**:  
   The native app directly navigates to the IdP with the STT. The IdP sets an SSO cookie (or refreshes the existing one), then redirects the user to the RP, which completes an OIDC flow silently, recognizing the already-authenticated user.

## RP-Initiated Flow

1. **App → RP**  
   - The native mobile app opens a URL pointing to the Relying Party, attaching the STT (e.g., `?session_token=XYZ123`).
   - Example:  
     ```
     https://rp.example.com/protected?session_token=XYZ123
     ```

2. **RP Parses the STT**  
   - The RP checks the incoming `session_token` parameter.
   - If it finds a valid token, it initiates an **OIDC Authorization Request** to the IdP, including the STT as a `login_hint` (or another relevant parameter the IdP will recognize).

3. **IdP Validates the STT**  
   - The IdP sees the `login_hint=XYZ123`.
   - It introspects or otherwise verifies the STT (e.g., single-use, not expired, tied to the correct user/client).
   - If valid, the IdP can issue an **Authorization Code** or **ID Token** to the RP without prompting the user to log in again.

4. **RP Creates a Web Session**  
   - The RP redeems the authorization code for tokens (if using the Authorization Code flow).
   - It sets a secure, HTTP-only session cookie, so the user remains logged in within the browser context.

5. **STT Marked as Used**  
   - The IdP (or relevant introspection service) marks the STT as consumed.
   - Replaying `XYZ123` again should fail.

**Advantages**  
- Reuses a familiar OIDC flow.  
- The RP initiates the sign-in process and controls how it manages the resulting session.  

**Considerations**  
- The RP must know how to handle and forward the STT.  
- The user’s browser will see at least one redirect to the IdP.


## IdP-Initiated Flow

1. **App Navigates Directly to the IdP**  
   - Instead of hitting the RP first, the native app instructs the in-app browser or system browser to load a URL on the IdP’s domain.
   - Example:  
     ```
     https://idp.example.com/redeem_session?
       session_token=XYZ123
       &redirect_uri=https://rp.example.com/welcome
     ```
   - This request includes both the STT and the final destination (`redirect_uri`).

2. **IdP Redeems the STT**  
   - The IdP verifies the STT (single-use, not expired, etc.).
   - If valid, the IdP **sets or refreshes the SSO cookie** in the user’s browser context.
   - Once the cookie is set, the IdP redirects the browser to the `redirect_uri` (the RP endpoint).

3. **RP Recognizes SSO Cookie**  
   - When the user’s browser lands on `https://rp.example.com/welcome`, the RP either:
     - (a) initiates a **silent OIDC flow** (e.g., a quick Authorization Request) and sees that the user already has an authenticated session at the IdP, or  
     - (b) trusts the existing SSO cookie if the RP and IdP share a domain or session mechanism.
   - No user interaction is required, assuming the IdP session cookie is valid.

4. **RP Creates a Web Session**  
   - The RP then sets its own local session cookie and redirects the user to the intended page.
   - The STT is marked as used on the IdP side.

**Advantages**  
- Centralized session creation at the IdP.  
- Guarantees that any subsequent OIDC flows from this browser context are silent or minimal in prompts.

**Considerations**  
- Requires the IdP to provide a dedicated endpoint (`/redeem_session` or similar).  
- The RP is somewhat “passive” at first, receiving the user post-redirect with a valid IdP session in place.


# Key Benefits
- **User Experience**: No repeated logins in in-app browser tabs.
- **Security**: Single-use tokens limit replay risks; strict expiration reduces attack windows.
- **Flexibility**: Works with both RP-initiated and IdP-initiated flows.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
