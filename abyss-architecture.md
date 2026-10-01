# Arquitetura Revisada — Abyss: Gerenciador Universal de Pacotes

## 1. Objetivo

O projeto não deve ser um gerenciador ligado ao Void Linux, XBPS, AUR ou qualquer outra distribuição.

Ele deve ser um **gerenciador universal de receitas, build e pacotes**, com:

- núcleo independente de distribuição;
- formato de pacote binário `.abyss`;
- importação de receitas Git, AUR, Void e Gentoo;
- build local usando as ferramentas nativas do projeto;
- instalação e atualização controladas pelo próprio gerenciador;
- armazenamento persistente mínimo;
- nenhuma clonagem permanente de repositórios;
- nenhuma cópia permanente dos sources;
- cache persistente de **1 GiB por padrão**, configurável para aumentar ou desligar;
- nenhum cache de build obrigatório;
- execução rápida para operações que não exigem compilação;
- possibilidade de usar pacotes binários pré-compilados quando houver compatibilidade;
- execução do gerenciador sempre como **usuário sem privilégios**;
- builds sempre sem `root`/`sudo`;
- instalação sistêmica, quando necessária, feita por um componente privilegiado mínimo e separado do gerenciador.

O projeto deve ser determinístico e modular. Não é necessário colocar um agente/LLM em cada etapa.

---

## 2. Mudança fundamental em relação à arquitetura antiga

A arquitetura antiga estava fortemente ligada ao XBPS:

```text
template
   ↓
xbps-src
   ↓
.xbps
```

e mantinha estruturas como:

```text
cache/git/
void-packages/
packages/
```

Isso aumenta armazenamento e cria acoplamento ao Void.

A nova arquitetura deve ser:

```text
Git / AUR / Void / Gentoo / URL
                │
                ▼
        Source/Recipe Adapters
                │
                ▼
          PackageSpec (IR)
                │
       ┌────────┴────────┐
       │                 │
       ▼                 ▼
    Build Engine      Package Format
       │                 │
       └────────┬────────┘
                ▼
             *.abyss
                │
                ▼
       Package Database
                │
                ▼
        Install / Update
```

O **PackageSpec** é o centro do projeto.

XBPS, DEB, RPM e outros formatos não são dependências do núcleo.

---

# 3. Princípios de projeto

## 3.1 Não armazenar o que pode ser consultado novamente

Não manter permanentemente:

```text
Git clone completo
AUR clone completo
Void clone completo
Gentoo clone completo
build workspace
masterdir equivalente
cópias duplicadas de sources
```

O projeto deve consultar as fontes quando necessário.

A única exceção é o **cache persistente controlado**, limitado por padrão a **1 GiB**.

Persistir somente:

```text
configuração
estado
receitas normalizadas
manifestos dos pacotes instalados
metadados mínimos
cache limitado
```

Regra do cache:

```text
padrão = 1 GiB
mínimo = 0 / desligado
máximo = definido pelo usuário
```

O limite deve ser global para o cache do gerenciador e aplicado por uma política LRU/uso mais antigo.

O cache não deve conter diretórios de build completos.

---

## 3.2 Nenhuma duplicação do source

Durante um build deve existir, idealmente:

```text
source/
build/
pkgroot/
```

e não:

```text
download/
source-copy/
build-copy/
package-copy/
```

Quando o build system permitir, o source deve ser usado diretamente.

Para builds out-of-tree:

```text
source/  → somente fonte
build/   → artefatos intermediários
pkgroot/ → arquivos finais
```

Após a geração do pacote:

```text
source/  → removido
build/   → removido
pkgroot/ → removido
```

O pacote final é opcionalmente mantido.

Por padrão:

```text
build → package → install → cleanup
```

---

# 4. Arquitetura geral

