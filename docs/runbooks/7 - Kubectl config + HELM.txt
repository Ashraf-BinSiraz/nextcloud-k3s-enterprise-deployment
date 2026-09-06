
###==================VERIFICATION OF KUBERNETES, NFS, TAILSCALE DAEMONS AND TAILNET DEVICES ARE READY=========================


#====== POST-REBOOT VERIFICATION (K3S + NFS + TAILSCALE) ===

#=======Verify K3s node is ready after reboot
	sudo k3s kubectl get nodes				# look for STATUS: Ready

#=======Verify all K3s system pods are healthy after reboot
	sudo k3s kubectl get pods -A				# look for STATUS: Running (1/1) or Completed for every pod

#=======Verify NFS server service survived reboot
	systemctl status nfs-server				# look for Active: active (exited), status=0/SUCCESS

#=======Verify Tailscale daemon survived reboot
	systemctl status tailscaled				# look for Active: active (running), Status: "Connected;..."

#=======Verify all tailnet devices still registered
	tailscale status					# confirms VM + laptop + phone all still listed
						  		  (phone may show "offline, last seen..." if not 
						  		  actively connected — normal, not a problem)




###===== NEXT (NOT YET DONE): TASK 7 — KUBECTL CONFIG + HELM ===

#=======Give your regular user access to kubectl (not just root)
	mkdir -p ~/.kube
	sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
	sudo chown $USER:$USER ~/.kube/config				# (copies K3s's cluster credentials to your user's home dir, 
   									  then changes ownership so you can read it without sudo every time)

#=======Install Helm (Kubernetes package manager)
	sudo pacman -S helm



#=======Attempted plain kubectl (not installed separately)
	### kubectl get nodes			# FAILED: command not found — expected, since 
						  K3s bundles kubectl inside its own binary. 
						  Use "sudo k3s kubectl" instead.

#=======Verify cluster still healthy (K3s's own kubectl)
	sudo k3s kubectl get nodes		# look for STATUS: Ready

#=======Verify Helm installed correctly
	helm version				# look for version.BuildInfo with no errors

#=======Verify kubeconfig file exists with correct ownership
	ls -la ~/.kube/config			# should show owner: student:student, 
						  permissions: -rw------- (600)

#=======Verify Helm can actually talk to the cluster
	helm list -A				# -A = all namespaces
						# lists every Helm release across the cluster

	RESULT: showed "traefik" and "traefik-crd" already deployed
	WHY: K3s has its own EMBEDDED Helm mechanism it uses internally 
	     to install default components (like Traefik) during its own 
	     startup — this happens automatically, before you ever install 
	     Helm yourself. Those "helm-install-traefik" pods seen earlier 
	     (Task 6) were K3s doing this. Your own "helm list -A" here is 
	     the first time YOU had visibility into that pre-existing state, 
	     since Helm wasn't installed on your user account until now.
	     Nothing was manually installed by mistake — this is expected 
	     K3s behaviour, separate from your own Helm installation.







