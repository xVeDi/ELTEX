# Подключение абонентов по технологии IPoE с использованием коммутаторов Eltex LTE-8X

Приводится решение задачи обеспечения защиты от самостоятельного ввода пользователем IP-адреса с использованием средств ExtremeOS.
Необходимость такого рода задач возникает при использовании в качестве коммутаторов доступа устройств, не имеющих встроенных функций IP-MAC-Port bindind или аналогичных. Примером  такого устройства является коммутатор Eltex LTE-8X (ПО 3.16.0), 
В операционной системе ExtremeOS задача решается с помощью механизма Source IP Lockdown.

## Постановка задачи
Обеспечить защиту от самостоятельного ввода пользователем IP-адреса с использованием средств ExtremeOS. Обеспечить параллельную работу на коммутаторе LTE-8X схем предоставления услуг PPPoE и IPoE

**Исходные данные**
Предполагается, что настроена supervlan и добавлен sublan. Для этой связки настроен и корректно работает bootprelay.

*Коммутатор Extreme*
42,48 порты – Uplink (кольцо ERPS)
40 порт - Downlink (в данном случае - LTE-8X)

*Абонентская vlan:*
sub_vlan_access (tag 1001)

*Стыковая vlan:*
Kolchugino_Subscribers 

*Supervlan:*
vsuper1

*Management vlan:*
Management_Kolchugino (tag 21)

*Multicast vlan:*
MulticastVlan (tag 250)

*PPPoE vlan:*
PPPoE_Kolchugino (tag 208)
PPPoE_Hydra_KLG (tag 221)

## Решение

### Настройка коммутатора LTE-8X
```
# Посредством командной строки или WEB-интерфейса необходимо создать группу правил (Rules) следующего содержания, и применять эту группу (Rules) для Profiles тех ONU, которые работают по схеме IPoE

LTE-8X-6(profile-rules)# rule show pon 

 0) 4: if (VID != 1001) then discard
 1) 5: if (VID == 1001) then DeleteTag
 2) 14: if (LinkIndex == 0x0) then path = port 0 queue 0; forward
 3) 14: if (LinkIndex == 0x1) then path = port 0 queue 1; forward
 4) 14: if (LinkIndex == 0x2) then path = port 1 queue 0; forward
 5) 14: if (LinkIndex == 0x3) then path = port 1 queue 1; forward

LTE-8X-6(profile-rules)# rule show uni0

 0) 0: if (L3Proto == 0x2) then ClearAddTag
 1) 5: if (Always) then AddTagVID = 1001
 2) 14: if (Always) then path = link 0 queue 0; forward

LTE-8X-6(profile-rules)# rule show uni1

 0) 4: if (L3Proto == 0x2) then ClearAddTag
 1) 5: if (Always) then AddTagVID = 1001
 2) 14: if (Always) then path = link 2 queue 0; forward
```

```
# Разрешить приём DHCP-offer и DHCP-ack от всех серверов (для VLAN PPPoE и IPoE сервера будут разными)
LTE-8X-6# 
LTE-8X-6# switch 
LTE-8X-6(switch)# configure 
LTE-8X-6(switch)(config)# no ip dhcp trusted-server-ip primary
LTE-8X-6(switch)(config)# no ip dhcp trusted-server-ip secondary
LTE-8X-6(switch)(config)# reconfig 
LTE-8X-6(switch)(config)# exit
LTE-8X-6(switch)# exit
LTE-8X-6# 
```

```
# Для КАЖДОГО чипа установить параметры работы DHCP local relay
LTE-8X-6# olt 0
LTE-8X-6(OLT0)# set layer3 dhcp_sw_learning yes 
LTE-8X-6(OLT0)# set layer3 dhcp_relay_agent_opt82 yes
LTE-8X-6(OLT0)# set layer3 opt82_for_unicast_dhcp yes
LTE-8X-6(OLT0)# set layer3 trust_other_dhcp_relay_agent yes
LTE-8X-6(OLT0)# set layer3 overwrite_client_opt82 yes
LTE-8X-6(OLT0)# set layer3 maxlearnedclients 500
LTE-8X-6(OLT0)# set layer3 timerupdateinterval 16
LTE-8X-6(OLT0)# set layer3 opt82format binary_alt
LTE-8X-6(OLT0)# reconfigure 
OLT0 reconfiguration successfull. ONTs configuration may take several minutes.
LTE-8X-6(OLT0)# 
```

