# Install, Configuration and Validation:
برای راه‌اندازی کوبرنتیز به حداقل ترین سیستم مورد نیاز 1 Master و 1 Worker نیازداریم.

## **Install and Configure**:

* **OS**: Debian or ubuntu
* **RAM**: minimum 2GB، این مقدار به ازای هر نود می‌باشد و حداقل ترین میزان می‌باشد.
* **CPU**: 2Core، این مقدار هم به ازای هر نود می‌باشدیعنی اگر یک Master و یک Worker داریم باید 4Core استقاده کنیم.
* **Network**: در بحث شبکه حتما باید هواسمان باشد که هر دو Node در بک شبکه باشند.
* **تنظیمات سرور**: در تنظیمات مربوط به سرور حتماً بایدموارد زیر در همه نودها یکسان باشد:
- ▪ Uniqe hostname
- ▪ MAC address
- ▪ product_uuid
* **swap disable**: به دلیل اینکه kubelet با Swap compatibility ندارد باید این مورد خاموش باشد.
* **Set Time Zone**: باید در همه نودها TimeZone یکی باشد (Asia/Tehran)
* **static IP**:روی همه نودها باید static IP ست کنیم چرا که در صورتی که آی پی سرور عوض بشه همه کانتینرهای کوبر از بین خواهند رفت.
* **DNS**: بهتر است برای بالاآوردن کلاسترهای کوبر از یک فیلتر شکن استفاده کنیم.
---

