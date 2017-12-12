---
title: Dealing with bad sectors on home PC
layout: post
---

I have a Lenovo desktop PC at home and some of the diagnostics on the hard drive were failing, like the Targeted Read Test.  The drive is a 2TB Seagate.  

I was able to get the bad sectors marked and the tests to pass, but it took some effort and multiple Google searches.  So I'm throwing this out there in case
 it helps anyone else.  Your milage may vary.  And as always, have your important data backed up and a recovery drive ready if your drive is at risk of failure.

*Issue #1*: Seagate recommends running SeaTools for DOS to "repair" bad sectors, from a bootable CD.  ("Repair" in quotes because the sector is not actually fixed,
but marked as bad so it's not used.)  Problem: no helpful message on start screen telling me what key to press to get into BIOS.  After Googling and trial and error
I figured out that holding down `F12` is the right magic to get into BIOS on this particular model.

*Issue #2*: CD drive not on boot device menu.  I had to go into setup, then on the Startup tab change from UEFI only to "Legacy Support".  Now the CD shows up
in the boot menu and you can select it.

*Issue #3*: SeaTools for DOS did not recognize my drive.  Solution: Go back into setup, then on the Devices tab change AHCI mode to IDE mode, now SeaTools will 
recognize the drive.

Once I worked through those issues I was able to run the SeaTools utilities on my drive and "repaired" the bad sectors.  There were only a handful, and because 
the drive is relatively empty no data was lost yet, but I'll have to keep an eye on it to see if the bad sectors proliferate.  Hopefully this buys me some
time before replacing the drive.  