```text
┌─────────────────────────────────────────────────────────────┐
│                    PACKAGE MANAGER CORE                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Package Registry                                            │
│  Dependency Resolver                                         │
│  Build Planner                                               │
│  Installer                                                   │
│  Package Database                                            │
│                                                             │
├───────────────────────┬─────────────────────────────────────┤
│                       │                                     │
│   Recipe / Source     │         Host / Platform             │
│        Adapters       │             Adapter                 │
│                       │                                     │
│ Git                   │ arch                                │
│ AUR                   │ libc                                │
│ Void                  │ kernel                              │
│ Gentoo                │ service manager                     │
│ URL/archive           │ native capabilities                 │
│                       │ native package information           │
├───────────────────────┴─────────────────────────────────────┤
│                                                             │
│                    PackageSpec / IR                          │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                     Build Engine                             │
│                                                             │
│  configure → compile → test → stage → package               │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                Abyss Package Format (*.abyss)                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

# 5. PackageSpec

Todas as entradas são convertidas para uma representação intermediária comum.

Exemplo conceitual:

```text
PackageSpec
├── identity
│   ├── name
│   ├── version
│   └── epoch
│
├── source
│   ├── type
│   ├── url
│   ├── revision/tag
│   ├── checksum
│   └── source files
│
├── metadata
│   ├── description
│   ├── homepage
│   ├── license
│   └── maintainer
│
├── dependencies
│   ├── runtime
│   ├── build
│   ├── check
│   └── optional
│
├── build
│   ├── build system
│   ├── options
│   ├── environment
│   ├── patches
│   ├── configure
│   ├── compile
│   ├── test
│   └── install
│
├── files
│   ├── extra files
│   └── install scripts
│
└── provenance
    ├── source type
    ├── source URL
    ├── imported recipe
    └── recipe revision
```

O PackageSpec não deve conter conceitos exclusivos de XBPS.

Por exemplo, evitar:

```text
xbps-src
masterdir
hostdir
template
```

no núcleo.

Esses conceitos pertencem a adapters específicos, caso sejam necessários.

---

# 6. Fontes de receitas

## 6.1 Git upstream

Git deve ser tratado como fonte de código e, quando possível, de metadados.

Para descobrir atualizações:

```text
git remote metadata
      ↓
tags / heads / commits
```

Não clonar o histórico inteiro.

Quando for necessário construir:

```text
tag/release
     ↓
source archive
     ↓
build workspace temporário
```

Caso seja necessário acessar Git diretamente:

```text
clone raso / checkout mínimo
```

somente dentro do workspace temporário.

O clone não deve virar cache permanente.

---

## 6.2 AUR

O AUR deve ser tratado como **fonte de receita**.

Fluxo:

```text
PKGBUILD
   ↓
AUR parser
   ↓
PackageSpec
```

Não executar o `PKGBUILD` como mecanismo de parsing.

Receitas declarativas podem ser convertidas automaticamente.

Trechos arbitrários de shell devem gerar:

```text
manual review
```

quando não houver tradução segura para o PackageSpec.

---

## 6.3 Void

O template Void é apenas outro formato de receita.

Fluxo:

```text
template
   ↓
Void parser
   ↓
PackageSpec
```

Não é necessário manter um checkout do `void-packages`.

Somente o template necessário deve ser obtido.

---

## 6.4 Gentoo

O ebuild é tratado como receita:

```text
ebuild
   ↓
Gentoo parser
   ↓
PackageSpec
```

Assim como no AUR, operações arbitrárias podem exigir análise manual em vez de execução automática.

---

## 6.5 Git + receita

Uma receita pode usar um upstream Git:

```text
recipe
   │
   └── source → Git
                ↓
             version
             tag
             commit
```

Isso permite separar:

```text
onde o software está
```

de:

```text
como o software deve ser compilado
```

---

# 7. Não criar conversores N × M

Evitar:

```text
AUR → XBPS
AUR → DEB
AUR → RPM

Void → XBPS
Void → DEB
Void → RPM

Gentoo → XBPS
...
```

Usar:

```text
AUR ──────┐
Void ─────┤
Gentoo ───┤
Git ──────┤
URL ──────┘
           ↓
      PackageSpec
           ↓
      Build Engine
           ↓
      Pacote *.abyss
