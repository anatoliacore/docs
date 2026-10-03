# AnatoliaCore — SDK & Terraform Kullanım Kılavuzu

Bu belge, AnatoliaCore Cloud'u **programatik** olarak nasıl kullanacağını anlatır:
Public REST API, Python SDK, TypeScript SDK ve Terraform provider. Hepsi aynı
temel üzerine oturur: **Public API + müşteri API anahtarı**.

---

## 1. Temel kavramlar

| Konu | Değer |
|---|---|
| Public API tabanı | `https://console.anatoliacore.com/api/public/v1` |
| Kimlik doğrulama | `X-API-Key: ac_live_...` başlığı |
| Yanıt biçimi | JSON |
| Mutasyonlar | Asenkron; `operation_id` döner, durum ayrıca sorgulanır |
| Idempotency | Her mutasyonda `Idempotency-Key` başlığı (SDK/TF otomatik üretir) |

Tüm istemciler (SDK'lar, Terraform) bu API'yi çağırır. Yani önce bir **API
anahtarı** üretmen gerekir.

---

## 2. API anahtarı oluşturma

1. Panelde **Developer & Account → API Keys** sayfasına gir.
2. **Yeni anahtar** oluştur; bir **isim** ve **scope** (yetki) listesi seç.
3. Anahtar (`ac_live_...`) **yalnızca bir kez** gösterilir — hemen güvenli bir
   yere kopyala. Panel sadece ilk 8 karakterlik ön eki saklar.

### Scope'lar (yetkiler)
Okuma/yazma ayrı verilir. Kaynak başına:
`instances`, `vdcs`, `volumes`, `floating_ips`, `security_groups`,
`backup_policies`, `ssh_keys`, `alarms`, `webhooks` → `:read` ve/veya `:write`;
ayrıca salt-okuma: `billing:read`, `catalog:read`, `operations:read`,
`snapshots:read`, `images:read`, `applications:read`, `platform:read`.

### ⚠️ Yazma (`*:write`) anahtarları için kaynak IP zorunlu
Herhangi bir `*:write` scope içeren anahtar (instance/vdc/volume/floating_ip/
security_group/backup_policy/ssh_keys/alarms/webhooks yazma) **`allowed_cidrs`
olmadan üretilemez** ve kullanılamaz. Anahtarı oluştururken, isteklerin
geleceği kaynak IP/aralığını CIDR olarak gir (ör. `203.0.113.10/32` veya ofis
bloğun). CIDR dışından gelen istek reddedilir. Salt-okuma anahtarları CIDR
gerektirmez.

> Bu, çalınan bir yazma anahtarının rastgele bir yerden kullanılmasını önleyen
> kasıtlı bir güvenlik kontrolüdür. Tüm anahtarlarını taze MFA ile tek seferde
> iptal edebilirsin.

---

## 3. Public REST API (doğrudan)

```bash
export AC="https://console.anatoliacore.com/api/public/v1"
export KEY="ac_live_..."

# Instance listele (okuma)
curl -s "$AC/instances" -H "X-API-Key: $KEY"

# Instance oluştur (yazma — anahtar CIDR-bağlı olmalı)
curl -s -X POST "$AC/instances" \
  -H "X-API-Key: $KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{"name":"web-01","cpu_cores":2,"ram_mb":4096,"disk_gb":40,"os_template":"ubuntu-24-04","vdc_id":"<vdc-uuid>"}'
```

Başlıca uç noktalar (hepsi `/api/public/v1` altında):
- **instances**: `GET /instances`, `GET /instances/{id}`, `POST /instances`,
  `POST /instances/{id}/resize`, `POST /instances/{id}/power`,
  `DELETE /instances/{id}`, `POST /instances/price-estimate`
- **vdcs**: `GET /vdcs`, `POST /vdcs`
- **volumes**: `GET/POST /volumes`, `.../attach`, `.../detach`, `.../resize`
- **floating-ips**: `GET/POST /floating-ips`, `.../associate`, `.../disassociate`
- **security-groups**: `GET/POST /security-groups` (+ kural ekle/sil)
- **backup-policies**: `GET/POST/PUT/DELETE /backup-policies`
- **ssh-keys**: `GET/POST/DELETE /ssh-keys`
- **catalog**: `GET /instance-types`, `GET /applications`
- **billing**: `GET /billing/cost-analysis`
- **platform**: `GET /platform/status`
- **operations**: `GET /operations`, `GET /operations/{id}`, `POST /operations/{id}/cancel`
- **webhooks**: `GET/POST/DELETE /webhooks`, `.../rotate-secret`, `webhook-events`, `webhook-deliveries`

### Asenkron operasyon modeli
Mutasyonlar (create/resize/delete...) işi **hemen bitirmez**; kalıcı bir
`operation_id` döner. Sonucu `GET /operations/{id}` ile poll et
(`status`: pending → running → succeeded/failed). SDK'larda `wait_operation`
/ `waitOperation` bu döngüyü senin için yapar.

---

## 4. Terraform Provider

Provider; instance, VDC, volume, floating IP, edge security group ve backup
policy kaynaklarını yönetir; instance tipi kataloğunu bir data source olarak sunar.

### Kurulum ve kimlik
```hcl
terraform {
  required_providers {
    anatoliacore = {
      source  = "anatoliacore/anatoliacore"
      version = "~> 1.0"
    }
  }
}

provider "anatoliacore" {
  # api_key   = var.ac_api_key   # ÖNERİLMEZ — env kullan
  # base_url  = "https://console.anatoliacore.com/api/public/v1"  # opsiyonel
}
```

Kimliği **ortam değişkeninden** ver (API anahtarını `.tf` dosyasına yazma):
```bash
export ANATOLIACORE_API_KEY='ac_live_...'
# İsteğe bağlı farklı ortam:
export ANATOLIACORE_BASE_URL='https://console.anatoliacore.com/api/public/v1'
```
> Terraform bir **yazma** anahtarı kullanır → anahtar CIDR-bağlı olmalı; plan/apply
> çalıştırdığın makinenin/CI runner'ının IP'si o CIDR içinde olmalı.

### Örnek yapılandırma
```hcl
data "anatoliacore_instance_types" "all" {}

resource "anatoliacore_vdc" "prod" {
  name          = "prod-vdc"
  cpu_limit_mhz = 16000
  ram_limit_mb  = 65536
  disk_limit_gb = 1000
}

resource "anatoliacore_instance" "web" {
  name        = "web-01"
  cpu_cores   = 2
  ram_mb      = 4096
  disk_gb     = 40
  os_template = "ubuntu-24-04"
  vdc_id      = anatoliacore_vdc.prod.id
  # ssh_key_id = "..."      # opsiyonel
  # user_data  = "#cloud-config\n..."  # opsiyonel, sensitive, değişince replace
}

resource "anatoliacore_volume" "data" {
  name        = "web-data"
  size_gb     = 100
  instance_id = anatoliacore_instance.web.id
}

output "web_ip" {
  value = anatoliacore_instance.web.ip_address
}
```

### İş akışı
```bash
terraform init
terraform plan
terraform apply
terraform destroy
```

### Notlar
- `anatoliacore_instance` alanları: `name`, `cpu_cores`, `ram_mb`, `disk_gb`,
  `os_template` (zorunlu); `vdc_id`, `ssh_key_id`, `user_data` (opsiyonel);
  `id`, `ip_address`, `status` (computed). `vdc_id / ssh_key_id / user_data`
  değişimi kaynağı **yeniden oluşturur** (requires-replace).
- Her mutasyon dahili benzersiz **idempotency key** kullanır; provider,
  operation kontratını takip eder ve **redirect'leri hata sayar** (kimlik başka
  bir host'a sızmasın diye).
- Mevcut bir kaynağı içeri almak: `terraform import anatoliacore_instance.web <instance-uuid>`.
- Kaynak oluşturma asenkron olduğundan apply, operasyon tamamlanana kadar bekler.

---

## 5. Python SDK

```bash
pip install anatoliacore        # ilk PyPI sürümüne kadar: pip install ./sdk/python
export ANATOLIACORE_API_KEY='ac_live_...'
```

```python
from anatoliacore import AnatoliaCore

cloud = AnatoliaCore(api_key="ac_live_...")   # veya env'den otomatik

# Okuma
types = cloud.list_instance_types()
vms   = cloud.list_instances(page=1, per_page=20, status="running")
vm    = cloud.get_instance("<id>")

# Yazma (asenkron) + operasyon takibi
result = cloud.request_with_metadata(
    "POST", "/instances",
    body={"name": "web", "cpu_cores": 2, "ram_mb": 4096, "disk_gb": 40,
          "os_template": "ubuntu-24-04", "vdc_id": "<vdc-uuid>"},
)
op = cloud.wait_operation(result.operation_id) if result.operation_id else None

# Kısayol metodları
cloud.create_instance({...}, idempotency_key="deploy-web-2026-08")
cloud.power_instance("<id>", "stop")     # start/stop/restart
cloud.delete_instance("<id>")
cloud.create_vdc({...})
cloud.list_vdcs()
cloud.get_operation("<op-id>")
cloud.cancel_operation("<op-id>")
```

- Mutasyon metodları, sen vermezsen rastgele bir **idempotency key** üretir;
  aynı mantıksal işlemi yeniden denerken **aynı** anahtarı ver.
- Uzun süren işler için `wait_operation` durable operasyonu poll eder.
- **CLI** de gelir: `anatoliacore instances list` vb.
- **Webhook doğrulama**: olayı işlemeden önce `cloud.verify_webhook(...)` çağır —
  HMAC, zaman toleransı, event ID bağlama ve boyut sınırı uygular.

---

## 6. TypeScript SDK

```bash
npm install @anatoliacore/sdk    # ilk npm sürümüne kadar: npm install ./sdk/typescript
```

```ts
import { AnatoliaCore } from "@anatoliacore/sdk"

const cloud = new AnatoliaCore({ apiKey: process.env.ANATOLIACORE_API_KEY! })

const instances = await cloud.listInstances()

const { operationId } = await cloud.requestWithMetadata("POST", "/instances", {
  body: { name: "web", cpu_cores: 2, ram_mb: 4096, disk_gb: 40,
          os_template: "ubuntu-24-04", vdc_id: "<vdc-uuid>" },
})
if (operationId) await cloud.waitOperation(operationId)
```

- **Yalnızca güvenilir sunucu tarafında** kullan; API anahtarını tarayıcı/mobil
  paketine gömme.
- `requestWithMetadata()` durable `operationId` döner; `waitOperation()`'a ver.
- Webhook'ta orijinal istek byte'ları üzerinde `verifyWebhook()` çağır.

---

## 7. Webhook'lar (olay bildirimleri)

`POST /webhooks` ile bir endpoint kaydet; imza sırrını al (`rotate-secret` ile
döndürülebilir). Gelen her olayı işlemeden önce SDK'nın `verify_webhook` /
`verifyWebhook` yardımcısıyla doğrula. Teslimatlar durable outbox'tan yeniden
denenir; `webhook-deliveries` ile durum görülür, `.../retry` ile elle yeniden
denenir. Ayrıntı: [`docs/WEBHOOKS.md`](WEBHOOKS.md).

---

## 8. Sık karşılaşılan sorunlar

| Belirti | Sebep / Çözüm |
|---|---|
| `401/403` yazma isteğinde | Anahtarın `*:write` scope'u yok **veya** kaynak IP `allowed_cidrs` dışında. |
| Yazma anahtarı oluşturulamıyor | `*:write` scope için `allowed_cidrs` zorunlu — CIDR ekle. |
| Terraform apply IP'den reddediliyor | Runner IP'sini anahtarın CIDR'ına ekle. |
| Mutasyon "bitmiyor" | Asenkron; `operation_id`'yi poll et / `wait_operation` kullan. |
| Aynı işlem iki kez çalıştı | Retry'da **aynı** `Idempotency-Key`'i kullan. |
| Anahtar kayıp | Sır yalnız oluşturmada gösterilir; yeni anahtar üret, eskisini iptal et. |

İlgili belgeler: [`docs/PUBLIC_API.md`](PUBLIC_API.md), [`docs/WEBHOOKS.md`](WEBHOOKS.md).
