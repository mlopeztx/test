# Rinde Gastos - Plan de Desarrollo

## Visión General

App web multi-tenant para gestión y rendición de gastos empresariales. Un **Cliente** (suscriptor) puede administrar múltiples **Empresas**, cada una con sus propias reglas, departamentos y empleados. Los usuarios pueden estar asociados a una o varias empresas del mismo cliente, pudiendo rendir y/o aprobar gastos en cada una.

Incluye:
- Registro de gastos con extracción automática de boletas por IA
- Integración con WhatsApp para subir boletas
- Flujo de aprobación configurable multinivel
- **Centralización contable**: una vez aprobada la rendición, se envía automáticamente al sistema contable (Django/Python) vía API para crear el voucher contable

---

## Stack Tecnológico

| Capa | Tecnología | Justificación |
|------|-----------|---------------|
| Frontend | Next.js 14 + TypeScript | App Router, SSR, API routes integradas |
| UI | Tailwind CSS + shadcn/ui | Componentes accesibles, responsive, rápidos de construir |
| Base de datos | PostgreSQL + Prisma ORM | Relacional, robusto, migraciones automáticas |
| Autenticación | NextAuth.js | Login con email/password y proveedores OAuth |
| Almacenamiento | AWS S3 | Para fotos de boletas |
| OCR / IA | AWS Textract (primario) + OpenAI GPT-4o Vision (fallback) | Textract tiene `AnalyzeExpense` especializado en boletas; GPT-4o como fallback para casos complejos |
| WhatsApp | Meta Business API (vía Twilio como wrapper) | Twilio simplifica la integración con Meta Business API |
| Contabilidad | API REST → Django/Python | Crear voucher contable al aprobar rendición |
| Hosting | AWS (producción) / Local (desarrollo) | EC2/ECS + RDS PostgreSQL |
| Dev local | Docker Compose | PostgreSQL + App local para desarrollo |

### Estrategia de procesamiento de boletas con IA

```
Boleta (imagen)
    │
    ▼
AWS Textract AnalyzeExpense ──→ ¿Extracción exitosa y confiable?
    │                                    │
    │ SÍ                                 │ NO (baja confianza)
    ▼                                    ▼
Datos estructurados              GPT-4o Vision (fallback)
(monto, vendor, fecha)          Prompt: "Extrae monto, proveedor,
    │                            fecha, categoría de esta boleta"
    ▼                                    │
Respuesta al usuario ◄──────────────────┘
```

**Alternativas evaluadas:**

| Alternativa | Costo aprox. | Veredicto |
|---|---|---|
| **AWS Textract** | ~$1.50/1000 págs | **Elegido** - Nativo AWS, `AnalyzeExpense` especializado |
| **OpenAI GPT-4o Vision** | ~$0.01-0.03/img | **Fallback** - Excelente comprensión contextual |
| **Claude Vision** | ~$0.01-0.03/img | Alternativa viable a GPT-4o |
| **Google Document AI** | ~$1.50/1000 págs | Descartado - No estamos en GCP |
| **Mindee** | Free 250/mes, luego $0.08/doc | Caro a escala |
| **Tesseract OCR** | Gratis | Muy impreciso para boletas chilenas |
| **PaddleOCR** | Gratis | Necesita GPU, complejo de mantener |

---

## Arquitectura Multi-Tenant

```
Client (Suscriptor)
├── name, plan, billingInfo
│
├── Company 1 (Empresa)
│   ├── name, rut, address, settings
│   ├── Departments[]
│   ├── ApprovalRules[] (propias de esta empresa)
│   ├── FuelConfig (propia)
│   ├── AccountingConfig (conexión API contabilidad)
│   └── Users[] (vía CompanyUser)
│
├── Company 2 (Empresa)
│   ├── ...misma estructura...
│   └── Users[] (pueden ser los mismos u otros)
│
└── Users[] (usuarios del cliente)
    └── CompanyUser[] (relación N:M con empresas)
        ├── companyId
        ├── userId
        └── role (employee | approver | admin)
```

**Reglas clave:**
- Un **User** pertenece a un **Client**
- Un **User** puede estar en 1 o más **Companies** del mismo Client
- El **rol** del usuario es por empresa (puede ser empleado en una y aprobador en otra)
- Cada **Company** tiene sus propias reglas de aprobación, categorías y configuración
- Los datos están aislados por Company (un usuario solo ve datos de las empresas donde está asignado)

---

## Módulos y Funcionalidades