```
# Установить идентификатор хоста = 0 для корректной работы DHCP-парсера
LTE-8X-6# set host_id 0
```

```
# Сохранить все настройки
LTE-8X-6# save
```



### Настройка коммутатора Extreme
```
# Разрешить весь трафик на 40 порту (LTE-8X) в 11 влане (менеджмент), 208,211 – PPPoE, 250- Multicast
create access-list allowMcast " vlan-id 250 ;" " permit  ;" application "Cli"
create access-list allowMng " vlan-id 21 ;" " permit  ;" application "Cli"
create access-list allowPPPoE " vlan-id 208 ;" " permit  ;" application "Cli"
create access-list allowPPPoE_Hydra " vlan-id 221 ;" " permit  ;" application "Cli"
configure access-list add allowMng last priority 0 zone SYSTEM ports 40 ingress
configure access-list add allowPPPoE last priority 0 zone SYSTEM ports 40 ingress
configure access-list add allowPPPoE_Hydra last priority 0 zone SYSTEM ports 40 ingress
configure access-list add allowMcast last priority 0 zone SYSTEM ports 40 ingress


# Настроить правила работы с Option 82. В данном случае - сохранять значения опций,  
# передаваемые абонентскими коммутаторами
configure ip-security dhcp-snooping information option
configure ip-security dhcp-snooping information policy keep

# Включить механизм dhcp-snooping в стыковой влане на обоих uplink-портах
# Указать в качестве действия при нарушении - не реагировать
enable ip-security dhcp-snooping vlan Kolchugino_Subscribers port 42 violation-action none 
enable ip-security dhcp-snooping vlan Kolchugino_Subscribers port 48 violation-action none 

# Включить механизм dhcp-snooping в абонентских вланах на порту downlink
# Указать в качестве действия при нарушении - блокировку mac-адреса (!!! Не всего порта !!!)
enable ip-security dhcp-snooping vlan sub_access_vlan port 40 violation-action drop-packet block-mac duration 5

# Включить механизм изоляции IP-адресов на 40 порту (LTE-8X)
enable ip-security dhcp-snooping vlan sub_access_vlan port 40 violation-action drop-packet block-mac duration 5
```
*Дополнительно*
```
# При необходимости, разрешить весь трафик от абонентов, не использующих DHCP (по одному access-list на абонента)
create access-list allow_major1 " source-address 100.101.0.2/32 ;" " permit  ;" application "Cli"
configure access-list add allow_major1 first ports 40 ingress
```

## Диагностика

**Со стороны Extreme**
```
# Смотреть базу привязок dhcp. Привязки будут в супервлане, не в сабвланах
show ip-security dhcp-snooping entries "vsuper1"
```

```
# Смотреть разрешенные IP-адреса и соответствующие порты
show ip-security source-ip-lockdown
```

```
# Смотреть access-list'ы на порту
show access-list port 48
```

**Со стороны LTE-8X**
```
# Смотреть arp-таблицу на LTE-8X (ПО 3.16.0)
sh ip arp table
```

## Примечания
* Для безусловного разрешения VLAN или IP на портах, с включенной функцией ip-security source-ip-lockdown, следует использовать однострочные ACL, и не использовать policy. Поскольку приоритет правил однострочных ACL выше приоритета функции ip-security source-ip-lockdown, а приоритет правил, применённых через policy - ниже. Последние просто не будут работать.
* Следует разрешать трафик всех не-IPOE VLAN, т.е. управление, PPPoE, multicast
