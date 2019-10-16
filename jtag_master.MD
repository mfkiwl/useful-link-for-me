`ifndef JTAG_MASTER_SV
`define JTAG_MASTER_SV

interface jtag_master #(
	parameter IR_WIDTH = 5,
	parameter DR_WIDTH = 40,
	parameter RESET_TIME = 100,
	parameter OUTPUT_DELAY = 0.01
) (
	input tck,
	input tdo,
	output reg tdi,
	output reg tms,
	output reg trst_n
);

reg tdi_d;
reg tms_d;
reg trst_n_d;

localparam	TEST_LOGIC_RESET = 0,
			RUN_TEST_IDLE = 1,
			SELECT_DR = 2,
			CAPTURE_DR = 3,
			SHIFT_DR = 4,
			EXIT1_DR = 5,
			PAUSE_DR = 6,
			EXIT2_DR = 7,
			UPDATE_DR = 8,
			SELECT_IR = 9,
			CAPTURE_IR = 10,
			SHIFT_IR = 11,
			EXIT1_IR = 12,
			PAUSE_IR = 13,
			EXIT2_IR = 14,
			UPDATE_IR = 15;
			
// 在下降沿改变tdo, tms, trst_n比较方便控制
always @(negedge tck)
begin
	tdi <= #OUTPUT_DELAY tdi_d;
	tms <= #OUTPUT_DELAY tms_d;
	trst_n <= #OUTPUT_DELAY trst_n_d;
end

initial
begin
	tdi_d = 0;
	tms_d = 0;
	trst_n_d = 0;
	// 持续复位一段时间，这段时间里tdo, tms可取随机值
	force tdi = $urandom_range(0, 1);
	force tms = $urandom_range(0, 1);	
	force trst_n = 0;
	#RESET_TIME;
	
	release tdi;
	release tms;
	release trst_n;
end

// 通过持续将trst_n拉低的方式让jtag_slave复位
task async_tap_reset;
	integer N = 50;
	tms_d = 1'b1;
	trst_n_d = 1'b0;
	// 将trst_n拉低N个周期
	repeat(N)
		@(posedge tck);
	tms_d = 1'b0;
	
	// repeat(5)
		// @(posedge tck);
endtask

// 通过让tms = 1的方式让jtag_slave复位
task sync_tap_reset;
	integer N = 50;
	
	@(posedge tck);
	tms_d = 1'b1;
	// 让tms = 1保持N个周期，按照TAP Controller State Transitions的规则来说slave应当回到
	// TEST_LOGIC_RESET状态
	repeat(N)
		@(posedge tck);
	tms_d = 1'b0;
	
	// repeat(5)
		// @(posedge tck);
endtask

task write_ir(
	input [IR_WIDTH-1:0] data
);
	// 在PAUSE_IR状态中停留多久
	bit [31:0] pause_cycle = $urandom_range(1, 8);
	integer i;
	
	@(posedge tck);
	
	tms_d = 1'b1;
	@(posedge tck);			// enter SELECT_DR
	@(posedge tck);			// enter SELECT_IR
	
	tms_d = 1'b0;
	@(posedge tck);			// enter CAPTURE_IR
	
	tms_d = 1'b0;
	@(posedge tck);			// enter SHIFT_IR
	
	for(i = 0; i < IR_WIDTH - 2; i++)
	begin
		tdi_d = data[i];		// from data[LSB]
		@(posedge tck);
	end
	
	tdi_d = data[IR_WIDTH-2];
	tms_d = 1'b1;
	// 发送完data[MSB-1]之后进入EXIT1_IR，即在发送data[MSB]之前在PAUSE_IR中停留几个周期
	// 测试jtag_slave能否正确处理PAUSE_IR状态
	@(posedge tck);	

	tms_d = 1'b0;
	@(posedge tck);			// enter PAUSE_IR
	
	repeat(pause_cycle)
		@(posedge tck);			// stay in PAUSE_IR
		
	tms_d = 1'b1;
	@(posedge tck);			// enter EXIT2_IR
	
	tms_d = 1'b0;
	@(posedge tck);			// enter SHIFT_IR
	
	// send data[MSB]
	tdi_d = data[IR_WIDTH-1];
	tms_d = 1'b1;
	@(posedge tck);			// enter EXIT1_IR
	
	tms_d = 1'b1;
	@(posedge tck);			// enter UPDATE_IR
	
	tms_d = 1'b0;
	@(posedge tck);			// enter RUN_TEST_IDLE

