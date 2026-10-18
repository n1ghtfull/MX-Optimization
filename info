# MX-Optimizer: руководство по NVIDIA и продвинутым твикам

Принцип проекта: **изменил — измерил — откатил**. Поэтому каждый пункт ниже помечен уровнем надёжности, а в проект попали только те команды, которые можно проверить и откатить.

| Метка | Значение |
|---|---|
| **[A]** | Механизм задокументирован Microsoft или NVIDIA |
| **[B]** | Известен в сообществе; эффект зависит от железа, драйвера и версии Windows. Требует замера |
| **[C]** | Реестрового механизма нет, параметр устарел или вреден. В проект не добавляется |

Формат разделов: сначала чистый листинг без комментариев (готов для вставки в `.bat`), затем пояснения строго под ним.

---

# Часть 1. NVIDIA

## 1.1 Главное ограничение: настройки 3D живут не в реестре

Почти все параметры «Панели управления NVIDIA → Управление параметрами 3D» хранятся в профильной базе драйвера (DRS, файл `nvdrsdb0.bin`). Они изменяются через NVAPI, Панель управления NVIDIA или NVIDIA Profile Inspector. Документированных ключей реестра для них не существует. Придуманные «ключи» в таких руководствах обычно ничего не делают.

Что это значит для вашего списка:

| Запрошено | Статус | Что можно сделать |
|---|---|---|
| Отключение энергосберегающих P-states | **[B]** | Ключи `PerfLevelSrc`, PowerMizer, `DisableDynamicPstate` (листинг ниже). Альтернатива **[A]**: `nvidia-smi -lgc` |
| «Предпочитать максимальную производительность» | **[C]** через реестр | Это параметр DRS. Ставится в Панели NVIDIA или Profile Inspector |
| Снятие ограничений интерфейса, «скрытые» 3D-параметры | **[C]** | Реестрового переключателя нет. Скрытые параметры показывает NVIDIA Profile Inspector |
| Low Latency Mode = Ultra | **[C]** | DRS. В играх с NVIDIA Reflex режим «On» плюс Reflex обычно лучше Ultra |
| V-Sync off, тройная буферизация | **[C]** | DRS. Тройная буферизация в драйвере работает только в OpenGL |
| Фильтрация текстур High Performance, оптимизации анизотропной и трилинейной фильтрации | **[C]** | DRS |
| Threaded Optimization | **[C]** | DRS (по умолчанию «Auto» уже включает его где нужно) |
| Режим питания шины | **[C]** | DRS и параметры PCIe в схеме питания (в проекте уже отключено через `ASPMOptOut` и `powercfg`) |
| Shader Cache «Unlimited» | **[C]** | DRS (Панель NVIDIA → «Размер кэша шейдеров»). Реестрового ключа нет |
| Телеметрия NVIDIA | **[A/B]** | Уже в `Preset_8` (службы, задачи, `OptInOrOutPreference`, `FTS`) |
| Отключение HDCP | **[B]** | `RMHdcpKeyglobZero`. Вывод о «разгрузке GPU» неверен, см. 1.4 |

Поэтому в `Preset_8` не добавлены пункты 2 и 3 вашего запроса: писать неработающие ключи нельзя, это ломает доверие к проекту. Остальное добавлено в `05_Experimental_Optional`, а не в основной пресет (причины в 1.3).

## 1.2 Листинг: поиск видеокарты NVIDIA и применение ключей

Скрипт не использует подветку `Video\0000` напрямую: она может оказаться встроенной графикой. Вместо этого он перебирает подветки класса видеоадаптеров и выбирает те, у которых производитель NVIDIA (`ven_10de`).

