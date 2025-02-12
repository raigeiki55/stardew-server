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
RUN /bin/sh -c "\
echo '#!/bin/sh' > /etc/services.d/xvnc/params && \
echo 'set -u # Treat unset variables as an error.' >> /etc/services.d/xvnc/params && \
echo '' >> /etc/services.d/xvnc/params && \
echo '# Activate debug logging.' >> /etc/services.d/xvnc/params && \
echo 'if is-bool-val-true \"${CONTAINER_DEBUG:-0}\"; then' >> /etc/services.d/xvnc/params && \
echo '    echo \"-Log\"' >> /etc/services.d/xvnc/params && \
echo '    echo \"*:stderr:100\"' >> /etc/services.d/xvnc/params && \
echo '    echo \"-verbose\"' >> /etc/services.d/xvnc/params && \
echo '    echo \"10\"' >> /etc/services.d/xvnc/params && \
echo 'fi' >> /etc/services.d/xvnc/params && \
echo '' >> /etc/services.d/xvnc/params && \
echo '# Make sure the X server listens on Unix only.' >> /etc/services.d/xvnc/params && \
echo 'echo \"-nolisten\"' >> /etc/services.d/xvnc/params && \
echo 'echo \"tcp\"' >> /etc/services.d/xvnc/params && \
echo 'echo \"-nolisten\"' >> /etc/services.d/xvnc/params && \
echo 'echo \"local\"' >> /etc/services.d/xvnc/params && \
echo 'echo \"-listen\"' >> /etc/services.d/xvnc/params && \
echo 'echo \"unix\"' >> /etc/services.d/xvnc/params && \
echo '' >> /etc/services.d/xvnc/params && \
echo '# Window size.' >> /etc/services.d/xvnc/params && \
echo 'echo \"-geometry\"' >> /etc/services.d/xvnc/params && \
echo 'echo \"${DISPLAY_WIDTH}x${DISPLAY_HEIGHT}\"' >> /etc/services.d/xvnc/params && \
echo '' >> /etc/services.d/xvnc/params && \
echo '# Color depth.' >> /etc/services.d/xvnc/params && \
echo 'echo \"-depth\"' >> /etc/services.d/xvnc/params && \
echo 'echo \"24\"' >> /etc/services.d/xvnc/params && \
echo '' >> /etc/services.d/xvnc/params && \
echo '# VNC listening port.' >> /etc/services.d/xvnc/params && \
echo 'if [ \"${VNC_LISTENING_PORT:-5900}\" -eq -1 ]; then' >> /etc/services.d/xvnc/params && \
echo '    echo \"-rfbport=-1\"' >> /etc/services.d/xvnc/params && \
echo 'elif is-bool-val-true \"${SECURE_CONNECTION:-0}\" && [ \"${SECURE_CONNECTION_VNC_METHOD:-SSL}\" = \"SSL\" ]; then' >> /etc/services.d/xvnc/params && \
echo '    echo \"-rfbport=-1\"' >> /etc/services.d/xvnc/params && \
echo 'else' >> /etc/services.d/xvnc/params && \
echo '    echo \"-rfbport=${VNC_LISTENING_PORT:-5900}\"' >> /etc/services.d/xvnc/params && \
echo '    if ifconfig -a | grep -wq inet6; then' >> /etc/services.d/xvnc/params && \
echo '        echo \"-UseIPv6=yes\"' >> /etc/services.d/xvnc/params && \
echo '    else' >> /etc/services.d/xvnc/params && \
echo '        echo \"-UseIPv6=no\"' >> /etc/services.d/xvnc/params && \
echo '    fi' >> /etc/services.d/xvnc/params && \
echo 'fi' >> /etc/services.d/xvnc/params && \
echo '' >> /etc/services.d/xvnc/params && \
echo '# VNC Unix socket.' >> /etc/services.d/xvnc/params && \
echo 'echo \"-rfbunixpath=/tmp/vnc.sock\"' >> /etc/services.d/xvnc/params && \
echo 'echo \"-rfbunixmode=0660\"' >> /etc/services.d/xvnc/params && \
echo '' >> /etc/services.d/xvnc/params && \
echo '# VNC security.' >> /etc/services.d/xvnc/params && \
echo 'PASSWORD_FILE=' >> /etc/services.d/xvnc/params && \
echo 'if [ -f /config/.vncpass ] && [ -n \"$(cat /config/.vncpass)\" ]; then' >> /etc/services.d/xvnc/params && \
echo '    PASSWORD_FILE=/config/.vncpass' >> /etc/services.d/xvnc/params && \
echo 'elif [ -f /tmp/.vncpass ] && [ -n \"$(cat /tmp/.vncpass)\" ]; then' >> /etc/services.d/xvnc/params && \
echo '    PASSWORD_FILE=/tmp/.vncpass' >> /etc/services.d/xvnc/params && \
echo 'fi' >> /etc/services.d/xvnc/params && \
echo '' >> /etc/services.d/xvnc/params && \
echo 'if [ -n "$PASSWORD_FILE" ]; then' >> /etc/services.d/xvnc/params && \
echo '    if is-bool-val-false "${SECURE_CONNECTION:-0}" || [ "${SECURE_CONNECTION_VNC_METHOD:-SSL}" = "SSL" ]; then' >> /etc/services.d/xvnc/params && \
echo '        echo "-SecurityTypes=VncAuth"' >> /etc/services.d/xvnc/params && \
echo '    else' >> /etc/services.d/xvnc/params && \
echo '        echo "-SecurityTypes=X509Vnc,TLSVnc"' >> /etc/services.d/xvnc/params && \
echo '    fi' >> /etc/services.d/xvnc/params && \
echo '    echo "-rfbauth=$PASSWORD_FILE"' >> /etc/services.d/xvnc/params && \
echo 'else' >> /etc/services.d/xvnc/params && \
echo '    if is-bool-val-false "${SECURE_CONNECTION:-0}" || [ "${SECURE_CONNECTION_VNC_METHOD:-SSL}" = "SSL" ]; then' >> /etc/services.d/xvnc/params && \
echo '        echo "-SecurityTypes=None"' >> /etc/services.d/xvnc/params && \
echo '    else' >> /etc/services.d/xvnc/params && \
echo '        echo "-SecurityTypes=X509None,TLSNone"' >> /etc/services.d/xvnc/params && \
echo '    fi' >> /etc/services.d/xvnc/params && \
echo '    echo "-InternalConnectionSecurityTypes=None"' >> /etc/services.d/xvnc/params && \
echo 'fi' >> /etc/services.d/xvnc/params && \
echo '' >> /etc/services.d/xvnc/params && \
echo 'if is-bool-val-true "${SECURE_CONNECTION:-0}" && [ "${SECURE_CONNECTION_VNC_METHOD:-SSL}" != "SSL" ]; then' >> /etc/services.d/xvnc/params && \
echo '    echo "-X509Key"' >> /etc/services.d/xvnc/params && \
echo '    echo "/config/certs/vnc-privkey.pem"' >> /etc/services.d/xvnc/params && \
echo '    echo "-X509Cert"' >> /etc/services.d/xvnc/params && \
echo '    echo "/config/certs/vnc-fullchain.pem"' >> /etc/services.d/xvnc/params && \
echo 'fi' >> /etc/services.d/xvnc/params && \
echo 'echo "-desktop=$APP_NAME"' >> /etc/services.d/xvnc/params && \
echo 'echo "$DISPLAY"' >> /etc/services.d/xvnc/params
  

WORKDIR /data
