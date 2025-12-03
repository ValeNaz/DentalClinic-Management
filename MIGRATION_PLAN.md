# Piano di Migrazione: Odoo Dental Clinic → React

## Panoramica del Progetto

**Obiettivo**: Migrare il Sistema di Gestione Clinica Dentale da Odoo a un'applicazione React moderna con backend Node.js/Express.

**Stack Tecnologico Target**:
- **Frontend**: React 18 + TypeScript + Vite
- **State Management**: Zustand
- **Forms**: React Hook Form + Zod
- **UI Library**: Shadcn/ui + Tailwind CSS
- **Backend**: Node.js + Express + TypeScript
- **Database**: PostgreSQL (migrato da Odoo)
- **ORM**: Prisma
- **Autenticazione**: JWT + Refresh Tokens
- **API**: REST + OpenAPI/Swagger

---

## Fase 1: Setup Infrastruttura Base

### 1.1 Struttura del Progetto Monorepo

```
dental-clinic-react/
├── apps/
│   ├── web/                    # Frontend React
│   │   ├── src/
│   │   │   ├── components/     # Componenti riutilizzabili
│   │   │   ├── features/       # Feature modules
│   │   │   ├── hooks/          # Custom hooks
│   │   │   ├── lib/            # Utilities
│   │   │   ├── pages/          # Route pages
│   │   │   ├── stores/         # Zustand stores
│   │   │   └── types/          # TypeScript types
│   │   └── package.json
│   │
│   └── api/                    # Backend Express
│       ├── src/
│       │   ├── controllers/    # Route handlers
│       │   ├── middleware/     # Auth, validation, etc.
│       │   ├── models/         # Prisma models
│       │   ├── routes/         # API routes
│       │   ├── services/       # Business logic
│       │   └── utils/          # Helpers
│       └── package.json
│
├── packages/
│   └── shared/                 # Tipi condivisi frontend/backend
│       ├── types/
│       └── validators/
│
├── docker-compose.yml
├── package.json                # Root workspace
└── turbo.json                  # Turborepo config
```

### 1.2 Configurazione Iniziale

- [ ] Inizializzare monorepo con pnpm workspaces + Turborepo
- [ ] Setup Vite + React + TypeScript per frontend
- [ ] Setup Express + TypeScript per backend
- [ ] Configurare ESLint + Prettier condivisi
- [ ] Setup Docker per PostgreSQL locale
- [ ] Configurare variabili ambiente (.env)

---

## Fase 2: Database e Schema

### 2.1 Schema Prisma (Mappatura da Odoo)

