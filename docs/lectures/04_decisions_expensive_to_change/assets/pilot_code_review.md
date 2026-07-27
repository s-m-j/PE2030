# Код пилота и его ревью: команда пишет сама, AI проверяет

> **Роль в лекции:** показывается в блоке 1 сценария ([../script.md](../script.md)) в двух частях — сначала код (без ревью), затем, после реакции зала, ревью AI. Замещает прежний пример «AI написал код с ошибкой»: здесь код пишет команда, а AI выступает рецензентом.

## Часть 1 — код, написанный командой (показывается в блоке 1)

К пятнице нужен рабочий пилот. Один из участников команды пишет код сопоставления сам, быстро:

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

Приложенный тест:

```python
def test_process_complaints_smoke():
    complaint = Complaint("C1", "ул. Садовая, 12", datetime(2026, 6, 1, 19, 0))
    measurements = [
        Measurement("S4", "PM2.5", 14.0, "µg/m³", datetime(2026, 6, 1, 19, 0)),
    ]
    result = process_complaints([complaint], measurements)
    assert result[0]["status"] == "обычный уровень"
```

Тест проходит. Код читается легко, назван понятно, использует типизацию и dataclasses. Демонстрация в пятницу проходит гладко.

---

## Часть 2 — ревью AI (показывается после реакции зала в блоке 2)

Перед показом результата специалисту команда просит AI сделать ревью кода на предмет скрытых архитектурных предположений.

> **Ответ AI:** «В функции `to_comparable_scale` есть строка, которая заслуживает отдельного внимания: `return measurement.value * 0.12`. Это единственное место во всём коде, которое определяет, при каком значении система сочтёт воздух опасным (порог `35.0` в `classify` сравнивается уже с результатом этого пересчёта). Коэффициент `0.12` не сопровождается ссылкой на источник — ни в коде, ни в комментарии, ни в тесте. Приложенный тест `test_process_complaints_smoke` ни разу не вызывает ветку с `index_0_500` — то есть не покрывает как раз то место, где принято самое рискованное предположение. Если этот коэффициент неверен, ошибка будет воспроизводиться в каждой классификации, использующей датчик VOC, и ничем не будет отмечена как приближение.
>
> Я не могу сказать, верен ли коэффициент `0.12` — это вопрос предметной области, а не кода. Рекомендую явно зафиксировать это как гипотезу, требующую проверки специалистом, а не как техническую деталь нормализации».

---

## Материал для разбора

- **Код написан человеком, а не AI** — это не повторение примера лекции 1. Ошибка, если коэффициент действительно неверен, — не «AI подвело», а обычное следствие спешки, которое могло произойти при любом способе написания кода.
- **AI обнаружило архитектурно значимое предположение, введённое человеком**, — не создало и не спрятало проблему, а указало на неё, включая то, что тест не покрывает рискованную ветку.
- **AI прямо назвало предел своей компетенции**: «я не могу сказать, верен ли коэффициент — это вопрос предметной области». Оно не пытается изобразить экспертизу, которой у него нет, и явно передаёт вопрос человеку — специалисту экологической службы.
- Сравните с [`adr_data_normalization.md`](adr_data_normalization.md): там то же архитектурное решение принято явно с самого начала. Здесь оно сначала принято неявно и обнаружено только благодаря ревью — стоит спросить, что случилось бы, если бы ревью не попросили вовсе.

Вопрос для обсуждения: как обнаруженный коэффициент следует классифицировать — как инженерную деталь нормализации или как предметную гипотезу, требующую подтверждения специалистом, — и кто вправе это подтвердить?
