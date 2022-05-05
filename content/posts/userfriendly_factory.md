---
title: "Дружелюбная Factory"
date: 2022-05-01T19:46:31+03:00
draft: false
toc: false
images:
categories: ["programming", "csharp", "ru"]
tags:
  - factory
  - design
  - pattern
---

Сталкивались ли вы с проблемой, когда у вас есть **factory**, вы её создаёте там, где вам нужен определенный тип объекта и вы знаете, что нет необходимости передавать все зависимости в конструктор **factory**, так как они просто не нужны для создания. Тогда вы передаёте default/null?

Как правило, выглядит так себе:

```csharp
var factory = new ObjFactory(null, docId, dbService, null, null);
var doc = factory.Create(ObjType.Doc);
```

У данного подхода я не вижу плюсов от слова 'совсем'.

Давайте лучше перейдём к минусам:
* Непонятно что мы проставляем в default. Т.е мы должны помнить какой параметр находится в какой позиции, а это проблематично, особенно, когда их много. [Named arguments](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/named-and-optional-arguments) может улучшить ситуацию. 
* Мы должны знать/смотреть детали реализации, для корректного конфигурирования фабрики.
* При необходимости новой зависимости которая уже передаётся в конструктор, никто нас не ~~погладит по головке~~ ударит по рукам, за то, что мы её используем в фабрике, но не исправили передачу данного аргумента (хочется получить сразу же ошибку компиляции).

# Решение
----

Используем эту фабрику:

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

Есть несколько вариантов решения данной проблемы. Рассмотрим каждый из них отдельно.

## Вариант 1 - Фабричные методы

Фабрика сама знает какие зависимости для какого типа отчёта нужны, так давайте перенесем эту ответственность в саму factory, с использованием фабричных методов.

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

### Примечание
Можно сделать так, чтобы эти методы возвращали сразу IReport, для избежания вызова **factory.Create(ReportType.MonthEndInventoryValue)**, чтобы не дублировать тип, так как это ещё одно потенциальное место для ошибки.

### Плюсы 
* При необходимости новой зависимости, будем менять сигнатуру метода и код перестанет компилироваться в местах вызова.

### Минусы
* Если мы хотил создать фабрику для нескольких отчётов, то такой подход не подходит, рассмотрим это далее.


## Вариант 2 - Fluent interface

Идея: добавляем методы для инициализации нужных полей. 

ReportFactory будет выглядить следующим образом:

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
* Пустой конструктор, т.е мы можем создать абсолютно пустую ReportFactory.

## Вариант 3 - Группируем поля

Выносим все необходимые зависимости для отчёта в отдельные объекты, чтобы они изначально группировались логически:

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

Почему в Create просто не передать объект в конструктор отчёта? 

Можно, но все зависит от ситуации:

1. Возможно вы используете чужой код и там уже передаются параметры отдельно.

2. Перед созданием может быть логика получения и подготовки данных, на основе параметров, переданных в фабрику. Скорее всего в реальности, абсолютно другие данные будут передаваться в конструктор, после получения необходимой информации.

Ссылки
------

* [Named arguments](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/named-and-optional-arguments)
* [Factory](https://refactoring.guru/design-patterns/factory-method)
* [Fluent interface](https://en.wikipedia.org/wiki/Fluent_interface#:~:text=In%20software%20engineering%2C%20a%20fluent,Eric%20Evans%20and%20Martin%20Fowler.)