### Fase 1 - MVP (Core)

#### 1.1 Autenticación y Gestión de Acceso
- Registro e inicio de sesión (email/password)
- Roles por empresa: **Empleado**, **Aprobador**, **Admin**
- Selector de empresa activa (para usuarios multi-empresa)
- Perfil de usuario (nombre, RUT, teléfono)
- Admin del cliente puede:
  - Crear/editar empresas
  - Invitar usuarios y asignarlos a empresas
  - Definir roles por empresa

#### 1.2 Gestión de Gastos
- Crear gasto manualmente (monto, proveedor, fecha, categoría, descripción)
- Moneda principal: **CLP** (soporte futuro para USD, UF)
- Subir foto de boleta desde el navegador
- **Extracción automática con IA** (Textract + GPT-4o fallback):
  - Monto total
  - Nombre del proveedor/comercio
  - Fecha de la boleta
  - Categoría sugerida
  - RUT del proveedor (si visible)
- Editar y eliminar gastos en estado borrador
- Categorías predefinidas (configurables por empresa):
  Alimentación, Transporte, Alojamiento, Oficina, Combustible, Representación, Otros
- Estados del gasto: **Borrador → Enviado → Aprobado / Rechazado**
- Cada gasto pertenece a una empresa específica

#### 1.3 Rendiciones de Gastos
- Agrupar múltiples gastos en una rendición
- Asignar nombre/descripción a la rendición
- Enviar rendición para aprobación
- Ver historial de rendiciones con estados
- Descargar rendición en PDF

#### 1.4 Flujo de Aprobación (básico)
- Aprobador ve rendiciones pendientes de su empresa
- Aprobar o rechazar con comentarios
- Notificación al empleado del resultado
- Vista de historial de aprobaciones

#### 1.5 Dashboard
- **Empleado**: mis gastos del mes, total gastado, rendiciones pendientes
- **Aprobador/Admin**:
  - Total aprobado, cantidad de gastos, miembros del equipo
  - Gráfico de tendencia últimos 6 meses
  - Distribución por estado (borrador, enviado, aprobado, rechazado)
  - Desglose por departamento y categoría
  - Top categorías de gasto
- Filtro por empresa (para usuarios multi-empresa)

---

### Fase 2 - Funcionalidades Avanzadas

#### 2.1 Integración Contable (Django/Python)
- Configuración por empresa:
  - URL base de la API contable
  - Credenciales de autenticación (API key / token)
  - Mapeo de categorías de gasto → cuentas contables
  - Centro de costo por departamento
- Al aprobar una rendición:
  1. Se arma el payload del voucher contable con los gastos detallados
  2. Se llama a la API Django `POST /api/vouchers/` con:
     - Fecha del voucher
     - Glosa (descripción de la rendición)
     - Líneas de detalle (cuenta, debe/haber, centro de costo)
     - Referencia a la rendición
     - Documentos adjuntos (URLs de boletas)
  3. Se registra el ID del voucher contable en la rendición
  4. Estado de la rendición: **Aprobado → Contabilizado**
- Retry automático si la API contable falla
- Log de sincronización (éxito/error por rendición)
- Vista admin: rendiciones pendientes de contabilizar

#### 2.2 Bot de WhatsApp
- Empleado envía foto de boleta al número de WhatsApp del sistema
- Bot identifica al usuario por número de teléfono vinculado
- Si el usuario tiene múltiples empresas, pregunta a cuál empresa asociar el gasto
- Bot responde con datos extraídos por IA:
  - "He detectado: $15.230 en Supermercado Líder, 24/02/2025, Categoría: Alimentación"
- Usuario puede corregir datos conversacionalmente:
  - "El monto es $15.330" → Bot actualiza
  - "Cambiar categoría a Oficina" → Bot actualiza
- Confirmar para registrar el gasto automáticamente
- Vincular número de WhatsApp con cuenta del usuario desde la app

#### 2.3 Reglas de Aprobación Configurables (por empresa)
- Crear reglas por:
  - **Rango de monto** (ej: > $100.000 requiere gerente)
  - **Categoría** (ej: Viajes siempre requiere aprobación extra)
  - **Departamento** (ej: Marketing tiene aprobador diferente)
- Aprobación multinivel (paso 1: jefe directo, paso 2: gerente)
- Prioridad entre reglas
- Asignar aprobador específico o por rol
- Cada empresa configura sus propias reglas

