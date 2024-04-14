#!/bin/sh
# Retrieve the stars using (system/executable)`gh` and correct the run-on json
# emitted by (command/option)`--paginate`. Expects logged in status.
#
# [ Parameters ]
# /GH/
	# The `gh` command to use.
#
# [ Returns ]
# Exits with the status returned by `gh`.
#
# [ Effects ]
# Emits JSON data to standard out.
##
SCRIPT="$0"

GH=gh
if test $# -gt 0
then
	GH="$1"
	shift 1
fi

# Required for starred_at time field.
ACCEPT='application/vnd.github.star+json'

# Insert commas, and wrap in brackets.
echo '['
"$GH" api --paginate -H "Accept: $ACCEPT" '/user/starred' | \
	sed 's/]\[/],[/g'
X=$?
echo ']'
exit $X
