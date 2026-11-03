# Overview do repositório Kirikae

Este documento apresenta a estrutura, o fluxo de execução e as principais decisões técnicas do
repositório `loddv/kirikae`. Ele foi produzido a partir do branch `main`, commit
`726c4a1` (`chore: update progress`), em 2026-11-03.

## 1. Resumo executivo

Kirikae é um **recompilador estático de executáveis do Nintendo Switch para programas nativos de
PC**, escrito integralmente em Rust. O projeto lê dumps já descriptografados, descobre funções ARM64,
converte as instruções para uma IR própria e gera um programa nativo que incorpora:

- o código recompilado do jogo;
- o runtime `kirikae-rt`, que emula memória, CPU, threads, kernel, serviços Horizon, GPU, áudio,
  vídeo e entrada;
- os dados do executável e, quando aplicável, o RomFS empacotado.

O workspace usa Rust 2024, versão `0.1.2`, licença dupla MIT ou Apache-2.0 e tem como plataformas
principais Windows x64 e Linux x64 com Vulkan 1.3. O código Rust possui aproximadamente 54 mil linhas
distribuídas por 159 arquivos, sendo o runtime responsável pela maior parte da implementação.

O projeto está em **early preview**: jogos comerciais completos já executam, mas ainda podem ocorrer
crashes, serviços ausentes e problemas de renderização. O dashboard em `docs/progress/` e os
relatórios por jogo mostram a cobertura atual.

## 2. Escopo e regras do projeto

Kirikae não é um emulador genérico tradicional nem um distribuidor de jogos. O repositório:

- aceita apenas dumps já descriptografados;
- não contém jogos, firmware, chaves, BIOS, SDKs ou executáveis gerados a partir de jogos;
- não implementa descriptografia;
- espera que o usuário use somente jogos extraídos do próprio console;
- não deve distribuir os executáveis recompilados, símbolos, perfis, saves ou capturas gerados;
- mantém o trabalho de engenharia reversa em referências públicas e código com licença compatível;
- não aceita código GPL, material vazado, material sob NDA ou dados de jogos em testes/documentação.

As regras completas estão em [LEGAL.md](LEGAL.md). As contribuições devem seguir
[CONTRIBUTING.md](CONTRIBUTING.md); issues e pull requests podem ser escritos em inglês ou português.

## 3. Fluxo geral

O fluxo de build é dividido em preparação, análise, geração e execução:

```text
dump (.dnsp, ExeFS, .nso, .nro)
             |
             v
loader: lê módulos, metadados e RomFS
             |
             v
recomp::discover: encontra entradas, símbolos, relocations,
                  chamadas, ADR/ADRP+ADD e jump tables
             |
             v
a64: decodifica ARM64 e levanta instruções para a IR
             |
             +----------------------------+
             |                            |
             v                            v
      Cranelift -> .obj              Rust backend -> Rust
             |                            |
             +-------------+--------------+
                           v
               projeto Cargo temporário
                           |
                           v
       programa nativo + kirikae-rt + dump embutido
```

Durante a execução, chamadas diretas tornam-se chamadas nativas. Saltos indiretos consultam uma
tabela de funções recompiladas; quando o endereço não foi descoberto no build, o runtime usa o
interpretador ARM64. Os endereços que caíram no interpretador são gravados em um perfil para que o
próximo build possa compilá-los.

## 4. Organização do workspace

O workspace é definido no [Cargo.toml](Cargo.toml) e contém oito crates:

| Crate | Responsabilidade | Pontos de entrada |
| --- | --- | --- |
| `kirikae-a64` | Decodificação ARM64/AArch64, suporte A32, IR, semântica de operações e SIMD/FP. | `crates/a64/src/ir.rs`, `lift.rs`, `ops.rs`, `simd/` |
| `kirikae-loader` | Leitura dos formatos de executáveis, organização dos módulos no espaço de endereços convidado e construção/leitura do RomFS. | `crates/loader/src/package.rs`, `nro.rs`, `nso.rs`, `nca.rs`, `romfs.rs` |
| `kirikae-recomp` | Descoberta de funções, construção de CFGs, backend Cranelift, backend Rust e geração do projeto Cargo do jogo. | `crates/recomp/src/discover.rs`, `clif.rs`, `rust_backend.rs`, `project.rs` |
| `kirikae-rt` | Runtime vinculado a cada programa recompilado: memória, CPU, interpretador, kernel, serviços, GPU, vídeo, áudio e input. | `crates/rt/src/process.rs`, `mem.rs`, `horizon.rs`, `gpu/`, `services/` |
| `kirikae-maxwell` | Decodificação de shaders Maxwell do Tegra X1, análise de fluxo e tradução para SPIR-V. | `crates/maxwell/src/decode.rs`, `cfg.rs`, `translate.rs` |
| `kirikae` | Ferramenta de linha de comando para build, execução interpretada, inspeção, emissão de Rust e comandos de desenvolvimento. | `crates/kirikae/src/main.rs` |
| `kirikae-gui` | Interface egui para selecionar dumps, definir a pasta de saída e executar o CLI. | `crates/gui/src/lib.rs`, `main.rs` |
| `kirikae-testkit` | Geração de NROs sintéticos para testes sem dados de jogos. | `crates/testkit/src/lib.rs` |

