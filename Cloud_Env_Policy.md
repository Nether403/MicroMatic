<!--
Cloud Allow-List (enable selectively):
  - registry.npmjs.org (GET, HEAD) for pnpm installs
  - github.com, api.github.com (GET, HEAD) for PR diffs
  - api.neon.tech (POST, GET) for database provisioning

HTTP verbs permitted:
  - GET, HEAD, OPTIONS globally
  - POST limited to api.neon.tech when provisioning
  - PUT/PATCH/DELETE disabled by default

Container caching:
  - Reuse pnpm store between runs
  - Cache Neon CLI binaries & scaffold templates

Default: network disabled; require explicit justification before toggling.
-->
