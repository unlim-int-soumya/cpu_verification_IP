# Verification IP for LC-4 Processor

### Current Testbench Architecture

![Testbench Architecture](https://github.com/unlim-int-soumya/lc4-vip/blob/main/lc4_pipeline%20v1/docs/Testbench%20Architecture.png)

## Introduction
This project was part of my independent study for Masters credit in ESE 5990, where efforts were made to develop verification IPs for verifying advanced LC4 processors. The core objective of the project was to develop a scalable and reusable Verification IP tailored for a pipelined superscalar LC4 Processor. During spring 2023, I took a course called Computer Organization and Design (CIS 5710), where I designed a pipelined 2-way superscalar processor using Verilog. In that RTL, it was verified using a simple testbench and trace files provided by the instructor. However, to exert more control over instruction generation, make the verification process more insightful, and enhance flexibility, there was a need to develop the test environments UVCs using SystemVerilog-UVM.

The main objective here was to brush up SystemVerilog, UVM Methodology, CPU Verification while creating a Verification IP for Pipelined Superscalar LC4 Processor that can be easily adapted for other RISC Processors with some modifications in Assembly Code. The challenges faced during the CIS 5710 course, including the absence of trace files for certain advanced features and the difficulty in locating errors within thousands of test cases, highlight the need for a robust and scalable verification solution. This study/project aims to address these challenges by creating a comprehensive testbench using SystemVerilog and UVM, ensuring thorough validation of critical aspects such as instruction execution, pipeline stages, hazard detection, and superscalar implementation by building Universal Verification Components (UVCs) such as Driver, Monitor,Checkers, Scoreboard, and so on, and analyzing Debugging Timing Diagrams.

## Tools used
In this project, I used Verilog HDL for writing the RTL code. For developing the entire test environment, I utilized SystemVerilog in conjunction with the UVM methodology. Additionally, I employed the Python language for automating the execution flow and scripting purposes. For simulation, I relied on Mentor Questa's QuestaSim tool.

## Process
Here, I designed verification IPs for two distinct designs: one for the Pipelined LC4 processor, capable of verifying the Single Cycle design as well, and another for the Superscalar design, capable of verifying the 2-way superscalar implementation following the Pipelined implementation.

### Testbench module (`testbench.sv`)
This is a top level module that instantiates the DUT, clock generator, and lc4_memory as well as contains the SystemVerilog-UVM based testbench environment. It also instantiates the assertion module, interface and generates the clock and reset signals that to be further supplied into the interface definition. The tb_top module serves as the top-level testbench for the LC4 processor design verification. It encompasses the test environment, clock generation, reset generation, interface instantiation, DUT instantiation, and test case execution.

**Module Components**
 - Clock and Reset Signals - The clk and rst signals are globally used within the testbench to drive the simulation.
 - Modules
  - `lc4_we_gen` - Module responsible for generating write enable signals.
  - `lc4_memory` - Module for interfacing with the memory unit of the LC4 processor.
 - Interface - The lc4_interface module is instantiated to provide the physical interface between the testbench and the DUT.
 - Testbench Environment - The testbench environment is set up using UVM methodology, incorporating transaction generation, configuration, and test execution.
 - Test Case Initialization - The testbench initializes the test case based on the provided command-line argument or defaults to a predefined test case if none is specified.

The testbench module takes the test name as command line argument and runs the test using the following method:

```
   initial begin
    // Retrieve the value of TEST_CASE command-line argument
    if ($value$plusargs("TEST_CASE=%s", test_case_name)) begin
      $display("Running test case: %s", test_case_name);
      run_test(test_case_name);
    end else begin
      $display("No test case name provided. Running default test.");
      run_test("lc4_5_test"); // You can set a default test case name here
    end
   end
```  

The complete implementation of this module can be found here.

### Sequence Item (`lc4_tx.sv`)
To initiate the construction of the environments, I initially defined two transaction packets: `lc4_tx` and `output_tx`. The `lc4_tx` packet was responsible for generating input transactions, encompassing `i_cur_insn` and `i_cur_dmem_data`. On the other hand, `output_tx` comprised all input signals (`i_cur_insn` and `i_cur_dmem_data`) along with output signals, including those utilized in the testbench for tracing signal values across various stages. Throughout the verification project, I continuously expanded the definition of trace signals based on my evolving requirements. Although I didn't specify any particular constraints within my sequence item class, I typically defined inline constraints while generating instructions in my sequence class.

The complete implementation of this class can be found here.

### Sequence Class (`lc4_sequence.sv`)
This class is an extension of `uvm_sequence` base class where I defined all type of generated sequence of instructions used in this project. 

```
class base_seq extends uvm_sequence #(lc4_tx);

  `uvm_object_utils(base_seq)

  InstructionDecoder decode;

  function new(string name = "base_seq");
    super.new(name);
  endfunction

  virtual task pre_body();
    uvm_phase phase;
    `ifdef UVM_VERSION_1_2
      // in UVM1.2, get starting phase from method
      phase = get_starting_phase();
    `else
      phase = starting_phase;
    `endif
    
    if (phase != null) begin
      phase.raise_objection(this, get_type_name());
      `uvm_info(get_type_name(), "raise objection", UVM_MEDIUM)
    end
  endtask : pre_body

  virtual task post_body();
    uvm_phase phase;
    `ifdef UVM_VERSION_1_2
      // in UVM1.2, get starting phase from method
      phase = get_starting_phase();
    `else
      phase = starting_phase;
    `endif

    if (phase != null) begin
      phase.drop_objection(this, get_type_name());
      `uvm_info(get_type_name(), "drop objection", UVM_MEDIUM)
    end
  endtask : post_body

endclass
```
The base_seq class serves as the foundation for generating instruction sequences. It inherits from the `uvm_sequence` class and includes an InstructionDecoder instance.

A set of virtual tasks, `pre_body()` and `post_body()`, handle pre- and post-sequence execution phases, respectively.

Below is an example of a child class inheriting from `base_seq` to generate a sequence of `NOP` instructions following a load instruction.

```
class load_nop_seq extends base_seq;

  `uvm_object_utils(load_nop_seq)

  function new(string name = "load_nop_seq");
    super.new(name);
  endfunction

  virtual task body();
    `uvm_info(get_type_name, "load_nop_seq", UVM_LOW)

    repeat(10) begin
      `uvm_do_with(req, {req.i_cur_insn[15:12] == 4'b0110;}); // any constraint
      `uvm_do_with(req, {req.i_cur_insn == 0;}); // any constraint
    end
  endtask

endclass: load_nop_seq

```
In this child class, named `load_nop_seq`, `NOP` instructions are generated after a load instruction. The `body()` task executes this sequence, creating 10 load followed by NOP instructions with specific constraints applied. Here, by using inline constraints, I was able to generate the types of instruction sequences I want.

The project utilizes various such child classes to define different sets of instruction sequences, enabling comprehensive testing of the design.

The complete implementation of this class can be found here.

### Test Class (`lc4_test_lib.sv`)
In the `base_test` class, I created a foundational test structure that serves as the parent class for all subsequent test libraries in the project. This class facilitates the creation of a hierarchical structure for all components and prints the topology during the `end_of_elaboration_phase()`.

Below is the implementation of `base_test` that was created.
```
class base_test extends uvm_test;

`uvm_component_utils(base_test)

lc4_env lc4;


function new(string name, uvm_component parent = null);
  super.new(name, parent);
endfunction

function void build_phase(uvm_phase phase);
  super.build_phase(phase);
  `uvm_info(get_type_name(),"Build Phase Executed", UVM_LOW)

  lc4 = lc4_env::type_id::create("lc4", this);

endfunction
  
function void end_of_elaboration_phase(uvm_phase phase);
  uvm_top.print_topology();
endfunction

endclass

```
**Class Components**
 - Extension of uvm_test - Inherits from the uvm_test class, which is part of the Universal Verification Methodology (UVM) library.
 - Component Utilities - Utilizes the uvm_component_utils macro to enable utilities such as factory registration and component hierarchy manipulation.
 - Environment Instances - Declares instances of the LC4 environment (lc4_env)
 - Constructor - Defines a constructor to initialize the test name and parent component.
 - Build Phase - During the build phase, creates an instance of the LC4 environment (lc4_env) and retrieves the virtual interface (vif) from the UVM configuration database.
 - End of Elaboration Phase - Prints the topology of the UVM components at the end of the elaboration phase using uvm_top.print_topology().

I further defined child classes for all the unique sequences that I had created in my sequence.sv file. I was able to define the verbosity and run the m_sequencer using the method outlined below.

```
class load_nop_test extends base_test;

`uvm_component_utils(load_nop_test)

function new(string name = "load_nop_test", uvm_component parent);
  super.new(name, parent);
endfunction

virtual task run_phase(uvm_phase phase);
  super.run_phase(phase);

  lc4.agent.driver.set_report_verbosity_level(UVM_HIGH);
  lc4.agent.monitor.set_report_verbosity_level(UVM_HIGH);
  lc4.scoreboard.set_report_verbosity_level(UVM_LOW);

  uvm_config_wrapper::set(this, "lc4.agent.sequencer.run_phase", "default_sequence", load_nop_seq::get_type());

endtask

endclass

```

This `load_nop_test` class is a child class of `base_test`, inheriting its functionalities. In the `run_phase` task, it first calls the run_phase task of its parent class (`base_test`). Then, it sets the verbosity levels for the driver, monitor, and scoreboard components of the `lc4 agent`. Finally, it configures the default sequence to be executed by the sequencer to `load_nop_seq`.

The complete implementation of this class can be found here.

### Interface (`lc4_interface.sv`)
Using the `lc4_interface`, communication between the testbench environment and the design modules is facilitated. It defines all signals utilized by the DUT. Then, it proceeds to define two clocking blocks: `drv_cb` and `mon_cb`. Following this, two modports are defined to specify the direction of signals for these clocking blocks.

Below is the implementation of `lc4_interface` that was created.

```
`timescale 1ns / 1ps

interface lc4_interface(input logic clk, input rst);

    logic gwe;             // global we for single-step clock
    logic [15:0] o_cur_pc;        // address to read from instruction memory
    logic [15:0]  i_cur_insn;    // output of instruction memory (pipe A)

    logic [15:0] o_dmem_addr;     // address to read/write from/to data memory
    logic [15:0]  i_cur_dmem_data; // contents of o_dmem_addr
    logic        o_dmem_we;       // data memory write enable
    logic [15:0] o_dmem_towrite;  // data to write to o_dmem_addr if we is set

    // other signals

    clocking drv_cb @ (posedge clk);

      default input #0 output #0;
      //default input #1step output #0;

      input gwe;             // global we for single-step clock

      input o_cur_pc;        // address to read from instruction memory
      output i_cur_insn;    // output of instruction memory (pipe A)

      input o_dmem_addr;     // address to read/write from/to data memory
      output i_cur_dmem_data; // contents of o_dmem_addr
      input o_dmem_we;       // data memory write enable
      input o_dmem_towrite;  // data to write to o_dmem_addr if we is set

      // other signals


  endclocking

  clocking mon_cb @ (posedge clk);
  
    //default input #0 output #1;
    
    default input #0;
          
    input gwe;             // global we for single-step clock
    input o_cur_pc;        // address to read from instruction memory
    input i_cur_insn;    // output of instruction memory (pipe A)

    input o_dmem_addr;     // address to read/write from/to data memory
    input i_cur_dmem_data; // contents of o_dmem_addr
    input o_dmem_we;       // data memory write enable
    input o_dmem_towrite;  // data to write to o_dmem_addr if we is set

    // other signals

 
  endclocking


  modport DRIVER (clocking drv_cb, input clk, rst);

  modport MONITOR (clocking mon_cb, input clk, rst);
   
endinterface

```
- The interface `lc4_interface` has input ports for the clock (`clk`) and reset (`rst`).
- It defines several other signals used for communication between the testbench and the DUT, such as `gwe` (global write enable), `o_cur_pc` (address to read from instruction memory), `i_cur_insn` (output of instruction memory), `o_dmem_addr` (address to read/write from/to data memory), `i_cur_dmem_data` (contents of o_dmem_addr), `o_dmem_we` (data memory write enable), and `o_dmem_towrite` (data to write to o_dmem_addr if write enable is set).
- Two clocking blocks are defined within the interface: `drv_cb` and `mon_cb`. These clocking blocks are used to specify the timing and direction of signals during simulation.
- The `drv_cb` clocking block is intended for driving signals from the testbench to the DUT. It includes inputs for all signals defined in the interface.
- The `mon_cb` clocking block is intended for monitoring signals from the DUT in the testbench environment. It also includes inputs for all signals defined in the interface.
- Two modports are defined: `DRIVER` and `MONITOR`. These modports specify the clocking block to be used (`drv_cb` for `DRIVER` and `mon_cb` for `MONITOR`) and include the clock (`clk`) and reset (`rst`) signals as well.

The complete implementation of this interface can be found here.

### Sequencer (`lc4_sequencer.sv`)
I implemented a simple sequencer class, which was a child class of the UVM component base class uvm_sequencer.

```
class lc4_sequencer extends uvm_sequencer #(lc4_tx);

`uvm_component_utils(lc4_sequencer);

function new(string name, uvm_component parent);
  super.new(name, parent);
endfunction

endclass

```

### Driver (`lc4_driver.sv`)
The lc4_driver was responsible for collecting the sequences generated by the sequence class from the seq_item_port and driving them to the interface and then to the DUT.

```

`define DRIV_IF vif.DRIVER.drv_cb


class lc4_driver extends uvm_driver #(lc4_tx);

`uvm_component_utils(lc4_driver)

function new(string name, uvm_component parent);
  super.new(name, parent);
endfunction

virtual lc4_interface vif;

//task declaration - extern indicates out-of-body declaration
extern virtual task drive_data(lc4_tx tx);  

//task declaration - extern indicates out-of-body declaration
extern virtual function void report_phase(uvm_phase phase);  

// Declare this property to count packets sent
int num_sent;

function void build_phase(uvm_phase phase);
  super.build_phase(phase);
  `uvm_info(get_type_name(),"Build Phase Executed", UVM_LOW)

  if(!uvm_config_db #(virtual lc4_interface)::get(this,"", "pif", vif)) begin
    `uvm_error("NOVIF", "vif not set")
  end

endfunction

task run_phase(uvm_phase phase); 
  `uvm_info(get_type_name(),"Run Phase Executed", UVM_LOW)

  forever begin
    seq_item_port.get_next_item(req);
    `uvm_info(get_type_name(), $sformatf("Sending Packet :\n%s", req.sprint()), UVM_HIGH)
    drive_data(req);
    seq_item_port.item_done();
  end

endtask


endclass

task lc4_driver::drive_data(lc4_tx tx);
 

  //@(posedge`DRIV_IF.gwe);
  @(posedge vif.DRIVER.clk);

  //wait(`DRIV_IF.gwe);

  `DRIV_IF.i_cur_insn <= tx.i_cur_insn;

  `DRIV_IF.i_cur_dmem_data <= tx.data;

  `uvm_info(get_type_name(),"Packet Sent to DUT!", UVM_LOW)  
  num_sent++;
  
endtask


// UVM report_phase
function void lc4_driver::report_phase(uvm_phase phase);
  `uvm_info(get_type_name(), $sformatf("Report: LC4 Driver Sent %0d Packets", num_sent), UVM_LOW)
endfunction : report_phase

```
The `lc4_driver` class serves as the component responsible for driving data from the testbench to the DUT (Design Under Test). Here's an overview of its functionality:

- Class extension, Initialization, and build phase:
  - It inherits from the uvm_driver class and is defined to work with transactions of type lc4_tx.
  - During the build phase, it retrieves the virtual interface (vif) from the configuration database.
- `run_phase()` Task Execution:
  - In the `run_phase` task, it continuously waits for transactions from the sequence item port (`seq_item_port`).
  - Upon receiving a transaction, it calls the `drive_data` task to drive the data onto the DUT's interface.
  - Once the transaction is processed, it notifies the sequence item port that the item is done.
- Driving Data (drive_data task):
  - This task is responsible for driving the data contained in the input transaction (`tx`) onto the DUT's interface.
  - It waits for a positive edge of the DUT's clock (`vif.DRIVER.clk`) before driving the data.
  - It assigns the instruction (`i_cur_insn`) and data (`i_cur_dmem_data`) fields of the transaction to the corresponding signals in the DUT's interface.
  - It increments the num_sent variable to keep track of the number of packets sent.
- Reporting:
  - The `report_phase` function provides a summary report of the number of packets sent by the driver component.
  - This report is generated at the end of the run phase.

### Monitor (`lc4_monitor.sv`)
The lc4_monitor class is responsible for monitoring signals from the DUT interface and collecting data packets for analysis. Here, I had to face some sampling issues as I wasn't getting all the packets collected before. However, I resolved those issues. It uses the `MONI_IF` interface to access the signals from the DUT.

```
`define MONI_IF vif.MONITOR.mon_cb


class lc4_monitor extends uvm_monitor;

`uvm_component_utils(lc4_monitor)

  
uvm_analysis_port #(out_tx) item_collect_port;

virtual lc4_interface vif;

out_tx pkt;

// Declare this property to count packets sent
int num_collected;


//task declaration - extern indicates out-of-body declaration
extern virtual task run_phase(uvm_phase phase);  
extern virtual task collect_data(out_tx tx);  

//task declaration - extern indicates out-of-body declaration
extern virtual function void report_phase(uvm_phase phase);  

function new(string name, uvm_component parent);
  super.new(name, parent);

  item_collect_port = new("item_collect_port", this);  

endfunction

function void build_phase(uvm_phase phase);
  super.build_phase(phase);
  `uvm_info(get_type_name(),"Build Phase Executed", UVM_LOW)
  if(!uvm_config_db #(virtual lc4_interface)::get(this,"", "pif", vif)) begin
    `uvm_error("NOVIF", "vif not set")
  end
endfunction
endclass

task lc4_monitor::run_phase(uvm_phase phase); 
  `uvm_info(get_type_name(),"Run Phase Executed", UVM_LOW)  
  super.run_phase(phase);

  forever begin
    pkt = out_tx::type_id::create("pkt", this);
    collect_data(pkt);
    item_collect_port.write(pkt);
    //`uvm_info(get_type_name(), $sformatf("Packets Collected! :\n%s", pkt.sprint()), UVM_MEDIUM)   
    num_collected++;
  end

endtask

task lc4_monitor::collect_data(out_tx tx);

  `uvm_info(get_type_name(),"Inside collect_data()", UVM_LOW)

  // waiting for posedge of gwe
  @(posedge `MONI_IF.gwe);

  tx.i_cur_insn_A = `MONI_IF.i_cur_insn_A;
  tx.i_cur_insn_B = `MONI_IF.i_cur_insn_B;
  tx.data = `MONI_IF.i_cur_dmem_data;

  // i_cur_insn gets enabled during gwe
  // one cycle after rising edge of gwe collect output
  @(posedge vif.MONITOR.clk);
  #3;
  
  tx.gwe = `MONI_IF.gwe; 
  tx.o_cur_pc = `MONI_IF.o_cur_pc; 
  
  tx.o_dmem_we = `MONI_IF.o_dmem_we; 
  
  tx.o_dmem_addr = `MONI_IF.o_dmem_addr; 
  tx.o_dmem_towrite = `MONI_IF.o_dmem_towrite; 
  tx.test_stall_A = `MONI_IF.test_stall_A; 
  tx.test_stall_B = `MONI_IF.test_stall_B; 

  // collecting other signals

  `uvm_info(get_type_name(), $sformatf("Packets Collected! :\n%s", tx.sprint()), UVM_MEDIUM)   
    
endtask

// UVM report_phase
function void lc4_monitor::report_phase(uvm_phase phase);
  `uvm_info(get_type_name(), $sformatf("Report: LC4 Monitor Collected %0d Packets", num_collected), UVM_MEDIUM)
