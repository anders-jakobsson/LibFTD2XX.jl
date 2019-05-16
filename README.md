# LibFTD2XX

[![Coverage Status](https://coveralls.io/repos/github/Gowerlabs/LibFTD2XX.jl/badge.svg?branch=master)](https://coveralls.io/github/Gowerlabs/LibFTD2XX.jl?branch=master)
[![codecov](https://codecov.io/gh/Gowerlabs/LibFTD2XX.jl/branch/master/graph/badge.svg)](https://codecov.io/gh/Gowerlabs/LibFTD2XX.jl)
[![Build Status](https://travis-ci.org/Gowerlabs/LibFTD2XX.jl.svg?branch=master)](https://travis-ci.org/Gowerlabs/LibFTD2XX.jl)
[![Build status](https://ci.appveyor.com/api/projects/status/ui8plnih785lw4jg/branch/master?svg=true)](https://ci.appveyor.com/project/samuelpowell/libftd2xx-jl/branch/master)

Julia wrapper for FTD2XX driver. For reference see the [D2XX Programmer's Guide (FT_000071)](http://www.ftdichip.com/Support/Documents/ProgramGuides/D2XX_Programmer's_Guide(FT_000071).pdf).

Contains methods and functions for interacting with D2XX devices. Most 
cross-platform functions are supported.

Direct access to functions detailed in the [D2XX Programmer's Guide (FT_000071)](http://www.ftdichip.com/Support/Documents/ProgramGuides/D2XX_Programmer's_Guide(FT_000071).pdf)
are available in in the submodule `Wrapper`.


Supports Julia v0.7 and above, on Windows (32- and 64-bit), Linux (64-bit, ARMv7, ARMv8), and MacOS.

## Example Code

The below is a demonstration for a port running at 2MBaud which echos what it receives.

```Julia
julia> using LibFTD2XX

julia> # Finding and configuring devices

julia> devices = D2XXDevices()
4-element Array{D2XXDevice,1}:
 D2XXDevice(0, 2, 7, 67330065, 0, "FT3V1RFFA", "USB <-> Serial Converter A", Base.RefValue{FT_HANDLE}(FT_HANDLE(Ptr{Nothing} @0x0000000000000000)))
 D2XXDevice(1, 2, 7, 67330065, 0, "FT3V1RFFB", "USB <-> Serial Converter B", Base.RefValue{FT_HANDLE}(FT_HANDLE(Ptr{Nothing} @0x0000000000000000)))
 D2XXDevice(2, 2, 7, 67330065, 0, "FT3V1RFFC", "USB <-> Serial Converter C", Base.RefValue{FT_HANDLE}(FT_HANDLE(Ptr{Nothing} @0x0000000000000000)))
 D2XXDevice(3, 2, 7, 67330065, 0, "FT3V1RFFD", "USB <-> Serial Converter D", Base.RefValue{FT_HANDLE}(FT_HANDLE(Ptr{Nothing} @0x0000000000000000)))

julia> isopen.(devices)
4-element BitArray{1}:
 false
 false
 false
 false

julia> device = devices[1]
D2XXDevice(0, 2, 7, 67330065, 0, "FT3V1RFFA", "USB <-> Serial Converter A", Base.RefValue{FT_HANDLE}(FT_HANDLE(Ptr{Nothing} @0x0000000000000000)))

julia> open(device)

julia> isopen(device)
true

julia> datacharacteristics(device, wordlength = BITS_8, stopbits = STOP_BITS_1, parity = PARITY_NONE)

julia> baudrate(device,2000000)

julia> timeout_read, timeout_wr = 200, 10; # milliseconds

julia> timeouts(device, timeout_read, timeout_wr)

julia> # Basic IO commands

julia> supertype(typeof(device))
IO

julia> write(device, Vector{UInt8}(codeunits("Hello")))
0x00000005

julia> bytesavailable(device)
0x00000005

julia> String(read(device, 5)) # read 5 bytes
"Hello"

julia> write(device, Vector{UInt8}(codeunits("World")))
0x00000005

julia> String(readavailable(device)) # read all available bytes
"World"

julia> write(device, Vector{UInt8}(codeunits("I will be deleted.")))
0x00000012

julia> bytesavailable(device)
0x00000012

julia> flush(device)

julia> bytesavailable(device)
0x00000000

julia> # Read Timeout behaviour

julia> tread = 1000 * @elapsed read(device, 5000) # nothing to read! Will timeout...
203.20976900000002

julia> timeout_read < 1.5*tread # 1.5*tread to allow for extra compile/run time.
true

julia> # Write Timeout behaviour (only tested on windows)

julia> buffer = zeros(UInt8, 5000);

julia> twr = 1000 * @elapsed nb = write(device, buffer) # Will timeout before finishing write!
22.997304

julia> timeout_wr < 1.5*twr
true

julia> nb # doesn't correctly report number written
0x00000000

julia> Int(bytesavailable(device))
3584

julia> timeout_wr < 1.5*twr
true

julia> flush(device)

julia> timeout_wr = 1000; # increase write timeout

julia> timeouts(device, timeout_read, timeout_wr)

julia> twr = 1000 * @elapsed nb = write(device, buffer) # Won't timeout before finishing write!
15.960230999999999

julia> nb # correctly reports number written
0x00001388

julia> Int(bytesavailable(device))
5000

julia> timeout_wr < 1.5*twr
false

julia> close(device)

julia> isopen(device)
false

```

## Linux support

It is likely that the kernel will automatically load VCP drivers when running on linux, which will prevent the D2XX drivers from accessing the device. Follow the guidance in the FTDI Linux driver [README](https://www.ftdichip.com/Drivers/D2XX/Linux/ReadMe-linux.txt) to unload the `ftdio_sio` and `usbserial` kernel modules before use. These can optionally be blacklisted if appropriate.

The D2XX drivers use raw USB access through `libusb` which may not be available to non-root users. A udev file is required to enable access to a specified group. A script to create the appropriate file and user group is available, e.g., [here](https://stackoverflow.com/questions/13419691/accessing-a-usb-device-with-libusb-1-0-as-a-non-root-user).
