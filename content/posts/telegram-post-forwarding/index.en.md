---
title: "TeleMirror: Telegram channel relay"
date: 2025-12-23
description: "Creating your own Telegram channel relay with filters and forwarding settings"
summary: "Creating your own Telegram channel relay with filters and forwarding settings"
categories: blog
tags: ["telegram", "python", "telethon", "bot", "forwarding", "mirroring"]
draft: false
images:
  - "cover.png"
featured_image: "cover.png"
cover:
    image: "cover.png"
    alt: "TeleMirror - github.com/khoben/telemirror"
    relative: true
---

Back in 2020, I wrote an article on vc.ru ["Creating and deploying a Telegram channel relay using Python and Heroku"](https://vc.ru/dev/158757-sozdanie-i-razvertyvanie-retranslyatora-telegram-kanalov-ispolzuya-python-i-heroku). Since then, Heroku no longer offers a free plan. But the project is still alive and being updated.

</> Project repository: https://github.com/khoben/telemirror

Actually, there is nothing supernatural about forwarding: we receive a message from the source channel and send a copy of it to the receiving channel.

In the simplest case, it looks like this:
```python
from telethon import TelegramClient, events

client = TelegramClient("telemirror", api_id, api_hash)
SOURCE_CHANNELS = [-10001, -10002] # channel IDs
TARGET_CHANNELS = [-10003, -10004]

@client.on(events.NewMessage(chats=SOURCE_CHANNELS))
async def handler(event):
    for target in TARGET_CHANNELS:
        await client.send_message(target, event.message)

client.start()
client.run_until_disconnected()
```

> Receiving a message → Sending a message

The only thing missing is some configuration and flexibility. Which messages to discard, whether to change the original text, from which channel to forward to which channel, etc. 

Modifiers, which can be called filters, are added to the simple scheme described above.

In [TeleMirror](https://github.com/khoben/telemirror), the simplest filter for messages looks like this:
```python
class MessageFilter(Protocol):
    async def process(
        self, entity: EventEntity, event_type: Type[EventLike]
    ) -> FilterResult[EventEntity]:
        if isinstance(entity, EventMessage):
            # Message processing: continue, modify, discard
            return await self._process_message(entity, event_type)

        if isinstance(entity, list):
            # Processing a group of messages - an album
            return await self._process_album(entity, event_type)

        return FilterResult(FilterAction.CONTINUE, entity)
```

Filters can modify messages or discard them.

Such filters can be combined into a group that will execute sequentially, processing the message:
```python
class CompositeMessageFilter(MessageFilter):
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

> Message reception → (filters) → Sending (or not) the processed message

The configuration for forwarding looks like this:
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
In general, the configuration for retransmission consists of channel mapping (*from->to*) and the configuration for this direction.

## Example of YAML configuration

```yaml
# Forwarding directions
directions:
  - from: [-1001, -1002, -1003] # Source channels
    to: [-100203] # Receiver channels

  - from: [-1000#3] # Forwarding from topic to topic
    to: [-1001#4]

  - from: [-100226]
    to: [-1006, -1008]
    disable_edit: false
    disable_delete: false
    mode: forward  # Forwarding type: copy or forward
    filters: # List of filters that will be applied sequentially
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

</> For more details, see the project repository: https://github.com/khoben/telemirror

## Possible problems

### Account ban or session deactivation

The retransmitter does not behave like a normal user: it is always online, does not view ads, etc. Do not use your main account as a userbot for the relay. Newly created accounts are at high risk of being banned, so it is worth using them as a regular user for a while.

### New messages are not coming through

Telegram optimizes message delivery, so not all updates may come through in real time. Sometimes, updates start coming in after the next restart.

## Where to run
Unfortunately, at the time of writing, I don't know of any service that offers a free plan for continuous operation of the application without crashes. You'll have to search for one or rent your own virtual server, which can be useful not only for the relay.