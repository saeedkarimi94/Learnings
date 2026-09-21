<div dir="rtl" lang="fa">


# راهنمای جامع نصب و راه‌اندازی کلاستر کوبرنتیز (با kubeadm)

## حداقل مشخصات سخت‌افزاری و پیش‌نیازها:
برای راه‌اندازی کلاستر آزمایشی به حداقل **۱ نود Master** و **۱ نود Worker** نیاز داریم:
* **سیستم‌عامل:** Ubuntu 22.04 / 24.04 یا Debian 12
* **رم (RAM):** حداقل ۲ گیگابایت به ازای هر نود
* **پردازنده (CPU):** حداقل ۲ هسته به ازای هر نود
* **شبکه:** اتصال و دید مستقیم (L2/L3) بین تمام نودها با Static IP
* **یکتایی هویت:** مقادیر `hostname`، `MAC Address` و `product_uuid` باید روی همه ماشین‌ها یکتا باشد.
* **غیرفعال بودن Swap:** به دلیل مدیریت مستقیم حافظه توسط Kubelet، مقدار Swap باید خاموش باشد.
```bash
# خاموش کردن موقت Swap
sudo swapoff -a

# خاموش کردن دائمی Swap (کامنت کردن خط swap در fstab)
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# تنظیم منطقه زمانی
sudo timedatectl set-timezone Asia/Tehran
```

# Install, Configuration and Validation:
برای راه‌اندازی کوبرنتیز به حداقل ترین سیستم مورد نیاز 1 Master و 1 Worker نیازداریم.

## **Install and Configure**:

* **OS**: Debian or ubuntu
* **RAM**: minimum 2GB، این مقدار به ازای هر نود می‌باشد و حداقل ترین میزان است..
* **CPU**: 2Core، این مقدار هم به ازای هر نود می‌باشدیعنی اگر یک Master و یک Worker داریم باید 4Core استقاده کنیم.
* **Network**: در بحث شبکه حتما باید هواسمان باشد که هر دو Node در بک شبکه باشند.
* **Server Setting**: در تنظیمات مربوط به سرور حتماً باید موارد زیر در همه نودها یکتا باشد:چرا؟ چون کوبرنتیز هر نود را با همین‌ها شناسایی می‌کند. اگر دو ماشین MAC یکسان داشته باشند (که بعضی VMهای کپی‌شده این اتفاق می‌افتد!)، کلاستر گیج می‌شود.
    - Unique hostname
    - MAC address
    - product_uuid
* **swap disable**: چرا؟ چون kubelet (ایجنت روی هر نود) طوری طراحی شده که منابع را خودش مدیریت کند. اگر Swap روشن باشد، سیستم‌عامل ممکن است بخشی از حافظهٔ کانتینر را به دیسک بریزد و این باعث عدم پیش‌بینی‌پذیری منابع و کندی غیرمنتظره می‌شود. پس kubelet اصلاً با Swap فعال بالا نمی‌آید (به‌طور پیش‌فرض).
    - برای اینکه بفهمیم که این مورد روشن است یا خاموش از دستور free -mh استفاده می‌کنیم و برای انکه خاموش کنیم از دستور swapoff -a استفاده میکنیم، ولی این دستور بعد از ریستارت شدن سرور باز برمیگردد برای اینکه برای همیشه خاموش بماند فایل /etc/fstab را باز میکنیم و قسمت مربوط به swap را کامنت میکنیم.
* **Set Time Zone**: باید در همه نودها TimeZone یکی باشد (Asia/Tehran)
```bash
timedatectl set-timezone Asia/Tehran
```
* **static IP**:روی همه نودها باید static IP ست کنیم چرا که در صورتی که آی پی سرور عوض بشه همه کانتینرهای کوبر از بین خواهند رفت.
* **DNS**: بهتر است برای بالاآوردن کلاسترهای کوبر از یک فیلتر شکن استفاده کنیم.
* **نکته**: فضاهای اختصاص داده شده بستگی به بزرگی کلاستر دارد، مقادیر داده شده همه پیشفرض و فقط برای راه‌اندازی ابتدایی کوبرنتیز می‌باشد.

## **Port and Protocols**:

