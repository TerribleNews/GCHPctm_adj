# GCHP Adjoint

This is a fork of the source code repository for GEOS-Chem. This fork is for development of the adjoint of GCHP. Hopefully the code in it will be rolled into the [gcst repo](https://github.com/geoschem/GCHP). 

If you are used to the adjoint version of GEOS-Chem classic, the new build and run systems will be very different for you. Before trying to build GCHP adjoint, I recommmend you at least familiarize yourself with the new [GCHP build process](https://gchp.readthedocs.io/en/latest/getting-started/quick-start.html). If you want to try to build the GEOS-Chem Support Team's standard version before you try this, the environment files listed below should help you. 


I have structured the code in such a way that downloading this repo and doing the regular `cmake` and `make` without any options will give you the standard forward executable. Because GCHP uses `cmake`, you can, in the same code directory tree, have multiple different configurations of the code. In this way you could have a `build` directory with the standard forward executable and a `build.adj` directory with the adjoint-enabled executable. The adjoint executable is set up so that separate runs of the same executable with different runtime configurations are required for the forward and reverse runs. My hope is that this makes the systems more flexible and also more portable to systems like [JEDI](https://www.jcsda.org/jcsda-project-jedi) in the future

**Documentation for the standard, forward-model code**: https://gchp.readthedocs.io/en/latest/

## Building GCHP Adjoint

These instructions are for building GCHP adjoint on the NASA Pleiades system. Most of the instructions should work the same elsewhere but the environment configuration will be slightly different. 


The first environment file you will need is the one you will save in your home directory and link to each run directory afer you've created it. Save these lines in `~/gchp.ifort18_sgimpi_pleiades.env`

```
#!/bin/bash
# Based on seastham's home/pleiades.basrch

# Builds NetCDF on Pleiades using the standard compiler set, ready for GCHP
#module load git

# load ifort, sgi mpi, and netcdf
module load comp-intel/2018.3.222
module load mpi-sgi/mpt
module load hdf4/4.2.12 # required by netcdf
module load hdf5/1.8.18_mpt # also required by netcdf?
module load netcdf/4.4.1.1_mpt


export ESMF_COMM=mpi
export ESMF_COMPILER=intel

# Tell GCHP where the MPI binaries, libraries and so on can be found
export MPI_ROOT=$( dirname $( dirname $( which mpiexec ) ) )

# Set up the compilers
)export FC=ifort
export F90=$FC
export F9X=$FC
export F77=$FC
export CC=gcc
export CXX=g++

export OMP_STACKSIZE=500m
ulimit -s unlimited

# Needed one NetCDF is installed
export NETCDF_HOME=$(nc-config --prefix)

export GC_BIN="$NETCDF_HOME/bin"
export GC_INCLUDE="$NETCDF_HOME/include"
export GC_LIB="$NETCDF_HOME/lib"

# If using NetCDF after the C/Fortran split (4.3+), then you will need to
# specify the following additional environment variables
export NETCDF_FORTRAN_HOME=$(nf-config --prefix)
export GC_F_BIN="$NETCDF_FORTRAN_HOME/bin"
export GC_F_INCLUDE="$NETCDF_FORTRAN_HOME/include"
export GC_F_LIB="$NETCDF_FORTRAN_HOME/lib"


export PATH=${NETCDF_FORTRAN_HOME}/bin:$PATH
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${NETCDF_FORTRAN_HOME}/lib

# Add NetCDF to path
export PATH=$PATH:${NETCDF_HOME}/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${NETCDF_HOME}/lib

# Disable OpenMP
export OMP_NUM_THREADS=1

# Attempt an alias for mpifort if not found
#command -v mpifort >/dev/null 2>&1 || alias mpifort=mpif90
export PATH=${PATH}:${HOME}/mpi_extra

# Set path to GMAO Fortran template library (gFTL)
export gFTL=$(readlink -f ./gFTL)

# Set ESMF optimization (g=debugging, O=optimized (capital o))
export ESMF_BOPT=O
```

### Downloading the code

Move to your source code diretory, e.g. `cd /noackup/[myusername]/GCHP/`. Clone the `add_ems_adj` branch of this repo. To use the ssh version, make sure you have your github ssh key added to your ssh-agent. Otherwise, use the second https version.
```
$ git clone -b add_ems_adj git@github.com:TerribleNews/GCHPctm_adj.git Code.GCHPadj
```
or
```
$ git clone -b add_ems_adj https://github.com/TerribleNews/GCHPctm_adj.git Code.GCHPadj
```
then
```
$ cd Code.GCHPadj
```

Create a file called `env.rc` in this directory and put the following lines in it:
```
module purge

# load ifort, sgi mpi, and netcdf
module load comp-intel/2018.3.222
module load mpi-sgi/mpt
module load hdf4/4.2.12 # required by netcdf
module load hdf5/1.8.18_mpt # also required by netcdf?
module load netcdf/4.4.1.1_mpt
module load python3/3.7.0 # maybe required for f2py?

export PATH=/nobackupp12/clee59/software/cmake-3.16.3-Linux-x86_64/bin/:$PATH

export ESMF_ROOT=/nobackupp12/clee59/software/

export OMP_STACKSIZE=500m
ulimit -s unlimited

export FC=ifort
export CC=icc
export CXX=icpc
```

Create a build directory and cd into it:
```
$ mkdir build
$ cd build
```

Use cmake to configure the build with the adjoint options turned on, then build the executable. This can be done on the head node or in an interactive compute session on pleaides. Make sure you source the `env.rc` file we created in the previous step.
```
$ . ../env.rc
$ cmake .. -DADJOINT=yes -DREVERSE_OPERATORS=yes
$ make -j
```

Create a base directory in which to store your run directories:
```
$ mkdir /nobackup/[myusername]/GCHP_adj_runs
```

From your `Code.GCHP_adj` directory, move into the `run` directory and execute the `createRunDir.sh` script, answering with the values after the `$` prompts below. Please ensure that you use `/nobackup/clee59/ExtData` as your path for ExtData, not your own nobackup dirctory.

```
$ cd run
$ ./createRunDir.sh

===========================================================
GCHP RUN DIRECTORY CREATION
===========================================================

-----------------------------------------------------------
Define path to ExtData.
This will be stored in /home###/[yourusername]/.geoschem/config for future automatic use.
-----------------------------------------------------------

-----------------------------------------------------------
Enter path for ExtData:
-----------------------------------------------------------
$ /nobackup/clee59/ExtData

-----------------------------------------------------------
Choose simulation type:
-----------------------------------------------------------
   1. Full chemistry
   2. TransportTracers
   3. CO2 w/ CMS-Flux emissions
$ 3

-----------------------------------------------------------
Choose meteorology source:
-----------------------------------------------------------
  1. MERRA-2 (Recommended)
  2. GEOS-FP
$ 1

-----------------------------------------------------------
Enter path where the run directory will be created:
-----------------------------------------------------------
$ /nobackup/[myusername]/GCHP_adj_runs

-----------------------------------------------------------
Enter run directory name, or press return to use default:

NOTE: This will be a subfolder of the path you entered above.
-----------------------------------------------------------
$ gchp_c24_adj

  -- You may modify these settings in runConfig.sh.
ERROR: Restart file specified in runConfig.sh not found: initial_GEOSChem_rst.c24_CO2.nc

-----------------------------------------------------------
Do you want to track run directory changes with git? (y/n)
-----------------------------------------------------------
$ n

Created /nobackup/[myusername]/GCHP_adj_runs/gchp_c24_adj
```

In order to run:
1. go to you run run directory (below)
2. copy the missing restart file (below)
3. set your gchp.env link (below)
4. fill in your own account and email in `gchp.pleiades.run` and `gchp.adjoint.run` (not below, just open those files in your favourite editor and the missing values will be in the first few PBS lines)

```
$ cd /nobackup/[myusername]/GCHP_adj_runs/gchp_c24_adj
$ cp /nobackup/clee59/SHARED/initial_GEOSChem_rst.c24_co2.nc ./
$ ./setEnvironment.sh ~/gchp.ifort18_sgimpi_pleiades.env
```

You are finally read to run:
```
$ qsub gchp.pleiades.run && tail -F gchp.log
```
Once that has finished you will need to move (not copy) the final checkpoint file to be your new restart file for the reverse run, then launch the reverse run:
```
$ mv gcchem_internal_checkpoint gcchem_internal_restart.20140901_060000z.nc4
$ qsub gchp.adjoint.run && tail -F gchp.log
```

That's it. Explore the netcdf files in `OutputDir`.
