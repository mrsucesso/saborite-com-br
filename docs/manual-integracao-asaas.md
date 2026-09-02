# Manual de Integração Asaas — Clientes, Assinaturas, Cobranças e Webhooks

**Finalidade:** configurar e manter a automação de mensalidades entre o Painel de Clientes e o Asaas.

**Escopo:** cadastro e vinculação de clientes, API Key, Webhook, criação de assinaturas, geração de cobranças, processamento de pagamentos e manutenção automatizada por IA.

**Revisão técnica:** documentação oficial Asaas v3, setembro de 2026.

---

# 1. Visão geral da arquitetura

A integração possui dois sentidos:

```text
PAINEL DE CLIENTES
        │
        │ API Key
        ▼
      ASAAS
        │
        │ Webhook
        ▼
PAINEL DE CLIENTES
```

### Painel → Asaas

A API é utilizada para:

- localizar clientes;
- criar clientes;
- atualizar clientes;
- criar assinaturas;
- consultar assinaturas;
- consultar cobranças;
- alterar assinaturas;
- cancelar ou inativar assinaturas;
- gerenciar Webhooks.

### Asaas → Painel

O Webhook é utilizado para informar:

- pagamento confirmado;
- pagamento recebido;
- pagamento estornado;
- alterações nas assinaturas;
- outros eventos financeiros que venham a ser necessários.

O Asaas recomenda Webhooks para manter sistemas sincronizados, evitando consultas constantes à API.

---

# 2. Identificadores que devem existir no sistema

Este é um dos pontos mais importantes da integração.

Não utilizar **nome do cliente** como chave de relacionamento entre os sistemas.

## 2.1 Cliente no sistema interno

Cada cliente deve possuir pelo menos:

```text
cliente_id
nome
cpf_cnpj
email
asaas_customer_id
```

Exemplo:

```text
cliente_id: 1847
nome: Empresa Exemplo Ltda
cpf_cnpj: 12345678000190
asaas_customer_id: cus_xxxxxxxxx
```

O identificador interno `1847` deve ser enviado ao Asaas como:

```text
externalReference = 1847
```

A documentação do Asaas recomenda utilizar `externalReference` para relacionar o cadastro do Asaas ao identificador da aplicação. Ao mesmo tempo, o `id` retornado pelo Asaas — normalmente `cus_...` — deve ser armazenado no sistema e reutilizado nas operações posteriores.

---

# 3. Cliente cadastrado manualmente no Asaas

Caso o cliente já tenha sido criado manualmente no painel do Asaas, não é necessário excluí-lo nem recadastrá-lo.

O procedimento correto é:

### 3.1 Localizar o cliente

Preferencialmente procurar por:

```text
cpfCnpj
```

ou, quando já existir:

```text
externalReference
```

Endpoint:

```text
GET /v3/customers
```

Filtros importantes:

```text
cpfCnpj
externalReference
email
name
```

A combinação entre documento e referência externa é mais confiável do que procurar somente pelo nome.

---

## 3.2 Recuperar o ID Asaas

O cadastro localizado possuirá um identificador semelhante a:

```text
cus_xxxxxxxxxxxxx
```

Esse é o:

```text
asaas_customer_id
```

O sistema interno deve armazená-lo.

---

## 3.3 Incluir o externalReference

O campo `externalReference` pode não aparecer na edição manual do cliente no painel do Asaas.

Ele pode ser incluído pela API:

```text
PUT /v3/customers/{asaas_customer_id}
```

Payload:

```json
{
  "externalReference": "1847"
}
```

O Asaas permite atualizar somente os campos desejados, sem recriar o cliente.

Resultado:

```text
SISTEMA
cliente_id = 1847

        │
        │ externalReference
        ▼

ASAAS
id = cus_xxxxxxxxx
externalReference = 1847
```

---

# 4. Regra para evitar clientes duplicados

Antes de criar qualquer cliente no Asaas:

### 1. Verificar se `asaas_customer_id` já existe no sistema.

Se existir:

```text
usar cus_...
```

Não criar outro cliente.

### 2. Caso não exista, procurar pelo `externalReference`.

```text
GET /v3/customers?externalReference={cliente_id}
```

### 3. Se não encontrar, procurar pelo CPF/CNPJ.

