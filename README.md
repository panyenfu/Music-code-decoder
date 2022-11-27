# Music-code-decoder

Mucodec

Muco_dec enables us to decode those encoded sequences of music notes to the original message. 
It mainly consists of 3 parts, Finite State machine (FSM), ASCII decoder and and byte_reg. FSM, is used for state control, ASCII decoder is used to map the detected 2 digits to the ASCII code of the corresponding character, byte_reg is used yo store the last valid din. Byte_reg has a valid indicator bit, and its value will change only when the valid signal equals to 1. The effective indicator will change according to the current state of FSM. Only when both byte_reg and din are valid will the second BOS, characters and EOS be judged. 
There are 5 states in the FSM. Starting with state_RESET, which is able to detect the first “07” sequence of BOS. When valid equals to 1, byte_reg will be updated. Subsequently, it will jump to state_BOS after detecting the first ”07” sequence of BOS, and invaliding the byte_reg. (If we don’t do that, the FSM will remain unchanged.) 
Next, as mentioned above, we have the state_BOS. It will detect the second “07” sequence of BOS. If the signal valid is 1, it will update byte_reg. When both byte_reg and d_in are valid, it will further differentiate the input sequence. If the input sequence is “07”, it will jump to state_TRANS and invalid the byte_reg; otherwise, it will jump to state_RESET. (If either byte_reg or d_in is invalid, the FSM status remains unchanged.) 
Moving on, we have the state_TRANS. It can detect characters sequence and the first “70” sequence of EOS. When the signal valid is 1 but the byte_reg is invalid, it will update and valid the byte_reg. Therefore, when both d_in byte_reg are valid, it will differentiate the input sequence. There are three possible outcomes by doing so. First, if the input sequence is “70”, it will jump to state_EOS and invalid the byte_reg. Second, if the input sequence includes digit “0” or “7”, it will jump to state_ERROR. Last, if the input sequence contains none of the above digits, the FSM status is going to remain unchanged, and the input sequence will continuously be translated to the ASCII code, as well as the corresponding characters. (If either byte_reg or d_in is invalid, FSM status remains unchanged.) 
Subsequently, with state_EOS, we are able to detect the second “70” sequence of EOS. When the signal valid is 1, it will update byte_reg. When both byte_reg and d_in are valid it will differentiate the inout sequence. If the sequence is “70”, it will jump to state_RESET and invalid the byte_reg; otherwise, it will jump to state_ERROR. (If either byte_reg or d_in is invalid, FSM status remains unchanged.) 
Last but not least, we have state_ERROR. It is able to invalid the byte_reg and output the “error” signal. Furthermore, by setting the signal error high, it will jump to state_RESET. 
 
symb_det 
 
