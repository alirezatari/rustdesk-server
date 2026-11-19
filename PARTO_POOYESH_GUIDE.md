# راهنمای تغییر سورس hbbs برای هوک کنترل دسترسی پرتو پویش

## گام ۱: فورک و کلون

۱. به آدرس https://github.com/rustdesk/rustdesk-server بروید
۲. دکمه Fork را بزنید تا یک کپی در اکانت GitHub خودتان بسازد
۳. فایل‌های این پوشه را (`rustdesk-server-fork`) داخل ریپوی فورک شده commit کنید:
   - `.github/workflows/build.yml` ← فایل GitHub Actions برای بیلد خودکار

## گام ۲: اضافه کردن وابستگی reqwest

در فایل `Cargo.toml` ریپوی فورک، این خط را به بخش `[dependencies]` اضافه کنید:

```toml
reqwest = { version = "0.12", default-features = false, features = ["blocking", "rustls-tls"] }
tokio = { version = "1", features = ["full"] }   # احتمالاً از قبل هست
```

## گام ۳: پیدا کردن محل هوک در سورس

فایل هدف: `src/rendezvous_server.rs`

در این فایل، تابعی که درخواست relay یا peer connection را پردازش می‌کند را پیدا کنید.
کلیدواژه‌های جستجو در فایل:
- `relay` یا `Relay`
- `punch_hole` یا `PunchHole`
- `fn handle_` + نام پیام

## گام ۴: کد هوک را اضافه کنید

**قبل** از هر جایی که سرور اطلاعات peer مقصد را به مبدأ می‌فرستد یا رله را برقرار می‌کند، این کد را وارد کنید:

```rust
// === Parto Pooyesh Access Control Hook ===
// این تابع را بالای فایل اضافه کنید:
fn check_connection_allowed(from_id: &str, to_id: &str) -> bool {
    let url = format!(
        "http://127.0.0.1:5000/api/authorize?from={}&to={}",
        from_id, to_id
    );
    match reqwest::blocking::get(&url) {
        Ok(resp) => resp.status().is_success(),
        Err(e) => {
            log::error!("Auth API unreachable: {}", e);
            false // در صورت قطع API، اتصال را رد کن
        }
    }
}

// در بدنه هندلر، قبل از تایید اتصال:
if !check_connection_allowed(&from_id, &to_id) {
    log::warn!("Blocked: {} -> {}", from_id, to_id);
    return; // پکت را رها کن
}
// === پایان هوک ===
```

## گام ۵: Push و بیلد خودکار

```bash
git add -A
git commit -m "Add Parto Pooyesh access control hook"
git push origin main
```

بعد از push، در تب **Actions** ریپوی GitHub بروید.
بیلد خودکار شروع می‌شود. پس از ~۵ دقیقه فایل‌های:
- `hbbs.exe`
- `hbbr.exe`

را از بخش **Artifacts** دانلود کنید.

## گام ۶: نصب روی سرور ویندوزی

فایل‌های دانلود‌شده را روی سرور ویندوزی کپی کنید و اجرا کنید:

```powershell
# هر دو سرویس باید همزمان اجرا شوند
Start-Process -FilePath "hbbs.exe" -ArgumentList "-r YOUR_SERVER_IP"
Start-Process -FilePath "hbbr.exe"
```

برای اجرا به عنوان Windows Service از NSSM استفاده کنید:
```powershell
nssm install hbbs "C:\RustDesk\hbbs.exe" "-r YOUR_SERVER_IP"
nssm install hbbr "C:\RustDesk\hbbr.exe"
nssm start hbbs
nssm start hbbr
```

## پورت‌های مورد نیاز (در Firewall باز شود):
- TCP/UDP 21115
- TCP/UDP 21116
- TCP 21117
- TCP 21118
- TCP 21119
