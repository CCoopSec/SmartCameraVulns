# CVE-2026-xxxx

**CVE ID:** Pending 

**Problem Type:** CWE-798 (Use of Hard-coded Credentials), CWE-312 (Cleartext Storage of Sensitive Information)

**Vendor:** Merkury Innovations

**Affected Product:** Merkury MI-CW051 IP Camera

**Product Page:** https://support.merkurysmart.com/hc/en-us/sections/15793498596507-CW051-Indoor-Smart-Camera

**Firmware Version:** 3.0.0.086

**Researcher:** Chase Cooper

#### Summary
The Merkury MI-CW051 IP Camera exposes highly sensitive cryptographic materials and user network credentials across both its writable filesystems and compiled vendor binaries. Physical memory dumping reveals that mTLS RSA private keys, X.509 client certificates, and user Wi-Fi credentials are saved in cleartext within writable JFFS2 partitions. Additionally, static analysis of proprietary executables contained within read-only SquashFS partitions reveals compiled, hardcoded RSA keys and certificates embedded directly inside binary code.

#### Root Cause
The device lacks storage partition encryption at rest and fails to utilize a hardware secure enclave or Trusted Execution Environment (TEE). Secrets required for network authentication are maintained in a cleartext JSON structure on flash memory, while critical cloud communication binaries (such as `/pepper/pepper_app`) statically compile cryptographic keys directly into executable code segments.

#### Impact
An attacker who extracts these cryptographic assets can bypass implemented certificate pinning defenses completely. Because the X.509 certificates and private keys bind the camera to the Pepper IoT / Smart Home Ventures cloud infrastructure, an attacker can impersonate target hardware devices, forge backend requests, or silently intercept and decrypt HTTPS and MQTT telemetry traffic. Furthermore, extracting cleartext local wireless credentials compromises the security of the host Wi-Fi network.

#### Reproduction (PoC)

**Prerequisites:**
- Merkury MI-CW051 IP Camera.
- Soldering station and Universal Programmer (e.g., XGecu T48).
- Reverse engineering & extraction suite (`binwalk`, `strings`, Ghidra).

**Steps:**
1. **Hardware Extraction:** Disassemble the camera chassis and desolder the 8MB SPI NOR Flash chip (ZBIT ZB25VQ64) from the PCB using hot air rework.
2. **Memory Dumping:** Insert the chip into the universal programmer, configure the software for the ZB25VQ64 parameters, and execute a sequential read to save the raw memory binary (`FW.bin`).
3. **Filesystem Extraction:** Run `binwalk -e FW.bin` to unpack all embedded partitions, including the writable JFFS2 volumes (`0x3B3000` and `0x3CFA7C`) and read-only SquashFS volumes (`0x1C3000` and `0x3DA000`).
4. **Partition Forensic Analysis (JFFS2 Cleartext Config):**
    - Navigate to the extracted `jffs2-root` or `jffs2-root-0` directory.
    - Open the JSON-formatted `config` file and extract the exposed plaintext key-value pairs: 
        - **Key 132 & 133:** Plaintext Wi-Fi SSID and Password.
        - **Key 134:** X.509 Client Certificate (Issued by Pepper IoT prod CA).
        - **Key 135:** 1024-bit RSA Private Key.
5. **Binary Static Analysis (SquashFS Hardcoded Credentials):**
    - Navigate to the extracted secondary SquashFS partition (`3DA000.squashfs`).
    - Import statically linked vendor binaries (such as `/pepper/pepper_app`) into Ghidra set to **ARM v5T (Little-Endian)** mode.
    - Perform string searches and inspect protocol serialization routines to uncover embedded X.509 certificates and hardcoded RSA keys compiled directly into the executable image.

<img width="971" height="141" alt="image" src="https://github.com/user-attachments/assets/5ed27f7a-6143-4893-8a7e-4b3eed502802" />

<img width="971" height="143" alt="image" src="https://github.com/user-attachments/assets/2d8283e1-fe3d-46ef-83cb-60132289fad3" />

#### Results
Sensitive mTLS keys and local network credentials reside in unencrypted, cleartext JSON files within JFFS2 partitions, and additional RSA private keys and X.509 certificates are compiled directly into proprietary binaries inside SquashFS partitions.

#### Recommended Mitigation
Cryptographic keys should be generated dynamically or stored securely within hardware enclaves (TPM/TEE), while storage partitions containing sensitive configurations must be encrypted at rest. Executable binaries should never contain hardcoded cryptographic credentials.
1. **Eliminate Hardcoded Cryptographic Assets:** Remove static RSA private keys and certificates from vendor application source code. Devices should receive unique certificates provisioned during manufacturing or through secure automated enrollment mechanisms (e.g., EST/SCEP).
2. **Encrypted Storage Partitions:** Implement disk-level encryption (such as LUKS or encrypted MTD blocks) for all non-volatile memory partitions storing configuration files or sensitive tokens.
3. **Hardware Secure Enclaves / TPM:** Store device private keys within a hardware Root of Trust or secure enclave that prevents key extraction even when physical access and memory dumping occur.


