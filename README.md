# The Buffee Accelerator

This is the hardware repo for Buffee, split from the [PJIT](https://github.com/nonarkitten/pseudo-jit) repo.

## Buffee

Buffee is a drop-in 68000 socket compatible accelerator designed to fit in the footprint of a DIP64 chip to be fully compatible with all possible 68000 systems. It is based on the Octavo OSD3558, a package-in-package design that integrates the Texas Instruments ARM Cortex-A8 AM3358 CPU, 512MB or 1GB of DDR3 memory, TPS65217C PMIC, LDO, all the needed passives, as well as 4KB of EEPROM into a single BGA package. The boad additionally features 16MB of Flash memory for boot-up as well as a GreenPAK SLG46826V to translate the TI's GPMC bus signals to M68K bus signals.

### Buffee Master Pinout Table
```
                                    PIN------------------------------------     SCHEMATIC-REFERENCES----------------------
    SIGNAL NAME             FN      AM335x  OSD335x     IO   NAME               LABEL         M68K BUS IO         GREENPAK  NOTES
    pr1_pru0_pru_r30_0      PRU0    A13     A1          O    MCASP0_ACLKX       PRU_VMA       U6.31 -> M68K_VMA   -         DNFW
    pr1_pru0_pru_r31_1      PRU0    B13     A2          I*   MCASP0_FSX         BBB_BGACK     U6.44 <- M68K_BGACK -         DNFW
    pr1_pru0_pru_r31_2      PRU0    D12     B2          I*   MCASP0_AXR0        BBB_RESET     U6.35 <- M68K_RST   -         DNFW
    pr1_pru0_pru_r31_3      PRU0    C12     B1          I*   MCASP0_AHCLKR      BBB_BR        U6.42 <- M68K_BR    -         DNFW
    pr1_pru0_pru_r31_4      PRU0    B12     A3          I    MCASP0_ACLKR       BBB_CLK7      U6.39 <- M68K_CLK7  -         DNFW
    pr1_pru0_pru_r30_5      PRU0    C13     B3          O    MCASP0_FSR         BBB_BG        U6.46 -> M68K_BG    -         DNFW
    pr1_pru0_pru_r30_6      PRU0    D13     C3          O    MCASP0_AXR1        PRU_EWAIT     -                   U3.2      Repurposable
    pr1_pru0_pru_r30_7      PRU0    A14     C4          O    MCASP0_AHCLKX      PRU_ECLK      U6.30 -> M68K_E     -         DNFW
    pr1_pru0_pru_r30_12     PRU0    G17     B15              MMC0_CLK           GPIO_SUPE     U5.9  -> M68K_FC2   U3.13     Also A24
    pr1_pru0_pru_r30_13     PRU0    G18     B16              MMC0_CMD           GPIO_IACK     -                   U3.15     Also A25
    pr1_pru0_pru_r31_16     PRU0    D14     B4          I*   XDMA_EVENT_INTR1   PRU_VPA       U6.28 <- M68K_VPA   -         DNFW

    (GreenPAK)              -       -       -           O                       M68K_UDS      Header              U3.3      No direct MCU/PRU access             
    (GreenPAK)              -       -       -           O                       M68K_LDS      Header              U3.4      No direct MCU/PRU access         
    (GreenPAK)              -       -       -           O                       M68K_FC0      Header              U3.5      No direct MCU/PRU access        
    (GreenPAK)              -       -       -           O                       M68K_RW       Header              U3.6      No direct MCU/PRU access              
    (GreenPAK)              -       -       -           O                       M68K_DTACK    Header              U3.7      No direct MCU/PRU access                 
    (GreenPAK)              -       -       -           O                       M68K_FC1      Header              U3.10     No direct MCU/PRU access

    gpmc_csn2               GPMC    V9      P1          I    GPMC_CSN2          GPMC_ASN      U4.13 -> M68K_AS    U3.20     DNFW     
    gpmc_oen_ren            GPMC    T7      N1          I    GPMC_OEN           GPMC_OEN      -                   U3.19     Repurposable ???         
    gpmc_wen                GPMC    U6      N2          I    GPMC_WEN           GPMC_WEN      -                   U3.18     Repurposable ???        
    gpmc_be1n               GPMC    U18     N14         I    GPMC_BEN1          GPMC_BEN1     -                   U3.17     DNFW          
    gpmc_be0n_cle           GPMC    T6      N3          I    GPMC_BEN0_CLE      GPMC_BEN0     -                   U3.16     DNFW          
    gpmc_wait0              GPMC    T17     P15         I    GPMC_WAIT0         GPMC_WAIT0    -                   U3.12     DNFW           

    OEn is asserted on reads
    WEn is asserted on writes
    ASn is always asserted

    So I think we can omit using OEn. This is not a PRU signal though, it's only accessible through GPIO and thus very slow.

    Recommended rework:

    Make GPMC_OEN (U3.19) GPIO_IACK since this can be a slow signal and needs to be driven by the CPU anyway.
    Reuse PRU_EWAIT (U3.2) and GPIO_IACK (U3.15) for GreenPAK <-> PRU signalling

    For our work here, this is simple renaming IACK and and EWAIT to DTACK and WAIT0 respectively and
    ensuring we're using the correct bit offsets.

        AM335x                                                     68K BUS
   ---------------------------------.
    pr1_pru0_pru_r30_0              | -------------------------->  VMA     
    pr1_pru0_pru_r31_1              | <--------------------------  BGACK   
    pr1_pru0_pru_r31_2              | <--------------------------  RESET   
    pr1_pru0_pru_r31_3              | <--------------------------  BR
    pr1_pru0_pru_r31_4              | <--------------------------  CLK7    
    pr1_pru0_pru_r30_5              | -------------------------->  BG
    pr1_pru0_pru_r30_7              | -------------------------->  ECLK    
    pr1_pru0_pru_r31_16             | <--------------------------  VPA     
                                    |   .----------------------->  AS
                                    |   |    .-----------.    
                         GPMC_CSN2  | --o--> |20       3 | ----->  UDS    
                         GPMC_WEN   | -----> |18 Green 4 | ----->  LDS    
                         GPMC_OEN*  | -----> |19  PAK  5 | ----->  FC0     
                         GPMC_BEN1  | -----> |17       6 | ----->  R/W
                        GPMC_WAIT0  | <----- |12       7 | <-----  DTACK  
                         GPMC_BEN0  | -----> |16       10| ----->  FC1     
                                    |        |           |
    pr1_pru0_pru_r3x_6   PRU_EWAIT  | ------ | 2         | 1, 14 Vcc
    pr1_pru0_pru_r3x_13  GPIO_INST  | ------ |15         | 8, 9  I2C   
    pr1_pru0_pru_r3x_12  GPIO_SUPE  | --o--- |13         | 11    Ground
   ---------------------------------'   |    '-----------'
                                        '----------------------->  FC2
```

### Version History

v0.6 "Alpha" hardware release
- does not work

v0.7 "Beta" hardware release
- reversed bytes lanes to unideal (correct byte, incorrect word/long)
  
v0.8 "Beta 2" hardware
- fixed byte lanes to ideal (correct word, incorrect byte/long)
- fixed reset signal for power-on
- fixed GreenPAK pinout

### Status (Post-Mortem)

#### Incorrect Documentation
- UDSn and LDSn can occur at the same time as ASn in writes, this is, in fact, how the 68020 operates and the '020 is considered "bus compatible" to the 68000 (especially the 'EC020 used in the Amiga 1200); this greatly simplifies the logic we need between the GPMC and M68K bus
- The GPMC WAIT signal is edge sensitive and is only checked while inside of an active read or write cycle; additionally, pulsing the GPMC wait signal within a single GPMC clock cycle will cause it's internal state machine to also completely ignore it
- The GPMC WAIT signal cannot actually operate like a "READY" or DTACK signal unless we're in NAND mode which is incompatible with "static RAM" mode necessary for 68K bus operation despite how this is implied in the documentation

