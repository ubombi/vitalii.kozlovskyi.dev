---
title: "CheatSheet"
date: 2021-11-22T01:18:48+01:00
type: "page"
menu:
  main:
    title: "CheatSheet"
ShowToc: true
TocOpen: true
---

### Bash
```bash
# Run command if binaty exists without additional output
command -v echo &> /dev/null && echo "works"
command -v thefuck &> /dev/null && eval $(thefuck --alias)

```

### Git
```bash
# Always use ssh keys instead of asking for a password.
git config --global url.ssh://git@github.com/.insteadOf https://github.com/
```

### Yubikey (hardware keys in general)
```
# OTP, OATH does not work unless pcscd is running
sudo systemctl enable --now pcscd.service

# Get oath OTP by name (substring match) into clipboard
ykman oath accounts code -s OpenVPN | xclip -selection clipboard
```


### Infra
#### K8S
Don't forget about official [kubectl CheatSheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/).
```
# Prefered way is `view-secret` krew plugin for kubectl.
kubectl get secret database-credentials -o jsonpath='{.data.password}' | base64 -d | xclip -selection clipboard

# This one is important :)
kubectl logs my-pod --previous

```

#### Terraform
**Never edit state file manually!**

##### Partial apply
When someone forgot to push changes, or terraform shows alot of unrelated changes, and there is no time to dig into it, it's possible to apply specific resources via `-target`. [Docs](https://learn.hashicorp.com/tutorials/terraform/resource-targeting?in=terraform/cli)
```bash
# can check existing resources with `terraform state list`
terraform apply -target="module.some_resource.name[42]" -target="another_resource"
```
Command would return partial output, but `terraform output` would work as expected, returning full state of outputs.

##### Move state
Before any operation with state make a backup, or use versioned S3 bucket.
```bash
# Pull from remove (local) to STDOUT
terraform state pull > backup.tfstate
```


During refactoring, or small restructure, renaming this can be useful
```bash
# check `terraform plan` and copy-paste SOURCE and DESTINATION paths from there.
terraform state mv [options] SOURCE DESTINATION
```

##### Move state between different state files
Hovever, in case if we fu\*ked up and need to move state, things get a bit more complicated
```
# TBD;
```



### Docker

#### AWS ECR Access
```bash
# It's possible manualy login, for one time use
docker login -u AWS -p $(aws ecr get-login-password) https://{account}.dkr.ecr.{region}.amazonaws.com
```

But the right way to do it is [ecr-login](https://github.com/awslabs/amazon-ecr-credential-helper). Add this into `~/.docker/config.json`
```json
{
	"credHelpers": {
		"{account}.dkr.ecr.{region}.amazonaws.com": "ecr-login"
	}
}
```