```bat
powershell -NoProfile -ExecutionPolicy Bypass -Command "Get-ChildItem 'HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4d36e968-e325-11ce-bfc1-08002be10318}' -ErrorAction SilentlyContinue | Where-Object { $_.PSChildName -match '^[0-9]{4}$' } | ForEach-Object { $pr = Get-ItemProperty $_.PSPath -ErrorAction SilentlyContinue; if ([string]$pr.MatchingDeviceId -match 'ven_10de') { New-ItemProperty -Path $_.PSPath -Name PerfLevelSrc -Value 8738 -PropertyType DWord -Force | Out-Null; New-ItemProperty -Path $_.PSPath -Name PowerMizerEnable -Value 1 -PropertyType DWord -Force | Out-Null; New-ItemProperty -Path $_.PSPath -Name PowerMizerLevel -Value 1 -PropertyType DWord -Force | Out-Null; New-ItemProperty -Path $_.PSPath -Name PowerMizerLevelAC -Value 1 -PropertyType DWord -Force | Out-Null; New-ItemProperty -Path $_.PSPath -Name DisableDynamicPstate -Value 1 -PropertyType DWord -Force | Out-Null } }"
powershell -NoProfile -ExecutionPolicy Bypass -Command "Get-ChildItem 'HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4d36e968-e325-11ce-bfc1-08002be10318}' -ErrorAction SilentlyContinue | Where-Object { $_.PSChildName -match '^[0-9]{4}$' } | ForEach-Object { $pr = Get-ItemProperty $_.PSPath -ErrorAction SilentlyContinue; if ([string]$pr.MatchingDeviceId -match 'ven_10de') { New-ItemProperty -Path $_.PSPath -Name RMHdcpKeyglobZero -Value 1 -PropertyType DWord -Force | Out-Null } }"
reg add "HKLM\SOFTWARE\NVIDIA Corporation\NvControlPanel2\Client" /v OptInOrOutPreference /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\NVIDIA Corporation\Global\FTS" /v EnableRID44231 /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\NVIDIA Corporation\Global\FTS" /v EnableRID64640 /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\NVIDIA Corporation\Global\FTS" /v EnableRID66610 /t REG_DWORD /d 0 /f
sc config NvTelemetryContainer start= disabled
powershell -NoProfile -ExecutionPolicy Bypass -Command "Get-ScheduledTask -TaskName 'NvTm*','NvDriverUpdateCheck*','NvProfileUpdater*','NvNodeLauncher*' -ErrorAction SilentlyContinue | Disable-ScheduledTask | Out-Null"
```

Пояснения к листингу:

1. **Перебор подветок.** Класс `{4d36e968-e325-11ce-bfc1-08002be10318}` содержит `0000`, `0001` и так далее, по одной на каждый видеоадаптер. Условие `ven_10de` в `MatchingDeviceId` отбирает только NVIDIA. `Video\{GUID}\0000` является ссылкой на те же данные, но перебор по классу надёжнее.
2. **`PerfLevelSrc = 0x2222` (8738)** **[B]**. Старый ключ, фиксирующий выбор уровня производительности. На архитектурах Turing и новее и на свежих драйверах часто игнорируется.
3. **`PowerMizerEnable`, `PowerMizerLevel`, `PowerMizerLevelAC`** **[B]**. Унаследованные от эпохи PowerMizer параметры; на современных GPU эффект не гарантирован.
4. **`DisableDynamicPstate = 1`** **[B]**. Запрещает динамическое переключение P-state. Побочный эффект: карта держит высокие частоты даже на рабочем столе (нагрев, шум, энергопотребление). Поэтому применяется только на настольных ПК и только по явному выбору.
5. **`RMHdcpKeyglobZero = 1`** **[B]**. Обнуляет глобальные ключи HDCP. См. 1.4.
6. **`OptInOrOutPreference`, `FTS\EnableRID*`** **[B]**. Общепринятые ключи отказа от телеметрии NVIDIA.
7. **`NvTelemetryContainer`** **[A]**. Служба телеметрии. Службы `NvContainerLocalSystem` и `NvContainerNetworkService` **отключать нельзя**: они нужны Панели управления и драйверу.
8. **Задачи `NvTm*`, `NvDriverUpdateCheck*`, `NvProfileUpdater*`, `NvNodeLauncher*`**. Отчёты, проверка обновлений и компоненты GeForce Experience. Отключение безопасно.

## 1.3 Почему блок не добавлен в основной Preset_8

- Эффект ключей группы **[B]** зависит от поколения GPU и драйвера, он не воспроизводим на всех системах.
- Принудительные максимальные частоты увеличивают нагрев и энергопотребление постоянно, а не только в игре.
- README проекта обещает не включать в рекомендуемый пакет твики без подтверждённого эффекта.

Поэтому пункты вынесены в `05_Experimental_Optional\Experimental_Menu.bat` (пункты 1 и 2) с откатом. Телеметрия NVIDIA осталась в `Preset_8`, потому что её отключение безопасно и не зависит от железа.

