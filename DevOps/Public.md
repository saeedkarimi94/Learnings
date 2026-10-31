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
---
## دانلود پکیج‌های نوگت
### روش اول:
```bash
set -e

# Format: "package_name:version"
packages=(
  "microsoft.aspnetcore.authentication.jwtbearer:10.0.11"
  "microsoft.aspnetcore.openapi:10.0.11"
  "microsoft.entityframeworkcore.sqlserver:10.0.11"
  "microsoft.entityframeworkcore.tools:10.0.11"
  "microsoft.entityframeworkcore.design:10.0.11"
  "swashbuckle.aspnetcore:10.2.3"
  "microsoft.data.sqlclient:6.1.6"
  "system.configuration.configurationmanager:9.0.13"
  "system.security.cryptography.pkcs:9.0.13"
)

for item in "${packages[@]}"; do
  pkg="${item%%:*}"
  ver="${item##*:}"
  echo "Downloading $pkg ($ver)..."
  curl -fSL -o "${pkg}.${ver}.nupkg" "https://www.nuget.org/api/v2/package/${pkg}/${ver}"
done

echo "All packages downloaded successfully!"
```
### روش دوم
```bash
curl -fSL -o microsoft.data.sqlclient.extensions.abstractions.7.0.1.nupkg "https://www.nuget.org/api/v2/package/Microsoft.Data.SqlClient.Extensions.Abstractions/7.0.1"
```
---
## پوش کردن پکیج‌های نوگت در نکسوز:
* ایجاد فایل زیر در مسیر پکیج‌ها:
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
* و مرحله بعد پوش کردن در نکسوز:
```bash
dotnet nuget push "*.nupkg" -k a0e46a9c-7a15-32ac-83c4-4005d39b60d5 -s http://192.168.57.50:8081/repository/nuget-hosted/ --skip-duplicate --configfile Nuget.config
```

---
