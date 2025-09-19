Управление модулями в проекте Solobot
1. Общая информация
Модули в системе Solobot позволяют расширять функциональность проекта. Управление ими доступно разработчикам после регистрации модуля.

2. Регистрация модуля
Регистрация осуществляется через форму “Подать заявку”.



После успешной регистрации в интерфейсе появится раздел «Управление модулями».



3. Возможности раздела «Управление модулями»
В этом разделе разработчик может:

Указывать описание модуля.
Задавать номер версии.
Загружать новый модуль.
Обновлять существующий модуль.
Прикреплять изображение модуля.
4. Требования и правила работы с модулями
Изображение модуля — формат 16:9.
Выдача лицензии осуществляется через ID пользователя в личном кабинете.
5. Рекомендации по оформлению
Поддерживайте актуальность описания модуля.
Следите за корректностью номера версии.
Перед загрузкой обновления тестируйте модуль на совместимость.
Теперь рассмотрим документацию по написанию модулей для Solo_bot.
Это руководство описывает, как создавать и подключать независимые модули для бота, включая структуру файлов, точки расширения (хуки), протокол вставки и удаления кнопок, периодические уведомления, быстрые сценарии оплаты, вебхуки и лучшие практики.

Цели модульной системы:

Изоляция — каждый модуль размещается в отдельной папке modules/<module_name>/, тексты и настройки хранятся внутри модуля.

Подключаемость — модуль подключается автоматически, если в файле router.py он экспортирует переменную router.

Расширяемость UI — с помощью хуков можно добавлять и удалять кнопки в существующих меню без внесения изменений в ядро.

Безопасность — ошибки модуля логируются и не приводят к остановке основного процесса.

Структура модуля. Минимальный набор файлов:

modules/
 my_module/
   __init__.py          # экспорт router (и/или другие публичные объекты)
   router.py            # основной код: aiogram Router, регистрации хуков
   texts.py             # пользовательские тексты модуля
   settings.py          # настройки модуля (в т.ч. флаги вкл/выкл)
   models.py            # (опционально) модели SQLAlchemy для таблиц модуля
   db.py                # (опционально) DAO-функции для работы с моделями модуля
Рекомендации: храните все тексты, кнопки и сообщения внутри модуля, например в texts.py. Настраиваемые параметры помещайте в settings.py. Для предотвращения циклических импортов используйте локальные импорты внутри обработчиков при необходимости.

Пример файла __init__.py:

__all__ = ("router",)
from .router import router
# ВАЖНО: чтобы таблицы модуля попали в metadata и создавались через init_db,
# импортируйте модели на уровне модуля
from . import models  # noqa: F401
Схема базы данных в модуле (models.py). Если модуль создаёт или расширяет хранилище, определяйте свои таблицы локально в modules/<name>/models.py.

Пример:

from datetime import datetime
from sqlalchemy import BigInteger, Column, DateTime, ForeignKey, String
from database.models import Base

class ChannelBonusClaim(Base):
   __tablename__ = "channel_bonus_claims"

   tg_id = Column(BigInteger, ForeignKey("users.tg_id", ondelete="CASCADE"), primary_key=True)
   claimed_at = Column(DateTime, default=datetime.utcnow)
DAO (рекомендуется вынести в db.py):

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
Регистрация и создание таблиц: бот вызывает Base.metadata.create_all() в database/init_db.py. Чтобы таблицы модуля попали в общее metadata, необходимо импортировать файл modules/<name>/models.py до вызова createall(). Рекомендуется добавлять строку from . import models в modules/<name>/__init__.py, так как загрузчик модулей импортирует modules.<name>.router, при этом выполняется __init__.py и модели регистрируются автоматически.

Примечания: не добавляйте модели модулей в database/models.py, держите схему модуля локально. Типы дат и времени согласовывайте с существующей базой данных (TIMESTAMP WITHOUT TIME ZONE — используйте naive UTC). Для сложных изменений схемы используйте миграции, например Alembic, в простых случаях достаточно create_all().

Загрузка модулей: загрузчик ищет модули в папке modules и импортирует modules.<name>.router. Если модуль экспортирует переменную router типа aiogram.Router, он подключается автоматически. Необязательные интеграции: функция get_webhook_data(), возвращающая словарь с ключами path и handler для вебхуков, и функция get_fast_flow_handler(), возвращающая словарь с ключами payment_key и handler для быстрого сценария оплаты.

См. реализацию в utils/modules_loader.py:

load_modules_from_folder() — подключение router

load_module_webhooks() — регистрация вебхуков модулей

load_module_fast_flow_handlers() — регистрация обработчиков быстрого сценария оплаты

Хуки (точки расширения) — именованные события, на которые модуль может подписаться через register_hook(name, func) из hooks.hooks. Доступные хуки на момент написания: start_link, start_menu, profile_menu, view_key_menu, pay_menu_buttons, admin_panel, periodic_notifications.

Пример регистрации хука в модуле:

from aiogram.types import InlineKeyboardButton
from aiogram.utils.keyboard import InlineKeyboardBuilder
from hooks.hooks import register_hook

def _build_button():
   return InlineKeyboardButton(text="Моя кнопка", callback_data="my_module|open")

async def profile_menu_hook(**kwargs):
   # Возвращаем операции вставки кнопок (см. протокол ниже)
   return {"after": "balance", "button": _build_button()}

