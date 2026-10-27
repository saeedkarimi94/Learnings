# دستورات سرور داکری:

## مشاهده آی پی‌ها و میزان ریسورس سرورهای کلاستر داکر:
```bash
for node in $(docker node ls --format '{{.Hostname}}'); do
  docker node inspect "$node" --format '{{.Description.Hostname}}|{{.Status.Addr}}|{{.Description.Resources.NanoCPUs}}|{{.Description.Resources.MemoryBytes}}'
done | awk -F'|' '{
  printf "%-18s %-16s CPU: %-4.0f  RAM: %.0f GB\n", $1, $2, $3/1e9, $4/1024/1024/1024
}'
```