---
name: java-telegram-bots
description: Use when building Telegram bots on Java with Spring Boot — implementing command routing, update processing, state persistence, async operations, bot testing, or setting up Telegram Bot API integration
version: 1.0.0
category: framework
tags: ['java', 'spring-boot', 'telegram', 'telegram-bots', 'long-polling', 'webhook']
related-skills: ['moai-lang-java', 'spring-data-jpa', 'unit-test-utility-methods']
updated: 2026-07-05
status: active
---

# Java Telegram Bots

## Quick Reference (30 seconds)

Expert guide for building production-grade Telegram bots in Java with Spring Boot using the TelegramBots v10 API.

**Auto-Triggers:** Java files with Telegram annotations, `pom.xml`/`build.gradle` with telegram dependencies, commands mentioning Telegram bot.

### Core Architecture Pipeline

```
Telegram
  ↓
Ingress (Long Polling / Webhook)
  ↓
TelegramClient (OkHttp)
  ↓
SpringLongPollingBot :: consume(Update)
  ↓
Dispatcher / CommandRouter
  ↓
CommandHandlers
  ↓
Services (business logic, AI, async)
  ↓
Repository (JPA persistence)
```

### Key Interfaces (TelegramBots v10 API)

| Interface | Responsibility |
|-----------|---------------|
| `SpringLongPollingBot` | Spring Boot auto-discovered bot — provides `getBotToken()` + `getUpdatesConsumer()` |
| `LongPollingSingleThreadUpdateConsumer` | Single-update `consume(Update)` callback |
| `TelegramClient` | Executes API calls (`OkHttpTelegramClient` bean auto-configured by starter) |
| `LongPollingUpdateConsumer` | Low-level update consumer interface |

---

## Setup

### Maven Dependencies

```xml
<dependency>
  <groupId>org.telegram</groupId>
  <artifactId>telegrambots-springboot-longpolling-starter</artifactId>
  <version>10.0.0</version>
</dependency>
<dependency>
  <groupId>org.telegram</groupId>
  <artifactId>telegrambots-client</artifactId>
  <version>10.0.0</version>
</dependency>
```

### application.yaml

```yaml
app:
  telegram:
    bot-token: ${TELEGRAM_BOT_TOKEN}
    bot-username: ${TELEGRAM_BOT_USERNAME}

spring:
  ai:
    openai:
      base-url: http://localhost:1234
      api-key: dummy
      chat:
        options:
          model: mistral
```

### Environment Variables

```env
TELEGRAM_BOT_TOKEN=123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11
TELEGRAM_BOT_USERNAME=MyAmazingBot
```

### Minimal Bot Class

```java
@Component
public class MyAmazingBot implements SpringLongPollingBot, LongPollingSingleThreadUpdateConsumer {
    private final TelegramClient telegramClient;

    public MyAmazingBot(@Value("${app.telegram.bot-token}") String botToken) {
        telegramClient = new OkHttpTelegramClient(botToken);
    }

    @Override
    public String getBotToken() {
        return botToken;
    }

    @Override
    public LongPollingUpdateConsumer getUpdatesConsumer() {
        return this;
    }

    @Override
    public void consume(Update update) {
        if (update.hasMessage() && update.getMessage().hasText()) {
            var message = SendMessage.builder()
                .chatId(update.getMessage().getChatId())
                .text(update.getMessage().getText())
                .build();
            try {
                telegramClient.execute(message);
            } catch (TelegramApiException e) {
                log.error("Failed to send message", e);
            }
        }
    }

    @AfterBotRegistration
    public void afterRegistration(BotSession botSession) {
        log.info("Bot registered, running state: {}", botSession.isRunning());
    }
}
```

---

## Implementation Guide (5 minutes)

### 1. Bot Class Structure

