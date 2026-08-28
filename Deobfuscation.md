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

With a symbolic execution engine correctly initialized and a correct CFG, the symoblic execution can begin.  

#### C. From symbolic execution to concolic execution : 

The method presented here is rather concolic execution that symbolic execution alone. Indeed, we must constraint the symbolic execution engine using real values from the binary to execute correctly the VM from start to exit.   
The first constraint is the VM bytecode. Already identified in the first phase, the initialization is the following one : 

```python 
# Comes from Tim Blazytko source code
def constraint_memory(address, num_of_bytes):
    """
    Reads `num_of_bytes` from the binary at a given address
    and builds symbolic formulas to pre-configure the symbolic
    execution engine for concolinc execution.
    """
    global container
    # read bytes from binary
    byte_stream = container.bin_stream.getbytes(address, num_of_bytes)
    # build symbolic memory address
    # Needed an update and change this 32 bits
    sym_address = ExprMem(ExprInt(address, 32), num_of_bytes * 8)
    # build symbolic memory value
    sym_value = ExprInt(int.from_bytes(
        byte_stream, byteorder='little'), num_of_bytes * 8)

    return sym_address, sym_value

# constraint bytecode using identified memory range 
sym_address, sym_value = constraint_memory(0x8049a9c, 0x8049ebb - 0x8049a9c)
sb.symbols[sym_address] = sym_value

```

Once the bytecode constrained, there certainly are some registers or memory regions that requires initialization, you can proceed the same way.     
Once the first static initialization phase performed, the second phase will start. The major advantage of this method and the algorithm presented previoulsy is that the symbolic execution engine will stop every time it encouters a unresolvable symbol. Every time the value of an element is unknown in its execution and the value is required to go further, the execution is stopped and the internal state of the engine is shown.   
**There are two advantages in this :** 
1. The engine identifies and provides you all the initialization constraints it requires to execute the VM. This facilitates and avoid a heavy RE work. 
2. This transforms the symbolic execution into concolic execution. Each time the engine is stopped, our work is now to identify the constraint to resolve for the engine to go further in the execution. 

Here is one example from a binary. During this phase, it is important to get the maximum information about the SE state as possible. In order to get this, the parameter ``step=True`` is important in the algorithm mentionned above : it will allow us to get the exact detail of the SE state at each executed IR instruction : 
```python
next_block = sb.run_block_at(ira_cfg, current_block, step=True)
```

Here is an example of a SE state dumped during this phase : 

```python 
current block: 0x8048436
________________________________________________________________________________
Instr MOV        ESI, 0x8049A95
Assignblk:
IRDst = loc_key_36
________________________________________________________________________________
ESI                = 0x8049A95
IRDst              = 0x804843B
________________________________________________________________________________
next block: 0x804843B
current block: 0x804843B
Instr LODSD
Assignblk:
EAX = @32[ESI[0:32]]
IRDst = df?(loc_key_83,loc_key_82)
________________________________________________________________________________
ESI                = 0x8049A95
IRDst              = df?(loc_key_83,loc_key_82)
EAX                = {@24[0x8049A95] 0 24, 0x41 24 32}
________________________________________________________________________________
next block: df?(loc_key_83,loc_key_82)                          <=== this requires a constraint
ESI                = 0x8049A95
IRDst              = df?(loc_key_83,loc_key_82)
EAX                = {@24[0x8049A95] 0 24, 0x41 24 32}
```
At every instruction executed, the SE computes the following bloc in the IR CFG representation. In the last one, we can observe that the SE doesn't know where to drive the execution between two different blocks.    
In that case, the df register in x86 assembly requires an initialization to continue the execution, we can initialize using the same method as the one presented with the bytecode above.     

#### D. Debugging the follow_execution script : 
In its release version, the Miasm's SE has a particular implementation of the function call. In my case, the VM dispatcher calls the decoder function and is part of the VM. In order to get the full execution, some modifications in the Miasm source code to modify its way to deal with function calls are required. 
After some debug, I figured out the following setup : 
1. Initialize the stack pointer pointing to the same address as the real one in our VM function, the value could be obtained by debugging the binary. 
2. Add the side effect of the ESP decrementation when performing a call in the Miasm function handling function calls : 
```python 
call_assignblk = AssignBlock(
            [
                ExprAssign(self.sp, ExprOp('-', self.sp, ExprInt(4,32)))        <== added esp operation 
            ],
            instr
        )
```
3. The push and pop procedures were already handled via the modifications I performed on the ASM CFG.    



**Conclusion about SE from VM entry to exit :** the major goal of this part is to build a concolic execution by driving the SE. The major part of the work presented in this part was Miasm technical setup and some ABI implementation support in the SE but still it's interesting elements to understand symbolic execution better and I assume that identical problems would be encountered when performing the same task with other tools. 

### 2. Build our own SE based VM disassembler : 
Using the complete execution gained from the previous phase, our next goal is to build our own SE-based disassembler.   
To do so, the first required step is to identify all the addresses in CFG of the first instruction of each VM handler. With this list, we can now perform small modifications of the algorithm presented above to hook execution of each VM handler at the first instruction : 

```python

def disassemble(sb, address):
    """
    Callback to dump individual VM handler information,
    execution context etc.
    """
    # fetch concrete value of current virtual instruction pointer
    vip = int(sb.symbols[ExprId("ESI", 32)])-4

    # catch the individual handlers and print execution context
    if int(address) == 0x8048462:
        print(f"vip={vip:x} ; hdlr={address} ;  inc word 0x8049a8e")
    elif int(address) == 0x804847a:
        print(f"vip={vip:x} ; hdlr={address} ;  mov word 0x8049a8e")


## Modifications proposed by Tim Blazytko
while basic_block_worklist:
    # get current block
    current_block = basic_block_worklist.pop()

    
    # if current block is a VM handler, we call our disassembler hook 
    if current_block.is_int() and int(current_block) in VM_HANDLERS:
        disassemble(sb, current_block)

    # symbolical execute block -> next_block: symbolic value/address to execute
    next_block = sb.run_block_at(ira_cfg, current_block, step=False)


    # is next block is integer or label, continue execution
    if next_block.is_int() or next_block.is_loc():
        basic_block_worklist.append(next_block)
```
The next step is now to exactly identify the actions performed by each VM handler and traduce this knowledge in the assembly of our choice. Using this, at each executed VM handler, we can have information about the actual handler's goal and rebuild the entire execution and the VM control flow. 

