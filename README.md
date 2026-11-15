# Memuat Ulang 3 VM VirtualBox Setelah Install Pop!_OS

Dibuat: 15 November 2026

**Lokasi backup:** `/media/alfian/New Volume/os/backupVM/` (di Pop!_OS bisa dimount ke `/mnt/winbackup/os/backupVM/`).

**Isi backup sudah terverifikasi:** 3 folder VM + `VirtualBox.xml` yang identik dengan config aktif, berisi 3 VM dan NAT Network `coba` — tidak ada entri `popos`.

---

## 1. Install VirtualBox 7.2

VM dibuat dengan VirtualBox **7.2.20**, jadi jangan pakai versi lebih lama.

```bash
sudo apt update
sudo apt install -y curl gnupg

curl -fsSL https://www.virtualbox.org/download/oracle_vbox_2016.asc \
  | sudo gpg --dearmor -o /usr/share/keyrings/oracle-virtualbox-2016.gpg

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/oracle-virtualbox-2016.gpg] https://download.virtualbox.org/virtualbox/debian noble contrib" \
  | sudo tee /etc/apt/sources.list.d/virtualbox.list >/dev/null

sudo apt update
sudo apt install -y virtualbox-7.2
sudo usermod -aG vboxusers "$USER"
```

- `noble` sesuai Pop!_OS 24.04. Jika memakai rilis Pop lain, sesuaikan codename-nya.
- **Logout lalu login lagi** agar grup `vboxusers` aktif.
- Jika **Secure Boot** aktif, ikuti prompt MOK saat instalasi DKMS, lalu enroll key saat reboot. Alternatif: matikan Secure Boot.
- Verifikasi:

```bash
vboxmanage --version
lsmod | grep vbox
```

---

## 2. Mount partisi backup

Gunakan mode **read-only** saat restore.

```bash
sudo mkdir -p /mnt/winbackup
sudo mount -t ntfs-3g -o ro /dev/nvme0n1p5 /mnt/winbackup
```

Jika partisi sudah otomatis termount di `/media/alfian/New Volume`, gunakan path itu saja:

```bash
BACKUP="/media/alfian/New Volume/os/backupVM"
```

Atau jika memakai mount manual:

```bash
BACKUP="/mnt/winbackup/os/backupVM"
ls "$BACKUP"
```

---

## 3. Salin VM kembali ke ext4

**Jangan jalankan VM langsung dari NTFS.** Salin dulu ke home.

```bash
mkdir -p "$HOME/VirtualBox VMs"

rsync -a --info=progress2 \
  "$BACKUP/kalii" \
  "$BACKUP/ubuntu server" \
  "$BACKUP/ubuntu serverr" \
  "$HOME/VirtualBox VMs/"
```

Perbaiki ownership dan permission:

```bash
sudo chown -R "$USER:$USER" "$HOME/VirtualBox VMs"
chmod -R u+rwX,go-rwx "$HOME/VirtualBox VMs"
```

Cek ukuran ketiga disk:

```bash
stat -c '%s %n' "$HOME/VirtualBox VMs"/*/*.vdi
```

Ukuran yang benar:

| VM | Ukuran `.vdi` |
|---|---:|
| `kalii` | 24.945.623.040 |
| `ubuntu server` | 7.299.137.536 |
| `ubuntu serverr` | 6.280.970.240 |

---

## 4. Kembalikan `VirtualBox.xml`

Pastikan VirtualBox **tidak sedang berjalan**:

```bash
pgrep -a VBoxSVC || true
```

Jika ada proses, tutup GUI VirtualBox dan tunggu sampai proses `VBoxSVC` berhenti.

Backup config baru (kalau sudah sempat dibuat), lalu salin config lama:

```bash
[ -f "$HOME/.config/VirtualBox/VirtualBox.xml" ] && \
  cp -a "$HOME/.config/VirtualBox/VirtualBox.xml" \
        "$HOME/.config/VirtualBox/VirtualBox.xml.fresh"

install -Dm600 "$BACKUP/VirtualBox.xml" \
  "$HOME/.config/VirtualBox/VirtualBox.xml"
```