endfunction : report_phase


```

- run_phase() Task execution
  - During the `run_phase`, the monitor continuously collects data packets by calling the collect_data task. Each collected packet is then written to the `item_collect_port` for further analysis. The `num_collected` variable keeps track of the number of packets collected.
- collect_data() task Execution
  - The `collect_data` task samples various signals from the DUT interface (`MONI_IF`) and populates the `out_tx` packet accordingly. These signals include instruction data, memory addresses, control signals, and other relevant information.
- The report_phase function reports the number of packets collected during the simulation.

### Agent (`lc4_agent.sv`)

The `lc4_agent` class, inherited from the `uvm_agent` base class, primarily serves as a container for `lc4_sequencer`, `lc4_driver`, and `lc4_monitor`. Therefore, it functions as an active agent within the testbench environment.

```
class lc4_agent extends uvm_agent;

`uvm_component_utils(lc4_agent)

lc4_driver driver;
lc4_monitor monitor;
lc4_sequencer sequencer;

function new(string name, uvm_component parent);
  super.new(name, parent);
endfunction

function void build_phase(uvm_phase phase);
  super.build_phase(phase);
  `uvm_info(get_type_name(),"Build Phase Executed", UVM_LOW)

  driver = lc4_driver::type_id::create("driver", this);
  monitor = lc4_monitor::type_id::create("monitor",this);
  sequencer = lc4_sequencer::type_id::create("sequencer",this);

