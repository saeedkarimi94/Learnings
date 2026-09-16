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

## فاز ۱: کارهای مشترک (باید روی همه سرورها اجرا شود)

### ۱. تنظیمات شبکه کرنل (Kernel Modules & Sysctl)
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

# تولید فایل کانفیگ پیش‌فرض
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
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

# اضافه کردن کلید امنیتی و مخزن کوبرنتیز
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
# قفل کردن نسخه‌ها برای جلوگیری از آپدیت ناخواسته
sudo apt-mark hold kubelet kubeadm kubectl
```
## فاز ۲: کارهای اختصاصی Master (Control Plane)

### ۱. مقداردهی اولیه کلاستر (kubeadm init)
این دستور را فقط روی سرور Master می‌زنیم. رنج شبکه پادها (pod-network-cidr) برای پلاگین شبکه (مثل Calico یا Flannel) مشخص می‌شود:
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<IP_MASTER_NODE>
```
* **نکته**: در پایان خروجی این دستور، یک خط شامل kubeadm join ... --token ... به شما می‌دهد. آن را در جایی کپی و ذخیره کنید!
### ۲. تنظیم دسترسی kubectl برای کاربر عادی
برای اینکه بتوانید با دستور kubectl با کلاستر کار کنید:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### ۳. نصب پلاگین شبکه پادها (CNI Plugin)
تا وقتی پلاگین شبکه نصب نشود، نودها در حالت NotReady می‌مانند چون پادها نمی‌توانند به هم آی‌پی بدهند. ما از Flannel یا Calico استفاده می‌کنیم (مثال با Flannel):
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
اکنون اگر دستور زیر را بزنید وضعیت مستر باید Ready شود:
```bash
kubectl get nodes
```
## فاز ۳: کارهای اختصاصی Workerها (Join کردن)
حالا وارد سرورهای Worker می‌شویم و دستوری که در خروجی kubeadm init دریافت کرده بودیم را با sudo اجرا می‌کنیم:
```bash
sudo kubeadm join <IP_MASTER_NODE>:6443 --token <TOKEN> \
--discovery-token-ca-cert-hash sha256:<HASH>
```
* **نکته**: توکن‌های پیش‌فرض kubeadm فقط ۲۴ ساعت اعتبار دارند و بعد از آن منقضی (Expire) می‌شوند.
اگر بعداً خواستی یک Worker جدید اضافه کنی یا دستور Join را گم کردیم، کافی است بروی روی سرور Master (Control Plane) و یکی از دو روش زیر را انجام دهیم:
1. روش اول، ساده‌ترین و تمیزترین راه
 روی نود Master دستور زیر را بزن:
```bash
kubeadm token create --print-join-command
```
کارکردش چیه؟
این دستور یک توکن معتبر جدید می‌سازد و دستور کامل kubeadm join ... را با تمام مقادیر (--token و --discovery-token-ca-cert-hash) کف ترمینال تحویلت می‌دهد! دقیقاً همان را کپی می‌کنی و روی Worker جدید اجرا می‌کنی.

2. **روش دوم: مرحله‌به‌مرحله و دیدن توکن‌ها (برای درک عمیق‌تر)**: اگر بخواهی ببینی چه توکن‌هایی الآن فعال هستند یا دستی مقادیر را درآوری:
* دیدن وضعیت توکن‌های فعلی روی Master:
```bash
kubeadm token list
```
اگر توکنی بود و وضعیتش معتبر (TTL داشت)، می‌توانی از همان استفاده کنی. اگر منقضی شده بود، یک توکن جدید می‌سازی:
```bash
kubeadm token create
```
این به تو یک استرینگ مثل abcdef.0123456789abcdef می‌دهد.

* درآوردن هش سرتیفیکیت (CA Cert Hash):
این مقدار تغییر نمی‌کند، ولی اگر نداشتی با این دستور روی Master محاسبه‌اش می‌کنی:
```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
  openssl rsa -pubin -outform der 2>/dev/null | \
  openssl dgst -sha256 -hex | sed 's/^.* //'
```
* سرهم کردن دستور Join:
```bash
sudo kubeadm join <IP_MASTER>:6443 --token <توکن_مرحله_اول> --discovery-token-ca-cert-hash sha256:<هش_مرحله_دوم>
```



## فاز ۴: اعتبارسنجی (Validation)
```bash
kubectl get nodes
```
خروجی باید چیزی شبیه به این باشد و وضعیت همه نودها Ready شده باشد:
```bash
NAME       STATUS   ROLES           AGE   VERSION
master     Ready    control-plane   10m   v1.30.x
worker-1   Ready    <none>          2m    v1.30.x
worker-2   Ready    <none>          2m    v1.30.x
```

# Installing kubeadm, kubelet and kubectl
جهت کار با کوبرنتیز نیازمند نصب بسته‌های زیر می‌باشیم:

1. **kubeadm**: دستوری برای بوت‌استرپ کردن کلاستر.
2. **kubelet**: کامپوننتی که روی تمام ماشین‌های موجود در کلاستر شما اجرا می‌شود و کارهایی مانند راه‌اندازی پادها و کانتینرها را انجام می‌دهد.
3.  **kubectl**: ابزار خط فرمان برای ارتباط با کلاستر شما.



---