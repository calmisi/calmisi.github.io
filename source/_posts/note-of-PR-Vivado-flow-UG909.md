---
title: note of PR Vivado flow (UG909)
date: 2016-12-07 16:17:22
tags:
- Vivado
- Xilinx
- Partial reconfiguration
categories: Virtex-7 Partial Reconfiguration
---

# 1.Create a Partial Reconfiguration Project
PR项目和标准的项目设计流一样。通过新建工程向导，选择target device,design sources and constraints, and set all the main project details.
新建工程的时候，静态区域（static portion）应该添加source files和constraints。
可以选择RTL design sources加入到每一个Reconfigurable Partition的第一个Reconfigurable Module中，或者什么都不加将其留为black boxes. 

**注意:只能添加一个RM的sources在工程创建的时候。the Partial Reconfiguration wizard将用来添加另外的RMs.**
<!-- more -->

当工程创建好了之后。Tools > Enable Partial Reconfiguration. 一旦被设置了之后，是不能撤销的。
如果看不到这个选项，需要检查一下有没有有效的Partial Reconfiguration license.
{% asset_img 1.png %}

enable了之后，会出现3个PR-specific 菜单项和窗口tabs:
1. A link to the Partial Reconfiguration Wizard in the Flow Navigator
2. A Partition Definitions tab in the Sources window
3. A Configurations tab in the Design Runs window  

--- 

# 2.Define Reconfigurable Partitions 
一旦将工程设为PR工程之后，Reconfigurable Partitions可以在RTL source hierarchy中被定义。
design hierarchy中的instances有如下的限制：
1. Are defined by RTL, DCP or EDIF sources
2. Do not contain embedded IP brought in as XCI sources
3. Do not contain Out-of-Context (OOC) modules in the underlying RTL
4. Do not contain parameters or generics of the port list that are evaluated at synthesis

在想要设为PR的模块上右键，选择`Create Partition Definition`，开始Reconfiguration Partition的创建过程。
如果design sources不存在的话，这个模块就是一个black box.
{% asset_img 2.jpg %}
然后设置Partition Definition和第一个RM(Reconfigurable Module)的名字。
{% asset_img 3.jpg %}

**Note:选中的模块的实例都会变成Reconfigurable Partition.所以要将不同的实例设置不同的模块名字。**

点击了OK之后。在Vivado中相应的模块实例的前面就变成了菱形，表示这是个Reconfigurable Partition.这些Design sources就被移动到了Partition Definitions tab以便单独的管理。
{% asset_img 4.jpg %}

--- 

# 3.Complete the Partial Reconfiguration Project Structure
设置完Reconfigurable Partitions后。可以通过Partial Reconfiguration Wizard来`为不同的Reconfigurable Partitions添加另外的Reconfigurable Modules` , `定义结合了MRs和static design的完整Configruations` , `declaring the set of runs that will be used to implement all the Configurations`

可以在Flow Navigator中或者tools菜单，可以打开Partial Reconfiguration Wizard。
{% asset_img 5.jpg %}

## 3.1 Edit Reconfigurable Modules
如果在创建PD(Partition Definition)的时候，RTL/netlist source已经有了，那么第一列就会列出对应的Reconfigurable Module。
可以点击绿色的加号按钮来创建新的RM。注意选择好正确的Partition Definition。
如果netlist sources被选中了，要勾选`Sources are already synthesized`并且申明netlist中的Top Module.
{% asset_img 6.jpg 'Creating a new Reconfigurable Module' %}
点击Ok后，会出现如下界面，可以编辑该项目的RMs
{% asset_img 7.jpg 'Set of Reconfigurable Modules defined' %}

## 3.2 Edit Configurations
`Each Configuration is a combination of the static logic plus one RM per RP; each Configuration is a full design image.`
每个Configuration都是static logic加上一个RP中的一个RM。都是一个完整的design image.

既可以手动地创建每一个Configuration,但是最简单的方法时让Vivado来自动地生成最小Configurations的集合，可以通过点击下图中`automatically create configurations`的链接。
{% asset_img 8.jpg 'Edit Configurations before creating any Configurations' %}
这将保证所有的RMs至少被包含一次。并且仅当目前还没有定义过configuration时，才能用这个方法。

If one Partition Definition has more RMs than another, a greybox RM will automatically be used for any RP that has all its RMs covered by prior Configurations. These default Configurations can be modified or renamed, and additional Configurations may be created if desired.(不懂这句。。。)

`Tip: Greybox modules不同于black box modules，因为它们不是空的。`
Greybox RMs have tie-off LUTs inserted to complete legal design connectivity in the absence of an RM and they ensure outputs do not float during operation. Vivado creates these by calling update_design -buffer_ports on selected modules.