```java
@Component
@Slf4j
public class TarotistBot implements SpringLongPollingBot, LongPollingSingleThreadUpdateConsumer {
    private final TelegramClient telegramClient;
    private final CommandRouter commandRouter;

    public TarotistBot(
            @Value("${app.telegram.bot-token}") String botToken,
            CommandRouter commandRouter) {
        telegramClient = new OkHttpTelegramClient(botToken);
        this.commandRouter = commandRouter;
    }

    @Override
    public String getBotToken() { return botToken; }

    @Override
    public LongPollingUpdateConsumer getUpdatesConsumer() { return this; }

    @Override
    public void consume(Update update) {
        try {
            commandRouter.route(update);
        } catch (Exception e) {
            log.error("Error processing update: {}", update.getUpdateId(), e);
            sendErrorMessage(update);
        }
    }

    public void execute(SendMessage message) {
        try {
            telegramClient.execute(message);
        } catch (TelegramApiException e) {
            log.error("Failed to execute SendMessage", e);
        }
    }

    private void sendErrorMessage(Update update) {
        if (update.hasMessage()) {
            execute(SendMessage.builder()
                .chatId(update.getMessage().getChatId())
                .text("Произошла внутренняя ошибка. Попробуйте позже.")
                .build());
        }
    }
}
```

### 2. Command Routing

```java
@Component
public class CommandRouter {
    private final Map<Command, CommandHandler> handlers;

    public CommandRouter(List<CommandHandler> handlerList) {
        handlers = handlerList.stream()
            .collect(Collectors.toMap(
                CommandHandler::getCommand,
                Function.identity()
            ));
    }

    public void route(Update update) {
        if (!update.hasMessage() || !update.getMessage().hasText()) return;

        String text = update.getMessage().getText().trim();
        Command command = Command.fromText(text);

        CommandHandler handler = handlers.get(command);
        if (handler != null) {
            handler.handle(update);
        } else {
            handleUnknown(update);
        }
    }

    private void handleUnknown(Update update) {
        var message = SendMessage.builder()
            .chatId(update.getMessage().getChatId())
            .text("Неизвестная команда. Используйте /help для списка команд.")
            .build();
        // send via bot or telegramClient
    }
}
```

```java
public enum Command {
    START("/start"),
    HELP("/help"),
    PROFILE("/profile"),
    HOROSCOPE("/horoscope"),
    TAROT("/tarot"),
    SUBSCRIPTION("/subscription");

    private final String text;

    Command(String text) { this.text = text; }

    public static Command fromText(String text) {
        for (Command cmd : values()) {
            if (cmd.text.equals(text)) return cmd;
        }
        return null;
    }
}
```

```java
public interface CommandHandler {
    Command getCommand();
    void handle(Update update);
}
```

### 3. Handler Implementation

```java
@Component
@RequiredArgsConstructor
public class StartCommandHandler implements CommandHandler {
    private final TarotistBot tarotistBot;

    @Override
    public Command getCommand() { return Command.START; }

    @Override
    public void handle(Update update) {
        long chatId = update.getMessage().getChatId();
        String userName = update.getMessage().getFrom().getFirstName();

        var message = SendMessage.builder()
            .chatId(chatId)
            .text(String.format("""
                Привет, %s! ✨
                
                Я — таролог-астролог. Я помогу тебе заглянуть в будущее.
                
                Доступные команды:
                /horoscope — твой гороскоп на сегодня
                /tarot — расклад Таро на неделю
                /profile — мой профиль
                /subscription — управление подпиской
                /help — помощь
                """, userName))
            .parseMode("HTML")
            .build();

        tarotistBot.execute(message);
    }
}
```

### 4. State Management with Redis

For multi-step conversations or persistent user state across restarts:

