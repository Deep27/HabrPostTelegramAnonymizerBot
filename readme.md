Каждый разработчик (и не только), который использует Telegram в повседневной жизни, хотя бы раз задумывался о том,
каково это - создать своего бота, на сколько это сложно и какой язык программирования лучше использовать?

## Создание нового проекта и подготовка
### 1. Добавление зависимостей в проект
Создадим новый maven-проект и отредактируем `pom.xml`, добавив в него необходимые зависимости: 
<details>
    <summary>pom.xml</summary>        
    
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>io.example</groupId>
    <artifactId>anonymizerbot</artifactId>
    <version>1.0-SNAPSHOT</version>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>8</source>
                    <target>8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>

    <dependencies>

        <!-- Telegram API -->
        <dependency>
            <groupId>org.telegram</groupId>
            <artifactId>telegrambots</artifactId>
            <version>LATEST</version>
        </dependency>
        <dependency>
            <groupId>org.telegram</groupId>
            <artifactId>telegrambotsextensions</artifactId>
            <version>LATEST</version>
        </dependency>

        <!-- Log4j 2 -->
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>2.11.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>2.11.1</version>
        </dependency>

    </dependencies>

</project>
``` 
* `Telegram API` - [библиотека для работы с Telegram API](https://github.com/rubenlagus/TelegramBots),
    содержит в себе классы и методы для взаимодействия с сервисами Telegram и некоторые расширения
    этих классов.
* `Log4j 2` - логгер. Основные возможности `log4j2`, которые я использую, это:
    * определение своих уровней логирования и их приоритетов;
    * определение своего цвета текста для каждого уровня логирования;
    * параллельный вывод логов в консоль и файл.
</details>
    
### 2. Настройка log4j2
Хорошим тоном считается использование в проектах логгеров, а не `System.out.println()`.
Создадим новый файл с настройками `log4j2` в `src/main/java/resources` с именем `log4j2.xml`:
<details>
    <summary>log4j2.xml</summary>

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<Configuration status="WARN">

    <CustomLevels>
        <CustomLevel name="STRANGE" intLevel="360"/>
        <CustomLevel name="SUCCESS" intLevel="340"/>
    </CustomLevels>

    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%highlight{%d{HH:mm:ss} [%t] %-5level %logger{36} - %msg%n}{STRANGE=bright yellow bold, SUCCESS=bright green bold}"/>
        </Console>
    </Appenders>

    <Loggers>
        <Logger name="io.deep27soft.deepanonymizerbot" level="info" additivity="false">
            <AppenderRef ref="Console"/>
        </Logger>
    </Loggers>

    <Root>
        <Appender ref="Console"/>
    </Root>

</Configuration>
```
Информацию о настройке и использовании `log4j2` можно найти в [официальной документации](https://logging.apache.org/log4j/2.x/).
</details>
    
### 3. Создание аккаунта для бота
Для этого нам необходимо обратиться за помощью к боту BotFather:
* найдем бота в поиске;

![botfather/welcome](images/botfather/welcome.jpg)     </details>
* выполним команду `/start`;
* выполним команду `/newbot`;

![botfather/newbot](images/botfather/newbot.jpg)
* зададим какое-нибудь имя нашему боту (должно заканчиваться на "Bot"). Я назвал его ExampleOfAnonymizerBot.

![botfather/newbot_setname](images/botfather/newbot_setname.jpg)

После выполнения этих команд мы получим токен, который нам понадобится для использования Bot API.
(`749430772:AAF54VXPZeGRgFWmjCto-c8EIm7Ydk_VCW0`)
    
## Реализация
### 1. Модель анонимного отправителя сообщений

Данные, необходимые нам от кажного пользователя:
- `User mUser` - информация о пользователе Telegram;
- `Chat mChat` - информация о чате пользователя и бота;
- `String mDisplayedName` - имя, от которого пользователь будет посылать сообщения другим пользователям бота.

<details>
    <summary>Anonymous.java</summary> 
    
```java
package io.example.anonymizerbot.model;

import org.telegram.telegrambots.meta.api.objects.Chat;
import org.telegram.telegrambots.meta.api.objects.User;

public final class Anonymous {

    private final User mUser;
    private final Chat mChat;
    private String mDisplayedName;

    public Anonymous(User user, Chat chat) {
        mUser = user;
        mChat = chat;
    }

    @Override
    public int hashCode() {
        return mUser.hashCode();
    }

    @Override
    public boolean equals(Object obj) {
        return obj instanceof Anonymous && ((Anonymous) obj).getUser().equals(mUser);
    }

    public User getUser() {
        return mUser;
    }

    public Chat getChat() {
        return mChat;
    }

    public String getDisplayedName() {
        return mDisplayedName;
    }

    public void setDisplayedName(String displayedName) {
        mDisplayedName = displayedName;
    }
}

```
</details> 

</br>

Обернем модель в `Anonymouses`, добавив несколько нужных методов для работы с анонимами:
<details>
    <summary>Anonymouses.java</summary> 
    
```java
package io.example.anonymizerbot.model;

import org.telegram.telegrambots.meta.api.objects.User;

import java.util.HashSet;
import java.util.Objects;
import java.util.Set;
import java.util.stream.Stream;

public final class Anonymouses {

    private final Set<Anonymous> mAnonymouses;

    public Anonymouses() {
        mAnonymouses = new HashSet<>();
    }

    public boolean setUserDisplayedName(User user, String name) {

        if (isDisplayedNameTaken(name)) {
            return false;
        } else {
            mAnonymouses.stream().filter(a -> a.getUser().equals(user)).forEach(a -> a.setDisplayedName(name));
            return true;
        }
    }

    public boolean removeAnonymous(User user) {
        return mAnonymouses.removeIf(a -> a.getUser().equals(user));
    }

    public boolean addAnonymous(Anonymous anonymous) {
        return mAnonymouses.add(anonymous);
    }

    public boolean hasAnonymous(User u) {
        return mAnonymouses.stream().anyMatch(a -> a.getUser().equals(u));
    }

    public String getDisplayedName(User u) {

        Anonymous anonymous = mAnonymouses.stream().filter(a -> a.getUser().equals(u)).findFirst().orElse(null);

        if (anonymous == null) {
            return null;
        }
        return anonymous.getDisplayedName();
    }

    public Stream<Anonymous> anonymouses() {
        return mAnonymouses.stream();
    }


    private boolean isDisplayedNameTaken(String name) {
        return mAnonymouses.stream().anyMatch(a -> Objects.equals(a.getDisplayedName(), name));
    }
} 
``` 
</details>

### 2. Интерфейс бота

Любая кастомная команда должна наследоваться от `BotCommand` и реализовывать метод
`execute(AbsSender sender, User user, Chat chat, String[] strings)`, который используется для обработки команд пользователей.

После того как мы обработаем команду пользователя, мы можем послать ему ответ, используя метод `execute` класса`AbsSender`,
который принимает на вход вышеупомянутый `execute(AbsSender sender, User user, Chat chat, String[] strings)`.

Здесь и далее чтобы не оборачивать каждый раз метод `AbsSender.execute`, который может выбросить исключение `TelegramApiException`,
в `try-catch`, и для того чтобы не прописывать в каждой команде вывод однообразных логов,
создадим класс `AnonymizerCommand`, а наши кастомные комады будем уже наследовать от него (обработку исключений в этом примере оставим):

<details>
    <summary>AnonymizerCommand.java</summary>
    
```java
package io.example.anonymizerbot.command;

import io.example.anonymizerbot.logger.LogLevel;
import io.example.anonymizerbot.logger.LogTemplate;
import org.apache.logging.log4j.Level;
import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.LogManager;
import org.telegram.telegrambots.extensions.bots.commandbot.commands.BotCommand;
import org.telegram.telegrambots.meta.api.methods.send.SendMessage;
import org.telegram.telegrambots.meta.api.objects.User;
import org.telegram.telegrambots.meta.bots.AbsSender;
import org.telegram.telegrambots.meta.exceptions.TelegramApiException;

abstract class AnonymizerCommand extends BotCommand {

    final Logger log = LogManager.getLogger(getClass());

    AnonymizerCommand(String commandIdentifier, String description) {
        super(commandIdentifier, description);
    }

    void execute(AbsSender sender, SendMessage message, User user) {
        try {
            sender.execute(message);
            log.log(Level.getLevel(LogLevel.SUCCESS), LogTemplate.COMMAND_SUCCESS, user.getId(), getCommandIdentifier());
        } catch (TelegramApiException e) {
            log.error(LogTemplate.COMMAND_EXCEPTION, user.getId(), getCommandIdentifier(), e);
        }
    }
} 
``` 
</details>

</br>

В методах логгера используются уровни (`Level`) и шаблоны сообщений (`LogTemplate`):
<details>
    <summary>LogLevel.java</summary>
    
```java
package io.example.anonymizerbot.logger;

public final class LogLevel {

    public static final String STRANGE = "STRANGE";
    public static final String SUCCESS = "SUCCESS";

    private LogLevel() {}
} 
```
</details>

<details>
    <summary>LogTemplate.java</summary>
    
```java
package io.example.anonymizerbot.logger;

public final class LogTemplate {

    public static final String MESSAGE_EXCEPTION = "User {} has caused an exception while sending message!";
    public static final String MESSAGE_PROCESSING = "Precessing user {}'s message.";
    public static final String MESSAGE_RECEIVED = "User {} has received message from another user {}.";
    public static final String MESSAGE_LOST = "User {} did not get message from another user {}.";

    public static final String COMMAND_PROCESSING = "User {} is executing '{}' command...";
    public static final String COMMAND_SUCCESS = "User {} has successfully executed '{}' command.";
    public static final String COMMAND_EXCEPTION = "User {} command '{}' has caused an exception!";

    private LogTemplate() {}
} 
```
</details>

</br>

Определим команды, на которые наш бот будет реагировать:
- `/start` - создаст нового `Anonymous` без имени и добавит его в коллекцию `Anonymouses`;
<details>
    <summary>StartCommand.java</summary>
    
```java
package io.example.anonymizerbot.command;

import io.example.anonymizerbot.logger.LogLevel;
import io.example.anonymizerbot.logger.LogTemplate;
import io.example.anonymizerbot.model.Anonymous;
import io.example.anonymizerbot.model.Anonymouses;
import org.apache.logging.log4j.Level;
import org.telegram.telegrambots.meta.api.methods.send.SendMessage;
import org.telegram.telegrambots.meta.api.objects.Chat;
import org.telegram.telegrambots.meta.api.objects.User;
import org.telegram.telegrambots.meta.bots.AbsSender;

public final class StartCommand extends AnonymizerCommand {

    private final Anonymouses mAnonymouses;

    public StartCommand(Anonymouses anonymouses) {
        super("start", "start using bot\n");
        mAnonymouses = anonymouses;
    }

    @Override
    public void execute(AbsSender absSender, User user, Chat chat, String[] strings) {

        log.info(LogTemplate.COMMAND_PROCESSING, user.getId(), getCommandIdentifier());

        StringBuilder sb = new StringBuilder();

        SendMessage message = new SendMessage();
        message.setChatId(chat.getId().toString());

        if (mAnonymouses.addAnonymous(new Anonymous(user, chat))) {
            log.info("User {} is trying to execute '{}' the first time. Added to users' list.", user.getId(), getCommandIdentifier());
            sb.append("Hi, ").append(user.getUserName()).append("! You've been added to bot users' list!\n")
                    .append("Please execute command:\n'/set_name <displayed_name>'\nwhere <displayed_name> is the name you want to use to hide your real name.");
        } else {
            log.log(Level.getLevel(LogLevel.STRANGE), "User {} has already executed '{}'. Is he trying to do it one more time?", user.getId(), getCommandIdentifier());
            sb.append("You've already started bot! You can send messages if you set your name (/set_name).");
        }

        message.setText(sb.toString());
        execute(absSender, message, user);
    }
} 
``` 
</details> 

</br>

- `/help` - выведет пользователю информацию обо всех доступных командах;

<details>
    <summary>HelpCommand.java</summary>
    
```java
package io.example.anonymizerbot.command;

import io.example.anonymizerbot.logger.LogTemplate;
import org.telegram.telegrambots.extensions.bots.commandbot.commands.ICommandRegistry;
import org.telegram.telegrambots.meta.api.methods.send.SendMessage;
import org.telegram.telegrambots.meta.api.objects.Chat;
import org.telegram.telegrambots.meta.api.objects.User;
import org.telegram.telegrambots.meta.bots.AbsSender;

public final class HelpCommand extends AnonymizerCommand {

    private final ICommandRegistry mCommandRegistry;

    public HelpCommand(ICommandRegistry commandRegistry) {
        super("help", "list all known commands\n");
        mCommandRegistry = commandRegistry;
    }

    @Override
    public void execute(AbsSender absSender, User user, Chat chat, String[] strings) {

        log.info(LogTemplate.COMMAND_PROCESSING, user.getId(), getCommandIdentifier());

        StringBuilder helpMessageBuilder = new StringBuilder("<b>Available commands:</b>\n\n");

        mCommandRegistry.getRegisteredCommands().forEach(cmd -> helpMessageBuilder.append(cmd.toString()).append("\n"));

        SendMessage helpMessage = new SendMessage();
        helpMessage.setChatId(chat.getId().toString());
        helpMessage.enableHtml(true);
        helpMessage.setText(helpMessageBuilder.toString());

        execute(absSender, helpMessage, user);
    }
} 
```
</details>

</br>

- `/set_name` - задаст пользователю имя, от которого будут отправляться анонимные сообщения;

<details>
    <summary>SetNameCommand.java</summary>
    
```java
package io.example.anonymizerbot.command;

import io.example.anonymizerbot.logger.LogLevel;
import io.example.anonymizerbot.logger.LogTemplate;
import io.example.anonymizerbot.model.Anonymouses;
import org.apache.logging.log4j.Level;
import org.telegram.telegrambots.meta.api.methods.send.SendMessage;
import org.telegram.telegrambots.meta.api.objects.Chat;
import org.telegram.telegrambots.meta.api.objects.User;
import org.telegram.telegrambots.meta.bots.AbsSender;

public final class SetNameCommand extends AnonymizerCommand {

    private final Anonymouses mAnonymouses;

    public SetNameCommand(Anonymouses anonymouses) {
        super("set_name", "set or change name that will be displayed with your messages\n");
        mAnonymouses = anonymouses;
    }

    @Override
    public void execute(AbsSender absSender, User user, Chat chat, String[] strings) {

        log.info(LogTemplate.COMMAND_PROCESSING, user.getId(), getCommandIdentifier());

        SendMessage message = new SendMessage();
        message.setChatId(chat.getId().toString());

        if (!mAnonymouses.hasAnonymous(user)) {
            log.log(Level.getLevel(LogLevel.STRANGE), "User {} is trying to execute '{}' without starting the bot!", user.getId(), getCommandIdentifier());
            message.setText("Firstly you should start the bot! Execute '/start' command!");
            execute(absSender, message, user);
            return;
        }

        String displayedName = getName(strings);

        if (displayedName == null) {
            log.log(Level.getLevel(LogLevel.STRANGE), "User {} is trying to set empty name.", user.getId());
            message.setText("You should use non-empty name!");
            execute(absSender, message, user);
            return;
        }

        StringBuilder sb = new StringBuilder();

        if (mAnonymouses.setUserDisplayedName(user, displayedName)) {

            if (mAnonymouses.getDisplayedName(user) == null) {
                log.info("User {} set a name '{}'", user.getId(), displayedName);
                sb.append("Your displayed name: '").append(displayedName)
                        .append("'. Now you can send messages to bot!");
            } else {
                log.info("User {} has changed name to '{}'", user.getId(), displayedName);
                sb.append("Your new displayed name: '").append(displayedName).append("'.");
            }
        } else {
            log.log(Level.getLevel(LogLevel.STRANGE), "User {} is trying to set taken name '{}'", user.getId(), displayedName);
            sb.append("Name ").append(displayedName).append(" is already in use! Choose another name!");
        }

        message.setText(sb.toString());
        execute(absSender, message, user);
    }

    private String getName(String[] strings) {

        if (strings == null || strings.length == 0) {
            return null;
        }

        String name = String.join(" ", strings);
        return name.replaceAll(" ", "").length() == 0 ? null : name;
    }
} 
```
</details>

</br>

- `/my_name` - отобразит текущее имя пользователя;

<details>
    <summary>MyNameCommand.java</summary>
    
```java
package io.example.anonymizerbot.command;

import io.example.anonymizerbot.logger.LogLevel;
import io.example.anonymizerbot.logger.LogTemplate;
import io.example.anonymizerbot.model.Anonymouses;
import org.apache.logging.log4j.Level;
import org.telegram.telegrambots.meta.api.methods.send.SendMessage;
import org.telegram.telegrambots.meta.api.objects.Chat;
import org.telegram.telegrambots.meta.api.objects.User;
import org.telegram.telegrambots.meta.bots.AbsSender;

public final class MyNameCommand extends AnonymizerCommand {

    private final Anonymouses mAnonymouses;

    public MyNameCommand(Anonymouses anonymouses) {
        super("my_name", "show your current name that will be displayed with your messages\n");
        mAnonymouses = anonymouses;
    }

    @Override
    public void execute(AbsSender absSender, User user, Chat chat, String[] strings) {

        log.info(LogTemplate.COMMAND_PROCESSING, user.getId(), getCommandIdentifier());

        StringBuilder sb = new StringBuilder();

        SendMessage message = new SendMessage();
        message.setChatId(chat.getId().toString());

        if (!mAnonymouses.hasAnonymous(user)) {

            sb.append("You are not in bot users' list! Send /start command!");
            log.log(Level.getLevel(LogLevel.STRANGE), "User {} is trying to execute '{}' without starting the bot.", user.getId(), getCommandIdentifier());

        } else if(mAnonymouses.getDisplayedName(user) == null) {

            sb.append("Currently you don't have a name.\nSet it using command:\n'/set_name <displayed_name>'");
            log.log(Level.getLevel(LogLevel.STRANGE), "User {} is trying to execute '{}' without having a name.", user.getId(), getCommandIdentifier());

        } else {

            log.info("User {} is executing '{}'. Name is '{}'.", user.getId(), getCommandIdentifier(), mAnonymouses.getDisplayedName(user));
            sb.append("Your current name: ").append(mAnonymouses.getDisplayedName(user));
        }

        message.setText(sb.toString());
        execute(absSender, message, user);
    }
} 
```
</details>

</br>

- `/stop` - удалит пользователя из коллекции анонимусов.

<details>
    <summary>StopCommand.java</summary>
    
```java
package io.example.anonymizerbot.command;

import io.example.anonymizerbot.logger.LogLevel;
import io.example.anonymizerbot.logger.LogTemplate;
import io.example.anonymizerbot.model.Anonymouses;
import org.apache.logging.log4j.Level;
import org.telegram.telegrambots.meta.api.methods.send.SendMessage;
import org.telegram.telegrambots.meta.api.objects.Chat;
import org.telegram.telegrambots.meta.api.objects.User;
import org.telegram.telegrambots.meta.bots.AbsSender;

public final class StopCommand extends AnonymizerCommand {

    private final Anonymouses mAnonymouses;

    public StopCommand(Anonymouses anonymouses) {
        super("stop", "remove yourself from bot users' list\n");
        mAnonymouses = anonymouses;
    }

    @Override
    public void execute(AbsSender absSender, User user, Chat chat, String[] strings) {

        log.info(LogTemplate.COMMAND_PROCESSING, user.getId(), getCommandIdentifier());

        StringBuilder sb = new StringBuilder();

        SendMessage message = new SendMessage();
        message.setChatId(chat.getId().toString());

        if (mAnonymouses.removeAnonymous(user)) {
            log.info("User {} has been removed from users list!", user.getId());
            sb.append("You've been removed from bot's users list! Bye!");
        } else {
            log.log(Level.getLevel(LogLevel.STRANGE), "User {} is trying to execute '{}' without having executed 'start' before!", user.getId(), getCommandIdentifier());
            sb.append("You were not in bot users' list. Bye!");
        }

        message.setText(sb.toString());
        execute(absSender, message, user);
    }
} 
```
</details>

</br>


