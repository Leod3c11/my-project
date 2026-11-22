# Guia Técnico: Driver PanVK para Mali-G720 MC8 (Compatível com Adrenotools)

## 1. Visão Geral do Pacote

Este pacote contém o driver Vulkan aberto **PanVK** (baseado no **Mesa 27.0.0-devel / r10**), portado e otimizado para GPUs **ARM Mali de 5ª Geração com Command Stream Frontend (CSF)**, com foco específico na **Mali-G720 MC8** (e variantes Immortalis).

O driver opera no modelo **rootless** (sem necessidade de root ou modificação do sistema):
- Comunica-se diretamente com o driver de kernel proprietário da ARM (`mali_kbase` via nó `/dev/mali0`) já presente na ROM do aparelho.
- É empacotado no formato padronizado do **Adrenotools** (`meta.json` + bibliotecas `.so`), permitindo ser injetado em tempo de execução via `dlopen`/hook em emuladores e camadas de compatibilidade como **Winlator, Bannerlator, Eden, Skyline, Yuzu, Sudachi, ARMSX2 e Vita3K**.

---

## 2. Arquivos Gerados no Workspace

| Arquivo | Descrição |
| :--- | :--- |
| **`panvk-mali-g720-adrenotools.zip`** | Pacote Universal Adrenotools (`meta.json` com `libraryName: "vulkan.panfrost.so"`, incluindo ambos `vulkan.panfrost.so` e `libvulkan_panfrost.so`). |
| **`panvk-mali-g720-winlator.zip`** | Pacote direcionado para Winlator / Bannerlator (`meta.json` com `libraryName: "libvulkan_panfrost.so"`). |
| **`dxvk.conf`** | Configurações recomendadas de DXVK para mitigar tela preta em jogos DirectX 11/12 via PanVK no Mali-G720. |

---

## 3. Diagnóstico e Correção dos Problemas de "Black Screen" (Tela Preta)

Em compilações anteriores ou genéricas do PanVK, diversos fatores causavam o sintoma de tela preta (frequentemente com áudio do jogo continuando ao fundo ou travamento total na inicialização). Abaixo detalhamos a causa raiz e as soluções aplicadas:

### Causa 1: `GPU_SHAREABILITY_FAULT` (0x88) na Importação do Swapchain Gralloc (CSF)
* **Origem:** Em GPUs Mali CSF (como G720, G615, G715) operando em barramentos ACE-Lite (`coherency_mode = 0`), a rotina `kbase_kmod_bo_import` requisitava flags com `BASE_MEM_SAME_VA` (`0x4000f`) para buffers `AHardwareBuffer` do swapchain do Android.
* **Sintoma:** O MMU da GPU disparava falha de compartilhamento (`GPU_SHAREABILITY_FAULT`, código de exceção `0x88`), seguido de timeout de sinalização de fence (`FENCE_SIGNAL` 3000ms) no kernel `mali_kbase`. Isso resultava no descarte de 100% dos draws e tela preta imediata ao abrir o contexto 3D.
* **Correção:** No backend CSF, o driver utiliza `BASE_MEM_COHERENT_LOCAL` (`0x80f`) sem impor `SAME_VA`, espelhando o comportamento do blob proprietário da ARM e garantindo compatibilidade com o subsistema de memória do Android.

### Causa 2: Syncobjs Não Inicializados no CSF (`vk_queue_set_lost CS_FAULT`)
* **Origem:** Na alocação dos objetos de sincronização das subfilas (`struct panvk_cs_sync64`), apenas o campo `seqno = 1` era preenchido; os campos `error` e `pad` permaneciam com lixo de memória reciclada do pool de buffers (`device->mempools.sync`).
* **Sintoma:** Na verificação de status (`gpu_queue_check_status`), o driver lia o campo `last_error != 0` e declarava falsamente a perda do dispositivo (`CS_FAULT`), gerando uma taxa de ~25% a 30% de congelamento na inicialização em camadas como DXVK e Winlator.
* **Correção:** Inicialização explícita com `memset` e preenchimento determinístico de todos os membros (`seqno = 1, error = 0, pad = 0`).