```

Isso reduz drasticamente a complexidade do projeto.

---

# 8. Build Engine

O Build Engine é próprio do projeto.

Ele não deve depender de `xbps-src`, `makepkg`, `ebuild`, `dpkg-buildpackage` ou `rpmbuild`.

Ele deve apenas executar os sistemas de build necessários:

```text
Autotools
CMake
Meson
Ninja
Make
Cargo
Go
Python
Java
Rust
...
```

O compilador continua sendo o compilador nativo do ambiente:

```text
gcc
clang
rustc
go
javac
...
```

O gerenciador não deve guardar toolchains completos por padrão.

---

# 9. Workspace temporário

O build deve usar um único workspace por tarefa:

```text
/tmp ou /var/tmp
└── abyss/
    └── build-<id>/
        ├── source/
        ├── build/
        └── pkgroot/
```

Ao terminar:

```text
build-<id>/
```

é removido.

O diretório pode ser configurável:

```text
TMPDIR
```

Para máquinas com muita RAM, um usuário pode escolher tmpfs.

Para máquinas com pouca RAM, usar armazenamento normal.

A escolha não deve ser obrigatória pelo projeto.

---

# 10. Abyss Package Format (*.abyss)

O Abyss utiliza um formato binário próprio para representar o pacote instalável:

```text
foo-1.2.3-x86_64.abyss
```

O arquivo `.abyss` é um contêiner de pacote e não deve ser apenas um `tar.zst` renomeado. Ele possui metadados estruturados, manifest, informações de integridade/assinatura e um payload comprimido.

## 10.1 Estrutura conceitual

```text
┌────────────────────────────────────────────┐
│ Header ABYS                                │
│ versão / flags / offsets / tamanhos        │
├────────────────────────────────────────────┤
│ Metadata                                   │
│ identidade / versão / ABI / dependências   │
├────────────────────────────────────────────┤
│ Manifest                                   │
│ arquivos / permissões / checksums          │
├────────────────────────────────────────────┤
│ Signature / integridade                    │
├────────────────────────────────────────────┤
│ Payload                                    │
│ tar + zstd                                 │
└────────────────────────────────────────────┘
```

## 10.2 Header

O header deve permitir que o Abyss identifique e inspecione o pacote sem descompactar o payload.

Estrutura conceitual:

```text
magic             4 bytes   "ABYS"
format_version    2 bytes
flags             2 bytes
metadata_offset   u64
metadata_size     u64
manifest_offset   u64
manifest_size     u64
payload_offset    u64
payload_size      u64
```

O formato deve definir explicitamente:

```text
endianness
alinhamento
versão do formato
flags reservadas
```

As estruturas binárias devem permitir evolução futura sem quebrar leitores antigos de forma silenciosa.

## 10.3 Metadata

Os metadados podem ser serializados em um formato compacto e estruturado, preferencialmente CBOR.

Exemplo conceitual:

```text
name = foo
version = 1.2.3
epoch = 0
architecture = x86_64
abi = linux-x86_64-glibc
fingerprint = sha256:...
```

O metadata do `.abyss` deve conter somente o necessário para identificar, validar e instalar o pacote.

## 10.4 Manifest

O manifest registra os arquivos que o pacote instala:

```text
/usr/bin/foo
/usr/lib/libfoo.so
/usr/share/man/man1/foo.1
```

Cada entrada deve poder registrar, quando aplicável:

```text
path
type
mode
uid
gid
size
checksum
```

O manifest é utilizado para:

```text
remoção
verificação de integridade
upgrade
checagem de conflitos de arquivos
```

## 10.5 Payload

O payload contém os arquivos reais do pacote em `tar + zstd`:

```text
payload.tar.zst
```

Exemplo:

```text
usr/
├── bin/
│   └── foo
├── lib/
│   └── libfoo.so
└── share/
    └── ...
```

O gerador deve fazer streaming de `pkgroot -> tar -> zstd -> .abyss`, evitando carregar o pacote inteiro na RAM.

## 10.6 Leitura sem extração

Graças aos offsets do header, comandos como:

```bash
abyss inspect foo.abyss
abyss list foo.abyss
```

podem ler metadata e manifest sem extrair o payload.

## 10.7 Pacote final e workspace

Durante o build:

```text
source/
build/
pkgroot/
```

Após a geração:

```text
pkgroot/
   ↓
manifest + metadata
   ↓
foo-1.2.3-x86_64.abyss
   ↓
