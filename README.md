from telegram import Update
from telegram.ext import CommandHandler, ContextTypes
import raffle_db
import random

# 🎯 /start command
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    user_id = user.id
    username = user.username or f"user_{user_id}"

    raffle_db.register_user(user_id, username)

    await update.message.reply_text(
        f"👋 Welcome {username}!\n"
        f"🎟 Use /buy to get raffle tickets\n"
        f"🎰 Try /play or /slots to win points\n"
        f"💸 Cash out with /cashout\n"
        f"🔥 Referral link: t.me/{context.bot.username}?start={user_id}"
    )

# 🎮 /play command (slot-style mini game)
async def play(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    if not raffle_db.has_enough_points(user_id, 1):
        await update.message.reply_text("❌ Not enough points to play. Use /daily or /refer.")
        return

    symbols = ["🍒", "🍋", "🍇", "🔔", "⭐", "💎"]
    grid = [[random.choice(symbols) for _ in range(5)] for _ in range(3)]
    flat = sum(grid, [])
    winnings = flat.count("💎") * 2 + flat.count("⭐")

    raffle_db.update_points(user_id, -1 + winnings)

    grid_display = "\n".join(" ".join(row) for row in grid)
    await update.message.reply_text(f"{grid_display}\n\nYou won {winnings} points!")

# 💰 /points balance check
async def points(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    points = raffle_db.get_user_points(user_id)
    await update.message.reply_text(f"💵 You have {points} points.")

# 🎁 /daily bonus
async def daily(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    bonus = raffle_db.claim_daily_bonus(user_id)
    if bonus:
        await update.message.reply_text(f"🎉 You received {bonus} daily points!")
    else:
        await update.message.reply_text("🕐 You already claimed your bonus today.")

# 🧑‍🤝‍🧑 /refer user
async def refer(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    if context.args:
        referrer_id = int(context.args[0])
        if referrer_id != user_id:
            raffle_db.add_referral(referrer_id, user_id)
            await update.message.reply_text("✅ Referral recorded!")
        else:
            await update.message.reply_text("❌ You can't refer yourself.")
    else:
        await update.message.reply_text("📢 Share this link: "
                                        f"t.me/{context.bot.username}?start={user_id}")

# 🏆 /leaderboard
async def leaderboard(update: Update, context: ContextTypes.DEFAULT_TYPE):
    leaders = raffle_db.get_leaderboard()
    if not leaders:
        await update.message.reply_text("📉 No users on the leaderboard yet.")
        return
    message = "🏆 Top Users:\n"
    for i, (username, points) in enumerate(leaders, start=1):
        message += f"{i}. {username} — {points} points\n"
    await update.message.reply_text(message)

# 🎰 /slots (visual version of /play)
async def slots(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    if not raffle_db.has_enough_points(user_id, 1.5):
        await update.message.reply_text("❌ Not enough points to spin (need 1.5).")
        return

    symbols = ["🍒", "🍋", "🍇", "🔔", "⭐", "💎"]
    grid = [[random.choice(symbols) for _ in range(5)] for _ in range(3)]
    flat = sum(grid, [])
    matches = flat.count("💎") * 2 + flat.count("⭐")
    winnings = matches

    raffle_db.update_points(user_id, -1.5 + winnings)

    grid_display = "\n".join(" ".join(row) for row in grid)
    await update.message.reply_text(f"{grid_display}\n\nYou won {winnings} points!")

# 🎟️ /cashout points to tickets
async def cashout(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    points = raffle_db.get_user_points(user_id)
    tickets = int(points // 1.5)
    if tickets == 0:
        await update.message.reply_text("❌ Not enough points to cash out.")
        return
    used_points = tickets * 1.5
    raffle_db.update_points(user_id, -used_points)
    raffle_db.issue_tickets(user_id, tickets)

    await update.message.reply_text(f"🎟️ You received {tickets} raffle tickets for {used_points} points.")

# ✅ Register all handlers
def register_handlers(app):
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("play", play))
    app.add_handler(CommandHandler("points", points))
    app.add_handler(CommandHandler("daily", daily))
    app.add_handler(CommandHandler("refer", refer))
    app.add_handler(CommandHandler("leaderboard", leaderboard))
    app.add_handler(CommandHandler("slots", slots))
    app.add_handler(CommandHandler("cashout", cashout))