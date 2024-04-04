"""
# Query for user agent executables and profile data directories.
"""
from collections.abc import Iterable
from .installations import locations

from fault.vector import recognition
from fault.system import process
from fault.system import files

# Agent Class -> Agent Factor
modules = {
	'mozilla-firefox': '.mozilla',
	'google-chrome': '.google',
	'apple-safari': '.apple',
}

restricted = {
	'-q': ('field-replace', True, 'qualified'),
	'-Q': ('field-replace', False, 'qualified'),
}
required = {
	'-A': ('set-add', 'agents'),
	'-u': ('field-replace', 'user-path'),
}
def main(inv:process.Invocation) -> process.Exit:
	inv.imports(['HOME', 'PATH'])

	config = {
		'agents': set(),
		'qualified': True,
		'user-path': None,
	}
	events = recognition.legacy(restricted, required, inv.argv)
	remainder = recognition.merge(config, events)
	inquery, = remainder # profiles | executables

	agentset = set(config['agents']) or set(modules.keys())
	if config['user-path'] is None:
		hd = files.root @ inv.environ['HOME']
	else:
		hd = (process.fs_pwd() @ config['user-path'])

	# Exclude results from unwanted agents.
	agentf = (lambda x: x[0] in agentset)

	if config['qualified']:
		qualify = (lambda x, y: x+'\t'+str(y))
	else:
		qualify = (lambda x, y: str(y))

	if inquery == 'executables':
		import os
		paths = [files.root@x for x in inv.environ['PATH'].split(os.pathsep)]
		for agent, path in filter(agentf, locations.executables(paths)):
			print(qualify(agent, path))
	else:
		# Agent module dependent requests.
		from importlib import import_module
		i = filter(agentf, locations.profiles(hd))

		if inquery == 'profiles':
			method = 'defaults'
		elif inquery == 'visitation':
			method = 'visitation'
		else:
			import sys
			sys.stderr.write(f"ERROR: unrecognized data request, {inquery!r}.\n")
			inv.exit(1)

		for agent, data in filter(agentf, i):
			am = import_module(modules[agent], package=__package__+'.installations')
			for ac, p, n in getattr(am, method)(agent, data):
				if n is None:
					continue
				print(qualify(ac, p/n))

	return inv.exit(0)
