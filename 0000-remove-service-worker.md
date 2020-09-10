- Start Date: 2020-09-10
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Remove service worker support from Sapper

## Summary

Remove the built-in service worker support from Sapper. Most users don't need
service workers and should probably remove them from their applications. For the few
that do need service workers, we can encourage them to include the service worker
separately.

## Motivation

Service workers are primarily meant to support progressive web apps (PWAs), which
are essentially replacements for native mobile applications. Service workers can be
a valuable tool for building PWAs which work offline like a native mobile app would.
However, the vast majority of sites built with Sapper do not fit this profile. Thus,
including a service worker in the template is a very opinionated decision that is a
poor fit for the majority of our users.

The existing service worker support has shown several complications:

* They very aggressively cache files
    * Hard to diagnose issues can occur when deploying a new version of a site
    * The caching continue to occur even with a hard refresh (i.e. Ctrl+F5).
      Developers and users are not familiar with needing to unregister a service
      worker in the Application tab in Chrome
    * The caching happens across different sites during local development. If a
      developer works on one site running at the default localhost:3000 and then
      switches to working on another site that they also run on the default port
      then strange issues will appear where content from the other site is served
    * The application server can appear to be running when it is not
    * If a user has a large amount of files in the `/static` directory (e.g. over
      50mb) then the application can stop responding.
* The default service worker downloads all `/client` files
    * This can result in excessive bandwidth and usage charges for end users on
      mobile devices
    * This results in a worse score on Chrome's built-in Lighthouse performance
      scoring
* None or almost none of the core Sapper developers use service workers themselves.
    * Bugs go undiscovered for a long time as a result. E.g. service workers were
      broken by default until Sapper 0.28.0
    * It is more difficult to support users, answer their questions, etc.
* There are gaps in the existing documentation. E.g. developers frequently ask if
  they can remove the service worker, which is not covered in the documentation
* For the few users who do want to utilize service workers, it is much harder for
  them to utilize [Google Workbox](https://developers.google.com/web/tools/workbox)
  or other libraries and tools offering worker support with Sapper controlling the
  process.
* It makes the implemention of Sapper as middleware more difficult because the
  Sapper middleware today serves these extra static files. It would be a cleaner
  separation to have static serving and dynamic routes serving completely separated.

## Detailed design

Please see [the PR in Sapper](https://github.com/sveltejs/sapper/pull/1502).

## How we teach this

This should be included in a major release of Sapper and documented in the migration
guide. For most users, the best course of action will likely be to simply stop using
service workers. For the small number of users that have built PWAs with Sapper or
otherwise have a compelling reason to utilize a service worker, we should suggest
that they serve those files with `sirv` just as they do for static content.

## Drawbacks

This will cause a migration cost for existing Sapper developers.

## Alternatives

There are two main alternatives:
* the status quo
* remove the service worker from Sapper, but update the template to separately
  include a service worker

I think that neither of these alternatives is a particularly good idea since the
vast majority of sites do not need a service worker.

## Unresolved questions

N/A
