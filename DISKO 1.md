# DISKO 1

[picoCTF](https://play.picoctf.org/practice/challenge/505)

### Forensics

- a file is given and you have to get the flag from it.

- file is named `disko-1.dd` and we will first check what type of file it is.
  
  ```
  $ file disko-1.dd
  
  disko-1.dd: DOS/MBR boot sector, code offset 0x58+2, 
  OEM-ID "mkfs.fat", Media descriptor 0xf8, 
  sectors/track 32, heads 8, 
  sectors 102400 (volumes > 32 MB), 
  FAT (32 bit), sectors/FAT 788, 
  serial number 0x241a4420, unlabeled
  ```

- now we analyse it.
  
  - `disko-1.dd` is the file name.
  
  - `DOS/MBR boot sector` file contains DOS style Master Boot Record.
    
    - This boot sector is typically found at the beginning of a disk or partition and contains the necessary code to start the boot process.
    
    - It refers to the volume boot sector in the MBR and not the MBR itself.
    
    - In linux they are found in file systems such as FAT or ext2.
  
  - `code offset 0x58+2` this tells where the executable boot code starts inside the boot sector.
    
    - Boot sector of a FAT file system is exactly 512 bytes.
    
    - it stores some metadata like file system type, sector count etc and also some machine code that is meant to boot the system. 
    
    - `0x58+2` is 90 bytes. 0x58 is 88 in decimal and +2 = 90.
    
    - now when a pc boot the BIOS loads first sector of 512 bytes at a fixed address and start executing it. But not all 512 bytes is code most is just data like BIOS parameter, block, filesystem info.
    
    - real boot code is started at a safe offset often around 62-90 bytes.
    
    - For **reverse engineering** or **malware analysis**, knowing where the boot code starts helps you **disassemble the machine code**.
      
      For **bootloader development**, it tells you where to inject or read the code from if you're building a custom boot sector.
      
      For **forensics**, it helps determine if the boot sector has been tampered with or if custom boot code has been injected (e.g., bootkits or MBR-based malware).
      
      If you open the disk in a hex editor, you’ll see:
      
      - From offset `0x00` to `0x59`: mostly filesystem metadata (OEM string, sector size, etc.)
      
      - From `0x5A` onward: the boot code starts (x86 instructions like `mov`, `jmp`, etc.)
  
  - `OEM-ID "mkfs.fat"` OEM id is 8 byte stored in boot sector and here it tells `mkfs.fat` was used to create the file system.
  
  - `Media descriptor 0xf8` this tells the type of storage
    
    - `0xF8` is standard for fixed non removable disks like hard disks.
  
  - `sectors/track 32, heads 8` this refers to CHS geometry (Cylinder HEad Sector), old method for addressing sectors.
    
    - 32 sectors per track and 8 heads (sides of platter)
    
    - this helps legacy tools and bios to navigate the disk.
  
  - `sectors 102400 (volumes > 32 MB)` total sectors in this filesystem is 102400. 
    
    - each sector is 512 bytes, so total is 102400 * 512 = around 50 mb. Since it is bigger than 32mb it is FAT32 partition.
  
  - `FAT (32 bit)` confirms that it is FAT32.
  
  - `sectors/FAT 788` FAT32 uses 2 FAT Tables each one sues 788 sectors. so 788 * 512 = 394kb per FAT.
  
  - `serial number 0x241a4420` every FAT volume has a volume serial number, usually based on date and time.
  
  - `unlabeled` this means that the label has no name.
  
  ---
  
  ### How to do forensics 101
  
  ##### 1. Verify and identify the image.
  
  - We get info using `file`  command we did previously.
  
  - We generate a `md5` or `sha` hash to make sure that the file is not modified later on.
  
  ##### 2. Extract partition info
  
  - we use `fdisk` to get disk image info
  
  ```
  $ fdisk -l disko-1.dd
  Disk disko-1.dd: 50 MiB, 52428800 bytes, 102400 sectors
  Units: sectors of 1 * 512 = 512 bytes
  Sector size (logical/physical): 512 bytes / 512 bytes
  I/O size (minimum/optimal): 512 bytes / 512 bytes
  Disklabel type: dos
  Disk identifier: 0x00000000
  ```
  
  - `Disk disko-1.dd: 50 MiB, 52428800 bytes, 102400 sectors`this tell the size of the disk and number of sectors. we can do 102400 * 512 = 5242880 bytes to confirm the total size.
  
  - `Units: sectors of 1 * 512 = 512 bytes` Each sector is **512 bytes** (standard for most disks)
  
  - `Sector size (logical/physical): 512 bytes / 512 bytes` Logical sector size what the os uses is 512 bytes and Physical sector what the hardware uses is 512 bytes. Both should be matching.
  
  - `I/O size (minimum/optimal): 512 bytes / 512 bytes` smallest block that it can read is 512 bytes and optimal is 512 bytes. This is typical for usb sticks or small virtual images.
  
  - `Disklabel type: dos` this shows it is DOS type partition AKA MBR.
  
  - `Disk identifier: 0x00000000` this is the MBR disk signature, a 32 bit id used by the OS to identify the disk.
    
    - here it is all 0, which is unusual. This means it is either incomplete or hand crafted MBR partition.
  
  ##### 3. Mount the image in read only so you don't modify it
  
  ```
  $ sudo mkdir /mnt/diskimage
  $ sudo mount -o ro, loop disko-1.dd /mnt/diskimage
  ```
  
  - now if there is a partition table then you will have to provide the offset argument too.
  
  ```
  $ sudo mount -o ro,loop,offset=$((start_sector*512)) disko-1.dd /mnt/diskimage
  ```
  
  like this.
  
  ##### 4. Now explore.
  
  - look for hidden files.
  
  - look at change date.

---

### Back to CTF

- Now CTF requires us to search for a flag in the disk image.

- I first mounted the image and tried to search for the flag but it didn't work. I used nvim and grep to look through some file. This means the flag is hidden.

- next i searched for `picoCTF` using strings in the disk image itself using: `strings disko-1.dd | grep -iE 'picoCTF'` this returned the flag. Ez right. no i failed to look for it using grep or xxd, so i went and opened it in autopsy.

- Here I saw that the flag was surrounded by null characters. That is why grep and nvim can't see it. looks like even xxd doesn't handle null charters (00).

- Only `strings` handles the null character. Sector was 58162.

- in the command `grep -iE` is there `E` is to enables extended regular expressions (ERE) and `i` is to make search case insensitive.
