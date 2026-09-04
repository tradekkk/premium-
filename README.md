import asyncio
import logging
from aiogram import Bot, Dispatcher, F, Router
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.types import (
    CallbackQuery,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
    LabeledPrice,
    Message,
    PreCheckoutQuery,
)

# Вставьте ваш токен от @BotFather
TOKEN = "YOUR_BOT_TOKEN_HERE"
# Токен платежного провайдера (получается через @BotFather -> /mybots -> Payments)
PROVIDER_TOKEN = "YOUR_PROVIDER_TOKEN_HERE"

router = Router()
logging.basicConfig(level=logging.INFO)


# Состояния для процесса покупки
class SubscriptionState(StatesGroup):
    waiting_for_payment = State()


# Тарифы подписки
TARIFFS = {
    "sub_1month": {
        "title": "Подписка на 1 месяц",
        "description": "Доступ ко всем премиум-функциям на 30 дней",
        "price": 29900,  # Цены указываются в копейках/центах (299.00 RUB)
        "currency": "RUB",
    },
    "sub_1year": {
        "title": "Подписка на 1 год (Выгода 30%)",
        "description": "Доступ ко всем премиум-функциям на 365 дней",
        "price": 249000,  # 2490.00 RUB
        "currency": "RUB",
    },
}


# Команда /start
@router.message(Command("start"))
async def cmd_start(message: Message):
  keyboard = InlineKeyboardMarkup(
      inline_keyboard=[
          [
              InlineKeyboardButton(
                  text="💎 Оформить подписку", callback_data="buy_subscription"
              )
          ],
          [
              InlineKeyboardButton(
                  text="👤 Личный кабинет", callback_data="profile"
              )
          ],
      ]
  )

  await message.answer(
      f"Привет, **{message.from_user.first_name}**!\n\n"
      "🤖 Этот бот позволяет приобретать автоматические подписки на наши эксклюзивные услуги.",
      reply_markup=keyboard,
      parse_mode="Markdown",
  )


# Меню выбора подписок
@router.callback_query(F.data == "buy_subscription")
async def choose_subscription(callback: CallbackQuery):
  keyboard = InlineKeyboardMarkup(
      inline_keyboard=[
          [
              InlineKeyboardButton(
                  text="📅 1 Месяц — 299 RUB", callback_data="sub_1month"
              )
          ],
          [
              InlineKeyboardButton(
                  text="🌟 1 Год — 2490 RUB", callback_data="sub_1year"
              )
          ],
          [
              InlineKeyboardButton(
                  text="« Назад", callback_data="back_to_main"
              )
          ],
      ]
  )

  await callback.message.edit_text(
      "Выберите подходящий тарифный план:", reply_markup=keyboard
  )
  await callback.answer()


# Возврат в главное меню
@router.callback_query(F.data == "back_to_main")
async def back_to_main(callback: CallbackQuery):
  keyboard = InlineKeyboardMarkup(
      inline_keyboard=[
          [
              InlineKeyboardButton(
                  text="💎 Оформить подписку", callback_data="buy_subscription"
              )
          ],
          [
              InlineKeyboardButton(
                  text="👤 Личный кабинет", callback_data="profile"
              )
          ],
      ]
  )
  await callback.message.edit_text(
      "Главное меню. Выберите нужное действие:", reply_markup=keyboard
  )
  await callback.answer()


# Генерация счета на оплату
@router.callback_query(F.data.in_(["sub_1month", "sub_1year"]))
async def process_tariff_choice(callback: CallbackQuery):
  tariff_key = callback.data
  tariff = TARIFFS[tariff_key]

  prices = [LabeledPrice(label=tariff["title"], amount=tariff["price"])]

  await callback.message.answer_invoice(
      title=tariff["title"],
      description=tariff["description"],
      payload=f"subscription_{tariff_key}",
      provider_token=PROVIDER_TOKEN,
      currency=tariff["currency"],
      prices=prices,
      start_parameter="time-limited-subscription",
      need_email=True,
  )
  await callback.answer()


# Предпроверка платежа (обязательный шаг Telegram)
@router.pre_checkout_query()
async def process_pre_checkout_query(pre_checkout_query: PreCheckoutQuery):
  await pre_checkout_query.answer(ok=True)


# Успешная оплата
@router.message(F.successful_payment)
async def process_successful_payment(message: Message):
  payment_info = message.successful_payment
  payload = payment_info.invoice_payload

  # Здесь вы должны добавить логику продления подписки в БД (например, SQLite/PostgreSQL)
  # Пример: activate_subscription(message.from_user.id, payload)

  await message.answer(
      "✅ **Оплата прошла успешно!**\n\n"
      "🎉 Ваша подписка активирована. Спасибо, что пользуетесь нашим сервисом!",
      parse_mode="Markdown",
  )


# Личный кабинет пользователя
@router.callback_query(F.data == "profile")
async def show_profile(callback: CallbackQuery):
  user_id = callback.from_user.id

  # Здесь делается запрос в вашу БД для проверки статуса подписки
  # status = get_user_subscription_status(user_id)
  is_active = False  # Заглушка

  sub_status = (
      "🟢 Активна (до 01.01.2027)" if is_active else "🔴 Отсутствует"
  )

  keyboard = InlineKeyboardMarkup(
      inline_keyboard=[
          [
              InlineKeyboardButton(
                  text="« Назад", callback_data="back_to_main"
              )
          ]
      ]
  )

  await callback.message.edit_text(
      f"👤 **Личный кабинет**\n\n"
      f"🆔 ID: `{user_id}`\n"
      f"📊 Статус подписки: {sub_status}",
      reply_markup=keyboard,
      parse_mode="Markdown",
  )
  await callback.answer()


async def main():
  bot = Bot(token=TOKEN)
  dp = Dispatcher()
  dp.include_router(router)

  print("Бот запущен и готов к работе...")
  await dp.start_polling(bot)


if __name__ == "__main__":
  asyncio.run(main())
