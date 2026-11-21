# saas-pay

<div align="center">
  <img src="capa/logo.svg" alt="Arckhé AI" height="48" />
  <p><strong>Arckhé AI</strong></p>
</div>


Kit de referÃªncia **copiar-e-colar** para montar o pagamento de um SaaS/comunidade
em Next.js (App Router) com **Asaas** (PIX, cartÃ£o de crÃ©dito Ã  vista ou parcelado,
boleto) + liberaÃ§Ã£o de acesso automÃ¡tica, e **login sem senha via magic link**
(Supabase Auth). Ã‰ a extraÃ§Ã£o documentada de como isso foi implementado em produÃ§Ã£o
nos projetos INEMA (`pay.inema.pro`, `inema.pro`, `inemaonline`) â€” sem nenhum dado
real de conta, chave ou domÃ­nio.

## ðŸ“– Guia de uso

Guia completo (landing + passo a passo): **https://inematds.github.io/saas-pay/guia/**

NÃ£o Ã© uma lib instalÃ¡vel (`npm install`) â€” Ã© um **guia + cÃ³digo de exemplo** pra vocÃª
adaptar e colar no seu projeto. Tudo em `examples/` Ã© TypeScript funcional, sÃ³ trocando
nomes de tabela/domÃ­nio pelos seus.

## SumÃ¡rio

