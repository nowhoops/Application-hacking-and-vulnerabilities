# H? HW Hacking is my jam

| For this task pc specs | vm/os |
|---------|-------------|
| (I7/128/8tb/) | Fedora |

Started at 16:30  FEB 26th
Task form: https://hhmoodle.haaga-helia.fi/course/view.php?id=45171

Download and install 
https://github.com/robbins/tp-link-decrypt

Download TapoV3 firmware binary

aws s3 cp s3://download.tplinkcloud.com/firmware/Tapo_C200v3_en_1.4.2_Build_250313_Rel.40499n_up_boot-signed_1747894968535.bin Tapo_C200v4_en_1.4.2.bin --no-sign-request

Download camera dump-file: ---

Tasks:

    decrypt firmware image
    Analyse the image file
    extract rootfs from the dump file
    extract rootfs from the image file
    search available applications
    analyse and try to open root password

I started everyting by downloading the tasks:
git clone https://github.com/robbins/tp-link-decrypt
go a file with this inside:
example.jpg  extract_keys.sh  Makefile  preinstall.sh  README.md  shell.nix  src

I ran ./preinstall.sh to to satisfy dependencies. 





