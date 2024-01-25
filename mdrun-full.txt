#!/bin/bash
# Hugo Azurmendi - 01/02/2024; ALL IN ONE MDRUN FILE (tested with AMBER 20 in Centos 7)
# email: hugo.azurmendi@fda.hhs.gov

# Adaptable script for running MD on GPUs of solvated polysaccharides (PS) in AMBER.
# This script was used as base to run all MD simulations on C. neoformans oligosaccharides,
# as described in the article by Hargett A, Azurmendi H.F., et al., in PNAS 2024.

# Requires basic familiarity with: Linux, BASH, and AMBER.
# Set CUDA explicitly for GPU run with 'MYCUDA' below (use 'nvidia-smi' to check available GPUs)
# Creates inputs and runs: full minimization (FM), preparation (PREPMDIN), and production (MDIN).
# Options available:
#  - PS stretching during preparation (see STRETCH2 below; not used with C. neoformans)
#  - Use of restraints during production (XYLRSTR; used for Xylose here)
# Autoimages INPCOORs, saves restart files as PDB for easy inspection.
# Saves trajectories in 'nc' (binary) format, with or without waters depending on TOPSAVE.

# Edit as needed below: MDTXT, TRUN, MYCUDA, TPREP, NN, TOPSAVE, PRMTOP, TRJDUR
#                       STEP, MDENDJOB, MDDIR, XYLRSTR 
# To run on terminal I make the script executable ('chmod -x mdrun-full.com');
# then execute with './mdrun-full.com' (in BASH terminal)
# It prompts for input coordinates then number of the final.
# Each trajectory may last 10 or 100 ns; with 100 ns enter '10' to get a 1 ms total trajectory).
# For MD from scratch it expects a '*/*.inpcrd' file; else, select a restart file (*.ncrst).

# Define bold and some convenient colors for screen output
bold=$(tput bold); normal=$(tput sgr0); NONE='\033[00m'; GREEN='\033[01;32m'
YELLOW='\033[01;33m'; CYAN='\033[01;36m'; WHITE='\033[01;37m'

# Comment to be saved in file 'MDcomment.txt' within folder mdlog for records
MDTXT="\n ++ MD prod comments for solvated Cryptococcus neoformans PS.
Starting files created with tleap (without initial torsions nor E minization).
Solvated with 8 A of TIP3FB waters and using HMR for speed (dt = 4 fs).
Xyl rings are restrainted with two torsions to maintain the normal 4C1 configuration.\n"

echo -e "\n Is the following text good to be saved  as information?\n $MDTXT"
read -e -p "[Y/N]?: " contYN
[[ $contYN == 'N' ]] && echo -e "Edit MDTXT line in this script and execute again.\n" && exit 1

NN="gxm12ru"			# Nickname, used in dirs and files
# (!) Set correct paths/filenames and variables below
# To save trajectories WITH waters comment TOPSAVE line below (could be huge!)
# Use TOPSAVE to point to AMBER parameter file WITHOUT waters and ions
TOPSAVE="build-GXM12RU/gxm12ru.prmtop"		# PRMTOP to save trj WITHOUT waters
PRMTOP="build-GXM12RU/gxm12ruHMR-tp3fb.prmtop"	# 'hmr' or 'HMR' in name required for HMR run

# Next, restraint file for Xyl rings; for other projects comment line if not needed.
XYLRSTR="build-GXM12RU/xylring2tor-gxm12ru.rstr"
MYCUDA=1			# Run 'nvidia-smi' to check for GPUs available.
TRUN=300			# Temperature of MD production (K)

# Prep T ramps: 1K -100ps- TPREP -500ps- TPREP -200ps- TRUN -200ps- TRUN	(1 ns total)
TPREP=310			# Temperature reached at preparation (K)
TRJDUR=100			# 10 or 100 [ns] (trjdur * mdendjob = MDIN traj duration)
STEP=10				# In ps (for saving; suggested 10 without waters, increase if saving waters.)
MDENDJOB=10			# Repeats of TRJDUR; job number is added to ROOTNAME
MDDIR="md"$TRUN"K-FB3"		# Output dir is: DD/MDDIR (see definitions below)

#Comment STRETCH2 if not wanted or needed
#STRETCH2=46			# Strecth  to distance +-5, first atom to last C6 on PREPMDIN

# Output to these (main + one(TRUN) + three) directories and files
ROOTNAME=$NN'_md';		DD=${NN}'-MD';			DDSUB=$DD'/'$MDDIR
LOGDIR=$DDSUB'/mdlog/';		NCDIR=$DDSUB'/nc/';		PDBDIR=$DDSUB'/pdbrst/'

echo -e "$MDTXT" > $LOGDIR/MDcomment.txt

### NO CHANGES REQUIRED BEYOND THIS POINT ## (for this and similar projects) 

