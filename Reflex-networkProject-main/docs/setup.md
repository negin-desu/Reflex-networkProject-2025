# راه‌اندازی محیط

قبل از شروع، باید محیط توسعه رو آماده کنید. این کار خیلی ساده‌ست.

## نصب Go

1. برید به [golang.org/download](https://go.dev/dl/)
2. آخرین نسخه Go رو برای سیستم‌عامل خودتون دانلود کنید
3. نصب کنید (روی ویندوز: فایل .msi رو اجرا کنید)
4. ترمینال رو باز کنید و تست کنید:

```bash
go version
```

باید چیزی شبیه `go version go1.21.x` ببینید.

**نکته مهم - نسخه Go**: 
- برای پیاده‌سازی پایه (Step 1-4): Go 1.21+ کافیه
- برای ECH (Encrypted Client Hello): نیاز به Go 1.25+ داره (یا کتابخانه‌های اضافی مثل `github.com/cloudflare/circl`)
- برای QUIC: نیاز به Go 1.21+ و کتابخانه `github.com/quic-go/quic-go` داره

اگه می‌خوای Step 5 (قابلیت‌های پیشرفته) رو پیاده‌سازی کنی، باید نسخه Go رو چک کنی. برای جزئیات بیشتر، برو سراغ [Step 5](step5-advanced.md).

## نصب Git

1. برید به [git-scm.com/download](https://git-scm.com/downloads)
2. Git رو دانلود و نصب کنید
3. ترمینال رو باز کنید و تست کنید:

```bash
git --version
```

## کلون کردن ریپو Reflex

حالا باید ریپو Reflex رو کلون کنید:

```bash
git clone https://github.com/soroushdeimi/reflex.git
cd reflex
```

## اضافه کردن Xray-Core

اگر پوشه `xray-core` وجود نداره، باید Xray-Core رو اضافه کنید:

```bash
git clone https://github.com/XTLS/Xray-core.git xray-core
```

یا اگر می‌خواید به عنوان submodule اضافه کنید:

```bash
git submodule add https://github.com/XTLS/Xray-core.git xray-core
git submodule update --init --recursive
```

## ساخت اولین بیلد

برید توی پوشه xray-core و بیلد کنید:

```bash
cd xray-core
go build -o xray ./main
```

اگر خطایی ندید، یعنی همه چیز درسته. یک فایل `xray` (یا `xray.exe` روی ویندوز) باید ساخته بشه.

## تست کردن

برای اینکه مطمئن بشید همه چیز کار می‌کنه، می‌تونید xray رو با فلگ help اجرا کنید:

```bash
./xray -h
```

باید لیست فلگ‌ها و گزینه‌ها رو ببینید.

## IDE پیشنهادی

برای نوشتن کد، می‌تونید از این‌ها استفاده کنید:
- **VS Code** با افزونه Go
- **GoLand** (اگر دسترسی دارید)
- هر ادیتور دیگه‌ای که دوست دارید

## آماده‌اید؟

حالا که محیط رو راه‌اندازی کردید، برید سراغ [پروتکل Reflex](protocol.md) تا بفهمید چی می‌خواید پیاده‌سازی کنید.

