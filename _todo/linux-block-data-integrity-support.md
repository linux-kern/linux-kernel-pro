https://lwn.net/Articles/280023/

This is the first take of my data integrity patches.  It's quite hard
to explain everything in a brief couple of paragraphs but I'll try.

There's more information to be found in the docs section at:

	http://oss.oracle.com/projects/data-integrity/


Here's the executive summary:


What's This All About?
----------------------

These patches allow data integrity information (checksum and more) to
be attached to I/Os at the block/filesystem layers and transferred
through the entire I/O stack all the way to the physical storage
device.

The integrity metadata can be generated in close proximity to the
original data.  Capable host adapters, RAID arrays and physical disks
can verify the data integrity and abort I/Os in case of a mismatch.

Right now this is SCSI disk only, but similar efforts are in progress
for SATA and SCSI tape.  With a few minor nits due to protocol
limitations, the proposed SATA format is identical to the SCSI ditto
for easy interoperability.


T10 DIF
-------

SCSI drives can usually be reformatted to 520-byte sectors, yielding 8
extra bytes per sector.  These 8 bytes have traditionally been used by
RAID controllers to store internal protection information.

DIF (Data Integrity Field) is an extension to the SCSI Block Commands
that standardizes the format of the 8 extra bytes and defines ways to
interact with the contents at the protocol level.  We refer to the
extra information as "integrity metadata" or "IMD".

Each 8-byte DIF tuple is split into three chunks:

	- a 16-bit guard tag containing a CRC of the 512-byte data
      	  portion of the sector.

	- a 16-bit application tag which is up for grabs.

	- a 32-bit reference tag which contains an incrementing
          counter for each sector.  For DIF Type 1 it also needs to
          match the physical LBA on the drive.

There are three types of DIF defined: Type 1, Type 2, and Type 3.  My
patches are Type 1 only, although Type 3 devices should work.  Type 2
depends on 32-byte CDBs and is in progress.

Since the DIF tuple format is standardized, both initiators and
targets (as well as potentially transport switches in-between) to
verify the integrity of the data going over the bus.

When writing, the HBA will DMA 512-byte sectors from host memory,
generate the matching integrity metadata and send out 520-byte sectors
on the wire.  The disk will verify the integrity of the data before
committing it to stable storage.

When reading, the drive will send 520-byte sectors to the HBA.  The
HBA will verify the data integrity and DMA 512-byte sectors to host
memory.

IOW, DIF provides means for added integrity protection between HBA and
disk.


Data Integrity Extensions
-------------------------

In order to provide true end-to-end data integrity we need to be able
to get access to the integrity metadata from the OS.  Dealing with
520-byte sectors is quite inconvenient, so we have worked with HBA
manufacturers to separate the data buffer scatter-gather from the
integrity metadata scatter-gather.

Also, the CRC16 is somewhat expensive to calculate in software.  So we
have also allowed alternate checksums to be used.  Currently we only
support the IP checksum which is fast and cheap to calculate.

These two features and a few more knobs constitute what is known as
DIX or the I/O Controller Data Integrity Extensions.

When writing, the HBA will DMA two scatterlists from host memory: One
containing the data as usual, and one containing the integrity
metadata.  The HBA will verify that the two are in agreement and
interleave them before sending them out on the wire as 520-byte
sectors.

When reading, the disk will return 520-byte sectors, the HBA will
verify the integrity, separate IMD from the data, and DMA to the two
separate scatterlists in host memory.


SCSI Layer Changes
------------------

At the SCSI level, there are a few changes required to support this:

 - an extra scatterlist for the integrity metadata

 - tweaks to sd.c to detect and handle disks formatted with DIF

 - sd.c must issue the right READ/WRITE commands when DIF is on

 - helper functions for HBA drivers

 - extra fields in scsi_host to signal the HBA driver's DIF
   capabilities


Block Layer Changes
-------------------

The main idea of DIF/DIX is to allow integrity metadata to be
generated as close to the original data as possible.  So in the long
run we'd like this to happen in userland.  Given mmap(), direct I/O,
etc. this obviously poses some challenges.  *cough*

For now the integrity metadata is generated at the block layer when an
I/O is submitted by the filesystem.  There are also functions that
allow filesystems to use the application tag to mark sectors for
future recovery or similar.

struct bio has been extended with a pointer to a struct bip which in
turn contains the integrity metadata.  The bip is essentially a
trimmed down bio with a bio_vec and some housekeeping.

There are a few hooks inserted in fs/bio.c and block/blk-* to allow
integrity metadata to be handled correctly when splitting, cloning and
merging.  Aside from that, the integrity stuff is completely opaque.

Because we don't want the block layer, filesystems, etc. to know about
DIF, DIX, tuple formats, etc. all the functions that interact with the
integrity metadata reside in the SCSI layer and are registered via a
callback handler template.  The block layer changes have been made so
that the upcoming standards for data integrity on SATA (T13 External
Path Protection) and SCSI tape will fit right in and can register
their own handlers.

I have included a more in-depth description of the block layer changes
in Documentation/block/data-integrity.txt.


The Patches
-----------

Since this patch set is quite intrusive all across the board, I
haven't been able to split it up in incremental pieces.  So the
patches are more or less a logical grouping and won't work
independently.

Right now everything has been set up so it can be toggled with
CONFIG_BLK_DEV_INTEGRITY and CONFIG_SCSI_PROTECTION.  There are a few
places where this makes things really icky in terms of either code or
#ifdefs.  I'm hoping we can eventually promote the data integrity
stuff to become a first class citizen and get rid of that cruft.

Other than that here are the patches.  I'm very interested in
comments, suggestions, etc.


How To Experiment Without DIF Hardware
--------------------------------------

# modprobe scsi_debug protection=1 guard=1 ato=1 dev_size_mb=1024

-- 
Martin K. Petersen	Oracle Linux Engineering

