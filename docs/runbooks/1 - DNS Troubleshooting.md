#1 == Where could DNS actually fail on you?

	1. systemd-resolved doesn't survive reboot correctly
	2. The symlink gets overwritten
	3. The upstream DNS server (172.16.10.53) becomes unreachable
	4. Some random shit happens.

#1 SOLUTION:
	1.sudo reboot
	2. systemctl status systemd-resolved
		# Active: active(running)=GOOD. Inactive(dead)=BAD. Disabled = Need Enabling. It means it 
		won't auto-start on boot even if currently running manually
	3. cat /etc/resolv.conf
		# managed by man:systemd-resolved(8)
		  nameserver 127.0.0.53
		  nameserver 172.16.10.53
	4. resolvectl status
		# DNS Servers: populated with real IPs (yours: 172.16.10.52, 172.16.10.53)
		  Current DNS Server: (will show server)
		  Fallback DNS Servers: backups (Quad9/Cloudflare/Google in your case)
	5. ping -c N <hostname>