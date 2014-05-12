# pdbremix

`pdbremix` is a library for computational structural biology.

The library consists of:

1. tools to analyze and view PDB structures
2. tools to run MD simulations and analyze trajectories
3. python interface to analyze PDB structures
4. python interface to run molecular-dynamics simulations
5. python interface to analyze trajectories

`Pdbremix` is written in pure Python and works with PyPy for useful speed-up.

## Installation

Install with `pip` is easiest:

	> pip install pdbremix

Otherwise download from github and install:

	https://github.com/boscoh/pdbremix/master/tarball

From here, you can access the unit tests and example files.

There are many wonderful tools in structural biology that have less-than-stellar interfaces. `pdbremix` wraps some of these tools with better interfaces and extra functionality. 

To check which of these tools that `pdbremix` can access from the path:

	> checkpdbremix

If you know where the binaries are, or if you want to add exotic flags to the binaries, edit the confguration file that is given with the `-o` flag:

	> vi `checkpdbremix -o`


## Tools to analyze PDB structures

`pdbremix` provides a bunch of tools to investigate PDB structures.

### Standalone tools in Pure Python

Some of them can be used straight out of the box:

- `pdbfetch` fetches PDB files from the RCSB website
- `pdbheader` displays summary of PDB files
- `pdbseq` displays sequences in a PDB
- `pdbchain` extracts chains from a PDB
- `pdbcheck` checks for common defects in a PDB
- `pdbstrip` cleans up PDB for MD simulations

These tools implement standard structural biology algorithms:

- `pdbvol` calculates volume of a PDB
- `pdbasa` calculates accessible surface-area of a PDB
- `pdbrmsd` calculates RMSD between PDB files

For these algorithmic tools, you get a large speed gain if you run them  through `pypy`:

	> pdbfetch 1be9
	> pdbstrip 1be9.pdb
	> pypy `which pdbvol` 1be9.pdb

Help for the tools is available with the `-h` option. 


### Wrappers around external tools

These command-line tools provide a better interface to external tools for some rather common structural biology use cases.

- `pdbshow` displays PDB structures in PYMOL with extras.

	1. provides useful display defaults: coloring by chains, ribbons on, sticks for sidechains. 
	2. Residue centering
	3. coloring by B-factor.
	4. removes solvent to view MD-generated files

- `pdboverlay` display homologous proteins using MAFFT, THESEUS and PYMOL.

	I believe this provides the best solution. Use one of the best sequence alignment tools to get the sequence alignment. Use the smart weighted RMSD alignment of THESEUS, and display this in PYMOL using a display schemes that highlights RMSD the homology between proteins.

- `pdbinsert` fill gaps in a PDB file with MODELLER

	Many PDB files have gaps in their structure. If you want to use them for simulation, you will have to patch these gaps. Now your answer is probably MODELLER but MODELLER is not designed for casual use. For this very common case:



## Simulation and Trajectory Tools

`pdbremix` provides a simplified interface to run molecular-dynamics on 3 standard MD packages: AMBER11+, GROMACS4.5+ and NAMD2.8+.

### Making topology files

To achieve this abstraction, `pdbremix` assumes all the files of a simulation will have a common basename. To generate these files from a PDB, run:

	> pdbfetch 1be9
	> pdbstrip 1be9.pdb
	> pdb2top 1be9.pdb sim AMBER11

This will make: 

1. sim.top, sim.crd - is all you need to run an AMBER simulation. 
2. sim.pdb - is a fleshed out PDB that is populated with AMBER generated hydrogen atoms, need for positional constraints.

The other packages are:
- AMBER11-GBSA
- NAMD2.8
- GROMACS4.5

### Running simulations

### Trajectory analysis

 `pdbremix` assumes that associated files for an MD trajectory has a *common basename*. 

In AMBER:

1. topology: md.top
2. coordinates: md.trj 
3. velocities: md.vel.trj

In GROMACS:

1. topology: md.top and associated md.\*.itp files
2. restart coordinates: md.gro
3. coordinates/velocities: md.trr 

In NAMD:

1. topology: md.psf
2. coordinates: md.dcd 
3. velocities: md.vel.dcd

Once named properly, these tools can be used:

- `trajstep` extracts basic parameters of atrajctory
- `trajvar` calculates energy and RMSD of trajectory

Use these tools to display trajectories: 

- `trajvmd` display trajectory in VMD *recommended*
- `trajchim` display trajectory in CHIMERA
- `trajpym` display trajectory in PYMOL *AMBER only*