### Dependências e características relevantes

- `kirikae-recomp` depende de Cranelift `0.136` e usa o `kirikae-a64`, `kirikae-loader` e
  `kirikae-rt`.
- `kirikae-rt` usa `ash` para Vulkan, `winit`/`softbuffer` para janela e apresentação, `cpal` para
  áudio, `vp9dec` para VP9 e opcionalmente OpenH264 para H.264.
- A feature padrão de `kirikae-rt` é `h264`; ela pode ser desabilitada com `default-features = false`.
- `kirikae-maxwell` gera SPIR-V com `rspirv`; os testes usam `naga` para validar a saída.
- O GUI usa `eframe`/egui e `rfd`, mas não incorpora a lógica do recompilador: inicia o binário
  `kirikae` localizado ao lado dele.
- `kirikae-testkit` cria programas artificiais para manter os testes livres de dados proprietários.

## 5. Componentes centrais

### 5.1 Loader e formatos de entrada

`kirikae-loader` normaliza diferentes formas de entrada para um `Executable` com módulos, entry
point, segmentos, metadados e RomFS:

- pastas com `exefs/` e `romfs/`;
- `.nro` de homebrew;
- `.nso` isolado;
- `.dnsp`, que contém o programa em PFS0/NCA plaintext;
- imagens RomFS ou `romfs.bin` ao lado do ExeFS.

O loader reconhece NRO, NSO/LZ4, PFS0, NCA, CNMT, NACP e RomFS. Os módulos são posicionados a partir
de `LOAD_BASE = 0x0800_0000`, na ordem `rtld`, `main`, `subsdk*` e `sdk`. O `main.npdm` fornece,
quando disponível, o title ID, o tamanho da stack principal e o recurso de memória do sistema.

Quando uma pasta `romfs/` não possui `romfs.bin`, o loader constrói a imagem uma vez e a reutiliza
nas execuções seguintes. Updates e DLC ainda não são suportados pelo fluxo principal.

### 5.2 Decodificação, IR e recompilação

O crate `kirikae-a64` mantém uma única fonte de verdade para as instruções:

1. o decoder interpreta a palavra ARM64;
2. o lifter produz uma IR pequena em `ir.rs`;
3. as operações concretas vivem em `ops.rs`;
4. o interpretador usa diretamente essa semântica;
5. os backends nativos são comparados com ela nos testes.

`kirikae-recomp::discover` procura funções a partir de:

- entry points dos módulos;
- ponteiros e símbolos dinâmicos;
- relocations e destinos de `BL`;
- padrões `ADR` e `ADRP+ADD`;
- jump tables e branches indiretos reconhecíveis;
- endereços registrados pelo perfil de execução anterior.

O backend Cranelift é o padrão, compila funções em paralelo e agrupa até 1024 funções por objeto.
O backend Rust é uma implementação de referência útil para inspeção e programas menores. A opção
experimental `--fast-regs` mantém registradores convidados em variáveis e exige Cranelift.

O projeto gerado contém um `package.bin`, o código compilado, um `build.rs` para linkar os objetos e
uma dependência local de `kirikae-rt`. O caminho do runtime pode ser alterado por
`KIRIKAE_RUNTIME`; pacotes de release já incluem `runtime/` ao lado dos binários.

### 5.3 Runtime e processo convidado

`kirikae-rt` cria um `Process` que reúne memória convidada, módulos, símbolos, handles, scheduler,
GPU, tabela HLE, estado dos serviços e estatísticas. As áreas principais são:

- **Memória e CPU:** reserva do espaço de endereços de 39 bits, estado dos registradores X/V,
  flags, TLS, monitor exclusivo, stack e mapeamentos físicos.
- **Execução:** tabela de funções nativas, cache de lookup, interpretador para código não descoberto,
  backtraces e contagem de instruções nativas versus interpretadas.
