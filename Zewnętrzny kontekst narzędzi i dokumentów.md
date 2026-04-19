---
source: aidevs4-lesson2_2
---



---
## Wpływ zewnętrznego kontekstu na zachowanie modelu
---

**The Why:**
- LLM nie „wykonuje" promptu jak kod — treści wstrzyknięte przez narzędzia mogą zmienić zachowanie agenta, prowadząc do błędów lub naruszeń bezpieczeństwa (prompt injection).
- Aktualnie nie istnieją narzędzia gwarantujące bezpieczeństwo, więc problem trzeba adresować na poziomie założeń (ograniczenie zakresu treści i umiejętności agenta).
- Rozbudowany kontekst negatywnie wpływa na skuteczność modelu — agent może „zapomnieć" lub zignorować instrukcje systemowe, ponieważ uwaga skupia się na nowo dodanych treściach.

**Summary:**
- Interakcja z LLM jest ustrukturyzowana (system/user/tool/assistant), a zewnętrzne dane niemal zawsze trafiają przez narzędzia.
- Podłączenie agenta do dowolnych źródeł danych jest analogiczne do formularza bez walidacji renderującego HTML (ryzyko XSS).
- Filtrowanie zapytań przez LLM nie jest pełnym zabezpieczeniem — instrukcja może oszukać też filtr; lepsze są: brak narzędzi do wrażliwych akcji oraz wymaganie zgody użytkownika.
- Kluczowe cele pracy z zewnętrznym kontekstem:
  - zawężenie źródeł dokumentów (ograniczenie prompt injection),
  - skuteczny system docierania do treści,
  - ograniczanie jednorazowo wczytywanego kontekstu (dekompozycja, subagenci),
  - UI ułatwiający nawigację po treści i prezentujący oryginalne fragmenty/odnośniki.

---
## Zasady obsługi kontekstu z zewnętrznych źródeł
---

**The Why:**
- Indeksowanie jest konieczne, gdy baza wiedzy obejmuje PDF, DOCX, XLSX, obrazy — bez niego agent nie wie, w którym pliku znajduje się szukana treść.
- Walidacja, moderacja i kontrola dostępu muszą być zaimplementowane programistycznie, ponieważ agent nie powinien być traktowany jako warstwa bezpieczeństwa.

**Summary:**
- Dotąd model łączyliśmy z plikami bezpośrednio (cały dokument lub fragmenty); to wystarcza nawet dla obszernych baz, bo agent czyta tylko wybrane treści.
- Indeksowanie obejmuje:
  - podział plików na chunki/dokumenty,
  - przypisanie metadanych (pochodzenie, kluczowe info),
  - generowanie nowych dokumentów (np. syntez),
  - opisy obrazów, transkrypcje audio/wideo,
  - mapy powiązań (np. grafy),
  - logikę synchronizacji z źródłem.
- Bazy wiedzy są dynamiczne — wymagają synchronizacji opartej o zdarzenia lub harmonogram.
- Bezpieczeństwo plików:
  - walidacja rozmiaru, formatu, mime-type, źródła,
  - moderacja tekstu i obrazów (np. OpenAI Moderation),
  - ochrona dostępu i uprawnienia,
  - otwarte linki — trudne do odgadnięcia i wygasające,
  - optymalizacja (limit OpenAI: 50 MB / zapytanie).
- Wszystkie zabezpieczenia wdrażamy programistycznie; agent dostaje tylko informację o błędzie i wskazówki.

---
## Formaty prezentowania zewnętrznych treści w kontekście
---

**The Why:**
- Forma wizualna dokumentu (obraz) bywa skuteczniejsza niż tekst — wg publikacji o DeepSeek-OCR pozwala uzyskać 9–10× lepszą kompresję przy ~96% precyzji.
- Informowanie o źródle i lokalizacji fragmentu jest istotne nie tylko dla modelu, ale też dla UI (cytaty, odnośniki obok wypowiedzi agenta).

