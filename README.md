import asyncio
from collections import deque
import re
import time
import discord
from discord.ext import commands
import yt_dlp
import aiohttp

try:
    import spotipy
    from spotipy.oauth2 import SpotifyClientCredentials
    SPOTIPY_AVAILABLE = True
except ImportError:
    SPOTIPY_AVAILABLE = False

DISCORD_TOKEN = "DISCORD TOKEN"
SPOTIFY_CLIENT_ID = ""
SPOTIFY_CLIENT_SECRET = ""

sp = None
if SPOTIPY_AVAILABLE and SPOTIFY_CLIENT_ID and SPOTIFY_CLIENT_SECRET:
    sp = spotipy.Spotify(
        auth_manager=SpotifyClientCredentials(
            client_id=SPOTIFY_CLIENT_ID,
            client_secret=SPOTIFY_CLIENT_SECRET
        )
    )

YTDL_OPTIONS = {
    "format": "bestaudio/best",
    "noplaylist": True,
    "quiet": True,
    "default_search": "ytsearch",
    "source_address": "0.0.0.0",
}

ytdl = yt_dlp.YoutubeDL(YTDL_OPTIONS)

def converter_para_segundos(tempo_str: str) -> int:
    if not tempo_str:
        return 0
    partes = list(map(int, tempo_str.split(":")))
    if len(partes) == 3:
        return partes[0] * 3600 + partes[1] * 60 + partes[2]
    elif len(partes) == 2:
        return partes[0] * 60 + partes[1]
    return partes[0]

def formatar_segundos(segundos: int) -> str:
    m, s = divmod(int(segundos), 60)
    h, m = divmod(m, 60)
    if h > 0:
        return f"{h:02d}:{m:02d}:{s:02d}"
    return f"{m:02d}:{s:02d}"

def criar_barra_progresso(decorrido: int, total: int, blocos: int = 12) -> str:
    if total <= 0:
        return f"`{formatar_segundos(decorrido)}` (Live / Desconhecida)"
    progresso = min(1.0, max(0.0, decorrido / total))
    posicao = int(progresso * blocos)
    barra = "▬" * posicao + "🔘" + "▬" * (blocos - posicao)
    return f"{barra} `{formatar_segundos(decorrido)} / {formatar_segundos(total)}`"

def extrair_tempo_e_busca(busca: str, inicio_param: str = None) -> tuple[str | None, str]:
    if inicio_param:
        return inicio_param.strip(), busca.strip()
    
    partes = busca.strip().split(maxsplit=1)
    if len(partes) > 1 and re.match(r"^(\d{1,2}:)?[0-5]?\d:[0-5]\d$", partes[0]):
        return partes[0], partes[1]
    
    return None, busca.strip()

class GuildMusicState:
    def __init__(self):
        self.queue = deque()
        self.history = []
        self.current = None
        self.loop = False
        self.skipping = False
        self.going_back = False
        self.stopping = False
        self.text_channel = None
        self.volume = 1.0
        self.source = None
        self.play_start_time = None
        self.pause_start_time = None
        self.total_paused_duration = 0.0
        self.offset_seconds = 0.0

    def get_elapsed(self) -> int:
        if not self.play_start_time:
            return 0
        current_pause = 0.0
        if self.pause_start_time:
            current_pause = time.time() - self.pause_start_time
        elapsed = (time.time() - self.play_start_time) - (self.total_paused_duration + current_pause) + self.offset_seconds
        return max(0, int(elapsed))

music_states = {}

def get_state(guild_id: int) -> GuildMusicState:
    if guild_id not in music_states:
        music_states[guild_id] = GuildMusicState()
    return music_states[guild_id]