- **Kernel Horizon:** SVCs em `horizon.rs`, threads host por thread convidada, prioridades, afinidade,
  sincronização, mutexes, condition variables, arbiters, handles, sessões, domains e HIPC/CMIF.
- **Serviços:** uma família por arquivo em `crates/rt/src/services/`, com fallback em `stub.rs`;
  `usage.rs` registra os comandos usados e alimenta os relatórios.
- **Perfil e cobertura:** endereços interpretados, comandos IPC, SVCs, ioctls `nvdrv` e instruções
  de shader são registrados para diagnosticar e orientar builds futuros.

O acesso convidado é deliberadamente concentrado na memória reservada. Um ponteiro inválido deve
causar uma falha controlada no espaço convidado, em vez de corromper memória arbitrária do host.

### 5.4 GPU, shaders e NVN

O caminho GPU principal é:

```text
nvdrv ioctl / pushbuffer
        -> processador de comandos Maxwell
        -> métodos 2D/3D, compute, DMA e MME
        -> recursos, texturas, buffers e imagens
        -> tradução Maxwell -> SPIR-V
        -> pipelines Vulkan e apresentação
```

O runtime contém:

- decodificação de comandos Maxwell em `gpu/`;
- renderer Vulkan, cache de pipelines, descriptors, submit e swapchain;
- formatos, ASTC, tiling block-linear, detiling, texturas 1D/2D/3D/cubemap;
- buffers de vértices/índices, render targets, readback e rastreamento de escritas do guest;
- `host1x` para NVDEC/VIC, incluindo VP9, H.264 opcional e composição de vídeo;
- áudio `audren`, mixer, DSP e saída do host.

O crate `kirikae-maxwell` traduz shaders para SPIR-V conforme o estágio (vertex, fragment ou
compute), seus descriptors, imagens, cbufs, shared memory, operações de textura e recursos de
debug. O modo NVN de host está em evolução:

- `trace` conta chamadas NVN;
- `shadow` cria um modelo host e compara cada draw com o processador de comandos;
- `encode` prevê registradores e relata divergências;
- `exec` executa no host os segmentos suportados e faz fallback por segmento;
- `check` mantém o caminho atual, mas valida o que o modelo teria produzido.

### 5.5 GUI e plugins

A GUI aceita arquivos `.dnsp`, `.nso`, `.nro` e pastas de jogo, além de conversores registrados por
wrappers. Ela:

1. valida e lista os arquivos;
2. permite definir a pasta de saída;
3. chama `kirikae build` em uma thread de trabalho;
4. encaminha stdout/stderr para o painel de log;
5. mostra estado de cada build e abre a pasta do executável concluído.

Plugins são crates externos passados com `kirikae build --plugin`. Eles são linkados junto ao jogo e
seu `install()` é chamado antes da execução. A API em `kirikae_rt::games` é destinada a ajustes
específicos de jogo, como ritmo de apresentação e ações por frame; correções de comportamento real
do hardware devem permanecer no runtime.

## 6. Interface de linha de comando

O binário principal é implementado em [crates/kirikae/src/main.rs](crates/kirikae/src/main.rs).

| Comando | Uso |
| --- | --- |
| `kirikae build <arquivo> -o <dir>` | Descobre, recompila e gera o executável nativo. Aceita `--backend`, `-j`, `--fast-regs`, `--plugin`, `--profile` e `--no-profile`. |
| `kirikae run <arquivo>` | Executa somente com o interpretador, útil para smoke tests e diagnóstico. |
| `kirikae info <arquivo>` | Exibe módulos, segmentos, entry point, metadados e cobertura do decoder. |
| `kirikae emit <arquivo.nro>` | Emite o programa Rust gerado pelo backend de referência. |
| `kirikae testgen <saida.nro>` | Gera o programa sintético usado pelos testes. |
| `kirikae dev simd-cases` / `simd-check` | Gera e valida casos SIMD/FP contra o oracle. |
| `kirikae dev extract <dump> <dir>` | Extrai ExeFS e RomFS para o layout de pasta aceito pelo build. |
| `kirikae dev module-dump <arquivo> <modulo> <saida>` | Exporta a imagem carregada de um módulo. |
| `kirikae dev syms <arquivo> [padrao]` | Lista símbolos de função exportados. |
| `kirikae dev romfs-ls` / `romfs-find` | Lista ou procura arquivos no RomFS. |

Exemplo mínimo:

```bash
cargo build --release -p kirikae -p kirikae-gui
kirikae build game.dnsp -o out
KIRIKAE_STATS=1 out/game.exe
```

