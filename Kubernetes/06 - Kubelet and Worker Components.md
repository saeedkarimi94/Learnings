## 1. بررسی عمیق kubelet (فرمانده محلی / نماینده تمام‌وقت Master روی نود)

اگر Control Plane یا همان `kube-apiserver` را **مغز مدیر مرکزی (HQ)** در نظر بگیریم، `kubelet` قطعاً **فرمانده میدانی (Field Commander) روی هر نود کارگر** است. Master تصمیم می‌گیرد، دستور می‌دهد و برنامه می‌ریزد؛ اما کسی که در عمل بیل می‌زند و مطمئن می‌شود کانتینرها واقعاً بالا آمده‌اند، `kubelet` است.

---

### 1.1 یک تشبیه ملموس برای شروع

**تشبیه: سرپرست عملیات یک کارگاه تولیدی**
تصور کن در یک کارخانه، یک مدیر مرکزی (Master) داریم که سفارش‌ها را می‌پذیرد و زمان‌بندی می‌کند. اما در هر واحد تولیدی (Worker Node) یک **سرپرست شیفت (kubelet)** نشسته است که:
* فقط و فقط با مدیر مرکزی (API Server) در ارتباط است،
* سفارش تولید (مانیفست پاد) را از مدیر می‌گیرد،
* به تیم زیردستش (Container Runtime مثل containerd) دستور اجرا می‌دهد،
* مدام گزارش وضعیت (سلامت کانتینرها) را به مدیر مرکزی برمی‌گرداند،
* و اگر کانتینری بمیرد یا کرش کند، خودش **دست به کار نمی‌شود**؛ فقط مشکل را به مدیر مرکزی گزارش می‌دهد تا Controlling Loop تصمیم بگیرد.

---

### 1.2 (نکته معماری مهم)، kubelet یک سرویس است، نه یک پاد!

بگذارید یک تفاوت اساسی را از همین ابتدا گوشزد کنم که در معماری کوبرنتیز خیلی‌ها را گول می‌زند:
* تمام اجزای Control Plane (etcd, apiserver, controller-manager, scheduler) به صورت **Static Pod** در مسیر `/etc/kubernetes/manifests/` اجرا می‌شوند.
* اما `kubelet` این‌طور نیست! `kubelet` به‌عنوان یک **سرویس سیستمعاملی (systemd service)** مستقیماً روی سیستم‌عامل نود (نه داخل یک کانتینر) اجرا می‌شود.

> **چرا؟** بسیار ساده، چون `kubelet` خودش باید اولین چیزی باشد که بالا می‌آید تا بتواند پادها و حتی خود Static Podهای Master را مدیریت کند. نمی‌تواند سوار بر چیزی باشد که وظیفه‌اش ساختن و نظارت بر آن است!
```bash
# مشاهده وضعیت سرویس kubelet به صورت systemd service
sudo systemctl status kubelet
```
---

### 1.3 وظایف کلیدی kubelet (در یک نگاه)
* ثبت و عضویت نود در کلاستر (Node Registration):
  - در اولین اجرا، kubelet خودش را با یک نام (مثل worker1) به API Server معرفی و ثبت می‌کند تا نود به عضویت کلاستر درآید.
* دریافت مانیفست پادها:
  - از دو منبع اصلی مانیفست پاد را می‌گیرد:
    - از API Server (پادهایی که Scheduler به این نود اختصاص داده).
    - از محلی Static Pods در مسیر /etc/kubernetes/manifests/.
* فرمان اجرا به Container Runtime (CRI):
  - مانیفست را به درگاه (gRPC) Container Runtime Interface خبر می‌دهد تا runtime (مثل containerd) کانتینرها را ساخته و اجرا کند.
* گزارش وضعیت (Heartbeat / Status Reporting):
  - مدام وضعیت سلامت کانتینرها (Ready/Running/Error) را از طریق API Server به etcd گزارش می‌دهد تا Controller Manager از سلامت همه‌چیز باخبر باشد.
* انجام چک‌های سلامت (Probes):
  - Liveness، Readiness و Startup Probes را اجرا می‌کند تا مطمئن شود کانتینرها سالم و آماده‌ی سرویس دادن هستند.
* نظارت بر منابع و گزارش متریک:
  - مصرف CPU/رم هر پاد را جمع‌آوری و به متریک سرورها گزارش می‌دهد.
---

### 1.4 کالبدشکافی فایل سرویس kubelet.service
روی نود ورکر، اگر مسیر زیر را باز کنید:
```bash
sudo cat /usr/lib/systemd/system/kubelet.service
```
* متوجه می‌شوید که این سرویس با چه پارامترهایی اجرا می‌شود. مهم‌ترین خط آن ارجاع به فایل کانفیگ است:<br>
[Service]<br>
ExecStart=/usr/bin/kubelet --config=/var/lib/kubelet/config.yaml 
---
### 1.5 مانیفست و تنظیمات اصلی: /var/lib/kubelet/config.yaml
بیشتر تنظیمات kubelet در این فایل YAML قرار دارد. بیایید چند فیلد کلیدی و حیاتی آن را مرور کنیم:
``` bash
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
# آدرس IP که kubelet روی آن به درخواست‌ها گوش می‌دهد (معمولاً IP خود نود)
address: 0.0.0.0
port: 10250 # پورت اصلی ارتباطی Kubelet (برای دریافت لاگ، ارسالات exec و...)
authentication:
  anonymous:
    enabled: false # اجازه دسترسی ناشناس به kubelet را می‌بندد (امنیت)
  webhook:
    enabled: true  # احراز هویت از طریق API Server انجام شود
authorization:
  mode: Webhook    # تایید مجوزها هم به API Server واگذار می‌شود
clusterDomain: cluster.local
clusterDNS:
  - 10.96.0.10     # آی‌پی سرویس CoreDNS برای حل نام‌ها در کلاستر
staticPodPath: /etc/kubernetes/manifests # مسیر مانیفست‌های استاتیک مستر
containerRuntimeEndpoint: unix:///var/run/containerd/containerd.sock # درگاه ارتباط با CRI
```
---
### 1.6 درگاه ارتباطی: Container Runtime Interface (CRI)
kubelet خودش مستقیماً به کرنل لینوکس دستور ساخت کانتینر نمی‌دهد؛ بلکه یک واسط استاندارد به نام CRI دارد.
* نقش CRI: یک پروتکل مبتنی بر gRPC است که دو بخش اصلی دارد:
  - Image Service: وظیفه دانلود، آپلود و مدیریت ایمیج‌ها (Image pull/push).
  - Runtime Service: وظیفه مدیریت چرخه حیات کانتینر (Start, Stop, Pause, Delete).
* نقش containerd: همان موتور اجرایی (Runtime) است که پشت این درگاه نشسته و دستورات kubelet را به کانتینرهای واقعی لینوکس (namespaces, cgroups) تبدیل می‌کند.
--- 
### 1.7 ابزارهای بررسی و عیب‌یابی kubelet روی نود
برای اینکه همیشه بتوانید وضعیت نود و kubelet را زیر نظر داشته باشید، این دستورات روتین DevOpsی را دم دست داشته باشید:
```bash
# 1. بررسی وضعیت سرویس kubelet
sudo systemctl status kubelet

# 2. مشاهده لاگ‌های زنده kubelet با journalctl
sudo journalctl -u kubelet -f -o cat

# 3. بررسی اینکه آیا kubelet می‌تواند به API Server وصل شود یا خیر
kubectl get nodes
```

