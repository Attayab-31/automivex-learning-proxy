# Automivex Learning Proxy

A minimal Vercel project that serves the existing Cloudflare Worker through `https://learn.automivex.com` using a fixed external rewrite. The browser URL remains on `learn.automivex.com` for requests that the Worker answers without issuing a redirect.

## What this project does

- Proxies all request paths to `https://automivex-learning.automivex-learning.workers.dev/` using Vercel external rewrites.
- Keeps the upstream hostname fixed in configuration; it is not a user-controlled proxy.
- Includes Vercel's documented `x-vercel-enable-rewrite-caching: 0` request header so dynamic responses are not cached by Vercel's external rewrite cache behavior.
- Contains no app runtime, functions, database, or additional service.

The wildcard `/:path*` captures nested paths (`/api/jobs/123` becomes `/api/jobs/123`). Query parameters are preserved by the rewrite behavior when they are not explicitly replaced in the destination. Vercel external rewrites proxy HTTP methods and request payloads; actual upstream behavior must still be checked against the Worker.

## Requirements

- A GitHub account and a Vercel account.
- Access to the Hostinger DNS zone for `automivex.com`.
- The existing Worker remains deployed and reachable.

No nameserver change is required. Do not import this project into or connect it to the Vercel project for the main website; create a separate Vercel project for this repository.

## 1. Create a separate GitHub repository

Create a **new, empty** GitHub repository named `automivex-learning-proxy`. Do not select a README, license, or `.gitignore` initializer if you plan to push this existing local project.

In PowerShell, from this project's directory, run:

```powershell
cd E:\reverse-automivex\automivex-learning-proxy
git init -b main
git add vercel.json README.md .gitignore
git commit -m "Add Vercel reverse proxy for learning Worker"
git remote add origin https://github.com/YOUR-GITHUB-USERNAME/automivex-learning-proxy.git
git push -u origin main
```

Replace `YOUR-GITHUB-USERNAME` with your GitHub username or organization. These commands operate only on this new directory and the new repository.

## 2. Import and deploy on Vercel

1. Open the [Vercel Dashboard](https://vercel.com/dashboard).
2. Choose **Add New → Project**, then import the new `automivex-learning-proxy` GitHub repository.
3. Keep the project root at the repository root. No framework, build command, output directory, or environment variables are required; Vercel can deploy the root `vercel.json` configuration as a static configuration project.
4. Deploy. Open the generated deployment URL and confirm the Worker response is returned.
5. In the new Vercel project, open **Settings → Domains**, add `learn.automivex.com`, and view the domain's DNS instructions.
6. Copy the exact record type, name, target/value, and any generated verification record shown by Vercel. Do not guess a CNAME target.

See Vercel's [domain setup documentation](https://vercel.com/docs/domains/working-with-domains/add-a-domain).

## 3. Add only the requested record in Hostinger

In Hostinger's DNS zone for `automivex.com`, add the record Vercel requests. For the expected subdomain routing record, the template is:

| Type | Name | Target | TTL |
|---|---|---|---|
| CNAME | `learn` | **Paste the exact CNAME target shown by this Vercel project** | Default |

If Vercel also requests domain ownership verification, add the generated TXT record with the exact name and value shown in the Vercel dashboard. This is a conditional additional verification record, not a substitute for the CNAME.

Keep Hostinger as the authoritative DNS provider and leave the existing root-domain A record, `www` CNAME, MX records, SPF, DMARC, DKIM, and every other existing DNS record unchanged. Do not replace nameservers or migrate the DNS zone to Cloudflare. Avoid creating conflicting records with the name `learn`; if one already exists, inspect it before making a change.

## 4. SSL and final verification

After DNS propagation:

1. In Vercel **Settings → Domains**, confirm `learn.automivex.com` is configured correctly and the certificate/SSL status is active.
2. Check DNS resolution for `learn.automivex.com` and confirm it follows the Vercel-provided record.
3. Visit `https://learn.automivex.com/` and compare the result with the Worker URL.
4. Test a nested path and query, for example `https://learn.automivex.com/api/jobs/123?check=1`.
5. Test application flows and supported methods (GET, HEAD, POST, PUT, PATCH, DELETE), including a JSON POST and authentication. Confirm the browser address stays on `learn.automivex.com` throughout.
6. Inspect response status, headers, cookies, and any `Location` header. Check Vercel deployment logs and the Worker response separately when a request fails.

A deployment preview can validate routing before attaching the custom domain. Production behavior is not confirmed until this project is deployed and these checks are run against it.

## Compatibility and limits

This repository only defines Vercel routing. The existing Worker's source was not provided, so its route behavior, authentication assumptions, redirects, cookie settings, CORS policy, body-size handling, response streaming, and large-payload handling cannot be inspected or certified here.

- **Redirects:** A rewrite does not change the address bar, but if the Worker responds with a redirect whose `Location` points at `workers.dev`, browsers can navigate there. Inspect redirects. If that occurs, adjust the Worker's redirect URL/origin handling to use `https://learn.automivex.com`; do not silently rewrite `Location` here.
- **Origin-sensitive logic:** Vercel forwards visitor-identifying headers such as `x-forwarded-host` and `x-forwarded-proto`. The Worker may still see upstream `Host` semantics that differ from direct custom-domain service. Verify canonical host checks, callback URLs, absolute links, CSRF and OAuth allowed origins, and auth redirect URI registration in the Worker/provider.
- **Cookies:** Verify `Domain`, `Path`, `Secure`, `SameSite`, and expiry attributes. A cookie scoped to `workers.dev` will not authenticate on `learn.automivex.com`; Worker-side cookie configuration may need to set an appropriate public-host cookie. No cookie rewriting is performed here.
- **CORS:** Same-origin browser calls to `learn.automivex.com` generally avoid cross-origin CORS; other origins still need the Worker to return the appropriate policy. Check preflight requests where used.
- **Methods and bodies:** External rewrites are intended to proxy requests, but test each method, JSON/body forwarding, authorization, and any Worker-specific handling. Platform request/response limits apply; this config does not add buffering or streaming code.
- **Caching:** The config sends Vercel's documented opt-out header on all matched paths. The Worker itself or other infrastructure may still apply its own caching behavior.
- **Authentication:** Any OAuth/OIDC application must allow the public hostname and correct callback URLs. Change those settings in the Worker or identity provider only as a separately reviewed action.

Do not claim compatibility for a behavior until it has been tested against the deployed Worker and Vercel project.

## Project files

- `vercel.json` — fixed-origin wildcard external rewrite and explicit external rewrite cache opt-out.
- `.gitignore` — excludes local dependencies, Vercel metadata, and environment files.
- `README.md` — this deployment and verification guide.
