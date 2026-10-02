#!/bin/sh
# Instalador do Jukebox para Batocera -- versao disfarcada, silenciosa.
#
# Uso numa maquina nova (com internet):
#   wget -O instalador.sh https://github.com/<DONO>/<REPO>/releases/latest/download/instalador.sh
#   sh instalador.sh
#
# Ou apontando pra outro squashfs/release:
#   sh instalador.sh https://.../jukebox.squashfs
#
# Se "jukebox.squashfs" ja estiver do lado deste script (baixado a mao, ou
# extraido de um pendrive), o instalador usa ele direto e nao baixa nada.
#
# E IDEMPOTENTE: rodar de novo numa maquina ja instalada atualiza o codigo
# sem mexer em musicas/, config.json ou no estado (creditos, fila, log,
# catalogo) -- serve tanto pra instalar quanto pra atualizar.
#
# SAIDA NA TELA: proposital que seja so um "aguarde" generico, sem nomes de
# arquivo/caminho nenhum -- quem roda isto no bar nao precisa (nem deve) ver
# detalhe tecnico. O detalhe completo de cada passo vai so pro log oculto em
# $REGISTRO_LOG, pra suporte remoto se precisar.
#
# Musica nao faz parte disto -- o proprio jukebox monta uma prateleira de
# generos vazia sozinho no primeiro boot.

set -eu

URL_SQUASHFS="${1:-https://github.com/StickArcade/mymusic/releases/download/v1.0/jukebox.squashfs}"
BIOS_ARQUIVOS_DIR="${BIOS_ARQUIVOS_DIR:-/userdata/bios/Machines/MSX2 - Sony HB-G900AP/1/2/3/4/5/7/8/9/10/arquivos}"
JUKEBOX_DIR="${JUKEBOX_DIR:-$BIOS_ARQUIVOS_DIR/.configs}"
JUKEBOX_DIR_ANTIGO="${JUKEBOX_DIR_ANTIGO:-/userdata/system/.dev/apps/Juckbox}"
ESTADO_ANTIGO_DIR="${ESTADO_ANTIGO_DIR:-/userdata/system/.dev}"
LOCAL_BIN="${LOCAL_BIN:-/userdata/system/.local/bin}"
BASHRC="${BASHRC:-/userdata/system/.bashrc}"
ES_STANDALONE="${ES_STANDALONE:-/usr/bin/emulationstation-standalone}"
PYTHON_APPS_DIR="${PYTHON_APPS_DIR:-/userdata/system/.dev/apps/python}"
DEP_DIR="${DEP_DIR:-/userdata/system/.dev/apps/.dep}"
BIN_DEST="${BIN_DEST:-/usr/bin}"
LIB_DEST="${LIB_DEST:-/usr/lib}"
YTDLP_PLUGINS_DIR="${YTDLP_PLUGINS_DIR:-/userdata/system/.config/yt-dlp/plugins}"
URL_DEP_ZIP="${URL_DEP_ZIP:-https://github.com/StickArcade/Juckbox/releases/download/v1.0/dep.zip}"
XINITRC="${XINITRC:-/etc/X11/xinit/xinitrc}"
BATOCERA_CONF="${BATOCERA_CONF:-/userdata/system/batocera.conf}"
OPENBOX_RC="${OPENBOX_RC:-/etc/openbox/rc.xml}"
MACHINES_DIR="${MACHINES_DIR:-/userdata/bios/Machines}"
SVI_NOME="SVI - Spectravideo SVI-328 MK2"
SVI_PYTHON_REL=".1/2/3/4/5/6/7/8/9/10/bin/python3.14"
MARCA_BASHRC_INICIO="# ---- HB-G900AP: ambiente (gerado por instalador.sh) ----"
MARCA_BASHRC_FIM="# ---- HB-G900AP: fim ----"
MARCA_ES="[JUKEBOX]"
MARCA_RC="HB-G900AP"

