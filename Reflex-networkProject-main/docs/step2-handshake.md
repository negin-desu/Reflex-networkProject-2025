# Step 2: Handshake و احراز هویت

در این مرحله، handshake ضمنی و احراز هویت رو پیاده‌سازی می‌کنیم.

## هندشیک ضمنی

ایده اصلی اینه که به جای یه handshake واضح (مثل TLS ClientHello/ServerHello)، تبادل کلید رو داخل بسته‌های اولیه که شبیه ترافیک عادی هستن پنهان کنیم. این یعنی از دید یه ناظر، ترافیک شبیه یه API call عادی به نظر می‌رسه، نه یه پروتکل پراکسی.

## بسته اولیه کلاینت

کلاینت یه بسته می‌فرسته که شبیه یه درخواست HTTP POST هست. بذار ببینیم ساختار دقیقش چیه:

### ساختار HTTP POST-like

برای اینکه شبیه ترافیک عادی باشه، بسته اولیه باید شبیه یه HTTP POST request باشه:

```
POST /api/v1/endpoint HTTP/1.1
Host: example.com
Content-Type: application/json
Content-Length: <length>

{
  "data": "<base64 encoded handshake data>"
}
```

ولی زیر پوستش، `data` field شامل این چیزهاست:
- کلید عمومی موقت (X25519) - 32 بایت
- UUID کاربر - 16 بایت (یا string)
- درخواست سیاست (Policy Request) - رمزنگاری شده با pre-shared key
- Timestamp - برای جلوگیری از replay
- Nonce - 16 بایت برای جلوگیری از replay

**نکته**: Pre-shared key رو می‌تونی در config تعریف کنی یا از UUID کاربر به عنوان seed استفاده کنی. برای سادگی، می‌تونی از UUID به عنوان pre-shared key استفاده کنی (hash شده).

```go
type ClientHandshake struct {
    PublicKey [32]byte  // کلید عمومی X25519
    UserID    [16]byte  // UUID (16 بایت)
    PolicyReq []byte    // درخواست سیاست (رمزنگاری شده با pre-shared key)
    Timestamp int64     // مهر زمانی (Unix timestamp)
    Nonce     [16]byte  // برای جلوگیری از replay
}

// ساختار کامل بسته اولیه (قبل از base64 encoding)
type ClientHandshakePacket struct {
    Magic      [4]byte  // برای تشخیص سریع (اختیاری)
    Handshake  ClientHandshake
}

// یا می‌تونی از magic number استفاده کنی (ساده‌تر برای تشخیص)
const ReflexMagic = 0x5246584C  // "REFX" در ASCII
```

**Magic Number یا HTTP POST-like؟**: می‌تونی هر دو رو پشتیبانی کنی:
1. اول magic number رو چک کن (سریع‌تر)
2. اگه magic number نبود، HTTP POST-like رو parse کن
3. اگه هیچکدوم نبود، به fallback بفرست

اینطوری هم سرعت داره (magic number) هم پنهان‌کاری (HTTP POST-like).

## پاسخ سرور

سرور با چیزی شبیه پاسخ HTTP 200 جواب می‌ده. داخلش:
- کلید عمومی موقت سرور
- اعطای سیاست (رمزنگاری شده)

```go
type ServerHandshake struct {
    PublicKey [32]byte  // کلید عمومی سرور
    PolicyGrant []byte  // اعطای سیاست (رمزنگاری شده)
}
```

## تبادل کلید

بعد از دریافت کلیدهای عمومی، هر دو طرف کلید مشترک رو محاسبه می‌کنن:

```go
import "golang.org/x/crypto/curve25519"

func deriveSharedKey(privateKey, peerPublicKey [32]byte) [32]byte {
    var shared [32]byte
    curve25519.ScalarMult(&shared, &privateKey, &peerPublicKey)
    return shared
}
```

از این کلید مشترک، کلید جلسه رو استخراج می‌کنیم:

```go
import (
    "crypto/sha256"
    "golang.org/x/crypto/hkdf"
)

func deriveSessionKey(sharedKey [32]byte, salt []byte) []byte {
    hkdf := hkdf.New(sha256.New, sharedKey[:], salt, []byte("reflex-session"))
    sessionKey := make([]byte, 32)
    hkdf.Read(sessionKey)
    return sessionKey
}
```

## احراز هویت با UUID

بعد از تبادل کلید، باید کاربر رو با UUID احراز هویت کنیم:

```go
import (
    "encoding/binary"
    "errors"
    "github.com/google/uuid"
)

func (h *Handler) authenticateUser(userID [16]byte) (*protocol.MemoryUser, error) {
    // تبدیل [16]byte به string UUID
    userIDStr := uuid.UUID(userID).String()
    
    for _, user := range h.clients {
        if user.Account.(*MemoryAccount).Id == userIDStr {
            return user, nil
        }
    }
    return nil, errors.New("user not found")
}

// یا اگه می‌خوای مستقیماً با [16]byte کار کنی:
func (h *Handler) authenticateUserBytes(userID [16]byte) (*protocol.MemoryUser, error) {
    for _, user := range h.clients {
        accountID := user.Account.(*MemoryAccount).Id
        // تبدیل string UUID به [16]byte و مقایسه
        parsedUUID, err := uuid.Parse(accountID)
        if err != nil {
            continue
        }
        if parsedUUID == uuid.UUID(userID) {
            return user, nil
        }
    }
    return nil, errors.New("user not found")
}
```

## پیاده‌سازی در Process

