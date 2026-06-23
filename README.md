# auto-recovery-point-win10
1. Включить и выделить место
Win+R → sysdm.cpl → вкладка «Защита системы» → выдели диск C: → «Настроить»:

Включить защиту системы
Ползунок «Максимальное использование» — поставь 5–10% (≈10–20 ГБ). Одна точка в среднем 0.5–2 ГБ, на 7+ хватит с запасом. Когда место кончится — Windows автоматически удалит самые старые. Это и есть встроенный механизм «оставлять только свежие».
ОК → ОК.

2. Снять 24-часовой кулдаун
У Windows по умолчанию есть защита: если точка уже создавалась за последние 1440 минут, новая не создаётся (даже принудительно). Снимаем:


New-ItemProperty `
  -Path "HKLM:\Software\Microsoft\Windows NT\CurrentVersion\SystemRestore" `
  -Name "SystemRestorePointCreationFrequency" `
  -Value 0 -PropertyType DWORD -Force
(PowerShell от админа.)

3. Создать ежедневную задачу
Тоже PowerShell от админа — это создаст задачу в Планировщике, которая раз в день делает точку от имени SYSTEM:


$action    = New-ScheduledTaskAction -Execute "powershell.exe" `
             -Argument '-NoProfile -WindowStyle Hidden -Command "Checkpoint-Computer -Description ''Daily Auto'' -RestorePointType MODIFY_SETTINGS"'
$trigger   = New-ScheduledTaskTrigger -Daily -At "09:00"
$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest
$settings  = New-ScheduledTaskSettingsSet -StartWhenAvailable -AllowStartIfOnBatteries `
             -DontStopIfGoingOnBatteries -MultipleInstances IgnoreNew

Register-ScheduledTask -TaskName "Daily Restore Point" `
  -Action $action -Trigger $trigger -Principal $principal -Settings $settings -Force
Что тут важно:

-StartWhenAvailable — если комп был выключен в 09:00, точка создастся при первом включении.
RestorePointType MODIFY_SETTINGS — обычный тип, без ограничений.
Время 09:00 подменяй на удобное.
Проверка
Запусти задачу руками первым же разом, чтобы убедиться, что всё работает:


Start-ScheduledTask -TaskName "Daily Restore Point"
# Подожди ~30 сек и:
Get-ComputerRestorePoint | Sort-Object CreationTime -Descending | Select-Object -First 5
Должна появиться свежая запись с описанием «Daily Auto».
