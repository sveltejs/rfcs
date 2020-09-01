- Start Date: 2020-08-31
- RFC PR: 
- Svelte Issue: 

# Sapper as Middleware

## Summary

The Sapper build process should emit a mostly self-contained blob of javascript (and supporting files) that
implements all server-side routes, SSR, and DOM code for the entire application. This javascript should
export (at least) two functions:

- A `middleware(req, res)` function that will serve a dynamic request.
- A `assets(req, res)` function that will serve a static asset request.

The supporting files *must* also include a manifest of files that can be uploaded to a CDN.

This javascript *must not* start an Express server.

## Motivation

Working with Sapper in production for a large scale project has identified a number of
challenges that arise from the unintended complexity of stale assumptions.

By reducing these assumptions to the *absolute minumum*, we aim to:

- Make it easier to deploy Sapper in efficient, scalable configurations
- Make it easy to use Sapper applications in more scenarios than just a single standalone site
- Make it easier to extend Sapper to suit specific project needs (without requiring modifications of Sapper itself)

As of 0.28, the Sapper build process emits a completely standalone directory in `__sapper__/build`
that starts an Express server listening on `$PORT` to serve the Sapper site.

To do this Sapper makes many nuanced assumptions about the dependencies and runtime environment of
applications it is building. Many of these assumptions are unnecessary sources of complexity during
both developent and deployment.

## Detailed design

The general behavior of client-side bundling for Sapper should remain unchanged. Components are compiled for
each route as SSR code and as DOM code. The behavior of the DOM (client-side) entry point is also unchanged.
We expect the client-side bundle to be just as completely self-contained as it is today.

This is in contrast with the server-side bundling operation. Instead of emitting a completely free-standing
NodeJS script that starts an Express server, it would instead emit a partially bundled .js file that the user
should include in a script of their own. This server-side .js file would look something like:

```javascript

export function middleware(req, res) {
  // Sapper routing logic here to handle dynamic requests ...
}

export function assets(req, res) {
  // sirv-based logic here to serve requests for static assets ...
}

```

There must also be a manifest that's emitted to make deployment as straightforward as possible. A simple
manifest might look something like:

```json
{
  "assets": {
    "directory": "assets",
    "entries": {
      "client/index.5bdaeb21.js": {
        "headers": [
          {
            "key": "Cache-Control",
            "value": "max-age=31536000, immutable"
          },
          {
            "key": "Content-Type",
            "value": "application/javascript"
          }
        ]
      },
      "index.css": {
        "headers": [
          {
            "key": "Content-Type",
            "value": "text/css"
          }
        ]
      }
    }
  },
  "functions": {
    "index.js": {}
  }
}
```

Bringing everything together, the emitted directory structure should look something like:

```
assets/
  client/
    index.5bdaeb21.js
  index.css
index.js
index.d.ts
manifest.json
```

A few important details of this layout:

- All static assets, including user-supplied (from `static/` directory) and Sapper-generated
  (from compilation of Svelte components) are under a single directory prefix
  (`assets/` in the example above)
  
- `manifest.json` is intended to only be used at deployment and development time, not *necessarily* at
  run-time. (This is unlike `build.json` in Sapper 0.28, which is used only at run-time)
  
- Users must use/mount the exported `middleware(req, res)` function in an Express-compatible webserver.
  The specific contract we expect for `req` and `res` parameters would be described by the types emitted
  in `index.d.ts`.
  
  Compatibility libraries should be written to adapt popular serverless environments to these contracts.

- Users are *not required* to use the exported `assets(req, res)` function in `index.js`. They can also
  provide their own asset serving logic based on the information in `manifest.json`.

## How we teach this

Updating the existing template projects should probably be enough for simple projects. We should also
add additional example projects that deploy Sapper applications in more scenarios.

## Drawbacks

Together, these changes represent a breaking change to Sapper's external contract. It's possible that
Sapper *never* wants to support the developer experience that's outlined here.

## Alternatives

I've previously used a series of RegExps to pull routes out of the generated code a "manually" built this
architecture with sed and shell script hackery.

## Unresolved questions

### Are people okay dropping Webpack support to make the above easier to implement?

Support for Webpack increases the surface area for configuration and testing and may not be worth the trouble.

### How satisfied are people with the current experience of `sapper dev`?

Making Sapper emit middleware would definitely affect how reloading dev experiences operate!

### What deployment targets do people care about?

To start with, we expect Sapper apps to be trivially deployable to at least:

- Cloudfront + AWS Lambda
- Vercel
- Vanilla NodeJS

And hopefully also other targets like:

- Cloud Functions for Firebase
- Cloudflare Workers

Others?