**Summary:**
- Wyniki wyszukiwania trafiają do kontekstu jako wyniki działania narzędzi — obowiązują zasady z S01E02 i S01E03.
- Nawigacja po bazie zwykle opiera się o dwa narzędzia: `search` i `read` (analogicznie do Files MCP).
- Agent nie widzi oryginału — przy raportach z wykresami/zrzutami warto przedstawiać dokument jako obraz.
- Markdown z osadzonymi obrazami (`![alt](link)`) wymaga ekstrakcji obrazów przez regex i przesłania ich zgodnie ze strukturą zapytania.
- Każdy chunk powinien nieść metadane: dokument źródłowy + pozycja fragmentu — kluczowe dla UI (cytaty, linki, raporty).

---
## Techniki indeksowania treści na potrzeby wyszukiwania
---

**The Why:**
- Pytanie „jak tworzyć dokumenty?" musi być zawsze zestawiane z „jak agent będzie do nich docierał?" — strategia chunkowania determinuje skuteczność wyszukiwania.
- Strategia podziału wpływa na strukturę metadanych: podział mechaniczny daje tylko wartości programistyczne, podział kontekstowy/tematyczny pozwala dodać tagi i słowa kluczowe.

**Summary:**
- Za wyszukiwanie w nowoczesnych systemach RAG odpowiada agent (Agentic RAG) korzystający z silników typu Algolia czy Qdrant.
- Pojedynczy dokument: zwykle 200–500 słów lub 500–4000 tokenów.
- Cztery strategie podziału:
  1. **Znaki** — stała liczba znaków, dla nieustrukturyzowanej treści.
  2. **Separatory** — podział rekurencyjny (nagłówki → akapity → zdania → znaki) dla zbliżonej długości fragmentów.
  3. **Kontekst** — separator + LLM dogenerowuje krótki kontekst sytuujący chunk w dokumencie (Anthropic Contextual Retrieval).
  4. **Tematyka** — LLM/agenci od podstaw dzielą dokument na tematy.
- Przykładowe implementacje: `02_02_chunking`.

```20:58:lesson_2_2/practical_examples/02_02_chunking/src/strategies/characters.py
CHUNK_SIZE = 1000
CHUNK_OVERLAP = 200


def chunk_by_characters(
    text: str,
    size: int = CHUNK_SIZE,
    overlap: int = CHUNK_OVERLAP,
) -> List[Dict]:
    chunks: List[str] = []
    start = 0

    while start < len(text):
        chunks.append(text[start: start + size])
        start += size - overlap

    return [
        {
            "content": content,
            "metadata": {
                "strategy": "characters",
                "index": i,
                "chars": len(content),
                "size": size,
                "overlap": overlap,
            },
        }
        for i, content in enumerate(chunks)
    ]
```

```30:30:lesson_2_2/practical_examples/02_02_chunking/src/strategies/separators.py
SEPARATORS = ["\n## ", "\n### ", "\n\n", "\n", ". ", " "]
```

```32:49:lesson_2_2/practical_examples/02_02_chunking/src/strategies/context.py
async def _enrich_chunk(chunk: Dict) -> Dict:
    context = await chat(
        f"<chunk>{chunk['content']}</chunk>",
        _CONTEXT_INSTRUCTIONS,
    )
    return {
        "content": chunk["content"],
        "metadata": {**chunk["metadata"], "strategy": "context", "context": context},
    }
```

---
## Silniki wyszukiwania, bazy wektorowe i pluginy
---

**The Why:**
- Sięganie po dedykowane silniki nie jest konieczne — wybór zależy od skali i wymagań; niższa złożoność architektury bywa wystarczającym uzasadnieniem dla rozwiązań prostszych.
- Trzymanie wszystkich danych w jednej bazie (np. SQLite z FTS5 + sqlite-vec) znacznie ułatwia pracę i synchronizację.

**Summary:**
- Możliwości: pełnoprawne silniki (Elasticsearch), bazy wektorowe (Qdrant), bazy grafowe (Neo4j).
- Dla małej/średniej skali wystarczą:
  - agent połączony bezpośrednio z systemem plików,
  - SQLite/PostgreSQL z rozszerzeniami: FTS5 (full-text), sqlite-vec (semantyka).
- „Hybrydowy RAG" (lex + semantic) nie wymaga już dwóch systemów — większość platform oferuje obie opcje natywnie.
- Integracja z dedykowanym silnikiem (np. Elasticsearch) wymusza synchronizację indeksu z bazą danych.

