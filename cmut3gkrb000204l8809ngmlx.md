---
title: "From Help Desk to Cloud: What Building My Resume on Azure Actually Taught Me"
datePublished: 2026-10-04T00:40:11.562Z
cuid: cmut3gkrb000204l8809ngmlx
slug: from-help-desk-to-cloud-what-building-my-resume-on-azure-actually-taught-me
ogImage: https://cdn.hashnode.com/uploads/og-images/6ac1452a5bd708f37dc9ed49/894678ee-e8eb-4ba0-8d76-b50ca1503aba.png

---

## Why I took on the Cloud Resume Challenge

I've spent 13 years in IT. I started out while I was in the Army and come to find out I was the only one in my unit who was tech savvy and knew how to work on computers then I decided this is what I wanted to do for my career. I got my CompTIA A plus certification and landed my first help desk job. Today I'm a Tier 2 analyst supporting a law firm's Microsoft environment: Entra ID, Intune, Group Policy, and a couple hundred endpoints.

I know how to keep an environment running. What I wanted was proof that I could build one in the cloud, and secure it, from nothing. I'm working toward cloud administration and security, studying for the AZ-900 and Security+, and practicing pentesting on the side.

So I took on the [Cloud Resume Challenge](https://cloudresumechallenge.dev/docs/the-challenge/azure/): host your resume on Azure, add a visitor counter backed by a database and an API, define the infrastructure as code, and deploy it all automatically. The result is live at [leemcnutt.dev](http://leemcnutt.dev).

This post isn't a tutorial. It's the list of things that broke, what they taught me, and the security decisions I made along the way.

## Lesson 1: The guide was already out of date

Step 5 of the challenge says to use Azure CDN for HTTPS. That service is gone for new customers. Microsoft stopped allowing new Azure CDN (classic) instances in October 2025, and it retires completely in 2027. The replacement is Azure Front Door.

I hadn't even started Front Door before I hit my next error: Microsoft.Cdn is not registered for the subscription. Every Azure service is delivered by a resource provider, and a subscription has to be registered with a provider before you can use it. Storage had registered itself automatically. Front Door hadn't. One click under Subscriptions > Resource providers fixed it.

**What I learned:** cloud documentation ages fast, so check what still exists before you build on it. And when Azure says something isn't registered, the error message tells you exactly which provider to turn on.

## Lesson 2: DNS taught me about subdomain takeover

I registered [leemcnutt.dev](http://leemcnutt.dev) at Namecheap and hosted its DNS in Azure DNS. That meant replacing Namecheap's name servers with Azure's four. That's **delegation**: the registrar holds the registration, but whoever the NS records point to controls the records.

The root domain was the tricky part. DNS doesn't allow a CNAME at the apex ([leemcnutt.dev](http://leemcnutt.dev) itself), because the root also has to hold NS and SOA records. Azure DNS solves this with an **alias record**, which points the root directly at an Azure resource. I used alias records for www too. If the target resource is ever deleted, an alias record stops answering instead of pointing at a name someone else could claim.

Then I found a record I didn't create: cdnverify.www, pointing at an [azureedge.net](http://azureedge.net) name. A setup wizard had added it for the old CDN's validation method. That name wasn't a resource I owned. A DNS record pointing at something you don't control is the textbook setup for **subdomain takeover**: if an attacker can create a resource with that name, they can serve content on your domain. I deleted it.

**What I learned:** audit your DNS zone for records you can't explain. Dangling records are one of the most common findings in real-world recon.

## Lesson 3: The certificate error that wasn't a misconfiguration

The Azure portal said my Front Door certificate was deployed. Chrome said NET::ERR\_CERT\_COMMON\_NAME\_INVALID. Instead of guessing, I asked the server directly what certificate it was presenting:

openssl s\_client -connect www.leemcnutt.dev:443 -servername [www.leemcnutt.dev](http://www.leemcnutt.dev) </dev/null 2>/dev/null \\  
  | openssl x509 -noout -subject -issuer -dates

The subject came back as CN=\*.[azureedge.net](http://azureedge.net). That's Microsoft's default edge certificate, not mine. The certificate was valid; it just didn't cover my domain name.

Nothing was misconfigured. Front Door's edge servers hadn't received my new configuration yet. Front Door hosts thousands of sites on shared servers, and it picks the right certificate using **SNI**, the hostname the browser sends during the TLS handshake. Until the config arrived, the edge didn't know [www.leemcnutt.dev](http://www.leemcnutt.dev), so it fell back to its default. I wrote a small loop to re-check every minute, and the subject flipped to my domain about half an hour later.

**What I learned:** read the evidence before changing anything. openssl s\_client shows exactly what a server presents, with no browser in the way. It's now one of my go-to troubleshooting and recon tools.

## Lesson 4: A visitor counter has a race condition

A counter sounds trivial: read the number, add one, save it. But picture two visitors arriving at the same instant. Both read 5, both add one, both save 6. One visit is lost. That's a **race condition**.

The fix is **optimistic concurrency** using an ETag. Every entity in Cosmos DB carries an ETag, a version stamp that changes on every write. My Python function saves the new count only if the ETag still matches what it read. If someone else wrote in between, Azure rejects the save, and the function re-reads and tries again, up to five times.

You can't reliably make two real visitors collide, so I tested it with a fake. The function receives its database client as a parameter, which let my pytest suite pass in a FakeTableClient that keeps the counter in memory and can pretend to lose the race on demand. Eleven tests cover the normal increment, the retry, giving up cleanly, a brand-new empty table, and error handling. They run in under a second with no Azure connection.

One of those tests is a security test. It forces a database error containing a fake account key, then checks that neither the key nor a stack trace appears in the API response. If anyone "improves" the error handling later by returning the exception text, that test fails and blocks the deployment.

**What I learned:** design code so its dependencies can be swapped out, and write tests for your security controls, not just your features.

## Lesson 5: Infrastructure as code isn't the portal in a file

I built the backend by hand in the portal first, so I understood every setting. Then I rewrote it in Bicep, which compiles to the ARM templates the challenge asks for. One file now creates 13 resources: Cosmos DB, the Function App, its storage, monitoring, a managed identity, and the role assignments between them.

Three things surprised me:

•         **Control plane versus data plane.** Bicep can create the Cosmos DB table, but not the data inside it. A fresh deployment had no counter, so the first visit failed. I changed the function to create the counter if it's missing, with a test for two first visitors arriving at once.

•         **The identity had a chicken-and-egg problem.** In the portal I used a system-assigned identity. In a template, that identity doesn't exist until the app is created, so the app can start before its permissions do. A user-assigned identity is its own resource, so the template creates it, grants its roles, and only then creates the app.

•         **The portal applied a secure default my template didn't.** My portal-built app got a long, random hostname that prevents subdomain takeover if the app is ever deleted. My Bicep version got a plain one. One line, autoGeneratedDomainNameLabelScope: 'TenantReuse', fixed it. I only noticed by comparing the two side by side.

I deployed the template to a throwaway resource group first, ran az deployment group what-if before every real deployment, and then migrated the live site onto the Bicep-managed backend and deleted the hand-built one.

**What I learned:** IaC makes your environment repeatable, but only as secure as what you write down. Compare it against a known-good build.

## Lesson 6: OIDC failed, and the error was a security feature

The challenge warns you not to commit Azure credentials to GitHub. I went further: my pipelines don't store an Azure secret anywhere, not even in GitHub's encrypted secrets. They use **OpenID Connect (OIDC)**. GitHub issues each workflow run a short-lived signed token, and Entra ID trusts it only if its subject matches a federated credential I created for my repo's main branch.

My first deployment failed with AADSTS700213: No matching federated identity record found. The error included the subject GitHub actually sent, so I compared it to what I'd configured:

•         **What I configured:** repo:mcnizzle02/cloud-resume-backend:ref:refs/heads/main

•         **What GitHub sent:** repo:mcnizzle02@<account ID>/cloud-resume-backend@<repo ID>:ref:refs/heads/main

GitHub had added numeric IDs for my account and repo. Names can be reused; IDs can't. Without them, if I ever deleted my repo, someone could create one with the same name and their workflows would produce the exact subject Azure trusts. That attack is called **repo jacking**. I updated the federated credential to the ID-based subject, re-ran the job, and it deployed.

**What I learned:** most federation failures come down to one value not matching exactly. When an auth error tells you what it received, compare it character by character.

## Lesson 7: Least privilege, all the way down

Each repo's pipeline has its own identity in Entra ID, and each gets only what its job requires.

<table style="min-width: 100px;"><colgroup><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p>Pipeline</p></td><td colspan="1" rowspan="1"><p>Permission</p></td><td colspan="1" rowspan="1"><p>Scope</p></td><td colspan="1" rowspan="1"><p>Why this narrow</p></td></tr><tr><td colspan="1" rowspan="1"><p>Backend</p></td><td colspan="1" rowspan="1"><p>Contributor</p></td><td colspan="1" rowspan="1"><p>One resource group</p></td><td colspan="1" rowspan="1"><p>Deploys Bicep, but can't touch anything else in the subscription</p></td></tr><tr><td colspan="1" rowspan="1"><p>Backend</p></td><td colspan="1" rowspan="1"><p>Role Based Access Control Administrator, with a condition</p></td><td colspan="1" rowspan="1"><p>One resource group</p></td><td colspan="1" rowspan="1"><p>Can assign only the two roles my template uses; it can't make anyone Owner</p></td></tr><tr><td colspan="1" rowspan="1"><p>Frontend</p></td><td colspan="1" rowspan="1"><p>Storage Blob Data Contributor</p></td><td colspan="1" rowspan="1"><p>The $web container only</p></td><td colspan="1" rowspan="1"><p>Can upload website files, nothing else in the storage account</p></td></tr><tr><td colspan="1" rowspan="1"><p>Frontend</p></td><td colspan="1" rowspan="1"><p>Custom "Front Door Cache Purger" role</p></td><td colspan="1" rowspan="1"><p>The Front Door profile</p></td><td colspan="1" rowspan="1"><p>Can purge the cache, but not change routes, domains, or certificates</p></td></tr></tbody></table>

The second row was the hardest decision. My Bicep template creates role assignments, which plain Contributor can't do. The easy answer was Owner, but a hijacked pipeline with Owner could grant anyone anything. Instead I used **constrained delegation**: a role condition that only allows assigning Storage Blob Data Owner and Monitoring Metrics Publisher.

The last row exists because the closest built-in role, CDN Profile Contributor, could rewrite my Front Door setup. My custom role has four actions: three reads and the purge.

**What I learned:** separate identities mean separate blast radiuses. If my frontend pipeline were compromised, the attacker could change my website files and nothing more.

## Lesson 8: The mistakes I caught in my own environment

Two of the most useful moments in this project were my own mistakes.

**Standing elevated access.** While cleaning up old role assignments, I found a warning: my account had **User Access Administrator at root scope**, marked permanent. That role can grant anyone any role on anything in the tenant. It came from a Global Admin setting, "Access management for Azure resources," that I'd switched on at some point and never switched off. I didn't need it, since I'm already Owner on my subscription. I turned it off. It's also a well-known attack path: an attacker who compromises a Global Admin flips that same toggle to jump into every Azure subscription.

**A key in a screenshot.** While troubleshooting my local function, I shared a screenshot with my editor open in the background. Part of my Cosmos DB account key was visible. Partly visible still counts as exposed, so I rotated it: regenerated the key, redeployed my Bicep template (which reads the key at deploy time and updates the Function App automatically), and updated my local settings. My counter was down for about a minute.

**What I learned:** mistakes happen. What matters is noticing them and responding the way you'd want a teammate to. Now I close anything with secrets in it before I take a screenshot.

## Security decisions at a glance

•         **No stored secrets.** Both pipelines authenticate with OIDC, and the Cosmos DB key is read at deploy time, never written in code.

•         **Keyless where possible.** The Function App's storage has account keys disabled entirely, and Application Insights requires identity-based authentication.

•         **Managed identity** with only Storage Blob Data Owner and Monitoring Metrics Publisher.

•         **Capped scaling** so a flood of requests can't run up my bill (a denial-of-wallet attack), backed by a budget alert.

•         **CORS limited** to my two domains, with credentials disabled. CORS stops browsers, not scripts, so it's one layer, not the whole defense.

•         **Generic error messages** that never return exception details, with a test that enforces it.

•         **An allowlist for published files.** The frontend pipeline copies only .html, .css, and .js files, so my README and repo internals never reach the public site.

•         **TLS 1.2 minimum** everywhere, plus HTTPS-only.

•         **Explicit cache headers** (Cache-Control: public, max-age=300) instead of letting browsers guess.

## What I'd do next

This is a portfolio project, not a production system. Here's what I'd change to close that gap:

•         **Managed identity for Cosmos DB,** so the last connection string disappears.

•         **Private endpoints** for Cosmos DB, so the database isn't reachable from the internet at all, even with a valid key.

•         **A WAF in front of the API** for rate limiting and request filtering.

•         **Unique visitors.** Right now every page load counts, including my own refreshes. Counting unique visitors means cookies or IP tracking, which brings privacy questions worth thinking through.

•         **Apex certificate renewal.** Front Door's managed certificate for [leemcnutt.dev](http://leemcnutt.dev) may need manual revalidation every few months, because the root domain uses an alias record instead of a CNAME. Automating that check is on my list.

## If you're starting the challenge

Build it in the portal first, then rebuild it in code. Clicking through every setting taught me what each one does, and that made the Bicep file make sense. When something breaks, read the error twice: almost every problem in this post was solved by an error message that told me exactly what was wrong. And treat security as part of the build, not a coat of paint at the end.

Thirteen years of keeping systems running gave me a head start on troubleshooting. This project gave me the cloud and security side to go with it.

•         **Live site:** [leemcnutt.dev](http://leemcnutt.dev)

•         **Frontend repo:** [github.com/mcnizzle02/cloud-resume-frontend](https://github.com/mcnizzle02/cloud-resume-frontend)

•         **Backend repo:** [github.com/mcnizzle02/cloud-resume-backend](https://github.com/mcnizzle02/cloud-resume-backend)

If you're working through the challenge or hiring for cloud and security roles, I'd love to connect on LinkedIn.