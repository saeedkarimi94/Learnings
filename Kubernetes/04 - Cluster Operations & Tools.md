# برخی دستورات کاربردی کوبرنتیز
### ۱. اتوکامپلیت kubectl:
#### برای اینکه دستورتان با زدن کلید Tab تکمیل شود (خیلی توی کار با ترمینال کمک می‌کند):
```bash
# فعال‌سازی موقت روی شل جاری
source <(kubectl completion bash)

# فعال‌سازی دائمی برای همیشه روی کاربر جاری
echo 'source <(kubectl completion bash) >> ~/.bashrc'
source ~/.bashrc
```
### ۲. دستورات پرکاربرد بررسی کلاستر (Health & Status)
#### این‌ها ابزارهای روزمره یک DevOps Engineer برای دیدن وضعیت کلاستر هستند:
```bash
kubectl version        # بررسی نسخه ابزارها
kubeadm version        # بررسی نسخه kubeadm
kubectl cluster-info   # بررسی آدرس API Server و CoreDNS

kubectl get node       # لیست نودها
kubectl get ns         # لیست Namespaceها
kubectl get pod        # لیست پادها در namespace پیش‌فرض
kubectl get pod -o wide # نمایش پادها به همراه IP و نود میزبان (خیلی مهم!)
kubectl get pod -n kube-system # دیدن پادهای حیاتی سیستم
```
### ۳. مدیریت برچسب‌ها (Labels) روی نودها
#### برای دسته‌بندی نودها (مثلاً مشخص کردن نقش یا ویژگی خاص برای یک Worker):
```bash
kubectl label node worker2 kubernetes.io/role=worker-2-B
```
### ۴. مدیریت پیشرفته توکن‌ها (Token Management)
#### دیدن توکن‌های فعال یا ساخت توکن با تاریخ انقضای سفارشی (مثلاً ۲ ساعت):
```bash
kubeadm token list
kubeadm token create --print-join-command --ttl 2h
```
### ۵. مسیر درایورهای شبکه (CNI Binaries)
#### محل قرارگیری باینری‌های درایورهای شبکه روی نودها:
```bash
cat /opt/cni/bin => network driver
```
### ۶. ابزار nerdctl (جایگزین Docker CLI برای containerd)
#### چون روی نودها از containerd استفاده می‌کنید، ابزار nerdctl دقیقاً شبیه دستورات docker (مثل nerdctl ps یا nerdctl run) کار می‌کند اما مستقیم با containerd ارتباط می‌گیرد بدون اینکه نیازی به نصب داکر باشد:
* مخزن گیت‌هاب: https://github.com/containerd/nerdctl
* مرجع دستورات: https://github.com/containerd/nerdctl/blob/main/docs/command-reference.md
#### نصب سریع:
```bash
  # دانلود و نصب پکیج کامل nerdctl-full
  wget https://github.com/containerd/nerdctl/releases/download/v1.7.6/nerdctl-full-1.7.6-linux-amd64.tar.gz
  sudo tar Cxzvf /usr/local nerdctl-full-1.7.6-linux-amd64.tar.gz
```
---
