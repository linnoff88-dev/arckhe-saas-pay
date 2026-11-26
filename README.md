ï»¿# saas-pay

<div align="center">
  <img src="capa/logo.svg" alt="Arckhé AI" height="48" />
  <p><strong>Arckhé AI</strong></p>
</div>


Kit de referêÂªncia **copiar-e-colar** para montar o pagamento de um SaaS/comunidade
em Next.js (App Router) com **Asaas** (PIX, cartêÂ£o de crêÂ©dito êÂ  vista ou parcelado,
boleto) + liberaêÂ§êÂ£o de acesso automêÂ¡tica, e **login sem senha via magic link**
(Supabase Auth). êâ° a extraêÂ§êÂ£o documentada de como isso foi implementado em produêÂ§êÂ£o
nos projetos INEMA (`pay.inema.pro`, `inema.pro`, `inemaonline`) ê¢â¬â sem nenhum dado
real de conta, chave ou domêÂ­nio.

## ê°Å¸ââ Guia de uso

Guia completo (landing + passo a passo): **https://inematds.github.io/saas-pay/guia/**

NêÂ£o êÂ© uma lib instalêÂ¡vel (`npm install`) ê¢â¬â êÂ© um **guia + cêÂ³digo de exemplo** pra vocêÂª
adaptar e colar no seu projeto. Tudo em `examples/` êÂ© TypeScript funcional, sêÂ³ trocando
nomes de tabela/domêÂ­nio pelos seus.

## SumêÂ¡rio

