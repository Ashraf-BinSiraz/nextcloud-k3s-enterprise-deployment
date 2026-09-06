#1======Tailscale Commands in linux

	tailscale status		# check tailscale connected device status
	systemctl status tailscaled	# check tailscale daemon status
	tailscale --help		# shows tailscale common commands
	
DNS name: gilravager.bongo-shiner.ts.net


#2====== NFS setup
	# What is NFS? 
	  Network File Sharing... which allows the kubernetes to keep the database so that 
	  containers can be swapped anytime when crashes or failes. But the database remain 
	  accessible to the containers everytime.

	# RPC is the underlying communication mechanism NFS is built on.
	# rpc(UID 32) and rpcuser (UID 34) (the system accounts)
	# RPC = the communication protocol NFS is built on top of
	# rpcbind = the "phone book" service that helps clients find where NFS services are 
	  actually listening
	# rpc / rpcuser = low-privilege system accounts that let these background services run securely, 
 	  without needing root access


#3====== Commands related to NFS Setup
	sudo pacman -S nfs-utils 	# To install the nfs;	-S = sync/install;
	sudo systemctl enable --now nfs-server		# starts the NFS server now and sets it to auto-start 
	 						  on every future boot.
					# "enable" This writes a permanent configuration in:
					  /etc/systemd/system/multi-user.target.wants/nfs-server.service
					  Actually points to: /usr/lib/systemd/system/nfs-server.service

	#### Systemd doesn't auto-start services just because they're installed — the symlink is the actual 
	switch that tells it "start this on every boot." No symlink = installed but dormant forever, even 
 	after reboots.
	
	
	systemctl status nfs-server	# NFS daemon status show



#4====== Create the shared writable directorry with full permission in server
	 with the path: /srv/nfs/nextcloud

	# Add the following line to adjust the SHARE SETTING of the folder above:   
 	  echo '/srv/nfs/nextcloud *(rw,async,no_subtree_check,no_root_squash,insecure,fsid=0)' | sudo tee /etc/exports

	# EXPLANATION=====
	/srv/nfs/nextcloud		# the folder being shared
	* 				# allow access from any host (fine here, since it's only reachable
					  within your VM/private network anyway)
	rw  				# read/write access
	async				# respond to write requests before changes are fully flushed to disk 
 					  (faster performance; a reasonable trade-off for this use case)
	no_subtree_check		# disables an extra consistency check that's often unnecessary and 
					  can cause issues; standard recommended practice for exports like this
	no_root_squash			# normally NFS maps a remote "root" user to a low-privilege user for 
	 				  security; this disables that, since Kubernetes/containers often need to 
 					  write as root-equivalent
	insecure			# allows connections from non-privileged ports (needed since some 
 					  Kubernetes/container network setups don't use "trusted" 
					  low-numbered ports)
	fsid=0 				# identifies this as the root export filesystem; a common requirement 
					  for NFSv4 setups (which is what Kubernetes' NFS provisioner will use)

	#=======STEP#2------EXPORTING THE SHARE SETTING---------------
	sudo exports -rav		# This re-reads /etc/exports and activates everything listed in it. 
					# -r = re-exports (reload for /etc/exports )
					  -a = does all entries
					  -v = gives verbose output so you can see confirmation.

					# Output should look like: exporting *:/srv/nfs/nextcloud 
					  confirming it worked.


#5====== VERIFICATION OF NFS file writablility and hsare settings

	# Service-level check (is nfs-server running?)
	systemctl status nfs-server

	# Supporting RPC service check
	systemctl status rpcbind

	# Confirm the export is active with correct settings
	sudo exportfs -v

	# Confirm config file content is correct
	cat /etc/exports

	# Confirm kernel-level NFS daemon threads are running
	ps aux | grep nfsd

	# Confirm the share is visible/mountable (client-side view)
	showmount -e localhost









