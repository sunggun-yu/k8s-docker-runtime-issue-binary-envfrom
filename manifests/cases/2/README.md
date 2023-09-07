# RunContainerError with smaller size of data

Test with smaller size of Secret data

## Condition

- Secret
- secret.png:  16087 bytes
- envFrom: secretRef

## Result

Failed with the following error message but not frozen container

```txt
NAME                              READY   STATUS              RESTARTS      AGE
case2-helloweb-778ff96d9d-hh8ss   0/1     RunContainerError   1 (15s ago)   16s
```

```log
Warning  Failed     11s (x5 over 104s)  kubelet            Error: failed to start container "hello-app": Error response from daemon: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: environment variable value can't contain null(\x0 ││ 0): "secret.png=�PNG\r\n\x1a\n\x00\x00\x00\rIHDR\x00\x00\x00M
```
