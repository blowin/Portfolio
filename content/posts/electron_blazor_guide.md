---
title: "Electron + Blazor = ♥"
date: 2022-05-20T17:15:40+03:00
draft: false
toc: false
categories: ["programming", "csharp", "ru"]
images:
tags:
  - blazor
  - web
  - electron
  - guide
---

Опишу, как можно собрать blazor проект с использованием electron. Это можно использовать, для любого ASP проекта.

# Шаги

1. Устанавливаем electronNet.cli (один раз)
  
```csharp
dotnet tool install --global electronNet.cli
```

1. Установить nuget пакет [ElectronNET.API](https://www.nuget.org/packages/ElectronNET.API/)
2. Добавить в Startup создание окна

```csharp
if (HybridSupport.IsElectronActive)
{
	Task.Run(async () =>
	{
		await Electron.WindowManager.CreateBrowserViewAsync();
		await Electron.WindowManager.CreateWindowAsync(new BrowserWindowOptions
		{
			MinWidth = 700,
			MinHeight = 500,
			Center = true
		});
	});   
}
```
4. Добавить UseElectron в Program.cs
```csharp
webBuilder.UseElectron(args).UseStartup<Startup>()
```

# Сборка

Переходим в папку с веб приложением (csproj) и запускаем

```csharp
electronize init
```
или
```csharp
electronize build /target win
```

Подробнее можно прочитать [тут](https://github.com/ElectronNET/Electron.NET/#-build)

# Запуск приложения 

```csharp
electronize start
```

# Ссылки
----
  
*   [ElectronNET.API (Nuget)](https://www.nuget.org/packages/ElectronNET.API/)