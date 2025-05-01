LES simulation of a vertical 3D round jet (no buoyancy)

Run the following commands in sequence:
1. blockMesh
2. surfaceFeatureExtract
3. snappyHexMesh -overwrite
4. decomposePar -force
5. mpirun -np 10 pimpleFoam -parallel | tee

Run the following commands to post-process:
1. reconstructPar
2. foamToVTK

The VTK can be viewed in paraview.

Note: A simple case of running the LES but not validated with experimental data. The spreading rate of even a simple jet can be challenging to simule using stand k-epsilon model of RANS.
