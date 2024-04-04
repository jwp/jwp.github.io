"""
# Converter for &<http://github.com/user/starred>.
# &.json.gh-allstars can be used to get a full list.
"""
import json
from itertools import chain
from collections.abc import Iterable

def transform(star):
	"""
	# Construct link-quad from a github star record.
	"""
	time = star.get('starred_at') or ''
	ld = star['repo']
	link = ld['html_url']
	icon = star.get('homepage') or ld.get('homepage') or ''
	title = ld['name']

	return (link, time, icon, title)

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

def star_fields(data):
	"""
	# Whether `repo` and `name` are in &data.
	"""
	if not 'repo' in data:
		return False
	if not 'name' in data:
		return False

	return True

if __name__ == '__main__':
	import sys
	pages = json.load(sys.stdin)
	records = chain.from_iterable(pages)
	sys.stdout.writelines(sequence(map(transform, records)))
