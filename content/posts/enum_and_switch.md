---
title: "Enum и switch, и что с ними не так"
date: 2020-09-04T14:00:00+03:00
draft: false
toc: false
categories: ["programming", "csharp", "ru"]
images:
tags:
  - enum
  - extension
  - visitor
  - design
  - pattern
---

Часто ли у вас было такое, что вы добавляли новое значение в enum и потом тратили часы на то, чтобы найти все места его использования, а затем добавить новый case, чтобы не получить ArgumentOutOfRangeException во время исполнения?

# Идея
----

Если проблема состоит только в switch операторе и отслеживании новых типов, тогда давайте избавимся от них!

Идея состоит в том, чтобы заменить использование switch паттерном visitor.

# Пример 1
--------

Предположим у нас есть какой-то API для работы с документами, от которого мы получаем необходимые данные и определяем его тип, а далее в зависимости от этого типа, необходимо делать различные операции.

Определим файл **DocumentType.cs**:

```csharp
    public enum DocumentType
    {
        Invoice,
        PrepaymentAccount
    }
    
    public interface IDocumentVisitor<out T>
    {
        T VisitInvoice();
        T VisitPrepaymentAccount();
    }
    
    public static class DocumentTypeExt
    {
        public static T Accept<T>(this DocumentType self, IDocumentVisitor<T> visitor)
        {
            switch (self)
            {
                case DocumentType.Invoice:
                    return visitor.VisitInvoice();
                case DocumentType.PrepaymentAccount:
                    return visitor.VisitPrepaymentAccount();
                default:
                    throw new ArgumentOutOfRangeException(nameof(self), self, null);
            }
        }
    }
```

И да, я предлагаю определять все связанные типы в одном файле, что не является идиоматичным для .Net разработчика. Но иногда это очень ухудшает упрощает понимание кода.

Опишем visitor который будет искать в базе нужный документ **DatabaseSearchVisitor.cs**:

  
```csharp
    public class DatabaseSearchVisitor : IDocumentVisitor<IDocument>
    {
        private ApiId _id;
        private Database _db;
    
        public DatabaseSearchVisitor(ApiId id, Database db)
        {
            _id = id;
            _db = db;
        }
    
        public IDocument VisitInvoice() => _db.SearchInvoice(_id);
        public IDocument VisitPrepaymentAccount() => _db.SearchPrepaymentAccount(_id);
    }
```
  
И потом его использование:

  
```csharp
    public void UpdateStatus(ApiDoc doc)
    {
        var searchVisitor = new DatabaseSearchVisitor(doc.Id, _db);
    
        var databaseDocument = doc.Type.Accept(searchVisitor);
    
        databaseDocument.Status = doc.Status;
    
        _db.SaveChanges();
    }
```
  
# Пример 2
--------

У нас есть события, которые выглядят следующим образом:

  
```csharp
    public enum PurseEventType
    {
        Increase,
        Decrease,
        Block,
        Unlock
    }
    
    public sealed class PurseEvent
    {
        public PurseEventType Type { get; }
        public string Json { get; }
    
        public PurseEvent(PurseEventType type, string json)
        {
            Type = type;
            Json = json;
        }
    }
```

Мы хотим отправлять уведомления пользователю на определенный тип событий. Тогда реализуем visitor:

  
```csharp
    public interface IPurseEventTypeVisitor<out T>
    {
        T VisitIncrease();
        T VisitDecrease();
        T VisitBlock();
        T VisitUnlock();
    }
    
    public sealed class PurseEventTypeNotificationVisitor : IPurseEventTypeVisitor<Missing>
    {
        private readonly INotificationManager _notificationManager;
        private readonly PurseEventParser _eventParser;
        private readonly PurseEvent _event;
    
        public PurseEventTypeNotificationVisitor(PurseEvent @event, PurseEventParser eventParser, INotificationManager notificationManager)
        {
            _notificationManager = notificationManager;
            _event = @event;
            _eventParser = eventParser;
        }
    
        public Missing VisitIncrease() => Missing.Value;
    
        public Missing VisitDecrease() => Missing.Value;
    
        public Missing VisitBlock()
        {
            var blockEvent = _eventParser.ParseBlock(_event);
            _notificationManager.NotifyBlockPurseEvent(blockEvent);
            return Missing.Value;
        }
    
        public Missing VisitUnlock()
        {
            var blockEvent = _eventParser.ParseUnlock(_event);
            _notificationManager.NotifyUnlockPurseEvent(blockEvent);
            return Missing.Value;
        }
    }
```