1. [VisÃ£o geral da arquitetura](#1-visÃ£o-geral-da-arquitetura)
2. [Asaas â€” conceitos](#2-asaas--conceitos)
3. [Cobrar: PIX, cartÃ£o e parcelamento](#3-cobrar-pix-cartÃ£o-e-parcelamento)
4. [Duas formas de vender: Link de Pagamento vs API](#4-duas-formas-de-vender-link-de-pagamento-vs-api)
5. [Webhook: liberar o acesso automaticamente](#5-webhook-liberar-o-acesso-automaticamente)
6. [CÃ¡lculo de vencimento / renovaÃ§Ã£o](#6-cÃ¡lculo-de-vencimento--renovaÃ§Ã£o)
7. [Login sem senha: Magic Link (Supabase)](#7-login-sem-senha-magic-link-supabase)
8. [Setup do zero (checklist)](#8-setup-do-zero-checklist)
9. [VariÃ¡veis de ambiente](#9-variÃ¡veis-de-ambiente)
10. [SeguranÃ§a](#10-seguranÃ§a)
11. [Arquivos deste repo](#11-arquivos-deste-repo)

---

## 1. VisÃ£o geral da arquitetura

```
                      â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
  usuÃ¡rio paga  â”€â”€â”€â–¶  â”‚  Asaas (gateway BR) â”‚  hospeda o checkout,
  (PIX/cartÃ£o/boleto) â”‚                     â”‚  nunca vemos dado de cartÃ£o
                      â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                                â”‚ webhook (POST) quando o status muda
                                â–¼
                      â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
                      â”‚  /api/asaas/webhook â”‚  valida token, calcula
                      â”‚  (sua rota Next.js) â”‚  vencimento, libera acesso
                      â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                                â–¼
                      â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
                      â”‚  seu banco (Supabase â”‚  pessoa + vencimento +
                      â”‚  /Postgres/D1/...)   â”‚  histÃ³rico de pagamentos
                      â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                                â–¼
                      â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
                      â”‚  email de acesso /   â”‚  liberaÃ§Ã£o automÃ¡tica:
                      â”‚  magic link de login  â”‚  sem senha, sem suporte manual
                      â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

PeÃ§as que se repetem em qualquer SaaS que cobra com Asaas:

- **Customer** â€” quem paga (nome, email, CPF/CNPJ).
- **Payment (cobranÃ§a)** â€” valor + vencimento + mÃ©todo, ligado a um customer. Gera
  `invoiceUrl` (a pÃ¡gina de pagamento hospedada).
- **Webhook** â€” o Asaas chama uma URL sua a cada mudanÃ§a de status de pagamento.
- **Assinante** (seu banco) â€” email â†’ data de vencimento do acesso. Ã‰ o que decide
  se alguÃ©m tem acesso ou nÃ£o, hoje.

O dado de cartÃ£o **nunca passa pelo seu servidor** â€” o Asaas hospeda a tela de
pagamento (Link de Pagamento) ou vocÃª usa a API sÃ³ pra criar a cobranÃ§a e redireciona
o cliente pro `invoiceUrl`.

---

## 2. Asaas â€” conceitos

Ambiente Ã© decidido pelo **prefixo da API key**, nÃ£o por uma flag separada:

```ts
// examples/asaas-client.ts
function baseUrl(key: string): string {
  return key.startsWith('$aact_prod') || key.startsWith('$aact_M')
    ? 'https://api.asaas.com/v3'      // produÃ§Ã£o, dinheiro real
    : 'https://sandbox.asaas.com/api/v3'; // sandbox, dinheiro fake
}
```

Toda chamada Ã  API leva a chave no header `access_token` (nÃ£o Ã© Bearer):

```ts
fetch(`${base}/customers/${id}`, {
  headers: { access_token: ASAAS_API_KEY, 'content-type': 'application/json' },
});
```

## 3. Cobrar: PIX, cartÃ£o e parcelamento

Via API (`POST /v3/payments`), o campo que muda tudo Ã© `billingType`:

```ts
// PIX â€” instantÃ¢neo, sem taxa de cartÃ£o
{
  customer: customerId,
  billingType: 'PIX',
  value: 97.00,          // em REAIS (float), nÃ£o centavos
  dueDate: '2026-08-05',
  description: 'Assinatura anual',
  externalReference: `user:${userId}`,
}

// Boleto
{ ...igual, billingType: 'BOLETO' }

// CartÃ£o Ã  vista (1x)
{ ...igual, billingType: 'CREDIT_CARD' }

// CartÃ£o PARCELADO â€” os dois campos extras sÃ£o o que faz o "nx"
{
  ...igual,
  billingType: 'CREDIT_CARD',
  installmentCount: 3,        // em quantas vezes
  totalValue: 291.00,         // valor TOTAL da compra (nÃ£o da parcela)
  // installmentValue: 97.00  // alternativa: valor de CADA parcela, em vez de totalValue
}
```

Regras que mordem:

- `value`/`totalValue` sÃ£o em **reais**, `97.00`, nÃ£o `9700` (centavos). Se seu banco
  guarda em centavos, converta na borda.
- Em parcelamento, mande **ou** `totalValue` **ou** `installmentValue` â€” nunca os dois.
- Parcelamento "sem juros" fica por conta de quem configura a taxa no painel Asaas
  (ConfiguraÃ§Ãµes â†’ Parcelamento) â€” a API nÃ£o tem um campo `interestFree` separado.
- Depois de criado, o pagamento tem `invoiceUrl` (redirecione o cliente pra lÃ¡) e,
  se PIX, dÃ¡ pra buscar o QR Code isolado:

```ts
export async function getPixQrCode(paymentId: string) {
  const res = await fetch(`${base}/payments/${paymentId}/pixQrCode`, {
    headers: { access_token: ASAAS_API_KEY },
  });
  return res.json(); // { encodedImage, payload, expirationDate }
}
```

Ver `examples/asaas-client.ts` para um cliente mÃ­nimo completo (customer + payment
+ PIX QR code).

## 4. Duas formas de vender: Link de Pagamento vs API

| | **A) Link de Pagamento** | **B) Via API** |
|---|---|---|
| CÃ³digo necessÃ¡rio | nenhum | o cliente de `asaas-client.ts` |
| Onde cola | qualquer pÃ¡gina, atÃ© fora do seu app | fluxo logado do seu produto |
| Cria o Payment | Asaas cria na hora que o cliente paga | vocÃª cria com `createPayment()` antes |
| Cupom / conta vinculada no seu banco | nÃ£o, direto | sim (`externalReference`) |
| Libera acesso automaticamente | sim, via webhook (casando por valor/e-mail) | sim, via webhook (casando por `externalReference`) |

**A) Ã© o caminho mais rÃ¡pido pra validar uma oferta**: Painel Asaas â†’ *CobranÃ§as â†’
Link de Pagamento* â†’ cria um link fixo por plano (ex.: mensal R$42, anual R$327) â†’
`https://www.asaas.com/c/XXXXXXXX` â†’ cola num botÃ£o `<a href>`. Zero backend pra
cobrar â€” sÃ³ precisa do webhook pra liberar o acesso (seÃ§Ã£o 5).

**B) Ã© o caminho pra checkout dentro do seu produto**, com valor dinÃ¢mico, cupom,
CPF, e vÃ­nculo direto com o `userId` logado via `externalReference`.

Este repo documenta e dÃ¡ exemplo dos dois, mas o **Link de Pagamento (A)** Ã© o que
recomendamos pra comeÃ§ar â€” Ã© o que foi usado no `pay.inema.pro` (ver
`examples/pricing-page.tsx`).

## 5. Webhook: liberar o acesso automaticamente

Rota `POST /api/asaas/webhook` (App Router, `runtime = 'nodejs'` â€” a service role
key do banco e criptografia de token nÃ£o rodam no Edge):

```ts
// examples/webhook-route.ts (resumo â€” arquivo completo tem tudo)
export async function POST(request: Request) {
  // 1. AutenticaÃ§Ã£o: o Asaas manda um header configurado por vocÃª no painel.
  const token = request.headers.get('asaas-access-token') ?? '';
  if (!tokenValido(token, process.env.ASAAS_WEBHOOK_SECRET!)) {
    return new Response('forbidden', { status: 403 });
  }

  const event = await request.json();
  // 2. SÃ³ agimos em confirmaÃ§Ã£o â€” ignora os outros ~20 tipos de evento do Asaas.
  if (!['PAYMENT_CONFIRMED', 'PAYMENT_RECEIVED'].includes(event.event)) {
    return new Response('ignored', { status: 200 });
  }

  const pagamento = extrairPagamento(event); // { id, valor, customerId, email }

  // 3. IdempotÃªncia: o Asaas PODE reenviar o mesmo evento. Dedup pelo id do pagamento
  //    ANTES de processar de novo â€” senÃ£o vocÃª libera acesso em dobro / soma vencimento 2x.
  if (await jaProcessado(pagamento.id)) return new Response('duplicate', { status: 200 });

  // 4. Email do comprador: vem no payload OU busca via API (customer sÃ³ por id).
  const email = pagamento.email ?? (await getCustomerEmail(pagamento.customerId));
  if (!email) { /* notifica vocÃª mesmo por email/Telegram â€” nÃ£o dÃ¡ pra liberar sem email */ }

  // 5. Grava o pagamento (marcador de idempotÃªncia) e SÃ“ DEPOIS atualiza o vencimento
  //    do assinante â€” nessa ordem, se cair no meio, o retry do Asaas refaz do jeito certo.
  await inserirPagamento({ ...pagamento, email });
  await upsertAssinante({ email, vencimento: calcularNovoVencimento(...) });

  // 6. Email de boas-vindas / acesso liberado (pode incluir o magic link, seÃ§Ã£o 7).
  await enviarEmailDeAcesso(email);

  return new Response('ok', { status: 200 }); // 200 = Asaas nÃ£o re-tenta
}
```

Pontos que **sempre** dÃ£o problema se pulados:

- **Auth do webhook**: comparar o token com `timingSafeEqual`, nÃ£o `===` (evita
  timing attack â€” trivial de fazer certo, entÃ£o faÃ§a).
- **IdempotÃªncia**: o Asaas re-tenta em erro 5xx e tambÃ©m pode duplicar eventos por
  conta prÃ³pria. Sem dedup por `paymentId`, um clique duplo em "confirmar" no painel
  de teste jÃ¡ quebra sua contagem de vencimento.
- **200 mesmo quando "ignoramos"**: eventos que vocÃª nÃ£o trata (ex.: `PAYMENT_OVERDUE`
  se vocÃª nÃ£o usa) devem responder 200, nÃ£o 400/500 â€” senÃ£o o Asaas fica re-tentando
  pra sempre achando que falhou.
- **5xx sÃ³ em erro genuÃ­no**: se algo inesperado quebrar no meio do processamento,
  devolva 500 de propÃ³sito â€” como as escritas sÃ£o idempotentes, o retry automÃ¡tico
  do Asaas conserta sozinho.

## 6. CÃ¡lculo de vencimento / renovaÃ§Ã£o

Regra simples e que cobre re-assinatura/renovaÃ§Ã£o/upgrade sem duplicar tempo:

```ts
// examples/vencimento.ts
export function calcVencimento(valor: number, vencimentoAtual: string|null, hoje: string) {
  const tipo = valor >= LIMIAR_ANUAL ? 'anual' : 'mensal'; // decide pelo valor pago
  const dias = tipo === 'anual' ? 365 : 30;
  // se jÃ¡ tem vencimento FUTURO, soma a partir dele (renovaÃ§Ã£o antecipada nÃ£o perde dias);
  // se venceu ou nunca assinou, conta a partir de hoje.
  const base = vencimentoAtual && vencimentoAtual > hoje ? vencimentoAtual : hoje;
  return { tipo, dataVencimento: addDias(base, dias) };
}
```

Isso Ã© o que permite: pagar de novo 5 dias antes de vencer â†’ nÃ£o perde esses 5 dias;
deixar vencer e pagar depois â†’ conta a partir de hoje, nÃ£o acumula "dÃ­vida" de dias
perdidos.

## 7. Login sem senha: Magic Link (Supabase)

Objetivo: quem pagou recebe um **link que loga direto**, sem criar senha. Dois casos
de uso do mesmo mecanismo:

- **Self-service**: a pessoa esqueceu como entrar / trocou de dispositivo â†’ pede o
  link de novo, ela mesma, informando o email que pagou.
- **Admin**: suporte gera o link manualmente pra alguÃ©m que nÃ£o recebeu o email.

### Gerar o link (servidor, `supabaseAdmin`)

```ts
// examples/magic-link.ts
const { data, error } = await supabaseAdmin.auth.admin.generateLink({
  type: 'magiclink',
  email,
  options: { redirectTo: `${origin}/auth/callback` },
});

// IMPORTANTE: nÃ£o use data.properties.action_link direto (o link hospedado do
// Supabase). Ele cai no fluxo PKCE (exchangeCodeForSession), que exige um
// `code_verifier` guardado no NAVEGADOR que iniciou o login â€” que nÃ£o existe
// quando o link nasce no servidor (Admin API). Use o token_hash cru:
const link = `${origin}/auth/callback?token_hash=${data.properties.hashed_token}&type=magiclink`;
```

### Consumir o link (rota de callback, cliente comum)

Uma Ãºnica rota `/auth/callback` recebe os dois fluxos possÃ­veis do seu app â€”
OAuth normal (Google, `?code=`) e magic link gerado no servidor (`?token_hash=`):

```ts
// examples/auth-callback.ts
export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const tokenHash = searchParams.get('token_hash');
  const type = searchParams.get('type');

  if (tokenHash && type) {
    // magic link gerado no servidor â†’ verifyOtp NÃƒO precisa de code_verifier
    const { error } = await supabase.auth.verifyOtp({ type, token_hash: tokenHash });
    return NextResponse.redirect(error ? `${origin}/entrar?erro=...` : origin);
  }

  const code = searchParams.get('code');
  // login OAuth normal (Google etc.) â†’ esse sim precisa do code_verifier do navegador
  const { error } = await supabase.auth.exchangeCodeForSession(code!);
  return NextResponse.redirect(error ? `${origin}/entrar?erro=...` : origin);
}
```

### Antes de gerar o link: **valide a assinatura**

Nunca gere magic link sÃ³ porque alguÃ©m digitou um email â€” confirme que aquele
email tem acesso pago e vÃ¡lido, senÃ£o vocÃª criou um login gratuito pra qualquer um:

```ts
const vencimento = await getVencimento(email);
if (!isAssinaturaValida(vencimento)) return { ok: false, motivo: vencimento ? 'expirado' : 'nao-assinante' };
```

E aplique **rate limit** na rota self-service (nÃ£o na admin) â€” Ã© a Ãºnica porta desse
fluxo exposta sem autenticaÃ§Ã£o prÃ©via; sem rate limit vira orÃ¡culo de "esse email Ã©
assinante?" pra qualquer um testando emails em massa. Registre toda tentativa
(sucesso/falha/motivo/IP/user-agent) â€” Ã© o log que resolve "nÃ£o recebi o link"
depois.

## 8. Setup do zero (checklist)

**Asaas**
1. Criar conta em [asaas.com](https://www.asaas.com) (ou sandbox em
   `sandbox.asaas.com` pra testar sem dinheiro real).
2. Painel â†’ IntegraÃ§Ãµes â†’ Chave de API â†’ copiar, guardar como `ASAAS_API_KEY`.
3. Painel â†’ IntegraÃ§Ãµes â†’ Webhooks â†’ nova URL apontando pra
   `https://seudominio.com/api/asaas/webhook`, eventos: no mÃ­nimo `PAYMENT_CONFIRMED`
   e `PAYMENT_RECEIVED`. Definir um token/segredo no campo de autenticaÃ§Ã£o â†’ guardar
   como `ASAAS_WEBHOOK_SECRET`.
4. Se for usar Link de Pagamento (caminho A da seÃ§Ã£o 4): Painel â†’ CobranÃ§as â†’ Link
   de Pagamento â†’ criar um por plano/valor.

**Supabase (magic link)**
1. Projeto Supabase jÃ¡ com Auth habilitado (email como provider).
2. Pegar a `SUPABASE_SERVICE_ROLE_KEY` (sÃ³ usada no servidor, nunca no client) â€”
   Ã© o que permite `auth.admin.generateLink`.
3. Configurar Site URL / Redirect URLs no painel Supabase incluindo
   `https://seudominio.com/auth/callback`.

**No seu projeto Next.js**
1. Copiar `examples/asaas-client.ts`, `examples/webhook-route.ts`,
   `examples/vencimento.ts` â†’ adaptar nomes de tabela pro seu schema.
2. Copiar `examples/magic-link.ts`, `examples/auth-callback.ts` se for usar login
   sem senha.
3. Rodar em sandbox primeiro (chave Asaas sem prefixo `$aact_prod`/`$aact_M`),
   confirmar 1 pagamento fake de ponta a ponta (pagar â†’ webhook â†’ banco â†’ email)
   antes de trocar pra chave de produÃ§Ã£o.

## 9. VariÃ¡veis de ambiente

```bash
ASAAS_API_KEY=              # painel Asaas â†’ IntegraÃ§Ãµes â†’ Chave de API
ASAAS_WEBHOOK_SECRET=       # o mesmo valor configurado no webhook do painel
ASAAS_BASE_URL=             # opcional â€” sÃ³ pra forÃ§ar sandbox/prod manualmente

SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=  # sÃ³ no servidor â€” nunca expor no client/browser
```

## 10. SeguranÃ§a

- Dado de cartÃ£o **nunca** passa pelo seu servidor â€” sempre pelo checkout hospedado
  do Asaas (Link de Pagamento) ou pelo SDK deles, nunca um form seu coletando nÃºmero
  de cartÃ£o.
- `ASAAS_WEBHOOK_SECRET` comparado com `timingSafeEqual`, nÃ£o `===`.
- `SUPABASE_SERVICE_ROLE_KEY` sÃ³ em cÃ³digo de servidor (rota API / server action),
  nunca em componente client nem em `NEXT_PUBLIC_*`.
- Magic link: sempre validar assinatura ativa antes de gerar; rate limit na rota
  self-service; registrar tentativas para auditoria.
- Webhook idempotente (dedup por `paymentId`) â€” assume que o Asaas pode reenviar.

## 11. Arquivos deste repo

```
examples/
  asaas-client.ts       cliente mÃ­nimo: customer, payment (PIX/cartÃ£o/parcelado/boleto), PIX QR code
  webhook-route.ts       rota de webhook completa (auth, idempotÃªncia, liberaÃ§Ã£o de acesso)
  vencimento.ts           cÃ¡lculo de vencimento/renovaÃ§Ã£o por valor pago
  magic-link.ts           geraÃ§Ã£o de magic link (Supabase Admin API) com validaÃ§Ã£o de assinatura
  auth-callback.ts        rota de callback que aceita OAuth normal e magic link
  pricing-page.tsx        pÃ¡gina de preÃ§os simples (Link de Pagamento), estilo cards
```

---

ExtraÃ­do e adaptado dos projetos INEMA (`pay.inema.pro`, `inema.pro`, `inemaonline`).
Sem chaves, domÃ­nios ou dados reais â€” troque tudo antes de usar em produÃ§Ã£o.

---

<div align="center">
  <img src="capa/logo.svg" alt="Arckhé AI" height="32" />
  <p><strong>Arckhé AI</strong></p>
</div>