* هنگام اجرای کوبرنتیز در محیطی با مرزهای شبکه‌ای سختگیرانه، مانند مرکز داده داخلی با فایروال‌های شبکه فیزیکی یا شبکه‌های مجازی در فضای ابری عمومی، آگاهی از پورت‌ها و پروتکل‌های مورد استفاده توسط اجزای کوبرنتیز مفید است. وپورت‌های مورد نیاز کوبرنتیز به شرح زیر است:

### **Control Plane**:
| Protocol | Direction | Port Range | Purpose                 | Used By            |
|----------|-----------|------------|-------------------------|--------------------|
| TCP      | Inbound   | 6443       | Kubernetes API server   | All                |
| TCP      | Inbound   | 2379-2380  | etcd server client API  | kube-apiserver, etcd |
| TCP      | Inbound   | 10250      | Kubelet API             | Self, Control plane |
| TCP      | Inbound   | 10259      | kube-scheduler          | Self               |
| TCP      | Inbound   | 10257      | kube-controller-manager | Self               |

### **Worker Node(s)**:
| Protocol | Direction | Port Range | Purpose                 | Used By            |
|----------|-----------|------------|-------------------------|--------------------|
| TCP      | Inbound   | 10250      | Kubelet API             | All                |
| TCP      | Inbound   | 30000-32767| NodePort Services       | All                |

# معماری و نقشه راه نصب (با kubeadm)

فرآیند راه‌اندازی کلاستر به این صورت است که کارهای زیر را در چند فاز انجام می‌دهیم:
1. **آماده‌سازی همه نودها (Master و Workerها):** پیش‌نیازهای شبکه کرنل + نصب Container Runtime (containerd) + نصب ابزارهای Kubeadm/Kubelet.
2. **راه‌اندازی Control Plane (فقط روی Master):** اجرای دستور `kubeadm init` و راه‌اندازی شبکه پادها (CNI).
3. **پیوستن Workerها به کلاستر (فقط روی Workerها):** اجرای دستور `kubeadm join`.

## آماده‌سازی مشترک (روی همه نودها: Master و Workers)

### ۱. تنظیمات ماژول‌های کرنل و شبکه
برای اینکه ترافیک کانتینرها به درستی فوروارد شود و فایروال iptables بسته‌ها را ببیند:

### بارگذاری ماژول‌های کرنل
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### اعمال تنظیمات شبکه در کرنل
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```
### ۲. نصب Container Runtime (انتخاب ما: containerd)
کوبرنتیز برای اجرای کانتینرها به یک Runtime نیاز دارد. روش استاندارد استفاده از containerd است:
```bash
# نصب containerd از پکیج‌منیجر Ubuntu
sudo apt update
sudo apt install -y containerd

# ساخت کانفیگ پیش‌فرض و فعال‌سازی SystemdCgroup
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# ری‌استارت سرویس
sudo systemctl restart containerd
sudo systemctl enable containerd
```
* نکته بسیار مهم (تنظیم cgroup): چون لینوکس و کلاستر از systemd استفاده می‌کنند، باید به containerd هم بگوییم درایور cgroup را روی systemd بگذارد (وگرنه kubelet کرش می‌کند)

### ۳. نصب ابزارهای کوبرنتیز (kubelet, kubeadm, kubectl)
* این سه ابزار پایه‌ای را روی همه ماشین‌ها نصب می‌کنیم:

* **kubelet**: سرویسی که روی سرور می‌ماند و پادها را اجرا می‌کند.
* **kubeadm**: ابزار راه‌اندازی و جوین کردن کلاستر.
* **kubectl**: ابزار خط فرمان برای صحبت با کلاستر.
```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg

نکته: کلید امضای عمومی مخازن بسته Kubernetes را دانلود کنید. کلید امضای یکسانی برای همه مخازن استفاده می‌شود، بنابراین می‌توانید نسخه موجود در URL را نادیده بگیرید:
اگر دایرکتوری `/etc/apt/keyrings` وجود ندارد، باید قبل از دستور curl ایجاد شود، نکته زیر را بخوانید.

# اضافه کردن کلید امنیتی و مخزن کوبرنتیز
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# نصب و قفل نسخه پکیج‌ها
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
---