# H? HW Hacking is my jam

| For this task pc specs | vm/os |
|------------------------|--------|
| (I7/128/8tb/)          | Fedora |

Started at **16:30 FEB 26th**  
Task form: https://hhmoodle.haaga-helia.fi/course/view.php?id=45171

---

## Downloads

- https://github.com/robbins/tp-link-decrypt  
- Download TapoV3 firmware binary

```
aws s3 cp s3://download.tplinkcloud.com/firmware/Tapo_C200v3_en_1.4.2_Build_250313_Rel.40499n_up_boot-signed_1747894968535.bin Tapo_C200v4_en_1.4.2.bin --no-sign-request
```

Download camera dump-file: ---

---

## Tasks

- decrypt firmware image  
- Analyse the image file  
- extract rootfs from the dump file  
- extract rootfs from the image file  
- search available applications  
- analyse and try to open root password  

---

## Process Notes

I started everything by downloading the tasks:

`git clone https://github.com/robbins/tp-link-decrypt`

Got a file with this inside:

example.jpg  
extract_keys.sh  
Makefile  
preinstall.sh  
README.md  
shell.nix  
src

Ran:

`./preinstall.sh`

Had to make few mods because I'm running Fedora:

- changed all `apt` to `dnf`
- changed few programs to compatible versions like binwalk → `sudo dnf install gcc make binwalk openssl`

New try:

`./preinstall.sh` → **Complete!**

---

## Extract Keys

`./extract_keys.sh`

Script didn’t produce `RSA_0_H` at first → crashed.  
After rerunning 4 times it worked.

`make` works now.

---

## Download Binary

```
aws s3 cp s3://download.tplinkcloud.com/firmware/Tapo_C200v3_en_1.4.2_Build_250313_Rel.40499n_up_boot-signed_1747894968535.bin Tapo_C200v4_en_1.4.2.bin --no-sign-request
```

---

## Decrypt Firmware

`bin/tp-link-decrypt Tapo_C200v4_en_1.4.2.bin`

Output:

TP-link firmware decrypt  
Watchful_IP & robbins 03-10-25 v0.0.4  
watchfulip.github.io  
Tapo firmware header found  
RSA-2048  
key/iv:  
KEY=9c6ba1d761e4eee17dfde90cfed603bd  
IV=8778f31423815ce85e9f186b60507edd  
Firmware verification successful  
Decrypted firmware written to Tapo_C200v4_en_1.4.2.bin.dec

decrypt firmware image: **check**

---

## Binwalk

`binwalk Tapo_C200v4_en_1.4.2.bin.dec`

Got:

- Squashfs filesystem, little endian, version 4.0, compression:xz  
- Linux, CPU: MIPS, image type: OS Kernel Image, compression: lzma  
- image name: "mips Ingenic Linux-3.10.14"

Root filesystem is SquashFS v4.

Analyse the image file: **check**

---

## Extract RootFS

`binwalk -e Tapo_C200v4_en_1.4.2.bin.dec`  
`cd _Tapo_C200v4_en_1.4.2.bin.dec.extracted/squashfs-root`

extract rootfs: **check**

---

## Extract Dump

`binwalk -e dump-tapo-c200v3-1.4.2.bin`

Info:

- u-boot-lzo.img bootloader  
- CPU = MIPS  
- Linux image name: "mips Ingenic Linux-3.10.14"  
-l

Warning:

Symlink points outside extraction directory:  
`squashfs-root/etc/sensor -> /tmp/base-files/cfg`

Firmware uses symlink to `/tmp` → possible config manipulation.

extract rootfs: **check**

---



---
## Tapo dump analysis
_dump-tapo-c200v3-1.4.2.bin-0.extracted/squashfs-root conteins:
squashfs-root/
 - `├── bin`
 - `├── config`
 - `├── etc`
 - `├── lib`
 - `└── usr`
 First things firt:
``` cat etc/passwd```
``` cat etc/shadow```
 Nothing ):

 - Tried to find somthings in the bin/main with:
 ```
strings bin/main | grep -i login
strings bin/main | grep -i root
strings bin/main | grep -i passwd
```

Sure that Authentication handled in /bin/main


## Interesting Files


### dpss_public_key.pem

-----BEGIN PUBLIC KEY-----  
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE++2MMSe9kWF6mJPkgsGcrqTCk8gb  
vTccKPGZDL5pZcmfW+2EzRjfXhx6clwg3Wl4ClFe8+37fLNgxPYodqGQgA==  
-----END PUBLIC KEY-----

