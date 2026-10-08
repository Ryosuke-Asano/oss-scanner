# Floorp threat model

## Project and scope

Floorp is a desktop web browser based on Firefox. The enrolled repository contains
Floorp's privileged TypeScript modules, browser chrome, internal pages, native
helpers, preferences, and patches to Firefox. The Gecko source and prebuilt native
Runtime are maintained separately in Floorp-Projects/Floorp-Runtime.

Focus on vulnerabilities introduced by Floorp's code, preferences, and patches.
The Dockerfile installs the public native Runtime selected and verified by
`floorp-runtime.lock.json`, applies Floorp's runtime patches, and builds Floorp's
production assets. It does not compile Gecko from source. A flaw in unchanged
upstream Firefox code should be reported to Mozilla; a Floorp change that exposes
or weakens an upstream security boundary is in scope.

This image runs Linux. Platform-specific macOS AppKit/Mach helpers and Windows
integration code remain available for source review, but their native build and
runtime validation require the corresponding operating system.

## Untrusted inputs and security boundaries

Web pages, frames, URLs, downloaded files, extension packages, imported settings,
and data obtained from external services are untrusted. Treat data crossing from
content processes, web content, extensions, or local HTTP clients into privileged
browser code as a security boundary. Internal pages must not expose privileged
operations to arbitrary web origins or accept unvalidated messages from them.

Important areas include:

- `browser-features/modules/actors/` and actor registration in
  `browser-features/modules/modules/BrowserGlue.sys.mts`: content/chrome IPC,
  principal checks, message validation, and operations on the correct tab/frame.
- `browser-features/modules/modules/os-server/`, `os-apis/`, and `os-automotor/`:
  local automation routes, authentication and origin checks, and access to tabs,
  cookies, downloads, files, and browser state. State whether the feature must be
  enabled and how a remote page or another local process can reach it.
- `browser-features/modules/modules/chrome-web-store/`: untrusted CRX/ZIP parsing,
  package identity, extension installation, and any path or privilege boundaries.
- `browser-features/modules/modules/pwa/`, `libs/pwa/`, and `native/`: application
  registration, profile selection, native helpers, command-line arguments, and
  filesystem operations involving data supplied by web applications.
- `browser-features/pages-*/` and `browser-features/chrome/`: script injection,
  rendering of imported or remote data, privileged internal pages, and unwanted
  changes to browser security settings.
- `bridge/`, `static/gecko/pref/`, `tools/patches/`, and
  `.github/patches/floorp-runtime/`: startup code, loader boundaries, CSP changes,
  and Floorp modifications to Firefox security behavior.

Use a normal user profile and the feature's shipped configuration when evaluating
reachability. Development/test settings and localhost Vite loaders are not the
production configuration. The image contains production assets without enabling
the browser HTTP loader. The browser test command deliberately creates a test
profile and development servers; do not confuse those settings with a shipped
vulnerability.

## Exercising the build offline

The checkout is `/src`. Production bundles and the staged browser are retained
under `/src/_dist/`; the Linux browser executable is
`/src/_dist/bin/floorp/floorp-bin`. Dependency caches are retained at
`/opt/deno-cache` and in the checkout's `node_modules`.

Run the host/tool tests with:

```sh
cd /src
deno task test:host
```

Run browser integration tests with a virtual display:

```sh
cd /src
xvfb-run -a deno task test
```

The browser runner automatically builds its test assets and starts a test browser,
Vite servers, and Marionette on loopback. These local services must remain usable
inside the offline container. Tests can be scoped, for example:
`xvfb-run -a deno task test -- --near browser-features/modules/modules/chrome-web-store`.
The locked upstream browser-test fixtures are collected during the online image
build. Production bundles, source code, caches, and the locked Runtime remain
available for reproduction and rebuilding.

## Reporting and severity

Prioritize exploitable crossings into browser chrome or native execution, access
to another origin's private data, unauthorized use of automation APIs, extension
installation bypasses, and unsafe filesystem writes. Rate a finding according to
demonstrated impact, default reachability, required user interaction, and whether
a feature is opt-in. Explain any remaining step between a primitive and the
claimed impact; a crash alone does not establish code execution.

Code execution or broad data theft triggered by an untrusted web page merits the
highest priority. Cross-origin data exposure, unauthorized privileged operations,
and writes outside an intended directory are serious even when they do not
demonstrate native code execution. Availability-only issues should clearly state
which process or data is affected and whether recovery is possible.

An explicitly installed privileged extension or user script already has the
capabilities the user granted it; report a violation of the intended boundary,
not the existence of that capability. Likewise, do not assume the attacker can
already edit the victim's application files or profile to demonstrate a remote
vulnerability.

Include a minimal reproducer, the affected feature and configuration, the
attacker's starting capabilities, the expected boundary, and a focused patch with
a regression test where practical. Reports go privately to `contact@floorp.app`.
