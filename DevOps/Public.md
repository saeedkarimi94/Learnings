# دستورات عمومی سرور:

## اتصال سرور لینوکسی به شکن:
```bash
sudo nano /etc/systemd/resolved.conf
```
```bash
[Resolve]
DNS=178.22.122.100 185.51.200.2
Domains=~.
```