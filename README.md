# Dynatrace OneAgent — Deploy via Ansible

Project Ansible untuk menginstal & mengelola **Dynatrace OneAgent** di banyak host
(Linux / Windows / AIX) menggunakan koleksi resmi
[`dynatrace.oneagent`](https://github.com/Dynatrace/Dynatrace-OneAgent-Ansible).

---

## Struktur project

```
ansible-integration/
├── ansible.cfg                         # konfigurasi Ansible (inventory, collections_path, ssh)
├── requirements.yml                    # koleksi Ansible (dynatrace.oneagent, ansible.windows)
├── requirements-python.txt             # dependency Python control node (ansible-core, pywinrm)
├── inventory/
│   ├── hosts.yml                       # daftar host, dikelompokkan per-OS (linux / windows)
│   └── group_vars/
│       ├── all/
│       │   ├── vars.yml                # konfigurasi OneAgent (versi, install args, host group)
│       │   └── vault.yml.example       # TEMPLATE rahasia → salin ke vault.yml lalu di-encrypt
│       ├── linux.yml                   # koneksi SSH + sudo untuk host Linux
│       └── windows.yml                 # koneksi WinRM untuk host Windows
├── playbooks/
│   ├── deploy-oneagent.yml             # instal OneAgent
│   └── uninstall-oneagent.yml          # uninstall OneAgent
├── .gitignore
└── README.md
```

---

## Prasyarat

| Kebutuhan | Keterangan |
|---|---|
| **Control node** | Linux / **WSL** / macOS. **Bukan Windows native** — control node Ansible tidak didukung di Windows. Di mesin ini WSL sudah tersedia, jadi jalankan semuanya di dalam WSL. |
| **Ansible** | `ansible-core >= 2.15` (belum terpasang di mesin ini). |
| **PaaS token** | Token khusus download installer (**bukan** API token). Lihat langkah 2. |
| **Konektivitas** | Host target harus bisa menjangkau URL cluster Dynatrace / ActiveGate untuk unduh installer. Kalau air-gapped, pakai installer lokal (lihat bagian Air-gapped). |
| **pywinrm** | Hanya untuk target Windows. |

---

## Langkah setup

### 0. Siapkan control node di WSL

Karena mesin ini Windows, jalankan Ansible di WSL. Buka terminal WSL, lalu **clone/copy folder ini ke filesystem Linux** (mis. `~/ansible-integration`) agar permission file bekerja normal:

```bash
# di dalam WSL
sudo apt update && sudo apt install -y python3 python3-pip
cp -r /mnt/c/Users/malik/OneDrive/Documents/works/Dynatrace/ansible-integration ~/ansible-integration
cd ~/ansible-integration
```

### 1. Install dependencies

```bash
python3 -m pip install -r requirements-python.txt
ansible-galaxy collection install -r requirements.yml -p ./collections
ansible --version        # verifikasi
```

### 2. Buat PaaS token di Dynatrace

> ⚠️ **Gunakan PaaS token, BUKAN API token.** Token yang dipakai MCP (`dt0c01...` di log)
> adalah API token dan tidak cocok untuk mengunduh installer.

1. Di Dynatrace: **Deploy Dynatrace → Set up PaaS integration**.
2. **Create token** (scope: *InstallerDownload*), salin nilainya (`dt0c01.XXXX.YYYY`).

### 3. Konfigurasi rahasia (Ansible Vault)

```bash
cp inventory/group_vars/all/vault.yml.example inventory/group_vars/all/vault.yml
# Isi environment URL + PaaS token, lalu encrypt:
ansible-vault encrypt inventory/group_vars/all/vault.yml
```

Isi `vault.yml`:
- `vault_oneagent_environment_url` — untuk **Managed**: `https://<cluster-host>:9999/e/<environment-id>`
- `vault_oneagent_paas_token` — PaaS token dari langkah 2

Edit lagi kapan pun dengan: `ansible-vault edit inventory/group_vars/all/vault.yml`.
File `vault.yml` sudah masuk `.gitignore` — jangan pernah commit versi terdekripsi.

### 4. Konfigurasi inventory & koneksi

- **`inventory/hosts.yml`** — isi hostname/IP host target di grup `linux` dan/atau `windows`.
- **`inventory/group_vars/linux.yml`** — set `ansible_user` (user SSH ber-sudo) & key/credential.
- **`inventory/group_vars/windows.yml`** — set `ansible_user` & aktifkan `vault_windows_password` di vault (khusus Windows).

### 5. Sesuaikan parameter OneAgent

Edit `inventory/group_vars/all/vars.yml`:
- **`--set-host-group=...`** — ganti `DT_ANSIBLE_DEMO` dengan host group Anda.
- `--set-infra-only` — `false` = full-stack, `true` = infrastructure-only.
- Opsional: `--set-host-property=...`, `--set-network-zone=...`.
- `oneagent_version` — `latest` atau versi terpin.

Referensi parameter: <https://docs.dynatrace.com/docs/shortlink/oneagent-installation-parameters>

---

## Menjalankan

```bash
# 1) Uji konektivitas ke target
ansible all -m ping --ask-vault-pass

# 2) Dry-run (lihat perubahan tanpa menerapkan)
ansible-playbook playbooks/deploy-oneagent.yml --ask-vault-pass --check

# 3) Deploy sungguhan
ansible-playbook playbooks/deploy-oneagent.yml --ask-vault-pass

# Batasi ke sebagian host saja:
ansible-playbook playbooks/deploy-oneagent.yml --ask-vault-pass --limit linux
```

### Verifikasi

- Di Dynatrace: **Hosts** → host baru muncul dengan host group yang di-set.
- Di host Linux: `sudo systemctl status oneagent` dan `/opt/dynatrace/oneagent/agent/tools/oneagentctl --get-host-group`.

---

## Uninstall

```bash
ansible-playbook playbooks/uninstall-oneagent.yml --ask-vault-pass
```

Playbook ini menjalankan role dengan `oneagent_package_state: absent`.

---

## Catatan: Dynatrace Managed

- Format URL environment: `https://<cluster-host>:9999/e/<environment-id>`.
- Host target harus bisa menjangkau cluster/ActiveGate di port tersebut untuk mengunduh installer.
- Kalau pakai ActiveGate sebagai sumber installer, arahkan `oneagent_environment_url` ke ActiveGate.

## Catatan: Windows

- `pip install pywinrm` di control node (sudah di `requirements-python.txt`).
- Aktifkan WinRM di host Windows (mis. skrip `ConfigureRemotingForAnsible.ps1`).
- Simpan password di vault sebagai `vault_windows_password`, lalu buka komentar barisnya di `windows.yml`.

## Air-gapped / installer lokal

Kalau host target tidak punya akses ke cluster:

1. Unduh installer OneAgent manual dari Dynatrace, simpan di `files/` (mis. `files/Dynatrace-OneAgent-Linux.sh`).
2. Di `vars.yml`, aktifkan: `oneagent_local_installer: files/Dynatrace-OneAgent-Linux.sh`.

Role akan menyalin installer dari control node ke tiap target, tanpa unduh dari internet.

---

## Troubleshooting

| Gejala | Penyebab umum |
|---|---|
| `403 Forbidden` saat unduh installer | Token salah tipe (harus **PaaS**, bukan API) atau scope kurang. |
| `Connection timed out` ke cluster | Firewall / host target tidak bisa menjangkau `:9999`. Pakai ActiveGate atau installer lokal. |
| `ssh: permission denied` | `ansible_user` / SSH key salah di `group_vars/linux.yml`. |
| `become` gagal | User SSH tidak punya sudo, atau butuh password sudo (`--ask-become-pass`). |
| Windows: `winrm` error | WinRM belum aktif / `pywinrm` belum terpasang / transport salah. |

---

## Keamanan

- **Jangan commit** `vault.yml` (sudah di-`.gitignore`), `.vault_pass`, atau installer di `files/`.
- Selalu enkripsi rahasia dengan `ansible-vault`.
- ⚠️ **Token yang bocor:** file `dynatrace-managed-mcp.log` di folder ini memuat API token
  plaintext. Revoke/rotate token itu di Dynatrace dan hapus lognya.
