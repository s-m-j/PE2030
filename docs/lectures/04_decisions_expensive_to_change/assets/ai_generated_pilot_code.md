# Предложение AI: реализация пилотного инкремента

> **Роль в лекции:** показывается в блоке 1 сценария ([../script.md](../script.md)) как готовый результат — без комментариев. Разбор скрытого решения открывается только после того, как зал сам попробует его найти (блок 2).

## Промпт

```
Реализуй сопоставление жалоб жителей с показаниями датчиков и официальных
станций за последний месяц, как описано в требованиях пилотного этапа.
Данные: официальные станции измеряют PM2.5 в µg/m³ (часовое среднее);
недорогие датчики измеряют VOC-индекс в условных единицах 0–500
(мгновенное значение). Нужно вывести для каждой жалобы ближайшее по
времени и месту показание и статус "обычный уровень / отклонение".
```

## Ответ AI (как показывается студентам)

```python
from dataclasses import dataclass
from datetime import datetime, timedelta


@dataclass
class Measurement:
    station_id: str
    pollutant: str
    value: float
    unit: str
    measured_at: datetime


@dataclass
class Complaint:
    complaint_id: str
    location: str
    reported_at: datetime


def find_nearest_measurement(complaint: Complaint, measurements: list[Measurement]) -> Measurement | None:
    candidates = [m for m in measurements if abs((m.measured_at - complaint.reported_at)) < timedelta(hours=2)]
    if not candidates:
        return None
    return min(candidates, key=lambda m: abs(m.measured_at - complaint.reported_at))


def to_comparable_scale(measurement: Measurement) -> float:
    """Приводит показание к единой сопоставимой шкале для сравнения."""
    if measurement.unit == "µg/m³":
        return measurement.value
    if measurement.unit == "index_0_500":
        # Грубое приведение индекса датчика к эквиваленту PM2.5
        return measurement.value * 0.12
    raise ValueError(f"Неизвестная единица измерения: {measurement.unit}")


def classify(complaint: Complaint, measurements: list[Measurement]) -> str:
    nearest = find_nearest_measurement(complaint, measurements)
    if nearest is None:
        return "недостаточно данных"
    comparable_value = to_comparable_scale(nearest)
    if comparable_value > 35.0:
        return "отклонение"
    return "обычный уровень"


def process_complaints(complaints: list[Complaint], measurements: list[Measurement]) -> list[dict]:
    return [
        {
            "complaint_id": c.complaint_id,
            "status": classify(c, measurements),
        }
        for c in complaints
    ]
```

### Тест, приложенный к ответу

```python
def test_process_complaints_smoke():
    complaint = Complaint("C1", "ул. Садовая, 12", datetime(2026, 6, 1, 19, 0))
    measurements = [
        Measurement("S4", "PM2.5", 14.0, "µg/m³", datetime(2026, 6, 1, 19, 0)),
    ]
    result = process_complaints([complaint], measurements)
    assert result[0]["status"] == "обычный уровень"
```

Тест проходит. Код читается легко, назван понятно, использует типизацию и dataclasses.

---

## Что скрыто в этом коде (не показывать до блока 2 — материал для разбора)

Код работает, проходит приложенный тест и делает именно то, что было явно попрошено. Архитектурно значимое решение находится в функции `to_comparable_scale`:

```python
if measurement.unit == "index_0_500":
    return measurement.value * 0.12
```

- **Коэффициент `0.12` не обоснован нигде** — ни в коде, ни в комментарии, ни в тесте. Он появился как «правдоподобное на вид» число, позволяющее получить результат в похожем диапазоне значений, что и PM2.5.
- **Решение зашито внутри функции классификации**, а не оформлено как отдельное, явно названное и обсуждаемое предположение. Тест проверяет только случай с официальной станцией — ни разу не вызывает ветку с датчиком VOC, то есть не покрывает как раз то место, где принято самое рискованное решение.
- **Граница между источником данных и логикой сравнения не проведена.** Если завтра появится второй недорогой датчик с другой шкалой или другим веществом, придётся руками добавлять ещё одну ветку `if` внутри той же функции — изменение источника данных требует правки логики сравнения.
- **Ошибка в коэффициенте будет воспроизводиться в каждой классификации**, использующей датчик VOC, и никак не будет видна снаружи: код не логирует и не помечает, что для этой жалобы использовалось грубое приближение, а не прямое измерение того же вещества, что и на официальной станции.
- **Это ровно тот класс проблемы, что в лекции 1**: убедительная, технически исправная панель, которая молча предполагает сопоставимость двух разных измеряемых величин. Здесь то же предположение просто перенесено с интерфейса внутрь кода — и от этого не стало более обоснованным, только менее заметным.

Сравните с явным подходом к тому же решению в [`adr_data_normalization.md`](adr_data_normalization.md).
