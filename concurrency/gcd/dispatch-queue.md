# Dispatch queue
Очередь - это объект, который управляет отправленными в неё блоками кода. Блоки, 
отправленные в очередь, исполняются в порядке FIFO на пуле потоков, управляемым системой.  
Очереди могут быть последовательными и конкурентными.
>[!IMPORTANT]
> Кроме `main` очереди, которая использует главный поток, система не гарантирует, на каком потоке любая другая очередь будет выполнять блоки.
## Оглавление
- [Serial queue](#serial-queue)
- [Concurrent queue](#concurrent-queue)
- [Предоставляемые очереди](#предоставляемые-очереди)
- [Quality of service](#quality-of-service)
- [Создание своей очереди](#создание-своей-очереди)
- [Проблема избыточного создания потоков](#проблема-избыточного-создания-потоков)
## Serial queue
Последовательная очередь выполняeт блоки по одной в порядке FIFO.
```
┌────────┐┌──────────────┐┌──────────┐┌──────────────────┐
│ Блок 1 ││ Блок 2       ││ Блок 3   ││ Блок 4           │
└────────┘└──────────────┘└──────────┘└──────────────────┘
```
>[!IMPORTANT]
> Сериализация блоков работает только внутри одной очереди. Очереди исполняют блоки конкурентно по отношению друг к другу.
## Concurrent queue
### Описание
Конкурентная очередь так же выполняет блоки в порядке FIFO, но может начать выполнение 
следующих блоков до завершения уже запущенных.
```
┌────────┐
│ Блок 1 │
└────────┘
  ┌──────────────┐
  │ Блок 2       │
  └──────────────┘
      ┌──────────┐
      │ Блок 3   │
      └──────────┘
        ┌──────────────────┐
        │ Блок 4           │
        └──────────────────┘
```
Количество блоков одновременно выполняемых очередью, зависит от многих факторов
- Остальные процессорные ресурсы могут быть заняты другой работой.
- Другие блоки этой очереди могут быть ещё не готовы к выполнению.
- Другие блоки могут ждать блокировку.
- Выполнение может быть ограничено семафором.
- Сейчас выполняется барьерный блок.
- Для данной очереди в данный момент времени может быть доступен только один поток
- На планирование может влиять приоритет, текущая нагрузка, энергосбережение и температурные ограничения.
> [!IMPORTANT]
> Таким образом, конкурентная очередь вообще не гарантирует параллельное выполнение блоков.
## Предоставляемые очереди
GCD предоставляет
- главную последовательную очередь, выполняемую на главном потоке
  - `DispatchQueue.main`
- пять глобальных конкурентных очередей, различающихся качеством обслуживания
  - `DispatchQueue.global(qos: .userInteractive)`
  - `DispatchQueue.global(qos: .userInitiated)`
  - `DispatchQueue.global(qos: .default)`
  - `DispatchQueue.global(qos: .utility)`
  - `DispatchQueue.global(qos: .background)`
## Quality of service
Ядро учитывает QoS при выборе способа оптимального способа выполнения блока кода с учётом доступных ресурсов. 
| QoS | Приоритет | Ядра CPU | Назначение |
| --- | --- | --- | --- |
| userInteractive | Наивысший | P-Cores | **Интерфейс и анимации.** Задачи, результат которых нужен пользователю прямо в эту миллисекунду |
| userInitiated | Высокий | Преимущественно P-Cores | **Действия, инициированные пользователем.** Пользователь активно ждет результат (пара секунд). |
| default | Средний | P-Cores / E-Cores | **Дефолтный выбор.** Подходит для общей бизнес-логики приложения. |
| utility | Низкий | Преимущественно E-Cores | **Долгие вычисления.** Задачи, которые занимают от нескольких секунд до минут. |
| background | Минимальный | E-Cores | **Фоновая работа.** Пользователь вообще не знает, что эта задача выполняется. |
| unspecified | | | |
> [!Tip]
> Качество обслуживания `unspecified` в `libdispatch` мапится в `default`
> ```c
> qos == DISPATCH_QOS_UNSPECIFIED ? DISPATCH_QOS_DEFAULT : qos
> ```

Также на уровне ядра
- потокам с более высоким QoS отдаётся приоритет в получении процессорного времени,
- чтение/запись файлов с более высоким QoS в условиях конкуренции происходит быстрее,
- запросы к сети с более высоким QoS обрабатываются сетевым стеком ОС быстрее.
## Создание своей очереди
Происходит через конструктор 
```swift
DispatchQueue(label:qos:attributes:autoreleaseFrequency:target)
```
- `label` : `String`
  - Строка, присваиваемая очереди для её однозначной идентификации в инструментах отладки (Instruments, sample, stackshots) и краш репортах. Поскольку приложения, библиотеки и фреймворки могут создавать собственные очереди диспетчеризации (dispatch queues), рекомендуется использовать формат именования в стиле обратного DNS (например, com.example.myqueue).
- `qos` : `DispatchQoS`
  - [Качество обслуживания](#quality-of-service).
- `attributes` : `DispatchQueue.Attributes`
  - Атрибуты очереди
    - `concurrent`
      - Указывает, что очередь конкурентная, в случае отсутствия атрибута очередь инициализируется как последовательная.
    - `initiallyInactive`
      - Атрибут используется для предотвращение планирования блоков очередью до вызова метода очереди `activate()`
- `autoreleaseFrequency` : `DispatchQueue.AutoreleaseFrequency`
  - Частота автоматического освобождения объектов очередью
    - `inherit`
      - Наследуется из целевой очереди
    - `workItem`
      - Очередь настраивает autorelease пул перед выполнением блока и освобождает объекты в этом пуле после завершения выполнения блока.
    - `never`
      - Очередь не создает autorelease пул.
- `target` : `DispatchQueue?`
  - Целевая очередь на которой будут выполняться блоки.
> [!Tip]
> Если `target` очередь не указана (дефолтное значение аргумента `nil`) или явно передан `nil`, то в `libdispatch` присваивается глобальная очередь с соответствующим QoS.
## Проблема избыточного создания потоков
Apple в документации [DispatchQueue](https://developer.apple.com/documentation/dispatch/dispatchqueue#Avoiding-Excessive-Thread-Creation) пишет:

> При проектировании задач для параллельного выполнения избегайте вызова методов, блокирующих текущий поток выполнения. Когда блок, запланированный в конкурентной очереди, блокирует поток, система создает дополнительные потоки для выполнения других запланированных блоков. Если слишком много задач вызывают блокировку, у вашего приложения может закончиться доступный лимит потоков.
>
  > Еще одна причина чрезмерного потребления потоков приложением — создание слишком большого количества конкурентных очередей. Поскольку каждая очередь расходует ресурсы потоков, создание дополнительных конкурентных очередей усугубляет проблему их избыточного потребления. Вместо создания конкурентных очередей передавайте задачи в одну из глобальных. Для задач, требующих последовательного выполнения, назначайте целевой очередью для своей последовательной очереди одну из глобальных конкурентных очередей. Это позволит сохранить последовательный порядок выполнения задач, минимизировав при этом количество отдельных очередей, требующих создания потоков.

Если с первым абзацем вроде всё понятно:
- Блокируем поток.
- Ядро создаёт новый, чтобы программа продолжала работу.
- Доступные процессу потоки заканчиваются.

То вот второй выглядит как набор бессмысленных предложений:
- Почему создание новых очередей усугубляет какие-то проблемы?
- Почему надо передавать задачи в глобальные конкурентные очереди вместо своих конкурентных?
- Почему для последовательной очереди надо назначать глобальную, а не, например, другую последовательную?
- Почему при передаче в глобальную конкурентную очередь сохранится последовательный порядок выполнения?

Раз официальная документация Apple нам тут не поможет, придётся лезть в исходники `libdispatch`.

Создание очереди
```c
DISPATCH_NOINLINE
static dispatch_queue_t
_dispatch_lane_create_with_target(const char *label, dispatch_queue_attr_t dqa,
		dispatch_queue_t tq, bool legacy)
{
	dispatch_queue_attr_info_t dqai = _dispatch_queue_attr_to_info(dqa);

	//
	// Step 1: Normalize arguments (qos, overcommit, tq)
	//

	dispatch_qos_t qos = dqai.dqai_qos;
#if !HAVE_PTHREAD_WORKQUEUE_QOS
	if (qos == DISPATCH_QOS_USER_INTERACTIVE) {
		dqai.dqai_qos = qos = DISPATCH_QOS_USER_INITIATED;
	}
	if (qos == DISPATCH_QOS_MAINTENANCE) {
		dqai.dqai_qos = qos = DISPATCH_QOS_BACKGROUND;
	}
#endif // !HAVE_PTHREAD_WORKQUEUE_QOS

	_dispatch_queue_attr_overcommit_t overcommit = dqai.dqai_overcommit;
	if (overcommit != _dispatch_queue_attr_overcommit_unspecified && tq) {
		if (tq->do_targetq) {
			DISPATCH_CLIENT_CRASH(tq, "Cannot specify both overcommit and "
					"a non-global target queue");
		}
	}

	if (tq && dx_type(tq) == DISPATCH_QUEUE_GLOBAL_ROOT_TYPE) {
		// Handle discrepancies between attr and target queue, attributes win
		if (overcommit == _dispatch_queue_attr_overcommit_unspecified) {
			if (tq->dq_priority & DISPATCH_PRIORITY_FLAG_OVERCOMMIT) {
				overcommit = _dispatch_queue_attr_overcommit_enabled;
			} else {
				overcommit = _dispatch_queue_attr_overcommit_disabled;
			}
		}
		if (qos == DISPATCH_QOS_UNSPECIFIED) {
			qos = _dispatch_priority_qos(tq->dq_priority);
		}
		tq = NULL;
	} else if (tq && !tq->do_targetq) {
		// target is a pthread or runloop root queue, setting QoS or overcommit
		// is disallowed
		if (overcommit != _dispatch_queue_attr_overcommit_unspecified) {
			DISPATCH_CLIENT_CRASH(tq, "Cannot specify an overcommit attribute "
					"and use this kind of target queue");
		}
	} else {
		if (overcommit == _dispatch_queue_attr_overcommit_unspecified) {
			// Serial queues default to overcommit!
			overcommit = dqai.dqai_concurrent ?
					_dispatch_queue_attr_overcommit_disabled :
					_dispatch_queue_attr_overcommit_enabled;
		}
	}
	if (!tq) {
		tq = _dispatch_get_root_queue(
				qos == DISPATCH_QOS_UNSPECIFIED ? DISPATCH_QOS_DEFAULT : qos,
				overcommit == _dispatch_queue_attr_overcommit_enabled)->_as_dq;
		if (unlikely(!tq)) {
			DISPATCH_CLIENT_CRASH(qos, "Invalid queue attribute");
		}
	}

	//
	// Step 2: Initialize the queue
	//

	if (legacy) {
		// if any of these attributes is specified, use non legacy classes
		if (dqai.dqai_inactive || dqai.dqai_autorelease_frequency) {
			legacy = false;
		}
	}

	const void *vtable;
	dispatch_queue_flags_t dqf = legacy ? DQF_MUTABLE : 0;
	if (dqai.dqai_concurrent) {
		vtable = DISPATCH_VTABLE(queue_concurrent);
	} else {
		vtable = DISPATCH_VTABLE(queue_serial);
	}
	switch (dqai.dqai_autorelease_frequency) {
	case DISPATCH_AUTORELEASE_FREQUENCY_NEVER:
		dqf |= DQF_AUTORELEASE_NEVER;
		break;
	case DISPATCH_AUTORELEASE_FREQUENCY_WORK_ITEM:
		dqf |= DQF_AUTORELEASE_ALWAYS;
		break;
	}
	if (label) {
		const char *tmp = _dispatch_strdup_if_mutable(label);
		if (tmp != label) {
			dqf |= DQF_LABEL_NEEDS_FREE;
			label = tmp;
		}
	}

	dispatch_lane_t dq = _dispatch_object_alloc(vtable,
			sizeof(struct dispatch_lane_s));
	_dispatch_queue_init(dq, dqf, dqai.dqai_concurrent ?
			DISPATCH_QUEUE_WIDTH_MAX : 1, DISPATCH_QUEUE_ROLE_INNER |
			(dqai.dqai_inactive ? DISPATCH_QUEUE_INACTIVE : 0));

	dq->dq_label = label;
	dq->dq_priority = _dispatch_priority_make((dispatch_qos_t)dqai.dqai_qos,
			dqai.dqai_relpri);
	if (overcommit == _dispatch_queue_attr_overcommit_enabled) {
		dq->dq_priority |= DISPATCH_PRIORITY_FLAG_OVERCOMMIT;
	}
	if (!dqai.dqai_inactive) {
		_dispatch_queue_priority_inherit_from_target(dq, tq);
		_dispatch_lane_inherit_wlh_from_target(dq, tq);
	}
	_dispatch_retain(tq);
	dq->do_targetq = tq;
	_dispatch_object_debug(dq, "%s", __func__);
	return _dispatch_trace_queue_create(dq)._dq;
}
```
