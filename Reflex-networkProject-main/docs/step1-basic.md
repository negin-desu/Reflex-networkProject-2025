# Step 1: ساختار اولیه

در این مرحله، ساختار اولیه پروتکل رو می‌سازیم.

## ساخت پکیج reflex

اول باید پکیج reflex رو در Xray-Core بسازید. ساختار باید اینطوری باشه:

```
reflex/
  xray-core/
    proxy/
      reflex/
        inbound/
          inbound.go
        outbound/
          outbound.go
        config.proto
```

پوشه‌ها رو بسازید:

```bash
cd xray-core
mkdir -p proxy/reflex/inbound
mkdir -p proxy/reflex/outbound
```

## تعریف config.proto

فایل `proxy/reflex/config.proto` رو بسازید:

```protobuf
syntax = "proto3";

package reflex.proxy;
option go_package = "github.com/xtls/xray-core/proxy/reflex";

message User {
  string id = 1;  // UUID کاربر
  string policy = 2;  // سیاست ترافیک (مثلاً "mimic-http2-api")
}

message Account {
  string id = 1;  // UUID کاربر
}

message InboundConfig {
  repeated User clients = 1;
  Fallback fallback = 2;
}

message Fallback {
  uint32 dest = 1;  // پورت مقصد fallback (مثلاً 80)
}

message OutboundConfig {
  string address = 1;
  uint32 port = 2;
  string id = 3;  // UUID کلاینت
}
```

بعد باید این فایل رو compile کنید. در پوشه `xray-core`:

```bash
cd xray-core
protoc --go_out=. proxy/reflex/config.proto
```

این دستور فایل `proxy/reflex/config.pb.go` رو تولید می‌کنه که شامل struct‌های Go برای config هست.

**نکته مهم**: بعد از compile کردن proto، باید parser رو در `infra/conf` اضافه کنی تا Xray بتونه config JSON رو parse کنه. این کار معمولاً در فایل `infra/conf/serial/` انجام می‌شه. بذار ببینیم چطوری:

```go
// در infra/conf/serial/reflex.go (یا فایل مشابه)
package serial

import (
    "github.com/xtls/xray-core/infra/conf"
    "github.com/xtls/xray-core/infra/conf/serial"
    "github.com/xtls/xray-core/proxy/reflex"
)

func init() {
    conf.RegisterConfigCreator("reflex", func() interface{} {
        return new(conf.ReflexInboundConfig)
    })
}
```

**نکته**: در واقعیت، باید یک `ReflexInboundConfig` در `infra/conf` تعریف کنی که JSON رو parse می‌کنه و به `reflex.InboundConfig` تبدیل می‌کنه. این کار معمولاً در فایل‌های `infra/conf/proxy/` انجام می‌شه. برای جزئیات بیشتر، به کدهای VMess یا VLESS در `infra/conf/proxy/` نگاه کن.

## ساخت inbound handler اولیه

فایل `proxy/reflex/inbound/inbound.go` رو بسازید:

```go
package inbound

import (
    "context"
    "google.golang.org/protobuf/proto"
    "github.com/xtls/xray-core/common"
    "github.com/xtls/xray-core/common/net"
    "github.com/xtls/xray-core/common/protocol"
    "github.com/xtls/xray-core/proxy"
    "github.com/xtls/xray-core/proxy/reflex"
    "github.com/xtls/xray-core/transport/internet/stat"
    "github.com/xtls/xray-core/features/routing"
)

type Handler struct {
    clients []*protocol.MemoryUser
    fallback *FallbackConfig
}

// MemoryAccount برای ذخیره اطلاعات کاربر
// باید protocol.Account interface رو implement کنه
type MemoryAccount struct {
    Id string
}

// Equals implements protocol.Account
func (a *MemoryAccount) Equals(account protocol.Account) bool {
    reflexAccount, ok := account.(*MemoryAccount)
    if !ok {
        return false
    }
    return a.Id == reflexAccount.Id
}

// ToProto implements protocol.Account
func (a *MemoryAccount) ToProto() proto.Message {
    return &reflex.Account{
        Id: a.Id,
    }
}

type FallbackConfig struct {
    Dest uint32
}

func (h *Handler) Network() []net.Network {
    return []net.Network{net.Network_TCP}
}

func (h *Handler) Process(ctx context.Context, network net.Network, conn stat.Connection, dispatcher routing.Dispatcher) error {
    // اینجا بعداً منطق اصلی رو اضافه می‌کنیم
    return nil
}

func init() {
    common.Must(common.RegisterConfig((*reflex.InboundConfig)(nil), func(ctx context.Context, config interface{}) (interface{}, error) {
        return New(ctx, config.(*reflex.InboundConfig))
    }))
}

func New(ctx context.Context, config *reflex.InboundConfig) (proxy.InboundHandler, error) {
    handler := &Handler{
        clients: make([]*protocol.MemoryUser, 0),
    }
    
    // تبدیل config به handler
    for _, client := range config.Clients {
        handler.clients = append(handler.clients, &protocol.MemoryUser{
            Email: client.Id,
            Account: &MemoryAccount{Id: client.Id},
        })
    }
    
    // تنظیم fallback اگر وجود داشته باشه
    if config.Fallback != nil {
        handler.fallback = &FallbackConfig{
            Dest: config.Fallback.Dest,
        }
    }
    
    return handler, nil
}
```

این کد اولیه‌ست. هنوز کامل نیست، ولی ساختار رو نشون می‌ده.

## ثبت پروتکل در Xray

برای اینکه Xray پروتکل شما رو بشناسه، باید در فایل `main/dist/main.go` (یا فایل مشابه) این import رو اضافه کنید:

```go
import (
    // ... سایر import ها
    _ "github.com/xtls/xray-core/proxy/reflex/inbound"
    _ "github.com/xtls/xray-core/proxy/reflex/outbound"
)
```

## تست اولیه

حالا بیلد کنید:

```bash
cd xray-core
go build -o xray ./main
```

اگر خطای compile ندید، یعنی ساختار اولیه درسته.

## چک‌لیست

- [ ] پکیج reflex ساخته شده
- [ ] config.proto تعریف شده و compile شده
- [ ] inbound handler اولیه ساخته شده
- [ ] پروتکل در main.go ثبت شده
- [ ] بیلد بدون خطا انجام می‌شه

## بعدی

وقتی این مرحله رو تموم کردید، برید سراغ [Step 2](step2-handshake.md) برای پیاده‌سازی handshake.

