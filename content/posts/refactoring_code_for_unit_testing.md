---
title: "Рефакторинг кода для unit тестирования"
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

# Цель

Цель статьи - показать как можно решить проблему при написании юнит тестов, когда в коде есть зависимости на внешний ресурс (база данных/сеть/файловая система).

# Введение

Для начала разберемся что же такое unit тестирование. Cогласно википедии:

Модульное тестирование, иногда блочное тестирование или юнит-тестирование (англ. unit testing) — процесс в программировании, позволяющий проверить на корректность отдельные модули исходного кода программы, наборы из одного или более программных модулей вместе с соответствующими управляющими данными, процедурами использования и обработки.

Честно говоря, описание из википедии кажется мне абстрактным. Так как не передаёт важное отличие интеграционных тестов от unit.

Я бы дал следующее определение для unit тестирования:

Unit тестирование - процесс в программировании, позволяющий проверить на корректность отдельные модули исходного кода программы, который работает в оперативной памяти и не взаимодействует с внешними источниками (файловая система, база данных, сеть и т.д).

# Проблема

Необходимо протестировать алгоритм группировки файлов по расширению.

```csharp
public class FileGrouper
{
    private readonly string _folderPath;

    public FileGrouper(string folderPath) => _folderPath = folderPath;

    public void ByExtension()
    {
        var groupFiles = Directory.GetFiles(_folderPath)
            .Select(fullFileName => new
            {
                FullFileName = fullFileName,
                FileName = Path.GetFileName(fullFileName),
                Extension = Path.GetExtension(fullFileName)
            })
            .Where(fileInfo => !string.IsNullOrEmpty(fileInfo.Extension))
            .ToLookup(arg => arg.Extension);

        foreach (var groupFile in groupFiles)
        {
            var newFolder = Path.Combine(_folderPath, groupFile.Key);
            if (!Directory.Exists(newFolder))
                Directory.CreateDirectory(newFolder);

            foreach (var file in groupFile)
            {
                var newFile = Path.Combine(newFolder, file.FileName);
                File.Move(file.FullFileName, newFile);
            }
        }
    }
}
```

На данный момент мы не можем написать unit тесты на этот код, так как в нём есть зависимость на файловую систему.

# Решение

Есть 2 основных способа избавиться от этой зависимости:
1. Создать интерфейс для работы с файловой системой **IFileSystem**, в котором будут все необходимые операции.
2. Создать базовый класс **FileGrouperBase** с абстрактными методами по обращению к файловой системе, а уже каждый наследник будет решать как их реализовать.

Лично мне импонирует первый вариант, так как он является более гибким. Тем более, что нам необходимо больше 1 метода для работы с файловой системой, и удобнее будет иметь для этого отдельную сущность.

## IFileSystem

Давайте опишем **IFileSystem**.

```csharp
public interface IFileSystem
{
    string[] GetFiles(string directory);
    bool DirectoryExists(string directory);
    void CreateDirectory(string directory);
    void MoveFile(string sourceFile, string destinationFile);
}
```

Сразу же пишем реализацию для реальной файловой системы:

```csharp
public class PhysicianFileSystem : IFileSystem
{
    public string[] GetFiles(string directory) 
        => Directory.GetFiles(directory);

    public bool DirectoryExists(string directory) 
        => Directory.Exists(directory);

    public void CreateDirectory(string directory) 
        => Directory.CreateDirectory(directory);

    public void MoveFile(string sourceFile, string destinationFile) 
        => File.Move(sourceFile, destinationFile);
}
```

## Дорабатываем FileGrouper

Изменения будут минимальными, заменяем все статические вызовы **File** и **Directory** на вызовы нового объекта.

```csharp
public class FileGrouper
{
    private readonly string _folderPath;
    // Добавили объект             ↓
    private readonly IFileSystem _fileSystem;

    public FileGrouper(string folderPath, IFileSystem fileSystem)
    {
        _folderPath = folderPath;
        _fileSystem = fileSystem;
    }

    public void ByExtension()
    {
        // Меняем вызов           ↓
        var groupFiles = _fileSystem.GetFiles(_folderPath)
            .Select(fullFileName => new
            {
                FullFileName = fullFileName,
                FileName = Path.GetFileName(fullFileName),
                Extension = Path.GetExtension(fullFileName)
            })
            .Where(fileInfo => !string.IsNullOrEmpty(fileInfo.Extension))
            .ToLookup(arg => arg.Extension);

        foreach (var group in groupFiles)
        {
            var newFolder = Path.Combine(_folderPath, group.Key);
            // Меняем вызовы        ↓
            if (!_fileSystem.DirectoryExists(newFolder))
                _fileSystem.CreateDirectory(newFolder);

            foreach (var file in group)
            {
                var newFile = Path.Combine(newFolder, file.FileName);
                // Меняем вызов   ↓
                _fileSystem.MoveFile(file.FullFileName, newFile);
            }
        }
    }
}
```

