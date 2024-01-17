Created by Hugo Azurmendi; January 2024
Email: hugo.azurmendi@fda.hhs.gov

This folder contains essential files to reproduce Molecular Dynamics (MD)
results of the 2024 PNAS article:
"The structure of a C. neoformans polysaccharide motif recognized by protective
antibodies: A combined NMR and MD study".

The MD studies were done using the AMBER20 software package and run on Linux,
Centos 7, GPUs. Basic familiarity with AMBER, BASH and Linux is required to use
the provided scripts.

Starting coordinates and parameter files were created using the 'tleap' AMBER
program. Within the 'build-*' folders are the input files to generate the
starting coordinates and parameter files. To execute, 'cd' to one of the 'build'
folders and in a BASH terminal enter (with AMBER properly installed; '[]' is
optional to generate an output log file):
 tleap -f leapIn*.txt [> leapIn*.log]

We implement the Hydrogen Mass Repartition (HMR) method to speed up MD
simulations. For this the normal parameter file generated (*.prmtop) must be
modified using the AMBER program 'parmed'.
In addition, modified saccharides may require adjustment of charges to make them
neutral. This was necessary for O-acetylated mannoses. The parmed input scripts
provided fix this as well. On terminal do:
 parmed -i parmedIn-*.txt [> parmedIn-*.log]

The commands above were excuted to provide directly all the results from that
action, with the output messages saved as '*.log' text files. You may rerun the
scripts, but keep in mind that 'parmed' doesn't overwrite parameter files.
Once the above steps are done the simulation can be started using the provided
adaptable script 'mdrun-full.com'. Read the comments in the script for details.

Two partial MD trajectories, discussed in the paper, are included in folder
Trj-PDBs, one for gxm10-Ac3 (2000 frames -> 2 mcs duration), and one for GXM12RU
(last 500 frames, 500 ns).
