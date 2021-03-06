## D4.2.4 Translation tables and the translation process

[`中文版`](../../zh/chapter_d4/d42_4_translation_tables_and_the_translation_proces.html)

The following subsections describe general properties of the translation tables and translation table walks, that are largely independent of the translation table format:
* [Translation table walks](#).
* [Security state of translation table lookups on page D4-1658](#).
* [Control of translation table walks on page D4-1658](#).
* [Security state of translation table lookups on page D4-1658](#).

See also [Selection between TTBR0 and TTBR1 on page D4-1670](#).

### Translation table walks

A translation table walk comprises one or more translation table lookups. The translation table walk is the set of lookups that are required to translate the virtual address to the physical address. For the Non-secure EL1&0 translation regime, this set includes lookups for both the stage 1 translation and the stage 2 translation. The information returned by a successful translation table walk is:
* The required physical address. If the access is from Secure state this includes identifying whether the access is to the Secure physical address space or the Non-secure physical address space, see [Security state of translation table lookups on page D4-1658](#).
* The memory attributes for the target memory region, as described in [Memory types and attributes on page B2-93](#). For more information about how the translation table descriptors specify these attributes see [Memory region attributes on page D4-1712](#).
* The access permissions for the target memory regions. For more information about how the translation table descriptors specify these permissions see Memory access control on page D4-1704.

The translation table walk starts with a read of the translation table for the initial lookup. The TTBR for the stage of translation holds the base address of this table. Each translation table lookup returns a descriptor, that indicates one of the following:

* The entry is the final entry of the walk. In this case, the entry contains the OA, and the permissions and attributes for the access.
* An additional level of lookup is required. In this case, the entry contains the translation table base address for that lookup. In addition:
    - The descriptor provides hierarchical attributes that are applied to the final translation, see Hierarchical control of Secure or Non-secure memory accesses on page D4-1703 and Hierarchical control of data access permissions on page D4-1706.
    - If the translation is in a Secure translation regime, the descriptor indicates whether that base address is in the Secure or Non-secure address space, unless a hierarchical control at a previous level of lookup has indicated that it must be in the Non-secure address space.
* The descriptor is invalid. In this case, the memory access generates a Translation fault.

Figure D4-7 on page D4-1657 gives a generalized view of a single stage of address translation, where three levels of lookup are required.

![](figure_d4_7.png)

ARM DDI 0487A.g ID070815
A translation table lookup from VMSAv8-64 performs a single-copy atomic 64-bit access to the translation table entry. This means the translation table entry is treated as a 64-bit object for the purpose of endianness. SCTLR.EE determines the endianness of the translation table lookups.

> **NOTE:**
**Dynamically changing translation table endianness**
Because any change to an SCTLR.EE, bit requires synchronization before it is visible to subsequent operations, ARM strongly recommends that any EE bit is changed only when either:  
* Executing at an Exception level that does not use the translation tables affected by the EE bit being changed.
* Executing with address translation disabled for any stage of translation affected by the EE bit being changed.
Address translation stages are disabled by setting an SCTLR.M bit to 0. See the appropriate register description for more information.

The appropriate TTBR holds the output address of the base of the translation table used for the initial lookup, and:
* For all address translation stages other than Non-secure EL1&0 stage 1 translations, the output address held in the TTBR, and any translation table base address returned by a translation table descriptor, is the PA of the base of the translation table.
* For Non-secure EL1&0 stage 1 translations, the output address held in the TTBR, and any translation table base address returned by a translation table descriptor, is the IPA of the base of the translation table. This means that if stage 2 address translation is enabled, each of these OAs is subject to second stage translation.

    > **NOTE:**
    TLB caching can be used to minimise the number of translation table lookups that must be performed. Because each stage 1 OA generated during a translation table walk is subject to a stage 2 translation, if the caching of translation table entries is ineffective, a VA to PA address translation with two stages of translation can give rise to multiple translation table lookups. The number of lookups required is given by the following equation:
    
    > (S1+1)*(S2+1) - 1
    
    > Where, for the Non-secure EL1&0 translation regime, S1 is the number of levels of lookup required for a stage 1 translation, and S2 is the number of levels of lookup required for a stage 2 translation.

The TTBR also determines the memory cacheability and shareability attributes that apply, for that stage of translation, to all translation table lookups generated by that stage of translation.
The Normal memory type is the memory type defined for a translation table lookup for a stage of translation.

> **NOTE:**
* In a two stage translation system, a translation table lookup from stage 1, that has the Normal memory type defined at stage 1 by this rule, can still be given the Device memory type as part of the stage 2 translation of that address. ARM strongly recommends against such a remapping of the memory type, and the architecture includes a trap of this behavior to EL2. For more information, see Stage 2 fault on a stage 1 translation table walk on page D4-1726.
* The rules about mismatched attributes given in Mismatched memory attributes on page B2-104 apply to the relationship between translation table walks and explicit memory accesses to the translation tables in the same way that they apply to the relationship between different explicit memory accesses to the same location. For this reason, ARM strongly recommends that the attributes that the TTBR applies to the translation tables are the same as the attributes that are applied for explicit accesses to the memory that holds the translation tables.


For more information see Overview of the VMSAv8-64 address translation stages[
](#). See also [Selection between TTBR0 and TTBR1 on page D4-1670](#).


#### Security state of translation table lookups

For a Non-secure translation regime, all translation table lookups are performed to Non-secure output addresses.

For a Secure translation regimes, the initial translation table lookup is performed to a Secure output address.

If the translation table descriptor returned as a result of that initial lookup points to a second translation table, then the NSTable bit in that descriptor determines whether that translation table lookup is made to Secure or to Non-secure output addresses.

This applies for all subsequent translation table lookups as part of that translation table walk, with the additional rule that any translation table descriptor that is returned from Non-secure memory is treated as if the NSTable bit in that descriptor indicates that the subsequent translation table lookup is to Non-secure memory.

#### Control of translation table walks

For the first stage of the EL1&0 translation regime, the TCR_EL1.{EPD0, EPD1} bits determine whether the translation tables for that regime are valid. EPD0 indicates whether the table that TTBR0_EL1 points to is valid, and EPD1 indicates whether the table that TTBR1_EL1 points to is valid. The effect of these bits is:

| | |
| -- | -- |
| EPDn == 0 | The translation table is valid, and can be used for a translation table lookup. |
| EPDn == 1 | If a TLB miss occurs based on TTBRn, a Translation fault is returned, and no translation table walk is performed. The fault is reported as a level 0 fault. |

