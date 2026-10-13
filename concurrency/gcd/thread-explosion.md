# Проблема избыточного создания потоков
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
## DISPATCH_PRIORITY_FLAG_OVERCOMMIT
Был введен в 2014 году вместе с вводом системы `Quality of Service`.  
До `QoS` у глобальных очередей был `priority`, а кастомные очереди создавались без подобного параметра вообще, он регулировался целевой очередью.
Это порождало некоторые проблемы:
- Глобальный `thread starvation`. Если низкоприоритетные задачи занимали весь пул потоков, высокоприоритетная задача не могла пробить этот затор, так как ядро не умело динамически повышать приоритеты потоков и перекидывать их на другие ядра.
- Ядро не могло на основе приоритетов раскидывать задачи между новыми процессорами с P- и E-ядрами. 

`overcommit` позволяет очереди создавать больше максимально определенного системой количества потоков `wq_max_threads`. Это сделано ради того, чтобы если один модуль приложения забьет все доступные потоки, второй модуль не повис намертво, а мог продолжать работу, пусть и ценой дополнительной нагрузки на процессор. 
### Создание очереди
Нас интересует кусок кода c обработкой расхождений между атрибутами очереди и целевой очередью
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
Здесь можно заметить, что если есть целевая очередь `tq`, и она корневая (только глобальные очереди имеют тип `DISPATCH_QUEUE_GLOBAL_ROOT_TYPE`), то если мы передаем флаг `overcommit` как `_dispatch_queue_attr_overcommit_unspecified`, то этот флаг наследуется от целевой очереди.  
А если целевая очередь не является глобальной, то свойство `overcommit` устанавливается отключенным `_dispatch_queue_attr_overcommit_disabled` для конкурентных очередей или включенным `_dispatch_queue_attr_overcommit_enabled` для последовательных.

Это объясняет тезис 
> Для задач, требующих последовательного выполнения, назначайте целевой очередью для своей последовательной очереди одну из глобальных конкурентных очередей. Это позволит сохранить последовательный порядок выполнения задач, минимизировав при этом количество отдельных очередей, требующих создания потоков.

Если мы создаём последовательную очередь и не указываем ей целевую, то последовательная очередь становится оверкоммитной.

В приложении есть по две глобальных очереди на каждый `QoS`, нон-оверкоммитная и оверкоммитная.
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
И целевая очередь в таком случае будет выбрана с `QoS` и флагом `overcommit` создаваемой очереди.
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