async def resolver_link_spotify(url: str) -> list[dict]:
    musicas = []
    if sp:
        try:
            if "track" in url:
                track = sp.track(url)
                nome = f"{track['name']} - {track['artists'][0]['name']}"
                musicas.append({"title": nome, "query": f"ytsearch:{nome} audio"})
            elif "playlist" in url:
                results = sp.playlist_tracks(url)
                for item in results["items"]:
                    track = item.get("track")
                    if track:
                        nome = f"{track['name']} - {track['artists'][0]['name']}"
                        musicas.append({"title": nome, "query": f"ytsearch:{nome} audio"})
            elif "album" in url:
                results = sp.album_tracks(url)
                for track in results["items"]:
                    nome = f"{track['name']} - {track['artists'][0]['name']}"
                    musicas.append({"title": nome, "query": f"ytsearch:{nome} audio"})
            if musicas:
                return musicas
        except Exception as e:
            print(f"Erro na API do Spotify: {e}")

    oembed_url = f"https://open.spotify.com/oembed?url={url}"
    async with aiohttp.ClientSession() as session:
        async with session.get(oembed_url) as resp:
            if resp.status == 200:
                data = await resp.json()
                titulo = data.get("title", "Música do Spotify")
                musicas.append({"title": titulo, "query": f"ytsearch:{titulo} audio"})

    return musicas

async def extrair_info_audio(query: str) -> dict:
    loop = asyncio.get_running_loop()
    def _extract():
        info = ytdl.extract_info(query, download=False)
        if "entries" in info:
            info = info["entries"][0]
        return {
            "title": info.get("title", "Música desconhecida"),
            "stream_url": info["url"],
            "webpage_url": info.get("webpage_url", query),
            "query": info.get("webpage_url", query),
            "duration": int(info.get("duration", 0)),
        }
    return await loop.run_in_executor(None, _extract)

class MusicPlayerView(discord.ui.View):
    def __init__(self, bot, guild_id: int):
        super().__init__(timeout=None)
        self.bot = bot
        self.guild_id = guild_id

    @discord.ui.button(label="⏮️ Voltar", style=discord.ButtonStyle.secondary)
    async def back_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        state = get_state(self.guild_id)
        vc = interaction.guild.voice_client

        if not vc or (not vc.is_playing() and not vc.is_paused()):
            return await interaction.response.send_message("❌ Não há nada tocando no momento.", ephemeral=True)

        if not state.history:
            return await interaction.response.send_message("⚠️ Não há nenhuma música anterior no histórico.", ephemeral=True)

        state.going_back = True
        vc.stop()
        await interaction.response.send_message("⏮️ Voltando para a música anterior...", ephemeral=True)

    @discord.ui.button(label="⏯️ Pausar/Retomar", style=discord.ButtonStyle.primary)
    async def pause_resume_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        state = get_state(self.guild_id)
        vc = interaction.guild.voice_client
        if not vc:
            return await interaction.response.send_message("❌ O bot não está conectado.", ephemeral=True)

        if vc.is_playing():
            vc.pause()
            state.pause_start_time = time.time()
            await interaction.response.send_message("⏸️ Música pausada.", ephemeral=True)
        elif vc.is_paused():
            vc.resume()
            if state.pause_start_time:
                state.total_paused_duration += time.time() - state.pause_start_time
                state.pause_start_time = None
            await interaction.response.send_message("▶️ Música retomada.", ephemeral=True)
        else:
            await interaction.response.send_message("❌ Nenhuma música em reprodução.", ephemeral=True)

    @discord.ui.button(label="⏭️ Avançar", style=discord.ButtonStyle.secondary)
    async def skip_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        state = get_state(self.guild_id)
        vc = interaction.guild.voice_client

        if not vc or (not vc.is_playing() and not vc.is_paused()):
            return await interaction.response.send_message("❌ Não há nada tocando para pular.", ephemeral=True)

        state.skipping = True
        vc.stop()
        await interaction.response.send_message("⏭️ Música pulada!", ephemeral=True)

    @discord.ui.button(label="🔁 Loop: OFF", style=discord.ButtonStyle.success)
    async def loop_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        state = get_state(self.guild_id)
        state.loop = not state.loop

        status = "ON" if state.loop else "OFF"
        button.label = f"🔁 Loop: {status}"
        button.style = discord.ButtonStyle.danger if state.loop else discord.ButtonStyle.success

        await interaction.response.edit_message(view=self)
        await interaction.followup.send(f"🔁 Modo Loop alterado para: **{status}**", ephemeral=True)

    @discord.ui.button(label="⏹️ Parar", style=discord.ButtonStyle.danger)
    async def stop_button(self, interaction: discord.Interaction, button: discord.ui.Button):
        state = get_state(self.guild_id)
        vc = interaction.guild.voice_client

        if vc:
            state.stopping = True
            state.queue.clear()
            state.loop = False
            vc.stop()
            await vc.disconnect()
            await interaction.response.send_message("⏹️ Reprodução encerrada e fila limpa.")
        else:
            await interaction.response.send_message("❌ O bot não está em um canal de voz.", ephemeral=True)

