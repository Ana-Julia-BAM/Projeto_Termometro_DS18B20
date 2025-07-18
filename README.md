# Projeto Termometro DS18B20
Termômetro Digital utilizando DS18B20 e micro 8051 com comunicação serial e interface bluetooth!

## Código em Assembly
```assembly
DQ       BIT P3.7

WDLSB    DATA 30H
WDMSB    DATA 31H
swpH     EQU 0FCh
swpL     EQU 066H

          ORG 0000H
          LJMP MAIN
          ORG 000BH
          LJMP TMR0_ISR

MAIN:
          CLR   EA
          MOV   TMOD, #21H
          MOV   TH0, #swpH
          MOV   TL0, #swpL
          SETB  ET0
          SETB  TR0

          MOV   TH1, #0FDH
          MOV   TL1, #0FDH
          SETB  TR1
          MOV   SCON, #50H
          ANL   PCON, #7FH
          
          MOV   R7, #0
          MOV   R2, #4
          MOV   R0, #40H
CLRBUF:   MOV   @R0, #00H
          INC   R0
          DJNZ  R2, CLRBUF

MAIN_LOOP:
          LCALL DSWD
          LCALL SEND_TEMP
          SJMP  MAIN_LOOP

TMR0_ISR:
    PUSH PSW
    PUSH ACC
    PUSH DPL
    PUSH DPH
    
    MOV TH0, #swpH
    MOV TL0, #swpL
    
    CLR EA
    CALL APAGA_DPY
    
    MOV A, R7
    CJNE A, #0, D1

D0: 
    MOV A, 43h
    CALL convert
    MOV P0, A
    MOV P2, #00011100b
    SJMP NEXT_DISPLAY

D1:
    CJNE A, #1, D2
    MOV A, 40h
    CALL convert
    MOV P0, A
    MOV P2, #00011000b
    SJMP NEXT_DISPLAY

D2:
    CJNE A, #2, D3
    MOV A, 41h
    CALL convert
    ORL A, #80h
    MOV P0, A
    MOV P2, #00010100b
    SJMP NEXT_DISPLAY

D3:
    CJNE A, #3, D4
    MOV A, 42h
    CALL convert
    MOV P0, A
    MOV P2, #00010000b
    SJMP NEXT_DISPLAY

D4:
    CJNE A, #4, D5
    MOV A, #01100011b
    MOV P0, A
    MOV P2, #00001100b
    SJMP NEXT_DISPLAY

D5:
    MOV A, #00111001b
    MOV P0, A
    MOV P2, #00001000b

NEXT_DISPLAY:
    INC R7
    CJNE R7, #6, END_ISR
    MOV R7, #0

END_ISR:
    SETB EA
    POP DPH
    POP DPL
    POP ACC
    POP PSW
    RETI

APAGA_DPY:
    MOV P0, #0
    MOV P2, #00011100B
    RET

SEND_TEMP:
    MOV A, 43H
    ADD A, #30H
    LCALL SEND_CHAR

    MOV A, 40H
    ADD A, #30H
    LCALL SEND_CHAR

    MOV A, 41H
    ADD A, #30H
    LCALL SEND_CHAR

    MOV A, #'.'
    LCALL SEND_CHAR

    MOV A, 42H
    ADD A, #30H
    LCALL SEND_CHAR

    MOV A, #' '
    LCALL SEND_CHAR
    MOV A, #'C'
    LCALL SEND_CHAR
    
    MOV A, #0AH
    LCALL SEND_CHAR
    
    MOV A, #0DH
    LCALL SEND_CHAR
    
    RET

SEND_CHAR:
    MOV SBUF, A
    CLR TI          
WAIT_TX:
    JNB TI, WAIT_TX
    RET

DSWD:
          LCALL RSTSNR
          JNB   F0, SKIP_DS
          MOV   R0, #0CCH
          LCALL SEND_BYTE
          MOV   R0, #44H
          LCALL SEND_BYTE

          SETB  EA
          MOV   48H, #5
DLY1:     MOV   49H, #255
DLY2:     MOV   4AH, #255
DLY3:     DJNZ  4AH, DLY3
          DJNZ  49H, DLY2
          DJNZ  48H, DLY1
          CLR   EA

          LCALL RSTSNR
          JNB   F0, SKIP_DS
          MOV   R0, #0CCH
          LCALL SEND_BYTE
          MOV   R0, #0BEH
          LCALL SEND_BYTE

          LCALL READ_BYTE
          MOV   WDLSB, A
          LCALL READ_BYTE
          MOV   WDMSB, A

          LCALL TRANS12
SKIP_DS:  SETB  EA
          RET

TRANS12:
    MOV   A, WDLSB
    ANL   A, #0FH
    MOV   B, #6
    MUL   AB
    MOV   B, #10
    DIV   AB
    MOV   R3, B
    CJNE  R3, #5, NO_ROUND
NO_ROUND:
    JC    SKIP_ROUND
    INC   A
SKIP_ROUND:
    MOV   42H, A

    MOV   A, WDLSB
    ANL   A, #0F0H
    SWAP  A
    MOV   R1, A
    MOV   A, WDMSB
    ANL   A, #0FH
    SWAP  A
    ORL   A, R1

    MOV   B, #100
    DIV   AB
    MOV   43H, A

    MOV   A, B
    MOV   B, #10
    DIV   AB
    MOV   40H, A
    MOV   41H, B

    RET

convert:
    ANL A, #0FH
    MOV DPTR, #table
    MOVC A, @A+DPTR
    RET

table:
        DB      00111111b
        DB      00000110b
        DB      01011011b
        DB      01001111b
        DB      01100110b
        DB      01101101b
        DB      01111101b
        DB      00000111b
        DB      01111111b
        DB      01101111b

SEND_BYTE:
          MOV   A, R0
          MOV   R5, #8
SEN_LOOP: CLR   C
          RRC   A
          JC    SEN_1
          LCALL WRITE_0
          SJMP  SEN_CONT
SEN_1:    LCALL WRITE_1
SEN_CONT: DJNZ  R5, SEN_LOOP
          RET

READ_BYTE:
          MOV   R5, #8
          CLR   A
READ_LOOP:
          LCALL READ
          RRC   A
          DJNZ  R5, READ_LOOP
          MOV   R0, A
          RET

RSTSNR:
          SETB  DQ
          NOP
          NOP
          CLR  DQ
          MOV   R6, #250
          DJNZ  R6, $
          MOV   R6, #50
          DJNZ  R6, $
          SETB  DQ
          MOV   R6, #15
          DJNZ  R6, $
          CALL  CHCK
          MOV   R6, #60
          DJNZ  R6, $
          SETB  DQ
          RET

CHCK:
          MOV   C, DQ
          JC    NO_DEV
          SETB  F0
          SJMP  CHK_DONE
NO_DEV:   CLR   F0
CHK_DONE: RET

WRITE_0:
          CLR   DQ
          MOV   R6, #30
          DJNZ  R6, $
          SETB  DQ
          RET

WRITE_1:
          CLR   DQ
          NOP
          NOP
          NOP
          NOP
          NOP
          SETB  DQ
          MOV   R6, #30
          DJNZ  R6, $
          RET

READ:
          SETB  DQ
          NOP
          NOP
          CLR   DQ
          NOP
          NOP
          SETB  DQ
          NOP
          NOP
          NOP
          NOP
          NOP
          NOP
          NOP
          MOV   C, DQ
          MOV   R6, #23
          DJNZ  R6, $
	  RET
	  
	  END