```prisma
// schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ==================== PAZIENTI ====================
model Patient {
  id            Int       @id @default(autoincrement())
  serialNumber  String    @unique @default(dbgenerated("'PAT-' || LPAD(nextval('patient_seq')::text, 6, '0')"))
  name          String
  contactNumber String?
  dateOfBirth   DateTime?
  gender        Gender?
  occupation    String?
  maritalStatus MaritalStatus?
  bloodType     BloodType?

  // Questionario medico
  question1     Boolean   @default(false)
  question1Note String?
  question2     Boolean   @default(false)
  question2Note String?

  // Relazioni
  appointments  Appointment[]
  prescriptions Prescription[]

  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
}

enum Gender {
  MALE
  FEMALE
}

enum MaritalStatus {
  SINGLE
  MARRIED
  DIVORCED
}

enum BloodType {
  A_POSITIVE
  A_NEGATIVE
  B_POSITIVE
  B_NEGATIVE
  AB_POSITIVE
  AB_NEGATIVE
  O_POSITIVE
  O_NEGATIVE
}

// ==================== DOTTORI ====================
model Doctor {
  id           Int           @id @default(autoincrement())
  name         String
  specialization String?
  email        String?       @unique
  phone        String?

  appointments Appointment[]

  createdAt    DateTime      @default(now())
  updatedAt    DateTime      @updatedAt
}

// ==================== APPUNTAMENTI ====================
model Appointment {
  id            Int               @id @default(autoincrement())
  serialNumber  String            @unique @default(dbgenerated("'APT-' || LPAD(nextval('appointment_seq')::text, 6, '0')"))

  patientId     Int
  patient       Patient           @relation(fields: [patientId], references: [id])

  doctorId      Int?
  doctor        Doctor?           @relation(fields: [doctorId], references: [id])

  assignedToId  Int?
  assignedTo    User?             @relation(fields: [assignedToId], references: [id])

  status        AppointmentStatus @default(DRAFT)
  type          AppointmentType   @default(RESERVED)

  startTime     DateTime
  endTime       DateTime
  allDay        Boolean           @default(false)

  chiefComplaints String?         @db.Text
  notes         String?           @db.Text

  // Relazioni
  procedures    DentalProcedure[]
  prescriptions Prescription[]
  attachments   Attachment[]

  createdAt     DateTime          @default(now())
  updatedAt     DateTime          @updatedAt
}

enum AppointmentStatus {
  DRAFT
  CONFIRMED
  IN_EXAM
  EXAM_COMPLETED
  COMPLETED
  CANCELLED
}

enum AppointmentType {
  RESERVED
  WALK_IN
}

// ==================== PROCEDURE DENTALI ====================
model DentalProcedure {
  id            Int         @id @default(autoincrement())

  appointmentId Int
  appointment   Appointment @relation(fields: [appointmentId], references: [id], onDelete: Cascade)

  toothNumber   Int         // 1-32 (sistema FDI)

  serviceId     Int?
  service       Service?    @relation(fields: [serviceId], references: [id])

  cost          Decimal     @db.Decimal(10, 2)
  notes         String?

  createdAt     DateTime    @default(now())
}

model Service {
  id          Int               @id @default(autoincrement())
  name        String
  description String?
  price       Decimal           @db.Decimal(10, 2)
  category    String?

  procedures  DentalProcedure[]

  createdAt   DateTime          @default(now())
  updatedAt   DateTime          @updatedAt
}

// ==================== PRESCRIZIONI ====================
model Prescription {
  id            Int                  @id @default(autoincrement())
  serialNumber  String               @unique @default(dbgenerated("'PRS-' || LPAD(nextval('prescription_seq')::text, 6, '0')"))

  patientId     Int
  patient       Patient              @relation(fields: [patientId], references: [id])

  appointmentId Int?
  appointment   Appointment?         @relation(fields: [appointmentId], references: [id])

  date          DateTime             @default(now())
  notes         String?              @db.Text

  lines         PrescriptionLine[]

  createdAt     DateTime             @default(now())
  updatedAt     DateTime             @updatedAt
}

model PrescriptionLine {
  id                 Int          @id @default(autoincrement())

  prescriptionId     Int
  prescription       Prescription @relation(fields: [prescriptionId], references: [id], onDelete: Cascade)

  medicineName       String
  therapeuticRegimen String?      // Posologia
  quantity           Int?
  notes              String?

  createdAt          DateTime     @default(now())
}

// ==================== ALLEGATI ====================
model Attachment {
  id            Int         @id @default(autoincrement())

  appointmentId Int
  appointment   Appointment @relation(fields: [appointmentId], references: [id], onDelete: Cascade)

  fileName      String
  fileType      String
  fileSize      Int
  filePath      String      // Path su storage (S3 o locale)

  uploadedAt    DateTime    @default(now())
}

// ==================== UTENTI & AUTH ====================
model User {
  id            Int           @id @default(autoincrement())
  email         String        @unique
  passwordHash  String
  name          String
  role          UserRole      @default(STAFF)

  refreshTokens RefreshToken[]
  appointments  Appointment[]

  createdAt     DateTime      @default(now())
  updatedAt     DateTime      @updatedAt
}

enum UserRole {
  ADMIN
  DOCTOR
  STAFF
}

model RefreshToken {
  id        Int      @id @default(autoincrement())
  token     String   @unique
  userId    Int
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  expiresAt DateTime
  createdAt DateTime @default(now())
}
```