Probably firmware signature or secure boot verification.

### Debug Binaries

- `./bin/gdbserver`
- `./bin/impdbg`

### Strings Output

`strings -t x bin/main | grep -i hmac`

Found:

- mbedtls_md_hmac_starts  
- mbedtls_md_hmac_update  
- mbedtls_md_hmac_finish  
- [telemetry]hmac_sha1_encode error  
- HMAC with password: %s  
- hmac-not-implemented  

Also PasswordDigest → ONVIF.


`strings bin/main | grep -i crypt`
`found ->update_root_passwd_for_encrypt`
`strings -t x bin/main | grep -i update_root_passwd_for_encrypt`
 `2ad37c update_root_passwd_for_encrypt`

Nothing actually too usefull I think.

<img width="1160" height="791" alt="image" src="https://github.com/user-attachments/assets/1da3f34c-e444-457d-86b8-9a7733c3348e" />

---
## CVE‑2025‑8065
(- https://www.cvedetails.com/cve/CVE-2025-8065)

I wanted to check if there is a CVE for the firmware version I’m analyzing (*Tapo_C200v3_en_1.4.2*).  
I found a possible vulnerability (CVE‑2025‑8065) related to the ONVIF XML parser.  
This is what I did and what I found. I had actually seen interesting ONVIF strings in the firmware. I tried to find it again and this is what I did, and what I got.

---

### Step 1  
**Search for ONVIF strings**

```
strings -t x bin/main | grep -i onvif
```

Output:

```
5481  0x0029b034 0x0069b034 24   25   .rodata ascii   soap_parse_element_value
5541  0x0029bc44 0x0069bc44 38   39   .rodata ascii   [ONVIF]soap_parse_element_value failed
6765  0x002a7964 0x006a7964 39   40   .rodata ascii   [ONVIF]soap_parse_element_value failed\n
```

This string likely names the function used for XML parsing.

---

### Step 2  
**Using `radare2`**  
Reference: https://gist.github.com/werew/cad8f30bc930bfca385554b443eec2a7

Command:

```
r2 -a mips -b 32 bin/main
```

Inside radare2:

```
aaa
axt @ 0x0069b034
```

Output:

```
fcn.0048b7fc; str.soap_parse_element_value 0x48b968 [DATA:r--] addiu a3, a3, -str.soap_parse_element_value
fcn.0048b7fc; str.soap_parse_element_value 0x48b9f8 [DATA:r--] addiu a3, a3, -str.soap_parse_element_value
```

Then:

```
s 0x0048b7fc
pdf
```

<img width="1162" height="969" alt="image" src="https://github.com/user-attachments/assets/680b4820-4b90-4b4d-839f-951b2e0c3d28" />

---

### Notes  
The function appears to parse SOAP element text.  
The ONVIF parsing error cases suggest it likely builds the element buffer.

I’ve never done anything with buffer overflows, so this is the best I can do for now (:

 

---



## Blog Reference

https://quentinkaiser.be/security/2025/07/25/rooting-tapo-c200/

Checking dump size:

`stat dump-tapo-c200v3-1.4.2.bin`

Size: 8388608 → matches 8MB.

dd command:

dd if=dump-tapo-c200v3-1.4.2.bin \
   of=kernel.bin \
   bs=1 \
   skip=$((0x070200)) \
   count=$((0x1b0000 - 0x070200))

Rest Ill do when I have time (:

---
## Sources

- https://static.tp-link.com/upload/gpl-code/2022/202211/20221130/c310v2_GPL.tar.bz2
- https://www.cvedetails.com/version/498090/Linux-Linux-Kernel-3.10.14.htm
- https://www.cvedetails.com/cve/CVE-2025-8065/
- https://gist.github.com/werew/cad8f30bc930bfca385554b443eec2a7
- http://download.tplinkcloud.com/firmware/ax6000v2-up-ver1-1-2-P1[20230731-rel41066]_1024_nosign_2023-07-31_11.26.17_1693471186048.bin.rollback
- http://download.tplinkcloud.com/firmware/Tapo_C210v1_en_1.3.1_Build_221218_Rel.73283n_u_1679534600836.bin
- https://static.tp-link.com/resources/gpl/rtk-maple_gpl.tar.gz

