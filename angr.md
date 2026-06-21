# angr : 
This part is composed of the notes I took during the reading of the angr [documentation](https://docs.angr.io/en/latest/core-concepts/index.html). As a first reading, the notes present here are almost entirely similar of the ones presented in the documentation, I just performed some rephrasing I better learn and understand what I read. For this version, my notes will follow the same plan as choosen by the redactors of the documentation but this may evolve later. 

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
angr is composed of many classes. A facility has been created to retrieve the most common objects : the ``Factory`` class. Three objects are presented in the next lines : ``Blocks`` , ``States`` and ``SimulationManager``. 

#### Blocks : 
angr analyzes code in units of basic blocks, the ``Block`` object allows to retrieve informations related to the corresponding block. 

- ``block = proj.factory.block(proj.entry)`` : lifts a block of code from the provided adddress.
- ``block.pp()`` : prints the corresponding disassevmbly. 
- ``block.instructions`` : prints the number of instructions that compose the block.
- ``block.instructions_addr`` : prints the addresses of the instructions. 
- ``block.capstone`` : returns the capstone disassembly python object. 
- ``block.vex`` : return the corresponding VEX object.

#### States : 
All execution performed with angr is a representation of a simulated program state a ``SimState``. When a program is loaded using the ``Project``, it only represents an initialization image of the program. 

- `` state = proj.factory.entry_state()`` : retrieve the SimState of the entry point of the program.   
Composition of a SimState : program's memory, registers, filesystem data etc... Or any data that could be changed by the program execution.   
- ``state.regs.rip`` : retrieve the rip value 
- ``state.mem.[proj.entry].int.resolved`` : interpret the memory at the entry point as C int.   

angr only works with bitvectors in its core. Here are some tricks to convert python ints to bitvectors : 
<details>
  <summary>BitVectors/python ints conversion</summary>
  
- ``bv = claripy.BVV(0x1234, 32)`` : create a BitVecVal initialized with 0x1234. 
- ``state.solver.eval(bv)`` : convert to python int
- ``state.regs.rsi = claripy.BVV(3, 64)`` : store a BitVecVal in rsi register
- ``state.mem[0x1000].long = 4`` : this will automatically be converted in a BV64
</details> 

Some details about the memory API usage : 
- ``mem[index]`` : access the corresponding address 
- ``.<type>`` : to specify how the memory should be interpreted
- ``.resolved`` : to get the stored value as a BV 
- ``.concerte`` : to get the value as a python int
By accessing the memory or registers, it is possible to encounter symbolic variables also called BitVector in z3 world. 

#### SimulationManagers : 
A state is a representation of the program at a given point in time. The SimulationManager is the interface to perform execution and go from stateX to stateX+1. 
- ``simgr = proj.factory.simulation_manager(state)`` : first, create the simulation manager, it takes a state or a list of states as a parameter. 
- ``simgr.active`` : retrieve the active stash. A simulation manager is composed of different stashes that will be discussed later. The most basic one is the active stash. A stash is composed of different states. 
- ``simgr.active[0]`` : retrieve the first state of the active stash. 
- ``simgr.step()`` : perform a basic bloc step of symbolic execution. It is now possible to check again the active stash. The state held in the active stash has been updated. However, the original state itself has not been updated, it's possible to use a state as a basis for different symbolic rounds of execution. 
- ``simg.active[0].regs.rip`` : has been updated by the symbolic execution. 
- ``state.regs.rip`` : is unchanged as described above. 


## Binary Loading : 


## Symbolic expressions and constraints solving : 
angr is able to perform symbolic execution. Each arithmetic operation with a symbolic variable will create an AST in the engine. Then these ASTs can be translated into constraints for an SMT solver (z3 is back!!). 

<details>
  <summary>Some more bitvectors operations</summary>
  
- ``bv = claripy.BVV(0x1234, 32)`` : create a BitVecVal initialized with 0x1234. 
- ``bv.zero_extend(32)`` : pad the bitvector on the left with the given number of zero bits 
- ``bv.sign_extend(32)`` : pad with a duplicated of the highest bit value to preserve the value of the bitvector under signed integer semantic
</details> 

### Symoblic variables and ASTs : 

- ``x = claripy.BVS('x', 64)`` : declare a 64 bits symoblic variable   
It's then possible to perform any operation on this symbolic variable but instead of retrieving a value, we will retrieve an AST. 
- ```(x + 1) / 2 ===> <BV64 (x_9_64 + 0x1) / 0x2>``` : Here we can observe the AST produced by adding one and dividing by 2.   
A simple symoblic variable is also an AST but with one layer deep only. 
```python
tree = (x+1)/x+2
```
- ``tree.op`` : allows to retrieve a strig corresponding to the current operation, in that case "__floordiv__"
- ``tree.args`` : allows to retrieve the operations inputs, generally, these are other ASTs. 


### Symbolic constraints : 
Peroforming a comparison between two ASTs will generates another AST : a symbolic boolean variable.   
By default, all comparisons are unsigned as in z3.   
**This illustrates the fact that a comparison between two elements should never be used in a if or while statement as the result is an AST that could never have a concrete value.** Instead, angr provides the solver utility : 
- ``test = 1 == 2 ; state.solver.is_true(test)`` : will return a boolean value that could be used in a comparison. 

## Constraints solving : 
How to get evaluation of a symoblic expression. First get symbolic variables, add constraints to a state and retrieve evaluated value. 
```python 
state.solver.add(x>y)
state.soler.add(y>2)
state.solver.eval(x)
```
Adding the constraints to the state will force the solver to consider constraints as assertions. It's also possible to request for y value and if no constraints have been added to the state between the declaration and the evaluation, the returned value will be consistent with the x value returned previously.  
- Test if a state is satisfiable regarding the defined constraints : ``state.satisfiable()``.   
Eval is used here to convert to convert a bitvector into a python primitive, this is why it can also be used to perform simple conversions.  
Variables are not tied to a state, they exist independently and can be used in other states. 


## Basic Execution : 
Simulation manager, basic execution and symoblic execution.  
- ``state.step()`` : perform one step of symbolic execution and return an object called ``angr.engines.successors.SimSuccessors``   
Symbolic execution in angr will produce many successors states. All of them can be classified and this is the main work in angr. the ``successors`` property return a list of "normal" successors states.  
**Basic example :** ``if (x>4)`` and x is defined as a symbolic variable.   
In angr core the comparison will be performed and return an AST ``<Bool x_32_1 > 4>``.   
From this, angr will create two different states, one with the valid comparison and the other one with invalid comparison.   
To state one, the constraint ``x>4`` will be added. To state two, the constraint ``!(x>4)`` will be added.  

This is where i realized what going from z3 to angr means. Angr works is entirely based on classifying states or find a way to discard the not interesting ones. The fauxware example available [here](https://docs.angr.io/en/latest/core-concepts/states.html#basic-execution) is the exact illustration of this. 