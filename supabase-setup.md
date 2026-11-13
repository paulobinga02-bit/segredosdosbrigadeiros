# Configuração do Supabase — compras reais e espectadores

## 1. Tabela de compras confirmadas (SQL Editor)

```sql
create table public.purchases (
  id uuid primary key default gen_random_uuid(),
  display_name text not null,        -- nome abreviado/autorizado, ex.: "Maria F."
  external_ref text unique,          -- ID da transacção no checkout (evita duplicados)
  confirmed boolean not null default false,
  created_at timestamptz not null default now()
);

alter table public.purchases enable row level security;

-- O público só lê compras confirmadas. Ninguém escreve pelo frontend.
create policy "ler compras confirmadas" on public.purchases
  for select to anon using (confirmed = true);

alter publication supabase_realtime add table public.purchases;
```

Só a `service_role` (no servidor) consegue inserir, porque não há política de insert.

## 2. Webhook autenticado (Edge Function `purchase-webhook`)

Só funciona se o checkout (Mentora) enviar webhooks de pagamento confirmado.
Confirma na documentação deles o nome dos campos e como assinam o pedido.

```ts
import { createClient } from "npm:@supabase/supabase-js@2";

Deno.serve(async (req) => {
  // Segredo partilhado com o checkout (defina WEBHOOK_SECRET nas Secrets)
  if (req.headers.get("x-webhook-secret") !== Deno.env.get("WEBHOOK_SECRET"))
    return new Response("unauthorized", { status: 401 });

  const body = await req.json();
  // AJUSTAR aos campos reais do checkout:
  if (body.status !== "paid") return new Response("ignored", { status: 200 });

  const full = String(body.customer_name ?? "").trim().split(/\s+/);
  const display = full.length > 1 ? `${full[0]} ${full[1][0]}.` : full[0]; // privacidade

  const sb = createClient(Deno.env.get("SUPABASE_URL")!, Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!);
  const { error } = await sb.from("purchases").upsert(
    { display_name: display, external_ref: String(body.transaction_id), confirmed: true },
    { onConflict: "external_ref" },
  );
  return new Response(error ? "error" : "ok", { status: error ? 500 : 200 });
});
```

Só guarda nomes de compradores que tenham autorizado a sua apresentação.

## 3. Espectadores (VSL)

Não precisa de tabela: usa o Realtime Presence do canal `vsl-viewers`.
Basta activar o Realtime no projecto. A sessão desaparece quando a pessoa pausa,
termina o vídeo ou fecha a página.

## 4. Na página

Preenche no objecto `CONFIG`: `SUPABASE_URL` e `SUPABASE_ANON_KEY` (chave anon, pública).
Nunca coloques a `service_role` no frontend.

## 5. Testes que faltam (precisam do projecto Supabase e de 2 dispositivos)

- Iniciar a VSL num dispositivo: o contador no outro passa a 1; pausar volta a ocultar.
- Fechar o separador: a sessão expira em poucos segundos.
- Inserir uma linha com `confirmed = true`: aparece a notificação durante 3s.
- Inserir com `confirmed = false`: não aparece nada.
