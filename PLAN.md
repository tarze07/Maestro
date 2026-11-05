# Maestro: Architektura i Plan Implementacji
> Hybrydowe środowisko programistyczne nowej generacji łączące ergonomię i responsywność **Zed** z orkiestracją wieloagentową **Orca**, zaprojektowane pod kątem rygorystycznych wymogów bezpieczeństwa sieci firmowych (On-Premises / Air-Gapped).

---

## 1. Wstęp i Geneza Projektu

Współczesna inżynieria oprogramowania przechodzi transformację od pojedynczych asystentów czatu (Copilot/ChatGPT) do **autonomicznych zespołów agentów AI (Multi-Agent Systems)** pracujących równolegle nad złożonymi zadaniami. 

**Maestro** łączy dwa kluczowe paradygmaty:
1. **Zed (Editor & Collaboration Benchmark):**
   - Błyskawiczny, minimalistyczny interfejs zoptymalizowany pod kątem braku opóźnień (zero-latency feel).
   - Współpraca w czasie rzeczywistym oparta o CRDT (Conflict-free Replicated Data Types).
   - Wbudowany panel asystenta z precyzyjnym zarządzaniem kontekstem (LSP, symbole, pliki, commity).
2. **Orca (Multi-Agent Fleet & Isolation Benchmark):**
   - Równoległa orkiestracja floty agentów (Agent Development Environment – ADE).
   - Izolacja agentów w niezależnych gałęziach i katalogach roboczych (**Git Worktrees**), eliminująca konflikty i nadpisywanie plików.
   - Architektura ról: *Koordynator / Architekt* planuje i dzieli zadania, *Agenci Domenowi* (Frontend, Backend, Tester, Reviewer) realizują je równolegle.
   - Wizualna inspekcja postępu, podgląd zmian (diff viewer) oraz kontrola człowieka w pętli (*Human-in-the-Loop*).
3. **Enterprise Air-Gapped Security:**
   - Praca w sieciach izolowanych / intranetach bez wycieku kodu na zewnątrz.
   - Wsparcie dla lokalnych modeli (Ollama, vLLM, LocalAI) lub dedykowanych korporacyjnych bramek (Azure OpenAI Private Endpoints, AWS PrivateLink).
   - Moduł DLP (Data Loss Prevention), skanowanie sekretów i restrykcyjne piaskownice (sandboxing) dla komend shellowych.

---

## 2. Architektura Systemu

System zaprojektowany jest w oparciu o architekturę **Klient-Serwer (Local Daemon / Enterprise Server)** w technologii **Node.js (TypeScript)**.

```mermaid
flowchart TB
    subgraph UI ["Warstwa Prezentacji (Zed-like UX)"]
        ClientUI["Maestro Studio (Electron / Tauri / Web)"]
        Monaco["Fast Code Editor (Monaco/CodeMirror 6 + CRDT)"]
        FleetDashboard["Agent Fleet Dashboard (Orca Kanban + Terminal Multi-View)"]
        DiffViewer["Interactive 3-Way Diff & Merge Approval"]
    end

    subgraph Core ["Maestro Core Daemon (Node.js / TypeScript)"]
        API["Fastify / tRPC / WebSocket Server"]
        
        subgraph Collab ["Współpraca i Stan"]
            YjsServer["Yjs CRDT Engine (Realtime Sync)"]
            StateManager["Project & Session State (SQLite / Drizzle)"]
        end

        subgraph Orchestrator ["Silnik Wieloagentowy (Orca Core)"]
            TaskPlanner["Coordinator Agent (Decomposition)"]
            AgentPool["Agent Pool & Lifecycle Manager"]
            MessageBus["Agent Event Bus (Local Pub/Sub)"]
            WorktreeMgr["Git Worktree Manager (Branch Isolation)"]
        end

        subgraph Security ["Moduł Bezpieczeństwa Firmowego"]
            DLP["DLP & PII Masker (Gitleaks / Secret Scanner)"]
            PolicyEngine["RBAC & Action Authorization (HITL Guard)"]
            AuditLogger["Append-Only Secure Audit Log (JSONL/SQLite)"]
            SandboxMgr["Execution Sandbox (Docker / Firejail / Isolated Subprocess)"]
        end

        subgraph LLM ["Bramka Modeli (LLM Provider Gateway)"]
            LocalLLM["Local Engines (Ollama / vLLM / LocalAI)"]
            EnterpriseProxy["Corporate Proxy (Azure OpenAI / VPC Bedrock)"]
        end
    end

    ClientUI <-->|WebSocket / tRPC| API
    TaskPlanner --> WorktreeMgr
    AgentPool --> SandboxMgr
    SandboxMgr --> WorktreeMgr
    AgentPool --> DLP
    DLP --> LLM
    PolicyEngine --> AuditLogger
```

