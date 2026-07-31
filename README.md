# Mock Server — Place to Pay

Mock server basado en WireMock para simular el flujo OTP de Place to Pay (P2P) en desarrollo local y Railway. Las credenciales de sandbox del PSP no permiten ejecutar este flujo, por lo que este mock cubre los endpoints necesarios.

## Requisitos

- Docker y Docker Compose

## Levantar en local

```bash
cd tools/mock-place-to-pay
docker compose up
```

El servidor queda disponible en `http://localhost:8090`.

Actualizar `mappings/payments/place_to_pay/config/config.json` en el bloque `sandbox`:

```json
"place_to_pay_base_url": "http://localhost:8090"
```

## Deploy en Railway

1. Crear un nuevo servicio en Railway apuntando al directorio `tools/mock-place-to-pay/`
2. Railway detecta el `Dockerfile` automáticamente
3. El puerto `8080` se expone en una URL pública (e.g. `https://mock-p2p.up.railway.app`)
4. Usar esa URL como `place_to_pay_base_url` en el entorno correspondiente

## Tarjetas de prueba

| Número de tarjeta    | Flujo                          | Resultado final          |
|----------------------|-------------------------------|--------------------------|
| `4111111111111111`   | Purchase directo (sin OTP)    | APPROVED                 |
| `4000000000000101`   | Purchase con OTP requerido    | APPROVED tras verify_otp |
| `4000000000000002`   | Purchase directo               | REJECTED                 |

Para el flujo OTP, usar código `123456` en el paso de validación.

## Endpoints mockeados

| Endpoint                        | Descripción                                      |
|---------------------------------|--------------------------------------------------|
| `POST /rest/gateway/information` | Info de tarjeta — retorna `requireOtp: true/false` según número |
| `POST /rest/gateway/otp/generate` | Genera OTP — siempre retorna OK                 |
| `POST /rest/gateway/otp/validate` | Valida OTP — OK con `123456`, error con cualquier otro |
| `POST /rest/gateway/process`     | Procesa pago — APPROVED por defecto, REJECTED para `4000000000000002` |
| `POST /rest/gateway/query`       | Consulta transacción — siempre APPROVED          |
| `POST /rest/gateway/transaction` | Captura/refund — siempre OK                     |

## Estructura de archivos

```
tools/mock-place-to-pay/
├── Dockerfile
├── docker-compose.yml
├── README.md
└── mappings/
    ├── information_otp_required.json   # priority 1 — tarjeta 4000000000000101
    ├── information_default.json        # priority 10 — cualquier otra tarjeta
    ├── otp_generate.json
    ├── otp_validate_success.json       # priority 1 — OTP 123456
    ├── otp_validate_invalid.json       # priority 10 — cualquier otro OTP
    ├── process_approved.json           # priority 10 — default APPROVED
    ├── process_rejected.json           # priority 1 — tarjeta 4000000000000002
    ├── query.json
    └── transaction.json
```

## Verificación

```bash
# Información con OTP requerido
curl -s -X POST http://localhost:8090/rest/gateway/information \
  -H "Content-Type: application/json" \
  -d '{"instrument":{"card":{"number":"4000000000000101"}}}' | jq .requireOtp
# Espera: true

# Información sin OTP
curl -s -X POST http://localhost:8090/rest/gateway/information \
  -H "Content-Type: application/json" \
  -d '{"instrument":{"card":{"number":"4111111111111111"}}}' | jq .requireOtp
# Espera: false

# Validar OTP correcto
curl -s -X POST http://localhost:8090/rest/gateway/otp/validate \
  -H "Content-Type: application/json" \
  -d '{"instrument":{"card":{"otp":"123456"}}}' | jq .validated
# Espera: true

# Validar OTP incorrecto
curl -s -X POST http://localhost:8090/rest/gateway/otp/validate \
  -H "Content-Type: application/json" \
  -d '{"instrument":{"card":{"otp":"999999"}}}' | jq .validated
# Espera: false
```
