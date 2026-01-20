import discord
from discord.ext import commands, tasks
import asyncio
import json
import os
import difflib
from discord import Permissions
import itertools
import random
from discord.ui import View, Button, Select, Modal, TextInput
from discord import app_commands
from datetime import timedelta

TOKEN = "MTQwMTkxMzUyNTMyNDA4NzMyNg.G9rvRh.LTcgsqqCqJh2i-V_uZVOZO1nwnZO15rhPHdylw"
ping_on_join_channels = {} 
CHANNELS_FILE = "ping_channels.json"
WHITELIST = {1372180385663815755, 1373297212234137715, 1435184908313301049, 1401913525324087326}

intents = discord.Intents.all()
intents.message_content = True
intents.members = True 
intents.guilds = True
intents.voice_states = True
bot = commands.Bot(command_prefix=".", intents=intents, help_command=None)
tree = bot.tree

def is_whitelisted(user: discord.User):
    return user.id in WHITELIST

def can_manage(interaction: discord.Interaction):
    return interaction.user.id in WHITELIST

def save_ping_channels():
    with open(CHANNELS_FILE, "w") as f:
        json.dump(ping_on_join_channels, f)

def load_ping_channels():
    global ping_on_join_channels
    if os.path.exists(CHANNELS_FILE):
        with open(CHANNELS_FILE, "r") as f:
            try:
                ping_on_join_channels = json.load(f)
                ping_on_join_channels = {str(k): v for k, v in ping_on_join_channels.items()}
            except json.JSONDecodeError:
                ping_on_join_channels = {}

load_ping_channels()

CONFIG_FILE = "config.json"
if os.path.exists(CONFIG_FILE):
    with open(CONFIG_FILE, "r") as f:
        channel_config = json.load(f)
else:
    channel_config = {}

ping_on_join_channels = {}


JOIN_LOG_FILE = "join_log_channels.json"

def load_join_logs():
    if os.path.exists(JOIN_LOG_FILE):
        with open(JOIN_LOG_FILE, "r") as f:
            return json.load(f)
    return {}

def save_join_logs():
    with open(JOIN_LOG_FILE, "w") as f:
        json.dump(join_log_channels, f, indent=4)

join_log_channels = load_join_logs()


R2V_FILE = "r2v_data.json"

def load_r2v_data():
    if os.path.exists(R2V_FILE):
        with open(R2V_FILE, "r") as f:
            try:
                return json.load(f)
            except json.JSONDecodeError:
                return {}
    return {}

def save_r2v_data():
    with open(R2V_FILE, "w") as f:
        json.dump(r2v_data, f, indent=4)

r2v_data = load_r2v_data()

autoroles = {}

def save_autoroles():

    pass

AUTOROLES_FILE = "autoroles.json"


if os.path.exists(AUTOROLES_FILE):
    with open(AUTOROLES_FILE, "r") as f:
        autoroles = json.load(f)

        autoroles = {int(k): v for k, v in autoroles.items()}
else:
    autoroles = {}

def save_autoroles():
    with open(AUTOROLES_FILE, "w") as f:
        json.dump(autoroles, f)

PERMS_FILE = "saved_perms.json"
DANGEROUS_PERMS = [  
    "administrator",
    "ban_members",
    "kick_members",
    "manage_channels",
    "manage_guild",
    "manage_roles",
    "manage_messages",
    "mention_everyone",
    "manage_webhooks",
    "manage_emojis_and_stickers"
]
saved_perms = {}
try:
    with open(PERMS_FILE, "r") as f:
        saved_perms = json.load(f)
        saved_perms = {int(k): v for k, v in saved_perms.items()}
except FileNotFoundError:
    saved_perms = {}

@bot.event
async def on_member_join(member: discord.Member):
    guild_id = member.guild.id
    
    channel_id = channel_config.get(guild_id, {}).get("welcome")
    channel = bot.get_channel(channel_id) if channel_id else None

    if channel:
        member_count = member.guild.member_count
        position_str = f"{member_count:,}"

        embed = discord.Embed(
            description=(
                f"{member.mention} welcome to {member.guild.name}\n\n"
                f"· You are our {position_str}th member\n"
                f"· Get active in the chat and boost the server\n"
                f"· Enjoy your stay"
            ),
            color=discord.Color.dark_theme()
        )

        embed.set_author(
            name=member.guild.name,
            icon_url=member.guild.icon.url if member.guild.icon else None
        )

        embed.set_thumbnail(
            url=member.avatar.url if member.avatar else member.default_avatar.url
        )

        await channel.send(embed=embed)

    channels = ping_on_join_channels.get(guild_id, [])
    if channels:
        send_tasks = []
        for c_id in channels:
            ch = member.guild.get_channel(c_id)
            if ch:
                send_tasks.append(ch.send(f"{member.mention}"))

        sent_messages = await asyncio.gather(*send_tasks, return_exceptions=True)

        await asyncio.sleep(5)
        delete_tasks = [
            msg.delete()
            for msg in sent_messages
            if isinstance(msg, discord.Message)
        ]
        await asyncio.gather(*delete_tasks, return_exceptions=True)

    if guild_id in join_log_channels:
        log_channel = bot.get_channel(join_log_channels[guild_id])
        if log_channel:
            embed = discord.Embed(
                title="Join Logs",
                description=f"{member.mention} has joined the server.",
                color=discord.Color.dark_theme()
            )
            embed.set_author(name=member.name, icon_url=member.display_avatar.url)
            view = JoinLogView(member.id)
            await log_channel.send(embed=embed, view=view)

    roles_to_add = [
        member.guild.get_role(rid)
        for rid in autoroles.get(guild_id, [])
        if member.guild.get_role(rid)
    ]

    if roles_to_add:
        try:
            await member.add_roles(*roles_to_add, reason="Autorole on join")
        except discord.Forbidden:
            print(f"Missing permission to add one or more autoroles to {member.name}")


@bot.event
async def on_member_remove(member: discord.Member):
    guild_id = str(member.guild.id)
    channel_id = channel_config.get(guild_id, {}).get("goodbye")
    channel = bot.get_channel(channel_id) if channel_id else None
    if not channel:
        return

    member_count = member.guild.member_count + 1
    position_str = f"{member_count:,}"

    max_roles_display = 3
    sorted_roles = sorted(
        [role for role in member.roles if role != member.guild.default_role],
        key=lambda r: r.position,
        reverse=True
    )
    displayed_roles = [role.mention for role in sorted_roles[:max_roles_display]]
    roles_text = "\n".join(displayed_roles) if displayed_roles else "No roles"

    embed = discord.Embed(
        description=(
            f"{member.mention} left\n\n"
            f"· Goodbye :(\n"
            f"· You were our {position_str}th member\n"
            f"· Roles they had\n\n"
            f"{roles_text}"
        ),
        color=discord.Color.dark_embed()
    )

    embed.set_author(
        name=member.guild.name,
        icon_url=member.guild.icon.url if member.guild.icon else None
    )

    embed.set_thumbnail(
        url=member.avatar.url if member.avatar else member.default_avatar.url
    )

    try:
        await channel.send(embed=embed)
    except Exception as e:
        print(f"[!] Failed to send goodbye embed: {e}")



async def strip_roles(member: discord.Member):
    """Remove all roles from a member (except @everyone)."""
    try:
        roles_to_remove = [r for r in member.roles if r != member.guild.default_role]
        await member.remove_roles(*roles_to_remove, reason="Anti-Nuke punishment")
        print(f"Stripped roles from {member}")
    except Exception as e:
        print(f"Failed to strip roles from {member}: {e}")


@bot.event
async def on_guild_channel_delete(channel):
    guild = channel.guild
    async for entry in guild.audit_logs(limit=1, action=discord.AuditLogAction.channel_delete):
        if entry.user.id not in WHITELIST:
            await strip_roles(entry.user)
            new_channel = await guild.create_text_channel(name=channel.name, category=channel.category)
            await new_channel.edit(position=channel.position)


@bot.event
async def on_guild_channel_create(channel):
    guild = channel.guild
    async for entry in guild.audit_logs(limit=1, action=discord.AuditLogAction.channel_create):
        if entry.user.id not in WHITELIST:
            await strip_roles(entry.user)
            await channel.delete()

@bot.event
async def on_guild_channel_update(before, after):
    guild = after.guild
    async for entry in guild.audit_logs(limit=1, action=discord.AuditLogAction.channel_update):
        if entry.user.id not in WHITELIST:
            await strip_roles(entry.user)
            try:
                await after.edit(
                    name=before.name,
                    topic=before.topic,
                    nsfw=before.nsfw,
                    position=before.position,
                    slowmode_delay=getattr(before, "slowmode_delay", 0),
                    category=before.category
                )
                
                await after.edit(overwrites=before.overwrites)
            except Exception as e:
                print(f"Failed to revert channel {after.id}: {e}")


@bot.event
async def on_guild_role_delete(role):
    guild = role.guild
    async for entry in guild.audit_logs(limit=1, action=discord.AuditLogAction.role_delete):
        if entry.user.id not in WHITELIST:
            await strip_roles(entry.user)
            await guild.create_role(name=role.name, permissions=role.permissions, colour=role.colour)


@bot.event
async def on_guild_role_create(role):
    guild = role.guild
    async for entry in guild.audit_logs(limit=1, action=discord.AuditLogAction.role_create):
        if entry.user.id not in WHITELIST:
            await strip_roles(entry.user)
            await role.delete()

@bot.event
async def on_guild_role_update(before, after):
    guild = after.guild
    async for entry in guild.audit_logs(limit=1, action=discord.AuditLogAction.role_update):
        if entry.user.id not in WHITELIST:
            await strip_roles(entry.user)
            try:
                await after.edit(
                    name=before.name,
                    color=before.color,
                    permissions=before.permissions,
                    hoist=before.hoist,
                    mentionable=before.mentionable,
                    position=before.position
                )
            except Exception as e:
                print(f"Failed to revert role {after.id}: {e}")

