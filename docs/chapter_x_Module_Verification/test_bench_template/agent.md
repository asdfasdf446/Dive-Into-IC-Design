xx_agent.sv

```systemverilog
//=============================================================================
// Project  : xx
//
// File Name: xx_agent.sv
//
// Author   : Name   : Fengfan Liu
//            Company: my_company
//            Year   : 10/27/2022
//
// Version:   0.1
//
//=============================================================================
// Description: Agent for xx
//=============================================================================

`ifndef XX_AGENT_SV
`define XX_AGENT_SV

class xx_agent extends uvm_agent;

  `uvm_componet_utils(xx_agent)

  uvm_analysis_port #(xx_item) analysis_port;

  xx_config    cfg;
  xx_sequencer sqr;
  xx_driver    drv;
  xx_monitor   mon;

  local int m_is_active = -1;

  extern function new(string name, uvm_component parent);

  extern function void build_phase(uvm_phase phase);
  extern function void connect_phase(uvm_phase phase);
  extern task main_phase(uvm_phase phase);
  extern function uvm_active_passive_enum get_is_active();

endclass: xx_agent

function xx_agent::new(string name, uvm_component parent);
  super.new(name, parent);
  analysis_port = new(""analysis_port", this);
endfunction: new

function void xx_agent::build_phase(uvm_phase phase);
  if (!uvm_config_db #(xx_config)::get(this, "", "config", cfg)) begin
    `uvm_fatal(get_type_name(), $psprintf("xx config not found"));
  end
  mon = xx_monitor::type_id::create("mon", this);
  if (get_is_active() == UVM_ACTIVE) begin
    drv = xx_driver::type_id::create("drv", this);
    sqr = src_req_sequencer::type_id::create("sqr", this);
  end
endfunction: build_phase

function void xx_agent::connect_phase(uvm_phase phase);
  if (cfg.vif == null) begin
    `uvm_warning(get_type_name(), "xx virtual interface is not set")
  end
  mon.vif = cfg.vif;
  mon.cfg = cfg;
  mon.analysis_port.connect(analysis_port);
  if (get_is_active() == UVM_ACTIVE) begin
    drv.seq_item_port.connect(sqr.seq_item_export);
    drv.vif = cfg.vif;
    drv.cfg = cfg;
  end
endfunction: connect_phase

task xx_agent::main_phase(uvm_phase phase);
  string kind = "HARD";
  forever begin
    @(negedge cfg.vif.rst_n);
    mon.handle_reset(kind);
    if (drv != null) begin
      drv.handle_reset(kind);
    end
    if (sqr != null) begin
      sqr.handle_reset(phase, kind);
    end
    @(posedge cfg.vif.rst_n);
  end
endtask: main_phase

function uvm_active_passive_enum xx_agent::get_is_active();
  if (m_is_active == -1) begin
    if (uvm_config_db #(uvm_bitstream_t)::get(this, "", "is_active", m_is_active)) begin
      if (m_is_active != cfg.is_active) begin
        `uvm_fatal(get_type_name(), $psprintf("is_active field in config_db conflicts with config object"));
      end else begin
        m_is_active = cfg.is_active;
      end
    end
  end
  return uvm_active_passive_enum'(m_is_active);
endfunction: get_is_active

`endif // XX_AGENT_SV
```

xx_config.sv

```systemverilog
//=============================================================================
// Project  : xx
//
// File Name: xx_config.sv
//
// Author   : Name   : Fengfan Liu
//            Company: my_company
//            Year   : 10/27/2022
//
// Version:   0.1
//
//=============================================================================
// Description: Configuration for agent xx
//=============================================================================

`ifndef XX_CONFIG_SV
`define XX_CONFIG_SV

class xx_config extends uvm_object;

  `uvm_object_utils(xx_config)

  virtual xx_interface vif;

  uvm_active_passive_enum is_active = UVM_ACTIVE;

  bit coverage_enable;
  bit checks_enable;
  // when enable, constraint all gap == 0
  bit performance_check_enable;

  rand int sub_seq_numbs;
  rand int sub_seq_times;
  rand bit is_repeat[100];
  rand int xx_gap[100];
  rand int xx_vld_high;
  rand int timeout;

  extern function new(string name = "xx_config");

  constraint xx_gap_cons {
    if (performance_check_enable) {
      foreach (xx_gap[i]) {
        xx_gap[i] == 0;
      }
    }
    else {
      foreach (xx_gap[i]) { 
        xx_gap[i] dist {0:/30, [5:5*2]:/30, [5*2:5*3]:/10, [5*3:5*5]:/10, [5*5:5*10]:/10, [5*10:5*20]:/9, [5*200:5*400]:/1};  
      }
    }
  }

  constraint xx_vld_high_cons {
    xx_vld_high == 1;
  }

  // all case should run total_req_num = sub_seq_times * sub_seq_numbs
  constraint req_num_cons {
    sub_seq_times dist {[100:200]:/20, [200:300]:/50, [300:400]:/30};
    sub_seq_numbs == 7;
  }


  constraint timeout_cons {
    timeout == 10000000;
  }

  // End of inlined include file

endclass : xx_config 


function xx_config::new(string name = "xx_config");
  super.new(name);
endfunction : new


`endif // XX_CONFIG_SV
```

xx_coverage.sv

```systemverilog
//=============================================================================
// Project  : xx
//
// File Name: xx_coverage.sv
//
// Author   : Name   : Fengfan Liu
//            Company: my_company
//            Year   : 10/27/2022
//
// Version:   0.1
//
//=============================================================================
// Description: Coverage for agent xx
//=============================================================================

`ifndef XX_COVERAGE_SV
  `define XX_COVERAGE_SV

  class xx_coverage extends uvm_subscriber #(xx_item);

    `uvm_component_utils(xx_coverage)

    xx_config cfg;
    xx_item   item;

    bit is_covered;

    covergroup cg_foo;
      option.per_instance = 1;

      cp_foo: coverpoint item.foo;
      cp_bar: coverpoint item.bar;

      foo: cross cp_foo, cp_bar;

    endgroup

    extern function new(string name, uvm_component parent);
    extern function void write(input xx_item t);
    extern function void build_phase(uvm_phase phase);
    extern function void report_phase(uvm_phase phase);

  endclass : xx_coverage 


  function xx_coverage::new(string name, uvm_component parent);
    super.new(name, parent);
    is_covered = 0;
    cg_foo = new();
  endfunction : new


  function void xx_coverage::write(input xx_item t);
    if (cfg.coverage_enable) begin
      item = t;
      cg_foo.sample();
      if (cg_foo.get_inst_coverage() >= 100) begin
        is_covered = 1;
      end
    end

  endfunction : write


  function void xx_coverage::build_phase(uvm_phase phase);
    if (!uvm_config_db #(xx_config)::get(this, "", "config", cfg)) begin
      `uvm_error(get_type_name(), "src_req config not found")
    end

  endfunction : build_phase


  function void xx_coverage::report_phase(uvm_phase phase);
    if (cfg.coverage_enable) begin
      `uvm_info(get_type_name(), $sformatf("Coverage score = %3.1f%%", cg_foo.get_inst_coverage()), UVM_MEDIUM)
    end else begin
      `uvm_info(get_type_name(), "Coverage disabled for this agent", UVM_MEDIUM)
    end

  endfunction : report_phase


`endif // XX_COVERAGE_SV
```

xx_driver.sv

```systemverilog

```

xx_interface.sv

```systemverilog

```

xx_item.sv

```systemverilog

```

xx_monitor.sv

```systemverilog

```

xx_sequence_library.sv

```systemverilog

```

xx_sequencer.sv

```systemverilog

```
