---
title: "Анатомия декоратора"
date: 2022-05-05T20:30:50+03:00
draft: false
toc: false
categories: ["programming", "csharp", "ru"]
images:
tags:
  - decorator
  - proxy
  - composite
  - logging
  - design
  - pattern
---

Как известно, декоратор является одним из структурных паттернов проектирования. Его удобно комбинировать с другими паттернами для достижения гибкой и расширяемой системы, без изменения существующего кода.

## Разбираемся на примере логирования

Допустим, перед нами появилась задача написать логирование для нашего приложения. Что же, давайте реализуем: 

```csharp
public enum LogLevel
{
    Debug,
    Info,
    Warn,
    Error,
    Fatal
}

public interface ILogger : IDisposable
{
    void Log(LogLevel level, string message, params object[] args);
}

public class ConsoleLogger : ILogger
{
    public void Log(LogLevel level, string message, params object[] args)
    {
        Console.Write("[{0}]: ", level);
        Console.WriteLine(message, args);
    }

    public void Dispose(){}
}
```

Что делать, если нам необходимо выключить логирование на определенных уровнях?

Первое, что может прийти в голову:

> * Добавить поле в ConsoleLogger и перед записью проверять, нужно ли логировать сообщение.
> * Сделать базовый класс, который будет делать эту проверку (лучше, но не так гибко, теперь нам обязательно нужно наследоваться от базового класса).

Эти решения будут работать, но вряд ли будут удобны в дальнейшей перспективе. 

## Используем декоратор

```csharp
public class FilterLogLevelLogger : ILogger
{
    private readonly ILogger _logger;
    private readonly LogLevel _minimumLogLevel;

    public FilterLogLevelLogger(ILogger logger, LogLevel minimumLogLevel)
    {
        _logger = logger;
        _minimumLogLevel = minimumLogLevel;
    }

    public void Log(LogLevel level, string message, params object[] args)
    {
        if (level < _minimumLogLevel)
            return;
        _logger.Log(level, message, args);
    }
    
    public void Dispose() => _logger.Dispose();
}
```

Комбинируем:

```csharp
using var consoleLogger = new FilterLogLevelLogger(new ConsoleLogger(), LogLevel.Info);
consoleLogger.Log(LogLevel.Debug, "Test");
consoleLogger.Log(LogLevel.Warn, "Test"); // выводит только это сообщение
```

# Composite + Decorator

Наше приложение усложняется и теперь нам мало простой записи в консоль, у нас появилась необходимость писать логи в файл. 

Это сделать просто:

```csharp
public class FileLogger : ILogger
{
    private readonly StreamWriter _writer;

    public FileLogger(string path) => _writer = File.AppendText(path);

    public void Log(LogLevel level, string message, params object[] args)
    {
        _writer.Write("[{0}]: ", level);
        _writer.WriteLine(message, args);
    }

    public void Dispose() => _writer.Dispose();
}
```

А что если мы хотим логировать и в файл, и в консоль? 

## Используем композит

```csharp
public class CompositeLogger : ILogger
{
    private readonly ICollection<ILogger> _loggers;

    public CompositeLogger(ICollection<ILogger> loggers) => _loggers = loggers;

    public void Log(LogLevel level, string message, params object[] args)
    {
        foreach (var logger in _loggers)
            logger.Log(level, message, args);
    }
    
    public void Dispose()
    {
        foreach (var logger in _loggers)
            logger.Dispose();
    }
}
```

Теперь мы можем использовать 2 логгера одновременно:

```csharp
var loggers = new ILogger[]
{
    new FileLogger("test.txt"),
    new ConsoleLogger()
};
using var consoleLogger = new CompositeLogger(loggers);
consoleLogger.Log(LogLevel.Debug, "Test");
consoleLogger.Log(LogLevel.Warn, "Test");
```

Благодаря использованию Composite мы также можем включить определенный уровень логирования для конкретного логгера или же для всех одновременно.

## Определенный уровень для всех логгеров одновременно

```csharp
var loggers = new ILogger[]
{
    new FileLogger("test.txt"),
    new ConsoleLogger()
};
//                            Магия тут ↓
using var consoleLogger = new FilterLogLevelLogger(new CompositeLogger(loggers), LogLevel.Info);
consoleLogger.Log(LogLevel.Info, "Test");
consoleLogger.Log(LogLevel.Warn, "Test");
```

