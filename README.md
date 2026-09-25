# MyApp Chat Analysis Pipeline

Pipeline NLP end-to-end per l'analisi dei messaggi di chat dell'app **MyApp** (dating app), sviluppata su Google Colab con GPU A100.

Estrae topic, sentiment, entità e revenue per fascia d'età degli utenti, con analisi comparativa opzionale via **Gemini AI**.

---

## Funzionalità

| Fase | Cosa fa |
|---|---|
| **Query DB** | Estrae messaggi da MySQL (Tracker + user table) per campagna/paese/device |
| **Preprocessing** | Pulizia testo, rimozione stopwords (IT / EN / American), lemmatizzazione |
| **Embeddings** | Codifica semantica con `paraphrase-multilingual-mpnet-base-v2` (sentence-transformers) |
| **BERTopic** | Topic modeling automatico per fascia d'età (Under 25, 25-40, 41-50, Over 50) |
| **Export Excel** | Output strutturato: topic, sentiment, entità NER (spaCy), word frequency, revenue per topic |
| **Pivot** | Matrice messaggi/utenti per `to_age × BERTopic_Topic` |
| **Gemini AI** | Report strategico comparativo tra fasce + analisi matching età operatrice (celle 6-7) |

---

## Paesi e configurazioni supportate

| Paese | Device | Join method |
|---|---|---|
| 🇮🇹 Italia | iOS / Android | `Tracker_premium` |
| 🇬🇧 UK | iOS / Android | `Tracker_premium` |
| 🇺🇸 USA | iOS / Android | `user_fields` |

Configurazione attiva selezionabile da `ACTIVE_IDX` in **Cella 1** (0–5).  
`RUN_ALL = True` esegue tutte le 6 configurazioni in sequenza.

---

## Struttura del notebook

```
Cella 0  – Installazione dipendenze
Cella 1  – Configurazioni (campaign IDs, paesi, device, parametri)
Cella 2  – Setup librerie, connessione DB, modello embedding
Cella 3  – Funzioni pipeline (query, preprocessing, embeddings, BERTopic, export)
Cella 4  – Esecuzione pipeline
Cella 5  – Diagnostica (verifica copertura dati prima di eseguire)
Cella 6  – Analisi Gemini AI: confronto fasce d'età
Cella 7  – Analisi Gemini AI: matching età operatrice (pivot)
```

---

## Output per ogni configurazione

Tutti i file vengono salvati in Google Drive sotto:
```
/content/drive/MyDrive/Colab Notebooks/MyApp_Results_{PAESE}_{DEVICE}/
```

| File | Contenuto |
|---|---|
| `Messaggi_{tag}_{fascia}.xlsx` | Dati per fascia: messaggi, topic, sentiment, entità, revenue |
| `MyApp_Results_{tag}_final_results.xlsx` | File unificato con tutti i sheet per fascia |
| `MyApp_Results_{tag}_Pivot_to_age.xlsx` | Pivot messaggi/utenti per età operatrice × topic |
| `Gemini_Analysis_{tag}.txt` | Report strategico Gemini AI (se eseguita cella 6) |
| `Gemini_Pivot_Analysis_{tag}.txt` | Analisi matching età operatrice (se eseguita cella 7) |

### Sheet presenti in ogni file per fascia

| Sheet | Descrizione |
|---|---|
| `Messaggi` | Messaggi filtrati con topic assegnato |
| `Topics_Descr` | Parole chiave per ogni topic BERTopic |
| `Topics_Freq` | Frequenza messaggi/utenti per topic |
| `Topics_FullRevenue` | Revenue aggregata per topic |
| `ClickId_Agg` | Revenue e topic per click ID |
| `Sentiment` | Distribuzione sentiment (Positive / Negative / Neutral) |
| `Entita` | Top 20 entità NER (spaCy) |
| `Parole` | Top 30 parole più frequenti (post-cleaning) |
| `Avanzata` | Conteggio tentativi scambio numero di telefono |

---

## Setup

### 1. Requisiti