---

## 3. Kluczowe Elementy Architektury

### 3.1. Hybryda Orca + Zed

| Cecha | Źródło inspiracji | Implementacja w Maestro |
|---|---|---|
| **Izolacja pracy agentów** | Orca | Każdy agent tworzy i działa w osobnym **Git Worktree** (`.maestro/worktrees/<task-id>`), nie dotykając gałęzi roboczej programisty do momentu zatwierdzenia. |
| **Pulpit floty agentów** | Orca | Tablica Kanban zadań podagentów, równoległe terminale na żywo, wskaźniki zużycia tokenów i postępu. |
| **Szybkość edycji i CRDT** | Zed | Lekki edytor tekstowy oparty na **CodeMirror 6 / Monaco** zintegrowany z **Yjs**; programista widzi pisanie kodu przez agenta w czasie rzeczywistym z możliwością natychmiastowej korekty. |
| **Głęboki kontekst kodu** | Zed | Integracja z protokołem **LSP (Language Server Protocol)** uruchamianym lokalnie w Node.js, dającym agentom dostęp do definicji typów, błędów składniowych i nawigacji. |
| **Human-in-the-Loop** | Orca + Zed | Tryb półautonomiczny: agent proponuje commity i diffy; deweloper jednym skrótem klawiszowym akceptuje, modyfikuje lub odrzuca zmiany. |

---

### 3.2. Bezpieczeństwo w Sieci Firmowej (Enterprise Intranet)

Bezpieczeństwo jest fundamentem projektu Maestro dla zastosowań korporacyjnych:

1. **Zero External Data Leakage (Air-Gapped Ready):**
   - Domyślny tryb działania bez dostępu do otwartego internetu.
   - Całość komunikacji z modelami odbywa się wewnątrz sieci korporacyjnej (np. `http://ollama-cluster.internal:11434` lub prywatny endpoint Azure AI Foundry / AWS Bedrock VPC).
2. **Lokalna Ochrona Przed Wyciekiem Danych (DLP & Secret Scanner):**
   - Weryfikacja każdego promptu i payloadu narzędziowego przed przekazaniem do modelu.
   - Wykrywanie i maskowanie kluczy API, haseł, tokenów JWT, adresów IP i numerów kart/PESEL przed wysłaniem zapytania do LLM.
3. **Bezpieczne Środowisko Wykonawcze (Execution Sandboxing):**
   - Agenci wykonują polecenia bash/systemowe wyłącznie w kontenerach Docker, systemd-nspawn lub procesach izolowanych chroot/firejail z montowaniem tylko dedykowanego worktree w trybie copy-on-write.
   - Czarna lista niebezpiecznych poleceń (np. `curl` na zewnątrz, modyfikacja plików systemowych, `rm -rf /`).
4. **Niezmienialny Dziennik Audytowy (Immutable Audit Trail):**
   - Każde wywołanie narzędzia, odczyt pliku, wykonanie komendy i odpowiedź modelu jest logowana do pliku audytowego ze sumą kontrolną HMAC.

---

### 3.3. Silnik Wieloagentowy (Multi-Agent System)

1. **Koordynator (Planner Agent):**
   - Przyjmuje zadanie wysokiego poziomu (np. GitHub Issue, wymaganie biznesowe).
   - Generuje plan w postaci DAG (Directed Acyclic Graph) z zależnościami między podzadaniami.
