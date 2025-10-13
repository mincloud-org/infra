# infra
## unlock f
`kubectl patch application <pod name>  -n <name space> -p '{"metadata":{"finalizers":[]}}' --type=merge`

## delete all 
`kubectl delete all --all -n <name space>`