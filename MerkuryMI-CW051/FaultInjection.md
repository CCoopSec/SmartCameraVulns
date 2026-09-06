# CVE-2026-xxxx

**CVE ID:** Pending

**Problem Type:** CWE-1191 (On-Chip Debug and Test Interface With Improper Access Control) & CWE-1332 (Improper Handling of Faults that Lead to Instruction Skip)

**Vendor:** Merkury Innovations

**Affected Product:** Merkury MI-CW051 IP Camera

**Product Page:** https://support.merkurysmart.com/hc/en-us/sections/15793498596507-CW051-Indoor-Smart-Camera

**Firmware Version:** 3.0.0.086

**Researcher:** Chase Cooper

**Summary**
The Merkury MI-CW051 IP Camera exposes active diagnostic hardware interfaces (UART) and unprotected SPI flash memory traces on its PCB. Due to a lack of physical obfuscation and hardware-level anti-tamper mechanisms, an attacker with physical access can execute a timing-based fault injection attack against the SPI flash chip. This physical disruption targets U-Boot during the boot sequence, forcing a read error that interrupts the kernel load process. The bootloader bypasses normal access controls and bootdelay restrictions, dropping the attacker into an unrestricted U-Boot menu console.

**A quick note**
The Merkury CW051 boot sequence contains a U-Boot environment with a bootdelay=1 configuration. Sending a keystroke during the
boot delay interrupted the automatic startup and provided access to the bootloader menu console. The bootargs environment variable was subsequently modified to change the kernel initialization behavior. Appending `init=/bin/sh` caused the system to initialize a shell instead of its normal init process, resulting in privileged shell access.

**Root Cause**
The SoC does not implement a hardware-backed Secure Boot chain and fails to properly handle physical read disruptions from the SPI NOR Flash (ZBIT ZB25VQ64). When memory access is interrupted, the bootloader defaults to an insecure, open console state rather than halting execution.

<img width="975" height="930" alt="image" src="https://github.com/user-attachments/assets/8a42d20f-66bf-460f-b557-8328c430163e" />

**Impact**
Dropping into the unrestricted U-Boot menu allows for total compromise of the host operating system. An attacker can persistently modify the `bootargs` environment variable (e.g., appending `init=/bin/sh`) to bypass standard initialization, install persistent backdoors in the startup scripts, or directly extract the entire firmware image over the serial connection.

**Reproduction (PoC)**

**Prerequisites:**

- Merkury MI-CW051 IP Camera.
- Multi-protocol interface board (e.g., Bus Pirate 5).
- Physical jumper wire.

**Steps:**

1. **PCB Enumeration:** Disassemble the camera chassis to expose the motherboard. Locate the active traces for the UART interface and the 8-pin ZBIT ZB25VQ64 SPI flash package. Identify the Data Out (DO) pin and the nearest Ground (GND) pin on the flash chip.
2. **Serial Connection:** Attach the Bus Pirate to the identified UART TX/RX pins. Set the baud rate to `115200` (8 data bits, no parity, 1 stop bit) and apply power to the camera to monitor the U-Boot initialization sequence.
3. **Fault Injection (Glitching):** Monitor the serial console output. Immediately after the first Auto Negotiation phase establishes U-Boot (look for the initial hardware initialization strings), but strictly before the second phase begins reading the kernel from flash memory, introduce a physical short by bridging the flash chip's DO pin to GND with the tweezers.
4. **Triggering the Bypass:** Hold the short for approximately 1-2 seconds until the serial console outputs a read error. Release the short.


The targeted disruption forces U-Boot to fail its read operation, bypasses the standard boot sequence, and defaults directly into an unrestricted bootloader menu console.

**Recommended Mitigation**

- **Debug Port Disablement:** Permanently burn out or disable UART/JTAG interfaces in production hardware builds to prevent raw console access.
- **Hardware-Backed Secure Boot:** Implement a hardware-backed Secure Boot root of trust. The SoC must cryptographically verify the digital signature of the bootloader and kernel; if the read fails, the system must halt completely.
- **Hardware Hardening:** Transition from exposed-pin IC packages (e.g., SOIC-8) to Ball Grid Array (BGA) packaging, or apply opaque epoxy potting over the SoC and Flash memory to severely complicate physical fault injection.    