cleanup do workspace
```

O `.abyss` é o artefato final distribuível/instalável.

# 11. Metadata mínima

O pacote deve conter somente o necessário para instalação e manutenção.

```text
name
version
architecture
abi
dependencies
provides
conflicts
files
permissions
ownership
checksums
install/remove actions
signature
```

Não colocar dentro do pacote informações redundantes que possam ser obtidas do repositório.

---

# 12. ABI e portabilidade

"Funcionar em qualquer distribuição" não significa que o mesmo binário Linux será compatível com qualquer sistema.

O pacote precisa declarar seu alvo:

```text
architecture:
    x86_64

libc:
    glibc

abi:
    linux-x86_64-glibc

cpu:
    baseline
```

Exemplos:

```text
linux-x86_64-glibc
linux-x86_64-musl
linux-aarch64-glibc
linux-aarch64-musl
```

Quando um pacote binário não for compatível:

```text
prebuilt package
       ↓
incompatível
       ↓
rebuild local
```

Isso permite que o mesmo PackageSpec produza o pacote correto para o host.

---

# 13. Host Adapter

O núcleo não deve assumir que o host é Void.

O adapter detecta:

```text
architecture
kernel
libc
init/service manager
filesystem
compiler/toolchain
native libraries
```

Exemplo:

```text
HostAdapter
├── Arch
├── Fedora
├── Debian
├── Void
├── Alpine
└── generic Linux
```

A diferença entre as distribuições fica restrita ao adapter.

O PackageSpec continua igual.

---

# 14. Dependências

O Abyss deve separar três conceitos diferentes:

```text
1. dependência declarada
2. dependência resolvida
3. dependência instalada
```

## 14.1 Dependência declarada

O `PackageSpec` descreve as restrições definidas pela receita:

```text
foo
 ├── runtime: bar >= 2.0
 ├── runtime: libpng >= 1.6,<2.0
 ├── optional: ffmpeg
 └── build: cmake >= 3.28
```

A restrição deve preservar operador e versão, e não somente o nome do pacote.

Tipos mínimos:

```text
runtime
build
check
optional
```

## 14.2 Dependência resolvida

Após o resolver, a restrição abstrata é associada a um artefato concreto:

```text
foo 1.5
 ├── bar 2.4.1
 └── libpng 1.6.50
```

O Abyss não deve armazenar uma cópia completa da árvore de dependências dentro de cada pacote. Deve armazenar as relações diretas:

```text
foo  → bar
foo  → libpng
bar  → zlib
```

O grafo completo é reconstruído pelo resolver quando necessário.

## 14.3 Dependência instalada

O estado local registra qual build realmente foi instalado e qual build resolveu cada aresta:

```text
foo-1.5.0 fingerprint ABC
    ↓
bar-2.4.1 fingerprint DEF
```

Isso é diferente de registrar somente `bar >= 2.0`.

## 14.4 Provides / capabilities

Dependências não devem obrigatoriamente apontar para um nome de pacote. O pacote pode fornecer capacidades:

```text
mesa   → provides: libGL
nvidia → provides: libGL
```

O resolver deve conseguir resolver:

```text
foo
 ↓
requires: libGL
 ↓
capability provider
```

O mesmo mecanismo pode ser utilizado para capacidades do host, por exemplo:

```text
libc
libGL.so.1
/usr/bin/python3
```

A separação entre pacote e capacidade do host deve ser preservada.

## 14.5 Dependency groups

O modelo deve suportar alternativas (`OR`) por meio de grupos de dependência:

```text
(libcurl OR wget)
(gcc OR clang)
```

Internamente:

```text
group_id = 10
 ├── libcurl
 └── wget
```

Isso evita criar regras especiais no resolver para cada tipo de alternativa.

## 14.6 Reverse dependencies

As relações instaladas devem ser navegáveis nos dois sentidos:

```text
foo → bar
app → bar
```

Assim, ao remover `foo`, o Abyss pode detectar que `bar` ainda é usado por `app`.

Se `bar` não possuir nenhum consumidor restante, ele pode ser considerado órfão.

## 14.7 Build dependencies não são runtime dependencies

Dependências utilizadas somente para compilar o pacote não devem ser tratadas como dependências de execução:

```text
foo
 ├── runtime → openssl
 ├── runtime → zlib
 └── build   → cmake
                → ninja
                → clang
