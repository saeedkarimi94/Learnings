<div dir="rtl" lang="fa">



## فاز ۲: کارهای اختصاصی Master (Control Plane)

### ۱. مقداردهی اولیه کلاستر (kubeadm init)
این دستور را فقط روی سرور Master می‌زنیم. رنج شبکه پادها (pod-network-cidr) برای پلاگین شبکه (مثل Calico یا Flannel) مشخص می‌شود:
```bash
sudo kubeadm init --pod-network-cidr=10.10.0.0/16 --apiserver-advertise-address=<IP_MASTER_NODE>
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
پس از اجرای `kubeadm init`، کلاستر به یک پلاگین شبکه نیاز دارد تا پادها بتوانند با یکدیگر صحبت کنند و نودها به وضعیت `Ready` برسند. بر اساس سناریو و نیاز کلاستر، یکی از سه گزینه زیر را انتخاب و نصب کنید:
#### 🔹 گزینه اول: Flannel (ساده‌ترین گزینه - مناسب یادگیری و تست)
شبکه‌ای بسیار سبک و بدون پیچیدگی امنیتی:
```bash
# رنج CIDR پیشنهادی در init: 10.244.0.0/16
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
#### 🔹 گزینه دوم: Calico (استاندارد محیط‌های عملیاتی و Production)
پشتیبانی کامل از Network Policy و فایروال داخلی بین پادها:
اگر در سازمان نیاز داشته باشید که مشخص کنید «پاد A فقط بتواند به دیتابیس وصل شود و پاد B نتواند پاد C را پینگ کند» (اصطلاحاً Network Policy)، کالیکو استانداردترین و محبوب‌ترین گزینه پروداکشن در دنیاست.
* پیش‌نیاز در kubeadm init:
کالیکو به‌صورت پیش‌فرض رنج 192.168.0.0/16 را پیشنهاد می‌دهد:
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```
* دستورات نصب Calico:
مدرن‌ترین و استانداردترین روش نصب Calico با استفاده از Tigera Operator است (فقط ۲ دستور):
```bash
# گام ۱: نصب اپراتور کالیکو
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

# گام ۲: نصب مانیفست‌های سفارشی (ساخت کلاستر شبکه)
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
```
   * نکته: اگر در kubeadm init رنج پاد را چیزی غیر از 192.168.0.0/16 گذاشتید (مثلاً همان 10.244.0.0/16)، قبل از اجرای دستور دوم، فایل custom-resources.yaml را دانلود کرده و خط cidr: 192.168.0.0/16 را متناسب با رنج خودتان ویرایش کنید.

#### 🔹 گزینه سوم: Cilium (مدرن‌ترین گزینه - مبتنی بر eBPF و پرفورمنس بالا)
چرا Cilium؟

سیلیوم مدرن‌ترین CNI حال حاضر است. به جای استفاده از iptables لینوکس (که در ترافیک بالا کند می‌شود)، کدهای C کوچکی را مستقیماً داخل هسته لینوکس (eBPF) کامپایل و اجرا می‌کند. سرعت سرسام‌آور، مانیتورینگ فوق‌العاده با ابزار Hubble، و فایروال سطح ۷ (مثلاً مسدود کردن متد DELETE در مسیر /api/users) از ویژگی‌های آن است.
* پیش‌نیاز:
کرنل لینوکس نسخه 4.9 به بالا (در Ubuntu 22.04 و 24.04 پیش‌فرض اوکی است).
* دستورات نصب Cilium:
بهترین و تمیزترین روش، استفاده از ابزار خط فرمان اختصاصی خودِ سیلیوم (Cilium CLI) است:
```bash
# گام ۱: دانلود و نصب باینری Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz

# گام ۲: نصب خودکار سیلیوم روی کلاستر
cilium install

# گام ۳: بررسی وضعیت سلامت شبکه (Status Check)
cilium status
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
</div>



---