## 1.4 Про HDCP и «разгрузку GPU»

Шифрование HDCP выполняется выделенным блоком дисплейного движка видеокарты, а не шейдерными ядрами. Заметного влияния на FPS и тайминги кадров оно не имеет. Отключение HDCP **ломает воспроизведение защищённого контента** (Netflix и другие сервисы в высоком качестве, Blu-ray) и может приводить к чёрному экрану в стриминговых приложениях. Поэтому в проекте это опциональный пункт с предупреждением, а не часть оптимизации.

## 1.5 Что делать с параметрами, которых нет в реестре

**Вариант 1. Вручную в Панели управления NVIDIA** (одноразово). Значения для минимальной задержки:

| Параметр | Значение | Примечание |
|---|---|---|
| Режим низкой задержки | Вкл. | «Ультра» полезен без NVIDIA Reflex; при упоре в GPU может снижать FPS. В играх с Reflex включайте Reflex в игре |
| Режим управления электропитанием | Предпочтителен режим максимальной производительности | Повышает нагрев в простое при некоторых драйверах |
| Вертикальный синхроимпульс | Выкл. | При разрывах используйте G-SYNC или FreeSync и ограничение FPS |
| Качество фильтрации текстур | Высокая производительность | Видимая разница минимальна |
| Потоковая оптимизация | Авто | Принудительное «Вкл.» нужно редко |
| Размер кэша шейдеров | Без ограничений (или драйвер по умолчанию) | Снижает подлагивания при повторной загрузке сцен |

**Вариант 2. NVIDIA Profile Inspector** (сторонняя утилита, в проект не включается, так как в нём нет `.exe`). Настройте профиль один раз, сохраните его через **Export** в файл `.nip` и импортируйте на других ПК. Тестировать и распространять такой файл можно только после проверки на реальных системах; ключи командной строки зависят от версии утилиты.

**Вариант 3. Фиксация частот штатной утилитой NVIDIA** **[A]** (Turing и новее, нужны права администратора, сбрасывается после перезагрузки):

```bat
nvidia-smi --query-gpu=clocks.max.gr --format=csv,noheader,nounits
nvidia-smi -lgc 2100,2100
nvidia-smi -rgc
```

1. Первая команда выводит максимальную частоту ядра (МГц). Подставьте это значение вместо `2100`.
2. `-lgc мин,макс` фиксирует диапазон частот ядра. Работает не на всех картах и драйверах.
3. `-rgc` сбрасывает фиксацию. Это легальная замена `PerfLevelSrc`, но она не сохраняется после перезагрузки.

## 1.6 Как проверить результат

```bat
nvidia-smi --query-gpu=pstate,clocks.gr,clocks.mem,power.draw,temperature.gpu --format=csv -l 1
```

Показывает P-state, частоты, потребление и температуру раз в секунду. Для кадровых метрик (1% и 0.1% low) используйте PresentMon или CapFrameX во время игры. Рекомендация: 3 прогона одной сцены до и 3 после изменения, сравнивать медианы.

---

# Часть 2. Продвинутые твики системы

## 2.1 Ввод: мышь, клавиатура, USB

```bat
reg add "HKLM\SYSTEM\CurrentControlSet\Services\mouclass\Parameters" /v MouseDataQueueSize /t REG_DWORD /d 20 /f
reg add "HKLM\SYSTEM\CurrentControlSet\Services\kbdclass\Parameters" /v KeyboardDataQueueSize /t REG_DWORD /d 20 /f
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\csrss.exe\PerfOptions" /v CpuPriorityClass /t REG_DWORD /d 3 /f
reg add "HKLM\SYSTEM\CurrentControlSet\Services\USB" /v DisableSelectiveSuspend /t REG_DWORD /d 1 /f
reg add "HKLM\SYSTEM\CurrentControlSet\Control\PriorityControl" /v Win32PrioritySeparation /t REG_DWORD /d 38 /f
```

