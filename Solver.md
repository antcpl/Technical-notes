# SMT/SAT and solvers 

I met SMT solvers when performing Reverse Engineering CTFs. These are my notes gathered from many different sources. My goal is to have a basic knowledge and understanding of SMT and SAT solvers to be able to understand the high level concepts and use them with efficiency during CTF tasks or in my work in general. 

## Glossary 
SAT = Boolean Satisfiability Problem   
CNF = Conjunctive Normal Form   
SMT = Satisfiability Modulo Theory    

## High level concepts and definitions : 
> SMT problem is a decision problem for logical formulas with respect to combinations of background theories (Quoted from the official Z3 documentation)

### SAT problem :
A decisionnal problem which, given a Boolean expression, determines if there is a combination of variables that satisfy the formula and makes it true. 

As an **input** the problem takes a Boolean expression with k variables defined and as an **output** returns true or false.   

Compute the truth table of such a formula will takes 2^k computations as variables have two possible values (True or False).   
Using brute force to solve this become impossible with a reasonable number of variables. This is where SAT solvers are useful. 


### SAT solver : 
SAT solver takes as input a Conjunctive Normal Form which is a standardized way to represent Boolean epressions. All Boolean expressions could be translated to CNF according to the [Cook-Levin Theorem](https://en.wikipedia.org/wiki/Cook%E2%80%93Levin_theorem).    
CNF expressions are composed of clauses consisting or terms, ORs and NOTs, all of them are glued together with AND into a full expression. We can memorize it with :
> CNF is AND of ORs  

SAT solvers are backed by SAT algorithms. They outperform bruteforce by trying to smartly find conflicts and reduce research scope. Some algorithm examples : Unit propagation, DPLL CDCL...

### SMT solver : 
SMT solver is composed of three main elements : a translator, a SAT solver and decisional procedures theory. The SMT solver job is to check the satisfiability of a formula : 
> Is formula X satisfiable modulo theory T.   


Different theories can be used depending on the problem definition : arithmetical theories, BitVectors, Arrays etc...   
From a user perspective all of this is hidden behind the SMT solver API. 

#### Solvers limits : 
SMT solvers can solve non-linear polynomial problem constraints.   
- Polynomial problem : expression composed of monomials that are summed or substracted together and monomials have only non-negative integers exponants. Example : ``x^3 + x +2``
- Non-linear problem : if any variables are multiplied together exponants >=2   
For example, this cannot be solved by an SMT solver : ``3**x + 2``


## Z3 : 
Z3 is an SMT solver created in 2013 and maintained by Microsoft. The basic Z3's input format is SMT-LIB2 which is the specification language standard for SMT solvers created in 2003. Z3 has APIs in many languages : Python, C++, Java etc... In the following parts we are going to focus on the Python API and global Z3 concepts.     
Given a **problem** and **constraints** on variables, Z3 tries to satisfy a set of constraints, a solution for the problem is called a **model**.   
A **model** is an interpretation that makes each asserted constraint True.   
Z3 as two different types of variables : **symbolic variables** and **constant variables**. 

### Z3 Python API :
As I only used Z3 in a reverse engineering context, I go through a description of my Z3 learning with this perspective in mind. Thus, the point of view could be really different according to the usage you have of it.   
In my case, for the moment, I only used Z3 to reproduce some crypto/obfuscation algorithm and try to find a key to solve it. In that case, the greatest challenge is to being able to reproduce the program logic with Z3 scripting and with a perfect accuracy. One bad choice in a symbolic variable definition and the output will be different and not the you are waiting for.   

#### API reference : 
- ``Solver()`` : creates a general purpose solver 
- ``solve()`` : function to solve a system of constraints 
- ``add()`` : add a constraint to the solver / this is also called assert a constraint 
- ``check()`` : solve asserted constraints
- ``simplify()`` : formula/expression simplifier function
- ``push()`` or ``pop()`` : allows to add or remove constraint from the constraint stack. (Z3 is based on a stack implementation, this is a subject I will have to cover)
- ``Implies()`` : Boolean logic operator used to create a bi-implication equivalent to == but used on boolean variables 
- ``sort()`` : each Z3 object has this method, it allows to retrieve and print the type of the selected object in the Z3 environment. 

#### Machine arithmetic theory :   
As exposed earlier Z3 and SMT in general use theory to resolve the problem provided. One important theory is the machine arithmetic theory. Machine arithmetic differs from the classic mathematical arithmetic. In that way, Z3 provides an entire theory to replicate at best the machine arithmetic. This is the theory most used when performing reverse engineering as from the assembly/decompiled source code we try to reproduce the actual algorithm execution.   
**Bit-Vector theory:** the goal is to replicate CPU arithmetic over fixed size bit vectors.   
- ``BitVec('x', 16)`` : creates a 16 bits sized symbolic variable named x 
- ``BitVec(10, 32)`` : creates a 32 bits sized constant equals to 10   
- **Signedness :** similarly to real machine implementation, there is no distinction between signed and unsigned bit vectors as they only represent data. The difference is made based on the operator used on the variables.   
``<, >, >=, <=, /, %`` are signed operator version.   
``ULT, ULE, UGT, UGE, UDiv, URem, LShR`` are usigned operator version.

#### Some Z3 concepts and basic usage : 
In this part, I will describe what could actually be in a source code you are trying to represent and what method to use to reproduce it using Z3.   
- **Symbolic variable modification :** symbolic variable modification/reassignation is impossible.   
Original source code : 
```
a = 12 
b = a + 12 
```  
Z3 implementation : 
```
a = Int('a')
b = Int('b')
s.add(b == a + 5)
```   

- **Conditional execution :**  
Original source code : 
```
if (a < 5)
    a = a + 1
```
Z3 implementation : 
```
a = Int('a')
a_1 = Int('a_1')
s.add(a_1 == If(a<5, a + 1, a))
```  

- **Perform signe extension :**
Z3 implementation : 
```
test = BitVec('test', 16)

ext_test = SignExt(16, test)
```  
We now have a 32 bits signe extended test version. 

## Methodology to use z3 in the context of crypto implementation breaking : 
Most CTFs necessitates the usage of z3 to solve some custom crypto implementation. The methodology described here is mine and is the result of many hours of struggling. Even if it has been designed in the CTF context, I assume it is general enough to be used in others. 

### Step 1 : identify the variables of the problem : 
- Algorithm to reproduce 
- Operand types of the algo 
- Operations used in the algo 
- Inputs of the algo : symbolic variables to declare and non-symbolic variables  
- Constraints to use to make the solver find solutions to the problem 

### Step 2 : implement the script : 
Nothing to be said here, the most important part is the next one. 

### Step 3 : debug and techniques : 
- Depending on the algorithm but sometimes, it could be interesting to isolate only one part of the entire execution to be sure of the correctness of the implementation. I mentionned sometimes because in some cases, this could be the opposite, isolate one part of the algo is not a good idea and will lead you into the wall. 
- Double/Triple/Infinity check of the inputs used. It's very important to be sure that the inputs of the z3 script are the same as the original script. It could be interesting to retrieve the inputs directly from the original binary in execution to be entirely sure. 
- Check the correctness of the reproduction implementation of the crypto elements. If the general logic is correct most of the time, in some cases, the programming language used could do some tricks on your operands you don't want to. Reimplementing the same crypto in a standalone script aside z3 and compare its output to the original one could be path to follow. 
- Always double check the signedness of the operands and operators and use the correct operators in z3. Sometimes original program performs casts during the crypto process ex : extend 32 bits value to 64 bits. This should absolutely be reproduced in the z3 script. 
- Do not hesitate to multiply the same task using different techniques to be sure of the result. For example, to get the inputs needed you may have to extract them from the original binary directly on the fly into the solver script but you can also extract them using a tool and place them into a file. Then check both results, they should match. 


## Interesting usages of Z3 : 

[Breaking petya crypto using Z3](https://0xec.blogspot.com/2016/04/reversing-petya-ransomware-with.html)

## References : 

[Présentation du SMT solver Z3 Nicolas Lhost](https://lim.univ-reunion.fr/staff/fred/Recherche/GT/docs/Z3.pdf)  
[SMT/SAT by example by Denis Yuritchev](https://smt.st/SAT_SMT_by_example.pdf)