### 2.2 Task Database

- [ ] Creare schema Prisma completo
- [ ] Creare migrations iniziali
- [ ] Script per sequenze automatiche (PAT-000001, etc.)
- [ ] Seed data per testing
- [ ] Script migrazione dati da Odoo (opzionale)

---

## Fase 3: Backend API

### 3.1 Struttura API Endpoints

```
API Base: /api/v1

AUTH
├── POST   /auth/login              # Login con email/password
├── POST   /auth/register           # Registrazione (solo admin)
├── POST   /auth/refresh            # Refresh access token
├── POST   /auth/logout             # Logout (invalida refresh token)
└── GET    /auth/me                 # Profilo utente corrente

PATIENTS
├── GET    /patients                # Lista pazienti (paginata)
├── GET    /patients/:id            # Dettaglio paziente
├── POST   /patients                # Crea paziente
├── PUT    /patients/:id            # Aggiorna paziente
├── DELETE /patients/:id            # Elimina paziente
└── GET    /patients/:id/appointments  # Appuntamenti paziente

DOCTORS
├── GET    /doctors                 # Lista dottori
├── GET    /doctors/:id             # Dettaglio dottore
├── POST   /doctors                 # Crea dottore
├── PUT    /doctors/:id             # Aggiorna dottore
└── DELETE /doctors/:id             # Elimina dottore

APPOINTMENTS
├── GET    /appointments            # Lista appuntamenti (filtri: data, status, dottore)
├── GET    /appointments/calendar   # Formato calendario
├── GET    /appointments/:id        # Dettaglio appuntamento
├── POST   /appointments            # Crea appuntamento
├── PUT    /appointments/:id        # Aggiorna appuntamento
├── PATCH  /appointments/:id/status # Cambia stato
├── DELETE /appointments/:id        # Elimina appuntamento
│
├── POST   /appointments/:id/procedures      # Aggiungi procedura
├── DELETE /appointments/:id/procedures/:pid # Rimuovi procedura
│
├── POST   /appointments/:id/attachments     # Upload allegato
└── DELETE /appointments/:id/attachments/:aid # Rimuovi allegato

PRESCRIPTIONS
├── GET    /prescriptions           # Lista prescrizioni
├── GET    /prescriptions/:id       # Dettaglio prescrizione
├── POST   /prescriptions           # Crea prescrizione
├── PUT    /prescriptions/:id       # Aggiorna prescrizione
└── DELETE /prescriptions/:id       # Elimina prescrizione

SERVICES
├── GET    /services                # Lista servizi/procedure
├── POST   /services                # Crea servizio
├── PUT    /services/:id            # Aggiorna servizio
└── DELETE /services/:id            # Elimina servizio

DASHBOARD
├── GET    /dashboard/stats         # Statistiche generali
└── GET    /dashboard/today         # Appuntamenti di oggi
```

### 3.2 Middleware

```typescript
// Middleware da implementare
- authMiddleware        // Verifica JWT
- roleMiddleware        // Controllo ruoli (admin, doctor, staff)
- validationMiddleware  // Validazione input con Zod
- errorMiddleware       // Gestione errori centralizzata
- rateLimitMiddleware   // Rate limiting
- corsMiddleware        // CORS configuration
- loggerMiddleware      // Request logging
```

### 3.3 Task Backend

