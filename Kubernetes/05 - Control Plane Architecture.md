# Control Plane Architecture
در این بخش اجزای اصلی مغز متفکر کلاستر (Control Plane) را بررسی می‌کنیم.
---
### مشاهده پادهای سیستمی
- برای مشاهده پادهای سیستمی از دستور زیر استفاده می‌کنیم
```bash
kubectl get pod -n kube-system
```
در خروجی دستور بالا پادهای سیستمی رو میبینید که مهمترین آن‌ها موارد زیر ‌می‌باشند و در صورتی که هرکدام از موارد زیر بالا نباشه کل کلاستر به مشکل بر میخورد:
```bash
karimi@Public-Agent12:~$ kubectl get pod -n kube-system
NAME                                     READY   STATUS    RESTARTS     AGE
etcd-public-agent12                      1/1     Running   0            13d
kube-apiserver-public-agent12            1/1     Running   0            13d
kube-controller-manager-public-agent12   1/1     Running   3 (6d ago)   13d
kube-scheduler-public-agent12            1/1     Running   3 (6d ago)   13d
```
---
## 1. etcd
- **تعریف:** یک پایگاه داده Key-Value توزیع شده و فوق‌سریع.
- **وظیفه:** ذخیره وضعیت (State) کل کلاستر. اگر etcd نباشد، کوبرنتیز هیچ چیزی از کلاستر نمی‌داند.
- **نکته کلیدی:** تنها جزء کلاستر است که وضعیت را در دیسک ذخیره می‌کند (Stateful). بقیه اجزا Stateless هستند.
---
## 2. API Server
- **وظیفه:** تنها قطعه‌ای است که با etcd صحبت می‌کند. تمام دستورات kubectl به اینجا می‌آید.
- **نقش:** احراز هویت (Auth) و اعتبارسنجی (Validation) درخواست‌ها.
---
## 3. Controller Manager
- **وظیفه:** بررسی حلقه کنترلی (Control Loop). 
- **مثال:** اگر در etcd نوشته شده ۳ تا پاد باید باشد، اما الان ۲ تا داریم، Controller Manager می‌فهمد که باید یکی دیگر بسازد و به Scheduler دستور می‌دهد.
---
## 4. Scheduler
- **وظیفه:** تصمیم می‌گیرد هر پاد روی کدام Worker Node قرار بگیرد (با توجه به منابع مثل CPU و RAM).
---
## فایل‌های سیستمی به صورت پیشفرش در دایرکتوری زیر ذخیره می‌شوند:


/etc/kubernetes/manifests/<br>
├── etcd.yaml<br>
├── kube-apiserver.yaml<br>
├── kube-controller-manager.yaml<br>
└── kube-scheduler.yaml<br>
###
