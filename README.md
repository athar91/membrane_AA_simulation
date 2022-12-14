# membrane_AA_simulation
**1. Orient protein "OPM membranes" [PPM2] [https://opm.phar.umich.edu/]**



If there are problems, just download the prealigned pdb structure and align your structure using pymol

In any case, from here, separate ligand and protein structures.



grep "MOL" XXX.pdb > ligand.pdb 

grep -v "MOL" XXX.pdb > protein.pdb 



Check protein structures with 



source /etc/modules.sh

module load amber20

source $AMBERHOME/amber.sh



 pdb4amber -i *pdb –o *pdb 





pdb4amber protein_prep.pdb -o protein_ok.pdb



**2. Parametrize the ligand** 



antechamber -i ligand.pdb -fi pdb -o ligand.mol2 -fo mol2 -c bcc -s 2 -rn MOL -at gaff2 ###alternatively ligands can be parameterized using QM

parmchk2 -i ligand.mol2 -f mol2 -o missing_gaff2.frcmod -at gaff2 

tleap -f convert.leap ####though tleap can automatically generate all parameter at once, but it's better to parameterize ligand separately,a/c to my experience 





 *********convert.leap********************* 

source leaprc.gaff2 

loadamberparams missing_gaff2.frcmod 

ligand=loadmol2 ligand.mol2 

saveoff ligand ligand.off 

quit 



**3.  Use packmol-memgen**




packmol-memgen --pdb protein_ok.pdb --lipids SDPC:POPC:CHL1 --ratio 7:7:6 --preoriented --salt --salt_c Na+ --saltcon 0.15 --dist 10 --dist_wat 15 --notprotonate



Where: protein_ok.pdb is the file coming from pdb4amber



#Original command

#packmol-memgen -p ${pdb_move} --salt --salt_c K+ --saltcon 0.15 --salt_override --parametrize --ffwat opc --ffprot ff19SB --gaff2 --ligand_param ../mol.frcmod:../mol.off --lipids POPE:POPG --ratio 2:1 --dist 17.5 --dist_wat 17.5 --keepligs --n_ter in --random --movebadrandom --lip_offset 1.1



4. ./shift_membrane.py -i protein_ok.pdb -m bilayer_protein_ok.pdb -o membrane.pdb [this python script needs to be in the working dir]



Take the PBC coordinates: e.g. Box X, Y, Z: 77.728 78.118 108.024

And use them in the leap script:

*********************tleap*********************
source /usr/local/amber20/dat/leap/cmd/leaprc.gaff2
source /usr/local/amber20/dat/leap/cmd/leaprc.protein.ff19SB
source /usr/local/amber20/dat/leap/cmd/leaprc.water.opc
source /home/mathar/andrea/leaprc.lipid21 
loadamberparams frcmod.ions1lm_126_iod_opc 
loadamberparams ligand.off 
loadamberparams missing_gaff2.frcmod
drug = loadmol2 ./ligand1.mol2 
receptor = loadpdb ./protein_ok.pdb  
membrane = loadpdb ./membrane.pdb 
system=combine{ receptor membrane drug } 
set system box {89.160 89.160 143.030}  # <- HERE THE COORDINATES TAKEN FROM “shift membrane”.
saveoff system system.lib
saveamberparm system system.prmtop system.inpcrd 
savepdb system system.pdb 
quit
****************************************


Note, if the *.py script doesn’t work, try to get the coordinates from the “FORCED” bilayer file, but then create the membrane starting not from the forced one.



Note 2: tleap needs the lipid21 files (they are at least 2, the .lib and the .dat one) in its working dir, together with “missing_gaff2.frcmod” and all the ligand files (such as ligand*.off/mol2). So, check carefully if everything is in the right place before using “teap -f script.leap”



5. Use the following commands, in a script (parmed -i script) for HmassRep.


**********************script for system.parmed***********
parm system.prmtop
HMassRepartition
outparm system.hmass.prmtop
quit
******************************


**6. Run the simulations. **
Simulation by Sander 

sander -O -i 01_Min.in -o 01_Min.out -p parm7 -c rst7 -r 01_Min.ncrst -inf 01_Min.mdinfo 

sander uses a consistent syntax for each step of MD simulation. Here is a summary of the command line options of sander: 

-O Overwrite the output files if they already exist 

-i 01_Min.in ------------Choose input file (default mdin) 

-o 01_Min.out ---------------------Write output file (default mdout) 

-p parm7 -------------------------Choose parameter and topology file parm7, "prmtop", "top") 

-c rst7 --------------------------Choose coordinate file rst7, "inpcrd", "restrt", "rst7", "crd" 

-r 01_Min.ncrst -----------------------------Write output restart file with last frame coordinates and velocities (default restrt) 

-inf 01_Min.mdinfo ----------------------------Write MD info file with simulation status (default mdinfo) 

-r   restart file (last set of xyz coordinates from the simulation) 
-x   file with trajectory (RAMP1_md.nc) 

$sander -O -i 02_Heat.in -o 02_Heat.out -p parm7 -c 01_Min.ncrst \ 
-r 02_Heat.ncrst -x 02_Heat.nc -inf 02_Heat.mdinfo 

 

Load Amber trajectory in VMD 

vmd *.prmtop *.nc 

 

Compiled script for membrane simulation is there in octopus, run_MD.sh 

 

Note: ncrst or rst file are complete coordinates, they ca easily be converted into pdb using the commands: 

ambpdb -p system.prmtop -c 01_Min.ncrst > 01_Min_out.pdb 

 

******** 

How to extract frame in amber (check for analysis Amber Basic Workshop - Tutorial 3 - section 6 (ambermd.org)) 

> $AMBERHOME/bin/cpptraj TC5b.prmtop < *.trajin > extract_frame9_2455.out 

And the content of .trjin file is till initial frame to end frame 

trajin equil9.nc 2455 2455 

trajout lowest_energy_struct.pdb pdb 

************************ 
