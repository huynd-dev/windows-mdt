# Hướng dẫn Deploy Windows 11 tự động cho Dell Latitude 5420 (MDT + WDS)

> Mục tiêu: Máy mới boot qua mạng (PXE) → tự cài Windows 11 → tự nạp đúng driver Latitude 5420 → tự cài bộ phần mềm chuẩn của công ty → join domain. Có thể mở rộng cho nhiều model khác sau này.

---

## 0. Tổng quan luồng

```
Máy 5420 mới  →  PXE Boot (WDS)  →  WinPE  →  MDT Wizard
      →  Phân vùng UEFI/GPT  →  Cài Windows 11
      →  Inject driver theo %Model% (chỉ Latitude 5420)
      →  Cài phần mềm chuẩn (silent)  →  Join Domain  →  Hoàn tất
```

Ba thành phần cài trên **server deploy** (Windows Server 2019/2022 hoặc 1 máy Win10/11 Pro chuyên dụng):
- **Windows ADK** + **WinPE add-on** — bộ công cụ tạo môi trường boot.
- **MDT (Microsoft Deployment Toolkit)** — bộ não điều phối (task sequence, driver theo model, app).
- **WDS (Windows Deployment Services)** — server PXE để máy boot qua mạng.

---

## 1. Chuẩn bị hạ tầng

| Hạng mục | Yêu cầu |
|---|---|
| Server deploy | Windows Server 2022 (khuyến nghị) hoặc Win10/11 Pro. RAM ≥ 8GB, ổ trống ≥ 200GB |
| Mạng | DHCP hoạt động; server và máy 5420 cùng VLAN/subnet (PXE nhạy với DHCP/relay) |
| Active Directory | Domain controller để join domain (nếu cần) |
| File ISO | Windows 11 ISO (bản tương ứng license công ty: Pro / Enterprise) |

**Tải về trước:**
1. Windows ADK for Windows 11 — trang Microsoft Learn ("Download and install the Windows ADK").
2. Windows PE add-on for the ADK (cùng trang).
3. MDT — tìm "Microsoft Deployment Toolkit download" trên trang Microsoft.
4. **Driver pack Latitude 5420 cho Windows 11** (xem mục 5).

> Lưu ý phiên bản: dùng ADK đời mới đủ hỗ trợ Windows 11. Cài đúng thứ tự: **ADK → WinPE add-on → MDT**.

---

## 2. Cài đặt các thành phần

### 2.1. Cài Windows ADK + WinPE add-on
- Chạy `adksetup.exe`, chọn tối thiểu: **Deployment Tools** + **Imaging and Configuration Designer (ICD)** + **User State Migration Tool**.
- Chạy `adkwinpesetup.exe`, cài **Windows Preinstallation Environment**.

### 2.2. Cài MDT
- Chạy file MSI của MDT (bản x64).

### 2.3. Cài & cấu hình WDS (trên Windows Server)
- Server Manager → Add Roles → **Windows Deployment Services** (cả Deployment Server + Transport Server).
- Mở **WDS console** → Configure Server → chọn **Integrated with Active Directory** (hoặc Standalone nếu không có AD).
- PXE Response: chọn **Respond to all client computers** (giai đoạn đầu để test; siết lại sau).

---

## 3. Tạo Deployment Share trong MDT

Mở **Deployment Workbench**:

1. Chuột phải **Deployment Shares → New Deployment Share**.
2. Đường dẫn ví dụ: `D:\DeploymentShare`, share name `DeploymentShare$`.
3. Sau khi tạo xong, dưới share sẽ có các nhánh: **Operating Systems, Applications, Out-of-Box Drivers, Task Sequences**.

---

## 4. Import Windows 11

1. Mount ISO Windows 11 (hoặc giải nén).
2. Trong Workbench → **Operating Systems** → chuột phải → **Import Operating System**.
3. Chọn **Full set of source files** → trỏ tới ổ ISO đã mount.
4. Đặt tên thư mục dễ nhận, ví dụ `Windows 11 Pro x64`.

---

## 5. Driver cho Dell Latitude 5420 (phần quan trọng nhất)

Dell cung cấp sẵn **Dell Command | Deploy Driver Pack** dạng file driver (`.cab`/DUP) đúng để dùng với MDT/SCCM — không phải tự gom driver lẻ.

### 5.1. Tải driver pack
- Lên trang Dell, tìm: **"Latitude 5420 Windows 11 Driver Pack"** (KB Dell `000193895`), hoặc vào trang tổng **"Dell Command | Deploy Driver Packs for Latitude Models"**.
- Tải gói **Windows 11** (đúng bản 5420 thường, không phải bản *Rugged* — đó là model khác).
- Giải nén ra một thư mục, ví dụ `C:\Drivers\Latitude_5420_Win11`.

> Dell đang chuyển định dạng pack từ `.cab` sang DUP; với 5420 (đời cũ) `.cab` vẫn được hỗ trợ. Cứ giải nén ra cấu trúc thư mục driver là dùng được.

