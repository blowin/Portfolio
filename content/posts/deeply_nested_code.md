---
title: "Слишком глубокий смысл"
date: 2022-06-08T17:52:47+04:00
draft: false
toc: false
categories: ["programming", "csharp", "ru", "refactoring"]
images:
tags:
  - design
  - clean_code
  - code_smell
  - deep_nested
---

> Мужество — лучшее смертоносное оружие: мужество убивает даже сострадание. Сострадание же есть наиболее глубокая пропасть: ибо, насколько глубоко человек заглядывает в жизнь, настолько глубоко заглядывает он и в страдание.
> 
> *Фридрих Вильгельм Ницше*

# Проблема

Обращали ли вы внимание, как тяжело читать код с большой вложенностью? Думаю, что да. Такой код очень тяжел в понимании и поддержании из-за чего подвержен большему количеству ошибок. В этой статье рассмотрим способы решения данной проблемы и посмотрим как меняется код в лучшую сторону, при соблюдении простых правил. 

Сегодня будем говорить о таком запахе кода, как 'Deeply Nested Code'.

> Также хотелось бы напомнить, что у меня [есть анализатор чистоты кода на базе Roslyn](https://github.com/blowin/BlowinCleanCode), где есть анализ данной проблемы.

# Решения

## Инвертирование условий

Давайте рассмотрим 2 куска кода и решим, какой из них более понятен.

### Первоначальная версия

```csharp
public void SendAsPdf(Guid reportId, Guid userId, DateTime date)
{
    var dataSource = LoadDataSource(reportId, userId, date);
    if (dataSource != null)
    {
        var report = CreateReport(dataSource);
        if (report != null)
        {
            report.PrintDate = DateTime.Now;
            report.AdditionalHeader = true;

            byte[] pdf = _reportGenerator.GeneratePdf(report);
            SendPdf(pdf);
        }
    }
}
```

### Новая версия

```csharp
public void SendAsPdf(Guid reportId, Guid userId, DateTime date)
{
    var dataSource = LoadDataSource(reportId, userId, date);
    if (dataSource == null) 
        return;
    
    var report = CreateReport(dataSource);
    if (report == null) 
        return;
    
    report.PrintDate = DateTime.Now;
    report.AdditionalHeader = true;

    byte[] pdf = _reportGenerator.GeneratePdf(report);
    SendPdf(pdf);
}
```

## Инкапсуляция

### Первоначальная версия

```csharp
public void RunProcess(string[] args)
{
    if (!_service.HasUnhandledItem)
    {
        if (_service.TimeToRun)
        {
            _service.Run(args);
            _logger.LogInfo("Run process {Args} {Time}", args, DateTime.UtcNow);
        }
    }
}
```

### Новая версия

Правильнее было бы объединить два свойства *HasUnhandledItem* и *TimeToRun* в одно правильно названное и использовать его.

```csharp
public void RunProcess(string[] args)
{
    if (_service.Ready)
    {
        _service.Run(args);
        _logger.LogInfo("Run process {Args} {Time}", args, DateTime.UtcNow);    
    }
}
```

### Применяем инверсию условий

```csharp
public void RunProcess(string[] args)
{
    if (!_service.Ready) 
        return;
    
    _service.Run(args);
    _logger.LogInfo("Run process {Args} {Time}", args, DateTime.UtcNow);
}
```

### Нет доступа к сорцам

Мы часто используем библиотечный код, который не можем изменить. Что делать в таком случае? Используем extension.

```csharp
// Класс с extension методом
public static bool Ready(this ProcessService self) => !_service.HasUnhandledItem && _service.TimeToRun;

// Использование
public void RunProcess(string[] args)
{
    if (!_service.Ready()) 
        return;
    
    _service.Run(args);
    _logger.LogInfo("Run process {Args} {Time}", args, DateTime.UtcNow);
}
```

## Выделение в метод

Часто приходится видеть огромный try catch, в котором написано много кода. Что не так с этим подходом, кроме того, что с каждым вложенным элементов усложняется когнитивная сложность? Например, такой код тяжелее понимать, потому что разработчик начинает исполнять его глазами и в каждом месте, где потенциально может произойти ошибка, прыгать в catch блоки и искать обработчик.

### Первоначальная версия

```csharp
public void UpdateStatus(IEnumerable<Document> documents, DocumentStatus newStatus)
{
    try
    {
        foreach (var document in documents)
        {
            var detail = LoadDetail(document);
            UpdateStatus(document, detail, newStatus);
        }
    }
    catch (ArgumentException e)
    {
        // Сложная обработка
    }
    catch (Exception e)
    {
        // Сложная обработка
    }
}
```

### Новая версия

Выделим логику из try в отдельный метод.

```csharp
public void UpdateStatus(IEnumerable<Document> documents, DocumentStatus newStatus)
{
    try
    {
        UpdateStatusCore(documents, newStatus);
    }
    catch (ArgumentException e)
    {
        // Сложная обработка
    }
    catch (Exception e)
    {
        // Сложная обработка
    }
}

private void UpdateStatusCore(IEnumerable<Document> documents, DocumentStatus newStatus)
{
    foreach (var document in documents)
    {
        var detail = LoadDetail(document);
        UpdateStatus(document, detail, newStatus);
    }
}
```

Кода стало больше, но этот код кудо удобнее поддерживать и он более интуитивен, чем первый вариант. В таком случае мы пишем код, будто тут нет никаких исключений, а реальную логику обработки выполняют на уровень выше. Обратите внимание, что новый метод private и его можно вызвать только внутри данного класса. Можно воспользовать [локальными функциями](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/local-functions), для того, чтобы только один метод мог использовать данную логику.

### Local function

```csharp
public void UpdateStatus(IEnumerable<Document> documents, DocumentStatus newStatus)
{
    try
    {
        UpdateStatusCore(documents, newStatus);
    }
    catch (ArgumentException e)
    {
        // Сложная обработка
    }
    catch (Exception e)
    {
        // Сложная обработка
    }

    void UpdateStatusCore(IEnumerable<Document> allDocuments, DocumentStatus statusForUpdate)
    {
        foreach (var document in allDocuments)
        {
            var detail = LoadDetail(document);
            UpdateStatus(document, detail, statusForUpdate);
        }
    }
}
```

Можно сделать UpdateStatusCore без параметров, но по-моему мнению это будет только больше путать разработчика, поэтому мы передаём каждый параметр явно, но всё зависит от конкретной ситуации.

## Императивность

Ежедневно мы используем циклы и пишем императивный код, но лучшей заменой такого подходя является декларативность. С её помощью мы избегаем большую вложенность и просто описываем цепочку действий. Хорошим примером является [linq](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/linq/).

### Первоначальная версия

```csharp
public IEnumerable<Document> FindDocumentsWithStatuses(IEnumerable<Document> documents, DocumentStatus searchStatus)
{
    foreach (var document in documents)
    {
        if (document.Status == searchStatus)
            yield return document;
    }
}
```

### Новая версия

```csharp
public IEnumerable<Document> FindDocumentsWithStatuses(IEnumerable<Document> documents, DocumentStatus searchStatus)
{
    return documents.Where(document => document.Status == searchStatus);
}
```

### Обновляем UpdateStatusCore

Доработаем UpdateStatusCore из прошлого примера.

```csharp
void UpdateStatusCore(IEnumerable<Document> allDocuments, DocumentStatus statusForUpdate)
{
    allDocuments.ForEach(doc => UpdateStatus(doc, LoadDetail(doc), statusForUpdate));
}
```

Или используем Select

```csharp
void UpdateStatusCore(IEnumerable<Document> allDocuments, DocumentStatus statusForUpdate)
{
    allDocuments
        .Select(doc => (Document: doc, Detail: LoadDetail(doc)))
        .ForEach(x => UpdateStatus(x.Document, x.Detail, statusForUpdate));
}
```

# Итог

В данной статье рассмотрели несколько простых способов, как можно улучшить понимаемость и поддреживаемость кода.

# Ссылки

* [BlowinCleanCode](https://github.com/blowin/BlowinCleanCode)
* [Code Smell](https://en.wikipedia.org/wiki/Code_smell)
* [Local functions](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/local-functions)
* [Linq](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/linq/)
* [Декларативное программирование](https://ru.wikipedia.org/wiki/%D0%94%D0%B5%D0%BA%D0%BB%D0%B0%D1%80%D0%B0%D1%82%D0%B8%D0%B2%D0%BD%D0%BE%D0%B5_%D0%BF%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5)
* [Императивное программирование](https://ru.wikipedia.org/wiki/%D0%98%D0%BC%D0%BF%D0%B5%D1%80%D0%B0%D1%82%D0%B8%D0%B2%D0%BD%D0%BE%D0%B5_%D0%BF%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5)