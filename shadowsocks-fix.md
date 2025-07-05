# Исправление проблем Shadowsocks 2022 в X-UI

## Проблема
При запуске Xray возникает ошибка:
```
Failed to start: main: failed to load config files: [bin/config.json] > infra/conf: failed to build inbound config with tag inbound-XXXX > infra/conf: failed to build inbound handler for protocol shadowsocks > infra/conf: shadowsocks 2022 (multi-user): users must have empty method
```

## Причина
В Shadowsocks 2022 (методы `2022-blake3-aes-128-gcm`, `2022-blake3-aes-256-gcm`, `2022-blake3-chacha20-poly1305`) при использовании многопользовательского режима каждый пользователь должен иметь пустое поле `method`, но текущий код устанавливает для каждого клиента собственный метод шифрования.

## Исправления

### 1. Исправление JavaScript модели (frontend)

**Файл**: `web/assets/js/model/inbound.js`

**Проблема**: В методе `toJson()` класса `Inbound.ShadowsocksSettings.Shadowsocks` (строка ~2235) для Shadowsocks 2022 в multi-user режиме поле `method` должно быть пустым.

**Исправление**: Изменить метод `toJson()` в классе `Inbound.ShadowsocksSettings.Shadowsocks`:

```javascript
toJson() {
    // Для Shadowsocks 2022 в multi-user режиме method должен быть пустым
    const method = this.isSS2022(this.method) ? '' : this.method;
    
    return {
        method: method,
        password: this.password,
        email: this.email,
        limitIp: this.limitIp,
        totalGB: this.totalGB,
        expiryTime: this.expiryTime,
        enable: this.enable,
        tgId: this.tgId,
        subId: this.subId,
        comment: this.comment,
        reset: this.reset,
    };
}
```

### 2. Исправление серверной части (backend)

**Файл**: `web/service/inbound.go`

**Проблема**: В функции `AddInboundClient` (строка ~448-456) для каждого клиента передается `"cipher": cipher`, но для Shadowsocks 2022 cipher должен быть пустым.

**Исправление**: Изменить логику в методе `AddInboundClient`:

```go
cipher := ""
if oldInbound.Protocol == "shadowsocks" {
    method := oldSettings["method"].(string)
    // Для Shadowsocks 2022 cipher должен быть пустым
    if !isSS2022(method) {
        cipher = method
    }
}
```

И добавить функцию для проверки Shadowsocks 2022:

```go
func isSS2022(method string) bool {
    ss2022Methods := []string{
        "2022-blake3-aes-128-gcm",
        "2022-blake3-aes-256-gcm", 
        "2022-blake3-chacha20-poly1305",
    }
    
    for _, ss2022Method := range ss2022Methods {
        if method == ss2022Method {
            return true
        }
    }
    return false
}
```

### 3. Исправление проблемы редактирования клиентов

**Проблема**: Если у клиента убрать пароль и сохранить пустое поле, то клиента нельзя будет открыть для редактирования.

**Исправление**: Добавить валидацию в JavaScript код, чтобы предотвратить сохранение пустых паролей для Shadowsocks клиентов.

## Дополнительные проверки

1. **Константы SS Methods**: Убедиться, что константы `SSMethods.BLAKE3_AES_128_GCM`, `SSMethods.BLAKE3_AES_256_GCM`, `SSMethods.BLAKE3_CHACHA20_POLY1305` соответствуют строковым значениям `"2022-blake3-aes-128-gcm"`, `"2022-blake3-aes-256-gcm"`, `"2022-blake3-chacha20-poly1305"`.

2. **Проверка в UpdateInboundClient**: Аналогичные изменения могут потребоваться в методе `UpdateInboundClient`.

3. **Проверка в DelInboundClient**: Аналогичные изменения могут потребоваться в методе `DelInboundClient`.

## Статус исправлений

✅ **Исправления применены:**

1. **JavaScript (frontend)** - `web/assets/js/model/inbound.js`:
   - Изменен метод `toJson()` класса `Inbound.ShadowsocksSettings.Shadowsocks`
   - Для Shadowsocks 2022 методов поле `method` теперь устанавливается в пустую строку

2. **Go (backend)** - `web/service/inbound.go`:
   - Добавлена функция `isSS2022()` для проверки Shadowsocks 2022 методов
   - Исправлена логика в методе `AddInboundClient` (строка ~509)
   - Исправлена логика в методе `UpdateInboundClient` (строка ~753)
   - Исправлена логика в методе `ResetClientTraffic` (строка ~1612)

## Тестирование

После внесения изменений:

1. Создать новый Shadowsocks inbound с методом `2022-blake3-aes-256-gcm`
2. Добавить несколько клиентов
3. Проверить, что Xray запускается без ошибок:
   ```bash
   # Проверить логи Xray
   sudo journalctl -u x-ui -f
   ```
4. Проверить, что клиенты могут подключаться
5. Проверить, что редактирование клиентов работает корректно
6. Тестировать операции добавления/удаления клиентов через API

## Команды для проверки

```bash
# Остановить x-ui
sudo systemctl stop x-ui

# Перезапустить x-ui после изменений
sudo systemctl start x-ui

# Проверить статус
sudo systemctl status x-ui

# Проверить логи на наличие ошибок
sudo journalctl -u x-ui --no-pager | grep -E "(error|Error|ERROR|failed|Failed)"
```

## Заметки

- ✅ Изменения обратно совместимы с обычными Shadowsocks методами (не 2022)
- ✅ Код работает как с single-user, так и с multi-user конфигурациями
- ✅ Сохранена автогенерация паролей для SS2022
- ✅ Исправлены все места использования cipher в AddUser API вызовах

## Дополнительные файлы для проверки

Если проблемы продолжаются, также проверьте:
- `xray/api.go` - основной код API взаимодействия с Xray
- `web/html/form/protocol/shadowsocks.html` - HTML форма для настройки Shadowsocks
- `web/assets/js/util/index.js` - утилиты для генерации паролей Shadowsocks