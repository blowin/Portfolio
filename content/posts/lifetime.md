---
title: "Умная ути-ути-утилизация"
date: 2022-09-12T07:09:06+03:00
draft: false
toc: false
categories: ["programming", "csharp", "ru", "lib hunter"]
images:
tags:
  - dispose
  - lifetime
---

> После очистки нашей истории от лжи не обязательно должна остаться только правда, порой вообще ничего не остается.
> 
> *Станислав Ежи Лец*

# Введение

Сегодня поговорим об управлении ресурами, а именно о методе Dispose и библиотеке [Lifetime](https://github.com/JetBrains/rd) от Jetbrains, которая помогает управлять утилизацией ресурсов.

IDisposable - Интерфейс, который предоставляет собой механизм для освобождения неуправляемых ресурсов (согласно MSDN).

Очень простое определение, за которым скрывается большое количество нюансов, но об этом далее.

# Проблема

Так что же не так с интерфейсом [IDisposable](https://docs.microsoft.com/ru-ru/dotnet/api/system.idisposable?view=net-6.0)? 

Сталкивались ли вы с вопросами на собеседовании "Как корректно реализовать dispose"? Ответили ли вы на него правильно с первого раза?

1. Сложность правильной реализации, это настолько не тривиальная задача, что у [Microsoft есть отдельная статья](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose) на эту тему. С добавлением [IAsyncDisposable](https://docs.microsoft.com/en-us/dotnet/api/system.iasyncdisposable?view=net-6.0) ситуация стала только запутаннее.
2. Когда нам передаётся объект, который реализует IDisposable, нам тоже нужно реализовать IDisposable интерфейс и вызвать соответствующий метод объекта, но к сожалению мы не всегда должны это делать. Например, жизненным циклом объекта должна управлять используемая библиотека(привет DI) или же кто-то на уровень выше (привет Parent form из WinForms).
3. Как определить, можем ли мы вызвать Dispose на объекте. Это является проблемой настолько, что в [некоторых классах стандартной библиотеки есть даже специальный флаг](https://docs.microsoft.com/en-us/dotnet/api/system.io.streamreader.-ctor?view=net-6.0#system-io-streamreader-ctor(system-string-system-boolean)), который нужно установить в true, если класс не должен вызвать Dispose у передаваемого объекта (см конструктор с *leaveOpen*).
4. Мы должны знать что тип, который нам приходит в конструкторе реализует IDiposable. 
5. Метод Dispose, как правило, располагают в конце класа, что влечет за собой возможность забыть добавить вызов Dispose для нового объекта переданного в конструктор.

# Решение

В Jetbrains придумали изящное решение этой проблемы с помощью инвертирования зависимостей. Теперь за уничтожение объекта отвечает не наш объект, а кто-то на уровень выше, объект только сообщает ему как это сделать. 

На Github есть исходный код библиотеки [RD](https://github.com/JetBrains/rd), которая содержит реализацию Lifetime. 

Есть реализации для: 
* [C++](https://github.com/JetBrains/rd/tree/master/rd-cpp).
* [C#](https://github.com/JetBrains/rd/tree/master/rd-net).
* [Kotlin](https://github.com/JetBrains/rd/tree/master/rd-kt).

Данных подход широко используется в [ReSharper Platform SDK](https://www.jetbrains.com/help/resharper/sdk/welcome.html), в его документации можно посмотреть более подробное описание, а я в свою очередь приведу пару примеров.

# Примеры 

Давайте сравним старый подход с Dispose и альтернативный вариант от Jetbrains.

## Версия с Dispose

```csharp
public class Service1 : IDisposable
{
    public void Dispose() => Console.WriteLine("Dispose Service1");
}

public class Service2
{
    public event Action? ItemAdded;
}

public class Root : IDisposable
{
    private Service1 _service1;
    private Service2 _service2;

    public Root(Service1 service1, Service2 service2)
    {
        _service1 = service1;
        _service2 = service2;
        service2.ItemAdded += Service2OnItemAdded;
    }

    public void Dispose()
    {
        _service2.ItemAdded -= Service2OnItemAdded;
        _service1.Dispose();
        Console.WriteLine("Dispose Root")
    }
}

// Main
using var root = new Root(new Service1(), new Service2());
root.Run();
```

## Добавим Lifetime

```csharp
// Не реализуем интерфейс IDisposable
public class Service1
{
    // Указываем, что по окончания жизни Lifetime, нужно очистить ресурсы
    public Service1(Lifetime lifetime) => lifetime.OnTermination(() => Console.WriteLine("Dispose Service1"));
}

// Service2 без изменений

// Не реализуем интерфейс IDisposable
public class Root
{
    private Service1 _service1;
    private Service2 _service2;

    public Root(Lifetime lifetime, Service1 service1, Service2 service2)
    {
        _service1 = service1;
        _service2 = service2;
        // Указываем, что хотим подписаться и сразу же указываем, что делать по окончанию жизни.
        lifetime.Bracket(() => service2.ItemAdded += Service2OnItemAdded,
            () => service2.ItemAdded -= Service2OnItemAdded);
        // Указываем, что нужно делать с текущим объектом по окончанию Lifetime
        lifetime.OnTermination(() => Console.WriteLine("Dispose Root"));
    }
}

// Main
// Создаём корневой LifetimeDefinition, а от него создаём Lifetime
using var lifetimeDef = new LifetimeDefinition();
var root = new Root(lifetimeDef.Lifetime, new Service1(lifetimeDef.Lifetime), new Service2());
root.Run();
// при вызове Dispose на LifetimeDefinition, будут уничтожено все дерево объектов, причём в правильном порядке.
```

## Дочерние LifetimeDefinition

В нашем приложении могут быть сущности с более коротким временем жизни, это как Scope из DI, чтобы такое осуществить, можно создавать дочерние LifetimeDefinition, которые могут завершиться раньше, чем родительский Lifetime.

Обратите внимание, что в таком случае, наш Service1 будет реализовывать IDisposable и Dispose нужно будет вызвать снаружи, либо создать LifetimeDefinition на уровень выше и передавать его Lifetime в Service1. 

```csharp
public class Service1 : IDisposable
{
    private readonly LifetimeDefinition _lifetimeDefinition;
    
    public Service1(Lifetime lifetime)
    {
        _lifetimeDefinition = new LifetimeDefinition(lifetime);
        _lifetimeDefinition.Lifetime.OnTermination(() => Console.WriteLine("Dispose Service1"));
    }

    public void Dispose() => _lifetimeDefinition.Dispose();
}
```

С помощью LifetimeDefinition реализуется древовидная структура. 

В приложении каждый компонент имеет своё время жизни, например, есть главный Lifetime (всего приложения), если он схлопывается, то всё должно утилизироваться. От него могут создаваться дочерние LifetimeDefinition, например, при открытии окна, которое в свою очередь может внутри породить новый LifetimeDefinition.

Если закрывается родительский LifetimeDefinition, то все дочерние компоненты будут утилизированы, но если закрывается дочерний LifetimeDefinition, то это никак не влияет на родительский. 

# Итог

С помощью Lifetime можно решить следующие проблемы:

- Разделение конструктора и метода Dispose.
- Теперь тип явно декларирует в конструкторе, что ему необходимо утилизировать свои ресурсы.
- Нет необходимости знать нужно ли утилизировать конкретный объект. Каждый объект сам несет ответственность за это.

Безусловно в этом подходе есть и минусы:
- Поддержка в DI.
  
Этот минус может стать большой проблемой для внедрения этого подхода в проект. Jetbrains научила использовать Lifetime свой DI, но к сожалению DI от Jetbrains не Open Source. 

# Ссылки

* [Lifetime (Github)](https://github.com/JetBrains/rd)
* [Implementating dispose (MSDN)](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose)
* [IAsyncDisposable (MSDN)](https://docs.microsoft.com/en-us/dotnet/api/system.iasyncdisposable?view=net-6.0)
* [IDisposable (MSDN)](https://docs.microsoft.com/ru-ru/dotnet/api/system.idisposable?view=net-6.0)
  
# Доклады

* [Дмитрий Иванов «Библиотека JetBrains.Lifetimes — новый взгляд на реактивное программирование» (DotNetRu)](https://www.youtube.com/watch?v=Sq_h5bVWJ0k)
* [Дмитрий Иванов — Реактивное многопроцессное взаимодействие: JetBrains Rider Framework (DotNext)](https://www.youtube.com/watch?v=cfPEN5_6UtI)
* [Станислав Сидристый «Шаблон Lifetime: для сложного Disposing» (DotNetRu)](https://www.youtube.com/watch?v=F5oOYKTFpcQ&list=PLBwwJL9lzKMbozGTh4k1VC67O5fLezyyB&index=6)