#### 2.4 Módulo de Combustible (por empresa)
- Configurar precio por litro de combustible (manual o automático desde CNE)
- Configurar eficiencia del vehículo (km/L)
- Tolerancia de desviación (%)
- Cálculo automático de tarifa efectiva por km
- Gastos de combustible con validación automática

#### 2.5 Reportes y Exportación
- Reportes por período, departamento, categoría, empresa
- Exportar a Excel/CSV
- Exportar rendiciones a PDF con detalle y fotos

---

### Fase 3 - Mejoras

#### 3.1 Notificaciones
- Email al enviar/aprobar/rechazar rendiciones
- Notificaciones push (PWA)
- Recordatorios de rendiciones pendientes

#### 3.2 App Móvil (PWA)
- Progressive Web App para experiencia nativa en móvil
- Cámara directa para capturar boletas
- Modo offline (guardar borradores sin conexión)

#### 3.3 Admin del Cliente
- Dashboard consolidado de todas las empresas del cliente
- Gestión de suscripción y facturación
- Reportes cross-empresa

---

## Modelo de Datos

```
Client (Suscriptor/Tenant)
├── id, name, plan, billingEmail, isActive
├── createdAt, updatedAt
│
├──→ Company[] (Empresas del cliente)
│    ├── id, clientId, name, rut, address, logo
│    ├── currency (default: CLP)
│    ├── isActive, createdAt, updatedAt
│    │
│    ├──→ Department[]
│    │    ├── id, companyId, name
│    │    └── managerId (FK → User)
│    │
│    ├──→ Category[] (categorías configurables)
│    │    ├── id, companyId, name, icon, isActive
│    │    └── accountingCode (código cuenta contable)
│    │
│    ├──→ ApprovalRule[]
│    │    ├── id, companyId, name
│    │    ├── minAmount, maxAmount
│    │    ├── categoryId, departmentId
│    │    ├── approverRole, approverId
│    │    ├── step, priority, isActive
│    │    └── createdAt, updatedAt
│    │
│    ├──→ FuelConfig
│    │    ├── id, companyId
│    │    ├── pricePerLiter, efficiency, tolerance
│    │    └── autoFetch, effectiveRate
│    │
│    └──→ AccountingConfig
│         ├── id, companyId
│         ├── apiBaseUrl, apiToken
│         ├── isActive
│         └── departmentCostCenterMap (JSON)
│
├──→ User[] (Usuarios del cliente)
│    ├── id, clientId, email, passwordHash
│    ├── name, rut, phone, whatsappPhone
│    ├── isActive, createdAt, updatedAt
│    │
│    └──→ CompanyUser[] (relación N:M con empresas)
│         ├── id, userId, companyId
│         ├── role (employee | approver | admin)
│         ├── departmentId
│         └── isActive
│
├──→ Expense[] (Gastos individuales)
│    ├── id, userId, companyId
│    ├── amount, currency (CLP)
│    ├── vendor, vendorRut, date
│    ├── categoryId, description
│    ├── receiptUrl (S3 key)
│    ├── status (draft | submitted | approved | rejected)
│    ├── aiExtractedData (JSON - datos crudos de Textract/GPT-4o)
│    ├── aiProvider (textract | openai)
│    ├── reportId (FK → ExpenseReport, nullable)
│    ├── source (web | whatsapp)
│    └── createdAt, updatedAt
│
├──→ ExpenseReport[] (Rendiciones)
│    ├── id, userId, companyId
│    ├── title, description
│    ├── status (draft | submitted | approved | rejected | accounted)
│    ├── totalAmount
│    ├── submittedAt, approvedAt, accountedAt
│    ├── currentApprovalStep
│    │
│    ├──→ Expense[] (gastos de esta rendición)
│    │
│    ├──→ ApprovalHistory[]
│    │    ├── id, reportId, approverId
│    │    ├── step, action (approved | rejected)
│    │    ├── comments
│    │    └── createdAt
│    │
│    └──→ AccountingSync
│         ├── id, reportId
│         ├── voucherId (ID del voucher en Django)
│         ├── status (pending | synced | error)
│         ├── payload (JSON enviado)
│         ├── response (JSON recibido)
│         ├── errorMessage
│         └── syncedAt, createdAt
│
└──→ WhatsAppSession[] (sesiones de conversación)
     ├── id, userId, companyId
     ├── phoneNumber, currentExpenseId
     ├── conversationState (awaiting_photo | confirming | selecting_company)
     ├── expiresAt
     └── createdAt
```

---

## Flujo de la Rendición (end-to-end)

