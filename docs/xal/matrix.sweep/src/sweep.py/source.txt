"""
# Normalize and filter link data.

# [ Adjustments ]
# /link/
	# - Force "insecure" schemes, ftp and http for consistency. (Publish with +s as desired)
	# - Optionally, filter by exact match or by prefix (pygtrie).
# /time-context/
	# - Convert into `ISO-8601` strings.
	# - Adjust precision when the point falls outside of limits.
# /icon/
	# - Transparent
# /titles/
	# - Normalize title whitespaces.
	# - Optionally, remove many emoji characters. (incomplete range list)
	# - Filter titles that are trivially redundant with the link itself.
"""
import sys
import functools
from collections.abc import Iterable
from typing import TypeAlias, Callable

from fault.system import process
from fault.vector import recognition

# Convert integer and other datetime representations to ISO-8601 strings.
from fault.time.types import Timestamp, Measure
from fault.time.system import utc

normal = {
	'https': 'http',
	'ftps': 'ftp',
}

def _unicode_emoji_context():
	ranges = list(map(ord,
		"\U00002600"
		"\U000026FF"
		"\U0001F1E0"
		"\U0001F1FF"
		"\U0001F191"
		"\U0001F19A"
		"\U0001F600"
		"\U0001F64F"
		"\U0001F680"
		"\U0001F6FF"
		"\U0001F300"
		"\U0001F5FF"
		"\U0001FC00"
		"\U0001FFFD"
	))
	first = min(ranges)
	last = max(ranges)
	pairs = list(zip(ranges[0::2], ranges[1::2]))

	@functools.lru_cache(64)
	def emojiclean(c, *, ord=ord, first=first, last=last, ranges=pairs):
		#! Would prefer "isemoji", but used with filter(), so backwards.
		i = ord(c)
		if i < first or i > last:
			# Fast path; outside of all ranges.
			return True

		for start, stop in ranges:
			if start <= i or i <= stop:
				return False
		return True
	return emojiclean

def demoji(string, *, replacement='', isemojiclean=_unicode_emoji_context()):
	"""
	# Destroy emojis and most sequences found in the &string.
	"""

	# Iterate through the string segments that end with VS-16.
	i = 0
	while i < len(string):
		eos = string.find('\uFE0F', i)
		if eos == -1:
			# No sequences to process
			yield from filter(isemojiclean, string[i:])
			break

		# VS-16 sequence.
		joins = string[i:eos].split('\N{ZWJ}')
		# Find the beginning of the VS-16 sequence.
		for fi, j in enumerate(reversed(joins)):
			if len(j) > 1:
				# Sequence start detected.
				fi = (len(joins) - fi)
				break
		else:
			# Entire segment is VS-16.
			fi = 1

		# Process normally up to the emoji sequence.
		# Maintain any joins.
		yield from filter(isemojiclean, '\N{ZWJ}'.join(joins[:fi]))

		# Continue beyond the sequence boundary.
		i = eos + 1

del _unicode_emoji_context

def relink(link):
	"""
	# Normalize the link in order to maximize reductions.
	# Trigger filtering with empty string returns.
	"""

	l = link.strip()

	# Get scheme for mapping to a normal form.
	eos = l.find(':', 0, 48)
	if eos == -1:
		# Probably not an scheme qualified RI.
		return ''
	else:
		scheme = l[:eos].lower()

	# Eliminate, often, superfluous trailing hash.
	if l.endswith('#'):
		end = -1
	else:
		end = None

	# Normalize scheme for reductions.
	scheme = normal.get(scheme, scheme)
	nlink = scheme + l[eos:end]

	return trailing(nlink)

# 1900 < ts < +100y
time_limits = (
	Timestamp.of(year=1900),
	utc().elapse(year=1).truncate('year'),
)

def restamp(ts:str, *, current=utc().truncate('minute'), limits=time_limits):
	"""
	# Interpret the timestamp, &ts, as a &Timestamp instance.
	# Parsing an `ISO-8601` formatted string or
	# converting a decimal unix epoch string.

	# When the interpreted timestamp is outside of the &limits,
	# adjust the precision until it falls within.

	# [ Returns ]
	# The interpreted &Timestamp instance truncated to the minute.
	"""

	if not ts:
		return current

	if not ts.isdigit():
		return Timestamp.of(iso=ts)

	# Reduce precision and handle precision failures.
	i = int(ts)
	tc = Timestamp.of(unix=i)

	# Presume unrecognized or unknown precision when outside of &limits.
	while tc > limits[1]:
		i = i // 1000
		tc = Timestamp.of(unix=i)
	while tc < limits[0]:
		i = i * 1000
		tc = Timestamp.of(unix=i)

	return tc
del time_limits

def retitle(title:str):
	"""
	# Normalize the whitespace in &title.
	"""

	return ' '.join(
		x for x in title.strip().split()
		if x not in {
			'\N{ZWJ}',
			'\N{ZWNJ}',
		}
	)

def sequence(record, *, FS='\t', RS='\n'):
	"""
	# Construct a sequenced record for serialization.
	"""

	link, time, icon, title = record

	# Pad the timestamp for consistency.
	t = str(time)
	sub = t.rfind('.')
	pad = '0' * (10 - (len(t) - sub))

	return ''.join([
		link, FS,
		t + pad, FS,
		icon, FS,
		title, FS,
		RS,
	])

def structure(line, *, fields=4, FS='\t', RS='\n', tuple=tuple):
	"""
	# Construct a tuple of four fields from the given line.
	"""

	assert line[-1] == RS
	return tuple(line[:-1].split(FS, fields-1))