### 5.2. Tạo cấu trúc Out-of-Box Drivers theo model (chuẩn để mở rộng)
Trong **Out-of-Box Drivers**, tạo cây thư mục theo `Make\Model`:

```
Out-of-Box Drivers
└── Dell Inc.
    └── Latitude 5420
        └── (import driver pack vào đây)
```

- Chuột phải nhánh `Latitude 5420` → **Import Drivers** → trỏ tới `C:\Drivers\Latitude_5420_Win11`.

> Tên `Dell Inc.` và `Latitude 5420` phải **khớp chính xác** với giá trị WMI mà MDT đọc được từ máy (`Make` = `Dell Inc.`, `Model` = `Latitude 5420`). Đây là chìa khóa để chỉ nạp đúng driver cho đúng model.

### 5.3. (Tùy chọn) Driver cho WinPE
Nếu WinPE không thấy card mạng hoặc ổ NVMe của 5420 khi boot, tải thêm **WinPE Driver Pack** của Dell và import vào một nhánh riêng (ví dụ `Out-of-Box Drivers\WinPE x64`), rồi gán nhánh đó cho boot image (mục 9). Với 5420 + ADK mới thường không cần, nhưng cứ chuẩn bị sẵn.

---

## 6. Đóng gói & import phần mềm công ty (Applications)

Nguyên tắc: mỗi phần mềm phải cài được **im lặng** (silent, không bấm chuột).

Ví dụ tham số silent phổ biến:
- MSI: `msiexec /i "app.msi" /quiet /norestart`
- Nhiều app EXE: `setup.exe /S` hoặc `/silent /norestart` (tùy nhà phát hành).

Cách import:
1. **Applications** → chuột phải → **New Application** → *Application with source files*.
2. Trỏ tới thư mục cài đặt; ở ô **Command line** nhập lệnh silent ở trên.
3. Lặp lại cho từng phần mềm (ví dụ: Office, trình duyệt chuẩn, antivirus, VPN, app nội bộ...).

> Mẹo: với bộ phần mềm dùng chung cho **mọi máy**, có thể gộp gọi qua 1 ứng dụng "bundle" chạy script; còn phần mềm **riêng theo model/phòng ban** thì để tách app và gán điều kiện (mục 9.3).

---

## 7. Tạo Task Sequence

1. **Task Sequences** → chuột phải → **New Task Sequence**.
2. Template: chọn **Standard Client Task Sequence**.
3. Đặt ID + tên, ví dụ `W11-5420` / `Deploy Win11 - Latitude 5420`.
4. Chọn OS đã import ở mục 4.
5. Product key: chọn theo license (KMS/MAK hoặc để trống nếu dùng AD-based activation).
6. Hoàn tất.

Task sequence mặc định đã có sẵn bước **Inject Drivers** và các bước cài Windows — ta chỉ tinh chỉnh ở mục 9.

---

## 8. Thêm bước cài phần mềm vào Task Sequence

Mở task sequence `W11-5420` → tab **Task Sequence**:

- Tới nhóm **State Restore** (sau bước Windows đã cài và join domain).
- **Add → General → Install Application**.
- Chọn *Install a single application* → trỏ tới app đã tạo ở mục 6. Lặp lại cho từng phần mềm, hoặc thêm 1 bước "Install Applications" để cài theo nhóm.

---

## 9. Cấu hình "theo model" — trái tim của setup

### 9.1. Selection Profile cho driver
Mục **Advanced Configuration → Selection Profiles** → New:
- Tên: `Drivers - Latitude 5420`.
- Tick đúng nhánh `Out-of-Box Drivers\Dell Inc.\Latitude 5420`.

### 9.2. Cho task sequence chỉ nạp đúng driver theo model
Trong task sequence → bước **Preinstall → Inject Drivers**:
- Cách đơn giản & bền vững: set **logic theo biến `%Model%`**. Trong `CustomSettings.ini` (mục 9.4), khai báo:

```ini
[Settings]
Priority=Model, Default

[Latitude 5420]
DriverGroup001=Dell Inc.\Latitude 5420
DriverSelectionProfile=Nothing

[Default]
OSInstall=YES
SkipBDDWelcome=YES
```

- Ở bước **Inject Drivers**, đặt **Choose a selection profile = Nothing** và **chọn "Install all drivers from the selection profile"** kết hợp biến `DriverGroup001`. Cách này khiến MDT chỉ nạp driver trong nhánh khớp `Make\Model` của máy đang chạy → 5420 chỉ nhận driver 5420.

> Đây là pattern chuẩn để sau này thêm model mới chỉ cần: tạo nhánh driver mới + thêm 1 khối `[Tên Model]` trong CustomSettings.ini. Không phải sửa task sequence.

### 9.3. Phần mềm theo model (nếu cần)
Trong bước **Install Application**, mở tab **Options → Add Condition → Task Sequence Variable**:
- Điều kiện: `Model` `equals` `Latitude 5420` → app chỉ cài cho 5420.
- Tương tự có thể điều kiện theo phòng ban bằng biến tùy chỉnh.

