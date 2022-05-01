---
title: "Дружелюбная Factory"
date: 2022-05-01T19:46:31+03:00
draft: false
toc: false
images:
categories: ["programming"]
tags:
  - csharp
  - factory
  - design
  - pattern
---

Сталкивались ли вы с проблемой, когда у вас есть фабрика по созданию объектов, вы её создаётся в месте, где вам нужен объект и есть зависимости, которые в данном случае точно не нужны для создания вашего объекта и вы либо их проставляется в default, либо же вообще пытаетесь передать?

Как правило, выглядит это ужасно:

```csharp
var factory = new ObjFactory(null, docId, dbService, null, null);
var doc = factory.Create(ObjType.Doc);
```

У данного подхода я не вижу плюсов от слова 'совсем'.

Давайте лучше перечислим минусы:
* Непонятно, что мы проставляем в default (да, [named arguments](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/named-and-optional-arguments) может улучшить ситуацию).
* Мы должны знать/смотреть детали реализации, для корректного конфигурирования фабрики.
* При необходимости новой зависимости которая уже передаётся в конструктор, никто нас не ~~погладит по головке~~ ударит по рукам, за то, что мы её используем в фабрике, но не исправили передачу данного аргумента (хочется получить сразу же ошибку компиляции).

# Решение
----

Будем использовать эту фабрику:

```csharp

public enum ReportType 
{
  SellThroughRateByProduct,
  MonthEndInventoryValue,
  PercentOfInventorySold
}

public interface IReport { }

public sealed record SellThroughRateByProductReport(ICollection<Guid> Products) : IReport;

public sealed record MonthEndInventoryValueReport(DateOnly Date) : IReport;

public sealed record PercentOfInventorySoldReport(ICollection<Guid> TypeOfProducts, DateOnly StartDate, DateOnly EndDate) : IReport;

public sealed class ReportFactory
{
    private readonly ICollection<Guid> _products;
    private readonly DateOnly _date;
    private readonly ICollection<Guid> _typeOfProducts;
    private readonly DateOnly _startDate;
    private readonly DateOnly _endDate;

    public ReportFactory(ICollection<Guid> products, DateOnly date, ICollection<Guid> typeOfProducts, DateOnly startDate, DateOnly endDate)
    {
        _products = products;
        _date = date;
        _typeOfProducts = typeOfProducts;
        _startDate = startDate;
        _endDate = endDate;
    }

    public IReport Create(ReportType type)
    {
        switch (type)
        {
            case ReportType.SellThroughRateByProduct:
                // сложный код по получению данных, инициализации и т.д 
                return new SellThroughRateByProductReport(_products);
            case ReportType.MonthEndInventoryValue:
                // сложный код по получению данных, инициализации и т.д
                return new MonthEndInventoryValueReport(_date);
            case ReportType.PercentOfInventorySold:
                // сложный код по получению данных, инициализации и т.д
                return new PercentOfInventorySoldReport(_typeOfProducts, _startDate, _endDate);
            default:
                throw new ArgumentOutOfRangeException(nameof(type), type, null);
        }
    }
}

```

У меня есть несколько вариантов решения данной проблемы. Рассмотрим каждый из них отдельно

## Вариант 1 - фабричные методы

Идея проста, фабрика сама знает какие зависимости для какого типа отчёта нужны, так давайте перенесем эту ответственность в саму factory, с использованием фабричных методов.

Дорабатываем ReportFactory:

```csharp
public static ReportFactory ForSellThroughRateByProduct(ICollection<Guid> products) =>
        new ReportFactory(products, default, Array.Empty<Guid>(), default, default);

public static ReportFactory ForMonthEndInventoryValue(DateOnly date) 
    => new ReportFactory(Array.Empty<Guid>(), date, Array.Empty<Guid>(), default, default);

public static ReportFactory ForPercentOfInventorySold(ICollection<Guid> typeOfProducts, DateOnly startDate, DateOnly endDate) 
    => new ReportFactory(Array.Empty<Guid>(), default, typeOfProducts, startDate, endDate);
```

### Использование
```csharp
var factory = ReportFactory.ForMonthEndInventoryValue(new DateOnly(2021, 1, 1));
var report = factory.Create(ReportType.MonthEndInventoryValue);
```

По-моему куда лучше. Можно сделать так, чтобы эти методы возвращали сразу IReport, для избежания вызова **factory.Create(ReportType.MonthEndInventoryValue)**, чтобы не дублировать тип, так как это ещё одно потенциальное место для ошибки.

