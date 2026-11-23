# syntax=docker/dockerfile:1
# ==============================================================================
# ComfyUI video imajı (RunPod) — tek Dockerfile, iki hedef (ada / blackwell)
# Hedefe özel ayarlar docker/targets/<hedef>.args dosyasındadır.
# İmajda sadece: torch, ComfyUI, custom node'lar, SageAttention.
# Modeller ve betikler Network Volume'da (/workspace) durur.
# ==============================================================================
ARG BASE_IMAGE=runpod/pytorch:1.0.2-cu1281-torch280-ubuntu2404
FROM ${BASE_IMAGE}

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# --- Hedefe özel ayarlar ---
# SageAttention hangi GPU mimarileri için derlenecek (GPU'suz CI'da şart)
ARG TORCH_CUDA_ARCH_LIST="8.0;8.6;8.9;9.0"
# Boş değilse torch bu paketlerle yeniden kurulur (Blackwell / CUDA 13 için)
ARG TORCH_PACKAGES=""
ARG TORCH_INDEX_URL=""
# Derleme paralelliği (GitHub runner: 4 CPU / 16 GB RAM)
ARG MAX_JOBS=2

# --- Sabitlenmiş sürümler (tekrarlanabilir build için) ---
ARG COMFYUI_REF=v0.46.2
ARG SAGEATTENTION_REF=d1a57a546c3d395b1ffcbeecc66d81db76f3b4b5
ARG MANAGER_REF=85d0221fac0d76a458be25696eecf7f61f8090bd
ARG GGUF_REF=6ea2651e7df66d7585f6ffee804b20e92fb38b8a
ARG KJNODES_REF=aa2b5ea5e7f61b51f5d26d726f23c1d74e951016
ARG VHS_REF=4d907bee61e92c2e65af3bd6383a4e4d356126d1
ARG WANWRAPPER_REF=a7439db30bed0c5b5fa437d77ae8a7bb220dfd00
ARG TRIPLEKSAMPLER_REF=b35509c84ff8a82f9111418aad8611a2ec3ef216
ARG CRYSTOOLS_REF=2f18256c5b5063937106f29a8e0a7db3ae3869b7

ENV DEBIAN_FRONTEND=noninteractive \
    PIP_NO_CACHE_DIR=1 \
    TORCH_CUDA_ARCH_LIST=${TORCH_CUDA_ARCH_LIST} \
    MAX_JOBS=${MAX_JOBS}

# 1. Sistem araçları
RUN apt-get update -qq \
 && apt-get install -y -qq --no-install-recommends aria2 curl ca-certificates \
 && rm -rf /var/lib/apt/lists/*

# 2. (Opsiyonel) torch'u hedef CUDA sürümüyle değiştir
RUN if [ -n "${TORCH_PACKAGES}" ]; then \
        pip install ${TORCH_PACKAGES} --index-url "${TORCH_INDEX_URL}"; \
    fi

# 3. torch sürümünü kilitle: eklentilerin requirements'ı torch'u sessizce değiştiremesin.
#    PIP_CONSTRAINT imajda kalır, ComfyUI-Manager kurulumlarını da korur.
RUN pip list --format=freeze \
      | grep -iE '^(torch|torchvision|torchaudio|triton)==' > /etc/pip-constraints.txt \
 && echo "Kilitlenen paketler:" && cat /etc/pip-constraints.txt
ENV PIP_CONSTRAINT=/etc/pip-constraints.txt

# 4. ComfyUI (/workspace volume'u maskelemesin diye /app içinde)
RUN git clone https://github.com/comfyanonymous/ComfyUI.git /app/ComfyUI \
 && git -C /app/ComfyUI checkout --quiet "${COMFYUI_REF}" \
 && pip install -r /app/ComfyUI/requirements.txt

# 5. Custom node'lar (her biri sabit commit'e)
RUN set -e; cd /app/ComfyUI/custom_nodes; \
    clone() { git clone --quiet "https://github.com/$1.git" "$2" && git -C "$2" checkout --quiet "$3"; }; \
    clone ltdrdata/ComfyUI-Manager              ComfyUI-Manager              "${MANAGER_REF}"; \
    clone city96/ComfyUI-GGUF                   ComfyUI-GGUF                 "${GGUF_REF}"; \
    clone kijai/ComfyUI-KJNodes                 ComfyUI-KJNodes              "${KJNODES_REF}"; \
    clone Kosinkadink/ComfyUI-VideoHelperSuite  ComfyUI-VideoHelperSuite     "${VHS_REF}"; \
    clone kijai/ComfyUI-WanVideoWrapper         ComfyUI-WanVideoWrapper      "${WANWRAPPER_REF}"; \
    clone VraethrDalkr/ComfyUI-TripleKSampler   ComfyUI-TripleKSampler       "${TRIPLEKSAMPLER_REF}"; \
    clone crystian/comfyui-crystools            comfyui-crystools            "${CRYSTOOLS_REF}"

# 6. Eklenti bağımlılıkları (hata olursa build dursun)
RUN pip install accelerate gguf deepdiff imageio-ffmpeg ftfy piexif diffusers py-cpuinfo \
 && for req in /app/ComfyUI/custom_nodes/*/requirements.txt; do \
        echo ">> $req"; pip install -r "$req"; \
    done

# 7. SageAttention 2.x — kaynaktan, TORCH_CUDA_ARCH_LIST mimarileri için
RUN pip install --no-build-isolation \
        "git+https://github.com/thu-ml/SageAttention.git@${SAGEATTENTION_REF}" \
 && pip show sageattention > /dev/null && echo "SageAttention kuruldu"
# (import testi burada yapılmaz: CI'da GPU/driver yok. start.sh çalışırken kontrol eder.)

# 8. ComfyUI-Manager: uv kullanmasın (CUDA paketlerini bozmasın)
RUN mkdir -p /app/ComfyUI/user/__manager \
 && printf '[default]\nuse_uv = False\npip_command = python3 -m pip\n' \
      > /app/ComfyUI/user/__manager/config.ini

EXPOSE 8188 8888 22

# Pod açılışı RunPod'un kendi /start.sh'ı ile olur (Jupyter + SSH). ComfyUI otomatik başlamaz;
# başlatma ve model indirme betikleri Network Volume'da durur (/workspace/scripts).
# (Önceki imajda çalıştığı doğrulanan WORKDIR)
WORKDIR /workspace
