#!/bin/sh
# Strip the tagging from a NETSCAPE bookmark file.
#
# Much is presumed about the order in which the attributes of the anchor tags appear.
# `HREF` *must* come before `ADD_DATE`. `ADD_DATE` *must* come before `ICON`.
#
# Only useful when html-tidy and xsltproc are unavailable.
##
FS="$(printf '\t')"
RS='
'

# Force one anchor per-line.
sed -E 's/(<[[:space:]]*[aA][[:space:]]+)/\
\1/g' | \
# Lines after and including the first anchor tag.
grep -A1 '<[aA][[:space:]]' | \
# Ignore lines that are probably not anchors. (Presumes formatting)
grep -v '^[[:space:]]*\(<DT>\|--\|</DL>\|</\?HTML>\)[[:space:]]*$' | \
grep -v '\(<META.*>\|</\?DL></\?p>\|<\/[hH][0-9]>\)' | \
# Position attribute contents into fields.
sed \
	-e 's/.*[hH][rR][eE][fF]="//;' \
	-e 's/" [aA][dD][dD]_[dD][aA][tT][eE]="/'"$FS"'/;' \
	-e 's/" [iI][cC][oO][nN]="/'"$FS"'/;' \
	-e 's/">\(.*\)<\/[aA]>.*$/'"$FS"'\1/;'
