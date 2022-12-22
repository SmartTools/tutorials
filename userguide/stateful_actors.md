---
layout: page
title: Stateful actors, state
parent: User Guide
nav_order: 5
---

# Акторы с состоянием
## Введение
Данный раздел документации предназначен для описания принципов работы с акторами, у которых есть состояние (stateful actors).

## Описание
Основная причина для наличия состояния в акторе - что-то надо постоянно хранить в памяти во время работы сервера. При разработке актора, у которого есть состояние, необходимо учитывать некоторые особенности, которые необязательны для акторов без состояния. Тем не менее, на них накладываются [те же рекомендации](actors_wrappers), что и на обычные акторы.

## Принципы и рекомендации
### Обертка для конструктора
Наличие конструктора всегда чревато тем, что у актора есть какое-то внутреннее состояние. В некоторых случаях в конструктор необходимо прокидывать некоторые параметры (например, реализацию некоторого интерфейса). В таких случаях разработчику необходимо написать специальную обертку, которая будет передавать параметры в конструктор. Принципы наименования обертки конструктора и её методов аналогичные обертке хэндлеров.

В отличии от обертки хэндлера, для обертки конструктора нет автоматической генерации кода, поэтому чтобы передать в конструктор актора параметры, в плагине, в котором происходит регистрация актора, необходимо сделать реализацию интерфейса, который соберет в себя данные, а затем передаст в актор.

#### Пример
На практике описанное выше выглядит следующим образом. Конструктор имеет следующий вид:
```java
public GetDocumentsExecutor(final GetDocumentsConfig wrapper) throws InitializationException {
    String connectionOptionsName = wrapper.getConnectionOptionsName();
    String connectionPoolName = wrapper.getConnectionPoolName();
    String collectionName = wrapper.getCollectionName();
    String searchTaskKey = wrapper.getSearchTaskKey();
    // и так далее
}
```

Обертка, соответственно, имеет следующий вид:
```java
public interface GetDocumentsConfig {
    String getConnectionOptionsName() throws ReadValueException;
    String getConnectionPoolName() throws ReadValueException;
    String getCollectionName() throws ReadValueException;
    String getSearchTaskKey() throws ReadValueException;
}
```

В плагине, где происходит регистрация актора, информацию о полях можно получить из вариативных аргументов `args` в лямбде стратегии, конфигурация актора для конкретной цепочки будет лежать под нулевым индексом. Процесс передачи данных в конструктор актора будет иметь следующий вид:

```java
IObject configurationObject = (IObject) args[0];

GetDocumentsConfig config = new GetDocumentsConfig() {

    @Override
    String getConnectionOptionsName() throws ReadValueException {
        return config.getValue(connectionOptionsFN);
    }

    @Override
    String getConnectionPoolName() throws ReadValueException {
        return config.getValue(connectionPoolFN);
    }

    @Override
    String getCollectionName() throws ReadValueException {
        return config.getValue(collectionNameFN);
    }

    @Override
    String getSearchTaskKey() throws ReadValueException {
        return config.getValue(searchTaskKeyFN);
    }
};

return new GetDocumentsExecutor(config);
```

**Замечание:** для упрощения примера тут не продемонстрировано формирование IFieldName, регистрация в IOC и обработка исключений.

### Передача аргументов
Может показаться, что хорошим местом для передачи аргументов будет секция `objects`, где и происходит объявление актора. Но по факту это приводит к тому, что параметры актора при переиспользовании модуля в других проектах могут оказаться неизменяемыми. Для этих целей рекомендуется выносить все параметры актора в properties, а в секции `objects` оставить только путь до properties.

#### Пример
Рассмотрим актор для хранения информации о текущем подключении к БД. Первоначально его объявление в секции `objects` имело следующий вид:
```json
{
    "name": "connection-options-storage",
    "kind": "actor",
    "dependency": "options storage",
    "url": "jdbc://localhost:5432/project_db",
    "username": "user",
    "password": "strong-password",
    "maxConnections": 20
}
```

Если по каким-то причинам нам пришлось перезагрузить сервер, или же модуль с этим актором переиспользуется на другом проекте, то единственная возможность поменять параметры - залезть в исходный код модуля и поправить их. Поэтому здесь мы заменяем параметры на путь до properties-файла.

```json
{
    "name": "connection-options-storage",
    "kind": "actor",
    "dependency": "options storage",
    "properties": "properties/connectionOptionsStorage.properties"
}
```

И в файле `connectionOptionsStorage.properties` параметры лежат в следующем виде:

```properties
url = jdbc://localhost:5432/project_db
username = user
password = strong-password
maxConnections = 20
```

### Управляющие хэндлеры
Когда сервер SmartActors запущен, единственная причина, по которой он должен перезагружаться - пропало электричество и сервер был перезагружен. Все остальное время он должен работать непрерывно. Это накладывает свои ограничения на акторы с состоянием, т.к. у администратора всегда должна быть возможность повлиять на состояние актора. Для этих целей разработчик актора с состоянием обязательно должен добавить управляющий (контрольный) хэндлер, который может повлиять на состояние.

#### Пример
Рассмотрим актор для отправки почты. У него в памяти должна лежать следующая информация - адрес почтового сервера, его порт и по какому протоколу идет отправка. Эта информация на старте сервера считывается из properties-файла. У актора есть один хэдлер - `send()`, который, собственно, и занимается отправкой почты. Выглядит это все следующим образом:

```java
private String host;
private String port;
private String protocol;

public EmailSender(final EmailSenderConfig config) {
    // заполнение внутреннего состояние
}

public void send(final SendEmailMessage message) {
    // отправка почты через почтовый сервер, информация о котором лежит в памяти
}
```

Предположим, что почтовый сервер переехал на другой адрес. Мы заменили информацию о нем в properties-файле, но сервер сейчас занимается обработкой данных, которую прерывать никак нельзя. В таком случае единственный способ заменить информацию о почтовом сервере - вызвать управляющий хэндлер `updateSettings()`.

```java
private String host;
private String port;
private String protocol;

public EmailSender(final EmailSenderConfig config) {
    // заполнение внутреннего состояние
}

public void send(final SendEmailMessage message) {
    // отправка почты через почтовый сервер, информация о котором лежит в памяти
}

public void updateSettings(final UpdateSettingsMessage message) {
    this.host = wrapper.getHost();
    this.port = wrapper.getPort();
    this.protocol = wrapper.getProtocol();
}
```

При вызове этого хэндлера информация о почтовом сервере в памяти актора поменяется, и все письма, которые будут отправляться с помощью этого актора, пойдут уже на новый почтовый сервер.

**Замечание:** для упрощения примера тут не продемонстрирована работа с исключениями и проверка на наличие значения у поля в обертке. При реализации управляющего хэндлера рекомендуется сделать проверку на наличие значение у полей в обертке, потому что может возникнуть ситуация, когда надо обновить только одно поле. 