Status: Early draft, does need improvement

## I/O map

    00 I R- maxw
    04 I R- maxh
    08 I R- curw
    0C I R- curh
    
    10 I -S command_queue(&)
    14 I -S immediate_command(&)
    18 I RS command_dma_len - in 32-bit words - command DMA activates when you write the uppermost byte of this
    1C I R- status_flags
    
    20 I RW fg_col - top bit, if set, means indexed, otherwise it's RGB
    24 I RW bg_col - top bit, if set, means indexed, otherwise it's RGB
    28 B RW pal_idx
    2C I sS pal_val
    
    30 I R- max_depth
    34 I RS cur_depth
    38 - -- ### RESERVED ###
    3C - -- ### RESERVED ###
    
    40 I -S get_xy_query (0x00YYYXXX)
    44 I R- xy_query_char
    48 I R- xy_query_fg (upper 8 bits are either the index or 0xFF)
    4C I R- xy_query_bg (upper 8 bits are either the index or 0xFF)

## Status flags

    00000000 00000000 00000000 00000000
    ........ .....||| .||||||| `""""""|
    ........ .....||| .||||||| .......`-    0-7 = free space left in command queue
    ........ .....||| .||||||`---------- bit  8 = screen currently bound
    ........ .....||| .|||||`----------- bit  9 = last screen bind successful
    ........ .....||| .||||`------------ bit 10 = command queue empty
    ........ .....||| .|||`------------- bit 11 = command queue full
    ........ .....||| .||`-------------- bit 12 = command queue ready for new command
    ........ .....||| .|`--------------- bit 13 = command queue overflow
    ........ .....||| .`---------------- bit 14 = IRQ0 active
    ........ .....||| ........ ........
    ........ .....||`------------------- bit 16 = DMA0 awaiting read from CPU
    ........ .....|`-------------------- bit 17 = DMA1 awaiting read from CPU
    ........ .....`--------------------- bit 18 = DMA2 awaiting write to CPU

## Overflow behaviour

If the command queue buffer overflows, the command queue overflow bit will be set.

The command queue will be stalled.

You must then clear the command queue - this will clear the bit and let the command queue run again.

**TODO:** Find a good command queue size. 128 might make sense.

## DMA map

* 0 = command data
* 1 = strings from CPU
* 2 = strings to CPU

## IRQ map

* 0 = queue signal IRQ

## Command queue ops

Ideally we want this to cover all the non-query stuff.

However, one query is provided which involves reading a string.

    0x41hhhwww = set rect w/h
    0x42yyyxxx = set rect x/y
    0x43yyyxxx = set delta x/y (sign-extended)
    
    0x45hhhwww = set screen w/h
    0x46hhhwww = set screen pan w/h
    0x47yyyxxx = set screen pan x/y
    
    0x50rrggbb = set FG direct
    0x51rrggbb = set BG direct
    0x520000ii = set FG indexed
    0x530000ii = set BG indexed
    
    0x580000ii
    0x00rrggbb = set palette colour
    
    0x6000cccc = fill with char
    0x6100cccc = set char, autoadvancing rx
    0x6200cccc = set char, autoadvancing ry
    0x6300cccc = set char, leaving rx, ry alone
    
    0x6800ssss = set string horizontal 16-bit (DMA1)
    0x6900ssss = set string horizontal UTF-8 (DMA1)
    0x6A00ssss = set string vertical 16-bit (DMA1)
    0x6B00ssss = set string vertical UTF-8 (DMA1)
    ^ s = size in transfer units, for UTF-8 these are 8 bits, for 16-bit these are 16 bits
    
    0x6E010000 = copy block
    
    0x6EFE0000 = signal IRQ0
    
    0x700100ss
    0x--------
    ..........
    0x-------- = bind screen address (s = size in bytes, padded upwards to fit)
    
    0x720100ss = DMA bind screen address (DMA1, s = size of buffer in bytes, uses byte mode)
    0x730100ss = get screen address (DMA2, s = size of buffer in bytes, padded with 0s, uses byte mode)

## Immediate command ops

    0x01000000 = clear command queue
    0x02000000 = acknowledge IRQ0

## Examples
### Direct mode

    gpu->immediate_command = GPU_IMM_CLEAR_QUEUE();
    gpu->command_queue = GPU_SET_RWH(gpu->wndw, gpu->wndh);
    gpu->command_queue = GPU_SET_RXY(gpu->wndx, gpu->wndy);
    gpu->command_queue = GPU_SET_BG_DIRECT(0x0000FF);
    gpu->command_queue = GPU_SET_FG_DIRECT(0xFFFFFF);
    gpu->command_queue = GPU_FILL(0x0020);
    dma_set_pointer(DMA_GPU_STRING, bsod_message);
    gpu->command_queue = GPU_SET_STRING_UTF8_HORIZONTAL(strlen(bsod_message));

### Command DMA mode

    dmatail = dmahead;
    *(dmatail++) = GPU_SET_RWH(gpu->wndw, gpu->wndh);
    *(dmatail++) = GPU_SET_RXY(gpu->wndx, gpu->wndy);
    *(dmatail++) = GPU_SET_BG_DIRECT(0x0000FF);
    *(dmatail++) = GPU_SET_FG_DIRECT(0xFFFFFF);
    *(dmatail++) = GPU_FILL(0x0020);
    dma_set_pointer(DMA_GPU_STRING, bsod_message);
    *(dmatail++) = GPU_SET_STRING_UTF8_HORIZONTAL(strlen(bsod_message));
    dma_set_pointer(DMA_GPU_COMMAND, dmahead);
    gpu->immediate_command = GPU_IMM_CLEAR_QUEUE();
    gpu->command_dma_len = dmatail - dmahead;

