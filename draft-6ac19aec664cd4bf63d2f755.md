---
title: "From Help Desk to Cloud: What Building My Resume on Azure Actually Taught Me"

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