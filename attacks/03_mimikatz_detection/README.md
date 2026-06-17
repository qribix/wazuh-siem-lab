# Анализ атаки: дамп учётных данных с помощью Mimikatz

**Агент:** siem (IP: 192.168.1.13)

**Сценарий:** Злоумышленник уже находится в системе и пытается извлечь учётные данные из памяти LSASS с помощью утилиты Mimikatz.

## Установка Sysmon

**Sysmon (System Monitor)** - системная служба Windows, которая логирует создания процессов, сетевые подключения и другие действия, помогая детектировать подозрительную активность.

1. Скачиваем Sysmon с [официального сайта](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon) и конфигурационный файл [SwiftOnSecurity](https://github.com/SwiftOnSecurity/sysmon-config).

   ![Скачивание Sysmon и конфига](screenshots/1.png)

2. Распаковываем архив и помещаем `sysmonconfig.xml` в папку с `Sysmon64.exe`.

   ![Распаковка файлов](screenshots/2.png)

3. Устанавливаем Sysmon через PowerShell:

   ```powershell
   .\Sysmon64.exe -i .\sysmonconfig.xml
   ```

4. Проверяем установку:

   ```powershell
   Get-Service Sysmon64
   ```

   ![Проверка службы Sysmon](screenshots/3.png)

5. Проверяем наличие Sysmon в **Event Viewer**:

   ```
   Applications and Services Logs -> Microsoft -> Windows -> Sysmon -> Operational
   ```

   ![Проверка в Event Viewer](screenshots/4.png)

## Настройка Wazuh для сбора логов Sysmon

Для передачи логов Sysmon в Wazuh добавляем в `ossec.conf` секцию:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

![ossec.conf](screenshots/5.png)

Перезапускаем службу:

```powershell
Restart-Service WazuhSvc
```

После установки Sysmon логирует все действия, которые передаются в Wazuh для анализа.

## Запуск Mimikatz

Скачиваем Mimikatz с [GitHub](https://github.com/ParrotSec/mimikatz/blob/master/x64/mimikatz.exe):

![Скачивание Mimikatz](screenshots/6.png)

![Mimikatz на ПК](screenshots/7.png)

Запускаем Mimikatz и добавляем его в исключения Defender:

```powershell
.\mimikatz.exe
Add-MpPreference -ExclusionProcess "mimikatz.exe"
```

![Запуск Mimikatz](screenshots/8.png)

## Создание правила обнаружения в Wazuh

После запуска Mimikatz в Wazuh не обнаружено алертов - стандартных правил недостаточно.

![Запуск Mimikatz](screenshots/9.png)

Настраиваем Wazuh на архивирование всех логов через `ossec.conf`:

```bash
nano /var/ossec/etc/ossec.conf
```

![ossec.conf на сервере](screenshots/10.png)

Перезапускаем менеджер:

```bash
systemctl restart wazuh-manager
```

Проверяем архивы:

```bash
cd /var/ossec/logs/archives/
ls
```

![Архивы логов](screenshots/11.png)

Настраиваем `filebeat.yml`:

![filebeat.yml](screenshots/12.png)

Перезапускаем Filebeat:

```bash
systemctl restart filebeat
```

Создаём индекс для архивов:

![Создание индекса 1](screenshots/13.png)
![Создание индекса 2](screenshots/14.png)
![Создание индекса 3](screenshots/15.png)
![Создание индекса 4](screenshots/16.png)
![Создание индекса 5](screenshots/17.png)

Снова запускаем Mimikatz и проверяем логи в Wazuh:

![Запуск mimikatz](screenshots/18.png)
![Проверка логов](screenshots/19.png)

Переходим в раздел правил и находим там правило Sysmon, которое будем использовать как основу для создания своего:

![Правило Sysmon](screenshots/20.png)
![Правило Sysmon 2](screenshots/21.png)
![Правило Sysmon 3](screenshots/22.png)

Копируем секцию `sysmon_event1` для создания кастомного правила:

![sysmon_event1](screenshots/23.png)

Создаём кастомное правило в `local_rules.xml`:

![Создание правила 1](screenshots/24.png)
![Создание правила 2](screenshots/25.png)
![Создание правила 3](screenshots/26.png)

Правило:

```xml
<rule id="100002" level="12">
  <if_group>sysmon_event1</if_group>
  <field name="win.eventdata.originalFileName" type="pcre2">(?i)mimikatz\.exe</field>
  <description>Mimikatz Usage Detected</description>
  <mitre>
    <id>T1003</id>
  </mitre>
</rule>
```

![Рестарт системы](screenshots/27.png)

**Важно:** Мы ищем по `originalFileName`, а не по `Image`. Это поле хранит оригинальное имя файла из метаданных, которое не меняется при переименовании. Если злоумышленник переименует `mimikatz.exe` в `notepad.exe`, правило всё равно сработает.

## Проверка обхода переименования

Переименовываем `mimikatz.exe`, запускаем и проверяем Wazuh:

![Переименование и запуск](screenshots/28.png)

![Проверка логов 1](screenshots/29.png)
![Проверка логов 2](screenshots/30.png)
![Алерт сработал](screenshots/31.png)
![Детали алерта](screenshots/32.png)

## Вывод

В ходе анализа обнаружена попытка использования Mimikatz для дампа учётных данных:

1. **Mimikatz запущен** на агенте Windows
2. **Sysmon зафиксировал** создание процесса
3. **Wazuh задетектил** по кастомному правилу (`originalFileName`)

---

## MITRE ATT&CK:

| Действие | Техника |
|----------|---------|
| Дамп учётных данных из памяти LSASS | **T1003.001** (Память процесса LSASS) |

---

## Примечание

Правило использует поле `win.eventdata.originalFileName`, которое проверяет оригинальное имя файла из метаданных, а не имя на диске. Это позволяет детектировать Mimikatz даже после переименования.

Wazuh из коробки не детектит Mimikatz - потребовалось создать кастомное правило уровня 12.

---

## Рекомендации

1. **Использовать Sysmon** для детального логирования процессов
2. **Настроить кастомные правила** на обнаружение известных утилит
3.  Включить сохранение всех логов в Wazuh, чтобы можно было искать события за любой период, а не только те, на которые сработали правила.
4. **Мониторить поля `originalFileName`** для обнаружения переименованных утилит

## Итог

Wazuh с Sysmon и кастомным правилом успешно задетектировал Mimikatz. Уровень алерта 12 - высокий приоритет, событие требует немедленного внимания аналитиков SOC. Переименование файла не помогло обойти правило.