# Initialize Kubernetes
مرحله Initialize کردن (مقداردهی اولیه کلاستر) در واقع همان نقطه صفر تولد کلاستر است؛ جایی که سرور عادی لینوکسی شما رسماً تبدیل به مغز متفکر (Master / Control Plane) می‌شود.این کار با دستور kubeadm init انجام می‌شود

## 1. دستور استاندارد برای Initialize کردن:
روی سرور Master این دستور را با دسترسی root یا sudo اجرا می‌کنیم:
```bash
sudo kubeadm init --apiserver-advertise-address=<IP_MASTER_NODE> --pod-network-cidr=10.10.0.0/16
```
* این فلگ‌ها (سوییچ‌ها) دقیقاً چه می‌کنند؟
  - --apiserver-advertise-address:
    - به کوبرنتیز می‌گوید: «آی‌پی سرور مستر در شبکه محلی این است». سایر نودها (Workerها) و خود kubectl باید با این آی‌پی صحبت کنند. اگر سرور شما چند کارت شبکه دارد، نوشتن این سوییچ حیاتی است.
  - --pod-network-cidr:
    - رنج IP پادها را مشخص می‌کند. شبکه داخلی پادها کاملاً مجزا از شبکه فیزیکی سرورهاست.

## 2. در پشت صحنه kubeadm init چه اتفاقاتی می‌افتد؟
وقتی اینتر را می‌زنید، kubeadm مراحل زیر را به ترتیب طی می‌کند:
1. Preflight Checks: چک می‌کند رم حداقل ۲ گیگ باشد، ۲ هسته CPU باشد، پورت ۶۴۴۳ باز باشد، Swap حتماً خاموش باشد و containerd بالا باشد.
2. تولید گواهی‌نامه‌ها (Certificates): تمام کلیدهای رمزنگاری TLS برای ارتباط امن اجزا در مسیر /etc/kubernetes/pki ساخته می‌شود.
3. تولید Kubeconfig: فایل‌های پیکربندی دسترسی مثل admin.conf در /etc/kubernetes/ ساخته می‌شوند.
4. اجرای اجزای Control Plane به عنوان Static Pod: کانتینرهای kube-apiserver، kube-controller-manager، kube-scheduler و etcd بالا می‌آیند (مانیفست آن‌ها در /etc/kubernetes/manifests/ ریخته می‌شود).
5. ساخت Bootstrap Token: یک توکن ساخته می‌شود تا Workerها بتوانند با آن احراز هویت کنند و به کلاستر ملحق شوند.

## 3. خروجی موفقیت‌آمیز و گام بعدی اجباری
وقتی کار تمام شود، پیامی مثل این می‌گیرید:
```bash
Your Kubernetes control-plane has initialized successfully!
```
در انتهای این پیام، کوبرنتیز دو کار اجباری به شما دیکته می‌کند:
* دادن دسترسی مدیریت به یوزر عادی (بدون root کار کردن):
این ۳ خط را کپی و روی همان مستر می‌زنید:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
با این کار، فایل کانفیگ ادمین داخل روت یوزر کپی می‌شود و از حالا دستور kubectl get nodes کار خواهد کرد.

* کپی کردن دستور Join:
در آخرین خط خروجی، یک دستور Join چاپ می‌شود شبیه این:
```bash
kubeadm join 192.168.1.5:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:1234567890abcdef...
```
این دستور را ذخیره می‌کنید تا مستقیماً روی سرورهای Worker اجرا کنید.

* ⚠️ یک نکته بسیار مهم بعد از Init:
اگر بلافاصله بعد از این مراحل دستور زیر را بزنید:

```bash
kubectl get nodes
```
وضعیت مستر NotReady است! تعجب نکنید؛ این کاملاً طبیعی است چون تا زمانی که CNI (پلاگین شبکه مثل Calico) را نصب نکنید، DNS کلاستر (CoreDNS) معلق می‌ماند و نود آماده به کار نمی‌شود. به محض اعمال فاز CNI، وضعیت مستر بلافاصله Ready می‌شود.

</div>

---