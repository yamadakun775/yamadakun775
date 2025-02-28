import discord
from discord.ext import commands
import asyncio
from dotenv import load_dotenv
import os

# .env ファイルから環境変数を読み込む
load_dotenv()

# トークンを環境変数から取得
TOKEN = os.getenv("DISCORD_TOKEN")

# Intents設定
intents = discord.Intents.default()
intents.messages = True
intents.guilds = True
intents.members = True
intents.message_content = True  # メッセージの内容を取得するために必要

# BOTのプレフィックス設定
bot = commands.Bot(command_prefix='!', intents=intents)

# 管理者かどうかのチェック関数
def is_admin(ctx):
    return ctx.author.guild_permissions.administrator

# スパム検出用の辞書
user_messages = {}

# 警告メッセージを送信する関数
async def send_warning(message, violation_reason, is_warned=False):
    user = message.author
    channel = message.channel

    # 違反ユーザーが注意人物かどうかを確認
    if is_warned:
        warn_message = f'⚠️ {user.mention} は **{violation_reason}** に違反しました。\n' \
                       '💬 このユーザーがスパム行為をしています。どうしますか？'
    else:
        warn_message = f'⚠️ {user.mention} は **{violation_reason}** に違反しました。\n'

    warning_message = await channel.send(warn_message)

    # Banボタンと投票ボタンを追加
    await warning_message.add_reaction("🔴")  # バンボタン
    await warning_message.add_reaction("⬆️")  # 注意人物指定
    await warning_message.add_reaction("⬇️")  # タイムアウト解除

    # 投票の処理
    await start_vote(message, warning_message, user, is_warned)

# 投票システム（注意人物指定 or タイムアウト解除）
async def start_vote(message, warning_message, user, is_warned):
    channel = message.channel
    votes = {"warn": 0, "unban": 0}

    def check(reaction, user_reacted):
        return user_reacted != bot.user and reaction.message.id == warning_message.id

    # 注意人物がスパムした場合、管理者のみ投票できる
    if is_warned:
        while True:
            reaction, user_reacted = await bot.wait_for("reaction_add", check=check)

            # 管理者のみ反応を受け付ける
            if is_admin(user_reacted):
                if reaction.emoji == "⬆️":
                    votes["warn"] += 1
                elif reaction.emoji == "⬇️":
                    votes["unban"] += 1

                # 3票以上になったら決定
                if votes["warn"] >= 3:
                    await apply_warn(user, channel)
                    break
                elif votes["unban"] >= 3:
                    await remove_timeout(user, channel)
                    break
    else:
        # 一般ユーザーの場合、投票は通常通り行われる
        while True:
            reaction, user_reacted = await bot.wait_for("reaction_add", check=check)

            if reaction.emoji == "⬆️":
                votes["warn"] += 1
                if is_admin(user_reacted):
                    votes["warn"] += 2  # 管理者の投票は即決
            elif reaction.emoji == "⬇️":
                votes["unban"] += 1
                if is_admin(user_reacted):
                    votes["unban"] += 2  # 管理者の投票は即決

            # 3票以上になったら決定
            if votes["warn"] >= 3:
                await apply_warn(user, channel)
                break
            elif votes["unban"] >= 3:
                await remove_timeout(user, channel)
                break

# 注意人物にする処理
async def apply_warn(user, channel):
    guild = channel.guild
    warn_role = discord.utils.get(guild.roles, name="注意人物")
    if warn_role is None:
        warn_role = await guild.create_role(name="注意人物")
    await user.add_roles(warn_role)
    await channel.send(f'⚠️ {user.mention} を「注意人物」に指定しました。')

# タイムアウト解除
async def remove_timeout(user, channel):
    try:
        await user.edit(timed_out_until=None)  # タイムアウト解除
        await channel.send(f'✅ {user.mention} のタイムアウトを解除しました。')
    except Exception as e:
        print(e)

# スパム検出と処理
@bot.event
async def on_message(message):
    if message.author.bot:  # BOTのメッセージは無視
        return

    # スパム検出のためにユーザーごとにメッセージのタイムスタンプを記録
    user_id = message.author.id
    now = message.created_at.timestamp()

    # メッセージ履歴を記録
    if user_id not in user_messages:
        user_messages[user_id] = []
    user_messages[user_id].append(now)

    # 7秒以内に5回以上メッセージを送信していた場合スパムとみなす
    user_messages[user_id] = [t for t in user_messages[user_id] if now - t <= 7]

    if len(user_messages[user_id]) >= 5:
        # スパム行為として警告を送信
        violation_reason = "スパム行為"
        
        # ユーザーが注意人物かどうかの確認
        is_warned = False
        member = message.guild.get_member(user_id)
        if member:
            if any(role.name == "注意人物" for role in member.roles):
                is_warned = True
        
        await send_warning(message, violation_reason, is_warned)

    await bot.process_commands(message)  # 他のコマンドも処理

# ボットが起動したときに表示するメッセージ
@bot.event
async def on_ready():
    print(f'Logged in as {bot.user}')

# ボットのトークンを実行時に読み込む
bot.run(TOKEN)