- [ ] Setup Express con TypeScript
- [ ] Configurare Prisma Client
- [ ] Implementare sistema JWT (access + refresh tokens)
- [ ] Creare middleware autenticazione
- [ ] Creare middleware validazione (Zod)
- [ ] Implementare CRUD Patients
- [ ] Implementare CRUD Doctors
- [ ] Implementare CRUD Appointments (con workflow stati)
- [ ] Implementare CRUD Prescriptions
- [ ] Implementare CRUD Services
- [ ] Implementare upload file (allegati)
- [ ] Implementare endpoint calendario
- [ ] Implementare dashboard/statistiche
- [ ] Configurare Swagger/OpenAPI
- [ ] Error handling centralizzato
- [ ] Logging con Winston/Pino

---

## Fase 4: Frontend React

### 4.1 Struttura Pagine e Route

```
ROUTES

/                           → Dashboard
/login                      → Login page

/patients                   → Lista pazienti
/patients/new               → Nuovo paziente
/patients/:id               → Dettaglio paziente
/patients/:id/edit          → Modifica paziente

/appointments               → Calendario appuntamenti
/appointments/list          → Lista tabellare
/appointments/new           → Nuovo appuntamento
/appointments/:id           → Dettaglio appuntamento
/appointments/:id/edit      → Modifica appuntamento

/doctors                    → Lista dottori
/doctors/new                → Nuovo dottore
/doctors/:id                → Dettaglio/modifica dottore

/prescriptions              → Lista prescrizioni
/prescriptions/:id          → Dettaglio prescrizione

/services                   → Gestione servizi/tariffe

/settings                   → Impostazioni
/settings/profile           → Profilo utente
/settings/users             → Gestione utenti (admin)
```

### 4.2 Componenti Principali

```
COMPONENTS

Layout/
├── AppLayout               # Layout principale con sidebar
├── Sidebar                 # Menu navigazione
├── Header                  # Top bar con user menu
└── PageContainer           # Wrapper pagine

Common/
├── DataTable               # Tabella dati con sorting/filtering
├── Calendar                # Calendario appuntamenti
├── Modal                   # Dialog modali
├── ConfirmDialog           # Conferma azioni
├── LoadingSpinner          # Loading states
├── EmptyState              # Stato vuoto
├── Pagination              # Paginazione
└── SearchInput             # Ricerca

Forms/
├── PatientForm             # Form paziente completo
├── AppointmentForm         # Form appuntamento
├── PrescriptionForm        # Form prescrizione
├── DoctorForm              # Form dottore
└── ServiceForm             # Form servizio

Features/
├── ToothChart              # Chart dentale interattivo (SVG)
├── AppointmentCalendar     # Vista calendario
├── AppointmentTimeline     # Timeline appuntamenti
├── PatientHistory          # Storico paziente
├── ProcedureSelector       # Selezione procedure per dente
└── StatusBadge             # Badge stato appuntamento

Dashboard/
├── StatsCards              # Cards statistiche
├── TodayAppointments       # Lista appuntamenti oggi
├── RecentPatients          # Pazienti recenti
└── QuickActions            # Azioni rapide
```

### 4.3 State Management (Zustand Stores)

```typescript
// stores/authStore.ts
interface AuthStore {
  user: User | null
  isAuthenticated: boolean
  login: (credentials) => Promise<void>
  logout: () => void
  refreshToken: () => Promise<void>
}

// stores/patientStore.ts
interface PatientStore {
  patients: Patient[]
  selectedPatient: Patient | null
  isLoading: boolean
  fetchPatients: (filters?) => Promise<void>
  fetchPatient: (id) => Promise<void>
  createPatient: (data) => Promise<void>
  updatePatient: (id, data) => Promise<void>
  deletePatient: (id) => Promise<void>
}

// stores/appointmentStore.ts
interface AppointmentStore {
  appointments: Appointment[]
  calendarEvents: CalendarEvent[]
  selectedAppointment: Appointment | null
  filters: AppointmentFilters
  isLoading: boolean
  fetchAppointments: (filters?) => Promise<void>
  fetchCalendarData: (startDate, endDate) => Promise<void>
  createAppointment: (data) => Promise<void>
  updateAppointment: (id, data) => Promise<void>
  updateStatus: (id, status) => Promise<void>
  deleteAppointment: (id) => Promise<void>
}

// stores/uiStore.ts
interface UIStore {
  sidebarOpen: boolean
  theme: 'light' | 'dark'
  toggleSidebar: () => void
  setTheme: (theme) => void
}
```

