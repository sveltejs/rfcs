- Start Date: 2020-06-25
- RFC PR: 
- Svelte Issue: ]

# Sapper Builders

## Summary

This RFC proposes a method of modularising build output from Sapper for ease of deployment across a variety of targets.

## Motivation

Sapper's current build output comprises of a series of files:

1. A build manifest
1. A client bundle
1. A server bundle (in non-exported mode)
1. An (optional) legacy client bundle
1. Static files

Deploying this is relatively simple, but certainly not optimised for certain deployment strategies such as serverless (lambda), and often relies on third-party, platform-specific builders.

If Sapper were to produce its output in a more modular fashion, it would facilitate the creation of a number of officially supported "builders" for major platforms, as well as opening the doors for third parties to easily produce builders which targetted other platforms.

Additionally, it would ease the creation of some desirable features:

1. Per-route bundle splitting
1. [Differential rendering on a per-route basis](https://github.com/sveltejs/sapper/issues/1324) - Allowing SSR, SSG, SPA, [JAM](https://github.com/sveltejs/sapper/issues/1093), on a per-route basis.
1. [Faster cold-boot time for serverless functions](https://github.com/sveltejs/sapper/issues/356)
1. [SPA Mode](https://github.com/sveltejs/sapper/issues/383)

It would facilitate easier deployment on some new platforms:

1. [Azure Functions](https://azure.microsoft.com/en-gb/services/functions)
1. [AWS Lambda](https://aws.amazon.com/lambda/)
1. [Google Cloud Functions](https://cloud.google.com/functions)
1. [IBM Cloud Functions](https://www.ibm.com/uk-en/cloud/functions)
1. [IPFS](https://ipfs.io/)
1. [Begin.com](https://begin.com/)
1. [Netlify](https://www.netlify.com/)
1. [Cloudflare Workers](https://blog.cloudflare.com/introducing-cloudflare-workers/)
1. [Fastly Edge](https://www.fastly.com/products/edge-compute)

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

* Client-side bundle goes into static dir (SPA)
* Individual modules built for each route
* entrypoint imports individual route modules (for local development + simple `node` deployment)
* `@sapper/[architecture]-builder` packages
* Removing export (an application which has no dynamic routes is automatically SSG)

Discuss:

1. How do we determine which routes are:
  * SPA (No preload)
  * SSG (?)
  * Hybrid / SSR (Preload function exists) - current behaviour

2. Questions around treatment of server routes

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Svelte patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Svelte guides must be
re-organized or altered? Does it change how Svelte is taught to new users
at any level?

> How should this feature be introduced and taught to existing Svelte
users?

To define:

SSG: Server-side generated pages
SPA: Single-page App
SSR: Server-side Rendering

## Drawbacks

This approach provides many benefits, however one potential drawback is the additional complexity of moving from a two-bundle, encapsulated deployment, served up by a single javascript file `node /__sapper__/build/index.js` into a collection of files relating to routes. It's important that a builder exists (and is possibly the default) for running the simplest possible instance of the app, with just node and a http server installed.

## Alternatives

NextJS [already has a solution for this](https://github.com/vercel/next.js/issues/9524), borne of a constraint in the architecture, however it has proven itself as a highly desirable feature. We can draw inspiration from the way they have addressed this problem. It's also worth noting that their problem is understandably very targetted to users hosting on their own Vercel platform, whereas the Sapper solution should consider a range of platforms.

Builders are a feature that Vercel uses quite heavily. However they can be quite complicated due to the abstraction they work over (such as serving serverless GO, PHP, etc). Since we control the input, Sapper builders should be a simple mapping excercise in most cases.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?