```text
GET /v3/customers?cpfCnpj={documento}
```

### 4. Se localizar cliente existente:

- salvar `cus_...` no sistema;
- atualizar `externalReference`, se necessário.

### 5. Somente criar um novo cliente quando nenhuma correspondência confiável for encontrada.

O Asaas permite cadastros duplicados, portanto a prevenção de duplicidade é responsabilidade da integração.

---

# 5. Criando um cliente novo pela API

Endpoint:

```text
POST /v3/customers
```

Exemplo:

```json
{
  "name": "Empresa Exemplo Ltda",
  "cpfCnpj": "12345678000190",
  "email": "cliente@exemplo.com",
  "externalReference": "1847"
}
```

Após a criação, armazenar obrigatoriamente:

```text
response.id
```

Exemplo:

```text
cus_000005219613
```

no campo:

```text
asaas_customer_id
```

O ID retornado é utilizado no campo `customer` de cobranças e assinaturas.

---

# 6. Gerando a API Key do Asaas

Procedimento observado no segundo vídeo.

No Asaas:

```text
Minha Conta
→ Integração
→ Chaves de API
→ Gerar chave de API
```

Definir um nome claro:

```text
Painel de Clientes
```

ou:

```text
Integração Produção
```

Depois:

1. clicar em **Avançar**;
2. solicitar código SMS;
3. informar o código;
4. gerar a chave;
5. copiar imediatamente.

A chave completa só é exibida no momento da criação.

A autenticação atual da API utiliza o header:

```text
access_token
```

e não `Authorization: Bearer`. Sandbox e Produção utilizam credenciais e ambientes distintos.

---

# 7. Armazenamento da API Key

Nunca armazenar a API Key:

```text
no código-fonte
no navegador
no frontend
em prompts de IA
em logs
em documentação
```

Utilizar:

```text
variável de ambiente
secret manager
vault
configuração protegida do servidor
```

O próprio Asaas recomenda armazenamento em cofre de segredos ou mecanismo equivalente.

---

# 8. Configurando a API Key no Painel

Conforme o segundo vídeo:

```text
Painel de Clientes
→ Responsável
→ API Keys
→ Token Asaas
```

Colar a chave gerada anteriormente.

Depois:

```text
Salvar
```

O valor deverá aparecer mascarado.

---

# 9. Campo “Identificação Asaas”

No vídeo existe também um campo denominado:

```text
Identificação Asaas
```

Antes de qualquer IA modificar esse campo, deve-se verificar no código da aplicação qual é sua função.

Não assumir automaticamente que ele corresponde a:

```text
customer.id
```

ou:

```text
externalReference
```

A manutenção automatizada deverá primeiro localizar onde esse campo é lido e escrito no código.

---

# 10. Preparar o endpoint do Webhook

Antes de configurar o Webhook no Asaas, o sistema precisa possuir um endpoint público.

Não utilizar:

```text
localhost
127.0.0.1
servidor acessível apenas pela rede interna
```

O primeiro vídeo terminou em erro justamente porque utilizava um endereço `localhost`.

O endpoint deve estar publicamente acessível por HTTPS e aceitar:

```text
POST
Content-Type: application/json
```

A documentação atual exige que a URL esteja preparada para receber requisições do Asaas.

---

# 11. Configurando o Webhook no Asaas

No painel:

```text
Minha Conta
→ Integração
→ Webhooks
→ Adicionar Webhook
```

Configuração observada nos vídeos:

```text
Ativo: Sim
API: v3
Fila de sincronização: Sim
Tipo de envio: Sequencial
```

Utilizar um nome descritivo:

```text
Painel de Clientes — Pagamentos
```

---

# 12. URL do Webhook

Copiar do Painel de Clientes a URL pública destinada à integração.

Cadastrar no campo:

```text
URL do Webhook
```

A URL deverá apontar para o endpoint real do sistema.

---

# 13. Token de autenticação do Webhook

No vídeo original esse campo ficou vazio.

Para Produção, a recomendação atual é utilizar um:

```text
authToken
```

exclusivo para o Webhook.

O Asaas envia esse valor no header:

```text
asaas-access-token
```

A aplicação deve validar esse header antes de processar o evento.

Não utilizar a própria API Key como `authToken`.

