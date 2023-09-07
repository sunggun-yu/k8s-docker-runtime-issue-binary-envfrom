# Frozen container in Docker runtime

## Condition

- Secrets
- data size of secret.png: 50113 bytes
- envFrom: secretRef

## Result

`Kubelet` failed to get container due to frozen container and marks node `NotReady` status.

Read more [logs](./kubelet.log)

- Kubelet failed to inspect the container with `operation timeout` due to frozen container:

  ```log
  Sep 07 21:48:13 minikube kubelet[2199]: E0907 21:48:13.187571    2199 remote_runtime.go:326] "StartContainer from runtime service failed" err="rpc error: code = Unknown desc = failed to get container \"542bb5c0da31cbe7960c46b97a41edd087ca7dfb9aa1b37f27ef4fecc27721c6\" log path: failed to inspect container \"542bb5c0da31cbe7960c46b97a41edd087ca7dfb9aa1b37f27ef4fecc27721c6\": operation timeout: context deadline exceeded" containerID="542bb5c0da31cbe7960c46b97a41edd087ca7dfb9aa1b37f27ef4fecc27721c6"
  ```

- you can find the frozen container

  ```bash
  minikube ssh
  ```

  ```bash
  docker@minikube:~$ docker ps -a | awk '{print $1}' | while read ID; do echo $ID; docker inspect $ID >/dev/null; done
  CONTAINER
  Error: No such object: CONTAINER
  542bb5c0da31
  ```

- Node became `NotReady` status after couple minutes:

  ```log
  Sep 07 21:49:17 minikube kubelet[2199]: I0907 21:49:17.171114    2199 setters.go:548] "Node became not ready" node="minikube" condition={Type:Ready Status:False LastHeartbeatTime:2023-09-07 21:49:17.170974959 +0000 UTC m=+1962.581166852 LastTransitionTime:2023-09-07 21:49:17.170974959 +0000 UTC m=+1962.581166852 Reason:KubeletNotReady Message:PLEG is not healthy: pleg was last seen active 3m1.104514415s ago; threshold is 3m0s}
  ```
