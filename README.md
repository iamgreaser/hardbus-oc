Continuing the proposal from here: https://github.com/MightyPirates/OpenComputers/issues/1730

**See main.md for the main proposal.**

## Rationale

In order to do this:

    gpu.fill(1, 1, gpu_w, gpu_h, " ")

OCMIPS does a call that looks a bit like this:

    *(volatile const char **)0xBFF00280 = "fill";
    *(volatile int32_t *)0xBFF00300 = 1; *(volatile int32_t *)0xBFF00304 = 6;
    *(volatile int32_t *)0xBFF00308 = 1; *(volatile int32_t *)0xBFF0030C = 6;
    *(volatile int32_t *)0xBFF00310 = gpu_w; *(volatile int32_t *)0xBFF00314 = 6;
    *(volatile int32_t *)0xBFF00318 = gpu_h; *(volatile int32_t *)0xBFF0031C = 6;
    *(volatile const char **)0xBFF00320 = " "; *(volatile int32_t *)0xBFF00324 = 4;
    *(volatile uint8_t *)0xBFF00286 = 5;

Translated it's more like this:

    ocbus_cmd = "fill";
    ocbus_arg[0].val = 1; ocbus_arg[0].typ = OCTYP_INT;
    ocbus_arg[1].val = 1; ocbus_arg[1].typ = OCTYP_INT;
    ocbus_arg[2].val = gpu_w; ocbus_arg[2].typ = OCTYP_INT;
    ocbus_arg[3].val = gpu_h; ocbus_arg[3].typ = OCTYP_INT;
    ocbus_arg[4].val = " "; ocbus_arg[4].typ = OCTYP_STR;
    ocbus_strobe_call_function = 5;

Not pictured: the code required to find the gpu, starring strncmp and memcpy.

Basically it's a pain in the arse to work with, from both the MIPS end and the Java end, and it gets even worse when one of the things that get returned code is a floating point number and you don't have floating point support available because, for a *totally* made-up example, you're writing code for the Linux kernel. (Project shelved for the time being, but that screenshot does make people think you're pulling their leg seeing as it's April Fools Day somewhere in the world. And yes, it still builds.)

It would be a lot more sensible to do something like this:

    ocdev_gpu->rx = 1;
    ocdev_gpu->ry = 1;
    ocdev_gpu->rw = gpu_w;
    ocdev_gpu->rh = gpu_h;
    ocdev_gpu->chr = 0x0020; // space
    ocdev_gpu->strobe_imm = OCGPU_FILL;

These are of course volatile, uncached hardware registers.

Which is fine for immediate, non-DMA data. A "set" command, on the other hand, does need DMA for maximum efficiency, but a non-DMA "setch" command would work just fine for architectures that don't have DMA at the time, especially if it autoincrements dx.

In the current GPU hardbus API proposal, it will look a bit more like this:

    gpu_send_command(GPU_SET_RXY(1, 1));
    gpu_send_command(GPU_SET_RWH(gpu_w, gpu_h));
    gpu_send_command(GPU_FILL(' '));

Or more precisely:

    gpu_send_command(0x42001001);
    gpu_send_command(0x43000000 | (gpu_w&0xFFF) | ((gpu_h&0xFFF)<<12));
    gpu_send_command(0x60000020);

Where `gpu_send_command` checks if the FIFO is empty before sending. Using a function here means that we can replace it with a function that builds up a buffer for sending commands via DMA.

