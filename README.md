# Планирование и анализ

[Задание 1.1](diagrams/section-1-1/README.md)

[Задание 1.2 - 1.3](diagrams/section-1-2/README.md)

[Задание 1.4](api/README.md)

# Задание 2.1

## Smart Home Monolith

### Описание

Проект “Smart Home Monolith” представляет собой монолитное приложение для управления отоплением и мониторинга температуры в умном доме. Пользователи могут удаленно включать/выключать отопление, устанавливать желаемую температуру и просматривать текущую температуру через веб-интерфейс.

В дополнение к монолиту, проект включает два микросервиса:

* сервис управления устройствами (device-management-service): отвечает за обработку команд управления устройствами через Kafka. 
* сервис телеметрии (telemetry-service): слушает Kafka-топик sensor-data и обрабатывает данные телеметрии от датчиков.

Сервисы реализованы в упрощённом формате, в целевом решении они взаимодействуют с MQTT брокером для получения телеметрии и отправки команд устройствам, с kafka для взаимодейтствия между собой и другими сервисами.

Монолит на текущий момент не работает с Kafka, в целевой схеме потребуется его доработка, чтобы он начал взаимодействовать через Kafka с другими сервисами.

Все компоненты взаимодействуют с общими зависимостями, такими как PostgreSQL, Zookeeper и Kafka, управляемыми с помощью Docker Compose.

### Пререквизиты

* docker
* docker compose

### Структура проекта

Проект организован в соответствии со стандартной структурой Maven проекта и включает следующие модули:

	•	smart-home-monolith/ - монолитное приложение.
	•	device-management/ - сервис управления устройствами.
	•	telemetry-service/ - сервис обработки телеметрии.
	•	compose.yml - файл Docker Compose для оркестрации всех сервисов.
	•	init.sql - скрипт инициализации баз данных.

### Проверяем работоспособность

```bash
git clone https://github.com/denrzv/yp-architecture-sprint-3.git
cd yp-architecture-sprint-3
git checkout sprint_3
```

Запуск сервисов:

```bash
docker compose up -d && docker compose logs -f
```

После запуска всех контейнеров выполняем создание устройства:

```bash
curl -v POST http://localhost:8081/api/devices \
     -H "Content-Type: application/json" \
     -d '{
           "name": "Thermostat Living Room",
           "type": "THERMOSTAT",
           "metadata": {
             "location": "Living Room",
             "model": "T1000"
           }
         }'
```

Пример ответа:

```bash
{"id":"79296af8-7908-471e-8b3c-663d8ab44497","name":"Thermostat Living Room","status":null,"type":"THERMOSTAT","metadata":{"location":"Living Room","model":"T1000"}}
```

в логе сервиса

```bash
device-management-service  | 2024-10-01T09:37:03.938Z  INFO 1 --- [device-management] [nio-8080-exec-1] r.y.p.s.d.service.DeviceService          : Device created: Device(id=null, name=Thermostat Living Room, status=null, type=THERMOSTAT, metadata={location=Living Room, model=T1000})
```

Изменение статуса, эмулируем сообщение от монолита:

```bash
 echo '{"deviceId":"79296af8-7908-471e-8b3c-663d8ab44497","command":"turn_on"}' | docker exec -i kafka kafka-console-producer.sh --bootstrap-server localhost:9092 --topic device_commands
```

в логе сервиса считывание статуса:

```bash
device-management-service  | 2024-10-01T09:38:39.847Z  INFO 1 --- [device-management] [ntainer#0-0-C-1] r.y.p.s.d.service.DeviceService          : Processing command: DeviceCommand(deviceId=908a9a99-b193-4aee-b966-e0d885ba3f2a, command=turn_on, parameters=null)
```

Эмулируем публикацию телеметрии от монолита:

```bash
echo '{"deviceId": "79296af8-7908-471e-8b3c-663d8ab44497", "temperature": 25.5}' | docker exec -i kafka kafka-console-producer.sh --bootstrap-server localhost:9092 --topic sensor_data
```

сервис телеметрии считывает новые данные и сохраняет в базу:

```bash
telemetry-service  | 2024-10-01T09:56:20.152Z  INFO 1 --- [telemetry-service] [ntainer#0-0-C-1] r.y.p.s.t.kafka.TelemetryDataListener    : Received telemetry data: TelemetryDataDTO(deviceId=79296af8-7908-471e-8b3c-663d8ab44497, temperature=25.5)
```

### Выключение

```bash
docker compose down -v
```
