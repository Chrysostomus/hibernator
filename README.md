# hibernator
Script to automate hibenation setup on archlinux based systems.

Hibernator adds resume hook if needed, and provides installed bootloaders with necessary kernel parameters to resume from hibernation. It uses existing swap partition, but if there is none, it creates swapfile and uses that instead. 

Hibernator accepts desired size of swapfile as arguments. Running hibernator 2G creates 2 Gb swapfil, hibernator 1000M creates 1000 Mb swapfile. The script defaults to 4G if no arguments are given.