### Плюсы 
* При необходимости новой зависимости, будем менять сигнатуру метода и код перестанет компилироваться в местах вызова.

### Минусы
* Что, если мы хотил создать фабрику для нескольких отчётов? (рассмотрим это далее).


## Вариант 2 - fluent interface

Идея: добавляем методы для инициализации нужных полей. 

ReportFactory будет это выглядить следующим образом:

```csharp
// Новый конструктор
public ReportFactory() 
        : this(Array.Empty<Guid>(), default, Array.Empty<Guid>(), default, default)
{
}

// Методы инициализации

public ReportFactory ForSellThroughRateByProduct(ICollection<Guid> products)
{
    _products = products;
    return this;
}

public ReportFactory ForMonthEndInventoryValue(DateOnly date)
{
    _date = date;
    return this;
}

public ReportFactory ForPercentOfInventorySold(ICollection<Guid> typeOfProducts, DateOnly startDate,
    DateOnly endDate)
{
    _typeOfProducts = typeOfProducts;
    _startDate = startDate;
    _endDate = endDate;
    return this;
}
```

### Использование
```csharp
var factory = new ReportFactory()
    .ForMonthEndInventoryValue(new DateOnly(2021, 1, 1))
    .ForSellThroughRateByProduct(new List<Guid> { bookId, tShirtId });
var report1 = factory.Create(ReportType.MonthEndInventoryValue);
var report2 = factory.Create(ReportType.SellThroughRateByProduct);
```

### Плюсы 
* При необходимости новой зависимости, будем менять сигнатуру метода и код перестанет компилироваться в местах вызова.
* Можем сконфигурировать фабрику для нужного количества отчётов.

### Минусы
* Пустой конструктор.

## Вариант 3 - группируем поля

Я бы ещё предложил вынести все необходимые зависимости для отчёта в отдельные объекты, чтобы они изначально группировались логически:

```csharp
public sealed record SellThroughRateByProductParameter(ICollection<Guid> Products);

public sealed record MonthEndInventoryValueReportParameter(DateOnly Date);

public sealed record PercentOfInventorySoldReportParameter(ICollection<Guid> TypeOfProducts, DateOnly StartDate, DateOnly EndDate);
```

Меняем ReportFactory:

```csharp
public sealed class ReportFactory
{
    private readonly SellThroughRateByProductParameter _sellThroughRateByProductParameter;
    private readonly MonthEndInventoryValueReportParameter _monthEndInventoryValueReportParameter;
    private readonly PercentOfInventorySoldReportParameter _percentOfInventorySoldReportParameter;

    public ReportFactory(SellThroughRateByProductParameter sellThroughRateByProductParameter, 
        MonthEndInventoryValueReportParameter monthEndInventoryValueReportParameter, 
        PercentOfInventorySoldReportParameter percentOfInventorySoldReportParameter)
    {
        _sellThroughRateByProductParameter = sellThroughRateByProductParameter;
        _monthEndInventoryValueReportParameter = monthEndInventoryValueReportParameter;
        _percentOfInventorySoldReportParameter = percentOfInventorySoldReportParameter;
    }

    public IReport Create(ReportType type)
    {
        switch (type)
        {
            case ReportType.SellThroughRateByProduct:
                return new SellThroughRateByProductReport(_sellThroughRateByProductParameter.Products);
            case ReportType.MonthEndInventoryValue:
                return new MonthEndInventoryValueReport(_monthEndInventoryValueReportParameter.Date);
            case ReportType.PercentOfInventorySold:
                return new PercentOfInventorySoldReport(_percentOfInventorySoldReportParameter.TypeOfProducts, _percentOfInventorySoldReportParameter.StartDate, _percentOfInventorySoldReportParameter.EndDate);
            default:
                throw new ArgumentOutOfRangeException(nameof(type), type, null);
        }
    }
}
```

Почему в Create я не передаю просто объект в конструктор? 

Все зависит от ситуации:

1. Возможно вы используете чужой код и там передаются параметры отдельно.

2. Перед созданием может быть логика получения данных, подготовка каких-то данных, на основе параметров, переданных в фабрику и в реальности другие данные будут передаваться в конструктор.

Этот подход можно использовать с теми, что были указаны выше.

Ссылки
------

* [Named arguments](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/named-and-optional-arguments)
* [Factory](https://refactoring.guru/design-patterns/factory-method)
* [Fluent interface](https://en.wikipedia.org/wiki/Fluent_interface#:~:text=In%20software%20engineering%2C%20a%20fluent,Eric%20Evans%20and%20Martin%20Fowler.)