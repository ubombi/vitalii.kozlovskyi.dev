---
date: 2021-12-06T18:01:34+01:00
ShowToc: true
TocOpen: true
Type: "posts"
title: "Move Resources Between Different TF States"
summary: "Copy-paste example of terraform states manipulation."
---

>**TL;DR:** It's possible, however dangerous. Do it in case you ~~already f\*cked up~~ really need it. 

> **Note:** It may be safer and faster to `terraform state rm` and `terraform state import`, for small changes.
> This article can have mistakes.

A state is just a JSON file, usually stored in a cloud. **You should never edit `.tfstate` files manually.**
Use terraform provided commands instead.


## How to

### Preparation (safe)
`terraform state pull` would fetch state from remote storage and output it as STDOUT. If you use remote state, local file would be empty. 

#### Fetch state
```bash
# Here we put evetyrhing into home folder
# cd ~/tf/source_dir
terraform state pull > ~/source.tfstate
# cd ~/tf/destination_dir
terraform state pull > ~/destination.tfstate
```

Now we have two backuped states. No risks so far.

#### Doublecheck exported state files
To avoid any conflicts with **remote** backend, use different folder.
```bash
terraform state list -state=~/source.tfstate # | grep 'mysupermodule'
terraform state list -state=~/destination.tfstate
```

### Move state between backups (safe)
The following command does not change the infrastructure. Here we would move JSON state between local files (no errors expected). 
```bash
# terraform state mv [options] SOURCE DESTINATION
# where options are `-state` and `-state-out`; Also `-lock=false` may be required 
terraform state mv -state=~/source.tfstate -state-out=~/destination.tfstate module.api.module.mysupermodule module.mysupermodule
```
Doublecheck states again. If the result is unexpected, start from the beginning.
```bash
terraform state list -state=~/source.tfstate # | grep 'mysupermodule'
terraform state list -state=~/destination.tfstate
```

### Update sources (safe)
Create/delete modules and resources to reflect same changes. 

### Optional check for paranoid (kinda safe)
{{% collapse summary="I'm scared, show it anyway!" %}} 
With **local** backend only P3 is required.
But with **remote** backend `-state` flag can not be used, so 

1. Edit backend in source code:
```terraform
terraform {
  required_version = "~> 0.14.0"
  # Comment any remote backend definition
  # backend "s3" {
  #   bucket = "here-i-store-my-states"
  #   key    = "example/terraform.tfstate"
  # }

  # Add local backend with path state you want to test
  backend "local" {
    path = "~/source.tfstate"
  }
}
```
2. Then init terraform. It would ask if you want to move state, answer No.
```bash
terraform init # Don't move the state
```
3. Now having local backend we can specify `-state` flag. (P3)
```bash
terraform plan -state=~/source.tfstate -refresh=false
```
4. **Undo changes made in step 1 and init terraform again**
{{% /collapse %}}


### Apply changes (dangerous)
If everything is as you expected, we need to push state. _(always do backups)_

```bash
# cd ~/tf/source_dir
terraform state push ~/source.tfstate
# cd ~/tf/destination_dir
terraform state push ~/destination.tfstate
```

**Check results with `terraform plan` in respective directories.**

Do not forget to commit and push the code.