These are some package specific tools: 

- `traj2amb` converts NAMD/GROMACS to AMBER trajectories _without_ solvent
- `grotrim` trim GROMACS .trr trajectory files



## Python interface to PDB structures

### Vector geometry library

As in any structural biology library, we provide a vector geometry library, which is called `v3`:

	from pdbremix import v3

`v3` was designed to be function-based, with vector and transform objects subclassed from arrays. This allows the library to easily switch between a pure Python version and a numpy-dependent version. If you want just the python version:

	import pdbremix.v3array as v3

Or the numpy version: 

	import pdbremix.v3numpy as v3

Vectors are created and copied by the `vector` function:

	v = v3.vector() # the zero vector
	z = v3.vector(1,2,3)
	w = v3.vector(z) # a copy

As vectors are subclassed from arrays, to access components:

	print v[0], v[1], v[2]

Most functions return by value, except for `set_vector`, which changes components in place:

	v3.set_vector(v, 2, 2, 2)

Here are a set of common vector operationsnthat returns by value:

	mag(v)
	scale(v, s)
	dot(v1, v2)
	cross(v1, v2)
	norm(v)
	parallel(v, axis)
	perpendicular(v, axis)

Vectors will be used to represent coordinates/points, velocities, displacements etc. Functions are provided to measure their geometric properties:

	distance(p1, p2)
	vec_angle(a, b)
	vec_dihedral(a, axis, c)
	dihedral(p1, p2, p3, p4)
	
	normalize_angle(angle)
	degrees(radians)
	radians(degrees)
	
	get_center(crds)
	get_width(crds)

#### Affine Transforms

We also need a representations for affine transforms, which involve a rotation and a translation. Such a transform `matrix` is designed to transform a vector `v`:

	v3.transform(matrix, v)

Most of the time, you would build a transform from these basic generating functions:

	v3.identity()
	v3.rotation(axis, theta)
	v3.rotation_at_center(axis, theta, center)
	v3.translation(t)
	v3.left_inverse(m)

And combine them in the correct sequence:

	c = v3.combine(a, b)

But if you do need to mess around with a transform, then you need to know that a transform is represented by a 4x3 matrix, of which the elements are accessed by:

	matrix_elem(matrix, i, j, val=None)

The matrix consists of 2 parts:

1. 3x3 rotational component:

		matrix_elem(m, i, j) for i=0..3, j=0..3

2. 3x1 translational component:

		matrix_elem(m, 3, i) for i=0..3 


#### Testing Vectors and Transforms

Finally, we introduce functions to test similarity for vectors and transforms:
 
	is_similar_mag(a, b, small=0.0001)
	is_similar_matrix(a, b, small=0.0001)
	is_similar_vector(a, b, small=0.0001)

And a bunch of random geometric object generators, used for testing:  

	random_mag() # random positive float from [0, 90]()
	random_real() # random float from [-90, 90]() 
	random_vector()
	random_rotation()
	random_matrix()


### Reading a PDB structure in a Soup

Here, we look at how to manipulate PDB structure. First, let's grab a PDB structure from the website using the `fetch` module:

	from pdbremix import fetch
	fetch.get_pdbs_with_http(['1be9'])

The main object for manipulating PDB structures is the Soup object in `pdbatoms`. We can read a Soup from a PDB file:

	from pdbremix import pdbatoms
	soup = pdbatoms.Soup("1be9.pdb")

### Soup as a list of atoms

A Soup is essentially a collection of atoms, which we can grab by:

	atoms = soup.atoms()

An atom has attributes:

  - pos (v3.vector)
  - vel (v3.vector)
  - mass (float)
  - charge (float)
  - type (str)
  - element (str)
  - num (int)
  - chain\_id (str)
  - res\_type (str)
  - res\_num (str)
  - res\_insert (str)
  - bfactor (float)
  - occupancy (float)
  - alt\_conform (str)
  - is\_hetatm (bool)

Since `atom.pos` and `atom.vel` are vectors, these can be manipulated by functions from the `v3` library. Atom contains one special method `transform` that handle affine transforms:

	displacment = v3.vector(1,0,0)
	translation = v3.translation(displacement)
	atom.transform(translation)
	soup.transform(translation)

For more information about vectors, see above.

The atoms returned by soup.atoms() is a Python list, and is meant to be searched for using Python idioms.

You search through it by looping and comparing:

	hydrogens = []
	for atom in soup.atoms():
	 if atom.elem == 'H':
	   hydogens.append(atom)