Для примера не будем ничего возвращать. Для этого можно воспользоваться типом Missing из System.Reflection или же написать тип Unit. В реальном проекте возвращался бы Result, например, с информацией об ошибке, если такие имеются.

И пример использования:

```csharp
    public void SendNotification(PurseEvent @event)
    {
        var notificationVisitor = new PurseEventTypeNotificationVisitor(@event, _eventParser, _notificationManager);
        @event.Type.Accept(notificationVisitor);
    }
```

# Дополнение
----------

### Если нужно быстрее

Если нужно использовать такой подход там, где важна производительность, в качестве visitor можно использовать структуры. Тогда код изменится следующим образом.

Метод расширение:

  
```csharp
    public static T Accept<TVisitor, T>(this DocumentType self, in TVisitor visitor)
        where TVisitor : IDocumentVisitor<T>
        {
            switch (self)
            {
                case DocumentType.Invoice:
                    return visitor.VisitInvoice();
                case DocumentType.PrepaymentAccount:
                    return visitor.VisitPrepaymentAccount();
                default:
                    throw new ArgumentOutOfRangeException(nameof(self), self, null);
            }
        }
```

Сам visitor остаётся прежним, только меняем class на struct.

И сам код обновления документа выглядит не так удобно, но работает быстро:

  
```csharp
    public void UpdateStatus(ApiDoc doc)
    {
        var searchVisitor = new DatabaseSearchVisitor(doc.Id, _db);
    
        var databaseDocument = doc.Type.Accept<DatabaseSearchVisitor, IDocument>(searchVisitor);
    
        databaseDocument.Status = doc.Status;
    
        _db.SaveChanges();
    }
```
  

При таком использовании generic, необходимо уточнять типы самому, так как компилятор не хочет способен вывести их автоматически.


### Читабельность и in-place реализация


Если нужно реализовать логику только в одном месте, то часто visitor — громоздко и не удобно. Поэтому есть альтернативное решение **match**.

Сразу пример со структурой:

  
```csharp
    public static T Match<T>(this DocumentType self, Func<T> invoiceCase, Func<T> prepaymentAccountCase)
    {
        var visitor = new FuncVisitor<T>(invoiceCase, prepaymentCase);
        return self.Accept<FuncVisitor<T>, T>(visitor);
    }
```
  
Сам **FuncVisitor**:

  
```csharp
    public readonly struct FuncVisitor<T> : IDocumentVisitor<T>
    {
        private readonly Func<T> _invoiceCase;
        private readonly Func<T> _prepaymentAccountCase;
    
        public FuncVisitor(Func<T> invoiceCase, Func<T> prepaymentAccountCase)
        {
            _invoiceCase = invoiceCase;
            _prepaymentAccountCase = prepaymentAccountCase;
        }
    
        public T VisitInvoice() => _invoiceCase();
        public T VisitPrepaymentAccount() => _prepaymentAccountCase();
    }
```

Использование **match**:

  
```csharp
public void UpdateStatus(ApiDoc doc)
{
    var databaseDocument = doc.Type.Match(
        () => _db.SearchInvoice(doc.Id),
        () => _db.SearchPrepaymentAccount(doc.Id)
    );
    
    databaseDocument.Status = doc.Status;
    
    _db.SaveChanges();
}
```


# Итог
----

При добавлении нового значения в enum необходимо:

&nbsp;&nbsp; 1.  Добавить метод в интерфейс.


&nbsp;&nbsp; 2.  Добавить его использование в метод расширение.

Для остальных мест компилятор подскажет нам, где необходимо реализовать новый метод.  
Таким образом мы избавляемся от проблемы забытого case в switch.

Это все еще не серебряная пуля, но может здорово помочь в работе с enum.

  
<br>

# Ссылки
------
  

*   [https://blog.ploeh.dk/2018/06/25/visitor-as-a-sum-type/](https://blog.ploeh.dk/2018/06/25/visitor-as-a-sum-type/)
*   [https://en.wikipedia.org/wiki/Unit\_type](https://en.wikipedia.org/wiki/Unit_type)
*   [https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/results](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/results)