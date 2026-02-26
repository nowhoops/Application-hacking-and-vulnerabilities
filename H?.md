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

/////
aws s3 cp s3://download.tplinkcloud.com/firmware/Tapo_C200v3_en_1.4.2_Build_250313_Rel.40499n_up_boot-signed_1747894968535.bin Tapo_C200v4_en_1.4.2.bin --no-sign-request
/////

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

/////
aws s3 cp s3://download.tplinkcloud.com/firmware/Tapo_C200v3_en_1.4.2_Build_250313_Rel.40499n_up_boot-signed_1747894968535.bin Tapo_C200v4_en_1.4.2.bin --no-sign-request
/////

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
- https://www.cvedetails.com/version/498090/Linux-Linux-Kernel-3.10.14.html

Warning:

Symlink points outside extraction directory:  
`squashfs-root/etc/sensor -> /tmp/base-files/cfg`

Firmware uses symlink to `/tmp` → possible config manipulation.

extract rootfs: **check**

---



---

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
Found: `update_root_passwd_for_encrypt`

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

---

## Sources

This program uses libsecurity GPL code from:  
https://static.tp-link.com/upload/gpl-code/2022/202211/20221130/c310v2_GPL.tar.bz2

TP-Link firmware links used:

- http://download.tplinkcloud.com/firmware/ax6000v2-up-ver1-1-2-P1[20230731-rel41066]_1024_nosign_2023-07-31_11.26.17_1693471186048.bin.rollback  
- http://download.tplinkcloud.com/firmware/Tapo_C210v1_en_1.3.1_Build_221218_Rel.73283n_u_1679534600836.bin  
- https://static.tp-link.com/resources/gpl/rtk-maple_gpl.tar.gz
