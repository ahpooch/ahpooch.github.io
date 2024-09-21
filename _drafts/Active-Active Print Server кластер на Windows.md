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
  path: /images/active-active-cluster_blog_post.png
  #lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: active to active print server cluster scheme
---
В этом посте я хочу рассказать про способ настройки Active/Active кластера из серверов печати на Windows. Оговорюсь сразу что данный кластер не подразумевает автоматического переключения, хотя возможно в дальнейшем я продумаю и опишу способ этого достичь. 
Официально Microsoft не поддерживает такую конфигурацию, предлагая только Active/Passive реализацию, что не подходит для обслуживания пользователей в разных офисах, либо потребует создания двух кластеров Active/Passive, по одному на каждый офис. Этот вариант выглядит неоптимальным по расходу серверных лицензий.

## Чего хотелось добиться
Содержать в каждом офисе лишь по одному активному серверу печати и в случае необходимости обслуживания или возникшей проблемы иметь возможность переключить поток задач печати на другой активный сервер кластера в соседнем офисе. Для таких переключений набор принтеров, установленных на серверах печати должен быть идентичным. То есть на каждом сервере будут установлены абсолютно все подключаемые пользователям принтеры. Если в Вашем случае в каждом из офисов по сотне устройств печати, то возможно такая реализация для Вас не подходит. Либо можно сгруппировать офисные сервера так, чтобы общее суммарное количество принтеров на них было разумным. Я буду описывать сценарий с одним кластером из двух офисных серверов, но ничего не мешает созданию такого кластера из трех серверов или разделению общего количества офисов на несколько кластеров, члены которого могут подменять других его участников.
Принтеры будут маппиться пользователям стандартным методом, через групповые политики по принципу ближайшего к пользователю сервера. Устройства печати, находящиеся в офисе "А" - пользователям офиса "А", принтеры из офиса "Б" - пользователям работающим из офиса "Б". Конечно при работе через "соседский" сервер в "ЧП" режиме, задачи печати будут доставляться и выполняться дольше обычного. Однако для сценария Dizaster Recovery и возможности в меньшей спешке восстановить работоспособность проблемного сервера такой временный режим работы будет отличным вариантом.

![Desktop View](/images/active-active-print-server-cluster.png){: width="2010" height="1022" .w-100 .normal}

Для того чтобы достигнуть такой конфигурации нам необходимо, чтобы принтеры сервера печати подключались пользователям не по имени сервера `\\servename-in-office-A.domain.com\my-lovely-printer`, а по некоторому CNAME в DNS. Например `\\PRINT-OFFICE-A.domain.com\my-lovely-printer-in-office-A`. Так как на серверах в кластере установлен абсолютно одинаковый набор серверов - в необходимый момент мы можем переключить всех пользоватьелей работающих с принтерами опубликованными на `\\PRINT-OFFICE-A.domain.com` на `\\PRINT-OFFICE-B.domain.com`. Для этого мы просто изменим CNAME `PRINT-OFFICE-A.domain.com`, чтобы он стал указывать не на `servername-in-office-A.domain.com`, а на `servername-in-office-B.domain.com`.

## Настройка

### Установка роли сервера печати
Для каждого из серверов установить роль сервера печати и оснастку управления
```powershell
#Проверка и установка роли Print Server
if ($null -eq (Get-WindowsFeature -Name Print-Server | ?{$_.InstallState -eq "Installed"})){
    Install-WindowsFeature -Name Print-Server
}
#Проверка и установка оснаски управления печатью
if ($null -eq (Get-WindowsFeature -Name RSAT-Print-Services | ?{$_.InstallState -eq "Installed"})){
    Install-WindowsFeature -Name RSAT-Print-Services
}

```

### Активация работы сервера печати через CNAME
Для того чтобы с серверами печати можно было работать через CNAME[^footnote1], необходимо выполнить на каждом из них изменение в реестре.
```powershell
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Print" -Name "DnsOnWire" -PropertyType DWord -Value 1 -Force
```

