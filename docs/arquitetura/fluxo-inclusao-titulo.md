
# 📡 Fluxo de Chamada da API – Inclusão de Título

---

## 🧾 Metadados do Documento

| Campo | Valor |
|-----|------|
| Sistema | NPC |
| Domínio | JD CONSULTORES |
| Responsável | LUCAS |
| Time | JD IMPLANTAÇÃO |
| Versão | 1.0 |
| Status | Aprovado |
| Última atualização | 20-01-2026 |

---

## 🎯 Objetivo

Descrever de forma clara e rastreável a arquitetura, responsabilidades e fluxos do sistema ou funcionalidade documentada.

---

## 🧠 Contexto

> **Contexto:**  
> Este documento descreve o fluxo completo de inclusão de título, desde a chamada da API até o retorno da Nuclea,
> incluindo idempotência, mensageria e persistência.

---

## 🔁 Visão Geral do Fluxo

```mermaid
flowchart TD
    A[Cliente / Sistema Origem] --> B[API Inclusão de Título]

    B --> C{Chave de Idempotência<br/>existe no Redis?}

    C -->|Sim| D[Retorna<br/>Chave_idempotência existente]

    C -->|Não| E[Publica mensagem<br/>no RabbitMQ]
    E --> F[Grava chave<br/>no Redis]

    E --> G[jdsfn-persister-worker-deploy]
    G --> H[(TbJdNpcDst_Titulo)]

    H --> I[jdsfn-router-worker-deploy]
    I --> J[jd.npc.dst.titulo.inclui.command.queue]

    J --> K[jdsfn-send-worker-deploy]
    K --> L[MQ - Nuclea]
    K --> M[jd.npc.dst.titulo.dda0101.send-result.event.queue]

    M --> N[jdsfn-persister-worker-deploy]
    N --> O[(TbJdSfn_MsgTransmitida)]
    N --> P[(TbJdNpcDst_TituloTrnsmtd)]

    L --> Q[Nuclea]
    Q -->|DDA0101R1| R[Sucesso]
    Q -->|DDA0101E| S[Erro]

    R --> T[jdsfn-receive-worker-deploy]
    S --> T

    T --> U[jd.sfn.receive.router.msg.queue]

    U --> V[jdsfn-router-worker-deploy]
    V --> W[jd.npc.dst.titulo.dda0101r1.event.queue]
    V --> X[jd.npc.dst.titulo.dda0101e.event.queue]
    V --> Y[jd.sfn.receive.persister.queue]

    Y --> Z[jdsfn-persister-worker-deploy]
    Z --> AA[(TbJdSfn_MsgRecebida)]

    W --> AB[jdnpc-destinataria-titulo-worker-deploy]
    X --> AB

    AB --> AC[(TbJdNpcDst_TituloAct)]
    AB --> AD[(TbJdNpcDst_TituloRjtd)]
```

---

## 1️⃣ Chamada Inicial da API

- **Endpoint**
```http
POST destinataria-titulo-api/jdnpc/destinatario/api/v1/titulo
```

- Serviço responsável:
  - `jdnpc-destinataria-titulo-api-deploy`

---

## 2️⃣ Verificação de Idempotência (Redis)

- Consulta ao Redis para validação da **Chave de Idempotência**
- Serviço:
  - `jdnpc-destinataria-titulo-api-deploy`

### Decisão
- ✅ Chave existente → retorna resposta anterior  
- ❌ Chave inexistente → continua o processamento

---

## 3️⃣ Processamento Inicial

### 📤 Publicação
- Envia mensagem para o **RabbitMQ**
- Grava chave no Redis

### 💾 Persistência Inicial
- Serviço:
  - `jdsfn-persister-worker-deploy`
- Tabela:
```sql
TbJdNpcDst_Titulo
```

---

## 4️⃣ Roteamento e Envio

### 🔀 Roteamento
- Serviço:
  - `jdsfn-router-worker-deploy`
- Fila:
```text
jd.npc.dst.titulo.inclui.command.queue
```

### 📡 Envio para MQ
- Serviço:
  - `jdsfn-send-worker-deploy`
- Fila:
```text
jd.npc.dst.titulo.dda0101.send-result.event.queue
```

### 🗃️ Persistência
- Serviço:
  - `jdsfn-persister-worker-deploy`
- Tabelas:
```sql
TbJdSfn_MsgTransmitida
TbJdNpcDst_TituloTrnsmtd
```

---

## 5️⃣ Processamento pela Nuclea

| Código | Resultado |
|------|----------|
| DDA0101R1 | Sucesso |
| DDA0101E | Erro |

---

## 6️⃣ Recebimento da Resposta

- Serviço:
  - `jdsfn-receive-worker-deploy`
- Fila:
```text
jd.sfn.receive.router.msg.queue
```

### Roteamento do Retorno
- Serviço:
  - `jdsfn-router-worker-deploy`
- Filas:
```text
jd.npc.dst.titulo.dda0101r1.event.queue
jd.npc.dst.titulo.dda0101e.event.queue
jd.sfn.receive.persister.queue
```

---

## 7️⃣ Persistência Final

### Mensagens Recebidas
- Serviço:
  - `jdsfn-persister-worker-deploy`
- Tabela:
```sql
TbJdSfn_MsgRecebida
```

### Status do Título

| Situação | Tabela |
|--------|--------|
| Sucesso | TbJdNpcDst_TituloAct |
| Erro | TbJdNpcDst_TituloRjtd |

---

## ✅ Conclusão

- Controle de idempotência via Redis  
- Mensageria desacoplada  
- Persistência em todas as etapas críticas  
- Tratamento completo de sucesso e erro