Как это работает: 

Вызываем Log, с определенным уровнем логирования у FilterLogLevelLogger, который уже начинает проверку на уровень логирования и если он должен быть залогирован, то он попадает ниже, в нашем случае, это ComposeLogger, который вызовет Log, на каждом из внутренних логгеров ConsoleLogger и FileLogger.

В текущем примере на консоль и в файл попадёт 2 сообщения. Так как они все подходят по уровню логирования.

## Определенный уровень логирования для каждого логгера отдельно

```csharp
var loggers = new ILogger[]
{
    // Магия тут ↓
    new FilterLogLevelLogger(new FileLogger("test.txt"), LogLevel.Warn),
    // Магия тут ↓
    new FilterLogLevelLogger(new ConsoleLogger(), LogLevel.Info)
};
using var consoleLogger = new CompositeLogger(loggers);
consoleLogger.Log(LogLevel.Info, "Test");
consoleLogger.Log(LogLevel.Warn, "Test");
```

Как это работает: 

Вызываем Log, с определенным уровнем логирования, на ComposeLogger, он в свою очередь берет каждый из внутренних логгеров и перенаправляет запрос к каждому из внутренних:

* new FilterLogLevelLogger(new FileLogger("test.txt"), LogLevel.Warn)
* new FilterLogLevelLogger(new ConsoleLogger(), LogLevel.Info)

Который уже начинает проверку на уровень логирования и если он должен быть залогирован, то он попадает ниже, т.е уже к конкретной реализации ConsoleLogger или FileLogger. 

В текущем примере на консоль попадёт 2 сообщения, так как они подходят по уровню, а в файл будет записано только одно сообщение, с уровнем Warn.

# Proxy + Decorator

Представим, появилось требование, что мы не должны открывать файл логирования на старте приложения. Допустим, что это занимает много ресурсов и нам нужно отложить это действие, настолько, насколько это возможно. 

## Сделаем новый декоратор

```csharp
public class LazyLogger : ILogger
{
    private readonly Lazy<ILogger> _lazyLogger;

    public LazyLogger(Func<ILogger> loggerFactory) => _lazyLogger = new Lazy<ILogger>(loggerFactory);

    public void Log(LogLevel level, string message, params object[] args)
    {
        _lazyLogger.Value.Log(level, message, args);
    }
    
    public void Dispose()
    {
        if(_lazyLogger.IsValueCreated)
            _lazyLogger.Value.Dispose();
    }
}
```

Использование:

```csharp
using var lazyLogger = new LazyLogger(() => new FileLogger("test.txt")); 
consoleLogger.Log(LogLevel.Warn, "Test");
```

Теперь, если логгер не будет нужен, то файл не будет открыт/создан во время работы приложения и это произойдёт только по необходимости

# Proxy + Composite + Decorator

Используем всё вместе:

```csharp
var loggers = new ILogger[]
{
    new FilterLogLevelLogger(
        new LazyLogger(() => new FileLogger("test.txt")), 
        LogLevel.Warn
    ),
    new FilterLogLevelLogger(new ConsoleLogger(), LogLevel.Info)
};
using var consoleLogger = new CompositeLogger(loggers);
consoleLogger.Log(LogLevel.Info, "Test");
```

В данном примере сообщение попадёт только в консоль и файл даже не будет создан, так как в него могут попасть сообщения с уровнем Warn и выше.

# Итог

Не стоит боятся комбинировать разные паттерны если правильно их применять, можно сделать гибкую и расширяемую систему. 

Конечно, так можно всё испортить, так что не стоит переусердствовать и всегда стоит думать, *а точно ли оно мне тут нужно?*

# Ссылки

* [Decorator](https://refactoring.guru/design-patterns/decorator)
* [Composite](https://refactoring.guru/design-patterns/composite)
* [Proxy](https://refactoring.guru/design-patterns/proxy)
* [Logging](https://en.wikipedia.org/wiki/Logging_(software)#:~:text=In%20computing%2C%20a%20log%20file,to%20a%20single%20log%20file.)