# AI Agent Skills

Коллекция готовых навыков (skills) для ИИ-агентов, совместимых с платформами вроде **Codex CLI**, **Claude Code**, **Cline**, **Continue.dev**, **GitHub Copilot CLI** и другими AI-ассистентами, поддерживающими формат AgentSkills.

## 📋 Навигация по навыкам

| Навык | Описание | Стек | Сложность |
|-------|----------|------|-----------|
| [java-telegram-bots](java-telegram-bots/SKILL.md) | Создание Telegram-ботов на Java + Spring Boot с роутингом команд, state management, async-операциями и тестированием | Java 17+, Spring Boot 3.x, TelegramBots v10 | Средний |

## 🚀 Быстрый старт

### Установка

```bash
# Для Cline / Continue.dev / Codex CLI
# Скопировать нужный навык в директорию skills вашего AI-агента:
cp -r java-telegram-bots ~/.config/opencode/skills/
# Или для ~/.agents (кросс-платформенный путь):
cp -r java-telegram-bots ~/.agents/skills/
```

### Использование

Навыки активируются автоматически, когда ИИ-агент распознаёт задачу, соответствующую описанию навыка. Можно также вызвать вручную:

```bash
# Попросить агента использовать конкретный навык:
# "Use the java-telegram-bots skill to create a new bot handler"
```

## 📂 Структура репозитория

```
agent-skills/
├── java-telegram-bots/
│   └── SKILL.md
└── ... (другие навыки)
```

Каждый навык — это директория с файлом `SKILL.md`, содержащим:

| Секция | Описание |
|--------|----------|
| **YAML frontmatter** | Имя, описание, теги, версия, связанные навыки |
| **Overview** | Краткое описание и когда применять |
| **Implementation** | Пошаговые инструкции с примерами кода |
| **Decision Rules** | Когда выбирать тот или иной подход |
| **Common Mistakes** | Типичные ошибки и их решения |
| **Resources** | Ссылки на документацию |

## 📖 Как добавить новый навык

1. Создайте директорию `skills/<название-навыка>/`
2. Напишите `SKILL.md` в формате AgentSkills (YAML frontmatter + Markdown)
3. Убедитесь, что `description` начинается с `Use when...` и описывает триггеры активации
4. Добавьте примеры кода и decision rules
5. Следуйте принципу TDD для документации: проверьте, что агент без навыка ошибается, а с навыком — нет

Подробнее: [writing-skills/SKILL.md](writing-skills/SKILL.md)

## 🧠 Формат навыка (AgentSkills Specification)

```yaml
---
name: skill-name-with-hyphens
description: Use when [конкретные условия и симптомы для активации]
version: 1.0.0
category: language | framework | testing | devops
tags: ['java', 'spring-boot', ...]
related-skills: ['skill-1', 'skill-2']
updated: 2026-07-05
status: active
---

# Skill Name

...
```

## 📄 Лицензия

MIT
