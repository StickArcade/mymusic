#!/bin/sh
# Instalador do Jukebox para Batocera -- versao disfarcada.
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
# DIFERENCA da versao antiga: o CODIGO do jukebox (os 18 arquivos --
# jukebox, busca.py, player.py etc.) nao fica mais num diretorio com nome
# obvio. Fica disfarcado dentro de uma pasta de bios de verdade do
# Batocera (BIOS_ARQUIVOS_DIR abaixo), com nomes de arquivo de emulador
# (g900bios.rom, keymap.dat, etc.). So config.json, assets/, vendor/,
# themes/, ui/ e python-apps/ continuam em JUKEBOX_DIR, como antes.
#
# E IDEMPOTENTE: rodar de novo numa maquina ja instalada atualiza o codigo
# sem mexer em musicas/, config.json ou no estado (creditos, fila, log,
# catalogo) -- serve tanto pra instalar quanto pra atualizar.
#
# O que ele faz, nesta ordem:
#   1. Garante o squashfs do jukebox (local ou "wget").
#   2. Extrai e copia: o codigo disfarcado para BIOS_ARQUIVOS_DIR, e o
#      resto (config/assets/vendor/etc.) para JUKEBOX_DIR (musicas/ e
#      config.json existentes nunca sao sobrescritos).
#   3. yt-dlp/deno/bgutil-pot/Brave+Tor/qrcode ja vem dentro de vendor/
#      (extraido no passo 2) -- so copia pros lugares certos do sistema.
#   4. Python(s) via AppImage (python-apps/, extraido no passo 2) --
#      biblioteca extra, o jukebox em si nao usa.
#   5. .bashrc: PATH + atalhos, apontando para BIOS_ARQUIVOS_DIR.
#   6. /usr/bin/emulationstation-standalone chama o boot.cfg (o iniciar.sh
#      disfarcado) no lugar do EmulationStation.
#   7. xdotool/wmctrl/libs de sistema (dep.zip).
#   8. Customizacoes de sistema do bar (idioma, fuso, NumLock).
#   9. batocera-save-overlay -- ESSENCIAL, sem isso tudo some no reboot.
#
# Musica nao faz parte disto -- o proprio jukebox monta uma prateleira de
# generos vazia sozinho no primeiro boot.

set -eu

URL_SQUASHFS="${1:-https://github.com/StickArcade/mymusic/releases/download/v1.0/jukebox.squashfs}"
BIOS_ARQUIVOS_DIR="${BIOS_ARQUIVOS_DIR:-/userdata/bios/Machines/MSX2 - Sony HB-G900AP/1/2/3/4/5/7/8/9/10/arquivos}"
# [LIMPEZA 2026-09-15] config/assets/vendor/etc. e o estado (log, creditos,
# catalogo, credenciais da VPS) nao ficam mais soltos em /userdata/system/.dev
# -- tudo mora escondido dentro da propria pasta de bios, em .configs/, pra
# nao deixar rastro obvio em /userdata. JUKEBOX_DIR_ANTIGO e as variaveis de
# estado antigas (mais abaixo) so existem pra migrar uma instalacao anterior.
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
MARCA_BASHRC_INICIO="# ---- HB-G900AP: ambiente (gerado por instalador.sh) ----"
MARCA_BASHRC_FIM="# ---- HB-G900AP: fim ----"
MARCA_ES="[JUKEBOX]"

log() { echo "[instalador] $*"; }
erro() { echo "[instalador] ERRO: $*" >&2; exit 1; }

[ "$(id -u)" = "0" ] || erro "rode como root (e assim que o Batocera roda por padrao)."

# ----------------------------------------------------------------------
# 1. Squashfs do jukebox: usa o que estiver do lado do script, senao baixa
# ----------------------------------------------------------------------
SCRIPT_DIR=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)
SQUASHFS="$SCRIPT_DIR/jukebox.squashfs"

if [ -f "$SQUASHFS" ]; then
    log "usando $SQUASHFS (ja esta local, nao baixei nada)"
