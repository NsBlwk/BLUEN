import discord
from discord.ext import commands
from discord import app_commands
import asyncio  # Adicionado para o sleep de 5 segundos
import io
import time  # Para controlar o tempo de uptime

class MeuBot(commands.Bot):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.start_time = time.time()  # Marca o tempo de inicialização do bot

    async def on_ready(self):
        await self.tree.sync()
        print(f"Bot conectado como {self.user}")

intents = discord.Intents.all()
client = MeuBot(command_prefix="!", intents=intents)

# CONFIGURAÇÕES
CATEGORIA_TICKETS = "Tickets"
CANAL_LOGS = "logs-tickets"
CARGO_STAFF = "🎫/Ticket's"

# Função para criar categoria e logs se não existir
async def setup_categoria_e_logs(guild):
    categoria = discord.utils.get(guild.categories, name=CATEGORIA_TICKETS)
    if not categoria:
        categoria = await guild.create_category(CATEGORIA_TICKETS)

    canal_logs = discord.utils.get(guild.text_channels, name=CANAL_LOGS)
    if not canal_logs:
        canal_logs = await guild.create_text_channel(CANAL_LOGS)

    return categoria, canal_logs

# Comando de enviar Painel
@client.tree.command(name="painel", description="Envia o Painel de Abertura de Tickets")
async def painel(interaction: discord.Interaction):
    await setup_categoria_e_logs(interaction.guild)

    embed = discord.Embed(
        title="🎟️ Painel de Suporte",
        description=(
            "Escolha abaixo o motivo para abrir seu ticket:\n\n"
            "🛠️ **Suporte** - Dúvidas, bugs ou ajuda.\n"
            "🤝 **Parcerias** - Solicitar parceria.\n"
            "⚠️ **Denúncia** - Reportar usuário/comportamento.\n"
            "❓ **Outros Assuntos** - Atendimento geral.\n\n"
            "Clique em um dos botões!"
        ),
        color=discord.Color.blurple()
    )
    await interaction.channel.send(embed=embed, view=MultiTicketView())
    await interaction.response.send_message("✅ Painel enviado!", ephemeral=True)

# View dos botões
class MultiTicketView(discord.ui.View):
    @discord.ui.button(label="Suporte", style=discord.ButtonStyle.primary, emoji="🛠️")
    async def suporte(self, interaction: discord.Interaction, button: discord.ui.Button):
        await criar_ticket(interaction, "suporte")

    @discord.ui.button(label="Parcerias", style=discord.ButtonStyle.success, emoji="🤝")
    async def parcerias(self, interaction: discord.Interaction, button: discord.ui.Button):
        await criar_ticket(interaction, "parceria")

    @discord.ui.button(label="Denúncia", style=discord.ButtonStyle.danger, emoji="⚠️")
    async def denuncia(self, interaction: discord.Interaction, button: discord.ui.Button):
        await criar_ticket(interaction, "denuncia")

    @discord.ui.button(label="Outros", style=discord.ButtonStyle.secondary, emoji="❓")
    async def outros(self, interaction: discord.Interaction, button: discord.ui.Button):
        await criar_ticket(interaction, "outros")

# Função para criar Ticket
async def criar_ticket(interaction: discord.Interaction, tipo: str):
    guild = interaction.guild
    categoria, canal_logs = await setup_categoria_e_logs(guild)
    cargo_staff = discord.utils.get(guild.roles, name=CARGO_STAFF)

    overwrites = {
        guild.default_role: discord.PermissionOverwrite(read_messages=False),
        interaction.user: discord.PermissionOverwrite(read_messages=True, send_messages=True),
    }

    if cargo_staff:
        overwrites[cargo_staff] = discord.PermissionOverwrite(read_messages=True, send_messages=True)

    ticket_channel = await guild.create_text_channel(
        name=f"{tipo}-{interaction.user.name}".replace(" ", "-").lower(),
        overwrites=overwrites,
        category=categoria,
        reason=f"Ticket de {tipo} criado por {interaction.user}"
    )

    embed = discord.Embed(
        title="🎟️ Ticket Aberto",
        description=f"Olá {interaction.user.mention}!\n\nAguarde, um membro da equipe irá te atender.\n\n**Motivo:** {tipo.capitalize()}",
        color=discord.Color.blurple()
    )
    await ticket_channel.send(embed=embed, view=FecharTicketView())

    await canal_logs.send(f"✅ Ticket `{ticket_channel.name}` aberto por {interaction.user.mention} ({interaction.user.id}) para **{tipo.capitalize()}**.")

    await interaction.response.send_message(f"✅ Seu ticket foi criado: {ticket_channel.mention}", ephemeral=True)

# View do botão Fechar Ticket
class FecharTicketView(discord.ui.View):
    @discord.ui.button(label="Fechar Ticket", style=discord.ButtonStyle.danger, emoji="🔒")
    async def fechar(self, interaction: discord.Interaction, button: discord.ui.Button):
        await fechar_ticket(interaction)

# Função para fechar Ticket
async def fechar_ticket(interaction: discord.Interaction):
    guild = interaction.guild
    canal_logs = discord.utils.get(guild.text_channels, name=CANAL_LOGS)

    if interaction.channel.category and interaction.channel.category.name == CATEGORIA_TICKETS:
        await interaction.response.send_message("🔒 Fechando o ticket em 5 segundos...", ephemeral=True)
        await asyncio.sleep(5)
        nome_canal = interaction.channel.name
        await interaction.channel.delete()

        if canal_logs:
            await canal_logs.send(f"❌ Ticket `{nome_canal}` fechado por {interaction.user.mention} ({interaction.user.id}).")
    else:
        await interaction.response.send_message("❌ Este comando só pode ser usado dentro de um ticket.", ephemeral=True)