@bot.event
async def on_guild_bot_add(member):
    if member.bot:
        guild = member.guild
        async for entry in guild.audit_logs(limit=1, action=discord.AuditLogAction.bot_add):
            if entry.user.id not in WHITELIST:
                await strip_roles(entry.user)
                try:
                    await member.kick(reason="Unauthorized bot added")
                except Exception as e:
                    print(f"Failed to kick bot {member.id}: {e}")

@bot.event
async def on_webhooks_update(channel):
    guild = channel.guild
    async for entry in guild.audit_logs(limit=1, action=discord.AuditLogAction.webhook_create):
        if entry.user.id not in WHITELIST:
            await strip_roles(entry.user)
            webhooks = await channel.webhooks()
            for webhook in webhooks:
                if webhook.user == entry.user:
                    try:
                        await webhook.delete(reason="Unauthorized webhook creation")
                    except Exception as e:
                        print(f"Failed to delete webhook {webhook.id}: {e}")



@bot.event
async def on_member_ban(guild, user):
    async for entry in guild.audit_logs(limit=1, action=discord.AuditLogAction.ban):
        if entry.user.id not in WHITELIST:
            await strip_roles(entry.user)
            await guild.unban(user)

@bot.event
async def on_member_kick(member):
    guild = member.guild
    async for entry in guild.audit_logs(limit=1, action=discord.AuditLogAction.kick):
        if entry.user.id not in WHITELIST:
            await strip_roles(entry.user)
            
CALL_CATEGORY_NAME = "Voice Calls"
J2C_CHANNEL_NAME = "Create a Call"
user_voice_channels = {}

done_emoji = "<:done:1414337207455580231>"
invalid_emoji = "<:invalid:1414337316524134592>"

def embed_response(ctx, message: str):
    return discord.Embed(
        description=f"{done_emoji} {ctx.author.mention} {message}",
        color=discord.Color(0x56f287)
    )

def error_response(ctx, message: str):
    return discord.Embed(
        description=f"{invalid_emoji} {ctx.author.mention} {message}",
        color=discord.Color(0xfee75d)
    )

@bot.command(aliases=["vm"])
async def voicemaster(ctx, arg: str = None):
    if not arg:
        return await ctx.send(embed=error_response(ctx, "Use: `.voicemaster setup` or `.voicemaster delete`"))

    guild = ctx.guild
    category = discord.utils.get(guild.categories, name=CALL_CATEGORY_NAME)


    if arg.lower() == "setup":
        try:
            if not category:
                category = await guild.create_category(CALL_CATEGORY_NAME)

            j2c = discord.utils.get(category.voice_channels, name=J2C_CHANNEL_NAME)
            if not j2c:
                await guild.create_voice_channel(J2C_CHANNEL_NAME, category=category)

            await ctx.send(embed=embed_response(ctx, "Voicemaster has been setup"))
        except Exception as e:
            await ctx.send(embed=error_response(ctx, f"Failed to setup VoiceMaster: `{e}`"))
        return


    elif arg.lower() == "delete":
        if not category:
            return await ctx.send(embed=error_response(ctx, "Voicemaster hasnt been setup"))

        try:

            for channel in category.channels:
                await channel.delete()


            await category.delete()


            user_voice_channels.clear()

            await ctx.send(embed=embed_response(ctx, "Voicemaster has been deleted"))
        except Exception as e:
            await ctx.send(embed=error_response(ctx, f"Failed to delete category: `{e}`"))
        return

    else:
        await ctx.send(embed=error_response(ctx, "use either `setup` or `delete`"))

class RenameModal(Modal):
    def __init__(self, vc):
        super().__init__(title="Rename Voice Channel")
        self.vc = vc
        self.name = TextInput(label="New Channel Name", placeholder="Enter new name")
        self.add_item(self.name)

    async def on_submit(self, interaction: discord.Interaction):
        await self.vc.edit(name=self.name.value)
        await interaction.response.send_message(f"Renamed to **{self.name.value}**", ephemeral=True)

class UserLimitModal(Modal):
    def __init__(self, vc):
        super().__init__(title="Set User Limit")
        self.vc = vc
        self.limit = TextInput(label="Limit (0 = unlimited)", placeholder="Example: 6")
        self.add_item(self.limit)

    async def on_submit(self, interaction: discord.Interaction):
        try:
            limit = int(self.limit.value)
            await self.vc.edit(user_limit=limit)
            await interaction.response.send_message(f"User limit set to **{limit}**", ephemeral=True)
        except ValueError:
            await interaction.response.send_message("Invalid number", ephemeral=True)
        
class VoiceControlPanel(View):
    def __init__(self):
        super().__init__(timeout=None)

    async def interaction_check(self, interaction: discord.Interaction):
        vc_id = user_voice_channels.get(interaction.user.id)
        if not vc_id:
            await interaction.response.send_message(
                embed=discord.Embed(
                    description=f"{invalid_emoji} You don't own a voice call!",
                    color=discord.Color(0xfee75d)
                ),
                ephemeral=True
            )
            return False
        return True

    @discord.ui.button(emoji="<:lock:1436411274958340206>", style=discord.ButtonStyle.secondary, custom_id="vc_lock")
    async def lock(self, interaction: discord.Interaction, button: Button):
        vc = interaction.guild.get_channel(user_voice_channels.get(interaction.user.id))
        await vc.set_permissions(interaction.guild.default_role, connect=False)
        await interaction.response.send_message("Locked your call", ephemeral=True)

    @discord.ui.button(emoji="<:unlock:1436411273679212725>", style=discord.ButtonStyle.secondary, custom_id="vc_unlock")
    async def unlock(self, interaction: discord.Interaction, button: Button):
        vc = interaction.guild.get_channel(user_voice_channels.get(interaction.user.id))
        await vc.set_permissions(interaction.guild.default_role, connect=True)
        await interaction.response.send_message("Unlocked your call", ephemeral=True)

    @discord.ui.button(emoji="<:hide:1436411272102019142>", style=discord.ButtonStyle.secondary, custom_id="vc_hide")
    async def hide(self, interaction: discord.Interaction, button: Button):
        vc = interaction.guild.get_channel(user_voice_channels.get(interaction.user.id))
        await vc.set_permissions(interaction.guild.default_role, view_channel=False)
        await interaction.response.send_message("Hidden your call", ephemeral=True)

    @discord.ui.button(emoji="<:reveal:1436411270692733069>", style=discord.ButtonStyle.secondary, custom_id="vc_reveal")
    async def reveal(self, interaction: discord.Interaction, button: Button):
        vc = interaction.guild.get_channel(user_voice_channels.get(interaction.user.id))
        await vc.set_permissions(interaction.guild.default_role, view_channel=True)
        await interaction.response.send_message("Revealed your call", ephemeral=True)

    @discord.ui.button(emoji="<:setuserlimit:1436411264644677652>", style=discord.ButtonStyle.secondary, custom_id="vc_limit")
    async def limit(self, interaction: discord.Interaction, button: Button):
        vc = interaction.guild.get_channel(user_voice_channels.get(interaction.user.id))
        await interaction.response.send_modal(UserLimitModal(vc))

    @discord.ui.button(emoji="<:rename:1436875018880553052>", style=discord.ButtonStyle.secondary, custom_id="vc_rename")
    async def rename(self, interaction: discord.Interaction, button: Button):
        vc = interaction.guild.get_channel(user_voice_channels.get(interaction.user.id))
        await interaction.response.send_modal(RenameModal(vc))

    @discord.ui.button(emoji="<:reset:1436411266410483905>", style=discord.ButtonStyle.secondary, custom_id="vc_reset")
    async def reset(self, interaction: discord.Interaction, button: Button):
        vc = interaction.guild.get_channel(user_voice_channels.get(interaction.user.id))
        await vc.edit(name=f"call-{interaction.user.name}", user_limit=0)
        await vc.set_permissions(interaction.guild.default_role, connect=True, view_channel=True)
        await interaction.response.send_message("Reset your call", ephemeral=True)

@bot.group(invoke_without_command=True)
async def vc(ctx):
    """Shows all VC commands with buttons if called without a subcommand."""
    embed = discord.Embed(
        title="VoiceMaster Commands",
        description=(
            "**Use the buttons below or commands:**\n\n"
            "**Lock & Unlock**\n"
            "`.vc lock` - Locks your VC.\n"
            "`.vc unlock` - Unlocks your VC.\n\n"
            "**Hide & Reveal**\n"
            "`.vc hide` - Hide your VC from everyone.\n"
            "`.vc reveal` - Shows your VC to everyone again.\n\n"
            "**Limit**\n"
            "`.vc limit <0-99>` - Set a member limit.\n\n"
            "**Rename**\n"
            "`.vc rename <name>` - Rename your VC.\n\n"
            "**Reset**\n"
            "`.vc reset` - Restores default VC settings.\n\n"
            "**Disconnet**\n"
            "`.vc disconnect <@member>` - Kick a user from your VC.\n\n"
            "**Permit & Revoke**\n"
            "`.vc permit <@member>` - Allow a user in your VC.\n"
            "`.vc revoke <@member>` - Deny a user from joining your VC."
        ),
        color=discord.Color.dark_embed()
    )
    embed.set_footer(text=f"Use `.vc <command>` to control your VC • VoiceMaster Active")

    await ctx.send(embed=embed, view=VoiceControlPanel())

@vc.command()
async def lock(ctx):
    vc_id = user_voice_channels.get(ctx.author.id)
    if not vc_id:
        return await ctx.send(embed=error_response(ctx, "You don't own a VC"))
    vc_channel = ctx.guild.get_channel(vc_id)
    await vc_channel.set_permissions(ctx.guild.default_role, connect=False)
    await ctx.send(embed=embed_response(ctx, "Locked your voice call."))

@vc.command()
async def unlock(ctx):
    vc_id = user_voice_channels.get(ctx.author.id)
    if not vc_id:
        return await ctx.send(embed=error_response(ctx, "You don't own a VC"))
    vc_channel = ctx.guild.get_channel(vc_id)
    await vc_channel.set_permissions(ctx.guild.default_role, connect=True)
    await ctx.send(embed=embed_response(ctx, "Unlocked your voice call."))

