## Local Port forwarding
- 🥷**Attacker IP**: 10.10.14.38
- 💻**Remote host IP**: 10.10.15.130 & 10.10.16.130 (internal network)
- 🎯**Targeted server IP**: 10.10.16.131 (internal network IP)
- 🏳️**Objective**: Access internal website (http://10.10.16.131:80/) via http://127.0.0.1:8443/
```bash
ssh -L 8443:10.10.16.131:443 user@10.10.15.130 # -i id_rsa to use private key
ssh -L <attacker_host_listening_port>:<internal_target_IP>:<internal_target_port> user@10.10.15.130
```
## Remote Port forwarding
- 🥷**Attacker IP**: 10.10.14.38
- 💻**Remote Pivot IP**: 10.10.15.130 & 10.10.16.130 (internal network)
- 🎯**Targeted server IP**: 10.10.16.131 (internal network IP)
- 🏳️**Objective**: Reverse shell come back to my attack host through pivot host
```bash
ssh -R 9001:localhost:9001 user@10.10.15.130 # -i id_rsa to use private key
ssh -R <pivot_listening_port>:<attacker_host_IP>:<attacker_host_listening_port> user@10.10.15.130
```
## Dynamic Port forwarding
- 🥷**Attacker IP**: 10.10.14.38
- 💻**Remote Pivot IP**: 10.10.15.130 & 10.10.16.130 (internal network)
- 🎯**Targeted server IP**: 10.10.16.131 (internal network IP)
- 🏳️**Objective**: Forward all my traffic through pivot host e.g: `proxychains nxc smb 10.10.16.131`
```bash
ssh -D 9050 user@10.10.15.130
ssh -D <attacker_listening_port> user@10.10.15.130
```
### File /etc/proxychains.conf - configuration
```conf
[ProxyList]
socks5 127.0.0.1 9050
```
