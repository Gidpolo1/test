# test
pp
module CYBERcobra (
  input  logic         clk_i,
  input  logic         rst_i,
  input  logic [15:0]  sw_i,
  output logic [31:0]  out_o
);

  logic [31:0] pc_reg;
  logic [31:0] next_pc;
  logic [31:0] pc_adder_b;
  logic        pc_adder_carry_out;
  logic [31:0] instr;

  logic        j_sel;
  logic        b_sel;
  logic [1:0]  ws_sel;
  logic [4:0]  alu_op;
  logic [4:0]  ra1;
  logic [4:0]  ra2;
  logic [4:0]  wa;

  logic [31:0] rf_const_extended;
  logic [31:0] offset_extended;

  logic [31:0] rf_rdata1;
  logic [31:0] rf_rdata2;
  logic [31:0] rf_wdata;
  logic        rf_we;

  logic [31:0] alu_result;
  logic        alu_flag;

  assign j_sel  = instr[31];
  assign b_sel  = instr[30];
  assign ws_sel = instr[29:28];
  assign alu_op = instr[27:23];
  assign ra1    = instr[22:18];
  assign ra2    = instr[17:13];
  assign wa     = instr[4:0];

  assign rf_const_extended = { {9{instr[27]}}, instr[27:5] };
  assign offset_extended = { {22{instr[12]}}, instr[12:5], 2'b00 };

  assign rf_we = !(j_sel || b_sel);

  always_comb begin
    case (ws_sel)
      2'b00:   rf_wdata = rf_const_extended;
      2'b01:   rf_wdata = alu_result;
      2'b10:   rf_wdata = {16'b0, sw_i};
      2'b11:   rf_wdata = 32'b0;
      default: rf_wdata = 32'b0;
    endcase
  end

  logic jump_taken;
  assign jump_taken = j_sel || (b_sel && alu_flag);

  assign pc_adder_b = jump_taken ? offset_extended : 32'd4;

  always_ff @(posedge clk_i) begin
    if (rst_i) begin
      pc_reg <= 32'b0;
    end else begin
      pc_reg <= next_pc;
    end
  end

  fulladder32 pc_adder (
    .a_i     (pc_reg),
    .b_i     (pc_adder_b),
    .carry_i (1'b0),
    .sum_o   (next_pc),
    .carry_o (pc_adder_carry_out)
  );

  instr_mem imem (
    .read_addr_i (pc_reg),
    .read_data_o (instr)
  );

  register_file rf (
    .clk_i          (clk_i),
    .write_enable_i (rf_we),
    .write_addr_i   (wa),
    .read_addr1_i   (ra1),
    .read_addr2_i   (ra2),
    .write_data_i   (rf_wdata),
    .read_data1_o   (rf_rdata1),
    .read_data2_o   (rf_rdata2)
  );

  alu my_alu (
    .alu_op_i    (alu_op),
    .a_i         (rf_rdata1),
    .b_i         (rf_rdata2),
    .result_o    (alu_result),
    .flag_o      (alu_flag)
  );

  assign out_o = rf_rdata1;

endmodule
