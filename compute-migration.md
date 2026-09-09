# Shared compute migration

Installed `getcolors/once` revision `173823ebdf6b90145db632e73ef8d8388cd93174`. Root launchers match the
installed skill payloads from a verified Skills CLI installation.
The existing lockfile records that installation.

Compute now uses colors-compute shared and node state under
`<profile>/compute/`. Application stages retain their existing ownership.
This refresh does not transfer resource ownership or apply infrastructure.
No live state, private credentials, or private key contents were read.

Before create, preserve the old state, inventory the existing resources and SSH
key ownership, and review explicit source/destination resource mappings and a
plan with no unintended replacement. Recognized legacy remote state causes the
library to refuse the operation. Do not discard old state or bypass that check.
The committed deployment profile and destroy protection are preserved.

Validation: the published blue, green, red launchers built the desired state
in temporary directories with a sanitized environment. Compute documents were
present and rendered backend configuration contained no credentials. Offline
builds do not establish live authentication, migrated state, or application health.

The external provider key reference is preserved. Verify operator SSH agent
access, or configure ssh-private-key-path when an explicit identity is needed.
