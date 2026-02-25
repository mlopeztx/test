# Rinde Gastos - Plan de Desarrollo

## Visión General

App web para gestión y rendición de gastos empresariales. Permite a los empleados registrar gastos (manualmente o vía foto de boleta), generar rendiciones, y a los supervisores aprobar/rechazar. Incluye integración con WhatsApp para subir boletas directamente desde el celular.

---

## Stack Tecnológico Propuesto

| Capa | Tecnología | Justificación |
|------|-----------|---------------|
| Frontend | Next.js 14 + TypeScript | App Router, SSR, API routes integradas |
| UI | Tailwind CSS + shadcn/ui | Componentes accesibles, responsive, rápidos de construir |
| Base de datos | PostgreSQL + Prisma ORM | Relacional, robusto, migraciones automáticas |
| Autenticación | NextAuth.js | Login con email/password y proveedores OAuth |
| Almacenamiento | AWS S3 / Cloudflare R2 | Para fotos de boletas |
| OCR / IA | OpenAI Vision API | Extracción de datos desde fotos de boletas |
| WhatsApp | Twilio WhatsApp API | Bot para recibir boletas por WhatsApp |
| Deploy | Vercel | Deploy automático, serverless |

---

## Módulos y Funcionalidades

### Fase 1 - MVP (Core)

#### 1.1 Autenticación y Usuarios
- Registro e inicio de sesión (email/password)
- Roles: **Empleado**, **Aprobador**, **Admin**
- Perfil de usuario (nombre, departamento, RUT)
- Invitación de usuarios por email

#### 1.2 Gestión de Gastos
- Crear gasto manualmente (monto, proveedor, fecha, categoría, descripción)
- Subir foto de boleta desde el navegador
- **Extracción automática con IA**: al subir una foto, la IA extrae:
  - Monto total
  - Nombre del proveedor/comercio
  - Fecha
  - Categoría sugerida
  - Moneda (CLP, USD, UF)
- Editar y eliminar gastos en estado borrador
- Categorías predefinidas: Alimentación, Transporte, Alojamiento, Oficina, Combustible, Otros
- Estados del gasto: **Borrador → Enviado → Aprobado / Rechazado**

#### 1.3 Rendiciones de Gastos
- Agrupar múltiples gastos en una rendición
- Asignar nombre/descripción a la rendición
- Enviar rendición para aprobación
- Ver historial de rendiciones con estados
- Descargar rendición en PDF

#### 1.4 Flujo de Aprobación
- Aprobador ve rendiciones pendientes de su equipo
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

---

### Fase 2 - Funcionalidades Avanzadas

#### 2.1 Bot de WhatsApp
- Empleado envía foto de boleta al número de WhatsApp del sistema
- Bot responde con datos extraídos por IA:
  - "He detectado: $15.230 en Supermercado Líder, 24/02/2025, Categoría: Alimentación"
- Usuario puede corregir datos conversacionalmente:
  - "El monto es $15.330" → Bot actualiza
  - "Cambiar categoría a Oficina" → Bot actualiza
- Confirmar para registrar el gasto automáticamente
- Vincular número de WhatsApp con cuenta del usuario

#### 2.2 Reglas de Aprobación Configurables
- Crear reglas por:
  - **Rango de monto** (ej: > $100.000 requiere gerente)
  - **Categoría** (ej: Viajes siempre requiere aprobación extra)
  - **Departamento** (ej: Marketing tiene aprobador diferente)
- Aprobación multinivel (paso 1: jefe directo, paso 2: gerente)
- Prioridad entre reglas
- Asignar aprobador específico o por rol

#### 2.3 Módulo de Combustible
- Configurar precio por litro de combustible (manual o automático desde CNE)
- Configurar eficiencia del vehículo (km/L)
- Tolerancia de desviación (%)
- Cálculo automático de tarifa efectiva por km
- Gastos de combustible con validación automática

#### 2.4 Reportes y Exportación
- Reportes por período, departamento, categoría
- Exportar a Excel/CSV
- Exportar rendiciones a PDF con detalle y fotos

---

### Fase 3 - Mejoras

#### 3.1 Notificaciones
- Email al enviar/aprobar/rechazar rendiciones
- Notificaciones push (PWA)
- Recordatorios de rendiciones pendientes

#### 3.2 Integraciones
- Integración contable (exportar a formato compatible con sistemas contables)
- API REST pública para integraciones externas

