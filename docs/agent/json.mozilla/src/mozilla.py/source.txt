"""
# Traverse the JSON document looking for objects that appear to be hyperlinks.
"""
import json
import collections
from itertools import chain
from dataclasses import dataclass, field
from collections.abc import Mapping, Iterable

# In priority order.
time_fields = [
	'dateAdded',
	'date-added',
	'lastModified',
	'lastAccessed',
]

uri_fields = [
	'uri',
	'url',
	'iri',
	'href',
]

title_fields = [
	'title',
	'name',
]

# Mozilla specific schemes.
ignore = set([
	'place',
	'about',
	'moz-extension',
])

# Used by &find.
extension_fields = [
	'children',
]

# Transparently handle LZ4 compressed files.
jlz4_header = b'mozLz40\x00'
jlz4_header_size = len(jlz4_header)

def decompress(reader: Iterable[bytes], *, header=b'mozLz40\x00') -> Iterable[bytes]:
	"""
	# lz4json decompression for mozilla session storage and backups.

	# Defers import of (pypi.org/package)&<lz4> until usage.
	"""

	from lz4.block import decompress as lz4_dc
	return (lz4_dc(bytearray().join(reader)),)

def link_escapes(link, tab=('\t', '%'+hex(ord('\t'))[2:].rjust(2, '0'))):
	"""
	# Force percent escaped tabs.
	"""

	return link.replace(*tab)

def link_normalize(link):
	"""
	# Strip whitespace and filter (mozilla) internal schemes.
	"""

	# Ignore whitespace.
	l = link.strip()
	if not l:
		return ''

	eos = l.find(':', 0, 48)
	if eos == -1:
		# Probably not an RI.
		return l

	# Filter unwanted browser links.
	scheme = l[:eos].lower()
	if scheme in ignore:
		return ''

	return scheme + l[eos:]

def structure(src:Iterable[bytes]):
	"""
	# Load the json data; transparently handles LZ4 compressed json.
	"""

	i = iter(src)

	first = b''
	for r in i:
		first += r
		if len(first) >= jlz4_header_size:
			if first[:jlz4_header_size] == jlz4_header:
				# Matched header, run decompression.
				ijson = decompress(chain((first[jlz4_header_size:],), i))
			else:
				# Presume raw json.
				ijson = chain((first,), i)
			break
	else:
		# Presume very small document.
		ijson = (first,)

	return json.loads(b''.join(ijson))

def exclude(windows, closedtabs=['_closedTabs']):
	"""
	# Filter closed tabs in the window records.
	"""

	count = 0

	for w in windows:
		for n in closedtabs:
			tabrecords = w.pop(n, ())
			for r in tabrecords:
				se = r.get('state', {'entries': []})
				count += len(se)
	return count

def find(struct):
	"""
	# Find all the links in the given data.
	"""

	containers = collections.deque((struct,))

	while containers:
		d = containers.popleft()
		if isinstance(d, Mapping):
			pass
		else:
			if isinstance(d, (list, tuple)):
				containers.extend(d)

			# Scalar of non-ri record.
			# JSON data, not expecting other container types.
			continue

		link = None
		for u in uri_fields:
			if u in d:
				link = d[u]
				break
		else:
			# Probably not a bookmark or tab link.
			containers.extend(d.values())
			continue

		for f in extension_fields:
			ext = d.get(f, None) or ()
			if not ext:
				pass
			elif isinstance(ext, Mapping):
				ext = ext.values()

			containers.extend(ext)

		# Minimal scrubbing here.
		link = link_escapes(link_normalize(link))
		if not link:
			# Filtering mozilla specific RIs: place:, about:, moz-extension:.
			continue

		yield link, d

def transform(ld):
	"""
	# Convert the link and (json) object pair into a tuple for sequencing.
	"""

	l, d = ld

	# Normalize spaces; important for serialization.
	for tk in title_fields:
		if tk in d:
			titlesrc = d[tk]
			break
	else:
		titlesrc = ''
	title = ' '.join(titlesrc.strip().split())

	icon = d.get('icon') or d.get('image') or d.get('iconUri') or ''

	for f in time_fields:
		if f in d:
			time = str(d[f] or '').strip()
			break
	else:
		time = ''

	return (l, time, icon, title)

def sequence(records, FS='\t', RS='\n') -> Iterable[str]:
	"""
	# Transform link-quads into serializable data records.
	"""

	for record in records:
		link, time, icon, title = record
		yield ''.join([
			link, FS,
			time, FS,
			icon, FS,
			title, FS,
			RS,
		])

def session_fields(data):
	"""
	# Whether &data appears to be a session.
	"""

	if 'windows' not in data:
		return False
	w = data['windows'][0]

	if 'tabs' not in w:
		return False
	e = w['tabs'][0]

	if 'entries' not in e:
		return False

	return True

def bookmarks_fields(data):
	"""
	# Whether &data appears to be a bookmark export.
	"""

	if data.get('root', '') != 'placesRoot':
		return False
	if not isinstance(data.get('children'), list):
		return False

	return True

def entry_fields(data):
	"""
	# Whether &data appears to have resource indicator data.
	"""

	if 'url' not in data and 'uri' not in data:
		return False
	if 'title' not in data:
		return False

	return True

if __name__ == '__main__':
	import sys
	records = find(structure(sys.stdin.buffer.readlines()))
	sys.stdout.writelines(sequence(map(transform, records)))
