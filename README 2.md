# AI Invoice Embeddings API

Piccolo progetto didattico per esplorare dati di fatture e progettare lo schema di una API di embeddings a supporto dell'invoice matching.

L'idea e' partire da fatture strutturate, osservare quali campi sono disponibili, creare versioni "noisy" dei dati e valutare se una ricerca semantica basata su embeddings riesce a recuperare la fattura ground truth corretta tra i candidati.

## Obiettivo del progetto

Il progetto simula un caso realistico di invoice matching: dato un insieme di fatture corrette, usate come ground truth, vengono create versioni alternative con errori sintetici. Questi errori imitano problemi comuni in contesti reali, come OCR imperfetto, inserimento manuale sbagliato, formati data/importo incoerenti, nomi aziendali abbreviati o campi parzialmente mancanti.

Il dataset risultante permette di costruire e valutare una piccola API di embeddings per invoice matching. L'idea non e' sostituire il fuzzy matching o le regole di SAP, ma affiancarle con un servizio di semantic candidate retrieval: SAP puo' inviare una fattura rumorosa o normalizzata, ricevere una shortlist di fatture candidate semanticamente simili, e poi validare i campi forti con le proprie logiche strutturate.

In questo progetto l'embedding serve soprattutto per confrontare informazioni testuali e semantiche, come nomi di aziende, indirizzi, descrizioni delle righe prodotto e contesto generale della fattura. Campi come quantita', totale, subtotale, sconti, date, numero fattura, VAT o PO number non sono il contenuto principale dell'embedding: sono segnali strutturati, piu' adatti a filtri, controlli esatti, fuzzy matching tradizionale o reranking lato SAP.

Questa separazione rende il progetto piu' realistico: l'API propone candidati semanticamente plausibili, mentre SAP resta responsabile della verifica contabile e dei controlli numerici.

## Dataset di partenza

Il file raw usato nel progetto e':

`data/raw/invoices_dataset.json`

Il dataset e' stato scaricato da Kaggle:

[7000 Invoice Images with JSON](https://www.kaggle.com/datasets/ananthakrishnanpv/7000-invoice-images-with-json?utm_source=chatgpt.com)

Ogni record contiene:

- il nome dell'immagine della fattura, per esempio `001.png`;
- una conversazione in stile instruction dataset;
- una risposta `gpt` con la fattura in formato JSON testuale.

## Sintesi EDA

La prima esplorazione si trova nel notebook:

`notebooks/eda_invoices_dataset.ipynb`

### Conteggi principali

| Metrica | Valore |
|---|---:|
| Record nel JSON raw | 7.000 |
| Fatture parsate correttamente | 7.000 |
| Fatture con errore di parsing | 0 |
| Righe prodotto totali | 28.067 |
| Prodotti medi per fattura | 4,01 |

### Sezioni principali presenti nelle fatture

| Sezione | Fatture con sezione presente |
|---|---:|
| `buyer` | 7.000 |
| `invoice` | 7.000 |
| `products` | 7.000 |
| `seller` | 7.000 |
| `payment` | 5.000 |
| `tax` | 4.999 |
| `supplier` | 1.000 |

### Alcuni campi utili

| Campo | Valori presenti | Nota |
|---|---:|---|
| `payment.total` | 3.000 | Totale fattura, non sempre disponibile |
| `buyer.country` | 2.001 | Spesso mancante |
| `products.description` | 28.067 | Presente in tutte le righe prodotto |
| `products.quantity` | 24.025 | Presente nella maggior parte delle righe prodotto |
| `products.unit_price` | 19.970 | Campo prezzo unitario piu' frequente |
| `products.amount` | 19.956 | Altro campo importo molto frequente |

## Dataset processati

La cartella `data/processed` contiene dataset derivati per esercizi di matching:

- `invoices_noisy_matching.csv`
- `invoices_noisy_matching.jsonl`
- `invoices_ground_truth_corpus.jsonl`
- `invoices_noisy_evaluation.csv`
- `invoices_noisy_evaluation.jsonl`
- `embedding_corpus.jsonl`
- `embedding_queries.jsonl`
- `corpus_embeddings.jsonl`
- `query_embeddings.jsonl`
- `semantic_search_results.jsonl`
- `semantic_search_metrics.json`

Ogni record contiene sia i campi originali (`gt_*`) sia i campi alterati (`noisy_*`), piu' alcune colonne di supporto come `noise_level`, `error_types` e `changed_fields`. In questo modo e' possibile sapere quali errori sono stati introdotti e misurare quanto bene un algoritmo riesce a recuperare la fattura corretta.

Per lo scenario API embeddings, `invoices_ground_truth_corpus.jsonl` rappresenta il corpus da indicizzare: contiene il 100% delle fatture ground truth. I file `invoices_noisy_evaluation.*` contengono invece il 30% delle fatture noisy, divise in 15% validation e 15% test tramite la colonna `evaluation_split`.

Lo split di evaluation e' stratificato su `noise_level` e sull'errore primario, cioe' il primo valore in `error_types`, cosi' validation e test mantengono una distribuzione simile dei livelli di rumore e delle principali famiglie di errore.

### Errori sintetici introdotti

Gli errori sono stati generati per imitare situazioni comuni nei processi di invoice matching reali, dove le informazioni possono arrivare da OCR, inserimento manuale, sistemi gestionali diversi o formati non standardizzati.

Le principali famiglie di errore sono:

- errori OCR su caratteri simili, per esempio `O`/`0`, `I`/`1`, `S`/`5`, `B`/`8`;
- caratteri mancanti, caratteri invertiti o spazi inseriti in punti errati;
- variazioni di maiuscole/minuscole nei campi testuali;
- abbreviazioni o normalizzazioni parziali dei nomi aziendali;
- scambio sintetico tra campi buyer e seller;
- date scritte in formati diversi, con giorno/mese invertiti o piccoli offset temporali;
- importi arrotondati, leggermente alterati o con cifre trasposte;
- descrizioni prodotto riordinate nella stessa fattura o sostituite con descrizioni prese da altre fatture;
- valori mancanti nei casi piu' rumorosi.

Questi errori sono utili per simulare casi in cui due fatture rappresentano lo stesso documento, ma non sono identiche a livello testuale o numerico. In questo modo il progetto puo' testare tecniche di matching piu' robuste della semplice uguaglianza esatta.

### Esempi reali di modifiche generate

Di seguito alcuni esempi presi dal dataset processato, utili per capire come il ground truth viene trasformato in una versione rumorosa.

| Record | Livello rumore | Campo | Ground truth | Valore noisy | Tipo errore |
|---:|---|---|---|---|---|
| 3 | `low` | `seller.company_name` | `Williams-Johnson` | `Wi1liams-Johnson` | Confusione OCR tra `l` e `1` |
| 16 | `low` | `invoice.date` | `09.11.2023` | `11.09.2023` | Inversione giorno/mese |
| 20 | `low` | `invoice.date` | `01.08.2005` | `01-08-05` | Variazione formato data |
| 21 | `low` | `seller.vat_no` | `lC44769628359` | `lC447696283S9` | Confusione OCR tra `5` e `S` |
| 22 | `low` | `seller.company_name` | `Anderson, Williams and Moore` | `Anderosn, Williams and Moore` | Caratteri adiacenti invertiti |
| 12 | `medium` | `payment.total` | `535.0` | `553.0` | Trasposizione di cifre |
| 12 | `medium` | `invoice.date` | `17.06.1993` | `16.06.1993` | Piccolo offset temporale |
| 6 | `high` | `seller.company_name` | `Warner-Smith` | `Warner-5mith` | Confusione OCR tra `S` e `5` |
| 6 | `high` | `buyer.company_name` | `Bradford, Stone and Johnson` | `bradford, stone and johnson` | Variazione maiuscole/minuscole |
| 6 | `high` | `products.0.unit_price` | valore presente | valore mancante | Campo rimosso nei casi piu' rumorosi |

Questi esempi mostrano tre famiglie di problemi: errori testuali piccoli ma rilevanti, differenze di formattazione e alterazioni numeriche. Sono casi semplici da interpretare per una persona, ma sufficienti a rendere fragile un matching basato solo su uguaglianza esatta.

### Gravita' del rumore

Ogni record riceve un livello di rumore:

| Livello | Distribuzione target | Descrizione |
|---|---:|---|
| `low` | 70% | Pochi errori leggeri, spesso su uno o pochi campi; raramente errori strutturali |
| `medium` | 20% | Piu' campi alterati, con combinazioni di errori testuali, date, importi o prodotti |
| `high` | 10% | Casi difficili, con molti campi modificati, possibili swap buyer/seller, prodotti sostituiti e valori mancanti |

La generazione usa un seed fisso (`42`), quindi il dataset e' riproducibile. Le percentuali non sono forzate esattamente, ma seguono la distribuzione target su 7.000 record.

Questi file sono generati dal notebook:

`notebooks/create_noisy_invoices.ipynb`

Per maggiori dettagli sui file processati e sulle colonne disponibili, vedere:

`data/processed/README.md`

## Preparazione dei testi per embeddings

La rappresentazione testuale per il retrieval semantico viene generata dal notebook:

`notebooks/prepare_embedding_jsonl.ipynb`

Il notebook legge:

- `data/processed/invoices_ground_truth_corpus.jsonl`
- `data/processed/invoices_noisy_evaluation.jsonl`

e produce:

- `data/processed/embedding_corpus.jsonl`, basato sulle fatture `ground_truth`;
- `data/processed/embedding_queries.jsonl`, basato sulle fatture `noisy_invoice` e con `evaluation_split` conservato nei metadata.

Ogni record contiene `record_id`, `source_image`, `embedding_text` e `metadata`.

`embedding_text` include solo segnali testuali e semantici utili agli embeddings: nome azienda seller, nome azienda buyer, indirizzo seller, indirizzo buyer, paese buyer e descrizioni prodotto.

I campi strutturati o numerici non vengono usati come contenuto principale dell'embedding. Numero fattura, data fattura, quantita', prezzi unitari, prezzi totali, subtotal, total, discount, VAT number e altri identificativi restano in `metadata`, dove potranno essere usati da SAP per filtri, controlli esatti, fuzzy matching o reranking.

Questo step prepara i dati soltanto: non esegue ancora chiamate a modelli embedding, vector database o API FastAPI.

## Check degli input per embeddings

Il notebook di validazione degli input testuali si trova in:

`notebooks/check_embedding_inputs.ipynb`

Il notebook legge:

- `data/processed/embedding_corpus.jsonl`
- `data/processed/embedding_queries.jsonl`

ed esegue sanity check su conteggi, schema, testi vuoti o troppo corti, statistiche di lunghezza, numero medio di descrizioni prodotto, esempi casuali, presenza di etichette vietate nel testo e correttezza di `evaluation_split` nelle queries.

Anche questo step non chiama modelli embedding, vector database o API FastAPI: serve solo a validare i dati prima del passo successivo.

## Generazione embeddings

Il notebook per generare e salvare i vettori si trova in:

`notebooks/generate_embeddings.ipynb`

Il notebook legge:

- `data/processed/embedding_corpus.jsonl`
- `data/processed/embedding_queries.jsonl`

e produce:

- `data/processed/corpus_embeddings.jsonl`
- `data/processed/query_embeddings.jsonl`

Ogni record di output conserva `record_id`, `source_image` e `metadata`, e aggiunge:

- `embedding_model`
- `embedding_dimension`
- `embedding`

Il notebook usa un modello embedding locale e gratuito tramite `sentence-transformers`.
Di default usa `sentence-transformers/all-MiniLM-L6-v2`, ma il nome modello puo' essere cambiato nella cella di configurazione o tramite variabile d'ambiente:

```powershell
$env:EMBEDDING_MODEL = "sentence-transformers/all-MiniLM-L6-v2"
$env:EMBEDDING_BATCH_SIZE = "64"
```

Non serve una API key. La prima esecuzione puo' scaricare il modello da Hugging Face; le esecuzioni successive usano la cache locale.

Se il pacchetto non e' gia' installato:

```powershell
pip install sentence-transformers
```

Il salvataggio e' atomico: il notebook scrive prima un file temporaneo e poi sostituisce il file finale. Rieseguire il notebook quindi non accoda record e non crea duplicati inconsistenti.

Anche questo step non implementa vector database o FastAPI: genera soltanto gli embeddings e li salva su file JSONL.

## Semantic search locale baseline

Il notebook baseline per il retrieval semantico locale si trova in:

`notebooks/semantic_search_baseline.ipynb`

Il notebook legge:

- `data/processed/corpus_embeddings.jsonl`
- `data/processed/query_embeddings.jsonl`

e produce:

- `data/processed/semantic_search_results.jsonl`
- `data/processed/semantic_search_metrics.json`

Usa gli embeddings gia' salvati e calcola la cosine similarity in memoria con NumPy. Per ogni query recupera i `top_k` record piu' simili del corpus, preservando `record_id`, `source_image`, `metadata` della query e una lista di candidati con `record_id`, `source_image`, `similarity_score` e metadata compatti.

Il match corretto e' definito in modo didattico: `query.record_id == corpus.record_id`. Le metriche salvate sono top-1 accuracy, top-3 accuracy, top-5 accuracy e MRR, sia complessive sia per `evaluation_split` quando presente nei metadata.

I percorsi e `top_k` sono configurabili nella cella iniziale oppure tramite variabili d'ambiente:

```powershell
$env:SEMANTIC_SEARCH_TOP_K = "5"
$env:SEMANTIC_CORPUS_EMBEDDINGS_PATH = "data/processed/corpus_embeddings.jsonl"
$env:SEMANTIC_QUERY_EMBEDDINGS_PATH = "data/processed/query_embeddings.jsonl"
$env:SEMANTIC_SEARCH_RESULTS_PATH = "data/processed/semantic_search_results.jsonl"
$env:SEMANTIC_SEARCH_METRICS_PATH = "data/processed/semantic_search_metrics.json"
```

Il notebook include controlli su record processati, dimensione vettori, record_id duplicati e query senza match nel corpus. Anche questo salvataggio e' atomico: rieseguire il notebook sostituisce i file finali e non accoda risultati inconsistenti.

Non implementa BM25, hybrid search, Weaviate, vector database o FastAPI: e' solo una baseline locale sugli embeddings gia' generati.

## Analisi degli errori di retrieval

Il notebook per l'analisi dei fallimenti si trova in:

`notebooks/analyze_retrieval_errors.ipynb`

Il notebook legge:

- `data/processed/semantic_search_results.jsonl`

e non produce file di output: e' pensato solo per l'esplorazione interattiva.

### Metriche della baseline

Il modello usato e' `sentence-transformers/all-MiniLM-L6-v2`. Le metriche ottenute su 2.100 query di evaluation sono:

| Split | Query | Top-1 acc. | Top-3 acc. | Top-5 acc. | MRR |
|---|---:|---:|---:|---:|---:|
| Totale | 2.100 | 98,8% | 99,0% | 99,0% | 98,9% |
| Validation | 1.050 | 98,9% | 99,1% | 99,1% | 99,0% |
| Test | 1.050 | 98,7% | 99,0% | 99,0% | 98,8% |

### Struttura del notebook

Il notebook e' diviso in undici sezioni:

1. import e percorsi;
2. caricamento di `semantic_search_results.jsonl`;
3. costruzione di un DataFrame flat con i campi rilevanti estratti dai metadata;
4. separazione tra fallimenti top-1 (`correct_rank > 1`) e successi (`correct_rank == 1`);
5. distribuzione dei fallimenti per `noise_level`, con tasso di fallimento per livello;
6. tasso di fallimento per tipo di errore (`error_types`), con conteggi su tutto il dataset e solo sui fallimenti;
7. tasso di fallimento per campo modificato (`changed_fields`), filtrato sui campi con almeno 5 occorrenze;
8. confronto tra campi modificati dentro l'`embedding_text` e campi solo in `metadata`, separato per fallimenti e successi — la sezione piu' diagnostica;
9. ispezione manuale dei 30 casi peggiori, cioe' quelli con `correct_rank` piu' alto;
10. metriche top-1, top-3, top-5 e MRR separate per `noise_level`;
11. riepilogo testuale compatto.

La sezione 8 risponde alla domanda principale: i fallimenti sono causati da errori su campi semantici come nomi azienda o descrizioni prodotto, oppure da errori su campi strutturati che l'embedding non vede? Se i fallimenti si concentrano su modifiche a `seller.company_name`, `buyer.company_name`, `products.*.description` o sugli errori strutturali `buyer_seller_swap` e `product_description_replace`, significa che i limiti della baseline sono intrinseci alla scelta di cosa mettere nell'`embedding_text`. Se invece i fallimenti sono distribuiti su campi solo in `metadata`, la baseline e' gia' robusta e i fallimenti non sono recuperabili con soli embeddings.

### Risultati dell'analisi

Su 2.100 query totali si contano 26 fallimenti top-1 (1,2%). Le metriche per livello di rumore mostrano un degrado netto solo sui casi piu' difficili:

| Noise level | Query | Top-1 acc. | Top-3 acc. | Top-5 acc. | MRR |
|---|---:|---:|---:|---:|---:|
| `low` | 1.465 | 99,8% | 99,8% | 99,8% | 99,8% |
| `medium` | 415 | 98,8% | 99,0% | 99,0% | 98,9% |
| `high` | 220 | 91,8% | 94,1% | 94,1% | 93,1% |

L'analisi degli `error_types` mostra che tutti i 26 fallimenti hanno `product_description_replace` tra i loro errori, con un tasso di fallimento del 10% per quel tipo. Gli altri errori strutturali `buyer_seller_swap` (5,7%) e `missing_value` (5,8%) contribuiscono ma co-occorrono quasi sempre con `product_description_replace` nei casi `high`. Tutti gli altri tipi di errore — OCR, trasposizioni di caratteri, variazioni di date, importi alterati — si attestano tra il 2% e il 3%, confermando che la baseline e' robusta su questi casi.

Il motivo e' strutturale: `product_description_replace` sostituisce la descrizione di uno o piu' prodotti con quella presa da un'altra fattura del corpus. Poiche' le descrizioni prodotto sono la parte piu' discriminante dell'`embedding_text`, il vettore della query si avvicina alla fattura sbagliata invece che alla ground truth. Si tratta di un limite intrinseco all'approccio semantico, non recuperabile solo con embeddings.

In un contesto reale, se le descrizioni prodotto nelle fatture noisy restano quelle originali (anche con altri errori), la baseline e' gia' sufficiente. Se invece le descrizioni vengono modificate o riclassificate, i 26 fallimenti rappresentano il limite naturale di questo approccio e richiederebbero un reranking con segnali strutturati come numero fattura, importo o data, gestito lato SAP.

## Note

Il progetto e' volutamente semplice e didattico. L'EDA usa soprattutto `pandas` e `matplotlib`, con codice leggibile e commenti essenziali.

Prossimi sviluppi naturali:

- confrontare questa baseline con BM25 o hybrid search;
- usare campi numerici, date e identificativi come metadata per filtri o reranking gestiti da SAP, non come contenuto principale dell'embedding;
- esporre il retrieval tramite API solo dopo aver validato bene il comportamento locale.
