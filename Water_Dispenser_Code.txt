CPU	"8051.TBL"		;Declare CPU
INCL	"8051.INC"

PA:	EQU	4000H		;Initiate Ports
PB:	EQU	4001H
PC:	EQU	4002H
PCTR:	EQU	4003H

ORG	2000H			;Line starts from 2000


MOV	DPTR, #PCTR		;PC as Input ports	
MOV	A, #10001001B		;PA PB as Output ports
MOVX	@DPTR, A

MOV TMOD, #00000001B		;Using Timer 0 Mode 1
MOV IE, #10000101B		;Using both INT 0 and INT 1
				;External INterrupt 0 and external interrupt 1

############################# MAIN FUNCTION #############################################

MAIN:	CLR	P1.5		;Clear P1.5 LED3 (Emergency Indicator)
	ACALL	DELAY		;Calling Subroutine
	ACALL	IR
	ACALL	OPEN
	ACALL	ULTRASONIC
	ACALL	IR2
	SJMP	MAIN		;Loop back to MAIN program 

############################ IR FUNCTION #############################################

IR: 	MOV	A, #00000000B	;Clear LED4 (IR indicator)
	MOV	DPTR, #PB
	MOVX	@DPTR, A
	MOV 	DPTR, #PC	;Get data input from PC
	MOVX 	A, @DPTR
	ANL	A, #00000001B	;AND with #00000001B
	CJNE	A, #00000001B,IR	;Loop to IR if not equal after compare with #00000001B
	ACALL	LED		;Call LED4 (IR Indicator)
	RET

############################# IR2 FUNCTION #############################################

IR2: 	MOV	A, #00000000B	
	MOV	DPTR, #PB
	MOVX	@DPTR, A
	MOV 	DPTR, #PC
	MOVX 	A, @DPTR
	ANL	A, #00000001B
	CJNE	A, #00000001B, MAIN	;Jump to IR if not equal after compare with #00000001B
	MOV	A, #00000010B		;Energise LED5 (IR Indicator 2)
	MOV	DPTR, #PA
	MOVX	@DPTR, A
	SJMP	IR2			;Loop IR2 subroutine

############################# LED FUNCTION #############################################

LED:	MOV	A, #00000001B	;Set Port B0 to energise LED4 (IR Indicator)
	MOV	DPTR, #PB
	MOVX	@DPTR, A
	ACALL	DELAY		;CAll Delay subroutine
	RET

############################# ULTRASONIC FUNCTION #############################################

ULTRASONIC:	MOV	tl0, #0FAH 	;10us HIGH for trigger
		MOV	th0, #0FFH
		SETB	P1.1		;Send HIGH to trigger
		SETB	tr0		;Start Timer 0

S_TRIG:	JNB	tf0, S_TRIG		;Wait for time flag reach HIGH
	CLR	P1.1			;Send LOW to stop trigger
	CLR	tr0			;Stop Timer 0
	CLR	tf0			;Clear timer flag 0
	CLR	A

ECHO:	JNB	P1.2, ECHO	;Check whether wave have been received or not

WAVE_TIME:	MOV	th0, #0FFH	;amount of time travelled by wave in 1 cm
		MOV	tl0, #0CDH
		SETB	tr0		;start timer 0 

DISTANCE:	JNB	tf0, DISTANCE	;wait for time flag 0 to reach HIGH 
		CLR	tr0		;stop timer 0 
		CLR	tf0		;clear timer flag 0 
		INC	A		;increase 1 count (1 cm)
		JB	P1.2, WAVE_TIME	;jump back to WAVER_TIME subroutine when 
					;there is still echo signal
COMPARE:	CJNE	A, #8, MEASURE	;if a less than 15, carry flag is set 
MEASURE:	JNC	MORE	;if carry flag is not set(more or equal 8) 
				;jump to MORE
LESS:		SETB	P1.3	;if less than 8
		CLR	P1.4
		ACALL	CLOSE	;Call CLOSE function
		RET

MORE:		SETB	P1.4	;if more or equal 8
		CLR	P1.3
		SJMP	ULTRASONIC

############################# DELAY FUNCTION #############################################

DELAY:	MOV R5, #28		;Delay for 3 seconds
HERE1:	MOV R4, #255
HERE2:	MOV R3, #255
HERE3:	DJNZ R3, HERE3
	DJNZ R4, HERE2
	DJNZ R5, HERE1
	RET

############################# OPEN FUNCTION (SERVO MOTOR) #############################################

OPEN: 	MOV R0, #20		
OPENA:	MOV	A, #00000001B	;Trigger HIGH
	MOV	DPTR, #PA
	MOVX	@DPTR, A
	MOV	TL0, #099H	;For 1.5ms 0 to 90 degree
	MOV 	TH0, #0FAH
	ACALL 	TIMER		;Call TIMER Function

	MOV	A, #00000000B	;Trigger LOW
	MOV	DPTR, #PA
	MOVX	@DPTR, A
	MOV 	TL0, #065H	;For 18.5ms 
 	MOV 	TH0, #0BDH	
	ACALL 	TIMER
	DJNZ	R0, OPENA
	RET

############################# CLOSE FUNCTION (SERVO MOTOR) #############################################

CLOSE: 	MOV R0, #20
CLOSEA:	MOV	A, #00000001B	;Trigger HIGH
	MOV	DPTR, #PA
	MOVX	@DPTR, A
	MOV	TL0, #065H	;For 1.0ms 90 to 0 degree
	MOV 	TH0, #0FCH
	ACALL 	TIMER

	MOV	A, #00000000B	;Trigger LOW
	MOV	DPTR, #PA
	MOVX	@DPTR, A
	MOV 	TL0, #099H	;For 18.5ms 
 	MOV 	TH0, #0BBH	
	ACALL 	TIMER
	DJNZ	R0, CLOSEA
	RET

############################# TIMER FUNCTION (SERVO MOTOR) #############################################

TIMER: 				;TIMER Function
	SETB 	TR0		;Start Timer
AGAIN:	JNB 	TF0, AGAIN	;If Timer Flag is not set reloop
	CLR 	TR0		;Stop Timer
	CLR 	TF0		;Clear Timer Flag
	RET

############################# INTERRUPT #############################################
============================= INT1 ==================================================

STIRRER: MOV	DPTR, #PA	;Trigger HIGH to Port A7
	 MOV	A, #10000000B	;Stirrer Function
	 MOVX	@DPTR, A
	 ACALL	DELAY		;Delay for 6 seconds
	 ACALL	DELAY
	 MOV	DPTR, #PA	;Trigger LOW (off)
	 MOV	A, #00000000B
	 MOVX	@DPTR, A
	 RETI

============================= INT0 ==================================================

EMER:	ACALL	CLOSE 		;Emergency Stop Function call CLOSE Function
	SETB	P1.5		;Trigger LED3 Emergency Stop Indicator
	MOV	DPTR, #PA
	MOV	A, #00000000B	;Trigger LOW to all Port A
	MOVX	@DPTR, A
	MOV	DPTR, #PB
	MOV	A, #00000000B  	;Trigger LOW to all Port B
	MOVX	@DPTR, A
	CLR	P1.3		;Clear LED1
	CLR 	P1.4		;Clear LED2
	RETI

#######################################################################################
LJMP	MAIN			;Long Jump to make sure when return interrupts fail
				;Program loop to MAIN Function
ORG	3FF6H			;External Interrupt 1 address
LJMP	STIRRER			;Jump to Stirrer Function


LJMP	MAIN

ORG	3FF0H			;External Interrupt 1 address
LJMP	EMER			;Jump to Emergency Stop Function

LJMP	MAIN

END
	