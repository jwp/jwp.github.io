"""
# Split a JSON array into a directory of files.

# Primarily used to normalize the depth of a collection so that
# &.json.shape can identify the protocol without descent.
"""
import os.path
import json

def fs(target, data):
	"""
	# Distribute the array elements into files named after their 1-based index.
	# Expects the &target path to already exist.

	# [ Parameters ]
	# /target/
		# The directory to place the indexes into.
	# /data/
		# The sequence to distribute.
	"""

	for i, idata in enumerate(data):
		ipath = os.path.join(target, str(i+1) + '.json')
		with open(ipath, 'w') as f:
			json.dump(idata, f)

if __name__ == '__main__':
	import sys
	fs(sys.argv[1], json.load(sys.stdin)) # [directory-path]
