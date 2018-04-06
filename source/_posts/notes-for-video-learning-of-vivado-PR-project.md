---
title: notes for video learning of vivado PR project
date: 2016-12-21 15:15:54
tags: 
- Vivado 
- Xilinx
- Partial reconfiguration
categories: Vivado development
---

# Vivado Design Suite QuickTake Video Tutorial: Partial Reconfiguration in Vivado 
https://www.xilinx.com/video/hardware/partial-reconfiguration-in-vivado.html

1.Synthesis:
    - Synthesise static logic and reconfigurable modules seperately
    - Use bottom-up or out-of-context synthesis

2.Partial Reconfiguration Control Sequence
- Initiation of reconfiguration implemented by the designer
	- Off-chip microprocessor or other controller,
	- On-chip state machine, processor or other logic
- Activate decoupling logic and reset
	1. Disconnect the reconfigurable region from the static region
	2. Deliver Partial Bitstream
	3. Region is automatically initialized
	4. Release decoupling logic when reconfiguration is complete.

3.Full configurations implemented in-context
	- Static design and reconfigurable modules stored in checkpoints.
<!-- more -->
---

## vivado development flow
Tcl console
详细信息可以参考UG909，Chapter 3: Vivado Software Flow
1. `open_checkpoint ./Sources/synth/static/system_stub.dcp`	(routed checkpoint)
2. `read_xdc ./Sources/xdc/system_stub.xdc`					(xilinx design constraints)
前2步，打开checkpoint,和top-level constriants.这些组成了static design.

3. `set_property HD.RECONFIGURABLE true [get_cells system_i/FILTER_ENGINE]`
在modules中选中需要设置为pr的模块(FILTER_ENGINE)

4. `read_checkpoint -cell system_i/FILTER_ENGINE ./Sources/synth/sobel/sobel_synth.dcp`
读入综合好的模块的checkpoint, for first partial reconfiguration.

5. 选中对应的module(FILTER_ENGINE),在Device视图中，选择Draw pblock.为PR module设置pblock.
注意：与clock boundary垂直对齐，为了应用（reset after configuration特性）。
`set_property RESET_AFTER_RECONFIG true [get_pblocks pblock_FILTER_ENGINE]`

6.  Tools -> Report -> Report DRC -> Partial Reconfiguration.

7. `opt_design`
8. `place_design`

9. 这个时候看Device视图。会看到之前勾的pblock中有白色的boxes。
那表示partition pins的位置，表示the interface between static design and reconfigurable modules.

10. `route_design` 

11. `write_checkpoint ./Checkpoint/sobel_fullroute_config.dcp`
保存为first configuration.

12. `update_design -black_box -cell system_i/FILTER_ENGINE`
做next configuration的时候，只用保持static design, 替换掉reconfigurable modules.
update_design -black_box将清空reconfigurable partiton.
这个时候，很多routes会变成黄色，表示它们only partially routed.

13. `write_checkpoint ./Checkpoint/static_routed.dcp`
14. `lock_design -level routing` 
clearing the partition之后，locked down the  static design,注意现在routes变成了虚线，表明它们是fixed.

14. `read_checkpoint -cell system_i/FILTER_ENGINE ./Sources/synth/sepia/sepia_synth.dcp`
载入一个新的模块的checkpoint.来填充reconfiguration partition.

15. `opt_design`
16. `place_design`
17. `route_design`
18. `write_checkpoint ./Checkpoint/sepia_fullroute_config.dcp`
按照第一个configuration的流程来。

19. `close_project`
20. `pr_verify ./Checkpoint/sepia_fullroute_config.dcp ./Checkpoint/sobel_fullroute_config.dcp -file verify.log`
比较2个routed configuration,看是否兼容。

21. `open_checkpoint ./Checkpoint/sobel_fullroute_config.dcp`
22. `write_bitstream ./Bitstram/sobel.bit`
最后，可以生成bitstream了。