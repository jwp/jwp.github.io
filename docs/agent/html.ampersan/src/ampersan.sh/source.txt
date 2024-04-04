#!/bin/sh
# Approximately sanitize ampersands for XSLT processing.
# Notably, clean up `href` attributes with improperly escaped URIs.
#
# Only useful when tidy or bsoup is unavailable.
##

# Destroy <DT> and </DT> tags to handle exports that fail to close.
sed -E 's/\<\/?[[:space:]]*[dD][tT][[:space:]]*\>//g' | \
# Translate common entity references.
sed -E \
	-e 's/&quot;/\&#34;/g' \
	-e 's/&amp;/\&#38;/g' \
	-e 's/&lt;/\&#60;/g' \
	-e 's/&gt;/\&#61;/g' \
	-e 's/&nl;/ /g' \
	-e 's/&newline;/ /g' \
	-e 's/&nbsp;/ /g' \
	-e 's/&tab;/ /g' \
	| \
# Isolate record of interest.
sed -E -e 's/[hH][rR][eE][fF][[:space:]]*=[[:space:]]*"[^"]*"/\
<<<&\
/' | \
# Operate on record of interest.
# Escape ampersands that are *probably* not valid references.
sed -E -e '/^\<\<\<[hH][rR][eE][fF]/s/&(...[^;])/\&#38;\1/g' | \
# Destroy XML safeties (Unlikely, but protects prior substitution from matching element content).
sed -e 's/<<<//'