@vc.command()
async def hide(ctx):
    vc_id = user_voice_channels.get(ctx.author.id)
    if not vc_id:
        return await ctx.send(embed=error_response(ctx, "You don't own a VC"))
    vc_channel = ctx.guild.get_channel(vc_id)
    await vc_channel.set_permissions(ctx.guild.default_role, view_channel=False)
    await ctx.send(embed=embed_response(ctx, "Your VC is now hidden."))

@vc.command()
async def reveal(ctx):
    vc_id = user_voice_channels.get(ctx.author.id)
    if not vc_id:
        return await ctx.send(embed=error_response(ctx, "You don't own a VC"))
    vc_channel = ctx.guild.get_channel(vc_id)
    await vc_channel.set_permissions(ctx.guild.default_role, view_channel=True)
    await ctx.send(embed=embed_response(ctx, "Your VC is now visible."))

@vc.command()
async def limit(ctx, limit: int):
    vc_id = user_voice_channels.get(ctx.author.id)
    if not vc_id:
        return await ctx.send(embed=error_response(ctx, "You don't own a VC"))
    if limit < 0 or limit > 99:
        return await ctx.send(embed=error_response(ctx, "Limit must be between 0 and 99."))
    vc_channel = ctx.guild.get_channel(vc_id)
    await vc_channel.edit(user_limit=limit)
    await ctx.send(embed=embed_response(ctx, f"Set user limit to **{limit}**."))

@vc.command()
async def disconnect(ctx, user: discord.Member):
    vc_id = user_voice_channels.get(ctx.author.id)
    if not vc_id:
        return await ctx.send(embed=error_response(ctx, "You don't own a VC"))
    vc_channel = ctx.guild.get_channel(vc_id)
    if user in vc_channel.members:
        await user.move_to(None)
        await ctx.send(embed=embed_response(ctx, f"Disconnected **{user.display_name}** from your VC."))
    else:
        await ctx.send(embed=error_response(ctx, "That user is not in your VC."))

@vc.command()
async def rename(ctx, *, name: str):
    vc_id = user_voice_channels.get(ctx.author.id)
    if not vc_id:
        return await ctx.send(embed=error_response(ctx, "You don't own a VC!"))
    vc_channel = ctx.guild.get_channel(vc_id)
    await vc_channel.edit(name=name)
    await ctx.send(embed=embed_response(ctx, f"Renamed your VC to **{name}**."))

@vc.command()
async def reset(ctx):
    vc_id = user_voice_channels.get(ctx.author.id)
    if not vc_id:
        return await ctx.send(embed=error_response(ctx, "You don't own a VC!"))
    vc_channel = ctx.guild.get_channel(vc_id)
    await vc_channel.edit(name=f"call-{ctx.author.name}", user_limit=0)
    await vc_channel.set_permissions(ctx.guild.default_role, connect=True, view_channel=True)
    await ctx.send(embed=embed_response(ctx, "Restored default settings for your VC."))

@vc.command()
async def permit(ctx, user: discord.Member):
    vc_id = user_voice_channels.get(ctx.author.id)
    if not vc_id:
        return await ctx.send(embed=error_response(ctx, "You don't own a VC!"))
    vc_channel = ctx.guild.get_channel(vc_id)
    await vc_channel.set_permissions(user, connect=True)
    await ctx.send(embed=embed_response(ctx, f"Permitted **{user.display_name}** to join your VC."))

@vc.command()
async def revoke(ctx, user: discord.Member):
    vc_id = user_voice_channels.get(ctx.author.id)
    if not vc_id:
        return await ctx.send(embed=error_response(ctx, "You don't own a VC!"))
    vc_channel = ctx.guild.get_channel(vc_id)
    await vc_channel.set_permissions(user, connect=False)
    await ctx.send(embed=embed_response(ctx, f"Revoked **{user.display_name}** from joining your VC."))

@bot.event
async def on_voice_state_update(member, before, after):
    guild = member.guild
    category = discord.utils.get(guild.categories, name=CALL_CATEGORY_NAME)
    if not category:
        return

    if after.channel and after.channel.name == J2C_CHANNEL_NAME:
        vc = await guild.create_voice_channel(f"call-{member.name}", category=category)
        await vc.set_permissions(member, connect=True, manage_channels=True, move_members=True)
        await member.move_to(vc)
        user_voice_channels[member.id] = vc.id

        txt = await guild.create_text_channel(f"call-{member.name}", category=category)
        await txt.set_permissions(member, read_messages=True, send_messages=True)
        await txt.set_permissions(guild.default_role, read_messages=False)

        embed = discord.Embed(
            title="VC Control Panel",
            description=(
            "**Use the buttons below or commands:**\n\n"
            "**Lock & Unlock**\n"
            "`.vc lock` - Locks your VC.\n"
            "`.vc unlock` - Unlocks your VC.\n\n"
            "**Hide & Reveal**\n"
            "`.vc hide` - Hide your VC from everyone.\n"
            "`.vc reveal` - Shows your VC to everyone again.\n\n"
            "**Limit**\n"
            "`.vc limit <0-99>` - Set a member limit.\n\n"
            "**Rename**\n"
            "`.vc rename <name>` - Rename your VC.\n\n"
            "**Reset**\n"
            "`.vc reset` - Restores default VC settings.\n\n"
            "**Disconnet**\n"
            "`.vc disconnect <@member>` - Kick a user from your VC.\n\n"
            "**Permit & Revoke**\n"
            "`.vc permit <@member>` - Allow a user in your VC.\n"
            "`.vc revoke <@member>` - Deny a user from joining your VC.\n\n"
            "**List all vc commands***\n"
            "`.vc` - List all vc commands."
            ),
            color=discord.Color.dark_embed()
        )
        embed.set_author(name=member.name, icon_url=member.display_avatar.url)
        embed.set_footer(text=f"{member.name}’s Private Call • VoiceMaster Active")
        await txt.send(embed=embed, view=VoiceControlPanel())

    if before.channel and before.channel.id in user_voice_channels.values():
        if len(before.channel.members) == 0:
            for uid, cid in list(user_voice_channels.items()):
                if cid == before.channel.id:
                    txt = discord.utils.get(before.channel.category.text_channels, name=f"call-{member.name}")
                    if txt:
                        await txt.delete()
                    await before.channel.delete()
                    user_voice_channels.pop(uid)
                    break    
                    
@bot.command()
async def on(ctx):
    done_emoji = "<:done:1414337207455580231>"
    invalid_emoji = "<:invalid:1414337316524134592>"

    def embed_response(message: str):
        return discord.Embed(
            description=f"{done_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0x56f287)
        )


    def error_response(message: str):
        return discord.Embed(
            description=f"{invalid_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0xfee75d)
        )

    if ctx.author.id not in WHITELIST:
        return await ctx.send(embed=error_response("You can't use this command"))

    global saved_perms
    if not saved_perms:
        try:
            with open(PERMS_FILE, "r") as f:
                saved_perms = json.load(f)
                saved_perms = {int(k): v for k, v in saved_perms.items()}
        except FileNotFoundError:
            return await ctx.send(embed=error_response("Nothing to restore"))

    bot_top_role = ctx.guild.me.top_role

    for role in ctx.guild.roles:
        if role.id not in saved_perms:
            continue

        perms_dict = saved_perms[role.id]
        perms = role.permissions
        updated = perms
        for perm, val in perms_dict.items():
            setattr(updated, perm, val)

        try:
            await role.edit(permissions=updated, reason="Restored dangerous permissions")
        except discord.Forbidden:
            if role.position > bot_top_role.position:
                continue
        except discord.HTTPException:
            continue

    saved_perms.clear()
    with open(PERMS_FILE, "w") as f:
        json.dump(saved_perms, f, indent=4)

    await ctx.send(embed=embed_response("**Enabled** Perms"))


@bot.command()
async def off(ctx):
    done_emoji = "<:done:1414337207455580231>"
    invalid_emoji = "<:invalid:1414337316524134592>"

    def embed_response(message: str):
        return discord.Embed(
            description=f"{done_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0x56f287)
        )


    def error_response(message: str):
        return discord.Embed(
            description=f"{invalid_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0xfee75d)
        )

    if ctx.author.id not in WHITELIST:
        return await ctx.send(embed=error_response("You can't use this command"))

    global saved_perms
    saved_perms.clear()
    bot_top_role = ctx.guild.me.top_role

    for role in ctx.guild.roles:
        if role.is_default():
            continue

        original = {perm: getattr(role.permissions, perm) for perm in DANGEROUS_PERMS}
        saved_perms[role.id] = original

        perms = role.permissions
        updated = perms
        for perm in DANGEROUS_PERMS:
            setattr(updated, perm, False)

        try:
            await role.edit(permissions=updated, reason="Disabled dangerous permissions")
        except discord.Forbidden:
            if role.position > bot_top_role.position:
                continue
        except discord.HTTPException:
            continue

    with open(PERMS_FILE, "w") as f:
        json.dump(saved_perms, f, indent=4)

    await ctx.send(embed=embed_response("**Disabled** Perms"))

@bot.command(name="welcome")
async def set_welcome(ctx, action: str = None, channel: discord.TextChannel = None):
    if not is_whitelisted(ctx.author):
        return

    done_emoji = "<:done:1414337207455580231>"
    invalid_emoji = "<:invalid:1414337316524134592>"

    def embed_response(message: str):
        return discord.Embed(
            description=f"{done_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0x56f287)
        )


    def error_response(message: str):
        return discord.Embed(
            description=f"{invalid_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0xfee75d)
        )
    
    if not is_whitelisted(ctx.author):
        return await ctx.send(embed=error_response("You can’t use this command"))

    if action != "set" or not channel:
        embed = discord.Embed(
            title=f"Command: {ctx.invoked_with}",
            description="Sets the welcome message channel",
            color=discord.Color.dark_theme()
        )
        embed.add_field(name="Category", value="Set Channel", inline=True)
        embed.add_field(name="Usage", value=f"```{ctx.prefix}{ctx.invoked_with} set #channel```", inline=False)
        embed.set_author(name=ctx.author.display_name, icon_url=ctx.author.display_avatar.url)
        return await ctx.send(embed=embed)

    guild_id = str(ctx.guild.id)
    if guild_id not in channel_config:
        channel_config[guild_id] = {}
    channel_config[guild_id]["welcome"] = channel.id

    with open(CONFIG_FILE, "w") as f:
        json.dump(channel_config, f, indent=4)

    await ctx.send(embed=embed_response(f"Welcome channel set to {channel.mention}"))


