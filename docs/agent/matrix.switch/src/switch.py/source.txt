"""
# Select and execute a processor for the resource list given to standard input.
"""
import sys
from dataclasses import dataclass, field
from fault.system import process
from fault.system import files

@dataclass
class Resource(object):
	"""
	# Fields available for selecting processing methods.
	"""

	r_selector: files.Path
	r_type: str
	r_encoding: str

class Unrecognized(Exception):
	"""
	# Could not identify how to process resource.
	"""

	def __init__(self, resource:Resource):
		self.x_subject = resource

	def __str__(self):
		return "could not identify processing method for " + \
			f"{self.x_subject.r_selector!r}" + \
			f"[{self.x_subject.r_type}]"

def reflectcase(r:Resource, *, fields=4):
	"""
	# cat; presumed tsv.
	"""

	import subprocess
	with r.r_selector.fs_open('rb') as f:
		p = subprocess.run(['/bin/cat'], stdin=f, check=True)

def jsoncase(r:Resource):
	"""
	# json.mozilla or json.github-stars
	"""

	from ..json import mozilla as m
	from ..json import github as gh
	excluded_count = 0

	with r.r_selector.fs_open('rb') as f:
		data = m.structure(f.readlines()) # json or ls4json.

		if isinstance(data, list):
			if data and gh.star_fields(data[0]):
				sys.stdout.writelines(
					gh.sequence(map(gh.transform, data))
				)
			else:
				pass
		else:
			if m.bookmarks_fields(data):
				pass
			elif m.session_fields(data):
				# Filter closed tabs.
				excluded_count += m.exclude(data['windows'])

			sys.stdout.writelines(
				m.sequence(map(m.transform, m.find(data)))
			)

def htmlcase(r:Resource):
	"""
	# tidy | xsltproc anchors.xsl
	"""

	from .. import html
	import subprocess

	htmlexe = (files.root @ html.__file__) * 'extract.sh'
	with r.r_selector.fs_open('rb') as f:
		p = subprocess.run([htmlexe], stdin=f, check=True)

def textcase(r:Resource, *, max_record_size=1024*200):
	"""
	# Prefer JSON, but check for TSV as well.
	"""

	with open(str(r.r_selector), 'rb') as f:
		sample = f.read(8)
		if sample[:1] in {b'[', b'{'}:
			# Presume json.
			scase = jsoncase
		else:
			if sample == b'mozLz40\x00':
				# mozilla json file
				scase = jsoncase
			elif sample.strip()[:1] in {b'[', b'{'}:
				scase = jsoncase
			elif r.r_selector.identifier.endswith('.tsv'):
				while b'\n' not in sample:
					d = f.read(1024)
					sample += d
					if len(sample) > max_record_size:
						raise Unrecognized(r)
					elif len(d) < 1024:
						raise Unrecognized(r)
				else:
					# Found a newline
					tc = sample.count(b'\t')
					if tc == 3:
						# Four fields, maybe TSV.
						scase = reflectcase
					elif sample.startswith(b'data:') or \
					sample.startswith(b'http') or \
					sample.startswith(b'ftp'):
						# Probably not JSON; presume TSV.
						scase = reflectcase
					else:
						raise Unrecognized(r)

				scase = reflectcase

	return scase(r)

cases = {
	'text/plain': textcase,

	'text/json': jsoncase,
	'application/json': jsoncase,
	'application/x-lz4+json': jsoncase,

	'text/html': htmlcase,
	'text/xml': htmlcase,
	'application/xhtml+xml': htmlcase,
}

def main(inv:process.Invocation) -> process.Exit:
	"""
	# Execute corresponding processors sequentially.
	"""

	for line in sys.stdin.readlines():
		mimetype, encoding, rpath = line.split('\t', 3)
		fp = (process.fs_pwd() @ rpath.removesuffix('\n'))
		r = Resource(fp, mimetype, encoding)
		if r.r_type in cases:
			cases[r.r_type](r)
