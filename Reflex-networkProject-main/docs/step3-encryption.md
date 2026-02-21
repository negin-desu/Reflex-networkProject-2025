# Step 3: رمزنگاری و پردازش بسته‌ها

بعد از handshake، باید داده‌ها رو رمزنگاری کنیم و بسته‌ها رو پردازش کنیم.

## ساختار Frame

هر بسته Reflex یک header کوچیک داره:

```
[طول (2 بایت)] [نوع (1 بایت)] [داده‌های رمزنگاری شده]
```

```go
const (
    FrameTypeData = 0x01
    FrameTypePadding = 0x02
    FrameTypeTiming = 0x03
    FrameTypeClose = 0x04
)

type Frame struct {
    Length uint16
    Type   uint8
    Payload []byte
}
```

## رمزنگاری با ChaCha20-Poly1305

برای رمزنگاری از AEAD استفاده می‌کنیم. Go پکیج `crypto/cipher` رو داره:

```go
import (
    "crypto/cipher"
    "golang.org/x/crypto/chacha20poly1305"
)

type Session struct {
    key []byte
    aead cipher.AEAD
    readNonce  uint64
    writeNonce uint64
}

func NewSession(sessionKey []byte) (*Session, error) {
    aead, err := chacha20poly1305.New(sessionKey)
    if err != nil {
        return nil, err
    }
    
    return &Session{
        key: sessionKey,
        aead: aead,
        readNonce: 0,
        writeNonce: 0,
    }, nil
}
```

## خواندن Frame

برای خواندن یک frame:

```go
import (
    "io"
    "encoding/binary"
)

func (s *Session) ReadFrame(reader io.Reader) (*Frame, error) {
    // خواندن header (3 بایت)
    header := make([]byte, 3)
    if _, err := io.ReadFull(reader, header); err != nil {
        return nil, err
    }
    
    length := binary.BigEndian.Uint16(header[0:2])
    frameType := header[2]
    
    // خواندن payload
    encryptedPayload := make([]byte, length)
    if _, err := io.ReadFull(reader, encryptedPayload); err != nil {
        return nil, err
    }
    
    // رمزگشایی
    nonce := make([]byte, 12)
    binary.BigEndian.PutUint64(nonce[4:], s.readNonce)
    s.readNonce++
    
    payload, err := s.aead.Open(nil, nonce, encryptedPayload, nil)
    if err != nil {
        return nil, err
    }
    
    return &Frame{
        Length: length,
        Type: frameType,
        Payload: payload,
    }, nil
}
```

## نوشتن Frame

برای نوشتن یک frame:

```go
import (
    "io"
    "encoding/binary"
)

func (s *Session) WriteFrame(writer io.Writer, frameType uint8, data []byte) error {
    // رمزنگاری
    nonce := make([]byte, 12)
    binary.BigEndian.PutUint64(nonce[4:], s.writeNonce)
    s.writeNonce++
    
    encrypted := s.aead.Seal(nil, nonce, data, nil)
    
    // نوشتن header
    header := make([]byte, 3)
    binary.BigEndian.PutUint16(header[0:2], uint16(len(encrypted)))
    header[2] = frameType
    
    if _, err := writer.Write(header); err != nil {
        return err
    }
    
    // نوشتن payload
    if _, err := writer.Write(encrypted); err != nil {
        return err
    }
    
    return nil
}
```

## پردازش داده‌ها

بعد از خواندن frame، باید نوعش رو چک کنیم و بر اساسش عمل کنیم:

```go
import (
    "bufio"
    "context"
    "errors"
    "github.com/xtls/xray-core/common/buf"
    "github.com/xtls/xray-core/common/net"
    "github.com/xtls/xray-core/common/protocol"
    "github.com/xtls/xray-core/transport/internet/stat"
    "github.com/xtls/xray-core/features/routing"
)

func (h *Handler) handleSession(ctx context.Context, reader *bufio.Reader, conn stat.Connection, dispatcher routing.Dispatcher, sessionKey []byte, user *protocol.MemoryUser) error {
    session, err := NewSession(sessionKey)
    if err != nil {
        return err
    }
    
    for {
        frame, err := session.ReadFrame(reader)
        if err != nil {
            return err
        }
        
        switch frame.Type {
        case FrameTypeData:
            // این داده‌های واقعی کاربره، باید به upstream بفرستیم
            // باید destination رو از frame payload استخراج کنی
            // اینجا یه مثال ساده - باید خودت پیاده‌سازی کنی:
            err := h.handleData(ctx, frame.Payload, conn, dispatcher, session, user)
            if err != nil {
                return err
            }
            // ادامه خواندن frame‌های بعدی
            continue
            
        case FrameTypePadding:
            // دستور padding - فعلاً نادیده می‌گیریم
            continue
            
        case FrameTypeTiming:
            // دستور timing - فعلاً نادیده می‌گیریم
            continue
            
        case FrameTypeClose:
            // بستن اتصال
            return nil
            
        default:
            return errors.New("unknown frame type")
        }
    }
}
```