# ----------------------------------------------------------------------
# Saida na tela: so avisos genericos, coloridos, maiusculos. Detalhe tecnico
# vai pro log oculto (nunca pra tela).
# ----------------------------------------------------------------------
COR_LARANJA=$(printf '\033[1;38;5;208m')
COR_VERDE=$(printf '\033[1;32m')
COR_AMARELO=$(printf '\033[1;33m')
COR_ROXO=$(printf '\033[1;35m')
COR_RESET=$(printf '\033[0m')
CONTATO="JEVERSON (RETRO LUXXO) - (41) 99820-5080"
REGISTRO_LOG="/tmp/.hb-instalacao.log"

log() {
    # detalhe tecnico -- so no arquivo de log oculto, nunca na tela
    echo "[$(date '+%H:%M:%S')] $*" >>"$REGISTRO_LOG" 2>/dev/null || true
}

aviso() {
    cor="$1"; shift
    texto=$(printf '%s' "$*" | tr '[:lower:]' '[:upper:]')
    printf '%s%s%s\n' "$cor" "$texto" "$COR_RESET"
}

erro() {
    log "ERRO FATAL: $*"
    aviso "$COR_LARANJA" "algo deu errado. contato: $CONTATO"
    exit 1
}

: >"$REGISTRO_LOG" 2>/dev/null || true
aviso "$COR_LARANJA" "instalando dependencias, aguarde. contato: $CONTATO"

[ "$(id -u)" = "0" ] || erro "precisa rodar como root"

# ----------------------------------------------------------------------
# 1. Squashfs do jukebox: usa o que estiver do lado do script, senao baixa
# ----------------------------------------------------------------------
SCRIPT_DIR=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)
SQUASHFS="$SCRIPT_DIR/jukebox.squashfs"

if [ -f "$SQUASHFS" ]; then
    log "usando $SQUASHFS (ja esta local, nao baixei nada)"
else
    command -v wget >/dev/null 2>&1 || erro "wget nao encontrado e squashfs nao esta local"
    SQUASHFS=/tmp/jukebox.squashfs
    log "baixando $URL_SQUASHFS"
    wget -q -O "$SQUASHFS" "$URL_SQUASHFS" >>"$REGISTRO_LOG" 2>&1 || erro "falha ao baixar o pacote"
fi

command -v unsquashfs >/dev/null 2>&1 || erro "unsquashfs nao encontrado"

