ExecdCount = 1
DaikerOpts =-e random -c 8 -r 4
SshOpts = -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=2
Ssh = ssh -i var/id_ed25519 -p 54321 $(SshOpts)
var/execd : $(addprefix var/execd,$(shell seq $(ExecdCount)))
	touch $@
ALL := $(addprefix var/,firewall master $(addprefix execd,$(shell seq $(ExecdCount))))
define ExecdTmpl
var/execd$1 : var/master-container
	setsid var/daiker build $(DaikerOpts) -d /dev/null & ps --no-headers -o pgid $$$$! | tr -d ' ' > $$@
endef
$(foreach E,$(shell seq $(ExecdCount)),$(eval $(call ExecdTmpl,$E)))
var/master-container : var/master-image
	fuser -k $@.qcow2 || :
	rm -f $@.qcow2
	var/daiker run $(DaikerOpts) -d -b $<.qcow2 $@.qcow2 &
	while ! $(Ssh) root@localhost ssh $(SshOpts) master id; do sleep 2; done
	rsync -ae "$(Ssh)" root@localhost:/var/cache/pacman/pkg/ var/overlay/var/cache/pacman/pkg
	touch $@
var/master-image : var/master-autoinstall.iso var/firewall-container
	fuser -k $@.qcow2 || :
	rm -f $@.qcow2
	var/daiker build $(DaikerOpts) -d -H 20 -i $< $@.qcow2
	touch $@
var/firewall-container : var/firewall-image
	fuser -k $@.qcow2 || :
	rm -f $@.qcow2
	var/daiker run $(DaikerOpts) -T 12345-54321 -b $<.qcow2 $@.qcow2 &
	while ! $(Ssh) root@localhost id; do sleep 2; done
	rsync -ae "$(Ssh)" root@localhost:/var/cache/pacman/pkg/ var/overlay/var/cache/pacman/pkg
	touch $@
var/firewall-image : var/firewall-autoinstall.iso
	fuser -k $@.qcow2 || :
	rm -f $@.qcow2
	var/daiker build $(DaikerOpts) -H 20 -T 12345-54321 -i $< $@.qcow2
	touch $@
var/%-autoinstall.iso : var/archlinux-x86_64.iso var/id_ed25519 var/daiker
	[ ! -d $@.d ] || chmod -R u+rwX $@.d && rm -rf $@.d
	mkdir -p $@.d
	bsdtar xfp $< -C$@.d
	chmod -R u+rwX $@.d
	sed -i "s/archiso_sys-linux.cfg/autoinstall.cfg\nINCLUDE archiso_sys-linux.cfg/; s/TIMEOUT 150/TIMEOUT 100/; s/DEFAULT arch64/DEFAULT $*/" $@.d/boot/syslinux/archiso_sys.cfg
	sed -i "s/ archisolabel=ARCH_202.../ archisolabel=ARCH_AUTO/" $@.d/boot/syslinux/archiso_sys-linux.cfg
	cd $@.d/arch/x86_64 && unsquashfs -no-xattrs airootfs.sfs
	rsync -a lib/autoinstall.cfg $@.d/boot/syslinux/
	rsync -a lib/overlay/ $@.d/arch/x86_64/squashfs-root/
	mkdir -p var/overlay/var/cache/pacman/pkg
	rsync -a var/overlay/ $@.d/arch/x86_64/squashfs-root/
	cp -p var/id_ed25519* $@.d/arch/x86_64/squashfs-root/root/.ssh/
	cat var/id_ed25519.pub > $@.d/arch/x86_64/squashfs-root/root/.ssh/authorized_keys
	rsync -a $@.d/arch/boot $@.d/arch/x86_64/squashfs-root/root/sge/firewall/overlay/var/ftpd/tftpboot/pxelinux/files/
	cd $@.d/arch/x86_64/squashfs-root/etc/systemd/system/multi-user.target.wants && ln -s ../sge-auto.service .
	rm $@.d/arch/x86_64/squashfs-root/etc/motd
	sed -i "s#/usr/bin/zsh#/bin/bash#" $@.d/arch/x86_64/squashfs-root/etc/passwd
	chmod -R u+rwX $@.d/arch/x86_64/squashfs-root
	cd $@.d/arch/x86_64 && rm airootfs.sfs && mksquashfs squashfs-root airootfs.sfs
	rm -r $@.d/arch/x86_64/squashfs-root
	genisoimage -quiet -l -r -J -V "ARCH_AUTO" -no-emul-boot -boot-load-size 4 -boot-info-table -b boot/syslinux/isolinux.bin -c boot/syslinux/boot.cat -o $@.tmp $@.d
	rm -rf $@.d
	mv $@.tmp $@

var/archlinux-x86_64.iso :
	mkdir -p $(@D)
	wget -cO $@.tmp http://repo.miserver.it.umich.edu/archlinux/iso/latest/archlinux-x86_64.iso
	mv $@.tmp $@
var/id_ed25519 :
	mkdir -p $(@D)
	ssh-keygen -C SGE -N "" -f $@
var/daiker :
	mkdir -p $(@D)
	wget -qcO $@.tmp https://raw.githubusercontent.com/daimh/daiker/master/daiker
	chmod +x $@.tmp
	mv $@.tmp $@
clean : reset
	rm -rf var
reset : killall
	chmod -R u+rwX var/*.d || :
	cd var && rm -f firewall* master* id_ed25519* daiker || :
killall :
	fuser -k var/* || :
	for pg in $(addprefix var/execd,$(shell seq $(ExecdCount))); do [ -f $$pg ] || continue; kill -- -$$(cat $$pg) || :; rm -f $$pg; done
