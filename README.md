
# Домашнее задание к занятию  «Кеширование Redis/memcached» - Бобков Александр
<details>
<summary><b>Задание 1. Кеширование</b></summary>

- Приведите примеры проблем, которые может решить кеширование. 

*Приведите ответ в свободной форме.*

### ОТВЕТ:
# Резервное копирование домашней директории
**Кеширование помогает решить следующие проблемы производительности и архитектуры:**
- Высокая задержка (Latency): Ускоряет отдачу данных за счет их хранения в оперативной памяти.
- Высокая нагрузка на БД: Защищает основную базу данных от повторных тяжелых запросов.
- Экономия сетевого трафика: Снижает объемы данных, передаваемых между серверами.
- Повторные сложные вычисления: Исключает затраты процессора на генерацию неизменяемых данных.
- Отказоустойчивость: Позволяет отдавать пользователям сохраненную копию данных при сбое источника.
</details>

------
------


<details>
<summary><b>Задание 2. Memcached</b></summary>

- Установите и запустите memcached.

*Приведите скриншот systemctl status memcached, где будет видно, что memcached запущен.*

------

### ОТВЕТ:
Установка и запуск сервиса в Debian выполнены с помощью команд:

```bash
sudo apt update && sudo apt install memcached -y
sudo systemctl start memcached
sudo systemctl enable memcached
sudo systemctl status memcached
```
**Скриншот статуса службы memcached:**
![Статус Memcached](./img/1.jpg)




</details>

-------
-------

<details>
<summary><c>Задание 3. Удаление по TTL в Memcached</c></summary>

- Запишите в memcached несколько ключей с любыми именами и значениями, для которых выставлен TTL 5. 

*Приведите скриншот, на котором видно, что спустя 5 секунд ключи удалились из базы.*
-------

### ОТВЕТ:
Для исключения ошибок интерпретации скрытых символов терминала (`CLIENT_ERROR bad data chunk`), запись и проверка удаления по TTL (5 секунд) выполнены атомарной отправкой корректных CRLF-строк через `printf`:

```bash
printf "set key1 0 5 4\r\nval1\r\nget key1\r\n" | nc localhost 11211 && sleep 6 && printf "get key1\r\n" | nc localhost 11211
```

**Результат выполнения команды:**
```text
STORED
VALUE key1 0 4
val1
END
END
```
*Пояснение:* Сначала ключ успешно создается (`STORED`) и считывается (`VALUE`). Спустя 6 секунд ожидания повторный запрос возвращает пустой ответ (`END`), подтверждая удаление ключа по TTL.

**Скриншот фиксации удаления ключей по TTL:**
![Удаление ключей по TTL](./img/2.jpg)







</details>

------
------


<details>
<summary><c>Задание 4*</c></summary>

- Напишите скрипт, который будет производить инкрементное резервное копирование домашней директории пользователя с помощью rsync на другой сервер
- Скрипт должен удалять старые резервные копии (сохранять только последние 5 штук)
- Напишите скрипт управления резервными копиями, в нем можно выбрать резервную копию и данные восстановятся к состоянию на момент создания данной резервной копии.
- На проверку направьте скрипт и скриншоты, демонстрирующие его работу в различных сценариях.

------
### ОТВЕТ:

## 1.  Для решения задачи по инкрементному копированию  будем использовать механизм hard links в rsync (флаг --link-dest). Это позволяет каждой копии выглядеть как полная, но занимать место только для измененных файлов.

- Сам скрипт:

