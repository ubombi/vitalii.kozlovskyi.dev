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

# default value for variable/argument
loglevel=${1:-info} # ${name:-value}


```

### Git
```bash
# Always use ssh keys instead of asking for a password.
git config --global url.ssh://git@github.com/.insteadOf https://github.com/

# Commit signing
git config --global user.signingkey SOMEKEYID! # Choose Key; note ending `!`
git config --global commit.gpgsign true # Auto sign without `-S`
# note: You can set up different keys per each repo.

# Of course profile
git config --global user.email "your@email.com"
git config --global user.Name "Name Surname"

git config --global pull.rebase true # Rebase own changes on never commit from upstream
# git config --global fetch.prune true # In case you need it

# Other
git rebase -i HEAD~9 # Squash / Edit lasn N=9 commits
git add -p # last check before commit.
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
>Before any operation with state make a backup, or use versioned S3 bucket.
>```bash
>terraform state pull > backup.tfstate
>```

During refactoring, or small restructure, renaming this can be useful
```bash
# check `terraform plan` and copy-paste SOURCE and DESTINATION paths from there.
terraform state mv [options] SOURCE DESTINATION
```

**[How to move resources between different terraform states]({{< ref "/posts/move-tf-resources-between-different-states.md" >}})**




### Docker

#### AWS ECR Access
```bash
# It's possible manually login, for one time use
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
