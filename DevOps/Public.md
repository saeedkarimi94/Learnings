# دستورات عمومی سرور:

## اتصال سرور لینوکسی به شکن:
```bash
sudo nano /etc/systemd/resolved.conf
```
```bash
[Resolve]
DNS=178.22.122.101 185.51.200.1
Domains=~.
```
```bash
sudo systemctl restart systemd-resolved
```
## دانلود پکیج‌های نوگت


## پوش کردن پکیج‌های نوگت:
```bash
mkdir ./nugetpackage
```
```bash
vim Nuget.config
```
```bash
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="PDNNuget"
         value="http://192.168.57.50:8081/repository/nuget-group/index.json"
         allowInsecureConnections="true" />

    <add key="PDNNugetPush"
         value="http://192.168.57.50:8081/repository/nuget-hosted/"
         allowInsecureConnections="true" />
  </packageSources>
</configuration>
```

---