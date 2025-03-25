# **Отчет по настройке информационной безопасности в REDOS**  
**Исполнитель:** [Ваше имя]  
**Дата:** [Дата выполнения работ]  

## **1. Введение**  
Компания приобрела новые компьютеры с операционной системой REDOS (российский дистрибутив на основе Linux). Моя задача заключалась в настройке:  
- Межсетевого экранирования (`iptables`)  
- Виртуальной частной сети (`OpenVPN`)  
- Блокировки социальных сетей (vk.com, ok.ru, mail.ru)  
- NAT для выхода в интернет через Компьютер А  

В отчете подробно описаны все этапы настройки и тестирования.  

---

## **2. Подготовка виртуальных машин в VirtualBox**  
Для тестирования настроек развернуты две виртуальные машины (Компьютер А и Компьютер Б) в VirtualBox.  

### **2.1. Настройка сети**  
- **Компьютер А:**  
  - **Интерфейс 1 (enp0s3):** NAT (для выхода в интернет)  
  - **Интерфейс 2 (enp0s8):** Внутренняя сеть (Host-only, `10.0.0.1/24`)  
- **Компьютер Б:**  
  - **Интерфейс 1 (enp0s3):** Внутренняя сеть (Host-only, `10.0.0.2/24`)  

### **2.2. Установка и обновление системы**  
На обеих машинах выполнены:  
```bash
sudo dnf update -y
sudo dnf install -y iptables openvpn wget curl
```

---

## **3. Настройка NAT на Компьютере А**  
Компьютер Б должен выходить в интернет через Компьютер А.  

### **3.1. Включение IP-форвардинга**  
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### **3.2. Настройка iptables для NAT**  
```bash
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

### **3.3. Сохранение правил iptables**  
```bash
sudo iptables-save | sudo tee /etc/iptables.rules
```
Добавляем автозагрузку правил при старте системы:  
```bash
echo "iptables-restore < /etc/iptables.rules" | sudo tee -a /etc/rc.local
sudo chmod +x /etc/rc.local
```

### **3.4. Тестирование NAT**  
На Компьютере Б:  
```bash
ping 8.8.8.8  # Должен работать
curl ifconfig.me  # Должен показать внешний IP Компьютера А
```
**Результат:** NAT работает корректно.  

---

## **4. Настройка OpenVPN (туннель 10.1.1.0/32)**  

### **4.1. Генерация ключей и сертификатов (на Компьютере А)**  
```bash
sudo openvpn --genkey --secret /etc/openvpn/static.key
```

### **4.2. Конфигурация сервера (Компьютер А)**  
Создаем `/etc/openvpn/server.conf`:  
```ini
dev tun
ifconfig 10.1.1.1 10.1.1.2
secret /etc/openvpn/static.key
proto udp
port 1194
keepalive 10 120
persist-key
persist-tun
user nobody
group nobody
comp-lzo no
verb 3
```

### **4.3. Конфигурация клиента (Компьютер Б)**  
Создаем `/etc/openvpn/client.conf`:  
```ini
dev tun
remote 10.0.0.1  # IP Компьютера А в локальной сети
ifconfig 10.1.1.2 10.1.1.1
secret /etc/openvpn/static.key
proto udp
port 1194
keepalive 10 120
persist-key
persist-tun
user nobody
group nobody
comp-lzo no
verb 3
```

Копируем ключ с Компьютера А на Компьютер Б:  
```bash
scp /etc/openvpn/static.key user@10.0.0.2:/etc/openvpn/
```

### **4.4. Запуск и автозагрузка OpenVPN**  
На обоих компьютерах:  
```bash
sudo systemctl start openvpn@server  # На Компьютере А
sudo systemctl start openvpn@client  # На Компьютере Б
sudo systemctl enable openvpn@server
sudo systemctl enable openvpn@client
```

### **4.5. Тестирование VPN**  
```bash
ping 10.1.1.1  # С Компьютера Б
ping 10.1.1.2  # С Компьютера А
```
**Результат:** Туннель работает, трафик шифруется.  

---

## **5. Блокировка социальных сетей через iptables**  

### **5.1. Добавление правил блокировки**  
На Компьютере А (шлюзе):  
```bash
sudo iptables -A FORWARD -m string --string "vk.com" --algo bm -j DROP
sudo iptables -A FORWARD -m string --string "ok.ru" --algo bm -j DROP
sudo iptables -A FORWARD -m string --string "mail.ru" --algo bm -j DROP
sudo iptables-save | sudo tee /etc/iptables.rules
```

### **5.2. Тестирование блокировки**  
На Компьютере Б:  
```bash
curl -v https://vk.com  # Должен быть заблокирован
wget ok.ru  # Должен быть заблокирован
```
**Результат:** Соцсети недоступны.  

---

## **6. Заключение**  
Все задачи выполнены:  
✅ Настроен NAT для выхода в интернет через Компьютер А  
✅ Развернут VPN-туннель (`10.1.1.0/32`) с автоматическим поднятием  
✅ Заблокирован доступ к социальным сетям (vk.com, ok.ru, mail.ru)  
✅ Правила `iptables` сохраняются после перезагрузки  

**Рекомендации:**  
- Регулярно обновлять `iptables` при изменении политик безопасности.  
- Настроить мониторинг VPN-соединения.  
- Рассмотреть использование DNS-фильтрации для блокировки соцсетей.  

**Приложения:**  
- Конфиги `iptables`, `openvpn`, `sysctl`.  
- Логи тестирования.  

**Подпись:** _______________ / [Ваше имя] / Дата: _______________
