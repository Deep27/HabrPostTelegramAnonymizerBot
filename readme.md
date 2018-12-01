Каждый разработчик (и не только), который использует Telegram в повседневной жизни, хотя бы раз задумывался о том,
каково это - создать своего бота, на сколько это сложно и какой язык программирования лучше использовать?

## Создание нового проекта и подготовка
1. Создадим новый maven-проект и отредактируем `pom.xml`, добавив в него необходимые зависимости:
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

    * `Telegram API` - 
    * `Log4j 2` - хорошие логи лучше чем `sout`. Основные возможности `log4j 2`, которые я использую, это:
        * определение своих уровней логирования и их приоритета;
        * определение своего цвета текста для каждого уровня логирования;
        * параллельный вывод логов в консоль и файл.
    
2. Создадим новый файл с настройками `log4j2` в `src/main/java/resources` с именем `log4j2.xml` и заполним его следующим содержимым:
    ```xml
    <?xml version="1.0" encoding="UTF-8" ?>
    <Configuration status="WARN">

        <CustomLevels>
            <CustomLevel name="QUERY_STRANGE" intLevel="360"/>
            <CustomLevel name="QUERY_SUCCESS" intLevel="340"/>
        </CustomLevels>

        <Appenders>
            <Console name="Console" target="SYSTEM_OUT">
                <PatternLayout pattern="%highlight{%d{HH:mm:ss} [%t] %-5level %logger{36} - %msg%n}{STRANGE_USER=bright yellow bold, SUCCESS_USER=bright green bold}"/>
            </Console>
        </Appenders>

        <Loggers>
            <Logger name="io.example.anonymizerbot" level="info" additivity="false">
                <AppenderRef ref="Console"/>
            </Logger>
        </Loggers>

        <Root>
            <Appender ref="Console"/>
        </Root> 
     
    </Configuration> 
    ```
    Информацию о настройке `log4j2` можно найти в [официальной документации](https://logging.apache.org/log4j/2.x/).
    
3. Откроем телеграм и создадим аккаунт для нового бота.
    Для этого нам необходимо обратиться за помощью к боту BotFather:
    * найдем бота в поиске;![botfather/welcome](images/botfather/welcome.jpg)
    * выполним команду `/start`;
    * выполним команду `/newbot`;![botfather/newbot](images/botfather/newbot.jpg)
    * зададим какое-нибудь имя нашему боту (должно заканчиваться на "Bot"). Я назвал его ExampleOfAnonymizerBot.![botfather/newbot_setname](images/botfather/newbot_setname.jpg)
    
    После выполнения этих команд мы получим токен, который нам понадобится для использования Bot API.
    Его нам необходимо сохранить.
    