1. `MouseDataQueueSize` и `KeyboardDataQueueSize` **[A: параметры существуют; эффект B/C]**. Размер очереди событий драйвера, по умолчанию 100. Уменьшение до 20 не доказано как улучшение; при очень высокой частоте опроса может приводить к потере событий. Пункт 3 экспериментального меню.
2. `csrss.exe\PerfOptions\CpuPriorityClass` **[B]**. Приоритет подсистемы, обрабатывающей часть ввода. **Значение 3 = High, значение 4 = Realtime**: во многих руководствах встречается ошибочная рекомендация «4», при ней возможны зависания системы. Пункт 4 экспериментального меню.
3. `DisableSelectiveSuspend` **[A]**. Отключает выборочную приостановку USB. Уже в `Preset_6`.
4. `Win32PrioritySeparation = 38` (0x26) **[B]**. Уже в `Preset_3`.

Чего сделать нельзя:

- **Частоту опроса HID-устройств в реестре Windows изменить нельзя** **[C]**. Она определяется прошивкой устройства (параметр bInterval), настраивается фирменной утилитой производителя мыши. Ключи вида «HID polling» в сети не имеют эффекта.
- **«Снижение приоритета опроса остальных контроллеров»** **[C]**: механизма в реестре нет. Единственный реальный способ: отключить неиспользуемые HID-устройства в Диспетчере устройств. Автоматизировать это в общем скрипте небезопасно (можно отключить клавиатуру).

## 2.2 Ядро и планировщик

```bat
bcdedit /set disabledynamictick yes
bcdedit /deletevalue useplatformclock
bcdedit /set hypervisorlaunchtype off
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\kernel" /v GlobalTimerResolutionRequests /t REG_DWORD /d 1 /f
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management" /v FeatureSettingsOverride /t REG_DWORD /d 3 /f
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management" /v FeatureSettingsOverrideMask /t REG_DWORD /d 3 /f
```

1. `disabledynamictick yes` **[B]**. Отключает динамические тики. Уже в `Preset_3`.
2. `deletevalue useplatformclock` **[B]**. Убирает принудительное использование HPET. Уже в `Preset_3`.
3. `hypervisorlaunchtype off` **[A]**. Отключает гипервизор. Перестают работать WSL2, Docker, Hyper-V, Sandbox и VBS. Пункт 7 экспериментального меню.
4. `GlobalTimerResolutionRequests = 1` **[B]**. Возвращает глобальное поведение запросов разрешения таймера. Уже в `Preset_3`.
5. `FeatureSettingsOverride = 3` и `Mask = 3` **[A]**. Задокументированы Microsoft (KB4072698): отключают программные защиты Spectre v2 и Meltdown. **Это снижает безопасность.** На процессорах с аппаратными исправлениями выигрыш обычно мал. По умолчанию проект этого не делает: пункт 6 экспериментального меню с подтверждением `YES`.

Не добавлять **[C]**: `tscsyncpolicy`, `x2apicpolicy`, `uselegacyapicmode` и другие «магические» параметры bcdedit. Измеримого эффекта нет, а риск проблем при загрузке есть.

### MMCSS и сеть

```bat
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile" /v NetworkThrottlingIndex /t REG_DWORD /d 4294967295 /f
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile" /v SystemResponsiveness /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile\Tasks\Games" /v "GPU Priority" /t REG_DWORD /d 8 /f
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile\Tasks\Games" /v Priority /t REG_DWORD /d 6 /f
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile\Tasks\Games" /v "Scheduling Category" /t REG_SZ /d High /f
```

1. `NetworkThrottlingIndex` **[B]**. Снимает ограничение на обработку сетевых пакетов мультимедийной подсистемой.
2. `SystemResponsiveness` **[B]**. Доля процессорного времени, резервируемая для фоновых задач. Документация указывает диапазон 0–100; система может применять нижнюю границу.
3. `Tasks\Games` **[B]**. Профиль приоритета для потоков, зарегистрированных в MMCSS. **Важно:** MMCSS влияет только на потоки, которые сами вызывают MMCSS API. Многие игры этого не делают. Отдельной настройки «распределения между сетью, звуком и игрой» в Windows не существует: сетевой параметр один (`NetworkThrottlingIndex`), у аудиозадач свои профили в `Tasks\Audio`.

### Память

```bat
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management" /v DisablePagingExecutive /t REG_DWORD /d 1 /f
powershell -NoProfile -ExecutionPolicy Bypass -Command "Disable-MMAgent -MemoryCompression -PageCombining"
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management" /v LargeSystemCache /t REG_DWORD /d 0 /f
```

