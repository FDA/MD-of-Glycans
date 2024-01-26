Created by Hugo Azurmendi; January 2024
Email: hugo.azurmendi@fda.hhs.gov
Link to this folder: https://github.com/FDA/MD-of-Glycans

This folder contains essential files to reproduce Molecular Dynamics (MD)
results of the 2024 PNAS article:
"The structure of a C. neoformans polysaccharide motif recognized by protective
antibodies: A combined NMR and MD study".

IMPORTANT: For easy exploration, we recommend downloading the whole directory by
clicking the green button "<> Code" and selecting 'Download zip'. This will
generate this folder with all content in your default Download folder (total
size is ~37 MB). You may then move the folder to a convenient location of your
choice.

The MD studies were done using the AMBER20 software package and run on Linux,
Centos 7, GPUs. Basic familiarity with AMBER, BASH and Linux is required to use
the provided scripts.

Starting coordinates and parameter files were created using the 'tleap' AMBER
program. Within the 'build-*' folders are the input files to generate the
starting coordinates and parameter files. To execute, 'cd' to one of the 'build'
folders and in terminal enter (with AMBER properly installed; '[]' is
optional to generate an output log file):
 tleap -f leapIn*.txt [> leapOut*.log]

We implemented the Hydrogen Mass Repartition (HMR) method to speed up MD
simulations. For this, the normal parameter file generated with 'tleap'
(*.prmtop) must be modified using the AMBER program 'parmed'.
In addition, modified saccharides may require adjustment of charges to make them
neutral. This was necessary for O-acetylated mannoses. The parmed input scripts
provided fix this as well. On terminal do:
 parmed -i parmedIn-*.txt [> parmedOut-*.log]

The commands above were executed to include results in the 'build-*' folders,
with the output messages saved as '*.log' text files. You may rerun the
scripts, but keep in mind that 'parmed' doesn't overwrite parameter files by
default.
Once the above steps are done the simulation can be started using the provided
adaptable script 'mdrun-full.txt'. Read the comments in the script for details.

Two partial MD trajectories, discussed in the paper, are included as compressed
zip files in the root folder, one for gxm10-Ac3 (2000 frames -> 2 mcs duration),
and one for GXM12RU (last 500 frames, 500 ns).
 
