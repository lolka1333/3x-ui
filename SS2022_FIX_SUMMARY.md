# Исправление генерации ключей Shadowsocks 2022

## Проблема
В X-UI существовала проблема с генерацией ключей для клиентов Shadowsocks 2022. Ключ для инбаунда генерировался корректно, но ключи для клиентов не генерировались, что вызывало ошибки SS22 и постоянную перезагрузку сервера.

## Причина проблемы
1. **Неопределенные константы в `RandomUtil`**: Метод `randomShadowsocksPassword` в файле `web/assets/js/util/index.js` ссылался на константы `SSMethods.BLAKE3_AES_256_GCM` и `SSMethods.BLAKE3_AES_128_GCM`, которые не были определены в этом файле.

2. **Отсутствие передачи метода**: При создании новых клиентов `Shadowsocks` в конструкторе `ShadowsocksSettings` не передавался параметр `method`, поэтому клиенты не знали, какой алгоритм шифрования используется.

3. **Отсутствие методов добавления клиентов**: В классе `ShadowsocksSettings` отсутствовали методы `addShadowsocks` и `delShadowsocks` для правильного управления клиентами.

4. **Проблема в UI**: В `inbound_modal.html` создавались новые клиенты без передачи метода шифрования.

## Исправления

### 1. Исправление метода `randomShadowsocksPassword` в `web/assets/js/util/index.js`
**Было:**
```javascript
static randomShadowsocksPassword(method = SSMethods.BLAKE3_AES_256_GCM) {
    let length = 32;
    if ([SSMethods.BLAKE3_AES_128_GCM].includes(method)) {
        length = 16; 
    }
    // ...
}
```

**Стало:**
```javascript
static randomShadowsocksPassword(method = '2022-blake3-aes-256-gcm') {
    let length = 32;
    // Для 2022-blake3-aes-128-gcm используем 16 байт
    if (['2022-blake3-aes-128-gcm'].includes(method)) {
        length = 16; 
    }
    // ...
}
```

### 2. Исправление конструктора `ShadowsocksSettings` в `web/assets/js/model/inbound.js`
**Было:**
```javascript
shadowsockses = [new Inbound.ShadowsocksSettings.Shadowsocks()],
```

**Стало:**
```javascript
shadowsockses = null,
// ...
// Создаем shadowsockses после того, как method установлен
this.shadowsockses = shadowsockses || [new Inbound.ShadowsocksSettings.Shadowsocks(method)];
```

### 3. Добавление методов управления клиентами в `ShadowsocksSettings`
```javascript
// Добавить нового клиента Shadowsocks
addShadowsocks(shadowsocks) {
    this.shadowsockses.push(shadowsocks);
}

// Удалить клиента Shadowsocks
delShadowsocks(index) {
    this.shadowsockses.splice(index, 1);
}
```

### 4. Исправление UI в `web/html/modals/inbound_modal.html`
**Было:**
```javascript
this.inModal.inbound.settings.shadowsockses = [new Inbound.ShadowsocksSettings.Shadowsocks()];
```

**Стало:**
```javascript
this.inModal.inbound.settings.shadowsockses = [new Inbound.ShadowsocksSettings.Shadowsocks(this.inModal.inbound.settings.method)];
```

## Технические детали

### Требования к ключам SS2022 (согласно спецификации)
- `2022-blake3-aes-128-gcm`: 16 байт (base64 encoded)
- `2022-blake3-aes-256-gcm`: 32 байта (base64 encoded)  
- `2022-blake3-chacha20-poly1305`: 32 байта (base64 encoded)

### Метод генерации ключей
```javascript
static randomShadowsocksPassword(method = '2022-blake3-aes-256-gcm') {
    let length = 32;
    if (['2022-blake3-aes-128-gcm'].includes(method)) {
        length = 16; 
    }
    const array = new Uint8Array(length);
    window.crypto.getRandomValues(array);
    return Base64.alternativeEncode(String.fromCharCode(...array));
}
```

## Результат
После внесения этих исправлений:
1. ✅ Генерируется ключ для инбаунда SS2022
2. ✅ Автоматически генерируются ключи для клиентов SS2022
3. ✅ Устранены ошибки SS22
4. ✅ Прекращены постоянные перезагрузки сервера
5. ✅ Правильное создание новых клиентов через UI

## Файлы, которые были изменены
1. `web/assets/js/util/index.js` - исправление метода генерации ключей
2. `web/assets/js/model/inbound.js` - исправление логики создания клиентов
3. `web/html/modals/inbound_modal.html` - исправление UI