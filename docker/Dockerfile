# n8n self-hosted con yt-dlp instalado
# Necesario para que el nodo Execute Command pueda descargar transcripciones.
#
# Build:  docker build -t n8n-loterias docker/
# Run:    docker run -it --rm -p 5678:5678 -v n8n_data:/home/node/.n8n n8n-loterias

FROM n8nio/n8n:latest

USER root

# ffmpeg: requerido por yt-dlp para algunos formatos
# python3 + pip: yt-dlp se instala vía pip
RUN apk add --no-cache ffmpeg python3 py3-pip \
 && pip install --break-system-packages yt-dlp \
 && yt-dlp --version

USER node
