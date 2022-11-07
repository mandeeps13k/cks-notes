# cks-notes

## Kubelet Config

- flag `--anonymous-auth=false` to be to set in Kubelet Config file to prevent UnAuthorized access to the API. by default, all access is Allowed.
- flag `--authorization-mode` needs to be set. By default All Access is Allowed.
- flag `--read-only-port` needs to be set to 0. By default it is set to `10255` and it exposes metrics over the API. Set `--read-only-port` to `0` to disable it.

## commands

- kubectl port-forward service/nginx 28080:80
- Run `kubectl proxy &` 


## Restrict Kernel Modules

- Include information in `/etc/modprobe.d/blacklist.conf`
- cat /etc/modprobe.d/blacklist.conf -> blacklist sctp 
                                        blacklist dccp
                                        
 ## Network
 
 - Open ports -> `netstat -an | grep -w LISTEN`
 - Checking used ports by services -> `cat /etc/services | grep -w 53` ; `netstat -natp | grep 9090`

## Services

- List All Services -> `cat /etc/services`
- `cat /lib/systemd/system/nginx.service`

## Restricting Syscalls (seccomp)

- check Seccomp support -> `grep -i seccomp /boot/config-$(uname -r)`
- check seccomp implemented on container -> `cat /proc/1/status | grep Seccomp`
- Seccomp modes -> 1: DISABLED , 2: STRICT, 3: FILTERED
- Seccomp in Pod Spec, `localHostProfile` path is relative to default seccomp path `/var/lib/kubelet/seccomp`

## AppArmor

- `systemctl status apparmor` ; check if apparmor module is enabled `cat /sys/module/apparmor/parameters/enabled`
- AppArmor profiles - `cat /sys/kernel/security/apparmor/profiles` ; `cat cat /sys/kernel/security/apparmor/profiles | grep <profile-name>`
- check appArmor status - `aa-status`
- Load AppArmor profiles -> `apparmor_parser /etc/apparmor.d/usr.sbin.nginx-updated`
- check loaded AppArmor profile status -> `aa-status | grep <profile-name>`
- get capabilities of a process -> get process id using `ps -ef | grep /usr/sbin/sshd` then use `getpcaps <PID>`
