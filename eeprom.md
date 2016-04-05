Status: Early draft

## I/O map

    00 B SS data
    04 B -S command(&)
    04 B R- status_flags
    08 I R- code_capacity
    0C I R- data_capacity

## DMA map

* 0 = main DMA

## Commands

    0x01: Reset
    0x20: Read code via MMIO
    0x21: Read data via MMIO
    0x22: Read label via MMIO
    0x28: Read code via DMA
    0x29: Read data via DMA
    0x2A: Read label via DMA
    0x30: Write code via MMIO
    0x31: Write data via MMIO
    0x32: Write label via MMIO
    0x38: Write code via DMA
    0x39: Write data via DMA
    0x3A: Write label via DMA

## Status flags

    00000000
    ....||||
    ....|||`- bit  0 = ready to accept new command
    ....||`-- bit  1 = currently processing command
    ....|`--- bit  2 = data ready to read
    ....`---- bit  3 = ready to accept data

### Effects these bits have

* Bit 0 clear: A "reset" command will cancel an operation.
  * **TODO: define what happens when other commands fire** - I'm not sure if they should cancel+override the current operation or be ignored.
* Bit 1 set: Any writes to `command` will be ignored.
* Bit 2 clear: Reads from `data` will return **UNDEFINED** results.
* Bit 3 clear: Writes to `data` will be ignored.
* Bit 2 set: Reads from `data` will return the last value received and continue the operation.
* Bit 3 set: Writes to `data` will continue the operation with the provided data.

## TODO

* `getChecksum`. To make this more realistic I'd prefer this to be done as a command.
* `makeReadonly`.
* Some basic method of locking/unlocking the chip for writing.
* Some method of determining if the chip is read-only.