@bot.command(name="goodbye")
async def set_goodbye(ctx, action: str = None, channel: discord.TextChannel = None):
    if not is_whitelisted(ctx.author):
        return

    done_emoji = "<:done:1414337207455580231>"
    invalid_emoji = "<:invalid:1414337316524134592>"

    def embed_response(message: str):
        return discord.Embed(
            description=f"{done_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0x56f287)
        )


    def error_response(message: str):
        return discord.Embed(
            description=f"{invalid_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0xfee75d)
        )

    if not is_whitelisted(ctx.author):
        return await ctx.send(embed=error_response("You can’t use this command"))

    if action != "set" or not channel:
        embed = discord.Embed(
            title=f"Command: {ctx.invoked_with}",
            description="Sets the goodbye message channel",
            color=discord.Color.dark_theme()
        )
        embed.add_field(name="Category", value="Channel Set", inline=True)
        embed.add_field(name="Usage", value=f"```{ctx.prefix}{ctx.invoked_with} set #channel```", inline=False)
        embed.set_author(name=ctx.author.display_name, icon_url=ctx.author.display_avatar.url)
        return await ctx.send(embed=embed)

    guild_id = str(ctx.guild.id)
    if guild_id not in channel_config:
        channel_config[guild_id] = {}
    channel_config[guild_id]["goodbye"] = channel.id

    with open(CONFIG_FILE, "w") as f:
        json.dump(channel_config, f, indent=4)

    await ctx.send(embed=embed_response(f"Goodbye channel set to {channel.mention}"))

@bot.command(name="test_welcome")
async def test_welcome(ctx):
    done_emoji = "<:done:1414337207455580231>"
    invalid_emoji = "<:invalid:1414337316524134592>"

    def embed_response(message: str):
        return discord.Embed(
            description=f"{done_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0x56f287)
        )

    def error_response(message: str):
        return discord.Embed(
            description=f"{invalid_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0xfee75d)
        )

    if not is_whitelisted(ctx.author):
        return await ctx.send(embed=error_response("You can’t use this command"))

    guild_id = str(ctx.guild.id)
    if guild_id not in channel_config or "welcome" not in channel_config[guild_id]:
        return await ctx.send(embed=error_response("No welcome channel set!"))

    channel = ctx.guild.get_channel(channel_config[guild_id]["welcome"])
    if not channel:
        return await ctx.send(embed=error_response("The welcome channel no longer exists."))

    await on_member_join(ctx.author)

    await ctx.send(embed=embed_response(f"Sent your actual welcome message to {channel.mention}"))


@bot.command(name="test_goodbye")
async def test_goodbye(ctx):
    done_emoji = "<:done:1414337207455580231>"
    invalid_emoji = "<:invalid:1414337316524134592>"

    def embed_response(message: str):
        return discord.Embed(
            description=f"{done_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0x56f287)
        )

    def error_response(message: str):
        return discord.Embed(
            description=f"{invalid_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0xfee75d)
        )

    if not is_whitelisted(ctx.author):
        return await ctx.send(embed=error_response("You can’t use this command"))

    guild_id = str(ctx.guild.id)
    if guild_id not in channel_config or "goodbye" not in channel_config[guild_id]:
        return await ctx.send(embed=error_response("No goodbye channel set!"))

    channel = ctx.guild.get_channel(channel_config[guild_id]["goodbye"])
    if not channel:
        return await ctx.send(embed=error_response("The goodbye channel no longer exists."))

    await on_member_remove(ctx.author)

    await ctx.send(embed=embed_response(f"Sent your actual goodbye message to {channel.mention}"))

@bot.command(name="ban")
@commands.has_permissions(ban_members=True)
async def ban(ctx, target: str = None, *, reason: str = "No reason provided"):
    done_emoji = "<:done:1414337207455580231>"
    invalid_emoji = "<:invalid:1414337316524134592>"

    def embed_response(message: str):
        return discord.Embed(
            description=f"{done_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0x56f287)
        )

    def error_response(message: str):
        return discord.Embed(
            description=f"{invalid_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0xfee75d)
        )

    if ctx.author.id not in WHITELIST:
        return await ctx.send(embed=error_response("You are not whitelisted to use this command."))

    if not target:
        embed = discord.Embed(
            title="Command: ban",
            description="Hard ban a user by mention or ID.",
            color=discord.Color.dark_theme()
        )
        embed.add_field(name="Category", value="Moderation", inline=True)
        embed.add_field(name="Usage", value=f"```{ctx.prefix}{ctx.invoked_with} @user [reason]```", inline=False)
        embed.set_author(name=ctx.author.display_name, icon_url=ctx.author.display_avatar.url)
        return await ctx.send(embed=embed)

    user = None
    if target.startswith("<@") and target.endswith(">"):
        target_id = int(target.strip("<@!>"))
        user = await bot.fetch_user(target_id)
    else:
        try:
            target_id = int(target)
            user = await bot.fetch_user(target_id)
        except ValueError:
            return await ctx.send(embed=error_response("Invalid user mention or ID."))

    if not user:
        return await ctx.send(embed=error_response("User not found."))

    try:
        await ctx.guild.ban(user, reason=f"[Hard Ban] {reason} — Banned by {ctx.author}")
        await ctx.send(embed=embed_response(f"Hard banned **{user}** (`{user.id}`)\n**Reason:** {reason}"))
    except discord.Forbidden:
        await ctx.send(embed=error_response("I don’t have permission to ban that user."))
    except Exception as e:
        await ctx.send(embed=error_response(f"Failed to ban user: `{e}`"))


@bot.command(name="unban")
@commands.has_permissions(ban_members=True)
async def unban(ctx, target: str = None):
    done_emoji = "<:done:1414337207455580231>"
    invalid_emoji = "<:invalid:1414337316524134592>"

    def embed_response(message: str):
        return discord.Embed(
            description=f"{done_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0x56f287)
        )

    def error_response(message: str):
        return discord.Embed(
            description=f"{invalid_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0xfee75d)
        )

    if ctx.author.id not in WHITELIST:
        return await ctx.send(embed=error_response("You are not whitelisted to use this command."))

    if not target:
        embed = discord.Embed(
            title="Command: unban",
            description="Unban a user by mention or ID.",
            color=discord.Color.dark_theme()
        )
        embed.add_field(name="Category", value="Moderation", inline=True)
        embed.add_field(name="Usage", value=f"```{ctx.prefix}{ctx.invoked_with} @user```", inline=False)
        embed.set_author(name=ctx.author.display_name, icon_url=ctx.author.display_avatar.url)
        return await ctx.send(embed=embed)

    if target.startswith("<@") and target.endswith(">"):
        target_id = int(target.strip("<@!>"))
    else:
        try:
            target_id = int(target)
        except ValueError:
            return await ctx.send(embed=error_response("Invalid user mention or ID."))

    try:
        user = await bot.fetch_user(target_id)
        await ctx.guild.unban(user, reason=f"[Hard Unban] Unbanned by {ctx.author}")
        await ctx.send(embed=embed_response(f"Unbanned **{user}** (`{user.id}`)"))
    except discord.NotFound:
        await ctx.send(embed=error_response("That user isn’t banned."))
    except discord.Forbidden:
        await ctx.send(embed=error_response("I don’t have permission to unban that user."))
    except Exception as e:
        await ctx.send(embed=error_response(f"Failed to unban user: `{e}`"))

    

@bot.command(aliases=["purge", "c"])
async def clear(ctx, amount: int = None):
    """Clears messages (ignoring pinned ones)."""

    done_emoji = "<:done:1414337207455580231>"
    invalid_emoji = "<:invalid:1414337316524134592>"

    def embed_response(message: str):
        return discord.Embed(
            description=f"{done_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0x56f287),
        )

    def error_response(message: str):
        return discord.Embed(
            description=f"{invalid_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0xfee75d),
        )

    if WHITELIST and ctx.author.id not in WHITELIST:
        return await ctx.send(embed=error_response("You can't use this command."))

    if not ctx.author.guild_permissions.manage_messages:
        return await ctx.send(embed=error_response("You don't have permission to clear messages."))


    if amount is None or amount <= 0:
        return await ctx.send(embed=error_response("Please provide a valid number of messages to clear."))

    try:
        deleted_count = 0
        async for message in ctx.channel.history(limit=amount + 1):
            if not message.pinned:
                try:
                    await message.delete()
                    deleted_count += 1
                except discord.Forbidden:
                    continue

        confirmation = await ctx.send(
            embed=embed_response(f"Cleared **{deleted_count - 1}** messages successfully!")
        )
        await asyncio.sleep(5)
        await confirmation.delete()

    except discord.Forbidden:
        await ctx.send(embed=error_response("I don't have permission to delete messages."))
    except discord.HTTPException as e:
        await ctx.send(embed=error_response(f"An error occurred: {e}"))

done_emoji = "<:done:1414337207455580231>"
invalid_emoji = "<:invalid:1414337316524134592>"