---
## Przeszukiwanie semantyczne i wybór modelu do embeddingu
---

**The Why:**
- Embedding pozwala odnaleźć **zbliżone znaczeniowo** treści nawet bez dopasowania na poziomie słów (np. „Kobieta" bliżej „Królowej" niż „Króla") — uzupełnia wyszukiwanie leksykalne.
- Modele embeddingowe podlegają tym samym zasadom co LLM: jeśli informacja nie jest w danych treningowych, model nie opisze jej znaczenia poprawnie.

**Summary:**
- Embedding = wektor liczb opisujący znaczenie tekstu; podobieństwo liczone np. **Cosine Similarity**.
- Dwa odrębne procesy:
  - **indeksowanie** (w tle, przy dodawaniu/aktualizacji dokumentów),
  - **wyszukiwanie** (zapytanie użytkownika → embedding → porównanie z bazą).
- Kryteria wyboru modelu embeddingowego:
  - wielkość modelu (cena, sprzęt),
  - liczba wymiarów (text-embedding-3-small = **1536**),
  - okno kontekstowe,
  - knowledge cutoff i wsparcie wielojęzyczne.
- Ranking: leaderboard Hugging Face MTEB.
- Implementacja referencyjna: `02_02_embedding`.

```149:182:lesson_2_2/practical_examples/02_02_embedding/app.py
async def embed(text: str) -> List[float]:
    async with httpx.AsyncClient() as client:
        response = await client.post(
            EMBEDDINGS_API_ENDPOINT,
            headers={
                "Content-Type": "application/json",
                "Authorization": f"Bearer {_AI_API_KEY}",
                **_EXTRA_API_HEADERS,
            },
            json={"model": MODEL, "input": text},
            timeout=60.0,
        )

    data: Dict[str, Any] = response.json()

    if data.get("error"):
        error = data["error"]
        message = (
            error.get("message") if isinstance(error, dict) else str(error)
        ) or str(error)
        raise RuntimeError(message)

    return data["data"][0]["embedding"]
```

```188:208:lesson_2_2/practical_examples/02_02_embedding/app.py
def cosine_similarity(a: List[float], b: List[float]) -> float:
    dot = 0.0
    norm_a = 0.0
    norm_b = 0.0
    for ai, bi in zip(a, b):
        dot += ai * bi
        norm_a += ai * ai
        norm_b += bi * bi
    denom = math.sqrt(norm_a) * math.sqrt(norm_b)
    return dot / denom if denom > 0 else 0.0
```

---
## Techniki przeszukiwania oraz wczytywania kontekstu (retrieval)
---

**The Why:**
- Wyszukiwanie hybrydowe może być **szybsze i bardziej kontrolowalne programistycznie** niż eksploracja systemu plików (CLI), ale kosztem skuteczności — dotarcie do właściwych treści wymaga „rozumowania", nie tylko dopasowania.
- Hybrydowe wyszukiwanie obsługuje multimodalność (obrazy z opisami) — czego CLI samo z siebie nie potrafi.
- Wsparcie wielojęzyczne: agent piszący zapytania niweluje problem tłumaczeń (np. embedding wielojęzyczny dopasuje polskie pytanie do angielskiej treści).
- CLI/grep eliminuje etap indeksowania — w wielu przypadkach to definitywna przewaga.
- Wniosek: pytanie powinno brzmieć „które podejście jest najlepsze dla **mojego** problemu?", nie „które jest najlepsze ogólnie?".

**Summary:**
- Architektura hybrydowego RAG (`02_02_hybrid_rag`):
  - skanuje katalog `workspace`, dzieli pliki na chunki, synchronizuje z SQLite + FTS + sqlite-vec,
  - `EMBEDDING_DIM` musi być **dokładnie taki sam** jak liczba wymiarów modelu (tu 1536).
- Agent przygotowuje **dwa zapytania**: listę słów kluczowych (FTS) i pytanie w języku naturalnym (semantyczne).
- Wyniki łączone przez **RRF (Reciprocal Rank Fusion)** — chunk wysoko w jednej liście a nisko w drugiej awansuje w finalnym rankingu.
- Wyszukiwania (FTS i semantic) mogą być stosowane **równolegle** — uzupełniają się.

```168:236:lesson_2_2/practical_examples/02_02_hybrid_rag/src/db/search.py
async def hybrid_search(
    conn: sqlite3.Connection,
    query: Dict[str, str],
    limit: int = 5,
) -> List[Dict[str, Any]]:
    keywords = query.get("keywords", "")
    semantic = query.get("semantic", "")
    fts_limit = limit * 3

    log.search_header(keywords, semantic)

    fts_results = search_fts(conn, keywords, fts_limit)
    log.search_fts(fts_results)

    vec_results: List[Dict[str, Any]] = []
    try:
        query_embeddings = await embed(semantic)
        query_embedding = query_embeddings[0]
        vec_results = search_vector(conn, query_embedding, fts_limit)
    except Exception as exc:  # noqa: BLE001
        log.warn(f"Semantic search unavailable: {exc}")

    log.search_vec(vec_results)

    scores: Dict[int, Dict[str, Any]] = {}

    for rank, r in enumerate(fts_results):
        chunk_id = r["id"]
        if chunk_id not in scores:
            scores[chunk_id] = dict(r)
            scores[chunk_id]["rrf"] = 0.0
        scores[chunk_id]["rrf"] += 1.0 / (RRF_K + rank + 1)
        scores[chunk_id]["fts_rank"] = rank + 1

    for rank, r in enumerate(vec_results):
        chunk_id = r["id"]
        if chunk_id not in scores:
            scores[chunk_id] = dict(r)
            scores[chunk_id]["rrf"] = 0.0
        scores[chunk_id]["rrf"] += 1.0 / (RRF_K + rank + 1)
        scores[chunk_id]["vec_rank"] = rank + 1
        scores[chunk_id]["vec_distance"] = r.get("vec_distance")

    merged = sorted(scores.values(), key=lambda x: x["rrf"], reverse=True)[:limit]
    log.search_rrf(merged)
```

```26:62:lesson_2_2/practical_examples/02_02_hybrid_rag/src/agent/tools.py
SEARCH_TOOL: Dict[str, Any] = {
    "type": "function",
    "name": "search",
    "description": (
        "Search the indexed knowledge base using hybrid search "
        "(full-text BM25 + semantic vector similarity). "
        "Returns the most relevant document chunks with content, source file, "
        "and section heading. "
        "Provide BOTH a keyword query for full-text search AND a natural language "
        "query for semantic search."
    ),
    "parameters": {
        "type": "object",
        "properties": {
            "keywords": {
                "type": "string",
                "description": (
                    "Keywords for full-text search (BM25) — specific terms, names, "
                    "and phrases that should appear in the text"
                ),
            },
            "semantic": {
                "type": "string",
                "description": (
                    "Natural language query for semantic/vector search — a question "
                    "or description of the concept you're looking for"
                ),
            },
            "limit": {
                "type": "number",
                "description": "Maximum number of results to return (default 5, max 20)",
            },
        },
        "required": ["keywords", "semantic"],
    },
    "strict": False,
}
```

---
## Główne wyzwania skuteczności RAG i zarządzania bazą wiedzy
---

**The Why:**
- Skuteczny system RAG **musi być dopasowany do zakresu danych oraz ich formatów** — uniwersalny skrypt nie zapewni realnej wartości.

**Summary:**
- Główne wyzwania:
  - **Wiedza podstawowa** — agent może pominąć przeszukiwanie, jeśli „uważa", że zna odpowiedź; ryzyko nieaktualnych danych i wieloznaczności terminów.
  - **Zakres wiedzy** — dotarcie do 100% informacji jest bardzo trudne; niekompletne wyniki łatwo prowadzą do halucynacji.
  - **Świadomość informacji** — szczątkowe opisy zasobów uniemożliwiają świadome formułowanie zapytań.
  - **Kontekst** — model nie zna naszego kontekstu („moje projekty" ≠ AI_devs); pomaga semantyka, jeszcze bardziej Graph RAG.
  - **Format danych** — obrazy, audio, wideo są dużo trudniejsze do przeszukania niż tekst, a często stanowią materiały firmowe.