```

Isso permite remover posteriormente apenas dependências de build que não sejam mais utilizadas.

## 14.8 Ordem de resolução

O resolver deve consultar nesta ordem:

```text
dependência declarada
        ↓
pacote Abyss instalado
        ↓
capacidade fornecida por outro pacote
        ↓
capacidade do host
        ↓
resolver build/instalação necessária
```

O objetivo é evitar instalar ou reconstruir uma dependência que já esteja satisfeita pelo sistema ou por outro pacote.

# 15. Build fingerprint

Para evitar rebuilds desnecessários, cada build recebe uma impressão determinística:

```text
BuildFingerprint =
    hash(
        PackageSpec,
        source checksum,
        patches,
        build options,
        architecture,
        libc/ABI,
        compiler identity,
        relevant environment
    )
```

Estado:

```text
foo
└── fingerprint: ABC123
```

Se nada relevante mudou, não reconstruir dentro de uma operação já em andamento.

O fingerprint não implica manter o build inteiro no disco.

---

# 16. Política de armazenamento

## Persistente

A separação deve seguir o padrão XDG:

```text
~/.config/abyss/
└── config.toml

~/.local/state/abyss/
├── state.db
├── packages/
│   └── <nome>.toml
└── logs/

~/.cache/abyss/
└── cache/
```

### Limite padrão do cache

```text
1 GiB
```

O cache vem **ligado por padrão** para acelerar operações repetidas.

O usuário pode:

```text
desligar
aumentar
reduzir
limpar
```

a qualquer momento.

Exemplos conceituais:

```text
cache = 1 GiB
cache = 5 GiB
cache = 20 GiB
cache = disabled
```

O cache deve usar armazenamento endereçado por conteúdo quando apropriado e evitar duplicatas.

### O que pode entrar no cache

Preferencialmente:

```text
source archives
metadata de upstream
package artifacts pré-compilados
```

Não armazenar permanentemente:

```text
source extraído
build directory
pkgroot
objetos intermediários
masterdir
```

O cache deve usar política de descarte automática quando atingir o limite.

---

## Não persistente

Por padrão:

```text
source extraído
build
pkgroot
download temporário
logs detalhados
artefatos intermediários
```

devem ser temporários.

Depois de:

```text
build → package → install
```

o workspace deve ser removido, preservando apenas o que estiver dentro do cache permitido ou o pacote que o usuário explicitamente decidiu manter.

---

# 17. Banco de dados

Usar uma única base pequena em SQLite:

```text
state.db
```

O banco representa o **estado da máquina**, e não o conteúdo completo dos pacotes.

## 17.1 Tabelas principais

### packages

Identidade lógica do pacote:

```text
packages
├── id
└── name
```

### package_builds

Representa uma versão/artefato concreto:

```text
package_builds
├── id
├── package_id
├── version
├── epoch
├── architecture
├── abi
├── fingerprint
└── checksum
```

Um mesmo pacote pode possuir múltiplos builds para ABI, arquitetura ou fingerprint diferentes.

### dependencies

Guarda as dependências declaradas de cada build:

```text
dependencies
├── package_build_id
├── type
├── name
├── operator
├── version
└── group_id
```

Exemplo:

```text
foo 1.5
 ├── runtime | bar    | >= | 2.0
 ├── runtime | libpng | >= | 1.6
 └── build   | cmake  | >= | 3.28
```

### provides

Registra capacidades fornecidas por um build:

```text
provides
├── package_build_id
├── capability
└── version
```

### installed_packages

Registra os builds realmente instalados:

```text
installed_packages
├── id
├── package_build_id
└── installed_at
```

### installed_dependencies

Representa o grafo efetivamente resolvido na máquina:

```text
installed_dependencies
├── parent_install_id
└── child_install_id
```

Exemplo:

```text
foo-1.5 (ABC)
   ↓
bar-2.4.1 (DEF)
   ↓
