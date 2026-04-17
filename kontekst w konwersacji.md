---
source: aidevs4-lesson2_1
---
# Podsumowanie Lekcji 2_1

  

## 🧠 Kluczowe Koncepty Teoretyczne i Praktyka

  

---

  

### 1. Instrukcja Systemowa jako Mapa, nie Encyklopedia

  

* **Teoria:** Instrukcja systemowa służy do **dopasowania zachowania modelu**, a nie do przekazania wszystkich informacji. Powinna działać jak mapa — dawać ogólną orientację w terenie, a nie szczegółowy opis każdego kamienia. Zawiera: **uniwersalne instrukcje**, informacje o **otoczeniu** (środowisko, użytkownik), stan **sesji** oraz zasady funkcjonowania w **zespole agentów**. Zbyt specyficzne instrukcje to szum.

* **Praktyka:**

  

```python

# src/config.py — instrukcja systemowa jako zestaw ogólnych zasad eksploracji,

# NIE jako lista kroków dla konkretnych dokumentów

"instructions": (

"You are an expert research assistant with access to a knowledge base "
"of AI_devs course notes (Polish-language S01*.md files)...\n\n"

"### Phase 1 — Scan (explore structure first)\n"
"- List the knowledge-base directory to discover available files.\n"
"- Do NOT read entire files upfront. Use search to locate relevant sections first.\n\n"

"### Phase 2 — Deepen (multi-angle keyword search)\n"
"- Run at least 3–5 distinct keyword searches covering different angles "
"(synonyms, related concepts, cause/effect, part/whole, examples).\n"

...

)

```

  

---

  

### 2. Sygnał vs. Szum w Kontekście

  

* **Teoria:** Proporcja istotnych informacji do zbędnych bezpośrednio wpływa na jakość działania agenta. Nie możemy dosłownie "kontrolować sygnału" w dynamicznych systemach, ale możemy **stwarzać warunki do jego wysokości** przez: poprawne dostarczanie kontekstu, wysokiej jakości logikę aplikacji, dopracowane instrukcje i schematy narzędzi, generyczne mechanizmy oraz przestrzeń na interwencję człowieka.

* **Praktyka:**

  

```python

# agent.py — błędy narzędzi nie przerywają pętli; model obserwuje błąd
# i samodzielnie decyduje, jak zareagować — to jest "przestrzeń na korektę"
try:
    result = await call_mcp_tool(client, name, args)
    output_str = json.dumps(result) if not isinstance(result, str) else result
    log.tool_result(name, True, output_str)
    return {"type": "function_call_output", "call_id": call_id, "output": output_str}
except Exception as exc:
    error_msg = json.dumps({"error": str(exc)})
    log.tool_result(name, False, error_msg)
    return {"type": "function_call_output", "call_id": call_id, "output": error_msg}

```

  

---

  

### 3. Agentic RAG — Kształtowanie Kontekstu przez Obserwację

  

* **Teoria:** Agent domyślnie **"nie wie, o czym wie"**. Musi być poprowadzony do aktywnego eksplorowania zasobów. Zamiast jednorazowego wyszukiwania, agent buduje kontekst iteracyjnie: **skanuje** strukturę, **pogłębia** poprzez synonimy i powiązane słowa kluczowe, **eksploruje** łańcuchy przyczynowo-skutkowe, **weryfikuje** pokrycie przed udzieleniem odpowiedzi. To jest Agentic RAG.

* **Praktyka:**

  

```python
# agent.py — pętla agentic loop: chat → tool call → wynik → kolejny krok
for step in range(1, MAX_STEPS + 1):
    response = await chat(input=messages, tools=openai_tools)
    tool_calls = extract_tool_calls(response)

    if not tool_calls:
        # Model zakończył eksplorację — ma finalną odpowiedź
        final_text = extract_text(response)
        messages.extend(response.get("output") or [])
        return {"response": final_text, "conversation_history": messages}

    # Dołącz wyniki narzędzi do historii — agent "obserwuje" otoczenie
    messages.extend(response.get("output") or [])
    tool_results = await asyncio.gather(
        *[_run_tool(mcp_client, tc) for tc in tool_calls]
    )
    messages.extend(tool_results)

```

  

---

  

### 4. Generalizowanie Instrukcji

  

* **Teoria:** Kluczowa umiejętność przy projektowaniu agentów: instrukcje muszą być **niezależne od konkretnych narzędzi i dokumentów**, elastycznie dopasowując się do każdej sytuacji. Zbyt bezpośrednia instrukcja (np. "linki do filmów przekazuj do `analyze_video`") jest anty-wzorcem — naprawia jeden błąd, ale nie adresuje kategorii problemu. Model może pomagać w iterowaniu instrukcji, ale programista musi samodzielnie ocenić jakość wyniku.

* **Praktyka:**

  

