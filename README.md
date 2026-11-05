# Guia: Configuração do Mesa Turnip Vulkan no Termux + Proot (glibc)

## 1. Objetivo

Configurar o driver gráfico **Mesa Turnip** dentro de uma distribuição Linux com glibc executada pelo Termux através do Proot, utilizando a GPU Adreno do dispositivo Android.
Testado no S25 Ultra com Adreno 830

Neste procedimento, utilizamos:

| Componente | Configuração |
|---|---|
| Host | Android + Termux |
| Ambiente Linux | Debian 13 (Trixie) ARM64 |
| glibc | 2.41 |
| GPU | Qualcomm Adreno 830 |
| Driver | Banners-Turnip |
| Variante | Linux runtime (`-Linux`) |
| Vulkan | 1.4 |
| Mesa | 27.0.0-devel |

**Repositório do driver:**  
https://github.com/The412Banner/Banners-Turnip/releases

É importante baixar a variante `-Linux`, pois ela é destinada a runtimes Linux com glibc. As variantes X11 e Wayland são destinadas a outros ambientes, como Bannerlator.

---

## Primeiro, instalação no Termux:

Termux:
``` bash
pkg update && pkg upgrade -y
pkg install x11-repo -y
pkg install termux-x11-nightly -y
```

## Segundo, instalação dentro do Debian:

``` bash
apt update && apt upgrade -y
apt install mesa-vulkan-drivers vulkan-tools mesa-utils glmark2-x11 x11-utils -y
```

## Baixar o driver Adreno especifico para Linux ARM com glibc (Linux comum)
Baixar o https://github.com/The412Banner/Banners-Turnip/releases que é a lib compilada para Linux ARM glibc (Bannerlator Linux runtime).

Vou separar o .so do JSON, porque cada um tem o seu lugar convencional.

Sugestão

- .so em /opt/turnip/ (diretório para software de terceiros, que o apt não mexe).
- JSON em /usr/local/share/vulkan/icd.d/, um dos caminhos que o loader Vulkan varre sozinho, reservado para instalações manuais.
- Não use /usr/share/vulkan/icd.d/ com o nome original. O pacote mesa-vulkan-drivers do Debian já instala um freedreno_icd.aarch64.json lá. O dpkg é dono desse arquivo, e um upgrade pode sobrescrevê-lo ou dar conflito. Por isso usei outro diretório e outro nome (turnip_banner_icd.json).

Modifique o .json com o novo path /opt/turnip/nome.so

## 2. Problema: dependência Wayland incompatível

Após baixar o driver e colocá-lo em `/opt/turnip`, executamos:

```bash
vulkaninfo --summary
```

O resultado foi:

```text
ERROR: [Loader Message] Code 0 :
/opt/turnip/libvulkan_freedreno.so: undefined symbol: wl_fixes_interface

loader_icd_scan: Failed loading library associated with ICD JSON

vkCreateInstance: Found no drivers!
```

### Diagnóstico

O driver dependia de símbolos que não estavam disponíveis na versão do Wayland instalada no Debian.

Verificamos a versão da glibc:

```bash
ldd --version | head -1
```

Resultado:

```text
ldd (Debian GLIBC 2.41-12+deb13u4) 2.41
```

A glibc atendia ao requisito do driver (2.38 ou superior).

Em seguida, verificamos a versão do Wayland:

```bash
dpkg-query -W -f='${Version}\n' libwayland-client0
```

Resultado:

```text
1.23.1-3
```

Também verificamos a existência do símbolo:

```bash
nm -D /lib/aarch64-linux-gnu/libwayland-client.so.0 | grep wl_fixes_interface
```

Nenhum resultado foi encontrado.

### Solução

Utilizamos uma versão mais recente da biblioteca Wayland, sem atualizar ou modificar os pacotes do Debian.

Baixamos o pacote ARM64 do Debian Sid:

```bash
wget https://deb.debian.org/debian/pool/main/w/wayland/libwayland-client0_1.26.0-1_arm64.deb
```

Criamos um diretório isolado:

```bash
mkdir -p /opt/turnip/lib
```

Extraímos o pacote:

```bash
mkdir -p /tmp/wayland

dpkg-deb -x libwayland-client0_1.26.0-1_arm64.deb /tmp/wayland
```

Copiamos as bibliotecas:

```bash
cp -a /tmp/wayland/usr/lib/aarch64-linux-gnu/libwayland-client.so* /opt/turnip/lib/
```