```java
@Service
@RequiredArgsConstructor
public class UserStateService {
    private final RedisTemplate<String, Object> redisTemplate;
    private static final String STATE_PREFIX = "user_state:";

    public Map<String, Object> getState(Long userId) {
        Map<Object, Object> entries = redisTemplate.opsForHash()
            .entries(STATE_PREFIX + userId);
        return entries.entrySet().stream()
            .collect(Collectors.toMap(
                e -> e.getKey().toString(),
                Map.Entry::getValue
            ));
    }

    public void setState(Long userId, String key, Object value) {
        redisTemplate.opsForHash()
            .put(STATE_PREFIX + userId, key, value);
    }

    public void clearState(Long userId) {
        redisTemplate.delete(STATE_PREFIX + userId);
    }
}
```

**Redis dependencies:**

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### 5. Async Operations (Virtual Threads)

For long-running tasks (AI calls, external API, report generation):

```java
@Service
@RequiredArgsConstructor
public class TarotReadingService {
    private final TarotCardRepository cardRepository;
    private final AiService aiService;

    public CompletableFuture<TarotReading> generateReadingAsync(Long userId) {
        return CompletableFuture.supplyAsync(() -> generateReading(userId));
    }

    public TarotReading generateReading(Long userId) {
        List<TarotCard> cards = selectRandomCards(3);
        String interpretation = aiService.generateInterpretation(userId, cards);
        return new TarotReading(cards, interpretation);
    }

    private List<TarotCard> selectRandomCards(int count) {
        List<TarotCard> allCards = cardRepository.findAll();
        Collections.shuffle(allCards);
        return allCards.stream()
            .limit(count)
            .map(card -> card.withRandomReversal())
            .toList();
    }
}
```

### 6. Structured Logging

```xml
<dependency>
  <groupId>net.logstash.logback</groupId>
  <artifactId>logstash-logback-encoder</artifactId>
</dependency>
```

```java
@Slf4j
public class LoggingHelper {
    public static MDC putContext(Update update, String operation) {
        MDC.put("traceId", UUID.randomUUID().toString());
        MDC.put("telegramUserId",
            String.valueOf(update.getMessage().getFrom().getId()));
        MDC.put("operationName", operation);
        return MDC.getCopyOfContextMap();
    }

    public static void clearContext() {
        MDC.clear();
    }
}
```

### 7. Error Handling

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    @ExceptionHandler(DomainException.class)
    public ResponseEntity<ErrorResponse> handleDomain(DomainException e) {
        log.warn("Domain exception: {}", e.getMessage());
        return ResponseEntity.badRequest()
            .body(new ErrorResponse(e.getMessage()));
    }

    @ExceptionHandler(TelegramApiException.class)
    public void handleTelegram(TelegramApiException e) {
        log.error("Telegram API error", e);
    }

    @ExceptionHandler(Exception.class)
    public void handleUnknown(Exception e) {
        log.error("Unexpected error", e);
    }
}
```

### 8. Sending Rich Messages

```java
private void sendTarotReading(long chatId, TarotReading reading) {
    var messageText = buildReadingText(reading);
    var message = SendMessage.builder()
        .chatId(chatId)
        .text(messageText)
        .parseMode("HTML")
        .replyMarkup(inlineKeyboardForTarot())
        .build();
    tarotistBot.execute(message);
}

private InlineKeyboardMarkup inlineKeyboardForTarot() {
    return InlineKeyboardMarkup.builder()
        .keyboard(List.of(
            List.of(
                InlineKeyboardButton.builder()
                    .text("Новый расклад 🔮")
                    .callbackData("tarot:new")
                    .build()
            )
        ))
        .build();
}
```

---

## Project Structure

```
src/main/java/ru/example/bot/
├── TarotistApplication.java
├── config/
│   └── BotProperties.java
├── controller/
│   ├── Command.java
│   ├── CommandRouter.java
│   └── handler/
│       ├── StartCommandHandler.java
│       ├── HelpCommandHandler.java
│       ├── HoroscopeCommandHandler.java
│       └── TarotCommandHandler.java
├── dto/
├── entity/
├── exception/
│   ├── DomainException.java
│   └── handler/GlobalExceptionHandler.java
├── infrastructure/
│   └── telegram/
│       ├── TarotistBot.java
│       └── config/TelegramBotConfig.java
├── repository/
└── service/
    ├── TarotReadingService.java
    └── UserService.java
