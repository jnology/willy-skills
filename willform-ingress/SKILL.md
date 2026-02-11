---
name: willform-ingress
description: Manage domain and URL routing for user applications
user-invocable: true
---

# Willform Ingress

Domain and URL management for user applications.

## Core Principle

Users only care about ONE thing: "Where can I see my app?"
All internal routing, DNS, and certificate details are invisible to the user.

## Default Domain

Every project automatically gets a domain:
```
https://{projectId}.willform.app
```

This is ready from the moment the project is created. No setup needed.

## User-Facing Commands

| User says | What to do |
|-----------|------------|
| "what's my address" / "my domain" | Show the current URL |
| "I want my own domain" / "add domain" | Start custom domain setup |
| "check my domain" / "check domain" | Verify domain is working |
| "remove my domain" / "remove domain" | Remove custom domain |
| "show all domains" / "list domains" | Show all configured domains |

## Responding to Users

### Showing the URL
Simply provide the URL. No technical details.

Good: "Here's your app: https://abc123.willform.app"
Bad: "Ingress is configured and DNS is routed through Cloudflare."

### Domain Status
Translate status to plain language.

| Internal Status | User Message |
|-----------------|--------------|
| Active | "Your app is up and running." |
| Pending DNS | "Setting things up. Just a moment." |
| SSL Provisioning | "Almost ready. Hang tight." |
| DNS Error | "There's a problem with the setup. Let me check." |
| Unreachable | "Can't reach the app right now. Looking into it..." |

## Custom Domain Setup

When a user wants their own domain (e.g., myshop.com):

### Step 1: Explain Simply
"To use your own domain, you'll need to set up two things."

### Step 2: DNS Instructions (Simplified)
Provide exact instructions for their domain provider:

```
1. Go to your domain settings and add these two records:

   Verification record:
   - Type: TXT
   - Name: _willform.{userDomain}
   - Value: {verification_token}

   Connection record:
   - Type: CNAME
   - Name: {userDomain}
   - Value: cname.willform.app

2. Once you've added them, just say "check it" and I'll verify.
```

### Step 3: Verification
When user says "check it":
- Check DNS records
- If verified: "All set! Your domain will be ready shortly. (Usually takes a few minutes.)"
- If not yet: "The settings haven't taken effect yet. It can take up to a few hours. Want me to check again later?"

### Step 4: Completion
When everything is ready:
"Your custom domain is ready! Check it out at https://{userDomain}"

## Internal: Domain Accessibility Check

Before reporting a URL to the user, always verify:

1. `curl -sf https://{domain} --max-time 10` returns HTTP 200
2. If not accessible, retry up to 3 times (10s interval)
3. Only provide URL after confirmed accessible

If not accessible after retries:
- User message: "Still getting things ready. I'll check again shortly."
- Do NOT provide the URL

## Internal: Routing Architecture

Default domains use Cloudflare Tunnel:
```
User -> Cloudflare Edge -> Cloudflare Tunnel -> K8s Service (app:3000)
```

Custom domains require:
1. DNS verification (TXT record)
2. CNAME or A record pointing to Willform
3. Cloudflare proxy configuration
4. Certificate provisioning via cert-manager

## FORBIDDEN Words

FORBIDDEN words: See willform-guard SKILL.