Progettato per **Google Colab** con GPU (A100 consigliato per velocità embedding).  
Funziona anche su Colab T4, ma più lento per dataset grandi.

### 2. Installazione (automatica — Cella 0)

```bash
pip install bertopic sentence-transformers openpyxl spacy nltk sqlalchemy pymysql hdbscan
python -m spacy download it_core_news_sm
python -m spacy download en_core_web_sm
```

### 3. Configurazione credenziali

1. Copia `.env.example` in `.env`
2. Compila le variabili con le tue credenziali
3. In **Cella 2**, modifica la stringa di connessione usando le variabili d'ambiente oppure compila direttamente (**non committare mai credenziali reali**)

```python
import os
engine = db.create_engine(
    f"mysql+pymysql://{os.environ['DB_USER']}:{os.environ['DB_PASS']}"
    f"@{os.environ['DB_HOST']}:{os.environ.get('DB_PORT','3306')}"
    f"/{os.environ['DB_NAME']}?charset=utf8mb4"
)
```

Su Colab puoi usare i **Secrets** (icona 🔑 nella sidebar) per impostare le variabili d'ambiente senza esporle nel notebook.

### 4. Gemini API Key (celle 6-7, opzionale)

Ottieni una API key gratuita su [Google AI Studio](https://aistudio.google.com/app/apikey) e inseriscila in **Cella 6**:

```python
GEMINI_API_KEY = "la-tua-api-key"
```

---

## Architettura tecnica

```
MySQL DB
  ├── union_MyApp_chat_message_offline  (messaggi)
  ├── MyApp_user_mrkt                   (profilo utenti)
  ├── Tracker_premium                     (click tracking)
  └── Tracker_split                       (revenue)
        │
        ▼
  Pandas DataFrame
        │
  ┌─────────────────────────────────────┐
  │  Preprocessing                      │
  │  • lowercase, rimozione URL/numeri  │
  │  • stopwords per lingua             │
  │  • lemmatizzazione (WordNet)        │
  └──────────────┬──────────────────────┘
                 │
  ┌──────────────▼──────────────────────┐
  │  SentenceTransformer                │
  │  paraphrase-multilingual-mpnet-v2   │
  │  batch GPU → embeddings 768-dim     │
  └──────────────┬──────────────────────┘
                 │
  ┌──────────────▼──────────────────────┐
  │  BERTopic (HDBSCAN + UMAP)         │
  │  • topic automatici per fascia età  │
  │  • checkpoint pickle su Drive       │
  └──────────────┬──────────────────────┘
                 │
  ┌──────────────▼──────────────────────┐
  │  Export Excel + Gemini AI report    │
  └─────────────────────────────────────┘
```

---

## Note sui checkpoint

La pipeline salva checkpoint intermedi su Google Drive per riprendere in caso di interruzione:
- `preprocessing_clean_{tag}.pkl` — dati preprocessati
- `embeddings_checkpoint_{tag}.pkl` — embeddings calcolati per fascia

Per **rieseguire da zero** una fascia, elimina i file `.pkl` e il relativo `.xlsx` dalla cartella Drive.

---

## Dipendenze principali

| Libreria | Versione consigliata | Uso |
|---|---|---|
| `bertopic` | ≥0.16 | Topic modeling |
| `sentence-transformers` | ≥2.7 | Embeddings multilingua |
| `hdbscan` | ≥0.8 | Clustering (dipendenza BERTopic) |
| `spacy` | ≥3.7 | NER (entità nominate) |
| `nltk` | ≥3.8 | Stopwords e lemmatizzazione |
| `sqlalchemy` + `pymysql` | — | Connessione MySQL |
| `pandas` | ≥2.0 | Manipolazione dati |
| `openpyxl` | ≥3.1 | Export Excel |
| `google-generativeai` | ≥0.8 | Gemini AI (opzionale) |

---

## Sicurezza

> **Non committare mai credenziali reali** nel notebook o in altri file.  
> Usa i **Colab Secrets** o variabili d'ambiente per DB e API key.  
> Il file `.env` è escluso dal tracking git tramite `.gitignore`.

