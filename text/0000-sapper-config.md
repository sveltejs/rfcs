- Start Date: 2020-05-21
- RFC PR: (leave this empty)
- Svelte Issue: (leave this empty)

# Sapper Configuration

## Summary

Place Sapper configuration in a `.config.js` file

## Motivation

There are a number of PRs or open issues for the Sapper project that would require providing a configuration option. We should handle configuration in a well-thought out and consistent manner. Deciding this would unblock review on a large number of Sapper PRs.

[language-tools already uses a `svelte.config.js`](https://github.com/sveltejs/language-tools/blob/3139ccd5b4306a914be4ba64b0c036fcd8888f78/packages/svelte-vscode/README.md#generic-setup) and [Svelte core was considering adopting it more widely](https://github.com/sveltejs/svelte/issues/1101).

### Sapper PRs that do some configuration

#### Middleware option
* https://github.com/sveltejs/sapper/pull/1215
* https://github.com/sveltejs/sapper/pull/1001
* https://github.com/sveltejs/sapper/pull/1001
* https://github.com/sveltejs/sapper/pull/994
* https://github.com/sveltejs/sapper/pull/984
* https://github.com/sveltejs/sapper/pull/960
* https://github.com/sveltejs/sapper/pull/953

#### Server middleware and client `start`
* https://github.com/sveltejs/sapper/pull/766

#### Separate middleware
* https://github.com/sveltejs/sapper/pull/1152 / https://github.com/sveltejs/sapper/pull/1037 (stick user-provided functions on `result` in a middleware)

#### CLI arg
* https://github.com/sveltejs/sapper/pull/1075
* https://github.com/sveltejs/sapper/pull/1021
* https://github.com/sveltejs/sapper/pull/1020
* https://github.com/sveltejs/sapper/pull/932
* https://github.com/sveltejs/sapper/pull/866
* https://github.com/sveltejs/sapper/pull/859
* https://github.com/sveltejs/sapper/pull/856
* https://github.com/sveltejs/sapper/pull/813

#### Environment variable
* https://github.com/sveltejs/sapper/pull/482

#### Doesn't use configuration, but probably could
* https://github.com/sveltejs/sapper/pull/985
* https://github.com/sveltejs/sapper/pull/921

## Detailed design

We would use a `.config.js` because it's inline with the existing `svelte.config.js`
used by `language-tools` and also because it's more flexible. For instance, it allows
providing types that could not be specified in YAML such as `function`.

The configuration file should be completely optional with sensible defaults.

### Format

I propose a `sapper.config.js` file with the following structure:

```
{
  bundler,      // Specify a bundler (rollup or webpack, blank for auto)
  cwd,          // Current working directory  (default .)
  ext,          // Custom Route Extension  (default .svelte .html)
  legacy,       // Create separate legacy build
  output,       // Sapper intermediate file output directory  (default src/node_modules/@sapper)
  port,         // Default of process.env.PORT  (default 3000)
  routes,       // Routes directory  (default src/routes)
  src,          // Source directory  (default src)
  middleware: {
	ignore      // Requests to ignore. Array | RegExp | function | 
	session     // Session creation function. (req: Req, res: Res) => any
  },
  dev: {
    buildDir,   // Development build directory  (default __sapper__/dev)
    devPort,    // Specify a port for development server
    hot,        // Use hot module replacement (requires webpack)  (default true)
    live,       // Reload on changes if not using --hot  (default true)
    open,       // Open a browser window
    port,       // Specify a port
    static,     // Static files directory  (default static)
  },
  export: {
    basepath,   // Specify a base path
    build,      // (Re)build app before exporting  (default true)
    buildDir,   // Intermediate build directory  (default __sapper__/build)
    concurrent, // Concurrent requests  (default 8)
    entry,      // Custom entry points (space separated)  (default /)
    host,       // Host header to use when crawling site
    timeout,    // Milliseconds to wait for a page (--no-timeout to disable)  (default 5000)
  }
}
```

The initial implementation would not concern the client-side application. Only one of the pending
PRs touched client-side config. It could easily be extended to include client configuration in the 
future, however.

### Precedence

There should be a consistent precedence hierarchy.

For the CLI tool:
* flags
* env vars
* global config
* config namespaced to CLI command (`build`, `dev`, or `export`)
* default value

CLI flags would be converted from dashed `option-name` to camelcase `optionName`.
ENV vars would be converted from capitalized `OPTION_NAME` to camelcase `optionName`.

For the middleware:
* options argument in middleware constructor
* `middleware` configuration
* default value

Flags and environment variables may be able to be supported in the middleware for simple values in
the future. However, the two existing options are both functions, which could not be specified in this
manner and so there currently is not a need to support this.

## How we teach this

This feature would be relatively easy to teach. We would update the Sapper docs
and template projects. Putting all configuration in a single place would improve
teachability over the current state. The proposal is backwards compatible, which
reduces education needs.

## Drawbacks

It's possibly yet another configuration file in people's projects depending on implementation.

There would now be multiple places configuration could be specified. That may confuse some new users.

It may become a harder to make changes to expected configuration if external tools are using the file format.

## Alternatives

### Status Quo

We could continue to specify options only as objects in the middleware constructor 
and command line flags for the CLI tool like we're doing today. There are a few
drawbacks with doing this.

There are many options such as the location of the source directory that must be
specified with each CLI command. It can become complicated to provide a long-list
of command line flags. Users will end up creating shell scripts to save their
configuration. Shell scripts are not as easily maintained as `.config.js` for most
JavaScript develoeprs.

Also, sharing with other tools is harder. Configuration that's in a standardized
location in a `.config.js` can easily be read by multiple tools and libraries.
[Prettier reported](https://github.com/sveltejs/svelte/issues/1101#issuecomment-357324492)
that putting off creation of a config file meant tools developed their own incompatible
methods for dealing with these question, which became a net negative.

## Unresolved questions

* Should Sapper's config go inside `svelte.config.js` under a `sapper` namespace or be a new `sapper.config.js` file?
  * There may be some config like the `legacy` option which is used by both tools. I don't know if Sapper does anything special with this today or it's just a straight passthrough to Svelte, but you could imagine Sapper doing extra work if this flag was detected. You probably don't want to have to set it twice. Should Sapper just defer to a `svelte.config.js` for this value or should we combine the files?
* Do we want all middleware configuration to be able to be provided by the `.config.js`?
  * It's a little awkward to specify a function taking a request and response in the config file if you're using TypeScript because you would need to import those types. It also just doesn't seem like where it should be done. Still it seems better for consistency that we allow all options to be specified that way. It makes it easier to understand for users and easier to implement. Users can simply provide the options via middleware constructor instead of config file for the ones where they prefer to do that
