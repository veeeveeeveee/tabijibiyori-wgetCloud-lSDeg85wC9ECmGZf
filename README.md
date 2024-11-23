
# 一、异步FIFO需要注意的问题


所谓异步FIFO，指的是写时钟与读时钟可以不同步，读时钟可以比写时钟快，反之亦然。思考一下，这样会直接地造成两个问题：


## 1\. 读满或者写满


由于异步FIFO的基本存储单元是双端口RAM，因此读写速率不一致，就会造成读满或者写满的问题。


## 2\. 跨时钟域的同步


为了判断读满、写满的情况，势必需要将写指针（告诉读模块，写到哪个位置了，我还可不可以继续读？）同步到读模块，（或者读指针同步到写模块，通知写模块，现在读到哪里了，我还能不能继续写啦？如果还没读，我再写一轮不就把数据覆盖了啊？），这样就会存在跨时钟域的同步问题。
因此，针对上述问题，我们解决办法如下：


## 3\. 针对问题一，将读指针与写指针进行比较，产生读空、写满标志。


思考一下如何判断读空、写满标志呢，假设有一个深度为8的RAM，那么其地址线的宽度为3，这里我们扩展一位，让最高位作为读空、写满标志，（实际给到RAM的只有\[2:0]）,其原理如下
假设写指针写到了0111，此时读指针也读到了0111，意味着读指针追上了写写指针，那么此时就是读空了；
假设写指针写到了1000（实际上是第二轮的000），此时读指针读到了000（第一轮的000），那么就是写满。
![image](https://img2024.cnblogs.com/blog/3539410/202411/3539410-20241122113255422-601058503.png)


因此，可以总结：
**当最高位相同，其余位相同认为是读空
当最高位不同，其余位相同认为是写满**


## 4\. 针对问题二：两级寄存器同步 \+ 格雷码


我们将读写指针编码为Gray码并打两拍进行同步。采用Gray码的原因可参考上一篇博客，简单来说就是Gray码相邻两个编码之间只存在一个bit变化，避免多个bit同时跳变的问题。再进行两拍同步就可以将读写指针进行异步时钟同步化了。


## 5\. 由于问题二，采用Gray码，导致我们判断读空及写满的逻辑需要稍微改变下了：


我们观察一下基于gray码的读空、写满的情况：
![image](https://img2024.cnblogs.com/blog/3539410/202411/3539410-20241122113316408-594245203.png)


**因此，用格雷码判断是否为读空或写满时应看最高位和次高位是否相等：即：
当最高位和次高位相同，其余位相同认为是读空
当最高位和次高位不同，其余位相同认为是写满**


# 二、异步FIFO架构


根据上述讨论，我们可以总结一个异步FIFO的架构包括以下几个部分：


1. 双端口RAM，作为FIFO的存储体。可以采用硬件描述的方式描述一个RAM，也可以采用IP核、原语的方式。
2. FIFO写模块，用于产生写地址，写使能，写满等信号
3. FIFO读模块，用于产生读地址，读使能、读空等信号。
4. Gray码转换模块，用于自然二进制与Gray码转换
5. 时钟同步，用于将读写指针打拍同步。
结构框架如下：
![image](https://img2024.cnblogs.com/blog/3539410/202411/3539410-20241122113340523-2111358148.png)


# 三、Verilog HDL



```
`// // =============================================================================
// File Name	: async_fifo.v
// Module		: async_fifo
// Function		: Synthesis Techniques for Asynchronous FIFO Design
// Type			: RTL
// -----------------------------------------------------------------------------
// Update History :
// -----------------------------------------------------------------------------
// Rev.Level	Date		 	Coded by		Contents
// 0.0.1		2024/11/22   	Dongyang		first edit
// End Revision
// =============================================================================
module async_fifo#(
        parameter  P_DATA_WIDTH  = 'd16 ,
        parameter  P_FIFO_DEPTH  = 'd8 
)
(
		input                                            rst_n			,
		input                                            fifo_wr_clk	,
		input	   [P_DATA_WIDTH - 1:0]                  fifo_wr_data	,
        input                                            fifo_rd_en     ,
        input                                            fifo_rd_clk    ,
		input                                            fifo_wr_clk    ,
        output reg                                       r_fifo_full	,
		output     [P_DATA_WIDTH - 1:0]                  fifo_rd_data   ,
		output reg                                       r_fifo_empty		
	);

//******************** function ************************/
// calculate fifo address width according to  P_FIFO_DEEP
function  integer clobg2(input integer number);
          for(clobg2 = 0 ; number >0 ; clobg2 = clobg2 + 1) begin
            number = number >> 1;
          end
endfunction


//******************* parameter ***********************/
localparam P_FIFO_ADDRESS_WIDTH =  clobg2(P_FIFO_DEPTH -1);

//******************* Siganal define*******************/

reg	    [P_FIFO_ADDRESS_WIDTH - 0:0]    rdaddress              ; //RAM地址，扩展一位用于同步
reg	    [P_FIFO_ADDRESS_WIDTH - 0:0]    wraddress              ;
wire	[P_FIFO_ADDRESS_WIDTH - 0:0]	gray_rdaddress         ;
wire	[P_FIFO_ADDRESS_WIDTH - 0:0]	gray_wraddress         ;
/*同步寄存器*/
reg	    [P_FIFO_ADDRESS_WIDTH - 0:0]    sync_w2r_r1,sync_w2r_r2;
reg	    [P_FIFO_ADDRESS_WIDTH - 0:0]    sync_r2w_r1,sync_r2w_r2;
wire                                    fifo_empty             ;
wire                                    fifo_full              ;
// ******************* assign *************************/		
		/*二进制转化为格雷码计数器*/
assign gray_rdaddress = (rdaddress >>1) ^ rdaddress;//(({1'b0,rdaddress[9:1]}) ^ rdaddress);
		
		/*二进制转化为格雷码计数器*/
assign gray_wraddress = (({1'b0,wraddress[P_FIFO_ADDRESS_WIDTH - 0:1]}) ^ wraddress);
		
assign fifo_empty = (gray_rdaddress == sync_w2r_r2);
		
assign fifo_full = (gray_wraddress == {~sync_r2w_r2[P_FIFO_ADDRESS_WIDTH - 0:P_FIFO_ADDRESS_WIDTH - 1],sync_r2w_r2[P_FIFO_ADDRESS_WIDTH - 2:0]});

//******************module inst **************************/
// 根据FPGA 型号例化RAM
		ram  ram(
			.data		(fifo_wr_data		),
			.rdaddress(rdaddress[P_FIFO_ADDRESS_WIDTH - 1:0]),
			.rdclock	(fifo_rd_clk	),
			
			.wraddress(wraddress[P_FIFO_ADDRESS_WIDTH - 1:0]),
			.wrclock	(fifo_wr_clk	),
			.wren		(fifo_wr_en	),
			.q			(fifo_rd_data)
			);	
//*******************always *******************************/
		/*在读时钟域同步FIFO空 sync_w2r_r2 为同步的写指针地址 延迟两拍 非实际 写指针值 但是确保不会发生未写入数据就读取*/	
		always@(posedge fifo_rd_clk or negedge rst_n)
		if(!rst_n)
			r_fifo_empty <= 1'b1;
		else 
			r_fifo_empty <= fifo_empty;


			/*在写时钟域判断FIFO满 sync_r2w_r2 实际延迟两个节拍 可能存在非满判断为满 但不会导致覆盖*/
		always@(posedge fifo_wr_clk or negedge rst_n)
		if(!rst_n)
			r_fifo_full <= 1'b1;
		else 									
			r_fifo_full <= fifo_full;//格雷码判断追及问题			
			
			
		/*读数据地址生成*/
		always@(posedge fifo_rd_clk or negedge rst_n)
		if(!rst_n)
			rdaddress <= 'b0;
		else if(fifo_rd_en && ~fifo_empty)begin
			rdaddress <= rdaddress + 1'b1;
		end
		
		/*写数据地址生成*/
		always@(posedge fifo_wr_clk or negedge rst_n)
		if(!rst_n)
			wraddress <= 'b0;
		else if(fifo_wr_en && ~r_fifo_full)begin
			wraddress <= wraddress + 1'b1;
		end
		
		/*同步读地址到写时钟域*/
		always@(posedge fifo_wr_clk or negedge rst_n)
		if(!rst_n)begin
			sync_r2w_r1 <= 'd0;
			sync_r2w_r2 <= 'd0;
		end else begin
			sync_r2w_r1 <= gray_rdaddress;
			sync_r2w_r2 <= sync_r2w_r1;		
		end

		/*同步写地址到读时钟域, 同步以后 存在延迟两个节拍*/
		always@(posedge fifo_rd_clk or negedge rst_n)
		if(!rst_n)begin
			sync_w2r_r1 <= 'd0;
			sync_w2r_r2 <= 'd0;
		end else begin
			sync_w2r_r1 <= gray_wraddress ;
			sync_w2r_r2 <= sync_w2r_r1;		
		end		
endmodule`

```

# 四、reference


1\.《Simulation and Synthesis Techniques for Asynchronous FIFO Design》Clifford E. Cummings, Sunburst Design, Inc.
cliffc@sunburst\-design.com
2\.[https://github.com/DeamonYang](https://github.com)


 本博客参考[樱花宇宙官网](https://yzygzn.com)。转载请注明出处！