Этого достаточно, чтобы можно было заменить работу с файловой системы на оперативную память или любой другой источник.

## Пишем тесты

В качестве библиотеки для тестирования будем использовать [xUnit](https://xunit.net/). Это дело вкуса, можно использовать любую другую библиотеку для unit тестирования.

Для работы тестов, нам понадобится написать отдельную реализацию **IFileSystem**.

```csharp
public sealed class MemoryFileSystem : IFileSystem
{
    private readonly List<string>  _availableFiles;

    public MemoryFileSystem(IEnumerable<string> availableFiles) 
        => _availableFiles = new List<string>(availableFiles);

    public List<(string Source, string Destination)> MoveDetail { get; } 
        = new();
    
    public string[] GetFiles(string directory) 
        => _availableFiles.ToArray();

    public bool DirectoryExists(string directory) => true;

    public void CreateDirectory(string directory){}

    public void MoveFile(string sourceFile, string destinationFile) 
        => MoveDetail.Add((sourceFile, destinationFile));
}
```

Так как алгоритм оперирует путями к файлам, то нам  достаточно сохранить путь файла который хотят переместить и место, куда он будет переносен алгоритмом.

### MemberData

Данные для тестирования выглядят следующим образом

```csharp
new List<object[]>
{
    new object[]
    {
        @"D:\folder",
        new List<string>(),
        new List<(string Source, string Destination)>()
    },
    new object[]
    {
        @"D:\folder2",
        new List<string>
        {
            @"D:\folder2\1.txt",
            @"D:\folder2\2.txt",
            @"D:\folder2\test.mp3",
            @"D:\folder2\test2.png",
        },
        new List<(string Source, string Destination)>
        {
            (@"D:\folder2\1.txt", @"D:\folder2\.txt\1.txt"),
            (@"D:\folder2\2.txt", @"D:\folder2\.txt\2.txt"),
            (@"D:\folder2\test.mp3", @"D:\folder2\.mp3\test.mp3"),
            (@"D:\folder2\test2.png", @"D:\folder2\.png\test2.png")
        }
    },
    new object[]
    {
        @"D:\folder3\tmp",
        new List<string>
        {
            @"D:\folder3\tmp\1.txt",
            @"D:\folder3\tmp\2.txt",
            @"D:\folder3\tmp\3",
            @"D:\folder3\tmp\test.png",
            @"D:\folder3\tmp\test2.png",
        },
        new List<(string Source, string Destination)>
        {
            (@"D:\folder3\tmp\1.txt", @"D:\folder3\tmp\.txt\1.txt"),
            (@"D:\folder3\tmp\2.txt", @"D:\folder3\tmp\.txt\2.txt"),
            (@"D:\folder3\tmp\test.png", @"D:\folder3\tmp\.png\test.png"),
            (@"D:\folder3\tmp\test2.png", @"D:\folder3\tmp\.png\test2.png")
        }
    },
};
```

### Проверяем группировку

```csharp
[Theory]
[MemberData(nameof(TestData))]
public void Should_group_files_with_extension(string path, 
    List<string> availableFiles, 
    List<(string Source, string Destination)> expect)
{
    // Arrange
    var fileSystem = new MemoryFileSystem(availableFiles);
    var group = new FileGrouper(path, fileSystem);
    
    // Act
    group.ByExtension();
    var actual = fileSystem.MoveDetail;

    // Assert
    Assert.Equal(expect, actual);
}
```

# Итог

Тестирование кода помогает помочь в проектировании класса и улучшить дизайн. Порой это приводит к избыточности, но это слихвой покрывется надёжностью кода.

Благодаря тестированию, процесс разработки идёт куда быстрее, так как разработчики меньше боятся поломать существующий функционал, например, очередным рефакторингом или добавлением новой функциональности.

# Ссылки

* [Модульное тестирование (Wiki)](https://ru.wikipedia.org/wiki/%D0%9C%D0%BE%D0%B4%D1%83%D0%BB%D1%8C%D0%BD%D0%BE%D0%B5_%D1%82%D0%B5%D1%81%D1%82%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5)
* [xUnit](https://xunit.net/)
* [Юнит-тестирование для чайников (Хабр)](https://habr.com/ru/post/169381/)
* [Тестирование в .NET (Microsoft)](https://learn.microsoft.com/ru-ru/dotnet/core/testing/)
* [Рекомендации по модульному тестированию для .NET Core и .NET Standard (Microsoft)](https://learn.microsoft.com/ru-ru/dotnet/core/testing/unit-testing-best-practices)