class AutoroleDropdown(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)

    def embed_response(self, interaction: discord.Interaction, message: str):
        return discord.Embed(
            description=f"{done_emoji} {interaction.user.mention} {message}",
            color=discord.Color(0x56f287)
        )

    def error_response(self, interaction: discord.Interaction, message: str):
        return discord.Embed(
            description=f"{invalid_emoji} {interaction.user.mention} {message}",
            color=discord.Color(0xfee75d)
        )

    @discord.ui.select(
        placeholder="Choose a subcommand",
        options=[
            discord.SelectOption(label="Add", description="Add a role to autoroles"),
            discord.SelectOption(label="Remove", description="Remove a role from autoroles"),
            discord.SelectOption(label="List", description="List all autoroles"),
        ],
    )
    async def select_callback(self, interaction: discord.Interaction, select: discord.ui.Select):
        choice = select.values[0]
        guild_id = interaction.guild.id

        if choice == "Add":
            options = [
                discord.SelectOption(label=r.name, value=str(r.id))
                for r in interaction.guild.roles if not r.is_default()
            ]
            if not options:
                return await interaction.response.send_message(
                    embed=self.error_response(interaction, "No roles available to add"), ephemeral=True
                )

            select_menu = discord.ui.Select(
                placeholder="Select a role to add to autoroles",
                options=options,
                min_values=1,
                max_values=1
            )

            async def add_role_callback(role_interaction: discord.Interaction):
                role_id = int(select_menu.values[0])
                autoroles.setdefault(guild_id, [])
                if role_id in autoroles[guild_id]:
                    return await role_interaction.response.send_message(
                        embed=self.error_response(role_interaction, "Role already in autoroles"), ephemeral=True
                    )
                autoroles[guild_id].append(role_id)
                save_autoroles()
                await role_interaction.response.send_message(
                    embed=self.embed_response(role_interaction, f"Added role <@&{role_id}> to autoroles"), ephemeral=True
                )

            select_menu.callback = add_role_callback
            view = discord.ui.View()
            view.add_item(select_menu)
            await interaction.response.send_message("Select a role to add:", view=view, ephemeral=True)

        elif choice == "Remove":
            guild_roles = autoroles.get(guild_id, [])
            if not guild_roles:
                return await interaction.response.send_message(
                    embed=self.error_response(interaction, "No autoroles set"), ephemeral=True
                )

            options = [
                discord.SelectOption(label=interaction.guild.get_role(r).name, value=str(r))
                for r in guild_roles
            ]
            remove_menu = discord.ui.Select(
                placeholder="Select a role to remove",
                options=options,
                min_values=1,
                max_values=1
            )

            async def remove_role_callback(role_interaction: discord.Interaction):
                role_id = int(remove_menu.values[0])
                autoroles[guild_id].remove(role_id)
                save_autoroles()
                await role_interaction.response.send_message(
                    embed=self.embed_response(role_interaction, f"Removed role <@&{role_id}> from autoroles"), ephemeral=True
                )

            remove_menu.callback = remove_role_callback
            view = discord.ui.View()
            view.add_item(remove_menu)
            await interaction.response.send_message("Select a role to remove:", view=view, ephemeral=True)

        elif choice == "List":
            guild_roles = autoroles.get(guild_id, [])
            if not guild_roles:
                return await interaction.response.send_message(
                    embed=self.error_response(interaction, "No autoroles set"), ephemeral=True
                )

            roles_text = "\n".join([f"<@&{r}>" for r in guild_roles])
            await interaction.response.send_message(
                embed=discord.Embed(
                    title="Autoroles List",
                    description=roles_text,
                    color=discord.Color(0x56f287)
                ),
                ephemeral=True
            )

@bot.command(name="autorole", aliases=["arole"])
async def autorole(ctx, subcommand: str = None, *, role_name: str = None):
    guild_roles = {r.name.lower(): r for r in ctx.guild.roles}
    guild_id = ctx.guild.id

    if not subcommand:

        embed = discord.Embed(
            title="Command: autorole",
            description="Add roles automatically when someone joins the server",
            color=discord.Color.dark_theme(),
        )
        embed.add_field(name="Aliases", value="`arole`", inline=False)
        embed.add_field(name="Module", value="Moderation", inline=True)
        embed.add_field(name="Permissions", value="Manage Server", inline=True)
        embed.add_field(
            name="Syntax",
            value="```\n.autorole [subcommand] [role]\n```",
            inline=False,
        )
        embed.add_field(name="Example", value="```\n.autorole add @Member\n```", inline=False)
        embed.set_author(name=ctx.author.display_name, icon_url=ctx.author.display_avatar.url)
        embed.set_footer(text="Select a subcommand from the dropdown below to view details.")
        return await ctx.send(embed=embed, view=AutoroleDropdown())

    subcommand = subcommand.lower()

    if subcommand == "add":
        if not role_name:
            return await ctx.send(embed=discord.Embed(
                description=f"{invalid_emoji} {ctx.author.mention} Please provide a role to add",
                color=discord.Color(0xfee75d)
            ))

        role = None
        if role_name.startswith("<@&") and role_name.endswith(">"):
            role_id = int(role_name[3:-1])
            role = ctx.guild.get_role(role_id)
        elif role_name.isdigit():
            role = ctx.guild.get_role(int(role_name))
        else:
            role = guild_roles.get(role_name.lower())

        if not role:
            return await ctx.send(embed=discord.Embed(
                description=f"{invalid_emoji} {ctx.author.mention} Role not found",
                color=discord.Color(0xfee75d)
            ))

        autoroles.setdefault(guild_id, [])
        if role.id in autoroles[guild_id]:
            return await ctx.send(embed=discord.Embed(
                description=f"{invalid_emoji} {ctx.author.mention} Role already in autoroles",
                color=discord.Color(0xfee75d)
            ))

        autoroles[guild_id].append(role.id)
        save_autoroles()
        await ctx.send(embed=discord.Embed(
            description=f"{done_emoji} {ctx.author.mention} Added role {role.mention} to autoroles",
            color=discord.Color(0x56f287)
        ))

    elif subcommand == "remove":
        if not role_name:
            return await ctx.send(embed=discord.Embed(
                description=f"{invalid_emoji} {ctx.author.mention} Please provide a role to remove",
                color=discord.Color(0xfee75d)
            ))

        role = None
        if role_name.startswith("<@&") and role_name.endswith(">"):
            role_id = int(role_name[3:-1])
            role = ctx.guild.get_role(role_id)
        elif role_name.isdigit():
            role = ctx.guild.get_role(int(role_name))
        else:
            role = guild_roles.get(role_name.lower())

        if not role:
            return await ctx.send(embed=discord.Embed(
                description=f"{invalid_emoji} {ctx.author.mention} Role not found",
                color=discord.Color(0xfee75d)
            ))

        guild_autoroles = autoroles.get(guild_id, [])
        if role.id not in guild_autoroles:
            return await ctx.send(embed=discord.Embed(
                description=f"{invalid_emoji} {ctx.author.mention} Role not in autoroles",
                color=discord.Color(0xfee75d)
            ))

        autoroles[guild_id].remove(role.id)
        save_autoroles()
        await ctx.send(embed=discord.Embed(
            description=f"{done_emoji} {ctx.author.mention} Removed role {role.mention} from autoroles",
            color=discord.Color(0x56f287)
        ))

    elif subcommand == "list":
        guild_autoroles = autoroles.get(guild_id, [])
        if not guild_autoroles:
            return await ctx.send(embed=discord.Embed(
                description=f"{invalid_emoji} {ctx.author.mention} No autoroles set",
        		color=discord.Color.dark_theme()
            ))

        roles_text = "\n".join([f"<@&{r}>" for r in guild_autoroles])
        await ctx.send(embed=discord.Embed(
            title="Autoroles List",
            description=roles_text,
        	color=discord.Color.dark_theme()
        ))

    else:
        await ctx.send(embed=discord.Embed(
            description=f"{invalid_emoji} {ctx.author.mention} Invalid subcommand. Use add, remove, or list.",
            color=discord.Color(0xfee75d)
        ))
        
@bot.command(aliases=["r", "rank"])
async def role(ctx, member: discord.Member = None, *, role_names: str = None):
    if not is_whitelisted(ctx.author):
        return

    done_emoji = "<:done:1414337207455580231>"
    invalid_emoji = "<:invalid:1414337316524134592>"

    def embed_response(message: str):
        return discord.Embed(
            description=f"{done_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0x56f287)
        )


    def error_response(message: str):
        return discord.Embed(
            description=f"{invalid_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0xfee75d)
        )


    if not member or not role_names:
        embed = discord.Embed(
            title=f"Command: {ctx.invoked_with}",
            description="Add or remove roles from a member by name.",
            color=discord.Color.dark_theme()
        )
        embed.add_field(name="Category", value="Moderation", inline=True)
        embed.add_field(
            name="Usage",
            value=f"```{ctx.prefix}{ctx.invoked_with} @member role1, role2```",
            inline=False
        )
        embed.set_author(name=ctx.author.display_name, icon_url=ctx.author.display_avatar.url)
        return await ctx.send(embed=embed)


    role_names_list = [name.strip() for name in role_names.split(",")]
    role_names_all = [r.name for r in ctx.guild.roles]
    changes = []

    for role_name in role_names_list:
        closest = difflib.get_close_matches(role_name, role_names_all, n=1, cutoff=0.4)
        if not closest:
            continue

        role = discord.utils.get(ctx.guild.roles, name=closest[0])

        if ctx.guild.me.top_role <= role:
            continue

        try:
            if role in member.roles:
                await member.remove_roles(role)
                changes.append(f"-{role.name}")
            else:
                await member.add_roles(role)
                changes.append(f"+{role.name}")
        except Exception:
            continue

    if changes:
        await ctx.send(embed=embed_response(f"{member.display_name} {', '.join(changes)}"))
    else:
        await ctx.send(embed=error_response("Failed to add or remove roles"))


recently_nuked = set()

@bot.command()
async def nuke(ctx):
    done_emoji = "<:done:1414337207455580231>"
    invalid_emoji = "<:invalid:1414337316524134592>"

    def embed_response(message: str):
        return discord.Embed(
            description=f"{done_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0x56f287)
        )


    def error_response(message: str):
        return discord.Embed(
            description=f"{invalid_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0xfee75d)
        )

    if ctx.author.id not in WHITELIST:
        return await ctx.send(embed=error_response("You can't use this command"))

    if ctx.channel.id in recently_nuked:
        return await ctx.send(embed=embed_response("This channel was just nuked"))

    recently_nuked.add(ctx.channel.id)

    try:
        new_channel = await ctx.channel.clone(reason=f"Nuked by {ctx.author}")
        await new_channel.edit(position=ctx.channel.position, sync_permissions=True)
        await ctx.channel.delete(reason="Nuked")

        await new_channel.send(embed=embed_response("Channel nuked successfully!"))

        await asyncio.sleep(10)
        recently_nuked.discard(new_channel.id)

    except discord.Forbidden:
        await ctx.send(embed=error_response("I do not have permission to nuke this channel."))
    except discord.HTTPException as e:
        await ctx.send(embed=error_response(f"An error occurred: {e}"))

        
