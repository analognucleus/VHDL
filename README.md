# VHDL
PMP


input_mux	:	process (busaddr, iow)
begin
	if (iow = '0') then
		outstatus	<= b"00000000"; --NOT CPUOK2fire
	--		data_bus <= "ZZZZZZZZ";
	else 
	
		if (mall'event and mall='1' ) then	-- address latch
				busaddr <= data_bus(7 downto 0);
		end if;
			
		if (clk'event and clk='1' ) then
			D1wrt <= iow;
			D2wrt <= iow and D1wrt;
			D3wrt <= iow and D2wrt;
			pwrite <= D2wrt and not D3wrt;
			
			if pwrite = '1' THEN
				case(busaddr) is
					when ADDR2WH =>
						data_bus(7 downto 0) <= outaddress;
						outmsgrdy <= '1';
					when DATA2WH =>
						data_bus(7 downto 0) <= outdata;
					when STATUS2CPLD
						data_bus(7 downto 0) <= outstatus;
					when RED_HI_ADDR
						data_bus(7 downto 0) <= red_thresh_hi;	
					when RED_LO_ADDR
						data_bus(7 downto 0) <= red_thresh_lo;
					when BLUE_HI_ADDR
						data_bus(7 downto 0) <= blue_thresh_hi;
					when BLUE_LO_ADDR
						data_bus(7 downto 0) <= blue_thresh_lo;
					when COLOR_SPEC_ADDR
						data_bus(7 downto 0) <= color_spec;
					when others => null;
				end case;
				
				else
					if outmsgbusy = '1' then outmsgrdy <= '0';
					end if;
			end if; -- pwrite
		end if; -- clk 
	end if; -- iow
end process input_mux;
