# VHDL
<CPUIntf>

-- Interfacing with micro controller
-- Program for Altera 
-- Copyright General Lasertronics Corp. 2013
-- PIC

library ieee;
use ieee.std_logic_1164.all;
use ieee.STD_logic_ARITH.all;
use ieee.std_logic_unsigned.all;

entity CPUIntf is port 
(
	clk, rst			:in 		std_logic;
	piow, pior		:in 		std_logic;						--from PIC	
	pmall				:in		std_logic; 						-- address latch

	data_bus			:inout	std_logic_vector(15 downto 0);	--to/from PIC
	pwrite			:buffer	std_logic;
	pread				:out		std_logic; -- PIC read
	
	inmsgdata		:in		std_logic_vector(7 downto 0);	--data message to CPU
	inmsgaddr		:in		std_logic_vector(5 downto 0);	-- from WH
	inmsgrdy			:in		std_logic;						-- message ready for CPU (address[7])
	inmsgmissed		:in		std_logic;						-- message over flow, missed by CPU (address[6]
	inmsgerror		:in		std_logic;						-- message from WH contains error
	inmsgsent		:out		std_logic;						-- 1 = received by CPU (status[7])
					
	LaserLeak		:in 		std_logic;						
	TrayLeak			:in 		std_logic;						
	LowFlowFault	:in 		std_logic;						
	DoorOpenFault	:in 		std_logic;						

	outmsgdata		:out		std_logic_vector(7 downto 0);	--data message from CPU
	
	outmsgaddr		:out		std_logic_vector(5 downto 0);	-- to WH
	outmsgrdy		:out		std_logic;						
	outmsgbusy		:in		std_logic;						-- status[7]

	ReadRedData		:in		std_logic_vector(7 downto 0);	--to/from Color Ops	
	ReadBluData		:in		std_logic_vector(7 downto 0);
	ReadDataReady	:in		std_logic;	
	Gotdata			:out		std_logic;
	WHisMark3		:in		std_logic;
	CPUOK2fire		:out		std_logic;
	
	ScanIndex		:in		std_logic_vector(6 downto 0);
	color_spec		:inout		std_logic_vector(7 downto 0);
	red_thresh_hi	:inout		std_logic_vector(7 downto 0);
	red_thresh_lo	:inout		std_logic_vector(7 downto 0);
	blue_thresh_hi	:inout		std_logic_vector(7 downto 0);
	blue_thresh_lo	:inout		std_logic_vector(7 downto 0);
	
	CPU_addr			:out	std_logic_vector(7 downto 0)
	
);
end CPUIntf;

architecture arch of CPUIntf is

	signal ior					:std_logic;	
	signal iow					:std_logic;
	signal busaddr				:std_logic_vector(7 downto 0);
	
	signal mall					:std_logic;
	signal D1wrt				:std_logic;
	signal D2wrt				:std_logic;
	signal D3wrt				:std_logic;
	
	signal outaddress			:std_logic_vector(7 downto 0);   -- latches for address
	signal outdata				:std_logic_vector(7 downto 0);	-- and data from CPU to WH
	signal outstatus			:std_logic_vector(7 downto 0);	-- latched status from CPU
	
	signal inaddress			:std_logic_vector(7 downto 0);	-- to combine 5 addr bits with ready,missed
	signal instatus			:std_logic_vector(7 downto 0);	-- to combine input status bit
	signal faultstaus			:std_logic_vector(7 downto 0);	-- to combine input fault status bit
	signal ScanIndex8			:std_logic_vector(7 downto 0);	-- to add 2 msb s to scan index	
	
	signal BReadRedData		:std_logic_vector(7 downto 0);	--buffered data from Color Ops	
	signal BReadBluData		:std_logic_vector(7 downto 0);	

	constant DATA2WH			:std_logic_vector(7 downto 0):= X"00";
	constant ADDR2WH			:std_logic_vector(7 downto 0):= X"01";
	constant STATUS2CPLD		:std_logic_vector(7 downto 0):= X"02";
	constant RED_HI_ADDR		:std_logic_vector(7 downto 0):= X"03";
	constant RED_LO_ADDR		:std_logic_vector(7 downto 0):= X"04";
	constant BLUE_HI_ADDR	:std_logic_vector(7 downto 0):= X"05";
	constant BLUE_LO_ADDR	:std_logic_vector(7 downto 0):= X"06";
	constant COLOR_SPEC_ADDR :std_logic_vector(7 downto 0):= X"07";
	
	constant DATA2CPU			:std_logic_vector(7 downto 0):= X"00";
	constant ADDR2CPU			:std_logic_vector(7 downto 0):= X"01";
	constant STATUS2CPU		:std_logic_vector(7 downto 0):= X"02";
	constant RED_DATA_ADDR	:std_logic_vector(7 downto 0):= X"03";
	constant BLU_DATA_ADDR	:std_logic_vector(7 downto 0):= X"04";
	constant SCAN_INDEX_ADDR	:std_logic_vector(7 downto 0):= X"05";
	constant FAULT_ADDR		:std_logic_vector(7 downto 0):= X"06";

	
	-- unidentified from LPEC
	---------------------------------------------------------------------
	constant REVLEVEL			:std_logic_vector(7 downto 0):= X"05";
	--------------------------------------------------------------------
	-- addresses for CPU write commands
	constant DATA2LC_ADDR	:std_logic_vector(7 downto 0):= X"00";		
	constant MISC2CPLD_ADDR	:std_logic_vector(7 downto 0):= X"01";
	-- Not used X"02";
	-- X"03"  - "0F" used on ScanGen
	

	-- addresses for CPU read commands
	constant MSG2WH_ADDR		:std_logic_vector(7 downto 0):= X"00";
	constant MISC_READBACK_ADDR	:std_logic_vector(7 downto 0):= X"01";
	constant STATUS2CPU_ADDR:std_logic_vector(7 downto 0):= X"02";
	constant REVLEVEL_ADDR	:std_logic_vector(7 downto 0):= X"03";
	
	

BEGIN

-----------------------------------  PIC PMP------------------------------- 
-- Interface to the PIC 32 PM bus.  Interfaces to address/ data multiplexed
-- PMP bus. 
-- Reads back status and data to CPU.

PicPMP : process (rst, iow, ior, clk, mall)
begin

	if (rst = '0') then
		outstatus	<= b"00000000"; --NOT CPUOK2fire
	else 
		if (mall'event and mall='1' ) then	
				busaddr <= data_bus(7 downto 0);
		end if;
			
		if (clk'event and clk='1' ) then
			D1wrt <= iow;
			D2wrt <= iow and D1wrt;
			D3wrt <= iow and D2wrt;
			pwrite <= D2wrt and not D3wrt;
			
			
			if pwrite = '1' THEN 		-- messages from PIC to FPGA
				case(busaddr) is 			-- readng from data_bus
					when ADDR2WH =>
						outmsgaddr <= data_bus(13 downto 8);		-- address in MSB
						outmsgdata <= data_bus(7 downto 0);			-- data in LSB
						outmsgrdy <= '1';
					when DATA2WH =>
						outdata <= data_bus(7 downto 0);
					when STATUS2CPLD =>									-- 2WH is old version? 
						outstatus <= data_bus(7 downto 0);
					when RED_HI_ADDR =>
						red_thresh_hi <= data_bus(7 downto 0);	
					when RED_LO_ADDR =>
						red_thresh_lo <= data_bus(7 downto 0);
					when BLUE_HI_ADDR =>
						blue_thresh_hi <= data_bus(7 downto 0);
					when BLUE_LO_ADDR =>
						blue_thresh_lo <= data_bus(7 downto 0);
					when COLOR_SPEC_ADDR =>
						color_spec <= data_bus(7 downto 0);
					when others => null;
				end case;
				
				else
					if outmsgbusy = '1' then outmsgrdy <= '0';
					end if;
			end if; -- pwrite
		end if; -- clk 
	end if; -- iow
end process PicPMP;

MsgRead: process (rst, clk, ior, inmsgrdy) 
BEGIN
	IF rst = '0' THEN
		inmsgsent	<= '0';
	ELSE
		IF (clk'event and clk='1' ) THEN
			IF busaddr = MSG2WH_ADDR AND ior='1' AND inmsgrdy = '1' THEN
				inmsgsent	<= '1';
			ELSE
				IF inmsgrdy = '0' THEN inmsgsent	<= '0'; END IF;
			END IF;
		END IF; -- (clk'event and 
	END IF; -- rst = '0' 
END PROCESS	MsgRead;


output_mux	:	process (data_bus, ior)		-- FPGA to PIC32
begin
	if (ior = '0') then
		 data_bus(7 downto 0) <= "ZZZZZZZZ";
	else
		 case(busaddr) is -- writing to data_bus
			when ADDR2CPU => 
				data_bus(7 downto 0) <= inaddress;
			when DATA2CPU => 
				data_bus(7 downto 0) <= inmsgdata;
			when STATUS2CPU =>
				data_bus(7 downto 0) <= instatus;
			when FAULT_ADDR =>
				data_bus(7 downto 0) <= faultstaus;
			when RED_DATA_ADDR =>
				data_bus (15 downto 8)	<= BReadRedData;
			when BLU_DATA_ADDR =>
				data_bus (15 downto 8)	<= BReadBluData;
			when SCAN_INDEX_ADDR =>
				data_bus (15 downto 8)	<= ScanIndex8;		
			when others => null;
		end case;	
	end if;
end process output_mux;	
		
inaddress(5 downto 0) <= inmsgaddr;
inaddress(6) 		<= inmsgmissed;
inaddress(7)		<= inmsgrdy;
instatus(7)			<= outmsgbusy;
faultstaus(3)		<=	DoorOpenFault;
faultstaus(4)		<=	LaserLeak;
faultstaus(5)		<=	TrayLeak;
faultstaus(6)		<=	LowFlowFault;

CPUOK2fire			<= outstatus(0);

ScanIndex8(7)		<= '0';
ScanIndex8(6 downto 0) <= ScanIndex;

outmsgdata			<= outdata;
outmsgaddr(5 downto 0) <= outaddress(5 downto 0);

instatus(0)			<= WHisMark3;

iow <= piow ;
ior <= pior ;
mall <= pmall;

pread <= pior;
CPU_addr	 <= busaddr;

END arch; 


