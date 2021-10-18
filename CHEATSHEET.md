### Yubikey (hardware keys in general)
```
# OTP, OATH does not work unless pcscd is running
sudo systemctl enable --now pcscd.service

# Get oath OTP by name (substring match) into clipboard
ykman oath accounts code -s OpenVPN | xclip -selection clipboard
```