zlib-1.3.1 (GHI)
```

Esse modelo evita armazenar a árvore completa repetidamente.

## 17.2 O que o state.db guarda

```text
package identity
package version
recipe/source reference
recipe revision
installed build
build fingerprint
ABI
dependency edges
provides
status
timestamps
```

Não armazenar no banco:

```text
source contents
Git objects
build directory
payload completo
logs completos
```

O pacote `.abyss` continua sendo a fonte do manifest e dos metadados do artefato quando estiver disponível.

## 17.3 Remoção e órfãos

Para remover `foo`:

```text
foo
 ↓
consultar installed_dependencies
 ↓
remover foo
 ↓
verificar consumidores de cada dependência
 ↓
manter dependências ainda usadas
 ↓
marcar/remover órfãs conforme política
```

O modelo permite implementar futuramente:

```text
abyss autoremove
```

sem precisar reconstruir a árvore a partir de todos os pacotes instalados.

# 18. Estratégia de cache

## Padrão: 1 GiB

```text
cache persistente
        │
        ▼
     máximo
      1 GiB
```

O objetivo do cache é acelerar operações repetidas sem permitir crescimento indefinido do consumo de disco.

Fluxo:

```text
consulta
   ↓
cache hit?
 ┌─┴───────┐
 │         │
sim       não
 │         │
 ▼         ▼
usar     download
cache       │
            ▼
          build
            │
            ▼
       guardar resultado
       se couber no limite
```

## Desligar

Quando o usuário não quiser cache:

```text
cache = disabled
```

Fluxo:

```text
consulta
 ↓
download
 ↓
build
 ↓
install
 ↓
delete
```

Nenhum artefato persistente deve permanecer por causa do cache.

## Aumentar

O limite pode ser alterado para qualquer valor adequado ao disco disponível:

```text
1 GiB
5 GiB
10 GiB
20 GiB
...
```

O gerenciador deve rejeitar valores inválidos e nunca ultrapassar o limite configurado.

## Cache de sessão

Durante uma única operação, arquivos necessários podem ser reutilizados entre etapas e dependências:

```text
source compartilhado
      ↓
vários builds dependentes
```

Esse cache de sessão é temporário mesmo quando o cache persistente está desligado.

Ao terminar:

```text
delete
```

## Evitar duplicação

Quando possível, utilizar conteúdo endereçado por hash:

```text
SHA-256(source)
      ↓
um único objeto no cache
```

Duas receitas que usam exatamente o mesmo arquivo não devem armazenar duas cópias.

---

# 19. Otimizações de velocidade

## 19.1 Descoberta paralela

Consultar simultaneamente:

```text
Git
AUR
Void
Gentoo
```

quando um pacote tiver múltiplas fontes.

---

## 19.2 Não baixar source durante check

Comando:

```text
abyss check foo
```

deve trabalhar somente com:

```text
metadata
tags
commits
recipe revision
checksums conhecidas
```

Não deve baixar o source completo.

---

## 19.3 Download somente no build

```text
check
   ↓
update required?
   │
   ├── não → terminou
   │
   └── sim
        ↓
      build
        ↓
      download source
```

---

## 19.4 Build paralelo

Dependências independentes:

```text
A ──┐
B ──┼──→ paralelo
C ──┘

D depende de A+B

A ──┐
B ──┘
 ↓
 D
```

O scheduler deve limitar jobs por:

```text
CPU
RAM
disco
```

e não apenas por número de cores.

---

## 19.5 Sem rebuild por pequenas mudanças de estado

Não reconstruir quando mudar apenas:

```text
metadata local
timestamp
logs
```

O fingerprint deve considerar somente dados que afetam o artefato.

---

# 20. Segurança e instalação

## Regra principal

**O gerenciador nunca deve ser executado com `sudo` e nunca deve exigir que o processo principal rode como `root`.**

Isto vale para:

```text
add
import
inspect
check
update
build
package
search
```

e também para a maior parte das operações de administração.

### Build sem privilégios

Todo o processo de build deve ocorrer como usuário normal:

```text
source
   ↓
build
   ↓
pkgroot
   ↓
