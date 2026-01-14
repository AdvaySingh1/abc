**Bounty**

In abc, it is difficult to know how the final graph nodes are related to the original nodes.

Sometimes, you can relate certain nodes back using dress but that does not always work.

The idea of this bounty is to keep track of all the nodes that were involved in the creation of a new node.

For example, if two nodes with the same function are merged, you mark the new node as being related to those original nodes. If that new node is again merged with another node, you mark the new node as being related to 3 original nodes. And so forth.

This tracking needs to work for all of the following abc commands:

&get -n
&st
&sweep
&dch
&nf
&st
&syn2
&if -g -K 6
&put
The output should be some sort of map of original nodes to final nodes. Every final node must have at least one source.

**scratch work**

Each of the listed commands operate on the internal `Gia_Man_t_` struct.

Within ABC, node associations are lost in the following senarios:
The `Abc_Ntk_t_` is switched from `ABC_NTK_NETLIST` or `ABC_NTK_LOGIC` to
`ABC_NTK_STRASH`. In this case, even commands on the `ABC_NTK_STRASH` network loose their internal node associations.

This switch occurs from the following functions:
Abc_NtkStrashBlifMv src/base/abc/abcBlifMv.c:416
Abc_NtkCreateTarget src/base/abc/abcNtk.c:1200
Abc_NtkCreateFromGias src/base/abc/abcNtk.c:2550
Abc_NtkFromAigPhase
Abc_NtkAfterTrim
Abc_NtkMiterInt
Io_ReadAiger
Bbl_ManToAig <- important
Res_WndStrash

Normal ABC algorithm (study to see why nodes are not preserved):
Iter:
Abc_CommandRewrite
Abc_CommandBalance
Abc_CommandRefactor

Abc_CommandRewrite
Abc_NtkRewrite
Rwr_NodeRewrite

Abc_NodeGetCutsRecursive
Dec_GraphUpdateNetwork
Rwr_CutEvaluate

In this chain, there is no preserving of the id of the object.
-> `Abc_Obj_t_` needs some sort of provenance

-> either the manager needs to keep the provenance or the node itself.

**&get** command
Manuel:
usage: &get [-cmnvh] <file>
converts the current network into GIA and moves it to the &-space
(if the network is a sequential logic network, normalizes the flops
to have const-0 initial values, equivalent to "undc; st; zero")
-c : toggles allowing simple GIA to be imported [default = no]
-m : toggles preserving the current mapping [default = no]
-n : toggles saving CI/CO names of the AIG [default = no]
-v : toggles additional verbose output [default = no]
-h : print the command usage
<file> : the file name



Great now make me a Project_Plan.md file? In this plan, I want this information.

I also want the other helpful things from notes.md (specifically I just want the location where each of these things are happening (with the cmdAdd), and then I want you to make a timline
Also, I'm thinking about adding a mapping between the current Gia_Man_t Gia_Obj_t (within the Gia_man_t_) and and original node list (with the IDs and names) in the intitial Abc_frame_t. This way, whever I read in the file, I can have the original name and then I can call this & command to print out the Gia_Man_t_ node mappings back to the original nodes. I'm thinking about appending the Vec_Vec_t to the Gia_Man_t in order to keep track of this mapping. Only if the current Gia_Man_t is not closed, is this command valid to execute. What do you think about this? Can you add this to a summary for me?


At the start of it, say modify this function:
Abc_CommandPrintLevel