**Por que fizemos isso?**

Dessa maneira, conseguimos utilizar uma versão mais recente do Wayland exclusivamente para o Turnip, sem substituir a biblioteca do sistema ou misturar permanentemente os repositórios Trixie e Sid.

### Verificação

Executamos:

```bash
nm -D /opt/turnip/lib/libwayland-client.so.0 | grep -E 'wl_fixes_interface|wl_display_dispatch_queue_timeout'
```

Os dois símbolos foram encontrados:

```text
T wl_display_dispatch_queue_timeout
D wl_fixes_interface
```

Isso confirmou que a biblioteca continha as implementações exigidas pelo driver.

---

## 3. Problema: linker não encontrava os símbolos necessários

Mesmo com a nova biblioteca instalada, precisávamos garantir que o linker dinâmico utilizasse a versão correta.

Para isso, configuramos temporariamente:

```bash
export LD_LIBRARY_PATH=/opt/turnip/lib:$LD_LIBRARY_PATH
```

Executamos:

```bash
ldd -r /opt/turnip/lib/libvulkan_freedreno.so
```

Inicialmente, sem a configuração correta, os resultados indicavam:

```text
undefined symbol: wl_fixes_interface
undefined symbol: wl_display_dispatch_queue_timeout
```

Após configurar o `LD_LIBRARY_PATH`, o resultado passou a mostrar:

```text
libwayland-client.so.0 => /opt/turnip/lib/libwayland-client.so.0
```

Nenhum símbolo indefinido permaneceu.

**Conclusão:** o driver estava carregando a versão correta do Wayland e todas as dependências verificadas estavam resolvidas.

---

## 4. Configuração do Vulkan ICD

O Vulkan Loader precisa saber onde encontrar o driver instalado.

Configuramos a variável:

```bash
export VK_ICD_FILENAMES=/usr/local/share/vulkan/icd.d/turnip_banner_icd.json
```

Esse é o caminho utilizado em nossa instalação. O manifesto JSON precisa apontar para a biblioteca `libvulkan_freedreno.so` correspondente.

Podemos conferir sua existência com:

```bash
cat /usr/local/share/vulkan/icd.d/turnip_banner_icd.json
```

E verificar se o arquivo da biblioteca está presente:

```bash
ls -l /opt/turnip/libvulkan_freedreno.so
```

---

## 5. Problema: ausência de DISPLAY e XDG_RUNTIME_DIR

Na primeira execução bem-sucedida:

```bash
vulkaninfo --summary
```

Recebemos:

```text
'DISPLAY' environment variable not set... skipping surface info

error: XDG_RUNTIME_DIR is invalid or not set in the environment.
```

### Diagnóstico

Não tínhamos uma interface gráfica instalada no proot.

Essas mensagens estão relacionadas à ausência de uma sessão gráfica e de um diretório de runtime apropriado.

Isso não impede necessariamente a inicialização do Vulkan nem a utilização da GPU para computação.

### Solução

Configuramos um diretório runtime:

```bash
export XDG_RUNTIME_DIR=/tmp/runtime-$USER

mkdir -p "$XDG_RUNTIME_DIR"
chmod 700 "$XDG_RUNTIME_DIR"
```

Não foi necessário instalar X11, Wayland compositor ou ambiente desktop.

Também não foi necessário configurar `DISPLAY` para executar `vulkaninfo --summary`.

---

## Scripts de inicialização

Para acessar o Linux com usuario especifico e compartilhamento de GPU:
``` bash
proot-distro login debian --user caio --shared-tmp --bind /dev/kgsl-3d0:/dev/kgsl-3d0
```

Para acessar o Linux iniciando o i3 junto automaticamente:
``` bash
#!/data/data/com.termux/files/usr/bin/bash

# Kill open X11 processes
kill -9 $(pgrep -f "termux.x11") 2>/dev/null

# Enable PulseAudio over Network
# pulseaudio --start --load="module-native-protocol-tcp auth-ip-acl=127.0.0.1 auth-anonymous=1" --exit-idle-time=-1

# Prepare termux-x11 session
termux-x11 :0 >/dev/null &

# Wait a bit until termux-x11 gets started.
sleep 3

# Launch Termux X11 main activity
am start --user 0 -n com.termux.x11/com.termux.x11.MainActivity > /dev/null 2>&1
sleep 1

proot-distro login debian \
    --user caio \
    --shared-tmp \
    --bind /dev/kgsl-3d0:/dev/kgsl-3d0 \
    -- /bin/bash -c '
        source ~/.bashrc
        export DISPLAY=:0
        exec dbus-run-session i3
    '
```