## پردازش داده‌های واقعی

برای ارسال داده‌های کاربر به upstream، باید از dispatcher استفاده کنیم:

```go
import (
    "errors"
    "github.com/xtls/xray-core/common/buf"
    "github.com/xtls/xray-core/features/routing"
)

func (h *Handler) handleData(ctx context.Context, data []byte, conn stat.Connection, dispatcher routing.Dispatcher, session *Session, user *protocol.MemoryUser) error {
    // ساخت destination از داده‌ها (باید از frame payload استخراج بشه)
    // اینجا یه مثال ساده - باید خودت پیاده‌سازی کنی:
    // 1. parse کردن destination از data (مثلاً address + port)
    // 2. ساخت net.Destination
    dest := net.TCPDestination(net.ParseAddress("example.com"), net.Port(80))
    
    // ارسال به upstream
    link, err := dispatcher.Dispatch(ctx, dest)
    if err != nil {
        return err
    }
    
    // نوشتن داده‌ها به upstream
    buffer := buf.FromBytes(data)
    if err := link.Writer.WriteMultiBuffer(buf.MultiBuffer{buffer}); err != nil {
        return err
    }
    
    // خواندن پاسخ از upstream و ارسال به کلاینت
    // این بخش باید bidirectional باشه - باید در goroutine جداگانه انجام بشه
    // **نکته مهم**: این یک پیاده‌سازی ساده‌ست. در واقعیت باید:
    // 1. یک goroutine برای خواندن از upstream و نوشتن به کلاینت
    // 2. یک goroutine برای خواندن از کلاینت و نوشتن به upstream
    // 3. مدیریت lifecycle و cleanup
    go func() {
        defer link.Writer.Close()
        for {
            mb, err := link.Reader.ReadMultiBuffer()
            if err != nil {
                return
            }
            // رمزنگاری و ارسال به کلاینت
            for _, b := range mb {
                if err := session.WriteFrame(conn, FrameTypeData, b.Bytes()); err != nil {
                    return
                }
                b.Release() // آزاد کردن buffer
            }
        }
    }()
    
    return nil
}
```

**نکته**: این یک پیاده‌سازی ساده‌ست. در واقعیت، باید destination رو از frame payload استخراج کنی و bidirectional forwarding رو پیاده‌سازی کنی.

## محافظت در برابر Replay

برای جلوگیری از replay attack، باید nonce‌ها رو چک کنیم. یک راه ساده اینه که nonce‌ها رو به ترتیب استفاده کنیم (که در کد بالا انجام دادیم). یک راه بهتر اینه که nonce‌های اخیر رو ذخیره کنیم:

```go
type NonceCache struct {
    seen map[uint64]bool
    mu   sync.Mutex
}

func (nc *NonceCache) Check(nonce uint64) bool {
    nc.mu.Lock()
    defer nc.mu.Unlock()
    
    if nc.seen[nonce] {
        return false  // replay detected
    }
    
    nc.seen[nonce] = true
    
    // فقط آخرین 1000 nonce رو نگه دار
    if len(nc.seen) > 1000 {
        // قدیمی‌ترین رو پاک کن
    }
    
    return true
}
```

## چک‌لیست

- [ ] ساختار Frame تعریف شده
- [ ] رمزنگاری با ChaCha20-Poly1305 کار می‌کنه
- [ ] خواندن و نوشتن frame پیاده‌سازی شده
- [ ] پردازش انواع مختلف frame کار می‌کنه
- [ ] محافظت در برابر replay پیاده‌سازی شده

## بعدی

وقتی رمزنگاری رو تموم کردید، برید سراغ [Step 4](step4-fallback.md) برای fallback و multiplexing.