```bash
#!/bin/bash

# --- НАСТРОЙКИ ---
SOURCE="$HOME/"
REMOTE_USER="user" # Под каким пользователем
REMOTE_HOST="192.168.32.130"  # IP принимающей стороны
BACKUP_ROOT="/tmp/backups"
TIMESTAMP=$(date +%Y-%m-%d_%H-%M-%S)
CURRENT_BACKUP="$BACKUP_ROOT/$TIMESTAMP"
LATEST_LINK="$BACKUP_ROOT/latest"

echo "--- Старт инкрементного бэкапа: $TIMESTAMP ---"

# 1. Подготовка структуры на сервере
ssh $REMOTE_USER@$REMOTE_HOST "mkdir -p $BACKUP_ROOT"

# 2. Запуск rsync
# Ссылка ../latest ищется относительно создаваемой папки бэкапа
if rsync -avz --delete --exclude='.*/' \
      --link-dest="../latest" \
      "$SOURCE" "$REMOTE_USER@$REMOTE_HOST:$CURRENT_BACKUP"; then
    
    echo "[OK] Данные переданы успешно."
    
    # 3. Обновление ссылки и ротация через алфавитную сортировку
    ssh $REMOTE_USER@$REMOTE_HOST "
        cd $BACKUP_ROOT
        
        # Находим самую новую папку (последняя в алфавитном списке)
        ACTUAL_NEWEST=\$(ls -1 | grep '^20' | sort | tail -n 1)
        
        if [ -n \"\$ACTUAL_NEWEST\" ]; then
            ln -snf \"\$ACTUAL_NEWEST\" latest
            echo \"Ссылка latest теперь указывает на: \$ACTUAL_NEWEST\"
        fi
        
        # Ротация: сортируем от новых к старым и удаляем всё после 5-й
        OLD_BACKUPS=\$(ls -1 | grep '^20' | sort -r | tail -n +6)
        
        if [ -n \"\$OLD_BACKUPS\" ]; then
            echo \"Удаляю лишние копии: \$OLD_BACKUPS\"
            rm -rf \$OLD_BACKUPS
        else
            echo \"В хранилище 5 или менее копий. Удаление не требуется.\"
        fi
    "
else
    echo "[ERROR] Ошибка rsync! Ссылка latest не обновлена."
    exit 1
fi

echo "--- Завершено ---"

```
* **Отработка скрипта**

<summary>Результат отработки скрипта</summary>
<img src="img/7.jpg" width = 100%>

<summary>Ротация на принимающем сервере</summary>
<img src="img/8.jpg" width = 100%>

## 2. Cкрипт управления резервными копиями/

```bash
#!/bin/bash

# --- НАСТРОЙКИ (должны совпадать с backup.sh) ---
REMOTE_USER="user"  #Под кем запускать
REMOTE_HOST="192.168.32.130"  # IP сервера на котором копия
BACKUP_ROOT="/tmp/backups"
RESTORE_PATH="$HOME/restored_data"

echo "=== Мастер восстановления данных ==="

# 1. Получаем список бэкапов, отсортированный от новых к старым
echo "Запрос списка копий с сервера..."
BACKUPS=$(ssh $REMOTE_USER@$REMOTE_HOST "ls -1 $BACKUP_ROOT | grep '^20' | sort -r")

if [ -z "$BACKUPS" ]; then
    echo "Ошибка: Резервные копии не найдены в $BACKUP_ROOT"
    exit 1
fi

# 2. Интерактивное меню
echo "Выберите копию для восстановления (1 — самая свежая):"
PS3="Введите номер: "

select SELECTED_BACKUP in $BACKUPS; do
    if [ -n "$SELECTED_BACKUP" ]; then
        echo "--> Выбрана точка: $SELECTED_BACKUP"
        
        # Подготовка локальной папки
        echo "Подготовка папки $RESTORE_PATH..."
        [ -d "$RESTORE_PATH" ] && rm -rf "$RESTORE_PATH"
        mkdir -p "$RESTORE_PATH"
        
        # 3. Копирование данных обратно с сервера
        echo "Начинаю загрузку данных..."
        if rsync -avz "$REMOTE_USER@$REMOTE_HOST:$BACKUP_ROOT/$SELECTED_BACKUP/" "$RESTORE_PATH/"; then
            echo "=== Успех! Данные восстановлены в: $RESTORE_PATH ==="
        else
            echo "!!! Произошла ошибка при передаче данных через rsync."
        fi
        break
    else
        echo "Неверный выбор. Пожалуйста, введите число из списка."
    fi
done

```
* **Отработка скрипта восстановления**

<summary>Результат отработки скрипта восстановления</summary>
<img src="img/9.jpg" width = 100%>
<img src="img/10.jpg" width = 100%>

<summary>Смотрим что данные восстановились</summary>
<img src="img/11.jpg" width = 100%>



</details>