O token do Webhook deve possuir entre 32 e 255 caracteres.

---

# 14. Eventos de pagamento utilizados

Nos vídeos foram selecionados:

```text
PAYMENT_CONFIRMED
PAYMENT_RECEIVED
PAYMENT_REFUNDED
```

Significado:

### PAYMENT_CONFIRMED

Pagamento efetuado, mas o saldo ainda não está disponível.

### PAYMENT_RECEIVED

Cobrança efetivamente recebida e saldo disponibilizado.

### PAYMENT_REFUNDED

Cobrança estornada.

Essas definições correspondem aos eventos atuais documentados pelo Asaas.

---

# 15. Cuidado com PAYMENT_CONFIRMED + PAYMENT_RECEIVED

Um mesmo pagamento pode passar sucessivamente por:

```text
PAYMENT_CONFIRMED
        ↓
PAYMENT_RECEIVED
```

Portanto:

**não aumentar o período do cliente duas vezes.**

A lógica correta é atualizar o estado da mesma cobrança.

Exemplo:

```text
payment.id = pay_ABC
```

PAYMENT_CONFIRMED:

```text
fatura pay_ABC → confirmada
```

Depois PAYMENT_RECEIVED:

```text
fatura pay_ABC → recebida
```

e não:

```text
+30 dias
+30 dias novamente
```

A regra de renovação deve ser idempotente.

---

# 16. Idempotência obrigatória

Os Webhooks do Asaas trabalham no modelo:

```text
at least once
```

Isso significa que o mesmo evento pode chegar mais de uma vez.

Cada evento possui:

```text
event.id
```

O sistema deve persistir esse ID.

Fluxo:

```text
Webhook recebido
      ↓
event.id já existe?
   ┌───────┴───────┐
   │               │
  SIM             NÃO
   │               │
HTTP 200       persistir ID
                   │
                   ▼
             processar evento
                   │
                   ▼
                HTTP 200
```

Nunca renovar assinatura novamente porque o Asaas reenviou o mesmo evento.

---

# 17. Eventos de assinatura

Além dos eventos financeiros, o Asaas possui eventos específicos para a assinatura:

```text
SUBSCRIPTION_CREATED
SUBSCRIPTION_UPDATED
SUBSCRIPTION_INACTIVATED
SUBSCRIPTION_DELETED
```

Eles são úteis para sincronizar o estado da assinatura.

Os eventos:

```text
SUBSCRIPTION_*
```

representam o ciclo da assinatura.

Os eventos:

```text
PAYMENT_*
```

representam o ciclo financeiro de cada cobrança gerada pela assinatura.

---

# 18. Criando a assinatura pela aplicação

Fluxo observado no terceiro vídeo:

```text
Cliente
→ Configurações
→ Módulos
→ Asaas
→ Assinatura
→ Buscar/Gerar
```

Se nenhuma assinatura for localizada:

```text
Nenhuma assinatura encontrada
```

o sistema solicita:

```text
Valor da mensalidade
```

No vídeo:

```text
R$ 300,00
```

Depois:

```text
Continuar
```

A plataforma cria ou localiza a assinatura.

---

# 19. Dados necessários para criação da assinatura

Endpoint:

```text
POST /v3/subscriptions
```

Campos fundamentais:

```text
customer
billingType
value
nextDueDate
cycle
description
externalReference
```

Exemplo conceitual:

```json
{
  "customer": "cus_xxxxxxxxx",
  "billingType": "PIX",
  "value": 300,
  "nextDueDate": "AAAA-MM-DD",
  "cycle": "MONTHLY",
  "description": "Plano Avançado",
  "externalReference": "assinatura_cliente_1847"
}
```

O `billingType` apresentado aqui é apenas exemplo; a implementação deve preservar a regra de pagamento definida pelo sistema.

O Asaas exige `customer`, `billingType`, `value`, `nextDueDate` e `cycle` para criação da assinatura e recomenda armazenar o identificador retornado.

---

# 20. Armazenar o ID da assinatura

Depois da criação, o Asaas retorna um identificador:

```text
sub_xxxxxxxxx
```

Esse valor deve ser armazenado no sistema:

```text
asaas_subscription_id
```

Sugestão de estrutura:

