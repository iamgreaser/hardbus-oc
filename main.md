This is based on how the MIPS data bus typically works, because it's reasonably simple and very flexible. Or at least how I think it works.

If there's something that sucks about it, please leave it in the comments and I can change it.

I'm pretty sure the ARM data bus works in a similar fashion. Probably the same for the 68000.

## MMIO bus

Components shall provide these three methods to architectures:

    void mmioWriteBus32(int addr, int mask, int data);
    int  mmioReadBus32(int addr, int mask);
    int  mmioGetAddressWidth();

It is **REQUIRED** that every byte of `mask` provided by the architecture is either `0x00` or `0xFF`. Real hardware would use a byte enable.

It is **NOT REQUIRED** that the lower 2 bits of `addr` are `00`. However, they can be ignored. The mask can also be ignored. Architecture authors **MUST** pass the appropriate address to `addr` and mask to `mask`, even if it's unaligned.

## DMA

The DMAChannel interface provides these methods:

    void dmaAlert(DMAChannel recip_chn);
    boolean dmaWrite(int data, int size);
    void dmaRelease();

Architecture and component code should implement the DMAChannel interface, preferably as separate classes.

Components shall provide these methods to architectures:

    int dmaChannelCount();
    DMAChannel dmaSetChannel(int cmp_chn, DMAChannel arc_chn);

`dmaChannelCount` returns the number of channels this component has.

For `dmaSetChannel`, if `arc_chn` is `null`, the channel is released. If this function returns `null`, the channel binding failed. Only one DMA channel binding can be active at any one time.

When either end wants to release a DMA channel, they call `dmaRelease()` on the DMA channel object. Both ends then **MUST NOT** continue to use this channel, although further `dmaRelease()` calls are permitted. Only one end needs to call this function. This function is called by `dmaSetChannel()` automatically.

Either end can call `dmaAlert` when they are ready to receive data.

For `dmaWrite`, `size` is either `8`, `16`, or `32`. Returns `true` if the data was accepted. The recipient can ignore the size, but the sender **MUST** provide it.

It is good practice to keep writing to `dmaWrite` until it returns `false`.

If DMA is not supported by a component, this will be an acceptable implementation:

    public int dmaChannelCount() { return 0; }
    public DMAChannel dmaSetChannel(int cmp_chn, DMAChannel arc_chn) { return null; }

A component **MUST NOT** attempt DMA on a DMAChannel that has since been unlinked.

With that said, an architecture **SHOULD** handle such a case without catching fire. Ignoring it is the best outcome. Unless your aim is to make an easily exploitable system.

## Interrupts

Components shall provide these two methods to architectures:

    int interruptPinCount();
    void interruptSetPin(int cmp_pin, IRQPin irq);

Architecture code should implement the IRQPin interface, preferably as a separate class.

The IRQPin interface provides these methods:

    void interruptFire(boolean state);
    void interruptRelease();

Interrupts from a component **MUST** only be fired from a valid `arc_pin`.

For `interruptSetPin`, `cmp_pin` refers to a pin on the component, not on the architecture. Only one IRQ pin binding can be active at any one time.

When the component wants to release an IRQ pin, it calls `interruptRelease()` on the IRQ pin object. Both ends then **MUST NOT** continue to use this channel, although further `interruptRelease()` calls are permitted. This function is called by `interruptSetPin()` automatically.

When the architecture wants to release an IRQ pin, it calls `interruptSetPin(cmp_pin, *)` with either a different IRQ pin, or `null`.

When an IRQ pin is released, the state of the pin **MUST** be set to `false`.

For `interruptFire`, `state` indicates the state of the interrupt pin. Remember to send `interruptFire(arc_pin, false);` when your interrupt is acknowledged.

Sidenote: If you are implementing a Z80 or 8080, the data that gets chucked on the data bus should be determined in the architecture implementation, not the component implementation. For vectored interrupts, you will ideally want to implement this as an `arc_pin` value, even though strictly speaking they all use the same physical interrupt pin.

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

HW API revision is provided by this method of the component:

    int hwapiRevision();

## Endianness

The bus will be in little-endian.

If this is to be used by a big-endian architecture, it is up to the architecture to decide how this will even work. However, it will most likely require translating the mask field, and possibly the data field as well if the data value is not repeated over the bus.

Ideally the address **SHOULD** match the address fed to the component documentation. If this is the case, then the mask (and possibly the data) **MUST** be adjusted to suit.

## Bus contention

Components shall provide these methods to architectures:

    boolean hardbusIsFree();
    boolean hardbusTryClaim(HardbusHandle hdl);
    void hardbusForceClaim(HardbusHandle hdl);
    boolean hardbusRelease(HardbusHandle hdl);

The HardbusHandle interface implements this method:

    boolean hardbusTryRelease();
    void hardbusForceRelease();

Architecture code should implement the HardbusHandle interface, preferably as a separate class.

For architectures, it is recommended that `hardbusTryClaim()` is used over `hardbusForceClaim()`.

An architecture **MUST NOT** claim the bus automatically for anything it isn't using.

An architecture **MUST** release all of its handles on shutdown. This facility may need to be provided by OpenComputers itself.

`hardbusTryClaim()` **MUST** return `false` if the bus is already in use and a call to `hardbusTryRelease()` on the current HardbusHandle returns `false`.

Both `hardbusRelease()` and `hardbusForceClaim()` release all DMA channels and IRQ pins, and call the `hardbusRelease()` method of the HardbusHandle.

An architecture **MUST NOT** use `hardbusRelease()` unless it holds the bus. If kicking out a bus master is necessary, it should use `hardbusForceClaim()`.

An architecture **MUST NOT** call its own HardbusHandle methods - these are **ONLY** to be used by the component. If the architecture wishes to release the Hardbus, it **MUST** call `hardbusRelease()` on the component.

While an architecture is not required to claim the bus before use, it is recommended. Trivial queries over the MMIO bus should be OK. DMA and IRQ usage or anything that affects state **SHOULD** claim the bus.

## Open bus

`0xFFFFFFFF`. Please respect this convention. You can use `return -1;` for this.

## Summary of interface methods

### HardbusComponent

    void mmioWriteBus32(int addr, int mask, int data);
    int mmioReadBus32(int addr, int mask);
    int mmioGetAddressWidth();
    int dmaChannelCount();
    DMAChannel dmaSetChannel(int cmp_chn, DMAChannel arc_chn);
    int interruptPinCount();
    void interruptSetPin(int cmp_pin, IRQPin irq);
    int hwapiRevision();
    boolean hardbusIsFree();
    boolean hardbusTryClaim(HardbusHandle hdl);
    void hardbusForceClaim(HardbusHandle hdl);
    boolean hardbusRelease(HardbusHandle hdl);

### DMAChannel

    void dmaAlert(DMAChannel recip_chn);
    boolean dmaWrite(int data, int size);
    void dmaRelease();

### IRQPin

    void interruptFire(boolean state);
    void interruptRelease();

### HardbusHandle

    boolean hardbusTryRelease();
    void hardbusForceRelease();

