# Step 4: Fallback و Multiplexing

این مرحله خیلی مهمه. باید طوری پیاده‌سازی کنیم که اگر کسی با پروتکل درست متصل نشد، ترافیک رو به یک وب‌سرور عادی بفرستیم (مثل Trojan).

## مشکل: مصرف شدن بایت‌ها

وقتی می‌خوایم چند بایت اول رو بخونیم تا ببینیم Reflex هست یا نه، اون بایت‌ها از connection مصرف می‌شن. اگر بعداً بخوایم اتصال رو به وب‌سرور بفرستیم، اون بایت‌ها دیگه نیستن و هندشیک TLS وب‌سرور خراب می‌شه.

## راه‌حل: bufio.Peek

از `bufio.Reader` استفاده می‌کنیم که به ما اجازه می‌ده بایت‌ها رو ببینیم بدون اینکه مصرف بشن:

```go
import (
    "bufio"
    "encoding/binary"
    "io"
    "github.com/xtls/xray-core/common/net"
    "github.com/xtls/xray-core/transport/internet/stat"
    "github.com/xtls/xray-core/features/routing"
)

const (
    // حداقل اندازه برای تشخیص handshake
    // Magic number (4) + حداقل اندازه handshake
    ReflexMinHandshakeSize = 64
)

func (h *Handler) Process(ctx context.Context, network net.Network, conn stat.Connection, dispatcher routing.Dispatcher) error {
    // Wrap connection در bufio.Reader
    reader := bufio.NewReader(conn)
    
    // Peek کردن چند بایت اول (بدون مصرف)
    peeked, err := reader.Peek(ReflexMinHandshakeSize)
    if err != nil {
        return err
    }
    
    // چک کردن که آیا Reflex هست یا نه
    if h.isReflexHandshake(peeked) {
        // Reflex هست - پردازش کن
        // باید منطق handshake رو از step2 صدا بزنی:
        if len(peeked) >= 4 {
            magic := binary.BigEndian.Uint32(peeked[0:4])
            if magic == ReflexMagic {
                return h.handleReflexMagic(reader, conn, dispatcher, ctx)
            }
        }
        if h.isHTTPPostLike(peeked) {
            return h.handleReflexHTTP(reader, conn, dispatcher, ctx)
        }
        // اگه هیچکدوم نبود، به fallback برو
        return h.handleFallback(ctx, reader, conn)
    } else {
        // Reflex نیست - به fallback بفرست
        return h.handleFallback(ctx, reader, conn)
    }
}
```

## تشخیص پروتکل Reflex

باید یه روش برای تشخیص بسته Reflex داشته باشیم. دو راه داریم:

### روش 1: Magic Number (سریع‌تر)

یه magic number در ابتدای بسته می‌ذاریم. این روش سریع‌تره ولی کمتر پنهان‌کاره:

```go
const ReflexMagic = 0x5246584C  // "REFX" در ASCII

func (h *Handler) isReflexMagic(data []byte) bool {
    if len(data) < 4 {
        return false
    }
    
    magic := binary.BigEndian.Uint32(data[0:4])
    return magic == ReflexMagic
}
```

### روش 2: HTTP POST-like (پنهان‌کارتر)

بسته اولیه رو طوری می‌سازیم که شبیه HTTP POST باشه. این روش پنهان‌کارتره ولی parse کردنش پیچیده‌تره:

```go
func (h *Handler) isHTTPPostLike(data []byte) bool {
    // چک کردن که آیا با "POST" شروع می‌شه
    if len(data) < 4 {
        return false
    }
    
    // چک کردن HTTP method
    if string(data[0:4]) != "POST" {
        return false
    }
    
    // چک کردن HTTP version
    // باید "HTTP/1.1" یا "HTTP/2" باشه
    // اینجا می‌تونی بیشتر چک کنی
    
    return true
}
```

### ترکیب هر دو (بهترین راه)

می‌تونی هر دو رو پشتیبانی کنی. اول magic number رو چک می‌کنی (سریع)، اگه نبود، HTTP POST-like رو چک می‌کنی:

