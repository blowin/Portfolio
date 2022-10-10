---
title: "Unit testing"
date: 2022-10-09T22:18:41+04:00
draft: false
toc: false
categories: ["programming", "csharp", "ru"]
images:
tags:
  - testing
  - unit
  - unit-testing
  - xUnit
---

> Количество дураков уменьшается, но качество их растет.
> 
> *Михаил Генин*

# Введение

Для начала разберемся что же такое unit тестирование. Cогласно википедии:

Модульное тестирование, иногда блочное тестирование или юнит-тестирование (англ. unit testing) — процесс в программировании, позволяющий проверить на корректность отдельные модули исходного кода программы, наборы из одного или более программных модулей вместе с соответствующими управляющими данными, процедурами использования и обработки.

Честно говоря, описание из википедии кажется мне абстрактным. Так как не передаёт важное отличие интеграционных тестов от unit.

Я бы дал следующее определение для unit тестирования:

Unit тестирование - процесс в программировании, позволяющий проверить на корректность отдельные модули исходного кода программы, который работает в оперативной памяти и не взаимодействует с внешними источниками (файловая система, база данных, сеть и т.д).

# Проблема

Предположим, что у нас есть репозиторий, который умеет сохранять и загружать пользователей в файл.

```csharp
public class UserRepository
{
    private readonly string _path;
    
    private static readonly JsonSerializerOptions Op = new()
    {
        WriteIndented = true
    };

    public UserRepository(string path)
    {
        _path = path ?? throw new ArgumentNullException(nameof(path));
    }

    public List<User> GetAll()
    {
        using var stream = File.Open(_path, FileMode.Open);
        var itemList = JsonSerializer.Deserialize<List<User>>(stream, Op);
        return itemList ?? new List<User>();
    }

    public void Save(List<User> items)
    {
        using var stream = File.Open(_path, FileMode.Create);
        JsonSerializer.Serialize(stream, items, Op);
    }
}
```

Модель пользователей выглядит следующим образом:

```csharp
public record User
{
    public Guid Id { get; set; }

    public string Name { get; set; }
    
    public DateTime BirthDate { get; set; }
    
    [JsonConstructor]
    public User(Guid id, string name, DateTime birthDate)
    {
        Id = id;
        Name = name;
        BirthDate = birthDate;
    }
}
```

На данный момент мы не можем написать unit тесты на этот код, так как в нём есть зависимость от файловой системы.

# Решение

Есть 2 основных способа избавиться от этой зависимости:
1. Создать интерфейс для работы с файловой системой **IFileProvider**, в котором будут все необходимые операции.
2. Создать базовый класс с абстрактным методом открытия файла, который будет возвращать **Stream**, а уже каждый наследник будет решать возвращать **FileStream**, **MemoryStream** или что-то ещё.

Лично мне импонирует первый подход, так как в нашем случае он является более гибким и нам наверняка понадобится больше 1 метода для работы с файловой системой, и удобнее будет иметь для этого отдельный объект.

## IFileProvider

Давайте опишем **IFileProvider**.

```csharp
public interface IFileProvider
{
    Stream Open(string path, FileMode mode);
}
```

Интерфейс будет очень прост - это 1 метод для открытия файла.

Сразу же пишем реализацию для реальной файловой системы:

```csharp
public class PhysicianFileProvider : IFileProvider
{
    public Stream Open(string path, FileMode mode) 
      => new FileStream(path, mode);
}
```

## Дорабатываем репозиторий

Изменения будут минимальными, заменяем 2 вызова **File.Open**, на вызов нашего провайдера.

```csharp
public class UserRepository
{
    private readonly string _path;
    private readonly IFileProvider _fileProvider;

    private static readonly JsonSerializerOptions Op = new()
    {
        WriteIndented = true
    };

    // Новый параметр в конструктор                   ↓
    public UserRepository(string path, IFileProvider? fileProvider = null)
    {
        _path = path ?? throw new ArgumentNullException(nameof(path));
        _fileProvider = fileProvider ?? new PhysicianFileProvider();
    }

    public List<User> GetAll()
    {
        // Меняем вызов             ↓
        using var stream = _fileProvider.Open(_path, FileMode.Open);
        var itemList = JsonSerializer.Deserialize<List<User>>(stream, Op);
        return itemList ?? new List<User>();
    }

    public void Save(List<User> items)
    {
        // Меняем вызов             ↓
        using var stream = _fileProvider.Open(_path, FileMode.Create);
        JsonSerializer.Serialize(stream, items, Op);
    }
}
```

После таких минимальных изменений, мы сможем заменить работу с файловой системой, на оперативную память, сеть или любой другой источник хранения.

## Пишем тесты