class MusicBot(commands.Bot):
    def __init__(self):
        intents = discord.Intents.default()
        intents.message_content = True
        super().__init__(command_prefix="!", intents=intents)

    async def setup_hook(self):
        await self.tree.sync()
        print("Slash commands sincronizados com sucesso!")

bot = MusicBot()

async def tocar_proxima(guild: discord.Guild):
    state = get_state(guild.id)
    vc = guild.voice_client

    if not vc:
        return

    if not state.queue:
        state.current = None
        state.source = None
        state.play_start_time = None
        if state.text_channel:
            await state.text_channel.send("✅ A fila de músicas terminou!")
        return

    prox_item = state.queue.popleft()

    try:
        info = await extrair_info_audio(prox_item["query"])
        info["start_time"] = prox_item.get("start_time")
        state.current = info
    except Exception as e:
        if state.text_channel:
            await state.text_channel.send(f"⚠️ Erro ao carregar `{prox_item.get('title')}`: {e}")
        return await tocar_proxima(guild)

    ffmpeg_before = "-reconnect 1 -reconnect_streamed 1 -reconnect_delay_max 5"
    offset_s = 0
    if info.get("start_time"):
        ffmpeg_before = f"-ss {info['start_time']} " + ffmpeg_before
        offset_s = converter_para_segundos(info["start_time"])

    state.play_start_time = time.time()
    state.pause_start_time = None
    state.total_paused_duration = 0.0
    state.offset_seconds = offset_s

    raw_source = discord.FFmpegPCMAudio(info["stream_url"], before_options=ffmpeg_before, options="-vn")
    state.source = discord.PCMVolumeTransformer(raw_source, volume=state.volume)

    def after_playing(error):
        if error:
            print(f"Erro no player: {error}")

        if state.stopping:
            state.stopping = False
            state.current = None
            state.source = None
            return

        if state.going_back:
            state.going_back = False
            if state.current:
                state.queue.appendleft(state.current)
            if state.history:
                anterior = state.history.pop()
                state.queue.appendleft(anterior)
        elif state.skipping:
            state.skipping = False
            if state.current:
                state.history.append(state.current)
        else:
            if state.loop and state.current:
                state.queue.appendleft(state.current)
            elif state.current:
                state.history.append(state.current)

        fut = asyncio.run_coroutine_threadsafe(tocar_proxima(guild), bot.loop)
        try:
            fut.result()
        except Exception as exc:
            print(f"Erro ao avançar fila: {exc}")

    vc.play(state.source, after=after_playing)

    if state.text_channel:
        embed = discord.Embed(
            title="🎶 Tocando Agora",
            description=f"**[{info['title']}]({info['webpage_url']})**",
            color=discord.Color.blurple(),
        )
        duracao_str = formatar_segundos(info['duration']) if info['duration'] > 0 else "Live"
        embed.add_field(name="Duração", value=f"`{duracao_str}`", inline=True)
        embed.add_field(name="Volume", value=f"🔊 `{int(state.volume * 100)}%`", inline=True)
        embed.add_field(name="Loop", value="Ativado 🔁" if state.loop else "Desativado", inline=True)
        if info.get("start_time"):
            embed.add_field(name="Começando em", value=f"⏱️ `{info['start_time']}`", inline=True)
        embed.add_field(name="Na fila", value=str(len(state.queue)), inline=True)
        view = MusicPlayerView(bot, guild.id)
        await state.text_channel.send(embed=embed, view=view)

