# ELTEX LTP-8X / MA4000. Управление RF-портом на оборудовании NTU-1C 

В этом документе отражены некоторые аспекты работы с RF-портом на коммутаторах ELTEX LTP-8X и MA-4000

## Предотвращения вещания на незарегистрированном оборудовании
Вещение КТВ через ONU с заводскими настройками по умолчанию включено.
Для предотвращения бесконтрольного вещания можно использовать функцию автоактивации.

* Создать профиль с выключенным RF-портом

```
LTP-8X(config)# template "t-NOTV"
LTP-8X(config)# define rf-port-state
LTP-8X(config)# rf-port-state disabled
LTP-8X(config)# exit
```

*  Настроить автоактивацию этим профилем ONU определённого типа (в нашем случае это NTU-1)
```
LTP-8X(config)# gpon olt auto-activation ont enable
LTP-8X(config)# gpon olt auto-activation ont type NTU-1C template "t-NOTV"
LTP-8X(config)# do commit
```

Этот метод имеет недостаток - при дальшейшем конфигурировании ONU с помощью CLI, необходимо откатывать автонастройки.

> In case of "Auto-Activation ONT" is enabled and any new auto-activated ONTs in configuration appeared, it needed comand "rollback" been entered before making any changes in configuration.

> Revision failed
> Other user commit config data before you or some new auto-activated ONTs appeared in configuration
> Now need: 1) make 'rollback' from root menu (all changed will be lost); 2) make changes of configuration again; 3) try 'commit' again

## Параметр rf-port-state

### Приоритеты параметров

Параметр rf-port-state есть как в составе шаблонов (template), так и непосредственно в конфигурации ONU. Значение параметра в шаблоне более приоритетное, поэтому если планируется управлять RF-портом для каждой конкретной ONU, то в шаблоне надо отключить этот параметр.
```
LTP-8X(ont-template)("t-XXX-TV")# undefine rf-port-state
```

### Конфигурирование rf-port-state из CLI
```
LTP-8X# configure terminal
LTP-8X(config)# interface ont 1/0
LTP-8X(config)(if-ont-1/0)# rf-port-state enabled 
LTP-8X(config)(if-ont-1/0)# do commit
LTP-8X(config)(if-ont-1/0)# 
```

### Конфигурирование rf-port-state по SNMP
```
snmpset -v2c -c community_rw -t 20 IP_ADDRESS .1.3.6.1.4.1.35265.1.22.3.4.1.32.<SLOT>.8.XX.XX.XX.XX.XX.XX.XX.XX i 1
```
где:
* SLOT = 1 для LTP-8X и номер платы для MA-4000
* i =  {disabled(0),enabled(1),noChange(2)}
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
snmpset -v2c -c community_rw -t 20 IP_ADDRESS .1.3.6.1.4.1.35265.1.22.3.4.1.32.1.8.69.76.84.88.114.1.10.4 i 1
```

## Сохраннение конфигурации
Все изменения состояния RF-портов хранятся в оперативной конфигурации (running-config) и могут быть потеряны после перезагрузки коммутатора. Для того, чтобы этого избежать, можно сделать следующее.

### Настройка автосохранения конфигурации
```
LTP-8X(config)# config autosave period 3600
```
### Сохранение конфигурации по SNMP (только LTP-8X)
```
snmpset -v2c -c community_rw -t 20 IP_ADDRESS .1.3.6.1.4.1.35265.1.22.1.50.0 u 1
```


## Ссылки

Dillinger is currently extended with the following plugins. Instructions on how to use them in your own application are linked below.

| Ресурсы |
| ------ |
| [МА4000-РХ. Руководство по SNMP. Версия ПО 3.26.1](https://eltex-co.ru/upload/iblock/ae5/ma4000_nastroyka_i_monitoring_po_snmp.pdf) | 