### 4.4 Task Frontend

- [ ] Setup Vite + React + TypeScript
- [ ] Configurare Tailwind CSS + Shadcn/ui
- [ ] Configurare React Router
- [ ] Setup Zustand stores
- [ ] Configurare Axios + interceptors
- [ ] Implementare autenticazione (login, logout, refresh)
- [ ] Creare layout principale (sidebar, header)
- [ ] Implementare Dashboard
- [ ] Implementare modulo Pazienti (CRUD completo)
- [ ] Implementare modulo Dottori
- [ ] Implementare modulo Appuntamenti
  - [ ] Vista calendario
  - [ ] Vista lista
  - [ ] Form appuntamento
  - [ ] Workflow stati
- [ ] Implementare ToothChart interattivo
- [ ] Implementare modulo Prescrizioni
- [ ] Implementare modulo Servizi
- [ ] Implementare upload allegati
- [ ] Implementare ricerca globale
- [ ] Responsive design (mobile)
- [ ] Dark mode (opzionale)

---

## Fase 5: Tooth Chart Component

### 5.1 Migrazione da Odoo OWL a React

Il componente ToothChart esistente va convertito da OWL a React mantenendo:
- SVG interattivo con 32 denti
- Click per selezionare/deselezionare denti
- Sincronizzazione con form procedure
- Stili esistenti (marked/unmarked)

```typescript
// components/ToothChart/ToothChart.tsx

interface ToothChartProps {
  selectedTeeth: number[]
  onToothClick: (toothNumber: number) => void
  readOnly?: boolean
}

const ToothChart: React.FC<ToothChartProps> = ({
  selectedTeeth,
  onToothClick,
  readOnly = false
}) => {
  // Implementazione con SVG inline o componente separato
}
```

### 5.2 Task ToothChart

- [ ] Convertire SVG esistente in componente React
- [ ] Implementare logica selezione denti
- [ ] Aggiungere tooltip con info dente
- [ ] Integrare con form appuntamento
- [ ] Mantenere stili SCSS esistenti
- [ ] Aggiungere animazioni hover/click
- [ ] Supporto touch per mobile

---

## Fase 6: Testing

### 6.1 Backend Testing

```
- Unit tests per services (Jest)
- Integration tests per API endpoints (Supertest)
- Test autenticazione e autorizzazione
- Test validazione input
```

### 6.2 Frontend Testing

```
- Unit tests componenti (Vitest + Testing Library)
- Test hooks personalizzati
- Test stores Zustand
- E2E tests flussi principali (Playwright)
```

### 6.3 Task Testing

- [ ] Setup Jest per backend
- [ ] Setup Vitest + Testing Library per frontend
- [ ] Setup Playwright per E2E
- [ ] Scrivere test critici per auth
- [ ] Scrivere test per CRUD operations
- [ ] Scrivere test E2E per flussi principali
- [ ] Configurare CI/CD per test automatici

---

## Fase 7: Deployment

### 7.1 Architettura Deployment

```
                    ┌─────────────────┐
                    │   Cloudflare    │
                    │   (CDN + DNS)   │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
     ┌────────▼────────┐          ┌────────▼────────┐
     │  Vercel/Netlify │          │   Railway/Fly   │
     │   (Frontend)    │          │   (Backend)     │
     └─────────────────┘          └────────┬────────┘
                                           │
                              ┌────────────┴────────────┐
                              │                         │
                    ┌─────────▼─────────┐    ┌─────────▼─────────┐
                    │   PostgreSQL      │    │   S3/Cloudflare   │
                    │   (Supabase/      │    │   R2 (Files)      │
                    │    Neon/Railway)  │    │                   │
                    └───────────────────┘    └───────────────────┘
```

