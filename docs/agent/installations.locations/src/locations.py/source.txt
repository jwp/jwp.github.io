"""
# Agent classifications associated with executable names and user relative data directories.
"""
from collections.abc import Iterable

from fault.system import files
from fault.system import process
from fault.system import query

profiles_index = {
	'google-chrome': [
		'.config/google-chrome',
		'.config/chromium',
		'Library/Application Support/Google/Chrome',
		'Library/Application Support/Google/Chrome Canary',
	],

	'mozilla-firefox': [
		'.mozilla/firefox',
		'Library/Application Support/Firefox',
	],

	'apple-safari': [
		'Library/Safari',
	],
}

def profiles(directory, *, Index=profiles_index):
	"""
	# Identify the available agent profile data.
	"""

	for agent, paths in Index.items():
		for p in paths:
			try:
				yield agent, (directory @ p).fs_require('/')
			except files.RequirementViolation:
				pass

executables_index = {
	'mozilla-firefox': [
		'firefox',
		'iceweasel',
		'/Applications/Firefox.app/Contents/MacOS/firefox',
		'/Applications/FirefoxDeveloperEdition.app/Contents/MacOS/firefox',
	],

	'google-chrome': [
		'chrome',
		'chromium',
		'/Applications/Google Chrome.app/Contents/MacOS/Google Chrome',
		'/Applications/Google Chrome Canary.app/Contents/MacOS/Google Chrome Canary',
	],

	'apple-safari': [
		'/Application/Safari.app/Contents/MacOS/Safari',
	]
}

def executables(paths: Iterable[files.Path], flags='x', *, Index=executables_index):
	"""
	# Identify the agent executables available on the system
	# with their associated classification.
	"""

	req = '.' + flags
	for agent, names in Index.items():
		for n in names:
			if n[:1] == '/':
				try:
					yield agent, (files.root @ n).fs_require(req)
				except files.RequirementViolation:
					pass
				else:
					continue

				# Check user home relative instead of &paths.
				# user/Applications/...
				paths = [query.home()]

			# Relative path. Scan paths(PATH).
			for p in paths:
				try:
					yield agent, (p @ n).fs_require(req)
				except files.RequirementViolation:
					pass
				else:
					break
			else:
				# Not found.
				pass
