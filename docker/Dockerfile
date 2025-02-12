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
    # Display
    DISPLAY=:0 \
    DISPLAY_HEIGHT=800 \
    DISPLAY_WIDTH=1280 \
    DISABLE_RENDERING=true\
    # SMAPI
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
    # Includes pgrep for service handling
    procps \
    unzip \
    # Includes xev for resize-handler.sh
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
    # Used by polybar taskbar script
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

RUN \
    # Clear apt cache
    apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* && \
    # Fix stuck dbus statoverride
    rm /var/lib/dpkg/statoverride && \
    # Prevent unnecessary logs
    rm /etc/cont-init.d/89-info.sh && \
    # Prevent overriding OpenGL drivers
    rm -rf /etc/cont-env.d/LIBGL_DRIVERS_PATH

# Copy mods 
COPY .output/mods /data/Mods

# Copy remaining files 
COPY docker/rootfs/ /

# Embed the patched xvnc parameters file.
# This replaces the original file with a version that omits the problematic option.
RUN cat << 'EOF' > /etc/services.d/xvnc/params
#!/bin/sh

set -u # Treat unset variables as an error.

# Activate debug logging.
if is-bool-val-true "${CONTAINER_DEBUG:-0}"; then
    echo "-Log"
    echo "*:stderr:100"
    echo "-verbose"
    echo "10"
fi

# Make sure the X server listens on Unix only.
echo "-nolisten"
echo "tcp"
echo "-nolisten"
echo "local"
echo "-listen"
echo "unix"

# Window size.
echo "-geometry"
echo "${DISPLAY_WIDTH}x${DISPLAY_HEIGHT}"

# Color depth.
echo "-depth"
echo "24"

# VNC listening port.
if [ "${VNC_LISTENING_PORT:-5900}" -eq -1 ]; then
    # VNC port is disabled.
    echo "-rfbport=-1"
elif is-bool-val-true "${SECURE_CONNECTION:-0}" && [ "${SECURE_CONNECTION_VNC_METHOD:-SSL}" = "SSL" ]; then
    # SSL secure connection enabled: disable TCP listening.
    echo "-rfbport=-1"
else
    echo "-rfbport=${VNC_LISTENING_PORT:-5900}"
    # Support for IPv6.
    if ifconfig -a | grep -wq inet6; then
        echo "-UseIPv6=yes"
    else
        echo "-UseIPv6=no"
    fi
fi

# VNC Unix socket.
echo "-rfbunixpath=/tmp/vnc.sock"
echo "-rfbunixmode=0660"
# VNC security.
#
# None:      No authentication, no encryption.
# VncAuth:   Standard VNC authentication, no encryption.
# Plain:
# TLSNone:   No authentication, TLS encryption.
# TLSVnc:    Standard VNC authentication, TLS encryption.
# TLSPlain:
# X509None:  No authentication, TLS encryption with server certificate verification.
# X509Vnc:   Standard VNC authentication, TLS encryption with server certificate verification.
# X509Plain:
#
PASSWORD_FILE=
if [ -f /config/.vncpass ] && [ -n "$(cat /config/.vncpass)" ]; then
    PASSWORD_FILE=/config/.vncpass
elif [ -f /tmp/.vncpass ] && [ -n "$(cat /tmp/.vncpass)" ]; then
    PASSWORD_FILE=/tmp/.vncpass
fi

if [ -n "$PASSWORD_FILE" ]; then
    # With password.
    if is-bool-val-false "${SECURE_CONNECTION:-0}" || [ "${SECURE_CONNECTION_VNC_METHOD:-SSL}" = "SSL" ]; then
        echo "-SecurityTypes=VncAuth"
    else
        echo "-SecurityTypes=X509Vnc,TLSVnc"
    fi
    # The problematic option is removed here.
    echo "-rfbauth=$PASSWORD_FILE"
else
    # Without password.
    if is-bool-val-false "${SECURE_CONNECTION:-0}" || [ "${SECURE_CONNECTION_VNC_METHOD:-SSL}" = "SSL" ]; then
        echo "-SecurityTypes=None"
    else
        echo "-SecurityTypes=X509None,TLSNone"
    fi
    echo "-InternalConnectionSecurityTypes=None"
fi

if is-bool-val-true "${SECURE_CONNECTION:-0}" && [ "${SECURE_CONNECTION_VNC_METHOD:-SSL}" != "SSL" ]; then
    echo "-X509Key"
    echo "/config/certs/vnc-privkey.pem"
    echo "-X509Cert"
    echo "/config/certs/vnc-fullchain.pem"
fi

# Desktop (app) name.
echo "-desktop=$APP_NAME"

# X11 display.
echo "$DISPLAY"
EOF

WORKDIR /data
