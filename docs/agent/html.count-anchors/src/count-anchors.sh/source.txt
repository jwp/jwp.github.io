#!/bin/sh
# Count the anchor tags by forcing the open tags into their own line
# and counting the rewritten form with `grep -c`.
# Primarily used to validate the processing performed by other tools.

exec sed -E -n 's/<[[:space:]]*[aA][[:space:]]+/\
<<<A\
/g;p' | \
grep -c '^<<<A$'