```text
cliente_id
asaas_customer_id
asaas_subscription_id
subscription_external_reference
```

Exemplo:

```text
cliente_id: 1847
asaas_customer_id: cus_ABCD
asaas_subscription_id: sub_XYZ
subscription_external_reference: cliente_1847_plano_avancado
```

---

# 21. Conferência da primeira cobrança

Depois de gerar a assinatura:

```text
Financeiro
→ Faturas
```

Deverá existir a primeira cobrança.

No vídeo:

```text
Valor: R$ 300,00
Status: Aguardando pagamento
```

Uma assinatura criada não significa que ela foi paga.

Cada cobrança possui seu próprio ciclo financeiro.

---

# 22. Fluxo completo de renovação

```text
Cliente possui assinatura
        ↓
Asaas gera cobrança
        ↓
Cliente realiza pagamento
        ↓
PAYMENT_CONFIRMED
        ↓
PAYMENT_RECEIVED
        ↓
Webhook chega à plataforma
        ↓
Plataforma identifica payment.id
        ↓
Identifica subscription
        ↓
Identifica customer
        ↓
Identifica cliente interno
        ↓
Atualiza fatura
        ↓
Renova/libera mensalidade
```

---

# 23. Identificação durante o Webhook

A preferência de resolução deve ser:

```text
payment.subscription
        ↓
asaas_subscription_id
        ↓
cliente interno
```

ou:

```text
payment.customer
        ↓
asaas_customer_id
        ↓
cliente interno
```

Não utilizar somente:

```text
nome
empresa
email
CPF/CNPJ
```

como chave operacional depois que os IDs Asaas já estiverem conhecidos.

---

# 24. Consultar cobranças de uma assinatura

Endpoint:

```text
GET /v3/subscriptions/{subscription_id}/payments
```

Útil para:

- conciliação;
- recuperação;
- auditoria;
- reconstrução de estado.

Não deve substituir Webhooks para monitoramento contínuo.

---

# 25. Consultar cobranças gerais

Endpoint:

```text
GET /v3/payments
```

É possível filtrar por:

```text
customer
subscription
status
externalReference
billingType
datas
```

Esse endpoint é útil para conciliação e auditorias financeiras.

---

# 26. Atualizar assinatura

Endpoint:

```text
PUT /v3/subscriptions/{id}
```

Pode ser utilizado para alterar:

- valor;
- próxima data;
- situação;
- outras configurações permitidas pela API.

Por padrão, alterações atingem cobranças futuras.

Cobranças já geradas não são alteradas automaticamente.

Quando suportado, existe a opção:

```text
updatePendingPayments
```

para alterar também cobranças pendentes.

---

# 27. Suspender temporariamente uma assinatura

Para pausar uma recorrência sem excluí-la:

```json
{
  "status": "INACTIVE"
}
```

Isso impede a geração de novas cobranças.

Cobranças já existentes permanecem.

Para reativar:

```text
status = ACTIVE
```

e definir nova:

```text
nextDueDate
```



---

# 28. Cancelar definitivamente uma assinatura

Endpoint:

```text
DELETE /v3/subscriptions/{id}
```

Isso encerra a recorrência.

A documentação atual informa que a remoção também elimina cobranças pendentes ou vencidas ainda pertencentes àquela recorrência. Portanto, esse comando deve ser considerado destrutivo e utilizado somente quando o cancelamento definitivo tiver sido confirmado.

---

# 29. Manutenção de Webhooks via API

A IA ou aplicação pode administrar os Webhooks pela API.

## Listar

```text
GET /v3/webhooks
```

## Consultar

```text
GET /v3/webhooks/{id}
```

## Criar

```text
POST /v3/webhooks
```

## Atualizar

```text
PUT /v3/webhooks/{id}
```

## Remover

```text
DELETE /v3/webhooks/{id}
```

A documentação atual permite alterar URL, eventos, `authToken`, tipo de envio e situação do Webhook.

---

# 30. Antes de criar um Webhook por API

A IA deve primeiro executar:

```text
GET /v3/webhooks
```

e verificar se o Webhook já existe.

Não criar Webhooks duplicados.

Se já existir:

```text
PUT /v3/webhooks/{id}
```

em vez de criar outro.

---

# 31. Monitoramento do Webhook