حالا باید منطق handshake رو در متد `Process` اضافه کنیم. ولی اول باید از `bufio.Reader` استفاده کنیم تا بتونیم peek کنیم (برای fallback):

```go
import (
    "bufio"
    "encoding/binary"
    "encoding/base64"
    "encoding/json"
    "io"
    "github.com/xtls/xray-core/common/net"
    "github.com/xtls/xray-core/common/protocol"
    "github.com/xtls/xray-core/transport/internet/stat"
    "github.com/xtls/xray-core/features/routing"
)

func (h *Handler) Process(ctx context.Context, network net.Network, conn stat.Connection, dispatcher routing.Dispatcher) error {
    // Wrap connection در bufio.Reader برای peek
    reader := bufio.NewReader(conn)
    
    // Peek کردن چند بایت اول
    peeked, err := reader.Peek(64) // حداقل برای magic number یا HTTP header
    if err != nil {
        return err
    }
    
    // چک کردن magic number (سریع‌تر)
    if len(peeked) >= 4 {
        magic := binary.BigEndian.Uint32(peeked[0:4])
        if magic == ReflexMagic {
            // Magic number پیدا شد - parse کن
            return h.handleReflexMagic(reader, conn, dispatcher, ctx)
        }
    }
    
    // چک کردن HTTP POST-like
    // باید این تابع رو خودت پیاده‌سازی کنی (در step4 توضیح داده شده)
    if h.isHTTPPostLike(peeked) {
        return h.handleReflexHTTP(reader, conn, dispatcher, ctx)
    }
    
    // هیچکدوم نبود - به fallback بفرست
    return h.handleFallback(ctx, reader, conn)
}

func (h *Handler) handleReflexMagic(reader *bufio.Reader, conn stat.Connection, dispatcher routing.Dispatcher, ctx context.Context) error {
    // خواندن magic number (4 بایت)
    magic := make([]byte, 4)
    io.ReadFull(reader, magic)
    
    // خواندن handshake packet
    var packet ClientHandshakePacket
    // ... parse کردن بقیه بسته
    
    return h.processHandshake(reader, conn, dispatcher, ctx, packet.Handshake)
}

func (h *Handler) handleReflexHTTP(reader *bufio.Reader, conn stat.Connection, dispatcher routing.Dispatcher, ctx context.Context) error {
    // Parse کردن HTTP POST request
    // استخراج base64 encoded data
    // decode کردن و parse کردن ClientHandshake
    
    // اینجا باید HTTP request رو parse کنی
    // برای سادگی، می‌تونی از یه HTTP parser استفاده کنی
    // یا خودت parse کنی
    
    var clientHS ClientHandshake
    // ... parse کردن از HTTP POST
    
    return h.processHandshake(reader, conn, dispatcher, ctx, clientHS)
}

func (h *Handler) processHandshake(reader *bufio.Reader, conn stat.Connection, dispatcher routing.Dispatcher, ctx context.Context, clientHS ClientHandshake) error {
    // تولید کلید موقت سرور
    // باید این تابع رو خودت پیاده‌سازی کنی:
    // func generateKeyPair() (privateKey [32]byte, publicKey [32]byte) {
    //     privateKey = [32]byte{...} // تولید کلید خصوصی تصادفی
    //     curve25519.ScalarBaseMult(&publicKey, &privateKey)
    //     return
    // }
    serverPrivateKey, serverPublicKey := generateKeyPair()
    
    // محاسبه کلید مشترک
    sharedKey := deriveSharedKey(serverPrivateKey, clientHS.PublicKey)
    sessionKey := deriveSessionKey(sharedKey, []byte("reflex-session"))
    
    // احراز هویت
    user, err := h.authenticateUser(clientHS.UserID)
    if err != nil {
        // اگر احراز هویت ناموفق بود، به fallback برو
        return h.handleFallback(ctx, reader, conn)
    }
    
    // ارسال پاسخ handshake (شبیه HTTP 200)
    serverHS := ServerHandshake{
        PublicKey: serverPublicKey,
        // باید این تابع رو خودت پیاده‌سازی کنی:
        // PolicyGrant: h.encryptPolicyGrant(user, sessionKey), // policy grant رو رمزنگاری کن
        PolicyGrant: []byte{}, // placeholder - باید پیاده‌سازی بشه
    }
    
    // ارسال پاسخ (شبیه HTTP 200)
    // باید این تابع رو خودت پیاده‌سازی کنی:
    // response := h.formatHTTPResponse(serverHS)
    // یا می‌تونی مستقیماً JSON بفرستی:
    response := []byte("HTTP/1.1 200 OK\r\nContent-Type: application/json\r\n\r\n{\"status\":\"ok\"}")
    conn.Write(response)
    
    // حالا جلسه برقرار شده، می‌تونیم داده‌ها رو پردازش کنیم
    return h.handleSession(ctx, reader, conn, dispatcher, sessionKey, user)
}
```

## مدیریت خطا

اگر مشکلی پیش اومد، نباید اتصال رو ناگهان ببندید. بهتره یک پاسخ عادی بفرستید (مثلاً HTTP 403) و بعد اتصال رو ببندید.

## چک‌لیست

- [ ] ساختار ClientHandshake و ServerHandshake تعریف شده
- [ ] تبادل کلید X25519 پیاده‌سازی شده
- [ ] استخراج کلید جلسه کار می‌کنه
- [ ] احراز هویت با UUID کار می‌کنه
- [ ] مدیریت خطا درست پیاده‌سازی شده

## بعدی

وقتی handshake رو تموم کردید، برید سراغ [Step 3](step3-encryption.md) برای رمزنگاری و پردازش بسته‌ها.