1. `DisablePagingExecutive = 1` **[A]**. Ядро и драйверы остаются в ОЗУ. Только при 16 ГБ ОЗУ и больше (в `Preset_6` проверяется автоматически).
2. `Disable-MMAgent` **[A]**. Отключает сжатие памяти и объединение страниц, снижая нагрузку на CPU. Только при достаточном объёме ОЗУ.
3. `LargeSystemCache = 0` **[A]**. Для настольных ПК оставлять 0. Значение 1 предназначено для файловых серверов и ухудшает игровую производительность.
4. **`IoPageLockLimit` [C].** (В запросе это названо `IoPageLimit`.) Параметр из Windows XP; на Windows 10/11 эффекта не подтверждено. Не добавлять.

### NTFS и NVMe

```bat
fsutil behavior set disablelastaccess 1
fsutil behavior set disable8dot3 1
fsutil behavior set memoryusage 2
fsutil behavior set mftzone 2
fsutil behavior set disabledeletenotify 0
fsutil behavior set encryptpagingfile 0
powershell -NoProfile -ExecutionPolicy Bypass -Command "Get-PnpDevice -Class SCSIAdapter -PresentOnly | Where-Object { $_.FriendlyName -match 'NVM' } | ForEach-Object { $k = 'HKLM:\SYSTEM\CurrentControlSet\Enum\' + $_.InstanceId + '\Device Parameters\StorPort'; New-Item -Path $k -Force | Out-Null; New-ItemProperty -Path $k -Name EnableIdlePowerManagement -Value 0 -PropertyType DWord -Force | Out-Null }"
```

1. `disablelastaccess`, `disable8dot3` **[A]**. Меньше записей метаданных. В `Preset_2`.
2. `memoryusage 2` **[A]**. Увеличивает кэш метаданных NTFS (пул). `mftzone 2` увеличивает резерв под MFT.
3. `disabledeletenotify 0` **[A]**. Включает TRIM.
4. `encryptpagingfile 0` **[A]**. Отключает шифрование файла подкачки (снижает нагрузку на CPU).
5. `EnableIdlePowerManagement = 0` **[A]**. Задокументированный параметр StorPort. Контроллер NVMe не уходит в простой, поэтому первый запрос после паузы быстрее. Пункт 5 экспериментального меню.

Не добавлять **[C]**: `fsutil usn deletejournal`. Журнал USN используют Windows Search, резервное копирование и антивирус. Удаление вызывает полную переиндексацию и ломает инкрементальные операции. Идентификаторы скрытых параметров NVMe-таймаутов в `powercfg` нужно сначала перечислить командой `powercfg /qh` на целевой системе, прежде чем менять.

---

# Часть 3. Расширение Deep Cleaner Ultimate

Добавлены категории **J** (дополнительные кэши и логи) и **K** (пересборка Ngen). Листинг:

```bat
del /f /s /q /a "%SystemRoot%\System32\wbem\Logs\*.log"
del /f /s /q /a "%SystemRoot%\System32\LogFiles\WMI\*.etl"
rd /s /q "%SystemRoot%\Microsoft.NET\Framework64\v4.0.30319\Temporary ASP.NET Files"
"%SystemRoot%\Microsoft.NET\Framework64\v4.0.30319\ngen.exe" executeQueuedItems
"%SystemRoot%\Microsoft.NET\Framework\v4.0.30319\ngen.exe" executeQueuedItems
```

1. Журналы WMI и ETL-файлы автологгеров (`System32\LogFiles\WMI`) накапливаются годами. Журналы CBS, DISM, WER, планировщика и событий уже очищаются категориями 7 и F.
2. Временные файлы ASP.NET, а также временные папки служб `LocalService`, `NetworkService` и `systemprofile` очищаются категорией J.
3. **Образы Ngen удалять нельзя** **[C]**. Папки `C:\Windows\assembly\NativeImages_*` содержат скомпилированные .NET-библиотеки; их удаление замедляет запуск .NET-программ до полной перекомпиляции. Правильное действие: `ngen executeQueuedItems` (категория K) компилирует накопленную очередь.
4. Каталог шрифтов пользователя `%LocalAppData%\Microsoft\Windows\Fonts` очищать нельзя: там лежат установленные шрифты. Безопасный кэш шрифтов (`FontCache`, `FNTCACHE.DAT`) уже в категории I.

Дополнительные команды обслуживания (в скрипт не включены, так как требуют осознанного запуска):

