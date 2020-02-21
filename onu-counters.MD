# ELTEX LTP-8X / MA4000. Получение счетчиков трафика с ONU NTU-1

В этом документе отражены некоторые аспекты работы получения счетчиков (Downstream - Total octets / Upstream - Total octets) для ONU на коммутаторах ELTEX LTP-8X и MA-4000

Предполагается, что для каждой ONU используется только один cross-connect

## Получение счетчиков из CLI
```
LTP-8X# show interface ont <GPON-port>/<ONT-ID> counters cross-connect
```
```
MA4000# show interface ont <SLOT>/<GPON-port>/<ONT-ID> counters cross-connect
```


## Получение счетчиков по SNMP

### Downstream - Total octets
```
snmpget -v2c -c community IP_ADDRESS .1.3.6.1.4.1.35265.1.22.3.3.10.1.6.<SLOT>.8.XX.XX.XX.XX.XX.XX.XX.XX.1.1
```
### Upstream - Total octets
```
snmpget -v2c -c community IP_ADDRESS .1.3.6.1.4.1.35265.1.22.3.3.11.1.6.<SLOT>.8.XX.XX.XX.XX.XX.XX.XX.XX.1.1
```

где:
* SLOT = 1 для LTP-8X и номер платы для MA-4000
* XX.XX.XX.XX.XX.XX.XX.XX - десятичное представление серийного номера ONU - восемь десятичных чисел

### Десятичное представление серийного номера ONU
Для перевода серийного номера ONU в десятичный вид можно воспользоваться [онлайн-конвертером](https://www.rapidtables.com/convert/number/ascii-hex-bin-dec-converter.html).
Порядок действий по переводу с помощью конвертера:

Предположим, ONU имеем серийный номер ELTX72010A04
1) В конвертере в поле __ASCII text__ сначала ввести буквы "ELTX"
2) В поле __Hex (bytes)__ появятся байты "45 4C 54 58"
3) К этим байтам в поле __Hex (bytes)__ добавить оставшуюся часть серийного номера, разбив по 2 цифры, т.е добавить "72 01 0A 04"
4) Таким образом, в поле __Hex (bytes)__ будут байты "45 4C 54 58 72 01 0A 04", а в поле __Decimal (bytes)__ появится их десятичное представление "69 76 84 88 114 1 10 4"
5) Теперь эти числа нужно разделить точками и использовать в SNMP-запросах

Пример:
```
snmpget -v2c -c community IP_ADDRESS .1.3.6.1.4.1.35265.1.22.3.3.11.1.6.<SLOT>.8.69.76.84.88.114.1.10.4.1.1
```
### Bash-скрипт для преобразования серийного номера ONU в десятичное представление
```
#!/bin/bash
serial=$1; 
a=${serial^^}
printf "%d.%d.%d.%d.%d.%d.%d.%d\n" "'${a:0}" "'${a:1}" "'${a:2}" "'${a:3}" "0x${a:4:2}" "0x${a:6:2}" "0x${a:8:2}" "0x${a:10:2}"
```
Пример:
```
# ./serial_dec.sh ELTX1A2B3C4D
69.76.84.88.26.43.60.77
```


## Ссылки

| Ресурсы | Ссылка |
| ------ | ------ |
| МА4000-РХ. Руководство по SNMP. | https://eltex-co.ru/catalog/olt-gpon/ma4000-px/ |
| MA4000-PX. Справочник команд CLI.| https://eltex-co.ru/catalog/olt-gpon/ma4000-px/ |
| LTP-8X (rev.C). Настройка и мониторинг по SNMP | https://eltex-co.ru/catalog/olt-gpon/olt_ltp-8x_rev-c/ |
| LTP-8X (rev.C). Справочник команд CLI. | https://eltex-co.ru/catalog/olt-gpon/olt_ltp-8x_rev-c/ |
| MIB для LTP-8X | https://mibs.observium.org/mib/ELTEX-LTP8X/# |
| ASCII,Hex,Binary,Decimal,Base64 converter | https://www.rapidtables.com/convert/number/ascii-hex-bin-dec-converter.html |