## 7. Perfil, cobertura e diagnóstico

### Perfil de recompilação

O executável grava periodicamente os endereços que precisaram do interpretador em
`<nome>.profile`. O próximo `kirikae build` usa esse arquivo por padrão e transforma esses endereços
em funções recompiladas. Um arquivo `.profile.guest` identifica o dump; perfis de outro jogo ou
versão são rejeitados. `--profile <arquivo>` seleciona outro perfil e `--no-profile` desativa o uso
de qualquer perfil.

### Cobertura de uso

O runtime gera `<nome>.coverage.json` e eventos adicionais com backtrace convidado, frame PNG e
cópia de saves quando encontra algo não implementado. Esses dados são locais e podem conter dados
do jogo; não devem ser adicionados ao repositório. Para analisá-los:

```bash
python tools/coverage_report.py out/game.coverage.json
python tools/progress.py --coverage out/
```

### Variáveis de ambiente

As opções de diagnóstico estão documentadas em [README.md](README.md), incluindo:

- performance: `KIRIKAE_STATS`, `KIRIKAE_PARALLEL`, `KIRIKAE_NO_FRAME_LIMIT`;
- compilação e execução: `KIRIKAE_RUNTIME`, `KIRIKAE_PROFILE`, `KIRIKAE_COVERAGE`;
- GPU: `KIRIKAE_TRACE_GPU`, `KIRIKAE_GPU_TIME`, `KIRIKAE_NVN`, dumps de shader, textura, cbuf e
  render target;
- janela e apresentação: `KIRIKAE_HEADLESS`, `KIRIKAE_VSYNC`, `KIRIKAE_NO_VULKAN`,
  `KIRIKAE_FRAME_DUMP`, `KIRIKAE_PRESENT_DUMP`;
- áudio e vídeo: `KIRIKAE_MUTE`, `KIRIKAE_AUDIO_DUMP`, `KIRIKAE_TRACE_AUDIO`,
  `KIRIKAE_TRACE_VIDEO`, `KIRIKAE_VIDEO_DUMP`;
- sistema e input: `KIRIKAE_SAVES`, `KIRIKAE_NPAD`, `KIRIKAE_INPUT`, `KIRIKAE_STRICT_IPC`.

## 8. Ferramentas e documentação gerada

O diretório `tools/` contém scripts sem dependências complexas para desenvolvimento:

| Grupo | Arquivos e finalidade |
| --- | --- |
| Progresso | `progress.py`, `progress_scan.py`, `progress_render.py`: varrem serviços/SVCs/ioctls/shaders e geram o dashboard em `docs/progress/`. |
| Cobertura | `coverage_report.py`: transforma os arquivos de cobertura em relatórios por interface. |
| Empacotamento | `package_release.py`: compila CLI/GUI e cria o ZIP Windows ou tarball Linux. |
| SIMD/FP | `simd_oracle.py`: compara a semântica com um modelo Cortex-A57; `sample.py` e `sample_report.py` apoiam amostras e relatórios. |
| GPU | `gen_maxwell_opcodes.py`: gera a tabela Maxwell; `frame_diff.py` compara capturas de frames. |
| Discovery | `tools/discovery/`: coleta, agrupa e prioriza dados de descoberta, com exemplos JSON e testes Python. |
| Referências | `tools/reference/horizon.txt`: IDs e nomes públicos usados pelo relatório de progresso. |

`docs/progress/` é artefato gerado. O dashboard contém HTML, SVG, JSON, badges e histórico; ele
deve ser regenerado após mudanças de runtime, não atualizado manualmente em cada pull request.

## 9. Testes e qualidade

Os testes são distribuídos por responsabilidade:

- `crates/a64/tests`: encodings A32 e regressões SIMD;
- `crates/recomp/tests`: operações Cranelift, registradores rápidos e geração end-to-end;
- `crates/kirikae/tests`: fluxo end-to-end usando programas do testkit;
- `crates/maxwell/tests`: validação da tradução para SPIR-V;
- `crates/rt/tests`: execução do interpretador;
- módulos individuais: testes unitários próximos ao código;
- `tools/*test*.py`: testes dos scripts de discovery, frame diff e outras ferramentas.

Comandos recomendados para desenvolvimento:

```bash
cargo build --release --workspace
cargo test --release --workspace
cargo clippy --release --workspace --all-targets -- -D warnings
```

O projeto mantém a mesma expectativa no CI para Windows e Linux. Em Windows, caminhos longos
podem exigir `CARGO_TARGET_DIR` apontando para uma pasta curta.

## 10. CI/CD

### CI (`.github/workflows/ci.yml`)