```
Empleado                    Sistema                     Aprobador(es)           Contabilidad (Django)
   │                          │                              │                        │
   ├─ Sube boleta ──────────→ │                              │                        │
   │  (web o WhatsApp)        ├─ Textract/GPT-4o extrae     │                        │
   │                          │  datos de la boleta          │                        │
   │ ◄── Datos extraídos ────┤                              │                        │
   │                          │                              │                        │
   ├─ Confirma/corrige ─────→ │                              │                        │
   │                          ├─ Crea gasto (borrador)      │                        │
   │                          │                              │                        │
   ├─ Crea rendición ───────→ │                              │                        │
   │  (agrupa gastos)         │                              │                        │
   │                          │                              │                        │
   ├─ Envía rendición ──────→ │                              │                        │
   │                          ├─ Evalúa reglas aprobación   │                        │
   │                          ├─ Notifica aprobador ───────→ │                        │
   │                          │                              │                        │
   │                          │         Aprueba/Rechaza ◄───┤                        │
   │                          │ ◄──────────────────────────┤                        │
   │                          │                              │                        │
   │                          ├─ Si aprobada:                │                        │
   │                          │  POST /api/vouchers/ ────────────────────────────────→│
   │                          │                              │                        │
   │                          │ ◄─── voucherId ─────────────────────────────────────┤
   │                          ├─ Estado: "Contabilizado"     │                        │
   │ ◄── Notificación ──────┤                              │                        │
   │                          │                              │                        │
```

---

## Estructura del Proyecto

```
rinde-gastos/
├── docker-compose.yml          # PostgreSQL + pgAdmin para dev local
├── prisma/
│   ├── schema.prisma           # Modelo de datos completo
│   ├── seed.ts                 # Datos de prueba
│   └── migrations/
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   │   ├── login/
│   │   │   └── register/
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx          # Layout con sidebar + selector empresa
│   │   │   ├── dashboard/          # Dashboard principal
│   │   │   ├── expenses/           # CRUD de gastos
│   │   │   │   ├── page.tsx        # Lista de gastos
│   │   │   │   ├── new/            # Nuevo gasto
│   │   │   │   └── [id]/           # Detalle/editar gasto
│   │   │   ├── reports/            # Rendiciones
│   │   │   │   ├── page.tsx        # Lista de rendiciones
│   │   │   │   ├── new/            # Nueva rendición
│   │   │   │   └── [id]/           # Detalle rendición
│   │   │   ├── approvals/          # Aprobaciones (rol aprobador)
│   │   │   ├── team/               # Gestión de equipo (admin)
│   │   │   ├── companies/          # Gestión de empresas (admin cliente)
│   │   │   └── settings/           # Configuración empresa
│   │   │       ├── categories/
│   │   │       ├── approval-rules/
│   │   │       ├── fuel/
│   │   │       └── accounting/     # Config integración contable
│   │   ├── api/
│   │   │   ├── auth/[...nextauth]/
│   │   │   ├── expenses/
│   │   │   ├── reports/
│   │   │   ├── reports/[id]/approve/
│   │   │   ├── upload/             # Upload de boletas a S3
│   │   │   ├── ai/extract/         # Extracción IA (Textract + fallback)
│   │   │   ├── accounting/sync/    # Sincronización con Django
│   │   │   └── whatsapp/webhook/   # Webhook WhatsApp
│   │   ├── layout.tsx
│   │   └── page.tsx                # Landing / redirect
│   ├── components/
│   │   ├── ui/                     # shadcn components
│   │   ├── dashboard/              # Gráficos, stats cards
│   │   ├── expenses/               # Formularios, listas, receipt viewer
│   │   ├── reports/                # Tabla rendiciones, PDF preview
│   │   └── layout/                 # Sidebar, navbar, company switcher
│   ├── lib/
│   │   ├── prisma.ts               # Cliente Prisma singleton
│   │   ├── auth.ts                 # Configuración NextAuth
│   │   ├── ai/
│   │   │   ├── textract.ts         # Cliente AWS Textract
│   │   │   ├── openai-vision.ts    # Cliente GPT-4o Vision (fallback)
│   │   │   └── extract.ts          # Orquestador: Textract → fallback
│   │   ├── s3.ts                   # Upload/download S3
│   │   ├── whatsapp.ts             # Cliente WhatsApp (Twilio)
│   │   ├── accounting.ts           # Cliente API contable Django
│   │   └── utils.ts                # Formateo CLP, fechas, etc.
│   ├── types/
│   │   └── index.ts
│   └── middleware.ts               # Auth + tenant middleware
├── public/
├── .env.local                      # Variables de entorno
├── package.json
├── tailwind.config.ts
├── tsconfig.json
└── next.config.js
```

