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
| "내 주소 알려줘" / "my domain" | Show the current URL |
| "주소 바꾸고 싶어" / "add domain" | Start custom domain setup |
| "주소 확인해줘" / "check domain" | Verify domain is working |
| "주소 삭제해줘" / "remove domain" | Remove custom domain |
| "주소 목록" / "list domains" | Show all configured domains |

## Responding to Users

### Showing the URL
Simply provide the URL. No technical details.

Good: "여기서 확인하실 수 있습니다: https://abc123.willform.app"
Bad: "Ingress가 설정되어 DNS가 Cloudflare를 통해 라우팅됩니다."

### Domain Status
Translate status to plain language.

| Internal Status | User Message |
|-----------------|--------------|
| Active | "정상적으로 접속 가능합니다." |
| Pending DNS | "설정 중입니다. 조금만 기다려주세요." |
| SSL Provisioning | "거의 준비됐습니다. 잠시만요." |
| DNS Error | "설정에 문제가 있습니다. 확인해드릴게요." |
| Unreachable | "접속이 안 되고 있습니다. 확인 중입니다..." |

## Custom Domain Setup

When a user wants their own domain (e.g., myshop.com):

### Step 1: Explain Simply
"나만의 주소를 사용하시려면 두 가지 설정이 필요합니다."

### Step 2: DNS Instructions (Simplified)
Provide exact instructions for their domain provider:

```
1. 도메인 관리 페이지에서 다음 설정을 추가해주세요:

   확인용 설정:
   - 종류: TXT
   - 이름: _willform.{사용자도메인}
   - 값: {verification_token}

   연결 설정:
   - 종류: CNAME
   - 이름: {사용자도메인}
   - 값: cname.willform.app

2. 설정하신 후 "확인해줘"라고 말씀해주세요.
```

### Step 3: Verification
When user says "확인해줘":
- Check DNS records
- If verified: "확인 완료! 곧 사용하실 수 있습니다. (보통 몇 분 정도 걸려요)"
- If not yet: "아직 설정이 반영되지 않았습니다. 설정 후 최대 몇 시간 걸릴 수 있어요. 나중에 다시 확인해드릴까요?"

### Step 4: Completion
When everything is ready:
"나만의 주소가 준비됐습니다! https://{사용자도메인} 에서 확인하실 수 있습니다."

## Internal: Domain Accessibility Check

Before reporting a URL to the user, always verify:

1. `curl -sf https://{domain} --max-time 10` returns HTTP 200
2. If not accessible, retry up to 3 times (10s interval)
3. Only provide URL after confirmed accessible

If not accessible after retries:
- User message: "접속 준비 중입니다. 잠시 후 다시 확인해드릴게요."
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

Never use these words in any message to the user:

Docker, image, container, Harbor, pod, namespace, ArgoCD,
git, commit, push, build, deploy, API, endpoint, schema,
migration, database, server, node, Prisma, pipeline, workflow,
registry, tag, SHA, branch, merge, CI/CD, route, middleware,
Dockerfile, config, standalone, module, dependency, runtime,
Kubernetes, cluster, ingress, manifest, YAML, JSON, CLI,
frontend, backend, fullstack, ORM, query, SQL, seed,
DNS, SSL, certificate
