# cks-notes

## Kubelet Config

- flag `--anonymous-auth=false` to be to set in Kubelet Config file to prevent UnAuthorized access to the API. by default, all access is Allowed.
- flag `--authorization-mode` needs to be set. By default All Access is Allowed.
- flag `--read-only-port` needs to be set to 0. By default it is set to `10255` and it exposes metrics over the API. Set `--read-only-port` to `0` to disable it.