### Сбор информации об активах печати
Первично нужно будет потратить время на детальный анализ всех принтеров которые подключены пользователям. В моем случае оказалось что все кейсы можно покрыть довольно мылым количество драйверов. Их необходимо расположить на сетевой шаре, доступной серверам. 
Так же эта шара должна быть доступна и пользователям. Это требование исходит из последствий проблемы Print Naghtmare, поменявшей привычный деплой принтеров пользователям через GPO, ведь теперь установка драйверов принтеров с помощью групповых политик невозможна с сервера печати без статуса локального администратора. Во всяком случае без понижения уровня безопасности. Однако я расскажу как доставить драйвера для печати пользователяи безопасным способом заранее, ещё до подключения принтеров. Это позволит использовать групповые политики маппинга так же удобно как и до появления угрозы Print Naghtmare. 

### Расположение драйверов в общедоступной папке
Если возможно, то создаём папку в DFS, либо просто публикуем на одном из серверов. 

# Скрипт установки принтеров
Для того чтобы набор установленных принтеров был одинаков для всех челнов кластера, мы будет устанавливать их используя Powershell скрипт.
В скрипте необходимо будет описать драйверы для каждого из принтеров указав имя (name), путь к INF файлу (netpath) и местонахождение INF файла после установки драйвера (storePattern). К сожаления часть пути является динамической, поэтому в StorePattern мы используем * для определения папки с неизвестным заранее именем. 
Этот скрипт необходимо подготовить под себя описав блоки с добавление нужных драйверов, после чего данным скриптом можно будет установить, и подготовить принтеры к использованию (опубликовать) на любом сервере.

```powershell
#Скрипт установки всех принтеров на новом сервере печати
#DuplexMode = @("OneSided","TwoSidedLongEdge","TwoSidedShortEdge")
#PaperSize = see https://docs.microsoft.com/en-us/powershell/module/printmanagement/set-printconfiguration?view=windosserver2019-ps#:~:text=%5B%5D-,Description,Collate
[CmdletBinding()]
param (
   [Parameter(Mandatory=$true)]
   [string]$CSVFile
)

try {
$PrintersCSVPath = Resolve-Path $CSVFile
$PrintersCSV = Get-Content -Path $PrintersCSVPath
} catch {
    throw "Can't obtain csv file!"
}

$Printers = $PrintersCSV | ConvertFrom-Csv -Delimiter ","
$drivers = @()
$drivers += [pscustomobject]@{
    name = 'Canon Generic Plus PCL6'
    netPath = '\\domain.com\fs\PrintDriversStore\CanonGenericPlusPCL6\x64\Driver\CNP60MA64.INF'
    storePattern = 'C:\Windows\System32\DriverStore\FileRepository\*\CNP60MA64.INF'
    storePath = ''
}
$drivers += [pscustomobject]@{
    name = 'HP LaserJet Pro MFP M426f-M427f PCL 6'
    netpath = '\\domain.com\fs\PrintDriversStore\HP_LJ_Pro_MFP_M426f\hpma5a2a_x64.inf'
    storePattern = 'C:\Windows\System32\DriverStore\FileRepository\*\hpma5a2a_x64.inf'
    storePath = ''
}
$drivers += [pscustomobject]@{
    name = 'HP Universal Printing PCL 6'
    netPath = '\\domain.com\fs\PrintDriversStore\HPUniversalPrintingPCL6\hpcu255u.inf'
    storePattern = 'C:\Windows\System32\DriverStore\FileRepository\*\hpcu255u.inf'
    storePath = ''
}
$drivers += [pscustomobject]@{
    name = 'Pantum CP1100 Series PCL6'
    netPath = '\\domain.com\fs\PrintDriversStore\Pantum CP1100 Series Windows Driver V2.0.62\PRINTER\CP1100PR.inf'
    storePattern = 'C:\Windows\System32\DriverStore\FileRepository\*\CP1100PR.inf'
    storePath = ''
}
$drivers += [pscustomobject]@{
    name = 'Pantum CM1100DN Series PCL6'
    netPath = '\\domain.com\fs\PrintDriversStore\Pantum CM1100 Series Windows Driver V2.0.38\PRINTER\CM1100PR.inf'
    storePattern = 'C:\Windows\System32\DriverStore\FileRepository\*\CM1100PR.inf'
    storePath = ''
}

<#
EXAMPLE
$drivers += [pscustomobject]@{
    name = 'Canon Generic Plus PCL6' # AS IN Get-PrinterDriver
    netPath = '\\domain.com\fs\PrintDriversStore\CanonGenericPlusPCL6\x64\Driver\CNP60MA64.INF'
    #'C:\Windows\System32\DriverStore\FileRepository\cnp60ma64.inf_amd64_f2f35da4a1de4401\CNP60MA64.INF'
    storePattern = 'C:\Windows\System32\DriverStore\FileRepository\*\CNP60MA64.INF'
    storePath = '' #(Get-ChildItem 'C:\Windows\System32\DriverStore\FileRepository\*\CNP60MA64.INF').FullName
}
#>

#pnputil.exe will add the driver to the store at C:\Windows\System32\DriverStore
foreach ($driver in $drivers){
    if (!((Get-PrinterDriver).Name -contains $driver.name)){
        pnputil.exe -a $driver.netPath -i
        $driver.storePath = Get-ChildItem $driver.storePattern
        Add-PrinterDriver -Name $driver.name -InfPath $($driver.storePath)
    }
}

#Создание портов и принтеров
foreach ($Printer in $Printers){
    if (!((Get-PrinterPort -Name $Printer.IP -ErrorAction SilentlyContinue).PrinterHostAddress -eq $Printer.IP)){
        Add-PrinterPort -Name $Printer.IP -PrinterHostAddress $Printer.IP
    }
    if(!(Get-Printer $Printer.Name -errorAction SilentlyContinue)){
        Add-Printer -DriverName $Printer.DriverFromInfFile -Name $Printer.Name -Comment $Printer.ModelFromWebUI -PortName $Printer.IP
    }
    Set-PrintConfiguration -PrinterName $Printer.Name -DuplexingMode $Printer.DuplexMode -PaperSize $Printer.PaperSize
    Set-Printer -Name $Printer.Name -Shared $true -ShareName $Printer.Name
}
```
{: file='.\Set-OfficePrinters.ps1'}