class RolesView(View):
    def __init__(self, ctx, roles, left_emoji, right_emoji, per_page=10):
        super().__init__(timeout=60)
        self.ctx = ctx
        self.roles = roles
        self.per_page = per_page
        self.pages = [roles[i:i + per_page] for i in range(0, len(roles), per_page)]
        self.total_pages = len(self.pages)
        self.current_page = 0
        self.message = None
        self.left_emoji = left_emoji
        self.right_emoji = right_emoji

        # Buttons
        self.previous_button = Button(style=discord.ButtonStyle.secondary, emoji=left_emoji)
        self.next_button = Button(style=discord.ButtonStyle.secondary, emoji=right_emoji)
        self.previous_button.callback = self.previous
        self.next_button.callback = self.next
        self.add_item(self.previous_button)
        self.add_item(self.next_button)

    def format_embed(self, index):
        embed = discord.Embed(
            title=f"Roles in {self.ctx.guild.name}",
            color=discord.Color.light_grey()
        )
        embed.set_author(
            name=self.ctx.author.display_name,
            icon_url=self.ctx.author.display_avatar.url
        )
        chunk = self.pages[index]
        role_list = "\n".join(
            f"{i + 1 + index * self.per_page}. {role.mention}" for i, role in enumerate(chunk)
        )
        embed.description = role_list
        embed.set_footer(text=f"{len(self.roles)} roles • Page {index + 1}/{self.total_pages}")
        return embed

    async def update_view(self):
        if self.message:
            await self.message.edit(embed=self.format_embed(self.current_page), view=self)

    async def previous(self, interaction: discord.Interaction):
        if interaction.user != self.ctx.author:
            return await interaction.response.send_message(
                "Only the command author can use these buttons.", ephemeral=True
            )
        await interaction.response.defer()
        if self.current_page > 0:
            self.current_page -= 1
            await self.update_view()

    async def next(self, interaction: discord.Interaction):
        if interaction.user != self.ctx.author:
            return await interaction.response.send_message(
                "Only the command author can use these buttons.", ephemeral=True
            )
        await interaction.response.defer()
        if self.current_page < self.total_pages - 1:
            self.current_page += 1
            await self.update_view()

    async def on_timeout(self):
        for item in self.children:
            item.disabled = True
        if self.message:
            await self.message.edit(view=self)


@bot.command()
async def roles(ctx):
    guild = ctx.guild
    roles = sorted(guild.roles[1:], key=lambda r: r.position, reverse=True)

    if not roles:
        return await ctx.send("No roles found.")

    # Your custom emojis
    left_emoji = discord.PartialEmoji.from_str("<:left:1426530741508112424>")
    right_emoji = discord.PartialEmoji.from_str("<:right:1426530762122858546>")

    view = RolesView(ctx, roles, left_emoji, right_emoji)
    view.message = await ctx.send(embed=view.format_embed(0), view=view)
        
@bot.command(name="shop")
async def shop(ctx):
    if not is_whitelisted(ctx.author):
        return

    embed1 = discord.Embed(
        title="**Wealthy Roles**",
        description=(
            "> <@&1385831059010093159> : $250\n"
            "> <@&1384561091522072598> : $135\n"
            "> <@&1384914915386724372> : $65\n\n"

            "### Custom\n"
            "> Custom Role : $20\n"
            "> Custom Role + Admin : $80"
        ),
        color=discord.Color.dark_theme()
    )

    embed2 = discord.Embed(
        title="**Advertisements**",
        description=(
        "> 1 Day : $10\n"
        "> 4 Day : $15\n"
        "> 7 Day : $35\n\n"

        "### With Ping\n"
        "> 1 Day + 1 Ping  : $17\n"
        "> 4 Day + 1 Ping  : $36\n"
        "> 7 Day + 2 Ping  : $60\n\n"

        "### Permanent\n"
        "> $150 : @everyone ping every week\n"
        "> $200 : @everyone ping every week + ping on join"
        ),
        color=discord.Color.dark_theme()
    )

    embed3 = discord.Embed(
        title="**Bans**",
        description=(
        "> Ban : 30\n"
        "> Unban : $25\n"
        "> Hard Ban : $50"
        ),
        color=discord.Color.dark_theme()
    )

    embed4 = discord.Embed(
        title="**How To Purchase**",
        description=(
        "> Message anyone with <@&1385317776440426566> Anyone else cannot sell you services.\n\n"

        "### What We Take\n"
        "> PayPal\n"
        "> Robux"
        ),
        color=discord.Color.dark_theme()
    )

    for embed in [embed1, embed2, embed3, embed4]:
        await ctx.send(embed=embed)
        
@bot.command(name="contributions")
async def contributions(ctx):
    embed = discord.Embed(
        color=discord.Color.dark_theme()
    )

    embed.set_author(
        name="All Games I Have Contributed For",
        icon_url="https://cdn.discordapp.com/attachments/1433584151860346930/1433587910854053918/IMG_2726.jpg?ex=69053c48&is=6903eac8&hm=e90ceae33969297da64cbe1034412d6cf6779509ce3557455a143600983741e9&"
    )

    embed.description = (
        "> **Hood Craft** : __**168.3K**__\n"
        "> **Dea Hood** : __**353.7K**__\n"
        "> **Da Fights** : __**1.5M**__\n"
        "> **Sun Hood** : __**42.1K**__\n"
        "> **Da Kitty** : __**4K**__\n"
        "> **Dem Hood** : __**20.7K**__\n"
        "> **Rey Hood** : __**36.7K**__\n"
        "> **Da Fort** : __**20.8K**__\n"
        "> **Bay Hood** : __**9.4K**__"
    )

    embed.set_footer(text="Last Updated • 30/10/2025")

    await ctx.send(embed=embed)

@tree.command(name="r2v", description="Create a persistent reaction role message.")
@app_commands.describe(
    emoji="The emoji people react with",
    role="The role users will get when reacting"
)
async def r2v(interaction: discord.Interaction, emoji: str, role: discord.Role):
    if not can_manage(interaction):
        return await interaction.response.send_message("You can't use this command.", ephemeral=True)

    msg = await interaction.channel.send("r2v")
    try:
        await msg.add_reaction(emoji)
    except Exception as e:
        return await interaction.response.send_message(f"Failed to react with {emoji}: {e}", ephemeral=True)

    r2v_data[str(msg.id)] = {
        "guild_id": interaction.guild.id,
        "channel_id": interaction.channel.id,
        "role_id": role.id,
        "emoji": emoji
    }
    save_r2v_data()

    await interaction.response.send_message(
        f"Reaction role set up\nEmoji: {emoji}\nRole: {role.mention}",
        ephemeral=True
    )


@bot.event
async def on_raw_reaction_add(payload: discord.RawReactionActionEvent):
    
    if payload.user_id == bot.user.id:
        return

    data = r2v_data.get(str(payload.message_id))
    if not data:
        return


    emoji_str = str(payload.emoji.id) if payload.emoji.id else payload.emoji.name
    target_emoji = data["emoji"].strip("<>:") 

    if emoji_str not in target_emoji and payload.emoji.name not in target_emoji:
        return  

    guild = bot.get_guild(data["guild_id"])
    if not guild:
        return
    member = guild.get_member(payload.user_id)
    if not member:
        return
    role = guild.get_role(data["role_id"])
    if not role:
        return

    try:
        await member.add_roles(role, reason="Reaction role added")
        print(f"Gave {role.name} to {member}")
    except discord.Forbidden:
        print(f"Missing permission to add role {role.name} to {member}")
    except Exception as e:
        print(f"Error adding role: {e}")


@bot.event
async def on_raw_reaction_remove(payload: discord.RawReactionActionEvent):
    if payload.user_id == bot.user.id:
        return

    data = r2v_data.get(str(payload.message_id))
    if not data:
        return

    emoji_str = str(payload.emoji.id) if payload.emoji.id else payload.emoji.name
    target_emoji = data["emoji"].strip("<>:")

    if emoji_str not in target_emoji and payload.emoji.name not in target_emoji:
        return

    guild = bot.get_guild(data["guild_id"])
    if not guild:
        return
    member = guild.get_member(payload.user_id)
    if not member:
        return
    role = guild.get_role(data["role_id"])
    if not role:
        return

    try:
        await member.remove_roles(role, reason="Reaction role removed")
        print(f"Removed {role.name} from {member}")
    except discord.Forbidden:
        print(f"Missing permission to remove role {role.name} from {member}")
    except Exception as e:
        print(f"Error removing role: {e}")