2. **Wyspecjalizowani Agenci (Worker Agents):**
   - **Research / Context Agent:** Analizuje bazę kodu za pomocą indeksu symboli (AST/LSP) i semantycznego wyszukiwania (SQLite-vec / Embeddings lokalne).
   - **Backend / Logic Agent:** Implementuje logikę w odizolowanym worktree.
   - **Frontend / UI Agent:** Przygotowuje komponenty i widoki.
   - **QA / Test Agent:** Uruchamia testy jednostkowe/integracyjne w piaskownicy i zgłasza usterki z powrotem do agenta programisty.
   - **Reviewer / Security Agent:** Przeprowadza statyczną analizę kodu (lintery, audyt bezpieczeństwa) przed scaleniem gałęzi.
3. **Arbiter Scalania (Merge Coordinator):**
   - Automatycznie rozwiązuje proste konflikty i przedstawia użytkownikowi interaktywny 3-kierunkowy podgląd zmian (3-way merge diff).

---

## 4. Stos Technologiczny

- **Runtime:** Node.js 22 LTS (z natywnym wsparciem dla TypeScript i WebSocketów)
- **Język:** TypeScript (Strict mode, Type-safety end-to-end)
- **Serwer & API:** Fastify + tRPC / WebSocket dla ultra-niskich opóźnień
- **Współpraca w czasie rzeczywistym:** Yjs (CRDT) + y-websocket
- **Edytor i UI:**
  - Frontend: React / Vite lub Solid.js (lekki, wysoka wydajność renderowania)
  - Silnik edycji: CodeMirror 6 lub Monaco Editor z rozszerzeniami Zed-like
  - Desktop: Tauri v2 lub Electron (Tauri rekomendowane ze względu na mniejszy narzut pamięci i bezpieczeństwo)
- **Baza Danych & Stan Lokalny:** SQLite (poprzez `better-sqlite3` i `drizzle-orm`) + `sqlite-vec` (dla lokalnych wektorów/RAG)
- **Integracja Narzędzi Deweloperskich:**
  - Git: `simple-git` lub `isomorphic-git` + natywne polecenia worktree
  - Terminal: `node-pty` do obsługi interaktywnych sesji terminala dla agentów
  - LSP: Klient Language Server Protocol w Node.js komunikujący się z lokalnymi serwerami językowymi (tsserver, gopls, rust-analyzer itp.)
- **Izolacja / Sandbox:** Dockerode (kontrola lokalnego demona Docker) / unshare / nsjail

---

## 5. Struktura Projektu (Monorepo)

```
Maestro/
├── apps/
│   ├── desktop/              # Aplikacja okienkowa (Tauri / Electron + React/Solid)
│   └── web/                  # Webowy interfejs dla zespołów (opcjonalny dashboard intranetowy)
├── packages/
│   ├── core/                 # Główny silnik Node.js (Daemon, state, orchestration)
│   ├── agent-runtime/        # Środowisko uruchomieniowe agentów (Worktrees, terminal pty, sandboxing)
│   ├── lsp-client/           # Adapter Language Server Protocol dla ekstrakcji kontekstu
│   ├── security-dlp/         # Moduł ochrony danych, audytu i weryfikacji uprawnień (Enterprise Guard)
│   ├── llm-gateway/          # Klient Ollama, vLLM, OpenAI-compatible proxy z fallbackiem i cache
│   ├── crdt-collab/          # Warstwa synchronizacji Yjs dokumentów i edycji na żywo
│   └── shared/               # Wspólne typy TypeScript, schematy walidacji (Zod)
├── docker/                   # Szablony kontenerów piaskownicy (agent-sandbox)
├── docs/                     # Dokumentacja architektury, przewodniki wdrożeniowe
├── package.json              # Workspace root (pnpm/npm workspaces)
├── tsconfig.json             # Globalna konfiguracja TypeScript
└── README.md
```

---

## 6. Harmonogram i Plan Implementacji (Fazy i Zadania GitHub)

