# ansible-join-ad

Playbook Ansible dùng để join các máy Ubuntu 22.04 vào Active Directory domain `vnsp.local`,
chuyển thể từ script `scriptJoinAD.sh` + playbook `joinDomain.yml` gốc (chạy qua AWX) sang
dạng role Ansible chuẩn, idempotent, có thể đưa lên GitHub.

## Cấu trúc

```
ansible-join-ad/
├── ansible.cfg
├── inventory/hosts.ini          # Danh sách host
├── group_vars/all/vars.yml      # Biến không nhạy cảm (domain, DNS, group...)
├── group_vars/all/vault.yml.example  # Mẫu biến nhạy cảm (mật khẩu join AD)
├── playbooks/join_domain.yml    # Playbook chính
└── roles/join_ad/               # Role thực hiện join AD
    ├── defaults/main.yml
    ├── tasks/main.yml
    ├── handlers/main.yml
    ├── templates/sssd.conf.j2
    ├── templates/resolv_head.j2
    └── meta/main.yml
```

## Chuẩn bị

1. Cài Ansible >= 2.14 trên control node.
2. Cập nhật `inventory/hosts.ini` với danh sách server thật.
3. Cập nhật `group_vars/all/vars.yml` (domain, IP DNS của các AD server, group được phép login...).
4. Tạo file vault chứa mật khẩu tài khoản dùng để join domain (KHÔNG commit plaintext):

   ```bash
   cp group_vars/all/vault.yml.example group_vars/all/vault.yml
   # sửa join_password trong file, sau đó mã hoá:
   ansible-vault encrypt group_vars/all/vault.yml
   ```

5. Đảm bảo user remote (`ansible_user` trong inventory hoặc `remote_user`) có quyền sudo
   NOPASSWD hoặc bạn dùng `--ask-become-pass`.

## Chạy playbook

```bash
ansible-playbook -i inventory/hosts.ini playbooks/join_domain.yml --ask-vault-pass
```

Chạy cho 1 host cụ thể:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/join_domain.yml --ask-vault-pass --limit server1.vnsp.local
```

## Những thay đổi so với bản gốc (và lý do)

- **Không copy script `.sh` sang remote rồi exec** – toàn bộ logic được viết lại bằng
  module Ansible (`hostname`, `apt`, `template`, `systemd`, `lineinfile`...) để playbook
  idempotent (chạy lại nhiều lần không lỗi, không tạo trùng dòng trong file cấu hình).
- **Mật khẩu join AD đưa vào Ansible Vault** thay vì hardcode dạng plaintext trong script
  (`_passwordTOJoin="XXXXXXXX"` trong bản gốc là rủi ro bảo mật nghiêm trọng nếu đẩy lên
  GitHub công khai).
- **Sửa lỗi `sudo cat <<EOF > file`**: trong bash, redirect (`>`) được shell cha xử lý
  *trước* khi `sudo` chạy `cat`, nên nếu user hiện tại không có quyền ghi vào
  `/etc/sssd/sssd.conf` hay `/etc/sudoers`, lệnh sẽ fail âm thầm dù có `sudo`. Ansible dùng
  module `template`/`lineinfile` chạy đúng quyền `become: true` nên không gặp vấn đề này.
- **Sửa lỗi cú pháp sudoers**: bản gốc có dòng `NOPASSWD:ALLs` (thừa chữ `s`) – dòng này
  sẽ làm `visudo`/sudoers parser lỗi và có thể khoá luôn quyền sudo trên server. Ansible
  dùng `validate: 'visudo -cf %s'` để kiểm tra cú pháp trước khi ghi.
- **Idempotent realm join**: kiểm tra `realm list` trước, chỉ join nếu domain chưa có
  trong danh sách, tránh lỗi "already joined" khi chạy lại playbook.
- **Không hardcode tài khoản domain cụ thể** (`rocketor.vn@vnsp.local`) trong role — đưa
  ra biến `sudo_admin_users` (list) trong `group_vars/all/vars.yml` để dễ tái sử dụng cho
  môi trường khác.

## Lưu ý bảo mật

- Không bao giờ commit `group_vars/all/vault.yml` ở dạng chưa mã hoá — `.gitignore` đã
  loại trừ sẵn tên file gốc, chỉ file `.example` được đưa lên repo.
- Cân nhắc dùng `ansible-vault` với password file quản lý qua secret manager (Vault,
  AWX Credential, GitHub Actions secret...) thay vì gõ tay `--ask-vault-pass` khi chạy CI/CD.
