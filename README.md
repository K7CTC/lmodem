# LMODEM

LMODEM is a file transfer protocol purpose built for LoRa.  Its name is a throwback to the protocols used in the BBS days (X, Y, and ZMODEM).

## Why?

The short answer, just to see if it was possible.  After completing work on my LoRa chat application I had all the requisite knowledge for sending and receiving ASCII bytes over LoRa.  One day I thought to myself how neat it would be to transfer an arbitrary file over LoRa.  I searched the internet for days looking for any examples of anyone attempting this feat.  Apparenly I was the first person, so I took it as a personal challenge to scratch build a protocol for the sole purpose of proving that I could transfer files over LoRa.

## Limitations

* Maximum file name length of 32 characters.
* Maximum "over the air" file size of 32,768 bytes (though the source file can be many times bigger).
* Maximim tested range ~20 miles line of sight.

## Features

* File integrity check using BLAKE2b secure hash algorithm provides 100% confidence in transferred files.
* Lempel-Ziv Markov chain Algorithm (LZMA) compression significantly reduces "over the air" file size.
* Partial transfer resume.  You can reattempt as many times as is necessary to finish an incomplete transfer without having to start over.
* LMODEM "modes" simplify underlying LoRa settings and provide a user friendly experience.
* LMODEM "channels" eliminate the need for users to manually specify an operating frequency.

## Operating Principles

The sending station takes a file (in the current working directory) as input.  The protocol performs some quick checks on the file first.  It checks that the file exists.  It gets the file size (on disk).  It checks that the filename does not exceed 32 characters.  It generates a secure has for the file with a message digest size of 32 bytes.  The file is then compressed (in memory) using LZMA.  The compressed file undergoes binary to text encoding using the Base85 encoding scheme.  Base85 is more efficient than Base64 and maintains compatibility with LoRa.  LMODEM then checks that the compressed and encoded file does not exceed 32kb.  32kb is the maximum over-the-air file size.  The file is then encoded in hexadecimal.  The file is then divided into 128 byte blocks (with any remainder in the final block).  The block numbers (zero padded) are then prefixed to the blocks themselves, thus creating the packets that will be sent over the air.  The sending station is now ready to present the file to the receiving station.

Once a basic handshake is performed, both the sending and receiving stations are "synchronized" and the sending station sends the file transfer details packet.  This packet consists of:

* Filename
* Size on disk (in bytes)
* Size over the air (in bytes)
* Block count
* Secure hash

The receiving station processes the file transfer details packet as follows:

* It displays the file transfer details to the user.
* It checks to see if the incoming file already exists.
    - If so, it compares the secure hash to see if the file is a duplicate.
        + If so, it tells the sending station that the file is a dupe and the transfer is aborted.
        + If not, the user and the sending station is informed that a file with the same name exists.
* It checks if a partial file exists.  If so, it is transferred from disk to memory.
* It checks the secure hash (stored in the partial file) against the incoming file.
    - If the hash matches, the file transfer is resumed.
        + Missing blocks are enumerated and requested from the sending station.
    - If the hash does not match, the partial file is purged from memory and the transfer begins.

The sending station is therefore requested to send all or a subset of file blocks.  When this task is completed, the sending station notifies the receiving station that all requested blocks have been sent.

The receiving station checks for any missing blocks.  If there are missing blocks, the secure hash of the incoming file is appended and a partial file is written to disk.  The users on both ends are instructed to try again.  It is suggested that a more robust LMODEM mode is selected.

If there are no missing blocks.  The receiving station reconstitues the file:

* The hexadecimal encoded blocks are sequentially concatenated into a single blob.
* The blob is decoded from hexadecimal back into Base85 encoded ASCII bytes.
* The blob is converted from Base85 back into binary.
* The binary blob is then decompressed (using LZMA).
* The decompressed binary is then written to disk (using the file name provided in the file transfer details).
* The file on disk is then hashed using BLAKE2b.
* The secure hashes are compared to guarantee file integrity.
* If the integrity check fails, the file is removed from disk.



# LMODEM Notes

## LMODEM LoStik Modes

### MODE1 - Short Range (Fast)
* PWR = 6
* BW = 500
* SF = sf8
* CR = 4/6
* WDT = 875
* Block Size = 192 bytes
* Max OTA Size = 49152 bytes

It takes 166ms to send 192 bytes in MODE1

### MODE2 - Medium Range (Balanced)
* PWR = 12
* BW = 250
* SF = sf10
* CR = 4/7
* WDT = 1600
* Block Size = 128 bytes
* Max OTA Size = 32768

It takes 859ms to send 128 bytes in MODE2

### MODE3 - Long Range (Slow)
* PWR = 17
* BW = 125
* SF = sf12
* CR = 4/8
* WDT = 8500
* Block Size = 64 bytes
* Max OTA Size = 16384

It takes about 4336ms to send 64 bytes in MODE3

## LMODEM Channels
1. 914 MHz
2. 915 MHz
3. 916 MHz
