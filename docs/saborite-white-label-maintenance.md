# Saborite — manutenção do painel administrativo white label

## Escopo

Maurício opera a Saborite como parceiro **white label**. A prioridade é a administração e manutenção da plataforma; gestão de clientes fica em segundo momento.

> Este documento registra o estado validado em 02/09/2026. Não contém credenciais, tokens, PINs ou códigos A2F.

## URLs oficiais

- Painel administrativo: https://painel.saborite.com/#/
- Site comercial: https://saborite.com/
- Central de ajuda do cliente: https://meajuda.gitbook.io/central-de-ajuda
- Apoio ao parceiro/suporte: https://meajuda.gitbook.io/apoio-suporte
- Checkout: https://checkout.saborite.com/
- API documentada: https://documenter.getpostman.com/view/8002308/2s9YXiYLjW

## Estrutura confirmada do painel

- Início e Dashboard
- Clientes/estabelecimentos
- Tickets
- Financeiro: Faturas, Faturamento, Tabela de valores e Calculadora
- Integrações: WhatsApp e Asaas
- Links úteis: Guia rápido, Apoio ao parceiro, Apoio ao suporte, Biblioteca de treinamentos, Central de ajuda, Status, Feedback, Comunidade e API
- Downloads: G-Instalador, G-Balança, G-PDV, G-Totem e G-Mesas
- Ajustes

### Ajustes administrativos

A tela de Ajustes contém:

- Dados da empresa/responsável, sistema, e-mail e telefone
- Link de ajuda, Jivochat
- Limites de emissão NFC-e e NF-e
- Token Asaas
- SMTP: autenticação, e-mail, senha, protocolo, host e porta
- Preços dos planos: Essencial, Profissional e Avançado
- Preços e plano mínimo de módulos adicionais
- Site, identificação da logo e identificação Asaas
- PIN de segurança do painel

## Venda: site → checkout → Asaas → loja

Os botões de planos do site foram verificados:

| Nome exibido no site | Preço exibido | Destino |
|---|---:|---|
| Básico | R$ 159,90/mês | `checkout.saborite.com/?plano=essencial&canal=site` |
| Essencial | R$ 229,90/mês | `checkout.saborite.com/?plano=profissional&canal=site` |
| Profissional | R$ 299,92/mês | `checkout.saborite.com/?plano=avançado&canal=site` |

O checkout reconhece os três slugs e apresenta “Continuar para o plano” correspondente. Após o pagamento, informa disponibilidade da loja em aproximadamente 5 minutos.

## Planos no painel

Valores lidos em Ajustes:

| Plano interno | Valor mensal |
|---|---:|
| Essencial | R$ 159,90 |
| Profissional | R$ 229,90 |
| Avançado | R$ 299,92 |

### Diagnóstico da nomenclatura

Os **valores estão alinhados** com os slugs do checkout, mas o nome comercial exibido no site está deslocado uma posição:

- Site “Básico” usa o slug interno `essencial` e o valor do plano interno Essencial.
- Site “Essencial” usa `profissional` e o valor do plano interno Profissional.
- Site “Profissional” usa `avançado` e o valor do plano interno Avançado.

A tela de Ajustes não apresentou campos para renomear os três planos; os campos observados permitem alterar valores. Como o site é gerenciado pela Gama Delivery, **não alterar valores, slugs ou assinaturas apenas para corrigir nomes**.

## Asaas

A documentação do parceiro orienta:

1. No Asaas, criar webhook para cobranças.
2. Usar a URL exibida no painel de clientes.
3. API V3 e envio sequencial.
4. Habilitar `PAYMENT_CONFIRMED`, `PAYMENT_RECEIVED`, `PAYMENT_OVERDUE` e `PAYMENT_REFUNDED`.
5. Gerar uma chave de API no Asaas.
6. Inserir o token em Painel → Ajustes → Token Asaas.
7. Para uma assinatura de loja: abrir loja → Suporte → Módulos → escolher plano → Asaas → Buscar/Gerar → Salvar.

### Regra de segurança financeira

- Não excluir/recriar planos ou assinaturas existentes.
- Não trocar valores de posição no painel.
- Não cancelar e recriar assinatura para corrigir nomenclatura.
- Não alterar descrição no Asaas sem testar em uma loja nova e confirmar como a Gama gera a assinatura.
- Mudanças de preço, plano ou cobrança exigem confirmação explícita e leitura posterior do resultado.

## Manutenção de lojas

Apoio ao parceiro documenta atualização assim:

1. Painel → Clientes.
2. Selecionar a loja.
3. Marcar a caixa à esquerda.
4. Clicar na nuvem.
5. Escolher a versão.
6. Clicar em atualizar.

Manter a mesma linha de versão (estável com estável; beta com beta). A documentação alerta que migrar para beta pode impedir o retorno para estável.

## API Gama Delivery / Saborite

- Produção: `https://api.saborite.com.br/api/`
- Homologação: `https://ambiente.delivery.app/api/`
- Login: `POST /autenticacao/entrar/` com e-mail e senha.
- Requisições autenticadas usam `x-api-key`.
- Limite documentado: 5 requisições/segundo.
- Principais áreas: pedidos, produtos, categorias, usuários, configurações, loja, relatórios, chat, motoboy, mesas, PDV, totem, taxas, notificações e webhooks.

## Procedimento operacional do MRdigital

1. Fazer leitura e identificar o escopo.
2. Conferir documentação da Central de ajuda ou Apoio ao parceiro.
3. Auditar painel e, quando necessário, checkout/Asaas.
4. Não alterar dados sem escopo explícito.
5. Para cobrança, plano, versão beta ou ação destrutiva: pedir confirmação.
6. Após qualquer alteração: ler novamente o alvo e validar o fluxo real.
7. Registrar resultado e data neste documento ou em anotação operacional no Notion.

## Estado atual

- Acesso ao painel administrativo validado.
- Estrutura de menus e Ajustes mapeada.
- Site, checkout e valores do painel auditados.
- Nenhuma configuração, plano, assinatura ou cobrança foi alterada.
- Próxima frente: manutenção administrativa white label; clientes depois.
