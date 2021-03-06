Staging Folders
================

.. meta::
    :description lang=en: Using InitialWorkDirRequirement.



CWL stages all input files and directories in a random read-only temp directory away from the working directory. If these are part of the basecommand, arguments, or have an inputBinding, CWL will handle the file paths for you. However you the above limitations will not work in the following situations:

1. The File or Directory needs to be staged in the Docker image, but is not part of the command.
2. You need to change the file or directory in some way.
3. You need the file or directory to be in the working directory.
4. The path to the file or directory needs to be predictable.

Part 1: Input files in the working directory
--------------------------------------------

Here is a python script that can use the sync to synapse function:

.. code-block:: python

	import synapseclient
	import argparse
	import synapseutils

	if __name__ == '__main__':

	    parser = argparse.ArgumentParser("Stores files in Synapse")

	    parser.add_argument(
		    '-m',
		    '--manifest_file',
		    type = str,
		    required=True)
	    
	    parser.add_argument(
		    '-c', 
		    '--synapse_config_file', 
		    type = str, 
		    required=True)   

	    args = parser.parse_args()

	    syn = synapseclient.Synapse(configPath=args.synapse_config_file)
	    syn.login()

	    synapseutils.sync.syncToSynapse(syn, args.manifest_file)


And here is the CWL tool that calls it:

.. code-block:: YAML


	#!/usr/bin/env cwl-runner
	#
	# Authors: Andrew Lamb

	cwlVersion: v1.0
	class: CommandLineTool

	requirements:
	- class: InitialWorkDirRequirement
	  listing: $(inputs.files)

	hints:
	  DockerRequirement:
	    dockerPull: quay.io/andrewelamb/python_synapse_client
	    
	baseCommand:
	- python3
	- /usr/local/bin/sync_to_synapse.py

	inputs:

	  files: File[]
	      
	  synapse_config_file:
	    type: File
	    inputBinding:
	      prefix: "--synapse_config_file"

	  manifest_file:
	    type: File
	    inputBinding:
	      prefix: "--manifest_file"


	 
	outputs: []


In the above example we need to pass a manifest to the tool that has the paths of the files to be uploaded. If we didn't stage the files in the working directory, CWL would put them all in their own randomly generated temp directories. By placing them in the working directory we know that the relative paths will be just the name of the file.

To stage the files specified in the input files parameter we include the following:

.. code-block:: YAML

	requirements:
	- class: InitialWorkDirRequirement
	  listing: $(inputs.files)


Notice that the below input does not have an inputBinding. This means its a parameter of the tool, but not the command the tool is constructing. This allows the file parameter to be referenced by the InitialWorkDirRequirement:

.. code-block:: YAML
	inputs:

	  files: File[]

Part 2: Creating a config file in the working directory
-------------------------------------------------------

The below tool needs a config file, where the last line is a directory that is being passed in an input. The directory will be put in a random location in the docker image, so the config file cannot be passed in as an input as well, but needs to be written after the path to the directory is known.

.. code-block:: YAML

	baseCommand: run-pipe

	arguments:
	- --config
	- config_drops.ini

	requirements:
	  - class: InlineJavascriptRequirement
	  - class: InitialWorkDirRequirement
	    listing:
	      - entryname: config_drops.ini
		entry: |
		  [Drops]
		  samtools = samtools
		  star = STAR
		  whitelistDir = /usr/app/baseqDrops/whitelist
		  cellranger_ref_hg38 = $(inputs.index_dir.path)

	inputs:
	- id: index_dir
	  type: Directory

The above tool produces a file called config_drops.ini in the working directory with 4 lines. The first three refer to paths in the docker image, the fourth line refers the input directory and will put the path generated by CWL into the config file.


Part 3: Making an input file or directory writable
--------------------------------------------------

If you need to make a file writable you can use the writable attribute:

.. code-block:: YAML

	requirements:
	  - class: InitialWorkDirRequirement
	    listing:
	      - entry: $(inputs.input_file)
		 writable: true

	inputs:
	- id: input_file
	  type: File

