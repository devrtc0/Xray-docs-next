# Исходящие подключения

Исходящие подключения используются для отправки данных. Доступные протоколы см. в разделе [Исходящие протоколы](./outbounds/).

## OutboundObject

`OutboundObject` соответствует дочернему элементу поля `outbounds` в конфигурационном файле.

::: tip
Первый элемент в списке используется как основной исходящий узел.  
Если совпадений с правилами маршрутизации нет или ни одно правило не сработало, трафик отправляется через основной исходящий узел.
:::

```json
{
  "outbounds": [
    {
      "sendThrough": "0.0.0.0",
      "protocol": "название протокола",
      "settings": {},
      "tag": "тег",
      "streamSettings": {},
      "proxySettings": {
        "tag": "another-outbound-tag"
      },
      "mux": {}
    }
  ]
}
```

> `sendThrough`: address

IP-адрес, используемый для отправки данных.  
Этот параметр используется, если на хосте настроено несколько IP-адресов.  
Значение по умолчанию - `"0.0.0.0"`.

Разрешается указывать блок IPv6 CIDR (например, `114:514:1919:810::/64`).  
Xray будет использовать случайный IP-адрес из этого блока для установления исходящих соединений.  
Необходимо правильно настроить сетевое подключение, таблицу маршрутизации и параметры ядра, чтобы разрешить Xray привязываться к любому IP-адресу из этого блока.

Специальное значение `origin` — при использовании этого значения запросы будут отправляться с IP-адреса, через который было установлено соединение с локальной машиной.

Например, если на машине есть целый блок IPv4-адресов `11.4.5.0/24` и прослушивается 0.0.0.0 (все IPv4 и IPv6 адреса на сетевой карте), и клиент подключается к локальной машине через `11.4.5.14`, то исходящие запросы также будут отправляться через `11.4.5.14`. Если же клиент использует `11.4.5.10` для подключения к локальной машине, то исходящие запросы будут отправляться через `11.4.5.10`. То же самое относится и к случаям, когда на машине есть целый блок/несколько IPv6-адресов.

> `protocol`: string

Название протокола подключения.  
Список доступных протоколов см. в разделе "Исходящие подключения" в левой части документации.

> `settings`: OutboundConfigurationObject

Конкретные настройки зависят от протокола.  
См. описание `OutboundConfigurationObject` для каждого протокола.

> `tag`: string

Тег этого исходящего подключения, используемый для идентификации этого подключения в других настройках.

::: danger
Если это поле не пустое, его значение должно быть **уникальным** среди всех тегов.
:::

> `streamSettings`: [StreamSettingsObject](./transport.md#streamsettingsobject)

Тип транспорта (transport) - это способ взаимодействия текущего узла Xray с другими узлами.

> `proxySettings`: [ProxySettingsObject](#proxysettingsobject)

Настройки исходящего прокси.  
Если исходящий прокси включен, параметр `streamSettings` этого исходящего подключения игнорируется.

> `mux`: [MuxObject](#muxobject)

Настройки Mux. Mux позволяет мультиплексировать несколько TCP-соединений через одно TCP-соединение. У Mux есть дополнительная функция: передача UDP-соединений как XUDP.

### ProxySettingsObject

```json
{
  "tag": "another-outbound-tag"
}
```

> `tag`: string

При указании тега другого исходящего подключения данные, отправляемые этим исходящим подключением, будут перенаправлены через указанное исходящее подключение.

::: danger
Этот способ пересылки **не использует** транспортный уровень.  
Если вам нужна пересылка с использованием транспортного уровня, используйте [SockOpt.dialerProxy](./transport.md#sockoptobject).
:::

::: danger
Этот параметр несовместим с SockOpt.dialerProxy.
:::

::: tip
Совместим с настройкой `transportLayer` в v2fly/v2ray-core [transportLayer](https://www.v2fly.org/config/outbounds.html#proxysettingsobject).
:::

### MuxObject

Функция Mux позволяет мультиплексировать несколько TCP-соединений по одному TCP-соединению.  
Подробнее см. [Mux.Cool](../../development/protocols/muxcool).  
Mux предназначен для сокращения задержек при установлении TCP-соединений, а не для увеличения пропускной способности.  
Использование Mux при просмотре видео, загрузке файлов или тестировании скорости обычно приводит к обратным результатам.  
Mux нужно включать только на клиенте, сервер автоматически адаптируется.

`MuxObject` соответствует полю `mux` в `OutboundObject`.

```json
{
  "enabled": true,
  "concurrency": 8,
  "xudpConcurrency": 16,
  "xudpProxyUDP443": "reject"
}
```

> `enabled`: true | false

Включить пересылку запросов через Mux.  
Значение по умолчанию - `false`.

> `concurrency`: number

Максимальное количество одновременных соединений. Минимальное значение `1`, максимальное значение `128`. Если параметр опущен или задан `0`, то используется значение `8`, любые значения, превышающие `128`, будут интерпретированы как 128.

Это значение определяет максимальное количество дочерних соединений, которые могут быть мультиплексированы по одному TCP-соединению.  
Например, если `concurrency` равен `8`, то при отправке 8 TCP-запросов клиентом Xray создаст только одно фактическое TCP-соединение, и все 8 запросов клиента будут передаваться по этому соединению.

Ядро не будет повторно использовать закрытые идентификаторы подсоединений, это означает, что это фактически количество повторного использования одного соединения. Например, если установлено значение `16`, и это соединение уже было повторно использовано 16 раз, из которых 10 уже закрыты, это не освободит для него десять слотов. Ядро всё равно будет считать, что это соединение исчерпало лимит повторного использования и откроет новое соединение.

::: tip
Если указать отрицательное значение, например `-1`, трафик TCP не будет проходить через Mux.
:::

> `xudpConcurrency`: number

Использовать новый агрегированный туннель XUDP (т.е. другое Mux-соединение) для проксирования UDP-трафика.  
Укажите максимальное количество одновременных дочерних UoT-подключений.  
Минимальное значение - `1`, максимальное значение - `1024`.  
Если этот параметр опущен или равен `0`, UDP-трафик будет использовать тот же путь, что и TCP-трафик (традиционное поведение).

::: tip
Если указать отрицательное значение, например `-1`, трафик UDP не будет проходить через Mux.  
Будет использоваться исходный способ передачи UDP-трафика для данного протокола прокси.  
Например, `Shadowsocks` будет использовать нативный UDP, а `VLESS` будет использовать UoT.
:::

> `xudpProxyUDP443`: string

Управление обработкой проксируемого трафика UDP/443 (QUIC) в Mux:

- По умолчанию `reject` - отклонять трафик (обычно браузеры автоматически переключаются на TCP HTTP/2).
- `allow` - разрешить трафик через Mux.
- `skip` - не использовать Mux для трафика UDP/443.  
   Будет использоваться исходный способ передачи UDP-трафика для данного протокола прокси.  
   Например, `Shadowsocks` будет использовать нативный UDP, а `VLESS` будет использовать UoT.
