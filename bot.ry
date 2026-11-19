import os
import asyncio
import logging
import aiohttp
from aiogram import Bot, Dispatcher, F, types
from aiogram.filters import CommandStart
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton

TELEGRAM_TOKEN = os.getenv("BOT_TOKEN")
AI_API_KEY = os.getenv("AI_API_KEY")

API_ENDPOINT = "https://api.aitunnel.ru/v1/chat/completions"
REQUEST_TIMEOUT = aiohttp.ClientTimeout(total=60)

logging.basicConfig(level=logging.INFO)
bot = Bot(token=TELEGRAM_TOKEN)
dp = Dispatcher()

class BotStates(StatesGroup):
    waiting_for_good_traits = State()
    waiting_for_bad_traits = State()

main_keyboard = ReplyKeyboardMarkup(
    keyboard=[
        [KeyboardButton(text="🌟 Узнать, на кого равняться")],
        [KeyboardButton(text="⚠️ Разобрать плохой пример")]
    ],
    resize_keyboard=True
)

PROMPT_GOOD = "Ты мудрый психолог. Оцени черты характера человека, на которого пользователь хочет равняться: 1) В чём сила этого ориентира; 2) Какие есть скрытые риски; 3) Приведи 2-3 реальных примера известных людей; 4) Посоветуй лучшую альтернативу, если образ противоречив. Пиши чётко и структурировано на русском языке."

PROMPT_BAD = "Ты мудрый психолог. Разбери плохой пример поведения по описанию пользователя: 1) Почему нельзя брать с него пример и к чему это приводит; 2) Приведи 2-3 примера из истории или жизни; 3) На кого лучше равняться вместо этого. Пиши чётко и структурировано на русском языке."

async def get_ai_response(system_prompt: str, user_text: str) -> str:
    headers = {
        "Authorization": f"Bearer {AI_API_KEY}",
        "Content-Type": "application/json"
    }
    payload = {
        "model": "gpt-4o-mini",
        "messages": [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_text}
        ],
        "temperature": 0.7,
        "max_tokens": 1500
    }
    try:
        async with aiohttp.ClientSession(timeout=REQUEST_TIMEOUT) as session:
            async with session.post(API_ENDPOINT, headers=headers, json=payload) as resp:
                raw_text = await resp.text()
                if resp.status != 200:
                    return f"Код {resp.status} | Ответ: {raw_text[:300]}"
                data = await resp.json()
                return data["choices"][0]["message"]["content"]
    except aiohttp.ClientError as e:
        return f"Проблема с сетью при обращении к API: {e}"
    except Exception as e:
        return f"Сбой: {e}"

@dp.message(CommandStart())
async def cmd_start(message: types.Message, state: FSMContext):
    await state.clear()
    await message.answer("Привет! Выбери режим ниже 👇", reply_markup=main_keyboard)

@dp.message(F.text == "🌟 Узнать, на кого равняться")
async def ask_good_traits(message: types.Message, state: FSMContext):
    await state.set_state(BotStates.waiting_for_good_traits)
    await message.answer("Опиши черты характера человека, на которого ты хочешь равняться:")

@dp.message(F.text == "⚠️ Разобрать плохой пример")
async def ask_bad_traits(message: types.Message, state: FSMContext):
    await state.set_state(BotStates.waiting_for_bad_traits)
    await message.answer("Опиши черты характера человека, которого нужно разобрать как плохой пример:")

@dp.message(BotStates.waiting_for_good_traits)
async def process_good_traits(message: types.Message, state: FSMContext):
    wait_msg = await message.answer("⏳ Анализирую...")
    answer = await get_ai_response(PROMPT_GOOD, message.text)
    await wait_msg.delete()
    await message.answer(answer[:4096], reply_markup=main_keyboard)
    await state.clear()

@dp.message(BotStates.waiting_for_bad_traits)
async def process_bad_traits(message: types.Message, state: FSMContext):
    wait_msg = await message.answer("⏳ Разбираю...")
    answer = await get_ai_response(PROMPT_BAD, message.text)
    await wait_msg.delete()
    await message.answer(answer[:4096], reply_markup=main_keyboard)
    await state.clear()

@dp.message()
async def fallback_handler(message: types.Message):
    await message.answer("Выбери кнопку внизу 👇", reply_markup=main_keyboard)

async def main():
    print("Бот запущен!")
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
