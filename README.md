# infra
## unlock f
`kubectl patch application <app name>  -n <name space> -p '{"metadata":{"finalizers":[]}}' --type=merge`

## delete all 
`kubectl delete all --all -n <name space>`