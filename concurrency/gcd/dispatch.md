### Оглавление
- [Dispatch](#dispatch)
  - [Описание](#описание)
  - [Преимущества над ручным управлением потоками](#преимущества-над-ручным-управлением-потоками)
- [Очереди диспетчеризации](#очереди-диспетчеризации)
  - [Описание](#описание-1)
    - [Последовательная очередь](#последовательная-очередь)
    - [Конкурентная очередь](#конкурентная-очередь)
  - [Получение очереди](#получение-очереди)
  - [Качество обслуживания](#качество-обслуживания)
- [Механизмы очередей](#механизмы-очередей)
  - [Проблемы документации](#проблемы-документации)
  - [Различия глобальных и созданных очередей](#различия-глобальных-и-созданных-очередей)
  - [Граф очередей](#граф-очередей)
  - [Оверкоммитные очереди](#оверкоммитные-очереди)
  - [Оверкоммит лэйна и его глобальной очереди](#оверкоммит-лэйна-и-его-глобальной-очереди)
  - [Вызов async](#вызов-async)
    - [В конкурентном лэйне](#в-конкурентном-лэйне)
    - [В последовательном лэйне](#в-последовательном-лэйне)
    - [В глобальной очереди](#в-глобальной-очереди)
# Dispatch
### Описание
Кроссплатформенный фреймворк для управления многопоточностью и конкурентного выполнения задач. Берёт на себя 
- взаимодействие с планировщиком ядра операционной системы,
- динамическое управление пулом потоков,
- оптимизацию выполнения задач под архитектуру процессора устройства.

Вместо ручной работы с потоками разработчик взаимодействует с очередями.
### Преимущества над ручным управлением потоками
- Не нужно писать код для управления потоками, система сама занимается этим, справляясь эффективнее.
- Система может динамически масштабировать количество потоков в зависимости от доступных ресурсов и текущих системных условий.
- Система обычно начинает выполнение задачи быстрее, чем при ручном создании потока.
- Синхронизация доступа к ресурсу последовательной очередью эффективнее так как не требует вызова ядра.
# Очереди диспетчеризации
## Описание
> [Из документации Apple](https://developer.apple.com/documentation/dispatch/dispatchqueue#overview):  
> Очереди диспетчеризации — это FIFO очереди, в которые приложение может передавать задачи в виде объектов-блоков. Такие очереди выполняют задачи либо последовательно, либо конкурентно. Задачи, переданные в очереди, выполняются в пуле потоков, управляемом системой. За исключением очереди, представляющей главный поток приложения, система не гарантирует, какой именно поток будет использован для выполнения задачи.

Стоит запомнить, что очереди ничего сами не выполняют, это пассивная структура с двусвязным списком задач.
### Последовательная очередь
Блоки из последовательной очереди выполняются по одному в FIFO порядке.
```
┌────────┐┌──────────────┐┌──────────┐┌──────────────────┐
│ Блок 1 ││ Блок 2       ││ Блок 3   ││ Блок 4           │
└────────┘└──────────────┘└──────────┘└──────────────────┘
```
### Конкурентная очередь
Блоки из конкурентной очереди выполняются так же в порядке FIFO, но выполнение следующего блока может начаться до завершения выполнения предыдущего
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
## Получение очереди
В Swift есть три варианта получить объект очереди
1. Получить главную очередь вызовом `DispatchQueue.main`
   > [Из документации Apple](https://developer.apple.com/documentation/dispatch/dispatchqueue/main):  
   > Очередь диспетчеризации, связанная с основным потоком текущего процесса.
2. Получить одну из глобальных очередей вызовом `DispatchQueue.global(qos:)`
   > [Из документации Apple](https://developer.apple.com/documentation/dispatch/dispatchqueue/global(qos:)):  
   > Возвращает глобальную системную очередь с указанным классом качества обслуживания.
3. Создать свою последовательную или конкурентную вызовом `DispatchQueue.init(label:qos:attributes:autoreleaseFrequency:target:)`
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
   > [Из документации Apple](https://developer.apple.com/documentation/dispatch/dispatchqueue/init(label:qos:attributes:autoreleasefrequency:target:)):  
   > Создает новую очередь диспетчеризации, в которую можно передавать блоки.

> [!Tip]
> Если `target` очередь не указана (дефолтное значение аргумента `nil`) или явно передан `nil`, то в `libdispatch` присваивается глобальная очередь с соответствующим QoS.

## Качество обслуживания
Учитывается ядром при выборе способа оптимального способа выполнения блока кода с учётом доступных ресурсов. 
- Потоки аппаратно разделяются на асимметричных ядрах процессора.
- Потокам с более высоким QoS отдаётся приоритет в получении процессорного времени.
- Чтение/запись файлов с более высоким QoS в условиях конкуренции происходит быстрее.
- Запросы к сети с более высоким QoS обрабатываются сетевым стеком ОС быстрее.
- Потоки с более высоким QoS обрабатываются GPU внеочерёдно, чтобы сохранить FPS.
- В условиях тротлинга/энергосбережения ресурсы урезаются сначала у потоков с более низким QoS.
- XPC-сервисы наследуют QoS обратившегося к ним потока.

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
# Механизмы очередей
## Проблемы документации
В [документации Apple](https://developer.apple.com/documentation/dispatch/dispatchqueue#Avoiding-Excessive-Thread-Creation) указано избегать создания избыточных потоков:
> При проектировании задач для конкурентного выполнения избегайте вызова методов, блокирующих текущий поток. Когда задача, запланированная в конкурентной очереди, блокирует поток, система создает дополнительные потоки для выполнения других задач из очереди. Если слишком много задач вызывают блокировку, у вашего приложения может закончиться доступный лимит потоков.  
Еще одна причина чрезмерного потребления потоков приложением — создание слишком большого количества конкурентных очередей. Поскольку каждая очередь расходует ресурсы потоков, создание дополнительных конкурентных очередей усугубляет проблему их избыточного потребления. Вместо создания собственных конкурентных очередей передавайте задачи в одну из глобальных очередей.  
Для задач, требующих последовательного выполнения, назначайте целевой очередью для своей последовательной очереди одну из глобальных параллельных очередей. Это позволит сохранить последовательный порядок выполнения задач, минимизировав при этом количество отдельных очередей, требующих создания потоков.

Здесь много непонятных моментов:
- Почему создание новых очередей усугубляет какие-то проблемы? Зачем тогда вообще есть возможность создавать очереди?
- Почему надо передавать задачи в глобальные конкурентные очереди вместо своих конкурентных?
- Почему для последовательной очереди надо назначать глобальную, а не, например, другую последовательную?
- Почему при передаче в глобальную конкурентную очередь сохранится последовательный порядок выполнения?
## Различия глобальных и созданных очередей
Если посмотреть на их структуру в коде `libdispatch`:
- Глобальная очередь - это `dispatch_queue_global_s`
  ```c
  #define DISPATCH_QUEUE_ROOT_CLASS_HEADER(x) \
	  struct dispatch_queue_s _as_dq[0]; \
	  DISPATCH_QUEUE_CLASS_HEADER(x, \
		  	struct dispatch_object_s *volatile dq_items_tail); \
	  int volatile dgq_thread_pool_size; \ 
	  struct dispatch_object_s *volatile dq_items_head; \
	  int volatile dgq_pending
  
  struct dispatch_queue_global_s {
	  DISPATCH_QUEUE_ROOT_CLASS_HEADER(lane);
  } DISPATCH_CACHELINE_ALIGN;
  ```
  У неё есть:
  - ссылки на голову `dq_items_head` и хвост `dq_items_tail` двусвязного списка задач;
  - количество потоков очереди `dgq_thread_pool_size`;
  - количество задач, ожидающих выполнения `dgq_pending`.
- Созданная очередь - это `dispatch_lane_s`
  ```c
  #define DISPATCH_LANE_CLASS_HEADER(x) \
	  struct dispatch_queue_s _as_dq[0]; \
	  DISPATCH_QUEUE_CLASS_HEADER(x, \
		  	struct dispatch_object_s *volatile dq_items_tail); \
	  dispatch_unfair_lock_s dq_sidelock; \
	  struct dispatch_object_s *volatile dq_items_head; \
	  uint32_t dq_side_suspend_cnt

  typedef struct dispatch_lane_s {
	  DISPATCH_LANE_CLASS_HEADER(lane);
	  /* 32bit hole on LP64 */
  } DISPATCH_ATOMIC64_ALIGN *dispatch_lane_t;
  ```
  У неё есть:
  - ссылки на голову `dq_items_head` и хвост `dq_items_tail` двусвязного списка задач;
  - счётчик приостановки `dq_side_suspend_cnt`;
  - вспомогательная блокировка `dq_sidelock`.
> [!Note]
> Отсюда и далее я буду называть созданную очередь лэйном для удобства формулировок, разделения объектов и запоминания механизма.
## Граф очередей
Когда мы создаём лэйн в Swift, у нас есть аргумент `target`, в котором может быть глобальная очередь или лэйн. Это позволяет динамически конструировать граф очередей. Каждая очередь, включая лэйны, является нодой графа, а указатели на `target` выступают в роли направленного ребра, связывающего дочернюю очередь с родительской. В итоге терминальными нодами этого графа становятся глобальные очереди приложения.

Так как граф должен сохранять древовидную структуру без циклов, закольцевание двух лэйнов на самих себя приведёт к крашу. Кроме того, если оставить аргумент `target` с дефолтным значением `nil` или же явно передать туда `nil`, `libdispatch` присвоит лэйну глобальную очередь с соответствующим качеством обслуживания в качестве таргета.

## Оверкоммитные очереди
У ядра есть параметр `wq_max_threads`, задающий ограничение на максимальное количество потоков в пуле, доступном процессу. Предотвращает перегрев процессора и траты процессорного времени на переключение контекста потоков, и, как следствие, троттлинга. Динамически рассчитывается на основе количества логических ядер процессора, умноженного на коэффициент масштабирования.

Но в случае, если все потоки уже заблокированы на выполнении задач, то новым задачам просто негде выполняться. В этом случае можно игнорировать ограничение максимального числа потоков в жертву сохранению `forward progress` приложения. Для этого у приложения существует по паре глобальных очередей на каждое качество обслуживания:
```c
struct dispatch_queue_global_s _dispatch_root_queues[] = {
#define _DISPATCH_ROOT_QUEUE_IDX(n, flags) \
		((flags & DISPATCH_PRIORITY_FLAG_OVERCOMMIT) ? \
		DISPATCH_ROOT_QUEUE_IDX_##n##_QOS_OVERCOMMIT : \
		DISPATCH_ROOT_QUEUE_IDX_##n##_QOS)
#define _DISPATCH_ROOT_QUEUE_ENTRY(n, flags, ...) \
	[_DISPATCH_ROOT_QUEUE_IDX(n, flags)] = { \
		DISPATCH_GLOBAL_OBJECT_HEADER(queue_global), \
		.dq_state = DISPATCH_ROOT_QUEUE_STATE_INIT_VALUE, \
		.do_ctxt = _dispatch_root_queue_ctxt(_DISPATCH_ROOT_QUEUE_IDX(n, flags)), \
		.dq_atomic_flags = DQF_WIDTH(DISPATCH_QUEUE_WIDTH_POOL), \
		.dq_priority = flags | ((flags & DISPATCH_PRIORITY_FLAG_FALLBACK) ? \
				_dispatch_priority_make_fallback(DISPATCH_QOS_##n) : \
				_dispatch_priority_make(DISPATCH_QOS_##n, 0)), \
		__VA_ARGS__ \
	}
	_DISPATCH_ROOT_QUEUE_ENTRY(MAINTENANCE, 0,
		.dq_label = "com.apple.root.maintenance-qos",
		.dq_serialnum = 4,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(MAINTENANCE, DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.maintenance-qos.overcommit",
		.dq_serialnum = 5,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(BACKGROUND, 0,
		.dq_label = "com.apple.root.background-qos",
		.dq_serialnum = 6,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(BACKGROUND, DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.background-qos.overcommit",
		.dq_serialnum = 7,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(UTILITY, 0,
		.dq_label = "com.apple.root.utility-qos",
		.dq_serialnum = 8,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(UTILITY, DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.utility-qos.overcommit",
		.dq_serialnum = 9,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(DEFAULT, DISPATCH_PRIORITY_FLAG_FALLBACK,
		.dq_label = "com.apple.root.default-qos",
		.dq_serialnum = 10,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(DEFAULT,
			DISPATCH_PRIORITY_FLAG_FALLBACK | DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.default-qos.overcommit",
		.dq_serialnum = 11,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(USER_INITIATED, 0,
		.dq_label = "com.apple.root.user-initiated-qos",
		.dq_serialnum = 12,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(USER_INITIATED, DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.user-initiated-qos.overcommit",
		.dq_serialnum = 13,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(USER_INTERACTIVE, 0,
		.dq_label = "com.apple.root.user-interactive-qos",
		.dq_serialnum = 14,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(USER_INTERACTIVE, DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.user-interactive-qos.overcommit",
		.dq_serialnum = 15,
	),
};
```
Очереди с `DISPATCH_PRIORITY_FLAG_OVERCOMMIT` игнорируют `wq_max_threads` при запросе на создание нового потока. 
## Оверкоммит лэйна и его глобальной очереди
В Swift при получении глобальной очереди вызовом `DispatchQueue.global(qos:)` отдается её неоверкоммитный вариант. Возможности выставить флаг оверкоммита лэйну тоже нет. Поэтому, чтобы выяснить, как вообще лэйн может попасть в оверкоммитную очередь, следует обратиться к исходникам `libdispatch`. Нас интересует кусок кода создания лэйна.
```c
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
```
Получается, что если целевая очередь передана, она глобальная, и флаг оверкоммита лэйну не выставлен, то флаг оверкоммита берется из бита флага оверкоммита целевой очереди. А вот если целевая очередь не глобальная и не поток/ранлуп, то последовательная очередь всегда становится оверкоммитной. И если целевая очередь отсутствует,
```c
if (!tq) {
	tq = _dispatch_get_root_queue(
		qos == DISPATCH_QOS_UNSPECIFIED ? DISPATCH_QOS_DEFAULT : qos,
		overcommit == _dispatch_queue_attr_overcommit_enabled)->_as_dq;
		if (unlikely(!tq)) {
			DISPATCH_CLIENT_CRASH(qos, "Invalid queue attribute");
		}
	}
```
она получается из массива глобальных очередей с качеством обслуживания и флагом оверкоммита, соответствующим качеству обслуживания и флагу оверкоммита создаваемого лэйна.

Таким образом: 
- `DispatchQueue(label: "lane.serial.overcommit")` - оверкоммитный последовательный лэйн, глобальная очередь - оверкоммитная.
- `DispatchQueue(label: "lane.serial", target: .global())` - неоверкоммитный последовательный лэйн, глобальная очередь - неоверкоммитная.

Но возможен еще вариант с созданием промежуточного лэйна:
```swift
let intermediateLane = DispatchQueue(label: "lane.intermediate", target: .global())
let lane = DispatchQueue(label: "lane.serial.overcommit", target: intermediateLane)
```
В таком случае:
- `intermediateLane` - неоверкоммитный последовательный лэйн, глобальная очередь - неоверкоммитная.
- `lane` - оверкоммитный последовательный лэйн, глобальная очередь - неоверкоммитная.
## Вызов async 
При вызове `queue.async` в `libdispatch` вызывается `dispatch_async`
```c
#ifdef __BLOCKS__
void
dispatch_async(dispatch_queue_t dq, dispatch_block_t work)
{
	dispatch_continuation_t dc = _dispatch_continuation_alloc();
	uintptr_t dc_flags = DC_FLAG_CONSUME;
	dispatch_qos_t qos;

	qos = _dispatch_continuation_init(dc, dq, work, 0, dc_flags);
	_dispatch_continuation_async(dq, dc, qos, dc->dc_flags);
}
#endif
```
Из него вызывается `_dispatch_continuation_async`
```c
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_continuation_async(dispatch_queue_class_t dqu,
		dispatch_continuation_t dc, dispatch_qos_t qos, uintptr_t dc_flags)
{
#if DISPATCH_INTROSPECTION
	if (!(dc_flags & DC_FLAG_NO_INTROSPECTION)) {
		_dispatch_trace_item_push(dqu, dc);
	}
#else
	(void)dc_flags;
#endif
	return dx_push(dqu._dq, dc, qos);
}
```
`dx_push` это макрос, достающий метод `dq_push` из `vtable` очереди (да-да, табличная диспетчеризация):
```c
#define dx_push(x, y, z) dx_vtable(x)->dq_push(x, y, z)
```
### В конкурентном лэйне
В таблице `dq_push` соответствует методу `_dispatch_lane_concurrent_push`
```c
DISPATCH_NOINLINE
void
_dispatch_lane_concurrent_push(dispatch_lane_t dq, dispatch_object_t dou,
		dispatch_qos_t qos)
{
	// <rdar://problem/24738102&24743140> reserving non barrier width
	// doesn't fail if only the ENQUEUED bit is set (unlike its barrier
	// width equivalent), so we have to check that this thread hasn't
	// enqueued anything ahead of this call or we can break ordering
	if (dq->dq_items_tail == NULL &&
			!_dispatch_object_is_waiter(dou) &&
			!_dispatch_object_is_barrier(dou) &&
			_dispatch_queue_try_acquire_async(dq)) {
		return _dispatch_continuation_redirect_push(dq, dou, qos);
	}

	_dispatch_lane_push(dq, dou, qos);
}
```
Если нет хвоста очереди (очередь пуста), объект не ожидающий и не барьерный, и возможно выделить слот под выполнение из текущей очереди в методе `_dispatch_queue_try_acquire_async`
```c
DISPATCH_ALWAYS_INLINE DISPATCH_WARN_RESULT
static inline bool
_dispatch_queue_try_acquire_async(dispatch_lane_t dq)
{
	uint64_t old_state, new_state;

	return os_atomic_rmw_loop2o(dq, dq_state, old_state, new_state, acquire, {
		if (unlikely(!_dq_state_is_runnable(old_state) ||
				_dq_state_is_dirty(old_state) ||
				_dq_state_has_pending_barrier(old_state))) {
			os_atomic_rmw_loop_give_up(return false);
		}
		new_state = old_state + DISPATCH_QUEUE_WIDTH_INTERVAL;
	});
}
```
Вызывается `_dispatch_continuation_redirect_push`
```c
DISPATCH_NOINLINE
static void
_dispatch_continuation_redirect_push(dispatch_lane_t dl,
		dispatch_object_t dou, dispatch_qos_t qos)
{
	if (likely(!_dispatch_object_is_redirection(dou))) {
		dou._dc = _dispatch_async_redirect_wrap(dl, dou);
	} else if (!dou._dc->dc_ctxt) {
		// find first queue in descending target queue order that has
		// an autorelease frequency set, and use that as the frequency for
		// this continuation.
		dou._dc->dc_ctxt = (void *)
		(uintptr_t)_dispatch_queue_autorelease_frequency(dl);
	}

	dispatch_queue_t dq = dl->do_targetq;
	if (!qos) qos = _dispatch_priority_qos(dq->dq_priority);
	dx_push(dq, dou, qos);
}
```
В котором мы вызываем `dx_push` уже на целевой очереди лэйна. В противном случае вызывается `_dispatch_lane_push`, как и у последовательного лэйна.
### В последовательном лэйне
В таблице `dq_push` соответствует методу `_dispatch_lane_push`
```c
DISPATCH_NOINLINE
void
_dispatch_lane_push(dispatch_lane_t dq, dispatch_object_t dou,
		dispatch_qos_t qos)
{
	dispatch_wakeup_flags_t flags = 0;
	struct dispatch_object_s *prev;

	if (unlikely(_dispatch_object_is_waiter(dou))) {
		return _dispatch_lane_push_waiter(dq, dou._dsc, qos);
	}

	dispatch_assert(!_dispatch_object_is_global(dq));
	qos = _dispatch_queue_push_qos(dq, qos);

	// If we are going to call dx_wakeup(), the queue must be retained before
	// the item we're pushing can be dequeued, which means:
	// - before we exchange the tail if we have to override
	// - before we set the head if we made the queue non empty.
	// Otherwise, if preempted between one of these and the call to dx_wakeup()
	// the blocks submitted to the queue may release the last reference to the
	// queue when invoked by _dispatch_lane_drain. <rdar://problem/6932776>

	prev = os_mpsc_push_update_tail(os_mpsc(dq, dq_items), dou._do, do_next);
	if (unlikely(os_mpsc_push_was_empty(prev))) {
		_dispatch_retain_2_unsafe(dq);
		flags = DISPATCH_WAKEUP_CONSUME_2 | DISPATCH_WAKEUP_MAKE_DIRTY;
	} else if (unlikely(_dispatch_queue_need_override(dq, qos))) {
		// There's a race here, _dispatch_queue_need_override may read a stale
		// dq_state value.
		//
		// If it's a stale load from the same drain streak, given that
		// the max qos is monotonic, too old a read can only cause an
		// unnecessary attempt at overriding which is harmless.
		//
		// We'll assume here that a stale load from an a previous drain streak
		// never happens in practice.
		_dispatch_retain_2_unsafe(dq);
		flags = DISPATCH_WAKEUP_CONSUME_2;
	}
	os_mpsc_push_update_prev(os_mpsc(dq, dq_items), prev, dou._do, do_next);
	if (flags) {
		return dx_wakeup(dq, qos, flags);
	}
}
```
### В глобальной очереди
```c
DISPATCH_NOINLINE
void
_dispatch_root_queue_push(dispatch_queue_global_t rq, dispatch_object_t dou,
		dispatch_qos_t qos)
{
#if DISPATCH_USE_KEVENT_WORKQUEUE
	dispatch_deferred_items_t ddi = _dispatch_deferred_items_get();
	if (unlikely(ddi && ddi->ddi_can_stash)) {
		dispatch_object_t old_dou = ddi->ddi_stashed_dou;
		dispatch_priority_t rq_overcommit;
		rq_overcommit = rq->dq_priority & DISPATCH_PRIORITY_FLAG_OVERCOMMIT;

		if (likely(!old_dou._do || rq_overcommit)) {
			dispatch_queue_global_t old_rq = ddi->ddi_stashed_rq;
			dispatch_qos_t old_qos = ddi->ddi_stashed_qos;
			ddi->ddi_stashed_rq = rq;
			ddi->ddi_stashed_dou = dou;
			ddi->ddi_stashed_qos = qos;
			_dispatch_debug("deferring item %p, rq %p, qos %d",
					dou._do, rq, qos);
			if (rq_overcommit) {
				ddi->ddi_can_stash = false;
			}
			if (likely(!old_dou._do)) {
				return;
			}
			// push the previously stashed item
			qos = old_qos;
			rq = old_rq;
			dou = old_dou;
		}
	}
#endif
#if HAVE_PTHREAD_WORKQUEUE_QOS
	if (_dispatch_root_queue_push_needs_override(rq, qos)) {
		return _dispatch_root_queue_push_override(rq, dou, qos);
	}
#else
	(void)qos;
#endif
	_dispatch_root_queue_push_inline(rq, dou, dou, 1);
}
