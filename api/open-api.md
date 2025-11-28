openapi: 3.0.3
info:
  title: CRM WebStyle - MadeTask Integration API
  version: 1.0.0

servers:
  - url: https://api.webstyle.ru/v1

paths:
  /contractors:
    post:
      summary: Создание подрядчика
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [email, full_name, specialization]
              properties:
                email: { type: string, format: email }
                full_name: { type: string }
                specialization: { type: string, enum: [designer, copywriter, developer] }
                phone: { type: string }
      responses:
        '201':
          description: Подрядчик создан
          content:
            application/json:
              schema:
                type: object
                properties:
                  id: { type: string }
                  registration_url: { type: string }
                  status: { type: string }

  /contractors/{contractorId}/tasks:
    post:
      summary: Создание задачи
      parameters:
        - name: contractorId
          in: path
          required: true
          schema: { type: string }
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [title, price, due_date]
              properties:
                title: { type: string }
                description: { type: string }
                price: { type: number, minimum: 0 }
                due_date: { type: string, format: date }
                project_id: { type: string }
      responses:
        '201':
          description: Задача создана
        '402':
          description: Подрядчик не зарегистрирован в MadeTask

  /tasks/{taskId}/accept:
    post:
      summary: Принять работу и инициировать выплату
      parameters:
        - name: taskId
          in: path
          required: true
          schema: { type: string }
      responses:
        '200':
          description: Выплата инициирована
        '402':
          description: Ошибка выплаты

  /reports/payments:
    get:
      summary: Отчет по выплатам
      parameters:
        - name: date_from
          in: query
          schema: { type: string, format: date }
        - name: date_to
          in: query
          schema: { type: string, format: date }
      responses:
        '200':
          description: Отчет сгенерирован
          content:
            application/json:
              schema:
                type: object
                properties:
                  data: { type: array, items: { type: object } }
                  excel_url: { type: string }

  /webhooks/madetask:
    post:
      summary: Webhook от MadeTask
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required: [event, data]
              properties:
                event: { type: string, enum: [contractor.registered, payment.completed, payment.failed] }
                data: { type: object }
      responses:
        '200':
          description: Webhook обработан

components:
  securitySchemes:
    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key

security:
  - ApiKeyAuth: []
