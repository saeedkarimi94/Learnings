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
