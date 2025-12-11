# MXene_MD-Simulation
#Compilation file for writting lammps code and running MD simulation of desalination using MXene  in lammps software
# make_ti3c2f2_6x6.py
# Usage: python make_ti3c2f2_6x6.py Ti3C2F2.cif
# Requires: pymatgen (pip install pymatgen), numpy (usually installed with pymatgen)
#
# This script:
#  loads the CIF given on the command line
#  makes a 6x6x1 supercell
#  writes a LAMMPS data file (atomic style), an XYZ and a PDB
#  writes a basic LAMMPS input script (no potential included)
#  creates a ZIP file containing the generated files

import sys
from pathlib import Path
from pymatgen.core import Structure
from pymatgen.io.lammps.data import LammpsData
import zipfile

if len(sys.argv) < 2:
    print("Usage: python make_ti3c2f2_6x6.py path/to/Ti3C2F2.cif")
    sys.exit(1)

cif_path = Path(sys.argv[1]).expanduser()
if not cif_path.exists():
    print("CIF file not found:", cif_path)
    sys.exit(1)

# load structure
s = Structure.from_file(str(cif_path))

# If the CIF is a conventional cell of Ti3C2, ensure F atoms are present.
# (If your CIF already contains F (Ti3C2F2) this is fine. If not, you must add termination atoms.)
print("Original structure: {} atoms, lattice: {}".format(len(s), s.lattice.matrix.tolist()))

# Make a 6x6x1 supercell (in-plane replication)
supercell = s.copy()
supercell.make_supercell([6,6,1])

# write LAMMPS data (atomic style)
# NOTE: LammpsData requires mapping of species -> types automatically.
lmp = LammpsData.from_structure(supercell, atom_style="atomic")
data_filename = Path("ti3c2f2_6x6.data")
lmp.write_file(str(data_filename))
print("Wrote LAMMPS data:", data_filename)

# write XYZ
xyz_filename = Path("ti3c2f2_6x6.xyz")
with xyz_filename.open("w") as f:
    f.write(str(len(supercell)) + "\n")
    f.write("Ti3C2F2 6x6 supercell\n")
    for site in supercell:
        sym = site.specie.symbol
        x,y,z = site.coords
        f.write(f"{sym} {x:.6f} {y:.6f} {z:.6f}\n")
print("Wrote XYZ:", xyz_filename)

# write PDB for visualization in VMD (simple ATOM lines)
pdb_filename = Path("ti3c2f2_6x6.pdb")
with pdb_filename.open("w") as f:
    f.write("HEADER    Ti3C2F2 6x6\n")
    for i,site in enumerate(supercell, start=1):
        sym = site.specie.symbol
        x,y,z = site.coords
        atomname = sym[:2].upper().ljust(2)
        f.write(f"ATOM  {i:5d} {atomname:>4s} TIX A   1    {x:8.3f}{y:8.3f}{z:8.3f}  1.00  0.00           {sym:>2s}\n")
    f.write("END\n")
print("Wrote PDB:", pdb_filename)

# write a simple LAMMPS input script (NO potential included)
in_filename = Path("in.ti3c2f2_6x6")
in_text = """# Basic LAMMPS input - edit to add a potential (Reax/c or other)
units metal
atom_style atomic
read_data ti3c2f2_6x6.data

# >>> YOU MUST ADD A PAIR_STYLE/PAIR_COEFF APPROPRIATE FOR Ti-C-F SYSTEM HERE <<<
# Example placeholder for Reax (you MUST provide a validated parameter file .ff):
# pair_style reax/c NULL
# pair_coeff * * /path/to/your/reaxff_param_file Ti C F

neighbor 2.0 bin
neigh_modify every 1 delay 0 check yes

fix 1 all nvt temp 300.0 300.0 0.1
thermo 100
thermo_style custom step temp pe etotal
timestep 0.001
run 1000
"""
in_filename.write_text(in_text)
print("Wrote LAMMPS input script (edit to add potential):", in_filename)

# bundle into a zip
zipname = Path("ti3c2f2_6x6_files.zip")
with zipfile.ZipFile(zipname, "w", compression=zipfile.ZIP_DEFLATED) as zf:
    zf.write(data_filename)
    zf.write(xyz_filename)
    zf.write(pdb_filename)
    zf.write(in_filename)
print("Created zip:", zipname)

print("\nDONE. The zip contains: data, xyz, pdb, and a basic input script.\nIMPORTANT: this data file has atomic positions only. You MUST supply a validated potential for Ti-C-F\n(e.g., a published ReaxFF parameter file) before production MD or any energy calculations.\n")