A automação deverá verificar periodicamente:

```text
enabled
interrupted
url
events
sendType
```

Especialmente:

```text
interrupted
```

Se estiver:

```text
true
```

a fila está interrompida.

O Asaas informa que falhas consecutivas podem interromper a fila e que eventos pendentes permanecem disponíveis por período limitado.

---

# 32. Política para uma IA de manutenção

A IA responsável pela integração deve obedecer esta sequência:

## Antes de qualquer alteração

1. consultar a documentação atual;
2. consultar o recurso atual pela API;
3. comparar estado atual com estado desejado;
4. produzir o plano de alteração;
5. modificar somente o necessário;
6. consultar novamente;
7. validar resultado;
8. registrar a operação.

---

# 33. Operações de leitura permitidas automaticamente

Uma IA pode normalmente executar sem alteração de estado:

```text
GET clientes
GET cliente
GET assinaturas
GET assinatura
GET cobranças
GET cobranças da assinatura
GET webhooks
GET webhook
```

---

# 34. Operações de escrita que devem ser controladas

São alterações de estado:

```text
POST cliente
PUT cliente
POST assinatura
PUT assinatura
DELETE assinatura
POST webhook
PUT webhook
DELETE webhook
```

Para Produção, operações destrutivas devem exigir uma política explícita de autorização.

Particularmente:

```text
DELETE subscription
DELETE webhook
```

---

# 35. MCP oficial do Asaas

O Asaas possui atualmente um **MCP Server oficial voltado à documentação da API**.

Ele permite que assistentes compatíveis:

- pesquisem a documentação;
- localizem endpoints;
- consultem parâmetros;
- consultem schemas;
- vejam estruturas de requisição e resposta;
- gerem exemplos;
- auxiliem na análise de erros;
- e, dependendo da ferramenta e autenticação disponível, auxiliem também na execução de requisições.



