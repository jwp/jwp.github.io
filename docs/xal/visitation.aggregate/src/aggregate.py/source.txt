#!/usr/bin/env python3
"""
# Given sorted visitation records to standard input, combine the records by minimizing the
# earliest visit, maximizing the latest visit, and summing the visit count.
# The combined set will be written to standard output.
"""
import sys
from fault.time.types import Timestamp
from fault.time.system import utc
from fault.system import process

def merge(former, latter):
	"""
	# Combine two records regarding the same subject.
	"""

	return (
		former[0],
		former[1] + latter[1],
		min(former[2], latter[2]),
		max(former[3], latter[3]),
	)

def structure(line, FS='\t', RS='\n'):
	subject, count, earliest, latest = line.split(FS)
	assert latest[-1:] == RS
	latest = latest[:-1]
	return (
		subject,
		int(count),
		Timestamp.of(iso=earliest) if earliest else utc(),
		Timestamp.of(iso=latest) if latest else utc(),
	)

def reduce(records):
	"""
	# Process visit records.
	"""

	init = last = ('', 0, utc(), utc())

	for current in records:
		if current[1] == 0:
			continue

		if current[0] == last[0]:
			last = merge(last, current)
		else:
			yield last
			last = current
	else:
		if last != init:
			yield last

def main(inv:process.Invocation) -> process.Exit:
	"""
	# Merge visitation metrics.
	"""

	inv.imports(['RS', 'FS'])
	rs = inv.environ['RS']
	fs = inv.environ['FS']
	last = ('', 0, utc(), utc())

	sys.stdout.writelines(
		''.join([
			r[0], fs,
			str(r[1]), fs,
			r[2].select('iso'), fs,
			r[3].select('iso'), rs
		]) for r in reduce(map(structure, sys.stdin.readlines()))
	)

	return inv.exit(0)