---

## Plan de Ejecución

### Fase 1 - MVP (~4-6 sprints)

| # | Tarea | Detalle |
|---|-------|---------|
| 1 | Setup proyecto | Next.js 14, Prisma, Tailwind, shadcn/ui, Docker Compose (PostgreSQL) |
| 2 | Modelo de datos | Schema Prisma multi-tenant (Client → Company → User), migraciones, seed |
| 3 | Auth + Multi-tenant | NextAuth, login/registro, selector de empresa, middleware tenant |
| 4 | CRUD Gastos | Crear, editar, eliminar, listar gastos por empresa |
| 5 | Upload boletas | Subir fotos a S3 (local: filesystem), preview de imagen |
| 6 | Extracción IA | AWS Textract AnalyzeExpense + GPT-4o fallback |
| 7 | Rendiciones | Crear rendición, agrupar gastos, enviar, generar PDF |
| 8 | Aprobaciones básicas | Flujo aprobar/rechazar, comentarios, historial |
| 9 | Dashboard | Métricas, gráficos (Recharts), filtros por empresa |

### Fase 2 - Avanzado (~4-5 sprints)

| # | Tarea | Detalle |
|---|-------|---------|
| 10 | Integración contable | Config por empresa, sync con API Django, estado "Contabilizado" |
| 11 | Bot WhatsApp | Webhook, flujo conversacional, extracción IA, multi-empresa |
| 12 | Reglas aprobación | Config multinivel por empresa, evaluación automática |
| 13 | Combustible | Config precios, eficiencia, validación automática |
| 14 | Reportes | Exportar Excel/CSV, PDF detallado |

### Fase 3 - Mejoras (~2-3 sprints)

| # | Tarea | Detalle |
|---|-------|---------|
| 15 | Notificaciones | Email (SES), push (PWA) |
| 16 | PWA | Service worker, offline, cámara |
| 17 | Admin cliente | Dashboard cross-empresa, billing |

---

## Configuración de Desarrollo Local

```bash
# 1. Clonar y entrar al proyecto
git clone <repo> && cd rinde-gastos

# 2. Instalar dependencias
npm install

# 3. Levantar PostgreSQL con Docker
docker-compose up -d

# 4. Configurar variables de entorno
cp .env.example .env.local
# Editar: DATABASE_URL, NEXTAUTH_SECRET, AWS keys, etc.

# 5. Ejecutar migraciones
npx prisma migrate dev

# 6. Seed de datos de prueba
npx prisma db seed

# 7. Iniciar app
npm run dev
# → http://localhost:3000
```

---

## API Contable (Django) - Contrato Esperado

La app llamará al sistema Django con el siguiente contrato al aprobar una rendición:

```json
POST {apiBaseUrl}/api/vouchers/
Authorization: Token {apiToken}
Content-Type: application/json

{
  "date": "2025-02-24",
  "description": "Rendición #123 - Juan Pérez - Gastos Febrero",
  "reference": "RINDE-123",
  "lines": [
    {
      "account_code": "6101001",
      "description": "Almuerzo cliente - Restaurant XYZ",
      "debit": 15230,
      "credit": 0,
      "cost_center": "CC-VENTAS"
    },
    {
      "account_code": "6102003",
      "description": "Taxi aeropuerto",
      "debit": 8500,
      "credit": 0,
      "cost_center": "CC-VENTAS"
    },
    {
      "account_code": "1101001",
      "description": "Contrapartida rendición",
      "debit": 0,
      "credit": 23730,
      "cost_center": null
    }
  ],
  "attachments": [
    "https://s3.amazonaws.com/bucket/receipts/receipt-001.jpg",
    "https://s3.amazonaws.com/bucket/receipts/receipt-002.jpg"
  ],
  "metadata": {
    "source": "rinde-gastos",
    "report_id": 123,
    "employee_rut": "12.345.678-9",
    "company_rut": "76.543.210-K"
  }
}

// Respuesta esperada:
{
  "id": 4567,
  "voucher_number": "V-2025-0234",
  "status": "created",
  "created_at": "2025-02-24T15:30:00Z"
}
```

> **Nota:** Este contrato se ajustará según la API real del sistema Django existente.