{% asset_img 9.jpg 'Edit Configurations after automatic generation of Configurations' %}
注意当一个RM应用在不只一个Configuration中时，implementation的结果可能会不同。
因为如果RM最初是在child run中implemented，那么每次都要place和route。
如果RM implementation最初是在parent configuration中完成的，那么implementation的结果将被reuse。

## 3.3 Edit Configuration Runs
当所有的Configuration都定义之后，最后要管理与之关联的Configuration Runs。
同样，Vivado也可以自动创建一些列的Configuration Runs。列表中的第一个Configuration会被定义成母的，剩下的Configurations会被设置为其子configuration。
{% asset_img 10.jpg 'Automatically generated Configuration Runs' %}

这个架构假定第一个configuration是最重要和challenging的。用户可以通过设置`Parent列`的值，来自行地更改父-子关系。
A Parent of a synthesis run (synth_1 here) indicates the Configuration (most notably the static part) will be implemented from the synthesized netlist, and a Parent of an implementation run (impl_1 here) indicates the parent's locked static implementation result will be used as the starting point.

**注意：为了确保在电子元件上的安全工作环境，一个锁定的静态image必须在任何configuration时都能保持是一样的，只有这样bitstream generetion才能创建兼容的full和partial bitstreams。这是通过建立一个parent-child关系来实现的。**

可以通过绿色的+号按钮增加新的Configuration Run.当所有的Configuration Runs都创建完成后，点击Next。然后会列出所有的新的elements.点击Finish才会真正地对工程进行更改。

In the Design Runs window, out-of-context synthesis runs are created for each Reconfigurable Module, and all Configuration Runs are generated. Relationships between parent and child runs are shown by the levels of indentation.
{% asset_img 11.jpg 'Design Runs for synthesis and implementation' %}
In addition, the Configurations tab now shows the composition details of each Configuration available in the project.
{% asset_img 12.jpg 'Configurations available in the project' %}

---

# 4.Adding or Modifying Reconfigurable Modules or Configurations
在源文件窗口的Partition Definitins tab，包含了每一个RM的源文件和constraints.
{% asset_img 13.jpg 'Sources shown in Partition Definitions tab' %}

---
# 5.Implementing the PR Design.
当所有必需的Configuration Runs都定义完了之后，就可以synthesize和implement了。
Partial Reconfiguration design中每一个Reconfigurable Partition都需要一个Pblock.否则会出现如下错误：
`ERROR: [DRC 23-20] Rule violation (HDPR-30) Missing PBLOCK On Reconfigurable Cell`
解决办法在UG909的page58

`Run Implementation` in the Flow Navigator.
{% asset_img 'flow navigator.jpg' 'flow navigator' %}

---
# 6.Generating Bitstreams
当所有期望的Configurations都已经placed和routed后，就可以生成bitstreams了。
和Implementation一下，直接点击Flow Navigator中的Generate Bitstream就好了。

---
# Unsupported Features.
The following features are not currently implemented:
• IP Integrator support is not yet in place. Block Diagrams cannot be included as RMs or within RMs. Modules within Block Diagrams cannot be set as RMs.
• IP is not yet supported within Reconfigurable Modules, at least for management within a single PR project. All IP should be managed in an external Manage IP project, then brought in as RTL or DCP. Alternatively, RMs containing IP can be synthesized external to the PR project and then brought in as an OOC DCP.
° Even though IP can be added to RMs and set to Global synthesis, it cannot be modified or updated. A future Vivado release will support customization of IP from the Partition Definitions window.
• RTL submodules within RMs must not be set to OOC synthesis. Do not nest OOC synthesis runs.
• Reconfigurable Module instances must not have parameters, generics or other variables evaluated at runtime. The port list for the top level of each RM must be fixed and consistent.
• Bitstream generation is not optimized for PR. Bitstreams can be generated from any implemented configuration, but will always create all full and partial bitstreams for the parent run, and partial bitstreams only for child runs. A future enhancement will allow users to pick which bitstreams are created from each configuration.
• The Write Project Tcl feature is not yet supported.
• Simulation is not supported from within the project.

# Known Limitations
• Once Partitions are defined, they cannot be undone. The only way to return to a flat non-PR project is to create a new one.
• Reuse of implemented Reconfigurable Modules from a child run is not supported. Only implementation results from RMs from the parent run can be reused in a child run.
• Child implementation runs cannot be set active. Flow Navigator actions work on just the parent run, or the parent and all child runs, depending on the action.