#### Actual Known Problems
- Buffee incorrectly maps the Function Code signals making it impossible to use these at all. This is technically fine for the Amiga as it does not use the FCx pins, but this would kill compatibility with most other 68000 systems. What's wrong specifically? The SUPE signal sets the FC2 bit and INST will set FC0 and FC1 to opposites but this doesn't allow for IACK. Conversely, if this pin is made to be IACK, then we cannot distinguish between instruction and data fetches; to fix we need both IACK and INST signals
- GPMC clocks are ONLY based on internal AM3358 and cannot be made to be synchronous to the 68K clock

#### Suspected Problems
- The GreenPAK is slow but not unusually, most CPLDs are in the > 10ns per gate territory; where the GreenPAK does add a lot of additional delay is with the inputs themselves (about 18ns simply at the input) and with the more complex logic devices such as counters (which add ~40ns overhead); some can be over-clocked but unreliably and inconsistently
- The PRU is very fast with a 5ns cycle time, but inputs are pipelined so a naive "echo pin" loop we're talking a 25ns propagation delay; additionally, the PRU is a RISC processor, not wired-logic, so the complexity of the code only worsens this delay and hitting ~35ns borders on catastrophic; on the bright-side, the PRU is known to over-clock and be very stable at 333MHz -- this comes with the cost of messing up the GPMC though since the use the same internal clock

## Proposed Fixes

There are these options in degree of change to the board.
1. Try and work with the existing design (fix the problem in software)
2. Respin the board to use the PRU more and GreenPAK less
3. Replace the GreenPAK with an iCE40




