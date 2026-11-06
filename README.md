<div align="center">

# 🖥️ آموزش ساخت سرور ویندوز ۶ ساعته رایگان در اندروید

![Windows](https://img.shields.io/badge/Windows-Server-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub-Actions-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)
![Tailscale](https://img.shields.io/badge/Tailscale-VPN-242424?style=for-the-badge&logo=tailscale&logoColor=white)
![Android](https://img.shields.io/badge/Android-Remote_Desktop-3DDC84?style=for-the-badge&logo=android&logoColor=white)

</div>

<div dir="rtl">

---

## 🧰 ابزارهای استفاده شده

| ابزار | لینک |
|:---|:---|
| 🌐 سایت Tailscale | [tailscale.com](https://tailscale.com) |
| 🐙 سایت GitHub | [github.com](https://github.com) |
| 📱 اپلیکیشن Tailscale (Google Play) | [دانلود](https://play.google.com/store/apps/details?id=com.tailscale.ipn) |
| 🔑 صفحه Auth keys | [console.tailscale.com](https://console.tailscale.com/admin/settings/keys) |
| 🖱️ اپلیکیشن Remote Desktop 8 (Myket) | [دانلود](https://myket.ir/app/com.microsoft.rdc.android) |

---

## 📋 مراحل کار

### 1️⃣ مرحله اول: ساخت اکانت Tailscale

1. وارد سایت [tailscale.com](https://tailscale.com) شوید و یک **اکانت** بسازید.
2. اپلیکیشن **Tailscale** را از لینک Google Play دانلود کنید.
3. بعد از وارد شدن به اپلیکیشن، روی دکمه **Connect** (اتصال به اکانت) بزنید.
4. وارد سایت tailscale.com می‌شوید؛ اپلیکیشن و اکانت خود را به هم وصل کنید.

---

### 2️⃣ مرحله دوم: ساخت Auth key

1. وارد این لینک شوید:
   [https://console.tailscale.com/admin/settings/keys](https://console.tailscale.com/admin/settings/keys)
2. روی گزینه **Auth keys** کلیک کنید.
3. روی **Generate auth key** بزنید.
4. در فیلد **Description** یک نام انتخاب کنید.
5. گزینه **Reusable** را **حتماً فعال** کنید.
6. بقیه گزینه‌ها را **تغییر ندهید**.

---

### 3️⃣ مرحله سوم: ساخت Repository در GitHub

1. وارد [GitHub](https://github.com) شوید.
2. یک **Repository** بسازید.
3. وارد Repository شوید. آدرس آن به این شکل است:

```
https://github.com/my-user/repository-name
```

---

### 4️⃣ مرحله چهارم: ساخت فایل Workflow

1. روی **Create new file** بزنید.
2. در قسمت **Name your file** دقیقاً این مقدار را وارد کنید:

```
.github/workflows/windows-rdp.yml
```

> ⚠️ **توجه:** مقدار با کاراکتر نقطه `.` شروع می‌شود و با فرمت `yml` تمام می‌شود.

3. محتوای زیر را داخل فایل قرار دهید:

```yaml
name: Windows Cloud RDP

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: windows-latest
    timeout-minutes: 360

    steps:
      - name: 1. Enable Remote Desktop & Firewall
        run: |
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -name "fDenyTSConnections" -Value 0
          Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -name "UserAuthentication" -Value 0

      - name: 2. Create Admin Account
        run: |
          $Password = ConvertTo-SecureString "P@ssw0rd1qazXSW@#" -AsPlainText -Force
          New-LocalUser -Name "NvDAdmin" -Password $Password -PasswordNeverExpires -ErrorAction SilentlyContinue
          Add-LocalGroupMember -Group "Administrators" -Member "NvDAdmin" -ErrorAction SilentlyContinue
          net user runneradmin P@ssw0rd1qazXSW@#

      - name: 3. Install & Connect Tailscale VPN
        env:
          TAILSCALE_AUTHKEY: ${{ secrets.TAILSCALE_AUTHKEY }}
        run: |
          # Download and install Tailscale silently
          Invoke-WebRequest "https://pkgs.tailscale.com/stable/tailscale-setup-latest-amd64.msi" -OutFile "C:\tailscale.msi"
          Start-Process msiexec -ArgumentList "/i C:\tailscale.msi /quiet /norestart" -Wait

          # Connect to Tailscale network
          Start-Sleep -Seconds 5
          & "C:\Program Files\Tailscale\tailscale.exe" up --authkey="$Env:TAILSCALE_AUTHKEY" --hostname="github-rdp-server"

          Start-Sleep -Seconds 8

          # Get Tailscale IP
          $ts_ip = & "C:\Program Files\Tailscale\tailscale.exe" ip -4

          Write-Host ""
          Write-Host "==================================================" -ForegroundColor Green
          Write-Host "  >>> RDP SERVER IS READY! <<<" -ForegroundColor Cyan
          Write-Host "  Tailscale IP : $ts_ip" -ForegroundColor Yellow
          Write-Host "  Username     : NvdAdmin" -ForegroundColor Yellow
          Write-Host "  Password     : P@ssw0rd1qazXSW@#" -ForegroundColor Yellow
          Write-Host "  >> Open mstsc and connect to: $ts_ip" -ForegroundColor Magenta
          Write-Host "==================================================" -ForegroundColor Green

      - name: 4. Keep Server Alive (48 Hours)
        run: |
          for ($i = 1; $i -le 2880; $i++) {
            Start-Sleep -Seconds 60
            Write-Host "[$i/2880] Server running..."
          }
```

> ⚠️ **توجه:** کد را از کلمه `name` شروع کنید تا آخرین `}`.

4. فایل را **Save** (Commit) کنید.

---

### 5️⃣ مرحله پنجم: ساخت Secret

1. وارد **Settings** همین Repository شوید.
2. قسمت **Secrets and variables** ← **Actions** را باز کنید.
3. روی **New repository secret** بزنید.
4. در فیلد **Name** دقیقاً این را وارد کنید:

```
TAILSCALE_AUTHKEY
```

> ⚠️ **توجه:** مقدار از حرف `T` تا حرف `Y` باشد.

5. در فیلد **Secret**، همان کلیدی که در مرحله دوم ساختید (Auth key) را وارد کنید.
6. **Save** کنید.

---

### 6️⃣ مرحله ششم: اجرای Workflow

1. از نوار بالای GitHub وارد قسمت **Actions** شوید.
2. وب‌سایت را روی **نسخه مرورگر دسکتاپ (Desktop site)** تنظیم کنید تا **Workflows** و گزینه **Windows Cloud RDP** دیده شود.
3. روی **Windows Cloud RDP** بزنید.
4. روی **Run workflow** بزنید تا Action اجرا شود.
5. یک باکس به نام **Build** دیده می‌شود؛ وارد آن شوید و **منتظر بمانید**.
6. صبر کنید تا Job به‌طور کامل اجرا شود.
7. در قسمت **Install & Connect Tailscale VPN** کلیک کنید تا محتوای داخل آن نمایش داده شود.

---

### 7️⃣ مرحله هفتم: اطلاعات سرور

بعد از اجرا، چنین چیزی می‌بینید:

```
==================================================
  >>> RDP SERVER IS READY! <<<
  Tailscale IP : 100.XX.XX.00
  Username     : NvdAdmin
  Password     : P@ssw0rd1qazXSW@#
  >> Open mstsc and connect to: 100.XX.XX.00
==================================================
```

🎉 **تبریک! سرور شما آماده است.**

IP، Username و Password شما قابل مشاهده است. شما می‌توانید **سرور ۶ ساعته** خود را استفاده کنید.

اگر سرور خاموش شد، می‌توانید وارد **Actions** شوید و دوباره **Workflow** را اجرا کنید.

---

### 8️⃣ مرحله هشتم: وارد شدن به سرور

قبل از وارد شدن به سرور، از ابزارهایی مثل **Remote Desktop 8** در مایکت استفاده کنید:

👉 [https://myket.ir/app/com.microsoft.rdc.android](https://myket.ir/app/com.microsoft.rdc.android)

> ❗ **مهم:** ابتدا وارد اپلیکیشن **Tailscale** شوید و گزینه **Connect** (یا **Connect to VPN**) را بزنید و **مجوز VPN** را بدهید. بعد وارد سرور شوید؛ وگرنه نمی‌توانید وارد سرور شوید.

---

</div>

<div align="center">

### ✅ تمام!

</div>