endtask

task write_dr(
	input [DR_WIDTH-1:0] data,
	input int len
);
	// 在PAUSE_DR状态中停留多久
	bit [31:0] pause_cycle = $urandom_range(1, 8);
	int i;
	
	@(posedge tck);
	
	tms_d = 1'b1;
	@(posedge tck);			// enter SELECT_DR
	
	tms_d = 1'b0;
	@(posedge tck);			// enter CAPTURE_DR
	
	tms_d = 1'b0;
	@(posedge tck);			// enter SHIFT_DR
	
	for(i = 0; i < len - 2; i++)
	begin
		tdi_d = data[i];	// from data[LSB]
		@(posedge tck);
	end
	
	tdi_d = data[len-2];
	tms_d = 1'b1;
	@(posedge tck); 		// enter EXIT1_DR
	
	tms_d = 1'b0;
	@(posedge tck); 		// enter PAUSE_DR
	
	repeat(pause_cycle)
		@(posedge tck);			// stay in PAUSE_DR
		
	tms_d = 1'b1;
	@(posedge tck);			// enter EXIT2_DR
	
	tms_d = 1'b0;
	@(posedge tck);			// enter SHIFT_DR
	
	// send data[MSB]
	tdi_d = data[len-1];
	tms_d = 1'b1;
	@(posedge tck);			// enter EXIT1_DR
	
	tms_d = 1'b1;
	@(posedge tck);			// enter UPDATE_DR
	
	tms_d = 1'b0;
	@(posedge tck);			// enter RUN_TEST_IDLE
	
endtask

task get_data(
	input int len,
	output [DR_WIDTH-1:0] data
);
	// 在PAUSE_DR状态中停留多久
	bit [31:0] pause_cycle = $urandom_range(1, 8);
	int i;
	
	@(posedge tck);
	// 注意此时发给jtag_slave的数据不能随便取，不然可能导致jtag_slave产生执行未知操作
	tdi_d = 1'b0;
	
	tms_d = 1'b1;
	@(posedge tck);			// enter SELECT_DR
	
	tms_d = 1'b0;
	@(posedge tck);			// enter CAPTURE_DR
	
	tms_d = 1'b0;
	@(posedge tck);			// enter SHIFT_DR
	
	/* 
	// donot enter PAUSE_DR
	for(i = 0; i < len; i++)
	begin
		// 一开始进入循环时，slave才刚刚进入SHIFT_DR状态，所以要等下一个posedge tck(slave会在下降沿改变tdo)
		// master才能收到来自于slave的tdo
		@(posedge tck);
		data[i] = tdo;		// from data[LSB]
	end
	
	tms_d = 1'b1;
	@(posedge tck); 		// enter EXIT1_DR
	@(posedge tck); 		// enter UPDATE_DR
	
	tms_d = 1'b0;
	@(posedge tck);			// enter RUN_TEST_IDLE
	*/
	 
	// enter PAUSE_DR
	for(i = 0; i < len - 2; i++)
	begin
		// 开始进入循环时，slave才刚刚进入SHIFT_DR状态，所以要等下一个posedge 
		// tck(slave会在下降沿改变tdo)
		// master才能收到来自于slave的tdo
		@(posedge tck);
		data[i] = tdo;		// from data[LSB]		
	end	

	tms_d = 1'b1;
	@(posedge tck); 		// enter EXIT1_DR
	data[len-2] = tdo;		// get data[MSB-1]
	
	tms_d = 1'b0;
	@(posedge tck); 		// enter PAUSE_DR
	
	repeat(pause_cycle)
		@(posedge tck);			// stay in PAUSE_DR
		
	tms_d = 1'b1;
	@(posedge tck);			// enter EXIT2_DR
	
	tms_d = 1'b0;
	@(posedge tck);			// enter SHIFT_DR
	

	tms_d = 1'b1;
	@(posedge tck);			// enter EXIT1_DR
	data[len-1] = tdo;		// get data[MSB]
	
	tms_d = 1'b1;
	@(posedge tck);			// enter UPDATE_DR
	
	tms_d = 1'b0;
	@(posedge tck);			// enter RUN_TEST_IDLE
	
endtask

endinterface
`endif
