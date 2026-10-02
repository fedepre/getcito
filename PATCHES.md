# PATCHES — fork GetCito (Upper Marketing)

Fork di `ai-search-guru/getcito-worlds-first-open-source-aio-aeo-or-geo-tool`
mantenuto da Upper Marketing per il deploy su `https://geo.uppermarketing.cloud`.

**Serve a**: rimettere queste modifiche in 10 secondi dopo un eventuale conflitto
risolto male da `Sync fork`. Se `Sync fork` produce un conflitto, GitHub chiede di
scegliere quale versione tenere sulle righe toccate: tieni **le nostre** (sono ~4
righe in 2 file) e poi confronta con questo file.

Ultimo aggiornamento: 02/10/2026 — allineato a upstream `2e56fa4`.

---

## Patch 1 — switch dei brand in modalità `local` (APPLICATA)

**File:** `apps/web/src/routes/_authed/app/index.tsx` (riga ~62)

```diff
 			// Single-org mode: redirect to the user's one org (created on signup).
-			if (!result.supportsMultiOrg && result.organizations.length > 0) {
+			if (!result.supportsMultiOrg && result.organizations.length === 1) {
 				throw redirect({ to: "/app/$brand", params: { brand: result.organizations[0].id } });
 			}
```

**Perché:** in `DEPLOYMENT_MODE=local` `supportsMultiOrg` è `false`, quindi con `> 0`
la pagina `/app` (Switch Brand) non si mostra **mai**: redirect su `organizations[0]`.
Con un brand il risultato era "torna sempre al primo"; con due brand (Studio Tiberi)
la pagina va in errore lato client ("Something went wrong") perché il redirect
avviene durante la navigazione.

Con `=== 1` la logica fa quello che dice il commento sopra la riga: un solo brand →
redirect; più brand → mostra la lista e lo switch funziona.

---

## Patch 2 — `end_page: 1` su BrightData Google AI Overview (APPLICATA)

**File:** `packages/lib/src/providers/registry/brightdata.ts` (riga ~161), dentro
`if (model === "google-ai-overview")`

```diff
 					start_page: 1,
-					end_page: 10,
+					end_page: 1,
 					collapse_aio: false
```

**Perché:** è l'unico target che chiede a BrightData di scansionare le pagine 1-10
della SERP, ma GetCito legge **solo il primo record**
(`const record = (Array.isArray(payload) ? payload[0] : payload) ?? {};`). Si pagano
fino a 10 record per prompt e se ne usa uno. L'AI Overview sta in cima alla pagina 1.

Da ~14 a ~5 crediti per prompt. È il presupposto per rimettere `google-ai-overview`
tra i `SCRAPE_TARGETS` (quinta superficie, la più importante in Italia). Nota:
aggiungerla **aumenta** il carico su BrightData di circa il 25%, quindi rende più
probabile la patch 3.

---

## Patch 3 — gate di concorrenza per BrightData (NON ANCORA APPLICATA)

**Da applicare solo se, dopo un giro di misura con le patch 1 e 2, i run falliscono
ancora.** Il codice upstream di oggi ha già dei rimedi sui burst che il 19/09 non
c'erano (cancellazione degli snapshot abbandonati nel `finally`, retry quando lo
snapshot "non è pronto"). Misura prima: se i run passano tutti, il gate non serve e
ci risparmiamo l'unica patch che può generare conflitti.

**File:** `packages/lib/src/providers/registry/brightdata.ts`

```diff
 import { bdclient } from "@brightdata/sdk";
+import { createGate } from "../concurrency";
 import type { Provider, ScrapeResult, ProviderOptions, ModelConfig } from "../types";
```

```diff
+/**
+ * BrightData è l'unico provider senza gate (Olostep 64, Azure 2). Il worker
+ * manda 4 superfici in parallelo per ogni prompt e più prompt insieme: con ~20
+ * richieste insieme BrightData risponde `400 Customer is not active`, che è un
+ * errore fuorviante — è un burst, non un account inattivo.
+ *
+ * Il gate copre trigger -> attesa snapshot, non solo il trigger: il tempo di
+ * attesa è la risorsa scarsa (stesso ragionamento di olostep.ts).
+ */
+const gate = createGate(2);
+
 export const brightdata: Provider = {
```

```diff
 	async run(model: string, prompt: string, options?: ProviderOptions): Promise<ScrapeResult> {
+		return gate(async () => {
 			const datasetId = options?.version ?? BD_DATASET_IDS[model];
 			...
+		});
 	},
```

**Nota — il diff 3 richiede più di una riga**: il corpo di `run` va rientrato dentro
la callback (o, meglio, il corpo va spostato in una funzione interna e `run` diventa
`return gate(() => runInner(...))`). Da fare con attenzione e rileggendo tutto il
blocco `try/finally` (che deve restare **dentro** il gate, altrimenti il cancel dello
snapshot scatta mentre un altro job aspetta).

---

## Come si aggiorna questo fork

1. GitHub → il fork → **Sync fork** → *Update branch*
   - è una **fusione**, non una sovrascrittura: le patch qui sopra restano e arrivano
     le novità upstream
2. Se GitHub segnala un conflitto, risolvi tenendo le **nostre** righe (sono ~4 in 2
   file) e usa questo documento per rimetterle
3. Dokploy → **Redeploy** (vedi `6-AGGIORNARE.txt`)

**Ridurre la manutenzione a zero:** proporre le patch 1 e 2 come Pull Request a
upstream. La 1 è chiaramente un bugfix (il commento sopra la riga contraddice il
codice), quindi ha buone probabilità di essere accettata. Se le accettano, il fork
non serve più.
