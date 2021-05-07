## Related Issues
* [#3940](https://github.com/rook/rook/issues/3940#issuecomment-744512688)
* [#4553](https://github.com/rook/rook/issues/4553#issuecomment-600587434)

## For China Users
The yaml file `operator.yaml` is customized for China users with specific image repositories.

## Order
```sh
kubectl apply -f common.yaml
kubectl apply -f operator.yaml
kubectl apply -f crds.yaml
kubectl -n rook-ceph create secret generic rook-ceph-crash-collector-keyring
kubectl apply -f cluster.yaml
```
