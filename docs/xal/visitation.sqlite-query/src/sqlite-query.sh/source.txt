#!/bin/sh
# Join links with the visitation data in the given database using the query class.
# [ Parameters ]
# /1/
	# The sqlite database file to query.
# [ Effects ]
# Reads links (sole column) from standard input and writes the joined visitation
# data to standard output.
##
TAB="$(printf '\t')"

if test $# -ne 2
then
	echo >&2 "ERROR: " \
		"requires exactly two arguments, $# given," \
		"the query class and the sqlite database file path."
	exit 1
fi
QUERYCLASS="$1"
SDF="$2"
shift 2

# Load RI set, join against visitation tables, and sequence to standard out.
cut -f1 -d "$TAB" | \
sqlite3 "$SDF" \
	".separator '$TAB'" \
	".read $XAL_CONTEXT/visitation/${QUERYCLASS}-setup.sql" \
	'.import /dev/stdin xvq' \
	".read $XAL_CONTEXT/visitation/${QUERYCLASS}-join.sql"
