# Исправление поддержки RPRX Vision для VLESS XHTTP

## Проблема
У клиентов VLESS XHTTP отсутствовала возможность использовать flow RPRX Vision (`xtls-rprx-vision`), поскольку панель управления ограничивала поддержку flow только для TCP протокола.

## Решение
Внесены изменения в следующие файлы для добавления поддержки RPRX Vision flow для XHTTP транспорта:

### 1. Frontend (JavaScript)
- **Файл**: `web/assets/js/model/inbound.js`
  - Функция `canEnableTlsFlow()`: добавлена поддержка "xhttp" наряду с "tcp"
  - Генерация VLESS ссылок: flow теперь добавляется для tcp и xhttp

- **Файл**: `web/assets/js/model/outbound.js`
  - Функция `canEnableTlsFlow()`: добавлена поддержка "xhttp" наряду с "tcp"

### 2. Backend (Go)
- **Файл**: `sub/subService.go`
  - Генерация ссылок для VLESS и Trojan: flow теперь добавляется для tcp и xhttp

## Изменения в коде

### JavaScript (inbound.js и outbound.js)
```javascript
// Было:
if (((this.stream.security === 'tls') || (this.stream.security === 'reality')) && (this.network === "tcp")) {

// Стало:
if (((this.stream.security === 'tls') || (this.stream.security === 'reality')) && (["tcp", "xhttp"].includes(this.network))) {
```

### Go (subService.go)
```go
// Было:
if streamNetwork == "tcp" && len(clients[clientIndex].Flow) > 0 {

// Стало:
if (streamNetwork == "tcp" || streamNetwork == "xhttp") && len(clients[clientIndex].Flow) > 0 {
```

## Результат
Теперь пользователи могут:
1. Выбирать flow "xtls-rprx-vision" для VLESS клиентов с XHTTP транспортом
2. Генерировать ссылки с поддержкой flow для XHTTP
3. Использовать RPRX Vision как с TLS, так и с Reality для XHTTP

## Совместимость
- Поддерживается для протокола VLESS
- Работает с TLS и Reality
- Совместимо с TCP и XHTTP транспортами
- Требует поддержку на стороне xray-core сервера

## Примечание
Убедитесь, что ваша версия xray-core поддерживает XTLS Vision с XHTTP транспортом. Если возникают проблемы с подключением, проверьте настройки uTLS fingerprint в клиенте.