# Guia Completo: Gerenciador de Pacotes Baseado em Templates do XBPS

## Índice

1. [Introdução e Objetivos](#1-introdução-e-objetivos)
2. [Arquitetura Geral](#2-arquitetura-geral)
3. [Estrutura de Diretórios](#3-estrutura-de-diretórios)
4. [Banco de Dados do Gerenciador](#4-banco-de-dados-do-gerenciador)
5. [Gerenciamento do Cache Git](#5-gerenciamento-do-cache-git)
6. [Descoberta de Novas Versões](#6-descoberta-de-novas-versões)
7. [Atualização Automática dos Templates](#7-atualização-automática-dos-templates)
8. [Integração Detalhada com o xbps-src](#8-integração-detalhada-com-o-xbps-src)
9. [Processo Completo de Build](#9-processo-completo-de-build)
10. [Criação e Atualização do Repositório XBPS](#10-criação-e-atualização-do-repositório-xbps)
11. [Instalação e Atualização de Pacotes](#11-instalação-e-atualização-de-pacotes)
12. [Fluxos Completos (Diagramas)](#12-fluxos-completos-diagramas)
13. [Implementação Sugerida (Bash)](#13-implementação-sugerida-bash)
14. [Melhorias Futuras](#14-melhorias-futuras)

---

## 1. Introdução e Objetivos

### O que é o XBPS?

O **XBPS (X Binary Package System)** é o gerenciador de pacotes nativo do Void Linux, projetado e implementado do zero. Ele é rápido, seguro e portável. O XBPS é composto por um conjunto de ferramentas:

| Ferramenta | Função |
|------------|--------|
| `xbps-install` | Instala e atualiza pacotes |
| `xbps-remove` | Remove pacotes |
| `xbps-query` | Consulta repositórios e pacotes instalados |
| `xbps-reconfigure` | Reconfigura pacotes instalados |
| `xbps-pkgdb` | Verifica e corrige erros no banco de dados |
| `xbps-rindex` | Gerencia repositórios binários locais |
| `xbps-alternatives` | Gerencia alternativas de pacotes |
| `xbps-create` | Cria pacotes binários |
| `xbps-checkvers` | Verifica versões desatualizadas |
| `xbps-fbulk` | Build em lote paralelo |

### O que é o xbps-src?

O **xbps-src** é o construtor de pacotes fonte do Void Linux. Ele busca templates, compila pacotes em um chroot isolado (usando Linux namespaces) e gera pacotes binários `.xbps`. **Não requer root para funcionar**.

**Características principais:**
- Build reproduzível via templates
- Detecção automática de SONAME para dependências runtime
- Chroot isolado usando namespaces (sem root)
- Cross-compilation nativo
- Suporte a glibc e musl

### Objetivos deste Guia

Este guia descreve uma arquitetura para um gerenciador de pacotes que:

- Utiliza o formato de templates do xbps-src
- Automatiza obtenção, atualização e compilação a partir de repositórios upstream
- Minimiza tráfego de rede usando Git (evita limites da API REST)
- Permite atualizações rápidas e incrementais
- Mantém compatibilidade total com o ecossistema XBPS

---

## 2. Arquitetura Geral

### Visão de Alto Nível

```
┌─────────────────────────────────────────────────────────────────┐
│                    Gerenciador de Pacotes                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │  Templates  │    │  Cache Git  │    │   Banco de  │         │
│  │  (locais)   │    │  (mirrors)  │    │   Dados     │         │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘         │
│         │                  │                  │                 │
│         └──────────────────┼──────────────────┘                 │
│                            │                                    │
│                            ▼                                    │
│                   ┌─────────────────┐                           │
│                   │   xbps-src      │                           │
│                   │   (backend)     │                           │
│                   └────────┬────────┘                           │
│                            │                                    │
│                            ▼                                    │
│                   ┌─────────────────┐                           │
│                   │  Pacotes XBPS   │                           │
│                   │  (.xbps files)  │                           │
│                   └─────────────────┘                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Componentes Principais

| Componente | Descrição |
|------------|-----------|
| **Templates** | Arquivos que definem como baixar, compilar e instalar um pacote |
| **Cache Git** | Mirrors locais dos repositórios upstream (clone --mirror) |
| **Banco de Dados** | Metadados de sincronização (versão, commit, tag, última atualização) |
| **xbps-src** | Backend de compilação (não é substituído, apenas orquestrado) |
| **Repositório XBPS** | Diretório com pacotes `.xbps` e índice `repodata` |

---

## 3. Estrutura de Diretórios

### Estrutura Completa

```
repository/
├── srcpkgs/                    # Templates de pacotes
│   ├── neovim/
│   │   └── template
│   ├── swaylock-effects/
│   │   └── template
│   ├── foot/
│   │   └── template
│   └── ...
│
├── cache/
│   └── git/                    # Mirrors Git locais
│       ├── mortie/
│       │   └── swaylock-effects.git/
│       ├── neovim/
│       │   └── neovim.git/
│       ├── swaywm/
│       │   └── sway.git/
│       └── ...
│
├── packages/                   # Pacotes XBPS compilados
│   ├── neovim-0.10.0_1.x86_64.xbps
│   ├── swaylock-effects-1.7.0_1.x86_64.xbps
│   └── ...
│
├── database/
│   ├── packages.db             # Banco de dados principal
│   └── packages.db-shm         # Shared memory (SQLite)
│
├── void-packages/              # Clone do void-packages
│   ├── xbps-src
│   ├── srcpkgs/
│   ├── common/
│   └── ...
│
├── manager                     # Script principal do gerenciador
├── config                      # Configuração do gerenciador
└── logs/                       # Logs de operações
    ├── sync.log
    ├── build.log
    └── errors.log
```

### Estrutura de um Template

```
srcpkgs/
└── swaylock-effects/
    ├── template            # Template principal (obrigatório)
    ├── patches/            # Patches (opcional)
    │   └── fix-build.patch
    ├── files/              # Arquivos adicionais (opcional)
    │   └── swaylock.conf
    └── install/            # Scripts pós-instalação (opcional)
        ├── msg
        └── post
```

---

## 4. Banco de Dados do Gerenciador

### Schema Detalhado

#### Opção 1: Formato Simples (CSV)

```bash
# packages.db (formato simples)
# pkgname|url|branch|last_commit|last_tag|version|revision|build_style|last_sync|last_build|status

neovim|https://github.com/neovim/neovim|master|abc123def|v0.10.0|0.10.0|1|cmake|2024-01-15T10:00:00|2024-01-15T10:30:00|built
swaylock-effects|https://github.com/mortie/swaylock-effects|master|456abcde|v1.7.0|1.7.0|1|meson|2024-01-15T10:00:00|2024-01-15T10:45:00|built
foot|https://github.com/programmerjake4/foot|master|789f012|v1.17.0|1.17.0|1|meson|2024-01-15T10:00:00||pending
```

#### Opção 2: SQLite (Recomendado para muitos pacotes)

```sql
-- Tabela principal de pacotes
CREATE TABLE packages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    pkgname TEXT NOT NULL UNIQUE,
    url TEXT NOT NULL,
    branch TEXT DEFAULT 'master',
    last_commit TEXT,
    last_tag TEXT,
    version TEXT,
    revision INTEGER DEFAULT 1,
    build_style TEXT,
    last_sync TIMESTAMP,
    last_build TIMESTAMP,
    status TEXT DEFAULT 'pending'  -- pending, built, error
);

-- Tabela de dependências
CREATE TABLE dependencies (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    pkgname TEXT NOT NULL,
    dep_type TEXT NOT NULL,  -- hostmakedepends, makedepends, depends
    dep_name TEXT NOT NULL,
    FOREIGN KEY (pkgname) REFERENCES packages(pkgname)
);

-- Tabela de histórico de builds
CREATE TABLE build_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    pkgname TEXT NOT NULL,
    version TEXT NOT NULL,
    build_date TIMESTAMP,
    status TEXT,  -- success, error
    log_path TEXT,
    FOREIGN KEY (pkgname) REFERENCES packages(pkgname)
);

-- Índices para performance
CREATE INDEX idx_packages_pkgname ON packages(pkgname);
CREATE INDEX idx_packages_status ON packages(status);
CREATE INDEX idx_dependencies_pkgname ON dependencies(pkgname);
```

### Operações no Banco de Dados

```bash
# Consultar pacote
grep "^neovim|" packages.db

# Atualizar pacote
sed -i "s|^neovim|.*|neovim|https://github.com/neovim/neovim|master|newcommit|v0.10.1|0.10.1|1|cmake|$(date -Iseconds)|$(date -Iseconds)|built|" packages.db

# Adicionar pacote
echo "novopacote|https://github.com/user/repo|master|||1.0|1|meson|||" >> packages.db

# Listar pacotes com erro
grep "|error$" packages.db

# Listar pacotes pendentes
grep "|pending$" packages.db

# Contar pacotes
wc -l packages.db
```

---

## 5. Gerenciamento do Cache Git

### Por que usar Git ao invés da API REST?

| Método | Limite documentado | Ideal para |
|--------|-------------------|------------|
| API REST | 60/h (sem token) / 5000/h (com token) | Metadados, releases |
| Raw GitHub | Sem limite público | Arquivos individuais |
| Tarball | Sem limite público | Download de versões |
| **git fetch** | **Sem limite público** | **Sincronização contínua** |
| git ls-remote | Sem limite público | Verificar HEAD/tags |

### Por que Git é mais eficiente?

1. **Delta transfers**: Git baixa apenas objetos alterados, não o repositório inteiro
2. **Compressão**: Objetos são comprimidos no protocolo
3. **Deduplicação**: Objetos idênticos não são baixados novamente
4. **Sem autenticação**: Repositórios públicos não precisam de token
5. **Otimizado para sincronização**: O GitHub espera uso intensivo de git

### Estrutura do Cache

```
cache/
└── git/
    ├── mortie/
    │   └── swaylock-effects.git/    # Mirror bare
    │       ├── HEAD
    │       ├── config
    │       ├── refs/
    │       │   ├── heads/
    │       │   └── tags/
    │       ├── objects/
    │       └── ...
    ├── neovim/
    │   └── neovim.git/
    └── ...
```

### Comandos Essenciais

#### Criar Mirror

```bash
# Criar mirror completo (bare clone)
git clone --mirror https://github.com/mortie/swaylock-effects.git \
    cache/git/mortie/swaylock-effects.git

# Para repositórios grandes, usar shallow (pode limitar funcionalidade)
git clone --mirror --depth 1 https://github.com/neovim/neovim.git \
    cache/git/neovim/neovim.git
```

#### Atualizar Mirror

```bash
# Buscar atualizações (apenas objetos novos)
git -C cache/git/mortie/swaylock-effects.git fetch --prune

# Verificar se há novas tags
git -C cache/git/mortie/swaylock-effects.git tag -l

# Verificar último commit
git -C cache/git/mortie/swaylock-effects.git rev-parse HEAD

# Listar tags ordenadas por versão
git -C cache/git/mortie/swaylock-effects.git tag -l "v*" --sort=-version:refname | head -1
```

#### Trabalhar com Tags

```bash
# Listar todas as tags
git -C cache/git/mortie/swaylock-effects.git tag -l

# Listar tags com info de data
git -C cache/git/mortie/swaylock-effects.git tag -l --format='%(creatordate:short) %(refname:short)'

# Obter hash de uma tag específica
git -C cache/git/mortie/swaylock-effects.git rev-parse v1.7.0

# Comparar tags
git -C cache/git/mortie/swaylock-effects.git log v1.6.0..v1.7.0 --oneline

# Obter data da tag
git -C cache/git/mortie/swaylock-effects.git log -1 --format=%ai v1.7.0
```

#### Usar Worktree (Alternativa ao Checkout)

```bash
# Criar worktree para teste (sem afetar o mirror)
git -C cache/git/mortie/swaylock-effects.git worktree add \
    /tmp/swaylock-build v1.7.0

# Listar worktrees
git -C cache/git/mortie/swaylock-effects.git worktree list

# Remover worktree após uso
git -C cache/git/mortie/swaylock-effects.git worktree remove /tmp/swaylock-build

# Limpar worktrees órfãos
git -C cache/git/mortie/swaylock-effects.git worktree prune
```

#### Baixar Arquivos Específicos (Sem Clone)

```bash
# Listar arquivos na tag
git -C cache/git/mortie/swaylock-effects.git archive v1.7.0 | tar -t

# Extrair arquivo específico
git -C cache/git/mortie/swaylock-effects.git archive v1.7.0 meson.build | tar -xO > /tmp/meson.build

# Extrair diretório inteiro
git -C cache/git/mortie/swaylock-effects.git archive v1.7.0 | tar -x -C /tmp/extract-dir/
```

### Limites e Boas Práticas

```bash
# Tamanho típico de um mirror bare
# Repositório pequeno (~10MB source): ~2-5MB bare
# Repositório médio (~100MB source): ~20-50MB bare
# Repositório grande (linux kernel): ~500MB-2GB bare

# Recomendação: manter mirrors para ~100-500 pacotes
# Isso consome ~1-5GB de espaço e é totalmente viável

# Para verificar tamanho do cache
du -sh cache/git/

# Para limpar objetos não referenciados
git -C cache/git/mortie/swaylock-effects.git gc --prune=now
```

---

## 6. Descoberta de Novas Versões

### Estratégia de Detecção

```
Para cada pacote no banco de dados
        │
        ▼
┌───────────────────┐
│   git fetch       │
│   --prune --tags  │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  Tem tag semanal  │
│  ou rolling?      │
└─────────┬─────────┘
          │
    ┌─────┴─────┐
    │           │
  Tag        Rolling
    │           │
    ▼           ▼
┌─────────┐ ┌─────────┐
│ Comparar│ │ Comparar│
│ tags    │ │ commits │
└────┬────┘ └────┬────┘
     │           │
     └─────┬─────┘
           │
           ▼
┌───────────────────┐
│   Nova versão     │
│   encontrada?     │
└─────────┬─────────┘
          │
    ┌─────┴─────┐
    │           │
   Não         Sim
    │           │
    ▼           ▼
┌────────┐ ┌─────────────┐
│ Próximo│ │ Atualizar   │
│ pacote │ │ template    │
└────────┘ └─────────────┘
```

### Scripts de Detecção

#### Detectar Nova Tag

```bash
#!/bin/bash
# detect-new-version.sh

detect_new_version() {
    local pkgname="$1"
    local cache_dir="cache/git/${pkgname}.git"

    if [ ! -d "${cache_dir}" ]; then
        echo "ERRO: Cache não encontrado para ${pkgname}"
        return 1
    fi

    # Última tag conhecida
    local last_tag
    last_tag=$(grep "^${pkgname}|" database/packages.db | cut -d'|' -f5)

    # Buscar novas tags
    git -C "${cache_dir}" fetch --prune --tags 2>/dev/null

    # Última tag disponível (ordem: versão decrescente)
    local newest_tag
    newest_tag=$(git -C "${cache_dir}" tag -l "v*" --sort=-version:refname | head -1)

    # Comparar
    if [ -n "${newest_tag}" ] && [ "${newest_tag}" != "${last_tag}" ]; then
        local new_version="${new_version#v}"
        local old_version="${last_tag#v}"
        echo "NOVA_VERSAO=${new_version}"
        echo "ANTIGA_VERSAO=${old_version}"
        echo "TAG=${newest_tag}"
        return 0
    fi

    echo "SEM_ATUALIZACAO"
    return 1
}
```

#### Detectar Novo Commit (Rolling Release)

```bash
#!/bin/bash
# detect-new-commit.sh

detect_new_commit() {
    local pkgname="$1"
    local cache_dir="cache/git/${pkgname}.git"

    if [ ! -d "${cache_dir}" ]; then
        echo "ERRO: Cache não encontrado para ${pkgname}"
        return 1
    fi

    # Último commit conhecido
    local last_commit
    last_commit=$(grep "^${pkgname}|" database/packages.db | cut -d'|' -f4)

    # Buscar último commit
    git -C "${cache_dir}" fetch --prune 2>/dev/null
    local newest_commit
    newest_commit=$(git -C "${cache_dir}" rev-parse HEAD)

    if [ "${newest_commit}" != "${last_commit}" ]; then
        echo "NOVO_COMMIT=${newest_commit}"
        echo "ANTIGO_COMMIT=${last_commit}"
        echo "SHORT=$(echo ${newest_commit} | cut -c1-7)"
        return 0
    fi

    echo "SEM_ATUALIZACAO"
    return 1
}
```

#### Usando xbps-checkvers (Alternativa Oficial)

O xbps-src possui uma ferramenta integrada para verificar versões:

```bash
# Verificar pacotes desatualizados (do void-packages)
./xbps-src update-check

# Verificar um pacote específico
./xbps-src update-check neovim

# Saída esperada:
# neovim: 0.9.5 -> 0.10.0
```

---

## 7. Atualização Automática dos Templates

### Template Básico (Antes)

```bash
# Template file for 'swaylock-effects'
pkgname=swaylock-effects
version=1.6.0
revision=1

build_style=meson

hostmakedepends="
 pkg-config
 meson
 ninja
"

makedepends="
 wayland-devel
 cairo-devel
 pam-devel
 libxkbcommon-devel
 gdk-pixbuf-devel
"

short_desc="Screen locker for Wayland with blur effects"
maintainer="Your Name <email@example.com>"
license="MIT"
homepage="https://github.com/mortie/swaylock-effects"

distfiles="https://github.com/mortie/swaylock-effects/archive/v${version}.tar.gz"
checksum="abc123def456..."
```

### Processo de Atualização

```bash
#!/bin/bash
# update-template.sh

update_template() {
    local pkgname="$1"
    local new_version="$2"
    local template_path="srcpkgs/${pkgname}/template"

    if [ ! -f "${template_path}" ]; then
        echo "ERRO: Template não encontrado: ${template_path}"
        return 1
    fi

    # 1. Baixar novo tarball para calcular checksum
    local tarball_url="https://github.com/mortie/swaylock-effects/archive/v${new_version}.tar.gz"
    local tarball="/tmp/${pkgname}-${new_version}.tar.gz"

    echo "Baixando ${tarball_url}..."
    curl -sL "${tarball_url}" -o "${tarball}"

    if [ ! -f "${tarball}" ]; then
        echo "ERRO: Falha ao baixar tarball"
        return 1
    fi

    # 2. Calcular novo checksum
    local new_checksum
    new_checksum=$(sha256sum "${tarball}" | cut -d' ' -f1)

    # 3. Atualizar template
    sed -i "s/^version=.*/version=${new_version}/" "${template_path}"
    sed -i "s/^checksum=.*/checksum=\"${new_checksum}\"/" "${template_path}"

    # 4. Resetar revision para 1
    sed -i "s/^revision=.*/revision=1/" "${template_path}"

    # 5. Limpar tarball temporário
    rm -f "${tarball}"

    echo "Template atualizado: ${pkgname} -> ${new_version}"
    echo "Novo checksum: ${new_checksum}"
}
```

### Template Atualizado (Depois)

```bash
# Template file for 'swaylock-effects'
pkgname=swaylock-effects
version=1.7.0
revision=1

build_style=meson

hostmakedepends="
 pkg-config
 meson
 ninja
"

makedepends="
 wayland-devel
 cairo-devel
 pam-devel
 libxkbcommon-devel
 gdk-pixbuf-devel
"

short_desc="Screen locker for Wayland with blur effects"
maintainer="Your Name <email@example.com>"
license="MIT"
homepage="https://github.com/mortie/swaylock-effects"

distfiles="https://github.com/mortie/swaylock-effects/archive/v${version}.tar.gz"
checksum="def456abc789..."
```

### Para Rolling Releases (Sem Tags)

```bash
#!/bin/bash
# update-template-commit.sh

update_template_commit() {
    local pkgname="$1"
    local cache_dir="cache/git/${pkgname}.git"
    local template_path="srcpkgs/${pkgname}/template"
    local github_url="https://github.com/user/repo"

    # Obter novo commit
    local new_commit
    new_commit=$(git -C "${cache_dir}" rev-parse HEAD)
    local short_commit
    short_commit=$(echo "${new_commit}" | cut -c1-7)

    # Atualizar distfiles para usar commit
    sed -i "s|distfiles=.*|distfiles=\"${github_url}/archive/${new_commit}.tar.gz>src.tar.gz\"|" "${template_path}"

    # Calcular checksum do novo tarball
    local tarball_url="${github_url}/archive/${new_commit}.tar.gz"
    local new_checksum
    new_checksum=$(curl -sL "${tarball_url}" | sha256sum | cut -d' ' -f1)

    sed -i "s/^checksum=.*/checksum=\"${new_checksum}\"/" "${template_path}"

    echo "Template atualizado com commit ${short_commit}"
}
```

### Criar Template do Zero

```bash
#!/bin/bash
# create-template.sh

create_template() {
    local pkgname="$1"
    local github_url="$2"
    local build_style="${3:-meson}"  # Default: meson

    # Extrair owner/repo do URL
    local owner_repo="${github_url#https://github.com/}"
    local owner=$(echo "${owner_repo}" | cut -d'/' -f1)
    local repo=$(echo "${owner_repo}" | cut -d'/' -f2)

    # Criar diretório
    mkdir -p "srcpkgs/${pkgname}"

    # Criar template básico
    cat > "srcpkgs/${pkgname}/template" << EOF
# Template file for '${pkgname}'
pkgname=${pkgname}
version=0.0.0
revision=1

build_style=${build_style}

hostmakedepends="
 pkg-config
"

makedepends="
"

short_desc="TODO: Adicionar descrição curta"
maintainer="Your Name <email@example.com>"
license="TODO: Adicionar licença"
homepage="${github_url}"

distfiles="${github_url}/archive/v\${version}.tar.gz"
checksum="TODO"
EOF

    echo "Template criado em srcpkgs/${pkgname}/template"
    echo "Edite o template e defina: version, checksum, dependências"
}
```

---

## 8. Integração Detalhada com o xbps-src

### Estrutura de Diretórios do xbps-src

```
void-packages/
├── xbps-src                    # Script principal
├── srcpkgs/                    # Templates
│   ├── foo/
│   │   ├── template            # Template do pacote
│   │   ├── patches/            # Patches (opcional)
│   │   ├── files/              # Arquivos adicionais
│   │   │   └── foo.conf
│   │   └── install/            # Scripts pós-instalação
│   │       └── msg
│   └── ...
├── common/
│   ├── build-style/            # Scripts de build
│   │   ├── cmake.sh
│   │   ├── meson.sh
│   │   ├── cargo.sh
│   │   ├── gnu-configure.sh
│   │   ├── go.sh
│   │   ├── python3-pep517.sh
│   │   ├── void.sh
│   │   └── ...
│   ├── build-helpers/          # Helpers
│   │   ├── cmake.sh
│   │   ├── meson.sh
│   │   ├── rust.sh
│   │   └── ...
│   ├── shlibs                  # Mapeamento SONAME -> pacote
│   ├── cross-profiles/         # Perfis de cross-compilation
│   └── environment/
│       ├── setup.sh
│       └── ...
├── etc/
│   ├── defaults.conf           # Configurações padrão
│   └── conf                    # Configurações locais
└── masterdir/                  # Chroot de build (criado após bootstrap)
```

### Configuração (etc/conf)

```bash
# Configurações recomendadas para performance
XBPS_MAKEJOBS=$(nproc)         # Usar todos os cores
XBPS_CCACHE=yes                # Habilitar ccache (acelera rebuilds)
XBPS_CHECK_PKGS=full           # Rodar testes completos

# Para pacotes restricted (não distribuídos oficialmente)
XBPS_ALLOW_RESTRICTED=yes

# Configurações adicionais
XBPS_DISTCC=no                 # Habilitar se tiver outros hosts
XBPS_PARALLEL_INSTALL=8        # Jobs paralelos para install
```

### Build Styles Disponíveis

| build_style | Uso | Comandos executados |
|-------------|-----|---------------------|
| `meson` | Projetos C/C++ com Meson | `meson setup build`, `meson compile -C build`, `meson install -C build` |
| `cmake` | Projetos C/C++ com CMake | `cmake -B build`, `cmake --build build`, `cmake --install build` |
| `gnu-configure` | Autotools/GNU configure | `./configure --prefix=/usr`, `make`, `make install DESTDIR=$DESTDIR` |
| `gnu-makefile` | Makefiles simples | `make`, `make install DESTDIR=$DESTDIR` |
| `cargo` | Projetos Rust/Cargo | `cargo build --release`, `cargo install` |
| `go` | Projetos Go | `go build`, `go install` |
| `python3-pep517` | Pacotes Python (PEP 517) | `python3 -m build`, `pip install` |
| `python3-module` | Módulos Python simples | `python3 setup.py install` |
| `qmake` | Projetos Qt (Qt5) | `qmake`, `make`, `make install` |
| `qmake6` | Projetos Qt (Qt6) | `qmake6`, `make`, `make install` |
| `void` | Binários pré-compilados | Apenas `do_install()` manual |
| `perl-module` | Módulos Perl | `perl Makefile.PL`, `make`, `make install` |
| `ruby-module` | Módulos Ruby | `gem build`, `gem install` |
| `fetch` | Apenas baixar arquivos | Sem build, apenas `do_install()` |

### Funções Helper (Globais)

Estas funções podem ser usadas em qualquer template:

```bash
# Instalação de arquivos
vbin <arquivo> [<nome>]              # Instala em usr/bin/ (modo 0755)
vman <arquivo>                       # Instala em usr/share/man/manN/
vdoc <arquivo>                       # Instala em usr/share/doc/$pkgname/
vlicense <arquivo>                   # Instala em usr/share/licenses/$pkgname/
vconf <arquivo>                      # Instala em etc/
vsconf <arquivo>                     # Instala em usr/share/examples/$pkgname/
vmkdir <diretorio> [<modo>]          # Cria diretório no DESTDIR
vcopy <padrao> <destino>             # Copia recursivamente
vmove <padrao>                       # Move arquivos

# Serviços runit
vsv <servico> [<facility>]           # Instala em /etc/sv/

# Completion de shell
vcompletion <arquivo> <shell> [<cmd>]  # bash, fish, ou zsh

# Extração
vextract [-C <dir>] [--strip-components=N] <arquivo>
vsrccopy <arquivo> <destino>

# Manipulação de texto
vsed -i <arquivo> -e <regex>         # sed com verificação de checksum
```

### Variáveis Importantes

```bash
# Diretórios
${wrksrc}                    # Diretório de trabalho (extraído)
${build_wrksrc}              # Subdiretório para build
${XBPS_BUILDDIR}             # masterdir/builddir
${DESTDIR}                   # masterdir/destdir/${sourcepkg}-${version}
${PKGDESTDIR}                # masterdir/destdir/${pkgname}-${version} (subpackages)
${FILESDIR}                  # srcpkgs/${pkgname}/files
${XBPS_SRCDISTDIR}           # Diretório com sources baixados
${XBPS_SRCPKGDIR}            # Diretório srcpkgs
${XBPS_WRAPPERDIR}           # Diretório de wrappers

# Arquitetura
${XBPS_MACHINE}              # Arquitetura do host (ex: x86_64)
${XBPS_TARGET_MACHINE}       # Arquitetura alvo (cross-compile)
${XBPS_LIBC}                 # libc (glibc ou musl)
${XBPS_TARGET_LIBC}          # libc alvo
${XBPS_WORDSIZE}             # 32 ou 64 bits
${XBPS_ENDIAN}               # le ou be
${XBPS_CROSS_BASE}           # Base para cross-compilation
${XBPS_RUST_TARGET}          # Target triplet para Rust

# Compilação
${makejobs}                  # -jN se XBPS_MAKEJOBS definido
${make_cmd}                  # Comando make (default: make)
${configure_script}          # Script de configure
${configure_args}            # Argumentos de configure
${make_build_args}           # Argumentos de build
${make_install_args}         # Argumentos de install
${make_check_target}         # Target para check (default: check)
${make_check_args}           # Argumentos para check

# Misc
${XBPS_FETCH_CMD}            # Comando para baixar arquivos
${XBPS_BUILD_ENVIRONMENT}    # Ambiente de CI
```

### Template Completo de Exemplo

```bash
# Template file for 'swaylock-effects'
pkgname=swaylock-effects
version=1.7.0
revision=1

build_style=meson

hostmakedepends="
 pkg-config
 meson
 ninja
 wayland
"

makedepends="
 wayland-devel
 cairo-devel
 pam-devel
 libxkbcommon-devel
 gdk-pixbuf-devel
"

checkdepends="
 bash
"

depends="
 xorg-server-xwayland
"

short_desc="Screen locker for Wayland with blur effects"
maintainer="Your Name <email@example.com>"
license="MIT"
homepage="https://github.com/mortie/swaylock-effects"

changelog="https://github.com/mortie/swaylock-effects/releases"

distfiles="https://github.com/mortie/swaylock-effects/archive/v${version}.tar.gz"
checksum="abc123def456..."

# Configurações opcionais do build
configure_args="-Dpam=enabled -Dgdk-pixbuf=enabled"

# Desabilitar se testes não funcionam
make_check=no

# Não gerar pacote de debug
nodebug=yes

# Arquivos de configuração
conf_files="/etc/swaylock/config"

# Tags para catalogação
tags="wayland security locker"

# Manter libtool archives
keep_libtool_archives=yes
```

---

## 9. Processo Completo de Build

### Fases de Build do xbps-src

```
┌─────────────────────────────────────────────────────────────────┐
│                     Fases de Build                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. setup      → Prepara ambiente                              │
│  2. fetch      → Baixa sources (distfiles)                     │
│  3. extract    → Extrai tarball em ${wrksrc}                   │
│  4. patch      → Aplica patches de srcpkgs/$pkgname/patches/   │
│  5. configure  → Executa ./configure ou equivalente            │
│  6. build      → Compila (make, ninja, cargo, etc)             │
│  7. check      → Roda testes (se configurado)                  │
│  8. install    → Instala em DESTDIR                            │
│  9. pkg        → Gera pacote .xbps                             │
│  10. clean     → Limpa (opcional)                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Comandos de Build

```bash
# Build completo (todas as fases)
./xbps-src pkg <pkgname>

# Build sem testes (pular check)
./xbps-src pkg <pkgname> -Q

# Forçar rebuild (mesmo se já existe)
./xbps-src -f pkg <pkgname>

# Build com mais opções de debug
./xbps-src -d pkg <pkgname>

# Preservar destdir após build (não limpar)
./xbps-src -C pkg <pkgname>

# Build com logs detalhados
./xbps-src -v pkg <pkgname>

# Rodar apenas uma fase
./xbps-src fetch <pkgname>
./xbps-src extract <pkgname>
./xbps-src patch <pkgname>
./xbps-src configure <pkgname>
./xbps-src build <pkgname>
./xbps-src check <pkgname>
./xbps-src install <pkgname>

# Verificar dependências
./xbps-src check-deps <pkgname>

# Listar opções disponíveis
./xbps-src -h
```

### Comandos Úteis

```bash
# Limpar build anterior
./xbps-src clean <pkgname>

# Limpar completamente (remover masterdir)
./xbps-src zap

# Reconstruir masterdir (após zmudanças no sistema)
./xbps-src binary-bootstrap

# Verificar versões disponíveis (do void-packages)
./xbps-src update-check

# Verificar um pacote específico
./xbps-src update-check neovim

# Instalar pacote local (usando xtools)
xi <pkgname>

# Ou manualmente
sudo xbps-install --repository hostdir/binpkgs <pkgname>

# Listar pacotes no repositório local
ls hostdir/binpkgs/*.xbps

# Verificar informações de um pacote
xbps-query --repository hostdir/binpkgs -i <pkgname>
```

### Estrutura do Pacote Gerado

```
hostdir/
└── binpkgs/
    ├── neovim-0.10.0_1.x86_64.xbps          # Pacote binário
    ├── neovim-0.10.0_1.x86_64.xbps.sig       # Assinatura
    ├── swaylock-effects-1.7.0_1.x86_64.xbps
    └── x86_64-repodata                        # Índice do repositório
```

### Estrutura Interna de um Pacote .xbps

Um pacote XBPS é um arquivo compactado que contém:

```
pacote.xbps
├── FILES.XML              # Lista de arquivos
├── META.XML               # Metadados do pacote
├── INSTALL                # Script pós-instalação
├── REMOVE                 # Script pré-remoção
├── INSTALL.msg            # Mensagem pós-instalação
├── REMOVE.msg             # Mensagem pré-remoção
└── usr/
    ├── bin/
    │   └── programa
    ├── lib/
    │   └── libfoo.so.1
    └── share/
        └── man/
            └── man1/
                └── programa.1
```

---

## 10. Criação e Atualização do Repositório XBPS

### Criar Repositório Local

```bash
# Criar diretório do repositório
mkdir -p repository

# Copiar pacotes compilados
cp hostdir/binpkgs/*.xbps repository/

# Criar índice do repositório
xbps-rindex -a repository/*.xbps

# Resultado:
# repository/x86_64-repodata (criado)
# repository/x86_64-repodata.sig (se assinatura habilitada)
```

### Atualizar Repositório

```bash
# Adicionar novos pacotes (atualiza índice automaticamente)
xbps-rindex -a repository/*.xbps

# Forçar re-registro (mesmo versão existente)
xbps-rindex -a -f repository/*.xbps

# Limpar entradas inválidas (arquivos inexistentes)
xbps-rindex -c repository/

# Remover pacotes obsoletos (versões antigas)
xbps-rindex -r repository/

# Verificar integridade do índice
xbps-rindex -c repository/
```

### Assinar Repositório (Recomendado)

```bash
# Gerar chaves de assinatura
xbps-rindex -S
# Cria: repository/xbps-signature (chave privada)

# Assinar repositório
xbps-rindex -s repository/
# Cria: repository/x86_64-repodata.sig

# Verificar assinatura
xbps-rindex -v repository/
```

### Compartilhar Repositório

#### Opção 1: Servir via HTTP (Python)

```bash
# Servir na porta 8080
cd repository
python3 -m http.server 8080

# Acessar: http://localhost:8080/
```

#### Opção 2: Servir via HTTP (nginx)

```nginx
# /etc/nginx/sites-available/void-repo
server {
    listen 80;
    server_name repo.local;

    root /path/to/repository;
    autoindex on;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

#### Opção 3: Servir via HTTP (Apache)

```apache
# /etc/apache2/sites-available/void-repo.conf
<VirtualHost *:80>
    ServerName repo.local
    DocumentRoot /path/to/repository

    <Directory /path/to/repository>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>
```

#### Opção 4: Usar com rclone (S3, etc)

```bash
# Sincronizar com S3
rclone sync repository/ s3:my-bucket/void-repo/

# Ou com Google Drive
rclone sync repository/ gdrive:void-repo/
```

### Configurar Cliente para Usar Repositório

```bash
# Criar arquivo de configuração
sudo mkdir -p /etc/xbps.d

# Para repositório local
echo "repository=/path/to/repository" | sudo tee /etc/xbps.d/00-local.conf

# Para repositório HTTP
echo "repository=http://localhost:8080/current" | sudo tee /etc/xbps.d/00-local.conf

# Para repositório com prioridade
echo "repository=/path/to/repository" | sudo tee /etc/xbps.d/01-custom.conf

# Sincronizar índices
sudo xbps-install -S
```

---

## 11. Instalação e Atualização de Pacotes

### Instalar de Repositório Local

```bash
# Instalar pacote específico
sudo xbps-install --repository repository/ <pkgname>

# Instalar com atualização de índices
sudo xbps-install -S --repository repository/ <pkgname>

# Instalar vários pacotes
sudo xbps-install --repository repository/ pkg1 pkg2 pkg3

# Forçar reinstalação
sudo xbps-install -f --repository repository/ <pkgname>
```

### Usando xtools (Recomendado)

```bash
# Instalar xtools
sudo xbps-install xtools

# Instalar pacote local (detecta automaticamente)
xi <pkgname>

# Listar pacotes disponíveis localmente
xpkg -m

# Buscar pacote
xargs xbps-query -Rs <pattern>

# Verificar dependências
xdeptree <pkgname>

# Verificar arquivos do pacote
xpkg -f <pkgname>
```

### Atualizar Pacotes

```bash
# Atualizar todos os pacotes
sudo xbps-install -Su

# Atualizar de repositório específico
sudo xbps-install -Su --repository repository/

# Forçar atualização (reinstall)
sudo xbps-install -f <pkgname>

# Verificar pacotes desatualizados
xbps-install -M
```

### Remover Pacotes

```bash
# Remover pacote
sudo xbps-remove <pkgname>

# Remover com dependências órfãs
sudo xbps-remove -o <pkgname>

# Remover completamente (incluindo configurações)
sudo xbps-remove -Oo <pkgname>

# Limpar cache de downloads
sudo xbps-remove -Oo
```

### Downgrade

```bash
# Usar xdowngrade (mais fácil)
sudo xdowngrade /var/cache/xbps/pkg-1.0_1.xbps

# Ou manualmente
sudo xbps-rindex -a /path/to/old-package.xbps
sudo xbps-install -R /path/to/ -f <pkgname>-<old_version>
```

### Gerenciar Pacotes

```bash
# Colocar pacote em hold (não atualizar)
sudo xbps-pkgdb -m hold <pkgname>

# Remover hold
sudo xbps-pkgdb -m unhold <pkgname>

# Marcar como manual (não ser detectado como órfão)
sudo xbps-pkgdb -m manual <pkgname>

# Marcar como automático (ser detectado como órfão)
sudo xbps-pkgdb -m auto <pkgname>

# Verificar erros no banco de dados
sudo xbps-pkgdb -a

# Forçar reconfiguração
sudo xbps-reconfigure -f <pkgname>
```

---

## 12. Fluxos Completos (Diagramas)

### Fluxo de Sincronização

```
┌─────────────────────────────────────────────────────────────────┐
│                     manager sync                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Para cada pacote │
                    │ em packages.db  │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  git fetch      │
                    │  --prune        │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  Nova tag ou    │
                    │  commit?        │
                    └────────┬────────┘
                             │
                ┌────────────┼────────────┐
                │            │            │
               Não          Sim         Erro
                │            │            │
                ▼            ▼            ▼
           ┌────────┐  ┌──────────┐  ┌─────────┐
           │ Próximo│  │ Atualizar│  │  Log    │
           │ pacote │  │ template │  │  erro   │
           └────────┘  └────┬─────┘  └─────────┘
                            │
                            ▼
                    ┌─────────────────┐
                    │  xbps-src pkg   │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  xbps-rindex -a │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  Atualizar      │
                    │  packages.db    │
                    └─────────────────┘
```

### Fluxo de Instalação

```
┌─────────────────────────────────────────────────────────────────┐
│                  manager install <pkgname>                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  Existe no      │
                    │  repository/?   │
                    └────────┬────────┘
                             │
                ┌────────────┼────────────┐
                │            │            │
               Não          Sim         Erro
                │            │            │
                ▼            │            ▼
        ┌──────────────┐     │       ┌─────────┐
        │  Verificar   │     │       │  Log    │
        │  packages.db │     │       │  erro   │
        └──────┬───────┘     │       └─────────┘
               │             │
               ▼             │
        ┌──────────────┐     │
        │  Sincronizar │     │
        │  cache git   │     │
        └──────┬───────┘     │
               │             │
               ▼             │
        ┌──────────────┐     │
        │  Atualizar   │     │
        │  template    │     │
        └──────┬───────┘     │
               │             │
               ▼             │
        ┌──────────────┐     │
        │  xbps-src    │     │
        │  pkg         │     │
        └──────┬───────┘     │
               │             │
               ▼             │
        ┌──────────────┐     │
        │  xbps-rindex │     │
        │  -a          │     │
        └──────┬───────┘     │
               │             │
               └──────┬──────┘
                      │
                      ▼
              ┌───────────────┐
              │  xbps-install │
              │  --repository │
              └───────────────┘
```

### Fluxo de Build Automatizado

```
┌─────────────────────────────────────────────────────────────────┐
│                    manager build <pkgname>                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  Ler template   │
                    │  (srcpkgs/)     │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  Verificar      │
                    │  dependências   │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  ./xbps-src pkg │
                    │  <pkgname>      │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  Sucesso?       │
                    └────────┬────────┘
                             │
                ┌────────────┼────────────┐
                │            │            │
               Sim          Não         Timeout
                │            │            │
                ▼            ▼            ▼
        ┌──────────────┐  ┌─────────┐  ┌─────────┐
        │  xbps-rindex │  │  Log    │  │  Retry  │
        │  -a          │  │  erro   │  │  ou log │
        └──────┬───────┘  └─────────┘  └─────────┘
               │
               ▼
        ┌──────────────┐
        │  Atualizar   │
        │  packages.db │
        └──────┬───────┘
               │
               ▼
        ┌──────────────┐
        │  Log sucesso │
        └──────────────┘
```

---

## 13. Implementação Sugerida (Bash)

### Script Principal (manager)

```bash
#!/bin/bash
# manager - Gerenciador de Pacotes Baseado em XBPS
# Uso: ./manager <comando> [pacote]

set -euo pipefail

# ============================================================
# Configurações
# ============================================================

REPO_DIR="$(cd "$(dirname "$0")" && pwd)"
PKGS_DB="${REPO_DIR}/database/packages.db"
CACHE_DIR="${REPO_DIR}/cache/git"
XBPS_SRC="${REPO_DIR}/void-packages"
SRC_PKGS="${REPO_DIR}/srcpkgs"
PACKAGES_DIR="${REPO_DIR}/packages"
LOGS_DIR="${REPO_DIR}/logs"

# Cores para output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

# ============================================================
# Funções auxiliares
# ============================================================

log() {
    echo -e "${GREEN}[$(date '+%Y-%m-%d %H:%M:%S')]${NC} $*"
}

error() {
    echo -e "${RED}[ERRO]${NC} $*" >&2
}

warn() {
    echo -e "${YELLOW}[AVISO]${NC} $*"
}

info() {
    echo -e "${BLUE}[INFO]${NC} $*"
}

# ============================================================
# Funções do Banco de Dados
# ============================================================

db_init() {
    mkdir -p "${REPO_DIR}/database"
    touch "${PKGS_DB}"
    log "Banco de dados inicializado"
}

db_get_package() {
    local pkgname="$1"
    grep "^${pkgname}|" "${PKGS_DB}" 2>/dev/null || true
}

db_update_package() {
    local pkgname="$1"
    local url="$2"
    local branch="$3"
    local last_commit="$4"
    local last_tag="$5"
    local version="$6"
    local revision="$7"
    local build_style="$8"
    local last_sync="$9"
    local last_build="${10}"
    local status="${11}"

    # Remover entrada antiga
    sed -i "/^${pkgname}|/d" "${PKGS_DB}" 2>/dev/null || true

    # Adicionar nova entrada
    echo "${pkgname}|${url}|${branch}|${last_commit}|${last_tag}|${version}|${revision}|${build_style}|${last_sync}|${last_build}|${status}" >> "${PKGS_DB}"
}

db_list_packages() {
    cut -d'|' -f1 "${PKGS_DB}" 2>/dev/null || true
}

db_get_field() {
    local pkgname="$1"
    local field="$2"
    db_get_package "${pkgname}" | cut -d'|' -f"${field}"
}

# ============================================================
# Funções do Cache Git
# ============================================================

git_sync_mirror() {
    local pkgname="$1"
    local url="$2"
    local owner
    owner=$(echo "${url}" | sed 's|https://github.com/||' | cut -d'/' -f1)
    local repo
    repo=$(echo "${url}" | sed 's|https://github.com/||' | cut -d'/' -f2 | sed 's|\.git$||')
    local cache_path="${CACHE_DIR}/${owner}/${repo}.git"

    if [ ! -d "${cache_path}" ]; then
        log "Criando mirror para ${pkgname}..."
        mkdir -p "${CACHE_DIR}/${owner}"
        git clone --mirror "${url}" "${cache_path}" 2>/dev/null
    else
        log "Atualizando mirror para ${pkgname}..."
        git -C "${cache_path}" fetch --prune --tags 2>/dev/null || true
    fi

    echo "${cache_path}"
}

git_get_newest_tag() {
    local cache_path="$1"
    git -C "${cache_path}" tag -l "v*" --sort=-version:refname 2>/dev/null | head -1
}

git_get_newest_commit() {
    local cache_path="$1"
    git -C "${cache_path}" rev-parse HEAD 2>/dev/null
}

git_get_tag_version() {
    # Remove prefix 'v' se existir
    echo "${1#v}"
}

# ============================================================
# Funções de Template
# ============================================================

template_update_version() {
    local pkgname="$1"
    local new_version="$2"
    local template="${SRC_PKGS}/${pkgname}/template"

    if [ ! -f "${template}" ]; then
        error "Template não encontrado: ${template}"
        return 1
    fi

    # Atualizar version
    sed -i "s/^version=.*/version=${new_version}/" "${template}"

    # Resetar revision
    sed -i "s/^revision=.*/revision=1/" "${template}"

    # Calcular novo checksum
    local distfiles_url
    distfiles_url=$(grep "^distfiles=" "${template}" | sed 's/distfiles="//' | sed 's/"$//')
    distfiles_url=$(eval echo "${distfiles_url//\$\{version\}/${new_version}}")

    info "Baixando tarball para calcular checksum..."
    local checksum
    checksum=$(curl -sL "${distfiles_url}" | sha256sum | cut -d' ' -f1)

    sed -i "s/^checksum=.*/checksum=\"${checksum}\"/" "${template}"

    log "Template ${pkgname} atualizado para versão ${new_version}"
}

# ============================================================
# Funções de Build
# ============================================================

build_package() {
    local pkgname="$1"

    log "Compilando ${pkgname}..."

    cd "${XBPS_SRC}"

    if ./xbps-src pkg "${pkgname}" 2>&1 | tee "${LOGS_DIR}/build-${pkgname}.log"; then
        log "Build de ${pkgname} concluído com sucesso"

        # Copiar pacotes para packages/
        mkdir -p "${PACKAGES_DIR}"
        cp hostdir/binpkgs/*.xbps "${PACKAGES_DIR}/" 2>/dev/null || true

        # Atualizar índice do repositório
        xbps-rindex -a "${PACKAGES_DIR}"/*.xbps 2>/dev/null || true

        cd "${REPO_DIR}"
        return 0
    else
        error "Build de ${pkgname} falhou"
        cd "${REPO_DIR}"
        return 1
    fi
}

# ============================================================
# Funções de Sincronização
# ============================================================

sync_package() {
    local pkgname="$1"
    local pkg_data
    pkg_data=$(db_get_package "${pkgname}")

    if [ -z "${pkg_data}" ]; then
        error "Pacote ${pkgname} não encontrado no banco de dados"
        return 1
    fi

    # Extrair dados
    IFS='|' read -r name url branch last_commit last_tag version revision build_style last_sync last_build status <<< "${pkg_data}"

    log "Sincronizando ${pkgname}..."

    # Sincronizar mirror
    local cache_path
    cache_path=$(git_sync_mirror "${pkgname}" "${url}")

    # Verificar nova tag
    local newest_tag
    newest_tag=$(git_get_newest_tag "${cache_path}")

    local newest_commit
    newest_commit=$(git_get_newest_commit "${cache_path}")

    local needs_update=false

    if [ -n "${newest_tag}" ] && [ "${newest_tag}" != "${last_tag}" ]; then
        log "Nova tag encontrada: ${newest_tag} (anterior: ${last_tag})"
        needs_update=true
    elif [ "${newest_commit}" != "${last_commit}" ]; then
        log "Novo commit encontrado: ${newest_commit:0:7} (anterior: ${last_commit:0:7})"
        needs_update=true
    fi

    if [ "${needs_update}" = true ]; then
        # Determinar nova versão
        local new_version
        if [ -n "${newest_tag}" ]; then
            new_version=$(git_get_tag_version "${newest_tag}")
        else
            new_version="${version}"  # Manter versão para rolling release
        fi

        # Atualizar template
        template_update_version "${pkgname}" "${new_version}"

        # Compilar
        if build_package "${pkgname}"; then
            db_update_package "${pkgname}" "${url}" "${branch}" "${newest_commit}" "${newest_tag}" "${new_version}" "${revision}" "${build_style}" "$(date -Iseconds)" "$(date -Iseconds)" "built"
            log "${pkgname} sincronizado e compilado com sucesso"
        else
            db_update_package "${pkgname}" "${url}" "${branch}" "${newest_commit}" "${newest_tag}" "${new_version}" "${revision}" "${build_style}" "$(date -Iseconds)" "" "error"
            error "Falha ao compilar ${pkgname}"
        fi
    else
        log "${pkgname} já está atualizado"
    fi
}

sync_all() {
    log "Iniciando sincronização de todos os pacotes..."

    while IFS= read -r pkgname; do
        sync_package "${pkgname}" || true
    done < <(db_list_packages)

    log "Sincronização concluída"
}

# ============================================================
# Funções de Instalação
# ============================================================

install_package() {
    local pkgname="$1"

    log "Instalando ${pkgname}..."

    if ls "${PACKAGES_DIR}/${pkgname}"*.xbps 1> /dev/null 2>&1; then
        sudo xbps-install --repository "${PACKAGES_DIR}/" "${pkgname}"
    else
        error "Pacote ${pkgname} não encontrado em ${PACKAGES_DIR}"
        return 1
    fi
}

# ============================================================
# Comandos
# ============================================================

cmd_help() {
    cat << EOF
Uso: manager <comando> [pacote]

Comandos:
    init                  Inicializar banco de dados e estrutura
    add <url>             Adicionar pacote do GitHub
    remove <pkgname>      Remover pacote do banco de dados
    sync [pkgname]        Sincronizar (todos ou específico)
    build [pkgname]       Compilar (todos ou específico)
    install <pkgname>     Instalar pacote
    list                  Listar pacotes
    status                Mostrar status do sistema
    help                  Mostrar esta ajuda

Exemplos:
    manager init
    manager add https://github.com/mortie/swaylock-effects
    manager sync swaylock-effects
    manager build
    manager install swaylock-effects
    manager list
    manager status
EOF
}

cmd_init() {
    mkdir -p "${CACHE_DIR}" "${PACKAGES_DIR}" "${LOGS_DIR}" "${REPO_DIR}/database"
    touch "${PKGS_DB}"
    log "Estrutura inicializada"
}

cmd_add() {
    local url="$1"
    local owner
    owner=$(echo "${url}" | sed 's|https://github.com/||' | cut -d'/' -f1)
    local repo
    repo=$(echo "${url}" | sed 's|https://github.com/||' | cut -d'/' -f2 | sed 's|\.git$||')
    local pkgname="${repo}"

    if db_get_package "${pkgname}" | grep -q .; then
        error "Pacote ${pkgname} já existe no banco de dados"
        return 1
    fi

    # Criar mirror
    local cache_path
    cache_path=$(git_sync_mirror "${pkgname}" "${url}")

    # Detectar informações
    local newest_tag
    newest_tag=$(git_get_newest_tag "${cache_path}")

    local newest_commit
    newest_commit=$(git_get_newest_commit "${cache_path}")

    local version
    if [ -n "${newest_tag}" ]; then
        version=$(git_get_tag_version "${newest_tag}")
    else
        version="0.0.1"
    fi

    # Adicionar ao banco de dados
    echo "${pkgname}|${url}|master|${newest_commit}|${newest_tag}|${version}|1|meson|$(date -Iseconds)||pending" >> "${PKGS_DB}"

    log "Pacote ${pkgname} adicionado ao banco de dados"
}

cmd_remove() {
    local pkgname="$1"
    sed -i "/^${pkgname}|/d" "${PKGS_DB}"
    log "Pacote ${pkgname} removido do banco de dados"
}

cmd_list() {
    echo "Pacotes no banco de dados:"
    echo "---------------------------"
    printf "%-25s %-10s %-8s %-10s\n" "NOME" "VERSÃO" "REV" "STATUS"
    echo "---------------------------"
    while IFS='|' read -r name url branch commit tag version revision style sync build status; do
        printf "%-25s %-10s %-8s %-10s\n" "${name}" "${version}" "${revision}" "${status}"
    done < "${PKGS_DB}"
}

cmd_status() {
    echo "Status do Gerenciador"
    echo "===================="
    echo ""
    echo "Pacotes totais: $(db_list_packages | wc -l)"
    echo "Pacotes construídos: $(grep '|built$' "${PKGS_DB}" | wc -l)"
    echo "Pacotes com erro: $(grep '|error$' "${PKGS_DB}" | wc -l)"
    echo "Pacotes pendentes: $(grep '|pending$' "${PKGS_DB}" | wc -l)"
    echo ""
    echo "Cache Git:"
    echo "  Mirrors: $(find "${CACHE_DIR}" -name "*.git" -type d 2>/dev/null | wc -l)"
    echo "  Tamanho: $(du -sh "${CACHE_DIR}" 2>/dev/null | cut -f1)"
    echo ""
    echo "Pacotes compilados:"
    echo "  Total: $(ls -1 "${PACKAGES_DIR}"/*.xbps 2>/dev/null | wc -l)"
    echo "  Tamanho: $(du -sh "${PACKAGES_DIR}" 2>/dev/null | cut -f1)"
}

# ============================================================
# Main
# ============================================================

main() {
    local cmd="${1:-help}"
    shift || true

    mkdir -p "${LOGS_DIR}"

    case "${cmd}" in
        init)
            cmd_init
            ;;
        add)
            if [ -z "${1:-}" ]; then
                error "Uso: manager add <url>"
                exit 1
            fi
            cmd_add "$1"
            ;;
        remove)
            if [ -z "${1:-}" ]; then
                error "Uso: manager remove <pkgname>"
                exit 1
            fi
            cmd_remove "$1"
            ;;
        sync)
            if [ -n "${1:-}" ]; then
                sync_package "$1"
            else
                sync_all
            fi
            ;;
        build)
            if [ -n "${1:-}" ]; then
                build_package "$1"
            else
                while IFS= read -r pkgname; do
                    build_package "${pkgname}" || true
                done < <(db_list_packages)
            fi
            ;;
        install)
            if [ -z "${1:-}" ]; then
                error "Uso: manager install <pkgname>"
                exit 1
            fi
            install_package "$1"
            ;;
        list)
            cmd_list
            ;;
        status)
            cmd_status
            ;;
        help|*)
            cmd_help
            ;;
    esac
}

main "$@"
```

### Como Usar

```bash
# 1. Tornar executável
chmod +x manager

# 2. Inicializar estrutura
./manager init

# 3. Adicionar pacotes do GitHub
./manager add https://github.com/mortie/swaylock-effects
./manager add https://github.com/neovim/neovim
./manager add https://github.com/programmerjake4/foot
./manager add https://github.com/sharkdp/bat

# 4. Verificar pacotes adicionados
./manager list

# 5. Sincronizar todos (buscar atualizações)
./manager sync

# 6. Sincronizar pacote específico
./manager sync swaylock-effects

# 7. Compilar todos os pacotes
./manager build

# 8. Compilar pacote específico
./manager build bat

# 9. Instalar pacote
./manager install bat

# 10. Verificar status
./manager status
```

### Exemplo de Saída

```
$ ./manager list
Pacotes no banco de dados:
---------------------------
NOME                      VERSÃO    REV      STATUS
---------------------------
neovim                    0.10.0    1        built
swaylock-effects          1.7.0     1        built
bat                       0.24.0    1        built
foot                      1.17.0    1        pending

$ ./manager status
Status do Gerenciador
====================

Pacotes totais: 4
Pacotes construídos: 3
Pacotes com erro: 0
Pacotes pendentes: 1

Cache Git:
  Mirrors: 4
  Tamanho: 15M

Pacotes compilados:
  Total: 3
  Tamanho: 25M
```

---

## 14. Melhorias Futuras

### Funcionalidades Planejadas

1. **Suporte a Sub-repositórios**
   - `nonfree/`, `multilib/`, `debug/`
   - Configuração por pacote

2. **Paralelização de Builds**
   - Usar `xargs -P` ou GNU parallel
   - Build simultâneo de pacotes independentes

3. **Assinatura de Pacotes**
   - Geração automática de chaves
   - Assinatura de repositórios

4. **Mirror Local HTTP**
   - Servidor embutido para distribuição
   - Cache de pacotes para reinstalação

5. **Integração com CI/CD**
   - GitHub Actions
   - Build automático em push

6. **Suporte a Cross-Compilation**
   - Builds para ARM, aarch64
   - Configuração por arquitetura

7. **Relatórios e Métricas**
   - Dashboard web
   - Histórico de builds
   - Gráficos de dependências

8. **Cache Binário**
   - Reuso de pacotes compilados
   - Detecção de mudanças em dependências

9. **Interface Web**
   - Dashboard para gerenciamento
   - Visualização de status
   - Logs de build

10. **Notificações**
    - Email quando há atualizações
    - Webhook para Discord/Slack

---

## Referências

- [Manual do xbps-src](https://github.com/void-linux/void-packages/blob/master/Manual.md)
- [README do void-packages](https://github.com/void-linux/void-packages/blob/master/README.md)
- [Tutoriais xbps-src](https://xbps-src-tutorials.github.io/)
- [Documentação do Void Linux](https://docs.voidlinux.org/)
- [xbps-mini-builder](https://github.com/the-maldridge/xbps-mini-builder)
- [Guia de Contribuição](https://github.com/void-linux/void-packages/blob/master/CONTRIBUTING.md)
- [xbps-rindex manpage](https://man.voidlinux.org/xbps-rindex)
- [xbps-install manpage](https://man.voidlinux.org/xbps-install.1)
- [xbps-query manpage](https://man.voidlinux.org/xbps-query.1)
- [Mini guide for xbps-src](https://gist.github.com/Piraty/2e8c9fa86d4eb70f22efb9e0ecdda235)

---

## Licença

Este guia é distribuído sob a licença MIT.