endfunction

function void connect_phase(uvm_phase phase);
  super.connect_phase(phase);

  //sequencer.seq_item_export.connect(driver.seq_item_port);
  driver.seq_item_port.connect(sequencer.seq_item_export);
endfunction

endclass

```

The `lc4_agent` class primarily serves two main functions: creating all components within it and establishing connections between the driver's `seq_item_port` and the sequencer's `seq_item_export`.

### Scoreboard (`lc4_scoreboard.sv`)

The `lc4_scoreboard` class extends the `uvm_scoreboard` base class and serves as a crucial component in the verification environment for an LC4 processor design. It manages the verification of instructions and the correctness of the processor's pipelining behavior.

The scoreboard contains various components and data structures necessary for its operation. These include the `out_tx` data structure, which represents transaction objects exchanged between the DUT and the scoreboard. Additionally, it utilizes queues (`item_q` and `nzp_q`) to store and manage transaction objects during verification. Here, item_q was used to store the previous instruction that was in the pipeline.

The class features an analysis port (analysis_imp_a) for receiving transaction objects from the driver or other components in the verification environment. Upon receiving a transaction, the scoreboard performs a series of checks and validations to ensure the correctness of the DUT's behavior.

```
`uvm_analysis_imp_decl(_port_a)
```

```
  function new(string name, uvm_component parent);
    super.new(name, parent);
    analysis_imp_a = new("analysis_imp_a", this);    
  endfunction : new
```

