# ⚔️ Discordia

**Plataforma de comunicação em tempo real com salas temáticas, moderação e minigames interativos.**

---

## Sumário

- [Visão Geral](#visão-geral)
- [Funcionalidades](#funcionalidades)
- [Arquitetura do Sistema](#arquitetura-do-sistema)
- [Stack Tecnológica](#stack-tecnológica)
- [Estrutura do Projeto](#estrutura-do-projeto)
- [Fluxo de Comunicação](#fluxo-de-comunicação)
- [Autenticação e Segurança](#autenticação-e-segurança)
- [Minigame: Pedra, Papel e Tesoura](#minigame-pedra-papel-e-tesoura)
- [Modelo de Dados](#modelo-de-dados)
- [Pré-requisitos e Instalação](#pré-requisitos-e-instalação)
- [Variáveis de Ambiente](#variáveis-de-ambiente)

---

## Visão Geral

**Discordia** é uma aplicação de chat em tempo real desenvolvida com o propósito de reunir usuários em salas de discussão temáticas. O nome é proposital: ao contrário de plataformas que buscam consenso passivo, o Discordia foi projetado para o debate vivo — um espaço onde opiniões divergem, conversas acontecem de forma dinâmica e a interação entre usuários vai além da troca de mensagens.

A plataforma combina funcionalidades robustas de comunicação em tempo real com um sistema de moderação hierárquica e um minigame integrado de **Pedra, Papel e Tesoura**, permitindo que membros de uma mesma sala se desafiem diretamente durante as conversas.

---

## Funcionalidades

### Autenticação e Sessões

- Cadastro de usuários com validação de dados
- Login com geração de token **JWT** (JSON Web Token)
- Sessões isoladas por aba do navegador, garantindo independência entre contextos simultâneos
- Controle de acesso baseado em papéis: `ADMIN` e `USER`

### Salas de Discussão

- Criação e exclusão de salas por usuários com perfil de administrador
- Entrada e saída de salas com atualização da lista de membros em **tempo real**
- Histórico de mensagens persistido por sala, carregado ao ingressar

### Mensagens e Interação

- Envio e recebimento de mensagens via **WebSocket** com protocolo **STOMP** sobre **SockJS**
- Indicador de digitação em tempo real, visível para todos os membros da sala
- Timestamps e identificação do remetente em cada mensagem

### Moderação

| Permissão | Administrador | Usuário Comum |
|-----------|:---:|:---:|
| Excluir qualquer mensagem | ✅ | ❌ |
| Excluir próprias mensagens | ✅ | ✅ |
| Criar salas | ✅ | ❌ |
| Excluir salas | ✅ | ❌ |
| Entrar em salas | ✅ | ✅ |

### Minigame Integrado

- Desafio de **Pedra, Papel e Tesoura** em tempo real entre membros da sala
- Fluxo completo: convite → aceite → escolha → resultado, tudo via WebSocket
- Resultado exibido para os participantes e, opcionalmente, para a sala

---

## Arquitetura do Sistema

O Discordia adota uma arquitetura de **três camadas desacopladas**, com comunicação assíncrona via mensageria:

```
┌─────────────────────────────────────────────────────┐
│                    CLIENTE (Browser)                │
│          React + Vite  │  SockJS + STOMP Client     │
└────────────┬───────────────────────┬────────────────┘
             │ HTTP/REST (JWT)       │ WebSocket (STOMP)
             ▼                      ▼
┌─────────────────────────────────────────────────────┐
│                  BACKEND (Spring Boot)              │
│  REST Controllers  │  WebSocket Handlers  │  Auth   │
│                    │                      │  (JWT)  │
│           ┌────────┴──────────┐                     │
│           │  Service Layer    │                     │
│           └────────┬──────────┘                     │
│                    │                                │
│         ┌──────────┴──────────┐                     │
│         │     RabbitMQ        │                     │
│         │  (Message Broker)   │                     │
│         └──────────┬──────────┘                     │
└──────────────────────────────────────────────────── ┘
                     │
             ┌───────▼───────┐
             │  PostgreSQL   │
             │  (Persistence)│
             └───────────────┘
```

### Camadas do Backend

**Controllers REST** — Gerenciam requisições HTTP para autenticação, criação de salas e consulta de histórico. Todas as rotas protegidas exigem token JWT no header `Authorization`.

**WebSocket Handlers** — Recebem e distribuem eventos em tempo real (mensagens, entrada/saída de membros, indicador de digitação, eventos do minigame) por meio de tópicos STOMP.

**Service Layer** — Contém a lógica de negócio, orquestração de eventos e integração com o broker de mensagens.

**RabbitMQ** — Atua como broker intermediário para desacoplar a produção de eventos da entrega aos assinantes, garantindo resiliência e escalabilidade no roteamento de mensagens.

---

## Stack Tecnológica

| Camada | Tecnologia | Finalidade |
|--------|------------|------------|
| **Backend** | Spring Boot 3.x | API REST e gerenciamento de WebSockets |
| **Frontend** | React 18 + Vite | Interface de usuário reativa e SPA |
| **Banco de Dados** | PostgreSQL 15+ | Persistência de usuários, salas e mensagens |
| **Mensageria** | RabbitMQ | Broker para roteamento de eventos em tempo real |
| **Tempo Real** | WebSocket + STOMP + SockJS | Canal bidirecional entre cliente e servidor |
| **Autenticação** | JWT (JSON Web Token) | Stateless auth com controle de acesso por papel |

---

## Estrutura do Projeto

```
discordia/
├── backend/                        # API Spring Boot
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/
│   │   │   │   └── com/discordia/
│   │   │   │       ├── config/     # Configurações (Security, WebSocket, RabbitMQ)
│   │   │   │       ├── controller/ # REST e WebSocket controllers
│   │   │   │       ├── dto/        # Objetos de transferência de dados
│   │   │   │       ├── model/      # Entidades JPA
│   │   │   │       ├── repository/ # Interfaces Spring Data JPA
│   │   │   │       ├── security/   # Filtros JWT e UserDetails
│   │   │   │       └── service/    # Lógica de negócio
│   │   │   └── resources/
│   │   │       └── application.yml
│   └── pom.xml
│
└── frontend/                       # Interface React
    ├── src/
    │   ├── components/             # Componentes reutilizáveis
    │   ├── pages/                  # Páginas da aplicação
    │   ├── hooks/                  # Custom hooks (WebSocket, auth)
    │   ├── services/               # Clientes HTTP e WebSocket
    │   └── context/                # Contextos React (Auth, Socket)
    ├── index.html
    └── vite.config.js
```

---

## Fluxo de Comunicação

### Mensagens em Tempo Real

```
Cliente A                   Servidor                  Cliente B
   │                           │                           │
   │── STOMP SEND ────────────>│                           │
   │   /app/room/{id}/send     │                           │
   │                           │── persiste no BD          │
   │                           │── publica no RabbitMQ     │
   │                           │                           │
   │<── STOMP MESSAGE ─────────│─── STOMP MESSAGE ────────>│
   │   /topic/room/{id}        │   /topic/room/{id}        │
```

### Indicador de Digitação

```
Cliente A digita            Servidor               Clientes na sala
   │                           │                           │
   │── STOMP SEND ────────────>│                           │
   │   /app/room/{id}/typing   │                           │
   │                           │── broadcast ─────────────>│
   │                           │   /topic/room/{id}/typing │
```

---

## Autenticação e Segurança

O Discordia utiliza **JWT stateless** para autenticação:

1. O cliente realiza login via `POST /api/auth/login` com credenciais
2. O servidor retorna um token JWT assinado contendo `userId`, `username` e `role`
3. O token é enviado no header `Authorization: Bearer <token>` em todas as requisições REST
4. Na conexão WebSocket, o token é validado no handshake inicial
5. Sessões são isoladas por aba do navegador, sem compartilhamento de estado entre contextos

O controle de acesso é aplicado em duas camadas:
- **Nível de rota** — via Spring Security, com base na role do token
- **Nível de negócio** — verificação explícita no service (ex: somente o dono da mensagem ou admin pode excluí-la)

---

## Minigame: Pedra, Papel e Tesoura

O minigame ocorre inteiramente via WebSocket, em tempo real entre dois membros da mesma sala.

### Fluxo do Jogo

```
Desafiante                  Servidor                  Desafiado
   │                           │                           │
   │── CHALLENGE ─────────────>│── notifica ──────────────>│
   │   /app/game/challenge     │                           │
   │                           │<── ACCEPT ────────────────│
   │                           │   /app/game/accept        │
   │<── GAME_START ────────────│── GAME_START ────────────>│
   │                           │                           │
   │── MOVE ──────────────────>│<── MOVE ──────────────────│
   │   /app/game/move          │   /app/game/move          │
   │                           │                           │
   │<── RESULT ────────────────│── RESULT ────────────────>│
   │   (vencedor, jogadas)     │   (vencedor, jogadas)     │
```

### Estados da Partida

`IDLE` → `CHALLENGED` → `IN_PROGRESS` → `FINISHED`

---

## Modelo de Dados

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│    users    │         │    rooms    │         │  messages   │
├─────────────┤         ├─────────────┤         ├─────────────┤
│ id (PK)     │    ┌───>│ id (PK)     │<───┐    │ id (PK)     │
│ username    │    │    │ name        │    │    │ content     │
│ password    │    │    │ description │    │    │ created_at  │
│ role        │    │    │ created_by  │    │    │ room_id(FK) │
│ created_at  │    │    │ created_at  │    └────│ user_id(FK) │
└─────────────┘    │    └─────────────┘         └─────────────┘
       │           │
       │    ┌──────────────┐
       └───>│ room_members │
            ├──────────────┤
            │ user_id (FK) │
            │ room_id (FK) │
            │ joined_at    │
            └──────────────┘
```

---

## Pré-requisitos e Instalação

### Requisitos

- Java 17+
- Node.js 18+
- PostgreSQL 15+
- RabbitMQ 3.x (com plugin STOMP habilitado)

### Backend

```bash
cd backend

# Configure as variáveis de ambiente (ver seção abaixo)
cp src/main/resources/application.yml.example src/main/resources/application.yml

# Build e execução
./mvnw spring-boot:run
```

### Frontend

```bash
cd frontend

npm install
npm run dev
```

A aplicação estará disponível em `http://localhost:5173` por padrão.

---

## Variáveis de Ambiente

### Backend (`application.yml`)

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/discordia
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

  rabbitmq:
    host: ${RABBITMQ_HOST:localhost}
    port: ${RABBITMQ_PORT:5672}
    username: ${RABBITMQ_USERNAME:guest}
    password: ${RABBITMQ_PASSWORD:guest}

app:
  jwt:
    secret: ${JWT_SECRET}
    expiration: 86400000  # 24 horas em ms
```

### Frontend (`.env`)

```env
VITE_API_BASE_URL=http://localhost:8080
VITE_WS_URL=http://localhost:8080/ws
```

---

Desenvolvido com ☕ e WebSockets.