### Searching through residues

As well, Soup contains a list of residues:

	residues = soup.residues()

A residue in this case represents a collection of atoms that forms a recognisable chemical group, whether a distinct molecule in the case of solvent, ions and ligands, or an actual residue that forms a polymer, as in amino acids in a protein or a nucleic acid in DNA. 

One key heuristic is that the type of atoms in a residue is unique. So the function:

   residue.atom(‘CA’)

will return an atom based on type in an atom.

So in a Soup, if you know the res atom\_type’s \_

  soup.residue\_by\_tag(‘A:15’).name(‘CA’)

Or by index:

  soup.residue(4).name(‘CA’)

You can loop through residues and atoms. To find all atoms surrounding a residue:

	for residue in soup.residues():
	 if residue.has\_atom("CA"):
	ca = residue("CA")
	for atom in soup.atoms():
	  if v3.distance(ca.pos, atom.pos) < 4:

Once you have the atoms you want, you can measure their geometric properties via the `v3` module.

Chains are not represented explicitly. In this author’s opinion, the data structure gets excessively complicated for little gain. It’s easier to just deal with it on a case-by-case basis.

- dealing with chains

- hetatms, atoms and other stuff

- solvent\_res\_type

- assign new chainIds
- adding, deleting atoms and residues
- changing properties of atoms and residues
- selecting residues and atoms

### Handling chains in PDB files

### Patching PDB structures

- pdbtext.py
- pdbtext parsing - cleaning structures
- - delete extra waters
- delete non-standard amino acids
- look for steric clashes
- identify chain breaks/missing amino acids
- reduce
- patch with modeller
- remove extra models
- alternate conformations

- common manipulations - protein.py
- moving things around
- saving b-Factors
- saving pdb files
- renumbring


## Structure Analysis

- asa.py
- rmsd.py
- raw calculation, in the only place that numpy is required, it uses the classic SVD decomposition in numpy to calculate the optimal superposition between two sets of points
- volume



### Making Images of Proteins

pymol.py

- frustrating to get the view you want, loading trajectories is a multiple step processing in viewers
- viewers Pymol, Chimera, VMD have scripting languages
- if we enforce our naming convention, can really save time
- e.g. load trajectories using a simple command line
- pipe in useful information and *transformations* 
	`pdbpym -b -c B:8 -t B:7 1be9.pdb`


## Python Interface to Molecular Dynamics

The library was originally written to do PUFF steered-molecular dynamics simulation that uses a pulsed force application. This method does not require the underlying MD package to support the method and can be carried out by manipulating restart files. The utilities to do this are:

- Restart files for maximum flexibility

- PUFF steered molecular dynamics

- Reading trajectories

Combines trajectory reading with topology
You need atomic masses and charges for calculations
Reads as coordinates and or as pdbatoms.Polymer structure
Common interface for AMBER, NAMD and GROMACS

- `pdbrestart`
- `pdbminimize`
- `pdbequil`
- `puff` runs a PUFF simulation
- `puffshow` displays pulling residues in PUFF sim

simulate.py
force.py
amber.py
gromacs.py
namd.py

traj.py

- coercing all necessary files under the same basename
- making topology, coordinate, velocity files
- choosing a reasonable robust subset of simulation parameters
- can handle proteins
	- only standard amino acids
	- checks for disulfide bonds
	- handle multiple chains
	- ignores cystallographic waters
	- ignores hydrogen atoms
- explicit waters
	- periodic cubic box
	- 10 Å padding
	- Langevin thermometer
	- Nose-Hoover barometer
	- Particle Ewald-Mesh Electrostatics
	- no bond constraints on protein
- implicit solvent
	- Generalized Born electrostatics
	- Surface Area tension hydrophobic term



# Notes
https://github.com/synapticarbors/pyqcprot
mpiexec -np 40 /home/bosco/bin/gromacs-4.0.7/bin/mdrun -v -s md.tpr -cpi md.cpt -append -deffnm md \>& md.mdrun.restart.log
gromacs ligand http://www.dddc.ac.cn/embo04/practicals/9\_15.htm
installing amber on mac http://amberonmac.blogspot.com.au/
amber ff on gromacs http://www.somewhereville.com/?p=114
ffamber http://ffamber.cnsm.csulb.edu/
http://web.mit.edu/vmd\_v1.9.1/namd-tutorial-unix.pdf
http://bionano.physics.illinois.edu/Tutorials/ssbTutorial.pdf
