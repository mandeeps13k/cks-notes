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

## CIS

- Always check Ownership of `/var/lib/etcd` Directory -> `chown etcd:etcd /var/lib/etcd`
- Check permissions for Kubelet Config -> `chmod 644 /var/lib/kubelet/config.yaml`
- Make Sure AnonymousUser is marked as `false` in kubelet config `/var/lib/kubelet/config.yaml`
``` 
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
```
- Make Sure authorization Mode is set to Node,RBAC in `kube-apiserver`

```
--authorization-mode=Node,RBAC
```



## Restricting Syscalls (seccomp)

- check Seccomp support -> `grep -i seccomp /boot/config-$(uname -r)`
- check seccomp implemented on container -> `cat /proc/1/status | grep Seccomp`
- Seccomp modes -> 1: DISABLED , 2: STRICT, 3: FILTERED
- Seccomp in Pod Spec, `localHostProfile` path is relative to default seccomp path `/var/lib/kubelet/seccomp`
- seccomp config in Pod Spec 
```
apiVersion: v1
kind: Pod
spec:
  securityContext:
     seccompProfile:
       type: Localhost
       localhostProfile: profiles/audit.json
```

## AppArmor

- `systemctl status apparmor` ; check if apparmor module is enabled `cat /sys/module/apparmor/parameters/enabled`
- AppArmor profiles - `cat /sys/kernel/security/apparmor/profiles` ; `cat /sys/kernel/security/apparmor/profiles | grep <profile-name>`
- check appArmor status - `aa-status`
- Load AppArmor profiles -> `apparmor_parser -q /etc/apparmor.d/usr.sbin.nginx-updated`
- check loaded AppArmor profile status -> `aa-status | grep <profile-name>`
- appArmor config is for every container and should be specified for container.
- get capabilities of a process -> get process id using `ps -ef | grep /usr/sbin/sshd` then use `getpcaps <PID>`
- Apply AppArmor profile to a Pod by using the below Annotation on the Pod:
```
annotations:
    container.apparmor.security.beta.kubernetes.io/<container_name>: <profile_ref>
```


## Admission Controllers

- View enabled Admission Controllers -> `kube-apiserver -h | grep enable-admissions-plugins`
- In Kubeadm run this command inside the kube-apiserver-controlplane POD, use this command -> `kubectl exec kube-apiserver-controlplane -n kube-system -- kube-apiserver -h | grep enable-admissions-plugins`
- To enable and Disable plugins, use `--enable-admission-plugins` and `--disable-admission-plugins` in kube-apiserver config


## ImagePolicyWebhook

- Enable in `kube-apiserver.yaml`
```
- --admission-control-config-file=/etc/kubernetes/admission-controllers/admission-configuration.yaml
- --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
```

- Always Check and verify that `admission-configuration` file  has been mounted in `kube-apiserver.yaml`.
```
volumeMounts:
  - mountPath: /etc/kubernetes/admission-controllers
      name: admission-controllers
      readOnly: true
volumes:
- hostPath:
     path: /etc/kubernetes/admission-controllers/
     type: DirectoryOrCreate
     name: admission-controllers
```

## OPA - Open Policy Agent

- Load Rego policy - `curl -X PUT --data-binary @sample.rego http://localhost:8181/v1/policies/samplepolicy` where `sample.rego` is the file containing policies.
- OPA Rego Policy is configured as a `ConfigMap` Object on the Pod. Make sure to extract the configMap output in YAML to investigate the ConfigMap.

## Falco

- To create a rule first check `/etc/falco/falco_rules.yaml` copy the rule from this file with search the rule syntax 
```
- rule: Launch Package Management Process in Container
  desc: Package management process ran inside container
  condition: >
    spawned_process
    and container
    and user.name != "_apt"
    and package_mgmt_procs
    and not package_mgmt_ancestor_procs
    and not user_known_package_manager_in_container
  output: >
  Package management process launched in container (user=%user.name user_loginuid=%user.loginuid
  command=%proc.cmdline container_id=%container.id container_name=%container.name
image=%container.image.repository:%container.image.tag)
  priority: ERROR
  tags: [process, mitre_persistence]
```

- Always create a new rule according to the data in `/etc/falco/falco_rules.local.yaml`.
- Run `service falco restart` OR `systemctl restart falco`

## Immutable File System

```
readOnlyRootFilesystem: true
privileged: false
```

## Encrypting Secrets at Rest

- Enable secret encrpytion on `kube-apiserver` by specifying flag `--encryption-provider-config` and passing a `EncryptionConfiguration` Config file.
- Make sure `EncryptionConfiguration` File is mounted as a volume on the kube-apiserver Pod.

## Image Security

- Use `imagePullSecrets:` in Pod Spec to use a Private Image Registry.
- For restricting sources for registry, use `ImagePolicyWebhook` Admission Controller.
- Reference the `ImagePolicyWebhook` configuration file from the file provided to the API server's command line flag `--admission-control-config-file`.
- Use trivy to scan Images 

## Trivy

- Use this Command to get desired Output with no. of Vulnerabilities :
```
trivy image -s HIGH,CRITICAL nginx | grep -i total
```

- Use this Command to List Down all the Image Names along with the existing Pods :
```
kubectl get pods --namespace default --output=custom-columns="NAME:.metadata.name,IMAGE:.spec.containers[*].image"
```


## Audit Log

- To enable Audit Log, use flag `--audit-log-path=/var/log/k8-audit.log` on `kube-apiserver` where the audit events are getting written to the the file `/var/log/k8-audit.log`
- To Make use of the Audit Policy file, pass the flag `--audit-policy-file=<path-to-file>` on the `kube-apiserver`
- Audit Log File -> `/etc/kubernetes/audit/polcy.yaml`
```
rules:
  # Log pod changes at RequestResponse level
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["pods"]
# A catch-all rule to log all other requests at the Metadata level.
  - level: Metadata
    omitStages:
      - "RequestReceived"
# log Secret resources audits, level Metadata 
  - level: Metadata
     resources:
     - group: ""
       resources: ["secrets"]
# for everything else don't log anything
   - level: None
```

- kube-api configurations as maxage=30 only 5 backups path `/var/log/Kubernetes/audit/audit.log`

```
- --audit-policy-file=/etc/kubernetes/audit/polcy.yaml
- --audit-log-path=/var/log/kubernetes/audit/audit.log
- --audit-log-maxage=30
- --audit-log-maxbackup=5
```

- Check the mounts for file and log if not present then add

```
volumeMounts:
  - mountPath: /etc/kubernetes/audit/audit-policy.yaml
    name: audit
    readOnly: true
  - mountPath: /var/log/kubernetes/audit/
    name: audit-log
    readOnly: false
volumes:
- name: audit
  hostPath:
    path: /etc/kubernetes/audit/audit-policy.yaml
    type: File

- name: audit-log
  hostPath:
    path: /var/log/kubernetes/audit/
    type: DirectoryOrCreate
```