1. [VisêÂ£o geral da arquitetura](#1-visêÂ£o-geral-da-arquitetura)
2. [Asaas ê¢â¬â conceitos](#2-asaas--conceitos)
3. [Cobrar: PIX, cartêÂ£o e parcelamento](#3-cobrar-pix-cartêÂ£o-e-parcelamento)
4. [Duas formas de vender: Link de Pagamento vs API](#4-duas-formas-de-vender-link-de-pagamento-vs-api)
5. [Webhook: liberar o acesso automaticamente](#5-webhook-liberar-o-acesso-automaticamente)
6. [CêÂ¡lculo de vencimento / renovaêÂ§êÂ£o](#6-cêÂ¡lculo-de-vencimento--renovaêÂ§êÂ£o)
7. [Login sem senha: Magic Link (Supabase)](#7-login-sem-senha-magic-link-supabase)
8. [Setup do zero (checklist)](#8-setup-do-zero-checklist)
9. [VariêÂ¡veis de ambiente](#9-variêÂ¡veis-de-ambiente)
10. [SeguranêÂ§a](#10-seguranêÂ§a)
11. [Arquivos deste repo](#11-arquivos-deste-repo)

---

## 1. VisêÂ£o geral da arquitetura

```
                      ê¢âÅê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢âÂ
  usuêÂ¡rio paga  ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢âÂ¶  ê¢ââ  Asaas (gateway BR) ê¢ââ  hospeda o checkout,
  (PIX/cartêÂ£o/boleto) ê¢ââ                     ê¢ââ  nunca vemos dado de cartêÂ£o
                      ê¢ââê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢âÂ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢âË
                                ê¢ââ webhook (POST) quando o status muda
                                ê¢âÂ¼
                      ê¢âÅê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢âÂ
                      ê¢ââ  /api/asaas/webhook ê¢ââ  valida token, calcula
                      ê¢ââ  (sua rota Next.js) ê¢ââ  vencimento, libera acesso
                      ê¢ââê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢âÂ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢âË
                                ê¢âÂ¼
                      ê¢âÅê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢âÂ
                      ê¢ââ  seu banco (Supabase ê¢ââ  pessoa + vencimento +
                      ê¢ââ  /Postgres/D1/...)   ê¢ââ  histêÂ³rico de pagamentos
                      ê¢ââê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢âÂ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢âË
                                ê¢âÂ¼
                      ê¢âÅê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢âÂ
                      ê¢ââ  email de acesso /   ê¢ââ  liberaêÂ§êÂ£o automêÂ¡tica:
                      ê¢ââ  magic link de login  ê¢ââ  sem senha, sem suporte manual
                      ê¢ââê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢ââ¬ê¢âË
```

PeêÂ§as que se repetem em qualquer SaaS que cobra com Asaas:

- **Customer** ê¢â¬â quem paga (nome, email, CPF/CNPJ).
- **Payment (cobranêÂ§a)** ê¢â¬â valor + vencimento + mêÂ©todo, ligado a um customer. Gera
  `invoiceUrl` (a pêÂ¡gina de pagamento hospedada).
- **Webhook** ê¢â¬â o Asaas chama uma URL sua a cada mudanêÂ§a de status de pagamento.
- **Assinante** (seu banco) ê¢â¬â email ê¢â â data de vencimento do acesso. êâ° o que decide
  se alguêÂ©m tem acesso ou nêÂ£o, hoje.

O dado de cartêÂ£o **nunca passa pelo seu servidor** ê¢â¬â o Asaas hospeda a tela de
pagamento (Link de Pagamento) ou vocêÂª usa a API sêÂ³ pra criar a cobranêÂ§a e redireciona
o cliente pro `invoiceUrl`.

---

## 2. Asaas ê¢â¬â conceitos

Ambiente êÂ© decidido pelo **prefixo da API key**, nêÂ£o por uma flag separada:

```ts
// examples/asaas-client.ts
function baseUrl(key: string): string {
  return key.startsWith('$aact_prod') || key.startsWith('$aact_M')
    ? 'https://api.asaas.com/v3'      // produêÂ§êÂ£o, dinheiro real
    : 'https://sandbox.asaas.com/api/v3'; // sandbox, dinheiro fake
}
```

Toda chamada êÂ  API leva a chave no header `access_token` (nêÂ£o êÂ© Bearer):

```ts
fetch(`${base}/customers/${id}`, {
  headers: { access_token: ASAAS_API_KEY, 'content-type': 'application/json' },
});
```

## 3. Cobrar: PIX, cartêÂ£o e parcelamento

Via API (`POST /v3/payments`), o campo que muda tudo êÂ© `billingType`:

```ts
// PIX ê¢â¬â instantêÂ¢neo, sem taxa de cartêÂ£o
{
  customer: customerId,
  billingType: 'PIX',
  value: 97.00,          // em REAIS (float), nêÂ£o centavos
  dueDate: '2026-08-05',
  description: 'Assinatura anual',
  externalReference: `user:${userId}`,
}

// Boleto
{ ...igual, billingType: 'BOLETO' }

// CartêÂ£o êÂ  vista (1x)
{ ...igual, billingType: 'CREDIT_CARD' }

// CartêÂ£o PARCELADO ê¢â¬â os dois campos extras sêÂ£o o que faz o "nx"
{
  ...igual,
  billingType: 'CREDIT_CARD',
  installmentCount: 3,        // em quantas vezes
  totalValue: 291.00,         // valor TOTAL da compra (nêÂ£o da parcela)
  // installmentValue: 97.00  // alternativa: valor de CADA parcela, em vez de totalValue
}
```

Regras que mordem:

- `value`/`totalValue` sêÂ£o em **reais**, `97.00`, nêÂ£o `9700` (centavos). Se seu banco
  guarda em centavos, converta na borda.
- Em parcelamento, mande **ou** `totalValue` **ou** `installmentValue` ê¢â¬â nunca os dois.
- Parcelamento "sem juros" fica por conta de quem configura a taxa no painel Asaas
  (ConfiguraêÂ§êÂµes ê¢â â Parcelamento) ê¢â¬â a API nêÂ£o tem um campo `interestFree` separado.
- Depois de criado, o pagamento tem `invoiceUrl` (redirecione o cliente pra lêÂ¡) e,
  se PIX, dêÂ¡ pra buscar o QR Code isolado:

```ts
export async function getPixQrCode(paymentId: string) {
  const res = await fetch(`${base}/payments/${paymentId}/pixQrCode`, {
    headers: { access_token: ASAAS_API_KEY },
  });
  return res.json(); // { encodedImage, payload, expirationDate }
}
```

Ver `examples/asaas-client.ts` para um cliente mêÂ­nimo completo (customer + payment
+ PIX QR code).

## 4. Duas formas de vender: Link de Pagamento vs API

| | **A) Link de Pagamento** | **B) Via API** |
|---|---|---|
| CêÂ³digo necessêÂ¡rio | nenhum | o cliente de `asaas-client.ts` |
| Onde cola | qualquer pêÂ¡gina, atêÂ© fora do seu app | fluxo logado do seu produto |
| Cria o Payment | Asaas cria na hora que o cliente paga | vocêÂª cria com `createPayment()` antes |
| Cupom / conta vinculada no seu banco | nêÂ£o, direto | sim (`externalReference`) |
| Libera acesso automaticamente | sim, via webhook (casando por valor/e-mail) | sim, via webhook (casando por `externalReference`) |

**A) êÂ© o caminho mais rêÂ¡pido pra validar uma oferta**: Painel Asaas ê¢â â *CobranêÂ§as ê¢â â
Link de Pagamento* ê¢â â cria um link fixo por plano (ex.: mensal R$42, anual R$327) ê¢â â
`https://www.asaas.com/c/XXXXXXXX` ê¢â â cola num botêÂ£o `<a href>`. Zero backend pra
cobrar ê¢â¬â sêÂ³ precisa do webhook pra liberar o acesso (seêÂ§êÂ£o 5).

**B) êÂ© o caminho pra checkout dentro do seu produto**, com valor dinêÂ¢mico, cupom,
CPF, e vêÂ­nculo direto com o `userId` logado via `externalReference`.

Este repo documenta e dêÂ¡ exemplo dos dois, mas o **Link de Pagamento (A)** êÂ© o que
recomendamos pra comeêÂ§ar ê¢â¬â êÂ© o que foi usado no `pay.inema.pro` (ver
`examples/pricing-page.tsx`).

## 5. Webhook: liberar o acesso automaticamente

Rota `POST /api/asaas/webhook` (App Router, `runtime = 'nodejs'` ê¢â¬â a service role
key do banco e criptografia de token nêÂ£o rodam no Edge):

```ts
// examples/webhook-route.ts (resumo ê¢â¬â arquivo completo tem tudo)
export async function POST(request: Request) {
  // 1. AutenticaêÂ§êÂ£o: o Asaas manda um header configurado por vocêÂª no painel.
  const token = request.headers.get('asaas-access-token') ?? '';
  if (!tokenValido(token, process.env.ASAAS_WEBHOOK_SECRET!)) {
    return new Response('forbidden', { status: 403 });
  }

  const event = await request.json();
  // 2. SêÂ³ agimos em confirmaêÂ§êÂ£o ê¢â¬â ignora os outros ~20 tipos de evento do Asaas.
  if (!['PAYMENT_CONFIRMED', 'PAYMENT_RECEIVED'].includes(event.event)) {
    return new Response('ignored', { status: 200 });
  }

  const pagamento = extrairPagamento(event); // { id, valor, customerId, email }

  // 3. IdempotêÂªncia: o Asaas PODE reenviar o mesmo evento. Dedup pelo id do pagamento
  //    ANTES de processar de novo ê¢â¬â senêÂ£o vocêÂª libera acesso em dobro / soma vencimento 2x.
  if (await jaProcessado(pagamento.id)) return new Response('duplicate', { status: 200 });

  // 4. Email do comprador: vem no payload OU busca via API (customer sêÂ³ por id).
  const email = pagamento.email ?? (await getCustomerEmail(pagamento.customerId));
  if (!email) { /* notifica vocêÂª mesmo por email/Telegram ê¢â¬â nêÂ£o dêÂ¡ pra liberar sem email */ }

  // 5. Grava o pagamento (marcador de idempotêÂªncia) e Sêâ DEPOIS atualiza o vencimento
  //    do assinante ê¢â¬â nessa ordem, se cair no meio, o retry do Asaas refaz do jeito certo.
  await inserirPagamento({ ...pagamento, email });
  await upsertAssinante({ email, vencimento: calcularNovoVencimento(...) });

  // 6. Email de boas-vindas / acesso liberado (pode incluir o magic link, seêÂ§êÂ£o 7).
  await enviarEmailDeAcesso(email);

  return new Response('ok', { status: 200 }); // 200 = Asaas nêÂ£o re-tenta
}
```

Pontos que **sempre** dêÂ£o problema se pulados:

- **Auth do webhook**: comparar o token com `timingSafeEqual`, nêÂ£o `===` (evita
  timing attack ê¢â¬â trivial de fazer certo, entêÂ£o faêÂ§a).
- **IdempotêÂªncia**: o Asaas re-tenta em erro 5xx e tambêÂ©m pode duplicar eventos por
  conta prêÂ³pria. Sem dedup por `paymentId`, um clique duplo em "confirmar" no painel
  de teste jêÂ¡ quebra sua contagem de vencimento.
- **200 mesmo quando "ignoramos"**: eventos que vocêÂª nêÂ£o trata (ex.: `PAYMENT_OVERDUE`
  se vocêÂª nêÂ£o usa) devem responder 200, nêÂ£o 400/500 ê¢â¬â senêÂ£o o Asaas fica re-tentando
  pra sempre achando que falhou.
- **5xx sêÂ³ em erro genuêÂ­no**: se algo inesperado quebrar no meio do processamento,
  devolva 500 de propêÂ³sito ê¢â¬â como as escritas sêÂ£o idempotentes, o retry automêÂ¡tico
  do Asaas conserta sozinho.

## 6. CêÂ¡lculo de vencimento / renovaêÂ§êÂ£o

Regra simples e que cobre re-assinatura/renovaêÂ§êÂ£o/upgrade sem duplicar tempo:

```ts
// examples/vencimento.ts
export function calcVencimento(valor: number, vencimentoAtual: string|null, hoje: string) {
  const tipo = valor >= LIMIAR_ANUAL ? 'anual' : 'mensal'; // decide pelo valor pago
  const dias = tipo === 'anual' ? 365 : 30;
  // se jêÂ¡ tem vencimento FUTURO, soma a partir dele (renovaêÂ§êÂ£o antecipada nêÂ£o perde dias);
  // se venceu ou nunca assinou, conta a partir de hoje.
  const base = vencimentoAtual && vencimentoAtual > hoje ? vencimentoAtual : hoje;
  return { tipo, dataVencimento: addDias(base, dias) };
}
```

Isso êÂ© o que permite: pagar de novo 5 dias antes de vencer ê¢â â nêÂ£o perde esses 5 dias;
deixar vencer e pagar depois ê¢â â conta a partir de hoje, nêÂ£o acumula "dêÂ­vida" de dias
perdidos.

## 7. Login sem senha: Magic Link (Supabase)

Objetivo: quem pagou recebe um **link que loga direto**, sem criar senha. Dois casos
de uso do mesmo mecanismo:

- **Self-service**: a pessoa esqueceu como entrar / trocou de dispositivo ê¢â â pede o
  link de novo, ela mesma, informando o email que pagou.
- **Admin**: suporte gera o link manualmente pra alguêÂ©m que nêÂ£o recebeu o email.

### Gerar o link (servidor, `supabaseAdmin`)

```ts
// examples/magic-link.ts
const { data, error } = await supabaseAdmin.auth.admin.generateLink({
  type: 'magiclink',
  email,
  options: { redirectTo: `${origin}/auth/callback` },
});

// IMPORTANTE: nêÂ£o use data.properties.action_link direto (o link hospedado do
// Supabase). Ele cai no fluxo PKCE (exchangeCodeForSession), que exige um
// `code_verifier` guardado no NAVEGADOR que iniciou o login ê¢â¬â que nêÂ£o existe
// quando o link nasce no servidor (Admin API). Use o token_hash cru:
const link = `${origin}/auth/callback?token_hash=${data.properties.hashed_token}&type=magiclink`;
```

### Consumir o link (rota de callback, cliente comum)

Uma êÂºnica rota `/auth/callback` recebe os dois fluxos possêÂ­veis do seu app ê¢â¬â
OAuth normal (Google, `?code=`) e magic link gerado no servidor (`?token_hash=`):

```ts
// examples/auth-callback.ts
export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const tokenHash = searchParams.get('token_hash');
  const type = searchParams.get('type');

  if (tokenHash && type) {
    // magic link gerado no servidor ê¢â â verifyOtp NêÆO precisa de code_verifier
    const { error } = await supabase.auth.verifyOtp({ type, token_hash: tokenHash });
    return NextResponse.redirect(error ? `${origin}/entrar?erro=...` : origin);
  }

  const code = searchParams.get('code');
  // login OAuth normal (Google etc.) ê¢â â esse sim precisa do code_verifier do navegador
  const { error } = await supabase.auth.exchangeCodeForSession(code!);
  return NextResponse.redirect(error ? `${origin}/entrar?erro=...` : origin);
}
```

### Antes de gerar o link: **valide a assinatura**

Nunca gere magic link sêÂ³ porque alguêÂ©m digitou um email ê¢â¬â confirme que aquele
email tem acesso pago e vêÂ¡lido, senêÂ£o vocêÂª criou um login gratuito pra qualquer um:

```ts
const vencimento = await getVencimento(email);
if (!isAssinaturaValida(vencimento)) return { ok: false, motivo: vencimento ? 'expirado' : 'nao-assinante' };
```

E aplique **rate limit** na rota self-service (nêÂ£o na admin) ê¢â¬â êÂ© a êÂºnica porta desse
fluxo exposta sem autenticaêÂ§êÂ£o prêÂ©via; sem rate limit vira orêÂ¡culo de "esse email êÂ©
assinante?" pra qualquer um testando emails em massa. Registre toda tentativa
(sucesso/falha/motivo/IP/user-agent) ê¢â¬â êÂ© o log que resolve "nêÂ£o recebi o link"
depois.

## 8. Setup do zero (checklist)

**Asaas**
1. Criar conta em [asaas.com](https://www.asaas.com) (ou sandbox em
   `sandbox.asaas.com` pra testar sem dinheiro real).
2. Painel ê¢â â IntegraêÂ§êÂµes ê¢â â Chave de API ê¢â â copiar, guardar como `ASAAS_API_KEY`.
3. Painel ê¢â â IntegraêÂ§êÂµes ê¢â â Webhooks ê¢â â nova URL apontando pra
   `https://seudominio.com/api/asaas/webhook`, eventos: no mêÂ­nimo `PAYMENT_CONFIRMED`
   e `PAYMENT_RECEIVED`. Definir um token/segredo no campo de autenticaêÂ§êÂ£o ê¢â â guardar
   como `ASAAS_WEBHOOK_SECRET`.
4. Se for usar Link de Pagamento (caminho A da seêÂ§êÂ£o 4): Painel ê¢â â CobranêÂ§as ê¢â â Link
   de Pagamento ê¢â â criar um por plano/valor.

**Supabase (magic link)**
1. Projeto Supabase jêÂ¡ com Auth habilitado (email como provider).
2. Pegar a `SUPABASE_SERVICE_ROLE_KEY` (sêÂ³ usada no servidor, nunca no client) ê¢â¬â
   êÂ© o que permite `auth.admin.generateLink`.
3. Configurar Site URL / Redirect URLs no painel Supabase incluindo
   `https://seudominio.com/auth/callback`.

**No seu projeto Next.js**
1. Copiar `examples/asaas-client.ts`, `examples/webhook-route.ts`,
   `examples/vencimento.ts` ê¢â â adaptar nomes de tabela pro seu schema.
2. Copiar `examples/magic-link.ts`, `examples/auth-callback.ts` se for usar login
   sem senha.
3. Rodar em sandbox primeiro (chave Asaas sem prefixo `$aact_prod`/`$aact_M`),
   confirmar 1 pagamento fake de ponta a ponta (pagar ê¢â â webhook ê¢â â banco ê¢â â email)
   antes de trocar pra chave de produêÂ§êÂ£o.

## 9. VariêÂ¡veis de ambiente

```bash
ASAAS_API_KEY=              # painel Asaas ê¢â â IntegraêÂ§êÂµes ê¢â â Chave de API
ASAAS_WEBHOOK_SECRET=       # o mesmo valor configurado no webhook do painel
ASAAS_BASE_URL=             # opcional ê¢â¬â sêÂ³ pra forêÂ§ar sandbox/prod manualmente

SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=  # sêÂ³ no servidor ê¢â¬â nunca expor no client/browser
```

## 10. SeguranêÂ§a

- Dado de cartêÂ£o **nunca** passa pelo seu servidor ê¢â¬â sempre pelo checkout hospedado
  do Asaas (Link de Pagamento) ou pelo SDK deles, nunca um form seu coletando nêÂºmero
  de cartêÂ£o.
- `ASAAS_WEBHOOK_SECRET` comparado com `timingSafeEqual`, nêÂ£o `===`.
- `SUPABASE_SERVICE_ROLE_KEY` sêÂ³ em cêÂ³digo de servidor (rota API / server action),
  nunca em componente client nem em `NEXT_PUBLIC_*`.
- Magic link: sempre validar assinatura ativa antes de gerar; rate limit na rota
  self-service; registrar tentativas para auditoria.
- Webhook idempotente (dedup por `paymentId`) ê¢â¬â assume que o Asaas pode reenviar.

## 11. Arquivos deste repo

```
examples/
  asaas-client.ts       cliente mêÂ­nimo: customer, payment (PIX/cartêÂ£o/parcelado/boleto), PIX QR code
  webhook-route.ts       rota de webhook completa (auth, idempotêÂªncia, liberaêÂ§êÂ£o de acesso)
  vencimento.ts           cêÂ¡lculo de vencimento/renovaêÂ§êÂ£o por valor pago
  magic-link.ts           geraêÂ§êÂ£o de magic link (Supabase Admin API) com validaêÂ§êÂ£o de assinatura
  auth-callback.ts        rota de callback que aceita OAuth normal e magic link
  pricing-page.tsx        pêÂ¡gina de preêÂ§os simples (Link de Pagamento), estilo cards
```

---

ExtraêÂ­do e adaptado dos projetos INEMA (`pay.inema.pro`, `inema.pro`, `inemaonline`).
Sem chaves, domêÂ­nios ou dados reais ê¢â¬â troque tudo antes de usar em produêÂ§êÂ£o.

---

<div align="center">
  <img src="capa/logo.svg" alt="Arckhé AI" height="32" />
  <p><strong>Arckhé AI</strong></p>
</div>