# Comando de Slash /fechar (dono do ticket ou staff)
@client.tree.command(name="fechar", description="Fecha o ticket atual")
async def slash_fechar(interaction: discord.Interaction):
    guild = interaction.guild
    canal_logs = discord.utils.get(guild.text_channels, name=CANAL_LOGS)
    cargo_staff = discord.utils.get(guild.roles, name=CARGO_STAFF)

    if interaction.channel.category and interaction.channel.category.name == CATEGORIA_TICKETS:
        dono_ticket = None
        for perm in interaction.channel.overwrites:
            if isinstance(perm, discord.Member):
                dono_ticket = perm
                break

        if interaction.user == dono_ticket or (cargo_staff and cargo_staff in interaction.user.roles):
            await interaction.response.send_message("🔒 Fechando o ticket em 5 segundos...", ephemeral=True)
            await asyncio.sleep(5)
            nome_canal = interaction.channel.name
            await interaction.channel.delete()

            if canal_logs:
                await canal_logs.send(f"❌ Ticket `{nome_canal}` fechado via `/fechar` por {interaction.user.mention} ({interaction.user.id}).")
        else:
            await interaction.response.send_message("❌ Você não tem permissão para fechar este ticket.", ephemeral=True)
    else:
        await interaction.response.send_message("❌ Este comando só pode ser usado dentro de um ticket.", ephemeral=True)

# Comando de Slash /embed
@client.tree.command(name="embed", description="Cria um embed personalizado avançado")
@app_commands.describe(
    titulo="Título do Embed",
    descricao="Descrição do Embed",
    cor="Cor em hexadecimal (ex: #3498db)",
    thumbnail="URL da imagem para thumbnail (opcional)",
    imagem="URL da imagem grande do embed (opcional)",
    footer="Texto do rodapé (opcional)",
    autor="Texto do autor do embed (opcional)"
)
async def embed(
    interaction: discord.Interaction,
    titulo: str,
    descricao: str,
    cor: str = "#5865F2",
    thumbnail: str = None,
    imagem: str = None,
    footer: str = None,
    autor: str = None
):
    try:
        color_int = int(cor.replace("#", ""), 16)
    except ValueError:
        await interaction.response.send_message("❌ Cor inválida! Use um formato hexadecimal, exemplo: `#5865F2`.", ephemeral=True)
        return

    embed = discord.Embed(
        title=titulo,
        description=descricao,
        color=color_int
    )

    if autor:
        embed.set_author(name=autor)
    if thumbnail:
        embed.set_thumbnail(url=thumbnail)
    if imagem:
        embed.set_image(url=imagem)
    if footer:
        embed.set_footer(text=footer)

    await interaction.channel.send(embed=embed)
    await interaction.response.send_message("✅ Embed enviado!", ephemeral=True)

# Comando de Slash /ping
@client.tree.command(name="ping", description="Mostra o ping e informações do bot.")
async def ping(interaction: discord.Interaction):
    latency = round(client.latency * 1000)

    embed = discord.Embed(
        title="🏓 Status do Bot",
        color=discord.Color.blurple()
    )

    embed.add_field(name="Ping", value=f"`{latency}ms`", inline=True)
    embed.add_field(name="Servidores", value=f"`{len(client.guilds)}`", inline=True)
    embed.add_field(name="Usuários", value=f"`{sum(g.member_count for g in client.guilds)}`", inline=True)
    embed.set_footer(text=f"Pedido por {interaction.user.name}", icon_url=interaction.user.avatar.url if interaction.user.avatar else None)

    await interaction.response.send_message(embed=embed, ephemeral=True)

# Comando de Slash /botinfo
@client.tree.command(name="botinfo", description="Mostra informações sobre o bot")
async def botinfo(interaction: discord.Interaction):
    embed = discord.Embed(
        title="📊 Informações do Bot",
        description=f"**Bot**: {client.user.name}\n**ID**: {client.user.id}",
        color=discord.Color.blurple()
    )
    embed.add_field(name="Versão do Discord.py", value=discord.__version__)
    embed.add_field(name="Total de Servidores", value=f"{len(client.guilds)}")
    embed.add_field(name="Total de Usuários", value=f"{sum(g.member_count for g in client.guilds)}")
    embed.set_footer(text=f"Pedido por {interaction.user.name}", icon_url=interaction.user.avatar.url if interaction.user.avatar else None)

    await interaction.response.send_message(embed=embed, ephemeral=True)

# Comando de Slash /uptime
@client.tree.command(name="uptime", description="Mostra o tempo em que o bot está online")
async def uptime(interaction: discord.Interaction):
    uptime_seconds = int(time.time() - client.start_time)
    uptime_str = str(datetime.timedelta(seconds=uptime_seconds))

    embed = discord.Embed(
        title="⏳ Tempo de Uptime",
        description=f"O bot está online há `{uptime_str}`.",
        color=discord.Color.blurple()
    )

    await interaction.response.send_message(embed=embed, ephemeral=True)

# TOKEN
client.run('MTM2NjE1Nzg2MTkwMTgzMjI1Mw.GwNOKW.MRqEgGT_vY5r6StukbpvnNUJd3QaZge8A4qGpE')  # <<< Substitua pelo seu token real!!