.abyss
```

Nenhuma etapa de compilação deve precisar de:

```text
sudo
root
setuid
```

Isso reduz o impacto de:

- scripts de build maliciosos;
- receitas de terceiros comprometidas;
- falhas no parser;
- comandos executados pelo sistema de build.

---

## Instalação de sistema

Instalar em diretórios como:

```text
/usr
/bin
/lib
/etc
```

normalmente exige privilégio.

Por isso o projeto deve separar:

```text
abyss
```

de:

```text
abyss-helper
```

### `abyss`

Processo normal, sem privilégios:

```text
CLI
parser
resolver
build
package
verification
```

### `abyss-helper`

Componente mínimo e privilegiado, acionado somente quando uma operação realmente exige acesso ao sistema.

Ele **não deve**:

```text
executar receitas
compilar software
executar shell arbitrário
receber comandos genéricos
rodar o gerenciador inteiro como root
```

Ele deve receber somente operações estruturadas, por exemplo:

```text
install package artifact
remove package
replace package files
update package database
```

Antes da instalação:

```text
.abyss
  ↓
verificar assinatura
  ↓
verificar integridade
  ↓
verificar manifest
  ↓
resolver dependências
  ↓
autorização do helper
  ↓
instalação
  ↓
registrar estado
```

O componente privilegiado deve operar sobre o artefato já construído e validado, e não sobre a árvore fonte.

### Autorização

A autorização deve ser feita por um mecanismo apropriado ao host, sem exigir:

```text
sudo abyss ...
```

Quando houver suporte no sistema, pode ser utilizada uma camada de autorização como PolicyKit/polkit.

Em hosts sem essa infraestrutura, o projeto deve permitir um mecanismo equivalente de autorização, mantendo a mesma regra:

```text
somente o helper recebe privilégios
o processo principal permanece sem privilégios
```

---

## Instalação por usuário

Sempre que possível, o gerenciador deve permitir uma instalação sem qualquer privilégio:

```text
$HOME/.local/bin
$HOME/.local/lib
$HOME/.local/share
```

Nesse modo:

```text
abyss install foo --user
```

não exige helper.

---

## Fluxo completo

```text
                    usuário normal
                         │
                         ▼
                    abyss CLI
                         │
          ┌──────────────┼──────────────┐
          │              │              │
        parser         resolver        build
          │              │              │
          └──────────────┼──────────────┘
                         │
                         ▼
                       .abyss
                         │
                    verificação
                         │
                precisa root?
                  ┌──────┴──────┐
                  │             │
                 não           sim
                  │             │
                  ▼             ▼
              instalar      autorização
              como user          │
                                 ▼
                           abyss-helper
                                 │
                                 ▼
                         instalar/remover
```

O `abyss` principal nunca deve ser reiniciado como root para concluir a operação.

---

# 21. Remoção

O manifest do pacote contém:

```text
file list
```

Assim:

```text
remove foo
    ↓
consultar manifest
    ↓
precisa acesso sistêmico?
    │
   ├── não → remover como usuário
   │
   └── sim → abyss-helper
                 ↓
            remover arquivos
                 ↓
            atualizar state.db
```

O `abyss` nunca executa a remoção diretamente como `root`.

Não é necessário manter uma cópia do pacote para saber o que foi instalado.

---

# 22. Rollback

Rollback completo consome espaço.

Por isso:

```text
rollback = opcional
```

Modo mínimo:

```text
sem snapshots
sem cópias completas
sem backups permanentes
```

Modo avançado:

```text
cache de pacotes
ou
filesystem snapshots
```

quando disponível.

---

# 23. Serviços

Não assumir systemd.

Criar uma camada:

```text
ServiceAdapter
├── systemd
├── runit
├── OpenRC
├── s6
└── none
```

O pacote declara a intenção:

```text
service:
    name: foo
    enable: true
    start: true
```

O adapter decide como realizar isso no host.

---

# 24. Estrutura final do projeto

```text
abyss/
├── core/
│   ├── package-spec
│   ├── dependency-resolver
│   ├── build-planner
│   ├── installer
│   └── database
│
├── sources/
│   ├── git
│   ├── aur
│   ├── void
│   ├── gentoo
│   └── archive
│
├── parsers/
│   ├── pkgbUILD
│   ├── void-template
│   ├── ebuild
│   └── generic
│
├── build/
│   ├── scheduler
│   ├── workspace
│   ├── sandbox
│   └── build-systems
│
├── package/
│   ├── pkgx
│   ├── compression
│   ├── manifest
│   └── signature
│
├── platform/
│   ├── linux
│   ├── libc
│   ├── services
│   └── native-packages
│
└── cli/
```

---

# 25. Fluxo completo

```text
abyss add <source>
        │
        ▼