Config ini sudah berisi:

- 3 VM: `kalii`, `ubuntu server`, `ubuntu serverr`
- NAT Network `NatNetwork` dan `coba`
- DHCP/DNS `coba`: `10.0.2.0/24`, gateway `10.0.2.1`, DNS `10.0.2.3`
- Domain `tail76667c.ts.net`

> Langkah ini hanya valid jika username dan home tetap `/home/alfian`. Kalau berbeda, gunakan **Opsi B** di bawah.

---

## 5. Verifikasi

```bash
vboxmanage list vms
vboxmanage list natnetworks
vboxmanage list dhcpservers
```

Yang diharapkan:

- 3 VM terdaftar
- NAT Network: `NatNetwork` dan `coba`
- DHCP server: `coba` dan `HostInterfaceNetworking-vboxnet0`

Cek konfigurasi network VM:

```bash
vboxmanage showvminfo "kalii" --machinereadable \
  | grep -E "VMState|nic1|nat-network1"
```

Harus muncul:

```text
VMState="poweroff"
nic1="natnetwork"
nat-network1="coba"
```

---

## 6. Jalankan VM

Lewat GUI VirtualBox, atau lewat terminal:

```bash
vboxmanage startvm "kalii"
vboxmanage startvm "ubuntu server"
vboxmanage startvm "ubuntu serverr"
```

Di dalam guest, cek koneksi:

```bash
ip addr
ping -c 3 10.0.2.1
```

---

## Opsi B — tanpa restore `VirtualBox.xml`

Gunakan ini jika username/home berbeda atau config XML bermasalah.

```bash
VBoxManage registervm "$HOME/VirtualBox VMs/kalii/kalii.vbox"
VBoxManage registervm "$HOME/VirtualBox VMs/ubuntu server/ubuntu server.vbox"
VBoxManage registervm "$HOME/VirtualBox VMs/ubuntu serverr/ubuntu serverr.vbox"

VBoxManage natnetwork add \
  --netname coba \
  --network 10.0.2.0/24 \
  --enable \
  --dhcp on

VBoxManage dhcpserver modify --network=coba \
  --set-opt=3 10.0.2.1 \
  --set-opt=6 10.0.2.3 \
  --set-opt=15 tail76667c.ts.net \
  --set-opt=119 tail76667c.ts.net
```

Jangan campur Opsi B dengan restore `VirtualBox.xml`, karena bisa membuat entri/VM ganda.

---

## Troubleshooting

### VM muncul sebagai *inaccessible*

Periksa apakah path di `.vbox` dan `VirtualBox.xml` cocok:

```bash
grep -n "location=" "$HOME/VirtualBox VMs/kalii/kalii.vbox"
vboxmanage showvminfo "kalii"
```

Jika home/username berbeda, gunakan Opsi B.

### Error "machine was created by a newer version"

Install VirtualBox **7.2.20 atau lebih baru**.

### Modul kernel tidak termuat

```bash
sudo apt install linux-headers-"$(uname -r)"
sudo modprobe vboxdrv
```

Jika Secure Boot aktif, enroll MOK atau matikan Secure Boot.

### UUID/VM duplikat

```bash
vboxmanage list vms
VBoxManage unregistervm <uuid>
```

Gunakan `unregistervm` **tanpa** `--delete` agar file VM tidak terhapus.

### NAT Network `coba` tidak ditemukan

Restore `VirtualBox.xml`, atau buat ulang dengan perintah pada Opsi B.

### Peringatan ISO hilang

VM Kali dan Ubuntu Server masih merujuk ISO di `/home/alfian/os/`. VM tetap bisa boot dari disk; attach ulang ISO jika diperlukan.

---

## Catatan

- Setelah selesai, unmount backup:

  ```bash
  sudo umount /mnt/winbackup
  ```

- Save game SpartanPort, screenshot, dan sesi opencode **belum terlihat** di folder `backupVM`; salin terpisah sesuai daftar path sebelumnya.
- Data game `~/game/CUSA01623` (48 GB) tidak termasuk backup VM.
