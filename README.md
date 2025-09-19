# Управление модулями в проекте **Solobot**

## 1. Общая информация

Модули в системе **Solobot** позволяют расширять функциональность проекта. Управление ими доступно разработчикам после регистрации модуля.

---

## 2. Регистрация модуля

Регистрация осуществляется через форму **«Подать заявку»**.

После успешной регистрации в интерфейсе появится раздел **«Управление модулями»**.

---

## 3. Возможности раздела «Управление модулями»

В этом разделе разработчик может:

* Указывать описание модуля.
* Задавать номер версии.
* Загружать новый модуль.
* Обновлять существующий модуль.
* Прикреплять изображение модуля.

---

## 4. Требования и правила работы с модулями

* Изображение модуля — формат **16:9**.
* Выдача лицензии осуществляется через **ID пользователя** в личном кабинете.

---

## 5. Рекомендации по оформлению

* Поддерживайте актуальность описания модуля.
* Следите за корректностью номера версии.
* Перед загрузкой обновления тестируйте модуль на совместимость.

---

## Документация по написанию модулей для **Solo\_bot**

Это руководство описывает, как создавать и подключать независимые модули для бота, включая:

* структуру файлов,
* точки расширения (хуки),
* протокол вставки и удаления кнопок,
* периодические уведомления,
* быстрые сценарии оплаты,
* вебхуки и лучшие практики.

---

### Цели модульной системы:

* **Изоляция** — каждый модуль размещается в отдельной папке `modules/<module_name>/`, тексты и настройки хранятся внутри модуля.
* **Подключаемость** — модуль подключается автоматически, если в файле `router.py` он экспортирует переменную `router`.
* **Расширяемость UI** — с помощью хуков можно добавлять и удалять кнопки в существующих меню без изменений в ядре.
* **Безопасность** — ошибки модуля логируются и не приводят к остановке основного процесса.

---

### Структура модуля (минимальный набор файлов):

```
modules/
 my_module/
   __init__.py          # экспорт router (и/или другие публичные объекты)
   router.py            # основной код: aiogram Router, регистрация хуков
   texts.py             # пользовательские тексты модуля
   settings.py          # настройки модуля (в т.ч. флаги вкл/выкл)
   models.py            # (опционально) модели SQLAlchemy
   db.py                # (опционально) DAO-функции для работы с моделями
```

Рекомендации:

* храните все тексты, кнопки и сообщения внутри модуля (например, в `texts.py`);
* настраиваемые параметры помещайте в `settings.py`;
* для предотвращения циклических импортов используйте локальные импорты в обработчиках.

---

### Пример файла `__init__.py`:

```python
__all__ = ("router",)
from .router import router
# ВАЖНО: чтобы таблицы модуля попали в metadata и создавались через init_db,
# импортируйте модели на уровне модуля
from . import models  # noqa: F401
```

---

### Пример схемы БД (`models.py`):

```python
from datetime import datetime
from sqlalchemy import BigInteger, Column, DateTime, ForeignKey, String
from database.models import Base

class ChannelBonusClaim(Base):
   __tablename__ = "channel_bonus_claims"

   tg_id = Column(BigInteger, ForeignKey("users.tg_id", ondelete="CASCADE"), primary_key=True)
   claimed_at = Column(DateTime, default=datetime.utcnow)
```

---

### DAO (`db.py`):

```python
from sqlalchemy import select
from sqlalchemy.dialects.postgresql import insert
from sqlalchemy.ext.asyncio import AsyncSession
from .models import ChannelBonusClaim

async def has_claim(session: AsyncSession, tg_id: int) -> bool:
   res = await session.execute(select(ChannelBonusClaim).where(ChannelBonusClaim.tg_id == tg_id))
   return res.scalar_one_or_none() is not None

async def set_claim(session: AsyncSession, tg_id: int):
   stmt = (
       insert(ChannelBonusClaim)
       .values(tg_id=tg_id)
       .on_conflict_do_nothing(index_elements=[ChannelBonusClaim.tg_id])
   )
   await session.execute(stmt)
   await session.commit()
```

---

### Загрузка модулей

* `load_modules_from_folder()` — подключение `router`
* `load_module_webhooks()` — регистрация вебхуков
* `load_module_fast_flow_handlers()` — обработчики быстрого сценария оплаты

---

### Хуки (точки расширения)

Пример регистрации:

```python
from aiogram.types import InlineKeyboardButton
from aiogram.utils.keyboard import InlineKeyboardBuilder
from hooks.hooks import register_hook

def _build_button():
   return InlineKeyboardButton(text="Моя кнопка", callback_data="my_module|open")

async def profile_menu_hook(**kwargs):
   return {"after": "balance", "button": _build_button()}

register_hook("profile_menu", profile_menu_hook)
```

---

### Примеры вставки/удаления кнопок

1. Убрать кнопку TV и добавить свою:

```python
return [
 {"remove": f"connect_tv|{key_name}"},
 {"button": InlineKeyboardButton(text="📺 Happ TV", callback_data=f"happ_tv|{key_name}")},
]
```

2. Добавить кнопку после «balance»:

```python
return {"after": "balance", "button": InlineKeyboardButton(text="🏆 Топ", callback_data="top_referrers")}
```

3. Удалить все кнопки с префиксом:

```python
return {"remove_prefix": "legacy_"}
```

---

### Периодические уведомления

```python
from hooks.hooks import register_hook

async def periodic_notify(bot, session, keys, **kwargs):
   pass

register_hook("periodic_notifications", periodic_notify)
```

---

### «Быстрый флоу» оплаты

```python
def get_fast_flow_handler():
   return {
       "payment_key": "my_flow",
       "handler": my_flow_handler,
   }
```

---

### Вебхуки

```python
def get_webhook_data():
   return {
       "path": "/api/my_module",
       "handler": my_webhook_handler
   }
```

---

### Пример минимального модуля с кнопкой

```python
# modules/hello_world/router.py
from aiogram import Router, F
from aiogram.types import CallbackQuery, InlineKeyboardButton
from aiogram.utils.keyboard import InlineKeyboardBuilder
from hooks.hooks import register_hook

router = Router()

def _btn():
   return InlineKeyboardButton(text="👋 Привет", callback_data="hello|open")

async def profile_menu_hook(**kwargs):
   return {"after": "balance", "button": _btn()}

register_hook("profile_menu", profile_menu_hook)

@router.callback_query(F.data == "hello|open")
async def open_hello(callback: CallbackQuery):
   kb = InlineKeyboardBuilder()
   kb.row(InlineKeyboardButton(text="⬅️ Назад", callback_data="profile"))
   await callback.message.edit_text("Привет из модуля!", reply_markup=kb.as_markup())
```

---

### Чек-лист при публикации модуля:

* Есть `router`, экспортированный из `__init__.py`.
* Все тексты находятся в `texts.py`, настройки — в `settings.py`.
* Хуки зарегистрированы через `register_hook`.
* FSM очищается при отменах и ошибках.
* Кнопки изменяются только через протокол операций.
* Сетевые вызовы с таймаутами и логированием ошибок.
* Даты и время согласованы с типами БД.

---
