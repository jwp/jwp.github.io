#!/bin/sh
# Join links with the visitation data in the given history database.
# [ Parameters ]
# /1/
	# The sqlite database file to query.
# [ Effects ]
# Reads links (sole column) from standard input and writes the joined visitation
# data to standard output.
##
if test $# -ne 1
then
	echo >&2 "ERROR: requires exactly one argument, $# given, the sqlite database file path."
	exit 1
fi
exec "$XAL_CONTEXT/visitation/sqlite-query.sh" google-history "$1"