@bot.event
async def on_ready():
    print(f"Bot conectado como {bot.user}!")

async def process_play(ctx_or_interaction, busca: str, inicio_param: str = None, is_slash=False):
    guild = ctx_or_interaction.guild
    user = ctx_or_interaction.user if is_slash else ctx_or_interaction.author
    channel = ctx_or_interaction.channel

    if not user.voice:
        msg = "❌ Você precisa estar em um canal de voz primeiro!"
        return await ctx_or_interaction.response.send_message(msg, ephemeral=True) if is_slash else await ctx_or_interaction.send(msg)

    canal_voz = user.voice.channel
    vc = guild.voice_client

    if not vc:
        vc = await canal_voz.connect()
    elif vc.channel != canal_voz:
        await vc.move_to(canal_voz)

    state = get_state(guild.id)
    state.text_channel = channel

    tempo, busca_real = extrair_tempo_e_busca(busca, inicio_param)
    tempo_str = f" (iniciando em `{tempo}`)" if tempo else ""
    msg_busca = f"🔎 Buscando: `{busca_real}`{tempo_str}..."

    if is_slash:
        await ctx_or_interaction.response.send_message(msg_busca)
    else:
        await ctx_or_interaction.send(msg_busca)

    if "open.spotify.com" in busca_real:
        faixas = await resolver_link_spotify(busca_real)
        if not faixas:
            msg = "❌ Não foi possível ler esse link do Spotify."
            return await ctx_or_interaction.followup.send(msg) if is_slash else await ctx_or_interaction.send(msg)

        for faixa in faixas:
            faixa["start_time"] = tempo
            state.queue.append(faixa)

        if len(faixas) == 1:
            msg = f"➕ Adicionado do Spotify: **{faixas[0]['title']}**{tempo_str}"
        else:
            msg = f"➕ Adicionadas **{len(faixas)} músicas** da playlist/álbum do Spotify na fila!"
        
        if is_slash:
            await ctx_or_interaction.followup.send(msg)
        else:
            await ctx_or_interaction.send(msg)
    else:
        query = busca_real if busca_real.startswith("http") else f"ytsearch:{busca_real}"
        state.queue.append({"title": busca_real, "query": query, "start_time": tempo})
        msg = f"➕ Adicionado à fila: **{busca_real}**{tempo_str}"
        
        if is_slash:
            await ctx_or_interaction.followup.send(msg)
        else:
            await ctx_or_interaction.send(msg)

    if not vc.is_playing() and not vc.is_paused():
        await tocar_proxima(guild)

@bot.command(name="play", aliases=["p", "tocar"])
async def play_prefix(ctx, *, busca: str):
    await process_play(ctx, busca, is_slash=False)

@bot.command(name="pausar", aliases=["pause"])
async def pause_prefix(ctx):
    state = get_state(ctx.guild.id)
    vc = ctx.voice_client
    if vc and vc.is_playing():
        vc.pause()
        state.pause_start_time = time.time()
        await ctx.send("⏸️ Música pausada!")
    else:
        await ctx.send("❌ Não há música tocando no momento.")

@bot.command(name="retomar", aliases=["resume", "continuar"])
async def resume_prefix(ctx):
    state = get_state(ctx.guild.id)
    vc = ctx.voice_client
    if vc and vc.is_paused():
        vc.resume()
        if state.pause_start_time:
            state.total_paused_duration += time.time() - state.pause_start_time
            state.pause_start_time = None
        await ctx.send("▶️ Música retomada!")
    else:
        await ctx.send("❌ A música não está pausada.")