```

---

## Testing

```java
@ExtendWith(MockitoExtension.class)
class StartCommandHandlerTest {
    @Mock private TarotistBot tarotistBot;
    @InjectMocks private StartCommandHandler handler;

    @Test
    void shouldHandleStartCommand() {
        var update = mock(Update.class);
        var message = mock(Message.class);
        var user = mock(User.class);

        when(update.hasMessage()).thenReturn(true);
        when(update.getMessage()).thenReturn(message);
        when(message.getChatId()).thenReturn(12345L);
        when(message.getFrom()).thenReturn(user);
        when(user.getFirstName()).thenReturn("John");

        handler.handle(update);

        verify(tarotistBot).execute(any(SendMessage.class));
    }
}
```

**Test configuration (`application-test.yaml`):**

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
  liquibase:
    enabled: false
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration

app:
  telegram:
    bot-token: test-token
    bot-username: test-bot
```

---

## Advanced Patterns

### Multi-Transport Support

```java
public enum BotTransport {
    LONG_POLLING,
    WEBHOOK,
    CUSTOM,
    NO_INGRESS
}
```

```yaml
app:
  telegram:
    mode: LONG_POLLING  # or WEBHOOK, CUSTOM, NO_INGRESS
```

### Custom Ingress

```java
@FunctionalInterface
public interface UpdateIngress {
    void start(UpdateConsumer consumer);
    void stop();
}
```

### Observability

```java
// Use Micrometer for metrics on update processing
@Configuration
public class BotMetricsConfig {
    @Bean
    public MeterRegistry meterRegistry() {
        return new SimpleMeterRegistry();
    }
}
```

---

## Decision Rules

| Decision | When | Why |
|----------|------|-----|
| **Long Polling vs Webhook** | Dev→Long Polling, Prod→Webhook | Long polling: simpler setup, no HTTPS endpoint needed. Webhook: lower latency, no polling overhead |
| **Redis vs DB for state** | Short-lived state→Redis, Persistent data→DB | Redis: fast, TTL keys. DB: ACID, complex queries, relationships |
| **Sync vs Async** | Fast ops (<100ms)→sync, Long ops→async | Never block the update consumer thread |
| **Direct reply vs Edit** | First response→reply, Update→edit | Edit prevents spam, keeps chat clean |
| **Image as binary vs URL** | Small images→URL, Large→CDN | Never store binary in DB |
| **Command enum vs Map** | Few commands→enum, Many/dynamic→Map | Enum: type-safe, easy to enumerate. Map: flexible, runtime-registerable |

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Blocking consumer thread with long I/O | Offload to `CompletableFuture` or Virtual Threads |
| In-memory state on restarts | Use Redis or database-backed state |
| Hardcoded bot token | Externalize via `@Value` + environment variables |
| No error handling in `consume()` | Wrap in try-catch with user notification |
| Exposing entities outside data layer | Use DTOs + mappers |
| No logging context | Use MDC with traceId, telegramUserId, operationName |
| Synchronous API calls in handler | Use async + immediate acknowledgment + edit on completion |

---

## Resources

**Official Documentation:**
- [TelegramBots v10 Documentation](https://rubenlagus.github.io/TelegramBotsDocumentation/)
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [TelegramBots GitHub](https://github.com/rubenlagus/TelegramBots)

**Spring Boot:**
- [Spring Boot Reference](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)
- [Spring Data Redis](https://docs.spring.io/spring-data/redis/docs/current/reference/html/)

**Best Practices:**
- [ksilisk/telegram-bot-spring](https://github.com/ksilisk/telegram-bot-spring) — modular framework example
- [Building Robust Telegram Bots](https://henrywithu.com/building-robust-telegram-bots/) — state management, async, error handling
