# Контрпример: контроль доступа на уровне API

> **Роль в лекции:** контрпример к [`ai_quick_auth_patch.md`](ai_quick_auth_patch.md), используется в блоке 6 сценария ([../script.md](../script.md)). Закрывает реальный путь атаки (открытый API), а не только его симптом (видимую ссылку на форму).

## Проверка прав на уровне API, а не только формы

```python
@app.route("/api/sources", methods=["POST"])
@require_role("data_administrator")   # проверка на уровне самого API
def register_source():
    data = request.get_json()
    source = SourceConfig(
        station_id=data["station_id"],
        source_type=data["source_type"],
        coefficient=data["coefficient"],
        registered_by=current_user.id,       # именная привязка действия
        registered_at=datetime.utcnow(),
    )
    validate_coefficient_range(source)        # наименьшие привилегии: ограниченный диапазон
    db.save_source_config(source)
    log_audit_event("source_registered", user=current_user.id, source=source.station_id)
    return {"status": "registered"}, 201
```

Ключевое отличие от AI-патча: проверка `@require_role` стоит на самом обработчике API, к которому обращается форма, — а не только на странице, через которую пользователь обычно до него добирается. Прямой запрос к `/api/sources`, минуя веб-форму, теперь тоже требует авторизации.

## Именные учётные записи вместо общего пароля

```python
def require_role(role: str):
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            token = request.headers.get("Authorization")
            user = verify_token(token)  # у каждого члена команды — свой токен
            if user is None or role not in user.roles:
                abort(403)
            g.current_user = user
            return fn(*args, **kwargs)
        return wrapper
    return decorator
```

Компрометация одной учётной записи не даёт доступа от имени всех — и позволяет отозвать доступ конкретного человека, не меняя общий секрет для всей команды.

## Ограничение диапазона — наименьшие привилегии на уровне данных

```python
def validate_coefficient_range(source: SourceConfig) -> None:
    """Коэффициент нормализации должен быть в разумных, заранее
    согласованных пределах — даже авторизованный администратор не
    может зарегистрировать источник с произвольным значением
    без дополнительного согласования."""
    if not (0.05 <= source.coefficient <= 1.0):
        raise ValueError(
            f"Коэффициент {source.coefficient} вне допустимого диапазона; "
            "для нестандартных значений требуется отдельное согласование"
        )
```

Это ограничивает последствия даже легитимной, но ошибочной или скомпрометированной учётной записи — коэффициент, близкий к нулю (как в исходной ситуации), не проходит валидацию в принципе, независимо от того, кто его пытается зарегистрировать.

## Журналирование действий

```python
def log_audit_event(action: str, user: str, source: str) -> None:
    audit_log.write({
        "action": action,
        "user": user,
        "source_id": source,
        "timestamp": datetime.utcnow().isoformat(),
    })
```

Каждая регистрация источника оставляет след: кто, когда, что именно зарегистрировал. Это не предотвращает злоупотребление само по себе, но делает его обнаружимым и расследуемым — то, чего полностью лишён вариант из [`ai_quick_auth_patch.md`](ai_quick_auth_patch.md).

---

## Сравнение с исходным патчем

| | `ai_quick_auth_patch.md` | Этот контрпример |
|---|---|---|
| Проверка прав | Только на HTML-странице | На самом API-эндпоинте |
| Учётные данные | Один пароль на всех, в коде | Именные токены |
| Ограничение действия | Нет — любой авторизованный может всё | Диапазон коэффициента ограничен |
| Журналирование | Отсутствует | Каждое действие с привязкой к пользователю |
| Прямой запрос к API в обход формы | По-прежнему открыт | Требует авторизации |
