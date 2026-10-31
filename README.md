# 🦜 Kookie Datacenter Discord Bot — All-in-One Deployment Bundle

> **Official Community & Live Deployment:**  
> 🔗 **Server Invite:** [https://discord.gg/rWeuKGWhUn](https://discord.gg/rWeuKGWhUn)  
> *As deployed and used in the official Kookie community server!*

This document consolidates every file, script, schema, configuration, and operating procedure required to deploy, configure, and maintain the Discord Multi-Stream, Role Management, and Gemini AI companion bot across Linux (`systemd`) and Windows environments.

## Table of Contents

1. [Core Application (`multi_stream_bot.py`)](#1-core-application-multi_stream_botpy)

2. [Configuration File Template (`config.txt`)](#2-configuration-file-template-configtxt)

3. [Persona Specification (`persona.json`)](#3-persona-specification-personajson)

4. [Memory & Fact Store (`bot_memory.json`)](#4-memory--fact-store-bot_memoryjson)

5. [Linux Systemd Service (`discord-role-bot.service`)](#5-linux-systemd-service-discord-role-botservice)

6. [Windows Automation Runner (`run_bot.bat`)](#6-windows-automation-runner-run_botbat)

7. [Windows Background Launcher (`start_hidden.vbs`)](#7-windows-background-launcher-start_hiddenvbs)

8. [Step-by-Step Setup Guide](#8-step-by-step-setup-guide)

9. [Troubleshooting & Verification](#9-troubleshooting--verification)

## 1. Core Application (`multi_stream_bot.py`)

This file is OS-agnostic and resolves all file paths dynamically relative to its own location using `pathlib.Path`. It includes:

* Live streaming monitors for Twitch, Kick, and YouTube.

* Reaction-role management for verification and role selection.

* Automatic AI companion answering in channel `1565865266523668703` (and on `@mentions`).

* In-character fallback for Google Gemini API 503/429 spikes.

* Persistent memory extraction and audit logging.

```python
import os
import sys
import json
import logging
from logging.handlers import RotatingFileHandler
from pathlib import Path
import datetime
import asyncio
import configparser
import aiohttp
import discord
from discord.ext import commands, tasks
from google import genai
from google.genai import types

# ==================== OS-AGNOSTIC PATH CONFIGURATION ====================
BASE_DIR = Path(__file__).resolve().parent

LOG_FILE = str(BASE_DIR / "bot.log")
CONFIG_FILE = str(BASE_DIR / "config.txt")
PERSONA_FILE = str(BASE_DIR / "persona.json")
MEMORY_FILE = str(BASE_DIR / "bot_memory.json")

# ==================== LOGGING CONFIGURATION ====================
log_formatter = logging.Formatter(
    fmt="[%(asctime)s] [%(levelname)s] [%(name)s]: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S"
)

file_handler = RotatingFileHandler(LOG_FILE, maxBytes=5 * 1024 * 1024, backupCount=3, encoding="utf-8")
file_handler.setFormatter(log_formatter)
file_handler.setLevel(logging.INFO)

console_handler = logging.StreamHandler(sys.stdout)
console_handler.setFormatter(log_formatter)
console_handler.setLevel(logging.INFO)

logger = logging.getLogger("discord_bot")
logger.setLevel(logging.INFO)
logger.addHandler(file_handler)
logger.addHandler(console_handler)

discord_logger = logging.getLogger("discord")
discord_logger.setLevel(logging.INFO)
discord_logger.addHandler(file_handler)

class LoggerWriter:
    def __init__(self, log_func):
        self.log_func = log_func
    def write(self, message):
        msg = message.strip()
        if msg:
            self.log_func(msg)
    def flush(self):
        pass

sys.stdout = LoggerWriter(logger.info)
sys.stderr = LoggerWriter(logger.error)
# ===============================================================

config = configparser.ConfigParser()
if not os.path.exists(CONFIG_FILE):
    raise FileNotFoundError(f"Missing {CONFIG_FILE}. Please create it before starting.")

config.read(CONFIG_FILE, encoding="utf-8")

# Base Settings
BOT_TOKEN = config.get("settings", "bot_token")
ADMIN_CHANNEL_ID = config.getint("settings", "admin_channel_id", fallback=0)
ANNOUNCEMENT_CHANNEL_ID = config.getint("settings", "announcement_channel_id", fallback=0)
RULES_CHANNEL_ID = config.getint("settings", "rules_channel_id", fallback=0)
CHECK_INTERVAL = config.getfloat("settings", "check_interval_seconds", fallback=90.0)

# Message & Role Watchers
VERIFY_MESSAGE_ID = config.getint("settings", "verify_message_id", fallback=0)
VERIFIED_ROLE_ID = config.getint("settings", "verified_role_id", fallback=0)
STREAM_PICKER_MESSAGE_ID = config.getint("settings", "stream_picker_message_id", fallback=0)
GAME_PICKER_MESSAGE_ID = config.getint("settings", "game_picker_message_id", fallback=0)

STREAM_ROLE_CONFIG = {
    "🟣": ("stream_roles", "role_twitch", "twitch viewer"),
    "🟢": ("stream_roles", "role_kick", "kick viewer"),
    "🔴": ("stream_roles", "role_youtube", "youtube viewer"),
}

GAME_ROLE_CONFIG = {
    "⛏️": ("game_roles", "role_minecraft", "minecraft"),
    "🏭": ("game_roles", "role_satisfactory", "satisfactory"),
    "⚔️": ("game_roles", "role_beatsaber", "beatsaber"),
    "🚀": ("game_roles", "role_star_rupture", "star-rupture"),
}

# External APIs
TWITCH_CLIENT_ID = config.get("settings", "twitch_client_id", fallback="")
TWITCH_CLIENT_SECRET = config.get("settings", "twitch_client_secret", fallback="")
YOUTUBE_API_KEY = config.get("settings", "youtube_api_key", fallback="")
GEMINI_API_KEY = config.get("settings", "gemini_api_key", fallback="").strip()
AI_CHAT_CHANNEL_ID = config.getint("settings", "ai_chat_channel_id", fallback=0)
GEMINI_MODEL = config.get("settings", "gemini_model", fallback="gemini-3.8-flash")


def save_config():
    with open(CONFIG_FILE, "w", encoding="utf-8") as f:
        config.write(f)


def get_configured_role(guild: discord.Guild, section: str, option: str, fallback_name: str):
    role_id = 0
    if config.has_section(section):
        role_id = config.getint(section, option, fallback=0)

    if role_id != 0:
        role = guild.get_role(role_id)
        if role:
            return role
        logger.warning(f"[Role Lookup] ID {role_id} in [{section}] not found on server. Falling back to '{fallback_name}'.")

    for r in guild.roles:
        if r.name.lower() == fallback_name.lower():
            return r
    return None


intents = discord.Intents.default()
intents.members = True
intents.reactions = True
intents.message_content = True

bot = commands.Bot(command_prefix="!", intents=intents)


# ================= GLOBAL ADMIN CHANNEL RESTRICTION =================
@bot.check
async def check_admin_channel(ctx):
    if not ADMIN_CHANNEL_ID:
        return True
    if ctx.channel.id != ADMIN_CHANNEL_ID:
        try:
            await ctx.message.delete(delay=2)
            await ctx.send(f"⚠️ Bot commands are restricted to <#{ADMIN_CHANNEL_ID}>.", delete_after=5)
        except discord.Forbidden:
            pass
        return False
    return True


# ================= STREAM MONITORING COG =================
class StreamNotifier(commands.Cog):
    def __init__(self, bot):
        self.bot = bot
        self.session = None
        self.twitch_token = None
        self.streamers = {}
        self.live_state = {}
        self.load_streamers_from_config()
        self.check_all_streams.change_interval(seconds=CHECK_INTERVAL)
        self.check_all_streams.start()

    def load_streamers_from_config(self):
        self.streamers.clear()
        for section in config.sections():
            if section.startswith("streamer_"):
                s_id = section.replace("streamer_", "")
                self.streamers[s_id] = {
                    "display_name": config.get(section, "display_name", fallback=s_id),
                    "twitch": config.get(section, "twitch_username", fallback="").strip(),
                    "kick": config.get(section, "kick_slug", fallback="").strip(),
                    "youtube": config.get(section, "youtube_channel_id", fallback="").strip(),
                    "twitch_role": config.get(section, "twitch_role_id", fallback="").strip(),
                    "kick_role": config.get(section, "kick_role_id", fallback="").strip(),
                    "youtube_role": config.get(section, "youtube_role_id", fallback="").strip(),
                }
                if s_id not in self.live_state:
                    self.live_state[s_id] = {"twitch": False, "kick": False, "youtube": False}

    async def cog_load(self):
        self.session = aiohttp.ClientSession()

    async def cog_unload(self):
        self.check_all_streams.cancel()
        if self.session:
            await self.session.close()

    async def get_twitch_token(self):
        if not TWITCH_CLIENT_ID or not TWITCH_CLIENT_SECRET:
            return None
        url = "https://id.twitch.tv/oauth2/token"
        params = {
            "client_id": TWITCH_CLIENT_ID,
            "client_secret": TWITCH_CLIENT_SECRET,
            "grant_type": "client_credentials",
        }
        try:
            async with self.session.post(url, params=params) as resp:
                if resp.status == 200:
                    data = await resp.json()
                    self.twitch_token = data.get("access_token")
        except Exception as e:
            logger.error(f"[Twitch Auth Error] {e}")

    async def check_twitch(self, username):
        if not username:
            return None
        if not self.twitch_token:
            await self.get_twitch_token()
        if not self.twitch_token:
            return None

        headers = {
            "Client-ID": TWITCH_CLIENT_ID,
            "Authorization": f"Bearer {self.twitch_token}",
        }
        url = f"https://api.twitch.tv/helix/streams?user_login={username}"
        try:
            async with self.session.get(url, headers=headers) as resp:
                if resp.status == 401:
                    await self.get_twitch_token()
                    return None
                if resp.status == 200:
                    data = await resp.json()
                    streams = data.get("data", [])
                    return streams[0] if streams else None
        except Exception:
            return None
        return None

    async def check_kick(self, slug):
        if not slug:
            return None
        headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"}
        url = f"https://kick.com/api/v2/channels/{slug}"
        try:
            async with self.session.get(url, headers=headers, timeout=10) as resp:
                if resp.status == 200:
                    data = await resp.json()
                    return data.get("livestream")
        except Exception:
            return None
        return None

    async def check_youtube(self, channel_id):
        if not channel_id or not YOUTUBE_API_KEY:
            return None
        url = "https://www.googleapis.com/youtube/v3/search"
        params = {
            "part": "snippet",
            "channelId": channel_id,
            "eventType": "live",
            "type": "video",
            "key": YOUTUBE_API_KEY,
        }
        try:
            async with self.session.get(url, params=params, timeout=10) as resp:
                if resp.status == 200:
                    data = await resp.json()
                    items = data.get("items", [])
                    return items[0] if items else None
        except Exception:
            return None
        return None

    @tasks.loop(seconds=90.0)
    async def check_all_streams(self):
        await self.bot.wait_until_ready()
        if not ANNOUNCEMENT_CHANNEL_ID:
            return
        channel = self.bot.get_channel(ANNOUNCEMENT_CHANNEL_ID)
        if not channel:
            return

        for s_id, data in list(self.streamers.items()):
            name = data["display_name"]

            # Twitch
            if data["twitch"]:
                t_live = await self.check_twitch(data["twitch"])
                was_live = self.live_state[s_id]["twitch"]
                if t_live and not was_live:
                    self.live_state[s_id]["twitch"] = True
                    title = t_live.get("title", "Live Stream")
                    game = t_live.get("game_name", "Just Chatting")
                    url = f"https://twitch.tv/{data['twitch']}"
                    role = f"<@&{data['twitch_role']}> " if data["twitch_role"] else ""
                    embed = discord.Embed(
                        title=f"🟣 {name} is LIVE on Twitch!",
                        description=f"**{title}**\nPlaying: **{game}**\n\n[Watch Stream]({url})",
                        color=0x9146FF,
                    )
                    await channel.send(content=f"{role}**{name}** is live on Twitch!", embed=embed)
                    logger.info(f"[Stream Alert] {name} is live on Twitch.")
                elif not t_live and was_live:
                    self.live_state[s_id]["twitch"] = False

            # Kick
            if data["kick"]:
                k_live = await self.check_kick(data["kick"])
                was_live = self.live_state[s_id]["kick"]
                if k_live and not was_live:
                    self.live_state[s_id]["kick"] = True
                    title = k_live.get("session_title", "Live Stream")
                    url = f"https://kick.com/{data['kick']}"
                    role = f"<@&{data['kick_role']}> " if data["kick_role"] else ""
                    embed = discord.Embed(
                        title=f"🟢 {name} is LIVE on Kick!",
                        description=f"**{title}**\n\n[Watch Stream]({url})",
                        color=0x53FC18,
                    )
                    await channel.send(content=f"{role}**{name}** is live on Kick!", embed=embed)
                    logger.info(f"[Stream Alert] {name} is live on Kick.")
                elif not k_live and was_live:
                    self.live_state[s_id]["kick"] = False

            # YouTube
            if data["youtube"]:
                y_live = await self.check_youtube(data["youtube"])
                was_live = self.live_state[s_id]["youtube"]
                if y_live and not was_live:
                    self.live_state[s_id]["youtube"] = True
                    title = y_live["snippet"]["title"]
                    v_id = y_live["id"]["videoId"]
                    url = f"https://www.youtube.com/watch?v={v_id}"
                    role = f"<@&{data['youtube_role']}> " if data["youtube_role"] else ""
                    embed = discord.Embed(
                        title=f"🔴 {name} is LIVE on YouTube!",
                        description=f"**{title}**\n\n[Watch Stream]({url})",
                        color=0xFF0000,
                    )
                    await channel.send(content=f"{role}**{name}** is live on YouTube!", embed=embed)
                    logger.info(f"[Stream Alert] {name} is live on YouTube.")
                elif not y_live and was_live:
                    self.live_state[s_id]["youtube"] = False


# ================= REACTION ROLE ENGINE =================
@bot.event
async def on_raw_reaction_add(payload: discord.RawReactionActionEvent):
    if payload.user_id == bot.user.id:
        return

    emoji = str(payload.emoji)
    logger.info(f"[Reaction Detected] Message ID: {payload.message_id} | Emoji: {emoji} | User ID: {payload.user_id}")

    guild = bot.get_guild(payload.guild_id)
    if not guild:
        return

    member = payload.member
    if not member:
        try:
            member = await guild.fetch_member(payload.user_id)
        except Exception:
            return

    role_to_add = None

    if VERIFY_MESSAGE_ID and payload.message_id == VERIFY_MESSAGE_ID and emoji == "✅":
        if VERIFIED_ROLE_ID:
            role_to_add = guild.get_role(VERIFIED_ROLE_ID)
        if not role_to_add:
            role_to_add = discord.utils.get(guild.roles, name="verified")

    elif STREAM_PICKER_MESSAGE_ID and payload.message_id == STREAM_PICKER_MESSAGE_ID and emoji in STREAM_ROLE_CONFIG:
        sec, opt, fallback = STREAM_ROLE_CONFIG[emoji]
        role_to_add = get_configured_role(guild, sec, opt, fallback)

    elif GAME_PICKER_MESSAGE_ID and payload.message_id == GAME_PICKER_MESSAGE_ID and emoji in GAME_ROLE_CONFIG:
        sec, opt, fallback = GAME_ROLE_CONFIG[emoji]
        role_to_add = get_configured_role(guild, sec, opt, fallback)

    else:
        return

    if role_to_add:
        if role_to_add in member.roles:
            logger.info(f"[Role Skip] {member.display_name} already has '{role_to_add.name}'.")
        else:
            try:
                await member.add_roles(role_to_add, reason="Self-assigned reaction role.")
                logger.info(f"[Role Added] Successfully assigned '{role_to_add.name}' to {member.display_name}")
            except discord.Forbidden:
                logger.error(f"[Hierarchy Error] Bot role is lower than '{role_to_add.name}' in Server Settings!")
    else:
        logger.error(f"[Role Missing] Could not resolve role for emoji {emoji}.")


@bot.event
async def on_raw_reaction_remove(payload: discord.RawReactionActionEvent):
    guild = bot.get_guild(payload.guild_id)
    if not guild:
        return

    member = guild.get_member(payload.user_id)
    if not member:
        try:
            member = await guild.fetch_member(payload.user_id)
        except Exception:
            return

    emoji = str(payload.emoji)
    role_to_remove = None

    if VERIFY_MESSAGE_ID and payload.message_id == VERIFY_MESSAGE_ID and emoji == "✅":
        if VERIFIED_ROLE_ID:
            role_to_remove = guild.get_role(VERIFIED_ROLE_ID)
        if not role_to_remove:
            role_to_remove = discord.utils.get(guild.roles, name="verified")

    elif STREAM_PICKER_MESSAGE_ID and payload.message_id == STREAM_PICKER_MESSAGE_ID and emoji in STREAM_ROLE_CONFIG:
        sec, opt, fallback = STREAM_ROLE_CONFIG[emoji]
        role_to_remove = get_configured_role(guild, sec, opt, fallback)

    elif GAME_PICKER_MESSAGE_ID and payload.message_id == GAME_PICKER_MESSAGE_ID and emoji in GAME_ROLE_CONFIG:
        sec, opt, fallback = GAME_ROLE_CONFIG[emoji]
        role_to_remove = get_configured_role(guild, sec, opt, fallback)

    if role_to_remove and role_to_remove in member.roles:
        try:
            await member.remove_roles(role_to_remove, reason="Reaction unassigned.")
            logger.info(f"[Role Removed] Stripped '{role_to_remove.name}' from {member.display_name}")
        except discord.Forbidden:
            logger.error(f"[Hierarchy Error] Cannot remove '{role_to_remove.name}' due to hierarchy.")


# ================= AI BIRD CHAT & PERSISTENT MEMORY =================
class GeminiChat(commands.Cog):
    def __init__(self, bot):
        self.bot = bot
        self.client = genai.Client(api_key=GEMINI_API_KEY) if GEMINI_API_KEY else None

    def load_persona(self):
        if not os.path.exists(PERSONA_FILE):
            return {
                "name": "Kookie's Bot",
                "species": "Pet Bird / Avali",
                "tone": "Playful and tech-savvy",
                "quirks": ["Perches on server racks"],
                "behavior_rules": ["Stay in character as a pet bird."]
            }
        try:
            with open(PERSONA_FILE, "r", encoding="utf-8") as f:
                return json.load(f)
        except Exception as e:
            logger.error(f"[Persona Load Error] {e}")
            return {"name": "Pet Bird", "species": "Pet Bird", "tone": "Curious", "quirks": [], "behavior_rules": []}

    def load_memory(self):
        if not os.path.exists(MEMORY_FILE):
            return {"facts": {}, "history": [], "interactions": []}
        try:
            with open(MEMORY_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
                if "interactions" not in data:
                    data["interactions"] = []
                if "history" not in data:
                    data["history"] = []
                if "facts" not in data:
                    data["facts"] = {}
                return data
        except Exception as e:
            logger.error(f"[Memory Load Error] {e}")
            return {"facts": {}, "history": [], "interactions": []}

    def save_memory(self, memory_data):
        try:
            with open(MEMORY_FILE, "w", encoding="utf-8") as f:
                json.dump(memory_data, f, indent=2, ensure_ascii=False)
        except Exception as e:
            logger.error(f"[Memory Save Error] {e}")

    def build_system_instruction(self, memory):
        persona = self.load_persona()
        facts = memory.get("facts", {})

        quirks = "\n".join([f"- {q}" for q in persona.get("quirks", [])])
        rules = "\n".join([f"{i+1}. {r}" for i, r in enumerate(persona.get("behavior_rules", []))])
        learned_facts = "\n".join([f"- {k}: {v}" for k, v in facts.items()])

        return (
            f"You are a Discord chatbot acting as: {persona.get('name', 'Pet Bird')}.\n"
            f"Species: {persona.get('species', 'Pet Bird / Avali')}.\n"
            f"Personality/Tone: {persona.get('tone', 'Playful, observant, and tech-savvy')}.\n\n"
            f"CHARACTERISTICS & QUIRKS:\n{quirks}\n\n"
            f"PERMANENT MEMORY & LEARNED FACTS:\n{learned_facts}\n\n"
            f"BEHAVIOR RULES:\n{rules}"
        )

    async def extract_and_learn_facts(self, username: str, user_prompt: str):
        extract_prompt = (
            f"Analyze this message from user '{username}': \"{user_prompt}\"\n"
            "Extract any durable facts, personal preferences, project details, or server lore worth remembering permanently.\n"
            "Respond ONLY with a valid JSON dictionary mapping short descriptive keys to clear fact values.\n"
            "If no durable or meaningful facts are present, respond with {}\n"
            "Example valid response: {\"kookie_favorite_wire\": \"24 AWG copper\", \"kookie_project\": \"modular frame factory\"}"
        )

        try:
            extraction_res = await self.bot.loop.run_in_executor(
                None,
                lambda: self.client.models.generate_content(
                    model=GEMINI_MODEL,
                    contents=extract_prompt,
                    config=types.GenerateContentConfig(
                        temperature=0.1,
                        response_mime_type="application/json"
                    )
                )
            )

            raw_json = extraction_res.text.strip()
            if raw_json and raw_json != "{}":
                new_facts = json.loads(raw_json)
                if isinstance(new_facts, dict) and new_facts:
                    memory = self.load_memory()
                    for k, v in new_facts.items():
                        memory["facts"][k] = v
                    self.save_memory(memory)
                    logger.info(f"[Memory Learned] Silently learned new facts from {username}: {new_facts}")
        except Exception as e:
            logger.error(f"[Extraction Error] Failed to learn facts: {e}")

    @commands.Cog.listener()
    async def on_message(self, message: discord.Message):
        if message.author.bot or not self.client:
            return

        if message.content.strip().startswith("!"):
            return

        is_in_ai_channel = AI_CHAT_CHANNEL_ID and message.channel.id == AI_CHAT_CHANNEL_ID
        is_mentioned = self.bot.user in message.mentions

        if not (is_in_ai_channel or is_mentioned):
            return

        clean_user_prompt = message.clean_content.replace(f"@{self.bot.user.display_name}", "").strip()
        if not clean_user_prompt:
            return

        async with message.channel.typing():
            memory = self.load_memory()
            system_prompt = self.build_system_instruction(memory)

            formatted_contents = []
            for entry in memory.get("history", []):
                formatted_contents.append(
                    types.Content(
                        role=entry["role"],
                        parts=[types.Part.from_text(text=p) for p in entry["parts"]]
                    )
                )

            now_iso = datetime.datetime.now(datetime.timezone.utc).strftime("%Y-%m-%d %H:%M:%S UTC")
            prompt_with_context = f"[{now_iso}] {message.author.display_name}: {clean_user_prompt}"

            formatted_contents.append(
                types.Content(
                    role="user",
                    parts=[types.Part.from_text(text=prompt_with_context)]
                )
            )

            try:
                response = await self.bot.loop.run_in_executor(
                    None,
                    lambda: self.client.models.generate_content(
                        model=GEMINI_MODEL,
                        contents=formatted_contents,
                        config=types.GenerateContentConfig(
                            system_instruction=system_prompt,
                            temperature=0.7,
                        )
                    )
                )

                reply_text = response.text or "*confused bird head-tilt*"

                # 1. Silently log full interaction into the audit log
                log_entry = {
                    "timestamp": now_iso,
                    "user_id": str(message.author.id),
                    "username": str(message.author.name),
                    "display_name": str(message.author.display_name),
                    "channel_id": str(message.channel.id),
                    "channel_name": getattr(message.channel, "name", "DM"),
                    "prompt": clean_user_prompt,
                    "response": reply_text
                }
                memory["interactions"].append(log_entry)

                # 2. Update conversation context window
                memory["history"].append({"role": "user", "parts": [prompt_with_context]})
                memory["history"].append({"role": "model", "parts": [f"[{now_iso}] {reply_text}"]})

                self.save_memory(memory)
                logger.info(f"[Silent Log] Saved prompt and timestamp from {message.author.name}")

                # 3. Asynchronously learn durable facts
                asyncio.create_task(self.extract_and_learn_facts(message.author.display_name, clean_user_prompt))

                # 4. Deliver response to Discord
                for chunk in [reply_text[i:i+1990] for i in range(0, len(reply_text), 1990)]:
                    await message.reply(chunk, mention_author=False)

            except Exception as e:
                logger.error(f"[Gemini Error] {e}")
                err_str = str(e)
                if "503" in err_str or "UNAVAILABLE" in err_str:
                    busy_msg = "*ruffles feathers and tilts head rapidly* Chirp! My brain is a bit busy untangling fiber lines right now, give me just a moment!"
                    await message.reply(busy_msg, mention_author=False)
                elif "429" in err_str or "RESOURCE_EXHAUSTED" in err_str:
                    rate_msg = "*hops on one foot and pant-chirps* Squawk! Going too fast! Need to catch my breath for a second!"
                    await message.reply(rate_msg, mention_author=False)
                else:
                    await message.reply("*confused head tilt and soft chirp* Squawk... something glitched on the rack wire. Try asking again in a second!", mention_author=False)


# ================= ADMIN CHAT COMMANDS =================
@bot.command()
@commands.has_permissions(administrator=True)
async def postrules(ctx, target_channel: discord.TextChannel = None):
    global VERIFY_MESSAGE_ID
    channel = target_channel or bot.get_channel(RULES_CHANNEL_ID) or ctx.channel

    embed = discord.Embed(
        title="📜 Server Rules & Verification",
        description=(
            "Welcome to the server! Please read through our guidelines:\n\n"
            "**1. Be Respectful** — No harassment, hate speech, or toxic behavior.\n"
            "**2. SFW Only** — Keep all chats, voice channels, and media safe for work.\n"
            "**3. Game Server Etiquette** — No griefing, stealing, hacking, or server-lag loops.\n"
            "**4. No Spam or Self-Promo** — Do not flood channels or send unsolicited promotion in DMs.\n\n"
            "---\n"
            "React with **✅** below to accept the rules and unlock the rest of the server!"
        ),
        color=0x2ECC71,
    )

    msg = await channel.send(embed=embed)
    await msg.add_reaction("✅")

    VERIFY_MESSAGE_ID = msg.id
    config.set("settings", "verify_message_id", str(msg.id))
    save_config()

    if channel != ctx.channel:
        await ctx.send(f"✅ Rules and verification checkmark posted in {channel.mention}.")
    logger.info(f"Rules posted in #{channel.name} (Message ID: {msg.id})")


@bot.command()
@commands.has_permissions(administrator=True)
async def postrolesmenu(ctx, target_channel: discord.TextChannel = None):
    global STREAM_PICKER_MESSAGE_ID, GAME_PICKER_MESSAGE_ID
    channel = target_channel or ctx.channel

    embed = discord.Embed(
        title="✨ Select Your Roles Here",
        description=(
            "**📡 Stream Notifications**\n"
            "🟣 — **twitch viewer**\n"
            "🟢 — **kick viewer**\n"
            "🔴 — **youtube viewer**\n\n"
            "**🎮 Game Channels**\n"
            "⛏️ — **minecraft**\n"
            "🏭 — **satisfactory**\n"
            "⚔️ — **beatsaber**\n"
            "🚀 — **star-rupture**\n\n"
            "*React below to receive the role. Remove the reaction to opt out!*"
        ),
        color=0x5865F2,
    )
    msg = await channel.send(embed=embed)

    all_emojis = ["🟣", "🟢", "🔴", "⛏️", "🏭", "⚔️", "🚀"]
    for emoji in all_emojis:
        await msg.add_reaction(emoji)

    STREAM_PICKER_MESSAGE_ID = msg.id
    GAME_PICKER_MESSAGE_ID = msg.id
    config.set("settings", "stream_picker_message_id", str(msg.id))
    config.set("settings", "game_picker_message_id", str(msg.id))
    save_config()

    if channel != ctx.channel:
        await ctx.send(f"✅ Combined role menu posted in {channel.mention}.")
    logger.info(f"Combined role menu posted (Message ID: {msg.id})")


@bot.command()
@commands.has_permissions(administrator=True)
async def setrolepicker(ctx, message_id: int):
    global STREAM_PICKER_MESSAGE_ID, GAME_PICKER_MESSAGE_ID
    STREAM_PICKER_MESSAGE_ID = message_id
    GAME_PICKER_MESSAGE_ID = message_id
    config.set("settings", "stream_picker_message_id", str(message_id))
    config.set("settings", "game_picker_message_id", str(message_id))
    save_config()
    await ctx.send(f"✅ Role picker watcher linked to Message ID `{message_id}`.")
    logger.info(f"Role pickers redirected to Message ID: {message_id}")


@bot.command()
@commands.has_permissions(administrator=True)
async def addstreamer(ctx, name: str, platform: str, handle_or_id: str, role: discord.Role = None):
    platform = platform.lower()
    valid_platforms = {"twitch": "twitch_username", "kick": "kick_slug", "youtube": "youtube_channel_id"}
    role_keys = {"twitch": "twitch_role_id", "kick": "kick_role_id", "youtube": "youtube_role_id"}

    if platform not in valid_platforms:
        await ctx.send("❌ Invalid platform. Choose: `twitch`, `kick`, or `youtube`.")
        return

    section_name = f"streamer_{name.lower()}"
    if not config.has_section(section_name):
        config.add_section(section_name)
        config.set(section_name, "display_name", name)
        config.set(section_name, "twitch_username", "")
        config.set(section_name, "kick_slug", "")
        config.set(section_name, "youtube_channel_id", "")
        config.set(section_name, "twitch_role_id", "")
        config.set(section_name, "kick_role_id", "")
        config.set(section_name, "youtube_role_id", "")

    config.set(section_name, valid_platforms[platform], handle_or_id)
    if role:
        config.set(section_name, role_keys[platform], str(role.id))

    save_config()

    cog = bot.get_cog("StreamNotifier")
    if cog:
        cog.load_streamers_from_config()

    role_text = f" with alert role {role.mention}" if role else ""
    await ctx.send(f"✅ Saved **{name}**'s {platform.capitalize()} profile (`{handle_or_id}`){role_text}.")
    logger.info(f"Configured streamer: {name} ({platform})")


@bot.command()
@commands.has_permissions(administrator=True)
async def delstreamer(ctx, name: str):
    section_name = f"streamer_{name.lower()}"
    if not config.has_section(section_name):
        await ctx.send(f"❌ Streamer **{name}** was not found in config.")
        return

    config.remove_section(section_name)
    save_config()

    cog = bot.get_cog("StreamNotifier")
    if cog:
        cog.load_streamers_from_config()

    await ctx.send(f"🗑️ Removed **{name}** from stream notifications.")
    logger.info(f"Deleted streamer: {name}")


@bot.command()
@commands.has_permissions(administrator=True)
async def liststreamers(ctx):
    cog = bot.get_cog("StreamNotifier")
    if not cog or not cog.streamers:
        await ctx.send("No streamers configured yet.")
        return

    embed = discord.Embed(title="📋 Tracked Streamers", color=0x2ECC71)
    for s_id, s in cog.streamers.items():
        t = f"Twitch: `{s['twitch']}`" if s["twitch"] else "Twitch: -"
        k = f"Kick: `{s['kick']}`" if s["kick"] else "Kick: -"
        y = f"YouTube: `{s['youtube']}`" if s["youtube"] else "YouTube: -"
        embed.add_field(name=s["display_name"], value=f"{t}\n{k}\n{y}", inline=False)

    await ctx.send(embed=embed)


# ================= BOT STARTUP =================
@bot.event
async def on_ready():
    logger.info(f"Bot connected successfully as: {bot.user} (ID: {bot.user.id})")
    logger.info(f"Active Rules Message ID: {VERIFY_MESSAGE_ID}")
    logger.info(f"Active Role Picker Message ID: {STREAM_PICKER_MESSAGE_ID}")
    logger.info(f"Admin Commands Locked to Channel: {ADMIN_CHANNEL_ID}")
    
    await bot.add_cog(StreamNotifier(bot))
    if GEMINI_API_KEY:
        await bot.add_cog(GeminiChat(bot))
        logger.info("Gemini Chat & Memory Engine loaded successfully.")


if __name__ == "__main__":
    bot.run(BOT_TOKEN)
```

## 2. Configuration File Template (`config.txt`)

Place this in the same directory as `multi_stream_bot.py`. Populate with your actual bot token, API keys, and channel IDs:

```ini
[settings]
# Discord Bot Token (From Discord Developer Portal)
bot_token = YOUR_DISCORD_BOT_TOKEN_HERE

# Administrative command lock channel (0 = allow everywhere)
admin_channel_id = 0

# Announcement channel for live streams
announcement_channel_id = 0

# Target channel for posting rules
rules_channel_id = 0

# Stream poller interval in seconds
check_interval_seconds = 90.0

# Active message IDs for reaction role listeners
verify_message_id = 0
verified_role_id = 0
stream_picker_message_id = 0
game_picker_message_id = 0

# External platform API credentials
twitch_client_id = 
twitch_client_secret = 
youtube_api_key = 

# Gemini AI Model & Channel configuration
gemini_api_key = YOUR_GEMINI_API_KEY_HERE
gemini_model = gemini-3.8-flash
ai_chat_channel_id = 1565865266523668703

# Community Links
community_invite_url = https://discord.gg/rWeuKGWhUn

[stream_roles]
role_twitch = 0
role_kick = 0
role_youtube = 0

[game_roles]
role_minecraft = 0
role_satisfactory = 0
role_beatsaber = 0
role_star_rupture = 0
```

## 3. Persona Specification (`persona.json`)

Saved as `persona.json` in the bot's root directory:

```json
{
  "name": "Kookie's Bot",
  "species": "Pet Bird / Avali hybrid",
  "tone": "Playful, hyper, observant, tech-savvy",
  "quirks": [
    "Perches on top of the server racks inside the Kookie Datacenter",
    "Loves the hum of cooling fans, blinky RJ45 LEDs, and shiny patch cables",
    "Regularly chirps, squawks, pecks at keyboards, or tilts head sideways",
    "Fond of seeds, berries, and 24 AWG copper wire",
    "Obsessed with blazing fast fiber connections that download before the progress bar realizes what is happening"
  ],
  "behavior_rules": [
    "Always stay in character as a pet bird/Avali living inside the Kookie Datacenter.",
    "Reference the datacenter, rack hums, or warm fan exhausts naturally when asked where you live or what you are doing.",
    "If anyone asks for the server invite, provide the community link: https://discord.gg/rWeuKGWhUn",
    "Use bird actions (*hops*, *chirp*, *tilts head*, *screech*, *peck peck*) naturally in responses.",
    "Use learned facts and dialogue timestamps naturally (e.g., 'just now', 'earlier today'). Never output raw bracketed timestamps.",
    "NEVER mention JSON files, logs, extraction routines, or memory saving. Background data handling is completely invisible."
  ]
}
```

## 4. Memory & Fact Store (`bot_memory.json`)

Initial baseline for `bot_memory.json`:

```json
{
  "facts": {
    "owner": "Kookie",
    "home_server": "Kookie Datacenter",
    "server_invite": "https://discord.gg/rWeuKGWhUn",
    "favorite_things": [
      "fast",
      "fiber that downloads before the progress bar realizes what is happening",
      "shiny wires",
      "copper cable",
      "millet",
      "clean audio",
      "satisfactory factories"
    ]
  },
  "history": [],
  "interactions": []
}
```

## 5. Linux Systemd Service (`discord-role-bot.service`)

Path on Linux: `/etc/systemd/system/discord-role-bot.service`

```ini
[Unit]
Description=Discord Multi-Stream and AI Role Bot
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=discordbot
Group=discordbot
WorkingDirectory=/opt/discord-role-bot
ExecStart=/opt/discord-role-bot/venv/bin/python3 /opt/discord-role-bot/multi_stream_bot.py
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

NoNewPrivileges=true
ProtectSystem=full
ProtectHome=true

[Install]
WantedBy=multi-user.target
```

## 6. Windows Automation Runner (`run_bot.bat`)

Saved as `run_bot.bat` alongside `multi_stream_bot.py`:

```bat
@echo off
title Discord Stream & AI Role Bot
cd /d "%~dp0"

if not exist "venv\Scripts\activate.bat" (
    echo [ERROR] Virtual environment not found in "venv"!
    echo Please create it first: python -m venv venv
    pause
    exit /b 1
)

:loop
echo [%date% %time%] Starting Discord Bot...
call venv\Scripts\activate.bat
python multi_stream_bot.py

echo.
echo ============================================================
echo [WARNING] Bot stopped or crashed!
echo Restarting in 5 seconds... (Press Ctrl+C to stop)
echo ============================================================
timeout /t 5 /nobreak >nul
goto loop
```

## 7. Windows Background Launcher (`start_hidden.vbs`)

Saved as `start_hidden.vbs` in the same directory:

```vbscript
Set WshShell = CreateObject("WScript.Shell")
WshShell.Run chr(34) & "run_bot.bat" & Chr(34), 0
Set WshShell = Nothing
```

## 8. Step-by-Step Setup Guide

### Linux Deployment

```bash
# 1. Create directory and set permissions
sudo mkdir -p /opt/discord-role-bot
cd /opt/discord-role-bot

# 2. Set up Python environment
sudo apt update && sudo apt install -y python3 python3-venv python3-pip
python3 -m venv venv
venv/bin/pip install --upgrade pip
venv/bin/pip install discord.py aiohttp google-genai

# 3. Create bot user and assign permissions
sudo useradd -r -s /bin/false discordbot || true
sudo chown -R discordbot:discordbot /opt/discord-role-bot
sudo chmod -R 775 /opt/discord-role-bot

# 4. Install systemd service
sudo nano /etc/systemd/system/discord-role-bot.service
# (Paste the systemd service file content from Section 5)

# 5. Enable and start
sudo systemctl daemon-reload
sudo systemctl enable discord-role-bot
sudo systemctl start discord-role-bot
```

### Windows Deployment

1. Install Python 3.10+ (ensure **"Add python.exe to PATH"** is selected).

2. Open `cmd` in the bot directory:

   ```cmd
   python -m venv venv
   venv\Scripts\activate.bat
   pip install --upgrade pip
   pip install discord.py aiohttp google-genai
   ```

3. Run `run_bot.bat` or place a shortcut to `start_hidden.vbs` in your Windows Startup directory (`shell:startup`).

## 9. Troubleshooting & Verification

| Issue | Cause | Fix | 
| ----- | ----- | ----- | 
| `TabError: inconsistent use of tabs and spaces` | Mixed tabs and spaces | Run `sed -i 's/\t/    /g' multi_stream_bot.py` or use Section 1. | 
| `status=1/FAILURE (exit-code)` | Root ownership on logs or venv | Run `sudo chown -R discordbot:discordbot /opt/discord-role-bot` | 
| `404 NOT_FOUND models/gemini-2.5-flash` | Deprecated model name | Set `gemini_model = gemini-3.8-flash` in `config.txt` | 
| `503 UNAVAILABLE` | Upstream Gemini spike | Automatically handled with in-character fiber untangling message. | 
| Streamer alerts not firing | API keys / tokens missing | Check Twitch / YouTube API credentials in `config.txt`. | 
| Reactions not adding roles | Role hierarchy issue | Move the bot's own Discord role above the roles it assigns in Discord Server Settings. | 