symb_det is the symbol detector used for detecting the Music code symbols. 
The symbol detector stays idle when there is no transmission due to the environment being silent. Since the symbols are transmitted at the rate of 16 symbols per second, symb_det is designed to revisit the waveform to look for next symbol at least every 1/16 seconds. Since the waveform changes every 1/16 seconds, it starts detecting after a little while as the signal begin to arrive, in order to avoid the instances when the sine wave frequencies changes from one to the next.  
In order to determine the frequency of the incoming sine wave, symb_det uses the zero-crossing detection (ZDC), where the time between two consecutive positive-to-negative or negative-to-positive movements of the sine wave with respect to the axis is used. The duration between two consecutive zero crossings with the same direction are measured to calculate the frequency. The time between the two zero-crossings is measured by counting the number of cycles between the two zero crossings.  
The count is given by the relation, cycle count = clock frequency / input sine wave frequency 
Once the frequency of the input sine wave is obtained, the frequencies are matched with their corresponding symbols ranging from 0 to 7. Since it is unlikely for the frequencies to be exact in the real world, several small ranges of frequencies are used to correspond to the symbols 0 to 7.  
The entity of symb_det includes the ports of 96Hz clock input, synchronous reset input, 12-bit ADC data input, symbol valid output, and 3-bit detection symbol output. The ports are: clk: in STD_LOGIC, clr: in STD_LOGIC, adc_data: in STD_LOGIC_VECTOR(11 DOWNTO 0), symbol_valid: out STD_LOGIC, symbol_out: out STD_LOGIC_VECTOR(2 DOWNTO 0). 
The architecture of symb_det defines the signals of symbols 0 to 7, a null symbol, and a signal counter. The symbols are s_null, s_0, s_1, s_2, s_3, s_4, s_5, s_6, s_7, s_8, s_9, of which s_8 and s_9 was not used eventually. 
At the beginning the counter is set to zero using the synchronous clear. The counter is also set to zero if the value of the counter exceeds a certain high value, otherwise it is incremented by 1 after every count. 
On the rising edge of the clock, symb_det starts detecting symbols if the ADC data exceeds a certain value.  
The counts are converted to corresponding symbols by mapping small ranges of counts (derived using the clock frequency and the frequency of input sine wave) to corresponding symbol. Ranges of values are taken to not miss the changes of the counts.because in real world scenario frequency my not be exact or exactly captured.  
Counts 180 to 186 correspond to s_0, 143 to 149 correspond to s_1, 119 to 125 correspond to s_2, 94 to 100 correspond to s_3, 79 to 85 correspond to s_4, 66 to 72 correspond to s_5, 52 to 58 correspond to s_6, 43 to 49 correspond to s_7, any other values correspond to s_null. After every clock cycle, the sign value is also stored. 
Furthermore, enable signals are generated based on 16Hz symbol rate and the detected symbols are output based on the 16Hz rate. The binary values of the output symbols are finally mapped from the corresponding symbol numbers to give the output signal values. 
when s_null => 
symbol_out <= "000";  
when s_7 => 
symbol_out <= "111"; 
when others => 
symbol_out <= "000"; 

myuart.vhd
 
myuart.vhd is contains the component for serial communication between the BASYS board (FPGA) and the computer through UART.  
For this UART program, we had the requirements of:- 
9600 Baud Rate 
8 data bits 
no parity bit (error bit) 
1 start and 1 stop bit.  
Also, the I/O ports :- 
Din[7:0] – 8 bit input data 
Wen – Write enable, assert when to write data 
Sout – 1 bit output data to be transmitted 
Busy – Asserted when UART is in the process of transmitting and can’t accept new data 
Clr and clk - inputs 
One important factor to take note of here is the clock rate. The input of our clock is 96k Hz but our baud rate (rate of data) is only 9600 as this is one of the standards/commonly used frequencies for UART. Hence, in our case, for every 10 clock cycles of 96kHz, the output sout will only change then. Hence, for each bit that sout outputs, it will be 10 clock cycles long. Hence, to do this, I have added another process called proc_baud_gen to replicate a 9600 Hz clock. In this process, at the rising edge, it will increase a signal called cnt by 1 if it is not more than 10 and if write enable wen signal is not 1. If so, cnt will reset both to 0. This ensures that cnt will count up to 9 (technically 10 but it counts starting from 0 to 9) and resets to 0 if it exceeds. When write enable is asserted, no data should be able to be process from that point on, hence, cnt will reset to 0 to ensure that the FSM knows that it is the start of the FSM and look out for the start bit. 
Based on these requirements, I designed the UART communication protocol with a finite state machine (FSM). For an FSM, there should be these components:- output decoder, state logic and input register. The output decoder reads the state of which the FSM is in and outputs to sout accordingly. The state logic defines when the FSM should move to a different state. The input register just ensures that the input is stored and managed correctly. These components run in parallel as “processes”. I will talk about it more in depth in the following paragraphs. 
Originally, there were only 3 states, i.e. S_IDLE, S_STARTBIT, S_STOPBIT, S_DATABIT. The input register would get the input when write enable input wen is asserted and copy it to our signal called din_values. However, if the clear clr input signal is asserted, then we will reset the internal signal din_values to all 0s. For the state logic and output register, it is different compared to the version now. If wen is detected, it will go to the start bit.  
 