```
virtual function void write_port_a(out_tx tx);
    `uvm_info(get_type_name(),$sformatf(" Inside write_port_a method. Recived packet On Analysis Imp Port"),UVM_HIGH)
    `uvm_info(get_type_name(),$sformatf(" Printing packet, \n %s",tx.sprint()),UVM_HIGH)
    // after receiving transaction as `tx` implemented scoreboarding logic here 
```


Key functions within the scoreboard handle specific verification tasks. For example, check_test_dmem_addr validates the memory address generated by the DUT, while check_test_regfile_data verifies the data written to the register file. Other functions like check_pipeline_logic ensure the correct behavior of the pipeline stages by comparing consecutive transactions.

By the help of reference model and using various methods defined inside the scoreboard class, I was able to verify various features.

```
    // verifies the data to be written into data memory on memory stage
    check_o_dmem_towrite(tx.o_dmem_towrite, tx.test_mem_insn, ref_m.WM_bypass, ref_m.wdata_regfile_w, tx.test_rt_data_mm);

    // verifies the `test_dmem_addr` test bench signal
    check_test_dmem_addr(tx.test_dmem_addr, tx.test_alu_out_w, tx.test_cur_insn);

    // verifies the data to be written into `reg_file`
    check_test_regfile_data(tx.test_regfile_data, ref_m.wdata_regfile_w);

    // verifies `nzp_bits` value registers
    check_test_nzp_new_bits(tx.test_nzp_new_bits, ref_m.wdata_regfile_w);  

    // verifies `nzp_we` testbench signal
    check_test_nzp_we(tx.test_nzp_we, DEC_WB.nzp_we);

    // verifies if a branch instruction is taken or not taken
    check_taken_not_taken(tx.test_brx_taken_x, ref_m.brx_taken_x);

    // checks if flush signal activates at the correct scenarios 
    check_flush_need(tx.test_brx_taken_x, DEC_EX.is_control_insn, ref_flush_need);
```

