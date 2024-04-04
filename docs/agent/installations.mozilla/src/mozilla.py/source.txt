"""
# Retrieve mozilla profile paths using the default selection, index number, or name.
"""
import os.path

class Profiles(object):
	"""
	# Profile directory lookup using `profiles.ini` file.

	# Class defers dependency imports until &load is called.
	"""

	profiles_path: str
	profiles_if: object

	def __init__(self, path, *, configsections=None):
		"""
		# [ Parameters ]
		# /path/
			# The path to the INI file to load.
		# /configsection/
			# Default to &None; set to the object to use as the loaded
			# configuration if &load is not being used.
		"""

		self.profiles_path = path
		self.profiles_if = configsections

	def load(self):
		"""
		# Import &configparser and load the INI file, &profiles_path.
		"""

		from configparser import ConfigParser
		self.profiles_if = ConfigParser()
		self.profiles_if.read(self.profiles_path)
		return self

	def p_defaults(self):
		"""
		# Scan for `Install*` sections and yield the `Default` field.
		"""

		for x in self.profiles_if.sections():
			if not x.lower().startswith('install'):
				continue

			s = self.profiles_if[x]
			yield s['Default']

	def p_index(self, index:str):
		"""
		# Get the Path using the profile "index" number.
		"""

		idxstr = 'Profile' + index
		if idxstr in self.profiles_if:
			yield self.profiles_if[idxstr]['Path']

	def p_name(self, name:str):
		"""
		# Scan for sections with a matching `Name` and yield its `Path`.
		"""

		for x in self.profiles_if.sections():
			s = self.profiles_if[x]
			if s.get('Name', '') == name:
				yield s['Path']

	@classmethod
	def find(Class, profiles_ini, inquery=None) -> str:
		"""
		# Find the default profile path or the one identified by &inquery.

		# [ Returns ]
		# Absolute path to the selected profile directory as a string.
		"""

		pv = Class(profiles_ini).load()

		if inquery is None:
			# <command> ini-path, show default
			i = pv.p_defaults()
		else:
			# <command> ini-path selection
			if isinstance(inquery, int):
				i = pv.p_index(inquery)
			else:
				i = pv.p_name(inquery)

		yield from i

def defaults(agent, path):
	"""
	# Identify all default profiles.
	"""

	for p in Profiles.find(path/'profiles.ini', inquery=None):
		fullpath = path@p
		yield (agent, fullpath.container, fullpath.identifier)

def visitation(agent, path):
	for a, p, n in defaults(agent, path):
		yield (a, p@n, 'places.sqlite')
