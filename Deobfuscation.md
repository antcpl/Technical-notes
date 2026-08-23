# Technical note : VM based obfuscation 

**Technical note gathering :** 
- high-level information about VM based obfuscation technique
- technical details and method to tackle it 

## Sources : 
I discovered VM based obfuscation during CTF. This leds me to make research about the subject to understand and learn how to break this type of obfuscation. All the notes presented here are mine but consists in rephrasing of the workshop and work proposed by Tim Blazytko in r2con2021 [workshop video available here](https://www.youtube.com/watch?v=b6udPT79itk) and blog post [here](https://synthesis.to/2021/10/21/vm_based_obfuscation.html). All credits goes to him.   
Another major source is the Miasm documentation : description of Miasm basic concepts come from there : [Documentation](https://github.com/cea-sec/miasm/tree/master/doc)

## Glossary : 
1. VM based obfuscation : principles and components 
2. Method steps to break it 
3. Implementation 
4. Sources

## I - VM based obfuscation : 

**Principle :** produce a binary composed of classical and known ISA instructions (x86_64, ARM etc...) embedding a code which itself recreates the logic of another/unidentified architecture.   

Examples of program generating VM based obfuscation : vmprotect.   

**Two (and a half) VM categories catgeories :** 
- Stack-based VMs 
- Register-based VMs 
- Hybrid 

**VM components :** 
- VM entry : handling the VM setup could be considered as a prologue. 
- Dispatcher or Fetch Decode Execute component : handling the instruction fetching, decoding and driving the execution towards the different handlers. 
        <details>
            <summary>Dispatcher type</summary>
        
            - Nested tree based dispatcher   
             

  
- VM handlers : from a high level pov, a handler is an element implementing one instruction in the VM "ISA". When looking at the code of the handler, it's composed of mutliple instructions from the original ISA.
- VM exit : this is a special handler, representing the end of the VM execution. 
- Bytecode : the VM is executing a bytecode written in its own ISA. The bytecode is located somewhere in the binary. 
- Virtual Instruction Pointer : is generally a register of the original architecture, points to the currently executed instruction in the bytecode. Is handled most of the time by the dispatcher. 


## II - Method to break it : 
Semi-automatic method described here, not up-to-date with AI evolution.  

### Manual recon phase :   
1. Identify the VM components and make a structural recap :
- Locate VM entry 
- Locate bytecode start address 
- Locate and count the VM handlers
- Locate the VM exit   
2. Identify the Virtual Instruction Pointer and Virtual Stack Pointer for relevant category 
3. Locate the VIP and VSP incrementation logic 
4. Understand the handlers calling convention

### Semi-automatic reverse phase :  
Goals phase : being able to generate a trace of the entire VM execution, create a VM disassembler to disassemble whole execution trace (VM code).    
Based on **Symbolic Execution**.   
A technical presentation of symbolic execution is available here. In the scope of this technical presentation, we will only focus on the benefits of using SE in our scenario.   
There are two major steps in this part of the work :  
1. Symbolically execute the entire VM, from VM entry handler to VM exit handler. 
2. Using this symbolic execution, write a VM disassembler.   
This phase will mostly present the logic and technical details about the miasm framework. 

To achieve this, I used the [Miasm framework](https://github.com/cea-sec/miasm) and code proposed by tim in its [Github](https://github.com/mrphrazer/r2con2021_deobfuscation/tree/main).  

### 1. Symbolically execute the VM from entry to exit : 
#### A. Miasm technical setup : 
This part is going to present technical details about the Miasm setup required to achieve our goal. This also presents all the elements I discovered and understood about the Miasm architecture and all the debugging I performed to modify the minimal setup provided by Tim to achieve my goal on the VM I was working on.    

The first central notion in Miasm is the locationDB object. 

**LocationDB :**  
Database responsible to handle the symbols of a binary. A ``location`` is an object representing code or data position. This is the entry point of a project. 

- LocationDB creation and filling : 
```python 
loc_db = LocationDB()
# automatically fill the symbol table using byte input stream
container = Container.from_stream(open(file_path, 'rb'), loc_db)
```

**Disassembly engine :**
The process is now the following : binary code -> Assembly CFG -> IR CFG. Then the IR CFG is passed to the symoblic execution engine. Indeed, the symbolic execution engine needs an IR representation to work. The benefits of the IR representation are : 
- Get a unified representation of the code that doesn't depends on the architecture 
- Get a minimal language 
- The side effects are explicit : when performing ``x + y`` in assembly, the side effects produced by the CPU is that the flags are updated. When working with an IR, this side effects are represented in the IR language itself. This is primordial for the Symbolic Execution engine to do its job.  

- Disassembly engine setup and ASM CFG creation : 

```python
machine = Machine(container.arch)
# initialize the disassembly engine using the binary data and the LocationDB 
mdis = machine.dis_engine(container.bin_stream, loc_db=loc_db)
# We initialize the lifter that will be used later using the disas engine and its LocationDB 
lifter = machine.lifter_model_call(mdis.loc_db)
# produce an assembly CFG, starting the disassembly from the start address
asm_cfg = mdis.dis_multiblock(start_addr)
```

Extract of the ASM CFG : it's composed of AsmBlock objects source available [AsmBlock](https://github.com/cea-sec/miasm/blob/master/miasm/core/asmblock.py#L84). Each AsmBlock has its own location in the LocationDB, has its binary code and finally a set of constraints.   
The constraints are AsmConstraint [objects](https://github.com/cea-sec/miasm/blob/master/miasm/core/asmblock.py#L42) represent the links between basic blocks. There are two types of constraints : ``c_next`` or ``c_to``, the difference between both constraint types is still obscure to me for the moment. 
```                 
loc_8048441                                     <= location 
XOR        EBX, EBX                             <= binary elements
CMP        BYTE PTR [0x8049A90], 0x0
JNZ        loc_804869c
->	c_next:loc_8048450 	c_to:loc_804869c        <= constraints
```

At this stage, we have an ASM CFG of our function implementing the VM in itself. For a minimal function implementing the VM this is sufficient.   

However, in my case, the VM decoder was implemented in another function called by the dispatcher and this was not supported by the code provided in the workshop. 

**Looking for the function call in the ASM CFG :**  
I looked in the ASM CFG to identify the block performing the function call and observed this : 

```
loc_804843b
LODSD      
CALL       loc_80488f0
->	c_next:loc_8048441
```
The constraint to the next location doesn't match the address of the call. I tried to look in the LocationDB if the ``loc_80488f0`` was present using this code : 

```python 
# First we need to retrieve a lockey, this corresponds to an entry in the DB 
decode_loc = loc_db.get_offset_location(0x80488f0)
# Then we can try to retrieve the block corresponding to this lockey
print(asm_cfg.loc_key_to_block(decode_loc))
```
This returns None, indicating that the lockey is present in the LocationDB but not in the ASM CFG. This observation indicates that the disassembler doesn't follow the calls when performing its disassembly phase.   


**Modifying the ASM CFG :**   
My goal was to modify the ASM CFG to get the intended representation of the function call. ASM CFG modifications is greatly supported by Miasm, different functions are available to tweak your CFG as you want. 

```python
# First retrieve lockeys of all the block involved in the call 
loc_call_block = loc_db.get_offset_location(0x804843b)
loc_ret_block = loc_db.get_offset_location(0x8048441)
loc_decode_block = loc_db.get_offset_location(0x80488f0)

# Then add the missing block to the ASM CFG 
asm_cfg.add_block(mdis.dis_block(0x80488f0))

# Just for debugging, it is possible to check if the block has been correctly added 
#decode_block = asm_cfg.loc_key_to_block(decode_loc)

# Then, delete the incorrect edge (constraint)
asm_cfg.del_edge(loc_call_block, loc_ret_block)


# Add two new edges call_block -> decode_block and decode_block -> ret_block
asm_cfg.add_edge(loc_call_block, loc_decode_block, 'c_next')
asm_cfg.add_edge(loc_decode_block, loc_ret_block, 'c_next')

```

ASM CFG after modifications : 

```
loc_804843b
LODSD
CALL       loc_80488f0
->      c_to:loc_80488f0                  <-- Here, the update has been done 

loc_80488f0                               <-- This block is missing in the previous AsmCFG
PUSH       EAX
MOV        BL, AL
AND        BL, 0x7
MOV        BYTE PTR [0x8049A92], BL
MOV        BL, AL
SHR        BL, 0x3
AND        BL, 0x7
MOV        BYTE PTR [0x8049A91], BL
PUSHW      BX
AND        BL, 0x1
MOV        BYTE PTR [0x8049A93], BL
POPW       BX
SHR        BL, 0x1
MOV        BYTE PTR [0x8049A94], BL
SHR        AL, 0x6
MOV        BYTE PTR [0x8049A90], AL
POP        EAX
RET
->      c_to:loc_8048441                   <-- This points to the return block
```

From now on, we have a correct ASM CFG we can move on to the Symbolic Execution. 

**Symoblic execution engine setup :**  
As described before, we have to translate ASM CFG into IR CFG before providing it to the Symbolic Execution engine. 

```python
# translate asm_cfg into ira_cfg
ira_cfg = lifter.new_ircfg_from_asmcfg(asm_cfg)

# init SE engine
sb = SymbolicExecutionEngine(lifter)
```

We now have a working setup and are able to symbolically execute faithfully the VM code. We can now, move on to the algorithm used to perform the symbolic execution. 

#### B. Follow Execution algorithm : 

**Algorithm description :**
The algorithm is the one provided by Tim Blazytko in its workshop available on its [Github](https://github.com/mrphrazer/r2con2021_deobfuscation/tree/main). I didn't perfom any modification on it.   

```python
# init worklist : with the start address (element are Integers) 
basic_block_worklist = [ExprInt(start_addr, 32)]

# worklist algorithm
while basic_block_worklist:
    # get current block
    current_block = basic_block_worklist.pop()

    print(f"current block: {current_block}")

    # symbolical execute block -> next_block: symbolic value/address to execute
    next_block = sb.run_block_at(ira_cfg, current_block, step=False)

    print(f"next block: {next_block}")

    # is next block is integer or label, continue execution
    if next_block.is_int() or next_block.is_loc():
        basic_block_worklist.append(next_block)
```  
The logic is pretty simple, we perform execution block by block using the IR CFG that we created before starting from the entry block. The Symbolic Execution engine could return an integer, this represents the next block to be executed or a location from the LocationDB representing the same thing. In that case, we continue the execution. Otherwise, the execution is stopped and the SE state is dump in the terminal.   