```

function void lc4_scoreboard::check_flush_need(logic test_brx_taken_x, logic is_ctrl_insn, logic ref_flush_need);

  if((test_brx_taken_x || is_ctrl_insn) !=  ref_flush_need) begin
    `uvm_error(get_type_name(), $sformatf(
    "\nFAILED: <FLUSH_NEED>! actual_flush_need=%0b ref_flush_need=%0b\n\n", (test_brx_taken_x || is_ctrl_insn), ref_flush_need))
  end else begin
    `uvm_info(get_type_name(),$sformatf(
    "\nPASSED: <FLUSH_NEED>! actual_flush_need=%0b ref_flush_need=%0b\n\n", (test_brx_taken_x || is_ctrl_insn), ref_flush_need), UVM_HIGH)
  end

endfunction

function void lc4_scoreboard::check_nzp_bits(logic [2:0] actual_nzp_bits, logic [2:0] ref_nzp_bits);

  if(actual_nzp_bits !=  ref_nzp_bits) begin
    `uvm_error(get_type_name(), $sformatf(
    "\nFAILED: <NZP_BITS>! actual_nzp_bits=%0h ref_nzp_bits=%0h\n\n", actual_nzp_bits, ref_nzp_bits))
  end else begin
    `uvm_info(get_type_name(),$sformatf(
    "\nPASSED: <NZP_BITS>! actual_nzp_bits=%0h ref_nzp_bits=%0h\n\n", actual_nzp_bits, ref_nzp_bits), UVM_HIGH)
  end

endfunction

function void lc4_scoreboard::check_taken_not_taken(logic [15:0] actual_brx_taken_x, logic [15:0] ref_brx_taken_x);

  if(actual_brx_taken_x !=  ref_brx_taken_x) begin
    `uvm_error(get_type_name(), $sformatf(
    "\nFAILED: <BRX_TAKEN_X>! actual_brx_taken_x=%0h ref_brx_taken_x=%0h\n\n", actual_brx_taken_x, ref_brx_taken_x))
  end else begin
    `uvm_info(get_type_name(),$sformatf(
    "\nPASSED: <BRX_TAKEN_X>! actual_brx_taken_x=%0h brx_taken_x=%0h\n\n", actual_brx_taken_x, ref_brx_taken_x), UVM_HIGH)
  end

endfunction

// other methods also defined in similar way

```

During the run phase, the scoreboard continuously processes incoming transactions, performs the necessary checks, and provides informative messages about the verification results.

Overall, the lc4_scoreboard class plays a crucial role in the verification process, ensuring the correctness and functionality of the LC4 processor design.

### Reference Model (`ref_model.sv`)
As mentioned earlier, the DUT outputs testbench signals from various stages, which are collected via the interface using the monitor and forwarded to the scoreboard for computation. The `ref_model` class played a significant role in verifying the features using the `lc4_scoreboard`. It essentially computed all desired output signals based on the input instructions sent and the previous instructions in the pipeline. Several logics needed verification, such as determining the next program counter value, checking if there's a need to stall the circuit, managing instruction pipelining in stages, and assessing whether a branch instruction is taken or not.

