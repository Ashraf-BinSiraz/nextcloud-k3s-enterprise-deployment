#2 === To Update Package manager on Archlinux

#2 SOLUTION
	1. sudo pacman -Syu	# -S = sync/install;
				  -y = refresh the package database;
				  -u = upgrade all installed package
	
	2. To check what's updated since last syn:
		2.1	tail -50 /var/log/pacman.log	
				# Look for: [2026-08-10T...] [ALPM] upgraded openssl (3.3.1-1 -> 3.3.2-1)
		2.2 	grep "upgraded" /var/log/pacman.log | tail -30
				# gives clean upgrade
	
	3. Always safe to reboot after kernal change with: sudo reboot
	4. Check after reboot: uname -r	# this will show latest kernal version



#3 === Install TAILSCALE
#3 SOLUTION
	1. sudo pacman -S tailscale	#  installs the Tailscale client package from Arch's official repos.
	2. sudo systemctl enable --now tailscaled
		# This starts the Tailscale background daemon <tailscaled> and sets it to auto-start on boot.
		# Created symlink '/etc/systemd/system/multi-user.target.wants/tailscaled.service' → 
 		  '/usr/lib/systemd/system/tailscaled.service'.
	3. To Verify status:
		systemctl status tailscaled	# Shows the daemon process status
						# check for Active: active (running)



#4 === Generate AUTH Key and SYNC
#4 SOLUTION
	1. Go back to tailscale.com in your browser (Admin Console, if it doesn't take you straight there)
	Click: Settings>keys>Generate auth key>"Generate Auth Key" button

	2. Copy the key.

	3. THE KEY: tskey-auth-khwDFBf6Bz11CNTRL-XcL79w7yAqQq49ZJMb1xpQNhtE8P8Azz

	4. Use the following command to link the login with the tailscale daemon:
		sudo tailscale up --authkey=tskey-auth-khwDFBf6Bz11CNTRL-XcL79w7yAqQq49ZJMb1xpQNhtE8P8Azz

	5. TO VERIFY:
		systemctl status tailscaled
		tailscale status


#5 === TAILNET IP ASSIGNMENT
#5 SOLUTIONS
	1. After successfull connectivity IPS will be assigned. Check TAILSCALE dashboard.
	2. Laptop IP: 		100.86.226.99
	3. VM(Gilravager) IP: 	100.103.220.52
	4. Test by pinging 



