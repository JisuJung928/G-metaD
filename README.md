Sample LAMMPS MD wrapper on VASP quantum DFT via client/server
coupling

`vasp_wrap.py` is a wrapper on the VASP quantum DFT
code so it can work as a "server" code which LAMMPS drives as a
"client" code to perform ab initio MD.  LAMMPS performs the MD
timestepping, sends VASP a current set of coordinates each timestep,
VASP computes forces and energy and virial and returns that info to
LAMMPS.

Messages are exchanged between MC and LAMMPS via a client/server
library (CSlib), which is included in the LAMMPS distribution in
lib/message.  As explained below you can choose to exchange data
between the two programs either via files or sockets (ZMQ).

----------------

Build LAMMPS with its MESSAGE package installed:

See the Build extras doc page and its MESSAGE package
section for details.

```bash
cd lammps/lib/message
python Install.py -m -z       # build CSlib with MPI and ZMQ support
cd lammps/src
make yes-message
make mpi
```

You can leave off the -z if you do not have ZMQ on your system.

----------------

Build the CSlib in a form usable by the `vasp_wrapper.py` script:

```bash
cd lammps/lib/message/cslib/src
make shlib            # build serial and parallel shared lib with ZMQ support
make shlib zmq=no     # build serial and parallel shared lib w/out ZMQ support
```

This will make a shared library versions of the CSlib, which Python
requires.  Python must be able to find both the cslib.py script and
the libcsnompi.so library in your `lammps/lib/message/cslib/src`
directory.  If it is not able to do this, you will get an error when
you run `vasp_wrapper.py`.

You can do this by augmenting two environment variables, either
from the command line, or in your shell start-up script.
Here is the sample syntax for the csh or tcsh shells:

```bash
export PYTHONPATH=$PYTHONPATH:/path/to/lammps/lib/message/cslib/src
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/path/to/lammps/lib/message/cslib/src
```

----------------

Prepare to use VASP and the `vasp_wrapper.py` script

Insure you have the necessary VASP input files in this
directory, suitable for the VASP calculation you want to perform:

* `INCAR`
* `KPOINTS`
* `POSCAR_template`
* `POTCAR`

Examples of all but the `POTCAR` file are provided.
The `POTCAR` file is a proprietary VASP file, so use one from your VASP installation.

Note that the `POSCAR_template` file should be matched to the LAMMPS
input script (# of atoms and atom types, box size, etc).

----------------

To run in client/server mode:

NOTE: The `vasp_wrap.py` script must be run with Python version 2, not
3.  This is because it used the CSlib python wrapper, which only
supports version 2.

Both the client (LAMMPS) and server (`vasp_wrap.py`) must use the same
messaging mode, namely `file` or `zmq`.  This is an argument to the
`vasp_wrap.py` code; it can be selected by setting the "mode" variable
when you run LAMMPS.  The default mode is `file`.

Here we assume LAMMPS was built to run in parallel, and the MESSAGE
package was installed with socket (ZMQ) support.  This means either of
the messaging modes can be used and LAMMPS can be run in serial or
parallel.  The `vasp_wrap.py` code is always run in serial, but it
launches VASP from Python via an mpirun command which can run VASP
itself in parallel.

When you run, the server should print out thermodynamic info every
timestep which corresponds to the forces and virial computed by VASP.

The examples below are commands you should use in two different
terminal windows.  The order of the two commands (client or server
launch) does not matter.  You can run them both in the same window if
you append a "&" character to the first one to run it in the
background.

--------------

File mode of messaging:

```bash
mpirun -np 1 lmp_mpi -v mode file -in in.client
python vasp_wrap.py file POSCAR_template "mpirun -np 1 vasp.x"
```

ZMQ mode of messaging:

```bash
mpirun -np 1 lmp_mpi -v mode zmq -in in.client
python vasp_wrap.py zmq POSCAR_template "mpirun -np 1 vasp.x"
```

---------------

You might have to set some environment variables for oversubscribing.
For example, the commands below can enable both processes (`vasp_wrap.py` and LAMMPS) use all cores,
and make waiting LAMMPS client not consume 100% CPU usage while waiting for MPI operations.

```bash
export PSM2_SHAREDCONTEXTS=YES                                                                                       
export PSM2_MAX_CONTEXTS_PER_JOB=8                                                                                   
export I_MPI_WAIT_MODE=1   
```