```python

# config.py — instrukcja opisuje ZASADY zachowania, nie konkrety
"### Phase 3 — Explore (follow related aspects)\n"
"- Consider what the user might also want to know about this topic.\n"
"- Follow cause/effect chains, prerequisite concepts, and practical implications.\n\n"
"### Phase 4 — Verify (check for gaps)\n"
"- Before writing your answer, ask: are there aspects I haven't searched yet?\n"
"- Run additional targeted searches to fill any gaps.\n\n"
"## Efficiency rules\n"
"- Never read an entire file in one call — search first, then read the relevant "
"line range identified in the search results.\n"
"- Run searches in parallel when multiple independent keywords need checking.\n\n"

```

  

---

  

### 5. Dynamiczna Instrukcja Systemowa i Prompt Cache

  

* **Teoria:** Okno kontekstowe to: `[system prompt] + [definicje narzędzi] + [konwersacja]`. Każda zmiana w instrukcji systemowej **unieważnia cache dla definicji narzędzi**, bo te znajdują się bezpośrednio pod nią. Dlatego informacje **dynamiczne** (np. aktualny plik, branch git) wstrzykuje się do **wiadomości użytkownika** (nie do system promptu), zachowując wysoki `cache hit`. Powtarzanie kluczowych instrukcji w wiadomościach użytkownika pomaga też zarządzać **uwagą modelu** w długich sesjach.

* **Praktyka:**

  

```python

# repl.py — historia konwersacji jest kumulowana między turami (multi-turn),
# ale instrukcja systemowa (config.py) pozostaje STATYCZNA — nie jest modyfikowana
conversation = create_conversation()
# ...
result = await run(
    user_input,
    mcp_client=mcp_client,
    mcp_tools=mcp_tools,
    conversation_history=conversation["history"],  # dynamika → w historii, nie w system prompt
)
conversation["history"] = result["conversation_history"]

```

  

---

  

### 6. Planowanie i Listy Zadań jako Zarządzanie Uwagą

  

* **Teoria:** Narzędzia agenta nie muszą wpływać na zewnętrzne systemy. Mogą służyć **zarządzaniu uwagą modelu**: lista zadań do odhaczyvania po każdym kroku **powtarza** priorytetowe cele w kontekście, co zapobiega "zgubieniu wątku" przy długich interakcjach. Treści **generowane przez model** (a nie przesyłane z zewnątrz) mogą mieć silniejszy wpływ na jego zachowanie.

* **Praktyka:**

  

```python

# agent.py — model widzi w historii wszystkie poprzednie kroki i wyniki narzędzi,
# co stanowi naturalną "listę wykonanych działań" i utrzymuje kontekst aktywnych celów
messages.extend(response.get("output") or [])  # wyniki modelu → historia
messages.extend(tool_results)                   # wyniki narzędzi → historia
# Każdy kolejny krok modelu "widzi" wszystko, co się już wydarzyło

```

  

---

  

### 7. Kontrola Stanu Poza Oknem Kontekstowym (Agent Harness)

  

* **Teoria:** Nie wszystko dzieje się w oknie kontekstu. **Agent Harness** to infrastruktura wokół modelu obejmująca: monitorowanie sesji przez hooki, budowanie pamięci w tle (np. Batch API), system plików jako współdzielona przestrzeń między agentami i sesjami, oraz aktualizacje otoczenia wstrzykiwane do kontekstu po spełnieniu warunków. Workspace agenta powinien mieć klarowną strukturę: załączniki użytkownika, notatki sesji, dokumenty publiczne, dokumenty wewnętrzne (inbox/outbox między agentami).

* **Praktyka:**

  

```python

# app.py — MCP file server jako zewnętrzna przestrzeń dla agenta;
# pliki istnieją niezależnie od okna kontekstowego i są dostępne między sesjami
mcp_client, exit_stack = await create_mcp_client("files")
mcp_tools = await list_mcp_tools(mcp_client)
# Agent korzysta z narzędzi MCP do eksploracji systemu plików — to jest jego "workspace"

```

  

---

  

### 8. Maskowanie Elementów Kontekstu (Prefilling)

  

* **Teoria:** Technika opisana przez zespół Manus: deterministyczne **uzupełnienie początku wypowiedzi modelu** tokenami wskazującymi konkretne narzędzia. Gdy agent deterministycznie musi użyć danej grupy narzędzi (np. przeglądarka), prefilling `<tool_call>{"name": "browser_...` fizycznie ogranicza możliwe akcje. Aktualnie **deprecated** w API Anthropic, ale ważna jako przykład niestandardowego myślenia o sterowaniu kontekstem.

* **Praktyka:** Technika rzadko dostępna w standardowych SDK; implementowana bezpośrednio w wywołaniach API poprzez wypełnienie pola `assistant` treścią przed otrzymaniem odpowiedzi modelu.