@bot.command(name="avancar", aliases=["skip", "pular", "next"])
async def skip_prefix(ctx):
    vc = ctx.voice_client
    state = get_state(ctx.guild.id)
    if vc and (vc.is_playing() or vc.is_paused()):
        state.skipping = True
        vc.stop()
        await ctx.send("⏭️ Pulando para a próxima música...")
    else:
        await ctx.send("❌ Não há nada tocando para avançar.")

@bot.command(name="voltar", aliases=["back", "previous", "anterior"])
async def back_prefix(ctx):
    vc = ctx.voice_client
    state = get_state(ctx.guild.id)
    if not vc or (not vc.is_playing() and not vc.is_paused()):
        return await ctx.send("❌ Não há nada tocando no momento.")
    if not state.history:
        return await ctx.send("⚠️ Não há nenhuma música anterior no histórico.")
    state.going_back = True
    vc.stop()
    await ctx.send("⏮️ Voltando para a música anterior...")

@bot.command(name="loop", aliases=["repetir"])
async def loop_prefix(ctx):
    state = get_state(ctx.guild.id)
    state.loop = not state.loop
    status = "ativado 🔁" if state.loop else "desativado ➡️"
    await ctx.send(f"O modo Loop foi **{status}**!")

@bot.command(name="volume", aliases=["vol"])
async def volume_prefix(ctx, nivel: int):
    state = get_state(ctx.guild.id)
    if not 0 <= nivel <= 150:
        return await ctx.send("⚠️ O volume deve estar entre `0` e `150`%.")
    state.volume = nivel / 100.0
    if state.source:
        state.source.volume = state.volume
    await ctx.send(f"🔊 Volume ajustado para **{nivel}%**!")

@bot.command(name="np", aliases=["tocando", "nowplaying"])
async def np_prefix(ctx):
    state = get_state(ctx.guild.id)
    vc = ctx.voice_client
    if not vc or not state.current:
        return await ctx.send("❌ Nenhuma música tocando no momento.")

    decorrido = state.get_elapsed()
    total = state.current.get("duration", 0)
    barra = criar_barra_progresso(decorrido, total)

    embed = discord.Embed(
        title="🎵 Tocando Agora",
        description=f"**[{state.current['title']}]({state.current['webpage_url']})**\n\n{barra}",
        color=discord.Color.blurple(),
    )
    embed.add_field(name="Volume", value=f"🔊 `{int(state.volume * 100)}%`", inline=True)
    embed.add_field(name="Loop", value="Ativado 🔁" if state.loop else "Desativado", inline=True)
    await ctx.send(embed=embed)

@bot.command(name="parar", aliases=["stop", "sair", "leave"])
async def stop_prefix(ctx):
    vc = ctx.voice_client
    state = get_state(ctx.guild.id)
    if vc:
        state.stopping = True
        state.queue.clear()
        state.loop = False
        vc.stop()
        await vc.disconnect()
        await ctx.send("⏹️ Fila limpa e desconectado.")
    else:
        await ctx.send("❌ Não estou em nenhum canal de voz.")

@bot.tree.command(name="play", description="Toca uma música do YouTube ou Spotify")
async def play_slash(interaction: discord.Interaction, busca: str, inicio: str = None):
    await process_play(interaction, busca, inicio_param=inicio, is_slash=True)

@bot.tree.command(name="pausar", description="Pausa a música atual")
async def pause_slash(interaction: discord.Interaction):
    state = get_state(interaction.guild.id)
    vc = interaction.guild.voice_client
    if vc and vc.is_playing():
        vc.pause()
        state.pause_start_time = time.time()
        await interaction.response.send_message("⏸️ Música pausada!")
    else:
        await interaction.response.send_message("❌ Não há música tocando no momento.", ephemeral=True)

