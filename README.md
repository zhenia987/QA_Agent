# QA Agent

Мультиагентная QA-система на базе [OpenCode](https://opencode.ai/docs/). Она берёт требования из **Confluence**, анализирует их, создаёт задачи в **Jira** и тестовую модель в **TestIt**. На каждом этапе проверяется Quality Gate и сохраняется сквозная трассировка «требование → задача → тест-кейс».

\---

## Как это работает

```mermaid
flowchart LR
    U(\\\[Пользователь]) <--> O\\\[QA Orchestrator Agent]
    O -->|1| R\\\[Requirement Reviewer Agent<br/>skill: requirement-review]
    O -->|2| T\\\[Task Designer Agent<br/>skill: task-design]
    O -->|3| D\\\[Test Designer Agent<br/>skill: test-design]
    R -. читает .-> C\\\[(Confluence)]
    T -. создаёт задачи .-> J\\\[(Jira)]
    D -. создаёт тест-кейсы .-> Q\\\[(TestIt)]
    R --> A1\\\[/artifacts/requirement-review/]
    T --> A2\\\[/artifacts/task-design/]
    D --> A3\\\[/artifacts/test-design/]
    O --> A4\\\[/artifacts/final-report/]
```

|Агент|Файл|Skill|Что делает|
|-|-|-|-|
|QA Orchestrator Agent|`.opencode/agents/qa-orchestrator-agent.md`|—|Ведёт Workflow, раздаёт этапы субагентам, проверяет Quality Gate, собирает итоговый отчёт. Только он общается с пользователем|
|Requirement Reviewer Agent|`.opencode/agents/requirement-review-agent.md`|`requirement-review`|Читает требования в Confluence, присваивает им ID, находит неоднозначности и противоречия|
|Task Designer Agent|`.opencode/agents/task-design-agent.md`|`task-design`|Создаёт в Jira задачи со ссылками на ID требований и не допускает дубликатов|
|Test Designer Agent|`.opencode/agents/test-design-agent.md`|`test-design`|Создаёт тест-кейсы в TestIt со ссылками на требования и задачи Jira|

Этапы выполняются **строго по очереди**: review → task-design → test-design. Следующий этап запускается, только если у предыдущего статус Quality Gate равен `passed`. Если статус `blocked` (противоречия, нет данных, недоступен внешний сервис), оркестратор останавливается и передаёт вопрос пользователю.

Общие правила для всех агентов (Context Management, ограничения, хранение Artifacts, эскалация, формат ответа) лежат в [`AGENTS.md`](./AGENTS.md).

\---

## Структура репозитория

```
QA\\\_Agent/
├── AGENTS.md                        # общие правила для всех агентов
├── .opencode/
│   ├── agents/                      # описания агентов (имя файла = имя агента)
│   │   ├── qa-orchestrator-agent.md
│   │   ├── requirement-review-agent.md
│   │   ├── task-design-agent.md
│   │   └── test-design-agent.md
│   ├── skills/                      # методики выполнения этапов
│   │   ├── requirement-review/SKILL.md
│   │   ├── task-design/SKILL.md
│   │   └── test-design/SKILL.md
│   └── tools/
│       └── confluence.ts            # custom tool: получить страницу Confluence по pageId
└── artifacts/                       # результаты этапов: /artifacts/<этап>/<версия>.md
    ├── requirement-review/1.md
    ├── task-design/1.md
    ├── test-design/1.md
    └── final-report/1.md
```

\---

## Требования

* **OpenCode** ([установка](https://opencode.ai/docs/));
* **Node.js 20+** (нужен для custom tool `confluence`);
* API-ключ LLM-провайдера, подключённый в OpenCode;
* **Atlassian Cloud** (Confluence и Jira) и API-токен Atlassian;
* **TestIt** на платном тарифе с доступом к [Test IT MCP](https://docs.testit.software/user-guide/mcp/mcp-setup.html) и API-токен TestIt ([работа с API](https://docs.testit.software/user-guide/work-with-api.html)).

\---

## Установка и настройка

### 1\. Установить OpenCode

```bash
# macOS / Linux
curl -fsSL https://opencode.ai/install | bash

# или через npm (в том числе Windows)
npm install -g opencode-ai
```

На Windows лучше работать через WSL.

### 2\. Клонировать репозиторий

```bash
git clone https://github.com/zhenia987/QA\\\_Agent.git
cd QA\\\_Agent
```

### 3\. Подключить LLM-провайдера

Запустите `opencode` в каталоге проекта, выполните команду `/connect` и выберите провайдера. Потом введите API-ключ.

> \\\*\\\*Не запускайте `/init`.\\\*\\\* Эта команда сгенерирует новый `AGENTS.md` и перезапишет общие правила агентов.

### 4\. Задать переменные окружения

Custom tool `confluence` читает учётные данные из переменных окружения:

|Переменная|Пример|Назначение|
|-|-|-|
|`CONFLUENCE\\\_URL`|`https://your-site.atlassian.net`|Адрес сайта Atlassian **без** `/wiki` и без `/` на конце|
|`CONFLUENCE\\\_EMAIL`|`user@company.com`|Email аккаунта Atlassian|
|`CONFLUENCE\\\_API\\\_TOKEN`|`ATATT3x...`|[API-токен Atlassian](https://id.atlassian.com/manage-profile/security/api-tokens)|
|`TESTIT\\\_URL`|`https://your-company.testit.software`|Адрес TestIt без `/` на конце (для Test IT MCP)|
|`TESTIT\\\_API\\\_TOKEN`|`NxNBZ3A5...`|API-токен TestIt (профиль пользователя → API-токены)|

Linux / macOS:

```bash
export CONFLUENCE\\\_URL="https://your-site.atlassian.net"
export CONFLUENCE\\\_EMAIL="user@company.com"
export CONFLUENCE\\\_API\\\_TOKEN="your\\\_atlassian\\\_token"
export TESTIT\\\_URL="https://your-company.testit.software"
export TESTIT\\\_API\\\_TOKEN="your\\\_testit\\\_token"
```

Windows (PowerShell):

```powershell
$env:CONFLUENCE\\\_URL="https://your-site.atlassian.net"
$env:CONFLUENCE\\\_EMAIL="user@company.com"
$env:CONFLUENCE\\\_API\\\_TOKEN="your\\\_atlassian\\\_token"
$env:TESTIT\\\_URL="https://your-company.testit.software"
$env:TESTIT\\\_API\\\_TOKEN="your\\\_testit\\\_token"
```

> Не коммитьте токены в репозиторий. Если используете файл `.env`, добавьте его в `.gitignore`.

Проверить доступ к Confluence можно так (`<PAGE\\\_ID>` — ID страницы с требованиями):

```bash
curl -u "$CONFLUENCE\\\_EMAIL:$CONFLUENCE\\\_API\\\_TOKEN" \\\\
  "$CONFLUENCE\\\_URL/wiki/api/v2/pages/<PAGE\\\_ID>?body-format=storage"
```

### 5\. Подключить Jira и TestIt через MCP

Для Jira и TestIt в репозитории нет своих инструментов. Агенты работают с ними через MCP-серверы, и их нужно один раз описать в конфиге OpenCode. Создайте в корне проекта файл `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "atlassian": {
      "type": "remote",
      "url": "https://mcp.atlassian.com/v2/mcp",
      "enabled": true
    },
    "testit": {
      "type": "remote",
      "url": "{env:TESTIT\\\_URL}/mcp",
      "enabled": true,
      "oauth": false,
      "headers": {
        "X-baseurl": "{env:TESTIT\\\_URL}",
        "X-token": "{env:TESTIT\\\_API\\\_TOKEN}"
      }
    }
  }
}
```

* **Jira** подключается через официальный [Atlassian Rovo MCP Server](https://github.com/atlassian/atlassian-mcp-server). Авторизация идёт через OAuth: выполните `opencode mcp auth atlassian` и войдите в браузере.
* **TestIt** подключается через официальный [Test IT MCP](https://docs.testit.software/user-guide/mcp/mcp-setup.html). Это удалённый сервер по адресу `<адрес TestIt>/mcp`. Адрес TestIt передаётся в заголовке `X-baseurl`, API-токен в заголовке `X-token`. Test IT MCP доступен только на платных тарифах.

> Для отладки лучше завести отдельный проект TestIt и отдельный токен с ограниченными правами: часть команд Test IT MCP изменяет данные (создаёт, архивирует кейсы, меняет статусы).

Если вы уже настроили эти MCP-серверы глобально в `\\\~/.config/opencode/opencode.json`, этот шаг можно пропустить.

### 6\. Проверить, что всё загрузилось

```bash
opencode agent list   # должны быть видны 4 агента из .opencode/agents
opencode mcp list     # atlassian и testit в статусе connected
```

\---

## Запуск

### Интерактивный режим

```bash
cd QA\\\_Agent
opencode
```

1. Переключитесь на агента **`qa-orchestrator-agent`** клавишей **Tab**.
2. Отправьте задачу и укажите все обязательные входные данные:

```
Проанализируй требования из Confluence (pageId: 3997697),
создай задачи в Jira в проекте SCRUM
и тестовую модель в TestIt в проекте QAI06.
```

Дальше оркестратор работает сам:

1. проверяет, что все входные данные на месте, и запрашивает недостающие;
2. запускает `requirement-review-agent` → проверяет Quality Gate;
3. если в требованиях есть противоречия или открытые вопросы, задаёт их вам и ждёт ответа;
4. запускает `task-design-agent` → проверяет Quality Gate;
5. запускает `test-design-agent` → проверяет Quality Gate;
6. проверяет трассировку «требование → задача → тест-кейс» и сохраняет итоговый отчёт.

Любой субагент можно вызвать и напрямую, упомянув его через `@`, например `@requirement-review-agent`. Так удобно перезапускать отдельный этап.

### Неинтерактивный режим

```bash
opencode run --agent qa-orchestrator-agent \\\\
  "Проанализируй требования из Confluence (pageId: 3997697), создай задачи в Jira в проекте SCRUM и тестовую модель в TestIt в проекте QAI06."
```

В этом режиме оркестратор не сможет задать уточняющие вопросы. Если в требованиях найдутся противоречия, Workflow закончится со статусом `blocked`.

\---

## Входные данные

|Параметр|Обязательный|Пример|
|-|-|-|
|Страница(ы) требований в Confluence (`pageId`)|да|`3997697`|
|Проект Jira (ключ)|да|`SCRUM`|
|Проект TestIt|да|`QAI06`|
|Дополнительные бизнес-правила, ограничения, критерии приёмки|нет|текстом в промте|

`pageId` берётся из URL страницы Confluence: `https://your-site.atlassian.net/wiki/spaces/XXX/pages/<pageId>/...`.

\---

## Результаты

Каждый этап сохраняет свой Artifact в `artifacts/<этап>/<версия>.md`. Старые версии остаются, новая версия получает следующий номер.

|Artifact|Путь|Содержимое|
|-|-|-|
|Requirement Review|`artifacts/requirement-review/<v>.md`|Требования с ID (FR, BR, CON, COND, AC), источники, проблемы, открытые вопросы|
|Task Design|`artifacts/task-design/<v>.md`|Задачи Jira, матрица «требование → задача», непокрытые требования|
|Test Model|`artifacts/test-design/<v>.md`|Тест-кейсы TestIt, матрица «требование → задача → тест-кейс»|
|Final Report|`artifacts/final-report/<v>.md`|Статусы этапов, созданные объекты, трассировка, открытые вопросы|

В каждом Artifact указаны версии входных Artifacts и статус Quality Gate: `passed`, `failed` или `blocked`.

В каталоге `artifacts/` лежит пример полного прогона по требованиям демо-приложения «GameDay» (Confluence page `3997697`, проект Jira `SCRUM`, проект TestIt `QAI06`).

\---

## Решение проблем

|Симптом|Причина и решение|
|-|-|
|`Не заданы переменные окружения Confluence`|Не заданы `CONFLUENCE\\\_URL`, `CONFLUENCE\\\_EMAIL` или `CONFLUENCE\\\_API\\\_TOKEN` в той сессии терминала, из которой запущен `opencode`|
|`Confluence API error: 401`|Неверный email или токен Atlassian|
|`Confluence API error: 404`|Неверный `pageId`, нет доступа к странице, или `CONFLUENCE\\\_URL` указан с `/wiki` на конце|
|Агенты не видны в `opencode agent list`|`opencode` запущен не из корня репозитория|
|Ошибка импорта `@opencode-ai/plugin` в `confluence.ts`|Выполните `npm install @opencode-ai/plugin` в каталоге `.opencode`|
|Jira недоступна|Проверьте `opencode mcp list` и повторите `opencode mcp auth atlassian`|
|TestIt недоступен|Проверьте `opencode mcp list`, значения `TESTIT\\\_URL` (без `/` на конце) и `TESTIT\\\_API\\\_TOKEN`, а также что Test IT MCP включён в вашем тарифе|
|Этап завершился со статусом `blocked`|Это ожидаемое поведение. Причина записана в Artifact этапа и в ответе оркестратора. Ответьте на вопросы и попросите оркестратора продолжить|

\---

## Ограничения

* Агенты **не удаляют и не изменяют** существующие задачи Jira и тест-кейсы TestIt, а Confluence только читают.
* Ограничения доступа к инструментам сейчас описаны текстом в инструкциях агентов. Чтобы закрепить их технически, добавьте в frontmatter агентов блок `permission` ([документация](https://opencode.ai/docs/agents/)).



