This is based on how the MIPS data bus typically works, because it's reasonably simple and very flexible. Or at least how I think it works. This is also reminding me that I really need to refactor OCMIPS.

If there's something that sucks about it, please leave it in the comments and I can change it.

I'm pretty sure the ARM data bus works in a similar fashion. Probably the same for the 68000.

## MMIO bus

Components shall provide these three functions to architectures:

    void mmioWriteMask32(int addr, int mask, int data);
    int   mmioReadMask32(int addr, int mask);
    int  mmioGetBusWidth();

It is **REQUIRED** that every byte of `mask` provided by the architecture is either `0x00` or `0xFF`. Real hardware would use a byte enable.

It is **NOT REQUIRED** that the lower 2 bits of `addr` are `00`. However, they can be ignored. The mask can also be ignored. Architecture authors **MUST** pass the appropriate address to `addr` and mask to `mask`, even if it's unaligned.

### This may get chucked out like that section that was marked as "this may get chucked out"

Architectures should have access to a utilities API to convert addresses to masks:

    int getMask8(int addr);
    int getMask16(int addr);
    int getMask32(int addr);
    int getHiMask8(int addr);
    int getHiMask16(int addr);
    int getHiMask32(int addr);
    int getData8(int addr, int data);
    int getData16(int addr, int data);
    int getData32(int addr, int data);

These are completely optional and to be blunt very easy to implement yourself:

    return 0xFF<<((addr&3)*8);
    return 0xFFFF<<((addr&3)*8);
    return 0xFFFFFFFF<<((addr&3)*8);
    return 0x000000FF>>>((4-(addr&3))*8);
    return 0x0000FFFF>>>((4-(addr&3))*8);
    return 0xFFFFFFFF>>>((4-(addr&3))*8);
    return data<<((addr&3)*8);
    return data<<((addr&3)*8);
    return data<<((addr&3)*8);

And of course `getHiMask8` is completely useless, but provided for completeness.

### DMA

Components shall provide these functions to architectures:

    int dmaChannelCount();
    void dmaSetChannel(int cmp_chn, int arc_chn);
    void dmaAlert(int cmp_chn/dst_chn, int arc_chn/src_chn);
    boolean dmaWrite(int cmp_chn/dst_chn, int data, int size);

Architectures shall provide these functions to components:

    void dmaAlert(int arc_chn/dst_chn, int cmp_chn/src_chn);
    boolean dmaWrite(int arc_chn/dst_chn, int data, int size);

`dmaChannelCount` returns the number of channels this component has. This function may be unnecessary.

For `dmaSetChannel`, an `arc_chn` of `-1` disables DMA for that channel.

Either end can call `dmaAlert` when they are ready to receive data.

For `dmaWrite`, `size` is either `8`, `16`, or `32`. Returns `true` if the data was accepted. The recipient can ignore the size, but the sender **MUST** provide it.

It is good practice to keep writing to `dmaWrite` until it returns `false`.

If DMA is not supported by a component, this will be an acceptable implementation:

    public int dmaChannelCount() { return 0; }
    public void dmaSetChannel(int cmp_chn, int arc_chn) { }
    public void dmaAlert(int cmp_chn, int arc_chn) { }
    public boolean dmaWrite(int cmp_chn, int data, int size) { return false; }

If DMA is not supported by an architecture, this will be an acceptable implementation:

    public void dmaAlert(int arc_chn, int cmp_chn) { }
    public boolean dmaWrite(int arc_chn, int data, int size) { return false; }

A component **MUST NOT** attempt DMA on an architecture channel that was not granted to it.

With that said, an architecture **SHOULD** handle such a case without catching fire. Ignoring it is the best outcome. Unless your aim is to make an easily exploitable system.

## Interrupts

Components shall provide these two functions to architectures:

    int interruptPinCount();
    void interruptSetToken(int pin, int token);

Architectures shall provide this function to components:

    void interrupt(int token);

Interrupts from a component **MUST** only be fired from a valid token.

For `interruptSetToken`, `pin` refers to a pin on the component, not on the architecture. If `token` is -1, this interrupt is disabled.

Sidenote: If you are implementing a Z80 or 8080, the data that gets chucked on the data bus should be determined in the architecture implementation, not the component implementation.

## Component headers for Plug 'n' Play bus

NOTE: This may actually be dropped from the spec and implemented in an architecture-dependent way.

Each component has a 256-byte structure as follows:

    00 30 = zero-padded component address string (0x30 bytes)
    30 0C = ** RESERVED **
    3C 04 = CRC-32 of address 
    40 30 = zero-padded component type string (0x30 bytes)
    70 0C = ** RESERVED **
    7C 04 = CRC-32 of type 
    80 04 = HW API revision for this component (bump every time it changes, please!)
    84 04 = MMIO address space size shift amount (size in bytes = 1<<[0x80])
    88 04 = DMA channel count
    8C 04 = IRQ pin count
    90 70 = ** RESERVED **

CRC-32 is as per zlib (0xEDB88320 right-shift Galois LFSR) and does not include the padding in its calculation.

OpenComputers should handle the header, all a component needs to do is provide the relevant fields - this does not include the CRC-32s, which will be handled by OC itself.

A component **MUST NOT** provide an address or type with NUL bytes (`"\x00"` / 0x00) within it.

HW API revision is provided by this function of the component:

    int hwapiRevision();

## Endianness

The bus will be in little-endian.

If this is to be used by a big-endian architecture, it is up to the architecture to decide how this will even work. However, it will most likely require translating the mask field, and possibly the data field as well if the data value is not repeated over the bus.

Ideally the address **SHOULD** match the address fed to the component documentation. If this is the case, then the mask (and possibly the data) **MUST** be adjusted to suit.

## Open bus

`0xFFFFFFFF`. Please respect this convention. You can use `return -1;` for this.

