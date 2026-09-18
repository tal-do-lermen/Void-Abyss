# Arquitetura Revisada — Gerenciador Universal de Pacotes

## 1. Objetivo

O projeto não deve ser um gerenciador ligado ao Void Linux, XBPS, AUR ou qualquer outra distribuição.

Ele deve ser um **gerenciador universal de receitas, build e pacotes**, com:

- núcleo independente de distribuição;
- formato binário próprio;
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
             *.pkgx
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
│                  Package Format (*.pkgx)                     │
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
      Package *.pkgx
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
└── pkgmgr/
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

# 10. Package Format próprio

O gerenciador deve gerar seu próprio formato:

```text
foo-1.2.3-x86_64.pkgx
```

Estrutura conceitual:

```text
PKGX
├── header
├── metadata
├── manifest
├── dependency information
├── optional signature
└── compressed payload
```

Payload:

```text
tar + zstd
```

O pacote pode ser um único arquivo.

Não criar:

```text
foo/
foo-build/
foo-files/
foo-metadata/
```

como estruturas permanentes.

---

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

Separar:

```text
package dependencies
```

de:

```text
host capabilities
```

Exemplo:

```text
foo
 ├── depends: bar
 ├── depends: libpng
 └── requires: libc
```

`bar` pode ser um pacote administrado pelo projeto.

`libc` pode ser uma capacidade fornecida pelo sistema.

O resolver deve primeiro verificar:

```text
pacote próprio
        ↓
pacote instalado
        ↓
capacidade do host
```

e somente depois decidir se é necessário instalar/buildar alguma dependência.

---

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
~/.config/pkgmgr/
└── config.toml

~/.local/state/pkgmgr/
├── state.db
├── packages/
│   └── <nome>.toml
└── logs/

~/.cache/pkgmgr/
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

Usar uma única base pequena:

```text
state.db
```

Guardar apenas:

```text
package
version
source
recipe revision
installed version
build fingerprint
ABI
status
timestamps
```

Não armazenar:

```text
source contents
Git objects
build logs completos
package payload
```

Para minimizar arquivos auxiliares, evitar WAL persistente por padrão.

Logs detalhados devem ser temporários ou limitados a um tamanho pequeno.

---

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
pkgmgr check foo
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
.pkgx
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
pkgmgr
```

de:

```text
pkgmgr-helper
```

### `pkgmgr`

Processo normal, sem privilégios:

```text
CLI
parser
resolver
build
package
verification
```

### `pkgmgr-helper`

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
.pkgx
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
sudo pkgmgr ...
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
pkgmgr install foo --user
```

não exige helper.

---

## Fluxo completo

```text
                    usuário normal
                         │
                         ▼
                    pkgmgr CLI
                         │
          ┌──────────────┼──────────────┐
          │              │              │
        parser         resolver        build
          │              │              │
          └──────────────┼──────────────┘
                         │
                         ▼
                       .pkgx
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
                           pkgmgr-helper
                                 │
                                 ▼
                         instalar/remover
```

O `pkgmgr` principal nunca deve ser reiniciado como root para concluir a operação.

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
   └── sim → pkgmgr-helper
                 ↓
            remover arquivos
                 ↓
            atualizar state.db
```

O `pkgmgr` nunca executa a remoção diretamente como `root`.

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
pkgmgr/
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
pkgmgr add <source>
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
pkgmgr update foo
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
*.pkgx
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
pkgmgr add <git-url>
pkgmgr import <aur|void|gentoo|file>
pkgmgr inspect <package>
pkgmgr check <package>
pkgmgr update <package>
pkgmgr build <package>
pkgmgr install <package>
pkgmgr remove <package>
pkgmgr upgrade
pkgmgr search <query>
pkgmgr list
pkgmgr info <package>
```

Operações relacionadas a armazenamento:

```text
pkgmgr cache status
pkgmgr cache clean
pkgmgr cache disable
pkgmgr package keep
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
Git → PackageSpec → build → .pkgx → install
```

Regra desde a primeira versão:

```text
pkgmgr nunca roda como root
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
remote *.pkgx
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
pkgmgr nunca roda como root
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
    *.pkgx
       │
       ▼
 Install
```

O resultado é um gerenciador próprio, com formato próprio, que pode usar receitas existentes sem transformar Void, AUR ou Gentoo em dependências estruturais do projeto.
