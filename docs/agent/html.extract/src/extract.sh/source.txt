#!/bin/sh
SCRIPT="$0"
SD="$(dirname "$SCRIPT")"
XAL="$(cd "$SD" && pwd)"
HTMLTIDYCONFIG="$XAL/truncate.i"

# Set context ELEMENT and apply anchor PREDICATE.
ELEMENT='.'
PREDICATE='[node()]'

if test $# -gt 0
then
	ELEMENT="$1"
	shift 1
fi

if test $# -gt 0
then
	PREDICATE="$1"
	shift 1
fi

# tidy | anchors.xsl; slightly tricky to get this to work with bookmark files.
# "${HXNORMALIZE:-hxnormalize} -xe | \
"${HTMLTIDY:-tidy}" config "$HTMLTIDYCONFIG" /dev/stdin 2>/dev/null | \
"${XSLTPROC:-xsltproc}" 2>/dev/null \
	--html --nonet --novalid --nowrite --nodtdattr \
	--stringparam SITE "${SITE:-http://[]}" \
	--stringparam PATH "${SITEPATH:-./}" \
	--stringparam FS "$FS" --stringparam RS "$RS" \
	--stringparam CONTEXT "$ELEMENT" \
	--stringparam PREDICATE "$PREDICATE" \
	--stringparam time-context "$XAL_DEFAULT_TIMECONTEXT" \
	"$XAL/anchors.xsl" /dev/stdin
