#!/bin/sh
# Invoke (html) tidy with the &truncate configuration.
SCRIPT="$0"
SD="$(dirname "$SCRIPT")"
XAL="$(cd "$SD" && pwd)"
TIDYCONFIG="$XAL/truncate.i"
exec tidy -config "$TIDYCONFIG" "$@"
