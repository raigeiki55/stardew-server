FROM jlesage/baseimage-gui:debian-11-v4

ENV APP_NAME="Stardew Dedicated Server" \
    USER_ID=0 \
    GROUP_ID=0 \
    HOME=/root \
    LANG=en_US.UTF-8 \
    LANGUAGE=en_US:en \
    LC_ALL=en_US.UTF-8 \
    ACCEPT_EULA=Y \
    DEBIAN_FRONTEND=noninteractive \
    TERMINAL=xterm \
    DARK_MODE=1 \
    DISPLAY=:0 \
    DISPLAY_HEIGHT=800 \
    DISPLAY_WIDTH=1280 \
    DISABLE_RENDERING=true\
    SMAPI_USE_CURRENT_SHELL=true \
    SMAPI_VERSION=4.1.10 \
    SMAPI_MODS_PATH=/data/Mods

# Add 32-bit architecture and update the sources list for SteamCMD
RUN printf "\n deb http://deb.debian.org/debian/ bullseye contrib non-free" >> /etc/apt/sources.list && \
    dpkg --add-architecture i386 && \
    apt-get update

# Install base packages
RUN apt-get update && apt-get install --no-install-recommends -y \
    ca-certificates \
    curl \
    locales \
    procps \
    unzip \
    x11-utils && \
    echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && \
    locale-gen

# Install SteamCMD
RUN echo steam steam/question select "I AGREE" | debconf-set-selections && \
    echo steam steam/license note '' | debconf-set-selections && \
    apt-get install -y steamcmd && \
    mv /usr/games/steamcmd /opt/base/bin/ && \
    steamcmd +force_install_dir /data/steam +@sSteamCmdForcePlatformType linux +login anonymous +app_update 1007 +quit && \
    mkdir -p /root/.steam/sdk64 && \
    mv /data/steam/linux64/steamclient.so /root/.steam/sdk64/steamclient.so && \
    rm -rf /data/steam

# Install GUI packages
RUN apt-get install -y \
    polybar \
    rofi \
    slop \
    wmctrl \
    xwallpaper

# Install additional packages
RUN apt-get install -y \
    exa \
    htop \
    nano

# Install TigerVNC (replaces pre-compiled x11 to include GLX extension)
RUN apt-get install -y tigervnc-standalone-server && \
    cp /usr/bin/Xvnc /opt/base/bin/Xvnc

# Install polybar theme
RUN add-pkg --virtual build git && \
    git clone --depth=1 https://github.com/adi1090x/polybar-themes.git && \
    cd ./polybar-themes && \
    chmod +x ./setup.sh && \
    echo "1" | ./setup.sh && \
    cd .. && \
    rm -rf ./polybar-themes && \
    del-pkg build

# Install WebVNC icon
COPY docker/rootfs/data/images/junimo.png /data/images/junimo.png
RUN install_app_icon.sh /data/images/junimo.png && \
    rm /data/images/junimo.png

RUN apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* && \
    rm /var/lib/dpkg/statoverride && \
    rm /etc/cont-init.d/89-info.sh && \
    rm -rf /etc/cont-env.d/LIBGL_DRIVERS_PATH

# Copy mods
COPY .output/mods /data/Mods

# Copy remaining files
COPY docker/rootfs/ /

# Embed the patched xvnc parameters file by writing it line-by-line.
RUN /bin/sh -c "printf '#!/bin/sh\n\
set -u # Treat unset variables as an error.\n\
\n\
# Activate debug logging.\n\
if is-bool-val-true \"\${CONTAINER_DEBUG:-0}\"; then\n\
    echo \"-Log\"\n\
    echo \"*:stderr:100\"\n\
    echo \"-verbose\"\n\
    echo \"10\"\n\
fi\n\
\n\
# Make sure the X server listens on Unix only.\n\
echo \"-nolisten\"\n\
echo \"tcp\"\n\
echo \"-nolisten\"\n\
echo \"local\"\n\
echo \"-listen\"\n\
echo \"unix\"\n\
\n\
# Window size.\n\
echo \"-geometry\"\n\
echo \"\${DISPLAY_WIDTH}x\${DISPLAY_HEIGHT}\"\n\
\n\
# Color depth.\n\
echo \"-depth\"\n\
echo \"24\"\n\
\n\
# VNC listening port.\n\
if [ \"\${VNC_LISTENING_PORT:-5900}\" -eq -1 ]; then\n\
    echo \"-rfbport=-1\"\n\
elif is-bool-val-true \"\${SECURE_CONNECTION:-0}\" && [ \"\${SECURE_CONNECTION_VNC_METHOD:-SSL}\" = \"SSL\" ]; then\n\
    echo \"-rfbport=-1\"\n\
else\n\
    echo \"-rfbport=\${VNC_LISTENING_PORT:-5900}\"\n\
    if ifconfig -a | grep -wq inet6; then\n\
        echo \"-UseIPv6=yes\"\n\
    else\n\
        echo \"-UseIPv6=no\"\n\
    fi\n\
fi\n\
\n\
# VNC Unix socket.\n\
echo \"-rfbunixpath=/tmp/vnc.sock\"\n\
echo \"-rfbunixmode=0660\"\n\
\n\
# VNC security.\n\
PASSWORD_FILE=\n\
if [ -f /config/.vncpass ] && [ -n \"$(cat /config/.vncpass)\" ]; then\n\
    PASSWORD_FILE=/config/.vncpass\n\
elif [ -f /tmp/.vncpass ] && [ -n \"$(cat /tmp/.vncpass)\" ]; then\n\
    PASSWORD_FILE=/tmp/.vncpass\n\
fi\n\
\n\
if [ -n \"\$PASSWORD_FILE\" ]; then\n\
    if is-bool-val-false \"\${SECURE_CONNECTION:-0}\" || [ \"\${SECURE_CONNECTION_VNC_METHOD:-SSL}\" = \"SSL\" ]; then\n\
        echo \"-SecurityTypes=VncAuth\"\n\
    else\n\
        echo \"-SecurityTypes=X509Vnc,TLSVnc\"\n\
    fi\n\
    echo \"-rfbauth=\$PASSWORD_FILE\"\n\
else\n\
    if is-bool-val-false \"\${SECURE_CONNECTION:-0}\" || [ \"\${SECURE_CONNECTION_VNC_METHOD:-SSL}\" = \"SSL\" ]; then\n\
        echo \"-SecurityTypes=None\"\n\
    else\n\
        echo \"-SecurityTypes=X509None,TLSNone\"\n\
    fi\n\
    echo \"-InternalConnectionSecurityTypes=None\"\n\
fi\n\
\n\
if is-bool-val-true \"\${SECURE_CONNECTION:-0}\" && [ \"\${SECURE_CONNECTION_VNC_METHOD:-SSL}\" != \"SSL\" ]; then\n\
    echo \"-X509Key\"\n\
    echo \"/config/certs/vnc-privkey.pem\"\n\
    echo \"-X509Cert\"\n\
    echo \"/config/certs/vnc-fullchain.pem\"\n\
fi\n\
\n\
echo \"-desktop=\$APP_NAME\"\n\
echo \"\$DISPLAY\"\n" > /etc/services.d/xvnc/params && chmod +x /etc/services.d/xvnc/params

WORKDIR /data
