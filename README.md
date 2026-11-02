# 0800 Beats

Bot de música para Discord desenvolvido em Python.

A ideia do projeto é oferecer uma opção gratuita para reproduzir músicas no Discord, sem depender de bots com funcionalidades pagas.

## Funcionalidades

- Reprodução de músicas do YouTube por link ou pesquisa.
- Suporte a links de músicas, álbuns e playlists do Spotify.
- Fila de reprodução.
- Histórico de músicas.
- Pausar e retomar músicas.
- Avançar ou voltar músicas.
- Repetir a música atual.
- Controle de volume de 0% a 150%.
- Iniciar músicas a partir de um tempo específico.
- Mostrar a música atual e o progresso da reprodução.
- Botões para controlar a música diretamente pelo Discord.
- Comandos tradicionais e Slash Commands.
- Filas independentes para cada servidor.

## Comandos

| Comando | Função |
|---|---|
| `!play` | Tocar uma música |
| `!pausar` | Pausar |
| `!retomar` | Retomar |
| `!avancar` | Pular música |
| `!voltar` | Voltar para a música anterior |
| `!loop` | Ativar ou desativar repetição |
| `!volume` | Alterar volume |
| `!np` | Mostrar música atual |
| `!parar` | Parar e desconectar |

Todos os comandos também podem ser utilizados com `/`.

## Tecnologias

- Python
- discord.py
- yt-dlp
- FFmpeg
- aiohttp
- Spotipy

## Como executar

É necessário ter Python 3.10 ou superior e FFmpeg instalado.

Clone o repositório:

```bash
git clone https://github.com/SEU_USUARIO/0800-beats.git
cd 0800-beats
```

Instale as dependências:

```bash
pip install discord.py yt-dlp aiohttp spotipy
```

Configure o token do bot na variável de ambiente `DISCORD_TOKEN`.

Execute o projeto:

```bash
python music.py
```

## Spotify

A integração com a API do Spotify é opcional. Sem as credenciais, o bot utiliza o oEmbed para obter informações dos links.
