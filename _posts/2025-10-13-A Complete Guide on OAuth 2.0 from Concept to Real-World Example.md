---
title: "A Complete Guide on OAuth 2.0: from Concept to Real-World Example"
date: 2025-10-13 10:00:00 +0000
categories: [Web, Security]
tags: [OAuth2, Google, Authorization]
pin: false
author: Jiaming HUANG
image:
  path: /assets/img/oauth2-diagram.png
  alt: OAuth 2.0 Flow Diagram
---

When you click “Login with Google,” “Continue with Facebook,” or “Sign in via GitHub,” you’re actually using the same security protocol behind the scenes — **OAuth 2.0**.  

This article explains what OAuth 2.0 is, why it’s needed, and how it works, step by step — with an example that shows the entire flow in practice.

---

## 1. What Is OAuth 2.0?

**OAuth 2.0** is an **open standard for authorization**.

It allows third-party applications to access data on behalf of a user **without needing the user’s password**.

> 🔑 Important distinction:  
> OAuth 2.0 is about **authorization** (“what you’re allowed to do”), not **authentication** (“who you are”).

---

## 2. The Core Idea: Secure Delegated Authorization

Let’s start with a simple analogy.

Imagine you’re staying at a hotel. You have a master keycard (your password) that can open your room, the gym, and the restaurant.  

Now you want to let the valet (a third-party app) park your car. You wouldn’t hand over your master card — that would give access to everything.  

Instead, you go to the front desk (the authorization server) and ask for a **temporary valet card** that only allows parking access.

This temporary card has:
- **Limited scope:** it only opens the parking area.  
- **Expiration:** it expires after a few hours.  
- **Revocability:** you can cancel it anytime at the front desk.

That’s exactly how **OAuth 2.0** works — it provides a secure, limited, and revocable way to delegate permissions **without exposing your real password**.

---

## 3. The Four Key Roles in OAuth 2.0

| Role                     | Analogy              | Description                           |
| ------------------------ | -------------------- | ------------------------------------- |
| **Resource Owner**       | You, the hotel guest | The user who owns the data            |
| **Client**               | The valet            | The third-party app requesting access |
| **Authorization Server** | The hotel front desk | Verifies identity and issues tokens   |
| **Resource Server**      | The parking lot      | Hosts the protected resources         |

In most cases, the **authorization server** and **resource server** are operated by the same provider — e.g., Google.

---

## 4. A Real Example: “Login with Google to Import Photos”

Let’s walk through a real-world example using the **Authorization Code Flow**, the most common and secure OAuth 2.0 pattern.

> Scenario:  
> You want to print photos using *AwesomePrints.com*, which needs access to your Google Photos.

---

### Step 1: User Starts the Process
You click the button:  
> “Import photos from Google Photos”

---

### Step 2: Redirect to Google Authorization Page
AwesomePrints redirects your browser to Google’s OAuth page, including:

- `client_id`: identifies AwesomePrints  
- `redirect_uri`: where Google should send you back  
- `scope`: the requested permission (e.g., read-only access to photos)  
- `response_type=code`: specifies that it wants an authorization code  

---

### Step 3: User Grants Permission
You log in to Google, then see a confirmation page:

> “AwesomePrints.com wants to view your Google Photos.”  
> [Allow] [Deny]

You click **Allow** — this action happens **on Google’s domain**, so your password is never shared with AwesomePrints.

---

### Step 4: Google Returns an Authorization Code
Google redirects you back to AwesomePrints with a **temporary code**:

```
https://awesomeprints.com/callback?code=ABC123XYZ
```

---

### Step 5: The Server Exchanges the Code for a Token
AwesomePrints’ **backend** (not the browser) sends a request to Google’s authorization server containing:

- the `code` it just received  
- its own `client_id`  
- its `client_secret` (stored securely on the server)  

---

### Step 6: Google Issues an Access Token
Google verifies everything and returns an **Access Token** — a digital key that allows AwesomePrints to access your photos.

---

### Step 7: The App Uses the Token
AwesomePrints sends a request to Google Photos API:

```http
GET /photos/api/v1/list
Authorization: Bearer <access_token>
```

---

### Step 8: Google Responds with Data
Google validates the token and returns your photo data.

✅ Done — AwesomePrints can now access your photos, **without ever seeing your password**.

---

## 5. Where Do `client_id` and `client_secret` Come From?

They aren’t randomly generated — they’re issued when a developer registers their app on Google’s developer platform.

### Steps to Obtain Them:

1. Go to [Google Cloud Console](https://console.cloud.google.com)  
2. Create a new project (e.g., “AwesomePrints App”)  
3. Enable the **Google Photos API**  
4. Configure the **OAuth consent screen** (name, logo, privacy policy, requested scopes)  
5. Create an **OAuth Client ID**, specifying:
   - Application type (e.g., “Web application”)  
   - Authorized redirect URIs (e.g., `https://awesomeprints.com/callback`)
6. Google then generates:
   ```text
   client_id: 8975...apps.googleusercontent.com
   client_secret: GOCSPX-...
   ```

> - **Client ID** → public identifier (like a username)  
> - **Client Secret** → private password stored on the backend only  

---

## 6. Why Not Just Use `client_id` and `client_secret` Directly to Get an Access Token?

That sounds simpler, but it’s **extremely unsafe**.

If the access token were sent directly through the browser, the following risks would occur:

| Risk                     | Description                                |
| ------------------------ | ------------------------------------------ |
| **Browser history leak** | Token could appear in URL and be stored    |
| **Referer header leak**  | Token might be sent to third-party sites   |
| **Malicious plugins**    | Browser extensions could steal the token   |
| **XSS attacks**          | Cross-site scripts could extract the token |

Even worse — the `client_secret` would need to be used in the front-end, exposing it to everyone.

---

### The Solution: The Authorization Code Middleman

OAuth 2.0 solves this by splitting the flow into two channels:

| Channel           | Location                | Data Exchanged                            |
| ----------------- | ----------------------- | ----------------------------------------- |
| **Front Channel** | User’s browser          | Temporary `authorization_code` (low risk) |
| **Back Channel**  | Secure server-to-server | `client_secret` and `access_token`        |

Even if someone steals the authorization code, it’s useless — they can’t exchange it for an access token without the secret key that only lives on the backend.

---

### Comparison Table

| Approach                | Exposed in Browser                           | Risk Level |
| ----------------------- | -------------------------------------------- | ---------- |
| Direct (no code)        | `access_token` (high value, long-lived)      | ❌ Unsafe   |
| Authorization Code Flow | `authorization_code` (short-lived, one-time) | ✅ Secure   |

This “extra step” makes all the difference — it isolates sensitive data from untrusted environments.

---

## 7. Summary

| Feature                             | Description                                             |
| ----------------------------------- | ------------------------------------------------------- |
| **High security**                   | Passwords are never exposed to third-party apps         |
| **Scoped access**                   | Users can control which permissions to grant            |
| **Revocable tokens**                | Users can revoke access anytime                         |
| **Front-end / Back-end separation** | Sensitive data is exchanged securely on the server side |

OAuth 2.0 may seem complex at first glance, but its layered design is what keeps the modern web secure.  

From “Login with Google” to GitHub integrations to enterprise APIs — **OAuth 2.0 is the foundation of safe, delegated access**.

---

## 💬 Final Thoughts

OAuth 2.0 isn’t just a “login protocol.”  
It’s a **carefully designed system of secure delegation**, built to ensure that:

> **You can let someone help you — without giving away your keys.**
