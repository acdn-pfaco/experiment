# How to prepare FRWY-LS1012A for secure boot

## References

- [Layerscape Linux Distribution POC User Guide - Rev. L6.1.1-1.0.0-23.08](../external/nxp/LLDPUG_RevL6.1.1_1.0.0_23.08.pdf)
- [QorIQ LS1012A Reference Manual - Rev. 3, 10/2020](../external/nxp/LS1012ARM.pdf)

## Pre-Requisities

- Unsecure image, for provisioning
- Secure image created with flex-builder
- Public key hash file (srk_hash.txt) generated from secure image build
- Generated OTPMK

## Useful information

Throughout this guide a few different addresses will be used.
Reference the table below (from [LS1012A Reference Manual](../external/nxp/LS1012ARM.pdf)) to check which registers are accessed.

| Address range         | Size  | Description                           |
|-----------------------|-------|---------------------------------------|
| 0x1E8_0000-0x1E8_FFFF | 64 KB | Security fuse processor (SFP)         |
| 0x1E9_0000-0x1E9_FFFF | 64 KB | SecMon                                |
| 0x1EE_0000            | 2 KB  | DCFG (SCRATCHWR offset at 0x1EE_0200) |


## Setup

- Put J37 to enable PROG_SFP
- Program unsecure image
- Boot and get access to u-boot shell

## Check Initial SNVS State and Value in SCRATCH Registers

```bash
md 0x1e90014 1
01e90014: 002b0088
md 0x1ee0200 1
01ee0200: 00000000
md 0x1e80024 1
01e80024: 00000000
```

## Write SRKH mirror registers

- Through I2C transactions, write to LDO1CT register to change LDO1EN bit in vr5100
    ```bash
    i2c mw 0x08 0x6c 0x10
    ```

- Assuming the following SRKH values were generated:
    ```bash
    cat srk_hash.txt
    SRK (Public Key) Hash:
    020c4869edefadaafc5e12cf651ebda8f2e08cd19db2db822f212a6dfa400560
        SFP SRKHR0 = 020c4869
        SFP SRKHR1 = edefadaa
        SFP SRKHR2 = fc5e12cf
        SFP SRKHR3 = 651ebda8
        SFP SRKHR4 = f2e08cd1
        SFP SRKHR5 = 9db2db82
        SFP SRKHR6 = 2f212a6d
        SFP SRKHR7 = fa400560
    ```

- Program the mirror registers with the following commands (**CAUTION:** note the endianess)
    ```bash
    mw.l 0x1e80254 0x69480c02
    mw.l 0x1e80258 0xaaadefed
    mw.l 0x1e8025c 0xcf125efc
    mw.l 0x1e80260 0xa8bd1e65
    mw.l 0x1e80264 0xd18ce0f2
    mw.l 0x1e80268 0x82dbb29d
    mw.l 0x1e8026c 0x6d2a212f
    mw.l 0x1e80270 0x600540fa
    ```

- Check if SRKH value is correct
    ```bash
    md 01e80254 8
    01e80254: 69480c02 aaadefed cf125efc a8bd1e65 ................ 
    01e80264: d18ce0f2 82dbb29d 6d2a212f 600540fa ................
    ```

## Write OTPMK registers

> **NOTE:** OTPMK is not used during secure boot, but it can be used by some algorithms of the secure engine, and must be programmed as part of the board preparation. At the present time we do not use it, so it's value is less critical.

- Verify that OTPMK is not already fused
    ```bash
    md $SNVS_HPSR_REG
    88000900
    ```

- Assuming the following OTPMK values were generated:
    ```bash
    OTPMK[255:0] is:
    8fb597cd571aecd5134ef16292b4fe5433a49a5ae1a0cf9d98b1d1975660e9ad

    NAME    |     BITS     |    VALUE  
    _________|______________|____________
    OTPMKR 0 | 255-224      |   8fb597cd 
    OTPMKR 1 | 223-192      |   571aecd5 
    OTPMKR 2 | 191-160      |   134ef162 
    OTPMKR 3 | 159-128      |   92b4fe54 
    OTPMKR 4 | 127- 96      |   33a49a5a 
    OTPMKR 5 |  95- 64      |   e1a0cf9d 
    OTPMKR 6 |  63- 32      |   98b1d197 
    OTPMKR 7 |  31-  0      |   5660e9ad
    ```

- Fuse OTPMK
    ```bash
    mw.l 0x1e80234 0xcd97b58f
    mw.l 0x1e80238 0xd5ec1a57
    mw.l 0x1e8023C 0x62f14e13
    mw.l 0x1e80240 0x54feb492
    mw.l 0x1e80244 0x5a9aa433
    mw.l 0x1e80248 0x9dcfa0e1
    mw.l 0x1e8024C 0x97d1b198
    mw.l 0x1e80250 0xade96056
    ```

- Verify that OTPMK is fused
    ```bash
    md $SNVS_HPSR_REG
    80000900
    ```

- Read OTPMK
    ```bash
    md 0x1e80234 8
    ```

> **NOTE:** OTPMK cannot be read in plain

## Burn fuses to permanently set SRKH and OTPMK

> **CAUTION:** Do not proceed to the steps in this topic until you are sure that OTPMK and SRKH are correctly fused, as explained in the topics above. **After the next step, fuses are burnt permanently, which cannot be undone.**

- Write SFP_INGR register
    ```bash
    mw 0x1e80020 0x2
    ```

## Check if procedured worked

```bash
md 0x1e90014 1
md 0x1ee0200 1
md 0x1e80024 1
```