detect source
        │
        ▼
parse recipe / source
        │
        ▼
PackageSpec
        │
        ▼
validate
        │
        ▼
state.db
```

Atualização:

```text
abyss update foo
        │
        ▼
consulta metadata
        │
        ▼
nova versão?
        │
   ┌────┴────┐
   │         │
  não       sim
   │         │
   ▼         ▼
 fim      PackageSpec
             │
             ▼
          build
```

Build:

```text
PackageSpec
     │
     ▼
Build Planner
     │
     ▼
Workspace temporário
     │
     ├── source
     ├── build
     └── pkgroot
     │
     ▼
compilar
     │
     ▼
manifest
     │
     ▼
*.abyss
     │
     ▼
install
     │
     ▼
cleanup
```

---

# 26. Interface

```text
abyss add <git-url>
abyss import <aur|void|gentoo|file>
abyss inspect <package>
abyss check <package>
abyss update <package>
abyss build <package>
abyss install <package>
abyss remove <package>
abyss upgrade
abyss search <query>
abyss list
abyss info <package>
```

Operações relacionadas a armazenamento:

```text
abyss cache status
abyss cache clean
abyss cache disable
abyss package keep
```

---

# 27. Prioridades de implementação

## Fase 1 — Core mínimo

Implementar primeiro:

```text
PackageSpec
state.db
CLI
Git source
generic build
pkgx format
user install/remove
privileged helper mínimo
```

Objetivo:

```text
Git → PackageSpec → build → .abyss → install
```

Regra desde a primeira versão:

```text
abyss nunca roda como root
build nunca roda como root
sudo não faz parte do fluxo
```

O helper privilegiado deve ser pequeno e isolado desde o início, em vez de colocar código de instalação privilegiado dentro do processo principal.

---

## Fase 2 — Receitas

Adicionar:

```text
AUR parser
Void parser
Gentoo parser
```

Fluxo:

```text
AUR/Void/Gentoo
      ↓
PackageSpec
      ↓
mesmo builder
```

---

## Fase 3 — Distro abstraction

Adicionar:

```text
HostAdapter
ServiceAdapter
native capability detection
```

---

## Fase 4 — Performance

Adicionar:

```text
parallel resolver
parallel downloads
parallel builds
incremental metadata
build fingerprints
session cache
content-addressed cache
LRU cache de até 1 GiB
```

O limite padrão do cache continua:

```text
1 GiB
```

e deve poder ser:

```text
disabled
```

ou aumentado pelo usuário.

---

## Fase 5 — Binary repository

Somente depois:

```text
remote *.abyss
```

Com suporte a:

```text
download
verify
install
cleanup
```

O cache binário local continua opcional.

---

# 28. Regras centrais do projeto

A arquitetura inteira deve seguir estas regras.

### 1. Disco

```text
Não armazenar permanentemente algo
que pode ser reconstruído ou consultado novamente
a baixo custo.
```

### 2. Cache

```text
1 GiB por padrão
0 = desligado
N GiB = limite escolhido pelo usuário
```

Nunca permitir que o cache cresça indefinidamente.

### 3. Segurança

```text
abyss nunca roda como root
build nunca roda como root
sudo não faz parte do fluxo
```

Acesso privilegiado deve ficar isolado em um helper mínimo e restrito.

### 4. Portabilidade

```text
Não depender de uma distribuição
para executar o núcleo do gerenciador.
```

Portanto:

```text
               CORE
                 │
          PackageSpec / IR
                 │
       ┌─────────┴─────────┐
       │                   │
     Sources             Host
       │                   │
 Git/AUR/Void/         Linux/ABI/
 Gentoo/URL            services
       │
       ▼
     Build
       │
       ▼
    *.abyss
       │
       ▼
 Install
```

O resultado é um gerenciador próprio, com formato próprio, que pode usar receitas existentes sem transformar Void, AUR ou Gentoo em dependências estruturais do projeto.