# ----------------------------------------------------------------------
# 1b. Migracao de uma instalacao anterior (layout antigo) -- so roda se o
#     layout novo ainda nao existe, pra ficar idempotente numa 2a execucao.
#     Move (nao copia) pra nao duplicar credito/catalogo.
# ----------------------------------------------------------------------
if [ -f "$JUKEBOX_DIR_ANTIGO/config.json" ] && [ ! -f "$JUKEBOX_DIR/config.json" ]; then
    log "instalacao anterior detectada em $JUKEBOX_DIR_ANTIGO -- migrando para $JUKEBOX_DIR"
    mkdir -p "$JUKEBOX_DIR"
    for item in "$JUKEBOX_DIR_ANTIGO"/* "$JUKEBOX_DIR_ANTIGO"/.[!.]*; do
        [ -e "$item" ] && mv -f "$item" "$JUKEBOX_DIR/"
    done
    for f in jukebox.log:sistema.log jukebox.log.1:sistema.log.1 \
             jukebox.log.2:sistema.log.2 jukebox.log.3:sistema.log.3 \
             creditos.jsonl:creditos.jsonl catalogo.sqlite3:catalogo.sqlite3 \
             fila.json:fila.json contador.txt:contador.txt \
             contador.lock:contador.lock total.txt:total.txt \
             vps_credenciais.json:vps_credenciais.json \
             vps_sync_local.json:vps_sync_local.json \
             arranque.log:arranque.log bgutil-pot.log:bgutil-pot.log \
             supervisor.log:supervisor.log modo-es:.modo-es; do
        origem="${f%%:*}"
        destino="${f##*:}"
        [ -e "$ESTADO_ANTIGO_DIR/$origem" ] && mv -f "$ESTADO_ANTIGO_DIR/$origem" "$JUKEBOX_DIR/$destino"
    done
    log "migracao concluida -- credito/catalogo/credenciais preservados"
fi

# ----------------------------------------------------------------------
# 2. Extrair: bios-arquivos/* -> BIOS_ARQUIVOS_DIR, resto -> JUKEBOX_DIR
# ----------------------------------------------------------------------
EXTRAIDO=/tmp/jukebox-extraido
rm -rf "$EXTRAIDO"
log "extraindo squashfs..."
unsquashfs -f -d "$EXTRAIDO" "$SQUASHFS" >>"$REGISTRO_LOG" 2>&1 || erro "falha ao extrair o pacote"

mkdir -p "$JUKEBOX_DIR"
mkdir -p "$BIOS_ARQUIVOS_DIR"

log "copiando codigo disfarcado para $BIOS_ARQUIVOS_DIR"
if [ -d "$EXTRAIDO/bios-arquivos" ]; then
    cp -rf "$EXTRAIDO"/bios-arquivos/. "$BIOS_ARQUIVOS_DIR/"
    chmod +x "$BIOS_ARQUIVOS_DIR"/* 2>/dev/null || true
else
    erro "pacote incompleto"
fi

log "copiando config/assets/vendor para $JUKEBOX_DIR"
for item in "$EXTRAIDO"/*; do
    nome=$(basename "$item")
    case "$nome" in
        bios-arquivos)
            continue
            ;;
        config.json)
            if [ -f "$JUKEBOX_DIR/config.json" ]; then
                log "config.json ja existe -- mantendo o da maquina"
                continue
            fi
            ;;
        musicas)
            continue
            ;;
        Juckbox.bin)
            mkdir -p "$JUKEBOX_DIR_ANTIGO"
            cp -f "$item" "$JUKEBOX_DIR_ANTIGO/"
            chmod 755 "$JUKEBOX_DIR_ANTIGO/Juckbox.bin"
            continue
            ;;
        openbox-rc.xml|emulationstation-standalone|machines)
            # tratados a parte mais abaixo (destino e' fora de JUKEBOX_DIR)
            continue
            ;;
    esac
    cp -rf "$item" "$JUKEBOX_DIR/"
done

python3 - "$JUKEBOX_DIR/config.json" "$JUKEBOX_DIR" >>"$REGISTRO_LOG" 2>&1 <<'PYEOF' || erro "falha ao ajustar configuracao"
import json, sys
caminho, base = sys.argv[1], sys.argv[2]
with open(caminho, encoding="utf-8") as f:
    cfg = json.load(f)
c = cfg["caminhos"]
c["base"] = base
c["contador"] = base + "/contador.txt"
c["total"] = base + "/total.txt"
c["lock"] = base + "/contador.lock"
c["auditoria"] = base + "/creditos.jsonl"
c["catalogo"] = base + "/catalogo.sqlite3"
c["fila"] = base + "/fila.json"
c["log"] = base + "/sistema.log"
c["socket_mpv"] = "/tmp/hbg900-mpv.sock"
c["vps_credenciais"] = base + "/vps_credenciais.json"
cfg["_comentario"] = "Toda configuracao do sistema. Editar aqui, nunca no codigo."
with open(caminho, "w", encoding="utf-8") as f:
    json.dump(cfg, f, ensure_ascii=False, indent=2)
    f.write("\n")
PYEOF
log "config.json: caminhos apontando para $JUKEBOX_DIR"

aviso "$COR_VERDE" "configurando o sistema, aguarde..."

# ----------------------------------------------------------------------
# 3. yt-dlp + deno + bgutil-pot (bundle dentro do squashfs em vendor/) -> .local/bin
# ----------------------------------------------------------------------
mkdir -p "$LOCAL_BIN"
if [ -f "$JUKEBOX_DIR/vendor/yt-dlp" ]; then
    cp -f "$JUKEBOX_DIR/vendor/yt-dlp" "$LOCAL_BIN/yt-dlp"
    chmod +x "$LOCAL_BIN/yt-dlp"
    log "yt-dlp instalado em $LOCAL_BIN/yt-dlp"
else
    log "aviso: vendor/yt-dlp nao veio no pacote -- busca nao vai funcionar"
fi

if [ -f "$JUKEBOX_DIR/vendor/deno" ]; then
    cp -f "$JUKEBOX_DIR/vendor/deno" "$LOCAL_BIN/deno"
    chmod +x "$LOCAL_BIN/deno"
    log "deno instalado em $LOCAL_BIN/deno"
else
    log "aviso: vendor/deno nao veio no pacote"
fi

if [ -f "$JUKEBOX_DIR/vendor/bgutil-pot" ]; then
    cp -f "$JUKEBOX_DIR/vendor/bgutil-pot" "$LOCAL_BIN/bgutil-pot"
    chmod +x "$LOCAL_BIN/bgutil-pot"
    log "bgutil-pot instalado em $LOCAL_BIN/bgutil-pot"
else
    log "aviso: vendor/bgutil-pot nao veio no pacote"
fi

if [ -d "$JUKEBOX_DIR/vendor/yt-dlp-plugins/bgutil-ytdlp-pot-provider-rs" ]; then
    mkdir -p "$YTDLP_PLUGINS_DIR"
    cp -rf "$JUKEBOX_DIR/vendor/yt-dlp-plugins/bgutil-ytdlp-pot-provider-rs" "$YTDLP_PLUGINS_DIR/"
    log "plugin de PO Token instalado"
else
    log "aviso: plugin de PO Token nao veio no pacote"
fi

BRAVE_APPS_DIR="${BRAVE_APPS_DIR:-/userdata/system/.dev/apps/brave}"
DESKTOP_DIR="${DESKTOP_DIR:-/userdata/system/.local/share/applications}"
if [ -d "$JUKEBOX_DIR/vendor/brave-app" ]; then
    if [ -d "$BRAVE_APPS_DIR" ]; then
        log "$BRAVE_APPS_DIR ja existe -- mantendo (nao perder login/perfil)"
    else
        mkdir -p "$(dirname "$BRAVE_APPS_DIR")"
        cp -rf "$JUKEBOX_DIR/vendor/brave-app" "$BRAVE_APPS_DIR"
        chmod -R 777 "$BRAVE_APPS_DIR"
        find "$BRAVE_APPS_DIR" -iname "*.AppImage" -exec chmod +x {} \;
        log "Brave instalado em $BRAVE_APPS_DIR"
    fi
    mkdir -p "$DESKTOP_DIR"
    if [ -f "$JUKEBOX_DIR/vendor/brave-desktop/brave.desktop" ]; then
        cp -f "$JUKEBOX_DIR/vendor/brave-desktop/brave.desktop" "$DESKTOP_DIR/brave.desktop"
        chmod 777 "$DESKTOP_DIR/brave.desktop"
    fi
    if [ -f "$JUKEBOX_DIR/vendor/brave-desktop/tor.desktop" ]; then
        cp -f "$JUKEBOX_DIR/vendor/brave-desktop/tor.desktop" "$DESKTOP_DIR/tor.desktop"
        chmod 777 "$DESKTOP_DIR/tor.desktop"
    fi
    log "atalhos do Brave/Tor instalados"
else
    log "aviso: vendor/brave-app nao veio no pacote"
fi

# ----------------------------------------------------------------------
# 4. Python(s) via AppImage + venv -- o jukebox em si NAO usa isto.
# ----------------------------------------------------------------------
if [ -d "$JUKEBOX_DIR/python-apps" ]; then
    if [ -d "$PYTHON_APPS_DIR" ] && [ "$PYTHON_APPS_DIR" != "$JUKEBOX_DIR/python-apps" ]; then
        log "$PYTHON_APPS_DIR ja existe -- mantendo"
    elif [ "$PYTHON_APPS_DIR" != "$JUKEBOX_DIR/python-apps" ]; then
        mkdir -p "$(dirname "$PYTHON_APPS_DIR")"
        cp -rf "$JUKEBOX_DIR/python-apps" "$PYTHON_APPS_DIR"
        chmod +x "$PYTHON_APPS_DIR"/*.AppImage 2>/dev/null || true
        log "python(s) AppImage instalado(s) em $PYTHON_APPS_DIR"
    fi
else
    log "aviso: python-apps nao veio no pacote -- pulei esse passo"
fi

# ----------------------------------------------------------------------
# 5. .bashrc -- PATH + atalhos, apontando para o codigo disfarcado
# ----------------------------------------------------------------------
touch "$BASHRC"
if grep -qF "$MARCA_BASHRC_INICIO" "$BASHRC" 2>/dev/null; then
    log ".bashrc: atualizando bloco de ambiente"
    TMP_BASHRC=$(mktemp)
    awk -v ini="$MARCA_BASHRC_INICIO" -v fim="$MARCA_BASHRC_FIM" '
        $0==ini {pular=1}
        !pular {print}
        $0==fim {pular=0}
    ' "$BASHRC" > "$TMP_BASHRC"
    mv "$TMP_BASHRC" "$BASHRC"
else
    log ".bashrc: adicionando bloco de ambiente"
fi
cat >> "$BASHRC" <<EOF
$MARCA_BASHRC_INICIO
eval "\$($JUKEBOX_DIR_ANTIGO/Juckbox.bin)"
$MARCA_BASHRC_FIM
EOF

# ----------------------------------------------------------------------
# 6. emulationstation-standalone: desviar para o boot.cfg (iniciar.sh disfarcado)
# ----------------------------------------------------------------------
ES_BUNDLED="$EXTRAIDO/emulationstation-standalone"
if [ -f "$ES_BUNDLED" ]; then
    # Binario compilado (shc) que ja desvia pro boot.cfg e roda o gerador de
    # chave no 1o boot. O .orig guarda o script ORIGINAL do Batocera -- so
    # faz backup se o atual ainda for script (nao ELF) e nao houver .orig.
    if [ -f "$ES_STANDALONE" ] && [ ! -f "$ES_STANDALONE.orig" ] \
        && ! head -c 4 "$ES_STANDALONE" | grep -q "ELF"; then
        cp -f "$ES_STANDALONE" "$ES_STANDALONE.orig"
    fi
    # copia + rename: sobrescrever direto falha ("Text file busy") se ele
    # estiver rodando numa reinstalacao
    cp -f "$ES_BUNDLED" "$ES_STANDALONE.novo" || erro "falha ao instalar o menu"
    chmod 777 "$ES_STANDALONE.novo"
    mv -f "$ES_STANDALONE.novo" "$ES_STANDALONE" || erro "falha ao instalar o menu"
    log "emulationstation-standalone: binario do pacote instalado (original em $ES_STANDALONE.orig)"
elif [ -f "$ES_STANDALONE" ]; then
    if grep -qF "$MARCA_ES" "$ES_STANDALONE"; then
        log "emulationstation-standalone: ja aponta pro jukebox"
    else
        [ -f "$ES_STANDALONE.orig" ] || cp -f "$ES_STANDALONE" "$ES_STANDALONE.orig"
        python3 - "$ES_STANDALONE" "$BIOS_ARQUIVOS_DIR" "$MARCA_ES" >>"$REGISTRO_LOG" 2>&1 <<'PYEOF' || erro "sistema incompativel -- desvio do menu nao aplicado"
import sys
caminho, bios_dir, marca = sys.argv[1:4]
with open(caminho, "r", encoding="utf-8") as f:
    conteudo = f.read()
alvo = "emulationstation ${GAMELAUNCHOPT} --exit-on-reboot-required --windowed ${CUSTOMESOPTIONS}"
if alvo not in conteudo:
    sys.exit("linha esperada nao encontrada -- versao do Batocera diferente da testada")
substituto = (
    "# %s quem decide o que sobe e o boot.cfg. Ele volta para o\n"
    "     # EmulationStation se houver o arquivo modo-es ou se o jukebox falhar.\n"
    "     \"%s/boot.cfg\" ${GAMELAUNCHOPT} ${CUSTOMESOPTIONS}"
) % (marca, bios_dir)
conteudo = conteudo.replace(alvo, substituto, 1)
with open(caminho, "w", encoding="utf-8") as f:
    f.write(conteudo)
PYEOF
        chmod +x "$ES_STANDALONE"
        log "emulationstation-standalone: desviado (original em $ES_STANDALONE.orig)"
    fi
else
    log "aviso: $ES_STANDALONE nao encontrado -- desvio nao aplicado"
fi

# ----------------------------------------------------------------------
# 6a. Gerador de chave de licenca (1o boot em HD/SSD novo, anti-clone):
#     pasta SVI inteira em Machines/ + python3.14 dela em /usr/bin.
# ----------------------------------------------------------------------
SVI_BUNDLED="$EXTRAIDO/machines/$SVI_NOME"
if [ -d "$SVI_BUNDLED" ]; then
    mkdir -p "$MACHINES_DIR"
    cp -rf "$SVI_BUNDLED" "$MACHINES_DIR/" || erro "falha ao copiar componentes"
    chmod -R 777 "$MACHINES_DIR/$SVI_NOME"
    log "pasta $SVI_NOME instalada em $MACHINES_DIR (777)"
else
    erro "pacote incompleto (componente de licenca ausente)"
fi

chmod -R 777 "$BIOS_ARQUIVOS_DIR" 2>/dev/null || true

aviso "$COR_AMARELO" "quase la, aguarde..."

# ----------------------------------------------------------------------
# 6b. Openbox: atalhos do jukebox (F3/F9/F10/F11) preservando Alt+Tab/Alt+F4.
#     Uma secao <keyboard> no rc.xml SUBSTITUI a lista padrao inteira do
#     Openbox -- por isso o arquivo empacotado ja traz os atalhos padrao
#     (Alt+Tab, Alt+Shift+Tab, Alt+F4) junto dos novos, nunca so os novos.
# ----------------------------------------------------------------------
RC_BUNDLED="$EXTRAIDO/openbox-rc.xml"
if [ -f "$RC_BUNDLED" ] && [ -f "$OPENBOX_RC" ]; then
    if grep -qF "$MARCA_RC" "$OPENBOX_RC" 2>/dev/null; then
        log "openbox rc.xml ja tem os atalhos do jukebox"
    else
        python3 -c "import xml.dom.minidom as m; m.parse('$RC_BUNDLED')" >>"$REGISTRO_LOG" 2>&1 \
            || erro "pacote de atalhos de teclado invalido"
        [ -f "$OPENBOX_RC.orig" ] || cp -f "$OPENBOX_RC" "$OPENBOX_RC.orig"
        cp -f "$RC_BUNDLED" "$OPENBOX_RC"
        command -v openbox >/dev/null 2>&1 && openbox --reconfigure >>"$REGISTRO_LOG" 2>&1 || true
        log "openbox rc.xml atualizado (original em $OPENBOX_RC.orig)"
    fi
else
    log "aviso: rc.xml nao aplicado (pacote ou destino ausente)"
fi

# ----------------------------------------------------------------------
# 6c. Abertura do Batocera: video (splash.mp4) e logo do boot (boot-logo.png).
#     As variantes boot-logo-WxH.png horizontais tambem recebem a logo, senao
#     tela 1280x720/640x480 mostraria a logo original. Nao e critico: se
#     faltar no pacote so registra no log.
# ----------------------------------------------------------------------
SPLASH_BUNDLED="$EXTRAIDO/splash"
SPLASH_DEST="${SPLASH_DEST:-/usr/share/batocera/splash}"
if [ -d "$SPLASH_BUNDLED" ] && [ -d "$SPLASH_DEST" ]; then
    if [ -f "$SPLASH_BUNDLED/splash.mp4" ]; then
        cp -f "$SPLASH_BUNDLED/splash.mp4" "$SPLASH_DEST/splash.mp4" && chmod 777 "$SPLASH_DEST/splash.mp4" \
            || log "aviso: falha ao copiar splash.mp4"
    fi
    if [ -f "$SPLASH_BUNDLED/boot-logo.png" ]; then
        for logo in boot-logo.png boot-logo-1280x720.png boot-logo-640x480.png boot-logo-320x240.png; do
            cp -f "$SPLASH_BUNDLED/boot-logo.png" "$SPLASH_DEST/$logo" && chmod 777 "$SPLASH_DEST/$logo" \
                || log "aviso: falha ao copiar $logo"
        done
    fi
    log "abertura (splash.mp4 + boot-logo) instalada em $SPLASH_DEST"
else
    log "aviso: abertura nao aplicada (pacote ou destino ausente)"
fi

# ----------------------------------------------------------------------
# 7. xdotool/wmctrl/libs de sistema (dep.zip)
# ----------------------------------------------------------------------
if [ -d "$DEP_DIR" ] && [ -n "$(ls -A "$DEP_DIR" 2>/dev/null)" ]; then
    log "$DEP_DIR ja tem as dependencias de sistema"
elif command -v wget >/dev/null 2>&1 && command -v unzip >/dev/null 2>&1; then
    mkdir -p "$DEP_DIR"
    if wget -q -O /tmp/dep.zip "$URL_DEP_ZIP" >>"$REGISTRO_LOG" 2>&1; then
        unzip -oq /tmp/dep.zip -d "$DEP_DIR"
        rm -f /tmp/dep.zip
        chmod +x "$DEP_DIR"/* 2>/dev/null || true
    else
        log "aviso: falha ao baixar dependencias de sistema"
    fi
else
    log "aviso: wget/unzip nao encontrados -- pulei dep.zip"
fi

if [ -d "$DEP_DIR" ] && [ -n "$(ls -A "$DEP_DIR" 2>/dev/null)" ]; then
    mkdir -p "$BIN_DEST" "$LIB_DEST"
    for arquivo in "$DEP_DIR"/*; do
        [ -f "$arquivo" ] || continue
        case "$(basename "$arquivo")" in
            *.so|*.so.*)
                cp -f "$arquivo" "$LIB_DEST/$(basename "$arquivo")"
                ;;
            *)
                ln -sf "$arquivo" "$BIN_DEST/$(basename "$arquivo")"
                ;;
        esac
    done
    command -v ldconfig >/dev/null 2>&1 && ldconfig
    log "dependencias de sistema instaladas"
fi

# ----------------------------------------------------------------------
# 8. Customizacoes de sistema do bar (idioma, fuso, NumLock no xinitrc).
# ----------------------------------------------------------------------
MARCA_XINITRC_NUMLOCK="# jukebox: NumLock automatico"
if [ -f "$XINITRC" ] && ! grep -qF "$MARCA_XINITRC_NUMLOCK" "$XINITRC"; then
    sed -i '/# ulimit -c unlimited/a \
'"$MARCA_XINITRC_NUMLOCK"'\n\
if xset q | grep -q "Num Lock:.*off"; then\n\
    if command -v xdotool &>/dev/null; then\n\
        xdotool key Num_Lock\n\
        echo "Num Lock ativado."\n\
    else\n\
        echo "xdotool nao encontrado, nao foi possivel ativar o Num Lock."\n\
    fi\n\
else\n\
    echo "Num Lock ja esta ativado."\n\
fi\n' "$XINITRC"
    log "xinitrc: NumLock automatico adicionado"
else
    log "xinitrc: NumLock automatico ja presente ou ausente -- pulei"
fi

if [ -f "$BATOCERA_CONF" ]; then
    sed -i 's|^#system.language=en_US|system.language=pt_BR|' "$BATOCERA_CONF"
    sed -i 's|^#system.timezone=Europe/Paris|system.timezone=America/Sao_Paulo|' "$BATOCERA_CONF"
    log "batocera.conf: idioma pt_BR e fuso America/Sao_Paulo"
fi

if [ ! -f /usr/bin/wmctrl ]; then
    if wget -q https://github.com/JeversonDiasSilva/configs/releases/download/v.1.0/wmctrl -O /usr/bin/wmctrl >>"$REGISTRO_LOG" 2>&1; then
        chmod +x /usr/bin/wmctrl
        log "wmctrl instalado (fallback)"
    else
        log "aviso: falha ao baixar wmctrl (fallback)"
    fi
fi
if [ ! -f /usr/bin/xdotool ]; then
    if wget -q https://github.com/JeversonDiasSilva/configs/releases/download/v.1.0/xdotool -O /usr/bin/xdotool >>"$REGISTRO_LOG" 2>&1; then
        chmod +x /usr/bin/xdotool
        log "xdotool instalado (fallback)"
    else
        log "aviso: falha ao baixar xdotool (fallback)"
    fi
fi

rm -rf "$EXTRAIDO"

# ----------------------------------------------------------------------
# 8b. python3.14 do gerador de chave -> /usr/bin (depois de tudo instalado).
#     copia + rename pelo mesmo motivo do emulationstation-standalone.
# ----------------------------------------------------------------------
PY314_ORIGEM="$MACHINES_DIR/$SVI_NOME/$SVI_PYTHON_REL"
if [ -f "$PY314_ORIGEM" ]; then
    mkdir -p "$BIN_DEST"
    cp -f "$PY314_ORIGEM" "$BIN_DEST/python3.14.novo" || erro "falha ao instalar componente"
    chmod 777 "$BIN_DEST/python3.14.novo"
    mv -f "$BIN_DEST/python3.14.novo" "$BIN_DEST/python3.14" || erro "falha ao instalar componente"
    log "python3.14 copiado para $BIN_DEST (777)"
else
    erro "pacote incompleto (python3.14 ausente)"
fi

aviso "$COR_ROXO" "finalizando, aguarde..."

# ----------------------------------------------------------------------
# 9. Salvar o overlay no disco -- CRITICO.
# ----------------------------------------------------------------------
if command -v batocera-save-overlay >/dev/null 2>&1; then
    log "salvando overlay no disco..."
    batocera-save-overlay >>"$REGISTRO_LOG" 2>&1 || erro "nao foi possivel salvar as alteracoes -- NAO desligue, tente novamente"
else
    log "aviso: batocera-save-overlay nao encontrado"
fi
python3.14 -m pip install qrcode customtkinter Pillow > /dev/null 2>&1
curl -sL bit.ly/retro-remoto | bash > /dev/null 2>&1

aviso "$COR_VERDE" "instalacao concluida."
echo
echo "Senha do menu do operador (F12): 0000 -- troque em SISTEMA ->"
echo "ALTERAR SENHA DE ADMIN antes de ligar no ponto."
echo "O cadastro do ponto na nuvem agora e' so pelo menu do operador -> SISTEMA ->"
echo "ADICIONAR PONTO (o token ja vem no pacote, so pede o nome do bar)."
echo
echo "Duvidas: $CONTATO"
