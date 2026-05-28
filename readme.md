<div align="center">

<img src="https://img.shields.io/badge/React-18-61DAFB?style=for-the-badge&logo=react&logoColor=black" alt="React 18"/>
<img src="https://img.shields.io/badge/PostgreSQL-15-336791?style=for-the-badge&logo=postgresql&logoColor=white" alt="PostgreSQL"/>
<img src="https://img.shields.io/badge/Vite-5-646CFF?style=for-the-badge&logo=vite&logoColor=white" alt="Vite"/>
<img src="https://img.shields.io/badge/Cypress-E2E-17202C?style=for-the-badge&logo=cypress&logoColor=white" alt="Cypress"/>
<img src="https://img.shields.io/badge/license-MIT-green?style=for-the-badge" alt="MIT"/>

# RH Manager Pro

**Sistema web completo de Gestão de Recursos Humanos**

Cadastro de funcionários · Departamentos · Cargos · Relatórios PDF · Auditoria · RBAC

[Funcionalidades](#-funcionalidades) · [Instalação](#-instalação) · [Banco de Dados](#-banco-de-dados) · [Permissões](#-permissões) · [Testes](#-testes-cypress) · [API](#-api-service-layer)

</div>

---

## ✨ Funcionalidades

| Módulo | Descrição |
|---|---|
| **Dashboard** | KPIs em tempo real, headcount por departamento, distribuição de status |
| **Funcionários** | CRUD completo com foto (webcam/upload), formulário em 4 abas, ficha PDF |
| **Departamentos** | Cards visuais, centro de custo, gestor responsável, cor de identificação |
| **Cargos** | Nível hierárquico, faixa salarial, vinculação a departamento |
| **Usuários** | Gestão de acesso com 4 perfis (Admin, Gestor RH, Gestor, Colaborador) |
| **Relatórios** | 4 relatórios exportáveis em PDF — funcionários, folha, headcount, departamentos |
| **Auditoria** | Log imutável de todas as ações, acessível somente pelo Administrador |

---

## 🚀 Instalação

### Pré-requisitos

- Node.js 18+
- npm 9+ ou yarn
- PostgreSQL 15+ (produção)

### Frontend (Vite)

```bash
# 1. Criar o projeto
npm create vite@latest rh-manager -- --template react
cd rh-manager && npm install

# 2. Substituir o App.jsx
cp hr-pro.jsx src/App.jsx

# 3. Atualizar src/main.jsx
```

```jsx
// src/main.jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App.jsx'

ReactDOM.createRoot(document.getElementById('root')).render(<App />)
```

```bash
# 4. Iniciar
npm run dev   # http://localhost:5173

# 5. Build de produção
npm run build
```

### Variáveis de Ambiente

Crie um arquivo `.env` na raiz:

```env
VITE_API_BASE_URL=http://localhost:8000
VITE_APP_VERSION=1.0.0
DATABASE_URL=postgresql://user:password@localhost:5432/rh_manager
JWT_SECRET=sua_chave_secreta_aqui
```

---

## 🗄️ Banco de Dados

O banco usa **PostgreSQL 15+** com UUIDs como chaves primárias e `pgcrypto` para geração automática.

### Diagrama ER

```
departments ──< positions ──< employees
     │                              │
     └──── system_users ────────────┘
                    │
               audit_logs
```

### Setup Completo

```sql
-- Habilitar extensão UUID
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Departamentos
CREATE TABLE departments (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name         VARCHAR(120) NOT NULL UNIQUE,
  cost_center  VARCHAR(20),
  manager_id   UUID,
  active       BOOLEAN NOT NULL DEFAULT TRUE,
  color        VARCHAR(7) DEFAULT '#1D4ED8',
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Cargos
CREATE TABLE positions (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title       VARCHAR(120) NOT NULL,
  dept_id     UUID NOT NULL REFERENCES departments(id) ON DELETE RESTRICT,
  level       VARCHAR(30) CHECK (level IN (
                'Estagiário','Júnior','Pleno','Sênior',
                'Especialista','Coordenador','Gerente','Diretor')),
  salary_min  NUMERIC(12,2),
  salary_max  NUMERIC(12,2),
  description TEXT,
  active      BOOLEAN NOT NULL DEFAULT TRUE,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Funcionários
CREATE TABLE employees (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name             VARCHAR(200) NOT NULL,
  email            VARCHAR(200) NOT NULL UNIQUE,
  cpf              VARCHAR(14)  NOT NULL UNIQUE,
  rg               VARCHAR(20),
  birth_date       DATE,
  gender           VARCHAR(40),
  marital_status   VARCHAR(40),
  education        VARCHAR(60),
  nationality      VARCHAR(60)  DEFAULT 'Brasileira',
  photo_url        TEXT,
  phone            VARCHAR(20),
  phone2           VARCHAR(20),
  cep              VARCHAR(9),
  address          VARCHAR(200),
  city             VARCHAR(100),
  state            CHAR(2),
  dept_id          UUID REFERENCES departments(id) ON DELETE SET NULL,
  position_id      UUID REFERENCES positions(id)   ON DELETE SET NULL,
  manager_id       UUID REFERENCES employees(id)   ON DELETE SET NULL,
  salary           NUMERIC(12,2) NOT NULL,
  contract_type    VARCHAR(20) CHECK (contract_type IN (
                     'CLT','PJ','Estágio','Temporário','Freelancer')),
  workshift        VARCHAR(20),
  start_date       DATE NOT NULL,
  end_date         DATE,
  status           VARCHAR(20) NOT NULL DEFAULT 'ativo'
                     CHECK (status IN ('ativo','férias','licença','inativo')),
  ctps             VARCHAR(20),
  pis              VARCHAR(20),
  voter_reg        VARCHAR(30),
  reservist        VARCHAR(30),
  bank_name        VARCHAR(100),
  agency           VARCHAR(20),
  account          VARCHAR(30),
  account_type     VARCHAR(20),
  emergency_name   VARCHAR(200),
  emergency_phone  VARCHAR(20),
  emergency_rel    VARCHAR(60),
  notes            TEXT,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Usuários do sistema
CREATE TABLE system_users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name          VARCHAR(200) NOT NULL,
  email         VARCHAR(200) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  role          VARCHAR(20) NOT NULL DEFAULT 'employee'
                  CHECK (role IN ('admin','rh','manager','employee')),
  dept_id       UUID REFERENCES departments(id) ON DELETE SET NULL,
  active        BOOLEAN NOT NULL DEFAULT TRUE,
  avatar        VARCHAR(4),
  last_login    TIMESTAMPTZ,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Log de auditoria
CREATE TABLE audit_logs (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID REFERENCES system_users(id) ON DELETE SET NULL,
  user_name   VARCHAR(200),
  action      TEXT NOT NULL,
  module      VARCHAR(40) NOT NULL
                CHECK (module IN (
                  'auth','employees','departments',
                  'positions','users','system')),
  ip_address  INET,
  user_agent  TEXT,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- FK de gestor do departamento
ALTER TABLE departments
  ADD CONSTRAINT fk_dept_manager
  FOREIGN KEY (manager_id) REFERENCES system_users(id) ON DELETE SET NULL;

-- Índices
CREATE INDEX idx_employees_dept   ON employees(dept_id);
CREATE INDEX idx_employees_status ON employees(status);
CREATE INDEX idx_employees_cpf    ON employees(cpf);
CREATE INDEX idx_positions_dept   ON positions(dept_id);
CREATE INDEX idx_audit_user       ON audit_logs(user_id);
CREATE INDEX idx_audit_created    ON audit_logs(created_at DESC);

-- Trigger updated_at
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = NOW(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_employees_updated
  BEFORE UPDATE ON employees
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE TRIGGER trg_departments_updated
  BEFORE UPDATE ON departments
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE TRIGGER trg_positions_updated
  BEFORE UPDATE ON positions
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

---

## 🔐 Permissões (RBAC)

| Perfil | `role` | Módulos Acessíveis |
|---|---|---|
| **Administrador** | `admin` | Todos os módulos |
| **Gestor RH** | `rh` | Funcionários, Departamentos, Cargos, Relatórios |
| **Gestor** | `manager` | Funcionários (leitura), Relatórios (leitura) |
| **Colaborador** | `employee` | Dashboard apenas |

### Contas de Demonstração

| E-mail | Senha | Perfil |
|---|---|---|
| `admin@empresa.com` | `admin123` | Administrador |
| `maria@empresa.com` | `rh1234` | Gestor RH |
| `carlos@empresa.com` | `gest123` | Gestor |
| `ana@empresa.com` | `colab123` | Colaborador |

---

## 🔌 API Service Layer

Toda comunicação com o backend está abstraída no objeto `Api` no topo de `App.jsx`. Para conectar ao PostgreSQL real, substitua apenas o corpo dessas funções:

```js
// ANTES (mock em memória):
list: async (f = {}) => { await delay(); return DB.employees.filter(...); },

// DEPOIS (backend real):
list: async (f = {}) => {
  const params = new URLSearchParams(f);
  const res = await fetch(`/api/employees?${params}`, {
    headers: { Authorization: `Bearer ${localStorage.getItem('token')}` }
  });
  if (!res.ok) throw new Error(await res.text());
  return res.json();
},
```

### Endpoints Esperados

| Método | Endpoint | Descrição |
|---|---|---|
| `POST` | `/api/auth/login` | Login → `{ user, token }` |
| `GET` | `/api/employees` | Listar (`?search=`, `?dept=`, `?status=`) |
| `POST` | `/api/employees` | Criar |
| `PUT` | `/api/employees/:id` | Atualizar |
| `DELETE` | `/api/employees/:id` | Remover |
| `GET/POST/PUT/DELETE` | `/api/departments` | CRUD departamentos |
| `GET/POST/PUT/DELETE` | `/api/positions` | CRUD cargos |
| `GET/POST/PUT/DELETE` | `/api/users` | CRUD usuários do sistema |
| `GET` | `/api/audit` | Log de auditoria |

---

## 🧪 Testes Cypress

O projeto inclui **50+ casos de teste E2E** em 11 suites, com seletores 100% baseados em `data-testid`.

```bash
npm install cypress --save-dev
mkdir -p cypress/e2e
cp hr.cy.js cypress/e2e/

npx cypress open   # modo interativo
npx cypress run    # headless / CI
```

### Suites

| # | Suite | Casos |
|---|---|---|
| 1 | Autenticação | Login, logout, credenciais inválidas, perfis |
| 2 | Dashboard | KPIs, tabela de admissões |
| 3 | Funcionários — Listagem | Busca, filtros de departamento e status |
| 4 | Funcionários — CRUD | Criar, editar, excluir, validação, abas |
| 5 | Departamentos — CRUD | Cards, cores, CRUD completo |
| 6 | Cargos — CRUD | Tabela, níveis, faixas salariais |
| 7 | Usuários — CRUD | Permissões, proteção self-delete |
| 8 | Relatórios / PDF | Exportação, filtros, stub de window.open |
| 9 | Auditoria | Log automático, acesso restrito |
| 10 | UI Geral | Toast, navegação, fechar modais |
| 11 | Controle de Acesso | Visibilidade por perfil |

### Comandos Customizados

```js
cy.loginAs('admin')               // faz login com o perfil
cy.navigateTo('employees')        // navega para o módulo
cy.fillEmployeeForm({ name, ... }) // preenche o formulário
cy.confirmDialog()                 // confirma diálogo de exclusão
cy.closeModal('employee-form-modal') // fecha modal pelo testid
```

---

## 📁 Estrutura do Projeto

```
rh-manager/
├── src/
│   └── App.jsx              # Aplicação completa (componente único)
├── cypress/
│   └── e2e/
│       └── hr.cy.js         # Casos de teste E2E
├── cypress.config.js        # Configuração do Cypress
├── vite.config.js
└── package.json
```

---

## 🗺️ Roadmap

- [ ] Autenticação JWT real com refresh token
- [ ] Integração com backend Node.js + Express
- [ ] Upload de foto para AWS S3 / Cloudflare R2
- [ ] Módulo de controle de férias e licenças
- [ ] Organograma visual por departamento
- [ ] Notificações por e-mail (aniversários, vencimento de contratos)
- [ ] Exportação de relatórios em Excel (XLSX)
- [ ] PWA — instalável no celular

---

## 🤝 Contribuindo

1. Fork o repositório
2. Crie uma branch: `git checkout -b feat/nome-da-feature`
3. Commit: `git commit -m 'feat: descrição clara'`
4. Push: `git push origin feat/nome-da-feature`
5. Abra um Pull Request

Certifique-se de que os testes Cypress estão passando antes de abrir o PR.

---

## 📄 Licença

MIT — livre para uso comercial e privado com atribuição.

---

<div align="center">
Feito com React · PostgreSQL · Cypress &nbsp;·&nbsp; Goiânia, GO · 2025
</div>

---

## 🖥️ Backend — Node.js + Express

O backend expõe a API REST consumida pelo frontend React.

### Instalação

```bash
cd backend
npm install express pg jsonwebtoken bcrypt cors dotenv multer
npm install nodemon --save-dev
npm run dev   # porta 3000
```

### Estrutura

```
backend/
├── src/
│   ├── server.js          # Entry point
│   ├── routes/            # auth, employees, departments, positions, users, audit
│   ├── middleware/
│   │   ├── auth.js        # Verificação JWT
│   │   └── audit.js       # Log automático
│   └── db/
│       └── index.js       # Pool de conexões PostgreSQL (node-postgres)
└── .env
```

### .env

```env
PORT=3000
DATABASE_URL=postgresql://usuario:senha@localhost:5432/rh_manager
JWT_SECRET=chave_secreta_longa_aqui
JWT_EXPIRES_IN=8h
FRONTEND_URL=http://localhost:5173
```

### Dependências do Backend

| Pacote | Função |
|---|---|
| `express` | Framework HTTP e roteamento |
| `pg` | Driver PostgreSQL para Node.js |
| `jsonwebtoken` | Geração e verificação de tokens JWT |
| `bcrypt` | Hash seguro de senhas |
| `cors` | Liberar requisições do frontend |
| `dotenv` | Variáveis de ambiente via `.env` |
| `multer` | Upload de arquivos (foto do funcionário) |
| `nodemon` | Reinício automático em desenvolvimento |

### Como rodar tudo junto

```bash
# Terminal 1 — banco
psql -U postgres -d rh_manager -f schema.sql

# Terminal 2 — backend
cd backend && npm run dev

# Terminal 3 — frontend
cd frontend && npm run dev

# Acessar: http://localhost:5173
```
