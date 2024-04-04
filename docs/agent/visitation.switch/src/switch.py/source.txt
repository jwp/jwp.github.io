"""
# Select and execute a processor for the resource list given to standard input.
"""
import sys
import subprocess
from dataclasses import dataclass, field
from fault.system import process
from fault.system import files

cases = {
	'mozilla-firefox': 'mozilla-places-report',
	'google-chrome': 'google-history-report',
}

def main(inv:process.Invocation) -> process.Exit:
	"""
	# Execute corresponding processors sequentially.
	"""

	links, = inv.argv
	lf = (process.fs_pwd() @ links)
	vd = (files.root@__file__) ** 1
	i = 0
	with files.Path.fs_tmpdir() as t:
		for line in sys.stdin.readlines():
			i += 1
			agent, rpath = line.split('\t', 2)
			fp = (process.fs_pwd() @ rpath.removesuffix('\n'))
			factor = cases[agent]
			script = vd / (factor + '.sh')
			copy = (t/('db-'+str(i)))
			copy.fs_replace(fp)

			try:
				with lf.fs_open('r') as f:
					r = subprocess.run([script, copy], stdin=f, check=True)
			except subprocess.CalledProcessError:
				pass
