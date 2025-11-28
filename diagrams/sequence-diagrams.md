ДИАГРАММА ПОСЛЕДОВАТЕЛЬНОСТИ

<img width="3452" height="8000" alt="image" src="https://github.com/user-attachments/assets/53faa3b8-bc38-4213-8409-4b4c6bbda7e4" />

@startuml
title Интеграция CRM WebStyle с MadeTask
actor "Менеджер" as Manager
actor "Подрядчик" as Contractor
actor "Бухгалтер" as Accountant
participant "CRM WebStyle" as CRM
participant "Система MadeTask" as MadeTask
participant "Email Сервис" as Email
database "База данных" as DB
participant "Банк подрядчика" as Bank

== Процесс создания аккаунта в CRM и отправки ссылки на регистрацию в MadeTask ==

Manager -> CRM: Заключает договор с подрядчиком
activate CRM

Manager -> CRM: Создает учетную запись подрядчика\nс ролью "Подрядчик"
CRM -> DB: Сохраняет данные подрядчика\n(email, ФИО, специализация, роль)
DB --> CRM: Подрядчик создан\n(статус: "registration")

CRM -> CRM: АВТОМАТИЧЕСКИ генерирует\nуникальную ссылку регистрации
Note over CRM: Ссылка: https://madetask.ru/signup?ref=webstyle&contractor_id=123

CRM -> Email: АВТОМАТИЧЕСКИ отправляет подрядчику email\nс ссылкой для регистрации в MadeTask
activate Email
Email -> Contractor: Email с темой:\n"Регистрация в платежной системе MadeTask"
Email --> CRM: Email отправлен успешно
deactivate Email

CRM -> DB: Обновляет статус: "registration_link_sent"
DB --> CRM: Статус обновлен
CRM --> Manager: Подтверждение создания аккаунта\nи отправки ссылки
deactivate CRM

== Регистрация подрядчика в MadeTask ==

Contractor -> MadeTask: Переходит по ссылке из email
activate MadeTask
MadeTask -> MadeTask: Определяет партнера (WebStyle)\nи contractor_id из URL
MadeTask -> Contractor: Отображает форму регистрации\nс предзаполненными данными

alt УСПЕШНАЯ РЕГИСТРАЦИЯ
  Contractor -> MadeTask: Заполняет данные\nи привязывает банковский счет
  MadeTask -> MadeTask: Выполняет KYC проверку\nи верификацию счета
  MadeTask --> Contractor: Регистрация завершена\n"Аккаунт готов к выплатам"
  
  MadeTask -> CRM: Webhook: POST /webhooks/madetask\n{\n  "event": "contractor.registered",\n  "contractor_id": "123",\n  "madetask_user_id": "mt_user_789",\n  "status": "verified"\n}
  activate CRM
  CRM -> DB: Обновляет статус подрядчика\nна "active", сохраняет madetask_user_id
  DB --> CRM: Данные обновлены
  CRM -> Manager: АВТОМАТИЧЕСКОЕ УВЕДОМЛЕНИЕ:\n"Подрядчик успешно зарегистрирован в MadeTask"
  CRM --> MadeTask: 200 OK
  deactivate CRM
  
else ОШИБКА РЕГИСТРАЦИИ
  Contractor -> MadeTask: Не может пройти верификацию
  MadeTask --> Contractor: "Требуется дополнительная\nверификация документов"
  Contractor -> Manager: Сообщает о проблемах\nс регистрацией
  Manager -> CRM: Проверяет статус регистрации
  activate CRM
  CRM --> Manager: Статус: "registration_link_sent"\n(ожидание регистрации)
  deactivate CRM
  Manager -> Contractor: Помогает решить проблему\nс верификацией
end

deactivate MadeTask

== Процесс создания и выполнения задачи ==

Manager -> CRM: Создает задачу в CRM
activate CRM
CRM -> CRM: Автоматически проверяет\nстатус подрядчика

alt ПОДРЯДЧИК АКТИВЕН
  CRM -> MadeTask: POST /api/v1/tasks\n(assignee_id: "mt_user_789", price, due_date)
  activate MadeTask
  MadeTask --> CRM: 201 Created\n(task_id: "mt_task_456")
  deactivate MadeTask
  
  CRM -> DB: Сохраняет задачу с external_task_id
  DB --> CRM: Задача сохранена
  CRM -> Contractor: АВТОМАТИЧЕСКОЕ УВЕДОМЛЕНИЕ:\n"Новая задача назначена"
  CRM --> Manager: Задача создана и синхронизирована
  
else ПОДРЯДЧИК НЕ АКТИВЕН
  CRM --> Manager: ОШИБКА: Подрядчик не зарегистрирован в MadeTask\nНеобходимо завершить регистрации
  Manager -> Contractor: Напоминает о необходимости\nрегистрации в MadeTask
end

deactivate CRM

Contractor -> CRM: Авторизуется в CRM\nс ограниченными правами
activate CRM
CRM -> Contractor: Отображает интерфейс подрядчика\n(только свои задачи)
Contractor -> CRM: Начинает работу над задачей
CRM -> DB: Обновляет статус задачи\nна "in_progress"
CRM -> MadeTask: PATCH /api/v1/tasks/mt_task_456\n{"status": "in_progress"}
MadeTask --> CRM: 200 OK
CRM --> Contractor: Статус обновлен

Contractor -> CRM: Загружает файлы с результатами
CRM -> DB: Сохраняет файлы
DB --> CRM: Файлы сохранены
CRM --> Contractor: Файлы успешно загружены

Contractor -> CRM: Отмечает задачу как выполненную
CRM -> DB: Обновляет статус задачи\nна "completed"
CRM -> MadeTask: PATCH /api/v1/tasks/mt_task_456\n{"status": "completed"}
MadeTask --> CRM: 200 OK
CRM -> Manager: Уведомление: "Задача выполнена\nготова к проверке"
CRM --> Contractor: Задача отмечена как выполненная
deactivate CRM

== Процесс приемки и выплаты ==

Manager -> CRM: Проверяет выполненную работу
activate CRM
CRM -> Manager: Отображает загруженные файлы\nи историю задачи
deactivate CRM

Manager -> CRM: Принимает работу, нажимает на "Произвести выплату"
activate CRM
CRM -> DB: Обновляет статус задачи\nна "accepted"

CRM -> MadeTask: POST /api/v1/tasks/mt_task_456/payout
activate MadeTask

alt УСПЕШНАЯ ВЫПЛАТА
  MadeTask -> Bank: Выполняет перевод
  Bank --> MadeTask: Перевод подтвержден
  MadeTask --> CRM: Платеж выполнен\n(payment_id: "mt_pay_123")
  deactivate MadeTask
  
  CRM -> DB: Сохраняет данные платежа
  DB --> CRM: Платеж сохранен
  CRM -> Accountant: Автоматическое уведомление о выплате\nдля учета в отчетности
  CRM -> Contractor: Уведомление о получении оплаты
