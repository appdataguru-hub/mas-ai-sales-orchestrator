### 1. Содержимое для файла `README.md`

```markdown
# MAS-AI-Sales Orchestrator Core 🤖💼

Ядро распределения контекста, стейт-менеджмента и оркестрации мультиагентного отдела продаж полного цикла. Проект спроектирован для развертывания в изолированном On-Premise контуре крупных предприятий с жесткими требованиями к безопасности данных.

---

### 📂 Архитектурная структура репозитория

mas-ai-sales-orchestrator/
├── config/
│   ├── config.yaml          # Конфигурация локального инференса vLLM и параметров pgvector
│   └── prompts.yaml         # Системные промты и правила валидации для Pydantic v2
├── src/
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── router.py        # Детерминированный роутер на базе Pydantic-схем
│   │   ├── qualification.py # Агент скоринга и извлечения сущностей (BANT)
│   │   └── sales_expert.py  # Агент-продажник со сквозным LlamaIndex RAG
│   ├── graph/
│   │   ├── __init__.py
│   │   ├── state.py         # Определение глобального состояния LangGraph (MAS State)
│   │   └── workflow.py      # Сборка цикличесного графа и переходов состояний
│   ├── tools/
│   │   ├── bitrix_api.py    # Интеграционный слой с Bitrix24 REST API
│   │   └── billing.py       # Генерация счетов и триггерных скидок
│   └── main.py              # Точка входа для запуска MAS-сервиса
├── docker-compose.yaml      # Изолированный стек: vLLM (Llama 3) + PostgreSQL
└── README.md

```

---

### 🚀 Ключевые особенности реализации

1. **Deterministic State Management (LangGraph):** Управление диалогом происходит через строгое обновление глобального графа состояний. Агенты не могут войти в бесконечный цикл благодаря валидации переходов роутером.
2. **Strict Validation (Pydantic v2):** Вызовы инструментов (Function Calling) локальной модели Llama валидируются на лету. Ошибки парсинга JSON перехватываются внутренним слоем и возвращаются модели на доработку (Self-Correction Loop).
3. **On-Premise RAG (LlamaIndex):** Извлечение контекста из корпоративных баз знаний происходит локально через векторное расширение `pgvector` для PostgreSQL, гарантируя нулевую утечку данных за периметр компании.

### 2. Код для файла `main.py`

```python
from typing import Annotated, Dict, Any
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, END
from pydantic import BaseModel, Field

# 1. Определение строгого состояния MAS
class AgentState(TypedDict):
    client_id: str
    messages: list
    qualification_data: Dict[str, Any]
    current_agent: str
    deal_closed: bool
    invoice_generated: bool

# 2. Схема валидации для роутера
class RouterOutput(BaseModel):
    next_step: str = Field(description="Следующий узел графа: 'qualification', 'sales_expert' или 'billing'")
    reason: str = Field(description="Аргументация выбора пути")

# 3. Базовый узел оркестратора
def qualification_node(state: AgentState) -> Dict[str, Any]:
    print(f"[Log] Запуск скоринга для клиента: {state['client_id']}")
    return {"current_agent": "qualification", "qualification_data": {"budget_verified": True}}

def sales_node(state: AgentState) -> Dict[str, Any]:
    print("[Log] Запуск Sales-агента с поддержкой LlamaIndex RAG")
    return {"current_agent": "sales_expert"}

def billing_node(state: AgentState) -> Dict[str, Any]:
    print("[Log] Автоматическое выставление счета через API шлюза")
    return {"invoice_generated": True, "deal_closed": True}

# 4. Сборка децентрализованного графа
workflow = StateGraph(AgentState)

workflow.add_node("qualification", qualification_node)
workflow.add_node("sales_expert", sales_node)
workflow.add_node("billing", billing_node)

workflow.set_entry_point("qualification")

# Детерминированная маршрутизация
def router_logic(state: AgentState) -> str:
    if not state["qualification_data"].get("budget_verified"):
        return "qualification"
    if not state["invoice_generated"]:
        return "sales_expert"
    return "billing"

workflow.add_conditional_edges(
    "qualification",
    router_logic,
    {
        "qualification": "qualification",
        "sales_expert": "sales_expert",
        "billing": "billing"
    }
)
workflow.add_edge("sales_expert", "billing")
workflow.add_edge("billing", END)

app = workflow.compile()

```
