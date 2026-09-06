=== TASK 8: NFS AS DEFAULT KUBERNETES STORAGE CLASS ===

#=======Add the NFS provisioner's Helm chart repository
	helm repo add nfs-subdir https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
						# adds a new source Helm can install charts from
						  (like adding a new pacman mirror/repo)

#=======Refresh Helm's local list of available charts
	helm repo update
						# pulls latest chart info from all added repos

#=======Install the NFS provisioner onto the cluster
	helm install nfs-storage nfs-subdir/nfs-subdir-external-provisioner \
	  --set nfs.server=127.0.0.1 \
	  --set nfs.path=/srv/nfs/nextcloud
						# nfs-storage = name given to this Helm release
						# nfs-subdir/nfs-subdir-external-provisioner = chart being installed
						# --set nfs.server=127.0.0.1 = NFS server is this same VM (localhost)
						# --set nfs.path=/srv/nfs/nextcloud = exported folder from Task 5

	VERIFY: sudo watch k3s kubectl get pods -A
	LOOK FOR: new pod "nfs-storage-nfs-subdir-external-provisioner-xxxxx" 
	          showing READY 1/1, STATUS Running
	(Ctrl+C to exit once confirmed)

#=======Set nfs-client as the DEFAULT storage class
	sudo k3s kubectl patch storageclass nfs-client -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
						# NOTE: PDF had a typo here ("is-defaultclass", 
						  missing hyphen) — corrected to "is-default-class"


+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
===============VERIFY OUTPUT: "storageclass.storage.k8s.io/nfs-client patched"=====================
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++


#=======Demote local-path from being the default
	sudo k3s kubectl patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
	VERIFY OUTPUT: "storageclass.storage.k8s.io/local-path patched"

#=======Confirm the switch took effect
	sudo k3s kubectl get storageclass
	LOOK FOR: 
	  - "nfs-client (default)"  <- must show "(default)" tag next to it
	  - "local-path"            <- must NOT show "(default)" tag anymore

	RESULT CONFIRMED:
	NAME                   PROVISIONER                                    DEFAULT?
	local-path             rancher.io/local-path                          no
	nfs-client (default)   cluster.local/nfs-storage-nfs-subdir-...       YES

	MEANING: Any new persistent storage requests from now on (including 
	NextCloud's database + files) will automatically use your NFS share 
	at /srv/nfs/nextcloud instead of local disk.











