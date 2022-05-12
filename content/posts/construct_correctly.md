---
title: "Конструируй правильно"
date: 2022-05-11T17:29:30+03:00
draft: false
toc: false
categories: ["programming", "csharp", "ru"]
images:
tags:
  - constructor
  - design
---

Конструктор - это точка входа в любой объект. Это метод, который служит инициализатором вашего типа, проверяет инварианты, переводит объект в состояние пригодное для использования.

Каждый день мы пишем свои типы, они в дальнейшем будут использовать и наши коллеги. Но сталкивались ли вы с тем, что после создания объекта вы получали ошибки связанные с тем, что после создания объекта какие-то из полей не были проинициализированы? Если не встречали, то вам очень повезло, к сожалению, я не из таких людей.

Так почему же возникают такие проблемы, если конструктор является такой же важной частью типа, как и его методы, которым нужно уделять не меньшее внимания?

Давайте разбираться, для этого будем использовать такой тип:

```csharp
public class Tree<T>
{
    private Node? _root;
    private IEqualityComparer<T> _comparer;

    public Tree(IEnumerable<T> items)
    {
      // ...
    }
    
    public Tree(IEnumerable<T> items, IEqualityComparer<T> comparer)
    {
        // ...
    }
    
    public Tree(IEqualityComparer<T> comparer)
    {
        // ...
    }

    private void AddRange(IEnumerable<T> items)
    {
      // ...
    }
    
    private class Node
    {
        // ...
    }
}
```

# Распространенная реализация

```csharp
public Tree(IEnumerable<T> items)
{
  _comparer = EqualityComparer<T>.comparer;
  AddRange(items);
}

public Tree(IEnumerable<T> items, IEqualityComparer<T> comparer)
{
    _comparer = comparer;
  AddRange(items);
}

public Tree(IEqualityComparer<T> comparer)
{
    _comparer = comparer ?? throw new ArgumentNullException(nameof(comparer));
}
```

В примере выше есть ошибка, заметили ли вы её? 

Давайте изучим код ещё раз, подумаем и БИНГО! В конструкторе 2, мы забыли сделать проверку на null. Получается, что при вызове этого конструктора и передаче туда null для **comparer**, нормальной ошибки от класса мы не получим и упадём в **AddRange**.

## Минусы
* В каждом конструкторе мы дублируем логику инициализации.
* Неинициализированные поля.
* Поменяли логику в одном конструкторе и оставили остальные без изменений.
* Разбухание кода, что влечет к его усложнению и затруднению в поддержке.

# Первичный конструктор

В типе всегда должен быть главный конструктор и остальные конструкторы в конечном счете всегда должны вызывать именно его. Тогда у нас будет единая точка, для инициализации типа, что помогает решить огромный пласт ошибок. 

Давайте исправим наш пример:

```csharp
public Tree(IEnumerable<T> items)
        : this(items, EqualityComparer<T>.Default)
{
}

public Tree(IEnumerable<T> items, IEqualityComparer<T> comparer)
    : this(comparer)
{
    AddRange(items);
}

public Tree(IEqualityComparer<T> comparer)
{
    _comparer = comparer ?? throw new ArgumentNullException(nameof(comparer));
}
```

У нас есть главный конструктор **Tree(IEqualityComparer<T> comparer)**, который вызывают все остальные в конечном счете. 

Мы знаем, что нашему типу обязательно нужен **IEqualityComparer<T>**, давайте его требовать в главном конструкторе. Там мы можем добавить проверку на null и мы точно будем знать, что там есть все нужные проверки.

Обратите внимание, что конструктор 1, вызывает конструктор 2, который в конечном счете вызывает главный конструктор (номер 3). 

# А что сейчас?

С появлением [record](https://kotlinlang.org/docs/classes.html#constructors). Данную идею включили в язык, чему я безумно рад. Но не каждый может использовать необходимую версию языка и не всегда нужно использовать record, для его компилятор требует вызова primary constructor.

```csharp
public record StreamAsString(Stream Stream)
{
    // без this(File.OpenRead(path)), не компилируется
    public StreamAsString(string path) : this(File.OpenRead(path))
    {
    }

    public string Content
    {
        get
        {
            using var reader = new StreamReader(Stream);
            return reader.ReadToEnd();
        }
    }
}
```

# Итог

Переиспользуя готовые конструкторы, мы перекладываем ответственность проверки на них и не будем задумываться о том, какие поля нам нужно инициализировать или как это правильно сделать. 

В конечном счете это кто-то должен будет реализовать, но это будет единая точка, что очень упрощает понимание и поддержание такого кода.

# Ссылки

*   [Record (C#)](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/tutorials/records)
*   [Primary constructor на уровне языка (Kotlin)](https://kotlinlang.org/docs/classes.html#constructors)