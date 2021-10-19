### Setup
- 2 YubiKey Hardware Keys [K1, K2]
- Root offline key for CA. Encrypted, backuped and saved in a safe places.
  - subkey S1 for signing + auth _-> K1_
  - subkey E1 for encryption _-> K1
  - subkey S2 for signing + auth _-> K2
  - subkey E2 for encryption _-> K2_

Root key is not stored on any of hardware keys. Only subkeys.

K2 is used as **backup for 2FA**, and additional set of keys is kinda bonus feature. 
While K1, for passwordless auth and git commit signing.


### Pitfalls 
#### Git
**TL;DR:** Use `!` after key ID in git config. Like `git config --global user.signingkey 00ABCD77FF00FF!`.


Having that setup, I found that git/gpg always asks me to insert main YubiKey, regardless of selected Key ID.
```bash
# configure git to use subkey S1
git config --global user.signingkey SOMES1ID
# check if works
git commit -m "test signing" -S
```

Surprisingly GPG asked to insert K2 instead of K1. _Why?.. Whatever..._ 
```bash
# check signature
git log --show-signature -n1 
```
Looks signed, but wait. Why it used the wrong key?
```
gpg: Signature made Tue 19 Oct 2021 01:35:37 PM CEST
gpg:                using EDDSA key S2   <---!!!!!!!!!!
gpg: Good signature from "Vitalii Kozlovskyi <ubombi@gmail.com>" [ultimate]
gpg:                 aka "Vitalii Kozlovskyi <vitalii@kozlovskyi.dev>" [ultimate]
```
And git is not one to blame. All is does, is passes `user.signingkey` value as `gpg --local-user`. 
But how to force gpg to use specific subkey? (it only happens with subkeys)