### Define PREPRSTR line parameters to stretch PS during equilibration (SEP: distance +-5)
# Looks on PDB file with same name as prmtop (without HMR/hmr in name)
if [ ! -z $STRETCH2 ]; then
PDB="${PRMTOP/prmtop/pdb}"; PDB="${PDB//HMR/}"; PDB="${PDB//hmr/}"
LC6=$(grep C6 $PDB | tail -n 1  | awk '{print $2 }')
PREPRSTR="extend.RSTR"; IAT1=1; IAT2=$LC6; SEP=$STRETCH2
fi
# End stretching definitions

# Multi-job parameters (MDIN, PRMTOP, RSTCOOR, are for pmemd -i -p -c(initial restart) )
PREPMDIN="prep${TRUN}K_1ns.gen"		# Equilibration input file, generated below
PMEMDRUN="pmemd.cuda"			# Use 'nohup pmemd.cuda' if wished

# Initialize other variables, create folders:
[ ! -d $DD ] && mkdir $DD
[ ! -d $DDSUB ] && mkdir $DDSUB
[ ! -d $LOGDIR ] && mkdir $LOGDIR
[ ! -d $NCDIR ] && mkdir $NCDIR
[ ! -d $PDBDIR ] && mkdir $PDBDIR

LRN=$LOGDIR$ROOTNAME; 	NRN=$NCDIR$ROOTNAME;	PRN=$PDBDIR$ROOTNAME

# Read atoms to save from TOPSAVE (7th line, 1st elem); copy top files to ncdir
[[ $TOPSAVE =~ 'prmtop' ]] && WATS=$(awk '{if(NR==7) print $1}' $TOPSAVE) \
&& cp $TOPSAVE $NCDIR || WATS=0 && cp $PRMTOP $NCDIR

export CUDA_VISIBLE_DEVICES=$MYCUDA		# End set CUDA

echo -e "\n	The parameter file is ===>>	${GREEN}$PRMTOP\n${WHITE}"
echo -e "\n	TEMPERATURE:	${YELLOW}TRUN = $TRUN K${WHITE}; (TPREP = $TPREP K)\n"
[ ! -f $PRMTOP ]  && echo -e "\n $PRMTOP doesn't exist; exiting. ${normal}\n" && exit 1

DT=0.002		# MD step in fs; if 'HMR' in name, set step to 0.004 fs below;
[[ $PRMTOP =~ 'hmr' || $PRMTOP =~ 'HMR' ]] && DT=0.004 \
   && echo -e "\n\tPRMTOP seems HMR: ${YELLOW} dt set to $DT ${normal} \n" \
   || echo -e "\n\tPRMTOP seems standard: ${YELLOW} dt set to $DT ${normal} \n"

### Save frame/10ps.
TOTSAVE=$(( TRJDUR * 1000 / $STEP )); NWX=`echo "$STEP/$DT" | bc`; NL=$(( TOTSAVE * NWX ))
echo -e "\n\t${bold}TRJ duration is $TRJDUR ns; save $TOTSAVE structures/trj (x $MDENDJOB)${normal}\
         ( 1 every $STEP ps, set with NWX ) \n"

[[ $WATS = 0 ]] && echo -e "${bold}\n\t${CYAN}WATS = 0${WHITE}, SAVING ALL ATOMS ${normal} \n" \
  || echo -e "${bold}\n	!!! You are saving ${CYAN}$WATS atoms${WHITE} !!!\n${YELLOW}"
[ ! -z $STRETCH2 ] &&   echo -e "${bold}${CYAN}\n\t \
Separating atoms 1 and $LC6 in PS.${WHITE} Distance: $STRETCH2 (+/- 5) Å.${normal}\n"

read -e -p "Enter inpcrd or restart filename ${normal}(may use tab for completion): " INPCOOR
[ ! -f $INPCOOR ] && echo "File doesn't exist, start over." && exit 1

MDSTARTJOB=0	# If 'inpcrd', set to 0; if not, set restart from context (No. before '.ncrst')
[[ $INPCOOR != *"inpcrd"* ]] && RJ=$(echo $INPCOOR | grep -o -P '.{4}ncrst' | grep -o '[0-9]*') \
  && MDSTARTJOB=$((RJ+1)) && echo -e "\n\t${bold}Starting job No. $MDSTARTJOB ${normal}"
MDCURRENTJOB=$MDSTARTJOB

# The command below (>) centers coordinates before saving the restart file
cat > auImage.gen << EOL
autoimage
EOL

MDIN="mdrun${T0}K-CUDA$CVD.gen"		# MDIN, production input file, generated below
cat > $MDIN << EOL
#100ns MD
 &cntrl
  imin=0, irest=1, ntx=5, ntxo=2, ioutfm=1, ntb=2, cut=8,
  ntp=1, pres0=1.0, taup=5.0, ntr=0, ntc=2, ntf=2,
  tempi=$TRUN, temp0=$TRUN, ntt=3, gamma_ln=2.0, ig=-1,
  nstlim=$NL, dt=$DT, ntwprt=$WATS,
  ntwx=$NWX, ntpr=50000, ntwr=500000, iwrap=1,
  /
  &wt type = 'END',  /
  DISANG=$XYLRSTR  
 /
EOL
### End MDIN writing