**Referência MCP oficial:** [MCP Server oficial do Asaas](https://docs.asaas.com/mcp?utm_source=chatgpt.com)

---

# 36. LLMs.txt do Asaas

O Asaas também fornece um índice específico para agentes e modelos de linguagem.

A IA de manutenção deve consultá-lo quando precisar descobrir documentação atualizada.

**Índice oficial:** [LLMs.txt oficial do Asaas](https://docs.asaas.com/llms.txt?utm_source=chatgpt.com)

A própria documentação informa que as páginas podem ser consultadas em formato Markdown e que a referência possui dados OpenAPI adequados para ferramentas de IA.

---

# 37. Regra para uso do MCP pela IA

O MCP oficial do Asaas deve ser usado principalmente como:

```text
FONTE DA VERDADE DA DOCUMENTAÇÃO
```

Antes de implementar ou alterar uma chamada:

```text
IA
 ↓
MCP Asaas
 ↓
confirma endpoint atual
 ↓
confirma campos
 ↓
confirma enums
 ↓
confirma comportamento
 ↓
só então altera o sistema
```

Não depender de memória antiga do modelo.

---

# 38. Credenciais e MCP

Nunca colocar no prompt:

```text
API Key de produção
Webhook authToken
senha
token de banco
secret de infraestrutura
```

A credencial deve ficar na camada de execução:

```text
IA
 ↓
Tool/MCP interno
 ↓
Secret Manager
 ↓
Asaas API
```

e não:

```text
IA recebe a API Key em texto
```

A documentação oficial do MCP do Asaas orienta explicitamente que a API Key não seja fornecida diretamente ao prompt do assistente.

---

# 39. Arquitetura recomendada para agente de manutenção

```text
                 ┌────────────────────┐
                 │        IA          │
                 └─────────┬──────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
      MCP DOCUMENTAÇÃO            MCP/TOOLS INTERNOS
           ASAAS                        │
              │                         │
              ▼                         ▼
        documentação              Backend protegido
        OpenAPI atual                   │
                                        │
                                        ▼
                                   Asaas API
                                        │
                                        ▼
                                    Produção
```

Assim existem duas funções separadas:

### MCP Asaas

Responde:

> “Como funciona atualmente a API?”

### MCP interno / ferramentas da aplicação

Executa:

> “Consulte o cliente 1847.”

> “Atualize esta assinatura.”

> “Verifique se o Webhook está interrompido.”

---

# 40. Ferramentas ideais para um MCP interno

Um MCP específico da plataforma deveria expor ferramentas de alto nível.

Exemplos:

```text
asaas_get_customer
asaas_link_existing_customer
asaas_create_customer
asaas_get_subscription
asaas_create_subscription
asaas_update_subscription
asaas_pause_subscription
asaas_resume_subscription
asaas_list_subscription_payments
asaas_get_payment
asaas_list_webhooks
asaas_check_webhook_health
asaas_update_webhook
```

Evitar oferecer para a IA apenas:

```text
http_request(method, url, body)
```

em Produção.

Ferramentas semânticas reduzem o risco de a IA chamar endpoints errados.

---

# 41. Ferramenta específica para vincular cliente manual

Sugestão:

```text
asaas_link_existing_customer(
    cliente_id,
    cpf_cnpj
)
```

Internamente:

```text
1. consulta banco local
2. verifica asaas_customer_id
3. procura Asaas pelo CPF/CNPJ
4. exige correspondência única
5. obtém cus_...
6. PUT externalReference = cliente_id
7. salva cus_... localmente
8. consulta novamente
9. retorna resultado
```

Essa ferramenta resolve especificamente os clientes que já foram cadastrados manualmente no Asaas.

---

# 42. Ferramenta “Buscar/Gerar Assinatura”

A funcionalidade mostrada no terceiro vídeo deveria seguir logicamente:

```text
asaas_get_or_create_subscription(cliente_id)
```

Fluxo:

```text
cliente_id
    ↓
buscar cliente local
    ↓
possui asaas_customer_id?
 ┌─────┴─────┐
 │           │
SIM         NÃO
 │           │
 │       vincular/criar cliente
 │           │
 └─────┬─────┘
       ↓
possui asaas_subscription_id?
 ┌─────┴─────┐
 │           │
SIM         NÃO
 │           │
consultar   procurar assinatura
Asaas       existente
 │           │
 │       ┌───┴────┐
 │       │        │
 │    achou     não achou
 │       │        │
 │       │      criar
 │       │        │
 └───────┴────────┘
          ↓
armazenar sub_...
```

---

# 43. Nunca usar “criar” como primeira operação

Para clientes e assinaturas, o padrão deverá ser:

```text
GET
 ↓
localizar
 ↓
reutilizar
```

e somente então:

```text
POST
```

caso o recurso realmente não exista.

---

# 44. Auditoria recomendada

Manter uma tabela ou log como:

```text
integration_audit
```

Campos sugeridos:

```text
id
timestamp
actor
action
entity_type
local_entity_id
asaas_entity_id
request_summary
response_status
success
error
```

O campo:

```text
actor
```

poderia registrar:

```text
usuario
sistema
agente_ia
webhook
```

---

# 45. Tabela para idempotência de Webhooks

Criar algo semelhante a:

```text
asaas_webhook_events
```

com:

```text
event_id UNIQUE
event_type
payment_id
subscription_id
received_at
processed_at
status
payload_hash
```

`event_id` deve possuir restrição de unicidade.

---

# 46. Regra de renovação

A regra não deve ser:

```text
recebeu Webhook
→ acrescentar 30 dias
```

Deve ser:

```text
recebeu Webhook
        ↓
validar origem
        ↓
persistir event.id
        ↓
identificar payment.id
        ↓
identificar subscription
        ↓
identificar cliente
        ↓
consultar estado atual
        ↓
aplicar transição
        ↓
registrar resultado
```

---

# 47. Estorno

Ao receber:

```text
PAYMENT_REFUNDED
```

não simplesmente subtrair 30 dias automaticamente.

Primeiro identificar:

```text
qual cobrança foi estornada
qual competência
qual assinatura
qual direito foi concedido por aquela cobrança
```

Depois aplicar a regra de negócio apropriada.

Isso evita remover acesso incorretamente quando existem:

- pagamentos posteriores;
- ajustes manuais;
- créditos;
- pagamentos antecipados.

---

# 48. Rotina de conciliação

Mesmo utilizando Webhooks, recomenda-se possuir uma rotina de reconciliação.

Por exemplo:

```text
1x por dia
```

ou conforme necessidade operacional.

A rotina poderia:

1. selecionar assinaturas ativas locais;
2. consultar divergências relevantes no Asaas;
3. verificar pagamentos recentes;
4. identificar Webhook perdido;
5. reparar somente divergências comprovadas;
6. gerar relatório.

Webhooks continuam sendo o mecanismo principal; consultas são utilizadas para recuperação e conciliação.

---

# 49. Checklist de implantação

## Cliente

- [ ] possui ID interno;
- [ ] possui CPF/CNPJ;
- [ ] possui `externalReference`;
- [ ] possui `asaas_customer_id`;
- [ ] não existe duplicidade no Asaas.

## API

- [ ] API Key criada;
- [ ] credencial armazenada como segredo;
- [ ] ambiente correto;
- [ ] chamada autenticada testada.

## Webhook

- [ ] URL pública HTTPS;
- [ ] endpoint aceita POST;
- [ ] `authToken` configurado;
- [ ] header `asaas-access-token` validado;
- [ ] Webhook ativo;
- [ ] fila não interrompida;
- [ ] eventos corretos selecionados;
- [ ] idempotência implementada.

## Assinatura

- [ ] `customer = cus_...`;
- [ ] valor correto;
- [ ] ciclo correto;
- [ ] vencimento correto;
- [ ] `externalReference` definido;
- [ ] `sub_...` armazenado no sistema.

## Cobrança

- [ ] primeira cobrança criada;
- [ ] payment ID armazenado;
- [ ] pagamento teste executado;
- [ ] Webhook recebido;
- [ ] assinatura renovada uma única vez.

---

# 50. Teste de homologação final

Executar em ambiente de teste:

```text
Criar/vincular cliente
        ↓
Criar assinatura
        ↓
Verificar sub_...
        ↓
Verificar cobrança
        ↓
Simular/realizar pagamento
        ↓
Receber Webhook
        ↓
Validar event.id
        ↓
Atualizar cobrança
        ↓
Atualizar cliente
        ↓
Confirmar acesso
```

Depois repetir propositalmente o mesmo Webhook.

Resultado esperado:

```text
primeira entrega → processada
segunda entrega → HTTP 200, sem duplicar renovação
```

---

# 51. Referências oficiais prioritárias para a IA

A IA responsável pela manutenção deverá consultar, nesta ordem:

### 1. MCP oficial do Asaas
[Abrir MCP oficial do Asaas](https://docs.asaas.com/mcp?utm_source=chatgpt.com)

### 2. Índice LLMs.txt
[Abrir índice para agentes de IA](https://docs.asaas.com/llms.txt?utm_source=chatgpt.com)

### 3. Autenticação
Consultar sempre antes de diagnosticar erros `401`.

### 4. Clientes
Criação, consulta e atualização.

### 5. Assinaturas
Criação, consulta, alteração e cancelamento.

### 6. Webhooks
Criação, atualização, eventos, segurança e idempotência.

---

# 52. Instrução-base para outra IA

Ao receber este manual, a IA deverá trabalhar segundo esta regra:

> Consulte a documentação atual do Asaas através do MCP oficial ou `llms.txt` antes de alterar qualquer integração. Não presuma que endpoints, enums ou schemas antigos continuam válidos.
>
> Para identificar clientes, utilize prioritariamente o ID interno, `externalReference` e `asaas_customer_id`. Não relacione cadastros apenas pelo nome.
>
> Antes de criar clientes, assinaturas ou Webhooks, consulte se o recurso já existe.
>
> Toda operação de escrita deverá ser seguida por uma consulta de verificação.
>
> API Keys e tokens devem ser obtidos exclusivamente pelo mecanismo seguro da infraestrutura e nunca inseridos no prompt, código-fonte, frontend ou logs.
>
> No processamento de Webhooks, valide o `asaas-access-token`, persista `event.id` antes da regra de negócio e implemente idempotência.
>
> Trate `PAYMENT_CONFIRMED`, `PAYMENT_RECEIVED` e outros estados como transições da mesma cobrança identificada por `payment.id`; nunca renove a mensalidade duas vezes pelo mesmo pagamento.
>
> Antes de comandos destrutivos, especialmente exclusão de assinatura ou Webhook, consulte o estado atual e exija autorização conforme a política da aplicação.