### 9.4. CustomSettings.ini — tự động hóa, bớt hỏi
Chuột phải Deployment Share → **Properties → Rules**. Ví dụ cấu hình giảm thao tác tay:

```ini
[Settings]
Priority=Model, Default

[Latitude 5420]
DriverGroup001=Dell Inc.\Latitude 5420
DriverSelectionProfile=Nothing

[Default]
OSInstall=YES
TimeZoneName=SE Asia Standard Time
KeyboardLocale=en-US
UserLocale=vi-VN
JoinDomain=congty.local
DomainAdmin=svc_join
DomainAdminDomain=CONGTY
DomainAdminPassword=*****
MachineObjectOU=OU=Workstations,DC=congty,DC=local
SkipDomainMembership=YES
SkipUserData=YES
SkipLocaleSelection=YES
SkipTimeZone=YES
SkipApplications=NO
SkipComputerName=NO
SkipSummary=YES
SkipFinalSummary=YES
FinishAction=REBOOT
```

> Mật khẩu domain để plaintext trong file này là rủi ro bảo mật — siết quyền NTFS thư mục `Control`, hoặc dùng tài khoản join chuyên dụng quyền tối thiểu. Có thể tách mật khẩu sang `Bootstrap.ini` và khóa truy cập share.

### 9.5. Bootstrap.ini (kết nối tới share khi đang ở WinPE)
Cùng cửa sổ Properties → **Bootstrap.ini**:

```ini
[Settings]
Priority=Default

[Default]
DeployRoot=\\TEN-SERVER\DeploymentShare$
UserID=svc_deploy
UserDomain=CONGTY
UserPassword=*****
SkipBDDWelcome=YES
```

---

## 10. Update Deployment Share & nạp boot image vào WDS

1. Chuột phải Deployment Share → **Update Deployment Share** → *Completely regenerate the boot images* (lần đầu).
2. MDT sinh ra file boot WIM tại `D:\DeploymentShare\Boot\LiteTouchPE_x64.wim`.
3. Mở **WDS console → Boot Images → Add Boot Image** → trỏ tới file WIM trên.
4. (Nếu cần driver WinPE) gán nhánh WinPE driver cho boot image trong tab **Windows PE** của Properties Deployment Share, rồi update lại.

---

## 11. Deploy lên máy Latitude 5420

### 11.1. Chuẩn bị BIOS máy 5420 (bắt buộc cho Windows 11)
Vào BIOS (phím **F2** lúc khởi động):
- **Boot Mode: UEFI** (tắt Legacy).
- **Secure Boot: Enabled**.
- **TPM (Security → TPM 2.0): Enabled** — Win11 yêu cầu.
- **SATA Operation**: nếu kẹt nhận ổ, thử **AHCI** (mặc định RAID/RST đôi khi cần driver storage trong WinPE).
- Bật **Enable UEFI Network Stack / Integrated NIC = Enabled with PXE** để boot mạng.

### 11.2. Boot & chạy
1. Khởi động máy → bấm **F12** → chọn **Onboard NIC (IPv4) / UEFI PXE**.
2. Máy nhận WDS → tải WinPE → hiện **MDT Wizard**.
3. Chọn task sequence `Deploy Win11 - Latitude 5420`.
4. Phần lớn bước còn lại tự chạy nhờ CustomSettings.ini: phân vùng → cài Win11 → inject driver 5420 → cài phần mềm → join domain → reboot.

### 11.3. Nghiệm thu
- Device Manager: không còn thiết bị "unknown" (driver 5420 nạp đủ).
- `winver`: đúng phiên bản Win11.
- Máy đã join domain, phần mềm chuẩn đã có mặt.

---

## 12. Mở rộng cho model khác sau này

Khi cần thêm model (ví dụ Latitude 5440, OptiPlex...):
1. Tải driver pack model đó → tạo nhánh `Out-of-Box Drivers\Dell Inc.\<Model>`.
2. Thêm khối `[<Model>]` trong CustomSettings.ini với `DriverGroup001=Dell Inc.\<Model>`.
3. (Nếu khác bộ phần mềm) thêm điều kiện `Model equals <Model>` cho các app.

→ Một task sequence dùng chung, driver và app tự rẽ nhánh theo `%Model%`. Đây chính là mô hình "1 hạ tầng – nhiều model theo chuẩn".

---

## Phụ lục: Checklist nhanh

- [ ] Cài đúng thứ tự ADK → WinPE → MDT
- [ ] WDS cấu hình PXE, cùng subnet/DHCP relay
- [ ] Import Windows 11 (full source)
- [ ] Driver pack 5420 Win11 → nhánh `Dell Inc.\Latitude 5420` (tên khớp WMI)
- [ ] Phần mềm đóng gói silent
- [ ] Task sequence + bước Install Application
- [ ] CustomSettings.ini có khối `[Latitude 5420]` + DriverGroup
- [ ] Update Deployment Share + add Boot Image vào WDS
- [ ] BIOS 5420: UEFI + Secure Boot + TPM 2.0 + PXE
- [ ] Test 1 máy trước khi triển khai hàng loạt
