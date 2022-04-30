---
title: "Enum и switch, сквозь года"
date: 2022-04-29T19:16:46+03:00
draft: false
toc: false
categories: ["programming"]
images:
tags:
  - enum
  - csharp
  - extension
  - visitor
  - design
  - pattern
  - SRP
---

Пару лет назад, [я писал статью](enum_and_switch.md). По прошествии нескольких лет, произошли определенные переосмысления, которые выросли в данную статью.

# Проблема
----

Проблема остаётся той же:
* огромные switch по всему коду
* проблемы с поддержкой этого ~~добра~~.

# Что используем для решения проблемы
----

Будем использовать библиотеку [SmartEnum](https://github.com/ardalis/SmartEnum).

> На самом деле, это не панацея и подобный функционал реализуется за 15 минут, если по каким-то причинам вы не хотите тянуть лишнюю зависимость в ~~ваше детище~~ ваш проект.

Благодаря этой библиотеке мы описываем класс. А уже дальше работает с этим типом, как с enum. 

К этой библиотеке существует множество плагинов, таких как: сериализации, работа с EF и многое другое. (подробнее в репозитории)

# Задача
----

Есть список объектов, нужно сделать экспорт в несколько форматов. 

# Решение
----

Сразу опишу типы, которые будем использовать:

```csharp
// Объект для экспорта
public class Project
{
    public string Name { get; set; }
    public DateOnly Date { get; set; }
    public DateOnly Deadline { get; set; }
}

// Интерфейс для экспорта
public interface IProjectExporter : IDisposable
{
    void Export(ICollection<Project> projects);
}

public sealed record JsonExporter(Stream Stream) : IProjectExporter
{
    public void Export(ICollection<Project> items) => JsonSerializer.Serialize(Stream, items);
    public void Dispose() {}
}

public sealed class XmlExporter : IProjectExporter
{
    private readonly StreamWriter _streamWriter;
    private readonly XmlSerializer _xmlSerializer;
    private readonly List<Project> _list;

    public XmlExporter(Stream stream)
    {
        _streamWriter = new StreamWriter(stream, null, -1, true);
        _xmlSerializer = new XmlSerializer(typeof(List<Project>));
        _list = new List<Project>();
    }
            
    public void Export(ICollection<Project> items)
    {
        _list.Clear();
        _list.AddRange(items);
        _xmlSerializer.Serialize(_streamWriter, _list);
    }

    public void Dispose()
    {
        _streamWriter.Flush();
        _streamWriter.Dispose();
    }
}
```

## Для начала рассмотрим пример того, как это решается обычно:

```csharp
public enum ExportType
{
    Json,
    Xml
}

public static IProjectExporter CreateExporter(ExportType type, Stream stream)
{
    switch (type)
    {
        case ExportType.Json:
            return new JsonExporter(stream);
        case ExportType.Xml:
            return new XmlExporter(stream);
        default:
            throw new ArgumentException("Invalid export type");
    }
}

public static string GetContentType(ExportType type)
{
    switch (type)
    {
        case ExportType.Json:
            return "application/json";
        case ExportType.Xml:
            return "text/xml";
        default:
            throw new ArgumentException("Invalid export type");
    }
}

public static string GetExtension(ExportType type)
{
    switch (type)
    {
        case ExportType.Json:
            return "json";
        case ExportType.Xml:
            return "xml";
        default:
            throw new ArgumentException("Invalid export type");
    }
}
```

### Минусы очевидны
* При добавлении нового типа, необходимо найти все switch и поправь их. А если в проекте ещё несколько solution'ов, то будет совсем больно.
* При желании добавить новую логику, можно забыть case. Да да да, я знаю, что сейчас ide генерируют все автоматически, но я часто встречал switch, где не все типы указаны, потому что человек ~~забыл~~ 'знает', что другие варианты точно не могут тут быть.

### Примечание
Если заменить статические методы на extension, то может будет не так больно.

## Решение с использованием SmartEnum:

Сразу отмечу, что классы реализующие IProjectExporter, можно сделать приватными для JsonExportType и XmlExportType соответственно.

```csharp
public abstract class ExportType : SmartEnum<ExportType>
{
    private ExportType(string name, int value) : base(name, value)
    {
    }

    public static readonly ExportType Json = new JsonExportType(1);
    public static readonly ExportType Xml = new XmlExportType(2);
    
    public abstract string ContentType { get; }
    
    public abstract IProjectExporter CreateExporter(Stream destinationStream);
    
    private sealed class JsonExportType : ExportType
    {
        public JsonExportType(int value) : base("json", value) {}

        public override string ContentType => "application/json";

        public override IProjectExporter CreateExporter(Stream destinationStream) => new JsonExporter(destinationStream);
    }
    
    private sealed class XmlExportType : ExportType
    {
        public XmlExportType(int value) : base("xml", value)
        {
        }

        public override string ContentType => "text/xml";

        public override IProjectExporter CreateExporter(Stream destinationStream) => new XmlExporter(destinationStream);
    }
}
```

### Пример использования

```csharp
public void Export(ExportType type, Stream destination)
{
    using var exporter = type.CreateExporter(destination);
    foreach (var page in GetPaged(10))
        exporter.Export(page);
}
```

# Что делать, если нужно дать возможность расширять наш тип снаружи?
----

Если для какого-то из типов необходимы свои сервисы, отдельная обработка, то это решается тем же подходом, что и в прошлой статье.

Внимание, типы из ExportType делаем публичными, для возможности работы с конкретным типом.

Опишем интерфейс:

```csharp
public interface IExportTypeVisitor<out TRes>
{
    TRes Visit(ExportType.JsonExportType type);
    TRes Visit(ExportType.XmlExportType type);
}
```

Exporter'ы вынесем наружу и сделаем разные конструкторы, чтобы проблема была очевидна:

```csharp
public sealed record JsonExporter(Stream Stream) : IProjectExporter
{
    public void Export(ICollection<Project> items) => JsonSerializer.Serialize(Stream, items);
    public void Dispose() {}
}

public sealed class XmlExporter : IProjectExporter
{
    private readonly StreamWriter _streamWriter;
    private readonly XmlSerializer _xmlSerializer;
    private readonly List<Project> _list;

    public XmlExporter(Stream stream, int pageSize, Encoding? encoding = null)
    {
        _streamWriter = new StreamWriter(stream, encoding, -1, true);
        _xmlSerializer = new XmlSerializer(typeof(List<Project>));
        _list = new List<Project>(pageSize);
    }
            
    public void Export(ICollection<Project> items)
    {
        _list.Clear();
        _list.AddRange(items);
        _xmlSerializer.Serialize(_streamWriter, _list);
    }

    public void Dispose()
    {
        _streamWriter.Flush();
        _streamWriter.Dispose();
    }
}
```

## Вариант 1 - Visitor

```csharp
public abstract class ExportType : SmartEnum<ExportType>
{
    private ExportType(string name, int value) : base(name, value)
    {
    }

    public static readonly ExportType Json = new JsonExportType(1);
    public static readonly ExportType Xml = new XmlExportType(2);

    public abstract TRes Accept<TRes>(IExportTypeVisitor<TRes> visitor);
    
    public sealed class JsonExportType : ExportType
    {
        public JsonExportType(int value) : base("json", value) {}
        
        public override TRes Accept<TRes>(IExportTypeVisitor<TRes> visitor) => visitor.Visit(this);
    }
    
    public sealed class XmlExportType : ExportType
    {
        public XmlExportType(int value) : base("xml", value)
        {
        }

        public override TRes Accept<TRes>(IExportTypeVisitor<TRes> visitor) => visitor.Visit(this);
    }
}
```

Visitor для создания Exporter'а

```csharp
public sealed class ProjectExporterVisitor : IExportTypeVisitor<IProjectExporter>
{
    private readonly Stream _stream;
    private readonly int _pageSize;

    public ProjectExporterVisitor(Stream stream, int pageSize)
    {
        _stream = stream;
        _pageSize = pageSize;
    }

    public IProjectExporter Visit(ExportType.JsonExportType type) => new JsonExporter(_stream);

    public IProjectExporter Visit(ExportType.XmlExportType type) => new XmlExporter(_stream, _pageSize, Encoding.UTF8);
}
```

Собираем всё воедино:

```csharp
public void Export(ExportType type, Stream destination)
{
    const int pageSize = 10;
    using var exporter = type.Accept(new ProjectExporterVisitor(destination, pageSize));
    foreach (var page in GetPaged(pageSize))
        exporter.Export(page);
}
```

## Вариант 2 - match + Func

В отличии от прошлого способа, у нас пропадает 2 сущности:
* ProjectExporterVisitor
* IExportTypeVisitor

Теперь ExportType будет выглядеть следующим образом:

```csharp
public abstract class ExportType : SmartEnum<ExportType>
{
    private ExportType(string name, int value) : base(name, value)
    {
    }

    public static readonly ExportType Json = new JsonExportType(1);
    public static readonly ExportType Xml = new XmlExportType(2);

    public abstract TRes Match<TRes>(Func<JsonExportType, TRes> visitJson, Func<XmlExportType, TRes> visitXml);
    
    public sealed class JsonExportType : ExportType
    {
        public JsonExportType(int value) : base("json", value) {}
        
        public override TRes Match<TRes>(Func<JsonExportType, TRes> visitJson, Func<XmlExportType, TRes> visitXml) 
            => visitJson(this);
    }
    
    public sealed class XmlExportType : ExportType
    {
        public XmlExportType(int value) : base("xml", value)
        {
        }

        public override TRes Match<TRes>(Func<JsonExportType, TRes> visitJson, Func<XmlExportType, TRes> visitXml)
            => visitXml(this);
    }
}
```

Собираем всё воедино:

```csharp
static void Export(ExportType type, Stream destination)
{
    const int pageSize = 10;
    using var exporter = type.Match<IProjectExporter>(
        _ => new JsonExporter(destination),
        _ => new XmlExporter(destination, pageSize, Encoding.UTF8));
    
    foreach (var page in GetPaged(pageSize))
        exporter.Export(page);
}
```

Данный способ удобен, когда не нужно переиспользовать Visitor. По-моему с вариантов 2, код более читабелен.

## Вариант 3 - FuncVisitor

Просто комбинируем 2 подхода

```csharp
static void Export(ExportType type, Stream destination)
{
    const int pageSize = 10;
    using var exporter = type.Accept(new FuncExporterVisitor<IProjectExporter>(
        _ => new JsonExporter(destination),
        _ => new XmlExporter(destination, pageSize, Encoding.UTF8));
    
    foreach (var page in GetPaged(pageSize))
        exporter.Export(page);
}
```

## Что по производительности?

Конечно, использование Func или Visitor, несет за собой лишние аллокации. Я вижу 2 решения:

* Меняем сигнатуру метода **Accept**. Принимаем не: **IExportTypeVisitor\<T\>**, а **TVisitor where TVisitor : IExportTypeVisitor\<T\>** (да, код будет более многословен, придётся указывать явно 2 generic параметра), но тогда мы сможем описывать структуры, а не классы в качестве Visitor'ов

* Кэшируем Visitor'ы. Можно сделать мутабельными, но тут возникают следующие проблемы:
    * Параллельное использование
    * Удобством применения (нужно reset'ить значения, не забывать установить, перед вызовом)

# Итог
----

Не стоит плодить по коду switch для одного enum, особенно, если нужна разная логика обработки каждого типа. 

Пользуясь прекрасным и неоднозначным принципом [SRP](https://stackify.com/solid-design-principles/), чтобы собрать необходимую логику воедино.

# Ссылки
----
  
*   [Прошлая статья](enum_and_switch.md)
*   [SmartEnum](https://github.com/ardalis/SmartEnum)
*   [SRP](https://stackify.com/solid-design-principles/)