Maintaining such structure was possible by employing this approach. I created separate methods inside `ref_model` to verify unique features and defined distinct methods for comparing these computed data with actual data. By calling each unique method and passing the actual value and reference value, the comparison took place, and in case of any error, it was reported.

```
class ref_model;

  // Method computing the reference outcomes for values to be written into nzp_bit registers
  function void nzp_bypassing(
    
    input
    bit [15:0] test_ex_insn,
    bit [15:0] test_mem_insn,
    bit [15:0] test_cur_insn,

    output
    bit nzp_mx_en,
    bit nzp_wx_en
    );

    bit is_nzp_consumer_x;
    bit is_nzp_producer_w;
    bit is_nzp_producer_mm;


    if((test_ex_insn[15:12] == 4'b0000) && ((test_ex_insn[11:9] == 3'b001) || (test_ex_insn[11:9] == 3'b010) || (test_ex_insn[11:9] == 3'b011) || 
      (test_ex_insn[11:9] == 3'b100) || (test_ex_insn[11:9] == 3'b101) || (test_ex_insn[11:9] == 3'b110) || (test_ex_insn[11:9] == 3'b111))) begin
      is_nzp_consumer_x = 1'b1;
    end 
    else begin
      is_nzp_consumer_x = 1'b0;
    end

    if((test_mem_insn[15:12] == 4'b0001) || (test_mem_insn[15:12] == 4'b0010) || (test_mem_insn[15:12] == 4'b0101) || 
      (test_mem_insn[15:12] == 4'b0110) || (test_mem_insn[15:12] == 4'b1001) || (test_mem_insn[15:12] == 4'b1010) || (test_mem_insn[15:12] == 4'b1101) || (test_mem_insn[15:12] == 4'b0100) || (test_mem_insn[15:12] == 4'b1100) || (test_mem_insn[15:12] == 4'b1111)) begin
      is_nzp_producer_mm = 1'b1;
    end else begin
      is_nzp_producer_mm = 1'b0;
    end

    if((test_cur_insn[15:12] == 4'b0001) || (test_cur_insn[15:12] == 4'b0010) || (test_cur_insn[15:12] == 4'b0101) || 
      (test_cur_insn[15:12] == 4'b0110) || (test_cur_insn[15:12] == 4'b1001) || (test_cur_insn[15:12] == 4'b1010) || (test_cur_insn[15:12] == 4'b1101) || (test_mem_insn[15:12] == 4'b0100) || (test_mem_insn[15:12] == 4'b1100) || (test_mem_insn[15:12] == 4'b1111)) begin
      is_nzp_producer_w = 1'b1;
    end else begin
      is_nzp_producer_w = 1'b0;
    end

    nzp_mx_en = is_nzp_producer_mm & is_nzp_consumer_x;

    nzp_wx_en = is_nzp_producer_w & is_nzp_consumer_x;

  endfunction


  // Method computing the reference outcomes for values to be written into nzp_bit registers
  function void nzp_calculation(
    bit [15:0] wdata_regfile_w,
    
    bit nzp_we_w,

    output
    bit [2:0] nzp_bits
    );

    bit [2:0] nzp_in_w;

    // I am already getting wdata_regfile_w in terms of test_regfile_data 
    // need to find it's nzp bits after evaluating the nzp_we_w signal
    // i get nzp_we_we -> test_nzp_we

    // by using these two find nzp bits
    
    if(wdata_regfile_w[15] == 1'b1) begin
      nzp_in_w = 3'b100;
    end else if(wdata_regfile_w == 0) begin
      nzp_in_w = 3'b010;
      end else begin
      nzp_in_w = 3'b001;
    end

    if(nzp_we_w) begin
      nzp_bits = nzp_in_w;
    end else begin
      nzp_bits = nzp_bits;
    end

    //$display("\n\n\ntime=%0t wdata_regfile_w=%0h nzp_we_w=%0b nzp_bits=%0h\n\n\n", $time, wdata_regfile_w, nzp_we_w, nzp_bits);

  endfunction

  // Method computing the reference outcome for flushing feature
  function bit is_flush_need(
      bit brx_taken_x,
      bit is_ctrl_insn_x
    );
    return (brx_taken_x || is_ctrl_insn_x);
  endfunction

endclass

// similarly other methods

```

*Scalability for future features implementations*
1. Checking if there's a need for outputting more testbench signals from the DUT. If so, then trace those signals and collect packets.
2. Defining a method inside the reference model class to find the desired outcomes and calling it on the scoreboard.
3. Defining a unique method that compares the actual outcome with the desired outcome.

### Coverage (`lc4_cov.sv`)

### Instruction Checker (`insn_validator.sv`)
This is a checker class which is an extension of `uvm_component` base class that I created, which performs operations separately to validate whether an instruction is valid or not. Although this could also be implemented within the `lc4_scoreboard`, I opted to create a separate checker file to handle instructions exclusively. If an invalid instruction is received from the DUT, it raises concerns.

```
`uvm_analysis_imp_decl(_port_b)

