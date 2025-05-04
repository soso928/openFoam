LES simulation of a vertical 3D round jet (no buoyancy)
(video: https://www.youtube.com/watch?v=VN0vufcgQMM)

Geometry:
1. Use either SketchUp (free version) or freeCAD to generate .stl
2. This example uses SketchUp
3. Use Salome to import the .stl and group the surfaces into 'nozzle' and 'pipe' - this is necessary so that you can specify the boundary conditions at these patches separately in openFOAM
4. Export the grouped surfaces from Salome as .stl (nozzle.stl and pipe.stl)
5. snappyHexMesh can then be used for meshing (like in this example)
6. Alternatively, you may also use salome to mesh (1D-2D-3D netgen), export as .unv, and run ideasUnvToFoam to generate the mesh (e.g. ideasUnvToFoam mesh.unv) - example shown in 3DJet_tetrahedron folder

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

Hardward: Just a regular mini desktop - in this case the Apple Mac mini 2024 (Apple M4 chip with 10-core CPU, 10-core GPU)