### 7.2 Opzioni Hosting

| Servizio | Frontend | Backend | Database | Costo Stimato |
|----------|----------|---------|----------|---------------|
| **Budget** | Vercel Free | Railway | Supabase Free | €0-20/mese |
| **Startup** | Vercel Pro | Railway Pro | Supabase Pro | €50-100/mese |
| **Enterprise** | AWS CloudFront | AWS ECS | AWS RDS | €200+/mese |

### 7.3 Task Deployment

- [ ] Configurare Docker per backend
- [ ] Setup database PostgreSQL (Supabase/Neon)
- [ ] Configurare storage file (S3/R2)
- [ ] Deploy frontend su Vercel
- [ ] Deploy backend su Railway/Fly.io
- [ ] Configurare variabili ambiente produzione
- [ ] Setup dominio e SSL
- [ ] Configurare backup database
- [ ] Setup monitoring (Sentry, LogRocket)
- [ ] Configurare CI/CD (GitHub Actions)

---

## Timeline e Priorità

### Sprint 1 (Settimana 1-2): Foundation
- [x] Analisi progetto Odoo esistente
- [ ] Setup monorepo e tooling
- [ ] Schema database Prisma
- [ ] Auth backend (JWT)
- [ ] Auth frontend (login page)

### Sprint 2 (Settimana 3-4): Core Backend
- [ ] CRUD Patients API
- [ ] CRUD Doctors API
- [ ] CRUD Appointments API (base)
- [ ] Validazione e error handling

### Sprint 3 (Settimana 5-6): Core Frontend
- [ ] Layout e navigazione
- [ ] Dashboard
- [ ] Modulo Pazienti completo
- [ ] Modulo Dottori

### Sprint 4 (Settimana 7-8): Appointments
- [ ] Calendario appuntamenti
- [ ] Form appuntamento completo
- [ ] Workflow stati
- [ ] ToothChart interattivo

### Sprint 5 (Settimana 9-10): Features Avanzate
- [ ] Prescrizioni
- [ ] Upload allegati
- [ ] Servizi/tariffe
- [ ] Ricerca e filtri avanzati

### Sprint 6 (Settimana 11-12): Polish & Deploy
- [ ] Testing E2E
- [ ] Bug fixing
- [ ] Performance optimization
- [ ] Deployment produzione
- [ ] Documentazione

---

## Rischi e Mitigazioni

| Rischio | Probabilità | Impatto | Mitigazione |
|---------|-------------|---------|-------------|
| Complessità form appuntamenti | Alta | Alto | Spezzare in sotto-componenti, testing incrementale |
| Migrazione dati esistenti | Media | Alto | Script migrazione + validazione manuale |
| Performance calendario | Media | Medio | Virtualizzazione, lazy loading, caching |
| Compatibilità browser | Bassa | Medio | Testing cross-browser, polyfills |
| Sicurezza API | Media | Alto | Audit sicurezza, rate limiting, validazione input |

---

## Note Finali

### Cosa si perde rispetto a Odoo
- Integrazione ERP (contabilità, fatturazione, POS)
- Sistema messaggistica/attività integrato
- Moduli aggiuntivi Odoo ecosystem
- Multi-company out-of-the-box

### Cosa si guadagna
- Performance superiore (SPA)
- UX moderna e responsive
- Facilità di customizzazione
- Hosting flessibile e economico
- Stack tecnologico più comune
- Possibilità di PWA/mobile app
- Controllo totale sul codice

### Prossimi Passi Immediati
1. Conferma stack tecnologico con stakeholder
2. Setup ambiente di sviluppo
3. Inizio Sprint 1

---

*Piano creato il: 2024*
*Ultima modifica: In attesa di approvazione*