class insn_validator extends uvm_component;
  
  out_tx tx;

  uvm_analysis_imp_port_b #(out_tx,insn_validator) analysis_imp_b; 


  InstructionDecoder decode;
  
  `uvm_component_utils(insn_validator)
  
  //--------------------------------------- 
  // Constructor
  //---------------------------------------
  function new(string name, uvm_component parent);
    super.new(name, parent);
    analysis_imp_b = new("analysis_imp_b", this);
  endfunction : new
  
  //---------------------------------------
  // Analysis port write method
  //---------------------------------------
  virtual function void write_port_b(out_tx trans);
    `uvm_info(get_type_name(),$sformatf(" Inside write_port_b method. Recived trans On Analysis Imp Port"),UVM_LOW)
    `uvm_info(get_type_name(),$sformatf(" Printing trans, \n %s",trans.sprint()),UVM_HIGH)

    decode.decode_instruction(trans.i_cur_insn);

  endfunction 
  
endclass : insn_validator

```

### Environment (`lc4_env.sv`)
`lc4_env` is just an extension of `uvm_env` base class that's just a container containing `lc4_agent`, `scoreboard`, `coverage`, and `checker` class. 

```
class lc4_env extends uvm_env;

`uvm_component_utils(lc4_env)

lc4_agent agent;
lc4_scoreboard scoreboard;
insn_validator insn_check;
lc4_cov cov;


extern virtual function void connect_phase(uvm_phase phase);

function new(string name, uvm_component parent);
  super.new(name, parent);
endfunction

function void build_phase(uvm_phase phase);
  super.build_phase(phase);
  `uvm_info(get_type_name(),"Build Phase Executed", UVM_LOW)
  agent = lc4_agent::type_id::create("agent", this);
  
  // creating scoreboard'
  scoreboard = lc4_scoreboard::type_id::create("scoreboard", this);
  
  insn_check = new("insn_check", this);


  cov = lc4_cov::type_id::create("cov",this);

endfunction

task run_phase(uvm_phase phase); 
  `uvm_info(get_type_name(),"Run Phase Executed", UVM_LOW)
endtask


endclass

function void lc4_env::connect_phase(uvm_phase phase);
  super.connect_phase(phase);
  // connecting 'monitor' to scoreboard'
  //scoreboard.analysis_imp_a.connect(agent.monitor.item_collect_port); // Incorrect method
  // 'uvm_analysis_port' has method connect()
  agent.monitor.item_collect_port.connect(scoreboard.analysis_imp_a);
  agent.monitor.item_collect_port.connect(insn_check.analysis_imp_b);
  agent.monitor.item_collect_port.connect(cov.analysis_export);

endfunction

```

It mainly did two things. First, it created all the components that were present inside, and then it connected the monitor class to the scoreboard and checker class's `analysis_imp` TLM ports and the `analysis_export` of the coverage class.

### Package (`lc4_package.sv`)

The SystemVerilog package contained all the objects/components in a specific order so that they can be compiled sequentially.

```
`include "uvm_macros.svh"
import uvm_pkg::*;

package lc4_pkg;

  `include "lc4_tx.sv"
  
  `include "insn_decode.sv"
  `include "lc4_seqs.sv"
  `include "lc4_sequencer.sv"
  `include "lc4_driver.sv"
  `include "lc4_monitor.sv"
  `include "insn_validator.sv"

  `include "ref_model.sv"

  `include "lc4_scoreboard.sv"
  `include "lc4_agent.sv"
  `include "lc4_cov.sv"
  `include "lc4_env.sv"

endpackage

```

### Assertion Module (`lc4_assertion.sv`)
I created a module to house all my assertion code, and in the testbench module, I used the bind construct to integrate the assertions with the DUT without disturbing the readability or altering any signals inside the DUT module. Some features were challenging to verify or too complex to handle using a scoreboard for computation. Let's say, hypothetically, there's a condition where signal A should be high for 3 cycles, followed by signal B being high for 2 cycles, and then signal C should be high for 1 cycle. Verifying this using a scoreboard would require collecting packets in each cycle and implementing counters/queues, which could become complex. However, this can be easily implemented using assertions.

Similarly, in a pipelined design, there were certain features to verify. For instance, as long as the stall signal is activated, the program counter shouldn't be incremented. Additionally, instructions in one stage should be forwarded to the next stage, and if there's no stall in every processor cycle, among other conditions.

## Verification Plan
RTL Design of processor contains 5-stages fetch, decode, execute, memory, writeback. The features to be verified were if the instructions were perfectly pipelined stage bys tage, if the computations are occuring correctly or not, If updatition of NZP registers occuring correctly or not etc. Also pipeline implementation would cause some hazards thatwe need to detect and resolve using stalling or handling dependencies. I also had to verify them.

I verified these things using two methods.

- **Using Assertion module** - I created a separate module called 'lc4_assertion.sv' that was defining some property groups to check various formal behaviours of LC4 pipeline design such as the transmiting the instruction at current stage to next stage in next cycle and so on, Stalling the circuit whenever a stall condition met, and incrementing the program counter etc.

Below is one examples of my assertion module code that I had defined to check the behaviour of the pipeline design.

```
  //--------------------------------------------------------------------------------//
  // Pipelined Stages Verification
  //--------------------------------------------------------------------------------//
  /*
  property _logic_prop;
    @(negedge clk) disable iff(rst)
    if($fell(gwe) && (!$isunknown(i_cur_insn))) 
	    ##[0:0] (test_dec_insn == i_cur_insn)
	    ##[4:4] (test_ex_insn == test_dec_insn_prev) 
	    ##[4:4] (test_mem_insn == test_mem_insn_prev) 
	    ##[4:4] (test_cur_insn==test_cur_insn_prev);
  endproperty
  */

  property pipeline_logic_prop(signal, signal_prev);
    @(negedge clk) disable iff(rst)
      if($fell(gwe) && (!$isunknown(signal))) 
        (signal == signal_prev);
  endproperty

```

