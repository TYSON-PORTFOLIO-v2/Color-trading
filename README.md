import random
import json
import os
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    CallbackQueryHandler,
    ContextTypes,
    MessageHandler,
    filters
)

# Bot configuration
BOT_TOKEN = "7784773658:AAFB2DRDUKTsUrjn6ALeYBQlpSh0O4CdCn4"
OWNER_USERNAME = "@TYSON_OWNER"  # Owner's Telegram username

# Game configuration
COLORS = ["🔴 Red", "🟢 Green", "🟣 Violet"]
COLOR_EMOJIS = {"🔴 Red": "🔴", "🟢 Green": "🟢", "🟣 Violet": "🟣"}
MULTIPLIERS = {"🔴 Red": 1.5, "🟢 Green": 2.0, "🟣 Violet": 3.0}
WIN_PROBABILITY = 0.4  # 40% chance to win

# User data management
USER_DATA_FILE = "user_data.json"

def load_user_data():
    if os.path.exists(USER_DATA_FILE):
        with open(USER_DATA_FILE, 'r') as f:
            return json.load(f)
    return {}

def save_user_data(user_data):
    with open(USER_DATA_FILE, 'w') as f:
        json.dump(user_data, f)

user_data = load_user_data()

# Telegram bot commands
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = str(update.effective_user.id)
    if user_id not in user_data:
        user_data[user_id] = {
            "balance": 1000,
            "username": update.effective_user.username,
            "first_name": update.effective_user.first_name
        }
        save_user_data(user_data)
    
    keyboard = [
        [InlineKeyboardButton("🎮 Play Game", callback_data="play")],
        [InlineKeyboardButton("💰 Balance", callback_data="balance"),
         InlineKeyboardButton("ℹ️ Help", callback_data="help")]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    
    await update.message.reply_text(
        f"🎉 Welcome to Color Prediction Game!\n\n"
        f"🏦 Starting Balance: 1000 coins\n"
        f"👑 Bot Owner: {OWNER_USERNAME}\n\n"
        "Choose an option:",
        reply_markup=reply_markup
    )

async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    if query.data == "play":
        await play_game(update, context)
    elif query.data == "balance":
        await check_balance(update, context)
    elif query.data == "help":
        await show_help(update, context)
    elif query.data == "back":
        await start(update, context)
    elif any(query.data.startswith(color) for color in COLORS):
        await process_bet(update, context)

async def play_game(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    keyboard = [
        [InlineKeyboardButton(color, callback_data=color)] for color in COLORS
    ]
    keyboard.append([InlineKeyboardButton("🔙 Back", callback_data="back")])
    
    reply_markup = InlineKeyboardMarkup(keyboard)
    
    await query.edit_message_text(
        "🎮 Place Your Bet:\n\n"
        "🔴 Red: 1.5x\n"
        "🟢 Green: 2.0x\n"
        "🟣 Violet: 3.0x\n\n"
        "Select a color:",
        reply_markup=reply_markup
    )

async def process_bet(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    user_id = str(query.from_user.id)
    selected_color = query.data
    
    # Ask for bet amount
    context.user_data["selected_color"] = selected_color
    await query.edit_message_text(f"Selected: {selected_color}\n\nEnter your bet amount:")

async def handle_bet_amount(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = str(update.message.from_user.id)
    try:
        bet_amount = int(update.message.text)
        if bet_amount <= 0:
            raise ValueError
        
        # Check balance
        if user_id not in user_data or bet_amount > user_data[user_id]["balance"]:
            await update.message.reply_text("❌ Insufficient balance!")
            return
        
        # Process game
        selected_color = context.user_data.get("selected_color")
        if not selected_color:
            await update.message.reply_text("❌ Please select a color first!")
            return
        
        # Deduct balance
        user_data[user_id]["balance"] -= bet_amount
        save_user_data(user_data)
        
        # Generate result
        win = random.random() < WIN_PROBABILITY
        winning_color = selected_color if win else random.choice(
            [c for c in COLORS if c != selected_color]
        )
        
        # Calculate payout
        if win:
            payout = int(bet_amount * MULTIPLIERS[selected_color])
            user_data[user_id]["balance"] += payout
            save_user_data(user_data)
            result_message = (
                f"🎉 You Won! {COLOR_EMOJIS[selected_color]}\n"
                f"💰 Payout: {payout} coins (+{payout - bet_amount})"
            )
        else:
            result_message = (
                f"❌ You Lost! Winning Color: {winning_color}\n"
                f"💸 Lost Amount: {bet_amount} coins"
            )
        
        # Show result
        await update.message.reply_text(
            f"🏁 Result:\n\n"
            f"Your choice: {selected_color}\n"
            f"Bet amount: {bet_amount} coins\n\n"
            f"{result_message}\n\n"
            f"💵 New Balance: {user_data[user_id]['balance']} coins"
        )
        
    except ValueError:
        await update.message.reply_text("❌ Invalid amount! Please enter a positive number.")

async def check_balance(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    user_id = str(query.from_user.id)
    
    if user_id in user_data:
        await query.edit_message_text(
            f"💰 Your Balance: {user_data[user_id]['balance']} coins\n\n"
            f"👑 Bot Owner: {OWNER_USERNAME}"
        )
    else:
        await query.edit_message_text("❌ You don't have an account yet! Use /start to begin.")

async def show_help(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.edit_message_text(
        f"🎮 Color Prediction Game Guide:\n\n"
        f"1. Place bets on colors with different multipliers\n"
        f"2. Win probabilities are equal for all colors\n"
        f"3. Game odds favor the house long-term\n\n"
        f"🔴 Red: 1.5x payout\n"
        f"🟢 Green: 2.0x payout\n"
        f"🟣 Violet: 3.0x payout\n\n"
        f"⚠️ This is a demo game - no real money involved\n"
        f"👑 Bot Owner: {OWNER_USERNAME}"
    )

def main():
    application = Application.builder().token(BOT_TOKEN).build()
    
    # Command handlers
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CallbackQueryHandler(button_handler))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_bet_amount))
    
    # Run bot
    application.run_polling()

if __name__ == "__main__":
    main()