### Causa 3: Dependência Estática de `libui.so` (Namespace do Linker Android)
* **Origem:** Compilações como a r9 tentavam carregar estaticamente `libui.so` do sistema para ler metadados do Gralloc. Emuladores Android executando como aplicativos comuns não possuem permissão no linker namespace para abrir `libui.so` de `/system/lib64/`.
* **Sintoma:** Falha silenciosa no `dlopen` do driver ou fallback indesejado para o driver proprietário defeituoso da fabricante, ou tela preta por falha na criação da swapchain.
* **Correção:** Implementação de parser interno e autônomo dos layouts de memória do Gralloc (Gralloc 0.3, 1.0, 2.x, 3.x e 4.x), tornando `libui.so` puramente opcional via `dlopen` dinâmico em tempo de execução.

### Causa 4: Otimização de Tile-Skipping de Hardware do Mali CSF (Stale Render Targets)
* **Origem:** As GPUs de 5ª geração (G610, G710, G720) possuem hardware que ignora a escrita de tiles (quadrantes da tela) caso eles não tenham mudado desde o último frame. Quando emuladores reutilizam a memória de um render target para múltiplas passagens, o hardware continuava confiando nos dados antigos da cache de tiles.
* **Sintoma:** Partes da tela congeladas, artefatos pretos piscando ou tela inteiramente preta.
* **Correção:** Invalidação forçada do estado dos tiles e atualização das flags de dirty state a cada transição de render pass.

### Causa 5: Falta de Descompressão de Texturas BCn (BC1–BC7)
* **Origem:** GPUs mobile Mali não possuem unidades fixas de silício para descomprimir formatos de textura típicos de PC (DirectX BC1 a BC7 / DXTn).
* **Sintoma:** Emuladores de PC (Winlator, Mobox) tentavam renderizar shaders com texturas não decodificadas, fazendo com que a cena ficasse preta (embora a interface do usuário desenhasse normalmente).
* **Correção:** Habilitação do pipeline de transcodificação de formatos no Mesa PanVK e inclusão do arquivo `dxvk.conf` com barreiras relaxadas (`d3d11.relaxedBarriers = True`).

### Causa 6: Erro no Separador de Pasta no `meta.json`
* **Origem:** Builds com nome contendo barras (`Bifrost/Valhall`) faziam com que o instalador do Winlator interpretasse a barra como diretório, gravando o driver em uma pasta corrompida chamada `tmp` e falhando ao carregar.
* **Correção:** Nome sanitizado sem caracteres especiais: `"PanVK Mali G720 MC8 (Mesa 27.0.0-devel)"`.

---

## 4. Como Usar o Driver no Adrenotools

### No Winlator / Bannerlator
1. Copie o arquivo `panvk-mali-g720-winlator.zip` (ou `panvk-mali-g720-adrenotools.zip`) para a memória interna do seu dispositivo.
2. Abra o **Winlator** e vá no menu lateral -> **Settings** -> **Driver Manager** (ou Gerenciador de Drivers).
3. Toque no ícone **+** (Instalar Driver) e selecione o arquivo `.zip`.
4. O driver aparecerá como `PanVK Mali G720 MC8`.
5. Edite o seu Container no Winlator:
   - Em **Graphics Driver**, selecione **PanVK Mali G720 MC8**.
   - Em **DXVK**, utilize uma versão estável (recomendado **DXVK 2.4.1** ou **2.5** com o arquivo `dxvk.conf` fornecido).

### No Yuzu / Sudachi / Skyline / Vita3K
1. Abra as **Configurações** do emulador.
2. Acesse **GPU Driver Manager** (Gerenciador de Drivers de GPU).
3. Toque em **Install** (Instalar driver customizado).
4. Selecione o arquivo `panvk-mali-g720-adrenotools.zip`.
5. Ative o driver recém-instalado.

---

## 5. Dicas Adicionais para Evitar Black Screen em Jogos
- **Variáveis de Ambiente Úteis (Winlator):**
  - `MESA_SHADER_CACHE_DISABLE=true` (caso ocorra crash por cache de shaders antigo corrompido).
  - `PANVK_STENCIL_LATE=1` (ajuda em títulos com geometria ou iluminação ausente).
- **Substituição de DXVK:** Coloque o arquivo `dxvk.conf` na mesma pasta do arquivo executável (`.exe`) do jogo dentro do container.