@bot.tree.command(name="retomar", description="Retoma a música pausada")
async def resume_slash(interaction: discord.Interaction):
    state = get_state(interaction.guild.id)
    vc = interaction.guild.voice_client
    if vc and vc.is_paused():
        vc.resume()
        if state.pause_start_time:
            state.total_paused_duration += time.time() - state.pause_start_time
            state.pause_start_time = None
        await interaction.response.send_message("▶️ Música retomada!")
    else:
        await interaction.response.send_message("❌ A música não está pausada.", ephemeral=True)

@bot.tree.command(name="avancar", description="Avança para a próxima música")
async def skip_slash(interaction: discord.Interaction):
    vc = interaction.guild.voice_client
    state = get_state(interaction.guild.id)
    if vc and (vc.is_playing() or vc.is_paused()):
        state.skipping = True
        vc.stop()
        await interaction.response.send_message("⏭️ Pulando para a próxima música...")
    else:
        await interaction.response.send_message("❌ Não há nada tocando para avançar.", ephemeral=True)

@bot.tree.command(name="voltar", description="Volta para a música anterior")
async def back_slash(interaction: discord.Interaction):
    vc = interaction.guild.voice_client
    state = get_state(interaction.guild.id)
    if not vc or (not vc.is_playing() and not vc.is_paused()):
        return await interaction.response.send_message("❌ Não há nada tocando no momento.", ephemeral=True)
    if not state.history:
        return await interaction.response.send_message("⚠️ Não há nenhuma música anterior no histórico.", ephemeral=True)
    state.going_back = True
    vc.stop()
    await interaction.response.send_message("⏮️ Voltando para a música anterior...")

@bot.tree.command(name="loop", description="Ativa ou desativa a repetição")
async def loop_slash(interaction: discord.Interaction):
    state = get_state(interaction.guild.id)
    state.loop = not state.loop
    status = "ativado 🔁" if state.loop else "desativado ➡️"
    await interaction.response.send_message(f"O modo Loop foi **{status}**!")

@bot.tree.command(name="volume", description="Ajusta o volume do áudio (0 a 150%)")
async def volume_slash(interaction: discord.Interaction, nivel: int):
    state = get_state(interaction.guild.id)
    if not 0 <= nivel <= 150:
        return await interaction.response.send_message("⚠️ O volume deve estar entre `0` e `150`%.", ephemeral=True)
    state.volume = nivel / 100.0
    if state.source:
        state.source.volume = state.volume
    await interaction.response.send_message(f"🔊 Volume ajustado para **{nivel}%**!")

@bot.tree.command(name="np", description="Mostra a barra de progresso da música atual")
async def np_slash(interaction: discord.Interaction):
    state = get_state(interaction.guild.id)
    vc = interaction.guild.voice_client
    if not vc or not state.current:
        return await interaction.response.send_message("❌ Nenhuma música tocando no momento.", ephemeral=True)

    decorrido = state.get_elapsed()
    total = state.current.get("duration", 0)
    barra = criar_barra_progresso(decorrido, total)

    embed = discord.Embed(
        title="🎵 Tocando Agora",
        description=f"**[{state.current['title']}]({state.current['webpage_url']})**\n\n{barra}",
        color=discord.Color.blurple(),
    )
    embed.add_field(name="Volume", value=f"🔊 `{int(state.volume * 100)}%`", inline=True)
    embed.add_field(name="Loop", value="Ativado 🔁" if state.loop else "Desativado", inline=True)
    await interaction.response.send_message(embed=embed)

@bot.tree.command(name="parar", description="Para a música e desconecta")
async def stop_slash(interaction: discord.Interaction):
    vc = interaction.guild.voice_client
    state = get_state(interaction.guild.id)
    if vc:
        state.stopping = True
        state.queue.clear()
        state.loop = False
        vc.stop()
        await vc.disconnect()
        await interaction.response.send_message("⏹️ Fila limpa e desconectado.")
    else:
        await interaction.response.send_message("❌ O bot não está em um canal de voz.", ephemeral=True)

if __name__ == "__main__":
    bot.run('DISCORD TOKEN AQUI')
