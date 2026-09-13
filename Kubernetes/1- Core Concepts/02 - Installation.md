# Install, Configuration and Validation:
برای راه‌اندازی کوبرنتیز به حداقل ترین سیستم مورد نیاز 1 Master و 1 Worker نیازداریم.

## **Install and Configure**:

* **OS**: Debian or ubuntu
* **RAM**: minimum 2GB، این مقدار به ازای هر نود می‌باشد و حداقل ترین میزان می‌باشد.
* **CPU**: 2Core، این مقدار هم به ازای هر نود می‌باشدیعنی اگر یک Master و یک Worker داریم باید 4Core استقاده کنیم.
* **Network**: در بحث شبکه حتما باید هواسمان باشد که هر دو Node در بک شبکه باشند.
* **Server Setting**: در تنظیمات مربوط به سرور حتماً باید موارد زیر در همه نودها یکسان باشد:
    - Unique hostname
    - MAC address
    - product_uuid
* **swap disable**: به دلیل اینکه kubelet با Swap compatibility ندارد باید این مورد خاموش باشد.
* **Set Time Zone**: باید در همه نودها TimeZone یکی باشد (Asia/Tehran)
* **static IP**:روی همه نودها باید static IP ست کنیم چرا که در صورتی که آی پی سرور عوض بشه همه کانتینرهای کوبر از بین خواهند رفت.
* **DNS**: بهتر است برای بالاآوردن کلاسترهای کوبر از یک فیلتر شکن استفاده کنیم.
* **نکته**: فضاهای اختصاص داده شده بستگی به بزرگی کلاستر دارد، مقادیر داده شده همه پیشفرض و فقط برای راه‌اندازی می‌باشد.

## **Port and Protocols**:

* هنگام اجرای کوبرنتیز در محیطی با مرزهای شبکه‌ای سختگیرانه، مانند مرکز داده داخلی با فایروال‌های شبکه فیزیکی یا شبکه‌های مجازی در فضای ابری عمومی، آگاهی از پورت‌ها و پروتکل‌های مورد استفاده توسط اجزای کوبرنتیز مفید است. وپورت‌های مورد نیاز کوبرنتیز به شرح زیر است:
| Protocol | Direction | Port Range | Purpose                 | Used By            |
|----------|-----------|------------|-------------------------|--------------------|
| TCP      | Inbound   | 6443       | Kubernetes API server   | All                |
| TCP      | Inbound   | 2379-2380  | etcd server client API  | kube-apiserver, etcd |
| TCP      | Inbound   | 10250      | Kubelet API             | Self, Control plane |
| TCP      | Inbound   | 10259      | kube-scheduler          | Self               |
| TCP      | Inbound   | 10257      | kube-controller-manager | Self               |

    - 6443: پورت پیشفرض Kubernetes API server
    - 2379-2380: etcd server client API
    - 10250: Kubelet API
    - 10259: kube-scheduler
    - 10257: kube-controller-manager

---