class PingOnJoinView(discord.ui.View):
    def __init__(self, guild_id: int):
        super().__init__(timeout=None)
        self.guild_id = str(guild_id)
        self.message: discord.Message | None = None

    def set_message(self, message: discord.Message):
        self.message = message

    async def update_embed(self):
        channels = ping_on_join_channels.get(self.guild_id, [])
        channel_mentions = [f"<#{cid}>" for cid in channels] if channels else ["*None*"]

        embed = discord.Embed(
            title="Ping on join",
            description="The bot will ping users in these channels once they join, and delete the ping afterwards.",
            color=discord.Color.dark_embed()
        )
        embed.add_field(name="Channels", value="\n".join(channel_mentions), inline=False)

        if self.message:
            try:
                await self.message.edit(embed=embed, view=self)
            except discord.NotFound:
                pass

    @discord.ui.button(label="Add Channel", style=discord.ButtonStyle.green, custom_id="pingonjoin_add")
    async def add_channel(self, interaction: discord.Interaction, button: discord.ui.Button):
        if not can_manage(interaction):
            return

        await interaction.response.send_message("Please mention a channel", ephemeral=True)

        def check(msg):
            return (
                msg.author == interaction.user and
                msg.channel == interaction.channel and
                len(msg.channel_mentions) > 0
            )

        try:
            msg = await bot.wait_for("message", timeout=30.0, check=check)
            mentioned_channel = msg.channel_mentions[0]
            guild_id = str(interaction.guild.id)

            ping_on_join_channels.setdefault(guild_id, [])

            if mentioned_channel.id in ping_on_join_channels[guild_id]:
                await interaction.followup.send(f"{mentioned_channel.mention} is already in the list.", ephemeral=True)
            else:
                ping_on_join_channels[guild_id].append(mentioned_channel.id)
                save_ping_channels()
                await interaction.followup.send(f"Added {mentioned_channel.mention} to ping-on-join.", ephemeral=True)
                await self.update_embed()

            await msg.delete()
        except asyncio.TimeoutError:
            await interaction.followup.send("Timed out waiting for a channel mention.", ephemeral=True)

    @discord.ui.button(label="Remove Channel", style=discord.ButtonStyle.red, custom_id="pingonjoin_remove")
    async def remove_channel(self, interaction: discord.Interaction, button: discord.ui.Button):
        if not can_manage(interaction):
            return

        await interaction.response.send_message("Please mention the channel you want to remove.", ephemeral=True)

        def check(msg):
            return (
                msg.author == interaction.user and
                msg.channel == interaction.channel and
                len(msg.channel_mentions) > 0
            )

        try:
            msg = await bot.wait_for("message", timeout=30.0, check=check)
            mentioned_channel = msg.channel_mentions[0]
            guild_id = str(interaction.guild.id)

            if guild_id in ping_on_join_channels and mentioned_channel.id in ping_on_join_channels[guild_id]:
                ping_on_join_channels[guild_id].remove(mentioned_channel.id)
                save_ping_channels()
                await interaction.followup.send(f"Removed {mentioned_channel.mention} from ping-on-join.", ephemeral=True)
                await self.update_embed()
            else:
                await interaction.followup.send(f"{mentioned_channel.mention} is not in the list.", ephemeral=True)

            await msg.delete()
        except asyncio.TimeoutError:
            await interaction.followup.send("Timed out waiting for a channel mention.", ephemeral=True)


class AddChannelModal(discord.ui.Modal):
    def __init__(self, view: PingOnJoinView):
        super().__init__(title="Add Channel for Ping on Join")
        self.view = view

        self.channel_input = discord.ui.TextInput(
            label="Channel",
            placeholder="#general or channel ID",
            required=True
        )
        self.add_item(self.channel_input)

    async def on_submit(self, interaction: discord.Interaction):
        if not can_manage(interaction):
            return

        value = self.channel_input.value.strip()
        channel = None

        if value.startswith("<#") and value.endswith(">"):
            try:
                channel_id = int(value[2:-1])
                channel = interaction.guild.get_channel(channel_id)
            except ValueError:
                pass
        else:
            try:
                channel_id = int(value)
                channel = interaction.guild.get_channel(channel_id)
            except ValueError:
                pass

        if not channel or not isinstance(channel, discord.TextChannel):
            return await interaction.response.send_message("Invalid channel provided.", ephemeral=True)

        guild_id = str(interaction.guild.id)
        ping_on_join_channels.setdefault(guild_id, [])

        if channel.id not in ping_on_join_channels[guild_id]:
            ping_on_join_channels[guild_id].append(channel.id)
            save_ping_channels()

            await interaction.response.send_message(f"Added {channel.mention} to ping-on-join.", ephemeral=True)
            await self.view.update_embed()
        else:
            await interaction.response.send_message(f"{channel.mention} is already in the list.", ephemeral=True)


@tree.command(name="pingonjoin", description="Manage Ping on Join settings.")
async def pingonjoin(interaction: discord.Interaction):
    if not can_manage(interaction):
        return

    guild_id = str(interaction.guild.id)
    channels = ping_on_join_channels.get(guild_id, [])
    channel_mentions = [f"<#{cid}>" for cid in channels] if channels else ["*None*"]

    embed = discord.Embed(
        title="Ping on join",
        description="The bot will ping users in these channels once they join, and delete the ping afterwards.",
        color=discord.Color.dark_theme()
    )
    embed.add_field(name="Channels", value="\n".join(channel_mentions), inline=False)

    view = PingOnJoinView(interaction.guild.id)
    await interaction.response.send_message(embed=embed, view=view, ephemeral=True)
    sent_message = await interaction.original_response()
    view.set_message(sent_message)
    
    
    
@tree.command(name="joinlogset", description="Set the join log channel.")
@app_commands.describe(channel="The channel where join logs will be sent")
async def joinlogset(interaction: discord.Interaction, channel: discord.TextChannel):
    if not can_manage(interaction):
        return await interaction.response.send_message("You can’t use this command.", ephemeral=True)

    guild_id = str(interaction.guild.id)
    join_log_channels[guild_id] = channel.id
    save_join_logs()
    await interaction.response.send_message(f"Join log channel set to {channel.mention}", ephemeral=True)


@tree.command(name="removejoinlog", description="Remove the join log channel.")
async def removejoinlog(interaction: discord.Interaction):
    if not can_manage(interaction):
        return await interaction.response.send_message("You can’t use this command.", ephemeral=True)

    guild_id = str(interaction.guild.id)
    if guild_id in join_log_channels:
        del join_log_channels[guild_id]
        save_join_logs()
        await interaction.response.send_message("Removed join log channel.", ephemeral=True)
    else:
        await interaction.response.send_message("No join log channel is set.", ephemeral=True)

class JoinLogView(discord.ui.View):
    def __init__(self, target_id: int):
        super().__init__(timeout=None)
        self.target_id = target_id

    async def interaction_check(self, interaction: discord.Interaction) -> bool:
        if not can_manage(interaction):
            await interaction.response.send_message("You are not whitelisted.", ephemeral=True)
            return False
        return True

    @discord.ui.button(label="User ID", style=discord.ButtonStyle.gray, custom_id="join_userid")
    async def userid_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.send_message(f"`{self.target_id}`", ephemeral=True)

    @discord.ui.button(label="Kick", style=discord.ButtonStyle.danger, custom_id="join_kick")
    async def kick_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        member = interaction.guild.get_member(self.target_id)
        if not member:
            return await interaction.response.send_message("User not found.", ephemeral=True)
        try:
            await member.kick(reason=f"Kicked by {interaction.user}")
            await interaction.response.send_message(f"Kicked `{member}`", ephemeral=True)
        except Exception as e:
            await interaction.response.send_message(f"Failed to kick: {e}", ephemeral=True)

    @discord.ui.button(label="Ban", style=discord.ButtonStyle.danger, custom_id="join_ban")
    async def ban_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        user = interaction.guild.get_member(self.target_id)
        if not user:
            user = await bot.fetch_user(self.target_id)
        try:
            await interaction.guild.ban(user, reason=f"Banned by {interaction.user}")
            await interaction.response.send_message(f"Banned `{user}`", ephemeral=True)
        except Exception as e:
            await interaction.response.send_message(f"Failed to ban: {e}", ephemeral=True)


done_emoji = "<:done:1414337207455580231>"
invalid_emoji = "<:invalid:1414337316524134592>"

def embed_response(ctx, message: str):
    return discord.Embed(
        description=f"{done_emoji} {ctx.author.mention} {message}",
        color=discord.Color(0x56f287)
    )

def error_response(ctx, message: str):
    return discord.Embed(
        description=f"{invalid_emoji} {ctx.author.mention} {message}",
        color=discord.Color(0xfee75d)
    )

async def get_member(ctx, user_id_or_mention):
    """Resolve a member from mention, user ID, or discord.Member"""
    try:
        if isinstance(user_id_or_mention, discord.Member):
            return user_id_or_mention
        user_id = int(user_id_or_mention.strip("<@!>"))
        return ctx.guild.get_member(user_id)
    except:
        return None

@bot.command(aliases=["m"])
@commands.has_permissions(moderate_members=True)
async def mute(ctx, user: str = None, duration: int = None, *, reason: str = None):
    if not user or not duration:
        embed = discord.Embed(
            title=f"Command: {ctx.invoked_with}",
            description="Temporarily mute (timeout) a member.",
            color=discord.Color.dark_theme()
        )
        embed.add_field(name="Category", value="Moderation", inline=True)
        embed.add_field(name="Usage", value=f"```{ctx.prefix}{ctx.invoked_with} @member/<ID> <minutes> [reason]```", inline=False)
        embed.set_author(name=ctx.author.display_name, icon_url=ctx.author.display_avatar.url)
        return await ctx.send(embed=embed)

    member = await get_member(ctx, user)
    if not member:
        return await ctx.send(embed=error_response(ctx, "Could not find that member."))

    try:
        await member.timeout(timedelta(minutes=duration), reason=reason)
        await ctx.send(embed=embed_response(ctx, f"{member.mention} has been muted for **{duration} minutes**."))
    except discord.Forbidden:
        await ctx.send(embed=error_response(ctx, "I don't have permission to mute that user."))
    except Exception as e:
        await ctx.send(embed=error_response(ctx, f"An error occurred: {e}"))


@bot.command(aliases=["unm"])
@commands.has_permissions(moderate_members=True)
async def unmute(ctx, user: str = None):
    if not user:
        embed = discord.Embed(
            title=f"Command: {ctx.invoked_with}",
            description="Remove a timeout from a member.",
            color=discord.Color.dark_theme()
        )
        embed.add_field(name="Category", value="Moderation", inline=True)
        embed.add_field(name="Usage", value=f"```{ctx.prefix}{ctx.invoked_with} @member/<ID>```", inline=False)
        embed.set_author(name=ctx.author.display_name, icon_url=ctx.author.display_avatar.url)
        return await ctx.send(embed=embed)

    member = await get_member(ctx, user)
    if not member:
        return await ctx.send(embed=error_response(ctx, "Could not find that member."))

    try:
        await member.timeout(None)
        await ctx.send(embed=embed_response(ctx, f"{member.mention} has been unmuted."))
    except discord.Forbidden:
        await ctx.send(embed=error_response(ctx, "I don't have permission to unmute that user."))
    except Exception as e:
        await ctx.send(embed=error_response(ctx, f"An error occurred: {e}"))