register_hook("profile_menu", profile_menu_hook)
Лучшие практики: принимать **kwargs и извлекать нужные параметры по имени, так как некоторые хуки передают session, chat_id, admin и т.д. Не делать предположений о наличии параметров, использовать значения по умолчанию.

Протокол вставки и удаления кнопок: для модификации клавиатуры используются операции, возвращаемые из хука, которые обрабатываются функцией insert_hook_buttons(). Поддерживаемые операции: добавление кнопки в конец, вставка новой кнопки после указанной, удаление кнопок по точному совпадению callback_data, удаление кнопок по префиксу callback_data. Можно возвращать список операций, включая вложенные списки — они будут объединены перед обработкой.

Примеры:

# 1) Убрать стандартную кнопку TV и добавить свою
return [
 {"remove": f"connect_tv|{key_name}"},
 {"button": InlineKeyboardButton(text="📺 Happ TV", callback_data=f"happ_tv|{key_name}")},
]

# 2) Добавить кнопку после кнопки «balance» в профиле
return {"after": "balance", "button": InlineKeyboardButton(text="🏆 Топ", callback_data="top_referrers")}

# 3) Удалить все кнопки с префиксом
return {"remove_prefix": "legacy_"}
Периодические уведомления: модуль может участвовать в них через хук periodic_notifications.

from hooks.hooks import register_hook

async def periodic_notify(bot, session, keys, **kwargs):
   # keys — список активных подписок (объекты моделей)
   # можно отправлять сообщения, использовать БД через session и т.д.
   pass

register_hook("periodic_notifications", periodic_notify)
Контекст выполнения: запуск в цикле handlers/notifications/general_notifications.py, передаётся bot, session (AsyncSession), keys (список ключей без замороженных). Используйте неблокирующие вызовы и таймауты при сетевых запросах.

«Быстрый флоу» оплаты (опционально):

def get_fast_flow_handler():
   return {
       "payment_key": "my_flow",  # ключ сценария
       "handler": my_flow_handler,  # async def handler(callback_or_message, session, **kwargs)
   }
Вебхуки модулей (опционально):

def get_webhook_data():
   return {
       "path": "/api/my_module",   # относительный путь
       "handler": my_webhook_handler # async def handler(request): ...
   }
Работа с FSM и UX: всегда добавляйте кнопку «Назад» или «Отмена» для выхода из состояния. Очищайте состояние при ошибках и после завершения сценария. Для кнопок используйте InlineKeyboardBuilder, для вывода — хелпер handlers.utils.edit_or_send_message.

Пример обработчика «Назад»:

@router.callback_query(F.data.startswith("my_flow_cancel|"))
async def my_flow_cancel(callback: CallbackQuery, state: FSMContext, session):
   await state.clear()
   # вернуться на экран карточки
   key_name = callback.data.split("|")[1]
   from handlers.keys.key_view import render_key_info
   import os
   await render_key_info(callback.message, session, key_name, os.path.join("img", "pic_view.jpg"))
Работа с базой данных и временем: используйте AsyncSession через DI-middleware. Для поля TIMESTAMP WITHOUT TIME ZONE передавайте naive UTC.

from datetime import datetime, timezone

_dt = datetime.fromisoformat(iso_str.replace("Z", "+00:00"))
start_dt = _dt.astimezone(timezone.utc).replace(tzinfo=None)  # naive UTC
Сетевые запросы: используйте aiohttp.ClientSession() с таймаутами 10–15 секунд, логируйте ответы при ошибках, кодируйте бинарные и секретные данные при необходимости.

Логи и обработка ошибок: логгер доступен как from logger import logger. Хуки выполняются с защитой, исключения логируются. В обработчиках отлавливайте ожидаемые ошибки и показывайте понятные пользователю сообщения.

Пример минимального модуля с кнопкой в профиле:

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
Пример замены стандартной кнопки в карточке подписки:

from aiogram.types import InlineKeyboardButton
from hooks.hooks import register_hook

def _happ_btn(key_name: str):
   return InlineKeyboardButton(text="📺 Подключить Happ TV", callback_data=f"happ_tv|{key_name}")

async def view_key_menu_hook(key_name: str, **kwargs):
   return [
       {"remove": f"connect_tv|{key_name}"},
       {"button": _happ_btn(key_name)},
   ]

register_hook("view_key_menu", view_key_menu_hook)
Тексты и настройки внутри модуля: пример settings.py:

ENABLE_BUTTON = True
START_AT_ISO = "2025-08-18T00:00:00Z"
MIN_TOPUP_RUB = 100
TOP_N = 5
Пример texts.py:

BTN_TOP = "🏆 Топ пригласивших"
TITLE = "<b>🏆 Топ пригласивших</b>\n"
ENTRY = "{place}. <code>{masked_id}</code> — {count} рефералов\n"
NO_DATA = "Пока нет участников, подходящих под условия."
Импортируйте тексты и настройки локально, чтобы избежать циклических зависимостей.

Советы по качеству кода: читайте и пишите в БД через существующие функции, держите обработчики короткими, выносите расчёты в отдельные функции, используйте DI для получения session.

Чек-лист при публикации модуля:

Есть router, экспортированный из __init__.py.
Все тексты находятся в texts.py, настройки — в settings.py.
Хуки зарегистрированы через register_hook.
FSM очищается при отменах и ошибках.
Кнопки изменяются только через протокол операций.
Сетевые вызовы с таймаутами и логированием ошибок.
Даты и время согласованы с типами БД.