## 6. Configurações permanentes no .bashrc

Depois de resolver os problemas, reunimos as configurações necessárias em um único lugar.

Editamos:

```bash
nano ~/.bashrc
```

Adicionamos ao final:

```bash
# Vulkan ICD
export VK_DRIVER_FILES="/usr/local/share/vulkan/icd.d/turnip_banner_icd.json"
export MESA_LOADER_DRIVER_OVERRIDE=zink
export TU_DEBUG=noconform

# Turnip Vulkan
export LD_LIBRARY_PATH="/opt/turnip/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"

# Runtime directory
export XDG_RUNTIME_DIR="/tmp/runtime-$USER"
mkdir -p "$XDG_RUNTIME_DIR"
chmod 700 "$XDG_RUNTIME_DIR"
```

Recarregamos as configurações:

```bash
source ~/.bashrc
```

A partir daí, não precisamos configurar essas variáveis manualmente a cada sessão.

**Observação:** `VK_ICD_FILENAMES` variavel antiga para `VK_DRIVER_FILES`.

---

## Instalação do i3

```bash
apt update

apt install -y \
    i3-wm \
    xterm \
    dbus-x11 \
    dbus-daemon \
    libgl1-mesa-dri \
    libglx-mesa0 \
    libegl-mesa0 \
    mesa-utils
```


## 7. Verificação final

Executamos:

```bash
vulkaninfo --summary
```

O resultado confirmou a inicialização correta do driver:

```text
VULKANINFO

Vulkan Instance Version: 1.4.309

Devices:
========

GPU0:
    apiVersion         = 1.4.367
    driverVersion      = 26.99.99
    vendorID           = 0x5143
    deviceID           = 0x44050001
    deviceType         = PHYSICAL_DEVICE_TYPE_INTEGRATED_GPU
    deviceName         = Adreno (TM) 830v1
    driverID           = DRIVER_ID_MESA_TURNIP
    driverName         = turnip Mesa driver
    driverInfo         = Mesa 27.0.0-devel (git-d76d1f6279)
    conformanceVersion = 1.4.6.1
```

### Resultado

| Verificação | Status |
|---|---|
| glibc compatível | OK |
| Wayland atualizado | OK |
| Símbolos necessários disponíveis | OK |
| Dependências do Turnip resolvidas | OK |
| Vulkan ICD carregado | OK |
| Adreno 830 reconhecida | OK |
| Vulkan 1.4 inicializado | OK |

---

## 8. Executando aplicações com a GPU

Com o ambiente configurado, aplicações que utilizam Vulkan podem acessar o driver sem precisar especificar manualmente a GPU.

O Vulkan Loader encontra o ICD através de `VK_ICD_FILENAMES`, e o driver disponibiliza a Adreno 830 como dispositivo Vulkan.

Entretanto, a aplicação precisa implementar suporte a Vulkan. Aplicações OpenGL, CUDA ou que utilizam renderização por software não passam a usar Turnip automaticamente.

Para verificar o carregamento do driver durante a execução de uma aplicação:

```bash
VK_LOADER_DEBUG=all programa
```

Para conferir novamente o dispositivo:

```bash
vulkaninfo --summary
```

Não é necessário configurar `DISPLAY` ou `WAYLAND_DISPLAY` para aplicações que realizam computação Vulkan sem apresentação gráfica.

---

## 9. Considerações finais

Alguns pontos importantes para futuras instalações:

1. Sempre utilizar a variante `-Linux` do Banners-Turnip quando o objetivo for um runtime glibc.
2. Conferir se a glibc atende ao requisito mínimo do driver.
3. Verificar as dependências com `ldd` e `ldd -r`.
4. Caso o Wayland do Debian seja antigo, utilizar uma versão mais recente isoladamente.
5. Manter as bibliotecas adicionais em `/opt/turnip/lib`, evitando substituir bibliotecas do sistema.
6. Configurar o manifesto ICD corretamente.
7. Não é necessário instalar um ambiente gráfico para inicializar Vulkan.

**Estado final:** Mesa Turnip 27.0.0-devel funcionando no Debian 13 ARM64 dentro do Termux/Proot, reconhecendo a Qualcomm Adreno 830 e expondo Vulkan 1.4.
