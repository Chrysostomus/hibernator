#!/usr/bin/env bash
# script to enable hibernation noninteractively, based on powerdown-functions
shopt -s nullglob extglob

swapfilesize=${1-4G}
hibernate_offset() {
    filefrag -v /swapfile | awk 'NR==4 {print $4}' | tr -d .
}

root_part() {
    grep " / " /etc/fstab | awk '{print $1}' | sed 's/PARTUUID/UUID/' | sed 's/PARTUUID/UUID/'
}

root_filesystem() {
    grep " / " /etc/fstab | awk '{print $3}'
}

btrfs_root () {
    [[ $(root_filesystem) == btrfs ]]
}

has_swap_part () {
    grep -qs swap /etc/fstab
}

can_suspend_to_disk () {
    [[ -f /swapfile ]] || has_swap_part
}

has_swap () {
    [[ -f /swapfile ]] || has_swap_part
}

swap_part() {
    awk '$3=="swap" {print $1; exit}' /etc/fstab
}

# lock the file until the script finishes
lock() {
    local LOCK=/tmp/hibernator.lock
    if ! mkdir "$LOCK" 2> /dev/null; then
        echo "Working... $LOCK"
        exit
    fi
    trap "rm -rf $LOCK" EXIT
}

if [[ $EUID != 0 ]]; then
    echo "[hibernator] must be run as root"
    exit 1
fi

has_resume_hook () {
    grep -qs -e resume -e systemd /etc/mkinitcpio.conf
}

resume_boot_option() {
    if [[ -f /swapfile ]]; then
        echo "resume=$(root_part) resume_offset=$(hibernate_offset)"
    elif has_swap_part; then
        echo "resume=$(swap_part)"
    fi
}

grub_resume_boot_option() {
	grub_root_part=$(find /dev/disk/ | grep "$(awk '$2~/^\/$/ {print $1}' /etc/fstab | cut -d= -f 2)")
	grub_swap_part=$(find /dev/disk/ | grep "$(awk '$3=="swap" {print $1; exit}' /etc/fstab | cut -d= -f 2)")
    if [[ -f /swapfile ]]; then
        echo "resume=$grub_root_part resume_offset=$(hibernate_offset)"
    elif has_swap_part; then
        echo "resume=$grub_swap_part"
    fi
}

create_swap_file() {
	#Create and setup swapfile
	fallocate -l $(echo $swapfilesize) /swapfile
	chmod 600 /swapfile
	mkswap /swapfile
	swapon /swapfile
	echo '/swapfile	swap	swap	defaults	0	0' >> /etc/fstab

}

add_kernel_parameters() {
	#Add needed kernel parameters to grub
	if [ -e /etc/default/grub ]; then
		cp /etc/default/grub /etc/default/grub.old
		sed -i "s/[[:blank:]]*$//" /etc/default/grub
		sed -i "/^GRUB_CMDLINE_LINUX_DEFAULT/ s~\"$~ $(grub_resume_boot_option)\"~g" /etc/default/grub && update-grub
	fi
	#Add needed kernel parameters to refind
	if [ -e /boot/refind_linux.conf ]; then
		cp /boot/refind_linux.conf /boot/refind_linux.conf.old
		grep -q "$(resume_boot_option)" /boot/refind_linux.conf || sed -i 's/"$/ '"$(resume_boot_option)"'"/' /boot/refind_linux.conf
	fi	
}

add_resume_hook() {
	#Add resume hook to initramfs if necessary
	if ! has_resume_hook; then
		cp /etc/mkinitcpio.conf /etc/mkinitcpio.conf.old
		sed -i '/^#/!s/filesystems/filesystems resume/g' /etc/mkinitcpio.conf
		echo "Adding resume hook to /etc/mkinitcpio.conf"
		mkinitcpio -P
	else 
		echo "Resume hook was already present"
	fi
}
#############################
main() {

lock

if ! has_swap; then
	echo "No swap partition or swapfile detected."
	if btrfs_root; then
		echo "Your root filesystem is btrfs and does not support swapfiles!"
		echo "Try again after manually creating a swap partition."
	else
		create_swap_file && echo "Created a swapfile."
		echo "Adding necessary kernel parameters to your bootloaders" && echo "" && add_kernel_parameters
		add_resume_hook
	fi
elif has_swap_part; then
	echo "Adding the necessary kernel parameters to your bootloaders" && echo "" && add_kernel_parameters 
	add_resume_hook
fi
}

main