FM=fullmin.gen			# FM, energy minimization input file creation
cat > $FM << EOL
Total system initial minimization
 &cntrl imin=1, maxcyc=5000, ncyc=1000, ntb=1, ntr=0, cut=8.0 /
EOL
### End FULLMIN writing

# If PREPRSTR is defined to elongate the PS, create restraint file
[ ! -z "$PREPRSTR" ] && SEP1=$(( SEP-5 )) && SEP2=$(( SEP+5 )) && \
cat > $NN-$PREPRSTR << EOL
 &rst iat=$IAT1,$IAT2, r1=0, r2=$SEP1, r3=$SEP2, r4=500, rk2= 1, rk3= 1, &end
EOL

# 
cat > $PREPMDIN << EOL
# Generated Amber prep file, 1 ns, heating up to 323K and ramping down to $TRUN
#
 &cntrl
 irest = 0, ntx = 1, ig = -1, ntwx = 0, ntpr = 500, ntwr = 50000, ntwx=250,
 nstlim = 500000, dt = 0.002, ntwprt=$WATS,
 ntt = 3, gamma_ln = 2, ntp = 1, taup = 5.0, ntb = 2, ntc = 2, ntf = 2, cut = 8.0, nmropt=1 /
# Use PME, skinnb=2.0[A] is default (sum of nonbonded extended to cut+skinnb)
 &ewald skinnb = 2.0, nbflag = 1, /
# T ramps: 1K -100ps- 323K -500ps- 323K -200ps- TRUN -200ps- TRUN	(1 ns total)
 &wt type = 'TEMP0', istep1 = 0, istep2 = 50000, value1 = 1.0, value2 = $TPREP, /
 &wt type = 'TEMP0', istep1 = 50001, istep2 = 250000, value1 = $TPREP, value2 = $TPREP, /
 &wt type = 'TEMP0', istep1 = 250001, istep2 = 400000, value1 = $TPREP, value2 = $TRUN, /
 &wt type = 'TEMP0', istep1 = 400001, istep2 = 500000, value1 = $TRUN, value2 = $TRUN, /
EOL
# The command '>>' appends lines to PREPMDIN file; if PREPRSTR is not null.
[ ! -z "$PREPRSTR" ] && cat >> $PREPMDIN << EOL
 &wt type = 'REST', istep1 = 1, istep2 = 100000, value1 = 0.001, value2 = 1, /
 &wt type = 'REST', istep1 = 100001, istep2 = 400000, value1 = 1, value2 = 0.01, /
 &wt type = 'REST', istep1 = 400001, istep2 = 500000, value1 = 0.01, value2 = 0.01, /
 &wt type = 'DUMPFREQ', istep1=1000 /
 &wt type = 'END', /
 DISANG=$NN-$PREPRSTR
 DUMPAVE=$NN-rstout.txt
 END  /
EOL
[ -z "$PREPRSTR" ] && cat >> $PREPMDIN << EOL
 &wt type = 'END', /
 END  /
EOL
### End PREPMDIN writing

### If 'restart job' = 0, do: 1- Full system minimization; 2- Equilibration
if [ $MDSTARTJOB -eq 0 ]; then
echo -e "\n Start full system minimization"
$PMEMDRUN -O -i $FM -o $LRN-FULLMIN.out -p $PRMTOP -c $INPCOOR -r $NRN-min.ncrst
INPCOOR=$NRN'0.ncrst'; IC=$INPCOOR		# Just clarity/convenience
echo -e "\n Start 1 ns system equilibration; output is $IC"
$PMEMDRUN -O -i $PREPMDIN -o $LRN-EQUIL.out -p $PRMTOP -c $NRN-min.ncrst -r $IC -x $NRN-EQUIL.nc
MDCURRENTJOB=1
fi

### PRODUCTION
while [ $MDCURRENTJOB -le $MDENDJOB ]; do
FECHA=`date`
# pmemd pars: -o, -c, -r, -x, -inf
MDOUT=$LRN$MDCURRENTJOB.out;	OUTRST=$NRN$MDCURRENTJOB.ncrst;	OUTMDCRD=$NRN$MDCURRENTJOB.nc
[ ! -f $INPCOOR ] && echo -e "$INPCOOR doesn't exist; exiting.\n\n" && exit 1

### cpptraj section: Autoimages restart file centering first molecule (prevents problems in long trajectories)
cpptraj -p $PRMTOP -y $INPCOOR -x $INPCOOR -i auImage.gen > "auImage$CVD.log"
cpptraj -p $PRMTOP -y $INPCOOR -x $PRN$MDCURRENTJOB"rst.pdb"	> "saverst$CVD.log" # Save restart as pdb

echo -e "\n Start new job; log: $MDOUT -- $FECHA -- CUDA = $CVD"
$PMEMDRUN -O -i $MDIN -o $MDOUT -p $PRMTOP -c $INPCOOR -r $OUTRST -x $OUTMDCRD

MDCURRENTJOB=$(($MDCURRENTJOB+1)); INPCOOR=$OUTRST
done
echo "ALL DONE; $FECHA. \n"