### [Faza 1: Fundamenty i Szkielet Rdzenia](https://github.com/tarze07/Maestro/milestone/1)
- [ ] [#2: Inicjalizacja struktury Monorepo (pnpm workspaces, TypeScript, tooling)](https://github.com/tarze07/Maestro/issues/2)
- [ ] [#3: Implementacja modułu packages/llm-gateway (Lokalne modele i bramki on-prem)](https://github.com/tarze07/Maestro/issues/3)
- [ ] [#4: Implementacja modułu packages/agent-runtime/worktree (Zarządzanie Git Worktrees)](https://github.com/tarze07/Maestro/issues/4)
- [ ] [#5: Implementacja silnika stanu i sesji w SQLite (packages/core/state)](https://github.com/tarze07/Maestro/issues/5)

### [Faza 2: Silnik Wieloagentowy i Piaskownica (Orca Parity)](https://github.com/tarze07/Maestro/milestone/2)
- [ ] [#6: Orkiestrator pętli ReAct/Plan-Execute i dekompozycji zadań (packages/core/orchestrator)](https://github.com/tarze07/Maestro/issues/6)
- [ ] [#7: Wirtualny terminal node-pty z obsługą wielu równoległych agentów](https://github.com/tarze07/Maestro/issues/7)
- [ ] [#8: Moduł piaskownicy wykonawczej (Sandboxing: Docker / Izolacja procesów)](https://github.com/tarze07/Maestro/issues/8)
- [ ] [#9: Zestaw narzędzi agenta (Agent Tools: File I/O, Test Runner, Linter, Git Diff)](https://github.com/tarze07/Maestro/issues/9)

### [Faza 3: Bezpieczeństwo Korporacyjne i Audyt (Enterprise Hardening)](https://github.com/tarze07/Maestro/milestone/3)
- [ ] [#10: Moduł DLP i skanowania sekretów przed wysłaniem promptu (packages/security-dlp)](https://github.com/tarze07/Maestro/issues/10)
- [ ] [#11: Niezmienialny rejestr audytowy zdarzeń i komend (Audit Trail z HMAC)](https://github.com/tarze07/Maestro/issues/11)
- [ ] [#12: Mechanizm weryfikacji i autoryzacji akcji Human-in-the-Loop (HITL Guard)](https://github.com/tarze07/Maestro/issues/12)

### [Faza 4: Interfejs Użytkownika i Kolaboracja (Zed Parity)](https://github.com/tarze07/Maestro/milestone/4)
- [ ] [#13: Klient edycyjny z obsługą CRDT (CodeMirror 6 / Tauri v2 + Yjs)](https://github.com/tarze07/Maestro/issues/13)
- [ ] [#14: Pulpit floty agentów (Agent Fleet Dashboard & Multi-Terminal View)](https://github.com/tarze07/Maestro/issues/14)
- [ ] [#15: Interaktywny 3-kierunkowy podgląd i zatwierdzanie zmian (3-Way Diff & Merge Approval)](https://github.com/tarze07/Maestro/issues/15)
- [ ] [#16: Integracja z lokalnym serwerem LSP (Language Server Protocol Client)](https://github.com/tarze07/Maestro/issues/16)

### [Faza 5: Testy Wydajnościowe i Wdrożenie Intranetowe](https://github.com/tarze07/Maestro/milestone/5)
- [ ] [#17: Weryfikacja szczelności środowiska air-gapped i braku transmisji WAN](https://github.com/tarze07/Maestro/issues/17)
- [ ] [#18: Przygotowanie pakietów wdrożenia intranetowego (Docker Compose & Helm)](https://github.com/tarze07/Maestro/issues/18)
- [ ] [#19: Dokumentacja techniczna wdrożenia i procedury compliance (ISO 27001 / SOC2 / RODO)](https://github.com/tarze07/Maestro/issues/19)

---

## 7. Weryfikacja Sukcesu i Metryki

1. **Bezpieczeństwo sieciowe:** 0 bajtów wysłanych poza zdefiniowane wewnętrzne IP/domeny firmowe (weryfikacja `tcpdump`/firewallem).
2. **Izolacja agentów:** Równoległa praca minimum 4 agentów na jednym repozytorium bez kolizji w plikach i stanie git.
3. **Responsywność (Zed feel):** Opóźnienie wprowadzania tekstu w edytorze < 16ms (60 FPS), synchronizacja stanu agenta < 50ms w sieci lokalnej.
4. **Wygoda pracy (Orca feel):** Czas od zatwierdzenia planu przez koordynatora do wygenerowania gotowego PR z testami skrócony o min. 60% względem pracy sekwencyjnej.