@bot.command()
@commands.has_permissions(kick_members=True)
async def kick(ctx, user: str = None, *, reason: str = None):
    if not user:
        embed = discord.Embed(
            title=f"Command: {ctx.invoked_with}",
            description="Kick a member from the server.",
            color=discord.Color.dark_theme()
        )
        embed.add_field(name="Category", value="Moderation", inline=True)
        embed.add_field(name="Usage", value=f"```{ctx.prefix}{ctx.invoked_with} @member/<ID> [reason]```", inline=False)
        embed.set_author(name=ctx.author.display_name, icon_url=ctx.author.display_avatar.url)
        return await ctx.send(embed=embed)

    member = await get_member(ctx, user)
    if not member:
        return await ctx.send(embed=error_response(ctx, "Could not find that member."))

    try:
        await member.kick(reason=reason)
        await ctx.send(embed=embed_response(ctx, f"{member.mention} has been kicked."))
    except discord.Forbidden:
        await ctx.send(embed=error_response(ctx, "I don't have permission to kick that user."))
    except Exception as e:
        await ctx.send(embed=error_response(ctx, f"An error occurred: {e}"))


@bot.command()
@commands.has_permissions(manage_channels=True)
async def lock(ctx):
    channel = ctx.channel
    await channel.set_permissions(ctx.guild.default_role, send_messages=False)
    await ctx.send(embed=embed_response(ctx, f"{channel.mention} has been locked."))


@bot.command()
@commands.has_permissions(manage_channels=True)
async def unlock(ctx):
    channel = ctx.channel
    await channel.set_permissions(ctx.guild.default_role, send_messages=True)
    await ctx.send(embed=embed_response(ctx, f"{channel.mention} has been unlocked."))


@bot.command()
@commands.has_permissions(ban_members=True)
async def banlist(ctx):
    bans = await ctx.guild.bans()
    if not bans:
        return await ctx.send(embed=error_response(ctx, "No banned users found."))

    description = "\n".join([f"{ban.user} (`{ban.user.id}`)" for ban in bans[:20]])
    embed = discord.Embed(
        title="Banned Users",
        description=description + ("\n\nShowing first 20 only..." if len(bans) > 20 else ""),
        color=discord.Color.dark_theme()
    )
    embed.set_author(name=ctx.author.display_name, icon_url=ctx.author.display_avatar.url)
    await ctx.send(embed=embed)


@bot.command(aliases=["ir"])
async def inrole(ctx, *, role_input: str = None):
    if not role_input:
        embed = discord.Embed(
            title=f"Command: {ctx.invoked_with}",
            description="Show all members who have a specific role.",
            color=discord.Color.dark_theme()
        )
        embed.add_field(name="Category", value="Utility", inline=True)
        embed.add_field(name="Usage", value=f"```{ctx.prefix}{ctx.invoked_with} role_name / role_ID / @role```", inline=False)
        embed.set_author(name=ctx.author.display_name, icon_url=ctx.author.display_avatar.url)
        return await ctx.send(embed=embed)

    role = None
    if role_input.startswith("<@&") and role_input.endswith(">"):
        try:
            role_id = int(role_input[3:-1])
            role = ctx.guild.get_role(role_id)
        except:
            role = None
    elif role_input.isdigit():
        role = ctx.guild.get_role(int(role_input))
    else:
        role = discord.utils.get(ctx.guild.roles, name=role_input)

    if not role:
        return await ctx.send(embed=error_response(ctx, "Role not found."))

    members = [m.mention for m in role.members]
    if not members:
        return await ctx.send(embed=error_response(ctx, f"No members have the {role.name} role."))

    embed = discord.Embed(
        title=f"Members with {role.name}",
        description="\n".join(members[:30]) + ("\n\nShowing first 30 only..." if len(members) > 30 else ""),
        color=discord.Color.dark_theme()
    )
    embed.set_author(name=ctx.author.display_name, icon_url=ctx.author.display_avatar.url)
    await ctx.send(embed=embed)


@bot.command(aliases=["nn"])
@commands.has_permissions(manage_nicknames=True)
async def nickname(ctx, user: str = None, *, nickname: str = None):
    if not user or not nickname:
        embed = discord.Embed(
            title=f"Command: {ctx.invoked_with}",
            description="Change a member's nickname.",
            color=discord.Color.dark_theme()
        )
        embed.add_field(name="Category", value="Moderation", inline=True)
        embed.add_field(name="Usage", value=f"```{ctx.prefix}{ctx.invoked_with} @member/<ID> new_nickname```", inline=False)
        embed.set_author(name=ctx.author.display_name, icon_url=ctx.author.display_avatar.url)
        return await ctx.send(embed=embed)

    member = await get_member(ctx, user)
    if not member:
        return await ctx.send(embed=error_response(ctx, "Could not find that member."))

    try:
        await member.edit(nick=nickname)
        await ctx.send(embed=embed_response(ctx, f"Changed {member.mention}'s nickname to **{nickname}**."))
    except discord.Forbidden:
        await ctx.send(embed=error_response(ctx, "I don't have permission to change that nickname."))


@bot.command(aliases=["fn"])
@commands.has_permissions(administrator=True)
async def forcenickname(ctx, user: str = None, *, nickname: str = None):
    if not user or not nickname:
        embed = discord.Embed(
            title=f"Command: {ctx.invoked_with}",
            description="Force change a member's nickname (ignores normal permission checks).",
            color=discord.Color.dark_theme()
        )
        embed.add_field(name="Category", value="Moderation", inline=True)
        embed.add_field(name="Usage", value=f"```{ctx.prefix}{ctx.invoked_with} @member/<ID> new_nickname```", inline=False)
        embed.set_author(name=ctx.author.display_name, icon_url=ctx.author.display_avatar.url)
        return await ctx.send(embed=embed)

    member = await get_member(ctx, user)
    if not member:
        return await ctx.send(embed=error_response(ctx, "Could not find that member."))

    try:
        await member.edit(nick=nickname)
        await ctx.send(embed=embed_response(ctx, f"Force-changed {member.mention}'s nickname to **{nickname}**."))
    except discord.Forbidden:
        await ctx.send(embed=error_response(ctx, "I don't have permission to change that nickname."))

categories = {
    "Mist Help": (
        "**How to use**\n"
        "Use the dropdown menu below to view all bot command categories.\n\n"
        "**Made By Sav**\n"
        "Add godsofesex"
    ),
    
    "Moderation": (
        "**Moderate your server with these command.**\n"
        "```yaml\n"
        "nuke, ban, unban, clear,\n"
        "mute, unmute, kick,\n"
        "nickname, nn, forcenickname, fn\n"
        "role, autorole add, autorole remove, autorole list,\n"
        "inrole, banlist, on, off\n"
        "```"
    ),

    "Voice Master": (
        "**Temporary voice channels for your server.**\n"
        "```yaml\n"
        "voicemaster setup, voicemaster delete,\n"
        "vc lock, vc unlock,\n"
        "vc hide, vc reveal,\n"
        "vc limit, vc rename,\n"
        "vc reset, vc disconnect\n"
        "```"
    ),

    "Channel Set": (
        "**Set embeds for when people join or leave.**\n"
        "```yaml\n"
        "welcome, test_welcome,\n"
        "goodbye, test_goodbye\n"
        "```"
    ),
}


category_counts = {}
for key, val in categories.items():

    count = val.count(",") + 1 if "```" in val else 0
    category_counts[key] = count


total_commands = sum(category_counts.values())


class HelpSelect(discord.ui.Select):
    def __init__(self):
        options = [
            discord.SelectOption(label=key, description="View commands in this category")
            for key in categories
        ]
        super().__init__(placeholder="Select a Category", options=options)

    async def callback(self, interaction: discord.Interaction):
        selected = self.values[0]
        bot_avatar = interaction.client.user.display_avatar.url
        user = interaction.user

        if selected == "Mist Help":
            embed = discord.Embed(
                title="Help Menu",
                description=categories[selected],
                color=discord.Color.dark_theme()
            )
            embed.set_author(name=user.display_name, icon_url=user.display_avatar.url)
            embed.set_thumbnail(url=bot_avatar)
            embed.set_footer(text=f"Total commands: {total_commands}")
        else:
            embed = discord.Embed(
                title=selected,
                description=categories[selected],
                color=discord.Color.dark_theme()
            )
            embed.set_author(name=user.display_name, icon_url=user.display_avatar.url)
            embed.set_thumbnail(url=bot_avatar)
            embed.set_footer(text=f"Commands in this category: {category_counts.get(selected, 0)}")

        await interaction.response.edit_message(embed=embed, view=self.view)

class HelpView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)
        self.add_item(HelpSelect())


@bot.command(aliases=["help", "commands", "cmds"])
async def h(ctx):
    invalid_emoji = "<:invalid:1414337316524134592>"

    def is_whitelisted(member):
        return True  

    def error_response(message: str):
        return discord.Embed(
            description=f"{invalid_emoji} {ctx.author.mention} {message}",
            color=discord.Color(0xfee75d)
        )

    if not is_whitelisted(ctx.author):
        return await ctx.send(embed=error_response("You can’t use this command"))

    bot_avatar = ctx.bot.user.display_avatar.url
    user = ctx.author

    embed = discord.Embed(
        title="Help Menu",
        description=categories["Mist Help"],
        color=discord.Color.dark_theme()
    )
    embed.set_author(name=user.display_name, icon_url=user.display_avatar.url)
    embed.set_thumbnail(url=bot_avatar)
    embed.set_footer(text=f"Total commands: {total_commands}")

    await ctx.send(embed=embed, view=HelpView())



@bot.event
async def on_ready():
    bot.add_view(PingOnJoinView(guild_id=0)) 
    bot.add_view(VoiceControlPanel())

    print(f"{bot.user} (ID: {bot.user.id}) is dominating rn.")
    await bot.tree.sync()
    print("VoiceMaster buttons working")


bot.run(TOKEN)
