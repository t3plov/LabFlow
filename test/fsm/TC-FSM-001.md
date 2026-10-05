#### TC-FSM-001: Студент: NOT_STARTED → IN_PROGRESS

- **Приоритет:** Обязательный
- **Шаги:** Студент вызывает PATCH `/submissions/{id}/status` со `IN_PROGRESS`.
- **Ожидаемый результат:** 200, статус изменён, запись в StatusHistory.
