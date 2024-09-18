---
layout: post
title: "Active-Active Print Server кластер на Windows"
description: "Способ настройки Active/Active кластера из серверов печати на Windows"
author: ahpooch
date: 2024-09-17 20:00 +0300
categories: [Windows, PrintServer]
tags: [Guide, Inspiration]
pin: true
math: true
mermaid: true
image:
  path: /images/active-active-print-server-cluster.png
  #lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: active to active print server cluster scheme
---
В этом посте я хочу рассказать про способ настройки Active/Active кластера из серверов печати на Windows.
Официально Microsoft не поддерживает такую конфигурацию, предлагая только Active/Passive реализацию, что не подходит для обслуживания пользователей в разных офисах, либо потребует создания двух кластеров Active/Passive, по одному на каждый офис. Этот вариант не выглядит оптимальным по расходу серверных лицензий.

## Чего я хотел добиться
Я хотел иметь возможность держать в каждом офисе по одному серверу печати, на котором установлены все принтеры компании. С помощью групповых политик необходимые принтеры должны быть установлены соответствующим группам пользователей. Принтеры, находящиеся в офисе "А" - пользователям офиса "А", принтеры из офиса "Б" - пользователям работающим из офиса "Б". При необходимости иметь возможность переключить пользователей на печать через принт-сервер в другом офисе. Конечно при работе через "соседский" сервер, задачи печати будут доставляться и выполняться дольше обычного. Однако для сценария Dizaster Recovery и возможности в меньшей спешке восстановить работоспособность проблемного сервера такой временный режим работы будет отличным вариантом.

![Desktop View](/images/active-active-print-server-cluster.png){: width="2010" height="1022" .w-100 .normal}

Для того чтобы достигнуть такой конфигурации нам необходимо, чтобы принтеры сервера печати подключались пользователям не по имени сервера `\\servename-in-office-A.domain.com\my-lovely-printer`, а по некоторому CNAME в DNS. Например `\\PRINT-OFFICE-A.domain.com\my-lovely-printer-in-office-A`. Так как на обоих серверах установлен абсолютно одинаковый набор серверов - в необходимый момент мы можем переключить всех пользоватьелей работающих с принтерами опубликованными на `\\PRINT-OFFICE-A.domain.com` на `\\PRINT-OFFICE-B.domain.com`. Для этого мы просто изменим CNAME `PRINT-OFFICE-A.domain.com`, чтобы она стала указывать не на `servername-in-office-A.domain.com`, а на `servername-in-office-B.domain.com`.

## Настройка

### Активация работы сервера печати через CNAME
Для того чтобы с серверами печати можно было работать через CNAME, необходимо выполнить на каждом из них изменение в реестре[^footnote1].
```powershell
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Print" -Name "DnsOnWire" -PropertyType DWord -Value 1 -Force
```

### Продолжение следует
To be continue...

## Ссылки
[^footnote1]: [Error message when you try to connect to a printer by using an alias (CNAME) resource record: Windows couldn't connect to the printer](https://docs.microsoft.com/en-us/troubleshoot/windows-server/printing/windows-couldnt-connect-printer)
