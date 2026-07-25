# Контрпример: изоляция и деградация вместо ретраев без ограничений

> **Роль в лекции:** контрпример к [`ai_incident_patch.md`](ai_incident_patch.md), используется в блоке 5 сценария ([../script.md](../script.md)). Решает ту же задачу (нестабильная станция не должна валить всё задание) через изоляцию, ограниченные повторные попытки и явную деградацию.

## Изоляция: сбой одного района не прерывает обработку остальных

```python
def run_daily_collection(district_ids: list[str]) -> CollectionReport:
    results = {}
    for district_id in district_ids:
        try:
            results[district_id] = collect_district(district_id)
        except DistrictCollectionFailed as exc:
            results[district_id] = DegradedResult(reason=str(exc))
            log_and_alert(f"Район {district_id} обработан с деградацией: {exc}")
            continue  # переходим к следующему району, а не падаем целиком
    return CollectionReport(results)
```

Исключение в обработке одного района ловится на уровне цикла — остальные районы обрабатываются независимо от исхода этого.

## Ограниченные, идемпотентные повторные попытки с задержкой

```python
def fetch_station_data(station_id: str, max_attempts: int = 3) -> list[Measurement]:
    delay = 2.0  # секунды, растёт экспоненциально
    for attempt in range(1, max_attempts + 1):
        try:
            response = station_api.get(station_id, request_id=make_idempotency_key(station_id))
            return parse_measurements(response)
        except StationAPIError:
            if attempt == max_attempts:
                raise DistrictCollectionFailed(f"станция {station_id} недоступна после {max_attempts} попыток")
            time.sleep(delay)
            delay *= 2  # экспоненциальный backoff: 2s, 4s, 8s
```

```python
def store_measurements(station_id: str, data: list[Measurement]) -> None:
    # upsert по (station_id, measured_at) — повторная запись того же
    # периода не создаёт дубликат, а обновляет существующую запись.
    for m in data:
        db.upsert_measurement(key=(m.station_id, m.measured_at), value=m)
```

- `max_attempts=3` и растущая задержка не позволяют превратить временный сбой источника в бесконечную нагрузку на него.
- `request_id` (ключ идемпотентности) и `upsert` вместо простой вставки делают повторную попытку безопасной: даже если ответ был потерян после успешной обработки на сервере, повторный запрос не создаёт задвоенную запись.

## Явная деградация вместо тишины

```python
def collect_district(district_id: str) -> DistrictResult:
    partial = {}
    for station_id in get_stations(district_id):
        try:
            partial[station_id] = fetch_station_data(station_id)
        except DistrictCollectionFailed:
            partial[station_id] = None  # источник недоступен — явно, не молча
    if all(v is None for v in partial.values()):
        raise DistrictCollectionFailed(f"все источники района {district_id} недоступны")
    return DistrictResult(partial, degraded=any(v is None for v in partial.values()))
```

Отчёт, отправляемый специалисту и общественному совету, включает явную пометку: «данные станции S-N1 недоступны с 19:40, показан последний известный результат» — вместо молчаливого отсутствия обновления или полного отказа всего отчёта.

## Мониторинг зависания, а не только исключений

```python
def check_collection_freshness() -> None:
    for district_id, last_success in get_last_successful_collection_times().items():
        if time.time() - last_success > 3600:  # SLO: не более часа без обновления
            alert(f"Район {district_id}: данные не обновлялись более часа")
```

Этот код прямо продолжает наблюдаемость из лекции 7 — но теперь ловит не только явную ошибку выполнения, а нарушение SLO по свежести данных, включая случай зависшего, а не упавшего задания.

---

## Сравнение с исходным патчем

| | `ai_incident_patch.md` | Этот контрпример |
|---|---|---|
| Сбой одной станции | Останавливает все районы | Изолирован, остальные обрабатываются |
| Повторные попытки | Без ограничения и задержки | 3 попытки, экспоненциальный backoff |
| Идемпотентность | Не рассмотрена, риск задвоения | Ключ идемпотентности + upsert |
| Видимость проблемы | Мониторинг молчит (задание не падает, а висит) | Алерт по нарушению SLO свежести данных |
| Результат для пользователя | Полное отсутствие обновления | Явная пометка деградации, остальные районы актуальны |
