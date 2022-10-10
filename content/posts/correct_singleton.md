---
title: "Как правильно быть одиноким?"
date: 2022-07-07T08:28:55+03:00
draft: false
toc: false
categories: ["programming", "csharp", "ru"]
images:
tags:
  - singleton
  - design
  - pattern
---

> Мне было одиноко, но удобно.
> 
> *Дэниел Фордж*

# Введение

Сегодня будем говорить про Singleton и то как стоит его использовать.

Одиночка (англ. Singleton) — порождающий шаблон проектирования, гарантирующий, что в однопоточном приложении будет единственный экземпляр некоторого класса, и предоставляющий глобальную точку доступа к этому экземпляру. (источник - Wiki)

В качестве примера будем использовать следующую сущность.

```csharp
public record Status(Guid Id, string Name);
```

# Проблема

Приходилось ли вам видеть код подобного рода?

```csharp
public static class StatusLoader
{
    public static Status[] Load()
    {
        // http/db call 
        return Array.Empty<Status>();
    }
}

// Используем
var statuses = StatusLoader.Load();
```

Что в этом коде плохо? На самом деле всё:
* Большая связанность
* Нет возможности заменить реализацию
* Нет возможности расширить реализацию

# Решение

Что делать, если ~~чешутся руки~~ очень хочется? Точно никогда не использовать *static class*. 

Статические классы можно использовать только если нужен класс с *extension*, во всех остальных случаях нужно реализовывать правильный singleton.

Что я имею в виду под правильный singleton'ом?

* У него есть свойство *instance*
* Потокобезопасный
* Расширяемый
* Заменяемый

Нужно соблюдать дополнительные правила, чтобы наш код не превратился в такой.

```csharp
var statuses = StatusLoader.Instance.Load();
```

Для этого мы НИКОГДА не обращаемся напрямую к свойству instance, для вызова метода/свойства. Всегда требуем в конструктор наш объект, а уже работаем с ним, как с обычным объектом. 

Правильный StatusLoader будет выглядеть следующим образом:

```csharp
public interface IStatusLoader
{
    Status[] Load();
}

public class StatusLoader : IStatusLoader
{
    private static readonly Lazy<StatusLoader> LazyInstance = new Lazy<StatusLoader>(() => new StatusLoader());

    public static StatusLoader Instance => LazyInstance.Value;

    public Status[] Load()
    {
        // http/db call 
        return Array.Empty<Status>();
    }
}
```

При таком подходе, мы можем реализовывать интерфейсы и в классе, где нам нужен *StatusLoader.Instance*, мы в конструктор передаём *IStatusLoader*. 

При таком подходе легко пишутся тесты и расширяется реализация, например, добавлением кэширования. Это можно сделать написанием прокси или созданием новых наследников.

Пример кэширующего прокси:

```csharp
public sealed class CacheStatusLoader : IStatusLoader
{
    private readonly IStatusLoader _origin;
    private Status[]? _cacheResult;

    public CacheStatusLoader(IStatusLoader origin) => _origin = origin;

    public Status[] Load() => _cacheResult ??= _origin.Load() ?? Array.Empty<Status>();
}
```

Теперь можно передавать загручик следующим образом:

```csharp
var loader = new CacheStatusLoader(StatusLoader.Instance);
DoSomeWork(loader);
```

# Итог

Не используйте статические классы. 

Если хотите сделать статический класс или он у вас уже имеется, то делайте это через singleton, при этом НИКОГДА не вызывайте методы следующим образом *Singleton.Instance.Do()*.

# Ссылки

* [Singleton pattern](https://en.wikipedia.org/wiki/Singleton_pattern)