```go
func (h *Handler) isReflexHandshake(data []byte) bool {
    // اول magic number رو چک کن (سریع‌تر)
    if h.isReflexMagic(data) {
        return true
    }
    
    // بعد HTTP POST-like رو چک کن (پنهان‌کارتر)
    if h.isHTTPPostLike(data) {
        return true
    }
    
    return false
}
```

اینطوری هم سرعت داره (magic number برای تشخیص سریع) هم پنهان‌کاری (HTTP POST-like برای ترافیک واقعی).

## Fallback به وب‌سرور

اگر Reflex نبود، باید اتصال رو به وب‌سرور بفرستیم. چون از `bufio.Reader` استفاده کردیم، بایت‌های peek شده هنوز در reader هستن و می‌تونیم اون‌ها رو بخونیم:

```go
import (
    "fmt"
    "net"
    "errors"
)

func (h *Handler) handleFallback(ctx context.Context, reader *bufio.Reader, conn stat.Connection) error {
    if h.fallback == nil {
        return errors.New("no fallback configured")
    }
    
    // ساخت یک connection wrapper که reader رو wrap می‌کنه
    // این باعث می‌شه بایت‌های peek شده هم خوانده بشن
    wrappedConn := &preloadedConn{
        Reader: reader,
        Connection: conn,
    }
    
    // اتصال به وب‌سرور محلی
    target, err := net.Dial("tcp", fmt.Sprintf("127.0.0.1:%d", h.fallback.Dest))
    if err != nil {
        return err
    }
    defer target.Close()
    
    // کپی کردن داده‌ها بین دو connection
    go io.Copy(target, wrappedConn)
    io.Copy(wrappedConn, target)
    
    return nil
}
```

## Preloaded Connection Wrapper

برای اینکه بایت‌های peek شده رو هم بخونیم، باید یک wrapper بسازیم:

```go
type preloadedConn struct {
    *bufio.Reader
    stat.Connection
}

func (pc *preloadedConn) Read(b []byte) (int, error) {
    return pc.Reader.Read(b)
}

func (pc *preloadedConn) Write(b []byte) (int, error) {
    return pc.Connection.Write(b)
}
```

## Multiplexing روی یک پورت

با این روش، می‌تونی هم Reflex و هم وب‌سرور رو روی یه پورت اجرا کنی. Xray به صورت خودکار اتصالات رو به handler مناسب می‌فرسته.

**چطوری کار می‌کنه؟**

1. یه اتصال TCP ورودی می‌آد
2. Handler چند بایت اول رو peek می‌کنه (بدون مصرف)
3. اگه Reflex بود → پردازش Reflex
4. اگه Reflex نبود → به fallback (وب‌سرور) می‌فرسته

این یعنی می‌تونی یه سرور رو طوری تنظیم کنی که:
- روی پورت 443 (HTTPS) گوش بده
- اگه Reflex بود → پروتکل Reflex رو پردازش کنه
- اگه HTTPS عادی بود → به nginx یا هر وب‌سرور دیگه‌ای بفرسته

از دید یه ناظر، سرور دقیقاً مثل یه وب‌سرور HTTPS عادی به نظر می‌رسه. هیچ نشونه‌ای از پراکسی نیست.

## تست کردن

برای تست، می‌تونید:

1. یک وب‌سرور ساده روی پورت 80 اجرا کنید (مثلاً با nginx یا یک سرور Go ساده)
2. Xray رو با fallback به پورت 80 تنظیم کنید
3. با مرورگر به پورت Xray وصل بشید - باید صفحه وب رو ببینید
4. با کلاینت Reflex وصل بشید - باید پروتکل Reflex کار کنه

## چک‌لیست

- [ ] استفاده از bufio.Peek برای تشخیص پروتکل
- [ ] تشخیص درست Reflex از ترافیک عادی
- [ ] Fallback به وب‌سرور کار می‌کنه
- [ ] بایت‌های peek شده درست به وب‌سرور فرستاده می‌شن
- [ ] تست با مرورگر و کلاینت Reflex موفقیت‌آمیزه

## بعدی

وقتی fallback رو تموم کردید، برید سراغ [Step 5](step5-advanced.md) برای قابلیت‌های پیشرفته (Traffic Morphing و ...).