#### 3.3 App Móvil (PWA)
- Progressive Web App para experiencia nativa en móvil
- Cámara directa para capturar boletas
- Modo offline (guardar borradores sin conexión)

---

## Modelo de Datos (Principales Entidades)

```
User
├── id, email, password, name, role, department, phone, whatsappLinked
│
├── Expense (gastos individuales)
│   ├── id, amount, currency, vendor, date, category, description
│   ├── receiptUrl (foto de boleta)
│   ├── status (draft, submitted, approved, rejected)
│   ├── aiExtracted (datos extraídos por IA)
│   └── reportId (FK a ExpenseReport)
│
├── ExpenseReport (rendiciones)
│   ├── id, title, description, status, totalAmount
│   ├── submittedAt, approvedAt
│   └── approverId (FK a User)
│
├── ApprovalRule (reglas de aprobación)
│   ├── id, name, minAmount, maxAmount
│   ├── category, department
│   ├── approverRole, approverId
│   ├── step, priority
│   └── isActive
│
├── FuelConfig (configuración combustible)
│   ├── id, pricePerLiter, efficiency, tolerance
│   └── autoFetch, effectiveRate
│
└── Department
    ├── id, name
    └── managerId (FK a User)
```

---

## Estructura del Proyecto

```
rinde-gastos/
├── prisma/
│   └── schema.prisma          # Modelo de datos
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   │   ├── login/
│   │   │   └── register/
│   │   ├── (dashboard)/
│   │   │   ├── dashboard/      # Dashboard principal
│   │   │   ├── expenses/       # CRUD de gastos
│   │   │   ├── reports/        # Rendiciones
│   │   │   ├── approvals/      # Aprobaciones
│   │   │   ├── settings/       # Configuración
│   │   │   └── team/           # Gestión de equipo
│   │   ├── api/
│   │   │   ├── auth/
│   │   │   ├── expenses/
│   │   │   ├── reports/
│   │   │   ├── upload/         # Upload de boletas
│   │   │   ├── ai/extract/     # Extracción IA
│   │   │   └── whatsapp/       # Webhook WhatsApp
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── components/
│   │   ├── ui/                 # shadcn components
│   │   ├── dashboard/          # Gráficos, stats
│   │   ├── expenses/           # Formularios, listas
│   │   └── layout/             # Sidebar, navbar
│   ├── lib/
│   │   ├── prisma.ts
│   │   ├── auth.ts
│   │   ├── ai.ts               # Lógica de extracción IA
│   │   ├── whatsapp.ts         # Cliente WhatsApp
│   │   └── utils.ts
│   └── types/
├── public/
├── package.json
├── tailwind.config.ts
└── tsconfig.json
```

---

## Plan de Ejecución

| Orden | Tarea | Descripción |
|-------|-------|-------------|
| 1 | Setup proyecto | Next.js, Prisma, Tailwind, shadcn/ui |
| 2 | Auth + Usuarios | Login, registro, roles, NextAuth |
| 3 | Modelo de datos | Schema Prisma, migraciones |
| 4 | CRUD Gastos | Crear, editar, eliminar gastos |
| 5 | Upload boletas | Subir fotos, almacenamiento |
| 6 | Extracción IA | Integrar OpenAI Vision para OCR |
| 7 | Rendiciones | Agrupar gastos, enviar, PDF |
| 8 | Aprobaciones | Flujo aprobar/rechazar |
| 9 | Dashboard | Gráficos, métricas, filtros |
| 10 | Bot WhatsApp | Webhook, procesamiento, conversación |
| 11 | Reglas aprobación | Configuración multinivel |
| 12 | Combustible | Módulo de configuración |
| 13 | Notificaciones | Email, push |
| 14 | PWA | Offline, cámara |

---

## Preguntas para Definir

1. **¿Moneda principal?** CLP (peso chileno) u otra?
2. **¿Multiempresa?** ¿Un solo tenant o soporte para múltiples empresas?
3. **¿Proveedor de WhatsApp?** Twilio, Meta Business API directo, u otro?
4. **¿Hosting preferido?** Vercel, AWS, otro?
5. **¿Base de datos?** PostgreSQL managed (Supabase, Neon, Railway)?
6. **¿Presupuesto para APIs?** OpenAI Vision tiene costo por imagen procesada
