---
title: Lab4-1 report

---

# Lab4-1 report
## List relevant code changes from rtl/firmware
### fir.c
```
#include "fir.h"

void __attribute__ ( ( section ( ".mprjram" ) ) ) initfir() {
	//initial your fir
	for(int i=0; i<N; i++){
		inputbuffer[i]=0;
		outputsignal[i]=0;
	}
}

int* __attribute__ ( ( section ( ".mprjram" ) ) ) fir(){
	initfir();
	//write down your fir
	for(int i=0; i<N; i++){
		inputbuffer[i]=inputsignal[i];
		for(int j=0; j<N; j++){
			if((i-j)>=0){
				outputsignal[i]+=taps[i]*inputbuffer[i-j];
			}
		}
	}
	return outputsignal;
}
```
This part of C code is divided into two parts. First, the initial value is reset to zero, and then the fir function is implemented. N input signals are input into the inputbuffer in order, and then the for loop is used to control the multiplication and accumulation.
### fir.h
```
#ifndef __FIR_H__
#define __FIR_H__

#define N 11

int taps[N] = {0,-10,-9,23,56,63,56,23,-9,-10,0};
int inputbuffer[N];
int inputsignal[N] = {1,2,3,4,5,6,7,8,9,10,11};
int outputsignal[N];
#endif
```
### user_proj_example.counter.v
```
    reg        ack_reg;
    reg [31:0] wbs_dat_o_reg;
    reg [3:0]  counter;

    // BRAM
    wire        en;
    wire [31:0] Do;
    wire [31:0] A;
    wire [3:0]  wen = wbs_we_i ? wbs_sel_i : 4'b0;
    wire [31:0] Di = wbs_dat_i;
    assign A = {10'h0, wbs_adr_i[23:2]};
    assign en = wbs_stb_i & wbs_cyc_i & (wbs_adr_i[31:24] == 8'h38);

    // output
    assign wbs_ack_o     = ack_reg;
    assign wbs_dat_o     = wbs_dat_o_reg;
    assign la_data_out   = {{(127-BITS){1'b0}}, wbs_dat_o_reg};

    always @(posedge wb_clk_i, posedge wb_rst_i) begin
        if (wb_rst_i) begin
            counter <= 4'b0;
        end else begin
            counter <= ack_reg ? 4'b0 : (en ? counter + 1'b1 : counter);
        end
    end

    always @(posedge wb_clk_i, posedge wb_rst_i) begin
        if (wb_rst_i) begin
            ack_reg        <= 1'b0;
            wbs_dat_o_reg  <= 32'b0;
        end else begin
            if (counter == DELAYS) begin
                ack_reg        <= 1'b1;
                wbs_dat_o_reg  <= Do;
            end else begin
                ack_reg        <= 1'b0;
                wbs_dat_o_reg  <= 32'b0;
            end
        end
    end

    
    bram user_bram (
        .CLK(wb_clk_i),
        .WE0(wen),
        .EN0(en),
        .Di0(Di),
        .Do0(Do),
        .A0(A)
    );

```
It introduces a fixed latency for BRAM accesses via Wishbone. The slave only responds after the delay, ensuring predictable timing for read/write operations.
![image](https://hackmd.io/_uploads/H1HBlRBMgx.png)
### bram.v
```
//16KB
parameter N = 12;
    (* ram_style = "block" *) reg [31:0] RAM[0:2**N-1];


    always @(posedge CLK)
        if(EN0) begin
            Do0 <= RAM[A0[N-1:0]];
            if(WE0[0]) RAM[A0[N-1:0]][7:0] <= Di0[7:0];
            if(WE0[1]) RAM[A0[N-1:0]][15:8] <= Di0[15:8];
            if(WE0[2]) RAM[A0[N-1:0]][23:16] <= Di0[23:16];
            if(WE0[3]) RAM[A0[N-1:0]][31:24] <= Di0[31:24];
        end
        else
            Do0 <= 32'b0;
endmodule
```
Select suitable N to decide the size of BRAM.
## Memory map & linker (lds)
the memory map:
```
MEMORY {
    vexriscv_debug : ORIGIN = 0xf00f0000, LENGTH = 0x00000100
    dff : ORIGIN = 0x10000000, LENGTH = 0x00000400
    dff2 : ORIGIN = 0x10000400, LENGTH = 0x00000200
    flash : ORIGIN = 0x10000000, LENGTH = 0x01000000
    mprj : ORIGIN = 0x30000000, LENGTH = 0x00100000
    mprjram : ORIGIN = 0x38000000, LENGTH = 0x00400000
    hk : ORIGIN = 0x26000000, LENGTH = 0x00100000
    csr : ORIGIN = 0xf0000000, LENGTH = 0x00010000
}
```
The range is from 0x38000000 to 0x380001a0.
## How to move code from spiflash to user project area memory
After compiling the code and generating the .hex file, we use the testbench to initialize the SPI flash model with that .hex file. Then, during simulation, the bootloader hardware in the design will transfer the code from the SPI flash to the user project area memory.
## How to execute code from user project memory
the management processor must jump to the user code address after loading.
