=== TASK 6: KUBERNETES (K3S) ===

#======Check if an AUR helper is already installed:
	which yay paru			# confirms whether yay/paru exist; neither found, 
					  so needed to build yay



#======Install build tools needed to compile AUR packages
	sudo pacman -S --needed base-devel git
  					# -S = install package(s)
  					# --needed = skip reinstalling if already present
  					  (base-devel = compilers/build tools, git = needed to clone AUR source)

#=======Download yay's build script from AUR
	git clone https://aur.archlinux.org/yay-bin.git
	cd yay-bin			# (yay-bin = precompiled version, faster than 
					  building yay from Go source)

#=======Build and install yay
	makepkg -si
  					# -s = auto-install any missing dependencies
  					# -i = install the resulting package after building
  					  (prompted: cleanBuild -> N, view PKGBUILD -> N, 
 					  diffs -> N, confirm install -> Y)

#=======Verify yay works
	cd ~
	which yay
	yay --version



#=======Install K3s from AUR
	yay -S k3s-bin 				# (same y/n prompts as above: cleanBuild N, 
 					  	  diffs N, confirm install Y)



#=======Enable and start K3s (auto-start on boot + start now)
	sudo systemctl enable --now k3s
  						# enable = auto-start on every future boot
  						# --now = also start immediately


#=======Verify node is ready
	sudo k3s kubectl get nodes		# look for STATUS: Ready



#=======Verify all system pods are healthy
	sudo k3s kubectl get pods -A		# look for STATUS: Running (1/1) or Completed for every pod

#=======Live-monitor pods until stable (if still starting up)
	sudo watch k3s kubectl get pods -A	# (Ctrl+C to exit once all pods show 
 						Running/Completed)

#========REBOOT TEST — confirm K3s survives reboot automatically
	sudo reboot
  
	# (after reboot, reconnect and re-run the two verification commands above)
      	# CONFIRMED: node Ready, all pods Running/Completed after reboot, no manual steps needed