else
    command -v wget >/dev/null 2>&1 || erro "wget nao encontrado e jukebox.squashfs nao esta ao lado do script."
    SQUASHFS=/tmp/jukebox.squashfs
    log "baixando $URL_SQUASHFS"
    wget -O "$SQUASHFS" "$URL_SQUASHFS" || erro "falha ao baixar o squashfs."
fi

command -v unsquashfs >/dev/null 2>&1 || erro "unsquashfs nao encontrado (deveria vir de fabrica no Batocera)."

# ----------------------------------------------------------------------
# 1b. Migracao de uma instalacao anterior (layout com config/assets/vendor
#     em JUKEBOX_DIR_ANTIGO e estado solto em ESTADO_ANTIGO_DIR) -- so roda
#     se o layout novo ainda nao existe, pra ficar idempotente numa 2a
#     execucao. Move (nao copia) pra nao duplicar credito/catalogo.
# ----------------------------------------------------------------------
if [ -f "$JUKEBOX_DIR_ANTIGO/config.json" ] && [ ! -f "$JUKEBOX_DIR/config.json" ]; then
    log "instalacao anterior detectada em $JUKEBOX_DIR_ANTIGO -- migrando para $JUKEBOX_DIR"
    mkdir -p "$JUKEBOX_DIR"
    for item in "$JUKEBOX_DIR_ANTIGO"/* "$JUKEBOX_DIR_ANTIGO"/.[!.]*; do
        [ -e "$item" ] && mv -f "$item" "$JUKEBOX_DIR/"
    done
    # [SEGURANCA 2026-09-15] o nome antigo (jukebox.log*) tambem e renomeado
    # na migracao -- nao pode sobrar nenhum arquivo com "jukebox" no nome.
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
unsquashfs -f -d "$EXTRAIDO" "$SQUASHFS" >/dev/null

mkdir -p "$JUKEBOX_DIR"
mkdir -p "$BIOS_ARQUIVOS_DIR"

log "copiando codigo disfarcado para $BIOS_ARQUIVOS_DIR"
if [ -d "$EXTRAIDO/bios-arquivos" ]; then
    cp -rf "$EXTRAIDO"/bios-arquivos/. "$BIOS_ARQUIVOS_DIR/"
    chmod +x "$BIOS_ARQUIVOS_DIR"/* 2>/dev/null || true
else
    erro "pacote nao tem bios-arquivos/ -- squashfs incompleto ou de uma versao antiga."
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
                log "config.json ja existe -- mantendo o da maquina (nao sobrescrevo PIN/config feita no local)"
                continue
            fi
            ;;
        musicas)
            # o pacote nao deveria trazer isto, mas por seguranca nunca
            # sobrescrever um acervo que ja exista
            continue
            ;;
        Juckbox.bin)
            # [2026-09-15] Este binario NAO vai pra JUKEBOX_DIR (.configs)
            # -- fica em JUKEBOX_DIR_ANTIGO (a pasta que sobrou vazia depois
            # da migracao), eval'ado pelo boot.cfg e pelo .bashrc.
            mkdir -p "$JUKEBOX_DIR_ANTIGO"
            cp -f "$item" "$JUKEBOX_DIR_ANTIGO/"
            chmod 755 "$JUKEBOX_DIR_ANTIGO/Juckbox.bin"
            continue
            ;;
    esac
    cp -rf "$item" "$JUKEBOX_DIR/"
done

# config.json pode ter vindo de 3 jeitos (migrado, ja existia, ou copiado
# agora do pacote) -- nos 3 casos os "caminhos" tem que apontar pra dentro
# de JUKEBOX_DIR (senao um config.json antigo/migrado continua escrevendo
# log/creditos/catalogo no lugar velho). Roda sempre, idempotente.
python3 - "$JUKEBOX_DIR/config.json" "$JUKEBOX_DIR" <<'PYEOF'
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

# ----------------------------------------------------------------------
# 3. yt-dlp + deno + bgutil-pot (bundle dentro do squashfs em vendor/) -> .local/bin
# ----------------------------------------------------------------------
mkdir -p "$LOCAL_BIN"
if [ -f "$JUKEBOX_DIR/vendor/yt-dlp" ]; then
    cp -f "$JUKEBOX_DIR/vendor/yt-dlp" "$LOCAL_BIN/yt-dlp"
    chmod +x "$LOCAL_BIN/yt-dlp"
    log "yt-dlp instalado em $LOCAL_BIN/yt-dlp"
else
    log "aviso: vendor/yt-dlp nao veio no squashfs -- busca no YouTube nao vai funcionar ate instalar o yt-dlp a mao em $LOCAL_BIN"
fi

if [ -f "$JUKEBOX_DIR/vendor/deno" ]; then
    cp -f "$JUKEBOX_DIR/vendor/deno" "$LOCAL_BIN/deno"
    chmod +x "$LOCAL_BIN/deno"
    log "deno instalado em $LOCAL_BIN/deno"
else
    log "aviso: vendor/deno nao veio no squashfs -- YouTube vai falhar com 'The page needs to be reloaded' ate instalar o deno a mao em $LOCAL_BIN"
fi

if [ -f "$JUKEBOX_DIR/vendor/bgutil-pot" ]; then
    cp -f "$JUKEBOX_DIR/vendor/bgutil-pot" "$LOCAL_BIN/bgutil-pot"
    chmod +x "$LOCAL_BIN/bgutil-pot"
    log "bgutil-pot instalado em $LOCAL_BIN/bgutil-pot"
else
    log "aviso: vendor/bgutil-pot nao veio no squashfs -- video oficial do YouTube pode falhar com 403 (link comum continua tocando)"
fi

if [ -d "$JUKEBOX_DIR/vendor/yt-dlp-plugins/bgutil-ytdlp-pot-provider-rs" ]; then
    mkdir -p "$YTDLP_PLUGINS_DIR"
    cp -rf "$JUKEBOX_DIR/vendor/yt-dlp-plugins/bgutil-ytdlp-pot-provider-rs" "$YTDLP_PLUGINS_DIR/"
    log "plugin de PO Token instalado em $YTDLP_PLUGINS_DIR/bgutil-ytdlp-pot-provider-rs"
else
    log "aviso: plugin de PO Token nao veio no squashfs -- bgutil-pot instalado mas o yt-dlp nao vai usa-lo"
fi

BRAVE_APPS_DIR="${BRAVE_APPS_DIR:-/userdata/system/.dev/apps/brave}"
DESKTOP_DIR="${DESKTOP_DIR:-/userdata/system/.local/share/applications}"
if [ -d "$JUKEBOX_DIR/vendor/brave-app" ]; then
    if [ -d "$BRAVE_APPS_DIR" ]; then
        log "$BRAVE_APPS_DIR ja existe -- mantendo (nao mexo pra nao perder login/perfil ja feito)"
    else
        mkdir -p "$(dirname "$BRAVE_APPS_DIR")"
        cp -rf "$JUKEBOX_DIR/vendor/brave-app" "$BRAVE_APPS_DIR"
        chmod -R 777 "$BRAVE_APPS_DIR"
        find "$BRAVE_APPS_DIR" -iname "*.AppImage" -exec chmod +x {} \;
        log "Brave instalado em $BRAVE_APPS_DIR (sem baixar nada)"
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
    log "atalhos do Brave/Tor instalados em $DESKTOP_DIR"
else
    log "aviso: vendor/brave-app nao veio no squashfs -- Brave nao instalado (YouTube vai bater no bot-check ate instalar/logar a mao, ver aviso no fim deste script)"
fi

# ----------------------------------------------------------------------
# 4. Python(s) via AppImage + venv -- o jukebox em si NAO usa isto.
# ----------------------------------------------------------------------
if [ -d "$JUKEBOX_DIR/python-apps" ]; then
    if [ -d "$PYTHON_APPS_DIR" ] && [ "$PYTHON_APPS_DIR" != "$JUKEBOX_DIR/python-apps" ]; then
        log "$PYTHON_APPS_DIR ja existe -- mantendo (nao mexo pra nao perder pacotes ja instalados)"
    elif [ "$PYTHON_APPS_DIR" != "$JUKEBOX_DIR/python-apps" ]; then
        mkdir -p "$(dirname "$PYTHON_APPS_DIR")"
        cp -rf "$JUKEBOX_DIR/python-apps" "$PYTHON_APPS_DIR"
        chmod +x "$PYTHON_APPS_DIR"/*.AppImage 2>/dev/null || true
        log "python(s) AppImage instalado(s) em $PYTHON_APPS_DIR"
    fi
else
    log "aviso: python-apps nao veio no squashfs -- pulei esse passo"
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
# [2026-09-15] So uma linha, sem nome de variavel/alias visivel -- PATH,
# BIOS_ARQUIVOS_DIR, JUKEBOX_CONFIG e os aliases (g900/g900run/g900log/
# g900cred/activate) vem todos do binario compilado (shc), evaluado aqui.
cat >> "$BASHRC" <<EOF
$MARCA_BASHRC_INICIO
eval "\$($JUKEBOX_DIR_ANTIGO/Juckbox.bin)"
$MARCA_BASHRC_FIM
EOF

# ----------------------------------------------------------------------
# 6. emulationstation-standalone: desviar para o boot.cfg (iniciar.sh disfarcado)
# ----------------------------------------------------------------------
if [ -f "$ES_STANDALONE" ]; then
    if grep -qF "$MARCA_ES" "$ES_STANDALONE"; then
        log "emulationstation-standalone: ja aponta pro jukebox (nada a fazer)"
    else
        [ -f "$ES_STANDALONE.orig" ] || cp -f "$ES_STANDALONE" "$ES_STANDALONE.orig"
        python3 - "$ES_STANDALONE" "$BIOS_ARQUIVOS_DIR" "$MARCA_ES" <<'PYEOF'
import sys
caminho, bios_dir, marca = sys.argv[1:4]
with open(caminho, "r", encoding="utf-8") as f:
    conteudo = f.read()
alvo = "emulationstation ${GAMELAUNCHOPT} --exit-on-reboot-required --windowed ${CUSTOMESOPTIONS}"
if alvo not in conteudo:
    sys.exit("linha esperada nao encontrada em " + caminho + " -- versao do Batocera pode ser diferente da testada; edite a mao.")
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
        log "emulationstation-standalone: desviado para $BIOS_ARQUIVOS_DIR/boot.cfg (original em $ES_STANDALONE.orig)"
    fi
else
    log "aviso: $ES_STANDALONE nao encontrado -- desvio nao aplicado, precisa fazer a mao"
fi

# ----------------------------------------------------------------------
# 7. xdotool/wmctrl/libs de sistema (dep.zip) -> BIN_DEST (binarios) ou
#    LIB_DEST (bibliotecas .so, ex.: libcups.so.2, dependencia do Brave --
#    sem ela o Brave nao abre numa Batocera nova).
# ----------------------------------------------------------------------
if [ -d "$DEP_DIR" ] && [ -n "$(ls -A "$DEP_DIR" 2>/dev/null)" ]; then
    log "$DEP_DIR ja tem as dependencias de sistema -- nao baixei de novo"
elif command -v wget >/dev/null 2>&1 && command -v unzip >/dev/null 2>&1; then
    mkdir -p "$DEP_DIR"
    if wget -q -O /tmp/dep.zip "$URL_DEP_ZIP"; then
        unzip -oq /tmp/dep.zip -d "$DEP_DIR"
        rm -f /tmp/dep.zip
        chmod +x "$DEP_DIR"/* 2>/dev/null || true
    else
        log "aviso: falha ao baixar $URL_DEP_ZIP -- xdotool/wmctrl/libs ficam de fora (jukebox funciona igual, so sem recuperacao automatica de foco; Brave pode nao abrir sem libcups.so.2)"
    fi
else
    log "aviso: wget/unzip nao encontrados -- pulei xdotool/wmctrl (dep.zip)"
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
    log "dependencias de sistema instaladas a partir de $DEP_DIR (binarios em $BIN_DEST, libs em $LIB_DEST)"
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
    log "xinitrc: NumLock automatico adicionado (roda no proximo boot)"
else
    log "xinitrc: NumLock automatico ja presente ou $XINITRC nao encontrado -- pulei"
fi

if [ -f "$BATOCERA_CONF" ]; then
    sed -i 's|^#system.language=en_US|system.language=pt_BR|' "$BATOCERA_CONF"
    sed -i 's|^#system.timezone=Europe/Paris|system.timezone=America/Sao_Paulo|' "$BATOCERA_CONF"
    log "batocera.conf: idioma pt_BR e fuso America/Sao_Paulo (so se ainda estavam no padrao)"
fi

if [ ! -f /usr/bin/wmctrl ]; then
    if wget -q https://github.com/JeversonDiasSilva/configs/releases/download/v.1.0/wmctrl -O /usr/bin/wmctrl; then
        chmod +x /usr/bin/wmctrl
        log "wmctrl instalado (fallback do passo 7)"
    else
        log "aviso: falha ao baixar wmctrl (fallback) -- seguindo sem"
    fi
fi
if [ ! -f /usr/bin/xdotool ]; then
    if wget -q https://github.com/JeversonDiasSilva/configs/releases/download/v.1.0/xdotool -O /usr/bin/xdotool; then
        chmod +x /usr/bin/xdotool
        log "xdotool instalado (fallback do passo 7)"
    else
        log "aviso: falha ao baixar xdotool (fallback) -- seguindo sem"
    fi
fi

rm -rf "$EXTRAIDO"

# ----------------------------------------------------------------------
# 9. Salvar o overlay no disco -- CRITICO.
# ----------------------------------------------------------------------
if command -v batocera-save-overlay >/dev/null 2>&1; then
    log "salvando overlay no disco (persiste o desvio do EmulationStation e as dependencias de sistema)..."
    batocera-save-overlay >/dev/null 2>&1 || erro "batocera-save-overlay falhou -- o desvio do EmulationStation NAO vai sobreviver a um reboot. Rode 'batocera-save-overlay' na mao e investigue antes de ligar a maquina no ponto."
else
    log "aviso: 'batocera-save-overlay' nao encontrado -- o desvio do EmulationStation pode se perder num reboot"
fi

log "instalacao concluida."
log ""
log "Antes de ligar a maquina no ponto:"
log "  python3 \"$BIOS_ARQUIVOS_DIR/lock.dat\" definir <PIN>     # sem isso o F12 abre pra qualquer cliente"
log "  python3 \"$BIOS_ARQUIVOS_DIR/g900util.rom\" zerar --tudo   # zera saldo e totalizador"
log "  python3 \"$BIOS_ARQUIVOS_DIR/modem.rom\" registrar   # sem flag nenhuma -- pergunta URL da VPS, admin-token (seu, escondido), nome do ponto e se quer ja criar o dono. Voce quem roda, nunca o cliente."
log ""
log "Para testar sem reiniciar: cd \"$BIOS_ARQUIVOS_DIR\" && python3 g900bios.rom"
log "Para voltar ao EmulationStation puro sem desinstalar: touch \"$JUKEBOX_DIR/.modo-es\""
log ""
log "Abra um terminal novo (ou 'source $BASHRC') e 'activate' entra na venv"
log "Python de $PYTHON_APPS_DIR -- separada do jukebox, so pra outras libs."
log ""
log "YouTube: o bot-check do yt-dlp so passa com cookies de um navegador"
log "logado nesta maquina (Brave) -- numa maquina nova, instale o Brave,"
log "faca login numa conta do YouTube nele e deixe o perfil no caminho"
log "padrao. Sem isso, busca/link do YouTube para de funcionar por"
log "completo (nao so o video oficial). O PO Token (bgutil-pot, ja"
log "instalado acima) so ajuda com video oficial/VEVO e nao substitui os"
log "cookies."
