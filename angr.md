# angr : 
This part is composed of the notes I took during the reading of the angr [documentation](https://docs.angr.io/en/latest/core-concepts/index.html). For this version, my notes will follow the same plan as choosen by the redactors of the documentation but this may evolve later. 

## Core concepts : 
This section is presented using basic code snippets examples to present the main concepts behind the angr implementation.    

### Project creation : 
```python 
import angr, monkeyhex 
proj = angr.Project("/bin/test")
```
The project is described as the control base in angr. Almost all the utilities and tools are created via the ``Project`` class.   
Examples of some basic project's properties : 
- ``proj.arch`` : the architecture for which the program is compiled for. 
- ``proj.entry`` : the entrypoint of the program. 

### Some project's properties, results of the loading process : 
The process of loading a binary is technically described here TODO : make a link to the loading part of technical binary analysis. In angr, the module handling this part is called CLE. Here are some properties populated by the module during the loading process : 
- ``proj.loader`` : indicate if the binary is loaded and the address space of the binary. 
- ``proj.loader.shared_objects`` : shared objects loaded with the binary and their memory ranges. 
- ``proj.loader.min_addr/proj.loader.max_addr`` : min and max address of the memory range of the main binary. 
- ``proj.loader.main_object`` : this returns the main object loaded by CLE. 
- ``proj.loader.main_object.execstack`` : return if the binary has an executable stack. 
- ``proj.loader.main_object.pic``: return if the binary is position independent.  

### Factory : 
angr is composed of many classes. A facility has been created to retrieve most common objects : the ``Factory`` class. Three objects are presented in the next lines : ``Blocks`` , ``States`` and ``SimulationManager``. 

#### Blocks : 
angr analyzes code in units of basic blocks, the ``Block`` object allows to retrieve informations related to the corresponding block. 

- ``block = proj.factory.block(proj.entry)`` : lifts a block of code from the provided adddress.
- ``block.pp()`` : prints the corresponding disassevmbly. 
- ``block.instructions`` : prints the number of instructions that compose the block.
- ``block.instructions_addr`` : prints the addresses of the instructions. 
- ``block.capstone`` : returns the capstone disassembly python object. 
- ``block.vex`` : return the corresponding VEX object.






