---
title: "Object Pipes"
date: 2022-05-04T12:50:20+03:00
draft: true
toc: false
categories: ["programming", "csharp", "ru"]
images:
tags:
  - untagged
---

# Заголовок
----

```csharp
public static class Extension
{
    public static IHandler<TIn, TOut2> Pipe<TIn, TOut, TOut2>(this IHandler<TIn, TOut> self, IHandler<TOut, TOut2> secondHandler) 
        => new PipeHandler<TIn, TOut, TOut2>(self, secondHandler);
    
    public static WrapToResultHandler<TIn, TOut> WrapToResult<TIn, TOut>(this IHandler<TIn, TOut> self) => new(self);
    
}

public interface IHandler<in TIn, out TOut>
{
    TOut Handle(TIn input);
}

public sealed class PipeHandler<TIn, TOut, TOut2> : IHandler<TIn,TOut2>
{
    private readonly IHandler<TIn, TOut> _handler1;
    private readonly IHandler<TOut, TOut2> _handler2;

    public PipeHandler(IHandler<TIn, TOut> handler1, IHandler<TOut, TOut2> handler2)
    {
        _handler1 = handler1;
        _handler2 = handler2;
    }


    public TOut2 Handle(TIn input)
    {
        var v1 = _handler1.Handle(input);
        return _handler2.Handle(v1);
    }
}

public sealed class FileReaderHandler : IHandler<string, string>
{
    public string Handle(string path) => File.ReadAllText(path);
}

public sealed class SplitHandler : IHandler<string, string[]>
{
    private readonly string _separator;

    public SplitHandler(string separator) => _separator = separator;

    public string[] Handle(string input) => input.Split(_separator);
}

public sealed class UniqSeqHandler<T> : IHandler<IEnumerable<T>, IEnumerable<T>>
{
    private readonly Func<T, T, bool> _comparer;

    public UniqSeqHandler(Func<T, T, bool> comparer) => _comparer = comparer;

    public IEnumerable<T> Handle(IEnumerable<T> input) => input.Distinct(_comparer);
}

public sealed class JoinStringHandler : IHandler<IEnumerable<string>, string>
{
    private readonly string _separator;

    public JoinStringHandler(string separator) => _separator = separator;

    public string Handle(IEnumerable<string> input) => string.Join(_separator, input);
}

public sealed class WrapToResultHandler<TIn, TOut> : IHandler<TIn, Result<TOut, Exception>>
{
    private readonly IHandler<TIn, TOut> _handler;

    public WrapToResultHandler(IHandler<TIn, TOut> handler) => _handler = handler;

    public Result<TOut, Exception> Handle(TIn input)
    {
        try
        {
            var res = _handler.Handle(input);
            return Result.Success<TOut, Exception>(res);
        }
        catch (Exception e)
        {
            return Result.Failure<TOut, Exception>(e);
        }
    }
}
```

```
1,2,3,1,4,5,3,11,12,11,13
```

```csharp
var result = new FileReaderHandler()
            .Pipe(new SplitHandler(","))
            .Pipe(new UniqSeqHandler<string>(StringComparer.OrdinalIgnoreCase.Equals))
            .Pipe(new JoinStringHandler(Environment.NewLine))
            .WrapToResult()
            .Handle("D:\\test.txt");
```

Итог
----

Ссылки
------

*   [Ссылка]()