В качестве библиотеки для тестирования будем использовать [xUnit](https://xunit.net/). Это дело вкуса, можно использовать любую другую библиотеку для Unit тестирования.

Для работы тестов, нам понадобится написать отдельную реализацию **IFileProvider**.

```csharp
public sealed class MemoryFileProvider : IFileProvider
{
    private readonly NonDisposableMemoryStream _memStream = new();

    public Stream Open(string path, FileMode mode)
    {
        _memStream.Position = 0;
        return _memStream;
    }

    public void Delete(string path) { }

    private sealed class NonDisposableMemoryStream : MemoryStream
    {
        protected override void Dispose(bool disposing) {}
    }
}
```

В рамках теста у нас будет создаваться 1 файл, поэтому можно сделать объект с 1 полем, но для корректной работы тестов, нам необходима возможность переиспользовать **MemoryStream** в конструкции **using**, поэтому создаём наследника и переопределяем метод **Dispose**.

Для удобства тестирования, напишем ещё 2 метода расширения для **IFileProvider**, для того, чтобы положить в него json и достать. Их можно хранить в сборке unit тестов.

```csharp
public static string ToJson(this IFileProvider self, string path)
{
    var stream = self.Open(path, FileMode.Open);
    using var streamReader = new StreamReader(stream);
    return streamReader.ReadToEnd();
}

public static void WithJson(this IFileProvider self, string path, 
  string json)
{
    var stream = self.Open(path, FileMode.OpenOrCreate);
    using var writer = new StreamWriter(stream);
    writer.Write(json);
}
```

### MemberData

Данные для тестирования выглядят следующим образом

```csharp
public static IEnumerable<object[]> TestData
{
    get
    {
        return new List<object[]>
        {
            new object[]
            {
                @"[]",
                new List<User>()
            },
            new object[]
            {
                @"[
{
""Id"": ""2d445f82-004f-4b91-ba16-4bfcd24d96e8"",
""Name"": ""Test"",
""BirthDate"": ""2022-10-10T00:00:00""
}
]",
                new List<User>
                {
                    new(Guid.Parse("2d445f82-004f-4b91-ba16-4bfcd24d96e8"), 
                      "Test", 
                      new DateTime(2022, 10, 10))
                }
            },
            new object[]
            {
                @"[
{
""Id"": ""2d445f82-004f-4b91-ba16-4bfcd24d96e8"",
""Name"": ""Test"",
""BirthDate"": ""2022-10-10T00:00:00""
},
{
""Id"": ""2d445f82-004f-4b91-ba16-4bfcd24d96e9"",
""Name"": ""Test 2"",
""BirthDate"": ""2021-10-10T00:00:00""
}
]",
                new List<User>
                {
                    new(Guid.Parse("2d445f82-004f-4b91-ba16-4bfcd24d96e8"), 
                      "Test", 
                      new DateTime(2022, 10, 10)),
                    new(Guid.Parse("2d445f82-004f-4b91-ba16-4bfcd24d96e9"), 
                      "Test 2", 
                      new DateTime(2021, 10, 10)),
                }
            },
        };
    }
}
```

### Метод Save

```csharp
[Theory]
[MemberData(nameof(TestData))]
public void Save(string expect, List<User> items)
{
    // Arrange
    const string filePath = "tmp.json";

    var fileProvider = new MemoryFileProvider();
    var repository = new UserRepository(filePath, fileProvider);

    // Act
    repository.Save(items);
    // Вызываем метод расширение, для проверки json
    var json = fileProvider.ToJson(filePath);

    // Assert
    Assert.Equal(expect, json);
}
```

### Метод GetAll

Он будет очень похож на тестирование метода **Save**, можно даже использовать **MemberData** из прошлого теста.

```csharp
[Theory]
[MemberData(nameof(TestData))]
public void GetAll(string initialValue, List<User> expect)
{
    // Arrange
    const string filePath = "tmp.json";

    var fileProvider = new MemoryFileProvider();

    // Заполняем стрим, используя метод расширение
    fileProvider.WithJson(filePath, initialValue);
    var repository = new UserRepository(filePath, fileProvider);

    // Act
    var result = repository.GetAll();

    // Assert
    Assert.Equal(expect, result);
}
```

# Интеграционное тестирование

С существующей реализацией можно легко написать интеграционные тесты на наш репозиторий. Для этого добавим 1 метод в **IFileProvider**

```csharp
public interface IFileProvider
{
    Stream Open(string path, FileMode mode);

    void Delete(string path);
}
```

Добавим метод в **PhysicianFileProvider**

```csharp
public void Delete(string path) => File.Delete(path);
```

Добавим метод в **MemoryFileProvider**

```csharp
public void Delete(string path) {}
```

Доработаем наши тесты, для этого обернем весь код в try finally, чтобы удалять файл по окончанию теста. Приведу пример теста метода **Save**, изменения во втором будут аналогичны:

```csharp
[Theory]
[MemberData(nameof(TestData))]
public void Save(string expect, List<User> items)
{
    // Arrange
    const string filePath = "tmp.json";

    var fileProvider = new MemoryFileProvider();
    try
    {
        var repository = new UserRepository(filePath, fileProvider);

        // Act
        repository.Save(items);
        var json = fileProvider.ToJson(filePath);
    
        // Assert
        Assert.Equal(expect, json);
    }
    finally
    {
        try
        {
            // Удаляем файл по окочанию теста
            fileProvider.Delete(filePath);
        }
        catch {}
    }
}
```

Теперь мы можем заменить **new MemoryFileProvider()** на **new PhysicianFileProvider()**, тогда тест будет тестировать взаимодействие с файловой системой. При желании мы можем передавать нужную реализацию **IFileProvider** в качестве аргумента теста.

# Итог

Тестирование кода помогает помочь в проектировании класса и улучшить дизайн. Порой это приводит к избыточности, но это слихвой покрывется надёжностью кода.

Благодаря тестированию, процесс разработки идёт куда быстрее, так как разработчики меньше боятся поломать существующий функционал, например, очередным рефакторингом или добавлением новой функциональности.

# Ссылки

* [Модульное тестирование (Wiki)](https://ru.wikipedia.org/wiki/%D0%9C%D0%BE%D0%B4%D1%83%D0%BB%D1%8C%D0%BD%D0%BE%D0%B5_%D1%82%D0%B5%D1%81%D1%82%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5)
* [xUnit](https://xunit.net/)
* [Юнит-тестирование для чайников (Хабр)](https://habr.com/ru/post/169381/)
* [Тестирование в .NET (Microsoft)](https://learn.microsoft.com/ru-ru/dotnet/core/testing/)
* [Рекомендации по модульному тестированию для .NET Core и .NET Standard (Microsoft)](https://learn.microsoft.com/ru-ru/dotnet/core/testing/unit-testing-best-practices)
