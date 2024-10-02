# Windows can't connect to SMB share

Sometimes you aren't able to access your SMB share until you've rebooted your PC. If you don't want to reboot, you can try the below:

1. Open Run (Win + R)
2. Search for `services.msc`
3. Restart the `Workstation` service.

Alternatively, you can do the below in Powershell:

```powershell
net stop Workstation
net start Workstation
```
