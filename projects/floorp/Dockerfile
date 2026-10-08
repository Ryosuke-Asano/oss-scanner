# syntax=docker/dockerfile:1
FROM denoland/deno:bin-2.9.6 AS deno
FROM node:22-bookworm-slim

COPY --from=deno /deno /usr/local/bin/deno
ENV DEBIAN_FRONTEND=noninteractive \
    DENO_DIR=/opt/deno-cache \
    DENO_TLS_CA_STORE=system \
    MOZ_HEADLESS=1

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git python3 build-essential pkg-config procps \
      xz-utils unzip zip xvfb xauth fonts-liberation \
      libgtk-3-0 libdbus-glib-1-2 libasound2 libnss3 libgbm1 libxt6 libx11-xcb1 \
    && rm -rf /var/lib/apt/lists/*

# Scanner contract: keep the Floorp checkout and all build products in /src.
# The scanner adds Git history to the final image; avoid copying it into every
# intermediate build layer as well.
COPY --exclude=.git . /src
WORKDIR /src

# proxy_ca is optional, for local checks behind a TLS-intercepting proxy.
# The scanner itself does not need it, and the certificate is not stored in the image.
RUN --mount=type=secret,id=proxy_ca <<'SH'
set -eu
if [ -f /run/secrets/proxy_ca ]; then
    export DENO_CERT=/run/secrets/proxy_ca
    export NODE_EXTRA_CA_CERTS=/run/secrets/proxy_ca
    export GIT_SSL_CAINFO=/run/secrets/proxy_ca
fi
deno install --frozen --allow-scripts

# Cache build and test drivers before the scanner loses Internet access.
deno cache --frozen tools/feles-build.ts tools/src/colocated_test_runner.ts tools/scripts/xhtml.ts
deno test --no-run --allow-read --allow-write tools/src/

# Fetch the public native Runtime selected by floorp-runtime.lock.json.
# The project installer verifies the release asset's identity and SHA-256.
# Gecko is prebuilt; this checkout's privileged modules and UI are built below.
deno run -A tools/runtime-lock/runtime_lock_cli.ts install-native
deno eval '
    import * as Patcher from "./tools/src/patcher.ts";
    import * as Pref from "./static/gecko/pref/pref.ts";
    Patcher.run("apply");
    Pref.run();
'
NODE_ENV=production deno task feles-build build --phase before-mach
deno eval '
    import * as Injector from "./tools/src/injector.ts";
    Injector.run("stage");
    await Injector.injectXhtmlFromTs({ allowBrowserHttpLoader: false });
'

# Collect the locked upstream browser-test fixtures without checking out Gecko.
# Git's partial clone fetches only the blobs requested by the test collector.
python3 - <<'PY'
import json
import subprocess

source = json.load(open("floorp-runtime.lock.json"))["source"]
runtime_dir = "/opt/floorp-runtime-tests"
subprocess.run(["git", "init", runtime_dir], check=True)
subprocess.run(["git", "-C", runtime_dir, "remote", "add", "origin",
                "https://github.com/" + source["repository"] + ".git"], check=True)
subprocess.run(["git", "-C", runtime_dir, "fetch", "--depth=1", "--filter=blob:none",
                "origin", source["commit"]], check=True)
subprocess.run(["git", "-C", runtime_dir, "update-ref", "HEAD", source["commit"]], check=True)
tree = subprocess.check_output(["git", "-C", runtime_dir, "rev-parse", "HEAD^{tree}"], text=True).strip()
assert tree == source["tree"], "Runtime source tree does not match the lock"
subprocess.run(["deno", "task", "firefox-tests:collect", "--runtime-dir", runtime_dir,
                "--out", "_dist/firefox-tests", "--source-repo", source["repository"],
                "--source-ref", source["ref"]], check=True)
PY
deno task firefox-tests:triage-browser
deno task firefox-tests:prepare-browser

# Run the host suite online to prime any imports it loads dynamically.
# Keep the image usable for diagnosis if an existing test fails.
deno task test:host || echo "WARNING: Floorp host tests reported failures"
SH

# The runnable staged browser is /src/_dist/bin/floorp/floorp-bin.
# Browser tests start local Vite/Marionette services; no Internet is required.
CMD ["bash"]