```bat
net stop WSearch
esentutl /d "%ProgramData%\Microsoft\Search\Data\Applications\Windows\Windows.edb"
net start WSearch
net stop wuauserv
net stop bits
ren "%SystemRoot%\SoftwareDistribution" SoftwareDistribution.old
net start bits
net start wuauserv
```

1. `esentutl /d` дефрагментирует базу индекса поиска. Нужно свободное место не меньше размера базы.
2. Блок со сбросом `SoftwareDistribution` решает проблемы Центра обновлений. Это ремонт, а не очистка.

---

# Часть 4. Семь диагностических проверок для анализатора

Все семь **уже реализованы** в `CHECK_OPTIMIZATION_PERCENT.bat` и выводятся отдельным блоком: в процент они не входят, потому что показывают состояние, а не применённые твики.

| № | Проверка | Как измеряется | Что считается проблемой |
|---|---|---|---|
| 1 | Нагрузка DPC и прерываний | CIM-класс `Win32_PerfFormattedData_Counters_ProcessorInformation`, свойства `PercentDPCTime`, `PercentInterruptTime`, 4 замера | DPC от 3 или прерывания от 2 в простое |
| 2 | MSI у сети, звука, NVMe и SATA | Ключ `MSISupported` в `Enum\...\MessageSignaledInterruptProperties` | Информационно: список устройств без MSI |
| 3 | Видеокарта NVIDIA | `nvidia-smi --query-gpu`: P-state, частоты, температура, поколение и ширина PCIe | В P0 ширина линии меньше максимальной |
| 4 | Частота обновления монитора | `Win32_VideoController`: `CurrentRefreshRate` и `MaxRefreshRate` | 60 Гц и меньше при доступных 100 Гц и больше |
| 5 | Подкачка и свободная память | `Win32_PerfFormattedData_PerfOS_Memory`: `PagesInputPersec`, `AvailableMBytes` | Чтение страниц выше 200 в секунду или свободно меньше 1500 МБ в простое |
| 6 | Лимиты CPU в схеме питания | `powercfg /q`: минимум, максимум состояния процессора, режим Boost | Максимум ниже 100 или Boost отключён |
| 7 | Фоновые потребители CPU | Приращение `TotalProcessorTime` процессов за 3 секунды | Процесс забирает более 3 в простое |

Свойства CIM используют английские имена, поэтому проверки не зависят от языка Windows. Пороги ориентировочные: их назначение показать вероятную причину задержек, а не вынести окончательный вердикт. Замер DPC в простое показывает только фоновый уровень; для поиска драйвера-виновника под нагрузкой используйте LatencyMon или xperf.

---

# Часть 5. Что попало в проект

| Элемент | Где |
|---|---|
| NVIDIA: фиксация частот, HDCP | `05_Experimental_Optional\Experimental_Menu.bat`, пункты 1 и 2 |
| Очереди ввода, приоритет csrss | пункты 3 и 4 |
| StorPort NVMe | пункт 5 |
| Spectre и Meltdown (с `YES`) | пункт 6 |
| Гипервизор | пункт 7 |
| Откат всех экспериментальных пунктов | `99_Rollback\ROLLBACK_Restore_Defaults.bat` и режим `U` в самом меню |
| Категории очистки J и K | `Deep_Cleaner_ULTIMATE.bat` |
| 7 диагностических проверок | `CHECK_OPTIMIZATION_PERCENT.bat` |
| Пункт `E` | главное меню `00_START_HERE.bat` |

# Часть 6. Методика замера «до и после»

1. Закройте фоновые программы, подождите 2 минуты после загрузки.
2. `00_START_HERE.bat` → `B` (полный анализ) и сохраните отчёт.
3. Примените один пункт из `Experimental_Menu` и перезагрузитесь.
4. Снова `B`. Сравнение с предыдущим запуском появится в консоли и HTML-отчёте.
5. В игре замерьте 1% и 0.1% low через PresentMon или CapFrameX: 3 прогона одной сцены до и после.
6. Если разница в пределах погрешности (обычно 2–3%), откатите пункт: он не стоит компромисса.

> Процент оптимизации в анализаторе считается по чек-листу самого MX-Optimizer. Для сравнения с другими утилитами используйте только нейтральные метрики: задержки, DPC, ping, джиттер и кадровые метрики в играх.
