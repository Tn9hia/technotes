## Network benchmark with iperf3
### Install
```
sudo apt update 
sudo apt install iperf3 -y
```
### Scenario
#### Server
Default port 5201
```
iperf3 -s
```
#### Client
```
iperf3 -c <IP_server>
```
#### Advance option
Benchmark UDP
```
iperf3 -c <IP_server> -u -b 10G
```
Benchmark multithreading
```
iperf3 -c <IP_server> -P <số luồng>
```
Benchmark in time range
```
iperf3 -c <IP_server> -t <thời gian>
```
Benchmark Round-Trip Time
```
iperf3 -c <IP_server> -R
```

### Result
- **Bandwidth (băng thông):** Tốc độ truyền tải tối đa đạt được.
- **Jitter:** Độ biến động trong thời gian gửi gói (chỉ áp dụng với UDP).
- **Packet Loss:** Tỷ lệ mất gói (chỉ áp dụng với UDP).
## Disk benchmark with fio
### Install
```
sudo apt update
sudo apt install fio -y
```
### Scenario
```
fio --name=bench_test --rw=write --size=1G --bs=4k --numjobs=1 --direct=1
```
fio --name=bench_test --rw=write --size=1G --bs=4k --numjobs=1 --direct=1`

**Ý nghĩa:**
- `--name=bench_test`: Tên bài test.
- `--rw=write`: Benchmark ghi tuần tự.
- `--size=1G`: Tạo file dung lượng 1GB để test.
- `--bs=4k`: Kích thước block (4 KB).
- `--numjobs=1`: Số luồng (threads) thực hiện.
- `--direct=1`: Ghi trực tiếp, bỏ qua cache hệ thống.
#### Read sequence
```
fio --name=seq_read --rw=read --size=2G --bs=128k --direct=1
```
#### Write sequence
```
fio --name=seq_write --rw=write --size=2G --bs=128k --direct=1
```
#### Read random
```
fio --name=rand_read --rw=randread --size=2G --bs=4k --iodepth=64 --direct=1
```
#### Write random
```
fio --name=rand_write --rw=randwrite --size=2G --bs=4k --iodepth=64 --direct=1
```
#### Mix workload
```
fio --name=mixed_rw --rw=randrw --rwmixread=70 --size=2G --bs=4k --iodepth=64 --direct=1
```


write sq iops cao nhất