def trailing(link):
	"""
	# Normalize host references to include trailing slash.
	"""

	if link.endswith('/') or link.startswith('/'):
		# Fast path.
		return link

	h = link.find('://')
	if h == -1:
		# Not an iauthority IRI.
		return link

	# Presumes escaped slashes.
	first = link.find('/', h + 3)
	if first == -1:
		# Normalize site reference.
		return link + '/'
	else:
		return link

def normalize(records, *, titlefilter=str):
	joined = None
	for fields in records:
		try:
			link, time, icon, title = fields
		except ValueError: # Abnormal record.
			# Fill missing fields.
			nfields = len(fields)
			link = fields[0]
			title = ''
			time = ''
			icon = ''

			if nfields > 1:
				title = fields[-1]

				if nfields > 2:
					# link, time, title
					time = fields[1]
					if nfields > 3:
						icon = fields[2]

		clean = relink(link)
		if not clean:
			continue

		# Normalize title spaces and check for link equivalence.
		tr = retitle(titlefilter(title))
		if relink(tr).lower() == clean.lower():
			# A distance measure would be a better comparison, but
			# this should catch most of the repetition.
			tr = ''

		# Truncate to minute and hide a xor-folded sha512.
		ts = restamp(time)

		yield (clean, ts, icon, tr)

def rewrite(find, records):
	"""
	# Adjust the prefixes on the given records.
	"""

	for record in records:
		link = record[0]

		prefix, delta = find(link)
		if delta is None:
			# Filtered.
			continue
		elif delta == '':
			# No-op.
			yield record
		else:
			# Reconstruct with adjusted prefix.
			yield (delta + link[len(prefix):],) + record[1:]

def rewriting(rwt:Iterable[tuple[str,str]]):
	"""
	# Construct a filter rewriting the prefixes defined in &rwt.

	# Rewrites to &None are discarded.
	"""

	from pygtrie import CharTrie

	# Rewrite the empty prefix to an empty string. (no-op)
	rws = {
		# Invariant; rewrite empty to empty.
		'': '',
	}
	rws.update((x, y or None) for x, y in rwt)
	roots = CharTrie(rws)

	# Find Longest Prefix.
	lp = roots.longest_prefix
	def flp(link, *, lp=roots.longest_prefix):
		# Match is always successful as the empty string
		# is rewritten to the empty string.
		match = lp(link)
		return match.key, match.value

	return functools.partial(rewrite, flp)

def ignoring(values, records):
	return (x for x in records if x[0] not in values)

def scanning(v, records):
	# Filter records where substring (v) is found in the link, title, or icon.
	for r in records:
		if v in r[0] or v in r[-1] or v in r[2]:
			continue
		else:
			# No matching substrings.
			yield r

def interpret_filters(files):
	"""
	# Load the filter files and emit the constructed filters.
	"""

	ctx = {'rewrite-target': ''}
	exact = []
	prefix = []
	substring = []

	# Append the prefix match with the current value of `ctx['rewrite-target']`.
	# Where the `>` instruction updates the field.
	prefix_rw = (lambda x: prefix.append((x, ctx['rewrite-target'])))

	cases = {
		'=': exact.append,
		'^': prefix_rw,
		'>': (lambda x: ctx.__setitem__('rewrite-target', x)),
		'*': substring.append,
	}
	# Ignore most spaces and comments.
	cases.update((x, (lambda x: None)) for x in '\n\t #')

	for p in files:
		with p.fs_open('r') as f:
			for line in f.readlines():
				add = cases[line[:1]]
				add(line[1:-1])

	if exact:
		yield functools.partial(ignoring, set(exact))
	if prefix:
		yield rewriting(prefix)
	for s in substring:
		yield functools.partial(scanning, s)

restricted = {
	'-E': ('field-replace', True, 'title-strip-emoji'),
}
required = {
	'-X': ('sequence-append', 'text-filters'),
	'-f': ('sequence-append', 'function-false-filters'),
	'-F': ('sequence-append', 'function-true-filters'),
}

def python_filters(filtertype, freferences, series):
	from importlib import import_module

	for ffilter in freferences:
		fmp, fa = ffilter.split(':', 1) # -fF module.path:attribute.path
		fav = fa.split('.')

		ff = import_module(fmp)
		for fa in fav:
			ff = getattr(ff, fa)
		def f(lq, Filter=ff):
			return ff(lq[0])
		series = filtertype(f, series)
	return series

def main(inv:process.Invocation) -> process.Exit:
	config = {
		'title-strip-emoji': False,
		'text-filters': [],
		'function-false-filters': [],
		'function-true-filters': [],
	}
	ev = recognition.legacy(restricted, required, inv.argv)
	remainder = recognition.merge(config, ev)

	efilter = str
	if config['title-strip-emoji']:
		efilter = (lambda x: ''.join(demoji(x)))

	records = map(structure, sys.stdin.readlines())
	first = normalize(records, titlefilter=efilter)
	series = first

	# -X options citing a filter file.
	if config['text-filters']:
		ifp = map(process.fs_pwd().__matmul__, config['text-filters'])

		for ifilter in interpret_filters(ifp):
			series = ifilter(series)

	if config['function-false-filters']:
		from itertools import filterfalse
		series = python_filters(filterfalse, config['function-false-filters'], series)
	if config['function-true-filters']:
		series = python_filters(filter, config['function-true-filters'], series)

	sys.stdout.writelines(map(sequence, series))
	return inv.exit(0)