This property, `pipeline_logic_prop`, is used to verify the logic between pipeline stages. It checks if the current value of a signal (`signal`) matches its previous value (`signal_prev`) when the global write enable (`gwe`) falls (`$fell(gwe)`) and the signal is not unknown (`!$isunknown(signal)`).

By defining such properties, we can systematically verify the correctness of the pipeline design behavior during simulation.
  
- **Using Scoreboard & Reference Model** - Second method included creating a scoreboard and a reference model class that works in conjuction with scoreboard class that computes the reference outputs of all the input signals. Scoreboard was also able to record the previous stage instruction by using a queue.  In this scenario, The in-order scoreboard was an ideal choice for computing the reference outputs and checking out with the actual outputs. However, without inserting one after one instructions in one queue and comparing one after another I tried to compute and compare the instructions in the real-time. This approach was capable of using less memory/data structure while doing the simulation.

To demonstrate how reference model worked with scoreboard, below is the code that defines the reference functionality of next_pc computation.

```
  function void next_pc_calculation(
    bit [15:0] insn_x,
    bit is_ctrl_insn_x,
    bit [15:0] alu_out_x,
    bit [2:0] nzp_bits,
    bit [15:0] o_cur_pc
  );

    // Identify branch instructions (excluding degenerate B0)
    brx_test_x = ((insn_x[15:12] == 4'b0000) && (insn_x != 0) && (insn_x[11:9] != 3'b000));


    //brx_taken_x = |(brx_test_x & nzp_bits);

    // Always predict not taken (assuming sequential execution)
    brx_taken_x = 0; // Set to 0 to always predict not taken

    // Next PC based on prediction (not taken)
    X_pc_br = o_cur_pc + 1'b1; // Always use PC+1 for not taken prediction

    flush_need = is_ctrl_insn_x; // Flush only for control instructions

    next_pc = ((insn_x[15:12] == 4'b0000) && (insn_x != 0)) ? X_pc_br : is_ctrl_insn_x ? alu_out_x : (o_cur_pc + 1'b1);
  endfunction

```

The `next_pc_calculation()` function is implemented within the `ref_model` class, which serves as the reference model for computing the next program counter (`next_pc`). This function determines the next PC based on the current instruction (`insn_x`), whether it's a control instruction (`is_ctrl_insn_x`), the ALU output (`alu_out_x`), the condition code bits (`nzp_bits`), and the current program counter (`o_cur_pc`).

```

check_taken_not_taken(tx.test_brx_taken_x, ref_m.brx_taken_x);

function void lc4_scoreboard::check_taken_not_taken(logic [15:0] actual_brx_taken_x, logic [15:0] ref_brx_taken_x);
  if(actual_brx_taken_x !=  ref_brx_taken_x) begin
    `uvm_error(get_type_name(), $sformatf(
    "\nFAILED: <BRX_TAKEN_X>! actual_brx_taken_x=%0h ref_brx_taken_x=%0h\n\n", actual_brx_taken_x, ref_brx_taken_x))
  end else begin
    `uvm_info(get_type_name(),$sformatf(
    "\nPASSED: <BRX_TAKEN_X>! actual_brx_taken_x=%0h brx_taken_x=%0h\n\n", actual_brx_taken_x, ref_brx_taken_x), UVM_HIGH)
  end
endfunction

```

The `lc4_scoreboard::check_taken_not_taken` function compares the actual and reference values of branch taken signals (`actual_brx_taken_x` and `ref_brx_taken_x`). If they match, it prints a `"PASSED"` message; otherwise, it prints a `"FAILED"` message. This comparison ensures that the branch prediction logic in the DUT aligns with the expected behavior defined by the reference model, aiding in verification of the processor design.

I implemented both methods in this Verification IP to better check the functionalities accurately. I am still researching improvements in these methods that can be even more structured and data structure efficient, meanwhile ensuring the verification of each aspect correctly.

The complete implementation of this interface can be found here.

## Modification of IP from Pipelined to Superscalar
In this project, I also worked on developing a verification IP to verify a 2-way superscalar logic implemented after pipelining. For this, I had to fetch two instructions at the same time and drive them into the DUT to observe its behavior. For this purpose, I had to generate two instructions simultaneously instead of just one. Rather than defining only one instruction inside the sequence item class (`lc4_tx` and `out_tx`), I defined two instructions and added a few more testbench signals to my `out_tx` sequence item class. It would have been better to keep it to one instruction and use multiple sequencers and drivers to drive instructions to the DUT. However, to keep the testbench environment less complex, I stuck with creating two instructions.

After making changes to the transaction class, I also had to modify the interface signals where the signals were updated. I had to define them again followed by instantiating the DUT for the superscalar architecture instead of the pipelined one. Following that, I drove two instructions to the DUT and collected all input/output signals using a monitor and broadcasting it to the scoreboard, coverage, and instruction validator classes.

Additionally, I had to update my reference model code to define the desired functionalities for the 2-way superscalar logic.

## Results
I finally created a verification IP for the pipelined and superscalar design modules. Since the circuits were initially free of trivial bugs, they were mostly error-free. However, during the verification project, I detected some non-trivial errors which I resolved.I also implemented the bypassing logic for NZP bit registers in the RTL design, which was previously missing, and verified it in my testbench. Due to this, I was able to fix some non-trivial bugs. Currently, for the pipelined design, I have achieved 30% functional coverage, and for the superscalar design, I have achieved 10%.

## Future directions
1. Currently, I am only creating one testbench environment containing a single agent for recording and verifying the signals at each stage. However, I can create five agents for each of these five stages so that they can separately work on verification with respect to each stage. That would improve the scalability of the project.
2. I also plan to implement a branch prediction simulator in my Design module and define some methods to verify this functionality.

## Conclusion