Executa em cada push para `main` e em pull requests, usando uma matriz
`windows-latest`/`ubuntu-latest`:

1. instala headers ALSA/udev no Linux;
2. instala Rust stable e Clippy;
3. restaura cache do Cargo;
4. compila o workspace em release;
5. executa Clippy com warnings como erro;
6. executa os testes do workspace.

### Release (`.github/workflows/release.yml`)

Executa para tags `v*` ou manualmente. Os jobs Windows e Linux:

1. instalam Rust stable e Python 3.12;
2. instalam bibliotecas de sistema necessárias no Linux;
3. executam `python tools/package_release.py`;
4. publicam ZIP Windows x64 e tarball Linux x64 como artefatos.

Para tags, o job `publish` cria a release do GitHub com notas geradas. Cada pacote inclui
`kirikae`, `kirikae-gui` e `runtime/`.

## 11. Requisitos de execução e desenvolvimento

### Usuário final

- Windows 10/11 x64 ou Linux x64;
- GPU com Vulkan 1.3;
- Rust/Cargo para compilar o jogo na máquina local;
- Windows: MSVC Build Tools com desenvolvimento Desktop em C++;
- Linux: toolchain C, `pkg-config`, headers ALSA/udev e glibc 2.35 ou mais recente;
- dump próprio já descriptografado.

### Desenvolvedor

- Rust stable e Cargo;
- Python 3 para empacotamento, progresso e utilitários;
- acesso a Vulkan para executar a parte gráfica;
- nenhum dado de jogo é necessário para executar a suíte padrão, pois o `kirikae-testkit` gera
  entradas sintéticas.

## 12. Estado atual do projeto

O snapshot de progresso versionado foi gerado em 2026-10-28. Os totais abaixo são a cobertura
declarada pelas tabelas públicas usadas pelo projeto, não uma medida de compatibilidade de todos os
jogos:

| Área | Implementado | Total | Cobertura |
| --- | ---: | ---: | ---: |
| Serviços Horizon | 417 | 1302 | 32,0% |
| SVCs do kernel | 45 | 124 | 36,3% |
| Ioctls do driver GPU | 58 | 138 | 42,0% |
| Instruções Maxwell | 79 | 146 | 54,1% |

O README informa que jogos comerciais completos já funcionam, incluindo menus, vídeos, áudio e
gameplay, mas a compatibilidade ainda depende dos serviços e instruções realmente usados por cada
jogo. A direção técnica destacada pelo projeto é ampliar a implementação de NVN no host sobre
Vulkan para reduzir o custo do processador de comandos.

## 13. Convenções para contribuição

- mantenha um conceito por arquivo e prefira nomes alinhados ao modelo (`vic.rs`, `arbiter.rs`,
  `hid.rs`);
- divida arquivos que se aproximarem de 1000 linhas;
- use `mod.rs` para módulos organizados em diretórios;
- adicione novos serviços em `crates/rt/src/services/`, um arquivo por família;
- mantenha pull requests focados;
- inicie commits com `feat:`, `fix:` ou `chore:`;
- preserve comentários apenas quando necessários, incluindo notas `// SAFETY:` para `unsafe`;
- não inclua `docs/progress` em mudanças comuns;
- descreva os testes executados no pull request.

## 14. Mapa rápido para navegação

| Necessidade | Arquivo ou diretório |
| --- | --- |
| Entender o desenho completo | [ARCHITECTURE.md](ARCHITECTURE.md) |
| Instalar, usar e configurar | [README.md](README.md) |
| Regras legais e fontes permitidas | [LEGAL.md](LEGAL.md) |
| Como contribuir | [CONTRIBUTING.md](CONTRIBUTING.md) |
| CLI | [crates/kirikae/src/main.rs](crates/kirikae/src/main.rs) |
| Entrada e formatos | [crates/loader/src/](crates/loader/src/) |
| Decoder e IR ARM64 | [crates/a64/src/](crates/a64/src/) |
| Descoberta e geração | [crates/recomp/src/](crates/recomp/src/) |
| Runtime e serviços | [crates/rt/src/](crates/rt/src/) |
| GPU e Vulkan | [crates/rt/src/gpu/](crates/rt/src/gpu/) |
| Maxwell/SPIR-V | [crates/maxwell/src/](crates/maxwell/src/) |
| Testes sintéticos | [crates/testkit/src/lib.rs](crates/testkit/src/lib.rs) |
| Métricas de cobertura | [docs/progress/index.html](docs/progress/index.html) |
| Automação | [.github/workflows/](.github/workflows/) |
