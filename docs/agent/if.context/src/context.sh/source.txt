# Usually copied into xal/if/* scripts.
export XAL_PRODUCT="$(dirname "$XAL_CONTEXT")"
export SHELL='/bin/sh'

# Format configuration.
export FORMAT='text/tsv'
export FS="$(printf '\t')"
export RS='
'

# fault-tool wrappers.
FT="$(command 2>/dev/null -v fault-tool)"
if test x"$FT" = x""
then
	FT=/opt/fault/bin/fault-tool
fi

Sort ()
{
	sort "-t$FS" -k1 "$@"
}

fs_select ()
{
	local NUL="$(printf '\0')"

	for f in "$@"
	do
		find "$f" -print0 -type 'f' | \
			# P1 to avoid tears.
			xargs -0 -P1 -- file --mime-type --mime-encoding | \
			# Rotate [file type encoding] to [type encoding file].
			sed -E -n "$(printf '%s%s%s%s%s' 's/' \
				'(.*):[[:space:]]+' \
				'(.*);[[:space:]]*' \
				'charset=(.*)[[:space:]]*$/' \
				"\2${FS}\3${FS}\1/p" \
			)"
	done
}

xalcmd ()
{
	local COMMAND="$1"; shift 1
	case "$COMMAND"
	in
		# Aliases
		aqx) xalifx query executables;;
		aqp) xalifx query profiles;;
		help|'') (
			cd "$XAL_CONTEXT/if" || return
			echo "Commands:"
			find . -type file | grep -Ev "setup|context" | sed 's/\.\//'"$FS"'/' | sort
			echo "Link aggregation:"
			echo "${FS}\$ xal process ./path/to/directory-of-links [filter-files ...] >./links.tsv"
			echo "Visitation aggregation:"
			echo "${FS}\$ xal query visitation | xal track ./links.tsv >./visits.tsv"
		) >&2;;
		*) xalifx "$COMMAND" "$@";;
	esac
}

# Execute with the pypi requirements.
xalfix ()
{
	"$FT" python -P"$XAL_CONTEXT/.pypi" -L"$XAL_PRODUCT" "$@"
}

xalexec ()
{
	exec "$FT" python -P"$XAL_CONTEXT/.pypi" -L"$XAL_PRODUCT" "$@"
}

xalifx ()
{
	cmd="$1"
	shift 1
	. "$XAL_CONTEXT/if/${cmd}" "$@"
}
