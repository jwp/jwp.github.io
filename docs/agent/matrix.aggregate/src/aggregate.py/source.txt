"""
# Merge link records.
"""
import sys
import functools
import hashlib
from fault.system import process
from fault.time.types import Timestamp, Measure

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
		title, RS,
	])

def structure(line, *, fields=4, FS='\t', RS='\n', tuple=tuple):
	"""
	# Construct a tuple of four fields from the given line.
	"""

	assert line[-1] == RS
	return tuple(line[:-1].split(FS, fields-1))

def merge(link, former, latter):
	"""
	# Merge the fields of identical links recognizing earlier time contexts
	# and non-empty fields as preferable.
	"""

	l, t, i, T = former
	nd = 0

	if link != former[0]:
		nd += 1

	if latter[1] < t:
		# Earliest timestamp wins.
		t = latter[1]
		nd += 1

	if latter[-1] and not T:
		# First non-empty title wins.
		T = latter[-1]
		nd += 1
	if not i and latter[2]:
		# First non-empty icon wins.
		i = latter[2]
		nd += 1

	if nd:
		return (link, t, i, T)
	else:
		# Avoid reconstruction when fields have not been changed.
		return former

@functools.lru_cache(8)
def slashstrip(s):
	return s.rstrip('/')

def slashagnostic(f, l) -> str:
	"""
	# Return the preferred link to represent the referent or an empty string
	# if they should not be considered equivalent.

	# Handles trailing-slash cases where nearly identical links are present.
	"""

	if slashstrip(f) == slashstrip(l):
		if len(l) > len(f):
			# Take the longer one regardless of superfluous slashes.
			return l
		else:
			return f
	else:
		return ''

def uhash(link):
	"""
	# Construct a time delta from the hash of the link.
	"""

	h = hashlib.sha512(usedforsecurity=False)
	assert h.digest_size == 64

	h.update(link.lower().encode('utf-8'))
	hd = h.digest()

	ns = 0
	for i in range(0, 64, 8):
		ns ^= int.from_bytes(hd[i:i+8])

	m = Measure.of(nanosecond=ns)
	delta = m.truncate('minute')
	return m.decrease(delta)

def stamped(record):
	t = Timestamp.of(iso=record[1]).truncate('minute')
	return (
		record[0],
		t.elapse(uhash(record[0])),
		record[2],
		record[3],
	)

def processor(records):
	"""
	# Merge duplicate link records; presumes &records is sorted.
	"""

	# Initial state that will be immediately overwritten by &merge.
	i = iter(records)
	try:
		y = next(i)
	except StopIteration:
		return

	for r in i:
		l = slashagnostic(r[0], y[0])
		if l:
			y = merge(l, y, r)
		else:
			yield y
			y = r
	else:
		# Last one.
		yield y

def main(inv:process.Invocation) -> process.Exit:
	records = map(structure, sys.stdin.readlines())
	processed = processor(records)
	hashed = map(stamped, processed)
	sys.stdout.writelines(map(sequence, hashed))
	return inv.exit(0)
