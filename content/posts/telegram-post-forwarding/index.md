---
title: "TeleMirror: ретранслятор Telegram-каналов"
date: 2025-12-23
description: "Создание собственного ретранслятора Telegram-каналов с фильтрами и настройками пересылки"
summary: "Создание собственного ретранслятора Telegram-каналов с фильтрами и настройками пересылки"
categories: blog
tags: ["telegram", "python", "telethon", "bot", "forwarding", "mirroring"]
draft: false
images:
  - "cover.png"
featured_image: "cover.png"
---

Еще в 2020 году написал статью на vc.ru ["Создание и развертывание ретранслятора Telegram каналов, используя Python и Heroku"](https://vc.ru/dev/158757-sozdanie-i-razvertyvanie-retranslyatora-telegram-kanalov-ispolzuya-python-i-heroku). С тех пор, Heroku уже не предлагает бесплатный тариф. Но проект жив и обновляется.

Репозиторий проекта: https://github.com/khoben/telemirror

Собственно, нет ничего сверхъестественного в пересылке: получаем сообщение из канала-источника и отправляем его копию в канал-приёмник.

В простейшем случае это выглядит так:
```python
from telethon import TelegramClient, events

client = TelegramClient("telemirror", api_id, api_hash)
SOURCE_CHANNELS = [-10001, -10002] # id каналов
TARGET_CHANNELS = [-10003, -10004]

@client.on(events.NewMessage(chats=SOURCE_CHANNELS))
async def handler(event):
    for target in TARGET_CHANNELS:
        await client.send_message(target, event.message)

client.start()
client.run_until_disconnected()
```

Не хватает лишь хоть какой-то конфигурации и гибкости. Какие сообщения отбрасывать, изменять оригинальный текст, из какого канала в какой пересылать и т.д. 

К простой схеме, описанной выше, добавляются модификаторы, которые можно назвать фильтрами.

В [TeleMirror](https://github.com/khoben/telemirror) простейший фильтр для сообщений выглядит так:
```python
class MessageFilter(Protocol):
    async def process(
        self, entity: EventEntity, event_type: Type[EventLike]
    ) -> FilterResult[EventEntity]:
        """Process **entity** with filter

        Args:
            entity (`EventEntity`): Source event entity
            event_type (`Type[EventLike]`): Type of event

        Returns:
            `FilterResult[EventEntity]`:
                Indicates that the filtered message should be forwarded

                Processed entity
        """
        if isinstance(entity, EventMessage):
            return await self._process_message(entity, event_type)

        if isinstance(entity, list):
            # Check for `EventAlbumMessage`: List of messages
            return await self._process_album(entity, event_type)

        return FilterResult(FilterAction.CONTINUE, entity)
```

Фильтры могут видоизменять сообщения или отбрасывать их.

Такие фильтры можно объединить в группу, которая будет выполнять поочередно, обрабатывая сообщение:
```python
class CompositeMessageFilter(MessageFilter):
    """Composite message filter that sequentially applies the filters

    Args:
        filters (`List[MessageFilter]`):
            Message filters
    """

    def __init__(self, filters: List[MessageFilter]) -> None:
        self._filters = filters

    async def process(
        self, entity: EventEntity, event_type: Type[EventLike]
    ) -> FilterResult[EventEntity]:
        for f in self._filters:
            filter_action, entity = await f.process(entity, event_type)
            match filter_action:
                case FilterAction.CONTINUE | True:
                    continue
                case FilterAction.DISCARD | False:
                    return FilterResult(FilterAction.DISCARD, entity)
                case FilterAction.FORCE_SEND:
                    return FilterResult(FilterAction.FORCE_SEND, entity)

        return FilterResult(FilterAction.CONTINUE, entity)
```

Конфигурация для пересылки имеет вид:
```python
CHAT_MAPPING: Dict[int, Dict[int, List["DirectionConfig"]]] = {}

@dataclass
class DirectionConfig:
    disable_delete: bool
    disable_edit: bool
    filters: MessageFilter
    from_topic_id: Optional[int] = None
    to_topic_id: Optional[int] = None
    mode: Literal["copy", "forward"] = "copy"
```
В общем, конфигурация для ретранслирования состоит из маппинг каналов (*from->to*) и конфига для этого направления.

## Пример YAML конфигурации

```yaml
# Направления пересылки
directions:
  - from: [-1001, -1002, -1003] # Каналы источники
    to: [-100203] # Каналы приёмники

  - from: [-1000#3] # Пересылка из топика в топик
    to: [-1001#4]

  - from: [-100226]
    to: [-1006, -1008]
    disable_edit: false
    disable_delete: false
    mode: forward  # Вид пересылки: копирование или пересылка
    filters: # Список фильтров, который будет применен поочередно
      - UrlMessageFilter:
          blacklist: !!set
            ? t.me
      - KeywordReplaceFilter:
          keywords:
            "google.com": "bing.com"
            "r'google\\.com.*'": "bing.com"
      - SkipWithKeywordsFilter:
          keywords: !!set
            ? "stopword"
            ? "r'badword.*'"
```

Подробнее смотрите в репозитории проекта: https://github.com/khoben/telemirror

## Возможные проблемы

### Бан аккаунта или деактивация сессии

Ретранслятор ведёт себя не как обычный пользователь: всегда онлайн, не смотрит рекламу и т.д. В качестве юзербота для ретранслятора используйте не свой основной аккаунт. Для вновь созданных аккаунтов большой риск бана, стоит некоторое время попользоваться им в режиме обычного пользователя.

### Новые сообщения не приходят

Telegram оптимизирует доставку сообщений, поэтому в реальном времени могут приходить не все обновления. Бывает такое, что со следующим перезапуском все же начинают приходить обновления.

## Где запустить
К сожалению, на момент написания статьи не знаю сервиса предоставляющего бесплатный тариф для 24/7 работы приложения. Придется хорошенько поискать или все-таки арендовать свой виртуальный сервер, который может пригодится не только для ретранслятора. Но и для, например, организации виртуальной приватной сети.