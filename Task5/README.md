# Проектирование GraphQL API для сервиса client-info

## 1. Анализ REST API

Из Swagger-документации видно, что REST API сервиса client-info имеет следующие ключевые ресурсы:

- `/clients/{id}` — получения информации о клиенте (включая базовые поля).
- `/clients/{id}/documents` — список документов клиента.
- `/clients/{id}/relatives` — список родственников клиента.

Также присутствуют определения сущностей:

### Client

- id
- name
- age

### Document

- id
- type
- number
- issueDate
- expiryDate

### Relative

- id
- relationType
- name
- age

## 2. Проектирование GraphQL-схемы

Основная цель — устранить необходимость множественных запросов к API и позволить клиентам запрашивать только нужные данные. Для этого создадим типы и запросы, отражающие структуру данных и логику взаимодействия.

### Определение типов (Types)

```graphql
type Client {
  id: ID!
  name: String!
  age: Int
  documents: [Document!]!
  relatives: [Relative!]!
}

type Document {
  id: ID!
  type: String!
  number: String!
  issueDate: String
  expiryDate: String
}

type Relative {
  id: ID!
  relationType: String!
  name: String!
  age: Int
}
```

### Запросы (Queries)

Для полного покрытия функциональности REST API, достаточно одного запроса `client`, который позволяет получить все данные клиента с возможностью выбора подмножества полей:

```graphql
type Query {
  client(id: ID!): Client
}
```

Таким образом, клиенты смогут делать запросы вида:

```graphql
query GetClient($id: ID!) {
  client(id: $id) {
    id
    name
    age
    documents {
      type
      number
    }
    relatives {
      name
      relationType
    }
  }
}
```

## 3. Преимущества GraphQL над REST API

- **Уменьшение количества запросов**: вместо 3-х вызовов REST (client, documents, relatives) — один запрос GraphQL.
- **Устранение over-fetching**: клиент получает только то, что запрашивает.
- **Гибкость**: потребители могут легко расширять или изменять запросы без изменения API.
- **Меньшая нагрузка на сервер**: снижается количество HTTP-запросов и, соответственно, RPS.

## Заключение

GraphQL-схема позволяет эффективно управлять сложной структурой данных клиента, устраняя проблему дублирования и избыточных вызовов. Это оптимизирует взаимодействие между веб-приложениями и core-app с сервисом client-info.