### Подготовка CSV файла с конфигурацией принтеров
Далее нам необходимо подготовить CSV файл с принтерами, которой мы будет передавать на вход скрипта устанавливающего принтеры.
В моем случа CSV файл с принтерами выглядит примерно так:
```CSV
IP,Name,ModelFromWebUI,DriverFromInfFile,DuplexMode,PaperSize
192.168.1.1,Office1-Main,Canon iR-ADV C3520 III,Canon Generic Plus PCL6,OneSided,A4
192.168.1.2,Office1-BUH,Canon iR-ADV C3520 III,Canon Generic Plus PCL6,OneSided,A4
192.168.1.3,Office1-HR,Canon imageRUNNER1435,Canon Generic Plus PCL6,OneSided,A4
192.168.1.4,Office1-MANAGERS,Canon iR-ADV C3720,Canon Generic Plus PCL6,OneSided,A4
192.168.1.5,Office1-DESIGNERS1,Canon iR-ADV C3720,Canon Generic Plus PCL6,OneSided,A4
102.168.2.1,Office2-DESIGNERS2,HP LaserJet MFP M426fdn,HP LaserJet Pro MFP M426f-M427f PCL 6,OneSided,A4
102.168.2.2,Office2-ADMINS,HP LaserJet MFP E52645,HP Universal Printing PCL 6,OneSided,A4
102.168.2.3,Office2-HR,HP LaserJet MFP E52645,HP Universal Printing PCL 6,OneSided,A4
102.168.2.4,Office2-Main,Canon iR-ADV C3520,Canon Generic Plus PCL6,OneSided,A4
102.168.3.1,Office2.1-HR,Pantum CM1100DN,Pantum CM1100DN Series PCL6,OneSided,A4
102.168.3.2,Office2.1-Main,Canon iR-ADV C3520,Canon Generic Plus PCL6,OneSided,A4
```
{: file='printers.csv'}

### Запуск скрипта установки принтеров
```powershell
& Set-Office Printer.ps1 -CSVFile printer.csv
```

`C:\Example.txt`{: .filepath}

### Групповые политики


### Продолжение следует
To be continue...

## Ссылки
[^footnote1]: [Error message when you try to connect to a printer by using an alias (CNAME) resource record: Windows couldn't connect to the printer](https://docs.microsoft.com/en-us/troubleshoot/windows-server/printing/windows-couldnt-connect-printer)
